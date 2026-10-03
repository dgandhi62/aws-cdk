# CDK Synth Performance

## What This Is

Domain knowledge for investigating CDK synthesis performance. Use this when a user mentions slow synth or asks you to investigate their synth time or make it faster.

> **Read this first — scope of "synth time."** When a user says "synth is slow," the wall-clock they measure spans **four** segments, and only one of them is CDK framework synthesis:
>
> ```
> [ CLI startup ] → [ module load ] → [ construction ] → [ framework synthesis ] → [ assembly write ]
>   IMDS/creds       require() graph    your app code       synthesize()             buildAssembly
>   telemetry        ts-node/tsx       asset staging        (prepareApp, refs,       validateTemplates
>   region lookup    transpile         child_process        synthesizeTree)
> ```
>
> The function-to-phase table further down covers framework synthesis in detail — but do **not** assume that's where the time is. Attribute to one of the four segments *first*, then zoom in.

## Measurement Discipline (do this before anything else)

Bad measurement produces confident wrong answers. Before you report any number or recommend any change:

- **Same workspace, same session, same commit.** Baseline and fixed-state must be measured in the same workspace and in the same session. Never compare a baseline on one commit against a fix on a different commit — you will attribute environmental or unrelated changes to your fix. 
- **At least two runs each, report the spread.** Run-to-run variance under load has been measured at ~7.5s — easily enough to swamp a real 26s gain in a single package. Take ≥2 runs for baseline and ≥2 for fixed state; report min/median, not a single number.
- **Watch what's inside the measured command.** A chained `&& npm run ...` (or `yarn build &&`) in `cdk.json`'s `"app"` field runs *inside* the measured wall time (~3–5s observed). A nested `yarn <script>` wrapper adds ~500–900ms per invocation. Decide explicitly whether you're measuring "synth" or "build + synth," and state it.
- **Diminishing returns floor.** For small apps that load `aws-cdk-lib`, ~2s of Node startup + library load is currently unavoidable regardless of toolchain. **Below ~5s wall-clock, further investigation has sharply diminishing returns** — say so and stop rather than chasing noise.

## Framing: wall-clock vs CPU time

The **first** branch in diagnosis is comparing total wall-clock against on-CPU time from the profile:

- **Wall-clock ≈ CPU time** → the process is doing real computation. Use the CPU profile and the phase mapping below to find the hot phase.
- **Wall-clock >> CPU time** (e.g. 14s wall but 0.73s sampled CPU) → the process is **blocked on I/O or a subprocess**, not computing. The CPU profile alone will *not* explain the gap. This points at:
  - **CLI-side network waits** — IMDS region lookup, credential providers, telemetry endpoint (see CLI Startup section).
  - **Synchronous child-process spawns** during construction — a dependency shelling out (see Construction / child_process section).
  - Subprocess I/O waits that fall outside the main profile (see Multi-profile note).

Establishing this ratio early prevents the most common dead end: profiling CPU for minutes when the time is actually spent waiting.

## Construction Phase (user code + its dependencies)

"Construction" is everything between `new App()` and `app.synth()`. In real-world apps — especially `ts-node`/`tsx` apps — this is frequently the **dominant** segment (one engagement: ~90%; construct instantiation ~38%, module load/compile ~18%, asset staging ~9.5%). The framework-synthesis sub-phases were a minority. The skill must attribute this time, not treat it as a black box.

What to look at:

- **Construct instantiation** — loops that create constructs (check iteration count × per-construct cost), deep trees, third-party construct libraries doing initialization work.
- **Module load / compile** — `Module._compile` / `require.extensions` walking the `aws-cdk-lib` dep graph. Largely structural and not app-addressable, but it sets the floor. `ts-node`/`tsx` transpile cost shows up here too.
- **Asset staging** — `AssetStaging` runs during construction (not synthesis). This is where bundling subprocesses and file copies happen.
- **Synchronous `child_process` from dependencies** — see next section; this is a high value construction check and the easiest to miss.

### Intercept child_process

A dependency that shells out synchronously during construction is invisible in the CDK phase mapping and easy to misread in a CPU profile (it looks like idle/wait, not hot CPU). 

**Technique:** intercept `child_process` with a ~30-line `--require` preload and **group calls by command + argv**. The duplicate count is what makes the bug obvious — in that engagement, **186 of 207 spawns resolved just 3 distinct compile-time-constant strings**, once per stack across ~60 stacks.

