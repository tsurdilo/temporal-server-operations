# Changelog — Temporal Server Dashboard

## v2.15.2 — 2026-09-16

### Changed
- **Visibility row description rewritten.** It said the row "tracks your Temporal Visibility store latencies and availability", which stopped being true in v2.15.0 when the row grew from 8 panels to 19. It now names what the row actually covers — write and read paths, per-store health under dual visibility, how close visibility tasks are to being dropped, and Elasticsearch bulk write health — and states the store-attribution rule up front, because that is the thing operators get wrong: only the `visibility_persistence_*` panels carry a store label.

### Documentation
- **Store-blindness now noted on all three ES panels, not just one.** v2.15.1 documented it on **ES Bulk Processor Errors by HTTP Status** only. It is structural and applies to the whole family: `NewProcessor` tags its metrics handler with the operation alone, and the `visibility_index_name` tag is applied around the visibility *manager*, not the store's bulk processor. **ES Bulk Processor Queue Depth** and **ES Write Confirm Latency vs Ack Timeout** carry the same caveat now.
- **Readme Visibility section opens with an attribution note** explaining which metric family can identify a store and which cannot, so the per-panel caveats have context.
- No panel, query, threshold or layout changes. A dashboard on v2.15.1 shows identical data; upgrade only for the corrected descriptions.

## v2.15.1 — 2026-09-15

### Fixed
All six corrections below came from running the v2.15.0 panels against a live dual-Elasticsearch cluster (one ES cluster, two indices). Every one of them was a claim that survived source review and PromQL validation but turned out to be wrong, or incomplete, against real data.

- **`error_type` label values were wrong in every panel description that named one.** The metrics exporter renders the Go error type with dots replaced by underscores, so the queryable values are **`persistence_TimeoutError`** and **`serviceerror_Unavailable`** — not `TimeoutError` and `Unavailable`. Anyone following the old descriptions was querying values that match nothing. Corrected on Visibility Errors by Type per Store, and the readme now states the naming rule rather than just listing examples.
- **ES Bulk Processor Errors by HTTP Status: `0` means Elasticsearch was *gone*, not slow — and the panel is empty when it is hung.** The bulk processor only records a status once Elasticsearch answers. A container that is paused (reachable, not responding) never completes the bulk, so no status is recorded and the panel stays flat. Confirmed both ways: `docker pause` → this panel empty, Write Error Rate empty, only Visibility Errors by Type moved (`persistence_TimeoutError`); `docker stop` → this panel `http_status=0` and Write Error Rate fired. The description now says a flat panel here is not an all-clear.
- **Visibility Tasks Dead-Lettered by Task Type over-reports data loss on Elasticsearch.** The bulk processor holds documents in memory. The store gives up at `worker.ESProcessorAckTimeout` and fails the task — possibly into the DLQ — but the document is still buffered and gets flushed once Elasticsearch recovers. Observed directly: twelve tasks dead-lettered during an outage (`dlq_writes` = 12, `tdbg dlq list` = 12 messages), and after recovery **all twelve records were present in both indices**. The description now says to verify against the store before assuming loss, and notes that SQL has no such buffer so a DLQ'd task there really did lose the write.
- **Visibility Task Failures & Internal Errors only moves while tasks are still retrying.** It counts failing attempts, so once tasks exhaust `history.TaskDLQUnexpectedErrorAttempts` and are set aside the line returns to zero while writes remain completely broken. Observed with the attempt limit at 3: `task_errors_internal` back to zero within a minute while not one of five new workflows reached either store. The description now names the durable signals instead — Write Request Rate flat on both stores, and new workflows never appearing.
- **Visibility Task Retry Depth: corrected in v2.15.0's own entry.** `task_attempt` is recorded by every task on completion (`Ack`), not only past 30 attempts, so a healthy cluster reads about 1 rather than being empty. Restated here because the live run confirmed it: the top bucket read 5 for tasks that had actually made ~3 attempts, which is the histogram bucket rounding in action.
- **Readme: `--dlq-type` takes the numeric category id, not a name.** `tdbg dlq read --dlq-type visibility` fails with `strconv.Atoi: parsing "visibility": invalid syntax`; the history task DLQ wants **4** for visibility. Also noted: without `--last-message-id` the command prompts, and with no terminal attached that prompt panics on EOF.

