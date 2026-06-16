# Request Latency High on Critical User-Facing Operations

**Severity:** Critical
**Component:** request
**Metric:** `temporal_request_latency` — p99 > 2s, operations `StartWorkflowExecution` / `SignalWithStartWorkflowExecution` / `ExecuteMultiOperation` / `SignalWorkflowExecution`
**Dashboard panel:** [Request Latency](../../../dashboards/sdk/temporal-sdk-go-readme.md) — panel 7
**Full alert definition:** [Alert #31 — alerts-index.md](../alerts-index.md#alert-31--request-latency-high-on-critical-user-facing-operations)

---

## Table of Contents

- [What this alert detects](#what-this-alert-detects)
- [Why it matters](#why-it-matters)
- [Triage steps](#triage-steps)
- [Related alerts](#related-alerts)

---

## What this alert detects

p99 latency for critical user-facing SDK operations has been above 2 seconds for 5 minutes on this namespace. Affected operations: `StartWorkflowExecution`, `SignalWithStartWorkflowExecution`, `SignalWorkflowExecution`, and `ExecuteMultiOperation` (`UpdateWithStartWorkflowExecution`).

## Why it matters

These operations are synchronous from the caller's perspective — elevated latency means your application code is blocked waiting on the server response, directly impacting your users and any workflows that depend on these calls completing. The SDK retries on transient errors but does not hide latency — every retry adds to the total wait time observed on this metric. If throttling is the cause and retries exhaust the 60-second budget, calls fail outright.

## Triage steps

**1. Check frontend service latency.**
Check the [**Frontend Service Latency** panel (§4 — Service Latencies)](../../../dashboards/server/temporal-server-readme.md) filtered to the affected operations — if server-side latency is elevated, that is the root cause. The SDK-side metric includes serialization and network time, so server-side latency is the more precise signal.

**2. Check persistence latency.**
Check the [**Persistence Latencies** panel (§3 — Persistence Requests, Latencies and Errors)](../../../dashboards/server/temporal-server-readme.md#3-persistence-requests-latencies-and-errors) filtered to `CreateWorkflowExecution` and `UpdateWorkflowExecution` — high persistence latency on these operations is the most common driver of elevated frontend latency for starts and signals. If the database is slow, the frontend service backs up and latency rises for all callers.

**3. Check for RESOURCE_EXHAUSTED on critical operations.**
Cross-check alert [#2a — RESOURCE_EXHAUSTED on Critical User-Facing Operations](./02a-resource-exhausted-critical-ops.md) — if the server is throttling these operations, the SDK retries add to the total observed latency on this metric. If both alerts are firing, RESOURCE_EXHAUSTED is the root cause of the latency.

**4. Check frontend pod CPU.**
If persistence latency looks normal and there is no throttling, check frontend pod CPU utilization via your infrastructure observability stack — a CPU-saturated frontend pod processes requests slower, directly increasing response latency.

## Related alerts

- [#2a — RESOURCE_EXHAUSTED on Critical User-Facing Operations](./02a-resource-exhausted-critical-ops.md)
- [#6 — Request Failure: INTERNAL](./06-internal.md)
- [#32 — WFT Execution Latency Critical](./32-wft-execution-latency-critical.md)
