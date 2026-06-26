# Temporal Server Grafana Alerts — Planning Document

> **Status: Work in Progress** — This document is a planning artifact for review and discussion before implementation begins. Alert definitions, thresholds, and runbook structure are subject to change based on feedback.

---

## Overview

This document outlines the planned Grafana alert rules for the Temporal Server Dashboard. The goal is to provide a set of importable, generic alert rules that any self-hosted Temporal operator can use as a starting point, adjust to their environment, and extend as needed.

**The alerts themselves are secondary to the runbooks.** Each alert will be accompanied by a runbook that describes:
- What the alert detects
- Why it matters
- Which dashboard panels to look at first
- Which dynamic config values are relevant
- What actions to take

This is designed as a self-service operational guide — operators should be able to respond to any alert without prior deep knowledge of Temporal internals.

---

## Design Principles

**Generic defaults, expected to be tuned.** Thresholds are set based on Temporal's default dynamic config values. Every deployment is different — operators are expected to adjust thresholds to match their cluster size, workload, and SLO requirements.

**Every alert maps to a dashboard panel.** All alerts correspond directly to a panel in the Temporal Server Dashboard. When an alert fires, the operator knows exactly where to look.

**Two-tier severity.** Alerts are either Critical (page, immediate action required) or Warning (investigate soon). This avoids alert fatigue from too many severity levels.

**Compound conditions where appropriate.** Some alerts are only meaningful in combination — for example, shard movement is expected during a restart but unexpected without one. These are modeled as compound PromQL conditions rather than simple threshold breaches.

**Namespace-scoped and cluster-wide variants.** Where meaningful, alerts will have both a cluster-wide version and a per-namespace version so operators can alert on specific namespaces separately.

---

## Deliverables

| Deliverable | Description | Status |
|---|---|---|
| `temporal-server-alerts.yaml` | Grafana alerting provisioning file — drop into `provisioning/alerting/` and alerts load automatically on Grafana startup. Datasource referenced by name (`Prometheus`) so no UID replacement needed. | ✅ Essential Set complete (14 alerts) |
| `runbooks/` | One markdown runbook per alert — structure created upfront, triage and remediation content filled in after alerts are tested and validated in Grafana | ✅ Stubs created, content TODO after testing |
| `README.md` | Setup instructions and index of all alerts with links to runbooks | ✅ Complete |

### Implementation Notes

**Delivery format:** Grafana alerting provisioning YAML (`provisioning/alerting/`), not an importable JSON file. This avoids per-installation datasource UID replacement and scales to the full alert inventory without requiring UI imports. Operators drop the file into their Grafana provisioning directory and alerts appear on startup.

**Datasource:** Referenced by name (`Prometheus`) — the Grafana default when adding a Prometheus datasource. Operators with a differently named datasource update the `datasourceUid` field in the YAML (one value, not per-alert).

**Dashboard links:** Every alert includes `__dashboardUid__` and `__panelId__` annotations so Grafana renders a direct clickthrough to the relevant panel when the alert fires. Dashboard UID is `temporal-overview-v1`. Panel IDs are stable across dashboard versions unless a panel is explicitly removed or recreated.

**Scope:** Essential Alert Set alerts are cluster-wide (no namespace filter). Per-namespace variants are deferred to a future iteration.

**YAML structure:**
- Grafana folder: `Temporal Server` — scoped to server alerts; SDK alerts will live in a separate folder when added
- Group name: `temporal-server-essential` — single group for the Essential Set
- Evaluation interval: `1m`

**Labels:** Every alert carries `severity: critical`, `service: temporal`, and `component: <value>`. Component values by alert:

| Alerts | Component |
|---|---|
| 1, 4, 27, 57 | `frontend` |
| 8, 34, 34b, 34f, 38 | `history` |
| 12, 30, 33 | `persistence` |
| 25 | `server` |
| 74 | `matching` |

**Annotations:** Every alert carries the following annotation fields. Content is per-alert — the structure is fixed, the text is not.
- `summary`: `"Temporal: <what happened> ({{ $value | printf \"%.2f\" }})"` — one line shown in notification subject
- `description`: what the alert detects, `Immediate action: <per-alert tailored text>`, `Runbook: <full GitHub URL>`
- `__dashboardUid__`: `temporal-overview-v1`
- `__panelId__`: per-alert (see panel ID table above)

**Rate interval:** All expressions use a hardcoded `5m` rate interval. There is no `$__rate_interval` equivalent for alert rules — Grafana computes that variable from the dashboard time range, which does not exist in alert evaluation context. `5m` is a safe default covering scrape intervals from 15s to 60s.

### PromQL Expressions

