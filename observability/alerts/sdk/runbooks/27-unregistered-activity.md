# Unregistered Activity Invocation

> **Go SDK only.** `temporal_unregistered_activity_invocation` is only emitted by the Go SDK. The Java Micrometer, Java OTel, and Core SDKs do not emit this metric — unregistered activity failures on those SDKs are tracked as general activity execution failures via `temporal_activity_execution_failed` (alert [#26](./26-activity-execution-failed.md)).

**Severity:** Critical
**Component:** activity
**Metric:** `temporal_unregistered_activity_invocation` — any occurrence
**Dashboard panel:** [Unregistered Activity Invocation](../../../dashboards/sdk/temporal-sdk-go-readme.md) — panel 65
**Full alert definition:** [Alert #27 — alerts-index.md](../alerts-index.md#alert-27--unregistered-activity-invocation)

---

## Table of Contents

- [What this alert detects](#what-this-alert-detects)
- [Why it matters](#why-it-matters)
- [Triage steps](#triage-steps)
- [Related alerts](#related-alerts)

---

## What this alert detects

The server dispatched an activity task to this worker for an activity type that the worker does not have registered. The worker cannot execute the task and will fail it immediately.

## Why it matters

Tasks will fail indefinitely on this worker — the server will retry the activity according to its retry policy, but if no worker on this task queue has the activity registered, every retry will also fail. The workflow execution waiting on this activity will be blocked until the retry policy is exhausted, at which point the activity fails permanently and the workflow must handle the error. This always indicates a code or deployment issue.

## Triage steps

**1. Identify the unregistered activity type.**
The metric carries an `activity_type` label — use it to identify exactly which activity type is missing. Check worker logs for additional context.

**2. Check your worker registration.**
Verify that the activity type shown in the alert is registered in the worker that is polling on the affected task queue. Check your worker startup code — the activity must be explicitly registered before the worker starts polling.

**3. Check for a deployment issue.**
If the activity was recently added or renamed, check whether the correct worker binary is deployed. A rolling deploy where old worker versions are still running will cause the old workers to receive tasks for newly registered activity types they do not know about. Wait for the rollout to complete or accelerate it.

**4. Check for a routing issue.**
Verify that the task queue name in the workflow code matches the task queue the worker is polling on. A mismatch — for example, a typo in the task queue name — will route activity tasks to workers that are not configured to handle them.

**5. Check for a worker restart during activity execution.**
If a worker was restarted mid-execution with a different binary that is missing the activity registration, in-flight tasks will be re-dispatched to a worker that cannot handle them. Check recent worker deployment history and pod restart counts.

## Related alerts

- [#13 — RespondActivityTaskCompleted Rate Dropped to Zero](./13-activity-completions-zero.md)
- [#15 — All Pollers Disconnected](./15-all-pollers-disconnected.md)
- [#26 — Activity Execution Failed Rate Elevated](./26-activity-execution-failed.md)
