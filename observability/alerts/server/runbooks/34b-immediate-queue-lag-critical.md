## Immediate Queue Lag Critical

**Severity:** Critical
**Component:** history
**Dashboard panel:** [Immediate Queue Lag per Pod](https://github.com/tsurdilo/temporal-metrics/blob/main/observability/dashboards/server/temporal-server-readme.md) — panel ID 2109

### What this alert detects

p99 immediate queue lag (tasks outstanding on the immediate processing queue) exceeds 3 million tasks on any `instance + task_category` combination for 15 consecutive minutes. The query preserves per-pod isolation (`sum by (instance, task_category, le)`), so a single stuck shard on one pod raises that pod's lag without being diluted by healthy pods in the fleet.

The 15-minute `for` duration requires 3 consecutive 5-minute metric emission cycles above threshold (the `emitShardInfoMetricsLogs` loop runs every 5 minutes). This filters transient bursts and confirms the lag is monotonically growing.

### Why it matters

A stuck shard stops processing all task categories assigned to it — transfer tasks, timer tasks, visibility tasks, and activity tasks all queue behind the stuck shard's processor. Workflows whose execution is owned by a stuck shard see no progress: timers do not fire, activities do not start, signals are not processed. The lag grows without bound until the shard is recovered.

One shard gets stuck while others remain healthy. Alert 34b fires when one `instance + task_category` line diverges from the fleet — the canonical stuck shard pattern.

### Triage steps

1. Open **Immediate Queue Lag per Pod** (panel 2109) in the Temporal Server dashboard — identify which `instance` and `task_category` are elevated; the stuck pod's line rises while all others hold steady or recover
2. Check **DB Pool Refresh Failure Rate per Pod** (panel 2111) — if this panel shows failures on the same instance, a DB connectivity loss caused the stuck shard; resolve the DB issue first
3. Check **Suspected Deadlocks per Pod** (panel 2113) — if `dd_current_suspected_deadlocks > 0` on the same instance, alert 34f should also be firing; deadlock requires pod restart
4. Check **Persistence Latencies** (panel 71) and **Shard Lock Latency** (panel 14) — elevated latency on the instance confirms DB-side pressure
5. Use `tdbg` to enumerate shards on the stuck pod and identify the specific shard ID

### Recovery path

**DB-caused stuck shard (most common):**

When a write error hits the `default` case in `handleWriteErrorLocked` (generic/unknown errors — connection failures, pool errors), the shard transitions from `contextStateAcquired` → `contextStateAcquiring` and starts a retry loop (`context_impl.go`) with exponential backoff (1s initial) and a **5-minute expiry window**. The shard keeps retrying `renewRangeLocked` without dropping.

Two sub-cases:
- **DB recovers within 5 minutes of the write failure**: the next retry in the loop succeeds → shard returns to `contextStateAcquired` within seconds of DB stability. No pod restart needed; lag drains automatically.
- **DB stays down past the 5-minute window**: the retry loop expires → shard transitions to Stopping/Stopped. The controller's `acquireShards()` loop (`history.acquireShardInterval`, default 1 minute) then picks up the unowned shard. Once DB is healthy, the new acquisition succeeds quickly. Recovery after DB is stable: **≤ 1 minute** (next controller tick).

No manual action is required if the DB is healthy. Monitor panel 2109 for the stuck pod's lag line to flatten and decline. If lag has not recovered within 5 minutes of confirmed DB stability, restart the affected history pod.

**Goroutine stall (no DB errors, no deadlock):**
- Restart the affected history pod — the shard will be re-acquired by another pod within ~1 minute (`history.acquireShardInterval` default)
- The lag on the restarting pod will spike briefly during handoff (shard movement) then clear on the new owner

### Relevant dynamic config

- `history.acquireShardInterval` — how frequently history pods attempt to acquire unowned shards (default 1m); controls how fast a dropped shard is reclaimed after the pod is restarted or the retry window expires
- `history.shardUpdateMinInterval` — minimum interval between shard info updates (default 5m); matches the metric emission frequency — this is why 3 cycles (15m) are required to confirm the shard is stuck
- `history.persistenceMaxQPS` — if hit, persistence throttling can slow the recovery retry loop
