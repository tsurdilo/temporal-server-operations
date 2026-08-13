## History Task DLQ Stranding

**Severity:** Critical
**Component:** history
**Dashboard panel:** [Dead-Lettered Tasks — Execution-Stranding](https://github.com/tsurdilo/temporal-metrics/blob/main/metrics/dashboards/server/temporal-server-readme.md) — panel ID 2202 (History Task DLQ / Terminal Failures group)

### What this alert detects

`dlq_writes` has a non-zero rate for an execution-stranding `operation` type, sustained for 10 minutes. The stranding set is:

- `TimerActiveTaskActivityRetryTimer` / `TimerStandbyTaskActivityRetryTimer` — the retry timer that re-dispatches a retrying activity to matching
- `TimerActiveTaskActivityTimeout` / `TimerStandbyTaskActivityTimeout` — the activity timeout timer
- `TransferActiveTaskActivity` / `TransferStandbyTaskActivity` — initial activity dispatch
- `TransferActiveTaskWorkflowTask` / `TransferStandbyTaskWorkflowTask` — workflow-task dispatch

A history task is sent to the DLQ after it fails processing with *unexpected* errors `history.TaskDLQUnexpectedErrorAttempts` times (default **70 ≈ 1h**), or immediately on a terminal/corruption error or a `history.TaskDLQErrorPattern` match. A DLQ'd task is removed from the active queue and is **not auto-retried**.

### Why it matters

When the dead-lettered task is an `ActivityRetryTimer` or `ActivityTimeout`, the activity's next attempt is never dispatched — it sits "pending / scheduled" with nothing in the task queue and **never self-recovers**.

The usual trigger is a prolonged database outage or overload: persistence operations start timing out (`context deadline exceeded` / `context canceled`), which count as unexpected errors. After ~1h (70 attempts) a wave of timer/retry tasks crosses the threshold and dead-letters. DB-agnostic — the same on Cassandra and SQL.

**What this alert deliberately does not fire on:** visibility (`VisibilityTask.*`), retention (`DeleteHistoryEvent`), and workflow-task-timeout DLQ writes — these do not strand a running execution (WFT timeouts are covered by alerts 56/76), so they are graphed on the dashboard but not paged here. Rate-limit rejections (`ResourceExhausted`) never reach the DLQ at all — they are excused and retried with no loss.

### Triage steps

1. Open the **Dead-Lettered Tasks — Execution-Stranding** panel (2202) and read the `operation` breakdown to see which task type is dead-lettering. `ActivityRetryTimer` is the classic strander.
2. Check the **Leading Indicator** panel (2205, `task_errors`) and the persistence panels — a sustained climb in unexpected task errors + `context deadline exceeded` on `persistence_error_with_type` confirms a database outage/overload as the root cause.
3. Confirm the root cause in history pod logs: `msg="Fail to process task"` with a rising `unexpected-error-attempts` field (approaching 70), followed by `msg="Task enqueued to DLQ"` when it crosses.
4. Identify the stranded workflows: `tdbg --address <frontend:7233> dlq --dlq-version v2 read --dlq-type 2 --cluster <clusterName> --max-message-count 100 --output-filename dlq-timer.txt` (category 2 = timer). The decoded payloads carry workflow IDs. `read` is read-only and safe.

### Remediation

**1. Fix the root cause.** The DLQ fired because persistence was unavailable or overloaded long enough for tasks to cross the 70-attempt threshold. Restore the database and size persistence so operations stop timing out (`history.persistenceMaxQPS` per host, `history.persistenceGlobalMaxQPS` cluster-wide).

**2. Recover the stranded work.** DLQ'd tasks are preserved, not lost — they just need to be re-driven or regenerated:

- **Bulk — redrive the DLQ** (re-enqueues the dead-lettered tasks for normal processing):
  ```
  tdbg --address <frontend:7233> dlq --dlq-version v2 merge --dlq-type 2 --cluster <clusterName> --last-message-id <id>
  ```
  Test on a small batch first, especially on SQL backends.
- **Per stuck activity — pause then unpause the activity.** This regenerates its activity task with a fresh `stamp` and dispatches a new attempt, bypassing the dead-lettered one.
- **Per stuck workflow — refresh its tasks.** Regenerates all of the workflow's tasks from current state, re-creating a dropped/dead-lettered workflow-task or timer:
  ```
  tdbg workflow refresh-tasks --workflow-id <wid> --run-id <rid>
  ```

**3. If a durable DB fix isn't immediate, buy time with dynamic config.** Two levers to stop transient outages from becoming permanent stranding:

- Raise `history.TaskDLQUnexpectedErrorAttempts` (default 70 ≈ 1h) to give the DB more time before a task dead-letters — e.g. 200 for ~3h.
- Or set `history.TaskDLQEnabled: false` to stop writing to the DLQ entirely — tasks then retry indefinitely and self-recover once the DB returns. Tradeoff: with the DLQ off, a genuinely poison/corrupt task retries forever instead of being quarantined.

### Tuning this alert

The condition is intentionally strict: any stranding-type DLQ write rate `> 0` sustained 10m. If your cluster legitimately produces occasional stranding-type DLQ writes, raise the threshold above 0 or lengthen the `for` window. The alert is cluster-wide (no namespace filter), consistent with the Essential Set.

### Relevant dynamic config

- `history.TaskDLQEnabled` — default `true`. Master switch for the history task DLQ.
- `history.TaskDLQUnexpectedErrorAttempts` — default `70` (≈ 1 hour). Attempts with unexpected errors before dead-lettering.
- `history.TaskDLQErrorPattern` — default empty. Regex; a matching task processing error dead-letters immediately.
- `history.TaskDLQInternalErrors` — default `false`. When true, `serviceerror.Internal` failures dead-letter immediately.
