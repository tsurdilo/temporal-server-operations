# Alert 34j — Shard IO Semaphore Deadlock Approaching

**Severity:** Warning  
**Component:** history  
**Metric:** `dd_shard_io_semaphore_latency`

---

## What this alert detects

`dd_shard_io_semaphore_latency` p99 exceeding 20 seconds, sustained for 5 minutes.

This metric is emitted by Temporal's internal deadlock detector — not by normal write traffic. The deadlock detector periodically tries to acquire and immediately release the shard IO semaphore as a health ping. If the ping cannot acquire the semaphore quickly, it means the semaphore is so heavily contended that even the background health check is being queued behind live traffic.

The deadlock detector's timeout for this ping is **40 seconds**, derived directly from source (`service/history/shard/context_impl.go`):

```go
// ioSemaphore is for the duration of a persistence op which has a persistence connection timeout of 10 sec.
Timeout: 10*time.Second + 30*time.Second,
```

If the ping cannot acquire the semaphore within 40 seconds, the deadlock detector declares the shard a suspected deadlock — which triggers alert 34f (`dd_current_suspected_deadlocks > 0`) and requires an immediate pod restart.

**This alert fires at 20s p99 — half the 40s threshold — to give operators time to act before the deadlock is declared.**

---

## Why it matters

`dd_shard_io_semaphore_latency` is distinct from `semaphore_latency` on normal write traffic:

- `semaphore_latency` — measured on every gated persistence write. Elevated values indicate writes are queuing within shards.
- `dd_shard_io_semaphore_latency` — measured only on the deadlock detector's health ping. Elevated values mean contention is severe enough to starve the background health check.

A spike in `dd_shard_io_semaphore_latency` with `semaphore_latency` also elevated indicates the shard is approaching a state where the deadlock detector will fire (alert 34f). A pod declared deadlocked must be restarted — there is no self-healing path.

---

## Triage steps

**1. Check `semaphore_latency{operation="ShardInfo",priority="0"}` p99**

If elevated alongside `dd_shard_io_semaphore_latency`, active write traffic is saturating the semaphore. The semaphore queue is so deep that even the health ping cannot get a slot within the detection window.

**2. Check DB health**

Look at `persistence_latency` on write operations (`UpdateWorkflowExecution`, `AddTasks`, `CreateWorkflowExecution`) and `persistence_sql_in_use / persistence_sql_max_open_conn`.

- If `persistence_latency` is elevated: DB writes are slow, so each semaphore slot is held longer, which deepens the queue. The DB is the root cause — fix it first. Do NOT raise `shardIOConcurrency` while the DB is under pressure.
- If `persistence_latency` is healthy: the semaphore depth is driven by write volume, not write duration. This is where raising `shardIOConcurrency` may help (SQL backends only).

**3. Check `semaphore_failures{operation="ShardInfo"}`**

A non-zero rate means callers are already giving up waiting for the semaphore. If both `semaphore_failures` and `dd_shard_io_semaphore_latency` are elevated, the situation is acute — work is being actively dropped.

**4. Check which pods are affected**

`dd_shard_io_semaphore_latency` is emitted per pod via the `instance` label. If only specific pods are affected, check `persistence_shard_rps` for uneven shard distribution — a hot-shard problem, not a global serialization problem.

---

## Remediation

### If DB is the root cause (`persistence_latency` elevated)

Do not touch `shardIOConcurrency`. Raising it adds more concurrent writers to an already-stressed DB and accelerates degradation.

1. Identify and address the DB issue — slow queries, connection pool exhaustion, DB CPU saturation
2. Once DB metrics recover, `semaphore_latency` and `dd_shard_io_semaphore_latency` should return to baseline organically

### If DB is healthy and backend is SQL (PostgreSQL / MySQL)

Consider raising `history.shardIOConcurrency` from its default of `1`. This is a dynamic config but **requires a rolling restart of history hosts to take effect** — the value is baked into the semaphore at shard context creation time.

**This alert is Warning severity for a reason — you have time to pick your moment.** A rolling restart of history hosts under production load will cause shards to temporarily move between pods. During the restart and until shards fully rebalance, end-to-end workflow latency will be elevated and your team should be aware. Schedule this change during a period of lower traffic or reduced business sensitivity. Do not rush it to resolve the alert faster — the alert is telling you the semaphore is under stress, not that a deadlock has been declared. If `dd_shard_io_semaphore_latency` is climbing toward 40s, that urgency changes — see the section below.

Before changing it, read the practical ceiling from the shard IO concurrency dashboard:

```
max safe value = persistence_sql_max_open_conn / numshards_gauge (per pod)
```

Start at `2`. After each rolling restart, check that `semaphore_latency` improved and DB metrics remained stable before incrementing further. Stop when `semaphore_latency` stops improving — that is the DB throughput ceiling.

See `observability/dashboards/server/shard-io-concurrency-readme.md` for the full step-by-step guide.

### If backend is Cassandra

`shardIOConcurrency` is hard-clamped to `1` on Cassandra and cannot be raised. If the semaphore is saturated on Cassandra, the only remediation paths are:
- Reducing write load on the affected shards
- Increasing shard count (cluster migration required)
- Addressing DB (Cassandra) performance

### If `dd_shard_io_semaphore_latency` is approaching 40s

The shard is close to being declared deadlocked (alert 34f). If you cannot resolve the root cause quickly:

1. Consider proactively restarting the affected history pod to force shard re-acquisition by another pod
2. Monitor `dd_current_suspected_deadlocks` — if it becomes non-zero, an immediate restart is required (see runbook `34f-shard-deadlock-detected.md`)

---

## Relationship to other alerts

| Alert | Metric | Relationship |
|---|---|---|
| **34j (this alert)** | `dd_shard_io_semaphore_latency` | Pre-fire warning — semaphore contention is severe enough to starve the deadlock detector health ping |
| **34f** | `dd_current_suspected_deadlocks` | The fire — deadlock declared, pod restart required. Fires after `dd_shard_io_semaphore_latency` exceeds 40s. |
| **8** | `semaphore_latency` | Normal write traffic semaphore latency — usually elevated alongside this alert |
| **34e** | DB pool refresh failures | DB-caused semaphore saturation — check this if persistence_latency is also elevated |

---

## Relevant dynamic configs

| Key | Default | Description |
|---|---|---|
| `history.shardIOConcurrency` | 1 | Max concurrent persistence writes per shard. SQL only — Cassandra hard-clamped to 1. Requires history host restart. |
| `history.shardIOTimeout` | 5s | Timeout for individual shard IO operations. |
