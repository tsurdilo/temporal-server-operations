# Changelog — Temporal Task Queue Partitions Dashboard

## v1.2.0 — 2026-08-10

### Added
- **Dispatch Rate-Limiting by Partition** panel in the *Sync Match & Dispatch — Path 1 Ceiling* row (`poller_scale_decision{reason="rate_limited"}`, `sum by (partition)`). This is the priority matcher's observable signal for the sync-match / dispatch (path 1) ceiling — the gap left when **v1.0.2** removed the classic-only `sync_throttle_count` panel. When the per-partition dispatch cap (`admin.matchingNamespaceTaskqueueToPartitionDispatchRate`, 1,000/s) throttles dispatch, matching records a poller-autoscaling decision tagged `reason="rate_limited"`. **Requires the opt-in dynamic config `matching.enablePollerScalingDecisionMetrics` (default off) — the panel reads empty otherwise** (title carries an "(opt-in)" hint; the readme documents the prerequisite). On the **classic** matcher use `sync_throttle_count` instead (not on this dashboard — classic-only). *Async Match Latency by Partition* was narrowed to half width to make room; no other queries changed. Verified against source (`service/matching/physical_task_queue_manager.go` `recordPollerScaleDecision`, `matcher_data.go` `onRateLimited`, `common/metrics/metric_defs.go`, `common/dynamicconfig/constants.go`).

## v1.1.0 — 2026-08-02

### Changed
- **Dashboard text consolidated to a single signpost.** Removed all per-panel info descriptions and the four per-row text notes. Added one text panel at the top linking the [Changing Task Queue Partitions playbook](../../../playbooks/change-task-queue-partitions.md) (how to decide & act) and this dashboard's readme (what each panel means), plus the "empty panels?" prerequisite hint. Panel meaning now lives only in the readme and decision guidance only in the playbook — nothing is duplicated across the dashboard, so there's a single place to maintain each. Panels relaid out to fill the gaps left by the removed text; no queries changed.
- The readme was reworked to match: per-row sections now list each panel's metric and what it measures, and defer "how to read this row for a partition decision" to the relevant playbook section.

## v1.0.4 — 2026-08-02

### Removed
- **Physical Backlog Age by Partition** panel. `physical_approximate_backlog_age_seconds` reflects only the oldest task the partition's reader holds **in memory** (a bounded, dispatch-driven buffer), so on a **write-bound** queue (large backlog, few/slow pollers) it reads **0 even with a large physical backlog** — it goes flat exactly when a backlog is building and blocked. A permanently-misleading age panel was worse than none. Use **Approximate Backlog Age by Partition** (`approximate_backlog_age_seconds`, logical) for backlog age and drain confirmation. Physical **Count** is kept (reliable, version-agnostic) and widened to fill the row. Verified against source (`service/matching/db.go`, `backlog_age_tracker.go`, `pri_task_reader.go`) and a live cluster (physical count 5,043, in-memory age 0, logical age 528s at the same instant). The removed metric's semantics are being raised with the Temporal server team.

## v1.0.3 — 2026-08-02

### Changed
- **Physical Backlog Age by Partition** panel description now carries a caveat: `physical_approximate_backlog_age_seconds` reflects only the oldest task the partition's reader holds **in memory**, so it can read **0 even when the physical backlog count is large** (backed-up reader — heavy write burst, few/slow pollers). It is computed from a different internal source than the count (`service/matching/db.go` in-memory `backlogAgeTracker`) and than the logical age. The readme now documents physical-vs-logical backlog and steers backlog-age / drain confirmation to `approximate_backlog_age_seconds` (logical), which is the reliable age signal. No query changes — panels unchanged; caveat only. Verified against source and a live cluster (physical count ~2,600/partition, physical age 0, logical age ~115s at the same instant).

## v1.0.2 — 2026-08-02

