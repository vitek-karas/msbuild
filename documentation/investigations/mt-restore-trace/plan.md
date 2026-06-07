# Plan: restore-warm ETL comparison for multi-proc vs multi-threaded MSBuild

## Goal

Use the two ETL traces as evidence to explain why the **multi-threaded** restore is slower than the comparable **multi-proc** restore, with the current emphasis on the **evaluation phase** and the previously identified shared-state bottleneck candidates.

Also use this investigation to answer a stronger question: do we now have a **good enough accounting of the evaluation-phase differences** between MT and non-MT?

- If yes, produce a concise summary of:
  - where evaluation time is spent;
  - which event families dominate the phase;
  - and how MT differs from non-MT at that same-project/same-pass scope.
- If no, define a **limited** set of MSBuild improvements to close the most important holes, preferably through:
  - better EventSource coverage;
  - additional payload/correlation fields on existing events;
  - or other narrowly-scoped instrumentation changes.

Inputs:

- `C:\repro\perf\msbuild-mt\restore-warm\restore-warm.etl`
- `C:\repro\perf\msbuild-mt\restore-warm-mt\restore-warm-mt.etl`

Supporting references:

- Existing source-scan bottleneck catalog: `documentation\investigations\parallel-bottlenecks\`
- MSBuild EventSource reference from PR #13961: `documentation\specs\event-source.md`

## Trace interpretation constraints

- The two traces are **not** whole-build equivalents.
- `restore-warm.etl` is from **one worker node process** in a **multi-proc** build, so it captures only the work seen by that node.
- `restore-warm-mt.etl` is from the **single process** running the whole build in **multi-threaded** mode.
- Therefore:
  - do **not** compare whole-trace duration directly;
  - do **not** compare whole-trace event counts directly;
  - only compare work after normalizing to the **same project**, **same phase**, and ideally the same repeated operation shape.

## Investigation shape

This investigation will run in phases.

## Current status

- Phase 1: complete
- Phase 2: complete for the current two-trace corpus; results are recorded in:
  - `phase2-equivalent-scope-comparison.md`
  - `phase2-preindexed-focus-details.md`
- Phase 3: complete for the current two-trace corpus; results are recorded in:
  - `phase3-evaluation-accounting.md`
- Phase 4: started
  - initial binlog correlation results are recorded in:
    - `phase4-binlog-correlation.md`
  - deeper follow-up remains open, especially evaluation-profiler work and any targeted instrumentation that may still be needed afterward.

### Phase 1 - Trace reconnaissance

Purpose: learn what these ETLs actually contain and what evidence can support a fair phase-2 comparison.

Questions Phase 1 must answer:

1. What providers and major event families are present in each trace?
2. What `Microsoft-Build` EventSource events are visible, especially evaluation-related events such as:
   - `Evaluate`
   - `EvaluateCondition`
   - `Parse`
   - `LoadDocument`
   - `ExpandGlob`
   - `ProjectGraphConstruction`
   - SDK resolver events when they affect restore evaluation
3. What fields can be used to align equivalent work across traces?
   - project path / file
   - node or process identity
   - activity IDs / related activity IDs
   - event names and payload fields
   - relative timing windows
4. What useful detail is available for deeper bottleneck analysis?
   - start/stop pairs and durations
   - evidence of waits, fan-in, or serialized regions
   - process/thread distribution
   - file/parse/import activity
5. What limitations must phase 2 respect?
   - ETL indexing truncation
   - missing parser-specific/kernel coverage in the MCP
   - topology mismatch between traces

Outputs from Phase 1:

- a short reconnaissance summary of what is present in the traces;
- a list of fields/signals that can anchor equivalent comparisons;
- a concrete Phase 2 plan focused on the most promising evaluation-side comparison method.

### Phase 2 - Equivalent-scope comparison

Purpose: compare one or more equivalent project/phase slices between the traces and test the bottleneck theory.

Concrete next shape after Phase 1 reconnaissance:

1. Start with one or more projects that clearly appear in both traces, beginning with `OrchardCore.DisplayManagement.csproj` because both traces expose its `BuildProject` and `Evaluate*` events.
2. Use `payload.projectFile` / `payload.projectPath` as the primary alignment keys.
3. For each anchor project, extract:
   - `BuildProject/Start` and `BuildProject/Stop` when present;
   - `Evaluate/Start` and `Evaluate/Stop` when present;
   - `EvaluatePass0` through `EvaluatePass5` start/stop pairs.
4. Use the pass windows as the main comparison unit, especially `EvaluatePass1`, which already appears fully for the same project in both traces.
5. For each pass window, gather nearby supporting events that may explain expansion:
   - `CachedSdkResolverServiceResolveSdk*`
   - `OutOfProcSdkResolverServiceRequestSdkPathFromMainNode*`
   - `LoadDocument*`
   - `Parse*`
   - `ExpandGlob*`
   - `RequestThreadProc*`
6. Do **not** use activity IDs alone as the join key in MT mode. Phase 1 showed that a single related activity can fan into events for multiple concurrent projects, so phase 2 should anchor by project path first and then narrow by time window and thread when needed.
7. Compare whether MT slowdown looks most like:
   - longer evaluation passes for the same project;
   - more side work inside the pass window;
   - more overlap on shared services such as SDK resolution, parsing, or import loading;
   - or non-evaluation coordination leaking into the same interval.
8. Map any observed expansion back to the source-scan bottleneck catalog, but allow a non-catalog explanation if the trace evidence points elsewhere.

### Phase 3 - Evaluation-phase accounting and gap check

Purpose: decide whether the current trace evidence explains the evaluation-phase differences well enough to write a durable summary, and if not, identify only the smallest instrumentation improvements needed to close the gaps.

Primary inputs for this phase:

- `phase2-equivalent-scope-comparison.md`
- `phase2-preindexed-focus-details.md`

The first note provides the pass-level comparison and the cross-project timing signal. The second note provides the narrower attribution details needed to judge whether the phase-2 story is truly explained or still partly opaque.

Concrete shape:

1. Treat the current two traces as the full available trace corpus for this phase.
2. Use the phase-2 project/pass results to assemble an evaluation-phase breakdown centered on these event families:
   - `EvaluatePass0`-`EvaluatePass5`
   - `EvaluateCondition`
   - `ApplyLazyItemOperations`
   - `ExpandGlob`
   - `LoadDocument`
   - `Parse`
   - `RequestThreadProc`
   - SDK resolver work when it falls inside the same evaluation slices
3. Start from the concrete phase-2 corpus that already exists:
   - the repeated OrchardCore shared-project cohort from `phase2-equivalent-scope-comparison.md`;
   - the five highest MT outliers in `phase2-preindexed-focus-details.md`;
   - and the original `OrchardCore.DisplayManagement.csproj` anchor project for a sanity check that the summary still matches the original investigation entry point.
4. For each family above, classify the current understanding as one of:
   - **well understood**
   - **partially understood**
   - **not attributable enough**
5. Use these decision rules when assigning that classification:
   - **well understood**: the family has a repeated MT/non-MT timing signal at same-project or tightly thread-scoped comparison, and the existing evidence is strong enough to say whether the slowdown is mostly productive evaluator work, overlap, or coordination;
   - **partially understood**: the family shows a real repeated signal, but one important question still lacks clean attribution, such as project ownership inside an overlapping MT window or whether the time is productive work versus handoff/wait behavior;
   - **not attributable enough**: the family may be large, but the current event payloads or correlation surfaces do not let us assign the cost to the right project/request or explain its role with confidence.
6. If the evaluation story is good enough, write a short phase summary that answers:
   - where the time goes inside evaluation;
   - which differences are systemic vs project-sensitive;
   - which differences look like productive work vs coordination/overlap effects.
7. In that summary, explicitly answer these family-specific questions because phase 2 already suggests different levels of understanding:
   - whether `EvaluateCondition` now looks systemic enough to count as explained;
   - whether `ExpandGlob` is systemic or only a project-sensitive amplifier;
   - whether `ApplyLazyItemOperations` is attributable enough on existing evidence, or still mostly inferred from enclosed totals and item-type samples;
   - whether `RequestThreadProc` remains blocked primarily by missing event payload;
   - whether `LoadDocument`, `Parse`, and SDK resolver work can now be demoted from likely primary causes.
8. If the evaluation story is **not** good enough, write a constrained gap list that names the smallest MSBuild changes likely to close it, for example:
   - add project/request identifiers to `RequestThreadProc`
   - add missing payload fields needed for project attribution on hot evaluation events
   - add a new event around a currently opaque evaluation sub-step only when an existing event cannot be extended cleanly
9. Keep the improvement list intentionally short and evidence-driven; do not turn this into a general instrumentation wishlist.
10. Do not pull in binlogs during this phase unless the trace-only path fails to answer the classification question for a top-priority family. Binlogs remain phase-4 follow-up by default.

Outputs from Phase 3:

- a summary of where evaluation time is spent and how MT differs from non-MT, **or**
- a short list of the remaining understanding gaps plus the limited MSBuild improvements needed to close them.
- The phase deliverable should also include a compact per-family table with:
  - evidence scope used;
  - current classification;
  - why that classification is justified;
  - and, only for `partially understood` / `not attributable enough`, the smallest missing instrumentation or correlation field.

## Phase 3 execution checklist

1. Re-read `phase2-equivalent-scope-comparison.md` and extract the repeated pass-level signals, especially the cross-project `EvaluatePass3` results.
2. Re-read `phase2-preindexed-focus-details.md` and extract the best attribution evidence for:
   - `EvaluateCondition`
   - `ApplyLazyItemOperations`
   - `ExpandGlob`
   - `RequestThreadProc`
3. Build a per-family evidence table covering:
   - comparison scope used;
   - whether the signal is systemic or project-sensitive;
   - whether attribution is project-local, thread-scoped, window-scoped, or only inferred.
4. Decide whether the evaluation-phase story is already durable enough without new instrumentation.
5. If yes, write the concise phase summary.
6. If no, write the smallest evidence-driven instrumentation gap list and stop there.

### Phase 4 - Optional cross-artifact follow-up after Phase 3

Note for the next phase after phase 3: matching **binlogs** are available for the same builds as these traces.

If phase 3 still leaves attribution holes, or if the evaluation summary would benefit from richer build context, use additional artifacts together with the traces to investigate whether cross-artifact correlation can improve:

1. project/build request attribution
2. target/task/import context around hot trace windows
3. interpretation of evaluation-side coordination and scheduling effects
4. semantic attribution inside evaluation itself using the built-in evaluation profiler (`/profileevaluation`)

### Phase 4 candidate: evaluation-profiler follow-up

The built-in evaluation profiler is a promising next artifact for this investigation because it reports evaluation cost by:

- pass
- file
- line
- expression
- inclusive / exclusive time

At a high level, it could improve this investigation by answering a different class of question than ETL:

1. **what expression/import/item definition is expensive?**  
   The profiler can attribute evaluation time to specific imported files and expressions, which may help explain which concrete project logic dominates the slow `EvaluatePass3` / lazy-item regions.
2. **is the MT slowdown concentrated in the same semantic evaluator work?**  
   If the same project is profiled in MT and non-MT modes, the profiler output could show whether the wall-clock expansion is concentrated in the same expressions, imports, and lazy-item rules on both sides.
3. **can it sharpen the current weak spots?**  
   It may help refine the current partial understanding around `ApplyLazyItemOperations` and lazy/default-item work by showing which item expressions dominate, even when ETL payloads are too weak for clean project attribution.

Important limit: the profiler is **not** a replacement for ETL.

- It is strong for **semantic attribution inside one project evaluation**.
- It is weak for **threading, overlap, contention, and request coordination**.

So the best use is as a complement to the ETL results:

- ETL answers **where the wall-clock regression lives** and whether coordination/overlap exists.
- `/profileevaluation` can answer **which project expressions/imports/lazy-item rules are responsible inside that slow evaluation slice**.

## Phase 1 execution checklist

1. Open both traces in the nettrace MCP.
2. Inventory top providers and trace metadata.
3. Describe `Microsoft-Build` schema in both traces.
4. Identify evaluation-related event types and their payload fields.
5. Pull representative sample events for likely anchor operations.
6. Determine the best alignment keys for cross-trace comparison.
7. Write the reconnaissance summary and replace the placeholder Phase 2 outline with a concrete next-step plan.

## Assumptions

- The current pass is trace analysis only; no code changes are expected from this phase.
- The nettrace MCP is the primary inspection tool, but its ETL indexing limitations must be documented when they affect conclusions.
- Phase 1 should stay lightweight and produce just enough structure to make Phase 2 targeted rather than exploratory.
- Matching binlogs exist for the same builds, but they are intentionally deferred until after phase 3 unless the trace-only path proves insufficient.
