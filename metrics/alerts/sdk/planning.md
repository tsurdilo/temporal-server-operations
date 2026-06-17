# Temporal SDK Grafana Alerts — Planning Document

> **Status: Work in Progress** — This document is a planning artifact for review and discussion before implementation begins. Alert definitions, thresholds, and runbook structure are subject to change based on feedback.

## Work Plan

1. **Descriptions** — Go through all alerts and finalize descriptions (what it detects, root causes, cross-references, remediation). **Next up: #13 (RespondActivityTaskCompleted Rate Dropped to Zero), then Section 2 onward.**
2. **README** — Create `metrics/alerts/sdk/README.md`: full alert index with severity, panel links per SDK (Micrometer / OTel / Go / Core), per-SDK PromQL variants, and index of the 4 YAML files. Operators should be able to look at any alert and immediately see the correct query for their SDK.
3. **Essential Set** — Define the essential alert list from the full inventory
4. **YAML + Runbooks** — One Grafana provisioning YAML per SDK reporter: `temporal-sdk-java-micrometer-alerts.yaml`, `temporal-sdk-java-otel-alerts.yaml`, `temporal-sdk-go-alerts.yaml`, `temporal-sdk-core-alerts.yaml`. One runbook markdown file per essential alert in `runbooks/`.
5. **Testing** — Test as many essential alerts as possible against the live Docker Compose stack before calling this done

> **Current step: Steps 1–4 COMPLETE. Next: Step 5 — testing against the live Docker Compose stack.**

## Upcoming Work

### Step 5 — Test Java Micrometer alerts against live Docker Compose stack
Use `deploy-temporal-metrics.sh` to deploy Java Micrometer dashboard + alerts. Test as many of the 23 essential alerts as possible and validate firing conditions.

### Step 6 — Multi-cluster replication split-brain playbook
Research and write a playbook covering the split-brain / FailoverVersion desync scenario raised in the Temporal community. Full thread and context below for reference.

**Scenario summary:** Two-cluster setup. After a reinstall of clusterB, its ClusterId changed. Both clusters ended up agreeing on ActiveClusterName but disagreed on clusterB's ClusterId — breaking clusterB → clusterA namespace replication. clusterA remained stuck at FailoverVersion=2 while clusterB advanced to v12. Workflows stamped at v11 on clusterA became permanently stuck because their branch version (11) exceeded the active namespace version (2), causing all mutable-state-version checks to fail.

**Recovery path confirmed by Yu Xia (Temporal server engineer):**
1. Remove and re-add clusterB registration on clusterA (`operator cluster remove` + re-add) — ClusterId is immutable, upsert will not fix a mismatch. Global namespace is briefly unavailable during this window.
2. Check namespace-replication DLQ for missed v11/v12 namespace updates (`tdbg dlq read/merge --dlq-type namespace --cluster cluster0300s3`). Safe to merge; worst case is re-DLQ. Check the original DLQ error first if possible.
3. Issue double failover **from clusterB** to advance FailoverVersion above 12: `cluster0300s3 → cluster0000s7` (v21 → v22). Must be from clusterB — issuing from clusterA at v2 would compute v11 which is stale and would be rejected, likely causing another desync.
4. Fixing FailoverVersion should automatically unpause stuck workflows. Without fixing it, terminate + restart still gets stuck.

**Research needed for playbook:**
- Trace server code path for the mutable-state-version vs namespace-active-version check (how exactly workflows become stuck)
- Understand how the original split-brain occurred — what sequence of events leads to ClusterId divergence after a reinstall
- Validate the remove+re-add procedure against source code (what state is cleaned up, what is preserved, what the brief unavailability window looks like)
- Confirm DLQ merge safety — what causes namespace updates to DLQ in the first place
- Document the FailoverVersion increment formula and why the failover must originate from the cluster with the highest known version

**Source:** Temporal community Slack thread, AvtorPaka + Yu Xia, 2026-06-16.

---

## Overview

This document outlines the planned Grafana alert rules for the Temporal SDK dashboards. The goal is to provide a set of importable, generic alert rules that any self-hosted Temporal operator can use as a starting point for SDK-side observability.

**Scope:** All four SDK reporters — Java Micrometer, Java OTel, Go SDK, Core SDK (TypeScript, Python, .NET, Ruby). Nexus alerts are deferred until Nexus support is better understood.

**One alert file per reporter.** Metric names differ across reporters: Micrometer appends `_total` to counters and `_seconds` to histograms; OTel uses raw SDK names; Go SDK uses PascalCase `status_code` label values; Core SDK metric names match Go SDK names. Combining these into one file via PromQL OR expressions would be brittle and confusing. Instead, each reporter gets its own provisioning YAML — operators drop in the file that matches their setup:
- `temporal-sdk-java-micrometer-alerts.yaml` — Java Micrometer reporter
- `temporal-sdk-java-otel-alerts.yaml` — Java OTel reporter (metric names without Micrometer suffixes)
- `temporal-sdk-go-alerts.yaml` — Go SDK
- `temporal-sdk-core-alerts.yaml` — Core SDK (TypeScript, Python, .NET, Ruby)

**One README, four YAMLs.** The README (`metrics/alerts/sdk/README.md`) is the single reference for all alerts. It shows every alert with its description, severity, and for each SDK: the panel it maps to and the correct PromQL query for that reporter. This lets operators see at a glance what to deploy regardless of which SDK they use.

**The alerts are secondary to the runbooks.** Each alert is accompanied by a runbook describing what it detects, why it matters, which dashboard panels to check, and what actions to take.

---

## Design Principles

**Same structure as server alerts.** Planning → Essential Set → YAML + runbooks. Same annotation schema, same two-tier severity (Critical / Warning), same PromQL rate interval (`5m`).

**SDK-side, not server-side.** These alerts fire based on what the SDK worker observes and reports. They complement server alerts — a server alert may tell you the cluster is throttling; an SDK alert tells you a specific worker type or task queue is impacted.

**Java Micrometer naming as baseline.** The primary alert file targets the Java Micrometer reporter. Micrometer appends `_total` to counters and `_seconds` to histogram buckets. OTel variants use raw SDK metric names. Go SDK uses PascalCase `status_code` labels. Each naming difference is called out per alert.

**`or vector(0)` for conditionally-emitted metrics.** `temporal_worker_task_slots_available` is only emitted by fixed-size slot suppliers — resource-based tuner deployments never emit it. Rather than using `noDataState`, alerts on this metric use `(metric or vector(0)) == 0` so Grafana always has a value to evaluate. The alert fires when slots are zero for the full `for` duration — a momentary zero does not trigger. Users on resource-based tuners who do not want this alert simply disable it. This keeps the expression simple and consistent with all other alerts.

**No `$__rate_interval` in alerts.** Alert rules evaluate outside of a dashboard time range context. All expressions use a hardcoded `5m` rate interval. This safely covers scrape intervals from 15s to 60s.

**Per-status-code severity in PromQL, not dashboard thresholds.** Grafana panel thresholds cannot distinguish severity by label value. Per-status-code severity (e.g. `NOT_FOUND` on respond ops = Critical, `ALREADY_EXISTS` = Warning) is encoded in separate alert rules with PromQL label filters.

**Task queue scoped.** Where meaningful, alerts break down by `task_queue` so operators can identify which worker pool is impacted.

---

## Deliverables

| Deliverable | Description | Status |
|---|---|---|
| `planning.md` | This document — full alert inventory + essential set | ✅ Complete |
| `temporal-sdk-java-micrometer-alerts.yaml` | Grafana provisioning YAML for Java Micrometer reporter — drop into `provisioning/alerting/` | ✅ Complete |
| `temporal-sdk-java-otel-alerts.yaml` | Grafana provisioning YAML for Java OTel reporter | ✅ Complete |
| `temporal-sdk-go-alerts.yaml` | Grafana provisioning YAML for Go SDK | ✅ Complete |
| `temporal-sdk-core-alerts.yaml` | Grafana provisioning YAML for Core SDK (TypeScript, Python, .NET, Ruby) | ✅ Complete |
| `README.md` | Setup guide, Essential Set table, and index of 4 YAML files. Full alert inventory in `alerts-index.md`. | ✅ Complete |
| `alerts-index.md` | Full alert index with all 36 alerts, descriptions, and per-SDK PromQL | ✅ Complete |
| `runbooks/` | One markdown runbook per essential alert (23 runbooks) | ✅ Complete |

---

## Implementation Notes

**Datasource:** Referenced by UID. The Micrometer and OTel reporters both push to the same Prometheus instance. Operators update a single `datasourceUid` value.

**Dashboard UIDs for panel links:**
- Java Micrometer: `temporal-sdk-java-micrometer-v1`
- Java OTel: `temporal-sdk-java-otel-v1`
- Go SDK: `temporal-sdk-go-v1`
- Core SDK: `temporal-sdk-core-v1`

**Grafana folder:** `Temporal SDK` — separate from the server alert folder `Temporal Server`.

**Group name:** `temporal-sdk-essential`

**Evaluation interval:** `1m`

**Labels on every alert:**
- `severity: critical` or `severity: warning`
- `service: temporal-sdk`
- `component: <worker|grpc|cache>` per alert

**Annotations on every alert:**
- `summary`: one line shown in notification subject
- `description`: what it detects + immediate action + runbook URL
- `__dashboardUid__`: per-alert
- `__panelId__`: per-alert

**Every alert needs a description.** After the full inventory and essential set are finalized, go through every alert and write a description covering: what the alert detects, what it typically means in practice (all plausible root causes), which dashboard panels to cross-check, and what immediate action to take. Description quality is the difference between an alert that helps and one that gets ignored.

**Metric name differences by reporter:**

