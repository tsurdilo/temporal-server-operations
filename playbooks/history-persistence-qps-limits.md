# History Persistence QPS Limits — Rejected Database Calls Playbook

**Audience**

Operators of self-hosted Temporal clusters.

**What this playbook covers**

The history service limits how fast it queries the database. This playbook covers what those limits
are, how they interact, and what to do when they start turning calls away with `RESOURCE_EXHAUSTED`
and cause `PersistenceLimit`.

That cause always means Temporal's own limiter turned the call away before the database was asked.
No database produces it, on any backend.

**What this playbook does not cover**

- **Database tuning.** If the answer turns out to be that the database needs more capacity or a
  better configuration, that work is outside this playbook.
- **Any other `RESOURCE_EXHAUSTED` cause.** `SystemOverloaded`, `PersistenceStorageLimit`,
  `BusyWorkflow`, `RpsLimit` and the rest all mean something other than this limiter, and each needs
  a different fix. Section 3 covers how to tell them apart.
- **Visibility persistence limits.** A history pod's visibility reads and writes go through a
  separate limiter with its own settings — `system.visibilityPersistenceMaxReadQPS` and
  `system.visibilityPersistenceMaxWriteQPS`, both 9000 — and its own metric. It returns the same
  error as the limiter covered here, so check *which operation* was rejected: raising
  `history.persistenceMaxQPS` does nothing for a throttled visibility call.
- **The other services' persistence limits.** Frontend, matching and worker have their own
  `persistenceMaxQPS` settings and companions. This playbook is about the history service only.

**Applies to**

The history service, on any persistence store — SQL or Cassandra.

**The limiter is store-independent.** It decides before the call reaches the store, so the settings,
the defaults and everything in this playbook are the same whichever engine you run.

**Dashboard**

