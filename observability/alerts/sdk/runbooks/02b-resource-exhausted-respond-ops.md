# RESOURCE_EXHAUSTED on Respond Operations

**Severity:** Critical
**Component:** request
**Metric:** `temporal_request_failure` — `status_code=RESOURCE_EXHAUSTED`, operations `RespondWorkflowTaskCompleted` / `RespondWorkflowTaskFailed` / `RespondActivityTaskCompleted` / `RespondActivityTaskFailed`
**Dashboard panel:** [Request Failures](../../../dashboards/sdk/temporal-sdk-go-readme.md) — panel 5
**Full alert definition:** [Alert #2b — alerts-index.md](../alerts-index.md#alert-2b--request-failure-resource_exhausted-on-respond-operations)

---

## Table of Contents

- [What this alert detects](#what-this-alert-detects)
- [Why it matters](#why-it-matters)
- [Triage steps](#triage-steps)
- [Related alerts](#related-alerts)

---

## What this alert detects

The server is returning `RESOURCE_EXHAUSTED` on workflow and activity task respond operations. The SDK retries these automatically, but sustained throttling means the respond call is delayed — the worker holds the task slot until the respond call succeeds, and the server-side task remains in-flight until the response lands.

## Why it matters

This is a leading indicator for alerts #1a and #1b. If the server keeps returning RESOURCE_EXHAUSTED on respond operations long enough, the workflow or activity task will time out — the server writes a `WorkflowTaskTimedOut` or `ActivityTaskTimedOut` event to history and reschedules the task. For workflow tasks this forces a sticky cache eviction and cold replay on retry. For local activities, a WFT timeout causes re-execution from scratch with potential duplicate side effects if not idempotent. Meanwhile, the worker continues to hold the slot for the in-flight task, reducing available concurrency for new tasks.

## Triage steps

**1. Check the server Resource Exhausted panel.**
Check the [**Resource Exhausted with Cause** panel (§6 — Throttling and Limits)](../../../dashboards/server/temporal-server-readme.md) on the server dashboard — identify the throttle cause (`RpsLimit`, `ConcurrentLimit`, `SystemOverloaded`, `CircuitBreakerOpen`).

**2. Check persistence latency.**
Check the [**Persistence Latencies** panel (§3 — Persistence Requests, Latencies and Errors)](../../../dashboards/server/temporal-server-readme.md#3-persistence-requests-latencies-and-errors) filtered to `UpdateWorkflowExecution` — high persistence latency on this operation is the most common root cause of throttling on respond operations. The frontend service backs up when the database is slow and starts shedding load.

**3. Check namespace RPS limits.**
If the cause is `RpsLimit`, consider increasing `frontend.namespaceRPS` via dynamic config — but only after confirming persistence latency is healthy.

**4. Check for cascading timeouts.**
Cross-check alert [#1a — NOT_FOUND on WFT Respond Operations](./01a-not-found-wft-respond.md) and alert [#1b — NOT_FOUND on Activity Respond Operations](./01b-not-found-activity-respond.md) — if those are also firing, throttling has already cascaded into task timeouts.

**5. Check for system overload.**
If the cause is `SystemOverloaded` or `CircuitBreakerOpen`, cross-check server alert [#30 — System Overload Throttling](../../server/runbooks/30-system-overload-throttling.md).

## Related alerts

- [#1a — NOT_FOUND on WFT Respond Operations](./01a-not-found-wft-respond.md)
- [#1b — NOT_FOUND on Activity Respond Operations](./01b-not-found-activity-respond.md)
- [#2a — RESOURCE_EXHAUSTED on Critical User-Facing Operations](./02a-resource-exhausted-critical-ops.md)
- [#6 — Request Failure: INTERNAL](./06-internal.md)
- [#32 — WFT Execution Latency Critical](./32-wft-execution-latency-critical.md)