| SDK metric | Micrometer (Prometheus) | OTel (Prometheus) |
|---|---|---|
| `temporal_request_failure` | `temporal_request_failure_total` | `temporal_request_failure_total` |
| `temporal_workflow_task_execution_failed` | `temporal_workflow_task_execution_failed_total` | `temporal_workflow_task_execution_failed_total` |
| `temporal_worker_task_slots_available` | `temporal_worker_task_slots_available` (gauge, no suffix) | `temporal_worker_task_slots_available` (gauge, no suffix) |
| `temporal_sticky_cache_miss` | `temporal_sticky_cache_miss_total` | `temporal_sticky_cache_miss_total` |
| `temporal_request_latency` histogram | `temporal_request_latency_seconds_bucket` | `temporal_request_latency_seconds_bucket` |
| `temporal_workflow_task_schedule_to_start_latency` histogram | `temporal_workflow_task_schedule_to_start_latency_seconds_bucket` | `temporal_workflow_task_schedule_to_start_latency_seconds_bucket` |

The planning document uses the raw SDK metric name. The YAML will use Micrometer names. OTel variant will be a separate file.

---

## Panel ID Reference

Panel IDs per dashboard for all alert-relevant panels. Use these for `__dashboardUid__` and `__panelId__` annotations in alert YAML.

| Panel | Java Micrometer | Java OTel | Go SDK | Core SDK |
|---|---|---|---|---|
| Requests | 3 | 3 | 3 | 3 |
| Long-Poll Requests | 4 | 4 | 4 | 4 |
| Request Failures | 5 | 5 | 5 | 5 |
| Long-Poll Request Failures | 6 | 6 | 6 | 6 |
| Workflow Failed | 14 | 14 | 14 | 14 |
| WFT Schedule To Start Latency | 22 | 22 | 22 | 22 |
| Activity Schedule To Start Latency | 23 | 23 | 23 | 23 |
| Worker Task Slots Available | 34 | 34 | 34 | 34 |
| Number of Active Pollers | 36 | 36 | 36 | 33 |
| WFT Execution Failed | 47 | 47 | 46 | 46 |
| WFT No Completion | 48 | 48 | 47 | — |
| WFT Heartbeat | 49 | 49 | — | — |
| Sticky Cache Forced Evictions | 55 | 55 | 55 | 55 |
| Activity Execution Failed | 63 | 63 | 63 | 64 |
| Unregistered Activity Invocation | 65 | 65 | 65 | 65 |
| Local Activity Execution Failed | 72 | 72 | 72 | 72 |
| Local Activity Execution Latency | 74 | 74 | 75 | 75 |
| Local Activity Total Execution Latency | 76 | 76 | — | 74 |

> **Dashboard UIDs:** Java Micrometer = `temporal-sdk-java-micrometer-v1`, Java OTel = `temporal-sdk-java-otel-v1`, Go = `temporal-sdk-go-v1`, Core = `temporal-sdk-core-v1`
> **Note on Core SDK panel 74:** The Core SDK dashboard has "Local Activity Total" at panel 74 — verify this maps to `temporal_local_activity_total_execution_latency` when building Core SDK alerts.
> **Note on Go SDK:** No Local Activity Total Execution Latency panel and no WFT Heartbeat panel — alerts 26 and 31 are not applicable to Go SDK.

---

## Alert Inventory

### Severity Legend
- 🔴 **Critical** — Page immediately, immediate action required
- ⚠️ **Warning** — Investigate soon, not urgent

---

### Section 1 — gRPC Request Failures

> Panels: Request Failures (5), Long-Poll Request Failures (6)
> Metric: `temporal_request_failure` / `temporal_long_request_failure`
> Tags: `operation`, `status_code`, `namespace`, `task_queue`

`temporal_request_failure` fires when the server returns a non-OK gRPC status code. Severity depends on the status code and the operation. Note: `UpdateWithStartWorkflowExecution` maps to the gRPC operation name `ExecuteMultiOperation`.

