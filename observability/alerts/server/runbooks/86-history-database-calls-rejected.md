## History Database Calls Rejected

**Severity:** Warning
**Component:** history
**Store types:** All
**Dashboard panel:** [History Rejected Database Calls Total by Scope](https://github.com/tsurdilo/temporal-metrics/blob/main/observability/dashboards/server/temporal-server-readme.md) — Persistence group. It runs the same expression as this alert, so the panel and the alert always agree. Its per-namespace twin, **Rejected Database Calls by Operation and Scope**, breaks the same metric down by namespace and operation.
**Playbook:** [History Persistence QPS Limits — Rejected Database Calls Playbook](https://github.com/tsurdilo/temporal-metrics/blob/main/playbooks/history-persistence-qps-limits.md)

### What this alert detects

`persistence_errors_resource_exhausted` above 10/s for 10 minutes, on the history service, grouped by `resource_exhausted_scope` **and** `resource_exhausted_cause`.

Database calls from the history service are being rejected before they reach the store. **Check the cause label first** — it decides whether this runbook applies at all.

### Why this alert exists

Nothing else catches the real size of it. The **Resource Exhausted with Cause** panel counts requests that *failed*; most rejections never fail a request, because the call is turned away, the task waits and retries, and the request eventually succeeds. Measured on a test cluster at one moment: **90 failed requests per second while 785 database calls per second were being rejected.**

### What the cause label tells you — check this first

| `resource_exhausted_cause` | What it means | Where to go |
|---|---|---|
| `PersistenceLimit` | Temporal's own rate limiter turned the call away. The database was never asked, so this says nothing about database health. | **this runbook** |
| `SystemOverloaded` | a Cassandra node reporting itself overloaded. Cassandra only. | alert 30, System Overload Throttling |
| `PersistenceStorageLimit` | Cassandra's disk-usage guardrail, past its failure threshold. Add storage. | not a rate-limit problem |

Each cause fires as its own alert instance, so the threshold is never crossed by summing unrelated causes together.

**On a cluster running a SQL store, `PersistenceLimit` is the only cause this metric can carry** — no SQL store raises resource-exhausted at all. The other two rows apply to Cassandra only.

**Everything below assumes `PersistenceLimit`.**

### First two things to check

1. **Is a cluster-wide limit set?** If `history.persistenceGlobalMaxQPS` is above `0`, it replaces `history.persistenceMaxQPS` entirely, and the per-pod value you are looking at does nothing.
2. **Was the database actually struggling?** Check **Persistence Latencies** across the event. Flat means the limiter was the only constraint and raising it is reasonable. Climbing means the limit was doing its job — do not raise it.

### What the scope label tells you

| `resource_exhausted_scope` | Which limit |
|---|---|
| `System` | the per-pod limit |
| `Namespace` | the per-namespace **or** per-shard limit — the two share a tag and can only be told apart from the server log message |

A mix of both at once is normal; different requests hit different limits.

### Everything else

Follow the [playbook](https://github.com/tsurdilo/temporal-metrics/blob/main/playbooks/history-persistence-qps-limits.md). It covers how the five settings interact, how to work out what is actually happening, which panels read clean during throttling and will mislead you, and how to decide whether raising a limit is the right move.

### Tuning this alert

Throttling is not always a problem — a brief burst that drains on its own is the limiter working as intended. The threshold is set at 10/s sustained for 10 minutes to skip those. Raise it on a cluster that throttles routinely under normal load; lower it if you want earlier warning.
