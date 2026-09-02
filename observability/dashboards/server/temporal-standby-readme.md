# Temporal Standby Cluster — Replication Health Dashboard

A dedicated Grafana dashboard for monitoring Temporal standby clusters in a multi-cluster replication setup, focused on replication fidelity and failover readiness.

> **Compatibility:** Temporal Server v1.20+ · Grafana 9.0+ · Prometheus

> **Current version:** v2.2.0 — see [CHANGELOG](./temporal-standby-changelog.md)

---

## Table of Contents

- [Overview](#overview)
- [When to Use This Dashboard](#when-to-use-this-dashboard)
- [Setup](#setup)
- [Template Variables](#template-variables)
- [Annotations](#annotations)
- [Groups and Panels](#groups-and-panels)
    - [Stream Health](#1-stream-health)
    - [Replication Lag](#2-replication-lag)
    - [Task Pipeline Health](#3-task-pipeline-health)
    - [Replication DLQ](#4-replication-dlq-cassandra-only)
    - [Namespace Replication](#5-namespace-replication)
    - [Standby Task Processing Behavior](#6-standby-task-processing-behavior)
    - [Standby Cluster Infrastructure](#7-standby-cluster-infrastructure)
    - [History Scavenger](#8-history-scavenger)
- [Key Dynamic Config Reference](#key-dynamic-config-reference)
- [Relationship to the Server Dashboard](#relationship-to-the-server-dashboard)
- [Related Resources](#related-resources)

---

## Overview

This dashboard answers a different question than the main [Temporal Server Dashboard](./temporal-server-readme.md). Where the server dashboard asks *"is this cluster working?"*, this dashboard asks *"is replication fidelity maintained and is this cluster ready to take over?"*

It is designed to be deployed once per standby cluster and used by operators to:

- Confirm the replication stream between active and standby is healthy
- Quantify how far behind the standby is from the active cluster
- Detect replication task failures, DLQ accumulation, and stream stalls before they affect failover readiness
- Distinguish between expected standby behavior (e.g. not-active task errors, scheduled queue lag) and actual problems

---

## When to Use This Dashboard

Use this dashboard if you are running Temporal in any of the following configurations:

- **Active/Passive multi-cluster** — one cluster actively serving traffic, one or more standbys receiving replicated state
- **Active/Active multi-cluster** — each cluster is active for a subset of namespaces; each cluster is also a standby for the other

This dashboard is **not** relevant for single-cluster Temporal deployments.

---

## Setup

1. Configure Prometheus metrics on each standby cluster (same as the active cluster):

```yaml
metrics:
  tags:
    environment: production
    cluster: my-standby-cluster      # important: tag your clusters distinctly
  prometheus:
    timerType: "histogram"
    listenAddress: "0.0.0.0:8000"
```

2. Ensure your Prometheus scrape config targets the standby cluster's metrics endpoint. If you are using a single Prometheus instance to scrape multiple clusters, add a `cluster` label to each scrape job so the template variables in this dashboard can filter correctly.

3. Import `temporal-standby-replication-dashboard.json` into Grafana via **Dashboards → Import**.

4. Select your Prometheus datasource when prompted.

5. For multiple standby clusters, import the dashboard once and use the **Standby Cluster** template variable to switch between them. Each standby cluster will appear as a separate option in the dropdown as long as its metrics are scraped into the same Prometheus instance.

---

## Template Variables

| Variable | Description | Notes |
|---|---|---|
| **Datasource** | Prometheus datasource to use | — |
| **Standby Cluster** | The standby cluster to monitor | Populated from `cluster` label on `replication_tasks_recv` |
| **Source (Active) Cluster** | The active cluster this standby is receiving from | Populated from `source_cluster` label; supports All for multi-source topologies |
| **Namespace** | Filters namespace-scoped panels | Supports All |
| **Percentile** | Histogram quantile for latency panels | Default `0.99` |

---

## Annotations

| Annotation | Signal | Color |
|---|---|---|
| **Service Restarts** | Any service restart on the standby cluster | Red |
| **Stream Stuck** | `replication_stream_stuck > 0` — stream has stopped making progress | Dark red band across all panels |

The stream stuck annotation renders as a red band across the entire dashboard time range for the duration of the stall. This makes it easy to correlate lag spikes, error increases, and other signals with stream health events.

---

## Groups and Panels

---

### 1. Stream Health

The first section you should look at. These are binary go/no-go signals for the standby replication stream, displayed as stat panels for immediate red/green visibility without needing to read a chart.

> **Primary alert:** `replication_stream_stuck` is the most important metric in this dashboard. Any non-zero value means the replication stream has stopped making progress entirely. It should be a paging alert.

| Panel | Description |
|---|---|
| **Stream Stuck** | Stat panel. Non-zero means the stream has stopped making progress and is no longer delivering tasks to the standby. This should be a paging alert. Turns red at any value above zero. |
| **Stream Panics** | Stat panel. Rate of unrecoverable stream-level panics across all History instances. Turns red at any value above zero. |
| **Stream Errors** | Stat panel. Rate of stream-level errors. Turns orange at 1, red at 10. Transient errors are expected during restarts; a sustained rate indicates a connectivity or configuration problem. |
| **Stream Channel Full** | Stat panel. Buffer pressure on the replication receiver — the channel that holds in-flight tasks is full. Indicates the standby is receiving tasks faster than it can process them. |
| **Stream Health Over Time** | Timeseries view of all four stream signals together. Useful for correlating stream health events with the lag and pipeline panels in sections 2 and 3. |

---

### 2. Replication Lag

Quantifies how far behind the standby cluster is relative to the active. The headline metric is `replication_tasks_lag` — the task ID delta between the two clusters.

> **Baseline note:** The default `StandbyClusterDelay` dynamic config is 5 minutes (`history.standbyClusterDelay`). This means some scheduled queue lag is expected and normal. The panels in this section are most useful for watching trends and growth rather than absolute values.

> **Active-side cause:** `replication_sender_rate_limit_latency` originates on the **active** cluster. If standby lag is growing but no standby-side metric explains it, elevated sender rate limit latency on the active cluster is the likely cause. Controlled by the `history.ReplicationEnableRateLimit` dynamic config on the active cluster.

| Panel | Description |
|---|---|
| **Replication Tasks Lag** | Gauge. Task ID delta between standby and active. The primary headline number for how far behind the standby is. Thresholds: 1,000 orange, 5,000 red. |
| **Recv Backlog Depth** | Gauge. Average depth of the receiver-side replication backlog on the standby. A growing backlog means the standby is receiving tasks faster than it can apply them. |
| **Send Backlog Depth** | Gauge. Average depth of the sender-side backlog on the active cluster. Elevated values here explain standby lag before any standby-side metric fires. |
| **Replication Latency — Time Behind (End-to-End)** | Timeseries. Dedicated view of `replication_latency` (p50 and p99) — wall-clock from when a task is created on the active to when the standby applies it. The p99 is the single best "how many seconds is the standby behind" signal. Red at 20s aligns with alert `FAILOVER-PRE-07` (handover blocker). Use this to decide whether reconnect churn or load bursts are actually keeping the standby behind, vs. just producing log noise. |
| **Replication Latencies** | Timeseries. End-to-end, queue, processing, transmission, and persistence load latencies at the selected percentile — the same `replication_latency` end-to-end series shown above, plus its sub-components for breaking down *where* latency comes from. High transmission latency points to network issues. High processing latency points to standby persistence pressure. |
| **Sender Rate Limit Latency (Active → Standby)** | Timeseries. Time the active cluster spent rate-limiting replication sends. An active-side signal that appears here as context for diagnosing standby lag with no other visible cause. |
| **Replication Task Generation and Load Latency** | Timeseries. Generation latency = time from workflow event to replication task creation on the active cluster. Load latency = persistence schedule-to-start for replication tasks. High values here mean the active cluster is slow to produce tasks, which contributes to end-to-end lag. |

---

### 3. Task Pipeline Health

Tracks the full replication task lifecycle on the standby receiver. The most important visual is the gap between the `received` and `applied` lines on the throughput panel — this is your real-time backlog indicator.

> **Standby-specific metric:** `task_errors_standby_retry_counter` only fires on standby clusters. It increments when a standby task cannot find its history events and must retry. This is tied to the `history.standbyTaskMissingEventsResendDelay` dynamic config (default: 10 minutes). A consistently high rate means the standby is regularly failing to retrieve events from the active cluster.

| Panel | Description |
|---|---|
| **Replication Task Throughput (Recv vs Applied vs Failed)** | Timeseries showing received, applied, failed, and skipped task rates together. The gap between received and applied is your live backlog. Any sustained failed/s rate requires immediate investigation. |
| **Standby Retry Counter** | Timeseries. Increments when standby tasks cannot find history events and must retry. Tied to `StandbyTaskMissingEventsResendDelay` (10min default). A high sustained rate indicates the standby is consistently failing to retrieve events from the active cluster. |
| **Replication Task Errors by Type** | Timeseries. Errors broken down by error type. Useful for diagnosing specific failure modes such as namespace not found, shard ownership lost, or network errors. |
| **Replication Task Attempts Histogram** | Timeseries. Average attempts per replication task. High values (above 2–3) mean tasks are struggling to apply and being retried repeatedly. Cross-reference with the standby retry counter and stream health. |
| **Backfill and Duplicate Events** | Timeseries. Backfill indicates recovery is in progress — expected after a restart or gap. Elevated duplicates indicate stream instability, such as a sender repeatedly resending the same tasks. |
| **Outlier Namespaces** | Timeseries. Namespaces that are disproportionately contributing to replication problems. Useful for isolating whether a broad replication issue is caused by a specific namespace. |

---

### 4. Replication DLQ ⚠️ Cassandra Only

> **Cassandra persistence only.** The history task DLQ is only implemented for Cassandra backends. On PostgreSQL or MySQL these panels will not emit data and should be ignored. The `history.TaskDLQEnabled` dynamic config must also be `true` (which is the default). Use `tdbg dlq` CLI to inspect and manage DLQ contents.

The DLQ is the last line of defense before replication tasks are permanently lost. Tasks land here after exhausting all retry attempts (default: 80 attempts, controlled by `history.ReplicationTaskProcessorErrorRetryMaxAttempts`).

| Panel | Description |
|---|---|
| **DLQ Non-Empty ⚠️ Cassandra Only** | Stat panel. Non-zero means tasks have failed past all retry attempts and landed in DLQ. Should be a paging alert. |
| **DLQ Enqueue Failures ⚠️ Cassandra Only** | Stat panel. Tasks that failed AND could not be written to DLQ. More severe than DLQ non-empty — it means failed tasks cannot even be preserved for later inspection. |
| **DLQ Max Level ⚠️ Cassandra Only** | Gauge. Highest task ID written to DLQ. Used together with ack level to calculate unprocessed DLQ depth. |
| **DLQ Ack Level ⚠️ Cassandra Only** | Gauge. Last task ID acknowledged from DLQ. The gap between max level and ack level is the number of unprocessed DLQ tasks. |
| **DLQ Depth Over Time ⚠️ Cassandra Only** | Timeseries showing max level, ack level, and the derived gap (max − ack) together. A growing gap means DLQ tasks are accumulating faster than they are being processed. A flat or shrinking gap means DLQ processing is keeping up. |

---

### 5. Namespace Replication

Namespace replication is a separate subsystem from history task replication. Namespace metadata (including namespace registration, config updates, and retention changes) is replicated independently. You can have healthy history replication but broken namespace replication, which causes subtle but impactful issues on failover — missing namespaces, stale configs, or incorrect active/passive assignments.

> Unlike the history task DLQ, the namespace replication DLQ is not Cassandra-specific and applies to all persistence backends.

| Panel | Description |
|---|---|
| **Namespace Replication Task Ack Level** | Timeseries. How far along namespace replication is. A flat line that is not advancing indicates namespace replication has stalled — this is distinct from history replication stalling. |
| **Namespace Replication DLQ Levels** | Timeseries. Gap between DLQ max level and ack level for namespace replication tasks. A growing gap means namespace replication tasks are failing past their retry limit. |
| **Namespace Replication DLQ Enqueue Requests** | Timeseries. Rate of namespace replication tasks being sent to DLQ. Any non-zero sustained rate means namespace metadata is failing to replicate. |

---

### 6. Standby Task Processing Behavior

These metrics will be non-zero on a healthy standby cluster by design. The panels in this section are most useful for detecting abnormal growth trends or unexpected spikes, not for alerting on absolute values.

> **Scheduled queue lag baseline:** The `history.standbyClusterDelay` dynamic config (default: 5 minutes / 300 seconds) introduces an intentional artificial delay on the standby's view of active cluster time. This means a scheduled queue lag below 300s is expected and normal. Alert on growth **beyond** this baseline, not on the value itself.

| Panel | Description |
|---|---|
| **Not-Active Task Errors (Expected on Standby)** | Timeseries. Expected to be non-zero on a standby — tasks are intentionally not processed as active. A sudden spike or a value that grows over time may indicate namespace misconfiguration or an incorrect active/passive cluster assignment for a namespace. |
| **Standby Scheduled Queue Lag** | Timeseries. Scheduled queue lag on the standby History service. The ~300s baseline is expected due to `StandbyClusterDelay`. Alert on growth beyond baseline, not on absolute value. Thresholds: 300s (5min) orange, 600s (10min) red. |
| **Task Terminal Failures and DLQ Failures** | Timeseries. Terminal failures = tasks that gave up after all retries. DLQ write failures = tasks that failed AND could not be written to DLQ. Both are serious signals requiring investigation. |
| **Namespace Handover Errors** | Timeseries. Errors caused by namespace handover state. A non-zero rate is expected and normal during a planned failover event. Unexpected values outside of a planned failover indicate a namespace stuck in handover state. |

---

### 7. Standby Cluster Infrastructure

The standby cluster maintains its own persistence layer and shard assignments even though it is not serving live user traffic. Persistence degradation on the standby directly impacts how quickly it can apply replication tasks, which increases replication lag.

| Panel | Description |
|---|---|
| **Persistence Latency** | Timeseries. Persistence latencies on the standby broken down by operation. High values here directly cause replication task processing latency — replicated workflow state must still be written to the standby's DB. Thresholds: 300ms orange, 1s red. |
| **Persistence Errors** | Timeseries. Persistence errors on the standby broken down by operation and error type. Any sustained error rate requires investigation — persistence errors will cause replication tasks to fail and retry. |
| **Shard Acquisition Latency** | Timeseries. Latency for acquiring shards on the standby History service. High values during failover preparation are a warning sign — a standby that is slow to acquire shards will take longer to become fully active after a failover. Thresholds: 200ms orange, 500ms red. |
| **Shard Movement** | Timeseries. Shard creation and removal rate. Heavy churn is expected during a failover event. Unexpected churn during normal standby operation may indicate history host instability. |
| **Service Restarts** | Timeseries. Rate of service restarts by service on the standby cluster. Frequent History service restarts will cause shard movement and replication stream reconnections. |
| **Persistence Availability** | Gauge. Percentage of persistence requests that succeeded. Thresholds: 99% green, 95% orange. A degraded standby persistence layer reduces replication throughput and failover readiness. |

---

### 8. History Scavenger

The history scavenger clears **leftover history** on this standby — `history_tree` / `history_node` rows whose workflow execution has already been deleted. It runs about every 12 hours (independently on every cluster) and skips any branch younger than `worker.historyScannerDataMinAge` (default **60 days**). For short-lived, high-volume workflows almost every branch is younger than 60 days, so the scavenger skips nearly all of them and this standby's history tables grow. Full mechanism, diagnosis, and remediation are in the **[XDC Standby Database Growth playbook](../../../playbooks/xdc-standby-database-growth.md)**.

These counters are emitted **only** by the history scavenger, always tagged `operation="HistoryScavenger"`. `scavenger_success` counts branches **handled** — kept *or* deleted — not deletions.

| Panel | Description |
|---|---|
| **Scavenger Activity — Skipped vs Handled** | `scavenger_skips` (branches skipped only because they are younger than `worker.historyScannerDataMinAge`) versus `scavenger_success` (branches handled without error — kept or deleted). When skipped dwarfs handled, the 60-day wait is blocking cleanup — lower `worker.historyScannerDataMinAge`. |
| **Scavenger Errors** | `scavenger_errors` — branches the scavenger failed to process (unreadable branch, or the mutable-state lookup / history-branch delete failed). Should sit at ~0; a sustained non-zero rate is a distinct problem from the 60-day wait. |

---

## Key Dynamic Config Reference

The following dynamic configs on the standby cluster have a direct effect on what you observe in this dashboard. See the server's `dynamicconfig` package for full descriptions.

| Config Key | Default | Effect on Dashboard |
|---|---|---|
| `history.standbyClusterDelay` | 5 minutes | Sets the expected baseline for scheduled queue lag in Section 6. |
| `history.standbyTaskMissingEventsResendDelay` | 10 minutes | How long standby waits before requesting missing events from active. Drives the standby retry counter in Section 3. |
| `history.standbyTaskMissingEventsDiscardDelay` | 15 minutes | How long before a standby task with missing events is discarded entirely. The window between resend and discard delay is the recovery window. |
| `history.ReplicationTaskProcessorErrorRetryMaxAttempts` | 80 attempts | Max retries before a task is sent to DLQ (Cassandra only). Drives DLQ panels in Section 4. |
| `history.ReplicationTaskProcessorErrorRetryExpiration` | 5 minutes | Max retry duration before DLQ. |
| `history.TaskDLQEnabled` | `true` | Must be true for DLQ panels in Section 4 to emit data. Cassandra only. |
| `history.ReplicationEnableRateLimit` | `true` | Active-side rate limiting. Elevated `replication_sender_rate_limit_latency` in Section 2 is caused by this config on the active cluster. |
| `history.ReplicationStreamSyncStatusDuration` | 1 second | How frequently the stream syncs status. Affects stream health panel responsiveness in Section 1. |

---

## Relationship to the Server Dashboard

This dashboard and the [Temporal Server Dashboard](./temporal-server-readme.md) are designed to be used together for multi-cluster deployments.

| Question | Dashboard to use |
|---|---|
| Is the active cluster healthy and serving traffic normally? | Server Dashboard |
| Is the active cluster generating and sending replication tasks? | Server Dashboard → Cluster Replication section |
| Is the standby receiving and applying replication tasks? | **This dashboard** → Sections 2 and 3 |
| Has the replication stream stalled? | **This dashboard** → Section 1 |
| Are replication tasks failing past all retries (Cassandra)? | **This dashboard** → Section 4 |
| Is namespace metadata replicating correctly? | **This dashboard** → Section 5 |
| Is the standby's own infrastructure healthy enough for failover? | **This dashboard** → Section 7 |

---
