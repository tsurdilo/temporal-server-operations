# Visibility and Archival Tasks in the DLQ — Detection & Recovery Playbook

## Visibility, archival and namespace retention

A Temporal cluster always has two persistence stores. The **primary persistence store** (such as Cassandra,
PostgreSQL or MySQL) holds the workflows and their event histories. The **visibility store**,
Elasticsearch in this document, holds a searchable record of each workflow (its ID, type, status,
start and close times and search attributes) that `temporal workflow list` and the UI use to list
and filter workflows. The visibility store is not optional: the server does not start without one.

**Namespace retention** is how long a closed workflow is kept before the server deletes it. Every
namespace has a retention period. The server requires one when the namespace is created, and the
tools that create namespaces fill one in when none is given: `temporal operator namespace create`
uses **72 hours** unless `--retention` says otherwise. It can be changed later with
`temporal operator namespace update --retention`; a change does not move deletes that are already
scheduled.

When a workflow's retention period runs out, the workflow is deleted from both stores: its event
history from the primary persistence store and its record from the visibility store.

A cluster can also enable **archival**, an optional feature that keeps a copy in the user's own
storage, the **archival backend**, such as an S3 or GCS bucket. History archival copies each closed
workflow's event history and visibility archival copies its visibility record; each is enabled on
its own, for the cluster and per namespace. Both copies can still be read after namespace retention
has deleted the workflow from the primary and visibility stores.

Archival and namespace retention run on separate clocks. Archival starts soon after the workflow
closes: its task is due at a random time up to `history.archivalProcessorArchiveDelay` (default **5
minutes**) after the workflow closes. Retention is also counted from the workflow's close time, but
runs out only after the namespace's retention period, which by default cannot be set below 1 hour
for a local namespace or 1 day for a global one. In normal operation, archival is done long before
namespace retention is reached.

> **Note:** On Temporal Cloud, namespace retention is set in whole days, so the minimum is **1 day**
> for every namespace, global or not. On a self-hosted cluster, we recommend the same: set namespace
> retention to at least 1 day, even for a local namespace where the server accepts as little as 1 hour.

Archival and retention are linked in one way: with archival enabled, the retention delete is
scheduled **only after archival succeeds**, so a workflow is not deleted before it is archived. By
default, visibility has no such link: retention deletes the workflow on time whether or not its
final visibility record was written.

One setting, `history.visibilityProcessorEnableCloseWorkflowCleanup`, can give visibility such a
link. Turning it on saves some space in the primary persistence store, but makes archival depend on
Elasticsearch, so an Elasticsearch failure can then hold up archival and namespace retention.

