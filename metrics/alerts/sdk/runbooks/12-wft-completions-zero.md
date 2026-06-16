# RespondWorkflowTaskCompleted Rate Dropped to Zero

**Severity:** Critical
**Component:** wft
**Metric:** `temporal_request` — operation `RespondWorkflowTaskCompleted`, rate == 0
**Dashboard panel:** [Request Rate](../../../dashboards/sdk/temporal-sdk-go-readme.md) — panel 4
**Full alert definition:** [Alert #12 — alerts-index.md](../alerts-index.md#alert-12--respondworkflowtaskcompleted-rate-dropped-to-zero)

---

## Table of Contents

- [What this alert detects](#what-this-alert-detects)
- [Why it matters](#why-it-matters)
- [Triage steps](#triage-steps)
- [Related alerts](#related-alerts)

---

## What this alert detects

The rate of `RespondWorkflowTaskCompleted` calls has dropped to zero for this task queue. Workers have fully stopped completing workflow tasks — workflow progress has halted.

## Why it matters

No WFT completions means no workflow executions are making progress. Signals, updates, timers, and activity results are accumulating in history but no worker is processing them. At scale, pending tasks accumulate on the server and put growing pressure on the matching and persistence layers. Depending on your workflow execution timeouts, executions may begin timing out if this persists.

## Triage steps

**1. Check if polling has also stopped.**
Check `temporal_num_pollers` for `worker_type=WorkflowWorker` on the SDK dashboard. If pollers are also at zero, workers are down entirely — cross-check alert [#15 — All Pollers Disconnected](./15-all-pollers-disconnected.md) and check auth failure alert [#4](../alerts-index.md#alert-4--request-failure-authentication--authorization-failure) and UNIMPLEMENTED alert [#5](./05-unimplemented.md). Check worker pod health in your infrastructure observability stack.

**2. Check if workers are failing tasks instead of completing them.**
If polling is active but completions are zero, check `temporal_workflow_task_execution_failed` on the SDK dashboard — workers may be consistently failing tasks. Cross-check alert [#21 — WFT Execution Failed: NonDeterminismError](./21-nde.md) and alert [#22 — WFT Execution Failed: GrpcMessageTooLarge](./22-grpc-message-too-large.md).

**3. Check worker task slots.**
Cross-check alert [#14 — Worker Task Slots Exhausted](./14-worker-slots-exhausted.md) for `worker_type=WorkflowWorker` — if all slots are occupied, the SDK blocks before issuing the next poll and no new tasks are picked up or completed.

**4. Check for RESOURCE_EXHAUSTED on respond operations.**
Cross-check alert [#2b — RESOURCE_EXHAUSTED on Respond Operations](./02b-resource-exhausted-respond-ops.md) — sustained throttling on `RespondWorkflowTaskCompleted` suppresses this counter directly since the SDK only increments it after a successful response.

**5. Check WFT schedule-to-start latency.**
Cross-check alerts [#19](./19-wft-schedule-to-start-critical.md) and [#19b](./19b-wft-schedule-to-start-severe.md) — if tasks are not being dispatched to workers, completions will drop to zero even when workers are healthy.

**6. Check overall cluster health.**
Check the server dashboard: [**Service Errors by Namespace** (§5)](../../../dashboards/server/temporal-server-readme.md), [**Persistence Availability** (§3)](../../../dashboards/server/temporal-server-readme.md#3-persistence-requests-latencies-and-errors), and [**Resource Exhausted with Cause** (§6)](../../../dashboards/server/temporal-server-readme.md).

## Related alerts

- [#2b — RESOURCE_EXHAUSTED on Respond Operations](./02b-resource-exhausted-respond-ops.md)
- [#14 — Worker Task Slots Exhausted](./14-worker-slots-exhausted.md)
- [#15 — All Pollers Disconnected](./15-all-pollers-disconnected.md)
- [#19 — WFT Schedule-To-Start Latency Critical](./19-wft-schedule-to-start-critical.md)
- [#21 — WFT Execution Failed: NonDeterminismError](./21-nde.md)
- [#22 — WFT Execution Failed: GrpcMessageTooLarge](./22-grpc-message-too-large.md)
