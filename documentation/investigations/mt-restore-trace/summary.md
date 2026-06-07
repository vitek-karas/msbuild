# MT restore trace investigation: short summary

This file keeps the **shortest useful summary** of the most important findings so far from the parallel-bottleneck trace work.

For the current two-trace corpus, **phase 2 is complete**. The detailed phase-2 outputs are:

- [phase2-equivalent-scope-comparison.md](./phase2-equivalent-scope-comparison.md)
- [phase2-preindexed-focus-details.md](./phase2-preindexed-focus-details.md)

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

## 4. The next explicit check is whether evaluation is now explained well enough end-to-end

The next planned phase is no longer just “follow the hottest events.” It should now answer a stricter question:

- do we understand **all material evaluation-phase differences** between MT and non-MT well enough to summarize where the time goes;
- or are there still attribution holes large enough to require a small amount of new MSBuild instrumentation first?

If the current traces are sufficient, the output should be a short evaluation-phase accounting summary. If they are not, the output should be a **limited**, evidence-driven list of instrumentation improvements rather than a broad wishlist.

Matching binlogs also exist for the same builds, but that cross-artifact work is intentionally deferred until **after** this evaluation-accounting pass.
