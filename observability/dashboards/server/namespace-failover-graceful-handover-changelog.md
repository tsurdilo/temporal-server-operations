# Changelog — Temporal Namespace Failover — Graceful Handover Dashboard

## v1.8.1 — 2026-11-21

### Fixed

- **Replication Tasks Sent per Second (Active) (117)** — description gives both causes of flat sending while lag rises, not only the standby.
- **Catchup Progress (%) (201)** — description says the panel does not filter by namespace and the label exists only from v1.32.0.

### Documentation

- Readme: **Replication Latency p99** — "task-count" corrected to task-ID distance; "guaranteed to roll back" replaced by the drain-step rule.
- Readme: **Catchup Progress %** and **Handover Progress — Catchup + Drain Combined** — metric lines no longer show a namespace filter the queries do not have.

## v1.8.0 — 2026-11-19

### Added

- **Row 1 — Pre-Flight**: **Replication Tasks Sent per Second (Active) (117)** stat — replication tasks the active is sending (`replication_tasks_send`, stream status messages left out).

### Changed

- **Replication Lag — Task-ID Distance ($p) (110)** — was **Replication Lag ($p task count)**; reads the active cluster, `increase` over 10 minutes, thresholds removed, description says the value is a task-ID distance.
- **Replication Lag Trend (Pre-Flight) (111)** — reads the active cluster, `increase` over 10 minutes.
- **Replication Lag — New Active → Old Active (Reverse Stream) (504)** — reads the new active (`$standby_cluster`), `increase` over 10 minutes.
- **Receive Buffer Full (Standby) (113)** — was **Send Channel Full (Active)**; reads the standby cluster.
- **Replication Lag — Task-ID Distance ($p) (110)** narrowed to 6 columns; **Replication Tasks Sent per Second (Active) (117)** and **Receive Buffer Full (Standby) (113)** widened to 6 so their titles fit.

### Removed

- **Send Backlog (Active) (112)** — always empty on streaming replication; replaced by **Replication Tasks Sent per Second (Active) (117)**.

## v1.7.0 — 2026-11-16

### Added

- **Row 1 — Pre-Flight**: **Replication DLQ Writes (116)** stat — replication updates the standby wrote to its replication DLQ (`dlq_writes`, `task_category="replication"`), counted since each history pod started.
- **Row 4 — Post-Handover Health**: **Replication DLQ Writes (Previously Active Cluster) (507)** time series — the same count on the cluster the namespace was handed over from.

### Changed

- **Row 1 — Pre-Flight**: **Standby Task Discards (114)** narrowed from 6 to 3 columns to make room for panel 116.
- **Row 3 — Flip Confirmed**: **UNAVAILABLE Errors Gone (402)**, **Forwarding Active (403)** and **Version Fence Fired (404)** widened from 6 to 8 columns to fill the row.

### Removed

- **Row 3 — Flip Confirmed**: **HANDOVER State Exited (401)**. Row 2b's **HANDOVER Drain Progress (%)** shows the same gauge.

### Fixed

- Template variables `$active_cluster` and `$standby_cluster` read `label_values(replication_tasks_lag_count, cluster)` instead of the bare histogram name, which matched nothing.
- **Row 1 — Pre-Flight**: **Replication Latency — Time Behind (Standby) (115)** moved out of the Row 1 header's panel list, so it is displayed.
- **Row 3 — Flip Confirmed**: **Forwarding Active (403)** and **Version Fence Fired (404)** show 0 in blue instead of red or "No data".
- **Row 4 — Post-Handover Health**: **Reverse Replication Active (New Active → Old Active) (506)** reads `replication_latency_count` on the old active from the new active instead of `replication_tasks_applied`.

## v1.6.0 — 2026-08-24

### Changed

- **Row 1 — Pre-Flight**: **Stream Errors (gRPC)** panel (id 104) threshold changed from **red ≥ 1** to **orange ≥ 1**. A steady non-zero `replication_stream_error` rate is expected — it is the cross-cluster connection being recycled by `frontend.keepAliveMaxConnectionAge` (default 5m), and the stream re-establishes within ~2s. Reconnect activity is advisory, never a red hard-blocker on its own; the real "stream broken" signal is **Stream Stuck** (`replication_stream_stuck`). Panel description updated to match.
- Alignment with the demotion of alert **FAILOVER-PRE-03** from critical to warning (advisory reconnect-rate signal), and the corresponding playbook §1.2 reframe.
- Dashboard `version` incremented.

## v1.5.0 — 2026-08-24

### Added

- **Row 1 — Pre-Flight**: new **Replication Latency — Time Behind (Standby)** full-width time series panel (`replication_latency` p50 and p99 on `$standby_cluster`). Thresholds: green, orange ≥ 5s, red ≥ 20s (red aligns with alert `FAILOVER-PRE-07`). This is the time-based "how many seconds is the standby behind" signal — the dashboard previously tracked lag only in task count (`replication_tasks_lag`). Use it to distinguish benign replication-stream reconnect churn from churn that is actually keeping the standby behind. See playbook section 1.3.

