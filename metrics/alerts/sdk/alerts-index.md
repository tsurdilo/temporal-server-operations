# Temporal SDK — Full Alert Index

Complete reference for all 36 SDK alert definitions. Descriptions, thresholds, and per-SDK PromQL expressions for every alert in the inventory.

> **Essential Set:** A curated subset of these alerts has been selected for deployment. See [README.md](./README.md) for the Essential Set, setup instructions, and runbook links.
> **Planning document:** See [planning.md](./planning.md) for design decisions and the full working notes.

---

## Table of Contents

- [Per-SDK Metric Naming](#per-sdk-metric-naming)
- [Section 1 — gRPC Request Failures](#section-1--grpc-request-failures) (#1a–#13)
- [Section 2 — Worker Lifecycle](#section-2--worker-lifecycle) (#14–#15)
- [Section 3 — Worker Cache](#section-3--worker-cache) (#16)
- [Section 4 — Workflow Executions](#section-4--workflow-executions) (#17)
- [Section 5 — Schedule To Start Latencies](#section-5--schedule-to-start-latencies) (#18–#20b)
- [Section 6 — Workflow Task Info](#section-6--workflow-task-info) (#21–#25)
- [Section 7 — Activity Task Info](#section-7--activity-task-info) (#26–#27)
- [Section 8 — Local Activity Info](#section-8--local-activity-info) (#28–#30)
- [Section 9 — Request Latency](#section-9--request-latency) (#31)
- [Section 10 — Execution Latencies](#section-10--execution-latencies) (#32–#35)
- [Section 11 — Workflow Lifecycle Rates](#section-11--workflow-lifecycle-rates) (#36)
- [Alerts Not Planned](#alerts-not-planned)

---

## Per-SDK Metric Naming

PromQL expressions in this document are shown in **Java Micrometer** form. Apply the following rules for other reporters:

| Reporter | Counter suffix | Histogram bucket suffix | `status_code` label casing |
|---|---|---|---|
| Java Micrometer | `_total` | `_seconds_bucket` | `UPPER_CASE` |
| Java OTel | `_total` | `_seconds_bucket` | `UPPER_CASE` |
| Go SDK | `_total` | `_seconds_bucket` | `PascalCase` (e.g. `NotFound`) |
| Core SDK (Rust/Python/.NET) | `_total` | `_seconds_bucket` | `UPPER_CASE` |

> `temporal_num_pollers` is a **gauge** — no `_total` suffix on any reporter.
> `temporal_worker_task_slots_available` is a **gauge** — no `_total` suffix on any reporter.
> `temporal_sticky_cache_size` is a **gauge** — no `_total` suffix on any reporter.

---

## Section 1 — gRPC Request Failures

> **SDK dashboard panels:** Request Failures (panel 5), Long-Poll Request Failures (panel 6)
> **Metrics:** `temporal_request_failure` / `temporal_long_request_failure`
> **Tags:** `namespace`, `operation`, `status_code`, `task_queue`

`temporal_request_failure` is incremented when the server returns a non-OK gRPC status code on a standard (non-long-poll) operation. `temporal_long_request_failure` covers poll operations and long-poll `GetWorkflowExecutionHistory`. Severity depends on the status code and the operation.

Note: `UpdateWithStartWorkflowExecution` maps to the gRPC operation name `ExecuteMultiOperation` in SDK metrics.

---

### Alert 1a — Request Failure: NOT_FOUND on WFT Respond Operations

| Field | Value |
|---|---|
| **Severity** | 🔴 Critical |
| **`for`** | 5m |
| **Metric** | `temporal_request_failure` |
| **Operations** | `RespondWorkflowTaskCompleted`, `RespondWorkflowTaskFailed` |
| **Status code** | `NOT_FOUND` |

**Description:** Worker responded with workflow task completion after the server already timed out the workflow task, or the workflow execution is no longer running. If the execution is no longer running, check whether it was explicitly terminated or whether an execution run timeout caused the server to time it out before the worker completed the task. First, check `temporal_workflow_task_execution_latency` — if elevated, check `temporal_workflow_task_replay_latency` next. If replay latency is high, this typically indicates too many replays or high latency during replay (check your data converters). If replay latency is normal but WFT execution latency is high, check worker resources (CPU, memory) and consider cold-start issues on the pod that picked up the task (check `identity` in the `WorkflowTaskStarted` event). If WFT execution latency exceeds the WFT timeout (default 10s), the server writes `WorkflowTaskTimedOut` events to history and reschedules the task — this adds delays to `temporal_workflow_endtoend_latency`. If you are running local activities, a WFT timeout will cause re-execution of local activities on the retried task — local activity results are not recorded in history between WFT heartbeats, so re-execution means they run again from scratch; if they are not idempotent this can cause duplicate side effects and real business impact. Also check RESOURCE_EXHAUSTED on respond operations alert (#2b) — on some SDK versions, server throttling on respond operations can cascade into a workflow task timeout. If SDK-side metrics look normal and the execution was not terminated or timed out, check the **Frontend Service Latency** panel (§4 — Service Latencies) filtered to `RespondWorkflowTaskCompleted`, and the **Persistence Latencies** panel (§3) filtered to `UpdateWorkflowExecution`.

**PromQL (Java Micrometer):**
```promql
sum by (namespace, status_code, operation) (
  rate(temporal_request_failure_total{status_code="NOT_FOUND", operation=~"RespondWorkflowTaskCompleted|RespondWorkflowTaskFailed"}[5m])
) > 0
```

---

### Alert 1b — Request Failure: NOT_FOUND on Activity Respond Operations

| Field | Value |
|---|---|
| **Severity** | 🔴 Critical |
| **`for`** | 5m |
| **Metric** | `temporal_request_failure` |
| **Operations** | `RespondActivityTaskCompleted`, `RespondActivityTaskFailed` |
| **Status code** | `NOT_FOUND` |

**Description:** Worker responded with activity task completion or failure after the server already timed out the activity task, or the workflow execution is no longer running. Root causes: (1) activity task timed out — the activity ran longer than its `scheduleToClose` or `startToClose` timeout; (2) workflow execution is no longer running — completed, explicitly terminated, or hit a workflow run timeout; (3) worker restart mid-execution — the in-flight task token is lost, the server reschedules the activity but the original worker may still attempt to respond. Check `temporal_activity_execution_latency` for the affected activity type — if elevated and approaching or exceeding your `startToClose` timeout, that is the direct cause. Also check worker CPU and memory utilization, and consider cold-start issues on the pod that picked up the task — identify the worker via the `identity` field in the `ActivityTaskStarted` event. If activity latency looks normal, check the **Frontend Service Latency** panel (§4) filtered to `RespondActivityTaskCompleted` and `RespondActivityTaskFailed`, and the **Persistence Latencies** panel (§3) filtered to `UpdateWorkflowExecution`.

**PromQL (Java Micrometer):**
```promql
sum by (namespace, status_code, operation) (
  rate(temporal_request_failure_total{status_code="NOT_FOUND", operation=~"RespondActivityTaskCompleted|RespondActivityTaskFailed"}[5m])
) > 0
```

---

### Alert 1c — Request Failure: NOT_FOUND on RecordActivityTaskHeartbeat

| Field | Value |
|---|---|
| **Severity** | ⚠️ Warning |
| **`for`** | 5m |
| **Metric** | `temporal_request_failure` |
| **Operation** | `RecordActivityTaskHeartbeat` |
| **Status code** | `NOT_FOUND` |

**Description:** The server returned NOT_FOUND when the worker attempted to heartbeat a running activity. The most common cause is the activity's `heartbeatTimeout` firing before the worker's next heartbeat call — when this happens, the server cancels the in-flight task and schedules it for retry. A second cause is `startToClose` timeout: if the activity has been running longer than its `startToClose` allows, the server times it out at the history level while the activity is still executing on the worker; the next heartbeat attempt then returns NOT_FOUND. Also check worker CPU utilization — a CPU-starved worker slows down between heartbeat calls, which can push the inter-heartbeat gap past `heartbeatTimeout` even when the activity is making progress. Cross-check RESOURCE_EXHAUSTED on `RecordActivityTaskHeartbeat` (alert #2d) — if the server is throttling heartbeat calls, the effective heartbeat interval increases and can trigger a server-side timeout even when the worker is calling on time. Finally, check history service memory: the last heartbeat details payload is stored in shard mutable state and held in memory for the life of the activity attempt — large heartbeat payloads on high-throughput activity workers can contribute to history host memory pressure. Note: normal workflow-side cancellation returns `CancelRequested=true` in the heartbeat response body, not a gRPC error, so NOT_FOUND on this operation is a reliable signal of timeout or forced closure rather than intentional cancellation.

**PromQL (Java Micrometer):**
```promql
sum by (namespace, status_code, operation) (
  rate(temporal_request_failure_total{status_code="NOT_FOUND", operation="RecordActivityTaskHeartbeat"}[5m])
) > 0
```

---

### Alert 2a — Request Failure: RESOURCE_EXHAUSTED on Critical User-Facing Operations

| Field | Value |
|---|---|
| **Severity** | 🔴 Critical |
| **`for`** | 1m |
| **Metric** | `temporal_request_failure` |
| **Operations** | `StartWorkflowExecution`, `SignalWithStartWorkflowExecution`, `ExecuteMultiOperation`, `UpdateWorkflowExecution`, `SignalWorkflowExecution` |
| **Status code** | `RESOURCE_EXHAUSTED` |

**Description:** Server is throttling critical user-facing operations. The SDK will retry these for up to 60 seconds — in many cases this still results in business impact as callers experience elevated latency. If throttling continues past 60 seconds the call fails and the client receives an error which should be handled in application code (log these failures so you can potentially backfill starts, signals, and updates if needed). From the server's perspective these operations are throttled last — if you are seeing RESOURCE_EXHAUSTED on these operations it is a strong indication that there is a significant amount of throttling happening across the cluster. Check the **Resource Exhausted with Cause** panel (§6 — Throttling and Limits) — if server-side throttling is elevated, also check the **Persistence Latencies** panel (§3) filtered to `CreateWorkflowExecution` and `UpdateWorkflowExecution`. Note: `UpdateWithStartWorkflowExecution` maps to the gRPC operation name `ExecuteMultiOperation` in SDK metrics.

**PromQL (Java Micrometer):**
```promql
sum by (namespace, status_code, operation) (
  rate(temporal_request_failure_total{status_code="RESOURCE_EXHAUSTED", operation=~"StartWorkflowExecution|SignalWithStartWorkflowExecution|ExecuteMultiOperation|UpdateWorkflowExecution|SignalWorkflowExecution"}[5m])
) > 0
```

---

### Alert 2b — Request Failure: RESOURCE_EXHAUSTED on Respond Operations

| Field | Value |
|---|---|
| **Severity** | 🔴 Critical |
| **`for`** | 5m |
| **Metric** | `temporal_request_failure` |
| **Operations** | `RespondWorkflowTaskCompleted`, `RespondWorkflowTaskFailed`, `RespondActivityTaskCompleted`, `RespondActivityTaskFailed` |
| **Status code** | `RESOURCE_EXHAUSTED` |

**Description:** Server is throttling workflow and activity task respond operations. This is a leading indicator for alerts #1a and #1b — if the server keeps returning RESOURCE_EXHAUSTED on these operations long enough, the task will eventually time out and the server will reschedule it, causing workflow execution delays. Check the **Resource Exhausted with Cause** panel (§6 — Throttling and Limits) and the **Persistence Latencies** panel (§3) filtered to `UpdateWorkflowExecution` — high persistence latency is a common root cause of cascading throttling on respond operations. Consider increasing namespace RPS limits via dynamic config (`frontend.namespaceRPS`).

**PromQL (Java Micrometer):**
```promql
sum by (namespace, status_code, operation) (
  rate(temporal_request_failure_total{status_code="RESOURCE_EXHAUSTED", operation=~"RespondWorkflowTaskCompleted|RespondWorkflowTaskFailed|RespondActivityTaskCompleted|RespondActivityTaskFailed"}[5m])
) > 0
```

---

### Alert 2c — Request Failure: RESOURCE_EXHAUSTED on Long-Poll Operations

| Field | Value |
|---|---|
| **Severity** | ⚠️ Warning |
| **`for`** | 5m |
| **Metric** | `temporal_long_request_failure` |
| **Operations** | `PollWorkflowTaskQueue`, `PollActivityTaskQueue`, `GetWorkflowExecutionHistory` (long-poll / `waitNewEvent=true`) |
| **Status code** | `RESOURCE_EXHAUSTED` |

**Description:** Uses `temporal_long_request_failure`, not `temporal_request_failure`. Poll operations and the long-poll variant of `GetWorkflowExecutionHistory` go through the SDK's long-poll interceptor. RESOURCE_EXHAUSTED on these operations is most commonly caused by the namespace concurrent poller limit being exceeded (`ConcurrentLimit` cause), but can also be caused by the namespace RPS limit (`RpsLimit` cause) if the poll rate itself exceeds `frontend.namespaceRPS`. Throttling on `PollWorkflowTaskQueue` or `PollActivityTaskQueue` causes workers to back off and poll less frequently, increasing schedule-to-start latency. Throttling on `GetWorkflowExecutionHistory` (long-poll) delays clients sync-waiting on a workflow result. Check the **Resource Exhausted with Cause** panel (§6) filtering by `ConcurrentLimit`. Cross-check schedule-to-start latency alerts (#18, #19) and All Pollers Disconnected alert (#15). Also check the **Total Concurrent Pollers** and **Max Concurrent Pollers per Frontend Pod** panels (§15 — Pollers). If the limit is consistently hit, increase `frontend.namespaceCount` (per instance) or `frontend.globalNamespaceCount` (global) via dynamic config.

**PromQL (Java Micrometer):**
```promql
sum by (namespace, status_code, operation) (
  rate(temporal_long_request_failure_total{status_code="RESOURCE_EXHAUSTED", operation=~"PollWorkflowTaskQueue|PollActivityTaskQueue|GetWorkflowExecutionHistory"}[5m])
) > 0
```

---

### Alert 2d — Request Failure: RESOURCE_EXHAUSTED on Other Operations

| Field | Value |
|---|---|
| **Severity** | ⚠️ Warning |
| **`for`** | 5m |
| **Metric** | `temporal_request_failure` |
| **Operations** | All operations not covered by alerts 2a, 2b, 2c, or #3 |
| **Status code** | `RESOURCE_EXHAUSTED` |

**Description:** Server is throttling SDK operations that are automatically retried. Sustained throttling can delay workflow end-to-end execution time and cause business impact even if errors are not immediately visible to callers. Notably, throttling on `GetWorkflowExecutionHistory` (used during workflow task replay to fetch history pages) will directly increase `temporal_workflow_task_replay_latency` and `temporal_workflow_task_execution_latency` — if sustained long enough this can cascade into WFT timeouts. Cross-check `temporal_workflow_task_replay_latency` on the dashboard if this alert fires alongside alert #1a. Check the **Resource Exhausted with Cause** panel (§6) to understand the scope of throttling. Consider increasing namespace RPS limits via dynamic config (`frontend.namespaceRPS`).

**PromQL (Java Micrometer):**
```promql
sum by (namespace, status_code, operation) (
  rate(temporal_request_failure_total{status_code="RESOURCE_EXHAUSTED", operation!~"StartWorkflowExecution|SignalWithStartWorkflowExecution|ExecuteMultiOperation|UpdateWorkflowExecution|SignalWorkflowExecution|RespondWorkflowTaskCompleted|RespondWorkflowTaskFailed|RespondActivityTaskCompleted|RespondActivityTaskFailed|GetSystemInfo|DescribeNamespace"}[5m])
) > 0
```

---

### Alert 3 — Worker Startup Throttled

| Field | Value |
|---|---|
| **Severity** | 🔴 Critical |
| **`for`** | 1m |
| **Metric** | `temporal_request_failure` |
| **Operations** | `GetSystemInfo`, `DescribeNamespace` |
| **Status code** | `RESOURCE_EXHAUSTED` |

**Description:** Server is throttling `GetSystemInfo` and `DescribeNamespace`, which are called by all SDK workers at startup. The server treats these as low priority under load and throttles them before other operations. The SDK retries for ~60 seconds — if throttling is sustained for that long the worker will fail to start and will not process any tasks. Check the **Resource Exhausted with Cause** panel (§6 — Throttling and Limits) and the **Total RPS** panel (§1 — Cluster Throughput).

**PromQL (Java Micrometer):**
```promql
sum by (namespace, status_code, operation) (
  rate(temporal_request_failure_total{status_code="RESOURCE_EXHAUSTED", operation=~"GetSystemInfo|DescribeNamespace"}[5m])
) > 0
```

---

### Alert 4 — Request Failure: Authentication / Authorization Failure

| Field | Value |
|---|---|
| **Severity** | 🔴 Critical |
| **`for`** | 5m |
| **Metric** | `temporal_request_failure` |
| **Operations** | Any |
| **Status codes** | `PERMISSION_DENIED`, `UNAUTHENTICATED` |

**Description:** Server is returning authentication or authorization errors for SDK operations. Root causes: missing, expired, or revoked credentials, or misconfigured auth policy. Can also occur transiently during server restarts — adjust `for` as needed for your environment. Check your credential configuration and auth policy. On the server side, cross-check the **Unauthorized Requests** panel (§19 — Authorization) to confirm the server is seeing the denied requests, and the **Authorization System Failures** panel — a non-zero value there indicates the auth plugin itself is failing (crash, misconfiguration, or network failure to an external auth service), which is a more urgent signal than a simple credential problem.

**PromQL (Java Micrometer):**
```promql
sum by (namespace, status_code, operation) (
  rate(temporal_request_failure_total{status_code=~"PERMISSION_DENIED|UNAUTHENTICATED"}[5m])
) > 0
```

---

### Alert 5 — Request Failure: UNIMPLEMENTED

| Field | Value |
|---|---|
| **Severity** | 🔴 Critical |
| **`for`** | 2m |
| **Metric** | `temporal_request_failure` |
| **Operations** | Any |
| **Status code** | `UNIMPLEMENTED` |

**Description:** Server is returning UNIMPLEMENTED for an SDK operation. By the time a worker reaches this point it has already successfully called `GetSystemInfo` and `DescribeNamespace`, so this is unlikely to be a simple version mismatch. Check your server deployment — wrong binary deployed, corrupted deployment, or a service routing issue sending requests to the wrong handler are the most likely causes. Also check if you are running a very old SDK version that is calling an API that has been removed or significantly changed on the server. Cross-check the **Service Panics** panel (§5 — Service Requests and Errors) — a wrong binary or corrupted deployment often produces panics alongside UNIMPLEMENTED responses.

**PromQL (Java Micrometer):**
```promql
sum by (namespace, status_code, operation) (
  rate(temporal_request_failure_total{status_code="UNIMPLEMENTED"}[5m])
) > 0
```

---

### Alert 6 — Request Failure: INTERNAL

| Field | Value |
|---|---|
| **Severity** | 🔴 Critical |
| **`for`** | 2m |
| **Metric** | `temporal_request_failure` |
| **Operations** | Any |
| **Status code** | `INTERNAL` |

**Description:** Server is returning INTERNAL errors for SDK operations. Short bursts are expected during server restarts. If sustained, this indicates infrastructure-level issues. Check the **Service Errors by Namespace** panel (§5), the **Service Panics** panel (§5 — any panic is a critical signal), the **Persistence Errors by Namespace and Operation** panel (§3), and the **Persistence Availability** panel (§3) — sustained persistence errors are a common root cause of INTERNAL responses. Also check server health and deployment.

**PromQL (Java Micrometer):**
```promql
sum by (namespace, status_code, operation) (
  rate(temporal_request_failure_total{status_code="INTERNAL"}[5m])
) > 0
```

---

### Alert 7 — Request Failure: NOT_FOUND on Signal / Update / Query

| Field | Value |
|---|---|
| **Severity** | ⚠️ Warning |
| **`for`** | 5m |
| **Metric** | `temporal_request_failure` |
| **Operations** | `SignalWorkflowExecution`, `UpdateWorkflowExecution`, `QueryWorkflow` |
| **Status code** | `NOT_FOUND` |

**Description:** Application is consistently receiving NOT_FOUND on signal, update, or query operations. For `SignalWorkflowExecution` and `UpdateWorkflowExecution`: the target workflow does not exist or has already completed. One possible cause is RESOURCE_EXHAUSTED throttling on these operations (cross-check alert #2a) — if the call was delayed long enough by throttling, the target execution may have completed by the time the request reached the server. For `QueryWorkflow`: the workflow does not exist, was closed and removed by namespace retention policy, or the client has set `QueryRejectCondition` to reject queries on non-running executions (e.g. `QUERY_REJECT_CONDITION_NOT_OPEN`) — in which case this is expected behavior and this alert may not be applicable. Check your workflow client's query reject condition configuration (`QueryRejectCondition` in Go SDK, `queryRejectCondition` in Java SDK).

**PromQL (Java Micrometer):**
```promql
sum by (namespace, status_code, operation) (
  rate(temporal_request_failure_total{status_code="NOT_FOUND", operation=~"SignalWorkflowExecution|UpdateWorkflowExecution|QueryWorkflow"}[5m])
) > 0
```

---

### Alert 8 — Request Failure: ALREADY_EXISTS on StartWorkflowExecution

| Field | Value |
|---|---|
| **Severity** | ⚠️ Warning |
| **`for`** | 10m |
| **Metric** | `temporal_request_failure` |
| **Operation** | `StartWorkflowExecution` |
| **Status code** | `ALREADY_EXISTS` |

**Description:** The server rejected the start because the workflow ID is already in use. The behavior depends on `WorkflowIdReusePolicy` and `WorkflowIdConflictPolicy`: with the default `ALLOW_DUPLICATE` reuse policy, ALREADY_EXISTS only fires when a workflow with this ID is currently running; with `REJECT_DUPLICATE`, it also fires for completed workflows within the retention period regardless of outcome; with `ALLOW_DUPLICATE_FAILED_ONLY`, it fires when the prior run completed successfully. Root causes: application is retrying starts for a workflow that is still running (workflows running longer than expected), a bug in workflow ID generation producing duplicate IDs, misconfigured `WorkflowIdReusePolicy` for the use case, or RESOURCE_EXHAUSTED throttling on `StartWorkflowExecution` (cross-check alert #2a) — if a start was delayed long enough by throttling, another client may have already reserved that workflow ID.

**PromQL (Java Micrometer):**
```promql
sum by (namespace, status_code, operation) (
  rate(temporal_request_failure_total{status_code="ALREADY_EXISTS", operation="StartWorkflowExecution"}[5m])
) > 0
```

---

### Alert 9 — Request Failure: FAILED_PRECONDITION on QueryWorkflow

| Field | Value |
|---|---|
| **Severity** | ⚠️ Warning |
| **`for`** | 2m |
| **Metric** | `temporal_request_failure` |
| **Operation** | `QueryWorkflow` |
| **Status code** | `FAILED_PRECONDITION` |

**Description:** No workers polling for the task queue — query cannot be routed to a worker. Cross-check `temporal_num_pollers` and the All Pollers Disconnected alert (#15).

**PromQL (Java Micrometer):**
```promql
sum by (namespace, status_code, operation) (
  rate(temporal_request_failure_total{status_code="FAILED_PRECONDITION", operation="QueryWorkflow"}[5m])
) > 0
```

---

### Alert 10 — Long-Poll Request Failure

| Field | Value |
|---|---|
| **Severity** | ⚠️ Warning |
| **`for`** | 5m |
| **Metric** | `temporal_long_request_failure` |
| **Operations** | Any |
| **Status codes** | Any non-OK |

**Description:** Long-poll failures are uncommon and warrant investigation — typically indicates server-side issues. Check the **Service Errors by Namespace** panel (§5) and the **Persistence Errors by Namespace and Operation** panel (§3). For `GetWorkflowExecutionHistory` long-poll failures in particular, also check frontend service CPU utilization via your infrastructure observability stack.

**PromQL (Java Micrometer):**
```promql
sum by (namespace, status_code, operation) (
  rate(temporal_long_request_failure_total[5m])
) > 0
```

---

### Alert 11 — GetWorkflowExecutionHistory Long-Poll Rate Sustained High

| Field | Value |
|---|---|
| **Severity** | ⚠️ Warning |
| **`for`** | 5m |
| **Metric** | `temporal_long_request` |
| **Operation** | `GetWorkflowExecutionHistory` |
| **Threshold** | rate > 30/s |

**Description:** Workers are polling event history at a sustained high rate. This can mean workers are replaying large amounts of event history, which will elevate `temporal_workflow_task_replay_latency` and `temporal_workflow_task_execution_latency`. Can also indicate the worker cache is too small and experiencing high eviction rates — cross-check alert #16 (Sticky Cache Forced Evictions High). At high rates this puts pressure on the database. Also check frontend CPU on the server side.

**PromQL (Java Micrometer):**
```promql
sum by (namespace, operation) (
  rate(temporal_long_request_total{operation="GetWorkflowExecutionHistory"}[5m])
) > 30
```

---

### Alert 12 — RespondWorkflowTaskCompleted Rate Dropped to Zero

| Field | Value |
|---|---|
| **Severity** | 🔴 Critical |
| **`for`** | 5m |
| **Metric** | `temporal_request` |
| **Operation** | `RespondWorkflowTaskCompleted` |
| **Scope** | Configure per `task_queue` |

**Description:** Workflow task completions have dropped to zero for this task queue — workflow progress has fully halted. First check if `PollWorkflowTaskQueue` rate is also zero, which would indicate workers are down entirely; if so, check auth failure alerts (#4) and worker startup throttling alerts (#3). If polling is active but completions are zero, check whether `temporal_workflow_task_execution_failed` is elevated — workers may be failing tasks instead of completing them. Also check WFT schedule-to-start latency alerts (#18, #19) for a timeout cascade. Check RESOURCE_EXHAUSTED on critical operations alert (#2a) — if Start, Signal, or Update operations are throttled, no new workflow tasks may be getting created. Check overall cluster health on the server dashboard: **Service Errors by Namespace** (§5), **Persistence Availability** (§3), and **Resource Exhausted with Cause** (§6). Scoped per `task_queue` — configure which task queues to monitor.

**PromQL (Java Micrometer):**
```promql
(
  sum by (namespace, task_queue) (
    rate(temporal_request_total{operation="RespondWorkflowTaskCompleted", task_queue="<configured>"}[5m])
  ) or vector(0)
) == 0
```

---

### Alert 13 — RespondActivityTaskCompleted Rate Dropped to Zero

| Field | Value |
|---|---|
| **Severity** | 🔴 Critical |
| **`for`** | 5m |
| **Metric** | `temporal_request` |
| **Operation** | `RespondActivityTaskCompleted` |
| **Scope** | Configure per `task_queue` |

**Description:** Activity task completions have dropped to zero for this task queue. First check if `PollActivityTaskQueue` rate is also zero — if polling has stopped, workers are down entirely; check auth failure alerts (#4) and worker startup throttling alerts (#3). If polling is active, check RESOURCE_EXHAUSTED on poll operations alert (#2c) — if the server is throttling `PollActivityTaskQueue`, workers back off and pick up tasks less frequently, which can suppress completions even when workers are alive. Check RESOURCE_EXHAUSTED on respond operations alert (#2b) — if the server is throttling `RespondActivityTaskCompleted`, the SDK only increments `temporal_request` after a successful response, so sustained throttling will suppress this counter directly. Also check for server-side connectivity problems: INTERNAL errors (#6) and UNIMPLEMENTED (#5) both indicate the server is not processing SDK operations normally. Finally, check for any workflow task-level problems that would prevent new activities from being scheduled — RESOURCE_EXHAUSTED on critical user-facing operations (#2a), RESOURCE_EXHAUSTED on `RespondWorkflowTaskCompleted` (#2b), NonDeterminismError (#21) or GrpcMessageTooLarge (#22) WFT failures, WFT schedule-to-start latency alerts (#18, #19), and alert #1a (NOT_FOUND on WFT respond operations). Check overall cluster health: **Service Errors by Namespace** (§5), **Persistence Availability** (§3), and **Resource Exhausted with Cause** (§6). Scoped per `task_queue` — configure which task queues to monitor.

**PromQL (Java Micrometer):**
```promql
(
  sum by (namespace, task_queue) (
    rate(temporal_request_total{operation="RespondActivityTaskCompleted", task_queue="<configured>"}[5m])
  ) or vector(0)
) == 0
```

---

## Section 2 — Worker Lifecycle

> **SDK dashboard panels:** Worker Task Slots Available (panel 34), Number of Active Pollers (panel 36)
> **Metrics:** `temporal_worker_task_slots_available`, `temporal_num_pollers`

---

### Alert 14 — Worker Task Slots Exhausted

| Field | Value |
|---|---|
| **Severity** | 🔴 Critical |
| **`for`** | 2m |
| **Metric** | `temporal_worker_task_slots_available` (gauge) |
| **Dimensions** | `namespace`, `worker_type`, `task_queue` |

**Description:** Uses `or vector(0)` so the alert evaluates even when the metric is absent — resource-based tuner deployments never emit `temporal_worker_task_slots_available`, so disable this alert if you use the resource-based tuner. Impact and remediation differ by `worker_type`:

**`WorkflowWorker`** — All workflow task execution slots are occupied — existing tasks are taking too long to complete and not releasing their slots. Check worker pod/container CPU — high CPU slows WFT execution directly and keeps slots occupied longer. Check `temporal_workflow_task_execution_latency` — sustained high values confirm something is holding slots: slow deterministic code, heavy computation inside workflow functions, or a blocked executor. For Python SDK workers, verify that no `async def` workflow code is blocking the event loop. Cross-check RESOURCE_EXHAUSTED on `RespondWorkflowTaskCompleted` (#2b) — if the server is throttling WFT completions, slots are not released until the respond call succeeds. Also check the GrpcMessageTooLarge alert (#22). Immediate action: scale up workflow worker pods or increase `maxConcurrentWorkflowTaskExecutionSize`.

**`ActivityWorker`** — All activity execution slots are occupied — existing activities are taking too long to complete and not releasing their slots. Check `temporal_activity_execution_latency` for the affected task queue and activity type — sustained high values confirm activities are holding slots far longer than expected. Check worker pod/container CPU. Cross-check RESOURCE_EXHAUSTED on `RespondActivityTaskCompleted` (#2b). Immediate action: scale up activity worker pods or increase `maxConcurrentActivityExecutionSize`.

**`LocalActivityWorker`** — All local activity execution slots are occupied — the worker cannot schedule new local activities. Local activities run within the WFT execution loop, so blocked slots hold up the entire WFT. The SDK responds by sending repeated WFT heartbeats — cross-check alert #25 (WFT Heartbeat Rate Elevated). If heartbeating continues past `history.workflowTaskHeartbeatTimeout` (default 30 minutes), the server will time out the heartbeat WFT and reschedule it — local activities will be re-executed from scratch with potential duplicate side effects if not idempotent. The root cause is almost always local activity code blocking and not returning. If they are calling downstream services, check whether those services are throttling — increasing slot counts in that scenario would only increase downstream pressure.

**PromQL (Java Micrometer):**
```promql
sum by (namespace, worker_type, task_queue) (
  temporal_worker_task_slots_available or vector(0)
) == 0
```

---

### Alert 15 — All Pollers Disconnected

| Field | Value |
|---|---|
| **Severity** | 🔴 Critical |
| **`for`** | 5m |
| **Metric** | `temporal_num_pollers` (gauge) |
| **Dimensions** | `namespace`, `worker_type`, `task_queue` |

**Description:** No active pollers for this `worker_type` / task queue — workers have stopped polling entirely. Tasks are accumulating on the server with no one to process them. The 5m `for` avoids false positives during normal pod restarts and rolling deploys.

**First check alert #14 (Worker Task Slots Exhausted)** — if all task slots are occupied, the SDK blocks before issuing the next poll (both Go and Java SDKs block on slot acquisition before incrementing `temporal_num_pollers`), so the gauge drops to zero as a secondary effect. In that case #14 is the root cause and #15 is a symptom. If slots are not exhausted, check auth failure alert (#4) — expired or revoked credentials are a common cause of pollers disconnecting. On the server side, cross-check the **Unauthorized Requests** and **Authorization System Failures** panels (§19 — Authorization). Also check worker startup throttling alert (#3) — if `GetSystemInfo` or `DescribeNamespace` are being throttled on reconnect, workers will fail to re-establish polling. Check INTERNAL errors alert (#6) and UNIMPLEMENTED alert (#5) — both indicate the server is not accepting SDK operations normally. Finally check your worker pod/container health via your infrastructure observability stack.

**PromQL (Java Micrometer):**
```promql
sum by (namespace, worker_type, task_queue) (
  temporal_num_pollers
) == 0
```

---

## Section 3 — Worker Cache

> **SDK dashboard panels:** Sticky Cache Hit (52), Sticky Cache Miss (53), Sticky Cache Size (54), Sticky Cache Forced Evictions (55)
> **Metrics:** `temporal_sticky_cache_hit`, `temporal_sticky_cache_miss`, `temporal_sticky_cache_size`, `temporal_sticky_cache_total_forced_eviction`

---

### Alert 16 — Sticky Cache Forced Evictions High

| Field | Value |
|---|---|
| **Severity** | ⚠️ Warning |
| **`for`** | 5m |
| **Metric** | `temporal_sticky_cache_total_forced_eviction` |
| **Threshold** | rate > 30/s |
| **Dimensions** | `namespace` |

**Description:** Some forced evictions are normal — the sticky cache is bounded by resources and cannot hold every concurrent workflow execution indefinitely. A sustained high rate is the signal. If worker pod memory utilization is low when this alert fires, the cache is likely undersized by count. Check `temporal_sticky_cache_size` (gauge) against `maxWorkflowCacheSize` — if the gauge is at or near the limit, increase `maxWorkflowCacheSize`. However, the cache can fill its memory budget before hitting the numeric slot limit if individual workflow executions carry unusually large in-memory state. Always check **per-pod** memory utilization via your infrastructure observability stack — do not average across all workers polling the same task queue, since sticky assignment is per-pod and one pod may hold far more or heavier executions than others. Long-running executions also contribute to sustained cache pressure. Also check alert #21 (WFT Execution Failed — NonDeterminismError) — NDE failures and WFT timeouts cause the server to reschedule tasks to the normal task queue, forcing cache evictions on the receiving worker. Each eviction means the next WFT for that execution requires a full cold replay. Cross-check the **WFT Execution Latency** panel — a rising eviction rate directly drives higher WFT execution latency.

**PromQL (Java Micrometer):**
```promql
sum by (namespace) (
  rate(temporal_sticky_cache_total_forced_eviction_total[5m])
) > 30
```

---

## Section 4 — Workflow Executions

> **SDK dashboard panel:** Workflow Failed (panel 14)
> **Metric:** `temporal_workflow_failed`

---

### Alert 17 — Workflow Execution Failed Rate Elevated

| Field | Value |
|---|---|
| **Severity** | ⚠️ Warning |
| **`for`** | 3m |
| **Metric** | `temporal_workflow_failed` |
| **Threshold** | rate > 20/s |
| **Dimensions** | `namespace`, `workflow_type`, `task_queue` |

**Description:** `temporal_workflow_failed` is incremented when a workflow execution closes with `FAILED` status — this includes unhandled errors, explicit `ApplicationError`/`ApplicationFailure` calls, and unhandled panics that bubble up to the workflow root. `ApplicationError`/`ApplicationFailure` instances marked with category `BENIGN` are excluded and do not increment this counter. By default, WFT-level failures such as NonDeterminismError go to `temporal_workflow_task_execution_failed` (covered by alerts #21–#23) and do not increment this counter. However, if you configure `WorkflowPanicPolicy=FailWorkflow` (Go SDK) or add an exception type to `setFailWorkflowExceptionTypes` (Java SDK), those failures will be converted to workflow-level failures and **will** increment `temporal_workflow_failed` — unless the failure is marked benign.

A sustained elevated rate means workflow executions are terminating with failure at an abnormal rate. This metric carries `workflow_type` and `task_queue` labels but does **not** carry `workflow_id` — at scale you cannot identify the specific failing executions from this metric alone. Check your worker logs, which should include the workflow ID associated with each failure. Then check the `failure` field in `WorkflowExecutionFailed` history events for the error message and type.

If your application intentionally fails workflow executions by design — saga compensations, explicit failure propagation, error signaling patterns — mark those `ApplicationError`/`ApplicationFailure` instances with category `BENIGN`. Both Go and Java SDKs skip the counter for benign failures, making this alert exclusively track unexpected failures without needing per-`workflow_type` threshold tuning.

**PromQL (Java Micrometer):**
```promql
sum by (namespace, workflow_type, task_queue) (
  rate(temporal_workflow_failed_total[5m])
) > 20
```

---

## Section 5 — Schedule To Start Latencies

> **SDK dashboard panels:** WFT Schedule To Start (panel 22), Activity Schedule To Start (panel 23)
> **Metrics:** `temporal_workflow_task_schedule_to_start_latency`, `temporal_activity_schedule_to_start_latency`

---

### Alert 18 — WFT Schedule-To-Start Latency High

| Field | Value |
|---|---|
| **Severity** | ⚠️ Warning |
| **`for`** | 5m |
| **Metric** | `temporal_workflow_task_schedule_to_start_latency` |
| **Threshold** | p99 > 1s |
| **Dimensions** | `namespace`, `task_queue` |

**Description:** p99 workflow task schedule-to-start latency has been above 1 second for 5 minutes on this namespace and task queue. Check worker pod/container CPU. Check poller counts via `temporal_num_pollers` — if pollers are dropping, cross-check alert #15 (All Pollers Disconnected). Check if poll operations are being rate limited — cross-check alert #2c (RESOURCE_EXHAUSTED on Long-Poll Operations) for `PollWorkflowTaskQueue`. Check for a large workflow task backlog on the server side — see the **Approximate Task Backlog** panel in the server dashboard (Matching service section).

**PromQL (Java Micrometer):**
```promql
histogram_quantile(0.99,
  sum by (namespace, task_queue, le) (
    rate(temporal_workflow_task_schedule_to_start_latency_seconds_bucket[5m])
  )
) > 1
```

---

### Alert 19 — WFT Schedule-To-Start Latency Critical

| Field | Value |
|---|---|
| **Severity** | 🔴 Critical |
| **`for`** | 5m |
| **Metric** | `temporal_workflow_task_schedule_to_start_latency` |
| **Threshold** | p99 > 5s |
| **Dimensions** | `namespace`, `task_queue` |

**Description:** p99 workflow task schedule-to-start latency has been above 5 seconds for 5 minutes on this namespace and task queue. This is severely impacting workflow execution end-to-end latencies. Check worker pod/container CPU. Check poller counts via `temporal_num_pollers` — if pollers are dropping, cross-check alert #15 (All Pollers Disconnected). Also check the **Total Concurrent Pollers** panel (§15 — Pollers) in the server dashboard — if the concurrent poller count for workflow tasks has dropped, your worker pool may have shrunk and horizontal scaling is needed. Check if poll operations are being rate limited — cross-check alert #2c for `PollWorkflowTaskQueue`. Check for a large workflow task backlog — see the **Approximate Task Backlog** panel in the server dashboard (Matching service section).

**PromQL (Java Micrometer):**
```promql
histogram_quantile(0.99,
  sum by (namespace, task_queue, le) (
    rate(temporal_workflow_task_schedule_to_start_latency_seconds_bucket[5m])
  )
) > 5
```

---

### Alert 19b — WFT Schedule-To-Start Latency Severely Elevated

| Field | Value |
|---|---|
| **Severity** | 🔴 Critical |
| **`for`** | 5m |
| **Metric** | `temporal_workflow_task_schedule_to_start_latency` |
| **Threshold** | p99 > 1800s (30m) |
| **Dimensions** | `namespace`, `task_queue` |

**Description:** p99 workflow task schedule-to-start latency has been above 30 minutes for 5 minutes on this namespace and task queue. This is severe degradation — workflow tasks are not being picked up and at large scale this can cause a very large task backlog that puts significant pressure on the server database. Check worker pod/container CPU. Check poller counts via `temporal_num_pollers` — if pollers are dropping, cross-check alert #15. Also check the **Total Concurrent Pollers** panel (§15 — Pollers) — if the concurrent poller count has dropped, your worker pool may have shrunk and horizontal scaling is needed. Check if poll operations are being rate limited — cross-check alert #2c for `PollWorkflowTaskQueue`. Check for a large workflow task backlog — see the **Approximate Task Backlog** panel in the server dashboard (Matching service section).

**PromQL (Java Micrometer):**
```promql
histogram_quantile(0.99,
  sum by (namespace, task_queue, le) (
    rate(temporal_workflow_task_schedule_to_start_latency_seconds_bucket[5m])
  )
) > 1800
```

---

### Alert 20 — Activity Schedule-To-Start Latency High

| Field | Value |
|---|---|
| **Severity** | ⚠️ Warning |
| **`for`** | 5m |
| **Metric** | `temporal_activity_schedule_to_start_latency` |
| **Threshold** | p99 > 10s |
| **Dimensions** | `namespace`, `activity_type`, `task_queue` |

**Description:** p99 activity task schedule-to-start latency has been above 10 seconds for 5 minutes on this namespace and task queue. Check worker pod/container CPU. Check poller counts via `temporal_num_pollers` — if pollers are dropping, cross-check alert #15. Check if poll operations are being rate limited — cross-check alert #2c for `PollActivityTaskQueue`. Check for a large activity task backlog — see the **Approximate Task Backlog** panel in the server dashboard (Matching service section). Also check activity failure rates — cross-check alert #26 (Activity Execution Failed Rate Elevated), as high churn of unexpected activity failures causes additional retry tasks to be created and can contribute to backlog growth. Note: some activity types are expected to have high schedule-to-start by design — tune the threshold or disable per `activity_type` if this is intentional.

**PromQL (Java Micrometer):**
```promql
histogram_quantile(0.99,
  sum by (namespace, activity_type, task_queue, le) (
    rate(temporal_activity_schedule_to_start_latency_seconds_bucket[5m])
  )
) > 10
```

---

### Alert 20b — Activity Schedule-To-Start Latency Severely Elevated

| Field | Value |
|---|---|
| **Severity** | 🔴 Critical |
| **`for`** | 5m |
| **Metric** | `temporal_activity_schedule_to_start_latency` |
| **Threshold** | p99 > 1800s (30m) |
| **Dimensions** | `namespace`, `activity_type`, `task_queue` |

**Description:** p99 activity task schedule-to-start latency has been above 30 minutes for 5 minutes on this namespace and task queue. This is severe degradation — activity tasks are not being picked up and at large scale this can cause a very large task backlog that puts significant pressure on the server database. Check worker pod/container CPU. Check poller counts via `temporal_num_pollers` — if pollers are dropping, cross-check alert #15. Also check the **Total Concurrent Pollers** panel (§15 — Pollers) — if the concurrent poller count for activity tasks has dropped, your worker pool may have shrunk and horizontal scaling is needed. Check if poll operations are being rate limited — cross-check alert #2c for `PollActivityTaskQueue`. Check the **Approximate Task Backlog** panel in the server dashboard. Also check activity failure rates — cross-check alert #26.

**PromQL (Java Micrometer):**
```promql
histogram_quantile(0.99,
  sum by (namespace, activity_type, task_queue, le) (
    rate(temporal_activity_schedule_to_start_latency_seconds_bucket[5m])
  )
) > 1800
```

---

## Section 6 — Workflow Task Info

> **SDK dashboard panels:** WFT Execution Failed (panel 47), WFT Heartbeat (panel 49)
> **Metrics:** `temporal_workflow_task_execution_failed`, `temporal_workflow_task_heartbeat`
> Note: `temporal_workflow_task_no_completion` is defined in Go SDK constants but never incremented — alert #24 has been removed.

---

### Alert 21 — WFT Execution Failed: NonDeterminismError

| Field | Value |
|---|---|
| **Severity** | 🔴 Critical |
| **`for`** | 1m |
| **Metric** | `temporal_workflow_task_execution_failed` |
| **Label** | `failure_reason="NonDeterminismError"` |
| **Dimensions** | `namespace`, `workflow_type`, `task_queue` |

**Description:** A workflow's replay produced a different command sequence than what is recorded in history. Workflow executions affected by this error are not progressing — the server continuously retries the workflow task, putting additional pressure on workflow workers. Affected executions remain in running status by default, prolonging their end-to-end execution time indefinitely. This metric does not carry `workflow_id` — check worker logs which will log the NDE with the associated workflow ID. In the Temporal UI, affected executions can also be found by querying the `TemporalReportedProblems` search attribute, which the server sets on executions experiencing repeated workflow task failures. NDE is always a code bug. Common causes include: adding, removing, or reordering workflow commands without a versioning guard; changing activity or timer parameters in existing workflow code; or deploying incompatible worker code against in-flight executions. Once the root cause is identified and a fix is deployed, affected executions will resume on their next workflow task retry automatically.

**PromQL (Java Micrometer):**
```promql
sum by (namespace, workflow_type, task_queue, failure_reason) (
  rate(temporal_workflow_task_execution_failed_total{failure_reason="NonDeterminismError"}[5m])
) > 0
```

---

### Alert 22 — WFT Execution Failed: GrpcMessageTooLarge

| Field | Value |
|---|---|
| **Severity** | 🔴 Critical |
| **`for`** | 1m |
| **Metric** | `temporal_workflow_task_execution_failed` |
| **Label** | `failure_reason="GrpcMessageTooLarge"` |
| **Dimensions** | `namespace`, `workflow_type`, `task_queue` |

**Description:** The WFT response payload exceeded the gRPC message size limit — this can be rejected by the gRPC library on the SDK side, a proxy or load balancer in the path (e.g. Envoy), or the gRPC library on the server side on receive. Because the Temporal server history service never saw the original `RespondWorkflowTaskCompleted` request, the SDK explicitly sends a follow-up `RespondWorkflowTaskFailed` with cause `WORKFLOW_TASK_FAILED_CAUSE_GRPC_MESSAGE_TOO_LARGE` to clean up the execution. The server then terminates the execution in response — ending it with `TERMINATED` status with no retry. Also check the **Workflow Terminate** panel in the server dashboard — it tracks the rate of terminated executions and will show a spike when this is happening at scale. This metric does not carry `workflow_id` — check worker logs for the affected workflow IDs. Root causes: oversized activity or workflow inputs/outputs, excessive signal or update accumulation in history, or too many commands batched in a single WFT response. Investigate which part of the WFT response is oversized — the fix depends entirely on the root cause. Terminated executions must be restarted manually if needed.

**PromQL (Java Micrometer):**
```promql
sum by (namespace, workflow_type, task_queue, failure_reason) (
  rate(temporal_workflow_task_execution_failed_total{failure_reason="GrpcMessageTooLarge"}[5m])
) > 0
```

---

### Alert 23 — WFT Execution Failed: WorkflowError

| Field | Value |
|---|---|
| **Severity** | ⚠️ Warning |
| **`for`** | 2m |
| **Metric** | `temporal_workflow_task_execution_failed` |
| **Label** | `failure_reason="WorkflowError"` |
| **Threshold** | rate > 20/s |
| **Dimensions** | `namespace`, `workflow_type`, `task_queue` |

**Description:** Workflow task failed due to an error in workflow code — this can be an intermittent failure, a panic, or a non-Temporal exception thrown inside the workflow function. The server will retry the workflow task, which may result in the same failure repeatedly until the issue is resolved. Check worker logs — this metric does not carry `workflow_id`, so worker logs are needed to identify the affected executions. Investigate and fix the root cause, then redeploy workers.

**PromQL (Java Micrometer):**
```promql
sum by (namespace, workflow_type, task_queue, failure_reason) (
  rate(temporal_workflow_task_execution_failed_total{failure_reason="WorkflowError"}[5m])
) > 20
```

---

### ~~Alert 24 — WFT No Completion Rate Elevated~~ (Removed)

`temporal_workflow_task_no_completion` is defined in Go SDK constants but never incremented anywhere in the SDK source. Do not create an alert for this metric.

---

### Alert 25 — WFT Heartbeat Rate Elevated

| Field | Value |
|---|---|
| **Severity** | ⚠️ Warning |
| **`for`** | 5m |
| **Metric** | `temporal_workflow_task_heartbeat` |
| **Threshold** | rate > 20/s |
| **Dimensions** | `namespace`, `workflow_type`, `task_queue` |

**Description:** The SDK is sending workflow task heartbeats to keep the WFT alive while local activities are still running. This means local activities are taking longer than expected — either a single execution is running long or local activities are retrying and the retry chain is accumulating. If heartbeating continues past the server's WFT heartbeat timeout (`history.workflowTaskHeartbeatTimeout`, default 30 minutes), the server will time out the WFT and reschedule it on the normal task queue — at which point local activities will be re-executed from scratch on the retried task, with potential duplicate side effects if they are not idempotent. Cross-check alert #14 (Worker Task Slots Exhausted) for `worker_type=LocalActivityWorker` — if local activity slots are exhausted, that is driving the heartbeating. Also cross-check alerts #29 and #30 (Local Activity Execution Latency and Total Execution Latency Exceeds WFT Heartbeat Timeout) to confirm whether individual attempts or the full retry chain is approaching the 30 minute limit.

**PromQL (Java Micrometer):**
```promql
sum by (namespace, workflow_type, task_queue) (
  rate(temporal_workflow_task_heartbeat_total[5m])
) > 20
```

---

## Section 7 — Activity Task Info

> **SDK dashboard panels:** Activity Execution Failed (panel 63), Unregistered Activity Invocation (panel 65)
> **Metrics:** `temporal_activity_execution_failed`, `temporal_unregistered_activity_invocation`

---

### Alert 26 — Activity Execution Failed Rate Elevated

| Field | Value |
|---|---|
| **Severity** | ⚠️ Warning |
| **`for`** | 5m |
| **Metric** | `temporal_activity_execution_failed` |
| **Threshold** | rate > 20/s |
| **Dimensions** | `namespace`, `activity_type`, `task_queue` |

**Description:** Activities are explicitly failing — not timing out but returning failures — at a sustained rate. If this is not expected, investigate activity code for the affected `activity_type`. A high failure rate drives a burst of retry tasks being scheduled, putting additional pressure on activity workers and on the server database if workers cannot keep up with task dispatch and tasks need to be persisted. Cross-check alert #20 (Activity Schedule-To-Start Latency High) — a growing backlog from retry bursts will show up there first. If your use case intentionally fails activities by design — polling patterns, saga compensations, flow control via exceptions — set `ApplicationFailure` category to `BENIGN` on those intentional failures. All SDKs support this and will suppress this metric for those failures, making the alert exclusively track unexpected failures.

**PromQL (Java Micrometer):**
```promql
sum by (namespace, activity_type, task_queue) (
  rate(temporal_activity_execution_failed_total[5m])
) > 20
```

---

### Alert 27 — Unregistered Activity Invocation

| Field | Value |
|---|---|
| **Severity** | 🔴 Critical |
| **`for`** | 1m |
| **Metric** | `temporal_unregistered_activity_invocation` |
| **Dimensions** | `namespace`, `activity_type`, `task_queue` |

**Description:** The server dispatched an activity to this worker that the worker does not have registered. Always indicates a code or deployment issue — wrong worker binary, missing activity registration, or routing misconfiguration. Tasks will fail indefinitely until resolved. Note: brief spikes can occur during rolling worker restarts when old worker versions briefly receive tasks for newly registered activity types — if this is expected in your deployment process, consider increasing `for` to 5m or disabling the alert during planned deploys.

**PromQL (Java Micrometer):**
```promql
sum by (namespace, activity_type, task_queue) (
  rate(temporal_unregistered_activity_invocation_total[5m])
) > 0
```

---

## Section 8 — Local Activity Info

> **SDK dashboard panels:** Local Activity Execution Failed (panel 72), Local Activity Execution Latency (panel 74), Local Activity Total Execution Latency (panel 76)
> **Metrics:** `temporal_local_activity_execution_failed`, `temporal_local_activity_execution_latency`, `temporal_local_activity_total_execution_latency`

---

### Alert 28 — Local Activity Execution Failed Rate Elevated

| Field | Value |
|---|---|
| **Severity** | ⚠️ Warning |
| **`for`** | 5m |
| **Metric** | `temporal_local_activity_execution_failed` |
| **Threshold** | rate > 20/s |
| **Dimensions** | `namespace`, `activity_type`, `task_queue` |

**Description:** Local activities are explicitly failing — not timing out but returning failures — at a sustained rate. If this is not expected, investigate local activity code for the affected `activity_type`. High failure rates drive retry churn which can extend the total local activity execution time — if the cumulative retry chain runs past the server's WFT heartbeat timeout (`history.workflowTaskHeartbeatTimeout`, default 30 minutes), the server will time out the WFT and reschedule it, causing local activities to be re-executed from scratch with potential duplicate side effects if not idempotent. Cross-check alert #25 (WFT Heartbeat Rate Elevated) and alert #30 (Local Activity Total Execution Latency Exceeds WFT Heartbeat Timeout) to confirm if this is happening. Also cross-check alert #14 (Worker Task Slots Exhausted) for `worker_type=LocalActivityWorker` — sustained retry churn can consume all available local activity slots. If your use case intentionally fails local activities by design, set `ApplicationFailure` category to `BENIGN` on those intentional failures to suppress this metric.

**PromQL (Java Micrometer):**
```promql
sum by (namespace, activity_type, task_queue) (
  rate(temporal_local_activity_execution_failed_total[5m])
) > 20
```

---

### Alert 29 — Local Activity Execution Latency Exceeds WFT Heartbeat Timeout

| Field | Value |
|---|---|
| **Severity** | 🔴 Critical |
| **`for`** | 5m |
| **Metric** | `temporal_local_activity_execution_latency` |
| **Threshold** | p99 > 1800s (30m) |
| **Dimensions** | `namespace`, `activity_type`, `task_queue` |

**Description:** A single local activity attempt is running longer than the server's WFT heartbeat timeout (default 30 minutes, `history.workflowTaskHeartbeatTimeout`). This is critical: the server will time out the heartbeat WFT and reschedule it on the normal task queue — the local activity will be re-executed from scratch, any pending signals, updates, or other workflow events will be delayed until the retried WFT completes, and overall workflow execution end-to-end latency will increase significantly, causing potentially high business impact. If the local activity is not idempotent, re-execution can also cause duplicate side effects. Cross-check alert #25 (WFT Heartbeat Rate Elevated) to confirm heartbeating is already happening.

Local activities cannot heartbeat and are designed for short, fast operations. A local activity running for 30 minutes is a design issue — consider converting it to a regular activity with heartbeating, which is the correct primitive for long-running work. If the local activity must remain a local activity, always set a `scheduleToCloseTimeout` less than 30 minutes so it fails with a timeout error before the server times out the WFT heartbeat, allowing the workflow to handle the failure gracefully rather than having the entire WFT re-executed from scratch.

**PromQL (Java Micrometer):**
```promql
histogram_quantile(0.99,
  sum by (namespace, activity_type, task_queue, le) (
    rate(temporal_local_activity_execution_latency_seconds_bucket[5m])
  )
) > 1800
```

---

### Alert 30 — Local Activity Total Execution Latency Exceeds WFT Heartbeat Timeout

| Field | Value |
|---|---|
| **Severity** | 🔴 Critical |
| **`for`** | 5m |
| **Metric** | `temporal_local_activity_total_execution_latency` |
| **Threshold** | p99 > 1800s (30m) |
| **Dimensions** | `namespace`, `activity_type`, `task_queue` |

**Description:** Cumulative retry time across all local activity attempts has exceeded the server's WFT heartbeat timeout (default 30 minutes, `history.workflowTaskHeartbeatTimeout`). Same impact as alert #29 — the server will time out the heartbeat WFT and reschedule it — but in this case individual attempts may be short; it is the full retry chain that has accumulated past 30 minutes. The local activity will be re-executed from scratch, any pending signals, updates, or other workflow events will be delayed, and overall workflow execution end-to-end latency will increase significantly, causing potentially high business impact. If the local activity is not idempotent, re-execution can also cause duplicate side effects. Cross-check alert #25 (WFT Heartbeat Rate Elevated) and alert #28 (Local Activity Execution Failed Rate Elevated) — high failure rates driving retry churn are the most common cause of this alert firing without #29 also firing. Fix the underlying failure causing retries, reduce max retry attempts, or increase backoff. If the work genuinely requires many retries over a long period, convert to a regular activity with heartbeating.

**PromQL (Java Micrometer):**
```promql
histogram_quantile(0.99,
  sum by (namespace, activity_type, task_queue, le) (
    rate(temporal_local_activity_total_execution_latency_seconds_bucket[5m])
  )
) > 1800
```

---

## Section 9 — Request Latency

> **SDK dashboard panel:** Request Latency (panel 3)
> **Metric:** `temporal_request_latency`
> **Tags:** `namespace`, `operation`

---

### Alert 31 — Request Latency High on Critical User-Facing Operations

| Field | Value |
|---|---|
| **Severity** | 🔴 Critical |
| **`for`** | 5m |
| **Metric** | `temporal_request_latency` |
| **Operations** | `StartWorkflowExecution`, `SignalWithStartWorkflowExecution`, `ExecuteMultiOperation`, `SignalWorkflowExecution` |
| **Threshold** | p99 > 2s |

**Description:** p99 latency for critical user-facing SDK operations has been above 2 seconds for 5 minutes on this namespace. These operations are synchronous from the caller's perspective — elevated latency here means your application code is blocked waiting on the server response, directly impacting your users and any workflows that depend on these calls completing. The SDK retries on transient errors but does not hide latency — every retry adds to the total wait.

Check the **Frontend Service Latency** panel (§4 — Service Latencies) filtered to the affected operations — if server-side latency is elevated, that is the root cause. Check the [**Persistence Latencies** panel (§3 — Persistence Requests, Latencies and Errors)](../../dashboards/server/temporal-server-readme.md#3-persistence-requests-latencies-and-errors) filtered to `CreateWorkflowExecution` and `UpdateWorkflowExecution` — high persistence latency on these operations is the most common driver of elevated frontend latency for starts and signals. If persistence looks healthy, check frontend pod CPU utilization. Also cross-check RESOURCE_EXHAUSTED on critical operations alert (#2a) — if the server is throttling these operations, the SDK retries add to the total observed latency on this metric.

Note: `UpdateWithStartWorkflowExecution` maps to the gRPC operation name `ExecuteMultiOperation` in SDK metrics.

**PromQL (Java Micrometer):**
```promql
histogram_quantile(0.99,
  sum by (namespace, operation, le) (
    rate(temporal_request_latency_seconds_bucket{operation=~"StartWorkflowExecution|SignalWithStartWorkflowExecution|ExecuteMultiOperation|SignalWorkflowExecution"}[5m])
  )
) > 2
```

---

## Section 10 — Execution Latencies

> **SDK dashboard panels:** WFT Execution Latency (panel 24), WFT Replay Latency (panel 26), Workflow End-to-End Latency (panel 16), Sticky Cache Size (panel 54)
> **Metrics:** `temporal_workflow_task_execution_latency`, `temporal_workflow_task_replay_latency`, `temporal_workflow_endtoend_latency`, `temporal_sticky_cache_size`

---

### Alert 32 — WFT Execution Latency Critical

| Field | Value |
|---|---|
| **Severity** | 🔴 Critical |
| **`for`** | 5m |
| **Metric** | `temporal_workflow_task_execution_latency` |
| **Threshold** | p99 > 10s |
| **Dimensions** | `namespace`, `task_queue` |

**Description:** p99 workflow task execution latency has been above 10 seconds for 5 minutes on this namespace and task queue. The default WFT timeout is 10 seconds (`history.defaultWorkflowTaskTimeout`) — at this latency level the server is actively timing out workflow tasks, writing `WorkflowTaskTimedOut` events to history and rescheduling them on the normal task queue. Each timeout forces a sticky cache eviction on the worker that held the execution — cross-check alert #16 (Sticky Cache Forced Evictions High). If you are running local activities, a WFT timeout will cause local activities to be re-executed from scratch on the retried task — local activity results are not checkpointed between WFT heartbeats, so re-execution means they run again; if they are not idempotent this can cause duplicate side effects and real business impact.

Check worker pod CPU and memory — high CPU is the most common cause of slow WFT execution. Check `temporal_workflow_task_replay_latency` — if replay latency is high, the time is being spent re-executing history rather than running new commands; cross-check alert #35 (WFT Replay Latency High) and alert #16 (Sticky Cache Forced Evictions High). If replay latency is normal, the time is in new command execution — check for blocking calls or heavy computation inside workflow functions. For Python SDK workers, verify that no `async def` workflow code is blocking the event loop. Cross-check RESOURCE_EXHAUSTED on respond operations alert (#2b) — if the server is throttling `RespondWorkflowTaskCompleted`, WFT execution latency inflates because the SDK holds the slot until the respond call succeeds.

**PromQL (Java Micrometer):**
```promql
histogram_quantile(0.99,
  sum by (namespace, task_queue, le) (
    rate(temporal_workflow_task_execution_latency_seconds_bucket[5m])
  )
) > 10
```

---

### Alert 33 — Workflow End-to-End Latency High

| Field | Value |
|---|---|
| **Severity** | ⚠️ Warning |
| **`for`** | 5m |
| **Metric** | `temporal_workflow_endtoend_latency` |
| **Threshold** | user-defined — see below |
| **Dimensions** | `namespace`, `workflow_type`, `task_queue` |
| **Essential set** | No — documentation/reference only |

**Description:** p99 workflow end-to-end latency has exceeded your defined threshold on this namespace and task queue. This metric measures the total time from when the workflow execution was started to when it completed — it is entirely workload-dependent and no generic threshold applies. You must set the threshold based on your SLO for the affected `workflow_type`.

**Important:** `temporal_workflow_endtoend_latency` is only emitted when a workflow execution completes. It is not a real-time metric — by the time this alert fires, the slow executions have already finished. For running executions you will not see this signal until they close, which means this alert can be a very late indication of a latency problem. If you need real-time detection of long-running executions, use a server-side visibility query instead — for example `ExecutionStatus = "Running" AND StartTime < now() - <your threshold>` — and alert on the count returned.

A sustained breach here is the composite signal that something upstream is wrong. Cross-check in order: WFT schedule-to-start latency alerts (#18, #19) — tasks sitting in the queue before a worker picks them up; WFT execution latency alert (#32) — workers taking too long to complete tasks; activity schedule-to-start latency alerts (#20, #20b) — activity tasks sitting in the queue; activity execution failed rate alert (#26) — high failure and retry churn extending total activity duration; sticky cache forced eviction alert (#16) — high eviction rate forcing cold replays on every WFT. Also check RESOURCE_EXHAUSTED on critical user-facing operations alert (#2a) and respond operations alert (#2b) — throttling on start, signal, and respond operations all add directly to end-to-end latency.

> **Threshold placeholder:** Replace `<YOUR_SLO_SECONDS>` with your p99 SLO in seconds before deploying.

**PromQL (Java Micrometer):**
```promql
histogram_quantile(0.99,
  sum by (namespace, workflow_type, task_queue, le) (
    rate(temporal_workflow_endtoend_latency_seconds_bucket[5m])
  )
) > <YOUR_SLO_SECONDS>
```

---

### Alert 34 — Sticky Cache Disabled

| Field | Value |
|---|---|
| **Severity** | ⚠️ Warning |
| **`for`** | 5m |
| **Metric** | `temporal_sticky_cache_size` (gauge) |
| **Condition** | `== 0` |
| **Dimensions** | `namespace`, `task_queue` |

**Description:** The sticky cache size gauge has been zero for 5 minutes — the worker's sticky execution cache is disabled. Every workflow task for every execution on this worker requires a full cold replay from the beginning of history: all history pages fetched from the server, all commands re-executed from scratch. At any meaningful scale this causes significant pressure on the frontend and persistence layers — cross-check the [**Persistence Latencies** panel (§3 — Persistence Requests, Latencies and Errors)](../../dashboards/server/temporal-server-readme.md#3-persistence-requests-latencies-and-errors) and the `GetWorkflowExecutionHistory` long-poll rate alert (#11). WFT execution latency will also be elevated — cross-check alert #32 (WFT Execution Latency Critical).

This is almost always a misconfiguration. In the Go SDK, setting `maxWorkflowCacheSize` to `0` disables the cache entirely. In the Java SDK, `setStickyQueueScheduleToStartTimeout` of zero or `setMaxConcurrentWorkflowTaskExecutionSize` of zero can have the same effect. Check your worker options and restore a non-zero cache size — the default is 10,000 in the Go SDK.

**PromQL (Java Micrometer):**
```promql
sum by (namespace, task_queue) (
  temporal_sticky_cache_size
) == 0
```

---

### Alert 35 — WFT Replay Latency High

| Field | Value |
|---|---|
| **Severity** | ⚠️ Warning |
| **`for`** | 5m |
| **Metric** | `temporal_workflow_task_replay_latency` |
| **Threshold** | p99 > 5s |
| **Dimensions** | `namespace`, `task_queue` |

**Description:** p99 workflow task replay latency has been above 5 seconds for 5 minutes on this namespace and task queue. Replay latency is the time the SDK spends re-executing recorded history commands before reaching the point where new commands can be scheduled. High replay latency directly inflates `temporal_workflow_task_execution_latency` — cross-check alert #32 (WFT Execution Latency Critical). If replay latency is the dominant component of WFT execution latency, the root cause is almost always one of: a very large history (many events accumulated without ContinueAsNew), slow data converter execution during replay, or high sticky cache eviction rate forcing frequent cold replays — cross-check alert #16 (Sticky Cache Forced Evictions High) and alert #34 (Sticky Cache Disabled).

Also cross-check RESOURCE_EXHAUSTED on other operations alert (#2d) filtered to `GetWorkflowExecutionHistory` — if the server is throttling history page fetches during replay, that directly adds to replay latency. Check the [**Persistence Latencies** panel (§3 — Persistence Requests, Latencies and Errors)](../../dashboards/server/temporal-server-readme.md#3-persistence-requests-latencies-and-errors) filtered to `GetWorkflowExecution` — high persistence latency on history reads is a common contributor. Also check the `GetWorkflowExecutionHistory` long-poll rate alert (#11) — a sustained high rate confirms workers are fetching large amounts of history repeatedly. If history size is the root cause, consider using ContinueAsNew to truncate history at logical checkpoints.

**PromQL (Java Micrometer):**
```promql
histogram_quantile(0.99,
  sum by (namespace, task_queue, le) (
    rate(temporal_workflow_task_replay_latency_seconds_bucket[5m])
  )
) > 5
```

---

## Section 11 — Workflow Lifecycle Rates

> **SDK dashboard panel:** ContinueAsNew (panel 18)
> **Metric:** `temporal_workflow_continue_as_new`
> **Tags:** `namespace`, `workflow_type`, `task_queue`

---

### Alert 36 — ContinueAsNew Rate Elevated

| Field | Value |
|---|---|
| **Severity** | ⚠️ Warning |
| **`for`** | 5m |
| **Metric** | `temporal_workflow_continue_as_new` |
| **Threshold** | rate > 100/s |
| **Dimensions** | `namespace`, `workflow_type`, `task_queue` |

**Description:** ContinueAsNew rate has been above 100 per second for 5 minutes on this namespace and task queue. Each ContinueAsNew closes the current workflow run and immediately starts a new one — from the server's perspective this is a workflow completion followed by a new workflow start, meaning every ContinueAsNew generates a `CreateWorkflowExecution` persistence write alongside the `UpdateWorkflowExecution` write for the closing run. At high rates this puts significant pressure on server persistence and can contribute to elevated latency across the cluster. Check the [**Persistence Latencies** panel (§3 — Persistence Requests, Latencies and Errors)](../../dashboards/server/temporal-server-readme.md#3-persistence-requests-latencies-and-errors) filtered to `CreateWorkflowExecution` and `UpdateWorkflowExecution` — if latency is elevated there, the ContinueAsNew burst is a likely contributor.

**Critically, check whether the high rate is concentrated on the same workflow ID.** All runs for the same workflow ID land on the same history shard — a single workflow ID ContinueAsNew-ing at a very high rate creates a hot shard on the history host, which can cause elevated shard lock latency and persistence contention localized to that shard. Check your worker logs to identify which workflow IDs are generating the ContinueAsNew calls. On the server side, cross-check the **Shard Lock Latency** panel (§2 — History Service) — a hot shard from a single high-CAN workflow ID will show up there as elevated lock latency on a specific pod.

A high ContinueAsNew rate is not always a problem — some use cases are designed around frequent history truncation. Investigate whether the rate is expected for the affected `workflow_type`. If it is not expected, check whether executions are hitting history size limits earlier than anticipated and being forced to ContinueAsNew more frequently than designed — reduce event accumulation per run or increase the ContinueAsNew interval. If the rate is expected but causing persistence pressure, consider throttling the workflow start rate or scaling persistence.

**PromQL (Java Micrometer):**
```promql
sum by (namespace, workflow_type, task_queue) (
  rate(temporal_workflow_continue_as_new_total[5m])
) > 100
```

---

## Alerts Not Planned

The following metrics were considered and explicitly excluded from the alert inventory:

| Metric | Reason |
|---|---|
| `temporal_long_request_latency` | Long-poll timeouts are expected behavior (server returns empty after ~60s). Alerting would produce constant false positives. |
| `temporal_activity_execution_latency` | User-defined activity duration — long-running use cases routinely run for hours. Not alertable generically. |
| `temporal_activity_succeed_endtoend_latency` | Same as above — includes all retries, entirely user-defined. |
| `temporal_local_activity_execution_latency` | Covered by WFT heartbeat alert (#25) — if local activities consistently run long, the heartbeat rate climbs. Direct latency alert would require per-workflow tuning. |
| `temporal_worker_task_slots_used` | Covered by slots available alert (#14) — if available drops to zero, used is at max. Redundant. |
| `temporal_worker_start` / `temporal_poller_start` | Event counters with no meaningful rate threshold — spikes are expected on rolling restarts. |
| `temporal_workflow_cancelled` | Expected workflow lifecycle event. Not alertable generically. |
| Nexus metrics | Deferred until Nexus support is better understood. |

---

## Related Resources

- [Essential Alert Set — README](./README.md)
- [Alert Planning Document](./planning.md) — design decisions and working notes
- [Temporal SDK Go Dashboard README](../../dashboards/sdk/temporal-sdk-go-readme.md)
- [Temporal SDK Java Micrometer Dashboard README](../../dashboards/sdk/temporal-sdk-java-micrometer-readme.md)
- [Temporal Server Dashboard README](../../dashboards/server/temporal-server-readme.md)
- [Temporal Server Alerts](../server/README.md)
- [Temporal Dynamic Config Reference](../../../dynamic_config/README.md)