All expressions are derived from the corresponding dashboard panel queries with `$__rate_interval` → `5m`, `$p` → `0.99`, and namespace filter removed.

| # | `for` | Expression |
|---|---|---|
| 1 | 5m | `sum(rate(service_requests{service_name="frontend"}[5m])) == 0` |
| 1b | 2m | `absent(rate(service_requests{service_name="frontend"}[5m]))` |
| 4 | 5m | `sum by (namespace) (rate(service_requests{service_name="frontend"}[5m])) == 0` |
| 8 | 5m | `histogram_quantile(0.99, sum by (le) (rate(semaphore_latency_bucket{operation="ShardInfo",service_name="history"}[5m]))) > 0.3` |
| 12 | 5m | `histogram_quantile(0.99, sum by (operation, le) (rate(persistence_latency_bucket{operation=~"CreateWorkflowExecution\|UpdateWorkflowExecution\|ConflictResolveWorkflowExecution\|GetWorkflowExecution\|GetCurrentExecution\|AppendHistoryNodes\|ReadHistoryBranch\|CreateTasks\|GetTasks\|GetTransferTasks\|GetTimerTasks\|CompleteTransferTask\|CompleteTimerTask"}[5m]))) > 1` |
| 25 | 1m | `sum by (service_name) (rate(service_panics[5m])) > 0` |
| 27 | 2m | `sum by (namespace) (rate(service_errors{service_name="frontend"}[5m])) / sum by (namespace) (rate(service_requests{service_name="frontend"}[5m])) > 0.3` |
| 30 | 1m | `sum(rate(service_errors_resource_exhausted{resource_exhausted_cause=~"RESOURCE_EXHAUSTED_CAUSE_SYSTEM_OVERLOADED\|RESOURCE_EXHAUSTED_CAUSE_CIRCUIT_BREAKER_OPEN"}[5m])) > 0` |
| 33 | — | removed from Essential Set — replication-only metric, not applicable to single active cluster |
| 34 | 10m | `(sum(rate(sharditem_created_count{service_name="history"}[5m])) + sum(rate(sharditem_removed_count{service_name="history"}[5m])) + sum(rate(shard_closed_count{service_name="history"}[5m])) > 0) unless (sum(increase(restarts{service_name="history"}[8m])) > 0)` |
| 34b | 15m | `histogram_quantile(0.99, sum by (instance, task_category, le) (rate(shardinfo_immediate_queue_lag_bucket{service_name="history"}[11m]))) > 3000000` |
| 34f | 1m | `sum by (instance) (dd_current_suspected_deadlocks{service_name="history"}) > 0` |
| 34j | 5m | `histogram_quantile(0.99, sum(rate(dd_shard_io_semaphore_latency_bucket{service_name="history"}[5m])) by (instance, le)) > 20` |
| 38 | 5m | `histogram_quantile(0.99, sum by (operation, le) (rate(shardinfo_scheduled_queue_lag_bucket{task_category="timer",service_name="history"}[5m]))) > 30` |
| 57 | 1m | `sum by (namespace) (service_pending_requests{service_name="frontend",namespace!="_unknown_",operation=~"PollWorkflowTaskQueue\|PollActivityTaskQueue"}) == 0` |
| 74 | 1m | `sum by (namespace, task_type) (rate(sync_throttle_count{service_name="matching"}[5m])) > 0` |

---

## Essential Alert Set

Operators who want focused coverage without implementing the full alert inventory should start here. Each alert covers a distinct, high-impact failure mode with no overlap. Together they span the full stack — frontend availability, service health, persistence, task processing, shard stability, worker connectivity, and task queue dispatch.

