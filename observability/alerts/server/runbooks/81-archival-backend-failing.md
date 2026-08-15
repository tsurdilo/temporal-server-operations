## Archival Backend Failing

**Severity:** Critical
**Component:** history
**Dashboard panel:** Archival Health group / Signal 1 — Archival Attempt Error Rate (Temporal Server Dashboard, `temporal-overview-v1`)

Archival attempts to the configured backend (S3 / GCS / custom provider) are failing. A **sustained** archival-backend failure on a cluster running production workloads can escalate and destabilize the cluster — it is worth acting on early.

**This is the earliest signal.** What it means, how it escalates, how to confirm it, how to pause archival, and how to recover are all covered in one place — read it there:

➡️ **[Detecting & Recovering from an Archival Backend Outage playbook](https://github.com/tsurdilo/temporal-metrics/blob/main/playbooks/detecting-recovering-archival-outage.md)**
