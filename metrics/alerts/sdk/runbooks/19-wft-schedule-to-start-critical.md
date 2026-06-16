# WFT Schedule-To-Start Latency Critical

**Severity:** Critical
**Component:** wft
**Metric:** `temporal_workflow_task_schedule_to_start_latency` — p99 > 5s
**Dashboard panel:** [WFT Schedule To Start](../../../dashboards/sdk/temporal-sdk-go-readme.md) — panel 22
**Full alert definition:** [Alert #19 — alerts-index.md](../alerts-index.md#alert-19--wft-schedule-to-start-latency-critical)

---

## Table of Contents

- [What this alert detects](#what-this-alert-detects)
- [Why it matters](#why-it-matters)
- [Triage steps](#triage-steps)
- [Related alerts](#related-alerts)

---

## What this alert detects

p99 workflow task schedule-to-start latency has been above 5 seconds for 5 minutes on this namespace and task queue. Workflow tasks are sitting in the queue for more than 5 seconds before a worker picks them up.

## Why it matters

Schedule-to-start latency directly adds to workflow execution end-to-end latency — every second tasks spend waiting in the queue is a second added to the time your workflows take to complete. At 5 seconds p99, workflow executions are experiencing significant delays. If the root cause is not addressed this can escalate to alert [#19b — WFT Schedule-To-Start Latency Severely Elevated](./19b-wft-schedule-to-start-severe.md), where tasks wait over 30 minutes and a large task backlog begins building on the server.

## Triage steps

**1. Check worker pod health.**
Check that worker pods are running and healthy in your infrastructure observability stack. If pods are down or restarting, bring them back up first.

**2. Check poller counts.**
Check `temporal_num_pollers` for `worker_type=WorkflowWorker` on the SDK dashboard — if pollers are dropping, cross-check alert [#15 — All Pollers Disconnected](./15-all-pollers-disconnected.md). Fewer pollers means fewer workers competing for tasks, which directly increases schedule-to-start latency.

**3. Check worker task slots.**
Cross-check alert [#14 — Worker Task Slots Exhausted](./14-worker-slots-exhausted.md) for `worker_type=WorkflowWorker` — if all slots are occupied, the SDK blocks before issuing the next poll, reducing effective poll rate and increasing schedule-to-start latency.

**4. Check for RESOURCE_EXHAUSTED on poll operations.**
Cross-check alert [#2c](../alerts-index.md#alert-2c--request-failure-resource_exhausted-on-long-poll-operations) filtered to `PollWorkflowTaskQueue` — if the server is throttling poll operations, workers back off and poll less frequently.

**5. Check the task backlog on the server.**
Check the [**Approximate Task Backlog** panel](../../../dashboards/server/temporal-server-readme.md) in the Matching service section of the server dashboard — a growing backlog confirms tasks are accumulating faster than workers can pick them up.

**6. Check the server-side concurrent poller count.**
Check the [**Total Concurrent Pollers** panel (§15 — Pollers)](../../../dashboards/server/temporal-server-readme.md) on the server dashboard — if the concurrent poller count for workflow tasks has dropped, your worker pool may have shrunk and horizontal scaling is needed.

## Related alerts

- [#14 — Worker Task Slots Exhausted](./14-worker-slots-exhausted.md)
- [#15 — All Pollers Disconnected](./15-all-pollers-disconnected.md)
- [#19b — WFT Schedule-To-Start Latency Severely Elevated](./19b-wft-schedule-to-start-severe.md)
- [#32 — WFT Execution Latency Critical](./32-wft-execution-latency-critical.md)
