# CDK Synth Performance

## What This Is

Domain knowledge for investigating CDK synthesis performance. Use this when a user mentions slow synth or asks you to investigate their synth time or make it faster.

## CDK Synthesis Phases

A CDK synth executes in distinct phases:

| Phase | What happens |
|-------|-------------|
| **Load** | Node requires modules, compiles TypeScript |
| **Construction** | User code runs — constructors, loops, file reads, all code before `app.synth()` |
| **Synthesis** | Framework resolves tokens, reifies dependencies, renders CloudFormation templates |
| **Bundling** | Docker builds, zip packaging for Lambda/assets |

## How to Capture Data

### Perf Counters (all languages)

```bash
CDK_PERF_COUNTERS_FILE=./perf-counters.json npx cdk synth
```

Output format:
```json
{
  "counters": {
    "phase:Load": 3200,
    "phase:Load(cnt)": 1,
    "phase:Construction": 52100,
    "phase:Construction(cnt)": 1,
    "phase:Synthesis": 39000,
    "phase:Synthesis(cnt)": 1,
    "Stack.resolve": 28400,
    "Stack.resolve(cnt)": 30600,
    "fs.readFileSync": 1200,
    "fs.readFileSync(cnt)": 847,
    "child_process.execSync": 4200,
    "child_process.execSync(cnt)": 3,
    "bundle:<source>": 8200,
    "bundle:<source>(cnt)": 1,
    "validateTemplates": 1100,
    "validateTemplates(cnt)": 1
  }
}
```

Values without `(cnt)` suffix are milliseconds. Values with `(cnt)` are call counts.

Currently instrumented operations:
- `phase:Load`, `phase:Construction`, `phase:Synthesis` — coarse phases
- `Stack.resolve` — token resolution
- `fs.*` — file I/O (readFileSync, writeFileSync, statSync, etc.)
- `child_process.*` — subprocess calls
- `bundle:<source>` — per-asset bundling
- `validateTemplates` — template validation
- `BundlingDockerImage.run`, `DockerImage.fromBuild` — Docker operations
- `AssetBundlingBindMount.run`, `AssetBundlingVolumeCopy.run` — asset bundling methods

### CPU Profile (TypeScript apps only)

```bash
NODE_OPTIONS="--cpu-prof --cpu-prof-dir=./cdk-perf --max-old-space-size=8192" npx cdk synth
```

Parse top functions:

```bash
node -e "
const p = JSON.parse(require('fs').readFileSync(process.argv[1],'utf-8'));
const m = new Map(p.nodes.map(n=>[n.id,n]));
const h = new Map();
for (const id of p.samples) {
  const n = m.get(id), k = (n.callFrame.functionName||'(anon)') + ' @ ' + (n.callFrame.url||'').split('/').slice(-2).join('/');
  h.set(k, (h.get(k)||0)+1);
}
const tot = p.samples.length, ms = p.timeDeltas.reduce((a,b)=>a+b,0)/1000;
console.log('Total: '+(ms/1000).toFixed(1)+'s');
[...h.entries()].sort((a,b)=>b[1]-a[1]).slice(0,25).forEach(([k,c])=>console.log((c/tot*100).toFixed(1).padStart(6)+'%  '+(c/tot*ms|0)+'ms  '+k));
" ./cdk-perf/*.cpuprofile
```

**Limitation:** CPU profiles are sampled (~1ms intervals). They estimate proportional time but do not provide exact call counts.

**Limitation:** `--cpu-prof` only flushes on graceful exit. If synth OOMs or is killed, the profile is empty.

### Stack Metrics (from cdk.out)

```bash
# Resource count per stack
for f in cdk.out/*.template.json; do echo "$(node -e "console.log(Object.keys(JSON.parse(require('fs').readFileSync('$f','utf-8')).Resources||{}).length)") $f"; done | sort -rn

# Template sizes
ls -lhS cdk.out/*.template.json | head -10

# Cross-stack exports
grep -c '"Export"' cdk.out/*.template.json 2>/dev/null | grep -v ':0$'

# Structural hash (find stacks with identical resource type compositions)
for f in cdk.out/*.template.json; do
  node -e "
    const t = JSON.parse(require('fs').readFileSync('$f','utf-8'));
    const types = Object.values(t.Resources||{}).map(r=>r.Type).sort().join(',');
    console.log(require('crypto').createHash('md5').update(types).digest('hex').slice(0,8) + ' $f');
  "
done | sort | uniq -c -w8 | sort -rn
```

