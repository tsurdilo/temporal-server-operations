# Changelog — Temporal Server Dashboard

## v2.14.0 — 2026-09-02

### Fixed
- **Hot-shard detector corrected to use `max`, not p999** (both Persistence-row panels from v2.13.0). Empirical testing on a 2048-shard cluster exposed that the v2.13.0 detector **missed a single hot shard** — the primary "one hot workflow id → one hot shard" case. A single hot shard sits at percentile `(N−1)/N` among `N` active shards, which on a large fleet is *above* the 99.9th percentile (1 of 2048 = the 99.95th), so `p999` landed on a normal shard and the skew read ~1 while one shard was genuinely on fire.
  - **Per-Shard Persistence RPS Distribution** — the hottest-shard line changed from `p999` → **`max` (`histogram_quantile(1.00, …)`)**, relabeled "max (hottest single shard)"; the `p99` line relabeled "p99 (many shards hot)" (it catches the *broad* Kind-1 case). p50 = typical shard.
  - **Hot-Shard Skew → replaced with "Hottest Shard RPS".** The v2.13.0 skew *ratio* (`max ÷ clamp_min(p50, 1)`) was unintuitive: because a typical shard is usually below 1 req/s, the divide-by-zero guard floored the denominator to 1, so the "ratio" collapsed to just `max` and read a meaningless "1.0" when idle. Replaced it with a plain **`max`** stat — the single busiest shard's req/s — titled **Hottest Shard RPS**, unit req/s, thresholds orange 200 / red 500 (illustrative, cluster-tunable). It answers "how hot is the hottest shard?" directly; compare against p50 on the distribution panel.
  - Panel descriptions and the readme now document: `max` is **coarse** (rounds to the histogram bucket boundary, so magnitude is approximate); both panels need **enough active shards** (busy fleet of hundreds-plus) to be meaningful and are noisy on tiny/idle clusters; `p99` is the many-shards-hot signal. Alert 84 updated in lockstep to use `max`.

## v2.13.0 — 2026-08-31

### Added
- **Persistence Requests, Latencies and Errors** group — two panels for **hot-shard detection**, backing the recurring "how do I find a hot shard?" question. There is no per-shard-tagged metric in Temporal (shard id is a log dimension only, by design), so hot-shard detection is a distribution read, not a single series. Both panels use `persistence_shard_rps` — a histogram of per-shard persistence RPS, computed per shard but recorded with **no `shard_id` or `namespace` label**; default-on (`system.persistenceHealthSignalMetricsEnabled`, default `true`, verified `common/dynamicconfig/constants.go`), emitted every 30s per history host from `common/persistence/health_signal_aggregator.go`, backend-agnostic (Cassandra and SQL alike). Both panels are cluster-wide and ignore `$namespace`.
  - **Per-Shard Persistence RPS Distribution (Hot-Shard Detector)** — p50 / p99 / p999 of `persistence_shard_rps_bucket{service_name="history"}` on one timeseries. `histogram_quantile(0.50|0.99|0.999, sum by (le) (rate(...[$__rate_interval])))`. p99/p999 far above p50 = a hot shard exists; all three together = balanced.
  - **Hot-Shard Skew (hottest ÷ typical shard RPS)** — a stat panel: `histogram_quantile(0.999, ...) / clamp_min(histogram_quantile(0.50, ...), 1)`. One-number skew glance; orange (3) / red (10) thresholds are illustrative and cluster-tunable.
- These panels detect **that** a hot shard exists, not **which** — the shard id comes from the `"Shard queue lag exceeds warn threshold."` WARN log (`history.emitShardLagLog`). Readme section 3 documents the full detect-then-localize flow and cross-links the Shard IO Concurrency playbook (serialization vs hot-shard distinction).

## v2.12.0 — 2026-08-24

### Added
- **History Scavenger** group (new, appended as the last row): detection surface for **unbounded history growth when the history scavenger falls behind** (most often on an XDC standby). The scavenger clears leftover history (`history_tree` / `history_node` rows whose execution is already deleted) but skips any branch younger than `worker.historyScannerDataMinAge` (default 60 days); for short-lived, high-volume workflows that is nearly every branch, so history piles up. Metric names verified against server source — `scavenger_success` / `scavenger_skips` / `scavenger_errors` are recorded only by `service/worker/scanner/history/scavenger.go`, always tagged `operation="HistoryScavenger"`. `scavenger_success` counts branches **handled** (kept or deleted), not deletions. Panels (each `$__rate_interval`, `$DS_PROMETHEUS` only — these metrics carry no `namespace` label):
  - **Scavenger Activity — Skipped vs Handled** — `sum(rate(scavenger_skips{operation="HistoryScavenger"}[$__rate_interval]))` versus `sum(rate(scavenger_success{operation="HistoryScavenger"}[$__rate_interval]))`. When skipped dwarfs handled, the 60-day wait is blocking cleanup.
  - **Scavenger Errors** — `sum(rate(scavenger_errors{operation="HistoryScavenger"}[$__rate_interval]))`. Should sit at ~0; a sustained non-zero rate is a distinct problem from the 60-day wait.
