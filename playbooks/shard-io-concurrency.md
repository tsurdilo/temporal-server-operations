# Playbook: Should I Raise `history.shardIOConcurrency` on My SQL Cluster?

**Applies to:** PostgreSQL and MySQL backends only. If your backend is Cassandra, stop — `history.shardIOConcurrency` is hard-clamped to `1` at shard context creation and cannot be changed.

---

## References

**Dashboard:** [Shard IO Concurrency — Operator Guide](../metrics/dashboards/server/shard-io-concurrency-readme.md) — v1.1.0

| Panel | Panel Name | What to look for |
|---|---|---|
| Row 1 | Shard IO Semaphore Latency (High Priority) | `semaphore_latency{priority="0"}` elevated — writes queuing within shards |
| Row 1 | Shard IO Semaphore Failures | Non-zero rate — callers timing out waiting for a semaphore slot |
| Row 2 | Max Safe shardIOConcurrency Per Pod | Hard ceiling — pool size ÷ shards per pod. Must be ≥ 2 before incrementing. |
| Row 3 | Persistence Write Latency | DB round-trip health — must be low before raising `shardIOConcurrency` |
| Row 3 | SQL Connection Pool Utilization | Pool ratio — must be below 0.8 before raising `shardIOConcurrency` |

**Alerts:** [`metrics/alerts/server/temporal-server-alerts.yaml`](../metrics/alerts/server/temporal-server-alerts.yaml)

| Alert UID | Alert Name | Fires when |
|---|---|---|
| `temporal-alert-034j` | Shard IO Semaphore Deadlock Approaching | `dd_shard_io_semaphore_latency` p99 > 20s for 5m — half the 40s deadlock threshold |
| `temporal-alert-034f` | Shard Deadlock Detected | `dd_current_suspected_deadlocks > 0` — pod restart required, tuning no longer relevant |

**Related dynamic config:**

| Key | Default | Notes |
|---|---|---|
| `history.shardIOConcurrency` | `1` | Max concurrent persistence writes per shard. SQL only. Requires history host restart. |
| `history.shardIOTimeout` | `5s` | Timeout for individual shard IO operations. |

---

## What this playbook answers

`history.shardIOConcurrency` controls how many persistence writes a single history shard can have in-flight simultaneously. The default is `1` — fully serialized. Raising it allows more concurrent writes per shard, which can reduce semaphore latency when the DB has headroom.

This playbook walks you through the dashboard panels in order to reach a yes or no decision — and if yes, tells you what value to set and how to apply it safely.

---

## Step 1 — Open the dashboard and check Row 1

**Panel: Shard IO Semaphore Latency (High Priority)**

Look at `semaphore_latency{operation="ShardInfo", priority="0"}` p99.

- **Low and stable (sub-millisecond to low ms):** the semaphore is not a bottleneck. Stop here — raising `shardIOConcurrency` will not help.
- **Elevated (tens of ms or higher):** writes are queuing within shards. Continue to Step 2.

**Panel: Shard IO Semaphore Failures**

Check the failures rate alongside latency. This changes urgency:

| Semaphore latency | Failures | What it means |
|---|---|---|
| Elevated | Zero | Writes are queuing but getting through. You have time — proceed deliberately. |
| Elevated | Non-zero, low | Some callers are timing out. Work is being dropped and retried. Act soon. |
| Elevated | Non-zero, sustained or growing | Queue is deep enough to consistently drop work. Act now. |
| Approaching 20s | Non-zero | Alert 34j territory — pre-incident. Act immediately on root cause. |
| At or above 40s | Any | Alert 34f — suspected deadlock declared. This is no longer a tuning problem. Restart the affected history pod and see runbook `34f-shard-deadlock-detected.md`. |

---

## Step 2 — Check Row 3: Persistence Write Latency

**Panel: Persistence Write Latency**

Look at `persistence_latency` p99 for `UpdateWorkflowExecution` and `CreateWorkflowExecution`.

This is the most important check before raising `shardIOConcurrency`.

- **Low and stable (< 10ms p99):** DB is fast and has headroom. Raising `shardIOConcurrency` will likely help. Continue to Step 3.
- **Moderate and stable (10–50ms p99):** DB has some latency but is not saturated. Raising `shardIOConcurrency` may still help but gains will be smaller. Continue to Step 3 with caution — monitor DB metrics closely after each increment.
- **Elevated or rising (> 50ms p99):** the DB is the bottleneck, not the semaphore. Raising `shardIOConcurrency` adds more concurrent writers to an already-stressed DB and will make things worse. **Do not raise it.** Fix the DB first — connection pool sizing, query performance, DB instance capacity.

**The key comparison:** semaphore latency high + persistence latency low = serialization bottleneck. Both high = DB saturation. Only the first case is solved by `shardIOConcurrency`.

