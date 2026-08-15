# WFT Schedule-To-Start Latency Severely Elevated

**Severity:** Critical
**Component:** wft
**Metric:** `temporal_workflow_task_schedule_to_start_latency` — p99 > 1800s (30m)
**Dashboard panel:** [WFT Schedule To Start](../../../dashboards/sdk/temporal-sdk-go-readme.md) — panel 22
**Full alert definition:** [Alert #19b — alerts-index.md](../alerts-index.md#alert-19b--wft-schedule-to-start-latency-severely-elevated)

---

## Table of Contents

- [What this alert detects](#what-this-alert-detects)
- [Why it matters](#why-it-matters)
- [Triage steps](#triage-steps)
- [Related alerts](#related-alerts)

---

## What this alert detects

p99 workflow task schedule-to-start latency has been above 30 minutes for 5 minutes on this namespace and task queue. Workflow tasks are sitting in the queue for over 30 minutes before a worker picks them up.

## Why it matters

This is severe degradation. At this level of latency, workflow executions are effectively stalled — any execution waiting on a workflow task is not making progress. At scale, a large task backlog accumulates on the server, putting significant pressure on the matching service and the underlying database. If the backlog grows large enough it can affect the entire cluster, not just the impacted namespace and task queue. Immediate action is required.

## Triage steps

**1. Check worker pod health.**
Check that worker pods are running and healthy in your infrastructure observability stack. If pods are down or restarting, bring them back up immediately — this is the most common cause of severe schedule-to-start latency.

**2. Check poller counts.**
Check `temporal_num_pollers` for `worker_type=WorkflowWorker` on the SDK dashboard. If pollers have dropped to zero, cross-check alert [#15 — All Pollers Disconnected](./15-all-pollers-disconnected.md) and follow that runbook first.

**3. Check worker task slots.**
Cross-check alert [#14 — Worker Task Slots Exhausted](./14-worker-slots-exhausted.md) for `worker_type=WorkflowWorker` — if all slots are occupied, no new tasks can be picked up regardless of how many pollers are active.

**4. Check the task backlog on the server.**
Check the [**Approximate Task Backlog** panel](../../../dashboards/server/temporal-server-readme.md) in the Matching service section of the server dashboard — at this latency level the backlog is likely already very large. A large backlog means even after workers recover, it will take time to drain before schedule-to-start latency returns to normal.

**5. Check the server-side concurrent poller count.**
Check the [**Total Concurrent Pollers** panel (§15 — Pollers)](../../../dashboards/server/temporal-server-readme.md) on the server dashboard — if the concurrent poller count for workflow tasks has dropped significantly, scale out worker pods horizontally to recover polling capacity faster.

**6. Check for RESOURCE_EXHAUSTED on poll operations.**
Cross-check alert [#2c](../alerts-index.md#alert-2c--request-failure-resource_exhausted-on-long-poll-operations) filtered to `PollWorkflowTaskQueue` — if the server is throttling poll operations at this scale, increasing the namespace concurrent poller limit via `frontend.namespaceCount` or `frontend.globalNamespaceCount` may be needed alongside scaling workers.

## Related alerts

- [#14 — Worker Task Slots Exhausted](./14-worker-slots-exhausted.md)
- [#15 — All Pollers Disconnected](./15-all-pollers-disconnected.md)
- [#19 — WFT Schedule-To-Start Latency Critical](./19-wft-schedule-to-start-critical.md)
- [#32 — WFT Execution Latency Critical](./32-wft-execution-latency-critical.md)