| # | Alert Name | Section | Panel | Panel ID | Why |
|---|---|---|---|---|---|
| 1 | Total RPS Drops to Zero | 1 — Cluster Throughput | Total RPS | 21 | Cluster is dead from the user's perspective |
| 1b | Frontend Metrics Absent | 1 — Cluster Throughput | Total RPS | 21 | Frontend pods stopped emitting metrics entirely — alert 1 cannot fire when the series disappears |
| 4 | Namespace RPS Drops to Zero | 1 — Cluster Throughput | RPS per Namespace | 20 | A specific namespace is receiving no traffic — workers disconnected or namespace misconfigured |
| 8 | Shard Lock Latency Critical | 2 — Shard and Workflow Lock Latencies | Shard Lock Latency | 14 | Shard lock pressure directly degrades all history operations |
| 12 | Persistence Latency Critical | 3 — Persistence | Persistence Latencies | 71 | DB slowness is the root cause of most cluster degradation |
| 25 | Service Panic Detected | 5 — Service Requests and Errors | Service Panics | 99 | Binary and unambiguous — a panic means something is fundamentally broken |
| 27 | Service Error Rate Critical | 5 — Service Requests and Errors | Service Errors by Namespace | 92 | High sustained frontend error rate means users are being actively impacted |
| 30 | System Overload Throttling | 6 — Throttling and Limits | Resource Exhausted with Cause | 121 | `SYSTEM_OVERLOADED` or `CIRCUIT_BREAKER_OPEN` cause means DB is overwhelmed or a circuit breaker has tripped and users are receiving errors right now |
| 33 | Transfer Tasks Being Discarded | 7 — Busy Workflow Throttling | Transfer Active Task Errors Discarded | 141 | Replication-only — `task_errors_discarded` only emits on standby clusters; not applicable to single active cluster deployments. Removed from Essential Set. |
| 34 | Unexpected Shard Movement | 8 — Shard Movement | Shards Created | 60 | Shard churn without a restart indicates DB pressure or membership instability. Runbook should also reference Service Restarts (panel 63). |
| 34b | Immediate Queue Lag Critical | 9 — Shard Queue Health | Immediate Queue Lag per Pod | 2109 | p99 immediate queue lag > 3M tasks on any `instance + task_category` — one pod's lag diverging monotonically from the fleet is the definitive stuck shard signal |
| 34f | Shard Deadlock Detected | 9 — Shard Queue Health | Suspected Deadlocks per Pod | 2113 | Binary and unambiguous — any `dd_current_suspected_deadlocks > 0` requires immediate pod restart. `noDataState: OK` because the metric is event-driven and does not emit baseline zero. |
| 38 | Timer Task Scheduling Lag Critical | 10 — History Timer Task Info | Timer Task Scheduling Latency | 325 | Timers firing 30s+ late means workflow timeouts and scheduled actions are functionally broken |
| 57 | All Pollers Disconnected | 15 — Pollers | Total Concurrent Pollers | 162 | All workers gone for a namespace — complete task processing halt |
| 74 | Matching Partition Sync Throttle Active | 13 — Matching Task Queue Info | Sync Throttle Count | 403 | Per-partition dispatch limit hit — if alert 57 is not also firing, task queue partitions need to be increased |

> **Dashboard UID for all panel links:** `temporal-overview-v1`

---

## Alert Inventory

### Severity Legend
- 🔴 **Critical** — Page immediately, immediate action required
- ⚠️ **Warning** — Investigate soon, action required but not urgent

---

### Section 0 — History Host Health

> **Dashboard:** These alerts correspond to the **[Temporal History Host Health Dashboard](../../dashboards/server/history-health-dashboard.json)** (`history-health-dashboard.json`), not the main Temporal Server Dashboard. This is the only section in this document that references a companion dashboard. All other sections (1–16) map to panels in `temporal-server.json`.

| # | Alert Name | Severity | Panel | Condition |
|---|---|---|---|---|
| 0a | History Pod Disappeared | 🔴 Critical | Fleet Size Change | `clamp_min(max_over_time(count(host_health{service_name="history"})[1h:1m]) - (count(host_health{service_name="history"}) or vector(0)), 0) >= 1` — fires when any pod stops emitting `host_health` compared to the fleet baseline seen in the last hour. A stopped or crashed pod goes absent rather than reporting NOT_SERVING, so this alert is required to catch pod loss. |
| 0b | History Pod Degraded | 🔴 Critical | Pods NOT_SERVING | `sum(clamp_max(host_health{service_name="history"} == 2, 1)) >= 1` — fires when any pod is actively reporting NOT_SERVING. Covers persistence/RPC threshold breaches and gRPC health failures past the 60s init window. |
| 0c | Metric Freshness Stale | 🔴 Critical | Metric Freshness | `time() - max(timestamp(host_health{service_name="history"})) > 120` — fires when `host_health` hasn't been updated in 120 seconds. Means the poller can't reach the frontend or the frontend is down. No `host_health` data from a running poller is itself a health signal. |

> **Note on alert 0a vs 0b**: these cover two distinct failure modes. A pod that crashes or is killed stops emitting `host_health` entirely — it will never appear as NOT_SERVING. A pod that is running but degraded (persistence latency, RPC errors, gRPC health not SERVING past init window) will emit `host_health = 2`. Both alerts are required; neither subsumes the other.

---

### Section 1 — Cluster Throughput

