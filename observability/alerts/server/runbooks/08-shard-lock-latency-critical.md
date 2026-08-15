## Shard Lock Latency Critical

**Severity:** Critical
**Component:** history
**Dashboard panel:** [Shard Lock Latency](https://github.com/tsurdilo/temporal-metrics/blob/main/observability/dashboards/server/temporal-server-readme.md) — panel ID 14

### What this alert detects

p99 shard lock latency exceeding 300ms for 5 consecutive minutes.

### Why it matters

High shard lock latency can indicate issues with the history service acquiring shards, elevated DB latencies holding the lock longer than normal, or "hot shard" conditions where a specific shard is under disproportionate pressure — for example, a workflow that reuses the same workflow ID at high frequency will always route to the same shard.

### Triage steps

1. Open the **Shard Lock Latency** panel (panel 14) in the Temporal Server dashboard — confirm the p99 value and which instances are affected
2. Check **Persistence Latencies** (panel 71) — slow DB operations hold the shard lock longer, causing other goroutines to queue; check if alert 12 is also firing
3. Check **Workflow Lock Latency** — if both are elevated the history service is under broad contention, not a single hot shard
4. Check CPU and memory on history pods — lock contention often accompanies resource saturation

### Relevant dynamic config

- `history.cacheMaxSize` — per-shard mutable state cache size in entries (default 512); too small causes frequent DB reads while holding the lock
- `history.hostLevelCacheMaxSize` — host-level mutable state cache in entries (default 128K)
- `history.cacheSizeBasedLimit` — if `true`, cache is limited by bytes instead of entry count; pair with `history.cacheMaxSizeBytes` and `history.hostLevelCacheMaxSizeBytes`
- `history.enableHostHistoryCache` — whether the host-level cache is used instead of per-shard (default `true` in newer versions)
- `history.shardIOConcurrency` — concurrency of persistence operations per shard (default 1); increasing this reduces shard lock contention on SQL backends but **requires a restart**
