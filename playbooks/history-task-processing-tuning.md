# History Task Processing — Tuning and Troubleshooting Playbook

The workflow code your developers write creates work for the server. Starting a workflow, scheduling
an activity, sleeping on a timer, a child workflow finishing — each of those turns into a **history
task**, and the history service saves it in the database. A saved task then has to be **processed**,
and processing it is what moves that workflow on to its next step.

**Task processing plays a big part in how fast executions move forward.**

It happens in three parts. Each has its own settings, and — just as important when you are tuning
them — each lives somewhere different:

| Part | What it does | Where it runs |
|---|---|---|
| **Loading** | reads saved tasks out of the database into memory | **per history shard**, with a separate cap for the whole history pod |
| **Scheduling** | decides how many of those tasks are allowed to start right now | **once per history pod**, shared by every shard that pod owns |
| **Executing** | runs the task — many at the same time, each one reading and updating a workflow in the database | **once per history pod**, in a fixed set of internal task slots |

Task processing is durable: tasks are written to the database, then read back and processed. **So
task processing itself produces database operations.**

Those internal task slots are **inside the history service** — they are not SDK workers, and they
have nothing to do with the worker service. Nothing you configure on your workers changes them.

More work means more tasks, and more tasks mean more database operations. Task processing is also
shared: the same loading, scheduling and executing on a history pod serve **every namespace** whose
workflows sit on that pod's shards, so one busy namespace uses capacity the others need too.

[History Persistence QPS Limits](./history-persistence-qps-limits.md) covers the limits that cap
those database operations. **This playbook is about the other side: tuning task processing so the
work fits inside them** — so that fewer tasks are rejected and retried while they are being
processed.

**The two go together, and getting either side wrong shows up in the other:**

