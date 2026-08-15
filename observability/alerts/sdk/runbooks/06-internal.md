# Request Failure: INTERNAL

**Severity:** Critical
**Component:** request
**Metric:** `temporal_request_failure` — `status_code=INTERNAL`, any operation
**Dashboard panel:** [Request Failures](../../../dashboards/sdk/temporal-sdk-go-readme.md) — panel 5
**Full alert definition:** [Alert #6 — alerts-index.md](../alerts-index.md#alert-6--request-failure-internal)

---

## Table of Contents

- [What this alert detects](#what-this-alert-detects)
- [Why it matters](#why-it-matters)
- [Triage steps](#triage-steps)
- [Related alerts](#related-alerts)

---

## What this alert detects

The server is returning `INTERNAL` errors for SDK operations. Short bursts are expected during server restarts and rolling deploys — adjust the `for` duration if your deployment process regularly triggers this alert. Sustained INTERNAL errors indicate infrastructure-level issues on the server side.

## Why it matters

INTERNAL errors are retried by the SDK, but sustained errors will eventually exhaust the retry budget and surface as failures to the caller. Workers receiving INTERNAL errors on poll operations will back off and poll less frequently, increasing schedule-to-start latency. INTERNAL errors on respond operations can cascade into task timeouts — cross-check alerts #1a and #1b.

## Triage steps

**1. Check the Service Panics panel.**
Check the [**Service Panics** panel (§5 — Service Requests and Errors)](../../../dashboards/server/temporal-server-readme.md) on the server dashboard — any panic is a critical signal and is almost always the root cause of sustained INTERNAL errors. Investigate the panic message immediately.

**2. Check the Service Errors panel.**
Check the [**Service Errors by Namespace** panel (§5)](../../../dashboards/server/temporal-server-readme.md) — confirm the scope of errors. If errors are isolated to a single namespace, the problem may be namespace-specific. If errors are cluster-wide, the problem is infrastructure-level.

**3. Check persistence errors and availability.**
Check the [**Persistence Errors by Namespace and Operation** panel (§3)](../../../dashboards/server/temporal-server-readme.md) and the [**Persistence Availability** panel (§3 — Persistence Requests, Latencies and Errors)](../../../dashboards/server/temporal-server-readme.md#3-persistence-requests-latencies-and-errors) — sustained persistence errors are a common root cause of INTERNAL responses. If the database is returning errors, the server wraps them as INTERNAL and returns them to the caller.

**4. Check server health and deployment.**
Verify all server pods are healthy and running the correct binary. Check pod logs for stack traces or error messages. If a recent deployment was made, consider rolling back.

## Related alerts

- [#1a — NOT_FOUND on WFT Respond Operations](./01a-not-found-wft-respond.md)
- [#1b — NOT_FOUND on Activity Respond Operations](./01b-not-found-activity-respond.md)
- [#5 — Request Failure: UNIMPLEMENTED](./05-unimplemented.md)
- [#15 — All Pollers Disconnected](./15-all-pollers-disconnected.md)
