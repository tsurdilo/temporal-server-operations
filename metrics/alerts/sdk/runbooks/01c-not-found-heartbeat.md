# NOT_FOUND on RecordActivityTaskHeartbeat

**Severity:** Warning
**Component:** activity
**Metric:** `temporal_request_failure` — `status_code=NOT_FOUND`, operation `RecordActivityTaskHeartbeat`
**Dashboard panel:** [Request Failures](../../../dashboards/sdk/temporal-sdk-go-readme.md) — panel 5
**Full alert definition:** [Alert #1c — alerts-index.md](../alerts-index.md#alert-1c--request-failure-not_found-on-recordactivitytaskheartbeat)

---

## Table of Contents

- [What this alert detects](#what-this-alert-detects)
- [Why it matters](#why-it-matters)
- [Triage steps](#triage-steps)
- [Related alerts](#related-alerts)

---

## What this alert detects

The SDK worker attempted to heartbeat a running activity and received `NOT_FOUND` back from the server. This means the server has already cancelled the in-flight activity task — either the `heartbeatTimeout` fired before the worker's next heartbeat call, the `startToClose` timeout expired while the activity was still executing, or the workflow execution is no longer running. Note: normal workflow-side cancellation returns `CancelRequested=true` in the heartbeat response body, not a gRPC error — so NOT_FOUND on this operation is a reliable signal of timeout or forced closure, not intentional cancellation.

## Why it matters

If `heartbeatTimeout` is the cause, the server has already timed out this activity attempt and scheduled it for retry (if within retry policy). The activity will be re-executed from scratch on the next attempt. If the activity is not idempotent, re-execution can cause duplicate side effects. Repeated NOT_FOUND on heartbeat calls indicates that the worker is consistently failing to heartbeat within the configured interval — the activity will keep timing out on every attempt until the root cause is fixed, consuming worker slots and generating unnecessary retry tasks.

## Triage steps

**1. Check the heartbeat interval vs heartbeatTimeout.**
The worker must call heartbeat more frequently than the configured `heartbeatTimeout`. If the activity is slow between heartbeat calls — due to CPU pressure, blocking I/O, or downstream throttling — the effective heartbeat interval increases. Compare the activity's heartbeat call frequency in worker logs against its `heartbeatTimeout` setting.

**2. Check worker CPU utilization.**
A CPU-starved worker slows down between heartbeat calls even when the activity is making progress. Check per-pod CPU utilization — if consistently high, consider reducing `maxConcurrentActivityExecutionSize` to lower per-pod concurrency, or scale out horizontally.

**3. Check for RESOURCE_EXHAUSTED on RecordActivityTaskHeartbeat.**
Cross-check alert [#2d — RESOURCE_EXHAUSTED on Other Operations](../alerts-index.md#alert-2d--request-failure-resource_exhausted-on-other-operations). If the server is throttling heartbeat calls, the effective heartbeat interval increases and can push past `heartbeatTimeout` even when the worker is calling on time. Check the [**Resource Exhausted with Cause** panel (§6 — Throttling and Limits)](../../../dashboards/server/temporal-server-readme.md) on the server dashboard.

**4. Check startToClose timeout.**
If the activity has been running longer than its `startToClose` timeout, the server times it out at the history level while the activity is still executing — the next heartbeat then returns NOT_FOUND. Check `temporal_activity_execution_latency` for the affected `activity_type` against the configured `startToClose` timeout.

**5. Check heartbeat payload size.**
The last heartbeat details payload is stored in shard mutable state and held in history service memory for the life of the activity attempt. Large heartbeat payloads on high-throughput activity workers can contribute to history host memory pressure. Keep heartbeat payloads small — store only the minimum progress state needed to resume on retry.

## Related alerts

- [#1b — NOT_FOUND on Activity Respond Operations](./01b-not-found-activity-respond.md)
- [#2d — RESOURCE_EXHAUSTED on Other Operations](../alerts-index.md#alert-2d--request-failure-resource_exhausted-on-other-operations)
- [#14 — Worker Task Slots Exhausted](./14-worker-slots-exhausted.md)
- [#26 — Activity Execution Failed Rate Elevated](./26-activity-execution-failed.md)
