# RESOURCE_EXHAUSTED on Critical User-Facing Operations

**Severity:** Critical
**Component:** request
**Metric:** `temporal_request_failure` — `status_code=RESOURCE_EXHAUSTED`, operations `StartWorkflowExecution` / `SignalWithStartWorkflowExecution` / `ExecuteMultiOperation` / `UpdateWorkflowExecution` / `SignalWorkflowExecution`
**Dashboard panel:** [Request Failures](../../../dashboards/sdk/temporal-sdk-go-readme.md) — panel 5
**Full alert definition:** [Alert #2a — alerts-index.md](../alerts-index.md#alert-2a--request-failure-resource_exhausted-on-critical-user-facing-operations)

---

## Table of Contents

- [What this alert detects](#what-this-alert-detects)
- [Why it matters](#why-it-matters)
- [Triage steps](#triage-steps)
- [Related alerts](#related-alerts)

---

## What this alert detects

The server is returning `RESOURCE_EXHAUSTED` on critical user-facing SDK operations — starts, signals, updates, and `UpdateWithStart`. The SDK retries these automatically for up to 60 seconds. If throttling is sustained beyond that, the call fails and the error propagates to the caller.

Note: `UpdateWithStartWorkflowExecution` maps to the gRPC operation name `ExecuteMultiOperation` in SDK metrics.

## Why it matters

These operations are on the critical path for your application — they are how your code starts workflows and delivers signals and updates to running executions. Even within the 60-second retry window, callers experience elevated latency. Beyond it, calls fail outright and your application must handle the error — if it does not, starts, signals, and updates are silently dropped. Dropped starts mean workflows never run. Dropped signals or updates mean running workflows never receive the input they are waiting for, potentially stalling them indefinitely. Log these failures in your application code so you can backfill starts and signals if needed.

From the server's perspective these operations are throttled last — if you are seeing RESOURCE_EXHAUSTED on these operations it is a strong signal that throttling is severe and widespread across the cluster.

## Triage steps

**1. Check the server Resource Exhausted panel.**
Check the [**Resource Exhausted with Cause** panel (§6 — Throttling and Limits)](../../../dashboards/server/temporal-server-readme.md) on the server dashboard — this shows the throttle cause (`RpsLimit`, `ConcurrentLimit`, `SystemOverloaded`, `CircuitBreakerOpen`) and helps identify whether this is a namespace RPS limit, a system-wide overload, or a circuit breaker firing.

**2. Check persistence latency.**
Check the [**Persistence Latencies** panel (§3 — Persistence Requests, Latencies and Errors)](../../../dashboards/server/temporal-server-readme.md#3-persistence-requests-latencies-and-errors) filtered to `CreateWorkflowExecution` and `UpdateWorkflowExecution` — high persistence latency is the most common root cause of cascading throttling on these operations. If the database is slow, the frontend service backs up and starts throttling callers.

**3. Check namespace RPS limits.**
If the cause is `RpsLimit`, check whether `frontend.namespaceRPS` is set too low for your current traffic volume. Consider increasing it via dynamic config — but only after confirming persistence latency is healthy, otherwise increasing RPS into an already overloaded system will make things worse.

**4. Check total cluster RPS.**
Check the [**Total RPS** panel (§1 — Cluster Throughput)](../../../dashboards/server/temporal-server-readme.md) — if total cluster RPS is at or near capacity, the problem is cluster-level saturation, not namespace limits. Consider scaling frontend pods or the underlying database.

**5. Check for system overload.**
If the cause is `SystemOverloaded` or `CircuitBreakerOpen`, cross-check server alert [#30 — System Overload Throttling](../../server/runbooks/30-system-overload-throttling.md) — this indicates the server itself is under severe pressure and is shedding load to protect itself.

## Related alerts

- [#2b — RESOURCE_EXHAUSTED on Respond Operations](./02b-resource-exhausted-respond-ops.md)
- [#6 — Request Failure: INTERNAL](./06-internal.md)
- [#12 — RespondWorkflowTaskCompleted Rate Dropped to Zero](./12-wft-completions-zero.md)
- [#31 — Request Latency High on Critical User-Facing Operations](./31-request-latency-critical-ops.md)
