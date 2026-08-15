# Worker Task Slots Exhausted

**Severity:** Critical
**Component:** worker
**Metric:** `temporal_worker_task_slots_available` (gauge) — value == 0
**Dashboard panel:** [Worker Task Slots Available](../../../dashboards/sdk/temporal-sdk-go-readme.md) — panel 34
**Full alert definition:** [Alert #14 — alerts-index.md](../alerts-index.md#alert-14--worker-task-slots-exhausted)

---

## Table of Contents

- [What this alert detects](#what-this-alert-detects)
- [Why it matters](#why-it-matters)
- [Triage steps](#triage-steps)
- [WorkflowWorker](#workflowworker)
- [ActivityWorker](#activityworker)
- [LocalActivityWorker](#localactivityworker)
- [Related alerts](#related-alerts)

---

## What this alert detects

All task execution slots for this `worker_type` and task queue are occupied — no new tasks can be picked up. The SDK blocks before issuing the next poll until a slot is released. Impact and remediation differ significantly by `worker_type` — check the `worker_type` label on the firing alert and follow the relevant section below.

## Why it matters

Slots exhausted means existing tasks are taking too long to complete and not releasing their slots. New tasks cannot be picked up until slots free up, causing schedule-to-start latency to rise. If slots remain exhausted, pollers will eventually drop to zero — cross-check alert [#15 — All Pollers Disconnected](./15-all-pollers-disconnected.md).

---

## WorkflowWorker

All workflow task execution slots are occupied. Existing tasks are taking too long to complete and not releasing their slots back to the pool.

**Triage steps:**

**1. Check WFT execution latency.**
Check `temporal_workflow_task_execution_latency` on the SDK dashboard — sustained high values confirm something is holding slots. Cross-check alert [#32 — WFT Execution Latency Critical](./32-wft-execution-latency-critical.md).

**2. Check worker pod CPU.**
High CPU slows WFT execution directly and keeps slots occupied longer. Check per-pod CPU utilization in your infrastructure observability stack.

**3. Check for blocking calls in workflow code.**
Slots are held until the WFT completes. If workflow code is performing blocking I/O, heavy computation, or calling non-Temporal APIs synchronously, the WFT will hold its slot far longer than expected. For Python SDK workers, verify that no `async def` workflow code is blocking the event loop.

**4. Check for RESOURCE_EXHAUSTED on respond operations.**
Cross-check alert [#2b — RESOURCE_EXHAUSTED on Respond Operations](./02b-resource-exhausted-respond-ops.md) — if the server is throttling `RespondWorkflowTaskCompleted`, slots are not released until the respond call succeeds.

**Immediate action:** Scale up workflow worker pods or increase `maxConcurrentWorkflowTaskExecutionSize`.

---

## ActivityWorker

All activity execution slots are occupied. Existing activities are taking too long to complete and not releasing their slots.

**Triage steps:**

**1. Check activity execution latency.**
Check `temporal_activity_execution_latency` for the affected task queue and `activity_type` — sustained high values confirm activities are holding slots far longer than expected.

**2. Check worker pod CPU.**
High CPU slows activity execution directly. Check per-pod CPU utilization.

**3. Check for downstream throttling.**
If activity slots are exhausted because activities are blocked waiting on a downstream service, increasing slot counts will only increase pressure on that downstream service and make the problem worse. Investigate what the activities are waiting on before scaling up concurrency.

**4. Check for RESOURCE_EXHAUSTED on respond operations.**
Cross-check alert [#2b — RESOURCE_EXHAUSTED on Respond Operations](./02b-resource-exhausted-respond-ops.md) — if the server is throttling `RespondActivityTaskCompleted`, slots are not released until the respond call succeeds.

**Immediate action:** Scale up activity worker pods or increase `maxConcurrentActivityExecutionSize`.

---

## LocalActivityWorker

All local activity execution slots are occupied. The worker cannot schedule or pick up new local activities.

**Triage steps:**

**1. Understand the impact.**
Local activities run within the WFT execution loop — blocked slots hold up the entire WFT. The SDK responds by sending repeated WFT heartbeats to keep the WFT alive. Cross-check alert [#25 — WFT Heartbeat Rate Elevated](../alerts-index.md#alert-25--wft-heartbeat-rate-elevated). If heartbeating continues past `history.workflowTaskHeartbeatTimeout` (default 30 minutes), the server will time out the heartbeat WFT and reschedule it — local activities will be re-executed from scratch with potential duplicate side effects if not idempotent.

**2. Check what local activities are waiting on.**
The root cause is almost always local activity code that is blocking and not returning. Check worker logs for the affected `activity_type`. If they are calling downstream services, check whether those services are throttling or slow — increasing slot counts in that scenario would only increase downstream pressure.

**3. Check worker pod CPU.**
High CPU slows local activity execution directly. Check per-pod CPU utilization.

**4. Cross-check local activity latency alerts.**
Cross-check alert [#29 — Local Activity Execution Latency Exceeds WFT Heartbeat Timeout](./29-la-execution-latency-timeout.md) and alert [#30 — Local Activity Total Execution Latency Exceeds WFT Heartbeat Timeout](./30-la-total-latency-timeout.md) — if either is firing alongside this alert, the WFT heartbeat timeout is imminent or already exceeded.

## Related alerts

- [#2b — RESOURCE_EXHAUSTED on Respond Operations](./02b-resource-exhausted-respond-ops.md)
- [#15 — All Pollers Disconnected](./15-all-pollers-disconnected.md)
- [#29 — Local Activity Execution Latency Exceeds WFT Heartbeat Timeout](./29-la-execution-latency-timeout.md)
- [#30 — Local Activity Total Execution Latency Exceeds WFT Heartbeat Timeout](./30-la-total-latency-timeout.md)
- [#32 — WFT Execution Latency Critical](./32-wft-execution-latency-critical.md)
