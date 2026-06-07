# Phase 2 follow-up: pre-indexed details for the focus event families

## Scope

This note follows `phase2-equivalent-scope-comparison.md` and uses the new nettrace **selective pre-index** support to pull more detailed payload data for the next-phase focus events:

1. `ApplyLazyItemOperations`
2. `EvaluateCondition`
3. `ExpandGlob`
4. `RequestThreadProc`

The traces are still:

- multi-proc trace: one worker node from `C:\repro\perf\msbuild-mt\restore-warm\restore-warm.etl`
- MT trace: the single process from `C:\repro\perf\msbuild-mt\restore-warm-mt\restore-warm-mt.etl`

## Selective index used

I created a narrow `Microsoft-Build` selective index in both traces for:

- event prefixes: `Evaluate*`, `ApplyLazyItemOperations*`, `ExpandGlob*`, `RequestThreadProc*`
- exact event names: `BuildProject/Start`, `BuildProject/Stop`
- payload fields: `projectFile`, `projectPath`, `condition`, `result`, `itemType`, `glob`, `rootDirectory`, `excludedPatterns`

Per trace, that selective view covered:

- **808,672 events**
- **939,607 indexed payload values**

So the new feature is already useful for this investigation shape.

## Most important practical limitation

The new index makes payload filtering much cheaper, but it also exposed a more specific limitation in the EventSource surface:

- `EvaluateCondition`
- `ApplyLazyItemOperations`
- `ExpandGlob`
- `RequestThreadProc`

do **not** carry `projectFile` / `projectPath` in their payloads.

That means:

1. broad pass-window queries in MT still mix work from overlapping projects;
2. the cleanest project attribution still comes from anchoring on a known pass window and then narrowing further by **thread** when possible;
3. `RequestThreadProc` remains the least attributable family because it also has **no payload fields at all**.

## Focused anchor: `OrchardCore.Resources.csproj` peak `EvaluatePass3`

I used the heaviest `EvaluatePass3` slice already identified in the prior note:

- non-MT worker: `11096.0842` -> `11370.7872` ms
- MT: `9622.0435` -> `10518.6143` ms

Then I queried the focus families inside those windows, first broadly and then with a tighter **same thread** filter:

- non-MT project thread: process `22672`, thread `20500`
- MT project thread: process `17748`, thread `20924`

## Detailed findings

### 1. `EvaluateCondition`: same condition shapes, not a different class of work

With the thread-scoped query, the early condition stream on `OrchardCore.Resources.csproj` is almost the same in both traces.

Representative conditions seen on **both** sides:

- `'$(TargetFramework)' == 'net8.0'`
- `'$(TargetFramework)' == 'net9.0'`
- `'$(EnableDefaultItems)' == 'true'`
- `'$(EnableDefaultCompileItems)' == 'true'`
- `'$(EnableDefaultEmbeddedResourceItems)' == 'true'`
- `'$(EnableDefaultContentItems)' != 'false' And '$(EnableDefaultWindowsAppSdkContentItems)' == 'true'`
- `'$(EnableDefaultPRIResourceItems)' != 'false' And '$(EnableDefaultWindowsAppSdkPRIResourceItems)' == 'true'`
- `'$(EnableDefaultItems)' == 'true' And '$(EnableDefaultNoneItems)' == 'true'`
- `'$(DisableImplicitFrameworkReferences)' != 'true' and '$(TargetFrameworkIdentifier)' == '.NETStandard' And '$(_TargetFrameworkVersionWithoutV)' < '2.1'`

So the MT slowdown does **not** look like a different kind of condition logic appearing only in MT. It looks more like **more overlapping evaluation of the same default-item / target-framework condition families**.

#### Perf summary

Thread-scoped paired-duration stats for `EvaluateCondition`:

