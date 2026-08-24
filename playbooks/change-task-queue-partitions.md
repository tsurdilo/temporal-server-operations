# Changing Task Queue Partitions Playbook

This playbook covers how to safely change the number of partitions behind a task queue on the matching service — raising or lowering the count — and, just as important, how to determine whether a partition change is actually needed in the first place.

---

## Table of Contents

- [References](#references)
- [Using the Dashboard with This Playbook](#using-the-dashboard-with-this-playbook)
- [What Are Task Queue Partitions?](#what-are-task-queue-partitions)
- [Which Matcher Are You On? (classic vs priority)](#which-matcher-are-you-on-classic-vs-priority)
- [The Rule for Changing Partition Counts](#the-rule-for-changing-partition-counts)
- [Scoping the Change: Namespace, Task Queue, or Task Type](#scoping-the-change-namespace-task-queue-or-task-type)
- [Defaults and Sizing](#defaults-and-sizing)
- [How to Increase Partitions](#how-to-increase-partitions)
  - [Required order](#required-order)
- [When to Increase Partitions](#when-to-increase-partitions)
  - [How a partition's throughput works](#how-a-partitions-throughput-works)
  - [What to watch on the dashboard](#what-to-watch-on-the-dashboard)
  - [Recommendations: before you increase](#recommendations-before-you-increase)
- [Adding Partitions Won't Drain an Existing Backlog](#adding-partitions-wont-drain-an-existing-backlog)
- [Monitoring After an Increase](#monitoring-after-an-increase)
- [How to Decrease Partitions](#how-to-decrease-partitions)
  - [Required order](#required-order-1)
- [When to Decrease Partitions](#when-to-decrease-partitions)
  - [You have more partitions than the workload needs](#you-have-more-partitions-than-the-workload-needs)
  - [You want a per-task-queue rate limit to be exact](#you-want-a-per-task-queue-rate-limit-to-be-exact)
- [Monitoring After a Decrease](#monitoring-after-a-decrease)
- [Signs of Trouble — Red Flags After a Change](#signs-of-trouble--red-flags-after-a-change)
- [Reading the Metrics](#reading-the-metrics)
  - [Backlog age can read 0 even when it isn't](#backlog-age-can-read-0-even-when-it-isnt)
  - [Activity queues can look empty: eager dispatch](#activity-queues-can-look-empty-eager-dispatch)
- [Detecting and Fixing an Existing `Write > Read` Misconfiguration](#detecting-and-fixing-an-existing-write--read-misconfiguration)
  - [How to confirm](#how-to-confirm)
  - [The fix](#the-fix)

---

## References

1. **[Temporal Task Queue Partitions dashboard](../observability/dashboards/server/task-queue-partitions-readme.md)** — the companion Grafana dashboard for this playbook: watch a single task queue's per-partition metrics while you change its partitions. Requires `metrics.breakdownByPartition` and `metrics.breakdownByTaskQueue` on for the queue(s) or namespace you're watching (they make the server emit the finer-grained per-partition metrics the dashboard reads). See **Turning on the per-partition metrics** just below the config table for sample YAML.

2. **[Server Overview dashboard](../observability/dashboards/server/temporal-server-readme.md)** — cluster-wide health. When you want the bigger picture, its Matching Task Queue Info and SDK Worker Info panels show the same signals (async match latency, task write throttle, approximate backlog, schedule-to-start) at cluster scale.

3. **Related server dynamic config** — none require a server restart after a change.

| Key | Default | Notes |
|---|---|---|
| `matching.numTaskqueueWritePartitions` | `4` | Number of write partitions **each** task queue gets (new tasks are written to these). Settable cluster-wide, per namespace, per task queue, or per task type — see [Scoping the Change](#scoping-the-change-namespace-task-queue-or-task-type). |
| `matching.numTaskqueueReadPartitions` | `4` | Number of read partitions **each** task queue gets (workers poll these). Settable cluster-wide, per namespace, per task queue, or per task type — see [Scoping the Change](#scoping-the-change-namespace-task-queue-or-task-type). |
| `metrics.breakdownByPartition` | `true` | Puts the real partition id in the `partition` tag. Required for per-partition backlog gauges. Settable per task queue — watch cardinality. |
| `metrics.breakdownByTaskQueue` | `true` | Puts the real task queue name in the `taskqueue` tag (Matching **and** History metrics). Also required for per-partition backlog gauges. Settable per task queue — watch cardinality. |
| `matching.backlogMetricsEmitInterval` | `1m` | Only relevant if you use Worker Versioning (v1.31.0+): how often the per-worker-version backlog breakdown is recomputed. Set `0` to turn that breakdown off. Otherwise ignore. |
| `matching.enablePollerScalingDecisionMetrics` | `false` | Opt-in. Emits `poller_scale_decision`, whose `reason="rate_limited"` series is the only per-partition signal for the sync-match / dispatch (path 1) ceiling on the default priority matcher. Enable it to populate the dashboard's **Dispatch Rate-Limiting by Partition** panel. Settable per task queue. |
| `admin.matchingNamespaceTaskqueueToPartitionDispatchRate` | `1000` | **Changing this is not recommended** — scale by adding partitions, not by raising the cap. Max dispatch qps per partition (for a namespace + task queue); caps both sync match and backlog dispatch. |
| `admin.matchingNamespaceToPartitionDispatchRate` | `10000` | **Changing this is not recommended.** The same per-partition cap, set at the namespace level (not a namespace-wide total across task queues). The effective per-partition limit is the **min** of this and the per-task-queue value above — so the `1000` binds by default. |

**Turning on the per-partition metrics.** The per-partition backlog gauges only emit when both `metrics.breakdownByPartition` and `metrics.breakdownByTaskQueue` are on (both default `true`; when off, the `partition` tag reads `__normal__` and `taskqueue` reads `__omitted__`). **If the dashboard shows no per-partition data, check these two first.**

Because both default to on, every task queue emits this per-partition detail out of the box. On a cluster with many task queues that's a lot of extra metrics for your monitoring system to store — and `metrics.breakdownByTaskQueue` adds the task-queue tag to History metrics too, not just Matching. If that overhead is a concern, a common pattern is to turn them **off cluster-wide and back on only for the one queue you're resizing** — you get the detail where you need it without the fleet-wide cost:

```yaml
metrics.breakdownByPartition:
  - value: false                    # default: off for every task queue
  - value: true                     # …except this one
    constraints:
      namespace: 'your-namespace'
      taskQueueName: 'your-task-queue'
metrics.breakdownByTaskQueue:
  - value: false
  - value: true
    constraints:
      namespace: 'your-namespace'
      taskQueueName: 'your-task-queue'
```

The order of the two entries doesn't matter — matching resolves the **most specific** constraint first, so the queue you named gets `true` and every other queue falls back to the `false` default. (Drop `taskQueueName` to scope a whole namespace instead. Each entry needs its own `value:` — a bare `- false` won't parse.)

**One more opt-in for the dashboard.** The dashboard's **Dispatch Rate-Limiting by Partition** panel reads `poller_scale_decision`, which matching emits only when `matching.enablePollerScalingDecisionMetrics` is on (default off) — so that panel stays empty until you enable it. It's the path-1 (sync-match / dispatch cap) signal on the default priority matcher; set it the same way (scope it to the queue you're resizing) when you want that panel populated.

4. **Alerting on a stuck partition backlog** — an optional per-queue alert (**Alert 83**) for a `Write > Read` stall or any non-draining per-partition backlog is documented in the [server alert index](../observability/alerts/server/alerts-index.md#alert-83--task-queue-partition-backlog-not-draining). It's not in the essential set — the threshold is workload-specific and the metric is opt-in per queue — so scope it to a queue you've instrumented and tune the threshold.

---

## Using the Dashboard with This Playbook

This playbook is meant to be read with the companion **[Task Queue Partitions dashboard](../observability/dashboards/server/task-queue-partitions-readme.md)** open on the exact queue you're changing — set its `namespace`, `taskqueue`, and `task_type` variables and keep it beside you. Nearly every signal below is a panel on it, called out by name where it matters, so at each step you're looking at a graph rather than guessing. (If the per-partition panels read empty, turn the metrics on — see [References](#references).)

The dashboard's rows line up with the work here:

| Dashboard row | Use it for |
|---|---|
| **Per-Partition Backlog** | Confirming a clean [drain during a decrease](#monitoring-after-a-decrease), watching [load spread after an increase](#monitoring-after-an-increase), the backlog read + dispatch ceiling, and [`Write > Read` detection](#detecting-and-fixing-an-existing-write--read-misconfiguration) |
| **Sync Match & Dispatch (Path 1)** | The [sync-match / dispatch-cap ceiling](#when-to-increase-partitions) |
| **Backlog Write (Path 2)** | The [write ceiling](#when-to-increase-partitions) — is matching persisting tasks fast enough? |
| **Rule-Outs** | Answering ["is matching even the bottleneck?"](#recommendations-before-you-increase) before you add partitions |

---

## What Are Task Queue Partitions?

A single task queue *name* is not one queue. Behind that name Temporal keeps a separate **workflow** task queue and **activity** task queue, each managed independently and each with its own partition count. (Workflow workers also get a per-worker **sticky** queue — a special case with no partitions, covered further below.)

The workflow and activity queues (the **normal** queues, as opposed to the per-worker sticky queue) are each split into several **partitions**. A partition is owned by one matching host at a time, and a single host owns many partitions across your task queues — matching distributes them across its hosts by consistent hashing, and reshuffles the affected ones when a host joins or leaves. Each partition has its own task backlog and its own dispatch rate limit. That is why partitions exist: a single partition would funnel the whole queue through one host and one rate-limit bucket, capping its throughput. Splitting it across several partitions spreads its tasks over multiple hosts and multiplies those rate-limit buckets, so a busy queue can scale. The default is 4 partitions per queue.

Tuning goes both ways. A high-throughput task queue can **add** partitions to raise its ceiling. **Reducing** partitions (down to 1) is just as valid, for two reasons. First, it frees matching-host resources that a quiet queue doesn't need. Second, a single partition is the only way to enforce a per-task-queue dispatch limit — like the SDK's `taskQueueActivitiesPerSecond` — exactly; with several partitions the limit is split across them and short-term dispatch can burst above the target (the math is in [Defaults and Sizing](#defaults-and-sizing)). This playbook covers changing the count safely in either direction.

A queue's partitions have two sides, sized **independently** — how many partitions new tasks are **written** to, and how many workers **read** (poll) for them. Both counts refer to the **same** numbered partitions, not two separate sets you add together (a queue with write count 4 and read count 4 has **4** partitions, not 8):

- **Write** — new tasks arriving for the queue are placed onto a partition.
- **Read** — workers poll a partition to pick tasks up. "Read" doesn't mean a single reader: many pollers can wait on the same partition at once, and each poll is spread across the queue's partitions by the frontend's matching-client load balancer, which routes it to the partition with the fewest outstanding pollers.

Each side is its own dynamic config:

| Config | Role |
|---|---|
| `matching.numTaskqueueWritePartitions` | The **write count** — how many partitions new tasks are placed on |
| `matching.numTaskqueueReadPartitions` | The **read count** — how many partitions workers poll for tasks |

Under normal operation the read and write counts should match — with 4 partitions, all 4 are written to and all 4 are polled. They are separate settings for one reason: when you *change* the partition count, you set them apart briefly so the change stays safe — the danger is writing tasks to partitions no worker is polling yet (see [The Rule for Changing Partition Counts](#the-rule-for-changing-partition-counts)). Deciding whether to change the count — and, when you do, changing it safely — is what the rest of this playbook covers. Both counts are dynamic config and take effect without a server restart.

Not every queue behind a task queue name is partitioned the same way. There are three kinds:

1. **Normal — the workflow and activity queues.** The shared queues workers poll, split into partitions (default 4). **These are what this playbook tunes** (everything above and below).
2. **Sticky — one per workflow worker.** The SDK creates it to route an execution's follow-up workflow tasks back to the worker caching its state. It has a **single, fixed partition** — the partition configs don't apply. In metrics it always appears under `partition="__sticky__"`, never a numbered partition (normal partitions show as `partition="0"`, `partition="1"`, and so on). If it backs up, the fix is worker-side — see below.
3. **Internal / system — including the per-namespace workers behind Schedules.** These default to a single partition, pinned by their internal name. Because that default is name-specific, a cluster-wide or per-namespace change to your own queues **won't affect them** (the name-specific default wins over a broader override). You *can* change one — e.g. the per-namespace Schedules queue — by targeting its exact internal name, but that's rarely necessary and normally best left alone.

**When a sticky queue backs up** — meaning a single worker can't keep up with the workflow tasks for the executions it has cached — the fix is worker-side, since the sticky task queue can't be partitioned. It's self-limiting, not a stuck state — the work falls back to the normal, partitioned workflow task queue, where any worker picks it up and replays the execution to rebuild state. There are two fallback paths. If the worker has stopped polling its sticky queue (no sticky poll in the last 10s), the server stops routing workflow tasks to sticky and schedules them on the normal queue directly (matching returns `StickyWorkerUnavailable`). If a task was already placed on sticky but isn't picked up within the sticky ScheduleToStart timeout (default 5s) — for example the worker just went down — it times out and is rescheduled on the normal queue. Recommended fixes, all worker-side:

- Add more workflow workers.
- Raise the worker's workflow-task concurrency (`MaxConcurrentWorkflowTaskExecutionSize`) and poller count.
- Increase the worker's sticky (workflow) cache size.

**You can see it happening** on either dashboard. The [Server Overview](../observability/dashboards/server/temporal-server-readme.md) dashboard graphs the fallback timeouts directly, in its *Workflow Task Schedule-to-Start Timeouts (sticky fallback)* panel. On the [companion dashboard](../observability/dashboards/server/task-queue-partitions-readme.md), pick `task_type = Workflow` — that view is the **normal** workflow queue. The workflow tasks that fall back from sticky get rescheduled onto it, so the normal queue's **Approximate Backlog** and **Schedule to Start Latency** panels climb.

---

## Which Matcher Are You On? (classic vs priority)

Matching has two task-dispatch implementations, and which one your cluster runs changes which metrics you get:

- **Classic matcher** — the original implementation.
- **Priority matcher** — adds priority-level dispatch (`matching.priorityLevels`). Selected by `matching.useNewMatcher`.

`matching.useNewMatcher` was **added in server v1.28.0** (opt-in) and **became the default in v1.31.0**:

- **v1.31.0+** — you are on the priority matcher unless you explicitly set `matching.useNewMatcher: false`. Turning it off returns you to the classic matcher — **not the recommended path**: the priority matcher is the default, and priority-level dispatch requires it. (`matching.enableFairness` is a further opt-in built on the priority matcher.)
- **v1.28.0–v1.30.x** — the classic matcher is the default; the priority matcher is available but off unless enabled.
- **before v1.28.0** — classic matcher only.

**What this means for this playbook.** The priority matcher doesn't emit some metrics the classic matcher did — most importantly **`sync_throttle_count`**. On a default (v1.31.0+) cluster it's absent, with no replacement, so it can't serve as a signal here — rely on the backlog signals (`approximate_backlog_*`, `asyncmatch_latency`) instead. The same applies to the [Matching Partition Sync Throttle Active](../observability/alerts/server/alerts-index.md#alert-74--matching-partition-sync-throttle-active) alert (`temporal-alert-074`): it keys on `sync_throttle_count`, so it only works on the classic matcher. Because it doesn't apply across all versions it's kept in the alerts index rather than the essential alert set — add it to your cluster's alerts if you run the classic matcher. Every other signal in this playbook is emitted regardless of matcher.

> The companion [Task Queue Partitions dashboard](../observability/dashboards/server/task-queue-partitions-readme.md) is **matcher-agnostic** — it uses base metric names that both matchers emit, so it works whichever matcher you're on.

---

## The Rule for Changing Partition Counts

Under normal operation you want the read count and write count **equal** — every partition being written to is also being polled. You only set them apart on purpose, while changing the partition count, and even then one rule holds:

> **`matching.numTaskqueueWritePartitions` must always be ≤ `matching.numTaskqueueReadPartitions` — not only before and after a change, but at every moment while you are applying one.**

In plain terms, move the two settings in whichever order keeps write ≤ read the whole time — which flips depending on direction:

- **Increasing** partitions → raise the **read** count first, then the write count.
- **Decreasing** partitions → lower the **write** count first (let the retiring partitions drain), then the read count.

**Quick reference — safe change order:**

```
RULE: write partitions ≤ read partitions — at every moment.

INCREASE  (e.g. 4 → 8)          DECREASE  (e.g. 8 → 4)
  read leads                      write leads
  1. raise READ                   1. lower WRITE
  2. raise WRITE                  2. drain retiring partitions
                                  3. lower READ
```

The reason is to avoid writing a task to a partition that no worker reads. Tasks are spread across the write partitions, but workers only poll the read partitions — so if there are more write partitions than read partitions, a task can land on a partition nobody is polling. It isn't lost: matching forwards that partition's backlog up to the root partition, which is always polled. But forwarding the task adds latency (it has to be moved before it can be picked up) and concentrates load on the root partition. Keeping `Write ≤ Read` guarantees every partition a task can be written to is also one that workers read. See [Detecting and Fixing an Existing `Write > Read` Misconfiguration](#detecting-and-fixing-an-existing-write--read-misconfiguration) if a queue is already in this state.

So the two procedures below — [increasing partitions](#how-to-increase-partitions) and [decreasing partitions](#how-to-decrease-partitions) — are the same idea: change the read count and the write count in the order that keeps `Write ≤ Read` true the whole way through. This ordering is the safeguard, and it works on every version: [Signs of Trouble](#signs-of-trouble--red-flags-after-a-change) and [Detecting and Fixing an Existing `Write > Read` Misconfiguration](#detecting-and-fixing-an-existing-write--read-misconfiguration) below cover how to spot and repair a bad state if one already exists.

---

## Scoping the Change: Namespace, Task Queue, or Task Type

The `constraints` block on `matching.numTaskqueueReadPartitions` / `matching.numTaskqueueWritePartitions` decides **what** the change applies to:

| `constraints` | Applies to |
|---|---|
| omitted / `{}` | every task queue in the cluster, across all namespaces |
| `namespace` | every task queue in that namespace |
| `namespace` + `taskQueueName` | that one task queue name — **both** its workflow and activity queues (what the examples below use) |
| `namespace` + `taskQueueName` + `taskType` | one task type of that name only |

This applies to **normal** task queues only (as covered in [the three kinds](#what-are-task-queue-partitions) above). A name's workflow and activity queues are partitioned independently, so the scopes above change **both** — to change just one, add a **`taskType`** constraint with value `Activity` or `Workflow`:

```yaml
# Activity queue only — the workflow queue of this name keeps its own (default) count
matching.numTaskqueueReadPartitions:
  - value: 8
    constraints:
      namespace: 'your-namespace'
      taskQueueName: 'your-task-queue'
      taskType: Activity
matching.numTaskqueueWritePartitions:
  - value: 8
    constraints:
      namespace: 'your-namespace'
      taskQueueName: 'your-task-queue'
      taskType: Activity
```

The step-by-step increase and decrease procedures later in this playbook use `namespace` + `taskQueueName` (no `taskType`), so they change both the workflow and activity queues of the name. The procedure is identical either way — add a `taskType` constraint to those steps to change just one type.

---

## Defaults and Sizing

- **Default: 4 partitions** — `matching.numTaskqueueReadPartitions` and `matching.numTaskqueueWritePartitions` both default to 4 for a **normal** task queue (your own workflow and activity queues).
- More partitions can raise a task queue's throughput ceiling, but each one adds load on the matching hosts. Increase only when the queue needs it — see [When to Increase Partitions](#when-to-increase-partitions).
- For low-traffic task queues, reducing partitions down to 1 can be beneficial to reduce resource consumption on matching hosts.
- Setting partitions to 1 also gives the strictest activity-dispatch rate control. A per–task-queue limit like the SDK's `taskQueueActivitiesPerSecond` is split across the read partitions and enforced independently on each: a partition dispatches at `rate ÷ numReadPartitions`, with its burst rounded **up** to `ceil(rate ÷ numReadPartitions)`. Because each partition's limiter runs on its own, the combined burst can reach `ceil(rate ÷ numReadPartitions) × numReadPartitions`, which overshoots the target whenever the rate doesn't divide evenly.
  - **Example:** `taskQueueActivitiesPerSecond = 10` over **4** partitions → `ceil(10 ÷ 4) = 3` per partition → up to `3 × 4 = 12` activities can dispatch in a burst.
  - With **1** partition → `ceil(10 ÷ 1) = 10` → burst is exactly the configured rate.
  - So one partition is the only way to enforce the limit precisely. More partitions keep the sustained rate near the target but let short bursts run above it.
- **The rate limit and the partition count set a *combined* ceiling — size them together.** A queue's real dispatch ceiling is **`min(taskQueueActivitiesPerSecond, numReadPartitions × sustainable-per-partition-rate)`**. Because `taskQueueActivitiesPerSecond` is a *per-queue* limit split evenly across partitions (`rate ÷ numReadPartitions`), the **aggregate is fixed regardless of partition count** — adding partitions just gives each a smaller slice of the same total. As a conservative planning figure, size each partition at about **~500 tasks/s** (headroom below the 1,000/s per-partition hard cap `admin.matchingNamespaceTaskqueueToPartitionDispatchRate`; real sustainable throughput is DB-bound, so this is a rule of thumb, not a guarantee).
  - **Setting the rate above `numReadPartitions × ~500` is wasted** — the partitions gate you first, so the extra headroom in the number does nothing. If the downstream genuinely needs a higher rate, raise the *partition count too*, or the smaller of the two silently wins. **Example:** `taskQueueActivitiesPerSecond = 3000` over **4** partitions can't be reached — 4 partitions deliver only ~2,000/s conservatively — so the effective cap is ~2,000, not 3,000; to actually use 3,000 you'd run ~6 partitions.

> This playbook helps you decide whether a task queue's partition count should change — see [When to Increase Partitions](#when-to-increase-partitions) and [How to Decrease Partitions](#how-to-decrease-partitions).

---

## How to Increase Partitions

Increasing takes **two steps**: raise the read count first, then the write count. Keeping read ahead of write is what holds `Write ≤ Read` the whole way through — it's the mirror image of the decrease procedure.

### Required order

The examples below raise **one** task queue from the default 4 to 8 using a per–task-queue override (matched on `namespace` + `taskQueueName`), leaving every other task queue at 4. Both entries live in the same key: the unconstrained `value: 4` is what all task queues get, and the constrained `value: 8` overrides just the one you name — the more specific entry always wins. That first `value: 4` only restates the built-in default; keep it to pin the default explicitly, or drop it to leave whatever cluster-wide default you already have untouched.

**Step 1 — Increase read partitions first**

Raise `matching.numTaskqueueReadPartitions` for that task queue to the target value. Workers begin polling the new partitions while they are still empty — no tasks are written to them yet.

```yaml
matching.numTaskqueueReadPartitions:
  - value: 4              # cluster-wide default — all other task queues
  - value: 8              # override: raise just this one queue
    constraints:
      namespace: 'your-namespace'
      taskQueueName: 'your-high-traffic-tq'
```

**Step 2 — Increase write partitions**

Once the read change has propagated to the matching hosts, raise `matching.numTaskqueueWritePartitions` for the same task queue to match. Every partition that now receives tasks is already in the read range, so workers' pollers can reach it — you are not writing to a partition no poller is able to poll.

```yaml
matching.numTaskqueueWritePartitions:
  - value: 4              # cluster-wide default — all other task queues
  - value: 8              # override: now match write to read for this queue
    constraints:
      namespace: 'your-namespace'
      taskQueueName: 'your-high-traffic-tq'
```

Make these as **two separate edits** — raise read, wait, then raise write. The file-based dynamic config client polls for changes on an interval (10s in the standard server config, 5s minimum), so pausing at least one interval between the edits lets every matching host pick up the higher read count before the higher write count. Combining them into one edit risks a host seeing the new write count before the new read count and briefly routing tasks to partitions no poller reads directly (added forwarding latency until it corrects).

Then verify the change landed cleanly — see [Monitoring After an Increase](#monitoring-after-an-increase).

---

## When to Increase Partitions

**Quick reference — does this partition need more capacity?**

```
A partition's real throughput = the SLOWEST of three ceilings:

  1. sync match             hand task straight to a poller (in-memory)   fastest, rarely the limit
  2. backlog write          persist tasks to the DB                    | the backlog path is
  3. backlog read+dispatch  read tasks back out, dispatch to pollers   | what decides under load

Before you add partitions:
  - Size so the BACKLOG stays small at PEAK load (aim: backlog age < 5s) — not by the sync-match rate.
  - More partitions aren't free: each adds matching-host load and needs pollers to cover it.
  - Then work through the RECOMMENDATIONS below:
      1. Find out WHY the write path is behind — partitions help in only one case
      2. Confirm matching is the bottleneck   — not pollers, the DB, or busy workflows
      3. Partitions WON'T drain a backlog that already exists

These signals are on the companion dashboard — per partition for backlog & write, per queue for the rule-outs.
```

### How a partition's throughput works

A partition's real limit is the **slowest** of the three paths in the card above. There's also a configured dispatch cap (`admin.matchingNamespaceTaskqueueToPartitionDispatchRate`, 1,000/s), but a partition only reaches it if the physical work keeps up — and sync match (path 1), an in-memory hand-off, is usually far faster than the two **backlog** paths, which go through the database. So the thing to size for is the backlog paths: write, then read + dispatch.

A task only falls to the backlog when sync match can't place it. Matching first holds it for up to `matching.syncMatchWaitDuration` (200ms) hoping a poller frees up; if none does, it spools the task to the database so it isn't lost while it waits. And once the backlog head is older than `matching.backlogNegligibleAge` (5s), matching stops sync-matching altogether and spools straight to the backlog, to keep dispatch roughly in order. Those backlog writes and reads are batched (up to 100 tasks per write, 1,000 per read) but still database round-trips — far slower than an in-memory hand-off.

**So tune to the backlog, not the sync-match rate.** A high sync-match rate just means workers are keeping up *right now* — it says nothing about how the partition copes under load. What you actually want is a partition sized so that, under your **expected peak workload**, the backlog stays **small and draining** rather than growing. Validate it like any capacity change: load-test with the workload you expect, watch the backlog panels, and pick the *smallest* partition count that keeps the backlog low. A good target is to hold **Approximate Backlog Age by Partition** under `matching.backlogNegligibleAge` (5s) — cross it and matching stops sync-matching entirely (as above), pushing every task onto the slower database-bound backlog path. (More partitions aren't free either, so smallest-that-keeps-up beats biggest.)

### What to watch on the dashboard

There's no single "add partitions now" metric — you'll generally just see latency rising. Instead you read each ceiling off its panels on the [companion dashboard](../observability/dashboards/server/task-queue-partitions-readme.md) (backlog and write signals per partition, rule-outs per queue), and each of the three shows up differently:

**Backlog write (path 2) — can matching persist tasks fast enough?**
**Watch: Task Write Throttle Count by Partition** (`task_write_throttle_count`). Any non-zero rate means that partition's write buffer (`matching.outstandingTaskAppendsThreshold`, 250 pending writes) is full and it's rejecting the task-writes history sends it — the write path is saturated, and the panel shows you *which* partition. Two companions on the same row: **Resource Exhausted by Cause** shows those same rejections from history's side (`AddActivityTask` / `AddWorkflowTask`, `SystemOverloaded` cause), and **Task Write Latency by Partition** rising means the database writes themselves are getting slow.

**Backlog read + dispatch (path 3) — can it drain the backlog fast enough?**
**Watch: Approximate Backlog Count** and **Approximate Backlog Age by Partition** (`approximate_backlog_count` / `approximate_backlog_age_seconds`). Bad looks like a backlog that **keeps growing and won't drain** — a short one that forms and clears is normal. There's no single metric for this ceiling, so confirm with **Async Match Latency by Partition** (`asyncmatch_latency`) climbing at the same time: together they mean tasks are arriving faster than the partition can read them back out and hand them to pollers.

**Sync match (path 1) — usually fine, rarely the real limit.**
There's no easy direct view of this one. Mostly you read it off the same **Approximate Backlog Count / Age by Partition** panels — when a partition maxes its dispatch cap (`admin.matchingNamespaceTaskqueueToPartitionDispatchRate`, 1,000/s), the overflow just spools to the backlog. For a *direct* read on the default (priority) matcher, the **Dispatch Rate-Limiting by Partition** panel shows it — but it's **opt-in**, empty unless `matching.enablePollerScalingDecisionMetrics` is on (see [References](#references)). (On the classic matcher the signal is `sync_throttle_count`, which isn't on this dashboard.)

### Recommendations: before you increase

Adding partitions helps in **one** situation: a partition's own **write** or **read + dispatch** path is maxed out while your database and matching hosts still have room to spread the work wider. Every other bottleneck is somewhere partitions can't reach — and spreading the same pollers across more partitions can even make things *worse*. So before you increase, check whether the dashboard is pointing at one of these instead:

| If the dashboard shows… | The bottleneck is likely… | Do this instead of adding partitions |
|---|---|---|
| Write throttling **with** rising **Task Write Latency** | the database write path itself | Scale the database — every partition writes to the *same* DB, so more of them won't help |
| Write throttling but **Task Write Latency** looks fine | the per-host persistence cap (`matching.persistenceMaxQPS`, 3,000/host) | If the DB has headroom, raise the cap or add matching hosts; a larger `matching.maxTaskBatchSize` (100) also drains the write buffer in fewer round-trips |
| High **Schedule to Start Latency** while the backlog is small or empty | too few worker pollers — not matching | Raise poller / slot counts. More partitions would spread the same pollers thinner, sending *more* tasks to the root before a worker picks them up |
| Per-partition **dispatch pinned at `taskQueueActivitiesPerSecond ÷ numReadPartitions`** while a **backlog persists** and **pollers sit idle** (`service_pending_requests` high) | the per-queue dispatch rate limit is capping throughput — often **intentional** (e.g. protecting a downstream) | Neither pollers nor partitions help: the cap is per-queue and split across partitions, so adding partitions only shrinks each slice. Raise `taskQueueActivitiesPerSecond` **only if the downstream can absorb more**; otherwise the backlog is the intended metering — and the idle pollers can be scaled down |
| High **Persistence Latency** (Update / CreateWorkflowExecution) | the history → database write path | Address the database; adding partitions doesn't touch this path |
| **Busy Workflow Throttling** climbing | workflow-level lock contention on history (`history.cacheNonUserContextLockTimeout`) | Not a matching bottleneck — partitions won't help |
| Matching hosts already hot (CPU / memory) | matching-fleet capacity | Scale out matching hosts first — each partition is loaded and served in memory on a host |

If none of these fit — a **write** or **backlog** panel shows a partition saturated while the DB and matching hosts have headroom — then adding partitions is the right move: more parallel write and read loops, spread across more hosts.

**And even then, partitions won't drain a backlog that already exists** — they only let *new* work skip ahead of the old pile. This one trips people up often enough to have its own section: see [Adding Partitions Won't Drain an Existing Backlog](#adding-partitions-wont-drain-an-existing-backlog).

---

## Adding Partitions Won't Drain an Existing Backlog

Adding partitions does not drain a backlog that has already built up. This is a common thing to try when a task queue has a large backlog, so it's worth knowing why it doesn't work.

A partition's backlog is never moved to another partition. A task stays on the partition it was written to until a poller reads it from there, and nothing rebalances an existing backlog across partitions. So if one partition already holds 700k tasks, adding partitions has no effect on those 700k tasks — they remain on their original partition.

New partitions only affect new tasks: they give new work somewhere else to land, so it isn't queued behind the existing backlog. There is a trade-off. A new partition starts with no tasks of its own, so its pollers are forwarded to partitions that do have tasks, and that forwarding is rate-limited (`matching.forwarderMaxOutstandingPolls`, 1 in-flight poll per partition; `matching.forwarderMaxRatePerSecond`, 10 per second). Spreading the same number of pollers across more partitions can reduce the overall dispatch rate rather than raise it.

A backlog drains as its partitions are read and their tasks dispatched to workers, so two things gate the drain rate:

- **Worker capacity.** If there aren't enough pollers or activity slots to take tasks as fast as they can be read, the backlog drains slowly regardless of anything else. Add pollers or activity slots, or use faster workers.
- **Rate limits on the drain path.** Reading the backlog out of the database is capped by `matching.persistenceMaxQPS` (3,000 qps per matching host, plus `matching.persistenceNamespaceMaxQPS` per namespace), and dispatch to pollers is capped per partition by `admin.matchingNamespaceTaskqueueToPartitionDispatchRate` (1,000/s). If a throttling signal is firing while the database and matching hosts still have headroom, raising the limit that's binding lets the backlog drain faster. If the underlying resource is already at its limit, raising a cap does not help.

So the order is: check the throttling panels, add worker capacity, and raise a binding rate limit only if the resource behind it has room. On the dashboard, Approximate Backlog Count / Age by Partition should come down as you do this; if it does not move, the partitions are at their read + dispatch ceiling (see [How a partition's throughput works](#how-a-partitions-throughput-works)), which more partitions would not change.

In short: add partitions to raise steady-state throughput before a backlog forms. To reduce a backlog that already exists, add worker capacity and clear any throttling on the drain path.

---

## Monitoring After an Increase

Open the dashboard on the queue you changed (`namespace` / `taskqueue` / `task_type`) and confirm three things:

1. **The new partitions are receiving and dispatching work.** After you raise write, **Tasks Added by Partition** should climb on the new high-index partitions (`Read-1 … newRead-1`). That's the direct confirmation they're in use — it stays non-zero even when a partition sync-matches everything and shows no backlog (so a new partition with zero backlog is fine, not a problem). Two failure signatures: if **Tasks Added** stays flat on a new partition, tasks aren't landing there and the write-count change didn't take; if **Tasks Added** climbs but that partition's **Approximate Backlog Age by Partition** also climbs and won't drain, tasks are landing but no worker is polling it — the read-count change didn't propagate, or workers didn't reconnect.

2. **The signal you acted on is receding.** Whatever drove the increase should ease on its panel: **Async Match Latency by Partition** falling, the backlog draining, or — if you enabled it — **Dispatch Rate-Limiting by Partition** dropping toward zero. (On the classic matcher, the equivalent is `sync_throttle_count` easing, which isn't on this dashboard.)

3. **The write count didn't end up higher than the read count.** Workers only poll partitions up to the read count, so if the write count is higher, tasks land on the extra partitions that nobody polls and they build a backlog that never drains. On the backlog panels this looks like a higher-numbered partition whose backlog keeps climbing and won't come down. If you see it, the read count is set too low for the write count — fix it with [Detecting and Fixing a `Write > Read` Misconfiguration](#detecting-and-fixing-an-existing-write--read-misconfiguration).

---

## How to Decrease Partitions

Decreasing takes **three steps**: lower the write count, let the partitions you're removing drain, then lower the read count. The order matters. Lowering the write count first stops new tasks from landing on the partitions you're dropping, so they can empty out. You keep the read count high while that happens, so workers keep polling those partitions and finish the tasks still on them — then, once they're empty, you lower the read count. Lower it too early and those partitions have no poller, so any tasks left on them are stuck until you raise the read count again.

### Required order

The examples below take the same task queue back down from 8 to 4 using its per–task-queue override. (If your target is the default of 4, you can instead remove the override entirely — but do it in the same write-first order below.)

**Step 1 — Decrease write partitions first**

Lower `matching.numTaskqueueWritePartitions` for that task queue to the target value. No new tasks will be written to the partitions being retired.

```yaml
matching.numTaskqueueWritePartitions:
  - value: 4              # cluster-wide default — all other task queues
  - value: 4              # this queue, lowered from 8 back to 4
    constraints:
      namespace: 'your-namespace'
      taskQueueName: 'your-high-traffic-tq'
```

**Step 2 — Wait for drain**

Keep the read count at the higher value and watch the partitions you're removing (the higher-numbered ones, at index ≥ the new write count) empty out. Three panels tell you when they're done:

- **Tasks Added by Partition** — the partitions you're removing should drop to zero, confirming Step 1 took effect and no new tasks are landing on them.
- **Approximate Backlog Count by Partition** and **Approximate Backlog Age by Partition** — their lines should fall to zero as workers finish the tasks already queued there.

**Move on to Step 3 only once both the count and the age read 0 on every partition you're removing.** Read the age from **Approximate Backlog Age by Partition** (the logical `approximate_backlog_age_seconds`) — the physical age gauge can show 0 while tasks are still queued (see [Backlog age can read 0 even when it isn't](#backlog-age-can-read-0-even-when-it-isnt)).

**Step 3 — Decrease read partitions**

Only after you are confident the extra partitions are drained, lower `matching.numTaskqueueReadPartitions` for the same task queue to match the write partition count.

```yaml
matching.numTaskqueueReadPartitions:
  - value: 4              # cluster-wide default — all other task queues
  - value: 4              # this queue, now lowered from 8 to match write
    constraints:
      namespace: 'your-namespace'
      taskQueueName: 'your-high-traffic-tq'
```

> **Why the drain step is not optional.** Lower the read count while a partition you're removing still holds backlog, and that backlog is **stranded**. Workers only poll partitions up to the read count, so the removed partitions get no more polls; with no polls and no new writes, they unload from the matching host after `matching.maxTaskQueueIdleTime` (5 min). An unloaded partition doesn't dispatch its tasks, so they sit undelivered in the database — not lost, but stuck (and able to hit their ScheduleToStart timeouts) — until you raise the read count again to reload them. It's the same failure as a [`Write > Read` misconfiguration](#detecting-and-fixing-an-existing-write--read-misconfiguration). Drain first, then lower read.

---

## When to Decrease Partitions

You lower the count less often than you raise it. Two situations call for it.

### You have more partitions than the workload needs

Partitions aren't free — each is held in memory on a matching host and needs pollers covering it. Past the point where your load fills them, the extra ones only add matching-host load and split a fixed set of pollers across more queues, which can *raise* dispatch latency instead of lowering it. There's no single metric that says "too many partitions," so judge it from two things on the dashboard:

- **Partitions are barely used, even at your busiest.** This is the clearest sign. On **Tasks Added by Partition**, if most partitions carry only a trickle of tasks at peak — with their backlog sitting near zero on **Approximate Backlog Count by Partition** — the count is larger than the workload needs, and you're paying matching-host cost for capacity you aren't using.
- **Schedule-to-start latency rises while backlogs stay small.** Tasks are being handed off but wait before a worker picks them up (**Schedule to Start Latency**). Assuming you have enough workers overall, that points to pollers spread across more partitions than they can cover — tasks keep landing on partitions where no poller is waiting. Fewer partitions concentrate the same pollers, so each is likelier to have one. (If you're actually short on workers, that's a worker problem — add workers, don't drop partitions.)

Only decrease for this reason when **no partition is at its throughput ceiling** — if a partition is backlog-bound, you need *more* capacity, not fewer (see [When to Increase Partitions](#when-to-increase-partitions)).

### You want a per-task-queue rate limit to be exact

A single partition is the only way to enforce a per-task-queue dispatch limit *precisely*. An SDK rate like `taskQueueActivitiesPerSecond` is split across the read partitions, so with several partitions short-term dispatch can burst above your target; on one partition it's exact (the math is in [Defaults and Sizing](#defaults-and-sizing)). If precise rate control matters more than parallelism for a queue, run it on a single partition.

**Before you decrease:**

- **Size for your peak, not a quiet moment** — pick the count for the busiest load you expect, so you're not resizing right back up.
- **Never strand a backlog** — only decrease through the drain-first procedure: lower write, wait for the partitions you're removing to drain to zero, then lower read. Dropping read while one of them still holds backlog strands those tasks. See [How to Decrease Partitions](#how-to-decrease-partitions).

---

## Monitoring After a Decrease

The check that tells you *when* to lower read is part of the procedure itself — the drain wait in [How to Decrease Partitions](#how-to-decrease-partitions) (Step 2). Once you've taken that final step and lowered read, confirm the smaller set of partitions is healthy:

- **The partitions you kept are handling the load.** On **Approximate Backlog Count / Age by Partition** the remaining partitions should hold steady with no growing backlog, and **Schedule to Start Latency** should stay flat. (On **Tasks Added by Partition** they'll each carry more now — the same work spread over fewer partitions — which is expected.) If a backlog starts building or latency climbs, you cut too far; raise the count back up.
- **Nothing was left behind.** On **Approximate Backlog Count / Age by Partition**, none of the partitions you removed should still show a backlog. If you followed the drain wait they won't — but if one does, it has no poller now, so raise the read count to reload and drain it.

---

## Signs of Trouble — Red Flags After a Change

Each sign below is something you can see on a dashboard panel — named in the first column, so you know where to look.

| Sign on the dashboard | What it means | What to do |
|---|---|---|
| A higher-numbered partition keeps building a backlog that won't drain, and/or latency climbs — **Approximate Backlog Count / Age by Partition**, **Schedule to Start Latency** | The write count ended up higher than the read count, so that partition gets tasks but no worker polls it — its tasks have to be forwarded to a partition that *is* polled, which adds delay | Raise `matching.numTaskqueueReadPartitions` so it's ≥ the write count — see [the fix](#the-fix) |
| After an **increase**, the new partitions stay empty or latency got worse instead of better — **Tasks Added by Partition** flat on the new partitions, **Schedule to Start Latency** / **Async Match Latency by Partition** up | The change didn't reach the workers: the read change didn't propagate to all matching hosts, or workers didn't reconnect to poll the new partitions | Confirm the read change propagated and workers reconnected, and that write ≤ read |
| After an **increase**, the throttle you were acting on didn't budge — **Task Write Throttle Count** / **Dispatch Rate-Limiting by Partition** flat (on the classic matcher, `sync_throttle_count`) | Matching partitions weren't the bottleneck | It's a worker problem, not a partition one — check **Schedule to Start Latency** and add pollers or activity slots |
| After a **decrease**, a partition you removed still shows a backlog — **Approximate Backlog Count / Age by Partition** | You lowered the read count before it finished draining, so its leftover tasks now have no worker polling them | Raise the read count to reload and drain it; next time, wait for the drain before lowering read |

---

## Reading the Metrics

Two reasons the numbers might not show what you expect. (Turning the per-partition metrics *on* is covered up front, under [References](#references) → **Turning on the per-partition metrics**.)

### Backlog age can read 0 even when it isn't

On the priority matcher, the **physical** backlog-age metric (`physical_approximate_backlog_age_seconds`) has a bug: it can read **0 even with thousands of tasks queued**. It's a known issue with a server-side fix underway. Until that lands, read backlog age from the dashboard's **Approximate Backlog Age by Partition** panel — it uses the reliable **logical** metric (`approximate_backlog_age_seconds`), the true age of the oldest waiting task. (This is also why the dashboard has no physical-*age* panel for now; the physical backlog **count** is unaffected.)

### Activity queues can look empty: eager dispatch

This affects **activity** task queues only. When `system.enableActivityEagerExecution` is on (server default `true`) and the SDK worker requests eager dispatch (the default in Go and Java), an activity scheduled during a workflow task is handed straight back to the completing worker **inline in the workflow-task response** — it never reaches the matching task queue. So an activity queue can show **flat, empty per-partition metrics even under heavy load**: no `approximate_backlog_*`, no `asyncmatch_latency`, no `task_schedule_to_start_latency` for `task_type="Activity"`.

**Because eager activities bypass matching entirely, changing partition counts does nothing for them.** Partitions only matter for activities that fall back to the task queue — which happens when the worker has no free activity slot, or when eager is disabled.

**Is it happening?** Watch the dashboard's **Eager Activity Dispatch** panel — it graphs `activity_eager_execution` (one count per eagerly-dispatched activity, per namespace and task queue; the `taskqueue` tag needs `metrics.breakdownByTaskQueue` on, same as the per-partition metrics). A high rate there while the activity backlog and Tasks Added panels sit flat is the confirmation.

To observe real matching-side load on an activity queue (for sizing, or to reproduce a backlog), the activities have to actually reach matching. Force that by either saturating the worker's activity slots — eager falls back to the task queue when no slot is free — or turning eager off. Turn it off **on the SDK/worker side** where you can: that scopes it to just the worker you're testing. The server setting `system.enableActivityEagerExecution` also works, but it's **namespace-wide** — setting it to `false` disables eager for *every* task queue in the namespace, not only the one you're investigating. (Workflow tasks never dispatch eagerly, so workflow queues are unaffected.)

---

## Detecting and Fixing an Existing `Write > Read` Misconfiguration

A task queue can end up with the **write count higher than the read count** — from a bad change order, a one-sided edit, or an override applied to only one of the two. When that happens, tasks get written to higher-numbered partitions that no worker polls directly.

**Usually these tasks aren't stuck, just slow.** While such a partition stays loaded, it forwards its backlog to a partition that *is* polled, so the tasks still get dispatched — they just take an extra hop, which adds latency and concentrates load on that one partition. Nothing is lost, but it defeats the point of the extra write partitions.

**They can get stuck in one case:** if an unpolled partition goes idle and unloads (`matching.maxTaskQueueIdleTime`, 5 min) while still holding a backlog, that backlog waits in the database until the partition loads again — and with no polls and no new writes, it may not reload promptly. You can catch it coming, though: a partition heading there shows a **backlog age that climbs and won't drain** on **Approximate Backlog Age by Partition**. A rising per-partition backlog age (`approximate_backlog_age_seconds`) is a good thing to alert on — it flags a `Write > Read` stall and any other non-draining backlog alike, and there's a ready-to-adapt rule for it in [Alert 83](../observability/alerts/server/alerts-index.md#alert-83--task-queue-partition-backlog-not-draining). Rising **Schedule to Start Latency** is the broader "tasks aren't getting picked up" catch-all. The fix is simple, so it's worth correcting rather than living with.

### How to confirm

1. Compare the two counts for the queue — `matching.numTaskqueueReadPartitions` vs `matching.numTaskqueueWritePartitions` (check the global default *and* any per-namespace / per-task-queue override). Write higher than read is the misconfiguration.
2. On the dashboard, open **Approximate Backlog Count / Age by Partition** and look at the higher-numbered partitions (index ≥ the read count). A backlog sitting on one of those confirms it. (Per-partition metrics must be on — see [References](#references).)

### The fix

Raise `matching.numTaskqueueReadPartitions` so it's **≥** the write count. Those partitions now fall in the range workers poll, so pollers read them directly — the forwarding latency goes away, and any partition that had unloaded with a leftover backlog reloads and drains. From there, decide your real target; if you want fewer partitions, use the [decrease procedure](#how-to-decrease-partitions).

If any tasks expired (hit their schedule-to-start timeout) while delayed, re-drive the affected workflows with `tdbg workflow refresh-tasks` (alias `rt`) — it regenerates a workflow's tasks from its current state and re-submits them. Use `--workflow_id` / `--run_id` for one execution, or the batch form for all executions matching a visibility query.
