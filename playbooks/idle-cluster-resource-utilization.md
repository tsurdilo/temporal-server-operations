# Idle Cluster Resource Utilization — Baseline Playbook

**Any system that keeps state and coordinates work has a resting load, and Temporal is no
different.** The baseline is the server keeping itself available and healthy: it runs underneath
whatever you put on the cluster, and **whatever your workflows consume is added on top of it** —
so the resources a cluster needs in total are the baseline plus the workload, never the workload
alone.

## Introduction

**A cluster's baseline grows with its size, not with your workflows.** How large the baseline is
depends on three things:

- **how many service instances you run**
- **how many history shards the cluster was created with**
- **how many namespaces are registered on it**

A large cluster with many shards and many namespaces has a larger baseline than a small one with
few of each.

Service instance count, shard count and namespace count differ from one cluster to the next, which
is why the numbers in this playbook are a starting point for measuring your own cluster rather
than a specification to copy.

**Knowing your own cluster's baseline is what makes provisioning more accurate, whatever size the
cluster is** — and the baseline is the part that usually goes unmeasured. This playbook measures
the baseline in database calls per second and in memory per service, and shows how much of each
comes from service instances, from shards, and from namespaces. CPU is covered too, but as a
pattern rather than a number to copy: which services it shows up in, and what makes it rise.