> **Recommendation:** keep `history.visibilityProcessorEnableCloseWorkflowCleanup` at its default,
> off. The rest of this document assumes it is off. If it is turned on in your cluster,
> [section 6](#6-close-workflow-cleanup-turned-on) covers what changes.

Normally, within minutes of a workflow closing, its visibility record shows it as closed and its
event history is archived, long before namespace retention deletes it. This document looks at what
happens when that does not go to plan, because the visibility store, the archival backend, or both
keep failing. Depending on the failure, the failing tasks either keep waiting until the store is
back, or are given up on and moved to the server's dead-letter queue (DLQ). Either way, the
workflow's retention date may arrive before the operator has fixed the store. What the server
does when the retention date arrives first is different for visibility and for archival, and is the
main subject of this document.

Updating the visibility record, archiving and deleting the workflow are each done by a task that
the history service creates and then processes. To follow what happens to these tasks when a store
is failing, the next section lists them. Three tasks are created when the workflow closes: one
updates its visibility record, one archives it, and one reports the result to a parent workflow
and applies the parent close policy to child workflows.
Two more tasks run when its namespace retention period is over: one deletes the workflow from the
primary persistence store, and one deletes its visibility record.

## Tasks created when a workflow closes

A workflow **closes** when it completes, fails, times out, is canceled, is terminated or continues
as new. All six ways of closing create the same tasks, listed below.

When a workflow closes, the history service creates tasks that write what visibility and archival
need, and runs them through its normal task processing. With archival enabled, three tasks are
created, in the same database write that records the workflow as closed. Each task's type is shown in
parentheses.

- **Close transfer task** (`TASK_TYPE_TRANSFER_CLOSE_EXECUTION`). Reports the result to a parent
  workflow and applies the parent close policy to child workflows. It does not use the visibility
  store or the archival backend, so the failures described below do not affect it. Stored in
  `transfer_tasks` with SQL persistence, and as a row in `executions` with Cassandra persistence.
- **Close execution visibility task** (`TASK_TYPE_VISIBILITY_CLOSE_EXECUTION`). Writes the closed
  state to the visibility store (Elasticsearch). Stored in `visibility_tasks` with SQL persistence,
  and as a row in `executions` with Cassandra persistence.
- **Archival task** (`TASK_TYPE_ARCHIVAL_ARCHIVE_EXECUTION`). Copies the workflow's event history
  and a visibility entry to the archival backend (S3, GCS and others). It is due at a random time up
  to `history.archivalProcessorArchiveDelay` (default **5 minutes**) after the workflow closes.
  Stored in `history_scheduled_tasks` with SQL persistence, and as a row in `executions` with
  Cassandra persistence.

Two more tasks carry out namespace retention:

- **Retention delete task** (`TASK_TYPE_DELETE_HISTORY_EVENT`). With archival enabled it is created
  when the archival task succeeds, normally within minutes of the workflow closing; without archival
  it is created in the same write that records the workflow as closed. It then waits until the
  workflow's close time + namespace retention + a random delay of up to
  `history.retentionTimerJitterDuration` (default **30 minutes**), and deletes the workflow and its
  event history from the database. Stored in `timer_tasks` with SQL persistence, and as a row in
  `executions` with Cassandra persistence.
- **Delete execution visibility task** (`TASK_TYPE_VISIBILITY_DELETE_EXECUTION`). Created by the
  retention delete task when it runs, at the end of the retention period, and removes the workflow's
  visibility record right away. Stored in `visibility_tasks` with SQL persistence, and as a row in
  `executions` with Cassandra persistence.

All five task types are processed like every other history task: the history shard that owns the
workflow reads the task back from the database, and the history service runs it.
[How task processing works](./history-task-processing-tuning.md#2-how-task-processing-works)
in the History Task Processing playbook covers loading, scheduling and running these tasks, and the
settings for each.

[Section 8](#8-task-types-named-in-the-sections-above) lists the five task types with the number each
is stored as, its queue and `--dlq-type`, its table with SQL persistence, its row type with
Cassandra persistence and its metric label.

## Failures, the DLQ and the scenarios covered

From here on, the document follows one cluster whose visibility store, archival backend, or both
keep failing. For each of these three scenarios it shows whether the failing tasks reach the DLQ, and
what the server does when the operator merges the DLQ **before** or **after** the workflows'
retention date.

**Cluster setup:** a Temporal Server cluster with **both visibility and archival enabled**, using
Elasticsearch as the visibility store.

**How visibility and archival tasks end up in the DLQ:**

- **The failure:** the visibility store or the archival backend keeps failing, so processing of the
  visibility and archival tasks keeps failing too.
  [How task processing works](./history-task-processing-tuning.md#2-how-task-processing-works)
  in the History Task Processing playbook explains that processing.
- **The DLQ:** after enough failed attempts, each failing task is moved to the history task DLQ
  (dead-letter queue), where it waits until an operator acts on it. The DLQ is not a separate
  system: it is the `queue_messages` table in the Temporal cluster's primary persistence store, the
  same with Cassandra persistence and SQL persistence (PostgreSQL, MySQL). Inside that one table, the
  DLQ keeps a separate queue for each kind of task. In this document, the **visibility DLQ** means
  the queue that holds visibility tasks, and the **archival DLQ** the queue that holds archival tasks.
  [Section 2](#2-how-visibility-and-archival-tasks-reach-the-dlq) covers when a failing task is moved,
  and [How the history task DLQ works](./detecting-recovering-archival-outage.md#1-how-the-history-task-dlq-works)
  in the archival outage playbook explains the DLQ itself.
- **Namespace retention:** the workflow's retention date can arrive while its tasks are still in
  the DLQ. With visibility tasks in the DLQ, the workflow is deleted on that date as usual. With
  archival tasks in the DLQ, it is not deleted until archival succeeds
  ([section 4](#4-archival-backend-failing-visibility-working) covers the time limit on that, 90
  days after the retention date by default).

**Merging or purging the DLQ:** a task in the DLQ is not lost. It has no expiry time and stays there
until an operator removes it. Temporal provides the `tdbg` command-line tool for this, and two of its
commands act on the DLQ. `tdbg dlq merge` **merges** the DLQ (also called redriving it): the tasks
are moved out of the DLQ and written back into the history service's task tables, and the history
service then processes them again like any other task. `tdbg dlq purge` **purges** it: the tasks are
deleted without being processed. Which of the two to run, and when, is the operator's decision: only
the operator knows when the failing store is fixed and whether the tasks are still wanted, so the
server keeps the tasks safe until then.

**Visibility and archival failure scenarios covered:**

- [Visibility store failing, archival working](#3-visibility-store-failing-archival-working)
- [Archival backend failing, visibility working](#4-archival-backend-failing-visibility-working)
- [Visibility store and archival backend both failing](#5-visibility-store-and-archival-backend-both-failing)

For each of these three scenarios, the document says:

- when the failing tasks go to the DLQ, and when they do not;
- what happens to the workflow, its visibility record and its archive while the tasks wait, in the
  DLQ or, when they do not go to the DLQ, in their normal queue;
- what the server does if the operator merges the DLQ (or the failing store comes back, when there
  is no DLQ) **before** namespace retention;
- what the server does if the operator merges the DLQ (or the failing store comes back) **after**
  namespace retention, and whether there is a point after which merging the DLQ can no longer help.

For each of the three scenarios, the document also shows how to recover without losing anything:
every closed workflow's event history still reaches the archival backend, as long as the archival
DLQ is merged before the history scanner's deadline
([section 4.2](#42-the-history-scanner-the-deadline-for-merging-the-archival-dlq)). Every workflow's
visibility record matches the workflow itself: a closed workflow shows as closed, and a deleted
workflow no longer appears in `temporal workflow list` or the UI instead of showing as **Running**.
If the history scanner deleted a workflow while Elasticsearch was still failing, that takes one
extra step ([section 7](#7-reading-merging-and-purging-the-dlq-with-tdbg)).

[Section 1](#1-short-answers-for-the-visibility-and-archival-failure-scenarios) gives a short answer
for each of the three scenarios. Sections [3](#3-visibility-store-failing-archival-working),
[4](#4-archival-backend-failing-visibility-working) and
[5](#5-visibility-store-and-archival-backend-both-failing) cover one scenario each in detail,
[section 6](#6-close-workflow-cleanup-turned-on) covers what changes with close workflow cleanup
turned on, and [section 7](#7-reading-merging-and-purging-the-dlq-with-tdbg) shows the `tdbg`
commands that read, merge and purge the DLQ.

## Audience and scope

**Audience:** Operators of self-hosted Temporal clusters.

**Related playbooks:**

- [Archival Backend Outage — Detection & Recovery](./detecting-recovering-archival-outage.md):
  detecting an archival backend outage, alerting on it, pausing archival while the outage lasts, and how the
  history task DLQ works.
- [History Task Processing — Tuning and Troubleshooting](./history-task-processing-tuning.md):
  how the history service processes its tasks, including the visibility and archival tasks this
  document follows.

**Applies to:**

- Clusters with archival enabled and Elasticsearch as the visibility store.
- Cassandra persistence and SQL persistence (PostgreSQL, MySQL).
- Server versions v1.29.0 through v1.32.1.

## Contents

1. [Short answers for the visibility and archival failure scenarios](#1-short-answers-for-the-visibility-and-archival-failure-scenarios)
2. [How visibility and archival tasks reach the DLQ](#2-how-visibility-and-archival-tasks-reach-the-dlq)
3. [Visibility store failing, archival working](#3-visibility-store-failing-archival-working)
   - 3.1 [Elasticsearch unreachable: visibility tasks wait, no DLQ](#31-elasticsearch-unreachable-visibility-tasks-wait-no-dlq)
   - 3.2 [Elasticsearch rejecting writes: visibility tasks go to the DLQ](#32-elasticsearch-rejecting-writes-visibility-tasks-go-to-the-dlq)
4. [Archival backend failing, visibility working](#4-archival-backend-failing-visibility-working)
   - 4.1 [Archival tasks go to the DLQ and namespace retention waits](#41-archival-tasks-go-to-the-dlq-and-namespace-retention-waits)
   - 4.2 [The history scanner: the deadline for merging the archival DLQ](#42-the-history-scanner-the-deadline-for-merging-the-archival-dlq)
   - 4.3 [Merging the archival DLQ before or after the retention date](#43-merging-the-archival-dlq-before-or-after-the-retention-date)
   - 4.4 [How a failing archival backend shows on the dashboards](#44-how-a-failing-archival-backend-shows-on-the-dashboards)
5. [Visibility store and archival backend both failing](#5-visibility-store-and-archival-backend-both-failing)
6. [Close workflow cleanup turned on](#6-close-workflow-cleanup-turned-on)
7. [Reading, merging and purging the DLQ with tdbg](#7-reading-merging-and-purging-the-dlq-with-tdbg)
8. [Task types named in the sections above](#8-task-types-named-in-the-sections-above)

---

## 1. Short answers for the visibility and archival failure scenarios

The short answers, for a cluster with visibility (Elasticsearch) and archival both enabled, at the
server's default settings. Every row is about a workflow that
**closes while the store is failing**:
it completes, fails, times out, is canceled, is terminated or continues as new, and so creates the
close tasks described in [Tasks created when a workflow closes](#tasks-created-when-a-workflow-closes).
Each row is one failure. The columns say:

- **Do the tasks go to the DLQ?** Whether the failing visibility or archival tasks are moved to the
  DLQ, and when.
- **While the tasks wait:** what the workflow, its visibility record and its archive look like after
  the workflow closes, while the store is still failing and its tasks are waiting, in the DLQ or in
  their normal queue.
- **DLQ merged before namespace retention** and **DLQ merged after namespace retention:** what
  happens when the operator merges the DLQ with `tdbg dlq merge` after fixing the store, before or
  after the workflow's retention date. For the row with no DLQ, these say what happens when
  Elasticsearch is back before or after that date.

| What failed | Do the tasks go to the DLQ? | While the tasks wait | DLQ merged before namespace retention | DLQ merged after namespace retention |
|---|---|---|---|---|
| **Visibility:** Elasticsearch unreachable | **No.** The visibility tasks wait with no time limit and finish by themselves when Elasticsearch is back | The visibility record shows **Running**. Visibility updates stop for every namespace. Archival and namespace retention run normally | No DLQ action needed, because the tasks never went to the DLQ. When Elasticsearch is back, the waiting tasks finish by themselves and the workflow shows as closed in visibility | No DLQ action needed, because the tasks never went to the DLQ. The workflow was already deleted on its retention date. When Elasticsearch is back, the waiting tasks remove its visibility record, which until then shows **Running** |
| **Visibility:** Elasticsearch up but rejecting writes | Yes, into the visibility DLQ, after `history.TaskDLQUnexpectedErrorAttempts` failed attempts (default **70**, about an hour): the close execution visibility tasks, and the delete execution visibility tasks if Elasticsearch is still failing at namespace retention | The visibility record shows **Running**. Archival and namespace retention run normally | The workflow shows as closed in visibility | The workflow was already deleted on its retention date. If Elasticsearch was fixed before that date, namespace retention removed its visibility record, and the merge has nothing left to do. If Elasticsearch was still rejecting writes on that date, the record shows **Running** until the merge removes it. There is no deadline for merging the visibility DLQ: tasks stay in the visibility DLQ until merged or purged, unlike the archival DLQ in the next row |
| **Archival:** archival backend failing | Yes, into the archival DLQ, after `history.TaskDLQUnexpectedErrorAttempts` failed attempts (default **70**, about an hour): the archival tasks | Namespace retention waits for archival, so the workflow stays in the primary persistence store past its retention date. Visibility shows it as closed | The workflow is archived, then deleted on its retention date | Up to 90 days (by default) after the retention date: the workflow is archived, then deleted right away. More than 90 days (by default) after: too late, see below the table |
| **Visibility and archival:** Elasticsearch rejecting writes and archival backend failing | Yes, after the same attempt limit: the close execution visibility tasks into the visibility DLQ, and the archival tasks into the archival DLQ | Namespace retention waits for archival, so the workflow stays in the primary persistence store. The visibility record shows **Running** | Merge both the visibility DLQ and the archival DLQ, in any order. The workflow shows as closed in visibility, is archived, then deleted on its retention date | Merge the visibility DLQ and the archival DLQ, in any order. Up to 90 days (by default) after the retention date: archived, then deleted. Later: too late, see below the table |

> **Recommendation:** once Elasticsearch or the archival backend is fixed, merge the visibility DLQ
> and the archival DLQ, whichever of the two holds tasks. Do not purge them. Purging the archival
> DLQ means those workflows are never archived, and purging the visibility DLQ leaves closed or
> already deleted workflows showing as **Running**. When Elasticsearch was unreachable rather than
> rejecting writes, there is nothing in the DLQ and nothing to do.
> [Section 7](#7-reading-merging-and-purging-the-dlq-with-tdbg) shows the `tdbg` commands for the merge.

> **Recommendation:** merge the archival DLQ no more than **90 days** (by default,
> `worker.executionDataDurationBuffer`) after the workflows' retention date. After that, the
> history scanner deletes those workflows without archiving them, and a merge can no longer archive
> them. The history scanner also removes their visibility records. If Elasticsearch is still
> rejecting writes at that moment, those records can stay **Running**, and step 7 of
> [section 7](#7-reading-merging-and-purging-the-dlq-with-tdbg) removes them.

The detailed sections start with
[section 2](#2-how-visibility-and-archival-tasks-reach-the-dlq), which explains how many
failed attempts it takes for a visibility or archival task to be moved to the DLQ, which errors
count, and where to see DLQ activity on the dashboard.

## 2. How visibility and archival tasks reach the DLQ

With `history.TaskDLQEnabled` on (default **true**), a history task goes to the DLQ after
`history.TaskDLQUnexpectedErrorAttempts` failed attempts (default **70**, about an hour). Only
unexpected errors count. An unreachable archival backend or an Elasticsearch write rejection counts;
a "resource exhausted" error does not, and a task failing with "resource exhausted" is retried with no
limit.

The attempt count is kept in memory by the history service. If the shard moves to another history
instance or the instance restarts, the count starts again, so reaching the DLQ can take longer than an
hour.

The visibility DLQ and the archival DLQ are two separate queues in the same `queue_messages` table.
The `tdbg dlq` commands (`read`, `merge`, `purge`) pick one with the `--dlq-type` flag, given after
the command, for example `tdbg dlq merge --dlq-type 4`:

| DLQ | `--dlq-type` value |
|---|---|
| Visibility | `4` |
| Archival | `5` |

**Where to see DLQ activity**, on the [Temporal Server dashboard](../observability/dashboards/server/temporal-server-readme.md):

- **Visibility Tasks Dead-Lettered by Task Type**, in row
  [16. Visibility](../observability/dashboards/server/temporal-server-readme.md#16-visibility): visibility tasks moving into the DLQ.
- **Signal 2 — History Task DLQ Writes & Write Failures**, in row
  [21. Archival Health](../observability/dashboards/server/temporal-server-readme.md#21-archival-health): archival tasks moving into the DLQ, and any failed
  attempt to write a task to the DLQ.
- **Dead-Letter Queue Depth by Category**, in row
  [20. History Task DLQ / Terminal Failures](../observability/dashboards/server/temporal-server-readme.md#20-history-task-dlq--terminal-failures):
  how many tasks are sitting in each DLQ right now. This panel refreshes only every three hours.

Two alerts that come with these dashboards watch the same panels:
[Alert 83 — Visibility Tasks Dead-Lettered](../observability/alerts/server/alerts-index.md#alert-83--visibility-tasks-dead-lettered)
fires when visibility tasks reach the visibility DLQ, and
[Alert 82 — History Task DLQ Write Failures](../observability/alerts/server/alerts-index.md#alert-82--history-task-dlq-write-failures)
fires when writing a task to the DLQ fails.

[Section 3](#3-visibility-store-failing-archival-working) starts with the first of the three
visibility and archival failure scenarios listed in
[Failures, the DLQ and the scenarios covered](#failures-the-dlq-and-the-scenarios-covered): the
visibility store failing while archival keeps working.

## 3. Visibility store failing, archival working

When Elasticsearch fails, archival and namespace retention still run on schedule: the workflow is
archived, then deleted from the primary persistence store on its retention date. Archival does not
need Elasticsearch, because **the archival task builds the archived visibility record from the
workflow's own state in the primary persistence store, not from Elasticsearch**. That state is the
workflow's execution record (also called its mutable state), not its event history. The execution
record already has the workflow's closed status and the workflow close time, so the archived
visibility record is correct even while Elasticsearch still shows the workflow as **Running**,
because the failing Elasticsearch could not record that the workflow closed.

What users notice is the visibility record. `temporal workflow list` and the UI's workflow list read
from Elasticsearch, so while Elasticsearch is failing they keep showing the closed workflow as
**Running**.

An Elasticsearch failure comes in two kinds that behave in opposite ways. When Elasticsearch cannot
be reached, the visibility tasks wait and never go to the DLQ
([section 3.1](#31-elasticsearch-unreachable-visibility-tasks-wait-no-dlq)). When Elasticsearch is up
but rejects writes, the visibility tasks go to the DLQ after the attempt limit
([section 3.2](#32-elasticsearch-rejecting-writes-visibility-tasks-go-to-the-dlq)).

### 3.1 Elasticsearch unreachable: visibility tasks wait, no DLQ

When Elasticsearch cannot be reached, visibility tasks do **not** go to the DLQ. Each visibility
task waits for Elasticsearch to come back, with no time limit, so it never reaches the attempt limit.

The waiting tasks are **not lost**, however long the outage lasts. Each task stays stored in the
primary persistence store, in the tables listed in
[Tasks created when a workflow closes](#tasks-created-when-a-workflow-closes), and the server deletes
it only after it completes. If a history service instance restarts during the outage, the
tasks are read again from the database.

While the visibility tasks wait for Elasticsearch to be reachable again:

- `temporal workflow list` and the UI show the closed workflow as **Running**.
- Visibility updates stop for **every namespace**, not only for the workflows that closed during the
  outage, because the waiting tasks hold up all visibility task processing.
- If the workflow's retention date arrives while Elasticsearch is still unreachable, namespace
  retention runs on time and deletes the workflow from the primary persistence store. The task that
  would remove its visibility record (the delete execution visibility task) waits too. So from then
  on, `temporal workflow list` and the UI show as **Running** a workflow that no longer exists.

When Elasticsearch is back, the waiting tasks finish by themselves, and no DLQ merge is needed:

- **Back before the workflow's retention date:** the workflow shows as closed in
  `temporal workflow list` and the UI, as normal.
- **Back after the workflow's retention date:** the workflow was already deleted, and its
  visibility record is now removed, so it no longer shows in the list or the UI.

**How an unreachable Elasticsearch shows on the dashboards.** It does not show as errors. On the
[Temporal Server dashboard](../observability/dashboards/server/temporal-server-readme.md), in row [16. Visibility](../observability/dashboards/server/temporal-server-readme.md#16-visibility):

- **Visibility Write Error Rate per Store:** a short burst of errors as Elasticsearch goes down, then
  nothing.
- **Visibility Task Processing by Operation:** drops to zero and stays there until Elasticsearch is
  back. Zero visibility activity while workflows keep closing is the sign of the outage.
- **Visibility Tasks Dead-Lettered by Task Type:** stays at zero, because nothing goes to the DLQ.

When Elasticsearch is back, **Visibility Task Processing by Operation** rises sharply for a short
time as the waiting tasks run, then returns to normal.

[Section 3.2](#32-elasticsearch-rejecting-writes-visibility-tasks-go-to-the-dlq)
covers Elasticsearch rejecting writes, the Elasticsearch failure that does fill the visibility DLQ.

### 3.2 Elasticsearch rejecting writes: visibility tasks go to the DLQ

When Elasticsearch is up but rejects the writes to visibility records (for example, because of a
write block on the index, or because Elasticsearch is overloaded or returns a server error for that
write), each attempt of a visibility task fails quickly, and the server retries it. If the errors
continue, the task goes to the visibility DLQ after `history.TaskDLQUnexpectedErrorAttempts` failed
attempts (default **70**, about an hour). As long as Elasticsearch keeps rejecting writes, every
workflow that closes adds its close execution visibility task to the visibility DLQ about an hour
later, at the default attempt limit.

While the close execution visibility task is in the DLQ:

- `temporal workflow list` and the UI show the closed workflow as **Running**, because the task that
  would mark it closed never ran.
- Archival and namespace retention run normally: the workflow is archived, then deleted on its
  retention date.

**How Elasticsearch rejecting writes shows on the dashboards.** On the
[Temporal Server dashboard](../observability/dashboards/server/temporal-server-readme.md):

- **Visibility Write Error Rate per Store**, in row
  [16. Visibility](../observability/dashboards/server/temporal-server-readme.md#16-visibility): rises
  as soon as Elasticsearch starts rejecting writes, because every failed attempt counts. This is the
  earliest sign, about an hour before the first visibility tasks reach the DLQ at the default attempt
  limit.
- **Visibility Tasks Dead-Lettered by Task Type**, in row
  [16. Visibility](../observability/dashboards/server/temporal-server-readme.md#16-visibility): rises
  as visibility tasks move into the DLQ, by task type (close execution, and delete execution once
  retention starts deleting these workflows).
- **Dead-Letter Queue Depth by Category**, in row
  [20. History Task DLQ / Terminal Failures](../observability/dashboards/server/temporal-server-readme.md#20-history-task-dlq--terminal-failures):
  how many tasks are in the visibility DLQ. This panel refreshes only every three hours, so it lags.

> **Recommendation:** do not wait to notice Elasticsearch rejecting writes on the dashboard: alert on
> it. The alerts that come with these dashboards include
> [Alert 59a — Visibility Store Write Errors](../observability/alerts/server/alerts-index.md#alert-59a--visibility-store-write-errors-warning),
> which fires on **Visibility Write Error Rate per Store** as soon as writes are rejected, and
> [Alert 83 — Visibility Tasks Dead-Lettered](../observability/alerts/server/alerts-index.md#alert-83--visibility-tasks-dead-lettered),
> which fires when visibility tasks reach the visibility DLQ. Alert 59a leaves about an hour, at the
> default attempt limit, to fix Elasticsearch before any visibility task goes to the DLQ.

What happens next depends on two things, each compared with the workflow's **retention date** (when
namespace retention deletes it): when Elasticsearch is fixed, and when the visibility DLQ is merged.
There are three cases.

**Case 1: Elasticsearch fixed and the DLQ merged before the retention date.** The merge moves the
close execution visibility tasks out of the DLQ and back into the history service's task tables, and
the history service processes them again. The workflow still exists, so the tasks write its closed
status to Elasticsearch, and `temporal workflow list` and the UI show it as closed. On the retention
date, namespace retention deletes the workflow and its visibility record as usual.

**Case 2: Elasticsearch fixed before the retention date, the DLQ merged after it.**

- **Until the retention date:** the workflow shows as **Running**, because its close execution
  visibility task is still in the DLQ.
- **On the retention date:** namespace retention deletes the workflow from the primary persistence
  store. It does not remove the visibility record itself. It creates a delete execution visibility
  task, and that task removes the record. Elasticsearch accepts writes again by then, so the record is
  removed and the workflow no longer shows in the list or the UI.
- **When the DLQ is merged later:** there is nothing left to do. The close execution visibility tasks
  are dropped, because their workflow no longer exists.

**Case 3: Elasticsearch still rejecting writes when the retention date arrives.**

- **On the retention date:** namespace retention deletes the workflow from the primary persistence
  store. The delete execution visibility task fails like the close task did, and after the attempt
  limit it goes to the visibility DLQ too.
- **After the retention date, until the DLQ is merged:** the workflow no longer exists, but
  `temporal workflow list` and the UI's workflow list still show it as **Running**. Opening it
  fails: `temporal workflow describe` returns "workflow not found", and clicking it in the UI's list
  opens a **404 Not Found** page instead of the workflow. Fixing Elasticsearch does not change this,
  because the server never processes tasks in the DLQ again on its own. Only a merge does.
- **When the DLQ is merged:** the merge moves both kinds of task back into the history service's
  task tables, and the history service processes them again. The close execution visibility tasks
  are dropped, because their workflow no longer exists. The delete execution visibility tasks remove
  the records.

> **Recommendation:** merge the visibility DLQ as soon as Elasticsearch accepts writes again, and
> before the workflows' retention date if you can (case 1). The workflows then show as closed for the
> rest of their retention period. If the retention date has already passed, still merge: in case 3,
> the merge is the only thing that removes the records still showing deleted workflows as
> **Running**. Do not purge the visibility DLQ, and do not merge it while Elasticsearch is still
> rejecting writes, or the tasks fail again and go back to the DLQ.

> **Recommendation:** after merging the visibility DLQ past the workflows' retention date (cases 2
> and 3), check the result on the dashboard, not in the logs. That merge drops the close execution
> visibility tasks, because their workflows were already deleted, and the server does not log
> dropped tasks. Dropping them is safe: a dropped task writes nothing to Elasticsearch, so it cannot
> create a **Closed** record for a workflow that no longer exists. Step 6 of
> [section 7](#7-reading-merging-and-purging-the-dlq-with-tdbg) names the dashboard panel to check
> (server v1.30.0 and later).

[Section 4](#4-archival-backend-failing-visibility-working) covers the second scenario: the
archival backend failing while Elasticsearch keeps working.

## 4. Archival backend failing, visibility working

When the archival backend fails, visibility keeps working: the close execution visibility task
writes the workflow's closed status to Elasticsearch as usual, so `temporal workflow list` and the UI
show the workflow as closed. What changes is when these workflows are deleted. The namespace
retention period itself does not change, but with archival enabled a workflow is deleted **only after
archival succeeds**, so a workflow that closes while archival is failing is not deleted on its
retention date. Also, unlike an unreachable Elasticsearch
([section 3.1](#31-elasticsearch-unreachable-visibility-tasks-wait-no-dlq)), an archival backend that
is down makes each archival attempt fail, so the archival tasks do reach the archival DLQ.

The rest of this section follows those archival tasks:

- [Section 4.1](#41-archival-tasks-go-to-the-dlq-and-namespace-retention-waits):
  what happens while the archival tasks are in the archival DLQ.
- [Section 4.2](#42-the-history-scanner-the-deadline-for-merging-the-archival-dlq):
  how long they can stay there before the history scanner deletes their workflows without archiving
  them.
- [Section 4.3](#43-merging-the-archival-dlq-before-or-after-the-retention-date):
  what a merge of the archival DLQ does, depending on when it is done.
- [Section 4.4](#44-how-a-failing-archival-backend-shows-on-the-dashboards):
  how each of these stages shows on the dashboards.

### 4.1 Archival tasks go to the DLQ and namespace retention waits

When the archival backend is down, each attempt of an archival task fails, and the server retries
it. If the errors continue, the task goes to the archival DLQ after
`history.TaskDLQUnexpectedErrorAttempts` failed attempts (default **70**, about an hour). As long as
the archival backend keeps failing, every workflow that closes adds its archival task to the archival
DLQ about an hour later, at the default attempt limit.

While the archival task is in the DLQ:

- `temporal workflow list` and the UI show the workflow as closed, as normal.
- Namespace retention does not delete the workflow. The retention delete is scheduled only after
  archival succeeds, so it is never scheduled, and the workflow and its event history stay in the
  primary persistence store past the retention date.
- The workflow stays there until the archival DLQ is merged, or until the history scanner deletes it.

[Section 4.2](#42-the-history-scanner-the-deadline-for-merging-the-archival-dlq)
covers the history scanner, and how long the archival tasks can wait in the archival DLQ before the
history scanner deletes their workflows.

### 4.2 The history scanner: the deadline for merging the archival DLQ

While the archival tasks wait in the archival DLQ, their workflows are not kept forever. The history
scanner, a background workflow in the worker service, deletes closed workflows older than their
namespace retention + `worker.executionDataDurationBuffer` (default **90 days**), counted from the
time the workflow closed. It deletes the workflow, its event history and its visibility record
**without archiving them**. So 90 days after the retention date (by default) is the deadline for
merging the archival DLQ: after the history scanner's delete, a merge can no longer archive the
workflow's event history.

The history scanner's delete does not remove the archival tasks from the archival DLQ: they stay
there until an operator merges or purges them.

The history scanner runs every 12 hours, a fixed schedule that no setting changes, so its delete
lands up to about 12 hours after the workflow reaches that age. It runs with both Cassandra
persistence and SQL persistence. The settings behind its delete, all at their defaults:

| Setting | Default | What the setting does |
|---|---|---|
| `worker.historyScannerEnabled` | `true` | Runs the history scanner (Cassandra and SQL persistence) |
| `worker.historyScannerVerifyRetention` | `true` | Lets the history scanner delete closed workflows past namespace retention + buffer |
| `worker.executionDataDurationBuffer` | `90d` | The buffer added to namespace retention |
| `worker.historyScannerDataMinAge` | `60d` | Skips event history younger than 60 days, which is shorter than retention + 90 days (by default), so it does not delay the delete |

[Section 4.3](#43-merging-the-archival-dlq-before-or-after-the-retention-date) covers what a
merge of the archival DLQ does before the retention date, after it, and after this deadline.

### 4.3 Merging the archival DLQ before or after the retention date

The merge moves the archival tasks out of the DLQ and back into the history service's task tables,
and the history service processes them again. What happens then depends on how long after the
workflow's **retention date** (when namespace retention would normally delete it) the merge runs.
There are three cases.

**Case 1: archival backend fixed and the DLQ merged before the retention date.** The history service
archives the workflow's full event history and visibility record, then schedules the retention
delete. On the retention date, namespace retention deletes the workflow as usual.

**Case 2: archival backend fixed and the DLQ merged after the retention date, but less than
90 days (by default) after it.** The history service archives the full event history and visibility
record. The retention date has already passed, so the retention delete runs right away and deletes the
workflow from the primary persistence store and from Elasticsearch. Nothing is lost: the workflow
shows in `temporal workflow list --archived`, and `temporal workflow show` reads its event history
from the archive.

**Case 3: archival backend fixed and the DLQ merged more than 90 days (by default) after the
retention date.** By then the history scanner
([section 4.2](#42-the-history-scanner-the-deadline-for-merging-the-archival-dlq)) has deleted the
workflow, its event history and its visibility record **without archiving them**. The event history
is lost, and the merge cannot archive it:

- **Merged within about an hour of the history scanner's delete:** right before it deletes a
  workflow, the history scanner loads the workflow into the history service's in-memory cache. The
  workflow stays in that cache after the delete, for up to `history.cacheTTL` (default **1 hour**),
  unless it is removed sooner, for example when the history service restarts. While it is still
  there, the merge reads the workflow from the cache and archives its visibility record, but its event
  history is already deleted, so none is archived. The workflow then shows in
  `temporal workflow list --archived`, but `temporal workflow show` returns "requested workflow
  history does not exist".
- **Merged later, once the workflow is no longer in the cache:** the archival tasks are dropped, and
  nothing is archived.

> **Recommendation:** merge the archival DLQ as soon as the archival backend accepts writes again,
> and before the workflows' retention date if you can (case 1). The workflows are then archived and
> deleted on time, instead of staying in the primary persistence store past their retention date. If
> the retention date has already passed, merge no more than **90 days** (by default) after it
> (case 2): up to then, the merge still archives everything. Do not purge the archival DLQ, because
> those workflows are then never archived, and do not merge it while the archival backend is still
> failing, or the tasks fail again and go back to the DLQ.
> [Section 7](#7-reading-merging-and-purging-the-dlq-with-tdbg) shows the `tdbg` commands for the merge.

[Section 4.4](#44-how-a-failing-archival-backend-shows-on-the-dashboards)
shows how each stage of this section, from the first failed archival attempt to the merge, shows on
the dashboards.

### 4.4 How a failing archival backend shows on the dashboards

Each stage of this section shows on the
[Temporal Server dashboard](../observability/dashboards/server/temporal-server-readme.md), in the
order it happens:

1. **Archival attempts start failing.** In row
   [21. Archival Health](../observability/dashboards/server/temporal-server-readme.md#21-archival-health),
   **Signal 1 — Archival Attempt Error Rate** rises right away. This is the earliest sign, about an
   hour before the first archival tasks reach the DLQ at the default attempt limit. In the same row,
   **Archival Errors by Type (non-retryable vs transient)** shows whether the failures are lasting
   (for example, the backend's address cannot be resolved) or passing (for example, timeouts).
2. **Archival tasks move into the archival DLQ**
   ([section 4.1](#41-archival-tasks-go-to-the-dlq-and-namespace-retention-waits)). In row 21,
   **Signal 2 — History Task DLQ Writes & Write Failures** rises.
3. **Tasks build up in the archival DLQ.** In row
   [20. History Task DLQ / Terminal Failures](../observability/dashboards/server/temporal-server-readme.md#20-history-task-dlq--terminal-failures),
   **Dead-Letter Queue Depth by Category** shows how many tasks are in the archival DLQ. This panel
   refreshes only every three hours, so it lags.
4. **The archival DLQ is merged**
   ([section 4.3](#43-merging-the-archival-dlq-before-or-after-the-retention-date)). In row 21,
   **Archival Attempts by Status (ok / err / rate_limit_exceeded)** shows `ok` attempts as the merged
   tasks are archived, and **Dead-Letter Queue Depth by Category** drops at its next refresh.

> **Recommendation:** do not wait to notice a failing archival backend on the dashboard: alert on
> **Signal 1 — Archival Attempt Error Rate**. The alerts that come with these dashboards include it
> as [Alert 81 — Archival Backend Failing](../observability/alerts/server/alerts-index.md#alert-81--archival-backend-failing),
> with a [runbook](../observability/alerts/server/runbooks/81-archival-backend-failing.md). It fires
> about an hour before the first archival tasks reach the archival DLQ, at the default attempt limit,
> which leaves time to fix the archival backend before archival tasks start going to the DLQ.

[Section 5](#5-visibility-store-and-archival-backend-both-failing) covers the third scenario: the
visibility store and the archival backend failing at the same time.

## 5. Visibility store and archival backend both failing

When Elasticsearch rejects writes and the archival backend is down at the same time, the effects of
[section 3.2](#32-elasticsearch-rejecting-writes-visibility-tasks-go-to-the-dlq) and
[section 4](#4-archival-backend-failing-visibility-working) add up. The close execution visibility
tasks go to the visibility DLQ, so the closed workflow shows as **Running**, and the archival tasks
go to the archival DLQ, so namespace retention waits and the workflow is not deleted on its
retention date. If Elasticsearch is unreachable rather than rejecting writes, its visibility tasks
wait instead of going to the DLQ
([section 3.1](#31-elasticsearch-unreachable-visibility-tasks-wait-no-dlq)), and only the archival
half of this section applies.

While the tasks are in the visibility DLQ and the archival DLQ:

- `temporal workflow list` and the UI show the closed workflow as **Running**, because its close
  execution visibility task is in the visibility DLQ.
- Namespace retention does not delete the workflow, because its archival task is in the archival
  DLQ ([section 4.1](#41-archival-tasks-go-to-the-dlq-and-namespace-retention-waits)). The workflow
  and its event history stay in the primary persistence store past the retention date.
- The workflow stays there until the archival DLQ is merged, or until the history scanner deletes
  it, 90 days after its retention date by default
  ([section 4.2](#42-the-history-scanner-the-deadline-for-merging-the-archival-dlq)).
- If Elasticsearch is still rejecting writes when the history scanner deletes the workflow, the
  workflow's visibility record is not removed. It keeps showing **Running** after Elasticsearch
  recovers, and merging the two DLQs later does not remove it, unless the merges run within about an
  hour of the delete, while the history service still holds the workflow in its in-memory cache (as
  in case 3 of [section 4.3](#43-merging-the-archival-dlq-before-or-after-the-retention-date)). Step 7 of
  [section 7](#7-reading-merging-and-purging-the-dlq-with-tdbg) shows how to remove these records.

**How Elasticsearch and the archival backend failing together show on the dashboards.** The panels
and alerts of both scenarios show at once:

- **Elasticsearch rejecting writes:** the panels in
  [section 3.2](#32-elasticsearch-rejecting-writes-visibility-tasks-go-to-the-dlq), and Alert 59a and
  Alert 83.
- **Archival backend failing:** the panels in
  [section 4.4](#44-how-a-failing-archival-backend-shows-on-the-dashboards), and Alert 81.

**Merging the visibility DLQ and the archival DLQ.** Once Elasticsearch and the archival backend
both accept writes again, merge both DLQs, and merge the archival DLQ no more than
90 days (by default) after the workflows' retention date
([section 4.2](#42-the-history-scanner-the-deadline-for-merging-the-archival-dlq)). Within that limit,
the order of the two merges does not matter:

- **Archival DLQ first:** the history service archives the full event history. The retention
  delete then removes the workflow and its visibility record: right away if the retention date has
  already passed, otherwise on the retention date. The later visibility DLQ merge
  drops the close execution visibility tasks, because their workflow no longer exists, and does not
  bring the record back.
- **Visibility DLQ first:** the history service processes the close execution visibility tasks
  again, so the workflow shows as closed. The later archival DLQ merge archives the full event
  history and then schedules the retention delete, which runs right away if the retention date has
  already passed.

Either merge order, archival DLQ first or visibility DLQ first, ends with the event history
archived and the workflow and its visibility record deleted.

> **Recommendation:** once Elasticsearch and the archival backend both accept writes again, merge
> the visibility DLQ and the archival DLQ, in either order. Merge the archival DLQ no more than
> **90 days** (by default) after the workflows' retention date, as in
> [section 4.3](#43-merging-the-archival-dlq-before-or-after-the-retention-date). Do not purge
> either DLQ, and do not merge either one while its store is still failing, or the tasks fail again
> and go back to the DLQ.

[Section 6](#6-close-workflow-cleanup-turned-on) covers the one setting that changes these
answers, `history.visibilityProcessorEnableCloseWorkflowCleanup`, for clusters where it is turned on.

## 6. Close workflow cleanup turned on

Sections [3](#3-visibility-store-failing-archival-working) to
[5](#5-visibility-store-and-archival-backend-both-failing) assume that
`history.visibilityProcessorEnableCloseWorkflowCleanup` is off, its default. This section covers
what changes when it is on. The recommendation in the introduction is to keep it off.

**What the setting does.** The setting is set per namespace. When it is on, once the close execution
visibility task has written the workflow's closed status to Elasticsearch, the server removes the
workflow's memo and search attributes from its execution record in the primary persistence store, to
save space there. From then on they are kept only in Elasticsearch.

**Why the setting makes archival depend on Elasticsearch.** The archived visibility record includes
the memo and search attributes. Once they are removed from the execution record, the archival task
reads them back from Elasticsearch, and if Elasticsearch cannot answer, the archival task fails.

This affects the workflows whose close was already recorded in Elasticsearch when Elasticsearch
failed. For a workflow that closes while Elasticsearch is already failing, the close execution
visibility task does not succeed, nothing is removed, and archival works from the execution record as
in [section 3](#3-visibility-store-failing-archival-working).

While Elasticsearch is failing, for those workflows:

- Each archival attempt fails, and after `history.TaskDLQUnexpectedErrorAttempts` failed attempts
  (default **70**, about an hour) the archival task goes to the archival DLQ. Unlike the visibility
  tasks in [section 3.1](#31-elasticsearch-unreachable-visibility-tasks-wait-no-dlq), the archival
  tasks reach the DLQ even when Elasticsearch is unreachable.
- Namespace retention waits for archival, as in
  [section 4.1](#41-archival-tasks-go-to-the-dlq-and-namespace-retention-waits): the workflow stays in
  the primary persistence store past its retention date.
- `temporal workflow describe` for these closed workflows fails with "Failed to fetch memo and search
  attributes", because it reads them back from Elasticsearch the same way.

**How the archival failure shows on the dashboards.** It does not show where a failing archival
backend does. The archival task fails before it reaches the archival backend, so **Signal 1 —
Archival Attempt Error Rate** stays flat and Alert 81 does not fire. On the
[Temporal Server dashboard](../observability/dashboards/server/temporal-server-readme.md), the
failure shows in:

- **Visibility & Archival Tasks — Attempts, Completions, Errors by Task Type**, in row
  [20. History Task DLQ / Terminal Failures](../observability/dashboards/server/temporal-server-readme.md#20-history-task-dlq--terminal-failures):
  errors on `ArchivalTaskArchiveExecution`.
- **Signal 2 — History Task DLQ Writes & Write Failures**, in row
  [21. Archival Health](../observability/dashboards/server/temporal-server-readme.md#21-archival-health):
  archival tasks moving into the archival DLQ.

The Elasticsearch failure itself shows as in
[section 3](#3-visibility-store-failing-archival-working), and how it shows depends on the kind of
failure:

- **Elasticsearch rejecting writes:** Alert 59a and Alert 83 fire, as in
  [section 3.2](#32-elasticsearch-rejecting-writes-visibility-tasks-go-to-the-dlq). They are the
  early warning for archival too.
- **Elasticsearch unreachable:** no alert covers it. **Visibility Write Error Rate per Store** shows
  only a short burst of errors as Elasticsearch goes down, Alert 83 does not fire because no
  visibility task goes to the DLQ, and Alert 81 does not fire, as shown above. The sign is **Visibility
  Task Processing by Operation** dropping to zero
  ([section 3.1](#31-elasticsearch-unreachable-visibility-tasks-wait-no-dlq)). About an hour later,
  **Signal 2** shows the archival tasks moving into the archival DLQ.

**Recovering.** Fix Elasticsearch first, then merge the archival DLQ, as in
[section 4.3](#43-merging-the-archival-dlq-before-or-after-the-retention-date). The archival tasks
read the memo and search attributes from Elasticsearch again, archive the workflow with them, and the
retention delete follows. Merging while Elasticsearch is still failing sends the tasks back to the
archival DLQ.

**Removing the dependency.** A second setting,
`history.disableFetchRelocatableAttributesFromVisibility` (default **false**), stops archival from
reading the memo and search attributes from Elasticsearch. With it set to true, archival and namespace
retention run on schedule while Elasticsearch is failing, but the archived visibility records of
these workflows have no memo and no search attributes.

> **Recommendation:** keep `history.visibilityProcessorEnableCloseWorkflowCleanup` off, as the
> introduction recommends. If it is on in your cluster, the archival alert does not fire for this
> failure. Rely on the Elasticsearch alerts from
> [section 3.2](#32-elasticsearch-rejecting-writes-visibility-tasks-go-to-the-dlq) when Elasticsearch
> rejects writes, and watch **Visibility Task Processing by Operation** for an unreachable
> Elasticsearch, which no alert covers. Fix Elasticsearch before merging the archival DLQ.

[Section 7](#7-reading-merging-and-purging-the-dlq-with-tdbg) shows the `tdbg` commands for reading,
merging and purging the DLQ that sections [3](#3-visibility-store-failing-archival-working) to
[6](#6-close-workflow-cleanup-turned-on) call for.

## 7. Reading, merging and purging the DLQ with tdbg

Sections [3](#3-visibility-store-failing-archival-working) to
[6](#6-close-workflow-cleanup-turned-on) say when to merge or purge each DLQ. This section shows how,
with `tdbg`, the admin command-line tool that comes with the Temporal server. Point it at the
frontend service with `--address`, or set `TEMPORAL_CLI_ADDRESS`. Use `--dlq-type 4` for the
visibility DLQ and `--dlq-type 5` for the archival DLQ, after the command name, as in the examples.

**1. Find the DLQs and the id of their newest task.**

```bash
tdbg dlq list
```

```
         QUEUENAME         | MESSAGECOUNT | LASTMESSAGEID
  4_active_active_B8jqXXpO |            3 |         11690
  5_active_active_B8jqXXpO |            3 |            43
  2_active_active_B8jqXXpO |            0 |            89
```

Each line is one DLQ. **MESSAGECOUNT** is how many tasks it holds now, and **LASTMESSAGEID** is the
id of its newest task, to pass as `--last-message-id` in the next steps. An empty DLQ still shows the
id of its last task.

The name starts with the `--dlq-type` ([section 8](#8-task-types-named-in-the-sections-above) lists
them), followed by the cluster name twice and a short code. It is not a namespace name: one DLQ holds
tasks from all namespaces.

**2. See which workflows the tasks belong to.**

```bash
tdbg dlq read --dlq-type 4 --last-message-id 11690 --output-filename visibility-dlq.json
```

Each task shows its workflow and its task type:

```json
{
  "message_id": 11690,
  "shard_id": 1652,
  "payload": {
    "namespace_id": "3e6e6aa4-460a-4be1-9735-1c46035e00c2",
    "workflow_id": "dlqret-cc-1",
    "run_id": "01a1b22b-d08c-7988-a994-03d89417c460",
    "task_type": "TASK_TYPE_VISIBILITY_CLOSE_EXECUTION",
    "task_id": "80740361",
    "visibility_time": "2026-11-06T18:43:10.981255345Z"
  }
}
```

`tdbg dlq read` shows at most 100 tasks; add `--max-message-count` to see more.
[Section 8](#8-task-types-named-in-the-sections-above) explains the task types.

**3. Merge the DLQ.**

```bash
tdbg dlq merge --dlq-type 4 --last-message-id 11690
```

The history service processes every task up to that id again, from all namespaces. Always give
`--last-message-id`. Without it, `tdbg` asks you to confirm, and when it runs without a terminal, for
example in a script, it crashes instead of asking unless `--yes` is also given.

The merge runs in the background and prints a job token. To check that it finished:

```bash
tdbg dlq job describe --job-token <job token>
```

```
  "operationType":  "DLQ_OPERATION_TYPE_MERGE",
  "operationState":  "DLQ_OPERATION_STATE_COMPLETED",
  "maxMessageId":  "11687",
  "lastProcessedMessageId":  "11687",
  "messagesProcessed":  "6"
```

`DLQ_OPERATION_STATE_COMPLETED` means the merge is done, and `messagesProcessed` is how many tasks it
merged.

**4. Purge the DLQ.**

```bash
tdbg dlq purge --dlq-type 4 --last-message-id 11690
```

A purge deletes the tasks without processing them. None of the scenarios in this document calls for
one.

**5. Merging the archival DLQ needs a newer `tdbg`.** `tdbg` up to v1.32.1 rejects `--dlq-type 5`
with `unknown dlq category 5`
([temporalio/temporal#11586](https://github.com/temporalio/temporal/issues/11586); fixed in
[temporalio/temporal#11587](https://github.com/temporalio/temporal/pull/11587), not yet in a
release). Until then, use the small Go program in the
[archival outage playbook](./detecting-recovering-archival-outage.md#redriving-the-archival-dlq),
which asks the server to do the merge directly. `tdbg dlq list` shows the archival DLQ with any
version.

**6. Check the result on the dashboard** (server v1.30.0 and later). When a merged task's workflow no
longer exists, the server drops the task without logging it. On the
[Temporal Server dashboard](../observability/dashboards/server/temporal-server-readme.md), in row
[20. History Task DLQ / Terminal Failures](../observability/dashboards/server/temporal-server-readme.md#20-history-task-dlq--terminal-failures),
**Visibility & Archival Tasks — Attempts, Completions, Errors by Task Type** shows how many tasks of
each type were attempted, completed and failed. Attempts with no completions and no errors are
dropped tasks. On older servers a dropped task is counted as completed.

**7. Remove the Running records left by the history scanner.** If the history scanner deleted a
workflow while Elasticsearch was rejecting writes
([section 5](#5-visibility-store-and-archival-backend-both-failing)), `temporal workflow list` keeps
showing the workflow as **Running**, while `temporal workflow describe` returns "workflow not found".
No merge fixes this. Once Elasticsearch accepts writes again, delete each such workflow again with
`tdbg`, using the workflow id and run id from the list:

```bash
tdbg --yes --namespace <namespace> execution delete --wid <workflow id> --rid <run id>
```

```
- Unable to load mutable state. Skipping workflow history deletion.. Error: workflow execution not found for workflow ID "dlqret-cc-1" and run ID "01a1b22b-d08c-7988-a994-03d89417c460"

Workflow execution deleted.
```

The warning is expected: the workflow is already gone, and the command still removes its **Running**
record.

These commands are used in
[section 3.2](#32-elasticsearch-rejecting-writes-visibility-tasks-go-to-the-dlq) (visibility DLQ),
[section 4.3](#43-merging-the-archival-dlq-before-or-after-the-retention-date) (archival DLQ),
[section 5](#5-visibility-store-and-archival-backend-both-failing) (both) and
[section 6](#6-close-workflow-cleanup-turned-on) (archival DLQ).

[Section 8](#8-task-types-named-in-the-sections-above) lists the task types named in this document,
with the names each goes by in `tdbg`, in the database and in metrics.

## 8. Task types named in the sections above

A task type goes by three names, depending on where it is read:

- **In `tdbg dlq read` output:** the uppercase name, such as `TASK_TYPE_VISIBILITY_CLOSE_EXECUTION`.
  It is the name of a value in the server's task type list.
- **In the database:** a number. Each task row stores the task as an encoded blob, and the task type
  inside that blob is the number from the table below. `tdbg` decodes the number and prints the
  uppercase name.
- **In metrics:** a shorter name, carried by both the `operation` and the `task_type` labels with the
  same value.

| Task | Name in `tdbg` | Number | Queue (`--dlq-type`) | Table with SQL persistence | Row `type` in `executions` with Cassandra persistence | Metric label value |
|---|---|---|---|---|---|---|
| Close transfer task | `TASK_TYPE_TRANSFER_CLOSE_EXECUTION` | 5 | transfer (`1`) | `transfer_tasks` | `2` (transfer task) | `TransferActiveTaskCloseExecution` |
| Close execution visibility task | `TASK_TYPE_VISIBILITY_CLOSE_EXECUTION` | 21 | visibility (`4`) | `visibility_tasks` | `6` (visibility task) | `VisibilityTaskCloseExecution` |
| Archival task | `TASK_TYPE_ARCHIVAL_ARCHIVE_EXECUTION` | 26 | archival (`5`) | `history_scheduled_tasks` | `5` (archival task) | `ArchivalTaskArchiveExecution` |
| Retention delete task | `TASK_TYPE_DELETE_HISTORY_EVENT` | 16 | timer (`2`) | `timer_tasks` | `3` (timer task) | `TimerActiveTaskDeleteHistoryEvent` |
| Delete execution visibility task | `TASK_TYPE_VISIBILITY_DELETE_EXECUTION` | 22 | visibility (`4`) | `visibility_tasks` | `6` (visibility task) | `VisibilityTaskDeleteExecution` |

With Cassandra persistence, every task in the table above is a row in the `executions` table, and the
row's `type` column says which kind of task it is. With either persistence, a task in the DLQ is
stored in the `queue_messages` table.

The metric label values for the close transfer task and the retention delete task above are the ones
used when the namespace is active on the cluster. On a cluster where the namespace is standby, the
same tasks are labelled `TransferStandbyTaskCloseExecution` and `TimerStandbyTaskDeleteHistoryEvent`.
The visibility and archival task labels are the same on both.
