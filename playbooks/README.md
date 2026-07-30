# Playbooks

Production-ready operational playbooks for self-hosted Temporal Server clusters.

---

## What Lives Here

A playbook graduates from [`tmp/`](../tmp/) into this directory when it meets all
of the following criteria:

| Criterion | What "done" looks like |
|-----------|------------------------|
| **Documented** | Written, reviewed, and accurate against server source code |
| **Dashboard** | Grafana panels exist that surface the signals described |
| **Alerts** | Alert rules wired in `metrics/alerts/server/temporal-server-alerts.yaml` |
| **Tested** | Failure scenarios exercised against a real cluster; metric signals confirmed |

Content in `tmp/` is in-progress. Do not treat it as finalized until it appears
here.

---

## Playbook Format

Every playbook in this directory follows the same header block:

```
## References

**Dashboard:** <dashboard name and version>
- Panel <id> — <panel name> — <what to look for>

**Alerts:**
- `<alert-uid>` — <alert name> — <what triggers it>

**Related config:** <dynconfig keys if applicable>
```

This makes it possible to jump directly from an alert firing to the right panel
and back.

---

## Index

| Playbook | Scenarios | Dashboard | Alerts |
|---------|-----------|-----------|--------|
| [History Host Health](./history-host-health.md) | Diagnosing and acting on `host_health` alerts — degraded pods, majority failure, silent poller failure, failover decision | [History Host Health](../metrics/dashboards/server/history-health-dashboard-readme.md) v1.3.0 | 0a, 0b, 0b-critical, 0c |
| [Dual Visibility](./dual-visibility.md) | Primary failure, secondary failure, both fail — SQL-backed only (ES TBD) | [Temporal Server](../metrics/dashboards/server/temporal-server-readme.md) v2.5.0+ | 059a, 059b, 059c |
| [Shard IO Concurrency](./shard-io-concurrency.md) | Should I raise `history.shardIOConcurrency`? — SQL only (PostgreSQL / MySQL) | [Shard IO Concurrency](../metrics/dashboards/server/shard-io-concurrency-readme.md) v1.1.0 | 034j, 034f |
| [Namespace Failover — Graceful Handover](./namespace-failover-graceful-handover.md) | Executing and monitoring a planned namespace failover using `namespace-handover-v2` — pre-flight, WaitReplication, HANDOVER drain, flip confirmation, post-handover health | [Namespace Failover — Graceful Handover](../metrics/dashboards/server/namespace-failover-graceful-handover-readme.md) v1.4.0 | FAILOVER-PRE-01, FAILOVER-PRE-02, FAILOVER-PRE-03, FAILOVER-PRE-04, FAILOVER-PRE-05, FAILOVER-PRE-06, FAILOVER-HANDOVER-01, FAILOVER-POST-01 |
| [Server Version Upgrade](./server-upgrade.md) | Upgrading a self-hosted cluster to a new version — schema migration (Cassandra / MySQL / PostgreSQL, SQL or ES visibility), binary rollout, verification, rollback, and multi-cluster order | Procedural (no dedicated dashboard) | — |
