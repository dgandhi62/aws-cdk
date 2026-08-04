# CDK Synth Performance

## What This Is

Domain knowledge for investigating CDK synthesis performance. Use this when a user mentions slow synth or asks you to investigate their synth time or make it faster.

## CDK Synthesis Phases

A CDK synth executes in distinct phases. The code path is in `packages/aws-cdk-lib/core/lib/private/synthesis.ts`:

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

Bundling happens during construction when assets are staged (`AssetStaging` constructor calls Docker/zip operations).

## How to Capture a CPU Profile

```bash
NODE_OPTIONS="--cpu-prof --cpu-prof-dir=./cdk-perf --max-old-space-size=8192" npx cdk synth
```

The `.cpuprofile` file will be in `./cdk-perf/`. It's a JSON file with:
- `nodes`: call frame objects with `id`, `callFrame` (functionName, url, lineNumber), `hitCount`, `children`
- `samples`: array of node IDs (one per sample interval, ~1ms)
- `timeDeltas`: array of microsecond deltas between samples

### Reading the Profile

The profile is JSON. Read it and look at the `nodes` array — each node has a `callFrame` with `functionName` and `url` (file path). The `samples` array tells you which node was on-CPU at each ~1ms tick. The `timeDeltas` array gives the microsecond gap between ticks.

To determine where time is spent, look at which functions appear in the call stack for each sample. A sample "belongs" to a phase based on which CDK framework function is an ancestor in the call tree.

### Limitations

- CPU profiles are sampled (~1ms intervals). They give proportional time estimates, not exact call counts.
- `--cpu-prof` only flushes on graceful exit. If synth OOMs or is killed, the profile is empty. Always use `--max-old-space-size=8192`.
- `--cpu-prof` breaks non-TypeScript CDK apps (Python/Java/Go/.NET) — it corrupts the jsii kernel stdout handshake.
- I/O-blocked time (waiting for Docker, file system) shows as idle in the CPU profile. Wall-clock cost of subprocess and I/O operations is underreported.

## Function Name → Phase Mapping

When reading a CPU profile, these CDK internal functions correspond to synthesis sub-phases:

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

## Structural Observations

These can be collected from `cdk.out` without profiling:

| Observation | Relevance |
|-------------|-----------|
| Resource count per stack | Rendering cost in `_toCloudFormation` scales linearly with element count |
| Template file size | Serialization + disk write cost |
| Cross-stack exports (Outputs with Export) | Each triggers token scan in consuming stacks during `resolveReferences` |
| Structurally identical stacks (same resource types) | Same rendering work repeated N times |
| `CfnInclude` resources in templates | Each one parsed a full CloudFormation template at construction time |
| Number of bundled assets | Each spawned a subprocess |

## Investigation Process

After capturing data, investigate the app itself to connect signals to causes.

### From Data to Code

1. **Identify the dominant phase** from the CPU profile phase breakdown. This tells you where to focus.

2. **If Construction dominates:** Read the user's app entry point. Find it from `cdk.json` → `"app"` field. Trace what runs between `new App()` and `app.synth()`. Look for:
   - Loops that instantiate constructs (check iteration count)
   - `fs.readFileSync` calls in user code
   - `child_process` calls in user code (bundling happens during construction)
   - Third-party construct libraries that do initialization work
   - Context lookups that might be slow

3. **If Synthesis dominates:** Look at which sub-phase is expensive:
   - **prepareApp dominant:**
     - Time in `findTransitiveDeps` / `addResourceDependency` → search user code for `.node.addDependency()` calls. Count them. Check if inside loops. Check resource counts of source/target constructs. `.node.addDependency()` expands to N×M resource-level edges.
     - Time in `resolveReferences` / `findAllReferences` / `findTokens` → count cross-stack exports in `cdk.out` templates. Each export causes CDK to render every CfnElement in the consuming stack to find tokens.
   - **synthesizeTree dominant:** Check total resource count per stack. `_toCloudFormation` renders every CfnElement. Each element is rendered at minimum twice (once in `findAllReferences`, once in `synthesizeTree`).
   - **invokeAspects dominant:** Check how many aspects are registered and how many constructs they visit.

4. **If Bundling dominates:** Find which assets are bundled. Check:
   - `cdk.out/asset.*` directories — what's in them, how large
   - Presence of `.dockerignore` in bundled source directories
   - Whether `bundling.local` is configured in the construct props
   - Docker build output for cache hit/miss patterns

5. **If Load dominates:** Check:
   - Whether app uses `ts-node` (runtime compilation) vs pre-compiled JavaScript
   - Size and depth of `node_modules`
   - Number of top-level imports/requires in the entry chain

### Correlating Signals

Cross-reference multiple data points to confirm a hypothesis:

- High time in `prepareApp` in profile + user code has `.node.addDependency()` in a loop + source/target constructs have many CfnResources → dependency expansion is the cost
- High time in `resolveReferences` + many cross-stack Outputs with Export in templates → reference resolution cost
- High time in `synthesizeTree` + stacks with >300 resources → rendering cost
- Construction dominant + profile shows time in user's `lib/` files, not in `aws-cdk-lib/` paths → user-code bottleneck
- Bundling dominant + large asset directories without `.dockerignore` → unbounded Docker build context

### Reading the User's Code

When examining the app:

- Start with `cdk.json` → find the `"app"` command → find the entry point file
- Trace the construct tree: App → Stage(s) → Stack(s) → Constructs
- Look for patterns that scale: loops creating constructs, constructs that reference other stacks, deep nesting
- Check custom constructs or third-party libraries for expensive initialization
- Count stacks and map how they reference each other (the dependency/reference graph)
- Look for `.node.addDependency()` usage vs `stack.addDependency()`

### What to Report

Present:

1. **Raw data** — the CPU profile phase breakdown and top functions
2. **Phase breakdown** — time attributed to each phase, with percentages
3. **Observations** — what the data shows, correlated with what you found in the code
4. **Root causes** — the specific code patterns or architectural decisions connected to the observed cost, with file paths and line numbers where possible
5. **Structural context** — relevant metrics from `cdk.out` (resource counts, template sizes, export counts)

Connect the numbers to the code. "X ms is spent in Y because your code does Z here [file:line]."
