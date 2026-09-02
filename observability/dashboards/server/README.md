# Temporal Server Dashboards

Grafana dashboards for monitoring a self-hosted Temporal Server cluster.

---

## Dashboards

### Temporal Server Dashboard — v2.14.0

Comprehensive cluster monitoring: throughput, persistence, service latencies, throttling, shard movement, workflow stats, matching, replication, and authorization.

| File | Description |
|---|---|
| [temporal-server.json](./temporal-server.json) | Grafana dashboard JSON — import this into Grafana |
| [temporal-server-readme.md](./temporal-server-readme.md) | Full panel reference and setup guide |
| [temporal-server-changelog.md](./temporal-server-changelog.md) | Version history |

---

### Temporal Standby Cluster — Replication Health — v2.2.0

Dedicated standby-side monitoring: replication lag, DLQ depth, standby task retries, stream health, and failover readiness.

| File | Description |
|---|---|
| [temporal-standby.json](./temporal-standby.json) | Grafana dashboard JSON — import this into Grafana |
| [temporal-standby-readme.md](./temporal-standby-readme.md) | Full panel reference and setup guide |
| [temporal-standby-changelog.md](./temporal-standby-changelog.md) | Version history |

---

### Temporal History Host Health Dashboard — v1.3.0

Deep health monitoring for the history service fleet: `host_health` state, gRPC readiness, persistence health, RPC health, and shard acquisition and movement.

| File | Description |
|---|---|
| [history-health-dashboard.json](./history-health-dashboard.json) | Grafana dashboard JSON — import this into Grafana |
| [history-health-dashboard-readme.md](./history-health-dashboard-readme.md) | Full panel reference and setup guide |
| [history-health-dashboard-changelog.md](./history-health-dashboard-changelog.md) | Version history |

---

### Temporal Shard IO Concurrency — v1.1.0

Decision guide for tuning `history.shardIOConcurrency`: shard IO semaphore health (latency and failures), the max-safe-value-per-pod ceiling, and the DB prerequisite check. SQL backends only (PostgreSQL, MySQL) — not applicable to Cassandra. Companion to the Shard IO Concurrency playbook.

| File | Description |
|---|---|
| [shard-io-concurrency.json](./shard-io-concurrency.json) | Grafana dashboard JSON — import this into Grafana |
| [shard-io-concurrency-readme.md](./shard-io-concurrency-readme.md) | Full panel reference and decision guide |
| [shard-io-concurrency-changelog.md](./shard-io-concurrency-changelog.md) | Version history |

---

### Namespace Failover — Graceful Handover Dashboard — v1.6.0

Pre-flight go/no-go checks, WaitReplication catchup progress, HANDOVER drain, flip confirmation, and post-handover health for planned namespace failovers using `namespace-handover-v2`. Multi-cluster replication only.

| File | Description |
|---|---|
| [namespace-failover-graceful-handover.json](./namespace-failover-graceful-handover.json) | Grafana dashboard JSON — import this into Grafana |
| [namespace-failover-graceful-handover-readme.md](./namespace-failover-graceful-handover-readme.md) | Full panel reference |
| [namespace-failover-graceful-handover-changelog.md](./namespace-failover-graceful-handover-changelog.md) | Version history |

---

### Temporal Task Queue Partitions — v1.4.0

Single-queue drill-down for changing a task queue's partition count safely: per-partition backlog drain and `Write > Read` detection, the three throughput ceilings (sync match, backlog write, backlog read+dispatch), and the rule-outs that tell you whether matching is even the bottleneck. Companion to the Changing Task Queue Partitions playbook.

| File | Description |
|---|---|
| [task-queue-partitions.json](./task-queue-partitions.json) | Grafana dashboard JSON — import this into Grafana |
| [task-queue-partitions-readme.md](./task-queue-partitions-readme.md) | Full panel reference and prerequisites |
| [task-queue-partitions-changelog.md](./task-queue-partitions-changelog.md) | Version history |

---

## Updating to a new version

When a new version is available, reimport the updated JSON into Grafana via **Dashboards → Import**. Because the dashboard UID is stable across versions, Grafana will update your existing dashboard in place rather than creating a duplicate. Check the relevant changelog before updating to see what changed.