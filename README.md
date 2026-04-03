# Understanding Temporal SDK Request Failure Metrics
## `temporal_request_failure` & `temporal_long_request_failure`

---

## Table of Contents

1. [Why These Two Metrics](#why-these-two-metrics)
2. [How the SDK Distinguishes the Two Metrics](#how-the-sdk-distinguishes-the-two-metrics)
3. [SDK gRPC Retry Behavior](#sdk-grpc-retry-behavior)
4. [Alert Severity and Business Impact](#alert-severity-and-business-impact)
4. [Diagnostic Tables](#diagnostic-tables)
    - [`temporal_long_request_failure`](#temporal_long_request_failure-diagnostic-table)
    - [`temporal_request_failure`](#temporal_request_failure-diagnostic-table)
5. [From SDK Metric to Server Dashboard](#from-sdk-metric-to-server-dashboard)
6. [Common `status_code` Values](#common-status_code-values)
7. [Metric Tag Format](#metric-tag-format)
8. [Server-Side Throttling: Rate Limit Buckets and Priority](#server-side-throttling-rate-limit-buckets-and-priority)
    - [Execution Bucket](#execution-bucket)
    - [Visibility Bucket](#visibility-bucket)
    - [NamespaceReplicationInducing Bucket](#namespacereplicationinducing-bucket)
    - [Priority Within Each Bucket](#priority-within-each-bucket)
9. [Operation Reference Tables](#operation-reference-tables)
    - [Execution Bucket Operations](#execution-bucket-operations)
    - [Visibility Bucket Operations](#visibility-bucket-operations)
    - [NamespaceReplicationInducing Bucket Operations](#namespacereplicationinducing-bucket-operations)
10. [Sources](#sources)

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

## SDK gRPC Retry Behavior

Understanding which gRPC status codes the SDK retries automatically is critical for correctly interpreting `temporal_request_failure` and `temporal_long_request_failure`. A failure that appears in these metrics does not necessarily mean business impact — if the SDK retries the call successfully within its retry window, the application may never see the error.

---

### Non-Retryable Status Codes

The SDK will **not** retry these. If you see them in your metrics, the call failed and was returned to the caller immediately. These always represent either a configuration issue, a code issue, or a genuine state conflict.

| Status Code | Reason |
|-------------|--------|
| `InvalidArgument` | Bad request — retrying with the same payload will not help |
| `NotFound` | Resource does not exist |
| `AlreadyExists` | Conflict — already created |
| `FailedPrecondition` | System not in the correct state for this operation |
| `PermissionDenied` | Authorization failure |
| `Unauthenticated` | No valid credentials |
| `Unimplemented` | Server does not support this RPC |

---

### Retryable Status Codes

The SDK **will** retry these with backoff. Seeing them in your metrics does not necessarily mean the call ultimately failed — the SDK may have retried successfully. Sustained high rates or durations approaching the SDK's retry window (~60 seconds by default) are the signal to watch for.

| Status Code | Reason |
|-------------|--------|
| `Unavailable` | Transient server or network issue — the most common retry case |
| `ResourceExhausted` | Rate limited or quota exceeded — SDK retries with backoff |
| `Unknown` | Unclassified error |
| `Internal` | Server-side error, may be transient |
| `DeadlineExceeded` | **Conditional** — only retried if the root gRPC context deadline has not yet expired. If the overall deadline is already past, the SDK stops retrying. |

---

### Important: gRPC Message Size Too Large

One notable behavioral change in recent SDK releases: the SDK will **no longer retry** "gRPC message size too large" errors. These occur when a request payload exceeds the Temporal service limit (typically 4 MB) and come back as `ResourceExhausted` with a specific message indicating the size violation. Previously these were retried pointlessly since resubmitting the same oversized payload will never succeed. If you see `ResourceExhausted` on operations like `StartWorkflowExecution` but your RPS metrics look healthy, check whether the payload size is the cause rather than throttling.

---

### What This Means for Your Metrics

Because retryable errors are automatically retried by the SDK, a spike in `temporal_request_failure` for a retryable status code like `ResourceExhausted` or `Unavailable` does not immediately mean your application is broken. The key question is always: **is the SDK successfully retrying within its window, or are retries being exhausted?**

For operations where exhausted retries have direct business consequences — like `StartWorkflowExecution` dropping a workflow that will never be recreated — your application code should handle the final error after SDK retries are exhausted, log the relevant context (workflow ID, input payload), and have a remediation path such as a dead-letter queue or manual backfill process.

---

## Alert Severity and Business Impact

Not all failures surfaced by these metrics are equal. Two operations can both return `ResourceExhausted` but one is a minor performance degradation while the other is actively blocking your business. This section gives guidance on which failures warrant immediate alerting, which to monitor passively, and which are largely expected and informational.

These tiers are general guidance — your specific alerting thresholds should be tuned to your workload and business requirements.

---

### 🔴 High Impact — Alert Immediately

Failures here directly block new work from being created, prevent running workflows from making progress, or have immediate user-facing or business consequences. Sustained occurrences warrant immediate investigation.

| Metric | Operation | Status Code | Why it matters |
|--------|-----------|-------------|----------------|
| `temporal_request_failure` | `StartWorkflowExecution` | `ResourceExhausted` | High impact but not necessarily alert-immediately on first occurrence. The Temporal SDK automatically retries `ResourceExhausted` responses with backoff — each SDK has default gRPC retry options, so intermittent throttling lasting less than ~60 seconds may not result in any actual business impact as the SDK will keep retrying. If seen at a high rate, monitor whether workflow executions are actually failing to be created (look for errors on the client side if you control the client code). If throttling is sustained close to or beyond the SDK's retry window (~60s), alert immediately as executions will start being dropped. **Critically: your client code should always handle `StartWorkflowExecution` failures after SDK retries are exhausted by logging the workflow ID and input payload. If the call ultimately fails, those logs are your only way to identify and backfill the "lost" executions.** |
| `temporal_request_failure` | `StartWorkflowExecution` | `Unavailable` / `Internal` | Both are retried automatically by the SDK. A single occurrence or intermittent low-rate occurrences can be transient server issues — no immediate alert needed if the SDK retries succeed. If seen continuously or at a high rate approaching the SDK retry window (~60s), alert immediately as executions will start being dropped. Same as with `ResourceExhausted`: ensure your client code handles the final failure after SDK retries are exhausted by logging the workflow ID and input payload so lost executions can be backfilled. |
| `temporal_request_failure` | `SignalWorkflowExecution` | `ResourceExhausted` | High impact but not necessarily alert-immediately on first occurrence. Has several distinct causes: (1) **RPS throttling** — namespace execution bucket quota exhausted, check `frontend.namespaceRPS`; (2) **Signal limit reached** — Temporal enforces a limit on the number of signals a single workflow execution can receive, controlled by `history.maximumSignalsPerExecution`. If this limit is hit, new signals to that execution will be rejected with `ResourceExhausted` — check this dynamic config if throttling metrics look healthy; (3) **ContinueAsNew in progress** — when a workflow execution is performing a ContinueAsNew, the server will temporarily return `ResourceExhausted` for incoming signals to that execution while it allows the ContinueAsNew to complete. This is fully expected and intentional — the SDK must retry the signal so it lands on the continued execution. Do not alert on isolated occurrences. If seen continuously or at high rate across many workflows, investigate RPS limits. Log workflow ID and signal payload on final failure after SDK retries are exhausted. |
| `temporal_request_failure` | `ExecuteMultiOperation` / `UpdateWorkflowExecution` / `SignalWithStartWorkflowExecution` | `ResourceExhausted` | Same pattern as `StartWorkflowExecution` — SDK retries automatically with backoff. Intermittent occurrences may not result in actual loss. If seen continuously or at high rate approaching the ~60s retry window, alert. Log workflow ID and relevant payload on final failure to enable backfill. |
| `temporal_request_failure` | `RespondWorkflowTaskCompleted` | `ResourceExhausted` | High impact but not necessarily alert-immediately on first occurrence. Can be caused by RPS throttling (`RpsLimit`) or server overload (`SystemOverload`) — check the Resource Exhausted with Cause panel on your server dashboard. Can also be caused by the WFT completion response payload exceeding the 4 MB gRPC message size limit — check payload sizes if RPS metrics look healthy. Sustained throttling here will cause WFTs to time out (default `WorkflowTaskTimeout` is 10s), which creates a dangerous feedback loop: timed-out WFTs get retried, generating more load. Critically, if your workflows use **local activities**, WFT timeouts caused by this throttling will lead to re-execution of those local activities — which can cause side effects if they are not idempotent. Do not alert on isolated intermittent occurrences, but if seen for longer than ~5-6 seconds or at a high rate, alert immediately given the 10s default timeout window. |
| `temporal_request_failure` | `RespondWorkflowTaskCompleted` | `NotFound` (sustained, high rate) | Alert immediately unless you can account for it with one of the known expected causes below. Has several distinct causes: (1) **WFT timeout** — the worker responded after the `WorkflowTaskTimeout` (default 10s) expired and the server already rescheduled the task; this means workers are severely overloaded or WFT processing is far too slow; (2) **Workflow execution completed** — the worker responded after the execution was already terminated, timed out, or otherwise completed; (3) **WorkflowRunTimeout or WorkflowExecutionTimeout** — if these timeouts are configured and firing at a high rate, the server will complete those executions before workers can respond, producing `NotFound`. Before alerting, check whether the rate correlates with a known batch of terminations or a known pattern of execution/run timeouts expiring. If none of those explain it, treat as a worker health emergency and check `workflow_task_schedule_to_start_latency`, WFT processing time, and worker resource utilization. |
| `temporal_request_failure` | `RecordActivityTaskHeartbeat` | `ResourceExhausted` | Heartbeats are high priority (P1) — if these are being throttled, heartbeat timeouts will fire, causing activities to be marked as timed out even though the worker is still running them. |
| `temporal_request_failure` | `RespondActivityTaskCompleted` | `ResourceExhausted` | Activity completions are being throttled. Work is being done but results cannot be reported back, causing activities to time out and retry unnecessarily. |
| `temporal_long_request_failure` | `PollWorkflowExecutionUpdate` | `DeadlineExceeded` (sustained) | Workflow updates are timing out consistently — workers are not making progress on WFTs. Direct user-facing impact if updates are used in synchronous request/response patterns. |
| `temporal_request_failure` | `UpdateWorkflowExecution` | `DeadlineExceeded` (sustained) | Same as above — synchronous update calls are timing out. |
| `temporal_request_failure` | `QueryWorkflow` | `DeadlineExceeded` (sustained) | Query calls are timing out — workers are not keeping up with WFTs. Direct user-facing impact if queries power read APIs. |

---

### 🟡 Medium Impact — Monitor Closely

Failures here degrade performance, increase latency, or indicate developing problems that will become high impact if left unaddressed. Worth alerting on at a lower threshold or after sustained duration.

| Metric | Operation | Status Code | Why it matters |
|--------|-----------|-------------|----------------|
| `temporal_long_request_failure` | `PollWorkflowTaskQueue` | `ResourceExhausted` | Workers are being throttled on polls. Workflows will still make progress but schedule-to-start latency for workflow tasks will increase. Not immediately business-critical but signals a capacity problem that will worsen. |
| `temporal_long_request_failure` | `PollActivityTaskQueue` | `ResourceExhausted` | Same as workflow poll — activity schedule-to-start latency increases. Activities will still complete but with higher latency. |
| `temporal_request_failure` | `RespondActivityTaskCompleted` | `NotFound` (elevated rate) | Activities are timing out before workers can respond. May indicate `StartToCloseTimeout` is too tight for the actual activity duration, or workers are struggling under load. |
| `temporal_request_failure` | `RecordActivityTaskHeartbeat` | `NotFound` | Heartbeat timeouts are firing. Activities are being considered timed out by the server while still running. Check `HeartbeatTimeout` configuration and heartbeat frequency. |
| `temporal_request_failure` | `ListWorkflowExecutions` | `ResourceExhausted` | Visibility quota exhausted. User-facing workflow list or search features will be degraded. Impact depends on how critical visibility APIs are to your application. |
| `temporal_request_failure` | `RespondWorkflowTaskCompleted` | `InvalidArgument` | Non-determinism or pending limit violations. This is a code or configuration issue that needs investigation — workflows will be stuck in a failed WFT retry loop. |
| `temporal_long_request_failure` | `PollWorkflowTaskQueue` | `Unavailable` (sustained) | Frontend unreachable or server health issue. Workers cannot poll — workflows stop making progress. |
| `temporal_long_request_failure` | `PollActivityTaskQueue` | `Unavailable` (sustained) | Same — activity workers cannot poll. |

---

### 🟢 Low Impact — Informational / Expected

Failures here are largely expected in normal operation, have built-in retry paths, or are inherently transient. Monitor for anomalous spikes but do not alert on isolated occurrences.

| Metric | Operation | Status Code | Why it matters |
|--------|-----------|-------------|----------------|
| `temporal_long_request_failure` | `PollWorkflowTaskQueue` | `Canceled` | Expected during worker graceful shutdown or proxy connection resets. Normal during deployments. |
| `temporal_long_request_failure` | `PollActivityTaskQueue` | `Canceled` | Same as above. |
| `temporal_long_request_failure` | `PollWorkflowTaskQueue` | `DeadlineExceeded` | Often caused by proxies cutting long-poll connections. Normal in many network configurations. Worth investigating only if rate is unusually high or correlates with other issues. |
| `temporal_long_request_failure` | `PollActivityTaskQueue` | `DeadlineExceeded` | Same as above. |
| `temporal_request_failure` | `RequestCancelWorkflowExecution` | `NotFound` | Expected in fire-and-forget cancel patterns where the workflow may have already completed. |
| `temporal_request_failure` | `TerminateWorkflowExecution` | `NotFound` | Same — workflow already completed before the terminate arrived. |
| `temporal_request_failure` | `SignalWorkflowExecution` | `NotFound` | Common when signaling workflows that may have already completed. Depending on your application design this may be fully expected. |
| `temporal_request_failure` | `StartWorkflowExecution` | `AlreadyExists` | Expected if your application uses workflow ID deduplication intentionally. Only worth alerting on if unexpected. |
| `temporal_request_failure` | `RespondWorkflowTaskFailed` | `NotFound` | WFT token stale — already rescheduled. Expected occasionally under load. |
| `temporal_request_failure` | Any | `Unavailable` / `Internal` (intermittent, low rate) | Transient server issues during restarts or scaling. Expected to self-resolve. Alert only if sustained or high rate. |

---

## Diagnostic Tables

These tables are the core troubleshooting reference. Each row maps an operation and a gRPC status code to what it means operationally — what triggered it, what to look for, and whether it warrants an alert.

An important distinction: the rate limit buckets and priority section covers **one specific failure class** — `ResourceExhausted` from server-side throttling. But `temporal_request_failure` and `temporal_long_request_failure` capture **every non-OK gRPC response** an operation can return. That includes throttling, yes, but also application-level errors (`NotFound`, `AlreadyExists`, `FailedPrecondition`), bad requests (`InvalidArgument`), auth failures (`PermissionDenied`, `Unauthenticated`), infrastructure problems (`Unavailable`), timeouts (`DeadlineExceeded` — which can be a client-side context timeout, a server-side processing timeout, or a proxy cutting a long-lived connection), and unexpected server faults (`Internal`). Each of these tells a different story depending on which operation it came from — a `DeadlineExceeded` on `QueryWorkflow` points to workers not making progress, whereas a `DeadlineExceeded` on `PollWorkflowTaskQueue` is often just a proxy cutting the long-poll connection and can be completely normal. The tables below capture that full picture per operation.

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
| `StartWorkflowExecution` | `ResourceExhausted` | Retried automatically by the SDK. High start rate is overwhelming the namespace quota — check `frontend.namespaceRPS` and `frontend.globalNamespaceRPS`. Can also be returned if the workflow input payload exceeds the 4 MB gRPC message size limit — in that case this is not a throttling issue but a payload size issue, check payload sizes if RPS metrics look healthy. Additionally, if `history.enableWorkflowIdReuseStartTimeValidation` is set to `true`, `ResourceExhausted` can be returned specifically for this throttle when the same workflow ID is being started very rapidly. In this case check your server-side Resource Exhausted with Cause panel and look for `BusyWorkflow` as the cause — this indicates the server is throttling rapid re-starts of the same workflow ID, which is expected behavior when this config is enabled. Intermittent occurrences may not result in actual loss if SDK retries succeed within ~60s. If seen continuously or approaching the retry window, alert. Always log workflow ID and input payload on final failure to enable backfill. |
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
| `SignalWithStartWorkflowExecution` | `ResourceExhausted` | Retried automatically by the SDK. Intermittent occurrences may not result in actual loss if retries succeed within the retry window (~60s). If seen continuously or at high rate, alert. Log workflow ID and signal payload on final failure to enable backfill. |
| `SignalWithStartWorkflowExecution` | `Unavailable` / `Internal` | Both retried automatically by the SDK. Intermittent occurrences can be transient server issues — no immediate alert needed if retries succeed. If seen continuously or approaching the SDK retry window (~60s), alert. Log workflow ID and payload on final failure. |
| `ExecuteMultiOperation` | `ResourceExhausted` | Retried automatically by the SDK. Intermittent occurrences may not result in actual loss if retries succeed within ~60s. If seen continuously or at high rate, alert. Additionally, if `history.enableWorkflowIdReuseStartTimeValidation` is set to `true`, `ResourceExhausted` can be returned when the same workflow ID is being used very rapidly — check the Resource Exhausted with Cause panel on your server dashboard for `BusyWorkflow` as the cause. Log workflow ID and payload on final failure to enable backfill. |
| `ExecuteMultiOperation` | `InvalidArgument` | One or more of the contained operations has a malformed request. |
| `ExecuteMultiOperation` | `AlreadyExists` | A `StartWorkflow` within the multi-operation found a conflicting running workflow. |
| `ExecuteMultiOperation` | `NotFound` | Namespace or workflow not found for one of the contained operations. |
| `ExecuteMultiOperation` | `Unavailable` / `Internal` | Both retried automatically by the SDK. Intermittent occurrences can be transient server issues — no immediate alert needed if retries succeed. If seen continuously or approaching the SDK retry window (~60s), alert. Log workflow ID and payload on final failure. |
| `UpdateWorkflowExecution` | `NotFound` | Workflow not found — completed, terminated, or exceeded retention. |
| `UpdateWorkflowExecution` | `ResourceExhausted` | Retried automatically by the SDK. Also subject to concurrent long-running request limit (`frontend.namespaceCount`) since this blocks until a WFT completes. Intermittent occurrences may not result in actual loss if retries succeed within the retry window (~60s). If seen continuously or at high rate, alert. Log workflow ID and update details on final failure. |
| `UpdateWorkflowExecution` | `DeadlineExceeded` | A timeout occurred while waiting for the WFT to complete. Causes include: (1) server-side DB timeouts, check persistence metrics; (2) worker or client pod restart, look for `ShutdownWorker`, `DescribeNamespace`, or `GetSystemInfo` around the same time; (3) workers struggling — non-determinism issues or generic task failures preventing WFT completion; (4) gRPC proxy timeout. Check `workflow_task_schedule_to_start_latency` and worker health. |
| `UpdateWorkflowExecution` | `InvalidArgument` | Bad update name, oversized payload, or too many pending updates. |
| `UpdateWorkflowExecution` | `FailedPrecondition` | Workflow is not in a state that can accept updates (e.g. already completed or paused). |
| `UpdateWorkflowExecution` | `Unavailable` / `Internal` | Both retried automatically by the SDK. Intermittent occurrences can be transient server issues — no immediate alert needed if retries succeed. If seen continuously or approaching the SDK retry window (~60s), alert. Log workflow ID and update details on final failure. |
| `RequestCancelWorkflowExecution` | `NotFound` | Workflow not found — already completed, terminated, or exceeded retention. Often benign in fire-and-forget cancel patterns. |
| `RequestCancelWorkflowExecution` | `ResourceExhausted` | Retried automatically by the SDK. Intermittent occurrences may not result in actual cancellation loss if retries succeed within ~60s. If seen continuously or at high rate, alert. |
| `RequestCancelWorkflowExecution` | `FailedPrecondition` | Workflow already in a terminal state. |
| `RequestCancelWorkflowExecution` | `PermissionDenied` | Auth failure. |
| `RequestCancelWorkflowExecution` | `Unavailable` | Typically related to server issues — frontend unreachable or other server-side errors. Can be intermittent during server scaling or restarts, in which case it is generally non-critical. If seen persistently, alert on this and check server health. |
| `RequestCancelWorkflowExecution` | `Internal` | Typically unexpected server-side issues. Can be intermittent during things like server restarts or persistence DB issues. If seen at a higher rate or over a longer duration, check server health. |
| `TerminateWorkflowExecution` | `NotFound` | Workflow not found. Already completed or exceeded retention. |
| `TerminateWorkflowExecution` | `ResourceExhausted` | Retried automatically by the SDK. Intermittent occurrences may not result in actual termination loss if retries succeed within ~60s. If seen continuously or at high rate, alert. |
| `TerminateWorkflowExecution` | `FailedPrecondition` | Workflow already in terminal state. |
| `TerminateWorkflowExecution` | `PermissionDenied` | Auth failure. |
| `TerminateWorkflowExecution` | `Unavailable` | Typically related to server issues — frontend unreachable or other server-side errors. Can be intermittent during server scaling or restarts, in which case it is generally non-critical. If seen persistently, alert on this and check server health. |
| `TerminateWorkflowExecution` | `Internal` | Typically unexpected server-side issues. Can be intermittent during things like server restarts or persistence DB issues. If seen at a higher rate or over a longer duration, check server health. |
| `ResetWorkflowExecution` | `NotFound` | Workflow or the specified reset point event not found. |
| `ResetWorkflowExecution` | `ResourceExhausted` | Retried automatically by the SDK. Intermittent occurrences may not result in actual loss if retries succeed within ~60s. If seen continuously or at high rate, alert. |
| `ResetWorkflowExecution` | `InvalidArgument` | Bad reset point — event ID out of range, or reset to a non-resettable event type. |
| `ResetWorkflowExecution` | `FailedPrecondition` | Workflow in a state that cannot be reset (e.g. still running with pending activities). |
| `ResetWorkflowExecution` | `Unavailable` | Typically related to server issues — frontend unreachable or other server-side errors. Can be intermittent during server scaling or restarts, in which case it is generally non-critical. If seen persistently, alert on this and check server health. |
| `ResetWorkflowExecution` | `Internal` | Typically unexpected server-side issues. Can be intermittent during things like server restarts or persistence DB issues. If seen at a higher rate or over a longer duration, check server health. |
| `DeleteWorkflowExecution` | `NotFound` | Workflow not found. |
| `DeleteWorkflowExecution` | `ResourceExhausted` | Retried automatically by the SDK. Intermittent occurrences may not result in actual loss if retries succeed within ~60s. If seen continuously or at high rate, alert. |
| `DeleteWorkflowExecution` | `FailedPrecondition` | Workflow is still running. Must cancel or terminate before deleting. |
| `DeleteWorkflowExecution` | `Unavailable` | Typically related to server issues — frontend unreachable or other server-side errors. Can be intermittent during server scaling or restarts, in which case it is generally non-critical. If seen persistently, alert on this and check server health. |
| `DeleteWorkflowExecution` | `Internal` | Typically unexpected server-side issues. Can be intermittent during things like server restarts or persistence DB issues. If seen at a higher rate or over a longer duration, check server health. |
| `QueryWorkflow` | `NotFound` | Workflow not found. |
| `QueryWorkflow` | `ResourceExhausted` | Retried automatically by the SDK. Also subject to concurrent long-running request limit since query blocks until a WFT completes. Intermittent occurrences may not result in query loss if retries succeed within ~60s. If seen continuously or at high rate, alert. |
| `QueryWorkflow` | `DeadlineExceeded` | A timeout occurred while waiting for the query WFT to complete. Causes include: (1) server-side DB timeouts, check persistence metrics; (2) worker or client pod restart, look for `ShutdownWorker`, `DescribeNamespace`, or `GetSystemInfo` around the same time; (3) workers struggling — non-determinism issues, generic task failures, or workers simply not keeping up with the WFT backlog; (4) gRPC proxy timeout. Check `workflow_task_schedule_to_start_latency` and worker health. |
| `QueryWorkflow` | `InvalidArgument` | Unknown query type or bad query arguments. |
| `QueryWorkflow` | `FailedPrecondition` | Workflow is in a state where queries are not supported (e.g. workflow completed and query handler state is unavailable). |
| `QueryWorkflow` | `Unavailable` | Typically related to server issues — frontend unreachable or other server-side errors. Can be intermittent during server scaling or restarts, in which case it is generally non-critical. If seen persistently, alert on this and check server health. |
| `QueryWorkflow` | `Internal` | Typically unexpected server-side issues. Can be intermittent during things like server restarts or persistence DB issues. If seen at a higher rate or over a longer duration, check server health. |
| `DescribeWorkflowExecution` | `NotFound` | Workflow not found — completed and exceeded retention, wrong ID, or wrong namespace. Very common when polling for workflow status after retention period. |
| `DescribeWorkflowExecution` | `ResourceExhausted` | Retried automatically by the SDK. Frequent polling of `DescribeWorkflowExecution` in tight loops is a common cause. Intermittent occurrences are non-critical if retries succeed. If seen continuously, consider switching to `GetWorkflowExecutionHistory` with `wait_new_event=true` as a more efficient pattern, and increase `frontend.namespaceRPS` if needed. |
| `DescribeWorkflowExecution` | `Unavailable` | Typically related to server issues — frontend unreachable or other server-side errors. Can be intermittent during server scaling or restarts, in which case it is generally non-critical. If seen persistently, alert on this and check server health. |
| `DescribeWorkflowExecution` | `Internal` | Typically unexpected server-side issues. Can be intermittent during things like server restarts or persistence DB issues. If seen at a higher rate or over a longer duration, check server health. |
| `GetWorkflowExecutionHistory` (non-long-poll) | `NotFound` | Workflow not found — exceeded retention or wrong ID. |
| `GetWorkflowExecutionHistory` (non-long-poll) | `ResourceExhausted` | Retried automatically by the SDK. History fetch has relatively high priority (P2) because it is required for replay. Intermittent occurrences are non-critical if retries succeed. Sustained exhaustion may indicate too many concurrent replays or SDK retries amplifying load. |
| `GetWorkflowExecutionHistory` (non-long-poll) | `DeadlineExceeded` | A timeout occurred fetching history. Causes include: (1) server-side DB timeouts — history can be large and slow to read, check persistence metrics; (2) worker or client pod restart, look for `ShutdownWorker`, `DescribeNamespace`, or `GetSystemInfo` around the same time; (3) gRPC proxy timeout. |
| `GetWorkflowExecutionHistory` (non-long-poll) | `InvalidArgument` | Bad page token or invalid history fetch parameters. |
| `GetWorkflowExecutionHistory` (non-long-poll) | `Unavailable` | Typically related to server issues — frontend unreachable or other server-side errors. Can be intermittent during server scaling or restarts, in which case it is generally non-critical. If seen persistently, alert on this and check server health. |
| `GetWorkflowExecutionHistory` (non-long-poll) | `Internal` | Typically unexpected server-side issues. Can be intermittent during things like server restarts or persistence DB issues. If seen at a higher rate or over a longer duration, check server health. |
| `GetWorkflowExecutionHistoryReverse` | `NotFound` | Workflow not found. |
| `GetWorkflowExecutionHistoryReverse` | `ResourceExhausted` | Retried automatically by the SDK. Intermittent occurrences are non-critical if retries succeed. |
| `GetWorkflowExecutionHistoryReverse` | `InvalidArgument` | Bad page token or parameters. |
| `GetWorkflowExecutionHistoryReverse` | `Unavailable` | Typically related to server issues — frontend unreachable or other server-side errors. Can be intermittent during server scaling or restarts, in which case it is generally non-critical. If seen persistently, alert on this and check server health. |
| `GetWorkflowExecutionHistoryReverse` | `Internal` | Typically unexpected server-side issues. Can be intermittent during things like server restarts or persistence DB issues. If seen at a higher rate or over a longer duration, check server health. |
| `RespondWorkflowTaskCompleted` | `NotFound` | Alert immediately unless explainable by one of the known expected causes. Has several distinct causes: (1) **WFT timeout** — the worker responded after the `WorkflowTaskTimeout` (default 10s) expired and the server already rescheduled the task; indicates workers are severely overloaded or WFT processing is far too slow relative to the timeout; (2) **Workflow execution completed** — the worker responded after the execution was already terminated externally or otherwise completed; (3) **WorkflowRunTimeout or WorkflowExecutionTimeout firing** — if these are configured and expiring at a high rate, executions complete before workers can respond. Before treating as an emergency, check whether the rate correlates with known terminations or expiring run/execution timeouts. If neither explains it, check `workflow_task_schedule_to_start_latency`, WFT processing times, and worker resource utilization. |
| `RespondWorkflowTaskCompleted` | `ResourceExhausted` | Retried automatically by the SDK, but the retry window is very tight — default `WorkflowTaskTimeout` is 10s. Can be caused by RPS throttling (`RpsLimit`) or server overload (`SystemOverload`) — check the Resource Exhausted with Cause panel on your server dashboard. Can also be caused by the WFT completion response exceeding the 4 MB gRPC message size limit. Sustained throttling causes WFTs to time out and retry, amplifying load in a feedback loop. If your workflows use **local activities**, WFT timeouts from this throttling will trigger re-execution of those local activities — a serious concern if they are not idempotent. Do not alert on isolated intermittent occurrences, but alert if seen for longer than ~5-6 seconds or at a high rate given the 10s timeout window. |
| `RespondWorkflowTaskCompleted` | `InvalidArgument` | Commands are invalid — non-determinism detected, unsupported command type, malformed payloads, or too many pending activities/timers/child workflows/signals (server enforces limits). |
| `RespondWorkflowTaskCompleted` | `FailedPrecondition` | Workflow state has changed in a way that invalidates the commands (e.g. workflow was reset). |
| `RespondWorkflowTaskCompleted` | `Unavailable` | Typically related to server issues — frontend unreachable or other server-side errors. Can be intermittent during server scaling or restarts, in which case it is generally non-critical. If seen persistently, alert on this and check server health. |
| `RespondWorkflowTaskCompleted` | `Internal` | Typically unexpected server-side issues. Can be intermittent during things like server restarts or persistence DB issues. If seen at a higher rate or over a longer duration, check server health. |
| `RespondWorkflowTaskFailed` | `NotFound` | Task token stale — WFT already timed out and rescheduled. Same underlying causes as `RespondWorkflowTaskCompleted` `NotFound`: WFT timeout, execution completed/terminated, run/execution timeout fired, high worker resource utilization, or high server-side persistence latencies. Check the same signals. |
| `RespondWorkflowTaskFailed` | `ResourceExhausted` | Execution bucket RPS exhausted at Priority 3. Lower priority than completion — under extreme load, failure reports may be dropped, causing WFT to time out instead. |
| `RespondWorkflowTaskFailed` | `InvalidArgument` | Malformed failure details. |
| `RespondWorkflowTaskFailed` | `Unavailable` | Typically related to server issues — frontend unreachable or other server-side errors. Can be intermittent during server scaling or restarts, in which case it is generally non-critical. If seen persistently, alert on this and check server health. |
| `RespondWorkflowTaskFailed` | `Internal` | Typically unexpected server-side issues. Can be intermittent during things like server restarts or persistence DB issues. If seen at a higher rate or over a longer duration, check server health. |
| `RespondActivityTaskCompleted` | `NotFound` | Activity task token is stale — has several distinct causes: (1) **StartToClose or ScheduleToClose timeout expired** — the activity took longer than its configured timeout and the server already marked it as timed out before the worker could respond; (2) **Workflow execution terminated** — the execution was terminated externally while the activity was running; (3) **Workflow execution timed out** — a `WorkflowRunTimeout` or `WorkflowExecutionTimeout` fired and completed the execution; (4) **Workflow canceled without waiting for activity cancellation** — if the workflow cancellation logic does not wait for the activity to fully cancel before completing, the worker may try to respond to an activity whose execution context is already gone. Check which cause applies by correlating with activity timeout metrics, termination patterns, or workflow cancellation logic. |
| `RespondActivityTaskCompleted` | `ResourceExhausted` | Retried automatically by the SDK. Can be caused by RPS throttling — check `frontend.namespaceRPS` and server-side Resource Exhausted with Cause panel. Can also be caused by the activity result payload exceeding the 4 MB gRPC message size limit — in this case retrying will never succeed and the SDK will eventually stop retrying. If RPS metrics look healthy, check whether the activity result payload is oversized. |
| `RespondActivityTaskCompleted` | `InvalidArgument` | Activity result payload exceeds the Temporal payload size limit (between 2 MB and 4 MB). Note the distinction: payloads between 2-4 MB return `InvalidArgument`, while payloads over 4 MB return `ResourceExhausted`. In both cases retrying will not help — the payload needs to be reduced. Consider using Temporal's blob storage pattern to store large results externally and pass a reference instead. |
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
| `RecordActivityTaskHeartbeat` | `NotFound` | Activity task token is stale — has two distinct causes with very different severity: (1) **Expected** — execution was terminated, timed out, or canceled; in this case the worker should detect the `NotFound` and stop the activity gracefully; (2) **Concerning** — high worker CPU or memory utilization slowing down processing to the point where the heartbeat call itself is delayed long enough for the heartbeat timeout to fire; in this case the activity will be retried or fail. Correlate with worker resource metrics to distinguish between the two. |
| `RecordActivityTaskHeartbeat` | `ResourceExhausted` | Alert on this. Can be caused by RPS throttling — check `frontend.namespaceRPS` and the Resource Exhausted with Cause panel on your server dashboard. Can also be caused by the heartbeat details payload exceeding the 4 MB gRPC message size limit. In either case this is serious — a failed heartbeat call means the server does not receive the expected heartbeat, which will trigger the heartbeat timeout and cause the activity to be retried or fail even though the worker is still running it. Reduce heartbeat payload size or increase RPS limits as appropriate. |
| `RecordActivityTaskHeartbeat` | `InvalidArgument` | Heartbeat details payload exceeds the Temporal payload size limit (between 2 MB and 4 MB). Same size distinction as activity completions — 2-4 MB returns `InvalidArgument`, over 4 MB returns `ResourceExhausted`. Both are serious for the same reason as `ResourceExhausted` — a failed heartbeat will trigger the heartbeat timeout. Reduce the amount of data passed in heartbeat details; heartbeats should carry minimal state, not large payloads. |
| `RecordActivityTaskHeartbeat` | `Unavailable` | Typically related to server issues — frontend unreachable or other server-side errors. Can be intermittent during server scaling or restarts, in which case it is generally non-critical. If seen persistently, alert on this and check server health. |
| `RecordActivityTaskHeartbeat` | `Internal` | Typically unexpected server-side issues. Can be intermittent during things like server restarts or persistence DB issues. If seen at a higher rate or over a longer duration, check server health. |
| `RespondQueryTaskCompleted` | `NotFound` | Query task no longer exists — most commonly the workflow execution was terminated or timed out while the worker was processing the query. Can also happen if the client that issued the `QueryWorkflow` call disconnected or its context timed out before the worker responded, causing the server to discard the query task. |
| `RespondQueryTaskCompleted` | `ResourceExhausted` | Server-side throttling. Check the Resource Exhausted with Cause panel on your server dashboard to identify whether it is `RpsLimit`, `ConcurrentLimit`, or `SystemOverload`. |

| `RespondQueryTaskCompleted` | `Unavailable` | Typically related to server issues — frontend unreachable or other server-side errors. Can be intermittent during server scaling or restarts, in which case it is generally non-critical. If seen persistently, alert on this and check server health. |
| `RespondQueryTaskCompleted` | `Internal` | Typically unexpected server-side issues. Can be intermittent during things like server restarts or persistence DB issues. If seen at a higher rate or over a longer duration, check server health. |
| `DescribeTaskQueue` | `NotFound` | Namespace or task queue does not exist. Check that the namespace and task queue name in the request are correct. |
| `DescribeTaskQueue` | `ResourceExhausted` | **Visibility bucket** RPS exhausted at Priority 1 — `DescribeTaskQueue` counts against the visibility quota, not the execution quota. The default visibility RPS is only 10 per instance, so frequent polling of task queue status from monitoring dashboards or tooling can quickly exhaust this quota and cause `ResourceExhausted` on unrelated visibility list calls. Check the Resource Exhausted with Cause panel and increase `frontend.namespaceRPS.visibility` if needed. |
| `DescribeTaskQueue` | `Unavailable` | Typically related to server issues — frontend unreachable or other server-side errors. Can be intermittent during server scaling or restarts, in which case it is generally non-critical. If seen persistently, alert on this and check server health. |
| `DescribeTaskQueue` | `Internal` | Typically unexpected server-side issues. Can be intermittent during things like server restarts or persistence DB issues. If seen at a higher rate or over a longer duration, check server health. |
| `ListWorkflowExecutions` | `ResourceExhausted` | Visibility bucket RPS exhausted at Priority 1. Default is 10 RPS per instance — the most commonly hit limit in production for teams with monitoring or user-facing workflow list APIs. Check the Resource Exhausted with Cause panel on your server dashboard and increase `frontend.namespaceRPS.visibility` if needed. |
| `ListWorkflowExecutions` | `Unavailable` | Typically related to server issues — frontend unreachable, or the visibility store (Elasticsearch/SQL) unavailable. Can be intermittent during server scaling or restarts, in which case it is generally non-critical. If seen persistently, alert on this and check server health and visibility store health. |
| `ListWorkflowExecutions` | `Internal` | Typically unexpected server-side issues. Can be intermittent during things like server restarts or persistence DB issues. If seen at a higher rate or over a longer duration, check server health. |
| `CountWorkflowExecutions` | `ResourceExhausted` | Visibility bucket RPS exhausted at Priority 1. Count queries can be expensive — frequent calls amplify visibility store load. Check the Resource Exhausted with Cause panel and increase `frontend.namespaceRPS.visibility` if needed. |
| `CountWorkflowExecutions` | `Unavailable` | Typically related to server issues or visibility store unavailable. Can be intermittent during scaling or restarts. If seen persistently, alert and check visibility store health. |
| `CountWorkflowExecutions` | `Internal` | Typically unexpected server-side issues. Can be intermittent during things like server restarts or persistence DB issues. If seen at a higher rate or over a longer duration, check server health. |
| `ListOpenWorkflowExecutions` | `ResourceExhausted` | Visibility bucket RPS exhausted at Priority 1. Check the Resource Exhausted with Cause panel and increase `frontend.namespaceRPS.visibility` if needed. |
| `ListOpenWorkflowExecutions` | `Unavailable` | Typically related to server issues or visibility store unavailable. Can be intermittent during scaling or restarts. If seen persistently, alert and check visibility store health. |
| `ListOpenWorkflowExecutions` | `Internal` | Typically unexpected server-side issues. Can be intermittent during things like server restarts or persistence DB issues. If seen at a higher rate or over a longer duration, check server health. |
| `ListClosedWorkflowExecutions` | `ResourceExhausted` | Visibility bucket RPS exhausted at Priority 1. Check the Resource Exhausted with Cause panel and increase `frontend.namespaceRPS.visibility` if needed. |
| `ListClosedWorkflowExecutions` | `Unavailable` | Typically related to server issues or visibility store unavailable. Can be intermittent during scaling or restarts. If seen persistently, alert and check visibility store health. |
| `ListClosedWorkflowExecutions` | `Internal` | Typically unexpected server-side issues. Can be intermittent during things like server restarts or persistence DB issues. If seen at a higher rate or over a longer duration, check server health. |
| `ScanWorkflowExecutions` | `ResourceExhausted` | Visibility bucket RPS exhausted at Priority 1. Scans are expensive — they read large unordered sets from the visibility store. Check the Resource Exhausted with Cause panel and increase `frontend.namespaceRPS.visibility` if needed. |
| `ScanWorkflowExecutions` | `Unavailable` | Typically related to server issues or visibility store unavailable. Can be intermittent during scaling or restarts. If seen persistently, alert and check visibility store health. |
| `ScanWorkflowExecutions` | `Internal` | Typically unexpected server-side issues. Can be intermittent during things like server restarts or persistence DB issues. If seen at a higher rate or over a longer duration, check server health. |
| `ListArchivedWorkflowExecutions` | `ResourceExhausted` | Visibility bucket RPS exhausted. Check the Resource Exhausted with Cause panel and increase `frontend.namespaceRPS.visibility` if needed. |
| `ListArchivedWorkflowExecutions` | `Unavailable` | Typically related to server issues or archival store unavailable. Can be intermittent during scaling or restarts. If seen persistently, alert and check archival store health. |
| `ListArchivedWorkflowExecutions` | `Internal` | Typically unexpected server-side issues. Can be intermittent during things like server restarts or persistence DB issues. If seen at a higher rate or over a longer duration, check server health. |
| `ListActivityExecutions` | `ResourceExhausted` | Visibility bucket RPS exhausted at Priority 1. Check the Resource Exhausted with Cause panel and increase `frontend.namespaceRPS.visibility` if needed. |
| `ListActivityExecutions` | `Unavailable` | Typically related to server issues or visibility store unavailable. Can be intermittent during scaling or restarts. If seen persistently, alert and check visibility store health. |
| `ListActivityExecutions` | `Internal` | Typically unexpected server-side issues. Can be intermittent during things like server restarts or persistence DB issues. If seen at a higher rate or over a longer duration, check server health. |
| `CountActivityExecutions` | `ResourceExhausted` | Visibility bucket RPS exhausted at Priority 1. Check the Resource Exhausted with Cause panel and increase `frontend.namespaceRPS.visibility` if needed. |
| `CountActivityExecutions` | `Unavailable` | Typically related to server issues or visibility store unavailable. Can be intermittent during scaling or restarts. If seen persistently, alert and check visibility store health. |
| `CountActivityExecutions` | `Internal` | Typically unexpected server-side issues. Can be intermittent during things like server restarts or persistence DB issues. If seen at a higher rate or over a longer duration, check server health. |
| `ListSchedules` | `ResourceExhausted` | Visibility bucket RPS exhausted at Priority 1. Check the Resource Exhausted with Cause panel and increase `frontend.namespaceRPS.visibility` if needed. |
| `ListSchedules` | `Unavailable` | Typically related to server issues or visibility store unavailable. Can be intermittent during scaling or restarts. If seen persistently, alert and check visibility store health. |
| `ListSchedules` | `Internal` | Typically unexpected server-side issues. Can be intermittent during things like server restarts or persistence DB issues. If seen at a higher rate or over a longer duration, check server health. |
| `CreateSchedule` | `AlreadyExists` | Schedule with this ID already exists. |
| `CreateSchedule` | `ResourceExhausted` | Server-side throttling — schedules are executed by the worker service role internally, which has its own RPS limits for schedule operations. Check the Resource Exhausted with Cause panel on your server dashboard. |
| `CreateSchedule` | `Unavailable` | Typically related to server issues — frontend unreachable or other server-side errors. Can be intermittent during server scaling or restarts, in which case it is generally non-critical. If seen persistently, alert on this and check server health. |
| `CreateSchedule` | `Internal` | Typically unexpected server-side issues. Can be intermittent during things like server restarts or persistence DB issues. If seen at a higher rate or over a longer duration, check server health. |
| `DescribeSchedule` | `NotFound` | Schedule does not exist — it was deleted or never created. |
| `DescribeSchedule` | `ResourceExhausted` | Server-side throttling. Check the Resource Exhausted with Cause panel on your server dashboard. |
| `DescribeSchedule` | `Unavailable` | Typically related to server issues — frontend unreachable or other server-side errors. Can be intermittent during server scaling or restarts, in which case it is generally non-critical. If seen persistently, alert on this and check server health. |
| `DescribeSchedule` | `Internal` | Typically unexpected server-side issues. Can be intermittent during things like server restarts or persistence DB issues. If seen at a higher rate or over a longer duration, check server health. |
| `UpdateSchedule` | `NotFound` | Schedule does not exist — it was deleted or never created. |
| `UpdateSchedule` | `ResourceExhausted` | Server-side throttling. Check the Resource Exhausted with Cause panel on your server dashboard. |
| `UpdateSchedule` | `Unavailable` | Typically related to server issues — frontend unreachable or other server-side errors. Can be intermittent during server scaling or restarts, in which case it is generally non-critical. If seen persistently, alert on this and check server health. |
| `UpdateSchedule` | `Internal` | Typically unexpected server-side issues. Can be intermittent during things like server restarts or persistence DB issues. If seen at a higher rate or over a longer duration, check server health. |
| `PatchSchedule` | `NotFound` | Schedule does not exist — it was deleted or never created. |
| `PatchSchedule` | `ResourceExhausted` | Server-side throttling. Check the Resource Exhausted with Cause panel on your server dashboard. |
| `PatchSchedule` | `Unavailable` | Typically related to server issues — frontend unreachable or other server-side errors. Can be intermittent during server scaling or restarts, in which case it is generally non-critical. If seen persistently, alert on this and check server health. |
| `PatchSchedule` | `Internal` | Typically unexpected server-side issues. Can be intermittent during things like server restarts or persistence DB issues. If seen at a higher rate or over a longer duration, check server health. |
| `DeleteSchedule` | `NotFound` | Schedule does not exist — it was deleted or never created. |
| `DeleteSchedule` | `ResourceExhausted` | Server-side throttling. Check the Resource Exhausted with Cause panel on your server dashboard. |
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

## Operation Reference Tables

These tables list all operations by rate limit bucket and priority. Refer here when you need to understand which bucket an operation belongs to or what priority it has under load.

### Execution Bucket Operations

#### Priority 0 — System / Informational

| Operation | Caller |
|-----------|--------|
| `GetClusterInfo` | Client / Worker |
| `GetSystemInfo` | Client / Worker |
| `GetSearchAttributes` | Client |
| `DescribeNamespace` | Client / Worker |
| `ListNamespaces` | Client |
| `DeprecateNamespace` | Client |

#### Priority 1 — External Event APIs + Task Progress

| Operation | Caller |
|-----------|--------|
| `StartWorkflowExecution` | Client |
| `SignalWorkflowExecution` | Client |
| `SignalWithStartWorkflowExecution` | Client |
| `UpdateWorkflowExecution` | Client |
| `ExecuteMultiOperation` | Client |
| `CreateSchedule` | Client |
| `StartBatchOperation` | Client |
| `StartActivityExecution` | Client |
| `RecordActivityTaskHeartbeat` | Worker |
| `RecordActivityTaskHeartbeatById` | Worker / App |
| `RespondActivityTaskCompleted` | Worker |
| `RespondWorkflowTaskCompleted` | Worker |
| `RespondQueryTaskCompleted` | Worker |
| `DispatchNexusTaskByNamespaceAndTaskQueue` | Nexus |
| `DispatchNexusTaskByEndpoint` | Nexus |
| `CompleteNexusOperation` | Nexus |

#### Priority 2 — State Change APIs

| Operation | Caller |
|-----------|--------|
| `RequestCancelWorkflowExecution` | Client |
| `TerminateWorkflowExecution` | Client |
| `ResetWorkflowExecution` | Client |
| `DeleteWorkflowExecution` | Client |
| `PauseWorkflowExecution` | Client |
| `UnpauseWorkflowExecution` | Client |
| `GetWorkflowExecutionHistory` (non-long-poll) | Client / Worker |
| `UpdateSchedule` | Client |
| `PatchSchedule` | Client |
| `DeleteSchedule` | Client |
| `StopBatchOperation` | Client |
| `PauseActivity` | Client |
| `UnpauseActivity` | Client |
| `ResetActivity` | Client |
| `UpdateActivityOptions` | Client |
| `RequestCancelActivityExecution` | Client |
| `TerminateActivityExecution` | Client |
| `DeleteActivityExecution` | Client |
| `UpdateWorkflowExecutionOptions` | Client |
| `SetWorkerDeploymentCurrentVersion` | Client |
| `SetWorkerDeploymentRampingVersion` | Client |
| `SetWorkerDeploymentManager` | Client |
| `DeleteWorkerDeployment` | Client |
| `DeleteWorkerDeploymentVersion` | Client |
| `UpdateWorkerDeploymentVersionMetadata` | Client |
| `UpdateTaskQueueConfig` | Worker |
| `CreateWorkflowRule` | Client |
| `DescribeWorkflowRule` | Client |
| `DeleteWorkflowRule` | Client |
| `ListWorkflowRules` | Client |
| `TriggerWorkflowRule` | Client |

#### Priority 3 — Status Querying + Failure/Cancel Reporting

| Operation | Caller |
|-----------|--------|
| `DescribeWorkflowExecution` | Client |
| `DescribeActivityExecution` | Client |
| `QueryWorkflow` | Client |
| `GetWorkerBuildIdCompatibility` | Worker / Client |
| `GetWorkerVersioningRules` | Worker / Client |
| `ListTaskQueuePartitions` | Worker |
| `DescribeSchedule` | Client |
| `ListScheduleMatchingTimes` | Client |
| `DescribeBatchOperation` | Client |
| `DescribeWorkerDeployment` | Client |
| `DescribeWorkerDeploymentVersion` | Client |
| `RespondActivityTaskFailed` | Worker |
| `RespondWorkflowTaskFailed` | Worker |

#### Priority 4 — Poll APIs and Other Low Priority

| Operation | Caller |
|-----------|--------|
| `PollWorkflowTaskQueue` | Worker |
| `PollActivityTaskQueue` | Worker |
| `PollNexusTaskQueue` | Worker |
| `PollWorkflowExecutionUpdate` | Client |
| `PollActivityExecution` | Client |
| `ResetStickyTaskQueue` | Worker |
| `GetWorkflowExecutionHistoryReverse` | Client |
| `RecordWorkerHeartbeat` | Worker |
| `FetchWorkerConfig` | Worker |
| `UpdateWorkerConfig` | Worker |
| `ShutdownWorker` | Worker |

#### Priority 5 — Lowest Priority

| Operation | Caller |
|-----------|--------|
| `GetWorkflowExecutionHistory` (long-poll / `wait_new_event=true`) | Worker / Client |
| `DescribeActivityExecution` (long-poll / `long_poll_token` set) | Client |
| OpenAPI v2/v3 endpoints | — |

---

### Visibility Bucket Operations

All at Priority 1 within the visibility bucket.

| Operation | Caller |
|-----------|--------|
| `ListWorkflowExecutions` | Client |
| `ListOpenWorkflowExecutions` | Client |
| `ListClosedWorkflowExecutions` | Client |
| `ListArchivedWorkflowExecutions` | Client |
| `ScanWorkflowExecutions` | Client |
| `CountWorkflowExecutions` | Client |
| `ListActivityExecutions` | Client |
| `CountActivityExecutions` | Client |
| `DescribeTaskQueue` | Client / Worker |
| `GetWorkerTaskReachability` | Worker / Client |
| `ListSchedules` | Client |
| `CountSchedules` | Client |
| `ListBatchOperations` | Client |
| `ListWorkerDeployments` | Client |
| `ListWorkers` | Client |
| `DescribeWorker` | Client |

---

### NamespaceReplicationInducing Bucket Operations

| Operation | Caller | Priority |
|-----------|--------|----------|
| `RegisterNamespace` | Admin | 1 |
| `UpdateNamespace` | Admin | 1 |
| `UpdateWorkerBuildIdCompatibility` | Worker / Admin | 2 |
| `UpdateWorkerVersioningRules` | Worker / Admin | 2 |

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
- [Temporal Server Dashboard](https://github.com/tsurdilo/my-temporal-dockercompose/blob/main/deployment/grafana/dashboards/temporal-server.json)