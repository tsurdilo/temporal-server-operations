## Matching Partition Sync Throttle Active

**Severity:** Critical
**Component:** matching
**Dashboard panel:** [Sync Throttle Count](https://github.com/tsurdilo/temporal-metrics/blob/main/metrics/dashboards/server/temporal-server-readme.md) — panel ID 403

### What this alert detects

Any non-zero rate of `sync_throttle_count{service_name="matching"}` per namespace and task type for 1 consecutive minute. Fires when any matching partition's rate limiter is actively rejecting sync match dispatch attempts.

### Why it matters

`sync_throttle_count` increments on the **sync match path** — when a new task arrives at a partition and tries to pair directly with a waiting poller without writing to the database. It fires when `rateLimiter.Wait()` times out within the `matching.syncMatchWaitDuration` window (default 200ms), meaning the per-partition rate limit was saturated and could not grant a token in time.

The effective per-partition rate limit is `min(admin.matchingNamespaceTaskqueueToPartitionDispatchRate, admin.matchingNamespaceToPartitionDispatchRate)` — by default `min(1,000, 10,000)` = 1,000/s per partition. This is not a hard-coded limit; both values are dynamic config and can be tuned.

**Important:** this metric is silent when the backlog is non-negligible. When backlog age exceeds `matching.backlogNegligibleAge` (default 5s), sync match is bypassed entirely and the rate limiter is never consulted — so a zero rate here does not rule out partition saturation. If the backlog has already grown, watch Tasks Persisted to DB and Approximate Task Backlog instead.

If this fires without alert 57 (All Pollers Disconnected) also firing, the task queues are hitting their per-partition dispatch ceiling with workers present.

### Triage steps

1. Open the **Sync Throttle Count** panel (panel 403) in the Temporal Server dashboard — confirm which namespace and task type are affected
2. Check if alert 57 (All Pollers Disconnected) is also firing — if so the throttling is a symptom of no workers, not a partition sizing issue
3. Check **Async Match Latency** and **Schedule to Start Latencies** — elevated latency alongside throttling confirms dispatch is backing up with workers present
4. Check the current number of task queue partitions for the affected task queue — the rate limit is applied per partition, so more partitions means a higher total dispatch ceiling; sticky task queues always use 1 partition and cannot be partitioned
5. If the alert fires briefly then goes quiet while backlog metrics climb, sync match has been suppressed by the backlog guard — shift focus to Tasks Persisted to DB and Approximate Task Backlog

### Relevant dynamic config

- `admin.matchingNamespaceTaskqueueToPartitionDispatchRate` — per-partition dispatch rate limit for a specific namespace + task queue (default 1,000/s); this is the binding limit in most cases
- `admin.matchingNamespaceToPartitionDispatchRate` — per-partition dispatch rate limit for an entire namespace across all task queues (default 10,000/s); effective rate is the min of both
- `matching.numTaskqueueWritePartitions` — number of write partitions per task queue (default 1); increasing this multiplies the total dispatch capacity proportionally
- `matching.numTaskqueueReadPartitions` — number of read partitions (default 1); should match write partitions
- `matching.syncMatchWaitDuration` — how long a sync match attempt waits before falling back to async (default 200ms); the rate limiter wait is bounded by this window
- `matching.backlogNegligibleAge` — threshold above which a backlog is considered significant and sync match is bypassed (default 5s); if this alert fires intermittently but Tasks Persisted is climbing, the guard is suppressing the metric