| # | Alert Name | Severity | Panel | Condition |
|---|---|---|---|---|
| 1 | Total RPS Drops to Zero | 🔴 Critical | Total RPS | Total frontend RPS drops to zero — frontend service is down, network connectivity is broken, or load balancer is misconfigured |
| 1b | Frontend Metrics Absent | 🔴 Critical | Total RPS | `absent()` guard for alert 1 — fires when `service_requests{service_name="frontend"}` series disappears entirely (pod crash, scrape failure). Alert 1 cannot detect this case because a missing series produces no value to evaluate. `noDataState: OK` required so Grafana treats the normal (series present) case as healthy. |
| 2 | Total RPS Approaching Limit | ⚠️ Warning | Total RPS | Total frontend RPS exceeds orange threshold (default 2,000 — assumes ~3 frontend instances, adjust to `frontend.rps` × instance count) |
| 3 | Total RPS Over Limit | 🔴 Critical | Total RPS | Total frontend RPS exceeds red threshold (default 7,000) — active throttling is happening, users are receiving `RpsLimit` errors |
| 4 | Namespace RPS Drops to Zero | 🔴 Critical | RPS per Namespace | Per-namespace RPS drops to zero — namespace is receiving no traffic, workers may have disconnected or namespace is misconfigured |
| 5 | Namespace RPS Approaching Limit | ⚠️ Warning | RPS per Namespace | Per-namespace RPS exceeds `frontend.namespaceRPS` warning level (default 2,400/host) |
| 6 | Namespace RPS Over Limit | 🔴 Critical | RPS per Namespace | Per-namespace RPS exceeds `frontend.namespaceRPS` hard limit — users are receiving `RpsLimit` throttle errors for this namespace |

---

### Section 2 — Shard and Workflow Lock Latencies

| # | Alert Name | Severity | Panel | Condition |
|---|---|---|---|---|
| 7 | Shard Lock Latency High | ⚠️ Warning | Shard Lock Latency | p99 shard lock latency exceeds 150ms |
| 8 | Shard Lock Latency Critical | 🔴 Critical | Shard Lock Latency | p99 shard lock latency exceeds 300ms |
| 9 | Workflow Lock Latency High | ⚠️ Warning | Workflow Lock Latency | p99 workflow lock latency exceeds 200ms |
| 10 | Workflow Lock Latency Critical | 🔴 Critical | Workflow Lock Latency | p99 workflow lock latency exceeds 400ms |

---

### Section 3 — Persistence

| # | Alert Name | Severity | Panel | Condition |
|---|---|---|---|---|
| 11 | Persistence Latency High | ⚠️ Warning | Persistence Latencies | p99 persistence latency exceeds 300ms for any operation |
| 12 | Persistence Latency Critical | 🔴 Critical | Persistence Latencies | p99 persistence latency exceeds 1s for any operation |
| 13 | Persistence QPS Approaching Limit | ⚠️ Warning | Persistence Requests Total | Total persistence req/s exceeds 7,000 (based on `history.persistenceMaxQPS` default 9,000/host) |
| 14 | Persistence Availability Degraded | ⚠️ Warning | Persistence Availability | Persistence success rate drops below 99% |
| 15 | Persistence Availability Critical | 🔴 Critical | Persistence Availability | Persistence success rate drops below 95% |
| 16 | Persistence Errors Elevated | ⚠️ Warning | Persistence Errors by Namespace | Sustained persistence error rate above zero for any namespace |
| 17 | SQL Connection Pool Near Saturation | ⚠️ Warning | SQL DB Connection Pool | Open connections exceeding 90% of configured max (SQL backends only) |
| 18 | SQL Connection Pool Saturated | 🔴 Critical | SQL DB Connection Pool | Open connections at or exceeding configured max (SQL backends only) |

---

### Section 4 — Service Latencies

| # | Alert Name | Severity | Panel | Condition |
|---|---|---|---|---|
| 19 | Frontend Service Latency High | ⚠️ Warning | Frontend Service Latency | p99 frontend latency exceeds 300ms (excludes PollWorkflowTaskQueue, PollActivityTaskQueue) |
| 20 | Frontend Service Latency Critical | 🔴 Critical | Frontend Service Latency | p99 frontend latency exceeds 2s |
| 21 | History Service Latency High | ⚠️ Warning | History Service Latency | p99 history latency exceeds 400ms |
| 22 | History Service Latency Critical | 🔴 Critical | History Service Latency | p99 history latency exceeds 2s |
| 23 | Matching Service Latency High | ⚠️ Warning | Matching Service Latency | p99 matching latency exceeds 400ms (excludes PollWorkflowTaskQueue, PollActivityTaskQueue, MatchingClientGetTaskQueueUserData) |
| 24 | Matching Service Latency Critical | 🔴 Critical | Matching Service Latency | p99 matching latency exceeds 2s |

---

