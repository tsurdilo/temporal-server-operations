# Temporal SDK Overview Grafana Dashboard

A Grafana dashboard for monitoring Temporal SDK clients and SDK workers using Prometheus metrics emitted by the SDK itself.

> **Compatibility:** Temporal SDK (Go, Java, Python, TypeScript, .NET, Ruby) · Grafana 9.0+ · Prometheus

---

## Table of Contents

- [Overview](#overview)
- [SDK Metric Naming Notes](#sdk-metric-naming-notes)
- [Template Variables](#template-variables)
- [Groups and Panels](#groups-and-panels)
    - [Regular and Long-Poll Requests and Failures](#1-regular-and-long-poll-requests-and-failures)
    - [Workflow Completion Stats](#2-workflow-completion-stats)

---

## Overview

This dashboard provides observability into the behavior of Temporal SDK clients and SDK workers from the SDK side, complementing the server-side metrics found in the Temporal Server Dashboard. It is built on Prometheus metrics emitted directly by Temporal SDKs and is designed to help operators:

- Monitor SDK request rates and long-poll activity
- Identify request failures and error patterns by operation and status code
- Diagnose connectivity and worker health issues from the client perspective

---

## SDK Metric Naming Notes

Temporal SDKs share the same metric names for request and failure counters across all SDK families. However, latency metrics differ between SDK families:

| SDK Family | Latency metric unit | Timer metric postfix |
|---|---|---|
| **Go SDK** | seconds | none |
| **Java SDK** | seconds | `_seconds` |
| **Core SDK** (Python, TypeScript, .NET, Ruby) | milliseconds | none |

Panels in this dashboard that use latency metrics are organized into separate rows per SDK family to account for these differences. Panels that use counter metrics (requests, failures) are shared across all SDK families as the metric names are identical.

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

### 1. Regular and Long-Poll Requests and Failures

This section focuses on requests and request failures for operations your SDK clients and SDK Workers make to Temporal server. These metrics are emitted by all SDK families under the same metric names, making them a reliable cross-SDK signal for connectivity health and error rates.

**Regular requests** are standard RPC calls such as `StartWorkflowExecution`, `SignalWorkflowExecution`, or `RespondActivityTaskCompleted`. **Long-poll requests** are the blocking poll calls SDK workers make to receive workflow and activity tasks from the server — `PollWorkflowTaskQueue` and `PollActivityTaskQueue`.

| Panel | Description |
|---|---|
| **SDK Requests** | Rate of regular SDK requests broken down by namespace, operation, and status code. A healthy worker should show a steady rate of operations. Unexpected drops may indicate connectivity issues or worker crashes. |
| **SDK Long-Poll Requests** | Rate of long-poll SDK requests broken down by namespace, operation, and status code. Long-poll requests represent active worker polling — a sustained rate here means workers are connected and waiting for tasks. A drop to zero means workers have stopped polling. |
| **SDK Request Failures** | Rate of regular SDK request failures broken down by namespace, operation, and status code. Any sustained failure rate warrants investigation. The `status_code` label helps distinguish transient server errors from client-side issues such as invalid arguments or resource exhaustion. |
| **SDK Long-Poll Request Failures** | Rate of long-poll SDK request failures broken down by namespace, operation, and status code. Failures on poll operations can indicate server-side issues or misconfigured worker credentials. |

---

### 2. Workflow Completion Stats

This section focuses on completion status. Note that these are really only requests from client/worker to complete workflow execution. Temporal server is the ultimate source of truth for any execution start and completion. Always reference server metrics completion stats to compare. Server can in some cases reject worker or client request to complete execution. Server can under some conditions also complete execution without client/worker requests.

| Panel | Description |
|---|---|
| **Workflow Completed** | Total count of workflow executions successfully completed by the SDK worker over the selected time range. |
| **Workflow Cancelled** | Total count of workflow executions cancelled by the SDK client or worker over the selected time range. |
| **Workflow Failed** | Total count of workflow executions that failed as reported by the SDK worker. Turns red at any non-zero value. Always cross-reference with server-side workflow failed metrics as the server may reject or override completion. |
| **Workflow Continue As New** | Total count of workflow executions that continued as new as reported by the SDK worker. A normal pattern for long-running workflows to reset their history size. |

---

## Related Resources

- [Temporal SDK Metrics Reference](https://docs.temporal.io/references/sdk-metrics)
- [Temporal Go SDK](https://github.com/temporalio/sdk-go)
- [Temporal Java SDK](https://github.com/temporalio/sdk-java)
- [Temporal Python SDK](https://github.com/temporalio/sdk-python)
- [Temporal TypeScript SDK](https://github.com/temporalio/sdk-typescript)
- [Temporal .NET SDK](https://github.com/temporalio/sdk-dotnet)
- [Temporal Community Forum](https://community.temporal.io)