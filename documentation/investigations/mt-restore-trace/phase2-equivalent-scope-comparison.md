# Phase 2 equivalent-scope comparison: evaluation slices

## Scope

This pass compares the two ETLs only at the **same project** and **same evaluation pass** level.

- multi-proc trace: one worker node from `C:\repro\perf\msbuild-mt\restore-warm\restore-warm.etl`
- MT trace: the single process from `C:\repro\perf\msbuild-mt\restore-warm-mt\restore-warm-mt.etl`

Whole-trace totals are still **not** comparable because the topologies differ.

## Trace status update

The current nettrace indexes are now **complete**, so the phase-1 truncation warning is no longer the active limit for these two files:

- `restore-warm.etl`: **18,234,141** indexed events
- `restore-warm-mt.etl`: **20,713,663** indexed events

The remaining important caveat is still provider coverage: ETL support indexes what is surfaced through the TraceEvent dynamic dispatcher, so some parser-specific or kernel/classic ETW events may still be absent.

## Reliable comparison unit

The trace evidence continues to support the phase-1 rule that the safe join key is:

1. project path
2. pass type (`EvaluatePass0`-`EvaluatePass5`)
3. local time window

Raw activity IDs and simple occurrence order are not reliable enough by themselves in the MT trace, especially once several projects overlap.

## Cross-project pass timing signal

Across the repeated shared OrchardCore `.csproj` cohort, the average pass durations point to **`EvaluatePass3`** as the dominant MT expansion, with **`EvaluatePass1`** as the secondary signal.

| Pass | non-MT avg | MT avg | Delta |
| --- | ---: | ---: | ---: |
| `EvaluatePass0` | 0.294 ms | 0.524 ms | +0.231 ms |
| `EvaluatePass1` | 26.846 ms | 66.841 ms | +39.995 ms |
| `EvaluatePass2` | 0.056 ms | 0.095 ms | +0.039 ms |
| `EvaluatePass3` | 39.834 ms | 106.241 ms | +66.406 ms |
| `EvaluatePass4` | 1.133 ms | 4.275 ms | +3.142 ms |
| `EvaluatePass5` | 2.705 ms | 4.824 ms | +2.119 ms |

That changes the emphasis from the phase-1 reconnaissance note: with the full indexes available, the strongest repeated slowdown is **not only in `EvaluatePass1`**.

## Highest-expansion projects in the sampled cohort

The largest same-project average evaluation deltas I found were:

| Project | non-MT `Evaluate` avg | MT `Evaluate` avg | Delta | Dominant pass delta |
| --- | ---: | ---: | ---: | ---: |
| `OrchardCore.ContentFields.csproj` | 83.127 ms | 535.786 ms | +452.659 ms | `EvaluatePass3` +402.630 ms |
| `OrchardCore.Resources.csproj` | 251.921 ms | 667.436 ms | +415.515 ms | `EvaluatePass3` +669.788 ms |
| `OrchardCore.Contents.csproj` | 106.558 ms | 514.533 ms | +407.975 ms | `EvaluatePass3` +345.954 ms |
| `OrchardCore.Users.csproj` | 144.874 ms | 454.613 ms | +309.739 ms | `EvaluatePass3` +131.048 ms |
| `OrchardCore.Seo.csproj` | 61.503 ms | 333.605 ms | +272.102 ms | `EvaluatePass3` +103.156 ms |

The recurring pattern is that the worst MT outliers are usually **`EvaluatePass3`-heavy**, even when `EvaluatePass1` is also slower.

## Anchor project: `OrchardCore.DisplayManagement.csproj`

For the original anchor project from phase 1:

- project: `C:\Users\alinama\work\testRepos\OrchardCore\src\OrchardCore\OrchardCore.DisplayManagement\OrchardCore.DisplayManagement.csproj`
- strongest comparable slice found in the full traces: **`EvaluatePass3`**

Peak `EvaluatePass3` slice:

| Trace | Duration |
| --- | ---: |
| non-MT | 108.828 ms |
| MT | 156.493 ms |
| delta | +47.665 ms |