```js
// spawn-trace.js — run with: NODE_OPTIONS="--require ./spawn-trace.js" cdk synth
const cp = require('child_process');
const counts = new Map();
for (const fn of ['spawnSync', 'execSync', 'execFileSync', 'spawn', 'exec', 'execFile']) {
  const orig = cp[fn];
  cp[fn] = function (...args) {
    const key = `${fn}: ${String(args[0])} ${Array.isArray(args[1]) ? args[1].join(' ') : ''}`.slice(0, 200);
    counts.set(key, (counts.get(key) || 0) + 1);
    return orig.apply(this, args);
  };
}
process.on('exit', () => {
  const rows = [...counts.entries()].sort((a, b) => b[1] - a[1]);
  console.error('\n=== child_process calls by command+argv (count desc) ===');
  for (const [k, n] of rows) console.error(`${String(n).padStart(5)}  ${k}`);
  console.error(`total: ${rows.reduce((s, [, n]) => s + n, 0)} calls, ${rows.length} distinct\n`);
});
```

A high total with a tiny distinct count = redundant work that should be cached or hoisted. A high distinct count = genuinely varied work (e.g. real per-asset bundling).

### "Does the library already do this?" — check before recommending code changes

Before recommending that the user hand-write an optimization (memoization, caching, batching), **verify the dependency doesn't already provide it in a newer version.** In one engagement, hand-written memoization was shipped that a dependency version bump made entirely redundant — the real fix was a **one-line dependency bump per package, no code changes.**

- Reading a dependency's *resolved* source and seeing no cache does **not** prove the installed behavior — a newer version may add it, and the resolved version may be old. (`@amzn/brazil` memoizes `brazil-path` from 2.0.7 on by default; only the package on an older pinned version lacked it.)
- The tell that you missed this: one package measures **zero improvement** from a change that helped others — that usually means it resolved a different (newer or older) version. Catch it by checking resolved versions across packages up front, not after.
- Order of preference for a fix: **dependency bump > config change > hand-written code.**

## How to Capture a CPU Profile

Run CDK synth with `NODE_OPTIONS` set to enable CPU profiling. Use `--cpu-prof` to produce a `.cpuprofile` file, `--cpu-prof-dir` to control where it lands, and `--max-old-space-size=8192` to prevent OOM before the profile flushes.

The resulting `.cpuprofile` is JSON with:
- `nodes`: call frame objects with `id`, `callFrame` (functionName, url, lineNumber), `hitCount`, `children`
- `samples`: array of node IDs (one per sample interval, ~1ms)
- `timeDeltas`: array of microsecond deltas between samples

### Multiple profiles — one per subprocess

`--cpu-prof` writes **one file per Node process**. A typical invocation chain (`yarn → cdk → node → ts-node`) produces 6–7 profiles. Guidance:
- The **largest file is almost always the app process** — start there.
- Each profile only sees its own process's CPU. **Time that's missing from every profile is usually subprocess I/O waits**, not real CPU work — reconcile the sum of sampled CPU against wall-clock, and if there's a large gap, go back to the wall-clock-vs-CPU branch and the child_process / CLI-startup sections.

### Reading the Profile

The profile is JSON. Read it and look at the `nodes` array — each node has a `callFrame` with `functionName` and `url` (file path). The `samples` array tells you which node was on-CPU at each ~1ms tick. The `timeDeltas` array gives the microsecond gap between ticks.

To determine where time is spent, look at which functions appear in the call stack for each sample. A sample "belongs" to a phase based on which function is an ancestor in the call tree. **Bucket samples into the four top-level segments first** (CLI startup / module load / construction / framework synthesis), then — only if framework synthesis is material — into the sub-phases below.

## Function Name → Phase Mapping (framework synthesis)

> This table covers the **framework-synthesis** segment only. If your profile is construction- or load-dominant, most samples will fall *outside* these functions — that is expected, and the Construction and CLI Startup sections above are where the guidance lives. Don't force construction samples into this table.

The code path is in `packages/aws-cdk-lib/core/lib/private/synthesis.ts`:

```
App constructor            → marks start time (performance.now())
  ... user code ...        → "Construction" phase (everything until app.synth())
App.synth()                → calls synthesize()
  synthesize():
    injectTreeMetadata()
    synthNestedAssemblies()
    invokeAspects() or invokeAspectsV2()
    injectMetadataResources()
    prepareApp()           → resolves cross-stack refs, reifies deps
    validateTree()
    synthesizeTree()       → renders each stack to CloudFormation JSON
    generateFeatureFlagReport()
    builder.buildAssembly()
    validateTemplates()
```

