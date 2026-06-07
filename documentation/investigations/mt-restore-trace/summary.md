# MT restore trace investigation: short summary

This file keeps the **shortest useful summary** of the most important findings so far from the parallel-bottleneck trace work.

For the current two-trace corpus, **phase 3 is now complete**. The detailed outputs are:

- [phase2-equivalent-scope-comparison.md](./phase2-equivalent-scope-comparison.md)
- [phase2-preindexed-focus-details.md](./phase2-preindexed-focus-details.md)
- [phase3-evaluation-accounting.md](./phase3-evaluation-accounting.md)

## 1. The biggest comparable MT slowdown is usually in `EvaluatePass3`

Across the repeated shared OrchardCore project cohort, the strongest same-project/same-phase regression is usually **`EvaluatePass3`**, with `EvaluatePass1` still slower but less dominant.

- cross-project averages: `EvaluatePass3` **39.834 ms -> 106.241 ms** (`+66.406 ms`)
- representative heavy projects: `OrchardCore.ContentFields`, `OrchardCore.Resources`, `OrchardCore.Contents`

**More detail:** [phase2-equivalent-scope-comparison.md](./phase2-equivalent-scope-comparison.md)

## 2. `EvaluateCondition` is a systemic MT regression; `ExpandGlob` is real but more project-sensitive

For the sampled MT outliers, the **condition payload shapes stay the same** between non-MT and MT, but the paired durations get much worse in MT. Across the current 5-project sample, thread-scoped `EvaluateCondition` time rises from **87.541 ms** to **1,515.490 ms**.

`ExpandGlob` also shows up in the hot slices, but the regression is **not as uniform**: some projects are clearly slower in MT, while others are neutral or better. Right now it looks more like a secondary amplifier than the main systemic driver.

**More detail:** [phase2-preindexed-focus-details.md](./phase2-preindexed-focus-details.md#cross-project-sanity-check)

## 3. `RequestThreadProc` is supporting evidence, not the main actionable finding

Inside the peak `EvaluatePass3` windows, enclosed `RequestThreadProc` spans expand sharply in MT. In the current 5-project sample they grow from **18 spans / 1,368.099 ms** to **79 spans / 19,079.643 ms**, often spread across many MT threads.

The useful interpretation is mostly **diagnostic**: this supports that the MT slowdown is happening inside MSBuild's own request/build orchestration and includes a real coordination/overlap component. By itself, though, `RequestThreadProc` is too broad to be a primary root-cause finding because it wraps productive work **plus** waiting/blocking, and the old event shape was only **window-scoped**, not cleanly project-attributed.

So the current actionable emphasis should stay on `EvaluateCondition` first, with `ExpandGlob` as a secondary/project-sensitive amplifier. A follow-up PR adds request/project context to `RequestThreadProc` for better future traces.

- perf details: [phase2-preindexed-focus-details.md](./phase2-preindexed-focus-details.md#requestthreadproc-yes-there-is-more-detailed-perf-data-but-it-is-still-window-scoped)
- instrumentation follow-up: [PR #13978](https://github.com/dotnet/msbuild/pull/13978)

## 4. Evaluation is now explained well enough for a durable summary

Phase 3 answers the stricter question from the earlier summary: for the current two-trace corpus, the evaluation story is now **good enough** without blocking on more instrumentation first.

The strongest durable conclusion is:

1. the biggest repeated same-project MT expansion is usually **`EvaluatePass3`**;
2. **`EvaluateCondition`** is the clearest systemic inner-phase regression;
3. **`ExpandGlob`** is real but looks more like a secondary/project-sensitive amplifier;
4. **`RequestThreadProc`** is useful supporting evidence for coordination/overlap, but not the main attributable root-cause finding.

The remaining attribution gaps around `RequestThreadProc` and `ApplyLazyItemOperations` are refinement gaps, not blockers for the current summary.

### Confidence and perf-data coverage by family

| Family | Perf data quality | Current read |
| --- | --- | --- |
| `EvaluatePass0`-`EvaluatePass5` | **solid** | Cross-project same-project/same-pass timings; `EvaluatePass3` is the biggest repeated MT regression (`39.834 ms -> 106.241 ms`, `+66.406 ms`) |
| `EvaluateCondition` | **solid** | Thread-scoped paired spans on the top 5 MT outliers; same condition set, much slower in MT (`87.541 ms -> 1,515.490 ms` total across sample) |
| `ExpandGlob` | **solid but mixed** | Thread-scoped paired spans and per-glob comparisons; real but project-sensitive amplifier, not the universal driver |
| `RequestThreadProc` | **real but weaker attribution** | Strong window-scoped perf signal (`18 / 1,368.099 ms` -> `79 / 19,079.643 ms`), but no payload, so not cleanly project/request-attributed |
| `ApplyLazyItemOperations` | **partial** | Clearly hot in heavy windows, but attribution is weaker; current interpretation relies on enclosed totals plus sampled item types rather than clean per-project paired spans |
| `LoadDocument` / `Parse` | **solid enough to demote** | Not primary drivers in the heavy slices we compared |
| SDK resolver work | **solid enough to demote for phase 3** | Useful earlier lead, but not the dominant recurring explanation for the evaluation slowdown |

**More detail:** [phase3-evaluation-accounting.md](./phase3-evaluation-accounting.md)

## 5. Matching binlogs remain optional follow-up, not a phase-3 prerequisite

Matching binlogs still exist for the same builds, but they remain a **follow-up option**, not something phase 3 needed in order to reach a useful conclusion.

## 6. Evaluation profiler is a strong semantic follow-up

Another useful next artifact is the built-in evaluation profiler (`/profileevaluation`).

At a high level, it complements the ETL work well:

- ETL is better for **timeline shape**, **same-project MT vs non-MT wall-clock comparison**, and **coordination/overlap signals**.
- `/profileevaluation` is better for **semantic attribution inside evaluation**, because it reports inclusive/exclusive time by **pass**, **file**, **line**, and **expression**.

That makes it a good candidate for follow-up work on the current weak spots, especially:

- determining which concrete lazy-item or item-evaluation expressions dominate the hot `EvaluatePass3` / lazy-item region;
- checking whether MT and non-MT are spending time in the **same** imported expressions and rules, just with very different wall-clock cost;
- refining the current partial understanding around `ApplyLazyItemOperations`.

It should be treated as a **complement**, not a replacement, because it will not explain thread-level contention or request coordination the way ETL can.