| Metric | non-MT | MT | Delta |
| --- | ---: | ---: | ---: |
| paired condition spans | 1,343 | 1,343 | 0 |
| distinct conditions | 172 | 172 | 0 |
| total duration | 18.801 ms | 487.787 ms | +468.986 ms |
| average duration | 0.013999 ms | 0.363207 ms | +0.349208 ms |
| max duration | 0.612900 ms | 33.034600 ms | +32.421700 ms |

Top condition payloads by total paired duration:

| Condition | non-MT total | MT total | Delta |
| --- | ---: | ---: | ---: |
| `'%(Link)' == '' And '%(DefiningProjectExtension)' != '.projitems' ...` | 13.073 ms | 422.102 ms | +409.029 ms |
| `'%(LinkBase)' != ''` | 3.750 ms | 20.136 ms | +16.386 ms |
| `'@(WindowsSdkSupportedTargetPlatformVersion)' == ''` | 0.372 ms | 33.035 ms | +32.663 ms |

So the **set of conditions is effectively the same**, but the **paired durations are dramatically worse in MT**, especially for the repeated item-link metadata conditions.

### 2. `ExpandGlob`: same glob program, more concurrent execution pressure in MT

For the same `OrchardCore.Resources.csproj` thread-scoped slice, the glob shapes also line up closely between traces.

Representative globs seen on both sides for the project:

- `Assets.*`
- `Assets\**`
- `**\*.props`
- `**\*.targets`
- `Properties\**`
- `**\*`
- `**/*.resx`
- `**/*.cs`

The exclude sets are also materially the same, centered on:

- `obj\**\*.props`
- `obj\**\*.targets`
- `**\*.cshtml`
- `Properties\**\*.cs`
- `Properties\**\*.resx`

So again, MT is not doing an obviously different globbing program for this project. The evidence points more toward **the same glob families being executed under heavier multi-threaded overlap**.

#### Perf summary

Thread-scoped paired-duration stats for `ExpandGlob`:

| Metric | non-MT | MT | Delta |
| --- | ---: | ---: | ---: |
| paired glob spans | 8 | 8 | 0 |
| distinct globs | 8 | 8 | 0 |
| total duration | 228.714 ms | 323.457 ms | +94.743 ms |
| average duration | 28.589250 ms | 40.432150 ms | +11.842900 ms |
| max duration | 81.824600 ms | 131.540100 ms | +49.715500 ms |

Per-glob paired durations:

| Glob | non-MT | MT | Delta |
| --- | ---: | ---: | ---: |
| `**\*.props` | 81.825 ms | 131.540 ms | +49.715 ms |
| `Assets\**` | 37.942 ms | 55.950 ms | +18.008 ms |
| `**\*` | 20.377 ms | 58.952 ms | +38.575 ms |
| `**/*.cs` | 27.819 ms | 29.959 ms | +2.140 ms |
| `**/*.resx` | 31.278 ms | 24.054 ms | -7.224 ms |
| `**\*.targets` | 28.740 ms | 22.029 ms | -6.711 ms |
| `Assets.*` | 0.702 ms | 0.921 ms | +0.219 ms |
| `Properties\**` | 0.031 ms | 0.052 ms | +0.021 ms |

So the glob **program** is the same, but the **performance profile is not**: the MT trace is slower overall, with the biggest regressions concentrated in `**\*.props` and the broad `**\*` walk.

### 3. `ApplyLazyItemOperations`: repeated mutations are dominated by the same item types

For the thread-scoped `OrchardCore.Resources.csproj` slice, the first visible item-operation burst is dominated on both sides by:

- `WindowsSdkSupportedTargetPlatformVersion`

In the broader MT window, additional interleaved item types also show up very early, especially:

- `ImplicitPackageReferenceVersion`
- `KnownFrameworkReference`

That is useful in two ways:

1. the project-local thread still shows the **same dominant item type family** as non-MT;
2. the broad MT window clearly contains **more interleaved item-operation traffic from concurrent work** in the same process.

So the current evidence again favors **amplified overlap / fan-in** over “MT is executing a different item-operation code path.”

### 4. `RequestThreadProc`: richer counts, but still poor attribution

