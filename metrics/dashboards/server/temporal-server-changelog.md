# Changelog — Temporal Server Dashboard

## v2.5.0 — 2026-06-11

### Added
- **Visibility** group: 3 new per-store panels using `visibility_persistence_*` metrics with `visibility_index_name` label to distinguish primary from secondary store health. Only meaningful when dual visibility is enabled; with a single store both series are identical.
  - **Visibility Write Request Rate per Store** — `sum(rate(visibility_persistence_requests{service_name="history"}[$__rate_interval])) by (visibility_index_name, operation)`. A flat line on one store while the other continues indicates that store has stopped receiving writes — either it is down or `system.secondaryVisibilityWritingMode` dynconfig has changed. Note: `visibility_persistence_*` metrics carry no `namespace` label so no namespace filter is applied.
  - **Visibility Write Error Rate per Store** — `sum(rate(visibility_persistence_errors{service_name="history"}[$__rate_interval])) by (visibility_index_name, operation)`. Primary alert signal for a visibility store outage. History retries failed visibility tasks indefinitely (backoff: 1s initial, 1.1× coefficient, 3-minute cap — no retry limit). Orange > 0.1 req/s, red > 1 req/s.
  - **Visibility Write Latency per Store** — `histogram_quantile($p, sum(rate(visibility_persistence_latency_bucket{service_name="history"}[$__rate_interval])) by (visibility_index_name, operation, le))`. Divergence between primary and secondary latency indicates one store is under pressure or recovering. Orange > 3s, red > 5s.

---

## v2.4.0 — 2026-05-28

### Added
- **Shard Queue Health** group: 2 new panels for task executor scheduler health (panels 7 and 8 of the group):
  - **Task Scheduler Latency per Operation** — `histogram_quantile($p, sum by (operation, le) (rate(task_latency_schedule_bucket{service_name="history"}[$__rate_interval])))`. In-memory schedule-to-start latency: time between a task being loaded into memory and acquiring an executor worker. Rises when the `history.transferProcessorSchedulerWorkerCount` goroutine pool is saturated. Primary signal when a bulk-processing namespace (e.g. mass terminations or deletions) is starving the shared worker pool. Orange > 500ms, red > 2s. `history.transferProcessorSchedulerWorkerCount` is hot-reloadable (no restart) — reduce it via dynamic config to throttle the saturating workload.
  - **Task Scheduler Throttled Rate per Operation** — `sum by (operation) (rate(task_scheduler_throttled{service_name="history"}[$__rate_interval]))`. Rate of tasks explicitly rejected by the scheduler. Complements the latency panel: latency rising = tasks queueing for a worker; throttled rising = tasks being turned away hard. Any sustained non-zero value warrants investigation.

### Fixed
- Dashboard title corrected from `v2.3.0` to `v2.4.0` (title was not bumped in v2.3.1, which was a metadata-only fix).

---

## v2.3.1 — 2026-05-27

### Fixed
- **Immediate Queue Lag per Pod** and **Scheduled Queue Lag per Pod**: changed rate window from `[$__rate_interval]` to `[11m]` (hardcoded). `shardinfo_immediate_queue_lag` and `shardinfo_scheduled_queue_lag` are emitted by `monitorQueueMetrics()` on a fixed 5-minute timer (`queueMetricUpdateInterval = 5 * time.Minute`, `context_impl.go:73`). Grafana's `$__rate_interval` resolves to ~1 minute (4 × default 15s scrape interval), which never spans 2 consecutive emissions — `histogram_quantile` returns NaN and both panels show No Data. A fixed `[11m]` window (>2× the emission interval) always captures at least 2 data points.

---

## v2.3.0 — 2026-05-27

### Added
- New panel group **Shard Queue Health** (group 9, inserted between Shard Movement and History Timer Task Info) with 6 panels for stuck shard detection:
  - **Immediate Queue Lag per Pod** — `histogram_quantile($p, sum by (instance, task_category, le) (rate(shardinfo_immediate_queue_lag_bucket{service_name="history"}[11m])))`. Orange > 500K tasks, red > 3M tasks. Primary signal for a stuck shard — one `instance + task_category` line rising monotonically while others recover.
  - **Scheduled Queue Lag per Pod** — same structure over `shardinfo_scheduled_queue_lag_bucket`. Orange > 10 min, red > 30 min.
  - **DB Pool Refresh Failure Rate per Pod** — `sum by (instance) (rate(persistence_session_refresh_failures{service_name="history"}[$__rate_interval]))`. Earliest signal for DB-caused stuck shards; fires before queue lag builds. SQL backends only.
  - **DB Pool Refresh Failure Ratio per Pod** — failures / attempts ratio. Orange > 10%, red > 50%. SQL backends only.
  - **Suspected Deadlocks (current) per Pod** — `sum by (instance) (dd_current_suspected_deadlocks{service_name="history"})`. Event-driven gauge; absence of data is healthy. Any value > 0 requires pod restart.
  - **Deadlock Event Rate per Pod** — `sum by (instance) (rate(dd_suspected_deadlocks{service_name="history"}[$__rate_interval]))`. Complements the gauge — shows cumulative detection events after the gauge has cleared.

### Changed
- Groups 9–18 renumbered to 10–19 to accommodate the new group

---

## v2.2.0 — 2026-05-15

### Fixed
- Excluded `_unknown_` namespace from all panels that group or filter by namespace. The `_unknown_` value is emitted by Temporal for internal/system-level requests that have no namespace context and should not appear as a selectable namespace or as a series in namespace-breakdown panels.
  - Namespace template variable query updated: `label_values(service_requests{namespace!="_unknown_"}, namespace)` — `_unknown_` no longer appears in the namespace dropdown
  - Panels patched: **Actions per Namespace** (18), **RPS per Namespace** (20), **Service Requests by Namespace and Operation** (93), **Service Errors by Namespace and Operation** (100), **Actual RPS vs Namespace Host RPS Limit** (123), **Outlier Namespaces** (2004)

---

## v2.1.0 — 2026-05-13

### Added
- New panel group **Worker Registry (In-memory)** (group 16, inserted between Visibility and Cluster Replication) with 5 panels:
  - **Workers Added** — rate of new worker registrations
  - **Workers Removed** — rate of removals across all causes (shutdown, TTL eviction, capacity eviction)
  - **Percentile of Num of Cached Entries** — estimated entry count derived from `capacity_utilization × 1e6` at the selected `$p` percentile across matching instances
  - **Percentile of Cache Utilization** — utilization as a percentage at the selected `$p` percentile, with threshold lines at 80% (orange) and 100% (red)
  - **Workers - Number of Activity Slots Used** — `histogram_quantile` of `worker_registry_activity_slots_used` at the selected `$p` percentile

### Changed
- Cluster Replication renumbered from group 16 → 17
- Authorization renumbered from group 17 → 18

---

## v2.0.0 — 2026-05-12

First versioned release. Prior changes were unversioned.

### Fixed
- Corrected metric name in Shard Movement > Shards Closed panel: `sharditem_closed_count` → `shard_closed_count`
