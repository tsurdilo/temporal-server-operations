# RespondActivityTaskCompleted Rate Dropped to Zero

**Severity:** Critical
**Component:** activity
**Metric:** `temporal_request` — operation `RespondActivityTaskCompleted`, rate == 0
**Dashboard panel:** [Request Rate](../../../dashboards/sdk/temporal-sdk-go-readme.md) — panel 4
**Full alert definition:** [Alert #13 — alerts-index.md](../alerts-index.md#alert-13--respondactivitytaskcompleted-rate-dropped-to-zero)

---

## Table of Contents

- [What this alert detects](#what-this-alert-detects)
- [Why it matters](#why-it-matters)
- [Triage steps](#triage-steps)
- [Related alerts](#related-alerts)

---

## What this alert detects

The rate of `RespondActivityTaskCompleted` calls has dropped to zero for this task queue. Workers have fully stopped completing activity tasks — any workflow execution waiting on an activity result is stalled.

## Why it matters

No activity completions means no workflows waiting on activity results can make progress. Depending on your activity `scheduleToClose` timeouts, activities will begin timing out if this persists — the server will retry them (if within retry policy) but with no workers completing them, retries will also accumulate. At scale, pending activity tasks build up on the server and put growing pressure on the matching and persistence layers.

## Triage steps

**1. Check if polling has also stopped.**
Check `temporal_num_pollers` for `worker_type=ActivityWorker` on the SDK dashboard. If pollers are also at zero, workers are down entirely — cross-check alert [#15 — All Pollers Disconnected](./15-all-pollers-disconnected.md) and check auth failure alert [#4](../alerts-index.md#alert-4--request-failure-authentication--authorization-failure) and UNIMPLEMENTED alert [#5](./05-unimplemented.md). Check worker pod health in your infrastructure observability stack.

**2. Check for RESOURCE_EXHAUSTED on poll operations.**
Cross-check alert [#2c](../alerts-index.md#alert-2c--request-failure-resource_exhausted-on-long-poll-operations) — if the server is throttling `PollActivityTaskQueue`, workers back off and pick up tasks less frequently, which can suppress completions even when workers are alive.

**3. Check for RESOURCE_EXHAUSTED on respond operations.**
Cross-check alert [#2b — RESOURCE_EXHAUSTED on Respond Operations](./02b-resource-exhausted-respond-ops.md) — sustained throttling on `RespondActivityTaskCompleted` suppresses this counter directly since the SDK only increments it after a successful response.

**4. Check worker task slots.**
Cross-check alert [#14 — Worker Task Slots Exhausted](./14-worker-slots-exhausted.md) for `worker_type=ActivityWorker` — if all slots are occupied, the SDK blocks before issuing the next poll and no new tasks are picked up or completed.

**5. Check for workflow task-level problems.**
Activity completions drop to zero when no new activities are being scheduled — which happens when workflow tasks are not completing. Cross-check alert [#12 — RespondWorkflowTaskCompleted Rate Dropped to Zero](./12-wft-completions-zero.md), alert [#21 — WFT Execution Failed: NonDeterminismError](./21-nde.md), alert [#22 — WFT Execution Failed: GrpcMessageTooLarge](./22-grpc-message-too-large.md), and WFT schedule-to-start alerts [#19](./19-wft-schedule-to-start-critical.md) and [#19b](./19b-wft-schedule-to-start-severe.md).

**6. Check overall cluster health.**
Check the server dashboard: [**Service Errors by Namespace** (§5)](../../../dashboards/server/temporal-server-readme.md), [**Persistence Availability** (§3)](../../../dashboards/server/temporal-server-readme.md#3-persistence-requests-latencies-and-errors), and [**Resource Exhausted with Cause** (§6)](../../../dashboards/server/temporal-server-readme.md).

## Related alerts

- [#2b — RESOURCE_EXHAUSTED on Respond Operations](./02b-resource-exhausted-respond-ops.md)
- [#12 — RespondWorkflowTaskCompleted Rate Dropped to Zero](./12-wft-completions-zero.md)
- [#14 — Worker Task Slots Exhausted](./14-worker-slots-exhausted.md)
- [#15 — All Pollers Disconnected](./15-all-pollers-disconnected.md)
- [#19 — WFT Schedule-To-Start Latency Critical](./19-wft-schedule-to-start-critical.md)
- [#21 — WFT Execution Failed: NonDeterminismError](./21-nde.md)
- [#26 — Activity Execution Failed Rate Elevated](./26-activity-execution-failed.md)