- Backs the new **[XDC Standby Database Growth on SQL playbook](../../../playbooks/xdc-standby-database-growth-sql.md)**.

### Fixed
- Table of Contents now lists **21. Archival Health** (omitted when that group was added in v2.11.0) and **22. History Scavenger**.

## v2.11.0 — 2026-08-12

### Added
- **Archival Health** group (new, appended as the last row): detection surface for a **sustained archival-backend (S3 / GCS / custom) outage**. A dead backend fails every closed workflow's archival task; after `history.TaskDLQUnexpectedErrorAttempts` (default 70 ≈ 1h) each is dead-lettered, and a large burst of DLQ writes can back-pressure the whole database. Two primary detection signals back new essential alerts 81 and 82. Metric name and `status` tag values verified against server source (`service/history/archival/archiver.go` — `status` ∈ {`ok`, `err`, `rate_limit_exceeded`}); `dlq_writes` `operation` tag verified against `service/history/queues/dlq_writer.go` (`OperationTag(taskType)` → `ArchivalTaskArchiveExecution`). Three panels:
  - **Signal 1 — Archival Attempt Error Rate** — `sum(rate(archiver_archive_latency_count{status="err"}[$__rate_interval]))`. `status="err"` excludes rate-limit rejections (`status="rate_limit_exceeded"`), so it reflects genuine backend failures. The **earliest** signal — fires ~1h before failing archival tasks reach the history task DLQ. Backs essential alert 81.
  - **Archival Attempts by Status (ok / err / rate_limit_exceeded)** — `sum(rate(archiver_archive_latency_count[$__rate_interval])) by (status)`. Context for Signal 1: confirms whether archival is succeeding at all and separates a genuine outage (`err`) from archival rate-limiting (`rate_limit_exceeded`).
  - **Signal 2 — History Task DLQ Writes & Write Failures** — two series: `sum(rate(dlq_writes{operation="ArchivalTaskArchiveExecution"}[$__rate_interval]))` (archival tasks reaching the DLQ after 70 failed attempts) and `sum(rate(task_dlq_failures[$__rate_interval]))` (writes to the single-partition DLQ themselves failing — database distress). Backs essential alert 82.
  - **Archival Errors by Type (non-retryable vs transient)** — `history_archiver_archive_non_retryable_error` / `_transient_error` and the `visibility_archiver_archive_*` equivalents (metric names verified against `common/metrics/metric_defs.go`). Non-retryable (bad endpoint / DNS NXDOMAIN) indicates a hard, sustained outage; transient (timeouts) may self-heal — supports the sustained-vs-intermittent judgment central to the playbook.
- Companion playbook: **[Detecting & Recovering from an Archival Backend Outage](../../../playbooks/detecting-recovering-archival-outage.md)** (full mechanism, detection queries, pause remediation, recovery).

## v2.10.0 — 2026-08-04

### Added
- **History Task DLQ / Terminal Failures** group (new, appended as the last row): surfaces history tasks being dead-lettered under database stress — a prolonged DB outage/overload drives timer/retry tasks past `history.TaskDLQUnexpectedErrorAttempts` (default 70 ≈ 1h) of unexpected errors (`context deadline exceeded` / `context canceled`) and into the history task DLQ (`history.TaskDLQEnabled`, default true), stranding activities. DB-agnostic (Cassandra and SQL alike). Four panels:
  - **Dead-Lettered Tasks — Execution-Stranding (page-worthy)** — `sum(rate(dlq_writes{operation=~"Timer(Active|Standby)TaskActivity(RetryTimer|Timeout)|Transfer(Active|Standby)Task(Activity|WorkflowTask)"}[$__rate_interval])) by (operation)`. The execution-stranding subset only; backs new essential alert 80.
  - **Dead-Lettered Tasks — Informational** — same metric filtered to `VisibilityTask.*|Timer(Active|Standby)TaskDeleteHistoryEvent|Timer(Active|Standby)TaskWorkflowTaskTimeout`; these do **not** strand a running execution (WFT timeouts are covered by alerts 56/76), so they are graphed but not paged.
  - **Task Terminal Failures (all DLQ paths)** — `sum(rate(task_terminal_failures[$__rate_interval]))`; covers the 70-attempt threshold, terminal/corruption, and `history.TaskDLQErrorPattern` paths.
  - **Leading Indicator — Unexpected Errors on Stranding Task Types** — `sum(rate(task_errors{operation=~<stranding set>}[$__rate_interval])) by (operation)`; the precursor that accumulates toward the 70-attempt threshold, so a sustained climb here precedes any DLQ write. Metric names and `operation` tag values verified against server source (`service/history/queues/dlq_writer.go`, `common/metrics/metric_defs.go`). **Note:** raw `dlq_writes` totals are dominated by non-stranding visibility/retention writes — always filter by `operation`.

