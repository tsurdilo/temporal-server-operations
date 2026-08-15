## All Pollers Disconnected

**Severity:** Critical
**Component:** frontend
**Dashboard panel:** [Total Concurrent Pollers](https://github.com/tsurdilo/temporal-metrics/blob/main/observability/dashboards/server/temporal-server-readme.md) — panel ID 162

### What this alert detects

`service_pending_requests` for `PollWorkflowTaskQueue` and `PollActivityTaskQueue` operations drops to zero for a specific namespace for 1 consecutive minute. This measures concurrent long-poll requests from workers — zero means no workers are actively polling for workflow or activity tasks in that namespace.

### Why it matters

With no pollers connected, workflow tasks and activity tasks queue up but nothing picks them up. Workflows stall at every task boundary — no progress until workers reconnect. All queued tasks are persisted to the database by the matching service, adding DB pressure while pollers are absent.

### Triage steps

1. Open the **Total Concurrent Pollers** panel (panel 162) in the Temporal Server dashboard — confirm which namespace dropped and when
2. Check if alert 1 or alert 4 are also firing — if frontend RPS is also zero the issue is cluster-wide, not worker-side
3. Check if workers for that namespace are running — look for worker process crashes, OOM kills, or deployment events
4. Check worker-side SDK metrics if available — worker poll errors or connection failures will appear there before the server sees zero pollers

### Relevant dynamic config

- `frontend.namespaceCount` — concurrent poller limit per namespace per frontend host (default 1,200); if set too low workers are throttled on poll requests with `RESOURCE_EXHAUSTED: namespace count limit exceeded`
- `frontend.globalNamespaceCount` — global concurrent long-running requests limit per namespace across all frontend instances (default 0, disabled)
