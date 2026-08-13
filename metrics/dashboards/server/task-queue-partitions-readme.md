# Temporal Task Queue Partitions Dashboard

This dashboard is an operator tool for **changing a task queue's partition count safely** and for judging whether a partition change is justified. It is the metrics companion to the [Changing Task Queue Partitions](../../../playbooks/change-task-queue-partitions.md) playbook — each row group maps to a step or decision in that procedure.

It shows metrics for the **single task queue you are focusing on** (`$namespace` + `$taskqueue` + `$task_type`). For a cluster-wide view, use the [Temporal Server — Overview](./temporal-server-readme.md) dashboard instead.

> **Compatibility:** Temporal Server v1.20+ · Grafana 9.0+ · Prometheus. The `physical_approximate_backlog_*` panels require **v1.31.0+** (empty on older versions — use the approximate panels instead).

> **Current version:** v1.1.0 — see [CHANGELOG](./task-queue-partitions-changelog.md)

The matching playbook is at [`playbooks/change-task-queue-partitions.md`](../../../playbooks/change-task-queue-partitions.md).

---

## Table of Contents

- [Prerequisites — enable the breakdown flags](#prerequisites--enable-the-breakdown-flags)
- [Template Variables](#template-variables)
- [Annotations](#annotations)
- [Row 1 — Per-Partition Backlog (Drain & Write>Read Detection)](#row-1--per-partition-backlog-drain--writeread-detection)
- [Row 2 — Sync Match & Dispatch (Path 1 Ceiling)](#row-2--sync-match--dispatch-path-1-ceiling)
- [Row 3 — Backlog Write (Path 2 Ceiling)](#row-3--backlog-write-path-2-ceiling)
- [Row 4 — Rule-Outs (Is Matching Actually the Bottleneck?)](#row-4--rule-outs-is-matching-actually-the-bottleneck)
- [What each panel is scoped to](#what-each-panel-is-scoped-to)

---

## Prerequisites — enable the breakdown flags

The per-partition panels (Rows 1–3) depend on **both** `metrics.breakdownByPartition` and `metrics.breakdownByTaskQueue` being **on** for the task queue you are viewing (both default `true` since server v1.25.0). If either is off, the per-partition/per-task-queue gauges are not emitted and the panels are empty.

Set them for the whole namespace or a specific task queue (sample below):

```yaml
metrics.breakdownByPartition:
  - value: true
    constraints:
      namespace: 'your-namespace'
      taskQueueName: 'your-task-queue'
metrics.breakdownByTaskQueue:
  - value: true
    constraints:
      namespace: 'your-namespace'
      taskQueueName: 'your-task-queue'
```

To scope to a single **task type** — the activity or workflow queue of that name — add a `taskType` constraint (`Activity` or `Workflow`):

```yaml
metrics.breakdownByPartition:
  - value: true
    constraints:
      namespace: 'your-namespace'
      taskQueueName: 'your-task-queue'
      taskType: Activity
metrics.breakdownByTaskQueue:
  - value: true
    constraints:
      namespace: 'your-namespace'
      taskQueueName: 'your-task-queue'
      taskType: Activity
```

Disabling **either** flag turns off the per-task-queue backlog gauges entirely. If a panel is unexpectedly empty, check these first.

---

## Template Variables

| Variable | Source | Notes |
|---|---|---|
| `$namespace` | `label_values(service_requests{namespace!="_unknown_"}, namespace)` | The namespace owning the task queue |
| `$taskqueue` | `label_values(approximate_backlog_count{namespace="$namespace", service_name=~"matching"}, taskqueue)` | The task queue being resized. Populated only when the breakdown flags are on and the queue is loaded (its backlog gauge emits, even at 0) |
| `$task_type` | Custom — `Activity` (default), `Workflow` | Which task type's metrics to view. A task queue name has a separate activity queue and workflow queue, each with its own partitions and backlog — this selects which one the panels show. (It does **not** scope a config change: the partition-count config, unless you add a `taskType` constraint, applies to both.) |
| `$p` | Custom (0.50–0.999) | Percentile for the latency (histogram) panels. Default 0.99 |

---

## Annotations

- **Matching Restarts** — a Grafana annotation that draws a **vertical red line** on the panels each time a matching pod restarts (`increase(restarts{service_name=~"matching"}) > 0`). Toggle it on/off with the annotation control at the top of the dashboard. A matching restart can itself cause brief latency spikes or a backlog reading dropping to 0, so when a red line sits next to a spike, **discount that moment** when deciding whether to increase or decrease partitions — it may be the restart, not real load. It has no effect on the data; optional, remove it if you find it noisy.

---

## Row 1 — Per-Partition Backlog (Drain & Write>Read Detection)

Per-partition backlog depth and age — one series per partition index.

| Panel | Metric | What it measures |
|---|---|---|
| **Approximate Backlog Count by Partition** | `approximate_backlog_count` | Tasks waiting in each partition's backlog. Summed across `worker_version` on v1.31.0+ |
| **Approximate Backlog Age by Partition** | `approximate_backlog_age_seconds` | Age of the oldest task in each partition's backlog — the reliable backlog-age signal |
| **Physical Backlog Count by Partition (v1.31.0+)** | `physical_approximate_backlog_count` | Version-agnostic backlog count. Empty pre-v1.31.0 |

> **How to read this row** — drain confirmation on a decrease, load spread on an increase, and `Write > Read` detection: see the playbook's [Monitoring After a Decrease](../../../playbooks/change-task-queue-partitions.md#monitoring-after-a-decrease) and [Monitoring After an Increase](../../../playbooks/change-task-queue-partitions.md#monitoring-after-an-increase), plus [Detecting and Fixing an Existing `Write > Read` Misconfiguration](../../../playbooks/change-task-queue-partitions.md#detecting-and-fixing-an-existing-write--read-misconfiguration).

> **Approximate vs. Physical backlog count.** Both count panels show the same backlog — use either. They diverge only under Worker Versioning: Approximate is split per worker version, Physical is the single combined total.

---

## Row 2 — Sync Match & Dispatch (Path 1 Ceiling)

| Panel | Metric | What it measures |
|---|---|---|
| **Async Match Latency by Partition** | `asyncmatch_latency` (`$p`) | Time from task creation to worker pickup on the async/backlog path, per partition |
| **Dispatch Rate-Limiting by Partition (opt-in)** | `poller_scale_decision{reason="rate_limited"}` | Rate at which the per-partition dispatch cap (`admin.matchingNamespaceTaskqueueToPartitionDispatchRate`, 1,000/s) is throttling dispatch, per partition — the priority matcher's path-1 (sync match / dispatch) saturation signal |

> **The Dispatch Rate-Limiting panel is opt-in and empty by default.** It reads `poller_scale_decision`, which matching emits **only when `matching.enablePollerScalingDecisionMetrics` is enabled** (default off). When the per-partition dispatch cap throttles dispatch, matching records a poller-autoscaling decision tagged `reason="rate_limited"`; a non-zero rate means the partition is hitting that cap. This is the priority matcher's stand-in for the classic-only `sync_throttle_count` (a dedicated Sync Throttle panel was removed in v1.0.2 because it's classic-only) — on the **classic** matcher, watch `sync_throttle_count` instead. Because the dispatch limiter caps both sync match and backlog dispatch, treat this as a dispatch-cap signal rather than sync-match-specifically.

> **How to read this row** — the sync-match (path 1) throughput ceiling and how to tell it apart from the other ceilings: see the playbook's [When to Increase Partitions](../../../playbooks/change-task-queue-partitions.md#when-to-increase-partitions).

---

## Row 3 — Backlog Write (Path 2 Ceiling)

| Panel | Metric | What it measures |
|---|---|---|
| **Task Write Throttle Count by Partition** | `task_write_throttle_count` | Rate of a partition falling behind writing tasks to persistence (its write buffer filling) |
| **Task Write Latency by Partition** | `task_write_latency` (`$p`) | Time to persist a task to the database, per partition |
| **Resource Exhausted by Cause (namespace)** | `service_errors_resource_exhausted` | ResourceExhausted errors by operation and cause. `SystemOverloaded` = write buffer full; `RpsLimit` = instance/namespace RPS limits. **Namespace-scoped** (no `taskqueue` tag) |

> **How to read this row** — the backlog-write (path 2) throughput ceiling: see the playbook's [When to Increase Partitions](../../../playbooks/change-task-queue-partitions.md#when-to-increase-partitions).

---

## Row 4 — Rule-Outs (Is Matching Actually the Bottleneck?)

| Panel | Metric | What it measures |
|---|---|---|
| **Schedule to Start Latency (queue)** | `task_schedule_to_start_latency` (`$p`) | Latency from task scheduling to worker pickup for this queue. **Queue-level** — not per-partition (history emits it with `partition=__normal__`) |
| **Busy Workflow Throttling** | `task_errors_throttled{resource_exhausted_cause="BusyWorkflow"}` | Rate of history tasks throttled by workflow-lock contention. **Namespace-scoped** |
| **Persistence Latency — Update / CreateWorkflowExecution** | `persistence_latency` (`$p`) | History write-path latency for `UpdateWorkflowExecution` / `CreateWorkflowExecution`. **Namespace-scoped** |

> **How to read this row** — using these to rule out a non-matching bottleneck before adding partitions: see the playbook's [Recommendations: before you increase](../../../playbooks/change-task-queue-partitions.md#recommendations-before-you-increase).

---

## What each panel is scoped to

Not every panel can be split by partition — some metrics don't carry a `partition` (or even a `taskqueue`) tag, so they show a coarser aggregate. This is what you'll actually see per panel:

**Per partition** — one line per partition index:
- Approximate Backlog Count / Age
- Physical Backlog Count
- Async Match Latency
- Dispatch Rate-Limiting (opt-in — needs `matching.enablePollerScalingDecisionMetrics`)
- Task Write Throttle Count / Latency

**Whole task queue** — one line for the queue, not split by partition:
- Schedule to Start Latency

**Whole namespace** — covers the entire namespace, **not** just the task queue you picked in `$taskqueue`:
- Resource Exhausted by Cause
- Busy Workflow Throttling
- Persistence Latency
