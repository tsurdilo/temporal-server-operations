## System Overload Throttling

**Severity:** Critical
**Component:** persistence
**Dashboard panel:** [Resource Exhausted with Cause](https://github.com/tsurdilo/temporal-metrics/blob/main/observability/dashboards/server/temporal-server-readme.md) — panel ID 121

### What this alert detects

Any non-zero rate of `service_errors_resource_exhausted` with cause `SystemOverloaded` or `CircuitBreakerOpen` for 1 consecutive minute.

### Why it matters

Unlike RPS or QPS limits which are operator-configured and expected, `SYSTEM_OVERLOADED` and `CIRCUIT_BREAKER_OPEN` are dynamic self-protection responses — the cluster is actively shedding load because it cannot keep up. Clients are receiving errors right now and workflows are not making progress.

### Triage steps

1. Open the **Resource Exhausted with Cause** panel (panel 121) in the Temporal Server dashboard — confirm which cause is active and the rate
2. Check **Persistence Latencies** (panel 71) — `SYSTEM_OVERLOADED` is almost always triggered by DB latency exceeding internal thresholds; check if alert 12 is also firing
3. Check **Persistence Requests Total** — if persistence QPS is near the `history.persistenceMaxQPS` limit, requests are queuing and triggering the overload signal
4. Check DB host metrics (CPU, IOPS, connections) — the root cause is typically at the DB layer, not the Temporal service

### Relevant dynamic config

- `history.persistenceMaxQPS` — max persistence QPS per history host (default 9,000); primary lever for controlling DB load
- `history.persistencePerShardNamespaceMaxQPS` — per-shard per-namespace cap (default 0, disabled); useful for isolating a noisy namespace driving the overload
- `frontend.persistenceMaxQPS` — frontend-side persistence QPS limit (default 2,000)
- `matching.persistenceMaxQPS` — matching-side persistence QPS limit (default 3,000)
