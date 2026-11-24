# Changelog — Temporal Standby Cluster — Replication Health Dashboard

## v2.4.1 — 2026-11-21

### Documentation

- **Replication Tasks Lag (1011)**: values above 100,000 show as exactly 100K; high percentiles sit at the cap after start-up and outages.

## v2.4.0 — 2026-11-19

### Fixed

- **Replication Tasks Lag (1011)**: reads a percentile over the last 10 minutes instead of an
  average of the metric's sum, which reads +Inf; orange and red thresholds removed.

### Documentation

- **Replication Tasks Lag (1011)**: a distance in task IDs, not a count of replication tasks;
  sampled on the cluster that holds the outgoing tasks, only when cleanup runs; stops while a remote
  cluster is not acknowledging.
- **Send Backlog Depth (1013)**: always empty on streaming replication.
- **Sender Rate Limit Latency (Active → Standby) (1015)**: measures a limiter that lets every task
  through, for low-priority sends only.
- **Replication Task Throughput (Recv vs Applied vs Failed) (1021)**: the applied series is empty
  on streaming replication.

## v2.3.0 — 2026-10-01

The whole Replication DLQ section was wrong in four ways at once, and three of them were silent.
It described the wrong DLQ, it was labelled Cassandra-only on all five panels, one panel read a
metric that does not exist in server source, and another aggregated a counter as if it were a
gauge so it could never return to zero.

### Fixed

- **The section described the wrong mechanism.** These panels show the **replication task DLQ** —
  where a replication task is parked after failing to apply
  `history.ReplicationTaskProcessorErrorRetryMaxAttempts` times (default 80). The text panel,
  the readme and the config table all attributed them to the **history task** DLQ and named
  `history.TaskDLQEnabled`, which has no effect on any of them.

- **"⚠️ Cassandra Only" was false** and is removed from the row title and all five panel titles,
  two legends, the text panel and the readme. `replication_tasks_dlq` exists in the PostgreSQL,
  MySQL and SQLite schemas with insert, read and delete paths in each plugin, and every metric in
  the section is emitted from generic replication and shard code with no store gate. The claim
  also appeared inverted beside the namespace replication panels, which said that queue was "not
  Cassandra-specific" *unlike* the history task DLQ — implying a distinction that does not exist.

- **Replication DLQ Enqueue Failures (1032)** read `replication_dlq_failed`, which has never
  existed in server source on any branch. The real metric is `replication_dlq_enqueue_failed`,
  present since the original Domain Replication DLQ commit and first tagged **v0.10.0** — so
  there is no version floor. One missing word meant this panel had never returned a point.

- **Replication DLQ Non-Empty Observations (1031)** did `sum(replication_dlq_non_empty)`.
  That metric is a **counter**, incremented once per periodic check that finds a non-empty DLQ,
  so the sum only ever climbed: once a DLQ had been non-empty even briefly the panel showed alarm
  permanently until the pod restarted. Now `sum(increase(...[11m]))`, which falls back to zero
  when the DLQ drains. The `[11m]` window is deliberate — the check runs on a 5-minute interval
  with **full jitter**, so gaps reach 5 minutes and anything shorter reads zero at random. Same
  reasoning as the fixed windows on the server dashboard's queue-lag panels.

- **Legends on 1031 and 1032** were `{{instance}}`, which renders empty because `sum()` drops
  every label. They now carry literal labels.

### Changed

- **The two level gauges and the depth timeseries** are retitled and re-described as **task-ID
  positions**, not message counts. Task IDs are allocated in per-shard ranges and are not
  contiguous, so the distance between max and ack says the DLQ is behind, not by how much — the
  old readme stated outright that the gap "is the number of unprocessed DLQ tasks", which it is
  not. **Replication DLQ Max Level** also only updates at the moment of a DLQ write, so it goes
  stale rather than falling: a flat line means no new dead-lettering, not an empty DLQ.

- **Replication DLQ Task-ID Distance (Max Minus Ack) (1035)** now records that both series are a
  `max` across all shards and source clusters while the `instance` tag is derived differently on
  each metric — the ack gauge tags the shard owning the DLQ, the max gauge a shard id derived
  from the source — so the two maxima may not come from the same shard. It is a directional
  signal; contents are counted with `tdbg dlq`. A per-shard subtraction was considered and not
  done, because the two tags could not be confirmed to mean the same thing.

### Panel titles changed

Five panels and one row were retitled. Nothing in `playbooks/` or `observability/alerts/server/`
referenced the old titles, and the only inbound link to the renamed section anchor was this
dashboard readme's own table of contents, updated in the same change.

| Old | New |
|---|---|
| 💀 Replication DLQ (⚠️ Cassandra Only) *(row)* | Replication DLQ |
| DLQ Non-Empty ⚠️ Cassandra Only | Replication DLQ Non-Empty Observations (11m) |
| DLQ Enqueue Failures ⚠️ Cassandra Only | Replication DLQ Enqueue Failures |
| DLQ Max Level ⚠️ Cassandra Only | Replication DLQ Max Level |
| DLQ Ack Level ⚠️ Cassandra Only | Replication DLQ Ack Level |
| DLQ Depth Over Time (Max - Ack Level) ⚠️ Cassandra Only | Replication DLQ Task-ID Distance (Max Minus Ack) |

---

## v2.2.0 — 2026-08-24

- Added **🧹 History Scavenger** row (appended last): surfaces whether this standby's history scavenger is keeping up with leftover-history cleanup. The scavenger skips branches younger than `worker.historyScannerDataMinAge` (default 60 days), so for short-lived, high-volume workflows it can skip nearly everything and let `history_tree` / `history_node` grow. Two panels — **Scavenger Activity — Skipped vs Handled** (`scavenger_skips` vs `scavenger_success`) and **Scavenger Errors** (`scavenger_errors`), all `operation="HistoryScavenger"`. Metric names and semantics verified against server source (`service/worker/scanner/history/scavenger.go`); `scavenger_success` = branches handled (kept or deleted), not deletions. Backs the new **[XDC Standby Database Growth on SQL playbook](../../../playbooks/xdc-standby-database-growth-sql.md)**.

## v2.1.0 — 2026-08-24

- Added **Replication Latency — Time Behind (End-to-End)** panel (id 1017) to the 📉 Replication Lag row. Dedicated view of `replication_latency` (create→apply wall-clock, p50 and p99) — the single best "how many seconds is the standby behind" signal. Red threshold at 20s aligns with alert `FAILOVER-PRE-07`. The metric was previously only visible as one of five series in the combined **Replication Latencies** panel.

## v2.0.0 — 2026-05-12

First versioned release. Prior changes were unversioned.
