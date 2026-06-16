# Local Activity Total Execution Latency Exceeds WFT Heartbeat Timeout

**Severity:** Critical
**Component:** local-activity
**Metric:** `temporal_local_activity_total_execution_latency` — p99 > 1800s (30m)
**Dashboard panel:** [Local Activity Total Execution Latency](../../../dashboards/sdk/temporal-sdk-go-readme.md) — panel 76
**Full alert definition:** [Alert #30 — alerts-index.md](../alerts-index.md#alert-30--local-activity-total-execution-latency-exceeds-wft-heartbeat-timeout)

---

## Table of Contents

- [What this alert detects](#what-this-alert-detects)
- [Why it matters](#why-it-matters)
- [Triage steps](#triage-steps)
- [Related alerts](#related-alerts)

---

## What this alert detects

The cumulative time across all retry attempts for a local activity has exceeded 30 minutes — the default server WFT heartbeat timeout (`history.workflowTaskHeartbeatTimeout`). Unlike alert #29 which tracks a single attempt, this alert fires when the full retry chain has accumulated past the timeout, even if individual attempts are short.

## Why it matters

Same impact as alert [#29](./29-la-execution-latency-timeout.md) — when the total execution time exceeds the WFT heartbeat timeout, the server times out the heartbeat WFT and reschedules it on the normal task queue. The local activity will be re-executed from scratch on the retried task. Any pending signals, updates, or other workflow events are delayed until the retried WFT completes. If the local activity is not idempotent, re-execution can cause duplicate side effects with real business impact.

Additionally, the local activity continues occupying a local activity executor slot for the entire duration of the retry chain. If multiple local activities are in this state simultaneously, all available slots can become occupied and no new local activities can start. Cross-check alert [#14 — Worker Task Slots Exhausted](./14-worker-slots-exhausted.md) for `worker_type=LocalActivityWorker`.

## Triage steps

**1. Identify the affected local activity type.**
The metric carries an `activity_type` label. Check worker logs for the affected `activity_type` and associated workflow IDs — logs will show individual attempt durations and failure messages across the retry chain.

**2. Check the failure rate driving retries.**
Cross-check alert [#28 — Local Activity Execution Failed Rate Elevated](../alerts-index.md#alert-28--local-activity-execution-failed-rate-elevated) — a high failure rate with aggressive retry policy is the most common cause of this alert firing without alert #29 also firing. Individual attempts are short but the retry chain accumulates. Fix the underlying failure cause first.

**3. Check whether WFT heartbeat timeouts have already occurred.**
By the time this alert fires, the server may have already timed out heartbeat WFTs and rescheduled them — check worker logs for WFT timeout errors and check the history of affected executions for `WorkflowTaskTimedOut` events. If timeouts have occurred, local activities have already been re-executed from scratch — verify whether they are idempotent and whether any duplicate side effects need to be addressed.

**4. Check whether LA slots are exhausted.**
Cross-check alert [#14 — Worker Task Slots Exhausted](./14-worker-slots-exhausted.md) for `worker_type=LocalActivityWorker` — if multiple local activities are accumulating long retry chains simultaneously, all available slots can become occupied.

**5. Fix the retry policy or design.**
If the work genuinely requires many retries over a long period, convert to a regular activity with heartbeating — this is the correct primitive for work that may take a long time across retries. If it must remain a local activity, reduce the maximum retry attempts or increase backoff between retries so the total chain stays under 30 minutes, or set a `scheduleToCloseTimeout` less than 30 minutes so it fails before the server times out the WFT heartbeat.

## Related alerts

- [#14 — Worker Task Slots Exhausted](./14-worker-slots-exhausted.md)
- [#25 — WFT Heartbeat Rate Elevated](../alerts-index.md#alert-25--wft-heartbeat-rate-elevated)
- [#28 — Local Activity Execution Failed Rate Elevated](../alerts-index.md#alert-28--local-activity-execution-failed-rate-elevated)
- [#29 — Local Activity Execution Latency Exceeds WFT Heartbeat Timeout](./29-la-execution-latency-timeout.md)
