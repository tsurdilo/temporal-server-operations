## Service Panic Detected

**Severity:** Critical
**Component:** server
**Dashboard panel:** [Service Panics](https://github.com/tsurdilo/temporal-metrics/blob/main/observability/dashboards/server/temporal-server-readme.md) — panel ID 99

### What this alert detects

Any non-zero rate of `service_panics` across any Temporal service for 1 consecutive minute. A single panic is enough to fire this alert.

### Why it matters

A Go panic is an unrecoverable runtime error. When a Temporal service panics, the affected goroutine crashes. Depending on where the panic occurs this can cause shard loss, incomplete workflow state transitions, or task corruption. Any panic in production is a signal that something fundamentally unexpected happened and warrants immediate investigation.

### Triage steps

1. Open the **Service Panics** panel (panel 99) in the Temporal Server dashboard — identify which service panicked and when
2. Check service logs for the panicking pod — the panic stack trace will be logged and is the primary diagnostic artifact
3. Check if the pod restarted — if the process crashed, alert 34 (Unexpected Shard Movement) may also fire as shards move to surviving pods
4. Check if any workflows are stuck — a panic during a state transition can leave workflow execution in an inconsistent state

### Relevant dynamic config

None — panics are code-level failures, not configuration issues.
