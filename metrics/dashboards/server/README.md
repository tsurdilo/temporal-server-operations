# Temporal Server Dashboards

Grafana dashboards for monitoring a self-hosted Temporal Server cluster.

---

## Dashboards

### Temporal Server Dashboard — v2.6.0

Comprehensive cluster monitoring: throughput, persistence, service latencies, throttling, shard movement, workflow stats, matching, replication, and authorization.

| File | Description |
|---|---|
| [temporal-server.json](./temporal-server.json) | Grafana dashboard JSON — import this into Grafana |
| [temporal-server-readme.md](./temporal-server-readme.md) | Full panel reference and setup guide |
| [temporal-server-changelog.md](./temporal-server-changelog.md) | Version history |

---

### Temporal Standby Cluster — Replication Health — v2.0.0

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

### Namespace Failover — Graceful Handover Dashboard — v1.4.0

Pre-flight go/no-go checks, WaitReplication catchup progress, HANDOVER drain, flip confirmation, and post-handover health for planned namespace failovers using `namespace-handover-v2`. Multi-cluster replication only.

| File | Description |
|---|---|
| [namespace-failover-graceful-handover.json](./namespace-failover-graceful-handover.json) | Grafana dashboard JSON — import this into Grafana |
| [namespace-failover-graceful-handover-readme.md](./namespace-failover-graceful-handover-readme.md) | Full panel reference |
| [namespace-failover-graceful-handover-changelog.md](./namespace-failover-graceful-handover-changelog.md) | Version history |

---

## Updating to a new version

When a new version is available, reimport the updated JSON into Grafana via **Dashboards → Import**. Because the dashboard UID is stable across versions, Grafana will update your existing dashboard in place rather than creating a duplicate. Check the relevant changelog before updating to see what changed.