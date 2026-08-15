# NOT_FOUND on Activity Respond Operations

**Severity:** Critical
**Component:** activity
**Metric:** `temporal_request_failure` — `status_code=NOT_FOUND`, operations `RespondActivityTaskCompleted` / `RespondActivityTaskFailed`
**Dashboard panel:** [Request Failures](../../../dashboards/sdk/temporal-sdk-go-readme.md) — panel 5
**Full alert definition:** [Alert #1b — alerts-index.md](../alerts-index.md#alert-1b--request-failure-not_found-on-activity-respond-operations)

---

## Table of Contents

- [What this alert detects](#what-this-alert-detects)
- [Why it matters](#why-it-matters)
- [Triage steps](#triage-steps)
- [Related alerts](#related-alerts)

---

## What this alert detects

The SDK worker responded to the server with an activity task completion or failure (`RespondActivityTaskCompleted` or `RespondActivityTaskFailed`) and received `NOT_FOUND` back. This means one of three things: (1) the activity task timed out — the activity ran longer than its `scheduleToClose` or `startToClose` timeout and the server discarded the in-flight task; (2) the workflow execution is no longer running — completed, explicitly terminated, or hit a workflow run timeout before the activity finished; (3) the worker restarted mid-execution — the in-flight task token was lost, the server rescheduled the activity, but the original worker still attempted to respond.

## Why it matters

A NOT_FOUND on an activity respond call means the result the worker just produced was discarded. The server has already rescheduled the activity for retry (if within retry policy) or failed the workflow. Repeated NOT_FOUND on activity respond operations is a strong signal that activities are consistently running past their configured timeouts, which will delay workflow end-to-end execution and consume unnecessary worker resources on work that will be thrown away.

## Triage steps

**1. Check activity execution latency.**
Look at `temporal_activity_execution_latency` on the SDK dashboard for the affected `activity_type` and task queue. If p99 is elevated and approaching or exceeding the `startToClose` timeout, activities are consistently finishing too late — that is the direct cause.

**2. Check worker pod resources.**
Check CPU and memory utilization on the activity worker pods — high CPU slows activity execution directly. Also check for cold-start issues on the pod that picked up the task: look at the `identity` field in the `ActivityTaskStarted` history event, or in the pending activity info if this is not the final attempt.

**3. Check whether the workflow execution is still running.**
Look at the execution status in the Temporal UI or via `tctl`. If the execution completed, was terminated, or hit a run timeout, the NOT_FOUND is expected and not a signal of a timeout problem — check whether the execution lifecycle is behaving as expected.

**4. Check for worker restarts.**
If worker pods are restarting frequently during activity execution, in-flight task tokens are lost on restart. The server reschedules the activity but the original worker may still attempt to respond after coming back up. Check pod restart counts in your infrastructure observability stack.

**5. Check for RESOURCE_EXHAUSTED on respond operations.**
Cross-check alert [#2b — RESOURCE_EXHAUSTED on Respond Operations](../alerts-index.md#alert-2b--request-failure-resource_exhausted-on-respond-operations). Sustained throttling on `RespondActivityTaskCompleted` can delay the respond call long enough for the server to time out the task before the response lands.

**6. Check server-side service and persistence latency.**
If activity latency looks normal, check the server dashboard:
- [**Frontend Service Latency** panel (§4 — Service Latencies)](../../../dashboards/server/temporal-server-readme.md) filtered to `RespondActivityTaskCompleted` and `RespondActivityTaskFailed`
- [**Persistence Latencies** panel (§3 — Persistence Requests, Latencies and Errors)](../../../dashboards/server/temporal-server-readme.md#3-persistence-requests-latencies-and-errors) filtered to `UpdateWorkflowExecution`

## Related alerts

- [#1c — NOT_FOUND on RecordActivityTaskHeartbeat](./01c-not-found-heartbeat.md)
- [#2b — RESOURCE_EXHAUSTED on Respond Operations](../alerts-index.md#alert-2b--request-failure-resource_exhausted-on-respond-operations)
- [#13 — RespondActivityTaskCompleted Rate Dropped to Zero](./13-activity-completions-zero.md)
- [#26 — Activity Execution Failed Rate Elevated](./26-activity-execution-failed.md)
