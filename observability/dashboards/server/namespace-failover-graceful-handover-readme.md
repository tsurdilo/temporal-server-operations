# Temporal Namespace Failover — Graceful Handover Dashboard

This dashboard is an operator tool for executing and monitoring a planned graceful namespace failover using the `namespace-handover-v2` workflow. Each row group maps to a phase of the handover — a row becomes relevant when the workflow enters that phase.

Throughout this document, **Step N** refers to the internal activity steps of the `namespace-handover-v2` workflow:

| Step | Activity | Phase |
|---|---|---|
| 1 | `GetMetadata` | WaitReplication |
| 2 | `GetMaxReplicationTaskIDs` | WaitReplication |
| 3 | `WaitReplication` loop | WaitReplication |
| 4 | `UpdateNamespaceState(HANDOVER)` | HANDOVER Drain — write unavailability window opens |
| 5 | `WaitHandover` loop | HANDOVER Drain — hard 30s cap before automatic rollback |
| 6 | `UpdateActiveCluster` | Flip Confirmed |

> **Compatibility:** Temporal Server v1.20+ · Grafana 9.0+ · Prometheus

> **Current version:** v1.8.1 — see [CHANGELOG](./namespace-failover-graceful-handover-changelog.md)

The matching playbook is at [`playbooks/namespace-failover-graceful-handover.md`](../../../playbooks/namespace-failover-graceful-handover.md).

---

## Table of Contents