### Section 5 — Service Requests and Errors

| # | Alert Name | Severity | Panel | Condition |
|---|---|---|---|---|
| 25 | Service Panic Detected | 🔴 Critical | Service Panics | Any service panic rate above zero — a Go panic means an unrecoverable error that can cause shard loss or task corruption |
| 26 | Service Error Rate Elevated | ⚠️ Warning | Service Errors by Namespace | Sustained service error rate for any namespace |
| 27 | Service Error Rate Critical | 🔴 Critical | Service Errors by Namespace | High sustained service error rate for any namespace |
| 28 | Namespace RPS Approaching Limit | ⚠️ Warning | Service Requests by Namespace | Per-namespace request rate approaching `frontend.namespaceRPS` limit |

---

### Section 6 — Throttling and Limits

| # | Alert Name | Severity | Panel | Condition |
|---|---|---|---|---|
| 29 | RPS/QPS Throttling Active | ⚠️ Warning | Resource Exhausted with Cause | Resource exhausted errors with cause `RpsLimit` or `QpsLimit` |
| 30 | System Overload Throttling | 🔴 Critical | Resource Exhausted with Cause | Resource exhausted errors with cause `SYSTEM_OVERLOADED` or `CIRCUIT_BREAKER_OPEN` — indicates DB is overwhelmed or a circuit breaker has tripped |

---

### Section 7 — Busy Workflow Throttling

| # | Alert Name | Severity | Panel | Condition |
|---|---|---|---|---|
| 31 | Busy Workflow Throttling Elevated | ⚠️ Warning | Transfer Active Task Errors Workflow Busy | Sustained busy workflow error rate |
| 32 | Busy Workflow Throttling Critical | 🔴 Critical | Transfer Active Task Errors Workflow Busy | High sustained busy workflow error rate |
| 33 | Transfer Tasks Being Discarded | 🔴 Critical | Transfer Active Task Errors Discarded | Any tasks being discarded — cluster gave up retrying, tasks are lost |

---

### Section 8 — Shard Movement

| # | Alert Name | Severity | Panel | Condition |
|---|---|---|---|---|
| 34 | Unexpected Shard Movement | 🔴 Critical | Shards Created/Removed/Closed + Service Restarts | Shard churn rate is elevated AND service restarts == 0 in the same window — shard movement without a restart indicates DB pressure, history host crash, or membership instability |

> **Note:** Shard movement during a planned restart or scaling event is expected and not alertable. This alert uses a compound condition to filter out the expected case.

---

### Section 9 — Shard Queue Health

| # | Alert Name | Severity | Panel | Condition |
|---|---|---|---|---|
| 34a | Immediate Queue Lag High | ⚠️ Warning | Immediate Queue Lag per Pod | p99 immediate queue lag exceeds 500K tasks on any `instance + task_category` — one pod's queue growing while others recover is an early stuck shard signal |
| 34b | Immediate Queue Lag Critical | 🔴 Critical | Immediate Queue Lag per Pod | p99 immediate queue lag exceeds 3M tasks on any `instance + task_category` — shard is definitively stuck; manual intervention required |
| 34c | Scheduled Queue Lag High | ⚠️ Warning | Scheduled Queue Lag per Pod | p99 scheduled queue lag exceeds 10 minutes on any `instance + task_category` |
| 34d | Scheduled Queue Lag Critical | 🔴 Critical | Scheduled Queue Lag per Pod | p99 scheduled queue lag exceeds 30 minutes on any `instance + task_category` |
| 34e | DB Pool Refresh Failure Elevated | 🔴 Critical | DB Pool Refresh Failure Rate per Pod | Any DB pool refresh failures detected — earliest signal for DB-caused stuck shards; fires before queue lag builds. SQL backends only. |
| 34f | Shard Deadlock Detected | 🔴 Critical | Suspected Deadlocks per Pod | Any current suspected deadlocks on any history pod — `dd_current_suspected_deadlocks > 0` requires immediate pod restart. `noDataState: OK` — metric is event-driven and does not emit baseline zero. |
| 34g | Task Scheduler Latency High | ⚠️ Warning | Task Scheduler Latency per Operation | p99 `task_latency_schedule` > 500ms sustained 5m — the host-level executor pool is starting to saturate. Most commonly caused by a namespace doing bulk operations (mass terminations, deletions) that consume all `history.transferProcessorSchedulerWorkerCount` goroutines (default 512). Not Essential Set: user impact surfaces first through service error rate or SDK schedule-to-start latency; remediation requires specific operational knowledge of `transferProcessorSchedulerWorkerCount`. |
| 34h | Task Scheduler Latency Critical | 🔴 Critical | Task Scheduler Latency per Operation | p99 `task_latency_schedule` > 2s sustained 5m — executor pool is severely saturated; workflow progress is actively delayed across all namespaces sharing the pod. `for: 5m` required to avoid false positives from transient bulk-operation bursts. |
| 34i | Task Scheduler Throttled | ⚠️ Warning | Task Scheduler Throttled Rate per Operation | Any sustained non-zero `task_scheduler_throttled` rate — tasks are being hard-rejected by the scheduler rather than queued, confirming the pool is at capacity. Use alongside 34g/34h: throttled rising with latency rising = structural saturation, not a burst. |
| 34j | Shard IO Semaphore Deadlock Approaching | ⚠️ Warning | Shard IO Concurrency Dashboard | `dd_shard_io_semaphore_latency` p99 > 20s sustained 5m — the deadlock detector's own health ping is being starved by live traffic. The detector timeout is 40s (10s DB op timeout + 30s grace, from source); this alert fires at half that threshold to allow intervention before alert 34f fires and a pod restart becomes mandatory. SQL and Cassandra both applicable — Cassandra can still experience semaphore saturation even at concurrency=1 under extreme load. |

