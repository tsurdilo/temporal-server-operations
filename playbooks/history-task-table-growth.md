# History Task Table Growth — Stuck Queue Cleanup Playbook

**A task table that will not stop growing has a handful of causes that need opposite responses.**
This playbook tells them apart, says what to do about each, and explains why.

## Note to readers

**Two open issues in the Temporal server bear directly on this playbook.** Neither had landed in a
released server as of **v1.32.1**, the latest release at the time of writing, and everything here
is written as though neither has.

- **[temporalio/temporal#12341](https://github.com/temporalio/temporal/issues/12341)** — the
  cleanup delete cannot recover once it exceeds its timeout. Proposes running the delete in bounded
  batches, and making the timeout a dynamic config setting.
- **[temporalio/temporal#12342](https://github.com/temporalio/temporal/issues/12342)** — the
  cleanup delete's query shape stops the database bounding its scan. SQL persistence only.

**Readers are encouraged to check each issue for its current status, and for the release any fix
shipped in**, before taking the behaviour described here as settled.

**The limit is called the delete query timeout throughout this playbook, rather than named by its
value.** That value is hard-coded to **five seconds** today and cannot be changed by configuration
— and making it configurable is one of the two things #12341 proposes, which is why the playbook
avoids building the number into its wording. Where the number matters for reading a dashboard, the
text gives it.

## Introduction

Every history task the server creates is a **row in a database table**. There is one table per
task category — timer tasks, transfer tasks, visibility tasks, archival tasks. The history service
writes a row for each piece of work a workflow's progress creates: some of it to be done straight
away, some at a set time in the future. Processing that task is what moves the workflow on to its
next step, and [History Task Processing](./history-task-processing-tuning.md) covers how that
processing is tuned.

**Processing a task does not delete its row.**

The history service removes rows separately, and in bulk. The rest of this introduction is how that
works, because everything in this playbook follows from it.

### How rows are removed

Each history shard runs a **queue** for every task category — an in-memory processor inside the
history service that reads that category's tasks for that shard out of the database and hands them
off to be executed. "Queue" is the server's own name for it; it is not a message queue, and it is
not the database table.

Each queue tracks one **position**: the fire time or task id of the **oldest task it has not yet
finished**.

**Two separate things act on that position, and telling them apart is the key to everything else in
this playbook:**

| | What it does | What it does to the position |
|---|---|---|
| **Task processing** | reads tasks out of the database and runs them | **Moves the position forward.** Tasks can finish in any order, so the position sits at the oldest one still outstanding. When that one finishes, the position jumps forward to the next still outstanding, past everything in between that is already done — so it can move a long way in one step, or not at all. |
| **Row deletion** | runs one `DELETE` against the task table, covering everything older than the position | **Follows the position.** One statement, one contiguous range of rows — far cheaper than deleting rows one at a time as they finish. |

Put plainly: **processing moves the position, and deletion removes rows up to it.**

**Deletion never gets ahead of processing, and that is a correctness guarantee.** A row whose task
might still need to run can never be removed, on any store, under any load. Cleanup cannot cost you
work.

**Following the position is also what makes cleanup cheap.** Deletion does not scan the table,
does not search for
completed work and keeps no per-row record of what has finished — the position already tells it
everything it needs. So the normal cost of cleaning up is one statement per interval, however many
tasks ran in it, rather than a delete per task.

What it does inherit is processing's progress: cleanup keeps up exactly as well as processing does.
On a cluster where processing keeps moving, deletion moves along behind it and clears out the
completed tasks.

### When cleanup runs

Deletion runs on a timer. Each category has its own interval setting and all five default to
**30 seconds**: `history.timerProcessorUpdateAckInterval`,
`history.transferProcessorUpdateAckInterval`, `history.visibilityProcessorUpdateAckInterval`,
`history.archivalProcessorUpdateAckInterval` and `history.outboundProcessorUpdateAckInterval`.
There is no single setting covering all of them.

On each of those intervals the queue **checkpoints**: it looks at where its position has got to and
writes that down. The `DELETE` is issued as part of that checkpoint, and only when the position has
moved since the last one. **If the position has not moved, the checkpoint still runs — there is
simply nothing to delete, and no statement is sent to the database.** So an idle cluster is not
repeatedly deleting nothing; it is repeatedly confirming there is nothing to delete, which costs it
nothing.

If you have read [History Task Processing](./history-task-processing-tuning.md), that is the same
loop that does the **loading** described there. Freeing rows is a fourth thing it does, alongside
loading, scheduling and executing.

### One queue per shard

**Task processing and cleanup both run per shard.** There is one queue **per shard, per task
category**, so a cluster
with a few thousand shards is running a few thousand timer queues, each tracking its own position
and cleaning up after itself, on whichever history pod currently owns that shard.

That independence is deliberate, and it is how the history service scales. Shards share nothing —
there is no central cleanup step they all queue behind, and no coordination between them to become
a bottleneck — so the work spreads across however many history pods own them. A cluster sized with
more shards simply has more of these loops running at once, which is what lets a single cluster
carry very high task volumes.

**The history scavenger is a different thing entirely.** The scavenger is a separate internal
workflow that removes
leftover workflow *history* — `history_node` and `history_tree` rows — and never touches the task
tables. If what is growing is history rather than tasks, it is covered in
[XDC Standby Database Growth on SQL](./xdc-standby-database-growth-sql.md#how-cleanup-works-and-why-the-standby-can-grow-larger);
the mechanics there apply to any cluster, not only a standby.

### Why one stuck task stops cleanup for a whole shard

One consequence of how the position is worked out is worth stating on its own:

> **The position sits at the oldest task that has not finished.** Tasks created after it keep being
> processed and completed normally — and they are. But their rows stay in the table, because
> deletion only removes rows *older* than the position. **Nothing created after that one task is
> deleted until that one task finishes**, however long ago those later tasks completed.

**Task tables are not organised by namespace.** Rows are ordered by shard and time, so every
namespace whose workflows land on a shard has its tasks mixed into the same sequence — and that one
sequence has one position. A task that cannot be processed for one namespace therefore delays
cleanup for every namespace with tasks on that shard.

### What stops a task finishing

**Why would one task stop finishing while everything around it keeps going?** Usually because one
namespace has been sitting at a rate limit for a long time. Its tasks keep being refused and
retried, while tasks for every other namespace are processed as normal — so the position stays at
that namespace's oldest task, and cleanup stops behind it.

**Two different limits can hold a task back, and each has its own playbook:**

- **Task scheduling limits** refuse the task *before* it runs, so it never reaches the database —
  [History Task Processing](./history-task-processing-tuning.md).
- **Persistence limits** refuse the database call the task makes *while* it runs —
  [History Persistence QPS Limits](./history-persistence-qps-limits.md).

Both can be set per namespace, which is what lets one namespace stall while the rest of the cluster
is fine. **There are two ways a namespace ends up held at its limit for long enough to matter:**

- **An incident.** A database outage, or any stretch where calls are failing, leaves one namespace
  with far more outstanding work than usual, and it then spends a long time at a limit that was
  never a problem before.
  [Section 4](#4-when-a-database-outage-leads-to-a-stalled-cleanup) follows that sequence through.
- **A limit that is simply set too low.** A namespace rate limit lowered deliberately — to contain
  noisy traffic, say — and then left in place as that namespace grew. No incident is involved,
  nothing is misbehaving, and nothing will clear it. A limit set too low is the harder of the two
  causes to spot, because there is no event to correlate the growth against.
  [Section 2](#2-how-the-server-decides-which-rows-are-safe-to-delete) explains why one namespace's
  limit reaches every other namespace sharing its shards.

**And one action that looks like a fix usually is not: deleting the namespace.** Deleting a
namespace is a workflow, not an instant operation. It **renames** the namespace first — to the
original name with `-deleted-` and part of its id appended — then deletes every workflow execution
inside it, and only removes the namespace record once that has finished. Until it finishes the
namespace still exists, its tasks still resolve, and they are still refused by the same limit as
before.

**The catch is that deleting those executions needs the very capacity the namespace does not
have.** Each one creates more tasks, and the server assigns that kind of cleanup work its lowest
priority class — the same class it gives tasks that are already being throttled. On a namespace
held at its limit, a deletion can therefore sit unfinished for as long as the limit holds, with the
position exactly where it was before you started.
[Section 6](#6-why-a-deleted-namespace-keeps-its-tasks-alive) covers how to recognise that state
and what to do instead.

Whichever limit refuses it, the task is **throttled, not failed** — and that is the right
behaviour. Throttling is
back-pressure doing its job: there is nothing wrong with the task, it simply arrived while its
namespace was at its limit, so it keeps its place and is tried again. Sending a task like that to
the dead-letter queue would be the wrong outcome, and the server does not do it.

**So throttling by itself is not something to act on.** When the rate subsides the retried task goes
through, the position moves, and the next checkpoint deletes everything that built up behind it.
Task processing carries on as normal and nothing is left over. This is a cluster protecting itself,
and it is working.

### Throttling that does not clear

**What this playbook is about is throttling that does not clear** — a namespace held at its limit
long
enough that the position never gets to move. Throttling that clears and throttling that does not
look exactly the same while they are happening; what separates them is how long each goes on, and
what accumulates meanwhile. Nothing in the error metrics or the
dead-letter queue will show you the difference, but **it is visible, just not where you would look
first**: [section 3](#3-why-a-stuck-task-produces-no-errors) explains why the obvious signals stay
clean, and [section 7](#7-detect-a-stalled-cleanup-on-the-dashboard) and
[section 8](#8-confirm-which-shards-and-which-namespace-are-affected) are how you detect and
confirm it.

A position that stays put for a long time leads to two very different situations, one after the
other, and they look nothing like each other:

| | What happens | What it leads to |
|---|---|---|
| **Cleanup quietly stops** | one task stops finishing, so the position stays where it is and no row behind it is removed | the table grows at whatever rate tasks are being written, for as long as it lasts. Workflows keep running normally throughout, so disk is the only thing that changes |
| **Cleanup restarts and cannot keep up** | the position moves again, but there is now so much to delete that the `DELETE` cannot finish inside the delete query timeout. It is cancelled, and removes nothing | the table keeps growing, and the database now spends real CPU, I/O and WAL on deletes that free no rows. The cost moves from disk alone to disk **and** database capacity, which your workloads are competing for |

### The delete query timeout

**The cleanup delete is given a fixed amount of time to finish, and anything still running when
that time is up is cancelled.** When the database is busy, or the range covers a very large number
of rows, the statement does not finish inside it — the server cancels it, and it frees nothing.

**The value is five seconds**, unchanged since 2022, and the
[note to readers](#note-to-readers) covers why it may not stay that way.

**A cancelled delete is retried quickly, not on the next 30-second interval.** The queue backs off
starting at 100 milliseconds and growing to a ceiling of its own, also **five seconds**, and it keeps retrying for as
long as the failure lasts. So a shard in this state makes an attempt roughly every ten seconds —
about five reaching the timeout, up to five backing off — rather than twice a minute.

While the timeout cannot be raised, **a delete that has grown too large has to be made smaller
another way**, and there are two. Removing rows leaves the same statement with less to do
([9.4](#94-deleting-task-rows-yourself-and-why-it-is-the-last-resort)). Reloading the
shard makes the server rebuild a small range and work up from there
([section 11](#11-reloading-shards-to-get-a-stalled-cleanup-running-again)). Several other
responses look right and are not —
[section 10](#10-what-not-to-do-when-cleanup-deletes-are-failing) covers those.

### Why a stalled cleanup is recoverable

**A stalled position can be detected long before the table matters, and detecting it there is the
whole point of this playbook.** A stalled position is quiet, and can go on for a long time with
nothing to show for it but a growing table — but the signals do exist.
[Section 7](#7-detect-a-stalled-cleanup-on-the-dashboard) is how to see a stalled position on the
dashboard, and [section 12](#12-alerting-on-a-stalled-cleanup) is how to be told about one
automatically.

**A cleanup delete that has already started timing out is also recoverable.** Timing-out deletes do
not clear by themselves — a span too large to delete inside the timeout stays exactly that large,
attempt after attempt — but the behaviour is understood and the remedy is known.
[Section 9](#9-get-task-row-cleanup-running-again) drains the backlog,
[section 11](#11-reloading-shards-to-get-a-stalled-cleanup-running-again) covers reloading the
affected shards, and [section 10](#10-what-not-to-do-when-cleanup-deletes-are-failing) covers what
not to try.

**Whether cleanup has stalled or its deletes are timing out, the goal is the same, and it is a
modest one:** get the row count down far enough
that an ordinary cleanup delete finishes inside the timeout again. Once a delete succeeds, the
position advances, and **the server resumes cleaning up on its own** with nothing further required
from you.

## Audience and scope

[History Persistence QPS Limits](./history-persistence-qps-limits.md) covers limits refusing
database calls, and [History Task Processing](./history-task-processing-tuning.md) covers a task
pipeline that is too slow. **This playbook is the third case: a pipeline that is completing its work
normally while nothing is being deleted behind it.** It also covers the checkpoint and
acknowledgement intervals that the task-processing playbook deliberately leaves out, because they
are what sets the cleanup cadence.

**Audience**

Operators of self-hosted Temporal clusters.

**This playbook does not cover**

- **Workflow history growth.** `history_node` and `history_tree` rows growing is a different
  subsystem with a different cause and a different cleanup mechanism — see
  [History Growth from Duplicate Workflow Starts](./history-growth-duplicate-workflow-starts.md) for
  orphaned history, and
  [XDC Standby Database Growth on SQL](./xdc-standby-database-growth-sql.md#how-cleanup-works-and-why-the-standby-can-grow-larger)
  for the history scavenger.
- **Matching task queues.** `approximate_backlog_count` and the task queue backlog metrics describe
  the queues your workers poll, which are a separate subsystem from the history task queues here.
  Matching backlogs and history task backlogs are easily confused because both carry the name.
- **The dead-letter queue.** Tasks that fail repeatedly and are set aside are a different problem —
  and as this playbook explains, a throttled task never gets there.
- **Database maintenance.** Reclaiming space after a large delete, vacuum behaviour and index bloat
  are your database's concern rather than Temporal's, though
  [section 9](#9-get-task-row-cleanup-running-again) says where they bite.

**Applies to** the history service, on any persistence store. Every mechanism here is in code shared
by all task categories and all store types; where a detail differs by category or by store, the text
says so.

---

## Contents

1. [Telling a real backlog from rows that were never deleted](#1-telling-a-real-backlog-from-rows-that-were-never-deleted)
2. [How the server decides which rows are safe to delete](#2-how-the-server-decides-which-rows-are-safe-to-delete)
3. [Why a stuck task produces no errors](#3-why-a-stuck-task-produces-no-errors)
4. [When a database outage leads to a stalled cleanup](#4-when-a-database-outage-leads-to-a-stalled-cleanup)
5. [Why some task types are the last to drain](#5-why-some-task-types-are-the-last-to-drain)
6. [Why a deleted namespace keeps its tasks alive](#6-why-a-deleted-namespace-keeps-its-tasks-alive)
7. [Detect a stalled cleanup on the dashboard](#7-detect-a-stalled-cleanup-on-the-dashboard)
8. [Confirm which shards and which namespace are affected](#8-confirm-which-shards-and-which-namespace-are-affected)
9. [Get task row cleanup running again](#9-get-task-row-cleanup-running-again)
10. [What not to do when cleanup deletes are failing](#10-what-not-to-do-when-cleanup-deletes-are-failing)
11. [Reloading shards to get a stalled cleanup running again](#11-reloading-shards-to-get-a-stalled-cleanup-running-again)
12. [Alerting on a stalled cleanup](#12-alerting-on-a-stalled-cleanup)
13. [Every setting named in this playbook](#13-every-setting-named-in-this-playbook)

---

## Fast track

**Working through a table that is growing right now**, these are the sections to take in order:

1. **[7.1 Start with the cleanup panels](#71-start-with-the-cleanup-panels)** — whether the rows
   are work still waiting to run or work that was never cleared away, and where nothing is being
   cleared, whether the server has stopped issuing deletes or is issuing them and having them fail.
2. **[8.1 Get the affected shards from the history service logs](#81-get-the-affected-shards-from-the-history-service-logs)**
   — narrows it from the whole cluster to specific shards, and to the namespace holding them up.
3. **[9. Get task row cleanup running again](#9-get-task-row-cleanup-running-again)** — clears
   whatever is stopping that namespace's tasks from finishing. Nothing else helps until this does.
4. **[11.6 How to reload the affected shards](#116-how-to-reload-the-affected-shards)**, or
   **[9.4 Deleting task rows yourself](#94-deleting-task-rows-yourself-and-why-it-is-the-last-resort)**
   — deals with rows the server can no longer remove on its own.

**Before trying anything from that list**, two responses that suggest themselves both make things
worse, and **[section 10](#10-what-not-to-do-when-cleanup-deletes-are-failing)** is why: waiting
for the cluster to catch up, and raising task throughput so the delete can get through.

**Set up in advance, a stalled cleanup announces itself**:
**[section 7](#7-detect-a-stalled-cleanup-on-the-dashboard)** for the dashboard panels, and
**[alert 91, Task Row Cleanup Failing](../observability/alerts/server/alerts-index.md#alert-91--task-row-cleanup-failing)**
for the one alert worth having.

---
## 1. Telling a real backlog from rows that were never deleted

A growing task table usually arrives as a disk alert, or as somebody noticing that one table has
become the largest in the database. The table is a task table — `timer_tasks` is the common one,
but `transfer_tasks` and
`visibility_tasks` behave the same way — and its row count has been climbing steadily for a long
time.

Then every check you would make comes back clean. None of these needs to be done by hand — each one
is a panel on the [server dashboard](../observability/dashboards/server/temporal-server-readme.md)
in this repository:

- **Workflows are completing normally.** Starts, completions and latency look like they always have
  — **[Workflow Success](../observability/dashboards/server/temporal-server-readme.md#11-workflow-stats)** and
  **[Actions per Namespace](../observability/dashboards/server/temporal-server-readme.md#1-cluster-throughput)**.
- **Task processing shows no errors, and nothing has been dead-lettered.**
  **[History Task Throughput](../observability/dashboards/server/temporal-server-readme.md#9-shard-queue-health)** is steady, and every panel in
  **[History Task DLQ / Terminal Failures](../observability/dashboards/server/temporal-server-readme.md#20-history-task-dlq--terminal-failures)** is empty.
- **The database is comfortable.** Latency, connections and refused calls are unremarkable on
  **[Persistence Latencies](../observability/dashboards/server/temporal-server-readme.md#3-persistence-requests-latencies-and-errors)** and
  **[Rejected Database Calls by Namespace](../observability/dashboards/server/temporal-server-readme.md#3-persistence-requests-latencies-and-errors)**.
- **No alert has fired**, and nothing in the logs stands out at the level you run at.

**A table growing while nothing is failing is the signature.** Every panel above is telling the
truth — there really is nothing wrong with workflow execution, with task processing, or with the
database. What has gone wrong is cleanup, and cleanup is not something a general-purpose cluster
dashboard tends to show at all. A cluster can therefore have perfectly good observability and still
not show that cleanup has stopped, which is the usual reason it is found late.

**Showing that cleanup has stopped is what the dashboard and alerts in this repository add.**
Cleanup has a
panel group of its own — **[History Task Cleanup](../observability/dashboards/server/temporal-server-readme.md#23-history-task-cleanup)** — charting the
deletes themselves: how long they take, how often they fail, and how often they succeed. Those
panels move while everything else stays flat.
[Section 7](#7-detect-a-stalled-cleanup-on-the-dashboard) walks through them and
[section 12](#12-alerting-on-a-stalled-cleanup) covers the alerts that go with them.

### 1.1 Why a task table's row count is not a measure of outstanding work

Both numbers get called "backlog", and telling them apart is the whole of this section. They are
counted in the same units, they sit in the same table, and only one of them is work:

| | What it is | What a large number means |
|---|---|---|
| **Rows in the task table** | every task row ever written for that shard and category that has not yet been deleted — **finished and unfinished alike** | either a lot of outstanding work, or a lot of finished work whose rows were never removed. On its own it does not say which. |
| **Pending tasks** | tasks the server has not finished yet | real outstanding work. The server is behind, and the question is throughput. |

The gap between row count and pending-task count exists because **processing a task does not delete
its row**. A row only goes away when the bulk delete described in
[section 2](#2-how-the-server-decides-which-rows-are-safe-to-delete) removes it. So a table can
hold a very large number of rows while the server has almost nothing left to do.

**Reading the row count as pending work sends you after the wrong thing.** Read a large table as a
backlog and the
obvious response is to make task processing faster — more throughput, higher limits, more capacity.
If the rows are finished work, none of that removes a single row, and raising throughput can make
the situation actively worse ([section 10](#10-what-not-to-do-when-cleanup-deletes-are-failing) covers why).

Telling retained rows from pending work needs something to compare the row count against. **1.2
works out the most
pending tasks a cluster can have at once**, which turns out to be a matter of arithmetic rather
than measurement.

### 1.2 How many pending tasks a cluster can have at once

How many pending tasks a cluster can have is settled by arithmetic rather than by measurement.

A workflow can set any number of timers, but the server does **not** write a task row for each one.
For each running execution it writes a task only for the **earliest unfired timer of each kind**.
When that one fires, the next is written. So at any moment one running execution holds at most:

- one **user timer** task — whatever your workflow code slept on, however many sleeps it has queued
  behind it,
- one **activity timeout** task,
- plus the workflow-level timeouts: workflow task timeout, run timeout and execution timeout.

**Timers with no task row of their own are not lost, and not held in memory.** Every pending timer
a workflow
has set is recorded in that workflow's **mutable state**, which is stored in the database like the
rest of the execution. A task row is not where a timer lives — it is a wake-up note saying
something needs attention at that time. When the earliest timer fires and its task is processed,
the server looks at what is left, and writes a task row for the new earliest. If the history pod
owning the shard is lost, the shard is reloaded elsewhere and mutable state is read back with every
pending timer still in it.

**Timers that come due together are fired together.** Processing one task fires every timer of that
kind that has already expired, in a single update — so ten timers set for the same moment cost one
task row and one pass, not ten. Timers set for ten *different* moments cost ten rows, written one
after another as each fires, but still only one outstanding at a time.

In practice that is **about four timer rows per running execution**, and on a test cluster it was
exactly that: 2,000 running workflows, each holding several timers, produced roughly 8,000 timer
rows.

**The pending count follows how many executions are running, not how many timers they have set.**
A thousand workflows each waiting on a thousand timers do not put a million rows in the task table.
They put about four thousand — the same as a thousand workflows waiting on one timer each. The
timers that have not had a row written yet are sitting in mutable state, costing nothing in the
task table until their turn comes.

That bound is what makes the comparison in 1.3 possible at all. The pending side of a task table
can be worked out from something you already know — how many workflows are running — so a row count
far above it has to be rows that were never deleted.

The other categories are bounded differently but just as tightly. Transfer, visibility and outbound
tasks have no fire time — their queues pick them up as soon as they are written, rather than waiting
for a clock. So the pending population is only whatever has arrived since the queue last read, which
depends on your current traffic rate and not at all on how long the cluster has been running.

**1.3 sets your row count against that pending-task figure** and says where each answer sends you
next.

### 1.3 Deciding between a real backlog and retained rows

**The tables to count are the history task tables.** The long-standing categories have one table
each — `timer_tasks`, `transfer_tasks` and `visibility_tasks`. Categories added later share two
generic tables, `history_scheduled_tasks` and `history_immediate_tasks`, with a `category_id`
column saying which category a row belongs to; archival and outbound tasks live there.

Count whichever table is growing. `timer_tasks` is the usual answer, because a timer row is written
when the timer is set and is not processed until its fire time arrives — so timer rows legitimately
sit in the table for as long as the workflow is waiting, where transfer and visibility rows are
picked up as soon as they are written.

**Take the row count from your database's table-statistics view rather than running `COUNT(*)`.**
On a table that has grown to this size a full count is a long scan, and the database is often
already under pressure by the time anybody looks. An approximate figure is all this comparison
needs. [Section 8](#8-confirm-which-shards-and-which-namespace-are-affected) has the exact queries,
including the per-shard and per-namespace breakdowns.

**Then work out roughly how many pending rows you should have**: a few rows for every workflow
execution currently running, using the figure from 1.2. You do not need to be precise, because a
real backlog and a table of retained rows are orders of magnitude apart:

- **The row count is in the same order of magnitude as that pending-row estimate.** This is a real
  backlog of pending work. The server is behind and the question is throughput —
  [History Task Processing](./history-task-processing-tuning.md) is the playbook for that, and
  [History Persistence QPS Limits](./history-persistence-qps-limits.md) if database calls are being
  refused. **Nothing in the rest of this playbook applies.**
- **The row count is far beyond that pending-row estimate** — tens or hundreds of times larger, or
  growing while the number of running workflows stays flat. The extra rows are finished work that
  was never deleted. **Cleanup has stopped**, and that is what the rest of this playbook is about.

**A table holding a real backlog and retained rows at the same time is the likeliest shape of
all**: a cohort of genuinely stuck tasks, plus everything written since, retained behind them.
[Section 8](#8-confirm-which-shards-and-which-namespace-are-affected) has a single read-only query
that separates stuck work from retained rows by looking at how the rows are spread over time. It is
worth running before any remediation, because it decides whether you still have work to drain or
only rows to remove.

**The dashboard will tell you which way a backlog is moving, but not how big it is.** No metric
counts task rows or pending tasks on any queue — every signal the server emits about queue state is
an age or a distance. That is enough to trend a backlog, and two panels are built for it:

- **[Immediate Queue Backlog Age by Category](../observability/dashboards/server/temporal-server-readme.md#9-shard-queue-health)** — the age of the oldest
  task still waiting, for transfer, visibility and outbound. Rising means the queue is falling
  behind; falling means it is catching up.
- **[Task Load Latency by Task Type](../observability/dashboards/server/temporal-server-readme.md#9-shard-queue-health)**, read at **p95** — the timer
  equivalent, because a timer loaded after its fire time reports exactly how overdue it is. One
  caveat worth knowing now: it only records when a task is **loaded**, so a queue that has stopped
  loading goes quiet rather than high.

Use both panels to tell whether a backlog is shrinking or still growing while you work.
**Sizing still means querying the database directly**, which is why this section is arithmetic and
[section 8](#8-confirm-which-shards-and-which-namespace-are-affected) is SQL.

**You can now tell a real backlog from rows that were never deleted, and you know which playbook
each one belongs to.** What neither tells you is why the rows were never removed in the first place.
**[Section 2](#2-how-the-server-decides-which-rows-are-safe-to-delete) is the deletion mechanism itself** — the rule the server follows to decide which rows
are safe to remove, and why one unfinished task stops cleanup for every namespace on its shard.

---

## 2. How the server decides which rows are safe to delete

**On a cluster running more than one namespace, this section is where the blast radius comes
from.** A shard carries tasks for every namespace whose workflows land on it, and all of those
namespaces share a single deletion watermark. A limit that applies to one namespace can therefore
stop row cleanup for all of them.

**A limit you set yourself and forgot does the same thing.** A namespace rate limit lowered to
contain
noisy traffic, and left in place after that traffic grew, holds the namespace's tasks back
indefinitely — its oldest unprocessed task never clears, so the watermark never moves past it. No
outage caused it, nothing recovers it on its own, and the rows accumulate for every other namespace
sharing those shards. The cluster looks healthy throughout.

The introduction gave you one position per queue. This section is how the server arrives at that
position — readers, scopes and the watermark — which is what turns the blast radius from a surprise
into something you can predict and check. It also gives you the vocabulary that
[section 8](#8-confirm-which-shards-and-which-namespace-are-affected) uses when you go and look at
a real shard.

### 2.1 A queue works through several ranges at once

**The queue here is the one described in the introduction: the in-memory processor for one task
category on one shard** — the timer queue on shard 412, say. Everything in this section happens
inside one of those. A cluster runs one per shard per category, each working independently.

Such a queue does not walk its task table from oldest to newest in a single line. It divides the
work it has left into **scopes**, and hands groups of scopes to **readers**:

- A **scope** is one stretch of the task sequence the queue still has to get through.
- A **reader** owns an ordered list of scopes and works through them.

**A reader belongs to one queue, so it is per shard and per category too** — not per history pod.
A pod owning 200 shards is running 200 timer queues, each with its own readers, none of which knows
about the others. The one thing those readers do share across the pod is a rate limit on how fast
they may load tasks out of the database, which
[section 11](#11-reloading-shards-to-get-a-stalled-cleanup-running-again) returns to.

Most of the time a queue has exactly one reader, called the default reader, holding one scope that
advances as work is completed. The queue creates a second reader when one namespace builds up a lot
of pending work, moving that namespace's tasks onto a second reader so they do not hold up
everything else in the default reader. [Section 4](#4-when-a-database-outage-leads-to-a-stalled-cleanup) covers when and
why a second reader appears, and
[section 8](#8-confirm-which-shards-and-which-namespace-are-affected) shows you how to list the
readers and scopes on a real shard.

**The server's name for several readers sharing one queue is multi-cursor** — each reader carries
its own cursor through the same task sequence. The term is not in any user-facing document, but it
is in the dynamic config settings that cap how many readers a queue may have, one for each
category:

| Setting | Default |
|---|---|
| `history.timerQueueMaxReaderCount` | 2 |
| `history.transferQueueMaxReaderCount` | 2 |
| `history.visibilityQueueMaxReaderCount` | 2 |
| `history.archivalQueueMaxReaderCount` | 2 |
| `history.outboundQueueMaxReaderCount` | 4 |

So a timer queue can have the default reader and **one** more, and nothing beyond that.

The outbound queue is the exception, and starts with four. Outbound tasks are the ones that call
**out of the cluster** — Nexus operations and callbacks — so unlike every other category they wait
on a system you do not control. It uses multi-cursor by design rather than in response to trouble,
keeping tasks for a slow destination on their own readers so they do not hold up tasks for
destinations that are responding normally.

**You can also see a shard's readers and scopes directly, with `tdbg`.** `tdbg` is the
administrative command line tool that ships alongside the server in the admin-tools image; it
exists to inspect internal state that no ordinary API exposes. Two of its commands matter here:
`tdbg shard describe --shard-id <n>` prints a shard's readers and the scopes each one
holds, and `tdbg shard list-tasks` lists task rows in a given range.
[Section 8](#8-confirm-which-shards-and-which-namespace-are-affected) uses both and explains how
to read what they return.

You now know that a queue splits its remaining work into scopes and hands them to readers. What
that does not tell you is how much work any single scope holds. **2.2 answers that**, and the
answer is what makes it possible for one task to pin an entire shard.

### 2.2 A scope is a boundary and a filter, not an amount of work

A scope has two parts:

| Part | What it is |
|---|---|
| **Range** | from one point in the task sequence up to another. For timer tasks those points are fire times, so a range is a stretch of time. |
| **Predicate** | a filter saying which tasks inside that range this scope is responsible for — for example, only tasks belonging to one namespace. |

**Only the range says where a scope sits**, and it is the range that the deletion watermark is
taken from in 2.3. The predicate says nothing about position at all — it narrows which tasks inside
the range the scope is responsible for.

**So the width of a scope tells you nothing about how much work is in it.** A scope covering two
months can hold a handful of tasks, if its filter matches only a handful. A scope covering ten
seconds can hold thousands.

That distinction matters the moment you look at a real shard. When
`tdbg shard describe` shows you one shard's timer queue — one shard, one category — and one of its
scopes spans months, two readings suggest themselves and both are wrong:

- **"This shard's timer queue is months behind, so it must have an enormous amount left to do."**
  Not necessarily. The range is wide; the work inside it may be almost nothing, because the
  predicate may match only a few tasks.
- **"There is hardly anything in that scope, so it cannot be doing any harm."** This is the
  dangerous one. How much work a scope holds has no bearing on how much it holds *back*. One
  unfinished task is enough to stop cleanup for the whole shard.

**Why one task is enough is the subject of 2.3**, which is how the server decides what is safe to
delete.

### 2.3 The deletion watermark is the lowest starting point across all readers

Each reader's scopes are kept in order, so a reader's **first** scope is its oldest unfinished
work. The server takes the **lower bound of each reader's first scope**, and the **lowest of those
across every reader on that queue** becomes the **deletion watermark** for that queue on that
shard.

**On a healthy queue there is only one reader**, so that calculation is not interesting: the
watermark is simply the lower bound of the default reader's first scope, and it advances as work
is completed. Taking the lowest across several readers only starts to matter once a second reader
exists — which, as 2.1 described, happens when a namespace has built up enough pending work to be
moved to a second reader. **That is the situation this playbook is about**, so it is worth being
precise about
what the calculation does.

Everything older than the watermark is safe to delete, because no reader has any remaining work
below it. Everything from the watermark onward is left alone.

**Two consequences follow:**

- **One reader holding one old scope sets the watermark for the entire queue.** It does not matter
  how far the other readers have got, or how many scopes they have finished. The lowest wins.
- **It does not matter how little that scope contains.** A scope holding a single unfinished task
  contributes its lower bound exactly as a scope holding a million would. A single task can
  therefore hold the watermark for a shard — and, because task tables are not organised by
  namespace, for every namespace with tasks on that shard.

A single unfinished task holding the watermark is the mechanism behind the introduction's claim
that cleanup moves forward only as fast as the slowest unfinished task.

**One reader stuck and one reader healthy is the state that is easy to miss, and worth picturing
concretely.** One queue, two readers:
the default reader working through its scopes normally, and a second reader holding one old scope
whose tasks are not clearing. The default reader keeps completing work, so workflows in every other
namespace run exactly as they always have. But the watermark is the lower of the two readers'
positions, so it sits
at the stuck reader's scope and does not move — and no row behind it is deleted, for any namespace
on that shard.

Workflow completions, task errors, dead-lettered tasks and database latency all report health,
because everything they measure genuinely is healthy. The one thing that has stopped is row
deletion, and no general-purpose cluster dashboard
reports that at all. It is visible — **[History Task Cleanup](../observability/dashboards/server/temporal-server-readme.md#23-history-task-cleanup)** is the
panel group for it, and [section 7](#7-detect-a-stalled-cleanup-on-the-dashboard) is how to read
it — but only if something is watching for it specifically.

**What the server then does with the watermark is issue a single database statement**, and the
shape of that statement is why cleanup deletes can later start timing out.

### 2.4 One statement removes everything below the watermark

At each checkpoint, if the watermark has moved since the last one, the queue issues **one** `DELETE`
covering everything from the previous watermark up to the new one, for that shard and that task
category. For scheduled categories such as timers the range is expressed in fire times; for
immediate categories such as transfer and visibility it is expressed in task ids.

Three properties of that statement matter later:

- **It is a single statement with no row limit.** The server does not delete in batches and does not
  chunk the range. However many rows fall inside the range, one statement is expected to remove all
  of them.
- **The range is whatever accumulated since the last successful delete**, not a fixed window. If the
  watermark has not moved for a long time and then moves a long way, the next statement covers all
  of it at once.
- **It is bounded only by the delete query timeout**, which cannot be changed by configuration.

On a healthy cluster none of those three properties is noticeable. The watermark advances a little
every thirty seconds, each delete removes a small number of rows, and the statement finishes in
milliseconds.
The properties above only start to matter once the watermark has been held still for a long time,
which is [section 10](#10-what-not-to-do-when-cleanup-deletes-are-failing).

**The three properties above are current behaviour, not settled design** — an open issue proposes
changing the two that make recovery hard, the single statement and the fixed timeout — see
[the note to readers](#note-to-readers).

**You now know how a task row is removed, and why one unfinished task anywhere in a queue stops
every row behind it from being deleted.** What that does not explain is why the task holding the
watermark does not show up as a problem — no error, no dead-lettered task, no alert.
**[Section 3](#3-why-a-stuck-task-produces-no-errors) is why a stuck task produces no errors**, and which signals do move when everything
else stays flat.

---

## 3. Why a stuck task produces no errors

A task held at a rate limit is **refused**, and a refusal is not a failure. The server treats the
two differently at every stage — what it counts, what it retries, what it gives up on — and the
result is that the task holding your watermark leaves nothing in `task_errors`, nothing in the
dead-letter queue, and no line in the logs. Counters do record it — just not the ones most error
panels and alerts are built on.

This section is why that happens, and what does move instead.

### 3.1 Two different limits refuse a task, at two different moments

Two kinds of limit can refuse a task — **scheduler admission** and **persistence** — and they do
it at different points. Which one is refusing decides which signal you will see.

**"Refused" and "throttled" mean the same thing here.** This playbook says *refused* because it is
the plainer word; the server says *throttled*, which is why its metrics are named
`task_scheduler_throttled` and `task_errors_throttled`. Neither word means the task failed.

| | Where the task is refused | What it costs |
|---|---|---|
| **Scheduler admission** | before the task runs at all. The scheduler paces how fast each namespace's tasks may be dispatched, and over that rate it simply declines to start them. | nothing. The task has done no work, touched no database, and holds no resources. |
| **Persistence rejection** | after the task has started. It ran, read what it needed, and its database call was refused by a persistence limit. | the read it already did is thrown away and repeated on the next attempt. |

**Persistence rejection is the expensive one**, and it is the subject of
[History Task Processing](./history-task-processing-tuning.md) — a task that reads, is refused, and
starts again from the read is doing the same work repeatedly. For this playbook the important point
is that **both end the same way**: the task is put back, tried again, and never finishes.

**Neither kind of refusal counts as a failure**, which is what the rest of this section is about.

### 3.2 What stops a task finishing, and what does not

This playbook keeps saying a task "cannot finish". It is worth being precise about what that means,
because the obvious reading — the task is failing — is the wrong one.

**A task stops holding the watermark in three ways:**

- **It succeeds.** It is acknowledged and done.
- **It fails repeatedly with an unexpected error.** After a set number of such attempts the server
  gives up on it: the task is sent to the dead-letter queue, or dropped if that is disabled.
  Dead-lettered or dropped, it is **acknowledged** either way, and it stops holding the watermark. It is preserved for you to
  inspect and replay, but it is no longer in the way.
- **Its target no longer exists.** A task whose workflow or namespace cannot be found is dropped
  and acknowledged immediately, on the grounds that there is nothing left to do.

**A task keeps holding the watermark when the server classifies its error as expected and
retryable.** There are a handful of those, and a refused task is the common one:

| Condition | What it means |
|---|---|
| **Resource exhausted** | refused by a rate limit — the subject of this section |
| **Namespace not active** | the namespace is active in a different cluster |
| **Dependency not completed** | the task is waiting on another task |
| **Standby retry** | a standby task waiting for replication to catch up |
| **Namespace handover** | the namespace is mid-handover |

**Expected-retryable conditions are not counted as failures**, so none of them advances the
attempt counter, and the task is retried for as long as the condition lasts.

**The result is the opposite of what you would expect: a task that genuinely fails eventually gets
out of the way, and a task that is merely being deferred never does.** Failure is bounded — seventy
attempts and the server sets the task aside. Deferral is not bounded at all.

### 3.3 What a refusal is not counted as

The distinction is made in one place in the server, and everything follows from it. When a task
returns a resource-exhausted error, that error is classified as **expected and retryable**, and the
code returns immediately — before reaching any of the handling that applies to real failures.

Three things are therefore skipped:

- **It is not counted in `task_errors`.** Refusals go to separate counters instead, so the panels
  built on `task_errors` — **[Total Timer Tasks Errors](../observability/dashboards/server/temporal-server-readme.md#10-history-timer-task-info)** among
  them — stay flat no matter how long the condition lasts, as do any alerts keyed on them. The
  refusal is not invisible — a persistence rejection is recorded on
  `persistence_errors_resource_exhausted`, tagged with which limit refused it, and the task layer
  records `task_errors_throttled`. **It is simply not in the error family**, which is where
  attention goes. 3.4 names all three counters and the panel for each.
- **It does not advance the attempt counter.** Tasks are sent to the dead-letter queue after a
  number of *unexpected* errors. A refusal never increments that count, so **there is no number of
  refusals that will ever dead-letter a task.** The attempt limit is not merely high; it is not
  reachable by this path.
- **Nothing is logged.** The throttle path records its counter and returns. Neither kind of refusal
  writes a log line at any level, so there is nothing to grep for.

  **One log line does exist, but it belongs to the failing-delete half of this playbook rather
  than to a throttled task.** When a cleanup
  delete starts failing, the queue logs `Error range completing queue task` — and it logs it on
  the shard's own logger, so every failing shard names itself. That is the most precise signal
  available anywhere in this playbook, and
  [section 8](#8-confirm-which-shards-and-which-namespace-are-affected) uses it to turn an
  open-ended cleanup into a targeted one. It says nothing about the throttled task, though, and it
  appears only once deletes are being attempted and timing out — so during the quiet phase
  described here, the logs are silent.

**Scheduler admission leaves even less behind**, because the task never ran: it appears only on
`task_scheduler_throttled`, and in no error metric of any kind.

**And the retrying does not wind down.** The retry interval grows and then caps, and the retry
policy has no expiry — so a task refused for a month is still being tried, at the same cadence it
reached in the first few minutes.

**So the error metrics and the dead-letter queue are genuinely clean, and no log mentions the
throttled task. 3.4 is where to look instead.**

### 3.4 Which signals do move

| Signal | Panel | What it means |
|---|---|---|
| `task_scheduler_throttled` | **[Task Scheduler Throttling by Namespace](../observability/dashboards/server/temporal-server-readme.md#9-shard-queue-health)** | tasks the scheduler declined to dispatch, grouped by namespace. Admission refusals appear here and nowhere else — the task never ran, so no persistence metric sees it. |
| `persistence_errors_resource_exhausted` | **[Rejected Database Calls by Namespace](../observability/dashboards/server/temporal-server-readme.md#3-persistence-requests-latencies-and-errors)** | calls that were dispatched and then refused by a persistence limit, grouped by namespace, with `resource_exhausted_scope` saying which limit did it — which tells you which setting to change. |
| `task_errors_throttled` | **[Throttled Tasks by Task Type and Namespace](../observability/dashboards/server/temporal-server-readme.md#9-shard-queue-health)** | the same persistence rejections seen from the task side, by **task type**. `task_type` carries the category as a prefix, so this is the panel that tells you a *timer* task was refused rather than a transfer one. |

**A non-zero reading on either throttle panel is not a fault.** Once scheduler limits are
configured, throttling is how the server paces work to what the database can take, so
**Task Scheduler Throttling by Namespace** showing a steady non-zero rate is a healthy cluster
doing its job. There is no value on that panel that means "broken", and an alert on it rising
above zero would fire constantly and tell you nothing.

**What marks a stuck namespace is that its throttling does not end.** Not that it is large.
[Section 2.3](#23-the-deletion-watermark-is-the-lowest-starting-point-across-all-readers) is the reason: the watermark is held by the oldest task that has not finished, and one
task is enough. A namespace throttled at a modest rate that never clears pins a shard exactly as
firmly as one throttled at a huge rate. **Do not wait for a dramatic number** — the number is not
what makes it harmful.

Read these three panels over **hours or days**, not minutes:

| Panel | What to look for |
|---|---|
| **[Task Scheduler Throttling by Namespace](../observability/dashboards/server/temporal-server-readme.md#9-shard-queue-health)** and **[Throttled Tasks by Task Type and Namespace](../observability/dashboards/server/temporal-server-readme.md#9-shard-queue-health)** | a namespace being throttled **continuously**, rather than in bursts that subside. Any namespace whose line never returns to its baseline is a candidate, at whatever rate. |
| The same two panels, next to **[RPS per Namespace](../observability/dashboards/server/temporal-server-readme.md#1-cluster-throughput)** | **a flat throttle line while that namespace's own traffic rises and falls.** Normal throttling tracks load: busy periods throttle more, quiet periods less. A throttle rate that holds steady while the namespace's request rate moves around it means work is not getting through, rather than arriving faster than usual. |
| The same comparison, in its clearest form | **throttling that continues while the namespace's traffic is at or near zero.** Nothing is driving new work, yet tasks are still being refused — so what is being retried is old work that cannot complete. This is the least ambiguous reading available on these panels, and it is what a namespace whose workflows have stopped, or that has been deleted, looks like. |
| **[History Task Throughput](../observability/dashboards/server/temporal-server-readme.md#9-shard-queue-health)** | every *other* namespace completing tasks at its usual rate. This rules out the alternative explanation: a cluster genuinely short of capacity slows everything down, not one namespace. |

**A namespace that stands out from the rest is not required.** On a cluster where one namespace
dominates the traffic anyway, or where the limits are tight for everyone, the stuck namespace may
look no different from its neighbours — and the readings above still hold.

**These panels tell you which namespace, not whether you have a problem.** What tells you there is
a problem is the task table growing while cleanup does not keep up, which is
[section 1](#1-telling-a-real-backlog-from-rows-that-were-never-deleted) and
[section 7](#7-detect-a-stalled-cleanup-on-the-dashboard). Come back here once you know cleanup has
stalled, to find out who is holding it.
[Section 12](#12-alerting-on-a-stalled-cleanup) covers what on these panels can and cannot be
alerted on.

**Three things to carry out of this section.** A refused task is recorded nowhere in the error
metrics, the dead-letter queue or the logs. It is also never abandoned — the retry policy has no
expiry and a refusal never counts toward the dead-letter threshold, so the condition will not end
by the server giving up. And the panels in 3.4 will tell you *which* namespace is holding a queue,
once you already know that a queue is being held.

**What none of that explains is how a namespace comes to be held at its limit in the first
place.** Nothing above says why one
namespace would suddenly have far more outstanding work than its neighbours, nor why that state
outlives whatever caused it. **[Section 4](#4-when-a-database-outage-leads-to-a-stalled-cleanup) is the onset**: how a database outage leaves one namespace
with a backlog it cannot work off, how the server then moves that namespace's tasks onto a second
reader, and why the cleanup position stays exactly where it is long after the outage has ended and
the cluster looks healthy again.

---

## 4. When a database outage leads to a stalled cleanup

A database outage is the most common way a cluster arrives at a stalled cleanup, and the steps
after the onset are the same whatever caused it. The other way in is a per-namespace limit left
below what a namespace sustains — the two refusals from [section 3.1](#31-two-different-limits-refuse-a-task-at-two-different-moments), covered in
[History Task Processing](./history-task-processing-tuning.md) and
[History Persistence QPS Limits](./history-persistence-qps-limits.md), which also explain how to
size them. Reviewing those limits against current traffic periodically is worth doing: a limit that
was right when it was set can be well short of what a namespace needs a year later.

**What happens, in four steps:**

| | What happens | Is anything wrong? |
|---|---|---|
| **1** | During the outage, one or more namespaces build up a large number of unfinished tasks. | No. It is real work that genuinely has to be done. |
| **2** | Once a namespace crosses a pending-task threshold, the server moves its tasks onto a second reader — all such namespaces onto the same one — so they stop holding up every other namespace on the shard. | No. It is a deliberate protection, and it works. |
| **3** | The other namespaces go back to working normally. | No. Their recovery is complete. |
| **4** | The tasks on that second reader still cannot finish, so the deletion watermark stays where they are and no rows are deleted — for any namespace on that shard. | Nothing has malfunctioned. Cleanup has stopped all the same. |

**Step 4 is the same tasks as step 1, seen from row deletion's side.** Moving them to a second
reader takes them out of the way of *task processing*, which is why every other namespace recovers.
It does nothing for *row deletion*, which removes rows strictly older than the oldest unfinished
task on the whole queue, both readers included. The tasks that have stopped slowing anyone down are
still the oldest unfinished tasks.

**The threshold in step 2 is 500 pending tasks, and it is easy to mistake it for the thing that
stops cleanup. It is not:**

| | What it does |
|---|---|
| **The oldest unfinished task not finishing** | **stops row deletion.** No threshold is involved — one task is enough, on whichever reader it sits. |
| **The 500-task threshold** | **decides whether the namespace is moved to a second reader.** It changes how visible the stall is and how likely it is to clear. It neither causes a stall nor is required for one. |

So a namespace well below 500 pending tasks can stall cleanup if one of its tasks cannot finish,
and a namespace that crosses 500 and then drains normally stalls nothing at all.

### 4.1 An outage leaves busy namespaces with far more outstanding work than usual

While the database is refusing or timing out calls, the server cannot record state transitions —
an activity completing, a workflow task finishing, a signal being applied, a child workflow
starting. Executions stay where they were, and every running execution keeps its timeout timers
live. So pending tasks rise in proportion to **how much work a namespace had in flight when the
outage began**, and every namespace busy at the time is affected, each in proportion to its own
concurrency.

**Most outages end here.** The database returns, the outstanding tasks complete, the watermark
catches up, and the accumulated rows are deleted in the ordinary way. What follows is about the
cases where those tasks cannot complete — which, as 3.2 set out, means being refused and retried
indefinitely rather than failing.

### 4.2 The server moves namespaces that fall far enough behind onto a second reader

A queue tracks pending tasks per **namespace**, and moves a namespace off the default reader once
it crosses a threshold:

| | |
|---|---|
| **Threshold** | `history.queueMoveGroupTaskCountBase`, default **500** pending tasks, counted **per shard**. A dynamic config setting, not a compiled-in constant. |
| **How many moves** | One. A timer queue allows two readers, so a namespace goes from reader 0 to reader 1 and there is nowhere further. The multiplier setting governs a move to a third reader, which does not exist here. |
| **How many namespaces** | Any number. The cap of two is on *readers*, not namespaces — every namespace over the threshold moves in one pass, onto the same reader 1. |
| **Logged?** | Yes, at `INFO`: `Too many pending tasks, moving group to next reader`, with the reader id, the pending count and the namespace. Clusters running at `WARN` do not get it. |
| **Worth tuning?** | **No, not for this problem** — see below. |

**Moving a namespace onto reader 1 is protective, and it works.** A large backlog left in the
default reader would consume
the capacity that reader needs to load tasks for everyone else on the shard.

**Neither the 500-task threshold nor the two-reader cap is a lever for a stalled cleanup**, though
each looks like one. The watermark is the lowest position across *all* readers, so no reader
setting changes
which task is oldest:

- **Raising the 500-task threshold** leaves the backlog in the default reader. The stall is
unchanged — the
  same task is still oldest — and now every other namespace on the shard is slowed down behind it.
  That makes a stalled cleanup easier to notice and worse to live with.
- **Allowing more readers** gives a namespace somewhere further to be moved to. But readers are
  served in id order, so each level down is served later than the last: the namespace drains more
  slowly, and its oldest task still sets the watermark.

**Try letting the stuck tasks complete before anything else**, which means the
per-namespace limits in [section 3.1](#31-two-different-limits-refuse-a-task-at-two-different-moments) — raised enough that the backlog drains rather than merely
inching forward. Deleting rows from the task table yourself is the fallback, for when raising
limits is not enough or the cleanup deletes have already started timing out:
[section 9](#9-get-task-row-cleanup-running-again) covers it, and how to do it without destroying other
namespaces' tasks.

**How likely a namespace is to cross 500 pending tasks on a shard depends on how many workflows
it runs at once.** [Section 1.2](#12-how-many-pending-tasks-a-cluster-can-have-at-once) established that each running execution accounts for roughly four
pending tasks, so for a namespace spread evenly across shards:

> pending tasks on one shard ≈ (concurrent executions × ~4) ÷ number of shards

Crossing 500 therefore needs about `125 × number of shards` concurrent executions in one namespace
— roughly a quarter of a million on a 2,048-shard cluster, sixty thousand on a 512-shard one.

**A namespace can cross 500 on a shard with far fewer concurrent executions than a quarter of a
million.** Workflows are assigned to shards by hashing their workflow id, so a workload
concentrated on a small set of ids puts far more of its executions onto a few shards than an even
split would. Those few shards reach 500 while the namespace as a whole is running far fewer
workflows at once than an even spread would require.
[Hot Shard Detection and Remediation](./hot-shard-detection-remediation.md) covers uneven shard
load. **And a cluster that never reaches 500 on any shard is not thereby safe** — being moved to a
second reader changes how visible a stall is and how likely it is to clear, not whether one can
happen at all.

### 4.3 The default reader races ahead while reader 1 holds the watermark

With the affected namespaces on reader 1, the default reader catches up quickly and every namespace
still on it returns to normal. **Reader 1 is not parked** — it keeps loading and running its own
tasks throughout. It is simply served after the default reader, which 4.4 returns to. But the
watermark is the **lower** of the two readers' positions, so
it sits at the oldest unfinished task on reader 1 and stays there. If several namespaces were
moved, the slowest of them alone decides the watermark — for all of them, and for every namespace
still on the default reader.

**While the watermark is held, no delete is attempted at all.** Everything below it was removed by
earlier deletes; each checkpoint finds the watermark unmoved and issues no statement. Rows written
in the meantime sit *above* it, where deletion never looks. **A stalled shard costs the database
nothing while it is stalled** — nothing about it is expensive until it starts to recover.

**A delete that large only happens when the watermark moves, and it does not move gradually.** The
tasks behind the
stuck one were completing normally the whole time — the position was held by a single unfinished
task, not by a queue of them. When that one task finishes, the position jumps straight past
everything already completed, to the next task still outstanding. On a queue that was held for
weeks, that is a jump of weeks in one step.

**So the enormous delete does not mean processing suddenly sped up.** It is the opposite: weeks of
*already-completed* work becoming eligible for deletion at once, because the one row that was
blocking it went away. The delete covers everything from the old watermark to the new one, so its
size is set by how long the position was held — not by how fast anything is running when it
releases.

**[Section 10](#10-what-not-to-do-when-cleanup-deletes-are-failing) covers what happens when that single
delete is too large to finish inside the delete query timeout.** The cause is worth stating carefully, because
it is easy to misread: the recovery is not what went wrong. The oversized delete is the
accumulated cost of the stall, and it only becomes visible once the queue starts working through
it. The thing that needs fixing is still the task that could not complete — everything else
follows from it.

**A namespace's workflows are spread across every shard, so the four steps above play out on many
shards at once.** Crossing 500 on one shard usually means crossing it on many, and each of those
shards reaches the same end state on its own. The row count you eventually notice in `timer_tasks`
is the sum across all of them.

### 4.4 Why the position may not recover once the database does

Once the database is healthy again, the namespaces on reader 1 still have a backlog several hundred
tasks deep on each shard to work through. **Most of the time they work through it.** Three things
make that slower on reader 1 than it would have been on the default reader:

| What works against them | Why it is there |
|---|---|
| **Their tasks are still being refused** | a backlog is by definition more work than the namespace normally does, so it runs straight into the two per-namespace limits from 3.1 — scheduler admission and persistence. A refused task never fails, never expires and never reaches the dead-letter queue, so it is retried for as long as the limit holds. |
| **Reader 1 is served last when *loading* tasks** | every reader on a history pod draws from one shared allowance for reading tasks out of the database, set by `history.timerProcessorMaxPollHostRPS` (0 by default, meaning it is derived from the pod's persistence limit). That allowance is handed out by reader id, so reader 0 is satisfied before reader 1 gets any. If several namespaces were moved to reader 1, they divide what reader 1 receives between them. **This governs reading tasks in, not the cleanup `DELETE`** — that statement is unaffected. |
| **Reader 1 holds fewer tasks in memory at once** | a non-default reader is given a lower ceiling on how many tasks it may hold pending, deliberately, so that a reader with a large backlog cannot fill a history pod's memory and force loaded work to be discarded. One ceiling, shared by every namespace on reader 1. |

**Refusal, loading order and the lower ceiling are each deliberate, not faults.** The limits
protect the database, the loading order protects the namespaces that are keeping up, and the lower
ceiling protects the history pod's memory. **What they change is the margin** — reader 1 works
through a backlog more slowly than the
default reader would, so a backlog that would have cleared on the default reader may not clear
here.

**A backlog stops clearing altogether in one of two situations:** a namespace's tasks drain more
slowly than its workflows create new ones, or its tasks cannot complete at all — which is what
[section 6](#6-why-a-deleted-namespace-keeps-its-tasks-alive) covers. Then reader 1 never empties
and the watermark never advances. **Only one namespace has to be stuck**: the others can drain
completely and it makes no difference to the watermark.

When that happens, the outage is over, the cluster is healthy, and cleanup has stopped. **No part
of the server will resolve it, so it does need finding — but finding it is what the rest of this
playbook is for, and every step from here is something you can do.**

[Section 7](#7-detect-a-stalled-cleanup-on-the-dashboard) is how to see it on the dashboard and
[section 12](#12-alerting-on-a-stalled-cleanup) is how to be told about it automatically, so that
next time it is caught early rather than found by a disk alert.
[Section 8](#8-confirm-which-shards-and-which-namespace-are-affected) gives you the exact shards
and namespace involved, and [sections 9](#9-get-task-row-cleanup-running-again) and
[10](#10-what-not-to-do-when-cleanup-deletes-are-failing) are how to clear it. **The server resumes cleaning
up by itself once the backlog is gone** — none of this is a state you have to manage permanently.

**You now know how a short outage can leave cleanup stopped until someone clears it**: a backlog
builds, the server moves it onto a second reader to protect everything else, and the moved work
then holds the watermark while being served last.

**What those four steps treat as interchangeable is the tasks themselves.** Nothing in them
explains why one task in a backlog sits unfinished far longer than the task next to it — and since
the watermark is held by whichever task has gone unfinished longest, that difference is what
decides which task ends up holding it.

**[Section 5](#5-why-some-task-types-are-the-last-to-drain) is task priority.** Different task types are given different shares of the scheduler's
capacity, with the work that cleans up finished workflows given the smallest share — the right call
in itself, since work your users are waiting on should go first. It is also why a task of one of
those types is far likelier than any other to be the one holding the watermark — and with it, the
deletion of every row written after it, for every namespace on that shard.

---

## 5. Why some task types are the last to drain

[Section 2](#2-how-the-server-decides-which-rows-are-safe-to-delete) established the rule everything else rests on: **the deletion watermark sits at the
oldest task that has not finished.** [Section 3](#3-why-a-stuck-task-produces-no-errors) and
[section 4](#4-when-a-database-outage-leads-to-a-stalled-cleanup) covered one reason a task does
not finish — it is being refused by a limit.

**A task can also stay unfinished simply by being outvoted.** The history task scheduler sorts every
task into one of three priority classes by its type and gives each class a share of its capacity.
A task in the smallest share is not refused by anything — it is simply asked to wait while other
work goes first, over and over, for as long as there is other work to do.

**That makes the lowest-priority tasks the likeliest candidates to be the one holding your
watermark**, and it is a route in that needs no outage and no misconfigured limit. A busy cluster
is enough on its own.

**Three words are easy to mix up, so to be exact about which is which:**

| Term | Examples | What it decides |
|---|---|---|
| **Category** | timer, transfer, visibility, archival | which queue works the task, and which table its row sits in. [Section 2](#2-how-the-server-decides-which-rows-are-safe-to-delete)'s subject. |
| **Type** | `ACTIVITY_TIMEOUT`, `DELETE_HISTORY_EVENT` | what the task actually does. Several types share a category: a timer queue carries activity timeouts and retention deletions alike. |
| **Priority class** | High, Low, Preemptable | what share of the scheduler's capacity the task competes for. |

**The type decides the priority class**, and the priority class decides the share. Category does
not come into it — two tasks in the same timer queue can be in different priority classes and
drain at very different rates.

### 5.1 The three priority classes, and what lands in each

| Priority class | Tasks assigned to it |
|---|---|
| **Preemptable** | the cleanup types — `DELETE_HISTORY_EVENT`, transfer and visibility delete-execution, and archival. Also **every task belonging to a namespace that is active in another cluster**, whatever its type, and any task type the server does not recognise. |
| **Low** | the timers that fire when something has **run out of time** — activity timeout, workflow task timeout, workflow run timeout, workflow execution timeout — plus worker commands, which the server sends to workers over Nexus. |
| **High** | everything else, which is most of what a cluster does: a workflow's own `sleep` expiring (`USER_TIMER`), an activity retry timer, handing a workflow task or an activity to a worker, starting a child workflow, delivering a signal, and the visibility records written when an execution starts or closes. |

### 5.2 What the shares actually are

The scheduler works through the priority classes on a weighted rotation. The default weights for
an active namespace:

| Priority class | Weight | Share of turns |
|---|---|---|
| **Preemptable** | **1** | **5%** |
| **Low** | 9 | 45% |
| **High** | 10 | 50% |

So a preemptable task gets roughly **one turn in twenty**, against ten for a high-priority one. A
backlog of cleanup tasks drains at about a tenth of the rate the same backlog of ordinary work
would.

**For a namespace whose work is handled by another cluster, every priority class is weighted 1** —
high, low and preemptable alike. Standby work is given no advantage over anything else, by design.

**The weights only bite when there is contention.** They decide who goes first when there is more
work than capacity, and nothing at all when there is not. On a queue that is keeping up, a
preemptable task runs as promptly as any other.

**The priority class costs you time, and the table gives you no sign of it.** A workflow's own
timer firing
is High; a retention deletion is Preemptable. Both are timer tasks, both sit in `timer_tasks`, and
nothing about the rows tells them apart. But a backlog of the second drains at roughly a tenth the
rate of a backlog of the first — so **two tables of identical size can take wildly different
amounts of time to clear**, and the row count alone will not tell you which you are looking at.
[Section 8](#8-confirm-which-shards-and-which-namespace-are-affected) is where you find out, by
reading the task types out of a sample of the rows.

### 5.3 How fast a backlog drains, by what it is made of

Three kinds of backlog turn up in a stalled cleanup, and priority treats them very differently.
Find yours here before deciding what to change:

| A backlog made of | Priority class | What to expect, and what to do |
|---|---|---|
| **Timeout timers** — the kind an outage leaves behind, as in [4.1](#41-an-outage-leaves-busy-namespaces-with-far-more-outstanding-work-than-usual) | **Low**, weight 9 of 20 | Almost the same share as ordinary work, so **priority is not what is holding it up**. The constraint is one of the two per-namespace limits from 3.1: scheduler admission, covered in [History Task Processing](./history-task-processing-tuning.md), or persistence, covered in [History Persistence QPS Limits](./history-persistence-qps-limits.md). Changing priority weights here achieves nothing. |
| **Retention cleanup** — `DELETE_HISTORY_EVENT` and delete-execution tasks, created as workflows pass their retention period or a namespace is deleted | **Preemptable**, weight 1 of 20 | For every twenty tasks the scheduler starts, one is this work. On a cluster with plenty of other work to do, the backlog creeps while everything else runs normally — and raising the per-namespace limits barely helps, because the limit is not what is refusing it. The lever is 5.4: raise the preemptable weight **for that namespace only**, or wait for a quieter period when there is less to compete with. |
| **Anything at all, for a namespace whose work is handled by another cluster** | **Preemptable**, and that namespace's weights are **1 / 1 / 1** | Two things combine: every one of its tasks is treated as preemptable whatever its type, and its own weight map gives all three classes equal weight. So nothing inside that namespace can be prioritised ahead of anything else, and the whole of it competes at the bottom alongside other namespaces' cleanup. Raising its weights changes nothing useful. The backlog drains when the competing load falls, or when the namespace becomes active in this cluster. |

**[Section 8](#8-confirm-which-shards-and-which-namespace-are-affected) tells you which of the
three you have** — it reads the task types out of a sample of the retained rows, and the mix is
what distinguishes a stuck outage backlog from stuck retention work.

**Knowing which one you have decides where to spend effort, because each lever is useless against
the other kind.** Raising per-namespace limits does little for retention work that is being
outvoted twenty to one — no limit is refusing it. Changing priority weights does nothing for a
timeout backlog that is already in the second-highest class. **5.4 is the weights**, and when
changing them is and is not worth doing.

### 5.4 Whether to change the priority weights

`history.timerProcessorSchedulerActiveRoundRobinWeights` sets the weights, with an equivalent for
each other category and a separate standby variant for each. **It takes a namespace constraint**,
which is what makes it usable here, and it is a hot reload — no restart.

**For an ordinary stalled cleanup, leave the weights alone.** They are not what stopped the
watermark, and raising the preemptable share cluster-wide takes capacity from work your users are
waiting on.

**Raising it is reasonable in one case: a namespace whose backlog is specifically cleanup work,
which you have decided to drain.** Three things produce that kind of backlog:

- **Retention expiry.** When a workflow passes its namespace's retention period the server emits a
  `DELETE_HISTORY_EVENT` task to remove its history. A namespace with high volume and short
  retention produces these continuously.
- **Namespace deletion.** Deleting a namespace creates a delete-execution task for every workflow
  in it — which is [section 6](#6-why-a-deleted-namespace-keeps-its-tasks-alive), and the case
  where this matters most.
- **Bulk workflow deletion**, through the API or a batch job, which creates the same task types.

All three land in the preemptable class, so on a busy cluster they get one turn in twenty whatever
else is true.

**A worked example.** Leave the cluster default alone and add one namespace-scoped override:

```yaml
history.timerProcessorSchedulerActiveRoundRobinWeights:
  # cluster default, unchanged
  - value:
      high: 10
      low: 9
      preemptable: 1
    constraints: {}
  # temporary: let one namespace's cleanup work compete while it drains
  - value:
      high: 10
      low: 9
      preemptable: 5
    constraints:
      namespace: 'orders-prod'
```

That takes the namespace's cleanup work from roughly 5% of turns to about 20%. **Raise it in steps
and watch what it costs**: the share has to come from somewhere, and on that namespace it comes
from the work its own users are waiting on. **Put it back when the backlog is gone** — there is no
reason to run with it permanently, and leaving it raised is the kind of forgotten setting
[section 2](#2-how-the-server-decides-which-rows-are-safe-to-delete) warned about.

[Section 9](#9-get-task-row-cleanup-running-again) covers using this alongside the other levers, in the order
that makes them safe.

**Three things now decide how long a task stays unfinished, and so how long it can hold the
watermark**: the per-namespace scheduler admission and persistence limits from
[section 3](#3-why-a-stuck-task-produces-no-errors), the reader it sits on from
[section 4](#4-when-a-database-outage-leads-to-a-stalled-cleanup), and the priority class its type
falls into. Each of the three makes a task take longer. **None of them stops it finishing
altogether** — give any of them enough time and the task eventually completes, the watermark moves,
and the rows go.

**[Section 6](#6-why-a-deleted-namespace-keeps-its-tasks-alive) covers a task that never completes
at all.** Faced with one namespace whose tasks will not drain, deleting that namespace is a natural
thing to reach for — remove the namespace, remove its tasks, release the watermark.

**Deleting the namespace does not work, because the deletion depends on what it is meant to fix.**
Deletion is a workflow, and it has to remove every execution in that namespace before the namespace
record itself goes. Removing executions creates more tasks — for the same namespace, in the lowest
priority class of all. So the deletion needs that namespace's task processing to work, and task
processing for that namespace is precisely what is not working. The deletion makes little or no
progress, the namespace record stays in place while it tries, its tasks keep resolving, and the
watermark does not move.

[Section 6](#6-why-a-deleted-namespace-keeps-its-tasks-alive) covers how to recognise that state,
the three signals the server gives you while it is happening, and what to do instead.

---

## 6. Why a deleted namespace keeps its tasks alive

**Picture where [section 3](#3-why-a-stuck-task-produces-no-errors),
[section 4](#4-when-a-database-outage-leads-to-a-stalled-cleanup) and
[section 5](#5-why-some-task-types-are-the-last-to-drain) have led.** One namespace's tasks are not completing — refused by a
limit, outvoted on priority, or both. That namespace holds the deletion watermark on every shard it
uses. No task rows are being removed on any of those shards, for any namespace, and the tables are
growing.

**Deleting the offending namespace looks like a clean way out of that, and the reasoning behind it
is sound.** The server really does drop tasks whose namespace cannot be found: it treats them as
invalid, acknowledges them, and the watermark moves on. Remove the namespace and the tasks holding
your cleanup have nothing left to act on.

> **The recommendation, before the explanation.** If a namespace's tasks are stalling cleanup,
> **do not delete that namespace to fix it.** Namespace deletion permanently destroys every
> workflow execution in that namespace — it is not a cleanup operation, and nothing in it is
> recoverable afterwards. It also will not work: the deletion cannot complete while the condition
> lasts, and it adds more work in the lowest priority class while it tries.
> [6.4](#64-what-to-do-instead) covers what to do instead, including what to do if a deletion is
> already running.

**Tasks are only dropped once the namespace record is actually gone, and getting there is the
problem.** Deleting a namespace is not a single operation — it is a workflow with several stages,
and removing the record is the last of them. Everything before it leaves the namespace in place,
and while the record is in place its tasks resolve normally and are refused exactly as they were
before.

### 6.1 What deleting a namespace actually does

| Stage | What happens |
|---|---|
| **1. Mark** | the namespace's state is set to **deleted**. It stops appearing in a namespace listing from this point on. |
| **2. Rename** | the namespace is renamed to its old name with `-deleted-` and the first five characters of its id appended — `orders-prod` becomes something like `orders-prod-deleted-a3f91`. The record itself stays. |
| **3. Reclaim** | a child workflow permanently deletes **every workflow execution** in the namespace, running and closed alike. This is the long stage, it is irreversible, and it is where the deletion stops if it is going to. |
| **4. Remove** | only once no executions remain is the namespace record itself deleted. |

**All of it runs as an ordinary Temporal workflow, which means you can look at it.** It runs in the
`temporal-system` namespace, and its workflow id is built from the namespace you asked to delete —
so it is predictable without searching:

| | |
|---|---|
| **Namespace** | `temporal-system` |
| **Workflow type** | `temporal-sys-delete-namespace-workflow` |
| **Workflow id** | `temporal-sys-delete-namespace-workflow/<the namespace you deleted>` |
| **Task queue** | `default-worker-tq`, with its activities on `temporal-sys-delete-namespace-activity-tq` |

```
temporal workflow describe -n temporal-system -w
"temporal-sys-delete-namespace-workflow/orders-prod"
```

**The id keeps the original name**, not the renamed one — so `orders-prod` finds it even though the
namespace itself is now `orders-prod-deleted-a3f91`. Its children,
`temporal-sys-reclaim-namespace-resources-workflow` and `temporal-sys-delete-executions-workflow`,
follow the same pattern and are where the time is actually spent.

**Stage 4, Remove, is the one that would get your rows deleted again.** While the record exists,
every task belonging to it still resolves to a real namespace, so the server keeps attempting those
tasks and keeps being refused. They stay unfinished, the watermark stays where they are, and **no
task rows are removed on those shards — for any namespace on them.**

Were the record genuinely gone, those same tasks would fail to resolve, be dropped as invalid and
acknowledged. The watermark would move, the next checkpoint would delete everything behind it, and
the tables would start shrinking. **The deletion has to get through stage 3, Reclaim, to reach that
point.**

### 6.2 Why the Reclaim stage stops

**Deleting an execution is not free: it creates tasks of its own.** Removing one execution produces
a `DELETE_HISTORY_EVENT` task and transfer and visibility delete-execution tasks — and
[section 5](#5-why-some-task-types-are-the-last-to-drain) showed that every one of those lands in
the **preemptable** class, the smallest share there is.

**Reclaim generates them quickly.** It runs concurrent delete activities, each rate-limited
separately, and the defaults multiply out to **400 execution deletes per second**:

| Setting | Default | |
|---|---|---|
| `frontend.deleteNamespaceDeleteActivityRPS` | 100 | per delete activity. **Can be changed while a deletion is running.** |
| `frontend.deleteNamespaceConcurrentDeleteExecutionsActivities` | 4 | how many run at once. Read once when the deletion starts, so changing it means restarting the deletion. |

**So Reclaim asks the namespace to absorb new tasks at several hundred a second, in the lowest
priority class, at a moment when that namespace is already not processing the tasks it has.** The
reason you reached for the deletion is the reason the deletion cannot finish.

**What follows is a loop.** The reclaim stage counts the remaining executions, finds the count has
not come down, and starts again. It does not fail and it does not give up; it keeps trying for as
long as the namespace cannot work through its own backlog.

**A namespace mid-deletion is in a worse state than it was before the deletion started.** Before,
the namespace held a backlog that would drain if its tasks could get through. Now it holds that
backlog plus a continuous supply of new cleanup tasks, all competing for the smallest share of the
scheduler.

### 6.3 How to recognise a stalled namespace deletion

**A namespace listing will not show it.** Namespaces in the deleted state are filtered out, so the
namespace holding your watermark is absent from the list you would check first — and it no longer
answers to the name you know it by.

**The history service, by contrast, can still see it.** The namespace registry deliberately
includes deleted namespaces when it loads, which is exactly why the tasks still resolve. So the
same namespace is absent from an operator's list and present in the server's own cache, at the same
time.

Three signals say this is what you are looking at:

| Signal | Where | What it means |
|---|---|---|
| **A namespace-deletion workflow that keeps running** | `temporal-sys-delete-namespace-workflow`, and its children `temporal-sys-reclaim-namespace-resources-workflow` and `temporal-sys-delete-executions-workflow` | the deletion was started and has not finished. A deletion that is progressing normally completes; one that sits for hours or days is stuck on Reclaim. |
| **`Some workflow executions still exist.`** | **worker** service logs, `WARN`, with a count | the reclaim stage checked and found executions remaining. Logged on every attempt, so the count is a progress meter — watch whether it falls. |
| **`No progress was made.`** | **worker** service logs, `WARN`, with the attempt number and the count | the count has not changed between attempts. The server's own comment on that branch names the cause: something has gone wrong on the task processor side. **Nothing else states the problem this plainly.** |

**Both log lines are on the worker service, not the history service** — a difference worth knowing
before grepping, since every other log line in this playbook comes from history. Both are at
`WARN`, so unlike the history-side lines they are visible on a cluster running at default log
levels.

### 6.4 What to do instead

**Namespace deletion permanently removes every workflow execution in that namespace**, running and
closed alike, and none of it can be recovered. That is what the operation exists to do. It is the
right choice only when you are certain none of that data will be wanted again — and it is never a
way to clear a stalled cleanup.

Three situations lead people here, and they call for different things:

| Where you are | What to do |
|---|---|
| **Considering deleting the namespace to clear the stall** | Do not. The deletion cannot finish while its tasks cannot complete, and it adds cleanup tasks in the lowest priority class while it tries. Clear the backlog by one of the routes below; delete the namespace afterwards if you still want it gone. |
| **A deletion is already running and making no progress** | Leave it running — cancelling it gains nothing and the namespace is already renamed. Consider lowering `frontend.deleteNamespaceDeleteActivityRPS` from its default of 100 first: it takes effect on a deletion that is already running, and it slows the rate at which the deletion adds new preemptable tasks to a namespace that cannot process them. Then clear the backlog by one of the routes below, and the deletion finishes on its own. |
| **You have independently decided the namespace and everything in it can go** | Clear the backlog first. A deletion started against a namespace that cannot process tasks sits in the Reclaim stage indefinitely. Satisfy yourself that no execution in it is still needed before starting — the stalled cleanup is a separate problem, and deleting data is not how it gets solved. |

**Before any of the routes below, there is a lever that costs nothing and needs no configuration
change: stop or slow new workflow starts in that namespace**, if the business can tolerate it.

A throttled namespace has a fixed amount of task processing available to it, and every new workflow
started competes for that same share while adding its own task rows above the watermark. Stopping
new starts points the whole of that share at the backlog instead of splitting it, and stops the
table growing underneath you at the same time. Slowing starts rather than stopping them helps in
proportion. Do it where the workflows are started — from the applications or schedules that call
`StartWorkflowExecution`.

> **Do not use namespace deprecation to achieve this.** Deprecating a namespace does block new
> starts, which makes it look like the right tool. It is a **one-way transition**: from
> `Deprecated` the only state a namespace can move to is `Deleted`, and it can never be returned to
> `Registered`. A namespace deprecated to buy a few hours cannot be brought back afterwards.

The routes out are then the same as for any stalled cleanup, and the deletion changes neither of
them:

- **Let the namespace's tasks through.** Raise the per-namespace limits from
  [section 3](#3-why-a-stuck-task-produces-no-errors) so its backlog can drain, and if the backlog
  is now mostly cleanup tasks, raise its preemptable weight as
  [5.4](#54-whether-to-change-the-priority-weights) describes. Give the Reclaim stage enough capacity and the
  deletion finishes on its own — at which point the record goes, the remaining tasks are dropped as
  invalid, and the watermark is released.
- **Or remove the rows directly**, which is [section 9](#9-get-task-row-cleanup-running-again). This is the
  faster route when the table has already grown large, and it is what that section is built around.

**On either route the deletion completes by itself**, and nothing about the deleted state has to
be undone. The namespace is not corrupt and the deletion is not broken — it is waiting for capacity
that this playbook is about restoring.

**Four things can leave a task unfinished and holding the watermark**: it is refused by a limit,
it is on a reader served last, it is outvoted on priority, or its namespace is mid-deletion. A real
incident is usually more than one of them at once.

**Signals have been named along the way, but never gathered.** Everything up to here pointed at a
panel here and a log line there, each at the point where the mechanism made sense of it.

**The rest of the playbook separates two different jobs, and it is worth knowing which one you are
reading for:**

| | |
|---|---|
| **Catching it early** | [Section 7](#7-detect-a-stalled-cleanup-on-the-dashboard) covers what the metrics show — including a panel that goes to zero within minutes of a watermark pinning, long before any table is large enough to notice. [Section 12](#12-alerting-on-a-stalled-cleanup) turns that into an alert, so it does not depend on anyone looking. |
| **Dealing with it once it has happened** | [Section 8](#8-confirm-which-shards-and-which-namespace-are-affected) finds the shards and the namespace, and [sections 9](#9-get-task-row-cleanup-running-again) and [10](#10-what-not-to-do-when-cleanup-deletes-are-failing) clear the backlog. |

**Catching it early and dealing with it both start from
[section 7](#7-detect-a-stalled-cleanup-on-the-dashboard)**, because the same panels answer "is this happening" and "is this what
is happening to me". It covers what to look at first, what each panel can and cannot tell you, and
which of them stay flat however bad the situation becomes.

---

## 7. Detect a stalled cleanup on the dashboard

Everything so far has been about what is happening inside the server. **This section is what the
server's own metrics let you observe of it** — which panels move, which do not, and in what order
to read them.

Every panel named here is on the
**[Temporal Server Dashboard](../observability/dashboards/server/temporal-server-readme.md)** in
this repository, and each panel links to its own entry in that readme. If you are not running the
dashboard yet, [the JSON to import](../observability/dashboards/server/temporal-server.json) is
alongside it.

**[Section 1](#1-telling-a-real-backlog-from-rows-that-were-never-deleted) asked you to tell a real
backlog of pending work from rows that were never deleted. One panel settles that, and 7.1 is where
it is.** Start there; the subsections after it are what to read once you know which of the two you
have.

### 7.1 Start with the cleanup panels

The
**[History Task Cleanup](../observability/dashboards/server/temporal-server-readme.md#23-history-task-cleanup)**
row exists for one purpose: showing whether task rows are being deleted.

All four panels are about one statement — the single `DELETE` from
[2.4](#24-one-statement-removes-everything-below-the-watermark) that removes rows from a task
table. One statement, per shard, per checkpoint, and only when the watermark moved. Each panel
breaks it down by **category** — the series are named `RangeCompleteTimerTasks`,
`RangeCompleteTransferTasks`, `RangeCompleteVisibilityTasks` and so on, one per task table.
**Read all four panels together**, because the pattern across them is what carries the meaning.

| Panel | What it shows |
|---|---|
| **Task Row Cleanup Attempts by Category** | how many of those `DELETE` statements are being issued per second. On a healthy cluster this is roughly your shard count divided by the 30-second checkpoint interval, for each category that has traffic. |
| **Task Row Cleanup Failures by Category** | how many of them failed, broken down by the error returned. A cancelled statement that hit the five-second timeout appears here. |
| **Task Row Cleanup Latency by Category** | how long each statement takes. Healthy is milliseconds; a line sitting at five seconds is the timeout being hit. |
| **Task Row Cleanup Success Rate** | the percentage succeeding, across all categories. One number to glance at. |

**No single panel tells you whether rows are being deleted; all four read together do.** Three
combinations come up in practice, and they call for completely different responses:

| What the four panels show together | What it means |
|---|---|
| **Attempts present, no failures, low latency** | cleanup is working. Whatever is growing your table, it is not a stalled watermark. |
| **Attempts absent or near zero** | **the watermark is pinned.** No delete is being issued because the position has not moved, so there is nothing new to delete. This is the quiet failure from [sections 3](#3-why-a-stuck-task-produces-no-errors) to [6](#6-why-a-deleted-namespace-keeps-its-tasks-alive), and it produces no errors anywhere because nothing is being attempted. |
| **Attempts present, failures present, latency at five seconds** | **the deletes themselves are failing.** The watermark has moved, the range is too large to clear inside the timeout, and each attempt is being cancelled. That is [section 10](#10-what-not-to-do-when-cleanup-deletes-are-failing), and it is the one state here worth alerting on — see below. |

**Failing deletes are worth an alert; a pinned watermark is not.** Cleanup deletes never fail on a
healthy cluster, so any sustained failure is actionable with no threshold to tune —
**[alert 91, Task Row Cleanup Failing](../observability/alerts/server/alerts-index.md#alert-91--task-row-cleanup-failing)**
is written for exactly that. A pinned watermark cannot be alerted on as simply, because a category
with no traffic also reports no attempts, so that one is read on the dashboard rather than paged
on. [Section 12](#12-alerting-on-a-stalled-cleanup) covers both in full.

**An empty panel is a finding here, not a gap.** Everywhere else on the dashboard, no data usually
means no problem. On Cleanup Attempts it means the opposite — and it is the single most direct
reading in this playbook.

**One caveat before you act on it:** categories with no task traffic at all report nothing, which
is correct rather than broken. Check that the category you care about has traffic — **[History Task
Throughput](../observability/dashboards/server/temporal-server-readme.md#9-shard-queue-health)** will tell you — before reading an empty Attempts panel as a
stalled watermark.

### 7.2 Tell whether a task queue is falling further behind or catching up

A task table that is large is not necessarily a task table that is still growing, and
[section 1](#1-telling-a-real-backlog-from-rows-that-were-never-deleted) made that distinction. Two
panels tell you which direction a queue is moving, without touching the database:

| Panel | Use |
|---|---|
| **[Immediate Queue Backlog Age by Category](../observability/dashboards/server/temporal-server-readme.md#9-shard-queue-health)** | the age of the oldest unprocessed task, for transfer, visibility and outbound. **Rising means falling further behind; falling means catching up.** There is no scheduled-queue equivalent. |
| **[Task Load Latency by Task Type](../observability/dashboards/server/temporal-server-readme.md#9-shard-queue-health)**, read at **p95** | the timer equivalent. A timer loaded after its fire time reports exactly how overdue it is, so p95 on timer types trends the same thing. |

**Both panels report an age, never an amount.** A queue ten minutes behind and a queue ten days
behind differ only in the number of seconds reported; neither panel says how many tasks are waiting
or how many rows are in the table, because no metric counts either. **Use them for direction
only** — rising means falling further behind, falling means catching up. For the size of the table
itself, [section 8](#8-confirm-which-shards-and-which-namespace-are-affected).

**Task Load Latency has a trap worth knowing.** It records only at the moment a task is **loaded**
from the database, so a queue that has stopped loading altogether emits nothing and the line goes
flat — which looks identical to a queue that is perfectly healthy. **Disambiguate it against
Cleanup Attempts from 7.1**: flat with attempts present is healthy, flat with attempts absent is a
queue that has stopped.

### 7.3 What the queue lag panels can and cannot tell you

**[Scheduled Queue Lag per Pod](../observability/dashboards/server/temporal-server-readme.md#9-shard-queue-health)**
has a name that suggests it should answer how far behind a queue is, and it is a natural panel to
reach for. It carries less than it appears to, and the limits are worth knowing before you rely on
it.

The panel plots a number of seconds, but only three readings on it mean anything distinct:

| What it reads | What that tells you |
|---|---|
| **around 5 minutes** | **Healthy.** The queue deliberately reads ahead of the present, so even a completely idle cluster reports roughly `history.timerProcessorMaxPollInterval` — five minutes at the default, measured at 494.6s on an idle test cluster. **Zero is not the healthy value; this is.** |
| **around 500 seconds** | The only intermediate step the metric offers. There is one histogram boundary between the healthy reading and the maximum, and this is it. |
| **exactly 1000** | **The highest number the panel can ever show.** The metric is a Seconds histogram whose top bucket is 1000s, about 16.7 minutes. A queue an hour behind and a queue a month behind both read 1000, and so does everything in between. |

**So the panel is very nearly a two-state indicator.** On a test cluster it went from its healthy
reading to 902 in a single sample, reached exactly 1000.000 within **eight minutes**, and then held
1000 — through a complete drain, and for three minutes after the queue was empty.

**Read it as "something is wrong", never as "how bad" or "is it improving".** It will not fall while
you fix the problem, and it will keep reading 1000 for minutes after you have fixed it. Use 7.1 and
7.2 to judge progress.

### 7.4 Narrow down which namespace is likely responsible

Once cleanup is confirmed stalled, these three panels point at a namespace. The readings are the
ones [section 3.4](#34-which-signals-do-move) set out, applied here:

| Panel | What it shows |
|---|---|
| **[Task Scheduler Throttling by Namespace](../observability/dashboards/server/temporal-server-readme.md#9-shard-queue-health)** | tasks the scheduler declined to dispatch, by namespace. |
| **[Throttled Tasks by Task Type and Namespace](../observability/dashboards/server/temporal-server-readme.md#9-shard-queue-health)** | the same refusals seen from the task side, by type — which tells you whether the stuck work is timer, transfer or cleanup. |
| **[Rejected Database Calls by Namespace](../observability/dashboards/server/temporal-server-readme.md#3-persistence-requests-latencies-and-errors)** | calls refused by a persistence limit, with the scope that refused them. |

**Look for throttling that does not end, not throttling that is large.**
[Section 3.4](#34-which-signals-do-move) covers the readings in full; the short version is that a
namespace throttled continuously, flat while its own traffic in
**[RPS per Namespace](../observability/dashboards/server/temporal-server-readme.md#1-cluster-throughput)**
rises and falls, is the candidate — and throttling that continues while that namespace's traffic is
at zero is close to conclusive.

**These panels give you a candidate, not a confirmation**, and the difference matters in one case: a
namespace whose deletion is stuck has no client traffic at all, so it may not stand out against
namespaces doing ordinary work.
[Section 8](#8-confirm-which-shards-and-which-namespace-are-affected) reads the namespace out of the
stuck scope itself, which is definitive — and where the two disagree, that reading is the right one.

### 7.5 The panels that stay clean, and why that is not reassurance

**While cleanup is stopped, the panels below report no errors and no change at all.** That is not
because the problem is mild — it is because none of them measures anything the problem touches:

| Panel | Why it stays flat |
|---|---|
| **[Total Timer Tasks Errors](../observability/dashboards/server/temporal-server-readme.md#10-history-timer-task-info)** and anything built on `task_errors` | a throttled task is not counted as an error — it goes to `task_scheduler_throttled` or `task_errors_throttled` instead, which is [7.4](#74-narrow-down-which-namespace-is-likely-responsible). |
| **[History Task DLQ / Terminal Failures](../observability/dashboards/server/temporal-server-readme.md#20-history-task-dlq--terminal-failures)** | a throttled task never advances the attempt counter that leads to the dead-letter queue, so it can never arrive there. |
| **[Workflow Success](../observability/dashboards/server/temporal-server-readme.md#11-workflow-stats)**, **[Persistence Latencies](../observability/dashboards/server/temporal-server-readme.md#3-persistence-requests-latencies-and-errors)** | nothing is wrong with workflow execution or with the database. |

**Clean error panels are evidence for a stalled cleanup, not evidence against one.** What
identifies a stalled cleanup is the combination: **cleanup attempts absent or failing, while the
three panels above stay clean.** Were those three busy instead, the problem would be a different
one — a task pipeline that is failing, rather than one that is quietly not finishing.

**The dashboard and alert 91 together establish three things**: that cleanup has stopped or its
deletes are failing, which task category it is, and probably which namespace is responsible. For a
cluster that is watching rather than already in trouble, that is enough — making sure you find out
this early is the point of [section 12](#12-alerting-on-a-stalled-cleanup).

**No metric carries the next three things you need, and you need them before changing anything**:
which shards are affected, how many rows are involved, and which task is holding the watermark.
**[Section 8](#8-confirm-which-shards-and-which-namespace-are-affected) is where each of them comes
from** — a log line that names the stuck shards outright, and read-only queries that size the table
and confirm the namespace.

---

## 8. Confirm which shards and which namespace are affected

[Section 7](#7-detect-a-stalled-cleanup-on-the-dashboard) establishes that cleanup has stopped and
which task category it is. **This section gets you the three things no metric carries**: which
shards are affected, which namespace is holding them, and how the retained rows are spread over
time — which is what decides whether there is still work to drain or only rows to delete.

**Everything here is read-only.** Nothing in this section changes the cluster, and it is all safe
to run while the problem is live. Changing things begins in
[section 9](#9-get-task-row-cleanup-running-again).

### 8.1 Get the affected shards from the history service logs

**The server names the stuck shards itself, and this is the single most precise signal in the
playbook.** When a cleanup delete fails, the queue logs:

```
Error range completing queue task
```

The line is logged at `ERROR` on the queue's own logger, which carries both the shard and the
queue it belongs to. Grep the history service logs for it:

```
Error range completing queue task   shard-id=412   component=timer-queue-processor  ...
Error range completing queue task   shard-id=1177  component=timer-queue-processor  ...
```

**Two things come out of it.** The `shard-id` tags give you the set of shards to work on. The
`component` tag names the **task category** — `timer-queue-processor`,
`transfer-queue-processor`, `visibility-queue-processor` or `archival-queue-processor` — which is
what tells you which table [8.3](#83-tell-a-stuck-cohort-from-ordinary-retained-traffic) should
query.

**That list turns an open-ended problem into a targeted one.** Instead of a table of unknown shape
across every shard in the cluster, you have a specific set to work on — and
[section 9](#9-get-task-row-cleanup-running-again) only needs to touch those.

**The line means one thing: a cleanup `DELETE` was issued and did not succeed.** It is logged on
every failure, whatever the cause, and the error it carries tells you which — a timed-out delete
is the case this playbook is about, but a database that is down or a shard whose ownership has
moved will log it too. Check the error text before assuming.

**It appears only while cleanup deletes are being attempted.** During the quiet phase — watermark
pinned, no delete issued at all — the line is absent, which matches the two combinations in
[7.1](#71-start-with-the-cleanup-panels). **An empty grep does not mean you have no problem**; it
means you are in the other one, and 8.2 onwards is how you find it.

**One more line is worth grepping for**, from
[4.2](#42-the-server-moves-namespaces-that-fall-far-enough-behind-onto-a-second-reader):

```
Too many pending tasks, moving group to next reader
```

That line is logged at `INFO`, so a cluster running at `WARN` will not have it — but where it
exists it
records when a namespace was moved to the second reader, how far behind it was, and **which
namespace**, which is the question 8.2 otherwise has to work at.

### 8.2 Read a shard's readers and scopes

For any shard from [8.1](#81-get-the-affected-shards-from-the-history-service-logs), `tdbg` shows
the structure [section 2](#2-how-the-server-decides-which-rows-are-safe-to-delete) described:

```
tdbg shard describe --shard-id 412
```

**The output covers every task category on that shard**, since each has its own queue and its own
readers. Read the one the `component` tag in 8.1 named, and ignore the rest — a healthy transfer
queue alongside a stuck timer queue is normal and not a second problem.

**What to look for, in order:**

| | |
|---|---|
| **How many readers** | one is normal. **Two means a namespace was moved**, which is [section 4](#4-when-a-database-outage-leads-to-a-stalled-cleanup). |
| **Each reader's first scope** | its lower bound is what that reader contributes to the watermark. |
| **The lowest of those lower bounds** | that is the deletion watermark for this queue, and the point behind which nothing is being deleted. Its age is the age of the problem. |
| **The predicate on that scope** | it names the namespace ids the scope covers. **This is the definitive answer to which namespace is holding cleanup on this shard** — [7.4](#74-narrow-down-which-namespace-is-likely-responsible) gives a candidate from the metrics, and this either confirms it or replaces it. |

**Expect small gaps on the default reader and do not be distracted by them.** A healthy default
reader often holds several scopes covering short, recent ranges — seconds to hours. What you are
looking for is one scope whose lower bound is orders of magnitude older than the rest.

**Scopes carry namespace ids, not names**, and a namespace that has been deleted no longer answers
to the name you know. List them straight from the database:

```sql
SELECT encode(id, 'hex') AS id_hex, name
FROM namespaces;
```

Strip the dashes from the id in the `tdbg` output and match it against `id_hex`. A name ending in
`-deleted-` and five hex characters is the case in
[section 6](#6-why-a-deleted-namespace-keeps-its-tasks-alive).

### 8.3 Tell a stuck cohort from ordinary retained traffic

**This query decides what [section 9](#9-get-task-row-cleanup-running-again) has to do**, and it is worth
running before anything else is changed. Count the rows on one affected shard by the day they
were due:

```sql
SELECT date_trunc('day', visibility_timestamp) AS day,
       count(*)                                AS rows
FROM timer_tasks
WHERE shard_id = 412
GROUP BY 1
ORDER BY 1;
```

**The table to query, and whether that query works at all, depends on the task category** — which
[8.1](#81-get-the-affected-shards-from-the-history-service-logs) gave you from the `component` tag,
or [7.1](#71-start-with-the-cleanup-panels) from which series moved on the cleanup panels. Only the
scheduled categories carry a timestamp:

| Category | Table | Has a time column? |
|---|---|---|
| **Timer** | `timer_tasks` | yes — `visibility_timestamp`, so the query above works as written |
| **Archival** | `history_scheduled_tasks`, with `AND category_id = 5` | yes — same column, same query |
| **Transfer** | `transfer_tasks` | **no** |
| **Visibility** | `visibility_tasks` | **no** |
| **Outbound** | `history_immediate_tasks`, with `AND category_id = 7` | **no** |

**For a category with no time column, bucket by `task_id` instead.** Task ids are allocated in
increasing order within a shard, so a dense block of ids reads the same way a dense day does — the
same shape, in id space rather than time:

```sql
SELECT (task_id / 100000) * 100000 AS id_bucket,
       count(*)                    AS rows
FROM transfer_tasks
WHERE shard_id = 412
GROUP BY 1
ORDER BY 1;
```

Adjust the bucket width to suit your row counts. What you lose is the ability to put a **date** on
the cohort, which is the most useful thing the timer version gives you — for that, correlate the id
range against when the incident happened.

You get one row per day, with the number of task rows due on it:

```
    day     |  rows
------------+---------
 2026-02-14 |  380412
 2026-02-15 |    3104
 2026-02-16 |    2998
 ...
 2026-05-02 |    3051
```

**You are looking for one thing: whether any single day holds far more rows than the others.**
Scan the counts column from top to bottom and ask that question — the individual numbers do not
matter, only whether one of them dwarfs the rest. Three answers are possible:

| If the counts are… | Then you have… | And [section 9](#9-get-task-row-cleanup-running-again) has to… |
|---|---|---|
| **flat** — every day roughly the same, nothing stands out | ordinary traffic retained behind a watermark that stopped. Those tasks all completed; only their rows are left. | **remove rows, and nothing else.** There is no work outstanding to drain. |
| **one tall day, the rest near zero** | a cohort of tasks from that date that never completed. | **get those tasks completed first.** Removing rows will not fix it — the tasks are still there and still holding the watermark. |
| **one tall day plus a steady tail** across every day since — the example above | both at once, which is the usual result. | **do the cohort first, then remove the tail.** That section is ordered this way for exactly this case. |

**If there is a tall day, note its date.** It is the date the watermark stopped, and it is normally
recognisable once you look — an outage, a deploy, a limit change. Knowing it tells you how long the
condition has run and gives you something to correlate against; it is also the date you will bound
any cleanup by in [section 9](#9-get-task-row-cleanup-running-again).

**Run that query one shard at a time, and do not widen it to the whole table.** An unqualified
`count(*)`
on a table that has grown this large is a long scan against a database that may already be under
pressure. One shard is enough to tell the shape, and
[1.3](#13-deciding-between-a-real-backlog-and-retained-rows) covers taking the overall size from
table statistics instead of counting.

### 8.4 Sample the rows, and what the sample cannot tell you

`tdbg` can list the task rows themselves over a range:

```
tdbg shard list-tasks --shard-id 412 --task-category timer \
  --min-visibility-ts 2026-02-14T00:00:00Z --max-visibility-ts 2026-02-15T00:00:00Z
```

`--task-category` takes `timer`, `transfer`, `visibility`, `archival` or `outbound` — the one
[8.1](#81-get-the-affected-shards-from-the-history-service-logs) named. The timestamp bounds only
apply to the scheduled categories; for the others, use `--min-task-id` and `--max-task-id` against
the id range [8.3](#83-tell-a-stuck-cohort-from-ordinary-retained-traffic) found.

**Read it as a sample, never as a census.** It pages, with a small default page size, so what comes
back is a window onto a range that may hold millions of rows. Treat the mix of task types and
namespaces in it as suggestive and nothing more.

**The listing also cannot tell you whether a row's task is finished.** The listing is a range scan
of the
table, and a completed task's row is indistinguishable from a pending one — which is the whole
problem this playbook describes, seen from the other side.

**What the sample is genuinely good for is the task type mix**, which
[section 5](#5-why-some-task-types-are-the-last-to-drain) showed decides how fast anything will
drain:

| What you see | What it suggests |
|---|---|
| mostly `DELETE_HISTORY_EVENT` and delete-execution types | retention or namespace-deletion work, in the preemptable class. Expect it to drain slowly, and see [5.4](#54-whether-to-change-the-priority-weights). |
| mostly timeout types — `ACTIVITY_TIMEOUT`, `WORKFLOW_TASK_TIMEOUT` | an outage cohort, as in [4.1](#41-an-outage-leaves-busy-namespaces-with-far-more-outstanding-work-than-usual). These drain at nearly full rate once the limits allow it. |
| a large population of `ACTIVITY_RETRY_TIMER` | activities stuck in retry loops. Their **absence** is equally informative — it argues against mass activity churn as the cause. |

**You will not be able to point at the one task that is holding the watermark — and you do not need
to.** It is the obvious next question after all of the above, so it is worth answering directly.

**Why you cannot identify it:** a task row carries no record of whether its task has finished, so
nothing in the table distinguishes the single incomplete task from the millions of completed ones
around it. That is the same property this whole playbook is about, seen one last time.

**Why it does not matter:** what you have instead is its fire time — the lower bound of the oldest
scope from [8.2](#82-read-a-shards-readers-and-scopes) — and the namespace it belongs to. Every
remedy in [section 9](#9-get-task-row-cleanup-running-again) works on a namespace, a set of shards and a
range of time. None of them needs to single out a row.

**You now have four things**: the shards involved, the namespace holding them, the task category,
and whether the retained rows are a stuck cohort, ordinary retained traffic, or both. Everything up
to this point has been read-only.

**[Section 9](#9-get-task-row-cleanup-running-again) is where the playbook starts changing things.** It takes
those four and works through them in a fixed order: establish whether the backlog is held by a
limit or by priority, lift whichever it is, and only then consider removing rows — which is a last
resort rather than a first move, for reasons that section makes plain. It also names the one
operation never to perform on a task table.

---

## 9. Get task row cleanup running again

[Section 8](#8-confirm-which-shards-and-which-namespace-are-affected) left you with the shards, the
namespace, the category, and whether you are looking at a stuck cohort, retained traffic, or both.
**This section acts on them, in an order chosen so that each step is reversible and the risky one
comes last.**

> **The one operation never to perform on a task table.** Do not delete rows filtered only by time.
> A task table has **no namespace column** —
> [1.3](#13-deciding-between-a-real-backlog-and-retained-rows) showed its columns, and a namespace is not
> among them — so a `DELETE ... WHERE visibility_timestamp < '...'` removes rows for **every**
> namespace on that shard, including live tasks for workflows that are running perfectly well.
> Those workflows then wait forever for timers that no longer exist. There is no undo, and nothing
> reports it.

**Working through all four in order matters more than any single one of them:**

| | Step | Risk |
|---|---|---|
| **1** | Work out whether a namespace limit or a priority weight is holding that namespace's tasks | none — read-only |
| **2** | Raise whichever of the two it turns out to be, a step at a time | reversible |
| **3** | Watch task rows start being deleted again, knowing which signals will mislead you | none |
| **4** | Delete task rows yourself — only if steps 1 to 3 cannot work | **irreversible** |

**Step 4 exists for one situation: tasks that can never complete, whatever you raise.** Where the
tasks can complete, steps 1 to 3 are enough and the server removes the rows itself — which is the
outcome to aim for, because it needs no judgement about which rows are safe to delete.

### 9.1 Work out whether a limit or priority is holding the tasks

**Two different things can keep a namespace's tasks from completing, and they need opposite
fixes.** Either the per-namespace rate limits from
[section 3.1](#31-two-different-limits-refuse-a-task-at-two-different-moments) are refusing its
tasks, or nothing is refusing them and they are losing the scheduler's rotation to higher-priority
work, which is [section 5](#5-why-some-task-types-are-the-last-to-drain). **Raising a rate limit
does nothing for the second case.** One panel tells them apart:

| On [Task Scheduler Throttling by Namespace](../observability/dashboards/server/temporal-server-readme.md#9-shard-queue-health) | Means | Lever |
|---|---|---|
| **High and sustained** for that namespace | **Rate-bound.** Its tasks are being refused faster than they get through. | raise `history.taskSchedulerNamespaceMaxQPS` and `history.taskSchedulerGlobalNamespaceMaxQPS` for that namespace — [9.2](#92-raise-a-namespace-limit-or-a-priority-weight-in-steps) |
| **At or near zero**, while the watermark stays frozen | **Starved, not refused.** Nothing is turning its tasks away; they are losing the rotation. Expect this when [8.4](#84-sample-the-rows-and-what-the-sample-cannot-tell-you) showed mostly cleanup task types. | raise the preemptable weight for that namespace in `history.timerProcessorSchedulerActiveRoundRobinWeights` — [5.4](#54-whether-to-change-the-priority-weights) has a worked example |
| **Either reading, but `task_errors` or the dead-letter panels are also non-zero** | Tasks are **failing**, not waiting — which this playbook's failure never does. | you are looking at a different problem; [section 3](#3-why-a-stuck-task-produces-no-errors) explains why |

**Tasks losing the rotation is easy to misdiagnose**, because the instinct is to raise the
namespace's rate limit anyway. Here is why that disappoints: cleanup task
types sit in the preemptable class, which gets roughly **5% of dispatch slots**. Raising that
namespace's QPS raises five percent of a larger number — proportional help at best, and often not
noticeable. **If the work that is stuck is cleanup work, the priority weight is by far the stronger
lever**, and raising the rate limit instead can look like the fix has failed when it was simply
aimed at the wrong thing.

### 9.2 Raise a namespace limit or a priority weight, in steps

Whichever lever [9.1](#91-work-out-whether-a-limit-or-priority-is-holding-the-tasks) pointed at,
**raise it gradually rather than in one move.** The reason is specific to this failure rather than
general caution.

A watermark that has been pinned for weeks does not inch forward when it releases — it jumps the
whole period in one step, and
[4.3](#43-the-default-reader-races-ahead-while-reader-1-holds-the-watermark) explained why. The
first cleanup delete after that jump has to cover every row accumulated across those weeks, as one
unbounded statement under the delete query timeout. **Raising a namespace's rate limit or priority
weight by a large factor unpins many shards at once, so they all attempt that oversized delete in
the same moment** — which is how a recovery turns into
[section 10](#10-what-not-to-do-when-cleanup-deletes-are-failing).

**The procedure, for whichever lever [9.1](#91-work-out-whether-a-limit-or-priority-is-holding-the-tasks)
pointed at:**

1. **Read the current value.** For a rate limit that is
   `history.taskSchedulerNamespaceMaxQPS` and `history.taskSchedulerGlobalNamespaceMaxQPS` for
   that namespace; for priority it is the `preemptable` weight in
   `history.timerProcessorSchedulerActiveRoundRobinWeights`, which defaults to 1 against high 10
   and low 9.
2. **Double it, scoped to the affected namespace only.** Both settings take a namespace
   constraint. Doubling is a starting point rather than a magic number — the principle is that each
   change should be small enough that you can undo it before it matters.

   ```yaml
   history.taskSchedulerGlobalNamespaceMaxQPS:
     # leave the cluster default alone
     - value: 1000
       constraints: {}
     # the namespace that is stuck, raised a step at a time
     - value: 2000
       constraints:
         namespace: 'orders-prod'
   ```

   For the priority case the setting is different but the shape is the same —
   [5.4](#54-whether-to-change-the-priority-weights) has that example. **A namespace mid-deletion
   is constrained by its renamed form**, `orders-prod-deleted-a3f91`, not the name you knew it by;
   [8.2](#82-read-a-shards-readers-and-scopes) is where you get it.
3. **Wait at least ten minutes before changing anything else.** The cleanup panels rate over
   five-minute windows and deletes fire on a thirty-second checkpoint, so a shorter wait tells you
   nothing. Both settings are hot reloads; there is nothing to restart.
4. **Check two things**: **[Task Row Cleanup Attempts by Category](../observability/dashboards/server/temporal-server-readme.md#23-history-task-cleanup)** rising off zero for that
   category, and the per-day counts from
   [8.3](#83-tell-a-stuck-cohort-from-ordinary-retained-traffic) falling on a re-run.
   [7.1](#71-start-with-the-cleanup-panels) covers reading that panel.
5. **Repeat from step 2 while the backlog is still shrinking too slowly**, and stop as soon as it
   is shrinking at all. There is no target value to reach — the point is movement, not a number.

**Three ways raising a limit goes wrong:**

- **Multiplying by ten in one change.** Every shard that namespace was pinning releases together,
  and they all attempt an oversized delete at once.
- **Changing again before the first change has shown its effect**, which leaves you unable to tell
  which change did what, or to undo the one that hurt.
- **Raising a cluster-wide limit to fix one namespace**, which hands the extra capacity to every
  namespace including the ones that were already fine.

**One lever that looks useful and is not: raising `history.timerProcessorUpdateAckInterval`.**
Lengthening the checkpoint interval does not slow down failing deletes — a delete that fails is
retried on its own backoff, which tops out at five seconds and ignores this setting entirely. What
it does do is make each *successful* delete cover a wider range, which works against you. Leave it
at its default.

### 9.3 Three signals that look like failure during recovery

Three things happen during a successful drain that look like failure. All three are measured, and
all three will mislead you if you are not expecting them.

| What you see | Why | What to do |
|---|---|---|
| **The row count goes up before it comes down** — by more than half, in one measurement | workflows that had been stuck complete in a rush, and completing workflows emit retention timers of their own. New rows arrive faster than old ones are removed, for a while. | nothing. Watch the trend over hours, not minutes. |
| **No visible progress, then sudden completion** — in one sample the aggregate watermark moved five seconds in a minute, then 47 minutes in a single step | the watermark across shards is a `min`. While any one shard lags, the aggregate shows that shard. Progress on all the others is invisible until the slowest catches up. | judge progress per shard, not in aggregate. |
| **The queue lag panel does not move at all** | [7.3](#73-what-the-queue-lag-panels-can-and-cannot-tell-you) — it saturates at 1000 and stays there, including for minutes after the queue is empty. | ignore it entirely during a drain. Use the cleanup panels from [7.1](#71-start-with-the-cleanup-panels). |

**What actually tells you the drain is working** is
**[Task Row Cleanup Attempts by Category](../observability/dashboards/server/temporal-server-readme.md#23-history-task-cleanup)** rising from zero on the affected category, and the
per-day counts from [8.3](#83-tell-a-stuck-cohort-from-ordinary-retained-traffic) falling when you
re-run the query.

### 9.4 Deleting task rows yourself, and why it is the last resort

**Delete rows only when the tasks they represent genuinely do not need to run.** There is one
clear case — a namespace whose executions are being deleted anyway, which is
[section 6](#6-why-a-deleted-namespace-keeps-its-tasks-alive) — and outside it the risk is severe.

> **One situation moves this up the list rather than leaving it last: cleanup deletes that are
> failing on the delete query timeout.** While that is happening the database is paying full cost in CPU, I/O
> and WAL for statements that remove no rows, so the condition is not merely persisting — it is
> consuming capacity your workloads need, and steps 2 and 3 make it worse rather than better.
> **Reducing the number of rows the statement has to match is the only thing that clears it**,
> which makes row deletion the primary remedy there rather than the last one. The safety conditions
> below still apply in full.
> [Section 10](#10-what-not-to-do-when-cleanup-deletes-are-failing) is that situation, and it has more to say
> about bounding the work.

**Check your server version first**, and which of the two changes proposed in
[#12341](#note-to-readers) landed, because they lead to different places:

| If the version you run has… | Then |
|---|---|
| **batched deletes** | an oversized delete finishes on its own, and nothing below is needed. The server clears the backlog once the watermark moves. |
| **a configurable timeout, but not batching** | you have more room, not a guarantee. Raise it and watch, but a range large enough will still exceed whatever you set, and everything below still applies. |
| **neither** | everything below applies as written. |

**You cannot target a namespace, which is what makes deleting rows risky.** A task table has no
namespace column, so no
`DELETE` you write can distinguish one namespace's rows from another's. The narrowest targeting
available is **a shard and a range of time**, and every namespace with rows in that window loses
them. [Section 8](#8-confirm-which-shards-and-which-namespace-are-affected) gives you both bounds:
the shard list from
[8.1](#81-get-the-affected-shards-from-the-history-service-logs), and the cohort's date from
[8.3](#83-tell-a-stuck-cohort-from-ordinary-retained-traffic).

**Before deleting anything, satisfy yourself of all four:**

- The shard is one [8.1](#81-get-the-affected-shards-from-the-history-service-logs) named, not a
  guess.
- The time range is bounded to the cohort, not open-ended.
- [8.3](#83-tell-a-stuck-cohort-from-ordinary-retained-traffic) shows that range is overwhelmingly
  one namespace's rows — if it is mixed, you are about to delete someone else's live timers.
- That namespace's executions are being deleted anyway, so its tasks have nothing left to do.

**If any one of those four conditions does not hold, go back to
[9.1](#91-work-out-whether-a-limit-or-priority-is-holding-the-tasks) and
[9.2](#92-raise-a-namespace-limit-or-a-priority-weight-in-steps).** Letting the tasks complete costs time;
deleting the wrong rows costs workflows that will never finish, with no error and no way to find
out which.

**Once the rows are gone, the deletes start succeeding again on their own.** The next cleanup
delete covers a range small enough to finish inside the timeout, the start of the range moves
forward again, and the
database stops paying for cancelled statements.

**Then go back to [9.2](#92-raise-a-namespace-limit-or-a-priority-weight-in-steps) and finish the
job.** Removing rows clears what had accumulated; it does nothing about the namespace whose tasks
could not get through in the first place. Raise its limit or weight so the remainder drains, and
the server takes over from there. The full order, when deletes were failing, is:

1. Delete the rows, bounded as above — the database stops burning capacity.
2. Watch the row count fall and the cleanup deletes start succeeding.
3. Return any setting you changed while firefighting to its default —
   `history.timerProcessorUpdateAckInterval` in particular, if it was raised.
4. Raise the stuck namespace's limit or weight per 9.2, so whatever is left drains.

**That last step is the one that gets forgotten**, because by then the alarming part is over. Skip
it and the namespace is still throttled, the watermark pins again, and the rows come back.

**Everything above assumes a cleanup delete succeeds once it is attempted.** Where deletes are
being attempted and cancelled at the delete query timeout,
[9.4](#94-deleting-task-rows-yourself-and-why-it-is-the-last-resort) is still the step that
applies — but two other responses will suggest themselves first, and both make the situation worse.

**[Section 10](#10-what-not-to-do-when-cleanup-deletes-are-failing) is the companion to this one:
what *not* to do while deletes are failing**, and why. Read it before acting on a cluster in that
state — waiting for it to pass and raising throughput so it can get through are the two natural
responses, and each of them deepens the problem rather than relieving it.

---

## 10. What not to do when cleanup deletes are failing

**You are here because cleanup deletes are being attempted and cancelled at the delete query timeout** — the
third combination in [7.1](#71-start-with-the-cleanup-panels), with
`Error range completing queue task` in the logs and database CPU climbing.

**[Section 9](#9-get-task-row-cleanup-running-again) tells you what to do about it;
[9.4](#94-deleting-task-rows-yourself-and-why-it-is-the-last-resort) is the step that applies. This
section tells you what not to do.** Two responses suggest themselves here, both more intuitive than
the thing that works, and **both make the situation worse**:

| Do not | Because |
|---|---|
| **Wait for it to pass.** The cluster has caught up before. | Nothing in the server makes the next attempt any smaller than the last. A delete too large to finish inside the delete query timeout stays exactly that large, retrying every ten seconds, indefinitely. Waiting does not improve the odds — it only spends database capacity. |
| **Raise task throughput so it can get through.** | The delete is not short of throughput — it is a single statement against a fixed timeout, and more throughput does not make it finish. What it does do is unpin more shards into the same state. |

**The rest of this section is why neither works**, and it is worth reading before acting — both of
them look like the responsible thing to do.

### 10.1 Why a failed cleanup delete stays failed

**A cleanup delete covers a span of the task table**, and
[2.4](#24-one-statement-removes-everything-below-the-watermark) described where that span begins
and ends:

| | |
|---|---|
| **It begins** | at the point deletion last reached successfully. Everything older than that is already gone. |
| **It ends** | at the deletion watermark — the oldest task that has not finished, from [2.3](#23-the-deletion-watermark-is-the-lowest-starting-point-across-all-readers). |

So the span is "everything between where we got to last time and where we may safely delete up to
now", and its size is the number of task rows inside it.

**When a cleanup delete fails, the beginning of that span does not move.** The queue advances it
only after a delete has succeeded; a failed attempt returns without recording anything, so the next
attempt starts from exactly the same place.

**Leaving the beginning where it is is deliberate, and right.** The delete is one transaction —
either every row in the span went or none did — so the queue cannot know how far it got. Moving the
beginning forward anyway would skip rows that still exist, and nothing would ever come back for
them.

**The span therefore never gets smaller, and that is the whole of this section.** A cleanup delete
that was too large to finish inside the timeout is still exactly that large on its next attempt, ten
seconds later, and on every attempt after that. Nothing in the server reduces it. **A delete that has
failed once will go on failing until rows are removed from that span by hand**, which is
[9.4](#94-deleting-task-rows-yourself-and-why-it-is-the-last-resort).

**The span does also grow, though slowly.** New rows keep arriving above the watermark, so the span
creeps wider — on a shard taking a few thousand task rows a day that is a fraction of a percent per
day, which is not what defeats the delete. What defeats it is that a span too large at the first
attempt stays too large at the thousandth. **The growth matters in one case**: during an active
drain, when workflows that had been stuck complete in a rush and emit retention timers of their
own, the span can widen quickly — which is
[9.3](#93-three-signals-that-look-like-failure-during-recovery)'s first warning seen from the
delete's side.

**So this is the one condition in the playbook with a clock running on it.** Everything else here
holds steady until you act. This one does not improve while you decide what to do, and it is not
idle in the meantime — [10.2](#102-what-a-cancelled-delete-costs) is what it spends.

### 10.2 What a cancelled delete costs

A cancelled delete removes no rows, and pays for the attempt anyway:

| | |
|---|---|
| **CPU, I/O and WAL** | the database scans and locks its way through the range until the client gives up. All of that work is discarded. |
| **A connection, each time** | every attempt arrives on a fresh connection — connect, authenticate, run, cancel, disconnect — so TLS and authentication cost is paid on top of the scan. |
| **Nothing to show for it** | no rows removed, so the next attempt faces the same range plus whatever arrived since. |

**Now multiply.** A failing shard retries roughly every ten seconds — about five spent reaching
the timeout, plus a backoff that tops out at five — with no expiry, so it keeps doing this for as
long as the condition lasts. Every affected shard is doing it independently, and
[8.1](#81-get-the-affected-shards-from-the-history-service-logs) tells you how many there are.

**One mercy: a cancelled statement leaves no dead tuples**, because nothing was deleted. The
vacuum pressure comes later, from the cleanup that eventually succeeds — which is ordinary database
maintenance rather than anything specific to this failure.

### 10.3 Task reads get slower too, for a different reason

**Cancelled cleanup deletes are the visible half of the load. Reading tasks gets more expensive
too, for an unrelated reason.** The query the queue uses to load scheduled tasks out of the
database expresses its lower bound as two alternatives joined by
`OR` — either at-or-after a timestamp with a task id at or above a value, or strictly after that
same timestamp. **An `OR` in that position stops the query planner deriving a lower bound on the
`visibility_timestamp` index**, so the scan starts at the oldest row in the shard and filters
forward. On a table holding a few hours of rows that costs nothing. On one holding months of
retained rows, every read walks all of them.

So a shard in this state is paying twice: once for deletes that remove nothing, and once for reads
that scan further than they should, each getting worse as the table grows.

The query shape is tracked as
[temporalio/temporal#12342](https://github.com/temporalio/temporal/issues/12342), which proposes
replacing the two `OR`ed alternatives with a single comparison over both columns at once, so the
planner can bound the scan. **SQL persistence only** — Cassandra is not affected, and the issue was
still open as of **v1.32.1**.

**The query shape is not something you can work around**, which is why this subsection explains a
cost rather than offering a lever. Removing rows shortens the scan along with everything else, so
[9.4](#94-deleting-task-rows-yourself-and-why-it-is-the-last-resort) relieves this too.

### 10.4 What actually clears a delete that keeps failing

**Only one thing: fewer rows in the span.** The timeout is fixed, the delete is unbounded,
and 10.1 showed the span never shrinks by itself — so the only variable left is how many rows the
delete has to match.

> **The procedure is [9.4](#94-deleting-task-rows-yourself-and-why-it-is-the-last-resort), and
> this section deliberately does not repeat it.** In outline: satisfy the four safety conditions,
> delete rows bounded by shard and by time range, let the cleanup deletes start succeeding again,
> and then raise the stuck namespace's limit so whatever remains drains. Every one of those steps
> has conditions attached, and 9.4 is where they are.

**What to expect once the rows are gone**, so you know it has worked:

- The next cleanup delete finishes inside the timeout, and the start of the range finally
  moves forward.
- `Error range completing queue task` stops appearing for that shard.
- Database CPU falls, and in a measured recovery it fell in a single step rather than tapering.
- **The deletes recover unaided** — no restart, and nothing left in a special configuration state.

**The namespace is still throttled, though.** Removing rows cleared
what had piled up, not the reason it piled up, so step 4 of
[9.4](#94-deleting-task-rows-yourself-and-why-it-is-the-last-resort) still remains.

**And check your server version before starting — but do not assume any fix removes the need for
this.** Batched deletes would; a configurable timeout would not, since a large enough range still
reaches whatever ceiling you set — the table in
[9.4](#94-deleting-task-rows-yourself-and-why-it-is-the-last-resort) sets the two apart. If a
configurable timeout is what landed on your version, treat it as breathing room while you do that
work, not as a reason to skip it.

**One more route out exists, and it does not touch the database directly.**
**[Section 11](#11-reloading-shards-to-get-a-stalled-cleanup-running-again)** covers reloading the affected shards — what that does to a failing
delete, what it costs the workflows running on them, and what it means for restarts already
scheduled while the work in [section 9](#9-get-task-row-cleanup-running-again) is under way.

---


## 11. Reloading shards to get a stalled cleanup running again

**Reloading a shard gets task row cleanup running again.** It is a real fix for the failing delete
in [section 10](#10-what-not-to-do-when-cleanup-deletes-are-failing), and that condition does not
recover by itself — without either a reload or the row removal in
[9.4](#94-deleting-task-rows-yourself-and-why-it-is-the-last-resort), the delete goes on failing
indefinitely.

**A reload is what happens whenever a shard is handed to a fresh owner.** Restarting a history pod
causes one for every shard that pod owns, all at once; `tdbg shard close-shard` causes one for a
single named shard. [11.6](#116-how-to-reload-the-affected-shards) covers which of
those to use; the mechanism below holds whichever one causes the reload.

**A shard reload does that job by re-reading every row cleanup never deleted.** The re-read does
not handle fewer rows than the delete had to — it covers exactly the same ones. What differs is
that it reads them **a page at a time, each page its own query with its own time budget**, where
the delete is a single statement that has to cover the whole span at once and gets one budget for
all of it. As the re-read works its way up, the point deletes run behind moves up with it, so each
delete now covers only the ground just re-read, and finishes easily. **So the re-read is the fix.**

**That same re-read is what a reload costs, in full.** Every one of those rows still has to come
off the database. **The rate is capped** — queue loading has its own allowance, deliberately set
below a pod's full database budget so that a reload cannot consume all of it, which
[11.4](#114-what-decides-how-large-the-read-gets) covers. What you cannot do is call a reload off
once it has started, or slow it below the limits already configured, and the work lands on a
database that is already carrying the problem.

**Everything else in this section follows from that double role.** The three things worth checking
before reloading are all questions about the same read:

- **Will it get anywhere?** If the namespace whose tasks stopped draining still cannot run them,
  the reload reads everything back and clears almost nothing.
- **Can the database take it?** The read is the size of the surplus and it lands on the one
  component already struggling, so that component's spare capacity decides.
- **What does it interrupt?** Nothing fails, but the workflows on a reloading shard make no
  progress until it is back.

**[11.6](#116-how-to-reload-the-affected-shards) turns all three into a procedure**, with the
queries, panels and commands to run and the arithmetic to do. The subsections before it are what
that procedure rests on — read them if you want to know why each step is there, or go straight to
11.6 if you do not.

### 11.1 Where a shard resumes from after a reload

Every shard is owned by one history pod at a time. Restarting a pod hands its shards to another
one, which loads each shard's saved state and carries on. Only the shards on the restarted pod are
affected.

Two positions decide what happens next:

- **How far the readers have got** — the position described in
  [2.1](#21-a-queue-works-through-several-ranges-at-once), below which every task has been handled.
- **Where the last cleanup delete finished** — the lower edge of the next `DELETE`, from
  [2.4](#24-one-statement-removes-everything-below-the-watermark).

**Only the reader position is written to the database.** Where the last delete finished is not
stored at all — the server works that out from the saved reader position when the shard loads.

The reader position and the deletion point could drift apart, and the server prevents that by the
order it does things in. At each
checkpoint it issues the cleanup `DELETE` **first**, and writes the reader position afterwards,
only once the delete has succeeded. **When the delete fails, the reader position is not written.**

That ordering has one consequence that matters more than any other here: **while deletes are
failing, nothing is being saved.** The readers go on working in memory and the queue keeps
advancing, but the saved position stays frozen at the last point cleanup successfully reached.

| | Where it is kept | After the pods come back |
|---|---|---|
| How far the readers have got | written to the database, but only once the delete covering that span has succeeded | **restored to the last successfully cleaned position** — not to where the readers had actually got to |
| Where the last cleanup delete finished | not stored — worked out on load from the saved reader position | rebuilt to the same point it was before |
| Everything handled since the last successful delete | held only in memory | **lost on reload, and done again** |

**So a reload sends the shard back to the last point cleanup succeeded, however long ago that was.**
On a healthy cluster that is the previous checkpoint, thirty seconds of work. Where deletes have
been failing for days, it is days.

Going back that far is what produces the re-read, and the next part is what the shard finds there.

### 11.2 Why going back that far means reading the surplus rows again

The span the shard has to work through again — from the last successful delete up to where the
readers had got to — **is exactly the set of rows that cleanup failed to delete.** Everything below
it was deleted successfully. Everything inside it is still sitting in the table, which is the
problem this playbook is about.

**So the rows read back are the rows that were never deleted.** This playbook calls those the
**surplus** from here on — the rows sitting in the table that cleanup should already have removed.
The size of the re-read is the size of the surplus, and nothing else sets it.

**Reading a row back is not the same as doing its work again.** A task read back after its work has
already happened finds nothing left to do and is acknowledged immediately rather than run again —
the third case in [3.2](#32-what-stops-a-task-finishing-and-what-does-not). The cost is reading and
discarding, not duplicate execution. Nothing is corrupted and no row is stranded: **a restart never
leaves rows behind that a later cleanup cannot reach.** The rows counted in
[section 8](#8-confirm-which-shards-and-which-namespace-are-affected) are the same rows
[9.4](#94-deleting-task-rows-yourself-and-why-it-is-the-last-resort) removes, before and after any
number of restarts.

**And this is what gets cleanup running again.** The re-read is not up against the same limit the
delete is. It is **paginated** — a page of `history.timerTaskBatchSize` rows at a time, 100 by
default, each page its own query with its own time budget — so no single read ever has to cover
the whole span, and the re-read as a whole has no deadline to miss. The delete has no such option:
it is one statement for one contiguous range, and it either covers that range inside the timeout
or covers none of it.

**A large surplus does not mean a large amount held in memory, and it does not hold up the shard.**
Two things bound it. The queue stops pulling in new tasks once it is holding
`history.queuePendingTasksMaxCount` of them — **10,000** by default — so a million undeleted rows
are worked through in batches, never loaded at once. And **the shard is owned and serving before
any of this starts**: ownership is a lease taken on the shard record itself, and the re-read runs
behind it in the queue's own loop. A shard with a million rows to get through answers requests
exactly as it did before, while cleanup catches up underneath.

**Reading the rows back is rarely the slow part.** At the default page size and loading rate a
million rows is seconds of reading. What takes the time is putting each of those tasks through the
scheduler to discover there is nothing to do, which is why
[11.3](#113-why-a-throttled-namespace-stops-the-re-read-making-progress) matters more to how long
this takes than the size of the read does.

**The re-read is therefore what breaks the span into pieces the delete can manage.** As the shard
works back through it, the deletion watermark advances a little at each checkpoint — only as far
as the re-read has reached. The next `DELETE` covers that much ground and no more, finishes well
inside the timeout, and **succeeds**. The one oversized statement from
[section 10](#10-what-not-to-do-when-cleanup-deletes-are-failing) is replaced by a long series of
small ones that each work.

| | Before the restart | After the restart |
|---|---|---|
| What the `DELETE` covers | the whole undeleted span, in one statement | one checkpoint's worth of re-read progress |
| Whether it finishes inside the timeout | no — cancelled every time | yes |
| What the database is doing | cancelled deletes, repeatedly | reading the surplus back, then deleting it in small pieces |
| Does the table shrink | no | yes, gradually, as the re-read proceeds |

**Those small deletes are the whole fix, and they depend on the re-read making progress.** The next
part is the one thing that stops it.

### 11.3 Why a throttled namespace stops the re-read making progress

The rows being read back belong to work that is already finished, so most are acknowledged on
sight. **But a task has to be dispatched, and then has to read the workflow's state, before
anything can discover that.** Both of those steps are rate-limited per namespace, and they are
the same two limits [3.1](#31-two-different-limits-refuse-a-task-at-two-different-moments) sets
out:

- **Scheduler admission** — `history.taskSchedulerNamespaceMaxQPS`, which paces how fast a
  namespace's tasks may be started at all. Over that rate they are not dispatched.
- **Persistence** — `history.persistenceNamespaceMaxQPS`, which refuses the database call the
  task makes once it is running.

Each has a playbook of its own:
**[History Task Processing](history-task-processing-tuning.md)** for the scheduler side, and
**[History Persistence QPS Limits](history-persistence-qps-limits.md)** for the persistence side —
both cover how to size the limit rather than just raise it.

**Those two settings default to `0`, meaning no per-namespace limit of their own** — each falls
back to the next limit out, and `history.persistenceMaxQPS` (**9000** per pod) is where the chain
ends. So on a
cluster where neither has been set, neither is what is holding the namespace back, and the cause
is the priority weighting in [section 5](#5-why-some-task-types-are-the-last-to-drain) instead.
[9.1](#91-work-out-whether-a-limit-or-priority-is-holding-the-tasks) is how to tell which.

**Either one is enough to hold the re-read up**, which is the same mechanism that stalled the
queue in [sections 3](#3-why-a-stuck-task-produces-no-errors) to
[6](#6-why-a-deleted-namespace-keeps-its-tasks-alive) — now applied to tasks whose work is already
done.

**So the re-read moves at whatever rate the namespaces in it are allowed.** Two outcomes follow
from the same mechanism:

| If, when the shard comes back | What the re-read does |
|---|---|
| the namespace can run its tasks again — whichever of the two limits was holding it has been raised, or its scheduler priority restored | it drains, the watermark advances, and the small deletes clear the surplus |
| either limit is still refusing its tasks, or its priority still leaves them waiting | it crawls at whatever rate the namespace is allowed, the watermark barely moves, few deletes are issued, and **the table keeps growing** while new tasks are still being written |

**Reloading before that namespace can run its tasks pays the whole cost of the re-read and clears
very little.** That is the first of the two checks at the top of this section, and
[9.1](#91-work-out-whether-a-limit-or-priority-is-holding-the-tasks) is where it is answered.

**Restarting more pods, or the same pod again, does not move a throttled namespace out of the way.**
Each restarted pod re-reads
its own shards under the same condition, so restarting the fleet multiplies the read across every
shard at once rather than making any of them more likely to drain. Restarting the same pod again
sends it back to the same position to read the same rows a second time, discarding whatever
progress the first re-read had made in memory. **If the first restart did not clear it, the throttle
is the reason, and another restart does not address the throttle.**

With the first check settled, the second one is about size — how much reading a restart actually
sets off.

### 11.4 What decides how large the read gets

**Queue loading is rate-limited, and reloads are why.** A pod does not read tasks at whatever rate
the database will take. It reads at a rate capped **below**
its own overall database allowance, specifically so that loading tasks for every shard at once —
which is what a restart causes — cannot use up the whole budget. For timer queues that cap is
`history.timerProcessorMaxPollHostRPS`, and left at its default of **0** it falls back to
**30 per cent** of `history.persistenceMaxQPS`.

**So a reload cannot saturate the database on its own.** What is worth knowing is how much it is
allowed, because on a default cluster that allowance is larger than the per-namespace settings
alone would suggest.

| Setting | Default | What it governs during a reload |
|---|---|---|
| `history.persistenceMaxQPS` | **9000** | what one history pod may ask of the database per second, on its own |
| `history.persistenceGlobalMaxQPS` | **0** — off | when set, a pod's allowance becomes a share of a cluster-wide figure, scaled by how many shards that pod owns. Left off, no cluster-wide ceiling exists at all |
| `history.persistenceNamespaceMaxQPS` | **0** — off | when set, caps one namespace's share of a pod's allowance. Left off, a single namespace may use the whole of it |

Three consequences follow, and they compound:

- **A pod that has just started gets its full allowance straight away.** It does not ramp up, and
  with no cluster-wide figure configured there is no share for it to work out.
- **Nothing holds the cluster total down.** Each pod is permitted its own figure independently, so
  the effective ceiling is that figure multiplied by the number of pods.
- **One namespace can take all of it.** With no per-namespace cap, the namespace holding the
  largest surplus — the one whose tasks stopped draining in the first place — is exactly the one
  whose re-read fills the allowance.

**If a cluster-wide figure is configured, the first minute after a pod starts runs on a different
number.** A pod re-reads its own quota once a minute, and at the first read its view of cluster
membership has not loaded yet, so it falls back to the per-pod setting instead of its cluster-wide
share. **Measured**: a pod set to 16000 under a cluster-wide limit of 36000 enforced 16000 on
startup and switched to its 20320 share exactly sixty seconds later —
[2.5 of the QPS playbook](history-persistence-qps-limits.md#25-a-global-limit-replaces-the-per-pod-one)
has the detail and how to read the number off the pod.

**That first minute is the one that matters here.** If the per-pod setting is higher than the pod's
fair share, the pod spends its first minute allowed to ask for more than it should — exactly the
minute its cache is cold and its shards are reading the surplus back.

**Setting these limits is what bounds the read**, and each side has its own playbook:

- **[History Persistence QPS Limits](history-persistence-qps-limits.md)** — the settings above and
  how to choose them. Start at
  **[1.2, on not relying on the defaults](history-persistence-qps-limits.md#12-set-them-deliberately--do-not-rely-on-the-defaults)**,
  and see
  **[2.6 for what each one does when left at `0`](history-persistence-qps-limits.md#26-what-each-setting-does-when-set-to-0)**.
- **[History Task Processing — Tuning and Troubleshooting](history-task-processing-tuning.md)** —
  holding down how much one busy namespace is allowed to push through the task scheduler.

**Limits already in place change what a restart costs; limits changed during one do not help much**,
because the re-read still has to happen. Treat them as something to have set in advance.

With both checks answered, the choice between reloading and removing the rows can be made on
its merits.

### 11.5 What a reload does to the workflows on a reloading shard

**Every reload pauses the workflows on the shard being reloaded.** While a shard is moving, calls
to it are answered with a shard-ownership error, and the caller re-resolves the new owner and
retries on its own. **No work is lost** — workflow state is durable throughout, nothing is dropped,
and no workflow fails because of a reload.

**What the workflows on a reloading shard do see is a pause.** Every workflow task, activity completion and timer
belonging to that shard waits while it is handed over, re-acquired and reloaded, and then runs
against a cold cache until it warms. On live production traffic that shows up as a latency bump —
not as failures, but not as nothing either.

**How much traffic that covers is the whole difference between restarting a pod and closing a
single shard.** A history pod holds every shard assigned to it — on a cluster with 2048 shards and a dozen pods,
well over a hundred — so restarting one pauses all of them together, including every shard with
nothing wrong with it. Closing a single shard pauses that shard alone.

**Which is often what decides this, and it is a separate question from whether a reload would
work.** Removing the rows in
[9.4](#94-deleting-task-rows-yourself-and-why-it-is-the-last-resort) touches the database only: no
shard moves, no cache is lost, and running workflows carry on undisturbed. **A cluster that cannot
accept a latency bump on live traffic has its answer already**, whatever the surplus looks like.

**The size of that pause is something you choose**, and the next part is how to keep it to the
shards that actually need reloading.

### 11.6 How to reload the affected shards

Three checks, then the reload itself. **Each check has something to run or read** — none of them is
a judgement call.

#### Check 1 — are cleanup deletes actually being attempted?

Open **[Task Row Cleanup Attempts by Category](../observability/dashboards/server/temporal-server-readme.md#23-history-task-cleanup)**
and **[Task Row Cleanup Failures by Category](../observability/dashboards/server/temporal-server-readme.md#23-history-task-cleanup)**
for the category [8.1](#81-get-the-affected-shards-from-the-history-service-logs) named.

| What the two panels show | What to do |
|---|---|
| **attempts present, failures present** | a reload is the right tool — carry on to check 2 |
| **attempts absent or near zero** | **stop.** The watermark is pinned, so the namespace still cannot run its tasks and a reload will read everything back and clear almost nothing. Go to [9.1](#91-work-out-whether-a-limit-or-priority-is-holding-the-tasks) and [9.2](#92-raise-a-namespace-limit-or-a-priority-weight-in-steps) first, and come back when attempts appear |

**Those two rows are the same reading as the three combinations in
[7.1](#71-start-with-the-cleanup-panels)**, applied to one decision.

#### Check 2 — whether the database can take the read

**Start with how much room the database has, not with how many rows there are.** The re-read is
extra load on the one component already struggling, so its spare capacity is what decides whether
a reload is safe — and it is the gate on everything else in this check.

| Database CPU before you start | What to do |
|---|---|
| **already high** | **do not reload.** Remove the rows instead — [9.4](#94-deleting-task-rows-yourself-and-why-it-is-the-last-resort) lets you choose the batch size, the rate and the hour, none of which a reload does |
| **comfortable** | a reload is reasonable, and the rest of this check tells you how much to expect |

**One history pod can be a large share of that headroom.** A pod restart reloads every shard it
owns at once, so the read arriving is the sum of those shards' surpluses, not one shard's — and
cold caches and the first-minute quota from
[11.4](#114-what-decides-how-large-the-read-gets) land on top of it. **Measured on one cluster:
restarting a single history pod of twelve took database CPU from roughly 20 per cent to roughly
40 per cent for a few minutes before it settled.** That is one measurement on one cluster and not
a threshold — what carries over is the shape, that one pod out of twelve can cost a fifth of the
database. Reloading one shard with `close-shard` costs a small fraction of that.

**With the headroom established, size the read.** Count the rows on one affected shard with the
query in [8.3](#83-tell-a-stuck-cohort-from-ordinary-retained-traffic), then work out its floor.

**A shard does not read its tasks back in one go. It reads a page at a time** — one `SELECT`
returning a fixed number of rows, then the next, working up through the span. Both how many rows
a page holds and how many pages a pod may fetch per second are capped:

| | Setting | Default |
|---|---|---|
| how many pages a pod may fetch per second, across all its timer queues | `history.timerProcessorMaxPollHostRPS` | **0**, meaning fall back to `history.persistenceMaxQPS` x **0.3** — so **2700** on defaults |
| how many rows one page returns | `history.timerTaskBatchSize` | **100** |

**On defaults that is 2700 x 100 = 270,000 rows per second**, shared across every shard reloading on
that pod. So:

```
fastest possible re-read  =  rows to read back  /  270,000 per second
```

**That division gives a floor rather than an estimate**, and acting on it goes like this:

| What the floor comes to | What to do |
|---|---|
| **long enough that you would not want the database working that hard for that long** | do not reload. Remove the rows instead — [9.4](#94-deleting-task-rows-yourself-and-why-it-is-the-last-resort) clears the same rows without any of the read |
| **short, which it usually will be** | the read is not what will hold you up, so this check does not stop you. **It does not clear you either** — carry on to check 3, and expect the real duration to be set by [check 1](#check-1--are-cleanup-deletes-actually-being-attempted) |

**A short floor is the common case, and it is worth knowing why.** At the defaults above even a
million rows is a few seconds of pure reading. What takes real time is putting each of those tasks
through the scheduler afterwards to find there is nothing left to do, and how fast that goes is
decided by whether the namespace can run its tasks at all —
[check 1](#check-1--are-cleanup-deletes-actually-being-attempted) again, with
[11.3](#113-why-a-throttled-namespace-stops-the-re-read-making-progress) for the reason.

**Substitute your own numbers if either setting has been changed from its default** — the rate
caps are the ones [11.4](#114-what-decides-how-large-the-read-gets) describes.

#### Check 3 — how much live traffic stops

Reloading pauses the workflows on the shards being reloaded, so the share of your traffic affected
is the share of your shards being reloaded:

| What you reload | Fraction of traffic that pauses |
|---|---|
| one shard with `close-shard` | **1 / `numHistoryShards`** — on a 2048-shard cluster, well under a tenth of a per cent |
| one pod | **that pod's shards / `numHistoryShards`** — a twelfth of the cluster on a twelve-pod cluster, and `tdbg shard describe` names which pod holds a given shard |

**Reloading one shard at a time is almost always worth it just for the smaller blast radius** —
the same fix, against a fraction of the traffic.

#### Then reload, one shard at a time

```bash
tdbg shard close-shard --shard-id <id>
```

That stops the shard and removes it from the pod holding it. It comes back on the next request that
touches it, or through the background acquisition loop within `history.acquireShardInterval`
(default **1 minute**), and resumes exactly as
[11.1](#111-where-a-shard-resumes-from-after-a-reload) describes — **for that one shard**.

After each shard, look at three things before doing the next:

| Look at | Carry on if |
|---|---|
| the two cleanup panels from [check 1](#check-1--are-cleanup-deletes-actually-being-attempted) | failures give way to clean attempts for that category |
| **[Persistence Latencies](../observability/dashboards/server/temporal-server-readme.md#3-persistence-requests-latencies-and-errors)** | the step up comes back down rather than staying up |
| the row count from [8.3](#83-tell-a-stuck-cohort-from-ordinary-retained-traffic), re-run | it is falling |

**A reading that goes the wrong way is reason to stop, not to move on to the next shard.** Which
check it sends you back to depends on which reading it is:

| What you see | Which check it sends you back to |
|---|---|
| failures carry on, or attempts fall away to nothing | [check 1](#check-1--are-cleanup-deletes-actually-being-attempted) — the namespace still cannot run its tasks |
| persistence latency steps up and stays up | [check 2](#check-2--whether-the-database-can-take-the-read) — the read is larger than the database is comfortably absorbing |
| the shard is back but its workflows are visibly behind | [check 3](#check-3--how-much-live-traffic-stops) — more traffic is paused than you accounted for |

**If you restart whole pods instead**, find the right ones rather than guessing:

```bash
tdbg shard describe --shard-id <id>
```

The `owner` field names the host currently holding that shard. **Restart one pod at a time** —
[11.3](#113-why-a-throttled-namespace-stops-the-re-read-making-progress) covers why restarting the
fleet together multiplies the read rather than improving the odds.

With a procedure in hand, the last question is whether to reload at all.

### 11.7 Choosing between a reload and removing the rows


**A restart and a manual row removal both end with cleanup running again.** Removing the rows works
without the re-read and without depending on anything being dispatched; a restart works only if the re-read drains. The
difference shows up at every step:

| | Reloading the shards (`close-shard`, or a pod restart) | [Deleting the rows yourself (9.4)](#94-deleting-task-rows-yourself-and-why-it-is-the-last-resort) |
|---|---|---|
| **Effect on running workflows** | **those on the reloaded shards pause and run cold for a while** | **none — no shard moves** |
| How cleanup catches up | the shard reads the whole surplus back, deleting it in small pieces | the rows are gone, so the next delete is small and succeeds |
| Whether it works at all | only if the namespaces in the re-read can be dispatched | always — rows do not need to be dispatched to be removed |
| What the database does | reads every surplus row, then deletes it | deletes in controlled batches, at a time you pick |
| How much control you have | none once the pods go down | batch size, rate and timing are all yours |
| Which shards are affected | all shards on the restarted pod, together | the shards you choose |
| Effort | one operation | a procedure, run against the database |

**Which makes the choice a question of size.** A surplus the database can comfortably read back in
one go is a reload; a surplus large enough that reading it back is itself a risk is
[9.4](#94-deleting-task-rows-yourself-and-why-it-is-the-last-resort) first, after which there is
nothing left to re-read and a restart is not needed at all.
[Section 8](#8-confirm-which-shards-and-which-namespace-are-affected) is what tells you which of
those you have.

**Restarts that are already scheduled need a different answer than restarts you would spend on
purpose.** A deploy, a node replacement or a maintenance window carries no correctness risk:
nothing is lost and nothing is stranded, so there is never a correctness reason to halt one.

**Whether to move it is a question about the database, and the same one check 2 asks.** A
scheduled restart reloads every shard on that pod together, so on a database that is already hot
it arrives as a step on top of a cluster that has no room for one.

- **Database CPU high, and the window is yours to move** — move it, and clear the rows first with
  [9.4](#94-deleting-task-rows-yourself-and-why-it-is-the-last-resort). Once the surplus is gone
  there is nothing left to re-read and the restart costs what a restart normally costs.
- **Database CPU high, and the window is not yours** — let it happen, but watch
  [Persistence Latencies](../observability/dashboards/server/temporal-server-readme.md#3-persistence-requests-latencies-and-errors)
  through it, and have [9.4](#94-deleting-task-rows-yourself-and-why-it-is-the-last-resort) ready
  rather than waiting to see whether the reload clears things on its own.
- **Database CPU comfortable** — let it happen. The reload may well do the work for you on that
  pod's shards, and the panels below are how you tell.

Four things to expect afterwards, so that none of them is misread:

| What you will see | What it means |
|---|---|
| **Database load rises before the table starts shrinking** | the rows are read back before they are deleted. Rising load after a restart is the recovery working, not a new fault |
| **Database load settles while the table is still large** | the startup burst has ended, which on a container platform is usually a few minutes. The re-read carries on underneath it. **Never judge completion from the database CPU graph** — use the two panels in the row below, and the row count from [8.3](#83-tell-a-stuck-cohort-from-ordinary-retained-traffic), since no metric reports the size of a task table |
| **[Task Row Cleanup Success Rate](../observability/dashboards/server/temporal-server-readme.md#23-history-task-cleanup)** back at 100%, and **[Task Row Cleanup Latency by Category](../observability/dashboards/server/temporal-server-readme.md#23-history-task-cleanup)** back to single-digit milliseconds | cleanup is running again on that shard's category. The latency panel is the sharper of the two: a delete being cancelled reports p99 in the **5–10 second** band, so the drop from there to milliseconds is unmistakable |
| **[Task Row Cleanup Attempts by Category](../observability/dashboards/server/temporal-server-readme.md#23-history-task-cleanup) falls away, instead of [Task Row Cleanup Failures by Category](../observability/dashboards/server/temporal-server-readme.md#23-history-task-cleanup) clearing** | the re-read has re-pinned on a throttled namespace. Attempts stopping altogether is the pinned-watermark reading from [7.1](#71-start-with-the-cleanup-panels), and it sends you back to [9.1](#91-work-out-whether-a-limit-or-priority-is-holding-the-tasks) |

**Two more panels are worth having open for the restart itself**, alongside the cleanup panels
above: **[Owned Shards (Total)](../observability/dashboards/server/temporal-server-readme.md#8-shard-movement)**,
which should return to your configured shard count and says the pod has finished coming back, and
**[Persistence Latencies](../observability/dashboards/server/temporal-server-readme.md#3-persistence-requests-latencies-and-errors)**,
where the read shows up as a step that should come back down rather than stay up.

With the restart question settled, the remaining one is how to be told about any of this without
watching a dashboard. **[Section 12](#12-alerting-on-a-stalled-cleanup)** covers which of the two
failure shapes can be alerted on, which cannot, and why.

---

## 12. Alerting on a stalled cleanup

**Cleanup reports a failing delete and a pinned watermark differently, and that difference decides
what can be alerted on.** A delete that is attempted and fails is counted, and on a healthy
cluster that count sits at a true zero — which makes it about the easiest thing there is to alert
on. A watermark that is pinned shows up the
other way round, as deletes no longer being attempted, and that reading is just as faithful. What
it is not is unambiguous: a queue with nothing to do reports the same zero, quite correctly.

**So a failing delete belongs in an alert and a pinned watermark belongs on a dashboard**, and
that split is what this section is about.

### 12.1 Alert 91 catches a cleanup delete that is failing

**[Alert 91 — Task Row Cleanup Failing](../observability/alerts/server/alerts-index.md#alert-91--task-row-cleanup-failing)** is written for
exactly the condition in [section 10](#10-what-not-to-do-when-cleanup-deletes-are-failing). The
alerts index carries the expression, the severity and the thresholds; what matters here is why it
is an unusually easy alert to run.

**Cleanup deletes never fail on a healthy cluster.** The baseline is a true zero rather than a low
number, so there is no threshold to tune and no quiet period to calibrate against — **any sustained
failure is actionable**. It is set to fire after **10 minutes**, which is long enough to ignore a
database blip and short enough to catch the condition while the table is still small.

**What to do when it fires** is the whole of this playbook, in order:

| Step | Go to |
|---|---|
| Confirm which shards, and which namespace behind them | [8.1](#81-get-the-affected-shards-from-the-history-service-logs) |
| Clear whatever is stopping that namespace's tasks | [9](#9-get-task-row-cleanup-running-again) |
| Deal with the rows the delete can no longer reach | [11.6](#116-how-to-reload-the-affected-shards) or [9.4](#94-deleting-task-rows-yourself-and-why-it-is-the-last-resort) |
| Check what not to try | [10](#10-what-not-to-do-when-cleanup-deletes-are-failing) |

**Alert 91 is in the Essential Set**, so it ships in
[`temporal-server-alerts.yaml`](../observability/alerts/server/temporal-server-alerts.yaml) with
the rest and needs no work to provision beyond deploying that file. Its
**[runbook](../observability/alerts/server/runbooks/91-task-row-cleanup-failing.md)** is
deliberately short: it names the panels, says what not to do, and routes here. **The procedure
lives in this playbook rather than being restated there**, so there is one copy to keep right.

**Failing deletes are the half that alerts cleanly.** A pinned watermark does not, and the next
part is why.

### 12.2 Why a pinned watermark cannot be alerted on the same way

The quiet failure from [sections 3](#3-why-a-stuck-task-produces-no-errors) to
[6](#6-why-a-deleted-namespace-keeps-its-tasks-alive) shows up as
**[Task Row Cleanup Attempts by Category](../observability/dashboards/server/temporal-server-readme.md#23-history-task-cleanup)** falling to zero: the watermark has not moved,
so no delete is issued, so there is nothing to count.

**A category with no work reports exactly the same thing**, which is what makes it unalertable. An archival queue
on a cluster that archives nothing, a visibility queue on a namespace with no traffic — both sit at
zero attempts permanently and correctly. An alert on "attempts are zero" would fire on every one of
them.

**Making it work needs a guard that says the category has traffic at all**, and that guard is the
hard part rather than the alert:

| What you would have to establish | Why it is awkward |
|---|---|
| that the category is receiving tasks at all | task creation and task cleanup are different metrics on different operations, so the guard is a second expression that has to agree with the one it is guarding |
| that it has been receiving them long enough to matter | a queue that has just started, or a pod that has just taken a shard, legitimately reports nothing yet |
| that the silence is not simply a quiet period | traffic that is genuinely bursty drops to zero between bursts without anything being wrong |

**None of that is impossible, and none of it is worth paging on.** A pinned watermark is not a
sudden event — it is a condition that holds for hours or days, and the table grows slowly enough
that reading the panels on a schedule catches it in good time.
**[7.1](#71-start-with-the-cleanup-panels)** is that reading, and it takes one glance at four
panels.

**So the honest answer is: alert on the loud half, and look at the quiet half.** Which leaves one
more thing worth alerting on, upstream of both.

### 12.3 Alert on the namespace that stops draining, not only on cleanup

Both failures in this playbook start the same way — a namespace whose tasks stop draining, from
[3](#3-why-a-stuck-task-produces-no-errors). **That has its own alerts, and they fire earlier than
anything here**, because they catch the throttling while it is still only throttling:

| Alert | Catches |
|---|---|
| **[Alert 86 — History Database Calls Rejected](../observability/alerts/server/alerts-index.md#alert-86--history-database-calls-rejected)** | database calls being refused by a persistence limit, which is one of the two limits in [3.1](#31-two-different-limits-refuse-a-task-at-two-different-moments) |
| **[Alert 87 — History Write-Reject Loop](../observability/alerts/server/alerts-index.md#alert-87--history-write-reject-loop)** | rejection that has become self-feeding, which is what makes a reload expensive in [11.4](#114-what-decides-how-large-the-read-gets) |
| **[Alert 38 — Timer Task Scheduling Lag Critical](../observability/alerts/server/alerts-index.md#alert-38--timer-task-scheduling-lag-critical)** | timer tasks running late, which a pinned queue eventually produces |

**Those three are not a substitute for alert 91.** Throttling that clears is normal and will fire them
without any cleanup problem existing, which is the distinction
[3](#3-why-a-stuck-task-produces-no-errors) draws. What they give you is a chance to deal with the
namespace before it has held the watermark long enough for the table to matter — and dealing with
it then is [9.2](#92-raise-a-namespace-limit-or-a-priority-weight-in-steps) alone, with none of the
row removal or reloading this playbook otherwise ends in.

With detection, confirmation, remedy and alerting all covered, the last thing left is every setting
the playbook has named along the way.
**[Section 13](#13-every-setting-named-in-this-playbook)** collects them in one place.

---

## 13. Every setting named in this playbook

**Every value below was read from the server source for v1.32.1.** Nothing here is a recommended
figure — defaults are what the server does if you say nothing, and the section that discusses each
one is where its trade-off is argued. **Check your own cluster's values before assuming a default
applies**: a setting changed years ago for an unrelated reason is a common cause of the conditions
in this playbook.

### 13.1 How often cleanup runs

One checkpoint per shard per interval, and the cleanup `DELETE` is issued as part of it — but only
when the deletion watermark has moved.

| Setting | Default | Where discussed |
|---|---|---|
| `history.timerProcessorUpdateAckInterval` | **30s** | [Introduction](#when-cleanup-runs) |
| `history.transferProcessorUpdateAckInterval` | **30s** | [Introduction](#when-cleanup-runs) |
| `history.visibilityProcessorUpdateAckInterval` | **30s** | [Introduction](#when-cleanup-runs) |
| `history.archivalProcessorUpdateAckInterval` | **30s** | [Introduction](#when-cleanup-runs) |
| `history.outboundProcessorUpdateAckInterval` | **30s** | [Introduction](#when-cleanup-runs) |

**There is no single setting covering all five**, and raising them does not help a failing delete —
[9.2](#92-raise-a-namespace-limit-or-a-priority-weight-in-steps) explains why a cancelled delete
retries on its own backoff and ignores this interval entirely.

### 13.2 What limits a namespace's tasks

The two limits that decide whether a namespace's tasks can run, and the priority weighting that
decides how often they get a turn.

| Setting | Default | Where discussed |
|---|---|---|
| `history.taskSchedulerNamespaceMaxQPS` | **0** — falls back to the namespace persistence limit | [3.1](#31-two-different-limits-refuse-a-task-at-two-different-moments), [11.3](#113-why-a-throttled-namespace-stops-the-re-read-making-progress) |
| `history.taskSchedulerGlobalNamespaceMaxQPS` | **0** — off | [9.2](#92-raise-a-namespace-limit-or-a-priority-weight-in-steps) |
| `history.persistenceNamespaceMaxQPS` | **0** — falls back to the per-pod limit | [3.1](#31-two-different-limits-refuse-a-task-at-two-different-moments), [11.3](#113-why-a-throttled-namespace-stops-the-re-read-making-progress) |
| `history.timerProcessorSchedulerActiveRoundRobinWeights` | **High 10, Low 9, Preemptable 1** | [5.1](#51-the-three-priority-classes-and-what-lands-in-each) |

**The scheduler and persistence namespace limits both default to off**, so on a cluster where
neither was ever set, neither is what is holding a namespace back — the priority weighting is. **[History Task Processing](history-task-processing-tuning.md)**
covers the scheduler side and **[History Persistence QPS Limits](history-persistence-qps-limits.md)**
the persistence side, both on how to size a limit rather than just raise it.

### 13.3 What a pod is allowed to ask of the database

These decide how much read a reload sets off, and how much of it one namespace can take.

| Setting | Default | Where discussed |
|---|---|---|
| `history.persistenceMaxQPS` | **9000** per pod | [11.4](#114-what-decides-how-large-the-read-gets) |
| `history.persistenceGlobalMaxQPS` | **0** — off, so no cluster-wide ceiling | [11.4](#114-what-decides-how-large-the-read-gets) |
| `history.timerProcessorMaxPollHostRPS` | **0** — falls back to 30% of `history.persistenceMaxQPS` | [11.4](#114-what-decides-how-large-the-read-gets), [11.6](#116-how-to-reload-the-affected-shards) |
| `history.timerTaskBatchSize` | **100** rows per page | [11.2](#112-why-going-back-that-far-means-reading-the-surplus-rows-again), [11.6](#116-how-to-reload-the-affected-shards) |
| `history.queuePendingTasksMaxCount` | **10000** | [11.2](#112-why-going-back-that-far-means-reading-the-surplus-rows-again) |
| `history.acquireShardInterval` | **1m** | [11.6](#116-how-to-reload-the-affected-shards) |

### 13.4 How a queue organises itself

Readers and scopes, and when the server moves a heavy namespace onto a second reader.

| Setting | Default | Where discussed |
|---|---|---|
| `history.timerQueueMaxReaderCount` | **2** | [2.1](#21-a-queue-works-through-several-ranges-at-once) |
| `history.transferQueueMaxReaderCount` | **2** | [2.1](#21-a-queue-works-through-several-ranges-at-once) |
| `history.visibilityQueueMaxReaderCount` | **2** | [2.1](#21-a-queue-works-through-several-ranges-at-once) |
| `history.archivalQueueMaxReaderCount` | **2** | [2.1](#21-a-queue-works-through-several-ranges-at-once) |
| `history.outboundQueueMaxReaderCount` | **4** | [2.1](#21-a-queue-works-through-several-ranges-at-once) |
| `history.queueMoveGroupTaskCountBase` | **500** | [4.2](#42-the-server-moves-namespaces-that-fall-far-enough-behind-onto-a-second-reader) |
| `history.timerProcessorMaxPollInterval` | **5m** | [7.3](#73-what-the-queue-lag-panels-can-and-cannot-tell-you) |

### 13.5 Namespace deletion

Both belong to the frontend, not the history service, and both pace the deletion workflow that
[section 6](#6-why-a-deleted-namespace-keeps-its-tasks-alive) describes.

| Setting | Default | Where discussed |
|---|---|---|
| `frontend.deleteNamespaceDeleteActivityRPS` | **100** | [6.3](#63-how-to-recognise-a-stalled-namespace-deletion) |
| `frontend.deleteNamespaceConcurrentDeleteExecutionsActivities` | **4** | [6.3](#63-how-to-recognise-a-stalled-namespace-deletion) |

### 13.6 Behaviour that is fixed rather than configurable

Three things this playbook turns on are set in the server rather than exposed as settings, which
is why the remedies take the shape they do. **Two of the three are already being looked at**, and
the [note to readers](#note-to-readers) at the top has the detail:

| Behaviour today | Why it shapes the remedy | Where it is going |
|---|---|---|
| **The delete query timeout** — five seconds, compiled in | an oversized delete cannot simply be given longer, so the span has to be made smaller instead | **[#12341](https://github.com/temporalio/temporal/issues/12341)** proposes making it a dynamic config setting |
| **One statement per delete** — no `LIMIT`, no chunking | the span cannot be split by configuration, which is why [9.4](#94-deleting-task-rows-yourself-and-why-it-is-the-last-resort) and [section 11](#11-reloading-shards-to-get-a-stalled-cleanup-running-again) exist | **[#12341](https://github.com/temporalio/temporal/issues/12341)** also proposes bounded batches, which would remove the problem rather than move it |
| **The retry backoff after a failed delete** — 100ms growing to a ceiling of five seconds | a failing shard reattempts roughly every ten seconds rather than waiting for the next checkpoint, which is why raising the checkpoint interval does nothing | nothing open, and nothing needed — retrying promptly is the right behaviour here |

---