The new index helps here less than for the other families because `RequestThreadProc` has no payload.

Even so, the targeted window query showed a clear difference:

- non-MT `OrchardCore.Resources` peak window: **34** `RequestThreadProc` start/stop events
- MT `OrchardCore.Resources` peak window: **152** `RequestThreadProc` start/stop events

And the MT sample shows starts/stops scattered across many threads in the single process, for example thread IDs:

- `8464`
- `7004`
- `12624`
- `25932`
- `21316`
- `25164`

This strengthens the earlier suspicion that `RequestThreadProc` is part of the MT expansion story, but the current EventSource shape still does not let us tie those waits cleanly back to one project the way we can for globs or conditions.

That limitation is primarily an **event-emission limitation**, not a nettrace indexing limitation:

- `MSBuildEventSource.RequestThreadProcStart()` and `RequestThreadProcStop()` take **no arguments** and emit **no payload** in `src\Framework\MSBuildEventSource.cs`.
- `RequestBuilder.RequestThreadProc(...)` calls those methods without attaching project, request, target, or builder identifiers in `src\Build\BackEnd\Components\RequestBuilder\RequestBuilder.cs`.

So the ETL tooling cannot index payload that was never emitted. The tooling might still grow some higher-level correlation heuristics later, but the root problem is that the event itself does not carry enough context.

## Cross-project sanity check

To check whether the `EvaluateCondition` / `ExpandGlob` signal was just a one-project artifact, I repeated the same comparison on the **top five shared OrchardCore MT outliers** from `phase2-equivalent-scope-comparison.md`:

1. `OrchardCore.ContentFields.csproj`
2. `OrchardCore.Resources.csproj`
3. `OrchardCore.Contents.csproj`
4. `OrchardCore.Users.csproj`
5. `OrchardCore.Seo.csproj`

Method:

- for each project, pick the **peak `EvaluatePass3` span** in each trace;
- for `EvaluateCondition` and `ExpandGlob`, measure **thread-scoped paired spans** on that pass thread;
- for `RequestThreadProc`, measure **all enclosed paired spans inside the same pass window** because the event still has no payload to support thread-local project attribution.

### `EvaluateCondition`: clearly systemic across the sampled outliers

Across all five projects, the thread-scoped condition set stayed effectively the same, but MT was slower every time:

| Project | non-MT total | MT total | Delta | Distinct conditions |
| --- | ---: | ---: | ---: | ---: |
| `OrchardCore.ContentFields` | 32.579 ms | 397.729 ms | +365.150 ms | 182 / 182 |
| `OrchardCore.Resources` | 18.801 ms | 467.234 ms | +448.433 ms | 172 / 172 |
| `OrchardCore.Contents` | 12.148 ms | 450.692 ms | +438.544 ms | 182 / 182 |
| `OrchardCore.Users` | 20.369 ms | 144.476 ms | +124.107 ms | 182 / 182 |
| `OrchardCore.Seo` | 3.644 ms | 55.359 ms | +51.715 ms | 182 / 182 |
| **Sample total** | **87.541 ms** | **1,515.490 ms** | **+1,427.949 ms** | **same payload shape on every row** |

So `EvaluateCondition` is **not** a one-off at this point. In this sample it is a consistent MT regression signal.

### `ExpandGlob`: real, but less universal than `EvaluateCondition`

The glob program also stayed the same for the five-project sample (`8` paired globs on each side for every project), but the perf regression was more mixed:

| Project | non-MT total | MT total | Delta |
| --- | ---: | ---: | ---: |
| `OrchardCore.ContentFields` | 34.684 ms | 32.024 ms | -2.660 ms |
| `OrchardCore.Resources` | 228.714 ms | 342.968 ms | +114.254 ms |
| `OrchardCore.Contents` | 74.528 ms | 55.016 ms | -19.512 ms |
| `OrchardCore.Users` | 80.462 ms | 49.161 ms | -31.301 ms |
| `OrchardCore.Seo` | 41.051 ms | 72.244 ms | +31.193 ms |
| **Sample total** | **459.439 ms** | **551.413 ms** | **+91.974 ms** |

