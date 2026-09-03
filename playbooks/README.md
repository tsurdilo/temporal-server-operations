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
| **Alerts** | Alert rules wired in `observability/alerts/server/temporal-server-alerts.yaml` |
| **Tested** | Failure scenarios exercised against a real cluster; metric signals confirmed |

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
| [History Host Health](./history-host-health.md) | Diagnosing and acting on `host_health` alerts — degraded pods, majority failure, silent poller failure, failover decision | [History Host Health](../observability/dashboards/server/history-health-dashboard-readme.md) v1.3.0 | 0a, 0b, 0b-critical, 0c |
| [Dual Visibility](./dual-visibility.md) | Primary failure, secondary failure, both fail — SQL-backed only (ES TBD) | [Temporal Server](../observability/dashboards/server/temporal-server-readme.md) v2.5.0+ | 059a, 059b, 059c |
| [Shard IO Concurrency](./shard-io-concurrency.md) | Should I raise `history.shardIOConcurrency`? — SQL only (PostgreSQL / MySQL) | [Shard IO Concurrency](../observability/dashboards/server/shard-io-concurrency-readme.md) v1.1.0 | 034j, 034f |
| [Namespace Failover — Graceful Handover](./namespace-failover-graceful-handover.md) | Executing and monitoring a planned namespace failover using `namespace-handover-v2` — pre-flight, WaitReplication, HANDOVER drain, flip confirmation, post-handover health | [Namespace Failover — Graceful Handover](../observability/dashboards/server/namespace-failover-graceful-handover-readme.md) v1.6.0 | FAILOVER-PRE-01, FAILOVER-PRE-02, FAILOVER-PRE-03, FAILOVER-PRE-04, FAILOVER-PRE-05, FAILOVER-PRE-06, FAILOVER-HANDOVER-01, FAILOVER-POST-01 |
| [Server Version Upgrade](./server-upgrade.md) | Upgrading a self-hosted cluster to a new version — schema migration (Cassandra / MySQL / PostgreSQL, SQL or ES visibility), binary rollout, verification, rollback, and multi-cluster order | Procedural (no dedicated dashboard) | — |
| [Detecting & Recovering from an Archival Backend Outage](./detecting-recovering-archival-outage.md) | A sustained archival backend (S3 / GCS / custom) outage under load — how the history task DLQ works, cluster impact, early detection, pausing archival, and recovery. Cassandra and SQL alike. | [Temporal Server](../observability/dashboards/server/temporal-server-readme.md#21-archival-health) v2.11.0+ | 081, 082 |
| [Changing Task Queue Partitions](./change-task-queue-partitions.md) | Changing a task queue's partition count safely — deciding when to increase or decrease, the drain-first decrease order, detecting and fixing a `Write > Read` misconfiguration, and why adding partitions won't drain an existing backlog | [Task Queue Partitions](../observability/dashboards/server/task-queue-partitions-readme.md) v1.4.0 | 83 (optional — documented, not in essential set) |
| [Hot Shard — Detection & Remediation](./hot-shard-detection-remediation.md) | One or a few history shards doing far more work than the rest — slowing every workflow on them, turning one history host into a resource hotspot, and (on a multi-tenant cluster) degrading other namespaces sharing the shard. Covers the two kinds (many different workflow IDs crowding a shard vs a single hot workflow ID), detecting it from the per-shard RPS distribution, finding the shard and the workflow ID or namespace behind it, and remediating with server-side start-rate and per-shard-namespace guardrails, shard count, and app-side pattern fixes. Cassandra and SQL alike. | [Temporal Server](../observability/dashboards/server/temporal-server-readme.md#3-persistence-requests-latencies-and-errors) v2.14.0+ | 84 (planned) |
| [XDC Standby Database Growth on SQL](./xdc-standby-database-growth-sql.md) | A standby cluster's database growing much larger than the active for a global namespace — completed/deleted-workflow cleanup falling behind. Covers the two gaps (`executions` rows the active already deleted but weren't replicated; leftover `history_tree` / `history_node` from failed deletes or the scavenger's 60-day wait), detecting them from scavenger metrics and SQL, remediating with a throttled admin delete plus lowering `worker.historyScannerDataMinAge`, and preventing recurrence via `history.enableDeleteWorkflowExecutionReplication`. SQL persistence only (Cassandra playbook TBD). | [Temporal Server](../observability/dashboards/server/temporal-server-readme.md#22-history-scavenger) v2.14.0+ / [Temporal Standby](../observability/dashboards/server/temporal-standby-readme.md#8-history-scavenger) v2.2.0+ | 85 (optional — documented, not in essential set) |