This playbook is written against the
[Temporal Server dashboard](../observability/dashboards/server/temporal-server-readme.md#groups-and-panels),
and refers to panels by name. Every panel it uses, with the dashboard group it sits in:

| Dashboard group | Panels this playbook uses |
|---|---|
| [3. Persistence Requests, Latencies and Errors](../observability/dashboards/server/temporal-server-readme.md#3-persistence-requests-latencies-and-errors) | **History Rejected Database Calls Total by Scope**<br>**Rejected Database Calls by Operation and Scope**<br>**Persistence Latencies**<br>**Persistence Errors by Namespace and Operation**<br>**Persistence Availability**<br>**Write-Reject Loop Indicator (cleared / cache miss)**<br>**Per-Shard Persistence RPS Distribution (Hot-Shard Detector)**<br>**Hottest Shard RPS**<br>**Adaptive Rate Limit Multiplier** |
| [6. Throttling and Limits](../observability/dashboards/server/temporal-server-readme.md#6-throttling-and-limits) | **Resource Exhausted with Cause** |
| [7. Busy Workflow Throttling](../observability/dashboards/server/temporal-server-readme.md#7-busy-workflow-throttling) | **Transfer Active Task Errors Throttled** |
| [8. Shard Movement](../observability/dashboards/server/temporal-server-readme.md#8-shard-movement) | **Owned Shards (Total)** |
| [9. Shard Queue Health](../observability/dashboards/server/temporal-server-readme.md#9-shard-queue-health) | **Immediate Queue Lag per Pod**<br>**Scheduled Queue Lag per Pod** |
| [10. History Timer Task Info](../observability/dashboards/server/temporal-server-readme.md#10-history-timer-task-info) | **Total Timer Tasks Errors**<br>**Timer Task Scheduling Latency** |

**Alert**

**[86 — History Database Calls Rejected](../observability/alerts/server/runbooks/86-history-database-calls-rejected.md)**
runs the same query as the first panel above, **History Rejected Database Calls Total by Scope** —
so the panel and the alert always agree — and fires when rejections from the history service exceed
**10/s for 10 minutes**. It groups by scope and cause rather than filtering on either, so each
cause alerts separately. Defined in
[temporal-server-alerts.yaml](../observability/alerts/server/temporal-server-alerts.yaml).

---

## Contents

1. [Why the history service limits its own database calls](#1-why-the-history-service-limits-its-own-database-calls)
2. [The settings, and how they interact](#2-the-settings-and-how-they-interact)
3. [How the limiter behaves in practice](#3-how-the-limiter-behaves-in-practice)
4. [Seeing what is going on](#4-seeing-what-is-going-on) — ending in [the decision](#46-the-decision--what-to-actually-do)
5. [Alerting on rejected database calls](#5-alerting-on-rejected-database-calls)
6. [Worked examples](#6-worked-examples)
7. [Reference — every setting this playbook mentions](#7-reference--every-setting-this-playbook-mentions)

---

## 1. Why the history service limits its own database calls

The history service does most of the database work in a Temporal cluster. It is the service that
keeps workflow executions durable, and durability means writing: every state change a workflow makes
is a read and then a write to the persistence store. Each history pod also carries background work
for every shard it owns — reading tasks off queues, recording queue progress, renewing shard
ownership — and that work is driven by how many shards the pod owns, not by how busy your workflows
are.

None of that work paces itself. A pod issues as many database calls as it has work for.

The store cannot absorb any amount of that. Every database has a ceiling: how many connections it
will hold open, and how much disk and CPU it has to serve reads and writes. Temporal's own
connection pool to it is bounded too (`maxConns` in the server's persistence config). And all four
services share that one store, for every namespace on the cluster.

**An overloaded database does not perform well.** Latency climbs, and the reads and writes a
workflow needs in order to move forward take longer and longer. Keeping the store out of that state
is the whole point of the QPS limits.

Temporal gives you those limits for exactly that purpose: to protect the persistence store from
being overloaded by its own cluster. They can be set at several levels — one pod, one namespace, one
shard, or the whole cluster — and
[section 2](#2-the-settings-and-how-they-interact) covers each of them and how they interact.

### 1.1 What happens when a QPS limit is reached

Once the QPS limits are in place, they can be reached — and it does not take an extreme
workload to get there. Ordinary growth in traffic, under-provisioned SDK workers, or the shape of
the use case itself all put more pressure on the database. When that pressure reaches a configured
limit, Temporal protects the database by throttling rather than passing the work on.

What throttling does:

- The call is **rejected before it reaches the database**, and the rejection goes straight back to
  the caller. There is no retry at this layer.
- For a history task, the task is **rescheduled** — first after 3 seconds, then 1.5x longer each
  attempt, up to a ceiling of 5 minutes, with **no limit on the number of attempts** and no overall
  time limit either. **These values are fixed in the server, not dynamic config** — there is no
  setting that changes them.
- Throttling counts as an **expected** error. It is never recorded as a task failure, and a task can
  never be sent to the dead-letter queue for being throttled, however long it goes on.
- **Queue lag grows.** A history task is a row in the database, written when the workflow's
  transaction committed. The queue deletes those rows only up to its lowest still-pending task, so a
  task waiting to retry holds that point back — and the gap it leaves is the lag you see on the
  dashboard.

**Nothing is lost if the pod restarts.** The retry timer lives in memory and does not survive a
restart. The task does, because it is still an undeleted row in the database, and the point the
queue has deleted up to is itself saved to the store. Whichever pod takes over the shard reads that
saved point, finds the task rows still sitting there, and runs them again. Retrying forever is safe
precisely because the database, not the pod's memory, is holding the work.

**Throttling and a slow database are not the same failure.** Both look like tasks retrying, but they
end very differently:

| What the task ran into | Counts toward the dead-letter threshold? | What happens to it |
|---|---|---|
| **throttling** — a limit turned the call away before it was sent | **no** | retried indefinitely. It can never be dead-lettered for this, however long it lasts. |
| **a timeout** — the call was sent and the database was too slow to answer | **yes** | after `history.TaskDLQUnexpectedErrorAttempts` such attempts (default **70**, roughly an hour) the task is moved to the dead-letter queue, stops retrying, and waits for an operator |

**Why the difference matters.** Throttling recovers on its own — the work waits, retries, and
completes with nobody doing anything. A dead-lettered task does not. It is taken out of the queue
and is never retried automatically, and for the task types that carry execution progress (activity
retry and timeout timers, activity and workflow task dispatch) that leaves **workflows stuck** until
an operator redrives the queue by hand with `tdbg dlq merge`. Other dead-lettered types, such as
visibility or retention work, do not strand a running execution.

So that is the trade the limit buys you: a delay that resolves itself, instead of work that needs a
person to recover it.

### 1.2 Set them deliberately — do not rely on the defaults

Every one of these limits has a default, so a cluster nobody has touched is already rate limiting
itself. That is not the same as being configured for your workload. A default can be wrong in three
different ways:

- **Too high** for a small or shared database, in which case it is not really protecting anything.
- **Too low** for a busy cluster, in which case ordinary load gets throttled for no good reason.
- **One number for everything.** Out of the box there is effectively one number per pod. Nothing caps a
  single noisy namespace, and nothing caps the cluster as a whole.

**This playbook recommends setting all three deliberately** rather than leaving any of them where
they landed:

| Limit | What it does for you |
|---|---|
| `history.persistenceGlobalMaxQPS` | Sets the cluster's total database budget and divides it among the pods. This is the number to match against what your database can actually take — and the one to prefer if your history fleet changes size, because the total stays put as pods come and go. See [check 1](#check-1--is-a-cluster-wide-limit-in-force). |
| `history.persistenceMaxQPS` | The per-pod budget. **When a cluster-wide limit is set the limiter ignores this** — but it still decides how fast queue readers poll the database for tasks, so set it to roughly [the effective per-pod rate](#25-a-global-limit-replaces-the-per-pod-one) rather than leaving it at its default. |
| `history.persistenceNamespaceMaxQPS` | Stops one namespace using up a pod's entire budget. Only does anything if it is set **lower** than the effective per-pod rate — setting it equal to the pod limit is [the most common mistake here](#24-do-not-set-the-per-namespace-limit-equal-to-the-per-pod-limit). |

These do not simply stack, and the interactions have real traps — a cluster-wide limit *replaces*
the per-pod one rather than capping it, a `0` usually does not mean "off", and a namespace limit set
to the same value as the pod limit achieves nothing.
[Section 2](#2-the-settings-and-how-they-interact) works through each of them, and through the two
finer-grained limits as well.

**One thing to be clear about: these limits throttle database calls, not task processing.** A
throttled history task is one whose database call was turned away part-way through; nothing held the
task back before it started. Pacing task processing is a different mechanism — the history task
scheduler — and it is off by default. See
[a note on task processing](#34-a-note-on-task-processing).

---

## 2. The settings, and how they interact

These are the settings behind a `PersistenceLimit` rejection. Every one of them changes one of three
things: **whether** a database call gets rejected at all, **which** of the three checks rejects it,
or **what the rejection looks like** in your metrics and logs. That is why this comes before any
diagnosis — without it, the error tells you almost nothing.

All of these are dynamic config. None of them needs a restart.

### 2.1 The five settings

| Setting | Scope | Default |
|---|---|---|
| `history.persistenceMaxQPS` | one history pod | **9000** |
| `history.persistenceNamespaceMaxQPS` | one namespace, on one history pod | 0 |
| `history.persistencePerShardNamespaceMaxQPS` | one namespace, on one shard | 0 |
| `history.persistenceGlobalMaxQPS` | whole cluster | 0 |
| `history.persistenceGlobalNamespaceMaxQPS` | one namespace, whole cluster | 0 |

There are five settings, but a database call is only ever checked against **three** limits. The two
cluster-wide settings are not extra checks — each one simply supplies the number for a check that
already exists:

| The check | Takes its number from |
|---|---|
| one namespace, on one shard | `history.persistencePerShardNamespaceMaxQPS` |
| one namespace, on this pod | `history.persistenceGlobalNamespaceMaxQPS` when that is set, otherwise `history.persistenceNamespaceMaxQPS` |
| this pod, everything it does | `history.persistenceGlobalMaxQPS` when that is set, otherwise `history.persistenceMaxQPS` |

So setting a cluster-wide limit does not add a check. It changes where an existing check gets its
number — covered in
[a global limit replaces the per-pod one](#25-a-global-limit-replaces-the-per-pod-one).

### 2.2 Three checks, in order

Every database call from a history pod goes through the three checks in this order. **The first one
to say no wins**, and the remaining checks are not consulted:

1. one namespace, on one shard
2. one namespace, on this pod
3. this pod, everything it does

The error you get back tells you which check rejected the call:

| Check that rejected it | `resource_exhausted_scope` tag | Log message |
|---|---|---|
| one namespace, on one shard | `Namespace` | `Namespace Per-Shard Persistence Max QPS Reached.` |
| one namespace, on this pod | `Namespace` | `Namespace Persistence Max QPS Reached.` |
| this pod, everything it does | `System` | `System Persistence Max QPS Reached.` |

The two namespace checks **carry the same scope tag**, so that tag on its own cannot tell you which
of the two rejected the call. The log message can, because each check has its own wording. Count
them in the history pod's log:

```bash
grep -hoE "(System|Namespace|Namespace Per-Shard) Persistence Max QPS Reached\." <logs> | sort | uniq -c
```

**One call carries one scope tag — but a graph shows you many calls.** Each rejected call is
rejected by exactly one check, so it carries exactly one scope tag. A panel or a query adds up
thousands of calls, and different calls run into different checks, so **expect to see `System` and
`Namespace` at the same time**. Measured on the test cluster, throttling showed up as a mix of the
two at the same moment. That is normal and does not mean anything is misconfigured.

### 2.3 Which check rejects the call, and which scope you see

The three checks are **separate token buckets, evaluated one after another, and evaluation stops at
the first rejection** — the later checks never run for a call that has already been turned away. So
if a call was over two limits at the same moment, only the first one in the order ever reports it.
(A check the call already passed has still spent a token from its own bucket, even though the call
was rejected further down.)

That makes "which check rejected it" a real question, and the order does not answer it. What decides
it is **how much traffic each check sees**, because they do not count the same calls.

This is also what makes the `resource_exhausted_scope` tag meaningful: the check that rejected the
call is the one the tag names.

| Check | Scope tag you see | Counts | So you run into this one when |
|---|---|---|---|
| per-shard, per-namespace | `Namespace` | one namespace's calls, on one shard | a single shard does a lot of work for that namespace |
| per-namespace | `Namespace` | one namespace's calls, across the whole pod | one namespace dominates that pod |
| **per-pod** | `System` | **everything the pod does** — every namespace, plus the pod's own internal work | total load on the pod is high. **The common case.** |

Note that two of the three report `Namespace`, which is why the tag on its own cannot tell you which
of them rejected the call — only the [log message](#22-three-checks-in-order) can.

**Not all traffic sees all three checks.** Some of a pod's database work is tagged with a namespace —
running workflow tasks, timers, activities, anything driven by an API call. The rest is tagged as
system work: loading tasks off queues, queue checkpointing, shard ownership updates. **System work
skips the per-namespace and per-shard checks entirely** and only meets the per-pod one.

So the per-pod bucket counts everything the other two count, and more besides.

### 2.4 Do not set the per-namespace limit equal to the per-pod limit

**This is the most common mistake made with these settings.** Setting
`history.persistenceNamespaceMaxQPS` to the same value as `history.persistenceMaxQPS` looks like it
caps a namespace at the pod's capacity. What it actually does is create a second ceiling at the same
height — one you then have to raise twice, every time.

**Why.** At `0` the namespace check does not switch off — it runs at **the pod's rate**. So setting
it explicitly to the same number as the pod limit behaves *exactly the same as leaving it at `0`* —
right up until the day you change the pod limit. At `0` the namespace check follows the new pod
number on its own. At an explicit equal value it stays where it was and becomes your ceiling. **That
is the entire trap: it costs nothing today and silently stops tracking tomorrow.**

The two are separate buckets refilling at the same rate, and the namespace check runs **first**. So
as soon as one namespace's traffic on its own goes above the rate, the namespace check is what
rejects the call, and the per-pod check is never reached.

**Measured** on the test cluster with both set to the same value:

| Scope that rejected | Rate |
|---|---|
| `Namespace` | **242.9/s** |
| `System` | 200.9/s |

The namespace limit rejected *more* than the pod limit did.

**What this looks like when it bites you.** You see throttling. You raise
`history.persistenceMaxQPS`. Nothing improves — because the namespace limit is still at the old
value and is now the one doing the rejecting. Raise one and the other becomes the ceiling.

**How much this costs you depends on how many namespaces you run.** *Reasoned from how the checks
are wired, not measured.*

- **A single namespace.** You are not losing any protection. There is nothing to isolate that
  namespace from, and the pod limit caps the pod's total load either way. The only cost is the
  double-raise — you change one number and nothing improves.
- **More than one namespace.** You lose the isolation too. A per-namespace limit exists to hold one
  namespace below the others, and set at the pod's value it holds nobody below anybody. So you have
  taken on the double-raise cost **and** got none of the benefit you set it for.

**What to do instead.** One of these two:

- **Leave `history.persistenceNamespaceMaxQPS` at `0`.** It then follows the pod limit on its own,
  and you have a single number to tune. This is the right choice unless you specifically want one
  namespace held below the rest. See [what each setting does at `0`](#26-what-each-setting-does-when-set-to-0).
- **Set it genuinely lower than the effective per-pod rate.** That is the only way it caps a noisy
  namespace instead of duplicating a ceiling you already have.

### 2.5 A global limit replaces the per-pod one

**Check this before you tune anything.**

If `history.persistenceGlobalMaxQPS` is above `0`, then `history.persistenceMaxQPS` is **ignored by
the persistence limiter**. Not added to. Not capped by. Ignored.

For the history service, each pod's rate becomes its share of the global number, **weighted by how
many shards that pod owns**:

```
effective per-pod rate  =  history.persistenceGlobalMaxQPS  x  ( shards this pod owns / numHistoryShards )
```

**Measured**, reading the rate straight off a pod: a cluster-wide limit of 36000 across 2048 shards,
with the pod owning 1156 of them, enforced **20320.3125** — which is `36000 x 1156/2048` to the last
decimal place.

The same applies to the namespace pair: if `history.persistenceGlobalNamespaceMaxQPS` is above `0`,
`history.persistenceNamespaceMaxQPS` is ignored.

**The two pairs are independent, though.** A cluster-wide *host* limit does not change the
namespace check at all — `history.persistenceGlobalMaxQPS` replaces `history.persistenceMaxQPS`
and nothing else. A namespace limit you have set stays in force, is still checked **first**, and
still rejects, however high the cluster-wide number is. **Measured**: with a cluster-wide limit of
36000 in force and `history.persistenceNamespaceMaxQPS` at 5, every rejection came back at scope
`Namespace` and none at `System` — see [6.1](#61-rejections-on-child-workflow-completions-with-both-limits-set-to-the-same-number).

**This is shard-weighted only for history.** Frontend, matching and worker divide a cluster-wide
limit flatly by the number of pods in that service. History is the exception, so that pods carrying
more shards get more budget.

Three things follow:

- **It is not a flat division by pod count.** Dividing the global number by your pod count gives you
  the *average*, which is fine for sizing. But shard ownership is never perfectly even, so real pods
  land either side of it and the busiest ones start throttling before the average suggests.
  **Measured** on a two-pod cluster with 2048 shards: the pods owned 1066 and 982 shards, so a
  global limit of 8000 gave them **4164/s and 3836/s** — not 4000 each.
- **The effective rate shifts when pods come and go.** Lose a pod and the survivors each get a
  larger share; add pods and each gets less. A per-pod limit never behaves that way.
- **For about the first minute after a pod starts, `history.persistenceMaxQPS` is the number in
  force** — the cluster-wide limit is not applied yet. The reason is not that shard ownership takes
  a minute to settle; it is that **the limiter only re-reads its quota once a minute**, and at the
  very first read the pod's view of cluster membership has not loaded, so it falls back to the
  per-pod setting.

  **Measured** on a pod set to `persistenceMaxQPS: 16000` under a cluster-wide limit of 36000: it
  enforced **16000** on startup and switched to **20320.3125** exactly sixty seconds later. So for
  that first minute it was running at a limit the cluster-wide setting was supposed to have
  replaced.

  This matters when you restart a pod or roll a deploy and then look at throttling: for the first
  minute you are reading the wrong limit. Read the real number off the pod rather than assuming —
  see below.

**Reading the rate a pod is really enforcing.** There is no metric for it. The one place it is
exposed is the pod's own log, which prints the value whenever it changes:

```bash
grep "Quota changed" <history-pod-log>
```

Each line carries a component and a scope, so look for the one tagged persistence and host. This
needs the pod's log level at `info`; at `warn` the line is never emitted. **Do this whenever the
configured number and the observed behaviour disagree** — it is the only way to confirm what a pod
is enforcing rather than what you believe you configured.

It is also how you read a restart. The first line after a pod starts shows the per-pod value, and a
second line a minute later shows the cluster-wide share taking over, so the sequence tells you which
limit was in force when.

> ### ⚠ A global limit does not switch the per-pod setting off
>
> `history.persistenceGlobalMaxQPS` replaces `history.persistenceMaxQPS` **for the persistence
> limiter only**. The per-pod value is still read, and still matters, for the rate at which history
> queue processors poll the database for tasks. A global limit does not apply there at all.
>
> | Queue reader | Poll ceiling |
> |---|---|
> | Transfer, timer, outbound | `history.persistenceMaxQPS` x 0.30, **each** |
> | Visibility, archival | `history.persistenceMaxQPS` x 0.15, **each** |
>
> Each queue has its own `history.<queue>ProcessorMaxPollHostRPS` setting; the ratios above are the
> fallback used when that is left at `0`, which is the default for all five. They exist so that
> loading tasks cannot consume the pod's entire database budget and starve everything else — which
> matters most just after a restart, when a pod has to load for every shard it owns at once.
>
> **These are not shares of one budget.** Each queue gets its own separate ceiling, and the five add
> up to **1.2x** the per-pod number rather than to 100% of it. They cap how fast each queue *may*
> poll; what it actually spends depends on how many tasks it needs. Queue loading is also system
> work, so it never passes through the per-namespace check — only the per-pod one.
>
> **So set `history.persistenceMaxQPS` to roughly the effective per-pod rate, even though the
> limiter ignores it.** Otherwise these ceilings are sized against a budget the pod does not have.
>
> *Reasoned from how the rates are wired, not measured.* Say a cluster is set up like this:
>
> | | |
> |---|---|
> | `history.persistenceGlobalMaxQPS` | 8000 |
> | history pods | 2 |
> | **effective rate per pod** | **about 4000/s** |
> | `history.persistenceMaxQPS` | 9000 — never changed from the default |
>
> The poll ceilings are worked out from the 9000, not from the 4000 the pod actually has:
>
> | Queue | Its poll ceiling |
> |---|---|
> | transfer | 0.30 x 9000 = 2700/s |
> | timer | 0.30 x 9000 = 2700/s |
> | outbound | 0.30 x 9000 = 2700/s |
> | **those three together** | **8100/s — against a 4000/s budget** |
>
> Task loading alone is permitted to ask for twice what the pod has, and nothing in the config looks
> wrong.
>
> Set `history.persistenceMaxQPS` to about 4000 instead and each of those ceilings becomes 1200/s,
> which is what you would expect from a 4000/s pod.
>
> This is also why changing `history.persistenceMaxQPS` on a cluster with a global limit set **can
> still change behaviour** — just not the behaviour you were aiming at. If you change it and
> something moves, this is the likely reason. It does not mean the global limit stopped applying.

### 2.6 What each setting does when set to `0`

This is the easiest thing here to get wrong, because `0` does something different depending on which
setting it is on.

| Setting you put `0` on | What happens |
|---|---|
| `history.persistenceNamespaceMaxQPS` | **Not off.** The per-namespace check keeps running, at the effective per-pod rate — `history.persistenceMaxQPS`, or `history.persistenceGlobalMaxQPS` if that is set. |
| `history.persistencePerShardNamespaceMaxQPS` | **Not off.** The per-shard check keeps running, also at the effective per-pod rate — `history.persistenceMaxQPS`, or `history.persistenceGlobalMaxQPS` if that is set. |
| `history.persistenceGlobalMaxQPS` | **Off.** `history.persistenceMaxQPS` supplies the per-pod rate instead. |
| `history.persistenceGlobalNamespaceMaxQPS` | **Off.** `history.persistenceNamespaceMaxQPS` supplies the per-namespace rate instead — and if that is `0` too, the effective per-pod rate does. |
| **`history.persistenceMaxQPS`** | **All three checks stop running.** See the warning below. |

So a cluster that has "left them alone" still has all three checks running. Two of them are simply
set to the pod number.

> ### ⚠ Setting the per-pod limit to `0` removes persistence rate limiting entirely
>
> Not just the per-pod check — **all three**. When the effective per-pod number is `0` or less, the
> limiters are never built at all, and the persistence layer runs unlimited. A per-namespace or
> per-shard limit you carefully set is discarded along with it.
>
> The one exception: if `history.persistenceGlobalMaxQPS` is above `0`, that supplies the per-pod
> number instead, and everything stays on.
>
> | `persistenceMaxQPS` | `persistenceGlobalMaxQPS` | Result |
> |---|---|---|
> | 9000 | 0 | all three checks on, at 9000 |
> | 0 | 0 | **no persistence rate limiting at all** |
> | 0 | 36000 | all three on, at each pod's share of 36000 — **but the queue reader poll ceilings are gone**, see below |
> | 9000 | 36000 | all three on, at each pod's share of 36000 — the 9000 is ignored |
>
> **If you want one limit out of the way, raise it. Do not zero the per-pod one to get there.**
>
> **Nothing will tell you this has happened.** There is no metric for the effective per-pod rate, and
> rejections simply stop — which looks identical to a healthy cluster that is not being throttled.
> The two ways to catch it:
>
> - **Guard it where dynamic config is set.** Refuse any change that leaves both
>   `history.persistenceMaxQPS` and `history.persistenceGlobalMaxQPS` at `0` or below. This is the
>   only reliable defence, because it stops the state being reachable.
> - **Watch the pod log.** The `Quota changed` line carries the computed quota, so a value of `0`
>   there means the limiters were not built. Needs the pod at `info` level.

---

## 3. How the limiter behaves in practice

Section 2 covered which of the three limits a database call is measured against, and what number
each limit uses. Those numbers are not the whole story. **Two database calls that look identical —
same operation, same pod, measured against the same limit — can get different answers.**

This section is about why, and what it costs:

| | |
|---|---|
| **3.1** | each limit is really **seven** separate allowances, and which one a call draws on is decided by *what triggered the call* — not by what the call does |
| **3.2** | how big a spike a limit absorbs before it starts rejecting |
| **3.3** | what a rejected call actually costs you |
| **3.4** | what these limits do **not** slow down, which is more than you would expect |

### 3.1 Seven buckets, one per priority

**How a limit works.** Each of the three limits is a *token bucket*. It holds tokens, refills them
at the rate you configured, and a database call has to take one token to go through. When the bucket
is empty, the call is rejected.

**Each limit is seven buckets, not one.** Every limit in section 2 is really seven separate buckets,
one for each priority level, numbered **0 (highest) through 6 (lowest)** — and all seven refill at
that limit's full rate. So:

| Limit | How many buckets |
|---|---|
| per-pod | seven, for the pod |
| per-namespace | seven **for each namespace** |
| per-shard, per-namespace | seven **for each namespace on each shard** |

**The buckets are per priority, not per operation.** This is the part that catches people out. There
is no bucket for `UpdateWorkflowExecution`. Unrelated operations share a bucket because they were
triggered at the same priority, and the *same* operation lands in a different bucket depending on
what triggered it. One `UpdateWorkflowExecution` write can end up in any of these:

| What triggered the write | Its priority |
|---|---|
| one of seven named API methods — `StartWorkflowExecution`, `SignalWorkflowExecution` and others | **1** |
| any other API call | **2** |
| a history task, most types | **4** |
| a history task for an activity or workflow task timeout, or a worker command | **5** |
| a history task that deletes or archives, or any task for a namespace active in another cluster | **6** |

> ### ⚠ Two different things share the name `UpdateWorkflowExecution`
>
> - **The Workflow Update API.** What an SDK client calls to send an update into a running workflow.
>   This is a frontend API method, and it is one of the seven that sit at **priority 1**.
> - **The database write.** What the history service issues whenever it saves changed workflow
>   state — which is most of what it does. This one has **no fixed priority at all**; it takes
>   whichever priority its caller had, per the table above.
>
> The table above is about the **database write**, not the API.
>
> The reason they behave differently is that the two names are looked up in different places. For a
> call that came from an API, the priority is decided by **which API method** started it, and the
> database operation's name is never looked at. The operation's name is only consulted for calls
> made by background work.

[Which bucket a call lands in](#which-bucket-a-call-lands-in) has the full rule and the complete
list of API methods.

**And "priority" means different things elsewhere in the server.** The same bucket mechanism is used
by the history task scheduler, the frontend and RPC limiters and the visibility store, and each has
**its own, different** set of priorities — the task scheduler's are not these. So "priority 4" only
means something once you have said which limiter you are talking about. Everything below is the
persistence limiter's.

#### How a call draws on the buckets

A call does two things:

1. It must **pass its own** bucket. If that bucket is empty, the call is turned away.
2. Having passed, it also **spends a token from every bucket below it**, without being checked
   against any of them.

Step 2 is the part that surprises people. It is easier to hold the other way round:

> **Each bucket is drained by its own priority and by everything above it.**

So the buckets are the same size but carry very different load:

| Bucket | Drained by | Effectively |
|---|---|---|
| 1 | priorities 0–1 | workflow starts, shard updates, queue checkpointing |
| 4 | priorities 0–4 | all of the above, **plus** most history task work |
| 6 | priorities 0–6 | **all traffic on the pod** |

That is why the bottom bucket is the real cap on total throughput, and why low-priority work runs
out of tokens first — it shares its bucket with everyone above it, while the top bucket has almost
no competition.

#### A worked example

*Illustrative — invented to show the arithmetic, not from any cluster.*

Per-pod limit **1000/s**, so all seven buckets refill at 1000 tokens per second. Workflow starts
arrive at **800/s** (priority 1) and child-workflow completions at **300/s** (priority 4).

| Bucket | Refills at | Spent by | Total drain |
|---|---|---|---|
| 1 | 1000/s | starts (checked here) | **800/s** — room to spare |
| 2, 3 | 1000/s | starts (side effect) | 800/s |
| **4** | 1000/s | starts (side effect) **+** child completions (checked here) | **1100/s — over** |
| 5, 6 | 1000/s | both, as side effects | 1100/s |

**The starts all succeed. About 100 child completions a second are turned away.** And from outside,
child completions were running at only 300/s against a 1000/s limit — nowhere near it on their own.

> **This is why "we were under the limit but saw rejections on one operation" is normal.** The
> operation being rejected is not the one using the budget. Look at what is running at a *higher*
> priority, not at the rejected operation's own rate.

#### Which bucket a call lands in

> **The persistence operation name does not decide this.** Priority comes from **who made the
> call**, and the rule differs by caller. The same database write sits in different buckets
> depending on what triggered it — an `UpdateWorkflowExecution` write is priority **1** when a
> workflow start does it and priority **4** when a history task does it. So do not look up a
> persistence operation here and expect one answer.

`0` is the highest priority.

| Priority | Decided by | What lands here |
|---|---|---|
| **0** | caller type | Operator API calls. Gets `system.operatorRPSRatio` (default 0.2) of the rate, not the full rate. |
| **1** | the **API method** being served | `StartWorkflowExecution`, `SignalWithStartWorkflowExecution`, `SignalWorkflowExecution`, `RequestCancelWorkflowExecution`, `TerminateWorkflowExecution`, `GetWorkflowExecutionHistory`, `UpdateWorkflowExecution` *(the Update API, not the database write of the same name)* |
| **1** | the **persistence operation** | `GetOrCreateShard`, `UpdateShard`, and `RangeCompleteHistoryTasks` for the transfer, timer and visibility queues — queue checkpointing, kept high so progress can always be recorded |
| **2** | caller type | Every other API call |
| **3** | the **persistence operation** | `GetHistoryTasks` for the transfer, timer and visibility queues — task loading, kept above other background work so tasks can always be read |
| **4** | the **history task type** | Most history tasks, including the child-completion write that lands on the parent's shard |
| **5** | the **history task type** | Activity timeouts, workflow task timeouts, workflow run and execution timeouts, worker commands |
| **6** | the **history task type** | Deletes, archival, unknown task types, and anything for a namespace active in another cluster |

Three things here are easy to miss:

- **Priority 1 is not only user traffic.** Shard updates and queue checkpointing sit there too. On a
  pod owning many shards that is constant background load competing with your workflow starts for
  the same bucket.
- **Queues other than transfer, timer and visibility are not promoted.** Outbound, archival and
  replication queue operations fall through to the caller-type default — priority 4 or lower.
- **The `operation` label on the dashboard is the persistence operation**, which as above does not
  by itself tell you the priority.

**The practical result:** a burst of workflow starts runs at priority 1 and drains buckets 2 through
6, so background work at 4, 5 and 6 starves first.

> **But that only holds while the limit still covers the priority-1 traffic.** **Measured** on the
> test cluster: for every workflow start that was rejected, how many child-workflow completions were
> rejected? Starts run at priority 1, child completions at priority 4, so a **big** number means the
> low-priority work is taking the hit, as intended.
>
> | Per-pod limit | Child completions rejected, per start rejected |
> |---|---|
> | 300 | **1.3** — starts were being turned away almost as often as the background work |
> | 2000 | **4.8** — background work was absorbing most of the rejection |
>
> At 2000 the priority order was doing its job. At 300 it was barely doing anything: a number near 1
> means high-priority and low-priority work are starving together. **Below the point where the
> priority-1 traffic alone fits, your API calls starve too.** Do not assume high priority is
> protecting them — group
> **[Rejected Database Calls by Operation and Scope](../observability/dashboards/server/temporal-server-readme.md#3-persistence-requests-latencies-and-errors)**
> by operation and look.

### 3.2 There is one second of burst room, and that is all

A bucket's size is its rate multiplied by `system.persistenceQPSBurstRatio`, which defaults to
**1.0**. So a pod limited to 9000 holds 9000 tokens — one second's worth.

A spike shorter than a second is absorbed. Anything longer is rejected.

#### Should you raise the burst ratio?

**Almost certainly not, and it is not a fix for throttling.** Raising it is the obvious-looking move
when you are being throttled, so it is worth knowing what happened when that was measured.

**Measured** twice on a clean test cluster, identical load both times, with this setting as the only
change — 1.0 to 4.0:

| At the peak | ratio 1.0 | ratio 4.0 | |
|---|---|---|---|
| throttled tasks/s | 805 | 869 | **+8%** |
| rejected database calls/s | 1079 | 1159 | **+7%** |
| database p99 | 0.2 s | 0.0 s | no harm done |

A few percent **worse** on every measure of throttling, and better on none.

**Why it does not help.** Burst room only buys time against a spike shorter than the bucket. Once
demand stays *above* the limit, the bucket empties and then stays empty no matter how big it is —
the extra tokens are spent in the first second and change nothing after that. If your throttling
lasts longer than the burst that caused it, demand is sustained and this is the wrong setting.

**Note also that this is a system-wide setting.** `system.persistenceQPSBurstRatio` applies to all
four services, so unlike the rest of the settings in this playbook it is not a history-specific
control — changing it changes the frontend, matching and worker bucket sizes at the same time.

### 3.3 What happens after a call is rejected

A rejected call is handed **straight back to whatever made it** — immediately, with no retry and no
waiting anywhere in between. So what happens next is entirely down to the caller, and two kinds of
caller matter.

**A history task made the call.** The task is put back on the queue and tried again later: after 3
seconds, then 1.5x longer each attempt, up to 5 minutes, for as long as it takes. Nothing is lost,
and queue lag grows while those tasks wait. This is the common case, and
[what happens when a QPS limit is reached](#11-what-happens-when-a-qps-limit-is-reached) covers it.

**An API call made it.** The `RESOURCE_EXHAUSTED` error travels back out to the client that sent the
request. Nothing on the server absorbs it on the client's behalf.

#### Not all rejections cost the same

Before a history task can do anything, it has to **read** the workflow's current state out of the
database. When it finishes, it has to **write** the new state back. Both of those are database
calls, so either one can be turned away — and the two cost very different amounts.

| The call that was rejected | What it costs you |
|---|---|
| the **read** | Nothing. No work had been done yet, so nothing is wasted. The task waits and tries again. |
| the **write** | The read and the work are both wasted. Temporal also drops the state it was holding in memory, so the next attempt has to read it out of the database all over again. |

**So a rejected write creates an extra read.** That is fine occasionally. When a lot of writes are
being rejected it becomes a cycle: each failed write causes another read, those reads spend the
budget the writes needed, and because the retries never stop the pod stays busy without finishing
much.

That cycle has a name — the **write-reject loop** — and it leaves a signature you can see on the
dashboard. It is worth knowing about because it changes which remedy works, so it is the thing to
check for before changing any setting.

### 3.4 A note on task processing

One more thing is wired into these limits, and it is worth knowing about even though this playbook
does not go into it.

The history task scheduler — the thing that decides how fast history tasks are handed out to be
worked on — has rate limits of its own, `history.taskSchedulerMaxQPS` and
`history.taskSchedulerNamespaceMaxQPS`. Both default to `0`, and `0` here means **"use the
persistence limit"**. So the two are connected out of the box, which is why it is easy to assume
they are one system. They are not.

Three things to know, and then we will leave it:

- **The scheduler's limiter is off by default.** `history.taskSchedulerEnableRateLimiter` is
  `false`, so on a cluster nobody has configured, nothing paces task dispatch at all. Tasks go out
  as fast as the scheduler can manage and are then turned away one at a time at the database —
  started and discarded, rather than never started.
- **Turning it on is not enough on its own.** It then starts in shadow mode
  (`history.taskSchedulerEnableRateLimiterShadowMode` defaults to `true`), which measures but holds
  nothing back.
- **The units are not the same.** Persistence limits count **database calls**. Task scheduler limits
  count **tasks**, and one task is several database calls — measured at 2.3 to 3.2 on the test
  cluster. So leaving the scheduler's QPS at `0` hands it a number several times too large for what
  it is counting.

Pacing task processing properly is a subject of its own, with its own settings and its own failure
modes, and it deserves a playbook of its own rather than a section here. Everything else in this
playbook is the persistence limits.

---

Sections 1 to 3 describe how these limits work when nothing has gone wrong. The rest of the playbook
is about the moment something has: throttling is happening on your cluster, and you need to work out
what is going on before you change anything.

---

## 4. Seeing what is going on

Throttling is unusually hard to read from a dashboard. Several panels you would naturally reach for
stay completely clean while it is happening, and the counters that *do* move disagree with each
other by an order of magnitude — all of which is normal and none of which is obvious.

So before working out which situation you are in, it is worth knowing which signals to trust.

### 4.1 First, confirm the limiter is what rejected the calls

Open **[History Rejected Database Calls Total by Scope](../observability/dashboards/server/temporal-server-readme.md#3-persistence-requests-latencies-and-errors)**
and look at the `resource_exhausted_cause` breakdown.

This panel ignores the dashboard's namespace selector, which is what you want here: a pod's own
internal work is filed under `system` rather than under your namespace
([why](#23-which-check-rejects-the-call-and-which-scope-you-see)), so a namespace selection hides
part of the picture. Its per-namespace twin, **[Rejected Database Calls by Operation and Scope](../observability/dashboards/server/temporal-server-readme.md#3-persistence-requests-latencies-and-errors)**,
is the one to use when you want to know *which* namespace or operation is being throttled — that is
[check 4](#check-4--who-is-being-throttled).

**If the cause is `PersistenceLimit`, you are in the right place.** Temporal turned the call away
itself, and the database was never asked — so nothing you see here says anything about database
health yet.

**If the cause is anything else, stop.** `SystemOverloaded` and `PersistenceStorageLimit` mean the
store pushed back, which is a different problem with a different fix.

**If your history store is SQL there is nothing to check here** — no SQL store raises
resource-exhausted at all, so `PersistenceLimit` is the only cause this metric can ever carry. The
other two are Cassandra only.

### 4.2 Four panels that stay clean while you are being throttled

All four were checked during real throttling on the test cluster. Each one reads perfectly healthy
while it is happening — not because the cluster is fine, but because none of them counts throttling
in the first place.

| Panel | What it reads | Why |
|---|---|---|
| **Persistence Availability** | **100%** | It is `100 - persistence_errors / persistence_requests`, and `persistence_errors` deliberately skips resource-exhausted errors. It also skips timeouts, shard-ownership-lost, condition-failed and not-found — so it stays at 100% through a struggling database too. |
| **Total Timer Tasks Errors** | **0** | It counts `task_errors`, which is *unexpected* errors only. Throttling is classed as expected, so it is counted somewhere else entirely. |
| **Timer Task Scheduling Latency** | minutes, and flat | This one has a high floor rather than a meaningful value. **Measured twice on an idle cluster with no workflows at all — 495 seconds, and 462 seconds at p95 on a different day** — and it did not move under load or under heavy throttling either time. Take your own quiet-cluster reading and treat that as your zero. |
| **Per-Shard Persistence RPS Distribution** | lower than reality | It is built from a **30-second** average against a token bucket that empties in one second. A 3-second spike at ten times the rate shows up here as roughly double. It cannot see a short burst at all. |

**Two panels that do show you what is happening:**

- **[History Rejected Database Calls Total by Scope](../observability/dashboards/server/temporal-server-readme.md#3-persistence-requests-latencies-and-errors)** —
  the rejections themselves, across every namespace. Its per-namespace twin **Rejected
  Database Calls by Operation and Scope** breaks the same metric down by operation.
- **[Persistence Errors by Namespace and Operation](../observability/dashboards/server/temporal-server-readme.md#3-persistence-requests-latencies-and-errors)** —
  built on `persistence_error_with_type`, which records **every** persistence error without
  exception. That includes the timeouts and other failures that Persistence Availability leaves out
  of its sum. **If you want to know whether the database itself is in trouble, this is the panel to
  use — not Persistence Availability.**

### 4.3 Three counters, three different numbers — all correct

There are three ways to count throttling and they do not agree. Comparing the wrong two is the usual
reason people conclude the numbers are broken.

Here is one real moment from the test cluster — all three read at the same instant, on the same
cluster, during the same throttling — to show how far apart they can sit:

| Metric | What it counts | Read at that moment |
|---|---|---|
| `service_errors_resource_exhausted` | **requests that failed.** One failed request counts once, however many database calls were rejected inside it | **111/s** |
| `persistence_errors_resource_exhausted` | **database calls turned away.** Includes every rejection whose caller simply retried and then succeeded | **1079/s** |
| `task_errors_throttled` | **history task attempts that were throttled** | **805/s** |

Nearly ten to one between the first two, at the same moment. Nothing was wrong with the cluster and
nothing was wrong with the metrics — they were counting three different events.

The gap reproduces. A later run on the same cluster at a heavier load read **250.5**, **2719.2**
and **2459.7** respectively — different absolute numbers, and the same ratio of **10.9 to 1**
between the first two. (Figures elsewhere in this playbook come from whichever run is named, so
they will not always match these.) Which is the point:
**if you read one of these expecting it to agree with another, you will draw the wrong conclusion
about how much throttling you have.**

> ### Use `persistence_errors_resource_exhausted`
>
> It is the only one of the three that reflects how much throttling is actually happening.
>
> `service_errors_resource_exhausted` sees only the part that broke through and failed a request —
> which, for background work, is none of it. Reading that number and concluding "only a little
> throttling" is the single most common mistake here. In the sample above it would have
> under-reported by a factor of nearly ten.
>
> **That metric is not useless, though — it answers a different question.** It counts requests that
> failed, tagged with the API or RPC method that failed, so it is exactly the right metric for *who
> is affected*. Use it for that in [check 4](#check-4--who-is-being-throttled), and use
> `persistence_errors_resource_exhausted` for how much throttling there is.

#### Why the task counter climbs so high

A throttled task is not lost, it is rescheduled — and it tries again, and again. So
`task_errors_throttled` is **not** "how many tasks are stuck". It is **how many attempts failed in
the last second**, and a backlog of tasks retrying on a timer produces those continuously. A few
thousand a second can be far fewer tasks, each retrying.

It is also why the number **falls away as sharply as it rose**: the attempts stop the moment tasks
start succeeding, even though no task was ever lost.

#### The same counter also records lock contention — read the two apart

`task_errors_throttled` is split by `resource_exhausted_cause`, and the
**[Transfer Active Task Errors Throttled](../observability/dashboards/server/temporal-server-readme.md#7-busy-workflow-throttling)**
panel draws each cause as its own line. Both causes are the server turning work away under load, so
they look similar on a graph — but what is running out is not the same thing:

| Line on that panel | What ran out |
|---|---|
| `PersistenceLimit` | **database budget.** A call was turned away by the QPS limiter. **This playbook.** |
| `BusyWorkflow` | **access to one workflow.** Most often a task waited for that workflow's lock and did not get it within `history.cacheNonUserContextLockTimeout`, 500ms by default. A few other conditions raise it too, such as a workflow that is closing. |

The distinction matters because nothing in this playbook moves the second one. Raising a QPS limit
does not reduce contention on a single workflow.

- `PersistenceLimit` high, `BusyWorkflow` near zero → the QPS limiter. Carry on here.
- `BusyWorkflow` high → contention on individual workflows. Start with the
  [Hot Shard](./hot-shard-detection-remediation.md) playbook instead.

### 4.4 The rate a pod is actually enforcing

There is no metric for this, and the number you configured is not always the number in force. When
the two disagree, read the real one off the pod's own log — the recipe is in
[a global limit replaces the per-pod one](#25-a-global-limit-replaces-the-per-pod-one).

### From reading the instruments to deciding what to change

Everything so far in this section has been about the instruments. You now know which panel shows the
rejections, which four panels stay clean while throttling is happening, which of the three counters
reflects the real volume, and how to find the rate a pod is genuinely enforcing.

None of that yet tells you what to change.

**That is what the rest of this section does.** It is a short sequence of readings, and the order
matters: the first one decides which setting the later ones even apply to, the second decides whether
raising anything is safe at all, and the last two decide whether the obvious fix is the right one.
Taken out of order, they are how people end up tuning a setting that was never the constraint.

Each check below ends with a recommendation and the dynamic config to go with it.

### 4.5 Four checks, and what each one tells you to change

**Where you are.** The history service is turning away its own database calls. You have confirmed
the cause is `PersistenceLimit` — the QPS limiter, not the store pushing back — and you have a real
figure for how much of it is happening, rather than the under-reported one.

**What you do not know yet** is the two things that decide what to do:

- whether this throttling is actually doing any harm, or is the limiter working as intended
- which of the five settings is the one genuinely holding your workload back

The four checks below answer both, and they also rule out the two ways of misdiagnosing this: tuning
a setting that was never in force, and raising a limit that was protecting a database which could not
take more.

Take them in order.

| # | Check | Where to look | What the answer means you should do |
|---|---|---|---|
| **1** | Is a cluster-wide limit in force? | `history.persistenceGlobalMaxQPS` in dynamic config | **Above `0`** → tune *that* setting. `history.persistenceMaxQPS` is not being enforced, so changing it will not move the throttling. **At `0`** → `history.persistenceMaxQPS` is the one to tune. |
| **2** | Was the database struggling? | **[Persistence Latencies](../observability/dashboards/server/temporal-server-readme.md#3-persistence-requests-latencies-and-errors)**, and **[Persistence Errors by Namespace and Operation](../observability/dashboards/server/temporal-server-readme.md#3-persistence-requests-latencies-and-errors)** for timeouts | **Flat, no timeouts** → raise the limit that check 1 identified. **Climbing, or timeouts present** → raise nothing. The limit was protecting a database that could not take more; add history pods or change how the work arrives instead. |
| **3** | Is the write-reject loop running? | **[Write-Reject Loop Indicator](../observability/dashboards/server/temporal-server-readme.md#3-persistence-requests-latencies-and-errors)** — the `workflow_context_cleared ÷ cache_miss` ratio | **Ratio above about 5** → the loop is running: rejected writes are throwing away completed reads, and the retries create fresh reads. Raising the limit from check 1 clears the loop, but only if check 2 said the database has room — if it did not, turn on admission control instead. **Ratio below about 1** → the loop is not running; those reads are ordinary cache misses. |
| **4** | Who is being throttled? | **[Resource Exhausted with Cause](../observability/dashboards/server/temporal-server-readme.md#6-throttling-and-limits)**, grouped by operation — **not** the persistence panel | **Requests are failing** → users or workers are getting errors, so this needs acting on. What you do depends on check 2 — see the combination table below. **Nothing here, or only internal RPCs** → no request actually failed, so this may need no change at all. |

#### Check 1 — Is a cluster-wide limit in force?

Look up `history.persistenceGlobalMaxQPS` in your dynamic config. There are only two answers.

**If it is `0`** — the usual case — then `history.persistenceMaxQPS` is the limit actually being
enforced. That is the setting the rest of this section means whenever it says "the limit". Move on to
check 2.

**If it is above `0`**, the limiter is using it *instead of* `history.persistenceMaxQPS`. The per-pod
setting is not being enforced at all, which has one blunt consequence: **raising
`history.persistenceMaxQPS` will not reduce your throttling by a single call.** Each pod's real rate
is its share of the cluster-wide number, weighted by how many shards it owns — see
[a global limit replaces the per-pod one](#25-a-global-limit-replaces-the-per-pod-one).

This is the first check because it decides which setting every later step is about.

> ### Recommendation: pick one of the two as your knob, and stop moving the other
>
> Having both set, and changing whichever comes to hand, is what makes this confusing in the first
> place.
>
> **Which one to pick comes down to whether your history fleet changes size.**
>
> | | Total load the cluster may send to the database |
> |---|---|
> | per-pod limit `P`, with `N` pods | `P x N` — **grows with the fleet** |
> | cluster-wide limit `G`, with `N` pods | **`G`, whatever `N` is** — each pod's share shrinks as pods are added |
>
> So on a fleet that scales, a per-pod limit quietly raises the ceiling every time you add pods.
> Going from 10 history pods to 20 **doubles the database load a per-pod limit permits**, with no
> config change and nothing in the config to show it happened. A cluster-wide limit holds that
> total flat and simply gives each pod a thinner slice.
>
> **If your history fleet autoscales, or you resize it by hand with any regularity, prefer
> `history.persistenceGlobalMaxQPS` as the control.** It is the only one of the two that expresses
> what you actually care about — how much load the database is willing to take — rather than a
> per-pod figure you then have to multiply in your head.
>
> On a fleet of fixed size, either works. The per-pod limit is simpler to reason about, and there
> is no hidden multiplier if the pod count never moves.
>
> **Two things follow from choosing the cluster-wide limit:**
>
> - **Scaling out history no longer buys you database throughput.** That is the point, but it
>   surprises people: add pods and each one gets a smaller share of the same total. If you are
>   scaling out *because* you need more database throughput, you have to raise
>   `persistenceGlobalMaxQPS` as well — and only after checking the database can take it.
> - **The per-pod number still matters, and it goes stale as the fleet grows.** You leave
>   `history.persistenceMaxQPS` set even though the limiter ignores it, for the reason given just
>   below — and the value is derived from the pod count, so it drifts when the fleet resizes.
>   Derive it at your normal operating size and revisit it if that size changes materially; being
>   somewhat out of date there degrades rather than breaks.
>
> **To tune the cluster as a whole** — keep `history.persistenceGlobalMaxQPS` and change only that
> from now on.
>
> Then set `history.persistenceMaxQPS` to roughly one pod's share of it, meaning
> `persistenceGlobalMaxQPS ÷ number of history pods`. **This is not about throttling** — the limiter
> ignores that value. It is because the queue readers still size their database polling off it, so
> left at the `9000` default they are sized for a budget the pod does not actually have.
>
> **Do not set it to `0` and drive everything from the cluster-wide limit.** That is a tempting
> tidy-up and it removes the ceiling rather than lowering it: with nothing left to compute from,
> each queue reader falls back to a hard-coded **100,000 per second**, which is effectively no
> limit at all. Task loading can then consume the pod's whole database budget — worst just after
> a restart, when a pod has to load for every shard it owns. The rate limiting itself still works
> in that configuration, because the limiter uses the calculated quota; it is only the poll
> ceilings that disappear, and nothing in the config looks wrong.
>
> **To tune each pod instead** — set `history.persistenceGlobalMaxQPS` to `0`. From that point
> `history.persistenceMaxQPS` is the enforced limit, and it is the only number you have to move.
>
> **Either way, one more thing to check.** Make sure `history.persistenceNamespaceMaxQPS` is not set
> to the same value as your per-pod limit. If it is, it is a second ceiling at the same height, and
> it will start rejecting calls the moment you raise the other one — see
> [do not set the per-namespace limit equal to the per-pod limit](#24-do-not-set-the-per-namespace-limit-equal-to-the-per-pod-limit).

#### Check 2 — Was the database struggling?

**This is the question the whole decision turns on.** Look at
**[Persistence Latencies](../observability/dashboards/server/temporal-server-readme.md#3-persistence-requests-latencies-and-errors)**
across the whole event, and check
**[Persistence Errors by Namespace and Operation](../observability/dashboards/server/temporal-server-readme.md#3-persistence-requests-latencies-and-errors)**
for timeouts. Do not use Persistence Availability — it stays at 100% either way.

| What you see | What it means |
|---|---|
| latency flat, no timeouts | The limiter was the only thing holding the workload back. The database had room. |
| latency climbing, or timeouts present | The limit was protecting a database that could not take more. (The climbing-latency half is measured; the timeout half is reasoned — a database slow enough to exceed Temporal's persistence timeout was never produced in testing.) |

> **Recommendation — latency was flat.** Raise the limit in steps, checking Persistence Latencies
> after each one. Which setting you raise is whichever one check 1 found to be the enforced limit,
> and there are two possibilities:
>
> ```yaml
> # check 1 found no cluster-wide limit — the common case
> history.persistenceMaxQPS:
>   - value: 12000          # was 9000, the default
> ```
>
> ```yaml
> # check 1 found a cluster-wide limit in force — raise that one instead
> history.persistenceGlobalMaxQPS:
>   - value: 48000          # was 36000
>
> # and move the per-pod value with it, to one pod's share — see check 1
> history.persistenceMaxQPS:
>   - value: 24000          # 48000 spread across 2 history pods
> ```
>
> **Those numbers are illustrative, not recommended values.** There is no correct step size. Start
> from whatever you have now and move by an amount whose effect you will actually be able to see — a
> third more rather than ten times more. Run the load, look at latency, repeat.
>
> **Stop when latency starts to climb.** That is the database telling you where its own ceiling is,
> and it is the only honest source for that number — there is nothing in the config that knows it.
>
> **Also turn on adaptive backoff, as a safety net under the raise.**
>
> This is a separate mechanism from the limits themselves. It lets the limiter lower its *own* rate
> below the number you configured when the database starts looking unhealthy, and bring it back up
> when the database recovers. It can never go **above** your number, so it cannot undo the raise —
> it only cushions it if you have gone too far.
>
> **How it knows.** Every database call the history service makes is already being timed — the same
> layer that emits the persistence metrics records each call's duration and whether it failed into a
> per-pod running average. Backoff simply reads that average:
>
> | Step | What happens |
> |---|---|
> | continuously | each database call's latency and outcome is recorded into a rolling average, over a window set by `system.persistenceHealthSignalWindowSize` (10s by default) |
> | every `refreshInterval` | the limiter reads the **average** latency and the error ratio |
> | if either threshold is exceeded | the rate is multiplied down by `rateBackoffStepSize` (0.3), never below `rateMultiMin` |
> | if both are fine and the rate is still reduced | it steps back up by `rateIncreaseStepSize` (0.1), never above `rateMultiMax` |
>
> That is why the threshold is compared against an **average** — an average is what the layer
> underneath actually keeps. It is also why backoff moves in steps rather than jumping: it only gets
> one reading every ten seconds.
>
> It is off by default. You only need to set the fields you are changing: anything you leave out
> keeps its default, because your values are merged over them.
>
> ```yaml
> history.persistenceDynamicRateLimitingParams:
>   - value:
>       enabled: true
>       latencyThreshold: 200.0   # milliseconds. Default 0.0 means "never back off on latency"
>       errorThreshold: 0.05      # Default 0.0 means "never back off on errors"
>       rateMultiMin: 0.2         # Default 0.8, which allows only a 20% cut
> ```
>
> Three things about that block:
>
> - **`enabled: true` on its own does nothing.** Both thresholds default to `0.0`, which means the
>   limiter never concludes the database is unhealthy. You have to set at least one of them.
> - **The default floor is too shallow to be a real backstop.** `rateMultiMin` defaults to `0.8`,
>   so out of the box backoff can only ever cut the rate by 20 per cent. **Measured**: at the
>   default the multiplier fell to `0.8` and stopped there, in a single step, and never went
>   lower however long the pressure lasted. With `rateMultiMin: 0.2` the same cluster stepped
>   down 0.8, 0.5, 0.2 — three steps of `rateBackoffStepSize` — and held at `0.2`. Recovery is
>   slower than backoff: it climbs in `rateIncreaseStepSize` increments of 0.1.
> - **`latencyThreshold` is measured against the pod's average persistence latency, not p99** — and
>   that average is kept in **whole milliseconds**. Each call is truncated to an integer before
>   being averaged, so a call taking 0.4 ms counts as **0**. **Measured** on a healthy test
>   cluster averaging **0.47 ms**: every call truncated to zero. Two consequences — the
>   threshold's real resolution is 1 ms, and reading your "healthy latency" off Persistence
>   Latencies can mislead you, because that panel shows sub-millisecond detail backoff cannot
>   see. Set the threshold from what your database does when it is *unhealthy*.
>
> No restart is needed.
>
> **Two ways this ends up doing nothing at all:**
>
> - If any single field in that block fails to parse, the whole setting silently falls back to its
>   defaults — which includes `enabled: false`.
> - It depends on the latency tracking underneath. Both
>   `system.persistenceHealthSignalMetricsEnabled` and
>   `system.persistenceHealthSignalAggregationEnabled` default to `true`, so this is normally
>   fine — but if either is off when the pod starts, there is no average latency to compare
>   against and backoff can never fire. **Both are read once at pod startup, not on every
>   evaluation**, so changing them takes effect only after a restart — confirmed by measurement:
>   setting the aggregation flag to `false` on a running pod changed nothing at all.
>
> **How to see whether it is doing anything.** Open
> **[Adaptive Rate Limit Multiplier](../observability/dashboards/server/temporal-server-readme.md#3-persistence-requests-latencies-and-errors)**.
> `1.0` means no reduction; anything lower is backoff actively cutting the pod's rate. The pod also
> logs `Health threshold exceeded, reducing rate limit.` when it acts, but at `info` level — so on a
> cluster running at `warn` the panel is the only place you will see it.
>
> **An empty panel does not mean the setting is off.** The value is only recorded when the
> multiplier moves, so an enabled limiter on a healthy database emits nothing at all. A line
> appearing proves backoff has acted; no line tells you only that it has not.
>
> **Recommendation — latency was climbing.** **Do not raise the limit.** It was doing its job, and
> raising it moves the problem into the database, where the failures are far less forgiving
> ([why that matters](#11-what-happens-when-a-qps-limit-is-reached)). No dynamic config setting makes
> the work smaller; the options are to add history pods, so each one carries fewer shards, or to
> change how the work arrives.

#### How low is too low? The saturation line

If check 2 says the database was fine, this tells you whether the limit is genuinely undersized for
your workload rather than just briefly exceeded.

**This one is a hand calculation, not a panel.** Nothing can plot it for you, because the rate a pod
is enforcing has no metric. But both numbers it needs are easy to get:

| Number you need | Where to get it |
|---|---|
| the effective per-pod rate | check 1 tells you which setting is enforced; [§4.4](#44-the-rate-a-pod-is-actually-enforcing) reads the real value off the pod |
| your shard count | **[Owned Shards (Total)](../observability/dashboards/server/temporal-server-readme.md#8-shard-movement)**, or query `sum(numshards_gauge{service_name="history"})` directly |

```
saturation line  =  effective per-pod rate  ÷  shard count
```

That is the average per-shard rate at which one pod runs out of budget. Compare it against
**[Per-Shard Persistence RPS Distribution](../observability/dashboards/server/temporal-server-readme.md#3-persistence-requests-latencies-and-errors)**:

| Where the line falls | What it means |
|---|---|
| above `max` | No single shard is filling a pod. Your problem is total volume or burstiness, not the per-shard spread. |
| between `p99` and `max` | The busiest shards are filling pods; ordinary ones are not. |
| below `p99` | Even unremarkable shards are over budget. The limit is low for this workload. |

> ### ⚠ Use `p99` and `max`, not `p50`
>
> **`p50` on this panel is usually not a real number.** The metric behind it has its lowest
> histogram bucket at **1 request per second**, and on a cluster with many shards almost every
> shard sits below that. When every observation lands in one bucket there is nothing to
> interpolate between, so the quantile reflects how full that bucket is rather than measuring
> anything.
>
> **Measured** on the test cluster: **79 per cent of shard observations were below 1/s even while
> the cluster was rejecting 2,719 calls per second.** `p50` read **0.5** on a completely idle
> cluster and **0.67** under that heavy load — both artifacts. `p99` and `max` read **13.0** and
> **50.0**, which are real, because they interpolate between real bucket boundaries.
>
> **So check the buckets have spread before trusting any of this.** If `max` is 1 and `p50` is
> 0.5, every shard is under 1/s, the panel cannot tell you anything, and you should judge from
> total volume instead.
>
> **Treat the result as a rough sizing check, not a verdict.** It is useful for telling a low
> limit apart from a hot shard, and for sanity-checking whether ordinary load already fills a
> pod. It is not precise enough to justify a change on its own — check 2 is what decides that.

> **To keep it in front of you**, add a threshold reference line at that value on the Per-Shard
> Persistence RPS Distribution panel. The dashboard already uses threshold lines this way on other
> panels. It has to be a number you type in rather than a query, for the same reason as above — the
> effective rate is not a metric, so nothing can derive it.

> **This is also what tells a hot shard apart from a low limit.** An uneven per-shard spread looks
> like a hot shard, but if
> **[Hottest Shard RPS](../observability/dashboards/server/temporal-server-readme.md#3-persistence-requests-latencies-and-errors)**
> shows a *low absolute rate*, no single shard is exhausting anything — the shape is misleading and
> the limit is the problem. Remember the per-shard panel is a 30-second average and cannot see a
> short burst at all.

#### Check 3 — Is the write-reject loop running?

Rejected writes throw away completed reads, and the retries read again
([not all rejections cost the same](#not-all-rejections-cost-the-same)). You do not have to work
anything out to spot that: open
**[Write-Reject Loop Indicator (cleared / cache miss)](../observability/dashboards/server/temporal-server-readme.md#3-persistence-requests-latencies-and-errors)**
and read the line labelled **loop indicator**. The panel does the division for you.

It plots three lines:

| Line | What it is |
|---|---|
| **loop indicator** | the ratio, across all pods. **This is the one to read.** |
| cleared/s, per pod | how often cached workflow state is being thrown away |
| cache miss/s, per pod | how often state had to be read because it genuinely was not cached |

The ratio is `workflow_context_cleared ÷ cache_miss` — state being discarded, divided by state that
was simply not there. Both go up under load; it is which one is winning that matters.

**How to read the number:**

| Ratio | What it means |
|---|---|
| **below about 1** | Normal. Reads are ordinary cache misses. |
| **around 1 to 2** | Some rejected writes, not a runaway loop. |
| **above about 5** | The loop is running. Cached state is being thrown away faster than it is being missed. |

**Measured** across four states on the test cluster, with the raw numbers so you can see both sides
move together:

| State | cleared/s | cache miss/s | ratio |
|---|---|---|---|
| healthy | 43 | 85 | **0.5** |
| cache too small, limit generous | 122 | 143 | **0.9** |
| partly throttled | 158 | 126 | **1.3** |
| **write-reject loop** | **546** | **60** | **9.2** |

**Do not look for cache misses near zero** — they were still 60/s inside the loop. Only the ratio
separates the two causes.

**One thing to know about this panel: it is cluster-wide, not per-namespace.** The namespace selector
at the top of the dashboard does not apply to it, because `workflow_context_cleared` carries no
namespace label at all. On a cluster with several busy namespaces the ratio blends them.

> **Recommendation — it depends on check 2.**
>
> **If the database had room** → raise the per-pod limit. That clears the loop on its own:
> **measured**, it took the indicator from **9.2 to 1.3**, because once writes stop being rejected
> they stop discarding good reads.
>
> **If the database had no room** → raising the limit is not available to you, and **no setting in
> this playbook fixes the loop.** What does fix it is holding tasks back *before* they start, so that
> an admitted task has the budget to finish both its read and its write. That is the history task
> scheduler's admission control, and it is deliberately out of scope here — see
> [a note on task processing](#34-a-note-on-task-processing). Turning it on without sizing it does
> nothing at all, and sizing it properly takes its own measurements, so it belongs in a playbook of
> its own rather than a box in this one.
>
> Until then, the two things that reduce the loop without any new setting are the same two as
> check 2: fewer shards per pod, by adding history pods, or less work arriving.

#### Check 4 — Who is being throttled?

**This check uses a different panel from the rest of section 4.** Group
**[Resource Exhausted with Cause](../observability/dashboards/server/temporal-server-readme.md#6-throttling-and-limits)**
by operation — not the persistence panel you have been using.

The reason is that the two panels label their operations differently, because they watch different
layers. On the persistence panel, `operation` is the **database** operation — `UpdateWorkflowExecution`,
`GetWorkflowExecution`, `GetTransferTasks` — and a database operation name can never tell you who was
affected, because the same `UpdateWorkflowExecution` write is made on behalf of a client call and on
behalf of a background task alike. **Resource Exhausted with Cause** labels each failure with the
**API or RPC method that failed**, and that is what identifies the caller.

So: the persistence panel tells you *how much* throttling there is. This one tells you *who is paying
for it*.

The distinction that matters is **whether anything outside the cluster is failing**. Some of what
this panel shows are calls an SDK client was waiting on; the rest are calls Temporal makes to itself
between its own services, which get retried without anyone noticing.

| What you see on Resource Exhausted with Cause | What it means |
|---|---|
| **client-facing methods** — `StartWorkflowExecution`, `SignalWorkflowExecution`, `UpdateWorkflowExecution` | Requests are failing back to SDK clients. Your users are seeing this. |
| **internal RPCs that carry workflow progress** — `RecordChildExecutionCompleted`, `RecordActivityTaskStarted`, `RespondWorkflowTaskCompleted` | No client request fails, but **executions stall.** A throttled `RecordChildExecutionCompleted` is a finished child that cannot tell its parent, so the parent waits. Users see slowness rather than errors — and they do notice. |
| **background work only** — deletes, archival, visibility | Genuinely just delayed. Nothing is lost and nothing is waiting on it. |
| **nothing at all here**, while Rejected Database Calls shows plenty | Every rejection was absorbed by a retry. The throttling is real and nobody outside the cluster noticed. |

> **Recommendation.** On its own, this check does not tell you what to change — it tells you **how
> urgent this is:**
>
> - **client-facing methods failing** — users are getting errors now. Urgent.
> - **internal RPCs that carry workflow progress** — no errors, but executions are stalling.
>   **Treat this as urgent too.** It is the case most often waved away as "only internal".
> - **background work only** — nothing is lost and nothing is waiting on it. You have time.
>
> What to actually do comes from putting this together with check 2 — see [the decision](#46-the-decision--what-to-actually-do).

> **Do not assume your API calls are protected.** They run at a high priority, but that only helps
> while the limit still covers the priority-1 traffic — **measured**, at a tight limit workflow
> starts were being rejected almost as often as background work
> ([seven buckets, one per priority](#31-seven-buckets-one-per-priority)). Check, do not assume.

### 4.6 The decision — what to actually do

This is where the playbook lands. Everything above it exists to get you two readings; this section
turns them into an action.

**If you have skipped straight here, take these first.** Each is one panel and under a minute:

| Reading | Where | What you are looking for |
|---|---|---|
| **Did the database have room?** | **[Persistence Latencies](../observability/dashboards/server/temporal-server-readme.md#3-persistence-requests-latencies-and-errors)** across the event, and **[Persistence Errors by Namespace and Operation](../observability/dashboards/server/temporal-server-readme.md#3-persistence-requests-latencies-and-errors)** for timeouts | flat with no timeouts, or climbing? |
| **Who is affected?** | **[Resource Exhausted with Cause](../observability/dashboards/server/temporal-server-readme.md#6-throttling-and-limits)**, grouped by operation | are client-facing methods failing, or only Temporal's own internal calls? |
| **Is the write-reject loop running?** | **[Write-Reject Loop Indicator](../observability/dashboards/server/temporal-server-readme.md#3-persistence-requests-latencies-and-errors)** | is the ratio above about 5? |

> **If the loop is running, clear it before anything else.** It consumes the very budget every action
> below is trying to free up, and clearing it can change which row you end up in. See
> [check 3](#check-3--is-the-write-reject-loop-running).

#### The four outcomes

| | **Client requests are failing** | **No client requests failing** |
|---|---|---|
| **Database had room** | **A — raise the limit.** The clearest case there is. | **B — your call.** Nothing is being lost. |
| **Database could not take more** | **C — the hard one.** No quick fix exists. | **D — leave it alone.** Working as designed. |

> **One qualifier on the right-hand column.** "No client requests failing" is not the same as
> "no user impact". If the throttled operations are the ones carrying workflow progress — child
> completions, activity task starts — then executions are stalling even though nothing returns
> an error. Treat those rows as though clients were affected. See
> [check 4](#check-4--who-is-being-throttled).

---

**Outcome A — database had room, clients affected: raise the limit.**

This is the case the limit is simply too low. The limiter is the only thing in the way, and it is
costing your users.

- **Change:** the limit that [check 1](#check-1--is-a-cluster-wide-limit-in-force) identified, in
  steps, with [adaptive backoff](#check-2--was-the-database-struggling) turned on underneath.
- **How you will know it worked:** rejections fall on **[Rejected Database Calls by Operation and Scope](../observability/dashboards/server/temporal-server-readme.md#3-persistence-requests-latencies-and-errors)**,
  **[Persistence Latencies](../observability/dashboards/server/temporal-server-readme.md#3-persistence-requests-latencies-and-errors)** stays flat, and queue lag comes down on
  **[Immediate Queue Lag per Pod](../observability/dashboards/server/temporal-server-readme.md#9-shard-queue-health)** and **[Scheduled Queue Lag per Pod](../observability/dashboards/server/temporal-server-readme.md#9-shard-queue-health)**.
- **Measured**, from one raise on the test cluster with the database healthy throughout: throttled
  operations **down 77%**, rejections on child completions **down 69%**, on workflow starts **down
  92%**, reads actually reaching the database **up 85%**, the backlog drained **41 times faster**,
  and the loop indicator fell from **9.2 to 1.3**.

> **Expect throttling to move rather than vanish.** After that raise there were still rejections —
> but on different operations. Before: workflow reads, driven by the loop. After: **queue task
> loading** (`GetTransferTasks`, `GetVisibilityTasks`, `GetTimerTasks`), which is a far more benign
> place to be throttled. So "still seeing `PersistenceLimit`" after a raise is normal. **Check which
> operation before deciding it did not work.**

---

**Outcome B — database had room, no client requests failing: your call.**

Background work was delayed and retried. Nothing was lost, no user saw anything, and the queue
caught up.

- Raise the limit if the queue lag itself is a problem for you — late timers, activities slow to
  start.
- Otherwise leave it. This is a judgement call, not an incident.
- **Before settling on "leave it", confirm it really was fine.** All four should hold: the loop
  indicator stayed below 1, persistence latency stayed flat, queue lag rose and came back down on
  its own, and nothing reached the dead-letter queue —
  **[Dead-Lettered Tasks — Execution-Stranding](../observability/dashboards/server/temporal-server-readme.md#20-history-task-dlq--terminal-failures)**.

---

**Outcome C — database could not take more, clients affected: the hard one.**

The limit was protecting a database already at its ceiling, and it is your users paying for that.
There is no setting that fixes this, and that is worth saying plainly.

- **Do not raise the limit.** It trades clean, recoverable rejections for timeouts — and timeouts
  *do* count toward the dead-letter threshold, where work stops retrying and waits for a human. See
  [what happens when a QPS limit is reached](#11-what-happens-when-a-qps-limit-is-reached).
- **What actually helps** is a database that can take more, or less work arriving. Give the database
  more capacity, or reduce the work: spread bursts of starts over a longer window, cut child-workflow
  fan-out, and look for repeated calls against the same workflow ID.
- **Adding history pods does not fix this one.** More pods do not make the database faster. Under a
  per-pod limit they raise the *total* load the cluster is allowed to send, which is the opposite of
  what you want here. Under a cluster-wide limit they do not change the total at all — each pod
  simply gets a smaller share.
- **Adding shards does not either, and cannot be done anyway.** The per-pod limit is *per pod*, so
  doubling the shard count across the same pods leaves the total budget exactly where it was. It
  halves the per-shard rate, which makes the per-shard distribution panel look healthier, and
  changes nothing about when a pod runs out of tokens. Shard count also cannot be changed on a live
  cluster, so this is an expensive way to achieve nothing.

---

**Outcome D — database could not take more, no client requests failing: leave it alone.**

This is the limiter doing precisely what it exists for: the database is at its ceiling, and the only
thing being delayed for it is background work that will retry until it succeeds.

Confirm the same four conditions as outcome B, then write down what normal looks like on your cluster
so the next person recognises it.

---

## 5. Alerting on rejected database calls

Section 4 assumes you are already looking at a dashboard. There is one alert in this set that tells
you instead — and it is worth knowing how much of section 4 it does and does not do for you.

| | |
|---|---|
| **What alert 86 tells you** | that history persistence rejections are happening, at what rate, and with which cause and scope. That is the confirmation step in [4.1](#41-first-confirm-the-limiter-is-what-rejected-the-calls), done for you. |
| **What alert 86 does not tell you** | any of [the four checks](#45-four-checks-and-what-each-one-tells-you-to-change) — not whether the database was struggling, not whether the write-reject loop is running, and not who is affected. Those still need you at the dashboard. |

### 5.1 Alert 86 — History Database Calls Rejected

**[86 — History Database Calls Rejected](../observability/alerts/server/runbooks/86-history-database-calls-rejected.md)**
watches history persistence rejections and fires above **10 per second, sustained for 10 minutes**.
It reports separately per scope and per cause, so a `PersistenceLimit` rejection and a store-side
`SystemOverloaded` one arrive as different alerts rather than being added together.

It carries `severity: warning`. How that routes is yours to decide — the alert set ships with labels
and no notification policy — but the reason it is not marked critical is worth knowing when you wire
it up: throttling is recoverable by design, work waits and retries, and the thing genuinely worth
waking someone for is a database in trouble rather than a limiter doing its job.

**When it fires, start at [the four checks](#45-four-checks-and-what-each-one-tells-you-to-change).**

### 5.2 Adjusting when alert 86 fires

Alert 86's condition — **more than 10 rejections per second, sustained for 10 minutes** — is a
starting point, not a target. Two things make it wrong for a given cluster:

| If your cluster... | Then... |
|---|---|
| throttles routinely under normal load, and recovers on its own every time | the alert will fire on healthy behaviour. Raise the rate, or lengthen the 10 minutes, until it only fires on something you would act on. |
| never normally throttles at all | you can afford to hear about it much earlier. Lower the rate. |

**Of the two numbers, the 10 minutes is doing more work than the 10 per second.** A burst that
throttles hard and drains in thirty seconds is the limiter working as intended, and the 10-minute
window is what keeps that quiet. Shorten it and you will hear about every burst.

**Set the threshold from your own baseline, not from this number.** Look at
**[Rejected Database Calls by Operation and Scope](../observability/dashboards/server/temporal-server-readme.md#3-persistence-requests-latencies-and-errors)**
across a normal week, take the level your cluster sits at when nothing is wrong, and set the alert
above it.

### 5.3 What no alert will catch

**Persistence rate limiting being switched off entirely.** If the effective per-pod rate reaches `0`,
the limiters are never built and nothing is rate limited at all — and there is no metric for it, so
no alert can watch for it. Rejections simply stop, which looks identical to a healthy cluster.

The defence is a guard where dynamic config is set, refusing any change that leaves both
`history.persistenceMaxQPS` and `history.persistenceGlobalMaxQPS` at `0` or below. See
[what each setting does when set to `0`](#26-what-each-setting-does-when-set-to-0).

---


---

## 6. Worked examples

These are three examples of this playbook's recommendations applied to a real situation — what was
on the panels, what the checks said, and what was done about it.

They **complement** the sections before them rather than replacing them. The checks in
[section 4.5](#45-four-checks-and-what-each-one-tells-you-to-change) and
[the decision](#46-the-decision--what-to-actually-do) are still the method; these just show what
working through that method looks like from beginning to end, with real numbers attached.

**Three examples cannot cover every case.** There are many more combinations of readings than
this, and your situation may well not match any of them — that is normal, and the checks still
give you an answer. Read these to see how the checks fit together and what a decision looks
like once they are answered, then take your own readings through section 4.5 and arrive at your
own outcome.

### 6.1 Rejections on child-workflow completions, with both limits set to the same number

**What you saw.** `RESOURCE_EXHAUSTED` on **[Resource Exhausted with Cause](../observability/dashboards/server/temporal-server-readme.md#6-throttling-and-limits)**, cause
`PersistenceLimit`, operation `RecordChildExecutionCompleted`. Nothing failing for SDK clients, but
parent workflows taking far longer than they should. And your config has
`history.persistenceMaxQPS` and `history.persistenceNamespaceMaxQPS` set to **the same value**.

**Working the checks.**

| Check | Answer |
|---|---|
| 1 — cluster-wide limit? | No, so `persistenceMaxQPS` is the enforced limit. **But the namespace limit is set to the same number** — [the most common mistake here](#24-do-not-set-the-per-namespace-limit-equal-to-the-per-pod-limit) |
| 2 — database struggling? | Answer this before anything else. Assume here that Persistence Latencies was flat with no timeouts |
| 3 — write-reject loop? | Check the indicator. Assume below 1 here |
| 4 — who is affected? | `RecordChildExecutionCompleted` is an **internal RPC that carries workflow progress** — a finished child that cannot tell its parent. No client error, but executions are stalling |

**The trap, and the reason this example is worth reading.** With the namespace rate equal to the pod
rate, the two are separate buckets refilling at the same speed — and **the namespace check runs
first**. So the namespace check is what rejects, and **raising `history.persistenceMaxQPS` would
change nothing**: the namespace limit stays where it is and keeps rejecting.

The `resource_exhausted_scope` tag says so directly. It reads **`Namespace`**, not `System`.
**Measured** on the test cluster with both set to the same value: Namespace-scope **1552/s** against
System-scope **472/s** — the namespace check rejecting **3.3 times more**.

**What to change, in this order.**

1. **Clear the duplicate ceiling first.** Either let the namespace limit follow the pod limit:

   ```yaml
   history.persistenceNamespaceMaxQPS:
     - value: 0
   ```

   or set it *genuinely lower* than the pod limit if you do want that namespace capped. What you must
   not leave is the two sitting at the same number.

   **A `0` here is safe, and a `0` on the pod limit is not.** Zeroing the namespace limit simply
   makes that check run at the effective per-pod rate, and nothing else reads the setting. Zeroing
   `history.persistenceMaxQPS` is a different matter entirely — see
   [what each setting does at `0`](#26-what-each-setting-does-when-set-to-0). Do not
   generalise from one to the other.

2. **Then raise the pod limit**, in steps, since check 2 said the database had room — see
   [check 2](#check-2--was-the-database-struggling).

**What happens next.** The scope tag on any remaining rejections moves from `Namespace` to `System`,
which tells you the namespace ceiling is gone and the pod limit is now the binding one. From there,
raising the pod limit actually moves the ceiling.

**Note what this fix does not do: it does not introduce a cluster-wide limit** — and a
cluster-wide limit would not have fixed this anyway. **The host limits and the namespace limits are
two independent pairs.** `history.persistenceGlobalMaxQPS` replaces `history.persistenceMaxQPS`, and
nothing else: `history.persistenceNamespaceMaxQPS` keeps the value you gave it, keeps being checked
**first**, and keeps rejecting. The only setting that replaces it is
`history.persistenceGlobalNamespaceMaxQPS`.

**Measured** on the test cluster: with `history.persistenceGlobalMaxQPS` at **36000** in force — a
per-pod share of roughly 20000 — and `history.persistenceNamespaceMaxQPS` set to **5**, read load
on one namespace was rejected at scope **`Namespace`** (`GetCurrentExecution`,
`GetWorkflowExecution`), with **no `System`-scope rejections at all**. The cluster-wide limit was
nowhere near binding; the per-pod namespace limit did every rejection.

So whichever kind of limit you prefer, **the duplicate
namespace ceiling has to be cleared either way** — which is why it is step 1, and why adding a
cluster-wide limit in the middle of an incident only changes which setting is enforced and brings in
shard-weighted division on top.

**Afterwards, a cluster-wide limit is worth adopting — and more so if your history fleet
autoscales.** A per-pod limit is a *per-pod* ceiling: `N` pods at `P` each permit `P x N` in total,
so the database's real ceiling rises every time the fleet grows, with no config change to show it.
The cluster-wide numbers hold the total flat instead. **The same arithmetic applies to the namespace
limit**, which is also per-pod — one namespace at `history.persistenceNamespaceMaxQPS` on `N` pods
can send `N` times that value — so if you want a namespace genuinely capped across the cluster, the
setting for that is `history.persistenceGlobalNamespaceMaxQPS`, not the per-pod one.

If you do adopt the cluster-wide pair, follow
[check 1](#check-1--is-a-cluster-wide-limit-in-force): tune the cluster-wide number from then on, and
**still leave `history.persistenceMaxQPS` set** to roughly one pod's share of it — the limiter
ignores it, but the queue processors size their polling from it, so **`0` there removes the poll
ceilings rather than lowering them**. Setting only a cluster-wide limit and zeroing the per-pod one
is the one combination to avoid.

**And do not file this under "only internal, it can wait."** No client request failed, but every
throttled child completion is a parent workflow left waiting, and that is visible to users as
slowness.

### 6.2 Heavy throttling, database perfectly healthy

*Measured end to end on the test cluster.*

**What you saw.** **[Rejected Database Calls by Operation and Scope](../observability/dashboards/server/temporal-server-readme.md#3-persistence-requests-latencies-and-errors)** at **2,719/s**,
dominated by `GetWorkflowExecution` at 1,682/s. On the service panel, `StartWorkflowExecution`
failing at **241/s**. Queue lag climbing.

**Working the checks.**

| Check | Reading | Answer |
|---|---|---|
| 1 | `persistenceGlobalMaxQPS` is `0` | `persistenceMaxQPS` is the knob |
| 2 | Persistence Latencies **p95 1.5 ms**, no timeouts | **flat** — the limiter was the only constraint |
| 3 | loop indicator **13.08**, cache_miss 189.8/s | **the loop is running** |
| 4 | `StartWorkflowExecution` present | **clients are failing** |

Note what the other panels said at that same moment: **Persistence Availability 100 per cent**, Total
Timer Tasks Errors **no data**, Timer Task Scheduling Latency **unchanged from idle**. Three panels
reading perfectly healthy through 2,719 rejections a second.

**Which outcome.** Database had room, clients affected — **outcome A, raise the limit.** And because
the database had headroom, raising it also clears the loop, so admission control is not needed.

**What to change.** Raise `history.persistenceMaxQPS` in steps, with adaptive backoff underneath and
`rateMultiMin` lowered from its default — see
[check 2's recommendation](#check-2--was-the-database-struggling).

**What happened next**, measured across one raise with the database healthy throughout: throttled
operations **down 77 per cent**, rejections on child completions **down 69 per cent**, on workflow
starts **down 92 per cent**, reads reaching the database **up 85 per cent**, the backlog drained **41
times faster**, and the loop indicator fell from **9.2 to 1.3**.

Throttling did not stop — it **moved**, to queue task loading (`GetTransferTasks`,
`GetVisibilityTasks`, `GetTimerTasks`), a far more benign place to be throttled. Check which
operation before concluding a raise did not work.

### 6.3 The database is at its ceiling

*Assembled from two measured runs rather than one incident. The latency figures and the Availability
reading are real, taken from a deliberately CPU-starved database; the rejection rate is from the run
in 6.2.*

**What you saw.** Rejections on the persistence panel, **and** Persistence Latencies climbing hard —
average latency **0.295 ms rising to 64 ms**, p95 to **220 ms**.

**Working the checks.** Check 2 is the only one you need here. Latency is climbing, so the limit was
protecting a database that could not take more.

**The panel that will mislead you.** **Persistence Availability read exactly 100.0** with the database
**218 times slower** than baseline. It is the obvious panel to reach for when asking "is my database
healthy", and it answers wrongly. Use **[Persistence Latencies](../observability/dashboards/server/temporal-server-readme.md#3-persistence-requests-latencies-and-errors)** and
**[Persistence Errors by Namespace and Operation](../observability/dashboards/server/temporal-server-readme.md#3-persistence-requests-latencies-and-errors)** instead.

**Which outcome.** **Outcome C — the hard one.** Do not raise the limit.

**What to change.** Nothing in dynamic config fixes this, and that is worth saying plainly:

- **Do not raise the limit.** It trades clean, recoverable rejections for timeouts — and timeouts do
  count toward the dead-letter threshold, where work stops retrying and waits for a person.
- **Give the database more capacity, or send it less work**: spread bursts of starts over a longer
  window, reduce child-workflow fan-out, look for repeated calls against the same workflow ID.
- **Adding history pods will not help**, and under a per-pod limit it makes things worse by raising
  the total load the cluster is allowed to send.

---

## 7. Reference — every setting this playbook mentions

All of these are dynamic config. Most take effect without a restart; the ones that do not are
marked in the tables below.

### 7.1 The five limits

These are the settings the playbook is about.

| Setting | Scope | Default | Where it is covered |
|---|---|---|---|
| `history.persistenceMaxQPS` | one history pod | **9000** | [2.1](#21-the-five-settings), [2.5](#25-a-global-limit-replaces-the-per-pod-one), [2.6](#26-what-each-setting-does-when-set-to-0) |
| `history.persistenceGlobalMaxQPS` | whole cluster | 0 | [2.5](#25-a-global-limit-replaces-the-per-pod-one) — above `0` it replaces the per-pod limit, weighted by shards owned |
| `history.persistenceNamespaceMaxQPS` | one namespace, on one pod | 0 | [2.4](#24-do-not-set-the-per-namespace-limit-equal-to-the-per-pod-limit) — do not set it equal to the pod limit |
| `history.persistenceGlobalNamespaceMaxQPS` | one namespace, whole cluster | 0 | [2.6](#26-what-each-setting-does-when-set-to-0) |
| `history.persistencePerShardNamespaceMaxQPS` | one namespace, on one shard | 0 | [2.1](#21-the-five-settings), [2.6](#26-what-each-setting-does-when-set-to-0) |

**`0` does not mean "off" on most of these** — see
[what each setting does when set to `0`](#26-what-each-setting-does-when-set-to-0).
Setting `history.persistenceMaxQPS` to `0` switches off all three checks entirely.

### 7.2 How the limiter behaves

| Setting | Default | What it does |
|---|---|---|
| `system.persistenceQPSBurstRatio` | 1.0 | Bucket size is the rate multiplied by this — one second's worth at the default. **Applies to all four services.** Raising it was measured to give no benefit under sustained demand: [3.2](#32-there-is-one-second-of-burst-room-and-that-is-all) |
| `system.operatorRPSRatio` | 0.2 | Operator API calls get this share of the rate rather than the whole of it: [3.1](#31-seven-buckets-one-per-priority) |

### 7.3 Adaptive backoff

One struct-valued setting, `history.persistenceDynamicRateLimitingParams`. Your YAML is **merged
over** these defaults, so you only need to set the fields you are changing. Covered in
[check 2](#check-2--was-the-database-struggling).

| Field | Default | Notes |
|---|---|---|
| `enabled` | **false** | On its own, `true` changes nothing — you must also set a threshold |
| `latencyThreshold` | **0.0** | Milliseconds, against the pod's **average** persistence latency. `0.0` means never back off on latency |
| `errorThreshold` | **0.0** | `0.0` means never back off on errors |
| `refreshInterval` | 10s | How often the limiter re-reads the health signals |
| `rateBackoffStepSize` | 0.3 | How much it cuts by per step |
| `rateIncreaseStepSize` | 0.1 | How much it recovers by per step |
| `rateMultiMin` | **0.8** | The floor. At the default it can only ever cut the rate by 20% |
| `rateMultiMax` | 1.0 | The ceiling. It can never exceed the limit you configured |

**Version note.** The setting has existed since **v1.21.0**, but `rateMultiMin` and
`rateMultiMax` only became configurable in **v1.23.0** — before that the floor was hardcoded at
`0.1` — and the `0.8` default arrived in **v1.25.0**. The warning about a 20-per-cent-only cut,
and the recommendation to set `0.2`, apply to **v1.25.0 and later**.

**It depends on two other settings being on.** Both default to `true`; with either off, the average
latency reads as `0` and backoff can never fire.

| Setting | Default | Notes |
|---|---|---|
| `system.persistenceHealthSignalMetricsEnabled` | true | **Read once at pod startup — needs a restart to change.** |
| `system.persistenceHealthSignalAggregationEnabled` | true | **Read once at pod startup — needs a restart to change.** Confirmed by measurement: setting it `false` on a running pod changed nothing. |
| `system.persistenceHealthSignalWindowSize` | 10s | The window the average is taken over. Also read once at startup. |

### 7.4 Mentioned here, but out of scope

These come up because they are easy to confuse with the five limits, or because they explain
something you will see. The playbook does not tell you to change any of them.

| Setting | Default | Why it appears |
|---|---|---|
| `system.visibilityPersistenceMaxReadQPS` | 9000 | Visibility has its own separate limiter that returns the **same error**. Raising `history.persistenceMaxQPS` does nothing for it |
| `system.visibilityPersistenceMaxWriteQPS` | 9000 | as above |
| `history.taskSchedulerEnableRateLimiter` | **false** | Task processing is not paced at all by default: [3.4](#34-a-note-on-task-processing) |
| `history.taskSchedulerEnableRateLimiterShadowMode` | **true** | Even once enabled it measures without holding anything back |
| `history.taskSchedulerMaxQPS` | 0 | At `0` it falls back to the persistence limit — a number in the wrong unit |
| `history.taskSchedulerNamespaceMaxQPS` | 0 | as above |
| `history.<queue>ProcessorMaxPollHostRPS` | 0 | At `0`, queue readers size their polling from `history.persistenceMaxQPS` — x 0.30 for transfer, timer and outbound, x 0.15 for visibility and archival: [2.5](#25-a-global-limit-replaces-the-per-pod-one) |
| `history.TaskDLQUnexpectedErrorAttempts` | 70 | Roughly an hour of database timeouts before a task is dead-lettered. Throttling never counts toward it: [1.1](#11-what-happens-when-a-qps-limit-is-reached) |
| `history.cacheNonUserContextLockTimeout` | 500ms | How long background work waits for a workflow lock before raising `BusyWorkflow`. **Requires a restart** |

### 7.5 Fixed behaviour — not configurable

| What | Value |
|---|---|
| Retry of a throttled database call | **none.** The rejection goes straight back to the caller |
| History task reschedule after throttling | starts at 3s, grows 1.5x, caps at 5 minutes, **no attempt limit and no overall time limit** |
| Number of priority buckets per limit | 7 |
| Order the three checks run in | per-shard, then per-namespace, then per-pod — stopping at the first rejection |
| Per-shard RPS metric window | 30 seconds |
