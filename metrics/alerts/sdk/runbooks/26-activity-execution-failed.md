# Activity Execution Failed Rate Elevated

**Severity:** Warning
**Component:** activity
**Metric:** `temporal_activity_execution_failed` — rate > 20/s
**Dashboard panel:** [Activity Execution Failed](../../../dashboards/sdk/temporal-sdk-go-readme.md) — panel 63
**Full alert definition:** [Alert #26 — alerts-index.md](../alerts-index.md#alert-26--activity-execution-failed-rate-elevated)

---

## Table of Contents

- [What this alert detects](#what-this-alert-detects)
- [Why it matters](#why-it-matters)
- [Triage steps](#triage-steps)
- [Related alerts](#related-alerts)

---

## What this alert detects

Activities are explicitly failing — not timing out but returning failures — at a sustained rate above 20 per second on this namespace and task queue. `ApplicationFailure` instances marked with category `BENIGN` are excluded and do not increment this counter, so this alert tracks unexpected failures only if your application uses benign failures correctly.

## Why it matters

A high failure rate drives a burst of retry tasks being scheduled on the server. If workers cannot keep up with the retry volume, activity task backlog grows — cross-check alert [#20b — Activity Schedule-To-Start Latency Severely Elevated](./20b-activity-schedule-to-start-severe.md). At large scale, sustained retry bursts put significant pressure on the matching service and the underlying database.

## Triage steps

**1. Identify which activity type is failing.**
The metric carries an `activity_type` label — use it to narrow down which activity is the source. Check worker logs for the affected `activity_type` to get the specific error messages and stack traces. Worker logs will also include the workflow ID associated with each failure.

**2. Investigate the root cause.**
Check whether this is a transient infrastructure issue (downstream service unavailable, network partition, database timeout) or a persistent code bug. Transient failures will naturally recover — monitor whether the rate drops on its own. Persistent failures require a code fix.

**3. Check downstream service health.**
If activities are calling external services, check the health of those services. A degraded downstream service is a common cause of sustained activity failure bursts. If the downstream is throttling, check whether your activity retry policy has appropriate backoff — without backoff, retry bursts can amplify downstream pressure.

**4. Check activity schedule-to-start latency.**
Cross-check alert [#20b — Activity Schedule-To-Start Latency Severely Elevated](./20b-activity-schedule-to-start-severe.md) — a growing retry backlog will show up there as elevated schedule-to-start latency even after the failure rate drops.

**5. Consider using benign failures for expected failures.**
If your use case intentionally fails activities by design — polling patterns, saga compensations, flow control via exceptions — mark those `ApplicationFailure` instances with category `BENIGN`. All SDKs support this and will suppress this metric for those failures, making this alert exclusively track unexpected failures without needing per-`activity_type` threshold tuning.

## Related alerts

- [#1b — NOT_FOUND on Activity Respond Operations](./01b-not-found-activity-respond.md)
- [#13 — RespondActivityTaskCompleted Rate Dropped to Zero](./13-activity-completions-zero.md)
- [#20b — Activity Schedule-To-Start Latency Severely Elevated](./20b-activity-schedule-to-start-severe.md)
- [#27 — Unregistered Activity Invocation](./27-unregistered-activity.md)
