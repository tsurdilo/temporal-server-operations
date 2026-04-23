# Temporal Java SDK Metrics

All metrics are prefixed with `temporal_` before being exported to their configured destination.
Histogram metrics in the Java SDK are measured in **seconds**.

**Sources:**
- [`temporal-serviceclient/src/main/java/io/temporal/serviceclient/MetricsType.java`](https://github.com/temporalio/sdk-java/blob/master/temporal-serviceclient/src/main/java/io/temporal/serviceclient/MetricsType.java)
- [`temporal-sdk/src/main/java/io/temporal/worker/MetricsType.java`](https://github.com/temporalio/sdk-java/blob/master/temporal-sdk/src/main/java/io/temporal/worker/MetricsType.java)
- [`temporal-sdk/src/main/java/io/temporal/internal/worker/WorkflowWorker.java`](https://github.com/temporalio/sdk-java/blob/master/temporal-sdk/src/main/java/io/temporal/internal/worker/WorkflowWorker.java)
- [`temporal-sdk/src/main/java/io/temporal/internal/worker/ActivityWorker.java`](https://github.com/temporalio/sdk-java/blob/master/temporal-sdk/src/main/java/io/temporal/internal/worker/ActivityWorker.java)
- [`temporal-sdk/src/main/java/io/temporal/internal/worker/LocalActivityWorker.java`](https://github.com/temporalio/sdk-java/blob/master/temporal-sdk/src/main/java/io/temporal/internal/worker/LocalActivityWorker.java)

---

## gRPC Client Metrics

Emitted on every outbound RPC call from the SDK to the Temporal server.
Defined in `temporal-serviceclient/MetricsType.java`.

| Metric | Type | Description |
|---|---|---|
| `temporal_request` | Counter | Every gRPC request sent |
| `temporal_request_failure` | Counter | Failed gRPC requests |
| `temporal_request_latency` | Histogram | Latency of short (non-poll) gRPC requests |
| `temporal_long_request` | Counter | Long-poll gRPC requests (`Poll*` calls) |
| `temporal_long_request_failure` | Counter | Failed long-poll requests |
| `temporal_long_request_latency` | Histogram | Latency of long-poll requests |

**Tags:** `operation` (RPC method name e.g. `PollWorkflowTaskQueue`), `namespace`, `task_queue`

---

## Worker Lifecycle Metrics

| Metric | Type | Description |
|---|---|---|
| `temporal_worker_start` | Counter | Emitted once when a worker is started |
| `temporal_worker_task_slots_available` | Gauge | Number of execution slots currently available |
| `temporal_worker_task_slots_used` | Gauge | Number of execution slots currently in use |

**Tags:** `worker_type` (`WorkflowWorker`, `ActivityWorker`, `LocalActivityWorker`, `NexusWorker`), `namespace`, `task_queue`

---

## Workflow Task Metrics

Emitted by `WorkflowWorker` during workflow task polling and execution.

| Metric | Type | Description |
|---|---|---|
| `temporal_workflow_task_queue_poll_empty` | Counter | Poll completed with no task returned (long-poll timeout) |
| `temporal_workflow_task_queue_poll_succeed` | Counter | Poll returned a workflow task |
| `temporal_workflow_task_schedule_to_start_latency` | Histogram | Time from task scheduled on server to worker picking it up |
| `temporal_workflow_task_execution_latency` | Histogram | Time to execute a single workflow task |
| `temporal_workflow_task_execution_total_latency` | Histogram | Total time including workflow run lock wait |
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
| `temporal_workflow_cancelled` | Counter | Workflow execution ended by cancellation |
| `temporal_workflow_failed` | Counter | Workflow execution failed |
| `temporal_workflow_continue_as_new` | Counter | Workflow execution ended with continue-as-new |
| `temporal_workflow_endtoend_latency` | Histogram | Total time from schedule to close for a single workflow run |

**Tags:** `workflow_type`, `namespace`, `task_queue`

---

## Workflow Cache Metrics

Emitted by `WorkflowExecutorCache` (sticky cache / workflow thread cache).

| Metric | Type | Description |
|---|---|---|
| `temporal_sticky_cache_hit` | Counter | Workflow task found a matching cached execution |
| `temporal_sticky_cache_miss` | Counter | Workflow task found no cached execution (cold replay required) |
| `temporal_sticky_cache_size` | Gauge | Current number of cached workflow executions |
| `temporal_sticky_cache_total_forced_eviction` | Counter | Executions forcibly evicted from the cache |

---

## Workflow Thread Metrics

Java SDK specific.

| Metric | Type | Description |
|---|---|---|
| `temporal_workflow_active_thread_count` | Gauge | Total live workflow threads in the worker process |

---

## Activity Task Metrics

Emitted by `ActivityWorker` during activity task polling and execution.

| Metric | Type | Description |
|---|---|---|
| `temporal_activity_poll_no_task` | Counter | Activity poll completed with no task returned |
| `temporal_activity_schedule_to_start_latency` | Histogram | Time from activity task scheduled to worker picking it up |
| `temporal_activity_execution_latency` | Histogram | Time to execute a single activity task attempt |
| `temporal_activity_succeed_endtoend_latency` | Histogram | Total time from first schedule to successful completion (across all retries) |
| `temporal_activity_execution_failed` | Counter | Activity task execution failed |
| `temporal_activity_execution_cancelled` | Counter | Activity task execution cancelled |
| `temporal_unregistered_activity_invocation` | Counter | Activity type dispatched to this worker is not registered |

**Tags:** `activity_type`, `namespace`, `task_queue`

---

## Local Activity Metrics

Emitted by `LocalActivityWorker`. Local activities execute in-process within the workflow worker.

| Metric | Type | Description |
|---|---|---|
| `temporal_local_activity_execution_latency` | Histogram | Time to execute a single local activity attempt |
| `temporal_local_activity_succeed_endtoend_latency` | Histogram | Total time from schedule to successful completion |
| `temporal_local_activity_total_execution_latency` | Histogram | Total execution time including all local retries |
| `temporal_local_activity_execution_failed` | Counter | Local activity execution failed |
| `temporal_local_activity_execution_cancelled` | Counter | Local activity execution cancelled |

**Tags:** `activity_type`, `namespace`, `task_queue`

---

## Nexus Task Metrics

Java SDK specific (also available in Go SDK).

| Metric | Type | Description |
|---|---|---|
| `temporal_nexus_poll_no_task` | Counter | Nexus task poll returned empty |
| `temporal_nexus_schedule_to_start_latency` | Histogram | Time from Nexus task scheduled to worker picking it up |
| `temporal_nexus_execution_latency` | Histogram | Time to execute a Nexus task |
| `temporal_nexus_execution_failed` | Counter | Nexus task execution failed |

**Tags:** `namespace`, `task_queue`

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

---

## Notes

- **Histogram unit:** Java SDK histograms are measured in **seconds** (unlike Core-based SDKs which use milliseconds by default).
- **`temporal_long_request` vs `temporal_request`:** Poll operations (`PollWorkflowTaskQueue`, `PollActivityTaskQueue`) are tracked under `temporal_long_request_*`. All other RPC calls use `temporal_request_*`.
- **`temporal_sticky_cache_miss`** is a strong signal that workers are doing full history replay, which is expensive. High miss rates indicate cache eviction pressure — consider tuning `workflowCacheSize` in `WorkerFactoryOptions`.
- **`temporal_workflow_task_schedule_to_start_latency`** is the primary signal for worker capacity issues. P95 above ~1 second indicates the task queue is backing up.
- **`temporal_activity_schedule_to_start_latency`** is the activity equivalent — high values mean activity workers are not keeping up with the task queue.
- Some SDKs may emit additional metrics beyond this list. Only metrics in the [official SDK metrics reference](https://docs.temporal.io/references/sdk-metrics) have guaranteed, defined behavior.

---

## Java SDK vs Go SDK Differences

| Area | Java SDK | Go SDK |
|---|---|---|
| Workflow canceled spelling | `temporal_workflow_cancelled` (two `l`s) | `temporal_workflow_canceled` (one `l`) |
| `workflow_task_execution_total_latency` | Present (includes workflow run lock wait time) | Not present |
| `workflow_task_replay_latency` | Not present | Present |
| `num_pollers` gauge | Not present | Present |
| `poller_start` counter | Not present | Present |
| `activity_execution_cancelled` counter | Present | Not present |
| `activity_task_error` counter | Not present | Present |
| Local activity deprecated aliases | Not present | `local_activity_total`, `local_activity_failed`, `local_activity_canceled` still emitted |
| Nexus endtoend latency | Not present | `temporal_nexus_task_endtoend_latency` present |
| `corrupted_signals` counter | Not present | Present |
| `status_code` tag on request failure | Not present | Present |
| `cause` tag | Not present | Present |

---

## Java SDK vs Core SDK Differences

The Core SDK powers TypeScript, Python, .NET, and Ruby. Use this table when building dashboards or alerts that need to work across both Java workers and Core-based workers.

| Area | Java SDK | Core SDK (TS/Python/.NET/Ruby) |
|---|---|---|
| Histogram unit | Seconds | Milliseconds by default (configurable to seconds) |
| Workflow canceled spelling | `temporal_workflow_cancelled` (two `l`s) | `temporal_workflow_canceled` (one `l`) |
| `activity_task_received` counter | Not present | Present |
| `activity_execution_cancelled` counter | Present | Not present |
| `workflow_task_execution_total_latency` | Present (includes workflow run lock wait) | Not present |
| `workflow_task_replay_latency` | Not present | Present |
| `workflow_task_no_completion` counter | Present | Not present |
| `num_pollers` gauge | Not present | Present |
| `local_activity_total` counter | Not present | Present |
| Nexus execution metric naming | `temporal_nexus_execution_latency` / `temporal_nexus_execution_failed` | `temporal_nexus_task_execution_latency` / `temporal_nexus_task_execution_failed` |
| `status_code` tag on request failure | Not present | Present (SCREAMING_SNAKE_CASE) |
| `workflow_active_thread_count` gauge | Present (JVM thread count) | Not present |