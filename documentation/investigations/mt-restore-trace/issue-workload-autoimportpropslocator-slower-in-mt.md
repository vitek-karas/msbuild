# Finding: `Microsoft.NET.SDK.WorkloadAutoImportPropsLocator` is slower in MT evaluation slices

## Scope

This note captures a focused finding from the `restore-warm` vs `restore-warm-mt` ETL comparison.

- multi-proc trace: one **worker node** from `C:\repro\perf\msbuild-mt\restore-warm\restore-warm.etl`
- MT trace: the single **multi-threaded process** from `C:\repro\perf\msbuild-mt\restore-warm-mt\restore-warm-mt.etl`

These numbers are **not** whole-build comparisons. They are normalized to the **same project** and the **same first `EvaluatePass1` slice**.

## Bottom line

Across the sampled project slices, the evaluation-time SDK lookup for:

- `Microsoft.NET.SDK.WorkloadAutoImportPropsLocator`

is consistently **slower in the MT trace** than the comparable out-of-proc worker-visible lookup in the multi-proc trace.

By contrast, the plain:

- `Microsoft.NET.Sdk`

lookup trends the **other way** and is generally **faster in-proc**.

That makes the workload auto-import props locator a much better candidate for the observed MT expansion than the basic SDK lookup.

## First anchor project

Project:

- `C:\Users\alinama\work\testRepos\OrchardCore\src\OrchardCore\OrchardCore.DisplayManagement\OrchardCore.DisplayManagement.csproj`

First `EvaluatePass1` timings:

| SDK | Out-of-proc worker-visible | In-proc MT | Delta |
| --- | ---: | ---: | ---: |
| `Microsoft.NET.Sdk` | 27.14 ms | 9.89 ms | in-proc faster by 17.25 ms |
| `Microsoft.NET.SDK.WorkloadAutoImportPropsLocator` | 95.18 ms | 111.80 ms | in-proc slower by 16.61 ms |

The full pass for this project was:

- non-MT worker trace: 138.36 ms
- MT trace: 233.80 ms

So the slower workload auto-import props locator explains part of the pass expansion, while the plain `Microsoft.NET.Sdk` lookup does not.

## Broader sample

I then checked **14 additional shared `.csproj` slices** in the first `EvaluatePass1` window where at least one side was not already on the trivial fully-cached fast path.

Projects:

- `OrchardCore.Admin.Abstractions.csproj`
- `OrchardCore.ContentManagement.Abstractions.csproj`
- `OrchardCore.ContentManagement.Display.csproj`
- `OrchardCore.ContentManagement.csproj`
- `OrchardCore.ContentTypes.Abstractions.csproj`
- `OrchardCore.Data.Abstractions.csproj`
- `OrchardCore.Data.csproj`
- `OrchardCore.Deployment.Abstractions.csproj`
- `OrchardCore.Deployment.Core.csproj`
- `OrchardCore.Recipes.Abstractions.csproj`
- `OrchardCore.Recipes.Core.csproj`
- `OrchardCore.ResourceManagement.Abstractions.csproj`
- `OrchardCore.ResourceManagement.csproj`
- `OrchardCore.Tests.csproj`

Average resolver timings across those 14 additional slices:

| SDK | Slices | Out-of-proc avg | In-proc avg | Delta |
| --- | ---: | ---: | ---: | ---: |
| `Microsoft.NET.Sdk` | 14 | 21.47 ms | 10.20 ms | in-proc faster by 11.27 ms |
| `Microsoft.NET.SDK.WorkloadAutoImportPropsLocator` | 14 | 91.14 ms | 112.23 ms | in-proc slower by 21.09 ms |

If the original `OrchardCore.DisplayManagement.csproj` anchor is included, the broader “interesting slice” average becomes:

| SDK | Slices | Out-of-proc avg | In-proc avg | Delta |
| --- | ---: | ---: | ---: | ---: |
| `Microsoft.NET.Sdk` | 15 | 21.84 ms | 10.18 ms | in-proc faster by 11.67 ms |
| `Microsoft.NET.SDK.WorkloadAutoImportPropsLocator` | 15 | 91.41 ms | 112.20 ms | in-proc slower by 20.80 ms |

## Additional observations

### 1. This is evaluation-time import plumbing, not task or target execution

The resolver activity appears inside `EvaluatePass1` because SDK-style projects need evaluation to resolve implicit `Sdk.props` / `Sdk.targets` imports before evaluation can continue.

### 2. The MT slowdown is not a generic “all in-proc SDK resolution is slower” effect

The data shows the opposite for `Microsoft.NET.Sdk`: in the sampled slices it is materially **faster** in-proc.

That narrows the suspicion to the workload auto-import props locator path, not to SDK resolution in general.

### 3. Many slow MT observations are marked cached

For the workload auto-import props locator, many MT samples in the slow ~112 ms range are marked `wasResultCached=true`, while the corresponding out-of-proc side is often the non-cached one.

That strongly suggests the slow MT observations are frequently **wait time on a shared in-proc cached/Lazy resolution**, not repeated fresh resolver execution on each project.