## v2.15.0 — 2026-09-15

### Added
- **Visibility group — 11 panels for running two visibility stores (dual visibility), and for Elasticsearch write-path health generally.** Backs the rewritten **[Dual Visibility playbook](../../../playbooks/dual-visibility.md)**. Before this release the group had three store-aware panels (2117–2119), all filtered `service_name="history"` — so the dashboard could only show **writes**, and there was no per-store view of the read path anywhere in the repo. Every metric name, tag and default below was verified against server source.
  - **Read path (new — previously no coverage at all).** Reads are issued by frontend, matching and worker, never by history, so nothing in the old group saw them. All three split by `visibility_index_name` and `service_name`, matching `operation=~"ListWorkflowExecutions|CountWorkflowExecutions|GetWorkflowExecution|ListChasmExecutions|CountChasmExecutions"`.
    - **Visibility Read Request Rate per Store** — `sum(rate(visibility_persistence_requests{…}[$__rate_interval])) by (visibility_index_name, service_name)`. Doubles as a read-routing readout: only the primary reports by default; a namespace with `system.enableReadFromSecondaryVisibility` set moves to the secondary; **both** stores reporting means `system.visibilityEnableShadowReadMode` is on.
    - **Visibility Read Error Rate per Store** — read failures per store. Alerts 059a/059b/059c filter `service_name="history"` and therefore **never fire on read failures**; this panel is the only per-store view of a broken workflow list. Orange 0.1 / red 1 req/s.
    - **Visibility Read Latency per Store** — selected percentile per store. Pairs with shadow read mode, whose results are discarded and so are observable only through metrics. Orange 3s / red 5s.
  - **Complete error picture — closes a structural blind spot in panel 2118.** `visibility_persistence_errors` is the `default` arm of the server's visibility error classifier and **skips** `TimeoutError`, `ResourceExhausted`, `NotFound`, `InvalidArgument` and `ConditionFailedError` (verified in `updateErrorMetric`). A flat 2118 is therefore not proof of health.
    - **Visibility Errors by Type per Store** — `sum(rate(visibility_persistence_error_with_type[$__rate_interval])) by (visibility_index_name, error_type, service_name)`. Counts **every** error. `error_type="TimeoutError"` is the only signal for an Elasticsearch write never confirmed inside `worker.ESProcessorAckTimeout` (30s default) — invisible on 2118. Orange 0.1 / red 1 req/s.
    - **Visibility Rate Limit Rejections per Store** — `visibility_persistence_resource_exhausted` by store and cause. `system.visibilityPersistenceMaxWriteQPS` / `MaxReadQPS` (both 9000) are applied **per store**, each store getting its own budget from the same setting. Also absent from 2118.
  - **Data-loss window.** Visibility tasks do **not** retry forever: a store error, an ES ack timeout or an ES rejection counts as an unexpected error, and at `history.TaskDLQUnexpectedErrorAttempts` (70, default-on via `history.TaskDLQEnabled=true`) the task is dead-lettered. With default retry gaps (1s, ×1.1, 180s cap, jittered to 80–100%) that is roughly **70 minutes**.
    - **Visibility Task Retry Depth (approaching DLQ)** — `histogram_quantile(1.00, sum by (operation, le) (rate(task_attempt_bucket{operation=~"VisibilityTask.*"}[$__rate_interval])))`. `task_attempt` is a dimensionless **histogram**, so the top bucket stands in for "deepest attempt count" and the reading is coarse (buckets step 1/2/5/10/20/50/100, so 35 attempts reads as 50 and 70 reads as 100, and the thresholds trigger on the bucket above) — same caveat as the hot-shard `max` panels from v2.14.0. Note it is recorded by **every** task on completion (`Ack`), not only past 30 attempts — the >30 gate is an additional in-flight emission — so the panel is populated on a healthy cluster at about 1 and the signal is the line climbing, not the presence of data. Thresholds mark 30 and 70 (records start being dropped).
    - **Visibility Tasks Dead-Lettered by Task Type** — `dlq_writes{operation=~"VisibilityTask.*"}` by operation. Narrower and more actionable than panels 2202/2203: 2202 matches only timer and transfer operations and will **never** show visibility, while 2203 blends visibility with retention and workflow-task-timeout. Task type decides whether the loss is permanent — a dropped `VisibilityTaskStartExecution` or `…UpsertExecution` is rebuilt by the workflow's next visibility write (every write is a full snapshot versioned on the task id), but a dropped `…CloseExecution` is the last task that workflow ever emits, and a dropped `…DeleteExecution` leaves a record that outlives its workflow.
    - **Visibility Task Failures & Internal Errors** — `task_errors` and `task_errors_internal` on `VisibilityTask.*` (note the emitted name is `task_errors`; the Go constant is `TaskFailures`). `task_errors_internal` rising while panel 2117 sits at zero on **both** stores is the signature of an invalid `system.secondaryVisibilityWritingMode`: only `off`, `on`, `dual` are accepted, anything else is rejected *above* the per-store metrics layer, so **no** `visibility_persistence_*` metric fires and the dashboard merely looks quiet. The config loader accepts any string with no warning and `temporal-server validate-dynamic-config` passes it. Reads keep working throughout.
  - **Elasticsearch write path (new — previously no coverage at all).** ES writes are asynchronous through a per-store bulk processor, built only by the history service. All three panels are labelled **(Elasticsearch only)** in the title and are empty on SQL clusters. They carry **no** `visibility_index_name`, so two ES stores collapse into one series — use Visibility Errors by Type per Store to attribute per store.
    - **ES Bulk Processor Errors by HTTP Status (Elasticsearch only)** — `elasticsearch_bulk_processor_errors` by `http_status`. `0` = cluster unreachable, `429` = overloaded, `400` = mapping, `404` = index missing (never self-recovers; dead-letters everything after ~70 min).
    - **ES Bulk Processor Queue Depth (Elasticsearch only)** — `elasticsearch_bulk_processor_queued_requests`, also a dimensionless histogram, read as the top bucket. Handing a document over **blocks** once the processor is busy, stalling history's visibility workers. Tuning knobs `worker.ESProcessorBulkActions` (500), `BulkSize` (16MB), `FlushInterval` (1s), `NumOfWorkers` (2) are read by **history** despite the `worker.` prefix, and unlike `ESProcessorAckTimeout` they are snapshotted at store construction — **a history restart is required** for them to take effect.
    - **ES Write Confirm Latency vs Ack Timeout (Elasticsearch only)** — `elasticsearch_bulk_processor_request_latency`, red line at 30s = `worker.ESProcessorAckTimeout`. Reaching it fails the visibility task as a timeout that 2118 does not record but which still counts toward the 70-attempt DLQ threshold.

