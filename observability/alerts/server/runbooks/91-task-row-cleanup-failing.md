## Task Row Cleanup Failing

**Severity:** High
**Component:** history
**Dashboard panels:** [Task Row Cleanup Failures by Category](../../../dashboards/server/temporal-server-readme.md#23-history-task-cleanup) (2412), [Task Row Cleanup Latency by Category](../../../dashboards/server/temporal-server-readme.md#23-history-task-cleanup) (2411)

### What this alert detects

Cleanup deletes against the history task tables are failing. One `DELETE` is issued per shard per
checkpoint, covering everything older than that queue's deletion watermark. **On a healthy cluster
those deletes never fail**, so the baseline is a true zero and any sustained failure is actionable
with no threshold to tune.

### Why it matters

A span too large to clear inside the delete's five-second timeout is cancelled, frees no rows, and
stays exactly that size on the next attempt — so the condition does not recover on its own. Each
cancelled attempt still pays full cost in CPU, I/O and WAL on the database while removing nothing,
which is why this is high rather than hygiene: it consumes capacity your workloads need, and the
table goes on growing meanwhile.

### What to do

**Follow the playbook — it is the full procedure for this alert, and this runbook deliberately does
not repeat it:**

> **[History Task Table Growth — Stuck Queue Cleanup Playbook](../../../../playbooks/history-task-table-growth.md)**

Its Fast track is the short path. In order:

| Step | Section |
|---|---|
| Confirm the reading, and which of the two cleanup failures this is | [7.1 Start with the cleanup panels](../../../../playbooks/history-task-table-growth.md#71-start-with-the-cleanup-panels) |
| Find the affected shards and the namespace behind them | [8.1 Get the affected shards from the history service logs](../../../../playbooks/history-task-table-growth.md#81-get-the-affected-shards-from-the-history-service-logs) |
| Clear whatever is stopping that namespace's tasks finishing | [9. Get task row cleanup running again](../../../../playbooks/history-task-table-growth.md#9-get-task-row-cleanup-running-again) |
| Remove the rows the delete can no longer reach, or reload the shards | [9.4](../../../../playbooks/history-task-table-growth.md#94-deleting-task-rows-yourself-and-why-it-is-the-last-resort) / [11.6](../../../../playbooks/history-task-table-growth.md#116-how-to-reload-the-affected-shards) |

### What not to do

**Do not wait for it to pass, and do not raise task throughput.** Both are covered in
[section 10](../../../../playbooks/history-task-table-growth.md#10-what-not-to-do-when-cleanup-deletes-are-failing)
— neither makes the next delete any smaller, and raising throughput unpins more shards into the
same state.

### What this alert does not tell you

**The quiet half of the same problem does not fire it.** A watermark pinned so that *no* delete is
attempted produces no failures to count, and shows instead as cleanup attempts falling to zero.
That cannot be alerted on cleanly, because a category with no work reports the same thing —
[section 12](../../../../playbooks/history-task-table-growth.md#12-alerting-on-a-stalled-cleanup)
explains why, and 7.1 is how to read it on the dashboard.

### Relevant dynamic config

None for the timeout itself — it is compiled into the server. The settings that decide whether the
namespace behind it can drain are collected in
[section 13](../../../../playbooks/history-task-table-growth.md#13-every-setting-named-in-this-playbook)
of the playbook.