## Function Name → Phase Mapping

When reading a CPU profile, these CDK internal functions correspond to synthesis sub-phases:

| Function | Phase |
|----------|-------|
| `synthesize` | Synthesis entry point |
| `synthNestedAssemblies` | Recursive synth of nested stages |
| `invokeAspects` / `invokeAspectsV2` | Aspect invocation |
| `prepareApp` | Prepare (resolves refs + reifies deps) |
| `resolveReferences` / `findTokens` | Cross-stack reference resolution |
| `reifyConstructDependencies` | Expanding construct deps to resource-level deps |
| `_addAssemblyDependency` / `operateOnDependency` | Individual dependency edge creation |
| `validateTree` | Running construct validators |
| `synthesizeTree` / `_toCloudFormation` | Rendering stacks to CloudFormation JSON |
| `buildAssembly` | Writing cloud assembly manifest |
| `Stack.resolve` / `resolve` | Token resolution |
| `BundlingDockerImage.run` / `DockerImage.fromBuild` | Docker asset bundling |

## Known Patterns

These are patterns observed in real CDK apps. When you see the signal, investigate the cause. The appropriate action depends on the user's architecture and constraints.

### Dependency Explosion

**Signal:** `_addAssemblyDependency` or `operateOnDependency` dominates the profile.

**Mechanism:** `.node.addDependency(otherConstruct)` expands to N×M resource-level dependency edges. The framework calls `_addAssemblyDependency` once per edge. 50 resources depending on 50 resources = 2,500 calls.

**What to look for:** Search the user's code for `.node.addDependency()` calls, particularly inside loops or applied broadly across constructs. Compare with `stack.addDependency()` which creates a single stack-level edge.

### Cross-Stack Reference Scan

**Signal:** `resolveReferences` / `findTokens` takes significant time. High `Stack.resolve` count.

**Mechanism:** Each cross-stack reference (using a value from one stack in another) forces CDK to render the producing stack's template to locate the token, then render the consuming stack to inject the import. More exports = more full-template renders.

**What to look for:** Count cross-stack exports in `cdk.out` templates. Examine how stacks reference each other's resources.

### Construction-Dominant

**Signal:** `phase:Construction` is the majority of total time. CPU profile shows time in user code files, not CDK framework files.

**Mechanism:** All code between `new App()` and `app.synth()` runs during construction. This includes any I/O, API calls, or computation the user performs.

**What to look for:** Examine user construct code for synchronous operations — file reads, HTTP calls, expensive computations, large iteration counts.

### Bundling-Dominant

**Signal:** `child_process.*` high in counters. `bundle:*` spans account for significant time.

**Mechanism:** Each bundled asset (Lambda, Docker image) spawns a subprocess. Docker builds without cache hit rebuild from scratch.

**What to look for:** Number of bundled assets. Docker build contexts (presence/absence of `.dockerignore`). Whether `bundling.local` is configured.

### File System Load

**Signal:** High `fs.readFileSync` count and time in counters.

**Mechanism:** `CfnInclude` reads and parses a CloudFormation template per instantiation. Asset fingerprinting reads files to compute hashes. Module resolution reads `package.json` files.

**What to look for:** `CfnInclude` usage count. Size of asset directories being fingerprinted. Number of packages in `node_modules`.

### Template Rendering Cost

**Signal:** `_toCloudFormation` / `synthesizeTree` takes significant time relative to resource count.

**Mechanism:** CDK renders templates during reference resolution (to find tokens) AND during final synthesis. Cross-stack references and nested stacks can multiply renders.

**What to look for:** Number of resources per stack. Template sizes. Whether `_toCloudFormation` call count exceeds stack count (indicates multiple renders).

