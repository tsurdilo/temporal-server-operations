# NOT_FOUND on RecordActivityTaskHeartbeat

**Severity:** Warning
**Component:** activity
**Metric:** `temporal_request_failure` — `status_code=NOT_FOUND`, operation `RecordActivityTaskHeartbeat`
**Dashboard panel:** [Request Failures](../../../dashboards/sdk/temporal-sdk-go-readme.md) — panel 5
**Full alert definition:** [Alert #1c — alerts-index.md](../alerts-index.md#alert-1c--request-failure-not_found-on-recordactivitytaskheartbeat)

---

## Table of Contents

- [What this alert detects](#what-this-alert-detects)
- [Why it matters](#why-it-matters)
- [Triage steps](#triage-steps)
- [Related alerts](#related-alerts)

---

## What this alert detects

The server answered heartbeat calls with `NOT_FOUND` for at least 5 minutes. The activity attempt that sent each call is no longer running: it timed out on its heartbeat timeout or its StartToClose timeout, or its workflow execution is no longer running. The server does not tell the SDK worker when an activity attempt times out. This `NOT_FOUND` is how the SDK worker finds out, and the SDK then cancels the activity.

## Why it matters

Until this answer, the timed-out activity attempt kept running on the SDK worker, at the same time as the next attempt, so work can be done twice ([section 1.2 of the playbook](../../../../playbooks/activity-heartbeat-timeouts.md#12-what-the-sdks-do-when-a-heartbeat-call-fails)). This alert misses heartbeat timeouts after which the timed-out attempt makes no more heartbeat calls, for example when its SDK worker crashed. Server alert 55 counts every heartbeat timeout.

## Triage steps

**Follow the [Activity Heartbeat Timeouts playbook](../../../../playbooks/activity-heartbeat-timeouts.md).** In short:

1. **Confirm the heartbeat timeouts** on [**Activity Heartbeat Timeout (Every Attempt)**](../../../dashboards/server/temporal-server-readme.md#14-sdk-workers-info). When it shows none, the `NOT_FOUND` answers come from StartToClose timeouts or from workflow executions that are no longer running, which the playbook does not cover.
2. **Find where the heartbeats were delayed**, with [section 5 of the playbook](../../../../playbooks/activity-heartbeat-timeouts.md#5-how-to-find-where-the-heartbeats-were-delayed).
3. **Follow the recommendation for what section 5 found**, in [section 6](../../../../playbooks/activity-heartbeat-timeouts.md#6-which-recommendations-apply-to-each-cause-of-late-heartbeats).

## Related alerts

- [Server alert 55 — Activity Heartbeat Timeouts](../../server/runbooks/55-activity-heartbeat-timeouts.md)
- [#1b — NOT_FOUND on Activity Respond Operations](./01b-not-found-activity-respond.md)
- [#2d — RESOURCE_EXHAUSTED on Other Operations](../alerts-index.md#alert-2d--request-failure-resource_exhausted-on-other-operations)
- [#14 — Worker Task Slots Exhausted](./14-worker-slots-exhausted.md)
- [#26 — Activity Execution Failed Rate Elevated](./26-activity-execution-failed.md)