### Removed
- **Sync Throttle Count by Partition** panel (Row 2). `sync_throttle_count` is emitted only by the classic `TaskMatcher`; the priority matcher — the default since server **v1.31.0** (`matching.useNewMatcher`, added v1.28.0, defaulted on in v1.31.0) — does not emit it and has no replacement metric. The panel was therefore always empty on any default modern cluster and misleading. Sync-match (path 1) saturation has no direct metric on the priority matcher; it is inferred from the backlog panels and async-match latency.

### Changed
- **Async Match Latency by Partition** widened to full width to fill Row 2, and the Row 2 note rewritten to explain the absence of a direct sync-match metric.

## v1.0.1 — 2026-08-02

### Fixed
- **Enum tag values corrected to their emitted form.** Every panel that filtered on `task_type` used the full protobuf enum name (`TASK_QUEUE_TYPE_ACTIVITY` / `TASK_QUEUE_TYPE_WORKFLOW`), and the Busy Workflow panel filtered `resource_exhausted_cause="RESOURCE_EXHAUSTED_CAUSE_BUSY_WORKFLOW"`. Temporal's `TaskQueueType` and `ResourceExhaustedCause` enums both define a custom `String()` that emits the short CamelCase form, so the actual tag values are `Activity` / `Workflow` and `BusyWorkflow` (also `RpsLimit` / `SystemOverloaded`). The full-enum-name filters matched nothing, so **every task_type-scoped panel read "No data"** — only the two namespace-only panels (Persistence Latency) showed data. The `$task_type` variable, all panel queries, and the panel descriptions now use the emitted short forms. Verified against a live cluster's Prometheus and against `go.temporal.io/api` `enums/v1` (`task_queue.pb.go`, `failed_cause.pb.go`).

## v1.0.0 — 2026-08-02

Initial release. Companion to the [Changing Task Queue Partitions](../../../playbooks/change-task-queue-partitions.md) playbook. Scoped to a single task queue (`$namespace` + `$taskqueue` + `$task_type`).

### Added

- **Row 1 — Per-Partition Backlog (Drain & Write>Read Detection)**: per-partition `approximate_backlog_count` / `approximate_backlog_age_seconds`, plus the v1.31.0+ version-agnostic `physical_approximate_backlog_count` / `physical_approximate_backlog_age_seconds`. The drain-confirmation and Write>Read-detection panels the increase/decrease procedures key on.
- **Row 2 — Sync Match & Dispatch (Path 1 Ceiling)**: per-partition `sync_throttle_count` (classic-matcher-only caveat noted) and per-partition `asyncmatch_latency` (`$p` percentile).
- **Row 3 — Backlog Write (Path 2 Ceiling)**: per-partition `task_write_throttle_count` and `task_write_latency` (`$p`), plus a namespace-scoped `service_errors_resource_exhausted` breakdown by operation and cause.
- **Row 4 — Rule-Outs**: queue-level `task_schedule_to_start_latency` (`$p`), namespace-scoped Busy Workflow throttling via `task_errors_throttled{resource_exhausted_cause="RESOURCE_EXHAUSTED_CAUSE_BUSY_WORKFLOW"}`, and `persistence_latency` for `UpdateWorkflowExecution` / `CreateWorkflowExecution` (`$p`).
- **Template variables**: `$namespace`, `$taskqueue` (dependent on `$namespace`), `$task_type` (activity default), `$p` (percentile, default 0.99).
- **Annotation**: Matching Restarts (`increase(restarts{service_name=~"matching"})`).

### Notes

- Every metric name, tag, and per-partition breakdown was verified against Temporal server source (`~/devel/temporal/temporal`) at `v1.29.0-135.0-1716-g683b3403f`, api module `v1.62.15-...`.
- The per-partition panels (Rows 1–3) require both `metrics.breakdownByPartition` and `metrics.breakdownByTaskQueue` on for the viewed task queue. See the readme prerequisites.
- `task_schedule_to_start_latency`, `service_errors_resource_exhausted`, `task_errors_throttled`, and `persistence_latency` do not carry a numeric `partition` tag, so those panels are queue- or namespace-level by design.
- `sync_throttle_count` is emitted only by the classic `TaskMatcher`; it reads flat on task queues served by the priority/fair matcher.
