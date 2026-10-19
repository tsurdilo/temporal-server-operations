# Temporal Shard IO Concurrency — Operator Guide

> **Current version:** v1.3.0 — see [CHANGELOG](./shard-io-concurrency-changelog.md)

**Dashboard:** `Temporal Shard IO Concurrency Dashboard v1.3.0`  
**UID:** `temporal-shard-io-v1`  
**Grafana version:** 9.0.0+  
**Datasource:** Prometheus

> **SQL backends only — PostgreSQL and MySQL.**  
> This dashboard has no actionable value on Cassandra clusters. The server hard-clamps `history.shardIOConcurrency` to `1` on Cassandra regardless of what is configured, and logs a warning at startup. If your persistence backend is Cassandra, close this dashboard.

> **Changing `history.shardIOConcurrency` requires a rolling restart of history hosts.**  
> It is a dynamic config but the value is read once at shard context creation time and baked into the semaphore — there is no hot-reload. Frontend, matching, and worker hosts are unaffected. A rolling restart is sufficient; no cluster downtime is required.

> **Start with the playbook.**  
> Whether to raise `history.shardIOConcurrency`, how far, and how to roll it out is covered step by step in the [Shard IO Concurrency on PostgreSQL and MySQL — Tuning Playbook](../../../playbooks/shard-io-concurrency.md). It names the panels on this dashboard it uses, in the order to read them.

---

## Purpose

This dashboard answers two operational questions:

1. **Should `history.shardIOConcurrency` be raised on this cluster?**
2. **If yes — what value should it be set to?**

The dashboard does not replace a full server health review. It is scoped specifically to the shard IO serialization bottleneck and the DB health prerequisite check needed to make that decision safely.

---

## Background

With `history.shardIOConcurrency=1` (the default), each history shard serializes the persistence writes that change execution state: one write is submitted, the shard waits for the DB acknowledgement, then the next write starts. Effective throughput per shard is bounded by `1 / DB_round_trip_latency`.

Raising `shardIOConcurrency` to N allows N writes to be in-flight simultaneously per shard, approaching `N / DB_round_trip_latency` until the DB's own throughput ceiling is reached.

**This only helps when the DB has headroom.** If the DB is already saturated, raising `shardIOConcurrency` increases write pressure on an already-overloaded system and drives latency higher. The panels on this dashboard are designed to distinguish these two cases.

For full context on this setting and its tradeoffs, see the [Shard IO Concurrency on PostgreSQL and MySQL — Tuning Playbook](../../../playbooks/shard-io-concurrency.md).

---

## Template Variables

| Variable | Description |
|---|---|
| `DS_PROMETHEUS` | Prometheus datasource selection |
| `namespace` | Temporal namespace filter. Use `All` for a cluster-wide view. |
| `p` | Percentile for latency panels (0.50, 0.75, 0.90, 0.95, 0.99, 0.999). Default to p99 for diagnosis. |

---

## Panel Groups

### Row 1 — Shard IO Semaphore

These panels show the health of the per-shard IO semaphore that gates concurrent persistence writes.

---

#### Panel: Shard IO Semaphore Latency (High Priority)

**Metric:** `semaphore_latency{operation="ShardInfo", priority="0"}`  
**Type:** Histogram — displayed as p99 (or selected percentile)  
**Priority filter:** `priority="0"` is high priority — active workflow traffic driven by API calls and high-priority background work. This is the traffic that matters for user-facing latency.

**What it shows:** How long a persistence write waits to acquire the shard IO semaphore. It is the wait only — the time the slot is then held for the database call is not included.

