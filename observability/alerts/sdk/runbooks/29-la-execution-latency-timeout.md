# Local Activity Execution Latency Exceeds WFT Heartbeat Timeout

**Severity:** Critical
**Component:** local-activity
**Metric:** `temporal_local_activity_execution_latency` — p99 > 1800s (30m)
**Dashboard panel:** [Local Activity Execution Latency](../../../dashboards/sdk/temporal-sdk-go-readme.md) — panel 74
**Full alert definition:** [Alert #29 — alerts-index.md](../alerts-index.md#alert-29--local-activity-execution-latency-exceeds-wft-heartbeat-timeout)

---

## Table of Contents

- [What this alert detects](#what-this-alert-detects)
- [Why it matters](#why-it-matters)
- [Triage steps](#triage-steps)
- [Related alerts](#related-alerts)

---

## What this alert detects

A single local activity attempt is running longer than 30 minutes — the default server WFT heartbeat timeout (`history.workflowTaskHeartbeatTimeout`). The SDK is sending WFT heartbeats to keep the workflow task alive while the local activity runs, but if a single attempt exceeds this timeout the server will time out the heartbeat WFT.

## Why it matters

When the server times out the heartbeat WFT it reschedules it on the normal task queue. The local activity will be re-executed from scratch on the retried task — local activities cannot heartbeat and their progress is not checkpointed between WFT heartbeats. Any pending signals, updates, or other workflow events are delayed until the retried WFT completes. Overall workflow execution end-to-end latency increases significantly. If the local activity is not idempotent, re-execution can cause duplicate side effects with real business impact.

Additionally, the local activity continues running on the SDK worker while heartbeating — occupying a local activity executor slot indefinitely. If multiple local activities are in this state simultaneously, all available local activity slots can become occupied, blocking any new local activities from starting. Cross-check alert [#14 — Worker Task Slots Exhausted](./14-worker-slots-exhausted.md) for `worker_type=LocalActivityWorker`.

Local activities are designed for short, fast operations. A single attempt running for 30 minutes is a design issue.

## Triage steps

**1. Identify the affected local activity type.**
The metric carries an `activity_type` label. Check worker logs for the affected `activity_type` and associated workflow IDs — logs will show what the local activity is doing and how long individual attempts are running.

**2. Investigate what the local activity is blocked on.**
Local activities running this long are almost always blocked on a downstream call — a slow external service, a database query, or a network timeout with a very long timeout setting. Check worker logs for the affected `activity_type` and check the health of any downstream services it calls. Fix the downstream issue or reduce the timeout on the downstream call so the local activity fails fast rather than hanging.

**3. Check whether WFT heartbeat timeouts have already occurred.**
By the time this alert fires, the server may have already timed out heartbeat WFTs and rescheduled them — check worker logs for WFT timeout errors and check the history of affected executions for `WorkflowTaskTimedOut` events. If timeouts have occurred, local activities have already been re-executed from scratch — verify whether they are idempotent and whether any duplicate side effects need to be addressed.

**4. Check whether LA slots are exhausted.**
Cross-check alert [#14 — Worker Task Slots Exhausted](./14-worker-slots-exhausted.md) for `worker_type=LocalActivityWorker` — if multiple local activities are running long simultaneously, all available slots can become occupied and no new local activities can start.

**5. Fix the design.**
If the local activity must perform long-running work, convert it to a regular activity with heartbeating — this is the correct primitive for work that runs longer than a few seconds. If it must remain a local activity, always set a `scheduleToCloseTimeout` less than 30 minutes so it fails with a timeout error before the server times out the WFT heartbeat, allowing the workflow to handle the failure gracefully rather than having the entire WFT re-executed from scratch.

## Related alerts

- [#14 — Worker Task Slots Exhausted](./14-worker-slots-exhausted.md)
- [#25 — WFT Heartbeat Rate Elevated](../alerts-index.md#alert-25--wft-heartbeat-rate-elevated)
- [#30 — Local Activity Total Execution Latency Exceeds WFT Heartbeat Timeout](./30-la-total-latency-timeout.md)
