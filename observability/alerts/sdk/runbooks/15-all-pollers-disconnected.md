# All Pollers Disconnected

**Severity:** Critical
**Component:** poller
**Metric:** `temporal_num_pollers` (gauge) — value == 0
**Dashboard panel:** [Number of Active Pollers](../../../dashboards/sdk/temporal-sdk-go-readme.md) — panel 36
**Full alert definition:** [Alert #15 — alerts-index.md](../alerts-index.md#alert-15--all-pollers-disconnected)

---

## Table of Contents

- [What this alert detects](#what-this-alert-detects)
- [Why it matters](#why-it-matters)
- [Triage steps](#triage-steps)
- [Related alerts](#related-alerts)

---

## What this alert detects

No active pollers for this `worker_type` and task queue — workers have stopped polling entirely. Tasks are accumulating on the server with no one to process them.

## Why it matters

With no pollers, no tasks are being picked up. Workflow and activity tasks accumulate on the server — at scale this grows into a large backlog that puts significant pressure on the matching and persistence layers. Depending on your execution timeouts, workflows may begin timing out while waiting for tasks to be processed.

## Triage steps

**1. Check if worker pods are running.**
Check pod status, restart counts, and logs in your infrastructure observability stack — workers may have crashed, been evicted, or OOM killed. This is the most common cause of pollers dropping to zero. If pods are down, bring them back up and verify they start cleanly.

**2. Check if slots are exhausted.**
Cross-check alert [#14 — Worker Task Slots Exhausted](./14-worker-slots-exhausted.md) for the same `worker_type` and task queue. Both Go and Java SDKs block on slot acquisition before incrementing `temporal_num_pollers` — so if all slots are occupied, the gauge drops to zero as a secondary effect. In that case #14 is the root cause and this alert is a symptom. Fix #14 first.

**3. Check for auth failures.**
Cross-check alert [#4](../alerts-index.md#alert-4--request-failure-authentication--authorization-failure) — expired or revoked credentials are a common cause of pollers disconnecting. On the server side, check the [**Unauthorized Requests** panel (§19 — Authorization)](../../../dashboards/server/temporal-server-readme.md) and the [**Authorization System Failures** panel (§19)](../../../dashboards/server/temporal-server-readme.md) — a non-zero value on Authorization System Failures indicates the auth plugin itself is failing, which is more urgent than a simple credential problem.

**4. Check for INTERNAL errors.**
Cross-check alert [#6 — INTERNAL](./06-internal.md) — sustained INTERNAL errors from the server can cause workers to back off and stop polling. Check the [**Service Panics** panel (§5)](../../../dashboards/server/temporal-server-readme.md) and [**Persistence Availability** panel (§3)](../../../dashboards/server/temporal-server-readme.md#3-persistence-requests-latencies-and-errors) on the server dashboard.

**5. Check the server-side poller count.**
Cross-check the [**Total Concurrent Pollers** panel (§15 — Pollers)](../../../dashboards/server/temporal-server-readme.md) on the server dashboard — if the server-side concurrent poller count for this task queue has also dropped, it confirms workers have fully disconnected from the server's perspective.

## Related alerts

- [#6 — Request Failure: INTERNAL](./06-internal.md)
- [#12 — RespondWorkflowTaskCompleted Rate Dropped to Zero](./12-wft-completions-zero.md)
- [#13 — RespondActivityTaskCompleted Rate Dropped to Zero](./13-activity-completions-zero.md)
- [#14 — Worker Task Slots Exhausted](./14-worker-slots-exhausted.md)
