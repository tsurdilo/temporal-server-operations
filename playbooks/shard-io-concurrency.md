# Shard IO Concurrency on PostgreSQL and MySQL — Tuning Playbook

**Temporal is a durable execution platform, and every change to a workflow execution's state is
written by exactly one owner.** Workflow executions are spread across history shards, and each
shard is owned by one history pod at a time. Each change also checks that the history pod writing
it still owns the shard, so when a shard moves between pods, the old owner can never write over the
new owner's work. On Cassandra, Temporal enforces the single-writer guarantee with Cassandra's
lightweight transactions, and writes the changes on one shard one at a time. To keep the same
guarantee on PostgreSQL and MySQL, Temporal writes each change to execution state in a single database
transaction that first checks, under a shared lock on the shard's record, that the history pod still owns the
shard. Shared locks do not block each other, so on these stores several changes on one shard can be
checked and written at the same time.

**Temporal enforces its single-writer guarantee on the most important write in a durable
execution: the one that saves an execution's new history events and state.** Executions are spread
across all of the cluster's shards by their namespace and workflow ID, so the number of shards sets
how many of these writes the cluster can have in flight at once.

**Because on PostgreSQL and MySQL each of these writes runs in its own database transaction,
several can safely be in flight on one shard at once.** How many is set by the dynamic config
`history.shardIOConcurrency`, default 1 — the same as on Cassandra. This playbook covers whether
raising it will help your cluster, how far to go, and how to roll it out.

**Audience**

Operators of self-hosted Temporal clusters.

**This playbook does not cover**

- **One shard carrying far more load than the rest.** Raising `history.shardIOConcurrency` lets that shard
  write faster; it does not move any of its load elsewhere. That is
  [Hot Shard Detection and Remediation](./hot-shard-detection-remediation.md).
- **The limits on how many database calls the history service may make.** Those are
  [History Persistence QPS Limits](./history-persistence-qps-limits.md).
