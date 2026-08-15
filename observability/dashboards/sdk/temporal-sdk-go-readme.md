# Temporal Go SDK Dashboard

A Grafana dashboard for monitoring Temporal Go SDK clients and workers.

> **Compatibility:** Temporal Go SDK · Grafana 9.0+ · Prometheus
>
> **Reporter:** This dashboard targets Go SDK workers using the default Prometheus metrics reporter. Metric names are exported as-is from the SDK — no `_total` or `_seconds` suffixes are appended. If you are using the Java SDK, use the `temporal-sdk-java-micrometer` or `temporal-sdk-java-otel` dashboard instead.

> **Current version:** v1.3.0 — see [CHANGELOG](./temporal-sdk-go-changelog.md)

---

## Table of Contents

- [Overview](#overview)
- [Go SDK Metric Naming Notes](#go-sdk-metric-naming-notes)
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

This dashboard provides observability into Temporal Go SDK clients and workers from the SDK side, using Prometheus metrics emitted by the Go SDK's built-in metrics reporter. It is designed to help operators:

- Monitor gRPC request rates and long-poll activity
- Identify request failures and latency issues by operation
- Diagnose worker health, task execution problems, and cache efficiency

---

## Go SDK Metric Naming Notes

The Go SDK exports metrics using the standard Prometheus client library. Metric names are used exactly as defined by the SDK — no `_total` suffix is appended to counters and no `_seconds` suffix is appended to histograms. This is different from the Java SDK with Micrometer, which applies both transformations automatically.

Note that the Go SDK uses `temporal_workflow_canceled` (one `l`) while the Java SDK uses `temporal_workflow_cancelled` (two `l`s).

### Go-specific metrics not present in the Java dashboards

| Metric | Description |
|---|---|
| `temporal_workflow_active_thread_count` | Number of live workflow goroutines in the worker process (gauge). |
| `temporal_activity_task_error` | Internal errors processing activity tasks, distinct from application-level failures. |
| `temporal_local_activity_error` | Internal errors during local activity execution. |
| `temporal_nexus_task_endtoend_latency_bucket` | Total end-to-end latency for a Nexus task. |

---

## Template Variables

| Variable | Description | Default |
|---|---|---|
| **Datasource** | Prometheus datasource to use | — |
| **Namespace** | Filters panels to a specific Temporal namespace | — |
| **Percentile** | Histogram quantile for latency panels | `0.99` |

All panels that show rates use `$__rate_interval` for Prometheus rate calculations. All latency panels respect the selected percentile variable. The Namespace variable is populated from `label_values(temporal_request, namespace)`.

---

## Groups and Panels

---

### 1. gRPC Requests and Request Latencies

This section focuses on gRPC metrics. These metrics are really useful for troubleshooting issues such as workflow task timeouts, reaching limits on throttling etc. General troubleshooting info can be found here: https://github.com/tsurdilo/temporal-server-operations/blob/main/metrics/SDK_METRICS_REQUEST_FAILURES.md

| Panel | Description |
|---|---|
| **Requests** | Rate of gRPC requests sent by the SDK broken down by operation and status code. Uses `temporal_request`. |
| **Long-Poll Requests** | Rate of long-poll gRPC requests (`Poll*` calls) broken down by operation and status code. A drop to zero means workers have stopped polling. Uses `temporal_long_request`. |
| **Request Failures** | Rate of failed gRPC requests broken down by operation and status code. Turns red at any non-zero value. Uses `temporal_request_failure`. |
| **Long-Poll Request Failures** | Rate of failed long-poll requests broken down by operation and status code. Uses `temporal_long_request_failure`. |
| **Request Latency** | Latency of short (non-poll) gRPC requests broken down by operation at the selected percentile. Uses `temporal_request_latency_bucket`. |
| **Long-Poll Request Latency** | Latency of long-poll gRPC requests broken down by operation at the selected percentile. Uses `temporal_long_request_latency_bucket`. |

---

### 2. Worker Lifecycle

This section focuses on worker lifecycle metrics. Use this group to monitor worker and poller activity, slot utilization, and overall goroutine health.

| Panel | Description |
|---|---|
| **Worker Start** | Rate of worker start events broken down by worker type and task queue. A spike may indicate workers restarting frequently. Uses `temporal_worker_start`. |
| **Poller Start** | Rate of poller goroutine start events broken down by poller type and task queue. Uses `temporal_poller_start`. |
| **Worker Task Slots Available** | Current number of execution slots available broken down by worker type and task queue. A value consistently near zero indicates workers are at capacity. Uses `temporal_worker_task_slots_available` (gauge). |
| **Worker Task Slots Used** | Current number of execution slots in use broken down by worker type and task queue. Uses `temporal_worker_task_slots_used` (gauge). |
| **Number of Active Pollers** | Current number of active pollers broken down by poller type and task queue. A drop to zero means workers have stopped polling entirely. Uses `temporal_num_pollers` (gauge). |
| **Workflow Active Goroutine Count** | Total number of live workflow goroutines in the worker process. A sustained climb may indicate a goroutine leak. Uses `temporal_workflow_active_thread_count` (gauge). |

---

### 3. Worker Cache

This section focuses on Worker Sticky Cache info. The sticky cache keeps workflow executions in memory to avoid replaying history on every task.

| Panel | Description |
|---|---|
| **Sticky Cache Hit** | Rate of workflow tasks that found a matching cached execution. A high hit rate means workers are efficiently avoiding cold replays. Uses `temporal_sticky_cache_hit`. |
| **Sticky Cache Miss** | Rate of workflow tasks that found no cached execution, requiring a cold replay. Turns orange at any non-zero value. Uses `temporal_sticky_cache_miss`. |
| **Sticky Cache Size** | Current number of workflow executions cached in the sticky cache. Uses `temporal_sticky_cache_size` (gauge). |
| **Sticky Cache Forced Evictions** | Rate of executions forcibly evicted from the sticky cache. Turns orange at any non-zero value. Indicates the cache may be undersized for the workload. Uses `temporal_sticky_cache_total_forced_eviction`. |

---

### 4. Workflow Executions Info

This section focuses on workflow execution completions and end to end latency of execution. All panels are broken down by `workflow_type` and `task_queue`.

| Panel | Description |
|---|---|
| **Workflow Completed** | Total count of workflow executions completed successfully over the selected time range. Uses `temporal_workflow_completed`. |
| **Workflow Canceled** | Total count of workflow executions ended by cancellation over the selected time range. Uses `temporal_workflow_canceled`. |
| **Workflow Failed** | Total count of workflow executions that failed over the selected time range. Turns red at any non-zero value. Uses `temporal_workflow_failed`. |
| **Workflow Continue As New** | Total count of workflow executions that ended with continue-as-new over the selected time range. Uses `temporal_workflow_continue_as_new`. |
| **Workflow End-to-End Latency** | Total time from schedule to close for a single workflow run at the selected percentile. Uses `temporal_workflow_endtoend_latency_bucket`. |

---

### 5. Schedule To Start Latencies

This section focuses on schedule to start latencies for workflow and activity tasks. High values here are a primary indicator of insufficient worker provisioning.

| Panel | Description |
|---|---|
| **Workflow Task Schedule To Start Latency** | Time from a workflow task being scheduled to a worker picking it up, broken down by task queue at the selected percentile. Uses `temporal_workflow_task_schedule_to_start_latency_bucket`. |
| **Activity Schedule To Start Latency** | Time from an activity task being scheduled to a worker picking it up, broken down by activity type and task queue at the selected percentile. Uses `temporal_activity_schedule_to_start_latency_bucket`. |

---

### 6. Workflow Task Info

This section focuses on Workflow Task Information. All panels are broken down by `workflow_type` and `task_queue`.

| Panel | Description |
|---|---|
| **Workflow Task Poll Empty** | Rate of polls that completed with no task returned (long-poll timeout). Uses `temporal_workflow_task_queue_poll_empty`. |
| **Workflow Task Poll Succeed** | Rate of polls that returned a workflow task. Uses `temporal_workflow_task_queue_poll_succeed`. |
| **Workflow Task Execution Latency** | Time to execute a single workflow task at the selected percentile. Uses `temporal_workflow_task_execution_latency_bucket`. |
| **Workflow Task Replay Latency** | Time spent replaying history during a workflow task at the selected percentile. Uses `temporal_workflow_task_replay_latency_bucket`. |
| **Workflow Task Execution Failed** | Rate of workflow task execution failures broken down by workflow type, task queue, and failure reason. Turns red at any non-zero value. `failure_reason` will be `NonDeterminismError` or `WorkflowError`. Uses `temporal_workflow_task_execution_failed`. |
| **Workflow Task No Completion** | Rate of workflow tasks processed but for which no completion was sent to the server. Turns orange at any non-zero value. Uses `temporal_workflow_task_no_completion`. |

---

### 7. Activity Task Info

This section focuses on information about Activity Tasks. All panels are broken down by `activity_type` and `task_queue`.

| Panel | Description |
|---|---|
| **Activity Poll No Task** | Rate of activity polls that completed with no task returned. Uses `temporal_activity_poll_no_task`. |
| **Activity Execution Failed** | Rate of activity task execution failures. Turns red at any non-zero value. Uses `temporal_activity_execution_failed`. |
| **Activity Task Error** | Rate of internal errors processing activity tasks, distinct from application-level failures. Turns red at any non-zero value. Uses `temporal_activity_task_error`. |
| **Unregistered Activity Invocation** | Rate of activity types dispatched to this worker that are not registered. Turns red at any non-zero value. Uses `temporal_unregistered_activity_invocation`. |
| **Activity Execution Latency** | Time to execute a single activity task attempt at the selected percentile. Uses `temporal_activity_execution_latency_bucket`. |
| **Activity Succeed End-to-End Latency** | Total time from first schedule to successful completion across all retries at the selected percentile. Uses `temporal_activity_succeed_endtoend_latency_bucket`. |

---

### 8. Local Activity Info

This section focuses on Local Activities information. All panels are broken down by `activity_type` and `task_queue`. Local activities execute in-process within the workflow worker.

| Panel | Description |
|---|---|
| **Local Activity Execution Failed** | Rate of local activity execution failures. Turns red at any non-zero value. Uses `temporal_local_activity_execution_failed`. |
| **Local Activity Execution Cancelled** | Rate of local activity execution cancellations. Turns orange at any non-zero value. Uses `temporal_local_activity_execution_cancelled`. |
| **Local Activity Error** | Rate of internal errors during local activity execution. Turns red at any non-zero value. Uses `temporal_local_activity_error`. |
| **Local Activity Execution Latency** | Time to execute a single local activity attempt at the selected percentile. Uses `temporal_local_activity_execution_latency_bucket`. |
| **Local Activity Succeed End-to-End Latency** | Total time from schedule to successful completion at the selected percentile. Uses `temporal_local_activity_succeed_endtoend_latency_bucket`. |

---

### 9. Nexus Tasks Info

This section focuses on Nexus Tasks information. All panels are filtered by `namespace` and broken down by `task_queue`.

| Panel | Description |
|---|---|
| **Nexus Poll No Task** | Rate of Nexus task polls that returned empty. Uses `temporal_nexus_poll_no_task`. |
| **Nexus Execution Failed** | Rate of Nexus task execution failures. Turns red at any non-zero value. Uses `temporal_nexus_execution_failed`. |
| **Nexus Schedule To Start Latency** | Time from a Nexus task being scheduled to a worker picking it up at the selected percentile. Uses `temporal_nexus_schedule_to_start_latency_bucket`. |
| **Nexus Execution Latency** | Time to execute a Nexus task at the selected percentile. Uses `temporal_nexus_execution_latency_bucket`. |
| **Nexus Task End-to-End Latency** | Total end-to-end time for a Nexus task at the selected percentile. Uses `temporal_nexus_task_endtoend_latency_bucket`. |

---