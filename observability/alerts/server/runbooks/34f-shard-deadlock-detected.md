## Shard Deadlock Detected

**Severity:** Critical
**Component:** history
**Dashboard panel:** [Suspected Deadlocks per Pod](https://github.com/tsurdilo/temporal-metrics/blob/main/observability/dashboards/server/temporal-server-readme.md) — panel ID 2113

### What this alert detects

`dd_current_suspected_deadlocks{service_name="history"} > 0` on any history pod. The metric is a gauge emitted by Temporal's internal deadlock detector (`DeadlockDetector`) when a goroutine has been blocked holding a lock for longer than the configured detection threshold. The gauge is event-driven — it emits only when a deadlock is suspected; it does not emit baseline zeros. `noDataState: OK` is required so that the normal state (no data = no deadlocks) does not trigger a false alert.

The alert fires after 1 minute sustained — long enough to confirm the gauge is not a transient sampling artifact, short enough that the pod has not yet crashed.

### Why it matters

A Go lock deadlock on a history pod causes all goroutines waiting on that lock to stall indefinitely. Because history shards share the pod's goroutine pool, a deadlock can block shard processors for any or all shards on that pod — queue lag (panel 2109) begins building immediately. Unlike DB-caused stuck shards, a deadlocked pod does not self-heal; the goroutine is permanently blocked until the pod is restarted.

The deadlock detector (`dd_current_suspected_deadlocks`) fires only for lock-based goroutine stalls — it does not fire for DB connectivity loss (see alert 34e for that signal).

### Triage steps

1. Open **Suspected Deadlocks per Pod** (panel 2113) — confirm which `instance` is reporting a non-zero value
2. Check **Deadlock Event Rate per Pod** (panel 2114) — shows the cumulative rate of deadlock detection events; useful after the gauge has cleared (post-restart) to confirm the deadlock was real and not a gauge sampling artifact
3. Check **Immediate Queue Lag per Pod** (panel 2109) — confirm lag is building on the same instance; the stuck pod's lag will be rising while other pods hold steady
4. Check history pod logs on the identified instance for `"DeadlockDetector: potential deadlock detected"` — this log line is emitted at the same time the gauge is set; it includes the goroutine stack trace that identifies the lock holder
5. Check **DB Pool Refresh Failure Rate per Pod** (panel 2111) — if this is also elevated, the deadlock may have been triggered by a blocked DB call rather than a pure lock cycle; in that case, resolve the DB issue first but still restart the pod

### Recovery

**A pod reporting `dd_current_suspected_deadlocks > 0` must be restarted.** There is no self-healing path — the blocked goroutine will not unblock. The shard will be re-acquired by another pod within ~1 minute (`history.acquireShardInterval` default).

Steps:
1. Identify the affected pod from panel 2113 or from the alert label `instance`
2. Capture the pod logs before restarting — the deadlock detector log entry contains the goroutine stack trace needed to file a bug report
3. Restart the pod
4. Confirm on **Suspected Deadlocks per Pod** (panel 2113) that the gauge goes absent (returns to noData state) after restart
5. Confirm on **Immediate Queue Lag per Pod** (panel 2109) that the lag on the restarting pod clears within 5–10 minutes of restart

### Relevant dynamic config

- `history.deadlockDetectorLockWaitTime` — how long a goroutine must be blocked before the detector fires (default 1s); lower values increase false positive rate on a loaded cluster; higher values delay detection
- `history.acquireShardInterval` — controls how fast the restarted pod's shards are claimed by another pod (default 1m); also controls how fast they return to the restarted pod after it comes back up