- **The persistence QPS limits set too high.** Task processing can then push the database past what
  it can actually do. Calls start timing out rather than being cleanly refused — and unlike being
  refused, timeouts count against a task's retry budget, so a task that keeps hitting them ends up
  in the dead-letter queue waiting for an operator. See
  [what happens when a QPS limit is reached](./history-persistence-qps-limits.md#11-what-happens-when-a-qps-limit-is-reached).
- **The persistence QPS limits set correctly, but the task processing limits left untuned.** Now the
  opposite: tasks are started that had no chance of finishing, are refused part-way through, and
  retry. Nothing is lost and nothing is dead-lettered, but the work is done twice and executions are
  slower than they need to be.

**Audience**

Operators of self-hosted Temporal clusters.

**What this playbook covers**

- How history task processing works, and where each part runs.
- How to tell whether task processing is what is hurting you.
- How to recognise the write-reject loop — refused writes that make the pod re-read state it
  already had, which in turn makes more writes fail.
- How to find the real cause of a surge in database reads.
- How to set the task scheduler's rate limits, and how to pick the numbers.
- How to turn them on safely on a running cluster, and how to tell when you have set them too low.
- How the server already decides which tasks go first, and what changes when you run more than one
  cluster.
- How to tune how fast tasks are read out of the database.
- Which settings to leave alone.
- How to be alerted when tasks stop completing and start retrying.

**What this playbook does not cover**

- The persistence QPS limits themselves — those are
  [History Persistence QPS Limits](./history-persistence-qps-limits.md).
- Steady-state background database load: checkpoint intervals and acknowledgement intervals.
- Worker provisioning — too few SDK workers creates a backlog of its own.

**Applies to** the history service, on any persistence store. Nothing here is specific to Cassandra
or to SQL.

---

## Contents

1. [The defaults, and when they stop being enough](#1-the-defaults-and-when-they-stop-being-enough)
    - [1.1 Why tuning task processing is worth doing](#11-why-tuning-task-processing-is-worth-doing)
    - [1.2 What an untuned cluster looks like](#12-what-an-untuned-cluster-looks-like)
2. [How task processing works](#2-how-task-processing-works)
    - [2.1 Loading — the queue readers](#21-loading--the-queue-readers)
    - [2.2 Scheduling — the task scheduler](#22-scheduling--the-task-scheduler)
    - [2.3 Executing — the task slots](#23-executing--the-task-slots)
    - [2.4 When loaded tasks build up in memory](#24-when-loaded-tasks-build-up-in-memory)
    - [2.5 Every control point in one place](#25-every-control-point-in-one-place)
3. [What happens when the persistence limit refuses a task's write](#3-what-happens-when-the-persistence-limit-refuses-a-tasks-write)
    - [3.1 The sequence that creates the loop](#31-the-sequence-that-creates-the-loop)
    - [3.2 Why the loop does not settle by itself](#32-why-the-loop-does-not-settle-by-itself)
    - [3.3 The one number that identifies the loop](#33-the-one-number-that-identifies-the-loop)
    - [3.4 What the loop does to client requests](#34-what-the-loop-does-to-client-requests)
    - [3.5 Before you change anything](#35-before-you-change-anything)
4. [The task scheduler's rate limits](#4-the-task-schedulers-rate-limits)
    - [4.1 What each setting controls](#41-what-each-setting-controls)
    - [4.2 Two traps that leave these settings doing nothing](#42-two-traps-that-leave-these-settings-doing-nothing)
    - [4.3 How to see what is really in force](#43-how-to-see-what-is-really-in-force)
5. [Sizing the limits and turning them on](#5-sizing-the-limits-and-turning-them-on)
    - [5.1 Step 1 — turn the limiter on, with shadow mode left on](#51-step-1--turn-the-limiter-on-with-shadow-mode-left-on)
    - [5.2 Step 2 — work out your numbers](#52-step-2--work-out-your-numbers)
    - [5.3 Step 3 — set the numbers, then take it out of shadow mode](#53-step-3--set-the-numbers-then-take-it-out-of-shadow-mode)
    - [5.4 Step 4 — adjust](#54-step-4--adjust)
6. [How the scheduler decides what goes first](#6-how-the-scheduler-decides-what-goes-first)
    - [6.1 Three priority classes, fixed by task type](#61-three-priority-classes-fixed-by-task-type)
    - [6.2 What you get without configuring anything](#62-what-you-get-without-configuring-anything)
    - [6.3 If you run more than one cluster](#63-if-you-run-more-than-one-cluster)
    - [6.4 Changing the priority weights](#64-changing-the-priority-weights)
7. [Tuning how fast tasks are loaded](#7-tuning-how-fast-tasks-are-loaded)
    - [7.1 Is loading taking more of the budget than it needs?](#71-is-loading-taking-more-of-the-budget-than-it-needs)
    - [7.2 What is your pod's real poll ceiling?](#72-what-is-your-pods-real-poll-ceiling)
    - [7.3 Choosing a poll ceiling, and knowing if you set it too low](#73-choosing-a-poll-ceiling-and-knowing-if-you-set-it-too-low)
8. [Alerting on the write-reject loop](#8-alerting-on-the-write-reject-loop)
    - [8.1 Alert 87 — History Write-Reject Loop](#81-alert-87--history-write-reject-loop)
    - [8.2 Adjusting alert 87 for your cluster](#82-adjusting-alert-87-for-your-cluster)
    - [8.3 Why there is no alert on task scheduler throttling](#83-why-there-is-no-alert-on-task-scheduler-throttling)
    - [8.4 The one thing no alert can tell you: whether your limits are actually on](#84-the-one-thing-no-alert-can-tell-you-whether-your-limits-are-actually-on)
9. [Every setting named in this playbook](#9-every-setting-named-in-this-playbook)
    - [9.1 The task scheduler — the settings this playbook is about](#91-the-task-scheduler--the-settings-this-playbook-is-about)
    - [9.2 Loading — tune after scheduling, not before](#92-loading--tune-after-scheduling-not-before)
    - [9.3 Read these, but do not change them](#93-read-these-but-do-not-change-them)
    - [9.4 Named here, but belonging to another playbook](#94-named-here-but-belonging-to-another-playbook)

---

## 1. The defaults, and when they stop being enough

Of the three parts, only one comes with no limit at all:

| Part | Limited by default? |
|---|---|
| **Loading** | **Yes** — a poll rate per history shard, plus a ceiling for the whole pod worked out from the persistence limit. |
| **Scheduling** | **No.** The settings exist — they are what this playbook is mostly about — but they are turned off. |
| **Executing** | Only by the fixed number of task slots, which caps how many tasks run **at once** but not how fast new ones start. |

So on a cluster nobody has configured, what decides how much work reaches the database is the number
of task slots, and the only thing that refuses any of it is the persistence QPS limits.

**For a small or steady workload that is fine**, and plenty of clusters never need anything more. It
stops being fine when more work arrives than the database is allowed to take — a burst of workflow
starts, a backlog draining after an outage or a deploy, several busy namespaces sharing a cluster.
Then tasks start that cannot finish. A task that has already read a workflow from the database, and
then has its write refused, has to begin again from the read. **The same work gets done more than
once**, which means the backlog drains more slowly and the repeated reads take database budget away
from everything else — including the calls your users are waiting on.

### 1.1 Why tuning task processing is worth doing

Tuning does not give your database more capacity. What it does is stop the same work being done
twice, so the capacity you already have goes further.

| | Default: no limit on scheduling | What tuning aims for |
|---|---|---|
| **Tasks** | start as soon as a slot is free, whether or not there is database budget to finish them | start at a rate where a task that starts can finish |
| **Database calls** | some are refused, and the work behind them is repeated | fewer refused calls, so more of the budget does useful work |
| **Backlog** | drains slowly, because the same tasks keep coming back | drains steadily, and usually faster than before |
| **Calls your users make** | compete with the repeated work | keep their share of the database budget |

The right-hand column is what you are aiming at, not a guarantee — the numbers have to be sized for
your cluster. Choosing the numbers is
[section 5](#5-sizing-the-limits-and-turning-them-on); telling when you have gone too far is
[5.4](#54-step-4--adjust).

**Sized correctly, nothing is given up.** A task that waits a moment before starting then finishes on
its first attempt, instead of starting straight away and being refused. The database receives work at
a rate it can absorb, and the same tasks stop coming back. Sized too low it does become a trade — the
backlog drains more slowly than it needs to — which is why the numbers matter, and why
[5.4](#54-step-4--adjust) covers how to check.

**Why this is not only a background problem.** Refused tasks are retried, and the retrying continues
for as long as the condition lasts. A history pod busy retrying thousands of tasks has less capacity
left for the requests your clients send, so **starting workflows and sending signals get slower too**.
That is the state where a whole cluster looks unwell, when what was actually being refused was task
processing.

### 1.2 What an untuned cluster looks like

Nothing warns you that scheduling has no limit on it, so the connection between what you see and a
task processing setting is rarely the first one anyone makes. These are the signs, and where to
check each one:

- **A read operation is taking most of your rejected database calls.** On
  **[Rejected Database Calls by Operation and Scope](../observability/dashboards/server/temporal-server-readme.md#3-persistence-requests-latencies-and-errors)**,
  `GetWorkflowExecution` dominates — rather than the `Get*Tasks` operations that load work.
- **Clients are being refused while a backlog is worked through.** On
  **[Resource Exhausted with Cause](../observability/dashboards/server/temporal-server-readme.md#6-throttling-and-limits)**, `StartWorkflowExecution` or
  `SignalWorkflowExecution` appear, and they clear once the backlog drains.
- **The backlog is not draining at the rate you would expect.** On
  **[Immediate Queue Lag per Pod](../observability/dashboards/server/temporal-server-readme.md#9-shard-queue-health)** and
  **[Scheduled Queue Lag per Pod](../observability/dashboards/server/temporal-server-readme.md#9-shard-queue-health)**, lag stays high or climbs.
- **You have already raised the persistence limits** as far as your database can take — see
  [History Persistence QPS Limits](./history-persistence-qps-limits.md) — and the work still does
  not fit.
- **A burst of work in one namespace is delaying task processing for the others.** Scheduling and
  executing are shared per history pod, so a namespace that suddenly creates a lot of tasks takes
  capacity the other namespaces need, and their workflows move forward more slowly. Compare the
  namespaces on **[History Task Throughput](../observability/dashboards/server/temporal-server-readme.md#9-shard-queue-health)** — one line far above the
  others is the namespace taking the capacity.

**If you have run into any of these, the rest of this playbook should help** — and it is worth a
read even if you have not: what is happening underneath, how to work out which one you are looking
at, what to change, and how to be told if it happens again.

**One exception, and it is worth ruling out first.** On
**[Rejected Database Calls by Operation and Scope](../observability/dashboards/server/temporal-server-readme.md#3-persistence-requests-latencies-and-errors)**,
look at *which* operations are being rejected. If they are `GetTransferTasks`, `GetTimerTasks` or
`GetVisibilityTasks`, that is task **loading** being refused rather than task processing. **This
playbook covers that too**, but as a separate, later step — loading is tuned after scheduling, and
the settings are different.

Refused loads are cheaper than refused writes: the task has not read anything yet, so no work is
thrown away and no retries build up. They do still delay work, though, so check whether it is
costing you anything:

- **[Resource Exhausted with Cause](../observability/dashboards/server/temporal-server-readme.md#6-throttling-and-limits)** — are any client operations
  failing?
- **[Immediate Queue Lag per Pod](../observability/dashboards/server/temporal-server-readme.md#9-shard-queue-health)** and
  **[Scheduled Queue Lag per Pod](../observability/dashboards/server/temporal-server-readme.md#9-shard-queue-health)** — is lag steady or falling, or
  climbing?

**Steady lag and no client operations** means the delay is being absorbed — nothing needs changing.
**Climbing lag** means tasks are being loaded more slowly than they are being created, which turns
into timers firing late and work dispatched late — that is worth acting on, in the loading step of
this playbook.

---

## 2. How task processing works

This section shows how the task processing described in
[section 1](#1-the-defaults-and-when-they-stop-being-enough) — the thing the rest of this
playbook tunes — actually works: the same three parts, loading, scheduling and executing, in enough
detail to see what each setting acts on. The aim is that by the end of it there is no doubt about
which part of task processing each setting later in this playbook is changing.

**Where the tasks come from.** The history service writes them itself. Whenever it handles an API
call or finishes another task, it saves the new workflow state and the tasks that follow from it in
the same database write — starting a workflow writes a transfer task, a workflow sleeping writes a
timer task, and so on. Those tasks then sit in the database until they are loaded back out.

**What each part does, and what it costs:**

| Part | What happens | Database calls it makes |
|---|---|---|
| **Loading** | batches of waiting tasks are read out of the database into memory | `GetTransferTasks`, `GetTimerTasks`, `GetVisibilityTasks` — written as `Get*Tasks` from here on |
| **Scheduling** | tasks already in memory wait their turn and are handed out to run | **none** — this part does not touch the database |
| **Executing** | the task runs: the workflow is read, the step is carried out, the new state is written back | `GetWorkflowExecution`, `UpdateWorkflowExecution` |

**Every task category has its own loading, scheduling and executing.** Five categories work this
way:

| Category | Also called | What it carries |
|---|---|---|
| **transfer** | the immediate queue | work to hand out now — dispatching workflow and activity tasks, starting child workflows, telling a parent its child finished |
| **timer** | the scheduled queue | work with a time attached — workflow and activity timeouts, and the timers a workflow sleeps on |
| **visibility** | | keeping the visibility store in step with what the workflow is doing |
| **outbound** | | work the cluster sends somewhere outside itself — Nexus operations and completion callbacks. Each task carries the destination it is for |
| **archival** | | moving finished workflow histories to archival storage |

They share nothing: each has its own readers, its own scheduler, its own task slots, and its own
settings. Where a setting below is written `<queue>Processor…`, there is one of it per category.

**Two other categories exist and are not part of this**, which is worth knowing if you see them
elsewhere:

- **replication** tasks are their own category with their own machinery and their own
  `history.replication…` settings. Nothing in this playbook applies to them.
- **memory timer** tasks are held in memory and never written to the database, so there is nothing
  to load and nothing to tune.

### 2.1 Loading — the queue readers

The reader pulls batches of tasks out of the database task tables and holds them in memory. It is
driven by work, not by a clock: as long as a batch comes back full, it immediately asks for another.
So under a backlog it loads continuously, and the only things pacing it are its own poll limits.

| Setting | Default | What it does |
|---|---|---|
| `history.<queue>ProcessorMaxPollRPS` | **20** | Poll rate for one shard. |
| `history.<queue>ProcessorMaxPollHostRPS` | **0** = off | Poll rate for the whole pod, across every shard it owns. |
| `history.<queue>ProcessorMaxPollInterval` | **1 minute** (transfer, visibility, outbound), **5 minutes** (timer, archival) | How often an idle queue re-checks anyway, with nothing to do. |

**With `history.<queue>ProcessorMaxPollHostRPS` left at `0` — the default — a pod's real poll
ceiling is worked out from the persistence setting instead.** It becomes
`history.persistenceMaxQPS × 0.30` for transfer, timer and outbound, and `× 0.15` for visibility and
archival. And if `history.persistenceMaxQPS` is itself `0`, the ceiling falls back to a hard-coded
**100,000 per second**, which is no ceiling at all. That is the same trap described in
[what each setting does when set to `0`](./history-persistence-qps-limits.md#26-what-each-setting-does-when-set-to-0) —
worth knowing here because it is the *reader* those numbers are sizing.

**Even with nothing to do, each queue keeps re-checking its shards.** New work does not have to be
discovered: the history pod that writes the tasks tells its own reader, in memory, as soon as the
write is done. The re-checking is a backstop for what that does not cover, bounded by
`history.<queue>ProcessorMaxPollInterval` — **1 minute** for transfer, visibility and outbound,
**5 minutes** for timer and archival. So a completely idle cluster still reads:

```
idle Get*Tasks per second  ≈  shards  ×  queues  ÷  poll interval
```

**Work it out for your own shard count before assuming a `Get*Tasks` rate is a problem.** On a
cluster with **2048 shards** — a common choice — with no work at all:

| | |
|---|---|
| three one-minute queues (transfer, visibility, outbound) | `2048 × 3 ÷ 60` ≈ **102 reads/s** |
| two five-minute queues (timer, archival) | `2048 × 2 ÷ 300` ≈ **14 reads/s** |
| **total, cluster-wide, idle** | ≈ **116 reads/s** |

Divided across the pods that own those shards — four history pods, say — that is about **29 reads a
second each**, before a single workflow runs.

**That floor moves with the interval, not with the poll rates** — and only for shards that are
genuinely idle, since a shard with pending work reads as often as that work requires, sooner than
the interval. Whether raising it is worth doing is a tuning decision, and
[7.3](#73-choosing-a-poll-ceiling-and-knowing-if-you-set-it-too-low) is where this playbook makes
it.

### 2.2 Scheduling — the task scheduler

The scheduler decides which of the loaded tasks start now. It is the only part with a rate limit
meant for exactly that.

It does not treat all tasks as one stream. Internally it keeps a **separate channel per namespace
per priority**, and serves them in weighted rotation. Two consequences worth knowing before you tune
anything:

- **Namespaces take turns.** Tasks are picked up in rotation across namespaces rather than in the
  order they arrived, so one namespace with an enormous backlog cannot take every slot. This is
  about the **order waiting tasks are picked up**, not about running one namespace at a time — tasks
  from many namespaces are running side by side in the slots.
- **Priority decides how often a channel gets its turn.** Active-namespace tasks default to weights
  of **10** for high priority, **9** for low, and **1** for preemptable, so a high-priority channel
  is visited ten times as often as a preemptable one. Standby-namespace tasks all sit at **1**,
  which is to say a standby cluster's tasks give way to an active one's.

The scheduler's rate limits are covered later in this playbook. What matters here is that **they
are off by default**, and that with them off the scheduler hands tasks out as fast as the slots
free up.

### 2.3 Executing — the task slots

Tasks the scheduler hands out go to a fixed set of internal task slots inside the history service —
again, not SDK workers. The set is **per history pod, per queue** — not per shard — and its size is
`history.<queue>ProcessorSchedulerWorkerCount`, which defaults to **512** for transfer, timer,
visibility and archival.

**Outbound is sized differently**, because its work is grouped by where it is going: concurrency is
`history.outboundQueue.groupLimiter.concurrency` (default **100**) per destination, with its own
rate cap in `history.outboundQueue.hostScheduler.maxTaskRPS` (default **100/s**).

**That is a ceiling, not a target.** It does not mean a quiet cluster runs 512 tasks at a time — it
means nothing stops it from doing so. With transfer, timer and visibility each having their own set,
a single history pod can be running roughly **1,500 tasks at once**.

**Each of those running tasks is a potential pair of database calls**, and both count against your
persistence limits:

- **The read is conditional.** A task needs the workflow's state, but if that state is already in
  the pod's cache it is used directly and there is no database call. Only when it is not cached does
  the task issue a **`GetWorkflowExecution`**. Remember that one — it is where this playbook's main
  failure gets its shape.
- **The write is not.** When the task changes something, the new state goes to the database as an
  **`UpdateWorkflowExecution`**.

**This is how a history pod with no scheduling limit can overload a database that is perfectly
adequate for the actual workload.** The total amount of work is not the problem — the rate is. When
a burst of tasks is loaded, nothing decides **how many of them start per second**: they start as
fast as slots come free, up to 512 per queue, and every one of them can put a read and a write to
the database at the same moment.

**When every slot is busy, the task cannot start.** Nothing is dropped and nothing waits in a
queue: it is put back and retried from memory a moment later. It never reached the database,
so no work is wasted — but it stays on the pod's books as a **pending task**, and that number is
what the next section is about.

### 2.4 When loaded tasks build up in memory

A task that has been loaded but has not finished is **pending**. It is sitting in the pod's memory,
either waiting for a free slot or waiting to be retried after it could not start. The pod counts
them, per queue.

**What makes the count rise:**

- **More work arriving than the slots can finish** — a burst of workflow starts, or a backlog
  draining after workers were down or a deploy paused things.
- **Tasks not finishing**, because the database calls they need are being refused or are slow, so
  they keep retrying. This is the situation this playbook is mostly about.
- **Tasks waiting their turn**, once you have set scheduling limits. A higher pending count after
  tuning is expected — that is the work waiting instead of starting and failing.

**Pending tasks are not at risk.** Loading a task does not remove it from the database — it stays
there until the queue has finished with it and records its progress past it. What is in memory is a
working copy. If the pod restarts, or the shard moves to another pod, the memory is discarded and
the new owner reads those tasks again from the last recorded position. Nothing is lost; the work is
just done later.

When the count gets high the pod reacts by itself, in two steps. **You do not set either of these
in normal operation** — they are here because they explain something you will see on the dashboard:

| Setting | Default | What happens when the pending count crosses it |
|---|---|---|
| `history.queuePendingTaskCriticalCount` | **9000** | The pod starts **unloading** tasks it has already loaded, until the count is back to about 7,200 — at most one round every 10 seconds. Unloaded tasks have to be read from the database again later. |
| `history.queuePendingTasksMaxCount` | **10000** | The reader stops loading and pauses for `history.<queue>ProcessorPollBackoffInterval` (**5 seconds** by default), then tries again. |

**What this costs you is extra database reads.** A task that is unloaded has to be read out of the
database again later, so the same task can be read several times before it finally runs. On the
dashboard that looks like a `Get*Tasks` rate far higher than the amount of work actually being
completed.

**It is easy to read that as the loading being at fault**, and to reach for the poll rate settings.
It is usually the wrong move. The pod is re-reading *because* tasks are not finishing — the reader
is never told that tasks cannot start, so it keeps loading, and the pod keeps throwing the excess
away. Fix what is stopping tasks from finishing, which is the scheduling side, and the pending count
falls, the unloading stops, and the extra reads disappear on their own.

**That is why this playbook tunes scheduling first and loading second.**

### 2.5 Every control point in one place

That is the whole of task processing: three parts, and the pod's own reaction when work backs up
between them. Before moving on, here is every point where work can be slowed down, and which of
them are yours to set.

| Where | What limits it | Settings | Yours to set? |
|---|---|---|---|
| **Loading** | poll rate per shard, poll rate per pod, and how often an idle queue re-checks | `history.<queue>ProcessorMaxPollRPS`<br>`history.<queue>ProcessorMaxPollHostRPS`<br>`history.<queue>ProcessorMaxPollInterval` | **Yes** — worth setting once scheduling is right |
| **Tasks waiting in memory** | how many loaded tasks the pod will hold before it starts unloading some | `history.queuePendingTaskCriticalCount`<br>`history.queuePendingTasksMaxCount` | **No** — the pod handles these itself |
| **Scheduling** | the rate at which tasks may start, per pod and per namespace, plus the priority weights | the `history.taskScheduler…` settings | **Yes — nothing is set by default, and this is the gap** |
| **Executing** | the number of task slots | `history.<queue>ProcessorSchedulerWorkerCount` | Only when tasks are waiting for a **slot** rather than for the database |
| **The database calls a running task makes** | the persistence QPS limits | the `history.persistence…` settings | Yes — in [History Persistence QPS Limits](./history-persistence-qps-limits.md) |

**The gap is scheduling.** Loading comes with limits out of the box, executing has its fixed number
of slots, and the database calls are covered by your persistence limits. Scheduling — the part that
decides how fast work is sent at the database — has nothing set at all.

**So the rest of this playbook tunes scheduling first, and loading after it.** That order is not
arbitrary: while tasks cannot finish, capping how fast they are loaded changes nothing, and the
extra reads go away on their own once tasks start completing again
([2.4](#24-when-loaded-tasks-build-up-in-memory)).

**Each of the two gets its own section**, with the settings to change, how to choose the numbers,
how to roll them out on a running cluster, and how to tell whether they are right — including when
you have set them too low.

**Before that, though, the next two sections are about the failure itself:** what actually goes
wrong when nothing limits scheduling, and how to confirm that is what you are looking at rather
than something that resembles it on a dashboard. The numbers you will choose later only make sense
once you have seen it.

---

## 3. What happens when the persistence limit refuses a task's write

Section 2 ended on the gap: nothing paces scheduling, so when a burst of work arrives, tasks start
faster than the database is allowed to take them. This section follows what happens next.

It is worth following closely, because the outcome is not what most people expect. A limit refusing
some calls sounds like it should slow things down a little. What actually happens is that **one
refused write creates more work than it prevents**, and the cluster gets busier the longer it goes
on. That is the problem the rest of this playbook is written to prevent.

### 3.1 The sequence that creates the loop

Take a task whose workflow is already in the pod's cache, so it needs no read to get going:

1. The task does its work and writes the result back: `UpdateWorkflowExecution`.
2. **The write is refused.** The persistence limit is at its ceiling.
3. The cached copy is now out of step with what is in the database, so the pod **discards it**.
4. The task tries again — and now it has to read the workflow back out of the database first, with
   `GetWorkflowExecution`. (Temporal calls this the workflow's *mutable state*; the metrics label it
   `cache_type="mutablestate"`.)
5. **So the second attempt costs a read and a write, where the first cost only a write** — and both
   come out of the same budget that refused it.

> ### That read costs more than one call
>
> `GetWorkflowExecution` counts as a single operation to the persistence limit and to every metric
> in this playbook. Underneath, **on a SQL store it is nine separate queries, run one after
> another** — the execution row, then activity, timer, child-workflow, cancel and signal records,
> buffered events, and the rest. On Cassandra it is one read, because those records sit in the same
> partition.
>
> **So on SQL, every re-read that a refused write causes is nine round trips to the database, while
> the limiter counts it as one.** The load a loop puts on a SQL database is far larger than the
> rejection numbers suggest.

**A refused write does not simply delay one task. It turns a task that needed no reads into one
that needs a read on every attempt** — and the task keeps attempting for as long as it takes.

**This is the write-reject loop:** a write is refused, the cached state is discarded, the retry
reads it back, and that read makes the refusals more likely for everyone else. It is called that
throughout the rest of this playbook, and on the dashboard panel that detects it. The loop is the
shape to recognise — refusals creating reads, reads crowding out writes, and round again.

### 3.2 Why the loop does not settle by itself

- **Throttled tasks retry indefinitely.** Being refused by a limit is not counted as a task
  failure. The server keeps a separate count of unexpected errors, and that is the count that can
  send a task to the dead-letter queue — a refusal never adds to it. So a throttled task is never
  dead-lettered and never gives up; it simply keeps coming back.
- **Every retry adds a read that was not needed before**, so the load grows while the work
  completed stays flat.
- **The pending count rises**, which makes the pod unload tasks it had already loaded and read them
  again later ([2.4](#24-when-loaded-tasks-build-up-in-memory)) — more reads still.

Each of those makes the others worse. That is why this does not look like a limit gently slowing
things down — it looks like a cluster working hard and getting nowhere: database calls climbing,
work being retried over and over, and workflows still not moving on.

### 3.3 The one number that identifies the loop

The signature is that **cached state is being thrown away faster than it is genuinely missing**.
Open **[Write-Reject Loop Indicator (cleared / cache miss)](../observability/dashboards/server/temporal-server-readme.md#3-persistence-requests-latencies-and-errors)**
and read the ratio.

| Ratio | What it means |
|---|---|
| **below about 1** | Normal. Reads are ordinary cache misses. |
| **around 1 to 2** | Some refused writes, not a runaway. |
| **above about 5** | The loop is running. |

**Measured on a test cluster**, driving a 2,400-workflow backlog into a budget far too small for it,
with no scheduling limits set:

| | |
|---|---|
| loop indicator | climbed **5.1 → 6.7 → 9.3 → 10.9**, peaking at **18.2** |
| state cleared | **85-108 per second** |
| genuine cache misses | **5 per second** |
| refused calls | up to **383 per second** |
| busiest refused operation | **`GetWorkflowExecution` at 102.6/s**, ahead of everything else |

**Two numbers in that table stop you fixing the wrong thing.**

- **Cache misses were tiny while clearing was enormous** — 5 a second against 85 to 108. The usual
  explanation for a read spike is a cache that is too small, and that would show up as *high* cache
  misses. It did not. The state was not being evicted; it was being thrown away because writes were
  refused. Resizing the cache would have changed nothing.
- **The reads dominate the rejections even though the writes are what is being refused.**
  `GetWorkflowExecution` sits at the top of the list at 102.6/s and `UpdateWorkflowExecution` well
  below it at 8/s, so it is tempting to treat the reads as the problem. They are the consequence;
  the refused writes are the cause. The reason the gap is so wide is that **once the state has been
  discarded, every attempt begins with a read — and an attempt whose read is refused never gets as
  far as its write.**

**This is not a SQL effect.** The limit counts one call per operation whatever the store is sitting
underneath it, so the same lopsided pattern appears on Cassandra. What differs between stores is
the real work behind each counted read, which is the point made above: nine queries on SQL, one on
Cassandra.

> **The ratio only means anything when both numbers are real.** On a quiet cluster almost nothing
> is cleared and almost nothing is missed, so the panel can read `0`, or show nothing at all. Look
> at the two lines themselves before trusting the ratio between them.

### 3.4 What the loop does to client requests

**Start with what does *not* happen.** The persistence limit sorts calls into priority buckets, and
the calls behind `StartWorkflowExecution`, `SignalWorkflowExecution` and the rest sit in a higher
bucket than background task processing. A high-priority call takes a token from its own bucket and
from the lower ones; a background call cannot touch the higher bucket at all. **So a loop in task
processing does not drain the budget your client calls depend on.** That protection is real, and on
a small cluster a loop can run without any client noticing.

What does reach clients is less direct:

- **Executions stop moving.** The tasks that are not completing are the ones that advance workflows
  — dispatching a workflow task, firing a timer, telling a parent that its child finished. No error
  is returned to anyone, but work sits still. This is the effect users report first, usually as
  "workflows are stuck".
- **The pod is busy with work that cannot finish.** Retrying tasks occupy the slots and the pod's
  capacity, so everything that pod does, including serving the history side of an API call, gets
  slower.
- **Per-workflow limits fill up while a workflow sits still, and some of those do return errors.**
  These limits count things waiting for a workflow task to run, so they drain only when task
  processing is working:
    - **Workflow Updates:** at most **10** may be in flight for one workflow
      (`history.maxInFlightUpdates`). An update finishes only when a workflow task runs, so a
      stalled workflow stops draining them and further updates come back to the client as
      `ResourceExhausted`, cause `ConcurrentLimit`. There is a size limit on the same registry.
    - **Signals** are not refused — they are buffered. But the buffer holds at most **100 events**
      or **2 MB** (`history.maximumBufferedEventsBatch`,
      `history.maximumBufferedEventsSizeInBytes`). Crossing either makes the server fail the
      in-flight workflow task and schedule a fresh one to flush the buffer — which adds task work
      to a pod that is already not keeping up.
- **The persistence limit itself refuses client calls only when client traffic is over the
  limit.** That is separate from the loop and easy to confuse with it. In the same test, while
  background reads were refused at 102.6/s, `CreateWorkflowExecution` was also refused at 14.6/s —
  not because the loop took its budget, but because the burst was starting workflows at about 333
  a second against a limit of 300.

**So the honest summary is:** the loop stalls progress and slows the pod down, and whether your
users see errors depends on whether their own traffic fits inside the limit. Both can be happening
at once, which is why the next section is about telling causes apart before changing anything.

### 3.5 Before you change anything

Section 3 has described one failure. Several other things produce a read spike that looks much like
it, and each has a different fix — so it is worth two minutes to confirm which one you have.

**Take one reading you have not taken yet.** Attempts are counted *before* the limit is applied, so
a refused call still shows up as a request. That means the read rate you are reacting to may be far
larger than what the database actually served:

1. Open **[Rejected Database Calls by Operation and Scope](../observability/dashboards/server/temporal-server-readme.md#3-persistence-requests-latencies-and-errors)**
   and note the rate for `GetWorkflowExecution`. That is **attempts**.
2. Open **[Database Calls That Reached the Database](../observability/dashboards/server/temporal-server-readme.md#3-persistence-requests-latencies-and-errors)**
   and note the same operation there. That is **what the store actually served**.
3. Compare them. In the test above the two were **132 a second attempted** against **about 29 a
   second reached** — a factor of four.
4. **From here on, use the number that reached the database** whenever you judge load or work out
   how many database calls a task costs. Sizing from attempts inflates everything that follows.

Then read off which situation you are in:

| What you see | Where you see it | What it means | What to do |
|---|---|---|---|
| Clearing high, cache misses low, ratio **above 5** | **Write-Reject Loop Indicator** | The [write-reject loop](#31-the-sequence-that-creates-the-loop) | **Carry on with this playbook.** The next section is the `history.taskScheduler…` settings |
| **Cache misses high** | **Write-Reject Loop Indicator**, the cache-miss line | Ordinary cache pressure: state is being evicted, not discarded | A different problem. Raise `history.hostLevelCacheMaxSize` (default **128,000** entries, with a byte cap in `history.hostLevelCacheMaxSizeBytes`) |
| Attempts far above what reached the store | **Rejected Database Calls by Operation and Scope** against **Database Calls That Reached the Database** | Part of the rate you are reacting to never touched the store | Nothing to fix. Re-read your numbers using what reached the database |
| Client-facing operations refused | **Resource Exhausted with Cause** | Client traffic is itself over the limit | That is the persistence playbook's decision: [History Persistence QPS Limits](./history-persistence-qps-limits.md) |
| Only `Get*Tasks` refused, nothing stalling | **Rejected Database Calls by Operation and Scope**, with **Immediate / Scheduled Queue Lag per Pod** steady | Loading is being throttled, not task processing | Handled later here, as the second step: the `history.<queue>ProcessorMaxPoll…` settings, after scheduling is right |
| `Get*Tasks` steady while little work is running | **Persistence Requests Total per Operation** | The idle poll floor, not churn | Nothing. Work out `shards × queues ÷ poll interval` first — about **116 a second** on a 2048-shard cluster with no workload |

#### If raising the persistence limit also clears the loop, why tune task processing at all?

Because raising the limit does not remove the wasted work — it buys enough database capacity to
absorb it. The re-reads, the discarded state and the repeated attempts all carry on; there is simply
room for them now. Three consequences follow:

- **You need a bigger database than the workload actually requires**, because a share of its
  capacity is being spent on work that is thrown away. In the test, four out of every five read
  attempts never reached the store.
- **It comes back.** The headroom you just bought is consumed by the next burst, or by ordinary
  growth, and the loop returns at the new ceiling.
- **It is not always available.** When the database is already at its limit there is nothing to
  raise, and this is the only route left.

**Raising the limit is a legitimate response** —
[History Persistence QPS Limits](./history-persistence-qps-limits.md) covers when to, and it was
measured there to clear the loop on a database that had room. Tuning task processing is the other half:
making the work fit what you have, so the capacity goes to tasks that finish.

#### And why not start by bringing the read rate down?

**The temptation.** `history.<queue>ProcessorMaxPollHostRPS` caps how fast a pod reads tasks out of
the database. When reads are the thing you can see being refused, capping reads looks like the fix.

**Why it does not work.** Those reads are a symptom, not the cause. Tasks are not finishing, so the
pod keeps loading and re-loading them ([2.4](#24-when-loaded-tasks-build-up-in-memory)). Capping
the reads makes no task finish any sooner — it only slows down the arrival of work that was already
waiting, and the pending count stays where it is.

> ### Recommendation: leave the poll settings alone for now
>
> Fix scheduling first. The extra reads are created by tasks that cannot finish, so they go away by
> themselves once tasks start completing — measured in this playbook's test as the difference
> between **383 refused calls a second** and **about 1**.
>
> Come back to `history.<queue>ProcessorMaxPollHostRPS` afterwards, as the second step. At that
> point it is capping steady-state reading, which is worth doing, rather than hiding a loop.

---

**Next:** you have confirmed the
[write-reject loop](#31-the-sequence-that-creates-the-loop), and you know the fix is to start fewer
tasks than the database can absorb. The next section is the settings that do that — what each one
controls, which of them are off by default, and the two traps that make them look as though they
are doing nothing.

---

## 4. The task scheduler's rate limits

These are the settings that decide how many tasks are allowed to start. There are seven of them,
and **on a cluster nobody has configured, every one is switched off or at zero.**

### 4.1 What each setting controls

**The four limits** — these are the numbers you set:

| Setting | Default | What it controls |
|---|---|---|
| `history.taskSchedulerMaxQPS` | **`0`** | Tasks per second **one pod** may start, across all namespaces. |
| `history.taskSchedulerNamespaceMaxQPS` | **`0`** | Tasks per second **one pod** may start **for each namespace**. Namespace-scoped: you can give different namespaces different values. |
| `history.taskSchedulerGlobalMaxQPS` | **`0`** | Tasks per second the **whole cluster** may start, across all namespaces. |
| `history.taskSchedulerGlobalNamespaceMaxQPS` | **`0`** | Tasks per second the **whole cluster** may start **for each namespace**. Namespace-scoped: you can give different namespaces different values. |

**What `0` means here is the second trap, below** — it is not "no limit".

**The three switches** — these decide whether and how the four limits apply:

| Setting | Default | What it controls |
|---|---|---|
| `history.taskSchedulerEnableRateLimiter` | **`false`** | The master switch. While it is `false`, none of the four limits above has any effect. |
| `history.taskSchedulerEnableRateLimiterShadowMode` | **`true`** | Measure-only mode. Counts what *would* have been held back, and holds nothing back. |
| `history.taskSchedulerRateLimiterStartupDelay` | **5 seconds** | How long after a pod starts before the limiter applies at all, so a restarting pod is not held back while it is still finding its feet. |

The two per-namespace settings take a value per namespace; the other two are a single value for
the whole cluster. So a busy namespace can be capped without touching the others, while the
all-namespaces limits stay as the ceiling over everything.

The four differ only in scope, and if you have read
[History Persistence QPS Limits](./history-persistence-qps-limits.md) the layout will be familiar,
because the persistence settings use the same one:

| | **Per history pod** | **Whole cluster** |
|---|---|---|
| **All namespaces together** | `taskSchedulerMaxQPS` | `taskSchedulerGlobalMaxQPS` |
| **Per namespace** | `taskSchedulerNamespaceMaxQPS` | `taskSchedulerGlobalNamespaceMaxQPS` |

**When a limit is reached, tasks wait — they are not refused.** The scheduler delays the task and
starts it when there is room. That is the point of moving the pushback here: waiting costs nothing,
while being refused at the database costs the work already done.

**You can see both halves of that on the dashboard:**

- **[Task Scheduler Throttled Rate per Operation](../observability/dashboards/server/temporal-server-readme.md#9-shard-queue-health)** counts every task the
  limit held back. Anything above zero means the limit is being reached. In this playbook's test it
  ran at **318 to 573 a second** while tasks completed at **60 a second** — a lot of waiting, and
  exactly what was wanted.
- **[Task Scheduler Throttling by Namespace](../observability/dashboards/server/temporal-server-readme.md#9-shard-queue-health)** is the same count split by
  namespace, which is the one to use when you have set a per-namespace limit.
- **[Task Scheduler Latency per Operation](../observability/dashboards/server/temporal-server-readme.md#9-shard-queue-health)** shows how long tasks are
  waiting between being loaded and starting. This is the number that tells you whether the waiting
  has become too much of a good thing.

**A throttling count above zero is not a problem to fix.** It is the setting doing its job, and it
is the signal you are aiming for once the numbers are right — which the next section covers.

> ### Which pair to use: the cluster-wide one
>
> **The `Global` settings are divided between pods by shard ownership**, exactly like
> `history.persistenceGlobalMaxQPS`. Measured on a two-pod cluster with 2048 shards, with
> `taskSchedulerGlobalNamespaceMaxQPS` set to **60 tasks per second**:
>
> | Pod | Shards it owned | Tasks per second that pod would start |
> |---|---|---|
> | first pod | 1049 | **30.7** — that is `60 × 1049 ÷ 2048` |
> | second pod | 999 | **29.3** — that is `60 × 999 ÷ 2048` |
> | **together** | **2048** | **60 — exactly the number configured** |
>
> So the value you set is the **cluster total**, and each pod enforces its share of it.
>
> **Prefer the cluster-wide pair, for two reasons.**
>
> - **It can be compared with your database budget without any arithmetic.** Your persistence
>   limit is a cluster-wide number of database calls a second; this is a cluster-wide number of
>   tasks a second. One divides into the other by the calls-per-task figure, and that is the whole
>   sizing calculation. With the per-pod setting you would have to multiply by your pod count
>   first to know what the cluster is allowed to start.
> - **It does not move when the fleet does.** Add history pods and each one takes a smaller share
>   of the same cluster total, so the database sees no change. A per-pod limit of 60 on four pods
>   permits 240 tasks a second; add two more pods and it silently permits 360.

### 4.2 Two traps that leave these settings doing nothing

Both are easy to walk into, and both leave you with a cluster that looks configured and behaves
exactly as it did before:

1. **The limiter is on, but shadow mode is still on with it** — so nothing is held back.
2. **The four limits are left at `0`** — which does not mean "no limit", it means "use the
   persistence number", and that number is several times too large.

#### Trap one: turning the limiter on is not enough

`history.taskSchedulerEnableRateLimiterShadowMode` defaults to **`true`**, which means "measure, do
not act". So a cluster with the master switch turned on and nothing else changed is **counting what
it would have held back, and holding nothing back.**

That default is useful rather than merely annoying, because shadow mode is how you size the limits
before they bite: **[Task Scheduler Throttled Rate per Operation](../observability/dashboards/server/temporal-server-readme.md#9-shard-queue-health)** counts
throttling in shadow mode exactly as it does when the limiter is live — the count is recorded before
the shadow-mode check — so the panel tells you what a given number *would* have delayed.

**Holding tasks back needs both switches**, like this:

```yaml
history.taskSchedulerEnableRateLimiter:
  - value: true
history.taskSchedulerEnableRateLimiterShadowMode:
  - value: false
```

**That is not the first thing to do, though.** Turn the limiter on and **leave shadow mode at
`true`**: read what it would have held back, work the numbers out from that, and only then set
shadow mode to `false`. Doing it in that order is the next section's job — this section is only
making sure you know the switch exists, because leaving it at its default is the commonest way for
these settings to appear set and do nothing.

#### Trap two: `0` does not mean "no limit"

Leaving the four limits at `0` does not switch them off. **A `0` means "use the persistence limit
instead"** — the scheduler falls back to the persistence rate for the same scope, per pod or
cluster-wide.

The trouble is that the two limits count different things:

| | What it counts |
|---|---|
| the persistence limits | **database calls** |
| the task scheduler limits | **tasks** |

**Processing one task makes several database calls.** At the least a write of the new state, usually
a read of that state before it, and on a SQL store that read is itself nine queries
([3.1](#31-the-sequence-that-creates-the-loop)). So a scheduler that inherits the persistence
number ends up set several times higher than the database will serve: it admits enough tasks to
generate three or four times the calls that the persistence limit accepts.

**What that costs you is the pacing, not the database.** The database is still protected — its own
limit does that job. But the refusals land at the database instead of at the scheduler, which is the
write-reject loop over again. **A scheduler limit that never engages until the database is already
refusing work is not doing anything for you.**

> **Measured on the test cluster:** **3.69** database calls per task, and 2.3 to 3.2 on a workload
> of a different shape. **Measure your own.** The next section — choosing the numbers and rolling
> them out — divides your database budget by this figure, so everything it tells you to set depends
> on it.

**So the settings need two things from you:** the switches set in the right order, and numbers of
their own rather than the inherited ones.

### 4.3 How to see what is really in force

There is no metric for the rate a pod is enforcing. The one place it appears is the pod's log, which
prints a line whenever the value changes:

```bash
grep "Quota changed" <history-pod-log>
```

Each line carries a component and a scope. For these settings look for
`component=task-scheduler` with `scope=host` or `scope=namespace`; the persistence limits appear on
the same log line format with `component=persistence`. **This needs the pod's log level at `info`**
— at `warn` the line is never written.

Reading those lines is how the shard-weighting above was confirmed, and it is the quickest way to
settle an argument about whether a setting took effect.

---

**Next — the division of labour between these two sections.** This one was *what the settings are*:
the four limits, the three switches, and the two traps. The next one is *what to do with them*.
**Nothing needs doing yet** — the next section walks through all four of these steps with you:

1. Turning the limiter on with shadow mode left on, so nothing is held back yet.
2. Working out your numbers, from your own database budget and your own calls-per-task figure.
3. Setting those numbers, taking it out of shadow mode, and checking the result on the panels.
4. Adjusting — including what to look at if you have set them too low.

---

## 5. Sizing the limits and turning them on

Four steps, in this order. **Step 1 changes nothing about how the cluster behaves**, so it is safe
on a running cluster at any time.

**One worked example runs through the whole section.** It is the test cluster: a database budget
of 300 calls a second, which turns into a namespace limit of 60 and an all-namespaces limit of 80.
Every number shown in a box or a YAML block below is from that example, and is there so you can
see the shape of the calculation — **substitute your own numbers, which will be different.**

Before starting, you need one thing in place: **a persistence limit that reflects what your
database can actually take.** Everything below is derived from it. If you have not set one, that is
[History Persistence QPS Limits](./history-persistence-qps-limits.md) — do that first, because
sizing task processing against a limit nobody chose gives you a number with nothing behind it.

### 5.1 Step 1 — turn the limiter on, with shadow mode left on

```yaml
history.taskSchedulerEnableRateLimiter:
  - value: true
```

That is all. `history.taskSchedulerEnableRateLimiterShadowMode` is already `true` by default, which
is what you want here: the limiter evaluates every task and counts what it would have held back,
and holds nothing back.

**What you get from this step** is a reading on
**[Task Scheduler Throttled Rate per Operation](../observability/dashboards/server/temporal-server-readme.md#9-shard-queue-health)**. With the four limits
still at `0`, the limiter is using the inherited persistence number
([trap two](#trap-two-0-does-not-mean-no-limit)), so expect that panel to be low or empty — that is
the point being demonstrated, not a problem.

### 5.2 Step 2 — work out your numbers

**Two figures go into this: your database budget, and how many database calls one task costs.**

**First, calls per task.** It is one division, and both numbers come off the dashboard:

`calls per task = database calls that reached the store per second ÷ tasks completed per second`

- **The top number** comes from
  **[Database Calls That Reached the Database](../observability/dashboards/server/temporal-server-readme.md#3-persistence-requests-latencies-and-errors)** —
  what the store actually served. **Not** the attempts, for the reason in
  [3.5](#35-before-you-change-anything): attempts include calls that were refused.
- **The bottom number** comes from **[History Task Throughput](../observability/dashboards/server/temporal-server-readme.md#9-shard-queue-health)**, using
  the `total` line. Take care not to use **Total Timer Tasks Processed** in the timer group instead —
  it is the same metric filtered to timer tasks only.
- **Measure while the cluster is doing ordinary work** — not during a loop, when the reads are
  inflated by retries, and not while it is idle, when there is too little to divide.

> **Measured on the test cluster: 3.69** calls per task on a parent-and-child workload, and 2.3 to
> 3.2 on a different shape. **Use your own number** — a workload with more activities or larger
> histories will differ.

**Then divide — one number from each side of the budget.**

- **`history.taskSchedulerGlobalNamespaceMaxQPS`** — how many tasks a *single namespace* may start
  across the cluster. Decide what share of the database budget that namespace should get, and
  divide:

  `namespace limit = that namespace's share of the budget ÷ calls per task`

- **`history.taskSchedulerGlobalMaxQPS`** — how many tasks *all namespaces together* may start
  across the cluster. Use the whole budget, and the same divisor:

  `all-namespaces limit = whole database budget ÷ calls per task`

> ### Worked example, on the test cluster
>
> A database budget of **300** calls a second, and **3.69** calls per task measured on that
> workload:
>
> | | |
> |---|---|
> | tasks that budget can support | `300 ÷ 3.69` ≈ **81 a second** |
> | `taskSchedulerGlobalNamespaceMaxQPS` | set to **60** — under the total, leaving room for other namespaces and for the pod's own background work |
> | `taskSchedulerGlobalMaxQPS` | set to **80** — close to the total, so it acts as the ceiling rather than the number that binds |
>
> Those two values are the ones the rest of this section follows through: what happened when they
> were enforced, and what happened when the namespace number was later pushed to 150.

Set the per-namespace number **below** the all-namespaces number. The per-namespace limit is the one
you want doing the work; the all-namespaces limit is the backstop for everything together.

In dynamic config the pair looks like this. **These are the example's numbers — yours come out of
the division above:**

```yaml
history.taskSchedulerGlobalNamespaceMaxQPS:
  - value: 60
history.taskSchedulerGlobalMaxQPS:
  - value: 80
```

**One thing to be clear about before setting it:** `taskSchedulerGlobalNamespaceMaxQPS` is a ceiling
applied to **each namespace separately**. A value of 60 with no constraint means *every* namespace
may start up to 60 tasks a second — it does not mean 60 shared between them. What stops the total
running away is `taskSchedulerGlobalMaxQPS`, which is why that one is set close to the whole budget.

**Where that is useful.** Say the cluster carries a few namespaces serving live traffic, where
latency matters, and one more that runs a large batch job once a day. The batch namespace does not
care whether its work finishes in a minute or in twenty, but while it is running it can create far
more tasks than everything else put together — and scheduling and executing are shared across the
pod ([2.5](#25-every-control-point-in-one-place)). Giving it a lower ceiling of its own means the
daily batch takes longer and the live namespaces keep their share:

```yaml
history.taskSchedulerGlobalNamespaceMaxQPS:
  - value: 20
    constraints:
      namespace: batch-reports
  - value: 60
```

That reads as: **`batch-reports` may start 20 tasks a second, every other namespace may start 60,
and all of them together are still capped by `taskSchedulerGlobalMaxQPS`.** The all-namespaces
setting takes no constraint — it is one number for the cluster.

### 5.3 Step 3 — set the numbers, then take it out of shadow mode

**Continuing the worked example:** the division in step 2 gave 60 for the namespace and 80 for all
namespaces together, and the limiter has been running in shadow mode since step 1. Setting those
two numbers and turning shadow mode off is what makes it act — **with your own numbers in place of
these two:**

```yaml
history.taskSchedulerGlobalNamespaceMaxQPS:
  - value: 60
history.taskSchedulerGlobalMaxQPS:
  - value: 80
history.taskSchedulerEnableRateLimiterShadowMode:
  - value: false
```

**Then check three panels.** These are the numbers the test cluster produced, going from no limits to
the settings above:

| Panel | Before | After | What it means |
|---|---|---|---|
| **[Rejected Database Calls by Operation and Scope](../observability/dashboards/server/temporal-server-readme.md#3-persistence-requests-latencies-and-errors)** | up to **383/s** | **1.2/s** | The database has stopped refusing work |
| **[Write-Reject Loop Indicator (cleared / cache miss)](../observability/dashboards/server/temporal-server-readme.md#3-persistence-requests-latencies-and-errors)** | peak **18.2** | falling to about **0** | The loop has stopped |
| **[Task Scheduler Throttled Rate per Operation](../observability/dashboards/server/temporal-server-readme.md#9-shard-queue-health)** — or **[Task Scheduler Throttling by Namespace](../observability/dashboards/server/temporal-server-readme.md#9-shard-queue-health)** for the per-namespace view | **0** | **318 to 573/s** | Tasks now wait before they start, instead of being refused after they have begun |

**A fourth reading confirms the setting took.** On
**[History Task Throughput](../observability/dashboards/server/temporal-server-readme.md#9-shard-queue-health)**, watch the line for the namespace you have
limited. On the test cluster it settled at **59.7 to 60.0 tasks a second** against a configured
`taskSchedulerGlobalNamespaceMaxQPS` of 60 — cluster-wide, as
[4.1](#41-what-each-setting-controls) describes. **If it does not settle near the number you set**,
read [what is really in force](#43-how-to-see-what-is-really-in-force) before changing anything
else: the most likely causes are shadow mode still being on, or the value not having been picked up
yet.

**Those readings also tell you when you are finished.** Your numbers will not be 60 and 80, but the
finished state looks the same on every cluster, so it is worth being explicit about it:

> ### What "tuned" looks like
>
> **Tasks waiting at the scheduler, and nothing being refused at the database.**
>
> - **Throttling above zero** on the scheduler panels — tasks are waiting, which is the setting
>   working, not a fault.
> - **Rejected database calls at or near zero** — nothing is being started that cannot finish.
> - **Queue lag falling** — the backlog is draining rather than holding.
>
> If you have the first two but lag is not falling, the limits are too low. If refusals come back,
> they are too high. Step 4 is how to move between those.

### 5.4 Step 4 — adjust

**Where you are now:** the limiter is live and holding tasks back — step 3 set the numbers and took
it out of shadow mode — and you have taken the three readings.

**If all three came out right, you are finished.** Leave the settings as they are and revisit them
when your workload or your database changes.

**If one of them did not**, the numbers are either too high or too low. Change the number and read
the same panels again — **a change takes effect within about a minute**, when the limiter next
re-reads its settings. Each direction has its own signature, and both were measured:

#### The limits are set too high

**What you see.** Refused database calls come back, the loop indicator climbs back above 5, and task
throughput sits at whatever number you set.

**What to do.** Lower `history.taskSchedulerGlobalNamespaceMaxQPS`. Any value above
`your budget ÷ your calls per task` produces this, so work that figure out again and set the
namespace number below it. Leave `history.taskSchedulerGlobalMaxQPS` where it is.

> **On the test cluster.** The per-namespace limit was pushed from 60 to 150, against the same
> budget of 300 calls a second. That asks for `150 × 3.69` ≈ **553 calls a second** from a budget of
> 300, and the panels showed it within a minute:
>
> | | |
> |---|---|
> | Rejected database calls | climbed back to **352/s** |
> | Loop indicator | back over 5, to **8.2** |
> | History task throughput | pinned at **150/s**, the number configured |
>
> The arithmetic predicted all three. **If refusals return after you raise a number, this is what
> has happened.**

#### The limits are set too low

**What you see.** Nothing refused, loop indicator flat — and queue lag that is not coming down. The
rejection panels look perfect, which is what makes this easy to miss.

**What to do.** Raise `history.taskSchedulerGlobalNamespaceMaxQPS`, and judge the result on queue
lag rather than on refusals. The two panels that show it:

- **[Immediate Queue Lag per Pod](../observability/dashboards/server/temporal-server-readme.md#9-shard-queue-health)** and
  **[Scheduled Queue Lag per Pod](../observability/dashboards/server/temporal-server-readme.md#9-shard-queue-health)** — lag flat or climbing while nothing is
  being refused means your limit, not the database, is the constraint.
- **[Task Scheduler Latency per Operation](../observability/dashboards/server/temporal-server-readme.md#9-shard-queue-health)** — how long tasks wait between
  being loaded and starting. Rising here is the cost of a limit set too low.

> **On the test cluster.** At 60 tasks a second the backlog drained at only about **2.2 workflows a
> second**. Nothing was refused and the loop indicator was flat — the work was simply being let
> through more slowly than it needed to be.

Either way, the method is the same:

> ### How to move the number
>
> 1. **Change `history.taskSchedulerGlobalNamespaceMaxQPS` only.** Leave
>    `history.taskSchedulerGlobalMaxQPS` alone — it is the ceiling over all namespaces together, not
>    the number you tune.
> 2. **Change it in steps, not in one jump.** If you are at 60 and the arithmetic says 81 is your
>    ceiling, try 70 and read the panels before trying anything higher. Going up a step at a time
>    shows you the value where refusals start coming back, and you then stay below it.
> 3. **Wait about a minute** for the limiter to pick the value up, then read the three readings from
>    step 3 again.
> 4. **Repeat until all three are right:** throttling above zero, refusals at zero, queue lag
>    falling.
>
> **If you cannot get all three at once**, your database budget is the constraint rather than these
> settings — whether to raise it is the decision in
> [History Persistence QPS Limits](./history-persistence-qps-limits.md).

---

**Next. Scheduling is now tuned** — the part that stops more tasks being started than can finish.
**Loading is not**, and it is the one thing left to tune. It comes two sections from here, because
capping how fast tasks are read only makes sense once tasks are completing.

What is in between, and what follows:

| | |
|---|---|
| **[Section 6](#6-how-the-scheduler-decides-what-goes-first)** | **Explanation, not tuning.** How the server already decides which tasks go first, why clean-up and archival never crowd out live work, how namespaces take turns, and what happens on a multi-cluster setup. Nothing in it needs configuring — skip it if you only want the remaining tuning step. |
| **[Section 7](#7-tuning-how-fast-tasks-are-loaded)** | **The second thing to tune: loading.** How fast tasks are read out of the database, now that they are completing. |
| [Section 8](#8-alerting-on-the-write-reject-loop) | **Alerting** — being told that refused database calls and the write-reject loop are starting again, rather than hearing it from someone whose workflows were slow. |
| Section 9 | **Reference** — every setting this playbook names, each with a verdict: tune it, read it but leave it alone, or out of scope here. |

---

## 6. How the scheduler decides what goes first

**This section adds no settings you need to change.** It describes the ordering the scheduler
already applies, because three questions come up as soon as someone starts tuning this, and all
three are answered by the same mechanism:

- Will archival and clean-up work crowd out the tasks that move workflows forward?
- On a multi-cluster setup, a namespace active in another cluster still has transfer, timer and
  visibility tasks here. Do they compete with the work for namespaces that are active here?
- Now that a per-namespace limit is set, what stops a busy namespace taking every turn *within* its
  limit?

### 6.1 Three priority classes, fixed by task type

Every task is put in one of three classes when it is loaded. **You cannot configure which class a
task lands in** — it follows from what the task is:

| Class | Which tasks | Turn frequency |
|---|---|---|
| **High** | Everything that moves a workflow forward: dispatching workflow and activity tasks, starting child workflows, telling a parent its child finished, and the rest. | weight **10** |
| **Low** | The timeout tasks: activity timeout, workflow task timeout, workflow run timeout, workflow execution timeout, and worker commands. | weight **9** |
| **Preemptable** | Clean-up work: deleting a workflow's history events once its retention period expires, deleting the execution record, removing it from the visibility store, and archiving it — plus any task type the pod does not recognise. | weight **1** |

The weights are how often each class gets a turn in the rotation: a High-class channel is visited
roughly **ten times as often** as a Preemptable one.

**Two things worth naming in that bottom row.** The clean-up tasks are the ones Temporal creates
when a workflow passes its retention period or is archived — if archival is what you are looking
at, that has a playbook of its own:
[Archival Backend Outage](./detecting-recovering-archival-outage.md). And the **history
scavenger** is *not* one of these tasks: it is a system workflow run by the worker service, so
nothing in this section changes how it behaves.

**One more rule applies if you run more than one cluster:** a task belonging to a namespace that
is active in a *different* cluster is put in the lowest class whatever its type. That, what
happens at a failover, and where replication fits are together in
[6.3](#63-if-you-run-more-than-one-cluster).

### 6.2 What you get without configuring anything

- **Deleting and archiving finished workflows cannot slow down running ones.** Those tasks are in
  the lowest class, so they get roughly one turn for every ten that go to work moving workflows
  forward. A large clear-out takes longer; it does not hold anything else up.
- **One busy namespace cannot take the whole scheduler.** Tasks waiting to start are held in
  separate groups by namespace and by class, and the scheduler takes from each group in turn. This
  decides **the order tasks are picked up in**, not how many run at once — tasks from several
  namespaces are running side by side throughout. It is what makes the batch example in
  [5.2](#52-step-2--work-out-your-numbers) work: the limit decides **how many tasks a second** that
  namespace may start, and the turn-taking keeps the other namespaces moving while it does.

**Where "without configuring anything" stops.** All three of those are about **ordering** — which
task goes next. None of them caps **how much** work a namespace does. A namespace that takes its
turn every time still uses slots and still spends database budget: taking turns stops it starving
the others, it does not stop it using most of the capacity.

Put as two questions, because they have opposite answers:

| | |
|---|---|
| **Which work goes first — do I have to arrange that?** | **No.** It is always on, there is nothing to switch on or tune, and it behaves as described above. Clean-up work, and work for namespaces active in another cluster, do not get in front of the tasks that move your workflows forward. |
| **How much of the cluster one namespace can use — is that handled too?** | **No, that one is yours.** Nothing caps it until you set `history.taskSchedulerGlobalNamespaceMaxQPS` — [section 5](#5-sizing-the-limits-and-turning-them-on). |

### 6.3 If you run more than one cluster

**Skip this if you run a single cluster** — none of it applies.

**Yes, a cluster does process tasks for a global namespace it is not active for.** It holds a copy
of that namespace's workflows, and it creates and runs **transfer, timer and outbound** tasks for
them. Those tasks run a different code path from the active one — instead of doing the work, they
check that the active cluster has done it and wait for the result to be replicated across.
(Visibility and archival tasks have no such split.)

**That work comes second, in two separate ways.** Both are automatic:

- **In the scheduler**, every one of those tasks is put in the lowest class whatever its type, and
  within that group all three classes carry the same weight, so nothing there gets ahead of
  anything else.
- **At the database**, those tasks make their persistence calls as preemptable work, which puts
  them in the lowest priority bucket of the persistence limit as well.

So a cluster that is active for some global namespaces and passive for others spends both its task
capacity and its database budget on the active ones first.

**A failover sorts itself out.** When a namespace becomes active here, each of its already-loaded
tasks moves up out of the lowest class the first time it runs, and its retry count is reset so it
does not carry attempts made while the namespace was passive. The weights switch across with it.
Nothing needs configuring. For the handover procedure itself, see
[Namespace Failover — Graceful Handover](./namespace-failover-graceful-handover.md).

**Replication tasks are not scheduled by the task scheduler.** They are a separate category with
their own machinery, and no setting in this playbook affects them. **They do share the database
budget, though:** in the test behind this playbook `RangeCompleteReplicationTasks` was being
refused at about 20 a second alongside everything else. So on a replicating cluster, replication is
part of what fills the persistence limit you size against.

> ### Recommendation: tune both clusters, and size the passive one for the load it would take on
>
> A cluster holding passive copies of global namespaces looks cheap to run, because standby tasks
> mostly check and wait. **Size it for what it would carry after a handover, not for what it
> carries now.** Two ways that goes wrong otherwise:
>
> - **A limit sized for passive work becomes the bottleneck the moment the namespace goes active.**
>   The workload arrives in full and meets a per-namespace limit chosen when there was almost
>   nothing to do, so the new active cluster throttles its own traffic.
> - **Leaving the limits at `0` on the passive cluster is no better.** They inherit the persistence
>   number ([trap two](#trap-two-0-does-not-mean-no-limit)), so the first busy hour after a handover
>   is the write-reject loop rather than a slow one.
>
> **Measure calls per task on the cluster where the namespace is active**, and use that figure when
> sizing both clusters. A number taken on the passive side will be too low — standby tasks verify
> and wait rather than doing the work, so they touch the database differently. *(Reasoned from how
> standby execution works; not measured.)*
>
> **After a handover, take the three readings again** on the cluster that has become active
> ([5.3](#53-step-3--set-the-numbers-then-take-it-out-of-shadow-mode)). Throughput for that
> namespace jumps from verifying to doing, and that is exactly the moment to check that the limit
> you chose in advance was the right one.

### 6.4 Changing the priority weights

**Back to the three classes in [6.1](#61-three-priority-classes-fixed-by-task-type).** Which class
a task lands in is fixed and cannot be configured — but the weights attached to those classes, the
**10**, the **9** and the **1**, can be. They are the only part of this section you *could*
change. This is about whether you should, and the answer is almost always no.

Two dynamic configs hold them, one pair per task category, and both take a value per namespace:

| Dynamic config | Applies to | Default |
|---|---|---|
| `history.<queue>ProcessorSchedulerActiveRoundRobinWeights` | namespaces this cluster is active for | High **10**, Low **9**, Preemptable **1** |
| `history.<queue>ProcessorSchedulerStandbyRoundRobinWeights` | namespaces active in another cluster | **1** for all three |

> ### Recommendation: keep the defaults
>
> **Do not change these.** The values Temporal ships already express the policy almost every
> cluster wants — work that moves workflows forward first, timeouts just behind it, clean-up last —
> and they hold on large clusters as well as small ones, because they set *proportions* rather than
> rates.
>
> Two further reasons:
>
> - **They change the order of work, not how much of it there is.** If too much work is reaching
>   the database, the rate limits in
>   [section 5](#5-sizing-the-limits-and-turning-them-on) are the fix. Re-weighting only changes
>   which tasks are made to wait.
> - **Nothing in this playbook was measured with them changed.** Every number here was taken with
>   the defaults in place, so a cluster with altered weights will not behave the way these examples
>   describe.
>
> **Then why are they configurable at all?** Because the weights are read per namespace, and the
> rotation runs over one queue per namespace per class. Giving one namespace lower weights than
> another makes its tasks get fewer turns — a way of biasing the whole cluster towards one
> namespace's work. **The per-namespace rate limit does that job more predictably**: it gives you a
> number in tasks per second that you can compare against a database budget, where a weight only
> tells you how often something gets a turn relative to everything else.

---

**Next: loading — the one part still untuned.**

It is easy to finish [section 5](#5-sizing-the-limits-and-turning-them-on) thinking the job is
done, and [section 6](#6-how-the-scheduler-decides-what-goes-first) was a step away from tuning
altogether. So here is where things stand against
the five control points listed in [2.5](#25-every-control-point-in-one-place):

| Control point | Where it stands |
|---|---|
| **Scheduling** | **Done** — [section 4](#4-the-task-schedulers-rate-limits) and [section 5](#5-sizing-the-limits-and-turning-them-on). |
| Tasks waiting in memory | Nothing to do. The pod manages these itself ([2.4](#24-when-loaded-tasks-build-up-in-memory)). |
| Executing | Left as it is, unless tasks are waiting for a slot rather than for the database. |
| The database calls a task makes | Not this playbook — [History Persistence QPS Limits](./history-persistence-qps-limits.md). |
| **Loading** | **Still untouched. This is what the next section is for.** |

**Why it is worth a section of its own.** Reading tasks out of the database costs database calls of
its own — the `Get*Tasks` operations — and they come out of the same budget as the reads and writes
that task execution needs. In the test behind this playbook, task loading was itself being refused
while execution was starved. Now that scheduling is fixed, three questions remain open:

- Is loading taking more of the database budget than it needs?
- What is your pod's real poll ceiling — including the case where it is silently **100,000 a
  second** because a cluster-wide persistence limit was set and the per-pod one was left at `0`?
- How would you know if you capped it too hard?

---

## 7. Tuning how fast tasks are loaded

Three questions, in order. **Start with the first one**: if your loading rate is already close to
the floor that your shard count produces on its own, there is nothing here to tune and you can skip
to the next section.

### 7.1 Is loading taking more of the budget than it needs?

**"Budget" here means your persistence limit** — the number of database calls a second the history
service is allowed to make ([History Persistence QPS Limits](./history-persistence-qps-limits.md)).
Loading spends from it and so does executing; this question is about how the two split it.

Open **[Task Loading Rate by Queue](../observability/dashboards/server/temporal-server-readme.md#9-shard-queue-health)** and compare what you see against the
floor below. **Each of the five task categories — transfer, timer, visibility, outbound and
archival — does its own loading**, and each of them re-checks every shard on a timer even when
there is no work at all. So there is a rate you cannot go below without changing the poll
intervals:

```
idle loading rate  ≈  shards  ×  queues  ÷  poll interval
```

**That is what this section calls the floor:** the rate your cluster reads tasks at when nothing is
happening at all. It is not zero, and it is not something a poll ceiling removes — a ceiling set
below the floor does not cancel the re-checks, it makes them wait their turn and happen late. The
only way to lower the floor itself is a longer poll interval.

On a cluster of **2048 shards** the floor is roughly **116 calls a second** with no workload
whatever: 102 from the three one-minute queues (transfer, visibility, outbound) and 14 from the two
five-minute ones (timer, archival). Work out the same figure for your own shard count before
deciding your loading rate is too high.

| What you see | What it means | What to do |
|---|---|---|
| A rate close to that floor | Normal. It is the cost of having shards, not of your workload | **Nothing.** Skip the rest of this section |
| Well above the floor, and tasks are completing | Loading is keeping up with real work | **Nothing needs fixing.** If loading is taking a large share of your database budget and you want some of it back, see [7.3 — choosing a poll ceiling](#73-choosing-a-poll-ceiling-and-knowing-if-you-set-it-too-low) |
| Well above the floor, and tasks are *not* completing | Loaded tasks are being thrown away and read again — the pod's response to a backlog it cannot clear ([2.4](#24-when-loaded-tasks-build-up-in-memory)) | **Go back to [section 5](#5-sizing-the-limits-and-turning-them-on).** Capping loading here will not fix it |

**Which row you are in decides what to do next.**

- **First row — at the floor.** Nothing to tune. The rate is what having shards costs.
- **Last row — high, and tasks are not completing.** Loading is not your problem; go back to
  [section 5](#5-sizing-the-limits-and-turning-them-on) and get tasks finishing first.
- **Middle row — loading is keeping up, and you want some of that budget back.** That is what
  [7.3](#73-choosing-a-poll-ceiling-and-knowing-if-you-set-it-too-low) is for, and it is a real tuning
  decision rather than a chore: **every call loading does not make is one that executing tasks
  can.** On a cluster where the database budget is the thing you are short of, this is worth doing.

**Whichever row you are in, go through [7.2](#72-what-is-your-pods-real-poll-ceiling).** It is not
tuning — it is working out a number that appears on no panel: **how much loading is allowed to
cost at its worst.**

7.1 told you what loading costs right now, with the work you have. The ceiling tells you what it
could cost **when every shard has to be loaded at once** — after a restart, a deploy, or shards
moving between pods. If that ceiling is far above your database budget, an ordinary restart can
spend the budget on reading tasks while the tasks already loaded wait to run.

**One combination in particular is worth checking for:** a cluster-wide persistence limit set in
`history.persistenceGlobalMaxQPS`, with the per-pod `history.persistenceMaxQPS` left at `0`. That
pairing leaves the poll ceiling at **100,000 a second** — 7.2 explains why, and how to close it.

### 7.2 What is your pod's real poll ceiling?

**Loading happens by polling.** The pod asks the database for the next batch of tasks for a shard,
gets them back, and asks again. **The same activity has two names**, which is worth getting
straight before the settings:

- **On the dashboard** each ask is a database call named `GetTransferTasks`, `GetTimerTasks` and
  so on — written `Get*Tasks` in this playbook, and counted on
  **[Task Loading Rate by Queue](../observability/dashboards/server/temporal-server-readme.md#9-shard-queue-health)** in
  [7.1](#71-is-loading-taking-more-of-the-budget-than-it-needs).
- **In the settings** the same asking is called **polling**, which is why every setting that
  limits it has `Poll` in its name.

So a **poll ceiling** is the most `Get*Tasks` calls a pod is allowed to make per second.

Two settings cap it, and the second is usually not set:

| Setting | Default | What it caps |
|---|---|---|
| `history.<queue>ProcessorMaxPollRPS` | **20** | One shard's reads, for one queue |
| `history.<queue>ProcessorMaxPollHostRPS` | **0** — not set | The whole pod's reads, for one queue |

**When the per-pod setting is `0`, the ceiling is worked out from the persistence limit instead:**

| `history.persistenceMaxQPS` | The pod's poll ceiling becomes |
|---|---|
| set to a real number | `persistenceMaxQPS × 0.30` for transfer, timer and outbound; `× 0.15` for visibility and archival |
| **left at `0`** | **a hard-coded 100,000 a second** — no ceiling in any practical sense |

> ### The case worth checking right now
>
> **If you set `history.persistenceGlobalMaxQPS` and left `history.persistenceMaxQPS` at `0`, your
> poll ceiling is 100,000 a second.** It is a reasonable-looking combination — you set one
> cluster-wide number and there is no obvious reason to also set a per-pod one — but the per-pod
> number is what the poll ceilings are calculated from, so leaving it at `0` does not lower the
> ceiling, it removes it. No pod will ever make 100,000 `Get*Tasks` calls a second, which is the
> point: nothing is holding loading back any more.
>
> The persistence limiter still protects the database, so nothing breaks loudly. Loading simply has
> no cap of its own, and that shows up worst just after a restart, when a pod has every one of its
> shards to load at once.
>
> **Two ways to fix it. Either is fine:**
>
> - **Give `history.persistenceMaxQPS` a real value** — roughly the value of
>   `history.persistenceGlobalMaxQPS` divided by the number of history pods you run. Nothing about
>   database protection changes: while `history.persistenceGlobalMaxQPS` is set, that is still the
>   limit the pod enforces, and `history.persistenceMaxQPS` is ignored for that purpose. It is used
>   for one other thing — working out the poll ceilings in the table above — and that is why it
>   needs a value. The persistence playbook says the same thing in
>   [what each setting does when set to `0`](./history-persistence-qps-limits.md#26-what-each-setting-does-when-set-to-0).
> - **Or set `history.<queue>ProcessorMaxPollHostRPS` yourself**, per queue. When that setting has a
>   value, it is the pod's poll ceiling directly and nothing is calculated from
>   `history.persistenceMaxQPS` at all.

### 7.3 Choosing a poll ceiling, and knowing if you set it too low

**The number being chosen here is a value for `history.<queue>ProcessorMaxPollHostRPS`** — the
per-pod poll ceiling from [7.2](#72-what-is-your-pods-real-poll-ceiling), set once per queue. It is
the only setting in this section that gives database budget back to executing tasks.

**You are here because of the middle row of the table in
[7.1](#71-is-loading-taking-more-of-the-budget-than-it-needs):** loading is keeping up with real
work, and it is taking a share of the database budget you would rather spend elsewhere. You can see
how large that share is by comparing
**[Task Loading Rate by Queue](../observability/dashboards/server/temporal-server-readme.md#9-shard-queue-health)**
with
**[Database Calls That Reached the Database](../observability/dashboards/server/temporal-server-readme.md#3-persistence-requests-latencies-and-errors)**.

**A starting point for the value:** the proportions the server uses itself when the setting is left
at `0`, which are the ones in the table in 7.2 — **30 per cent** of a pod's database budget for
transfer, timer and outbound, **15 per cent** for visibility and archival. For a cluster-wide budget
of 300 calls a second across two pods, that is `(300 ÷ 2) × 0.30` ≈ **45 a second** for each of the
three busy queues. Setting it there changes nothing on its own; it makes the ceiling explicit, so
you can then lower it and watch what happens.

**The signal that the ceiling is too low** is
**[Task Load Latency by Task Type](../observability/dashboards/server/temporal-server-readme.md#9-shard-queue-health)** —
how long tasks waited before being loaded.

**The panel draws one line per task type, not one per queue** — but every task type name begins
with the name of the queue it came from, so the queue is readable straight off the legend:

| Queue | Lines on the panel look like |
|---|---|
| **transfer** | `TransferActiveTaskActivity`, `TransferActiveTaskWorkflowTask`, `TransferStandby…` |
| **timer** | `TimerActiveTaskUserTimer`, `TimerActiveTaskActivityTimeout`, `TimerStandby…` |
| **visibility** | `VisibilityTaskStartExecution`, `VisibilityTaskCloseExecution` |
| **outbound** | `OutboundActive.…`, `OutboundStandby.…` |
| **archival** | `ArchivalTaskArchiveExecution` |

**Read the queue off the line before reading the number**, because the number means two different
things:

| Queue | What the number is | Healthy reading |
|---|---|---|
| **transfer, visibility, outbound** — tasks due the moment they are created | Time from the task being created to being loaded | Low and steady |
| **timer, archival** — tasks due at a time in the future | **How late the load was against the task's fire time**, floored at zero. The timer's own duration is never counted | **Near zero.** Tasks loaded early by the read-ahead window record 0 |

**On the timer and archival lines, take that second row slowly** — it is the one that gets
misread. The number is measured **when the task is read out of the database**, not when it runs,
and a timer task's due time is the timestamp the row is stored under. So the measurement is:
**how long after the task became due did the pod read it out?** Read out before it was due — which
the read-ahead window normally manages — and it records 0.

Two consequences worth being clear about:

- **The timer's own length is never in the number.** A 24-hour timer read out on time records 0,
  not 24 hours.
- **A number above zero is work that already happened late.** 30 seconds on
  `TimerActiveTaskActivityTimeout` means those activity timeouts sat due-but-unread for 30 seconds,
  so the timeout was recorded, and the retry scheduled, 30 seconds later than they should have been.

**That is the clearest sign a poll ceiling is too tight.** One warning if you are reading this on a
dashboard of your own rather than the one linked above: the metric behind the panel is
`task_latency_load`, and the description it ships with — "from task generation to loading" —
describes only the first group. There is no second metric for the second group; it is the same
measurement, meaning something different.

> ### Recommendation: raise the poll interval before you lower the poll ceiling
>
> **A third setting belongs in this conversation, and it is not a ceiling. First, who tells a
> reader there is something to read.** Not the database, and not another pod: **the history pod
> that writes the tasks tells its own reader, in memory, the moment the write is done.** Same pod,
> same shard, no round trip. On a scheduled queue the reader is told the earliest fire time among
> the new tasks and sets a timer for it rather than reading immediately.
>
> **`history.<queue>ProcessorMaxPollInterval` is the backstop for when that does not happen** — the
> longest a reader will go without re-checking its shard anyway: **1 minute** for transfer,
> visibility and outbound, **5 minutes** for timer and archival. It is the divisor in
> [the floor formula in 7.1](#71-is-loading-taking-more-of-the-budget-than-it-needs): more shards,
> or a shorter interval, means more re-checks a second whether or not there is work.
>
> **`history.<queue>ProcessorMaxPollHostRPS` and `history.<queue>ProcessorMaxPollInterval` are not
> two ways of doing the same thing.** `MaxPollHostRPS` limits how fast the pod is allowed to read;
> it does not change how many re-checks want to happen. Set it below the floor and the re-checks do
> not disappear — each one waits for the limiter and the shard is read later than its interval,
> which is exactly the lateness the panel above measures. Raising `MaxPollInterval` removes
> re-checks instead, so the floor itself comes down.
>
> **So if your loading rate is close to the floor and you want it lower, raise the interval** rather
> than lowering the ceiling. The risk is small, because of who does the telling: work written while
> the pod owns the shard is announced in memory as it arrives, so the backstop only matters for
> tasks no live notification covered — those written before the shard moved to this pod, most
> obviously after a restart or a rebalance.
>
> **It only helps shards that are genuinely idle, though.** A shard with pending work reads as often
> as that work requires, which is sooner than the interval — so on a busy cluster this is a weak
> lever and `history.<queue>ProcessorMaxPollHostRPS` is the one that bites.

---

**Next:** scheduling and loading are now tuned for the cluster you have today. They will not stay
tuned — traffic grows, pods are added and removed, someone raises a limit for an unrelated reason —
and the numbers you just worked out quietly stop matching the cluster they were worked out for.

**Section 8 is an alert for the write-reject loop**, for exactly that. It is not a substitute for
anything in this playbook — it is the opposite: it tells you when to come back and run
[section 3](#3-what-happens-when-the-persistence-limit-refuses-a-tasks-write) and
[section 5](#5-sizing-the-limits-and-turning-them-on) again — while it is still a shape on a graph,
and early enough to fix the latency before anyone using your workflows has to report it.

---

## 8. Alerting on the write-reject loop

**One alert in the set belongs to this playbook: alert 87**, which watches for the write-reject
loop from [section 3](#3-what-happens-when-the-persistence-limit-refuses-a-tasks-write). It is
worth knowing how much of this playbook it does and does not do for you.

| | |
|---|---|
| **What alert 87 tells you** | that the write-reject loop is running right now — the ratio from [3.3](#33-the-one-number-that-identifies-the-loop), confirmed against real rejections, done for you |
| **What alert 87 does not tell you** | anything about what to change. Not which namespace, not whether your scheduler limits are set, not whether the database was struggling. Those are [3.5](#35-before-you-change-anything) and [section 5](#5-sizing-the-limits-and-turning-them-on), and they still need you at the dashboard |

### 8.1 Alert 87 — History Write-Reject Loop

**[87 — History Write-Reject Loop](../observability/alerts/server/runbooks/87-history-write-reject-loop.md)**
fires when **both** of these hold for **10 minutes**:

| What it measures | What has to be true |
|---|---|
| **The loop ratio** — cached workflow state discarded, divided by state genuinely missing from the cache. The number in [3.3](#33-the-one-number-that-identifies-the-loop) | **higher than 5.** Strictly higher: a ratio of exactly 5 does not fire |
| **The rejection rate** — database calls the history service tried to make and Temporal's own limiter refused | **more than 10 refused calls per second**, added up across every history pod and averaged over five minutes |

Grafana re-checks the rule **once a minute** — the evaluation interval the alert set ships with —
and both conditions have to be true on every check for ten minutes before it fires. Either one
dropping below its number resets the ten minutes.

**Why both conditions.** A refused write is not the only reason the history service discards
cached state — a few of its internal recovery paths do it too, and they have nothing to do with
rate limiting. So a high ratio on its own does not prove the loop is running. Pairing it with
refused calls is what makes the alert specific to the loop rather than to state being discarded
for any reason.

#### Alert 87 and alert 86 are layers, not alternatives

[Alert 86 — History Database Calls Rejected](./history-persistence-qps-limits.md#51-alert-86--history-database-calls-rejected)
belongs to the companion playbook,
[History Persistence QPS Limits](./history-persistence-qps-limits.md). It watches one thing:
database calls the limiter refused. **Alert 87 watches those same refused calls and the loop ratio
together** — which is why one is a warning and the other is critical.

| | Alert 86 | Alert 87 |
|---|---|---|
| **Watches** | refused database calls | refused database calls **and** the loop ratio |
| **Severity** | `warning` | `critical` |
| **What it means** | the limiter is turning work away | the refusals have stopped draining on their own |
| **Why that severity** | refused calls recover by themselves — work waits, retries, and drains once the pressure passes | the loop supplies its own load, so it does not drain; and the obvious response, raising the persistence limit, makes it worse |
| **Where the fix is** | [History Persistence QPS Limits](./history-persistence-qps-limits.md) | this playbook, starting at [3.5](#35-before-you-change-anything) |

#### What you are looking at when they fire

| What is firing | What it means | What to do |
|---|---|---|
| **86 only** | The limiter is refusing calls, and they are draining. Common, and not by itself a loop. | The companion playbook's [four checks](./history-persistence-qps-limits.md#45-four-checks-and-what-each-one-tells-you-to-change) |
| **86 and 87** | The normal picture for the loop. 87 requires refused calls, so 86's condition is usually met too. | [3.5](#35-before-you-change-anything) |
| **87 only** | Still the loop — read it exactly as the row above. The two alerts count refusals differently: 86 counts each scope and cause on its own, 87 adds every `PersistenceLimit` refusal together. A cluster whose refusals are split evenly across scopes can pass 87's threshold while each of 86's instances stays under its own. | [3.5](#35-before-you-change-anything) |
| **87 alone with no refusals at all** | Not possible. The rule cannot fire without the rejection condition. If you think you are seeing it, check that the rule deployed is the one in this set. | — |

**Why [3.5](#35-before-you-change-anything) is the landing point** rather than any of the tuning
sections: it is the routing table, and it sends you to whichever of
[4](#4-the-task-schedulers-rate-limits), [5](#5-sizing-the-limits-and-turning-them-on) or
[7](#7-tuning-how-fast-tasks-are-loaded) your cluster actually needs. Which one that is depends on
what is already set, and the alert cannot know that.

### 8.2 Adjusting alert 87 for your cluster

**Leave the ratio of 5 alone. The number worth revisiting is the rejection gate.**

> ### Recommendation: keep alert 87's rejection gate equal to alert 86's threshold
>
> Both ship at **10 refused calls a second**. If you raise alert 86's threshold — the companion
> playbook's [5.2](./history-persistence-qps-limits.md#52-adjusting-when-alert-86-fires) says when
> that is the right thing to do, on a cluster that throttles routinely and recovers every time —
> **raise 87's gate to the same number in the same change.** Leave them apart and 87 fires on a
> level of rejection you have already decided is normal for your cluster.
>
> The gate is the second half of alert 87's expression, the one comparing
> `persistence_errors_resource_exhausted` against `10`.

> ### Where the ratio of 5 comes from
>
> A test cluster driven into the loop deliberately, twice, with a backlog far larger than the
> database budget it was given:
>
> | | first run | second run |
> |---|---|---|
> | healthy reading, before | below **1** | below **1** |
> | while the loop ran | **5.1 → 6.7 → 9.3 → 10.9** | **6.3 → 8.0 → 8.8 → 12.6** |
> | peak | **18.2** | **14.6** |
>
> On the second run the alert itself was watched through the whole cycle on that cluster: silent
> while the cluster was idle, **pending** within half a minute of the loop forming, **firing**
> exactly ten minutes later with the indicator at 14.6, and silent again about twenty seconds after
> the limit was restored and the backlog drained.
>
> Two runs is two runs, not a survey. What they suggest is that the gap between healthy and broken
> here is wide rather than narrow, which is why this is the one number in this section left at a
> single value rather than tuned per cluster.

> ### Do not remove the `> 0` guards from the alert's query
>
> The expression filters both sides of the ratio to `> 0` before dividing. That looks redundant and
> is not: **on an idle cluster both rates are `0`, and `0 ÷ 0` in PromQL is `NaN`.** Grafana's
> threshold step treats `NaN` as breaching, so the unguarded ratio fires continuously on a cluster
> doing nothing at all.
>
> **Verified on a test cluster:** the bare ratio returned `NaN`; the guarded expression returned an
> empty result, which with `noDataState: OK` keeps the rule silent. The dashboard panel is
> deliberately left unguarded — a gap in a graph is informative, a permanently firing alert is not.

### 8.3 Why there is no alert on task scheduler throttling

**Because after [section 5](#5-sizing-the-limits-and-turning-them-on) it would fire on a healthy
cluster.** The scheduler holding tasks back is the state you were aiming for — it is what
[What "tuned" looks like](#53-step-3--set-the-numbers-then-take-it-out-of-shadow-mode) describes,
and the limits in [4.1](#41-what-each-setting-controls) exist to produce it. A limit that never
throttles is a limit set above what the cluster can produce, which is a limit doing nothing. So
the same signal would have to mean both "this is working" and "something is wrong", and it cannot.

**Read it on the dashboard instead.**
**[Task Scheduler Throttling by Namespace](../observability/dashboards/server/temporal-server-readme.md#9-shard-queue-health)**
shows which namespaces are being held back and by how much. That is the question worth asking —
not whether throttling is happening at all.

#### When task scheduler throttling does mean something

**Throttling here means the task scheduler holding tasks back** — tasks that were ready to start
and were made to wait because one of the four limits in [4.1](#41-what-each-setting-controls) was
reached. It is not the persistence limiter refusing a database call, which is
[section 3](#3-what-happens-when-the-persistence-limit-refuses-a-tasks-write), and it is not a
worker being throttled.

A steady level of it is fine — that is the limiter working. **A level that keeps climbing is your
cluster telling you it has outgrown the budget the limits were sized against**, and that is not
something a lower limit fixes.

Two things move together when it happens:

| What you see | What it means |
|---|---|
| Throttling rises, and tasks still complete at the rate you expect | The limits are doing their job against a workload that has grown. Re-run [5.2](#52-step-2--work-out-your-numbers) against your current database budget |
| Throttling rises **and** task latency rises with it | More tasks are being produced than the budget can ever carry. No setting in this playbook creates capacity — this is a decision about the database: more of it, or less work arriving |

The second row is the honest limit of what tuning can do. These settings decide **which** work gets
the budget and in what order; they cannot make the budget larger.

### 8.4 The one thing no alert can tell you: whether your limits are actually on

**A cluster where the limits are set but inert looks exactly like a cluster where they are
working.** There is no metric for the rate a pod is enforcing, so this cannot be alerted on at all
— it has to be checked. Both ways it happens are the traps in
[4.2](#42-two-traps-that-leave-these-settings-doing-nothing):

| What is wrong | What you see | What is actually happening |
|---|---|---|
| **`history.taskSchedulerEnableRateLimiterShadowMode` is still `true`** — its default | The throttling panels show throttling, so the limiter looks live | Nothing is held back. Shadow mode counts what *would* have been held back |
| **The four limits are still `0`** — `history.taskSchedulerMaxQPS`, `history.taskSchedulerNamespaceMaxQPS`, `history.taskSchedulerGlobalMaxQPS`, `history.taskSchedulerGlobalNamespaceMaxQPS` | The settings are present in dynamic config, so they look configured | `0` is not "no limit". Each falls back to the matching persistence number, which counts database calls rather than tasks and is several times too large |

**So make the check part of the change.** Every time you touch these settings, read back what the
pod is enforcing using [4.3](#43-how-to-see-what-is-really-in-force) — the `Quota changed` line in
the pod log, which is the only place the effective rate appears. Confirming it takes one command;
not confirming it can leave you believing a cluster is tuned while it runs exactly as it did
before.

---

## 9. Every setting named in this playbook

One row per setting, with its default, what it does, and where in this playbook it is discussed.
**The rightmost column is the verdict** — whether this playbook expects you to change it.

> ### `<queue>` is a placeholder, not a literal
>
> Every setting written `history.<queue>Processor…` exists five times, once per queue. Replace
> `<queue>` with `transfer`, `timer`, `visibility`, `outbound` or `archival` — for example
> `history.transferProcessorMaxPollHostRPS`. There is no setting that covers all five at once, so
> a change you want everywhere is five entries in dynamic config.

### 9.1 The task scheduler — the settings this playbook is about

Nothing here is set by default. Sizing them is [section 5](#5-sizing-the-limits-and-turning-them-on).

| Setting | Default | What it does | Verdict |
|---|---|---|---|
| `history.taskSchedulerEnableRateLimiter` | `false` | Master switch. While `false`, the four limits below do nothing | **Set it.** [5.1](#51-step-1--turn-the-limiter-on-with-shadow-mode-left-on) |
| `history.taskSchedulerEnableRateLimiterShadowMode` | `true` | Measure-only. Counts what would have been held back and holds nothing back | **Set it to `false`** once sized — [5.3](#53-step-3--set-the-numbers-then-take-it-out-of-shadow-mode). Leaving it `true` is [trap one](#42-two-traps-that-leave-these-settings-doing-nothing) |
| `history.taskSchedulerGlobalNamespaceMaxQPS` | `0` | Tasks per second the **whole cluster** may start **for one namespace** | **Tune.** The one that usually binds — [5.2](#52-step-2--work-out-your-numbers) |
| `history.taskSchedulerGlobalMaxQPS` | `0` | Tasks per second the **whole cluster** may start, all namespaces | **Tune.** The ceiling over everything — [5.2](#52-step-2--work-out-your-numbers) |
| `history.taskSchedulerNamespaceMaxQPS` | `0` | Same, but **per pod** rather than cluster-wide | Use only if you are not using the cluster-wide pair — [4.1](#41-what-each-setting-controls) |
| `history.taskSchedulerMaxQPS` | `0` | Same, but **per pod**, all namespaces | Use only if you are not using the cluster-wide pair — [4.1](#41-what-each-setting-controls) |
| `history.taskSchedulerRateLimiterStartupDelay` | `5s` | Grace period after a pod starts before the limiter applies | **Leave alone.** It exists so a restarting pod is not held back while it is still finding its feet |

**`0` on any of the four limits is not "no limit"** — it falls back to the persistence number, which
counts database calls rather than tasks. That is [trap two](#42-two-traps-that-leave-these-settings-doing-nothing).

### 9.2 Loading — tune after scheduling, not before

| Setting | Default | What it does | Verdict |
|---|---|---|---|
| `history.<queue>ProcessorMaxPollHostRPS` | `0` = not set | The **pod's** ceiling on `Get*Tasks` calls for one queue | **Tune, if loading is taking budget you want back** — [7.3](#73-choosing-a-poll-ceiling-and-knowing-if-you-set-it-too-low). At `0` the ceiling is computed from `history.persistenceMaxQPS` — [7.2](#72-what-is-your-pods-real-poll-ceiling) |
| `history.<queue>ProcessorMaxPollInterval` | **1 min** (transfer, visibility, outbound), **5 min** (timer, archival) | Longest a reader goes without re-checking a shard when nothing has told it there is work | **Raise this before lowering the ceiling** — [7.3](#73-choosing-a-poll-ceiling-and-knowing-if-you-set-it-too-low) |
| `history.<queue>ProcessorMaxPollRPS` | **20** | The same ceiling, but for **one shard** | **Leave alone.** The per-pod setting is the one that governs a pod's total load on the database |

### 9.3 Read these, but do not change them

They explain what you are seeing. None of them is the fix for anything in this playbook.

| Setting | Default | What it does | Where it comes up |
|---|---|---|---|
| `history.<queue>ProcessorSchedulerWorkerCount` | **512** | Task slots per pod per queue — how many tasks run at once. Exists for transfer, timer, visibility and archival; **the outbound queue has no such setting**, it uses the two below | [2.3](#23-executing--the-task-slots). Raise only when tasks wait for a **slot** rather than for the database |
| `history.queuePendingTaskCriticalCount` | **9000** | Loaded tasks held before the pod starts unloading some | [2.4](#24-when-loaded-tasks-build-up-in-memory) |
| `history.queuePendingTasksMaxCount` | **10000** | Point at which the reader stops loading entirely and pauses | [2.4](#24-when-loaded-tasks-build-up-in-memory) |
| `history.<queue>ProcessorPollBackoffInterval` | **5s** | How long that pause lasts | [2.4](#24-when-loaded-tasks-build-up-in-memory) |
| `history.<queue>ProcessorSchedulerActiveRoundRobinWeights` | **10 / 9 / 1** | Priority weights for namespaces this cluster is active for. Transfer, timer and visibility only | [6.4](#64-changing-the-priority-weights) |
| `history.<queue>ProcessorSchedulerStandbyRoundRobinWeights` | **1 / 1 / 1** | Priority weights for namespaces active elsewhere. Transfer, timer and visibility only | [6.4](#64-changing-the-priority-weights) |
| `history.outboundQueue.hostScheduler.maxTaskRPS` | **100/s** | The outbound queue's own rate cap — it does not use the shared task scheduler | [2.3](#23-executing--the-task-slots) |
| `history.outboundQueue.groupLimiter.concurrency` | **100** | Outbound tasks in flight per destination | [2.3](#23-executing--the-task-slots) |

### 9.4 Named here, but belonging to another playbook

| Setting | Default | Why it appears here | Where it belongs |
|---|---|---|---|
| `history.persistenceMaxQPS` | **9000** | The poll ceilings in [7.2](#72-what-is-your-pods-real-poll-ceiling) are computed from it, and at `0` they fall back to a hard-coded 100,000 | [History Persistence QPS Limits](./history-persistence-qps-limits.md) |
| `history.persistenceGlobalMaxQPS` | `0` | Setting this while leaving `history.persistenceMaxQPS` at `0` is what removes the poll ceilings — [7.2](#72-what-is-your-pods-real-poll-ceiling) | [History Persistence QPS Limits](./history-persistence-qps-limits.md) |
| `history.hostLevelCacheMaxSize` | **128000** | The obvious suspect when reads spike, and the wrong one during the write-reject loop — [3.3](#33-the-one-number-that-identifies-the-loop) shows why | Not a task processing setting |
| `history.hostLevelCacheMaxSizeBytes` | ~**1 GB** | The byte-based limit on the same cache | Not a task processing setting |
| `history.maxInFlightUpdates` | **10** | One of the client-visible symptoms in [3.4](#34-what-the-loop-does-to-client-requests) — updates queue behind workflow tasks that are not completing | Not a task processing setting |
| `history.maximumBufferedEventsBatch` | **100** | Signals buffer while a workflow task cannot complete; past this the workflow task is force-failed — [3.4](#34-what-the-loop-does-to-client-requests) | Not a task processing setting |
| `history.maximumBufferedEventsSizeInBytes` | **2 MB** | The byte-based limit on the same buffer | Not a task processing setting |
