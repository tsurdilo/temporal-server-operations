# Temporal Core SDK Metrics

The Core SDK is a shared Rust library that powers the **TypeScript, Python, .NET, and Ruby** Temporal SDKs.
Metrics defined here are emitted by all of those SDKs identically.

> **Histogram unit:** Milliseconds by default. Can be configured to use seconds per-SDK.
> This differs from Java and Go SDKs which always use seconds.

## Table of Contents

- [gRPC Client Metrics](#grpc-client-metrics)
- [Worker Lifecycle Metrics](#worker-lifecycle-metrics)
- [Workflow Task Metrics](#workflow-task-metrics)
- [Workflow Execution Metrics](#workflow-execution-metrics)
- [Workflow Cache Metrics](#workflow-cache-metrics)
- [Activity Task Metrics](#activity-task-metrics)
- [Local Activity Metrics](#local-activity-metrics)
- [Nexus Task Metrics](#nexus-task-metrics)
- [Resource-Based Tuner Metrics](#resource-based-tuner-metrics)
- [Metric Tags Reference](#metric-tags-reference)
- [Notes](#notes)
- [Core SDK vs Java/Go SDK Differences](#core-sdk-vs-javago-sdk-differences)

---

**Sources:**
- [`crates/sdk-core/src/telemetry/metrics.rs`](https://github.com/temporalio/sdk-core/blob/master/crates/sdk-core/src/telemetry/metrics.rs) — Worker metrics
- [`crates/client/src/metrics.rs`](https://github.com/temporalio/sdk-core/blob/master/crates/client/src/metrics.rs) — gRPC client metrics

---

## gRPC Client Metrics

Emitted by `crates/client/src/metrics.rs` on every outbound RPC call.

> **Note:** The Core SDK client metrics are **not** prefixed with `temporal_` by the Core library itself — the prefix is applied by each language SDK's metrics bridge. In practice all Core-based SDKs emit these with the `temporal_` prefix, but some older TypeScript SDK versions had a bug where the prefix was missing ([sdk-typescript#1353](https://github.com/temporalio/sdk-typescript/issues/1353)).

| Metric | Type | Description |
|---|---|---|
| `temporal_request` | Counter | Successful short (non-poll) gRPC requests |
| `temporal_request_failure` | Counter | Failed short gRPC requests |
| `temporal_request_latency` | Histogram | Latency of short gRPC requests |
| `temporal_long_request` | Counter | Successful long-poll gRPC requests (`Poll*` calls) |
| `temporal_long_request_failure` | Counter | Failed long-poll requests |
| `temporal_long_request_latency` | Histogram | Latency of long-poll requests |

**Tags:** `operation` (RPC method name), `namespace`, `task_queue`, `status_code` (gRPC status code on failure, SCREAMING_SNAKE_CASE e.g. `FAILED_PRECONDITION`)

---

## Worker Lifecycle Metrics

| Metric | Type | Description |
|---|---|---|
| `temporal_worker_start` | Counter | Emitted once when a worker is registered/started |
| `temporal_worker_task_slots_available` | Gauge | Number of execution slots currently available |
| `temporal_worker_task_slots_used` | Gauge | Number of execution slots currently in use |
| `temporal_num_pollers` | Gauge | Current number of active pollers |

**Tags:** `worker_type`, `namespace`, `task_queue`

`temporal_num_pollers` also carries a `poller_type` tag.

---

## Workflow Task Metrics

| Metric | Type | Description |
|---|---|---|
| `temporal_workflow_task_queue_poll_empty` | Counter | Poll completed with no task returned (long-poll timeout) |
| `temporal_workflow_task_queue_poll_succeed` | Counter | Poll returned a workflow task |
| `temporal_workflow_task_schedule_to_start_latency` | Histogram | Time from task scheduled on server to worker picking it up |
| `temporal_workflow_task_execution_latency` | Histogram | Time to execute a single workflow task |
| `temporal_workflow_task_replay_latency` | Histogram | Time spent replaying history to catch up during a workflow task |
| `temporal_workflow_task_execution_failed` | Counter | Workflow task execution failed |

**Tags:** `workflow_type`, `namespace`, `task_queue`

**`temporal_workflow_task_execution_failed` failure reason tag values:**

| `failure_reason` value | Meaning |
|---|---|
| `NonDeterminismError` | Task failed due to a non-determinism error |
| `WorkflowError` | Task failed for any other reason |

---

## Workflow Execution Metrics

Emitted when a workflow execution reaches a terminal state.

| Metric | Type | Description |
|---|---|---|
| `temporal_workflow_completed` | Counter | Workflow execution completed successfully |
| `temporal_workflow_canceled` | Counter | Workflow execution ended by cancellation |
| `temporal_workflow_failed` | Counter | Workflow execution failed |
| `temporal_workflow_continue_as_new` | Counter | Workflow execution ended with continue-as-new |
| `temporal_workflow_endtoend_latency` | Histogram | Total time from schedule to close for a single workflow run |

**Tags:** `workflow_type`, `namespace`, `task_queue`

---

## Workflow Cache Metrics

Emitted by the sticky cache (workflow execution cache).

| Metric | Type | Description |
|---|---|---|
| `temporal_sticky_cache_hit` | Counter | Workflow task found a matching cached execution |
| `temporal_sticky_cache_miss` | Counter | Workflow task found no cached execution (full replay required) |
| `temporal_sticky_cache_size` | Gauge | Current number of cached workflow executions |
| `temporal_sticky_cache_total_forced_eviction` | Counter | Executions forcibly evicted from cache |

---

## Activity Task Metrics

| Metric | Type | Description |
|---|---|---|
| `temporal_activity_poll_no_task` | Counter | Activity poll completed with no task returned |
| `temporal_activity_task_received` | Counter | An activity task was received and is being processed |
| `temporal_activity_schedule_to_start_latency` | Histogram | Time from activity task scheduled to worker picking it up |
| `temporal_activity_execution_latency` | Histogram | Time from when Core dispatched the task to lang completing it (success or failure) |
| `temporal_activity_succeed_endtoend_latency` | Histogram | Total time from first schedule to successful completion (across all retries) |
| `temporal_activity_execution_failed` | Counter | Activity execution failed |
| `temporal_unregistered_activity_invocation` | Counter | Activity type dispatched to this worker is not registered |

**Tags:** `activity_type`, `namespace`, `task_queue`

> **`temporal_activity_task_received`** is Core SDK specific — it fires when Core hands an activity task to the language layer. Not present in Java or Go SDKs.

---

## Local Activity Metrics

| Metric | Type | Description |
|---|---|---|
| `temporal_local_activity_execution_latency` | Histogram | Time to execute a single local activity attempt |
| `temporal_local_activity_succeed_endtoend_latency` | Histogram | Total time from schedule to successful completion |
| `temporal_local_activity_execution_failed` | Counter | Local activity execution failed |
| `temporal_local_activity_execution_cancelled` | Counter | Local activity execution cancelled |
| `temporal_local_activity_total` | Counter | Total count of local activity executions attempted |

**Tags:** `activity_type`, `namespace`, `task_queue`

---

## Nexus Task Metrics

| Metric | Type | Description |
|---|---|---|
| `temporal_nexus_poll_no_task` | Counter | Nexus task poll returned empty |
| `temporal_nexus_schedule_to_start_latency` | Histogram | Time from Nexus task scheduled (when request hit Frontend) to worker picking it up |
| `temporal_nexus_task_execution_latency` | Histogram | Time to execute a Nexus task |
| `temporal_nexus_task_endtoend_latency` | Histogram | Total end-to-end time for a Nexus task |
| `temporal_nexus_task_execution_failed` | Counter | Nexus task execution failed |

**Tags:** `namespace`, `task_queue`, `nexus_service`, `nexus_operation`

**`temporal_nexus_task_execution_failed` failure reason tag values:**

| `failure_reason` value | Meaning |
|---|---|
| `internal_sdk_error` | Unexpected internal error within the SDK — indicates a bug |
| `handler_error_{TYPE}` | User handler returned a predefined Nexus error type. If handler returned an unexpected error, `TYPE` is `INTERNAL` |

---

## Resource-Based Tuner Metrics

Emitted only when the resource-based slot supplier (auto-tuner) is enabled.

| Metric | Type | Description |
|---|---|---|
| `temporal_worker_task_slots_available` | Gauge | Available slots (also in worker lifecycle section; here tagged with resource tuner context) |
| `temporal_worker_task_slots_used` | Gauge | Used slots (same as above) |

The resource-based tuner exposes CPU and memory utilization through the metrics system so slot counts can be observed alongside resource pressure. The raw CPU/memory values are not emitted as separate named metrics but are available via the tuner's internal state.

---

## Metric Tags Reference

| Tag | Description | Example values |
|---|---|---|
| `namespace` | Temporal namespace the worker is bound to | `default`, `my-namespace` |
| `task_queue` | Task queue being polled | `my-task-queue` |
| `workflow_type` | Name of the workflow function | `MoneyTransferWorkflow` |
| `activity_type` | Name of the activity function | `WithdrawActivity` |
| `worker_type` | Type of worker emitting the metric | `WorkflowWorker`, `ActivityWorker`, `LocalActivityWorker`, `NexusWorker` |
| `poller_type` | Type of poller | `workflow_task`, `activity_task`, `nexus_task`, `sticky_workflow_task` |
| `operation` | gRPC RPC method name | `PollWorkflowTaskQueue`, `RespondWorkflowTaskCompleted` |
| `failure_reason` | Reason for task failure | `NonDeterminismError`, `WorkflowError`, `handler_error_INTERNAL` |
| `nexus_service` | Nexus service name | — |
| `nexus_operation` | Nexus operation name | — |
| `status_code` | gRPC status code on request failure (SCREAMING_SNAKE_CASE) | `FAILED_PRECONDITION`, `UNAVAILABLE`, `DEADLINE_EXCEEDED` |

---

## Core SDK vs Java/Go SDK Differences

| Area | Core SDK (TS/Python/.NET/Ruby) | Java SDK | Go SDK |
|---|---|---|---|
| Histogram unit | Milliseconds (configurable to seconds) | Seconds | Seconds |
| `activity_task_received` counter | Present | Not present | Not present |
| `activity_execution_cancelled` counter | Not present | Present | Not present |
| `workflow_task_execution_total_latency` | Not present | Present | Not present |
| `workflow_task_no_completion` counter | Not present | Present | Present |
| `local_activity_total` counter | Present | Not present | Present (deprecated alias) |
| Workflow canceled spelling | `temporal_workflow_canceled` (one `l`) | `temporal_workflow_cancelled` (two `l`s) | `temporal_workflow_canceled` (one `l`) |
| Nexus execution metric naming | `temporal_nexus_task_execution_latency` / `temporal_nexus_task_execution_failed` | `temporal_nexus_execution_latency` / `temporal_nexus_execution_failed` | `temporal_nexus_execution_latency` / `temporal_nexus_execution_failed` |
| `status_code` tag format | SCREAMING_SNAKE_CASE e.g. `FAILED_PRECONDITION` | SCREAMING_SNAKE_CASE e.g. `FAILED_PRECONDITION` | PascalCase e.g. `FailedPrecondition` |
| `workflow_active_thread_count` gauge | Not present | Present (JVM threads) | Present (goroutines) |
| Local activity `worker_type` value | Not a separate value | `LocalActivityWorker` | `LocalActivityWorker` |

---

## Notes

- **Histogram unit is configurable.** Core-based SDKs default to milliseconds but can be configured to emit seconds. Check the SDK-specific runtime/telemetry configuration docs for your language.
- **`temporal_sticky_cache_miss`** means the worker performed a full history replay. High miss rates indicate cache size pressure — tune the workflow cache size in your worker options.
- **`temporal_workflow_task_schedule_to_start_latency`** is the primary signal for workflow worker capacity. P95 above ~1 second indicates task queue backlog.
- **`temporal_activity_schedule_to_start_latency`** is the activity equivalent — high values mean activity workers cannot keep up.
- **`temporal_workflow_task_replay_latency`** measures only the replay portion of a workflow task. High values indicate large or complex histories that take a long time to replay.
- **`temporal_activity_task_received`** is unique to Core SDK — it records when Core hands an activity task to the language layer, before execution begins. Comparing it to `temporal_activity_schedule_to_start_latency` can reveal queuing within the language runtime itself.
- **gRPC `status_code` tag** uses SCREAMING_SNAKE_CASE matching the gRPC spec (e.g. `FAILED_PRECONDITION`, not `FailedPrecondition`).
- Some SDKs may emit additional metrics beyond this list. Only metrics in the [official SDK metrics reference](https://docs.temporal.io/references/sdk-metrics) have guaranteed, defined behavior.