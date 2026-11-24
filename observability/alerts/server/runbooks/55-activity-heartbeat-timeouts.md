## Activity Heartbeat Timeouts

**Severity:** Warning
**Component:** history
**Dashboard panels:** [Activity Heartbeat Timeout (Every Attempt)](../../../dashboards/server/temporal-server-readme.md#14-sdk-workers-info) (3101), [Heartbeat Calls Refused by Cause](../../../dashboards/server/temporal-server-readme.md#6-throttling-and-limits) (3105), [Heartbeat Call Latency (Frontend and History)](../../../dashboards/server/temporal-server-readme.md#4-service-latencies)

### What this alert detects

Activity attempts in a namespace have timed out on their heartbeat timeout for at least 10
minutes, with no gap of 5 minutes or more between heartbeat timeouts. The count (`activity_task_timeout` with `timeout_type="Heartbeat"`) includes every
timed-out attempt, also the ones the server retries. The `namespace` label names the namespace.
Needs server v1.29.0 or later; older servers do not emit the metric, and the alert never fires
there.

### Why it matters

**A heartbeat timeout that the server retries does not fail the workflow, so workflows can look
healthy while it goes on.** Each timed-out attempt starts the activity code again, which continues
from the last heartbeat details at best, so work is done twice. The panel that counts only final heartbeat timeouts,
**Activity Heartbeat Timeout (Final Attempt)**, stays at zero while every timeout is retried.

**A single SDK worker crash makes a short burst of heartbeat timeouts**, one per activity the SDK
worker was running, and usually does not keep this alert firing. Heartbeat timeouts that go on for
10 minutes mean heartbeats keep arriving late: the server refuses or slows the heartbeat calls, or
the SDK workers send them late.

### Check this first

- **[Activity Heartbeat Timeout (Every Attempt)](../../../dashboards/server/temporal-server-readme.md#14-sdk-workers-info)**:
  which activity types time out. One activity type points at its code or its SDK workers; many at
  once point at something they share.
- **[Heartbeat Calls Refused by Cause](../../../dashboards/server/temporal-server-readme.md#6-throttling-and-limits)**:
  refusals that rise together with the heartbeat timeouts.
- **[Heartbeat Call Latency (Frontend and History)](../../../dashboards/server/temporal-server-readme.md#4-service-latencies)**:
  heartbeat calls that take close to the time they have.
- **The SDK dashboard** for the SDK workers, Request Latency and Request Failures for
  `RecordActivityTaskHeartbeat`, and the SDK worker hosts' CPU.

### What to do

**Follow the playbook. It is the full procedure for this alert, and this runbook does not repeat
it:**

> **[Activity Heartbeat Timeouts](../../../../playbooks/activity-heartbeat-timeouts.md)**

In short:

1. **Find where the heartbeats were delayed**, with the checks in
   [section 5](../../../../playbooks/activity-heartbeat-timeouts.md#5-how-to-find-where-the-heartbeats-were-delayed).
2. **Follow the recommendation for what the checks found**, in
   [section 6](../../../../playbooks/activity-heartbeat-timeouts.md#6-which-recommendations-apply-to-each-cause-of-late-heartbeats).
3. **Go through section 5 again after each fix.** Heartbeats can be late for more than one reason
   at the same time.

### What not to do

- **Do not lower the heartbeat timeout of running activities** with the Temporal CLI command
  `temporal activity update-options`. The SDK keeps sending heartbeats at the batching window of the
  original heartbeat timeout, so every activity attempt times out.
- **Do not add `TemporalTimeout:Heartbeat` to an activity's non-retryable error types** to stop the
  retries. The activity then fails on its first heartbeat timeout.
- **Do not raise heartbeat timeouts before finding where the heartbeats were delayed.** A longer
  heartbeat timeout also makes the server notice a crashed SDK worker later.

### Gaps in this alert

- It sums over all activity types in the namespace. Read the panel for which activity types time
  out.
- It does not say where the heartbeats were delayed.
- Activities with no heartbeat timeout never time out on one, so they are never counted.
- Servers older than v1.29.0 never fire it. On those servers, read Activity Heartbeat Timeout
  (Final Attempt), which counts only heartbeat timeouts that end the activity.
