# Temporal Go SDK Metrics

All metrics are prefixed with `temporal_` before being exported to their configured destination.
Histogram metrics in the Go SDK are measured in **seconds**.

**Source:**
- [`internal/common/metrics/constants.go`](https://github.com/temporalio/sdk-go/blob/master/internal/common/metrics/constants.go)

---

## gRPC Client Metrics

Emitted on every outbound RPC call from the SDK to the Temporal server.

| Metric | Type | Description |
|---|---|---|
| `temporal_request` | Counter | Every short (non-poll) gRPC request sent |
| `temporal_request_failure` | Counter | Failed short gRPC requests |
| `temporal_request_latency` | Histogram | Latency of short gRPC requests |
| `temporal_long_request` | Counter | Long-poll gRPC requests (`Poll*` calls) |
| `temporal_long_request_failure` | Counter | Failed long-poll requests |
| `temporal_long_request_latency` | Histogram | Latency of long-poll requests |

**Tags:** `operation` (RPC method name), `namespace`, `task_queue`, `status_code` (on failure)

---

## Worker Lifecycle Metrics

| Metric | Type | Description |
|---|---|---|
| `temporal_worker_start` | Counter | Emitted once when a worker is started |
| `temporal_worker_task_slots_available` | Gauge | Number of execution slots currently available |
| `temporal_worker_task_slots_used` | Gauge | Number of execution slots currently in use |
| `temporal_poller_start` | Counter | Emitted each time a poller goroutine starts |
| `temporal_num_pollers` | Gauge | Current number of active pollers |

**Tags:** `worker_type` (`WorkflowWorker`, `ActivityWorker`, `LocalActivityWorker`, `NexusWorker`), `namespace`, `task_queue`

`temporal_poller_start` and `temporal_num_pollers` also carry a `poller_type` tag.

---

## Workflow Task Metrics

Emitted during workflow task polling and execution.

| Metric | Type | Description |
|---|---|---|
| `temporal_workflow_task_queue_poll_empty` | Counter | Poll completed with no task returned (long-poll timeout) |
| `temporal_workflow_task_queue_poll_succeed` | Counter | Poll returned a workflow task |
| `temporal_workflow_task_schedule_to_start_latency` | Histogram | Time from task scheduled on server to worker picking it up |
| `temporal_workflow_task_execution_latency` | Histogram | Time to execute a single workflow task |
| `temporal_workflow_task_replay_latency` | Histogram | Time spent replaying history during a workflow task |
| `temporal_workflow_task_execution_failed` | Counter | Workflow task execution failed |
| `temporal_workflow_task_no_completion` | Counter | Task was processed but no completion was sent to the server |

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

> **Note:** The Go SDK uses `temporal_workflow_canceled` (one `l`). The Java SDK uses `temporal_workflow_cancelled` (two `l`s).

---

## Workflow Cache Metrics

Emitted by the sticky cache (workflow execution cache).

| Metric | Type | Description |
|---|---|---|
| `temporal_sticky_cache_hit` | Counter | Workflow task found a matching cached execution |
| `temporal_sticky_cache_miss` | Counter | Workflow task found no cached execution (cold replay required) |
| `temporal_sticky_cache_size` | Gauge | Current number of cached workflow executions |
| `temporal_sticky_cache_total_forced_eviction` | Counter | Executions forcibly evicted from the cache |

---

## Workflow Thread Metrics

| Metric | Type | Description |
|---|---|---|
| `temporal_workflow_active_thread_count` | Gauge | Total live workflow goroutines in the worker process |

---

## Activity Task Metrics

Emitted during activity task polling and execution.

| Metric | Type | Description |
|---|---|---|
| `temporal_activity_poll_no_task` | Counter | Activity poll completed with no task returned |
| `temporal_activity_schedule_to_start_latency` | Histogram | Time from activity task scheduled to worker picking it up |
| `temporal_activity_execution_latency` | Histogram | Time to execute a single activity task attempt |
| `temporal_activity_succeed_endtoend_latency` | Histogram | Total time from first schedule to successful completion (across all retries) |
| `temporal_activity_execution_failed` | Counter | Activity task execution failed |
| `temporal_activity_task_error` | Counter | Internal error processing an activity task (distinct from application-level failures) |
| `temporal_unregistered_activity_invocation` | Counter | Activity type dispatched to this worker is not registered |

**Tags:** `activity_type`, `namespace`, `task_queue`

> **Note:** `temporal_activity_execution_cancelled` is not present in the Go SDK (unlike Java). Cancellations in Go are surfaced as failures or as a clean return depending on how the activity handles context cancellation.

---

## Local Activity Metrics

Emitted for local activities, which execute in-process within the workflow worker.

| Metric | Type | Description |
|---|---|---|
| `temporal_local_activity_execution_latency` | Histogram | Time to execute a single local activity attempt |
| `temporal_local_activity_succeed_endtoend_latency` | Histogram | Total time from schedule to successful completion |
| `temporal_local_activity_execution_failed` | Counter | Local activity execution failed |
| `temporal_local_activity_execution_cancelled` | Counter | Local activity execution cancelled |
| `temporal_local_activity_error` | Counter | Internal error during local activity execution |

**Deprecated aliases** (still emitted for backwards compatibility):

| Deprecated metric | Replaced by |
|---|---|
| `temporal_local_activity_failed` | `temporal_local_activity_execution_failed` |
| `temporal_local_activity_canceled` | `temporal_local_activity_execution_cancelled` |
| `temporal_local_activity_total` | *(no direct replacement; was a total execution counter)* |

**Tags:** `activity_type`, `namespace`, `task_queue`

---

## Nexus Task Metrics

Go SDK specific (also available in Java SDK).

| Metric | Type | Description |
|---|---|---|
| `temporal_nexus_poll_no_task` | Counter | Nexus task poll returned empty |
| `temporal_nexus_schedule_to_start_latency` | Histogram | Time from Nexus task scheduled to worker picking it up |
| `temporal_nexus_execution_latency` | Histogram | Time to execute a Nexus task |
| `temporal_nexus_task_endtoend_latency` | Histogram | Total end-to-end time for a Nexus task |
| `temporal_nexus_execution_failed` | Counter | Nexus task execution failed |

**Tags:** `namespace`, `task_queue`, `nexus_service`, `nexus_operation`

---

## Signal Metrics

| Metric | Type | Description |
|---|---|---|
| `temporal_corrupted_signals` | Counter | Signals received that could not be deserialized |

**Tags:** `workflow_type`, `namespace`, `task_queue`

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
| `failure_reason` | Reason for task execution failure | `NonDeterminismError`, `WorkflowError` |
| `nexus_service` | Nexus service name | — |
| `nexus_operation` | Nexus operation name | — |
| `status_code` | gRPC status code on request failure | `FailedPrecondition`, `Unavailable` |
| `client_name` | SDK client identifier | `temporal_go` |

---

## Go SDK vs Java SDK Differences

| Area | Go SDK | Java SDK |
|---|---|---|
| Histogram unit | Seconds | Seconds |
| Workflow canceled spelling | `temporal_workflow_canceled` | `temporal_workflow_cancelled` |
| Local activity deprecated metrics | `local_activity_total`, `local_activity_failed`, `local_activity_canceled` still emitted | Not present |
| `activity_task_error` | Present | Not present |
| `workflow_task_replay_latency` | Present | Not present |
| `num_pollers` gauge | Present | Not present |
| `workflow_active_thread_count` | Goroutine count | Thread count |
| Activity cancellation counter | Not present | `temporal_activity_execution_cancelled` |
| `worker_task_slots_used` | Present | Present |

---

## Notes

- **`temporal_long_request` vs `temporal_request`:** Poll operations (`PollWorkflowTaskQueue`, `PollActivityTaskQueue`, `PollNexusTaskQueue`) emit under `temporal_long_request_*`. All other RPC calls use `temporal_request_*`.
- **`temporal_sticky_cache_miss`** means the worker must replay the full workflow history from scratch. High miss rates indicate cache eviction pressure — tune `WorkerOptions.MaxWorkflowExecutionCacheSize`.
- **`temporal_workflow_task_schedule_to_start_latency`** is the primary signal for workflow worker capacity. P95 above ~1 second indicates task queue backlog.
- **`temporal_activity_schedule_to_start_latency`** is the activity equivalent — high values mean activity workers cannot keep up.
- **`temporal_workflow_task_replay_latency`** is Go-specific and measures just the replay portion of a workflow task. High values here indicate large or expensive history replay.
- **`temporal_corrupted_signals`** fires when a signal payload cannot be deserialized — typically a schema mismatch between signal sender and workflow definition.
- Some SDKs may emit additional metrics beyond this list. Only metrics in the [official SDK metrics reference](https://docs.temporal.io/references/sdk-metrics) have guaranteed, defined behavior.