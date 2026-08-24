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

> **Current version:** v1.5.0 — see [CHANGELOG](./namespace-failover-graceful-handover-changelog.md)

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
  - [Replication Lag](#panel-replication-lag-gauge)
  - [Send Backlog (Active)](#panel-send-backlog-active-stat)
  - [Send Channel Full (Active)](#panel-send-channel-full-active-stat)
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
  - [HANDOVER State Exited](#panel-handover-state-exited-stat)
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

---

## Template Variables

| Variable | Source | Notes |
|---|---|---|
| `$active_cluster` | `label_values(replication_tasks_lag, cluster)` | The cluster that is currently active for the namespace — the source of replication and the old active post-flip |
| `$standby_cluster` | `label_values(replication_tasks_lag, cluster)` | The target cluster — receives replication, becomes the new active after flip |
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
**Thresholds:** green = 0, red > 0  
gRPC protocol-layer errors on the replication stream on the standby cluster, distinct from application-level errors (`replication_service_error`). `StreamError` is non-retryable — a single occurrence causes the stream to die and reconnect immediately via the outer `WrapEventLoop` loop.

> **Note on the `cluster` label:** `cluster` here is a scrape-level label added by Prometheus configuration, not a server-emitted dimension. The actual server-emitted labels for filtering by stream direction are `from_cluster_id` and `to_cluster_id` (integer cluster IDs).

**Alert:** [FAILOVER-PRE-03](../../../observability/alerts/server/alerts-index.md#alert-failover-pre-03--replication-stream-errors-sustained) — fires after 2 minutes sustained, severity critical.

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

### Panel: Replication Lag (gauge)

**Metric:** `histogram_quantile($p, sum(rate(replication_tasks_lag_bucket{cluster="$standby_cluster"})) by (le))`  
**Cluster:** standby  
**Thresholds:** orange ≥ 1 000 tasks, red ≥ 5 000 tasks  
p`$p` of outstanding replication task lag on the standby — the number of tasks generated on the active that have not yet been applied on the standby. This value is compared against the `AllowedLaggingTasks` workflow input at WaitReplication exit time.

See [playbook section 1.3](../../../playbooks/namespace-failover-graceful-handover.md#13-is-replication-lag-low-enough-to-proceed) for interpretation and go/no-go guidance.

### Panel: Send Backlog (Active) (stat)

**Metric:** `histogram_quantile(0.99, sum(rate(replication_task_send_backlog_bucket{cluster="$active_cluster"}[$__rate_interval])) by (le))`  
**Cluster:** active  
**Thresholds:** green = 0–9, orange ≥ 10, red ≥ 100  
p99 of tasks queued inside the active cluster's replication sender waiting to be dispatched over the stream. Reflects sender-side capacity; relevant when lag is flat at a non-zero value.

> Note: `cluster` is a scrape-level label, not server-emitted.

See [playbook section 1.3](../../../playbooks/namespace-failover-graceful-handover.md#13-is-replication-lag-low-enough-to-proceed) for interpretation and go/no-go guidance.

### Panel: Send Channel Full (Active) (stat)

**Metric:** `sum(rate(replication_stream_channel_full{cluster="$active_cluster"}[$__rate_interval]))`  
**Cluster:** active  
**Thresholds:** green = 0, red ≥ 1  
Rate of events where the sender's internal Go dispatch channel is full — the earliest sender-side capacity signal, preceding a rising send backlog.

> Note: `cluster` is a scrape-level label, not server-emitted.

See [playbook section 1.3](../../../playbooks/namespace-failover-graceful-handover.md#13-is-replication-lag-low-enough-to-proceed) for interpretation and go/no-go guidance.

### Panel: Replication Lag Trend (time series)

**Metrics:** p50 and p99 of `replication_tasks_lag_bucket{cluster="$standby_cluster"}`  
**Cluster:** standby  
15-minute trajectory of replication lag on the standby cluster. Two series: p50 and p99.

See [playbook section 1.3](../../../playbooks/namespace-failover-graceful-handover.md#13-is-replication-lag-low-enough-to-proceed) for interpretation and go/no-go guidance.

### Panel: Replication Latency p99 (time series)

**Metrics:** p50 and p99 of `replication_latency_bucket{source_cluster="$active_cluster"}`  
**Cluster:** standby  
**Thresholds:** green < 10s, orange ≥ 10s, red ≥ 20s  
End-to-end replication latency — wall-clock time from task creation on the active to task application on the standby (server source: `service/history/replication/executable_task.go`). The p99 is the time-based "how many seconds is the standby behind" signal, complementing the task-count `replication_tasks_lag` panels above. Use it to distinguish benign replication-stream reconnect churn from churn that is actually keeping the standby behind. If p99 is approaching 30 seconds, any task in flight when HANDOVER starts cannot drain within the 30-second window and the handover is guaranteed to roll back.

**Alert:** [FAILOVER-PRE-07](../../../observability/alerts/server/alerts-index.md#alert-failover-pre-07--replication-latency-too-high-for-safe-handover) — fires at p99 ≥ 20s, severity critical.

See [playbook section 1.3](../../../playbooks/namespace-failover-graceful-handover.md#13-is-replication-lag-low-enough-to-proceed) for interpretation and go/no-go guidance.

---

## Row 2a — WaitReplication (Steps 1–3) — No Client Impact

### Panel: Catchup Progress % (gauge)

**Metric:** `sum(catchup_ready_shard_count{target_cluster="$standby_cluster", namespace="$namespace"}) / $total_shards * 100`  
**Thresholds:** red < 50%, orange 50–90%, green ≥ 90%  
Fraction of shards that have reported lag within `AllowedLaggingSeconds`. Rises from 0 to 100% during WaitReplication (Steps 1–3).

> Note: `catchup_ready_shard_count` emits a `target_cluster` label (not `cluster`) and supports `namespace="$namespace"` filtering.

See [playbook section 2a](../../../playbooks/namespace-failover-graceful-handover.md#2a-handover-workflow-started--monitor-standby-catchup-steps-13) for interpretation and go/no-go guidance.

### Panel: Handover Progress — Catchup + Drain Combined (time series)

**Metrics:**
- `sum(catchup_ready_shard_count{target_cluster="$standby_cluster", namespace="$namespace"})` — rises during Steps 1–3
- `sum(handover_ready_shard_count{target_cluster="$standby_cluster", namespace="$namespace"})` — rises during Steps 4–5
- `$total_shards` reference line (dashed gray)

Both metrics emit a `target_cluster` label (not `cluster`). `catchup_ready_shard_count` also supports `namespace="$namespace"` filtering. Combined view of both catchup and drain progress across the entire handover.

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

### Panel: HANDOVER State Exited (stat)

**Metric:** `sum(handover_ready_shard_count{target_cluster="$standby_cluster", namespace="$namespace"})`  
**Cluster:** standby (target)  
Count of history shards that confirmed the HANDOVER watermark during the drain window. After WaitHandover stops, this gauge goes stale/NaN in Prometheus — it does not return to 0 after the flip completes.

> Note: `handover_ready_shard_count` emits a `target_cluster` label (not `cluster`). Use `target_cluster="$standby_cluster"` in PromQL expressions.

See [playbook section 3](../../../playbooks/namespace-failover-graceful-handover.md#3-flip-completed--confirm-the-namespace-switched-correctly-step-6) for interpretation and go/no-go guidance.

### Panel: UNAVAILABLE Errors Gone (stat)

**Metric:** `rate(client_redirection_errors{cluster="$active_cluster", namespace="$namespace", error_type="Unavailable"})`  
**Cluster:** `$active_cluster` — now standby after flip  
**Thresholds:** green = 0, red > 0  
Rate of `Unavailable` errors on the old active cluster. Should drop to zero after the flip as HANDOVER state exits and the namespace cache refreshes.

See [playbook section 3](../../../playbooks/namespace-failover-graceful-handover.md#3-flip-completed--confirm-the-namespace-switched-correctly-step-6) for interpretation and go/no-go guidance.

### Panel: Forwarding Active (stat)

**Metric:** `rate(client_redirection_requests{cluster="$active_cluster", namespace="$namespace"})`  
**Cluster:** `$active_cluster` — now standby after flip  
**Thresholds:** green > 0, red = 0  
Rate of requests on the old active cluster being forwarded to the new active via `dcRedirectionPolicy`.

See [playbook section 3](../../../playbooks/namespace-failover-graceful-handover.md#3-flip-completed--confirm-the-namespace-switched-correctly-step-6) for interpretation and go/no-go guidance.

### Panel: Version Fence Fired (stat)

**Metric:** `rate(task_errors_version_mismatch{cluster="$active_cluster"})`  
**Cluster:** old active (now standby) — do not change `$active_cluster`  
**Thresholds:** green > 0, red = 0  
Rate at which the old active's task executor drops tasks whose failover version stamp no longer matches the namespace's current failover version. Fires as the old active flushes tasks created before the flip.

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

**Metric:** `histogram_quantile` of `replication_tasks_lag_bucket{cluster="$active_cluster"}` (old active, now standby)  
**Cluster:** old active (now standby)  
Replication lag on the reverse stream — tasks generated on the new active that have not yet been applied on the old active (now standby). Should be near 0 immediately after flip given WaitReplication's pre-flight catchup.

> Note: the old active (now standby) will show a scheduled queue lag of approximately `history.standbyClusterDelay` (default 5 minutes) — the intentional clock offset on the standby timer executor.

See [playbook section 4](../../../playbooks/namespace-failover-graceful-handover.md#4-monitor-the-new-active-cluster-after-the-handover) for interpretation and go/no-go guidance.

### Panel: Forwarding Success Rate (time series)

**Metric:** `rate(client_redirection_requests{cluster="$active_cluster", namespace="$namespace"})`  
**Cluster:** old active (now standby)  
Rate of successful forwarded requests from the old active to the new active. Declines as workers are re-pointed to the new active.

See [playbook section 4](../../../playbooks/namespace-failover-graceful-handover.md#4-monitor-the-new-active-cluster-after-the-handover) for interpretation and go/no-go guidance.

### Panel: Reverse Replication Active (time series)

**Metric:** `rate(replication_tasks_applied{cluster="$standby_cluster"})`  
**Cluster:** `$standby_cluster` — the new active after the flip  
Rate of replication tasks applied on the old active (now standby), confirming the reverse replication stream from the new active is established and flowing. Should go non-zero shortly after the flip as the new active's namespace cache refreshes (default 2s).

> Note: do not swap dashboard variables after the flip — leave `$standby_cluster` pointing at the new active throughout.

See [playbook section 4](../../../playbooks/namespace-failover-graceful-handover.md#4-monitor-the-new-active-cluster-after-the-handover) for interpretation and go/no-go guidance.