---

## Step 3 — Check Row 3: SQL Connection Pool Utilization

**Panel: SQL Connection Pool Utilization**

Look at the pool ratio (`persistence_sql_in_use / persistence_sql_max_open_conn`).

- **Below 0.8:** pool has headroom. Safe to raise `shardIOConcurrency`. Continue to Step 4.
- **Above 0.8:** pool is under pressure. Raising `shardIOConcurrency` may push it to exhaustion. Fix the pool first — increase `max_open_connections` on your SQL config, then re-evaluate.
- **Approaching 1.0:** do not raise `shardIOConcurrency`. Pool exhaustion will cause writes to queue for a connection before they even reach the semaphore, making things worse.

---

## Step 4 — Check Row 2: What value can I safely set?

**Panel: Max Safe shardIOConcurrency Per Pod**

This panel shows `persistence_sql_max_open_conn{db_kind="main"} / numshards_gauge` per history pod. It is the hard upper bound — setting `shardIOConcurrency` above this value risks pool exhaustion under burst load.

| Panel value | What it means |
|---|---|
| < 1 | Pool is undersized relative to shard count. Cannot safely raise above `1`. Fix the pool or reduce shards per pod first. |
| ≥ 1 and < 2 | Marginal headroom. Raising to `2` is borderline — monitor pool utilization closely. |
| ≥ 2 | Meaningful headroom. You can increment. Use the lowest value across all pods as your ceiling. |

**This is an upper bound, not a target.** The DB throughput ceiling — where adding more concurrency stops helping — is almost always reached well before the pool ceiling. Your metrics will show you where to stop.

If pods show different ceiling values (e.g., during a rolling restart or shard rebalancing), always use the lowest value across all pods as your ceiling.

---

## Step 5 — Decision

| Semaphore latency | Persistence latency | Pool ratio | Ceiling panel | Decision |
|---|---|---|---|---|
| High | Low | Low | ≥ 2 | **Yes — raise it.** Continue to Step 6. |
| High | Low | Low | < 1 | **No** — fix pool first, then re-evaluate. |
| High | Low | High | Any | **No** — fix pool first, then re-evaluate. |
| High | High | Any | Any | **No** — fix DB first, then re-evaluate. |
| Low | Any | Any | Any | **No** — semaphore is not the bottleneck. Check: `persistence_latency` (slow DB round-trip), `history_workflow_execution_cache_lock_hold_duration` (workflow lock contention), or immediate queue lag. |

---

## Step 6 — Raise it (only if Step 5 said yes)

**Dynamic config key:** `history.shardIOConcurrency`  
**Default:** `1`  
**Requires:** Rolling restart of history hosts. Frontend, matching, and worker are unaffected.

> **Pick your moment.** A rolling restart of history hosts under production load causes shards to temporarily move between pods. During the restart and until shards fully rebalance, end-to-end workflow latency will be elevated. Schedule this during lower-traffic hours unless `semaphore_failures` is sustained and work is actively being dropped.

### Procedure

1. Set the new value in your dynamic config (etcd, ConfigMap, file):
   ```
   history.shardIOConcurrency: 2
   ```
2. Perform a rolling restart of history hosts — one pod at a time.
3. After each pod restarts, wait for it to reacquire its shards and stabilize (~30–60s), then check:

| Metric | Good sign | Stop and roll back if |
|---|---|---|
| `semaphore_latency{priority="0"}` p99 | Dropping | No improvement |
| `persistence_latency{operation="UpdateWorkflowExecution"}` p99 | Stable | Rising significantly |
| Pool ratio (`persistence_sql_in_use / persistence_sql_max_open_conn`) | Stable, well below 1.0 | Approaching 1.0 |
| `semaphore_failures` rate | Dropping toward zero | Still elevated |

4. If metrics look good after the first pod, continue the rolling restart across the remaining pods.
5. Once all pods are restarted, confirm resolution (Step 7).

### When to stop incrementing

Start at `2`. If `semaphore_latency` is still elevated after stabilizing, you can increment again — but stop when either:

- `semaphore_latency` stops improving between steps — DB throughput ceiling reached.
- `persistence_latency` starts rising or pool ratio approaches 1.0 — roll back one step.

Never exceed the value shown in the **Max Safe shardIOConcurrency Per Pod** panel.

---

## Step 7 — Confirm resolution (only if Step 6 was performed)

After the rolling restart is complete and the cluster has stabilized, all of the following should be true:

- `semaphore_latency{priority="0"}` p99 returned to baseline
- `semaphore_failures` at zero
- `persistence_latency` on write operations stable
- Pool ratio not elevated
- `dd_shard_io_semaphore_latency` returned to baseline (if it was elevated)
