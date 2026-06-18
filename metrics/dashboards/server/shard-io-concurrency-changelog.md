# Changelog — Temporal Shard IO Concurrency Dashboard

## v1.1.0 — 2026-06-17

### Added

- **Max Safe shardIOConcurrency Per Pod** panel (new Row 2) — shows `persistence_sql_max_open_conn / numshards_gauge` per pod using the shared `instance` label. Gives operators the practical ceiling for `history.shardIOConcurrency` without manual math. Each line is one history pod; use the minimum across pods as the ceiling. DB throughput saturation typically occurs before this ceiling is reached, which is why values above 8 rarely help regardless of what the panel shows.

---

## v1.0.0 — 2026-06-17

### Added

Initial release. SQL backends only (PostgreSQL, MySQL) — not applicable to Cassandra. Four panels organized into two rows covering the complete diagnostic signal set for evaluating whether `history.shardIOConcurrency` should be raised and what value to set it to.

**Row 1 — Shard IO Semaphore**
- **Shard IO Semaphore Latency (High Priority)** — `semaphore_latency{priority="0"}` p99. Primary signal. Elevated values indicate writes are queuing within individual shards.
- **Shard IO Semaphore Failures** — `semaphore_failures` rate. Non-zero values indicate callers are timing out waiting for a semaphore slot — more urgent than latency alone.

**Row 2 — Persistence / DB Health**
- **Persistence Write Latency** — `persistence_latency` p99 for `UpdateWorkflowExecution` and `AddTasks`. Used in conjunction with semaphore latency to distinguish serialization bottleneck from DB saturation.
- **SQL Connection Pool Utilization** — `persistence_sql_in_use / persistence_sql_max_open_conn` ratio. Approaching 1.0 indicates DB connection pool pressure — a contra-indicator for raising `shardIOConcurrency`.