> **Note:** All Shard Queue Health alerts aggregate `sum by (instance, task_category, le)` to preserve per-pod isolation. A stuck shard on one pod will not be diluted by healthy shards on other pods — the stuck pod's lag diverges visibly from the rest of the fleet. Alerts 34c–34e apply to SQL backends only (Cassandra does not expose DB pool refresh metrics). Alert 34f uses `noDataState: OK` — absence of data means no deadlocks detected, which is the healthy state. Alerts 34g–34i are not in the Essential Set: the user-visible impact of scheduler saturation is already captured by alert 27 (Service Error Rate Critical) and SDK-side alerts 48/49 (schedule-to-start latency); remediation (tuning `history.transferProcessorSchedulerWorkerCount` via dynamic config, cell-level only, no restart required) requires operational knowledge not suited to an on-call runbook.

---

### Section 10 — History Timer Task Info

| # | Alert Name | Severity | Panel | Condition |
|---|---|---|---|---|
| 35 | Timer Task Processing Latency High | ⚠️ Warning | Timer Task Processing Latency | p99 timer processing latency exceeds 300ms |
| 36 | Timer Task Processing Latency Critical | 🔴 Critical | Timer Task Processing Latency | p99 timer processing latency exceeds 2s |
| 37 | Timer Task Scheduling Lag High | ⚠️ Warning | Timer Task Scheduling Latency | Timers firing more than 5s late |
| 38 | Timer Task Scheduling Lag Critical | 🔴 Critical | Timer Task Scheduling Latency | Timers firing more than 30s late |
| 39 | Timer Task Errors Elevated | ⚠️ Warning | Total Timer Tasks Errors | Sustained timer task error rate |

---

### Section 11 — Workflow Stats

| # | Alert Name | Severity | Panel | Condition |
|---|---|---|---|---|
| 40 | Workflow Execution Limit Exceeded | ⚠️ Warning | Workflow Limit Exceeded | Any workflow tasks failing due to pending activity, child workflow, or cancel request limits |
| 41 | Blob Size Limit Exceeded | ⚠️ Warning | Blob Size Errors | Any requests failing due to blob size limits — indicates SDK payloads too large |

---

### Section 12 — Workflow Execution History Info

| # | Alert Name | Severity | Panel | Condition |
|---|---|---|---|---|
| 42 | Workflow History Size Warning | ⚠️ Warning | Workflow History Size | p99 history size exceeds 4MB for any namespace |
| 43 | Workflow History Size Critical | 🔴 Critical | Workflow History Size | p99 history size exceeds 30MB for any namespace |
| 44 | Workflow History Event Count Warning | ⚠️ Warning | Workflow History Event Count | p99 event count exceeds 4,096 for any namespace |
| 45 | Workflow History Event Count Critical | 🔴 Critical | Workflow History Event Count | p99 event count exceeds 30,720 for any namespace |
| 46 | Mutable State Size Warning | ⚠️ Warning | Mutable State Size | p99 mutable state size exceeds 2MB for any namespace |
| 47 | Mutable State Size Critical | 🔴 Critical | Mutable State Size | p99 mutable state size exceeds 10MB for any namespace |
| 75 | ReadHistoryBranch Latency High | ⚠️ Warning | Persistence Latency by Operation | p99 `ReadHistoryBranch` latency exceeds 5s — approaching the 10s `RecordWorkflowTaskStarted` context deadline; WFT dispatch will begin failing before latency reaches 10s. Alert 12 covers persistence broadly at 1s; this alert is operation-specific and fires at the threshold where WFT dispatch is at risk. Cross-check `complete_workflow_task_sticky_disabled_count` (alert 77): if sticky eviction is also elevated, full-history reads from slow workers are driving the pressure. |