### Fixed
- **Readme: corrected "History retries indefinitely" on Visibility Write Error Rate per Store.** It does not. Visibility tasks are dead-lettered after `history.TaskDLQUnexpectedErrorAttempts` unexpected attempts (70 by default, ~70 minutes), after which the record is no longer written to that store. The readme now states the real behaviour and the per-task-type consequences.
- **Readme: Visibility Write Error Rate per Store no longer described as "the primary alert signal."** It counts only a subset of visibility errors; the readme now says a flat line is not proof of health and points at Visibility Errors by Type per Store.
- **Readme: dropped `ScanWorkflowExecutions` from the Visibility Availability description** — that operation scope is defined in server source but never emitted.
- **Readme: every panel in the Visibility group now carries an explicit store-type column** (`SQL + ES` or `Elasticsearch only`), so it is clear which panels are empty on a SQL cluster.

### Known gaps
- Panels 2130–2132 carry no `visibility_index_name` label, so with **two** Elasticsearch stores they cannot be attributed to one store. This is a server-side metric limitation, not a dashboard one.
- No alert exists for visibility tasks being dead-lettered (`dlq_writes{operation=~"VisibilityTask.*"}`) or for `visibility_persistence_error_with_type`. Alert 080 matches only timer and transfer operations.
- The Elasticsearch panels are derived from server source but have not yet been confirmed against a live Elasticsearch outage.

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