[What a Temporal cluster is made of](#what-a-temporal-cluster-is-made-of) describes what a service
instance, a history shard and a namespace each are.

### What a Temporal cluster is made of

A self-hosted Temporal cluster is not a single process. It is four service roles — **frontend**,
**history**, **matching** and **worker** — normally run as separate processes, usually several
instances of each, plus an optional fifth role, the **internal frontend**. Each role does a
different job and each role carries a share of the baseline. Wherever this playbook says *service
instance*, it means one running instance of one of those roles.

**History shards** are the fixed number of internal slots a cluster divides all of its execution
processing across. They are shared out among the history service instances, so each history
instance owns a portion of them and processes the workflow executions on those shards; add a
history instance and the shards redistribute across the larger set. The shard count itself is
chosen when the cluster is first created and cannot be changed afterwards.

**Namespaces** are Temporal's unit of isolation, each with its own task queues, retention and
access control. Every workflow execution runs within a namespace and is identified by its
namespace, its workflow ID and its run ID; within one namespace, only one workflow execution with
a given workflow ID can be running at a time. Unlike shards, namespaces are added and removed
while the cluster runs.

**Service instances, history shards and namespaces are now described.** [Background work a
Temporal server always runs](#background-work-a-temporal-server-always-runs) describes the kinds
of background work the Temporal server performs on its own.

### Background work a Temporal server always runs

1. **Saving each shard's progress.** Each shard belongs to one history instance at a time, and
   that instance writes to the database which of the shard's tasks it has finished and which are
   still outstanding — every five minutes by default ([section
   4.4](#44-settings-for-how-often-each-shard-reads-and-writes) covers the setting), even when
   nothing has changed. When a shard moves to another history instance, the new owner continues
   from that record, so no shard work is ever lost.
2. **Firing workflow timers and timeouts on time.** Each history shard has a **timer queue**: a
   processor in the history service that reads the shard's timer tasks from the database — not a
   task queue in the matching service. Every five minutes by default ([section
   4.4](#44-settings-for-how-often-each-shard-reads-and-writes) covers the setting), the timer
   queue reads ahead for timers coming due, so each timer fires on time: the timers a workflow
   sets itself — a fired timer wakes the workflow and hands its next step to your SDK workers —
   and the server's own activity and workflow timeouts, retries and retention cleanup. When
   archival is enabled in the cluster's static configuration, each shard also runs an **archival
   queue** that does the same for archival: every execution that closes in a namespace with
   archival enabled is archived a few minutes after it closes. Both queues run whether or not
   anything is scheduled.
3. **Refreshing namespace settings.** Every service instance — frontend, history, matching,
   worker, and internal frontend if you run one — keeps the settings of every namespace in memory,
   and re-reads the full list from the database every two seconds by default ([section
   3.3](#33-settings-for-service-instance-and-cluster-wide-database-calls-at-rest) covers the
   setting), so a namespace added or changed while the cluster runs reaches every service instance
   within seconds.
4. **Refreshing cluster metadata.** Every service instance re-reads the cluster's own record — its
   name, its address and the shard count it was created with — from the database once a minute by
   default ([section
   3.3](#33-settings-for-service-instance-and-cluster-wide-database-calls-at-rest) covers the
   setting).
5. **Refreshing the Nexus endpoint list.** One matching instance, for the whole cluster, checks
   the list of Nexus endpoints against the database every ten seconds by default ([section
   3.3](#33-settings-for-service-instance-and-cluster-wide-database-calls-at-rest) covers the
   setting).
6. **Tracking cluster membership.** A Temporal cluster scales by adding and removing service
   instances while it runs. Every service instance records itself in the database every ten to
   fifteen seconds (not configurable), so the cluster always knows its current members and hands
   the work of one that leaves to the ones that remain.
7. **Running a worker for every namespace.** The worker service runs one internal worker per
   namespace, which carries out schedules, batch operations and Worker Deployment and keeps polls
   open on that namespace's internal task queue. The worker service checks that every namespace
   has its worker every ten minutes (not configurable), and also immediately whenever a namespace
   changes or a worker service instance is added or removed.
8. **Confirming task queue ownership.** Each task queue is split into partitions, and each
   partition is owned by one matching instance at a time. Every minute by default, matching
   re-reads the record of each partition it has loaded to confirm it still owns it — every five
   minutes for the per-namespace worker's task queue — and every five minutes by default it
   re-reads each task queue's settings ([section
   3.3](#33-settings-for-service-instance-and-cluster-wide-database-calls-at-rest) and [section
   5.7](#57-settings-for-the-per-namespace-work) cover the settings). [Change Task Queue
   Partitions](./change-task-queue-partitions.md#what-are-task-queue-partitions) explains
   partitions.
9. **Cleaning up expired tasks and leftover workflow history.** Twice a day (not configurable) the
   worker service runs two scans. The **task queue scan**, only when the primary store is a SQL
   database (PostgreSQL, MySQL or SQLite), deletes expired tasks and task queues unused for two
   days. The **history scan** deletes leftover workflow history — [History Growth from Duplicate
   Workflow Starts](./history-growth-duplicate-workflow-starts.md) and [XDC Standby Database
   Growth on SQL](./xdc-standby-database-growth-sql.md) cover it. The history service's own task
   tables are cleaned continuously instead — [History Task Table
   Growth](./history-task-table-growth.md).

Three of these background operations grow as namespaces are added — refreshing namespace settings,
running a worker for every namespace and confirming task queue ownership ([points 3, 7 and
8](#background-work-a-temporal-server-always-runs)) — which raises a question worth settling
before any measurement begins.

### How many namespaces a cluster should hold

Nothing in the server caps how many namespaces a cluster can have; what limits it is resources.
**Every namespace adds to a cluster's baseline, whether or not a workflow ever runs in it**, and
that resting part is what this playbook measures. What a namespace's workflows add on top depends
on the workload you run in it, which this playbook does not cover.

So *"how many namespaces can this cluster hold?"* is a sizing question with two parts: what each
namespace adds at rest, which [section
5](#5-database-calls-memory-cpu-and-metric-series-per-namespace) gives, and the workload each
namespace will carry, which you size separately for your own workflows.

**What a namespace adds at rest is now placed in context.** [Which resources this playbook
measures](#which-resources-this-playbook-measures) describes the resources the numbers in this
playbook are given in.

### Which resources this playbook measures

A Temporal cluster uses **two separate databases**. The **primary store** holds workflow state,
history events and the server's own bookkeeping. The **visibility store** holds the searchable
index behind workflow list and query APIs, and can be the same engine or a different one.

**Every database number in this playbook is for the primary store, because at rest the visibility
store is idle.** The server writes to it only when a workflow execution starts, changes its search
attributes or memo, closes or is deleted, and reads from it only when someone lists or counts
workflows — all of it workload. In testing, the history service's visibility queue had no work at
all while the cluster was idle. The one exception is the twice-daily history and task queue scans
([point 9 of the background work](#background-work-a-temporal-server-always-runs)): they run as
internal workflows, so each run records its own start and close in visibility.

Two resources are measured here as numbers you can size with:

- **Database calls per second.** A *database call* is one request the server sends to the primary
  store — a read, a write or a delete, all counted together. Most of them are driven by the number
  of history shards. Where the mix matters, the playbook says how many are reads, writes or
  deletes.
- **Memory, per service instance**, given as two numbers:
  - **live heap** — the memory the server's own data takes up after garbage collection has run.
    Use it to see what grows, and by how much, as shards and namespaces are added.
  - **resident memory** — all the memory the process occupies on the machine, including memory Go
    keeps in reserve and the runtime itself. At rest it is several times live heap, and it is what
    to compare with a pod's memory limit.

History memory is set by shard count, which is fixed when the cluster is created. The memory that
keeps growing over a cluster's life is in the worker, the internal frontend and matching, and it
grows with namespace count.

**CPU used by the Temporal services was measured, but only its pattern is given.** At rest, most
of it is in the history service and follows shard count. Adding namespaces raises it only in the
worker, the internal frontend and matching. The CPU percentages depend on the machine, so this
playbook gives no CPU numbers to copy.

**What this playbook measures is now covered.** [Audience and scope](#audience-and-scope) says who
the playbook is for, what it does not cover, and which clusters it applies to.

## Audience and scope

**Audience**

Operators of self-hosted Temporal clusters.

**This playbook does not cover**

- **Choosing a history shard count, or sizing a cluster for throughput.** Both depend on the
  workload you plan to run; this playbook covers only what a cluster uses at rest.
- **Disk.** Resource utilization here means database calls, memory and CPU, not bytes stored.
- **Running a namespace deletion.** This playbook covers only the memory and metric series that
  deleted namespaces leave behind — [section
  5.6](#56-deleted-namespaces-memory-and-metric-series-freed-by-restarting-the-worker-internal-frontend-and-matching-services).
  How to run a deletion is in [Namespace Deletion — System Workflow](./namespace-deletion.md).
- **Resource use at rest in a multi-cluster setup.** Every number here is for a single cluster; a
  cluster that replicates to another does replication work on top.

**Applies to**

- **Single-cluster deployments with any primary store database.** Every number was measured on
  PostgreSQL. The one difference between databases is the task queue scan, which runs only on SQL
  databases: PostgreSQL, MySQL and SQLite.
- **Server v1.19 or later.** Before v1.19, every history queue read the database on every poll
  interval even when idle, so older servers have a larger baseline than described here.

**Dashboard**

This playbook is written against the
[Temporal Server dashboard](../observability/dashboards/server/temporal-server-readme.md#groups-and-panels),
**v2.28.0 or later**, and refers to panels by name. Every panel it uses, with the dashboard group
it sits in:

| Dashboard group | Panels this playbook uses |
|---|---|
| [24. Resting Load (Idle Baseline)](../observability/dashboards/server/temporal-server-readme.md#24-resting-load-idle-baseline) | **Database Calls Total, 30-Minute Average**<br>**Database Calls per Operation, 30-Minute Average**<br>**Namespaces Registered**<br>**Namespace Registry Refresh Latency**<br>**Metric Series Scraped per Service**<br>**Heap per Service Instance, 10-Minute Low** |
| [3. Persistence Requests, Latencies and Errors](../observability/dashboards/server/temporal-server-readme.md#3-persistence-requests-latencies-and-errors) | **Persistence Requests Total per Operation** — named only to say why it is not used for reading a baseline |
| [20. History Task DLQ / Terminal Failures](../observability/dashboards/server/temporal-server-readme.md#20-history-task-dlq--terminal-failures) | **Dead-Letter Queue Depth by Category** — named only for how often it updates |

Notes on these panels:

- **Namespaces Registered** and **Namespace Registry Refresh Latency** are empty on servers before
  v1.31.0.
- Memory is shown as Go heap. Resident memory and CPU are not exported by the server; read them
  from your container metrics.
- [Section 9.3](#93-metrics) lists every metric behind these panels.

**Alerts**

This playbook has no alerts: a cluster's resting load is its normal state, so an alert on it would
fire on every healthy cluster. [Section 7.5](#75-why-resting-load-has-no-alert) names the one
signal worth watching instead.

**How this playbook is laid out**

- [Section 1](#1-what-counts-as-an-idle-temporal-cluster) and [section
  2](#2-what-resource-use-scales-with-service-instances-shards-and-namespaces) define an idle
  cluster and the three counts its baseline depends on.
- [Section 3](#3-database-calls-memory-and-cpu-per-service-instance) to [section
  6](#6-database-calls-from-restarts-and-periodic-jobs) take each part of the baseline on its own
  — service instances, history shards, namespaces, and the periods with extra database calls — and
  explain what each adds and why.
- [Section 7](#7-measuring-the-baseline-of-a-cluster-you-already-run) brings those results
  together: the test cluster's totals, a per-service breakdown, and a checklist for a clean
  reading on your own cluster.
- [Section 8](#8-calculating-the-baseline-of-a-cluster-you-are-planning) works out the baseline of
  a cluster you are planning, and [section 9](#9-settings-and-metrics-reference) lists every
  setting and metric.

If you want the results first, [section 7](#7-measuring-the-baseline-of-a-cluster-you-already-run)
is a good place to start; [sections 3](#3-database-calls-memory-and-cpu-per-service-instance) to
[6](#6-database-calls-from-restarts-and-periodic-jobs) explain where each of its numbers comes
from.

---

## Contents

1. [What counts as an idle Temporal cluster](#1-what-counts-as-an-idle-temporal-cluster)
   - [1.1 Two states that are not a baseline](#11-two-states-that-are-not-a-baseline)
   - [1.2 What was measured, and on which test cluster](#12-what-was-measured-and-on-which-test-cluster)
2. [What resource use scales with: service instances, shards and namespaces](#2-what-resource-use-scales-with-service-instances-shards-and-namespaces)
3. [Database calls, memory and CPU per service instance](#3-database-calls-memory-and-cpu-per-service-instance)
   - [3.1 Database calls each service instance makes at rest](#31-database-calls-each-service-instance-makes-at-rest)
   - [3.2 Database calls made once for the whole cluster](#32-database-calls-made-once-for-the-whole-cluster)
   - [3.3 Settings for service instance and cluster-wide database calls at rest](#33-settings-for-service-instance-and-cluster-wide-database-calls-at-rest)
   - [3.4 Memory and CPU a service instance holds at rest](#34-memory-and-cpu-a-service-instance-holds-at-rest)
4. [Database calls, memory and CPU from history shards](#4-database-calls-memory-and-cpu-from-history-shards)
   - [4.1 What each shard does at rest](#41-what-each-shard-does-at-rest)
   - [4.2 Transfer, visibility, outbound and replication queues at rest](#42-transfer-visibility-outbound-and-replication-queues-at-rest)
   - [4.3 Archival enabled in the static configuration](#43-archival-enabled-in-the-static-configuration)
   - [4.4 Settings for how often each shard reads and writes](#44-settings-for-how-often-each-shard-reads-and-writes)
   - [4.5 History memory and CPU follow shard count](#45-history-memory-and-cpu-follow-shard-count)
5. [Database calls, memory, CPU and metric series per namespace](#5-database-calls-memory-cpu-and-metric-series-per-namespace)
   - [5.1 Database calls each namespace adds](#51-database-calls-each-namespace-adds)
   - [5.2 The per-namespace worker and its polls](#52-the-per-namespace-worker-and-its-polls)
   - [5.3 Worker, internal frontend and matching memory per namespace](#53-worker-internal-frontend-and-matching-memory-per-namespace)
   - [5.4 Worker, internal frontend and matching CPU per namespace](#54-worker-internal-frontend-and-matching-cpu-per-namespace)
   - [5.5 Metric series each namespace adds](#55-metric-series-each-namespace-adds)
   - [5.6 Deleted namespaces' memory and metric series, freed by restarting the worker, internal frontend and matching services](#56-deleted-namespaces-memory-and-metric-series-freed-by-restarting-the-worker-internal-frontend-and-matching-services)
   - [5.7 Settings for the per-namespace work](#57-settings-for-the-per-namespace-work)
6. [Database calls from restarts and periodic jobs](#6-database-calls-from-restarts-and-periodic-jobs)
   - [6.1 Database calls after a full cluster restart](#61-database-calls-after-a-full-cluster-restart)
   - [6.2 History and task queue scans, twice a day](#62-history-and-task-queue-scans-twice-a-day)
   - [6.3 Dead-letter queue depth, counted every three hours](#63-dead-letter-queue-depth-counted-every-three-hours)
7. [Measuring the baseline of a cluster you already run](#7-measuring-the-baseline-of-a-cluster-you-already-run)
   - [7.1 Test cluster results](#71-test-cluster-results)
   - [7.2 Checklist for a clean baseline reading](#72-checklist-for-a-clean-baseline-reading)
   - [7.3 What each panel on the Resting Load row should show](#73-what-each-panel-on-the-resting-load-row-should-show)
   - [7.4 Re-taking a baseline after the cluster changes](#74-re-taking-a-baseline-after-the-cluster-changes)
   - [7.5 Why resting load has no alert](#75-why-resting-load-has-no-alert)
8. [Calculating the baseline of a cluster you are planning](#8-calculating-the-baseline-of-a-cluster-you-are-planning)
   - [8.1 Database calls at rest](#81-database-calls-at-rest)
   - [8.2 Memory per service instance at rest](#82-memory-per-service-instance-at-rest)
   - [8.3 Metric series](#83-metric-series)
   - [8.4 Calculating the baseline of an example cluster, step by step](#84-calculating-the-baseline-of-an-example-cluster-step-by-step)
   - [8.5 What the arithmetic does not cover](#85-what-the-arithmetic-does-not-cover)
   - [8.6 Explaining a measured baseline above its calculated level](#86-explaining-a-measured-baseline-above-its-calculated-level)
9. [Settings and metrics reference](#9-settings-and-metrics-reference)
   - [9.1 Dynamic config](#91-dynamic-config)
   - [9.2 Static configuration and environment](#92-static-configuration-and-environment)
   - [9.3 Metrics](#93-metrics)

---

## 1. What counts as an idle Temporal cluster

An **idle cluster**, throughout this playbook, means all of the following at once:

- every service instance is running, and every history shard is owned by a history instance
- namespaces are registered — how many is one of the inputs to the baseline
- no workflows are running, and none have run recently
- no SDK workers are connected: connected workers hold long polls, which use memory and
  connections, so they count as workload even when nothing is executing

The server's own background work, described in the introduction, is part of the baseline, not an
exception to it.

**What "at rest" means in this playbook.** Every number described as *at rest* was measured on an
idle cluster, with no workload. The background work behind those numbers does not stop when
workload arrives: it keeps running underneath, and parts of it grow with workload — [section
7.2](#72-checklist-for-a-clean-baseline-reading) explains which. Read *at rest* as the part of a
cluster's resource use that is there whatever the workload.

**The definition of an idle cluster is now set.** [Section
1.1](#11-two-states-that-are-not-a-baseline) covers two states that look idle and are not.

### 1.1 Two states that are not a baseline

- **A cluster that restarted in the last half hour.** A full restart produces a burst of database
  calls, then five-minute waves still visible half an hour later — [section
  6.1](#61-database-calls-after-a-full-cluster-restart) has the measured curve.
- **A cluster in the middle of a history or task queue scan.** These scans run twice a day —
  [section 6.2](#62-history-and-task-queue-scans-twice-a-day).

Baseline measurements taken in either state are skewed, usually too high, so they do not show the
cluster's real baseline.

**What counts as an idle cluster is now covered.** [Section
1.2](#12-what-was-measured-and-on-which-test-cluster) describes the setup the baseline numbers in
this playbook were measured on.

### 1.2 What was measured, and on which test cluster

Every number in this playbook that is not a server default was **measured on one test cluster**:

- **Store:** PostgreSQL, with Elasticsearch for visibility
- **History shards:** 2048; one measurement was repeated at 512 to see what changes with shard
  count
- **Service instances:** eight — two history, two matching, two frontend, one internal frontend
  and one worker — all running on one machine
- **Server:** equivalent to release v1.32.1 for everything measured here
- **Settings:** every dynamic config setting at the server's default
- **Namespaces:** the one input that varies between measurements; the count is given with each
  number

How the numbers were taken:

- **Archival was enabled**, as it is in the configuration templates the server ships with. It
  doubles the timer work in [point 2 of the background
  work](#background-work-a-temporal-server-always-runs); where a number was measured with archival
  off, it says so.
- **An external health poller ran against the cluster**, checking history service health as
  [History Host Health](./history-host-health.md#8-setting-up-the-deephealthcheck-poller)
  describes. It makes no database calls, so it has no effect on the numbers here; a cluster
  without one has the same numbers.
- **Database calls per second** were averaged over whole five-minute periods — the length of the
  shards' own cycle — starting at least ten minutes after the service instances were last
  restarted, so the burst of database calls at startup is left out.
- **Memory** was read two ways for each service instance: live heap, right after forcing a garbage
  collection, and resident memory.
- **CPU** was each service's own CPU counter, averaged over ten to twenty minutes.

Your own cluster's numbers will differ. What these numbers show reliably is the pattern — which
service uses the most, which uses almost nothing, and which setting changes what — so measure your
own baseline as well; [section 7](#7-measuring-the-baseline-of-a-cluster-you-already-run)
describes how.

**What was measured is now covered.** [Section
2](#2-what-resource-use-scales-with-service-instances-shards-and-namespaces) describes the three
counts that decide how large a cluster's baseline is: service instance count, shard count and
namespace count.

---

## 2. What resource use scales with: service instances, shards and namespaces

This section gives the structure for the rest of the playbook: the three counts an idle cluster's
resource use depends on — service instance count, shard count and namespace count — and what each
count adds in database calls and in memory. [Section
3](#3-database-calls-memory-and-cpu-per-service-instance), [section
4](#4-database-calls-memory-and-cpu-from-history-shards) and [section
5](#5-database-calls-memory-cpu-and-metric-series-per-namespace) then cover one count each.

Each table below first names the services a count applies to — every service for service instance
count, the history service alone for shard count.

**Service instance count** — how many frontend, history, matching, worker and internal frontend
instances you run.

| | What each service instance adds |
|---|---|
| Which services | All of them: frontend, history, matching, worker and internal frontend |
| Database calls | Re-reads the namespace list every two seconds, refreshes its membership entry every ten to fifteen seconds, and re-reads the cluster's record once a minute |
| Memory | A fixed amount of memory, depending on the service instance's role |

**History shard count** — fixed when the cluster is created.

| | What the shards add |
|---|---|
| Which services | History only — the shards are spread across the history instances |
| Database calls | Every five minutes, each shard reads ahead for timers coming due, deletes the timers already handled and saves its progress. With archival enabled, each shard also does the same reads and deletes for archival tasks |
| Memory | Each history instance's memory grows with the number of shards it owns |

**Namespace count** — how many namespaces are registered.

| | What each namespace adds |
|---|---|
| Which services | Matching and every service instance for database calls; worker, internal frontend and matching for memory |
| Database calls | Four reads every five minutes, made by the matching service, and one more row read on each namespace list refresh, made by every service instance |
| Memory | Worker, internal frontend and matching memory, growing with every namespace |

The matching service also makes about 0.85 database calls per second that none of the three counts
changes; [section 3.2](#32-database-calls-made-once-for-the-whole-cluster) covers where they come
from.

These three tables describe a cluster at the server's default settings; [section
3.3](#33-settings-for-service-instance-and-cluster-wide-database-calls-at-rest), [section
4.4](#44-settings-for-how-often-each-shard-reads-and-writes) and [section
5.7](#57-settings-for-the-per-namespace-work) cover the settings that change them.

**Service instance count, shard count and namespace count are now described.** [Section
3](#3-database-calls-memory-and-cpu-per-service-instance) takes service instance count first: the
database calls, memory and CPU each service instance adds.

---

## 3. Database calls, memory and CPU per service instance

This section covers what each service instance adds at rest, before shards or namespaces are
considered: database calls, memory and CPU, and the settings that change them. A *service
instance* is one running frontend, history, matching, worker or internal frontend process.

[Sections 3.1](#31-database-calls-each-service-instance-makes-at-rest) to
[3.3](#33-settings-for-service-instance-and-cluster-wide-database-calls-at-rest) cover database
calls and the settings that change them; [section
3.4](#34-memory-and-cpu-a-service-instance-holds-at-rest) covers memory and CPU.

### 3.1 Database calls each service instance makes at rest

Every service instance, whatever its role, makes the same three kinds of database call while the
cluster is idle. They are refreshing namespace settings, refreshing cluster metadata and tracking
cluster membership — [points 3, 4 and 6 of the background
work](#background-work-a-temporal-server-always-runs).

The intervals below are the server's defaults. Two of them are dynamic config settings, named in
the table and covered in [section
3.3](#33-settings-for-service-instance-and-cluster-wide-database-calls-at-rest); the membership
refresh is fixed in the server's code.

| Work | How often, by default | Dynamic config setting | Database calls per second, per service instance |
|---|---|---|---:|
| Re-read the namespace list | Every 2 seconds, one read per 1000 namespaces | `system.namespaceCacheRefreshInterval` | 0.5 |
| Refresh its membership entry | Every 10 to 15 seconds, 12.5 on average | None — fixed | 0.08 |
| Re-read the cluster's own record | Once a minute | `system.clusterMetadataRefreshInterval` | 0.017 |
| **Total** | | | **about 0.6** |

**Each service instance adds about 0.6 database calls per second, whatever its role.** A frontend,
a history and a worker instance all make the same three database calls at the same rates.

For example, the test cluster ran eight service instances: 8 × 0.6 is about 4.8 database calls per
second. That is what was measured — each service instance's own rate came within one percent of
0.6 database calls per second — and it was about a tenth of the cluster's 53.6 database calls per
second at rest. Adding two more matching instances to that cluster would add about 1.2 database
calls per second.

**Adding namespaces barely changes how many namespace list reads a service instance makes.** Every
two seconds, each service instance fetches the whole namespace list, up to 1,000 namespaces per
database call: one database call per refresh up to 1,000 namespaces, two from 1,001 to 2,000, and
so on. Each database call does return more as namespaces are added; [section
5.1](#51-database-calls-each-namespace-adds) covers that growth.

**Where to see these database calls.**
**[Database Calls per Operation, 30-Minute Average](../observability/dashboards/server/temporal-server-readme.md#24-resting-load-idle-baseline)**
shows every database call by name. The three database calls each service instance makes appear
there as `ListNamespaces` (the namespace list read), `UpsertClusterMembership` (the membership
refresh) and `ListClusterMetadata` (the cluster record read).

**The database calls each service instance makes are now covered.** [Section
3.2](#32-database-calls-made-once-for-the-whole-cluster) covers the two pieces of background work
that run once for the whole cluster instead.

### 3.2 Database calls made once for the whole cluster

Two pieces of background work run once for the whole cluster, not once per service instance.
Adding service instances spreads them out without adding to them. The Nexus check and the internal
task queues are both reads made by the matching service.

**The Nexus endpoint check** — [point 5 of the background
work](#background-work-a-temporal-server-always-runs). One matching instance owns it for the whole
cluster and reads the endpoint list every ten seconds: **0.1 reads per second**, however many
matching instances you run.

**The worker service's own internal task queues** — [point 8 of the background
work](#background-work-a-temporal-server-always-runs). The worker service runs internal workers of
its own in the `temporal-system` namespace, and the matching service holds the task queues they
poll. Each task queue has a workflow side and an activity side, and by default each side is split
into four partitions; [Change Task Queue
Partitions](./change-task-queue-partitions.md#what-are-task-queue-partitions) explains what a
partition is. Every loaded partition confirms which matching instance owns it with one database
read a minute. At the server's defaults, on a SQL database:

| Internal task queue | Used for | Partitions, by default |
|---|---|---:|
| `default-worker-tq` | System workflows, such as namespace deletion and dead-letter queue jobs | 8 (4 workflow, 4 activity) |
| `temporal-sys-history-scanner-taskqueue-0` | The history scan | 8 |
| `temporal-sys-tq-scanner-taskqueue-0` | The task queue scan — SQL databases only | 8 |
| `temporal-sys-processor-parent-close-policy` | Applying a parent workflow's close policy to its child workflows | 8 |
| `temporal-sys-dlq-activity-tq` | Activities of the dead-letter queue jobs | 4 (activity only) |
| `temporal-sys-delete-namespace-activity-tq`, `temporal-sys-migration-activity-tq` | Activities of namespace deletion and of namespace migration | 1 each — the server sets these to one partition |
| Sticky queues | One per internal workflow worker, so the worker can keep workflow state cached | 4 (one partition each) |
| **Total** | | **about 42** |

About 42 partitions at one read a minute is 0.7 reads per second. The test cluster measured
**about 0.75**; the small difference was not traced to a particular partition. On Cassandra, the
worker service does not run the task queue scan, so it never polls that scan's task queue: the
total drops by its 8 partitions and its sticky queue, to about 33.

**Adding up the database calls that do not come from shards.** A database call is one request from
the server to its database, read or write. On the test cluster, at two namespaces:

- **Eight service instances × about 0.6 = about 4.8 database calls per second** — the namespace
  list reads, cluster record reads and membership writes from [section
  3.1](#31-database-calls-each-service-instance-makes-at-rest).
- **The Nexus endpoint check: 0.1 reads per second.**
- **The internal task queues: about 0.75 reads per second.**
- **Two namespaces × about 0.013 = about 0.03 reads per second** — [section
  5.1](#51-database-calls-each-namespace-adds) covers them.

Together that is **5.66 database calls per second**. The measured rate was 5.68 at 2048 shards and
5.71 at 512 — the same at both shard counts, so this part does not depend on shards. Every other
database call at rest comes from the shards, which [section
4](#4-database-calls-memory-and-cpu-from-history-shards) covers.

**Database calls per service instance and per cluster are now covered.** [Section
3.3](#33-settings-for-service-instance-and-cluster-wide-database-calls-at-rest) covers the
settings that change them.

### 3.3 Settings for service instance and cluster-wide database calls at rest

The table below lists the dynamic config settings behind the database calls in [section
3.1](#31-database-calls-each-service-instance-makes-at-rest) and [section
3.2](#32-database-calls-made-once-for-the-whole-cluster): each default, what raising it does and
what 0 does. These effects come from the server's code and were not measured; if you change one of
these settings, measure your resting load again with [section
7](#7-measuring-the-baseline-of-a-cluster-you-already-run).

| Setting | Default | What raising it does | What `0` does |
|---|---|---|---|
| `system.namespaceCacheRefreshInterval` | 2s | Fewer namespace list reads, in proportion. A changed namespace setting takes up to that long to reach every service instance. | Every service instance reads the whole namespace list again as soon as the previous read returns, without ever pausing. |
| `system.clusterMetadataRefreshInterval` | 1m | Fewer reads of the cluster's record. Changes to another cluster's address take longer to be picked up. The value is read once, when the service instance starts, so a change takes effect only after the service instance restarts. | No service instance can start, so the cluster does not come up: the server cannot start a timer with a zero interval. |
| `matching.nexusEndpointsRefreshInterval` | 10s | Fewer Nexus endpoint reads, on one matching instance only. | The matching instance that owns the Nexus check reads the endpoint list again as soon as the previous read returns, without ever pausing. |
| `matching.updateAckInterval` | 1m (5m for the per-namespace worker's task queue) | Fewer ownership reads per task queue partition. | Every loaded partition reads its ownership record again as soon as the previous read returns, without ever pausing. |
| `matching.numTaskqueueReadPartitions`, `matching.numTaskqueueWritePartitions` | 4 per task queue; 1 for a few internal queues the server sets itself ([section 3.2](#32-database-calls-made-once-for-the-whole-cluster)) | More partitions on the internal task queues that have four; every partition loaded adds one read a minute. Change the read and write settings together — [Changing Task Queue Partitions](./change-task-queue-partitions.md) covers how. | The same as 1: one partition per task queue. |

The membership write in [section 3.1](#31-database-calls-each-service-instance-makes-at-rest) has
no setting. Every service instance writes its row in the cluster membership table every ten to
fifteen seconds, so the other service instances can find it, and that interval is fixed in the
server's code.

> **Recommendation: never set these intervals to 0.** A 0 does not turn the reads off — it makes
> them run without any pause, or stops the cluster from starting:
>
> - `system.namespaceCacheRefreshInterval` at 0 — every service instance re-reads the namespace
>   list nonstop.
> - `matching.nexusEndpointsRefreshInterval` at 0 — the matching instance that owns the Nexus check
>   re-reads the endpoint list nonstop.
> - `matching.updateAckInterval` at 0 — every loaded task queue partition re-reads its ownership
>   record nonstop; with `matching.enableFairness` turned on, the matching service crashes instead.
> - `system.clusterMetadataRefreshInterval` at 0 — no service instance can start. Every frontend,
>   internal frontend, history, matching and worker process exits as it starts, so the cluster does
>   not come up. Instances already running keep the old value until they are restarted.
>
> To make any of these reads less frequent, raise that setting's interval instead of setting it to
> 0.

**For resting load, leave every setting in this section at the server's default.** The database
calls they control are a small part of a resting cluster: on the test cluster, the eight service
instances and the two cluster-wide reads together made about 5.7 of the 53.6 database calls per
second. Raising these intervals would save little, and lowering them adds database calls. The one
setting worth revisiting is `system.namespaceCacheRefreshInterval`, and only on a cluster with
many namespaces; [section 5.7](#57-settings-for-the-per-namespace-work) covers it.

**The settings for these database calls are now covered.** [Section
3.4](#34-memory-and-cpu-a-service-instance-holds-at-rest) covers the memory and CPU each service
instance holds.

### 3.4 Memory and CPU a service instance holds at rest

Each service instance holds a fixed amount of memory and uses a small amount of CPU before any
shard or namespace work is added. Measured on the test cluster, at two namespaces, in two separate
runs:

| Service | Live heap (MB) | Resident memory (MB) | CPU (% of one core) |
|---|---:|---:|---:|
| Frontend | 15–19 | 58–88 | 0.6–0.8 |
| Internal frontend | 20–21 | 87–96 | 0.5–0.7 |
| Matching | 18–20 | 75–102 | 0.7–1.0 |
| Worker | 32 | 103–112 | 0.5–0.7 |
| History | depends on the shards it owns | depends on the shards it owns | depends on the shards it owns |

**Set each pod's memory limit from the resident memory column, not from live heap.** Resident
memory is all the memory the process takes up on the machine, and it is what a pod's memory limit
is compared against. Live heap is only the part the server is actively using after garbage
collection; at rest, resident memory was three to six times larger. The ranges in the table above
are the spread between two runs on the test cluster.

**The history service's memory and CPU depend on its shards.** Almost all of a history instance's
memory and CPU at rest depends on how many shards it owns, so [section
4.5](#45-history-memory-and-cpu-follow-shard-count) gives its numbers instead of the table above.

**Use the CPU numbers in the table above to compare services, not to size pods.** The CPU numbers
were measured on one machine and depend on its core speed and core count, so they do not say how
many cores a pod needs. What they do show is that, outside the history service, each service
instance at rest used about one percent of a core or less.

**Next:** each service instance adds about 0.6 database calls per second and a fixed amount of
memory, and that part of the baseline is small. The part that is not small comes from history
shards — [section 4](#4-database-calls-memory-and-cpu-from-history-shards) takes shard count,
which accounts for about nine tenths of the database calls at rest.

---

## 4. Database calls, memory and CPU from history shards

This section covers the database calls, memory and CPU that history shards add at rest, and the
settings that multiply them.

[Sections 4.1](#41-what-each-shard-does-at-rest) to
[4.4](#44-settings-for-how-often-each-shard-reads-and-writes) cover database calls and the
settings that change them; [section 4.5](#45-history-memory-and-cpu-follow-shard-count) covers
history memory and CPU.

### 4.1 What each shard does at rest

Everything in this section runs on every shard all the time, with or without workload; at rest it
is all a shard does ([section 1](#1-what-counts-as-an-idle-temporal-cluster) defines *at rest*).

Each shard keeps its pending work in the database, sorted by kind into a few **queues**: one for
timers, one for archival, and others for work that is due right away. These queues belong to the
history service; they are not the task queues your workers poll. The history instance that owns
the shard reads each queue to find work.

With no workflows running, only the timer queue and the archival queue make database calls, plus a
regular save of the shard's progress:

| What each shard does, every five minutes by default | Database calls | Operation name |
|---|---|---|
| Looks ahead in its timer queue for timers coming due; at rest, these reads come back empty | 2 reads | `GetTimerTasks` |
| Clears the part of its timer queue it has already read past | 1 delete | `RangeCompleteTimerTasks` |
| Saves its progress, even when nothing has changed | 1 write | `UpdateShard` |
| Only when archival is enabled: the same two reads and one delete on its archival queue | 2 reads, 1 delete | `GetArchivalTasks`, `RangeCompleteArchivalTasks` |

Every five minutes is the server's default for all four rows of the table above; [section
4.4](#44-settings-for-how-often-each-shard-reads-and-writes) covers the settings for that
five-minute interval.

**The shard-driven load is therefore a simple multiplication:**

- with archival enabled: 7 database calls per shard every 300 seconds (4 reads, 2 deletes, 1
  write), about **0.023 database calls per second per shard**
- with archival disabled: 4 database calls per shard every 300 seconds (2 reads, 1 delete, 1
  write), about **0.013 database calls per second per shard**

| Database calls per second from the shards | 2048 shards | 512 shards |
|---|---:|---:|
| Predicted, archival enabled (shards × 7 ÷ 300) | 47.8 | 11.9 |
| Measured in testing | 47.9 | 11.9 |

Everything outside the shard-driven part stayed the same at 2048 and at 512 shards — 5.68 and 5.71
database calls per second — so shard count changes the shard-driven part and nothing else. All
five operations show on
**[Database Calls per Operation, 30-Minute Average](../observability/dashboards/server/temporal-server-readme.md#24-resting-load-idle-baseline)**.

**What the timer and archival queues do at rest is now covered.** [Section
4.2](#42-transfer-visibility-outbound-and-replication-queues-at-rest) covers the shard's other
queues, and why they add nothing at rest.

### 4.2 Transfer, visibility, outbound and replication queues at rest

Every shard also has a transfer queue, a visibility queue, an outbound queue and a replication
queue. **At rest, none of them makes a database call.** They read only after a task of their kind
has been written, and on an idle cluster none is. The timer queue is different: it waits for a
point in time, and time keeps passing, so it keeps reading ahead.

Two consequences follow:

- **Lowering the poll interval of these queues does not change resting load.** Measured on the
  test cluster: `history.transferProcessorMaxPollInterval` lowered from 1 minute to 10 seconds
  left transfer reads at zero. The same applies to `history.visibilityProcessorMaxPollInterval`
  and `history.outboundProcessorMaxPollInterval` (both 1 minute by default), by the same
  mechanism. These intervals matter under workload, which [History Task
  Processing](./history-task-processing-tuning.md#7-tuning-how-fast-tasks-are-loaded) covers.
- **Versions before v1.19 behave differently.** Up to v1.18, the transfer and visibility queues
  read the database on every poll interval whether or not they had tasks. An idle cluster on those
  versions makes far more database calls than the multiplication in [section
  4.1](#41-what-each-shard-does-at-rest) gives.

The timer, archival, transfer, visibility and outbound queues also each run a checkpoint every 30
seconds by default. At rest the checkpoint adds no database calls of its own: it deletes rows only
after the queue has read past them, and saves the shard's progress only every five minutes — the
delete and the write already counted in [section 4.1](#41-what-each-shard-does-at-rest).

**The transfer, visibility, outbound and replication queues are now covered.** [Section
4.3](#43-archival-enabled-in-the-static-configuration) covers the archival queue, which does make
database calls at rest when archival is enabled.

### 4.3 Archival enabled in the static configuration

**Every shard runs an archival queue when archival is enabled for history or for visibility in the
cluster's static configuration** — the `archival:` block of the server's configuration file — even
if no namespace archives anything. A namespace's own archival setting does not start or stop the
queue.

**The configuration templates the server ships with enable archival for history and for
visibility**, so a cluster built from them runs the archival queue on every shard.

> **Recommendation: if no namespace on the cluster uses archival, make sure archival is turned off
> in the static configuration.** Enabled but unused, it adds database calls on every shard and does
> nothing else. If any namespace uses archival, keep it enabled: those database calls are part of
> running it.

**Turning off unused archival takes about two fifths off the resting rate at low namespace
counts.** Measured on the test cluster at 2048 shards: the archival reads and deletes, about 20.6
database calls per second, were gone entirely, and the resting rate at 500 namespaces fell from
60.4 to 38.2 database calls per second, read over a window just under two five-minute cycles,
which reads slightly low. History memory fell by about 17 MB per history instance and history CPU
by about 0.4 points of a core.

**Archival's share of the database calls is now covered.** [Section
4.4](#44-settings-for-how-often-each-shard-reads-and-writes) covers the settings that change how
often each shard makes its database calls.

### 4.4 Settings for how often each shard reads and writes

Each shard makes its database calls once per cycle — every five minutes by default ([section
4.1](#41-what-each-shard-does-at-rest)). The settings below set those cycles. Shortening a cycle
raises the database calls on every shard at once: the shard-driven load is shards × database calls
per cycle ÷ cycle length. Three of these settings were changed on the test cluster, and their
measured effect is in the table below; every other effect comes from the server's code.

| Setting | Default | Effect at rest | What `0` does |
|---|---|---|---|
| `history.timerProcessorMaxPollInterval` | 5m | Sets the timer queue's cycle. Measured at 1 minute, 2048 shards: timer reads 13.7 → **67.1** database calls/s, timer deletes 6.9 → **33.5**. | Every shard re-reads its timer queue about once a second (`history.timerProcessorMaxTimeShift`, 1 second by default) instead of every five minutes. |
| `history.archivalProcessorMaxPollInterval` | 5m | The same, for the archival queue, when archival is enabled. | The same, for the archival queue. |
| `history.shardUpdateMinInterval` | 5m | How often each shard saves its progress. Measured at 1 minute, 2048 shards: shard writes 6.8 → **32.4** database calls/s. | A progress write on every checkpoint — several per shard every 30 seconds. |
| `history.timerProcessorUpdateAckInterval`, `history.archivalProcessorUpdateAckInterval` | 30s | How often the checkpoint runs. Adds no database calls of its own at rest. | Checkpoints run without ever pausing, adding few database calls at rest. |
| `history.transferProcessorMaxPollInterval`, `history.visibilityProcessorMaxPollInterval`, `history.outboundProcessorMaxPollInterval` | 1m | **None at rest** — those queues do not read without tasks. Measured for transfer. | No database calls at rest, but each of these queues on every shard re-polls without pausing. |

> **Recommendation: never set these intervals to 0.** A 0 does not turn the work off — it makes
> it run far more often, or without any pause, on every shard:
>
> - `history.timerProcessorMaxPollInterval` and `history.archivalProcessorMaxPollInterval` at 0 —
>   every shard re-reads its timer or archival queue about once a second instead of every five
>   minutes.
> - `history.shardUpdateMinInterval` at 0 — every shard saves its progress on every checkpoint,
>   several times every 30 seconds.
> - `history.timerProcessorUpdateAckInterval` and `history.archivalProcessorUpdateAckInterval` at
>   0 — every shard runs its checkpoint nonstop.
>
> To make any of these less frequent, raise that setting's interval instead.

**Lowering `history.timerProcessorMaxPollInterval` and `history.shardUpdateMinInterval` to 1
minute tripled the resting rate in testing, from 53.6 to 158.8 database calls per second**, with
no workload at all, and the database's own CPU went from 0.7 to 1.8 percent of a core. Shorter
intervals make a cluster pick up timers and record progress more often; at rest, the database
calls that adds are made by every shard, all the time. When a resting rate is well above what
[section 4.1](#41-what-each-shard-does-at-rest) predicts, `history.timerProcessorMaxPollInterval`
and `history.shardUpdateMinInterval` are the first place to look.

[History Persistence QPS
Limits](./history-persistence-qps-limits.md#2-the-settings-and-how-they-interact) covers the
limits that turn database calls away once load is high, and is the place to go if a multiplied
resting load is meeting them.

**The settings for the shard-driven database calls are now covered.** [Section
4.5](#45-history-memory-and-cpu-follow-shard-count) covers history memory and CPU, which also
follow shard count.

### 4.5 History memory and CPU follow shard count

**A history instance's memory and CPU at rest grow with the number of shards it owns.** Measured
on the test cluster at two shard counts, with each of the two history instances owning half the
shards:

| | 2048 shards (about 1024 per history instance) | 512 shards (about 256 per history instance) |
|---|---:|---:|
| Live heap per history instance | 124 MB | 46 MB |
| Resident memory per history instance | about 390 MB | about 145 MB |
| CPU per history instance (% of one core) | 3.6 | 1.25 |

**Each shard a history instance owns adds about 0.1 MB of live heap.** The rest of a history
instance's live heap, about 20 MB, is fixed: it does not depend on how many shards the instance
owns. Both numbers are worked out from the 2048-shard and 512-shard measurements in the table
above. Two measurements are not enough to be sure memory grows in a straight line, so do not rely
on 0.1 MB per shard for a cluster with far fewer than 512 or far more than 2048 shards: measure
history memory on your own cluster instead, with the panels in [section
7.3](#73-what-each-panel-on-the-resting-load-row-should-show). No other service's memory changed
with shard count.

**Adding history instances adds little history memory, because the shards are shared out among
more history instances.** Each history instance then owns fewer shards and holds less memory for
them. The memory for shards, summed across all history instances, stays about the same; each extra
history instance adds only its own fixed 20 MB or so of live heap. The test cluster always ran two
history instances, so after adding history instances, check every history instance's memory on
**[Heap per Service Instance, 10-Minute Low](../observability/dashboards/server/temporal-server-readme.md#24-resting-load-idle-baseline)**.

Namespace count, by contrast, did not raise history memory or CPU at all — measured flat from two
namespaces to five hundred.

**Next:** shard count sets most of the database calls at rest, and its effect is a plain
multiplication that changes only with shard count, archival and two settings. Namespace count
works differently — small in database calls, large in memory — and [section
5](#5-database-calls-memory-cpu-and-metric-series-per-namespace) takes it.

---

## 5. Database calls, memory, CPU and metric series per namespace

This section covers what each namespace adds at rest — database calls, memory, CPU and metric
series — and which services hold it. A namespace adds load even if no workflow ever runs in it,
because every namespace gets its own internal worker and its own task queues in the matching
service. Namespace count is the input that changes most over a cluster's life, so this is the
section to come back to when deciding how many namespaces a cluster can hold.

### 5.1 Database calls each namespace adds

**Each namespace adds four database reads every five minutes — about 0.013 database calls per
second.** Every namespace gets its own internal worker ([section
5.2](#52-the-per-namespace-worker-and-its-polls)), and that internal worker polls a task queue of
its own in the namespace, `temporal-sys-per-ns-tq` ([point 8 of the background
work](#background-work-a-temporal-server-always-runs)). The matching service makes the four reads
for that task queue, each every five minutes by default:

- **three ownership reads** (`GetTaskQueue`), one for each part of the task queue: a workflow
  partition, an activity partition and a sticky queue. The server sets this task queue to one
  partition instead of the default four ([section
  3.2](#32-database-calls-made-once-for-the-whole-cluster) explains partitions), so it has three
  parts.
- **one settings read** (`GetTaskQueueUserData`) for the task queue.

Measured on the test cluster, from 2 to 500 namespaces:

- the resting rate rose by **6.8 database calls per second** — about **0.0136 per namespace**
- all of the rise came from those two reads, `GetTaskQueue` and `GetTaskQueueUserData`

**The per-namespace rate grows in a straight line with namespace count**, so by the same
arithmetic:

- 1,000 namespaces add about **13 database calls per second**
- 10,000 namespaces add about **130 database calls per second**

**The namespace list read from [section
3.1](#31-database-calls-each-service-instance-makes-at-rest) grows differently.** Every two
seconds, each service instance reads the whole namespace list from the database. Up to 1,000
namespaces that is one database read per refresh, so adding namespaces does not add reads. What
grows is each read: it returns every namespace, so each refresh takes longer.

On the test cluster at 4 namespaces, one refresh averaged under 1 millisecond on every service
instance. Refresh times at larger namespace counts were not recorded, so watch
**[Namespace Registry Refresh Latency](../observability/dashboards/server/temporal-server-readme.md#24-resting-load-idle-baseline)**
on your own cluster as namespaces are added: it shows how long each refresh takes, for each
service.

**On Cassandra, every namespace list read goes to the same few nodes.** Cassandra keeps all
namespace rows in one partition, so the read every service instance makes every two seconds lands
on the nodes that hold that partition. The test cluster ran PostgreSQL, so this was not measured;
on a Cassandra cluster with many namespaces and service instances, check the read load on those
nodes.

**The database calls each namespace adds are now covered.** They are the small part of a
namespace's resting load; [section 5.2](#52-the-per-namespace-worker-and-its-polls) covers the
internal worker each namespace gets, where the larger part comes from.

### 5.2 The per-namespace worker and its polls

The per-namespace worker ([point 7 of the background
work](#background-work-a-temporal-server-always-runs)) is a Temporal SDK worker that the worker
service runs to carry out schedules, batch operations and Worker Deployment. **Every namespace
active on the cluster gets a per-namespace worker, whether or not any of those features is used**,
because parts of it are always on.

At the server's defaults, each per-namespace worker holds **four long polls** open on its
namespace's internal task queue: two for workflow tasks and two for activity tasks. On the test
cluster that was exactly 8 polls at 2 namespaces and 2,000 at 500. The polls run from the worker
service, through the internal frontend — or through the public frontends, when the cluster has no
internal frontend — to the matching service, which holds them open. A poll that gets no task ends
after a minute — `matching.longPollExpirationInterval`, 1 minute by default — and a new poll
replaces it straight away, so four polls per namespace are always open.

**Why the polls cost memory and CPU.** For every namespace, each of the three services on that
path keeps something running all the time:

- **Worker service** — a separate SDK client and SDK worker for the namespace, with its own
  pollers.
- **Internal frontend** — the namespace's four open poll requests.
- **Matching service** — the namespace's task queue, loaded in memory as the three parts in
  [section 5.1](#51-database-calls-each-namespace-adds), each with background work that re-reads
  it from the database; plus the four polls waiting for a task.

[Section 5.3](#53-worker-internal-frontend-and-matching-memory-per-namespace) and [section
5.4](#54-worker-internal-frontend-and-matching-cpu-per-namespace) give the memory and CPU measured
on those three services.

**When several internal frontends run, the polls are spread across them evenly by default**, so
adding an internal frontend instance splits the per-namespace memory and CPU among more instances.

> **Recommendation: run an internal frontend.** With an internal frontend deployed, the
> per-namespace polls stay off the public frontends that serve your clients: on the test cluster,
> their memory and CPU did not change as namespaces were added. The internal frontend's own memory
> and CPU grow with every namespace instead; [section
> 5.3](#53-worker-internal-frontend-and-matching-memory-per-namespace) covers adding instances as
> namespace count grows.

**The per-namespace worker and the path its polls take are now covered.** [Section
5.3](#53-worker-internal-frontend-and-matching-memory-per-namespace) gives the memory that path
adds per namespace.

### 5.3 Worker, internal frontend and matching memory per namespace

**Each namespace adds memory to three services: the worker service, the internal frontend and the
matching service.** Measured on the test cluster, which ran one worker instance, one internal
frontend, two matching instances and two frontend instances, each freshly started at 2 and again
at 500 namespaces. Every number in the table below is for one service instance:

| Service instance | Live heap, 2 namespaces | Live heap, 500 namespaces | Live heap added per namespace | Resident memory, 2 namespaces | Resident memory, 500 namespaces |
|---|---:|---:|---:|---:|---:|
| Worker (the only one) | 32 MB | 574 MB | 1.09 MB | about 110 MB | about 530 MB |
| Internal frontend (the only one) | 20 MB | 156 MB | 0.27 MB | about 95 MB | about 410 MB |
| Matching, first instance | 20 MB | 118 MB | 0.20 MB | about 100 MB | about 350 MB |
| Matching, second instance | 18 MB | 140 MB | 0.24 MB | about 95 MB | about 370 MB |
| Frontend, both instances | 15–17 MB | 14 MB | none | about 85 MB | about 55 MB |
| History, both instances | depends on the shards it owns | depends on the shards it owns | none | depends on the shards it owns | depends on the shards it owns |

> **Note:** namespaces add no memory to the history service. A history instance's memory depends
> on how many shards it owns; [section 4.5](#45-history-memory-and-cpu-follow-shard-count) gives
> those numbers.

**Across the whole test cluster, each namespace added about 1.8 MB of live heap.** At 2
namespaces, all eight service instances together held about 370 MB of live heap. At 1.8 MB per
namespace, namespaces add that much again by about 200 namespaces, so above that, most of the test
cluster's live heap was there because of its namespaces. With more shards or service instances,
that point comes later.

> **Recommendation: set pod memory limits from resident memory, not from live heap, and measure your
> own pods.** A pod's memory limit is compared against resident memory. Resident memory was measured
> at only two namespace counts, and it is not a fixed multiple of live heap, so the table above
> gives no per-namespace number for it. Start from the **Resident memory, 2 namespaces** column for
> a cluster with a few namespaces, and from **Resident memory, 500 namespaces** for one with
> hundreds; then measure your pods at your own namespace count, as [section
> 7.3](#73-what-each-panel-on-the-resting-load-row-should-show) describes.

> **Recommendation: add worker service, internal frontend and matching instances as namespace count
> grows.** These three services hold the per-namespace memory and CPU, and adding instances of each
> spreads it:
>
> - **Worker service** — each namespace's internal worker runs on one worker instance, which the
>   server picks from the namespace's ID; when you add worker instances, the server moves some
>   namespaces to the new ones.
> - **Internal frontend** — the polls are spread evenly across internal frontend instances
>   ([section 5.2](#52-the-per-namespace-worker-and-its-polls)).
> - **Matching service** — each task queue partition is owned by one matching instance; adding
>   matching instances spreads the partitions across more of them.
>
> Adding instances divides this memory and CPU among more pods; it does not reduce the total.

**Namespace memory is now covered.** [Section
5.4](#54-worker-internal-frontend-and-matching-cpu-per-namespace) covers CPU, the other resource
namespace count raises.

### 5.4 Worker, internal frontend and matching CPU per namespace

**Each namespace adds CPU to the same three services it adds memory to: the worker service, the
internal frontend and the matching service.** Measured on the test cluster as 20-minute averages,
each service instance freshly started at 2 and again at 500 namespaces. Every number is the CPU
one service instance used, as a percentage of one core:

| Service instance | CPU, 2 namespaces (% of one core) | CPU, 500 namespaces (% of one core) |
|---|---:|---:|
| Worker (the only one) | 0.7 | 3.5 |
| Internal frontend (the only one) | 0.7 | 4.5 |
| Matching, first instance | 1.0 | 3.7 |
| Matching, second instance | 1.0 | 4.3 |
| Frontend, both instances | 0.8 | 0.8 |
| History, both instances | 3.6 | 3.1 and 3.2 |

All Temporal services together went from 12.3 to 23.8 percent of one core.

**The added CPU comes from the per-namespace worker and its polls, not from the namespace list
read.** Every service instance reads the namespace list ([section
3.1](#31-database-calls-each-service-instance-makes-at-rest)), but the public frontends and
history, which do nothing else for each namespace, did not rise. Only the three services on the
poll path in [section 5.2](#52-the-per-namespace-worker-and-its-polls) did.

**Compare services, not their CPU numbers.** The CPU numbers in the table above depend on the test
machine's core speed and core count. Which services rise with namespace count, and which do not,
is what to look for on your own cluster.

> **Recommendation: when planning for more namespaces, plan extra CPU for the worker service, the
> internal frontend and the matching service only.** The public frontends and history needed no
> extra CPU for namespaces. Read those three services' CPU per pod from your container metrics as
> namespaces are added, and add instances as the recommendation in [section
> 5.3](#53-worker-internal-frontend-and-matching-memory-per-namespace) describes.

**Namespace CPU is now covered.** [Section 5.5](#55-metric-series-each-namespace-adds) covers what
namespaces add to monitoring the cluster: metric series.

### 5.5 Metric series each namespace adds

**Metric series add load to your monitoring stack, not to the cluster.** They make no database
calls, and the memory a service instance uses to hold its own series is already inside the live
heap in [section 5.3](#53-worker-internal-frontend-and-matching-memory-per-namespace). What they
add is work for whatever scrapes and stores them, such as Prometheus.

The server labels many of its metrics with the namespace, and with the task queue and partition,
so every namespace adds series to whatever scrapes the cluster. **Measured on the test cluster at
server defaults, each namespace added about 1,060 series across the cluster**, almost all of them
in the matching service, the internal frontend and the worker service:

| Service | Series per service instance at 2 namespaces | At 500 namespaces |
|---|---:|---:|
| Matching | about 2,000 | 122,000 and 147,000 |
| Internal frontend | about 1,300 | 158,000 |
| Worker | about 1,200 | 104,000 |
| Frontend and history | about 600 to 2,100 | about 800 for the frontends, about 2,100 for history |

**[Metric Series Scraped per Service](../observability/dashboards/server/temporal-server-readme.md#24-resting-load-idle-baseline)**
shows the count for your own cluster, from Prometheus's own `scrape_samples_scraped` metric. On
the test cluster, whose monitoring host was small, an earlier run with several hundred namespaces
ran Prometheus out of memory while every Temporal service stayed healthy — a reminder that the
monitoring stack can reach its limit before the cluster does.

> **Recommendation: watch your monitoring stack's memory as you add namespaces.** Each namespace
> adds about 1,060 series, so whatever scrapes and stores the cluster's metrics grows with
> namespace count. Track the series count on
> **[Metric Series Scraped per Service](../observability/dashboards/server/temporal-server-readme.md#24-resting-load-idle-baseline)**
> and the memory of your Prometheus (or other metrics store) together, and size it from the series
> count rather than from the number of namespaces.

> **Recommendation: turn the per-task-queue and per-partition labels off by default, and back on
> only where you need them.** `metrics.breakdownByTaskQueue` puts the real task queue name on
> matching and history metrics, and `metrics.breakdownByPartition` puts the real partition on
> matching metrics. The two settings are `true` by default, and each can be set per namespace and
> per task queue. Turning them off cluster-wide and on only for the namespaces or task queues
> you want task-queue-level detail for keeps that detail where it is used:
>
> ```yaml
> metrics.breakdownByTaskQueue:
>   - value: false                  # off for every namespace and task queue
>   - value: true                   # on for this namespace
>     constraints:
>       namespace: 'your-namespace'
> metrics.breakdownByPartition:
>   - value: false
>   - value: true
>     constraints:
>       namespace: 'your-namespace'
> ```
>
> Where they are off, the per-task-queue backlog gauges are not emitted, so per-task-queue panels —
> such as those used in [Changing Task Queue Partitions](./change-task-queue-partitions.md) — are
> empty for those namespaces. Compare
> **[Metric Series Scraped per Service](../observability/dashboards/server/temporal-server-readme.md#24-resting-load-idle-baseline)**
> before and after the change to see how many series it saves.

**Metric series are now covered, for namespaces that are still registered.** [Section
5.6](#56-deleted-namespaces-memory-and-metric-series-freed-by-restarting-the-worker-internal-frontend-and-matching-services)
covers what happens when namespaces are deleted: part of their memory, and none of their metric
series, is freed until the worker, internal frontend and matching instances restart.

### 5.6 Deleted namespaces' memory and metric series, freed by restarting the worker, internal frontend and matching services

**Deleting namespaces frees only part of their memory until the worker, internal frontend and
matching instances restart.** Measured on the test cluster, on the same processes, ten minutes
after deleting 498 empty namespaces and forcing garbage collection:

| Service | Share of its namespace-driven live heap freed |
|---|---:|
| Worker | about 86 % |
| Internal frontend | about 62 % |
| Matching | about 30 % |

**The metric series for deleted namespaces stay in the output too**, until the process restarts.
On the test cluster the worker service's series count went up, not down, after the delete.

> **Recommendation: after deleting many namespaces, do a rolling restart of the worker service, the
> internal frontend and the matching service.** A freshly started service instance holds only what
> the remaining namespaces need. Until the restart, most of the matching
> service's memory, part of the internal frontend's, and all of the metric series stay; that is
> expected, not a fault.

Deleting a namespace also adds database calls while the deletion runs; [Namespace Deletion —
System Workflow](./namespace-deletion.md#8-deleting-more-than-one-namespace-at-a-time) covers how
many and for how long.

**Deleted namespaces are now covered.** [Section 5.7](#57-settings-for-the-per-namespace-work)
lists the dynamic config settings that change what each namespace adds.

### 5.7 Settings for the per-namespace work

The table below lists the dynamic config settings that change what each namespace adds: each
default, what changing it does and what 0 does. These effects come from the server's code and were
not measured; if you change one of these settings, measure your resting load again with [section
7](#7-measuring-the-baseline-of-a-cluster-you-already-run).

| Setting | Default | What changing it does | What `0` does |
|---|---|---|---|
| `worker.perNamespaceWorkerCount` | 1 | Internal workers per namespace. Raising it spreads a namespace's worker over more worker instances and multiplies its polls. Can be set per namespace. | Stops the namespace's internal worker; schedules, batch operations and Worker Deployment stop working in that namespace. |
| `worker.perNamespaceWorkerOptions` | 2 workflow and 2 activity pollers | Pollers per internal worker. Can be set per namespace. | Raised to 2, the minimum. |
| `system.namespaceCacheRefreshInterval` | 2s | How often every service instance re-reads the namespace list. Raising it means fewer reads, but a namespace change takes longer to reach every service instance. | Every service instance reads the namespace list without ever pausing. |
| `matching.getUserDataRefresh` | 5m | How often the matching service re-reads each task queue's settings (`GetTaskQueueUserData`). | The matching service reads those settings without ever pausing. |
| `worker.perNamespaceWorkerStartRate` | 10 per second | How fast each worker instance starts internal workers. Affects start-up time only, not resting load. | Not determined; leave at the default. |
| `metrics.breakdownByTaskQueue`, `metrics.breakdownByPartition` | true | Turned off, they cut metric series ([section 5.5](#55-metric-series-each-namespace-adds)). | Not applicable (true or false). |

> **Recommendation: never set `system.namespaceCacheRefreshInterval` or
> `matching.getUserDataRefresh` to 0.** At 0, every service instance reads the namespace list
> nonstop, and the matching service reads every task queue's settings nonstop. To make either
> read less frequent, raise that setting's interval instead.

> **Recommendation: set `worker.perNamespaceWorkerCount` to 0 only for a namespace that will never
> use schedules, batch operations or Worker Deployment.** It is the only way to remove a
> namespace's share of the resting load short of deleting the namespace, and it stops those
> features in that namespace.

**Next:** each namespace adds a small, straight-line database cost and a larger memory and CPU
cost on the worker, internal frontend and matching. With service instances, shards and namespaces
all covered, what is left is the load that appears on a resting cluster without being part of its
baseline — [section 6](#6-database-calls-from-restarts-and-periodic-jobs).

---

## 6. Database calls from restarts and periodic jobs

For limited periods, the cluster makes extra database calls: after a full cluster restart, during
the twice-daily history and task queue scans, and from one other periodic job. They happen with or
without workload, so they matter twice:

- **When sizing a cluster**, plan headroom for them on top of the baseline and the workload.
- **When measuring the baseline**, read outside these periods, so they are not counted in.

This section covers how many database calls each kind adds and for how long.

### 6.1 Database calls after a full cluster restart

**A full cluster restart adds a burst of database calls for about two minutes, then a wave every
five minutes that was still visible half an hour later.** Measured on the test cluster at 2048
shards and two namespaces, with every service instance restarted at once:

| Time after the restart | Database calls per second |
|---|---:|
| First minute | 653 — about twelve times the resting rate |
| Second minute | 118 |
| Minutes 2 to 4 | about 6 |
| From minute 4 | waves every five minutes, from about 6 to about 150 |
| Averaged over whole five-minute cycles, from minute 5 | 53 to 54 — the resting rate |

**The first-minute burst in the table above, 653 database calls per second, comes from starting up
every shard.** As each shard is taken over by a history instance, every one of its queues —
including the transfer, visibility and outbound queues that read nothing at rest — reads once and
records where it starts from, and the shard's ownership is recorded. That is a fixed amount of
work per shard, done once.

**The five-minute waves after the burst come from the timer queue's five-minute cycle.** Every
shard started its cycle at the same moment, so all of them read and delete together, then all wait
together. The cycles carry a small random spread, so they drift apart, but slowly: half an hour
after the restart, one-minute readings still swung between about a third and twice the true
average.

> **Recommendation: after a full cluster restart, wait at least ten minutes before reading the
> baseline, then read it over whole five-minute cycles or over 30 minutes.**
> **[Database Calls Total, 30-Minute Average](../observability/dashboards/server/temporal-server-readme.md#24-resting-load-idle-baseline)**
> does that; short-window panels such as
> **[Persistence Requests Total per Operation](../observability/dashboards/server/temporal-server-readme.md#3-persistence-requests-latencies-and-errors)**
> show the waves instead.
>
> If you create namespaces for a baseline test, create them first and then restart the service
> instances, as the test cluster did, so every service instance starts with the namespaces already
> in place; then wait and read as above.


**A rolling restart of the service instances adds a smaller burst.** Restarting one history
instance at a time moves and starts up only the shards that instance owns, and restarting the
other services starts up no shards at all. Rolling restarts were not measured, so read the
baseline after one the same way as after a full cluster restart.

**Database calls after a full cluster restart are now covered.** [Section
6.2](#62-history-and-task-queue-scans-twice-a-day) covers the history and task queue scans, which
the server runs at 00:00 and 12:00 UTC — a fixed time that no setting changes.

### 6.2 History and task queue scans, twice a day

The history and task queue scans ([point 9 of the background
work](#background-work-a-temporal-server-always-runs)) delete leftover workflow history, expired
tasks and task queues unused for two days. They run as internal workflows in the worker service,
at **00:00 and 12:00 UTC**. Which scans run depends on the primary store's database and on the
settings in the table below:

| Scan | Primary store database | Setting | Default |
|---|---|---|---|
| History scan | any | `worker.historyScannerEnabled` | on |
| Task queue scan | SQL only: PostgreSQL, MySQL or SQLite | `worker.taskQueueScannerEnabled` | on |
| Executions scan | Cassandra only | `worker.executionsScannerEnabled` | off |
| Build ID scan | any | `worker.buildIdScavengerEnabled` | off |

**The history scan reads at most `worker.scannerPersistenceMaxQPS` history branches per second —
100 by default.** Each old branch it reads can lead to more database calls, some of them made by
the history service, so the scan's database calls can go above 100 per second. **The task queue
scan has no limit of its own**: it works through up to 32 task queues at a time, held back only by
the worker service's overall database limit, `worker.persistenceMaxQPS` — 500 per second per
worker instance by default — which it shares with everything else the worker service does. What a
scan actually costs depends on how much it has to go through: the history scan pages through every
stored workflow history on the cluster, and the task queue scan through every task queue in the
database. The history scan's work grows with the stored histories, and the task queue scan's with
the number of task queues, including one per namespace.

**No scan was captured during testing**: every measurement was shorter than the twelve hours
between scans. So this section gives the scan schedule and its rate limit, not a measured size.
Two consequences hold all the same:

- **A rise in database calls that starts at 00:00 or 12:00 UTC is the scans, not a fault**, as
  long as it falls back to the resting rate once the scans finish.
- **A baseline reading leaves the scans out.** The scans start only every twelve hours, so a
  30-minute reading taken away from 00:00 and 12:00 UTC does not include them.

> **Recommendation: when sizing the database, plan headroom for the scans on top of the resting
> rate, for the hours they run.** The history scan reads up to 100 history branches per second, and
> each branch can lead to several more database calls. On a SQL database, the task queue scan can use
> up to `worker.persistenceMaxQPS`, 500 database calls per second by default, on the one worker
> instance that runs it. The resting rate includes neither.

**The history and task queue scans are now covered.** [Section
6.3](#63-dead-letter-queue-depth-counted-every-three-hours) covers one more job the server runs on
a long timer: counting the dead-letter queues.

### 6.3 Dead-letter queue depth, counted every three hours

**Every three hours, the server counts the tasks in each history task dead-letter queue.** This
job makes too few database calls to show in a resting rate, but it changes how you read one
dashboard panel. Shards are numbered from 1 up to the shard count, and only one history instance
does this count: the one that currently owns shard 1. It reports the counts as the
`dlq_message_count` gauge, which
**[Dead-Letter Queue Depth by Category](../observability/dashboards/server/temporal-server-readme.md#20-history-task-dlq--terminal-failures)**
shows.

> **Recommendation: read Dead-Letter Queue Depth by Category as a count taken every three hours.**
> A flat line between updates does not mean the depth stopped changing; `tdbg dlq list` gives the
> current count.

**Next:** every source of load on a resting cluster is now covered — the baseline in [section
3](#3-database-calls-memory-and-cpu-per-service-instance) to [section
5](#5-database-calls-memory-cpu-and-metric-series-per-namespace), and the extra database calls for
limited periods, here. [Section 7](#7-measuring-the-baseline-of-a-cluster-you-already-run) covers
measuring your own cluster's baseline, starting with what the test cluster measured.

---

## 7. Measuring the baseline of a cluster you already run

[Section 3](#3-database-calls-memory-and-cpu-per-service-instance) to [section
6](#6-database-calls-from-restarts-and-periodic-jobs) measured each part of the baseline on its
own: service instances, shards, namespaces, and the periods with extra database calls. This
section puts those test cluster results together in one table, then gathers the recommendations
for running the same measurement on a cluster you already run into one checklist, what each panel
should show, and what to do when your reading is higher than expected. If the cluster does not
exist yet and you are sizing it, there is nothing to measure: [section
8](#8-calculating-the-baseline-of-a-cluster-you-are-planning) shows how to work out its baseline
from the same numbers instead.

### 7.1 Test cluster results

**Setup measured** ([section 1.2](#12-what-was-measured-and-on-which-test-cluster) has the full
setup):

- eight service instances on one machine: two history, two matching, two frontend, one internal
  frontend and one worker
- PostgreSQL, 2048 history shards, archival enabled
- every dynamic config setting at the server's default, on a server equivalent to v1.32.1
- no workflows running and no SDK workers connected

**How each reading was taken:**

- every service instance was restarted, and nothing was read for the first ten minutes
- database calls and CPU were averaged from minute 10 to minute 30 — four full five-minute cycles
- memory was read at minute 30: live heap right after forcing garbage collection, resident memory
  from the running processes

> **Note:** these numbers come from one test machine. Use them to see how large a baseline is and
> which parts of it grow, not as your own cluster's numbers: measure your own with the checklist in
> [section 7.2](#72-checklist-for-a-clean-baseline-reading).

**Totals for the whole test cluster, at rest:**

| Resource | 2 namespaces, archival enabled | 500 namespaces, archival enabled | 500 namespaces, archival turned off |
|---|---:|---:|---:|
| Database calls per second | 53.6 | 60.4 | 38.2 |
| Live heap, all eight service instances together | about 370 MB | about 1,250 MB | about 1,180 MB |
| Resident memory, all eight service instances together | about 1,360 MB | about 2,530 MB | about 2,390 MB |
| CPU, all Temporal services together (% of one core) | 12.3 | 23.8 | 23.6 |
| Metric series | about 12,000 | about 540,000 | not counted |

The **500 namespaces, archival turned off** column was read earlier than the others: database
calls and CPU over minutes 10 to 20, and memory at minute 20; [section
4.3](#43-archival-enabled-in-the-static-configuration) covers turning archival off.

**Per service instance, archival enabled, at 2 namespaces:**

| Service instance | Database calls per second | Live heap | Resident memory | CPU (% of one core) |
|---|---:|---:|---:|---:|
| History, first instance | 24.9 | 124 MB | 386 MB | 3.7 |
| History, second instance | 24.1 | 124 MB | 396 MB | 3.6 |
| Matching, first instance | 1.2 | 20 MB | 102 MB | 1.0 |
| Matching, second instance | 0.9 | 18 MB | 93 MB | 1.0 |
| Frontend, first instance | 0.6 | 17 MB | 88 MB | 0.8 |
| Frontend, second instance | 0.6 | 15 MB | 83 MB | 0.8 |
| Internal frontend (only one) | 0.6 | 20 MB | 96 MB | 0.7 |
| Worker service (only one) | 0.6 | 32 MB | 112 MB | 0.7 |

**The same, at 500 namespaces:**

| Service instance | Database calls per second | Live heap | Resident memory | CPU (% of one core) |
|---|---:|---:|---:|---:|
| History, first instance | 24.1 | 117 MB | 334 MB | 3.1 |
| History, second instance | 25.1 | 122 MB | 428 MB | 3.2 |
| Matching, first instance | 4.1 | 118 MB | 351 MB | 3.7 |
| Matching, second instance | 4.6 | 140 MB | 372 MB | 4.3 |
| Frontend, first instance | 0.6 | 14 MB | 56 MB | 0.8 |
| Frontend, second instance | 0.6 | 14 MB | 57 MB | 0.8 |
| Internal frontend (only one) | 0.6 | 156 MB | 409 MB | 4.5 |
| Worker service (only one) | 0.6 | 574 MB | 527 MB | 3.5 |

**What the per-service tables show:**

- **The history service makes most of the database calls**: 49 of 53.6 per second at 2 namespaces,
  and 49 of 60.4 at 500. They come from the shards ([section
  4](#4-database-calls-memory-and-cpu-from-history-shards)), so shard count, not namespace count,
  sets most of a resting cluster's database load.
- **Every other service instance makes about 0.6 database calls per second**, except the matching
  service, whose instances rose from about 1 to between 4 and 5 at 500 namespaces.
- **Namespaces drive memory, and the worker service most of all**: its live heap grew from 32 MB
  to 574 MB, the internal frontend's from 20 MB to 156 MB, and each matching instance's from about
  20 MB to between 118 and 140 MB. History and the public frontends did not grow ([section
  5](#5-database-calls-memory-cpu-and-metric-series-per-namespace)).
- **The public frontends used no more memory or CPU at 500 namespaces than at 2** — about 14 to 17
  MB of live heap and 0.8% of one core each — because the per-namespace polls went through the
  internal frontend. On a cluster without an internal frontend, those polls go through the public
  frontends that serve your clients instead ([section
  5.2](#52-the-per-namespace-worker-and-its-polls)).
- **Turning off unused archival was the largest saving measured**: 60.4 to 38.2 database calls per
  second at 500 namespaces ([section 4.3](#43-archival-enabled-in-the-static-configuration)).
- **CPU at rest stays small**: on the test machine, no service instance used more than 4.5% of one
  core, even at 500 namespaces.

**Test cluster results are now covered.** [Section
7.2](#72-checklist-for-a-clean-baseline-reading) gathers the recommendations from earlier sections
into one checklist for taking a clean baseline reading on your own cluster.

### 7.2 Checklist for a clean baseline reading

The recommendations from earlier sections, in the order you apply them:

1. **Make the cluster idle**, as [section 1](#1-what-counts-as-an-idle-temporal-cluster) defines
   it: every service instance up, no workflows running, no SDK workers connected.
2. **Register your namespaces, then restart the service instances**, so every service instance
   starts with the namespaces in place, as the test cluster did.
3. **Wait at least ten minutes after the restart, and better thirty**, for the burst and the waves
   in [section 6.1](#61-database-calls-after-a-full-cluster-restart) to settle.
4. **Pick a window that does not cross 00:00 or 12:00 UTC**, when the history and task queue scans
   in [section 6.2](#62-history-and-task-queue-scans-twice-a-day) may run.
5. **Read the
   [Resting Load (Idle Baseline)](../observability/dashboards/server/temporal-server-readme.md#24-resting-load-idle-baseline)
   row of the server dashboard.** Its 30-minute panels average over whole five-minute cycles for
   you; [section 7.3](#73-what-each-panel-on-the-resting-load-row-should-show) says what each
   panel should show. Read resident memory and CPU per pod from your container metrics.
6. **Compare your reading with the calculation for your cluster** in [section
   8](#8-calculating-the-baseline-of-a-cluster-you-are-planning). If the reading is higher,
   [section 8.6](#86-explaining-a-measured-baseline-above-its-calculated-level) lists the causes.

**A cluster carrying workload cannot show its baseline directly**, because the timer queue, the
shard progress writes and the matching service's reads all grow with workload too, and nothing on
the dashboard separates the resting part from the rest. To know a production cluster's baseline,
measure it before workload arrives — on the cluster itself before go-live, or on another cluster
with the same shard count, service instances, namespaces and settings — or calculate it with
[section 8](#8-calculating-the-baseline-of-a-cluster-you-are-planning).

**The checklist for a clean baseline reading is now covered.** [Section
7.3](#73-what-each-panel-on-the-resting-load-row-should-show) says what each panel on the Resting
Load row should show.

### 7.3 What each panel on the Resting Load row should show

| Panel | What a resting cluster shows | Compare with |
|---|---|---|
| **[Database Calls Total, 30-Minute Average](../observability/dashboards/server/temporal-server-readme.md#24-resting-load-idle-baseline)** | A flat line, close to the calculated rate | [Section 8.1](#81-database-calls-at-rest) |
| **[Database Calls per Operation, 30-Minute Average](../observability/dashboards/server/temporal-server-readme.md#24-resting-load-idle-baseline)** | The operations listed under this table, at about their listed rates | [Section 4.1](#41-what-each-shard-does-at-rest), [section 3.1](#31-database-calls-each-service-instance-makes-at-rest), [section 5.1](#51-database-calls-each-namespace-adds) |
| **[Namespaces Registered](../observability/dashboards/server/temporal-server-readme.md#24-resting-load-idle-baseline)** (`total_namespaces`) | The same number on every service | Your own namespace count |
| **[Namespace Registry Refresh Latency](../observability/dashboards/server/temporal-server-readme.md#24-resting-load-idle-baseline)** | Milliseconds, far below the 2-second line | [Section 5.1](#51-database-calls-each-namespace-adds) |
| **[Metric Series Scraped per Service](../observability/dashboards/server/temporal-server-readme.md#24-resting-load-idle-baseline)** | Matching, internal frontend and worker well above history and the public frontends once there are more than a few namespaces | [Section 8.3](#83-metric-series) |

**Operations on Database Calls per Operation at rest**, at the server's default settings:

| Operation | Made by | Rate at rest |
|---|---|---|
| `GetTimerTasks` | History | 2 per shard every five minutes |
| `RangeCompleteTimerTasks` | History | 1 per shard every five minutes |
| `UpdateShard` | History | 1 per shard every five minutes |
| `GetArchivalTasks`, `RangeCompleteArchivalTasks` | History | The same as `GetTimerTasks` and `RangeCompleteTimerTasks`, only when archival is enabled |
| `ListNamespaces` | Every service instance | 0.5 per second each |
| `UpsertClusterMembership` | Every service instance | About 0.08 per second each |
| `ListClusterMetadata` | Every service instance | Once a minute each |
| `GetTaskQueue` | Matching | About 0.75 per second, plus 3 per namespace every five minutes |
| `GetTaskQueueUserData` | Matching | 1 per namespace every five minutes |
| `ListNexusEndpoints` | One matching instance | 0.1 per second |

Small rates of `GetTasks`, `UpdateTaskQueue`, `GetClusterMetadata` and
`GetCurrentClusterMetadata`, around 0.02 per second or less, are also normal.

**`GetTransferTasks`, `GetVisibilityTasks` and `GetOutboundTasks` should not appear at all on a
resting cluster.** Those queues read only when a workflow has written a task for them ([section
4.2](#42-transfer-visibility-outbound-and-replication-queues-at-rest)), so seeing them means the
cluster is not idle.

**Reading resident memory and CPU per service instance, and two panels on older servers:**

- **Resident memory and CPU are not on the dashboard**, because the server does not export
  resident memory. Read both per pod from your container metrics.
- **[Heap per Service Instance, 10-Minute Low](../observability/dashboards/server/temporal-server-readme.md#24-resting-load-idle-baseline)**
  shows each service instance's Go heap. In testing, its low point read 0 to 28 percent above live
  heap, close enough to compare with the live-heap numbers in [section
  8.2](#82-memory-per-service-instance-at-rest) and [section
  5.3](#53-worker-internal-frontend-and-matching-memory-per-namespace).
- **Namespaces Registered** and **Namespace Registry Refresh Latency** stay empty on servers
  before v1.31.0, which do not emit the namespace count or the refresh time.

**What a resting cluster's panels should show is now covered.** [Section
7.4](#74-re-taking-a-baseline-after-the-cluster-changes) covers how the resting load changes
slowly as a cluster ages; if a reading is higher than the calculation in [section
8](#8-calculating-the-baseline-of-a-cluster-you-are-planning), [section
8.6](#86-explaining-a-measured-baseline-above-its-calculated-level) explains why.

### 7.4 Re-taking a baseline after the cluster changes

**A baseline is a snapshot.** It moves whenever one of the inputs it depends on changes. Shard
count is the only input that never changes; everything else does over a cluster's life:

| What changes | What moves | Where to see it |
|---|---|---|
| Namespaces added | The matching service's database calls, and the memory, CPU and metric series of the worker service, the internal frontend and the matching service ([section 5](#5-database-calls-memory-cpu-and-metric-series-per-namespace)) | **[Namespaces Registered](../observability/dashboards/server/temporal-server-readme.md#24-resting-load-idle-baseline)**, **[Heap per Service Instance, 10-Minute Low](../observability/dashboards/server/temporal-server-readme.md#24-resting-load-idle-baseline)**, **[Metric Series Scraped per Service](../observability/dashboards/server/temporal-server-readme.md#24-resting-load-idle-baseline)** |
| Namespaces deleted | Memory and metric series stay near their old level until those services restart ([section 5.6](#56-deleted-namespaces-memory-and-metric-series-freed-by-restarting-the-worker-internal-frontend-and-matching-services)) | **[Heap per Service Instance, 10-Minute Low](../observability/dashboards/server/temporal-server-readme.md#24-resting-load-idle-baseline)**, **[Metric Series Scraped per Service](../observability/dashboards/server/temporal-server-readme.md#24-resting-load-idle-baseline)** |
| A large namespace list | Each namespace list refresh takes longer ([section 5.1](#51-database-calls-each-namespace-adds)) | **[Namespace Registry Refresh Latency](../observability/dashboards/server/temporal-server-readme.md#24-resting-load-idle-baseline)** |
| Service instances added | About 0.6 database calls per second each; per-namespace memory and CPU spread across more pods ([section 3](#3-database-calls-memory-and-cpu-per-service-instance)) | **[Database Calls Total, 30-Minute Average](../observability/dashboards/server/temporal-server-readme.md#24-resting-load-idle-baseline)** |
| A setting changed | The database calls it controls ([section 3.3](#33-settings-for-service-instance-and-cluster-wide-database-calls-at-rest), [section 4.4](#44-settings-for-how-often-each-shard-reads-and-writes), [section 5.7](#57-settings-for-the-per-namespace-work)) | **[Database Calls per Operation, 30-Minute Average](../observability/dashboards/server/temporal-server-readme.md#24-resting-load-idle-baseline)** |
| Archival turned on or off | The archival queue's database calls on every shard ([section 4.3](#43-archival-enabled-in-the-static-configuration)) | **[Database Calls per Operation, 30-Minute Average](../observability/dashboards/server/temporal-server-readme.md#24-resting-load-idle-baseline)** |

> **Recommendation: re-take the baseline after any change in the table above**, using the checklist
> in [section 7.2](#72-checklist-for-a-clean-baseline-reading), and keep each reading next to the
> change that caused it. Act on a rising
> **[Namespace Registry Refresh Latency](../observability/dashboards/server/temporal-server-readme.md#24-resting-load-idle-baseline)** long before it nears the 2-second line, by planning
> memory and monitoring capacity for the namespace count it reflects.

**Re-taking a baseline after changes is now covered: changes are something to plan for, not to be
paged for.** [Section 7.5](#75-why-resting-load-has-no-alert) explains why resting load has no
alert.

### 7.5 Why resting load has no alert

**No alert ships with this playbook, on purpose.** The resting load is the cluster's normal steady
state: a level, set by its configuration, that only moves when an input moves. A page for the
steady state would fire on every healthy cluster.

The one signal that would justify an alert is
**[Namespace Registry Refresh Latency](../observability/dashboards/server/temporal-server-readme.md#24-resting-load-idle-baseline)**
approaching `system.namespaceCacheRefreshInterval` (2 seconds by default): once a refresh takes
nearly as long as the time between refreshes, every service instance is spending most of its time
refreshing the namespace list. No alert is provided for it: the test cluster gave no measured
refresh time anywhere near the 2-second line, so there was no measured point at which to set a
threshold.

**Next:** measuring your own cluster's baseline is now covered. [Section
8](#8-calculating-the-baseline-of-a-cluster-you-are-planning) covers calculating the baseline of a
cluster that does not exist yet, from its service instance, shard and namespace counts.

---

## 8. Calculating the baseline of a cluster you are planning

This section covers how to calculate what a planned cluster will use at rest from how many service
instances, shards and namespaces it will have. It puts the per-service-instance, per-shard and
per-namespace numbers from [section 3](#3-database-calls-memory-and-cpu-per-service-instance),
[section 4](#4-database-calls-memory-and-cpu-from-history-shards) and [section
5](#5-database-calls-memory-cpu-and-metric-series-per-namespace) into one calculation, checks the
calculation against what the test cluster measured, and works one example through. Every input is
at the server's defaults; a cluster with lowered intervals needs the multipliers from [section
4.4](#44-settings-for-how-often-each-shard-reads-and-writes) applied first.

### 8.1 Database calls at rest

Add four parts:

| Part | Database calls per second | From |
|---|---|---|
| Service instances | **0.6 × service instances**, plus 0.5 per service instance for every further thousand namespaces above 1,000 | [Section 3.1](#31-database-calls-each-service-instance-makes-at-rest) |
| Once for the cluster | **0.85** — the Nexus check and the worker service's own task queues | [Section 3.2](#32-database-calls-made-once-for-the-whole-cluster) |
| History shards | **shards × 7 ÷ 300** with archival enabled, **shards × 4 ÷ 300** without | [Section 4.1](#41-what-each-shard-does-at-rest) |
| Namespaces | **namespaces × 4 ÷ 300** | [Section 5.1](#51-database-calls-each-namespace-adds) |

**Adding the service-instance, cluster, shard and namespace parts reproduces every measurement
taken in testing to within a few percent:**

| Test cluster | Calculated database calls per second | Measured database calls per second |
|---|---:|---:|
| 8 service instances, 2048 shards, 2 namespaces, archival enabled | 53.5 | 53.6 |
| 8 service instances, 2048 shards, 500 namespaces, archival enabled | 60.1 | 60.4 |
| 8 service instances, 2048 shards, 500 namespaces, archival disabled | 39.6 | 38.2 |
| 8 service instances, 512 shards, 2 namespaces, archival enabled | 17.6 | 17.5 |

The archival-disabled measurement was taken over a ten-minute window, shorter than the others,
which is why it is furthest from the calculation.

**The shards make most of the calculated database calls.** On the test cluster, the shards gave
47.9 of the 53.6 measured database calls per second; the eight service instances gave 4.8, the
cluster-wide reads 0.85, and two namespaces less than 0.1. So get the shard count and whether
archival is enabled right first: the other parts move the result much less, unless the cluster has
thousands of namespaces — 1,000 namespaces add about 13 database calls per second.

**The database-call calculation is now covered.** [Section
8.2](#82-memory-per-service-instance-at-rest) covers memory, which can be calculated in live heap,
with a caveat for resident memory.

### 8.2 Memory per service instance at rest

**Live heap** can be added up per service instance, and checked on a running cluster with
**[Heap per Service Instance, 10-Minute Low](../observability/dashboards/server/temporal-server-readme.md#24-resting-load-idle-baseline)**:

| Service | Live heap per service instance |
|---|---|
| History | about **20 MB + 0.1 MB per shard that history instance owns**, worked out from 512 and 2048 shards; outside that range, measure ([section 4.5](#45-history-memory-and-cpu-follow-shard-count)) |
| Worker | about **32 MB + 1.09 MB per namespace whose internal worker runs on that worker instance** |
| Internal frontend | about **20 MB + 0.27 MB per namespace ÷ internal frontend instances** |
| Matching | about **19 MB + 0.44 MB per namespace ÷ matching instances** |
| Frontend | about **17 MB**, not changed by namespaces when an internal frontend is deployed |

How to use the table above for one service instance:

- **History:** the cluster's shards are shared out about evenly among the history instances, and
  each shard is handled by one history instance at a time. So divide the shard count by the number
  of history instances, then use that number in the history row. On the test cluster, 2048 shards
  over 2 history instances is about 1,024 each: 20 + 0.1 × 1,024 is about 122 MB, against 124 MB
  measured.
- **Worker service:** each namespace's internal worker runs on one worker instance. Divide the
  namespace count by the number of worker instances, assuming an even spread, and use that number
  in the worker row. On the test cluster, 500 namespaces on its one worker instance: 32 + 1.09 ×
  500 is about 577 MB, against 574 MB measured.

> **Recommendation: size every worker pod for the largest worker instance, not the average.**
> Namespaces are placed on worker instances by their IDs, so the spread need not be even; read
> each worker instance on
> **[Heap per Service Instance, 10-Minute Low](../observability/dashboards/server/temporal-server-readme.md#24-resting-load-idle-baseline)**
> and size from the highest.

> **Recommendation: use the live-heap calculation to compare services and see what grows, and set
> pod memory limits from resident memory measured on your own cluster.** A pod's memory limit is
> compared against resident memory, which is not a fixed multiple of live heap: on the test cluster
> it ranged from about equal to live heap to about six times it. Until you can measure, start from
> the resident memory columns in [section 7.1](#71-test-cluster-results).

**The memory calculation is now covered.** [Section 8.3](#83-metric-series) gives the number for
sizing the monitoring stack: metric series.

### 8.3 Metric series

Add two parts:

| Part | Metric series | From |
|---|---:|---|
| Base, for eight service instances at 2 namespaces | about 12,000 | [Section 5.5](#55-metric-series-each-namespace-adds) |
| Each namespace | about 1,060, almost all on the matching service, the internal frontend and the worker service | [Section 5.5](#55-metric-series-each-namespace-adds) |

For example, 100 namespaces come to about 12,000 + 1,060 × 100, or about 118,000 series; 500
namespaces come to about 542,000, against about 540,000 measured on the test cluster.

> **Recommendation: size your metrics store from this count, and check it after scaling.** Read
> the actual count on
> **[Metric Series Scraped per Service](../observability/dashboards/server/temporal-server-readme.md#24-resting-load-idle-baseline)**
> after scaling the matching service, the internal frontend or the worker service. The 1,060 per
> namespace was measured with eight service instances. If the count is too high, turn off the
> per-task-queue and per-partition labels as [section 5.5](#55-metric-series-each-namespace-adds)
> recommends.

**The calculations for database calls, memory and metric series are now covered.** [Section
8.4](#84-calculating-the-baseline-of-an-example-cluster-step-by-step) runs them for one cluster.

### 8.4 Calculating the baseline of an example cluster, step by step

This example puts [section 8.1](#81-database-calls-at-rest) to [section 8.3](#83-metric-series)
together for one planned cluster, one step at a time. The cluster is made up to show the
arithmetic; its numbers come only from this section.

**The planned cluster:** 3 history, 3 matching, 2 frontend, 2 internal frontend and 2 worker
instances — 12 service instances — with 4096 history shards and 800 namespaces, and archival
disabled.

**Database calls at rest:**

| Part | Arithmetic | Database calls per second |
|---|---|---:|
| Service instances | 12 × 0.6 (800 namespaces is under a thousand) | 7.2 |
| Once for the cluster | | 0.85 |
| History shards, archival disabled | 4096 × 4 ÷ 300 | 54.6 |
| Namespaces | 800 × 4 ÷ 300 | 10.7 |
| **Total** | | **about 73** |

With archival enabled, the shard part becomes 4096 × 7 ÷ 300 = 95.6, and the total about 114 — the
largest single difference any one setting makes to this cluster.

**Live heap per service instance:**

| Service | Arithmetic | Live heap |
|---|---|---:|
| History, each of 3 | 20 + 0.1 × (4096 ÷ 3) | about 157 MB |
| Worker, each of 2 | 32 + 1.09 × (800 ÷ 2) | about 468 MB |
| Internal frontend, each of 2 | 20 + 0.27 × 800 ÷ 2 | about 128 MB |
| Matching, each of 3 | 19 + 0.44 × 800 ÷ 3 | about 136 MB |
| Frontend, each of 2 | | about 17 MB |

The worker instances carry the most memory per service instance, then history, which is the
pattern to expect whenever namespace count is in the hundreds. Resident memory on each pod will be
larger; [section 8.2](#82-memory-per-service-instance-at-rest) says why no single factor converts
live heap to resident memory.

**Metric series:** 12,000 + 800 × 1,060 — **about 860,000**, to plan Prometheus capacity from.

**The worked example is now complete.** [Section 8.5](#85-what-the-arithmetic-does-not-cover)
names what the arithmetic leaves out.

### 8.5 What the arithmetic does not cover

- **CPU.** No per-service-instance, per-shard or per-namespace CPU number is given, because CPU
  depends on core speed and core count. [Section
  5.4](#54-worker-internal-frontend-and-matching-cpu-per-namespace) gives the shape: history at
  rest follows shards, namespaces raise the worker, the internal frontend and matching.
- **Extra database calls for limited periods.** Restarts and the twice-daily scans in [section
  6](#6-database-calls-from-restarts-and-periodic-jobs) come on top, for as long as they last.
- **Visibility.** The visibility store is not part of any number here.
- **Anything not measured here** — more than 500 namespaces, a primary store database other than
  PostgreSQL, no internal frontend, more than one internal worker per namespace, or a server
  before v1.19. Measure your own cluster with [section
  7](#7-measuring-the-baseline-of-a-cluster-you-already-run).

**What the arithmetic does not cover is now covered.** [Section
8.6](#86-explaining-a-measured-baseline-above-its-calculated-level) covers what explains a
measured baseline that is higher than the calculation.

### 8.6 Explaining a measured baseline above its calculated level

With a baseline calculated in this section and one measured with [section
7](#7-measuring-the-baseline-of-a-cluster-you-already-run), the two should be close. When the
measured baseline is higher, the table below lists the usual causes. Work down it; each line names
what to look at on
**[Database Calls per Operation, 30-Minute Average](../observability/dashboards/server/temporal-server-readme.md#24-resting-load-idle-baseline)**
and what it means.

| What the panel shows | What it means | Where to go |
|---|---|---|
| Every operation high, and the line still falling | The cluster restarted within the last half hour | Read again later — [section 6.1](#61-database-calls-after-a-full-cluster-restart) |
| `GetTimerTasks` and `RangeCompleteTimerTasks` above shards × 2 ÷ 300 and shards × 1 ÷ 300 | `history.timerProcessorMaxPollInterval` is below its 5-minute default | [Section 4.4](#44-settings-for-how-often-each-shard-reads-and-writes) |
| `UpdateShard` above shards ÷ 300 | `history.shardUpdateMinInterval` is below its 5-minute default, or the cluster is not idle | [Section 4.4](#44-settings-for-how-often-each-shard-reads-and-writes) |
| `GetArchivalTasks` present when archival was assumed off | Archival is enabled in the static configuration | [Section 4.3](#43-archival-enabled-in-the-static-configuration) |
| `GetTransferTasks`, `GetVisibilityTasks` or `GetOutboundTasks` present | Something is writing tasks — workflows are running, so the cluster is not idle | [Section 1](#1-what-counts-as-an-idle-temporal-cluster) |
| `ListNamespaces` above 0.5 × service instances | More than a thousand namespaces, more service instances than assumed, or `system.namespaceCacheRefreshInterval` below 2 seconds | [Section 3.3](#33-settings-for-service-instance-and-cluster-wide-database-calls-at-rest) |
| `GetTaskQueue` far above 0.75 + namespaces × 3 ÷ 300 | More task queues are loaded than the internal ones — application workers are polling — or `matching.updateAckInterval` is lowered | [Section 3.3](#33-settings-for-service-instance-and-cluster-wide-database-calls-at-rest) |

**Every cause in the table above is something you can check and change.** A measured baseline
above the calculation rarely points to a problem in the server; it is almost always a recent
restart of the service instances, a changed setting, or a cluster that is not quite idle. Check
them in that order: the time since the service instances last restarted, your dynamic config
against the defaults in [section 9.1](#91-dynamic-config), and the `archival:` block of the static
configuration. Once those match what the calculation assumed, the measured and calculated
baselines should agree.

**Next:** every setting named in this playbook has appeared in the section that owns it. [Section
9](#9-settings-and-metrics-reference) collects them in one table.

---

## 9. Settings and metrics reference

Every setting and metric this playbook names, for looking up. What each setting does at rest, and
what to do with it, is in the section its row links to. For the settings that must never be set to
0 — listed in the recommendations in [section
3.3](#33-settings-for-service-instance-and-cluster-wide-database-calls-at-rest), [section
4.4](#44-settings-for-how-often-each-shard-reads-and-writes) and [section
5.7](#57-settings-for-the-per-namespace-work) — up to v1.32.1 the server neither rejects nor
corrects a 0 ([temporalio/temporal#12652](https://github.com/temporalio/temporal/issues/12652)).

### 9.1 Dynamic config

| Setting | Default | What it controls | Section |
|---|---|---|---|
| `history.timerProcessorMaxPollInterval` | 5m | How often each shard reads its timer queue | [4.4](#44-settings-for-how-often-each-shard-reads-and-writes) |
| `history.archivalProcessorMaxPollInterval` | 5m | How often each shard reads its archival queue | [4.4](#44-settings-for-how-often-each-shard-reads-and-writes) |
| `history.shardUpdateMinInterval` | 5m | How often each shard saves its progress | [4.4](#44-settings-for-how-often-each-shard-reads-and-writes) |
| `history.timerProcessorUpdateAckInterval`, `history.archivalProcessorUpdateAckInterval` | 30s | How often the timer and archival queues run their checkpoint | [4.4](#44-settings-for-how-often-each-shard-reads-and-writes) |
| `history.transferProcessorMaxPollInterval`, `history.visibilityProcessorMaxPollInterval`, `history.outboundProcessorMaxPollInterval` | 1m | How often those queues poll; no effect at rest | [4.2](#42-transfer-visibility-outbound-and-replication-queues-at-rest) |
| `history.timerProcessorMaxTimeShift` | 1s | How far past the current time the timer and archival queues read | [4.4](#44-settings-for-how-often-each-shard-reads-and-writes) |
| `system.namespaceCacheRefreshInterval` | 2s | How often every service instance re-reads the namespace list | [3.3](#33-settings-for-service-instance-and-cluster-wide-database-calls-at-rest), [5.7](#57-settings-for-the-per-namespace-work) |
| `system.clusterMetadataRefreshInterval` | 1m | How often every service instance re-reads the cluster's record; read only at startup | [3.3](#33-settings-for-service-instance-and-cluster-wide-database-calls-at-rest) |
| `matching.nexusEndpointsRefreshInterval` | 10s | How often one matching instance checks the Nexus endpoint list | [3.3](#33-settings-for-service-instance-and-cluster-wide-database-calls-at-rest) |
| `matching.updateAckInterval` | 1m; 5m for the per-namespace worker's task queue | How often each loaded task queue partition confirms which matching instance owns it | [3.3](#33-settings-for-service-instance-and-cluster-wide-database-calls-at-rest) |
| `matching.numTaskqueueReadPartitions`, `matching.numTaskqueueWritePartitions` | 4; 1 for a few internal queues | Partitions per task queue | [3.3](#33-settings-for-service-instance-and-cluster-wide-database-calls-at-rest) |
| `matching.getUserDataRefresh` | 5m | How often the matching service re-reads each task queue's settings | [5.7](#57-settings-for-the-per-namespace-work) |
| `matching.longPollExpirationInterval` | 1m | How long a poll with no task stays open | [5.2](#52-the-per-namespace-worker-and-its-polls) |
| `worker.perNamespaceWorkerCount` | 1, per namespace | Internal workers per namespace; 0 stops the namespace's internal worker | [5.7](#57-settings-for-the-per-namespace-work) |
| `worker.perNamespaceWorkerOptions` | 2 workflow and 2 activity pollers | Pollers per internal worker | [5.7](#57-settings-for-the-per-namespace-work) |
| `worker.perNamespaceWorkerStartRate` | 10 per second | How fast each worker instance starts internal workers | [5.7](#57-settings-for-the-per-namespace-work) |
| `metrics.breakdownByTaskQueue`, `metrics.breakdownByPartition` | true | Whether metrics carry the real task queue name and partition | [5.5](#55-metric-series-each-namespace-adds) |
| `worker.historyScannerEnabled` | true | Whether the history scan runs | [6.2](#62-history-and-task-queue-scans-twice-a-day) |
| `worker.taskQueueScannerEnabled` | true (SQL databases only) | Whether the task queue scan runs | [6.2](#62-history-and-task-queue-scans-twice-a-day) |
| `worker.executionsScannerEnabled` | false (Cassandra only) | Whether the executions scan runs | [6.2](#62-history-and-task-queue-scans-twice-a-day) |
| `worker.buildIdScavengerEnabled` | false | Whether the build ID scan runs | [6.2](#62-history-and-task-queue-scans-twice-a-day) |
| `worker.scannerPersistenceMaxQPS` | 100 | The history scan's database call limit | [6.2](#62-history-and-task-queue-scans-twice-a-day) |
| `worker.persistenceMaxQPS` | 500 per worker instance | The worker service's overall database call limit, which also holds back the task queue scan | [6.2](#62-history-and-task-queue-scans-twice-a-day) |

### 9.2 Static configuration and environment

| Setting | Default | What it controls | Section |
|---|---|---|---|
| `archival:` block — `archival.history.state`, `archival.visibility.state` | `enabled` in the configuration templates the server ships with | Whether every shard runs an archival queue | [4.3](#43-archival-enabled-in-the-static-configuration) |

Shard count is not in either table because it cannot be changed after a cluster is created;
[section 4](#4-database-calls-memory-and-cpu-from-history-shards) covers what each shard adds to
database calls, memory and CPU.

### 9.3 Metrics

Every metric this playbook relies on, the panel that shows it, and where the playbook uses it. The
operation names in [section 4.1](#41-what-each-shard-does-at-rest) and [section
8.6](#86-explaining-a-measured-baseline-above-its-calculated-level) — `GetTimerTasks`,
`UpdateShard`, `ListNamespaces` and the rest — are values of the `operation` label on
`persistence_requests`.

| Metric | What it measures | Panel | Section |
|---|---|---|---|
| `persistence_requests` | Database calls, by `operation` and `service_name` | **[Database Calls per Operation, 30-Minute Average](../observability/dashboards/server/temporal-server-readme.md#24-resting-load-idle-baseline)**, **[Database Calls Total, 30-Minute Average](../observability/dashboards/server/temporal-server-readme.md#24-resting-load-idle-baseline)**; over a short window, **[Persistence Requests Total per Operation](../observability/dashboards/server/temporal-server-readme.md#3-persistence-requests-latencies-and-errors)** | [3](#3-database-calls-memory-and-cpu-per-service-instance) to [8](#8-calculating-the-baseline-of-a-cluster-you-are-planning) |
| `total_namespaces` | Namespaces in each service instance's namespace list, including those in the DELETED state until their rows are removed. From v1.31.0 | **[Namespaces Registered](../observability/dashboards/server/temporal-server-readme.md#24-resting-load-idle-baseline)** | [5](#5-database-calls-memory-cpu-and-metric-series-per-namespace), [7.4](#74-re-taking-a-baseline-after-the-cluster-changes) |
| `namespace_registry_refresh_latency` | Time one full refresh of the namespace list takes, per service. From v1.31.0 | **[Namespace Registry Refresh Latency](../observability/dashboards/server/temporal-server-readme.md#24-resting-load-idle-baseline)** | [5.1](#51-database-calls-each-namespace-adds), [7.4](#74-re-taking-a-baseline-after-the-cluster-changes), [7.5](#75-why-resting-load-has-no-alert) |
| `scrape_samples_scraped` | Series Prometheus collects from each target on every scrape — a Prometheus metric, not a Temporal one | **[Metric Series Scraped per Service](../observability/dashboards/server/temporal-server-readme.md#24-resting-load-idle-baseline)** | [5.5](#55-metric-series-each-namespace-adds), [8.3](#83-metric-series) |
| `memory_heap` | The Go runtime's allocated heap per service instance. It equals *live heap* only right after a garbage collection, so compare its low points with live heap | **[Heap per Service Instance, 10-Minute Low](../observability/dashboards/server/temporal-server-readme.md#24-resting-load-idle-baseline)** | [3.4](#34-memory-and-cpu-a-service-instance-holds-at-rest), [5.3](#53-worker-internal-frontend-and-matching-memory-per-namespace), [8.2](#82-memory-per-service-instance-at-rest) |
| `dlq_message_count` | Tasks waiting in each history task dead-letter queue, counted every three hours by the history instance that owns shard 1 | **[Dead-Letter Queue Depth by Category](../observability/dashboards/server/temporal-server-readme.md#20-history-task-dlq--terminal-failures)** | [6.3](#63-dead-letter-queue-depth-counted-every-three-hours) |
| Container resident memory and CPU | All the memory each process occupies; compare it with a pod's memory limit. Not exported by the server; from your container platform's own metrics | No panel | [3.4](#34-memory-and-cpu-a-service-instance-holds-at-rest), [5.3](#53-worker-internal-frontend-and-matching-memory-per-namespace), [8.2](#82-memory-per-service-instance-at-rest) |
