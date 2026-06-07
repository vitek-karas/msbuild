# Phase 4 follow-up: using binlogs together with ETL traces

## Goal of this pass

Use the new restore binlogs to improve the phase-3 result in the places where ETL alone was still weak:

1. line up the same project evaluations more cleanly across MT and non-MT;
2. determine whether MT is evaluating a **different semantic workload** or the **same workload more slowly**;
3. decide what binlogs can and cannot add before moving to deeper follow-up such as `/profileevaluation`.

Inputs:

- non-MT worker-trace build:
  - trace: `C:\repro\perf\msbuild-mt\restore-warm\restore-warm.etl`
  - binlog: `C:\repro\perf\msbuild-mt\restore-warm\warm-restore.binlog`
- MT build:
  - trace: `C:\repro\perf\msbuild-mt\restore-warm-mt\restore-warm-mt.etl`
  - binlog: `C:\repro\perf\msbuild-mt\restore-warm-mt\warm-restore-mt.binlog`

## How binlogs and traces work together best

At a high level, the best division of labor is:

- **ETL trace**: answers **when** the regression happens and how it behaves on the timeline
  - same-project evaluation duration
  - pass breakdown (`EvaluatePass0`-`EvaluatePass5`)
  - thread overlap
  - coordination-like signals such as `RequestThreadProc`
- **binlog**: answers **what semantic evaluation instance** we are looking at
  - which restore target entry point triggered the project
  - which global properties distinguish one evaluation from another
  - what evaluated properties and items exist in that evaluation

So binlog is best used here as a **semantic identity and parity artifact**, not as a replacement for the ETL timing analysis.

## First useful result: the same project is evaluated multiple times, and all of them are slower in MT

For the hot anchor project:

- `OrchardCore.Resources.csproj`

the binlogs show three distinct restore-time evaluations in both builds:

| Evaluation kind | non-MT duration | MT duration | Delta |
| --- | ---: | ---: | ---: |
| solution-scoped restore evaluation (no `TargetFramework`) | 125 ms | 309 ms | +184 ms |
| per-framework restore evaluation (`TargetFramework=net8.0`) | 227 ms | 534 ms | +307 ms |
| per-framework restore evaluation (`TargetFramework=net9.0`) | 216 ms | 647 ms | +431 ms |

That is useful because it shows the MT regression is **not just one anomalous evaluation instance** for this project. The same project is slower across the full restore-time evaluation set.

## Cross-project check: the same pattern holds on the other top ETL outliers

The same broad shape appears in the other heavy projects from the trace work:

| Project | non-MT evaluations (ms) | MT evaluations (ms) |
| --- | --- | --- |
| `OrchardCore.ContentFields` | 33 / 92 / 64 | 113 / 390 / 342 |
| `OrchardCore.Resources` | 125 / 227 / 216 | 309 / 534 / 647 |
| `OrchardCore.Contents` | 61 / 88 / 108 | 96 / 290 / 325 |
| `OrchardCore.Users` | 41 / 97 / 78 | 111 / 322 / 339 |
| `OrchardCore.Seo` | 30 / 77 / 51 | 85 / 209 / 177 |

So the binlog agrees with the trace result that the slowdown is **systemic across the sampled heavy projects**, not confined to one one-off project/pass coincidence.

## Second useful result: the sampled MT evaluation is semantically the same project workload

For the hot `OrchardCore.Resources.csproj` `TargetFramework=net8.0` evaluation, the binlog shows the same key evaluation identity on both sides:

- `TargetFramework=net8.0`
- `EnableDefaultItems=true`
- `EnableDefaultCompileItems=true`
- `EnableDefaultEmbeddedResourceItems=true`
- `EnableDefaultContentItems=false`
- `EnableDefaultNoneItems=false`

And for the sampled hot item families that overlapped with the ETL findings, the retrieved item sets matched in count between non-MT and MT:

| Item type | non-MT count | MT count |
| --- | ---: | ---: |
| `Compile` | 11 | 11 |
| `EmbeddedResource` | 20 | 20 |
| `ImplicitPackageReferenceVersion` | 9 | 9 |
| `KnownFrameworkReference` | 20 | 20 |
| `None` | 20 | 20 |
| `WindowsSdkSupportedTargetPlatformVersion` | 20 | 20 |

The sampled item names and property values also aligned.

That does **not** prove every last evaluation detail is identical, but it is strong evidence that the MT regression is **not explained by the project evaluating a visibly different restore graph or a different default-item/property configuration** in this sampled hot case.

## Third useful result: the restore target shape is the same

Searching the project nodes in both binlogs shows the same broad restore-time target families for `OrchardCore.Resources.csproj`, including:

- `_IsProjectRestoreSupported`
- `_GenerateRestoreProjectPathWalk`
- `_GenerateRestoreProjectPathItemsPerFramework`
- `_GenerateRestoreGraphProjectEntry`
- `_GetRestoreSettingsPerFramework`
- `_GenerateProjectRestoreGraph`
- `GetAllRuntimeIdentifiers`
- `_GenerateProjectRestoreGraphPerFramework`

So the binlogs reinforce that the MT investigation is not comparing a different restore pipeline. It is comparing the **same restore/evaluation shape** running under a different execution topology.

## What binlogs improved

The binlogs make the current picture sharper in three ways:

1. they provide a clean **evaluation-instance identity** (`TargetFramework`, restore globals);
2. they show the slowdown repeats across the **full set of restore-time evaluations** for the same project;
3. they support the reading that MT is often evaluating the **same semantic workload**, just much more slowly.

## What binlogs still do not explain

The binlogs still do **not** answer the central mechanism question:

> why do those same evaluations become slower in MT?

They do not directly explain:

- why `EvaluateCondition` gets dramatically slower;
- how much of the hot `ApplyLazyItemOperations` traffic is project-local work versus overlap;
- whether the main mechanism is lock contention, repeated recomputation, cache interference, or scheduler/handoff delay;
- or what exactly the many `RequestThreadProc` intervals mean in terms of productive work vs waiting.

That remains the reason ETL is still necessary.

## Best interpretation after combining both artifacts

The combined trace+binlog reading is now:

1. the same restore-time project evaluations are repeatedly slower in MT;
2. the sampled hot evaluations do **not** obviously differ in project semantics, key properties, or sampled item sets;
3. therefore the current best hypothesis shifts further away from “MT is evaluating different content” and further toward:
   - the same evaluator workload under heavier concurrency,
   - with significant slowdown in evaluator-side condition/item work,
   - plus coordination/overlap effects that ETL sees but binlog cannot attribute precisely.

## Best next step

The next high-value artifact is still the evaluation profiler (`/profileevaluation`):

- binlog has already helped show **which evaluation instance** is being compared;
- ETL has already shown **where the time expands**;
- `/profileevaluation` is the most promising next step for answering **which exact expressions/imports/lazy-item rules dominate the slow evaluation instance**.
