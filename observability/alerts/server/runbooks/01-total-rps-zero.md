## Total RPS Drops to Zero

**Severity:** Critical
**Component:** frontend
**Dashboard panel:** [Total RPS](https://github.com/tsurdilo/temporal-metrics/blob/main/observability/dashboards/server/temporal-server-readme.md) — panel ID 21

### What this alert detects

Total `service_requests` rate across all frontend instances drops to zero for 5 consecutive minutes. Because the metric is still being reported, at least one frontend pod is up and being scraped by Prometheus — but the cluster is receiving no incoming requests. If the frontend pods themselves are gone, alert 1b fires instead.

### Why it matters

The Temporal frontend service is not reachable. This means clients cannot connect to the cluster, but it also means other Temporal services (such as the worker service) that communicate through the frontend are equally cut off. The entire cluster is effectively unavailable.

### Triage steps

1. Open the **Total RPS** panel (panel 21) in the Temporal Server dashboard — confirm the drop and note when it started
2. Check frontend pod health — are all pods running and passing readiness checks?
3. Check `service_errors` rate — if errors are elevated the frontend is receiving requests but failing them; note that zero errors does not mean everything is fine, a frontend that cannot receive requests also cannot count errors

### Relevant dynamic config

- `frontend.rps` — per-host RPS limit; if set to `0` the frontend rejects all incoming requests
- `frontend.globalNamespaceRPS` — cluster-wide namespace RPS limit; if set to `0` all namespace-scoped requests are rejected