---

### Section 13 — Matching Task Queue Info

| # | Alert Name | Severity | Panel | Condition |
|---|---|---|---|---|
| 74 | Matching Partition Sync Throttle Active | 🔴 Critical | Sync Throttle Count | Any sustained non-zero sync throttle rate — the per-partition dispatch limit (default 1,000 tasks/s) is being hit. Triage by cross-checking alert 57 (All Pollers Disconnected): if alert 57 is also firing, fix worker provisioning first. If alert 57 is not firing, workers are healthy and the bottleneck is partition count — increase `matching.numTaskqueuePartitions`. |

> **Alerts not planned:** Async Match Latency high/critical is intentionally omitted — schedule-to-start alerts (48/49 in Section 14) already cover the end-user impact. Task Write Throttle Count is omitted because it typically indicates worker shortage rather than partition count, making it redundant with Section 13. Sync Match Latency is omitted as it is most useful as a comparative signal alongside async match, not as a standalone alert.

---

### Section 14 — SDK Workers Info

| # | Alert Name | Severity | Panel | Condition |
|---|---|---|---|---|
| 48 | Schedule to Start Latency High | ⚠️ Warning | Schedule to Start Latencies | p99 schedule-to-start latency exceeds 200ms — primary signal for insufficient workers |
| 49 | Schedule to Start Latency Critical | 🔴 Critical | Schedule to Start Latencies | p99 schedule-to-start latency exceeds 1s |
| 50 | Task Backlog Growing | ⚠️ Warning | Approximate Task Backlog | Task backlog exceeds 1,000 tasks for any namespace + task type |
| 51 | Task Backlog Critical | 🔴 Critical | Approximate Task Backlog | Task backlog exceeds 2,000 tasks |
| 52 | Tasks Persisted Rate High | ⚠️ Warning | Tasks Persisted to DB | CreateTasks rate exceeds 1,000 req/s — sustained increase indicates workers not keeping up |
| 53 | Activity ScheduleToStart Timeout | ⚠️ Warning | Activity ScheduleToStart Timeout | Any sustained activity schedule-to-start timeouts — workers not polling fast enough |
| 54 | Activity StartToClose Timeout | ⚠️ Warning | Activity StartToClose Timeout | Any sustained activity start-to-close timeouts |
| 55 | Activity Heartbeat Timeout | ⚠️ Warning | Activity Heartbeat Timeout | Any sustained heartbeat timeouts — workers crashing during long activities |
| 56 | Workflow Task Timeout (Sticky) | ⚠️ Warning | Workflow Task StartToClose Timeouts | Any sustained workflow task timeouts on sticky task queues |
| 76 | Workflow Task ScheduleToStart Timeout | ⚠️ Warning | Schedule to Start Latencies | Any sustained `schedule_to_start_timeout` on workflow tasks — tasks expired in matching before a worker polled. Lagging confirmation that WFTs are stuck; by the time this fires, tasks have already been waiting longer than the S2S timeout. Cross-check alert 75 (ReadHistoryBranch latency) and alert 77 (sticky eviction) to determine whether the root cause is DB pressure or worker throughput. |
| 77 | Sticky Task Queue Not Being Used | ⚠️ Warning | Workflow Task Sticky Completion Rate | `complete_workflow_task_sticky_disabled_count` rising relative to `complete_workflow_task_sticky_enabled_count` — workers are not setting or maintaining sticky task queues. These metrics are emitted on WFT completion based on whether the SDK included `StickyAttributes` in the response; they are not a direct server-side eviction counter (the actual sticky→non-sticky fallback when matching returns `StickyWorkerUnavailable` has no dedicated metric). When sticky is not in use, every WFT dispatch triggers a full `ReadHistoryBranch` read instead of a partial one — a significant DB impact for long-running workflows. Leading indicator for alert 75. |

---

### Section 15 — Pollers

| # | Alert Name | Severity | Panel | Condition |
|---|---|---|---|---|
| 57 | All Pollers Disconnected | 🔴 Critical | Total Concurrent Pollers | Concurrent pollers drops to zero for a namespace — all workers disconnected, no tasks will be processed |
| 58 | Pollers Approaching Concurrent Limit | ⚠️ Warning | Total Concurrent Pollers | Concurrent pollers approaching `frontend.namespaceCount` limit (default 1,200/instance) — will trigger `ConcurrentLimit` throttling |

