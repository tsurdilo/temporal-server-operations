# Temporal Server — Full Alert Index

Complete reference for all 91 server alert definitions across 19 sections. Includes both the Essential Set (implemented in `temporal-server-alerts.yaml`) and the full planned inventory.

> **Essential Set:** A curated subset of 17 alerts has been selected for deployment. See [README.md](./README.md) for setup instructions and runbook links.
> **Planning document:** See [planning.md](./planning.md) for design decisions and the full working notes.

> **Tuning note:** All thresholds and `for` durations documented here are baselines — calibrated starting points that should work well for most production deployments. Different workloads, cluster sizes, and SLO requirements will need different values. The `for` duration controls how long a condition must hold continuously before the alert fires: shorter values catch problems faster at the cost of more noise from transient spikes; longer values reduce false positives but delay detection. Treat every value here as a starting point and adjust to your environment.

---

## Status Key

- ✅ **Implemented** — alert is in `temporal-server-alerts.yaml` and deployed
- 📋 **Planned** — defined in planning.md, not yet implemented

---

## Table of Contents

- [Section 0 — History Host Health](#section-0--history-host-health) (#0a–#0c)
- [Section 1 — Cluster Throughput](#section-1--cluster-throughput) (#1–#6)
- [Section 2 — Shard and Workflow Lock Latencies](#section-2--shard-and-workflow-lock-latencies) (#7–#10)
- [Section 3 — Persistence](#section-3--persistence) (#11–#18)
- [Section 4 — Service Latencies](#section-4--service-latencies) (#19–#24)
- [Section 5 — Service Requests and Errors](#section-5--service-requests-and-errors) (#25–#28)
- [Section 6 — Throttling and Limits](#section-6--throttling-and-limits) (#29–#30)
- [Section 7 — Busy Workflow Throttling](#section-7--busy-workflow-throttling) (#31–#33)
- [Section 8 — Shard Movement](#section-8--shard-movement) (#34)
- [Section 9 — Shard Queue Health](#section-9--shard-queue-health) (#34a–#34i)
- [Section 10 — History Timer Task Info](#section-10--history-timer-task-info) (#35–#39)
- [Section 11 — Workflow Stats](#section-11--workflow-stats) (#40–#41)
- [Section 12 — Workflow Execution History Info](#section-12--workflow-execution-history-info) (#42–#47)
- [Section 13 — Matching Task Queue Info](#section-13--matching-task-queue-info) (#74)
- [Section 14 — SDK Workers Info](#section-14--sdk-workers-info) (#48–#56)
- [Section 15 — Pollers](#section-15--pollers) (#57–#58)
- [Section 16 — Visibility](#section-16--visibility) (#59–#63, #59a–#59c)
- [Section 18 — Cluster Replication](#section-18--cluster-replication) (#64–#71)
- [Section 19 — Authorization](#section-19--authorization) (#72–#73)

---

## Section 0 — History Host Health

> **Dashboard:** [Temporal History Host Health Dashboard](../../dashboards/server/history-health-dashboard.json) (`history-health-dashboard.json`) — **not** the main Temporal Server Dashboard. This is the only section that references the companion health dashboard.
> **Metric:** `host_health` (`HistoryHostHealthGauge`) — emitted only when an external poller calls `AdminHandler.DeepHealthCheck` on each history pod.

### Alert 0a — History Pod Disappeared

| Field | Value |
|---|---|
| Status | 📋 Planned |
| Severity | critical |
| Component | history |

**Condition:** `clamp_min(max_over_time(count(host_health{service_name="history"})[1h:1m]) - (count(host_health{service_name="history"}) or vector(0)), 0) >= 1`

Fires when any pod stops emitting `host_health` compared to the fleet baseline seen in the last hour. A crashed or killed pod goes absent rather than reporting NOT_SERVING — this alert is required alongside 0b to catch pod loss. Alert 0b cannot detect this case.

---

### Alert 0b — History Pod Degraded

| Field | Value |
|---|---|
| Status | 📋 Planned |
| Severity | critical |
| Component | history |

**Condition:** `sum(clamp_max(host_health{service_name="history"} == 2, 1)) >= 1`

Fires when any pod is actively reporting `NOT_SERVING` (`host_health == 2`). Covers persistence/RPC threshold breaches and gRPC health failures past the 60s init window.

---

### Alert 0c — Metric Freshness Stale

| Field | Value |
|---|---|
| Status | 📋 Planned |
| Severity | critical |
| Component | history |

**Condition:** `time() - max(timestamp(host_health{service_name="history"})) > 120`

Fires when `host_health` hasn't been updated in 120 seconds. Means the external poller cannot reach the frontend, or the frontend is down. Absence of data from a running poller is itself a health signal.

---

## Section 1 — Cluster Throughput

> **Dashboard panels:** Total RPS (panel 21), RPS per Namespace (panel 20)
> **Metric:** `service_requests`
> **Component:** frontend

### Alert 1 — Total RPS Drops to Zero

| Field | Value |
|---|---|
| Status | ✅ Implemented |
| UID | `temporal-alert-001` |
| Severity | critical |
| Panel | 21 |
| `for` | 5m |
| `noDataState` | NoData |

**Condition:** `sum(rate(service_requests{service_name="frontend"}[5m])) < 1`

Total frontend RPS has dropped to zero. The frontend is up and reporting metrics but receiving no requests — the cluster is unreachable from clients and from other Temporal services.

**Runbook:** [01-total-rps-zero.md](./runbooks/01-total-rps-zero.md)

---

### Alert 1b — Frontend Metrics Absent

| Field | Value |
|---|---|
| Status | ✅ Implemented |
| UID | `temporal-alert-001b` |
| Severity | critical |
| Panel | 21 |
| `for` | 2m |
| `noDataState` | OK |

**Condition:** `absent(rate(service_requests{service_name="frontend"}[5m])) > 0`

Companion to alert 1. Fires when the `service_requests` series for the frontend disappears entirely (pod crash, scrape failure). Alert 1 cannot fire in this case because there is no value to evaluate. `noDataState: OK` — when the series is present, `absent()` returns no data, which is the healthy state.

**Runbook:** [01b-frontend-metrics-absent.md](./runbooks/01b-frontend-metrics-absent.md)

---

### Alert 2 — Total RPS Approaching Limit

| Field | Value |
|---|---|
| Status | 📋 Planned |
| Severity | warning |
| Panel | Total RPS |

**Condition:** Total frontend RPS exceeds warning threshold (default ~2,000 — assumes ~3 frontend instances; adjust to `frontend.rps` × instance count).

---

### Alert 3 — Total RPS Over Limit

| Field | Value |
|---|---|
| Status | 📋 Planned |
| Severity | critical |
| Panel | Total RPS |

**Condition:** Total frontend RPS exceeds red threshold (default ~7,000) — active throttling is happening, users are receiving `RpsLimit` errors.

---

### Alert 4 — Namespace RPS Drops to Zero

| Field | Value |
|---|---|
| Status | ✅ Implemented |
| UID | `temporal-alert-004` |
| Severity | critical |
| Panel | 20 |
| `for` | 5m |
| `noDataState` | NoData |

**Condition:** `sum by (namespace) (rate(service_requests{service_name="frontend",namespace!="_unknown_"}[5m])) < 1`

A specific namespace is receiving no frontend requests. Other namespaces may be healthy. Check if alert 1 is also firing — if so this is cluster-wide; otherwise check if workers for this namespace are running and connected.

**Runbook:** [04-namespace-rps-zero.md](./runbooks/04-namespace-rps-zero.md)

---

### Alert 5 — Namespace RPS Approaching Limit

| Field | Value |
|---|---|
| Status | 📋 Planned |
| Severity | warning |
| Panel | RPS per Namespace |

**Condition:** Per-namespace RPS approaches `frontend.namespaceRPS` warning level (default 2,400/host).

---

### Alert 6 — Namespace RPS Over Limit

| Field | Value |
|---|---|
| Status | 📋 Planned |
| Severity | critical |
| Panel | RPS per Namespace |

**Condition:** Per-namespace RPS exceeds `frontend.namespaceRPS` hard limit — users are receiving `RpsLimit` throttle errors for this namespace.

---

## Section 2 — Shard and Workflow Lock Latencies

> **Dashboard panels:** Shard Lock Latency (panel 14), Workflow Lock Latency
> **Metrics:** `semaphore_latency`
> **Component:** history

### Alert 7 — Shard Lock Latency High

| Field | Value |
|---|---|
| Status | 📋 Planned |
| Severity | warning |
| Panel | Shard Lock Latency (14) |

**Condition:** `histogram_quantile(0.99, ...(semaphore_latency_bucket{operation="ShardInfo"}...)) > 0.15`

p99 shard lock latency exceeds 150ms.

---

### Alert 8 — Shard Lock Latency Critical

| Field | Value |
|---|---|
| Status | ✅ Implemented |
| UID | `temporal-alert-008` |
| Severity | critical |
| Panel | 14 |
| `for` | 5m |
| `noDataState` | NoData |

**Condition:** `histogram_quantile(0.99, sum by (le) (rate(semaphore_latency_bucket{operation="ShardInfo",service_name="history"}[5m]))) > 0.3`

**Threshold:** p99 > 300ms

p99 shard lock latency exceeds 300ms. Indicates history service issues, elevated DB latencies holding the lock longer, or hot shard conditions from high-frequency reuse of the same workflow ID.

**Runbook:** [08-shard-lock-latency-critical.md](./runbooks/08-shard-lock-latency-critical.md)

---

### Alert 9 — Workflow Lock Latency High

| Field | Value |
|---|---|
| Status | 📋 Planned |
| Severity | warning |
| Panel | Workflow Lock Latency |

**Condition:** p99 workflow lock latency exceeds 200ms.

---

### Alert 10 — Workflow Lock Latency Critical

| Field | Value |
|---|---|
| Status | 📋 Planned |
| Severity | critical |
| Panel | Workflow Lock Latency |

**Condition:** p99 workflow lock latency exceeds 400ms.

---

## Section 3 — Persistence

> **Dashboard panels:** Persistence Latencies (panel 71), Persistence Requests Total, Persistence Availability, Persistence Errors by Namespace, SQL DB Connection Pool
> **Metric:** `persistence_latency`, `persistence_requests`, `persistence_errors`
> **Component:** persistence

### Alert 11 — Persistence Latency High

| Field | Value |
|---|---|
| Status | 📋 Planned |
| Severity | warning |
| Panel | Persistence Latencies (71) |

**Condition:** p99 persistence latency exceeds 300ms for any covered critical-path operation.

---

### Alert 12 — Persistence Latency Critical

| Field | Value |
|---|---|
| Status | ✅ Implemented |
| UID | `temporal-alert-012` |
| Severity | critical |
| Panel | 71 |
| `for` | 5m |
| `noDataState` | NoData |

**Condition:**
```promql
histogram_quantile(0.99, sum by (operation, le) (rate(persistence_latency_bucket{
  operation=~"CreateWorkflowExecution|UpdateWorkflowExecution|ConflictResolveWorkflowExecution|
              GetWorkflowExecution|GetCurrentExecution|AppendHistoryNodes|ReadHistoryBranch|
              CreateTasks|GetTasks|GetTransferTasks|GetTimerTasks|CompleteTransferTask|CompleteTimerTask"
}[5m]))) > 1
```

**Threshold:** p99 > 1s

p99 persistence latency has exceeded 1s for a critical-path DB operation. Persistence latencies can cause delays writing and reading from the database, causing slowness for critical operations that support forward progress of workflow executions.

**Runbook:** [12-persistence-latency-critical.md](./runbooks/12-persistence-latency-critical.md)

---

### Alert 13 — Persistence QPS Approaching Limit

| Field | Value |
|---|---|
| Status | 📋 Planned |
| Severity | warning |
| Panel | Persistence Requests Total |

**Condition:** Total persistence req/s exceeds 7,000 (based on `history.persistenceMaxQPS` default 9,000/host).

---

### Alert 14 — Persistence Availability Degraded

| Field | Value |
|---|---|
| Status | 📋 Planned |
| Severity | warning |
| Panel | Persistence Availability |

**Condition:** Persistence success rate drops below 99%.

---

### Alert 15 — Persistence Availability Critical

| Field | Value |
|---|---|
| Status | 📋 Planned |
| Severity | critical |
| Panel | Persistence Availability |

**Condition:** Persistence success rate drops below 95%.

---

### Alert 16 — Persistence Errors Elevated

| Field | Value |
|---|---|
| Status | 📋 Planned |
| Severity | warning |
| Panel | Persistence Errors by Namespace |

**Condition:** Sustained persistence error rate above zero for any namespace.

---

### Alert 17 — SQL Connection Pool Near Saturation

| Field | Value |
|---|---|
| Status | 📋 Planned |
| Severity | warning |
| Panel | SQL DB Connection Pool |

**Condition:** Open connections exceeding 90% of configured max. SQL backends only.

---

### Alert 18 — SQL Connection Pool Saturated

| Field | Value |
|---|---|
| Status | 📋 Planned |
| Severity | critical |
| Panel | SQL DB Connection Pool |

**Condition:** Open connections at or exceeding configured max. SQL backends only.

---

## Section 4 — Service Latencies

> **Dashboard panels:** Frontend/History/Matching Service Latency panels
> **Metric:** `service_latency`
> **Component:** frontend, history, matching

### Alert 19 — Frontend Service Latency High

| Field | Value |
|---|---|
| Status | 📋 Planned |
| Severity | warning |
| Panel | Frontend Service Latency |

**Condition:** p99 frontend latency exceeds 300ms (excludes `PollWorkflowTaskQueue`, `PollActivityTaskQueue`).

---

### Alert 20 — Frontend Service Latency Critical

| Field | Value |
|---|---|
| Status | 📋 Planned |
| Severity | critical |
| Panel | Frontend Service Latency |

**Condition:** p99 frontend latency exceeds 2s.

---

### Alert 21 — History Service Latency High

| Field | Value |
|---|---|
| Status | 📋 Planned |
| Severity | warning |
| Panel | History Service Latency |

**Condition:** p99 history latency exceeds 400ms.

---

### Alert 22 — History Service Latency Critical

| Field | Value |
|---|---|
| Status | 📋 Planned |
| Severity | critical |
| Panel | History Service Latency |

**Condition:** p99 history latency exceeds 2s.

---

### Alert 23 — Matching Service Latency High

| Field | Value |
|---|---|
| Status | 📋 Planned |
| Severity | warning |
| Panel | Matching Service Latency |

**Condition:** p99 matching latency exceeds 400ms (excludes `PollWorkflowTaskQueue`, `PollActivityTaskQueue`, `MatchingClientGetTaskQueueUserData`).

---

### Alert 24 — Matching Service Latency Critical

| Field | Value |
|---|---|
| Status | 📋 Planned |
| Severity | critical |
| Panel | Matching Service Latency |

**Condition:** p99 matching latency exceeds 2s.

---

## Section 5 — Service Requests and Errors

> **Dashboard panels:** Service Panics (panel 99), Service Errors by Namespace (panel 92), Service Requests by Namespace
> **Metrics:** `service_panics`, `service_errors`, `service_requests`
> **Component:** server, frontend

### Alert 25 — Service Panic Detected

| Field | Value |
|---|---|
| Status | ✅ Implemented |
| UID | `temporal-alert-025` |
| Severity | critical |
| Panel | 99 |
| `for` | 1m |
| `noDataState` | NoData |

**Condition:** `sum by (service_name) (rate(service_panics[5m])) > 0`

A Go panic has been detected in a Temporal service. Depending on where the panic occurs this can cause shard loss, incomplete workflow state transitions, or task corruption. The panic stack trace in pod logs is the primary diagnostic artifact.

**Runbook:** [25-service-panic-detected.md](./runbooks/25-service-panic-detected.md)

---

### Alert 26 — Service Error Rate Elevated

| Field | Value |
|---|---|
| Status | 📋 Planned |
| Severity | warning |
| Panel | Service Errors by Namespace (92) |

**Condition:** Sustained service error rate for any namespace above zero.

---

### Alert 27 — Service Error Rate Critical

| Field | Value |
|---|---|
| Status | ✅ Implemented |
| UID | `temporal-alert-027` |
| Severity | critical |
| Panel | 92 |
| `for` | 2m |
| `noDataState` | NoData |

**Condition:** `sum by (namespace) (rate(service_errors{service_name="frontend",namespace!="_unknown_"}[5m])) / sum by (namespace) (rate(service_requests{service_name="frontend",namespace!="_unknown_"}[5m])) > 0.3`

**Threshold:** > 30% error ratio

More than 30% of frontend requests are returning errors for a namespace. Check Service Errors by Namespace and Persistence Latencies panels to identify root cause.

**Runbook:** [27-service-error-rate-critical.md](./runbooks/27-service-error-rate-critical.md)

---

### Alert 28 — Namespace RPS Approaching Limit

| Field | Value |
|---|---|
| Status | 📋 Planned |
| Severity | warning |
| Panel | Service Requests by Namespace |

**Condition:** Per-namespace request rate approaching `frontend.namespaceRPS` limit.

---

## Section 6 — Throttling and Limits

> **Dashboard panel:** Resource Exhausted with Cause (panel 121)
> **Metric:** `service_errors_resource_exhausted`
> **Component:** persistence

### Alert 29 — RPS/QPS Throttling Active

| Field | Value |
|---|---|
| Status | 📋 Planned |
| Severity | warning |
| Panel | Resource Exhausted with Cause (121) |

**Condition:** `sum(rate(service_errors_resource_exhausted{resource_exhausted_cause=~"RESOURCE_EXHAUSTED_CAUSE_RPS_LIMIT|RESOURCE_EXHAUSTED_CAUSE_PERSISTENCE_LIMIT"}[5m])) > 0`

Resource exhausted errors with cause `RpsLimit` or `QpsLimit` — configured rate limits are being hit.

---

### Alert 30 — System Overload Throttling

| Field | Value |
|---|---|
| Status | ✅ Implemented |
| UID | `temporal-alert-030` |
| Severity | critical |
| Panel | 121 |
| `for` | 1m |
| `noDataState` | NoData |

**Condition:** `sum(rate(service_errors_resource_exhausted{resource_exhausted_cause=~"RESOURCE_EXHAUSTED_CAUSE_SYSTEM_OVERLOADED|RESOURCE_EXHAUSTED_CAUSE_CIRCUIT_BREAKER_OPEN"}[5m])) > 0`

Resource exhausted errors with cause `SYSTEM_OVERLOADED` or `CIRCUIT_BREAKER_OPEN`. Unlike configured RPS/QPS limits, these are dynamic self-protection responses — the cluster is actively shedding load because it cannot keep up.

**Runbook:** [30-system-overload-throttling.md](./runbooks/30-system-overload-throttling.md)

---

## Section 7 — Busy Workflow Throttling

> **Dashboard panels:** Transfer Active Task Errors Workflow Busy, Transfer Active Task Errors Discarded (panel 141)
> **Metric:** `task_errors`, `task_errors_discarded`
> **Component:** history

### Alert 31 — Busy Workflow Throttling Elevated

| Field | Value |
|---|---|
| Status | 📋 Planned |
| Severity | warning |
| Panel | Transfer Active Task Errors Workflow Busy |

**Condition:** Sustained busy workflow error rate above zero.

---

### Alert 32 — Busy Workflow Throttling Critical

| Field | Value |
|---|---|
| Status | 📋 Planned |
| Severity | critical |
| Panel | Transfer Active Task Errors Workflow Busy |

**Condition:** High sustained busy workflow error rate.

---

### Alert 33 — Transfer Tasks Being Discarded

| Field | Value |
|---|---|
| Status | 📋 Planned (removed from Essential Set) |
| Severity | critical |
| Panel | Transfer Active Task Errors Discarded (141) |

**Condition:** Any tasks being discarded — cluster gave up retrying, tasks are lost.

> Removed from Essential Set: `task_errors_discarded` only emits on standby clusters. Not applicable to single active cluster deployments.

---

## Section 8 — Shard Movement

> **Dashboard panels:** Shards Created/Removed/Closed (panel 60), Service Restarts (panel 63)
> **Metrics:** `sharditem_created_count`, `sharditem_removed_count`, `shard_closed_count`, `restarts`
> **Component:** history

### Alert 34 — Unexpected Shard Movement

| Field | Value |
|---|---|
| Status | ✅ Implemented |
| UID | `temporal-alert-034` |
| Severity | critical |
| Panel | 60 |
| `for` | 10m |
| `noDataState` | NoData |

**Condition:**
```promql
(
  sum(rate(sharditem_created_count{service_name="history"}[5m]))
  + sum(rate(sharditem_removed_count{service_name="history"}[5m]))
  + sum(rate(shard_closed_count{service_name="history"}[5m]))
  > 0
)
unless (sum(increase(restarts{service_name="history"}[8m])) > 0)
```

Shard churn detected without a corresponding history pod restart in the last 8 minutes. Shard movement during planned restarts or scaling is expected and filtered. Unexpected movement indicates DB pressure causing ownership loss, membership instability, or a history pod unable to maintain shard leases.

**Runbook:** [34-unexpected-shard-movement.md](./runbooks/34-unexpected-shard-movement.md)

---

## Section 9 — Shard Queue Health

> **Dashboard panels:** Immediate Queue Lag per Pod (panel 2109), Scheduled Queue Lag per Pod, DB Pool Refresh Failure Rate per Pod, Suspected Deadlocks per Pod (panel 2113), Task Scheduler Latency per Operation, Task Scheduler Throttled Rate per Operation
> **Metrics:** `shardinfo_immediate_queue_lag`, `shardinfo_scheduled_queue_lag`, `dd_current_suspected_deadlocks`, `task_latency_schedule`, `task_scheduler_throttled`
> **Component:** history

### Alert 34a — Immediate Queue Lag High

| Field | Value |
|---|---|
| Status | 📋 Planned |
| Severity | warning |
| Panel | Immediate Queue Lag per Pod (2109) |

**Condition:** p99 immediate queue lag exceeds 500K tasks on any `instance + task_category`. One pod's queue growing while others stay flat is an early stuck shard signal.

---

### Alert 34b — Immediate Queue Lag Critical

| Field | Value |
|---|---|
| Status | ✅ Implemented |
| UID | `temporal-alert-034b` |
| Severity | critical |
| Panel | 2109 |
| `for` | 15m |
| `noDataState` | NoData |

**Condition:** `histogram_quantile(0.99, sum by (instance, task_category, le) (rate(shardinfo_immediate_queue_lag_bucket{service_name="history"}[11m]))) > 3000000`

**Threshold:** p99 lag > 3,000,000 tasks

A shard on a history pod is stuck — tasks are queuing but not being processed. Workers may report workflows created but no tasks dispatched.

**Runbook:** [34b-immediate-queue-lag-critical.md](./runbooks/34b-immediate-queue-lag-critical.md)

---

### Alert 34c — Scheduled Queue Lag High

| Field | Value |
|---|---|
| Status | 📋 Planned |
| Severity | warning |
| Panel | Scheduled Queue Lag per Pod |

**Condition:** p99 scheduled queue lag exceeds 10 minutes on any `instance + task_category`.

---

### Alert 34d — Scheduled Queue Lag Critical

| Field | Value |
|---|---|
| Status | 📋 Planned |
| Severity | critical |
| Panel | Scheduled Queue Lag per Pod |

**Condition:** p99 scheduled queue lag exceeds 30 minutes on any `instance + task_category`.

---

### Alert 34e — DB Pool Refresh Failure Elevated

| Field | Value |
|---|---|
| Status | 📋 Planned |
| Severity | critical |
| Panel | DB Pool Refresh Failure Rate per Pod |

**Condition:** Any DB pool refresh failures detected. Earliest signal for DB-caused stuck shards — fires before queue lag builds. SQL backends only.

---

### Alert 34f — Shard Deadlock Detected

| Field | Value |
|---|---|
| Status | ✅ Implemented |
| UID | `temporal-alert-034f` |
| Severity | critical |
| Panel | 2113 |
| `for` | 1m |
| `noDataState` | OK |

**Condition:** `sum by (instance) (dd_current_suspected_deadlocks{service_name="history"}) > 0`

The deadlock detector has reported at least one unresolved suspected deadlock on a history pod. A shard's `rwLock` or `ioSemaphore` ping timed out — the lock is currently held and blocking all operations on the affected shard(s). `noDataState: OK` — the metric is event-driven and does not emit baseline zero.

**Runbook:** [34f-shard-deadlock-detected.md](./runbooks/34f-shard-deadlock-detected.md)

---

### Alert 34g — Task Scheduler Latency High

| Field | Value |
|---|---|
| Status | 📋 Planned |
| Severity | warning |
| Panel | Task Scheduler Latency per Operation |

**Condition:** p99 `task_latency_schedule` > 500ms sustained 5m. The host-level executor pool is starting to saturate. Most commonly caused by a namespace doing bulk operations (mass terminations, deletions) that consume all `history.transferProcessorSchedulerWorkerCount` goroutines (default 512).

> Not in Essential Set: user impact surfaces first through service error rate or SDK schedule-to-start latency; remediation requires specific operational knowledge of `transferProcessorSchedulerWorkerCount`.

---

### Alert 34h — Task Scheduler Latency Critical

| Field | Value |
|---|---|
| Status | 📋 Planned |
| Severity | critical |
| Panel | Task Scheduler Latency per Operation |

**Condition:** p99 `task_latency_schedule` > 2s sustained 5m. The executor pool is severely saturated; workflow progress is actively delayed across all namespaces sharing the pod.

---

### Alert 34i — Task Scheduler Throttled

| Field | Value |
|---|---|
| Status | 📋 Planned |
| Severity | warning |
| Panel | Task Scheduler Throttled Rate per Operation |

**Condition:** Any sustained non-zero `task_scheduler_throttled` rate. Tasks are being hard-rejected by the scheduler rather than queued, confirming the pool is at capacity. Use alongside 34g/34h: throttled rising with latency rising = structural saturation, not a burst.

---

## Section 10 — History Timer Task Info

> **Dashboard panels:** Timer Task Processing Latency, Timer Task Scheduling Latency (panel 325), Total Timer Tasks Errors
> **Metrics:** `timer_processing_latency`, `shardinfo_scheduled_queue_lag`, `timer_task_errors`
> **Component:** history

### Alert 35 — Timer Task Processing Latency High

| Field | Value |
|---|---|
| Status | 📋 Planned |
| Severity | warning |
| Panel | Timer Task Processing Latency |

**Condition:** p99 timer processing latency exceeds 300ms.

---

### Alert 36 — Timer Task Processing Latency Critical

| Field | Value |
|---|---|
| Status | 📋 Planned |
| Severity | critical |
| Panel | Timer Task Processing Latency |

**Condition:** p99 timer processing latency exceeds 2s.

---

### Alert 37 — Timer Task Scheduling Lag High

| Field | Value |
|---|---|
| Status | 📋 Planned |
| Severity | warning |
| Panel | Timer Task Scheduling Latency (325) |

**Condition:** `histogram_quantile(0.99, ...(shardinfo_scheduled_queue_lag_bucket{task_category="timer"}...)) > 5`

Timers firing more than 5s late.

---

### Alert 38 — Timer Task Scheduling Lag Critical

| Field | Value |
|---|---|
| Status | ✅ Implemented |
| UID | `temporal-alert-038` |
| Severity | critical |
| Panel | 325 |
| `for` | 5m |
| `noDataState` | NoData |

**Condition:** `histogram_quantile(0.99, sum by (operation, le) (rate(shardinfo_scheduled_queue_lag_bucket{task_category="timer",service_name="history"}[5m]))) > 30`

**Threshold:** p99 lag > 30s

Timer task scheduling lag has exceeded 30 seconds. Workflow timers, scheduled activities, and workflow timeouts are firing late — deadlines, heartbeat timeouts, and schedule-to-start timeouts are affected. Note: sparse data on idle clusters can produce non-zero p99 artifacts — confirm there is actual timer activity before investigating.

**Runbook:** [38-timer-scheduling-lag-critical.md](./runbooks/38-timer-scheduling-lag-critical.md)

---

### Alert 39 — Timer Task Errors Elevated

| Field | Value |
|---|---|
| Status | 📋 Planned |
| Severity | warning |
| Panel | Total Timer Tasks Errors |

**Condition:** Sustained timer task error rate above zero.

---

## Section 11 — Workflow Stats

> **Dashboard panels:** Workflow Limit Exceeded, Blob Size Errors
> **Metrics:** `workflow_too_many_pending_*`, `blob_size_exceeded_count`
> **Component:** history

### Alert 40 — Workflow Execution Limit Exceeded

| Field | Value |
|---|---|
| Status | 📋 Planned |
| Severity | warning |
| Panel | Workflow Limit Exceeded |

**Condition:** Any workflow tasks failing due to pending activity, child workflow, or cancel request limits.

---

### Alert 41 — Blob Size Limit Exceeded

| Field | Value |
|---|---|
| Status | 📋 Planned |
| Severity | warning |
| Panel | Blob Size Errors |

**Condition:** Any requests failing due to blob size limits — indicates SDK payloads too large.

---

## Section 12 — Workflow Execution History Info

> **Dashboard panels:** Workflow History Size, Workflow History Event Count, Mutable State Size
> **Metrics:** `workflow_completed_latency`, `history_size`, `history_count`, `mutable_state_size`
> **Component:** history

### Alert 42 — Workflow History Size Warning

| Field | Value |
|---|---|
| Status | 📋 Planned |
| Severity | warning |
| Panel | Workflow History Size |

**Condition:** p99 history size exceeds 4MB for any namespace.

---

### Alert 43 — Workflow History Size Critical

| Field | Value |
|---|---|
| Status | 📋 Planned |
| Severity | critical |
| Panel | Workflow History Size |

**Condition:** p99 history size exceeds 30MB for any namespace.

---

### Alert 44 — Workflow History Event Count Warning

| Field | Value |
|---|---|
| Status | 📋 Planned |
| Severity | warning |
| Panel | Workflow History Event Count |

**Condition:** p99 event count exceeds 4,096 for any namespace.

---

### Alert 45 — Workflow History Event Count Critical

| Field | Value |
|---|---|
| Status | 📋 Planned |
| Severity | critical |
| Panel | Workflow History Event Count |

**Condition:** p99 event count exceeds 30,720 for any namespace.

---

### Alert 46 — Mutable State Size Warning

| Field | Value |
|---|---|
| Status | 📋 Planned |
| Severity | warning |
| Panel | Mutable State Size |

**Condition:** p99 mutable state size exceeds 2MB for any namespace.

---

### Alert 47 — Mutable State Size Critical

| Field | Value |
|---|---|
| Status | 📋 Planned |
| Severity | critical |
| Panel | Mutable State Size |

**Condition:** p99 mutable state size exceeds 10MB for any namespace.

---

## Section 13 — Matching Task Queue Info

> **Dashboard panels:** Sync Throttle Count (panel 403)
> **Metric:** `sync_throttle_count`
> **Component:** matching

### Alert 74 — Matching Partition Sync Throttle Active

| Field | Value |
|---|---|
| Status | ✅ Implemented |
| UID | `temporal-alert-074` |
| Severity | critical |
| Panel | 403 |
| `for` | 1m |
| `noDataState` | NoData |

**Condition:** `sum by (namespace, task_type) (rate(sync_throttle_count{service_name="matching",namespace!="_unknown_"}[5m])) > 0`

The matching service sync dispatch limit is being hit for a namespace and task type. If alert 57 is also firing, fix worker provisioning first. If pollers are healthy, increase `matching.numTaskqueueWritePartitions` and `matching.numTaskqueueReadPartitions` for the affected task queue.

**Runbook:** [74-matching-sync-throttle-active.md](./runbooks/74-matching-sync-throttle-active.md)

> **Alerts not planned for this section:** Async Match Latency high/critical — schedule-to-start alerts (48/49 in Section 14) already cover the end-user impact. Task Write Throttle Count — typically indicates worker shortage rather than partition count, redundant with Section 14. Sync Match Latency — most useful as a comparative signal alongside async match, not as a standalone alert.

---

## Section 14 — SDK Workers Info

> **Dashboard panels:** Schedule to Start Latencies, Approximate Task Backlog, Tasks Persisted to DB, Activity Timeout panels, Workflow Task StartToClose Timeouts
> **Metrics:** `schedule_to_start_latency`, `approximate_backlog_count`, `task_requests`, `*_timeout`
> **Component:** matching / history

### Alert 48 — Schedule to Start Latency High

| Field | Value |
|---|---|
| Status | 📋 Planned |
| Severity | warning |
| Panel | Schedule to Start Latencies |

**Condition:** p99 schedule-to-start latency exceeds 200ms — primary signal for insufficient workers.

---

### Alert 49 — Schedule to Start Latency Critical

| Field | Value |
|---|---|
| Status | 📋 Planned |
| Severity | critical |
| Panel | Schedule to Start Latencies |

**Condition:** p99 schedule-to-start latency exceeds 1s.

---

### Alert 50 — Task Backlog Growing

| Field | Value |
|---|---|
| Status | 📋 Planned |
| Severity | warning |
| Panel | Approximate Task Backlog |

**Condition:** Task backlog exceeds 1,000 tasks for any namespace + task type.

---

### Alert 51 — Task Backlog Critical

| Field | Value |
|---|---|
| Status | 📋 Planned |
| Severity | critical |
| Panel | Approximate Task Backlog |

**Condition:** Task backlog exceeds 2,000 tasks.

---

### Alert 52 — Tasks Persisted Rate High

| Field | Value |
|---|---|
| Status | 📋 Planned |
| Severity | warning |
| Panel | Tasks Persisted to DB |

**Condition:** `CreateTasks` rate exceeds 1,000 req/s — sustained increase indicates workers not keeping up.

---

### Alert 53 — Activity ScheduleToStart Timeout

| Field | Value |
|---|---|
| Status | 📋 Planned |
| Severity | warning |
| Panel | Activity ScheduleToStart Timeout |

**Condition:** Any sustained activity schedule-to-start timeouts — workers not polling fast enough.

---

### Alert 54 — Activity StartToClose Timeout

| Field | Value |
|---|---|
| Status | 📋 Planned |
| Severity | warning |
| Panel | Activity StartToClose Timeout |

**Condition:** Any sustained activity start-to-close timeouts.

---

### Alert 55 — Activity Heartbeat Timeout

| Field | Value |
|---|---|
| Status | 📋 Planned |
| Severity | warning |
| Panel | Activity Heartbeat Timeout |

**Condition:** Any sustained heartbeat timeouts — workers crashing during long activities.

---

### Alert 56 — Workflow Task Timeout (Sticky)

| Field | Value |
|---|---|
| Status | 📋 Planned |
| Severity | warning |
| Panel | Workflow Task StartToClose Timeouts |

**Condition:** Any sustained workflow task timeouts on sticky task queues.

---

## Section 15 — Pollers

> **Dashboard panel:** Total Concurrent Pollers (panel 162)
> **Metric:** `service_pending_requests`
> **Component:** frontend

### Alert 57 — All Pollers Disconnected

| Field | Value |
|---|---|
| Status | ✅ Implemented |
| UID | `temporal-alert-057` |
| Severity | critical |
| Panel | 162 |
| `for` | 1m |
| `noDataState` | NoData |

**Condition:** `sum by (namespace) (service_pending_requests{service_name="frontend",namespace!="_unknown_",operation=~"PollWorkflowTaskQueue|PollActivityTaskQueue"}) < 1`

No workers are actively polling for workflow or activity tasks in a namespace. Workflows stall at every task boundary. Queued tasks are persisted to the database by the matching service, adding DB pressure while pollers are absent.

**Runbook:** [57-all-pollers-disconnected.md](./runbooks/57-all-pollers-disconnected.md)

---

### Alert 58 — Pollers Approaching Concurrent Limit

| Field | Value |
|---|---|
| Status | 📋 Planned |
| Severity | warning |
| Panel | Total Concurrent Pollers (162) |

**Condition:** Concurrent pollers approaching `frontend.namespaceCount` limit (default 1,200/instance) — will trigger `ConcurrentLimit` throttling.

---

## Section 16 — Visibility

> **Dashboard panels:** Visibility Availability, Visibility Latencies per Operation, Visibility Task End-to-End Latencies, Visibility Write Error Rate per Store (panel 2118), Visibility Write Latency per Store (panel 2119)
> **Metrics:** `visibility_persistence_requests`, `visibility_persistence_latency`, `visibility_persistence_errors`
> **Component:** history

### Alert 59 — Visibility Availability Degraded

| Field | Value |
|---|---|
| Status | 📋 Planned |
| Severity | warning |
| Panel | Visibility Availability |

**Condition:** Visibility success rate drops below 99%.

---

### Alert 60 — Visibility Availability Critical

| Field | Value |
|---|---|
| Status | 📋 Planned |
| Severity | critical |
| Panel | Visibility Availability |

**Condition:** Visibility success rate drops below 95%.

---

### Alert 61 — Visibility Latency High

| Field | Value |
|---|---|
| Status | 📋 Planned |
| Severity | warning |
| Panel | Visibility Latencies per Operation |

**Condition:** p99 visibility latency exceeds 3s.

---

### Alert 62 — Visibility Latency Critical

| Field | Value |
|---|---|
| Status | 📋 Planned |
| Severity | critical |
| Panel | Visibility Latencies per Operation |

**Condition:** p99 visibility latency exceeds 5s.

---

### Alert 63 — Visibility End-to-End Latency High

| Field | Value |
|---|---|
| Status | 📋 Planned |
| Severity | warning |
| Panel | Visibility Task End-to-End Latencies |

**Condition:** End-to-end visibility task latency exceeds 3s — workflow state changes taking too long to appear in search.

---

### Alert 59a — Visibility Store Write Errors (Warning)

| Field | Value |
|---|---|
| Status | ✅ Implemented |
| UID | `temporal-alert-059a` |
| Severity | warning |
| Panel | 2118 |
| `for` | 2m |
| `noDataState` | OK |

**Condition:** `sum(rate(visibility_persistence_errors{service_name="history"}[5m])) by (visibility_index_name) > 0.1`

**Threshold:** > 0.1 errors/s

A visibility store is returning write errors at low rate. History retries indefinitely — workflows are unaffected but visibility data is accumulating in the retry queue. `noDataState: OK` — metric only emits when errors occur; absence is healthy. Only meaningful when `system.secondaryVisibilityWritingMode = dual`.

**Runbook:** [59a-visibility-store-write-errors.md](./runbooks/59a-visibility-store-write-errors.md)

---

### Alert 59b — Visibility Store Write Errors (Critical)

| Field | Value |
|---|---|
| Status | ✅ Implemented |
| UID | `temporal-alert-059b` |
| Severity | critical |
| Panel | 2118 |
| `for` | 1m |
| `noDataState` | OK |

**Condition:** `sum(rate(visibility_persistence_errors{service_name="history"}[5m])) by (visibility_index_name) > 1`

**Threshold:** > 1 error/s

Visibility store write error rate has exceeded 1 req/s — the store is effectively unavailable for writes. If `visibility_index_name = temporal_visibility` (primary), workflow list/describe/count operations will fail immediately. If secondary, workflow list still works but data is diverging.

**Runbook:** [59a-visibility-store-write-errors.md](./runbooks/59a-visibility-store-write-errors.md)

---

### Alert 59c — Visibility Store Write Latency High

| Field | Value |
|---|---|
| Status | ✅ Implemented |
| UID | `temporal-alert-059c` |
| Severity | warning |
| Panel | 2119 |
| `for` | 5m |
| `noDataState` | OK |

**Condition:** `histogram_quantile(0.99, sum(rate(visibility_persistence_latency_bucket{service_name="history"}[5m])) by (visibility_index_name, le)) > 3`

**Threshold:** p99 > 3s

p99 write latency to a visibility store has exceeded 3s. May indicate recovery from an outage (draining backlog), disk/IO pressure, or approaching connection pool exhaustion. Latency divergence between the two stores indicates one is under pressure while the other is healthy.

**Runbook:** [59a-visibility-store-write-errors.md](./runbooks/59a-visibility-store-write-errors.md)

---

## Section 18 — Cluster Replication

> **Dashboard panels:** Stream Health, Replication Latencies, DLQ Writes and Failures, Replication Task Throughput
> **Metrics:** `replication_stream_stuck`, `replication_latency`, `replication_dlq_non_empty`, `replication_dlq_enqueue_failures`
> **Component:** history
> **Note:** Replication alerts are only meaningful on multi-cluster active-standby deployments.

### Alert 64 — Replication Stream Stuck

| Field | Value |
|---|---|
| Status | 📋 Planned |
| Severity | critical |
| Panel | Stream Health |

**Condition:** `replication_stream_stuck` is non-zero sustained — replication has completely stalled.

---

### Alert 65 — Replication Stream Errors

| Field | Value |
|---|---|
| Status | 📋 Planned |
| Severity | warning |
| Panel | Stream Health |

**Condition:** Sustained replication stream errors above zero.

---

### Alert 66 — Replication Stream Panic

| Field | Value |
|---|---|
| Status | 📋 Planned |
| Severity | critical |
| Panel | Stream Health |

**Condition:** Any replication stream panics.

---

### Alert 67 — Replication Latency High

| Field | Value |
|---|---|
| Status | 📋 Planned |
| Severity | warning |
| Panel | Replication Latencies |

**Condition:** p99 end-to-end replication latency exceeds 2s.

---

### Alert 68 — Replication Latency Critical

| Field | Value |
|---|---|
| Status | 📋 Planned |
| Severity | critical |
| Panel | Replication Latencies |

**Condition:** p99 end-to-end replication latency exceeds 4s.

---

### Alert 69 — Replication DLQ Non-Empty

| Field | Value |
|---|---|
| Status | 📋 Planned |
| Severity | critical |
| Panel | DLQ Writes and Failures |

**Condition:** `replication_dlq_non_empty` is non-zero — tasks have failed past all retries and are preserved in the DLQ for inspection. Cassandra only.

---

### Alert 70 — Replication DLQ Enqueue Failures

| Field | Value |
|---|---|
| Status | 📋 Planned |
| Severity | critical |
| Panel | DLQ Writes and Failures |

**Condition:** DLQ enqueue failures detected — failed tasks cannot even be preserved for inspection. Cassandra only.

---

### Alert 71 — Replication Tasks Failing

| Field | Value |
|---|---|
| Status | 📋 Planned |
| Severity | warning |
| Panel | Replication Task Throughput |

**Condition:** Sustained replication task failure rate above zero.

---

## Section 19 — Authorization

> **Dashboard panels:** Authorization System Failures, Unauthorized Requests
> **Metrics:** `authorization_failure`, `unauthorized_request`
> **Component:** frontend

### Alert 72 — Authorization System Failure

| Field | Value |
|---|---|
| Status | 📋 Planned |
| Severity | critical |
| Panel | Authorization System Failures |

**Condition:** Any auth system failures — auth plugin broken; all requests may start failing if auth is enforced.

---

### Alert 73 — Unauthorized Request Spike

| Field | Value |
|---|---|
| Status | 📋 Planned |
| Severity | warning |
| Panel | Unauthorized Requests |

**Condition:** Sustained spike in unauthorized requests — may indicate misconfigured permissions or credential issues.

---

## Implementation Summary

| Status | Count |
|---|---|
| ✅ Implemented | 17 |
| 📋 Planned | 74 |
| **Total** | **91** |

| Severity | Implemented | Planned | Total |
|---|---|---|---|
| 🔴 Critical | 14 | 31 | 45 |
| ⚠️ Warning | 3 | 43 | 46 |
| **Total** | **17** | **74** | **91** |
