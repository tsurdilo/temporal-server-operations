# Runbooks

Production-ready operational runbooks for self-hosted Temporal Server clusters.

---

## What Lives Here

A runbook graduates from [`tmp/`](../tmp/) into this directory when it meets all
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

## Runbook Format

Every runbook in this directory follows the same header block:

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

| Runbook | Scenarios | Dashboard Version | Alerts |
|---------|-----------|-------------------|--------|
| [Dual Visibility](./dual-visibility.md) | Primary failure, secondary failure, both fail — SQL-backed only (ES TBD) | v2.5.0+ | 059a, 059b, 059c |