### 4. Fully cached calls are tiny on both sides

When both sides are already on the fully cached fast path, the timings collapse:

| SDK | Slices | Out-of-proc avg | In-proc avg | Delta |
| --- | ---: | ---: | ---: | ---: |
| `Microsoft.NET.Sdk` | 169 | 0.0085 ms | 0.0102 ms | +0.0016 ms |
| `Microsoft.NET.SDK.WorkloadAutoImportPropsLocator` | 169 | 0.0102 ms | 0.1248 ms | +0.1146 ms |

So the important signal is not the steady-state cached cost. The important signal is the **shared wait / first-resolution behavior** that shows up in the interesting project slices.

## Source-code findings

The source supports a more concrete version of the contention hypothesis.

### 1. `WorkloadAutoImportPropsLocator` is resolved by the workload SDK resolver path, not the plain in-box SDK path

- In `dotnet/sdk`, `Microsoft.NET.Sdk.ImportWorkloads.props` imports:
  - `<Import Project="AutoImport.props" Sdk="Microsoft.NET.SDK.WorkloadAutoImportPropsLocator" />`
- In `dotnet/msbuild`, the normal `Microsoft.NET.Sdk` case can often be satisfied by the default SDK resolver fast path, but `Microsoft.NET.SDK.WorkloadAutoImportPropsLocator` falls through to the workload resolver path instead.

That lines up with the trace result where `Microsoft.NET.Sdk` is generally faster in-proc while the workload auto-import locator is not.

### 2. The workload resolver does real per-installation scanning work for this SDK name

In `dotnet/sdk` (`src/Resolvers/Microsoft.NET.Sdk.WorkloadMSBuildSdkResolver/CachingWorkloadResolver.cs`), the resolver special-cases:

- `Microsoft.NET.SDK.WorkloadAutoImportPropsLocator`

and resolves it by:

1. enumerating installed workload packs of kind `Sdk`
2. building `$(pack)\Sdk\AutoImport.props` candidate paths
3. calling `File.Exists` for each candidate
4. returning the distinct set of matching SDK folders

So this lookup is not a trivial dictionary hit. Its cold or first-hit cost depends on the installed workload set and filesystem probes across those packs.

### 3. The workload resolver serializes its own body under a lock

Still in `CachingWorkloadResolver.cs`, the public `Resolve(...)` method does all of this under:

- `lock (_lockObject)`

while also:

- lazily creating `SdkDirectoryWorkloadManifestProvider`
- lazily creating `WorkloadResolver.Create(...)`
- caching results by SDK name

So even inside the resolver implementation, workload-related resolution is intentionally serialized per cached resolver instance.

### 4. MSBuild adds a second shared wait layer on top: per-submission single-flight by SDK name

In `dotnet/msbuild`:

- `MainNodeSdkResolverService` uses `CachingSdkResolverService`
- `CachingSdkResolverService` caches by:
  - submission id
  - SDK name
- the cache value is `Lazy<SdkResult>`

That means all projects in the same submission that request:

- `Microsoft.NET.SDK.WorkloadAutoImportPropsLocator`

share one in-flight `Lazy<SdkResult>`. One thread does the real resolver work; the rest block on `resultLazy.Value` and still show up as:

- `wasResultCached=true`

This is an especially strong fit for the ETL pattern in this note: many of the slow MT observations are marked cached, which is exactly what followers waiting behind the first `Lazy` resolution would look like.

### 5. Why MT can look worse than the sampled worker-node trace

In multi-threaded mode, many projects in the same process and submission can hit the same shared:

- MSBuild `Lazy<SdkResult>` for this SDK name
- workload-resolver internal lock

very early in `EvaluatePass1`.

In the sampled non-MT trace, we are looking at only one out-of-proc worker node, not the whole multi-proc build topology. That worker can still pay the real workload scan cost, but it does not expose the same whole-process in-proc fan-in shape that MT does.

## Interpretation

The current trace evidence plus source inspection supports this more specific hypothesis:

> In MT mode, `Microsoft.NET.SDK.WorkloadAutoImportPropsLocator` resolution is expensive enough on its first hit to matter, and MSBuild then amplifies that cost by making all same-submission callers share one in-flight `Lazy<SdkResult>`. The underlying workload resolver also serializes its own body with a lock while it builds workload-manifest state and scans installed workload SDK packs for `AutoImport.props`. The result is that many projects pay mostly **wait time** inside `EvaluatePass1`, often while being reported as cached.

The evidence does **not** support the broader claim that all SDK resolution is worse in MT. The problem appears more specific than that.

## Next checks

1. Confirm in the trace whether one thread is doing the actual workload resolver work while peer evaluation threads are stalled in the cached resolver wrapper for the same SDK name.
2. Estimate how many installed workload SDK packs are present on the repro machine, since the `AutoImport.props` scan cost scales with that set.
3. Check whether the MT slowdown is dominated by MSBuild's outer same-SDK `Lazy` wait, the resolver's inner lock/workload scan, or both.