- [Template Variables](#template-variables)
- [Annotations](#annotations)
- [Row 1 — Pre-Flight (Go / No-Go)](#row-1--pre-flight-go--no-go)
  - [Stream Stuck](#panel-stream-stuck-stat)
  - [DLQ Enqueue Failed](#panel-dlq-enqueue-failed-stat)
  - [Receiver Backlog Depth](#panel-receiver-backlog-depth-stat)
  - [Stream Errors — gRPC](#panel-stream-errors--grpc-stat)
  - [Stream Service Errors](#panel-stream-service-errors-stat)
  - [Backfill Activity](#panel-backfill-activity-stat)
  - [Standby Task Discards](#panel-standby-task-discards-stat)
  - [Replication DLQ Writes](#panel-replication-dlq-writes-stat)
  - [Replication Lag](#panel-replication-lag-gauge)
  - [Replication Tasks Sent per Second (Active)](#panel-replication-tasks-sent-per-second-active-stat)
  - [Receive Buffer Full (Standby)](#panel-receive-buffer-full-standby-stat)
  - [Replication Lag Trend](#panel-replication-lag-trend-time-series)
  - [Replication Latency p99](#panel-replication-latency-p99-time-series)
- [Row 2a — WaitReplication (Steps 1–3)](#row-2a--waitreplication-steps-13--no-client-impact)
  - [Catchup Progress %](#panel-catchup-progress--gauge)
  - [Handover Progress — Combined](#panel-handover-progress--catchup--drain-combined-time-series)
- [Row 2b — HANDOVER Drain (Steps 4–5)](#row-2b--handover-drain-steps-45)
  - [HANDOVER Drain Progress %](#panel-handover-drain-progress--gauge)
  - [UNAVAILABLE Error Burst](#panel-unavailable-error-burst-expected-time-series)
  - [Namespace Handover Task Errors](#panel-namespace-handover-task-errors-time-series)
- [Row 3 — Flip Confirmed (Step 6)](#row-3--flip-confirmed-step-6)
  - [UNAVAILABLE Errors Gone](#panel-unavailable-errors-gone-stat)
  - [Forwarding Active](#panel-forwarding-active-stat)
  - [Version Fence Fired](#panel-version-fence-fired-stat)
- [Row 4 — Post-Handover Health](#row-4--post-handover-health)
  - [Version Mismatch Decay](#panel-version-mismatch-decay-time-series)
  - [Forwarding Error Rate by Type](#panel-forwarding-error-rate-by-type-time-series)
  - [WFT Schedule-to-Start (New Active)](#panel-wft-schedule-to-start-new-active-time-series)
  - [Replication Lag — Reverse Stream](#panel-replication-lag--reverse-stream-time-series)
  - [Forwarding Success Rate](#panel-forwarding-success-rate-time-series)
  - [Reverse Replication Active](#panel-reverse-replication-active-time-series)
  - [Replication DLQ Writes (Previously Active Cluster)](#panel-replication-dlq-writes-previously-active-cluster-time-series)

---

## Template Variables

| Variable | Source | Notes |
|---|---|---|
| `$active_cluster` | `label_values(replication_tasks_lag_count, cluster)` | The cluster that is currently active for the namespace — the source of replication and the old active post-flip |
| `$standby_cluster` | `label_values(replication_tasks_lag_count, cluster)` | The target cluster — receives replication, becomes the new active after flip |

`replication_tasks_lag` is a histogram, so Prometheus stores it only as `replication_tasks_lag_bucket`, `_count` and `_sum`. Both cluster variables read `_count`; before v1.7.0 they read the bare name, matched nothing, and left both dropdowns empty.
| `$namespace` | `label_values(handover_ready_shard_count, namespace)` | The global namespace being failed over |
| `$total_shards` | Custom (512/1024/2048/4096) | Total shard count for this cluster. Default 2048. Must match `numHistoryShards` in cluster config |
| `$p` | Custom (0.50–0.999) | Percentile for histogram panels. Default 0.99 |

**Prerequisites — the `cluster` label must be present on all metrics.**

The Temporal server does not emit a `cluster` label on its metrics by default. The `cluster` label used throughout this dashboard's PromQL expressions must be added explicitly. Without it, all `cluster="$active_cluster"` and `cluster="$standby_cluster"` filters return nothing and every panel is empty.

**Option 1 (recommended) — server static config.** Add a `cluster` tag directly in the Temporal server's metrics configuration. This tag is then emitted on every metric the server produces:

```yaml
metrics:
  tags:
    cluster: active        # use a different value on each cluster (e.g. "active", "standby", or your cluster names)
  prometheus:
    framework: tally       # or otel — tags field works for both
    listenAddress: "0.0.0.0:8000"
```

Set a different `cluster` value in the config on each cluster. The value can be any string — it just needs to be consistent with what you select in the `$active_cluster` and `$standby_cluster` dashboard dropdowns.

**Option 2 — Prometheus scrape config.** If you cannot change the server config, add the label as an external label on each Prometheus scrape job instead:

```yaml
scrape_configs:
  - job_name: temporal-active
    static_configs:
      - targets: ['active-frontend:8000']
        labels:
          cluster: active
  - job_name: temporal-standby
    static_configs:
      - targets: ['standby-frontend:8000']
        labels:
          cluster: standby
```

If you use service discovery (Kubernetes, Consul), add the label via `relabel_configs` with `target_label: cluster`.

---

## Annotations

| Annotation | Trigger | Color | What it marks |
|---|---|---|---|
| HANDOVER Active | `handover_ready_shard_count{namespace="$namespace"} > 0` | Red band | Write unavailability window — open until drain completes or rollback occurs |
| Flip Confirmed | `increase(task_errors_version_mismatch{cluster="$active_cluster"}[$__rate_interval]) > 0` | Green line | Old active's version fence fired — flip has occurred |

Both annotations are **off by default**. Toggle them on during a handover to overlay them on all time series panels. When both are enabled you get a clear visual timeline: the red band opens when HANDOVER drain starts, the green line fires inside it at the flip moment, and the red band closes when drain completes. This makes it easy to measure the drain window duration directly on the graphs.

---

## Row 1 — Pre-Flight (Go / No-Go)

### Panel: Stream Stuck (stat)

**Metric:** `rate(replication_stream_stuck{cluster="$standby_cluster"})`  
**Cluster:** standby  
**Thresholds:** green = 0, red > 0  
Counts replication streams that have stopped making progress on the standby cluster.

**Alert:** [FAILOVER-PRE-01](../../../observability/alerts/server/alerts-index.md#alert-failover-pre-01--replication-stream-stuck-pre-handover) — fires after 2 minutes sustained, severity critical.

See [playbook section 1.2](../../../playbooks/namespace-failover-graceful-handover.md#12-is-the-replication-stream-between-clusters-healthy) for interpretation and go/no-go guidance.

### Panel: DLQ Enqueue Failed (stat)

**Metric:** `rate(replication_dlq_enqueue_failed{cluster="$standby_cluster"})`  
**Cluster:** standby  
**Thresholds:** green = 0, red > 0  
Rate of replication tasks failing into the DLQ on the stream ingestion path on the standby cluster.

> Note: `replication_dlq_max_level` and `replication_dlq_ack_level` are not used here. This dashboard targets the replication stream path — `dlq_enqueue_failed` is the correct DLQ signal on that path.

**Alert:** [FAILOVER-PRE-02](../../../observability/alerts/server/alerts-index.md#alert-failover-pre-02--replication-dlq-enqueue-failing) — fires after 2 minutes sustained, severity critical.

See [playbook section 1.2](../../../playbooks/namespace-failover-graceful-handover.md#12-is-the-replication-stream-between-clusters-healthy) for interpretation and go/no-go guidance.

### Panel: Receiver Backlog Depth (stat)

**Metric:** `histogram_quantile(0.99, rate(replication_tasks_recv_backlog_bucket{cluster="$standby_cluster"}))`  
**Cluster:** standby  
**Thresholds:** orange ≥ 400, red ≥ 500  
p99 depth of the standby receiver's in-memory buffer — tasks that have arrived over the replication stream and are waiting to be handed to the task processor. Exceeding `history.ReplicationReceiverMaxOutstandingTaskCount` (default 500) triggers flow control: the receiver embeds a `PAUSE` command in its next ACK and the sender stops dispatching.

**Alerts:** [FAILOVER-PRE-05](../../../observability/alerts/server/alerts-index.md#alert-failover-pre-05--receiver-backlog-near-limit) (≥ 400, warning) · [FAILOVER-PRE-04](../../../observability/alerts/server/alerts-index.md#alert-failover-pre-04--receiver-backlog-at-flow-control-limit) (≥ 500, critical).

See [playbook section 1.5](../../../playbooks/namespace-failover-graceful-handover.md#15-is-the-standby-keeping-up-with-incoming-replication-tasks) for interpretation and go/no-go guidance.

### Panel: Stream Errors — gRPC (stat)

**Metric:** `rate(replication_stream_error{cluster="$standby_cluster"})`  
**Cluster:** standby  
**Thresholds:** green = 0, orange > 0 (advisory — never a red hard-blocker)  
Rate of replication stream reconnects on the standby (each `StreamError` tears the stream down and it re-establishes immediately via the outer `WrapEventLoop` loop). **A steady non-zero rate is expected**, not a fault: it is the cross-cluster connection being recycled by `frontend.keepAliveMaxConnectionAge` (default 5m), reconnecting every few minutes and resuming from its watermark within ~2s — no data lost. On its own this is advisory. It only signals a problem when paired with stalled progress: cross-check the **Stream Stuck** panel (`replication_stream_stuck` = acknowledged position not advancing — the real "stream broken" signal) and the **Replication Lag / Replication Latency** panels.

> **Note on the `cluster` label:** `cluster` here is a scrape-level label added by Prometheus configuration, not a server-emitted dimension. The actual server-emitted labels for filtering by stream direction are `from_cluster_id` and `to_cluster_id` (integer cluster IDs).

**Alert:** [FAILOVER-PRE-03](../../../observability/alerts/server/alerts-index.md#alert-failover-pre-03--replication-stream-reconnect-rate-elevated) — fires after 2 minutes sustained, severity **warning** (advisory). The hard blocker is `FAILOVER-PRE-01` (Stream Stuck).

See [playbook section 1.2](../../../playbooks/namespace-failover-graceful-handover.md#12-is-the-replication-stream-between-clusters-healthy) for interpretation and go/no-go guidance.

### Panel: Stream Service Errors (stat)

**Metric:** `rate(replication_service_error{cluster="$standby_cluster"})`  
**Cluster:** standby  
**Thresholds:** green = 0, red > 0  
Application-level errors from the replication stream event loop on the standby cluster — context cancellations, cluster metadata lookup failures, and internal scheduler state violations. Distinct from gRPC transport failures (`replication_stream_error`).

> **Note on the `cluster` label:** `cluster` here is a scrape-level label added by Prometheus configuration, not a server-emitted dimension. The actual server-emitted labels for filtering by stream direction are `from_cluster_id` and `to_cluster_id` (integer cluster IDs).

See [playbook section 1.2](../../../playbooks/namespace-failover-graceful-handover.md#12-is-the-replication-stream-between-clusters-healthy) for interpretation and go/no-go guidance.

### Panel: Backfill Activity (stat)

**Metric:** `rate(replication_tasks_back_fill{cluster="$standby_cluster"})`  
**Cluster:** standby  
**Thresholds:** no fixed threshold  
Rate of synchronous `ResendHistoryEvents` calls from the standby back to the active, fired when a replication task being applied discovers it is missing prerequisite history events. Fires deep inside task execution, entirely separate from the stream ingestion buffer (`recv_backlog`). Companion histogram: `replication_tasks_back_fill_latency`.

See [playbook section 1.4](../../../playbooks/namespace-failover-graceful-handover.md#14-is-the-standby-making-extra-round-trips-to-fetch-missing-history) for interpretation and go/no-go guidance.

### Panel: Standby Task Discards (stat)

**Metric:** `sum(rate(task_errors_discarded{}[5m]))`  
**Cluster:** standby  
**Thresholds:** green = 0, red ≥ 1  
Rate of standby transfer and timer tasks discarded because `history.standbyTaskMissingEventsDiscardDelay` (default 15 minutes) expired before the matching history events arrived from the active. Labels: `namespace`, `task_type` (e.g. `TransferWorkflowTask`, `TimerWorkflowTaskTimeoutTask`), `operation`. No WorkflowID label emitted.

**Alert:** [FAILOVER-PRE-06](../../../observability/alerts/server/alerts-index.md#alert-failover-pre-06--standby-task-discards-detected) — fires after 2 minutes sustained, severity warning.

See [playbook section 1.6](../../../playbooks/namespace-failover-graceful-handover.md#16-are-there-workflow-executions-on-the-standby-that-will-be-stuck-after-the-flip) for interpretation and go/no-go guidance.

### Panel: Replication DLQ Writes (stat)

**Metric:** `sum(dlq_writes{cluster="$standby_cluster", task_category="replication"}) or vector(0)`  
**Cluster:** standby  
**Thresholds:** green = 0, red ≥ 1  
Replication updates the standby could not apply and wrote to its replication DLQ, counted since each history pod started. Each one means the standby's copy of a workflow is out of date until the DLQ is merged, and after the flip that out-of-date copy is the one that runs.

How to read it:

- **0** is the healthy reading, and what the panel shows when nothing was ever written.
- **Any non-zero value** — list the standby's replication DLQ (`tdbg dlq list`) and merge it before handing over. The panel is the prompt to look; the DLQ listing is the answer.
- **The count does not go down when the DLQ is merged.** It only resets when a history pod restarts. A non-zero count with an empty DLQ listing means the writes were already merged.
- **It counts every namespace on the cluster** and ignores `$namespace`. `dlq_writes` has a `namespace` label only from server v1.32.0, so the panel sums across namespaces to read the same on every version.

It reads the raw counter rather than a rate on purpose. A single write is the whole signal here, and a rate over a counter misses the write that creates the series.

**Version floor:** replication updates are written to this DLQ (and counted by `dlq_writes` with `task_category`) by default from server v1.28.0. On v1.25.0–v1.27.x the panel stays 0 unless `history.enableHistoryReplicationDLQV2` is `true`; below v1.25.0 `dlq_writes` has no `task_category` label and the panel stays 0.

> Note: `dlq_message_count` (the DLQ size gauge) is not used. The server emits it only every 3 hours (not configurable), too late for a pre-flight check. **DLQ Enqueue Failed** counts only writes to the DLQ that failed, never ones that succeeded.

See [playbook section 1.2](../../../playbooks/namespace-failover-graceful-handover.md#12-is-the-replication-stream-between-clusters-healthy) for interpretation and go/no-go guidance.

### Panel: Replication Lag (gauge)

Panel title: **Replication Lag — Task-ID Distance ($p)**

**Metric:** `histogram_quantile($p, sum by (le) (increase(replication_tasks_lag_bucket{cluster="$active_cluster"}[10m])))`  
**Cluster:** active  
**Thresholds:** none (no level of this value has a fixed meaning)  
How far the standby's acknowledgements trail the active, as a **distance in task IDs**, p`$p` across shards over the last 10 minutes. It is not a count of replication tasks waiting. Each shard numbers every history task it creates, of any kind, from one sequence, so the value also grows with the active's other task writes. On a test cluster under steady load, the active read under 1,000 throughout.

The active emits it only when its replication task cleanup advances, about once a minute per shard. When the standby stops acknowledging, cleanup stops advancing and **no new values arrive**; the value does not climb. Because the panel looks back 10 minutes, it keeps showing the last values during a short outage and goes empty only after about 10 minutes without acknowledgements. On a test cluster, samples stopped about 2 minutes into a 4-minute standby outage, and once the standby was back the value jumped to the 100K cap (see below).

Values above 100,000 show as exactly 100K, the top of the metric's buckets, so a reading of 100K means "100K or more". On a test cluster p99 sat at the cap for about 10 minutes after the clusters started, after a 4-minute standby outage, and on the reverse stream right after a handover, until those samples left the 10-minute window; p50 stayed low (about 200 on the reverse stream with load on the new active). Read p50 for the usual level and p99 for spikes. The same applies to [Replication Lag Trend](#panel-replication-lag-trend-time-series) and [Replication Lag — Reverse Stream](#panel-replication-lag--reverse-stream-time-series).

Use it for direction: rising, falling, flat. For how far behind the standby is in seconds, which is what the 30-second drain window depends on, use [Replication Latency p99](#panel-replication-latency-p99-time-series).

Do not copy this value into the handover workflow's `AllowedLaggingTasks` input. WaitReplication measures a smaller distance per shard (newest replication task minus the last one the standby acknowledged, zero when fully caught up).

Before v1.8.0 this panel read the standby cluster, which holds no tasks for the forward stream.

See [playbook section 1.3](../../../playbooks/namespace-failover-graceful-handover.md#13-is-replication-lag-low-enough-to-proceed) for interpretation and go/no-go guidance.

### Panel: Replication Tasks Sent per Second (Active) (stat)

**Metric:** `sum(rate(replication_tasks_send{cluster="$active_cluster", operation!="SyncWatermark"}[$__rate_interval]))`  
**Cluster:** active  
**Thresholds:** none  
Replication tasks per second the active is sending to the standby, all task types. `SyncWatermark` is left out: it is a stream status message, sent about once a second per stream, not a replication task. Zero is normal when no replicated namespace has traffic.

Read it next to the lag panels. Lag rising while this panel stays flat has two causes. If [Receive Buffer Full (Standby)](#panel-receive-buffer-full-standby-stat) is non-zero, the standby is the limit: check [Receiver Backlog Depth](#panel-receiver-backlog-depth-stat), standby history pod CPU and database latency. If Receive Buffer Full is zero, the active is not sending any faster. With `history.EnableReplicationTaskTieredProcessing` set to `true`, sending is capped at `history.ReplicationStreamSenderHighPriorityQPS` (default 100 tasks/s) per stream. With it `false` (the default) there is no send cap.

Added in v1.8.0, replacing **Send Backlog (Active)**. That panel read `replication_task_send_backlog`, which only the older pull-based replication path emits, so it was always empty on streaming replication (the default).

> Note: `cluster` is a scrape-level label, not server-emitted.

See [playbook section 1.2](../../../playbooks/namespace-failover-graceful-handover.md#12-is-the-replication-stream-between-clusters-healthy) for interpretation and go/no-go guidance.

### Panel: Receive Buffer Full (Standby) (stat)

**Metric:** `sum(rate(replication_stream_channel_full{cluster="$standby_cluster"}[$__rate_interval]))`  
**Cluster:** standby  
**Thresholds:** green = 0, red ≥ 1  
Times per second the standby's buffer of received replication messages (512 per stream) was full when a new message arrived. The standby is not taking messages off the stream as fast as the active sends them, so applying on the standby is the bottleneck. Short bursts during catch-up after an outage are expected. Sustained under steady load, check standby history pod CPU, database latency and [Receiver Backlog Depth](#panel-receiver-backlog-depth-stat).

The metric is emitted only by the receiving side of a stream, never by the sender. Before v1.8.0 this panel was titled **Send Channel Full (Active)** and read the active cluster, where it only reflects the reverse direction.

> Note: `cluster` is a scrape-level label, not server-emitted.

See [playbook section 1.2](../../../playbooks/namespace-failover-graceful-handover.md#12-is-the-replication-stream-between-clusters-healthy) for interpretation and go/no-go guidance.

### Panel: Replication Lag Trend (time series)

**Metrics:** p50 and p99 of `increase(replication_tasks_lag_bucket{cluster="$active_cluster"}[10m])`  
**Cluster:** active  
Trajectory of the same task-ID distance as [Replication Lag](#panel-replication-lag-gauge), p50 and p99 over 10-minute windows. Use the direction of the lines, not their height. A gap means the standby has not acknowledged for about 10 minutes (see the gauge). Before v1.8.0 this panel read the standby cluster.

See [playbook section 1.3](../../../playbooks/namespace-failover-graceful-handover.md#13-is-replication-lag-low-enough-to-proceed) for interpretation and go/no-go guidance.

### Panel: Replication Latency p99 (time series)

**Metrics:** p50 and p99 of `replication_latency_bucket{source_cluster="$active_cluster"}`  
**Cluster:** standby  
**Thresholds:** green < 10s, orange ≥ 10s, red ≥ 20s  
End-to-end replication latency — wall-clock time from task creation on the active to task application on the standby (server source: `service/history/replication/executable_task.go`). The p99 is the time-based "how many seconds is the standby behind" signal, complementing the task-ID distance `replication_tasks_lag` panels above. Use it to distinguish benign replication-stream reconnect churn from churn that is actually keeping the standby behind. The handover's drain step waits at most `HandoverTimeoutSeconds`, capped at 30. A p99 close to 30 seconds means a task sent just before HANDOVER can take as long as the whole drain step, and the handover rolls back if the drain does not finish in time.

> Fixed in v1.7.0: from v1.5.0 to v1.6.0 this panel was nested inside the Row 1 header while the row was expanded, so Grafana never displayed it.

**Alert:** [FAILOVER-PRE-07](../../../observability/alerts/server/alerts-index.md#alert-failover-pre-07--replication-latency-too-high-for-safe-handover) — fires at p99 ≥ 20s, severity critical.

See [playbook section 1.3](../../../playbooks/namespace-failover-graceful-handover.md#13-is-replication-lag-low-enough-to-proceed) for interpretation and go/no-go guidance.

---

## Row 2a — WaitReplication (Steps 1–3) — No Client Impact

### Panel: Catchup Progress % (gauge)

**Metric:** `sum(catchup_ready_shard_count{target_cluster="$standby_cluster"}) / $total_shards * 100`  
**Thresholds:** red < 50%, orange 50–90%, green ≥ 90%  
Percentage of shards that WaitReplication counts as caught up, by the rule in [playbook section 1.3](../../../playbooks/namespace-failover-graceful-handover.md#13-is-replication-lag-low-enough-to-proceed). Rises from 0 to 100% during WaitReplication (Steps 1–3). Each shard's replication stream carries every replicated namespace, so the value covers all of them. Empty when no handover workflow is running.

> Note: `catchup_ready_shard_count` emits a `target_cluster` label (not `cluster`). It has a `namespace` label only from v1.32.0, naming the handover workflow that recorded the value; the panel does not filter on it.

See [playbook section 2a](../../../playbooks/namespace-failover-graceful-handover.md#2a-handover-workflow-started--monitor-standby-catchup-steps-13) for interpretation and go/no-go guidance.

### Panel: Handover Progress — Catchup + Drain Combined (time series)

**Metrics:**
- `sum(catchup_ready_shard_count{target_cluster="$standby_cluster"})` — rises during Steps 1–3
- `sum(handover_ready_shard_count{target_cluster="$standby_cluster", namespace="$namespace"})` — rises during Steps 4–5
- `$total_shards` reference line (dashed gray)

Both metrics emit a `target_cluster` label (not `cluster`). `handover_ready_shard_count` is filtered by `namespace="$namespace"`; `catchup_ready_shard_count` is not, because it has a `namespace` label only from v1.32.0. Combined view of both catchup and drain progress across the entire handover.

See [playbook section 2a](../../../playbooks/namespace-failover-graceful-handover.md#2a-handover-workflow-started--monitor-standby-catchup-steps-13) for interpretation and go/no-go guidance.

---

## Row 2b — HANDOVER Drain (Steps 4–5)

### Panel: HANDOVER Drain Progress % (gauge)

**Metric:** `sum(handover_ready_shard_count{target_cluster="$standby_cluster", namespace="$namespace"}) / $total_shards * 100`  
**Thresholds:** red < 50%, orange 50–90%, green ≥ 90%  
Fraction of shards that have confirmed the HANDOVER replication task has been applied on the standby (up to the `HandoverReplicationTaskId` snapshot). Capped by `HandoverTimeoutSeconds` (server max: 30s, hardcoded in `handover_workflow.go`).

> Note: `handover_ready_shard_count` emits a `target_cluster` label (not `cluster`). Use `target_cluster="$standby_cluster"` in PromQL expressions.

**Alert:** [FAILOVER-HANDOVER-01](../../../observability/alerts/server/alerts-index.md#alert-failover-handover-01--handover-drain-stalled) — fires after 20s stalled, severity critical.

See [playbook section 2b](../../../playbooks/namespace-failover-graceful-handover.md#2b-namespace-is-in-handover--monitor-the-drain-window-steps-45) for interpretation and go/no-go guidance.

### Panel: UNAVAILABLE Error Burst (Expected) (time series)

**Metric:** `rate(client_redirection_errors{cluster="$active_cluster", namespace="$namespace", error_type="Unavailable"})`  
**Cluster:** active (where `namespace-handover-v2` was started)  
Rate of `Unavailable` errors returned by the active cluster's frontend for write APIs during HANDOVER state (Steps 4–5). Fired by `NamespaceValidatorInterceptor.checkReplicationState` (default config) or `NamespaceHandoverInterceptor` (when `system.enableNamespaceHandoverWait=true`). Read-only and namespace admin APIs are unaffected. `all-apis-forwarding` policy has no effect on this interceptor path.

**Alert:** [FAILOVER-HANDOVER-01](../../../observability/alerts/server/alerts-index.md#alert-failover-handover-01--handover-drain-stalled) — if this burst does not clear within the drain window, drain has stalled.

See [playbook section 2b](../../../playbooks/namespace-failover-graceful-handover.md#2b-namespace-is-in-handover--monitor-the-drain-window-steps-45) for interpretation and go/no-go guidance.

### Panel: Namespace Handover Task Errors (time series)

**Metric:** `rate(task_errors_namespace_handover{cluster="$active_cluster", namespace="$namespace"})`  
**Cluster:** active  
Rate of history task execution attempts blocked because the namespace is in HANDOVER state (Steps 4–5). Each blocked attempt on every retry cycle increments this counter for the entire duration of the drain window.

See [playbook section 2b](../../../playbooks/namespace-failover-graceful-handover.md#2b-namespace-is-in-handover--monitor-the-drain-window-steps-45) for interpretation and go/no-go guidance.

---

## Row 3 — Flip Confirmed (Step 6)

### Panel: UNAVAILABLE Errors Gone (stat)

**Metric:** `rate(client_redirection_errors{cluster="$active_cluster", namespace="$namespace", error_type="Unavailable"})`  
**Cluster:** `$active_cluster` — now standby after flip  
**Thresholds:** green = 0, red > 0  
Rate of `Unavailable` errors on the old active cluster. Should drop to zero after the flip as HANDOVER state exits and the namespace cache refreshes.

See [playbook section 3](../../../playbooks/namespace-failover-graceful-handover.md#3-flip-completed--confirm-the-namespace-switched-correctly-step-6) for interpretation and go/no-go guidance.

### Panel: Forwarding Active (stat)

**Metric:** `rate(client_redirection_requests{cluster="$active_cluster", namespace="$namespace"})`  
**Cluster:** `$active_cluster` — now standby after flip  
**Thresholds:** green > 0, blue = 0  
Rate of requests on the old active cluster being forwarded to the new active via `dcRedirectionPolicy`. Zero is normal when no client or worker calls the old active, for example when all of them already point at the new active, so 0 shows blue, not red. Shows 0 rather than "No data" when nothing was forwarded.

See [playbook section 3](../../../playbooks/namespace-failover-graceful-handover.md#3-flip-completed--confirm-the-namespace-switched-correctly-step-6) for interpretation and go/no-go guidance.

### Panel: Version Fence Fired (stat)

**Metric:** `rate(task_errors_version_mismatch{cluster="$active_cluster"})`  
**Cluster:** old active (now standby) — do not change `$active_cluster`  
**Thresholds:** green > 0, blue = 0  
Rate at which the old active's task executor drops tasks whose failover version stamp no longer matches the namespace's current failover version. Fires as the old active flushes tasks created before the flip. Zero is also a normal outcome — the old active had no pending tasks left when the flip happened — so 0 shows blue, not red, and shows 0 rather than "No data".

See [playbook section 3](../../../playbooks/namespace-failover-graceful-handover.md#3-flip-completed--confirm-the-namespace-switched-correctly-step-6) for interpretation and go/no-go guidance.

---

## Row 4 — Post-Handover Health

### Panel: Version Mismatch Decay (time series)

**Metric:** `rate(task_errors_version_mismatch{cluster="$active_cluster"})`  
**Cluster:** old active (now standby) — do not change `$active_cluster`  
Rate of version-fenced task drops on the old active as its queues flush tasks created before the flip. Expected pattern: spike shortly after flip, then decay to 0.

**Alert:** [FAILOVER-POST-03](../../../observability/alerts/server/alerts-index.md#alert-failover-post-03--version-mismatch-not-decaying) — fires if rate does not decay within 5 minutes, severity warning. 📋 Planned — not yet provisioned in `temporal-failover-alerts.yaml`.

See [playbook section 4](../../../playbooks/namespace-failover-graceful-handover.md#4-monitor-the-new-active-cluster-after-the-handover) for interpretation and go/no-go guidance.

### Panel: Forwarding Error Rate by Type (time series)

**Metric:** `sum by (error_type) (rate(client_redirection_errors{cluster="$active_cluster", namespace="$namespace"}))`  
**Cluster:** old active (now standby)  
Forwarding errors broken down by `error_type` label. Key types: `Unavailable` (new active unreachable), `ResourceExhausted` (new active throttling), `FailedPrecondition` (Update APIs not in forwarding whitelist — see `system.forceNamespaceSelectedAPIAutoForwarding`).

**Alert:** [FAILOVER-POST-01](../../../observability/alerts/server/alerts-index.md#alert-failover-post-01--forwarding-failedprecondition-errors) — fires on sustained `FailedPrecondition` errors after 2 minutes, severity warning.

See [playbook section 4](../../../playbooks/namespace-failover-graceful-handover.md#4-monitor-the-new-active-cluster-after-the-handover) for interpretation and go/no-go guidance.

### Panel: WFT Schedule-to-Start (New Active) (time series)

**Metric:** `histogram_quantile` of `task_schedule_to_start_latency_bucket{cluster="$standby_cluster"}`  
**Cluster:** `$standby_cluster` — the new active after the flip  
**Thresholds:** orange > 1s, red > 5s  
Time from when the new active schedules a task to when a worker picks it up. Covers both workflow task and activity task starts. A brief spike after the flip is expected as queue processors initialise; the duration varies by workload.

> Note: this metric only fires when a task has been scheduled. Genuinely stuck executions that were never re-scheduled on the new active will not appear here.

**Alert:** [FAILOVER-POST-02](../../../observability/alerts/server/alerts-index.md#alert-failover-post-02--wft-schedule-to-start-elevated-post-flip) — fires if p99 > 5s more than 3 minutes after the flip, severity warning. 📋 Planned — not yet provisioned in `temporal-failover-alerts.yaml`.

See [playbook section 4](../../../playbooks/namespace-failover-graceful-handover.md#4-monitor-the-new-active-cluster-after-the-handover) for interpretation and go/no-go guidance.

### Panel: Replication Lag — Reverse Stream (time series)

**Metric:** p50 and p99 of `increase(replication_tasks_lag_bucket{cluster="$standby_cluster"}[10m])` (the new active)  
**Cluster:** new active (the cluster selected as `$standby_cluster`)  
The reverse stream after the flip: the same task-ID distance as [Replication Lag](#panel-replication-lag-gauge), read on the new active, which now holds the replication tasks for the old active. Direction only, not a count of tasks. Read it only after the flip; before it, this shows the standby's own value, which says nothing about the forward stream. Goes empty after about 10 minutes without acknowledgements from the old active. Before v1.8.0 this panel read the old active, which holds no tasks for the reverse stream.

> Note: the old active (now standby) will show a scheduled queue lag of approximately `history.standbyClusterDelay` (default 5 minutes) — the intentional clock offset on the standby timer executor.

See [playbook section 4](../../../playbooks/namespace-failover-graceful-handover.md#4-monitor-the-new-active-cluster-after-the-handover) for interpretation and go/no-go guidance.

### Panel: Forwarding Success Rate (time series)

**Metric:** `rate(client_redirection_requests{cluster="$active_cluster", namespace="$namespace"})`  
**Cluster:** old active (now standby)  
Rate of successful forwarded requests from the old active to the new active. Declines as workers are re-pointed to the new active.

See [playbook section 4](../../../playbooks/namespace-failover-graceful-handover.md#4-monitor-the-new-active-cluster-after-the-handover) for interpretation and go/no-go guidance.

### Panel: Reverse Replication Active (time series)

**Metric:** `rate(replication_latency_count{cluster="$active_cluster", source_cluster="$standby_cluster"})`  
**Cluster:** `$active_cluster` — the old active, now the standby — receiving from `$standby_cluster`, the new active  
Rate of replication tasks from the new active that finished on the old active, confirming the reverse replication stream is established and flowing. Should go non-zero shortly after the flip while the namespace has traffic. It counts every task that finished, applied or failed, and every namespace on the cluster.

> Fixed in v1.7.0: before, this panel read `replication_tasks_applied` on the new active. Only the old pull-based replication processor records that metric, so the panel was always empty on streaming replication.

> Note: do not swap dashboard variables after the flip — leave `$standby_cluster` pointing at the new active throughout.

See [playbook section 4](../../../playbooks/namespace-failover-graceful-handover.md#4-monitor-the-new-active-cluster-after-the-handover) for interpretation and go/no-go guidance.

### Panel: Replication DLQ Writes (Previously Active Cluster) (time series)

**Metric:** `sum(dlq_writes{cluster="$active_cluster", task_category="replication"}) or vector(0)`  
**Cluster:** `$active_cluster` — the cluster the namespace was handed over from, now the standby  
Replication updates that cluster could not apply and wrote to its replication DLQ, counted since each history pod started. A step up is a write.

After handing over a namespace that was a local namespace before it was made global, the cluster handed over from can reject the first update for some workflows that were started before the namespace was made global. Most of those updates are applied at the next reconnect of the replication connection; any that run out of retries first are written to the DLQ and show here. A write means that cluster's copy of a workflow is out of date. Merge its replication DLQ before handing the namespace back, or the out-of-date copy is the one that runs after the hand-back.

- **Flat at 0** is the healthy reading, and what the panel shows when nothing was ever written.
- **The count does not go down when the DLQ is merged**, and resets when a history pod restarts.
- **It counts every namespace on the cluster** and ignores `$namespace`, for the same reason as **Replication DLQ Writes**.

**Version floor:** same as **Replication DLQ Writes** — by default from server v1.28.0.

See [playbook section 4](../../../playbooks/namespace-failover-graceful-handover.md#4-monitor-the-new-active-cluster-after-the-handover) for interpretation and what to do.
