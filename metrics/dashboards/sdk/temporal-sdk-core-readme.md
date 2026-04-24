# Temporal Core SDK Dashboard (TypeScript, Python, .NET, Ruby)

A Grafana dashboard for monitoring Temporal workers built on the Core SDK — the shared Rust library that powers the **TypeScript, Python, .NET, and Ruby** Temporal SDKs.

> **Compatibility:** Temporal Core SDK (TypeScript, Python, .NET, Ruby) · Grafana 9.0+ · Prometheus

---

## Table of Contents

- [Overview](#overview)
- [Metric Naming Notes](#metric-naming-notes)
- [Template Variables](#template-variables)
- [Groups and Panels](#groups-and-panels)
    - [gRPC Requests and Request Latencies](#1-grpc-requests-and-request-latencies)
    - [Worker Lifecycle](#2-worker-lifecycle)
    - [Worker Cache](#3-worker-cache)
    - [Workflow Executions Info](#4-workflow-executions-info)
    - [Schedule To Start Latencies](#5-schedule-to-start-latencies)
    - [Workflow Task Info](#6-workflow-task-info)
    - [Activity Task Info](#7-activity-task-info)
    - [Local Activity Info](#8-local-activity-info)
    - [Nexus Tasks Info](#9-nexus-tasks-info)

---

## Overview

This dashboard provides observability into Temporal workers built on the Core SDK. Since the Core SDK is a shared Rust library, all metrics documented here are emitted identically across TypeScript, Python, .NET, and Ruby SDKs. It is designed to help operators:

- Monitor gRPC request rates and long-poll activity
- Identify request failures and latency issues by operation
- Diagnose worker health, task execution problems, and cache efficiency

---

## Metric Naming Notes

**Histogram unit:** The Core SDK emits histogram metrics in **milliseconds** by default. All latency panels in this dashboard use `ms` as the display unit. If you have configured your SDK to emit histograms in seconds, update the panel units accordingly.

This differs from the Go and Java SDK dashboards which always use seconds.

Notable Core SDK differences vs Go and Java SDKs:

| Area | Core SDK | Go SDK | Java SDK |
|---|---|---|---|
| Histogram unit | Milliseconds (default) | Seconds | Seconds |
| `activity_task_received` counter | Present | Not present | Not present |
| `local_activity_total` counter | Present | Present (deprecated alias) | Not present |
| `workflow_task_no_completion` counter | Not present | Present | Present |
| `activity_task_error` counter | Not present | Present | Not present |
| `local_activity_error` counter | Not present | Present | Not present |
| `poller_start` counter | Not present | Present | Present |
| `workflow_active_thread_count` gauge | Not present | Present | Present |
| Nexus execution metric naming | `temporal_nexus_task_execution_latency` / `temporal_nexus_task_execution_failed` | `temporal_nexus_execution_latency` / `temporal_nexus_execution_failed` | `temporal_nexus_execution_latency` / `temporal_nexus_execution_failed` |
| `status_code` tag format | SCREAMING_SNAKE_CASE e.g. `FAILED_PRECONDITION` | PascalCase e.g. `FailedPrecondition` | SCREAMING_SNAKE_CASE |

---

## Template Variables

| Variable | Description | Default |
|---|---|---|
| **Datasource** | Prometheus datasource to use | — |
| **Namespace** | Filters panels to a specific Temporal namespace | — |
| **Percentile** | Histogram quantile for latency panels | `0.99` |

All panels that show rates use `$__rate_interval` for Prometheus rate calculations. All latency panels respect the selected percentile variable.

---

## Groups and Panels

---

### 1. gRPC Requests and Request Latencies

This section focuses on gRPC metrics. These metrics are really useful for troubleshooting issues such as workflow task timeouts, reaching limits on throttling etc. General troubleshooting info can be found here: https://github.com/tsurdilo/temporal-server-operations/blob/main/metrics/SDK_METRICS_REQUEST_FAILURES.md

Note: The `status_code` tag in Core SDK uses SCREAMING_SNAKE_CASE (e.g. `FAILED_PRECONDITION`), unlike the Go SDK which uses PascalCase.

| Panel | Description |
|---|---|
| **Requests** | Rate of gRPC requests sent by the SDK broken down by operation and status code. |
| **Long-Poll Requests** | Rate of long-poll gRPC requests (`Poll*` calls) broken down by operation and status code. A drop to zero means workers have stopped polling. |
| **Request Failures** | Rate of failed gRPC requests broken down by operation and status code. Turns red at any non-zero value. |
| **Long-Poll Request Failures** | Rate of failed long-poll requests broken down by operation and status code. |
| **Request Latency** | Latency of short (non-poll) gRPC requests broken down by operation at the selected percentile. Uses `temporal_request_latency_bucket`. Reported in milliseconds. |
| **Long-Poll Request Latency** | Latency of long-poll gRPC requests broken down by operation at the selected percentile. Uses `temporal_long_request_latency_bucket`. Reported in milliseconds. |

---

### 2. Worker Lifecycle

This section focuses on worker lifecycle metrics. Note that `temporal_poller_start` is not emitted by the Core SDK — use `temporal_num_pollers` to observe poller activity instead.

| Panel | Description |
|---|---|
| **Worker Start** | Rate of worker start events broken down by worker type and task queue. |
| **Number of Active Pollers** | Current number of active pollers broken down by poller type and task queue. A drop to zero means workers have stopped polling entirely. |
| **Worker Task Slots Available** | Current number of execution slots available broken down by worker type and task queue. A value consistently near zero indicates workers are at capacity. |
| **Worker Task Slots Used** | Current number of execution slots in use broken down by worker type and task queue. |

---

### 3. Worker Cache

This section focuses on Worker Sticky Cache info. The sticky cache keeps workflow executions in memory to avoid replaying history on every task.

| Panel | Description |
|---|---|
| **Sticky Cache Hit** | Rate of workflow tasks that found a matching cached execution. A high hit rate means workers are efficiently avoiding full replays. |
| **Sticky Cache Miss** | Rate of workflow tasks that found no cached execution, requiring a full replay. Turns orange at any non-zero value. |
| **Sticky Cache Size** | Current number of workflow executions cached in the sticky cache. |
| **Sticky Cache Forced Evictions** | Rate of executions forcibly evicted from the sticky cache. Turns orange at any non-zero value. Indicates the cache may be undersized for the workload. |

---

### 4. Workflow Executions Info

This section focuses on workflow execution completions and end to end latency of execution. All panels are broken down by `workflow_type` and `task_queue`.

| Panel | Description |
|---|---|
| **Workflow Completed** | Total count of workflow executions completed successfully over the selected time range. |
| **Workflow Canceled** | Total count of workflow executions ended by cancellation. Uses `temporal_workflow_canceled` (one `l`). |
| **Workflow Failed** | Total count of workflow executions that failed. Turns red at any non-zero value. |
| **Workflow Continue As New** | Total count of workflow executions that ended with continue-as-new. |
| **Workflow End-to-End Latency** | Total time from schedule to close for a single workflow run at the selected percentile. Uses `temporal_workflow_endtoend_latency_bucket`. Reported in milliseconds. |

---

### 5. Schedule To Start Latencies

This section focuses on schedule to start latencies for workflow and activity tasks. High values here are a primary indicator of insufficient worker provisioning.

| Panel | Description |
|---|---|
| **Workflow Task Schedule To Start Latency** | Time from a workflow task being scheduled to a worker picking it up at the selected percentile. Uses `temporal_workflow_task_schedule_to_start_latency_bucket`. Reported in milliseconds. |
| **Activity Schedule To Start Latency** | Time from an activity task being scheduled to a worker picking it up at the selected percentile. Uses `temporal_activity_schedule_to_start_latency_bucket`. Reported in milliseconds. |

---

### 6. Workflow Task Info

This section focuses on Workflow Task Information. All panels are broken down by `workflow_type` and `task_queue`. Note that `temporal_workflow_task_no_completion` is not present in the Core SDK.

| Panel | Description |
|---|---|
| **Workflow Task Poll Empty** | Rate of polls that completed with no task returned (long-poll timeout). |
| **Workflow Task Poll Succeed** | Rate of polls that returned a workflow task. |
| **Workflow Task Execution Latency** | Time to execute a single workflow task at the selected percentile. Uses `temporal_workflow_task_execution_latency_bucket`. Reported in milliseconds. |
| **Workflow Task Replay Latency** | Time spent replaying history during a workflow task at the selected percentile. Uses `temporal_workflow_task_replay_latency_bucket`. Reported in milliseconds. |
| **Workflow Task Execution Failed** | Rate of workflow task execution failures broken down by workflow type, task queue, and failure reason. Turns red at any non-zero value. `failure_reason` will be `NonDeterminismError` or `WorkflowError`. |

---

### 7. Activity Task Info

This section focuses on information about Activity Tasks. All panels are broken down by `activity_type` and `task_queue`. Note that `temporal_activity_task_received` is Core SDK specific and not present in Go or Java SDKs.

| Panel | Description |
|---|---|
| **Activity Poll No Task** | Rate of activity polls that completed with no task returned. |
| **Activity Task Received** | Rate of activity tasks received and dispatched to the language layer. Core SDK specific — fires when Core hands a task to the language runtime. |
| **Activity Execution Failed** | Rate of activity task execution failures. Turns red at any non-zero value. |
| **Unregistered Activity Invocation** | Rate of activity types dispatched to this worker that are not registered. Turns red at any non-zero value. |
| **Activity Execution Latency** | Time from when Core dispatched the task to lang completing it at the selected percentile. Uses `temporal_activity_execution_latency_bucket`. Reported in milliseconds. |
| **Activity Succeed End-to-End Latency** | Total time from first schedule to successful completion across all retries at the selected percentile. Uses `temporal_activity_succeed_endtoend_latency_bucket`. Reported in milliseconds. |

---

### 8. Local Activity Info

This section focuses on Local Activities information. All panels are broken down by `activity_type` and `task_queue`. Local activities execute in-process within the workflow worker.

| Panel | Description |
|---|---|
| **Local Activity Execution Failed** | Rate of local activity execution failures. Turns red at any non-zero value. |
| **Local Activity Execution Cancelled** | Rate of local activity execution cancellations. Turns orange at any non-zero value. |
| **Local Activity Total** | Rate of total local activity executions attempted. Core SDK specific counter. |
| **Local Activity Execution Latency** | Time to execute a single local activity attempt at the selected percentile. Uses `temporal_local_activity_execution_latency_bucket`. Reported in milliseconds. |
| **Local Activity Succeed End-to-End Latency** | Total time from schedule to successful completion at the selected percentile. Uses `temporal_local_activity_succeed_endtoend_latency_bucket`. Reported in milliseconds. |

---

### 9. Nexus Tasks Info

This section focuses on Nexus Tasks information. All panels are filtered by `namespace` and broken down by `task_queue`. Note that Core SDK uses `temporal_nexus_task_execution_latency` and `temporal_nexus_task_execution_failed` — different names from Go and Java SDKs.

| Panel | Description |
|---|---|
| **Nexus Poll No Task** | Rate of Nexus task polls that returned empty. |
| **Nexus Task Execution Failed** | Rate of Nexus task execution failures broken down by task queue and failure reason. Uses `temporal_nexus_task_execution_failed`. `failure_reason` values include `internal_sdk_error` and `handler_error_{TYPE}`. |
| **Nexus Schedule To Start Latency** | Time from a Nexus task being scheduled to a worker picking it up at the selected percentile. Uses `temporal_nexus_schedule_to_start_latency_bucket`. Reported in milliseconds. |
| **Nexus Task Execution Latency** | Time to execute a Nexus task at the selected percentile. Uses `temporal_nexus_task_execution_latency_bucket`. Reported in milliseconds. |
| **Nexus Task End-to-End Latency** | Total end-to-end time for a Nexus task at the selected percentile. Uses `temporal_nexus_task_endtoend_latency_bucket`. Reported in milliseconds. |

---

## Related Resources

- [Temporal SDK Metrics Reference](https://docs.temporal.io/references/sdk-metrics)
- [Temporal TypeScript SDK](https://github.com/temporalio/sdk-typescript)
- [Temporal Python SDK](https://github.com/temporalio/sdk-python)
- [Temporal .NET SDK](https://github.com/temporalio/sdk-dotnet)
- [Temporal Ruby SDK](https://github.com/temporalio/sdk-ruby)
- [SDK Metrics Request Failures Troubleshooting](https://github.com/tsurdilo/temporal-server-operations/blob/main/metrics/SDK_METRICS_REQUEST_FAILURES.md)
- [Temporal Community Forum](https://community.temporal.io)