So `ExpandGlob` is also **not** just a one-project artifact, but the current evidence is more nuanced:

1. it regresses in some of the heaviest MT outliers (`Resources`, `Seo`);
2. it is neutral-to-better in others (`ContentFields`, `Contents`, `Users`);
3. as a result, it looks like a **secondary and project-sensitive amplifier**, not a uniformly systemic driver on the same level as `EvaluateCondition`.

### `RequestThreadProc`: yes, there is more detailed perf data, but it is still window-scoped

Even without payload, we can still pair `RequestThreadProc` start/stop events and measure their enclosed spans inside the same peak `EvaluatePass3` window:

| Project | non-MT spans / total | MT spans / total | MT avg | MT max | Distinct MT threads |
| --- | ---: | ---: | ---: | ---: | ---: |
| `OrchardCore.ContentFields` | 0 / 0.000 ms | 12 / 2,955.430 ms | 246.286 ms | 388.130 ms | 11 |
| `OrchardCore.Resources` | 11 / 886.362 ms | 42 / 9,847.442 ms | 234.463 ms | 530.112 ms | 20 |
| `OrchardCore.Contents` | 4 / 268.541 ms | 19 / 4,950.416 ms | 260.548 ms | 415.064 ms | 17 |
| `OrchardCore.Users` | 3 / 213.196 ms | 6 / 1,326.355 ms | 221.059 ms | 269.791 ms | 6 |
| `OrchardCore.Seo` | 0 / 0.000 ms | 0 / 0.000 ms | 0.000 ms | 0.000 ms | 0 |
| **Sample total** | **18 / 1,368.099 ms** | **79 / 19,079.643 ms** | **241.515 ms** | **530.112 ms** | **up to 20** |

That means we **do** have quantitative `RequestThreadProc` evidence beyond raw counts:

1. enclosed `RequestThreadProc` work is dramatically larger in the MT peak windows for `ContentFields`, `Resources`, `Contents`, and `Users`;
2. the MT windows also fan across many more threads;
3. `Seo` simply had no fully enclosed `RequestThreadProc` spans in the chosen peak pass window, so it is a “no signal in this window” case, not evidence against the broader pattern.

The important caveat is unchanged: these `RequestThreadProc` numbers are still **window-scoped coordination cost**, not perfectly project-attributed cost, because the event payload does not identify the project/request directly.

## What the new pre-indexing capability was good for

The new selective index already made several useful things easier:

1. fast payload filtering on `condition`, `itemType`, `glob`, and `rootDirectory`
2. narrow thread-scoped queries inside a heavy pass window
3. practical confirmation that the MT-heavy slices are mostly doing the **same kinds of condition/glob/item work**, not a clearly different semantic workload

## What this changes in the hypothesis

This follow-up makes the current hypothesis more specific:

1. The hot MT `EvaluatePass3` slices are still dominated by:
   - `EvaluateCondition`
   - `ApplyLazyItemOperations`
   - `ExpandGlob`
   - `RequestThreadProc`
2. But for at least the `OrchardCore.Resources.csproj` anchor, the **condition text**, **glob shapes**, and **dominant item types** are substantially the same between non-MT and MT.
3. That makes the slowdown look less like “MT runs a different evaluator program” and more like:
   - **the same evaluator work under higher concurrency**
   - **more same-process interleaving**
   - and likely **more coordination / wakeup / scheduling overhead**, especially around `RequestThreadProc`

## Best next step

The strongest next-source-scan target is now:

1. how `EvaluatePass3` work fans across threads in MT,
2. how much `RequestThreadProc` is pure coordination overhead versus productive work handoff,
3. and whether the `LazyItemEvaluator` / default-item / glob path has a shared-state or handoff pattern that gets amplified when many projects hit the same evaluation stage together.
