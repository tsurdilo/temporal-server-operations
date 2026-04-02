# Understanding Temporal SDK Request Failure Metrics
## `temporal_request_failure` & `temporal_long_request_failure`

---

## Table of Contents

1. [Why These Two Metrics](#why-these-two-metrics)
2. [How the SDK Distinguishes the Two Metrics](#how-the-sdk-distinguishes-the-two-metrics)
3. [Server-Side Throttling: Rate Limit Buckets and Priority](#server-side-throttling-rate-limit-buckets-and-priority)
    - [Execution Bucket](#execution-bucket)
    - [Visibility Bucket](#visibility-bucket)
    - [NamespaceReplicationInducing Bucket](#namespacereplicationinducing-bucket)
    - [Priority Within Each Bucket](#priority-within-each-bucket)
4. [`temporal_long_request_failure` — Long-Poll Operations](#temporal_long_request_failure--long-poll-operations)
5. [`temporal_request_failure` — Standard Operations](#temporal_request_failure--standard-operations)
    - [Execution Bucket](#execution-bucket-1)
        - [Priority 0 — System / Informational](#priority-0--system--informational-operator-equivalent)
        - [Priority 1 — External Event APIs + Task Progress](#priority-1--external-event-apis--task-progress-completions--heartbeats)
        - [Priority 2 — State Change APIs](#priority-2--state-change-apis)
        - [Priority 3 — Status Querying + Failure/Cancel Reporting](#priority-3--status-querying--failurecancel-reporting)
        - [Priority 4 — Poll APIs and Other Low Priority](#priority-4--poll-apis-and-other-low-priority)
        - [Priority 5 — Lowest Priority](#priority-5--lowest-priority)
    - [Visibility Bucket](#visibility-bucket-1)
    - [NamespaceReplicationInducing Bucket](#namespacereplicationinducing-bucket-1)
6. [Diagnostic Tables](#diagnostic-tables)
    - [`temporal_long_request_failure`](#temporal_long_request_failure-diagnostic-table)
    - [`temporal_request_failure`](#temporal_request_failure-diagnostic-table)
10. [From SDK Metric to Server Dashboard](#from-sdk-metric-to-server-dashboard)
8. [Metric Tag Format](#metric-tag-format)
9. [Sources](#sources)

---

## Why These Two Metrics

When debugging Temporal applications, `temporal_request_failure` and `temporal_long_request_failure` are by far the most useful SDK metrics to start with. They are not narrowly scoped to one part of the system — they surface issues on both the SDK worker side and the server side, making them a natural first stop during an incident and a reliable starting point for knowing which cluster-side metrics to dig into next.

`temporal_long_request_failure` is particularly important for alerting on server-side rate limiting, since it covers the long-poll operations like `PollWorkflowTaskQueue` and `PollActivityTaskQueue` which are the lowest priority in the execution rate limit bucket and therefore the first to be shed when the server is under pressure.

`temporal_request_failure` casts a wider net — it covers all standard client and worker operations, making it indispensable for understanding things like workflow and activity timeouts, task completion failures, worker crashes, and visibility issues. Many of the errors that surface here are the first observable signal of a deeper problem, whether that is a misconfigured timeout, a worker that is falling behind, or a server that is struggling under load.

It is also important to configure SDK metrics on both your SDK worker processes and your SDK clients — the code that starts, signals, and otherwise interacts with workflow executions. This matters because `temporal_request_failure` then covers not only worker-specific operations like `RespondWorkflowTaskCompleted`, but also critical client operations like `StartWorkflowExecution`. A throttled or failing `StartWorkflowExecution` is something you absolutely want to alert on, and you will only see it in this metric if your client process has metrics configured.

---

## How the SDK Distinguishes the Two Metrics

All Temporal SDKs wrap every outgoing RPC in a gRPC unary interceptor that records metrics before and after each call. The routing decision is simple:

- If the call's context has `LongPollContextKey = true` → **`temporal_long_request_failure`**
- All other calls → **`temporal_request_failure`**

The `operation` tag is extracted directly from the gRPC method path (everything after the last `/`), so it matches the proto `rpc` name exactly. Both metrics carry:

| Tag | Description |
|-----|-------------|
| `operation` | The exact proto RPC name (see tables below) |
| `status_code` | gRPC status code of the failed non-OK response |
| `namespace` | Namespace extracted from the request |

---

## Server-Side Throttling: Rate Limit Buckets and Priority

When a `ResourceExhausted` error surfaces in `temporal_request_failure` or `temporal_long_request_failure`, it means the server's rate limiter rejected the call. Understanding *which* rate limiter, *at what priority*, and *which dynamic config knob controls it* is essential for diagnosis and tuning.

The Temporal frontend uses **three independent rate limit buckets**, each with its own RPS quota and internal priority queue. They are configured independently and do not share capacity.

---

### Execution Bucket

Covers most workflow, worker, and admin APIs.

| Scope | Dynamic Config Key | Default | Notes |
|-------|--------------------|---------|-------|
| Per-instance, per-namespace | `frontend.namespaceRPS` | `0` (unlimited) | Primary per-host throttle. Set this to cap a single frontend instance's RPS for a namespace |
| Global, per-namespace | `frontend.globalNamespaceRPS` | `0` (disabled) | Cluster-wide cap distributed evenly across frontend instances. Takes precedence over per-instance when set |
| Burst ratio (per-instance) | `frontend.namespaceBurstRatio` | `2.0` | Burst capacity as a multiplier of the effective RPS. Must be ≥ 1 |
| Concurrent long-running (per-instance) | `frontend.namespaceCount` | `0` (unlimited) | Max concurrent long-running requests (polls, queries, updates) per namespace per API per instance |
| Concurrent long-running (global) | `frontend.globalNamespaceCount` | `0` (disabled) | Same as above but cluster-wide |

> **How per-instance and global interact:** The effective per-instance limit is `min(perInstanceRPS, globalRPS / numFrontendInstances)`. If `globalNamespaceRPS` is 0, only `namespaceRPS` applies. If both are 0, the bucket is unlimited.

---

### Visibility Bucket

Covers all APIs backed by the visibility store (Elasticsearch / SQL). Completely separate quota from the Execution bucket — exhausting visibility RPS does **not** affect workflow start/signal RPS and vice versa.

| Scope | Dynamic Config Key | Default | Notes |
|-------|--------------------|---------|-------|
| Per-instance, per-namespace | `frontend.namespaceRPS.visibility` | `10` | **Very low default** — easy to hit in production with frequent list/count calls |
| Global, per-namespace | `frontend.globalNamespaceRPS.visibility` | `0` (disabled) | Cluster-wide visibility cap distributed across instances |
| Burst ratio (per-instance) | `frontend.namespaceBurstRatio.visibility` | `1.0` | Burst ratio for visibility. Defaults to 1 (no burst headroom). Must be ≥ 1 |

> ⚠️ The `frontend.namespaceRPS.visibility` default of **10 RPS** is intentionally conservative. In production workloads with dashboards, monitoring, or frequent `ListWorkflowExecutions` calls, this is frequently the first limit hit. It is also why `DescribeTaskQueue` (which queries visibility for reachability data) can trigger `ResourceExhausted` even when execution RPS looks healthy.

---

### NamespaceReplicationInducing Bucket

Covers APIs that write to the namespace replication queue — a critical, unpartitioned resource also used for failover messages between clusters. Isolated to prevent flooding that queue.

| Scope | Dynamic Config Key | Default | Notes |
|-------|--------------------|---------|-------|
| Global (cluster-wide) | `frontend.rps.namespaceReplicationInducingAPIs` | `20` | Hard cluster-wide cap; applies regardless of namespace |
| Per-instance, per-namespace | `frontend.namespaceRPS.namespaceReplicationInducingAPIs` | `0` (unlimited) | Per-host per-namespace cap; experimental |
| Global, per-namespace | `frontend.globalNamespaceRPS.namespaceReplicationInducingAPIs` | `0` (disabled) | Cluster-wide per-namespace cap; experimental |
| Burst ratio (per-instance) | `frontend.namespaceBurstRatio.namespaceReplicationInducingAPIs` | `10.0` | Burst multiplier for replication-inducing APIs |

> The global `frontend.rps.namespaceReplicationInducingAPIs` at **20 RPS** is a cluster-wide ceiling shared across all namespaces. Automation that bulk-registers namespaces or rapidly updates versioning rules can hit this. This was introduced in v1.21.0 when `UpdateWorkerBuildIdCompatibility`, `UpdateNamespace`, and `RegisterNamespace` were moved to their own queue.

---

### Priority Within Each Bucket

Within each bucket, every API has a **priority**. When the bucket is under load, **lower-priority calls are dropped first**. Requests from Web UI or `tctl` (`CallerType = Operator`) are always elevated to **Priority 0** regardless of the API, giving them precedence over all SDK traffic.

Priority scale: **0 = highest, 5 = lowest**

---

## `temporal_long_request_failure` — Long-Poll Operations

These are the **only** operations where the SDK sets `LongPollContextKey = true`. The server holds the connection open until a task is available or the poll timeout (~60s) is reached.

| Operation | Bucket | Priority | Notes |
|-----------|--------|----------|-------|
| `PollWorkflowTaskQueue` | Execution | **4** | Regular + sticky task queue polls |
| `PollActivityTaskQueue` | Execution | **4** | Activity task polls |
| `PollNexusTaskQueue` | Execution | **4** | Nexus task polls |
| `GetWorkflowExecutionHistory` (long-poll) | Execution | **5** | Only when `wait_new_event=true`. Tracked server-side as `PollWorkflowExecutionHistory`. Priority 5 so spikes don't starve Poll* APIs |

> **Note on `GetWorkflowExecutionHistory`:** The same RPC covers two cases. When `wait_new_event=false` (plain fetch), it falls under `temporal_request_failure` at **Priority 2**. When `wait_new_event=true` (long-poll), it falls under `temporal_long_request_failure` at **Priority 5**. The server internally routes these to different quota slots.

> **Note on Poll* blocking:** `PollWorkflowTaskQueue`, `PollActivityTaskQueue`, and `PollNexusTaskQueue` are in `PollTaskAPISet` — when `FrontendPollWaitForNamespaceRateLimitToken` is enabled, these will *block and wait* for a token rather than reject immediately, avoiding spurious `ResourceExhausted` errors on poll calls.

---

## `temporal_request_failure` — Standard Operations

### Execution Bucket

Most workflow and worker operations share the Execution RPS quota.

#### Priority 0 — System / Informational (Operator-equivalent)

These are exempt from most shedding. They are the APIs needed to bootstrap any client connection.

| Operation | Caller | Description |
|-----------|--------|-------------|
| `GetClusterInfo` | Client / Worker | Cluster metadata |
| `GetSystemInfo` | Client / Worker | Server capability/feature flags |
| `GetSearchAttributes` | Client | List search attribute keys (deprecated; prefer OperatorService) |
| `DescribeNamespace` | Client / Worker | Namespace info and config |
| `ListNamespaces` | Client | List all namespaces |
| `DeprecateNamespace` | Client | Deprecate a namespace |

#### Priority 1 — External Event APIs + Task Progress (Completions & Heartbeats)

High-priority because rejecting these would force retries, amplifying load.

| Operation | Caller | Description |
|-----------|--------|-------------|
| `StartWorkflowExecution` | Client | Start a new workflow |
| `SignalWorkflowExecution` | Client | Send a signal to a running workflow |
| `SignalWithStartWorkflowExecution` | Client | Signal-or-start a workflow |
| `UpdateWorkflowExecution` | Client | Send an update to a workflow (blocks until WFT completes) |
| `ExecuteMultiOperation` | Client | Atomically execute multiple operations |
| `CreateSchedule` | Client | Create a new schedule |
| `StartBatchOperation` | Client | Start a bulk workflow operation |
| `StartActivityExecution` | Client | Start a standalone activity |
| `RecordActivityTaskHeartbeat` | Worker | Heartbeat a running activity by task token |
| `RecordActivityTaskHeartbeatById` | Worker / App | Heartbeat a running activity by ID |
| `RespondActivityTaskCompleted` | Worker | Complete an activity by task token |
| `RespondActivityTaskCompletedById` | Worker / App | Complete an activity by ID (async completion) |
| `RespondWorkflowTaskCompleted` | Worker | Complete a workflow task, returning commands |
| `RespondQueryTaskCompleted` | Worker | Complete a query task |
| `RespondNexusTaskCompleted` | Worker | Complete a Nexus task |
| `DispatchNexusTaskByNamespaceAndTaskQueue` | Nexus | Dispatch a Nexus task via task queue |
| `DispatchNexusTaskByEndpoint` | Nexus | Dispatch a Nexus task via endpoint |
| `CompleteNexusOperation` | Nexus | Complete an async Nexus operation |

#### Priority 2 — State Change APIs

Mutations that change execution state. Also includes history fetch (high relative priority because it is required for replay).

| Operation | Caller | Description |
|-----------|--------|-------------|
| `RequestCancelWorkflowExecution` | Client | Graceful cancellation request |
| `TerminateWorkflowExecution` | Client | Forceful termination |
| `ResetWorkflowExecution` | Client | Reset to prior history event |
| `DeleteWorkflowExecution` | Client | Delete a closed workflow |
| `PauseWorkflowExecution` | Client | Pause a running workflow |
| `UnpauseWorkflowExecution` | Client | Unpause a workflow |
| `GetWorkflowExecutionHistory` | Client / Worker | Fetch history (non-long-poll). Higher priority because required for replay |
| `UpdateSchedule` | Client | Modify a schedule |
| `PatchSchedule` | Client | Trigger/pause/backfill a schedule |
| `DeleteSchedule` | Client | Delete a schedule |
| `StopBatchOperation` | Client | Stop a running batch operation |
| `PauseActivity` | Client | Pause a running activity |
| `UnpauseActivity` | Client | Unpause an activity |
| `ResetActivity` | Client | Reset an activity execution |
| `UpdateActivityOptions` | Client | Update options on a running activity |
| `RequestCancelActivityExecution` | Client | Request activity cancellation |
| `TerminateActivityExecution` | Client | Forcefully terminate an activity |
| `DeleteActivityExecution` | Client | Delete an activity execution record |
| `UpdateWorkflowExecutionOptions` | Client | Update workflow-level options |
| `SetWorkerDeploymentCurrentVersion` | Client | Promote a deployment version to current |
| `SetWorkerDeploymentRampingVersion` | Client | Configure a canary/ramping version |
| `SetWorkerDeploymentManager` | Client | Set deployment manager |
| `DeleteWorkerDeployment` | Client | Delete a deployment record |
| `DeleteWorkerDeploymentVersion` | Client | Delete a specific version record |
| `UpdateWorkerDeploymentVersionMetadata` | Client | Update deployment version metadata |
| `UpdateTaskQueueConfig` | Worker | Update task queue config |
| `CreateWorkflowRule` | Client | Create a workflow rule (experimental) |
| `DescribeWorkflowRule` | Client | Get a workflow rule (experimental) |
| `DeleteWorkflowRule` | Client | Delete a workflow rule (experimental) |
| `ListWorkflowRules` | Client | List workflow rules (experimental) |
| `TriggerWorkflowRule` | Client | Manually trigger a rule (experimental) |

#### Priority 3 — Status Querying + Failure/Cancel Reporting

Read-only describe APIs, plus task failure/cancel responses (lower priority because the task will be retried anyway).

| Operation | Caller | Description |
|-----------|--------|-------------|
| `DescribeWorkflowExecution` | Client | Current workflow state and metadata |
| `DescribeActivityExecution` | Client | Activity execution info (non-long-poll) |
| `DescribeTaskQueue` | Client / Worker | Task queue info — **counts against Visibility RPS** |
| `QueryWorkflow` | Client | Synchronously query a workflow (blocks until WFT completes) |
| `GetWorkerBuildIdCompatibility` | Worker / Client | Build ID compatibility info (deprecated versioning) |
| `GetWorkerVersioningRules` | Worker / Client | Versioning rules for a task queue |
| `ListTaskQueuePartitions` | Worker | Task queue partition list |
| `DescribeSchedule` | Client | Current schedule state |
| `ListScheduleMatchingTimes` | Client | Preview schedule fire times |
| `DescribeBatchOperation` | Client | Batch operation status |
| `DescribeWorkerDeployment` | Client | Worker deployment info |
| `DescribeWorkerDeploymentVersion` | Client | Deployment version info |
| `RespondActivityTaskCanceled` | Worker | Confirm activity cancellation by task token |
| `RespondActivityTaskCanceledById` | Worker / App | Confirm activity cancellation by ID |
| `RespondActivityTaskFailed` | Worker | Fail an activity by task token |
| `RespondActivityTaskFailedById` | Worker / App | Fail an activity by ID |
| `RespondWorkflowTaskFailed` | Worker | Report workflow task panic/non-determinism |
| `RespondNexusTaskFailed` | Worker | Fail a Nexus task |

#### Priority 4 — Poll APIs and Other Low Priority

| Operation | Caller | Description |
|-----------|--------|-------------|
| `PollWorkflowTaskQueue` | Worker | **(Also long-poll)** Workflow task poll |
| `PollActivityTaskQueue` | Worker | **(Also long-poll)** Activity task poll |
| `PollNexusTaskQueue` | Worker | **(Also long-poll)** Nexus task poll |
| `PollWorkflowExecutionUpdate` | Client | Poll for an in-progress workflow update result |
| `PollActivityExecution` | Client | Poll for activity execution state changes |
| `ResetStickyTaskQueue` | Worker | Clear sticky task queue for a workflow |
| `GetWorkflowExecutionHistoryReverse` | Client | Fetch history in reverse order |
| `RecordWorkerHeartbeat` | Worker | Worker process heartbeat |
| `FetchWorkerConfig` | Worker | Fetch worker configuration |
| `UpdateWorkerConfig` | Worker | Update worker configuration |
| `ShutdownWorker` | Worker | Signal a worker to shut down gracefully |

#### Priority 5 — Lowest Priority

| Operation | Caller | Description |
|-----------|--------|-------------|
| `GetWorkflowExecutionHistory` (long-poll variant) | Worker / Client | `wait_new_event=true` — server-side alias `PollWorkflowExecutionHistory`. Downprioritized vs Poll* |
| `DescribeActivityExecution` (long-poll variant) | Client | `long_poll_token` set — server-side alias `PollActivityExecutionDescription` |
| OpenAPI v2/v3 docs endpoints | — | Informational, not required for Temporal to function |

---

### Visibility Bucket

These APIs hit the visibility store (Elasticsearch / SQL) and share a **separate** Visibility RPS quota. All currently sit at **Priority 1** within that bucket.

| Operation | Caller | Description |
|-----------|--------|-------------|
| `ListWorkflowExecutions` | Client | Advanced visibility query |
| `ListOpenWorkflowExecutions` | Client | Basic visibility — open workflows |
| `ListClosedWorkflowExecutions` | Client | Basic visibility — closed workflows |
| `ListArchivedWorkflowExecutions` | Client | Archived workflow list |
| `ScanWorkflowExecutions` | Client | Large unordered scan |
| `CountWorkflowExecutions` | Client | Count workflows matching a query |
| `ListActivityExecutions` | Client | List activity executions |
| `CountActivityExecutions` | Client | Count activity executions |
| `DescribeTaskQueue` | Client / Worker | Task queue info — despite its name, hits visibility for reachability data |
| `GetWorkerTaskReachability` | Worker / Client | Determine build ID reachability via visibility |
| `ListSchedules` | Client | List schedules (backed by visibility) |
| `CountSchedules` | Client | Count schedules |
| `ListBatchOperations` | Client | List batch operations |
| `ListWorkerDeployments` | Client | List worker deployments |
| `ListWorkers` | Client | List registered workers |
| `DescribeWorker` | Client | Describe a registered worker |

---

### NamespaceReplicationInducing Bucket

These APIs write to the **namespace replication queue**, which is also used for critical failover messages. They have their own isolated RPS quota to prevent flooding that queue.

| Operation | Caller | Priority | Description |
|-----------|--------|----------|-------------|
| `RegisterNamespace` | Admin | **1** | Create a new namespace |
| `UpdateNamespace` | Admin | **1** | Modify namespace configuration |
| `UpdateWorkerBuildIdCompatibility` | Worker / Admin | **2** | Register build ID compatibility rules (deprecated versioning) |
| `UpdateWorkerVersioningRules` | Worker / Admin | **2** | Modify task queue versioning rules |

---

## Diagnostic Tables

These tables are the core troubleshooting reference. Each row maps an operation and a gRPC status code to what it means operationally — what triggered it, what to look for, and whether it warrants an alert.

An important distinction: the rate limit buckets and priority section above covers **one specific failure class** — `ResourceExhausted` from server-side throttling. But `temporal_request_failure` and `temporal_long_request_failure` capture **every non-OK gRPC response** an operation can return. That includes throttling, yes, but also application-level errors (`NotFound`, `AlreadyExists`, `FailedPrecondition`), bad requests (`InvalidArgument`), auth failures (`PermissionDenied`, `Unauthenticated`), infrastructure problems (`Unavailable`), timeouts (`DeadlineExceeded` — which can be a client-side context timeout, a server-side processing timeout, or a proxy cutting a long-lived connection), and unexpected server faults (`Internal`). Each of these tells a different story depending on which operation it came from — a `DeadlineExceeded` on `QueryWorkflow` points to workers not making progress, whereas a `DeadlineExceeded` on `PollWorkflowTaskQueue` is often just a proxy cutting the long-poll connection and can be completely normal. The tables below capture that full picture per operation.

---

### `temporal_long_request_failure` Diagnostic Table

| Operation | Status Code | What it means / What to look for |
|-----------|-------------|-----------------------------------|
| `PollWorkflowTaskQueue` | `ResourceExhausted` | Always throttling-related. On OSS this is typically an RPS limit (`frontend.namespaceRPS`) or the concurrent long-running request limit (`frontend.namespaceCount`) being exceeded — the latter meaning too many simultaneous long-poll requests are open. On Temporal Cloud it can additionally mean throttling on Actions Per Second (APS) or Operations Per Second (OPS) limits. To determine the exact cause, check your server-side cluster metrics and look at the `cause` tag — key values to look for are `RpsLimit`, `ConcurrentLimit`, and `SystemExhausted`. On Temporal Cloud also check for `ApsLimit`. Note that workflow and activity pollers share the same namespace quota, so high counts of both can jointly exhaust the limit. |
| `PollWorkflowTaskQueue` | `NotFound` | Pretty rare. Most commonly seen if a worker is connected to a namespace that gets deleted while the worker is still running. Can also occasionally appear transiently during server restarts. If seen persistently outside of those two scenarios, verify the namespace name in your worker configuration. |
| `PollWorkflowTaskQueue` | `Unavailable` | Typically related to server issues — for example the frontend being unreachable or other server-side errors such as "Not enough hosts to serve request". Can be intermittent during server scaling or restarts, in which case it is generally non-critical. If seen persistently, alert on this and check server health. |
| `PollWorkflowTaskQueue` | `DeadlineExceeded` | A timeout occurred — this can have several causes: (1) server-side DB timeouts, check persistence metrics on your cluster; (2) worker or client pod restart that canceled the in-flight poll, look for `ShutdownWorker`, `DescribeNamespace`, or `GetSystemInfo` operations around the same time as a first indicator of worker restarts; (3) a gRPC proxy with an aggressively low timeout cutting the long-poll connection. On polls specifically this is often benign and caused by proxies — context matters. |
| `PollWorkflowTaskQueue` | `Unauthenticated` | Missing or invalid auth token. Check mTLS certificates or API key configuration on the worker. |
| `PollWorkflowTaskQueue` | `Internal` | Typically unexpected server-side issues. Can be intermittent during things like server restarts or persistence DB issues. If seen at a higher rate or over a longer duration, check server health. |
| `PollWorkflowTaskQueue` | `Canceled` | Worker shut down and canceled the in-flight poll context. Expected during graceful shutdown. Can also be caused by a gRPC proxy canceling the connection. Elevated rates outside of deployments or proxy maintenance warrant investigation. |
| `PollActivityTaskQueue` | `ResourceExhausted` | Always throttling-related. On OSS this is typically an RPS limit (`frontend.namespaceRPS`) or the concurrent long-running request limit (`frontend.namespaceCount`) being exceeded — the latter meaning too many simultaneous long-poll requests are open. On Temporal Cloud it can additionally mean throttling on Actions Per Second (APS) or Operations Per Second (OPS) limits. To determine the exact cause, check your server-side cluster metrics and look at the `cause` tag — key values to look for are `RpsLimit`, `ConcurrentLimit`, and `SystemExhausted`. On Temporal Cloud also check for `ApsLimit`. Note that activity and workflow pollers share the same namespace quota, so high counts of both can jointly exhaust the limit. |
| `PollActivityTaskQueue` | `NotFound` | Namespace not found. Same causes as workflow poll. |
| `PollActivityTaskQueue` | `Unavailable` | Typically related to server issues — frontend unreachable or other server-side errors. Can be intermittent during server scaling or restarts, in which case it is generally non-critical. If seen persistently, alert on this and check server health. |
| `PollActivityTaskQueue` | `DeadlineExceeded` | A timeout occurred — this can have several causes: (1) server-side DB timeouts, check persistence metrics on your cluster; (2) worker or client pod restart, look for `ShutdownWorker`, `DescribeNamespace`, or `GetSystemInfo` operations around the same time; (3) a gRPC proxy with an aggressively low timeout cutting the long-poll. On polls specifically this is often benign and caused by proxies — context matters. |
| `PollActivityTaskQueue` | `Unauthenticated` | Invalid or missing credentials. |
| `PollActivityTaskQueue` | `Internal` | Typically unexpected server-side issues. Can be intermittent during things like server restarts or persistence DB issues. If seen at a higher rate or over a longer duration, check server health. |
| `PollActivityTaskQueue` | `Canceled` | Worker shutdown canceled the poll. Expected during graceful shutdown. Can also be caused by a gRPC proxy canceling the connection. Elevated rates outside of deployments or proxy maintenance warrant investigation. |
| `PollWorkflowExecutionUpdate` | `ResourceExhausted` | Execution bucket RPS exhausted. Update polling competes with other Priority 4 operations. |
| `PollWorkflowExecutionUpdate` | `NotFound` | Update ID does not exist on the specified workflow execution. Could indicate the update was never admitted, the workflow completed before the update was processed, or wrong workflow/update ID. |
| `PollWorkflowExecutionUpdate` | `DeadlineExceeded` | A timeout occurred while waiting for the update's workflow task to complete. Causes include: (1) server-side DB timeouts, check persistence metrics; (2) worker or client pod restart, look for `ShutdownWorker`, `DescribeNamespace`, or `GetSystemInfo` around the same time; (3) workers struggling — non-determinism issues or generic task failures will prevent WFTs from completing, which surfaces here; (4) gRPC proxy timeout. Check `workflow_task_schedule_to_start_latency` and worker health. |
| `PollWorkflowExecutionUpdate` | `Unavailable` | Typically related to server issues — frontend unreachable or other server-side errors. Can be intermittent during server scaling or restarts, in which case it is generally non-critical. If seen persistently, alert on this and check server health. |
| `PollWorkflowExecutionUpdate` | `Internal` | Typically unexpected server-side issues. Can be intermittent during things like server restarts or persistence DB issues. If seen at a higher rate or over a longer duration, check server health. |
| `PollWorkflowExecutionUpdate` | `Canceled` | Can have several causes: the client canceled the poll because its own context was canceled (e.g. the caller gave up waiting); the server canceled the request due to intermittent server-side issues; or a gRPC proxy canceled the connection. Check whether cancellations correlate with server restarts or proxy events. |
| `GetWorkflowExecutionHistory` (long-poll) | `ResourceExhausted` | Can have several causes: (1) server-side RPS throttling — execution bucket exhausted at Priority 5, the lowest priority and first to be shed under load; check `cause` tag on server metrics for `RpsLimit`, `ConcurrentLimit`, or `SystemExhausted`. On Temporal Cloud also check for `ApsLimit` and `OpsLimit`; (2) concurrent long-running request limit (`frontend.namespaceCount`) exceeded; (3) gRPC proxy response size limit — Temporal server can return up to 128 MB of history in a single response, and if your proxy is configured to accept less than that it will return `ResourceExhausted`. If this is the cause, increase the proxy's max receive message size to at least 128 MB. |
| `GetWorkflowExecutionHistory` (long-poll) | `DeadlineExceeded` | A timeout occurred while waiting for new history events. Causes include: (1) server-side DB timeouts, check persistence metrics; (2) worker or client pod restart, look for `ShutdownWorker`, `DescribeNamespace`, or `GetSystemInfo` around the same time; (3) gRPC proxy timeout cutting the long-poll. Semi-expected if the workflow is genuinely blocked for a long time, but consistently high rates warrant investigation. |
| `GetWorkflowExecutionHistory` (long-poll) | `Unavailable` | Typically related to server issues — frontend unreachable or other server-side errors. Can be intermittent during server scaling or restarts, in which case it is generally non-critical. If seen persistently, alert on this and check server health. |
| `GetWorkflowExecutionHistory` (long-poll) | `Internal` | Typically unexpected server-side issues. Can be intermittent during things like server restarts or persistence DB issues. If seen at a higher rate or over a longer duration, check server health. |
| `GetWorkflowExecutionHistory` (long-poll) | `Canceled` | Caller canceled the wait — expected when the watching client disconnects. Can also be caused by server-side DB timeouts that force the request to be abandoned; check persistence latency and error metrics as well as service error graphs in your server metrics. |

---

### `temporal_request_failure` Diagnostic Table

| Operation | Status Code | What it means / What to look for |
|-----------|-------------|-----------------------------------|
| `StartWorkflowExecution` | `AlreadyExists` | A workflow with this ID is already running. Expected if your application uses workflow ID deduplication intentionally. If unexpected, check for duplicate start calls or ID generation logic. |
| `StartWorkflowExecution` | `ResourceExhausted` | Execution bucket RPS exhausted at Priority 1. High start rate is overwhelming the namespace quota. Check `frontend.namespaceRPS` and `frontend.globalNamespaceRPS`. |
| `StartWorkflowExecution` | `InvalidArgument` | Malformed request — bad workflow type, missing task queue, invalid timeout values, or oversized input payload (check `BlobSizeLimitError`). |
| `StartWorkflowExecution` | `NotFound` | Namespace not found. |
| `StartWorkflowExecution` | `PermissionDenied` | Auth failure. |
| `StartWorkflowExecution` | `Unauthenticated` | Invalid credentials. |
| `StartWorkflowExecution` | `Unavailable` | Typically related to server issues — frontend unreachable or other server-side errors. Can be intermittent during server scaling or restarts, in which case it is generally non-critical. If seen persistently, alert on this and check server health. |
| `StartWorkflowExecution` | `Internal` | Typically unexpected server-side issues. Can be intermittent during things like server restarts or persistence DB issues. If seen at a higher rate or over a longer duration, check server health. |
| `SignalWorkflowExecution` | `NotFound` | Workflow not found — it completed, was terminated, or exceeded retention. Very common in systems that signal workflows without checking if they are still running. |
| `SignalWorkflowExecution` | `ResourceExhausted` | Execution bucket RPS exhausted at Priority 1. |
| `SignalWorkflowExecution` | `InvalidArgument` | Bad signal name, oversized signal payload, or too many pending signals (server enforces a limit on buffered signals per workflow). |
| `SignalWorkflowExecution` | `PermissionDenied` | Auth failure. |
| `SignalWorkflowExecution` | `Unauthenticated` | Invalid credentials. |
| `SignalWorkflowExecution` | `Unavailable` | Typically related to server issues — frontend unreachable or other server-side errors. Can be intermittent during server scaling or restarts, in which case it is generally non-critical. If seen persistently, alert on this and check server health. |
| `SignalWorkflowExecution` | `Internal` | Typically unexpected server-side issues. Can be intermittent during things like server restarts or persistence DB issues. If seen at a higher rate or over a longer duration, check server health. |
| `SignalWithStartWorkflowExecution` | `ResourceExhausted` | Execution bucket RPS exhausted at Priority 1. |
| `SignalWithStartWorkflowExecution` | `InvalidArgument` | Malformed request — bad workflow type, bad signal name, oversized payload, or invalid options. |
| `SignalWithStartWorkflowExecution` | `NotFound` | Namespace not found. |
| `SignalWithStartWorkflowExecution` | `PermissionDenied` | Auth failure. |
| `SignalWithStartWorkflowExecution` | `Unavailable` | Typically related to server issues — frontend unreachable or other server-side errors. Can be intermittent during server scaling or restarts, in which case it is generally non-critical. If seen persistently, alert on this and check server health. |
| `SignalWithStartWorkflowExecution` | `Internal` | Typically unexpected server-side issues. Can be intermittent during things like server restarts or persistence DB issues. If seen at a higher rate or over a longer duration, check server health. |
| `ExecuteMultiOperation` | `ResourceExhausted` | Execution bucket RPS exhausted at Priority 1. |
| `ExecuteMultiOperation` | `InvalidArgument` | One or more of the contained operations has a malformed request. |
| `ExecuteMultiOperation` | `AlreadyExists` | A `StartWorkflow` within the multi-operation found a conflicting running workflow. |
| `ExecuteMultiOperation` | `NotFound` | Namespace or workflow not found for one of the contained operations. |
| `ExecuteMultiOperation` | `Unavailable` | Typically related to server issues — frontend unreachable or other server-side errors. Can be intermittent during server scaling or restarts, in which case it is generally non-critical. If seen persistently, alert on this and check server health. |
| `ExecuteMultiOperation` | `Internal` | Typically unexpected server-side issues. Can be intermittent during things like server restarts or persistence DB issues. If seen at a higher rate or over a longer duration, check server health. |
| `UpdateWorkflowExecution` | `NotFound` | Workflow not found — completed, terminated, or exceeded retention. |
| `UpdateWorkflowExecution` | `ResourceExhausted` | Execution bucket RPS exhausted at Priority 1. Also subject to concurrent long-running request limit (`frontend.namespaceCount`) since this blocks until a WFT completes. |
| `UpdateWorkflowExecution` | `DeadlineExceeded` | A timeout occurred while waiting for the WFT to complete. Causes include: (1) server-side DB timeouts, check persistence metrics; (2) worker or client pod restart, look for `ShutdownWorker`, `DescribeNamespace`, or `GetSystemInfo` around the same time; (3) workers struggling — non-determinism issues or generic task failures preventing WFT completion; (4) gRPC proxy timeout. Check `workflow_task_schedule_to_start_latency` and worker health. |
| `UpdateWorkflowExecution` | `InvalidArgument` | Bad update name, oversized payload, or too many pending updates. |
| `UpdateWorkflowExecution` | `FailedPrecondition` | Workflow is not in a state that can accept updates (e.g. already completed or paused). |
| `UpdateWorkflowExecution` | `Unavailable` | Typically related to server issues — frontend unreachable or other server-side errors. Can be intermittent during server scaling or restarts, in which case it is generally non-critical. If seen persistently, alert on this and check server health. |
| `UpdateWorkflowExecution` | `Internal` | Typically unexpected server-side issues. Can be intermittent during things like server restarts or persistence DB issues. If seen at a higher rate or over a longer duration, check server health. |
| `RequestCancelWorkflowExecution` | `NotFound` | Workflow not found — already completed, terminated, or exceeded retention. Often benign in fire-and-forget cancel patterns. |
| `RequestCancelWorkflowExecution` | `ResourceExhausted` | Execution bucket RPS exhausted at Priority 2. |
| `RequestCancelWorkflowExecution` | `FailedPrecondition` | Workflow already in a terminal state. |
| `RequestCancelWorkflowExecution` | `PermissionDenied` | Auth failure. |
| `RequestCancelWorkflowExecution` | `Unavailable` | Typically related to server issues — frontend unreachable or other server-side errors. Can be intermittent during server scaling or restarts, in which case it is generally non-critical. If seen persistently, alert on this and check server health. |
| `RequestCancelWorkflowExecution` | `Internal` | Typically unexpected server-side issues. Can be intermittent during things like server restarts or persistence DB issues. If seen at a higher rate or over a longer duration, check server health. |
| `TerminateWorkflowExecution` | `NotFound` | Workflow not found. Already completed or exceeded retention. |
| `TerminateWorkflowExecution` | `ResourceExhausted` | Execution bucket RPS exhausted at Priority 2. |
| `TerminateWorkflowExecution` | `FailedPrecondition` | Workflow already in terminal state. |
| `TerminateWorkflowExecution` | `PermissionDenied` | Auth failure. |
| `TerminateWorkflowExecution` | `Unavailable` | Typically related to server issues — frontend unreachable or other server-side errors. Can be intermittent during server scaling or restarts, in which case it is generally non-critical. If seen persistently, alert on this and check server health. |
| `TerminateWorkflowExecution` | `Internal` | Typically unexpected server-side issues. Can be intermittent during things like server restarts or persistence DB issues. If seen at a higher rate or over a longer duration, check server health. |
| `ResetWorkflowExecution` | `NotFound` | Workflow or the specified reset point event not found. |
| `ResetWorkflowExecution` | `ResourceExhausted` | Execution bucket RPS exhausted at Priority 2. |
| `ResetWorkflowExecution` | `InvalidArgument` | Bad reset point — event ID out of range, or reset to a non-resettable event type. |
| `ResetWorkflowExecution` | `FailedPrecondition` | Workflow in a state that cannot be reset (e.g. still running with pending activities). |
| `ResetWorkflowExecution` | `Unavailable` | Typically related to server issues — frontend unreachable or other server-side errors. Can be intermittent during server scaling or restarts, in which case it is generally non-critical. If seen persistently, alert on this and check server health. |
| `ResetWorkflowExecution` | `Internal` | Typically unexpected server-side issues. Can be intermittent during things like server restarts or persistence DB issues. If seen at a higher rate or over a longer duration, check server health. |
| `DeleteWorkflowExecution` | `NotFound` | Workflow not found. |
| `DeleteWorkflowExecution` | `ResourceExhausted` | Execution bucket RPS exhausted at Priority 2. |
| `DeleteWorkflowExecution` | `FailedPrecondition` | Workflow is still running. Must cancel or terminate before deleting. |
| `DeleteWorkflowExecution` | `Unavailable` | Typically related to server issues — frontend unreachable or other server-side errors. Can be intermittent during server scaling or restarts, in which case it is generally non-critical. If seen persistently, alert on this and check server health. |
| `DeleteWorkflowExecution` | `Internal` | Typically unexpected server-side issues. Can be intermittent during things like server restarts or persistence DB issues. If seen at a higher rate or over a longer duration, check server health. |
| `QueryWorkflow` | `NotFound` | Workflow not found. |
| `QueryWorkflow` | `ResourceExhausted` | Execution bucket RPS exhausted at Priority 3. Also subject to concurrent long-running request limit since query blocks until a WFT completes. |
| `QueryWorkflow` | `DeadlineExceeded` | A timeout occurred while waiting for the query WFT to complete. Causes include: (1) server-side DB timeouts, check persistence metrics; (2) worker or client pod restart, look for `ShutdownWorker`, `DescribeNamespace`, or `GetSystemInfo` around the same time; (3) workers struggling — non-determinism issues, generic task failures, or workers simply not keeping up with the WFT backlog; (4) gRPC proxy timeout. Check `workflow_task_schedule_to_start_latency` and worker health. |
| `QueryWorkflow` | `InvalidArgument` | Unknown query type or bad query arguments. |
| `QueryWorkflow` | `FailedPrecondition` | Workflow is in a state where queries are not supported (e.g. workflow completed and query handler state is unavailable). |
| `QueryWorkflow` | `Unavailable` | Typically related to server issues — frontend unreachable or other server-side errors. Can be intermittent during server scaling or restarts, in which case it is generally non-critical. If seen persistently, alert on this and check server health. |
| `QueryWorkflow` | `Internal` | Typically unexpected server-side issues. Can be intermittent during things like server restarts or persistence DB issues. If seen at a higher rate or over a longer duration, check server health. |
| `DescribeWorkflowExecution` | `NotFound` | Workflow not found — completed and exceeded retention, wrong ID, or wrong namespace. Very common when polling for workflow status after retention period. |
| `DescribeWorkflowExecution` | `ResourceExhausted` | Execution bucket RPS exhausted at Priority 3. Frequent polling of `DescribeWorkflowExecution` in tight loops is a common cause. Consider using `GetWorkflowExecutionHistory` with `wait_new_event=true` instead. |
| `DescribeWorkflowExecution` | `Unavailable` | Typically related to server issues — frontend unreachable or other server-side errors. Can be intermittent during server scaling or restarts, in which case it is generally non-critical. If seen persistently, alert on this and check server health. |
| `DescribeWorkflowExecution` | `Internal` | Typically unexpected server-side issues. Can be intermittent during things like server restarts or persistence DB issues. If seen at a higher rate or over a longer duration, check server health. |
| `GetWorkflowExecutionHistory` (non-long-poll) | `NotFound` | Workflow not found — exceeded retention or wrong ID. |
| `GetWorkflowExecutionHistory` (non-long-poll) | `ResourceExhausted` | Execution bucket RPS exhausted at Priority 2. History fetch has relatively high priority because it is required for replay. Sustained exhaustion may indicate too many concurrent replays or SDK retries. |
| `GetWorkflowExecutionHistory` (non-long-poll) | `DeadlineExceeded` | A timeout occurred fetching history. Causes include: (1) server-side DB timeouts — history can be large and slow to read, check persistence metrics; (2) worker or client pod restart, look for `ShutdownWorker`, `DescribeNamespace`, or `GetSystemInfo` around the same time; (3) gRPC proxy timeout. |
| `GetWorkflowExecutionHistory` (non-long-poll) | `InvalidArgument` | Bad page token or invalid history fetch parameters. |
| `GetWorkflowExecutionHistory` (non-long-poll) | `Unavailable` | Typically related to server issues — frontend unreachable or other server-side errors. Can be intermittent during server scaling or restarts, in which case it is generally non-critical. If seen persistently, alert on this and check server health. |
| `GetWorkflowExecutionHistory` (non-long-poll) | `Internal` | Typically unexpected server-side issues. Can be intermittent during things like server restarts or persistence DB issues. If seen at a higher rate or over a longer duration, check server health. |
| `GetWorkflowExecutionHistoryReverse` | `NotFound` | Workflow not found. |
| `GetWorkflowExecutionHistoryReverse` | `ResourceExhausted` | Execution bucket RPS exhausted at Priority 4. |
| `GetWorkflowExecutionHistoryReverse` | `InvalidArgument` | Bad page token or parameters. |
| `GetWorkflowExecutionHistoryReverse` | `Unavailable` | Typically related to server issues — frontend unreachable or other server-side errors. Can be intermittent during server scaling or restarts, in which case it is generally non-critical. If seen persistently, alert on this and check server health. |
| `GetWorkflowExecutionHistoryReverse` | `Internal` | Typically unexpected server-side issues. Can be intermittent during things like server restarts or persistence DB issues. If seen at a higher rate or over a longer duration, check server health. |
| `RespondWorkflowTaskCompleted` | `NotFound` | Task token is invalid or stale — the WFT timed out and was rescheduled on another worker before this worker could respond. Common when WFT processing is slow relative to `WorkflowTaskTimeout`. |
| `RespondWorkflowTaskCompleted` | `ResourceExhausted` | Execution bucket RPS exhausted at Priority 1. Sustained failures here will cause WFTs to time out and be retried, amplifying load. |
| `RespondWorkflowTaskCompleted` | `InvalidArgument` | Commands are invalid — non-determinism detected, unsupported command type, malformed payloads, or too many pending activities/timers/child workflows/signals (server enforces limits). |
| `RespondWorkflowTaskCompleted` | `FailedPrecondition` | Workflow state has changed in a way that invalidates the commands (e.g. workflow was reset). |
| `RespondWorkflowTaskCompleted` | `Unavailable` | Typically related to server issues — frontend unreachable or other server-side errors. Can be intermittent during server scaling or restarts, in which case it is generally non-critical. If seen persistently, alert on this and check server health. |
| `RespondWorkflowTaskCompleted` | `Internal` | Typically unexpected server-side issues. Can be intermittent during things like server restarts or persistence DB issues. If seen at a higher rate or over a longer duration, check server health. |
| `RespondWorkflowTaskFailed` | `NotFound` | Task token stale — WFT already timed out and rescheduled. |
| `RespondWorkflowTaskFailed` | `ResourceExhausted` | Execution bucket RPS exhausted at Priority 3. Lower priority than completion — under extreme load, failure reports may be dropped, causing WFT to time out instead. |
| `RespondWorkflowTaskFailed` | `InvalidArgument` | Malformed failure details. |
| `RespondWorkflowTaskFailed` | `Unavailable` | Typically related to server issues — frontend unreachable or other server-side errors. Can be intermittent during server scaling or restarts, in which case it is generally non-critical. If seen persistently, alert on this and check server health. |
| `RespondWorkflowTaskFailed` | `Internal` | Typically unexpected server-side issues. Can be intermittent during things like server restarts or persistence DB issues. If seen at a higher rate or over a longer duration, check server health. |
| `RespondActivityTaskCompleted` | `NotFound` | Task token stale — activity timed out and was rescheduled, or was canceled/terminated externally before the worker responded. Very common in systems with tight `StartToCloseTimeout`. |
| `RespondActivityTaskCompleted` | `ResourceExhausted` | Execution bucket RPS exhausted at Priority 1. |
| `RespondActivityTaskCompleted` | `InvalidArgument` | Malformed result payload (oversized) or invalid task token. |
| `RespondActivityTaskCompleted` | `Unavailable` | Typically related to server issues — frontend unreachable or other server-side errors. Can be intermittent during server scaling or restarts, in which case it is generally non-critical. If seen persistently, alert on this and check server health. |
| `RespondActivityTaskCompleted` | `Internal` | Typically unexpected server-side issues. Can be intermittent during things like server restarts or persistence DB issues. If seen at a higher rate or over a longer duration, check server health. |
| `RespondActivityTaskCompletedById` | `NotFound` | Activity with this workflow/activity ID not found — already completed, timed out, or canceled. |
| `RespondActivityTaskCompletedById` | `ResourceExhausted` | Execution bucket RPS exhausted at Priority 1. |
| `RespondActivityTaskCompletedById` | `InvalidArgument` | Malformed result or bad ID. |
| `RespondActivityTaskCompletedById` | `Unavailable` | Typically related to server issues — frontend unreachable or other server-side errors. Can be intermittent during server scaling or restarts, in which case it is generally non-critical. If seen persistently, alert on this and check server health. |
| `RespondActivityTaskCompletedById` | `Internal` | Typically unexpected server-side issues. Can be intermittent during things like server restarts or persistence DB issues. If seen at a higher rate or over a longer duration, check server health. |
| `RespondActivityTaskFailed` | `NotFound` | Task token stale — activity timed out or was externally canceled before the failure report arrived. |
| `RespondActivityTaskFailed` | `ResourceExhausted` | Execution bucket RPS exhausted at Priority 3. Lower than completion — under extreme load this may be dropped. |
| `RespondActivityTaskFailed` | `InvalidArgument` | Malformed failure payload or oversized error details. |
| `RespondActivityTaskFailed` | `Unavailable` | Typically related to server issues — frontend unreachable or other server-side errors. Can be intermittent during server scaling or restarts, in which case it is generally non-critical. If seen persistently, alert on this and check server health. |
| `RespondActivityTaskFailed` | `Internal` | Typically unexpected server-side issues. Can be intermittent during things like server restarts or persistence DB issues. If seen at a higher rate or over a longer duration, check server health. |
| `RespondActivityTaskFailedById` | `NotFound` | Activity not found — timed out, canceled, or already completed. |
| `RespondActivityTaskFailedById` | `ResourceExhausted` | Execution bucket RPS exhausted at Priority 3. |
| `RespondActivityTaskFailedById` | `Unavailable` | Typically related to server issues — frontend unreachable or other server-side errors. Can be intermittent during server scaling or restarts, in which case it is generally non-critical. If seen persistently, alert on this and check server health. |
| `RespondActivityTaskFailedById` | `Internal` | Typically unexpected server-side issues. Can be intermittent during things like server restarts or persistence DB issues. If seen at a higher rate or over a longer duration, check server health. |
| `RespondActivityTaskCanceled` | `NotFound` | Task token stale. |
| `RespondActivityTaskCanceled` | `ResourceExhausted` | Execution bucket RPS exhausted at Priority 3. |
| `RespondActivityTaskCanceled` | `Unavailable` | Typically related to server issues — frontend unreachable or other server-side errors. Can be intermittent during server scaling or restarts, in which case it is generally non-critical. If seen persistently, alert on this and check server health. |
| `RespondActivityTaskCanceled` | `Internal` | Typically unexpected server-side issues. Can be intermittent during things like server restarts or persistence DB issues. If seen at a higher rate or over a longer duration, check server health. |
| `RespondActivityTaskCanceledById` | `NotFound` | Activity not found. |
| `RespondActivityTaskCanceledById` | `ResourceExhausted` | Execution bucket RPS exhausted at Priority 3. |
| `RespondActivityTaskCanceledById` | `Unavailable` | Typically related to server issues — frontend unreachable or other server-side errors. Can be intermittent during server scaling or restarts, in which case it is generally non-critical. If seen persistently, alert on this and check server health. |
| `RespondActivityTaskCanceledById` | `Internal` | Typically unexpected server-side issues. Can be intermittent during things like server restarts or persistence DB issues. If seen at a higher rate or over a longer duration, check server health. |
| `RecordActivityTaskHeartbeat` | `NotFound` | Task token stale — activity timed out (heartbeat timeout exceeded) or was canceled/terminated. A heartbeat `NotFound` is the canonical signal that an activity's heartbeat timeout was missed. Check `HeartbeatTimeout` configuration and whether the activity loop calls heartbeat frequently enough. |
| `RecordActivityTaskHeartbeat` | `ResourceExhausted` | Execution bucket RPS exhausted at Priority 1. Heartbeats are high priority — sustained exhaustion here is serious as it causes heartbeat timeouts. |
| `RecordActivityTaskHeartbeat` | `InvalidArgument` | Oversized heartbeat details payload. |
| `RecordActivityTaskHeartbeat` | `Unavailable` | Typically related to server issues — frontend unreachable or other server-side errors. Can be intermittent during server scaling or restarts, in which case it is generally non-critical. If seen persistently, alert on this and check server health. |
| `RecordActivityTaskHeartbeat` | `Internal` | Typically unexpected server-side issues. Can be intermittent during things like server restarts or persistence DB issues. If seen at a higher rate or over a longer duration, check server health. |
| `RecordActivityTaskHeartbeatById` | `NotFound` | Activity not found — same heartbeat timeout signal as token-based heartbeat. |
| `RecordActivityTaskHeartbeatById` | `ResourceExhausted` | Execution bucket RPS exhausted at Priority 1. |
| `RecordActivityTaskHeartbeatById` | `InvalidArgument` | Oversized payload. |
| `RecordActivityTaskHeartbeatById` | `Unavailable` | Typically related to server issues — frontend unreachable or other server-side errors. Can be intermittent during server scaling or restarts, in which case it is generally non-critical. If seen persistently, alert on this and check server health. |
| `RecordActivityTaskHeartbeatById` | `Internal` | Typically unexpected server-side issues. Can be intermittent during things like server restarts or persistence DB issues. If seen at a higher rate or over a longer duration, check server health. |
| `RespondQueryTaskCompleted` | `NotFound` | Query task no longer exists — query timed out before the worker could respond, or the calling client disconnected. |
| `RespondQueryTaskCompleted` | `ResourceExhausted` | Execution bucket RPS exhausted at Priority 1. |
| `RespondQueryTaskCompleted` | `InvalidArgument` | Malformed query result or oversized response payload. |
| `RespondQueryTaskCompleted` | `Unavailable` | Typically related to server issues — frontend unreachable or other server-side errors. Can be intermittent during server scaling or restarts, in which case it is generally non-critical. If seen persistently, alert on this and check server health. |
| `RespondQueryTaskCompleted` | `Internal` | Typically unexpected server-side issues. Can be intermittent during things like server restarts or persistence DB issues. If seen at a higher rate or over a longer duration, check server health. |
| `DescribeTaskQueue` | `NotFound` | Namespace or task queue not found. |
| `DescribeTaskQueue` | `ResourceExhausted` | **Visibility bucket** RPS exhausted at Priority 1 — `DescribeTaskQueue` counts against visibility quota, not execution quota. Low default of 10 RPS means frequent polling of task queue status (e.g. monitoring dashboards) can exhaust the visibility quota and cause `ResourceExhausted` on unrelated list calls. |
| `DescribeTaskQueue` | `InvalidArgument` | Bad task queue name or unsupported describe options. |
| `DescribeTaskQueue` | `Unavailable` | Typically related to server issues — frontend unreachable or other server-side errors. Can be intermittent during server scaling or restarts, in which case it is generally non-critical. If seen persistently, alert on this and check server health. |
| `DescribeTaskQueue` | `Internal` | Typically unexpected server-side issues. Can be intermittent during things like server restarts or persistence DB issues. If seen at a higher rate or over a longer duration, check server health. |
| `ListWorkflowExecutions` | `ResourceExhausted` | Visibility bucket RPS exhausted at Priority 1. Default is 10 RPS per instance — the most commonly hit limit in production for teams with monitoring or user-facing workflow list APIs. Increase `frontend.namespaceRPS.visibility`. |
| `ListWorkflowExecutions` | `InvalidArgument` | Malformed query syntax, unsupported search attribute, or page size exceeds `frontend.visibilityMaxPageSize`. |
| `ListWorkflowExecutions` | `Unavailable` | Typically related to server issues — frontend unreachable, or the visibility store (Elasticsearch/SQL) unavailable. Can be intermittent during server scaling or restarts, in which case it is generally non-critical. If seen persistently, alert on this and check server health and visibility store health. |
| `ListWorkflowExecutions` | `Internal` | Typically unexpected server-side issues. Can be intermittent during things like server restarts or persistence DB issues. If seen at a higher rate or over a longer duration, check server health. |
| `CountWorkflowExecutions` | `ResourceExhausted` | Visibility bucket RPS exhausted at Priority 1. Count queries can be expensive — frequent calls amplify visibility store load. |
| `CountWorkflowExecutions` | `InvalidArgument` | Malformed query syntax. |
| `CountWorkflowExecutions` | `Unavailable` | Typically related to server issues or visibility store unavailable. Can be intermittent during scaling or restarts. If seen persistently, alert and check visibility store health. |
| `CountWorkflowExecutions` | `Internal` | Typically unexpected server-side issues. Can be intermittent during things like server restarts or persistence DB issues. If seen at a higher rate or over a longer duration, check server health. |
| `ListOpenWorkflowExecutions` | `ResourceExhausted` | Visibility bucket RPS exhausted at Priority 1. |
| `ListOpenWorkflowExecutions` | `InvalidArgument` | Invalid filter combination or page size. |
| `ListOpenWorkflowExecutions` | `Unavailable` | Typically related to server issues or visibility store unavailable. Can be intermittent during scaling or restarts. If seen persistently, alert and check visibility store health. |
| `ListOpenWorkflowExecutions` | `Internal` | Typically unexpected server-side issues. Can be intermittent during things like server restarts or persistence DB issues. If seen at a higher rate or over a longer duration, check server health. |
| `ListClosedWorkflowExecutions` | `ResourceExhausted` | Visibility bucket RPS exhausted at Priority 1. |
| `ListClosedWorkflowExecutions` | `InvalidArgument` | Invalid filter or page size. |
| `ListClosedWorkflowExecutions` | `Unavailable` | Typically related to server issues or visibility store unavailable. Can be intermittent during scaling or restarts. If seen persistently, alert and check visibility store health. |
| `ListClosedWorkflowExecutions` | `Internal` | Typically unexpected server-side issues. Can be intermittent during things like server restarts or persistence DB issues. If seen at a higher rate or over a longer duration, check server health. |
| `ScanWorkflowExecutions` | `ResourceExhausted` | Visibility bucket RPS exhausted at Priority 1. Scans are expensive — they read large unordered sets from the visibility store. |
| `ScanWorkflowExecutions` | `InvalidArgument` | Malformed query or invalid options. |
| `ScanWorkflowExecutions` | `Unavailable` | Typically related to server issues or visibility store unavailable. Can be intermittent during scaling or restarts. If seen persistently, alert and check visibility store health. |
| `ScanWorkflowExecutions` | `Internal` | Typically unexpected server-side issues. Can be intermittent during things like server restarts or persistence DB issues. If seen at a higher rate or over a longer duration, check server health. |
| `ListArchivedWorkflowExecutions` | `ResourceExhausted` | Visibility bucket RPS exhausted. |
| `ListArchivedWorkflowExecutions` | `InvalidArgument` | Archival not enabled or malformed query. |
| `ListArchivedWorkflowExecutions` | `Unavailable` | Typically related to server issues or archival store unavailable. Can be intermittent during scaling or restarts. If seen persistently, alert and check archival store health. |
| `ListArchivedWorkflowExecutions` | `Internal` | Typically unexpected server-side issues. Can be intermittent during things like server restarts or persistence DB issues. If seen at a higher rate or over a longer duration, check server health. |
| `ListActivityExecutions` | `ResourceExhausted` | Visibility bucket RPS exhausted at Priority 1. |
| `ListActivityExecutions` | `InvalidArgument` | Malformed query or bad filter. |
| `ListActivityExecutions` | `Unavailable` | Typically related to server issues or visibility store unavailable. Can be intermittent during scaling or restarts. If seen persistently, alert and check visibility store health. |
| `ListActivityExecutions` | `Internal` | Typically unexpected server-side issues. Can be intermittent during things like server restarts or persistence DB issues. If seen at a higher rate or over a longer duration, check server health. |
| `CountActivityExecutions` | `ResourceExhausted` | Visibility bucket RPS exhausted at Priority 1. |
| `CountActivityExecutions` | `InvalidArgument` | Malformed query. |
| `CountActivityExecutions` | `Unavailable` | Typically related to server issues or visibility store unavailable. Can be intermittent during scaling or restarts. If seen persistently, alert and check visibility store health. |
| `CountActivityExecutions` | `Internal` | Typically unexpected server-side issues. Can be intermittent during things like server restarts or persistence DB issues. If seen at a higher rate or over a longer duration, check server health. |
| `ListSchedules` | `ResourceExhausted` | Visibility bucket RPS exhausted at Priority 1. |
| `ListSchedules` | `InvalidArgument` | Bad page size or filter. |
| `ListSchedules` | `Unavailable` | Typically related to server issues or visibility store unavailable. Can be intermittent during scaling or restarts. If seen persistently, alert and check visibility store health. |
| `ListSchedules` | `Internal` | Typically unexpected server-side issues. Can be intermittent during things like server restarts or persistence DB issues. If seen at a higher rate or over a longer duration, check server health. |
| `CreateSchedule` | `AlreadyExists` | Schedule with this ID already exists. |
| `CreateSchedule` | `ResourceExhausted` | Execution bucket RPS exhausted at Priority 1. |
| `CreateSchedule` | `InvalidArgument` | Malformed schedule spec, bad cron expression, or invalid policy. |
| `CreateSchedule` | `Unavailable` | Typically related to server issues — frontend unreachable or other server-side errors. Can be intermittent during server scaling or restarts, in which case it is generally non-critical. If seen persistently, alert on this and check server health. |
| `CreateSchedule` | `Internal` | Typically unexpected server-side issues. Can be intermittent during things like server restarts or persistence DB issues. If seen at a higher rate or over a longer duration, check server health. |
| `DescribeSchedule` | `NotFound` | Schedule not found. |
| `DescribeSchedule` | `ResourceExhausted` | Execution bucket RPS exhausted at Priority 3. |
| `DescribeSchedule` | `Unavailable` | Typically related to server issues — frontend unreachable or other server-side errors. Can be intermittent during server scaling or restarts, in which case it is generally non-critical. If seen persistently, alert on this and check server health. |
| `DescribeSchedule` | `Internal` | Typically unexpected server-side issues. Can be intermittent during things like server restarts or persistence DB issues. If seen at a higher rate or over a longer duration, check server health. |
| `UpdateSchedule` | `NotFound` | Schedule not found. |
| `UpdateSchedule` | `ResourceExhausted` | Execution bucket RPS exhausted at Priority 2. |
| `UpdateSchedule` | `InvalidArgument` | Malformed schedule spec. |
| `UpdateSchedule` | `Unavailable` | Typically related to server issues — frontend unreachable or other server-side errors. Can be intermittent during server scaling or restarts, in which case it is generally non-critical. If seen persistently, alert on this and check server health. |
| `UpdateSchedule` | `Internal` | Typically unexpected server-side issues. Can be intermittent during things like server restarts or persistence DB issues. If seen at a higher rate or over a longer duration, check server health. |
| `PatchSchedule` | `NotFound` | Schedule not found. |
| `PatchSchedule` | `ResourceExhausted` | Execution bucket RPS exhausted at Priority 2. |
| `PatchSchedule` | `InvalidArgument` | Invalid patch request (bad backfill range, etc.). |
| `PatchSchedule` | `Unavailable` | Typically related to server issues — frontend unreachable or other server-side errors. Can be intermittent during server scaling or restarts, in which case it is generally non-critical. If seen persistently, alert on this and check server health. |
| `PatchSchedule` | `Internal` | Typically unexpected server-side issues. Can be intermittent during things like server restarts or persistence DB issues. If seen at a higher rate or over a longer duration, check server health. |
| `DeleteSchedule` | `NotFound` | Schedule not found. |
| `DeleteSchedule` | `ResourceExhausted` | Execution bucket RPS exhausted at Priority 2. |
| `DeleteSchedule` | `Unavailable` | Typically related to server issues — frontend unreachable or other server-side errors. Can be intermittent during server scaling or restarts, in which case it is generally non-critical. If seen persistently, alert on this and check server health. |
| `DeleteSchedule` | `Internal` | Typically unexpected server-side issues. Can be intermittent during things like server restarts or persistence DB issues. If seen at a higher rate or over a longer duration, check server health. |
| `DescribeNamespace` | `NotFound` | Namespace not found. Worker or client configured with wrong namespace name. |
| `DescribeNamespace` | `ResourceExhausted` | Execution bucket RPS exhausted at Priority 0 — very rare since P0 is shed last. Indicates extreme load. |
| `DescribeNamespace` | `Unavailable` | Typically related to server issues — frontend unreachable or other server-side errors. Can be intermittent during server scaling or restarts, in which case it is generally non-critical. If seen persistently, alert on this and check server health. |
| `DescribeNamespace` | `Internal` | Typically unexpected server-side issues. Can be intermittent during things like server restarts or persistence DB issues. If seen at a higher rate or over a longer duration, check server health. |
| `RegisterNamespace` | `AlreadyExists` | Namespace already registered. |
| `RegisterNamespace` | `ResourceExhausted` | NamespaceReplicationInducing bucket exhausted — default global limit is 20 RPS cluster-wide. |
| `RegisterNamespace` | `InvalidArgument` | Bad namespace name, invalid retention period, or unsupported config. |
| `RegisterNamespace` | `Unavailable` | Typically related to server issues — frontend unreachable or other server-side errors. Can be intermittent during server scaling or restarts, in which case it is generally non-critical. If seen persistently, alert on this and check server health. |
| `RegisterNamespace` | `Internal` | Typically unexpected server-side issues. Can be intermittent during things like server restarts or persistence DB issues. If seen at a higher rate or over a longer duration, check server health. |
| `UpdateNamespace` | `NotFound` | Namespace not found. |
| `UpdateNamespace` | `ResourceExhausted` | NamespaceReplicationInducing bucket exhausted — 20 RPS global default. |
| `UpdateNamespace` | `InvalidArgument` | Bad config — invalid retention, unsupported archival state, etc. |
| `UpdateNamespace` | `FailedPrecondition` | Namespace in a state that prevents update (e.g. deprecated). |
| `UpdateNamespace` | `Unavailable` | Typically related to server issues — frontend unreachable or other server-side errors. Can be intermittent during server scaling or restarts, in which case it is generally non-critical. If seen persistently, alert on this and check server health. |
| `UpdateNamespace` | `Internal` | Typically unexpected server-side issues. Can be intermittent during things like server restarts or persistence DB issues. If seen at a higher rate or over a longer duration, check server health. |
| `UpdateWorkerBuildIdCompatibility` | `ResourceExhausted` | NamespaceReplicationInducing bucket exhausted at Priority 2. |
| `UpdateWorkerBuildIdCompatibility` | `NotFound` | Namespace not found. |
| `UpdateWorkerBuildIdCompatibility` | `InvalidArgument` | Invalid build ID, exceeds `limit.workerBuildIdSize`, or would exceed `limit.versionCompatibleSetLimitPerQueue`. |
| `UpdateWorkerBuildIdCompatibility` | `FailedPrecondition` | Versioning data constraints violated. |
| `UpdateWorkerBuildIdCompatibility` | `Unavailable` | Typically related to server issues — frontend unreachable or other server-side errors. Can be intermittent during server scaling or restarts, in which case it is generally non-critical. If seen persistently, alert on this and check server health. |
| `UpdateWorkerBuildIdCompatibility` | `Internal` | Typically unexpected server-side issues. Can be intermittent during things like server restarts or persistence DB issues. If seen at a higher rate or over a longer duration, check server health. |
| `UpdateWorkerVersioningRules` | `ResourceExhausted` | NamespaceReplicationInducing bucket exhausted at Priority 2. |
| `UpdateWorkerVersioningRules` | `NotFound` | Namespace or task queue not found. |
| `UpdateWorkerVersioningRules` | `InvalidArgument` | Invalid versioning rules — conflicting assignments, bad build ID format. |
| `UpdateWorkerVersioningRules` | `FailedPrecondition` | Conflict — rules have been modified since last read (optimistic concurrency conflict). Retry after re-fetching current rules. |
| `UpdateWorkerVersioningRules` | `Unavailable` | Typically related to server issues — frontend unreachable or other server-side errors. Can be intermittent during server scaling or restarts, in which case it is generally non-critical. If seen persistently, alert on this and check server health. |
| `UpdateWorkerVersioningRules` | `Internal` | Typically unexpected server-side issues. Can be intermittent during things like server restarts or persistence DB issues. If seen at a higher rate or over a longer duration, check server health. |
| `GetSystemInfo` | `ResourceExhausted` | Execution bucket RPS exhausted at Priority 0 — extremely rare. |
| `GetSystemInfo` | `Unavailable` | Typically related to server issues — frontend unreachable or other server-side errors. Can be intermittent during server scaling or restarts, in which case it is generally non-critical. If seen persistently, alert on this and check server health. |
| `GetSystemInfo` | `Internal` | Typically unexpected server-side issues. Can be intermittent during things like server restarts or persistence DB issues. If seen at a higher rate or over a longer duration, check server health. |
| `GetClusterInfo` | `ResourceExhausted` | Execution bucket RPS exhausted at Priority 0 — extremely rare. |
| `GetClusterInfo` | `Unavailable` | Typically related to server issues — frontend unreachable or other server-side errors. Can be intermittent during server scaling or restarts, in which case it is generally non-critical. If seen persistently, alert on this and check server health. |
| `GetClusterInfo` | `Internal` | Typically unexpected server-side issues. Can be intermittent during things like server restarts or persistence DB issues. If seen at a higher rate or over a longer duration, check server health. |
| `ResetStickyTaskQueue` | `NotFound` | Workflow not found. |
| `ResetStickyTaskQueue` | `ResourceExhausted` | Execution bucket RPS exhausted at Priority 4. |
| `ResetStickyTaskQueue` | `Unavailable` | Typically related to server issues — frontend unreachable or other server-side errors. Can be intermittent during server scaling or restarts, in which case it is generally non-critical. If seen persistently, alert on this and check server health. |
| `ResetStickyTaskQueue` | `Internal` | Typically unexpected server-side issues. Can be intermittent during things like server restarts or persistence DB issues. If seen at a higher rate or over a longer duration, check server health. |

---

## From SDK Metric to Server Dashboard

When an SDK metric spike points to a server-side issue, the next step is always to correlate it against your server dashboard. The table below maps the most important SDK failure patterns to the specific panels in the Temporal Server Dashboard and what to look for in each one.

The dashboard referenced here is the [Temporal Server Dashboard](https://github.com/tsurdilo/my-temporal-dockercompose/blob/main/deployment/grafana/dashboards/temporal-server.json), which covers the key server-side signal areas. All panels are namespace-scoped.

---

### `ResourceExhausted` on any operation

| Dashboard Section | Panel | What to look for |
|-------------------|-------|-----------------|
| Server Overview — Throttling | **Resource Exhausted with Cause** (`service_errors_resource_exhausted` by `operation` and `resource_exhausted_cause`) | This is your first stop. Look for `RpsLimit` (namespace RPS quota hit), `ConcurrentLimit` (too many simultaneous long-running requests), or `SystemOverload` (DB under pressure). The `operation` label will confirm which API is being throttled server-side, correlating directly to the `operation` tag on your SDK metric. |

---

### `Internal` or `Unavailable` on any operation

| Dashboard Section | Panel | What to look for |
|-------------------|-------|-----------------|
| Server Overview — Service Requests and Errors | **Service Errors** (`service_errors` on frontend) | A spike here confirms the server is returning errors. Look for a rate increase that correlates in time with your SDK metric spike. |
| Server Overview — Service Requests and Errors | **Server Errors by Type** (`service_error_with_type` by `error_type`) | Breaks down what kind of errors the frontend is returning — e.g. `persistence`, `resource_exhausted`, `deadline_exceeded`. Helps narrow down whether it is a DB issue, an overload condition, or something else. |
| Server Overview — Persistence Requests, Latencies and Errors | **Persistence Errors** (`persistence_error_with_type` by `operation` and `error_type`) | If `Internal` or `Unavailable` correlates with persistence errors, this confirms a DB-level issue. Look for elevated error rates on operations like `GetWorkflowExecution`, `UpdateWorkflowExecution`, or `CreateTasks`. |
| Server Overview — Persistence Requests, Latencies and Errors | **Persistence Availability** (gauge) | If this drops below 99%, your DB is struggling and will be causing cascading errors across all SDK operations. |
| Server Overview — Shard Movement | **Service Restarts** (`restarts{}` by `service_name`) | A spike here means a Temporal service restarted, which causes transient `Unavailable` and `Internal` errors across all operations. Correlate the restart time with your SDK metric spike. |

---

### `DeadlineExceeded` on any operation

| Dashboard Section | Panel | What to look for |
|-------------------|-------|-----------------|
| Server Overview — Persistence Requests, Latencies and Errors | **Persistence Latencies** (`persistence_latency_bucket` by `operation`) | High persistence latency is the most common server-side cause of `DeadlineExceeded`. Look for p99 latency spikes, especially on `GetWorkflowExecution` or history-related operations. |
| Server Overview — Cluster Throughput | **Shard Lock Latency** (`semaphore_latency_bucket` for `ShardInfo`) | Elevated shard lock latency means history service shards are contended, which introduces latency to all operations and can cause timeouts. Orange threshold at 200ms, red at 400ms. |
| Server Overview — Cluster Throughput | **Workflow Lock Latency** (`cache_latency_bucket` for `HistoryCacheGetOrCreate`) | High workflow lock latency means per-execution updates are queueing up. Common in high fan-out workflows. Orange at 200ms, red at 500ms. |
| Server Overview — Shard Movement | **Shards Created / Removed / Closed** | Shard movement during cluster scaling or restarts introduces transient latency spikes that can cause `DeadlineExceeded`. Correlate shard movement timing with your SDK metric spike. |
| Server Overview — Shard Movement | **Service Restarts** | Worker or service pod restarts will cancel in-flight requests and cause `DeadlineExceeded` on the client side. |

---

### `DeadlineExceeded` on `QueryWorkflow`, `UpdateWorkflowExecution`, or `PollWorkflowExecutionUpdate` specifically

These three operations block until a Workflow Task completes, so their `DeadlineExceeded` is often a worker health signal rather than a pure server issue.

| Dashboard Section | Panel | What to look for |
|-------------------|-------|-----------------|
| Server Overview — SDK Workers | **Schedule to Start Latencies** (`task_schedule_to_start_latency_bucket` by `task_type` and `operation`) | This is your primary signal. High schedule-to-start latency for `workflow` tasks means workers are not picking up WFTs fast enough. Orange threshold at 500ms, red at 2s. |
| Server Overview — SDK Workers | **Tasks Persisted to DB** (`persistence_requests` for `CreateTasks`) | A sustained increase here means the matching service cannot dispatch tasks to a waiting poller and is writing them to the database instead — a sign of worker under-provisioning or a growing backlog. |
| Server Overview — SDK Workers | **Sync Match Rate** (`syncmatch_latency_bucket` on matching) | Low or degrading sync match rate means tasks are not being picked up by waiting pollers within the sync match window (~500ms), indicating insufficient pollers or worker capacity. |
| Server Overview — Task Timeouts and Backlog | **Approximate Task Backlog** (`approximate_backlog_count` by `task_type`) | A growing backlog confirms workers are falling behind. If this is rising alongside `DeadlineExceeded` on query/update operations, workers need to be scaled up. |

---

### `NotFound` on `RespondActivityTaskCompleted`, `RespondActivityTaskFailed`, or `RecordActivityTaskHeartbeat`

These `NotFound` responses mean the activity's task token was stale — the activity timed out on the server before the worker could respond.

| Dashboard Section | Panel | What to look for |
|-------------------|-------|-----------------|
| Server Overview — Task Timeouts and Backlog | **Activity StartToClose Timeout** (`start_to_close_timeout` for `TimerActiveTaskActivityTimeout`) | A spike here directly confirms activities are timing out before completion. Correlate timing with your SDK `NotFound` spike. |
| Server Overview — Task Timeouts and Backlog | **Activity Heartbeat Timeout** (`heartbeat_timeout` for `TimerActiveTaskActivityTimeout`) | For `RecordActivityTaskHeartbeat` `NotFound` specifically — this confirms heartbeat timeouts are firing. Check your `HeartbeatTimeout` configuration and heartbeat frequency. |
| Server Overview — Task Timeouts and Backlog | **Approximate Task Backlog** | A growing backlog means activities are waiting too long to be picked up in the first place, contributing to `ScheduleToStart` timeouts upstream of the `StartToClose`. |

---

### `NotFound` on `RespondWorkflowTaskCompleted` or `RespondWorkflowTaskFailed`

Means the WFT timed out and was rescheduled before the worker could respond.

| Dashboard Section | Panel | What to look for |
|-------------------|-------|-----------------|
| Server Overview — Task Timeouts and Backlog | **Workflow Task StartToClose Timeouts** (`start_to_close_timeout` for `TimerActiveTaskWorkflowTaskTimeout`) | A spike here confirms WFTs are timing out. This usually means WFT processing on the worker is slower than the `WorkflowTaskTimeout`. |
| Server Overview — SDK Workers | **Schedule to Start Latencies** | If WFTs are timing out, schedule-to-start latency will also be elevated — check whether the issue is the worker not picking up tasks fast enough, or processing them too slowly once picked up. |

---

### `ResourceExhausted` on visibility operations (`ListWorkflowExecutions`, `CountWorkflowExecutions`, `DescribeTaskQueue`, etc.)

| Dashboard Section | Panel | What to look for |
|-------------------|-------|-----------------|
| Server Overview — Throttling | **Resource Exhausted with Cause** | Filter by the visibility operation name. Look for `RpsLimit` with a cause tied to the visibility bucket. |
| Server Overview — Visibility | **Visibility Availability** (gauge) | If visibility availability drops below 99%, the visibility store itself is struggling, which will amplify throttling as the server protects it with rate limits. |
| Server Overview — Visibility | **Visibility Latencies per Operation** (`task_latency_bucket` for `VisibilityTask.*`) | High visibility task latency means the underlying store (Elasticsearch/SQL) is slow, which can cause cascading errors on visibility API calls. |

---

### `Canceled` or `DeadlineExceeded` on poll operations correlating with service restarts

| Dashboard Section | Panel | What to look for |
|-------------------|-------|-----------------|
| Server Overview — Shard Movement | **Service Restarts** (`restarts{}` by `service_name`) | Confirm whether a service restart happened at the same time as your SDK metric spike. Restarts cause in-flight polls to be canceled or timed out. |
| Server Overview — Shard Movement | **Shards Created / Removed / Closed** | Shard movement during a restart or scaling event introduces transient disruption to poll operations. A burst of shard closures followed by creations is a normal restart pattern but will cause transient SDK errors. |

---

## Common `status_code` Values

| gRPC Code | Typical Meaning in Temporal |
|-----------|----------------------------|
| `NotFound` | Workflow/namespace/schedule doesn't exist or fell off retention |
| `AlreadyExists` | Workflow ID collision on start; duplicate schedule ID |
| `InvalidArgument` | Bad request parameters |
| `ResourceExhausted` | Rate limited — also emits `temporal_request_resource_exhausted` with a `cause` tag indicating which quota was hit |
| `Unavailable` | Server unreachable or transient infrastructure error |
| `DeadlineExceeded` | Client-side context timeout, server-side processing timeout, or proxy cutting a long-lived connection |
| `PermissionDenied` | Authorization failure |
| `Unauthenticated` | Missing or invalid credentials |
| `FailedPrecondition` | Workflow not in the correct state for the operation |
| `Internal` | Unexpected server-side error |
| `Canceled` | RPC canceled by the client |

---

## Metric Tag Format

```
# High-priority completion — P1 in Execution bucket
temporal_request_failure{
  namespace="your-namespace",
  operation="RespondWorkflowTaskCompleted",
  status_code="Internal"
}

# Visibility quota exhausted
temporal_request_failure{
  namespace="your-namespace",
  operation="ListWorkflowExecutions",
  status_code="ResourceExhausted"
}

# Long-poll failure — P4 in Execution bucket
temporal_long_request_failure{
  namespace="your-namespace",
  operation="PollWorkflowTaskQueue",
  status_code="ResourceExhausted"
}
```

---

## Sources

- [`temporalio/temporal` — `service/frontend/configs/quotas.go`](https://github.com/temporalio/temporal/blob/main/service/frontend/configs/quotas.go)
- [`temporalio/sdk-go` — `internal/common/metrics/grpc.go`](https://github.com/temporalio/sdk-go/blob/master/internal/common/metrics/grpc.go) *(used as reference implementation — the same interceptor pattern applies across all Temporal SDKs)*
- [`temporalio/sdk-go` — `internal/common/metrics/constants.go`](https://github.com/temporalio/sdk-go/blob/master/internal/common/metrics/constants.go) *(metric name and tag constants)*
- [`temporalio/api` — `workflowservice/v1/service.proto`](https://github.com/temporalio/api/blob/master/temporal/api/workflowservice/v1/service.proto)
- [`temporalio/api` — `operatorservice/v1/service.proto`](https://github.com/temporalio/api/blob/master/temporal/api/operatorservice/v1/service.proto)
- [Temporal TypeScript SDK API Reference — WorkflowService](https://typescript.temporal.io/api/classes/proto.temporal.api.workflowservice.v1.WorkflowService-1)
- [Temporal SDK Metrics Reference](https://docs.temporal.io/references/sdk-metrics)
- [Temporal Performance Bottlenecks Guide](https://docs.temporal.io/troubleshooting/performance-bottlenecks)