# NOT_FOUND on WFT Respond Operations

**Severity:** Critical
**Component:** wft
**Metric:** `temporal_request_failure` — `status_code=NOT_FOUND`, operations `RespondWorkflowTaskCompleted` / `RespondWorkflowTaskFailed`
**Dashboard panel:** [Request Failures](../../../dashboards/sdk/temporal-sdk-go-readme.md) — panel 5
**Full alert definition:** [Alert #1a — alerts-index.md](../alerts-index.md#alert-1a--request-failure-not_found-on-wft-respond-operations)

---

## Table of Contents

- [What this alert detects](#what-this-alert-detects)
- [Why it matters](#why-it-matters)
- [Triage steps](#triage-steps)
- [Related alerts](#related-alerts)

---

## What this alert detects

The SDK worker responded to the server with a workflow task completion or failure (`RespondWorkflowTaskCompleted` or `RespondWorkflowTaskFailed`) and received `NOT_FOUND` back. This means one of two things: (1) the server already timed out the workflow task before the worker finished executing it and sent the response, or (2) the workflow execution is no longer running — it was explicitly terminated, hit a run timeout, or completed by another path while this worker was still executing the task.

## Why it matters

A NOT_FOUND on a respond call means the workflow task the worker just executed was discarded by the server. The server has already rescheduled the task. If the cause is a WFT timeout, the server writes a `WorkflowTaskTimedOut` event to history — this adds latency to workflow end-to-end execution. If you are running local activities, a WFT timeout will cause local activities to be re-executed from scratch on the retried task, since local activity results are not checkpointed in history between WFT heartbeats. If local activities are not idempotent, re-execution can cause duplicate side effects with real business impact.

## Triage steps

**1. Check WFT execution latency.**
Look at `temporal_workflow_task_execution_latency` on the SDK dashboard for the affected namespace and task queue. If p99 is elevated and approaching or exceeding the WFT timeout (default 10s), the worker is consistently finishing too late and the server is timing out tasks.

**2. Check WFT replay latency.**
If WFT execution latency is elevated, check `temporal_workflow_task_replay_latency` next. High replay latency means the worker is spending too long re-executing history before reaching the new command — check for large history size and cross-check alert [#35 — WFT Replay Latency High](../alerts-index.md#alert-35--wft-replay-latency-high). Also check alert [#16 — Sticky Cache Forced Evictions High](../alerts-index.md#alert-16--sticky-cache-forced-evictions-high) and alert [#34 — Sticky Cache Disabled](../alerts-index.md#alert-34--sticky-cache-disabled) — high eviction rates force frequent cold replays which inflate execution latency.

**3. Check worker pod resources.**
Check CPU and memory utilization on the worker pods — high CPU slows WFT execution directly. Also check for cold-start issues on the pod that picked up the task: look at the `identity` field in the `WorkflowTaskStarted` history event to identify which worker pod ran the task.

**4. Check for RESOURCE_EXHAUSTED on respond operations.**
Cross-check alert [#2b — RESOURCE_EXHAUSTED on Respond Operations](../alerts-index.md#alert-2b--request-failure-resource_exhausted-on-respond-operations). On some SDK versions, sustained server throttling on `RespondWorkflowTaskCompleted` can cascade into a workflow task timeout — the task times out while the SDK is still retrying the throttled respond call.

**5. Check server-side service and persistence latency.**
If SDK-side metrics look normal and the execution was not terminated or timed out, check the server dashboard:
- [**Frontend Service Latency** panel (§4 — Service Latencies)](../../../dashboards/server/temporal-server-readme.md) filtered to `RespondWorkflowTaskCompleted`
- [**Persistence Latencies** panel (§3 — Persistence Requests, Latencies and Errors)](../../../dashboards/server/temporal-server-readme.md#3-persistence-requests-latencies-and-errors) filtered to `UpdateWorkflowExecution`

**6. Check whether the execution was explicitly terminated.**
If the NOT_FOUND is on `RespondWorkflowTaskFailed` (not Completed), also check whether the execution was terminated intentionally — look at the `WorkflowExecutionTerminated` event in history and cross-check the **Workflow Terminate** panel on the server dashboard.

## Related alerts

- [#2b — RESOURCE_EXHAUSTED on Respond Operations](../alerts-index.md#alert-2b--request-failure-resource_exhausted-on-respond-operations)
- [#16 — Sticky Cache Forced Evictions High](../alerts-index.md#alert-16--sticky-cache-forced-evictions-high)
- [#32 — WFT Execution Latency Critical](../alerts-index.md#alert-32--wft-execution-latency-critical)
- [#34 — Sticky Cache Disabled](../alerts-index.md#alert-34--sticky-cache-disabled)
- [#35 — WFT Replay Latency High](../alerts-index.md#alert-35--wft-replay-latency-high)
