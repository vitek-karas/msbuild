# Phase 1 reconnaissance: restore-warm vs restore-warm-mt

## Scope

This pass only inventories what the two ETLs expose and defines the most credible path for the next comparison phase. It does **not** try to explain the full slowdown yet.

Trace inputs:

- multi-proc single worker node: `C:\repro\perf\msbuild-mt\restore-warm\restore-warm.etl`
- multi-threaded single process: `C:\repro\perf\msbuild-mt\restore-warm-mt\restore-warm-mt.etl`

## What is present in the traces

Both traces can now be opened through the nettrace MCP as ETL inputs.

Important limitations visible immediately:

- both traces were truncated after **1,000,000 indexed events** by the MCP;
- ETL support warns that parser-specific or classic/kernel ETW events may be missing from the index;
- the two traces represent different topologies, so whole-trace totals are not comparable.

Even with those limits, both traces expose enough `Microsoft-Build` EventSource data to support an evaluation-focused comparison.

## `Microsoft-Build` events confirmed in both traces

Both traces expose these useful evaluation-adjacent event families:

- `BuildProject`
- `Evaluate`
- `EvaluatePass0` through `EvaluatePass5`
- `EvaluateCondition`
- `LoadDocument`
- `Parse`
- `ExpandGlob`
- `CachedSdkResolverServiceResolveSdk`
- `RequestThreadProc`

The non-MT worker trace also exposes:

- `OutOfProcSdkResolverServiceRequestSdkPathFromMainNode`

That is an important topology-specific difference: the multi-proc worker may show the request to the main node rather than the same local resolver work shape that appears in MT mode.

## Best alignment keys found

### Strong anchors

1. `payload.projectFile` on `Evaluate*` and `EvaluatePass*`
2. `payload.projectPath` on `BuildProject*` and several SDK resolver events
3. event time windows within the same project slice
4. thread ID as a secondary narrowing key inside a known project slice

### Weak or unsafe anchors

- whole-trace counts or totals
- activity IDs by themselves in MT mode

The activity ID structure is useful for local context, but it is **not** reliable enough to join project-specific work by itself in the MT trace. During reconnaissance, querying by a pass activity's `relatedActivityId` in MT returned child events from **multiple different projects**. That means phase 2 must anchor by project path first, then use time/thread narrowing where needed.

## Common project anchor already verified

`C:\Users\alinama\work\testRepos\OrchardCore\src\OrchardCore\OrchardCore.DisplayManagement\OrchardCore.DisplayManagement.csproj`

appears in both traces with:

- `BuildProject/Start`
- `Evaluate/Start`
- `EvaluatePass0`
- `EvaluatePass1`
- follow-on SDK resolver events

This makes it a strong first candidate for the equivalent-scope comparison.

## Preliminary timing signal

For `OrchardCore.DisplayManagement.csproj`, both traces expose a full `EvaluatePass1` interval:

- non-MT worker trace: about **138 ms**
- MT trace: about **234 ms**

This is only a reconnaissance-level signal, not a conclusion, but it shows that a same-project, same-pass comparison is feasible and may already reveal where the MT mode expands.

## What detailed information is likely useful in phase 2

1. Per-project timings for `Evaluate` and `EvaluatePass0`-`EvaluatePass5`
2. Supporting event counts and durations within each pass window:
   - SDK resolver work
   - document loads
   - parses
   - glob expansion
3. Thread usage inside the per-project evaluation window
4. Whether the MT trace shows additional side work inside the same pass interval that the worker trace does not
5. Whether the trace evidence matches one of the source-scan candidates:
   - `ProjectRootElementCache`
   - `ConditionEvaluator`
   - SDK resolution funnel
   - other evaluation/import reuse bottlenecks

## Phase 2 plan

1. Start with `OrchardCore.DisplayManagement.csproj` and one or two additional common projects.
2. Extract the per-project `Evaluate*` and `EvaluatePass*` intervals in both traces.
3. Compare pass durations directly at the project slice level, not the whole trace level.
4. For the slowest-expanded MT pass, gather nearby `LoadDocument`, `Parse`, `ExpandGlob`, resolver, and `RequestThreadProc` events in that pass window.
5. Decide whether the slowdown looks like:
   - extra evaluator work,
   - shared-service contention,
   - topology-specific resolver behavior,
   - or some other cause not predicted by the source scan.

## Bottom line from phase 1

The traces contain enough MSBuild EventSource data to support a fair project-scoped evaluation comparison, but the comparison has to be driven by **project path + pass window**, not by whole-trace totals and not by raw activity IDs alone.