### Token Resolution Volume

**Signal:** `Stack.resolve` has very high call count (visible in perf counters).

**Mechanism:** Every token in every resource property must be resolved during synthesis. More resources × more properties × more tokens = more resolve calls.

**What to look for:** Total resource count.

## Structural Observations

These can be collected from `cdk.out` without profiling:

| Observation | What it tells you |
|-------------|-------------------|
| Resource count per stack | Rendering cost scales with resource count |
| Template file size | Large templates take longer to render and serialize |
| Cross-stack export count | Each export increases reference resolution work |
| Number of structurally identical stacks (same resource types) | Repeated work pattern — same template rendered N times |
| Number of `CfnInclude` resources | Each one parses a full CloudFormation template at construction time |
| Number of bundled assets | Each spawns a subprocess (usually Docker) |

## Investigation Process

After capturing data, investigate the app itself to connect signals to causes.

### From Data to Code

1. **Identify the dominant phase** from perf counters (`phase:Load`, `phase:Construction`, `phase:Synthesis`). This tells you where to look.

2. **If Construction dominates:** Read the user's app entry point (usually `bin/*.ts` or `lib/*.ts`). Trace what runs between `new App()` and `app.synth()`. Look for:
   - Loops that instantiate constructs
   - File reads (`fs.readFileSync` calls in user code)
   - Any synchronous I/O or computation
   - Third-party construct libraries that do work in their constructors

3. **If Synthesis dominates:** Look at the profile for which sub-function is expensive, then trace into the app's architecture:
   - High `_addAssemblyDependency` → search user code for `.node.addDependency(` calls. Count them. Check if they're in loops.
   - High `resolveReferences` / `findTokens` → count cross-stack references. Look at how stacks pass values to each other (`stack.exportValue`, referencing `otherStack.resource.attr`).
   - High `_toCloudFormation` → check if call count exceeds stack count (indicates re-rendering). Look at nested stacks and cross-stack ref patterns.
   - High `Stack.resolve` count → look at total resource count and CDK version.

4. **If Bundling dominates:** Find which assets are bundled. Check:
   - `cdk.out/asset.*` directories — what's in them, how large
   - Presence of `.dockerignore` in bundled source directories
   - Whether `bundling.local` is configured in the construct props
   - Docker build output for cache hit/miss patterns

5. **If Load dominates:** Check:
   - `node_modules` size and depth
   - Whether the app uses `ts-node` (compiles on the fly) vs pre-compiled
   - Number of `require`/`import` statements in the entry point chain

### Correlating Signals

Cross-reference multiple data points to confirm a hypothesis:

- High `_addAssemblyDependency` in profile + user code has `.node.addDependency()` in a loop → confirmed dependency explosion
- High `fs.readFileSync` count + user code has multiple `CfnInclude` instantiations → confirmed template parsing overhead
- High `child_process.execSync` time + large asset directories without `.dockerignore` → confirmed unbounded Docker context
- High `Stack.resolve` count + many cross-stack exports in templates + high `resolveReferences` time → confirmed reference resolution overhead
- `phase:Construction` dominant + profile shows time in user's `lib/` files, not `node_modules/aws-cdk-lib/` → confirmed user-code bottleneck

### Reading the User's Code

When examining the app:

- Start with `cdk.json` → find the `"app"` command → find the entry point
- Trace the construct tree: App → Stage(s) → Stack(s) → Constructs
- Look for patterns that scale poorly: loops creating constructs, constructs that reference other stacks, deep nesting
- Check if custom constructs or third-party libraries do expensive initialization
- Look at how many stacks exist and how they reference each other (the dependency graph)

### What to Report

Present:

1. **Raw data** — the actual counters and/or profile top functions
2. **Phase breakdown** — time attributed to each phase
3. **Observations** — what the data shows, correlated with what you found in the code
4. **Root causes** — the specific code patterns or architectural decisions connected to the observed cost
5. **Context** — relevant structural metrics from `cdk.out`

Connect the numbers to the code. "X ms is spent in Y because your code does Z here [file:line]."
