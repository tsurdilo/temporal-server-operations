# Playbooks

Production-ready operational playbooks for self-hosted Temporal Server clusters.

---

## What Lives Here

A playbook belongs in this directory once it meets all of the following criteria:

| Criterion | What "done" looks like |
|-----------|------------------------|
| **Documented** | Written, reviewed, and accurate against server source code |
| **Dashboard** | Grafana panels exist that surface the signals described |
| **Alerts** | Alert rules wired in `observability/alerts/server/temporal-server-alerts.yaml` |
| **Tested** | Failure scenarios exercised against a real cluster; metric signals confirmed |

---

## Playbook Format

There is no fixed template — each playbook is shaped around how its problem presents. What they all
do is name the dashboard groups and panels they rely on, and the alerts that fire for them, so you
can get from an alert to the right panel and back. A few are procedural and have neither.

---

## Index

### History service

| Playbook | Scenarios | Dashboard | Alerts |
|---------|-----------|-----------|--------|
| [History Task Processing — Tuning and Troubleshooting](./history-task-processing-tuning.md) | **Doubles as a reference on how the history service processes tasks.** The three parts — loading, scheduling, executing — where each one runs, what limits it, and how the scheduler already prioritises work across namespaces, priorities and clusters. Then the failure it exists for: the **write-reject loop**, where a refused write makes the pod discard cached workflow state, every retry has to read that state back, and the reads crowd out the writes that would end it. How to confirm the loop from one number, size the task scheduler's rate limits (**off by default**, and `0` does not mean unlimited) so tasks that start can finish, pace how fast tasks are read out of the database, and what to do when tuning has run out of room. Every default, ratio and behaviour verified against server source. Any persistence store; single- or multi-cluster. | [Temporal Server](../observability/dashboards/server/temporal-server-readme.md#9-shard-queue-health) v2.17.0+ | 87 |
| [History Persistence QPS Limits — Rejected Database Calls](./history-persistence-qps-limits.md) | `RESOURCE_EXHAUSTED` with cause `PersistenceLimit` — the history service's own limiter turning database calls away before they reach the store. How the five settings interact, which one is doing the rejecting, and whether to raise it or leave it alone. Any persistence store. | [Temporal Server](../observability/dashboards/server/temporal-server-readme.md#3-persistence-requests-latencies-and-errors) v2.16.0+ | 86 |
| [Hot Shard — Detection & Remediation](./hot-shard-detection-remediation.md) | One or a few history shards doing far more work than the rest, slowing every workflow on them and overloading a history host. Covers the two kinds (many workflow IDs crowding a shard vs one hot workflow ID), how to find the shard and the cause, and how to remediate. Cassandra and SQL alike. | [Temporal Server](../observability/dashboards/server/temporal-server-readme.md#3-persistence-requests-latencies-and-errors) v2.14.0+ | 89 (planned) |
| [History Growth from Duplicate Workflow Starts](./history-growth-duplicate-workflow-starts.md) | The `history_node` / `history_tree` tables growing because duplicate workflow starts leave history behind (a start writes its history before the duplicate is detected). How to detect it, clear it with the scavenger, and prevent it. Any persistence store; single- or multi-cluster. | [Temporal Server](../observability/dashboards/server/temporal-server-readme.md#5-service-requests-and-errors) v2.14.0+ | 90 (optional) |
| [Shard IO Concurrency](./shard-io-concurrency.md) | Should I raise `history.shardIOConcurrency`? — SQL only (PostgreSQL / MySQL) | [Shard IO Concurrency](../observability/dashboards/server/shard-io-concurrency-readme.md) v1.1.0 | 034j, 034f |
| [History Host Health](./history-host-health.md) | Diagnosing and acting on `host_health` alerts — degraded pods, majority failure, silent poller failure, failover decision | [History Host Health](../observability/dashboards/server/history-health-dashboard-readme.md) v1.3.0 | 0a, 0b, 0b-critical, 0c |

### Matching service

| Playbook | Scenarios | Dashboard | Alerts |
|---------|-----------|-----------|--------|
| [Changing Task Queue Partitions](./change-task-queue-partitions.md) | Changing a task queue's partition count safely — deciding when to increase or decrease, the drain-first decrease order, detecting and fixing a `Write > Read` misconfiguration, and why adding partitions won't drain an existing backlog | [Task Queue Partitions](../observability/dashboards/server/task-queue-partitions-readme.md) v1.4.0 | 88 (optional — documented, not in essential set) |

### Cluster operations

| Playbook | Scenarios | Dashboard | Alerts |
|---------|-----------|-----------|--------|
| [Server Version Upgrade](./server-upgrade.md) | Upgrading a self-hosted cluster to a new version — schema migration (Cassandra / MySQL / PostgreSQL, SQL or ES visibility), binary rollout, verification, rollback, and multi-cluster order | Procedural (no dedicated dashboard) | — |

### Multi-cluster and replication

| Playbook | Scenarios | Dashboard | Alerts |
|---------|-----------|-----------|--------|
| [Namespace Failover — Graceful Handover](./namespace-failover-graceful-handover.md) | Executing and monitoring a planned namespace failover using `namespace-handover-v2` — pre-flight, WaitReplication, HANDOVER drain, flip confirmation, post-handover health | [Namespace Failover — Graceful Handover](../observability/dashboards/server/namespace-failover-graceful-handover-readme.md) v1.6.0 | FAILOVER-PRE-01, FAILOVER-PRE-02, FAILOVER-PRE-03, FAILOVER-PRE-04, FAILOVER-PRE-05, FAILOVER-PRE-06, FAILOVER-HANDOVER-01, FAILOVER-POST-01 |
| [XDC Standby Database Growth on SQL](./xdc-standby-database-growth-sql.md) | A standby cluster's database growing much larger than the active for a global namespace, because completed-workflow cleanup falls behind. How to detect the two gaps (leftover records and leftover history), remediate, and prevent recurrence. SQL persistence only (Cassandra playbook TBD). | [Temporal Server](../observability/dashboards/server/temporal-server-readme.md#22-history-scavenger) v2.14.0+ / [Temporal Standby](../observability/dashboards/server/temporal-standby-readme.md#8-history-scavenger) v2.2.0+ | 90 (optional — documented, not in essential set) |

### Visibility and archival

| Playbook | Scenarios | Dashboard | Alerts |
|---------|-----------|-----------|--------|
| [Dual Visibility Operations](./dual-visibility.md) | Running visibility on two stores at once — configuring the pair, detecting which store is failing, remediating before records are dropped, and moving reads and writes for a failover or migration. Both stores SQL (PostgreSQL / MySQL) or both Elasticsearch. | [Temporal Server](../observability/dashboards/server/temporal-server-readme.md#16-visibility) v2.15.0+ | 059a, 059b, 059c, 083, 084, 085 |
| [Detecting & Recovering from an Archival Backend Outage](./detecting-recovering-archival-outage.md) | A sustained archival backend (S3 / GCS / custom) outage under load — how the history task DLQ works, cluster impact, early detection, pausing archival, and recovery. Cassandra and SQL alike. | [Temporal Server](../observability/dashboards/server/temporal-server-readme.md#21-archival-health) v2.11.0+ | 081, 082 |