- **Shard State Lock Latency.** This playbook covers the wait on
  [**Shard Lock Latency**](../observability/dashboards/server/temporal-server-readme.md#2-shard-and-workflow-lock-latencies), not on the similarly named
  [**Shard State Lock Latency**](../observability/dashboards/server/temporal-server-readme.md#2-shard-and-workflow-lock-latencies) — a different lock that `history.shardIOConcurrency` does not
  change.

**Dashboards**

**The Shard IO Concurrency dashboard is the one to work from** — every step in this playbook reads
its panels. The Temporal Server dashboard is used only for a few specific panels:
**Suspected Deadlocks (current) per Pod**, behind alert
[34f — Shard Deadlock Detected](../observability/alerts/server/runbooks/34f-shard-deadlock-detected.md);
**Shard Lock Latency**, an overall view of how long changes to execution state wait their turn on a
shard; **Shard State Lock Latency**, to tell that panel apart from Shard Lock Latency; and the shard
movement panels, to see when shards have settled during the restarts in
[Step 6](#31-step-6--raise-historyshardioconcurrency-one-step-at-a-time).

| Dashboard | Panels this playbook uses |
|---|---|
| [Shard IO Concurrency](../observability/dashboards/server/shard-io-concurrency-readme.md) | [**Shard IO Semaphore Latency (High Priority)**](../observability/dashboards/server/shard-io-concurrency-readme.md#panel-shard-io-semaphore-latency-high-priority)<br>[**Shard IO Semaphore Failures**](../observability/dashboards/server/shard-io-concurrency-readme.md#panel-shard-io-semaphore-failures)<br>[**Shard IO Deadlock Check Latency**](../observability/dashboards/server/shard-io-concurrency-readme.md#panel-shard-io-deadlock-check-latency)<br>[**Max Safe shardIOConcurrency Per Pod  (pool_size ÷ shards_per_pod)**](../observability/dashboards/server/shard-io-concurrency-readme.md#panel-max-safe-shardioconcurrency-per-pod--pool_size--shards_per_pod)<br>[**Persistence Write Latency**](../observability/dashboards/server/shard-io-concurrency-readme.md#panel-persistence-write-latency)<br>[**SQL Connection Pool Utilization & Resource Exhausted Errors**](../observability/dashboards/server/shard-io-concurrency-readme.md#panel-sql-connection-pool-utilization--resource-exhausted-errors) |
| [Temporal Server](../observability/dashboards/server/temporal-server-readme.md) | [**Shard Lock Latency**](../observability/dashboards/server/temporal-server-readme.md#2-shard-and-workflow-lock-latencies)<br>[**Shard State Lock Latency**](../observability/dashboards/server/temporal-server-readme.md#2-shard-and-workflow-lock-latencies)<br>[**Suspected Deadlocks (current) per Pod**](../observability/dashboards/server/temporal-server-readme.md#9-shard-queue-health)<br>[**Owned Shards (Total)**](../observability/dashboards/server/temporal-server-readme.md#8-shard-movement)<br>[**Shards Created**](../observability/dashboards/server/temporal-server-readme.md#8-shard-movement)<br>[**Shards Removed**](../observability/dashboards/server/temporal-server-readme.md#8-shard-movement)<br>[**Shards Closed**](../observability/dashboards/server/temporal-server-readme.md#8-shard-movement) |

**Alerts**

| Alert | Fires when |
|---|---|
| [34j — Shard IO Semaphore Deadlock Approaching](../observability/alerts/server/runbooks/34j-shard-io-semaphore-deadlock-approaching.md) | the server's deadlock check has waited more than 20 seconds for its turn to write on a shard |
| [34f — Shard Deadlock Detected](../observability/alerts/server/runbooks/34f-shard-deadlock-detected.md) | the deadlock check has given up and declared a suspected deadlock |

**Applies to**

- **SQL persistence — PostgreSQL and MySQL**, the only stores where `history.shardIOConcurrency`
  can be raised. Cassandra is not applicable to this playbook.
- **Clusters whose number of history pods stays fairly steady.** If your cluster scales its history
  pods up and down often, the recommendation is to leave `history.shardIOConcurrency` at 1: a
  value that is safe at one size can be too high after a scale-down, because each remaining pod
  then owns more shards. [Section 4](#4-when-to-re-check-historyshardioconcurrency) explains why, and how to size `history.shardIOConcurrency`
  if you decide to raise the value on such a cluster anyway.

---

## Contents

1. [How changes to execution state on one shard wait their turn](#1-how-changes-to-execution-state-on-one-shard-wait-their-turn)
2. [Deciding whether to raise `history.shardIOConcurrency`](#2-deciding-whether-to-raise-historyshardioconcurrency)
    - [2.1 Step 1 — Check whether writes are queuing inside shards](#21-step-1--check-whether-writes-are-queuing-inside-shards)
    - [2.2 Step 2 — Check whether the database is the slow part](#22-step-2--check-whether-the-database-is-the-slow-part)
    - [2.3 Step 3 — Find the highest value the connection pool can support](#23-step-3--find-the-highest-value-the-connection-pool-can-support)
    - [2.4 Step 4 — Check that the connection pool has room for the next step up](#24-step-4--check-that-the-connection-pool-has-room-for-the-next-step-up)
    - [2.5 Step 5 — Decide whether to raise `history.shardIOConcurrency`](#25-step-5--decide-whether-to-raise-historyshardioconcurrency)
3. [Raising `history.shardIOConcurrency`](#3-raising-historyshardioconcurrency)
    - [3.1 Step 6 — Raise `history.shardIOConcurrency` one step at a time](#31-step-6--raise-historyshardioconcurrency-one-step-at-a-time)
    - [3.2 Step 7 — Confirm the higher value worked](#32-step-7--confirm-the-higher-value-worked)
4. [When to re-check `history.shardIOConcurrency`](#4-when-to-re-check-historyshardioconcurrency)
5. [Alerts for shard IO pressure](#5-alerts-for-shard-io-pressure)
6. [Every setting named in this playbook](#6-every-setting-named-in-this-playbook)

---

## 1. How changes to execution state on one shard wait their turn

*This section provides explanation and recommendations. For the steps that decide whether to raise
`history.shardIOConcurrency`, start at
[section 2.1 — Step 1](#21-step-1--check-whether-writes-are-queuing-inside-shards).*

**What this playbook means by an IO slot.** One IO slot lets one change to execution state be in
flight in the database. Each shard has `history.shardIOConcurrency` of them — 1 by default. A change
takes a free IO slot before its database call and returns it when the call finishes; if none is free,
it waits. The server calls a shard's IO slots its **shard IO semaphore**, the name the dashboard panels
use.

**`history.shardIOConcurrency` is set in dynamic config, but a new value needs a restart of the
history pods.** Many dynamic config settings take effect while the server runs; this one is read
only when a history pod takes ownership of a shard, so a shard picks up a new value after its pod
restarts.

> **Recommendation: after changing `history.shardIOConcurrency`, restart every history pod.** The
> value is read for each shard when a pod takes ownership of it, so restarting only some pods
> leaves the cluster with a mix — shards on the new value and shards on the old one — and nothing
> on the dashboards shows which is which. A rolling restart of all history pods puts every shard on
> the new value.
> [Section 3.1 — Step 6](#31-step-6--raise-historyshardioconcurrency-one-step-at-a-time) covers how to
> do it safely.

**Writes waiting for an IO slot wait in two lanes, and workflow progress always goes first:**

| IO slot lane | Which work waits in it |
|---|---|
| **High priority** | Requests from clients and workers — starting workflows, sending signals, workers reporting results. Operator requests. The server moving workflows forward — dispatching workflow and activity tasks, starting child workflows, telling a parent its child finished, firing timers and timeouts. The shard's own bookkeeping — recording how far it has processed its work, which runs on every shard whether or not the cluster has any workload, and checking that it still owns the shard when shards move. |
| **Low priority** | Clean-up of workflows past their retention, archiving closed workflows, work for namespaces active on another cluster, and applying changes replicated from another cluster. |

The server puts each write in a lane based on its caller type: writes with the preemptable caller
type go in the low-priority lane, and all other writes go in the high-priority lane.

**The low-priority lane gets an IO slot only when the high-priority lane is empty.**

- When an IO slot frees up, it goes to the oldest waiting high-priority write. Only if no
  high-priority write is waiting does it go to the oldest waiting low-priority write.
- A new high-priority write takes a free IO slot straight away, even if low-priority writes are
  already waiting.
- A new low-priority write takes a free IO slot only if nothing is waiting in either lane;
  otherwise it joins the back of the low-priority lane.
- Nothing is interrupted: if a low-priority write already holds the IO slot, a high-priority write
  that arrives waits for that one database call to finish.
- Within each lane there is no further ordering — an operator request, a client request and a
  timer firing all wait in the same first-come, first-served line.

Under steady high-priority load, low-priority writes can wait a long time, until the high-priority
lane drains. Background work waiting behind workflow progress is by design.

**Read operations do not go through the shard IO semaphore.** Loading an execution's state, reading
its history and reading tasks are also done per shard, but they never wait for an IO slot.

**Why the IO slot wait grows.** A write holds its IO slot for as long as the database takes to answer, so the
wait on a shard is roughly the database time of every write queued ahead of it. Two things make the IO slot wait longer:

- **Slower database writes** — every write in the queue takes longer.
- **More writes on the same shard** — more writes in the queue.

**More IO slots help only when the database has room to spare.** If writes are queuing behind each
other on a shard and the database answers quickly, more IO slots let them run side by side. If the
database is the slow part, more writes in flight add to its load and every write slows further. If
one shard receives far more writes than the others — a hot shard — more IO slots can let it write
faster, but they do not spread its load; that is
[Hot Shard — Detection & Remediation](./hot-shard-detection-remediation.md).

**What is now covered is what an IO slot is and why writes wait for one.**
[Section 2](#2-deciding-whether-to-raise-historyshardioconcurrency) uses that to decide whether
raising `history.shardIOConcurrency` will help, starting with which dashboard panels show the IO
slot wait and whether the wait is high enough to act on.

---

## 2. Deciding whether to raise `history.shardIOConcurrency`

**The five steps in this section decide whether raising `history.shardIOConcurrency` will help, and
how far it can go.** Work through them in order. Each step tells you either to continue to the next one
or, when raising `history.shardIOConcurrency` is not the right fix, what to do instead.
[Step 5](#25-step-5--decide-whether-to-raise-historyshardioconcurrency) is the final step: with the
answers from the steps before it, you can decide with confidence whether to raise
`history.shardIOConcurrency`, and how high it can go.

### 2.1 Step 1 — Check whether writes are queuing inside shards

**What Step 1 decides:** whether writes are waiting for IO slots long enough to be worth acting on.

**What to do:**

1. **Check whether alert
   [34j — Shard IO Semaphore Deadlock Approaching](../observability/alerts/server/runbooks/34j-shard-io-semaphore-deadlock-approaching.md)
   or [34f — Shard Deadlock Detected](../observability/alerts/server/runbooks/34f-shard-deadlock-detected.md)
   is firing.** If either is, **stop here** and deal with the alert first —
   [section 5](#5-alerts-for-shard-io-pressure) covers what alerts 34j and 34f mean. Come back to Step 1
   once it has cleared.

2. **On the [Shard IO Concurrency](../observability/dashboards/server/shard-io-concurrency-readme.md) dashboard, open
   [**Shard IO Semaphore Latency (High Priority)**](../observability/dashboards/server/shard-io-concurrency-readme.md#panel-shard-io-semaphore-latency-high-priority).**
   It shows how long high-priority writes — the ones that serve requests and move workflows
   forward — waited for an IO slot, at the percentile chosen in the dashboard's **Percentile**
   selector (p99 by default). Read its current value:

   | Shard IO Semaphore Latency (High Priority) | What to do |
   |---|---|
   | below 20 ms — under the panel's orange line | Writes are not waiting long enough for IO slots to matter, and raising `history.shardIOConcurrency` will not help. **Stop here** — nothing in this playbook needs changing; the panels listed right after this table are other places worth checking. |
   | 20 ms or more — at or above the orange line; above 100 ms it crosses the red line | Writes are queuing inside shards. Next, open Shard IO Semaphore Failures, below. |

   The 20 ms and 100 ms lines are this KB's rule of thumb, not limits the server enforces.

   **If Shard IO Semaphore Latency (High Priority) is below 20 ms,** changes to execution state are
   not waiting for IO slots. `history.shardIOConcurrency` is fine as it is, and raising it will not
   help. If you are looking for other places to check while tuning your cluster, these playbooks are
   good next stops:

   - [History Task Processing — Tuning and Troubleshooting](./history-task-processing-tuning.md) —
     the server's own work falling behind, or being retried because the database refused it.
   - [History Persistence QPS Limits — Rejected Database Calls](./history-persistence-qps-limits.md)
     — database calls turned away by the history service's own limits.
   - [Hot Shard — Detection & Remediation](./hot-shard-detection-remediation.md) — one shard, or
     one busy workflow, taking far more of the load than the rest.
   - [History Host Health](./history-host-health.md) — history pods that are degraded or not
     serving.

3. **On the same dashboard, open
   [**Shard IO Semaphore Failures**](../observability/dashboards/server/shard-io-concurrency-readme.md#panel-shard-io-semaphore-failures).** It counts writes
   that gave up waiting for an IO slot because the request or task behind them hit its deadline —
   for a client request, the deadline the client set on its call; for the server's own work, a fixed
   timeout of a few seconds per task — and the write failed. Use it to judge how soon to finish the
   remaining steps. A reading above zero does not mean raising `history.shardIOConcurrency` is the fix: Steps 2 to 5
   decide that, and the fix may be the database instead.

   | Shard IO Semaphore Failures | What it means |
   |---|---|
   | zero | Writes wait, but all of them get through. Work through the remaining steps at your own pace. |
   | above zero, low | Some writes are failing and being retried. Finish the remaining steps soon. |
   | above zero, steady or rising | Writes are failing all the time. Finish the remaining steps now, and act on whatever they find. |

4. **Rule out a hot shard.** Follow
   [Hot Shard — Detection & Remediation](./hot-shard-detection-remediation.md) to check whether one
   or a few shards are taking far more writes than the rest. If they are, raising
   `history.shardIOConcurrency` will not spread that load: **stop here**, fix the hot shard first,
   then come back and start again at
   [Step 1](#21-step-1--check-whether-writes-are-queuing-inside-shards).

5. **If Shard IO Semaphore Latency (High Priority) is at 20 ms or more and there is no hot shard,
   continue to [Step 2](#22-step-2--check-whether-the-database-is-the-slow-part).**

**What is now covered is that no deadlock alert is firing, that high-priority writes are waiting
20 ms or more for IO slots, how soon the remaining steps need finishing, and that a hot shard is not
the cause.** [Step 2](#22-step-2--check-whether-the-database-is-the-slow-part) reads
**Persistence Write Latency** to tell whether the waiting writes are held up by too few IO slots
on the shard, or by a database that is slow to answer. Raising `history.shardIOConcurrency` fixes
too few IO slots, not a slow database.

### 2.2 Step 2 — Check whether the database is the slow part

**What Step 2 decides:** whether writes wait because a shard has too few IO slots, or because the
database itself is slow to answer.

**What to do:**

1. **On the [Shard IO Concurrency](../observability/dashboards/server/shard-io-concurrency-readme.md) dashboard, open [**Persistence Write Latency**](../observability/dashboards/server/shard-io-concurrency-readme.md#panel-persistence-write-latency).** It shows how long the database takes to answer the
   three writes that matter here — `UpdateWorkflowExecution`, `CreateWorkflowExecution` and
   `AddTasks` — one line each, at the selected percentile. Read the highest of the three lines:

   | Persistence Write Latency | What it means | What to do |
   |---|---|---|
   | low and steady — under about 10 ms | The database answers quickly. Writes are waiting for an IO slot, not for the database. | Continue to [Step 3](#23-step-3--find-the-highest-value-the-connection-pool-can-support). |
   | moderate and steady — about 10 to 50 ms | The database has some room. More IO slots may help, by less. | Continue to [Step 3](#23-step-3--find-the-highest-value-the-connection-pool-can-support), and raise in single steps later. |
   | high or rising — above about 50 ms | The database is the slow part. More writes in flight make it slower. | **Stop here.** Do not raise `history.shardIOConcurrency`. Fix the database first — its capacity, slow queries, or the connection pool. |

   The 10 ms and 50 ms bands are this KB's rule of thumb, not values from the server.

**What is now covered is whether the database has room for more writes at once.**
[Step 3](#23-step-3--find-the-highest-value-the-connection-pool-can-support) turns the other resource every extra write in flight uses — a database connection —
into an upper limit for `history.shardIOConcurrency`.

### 2.3 Step 3 — Find the highest value the connection pool can support

**What Step 3 decides:** the highest value `history.shardIOConcurrency` may ever be set to on your
cluster. Each write in flight holds one connection from its history pod's pool, so the pool size
caps how many writes can be in flight at once.

**What to do:**

1. **Wait until no history pod is restarting.** During a restart, shards move between pods and the
   values on the next panel jump around until the shards have settled.

2. **On the [Shard IO Concurrency](../observability/dashboards/server/shard-io-concurrency-readme.md) dashboard, open
   [**Max Safe shardIOConcurrency Per Pod  (pool_size ÷ shards_per_pod)**](../observability/dashboards/server/shard-io-concurrency-readme.md#panel-max-safe-shardioconcurrency-per-pod--pool_size--shards_per_pod).**
   It has one line per history pod, labelled **ceiling — *pod***: the pod's maximum database
   connections (`maxConns`) divided by the number of shards it owns. For example — made-up numbers, the panel's own arithmetic — a pool
   of 100 connections on a pod that owns 50 shards gives `100 ÷ 50 = 2` — if all 50 shards had 2
   writes in flight, they would hold all 100 connections.

3. **Take the lowest line across all pods.** This playbook calls that value the **ceiling**:

   | Lowest ceiling | What to do |
   |---|---|
   | 0 on every pod | `maxConns` is not set, so the pool has no limit and there is nothing to divide. Use [Step 2](#22-step-2--check-whether-the-database-is-the-slow-part) and your database's own connection limit to judge how far to go instead, and skip the comparison in Step 4. |
   | below 2 | The pool cannot serve even 2 writes in flight per shard. **Stop here.** Raise `maxConns`, or run more history pods so each owns fewer shards, then start this step again. |
   | 2 or more | Note the ceiling. It is the highest value `history.shardIOConcurrency` may go, and [Step 4](#24-step-4--check-that-the-connection-pool-has-room-for-the-next-step-up) uses it. |

**The ceiling from Step 3 is reasoning, not a server rule, and a generous one.** The same pool also serves calls
that take no IO slot, such as reading tasks, so not every connection is free for writes. The
database usually stops getting faster before the pool runs out; [Step 6](#31-step-6--raise-historyshardioconcurrency-one-step-at-a-time) finds where it does.

**What is now covered is the highest value the connection pool can support.** [Step 4](#24-step-4--check-that-the-connection-pool-has-room-for-the-next-step-up) uses
that ceiling to check whether the pool has enough free connections to raise `history.shardIOConcurrency` by 1.

### 2.4 Step 4 — Check that the connection pool has room for the next step up

**What Step 4 decides:** whether each history pod's connection pool has enough free connections
for `history.shardIOConcurrency` to go up by 1. Each +1 can add one connection in use for every
shard the pod owns, if all of those shards write at the same time.

**What to do:**

1. **Look up your limit for pool use, from the ceiling you noted in
   [Step 3](#23-step-3--find-the-highest-value-the-connection-pool-can-support).** Raising
   `history.shardIOConcurrency` by 1 needs one free connection for every shard on the pod. The table gives the highest
   share of the pool that may already be in use and still leave that many free:

   | Ceiling from Step 3 | Highest share of the pool in use allowed, `1 − 1 ÷ ceiling` |
   |---|---|
   | 2 | 50% — half the pool |
   | 3 | 67% |
   | 4 | 75% |
   | 5 | 80% |
   | 10 | 90% |

   For a ceiling not in the table, the limit is `1 − 1 ÷ ceiling`: a ceiling of 8 gives
   `1 − 0.125 = 0.875`, which is 87.5%.

   **Note your limit.** The next item compares your pool reading with it: at or below the limit, you
   continue; above it, you stop.

2. **On the [Shard IO Concurrency](../observability/dashboards/server/shard-io-concurrency-readme.md) dashboard, open
   [**SQL Connection Pool Utilization & Resource Exhausted Errors**](../observability/dashboards/server/shard-io-concurrency-readme.md#panel-sql-connection-pool-utilization--resource-exhausted-errors).**
   It has one **connection pool utilization (main)** line per history pod, labelled with the pod:
   the share of that pod's connections in use, as a percentage of `maxConns`. Find the highest
   line, and use the table below to compare it with the limit you looked up from your Step 3
   ceiling:

   | Highest connection pool utilization (main) | What to do |
   |---|---|
   | at or below your limit | The pool has room for +1. |
   | above your limit | Not enough free connections. **Stop here.** Raise `maxConns` in the SQL section of your persistence config, restart the history pods, and start again at [Step 3](#23-step-3--find-the-highest-value-the-connection-pool-can-support) — a bigger pool changes the ceiling too. |

   **An example — the numbers are made up, the arithmetic is this step's:** your Step 3 ceiling is 5, so your limit is 80%. If the
   highest connection pool utilization (main) line reads 70%, the pool has room. If it reads 90%,
   it does not — stop and grow the pool first.

   The limit assumes every shard on the pod writes at the same time, which is deliberately
   cautious. It is reasoning, not a value from the server.

3. **On the same panel, read the resource exhausted errors/s line.** It counts database calls that
   the history service's own persistence limits turned away. It should be zero. Above zero, the
   limit on database calls is the bottleneck, not the IO slots. **Stop here,** work through
   [History Persistence QPS Limits](./history-persistence-qps-limits.md) to fix it, then come back
   and start again at [Step 1](#21-step-1--check-whether-writes-are-queuing-inside-shards) if writes
   are still waiting for IO slots.

4. **If the pool has room for +1 and no database calls were turned away, continue to
   [Step 5](#25-step-5--decide-whether-to-raise-historyshardioconcurrency).**

**What is now covered is every input to the decision.** [Step 5](#25-step-5--decide-whether-to-raise-historyshardioconcurrency) turns the answers from Steps 1
to 4 into a decision.

### 2.5 Step 5 — Decide whether to raise `history.shardIOConcurrency`

**What Step 5 decides:** whether to raise `history.shardIOConcurrency`. There is no single target
value to pick here: if the answer is yes, you start at 2 and go up by 1 at a time, and the
ceiling from [Step 3](#23-step-3--find-the-highest-value-the-connection-pool-can-support) is the
highest it may go. [Section 3](#3-raising-historyshardioconcurrency) finds the value that actually
helps.

**What to do:**

1. **Find the row that matches your answers from Steps 1 to 4.** The rows assume the other checks
   in those steps passed — no firing alert, no hot shard, no database calls turned away; any of
   those stops you earlier.

   | Step 1: Shard IO Semaphore Latency (High Priority) | Step 2: Persistence Write Latency | Step 3: ceiling | Step 4: room in the pool | Decision |
   |---|---|---|---|---|
   | 20 ms or more | under about 50 ms | 2 or more | yes | **Raise `history.shardIOConcurrency`** by 1 at a time, as the recommendation below the table describes. Continue to [section 3](#3-raising-historyshardioconcurrency). |
   | 20 ms or more | under about 50 ms | 0 — `maxConns` not set | not checked | **Raise `history.shardIOConcurrency`, with care.** The connection pool has no limit, so judge how far to go from Step 2 and your database's own connection limit. Continue to [section 3](#3-raising-historyshardioconcurrency). |
   | 20 ms or more | under about 50 ms | below 2 | not checked | **Do not raise `history.shardIOConcurrency` yet.** Raise `maxConns` — the size of each history pod's database connection pool — or run more history pods so each owns fewer shards. Then start again at [Step 1](#21-step-1--check-whether-writes-are-queuing-inside-shards). |
   | 20 ms or more | under about 50 ms | 2 or more | no | **Do not raise `history.shardIOConcurrency` yet.** Raise `maxConns` — the size of each history pod's database connection pool. Then start again at [Step 1](#21-step-1--check-whether-writes-are-queuing-inside-shards). |
   | 20 ms or more | above about 50 ms | any | any | **Do not raise `history.shardIOConcurrency`.** Fix the database first, then start again at [Step 1](#21-step-1--check-whether-writes-are-queuing-inside-shards). |

> **Recommendation: raise `history.shardIOConcurrency` by 1 at a time.** Go from the default of 1 to
> 2 first. Once every history pod is running the new value, go through Steps 1 to 5 again from
> [Step 1](#21-step-1--check-whether-writes-are-queuing-inside-shards). If Step 5 still says to raise `history.shardIOConcurrency`, add 1 more — 3, then 4
> — and repeat. Stop when Steps 1 to 5 say not to raise the value again.

**What is now covered is whether to raise `history.shardIOConcurrency`.** If Step 5 told you to
raise `history.shardIOConcurrency` by 1, [section 3](#3-raising-historyshardioconcurrency) shows how
to roll the new value out safely and confirm that it worked. If Step 5 told you not to raise
`history.shardIOConcurrency`, its decision table says what to do instead.

---

## 3. Raising `history.shardIOConcurrency`

**Only if [Step 5](#25-step-5--decide-whether-to-raise-historyshardioconcurrency) decided to raise
`history.shardIOConcurrency`.** [Step 6](#31-step-6--raise-historyshardioconcurrency-one-step-at-a-time) rolls the new value
out safely, one history pod at a time, and
[Step 7](#32-step-7--confirm-the-higher-value-worked) confirms it did what it was meant to.

### 3.1 Step 6 — Raise `history.shardIOConcurrency` one step at a time

**What Step 6 decides:** nothing new — Step 5 has decided. Step 6 puts the new value on every
history pod safely.

**Before you start:**

- **The change needs a rolling restart of the history pods.** The value is read when a pod takes
  ownership of a shard, so pods keep the old value until they restart. Frontend, matching and
  worker pods are not affected.
- **Pick a quiet period if you can.** While a history pod restarts, its shards move to other pods
  and requests on those shards are slower until they settle. If [**Shard IO Semaphore Failures**](../observability/dashboards/server/shard-io-concurrency-readme.md#panel-shard-io-semaphore-failures) is steadily above zero,
  writes are already failing, and waiting for a quiet period costs more than the restart.
- **Change the value only when you are ready to restart every history pod straight away.** Once the
  new value is in dynamic config, any history pod that restarts — planned or not — hands its shards
  to other pods, and those shards pick up the new value while the rest keep the old one. So do not
  set the value and wait for a quiet period: choose the period first, then change the value and
  start the rolling restart. If a history pod restarts unexpectedly after you have changed the value
  but before you have restarted every pod, carry on with the rolling restart — or roll back — then,
  so every shard ends up on the same value.

**What to do:**

1. **Note the current readings on the [Shard IO Concurrency](../observability/dashboards/server/shard-io-concurrency-readme.md) dashboard** — [**Shard IO Semaphore Latency (High Priority)**](../observability/dashboards/server/shard-io-concurrency-readme.md#panel-shard-io-semaphore-latency-high-priority), [**Shard IO Semaphore Failures**](../observability/dashboards/server/shard-io-concurrency-readme.md#panel-shard-io-semaphore-failures), [**Persistence Write Latency**](../observability/dashboards/server/shard-io-concurrency-readme.md#panel-persistence-write-latency), and
   [**SQL Connection Pool Utilization & Resource Exhausted Errors**](../observability/dashboards/server/shard-io-concurrency-readme.md#panel-sql-connection-pool-utilization--resource-exhausted-errors). [Step 7](#32-step-7--confirm-the-higher-value-worked) compares the panels after the
   change with these readings.

2. **Set the new value in your dynamic config** — your current value plus 1. The first time, that
   is `2`:

   ```yaml
   history.shardIOConcurrency:
     - value: 2
   ```

3. **Restart every history pod, one at a time.** Restarting one at a time keeps most shards serving
   while each pod's shards move. Before restarting the next pod, wait until the shards have settled,
   using the shard movement panels on the [Temporal Server](../observability/dashboards/server/temporal-server-readme.md) dashboard:

   | Panel | Settled when |
   |---|---|
   | [**Owned Shards (Total)**](../observability/dashboards/server/temporal-server-readme.md#8-shard-movement) | it is back to your cluster's configured number of shards |
   | [**Shards Created**](../observability/dashboards/server/temporal-server-readme.md#8-shard-movement), [**Shards Removed**](../observability/dashboards/server/temporal-server-readme.md#8-shard-movement) and [**Shards Closed**](../observability/dashboards/server/temporal-server-readme.md#8-shard-movement) | the spikes from the restart have dropped back to near zero |

4. **Go on to [Step 7](#32-step-7--confirm-the-higher-value-worked) once every history pod has restarted.** The dashboard panels combine
   all history pods, so they only show the effect of the new value once every pod is running it.

**What is now covered is putting the new value on every history pod.** [Step 7](#32-step-7--confirm-the-higher-value-worked) checks
whether it helped.

### 3.2 Step 7 — Confirm the higher value worked

**What Step 7 decides:** whether the new value of `history.shardIOConcurrency` fixed the wait,
once every history pod is running it.

**What to do:**

1. **Wait until every history pod has restarted and the shards have settled.**

2. **On the [Shard IO Concurrency](../observability/dashboards/server/shard-io-concurrency-readme.md) dashboard, compare each panel with the readings you noted before the change**
   in [Step 6](#31-step-6--raise-historyshardioconcurrency-one-step-at-a-time):

   | Panel | The change worked | Roll back if |
   |---|---|---|
   | [**Shard IO Semaphore Latency (High Priority)**](../observability/dashboards/server/shard-io-concurrency-readme.md#panel-shard-io-semaphore-latency-high-priority) | lower than before | not lower than before |
   | [**Shard IO Semaphore Failures**](../observability/dashboards/server/shard-io-concurrency-readme.md#panel-shard-io-semaphore-failures) | lower than before, or zero | higher than before |
   | [**Persistence Write Latency**](../observability/dashboards/server/shard-io-concurrency-readme.md#panel-persistence-write-latency) | not clearly higher than before, and still under about 50 ms | clearly higher than before, or above about 50 ms |
   | [**SQL Connection Pool Utilization & Resource Exhausted Errors**](../observability/dashboards/server/shard-io-concurrency-readme.md#panel-sql-connection-pool-utilization--resource-exhausted-errors) | no connection pool utilization (main) line close to 100%; resource exhausted errors/s zero | any line close to 100%, or resource exhausted errors/s above zero |
   | [**Shard IO Deadlock Check Latency**](../observability/dashboards/server/shard-io-concurrency-readme.md#panel-shard-io-deadlock-check-latency) | near zero | rising |

3. **If any panel says to roll back, roll back:** set `history.shardIOConcurrency` back to its previous
   value and restart every history pod again, one at a time, as in [Step 6](#31-step-6--raise-historyshardioconcurrency-one-step-at-a-time). Then start again
   at [Step 1](#21-step-1--check-whether-writes-are-queuing-inside-shards) to find what else is slowing writes down.

4. **If every panel says the change worked,** go through Steps 1 to 5 again, as the recommendation in
   [Step 5](#25-step-5--decide-whether-to-raise-historyshardioconcurrency) describes. If Step 5 still says to raise `history.shardIOConcurrency`, add 1 more
   and repeat Steps 6 and 7 — never going above the ceiling from [Step 3](#23-step-3--find-the-highest-value-the-connection-pool-can-support).

**What is now covered is rolling out and confirming a higher `history.shardIOConcurrency`.** A value
that is right today can stop being right when the cluster changes —
[section 4](#4-when-to-re-check-historyshardioconcurrency) lists the changes that call for a re-check.

---

## 4. When to re-check `history.shardIOConcurrency`

**Each history pod has its own connection pool of `maxConns` connections, and that pool does not
grow when the pod takes on more shards.** The ceiling from [Step 3](#23-step-3--find-the-highest-value-the-connection-pool-can-support) is `maxConns` divided by
the shards a pod owns, so two infrastructure changes lower the ceiling:

| Change | Why it lowers the ceiling |
|---|---|
| **Fewer history pods** | Each remaining pod owns more shards, so the same pool is shared by more of them. |
| **A smaller connection pool** | Lowering `maxConns` lowers the ceiling by the same proportion. |

**An example — the cluster's numbers are made up, the arithmetic is the one Step 3 uses:** a
cluster of 2,048 shards on 16 history pods, with `maxConns: 512` — so each history pod has its own
pool of 512 connections. Each pod owns 128 shards, so the ceiling is `512 ÷ 128 = 4`. Scale down to
8 pods and each owns 256 shards: the ceiling drops to `512 ÷ 256 = 2`. A cluster that had raised
`history.shardIOConcurrency` to 4 now has pods that may need up to `256 × 4 = 1,024` connections
from a pool of 512 — writes would hold an IO slot and then wait for a connection.

Scaling up is the safe direction: more pods means fewer shards per pod, so the ceiling rises.

> **Recommendation: if your cluster scales its history pods up and down often, leave
> `history.shardIOConcurrency` at 1.** Every scale-down raises the number of shards each pod owns,
> and a value that was safe at one size can be too high at the next. If you do raise `history.shardIOConcurrency` on such a
> cluster, size `history.shardIOConcurrency` for the smallest number of history pods the cluster ever runs: work out the
> ceiling in [Step 3](#23-step-3--find-the-highest-value-the-connection-pool-can-support) as if the cluster were at that size, and never raise the value above it.

**Before a planned scale-down or a smaller pool,** re-run [Step 3](#23-step-3--find-the-highest-value-the-connection-pool-can-support) and [Step 4](#24-step-4--check-that-the-connection-pool-has-room-for-the-next-step-up) for the
new size. If the new ceiling is below your current `history.shardIOConcurrency`, lower the value and
do the rolling restart in [Step 6](#31-step-6--raise-historyshardioconcurrency-one-step-at-a-time) **before** making the infrastructure change, not after.

**What is now covered is keeping `history.shardIOConcurrency` right as the cluster changes.**
[Section 5](#5-alerts-for-shard-io-pressure) covers alerts 34j and 34f, which fire when IO slots stop
freeing up altogether.

---

## 5. Alerts for shard IO pressure

**Alerts 34j and 34f both come from the server's deadlock check.** Once a minute, the deadlock check
takes and immediately returns an IO slot on every shard, and records how long it waited on
[**Shard IO Deadlock Check Latency**](../observability/dashboards/server/shard-io-concurrency-readme.md#panel-shard-io-deadlock-check-latency).
After 40 seconds without an IO slot — 10 seconds for a database call plus 30 seconds of grace — it
declares a suspected deadlock. Healthy shards read close to zero on that panel.

| Alert | Fires when | What to do |
|---|---|---|
| [34j — Shard IO Semaphore Deadlock Approaching](../observability/alerts/server/runbooks/34j-shard-io-semaphore-deadlock-approaching.md) | [**Shard IO Deadlock Check Latency**](../observability/dashboards/server/shard-io-concurrency-readme.md#panel-shard-io-deadlock-check-latency) above 20 s for 5 minutes — half the 40-second limit | Find out why IO slots are held so long: run [Step 1](#21-step-1--check-whether-writes-are-queuing-inside-shards) and [Step 2](#22-step-2--check-whether-the-database-is-the-slow-part). Do not raise the IO slot count while the database is slow. |
| [34f — Shard Deadlock Detected](../observability/alerts/server/runbooks/34f-shard-deadlock-detected.md) | [**Suspected Deadlocks (current) per Pod**](../observability/dashboards/server/temporal-server-readme.md#9-shard-queue-health) above zero | Follow the 34f runbook. A suspected deadlock is an incident, not a tuning question. |

**What neither alert tells you** is whether raising the IO slot count would help. A reading of 20 ms or more
on Shard IO Semaphore Latency (High Priority) is far below the 20 seconds alert 34j fires at, and can
still be worth fixing — that is what
[Steps 1 to 5](#21-step-1--check-whether-writes-are-queuing-inside-shards) decide.

**What is now covered is the alerting.** [Section 6](#6-every-setting-named-in-this-playbook)
lists every setting this playbook names, and which of them to change.

---

## 6. Every setting named in this playbook

| Setting | Default | Verdict |
|---|---|---|
| `history.shardIOConcurrency` | `1` | **Tune it**, on PostgreSQL and MySQL only, by working through this playbook. Takes effect when a history pod restarts. On Cassandra the server forces it to `1`. `0` does not mean unlimited: a shard would have no IO slots and no write could proceed. |
| `history.shardIOTimeout` | `5s` | **Read it, leave it.** Bounds the shard's own database calls — loading its metadata, writing its progress and renewing its ownership. It does not limit the IO slot wait. |
| `system.deadlock.Interval` | `1m` | **Leave it.** How often the deadlock check runs, and so how often **Shard IO Deadlock Check Latency** gets a new value. |
| `maxConns` (SQL section of the persistence config file) | not set — no limit | **Belongs to your persistence config**, not dynamic config. Sets each history pod's connection pool size, which [Step 3](#23-step-3--find-the-highest-value-the-connection-pool-can-support) divides by shards per pod. Left unset, the pool has no limit and **Max Safe shardIOConcurrency Per Pod  (pool_size ÷ shards_per_pod)** reads 0. |
