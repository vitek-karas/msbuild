# Nettrace MCP Improvements

Append new, generally useful MCP improvement suggestions here after turns that used the `nettrace` MCP server and identified an actionable improvement.

## 2026-06-05 - ETL support and clearer input guidance

- The current `open_trace` tool only supports `.nettrace` files. The tool description says this, but the failure mode on an `.etl` input was not self-explanatory enough during interactive use. Improve the tool/runtime error so it explicitly says that `.etl` is unsupported and points the user to the supported input formats or conversion workflow.
- Add a generally useful import/open capability for Windows ETL traces, or a companion tool that can inspect ETL metadata/providers before import. ETL is a common format for Windows performance investigations, and being able to either ingest it directly or clearly bridge from ETL to the MCP's event model would make the server much more useful beyond this one MSBuild scenario.

## 2026-06-05 - Richer open_trace failures

- `open_trace` still returns a generic "An error occurred invoking 'open_trace'" failure even after ETL support was supposedly added. Surface the underlying exception category/message (for example: unsupported ETL shape, missing parser dependency, file access issue, or corrupted trace) so users can distinguish server-reload issues from real trace parsing failures.

## 2026-06-05 - Scoped indexing for large traces

- Large ETL traces are currently truncated after 1,000,000 indexed events. Add a generally useful way to open/index a scoped subset of a trace first (for example by provider, process, or time window), or allow deferred/on-demand indexing beyond the initial cap, so investigations can stay complete without loading the entire trace up front.

## 2026-06-05 - Aggregation and paired-duration summaries

- Add a generally useful trace-summary tool that can group by fields (for example `eventName`, `payload.projectFile`, `processId`, or `threadId`) and optionally pair `Start`/`Stop` events into duration summaries. That would make it much easier to answer questions like "which projects have the longest `EvaluatePass1` durations?" or "what are the top project-scoped event counts?" without many manual `QueryEvents` calls.

## 2026-06-06 - Long-running open with progress

- Opening large ETLs without an indexing cap can exceed the current request timeout. Add a generally useful long-running `open_trace` mode with progress reporting and/or background indexing, plus a way to inspect partial readiness, so large traces remain usable without forcing the caller to choose between truncation and timeouts.

## 2026-06-06 - Document background open behavior

- After background indexing was added, the MCP workflow changed in a useful way, but the expected polling/readiness flow is not obvious from the tool surface alone. Add explicit guidance in the `open_trace` response or tool docs describing the ready/not-ready states and the next recommended call sequence so users do not have to infer the new workflow by trial and error.

## 2026-06-06 - Easier full-result retrieval across pages

- `query_events` paging is easy to underestimate on large traces: a first page for a project can look complete while later pages contain the rest of the same logical operation sequence. Add a generally useful `query_all`/`drain_pages` option, or return clearer page-level guidance for complete logical sequences, so users can avoid accidental partial conclusions from the first page only.

## 2026-06-06 - Stalled finalization visibility

- `open_trace` can appear to reach a terminal event count and then remain in `status: indexing` with unchanged progress for repeated polls, leaving `describe_trace_schema`/`query_events` blocked even though indexing looks effectively complete. Add either an explicit `finalizing` state with heartbeat details or stalled-index detection/timeout diagnostics so callers can tell whether they should keep waiting or treat the open as stuck.

## 2026-06-07 - Span aggregation inside enclosing windows

- Add a generally useful analysis tool that can pair `Start`/`Stop` events into spans, restrict them to an enclosing span/window (for example `EvaluatePass1` grouped by `payload.projectFile`), and then aggregate inner spans by payload fields such as `payload.sdkName`, `processId`, `threadId`, or cache flags. This would make investigations like "average SDK resolver time per project slice across two traces" possible in one server-side query instead of many event-page queries plus local post-processing.

## 2026-06-07 - Higher-level comparison and contention attribution

- Add a generally useful cross-trace comparison capability that can align the same logical operation across two traces (for example by project, activity, phase, event name, or payload keys) and summarize deltas instead of forcing manual paging and hand-alignment. This would help investigations that are trying to answer "what got slower between trace A and trace B?" at a semantic level rather than an event-by-event level.
- Add a generally useful contention/wait-attribution view that can explain when many intervals are mostly followers waiting on one underlying operation, lock, or single-flight computation. For performance investigations, this would make it much faster to distinguish "many repeated expensive operations" from "one expensive operation plus many waiters."

## 2026-06-07 - Long-running project-scoped queries

- Add a generally useful long-running/background `query_events` mode, or a configurable higher timeout, for targeted but still heavy ETL queries such as "all Microsoft-Build events for one project with full payloads." Today those queries can time out even when they are the natural next step after schema discovery, which pushes investigations toward direct cache access instead of staying inside the MCP workflow.

## 2026-06-07 - Proposal: selective pre-indexing for ETL investigations

- Add a new two-phase indexing workflow for large traces:
  1. a cheap schema-discovery/open step that inventories providers, event names, and payload field names without fully indexing all payload properties;
  2. a selective pre-index step where the caller specifies the small set of providers, event families, and payload fields they expect to query.
- The selective pre-index request should support filters like:
  - provider names (for example `Microsoft-Build`)
  - event names or prefixes (for example `Evaluate*`, `LoadDocument`, `Parse`, `ExpandGlob`)
  - top-level fields already stored on every event (`timeStampRelativeMSec`, `processId`, `threadId`, `activityId`, `relatedActivityId`)
  - payload fields to materialize/index (for example `projectFile`, `projectPath`, `sdkName`, `wasResultCached`, `fullPath`, `projectFileName`, `glob`, `rootDirectory`)
- The indexed view should be extensible: if the investigation changes direction, the caller should be able to add a few more payload fields or event families later without rebuilding the whole trace from scratch.
- This would fit the real workflow from this session well: the investigation only needed a narrow `Microsoft-Build` slice and a small set of project/pass/resolver fields, but the current MCP had to pay the cost of broad indexing before those needs could be exploited.

## 2026-06-07 - Ready transition after full ETL indexing

- `open_trace` can still remain in `status: indexing` even after `indexedEventCount == totalObservedEvents`, `progressFraction == 1`, and the estimated remaining time is 0. In that state the new selective `create_index` feature is unreachable because it still requires a ready trace. Add an explicit post-index finalization state with progress, or transition to a queryable ready-for-create-index state as soon as the base event index is complete.
