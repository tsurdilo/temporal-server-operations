# Use Case: The Sleep-and-Wake Pattern — Many Timers Firing at the Same Time Across Workflows

**Long-running workflows that sleep and then wake up to carry on are a normal, well-supported way to
use Temporal.** Workflows that wait until a fixed time before their next step rely on durable
timers, which the Temporal server creates, keeps track of and fires. Because the server manages the
timers, a workflow can sleep for any length of time without keeping any SDK worker busy.

Reminders, follow-up campaigns, business-hours processing and daily jobs are examples of use cases
implemented this way, which this playbook calls the **sleep-and-wake pattern**. Workflows that
follow the pattern have at least one of these characteristics:

- **they are long-running and wait between steps**, with a workflow sleep, a timer, or an await
  with a timeout;
- **they are started with a start delay, or with the older cron start option**, and wait for a
  server timer before they run their first step.

**This playbook is about many running workflows that follow the sleep-and-wake pattern, with timers
that fire at the same time or close to each other.** At small scale, almost any use case runs on
Temporal without noticeable pressure on the cluster, the database or the SDK workers. At large
scale, no single workflow is the problem: when many timers fire close to each other, the work of all
their workflows arrives at once, and that can put pressure on the cluster, the database and the SDK
workers.

The goal of this playbook is to help you run the sleep-and-wake pattern successfully at large scale.
After [section 1](#1-timers-and-workflow-history-in-the-sleep-and-wake-pattern) describes the
pattern in detail, it covers:

- **Understanding the pressure** that many timers firing close to each other can put on the cluster,
  the database and the SDK workers ([section 2](#2-what-happens-in-the-cluster-when-a-timer-fires)
  and [section 3](#3-how-much-pressure-the-timer-firings-add)).
- **Observing whether your implementation creates that pressure**, today and as the number of
  workflows or the size of their histories grows
  ([section 4](#4-observing-the-pressure-from-the-timer-firings-on-your-dashboards)).
- **Recommendations that reduce that pressure** in the workflow implementation, the SDK workers
  and the cluster, so the pattern runs well at large scale today and keeps running well as the
  scale grows ([section 5](#5-recommendations-for-running-the-sleep-and-wake-pattern-at-large-scale)).

## Audience

Developers who write Temporal applications, and operators of self-hosted Temporal clusters.

## Applies to

Any server version, any persistence store and any Temporal SDK, single-cluster or multi-cluster.
The recommendations apply to every Temporal SDK. Where this playbook names SDK options or SDK
behavior, it describes the Go and Java SDKs; check the same detail in the documentation of your SDK.

## Dashboards

Every panel this playbook names is on one of the dashboards in this repository, under the exact
title used here:

| Dashboard | What it shows |
|---|---|
| [Temporal Server](../observability/dashboards/server/temporal-server-readme.md) | The history service, the matching service and the database calls the server makes |
| [Go SDK](../observability/dashboards/sdk/temporal-sdk-go-readme.md) | SDK workers built with the Go SDK |
| [Java SDK (Micrometer)](../observability/dashboards/sdk/temporal-sdk-java-micrometer-readme.md) | SDK workers built with the Java SDK, reporting through Micrometer |
| [Java SDK (OpenTelemetry)](../observability/dashboards/sdk/temporal-sdk-java-otel-readme.md) | SDK workers built with the Java SDK, reporting through OpenTelemetry |
| [Core SDK](../observability/dashboards/sdk/temporal-sdk-core-readme.md) | SDK workers built with the TypeScript, Python, .NET or Ruby SDK |

## Contents

- [1. Timers and workflow history in the sleep-and-wake pattern](#1-timers-and-workflow-history-in-the-sleep-and-wake-pattern)
  - [1.1 Workflows with a large event history count](#11-workflows-with-a-large-event-history-count)
  - [1.2 Workflows with a large event history size](#12-workflows-with-a-large-event-history-size)
  - [1.3 Workflows with a small event history](#13-workflows-with-a-small-event-history)
  - [1.4 Workflows started with a start delay or the cron start option](#14-workflows-started-with-a-start-delay-or-the-cron-start-option)
- [2. What happens in the cluster when a timer fires](#2-what-happens-in-the-cluster-when-a-timer-fires)
  - [2.1 Timer tasks in the history service](#21-timer-tasks-in-the-history-service)
  - [2.2 Workflow tasks in the matching service](#22-workflow-tasks-in-the-matching-service)
  - [2.3 Reading the event history when a workflow task is recorded as started](#23-reading-the-event-history-when-a-workflow-task-is-recorded-as-started)
  - [2.4 Event history replay and task slots on SDK workers](#24-event-history-replay-and-task-slots-on-sdk-workers)
- [3. How much pressure the timer firings add](#3-how-much-pressure-the-timer-firings-add)
- [4. Observing the pressure from the timer firings on your dashboards](#4-observing-the-pressure-from-the-timer-firings-on-your-dashboards)
  - [4.1 Timer task metrics in the history service](#41-timer-task-metrics-in-the-history-service)
  - [4.2 Workflow task metrics in the matching service](#42-workflow-task-metrics-in-the-matching-service)
  - [4.3 Metrics from reading the event history in the database](#43-metrics-from-reading-the-event-history-in-the-database)
  - [4.4 Event history replay and task slot metrics on SDK workers](#44-event-history-replay-and-task-slot-metrics-on-sdk-workers)
  - [4.5 Trends that show the wake-up bursts growing](#45-trends-that-show-the-wake-up-bursts-growing)
- [5. Recommendations for running the sleep-and-wake pattern at large scale](#5-recommendations-for-running-the-sleep-and-wake-pattern-at-large-scale)
  - [5.1 Continue-as-new before the long sleep (workflow implementation)](#51-continue-as-new-before-the-long-sleep-workflow-implementation)
  - [5.2 Spread the wake-up times (workflow implementation)](#52-spread-the-wake-up-times-workflow-implementation)
  - [5.3 Keep payloads small (workflow implementation)](#53-keep-payloads-small-workflow-implementation)
  - [5.4 Keep SDK workers running and size their cache (SDK workers)](#54-keep-sdk-workers-running-and-size-their-cache-sdk-workers)
  - [5.5 Cluster limits and database capacity (cluster)](#55-cluster-limits-and-database-capacity-cluster)

---

## 1. Timers and workflow history in the sleep-and-wake pattern

As the introduction says, workflows that follow the sleep-and-wake pattern have at least one of
these characteristics:

- they are long-running and wait between steps, with a workflow sleep, a timer, or an await with a
  timeout;
- they are started with a start delay, or with the older cron start option.

Each characteristic comes from a different kind of timer.

**Waiting between steps uses a user timer**: a timer the workflow's own business logic creates, in
the workflow code. A long-running workflow that waits between steps repeats the same cycle:

1. **Run a step, or a series of steps**, where each step is a Temporal activity or a child workflow.
2. **Wait on a user timer.** The workflow code creates the timer in one of three ways:
   - a **workflow sleep** (`workflow.Sleep` in Go, `Workflow.sleep` in Java);
   - a **timer** the workflow creates and then waits for (`workflow.NewTimer`, `Workflow.newTimer`);
   - an **await with a timeout**, which waits for a condition but no longer than the timeout
     (`workflow.AwaitWithTimeout`, `Workflow.await` with a duration). It wakes early if the
     condition becomes true first, for example after a signal.
3. **Wake up when the timer fires.** Until then the workflow execution makes no progress. When the
   timer fires, the workflow runs its next step or series of steps, then waits on its next user
   timer. The cycle repeats until the workflow ends.

Changing a user timer means changing the workflow code.

**A start delay or a cron start uses a start timer**: a timer the server creates from the options
the workflow is started with: the start delay passed in the `StartWorkflowExecution` request, or
the cron expression of the older cron start option. It holds the workflow before its first step;
when it fires, the workflow runs its first step and from then on can follow the cycle above.
Changing a start timer means changing the code that starts the workflow.
[Section 1.4](#14-workflows-started-with-a-start-delay-or-the-cron-start-option) covers these
workflows.

**The kind of timer matters because of the workflow's history at the moment the timer fires.** At
large scale both kinds of timer can put pressure on the cluster, but a user timer can put on more:

- **When a start timer fires**, the workflow's history contains the `WorkflowExecutionStarted`
  event with the workflow's input, plus any signals the workflow received while it waited.
- **When a user timer fires**, the history contains everything the workflow has done so far. The
  more times a workflow goes through the sleep-and-wake cycle, the more events its history contains
  and the larger it gets.

Where the playbook says only "timer", what it says applies to both kinds.

[Sections 1.1 to 1.4](#11-workflows-with-a-large-event-history-count) describe four kinds of
sleep-and-wake workflows. In all four, many timers fire at the same time or close to it. What
differs is the cost of each timer firing, and that depends on the workflow's history when the timer
fires:

- **Large histories ([1.1](#11-workflows-with-a-large-event-history-count),
  [1.2](#12-workflows-with-a-large-event-history-size)):** many events, or large ones. The pressure
  comes from two things, above all on the database:
  - How many timers fire close to each other.
  - The size of each workflow's event history.
- **Small histories ([1.3](#13-workflows-with-a-small-event-history),
  [1.4](#14-workflows-started-with-a-start-delay-or-the-cron-start-option)):** the pressure comes
  only from how many timers fire close to each other.

### 1.1 Workflows with a large event history count

The workflows run for days or weeks and repeat the same cycle: sleep, run one or more steps, sleep
again. Every cycle adds events to the workflow's history:

- The timer starting and firing.
- The workflow task that handles the wake-up.
- Three events for every activity: scheduled, started and completed.
- Three events for every child workflow: initiated, started and completed.

After weeks, a workflow's history can hold thousands of events. It keeps growing until the workflow
ends.

### 1.2 Workflows with a large event history size

These workflows may have few events, but each event is large. This happens when:

- Activities or child workflows take or return large payloads. Every input and result is stored in
  the history.
- Many activities or child workflows run between two sleeps, each adding its own input and result.

### 1.3 Workflows with a small event history

These workflows are woken by user timers, and have short histories and small payloads. The
pressure comes only from how many of them wake up in the same minute.

### 1.4 Workflows started with a start delay or the cron start option

These workflows put the same kind of pressure on the cluster as those in
[section 1.3](#13-workflows-with-a-small-event-history), but they are woken by a start timer instead
of a user timer. The code that starts the workflow sets the start timer, with one of these options:

- **A start delay:** `StartDelay` in the Go SDK's start options, `setStartDelay` in the Java SDK's
  `WorkflowOptions`.
- **The cron start option:** `CronSchedule` in Go, `setCronSchedule` in Java. A cron workflow waits
  again before every new run.

When the start timer fires, the workflow runs its first step. Its history at that moment contains
the start event with the workflow's input, plus any signals the workflow received while it waited.
A large input or many signals make that history larger.

When a timer fires, the cluster and the SDK workers do the same steps of work for all four kinds of
sleep-and-wake workflows. [Section 2](#2-what-happens-in-the-cluster-when-a-timer-fires) describes
those steps, and [section 3](#3-how-much-pressure-the-timer-firings-add) how much pressure they add
for each kind.

## 2. What happens in the cluster when a timer fires

This section describes the work the cluster and the SDK workers do each time a timer fires, whether
it is a user timer or a start timer, until the workflow runs its next step. In this playbook a
**wake-up** is that work for one workflow.

These steps are how Temporal runs every workflow that a timer wakes up, and on their own they are
not a problem. Knowing them makes it easier to see, in
[section 3](#3-how-much-pressure-the-timer-firings-add), where many workflows waking close to each
other add pressure, and why the size of the event history matters.

**Every timer that fires starts the same four steps of work:**

1. **The history service processes the timer task and sends a workflow task to the matching
   service.** The timer task gets the workflow's current state, adds `TimerFired` (for a user timer)
   and `WorkflowTaskScheduled` to the event history, and writes them to the database. This creates a
   workflow task, which the history service processes as a separate task: it sends the workflow task
   to the matching service, without any history events, on one of two task queues:
   - **The workflow's sticky queue**, if it has one. Every SDK worker has its own sticky queue, and
     a workflow gets it when that SDK worker completes a workflow task for it. The workflow task
     then reaches the SDK worker that ran the workflow last, which may still hold the workflow in
     its **worker in-memory cache**. A workflow woken by a user timer usually has a sticky queue.
   - **The normal task queue** otherwise, polled by all SDK workers on that task queue. A workflow
     woken by a start timer (a start delay or the cron start option) always uses it: the wake-up is
     the run's first workflow task, so no SDK worker has completed one for it yet.
2. **The matching service pairs the workflow task with a poll on the same task queue.**
   An SDK worker asks for workflow tasks with `PollWorkflowTaskQueue` requests to the frontend
   service, which passes them on to the matching service. Each request names one task queue: the
   normal task queue, or the sticky queue of that SDK worker. The SDK worker splits its polls
   between the two. With no poll waiting, the workflow task waits in a backlog. If the SDK worker
   behind a sticky queue has stopped polling, or does not pick the workflow task up in time, the
   history service puts the wake-up on the normal task queue instead.
3. **The history service records the workflow task as started and reads its history events.**
   Before it answers the poll, the matching service asks the history service to record it as
   started. The history service reads from the database the events to send with it:
   - **Sticky queue:** only the events added since that SDK worker last ran the workflow.
   - **Normal task queue:** the event history from the first event, up to one **page** (256 events
     by default).

   The history service returns those events to the matching service. The matching service returns
   the workflow task and its events to the frontend service, which answers the SDK worker's poll.
4. **The SDK worker runs the workflow task.**
   - If the workflow is in its worker in-memory cache, it applies the new events and runs the
     workflow's next step.
   - If not, it replays the whole event history. It first requests the events it was not sent,
     one page at a time: the whole event history if the workflow task came from the sticky queue,
     or the pages after the first one if it came from the normal task queue. Each page is another
     database read.

[Sections 2.1 to 2.4](#21-timer-tasks-in-the-history-service) take the four steps of a wake-up
one at a time, and what each step costs.
[Section 3](#3-how-much-pressure-the-timer-firings-add) then puts them together.

### 2.1 Timer tasks in the history service

When a workflow starts a user timer, or is started with a start timer, the history service saves a
**timer task** in the database with the time the timer fires. The history service's **timer
queue** loads each shard's timer tasks from the database as their fire times come, and then
processes them, many at a time. Processing a timer task means three things:

1. Getting the workflow's **current state** (the server calls it mutable state), which the history
   service keeps apart from the event history. The timer task uses it to check that the workflow is
   still running and that the timer is still pending: while the workflow slept, it may have ended,
   for example by being terminated, or its code may have cancelled the timer. If either check fails,
   the timer task is dropped and nothing else happens.
2. Adding the new events to the event history: `TimerFired` for a user timer, then
   `WorkflowTaskScheduled` for both kinds of timer.
3. Saving the new events and the updated current state to the database. The updated current state
   is saved together with a small task record that tells the history service to send the new
   workflow task to the matching service.

The history service then processes the workflow task as a separate task. It gets the workflow's
current state again, usually still in memory, to check that the workflow is still running and the
workflow task is still pending. Then it sends the workflow task to the matching service.

**The timer task and the workflow task are both work for the history service and its database:**

- **Getting the current state can be a database read.** The timer task and the workflow task both
  get the workflow's current state. Each history pod keeps the current state of recently used
  workflows in an in-memory cache, shared by all its shards. A workflow drops out of that cache
  when:
  - an hour has passed since its current state was loaded (`history.cacheTTL`), even if it was used
    in between;
  - the cache is full (`history.hostLevelCacheMaxSize`, default 128,000 workflows per pod) and
    other workflows need the room;
  - the history pod restarts, or the workflow's shard moves to another pod.

  The history service then reads the current state back from the database. A workflow that sleeps
  longer than an hour needs this read when it wakes, unless something else, such as a signal, loaded
  its current state within the last hour. The read is larger when the workflow waits on many things
  (pending activities, timers and child workflows) or has large search attributes and memo.
- **All namespaces share task processing.** Each shard holds workflows from every namespace on the
  cluster, and each history pod processes the tasks of all of them together, with the same
  processing limits and task slots. A large burst of timer tasks, and the workflow tasks that follow
  them, can delay the tasks of other namespaces too, not only those of the namespace whose timers
  fire. The
  [History Task Processing](./history-task-processing-tuning.md#2-how-task-processing-works)
  playbook explains how the history service processes tasks, and
  [how to pace a burst](./history-task-processing-tuning.md#5-sizing-the-limits-and-turning-them-on).
- **The history service limits its own database calls** per pod (`history.persistenceMaxQPS`,
  default 9,000 per second) and, when set, per namespace (`history.persistenceNamespaceMaxQPS`,
  default 0, which falls back to the per-pod limit). A namespace running the sleep-and-wake pattern
  at large scale can use a large share of that budget during a burst, which is how it can put
  pressure on other namespaces.
  [History Persistence QPS Limits](./history-persistence-qps-limits.md#12-set-them-deliberately--do-not-rely-on-the-defaults)
  covers how to set these limits.

Once the workflow task reaches the matching service,
[section 2.2](#22-workflow-tasks-in-the-matching-service) covers what happens to it there.

### 2.2 Workflow tasks in the matching service

Every wake-up is one workflow task in the matching service, on the sticky queue or the normal task
queue that the history service chose for it
([steps 1 and 2 in section 2](#2-what-happens-in-the-cluster-when-a-timer-fires)). The matching
service holds the workflow task until an SDK worker polls that task queue.

**When many timers fire close to each other, workflow tasks can arrive faster than the matching
service can match them with polls.** Two things limit how fast it matches them:

- **How fast each partition of the task queue can match tasks with polls.** A normal task queue has
  4 partitions by default, and a sticky queue has one. Each partition is capped at 1,000 tasks per
  second by default (`admin.matchingNamespaceTaskqueueToPartitionDispatchRate`), but that is only a
  cap. Once workflow tasks fall into a backlog, the partition writes them to the database and reads
  them back, and it can match them only as fast as those database round trips allow, which can be
  well below the cap.
  [Changing Task Queue Partitions](./change-task-queue-partitions.md#how-a-partitions-throughput-works)
  explains these limits.
- **How fast the SDK workers poll.** An SDK worker gets a workflow task only when it polls. With too
  few SDK workers, too few pollers per SDK worker, or SDK workers busy with other work, the polls
  come in slower than the workflow tasks arrive.

Workflow tasks that cannot be matched with a poll straight away wait in a backlog.
[Changing Task Queue Partitions](./change-task-queue-partitions.md#when-to-increase-partitions)
covers whether more partitions would help.

**A backlog adds database work that grows with the number of waiting workflow tasks, not with the
size of the event history.** The matching service writes each waiting workflow task to the database
and reads it back to match it with a poll; matched tasks are later removed together, in one range
delete. What it stores is a small reference to
the workflow task, never the event history. So a backlog costs the same for workflows with a small
event history as for workflows with a large one. The event history size matters later, when the
history service reads the history events
([section 2.3](#23-reading-the-event-history-when-a-workflow-task-is-recorded-as-started)).

**When the workflow's next step is an activity, the burst of wake-ups is followed by a burst of
activity tasks.** Once a woken workflow runs, the SDK worker schedules its activities. For each
activity, the history service records it and sends an activity task to the matching service, which
matches it with an SDK worker's poll the same way as a workflow task. The matching service is
skipped only when the activity goes straight back to the SDK worker that scheduled it (eager
activity dispatch). That needs the server to allow it (`system.enableActivityEagerExecution`, on by
default from server v1.32.0, off before), and the SDK worker to ask for it and have a free activity
slot.
[Changing Task Queue Partitions — eager dispatch](./change-task-queue-partitions.md#activity-queues-can-look-empty-eager-dispatch)
covers how eager dispatch changes what the matching service sees.

Before the workflow task goes back to the SDK worker, the history service reads the workflow's
history events, as [section 2.3](#23-reading-the-event-history-when-a-workflow-task-is-recorded-as-started)
describes.

### 2.3 Reading the event history when a workflow task is recorded as started

When the matching service pairs a workflow task with a poll, it first asks the history service to
record that the workflow task has started. In the same request, the history service reads from the
database the history events to send with it. In server versions before v1.23.0, the frontend
service makes this read instead, after the matching service answers; the database reads the same
events either way. How many events it reads depends on which task queue the workflow task came
from:

- **The workflow's sticky queue.** The history service reads only the new events, a few rows. If the
  workflow is no longer in the worker in-memory cache of that SDK worker, the SDK worker then
  fetches the whole event history itself. After a sleep of hours this is likely: the cache drops the
  least recently used workflows when it is full.
- **The workflow's normal task queue.** The history service reads the first page of the workflow's
  whole history, and the SDK worker that picks the task up fetches the rest. A user-timer wake-up
  uses the normal task queue in two cases:
  - the SDK worker behind the sticky queue did not pick the task up within its sticky
    schedule-to-start timeout (an SDK worker option, 5 seconds by default in the Go and Java SDKs);
  - the SDK worker behind the sticky queue is gone, for example because it was restarted or scaled
    down. If nothing has polled the sticky queue in the last 10 seconds (fixed, not configurable),
    the matching service rejects the workflow task, and the history service adds it to the normal
    task queue at once.

A start-timer wake-up always uses the normal task queue. The workflow it wakes has a short event
history: its start event with the input, plus any signals received while it waited
([section 1.4](#14-workflows-started-with-a-start-delay-or-the-cron-start-option)).

**You do not fully control whether a wake-up reads only the new events or the full event history.**
That depends on the worker in-memory caches of the SDK workers, how busy the SDK workers are, and
whether they were restarted or scaled down while the workflow slept.

**For a long-running workflow, every wake-up finds a larger event history than the one before.**
Each cycle adds events, so both the event history count and the event history size grow, and so
does the read when the full event history is needed. Take an **example workflow** that runs 10
activities a day for 60 days, with a 3 KB input to each activity, and sleeps between them and
overnight: its full event history is a few dozen KB on day 2 and about 2 MB on day 60, most of it
the 600 activity inputs (about 1.8 MB).

[Section 2.4](#24-event-history-replay-and-task-slots-on-sdk-workers) covers what the SDK worker does with the
history events it was sent.

### 2.4 Event history replay and task slots on SDK workers

Every workflow task takes one of the SDK worker's workflow task slots until it completes, and the
time it takes counts against the **workflow task timeout** (10 seconds by default, set per workflow
when it is started). If the workflow is not in the worker in-memory cache of the SDK worker, that
time includes fetching the remaining pages of the event history and replaying it. The larger the
event history, the longer this takes:

- the workflow task slot stays busy for longer, so the SDK worker can run fewer workflow tasks at
  once;
- the remaining pages are requests to the frontend service. When many SDK workers fetch pages at
  once, the frontend service's request limits can throttle them, which makes the workflow task take
  even longer;
- with a large enough event history, the workflow task can time out. The history service then
  schedules it again on the normal task queue, which means reading the full event history again.

**SDK workers that fall behind make the wake-ups that follow cost more.** If the SDK workers cannot
keep up with a burst, sticky tasks wait longer than the sticky schedule-to-start timeout, and the
history service schedules them again on the normal task queue. Each of those becomes a full-history
read from the database
([section 2.3](#23-reading-the-event-history-when-a-workflow-task-is-recorded-as-started)) and a full replay on
an SDK worker that is already busy, which makes more tasks miss the timeout.

Running the workflow task on the SDK worker is the last step of a wake-up.
[Section 3](#3-how-much-pressure-the-timer-firings-add) looks at how much pressure all four steps of
a wake-up add together.

## 3. How much pressure the timer firings add

[Section 2](#2-what-happens-in-the-cluster-when-a-timer-fires) described the four steps every
wake-up goes through. This section puts them together: what makes a burst of wake-ups put more or
less pressure on the cluster, the database and the SDK workers.

**Four things decide how much pressure a burst of wake-ups adds:**

1. **How many workflows wake close to each other.** Each wake-up goes through all four steps of
   [section 2](#2-what-happens-in-the-cluster-when-a-timer-fires), so this adds pressure at every
   step, for every kind of sleep-and-wake workflow. Wake-up times spread over an hour add little;
   every workflow's timer set to the same clock time adds the most.
2. **The size of the event history at wake-up.** This matters for long-running workflows
   ([1.1](#11-workflows-with-a-large-event-history-count),
   [1.2](#12-workflows-with-a-large-event-history-size)), whose event history grows with every
   cycle. It matters little for workflows started with a start delay or the cron
   start option ([1.4](#14-workflows-started-with-a-start-delay-or-the-cron-start-option)), whose
   event history at wake-up is the start event and any signals. The size counts whenever the full
   event history is read from the database
   ([section 2.3](#23-reading-the-event-history-when-a-workflow-task-is-recorded-as-started)) and replayed on an
   SDK worker ([section 2.4](#24-event-history-replay-and-task-slots-on-sdk-workers)): a few dozen small events
   are a few rows, thousands of events or large payloads are megabytes for every wake-up.
3. **Whether the SDK worker still holds the workflow in its worker in-memory cache.** This decides
   whether a wake-up reads and replays only the new events or the full event history
   ([section 2.3](#23-reading-the-event-history-when-a-workflow-task-is-recorded-as-started)). The workflow is
   more likely to still be in the cache after a short sleep on an SDK worker that keeps up. It is
   less likely after a sleep of many hours, on an SDK worker that falls behind, or when the SDK
   workers were restarted or scaled down while the workflow slept.
4. **The size of the workflow's current state.** The current state (mutable state) is what the
   history service keeps about a workflow apart from its event history: its pending activities,
   timers and child workflows, its search attributes and its memo. The history service needs it to
   process the timer task and the workflow task
   ([section 2.1](#21-timer-tasks-in-the-history-service)). It keeps the current state in memory
   for up to an hour, so after a longer sleep it reads the current state back from the database.
   This read is separate from reading the event history, and the more the current state holds, the
   larger it is.

**The four kinds of sleep-and-wake workflows from
[section 1](#1-timers-and-workflow-history-in-the-sleep-and-wake-pattern) feel these differently:**

| Kind of workflow | How many wake close to each other | Size of the event history at wake-up |
|---|---|---|
| Large event history count ([1.1](#11-workflows-with-a-large-event-history-count)) | Adds pressure | Adds pressure, and grows with the workflow's age |
| Large event history size ([1.2](#12-workflows-with-a-large-event-history-size)) | Adds pressure | Adds pressure, and grows with the size of the payloads |
| Small event history ([1.3](#13-workflows-with-a-small-event-history)) | Adds pressure | Little |
| Started with a start delay or the cron start option ([1.4](#14-workflows-started-with-a-start-delay-or-the-cron-start-option)) | Adds pressure | Little: the start event with the input, and any signals |

**What the table above comes down to, by kind of workflow:**

- **Workflows with a small event history, and workflows started with a start delay or the cron
  start option:** the pressure is task throughput, that is timer tasks in the history service,
  workflow tasks in the matching service, and the SDK workers' task slots.
- **Workflows with a large event history count or size:** the same task throughput, plus reading
  and replaying the full event history.

**How many wake-ups make a burst large depends on the cluster.** There is no fixed number: the same
burst can be easy for one cluster and too much for another. Two kinds of cluster feel a burst
sooner:

- **A cluster with little capacity**, with few history and matching instances and a small database.
- **A cluster shared by many namespaces.** The timer firings of one namespace also put pressure on
  the other namespaces, because they share the history service's task processing and database calls
  ([section 2.1](#21-timer-tasks-in-the-history-service)).

**A burst slows the steps of a wake-up down, and usually does not drop the work in them.** A timer
task or a database call that the history service's own limits turn away is retried until it
succeeds. The exception is a database that is pushed past what it can answer: its calls then time
out instead of being turned away, and a task that keeps timing out is moved to the history
service's dead-letter queue after about an hour. A timer task or a workflow task in the dead-letter
queue leaves its workflow stuck until an operator moves the task back.
[History Persistence QPS Limits](./history-persistence-qps-limits.md#11-what-happens-when-a-qps-limit-is-reached)
explains when tasks are dead-lettered and how to recover them.

**The sleep-and-wake pattern runs on any cluster, including one with a small history service or a
small database.** None of the pressure above is a reason not to use it. What changes with the
cluster is how many wake-ups it takes before the pressure shows, and which steps of a wake-up feel
it first. [Section 4](#4-observing-the-pressure-from-the-timer-firings-on-your-dashboards) shows how to see
that pressure on your own cluster, step by step, and
[section 5](#5-recommendations-for-running-the-sleep-and-wake-pattern-at-large-scale) how to reduce
it, so the pattern keeps running well as the number of workflows grows.

## 4. Observing the pressure from the timer firings on your dashboards

This section shows how to use server and SDK metrics to observe the pressure that the
sleep-and-wake pattern can put on your own cluster. It follows the four steps of a wake-up from
[section 2](#2-what-happens-in-the-cluster-when-a-timer-fires), then the trends to keep watching as
the sleep-and-wake pattern grows:

- [4.1 Timer task metrics in the history service](#41-timer-task-metrics-in-the-history-service)
- [4.2 Workflow task metrics in the matching service](#42-workflow-task-metrics-in-the-matching-service)
- [4.3 Metrics from reading the event history in the database](#43-metrics-from-reading-the-event-history-in-the-database)
- [4.4 Event history replay and task slot metrics on SDK workers](#44-event-history-replay-and-task-slot-metrics-on-sdk-workers)
- [4.5 Trends that show the wake-up bursts growing](#45-trends-that-show-the-wake-up-bursts-growing)

**Read the panels in [sections 4.1 to 4.4](#41-timer-task-metrics-in-the-history-service) over the
minutes around the wake-up time**, when many timers fire close together.

### 4.1 Timer task metrics in the history service

The timer tasks of [section 2.1](#21-timer-tasks-in-the-history-service) show on these panels:

| What to look at | Normal at the wake-up time | Worth acting on | Related sections and alerts |
|---|---|---|---|
| Timer tasks across the whole cluster, on **[Total Timer Tasks Processed](../observability/dashboards/server/temporal-server-readme.md#10-history-timer-task-info)** | A spike at the wake-up time that drops back within minutes | The spike at the wake-up time gets higher over time (see [section 4.5](#45-trends-that-show-the-wake-up-bursts-growing)) | [5.2](#52-spread-the-wake-up-times-workflow-implementation) |
| Timer types in the namespace selected on the dashboard, on **[Timer Tasks Processed by Type](../observability/dashboards/server/temporal-server-readme.md#10-history-timer-task-info)**: user timers as `TimerActiveTaskUserTimer`, start timers ([section 1.4](#14-workflows-started-with-a-start-delay-or-the-cron-start-option)) and workflow retry waits as `TimerActiveTaskWorkflowBackoffTimer` | The timer type your workflows use spikes at the wake-up time. `TimerActiveTaskWorkflowTaskTimeout` rises with it: these are checks for the workflow task timeouts of the woken workflows, processed even when the workflow task finished in time, not timeouts | The spike of that type gets higher over time (see [section 4.5](#45-trends-that-show-the-wake-up-bursts-growing)) | [5.2](#52-spread-the-wake-up-times-workflow-implementation) |
| How long each timer task takes, on **[Timer Task Processing Latency](../observability/dashboards/server/temporal-server-readme.md#10-history-timer-task-info)** | Rises during the spike and drops back once the spike has drained | Stays high after the spike | [History Task Processing](./history-task-processing-tuning.md#2-how-task-processing-works); [alert 35](../observability/alerts/server/alerts-index.md#alert-35--timer-task-processing-latency-high) |
| Whether timers fire late, on **[Task Load Latency by Task Type](../observability/dashboards/server/temporal-server-readme.md#9-shard-queue-health)** (the `TimerActive` series, at the 0.95 percentile) | Close to zero, with a short rise at the wake-up time | Keeps rising after the wake-up time: the timer queue is behind and timers fire late | [History Task Processing](./history-task-processing-tuning.md#2-how-task-processing-works) |
| The workflow's current state loaded from the database when it is not in the history cache: the `GetWorkflowExecution` persistence operation on **[Persistence Requests Total per Operation](../observability/dashboards/server/temporal-server-readme.md#3-persistence-requests-latencies-and-errors)** | Rises with the spike: workflows that slept longer than an hour have their current state read back | **[Write-Reject Loop Indicator (cleared / cache miss)](../observability/dashboards/server/temporal-server-readme.md#3-persistence-requests-latencies-and-errors)** well above its healthy value, or `PersistenceLimit` on **[Resource Exhausted with Cause](../observability/dashboards/server/temporal-server-readme.md#6-throttling-and-limits)**: refused writes are turning into extra reads | [History Task Processing: the write-reject loop](./history-task-processing-tuning.md#3-what-happens-when-the-persistence-limit-refuses-a-tasks-write); [History Persistence QPS Limits](./history-persistence-qps-limits.md#41-first-confirm-the-limiter-is-what-rejected-the-calls); [alert 87](../observability/alerts/server/alerts-index.md#alert-87--history-write-reject-loop), [alert 29](../observability/alerts/server/alerts-index.md#alert-29--rpsqps-throttling-active) |
| Wake-up tasks moved to the dead-letter queue (operations ending in `UserTimer`, `WorkflowBackoffTimer` and `WorkflowTask`), on **[Dead-Lettered Tasks — Execution-Stranding (page-worthy)](../observability/dashboards/server/temporal-server-readme.md#20-history-task-dlq--terminal-failures)** and **[Task Retry Depth — Stranding Types (approaching DLQ)](../observability/dashboards/server/temporal-server-readme.md#20-history-task-dlq--terminal-failures)** | None dead-lettered. Retry depth can rise while tasks are throttled | Any dead-lettered: the workflow stays asleep until the tasks are moved back. Before that, retry depth climbs toward 70 while database calls time out (throttling also raises it, but never dead-letters) | [History Persistence QPS Limits](./history-persistence-qps-limits.md#11-what-happens-when-a-qps-limit-is-reached); [alert 80](../observability/alerts/server/alerts-index.md#alert-80--history-task-dlq-stranding) |

The next step of a wake-up is in the matching service: each timer that fires schedules a workflow
task, which waits there for an SDK worker's poll. [Section 4.2](#42-workflow-task-metrics-in-the-matching-service) covers
the metrics that show how long workflow tasks wait and whether they back up.

### 4.2 Workflow task metrics in the matching service

The workflow tasks of [section 2.2](#22-workflow-tasks-in-the-matching-service) show on these
panels:

| What to look at | Normal at the wake-up time | Worth acting on | Related sections and alerts |
|---|---|---|---|
| Workflow tasks waiting for an SDK worker, on **[Schedule to Start Latencies](../observability/dashboards/server/temporal-server-readme.md#14-sdk-workers-info)** and **[Approximate Task Backlog](../observability/dashboards/server/temporal-server-readme.md#14-sdk-workers-info)** | Schedule-to-start latency and the task queue backlog rise during the burst, and the task queue backlog drains within minutes | The task queue backlog keeps growing, or does not drain before the next burst | [Changing Task Queue Partitions](./change-task-queue-partitions.md#when-to-increase-partitions); [5.4](#54-keep-sdk-workers-running-and-size-their-cache-sdk-workers); [alert 48](../observability/alerts/server/alerts-index.md#alert-48--schedule-to-start-latency-high), [alert 50](../observability/alerts/server/alerts-index.md#alert-50--task-backlog-growing), [alert 88](../observability/alerts/server/alerts-index.md#alert-88--task-queue-partition-backlog-not-draining) |
| Workflow tasks written to the backlog in the database, on **[Tasks Persisted to DB (can indicate task backlog)](../observability/dashboards/server/temporal-server-readme.md#14-sdk-workers-info)** | Some at the peak of the burst | High for the whole burst: most workflow tasks are going through the database instead of straight to a poll | [Changing Task Queue Partitions](./change-task-queue-partitions.md#how-a-partitions-throughput-works); [alert 52](../observability/alerts/server/alerts-index.md#alert-52--tasks-persisted-rate-high) |
| Partitions of the workflow task queue falling behind, on **[Task Write Throttle Count by Partition](../observability/dashboards/server/task-queue-partitions-readme.md#row-3--backlog-write-path-2-ceiling)** and **[Approximate Backlog Age by Partition](../observability/dashboards/server/task-queue-partitions-readme.md#row-1--per-partition-backlog-drain--writeread-detection)** | No write throttling, and a backlog age under 5 seconds on every partition | Any write throttling, or a backlog age above 5 seconds: above that age the matching service stops matching workflow tasks straight to polls and sends every one through the database | [5.5](#55-cluster-limits-and-database-capacity-cluster); [Changing Task Queue Partitions](./change-task-queue-partitions.md#what-to-watch-on-the-dashboard); [alert 88](../observability/alerts/server/alerts-index.md#alert-88--task-queue-partition-backlog-not-draining) |
| Sticky workflow tasks that timed out and were scheduled again on the normal task queue, on **[Workflow Task Schedule-to-Start Timeouts (sticky fallback)](../observability/dashboards/server/temporal-server-readme.md#14-sdk-workers-info)** | Close to zero; a few after an SDK worker restart | Rises at the wake-up time: SDK workers are not picking up their sticky tasks in time, and every such task becomes a read of the full event history | [5.4](#54-keep-sdk-workers-running-and-size-their-cache-sdk-workers); [alert 76](../observability/alerts/server/alerts-index.md#alert-76--workflow-task-scheduletostart-timeout) |
| Open polls on the frontend service, on **[Total Concurrent Pollers](../observability/dashboards/server/temporal-server-readme.md#15-pollers)** and **[Max Concurrent Pollers per Frontend Pod](../observability/dashboards/server/temporal-server-readme.md#15-pollers)** | Rises at the wake-up time when SDK workers add pollers as tasks arrive (poller autoscaling) or more SDK workers start; steady otherwise | **[Max Concurrent Pollers per Frontend Pod](../observability/dashboards/server/temporal-server-readme.md#15-pollers)** close to `frontend.namespaceCount`, which applies to each poll API on each frontend instance, or `ConcurrentLimit` on **[Resource Exhausted with Cause](../observability/dashboards/server/temporal-server-readme.md#6-throttling-and-limits)** | [5.5](#55-cluster-limits-and-database-capacity-cluster); [alert 58](../observability/alerts/server/alerts-index.md#alert-58--pollers-approaching-concurrent-limit) |

> **Note:** no panel counts the sticky workflow tasks that the matching service rejects because the
> SDK worker behind the sticky queue has stopped (the 10-second case in
> [section 2.3](#23-reading-the-event-history-when-a-workflow-task-is-recorded-as-started)). Those show up
> on the SDK side as cache misses and replays.

> **Note:** **[Task Write Throttle Count by Partition](../observability/dashboards/server/task-queue-partitions-readme.md#row-3--backlog-write-path-2-ceiling)** and
> **[Approximate Backlog Age by Partition](../observability/dashboards/server/task-queue-partitions-readme.md#row-1--per-partition-backlog-drain--writeread-detection)**
> are on the [Task Queue Partitions dashboard](../observability/dashboards/server/task-queue-partitions-readme.md), not the Temporal Server
> dashboard. They stay empty until the breakdown flags in its
> [prerequisites](../observability/dashboards/server/task-queue-partitions-readme.md#prerequisites--enable-the-breakdown-flags) are turned on.

The next step of a wake-up is the read of the event history: when the matching service pairs a
workflow task with a poll, it asks the history service to record the workflow task as started, and
the history service reads the workflow's event history from the database.
[Section 4.3](#43-metrics-from-reading-the-event-history-in-the-database) covers the metrics that show how many
of those reads there are and how long they take.

### 4.3 Metrics from reading the event history in the database

The event history reads of
[section 2.3](#23-reading-the-event-history-when-a-workflow-task-is-recorded-as-started) show on these panels:

| What to look at | Normal at the wake-up time | Worth acting on | Related sections and alerts |
|---|---|---|---|
| Reads of the full event history: the `ReadHistoryBranch` persistence operation (or the `ReadRawHistoryBranch` persistence operation, when `history.sendRawHistoryBetweenInternalServices` is set to true) on **[Persistence Requests Total per Operation](../observability/dashboards/server/temporal-server-readme.md#3-persistence-requests-latencies-and-errors)**, and their latency on **[Persistence Latencies](../observability/dashboards/server/temporal-server-readme.md#3-persistence-requests-latencies-and-errors)** | Reads rise with the burst, latency stays close to normal | Latency rises with the reads, or the database's storage read I/O is at its limit during the burst | [5.1](#51-continue-as-new-before-the-long-sleep-workflow-implementation), [5.3](#53-keep-payloads-small-workflow-implementation), [5.5](#55-cluster-limits-and-database-capacity-cluster); [alert 11](../observability/alerts/server/alerts-index.md#alert-11--persistence-latency-high), [alert 13](../observability/alerts/server/alerts-index.md#alert-13--persistence-qps-approaching-limit) |

**Also check your database's own monitoring at the wake-up time.** The panels above show the
event history reads from the history service's side. Your database's own monitoring, outside the
Temporal dashboards, shows what those reads cost the database. A burst of reads of the full event
history shows there as:

- **Storage read I/O at or near its limit** (read operations per second or read throughput,
  whichever your storage limits) for the minutes around the wake-up time.
- **Database CPU staying low** at the same time. The database is waiting for data to come off the
  storage, not computing.

Most of these reads come from storage rather than the database's memory, because event history
rows written hours or days before the wake-up are usually no longer in its memory cache. Storage
read I/O at its limit with low CPU means the storage limits how fast the wake-ups can read their
event histories.

The next step of a wake-up is on the SDK worker: it replays the event history it was sent, and
holds a workflow task slot while it does. [Section 4.4](#44-event-history-replay-and-task-slot-metrics-on-sdk-workers) covers the
metrics that show how long the replays take and whether the SDK workers run out of slots.

### 4.4 Event history replay and task slot metrics on SDK workers

The event history replays and task slots of
[section 2.4](#24-event-history-replay-and-task-slots-on-sdk-workers) show on these panels. The
links go to the [Go SDK dashboard readme](../observability/dashboards/sdk/temporal-sdk-go-readme.md); the Java (Micrometer and OTel)
and Core SDK dashboards, listed under [Dashboards](#dashboards), have panels with the same titles
in the same groups.

| What to look at | Normal at the wake-up time | Worth acting on | Related sections and alerts |
|---|---|---|---|
| Time until the matching service pairs a workflow task with an SDK worker's poll, on **[Workflow Task Schedule To Start Latency](../observability/dashboards/sdk/temporal-sdk-go-readme.md#5-schedule-to-start-latencies)** (**Schedule To Start Latencies** group) | Rises at the wake-up time and drops back within minutes. It measures the same wait as **[Schedule to Start Latencies](../observability/dashboards/server/temporal-server-readme.md#14-sdk-workers-info)** on the server dashboard, split by task queue | Stays high after the burst, or is high on some task queues while their backlog stays small: the SDK workers on those task queues do not have enough pollers or free slots, and more partitions would not help | [5.4](#54-keep-sdk-workers-running-and-size-their-cache-sdk-workers); [Changing Task Queue Partitions](./change-task-queue-partitions.md#recommendations-before-you-increase) |
| Sticky workflow tasks for workflows already evicted from the worker in-memory cache, so the SDK worker fetches the full event history, on **[Sticky Cache Miss](../observability/dashboards/sdk/temporal-sdk-go-readme.md#3-worker-cache)** (**Worker Cache** group) | Rises at the wake-up time after sleeps of many hours | Also high for short sleeps during the day: the worker in-memory cache is too small, or SDK workers were restarted | [5.4](#54-keep-sdk-workers-running-and-size-their-cache-sdk-workers) |
| Running workflows evicted from the worker in-memory cache because it is full, on **[Sticky Cache Size](../observability/dashboards/sdk/temporal-sdk-go-readme.md#3-worker-cache)** and **[Sticky Cache Forced Evictions](../observability/dashboards/sdk/temporal-sdk-go-readme.md#3-worker-cache)** (**Worker Cache** group) | Cache size at its limit | A steady rate of forced evictions, which cause the worker in-memory cache misses in the row above (Go SDK; see the note below for Java) | [5.4](#54-keep-sdk-workers-running-and-size-their-cache-sdk-workers) |
| Time spent replaying the event history, on **[Workflow Task Replay Latency](../observability/dashboards/sdk/temporal-sdk-go-readme.md#6-workflow-task-info)** (**Workflow Task Info** group) | Rises at the wake-up time | Getting close to the workflow task timeout (10 seconds by default) | [5.1](#51-continue-as-new-before-the-long-sleep-workflow-implementation), [5.3](#53-keep-payloads-small-workflow-implementation) |
| Available workflow task slots on the SDK worker, on **[Worker Task Slots Available](../observability/dashboards/sdk/temporal-sdk-go-readme.md#2-worker-lifecycle)** (**Worker Lifecycle** group) | Dips during the burst | Stays at zero for a long period, not just for a moment at the wake-up time | [5.4](#54-keep-sdk-workers-running-and-size-their-cache-sdk-workers) |
| History page requests throttled: `GetWorkflowExecutionHistory` with `RESOURCE_EXHAUSTED` on **[Request Failures](../observability/dashboards/sdk/temporal-sdk-go-readme.md#1-grpc-requests-and-request-latencies)** (**gRPC Requests and Request Latencies** group), and `RpsLimit` on **[Resource Exhausted with Cause](../observability/dashboards/server/temporal-server-readme.md#6-throttling-and-limits)** | None | Any during the burst | [5.5](#55-cluster-limits-and-database-capacity-cluster); [alert 5](../observability/alerts/server/alerts-index.md#alert-5--namespace-rps-approaching-limit), [alert 6](../observability/alerts/server/alerts-index.md#alert-6--namespace-rps-over-limit) |
| Workflow tasks timing out on the SDK worker, on **[Workflow Task StartToClose Timeouts per Namespace (sticky tq)](../observability/dashboards/server/temporal-server-readme.md#14-sdk-workers-info)** | None | Any at the wake-up time: each timeout means another read of the full event history | [5.1](#51-continue-as-new-before-the-long-sleep-workflow-implementation), [5.5](#55-cluster-limits-and-database-capacity-cluster); [alert 56](../observability/alerts/server/alerts-index.md#alert-56--workflow-task-timeout-sticky) |

Two differences between SDKs matter when reading the SDK panels:

- **Go SDK:** **[Sticky Cache Forced Evictions](../observability/dashboards/sdk/temporal-sdk-go-readme.md#3-worker-cache)**
  counts every time the cache drops a running workflow to make room, so a steady non-zero rate means
  the cache is smaller than the number of workflows that SDK worker is serving.
- **Java SDK:** **[Sticky Cache Forced Evictions](../observability/dashboards/sdk/temporal-sdk-java-micrometer-readme.md#3-worker-cache)** on the
  Java dashboards counts workflows dropped to free a workflow thread, and workflows removed from the
  cache when they complete or fail. It does not count workflows dropped because the cache is full,
  so read **[Sticky Cache Miss](../observability/dashboards/sdk/temporal-sdk-java-micrometer-readme.md#3-worker-cache)** instead.

[Sections 4.1 to 4.4](#41-timer-task-metrics-in-the-history-service) cover what to read during a
wake-up time, while the timers are firing. Comparing one wake-up time with the ones before it shows
whether the pressure from the timer firings is growing as the sleep-and-wake pattern grows.
[Section 4.5](#45-trends-that-show-the-wake-up-bursts-growing) covers the trends to watch for that.

### 4.5 Trends that show the wake-up bursts growing

Keep watching the panels in the table below as the sleep-and-wake pattern grows. Comparing the same
wake-up window over time shows whether the next bursts will put on more pressure:

| Trend over time | Panel | What a rise means | Related sections and alerts |
|---|---|---|---|
| Event histories getting longer or larger | **[Workflow History Size](../observability/dashboards/server/temporal-server-readme.md#12-workflow-execution-history-info)** and **[Workflow History Event Count](../observability/dashboards/server/temporal-server-readme.md#12-workflow-execution-history-info)**. These panels combine several recordings of history size, so read them as a trend, not as the size of any one workflow | Each read and replay of the full event history is getting larger ([2.3](#23-reading-the-event-history-when-a-workflow-task-is-recorded-as-started), [2.4](#24-event-history-replay-and-task-slots-on-sdk-workers)) | [5.1](#51-continue-as-new-before-the-long-sleep-workflow-implementation), [5.3](#53-keep-payloads-small-workflow-implementation); [alert 42](../observability/alerts/server/alerts-index.md#alert-42--workflow-history-size-warning) |
| The workflows' current state getting larger | **[Mutable State Size](../observability/dashboards/server/temporal-server-readme.md#12-workflow-execution-history-info)**, **[Search Attributes Size](../observability/dashboards/server/temporal-server-readme.md#12-workflow-execution-history-info)** and **[Memo Size](../observability/dashboards/server/temporal-server-readme.md#12-workflow-execution-history-info)** | Each reload of the current state at wake-up is getting larger ([2.1](#21-timer-tasks-in-the-history-service)) | [5.3](#53-keep-payloads-small-workflow-implementation); [alert 46](../observability/alerts/server/alerts-index.md#alert-46--mutable-state-size-warning) |
| A higher spike of timer tasks at the wake-up time | **[Total Timer Tasks Processed](../observability/dashboards/server/temporal-server-readme.md#10-history-timer-task-info)** and **[Timer Tasks Processed by Type](../observability/dashboards/server/temporal-server-readme.md#10-history-timer-task-info)** | More timers fire together: more workflows, or more of them set to the same time ([2.1](#21-timer-tasks-in-the-history-service)) | [5.2](#52-spread-the-wake-up-times-workflow-implementation), [5.5](#55-cluster-limits-and-database-capacity-cluster) |
| More reads of the full event history at the wake-up time | **[Persistence Requests Total per Operation](../observability/dashboards/server/temporal-server-readme.md#3-persistence-requests-latencies-and-errors)** (the `ReadHistoryBranch` persistence operation) and **[Persistence Latencies](../observability/dashboards/server/temporal-server-readme.md#3-persistence-requests-latencies-and-errors)** | More wake-ups miss the sticky queue or the worker in-memory cache, or event histories are larger ([2.3](#23-reading-the-event-history-when-a-workflow-task-is-recorded-as-started)) | [5.1](#51-continue-as-new-before-the-long-sleep-workflow-implementation), [5.4](#54-keep-sdk-workers-running-and-size-their-cache-sdk-workers) |
| More sticky fallbacks at the wake-up time | **[Workflow Task Schedule-to-Start Timeouts (sticky fallback)](../observability/dashboards/server/temporal-server-readme.md#14-sdk-workers-info)** | SDK workers are falling further behind during the burst, so more wake-ups read the full event history ([2.4](#24-event-history-replay-and-task-slots-on-sdk-workers)) | [5.4](#54-keep-sdk-workers-running-and-size-their-cache-sdk-workers) |
| A backlog appearing or growing at the wake-up time | **[Approximate Task Backlog](../observability/dashboards/server/temporal-server-readme.md#14-sdk-workers-info)** and **[Tasks Persisted to DB (can indicate task backlog)](../observability/dashboards/server/temporal-server-readme.md#14-sdk-workers-info)** | SDK workers or task queue partitions are not keeping up with the burst ([2.2](#22-workflow-tasks-in-the-matching-service)) | [5.4](#54-keep-sdk-workers-running-and-size-their-cache-sdk-workers), [5.5](#55-cluster-limits-and-database-capacity-cluster) |

With the metrics to watch identified,
[section 5](#5-recommendations-for-running-the-sleep-and-wake-pattern-at-large-scale) gives the
recommendations that reduce the pressure they show.

## 5. Recommendations for running the sleep-and-wake pattern at large scale

Every recommendation in this section keeps the sleep-and-wake pattern itself: the workflows still
sleep and wake on time. The recommendations are ways to improve an implementation you may already
run, not a rewrite of it. Each recommendation stands on its own, so check your workflows, SDK
workers and cluster against them and adopt the ones that fit.

**In the workflow implementation:**

- **[5.1 Continue-as-new before the long sleep](#51-continue-as-new-before-the-long-sleep-workflow-implementation):** each
  wake-up reads and replays a short event history instead of the workflow's whole past
  ([sections 2.3](#23-reading-the-event-history-when-a-workflow-task-is-recorded-as-started) and [2.4](#24-event-history-replay-and-task-slots-on-sdk-workers)). It removes the most database reads, so it comes first.
- **[5.2 Spread the wake-up times](#52-spread-the-wake-up-times-workflow-implementation):** fewer workflows wake at the
  same moment, which lowers the peak at every step of a wake-up
  ([sections 2.1](#21-timer-tasks-in-the-history-service) to [2.4](#24-event-history-replay-and-task-slots-on-sdk-workers)).
- **[5.3 Keep payloads small](#53-keep-payloads-small-workflow-implementation):** every read of the event history and
  of the workflow's current state is smaller ([sections 2.1](#21-timer-tasks-in-the-history-service), [2.3](#23-reading-the-event-history-when-a-workflow-task-is-recorded-as-started) and
  [2.4](#24-event-history-replay-and-task-slots-on-sdk-workers)).

**For the SDK workers:**

- **[5.4 Keep SDK workers running and size their cache](#54-keep-sdk-workers-running-and-size-their-cache-sdk-workers):**
  more wake-ups apply only the new events instead of reading the full event history
  ([sections 2.2](#22-workflow-tasks-in-the-matching-service) to [2.4](#24-event-history-replay-and-task-slots-on-sdk-workers)).

**For the cluster:**

- **[5.5 Cluster limits and database capacity](#55-cluster-limits-and-database-capacity-cluster):**
  the burst of wake-ups is kept from using capacity other namespaces need, and the database is
  sized to absorb it ([sections 2.1](#21-timer-tasks-in-the-history-service) to [2.4](#24-event-history-replay-and-task-slots-on-sdk-workers)).

**Recommendation: as the workload that runs the sleep-and-wake pattern grows, keep checking the
trends in [section 4.5](#45-trends-that-show-the-wake-up-bursts-growing)**, and act on a rising
trend before the wake-ups reach a limit of the cluster or the database.

### 5.1 Continue-as-new before the long sleep (workflow implementation)

**Recommendation: just before the workflow goes into its longest sleep, continue-as-new, and have
the new run start with the sleep.** Continue-as-new closes the current run and starts a new run of
the same workflow, with the same workflow ID, and passes the state the next run needs as its input.
The new run starts with an empty event history.

**Why continue-as-new helps.** The size of the event history at wake-up is one of the four things
that decide how much pressure a burst of wake-ups adds
([section 3](#3-how-much-pressure-the-timer-firings-add)), and for a long-running workflow it grows
with every cycle. Continuing-as-new before the long sleep keeps it to one cycle, which brings these
benefits:

- **Smaller database reads at each wake-up.** A wake-up that reads the full event history
  ([section 2.3](#23-reading-the-event-history-when-a-workflow-task-is-recorded-as-started)) reads one cycle of events
  instead of the workflow's whole past. In the
  [example workflow in section 2.3](#23-reading-the-event-history-when-a-workflow-task-is-recorded-as-started), that is
  about 30 KB and 110 events, a single page, instead of about 2 MB and 6,600 events, some 26 pages,
  by day 60.
- **Shorter event history replays on the SDK workers.** Replaying one cycle of events takes far less
  time, so each wake-up holds a workflow task slot for less time and stays well inside the workflow
  task timeout ([section 2.4](#24-event-history-replay-and-task-slots-on-sdk-workers)).
- **Missing the worker in-memory cache costs little.** A wake-up that misses the cache still reads
  the full event history, but that event history is now short. This is why continue-as-new removes
  more database reads than a larger worker in-memory cache can: it does not depend on the cache.

**How to see the change on the dashboards** from
[section 4](#4-observing-the-pressure-from-the-timer-firings-on-your-dashboards):

- **[Workflow History Size](../observability/dashboards/server/temporal-server-readme.md#12-workflow-execution-history-info)** and
  **[Workflow History Event Count](../observability/dashboards/server/temporal-server-readme.md#12-workflow-execution-history-info)**
  stop growing over time and level off at the size of one cycle
  ([section 4.5](#45-trends-that-show-the-wake-up-bursts-growing)).
- The `ReadHistoryBranch` persistence operation on
  **[Persistence Requests Total per Operation](../observability/dashboards/server/temporal-server-readme.md#3-persistence-requests-latencies-and-errors)**
  and **[Persistence Latencies](../observability/dashboards/server/temporal-server-readme.md#3-persistence-requests-latencies-and-errors)**
  shows fewer reads at the wake-up time, with lower latency, and the database's own monitoring
  shows less storage read I/O ([section 4.3](#43-metrics-from-reading-the-event-history-in-the-database)).
- **[Workflow Task Replay Latency](../observability/dashboards/sdk/temporal-sdk-go-readme.md#6-workflow-task-info)** drops at the wake-up
  time ([section 4.4](#44-event-history-replay-and-task-slot-metrics-on-sdk-workers)).

**How to apply continue-as-new:**

- **Keep the input to the next run small.** The input is stored in the new run's first event,
  which every replay of that run reads. Pass the state the next run needs, not the whole history of
  what came before.
- **Read pending signals before continuing-as-new.** If a signal arrives while the workflow task
  that continues-as-new is running, the server rejects that task and the SDK worker runs it again,
  now with the signal. Read any pending signals first.
- **Choose where to continue-as-new by the workflow's own rhythm.** Continuing-as-new before the
  longest sleep keeps each event history to one cycle and starts one new run per cycle.
  Continuing-as-new before every sleep keeps event histories shortest but starts a new run for
  every sleep.
- **Continue-as-new before the server suggests it.** For most workflows, the recommended time to
  continue-as-new is when the server suggests it: once an event history passes 4,096 events or
  4 MB, the server sets a continue-as-new suggested flag that the workflow reads from its workflow
  info. For the sleep-and-wake pattern at large scale, continue-as-new earlier, before the longest
  sleep. By the time the flag is set, every wake-up has been reading a large event history for a
  long time: the [example workflow in section 2.3](#23-reading-the-event-history-when-a-workflow-task-is-recorded-as-started) writes
  about 110 events a day, so it reaches 4,096 events only after about 37 days.
- **Adding continue-as-new to workflows that are already running.** The running workflows must
  still replay their existing event history the same way, so roll the change out with workflow
  versioning (patching) or Worker Versioning.

**What continue-as-new costs.** Continue-as-new does not add event history: the same events are
written, split into shorter runs. What it adds is one more run for every cycle:

- **One more run start and close.** Each continue-as-new writes the history events of both runs,
  then saves the old run's close and the new run's start together, in one transaction on SQL or one
  batch on Cassandra. It also adds a close and an open record in visibility. The new run starts with
  one workflow task, on the normal task queue, that sets its timer.
- **More closed runs to keep and clean up.** Every closed run stays in the persistence and
  visibility stores until the namespace's retention period ends, and then retention deletes it from
  both. With one closed run per cycle, the namespace keeps more closed runs within its retention
  period, and retention has more of them to delete. If archival is turned on for the namespace,
  each closed run is also archived.

**If the closed runs are not needed for long, consider lowering the namespace's retention
period.** The persistence and visibility stores then keep fewer closed runs at any time. Retention
still deletes one closed run per cycle.

The database trades these small writes per cycle for removing a large read at every wake-up.

Continue-as-new makes each wake-up read little.
[Section 5.2](#52-spread-the-wake-up-times-workflow-implementation) spreads the wake-ups themselves,
which lowers the peak that the history service, the matching service and the database all have to
absorb.

### 5.2 Spread the wake-up times (workflow implementation)

**Recommendation: add a random extra time to each workflow's sleep.** If a workflow would sleep for
a given time, have it sleep for that time plus a random extra time, for example between 1 and 30
minutes.

**Why spreading the wake-up times helps.** How many workflows wake close to each other is one of
the four things that decide how much pressure a burst of wake-ups adds
([section 3](#3-how-much-pressure-the-timer-firings-add)), and it adds pressure at every step of a
wake-up. Spreading the same wake-ups over a longer period instead of all at once brings these
benefits:

- **A lower peak of timer tasks** in the history service ([section 2.1](#21-timer-tasks-in-the-history-service)).
- **A smaller backlog in the matching service.** Workflow tasks arrive more slowly, so more of them
  are paired with a poll straight away ([section 2.2](#22-workflow-tasks-in-the-matching-service)).
- **A lower peak of event history reads** on the database ([section 2.3](#23-reading-the-event-history-when-a-workflow-task-is-recorded-as-started)).
- **Fewer event history replays at once** on the SDK workers, so fewer workflow task slots are in
  use at the same time ([section 2.4](#24-event-history-replay-and-task-slots-on-sdk-workers)).

The database's storage read I/O and the SDK workers' capacity have to cover the peak, not the
average, so a lower peak lets both be smaller.

**How to see the change on the dashboards** from
[section 4](#4-observing-the-pressure-from-the-timer-firings-on-your-dashboards):

- The spike on **[Timer Tasks Processed by Type](../observability/dashboards/server/temporal-server-readme.md#10-history-timer-task-info)** at the
  wake-up time is lower and wider
  ([section 4.1](#41-timer-task-metrics-in-the-history-service)).
- **[Schedule to Start Latencies](../observability/dashboards/server/temporal-server-readme.md#14-sdk-workers-info)** and
  **[Approximate Task Backlog](../observability/dashboards/server/temporal-server-readme.md#14-sdk-workers-info)** stay lower during the
  wake-ups ([section 4.2](#42-workflow-task-metrics-in-the-matching-service)).
- The peak of the `ReadHistoryBranch` persistence operation on
  **[Persistence Requests Total per Operation](../observability/dashboards/server/temporal-server-readme.md#3-persistence-requests-latencies-and-errors)**,
  and the peak of storage read I/O on the database's own monitoring, are lower
  ([section 4.3](#43-metrics-from-reading-the-event-history-in-the-database)).
- **[Worker Task Slots Available](../observability/dashboards/sdk/temporal-sdk-go-readme.md#2-worker-lifecycle)** stays above zero
  ([section 4.4](#44-event-history-replay-and-task-slot-metrics-on-sdk-workers)).

**How to spread the wake-up times:**

- **The language's own random number generator is fine for the sleep time.** When the SDK worker
  replays the event history, it matches each timer to the one already recorded by the timer's ID,
  not by its duration, and the server keeps the timer as it was first started. So a different
  random value on replay does not change when the workflow wakes. Use the random value only for the
  sleep time: if it also decides anything else in the workflow, use a replay-safe source, such as
  `Workflow.newRandom()` in Java.
- **For start timers, add the same random extra time to the start delay.** The code that starts
  the workflow chooses the start delay: if the delay would be a given time, set it to that time
  plus a random extra time, as for the sleeps.
- **For workflows using the cron start option, there is no good way to add a random extra time.**
  The cron start option has no setting for it, and the times in a cron expression usually come from
  the business logic, so they cannot simply be changed at random. Schedules, which can replace the
  cron start option, do support a random extra time (jitter) on every run. Moving to Schedules is a
  possibility to consider, not a requirement: many schedules add load of their own to the cluster.

**What spreading the wake-up times costs.** Each workflow wakes a little later than it would have,
by up to the largest random extra time you choose, so pick a range the use case can accept. The
cluster does no extra work: the timers fire over a longer period instead of all at once.

Fewer wake-ups at once lowers the peak.
[Section 5.3](#53-keep-payloads-small-workflow-implementation) makes each event in an event history
smaller, by keeping its payloads small.

### 5.3 Keep payloads small (workflow implementation)

**Recommendation: keep activity inputs and results, search attributes and memo small.** Every
activity's input is stored in its scheduled event and its result in its completed event. Search
attributes and memo are part of the workflow's current state.

**Why keeping payloads small helps.** Two of the four things that decide how much pressure a burst
of wake-ups adds are the size of the event history and the size of the workflow's current state
([section 3](#3-how-much-pressure-the-timer-firings-add)). Smaller payloads make both smaller,
whether or not the workflow continues-as-new, which brings these benefits:

- **Smaller database reads at each wake-up.** A wake-up that reads the full event history
  ([section 2.3](#23-reading-the-event-history-when-a-workflow-task-is-recorded-as-started)) reads fewer bytes, because
  every activity input and result in it is smaller.
- **Shorter event history replays on the SDK workers.** Less data to fetch and decode means each
  replay finishes sooner ([section 2.4](#24-event-history-replay-and-task-slots-on-sdk-workers)).
- **Smaller reads of the workflow's current state.** After a sleep of more than an hour, the history
  service reads the current state back from the database, and search attributes and memo are part
  of it ([section 2.1](#21-timer-tasks-in-the-history-service)).

**How to see the change on the dashboards** from
[section 4](#4-observing-the-pressure-from-the-timer-firings-on-your-dashboards):

- **[Workflow History Size](../observability/dashboards/server/temporal-server-readme.md#12-workflow-execution-history-info)** grows more slowly,
  while **[Workflow History Event Count](../observability/dashboards/server/temporal-server-readme.md#12-workflow-execution-history-info)** does
  not change: the same events, each event smaller
  ([section 4.5](#45-trends-that-show-the-wake-up-bursts-growing)).
- **[Mutable State Size](../observability/dashboards/server/temporal-server-readme.md#12-workflow-execution-history-info)**,
  **[Search Attributes Size](../observability/dashboards/server/temporal-server-readme.md#12-workflow-execution-history-info)** and
  **[Memo Size](../observability/dashboards/server/temporal-server-readme.md#12-workflow-execution-history-info)** are lower
  ([section 4.5](#45-trends-that-show-the-wake-up-bursts-growing)).
- **[Persistence Latencies](../observability/dashboards/server/temporal-server-readme.md#3-persistence-requests-latencies-and-errors)** for the
  `ReadHistoryBranch` persistence operation, and storage read I/O on the database's own monitoring,
  are lower at the wake-up time
  ([section 4.3](#43-metrics-from-reading-the-event-history-in-the-database)).
- **[Workflow Task Replay Latency](../observability/dashboards/sdk/temporal-sdk-go-readme.md#6-workflow-task-info)** is lower at the wake-up
  time ([section 4.4](#44-event-history-replay-and-task-slot-metrics-on-sdk-workers)).

**How to keep payloads small:**

- **Pass identifiers instead of whole objects.** An activity that receives a record's ID and loads
  the record itself stores a few bytes in the event history instead of the whole record.
- **Do not repeat data inside one payload.** A step configuration included twice in the same input
  doubles that part of every scheduled event.
- **Return only what the workflow uses** from an activity.
- **Keep search attributes and memo to what you search or display.**
- **Compress payloads.** The SDKs accept a payload codec that runs on the client and worker side
  before data reaches the server, and the Go and Java SDKs include a zlib codec. The Go SDK's zlib
  codec only stores the compressed form when it is smaller, and payloads written before it was
  added still decode, so it can be added to running workflows. How much it saves depends on the
  payloads, so measure it on yours.
- **Store large payloads outside Temporal.** Recent Go and Java SDKs can store large payloads in an
  external store, such as Amazon S3 or Google Cloud Storage, and put only a small reference to each
  one in the event history. By default this applies to payloads of 256 KiB and larger. The feature
  is experimental: Go SDK v1.42.0 and later, with ready-made S3 and GCS drivers; Java SDK v1.36.0
  and later, where you provide the driver for your store.

**What keeping payloads small costs.** Nothing on the cluster. The costs are in the application:

- **Passing identifiers** moves the reads of the full record to the activities, against your own
  data store.
- **A payload codec** uses CPU on the clients and SDK workers. Every client and SDK worker must use
  the same codec, and the UI needs a codec server to show the payloads.
- **External payload storage** moves the reads of those payloads from the Temporal database to the
  external store: an SDK worker that replays the event history fetches them from there. Every
  client and SDK worker needs access to the store, and Temporal does not delete the stored payloads
  when retention deletes the run, so the store needs its own expiry.

Smaller payloads make every read of an event history lighter.
[Section 5.4](#54-keep-sdk-workers-running-and-size-their-cache-sdk-workers) covers the SDK workers
that do the replays.

### 5.4 Keep SDK workers running and size their cache (SDK workers)

**Recommendation: where you can, keep the SDK workers running while the workflows sleep and
through the wake-up window, instead of scaling them down.**

**If you cannot keep the SDK workers running, continue-as-new before the long sleep
([section 5.1](#51-continue-as-new-before-the-long-sleep-workflow-implementation)) becomes the important
recommendation.** Stopping an SDK worker throws away its worker in-memory cache, so every workflow
it ran reads its full event history when it wakes. Continue-as-new keeps that event history short,
so losing the cache costs little. Adding SDK workers does not lose any cache: the server keeps
routing each workflow to the SDK worker that last ran it.

**Why keeping SDK workers running helps.** Whether the SDK worker still holds the workflow in its
worker in-memory cache is one of the four things that decide how much pressure a burst of wake-ups
adds ([section 3](#3-how-much-pressure-the-timer-firings-add)). An SDK worker that keeps running
keeps its cache and keeps polling its sticky queue, which brings these benefits:

- **More wake-ups read only the new events.** A wake-up on the sticky queue of a running SDK worker
  that still holds the workflow reads a few rows instead of the full event history
  ([section 2.3](#23-reading-the-event-history-when-a-workflow-task-is-recorded-as-started)).
- **Fewer full event history replays** on the SDK workers
  ([section 2.4](#24-event-history-replay-and-task-slots-on-sdk-workers)).
- **Fewer sticky workflow tasks moved to the normal task queue.** When an SDK worker stops, its
  sticky queue is no longer polled, and every wake-up for its workflows goes to the normal task
  queue with a read of the full event history
  ([section 2.3](#23-reading-the-event-history-when-a-workflow-task-is-recorded-as-started)).

**How to see the change on the dashboards** from
[section 4](#4-observing-the-pressure-from-the-timer-firings-on-your-dashboards):

- **[Sticky Cache Miss](../observability/dashboards/sdk/temporal-sdk-go-readme.md#3-worker-cache)** is lower at the wake-up time
  ([section 4.4](#44-event-history-replay-and-task-slot-metrics-on-sdk-workers)).
- **[Workflow Task Schedule-to-Start Timeouts (sticky fallback)](../observability/dashboards/server/temporal-server-readme.md#14-sdk-workers-info)**
  stays close to zero ([section 4.2](#42-workflow-task-metrics-in-the-matching-service)).
- The `ReadHistoryBranch` persistence operation on
  **[Persistence Requests Total per Operation](../observability/dashboards/server/temporal-server-readme.md#3-persistence-requests-latencies-and-errors)**
  shows fewer reads at the wake-up time
  ([section 4.3](#43-metrics-from-reading-the-event-history-in-the-database)).

**How to tune the SDK workers you run.** If the metrics in
[section 4.4](#44-event-history-replay-and-task-slot-metrics-on-sdk-workers) show cache misses or
workflow task slots running out at the wake-up time, tune the worker in-memory cache size and the
number of workflow task slots on the SDK workers you already run, within their memory limits.

**What keeping SDK workers running costs.** SDK workers kept running while the workflows sleep use
compute during hours when they have little to do, and a larger worker in-memory cache needs more
memory on every SDK worker. That cost may not be acceptable for your
use case. If it is not, scale the SDK workers down and apply continue-as-new before the long sleep
([section 5.1](#51-continue-as-new-before-the-long-sleep-workflow-implementation)) at least: it keeps the
event history each wake-up reads short, with or without the cache.

The workflow and SDK worker recommendations are now covered.
[Section 5.5](#55-cluster-limits-and-database-capacity-cluster) covers the cluster: its limits, and
the database while the workflow changes roll out.

### 5.5 Cluster limits and database capacity (cluster)

**Recommendation: where the metrics in
[section 4](#4-observing-the-pressure-from-the-timer-firings-on-your-dashboards) show the cluster or
the database reaching a limit during the wake-ups, tune the part they point to:**

- **Timer tasks backing up in the history service** ([section 4.1](#41-timer-task-metrics-in-the-history-service)):
  tune the history service's task processing with [History Task Processing](./history-task-processing-tuning.md).
  Give the namespace that runs the sleep-and-wake pattern its own task processing limit: the task
  scheduler's rate limiter is off by default (`history.taskSchedulerEnableRateLimiter`), and
  [History Task Processing](./history-task-processing-tuning.md#5-sizing-the-limits-and-turning-them-on) covers how to turn it on
  and size it.
- **Database calls refused or slow** ([sections 4.1](#41-timer-task-metrics-in-the-history-service) and
  [4.3](#43-metrics-from-reading-the-event-history-in-the-database)): set the history service's database call limits, per pod and
  per namespace, to what your database can answer, with
  [History Persistence QPS Limits](./history-persistence-qps-limits.md#12-set-them-deliberately--do-not-rely-on-the-defaults).
  The per-namespace limit, `history.persistenceNamespaceMaxQPS`, is not set by default. Limits set
  higher than the database can answer let the wake-ups push it into timeouts, and tasks that keep
  timing out end up in the dead-letter queue. Limits that fit make some of the namespace's wake-ups
  wait instead: throttled tasks are retried until they get through, and are never dead-lettered.
- **Workflow tasks backing up in the matching service** ([section 4.2](#42-workflow-task-metrics-in-the-matching-service)):
  check that the workflow task queue has enough partitions for the wake-up peak, with
  [Changing Task Queue Partitions](./change-task-queue-partitions.md#when-to-increase-partitions). Partitions matter most for
  workflows that wake on the normal task queue: workflows started with a start delay or the cron
  start option, and workflows whose SDK workers were restarted or scaled down while they slept.
- **The SDK workers' requests refused by the frontend service** ([sections 4.2](#42-workflow-task-metrics-in-the-matching-service)
  and [4.4](#44-event-history-replay-and-task-slot-metrics-on-sdk-workers)): `RpsLimit` on `GetWorkflowExecutionHistory` or
  `ConcurrentLimit` on the poll APIs in
  **[Resource Exhausted with Cause](../observability/dashboards/server/temporal-server-readme.md#6-throttling-and-limits)** means the namespace's
  frontend limits are too low for the wake-ups. Raise `frontend.namespaceRPS` (history page
  requests have a lower priority than starts, signals and task completions, so they are throttled
  before those) or `frontend.namespaceCount` (open polls grow with the number of SDK workers and
  their pollers) for the namespace.
- **The database itself** ([section 4.3](#43-metrics-from-reading-the-event-history-in-the-database)): check your database's own
  metrics at the wake-up time, especially storage read I/O and CPU. Storage read I/O at its limit
  with low CPU means the database needs more storage read I/O, not more CPU. Keep that storage read
  I/O until [sections 5.1 to 5.3](#51-continue-as-new-before-the-long-sleep-workflow-implementation) are deployed and running
  workflows have passed their next continue-as-new point, then reduce it while watching the same
  metrics.

**Why tuning the cluster's limits and the database's capacity helps.** How many wake-ups make a
burst large depends on the cluster and its database
([section 3](#3-how-much-pressure-the-timer-firings-add)). Limits and capacity tuned for the
wake-ups bring these benefits:

- **Fewer noisy-neighbor problems on a cluster with many namespaces.** Per-namespace limits in the
  history service keep one namespace's wake-ups from taking the task processing and database
  capacity that other namespaces need ([section 2.1](#21-timer-tasks-in-the-history-service)).
- **No workflow is stuck in the dead-letter queue.** A database that is not pushed past what it can
  answer does not time out, so no task keeps failing until it is dead-lettered
  ([section 3](#3-how-much-pressure-the-timer-firings-add)).
- **The frontend service does not throttle the SDK workers during the wake-ups.** With frontend
  limits that fit the wake-ups, the SDK workers' requests for history pages and their polls for
  workflow tasks are not refused with `RESOURCE_EXHAUSTED`
  ([sections 2.2](#22-workflow-tasks-in-the-matching-service) and [2.4](#24-event-history-replay-and-task-slots-on-sdk-workers)).
- **Workflow tasks do not back up** in the task queue's partitions
  ([section 2.2](#22-workflow-tasks-in-the-matching-service)).
- **Reads of the full event history finish quickly**, with storage read I/O to spare
  ([section 2.3](#23-reading-the-event-history-when-a-workflow-task-is-recorded-as-started)).

**How to see the change on the dashboards**, for each benefit above:

- **Fewer noisy-neighbor problems:**
  **[Task Scheduler Throttling by Namespace](../observability/dashboards/server/temporal-server-readme.md#9-shard-queue-health)** and
  **[Rejected Database Calls by Namespace](../observability/dashboards/server/temporal-server-readme.md#3-persistence-requests-latencies-and-errors)**
  show the throttling on the namespace that runs the sleep-and-wake pattern, at its own limits, and
  not on the other namespaces.
- **No workflow stuck in the dead-letter queue:**
  **[Dead-Lettered Tasks — Execution-Stranding (page-worthy)](../observability/dashboards/server/temporal-server-readme.md#20-history-task-dlq--terminal-failures)**
  stays empty. **[Task Retry Depth — Stranding Types (approaching DLQ)](../observability/dashboards/server/temporal-server-readme.md#20-history-task-dlq--terminal-failures)**
  does not climb toward 70 while database calls time out
  ([section 4.1](#41-timer-task-metrics-in-the-history-service)).
- **The frontend service does not throttle the SDK workers:**
  **[Resource Exhausted with Cause](../observability/dashboards/server/temporal-server-readme.md#6-throttling-and-limits)** shows no `RpsLimit` on
  `GetWorkflowExecutionHistory` and no `ConcurrentLimit` on `PollWorkflowTaskQueue` or
  `PollActivityTaskQueue`, and
  **[Request Failures](../observability/dashboards/sdk/temporal-sdk-go-readme.md#1-grpc-requests-and-request-latencies)** on the SDK dashboards shows
  no `RESOURCE_EXHAUSTED` on `GetWorkflowExecutionHistory`. The namespace's request rate stays
  below its limit on
  **[Actual RPS vs Namespace Host RPS Limit](../observability/dashboards/server/temporal-server-readme.md#6-throttling-and-limits)**, open polls stay below
  `frontend.namespaceCount` on
  **[Max Concurrent Pollers per Frontend Pod](../observability/dashboards/server/temporal-server-readme.md#15-pollers)**, and
  **[Workflow Task StartToClose Timeouts per Namespace (sticky tq)](../observability/dashboards/server/temporal-server-readme.md#14-sdk-workers-info)** stays at zero
  ([sections 4.2](#42-workflow-task-metrics-in-the-matching-service) and [4.4](#44-event-history-replay-and-task-slot-metrics-on-sdk-workers)).
- **Workflow tasks do not back up:**
  **[Approximate Task Backlog](../observability/dashboards/server/temporal-server-readme.md#14-sdk-workers-info)** and
  **[Schedule to Start Latencies](../observability/dashboards/server/temporal-server-readme.md#14-sdk-workers-info)** drop back within minutes of
  the wake-up time, and
  **[Task Write Throttle Count by Partition](../observability/dashboards/server/task-queue-partitions-readme.md#row-3--backlog-write-path-2-ceiling)** and
  **[Approximate Backlog Age by Partition](../observability/dashboards/server/task-queue-partitions-readme.md#row-1--per-partition-backlog-drain--writeread-detection)**
  stay low ([section 4.2](#42-workflow-task-metrics-in-the-matching-service)).
- **Reads of the full event history finish quickly:**
  **[Persistence Latencies](../observability/dashboards/server/temporal-server-readme.md#3-persistence-requests-latencies-and-errors)** stays close to
  normal at the wake-up time ([section 4.3](#43-metrics-from-reading-the-event-history-in-the-database)). For the database itself,
  check that storage read I/O stays below its limit on your database's own monitoring.

**Two database-side changes do not help with many wake-ups at once:**

- **A read replica (MySQL and PostgreSQL).** The server connects to one database address per store
  (`connectAddr` in the SQL persistence configuration), so a read replica gets no reads from
  Temporal. Pointing the server at the replica instead is not safe either: the history service's
  reads need to see its latest writes. Cassandra has no separate read replica to add: the server
  connects to the cluster's nodes (`hosts`), and the nodes that hold a row's copies serve its
  reads.
- **More database CPU.** The reads of the full event history wait on the storage, not on the
  processor.
