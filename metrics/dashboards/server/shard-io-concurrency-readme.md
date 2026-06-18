# Temporal Shard IO Concurrency — Operator Guide

**Dashboard:** `Temporal Shard IO Concurrency Dashboard v1.1.0`  
**UID:** `temporal-shard-io-v1`  
**Grafana version:** 9.0.0+  
**Datasource:** Prometheus

> **SQL backends only — PostgreSQL and MySQL.**  
> This dashboard has no actionable value on Cassandra clusters. The server hard-clamps `history.shardIOConcurrency` to `1` on Cassandra regardless of what is configured, and logs a warning at startup. If your persistence backend is Cassandra, close this dashboard.

> **Changing `history.shardIOConcurrency` requires a rolling restart of history hosts.**  
> It is a dynamic config but the value is read once at shard context creation time and baked into the semaphore — there is no hot-reload. Frontend, matching, and worker hosts are unaffected. A rolling restart is sufficient; no cluster downtime is required.

> **Start with the [Decision Guide](#decision-guide).**  
> The panels on this dashboard feed a decision matrix that tells you whether to raise `history.shardIOConcurrency` and what value to set it to. If you opened this dashboard because of an alert or a latency complaint, skip to the Decision Guide first — it will tell you which panels to read and in what order.

---

## Purpose

This dashboard answers two operational questions:

1. **Should `history.shardIOConcurrency` be raised on this cluster?**
2. **If yes — what value should it be set to?**

The dashboard does not replace a full server health review. It is scoped specifically to the shard IO serialization bottleneck and the DB health prerequisite check needed to make that decision safely.

---

## Background

With `history.shardIOConcurrency=1` (the default), each history shard serializes all persistence writes end-to-end: one write is submitted, the shard waits for the DB acknowledgement, then the next write starts. Effective throughput per shard is bounded by `1 / DB_round_trip_latency`.

Raising `shardIOConcurrency` to N allows N writes to be in-flight simultaneously per shard, approaching `N / DB_round_trip_latency` until the DB's own throughput ceiling is reached.

**This only helps when the DB has headroom.** If the DB is already saturated, raising `shardIOConcurrency` increases write pressure on an already-overloaded system and drives latency higher. The panels on this dashboard are designed to distinguish these two cases.

For full context on this setting, the tradeoffs, and the comparison with increasing shard count, see [`tmp/history-shard-io-concurrency.md`](../../../../tmp/history-shard-io-concurrency.md).

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

**What it shows:** How long a persistence write waits to acquire the shard IO semaphore, including both queue wait time and the time the slot is held during the DB round-trip.

**How to interpret:**

| Value | Meaning |
|---|---|
| Low and stable | Semaphore is not a bottleneck. No action needed. |
| Elevated, rising gradually | Writes are queuing within shards. Check Row 2 to determine cause. |
| Elevated with `persistence_latency` low (Row 2) | Serialization bottleneck. DB has headroom. Consider raising `shardIOConcurrency`. |
| Elevated with `persistence_latency` also high (Row 2) | DB is the bottleneck, not serialization. Do **not** raise `shardIOConcurrency`. Fix the DB first. |
| Spiking to 40s+ | Deadlock detector threshold approached. Urgent — the shard may be declared deadlocked. |

**Note on `priority="1"` (low priority):** Standby replication tasks and NDC replication run at low priority. If only `priority="1"` is elevated while `priority="0"` is healthy, active traffic is unaffected — this is a replication lag issue, not a shard IO concurrency issue.

---

#### Panel: Shard IO Semaphore Failures

**Metric:** `semaphore_failures{operation="ShardInfo"}`  
**Type:** Counter — displayed as rate

**What it shows:** The rate at which callers gave up waiting for the semaphore — their context deadline expired before a slot became available.

**How to interpret:**

| Value | Meaning |
|---|---|
| Zero | Normal. |
| Non-zero, low rate | Semaphore is occasionally saturated under burst load. Monitor. |
| Non-zero, sustained or growing | Active work is being dropped. The queue depth is deep enough that callers time out before getting a slot. Treat as urgent alongside elevated semaphore latency. |

A non-zero `semaphore_failures` rate combined with elevated `semaphore_latency` is a stronger signal than either alone. Callers are not just waiting longer — they are giving up. This changes both the urgency and the decision:

| `semaphore_latency` | `semaphore_failures` | What it means | Urgency |
|---|---|---|---|
| Elevated | Zero | Writes queuing but getting through. No work dropped. | Monitor — schedule change at low-traffic time |
| Elevated | Non-zero, low | Some callers timing out. Work is being dropped and retried. | Urgent — raise `shardIOConcurrency` soon if DB is healthy |
| Elevated | Non-zero, sustained/growing | Queue depth is deep enough to consistently drop work. | Very urgent — act now |
| Approaching 20s | Non-zero | Approaching alert 34j threshold. | Pre-incident — act immediately on root cause |
| At/above 40s | Any | Deadlock detector threshold reached — alert 34f fires. | Incident — pod restart, not a tuning problem |

---

### Row 2 — Practical Ceiling

#### Panel: Max Safe shardIOConcurrency Per Pod

**Metric:** `persistence_sql_max_open_conn{db_kind="main"} / on(instance) group_left() numshards_gauge{operation="ShardController"}`  
**Type:** Time series — one line per history pod (by `instance` label)

**What it shows:** The maximum value `history.shardIOConcurrency` can safely be set to on each pod, calculated as the SQL connection pool size divided by the number of shards that pod currently owns. Beyond this value, every shard having its maximum number of writes in-flight simultaneously would exhaust the connection pool.

**How to interpret:**

| Panel value | Meaning | Action |
|---|---|---|
| < 1 | Connection pool is undersized relative to shard count. The default of `1` is already the maximum safe value. | Do not raise `shardIOConcurrency`. Raising it risks pool exhaustion. If `semaphore_latency` is elevated, the fix is increasing the SQL connection pool size or reducing shards per pod — not raising this config. |
| ≥ 1 and < 2 | Marginal headroom. Raising to `2` is borderline. | Only attempt if `semaphore_latency` is clearly elevated and `persistence_latency` is healthy. Monitor pool utilization (Row 3) closely after the restart. |
| ≥ 2 | Meaningful headroom exists. You can increment safely. | Follow the step-by-step procedure in the Decision Guide. Use this value as your hard upper bound — stop incrementing when `semaphore_latency` stops improving, which will happen before you reach this ceiling. |

**This panel is a hard upper bound, not a recommendation.** Seeing a ceiling of `8` does not mean you should set `shardIOConcurrency=8`. It means `8` is the maximum that won't exhaust the pool. The DB throughput ceiling — the point where adding more concurrency stops helping — is almost always reached well before the pool ceiling. Your metrics (Row 1 semaphore latency, Row 3 persistence latency) tell you where that real stopping point is.

If pods own different numbers of shards (e.g., during a rolling restart or shard rebalancing), lines will diverge temporarily — always use the lowest value across pods as your ceiling.

---

### Row 3 — Persistence / DB Health

These panels provide the prerequisite DB health check. They determine whether elevated semaphore latency (Row 1) is caused by a serialization bottleneck that `shardIOConcurrency` can fix, or by DB saturation that it cannot.

**Rule:** Do not raise `shardIOConcurrency` if any panel in this row shows an unhealthy state.

---

#### Panel: Persistence Write Latency

**Metrics:**
- `persistence_latency{operation="UpdateWorkflowExecution"}` — the most common write operation
- `persistence_latency{operation="AddTasks"}` — task writes, also gated by the semaphore
- `persistence_latency{operation="CreateWorkflowExecution"}` — new execution writes

**Type:** Histogram — displayed as p99 (or selected percentile)

**What it shows:** The DB round-trip latency for individual persistence write operations. This is the time spent inside the DB, after the shard IO semaphore slot has been acquired.

**How to interpret:**

| Value | Meaning |
|---|---|
| Low and stable (< 10ms p99) | DB is healthy and fast. If `semaphore_latency` is also elevated, serialization is the bottleneck — `shardIOConcurrency` will help. |
| Moderate and stable (10–50ms p99) | DB has some latency but is not saturated. `shardIOConcurrency` may still help but gains will be smaller. |
| Elevated and rising (> 50ms p99) | DB is under pressure. Do **not** raise `shardIOConcurrency`. |
| Elevated with `persistence_errors_resource_exhausted` non-zero | DB is rejecting requests. Raising `shardIOConcurrency` is actively harmful here. |

**The key diagnostic comparison:** look at this panel and the semaphore latency panel together.

- `semaphore_latency` high + `persistence_latency` low → **serialization bottleneck** → raise `shardIOConcurrency`
- Both high → **DB saturation** → fix the DB, do not raise `shardIOConcurrency`
- `semaphore_latency` low + `persistence_latency` high → DB is slow but the semaphore is not the bottleneck — the shard IO pipeline is not the limiting factor

---

#### Panel: SQL Connection Pool Utilization

**Metrics:**
- `persistence_sql_in_use` (by `db_kind`) — connections currently executing a query
- `persistence_sql_max_open_conn` (by `db_kind`) — connection pool size
- Displayed as ratio: `persistence_sql_in_use / persistence_sql_max_open_conn`

**Type:** Gauge / time series — ratio from 0.0 to 1.0

**What it shows:** What fraction of the SQL connection pool is actively in use. This ratio (`persistence_sql_in_use / persistence_sql_max_open_conn`) is what the Decision Guide refers to as the **pool ratio**. It ranges from 0.0 (pool idle) to 1.0 (pool fully exhausted). A ratio approaching 1.0 means the pool is exhausted — new write requests queue waiting for a free connection rather than executing immediately.

**How to interpret:**

| Ratio | Meaning |
|---|---|
| < 0.5 | Pool has ample headroom. Raising `shardIOConcurrency` is safe from a connection perspective. |
| 0.5 – 0.8 | Moderate utilization. Raising `shardIOConcurrency` will increase this further — monitor carefully. |
| > 0.8 | Pool is under pressure. Raising `shardIOConcurrency` may push it to exhaustion. |
| Approaching 1.0 | Pool exhaustion. Do **not** raise `shardIOConcurrency`. Adding more concurrent writers will cause connection queuing and drive `persistence_latency` higher. |

**Note:** The `db_kind` label distinguishes between database engines (e.g., `postgres`, `mysql`). If multiple SQL stores are configured, each will appear as a separate series.

---

## Decision Guide

### Step 1 — Should you raise it?

| `semaphore_latency` | `persistence_latency` | Pool ratio | Action |
|---|---|---|---|
| High | Low | Low | Yes — serialization bottleneck, DB has headroom. Continue to Step 2. |
| High | Low | High | Fix connection pool or DB capacity first, then re-evaluate |
| High | High | Any | DB is saturated — fix DB first, do not raise `shardIOConcurrency` |
| Low | Any | Any | `shardIOConcurrency` is not the bottleneck — look elsewhere |
| Any | Any | High | DB connection pool pressure — investigate pool sizing or query performance |

### Step 2 — What value to set?

**Dynamic config key:** `history.shardIOConcurrency`  
**Default:** `1`  
**Requires:** Rolling restart of history hosts only (frontend, matching, worker unaffected)

The value controls the IO pipeline depth per shard. It is not a global throughput dial — it multiplies per shard. With 512 shards on a pod and `shardIOConcurrency=4`, that pod can have up to 2048 concurrent DB writes in-flight under burst load. This directly affects connection pool utilization.

**Principled ceiling — never exceed this:**

```
max safe value ≈ persistence_sql_max_open_conn / shards_per_pod
```

This value is displayed directly in the **"Max Safe shardIOConcurrency Per Pod"** panel (Row 2 of this dashboard), calculated per pod using the `instance` label that both metrics share. Read it there — no manual math needed. In practice stop well below the ceiling: the DB throughput ceiling is usually hit before pool exhaustion, which is why values above 8 rarely help.

**Before incrementing — check the ceiling panel first:**

If the **"Max Safe shardIOConcurrency Per Pod"** panel (Row 2) shows a value **below 1**, stop here. Your SQL connection pool is too small relative to the number of shards per pod. Raising `shardIOConcurrency` will exhaust the pool. The fix is increasing `persistence_sql_max_open_conn` (SQL pool size) or adding more history pods to reduce shards per pod — not raising this config. Come back to this guide after the pool is resized.

If the panel shows **1 or above**, continue.

**How to increment:**

Start at `2`. After each step, do a rolling restart of history hosts, wait for the cluster to stabilize, then re-read this dashboard — specifically the semaphore latency panel and the connection pool panel. Do not jump multiple steps at once.

**Continue incrementing if:**
- `semaphore_latency` improved and is still elevated
- `persistence_latency` on write operations remained stable
- Connection pool ratio stayed well below 1.0

**Stop incrementing when either of these is true:**
- `semaphore_latency` is no longer improving between steps — the DB throughput ceiling has been reached. Further increments add load without adding throughput.
- `persistence_latency` is rising or the connection pool ratio is approaching 1.0 — roll back one step.

Never exceed the value shown in the **"Max Safe shardIOConcurrency Per Pod"** panel — that is the hard pool ceiling. The DB throughput ceiling is typically reached well before the pool ceiling, which is why the metrics above will tell you when to stop before you ever approach it.

**If raising to `2` produces no improvement in `semaphore_latency`:** the bottleneck is not write serialization. Check whether `persistence_latency` itself is elevated (the round-trip is slow regardless of concurrency) or whether `history_workflow_execution_cache_lock_hold_duration` is high (tasks are slow to complete while holding the semaphore slot). Neither is fixed by `shardIOConcurrency`.

---

## Thresholds Summary

| Panel | Orange | Red |
|---|---|---|
| Semaphore latency p99 | > 20ms | > 100ms |
| Semaphore failures rate | > 0 | > 0.5/s |
| Persistence write latency p99 | > 20ms | > 50ms |
| Connection pool ratio | > 0.8 | > 0.95 |