## v2.9.0 — 2026-08-02

### Removed
- **Matching Task Queue Info** group: removed the **Sync Throttle Count** panel (`sync_throttle_count`). That metric is emitted only by the classic `TaskMatcher`; the priority matcher — the default since server **v1.31.0** (`matching.useNewMatcher`, added v1.28.0, defaulted on in v1.31.0) — does not emit it and has no replacement. On any default modern cluster the panel was permanently empty, which reads as false reassurance ("no throttling") on a metric that simply doesn't exist. Sync-match (path 1) saturation has no direct metric on the priority matcher; infer it from Approximate Task Backlog and Async Match Latency. Task Write Throttle Count moved into the freed grid slot. Classic-matcher operators can still alert on `sync_throttle_count` directly (alert `temporal-alert-074`); see the Changing Task Queue Partitions playbook's matcher-selection section.

## v2.8.1 — 2026-08-02

### Fixed
- **Transfer Active Task Errors Workflow Busy** panel: corrected the `resource_exhausted_cause` filter value from the full enum name `RESOURCE_EXHAUSTED_CAUSE_BUSY_WORKFLOW` to the emitted short form **`BusyWorkflow`**. Temporal's `ResourceExhaustedCause` enum defines a custom `String()` (`go.temporal.io/api` `enums/v1/failed_cause.pb.go`) that renders the short CamelCase form, so the tag value in Prometheus is `BusyWorkflow`. As shipped in v2.8.0 the filter matched nothing and the panel still read flat. Verified against source and against a live cluster.

## v2.8.0 — 2026-08-02

### Fixed
- **Busy Workflow Throttling** group: **Transfer Active Task Errors Workflow Busy** panel now queries `task_errors_throttled{operation=~"TransferActive.*",resource_exhausted_cause="RESOURCE_EXHAUSTED_CAUSE_BUSY_WORKFLOW"}` instead of `task_errors_workflow_busy`. The `task_errors_workflow_busy` counter is defined in server source (`common/metrics/metric_defs.go`) but never emitted — it has no `.Record()` call site — so the panel read flat on all versions. The busy-workflow condition is now surfaced by `task_errors_throttled` with cause `RESOURCE_EXHAUSTED_CAUSE_BUSY_WORKFLOW` (`service/history/queues/executable.go`). The `operation=~"TransferActive.*"` scope is preserved: `task_errors_throttled` carries an `operation` tag whose transfer-active values (`TransferActiveTaskActivity`, `TransferActiveTaskWorkflowTask`, …) still match the filter.

## v2.7.0 — 2026-07-27

### Added
- **SDK Workers Info** group: new **Workflow Task Schedule-to-Start Timeouts (sticky fallback)** panel — `sum(rate(schedule_to_start_timeout{operation="TimerActiveTaskWorkflowTaskTimeout",namespace="$namespace"}[$__rate_interval])) by (namespace, operation)`. Placed directly after **Workflow Task StartToClose Timeouts (sticky tq)**. Fires when a workflow task on a sticky task queue is not picked up within the sticky ScheduleToStart timeout (server default 5s, `service/history/tasks/workflow_task_timer.go`) and is rescheduled onto the normal task queue. Normal-queue workflow tasks carry no ScheduleToStart timeout, so this operation isolates the sticky fallback. The separate matching-side fast path (`StickyWorkerUnavailable`, returned after the ~10s `stickyPollerUnavailableWindow`, `service/matching/matching_engine.go`) has no dedicated counter and is not graphable.

## v2.6.0 — 2026-07-02

### Added
- **Shard Movement** group: new **Owned Shards (Total)** panel — `sum(numshards_gauge{service_name="history"})`. Should equal the cluster's configured total shard count at all times; a sustained deficit means a shard has no owner. Backs new essential alerts 78/79.

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
