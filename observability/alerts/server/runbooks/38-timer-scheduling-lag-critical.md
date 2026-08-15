## Timer Task Scheduling Lag Critical

**Severity:** Critical
**Component:** history
**Dashboard panel:** [Timer Task Scheduling Latency](https://github.com/tsurdilo/temporal-metrics/blob/main/observability/dashboards/server/temporal-server-readme.md) — panel ID 325

### What this alert detects

p99 timer task scheduling lag exceeding 30 seconds for 5 consecutive minutes, measured via `shardinfo_scheduled_queue_lag_bucket{task_category="timer"}`.

### Why it matters

Timer lag means workflow timers, scheduled activities, and workflow timeouts are firing late. A 30s+ lag means any time-sensitive workflow logic — deadlines, heartbeat timeouts, schedule-to-start timeouts — is effectively broken. Workflows waiting on timers will not advance on time.

### Triage steps

1. Open the **Timer Task Scheduling Latency** panel (panel 325) in the Temporal Server dashboard — confirm the p99 value and when it started
2. Check **Persistence Latencies** (panel 71) — timer processing requires DB reads and writes; slow persistence is the most common cause of timer lag
3. Check **Shard Lock Latency** (panel 14) — timer tasks compete for the same shard lock as all other history operations
4. Check history pod CPU and memory — timer processing is done in-process on history pods; resource saturation causes task processing backlog
5. Note: on an idle cluster a non-zero p99 value from `histogram_quantile` on sparse data is expected and not a real signal — confirm there is actual timer activity before investigating

### Relevant dynamic config

- `history.timerProcessorSchedulerWorkerCount` — number of timer processor workers per history host (default 512); increase if timer lag persists under high timer volume
- `history.timerProcessorMaxPollHostRPS` — max poll rate for timer processors; tune to reduce DB load if timer processing is causing persistence pressure
- `history.timerProcessorCompleteTimerInterval` — how frequently the timer processor checks for due timers (default 1s)
