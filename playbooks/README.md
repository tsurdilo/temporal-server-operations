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
| [Dual Visibility](./dual-visibility.md) | Primary failure, secondary failure, both fail — SQL-backed only (ES TBD) | [Temporal Server](../metrics/dashboards/server/temporal-server-readme.md) v2.5.0+ | 059a, 059b, 059c |
| [Shard IO Concurrency](./shard-io-concurrency.md) | Should I raise `history.shardIOConcurrency`? — SQL only (PostgreSQL / MySQL) | [Shard IO Concurrency](../metrics/dashboards/server/shard-io-concurrency-readme.md) v1.1.0 | 034j, 034f |