Inside that pass window, the clearest MT expansion is **condition evaluation**, not document parsing/import loading:

| Inner event family | non-MT enclosed total | MT enclosed total |
| --- | ---: | ---: |
| `EvaluateCondition` | 48.183 ms | 197.683 ms |
| `ApplyLazyItemOperations` | 471.597 ms | 403.408 ms |
| `ExpandGlob` | 301.616 ms | 119.288 ms |
| `LoadDocument` | 42.503 ms | 7.130 ms |
| `Parse` | 40.268 ms | 0.588 ms |

So for this anchor, the full-trace view weakens the earlier “pass-1 import loading” suspicion. The slice that actually expands most is later and looks more like **evaluator-side condition/item work**.

## Two heavier representative MT outliers

The next two representative projects show the same `EvaluatePass3` skew, but much more strongly.

### `OrchardCore.ContentFields.csproj`

Peak `EvaluatePass3` slice:

- non-MT: **85.463 ms**
- MT: **510.146 ms**
- delta: **+424.683 ms**

Inside that pass window, the MT trace shows much heavier evaluator-side activity:

| Inner event family | non-MT | MT |
| --- | ---: | ---: |
| `ApplyLazyItemOperations` count / enclosed total | 1,767 / 140.418 ms | 11,593 / 3,604.353 ms |
| `EvaluateCondition` count / enclosed total | 1,797 / 51.730 ms | 7,960 / 2,831.456 ms |
| `ExpandGlob` count / enclosed total | 22 / 93.414 ms | 130 / 765.591 ms |
| `RequestThreadProc` count / enclosed total | 0 / 0 ms | 12 / 2,955.430 ms |

### `OrchardCore.Resources.csproj`

Peak `EvaluatePass3` slice:

- non-MT: **274.703 ms**
- MT: **913.337 ms**
- delta: **+638.634 ms**

The same shape appears, only larger:

| Inner event family | non-MT | MT |
| --- | ---: | ---: |
| `ApplyLazyItemOperations` count / enclosed total | 6,150 / 785.221 ms | 21,128 / 5,051.906 ms |
| `EvaluateCondition` count / enclosed total | 5,154 / 83.937 ms | 14,211 / 2,597.327 ms |
| `ExpandGlob` count / enclosed total | 71 / 489.793 ms | 276 / 1,743.097 ms |
| `RequestThreadProc` count / enclosed total | 11 / 886.362 ms | 42 / 9,847.442 ms |

## Interpretation

The strongest phase-2 trace signal is now:

1. The MT slowdown is real at the **same-project evaluation-slice** level.
2. The biggest repeated expansion is usually **`EvaluatePass3`**, with **`EvaluatePass1`** still slower but less dominant.
3. The heaviest MT slices are associated much more with:
   - `EvaluateCondition`
   - `ApplyLazyItemOperations`
   - `ExpandGlob`
   - `RequestThreadProc`
4. The heaviest slices are **not** primarily explained by:
   - `LoadDocument`
   - `Parse`
   - obvious SDK resolver duration

That shifts the next-source-scan priority away from “document loading / parsing cache misses” and toward **evaluation-time item/condition/glob work plus request-thread coordination inside the MT process**.

## Important reading note on enclosed totals

The “enclosed total” numbers above are sums of all start/stop spans fully contained in the chosen pass window. They can exceed the pass wall-clock duration because many inner spans overlap on different threads or nest inside one another. They are useful as a **shape/composition** signal, not as a replacement for the pass duration itself.

## Next phase recommendation

The next pass should center on **`EvaluatePass3`**, not just `EvaluatePass1`, and should map these trace-heavy families back to source:

1. `ApplyLazyItemOperations`
2. `EvaluateCondition`
3. `ExpandGlob`
4. `RequestThreadProc`

If one source bottleneck is going to explain the largest MT evaluation regressions, the current trace evidence says it is more likely to live in that cluster than in parse/import loading.
