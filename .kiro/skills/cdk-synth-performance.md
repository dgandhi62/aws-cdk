# CDK Synth Performance

## What This Is

Domain knowledge for investigating CDK synthesis performance. Use this when a user mentions slow synth or asks you to investigate their synth time or make it faster.

## CDK Synthesis Phases

A CDK synth executes in distinct phases. The code path is in `packages/aws-cdk-lib/core/lib/private/synthesis.ts`:

```
App constructor            → marks start time (performance.now())
  ... user code ...        → "Construction" phase (everything until app.synth())
App.synth()                → measures "phase:Construction", then calls synthesize()
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
                           → measures "phase:Synthesis"
```

Bundling happens during construction when assets are staged (`AssetStaging` constructor calls Docker/zip operations).

## How to Capture Data

### Perf Counters

```bash
CDK_PERF_COUNTERS_FILE=./perf-counters.json npx cdk synth
```

**Important limitation:** The counters file is only written if the per-stack synth time exceeds a threshold (default: 10,000ms per stack). This is controlled by:
- `performanceReporting` App prop (default: `true`)  
- Context key `@aws-cdk/core.slowSynthThreshold` (default: `10000`)

To force counters emission for fast apps, set the threshold to 0 in `cdk.json`:
```json
{
  "context": {
    "@aws-cdk/core.slowSynthThreshold": 0
  }
}
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
    "bundle:lambda/handler": 8200,
    "bundle:lambda/handler(cnt)": 1,
    "validateTemplates": 1100,
    "validateTemplates(cnt)": 1
  }
}
```

Values without `(cnt)` suffix are milliseconds. Values with `(cnt)` are invocation counts.

Currently instrumented: `phase:Load`, `phase:Construction`, `phase:Synthesis`, `Stack.resolve`, `fs.*`, `child_process.*`, `bundle:<source>`, `validateTemplates`, `BundlingDockerImage.run`, `DockerImage.fromBuild`, `AssetBundlingBindMount.run`, `AssetBundlingVolumeCopy.run`.

### CPU Profile (TypeScript apps only)

```bash
NODE_OPTIONS="--cpu-prof --cpu-prof-dir=./cdk-perf --max-old-space-size=8192" npx cdk synth
```

Parse top functions by self-time (where the CPU is actually spending time):

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

**Limitations:**
- CPU profiles are sampled (~1ms intervals). They give proportional time estimates, not exact call counts.
- `--cpu-prof` only flushes on graceful exit. If synth OOMs or is killed, the profile is empty. Always use `--max-old-space-size=8192`.
- `--cpu-prof` breaks non-TypeScript CDK apps (Python/Java/Go/.NET) — it corrupts the jsii kernel stdout handshake.
- I/O-blocked time (waiting for Docker, file system) shows as idle — the CPU profile underreports wall-clock cost of subprocess and I/O operations.

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

When reading a CPU profile, these CDK internal functions correspond to synthesis sub-phases. Verified against the codebase:

| Function | File | What it does |
|----------|------|-------------|
| `synthesize` | `core/lib/private/synthesis.ts` | Top-level synthesis orchestrator |
| `synthNestedAssemblies` | `core/lib/private/synthesis.ts` | Calls `synth()` on nested Stage constructs |
| `invokeAspects` | `core/lib/private/synthesis.ts` | Runs registered aspects on all constructs |
| `invokeAspectsV2` | `core/lib/private/synthesis.ts` | Aspects with stabilization loop (if feature flag enabled) |
| `prepareApp` | `core/lib/private/prepare-app.ts` | Reifies construct deps to resource deps + resolves cross-stack refs |
| `findTransitiveDeps` | `core/lib/private/prepare-app.ts` | Collects all `.node.dependencies` across the tree |
| `resolveReferences` | `core/lib/private/refs.ts` | Finds and resolves cross-stack token references |
| `findAllReferences` | `core/lib/private/refs.ts` | Iterates all CfnElements, calls `findTokens` on each |
| `findTokens` | `core/lib/private/resolve.ts` | Renders a CfnElement via `_toCloudFormation()` to discover tokens |
| `operateOnDependency` | `core/lib/deps.ts` | Adds/removes a dependency edge between two elements |
| `_addAssemblyDependency` | `core/lib/stack.ts` | Records a stack-to-stack dependency (when no common ancestor stack) |
| `addResourceDependency` | called in `prepare-app.ts` loop | Adds CloudFormation `DependsOn` between resources |
| `validateTree` | `core/lib/private/synthesis.ts` | Calls `.node.validate()` on every construct |
| `synthesizeTree` | `core/lib/private/synthesis.ts` | Visits all constructs, calls `stack.synthesizer.synthesize()` per stack |
| `_toCloudFormation` | `core/lib/stack.ts` | Renders all CfnElements in a stack to CloudFormation template JSON |
| `buildAssembly` | `cloud-assembly-api` package | Writes the cloud assembly manifest to disk |
| `validateTemplates` | `core/lib/private/synthesis-validation.ts` | Runs policy validation plugins against rendered templates |
| `BundlingDockerImage.run` | `core/lib/bundling.ts` | Executes Docker container for asset bundling |
| `DockerImage.fromBuild` | `core/lib/bundling.ts` | Builds a Docker image from Dockerfile |

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

