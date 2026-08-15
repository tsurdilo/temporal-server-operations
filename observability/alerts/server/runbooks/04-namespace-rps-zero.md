## Namespace RPS Drops to Zero

**Severity:** Critical
**Component:** frontend
**Dashboard panel:** [RPS per Namespace](https://github.com/tsurdilo/temporal-metrics/blob/main/observability/dashboards/server/temporal-server-readme.md) — panel ID 20

### What this alert detects

`service_requests{service_name="frontend"}` rate drops to zero for a specific namespace for 5 consecutive minutes. Unlike alert 1 which fires when the entire cluster goes silent, this fires per namespace — other namespaces may be healthy while one goes dark. The `_unknown_` namespace is excluded.

### Why it matters

Workers for that namespace have stopped receiving task assignments and clients can no longer interact with workflows in that namespace. The impact is scoped — other namespaces are unaffected — but for the affected namespace it is a complete processing halt.

### Triage steps

1. Open the **RPS per Namespace** panel (panel 20) in the Temporal Server dashboard — confirm which namespace dropped and when
2. Check if alert 1 is also firing — if so this is a cluster-wide outage, not namespace-specific
3. Check if workers for that namespace are running and connected
4. Check if the namespace exists and is not deleted or archived

### Relevant dynamic config

- `frontend.namespaceRPS` — per-namespace per-host RPS limit; if set to `0` all requests for that namespace are rejected
- `frontend.globalNamespaceRPS` — cluster-wide per-namespace RPS limit; if set to `0` same effect
