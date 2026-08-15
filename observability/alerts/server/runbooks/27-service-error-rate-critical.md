## Service Error Rate Critical

**Severity:** Critical
**Component:** frontend
**Dashboard panel:** [Service Errors by Namespace](https://github.com/tsurdilo/temporal-metrics/blob/main/observability/dashboards/server/temporal-server-readme.md) — panel ID 92

### What this alert detects

Frontend service error rate exceeds 30% for a specific namespace for 2 consecutive minutes.

### Why it matters

Service errors can significantly affect user namespaces and their workflows.

### Triage steps

1. Open the **Service Errors by Namespace** panel (panel 92) in the Temporal Server dashboard — identify which namespace and which operations are erroring
2. Check **Service Requests by Namespace and Operation** (panel 93) alongside errors to understand which specific operations are failing
3. Check **Persistence Latencies** (panel 71) and **Persistence Errors by Namespace** — DB slowness or errors are a common driver of elevated frontend error rates
4. Check if alert 12 (Persistence Latency Critical) or alert 30 (System Overload Throttling) are also firing — both can drive high frontend error rates
5. Check if alert 8 (Shard Lock Latency Critical) is firing — history service contention causes frontend request timeouts
6. Check the error type breakdown — `RESOURCE_EXHAUSTED` errors point to throttling, `UNAVAILABLE` errors point to service connectivity issues

### Relevant dynamic config

- `frontend.namespaceRPS` — per-namespace per-host RPS limit; if too low relative to traffic it generates `RpsLimit` errors that inflate the error rate
- `frontend.globalNamespaceRPS` — cluster-wide per-namespace RPS limit; same effect
- `frontend.maxWorkflowExecutionHistoryLength` — if exceeded, workflow tasks fail with an error that counts toward the error rate