| # | Alert Name | Severity | `for` | Operations / Status Code | Description |
|---|---|---|---|---|---|
| 1a | Request Failure — NOT_FOUND on Workflow Task Respond Operations | 🔴 Critical | 5m | `status_code="NOT_FOUND"`, operations: `RespondWorkflowTaskCompleted`, `RespondWorkflowTaskFailed` | Worker responded with workflow task completion after the server already timed out the workflow task, or the workflow execution is no longer running. If the execution is no longer running, check whether it was explicitly terminated or whether an execution run timeout caused the server to time it out before the worker completed the task. **First, check `temporal_workflow_task_execution_latency`** — if elevated, check `temporal_workflow_task_replay_latency` next. If replay latency is high, this typically indicates too many replays or high latency during replay (check your data converters). If replay latency is normal but WFT execution latency is high, check worker resources (CPU, memory) and consider cold-start issues on the pod that picked up the task (check `identity` in the `WorkflowTaskStarted` event). If WFT execution latency exceeds the WFT timeout (default 10s), the server writes `WorkflowTaskTimedOut` events to history and reschedules the task — this adds delays to `temporal_workflow_endtoend_latency`. If you are running local activities, a WFT timeout will cause re-execution of local activities on the retried task — local activity results are not recorded in history between WFT heartbeats, so re-execution means they run again from scratch; if they are not idempotent this can cause duplicate side effects and real business impact. Also check RESOURCE_EXHAUSTED on respond operations alert (#2b) — on some SDK versions, server throttling on respond operations can cascade into a workflow task timeout. If SDK-side metrics look normal and the execution was not terminated or timed out, check the **Frontend Service Latency** panel (Section 4 — Service Latencies) filtered to `RespondWorkflowTaskCompleted`, and the **Persistence Latencies** panel (Section 3) filtered to `UpdateWorkflowExecution`. |
| 1b | Request Failure — NOT_FOUND on Activity Task Respond Operations | 🔴 Critical | 5m | `status_code="NOT_FOUND"`, operations: `RespondActivityTaskCompleted`, `RespondActivityTaskFailed` | Worker responded with activity task completion or failure after the server already timed out the activity task, or the workflow execution is no longer running. Root causes: (1) activity task timed out — the activity ran longer than its `scheduleToClose` or `startToClose` timeout; (2) workflow execution is no longer running — completed (and workflow code did not sync-wait for activity completion), explicitly terminated, or hit a workflow run timeout; (3) worker restart mid-execution — the in-flight task token is lost, the server reschedules the activity but the original worker may still attempt to respond. **Check `temporal_activity_execution_latency`** for the affected activity type — if elevated and approaching or exceeding your `startToClose` timeout, that is the direct cause. If activity execution latency is unexpectedly high, also check worker CPU and memory utilization, and consider cold-start issues on the pod that picked up the task — identify the worker via the `identity` field in the `ActivityTaskStarted` event, or in the pending activity info if this is not the final attempt. If activity latency looks normal, check the **Frontend Service Latency** panel (Section 4 — Service Latencies) filtered to `RespondActivityTaskCompleted` and `RespondActivityTaskFailed`, and the **Persistence Latencies** panel (Section 3) filtered to `UpdateWorkflowExecution`. |
| 1c | Request Failure — NOT_FOUND on RecordActivityTaskHeartbeat | ⚠️ Warning | 5m | `status_code="NOT_FOUND"`, operation: `RecordActivityTaskHeartbeat` | The server returned NOT_FOUND when the worker attempted to heartbeat a running activity. The most common cause is the activity's `heartbeatTimeout` firing before the worker's next heartbeat call — when this happens, the server cancels the in-flight task and schedules it for retry. A second cause is `startToClose` timeout: if the activity has been running longer than its `startToClose` allows, the server times it out at the history level while the activity is still executing on the worker; the next heartbeat attempt then returns NOT_FOUND. Also check worker CPU utilization — a CPU-starved worker slows down between heartbeat calls, which can push the inter-heartbeat gap past `heartbeatTimeout` even when the activity is making progress. Cross-check RESOURCE_EXHAUSTED on `RecordActivityTaskHeartbeat` (alert #2d) — if the server is throttling heartbeat calls, the effective heartbeat interval increases and can trigger a server-side timeout even when the worker is calling on time. Finally, check history service memory: the last heartbeat details payload is stored in shard mutable state and held in memory for the life of the activity attempt — large heartbeat payloads on high-throughput activity workers can contribute to history host memory pressure. Note: normal workflow-side cancellation returns `CancelRequested=true` in the heartbeat response body, not a gRPC error, so NOT_FOUND on this operation is a reliable signal of timeout or forced closure rather than intentional cancellation. |
| 2a | Request Failure — RESOURCE_EXHAUSTED on Critical User-Facing Operations | 🔴 Critical | 1m | `status_code="RESOURCE_EXHAUSTED"`, operations: `StartWorkflowExecution`, `SignalWithStartWorkflowExecution`, `ExecuteMultiOperation`, `UpdateWorkflowExecution`, `SignalWorkflowExecution` | Server is throttling critical user-facing operations. The SDK will retry these for up to 60 seconds — in many cases this still results in business impact as callers experience elevated latency. If throttling continues past 60 seconds the call fails and the client receives an error which should be handled in application code (log these failures so you can potentially backfill starts, signals, and updates if needed). From the server's perspective these operations are throttled last — if you are seeing RESOURCE_EXHAUSTED on these operations it is a strong indication that there is a significant amount of throttling happening across the cluster. Check the **Resource Exhausted with Cause** panel (Section 6 — Throttling and Limits) — if server-side throttling is elevated, also check the **Persistence Latencies** panel (Section 3) filtered to `CreateWorkflowExecution` and `UpdateWorkflowExecution`. Note: `UpdateWithStartWorkflowExecution` maps to the gRPC operation name `ExecuteMultiOperation` in SDK metrics. |
| 2b | Request Failure — RESOURCE_EXHAUSTED on Respond Operations | 🔴 Critical | 5m | `status_code="RESOURCE_EXHAUSTED"`, operations: `RespondWorkflowTaskCompleted`, `RespondWorkflowTaskFailed`, `RespondActivityTaskCompleted`, `RespondActivityTaskFailed` | Server is throttling workflow and activity task respond operations. This is a leading indicator for alerts #1a and #1b — if the server keeps returning RESOURCE_EXHAUSTED on these operations long enough, the task will eventually time out and the server will reschedule it, causing workflow execution delays. Check the **Resource Exhausted with Cause** panel (Section 6 — Throttling and Limits) and the **Persistence Latencies** panel (Section 3) filtered to `UpdateWorkflowExecution` — high persistence latency is a common root cause of cascading throttling on respond operations. Consider increasing namespace RPS limits via dynamic config (`frontend.namespaceRPS`). |
| 2c | Request Failure — RESOURCE_EXHAUSTED on Long-Poll Operations | ⚠️ Warning | 5m | `status_code="RESOURCE_EXHAUSTED"`, metric: `temporal_long_request_failure`, operations: `PollWorkflowTaskQueue`, `PollActivityTaskQueue`, `GetWorkflowExecutionHistory` (long-poll / waitNewEvent=true variant) | **Uses `temporal_long_request_failure`, not `temporal_request_failure`.** Poll operations (`PollWorkflowTaskQueue`, `PollActivityTaskQueue`) and the long-poll variant of `GetWorkflowExecutionHistory` (used by clients sync-waiting on workflow completion) go through the SDK's long-poll interceptor and are recorded in `temporal_long_request_failure`. RESOURCE_EXHAUSTED on these operations is most commonly caused by the namespace concurrent poller limit being exceeded (`ConcurrentLimit` cause), but can also be caused by the namespace RPS limit (`RpsLimit` cause) if the poll rate itself exceeds `frontend.namespaceRPS`. Impact depends on which operation is throttled: throttling on `PollWorkflowTaskQueue` or `PollActivityTaskQueue` causes workers to back off and poll less frequently, causing tasks to queue on the server and increasing schedule-to-start latency; throttling on `GetWorkflowExecutionHistory` (long-poll) causes delays for any client sync-waiting on a workflow result. Note: RESOURCE_EXHAUSTED on the non-long-poll `GetWorkflowExecutionHistory` (used during workflow task replay to fetch history pages) goes through `temporal_request_failure` and is covered by alert #2d. On the server side concurrent poller throttling shows up as RESOURCE_EXHAUSTED with cause `ConcurrentLimit` — check the **Resource Exhausted with Cause** panel (Section 6 — Throttling and Limits) filtering by that cause. Cross-check schedule-to-start latency alerts (#18, #19) and the All Pollers Disconnected alert (#15). Also check the **Total Concurrent Pollers** and **Max Concurrent Pollers per Frontend Pod** panels (Section 15 — Pollers) — `frontend.namespaceCount` is a per-instance limit, so losing frontend pods reduces the effective total limit proportionally; pod count itself is monitored via your infrastructure observability stack. If the limit is consistently hit, increase `frontend.namespaceCount` (per instance) or `frontend.globalNamespaceCount` (global) via dynamic config. |
| 2d | Request Failure — RESOURCE_EXHAUSTED on Other Operations | ⚠️ Warning | 5m | `status_code="RESOURCE_EXHAUSTED"`, all operations not covered by alerts 2a, 2b, 2c, or #3 | Server is throttling SDK operations that are automatically retried. Sustained throttling on these operations can delay workflow end-to-end execution time and cause business impact even if errors are not immediately visible to callers. Notably, throttling on `GetWorkflowExecutionHistory` (used during workflow task replay to fetch history pages) will directly increase `temporal_workflow_task_replay_latency` and `temporal_workflow_task_execution_latency` — if sustained long enough this can cascade into WFT timeouts. Cross-check `temporal_workflow_task_replay_latency` on the dashboard if this alert fires alongside alert #1a. Check the **Resource Exhausted with Cause** panel (Section 6 — Throttling and Limits) to understand the scope of throttling. Consider increasing namespace RPS limits via dynamic config (`frontend.namespaceRPS`). |
| 3 | Worker Startup Throttled | 🔴 Critical | 1m | `status_code="RESOURCE_EXHAUSTED"`, operations: `GetSystemInfo`, `DescribeNamespace` | Server is throttling `GetSystemInfo` and `DescribeNamespace`, which are called by all SDK workers at startup. The server treats these as low priority under load and throttles them before other operations. The SDK retries for ~60 seconds — if throttling is sustained for that long the worker will fail to start and will not process any tasks. Check the **Resource Exhausted with Cause** panel (Section 6 — Throttling and Limits) and the **Total RPS** panel (Section 1 — Cluster Throughput). |
| 4 | Request Failure — Authentication / Authorization Failure | 🔴 Critical | 5m | `status_code=~"PERMISSION_DENIED\|UNAUTHENTICATED"`, any operation | Server is returning authentication or authorization errors for SDK operations. Root causes: missing, expired, or revoked credentials, or misconfigured auth policy. Can also occur transiently during server restarts — adjust `for` as needed for your environment. Check your credential configuration and auth policy. On the server side, cross-check the **Unauthorized Requests** panel (Section 19 — Authorization) to confirm the server is seeing the denied requests, and the **Authorization System Failures** panel — a non-zero value there indicates the auth plugin itself is failing (crash, misconfiguration, or network failure to an external auth service), which is a more urgent signal than a simple credential problem. |
| 5 | Request Failure — UNIMPLEMENTED | 🔴 Critical | 2m | `status_code="UNIMPLEMENTED"`, any operation | Server is returning UNIMPLEMENTED for an SDK operation. By the time a worker reaches this point it has already successfully called `GetSystemInfo` and `DescribeNamespace`, so this is unlikely to be a simple version mismatch. Check your server deployment — wrong binary deployed, corrupted deployment, or a service routing issue sending requests to the wrong handler are the most likely causes. Also check if you are running a very old SDK version that is calling an API that has been removed or significantly changed on the server. Cross-check the **Service Panics** panel (Section 5 — Service Requests and Errors) — a wrong binary or corrupted deployment often produces panics alongside UNIMPLEMENTED responses. |
| 6 | Request Failure — INTERNAL | 🔴 Critical | 2m | `status_code="INTERNAL"`, any operation | Server is returning INTERNAL errors for SDK operations. Short bursts are expected during server restarts. If sustained, this indicates infrastructure-level issues. Check the **Service Errors by Namespace** panel (Section 5 — Service Requests and Errors), the **Service Panics** panel (Section 5 — any panic is a critical signal), the **Persistence Errors by Namespace and Operation** panel (Section 3), and the **Persistence Availability** panel (Section 3) — sustained persistence errors are a common root cause of INTERNAL responses. Also check server health and deployment. |
| 7 | Request Failure — NOT_FOUND on Signal / Update / Query | ⚠️ Warning | 5m | `status_code="NOT_FOUND"`, operations: `SignalWorkflowExecution`, `UpdateWorkflowExecution`, `QueryWorkflow` | Application is consistently receiving NOT_FOUND on signal, update, or query operations. Root causes differ by operation. For `SignalWorkflowExecution` and `UpdateWorkflowExecution`: the target workflow does not exist or has already completed. One possible cause is RESOURCE_EXHAUSTED throttling on these operations (cross-check alert #2a) — if the call was delayed long enough by throttling, the target execution may have completed by the time the request reached the server. For `QueryWorkflow`: the workflow does not exist, was closed and removed by namespace retention policy, or the client has set `QueryRejectCondition` to reject queries on non-running executions (e.g. `QUERY_REJECT_CONDITION_NOT_OPEN`) — in which case this is expected behavior and this alert may not be applicable. Check your workflow client's query reject condition configuration (`QueryRejectCondition` in Go SDK, `queryRejectCondition` in Java SDK, equivalent option in Core-based SDKs). |
| 8 | Request Failure — ALREADY_EXISTS on StartWorkflowExecution | ⚠️ Warning | 10m | `status_code="ALREADY_EXISTS"`, operation: `StartWorkflowExecution` | The server rejected the start because the workflow ID is already in use. The behavior depends on `WorkflowIdReusePolicy` and `WorkflowIdConflictPolicy`: with the default `ALLOW_DUPLICATE` reuse policy, ALREADY_EXISTS only fires when a workflow with this ID is currently running; with `REJECT_DUPLICATE`, it also fires for completed workflows within the retention period regardless of outcome; with `ALLOW_DUPLICATE_FAILED_ONLY`, it fires when the prior run completed successfully. Root causes: application is retrying starts for a workflow that is still running (workflows running longer than expected), a bug in workflow ID generation producing duplicate IDs, misconfigured `WorkflowIdReusePolicy` for the use case, or RESOURCE_EXHAUSTED throttling on `StartWorkflowExecution` (cross-check alert #2a) — if a start was delayed long enough by throttling, another client may have already reserved that workflow ID. |
| 9 | Request Failure — FAILED_PRECONDITION on QueryWorkflow | ⚠️ Warning | 2m | `status_code="FAILED_PRECONDITION"`, operation: `QueryWorkflow` | No workers polling for the task queue — query cannot be routed to a worker. Cross-check `temporal_num_pollers` and the All Pollers Disconnected alert (#15). |
| 10 | Long-Poll Request Failure | ⚠️ Warning | 5m | Any `temporal_long_request_failure` rate > 0 | Long-poll failures are uncommon and warrant investigation — typically indicates server-side issues. Check the **Service Errors by Namespace** panel (Section 5 — Service Requests and Errors) and the **Persistence Errors by Namespace and Operation** panel (Section 3). For `GetWorkflowExecutionHistory` long-poll failures in particular, also check frontend service CPU utilization via your infrastructure observability stack. |
| 11 | GetWorkflowExecutionHistory Long-Poll Rate Sustained High | ⚠️ Warning | 5m | `rate(temporal_long_request_total{operation="GetWorkflowExecutionHistory"}[5m]) > 30` | Workers are polling event history at a sustained high rate. This can mean workers are replaying large amounts of event history, which will elevate `temporal_workflow_task_replay_latency` and `temporal_workflow_task_execution_latency`. Can also indicate the worker cache is too small and experiencing high eviction rates — cross-check alert #16 (Sticky Cache Forced Evictions High). At high rates this puts pressure on the database. Also check frontend CPU on the server side. |
| 12 | RespondWorkflowTaskCompleted Rate Dropped to Zero | 🔴 Critical | 5m | `(rate(temporal_request_total{operation="RespondWorkflowTaskCompleted", task_queue="<configured>"}[5m]) or vector(0)) == 0` | Workflow task completions have dropped to zero for this task queue — workflow progress has fully halted. First check if `PollWorkflowTaskQueue` rate is also zero, which would indicate workers are down entirely; if so, check auth failure alerts (#4) and worker startup throttling alerts (#3). If polling is active but completions are zero, check whether `temporal_workflow_task_execution_failed` is elevated — workers may be failing tasks instead of completing them. Also check WFT schedule-to-start latency alerts (#18, #19) for a timeout cascade. Check RESOURCE_EXHAUSTED on critical operations alert (#2a) — if Start, Signal, or Update operations are throttled, no new workflow tasks may be getting created. Check overall cluster health on the server dashboard: **Service Errors by Namespace** (Section 5 — Service Requests and Errors), **Persistence Availability** (Section 3), and **Resource Exhausted with Cause** (Section 6 — Throttling and Limits). Scoped per `task_queue` — configure which task queues to monitor. |
| 13 | RespondActivityTaskCompleted Rate Dropped to Zero | 🔴 Critical | 5m | `(rate(temporal_request_total{operation="RespondActivityTaskCompleted", task_queue="<configured>"}[5m]) or vector(0)) == 0` | Activity task completions have dropped to zero for this task queue. First check if `PollActivityTaskQueue` rate is also zero — if polling has stopped, workers are down entirely; check auth failure alerts (#4) and worker startup throttling alerts (#3). If polling is active, check RESOURCE_EXHAUSTED on poll operations alert (#2c) — if the server is throttling `PollActivityTaskQueue`, workers back off and pick up tasks less frequently, which can suppress completions even when workers are alive. Check RESOURCE_EXHAUSTED on respond operations alert (#2b) — if the server is throttling `RespondActivityTaskCompleted`, the SDK only increments `temporal_request` after a successful response, so sustained throttling will suppress this counter directly. Also check for server-side connectivity problems: INTERNAL errors (#6) and UNIMPLEMENTED (#5) both indicate the server is not processing SDK operations normally. Finally, check for any workflow task-level problems that would prevent new activities from being scheduled in the first place — RESOURCE_EXHAUSTED on critical user-facing operations (#2a) preventing new workflow instances from starting, RESOURCE_EXHAUSTED on `RespondWorkflowTaskCompleted` (#2b) preventing WFTs from completing, NonDeterminismError (#21) or GrpcMessageTooLarge (#22) WFT failures leaving workflows permanently stuck, WFT schedule-to-start latency alerts (#18, #19) indicating workflow workers are not picking up tasks, and alert #1a (NOT_FOUND on WFT respond operations) indicating WFT timeouts. If any of these are firing, workflow code is not running or not starting and no new activities will be scheduled regardless of activity worker health. Check overall cluster health on the server dashboard: **Service Errors by Namespace** (Section 5 — Service Requests and Errors), **Persistence Availability** (Section 3), and **Resource Exhausted with Cause** (Section 6 — Throttling and Limits). Scoped per `task_queue` — configure which task queues to monitor. |

> **Reusable pattern — Operation Rate Dropped to Zero:** Alerts 12 and 13 use the pattern `(rate(metric{operation="X", task_queue="Y"}[5m]) or vector(0)) == 0` with `for: 5m`. Operators can apply this same pattern to any operation they want to track — for example `RespondWorkflowTaskFailed`, `RespondActivityTaskFailed`, or any other operation that should be consistently active for a given task queue. Configure the `task_queue` label matcher to scope the alert to task queues that are expected to have continuous activity.

---

### Section 2 — Worker Lifecycle

> Panels: Worker Task Slots Available (34), Number of Active Pollers (36)
> Metrics: `temporal_worker_task_slots_available`, `temporal_num_pollers`

| # | Alert Name | Severity | Condition |
|---|---|---|---|
| 14 | Worker Task Slots Exhausted | 🔴 Critical | `(temporal_worker_task_slots_available{worker_type=~"WorkflowWorker\|ActivityWorker\|LocalActivityWorker"} or vector(0)) == 0` for 2m, by `namespace`, `worker_type` and `task_queue`. Uses `or vector(0)` so the alert evaluates even when the metric is absent — resource-based tuner deployments never emit `temporal_worker_task_slots_available`, so disable this alert if you use the resource-based tuner. Impact and remediation differ by `worker_type`: **`WorkflowWorker`** — All workflow task execution slots are occupied — existing tasks are taking too long to complete and not releasing their slots back to the pool. Check worker pod/container CPU — high CPU slows WFT execution directly and keeps slots occupied longer. Check `temporal_workflow_task_execution_latency` — sustained high values confirm something is holding slots: slow deterministic code, heavy computation inside workflow functions, or a blocked executor. For Python SDK workers in particular, verify that no `async def` workflow code is blocking the event loop — a single blocking call starves all coroutines on that worker and can exhaust the slot pool completely. Cross-check RESOURCE_EXHAUSTED on `RespondWorkflowTaskCompleted` (#2b) — if the server is throttling WFT completions, slots are not released until the respond call succeeds. Also check the GrpcMessageTooLarge alert (#22) — an oversized WFT response causes the SDK to fail the task rather than complete it, holding the slot until the retry cycle resolves. Cross-check WFT schedule-to-start latency alerts (#18, #19). Immediate action: scale up workflow worker pods or increase `maxConcurrentWorkflowTaskExecutionSize`. **`ActivityWorker`** — All activity execution slots are occupied — existing activities are taking too long to complete and not releasing their slots. Most commonly this means activity code is blocking and not returning: check `temporal_activity_execution_latency` for the affected task queue and activity type — sustained high values confirm activities are holding slots far longer than expected. Check worker pod/container CPU — high CPU slows activity execution directly. Cross-check RESOURCE_EXHAUSTED on `RespondActivityTaskCompleted` (#2b) — if the server is throttling activity completions, the SDK cannot release the slot until the respond call succeeds. If you intentionally run a single-slot long-running activity worker, this alert is expected to fire — disable it for that task queue and worker type. Immediate action: scale up activity worker pods or increase `maxConcurrentActivityExecutionSize`. **`LocalActivityWorker`** — All local activity execution slots are occupied — the worker cannot schedule or pick up new local activities. This is dangerous: local activities run within the WFT execution loop, so blocked slots hold up the entire WFT. The SDK responds by sending repeated WFT heartbeats to keep the WFT alive — cross-check alert #25 (WFT Heartbeat Rate Elevated). If local activity slots remain exhausted and heartbeating continues past the server's WFT heartbeat timeout (`history.workflowTaskHeartbeatTimeout`, default 30 minutes), the server will time out the heartbeat WFT and reschedule it on the normal task queue — at which point the local activities will be re-executed from scratch on the retried task, with potential duplicate side effects if they are not idempotent (cross-check alert #1a). The root cause is almost always local activity code that is blocking and not returning — investigate what the activities are waiting on. If they are calling downstream services, check whether those services are throttling or slow, as increasing slot counts in that scenario would only increase downstream pressure and make the problem worse. Also check worker pod/container CPU — high CPU slows local activity execution directly. |
| 15 | All Pollers Disconnected | 🔴 Critical | `temporal_num_pollers == 0` for 5m, by `namespace`, `worker_type` and `task_queue`. No active pollers for this `worker_type` / task queue — workers have stopped polling entirely. Tasks are accumulating on the server with no one to process them. The 5m `for` avoids false positives during normal pod restarts and rolling deploys — if it fires, the outage has been sustained long enough to be a real problem. **First check alert #14 (Worker Task Slots Exhausted)** — if all task slots are occupied, the SDK blocks before issuing the next poll (both Go and Java SDKs block on slot acquisition before incrementing `temporal_num_pollers`), so the gauge drops to zero as a secondary effect. In that case #14 is the root cause and #15 is a symptom. If slots are not exhausted, check auth failure alert (#4) — expired or revoked credentials are a common cause of pollers disconnecting and failing to reconnect. On the server side, cross-check the **Unauthorized Requests** and **Authorization System Failures** panels (§19 — Authorization) to confirm whether the server is rejecting connections. Also check worker startup throttling alert (#3) — if `GetSystemInfo` or `DescribeNamespace` are being throttled on reconnect, workers will fail to re-establish polling after a restart. Check INTERNAL errors alert (#6) and UNIMPLEMENTED alert (#5) — both indicate the server is not accepting SDK operations normally, which will prevent pollers from reconnecting. Finally check your worker pod/container health via your infrastructure observability stack — a crashed or OOM-killed worker process will show zero pollers until it restarts and reconnects. |

---

### Section 3 — Worker Cache

> Panels: Sticky Cache Hit (52), Sticky Cache Miss (53), Sticky Cache Size (54), Sticky Cache Forced Evictions (55)
> Metrics: `temporal_sticky_cache_hit`, `temporal_sticky_cache_miss`, `temporal_sticky_cache_size`, `temporal_sticky_cache_total_forced_eviction`

| # | Alert Name | Severity | Condition |
|---|---|---|---|
| 16 | Sticky Cache Forced Evictions High | ⚠️ Warning | `rate(temporal_sticky_cache_total_forced_eviction[5m]) > 30` for 5m. Some forced evictions are normal — the sticky cache is bounded by resources and cannot hold every concurrent workflow execution indefinitely. A sustained high rate is the signal. If worker pod memory utilization is low when this alert fires, the cache is likely undersized by count. Check `temporal_sticky_cache_size` (gauge) against `maxWorkflowCacheSize` — if the gauge is at or near the limit, increase `maxWorkflowCacheSize`. However, the cache can fill its memory budget before hitting the numeric slot limit if individual workflow executions carry unusually large in-memory state — for example, workflows that accumulate large data in their execution state over many events. In that case pod memory pressure forces evictions even though `temporal_sticky_cache_size` appears below the numeric limit. Always check **per-pod** memory utilization via your infrastructure observability stack — do not average across all workers polling the same task queue, since sticky assignment is per-pod and one pod may hold far more or heavier executions than others. Long-running executions also contribute to sustained cache pressure: the longer a workflow runs, the longer its slot is occupied in the cache, reducing effective cache capacity for shorter-lived workflows running on the same workers. Also check alert #21 (WFT Execution Failed — NonDeterminismError) — NDE failures and WFT timeouts cause the server to reschedule tasks to the normal (non-sticky) task queue, forcing cache evictions on the receiving worker and triggering cache churn across the fleet even when pod memory is not the limiting factor. Each eviction means the next WFT for that execution requires a full cold replay — all history pages fetched from the server and re-executed from scratch. Cross-check the **WFT Execution Latency** panel — a rising eviction rate directly drives higher WFT execution latency. |

> **Panels not alerted:** Sticky Cache Hit — no meaningful generic threshold; Sticky Cache Miss — captured indirectly via forced evictions; Sticky Cache Size — gauge with no meaningful generic threshold.

---

### Section 4 — Workflow Executions

> Panels: Workflow Failed (14)
> Metric: `temporal_workflow_failed`

| # | Alert Name | Severity | Condition |
|---|---|---|---|
| 17 | Workflow Execution Failed Rate Elevated | ⚠️ Warning | `rate(temporal_workflow_failed[5m]) > 20` for 3m, by `namespace`, `workflow_type` and `task_queue`. `temporal_workflow_failed` is incremented when a workflow execution closes with `FAILED` status — this includes unhandled errors, explicit `ApplicationError`/`ApplicationFailure` calls, and unhandled panics that bubble up to the workflow root. `ApplicationError`/`ApplicationFailure` instances marked with category `BENIGN` are excluded and do not increment this counter. By default, WFT-level failures such as NonDeterminismError go to `temporal_workflow_task_execution_failed` (covered by alerts #21–#23) and do not increment this counter. However, if you configure `WorkflowPanicPolicy=FailWorkflow` (Go SDK) or add an exception type to `setFailWorkflowExceptionTypes` (Java SDK), those failures will be converted to workflow-level failures and **will** increment `temporal_workflow_failed` — unless the failure is marked benign. A sustained elevated rate means workflow executions are terminating with failure at an abnormal rate. This metric carries `workflow_type` and `task_queue` labels but does **not** carry `workflow_id` — at scale you cannot identify the specific failing executions from this metric alone. Check your worker logs, which should include the workflow ID associated with each failure, to identify the specific executions. Then check the `failure` field in `WorkflowExecutionFailed` history events for the error message and type. If your application intentionally fails workflow executions by design — saga compensations, explicit failure propagation, error signaling patterns — mark those `ApplicationError`/`ApplicationFailure` instances with category `BENIGN`. Both Go and Java SDKs skip the counter for benign failures, making this alert exclusively track unexpected failures without needing per-`workflow_type` threshold tuning. |

---

### Section 5 — Schedule To Start Latencies

> Panels: WFT Schedule To Start (22), Activity Schedule To Start (23)
> Metrics: `temporal_workflow_task_schedule_to_start_latency`, `temporal_activity_schedule_to_start_latency`

| # | Alert Name | Severity | Condition |
|---|---|---|---|
| 18 | WFT Schedule-To-Start Latency High | ⚠️ Warning | p99 `temporal_workflow_task_schedule_to_start_latency` > 1s for 5m, by `namespace` and `task_queue`. p99 workflow task schedule-to-start latency has been above 1 second for 5 minutes on this namespace and task queue. Check worker pod/container CPU. Check poller counts via `temporal_num_pollers` — if pollers are dropping, cross-check alert #15 (All Pollers Disconnected). Check if poll operations are being rate limited — cross-check alert #2c (RESOURCE_EXHAUSTED on Long-Poll Operations) for `PollWorkflowTaskQueue`. Check for a large workflow task backlog on the server side — see the **Approximate Task Backlog** panel in the server dashboard (Matching service section). |
| 19 | WFT Schedule-To-Start Latency Critical | 🔴 Critical | p99 `temporal_workflow_task_schedule_to_start_latency` > 5s for 5m, by `namespace` and `task_queue`. p99 workflow task schedule-to-start latency has been above 5 seconds for 5 minutes on this namespace and task queue. This is severely impacting workflow execution end-to-end latencies. Check worker pod/container CPU. Check poller counts via `temporal_num_pollers` — if pollers are dropping, cross-check alert #15 (All Pollers Disconnected). Also check the **Total Concurrent Pollers** panel (§15 — Pollers) in the server dashboard — if the concurrent poller count for workflow tasks has dropped, your worker pool may have shrunk and horizontal scaling is needed. Check if poll operations are being rate limited — cross-check alert #2c (RESOURCE_EXHAUSTED on Long-Poll Operations) for `PollWorkflowTaskQueue`. Check for a large workflow task backlog on the server side — see the **Approximate Task Backlog** panel in the server dashboard (Matching service section). |
| 19b | WFT Schedule-To-Start Latency Severely Elevated | 🔴 Critical | p99 `temporal_workflow_task_schedule_to_start_latency` > 30m for 5m, by `namespace` and `task_queue`. p99 workflow task schedule-to-start latency has been above 30 minutes for 5 minutes on this namespace and task queue. This is severe degradation — workflow tasks are not being picked up and at large scale this can cause a very large task backlog that puts significant pressure on the server database. Check worker pod/container CPU. Check poller counts via `temporal_num_pollers` — if pollers are dropping, cross-check alert #15 (All Pollers Disconnected). Also check the **Total Concurrent Pollers** panel (§15 — Pollers) in the server dashboard — if the concurrent poller count for workflow tasks has dropped, your worker pool may have shrunk and horizontal scaling is needed. Check if poll operations are being rate limited — cross-check alert #2c (RESOURCE_EXHAUSTED on Long-Poll Operations) for `PollWorkflowTaskQueue`. Check for a large workflow task backlog on the server side — see the **Approximate Task Backlog** panel in the server dashboard (Matching service section). |
| 20 | Activity Schedule-To-Start Latency High | ⚠️ Warning | p99 `temporal_activity_schedule_to_start_latency` > 10s for 5m, by `namespace`, `activity_type` and `task_queue`. p99 activity task schedule-to-start latency has been above 10 seconds for 5 minutes on this namespace and task queue. Check worker pod/container CPU. Check poller counts via `temporal_num_pollers` — if pollers are dropping, cross-check alert #15 (All Pollers Disconnected). Check if poll operations are being rate limited — cross-check alert #2c (RESOURCE_EXHAUSTED on Long-Poll Operations) for `PollActivityTaskQueue`. Check for a large activity task backlog on the server side — see the **Approximate Task Backlog** panel in the server dashboard (Matching service section). Also check activity failure rates — cross-check alert #26 (Activity Execution Failed Rate Elevated), as high churn of unexpected activity failures causes additional retry tasks to be created and can contribute to backlog growth. Note: some activity types are expected to have high schedule-to-start by design — tune the threshold or disable per `activity_type` if this is intentional. |
| 20b | Activity Schedule-To-Start Latency Severely Elevated | 🔴 Critical | p99 `temporal_activity_schedule_to_start_latency` > 30m for 5m, by `namespace`, `activity_type` and `task_queue`. p99 activity task schedule-to-start latency has been above 30 minutes for 5 minutes on this namespace and task queue. This is severe degradation — activity tasks are not being picked up and at large scale this can cause a very large task backlog that puts significant pressure on the server database. Check worker pod/container CPU. Check poller counts via `temporal_num_pollers` — if pollers are dropping, cross-check alert #15 (All Pollers Disconnected). Also check the **Total Concurrent Pollers** panel (§15 — Pollers) in the server dashboard — if the concurrent poller count for activity tasks has dropped, your worker pool may have shrunk and horizontal scaling is needed. Check if poll operations are being rate limited — cross-check alert #2c (RESOURCE_EXHAUSTED on Long-Poll Operations) for `PollActivityTaskQueue`. Check for a large activity task backlog on the server side — see the **Approximate Task Backlog** panel in the server dashboard (Matching service section). Also check activity failure rates — cross-check alert #26 (Activity Execution Failed Rate Elevated), as high churn of unexpected activity failures causes additional retry tasks to be created and can contribute to backlog growth. |

---

### Section 6 — Workflow Task Info

> Panels: WFT Execution Failed (47), WFT No Completion (48), WFT Heartbeat (49)
> Metrics: `temporal_workflow_task_execution_failed`, `temporal_workflow_task_no_completion`, `temporal_workflow_task_heartbeat`

| # | Alert Name | Severity | `for` | Condition |
|---|---|---|---|---|
| 21 | WFT Execution Failed — NonDeterminismError | 🔴 Critical | 1m | `rate(temporal_workflow_task_execution_failed{failure_reason="NonDeterminismError"}[5m]) > 0`, by `namespace`, `workflow_type` and `task_queue`. A workflow's replay produced a different command sequence than what is recorded in history. Workflow executions affected by this error are not progressing — the server continuously retries the workflow task, putting additional pressure on workflow workers. Affected executions remain in running status by default, prolonging their end-to-end execution time indefinitely. This metric does not carry `workflow_id` — check worker logs which will log the NDE with the associated workflow ID. In the Temporal UI, affected executions can also be found by querying the `TemporalReportedProblems` search attribute, which the server sets on executions experiencing repeated workflow task failures. NDE is always a code bug. Investigate why the non-determinism occurred — common causes include: adding, removing, or reordering workflow commands without a versioning guard; changing activity or timer parameters in existing workflow code; or deploying incompatible worker code against in-flight executions. Once the root cause is identified and a fix is deployed, affected executions will resume on their next workflow task retry automatically. |
| 22 | WFT Execution Failed — GrpcMessageTooLarge | 🔴 Critical | 1m | `rate(temporal_workflow_task_execution_failed{failure_reason="GrpcMessageTooLarge"}[5m]) > 0`, by `namespace`, `workflow_type` and `task_queue`. The WFT response payload exceeded the gRPC message size limit — this can be rejected by the gRPC library on the SDK side, a proxy or load balancer in the path (e.g. Envoy), or the gRPC library on the server side on receive. Because the Temporal server history service never saw the original `RespondWorkflowTaskCompleted` request, the SDK explicitly sends a follow-up `RespondWorkflowTaskFailed` with cause `WORKFLOW_TASK_FAILED_CAUSE_GRPC_MESSAGE_TOO_LARGE` to clean up the execution. The server then terminates the execution in response — ending it with `TERMINATED` status with no retry. Also check the **Workflow Terminate** panel in the server dashboard — it tracks the rate of terminated executions and will show a spike when this is happening at scale. This metric does not carry `workflow_id` — check worker logs for the affected workflow IDs. Root causes: oversized activity or workflow inputs/outputs, excessive signal or update accumulation in history, or too many commands batched in a single WFT response. Investigate which part of the WFT response is oversized — the fix depends entirely on the root cause. Terminated executions must be restarted manually if needed. |
| 23 | WFT Execution Failed — WorkflowError | ⚠️ Warning | 2m | `rate(temporal_workflow_task_execution_failed{failure_reason="WorkflowError"}[5m]) > 20`, by `namespace`, `workflow_type` and `task_queue`. Workflow task failed due to an error in workflow code — this can be an intermittent failure, a panic, or a non-Temporal exception thrown inside the workflow function. The server will retry the workflow task, which may result in the same failure repeatedly until the issue is resolved. Check worker logs — this metric does not carry `workflow_id`, so worker logs are needed to identify the affected executions. Investigate and fix the root cause, then redeploy workers. |
| 24 | ~~WFT No Completion Rate Elevated~~ | — | — | **Removed** — `temporal_workflow_task_no_completion` is defined in Go SDK constants but never incremented anywhere in the SDK source. Not emitting — do not create an alert for this metric. |
| 25 | WFT Heartbeat Rate Elevated | ⚠️ Warning | 5m | `rate(temporal_workflow_task_heartbeat[5m]) > 20`, by `namespace`, `workflow_type` and `task_queue`. The SDK is sending workflow task heartbeats to keep the WFT alive while local activities are still running. This means local activities are taking longer than expected — either a single execution is running long or local activities are retrying and the retry chain is accumulating. If heartbeating continues past the server's WFT heartbeat timeout (`history.workflowTaskHeartbeatTimeout`, default 30 minutes), the server will time out the WFT and reschedule it on the normal task queue — at which point local activities will be re-executed from scratch on the retried task, with potential duplicate side effects if they are not idempotent. Cross-check alert #14 (Worker Task Slots Exhausted) for `worker_type=LocalActivityWorker` — if local activity slots are exhausted, that is driving the heartbeating. Also cross-check alerts #29 and #30 (Local Activity Execution Latency and Total Execution Latency Exceeds WFT Heartbeat Timeout) to confirm whether individual attempts or the full retry chain is approaching the 30 minute limit. |

---

### Section 7 — Activity Task Info

> Panels: Activity Execution Failed (63), Unregistered Activity Invocation (65)
> Metrics: `temporal_activity_execution_failed`, `temporal_unregistered_activity_invocation`

| # | Alert Name | Severity | `for` | Condition |
|---|---|---|---|---|
| 26 | Activity Execution Failed Rate Elevated | ⚠️ Warning | 5m | `rate(temporal_activity_execution_failed[5m]) > 20`, by `namespace`, `activity_type` and `task_queue`. Activities are explicitly failing — not timing out but returning failures — at a sustained rate. If this is not expected, investigate activity code for the affected `activity_type`. A high failure rate drives a burst of retry tasks being scheduled, putting additional pressure on activity workers and on the server database if workers cannot keep up with task dispatch and tasks need to be persisted. Cross-check alert #20 (Activity Schedule-To-Start Latency High) — a growing backlog from retry bursts will show up there first. If your use case intentionally fails activities by design — polling patterns, saga compensations, flow control via exceptions — set `ApplicationFailure` category to `BENIGN` on those intentional failures. All SDKs support this and will suppress this metric for those failures, making the alert exclusively track unexpected failures. |
| 27 | Unregistered Activity Invocation | 🔴 Critical | 1m | `rate(temporal_unregistered_activity_invocation[5m]) > 0`, by `namespace`, `activity_type` and `task_queue`. The server dispatched an activity to this worker that the worker does not have registered. Always indicates a code or deployment issue — wrong worker binary, missing activity registration, or routing misconfiguration. Tasks will fail indefinitely until resolved. Note: brief spikes can occur during rolling worker restarts when old worker versions briefly receive tasks for newly registered activity types — if this is expected in your deployment process, consider increasing `for` to 5m or disabling the alert during planned deploys. |

---

### Section 8 — Local Activity Info

> Panels: Local Activity Execution Failed (72), Local Activity Execution Cancelled (73), Local Activity Execution Latency (74), Local Activity Succeed End-to-End Latency (75), Local Activity Total Execution Latency (76)
> Metrics: `temporal_local_activity_execution_failed`, `temporal_local_activity_execution_cancelled`, `temporal_local_activity_execution_latency`, `temporal_local_activity_succeed_endtoend_latency`, `temporal_local_activity_total_execution_latency`

| # | Alert Name | Severity | `for` | Condition |
|---|---|---|---|---|
| 28 | Local Activity Execution Failed Rate Elevated | ⚠️ Warning | 5m | `rate(temporal_local_activity_execution_failed[5m]) > 20`, by `namespace`, `activity_type` and `task_queue`. Local activities are explicitly failing — not timing out but returning failures — at a sustained rate. If this is not expected, investigate local activity code for the affected `activity_type`. High failure rates drive retry churn which can extend the total local activity execution time — if the cumulative retry chain runs past the server's WFT heartbeat timeout (`history.workflowTaskHeartbeatTimeout`, default 30 minutes), the server will time out the WFT and reschedule it, causing local activities to be re-executed from scratch with potential duplicate side effects if not idempotent. Cross-check alert #25 (WFT Heartbeat Rate Elevated) and alert #30 (Local Activity Total Execution Latency Exceeds WFT Heartbeat Timeout) to confirm if this is happening. Also cross-check alert #14 (Worker Task Slots Exhausted) for `worker_type=LocalActivityWorker` — sustained retry churn can consume all available local activity slots, blocking the worker from picking up new local activities entirely. If your use case intentionally fails local activities by design, set `ApplicationFailure` category to `BENIGN` on those intentional failures to suppress this metric. |
| 29 | Local Activity Execution Latency Exceeds WFT Heartbeat Timeout | 🔴 Critical | 5m | `histogram_quantile(0.99, sum by (namespace, activity_type, task_queue, le) (rate(temporal_local_activity_execution_latency_seconds_bucket[5m]))) > 1800`, by `namespace`, `activity_type` and `task_queue`. A single local activity attempt is running longer than the server's WFT heartbeat timeout (default 30 minutes, `history.workflowTaskHeartbeatTimeout`). This is critical: the server will time out the heartbeat WFT and reschedule it on the normal task queue — the local activity will be re-executed from scratch, any pending signals, updates, or other workflow events will be delayed until the retried WFT completes, and overall workflow execution end-to-end latency will increase significantly, causing potentially high business impact. If the local activity is not idempotent, re-execution can also cause duplicate side effects. Cross-check alert #25 (WFT Heartbeat Rate Elevated) to confirm heartbeating is already happening. Local activities cannot heartbeat and are designed for short, fast operations. A local activity running for 30 minutes is a design issue — consider converting it to a regular activity with heartbeating, which is the correct primitive for long-running work. If the local activity must remain a local activity, always set a `scheduleToCloseTimeout` less than 30 minutes so it fails with a timeout error before the server times out the WFT heartbeat, allowing the workflow to handle the failure gracefully rather than having the entire WFT re-executed from scratch. |
| 30 | Local Activity Total Execution Latency Exceeds WFT Heartbeat Timeout | 🔴 Critical | 5m | `histogram_quantile(0.99, sum by (namespace, activity_type, task_queue, le) (rate(temporal_local_activity_total_execution_latency_seconds_bucket[5m]))) > 1800`, by `namespace`, `activity_type` and `task_queue`. Cumulative retry time across all local activity attempts has exceeded the server's WFT heartbeat timeout (default 30 minutes, `history.workflowTaskHeartbeatTimeout`). Same impact as alert #29 — the server will time out the heartbeat WFT and reschedule it on the normal task queue — but in this case individual attempts may be short; it is the full retry chain that has accumulated past 30 minutes. The local activity will be re-executed from scratch, any pending signals, updates, or other workflow events will be delayed, and overall workflow execution end-to-end latency will increase significantly, causing potentially high business impact. If the local activity is not idempotent, re-execution can also cause duplicate side effects. Cross-check alert #25 (WFT Heartbeat Rate Elevated) and alert #28 (Local Activity Execution Failed Rate Elevated) — high failure rates driving retry churn are the most common cause of this alert firing without #29 also firing. Fix the underlying failure causing retries, reduce max retry attempts, or increase backoff. If the work genuinely requires many retries over a long period, convert to a regular activity with heartbeating. |

> **Panels not alerted:** Local Activity Execution Cancelled — expected lifecycle event; Local Activity Succeed End-to-End Latency — user-defined, not alertable generically.

---

## Notes on Alerts Not Planned

| Metric | Reason skipped |
|---|---|
| `temporal_long_request_latency` | Long-poll timeouts are expected behavior (server returns empty after ~60s). Alerting would produce constant false positives. `GetWorkflowExecutionHistory` latency is the one exception but cannot be isolated in this metric without op filtering. |
| `temporal_activity_execution_latency` | User-defined activity duration — LLM/batch use cases routinely run for hours. Not alertable generically. |
| `temporal_activity_succeed_endtoend_latency` | Same as above — includes all retries, entirely user-defined. |
| `temporal_local_activity_execution_latency` | Covered by WFT heartbeat alert (#25) — if LAs consistently run long the heartbeat rate climbs. Direct latency alert would require per-workflow tuning. |
| `temporal_worker_task_slots_used` | Covered by slots available alert (#14) — used + available = max. If available drops to zero, used is at max. Redundant alert. |
| `temporal_worker_start` / `temporal_poller_start` | Event counters with no meaningful rate threshold — spikes are expected on rolling restarts. |
| `temporal_workflow_cancelled` | Expected workflow lifecycle event. Not alertable generically. |
| Nexus metrics | Deferred until Nexus support is better understood. |

## Alerts Added in Review (31–36)

| # | Alert | Severity | Metric | Threshold | `for` |
|---|---|---|---|---|---|
| 31 | Request Latency High on Critical User-Facing Operations | 🔴 Critical | `temporal_request_latency` | p99 > 2s on `StartWorkflowExecution`, `SignalWithStartWorkflowExecution`, `ExecuteMultiOperation`, `SignalWorkflowExecution` | 5m |
| 32 | WFT Execution Latency Critical | 🔴 Critical | `temporal_workflow_task_execution_latency` | p99 > 10s | 5m |
| 33 | Workflow End-to-End Latency High | ⚠️ Warning | `temporal_workflow_endtoend_latency` | user-defined placeholder | 5m |
| 34 | Sticky Cache Disabled | ⚠️ Warning | `temporal_sticky_cache_size` | == 0 | 5m |
| 35 | WFT Replay Latency High | ⚠️ Warning | `temporal_workflow_task_replay_latency` | p99 > 5s | 5m |
| 36 | ContinueAsNew Rate Elevated | ⚠️ Warning | `temporal_workflow_continue_as_new` | rate > 100/s | 5m |

### Alert 31 — Request Latency High on Critical User-Facing Operations

🔴 Critical | `p99 > 2s` for 5m | by `namespace`, `operation`

p99 latency for critical user-facing SDK operations has been above 2 seconds for 5 minutes on this namespace. These operations are synchronous from the caller's perspective — elevated latency here means your application code is blocked waiting on the server response, directly impacting your users and any workflows that depend on these calls completing. The SDK retries on transient errors but does not hide latency — every retry adds to the total wait.

Check the **Frontend Service Latency** panel (§4 — Service Latencies) filtered to the affected operations — if server-side latency is elevated, that is the root cause. Check the [**Persistence Latencies** panel (§3 — Persistence Requests, Latencies and Errors)](../../dashboards/server/temporal-server-readme.md#3-persistence-requests-latencies-and-errors) filtered to `CreateWorkflowExecution` and `UpdateWorkflowExecution` — high persistence latency on these operations is the most common driver of elevated frontend latency for starts and signals. If persistence looks healthy, check frontend pod CPU utilization. Also cross-check RESOURCE_EXHAUSTED on critical operations alert (#2a) — if the server is throttling these operations, the SDK retries add to the total observed latency on this metric.

Note: `UpdateWithStartWorkflowExecution` maps to the gRPC operation name `ExecuteMultiOperation` in SDK metrics.

Query: `histogram_quantile(0.99, sum by (namespace, operation, le) (rate(temporal_request_latency_seconds_bucket{operation=~"StartWorkflowExecution|SignalWithStartWorkflowExecution|ExecuteMultiOperation|SignalWorkflowExecution"}[5m]))) > 2`

### Alert 32 — WFT Execution Latency Critical

🔴 Critical | `p99 > 10s` for 5m | by `namespace`, `task_queue`

p99 workflow task execution latency has been above 10 seconds for 5 minutes on this namespace and task queue. The default WFT timeout is 10 seconds (`history.defaultWorkflowTaskTimeout`, confirmed `primitives/constants.go:16`) — at this latency level the server is actively timing out workflow tasks, writing `WorkflowTaskTimedOut` events to history and rescheduling them on the normal task queue. Each timeout forces a sticky cache eviction on the worker that held the execution — cross-check alert #16 (Sticky Cache Forced Evictions High). If you are running local activities, a WFT timeout will cause local activities to be re-executed from scratch on the retried task — local activity results are not checkpointed between WFT heartbeats, so re-execution means they run again; if they are not idempotent this can cause duplicate side effects and real business impact.

Check worker pod CPU and memory — high CPU is the most common cause of slow WFT execution. Check `temporal_workflow_task_replay_latency` — if replay latency is high, the time is being spent re-executing history rather than running new commands; cross-check alert #35 (WFT Replay Latency High) and alert #16 (Sticky Cache Forced Evictions High). If replay latency is normal, the time is in new command execution — check for blocking calls or heavy computation inside workflow functions. For Python SDK workers, verify that no `async def` workflow code is blocking the event loop. Cross-check RESOURCE_EXHAUSTED on respond operations alert (#2b) — if the server is throttling `RespondWorkflowTaskCompleted`, WFT execution latency inflates because the SDK holds the slot until the respond call succeeds.

Query: `histogram_quantile(0.99, sum by (namespace, task_queue, le) (rate(temporal_workflow_task_execution_latency_seconds_bucket[5m]))) > 10`

### Alert 33 — Workflow End-to-End Latency High

⚠️ Warning | `p99 > <YOUR_SLO_SECONDS>` for 5m | by `namespace`, `workflow_type`, `task_queue` | Not in essential set

p99 workflow end-to-end latency has exceeded your defined threshold on this namespace and task queue. This metric measures the total time from when the workflow execution was started to when it completed — it is entirely workload-dependent and no generic threshold applies. You must set the threshold based on your SLO for the affected `workflow_type`.

**Important:** `temporal_workflow_endtoend_latency` is only emitted when a workflow execution completes. It is not a real-time metric — by the time this alert fires, the slow executions have already finished. For running executions you will not see this signal until they close, which means this alert can be a very late indication of a latency problem. If you need real-time detection of long-running executions, use a server-side visibility query instead — for example `ExecutionStatus = "Running" AND StartTime < now() - <your threshold>` — and alert on the count returned.

A sustained breach here is the composite signal that something upstream is wrong. Cross-check in order: WFT schedule-to-start latency alerts (#18, #19); WFT execution latency alert (#32); activity schedule-to-start latency alerts (#20, #20b); activity execution failed rate alert (#26); sticky cache forced eviction alert (#16). Also check RESOURCE_EXHAUSTED on critical user-facing operations alert (#2a) and respond operations alert (#2b).

Query: `histogram_quantile(0.99, sum by (namespace, workflow_type, task_queue, le) (rate(temporal_workflow_endtoend_latency_seconds_bucket[5m]))) > <YOUR_SLO_SECONDS>`

### Alert 34 — Sticky Cache Disabled

⚠️ Warning | `== 0` for 5m | by `namespace`, `task_queue`

The sticky cache size gauge has been zero for 5 minutes — the worker's sticky execution cache is disabled. Every workflow task for every execution on this worker requires a full cold replay from the beginning of history: all history pages fetched from the server, all commands re-executed from scratch. At any meaningful scale this causes significant pressure on the frontend and persistence layers — cross-check the [**Persistence Latencies** panel (§3)](../../dashboards/server/temporal-server-readme.md#3-persistence-requests-latencies-and-errors) and the `GetWorkflowExecutionHistory` long-poll rate alert (#11). WFT execution latency will also be elevated — cross-check alert #32 (WFT Execution Latency Critical).

This is almost always a misconfiguration. In the Go SDK, setting `maxWorkflowCacheSize` to `0` disables the cache entirely. In the Java SDK, `setStickyQueueScheduleToStartTimeout` of zero or `setMaxConcurrentWorkflowTaskExecutionSize` of zero can have the same effect. Check your worker options and restore a non-zero cache size — the default is 10,000 in the Go SDK.

Query: `sum by (namespace, task_queue) (temporal_sticky_cache_size) == 0`

### Alert 35 — WFT Replay Latency High

⚠️ Warning | `p99 > 5s` for 5m | by `namespace`, `task_queue`

p99 workflow task replay latency has been above 5 seconds for 5 minutes on this namespace and task queue. Replay latency is the time the SDK spends re-executing recorded history commands before reaching the point where new commands can be scheduled. High replay latency directly inflates `temporal_workflow_task_execution_latency` — cross-check alert #32 (WFT Execution Latency Critical). If replay latency is the dominant component of WFT execution latency, the root cause is almost always one of: a very large history (many events accumulated without ContinueAsNew), slow data converter execution during replay, or high sticky cache eviction rate forcing frequent cold replays — cross-check alert #16 (Sticky Cache Forced Evictions High) and alert #34 (Sticky Cache Disabled).

Also cross-check RESOURCE_EXHAUSTED on other operations alert (#2d) filtered to `GetWorkflowExecutionHistory` — if the server is throttling history page fetches during replay, that directly adds to replay latency. Check the [**Persistence Latencies** panel (§3)](../../dashboards/server/temporal-server-readme.md#3-persistence-requests-latencies-and-errors) filtered to `GetWorkflowExecution` — high persistence latency on history reads is a common contributor. Also check the `GetWorkflowExecutionHistory` long-poll rate alert (#11) — a sustained high rate confirms workers are fetching large amounts of history repeatedly. If history size is the root cause, consider using ContinueAsNew to truncate history at logical checkpoints.

Query: `histogram_quantile(0.99, sum by (namespace, task_queue, le) (rate(temporal_workflow_task_replay_latency_seconds_bucket[5m]))) > 5`

### Alert 36 — ContinueAsNew Rate Elevated

⚠️ Warning | `rate > 100/s` for 5m | by `namespace`, `workflow_type`, `task_queue`

ContinueAsNew rate has been above 100 per second for 5 minutes on this namespace and task queue. Each ContinueAsNew closes the current workflow run and immediately starts a new one — from the server's perspective this is a workflow completion followed by a new workflow start, meaning every ContinueAsNew generates a `CreateWorkflowExecution` persistence write alongside the `UpdateWorkflowExecution` write for the closing run. At high rates this puts significant pressure on server persistence and can contribute to elevated latency across the cluster. Check the [**Persistence Latencies** panel (§3)](../../dashboards/server/temporal-server-readme.md#3-persistence-requests-latencies-and-errors) filtered to `CreateWorkflowExecution` and `UpdateWorkflowExecution` — if latency is elevated there, the ContinueAsNew burst is a likely contributor.

**Critically, check whether the high rate is concentrated on the same workflow ID.** All runs for the same workflow ID land on the same history shard — a single workflow ID ContinueAsNew-ing at a very high rate creates a hot shard on the history host, which can cause elevated shard lock latency and persistence contention localized to that shard. Check your worker logs to identify which workflow IDs are generating the ContinueAsNew calls. On the server side, cross-check the **Shard Lock Latency** panel (§2 — History Service) — a hot shard from a single high-CAN workflow ID will show up there as elevated lock latency on a specific pod.

A high ContinueAsNew rate is not always a problem — some use cases are designed around frequent history truncation. Investigate whether the rate is expected for the affected `workflow_type`. If it is not expected, check whether executions are hitting history size limits earlier than anticipated and being forced to ContinueAsNew more frequently than designed — reduce event accumulation per run or increase the ContinueAsNew interval. If the rate is expected but causing persistence pressure, consider throttling the workflow start rate or scaling persistence.

Query: `sum by (namespace, workflow_type, task_queue) (rate(temporal_workflow_continue_as_new_total[5m])) > 100`

---

## Essential Alert Set

From the full inventory above, the following alerts form the Essential Set — the minimum set that provides meaningful coverage across all critical failure modes with no overlap.

> **Note:** This table is a draft — will be finalized after all descriptions are complete and after full renumbering is done. Numbers below reflect the current renumbered inventory.

| # | Alert Name | Section | Panel | Panel ID | Why Essential |
|---|---|---|---|---|---|
| 1a | Request Failure — NOT_FOUND on WFT Respond Operations | 1 — gRPC Request Failures | Request Failures | 5 | Task token gone on WFT respond. Binary and unambiguous — always a problem. |
| 1b | Request Failure — NOT_FOUND on Activity Respond Operations | 1 — gRPC Request Failures | Request Failures | 5 | Task token gone on activity respond. Same reasoning as 1a. |
| 2a | Request Failure — RESOURCE_EXHAUSTED on Critical User-Facing Ops | 1 — gRPC Request Failures | Request Failures | 5 | Throttling on start/signal/update = user-facing errors right now. |
| 2b | Request Failure — RESOURCE_EXHAUSTED on Respond Operations | 1 — gRPC Request Failures | Request Failures | 5 | Throttling on respond ops is a leading indicator for 1a and 1b. |
| 4 | Request Failure — Auth/AuthZ Failure | 1 — gRPC Request Failures | Request Failures | 5 | Credential failure — worker cannot communicate with the server at all. |
| 5 | Request Failure — UNIMPLEMENTED | 1 — gRPC Request Failures | Request Failures | 5 | Server/SDK version mismatch or wrong binary deployed. |
| 14 | Worker Task Slots Exhausted | 2 — Worker Lifecycle | Worker Task Slots Available | 34 | Workers at capacity = tasks queueing server-side. Most important capacity signal. |
| 15 | All Pollers Disconnected | 2 — Worker Lifecycle | Number of Active Pollers | 36 | Complete task processing halt for this worker type and task queue. |
| 16 | Sticky Cache Forced Evictions High | 3 — Worker Cache | Sticky Cache Forced Evictions | 55 | High eviction rate = cold replays on every task — severe throughput degradation. |
| 21 | WFT Execution Failed — NonDeterminismError | 6 — Workflow Task Info | Workflow Task Execution Failed | 47 | Code bug — workflow permanently broken. Any occurrence requires immediate developer attention. |
| 27 | Unregistered Activity Invocation | 7 — Activity Task Info | Unregistered Activity Invocation | 65 | Deployment bug — wrong worker binary or missing registration. Tasks will fail indefinitely. |
| 18 | WFT Schedule-To-Start Latency High | 5 — Schedule To Start | WFT Schedule To Start Latency | 22 | Insufficient workers — leading indicator before slot exhaustion fires. |
| 19 | WFT Schedule-To-Start Latency Critical | 5 — Schedule To Start | WFT Schedule To Start Latency | 22 | Past sticky timeout — workflow tasks being dropped to normal queue. Immediate scaling needed. |

> **Dashboard UID for panel links:** `temporal-sdk-java-micrometer-v1` (Micrometer YAML), `temporal-sdk-java-otel-v1` (OTel YAML), `temporal-sdk-go-v1` (Go YAML), `temporal-sdk-core-v1` (Core YAML)

---

## PromQL Expressions (Essential Set)

> **Note:** Expressions below use Java Micrometer naming (`_total` suffix on counters, `_seconds` on histogram buckets). OTel variants drop those suffixes. Go and Core SDK variants use the same base names as OTel but may differ in `status_code` label casing (PascalCase for Go SDK). Full per-SDK expression table will be built as part of Step 2 (README).

| # | `for` | Expression (Java Micrometer) |
|---|---|---|
| 1a | 5m | `sum by (namespace, status_code, operation) (rate(temporal_request_failure_total{status_code="NOT_FOUND", operation=~"RespondWorkflowTaskCompleted\|RespondWorkflowTaskFailed"}[5m])) > 0` |
| 1b | 5m | `sum by (namespace, status_code, operation) (rate(temporal_request_failure_total{status_code="NOT_FOUND", operation=~"RespondActivityTaskCompleted\|RespondActivityTaskFailed"}[5m])) > 0` |
| 1c | 5m | `sum by (namespace, status_code, operation) (rate(temporal_request_failure_total{status_code="NOT_FOUND", operation="RecordActivityTaskHeartbeat"}[5m])) > 0` |
| 2a | 1m | `sum by (namespace, status_code, operation) (rate(temporal_request_failure_total{status_code="RESOURCE_EXHAUSTED", operation=~"StartWorkflowExecution\|SignalWithStartWorkflowExecution\|ExecuteMultiOperation\|UpdateWorkflowExecution\|SignalWorkflowExecution"}[5m])) > 0` |
| 2b | 5m | `sum by (namespace, status_code, operation) (rate(temporal_request_failure_total{status_code="RESOURCE_EXHAUSTED", operation=~"RespondWorkflowTaskCompleted\|RespondWorkflowTaskFailed\|RespondActivityTaskCompleted\|RespondActivityTaskFailed"}[5m])) > 0` |
| 4 | 5m | `sum by (namespace, status_code, operation) (rate(temporal_request_failure_total{status_code=~"PERMISSION_DENIED\|UNAUTHENTICATED"}[5m])) > 0` |
| 5 | 2m | `sum by (namespace, status_code, operation) (rate(temporal_request_failure_total{status_code="UNIMPLEMENTED"}[5m])) > 0` |
| 14 | 2m | `sum by (namespace, worker_type, task_queue) (temporal_worker_task_slots_available or vector(0)) == 0` |
| 15 | 5m | `sum by (namespace, worker_type, task_queue) (temporal_num_pollers) == 0` |
| 16 | 5m | `sum by (namespace) (rate(temporal_sticky_cache_total_forced_eviction_total[5m])) > 30` |
| 21 | 1m | `sum by (namespace, workflow_type, task_queue, failure_reason) (rate(temporal_workflow_task_execution_failed_total{failure_reason="NonDeterminismError"}[5m])) > 0` |
| 27 | 1m | `sum by (namespace, activity_type, task_queue) (rate(temporal_unregistered_activity_invocation_total[5m])) > 0` |
| 18 | 5m | `histogram_quantile(0.99, sum by (namespace, task_queue, le) (rate(temporal_workflow_task_schedule_to_start_latency_seconds_bucket[5m]))) > 1` |
| 19 | 5m | `histogram_quantile(0.99, sum by (namespace, task_queue, le) (rate(temporal_workflow_task_schedule_to_start_latency_seconds_bucket[5m]))) > 5` |