---

### Section 16 — Visibility

| # | Alert Name | Severity | Panel | Condition |
|---|---|---|---|---|
| 59 | Visibility Availability Degraded | ⚠️ Warning | Visibility Availability | Visibility success rate drops below 99% |
| 60 | Visibility Availability Critical | 🔴 Critical | Visibility Availability | Visibility success rate drops below 95% |
| 61 | Visibility Latency High | ⚠️ Warning | Visibility Latencies per Operation | p99 visibility latency exceeds 3s |
| 62 | Visibility Latency Critical | 🔴 Critical | Visibility Latencies per Operation | p99 visibility latency exceeds 5s |
| 63 | Visibility End-to-End Latency High | ⚠️ Warning | Visibility Task End-to-End Latencies | End-to-end visibility task latency exceeds 3s — workflow state changes taking too long to appear in search |

#### Dual Visibility Store Alerts (implemented — dashboard v2.5.0+)

These alerts use `visibility_persistence_*` metrics with the `visibility_index_name`
label (`temporal_visibility` = primary, `temporal_visibility_secondary` = secondary).
Only meaningful when `system.secondaryVisibilityWritingMode = dual`. The `visibility_persistence_*`
metrics carry no `namespace` label — no namespace filter is applied.

| # | Alert Name | Severity | Panel | Metric | Threshold |
|---|---|---|---|---|---|
| 59a | Visibility Store Write Errors (Warning) | ⚠️ Warning | Visibility Write Error Rate per Store (2118) | `visibility_persistence_errors` by `visibility_index_name` | > 0.1 req/s for 2m |
| 59b | Visibility Store Write Errors (Critical) | 🔴 Critical | Visibility Write Error Rate per Store (2118) | `visibility_persistence_errors` by `visibility_index_name` | > 1 req/s for 1m |
| 59c | Visibility Store Write Latency High | ⚠️ Warning | Visibility Write Latency per Store (2119) | `visibility_persistence_latency` p99 by `visibility_index_name` | > 3s for 5m |

Runbook: `runbooks/59a-visibility-store-write-errors.md`

---

### Section 17 — Worker Registry (In-memory)

> **No alerts planned for this section.** The dashboard panels (Workers Added/Removed, cache utilization, activity slot usage) are useful for observing worker churn and registry capacity, but registry capacity issues are expected to surface first through Section 14 signals (schedule-to-start latency, task backlog). Consider adding a registry capacity saturation alert (utilization > 100% sustained) in a future iteration.

---

### Section 18 — Cluster Replication

| # | Alert Name | Severity | Panel | Condition |
|---|---|---|---|---|
| 64 | Replication Stream Stuck | 🔴 Critical | Stream Health | `replication_stream_stuck` is non-zero sustained — replication has completely stalled |
| 65 | Replication Stream Errors | ⚠️ Warning | Stream Health | Sustained replication stream errors |
| 66 | Replication Stream Panic | 🔴 Critical | Stream Health | Any replication stream panics |
| 67 | Replication Latency High | ⚠️ Warning | Replication Latencies | p99 end-to-end replication latency exceeds 2s |
| 68 | Replication Latency Critical | 🔴 Critical | Replication Latencies | p99 end-to-end replication latency exceeds 4s |
| 69 | Replication DLQ Non-Empty | 🔴 Critical | DLQ Writes and Failures | `replication_dlq_non_empty` is non-zero — tasks have failed past all retries (Cassandra only) |
| 70 | Replication DLQ Enqueue Failures | 🔴 Critical | DLQ Writes and Failures | DLQ enqueue failures detected — failed tasks cannot even be preserved for inspection (Cassandra only) |
| 71 | Replication Tasks Failing | ⚠️ Warning | Replication Task Throughput | Sustained replication task failure rate |

---

### Section 19 — Authorization

| # | Alert Name | Severity | Panel | Condition |
|---|---|---|---|---|
| 72 | Authorization System Failure | 🔴 Critical | Authorization System Failures | Any auth system failures — auth plugin broken, all requests may start failing |
| 73 | Unauthorized Request Spike | ⚠️ Warning | Unauthorized Requests | Sustained spike in unauthorized requests — may indicate misconfigured permissions or credential issues |

---

## Summary

| Severity | Count |
|---|---|
| 🔴 Critical | 45 |
| ⚠️ Warning | 49 |
| **Total** | **94** |

> Counts include dual visibility store alerts 59a–59c (implemented 2026-06-11).

---

## Related Resources

- [Temporal Server Dashboard README](../../dashboards/server/temporal-server-readme.md)
- [Temporal Dynamic Config Reference](../../../dynamic_config/README.md)