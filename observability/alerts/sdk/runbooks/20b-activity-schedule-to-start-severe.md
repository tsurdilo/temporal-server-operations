# Activity Schedule-To-Start Latency Severely Elevated

**Severity:** Critical
**Component:** activity
**Metric:** `temporal_activity_schedule_to_start_latency` — p99 > 1800s (30m)
**Dashboard panel:** [Activity Schedule To Start](../../../dashboards/sdk/temporal-sdk-go-readme.md) — panel 23
**Full alert definition:** [Alert #20b — alerts-index.md](../alerts-index.md#alert-20b--activity-schedule-to-start-latency-severely-elevated)

---

## Table of Contents

- [What this alert detects](#what-this-alert-detects)
- [Why it matters](#why-it-matters)
- [Triage steps](#triage-steps)
- [Related alerts](#related-alerts)

---

## What this alert detects

p99 activity task schedule-to-start latency has been above 30 minutes for 5 minutes on this namespace and task queue. Activity tasks are sitting in the queue for over 30 minutes before a worker picks them up.

## Why it matters

This is severe degradation. Any workflow execution waiting on an activity result is stalled for the duration of this latency. At scale, a large activity task backlog accumulates on the server, putting significant pressure on the matching service and the underlying database. A growing backlog can affect the entire cluster if left unaddressed. Immediate action is required.

## Triage steps

**1. Check worker pod health.**
Check that activity worker pods are running and healthy in your infrastructure observability stack. If pods are down or restarting, bring them back up immediately — this is the most common cause of severe schedule-to-start latency.

**2. Check poller counts.**
Check `temporal_num_pollers` for `worker_type=ActivityWorker` on the SDK dashboard. If pollers have dropped to zero, cross-check alert [#15 — All Pollers Disconnected](./15-all-pollers-disconnected.md) and follow that runbook first.

**3. Check worker task slots.**
Cross-check alert [#14 — Worker Task Slots Exhausted](./14-worker-slots-exhausted.md) for `worker_type=ActivityWorker` — if all slots are occupied, no new tasks can be picked up regardless of how many pollers are active.

**4. Check the task backlog on the server.**
Check the [**Approximate Task Backlog** panel](../../../dashboards/server/temporal-server-readme.md) in the Matching service section of the server dashboard — at this latency level the backlog is likely already very large. A large backlog means even after workers recover, it will take time to drain before schedule-to-start latency returns to normal.

**5. Check the server-side concurrent poller count.**
Check the [**Total Concurrent Pollers** panel (§15 — Pollers)](../../../dashboards/server/temporal-server-readme.md) on the server dashboard — if the concurrent poller count for activity tasks has dropped significantly, scale out activity worker pods horizontally to recover polling capacity faster.

**6. Check for RESOURCE_EXHAUSTED on poll operations.**
Cross-check alert [#2c](../alerts-index.md#alert-2c--request-failure-resource_exhausted-on-long-poll-operations) filtered to `PollActivityTaskQueue` — if the server is throttling poll operations at this scale, increasing the namespace concurrent poller limit via `frontend.namespaceCount` or `frontend.globalNamespaceCount` may be needed alongside scaling workers.

**7. Check activity failure rates.**
Cross-check alert [#26 — Activity Execution Failed Rate Elevated](./26-activity-execution-failed.md) — high churn of unexpected activity failures generates additional retry tasks, which can contribute to backlog growth and keep schedule-to-start latency elevated even after worker capacity is restored.

## Related alerts

- [#14 — Worker Task Slots Exhausted](./14-worker-slots-exhausted.md)
- [#15 — All Pollers Disconnected](./15-all-pollers-disconnected.md)
- [#19b — WFT Schedule-To-Start Latency Severely Elevated](./19b-wft-schedule-to-start-severe.md)
- [#26 — Activity Execution Failed Rate Elevated](./26-activity-execution-failed.md)
