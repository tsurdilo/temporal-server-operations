# Temporal Server Operations

An operational knowledge base for self-hosted Temporal clusters. Contains Grafana dashboards, Grafana alert provisioning YAMLs, operational playbooks, and dynamic config reference, covering both the Temporal server and all Temporal SDKs.

Everything here is designed to be used directly: drop dashboards into Grafana, drop alert YAMLs into `provisioning/alerting/`, and follow playbooks against a real cluster.

Community feedback and contributions are always welcome — if something doesn't work in your environment, a threshold feels off, or you have operational knowledge worth sharing, open an issue or PR.

---

## [Playbooks](playbooks/README.md)

Production-ready operational playbooks for self-hosted Temporal clusters. Each playbook has been tested against a real cluster and cross-references the specific dashboard panels and alert rules that surface its signals.

---

## Observability

### Dashboards

- [Server Dashboards](observability/dashboards/server/README.md) — Grafana dashboards for monitoring a self-hosted Temporal Server cluster.
- [SDK Dashboards](observability/dashboards/sdk/README.md) — Grafana dashboards for monitoring Temporal SDK clients and workers (Java, Go, TypeScript, Python, .NET, Ruby).
- [Troubleshooting Dashboards](observability/dashboards/troubleshooting/README.md) — Grafana dashboards focused on troubleshooting specific Temporal operational issues.

### Alerts

- [Server Alerts](observability/alerts/server/README.md) — Grafana alerting provisioning rules for a self-hosted Temporal Server cluster. Covers the essential alert set, dual visibility store alerts, and namespace failover graceful handover alerts. Each alert links to the relevant dashboard panel and playbook section.
- [SDK Alerts](observability/alerts/sdk/README.md) — Grafana alerting provisioning rules for Temporal SDK clients and workers. One YAML per SDK reporter (Java Micrometer, Java OTel, Go, Core). Each alert links to a runbook with diagnosis and recovery steps.

### Metric Reference

- [Metrics References](observability/metric-reference/README.md) — per-metric reference docs for the Temporal server and all SDKs (Go, Java, Core).

---

## [Dynamic Config](dynamic-config/README.md)

OSS Temporal server dynamic config reference, dynamic config YAML samples, and troubleshooting info.

---

## Related Projects

- [temporal-etcd-dynconfig](https://github.com/tsurdilo/temporal-etcd-dynconfig) — etcd-backed dynamic config client for Temporal
- [temporal-configmap-dynconfig](https://github.com/tsurdilo/temporal-configmap-dynconfig) — Kubernetes ConfigMap-backed dynamic config client for Temporal
- [my-temporal-dockercompose](https://github.com/tsurdilo/my-temporal-dockercompose) — Docker Compose reference deployment of a self-hosted Temporal cluster with a full observability stack
- [temporal-helm-superchart](https://github.com/tsurdilo/temporal-helm-superchart) — Helm super-chart wrapping the upstream Temporal chart with a full observability stack