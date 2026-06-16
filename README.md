# Temporal Server Operations

This repository includes various Temporal Server operational artifacts.

---

## [Dynamic Config](dynamic_config/README.md)

OSS Temporal server dynamic config reference, dynamic config YAML samples, and troubleshooting info.

---

## [Metrics](metrics/references/README.md)

OSS Temporal server metrics references.

### Dashboards

- [Server Dashboards](metrics/dashboards/server/README.md) — Grafana dashboards for monitoring a self-hosted Temporal Server cluster, including:
    - [Server Overview Dashboard](metrics/dashboards/server/temporal-server-readme.md) — cluster health, throughput, persistence, and service metrics
    - [Standby Cluster Dashboard](metrics/dashboards/server/temporal-standby-readme.md) — replication health, lag, and failover readiness for standby clusters
    - [History Host Health Dashboard](metrics/dashboards/server/history-health-dashboard-readme.md) — per-pod `host_health` gauge, NOT_SERVING detection, and fleet-level aggregation
- [SDK Dashboards](metrics/dashboards/sdk/README.md) — Grafana dashboards for monitoring Temporal SDK clients and workers (Java, Go, TypeScript, Python, .NET, Ruby).
- [Troubleshooting Dashboards](metrics/dashboards/troubleshooting/README.md) — Grafana dashboards focused on troubleshooting specific Temporal operational issues.

### Alerts

- [Server Alerts](metrics/alerts/server/README.md) — Grafana alerting provisioning rules for a self-hosted Temporal Server cluster. Covers the essential alert set plus dual visibility store alerts. Each alert links to a runbook with diagnosis and recovery steps.
- [SDK Alerts](metrics/alerts/sdk/README.md) — Grafana alerting provisioning rules for Temporal SDK clients and workers. One YAML per SDK reporter (Java Micrometer, Java OTel, Go, Core). Each alert links to a runbook with diagnosis and recovery steps.

---

## [Playbooks](playbooks/README.md)

Production-ready operational playbooks for self-hosted Temporal clusters. Each playbook has been tested against a real cluster and cross-references the specific dashboard panels and alert rules that surface its signals.

Content graduates here from `tmp/` once it meets the full criteria: documented, dashboard panels in place, alerts wired up, and failure scenarios tested live.

| Playbook | Covers |
|---------|--------|
| [Dual Visibility](playbooks/dual-visibility.md) | Primary failure, secondary failure, both stores fail — detection via `visibility_persistence_*` metrics, recovery steps, write mode management. **SQL-backed dual visibility only** — Elasticsearch coverage TBD. |