**How to use it:** [Step 1 of the Shard IO Concurrency on PostgreSQL and MySQL — Tuning Playbook](../../../playbooks/shard-io-concurrency.md#21-step-1--check-whether-writes-are-queuing-inside-shards) says how to read this panel and what to do next.

---

#### Panel: Shard IO Semaphore Failures

**Metric:** `semaphore_failures{operation="ShardInfo"}`  
**Type:** Counter — displayed as rate

**What it shows:** The rate at which callers gave up waiting for the semaphore — their context deadline expired before a slot became available.

**How to use it:** [Step 1 of the Shard IO Concurrency on PostgreSQL and MySQL — Tuning Playbook](../../../playbooks/shard-io-concurrency.md#21-step-1--check-whether-writes-are-queuing-inside-shards) says how to read this panel and what to do next.

---

#### Panel: Shard IO Deadlock Check Latency

**Metric:** `dd_shard_io_semaphore_latency`  
**Type:** Histogram — displayed per history pod at the selected percentile

**What it shows:** How long the server's deadlock detector waited to get a shard IO slot. The detector tries every shard on a timer (`system.deadlock.Interval`, default 1 minute), takes a slot and releases it straight away. It is not real write traffic — it is a probe that waits in the same queue as real writes.

**How to interpret:**

| Value | Meaning |
|---|---|
| Near zero | Normal. Slots are free often enough that the probe gets one immediately. |
| Rising, seconds | A shard's slots are held long enough that the probe queues behind real writes. Read with **Shard IO Semaphore Latency (High Priority)** and **Persistence Write Latency**. |
| Above 20s | Alert 34j fires (orange threshold). |
| 40s | The detector gives up and declares a suspected deadlock (red threshold) — see **Suspected Deadlocks (current) per Pod** on the server dashboard and alert 34f. |

The 40s limit is the detector's own timeout for this probe: 10s for the database call plus 30s grace. Does not honour `$namespace`.

---

### Row 2 — Practical Ceiling

#### Panel: Max Safe shardIOConcurrency Per Pod  (pool_size ÷ shards_per_pod)

**Metric:** `persistence_sql_max_open_conn{db_kind="main"} / on(instance) group_left() numshards_gauge{operation="ShardController"}`  
**Type:** Time series — one line per history pod (by `instance` label)

**What it shows:** The maximum value `history.shardIOConcurrency` can safely be set to on each pod, calculated as the SQL connection pool size divided by the number of shards that pod currently owns. Beyond this value, every shard having its maximum number of writes in-flight simultaneously would exhaust the connection pool.

**How to use it:** [Step 3 of the Shard IO Concurrency on PostgreSQL and MySQL — Tuning Playbook](../../../playbooks/shard-io-concurrency.md#23-step-3--find-the-highest-value-the-connection-pool-can-support) says how to read this panel and what to do next.

---

### Row 3 — Persistence / DB Health

These panels provide the prerequisite DB health check. They determine whether elevated semaphore latency (Row 1) is caused by a serialization bottleneck that `shardIOConcurrency` can fix, or by DB saturation that it cannot.

---

#### Panel: Persistence Write Latency

**Metrics:**
- `persistence_latency{operation="UpdateWorkflowExecution"}` — the most common write operation
- `persistence_latency{operation="AddTasks"}` — task writes, also gated by the semaphore
- `persistence_latency{operation="CreateWorkflowExecution"}` — new execution writes

**Type:** Histogram — displayed as p99 (or selected percentile)

**What it shows:** The DB round-trip latency for individual persistence write operations. This is the time spent inside the DB, after the shard IO semaphore slot has been acquired.

**How to use it:** [Step 2 of the Shard IO Concurrency on PostgreSQL and MySQL — Tuning Playbook](../../../playbooks/shard-io-concurrency.md#22-step-2--check-whether-the-database-is-the-slow-part) says how to read this panel and what to do next.

---

#### Panel: SQL Connection Pool Utilization & Resource Exhausted Errors

**Metrics:**
- `persistence_sql_in_use{service_name="history",db_kind="main"}` — connections currently executing a query
- `persistence_sql_max_open_conn{service_name="history",db_kind="main"}` — connection pool size (`maxConns`)
- `persistence_errors_resource_exhausted{service_name="history"}` — database calls turned away by the history service's own persistence limits

**Type:** Time series — one line per history pod, shown as a percentage, plus the resource exhausted rate on the right axis

**What it shows:** The share of each history pod's main-database connection pool in use. 100% means every connection is busy and new writes wait for one. Only history pods are shown (from v1.3.0); each line is labelled with its pod.

**How to use it:** [Step 4 of the Shard IO Concurrency on PostgreSQL and MySQL — Tuning Playbook](../../../playbooks/shard-io-concurrency.md#24-step-4--check-that-the-connection-pool-has-room-for-the-next-step-up) says how to read this panel and what to do next.

---

## Deciding whether to raise `history.shardIOConcurrency`

This is covered step by step in the [Shard IO Concurrency on PostgreSQL and MySQL — Tuning Playbook](../../../playbooks/shard-io-concurrency.md): whether writes are waiting, whether the
database and the connection pool have room, how high the value can go, how to roll it out, and how
to confirm it worked.

---

## Threshold Lines on the Panels

The orange and red lines drawn on the panels. They are general warning levels, not decision rules —
the playbook's steps decide what to do.

| Panel | Orange | Red |
|---|---|---|
| Shard IO Semaphore Latency (High Priority) | 20 ms | 100 ms |
| Shard IO Semaphore Failures | 0.001/s | 0.5/s |
| Shard IO Deadlock Check Latency | 20 s | 40 s |
| Persistence Write Latency | 20 ms | 50 ms |
| SQL Connection Pool Utilization & Resource Exhausted Errors | 80% | 95% |