| Function | File | What it does |
|----------|------|-------------|
| `synthesize` | `core/lib/private/synthesis.ts` | Top-level synthesis orchestrator |
| `synthNestedAssemblies` | `core/lib/private/synthesis.ts` | Calls `synth()` on nested Stage constructs |
| `invokeAspects` | `core/lib/private/synthesis.ts` | Runs registered aspects on all constructs |
| `invokeAspectsV2` | `core/lib/private/synthesis.ts` | Aspects with stabilization loop (feature flag) |
| `prepareApp` | `core/lib/private/prepare-app.ts` | Reifies construct deps to resource deps + resolves cross-stack refs |
| `findTransitiveDeps` | `core/lib/private/prepare-app.ts` | Collects all `.node.dependencies` across the tree |
| `resolveReferences` | `core/lib/private/refs.ts` | Finds and resolves cross-stack token references |
| `findAllReferences` | `core/lib/private/refs.ts` | Iterates all CfnElements, calls `findTokens` on each |
| `findTokens` | `core/lib/private/resolve.ts` | Renders a CfnElement via `_toCloudFormation()` to discover tokens |
| `operateOnDependency` | `core/lib/deps.ts` | Adds/removes a dependency edge between two elements |
| `_addAssemblyDependency` | `core/lib/stack.ts` | Records a stack-to-stack dependency |
| `validateTree` | `core/lib/private/synthesis.ts` | Calls `.node.validate()` on every construct |
| `synthesizeTree` | `core/lib/private/synthesis.ts` | Visits all constructs, calls `stack.synthesizer.synthesize()` per stack |
| `_toCloudFormation` | `core/lib/stack.ts` | Renders all CfnElements in a stack to CloudFormation template JSON |
| `buildAssembly` | `cloud-assembly-api` package | Writes the cloud assembly manifest to disk |
| `validateTemplates` | `core/lib/private/synthesis-validation.ts` | Runs policy validation plugins against rendered templates |
| `BundlingDockerImage.run` | `core/lib/bundling.ts` | Executes Docker container for asset bundling |
| `DockerImage.fromBuild` | `core/lib/bundling.ts` | Builds a Docker image from Dockerfile |

## Stack Metrics (from cdk.out)

After synth, `cdk.out/` contains one `*.template.json` per stack. From these you can extract:

- **Resource count** — number of keys in the `Resources` object of each template
- **Template file size** — file size on disk
- **Cross-stack exports** — Outputs that have an `Export` field
- **Structural similarity** — stacks with the same set of resource Types (indicates repeated patterns)
- **Asset size & file counts** — from `cdk.out/asset.*` (see Asset Size & Deploy Hygiene)

## Structural Observations

These can be collected from `cdk.out` without profiling:

| Observation | Relevance |
|-------------|-----------|
| Resource count per stack | Rendering cost in `_toCloudFormation` scales linearly with element count |
| Template file size | Serialization + disk write cost |
| Cross-stack exports (Outputs with Export) | Each triggers token scan in consuming stacks during `resolveReferences` |
| Structurally identical stacks (same resource types) | Same rendering work repeated N times |
| `CfnInclude` resources in templates | Each one parsed a full CloudFormation template at construction time |
| Number of bundled assets | Each spawned a subprocess (see child_process interception) |
| Staged asset size / file count | Deploy hygiene; dead weight (`.d.ts`, source, vendored SDKs) inflates deploy artifacts |

## Investigation Guidance

These are things to keep in mind when investigating, not a rigid procedure. Use your judgment based on what the data shows.

### Suggested order of operations