### Changed

- **Row 1 — Pre-Flight**: added a panel row below the lag gauge/trend; Rows 2a–4 shifted down accordingly. No existing panels changed.
- Dashboard `version` incremented.

---

## v1.4.0 — 2026-06-26

### Added

- **Row 4 — Post-Handover Health**: new **Reverse Replication Active** time series panel (`replication_tasks_applied` rate on `$standby_cluster`). Confirms the reverse replication stream from the new active to the old active is established and flowing post-flip. Full-width panel below Forwarding Rate.

### Changed

- Dashboard `version` incremented.

---

## v1.3.0 — 2026-06-25

### Added

- **Row 1 — Pre-Flight**: new **Standby Task Discards** stat panel (`task_errors_discarded` rate, unfiltered). Thresholds: green = 0, red ≥ 1. Any non-zero rate = workflow executions accumulating that will be stuck post-failover. Links to playbook section 1.7 for Loki workflow ID lookup. Alert `FAILOVER-PRE-06`.

### Changed

- **Row 1 — Pre-Flight**: existing 6 stat panels resized from w=4 to w=3 to accommodate the new Standby Task Discards panel at w=6.
- Dashboard `version` incremented.

---

## v1.2.0 — 2026-06-25

### Added

- **Row 1 — Pre-Flight**: new **Send Channel Full (Active)** stat panel (`replication_stream_channel_full` rate on `$active_cluster`). Thresholds: green = 0, red ≥ 1. Stacked below the Send Backlog panel in the same column. Fires when the sender's internal dispatch channel is saturated — the earliest sender-side capacity signal, preceding a rising send backlog.

### Changed

- **Row 1 — Pre-Flight**: Send Backlog (Active) panel height reduced from 7 to 3 rows to make room for the stacked Send Channel Full panel below it (combined height 3+4=7, matching the adjacent gauge and time series).
- Dashboard `version` incremented.

---

## v1.1.0 — 2026-06-25

### Added

- **Row 1 — Pre-Flight**: new **Send Backlog (Active)** stat panel (`replication_task_send_backlog` p99 on `$active_cluster`). Thresholds: green 0–9, orange ≥ 10, red ≥ 100. Positioned between the Replication Lag gauge and the Replication Lag Trend time series — diagnostic for the "lag is flat" case: non-zero means the sender is queued and raising `history.ReplicationStreamSenderHighPriorityQPS` will help immediately.

### Changed

- **Row 1 — Pre-Flight**: Replication Lag Trend time series width reduced from 16 to 12 columns to accommodate the new Send Backlog panel.
- Dashboard `version` incremented.

---

## v1.0.0 — 2026-06-25

Initial release.

### Added

- **Row 1 — Pre-Flight**: go/no-go stat panels for stream health (`replication_stream_stuck`, `replication_dlq_enqueue_failed`), receiver backlog depth (p99, orange ≥ 400 / red ≥ 500), gRPC and service stream errors, backfill activity rate, and a replication lag gauge + 15-minute trend time series. Hard blockers highlighted red.
- **Row 2a — WaitReplication**: catchup progress gauge (`catchup_ready_shard_count / $total_shards`, Steps 1–3, no client impact) and combined catchup + drain time series headline panel with `$total_shards` reference line.
- **Row 2b — HANDOVER Drain**: drain progress gauge (`handover_ready_shard_count / $total_shards`), UNAVAILABLE error burst panel (`client_redirection_errors{error_type="Unavailable"}`, expected spike), and namespace handover task error counter (`task_errors_namespace_handover`).
- **Row 3 — Flip Confirmed**: four stat panels confirming HANDOVER state exited (`handover_ready_shard_count = 0`), UNAVAILABLE errors cleared, forwarding active (`client_redirection_requests > 0`), and version fence fired (`task_errors_version_mismatch > 0`).
- **Row 4 — Post-Handover Health**: version mismatch decay, forwarding error rate broken out by `error_type`, WFT schedule-to-start latency on new active (orange > 1s / red > 5s), reverse replication lag on old active, and forwarding success rate.
- **Annotations**: HANDOVER Active (red band, fires when `handover_ready_shard_count > 0`) and Flip Confirmed (green line, fires on `increase(task_errors_version_mismatch)`).
- **Template variables**: `$active_cluster`, `$standby_cluster`, `$namespace`, `$total_shards` (custom 512/1024/2048/4096, default 2048), `$p` (percentile, default 0.99).

### Known limitations

- `catchup_ready_shard_count` has no namespace label in the base server code (server PR pending as of 2026-06-25). The catchup progress gauge aggregates all global namespaces. With a single global namespace the reading is accurate; with multiple, use the combined time series panel to monitor relative progress.
