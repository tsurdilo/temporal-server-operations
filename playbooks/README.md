# Playbooks

Playbooks for running Temporal at scale on self-hosted clusters. They come in two groups:

- **[Use-case playbooks](#use-case-playbooks)**, for developers who write Temporal applications and
  for the operators who run the cluster. Each one starts from a workload pattern. It explains the
  pressure the pattern puts on the cluster, shows how to see it on the dashboards, and recommends
  changes to the application, the SDK workers and the cluster.
- **[Server playbooks](#server-playbooks)**, for Temporal cluster operators. Each one starts from a
  server symptom or a server component. It explains how that part of the server works internally,
  how to detect problems with it on the dashboards, and how to tune the cluster for large scale.

---

## What Lives Here

A playbook belongs in this directory once it meets all of the following criteria:

| Criterion | What "done" looks like |
|-----------|------------------------|
| **Documented** | Written, reviewed, and accurate against server source code |
| **Dashboard** | Grafana panels exist that surface the signals described |
| **Alerts** | Alert rules wired in `observability/alerts/server/temporal-server-alerts.yaml` |
| **Tested** | Failure scenarios exercised against a real cluster; metric signals confirmed |

Use-case playbooks list the existing alerts that cover the pressure they describe; a workload pattern
is not a failure, so it does not get an alert of its own.

---

## Playbook Format

There is no fixed template — each playbook is shaped around how its problem presents. What they all
do is name the dashboard groups and panels they rely on, and the alerts that fire for them, so you
can get from an alert to the right panel and back. A few are procedural and have neither.

---

## Use-case playbooks

For developers who write Temporal applications and for the operators who run the cluster. Each
playbook starts from what the application does, and covers both the application side and the
cluster side.

| Playbook | Use case | Dashboards | Alerts |
|---------|----------|------------|--------|
| [The Sleep-and-Wake Pattern — Many Timers Firing at the Same Time](./use-case-sleep-and-wake-pattern.md) | Many workflows that sleep and wake up close to each other, such as reminders, follow-ups or daily jobs. How to optimize both the workflow code and the cluster to run this pattern at large scale. | [Temporal Server dashboard](../observability/dashboards/server/temporal-server-readme.md) v2.32.0+, [SDK dashboards](../observability/dashboards/sdk/README.md) | [5](../observability/alerts/server/alerts-index.md#alert-5--namespace-rps-approaching-limit), [6](../observability/alerts/server/alerts-index.md#alert-6--namespace-rps-over-limit), [11](../observability/alerts/server/alerts-index.md#alert-11--persistence-latency-high), [13](../observability/alerts/server/alerts-index.md#alert-13--persistence-qps-approaching-limit), [29](../observability/alerts/server/alerts-index.md#alert-29--rpsqps-throttling-active), [35](../observability/alerts/server/alerts-index.md#alert-35--timer-task-processing-latency-high), [42](../observability/alerts/server/alerts-index.md#alert-42--workflow-history-size-warning), [46](../observability/alerts/server/alerts-index.md#alert-46--mutable-state-size-warning), [48](../observability/alerts/server/alerts-index.md#alert-48--schedule-to-start-latency-high), [50](../observability/alerts/server/alerts-index.md#alert-50--task-backlog-growing), [52](../observability/alerts/server/alerts-index.md#alert-52--tasks-persisted-rate-high), [56](../observability/alerts/server/alerts-index.md#alert-56--workflow-task-timeout-sticky), [58](../observability/alerts/server/alerts-index.md#alert-58--pollers-approaching-concurrent-limit), [76](../observability/alerts/server/alerts-index.md#alert-76--workflow-task-scheduletostart-timeout), [80](../observability/alerts/server/alerts-index.md#alert-80--history-task-dlq-stranding), [87](../observability/alerts/server/alerts-index.md#alert-87--history-write-reject-loop), [88](../observability/alerts/server/alerts-index.md#alert-88--task-queue-partition-backlog-not-draining) |

---

## Server playbooks

For Temporal cluster operators. Each playbook starts from a server symptom or a server component,
explains how that part of the server works internally, and recommends how to tune the cluster for
large scale.

### History service

| Playbook | Scenarios | Dashboard | Alerts |
|---------|-----------|-----------|--------|
| [History Task Processing — Tuning and Troubleshooting](./history-task-processing-tuning.md) | How the history service loads, schedules and runs its tasks, and how to tune task processing for large scale. How to detect and stop the write-reject loop, where refused database writes turn into repeated reads. Any persistence store; single- or multi-cluster. | [Temporal Server dashboard](../observability/dashboards/server/temporal-server-readme.md#9-shard-queue-health) v2.17.0+ | [87](../observability/alerts/server/alerts-index.md#alert-87--history-write-reject-loop) |
| [History Task Table Growth — Stuck Queue Cleanup](./history-task-table-growth.md) | A task table growing because rows are never deleted, not because work is backing up — one task that cannot finish stops row cleanup for every namespace on its shard. How to tell retained rows from a real backlog, find the shard and namespace responsible, get cleanup running again, and clear what the server can no longer reach. Any persistence store. | [Temporal Server dashboard](../observability/dashboards/server/temporal-server-readme.md#23-history-task-cleanup) v2.25.0+ | [91](../observability/alerts/server/alerts-index.md#alert-91--task-row-cleanup-failing) |
| [History Persistence QPS Limits — Rejected Database Calls](./history-persistence-qps-limits.md) | `RESOURCE_EXHAUSTED` with cause `PersistenceLimit` — the history service's own limiter turning database calls away before they reach the store. How the five settings interact, which one is doing the rejecting, and whether to raise it or leave it alone. Any persistence store. | [Temporal Server dashboard](../observability/dashboards/server/temporal-server-readme.md#3-persistence-requests-latencies-and-errors) v2.16.0+ | [86](../observability/alerts/server/alerts-index.md#alert-86--history-database-calls-rejected) |
| [Hot Shard — Detection & Remediation](./hot-shard-detection-remediation.md) | One or a few history shards doing far more work than the rest, slowing every workflow on them and overloading a history host. Covers the two kinds (many workflow IDs crowding a shard vs one hot workflow ID), how to find the shard and the cause, and how to remediate. Cassandra and SQL alike. | [Temporal Server dashboard](../observability/dashboards/server/temporal-server-readme.md#3-persistence-requests-latencies-and-errors) v2.14.0+ | [89](../observability/alerts/server/alerts-index.md#alert-89--hot-shard-uneven-per-shard-load) |
| [History Growth from Duplicate Workflow Starts](./history-growth-duplicate-workflow-starts.md) | The `history_node` / `history_tree` tables growing because duplicate workflow starts leave history behind (a start writes its history before the duplicate is detected). How to detect it, clear it with the scavenger, and prevent it. Any persistence store; single- or multi-cluster. | [Temporal Server dashboard](../observability/dashboards/server/temporal-server-readme.md#5-service-requests-and-errors) v2.14.0+ | [90](../observability/alerts/server/alerts-index.md#alert-90--history-scavenger-errors) |
| [Shard IO Concurrency on PostgreSQL and MySQL — Tuning](./shard-io-concurrency.md) | Whether raising `history.shardIOConcurrency` will help, how high to go, and how to roll it out. PostgreSQL and MySQL only. | [Shard IO Concurrency dashboard](../observability/dashboards/server/shard-io-concurrency-readme.md) v1.3.0 | [34j](../observability/alerts/server/alerts-index.md#alert-34j--shard-io-semaphore-deadlock-approaching), [34f](../observability/alerts/server/alerts-index.md#alert-34f--shard-deadlock-detected) |
| [History Host Health](./history-host-health.md) | Diagnosing and acting on `host_health` alerts — degraded pods, majority failure, silent poller failure, failover decision | [History Host Health dashboard](../observability/dashboards/server/history-health-dashboard-readme.md) v1.3.0 | [0a](../observability/alerts/server/alerts-index.md#alert-0a--history-pod-disappeared), [0b](../observability/alerts/server/alerts-index.md#alert-0b--history-pod-degraded-warning), [0b-critical](../observability/alerts/server/alerts-index.md#alert-0b-critical--history-fleet-majority-degraded-critical), [0c](../observability/alerts/server/alerts-index.md#alert-0c--metric-freshness-stale) |

### Matching service

| Playbook | Scenarios | Dashboard | Alerts |
|---------|-----------|-----------|--------|
| [Changing Task Queue Partitions](./change-task-queue-partitions.md) | Changing a task queue's partition count safely — deciding when to increase or decrease, the drain-first decrease order, detecting and fixing a `Write > Read` misconfiguration, and why adding partitions won't drain an existing backlog | [Task Queue Partitions dashboard](../observability/dashboards/server/task-queue-partitions-readme.md) v1.4.0 | [88](../observability/alerts/server/alerts-index.md#alert-88--task-queue-partition-backlog-not-draining) |

### Cluster operations

| Playbook | Scenarios | Dashboard | Alerts |
|---------|-----------|-----------|--------|
| [Server Version Upgrade](./server-upgrade.md) | Upgrading a self-hosted cluster to a new version — schema migration (Cassandra / MySQL / PostgreSQL, SQL or ES visibility), binary rollout, verification, rollback, and multi-cluster order | Procedural (no dedicated dashboard) | None |
| [Idle Cluster Resource Utilization — Baseline](./idle-cluster-resource-utilization.md) | What a cluster uses with no workload on it — database calls, memory, CPU and metric series — and how much comes from service instances, history shards and namespaces. How to measure your own baseline, and calculate one for a cluster you are planning. Single cluster; v1.19+. | [Temporal Server dashboard](../observability/dashboards/server/temporal-server-readme.md#24-resting-load-idle-baseline) v2.28.0+ | None |

### Multi-cluster and replication

| Playbook | Scenarios | Dashboard | Alerts |
|---------|-----------|-----------|--------|
| [Namespace Failover — Graceful Handover](./namespace-failover-graceful-handover.md) | Executing and monitoring a planned namespace failover using `namespace-handover-v2` — pre-flight, WaitReplication, HANDOVER drain, flip confirmation, post-handover health | [Namespace Failover — Graceful Handover dashboard](../observability/dashboards/server/namespace-failover-graceful-handover-readme.md) v1.6.0 | [FAILOVER-PRE-01](../observability/alerts/server/alerts-index.md#alert-failover-pre-01--replication-stream-stuck-pre-handover), [FAILOVER-PRE-02](../observability/alerts/server/alerts-index.md#alert-failover-pre-02--replication-dlq-enqueue-failing), [FAILOVER-PRE-03](../observability/alerts/server/alerts-index.md#alert-failover-pre-03--replication-stream-reconnect-rate-elevated), [FAILOVER-PRE-04](../observability/alerts/server/alerts-index.md#alert-failover-pre-04--receiver-backlog-at-flow-control-limit), [FAILOVER-PRE-05](../observability/alerts/server/alerts-index.md#alert-failover-pre-05--receiver-backlog-near-limit), [FAILOVER-PRE-06](../observability/alerts/server/alerts-index.md#alert-failover-pre-06--standby-task-discards-detected), [FAILOVER-HANDOVER-01](../observability/alerts/server/alerts-index.md#alert-failover-handover-01--handover-drain-stalled), [FAILOVER-POST-01](../observability/alerts/server/alerts-index.md#alert-failover-post-01--forwarding-failedprecondition-errors) |
| [XDC Standby Database Growth on SQL](./xdc-standby-database-growth-sql.md) | A standby cluster's database growing much larger than the active for a global namespace, because completed-workflow cleanup falls behind. How to detect the two gaps (leftover records and leftover history), remediate, and prevent recurrence. SQL persistence only (Cassandra playbook TBD). | [Temporal Server dashboard](../observability/dashboards/server/temporal-server-readme.md#22-history-scavenger) v2.14.0+ / [Temporal Standby dashboard](../observability/dashboards/server/temporal-standby-readme.md#8-history-scavenger) v2.2.0+ | [90](../observability/alerts/server/alerts-index.md#alert-90--history-scavenger-errors) |

### Server internal workflows

**Temporal runs some operations as workflows on the cluster itself**, in the `temporal-system`
namespace, rather than as a synchronous API call. Playbooks here cover the details of those
internal system workflows.

| Playbook | Scenarios | Dashboard | Alerts |
|---------|-----------|-----------|--------|
| [Namespace Deletion — System Workflow](./namespace-deletion.md) | Deleting a namespace on a running cluster — what the internal workflows do, why a success response does not mean the namespace is gone, when a deletion is the wrong tool, how to follow one and restart one that stopped. All persistence stores; single and multi-cluster. | [Temporal Server dashboard](../observability/dashboards/server/temporal-server-readme.md#25-namespace-deletion) v2.27.0+ | [92](../observability/alerts/server/alerts-index.md#alert-92--namespace-deletion-failed) |

### Visibility and archival

| Playbook | Scenarios | Dashboard | Alerts |
|---------|-----------|-----------|--------|
| [Dual Visibility Operations](./dual-visibility.md) | Running visibility on two stores at once — configuring the pair, detecting which store is failing, remediating before records are dropped, and moving reads and writes for a failover or migration. Both stores SQL (PostgreSQL / MySQL) or both Elasticsearch. | [Temporal Server dashboard](../observability/dashboards/server/temporal-server-readme.md#16-visibility) v2.15.0+ | [59a](../observability/alerts/server/alerts-index.md#alert-59a--visibility-store-write-errors-warning), [59b](../observability/alerts/server/alerts-index.md#alert-59b--visibility-store-write-errors-critical), [59c](../observability/alerts/server/alerts-index.md#alert-59c--visibility-store-write-latency-high), [83](../observability/alerts/server/alerts-index.md#alert-83--visibility-tasks-dead-lettered), [84](../observability/alerts/server/alerts-index.md#alert-84--visibility-store-not-acknowledging-writes), [85](../observability/alerts/server/alerts-index.md#alert-85--visibility-read-errors) |
| [Detecting & Recovering from an Archival Backend Outage](./detecting-recovering-archival-outage.md) | A sustained archival backend (S3 / GCS / custom) outage under load — how the history task DLQ works, cluster impact, early detection, pausing archival, and recovery. Cassandra and SQL alike. | [Temporal Server dashboard](../observability/dashboards/server/temporal-server-readme.md#21-archival-health) v2.11.0+ | [81](../observability/alerts/server/alerts-index.md#alert-81--archival-backend-failing), [82](../observability/alerts/server/alerts-index.md#alert-82--history-task-dlq-write-failures) |