1. **Measure with discipline** (same workspace/session/commit, ≥2 runs, note variance; know what's inside the measured command).
2. **Compare wall-clock vs CPU time.** If wall >> CPU, go to CLI Startup and child_process sections before profiling further.
3. **Bucket the profile into the four top-level segments** (CLI startup / module load / construction / framework synthesis). Report the split — this one table ("construction 70%, synthesis a minority") is frequently the single most valuable output, because it rules out whole classes of hypothesis at a glance.
4. **Drill into the dominant segment** using the matching section below.
5. **Before recommending code:** check the diminishing-returns floor and the "does the library already do this?" question.

### Identifying the Dominant Segment

**CLI-startup-dominant** (wall >> CPU, time before `new App()`): see CDK CLI Startup — IMDS/region lookup, credential chain, telemetry.

**Load-dominant:**
   - Whether app uses `ts-node`/`tsx` (runtime compilation) vs pre-compiled JavaScript. Note: swapping toolchains often doesn't pay off — one engagement found `tsx` *slower* than `ts-node`, and precompiled JS only 7% faster but it broke a zero-build DX convention. Measure before recommending.
   - Size and depth of `node_modules`; number of top-level imports in the entry chain.
   - This is largely structural and sets the floor; don't over-invest below ~5s.

**Construction-dominant:** see Construction Phase. Find the app entry point via `cdk.json` → `"app"`. Look for:
   - Loops that instantiate constructs (check iteration count)
   - `fs.readFileSync` calls in user code
   - `child_process` calls in user code **or in dependencies** (run the spawn-trace preload)
   - Third-party construct libraries that do initialization work
   - Asset staging (large/many assets)
   - Context lookups that might be slow

**Synthesis-dominant:** Look at which sub-phase is expensive in the profile:
   - **prepareApp dominant:**
     - Time in `findTransitiveDeps` / `addResourceDependency` → search user code for `.node.addDependency()` calls. Count them. Check if inside loops. `.node.addDependency()` expands to N×M resource-level edges.
     - Time in `resolveReferences` / `findAllReferences` / `findTokens` → count cross-stack exports in `cdk.out` templates. Each export causes CDK to render every CfnElement in the consuming stack to find tokens.
   - **synthesizeTree dominant:** Check total resource count per stack. `_toCloudFormation` renders every CfnElement. Each element is rendered at minimum twice (once in `findAllReferences`, once in `synthesizeTree`).
   - **invokeAspects dominant:** Check how many aspects are registered and how many constructs they visit.

**Bundling-dominant:** Find which assets are bundled. Check:
   - `cdk.out/asset.*` directories — what's in them, how large, file counts
   - Presence of `.dockerignore` in bundled source directories
   - Whether `bundling.local` is configured in the construct props
   - Docker build output for cache hit/miss patterns

### Correlating Signals

- High time in `prepareApp` + user code has `.node.addDependency()` in a loop + source/target constructs have many CfnResources → dependency expansion is the cost
- High time in `resolveReferences` + many cross-stack Outputs with Export in templates → reference resolution cost
- High time in `synthesizeTree` + stacks with >300 resources → rendering cost
- Construction dominant + profile shows time in user's `lib/` files, not `aws-cdk-lib/` → user-code bottleneck
- Wall >> CPU + verbose log shows IMDS lookups → CLI-startup network wait
- Wall >> CPU + spawn-trace shows a high total with a tiny distinct count → redundant dependency subprocess spawns during construction
- Bundling dominant + large asset directories without `.dockerignore` → unbounded Docker build context
- Large staged asset + high `.d.ts`/source/vendored-SDK file count → deploy-hygiene problem (fix even if synth time is unaffected)

### Reading the User's Code

- Start with `cdk.json` → find the `"app"` command → find the entry point file. Note any `&& build` chaining in the app field (it's inside your measured time).
- Trace the construct tree: App → Stage(s) → Stack(s) → Constructs
- Look for patterns that scale: loops creating constructs, cross-stack references, deep nesting
- Check custom constructs or third-party libraries for expensive initialization **and synchronous child_process calls**
- Count stacks and map how they reference each other

### What to Report

Present:

1. **Measurement setup** — workspace/commit, number of runs, run-to-run variance, and exactly what the measured command includes (build chaining, yarn wrappers).
2. **Wall-clock vs CPU** — the ratio, and what it implies (compute-bound vs I/O/subprocess-bound).
3. **Top-level segment breakdown** — CLI startup / module load / construction / framework synthesis, with percentages. Lead with this.
4. **Raw data** — the CPU profile top functions and, where relevant, the child_process call grouping.
5. **Observations** — what the data shows, correlated with what you found in the code.
6. **Root causes** — the specific code patterns, dependency versions, or CLI behavior connected to the observed cost, with file paths / line numbers / versions where possible.
7. **Structural context** — resource counts, template sizes, export counts, **and staged asset sizes / file counts**.
8. **Diminishing-returns call** — if the app is already near the Node-startup floor (<~5s), say so and recommend stopping.

Connect the numbers to the code. "X ms is spent in Y because your code (or dependency Z at version V) does W here [file:line]."