1. **Identify the dominant phase** from perf counters. Look at `phase:Load`, `phase:Construction`, `phase:Synthesis` values. This tells you where to focus.

2. **If Construction dominates:** Read the user's app entry point. Find it from `cdk.json` → `"app"` field. Trace what runs between `new App()` and `app.synth()`. Look for:
   - Loops that instantiate constructs (check iteration count)
   - `fs.readFileSync` calls in user code (not framework)
   - `child_process` calls in user code (bundling happens during construction)
   - Third-party construct libraries that do initialization work
   - Context lookups that might be slow

3. **If Synthesis dominates:** The perf counters don't break down synthesis sub-phases. Use the CPU profile to identify which sub-function is expensive:
   - Time in `prepareApp` / `findTransitiveDeps` / `addResourceDependency` → search user code for `.node.addDependency()` calls. Count them. Check if inside loops. Check resource counts of source/target constructs.
   - Time in `resolveReferences` / `findAllReferences` / `findTokens` → count cross-stack exports in `cdk.out` templates. Look at how stacks reference each other's attributes.
   - Time in `synthesizeTree` / `_toCloudFormation` → check total resource count per stack. Large stacks render slowly.
   - Time in `Stack.resolve` → the `Stack.resolve(cnt)` counter gives exact outermost call count. High count = many tokens being resolved.

4. **If Bundling dominates:** Check `child_process.*` and `bundle:*` counters. Find which assets:
   - Look at `cdk.out/asset.*` directories
   - Check for `.dockerignore` in bundled source directories
   - Check if `bundling.local` is configured in construct props
   - Check Docker layer cache behavior (run Docker build manually to see)

5. **If Load dominates:** Check:
   - Whether app uses `ts-node` (runtime compilation) vs pre-compiled JavaScript
   - Size and depth of `node_modules`
   - Number of top-level imports/requires in the entry chain

### Correlating Signals

Cross-reference data points to confirm a hypothesis:

- High time in `prepareApp` in profile + user code has `.node.addDependency()` in a loop + source/target constructs have many CfnResources → dependency expansion is the cost
- High `fs.readFileSync(cnt)` in counters + user code has multiple `CfnInclude` instantiations → template parsing overhead
- High `child_process.execSync` time in counters + large asset source directories without `.dockerignore` → unbounded Docker build context
- High `Stack.resolve(cnt)` + many cross-stack Outputs with Export in templates + high time in `resolveReferences` in profile → reference resolution cost
- `phase:Construction` dominant + profile shows time in user's `lib/` files, not in `aws-cdk-lib/` paths → user-code bottleneck

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

1. **Raw data** — the actual perf counter values and/or CPU profile top functions
2. **Phase breakdown** — time attributed to each phase, with percentages
3. **Observations** — what the data shows, correlated with what you found in the code
4. **Root causes** — the specific code patterns or architectural decisions connected to the observed cost, with file paths and line numbers where possible
5. **Structural context** — relevant metrics from `cdk.out` (resource counts, template sizes, export counts)

Connect the numbers to the code. "X ms is spent in Y because your code does Z here [file:line]."
