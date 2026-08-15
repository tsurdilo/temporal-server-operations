## History Task DLQ Write Failures

**Severity:** Critical
**Component:** history
**Dashboard panel:** Archival Health group / Signal 2 — History Task DLQ Writes & Write Failures (Temporal Server Dashboard, `temporal-overview-v1`)

Writes to the history task DLQ are themselves **failing** — and failed DLQ writes are retried essentially **forever** (they are never dropped). This points to a database under pressure, and the continuous retries can add still more load and impact the cluster.

This is general — it can apply to any history task type, not just archival — but one of the situations that drives it is a sustained archival-backend outage under load, which floods the single-partition DLQ. For that scenario, and for the mechanics and remediation, see the related playbook:

➡️ **[Detecting & Recovering from an Archival Backend Outage playbook](https://github.com/tsurdilo/temporal-metrics/blob/main/playbooks/detecting-recovering-archival-outage.md)**
