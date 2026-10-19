# Changelog — Temporal Shard IO Concurrency Dashboard

## v1.3.0 — 2026-10-19

### Changed

- **SQL Connection Pool Utilization & Resource Exhausted Errors (4)**: both lines now show the
  history service only; the pool line is one line per history pod, labelled with the pod. The
  description no longer gives fixed utilization bands.

### Documentation

- Readme: the Decision Guide and the per-panel decision tables are replaced by links to the
  Shard IO Concurrency on PostgreSQL and MySQL — Tuning Playbook, which is now the one place the
  procedure lives. The threshold table lists the lines actually drawn on each panel.

---

## v1.2.0 — 2026-10-19

### Added

- **Shard IO Deadlock Check Latency (6)**, on the **Shard IO Semaphore** row: the deadlock
  detector's wait for a shard IO slot (`dd_shard_io_semaphore_latency`), per pod, with lines at
  20s and 40s.

### Changed

- **Shard IO Semaphore Latency (High Priority) (1)** and **Shard IO Semaphore Failures (2)**
  narrowed; the row is now three panels across.
- Alert 34j now links to panel 6 instead of panel 1.

### Fixed

- **Shard IO Semaphore Latency (High Priority) (1)** description: the wait only, not the time
  the slot is held.

### Documentation

- Readme: added the `Current version` line; panel headings now use the exact panel titles.

---

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
