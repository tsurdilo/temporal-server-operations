# Request Failure: UNIMPLEMENTED

**Severity:** Critical
**Component:** request
**Metric:** `temporal_request_failure` — `status_code=UNIMPLEMENTED`, any operation
**Dashboard panel:** [Request Failures](../../../dashboards/sdk/temporal-sdk-go-readme.md) — panel 5
**Full alert definition:** [Alert #5 — alerts-index.md](../alerts-index.md#alert-5--request-failure-unimplemented)

---

## Table of Contents

- [What this alert detects](#what-this-alert-detects)
- [Why it matters](#why-it-matters)
- [Triage steps](#triage-steps)
- [Related alerts](#related-alerts)

---

## What this alert detects

The server is returning `UNIMPLEMENTED` for an SDK operation. By the time a worker reaches this point it has already successfully called `GetSystemInfo` and `DescribeNamespace` — so this is unlikely to be a simple SDK/server version mismatch on a freshly deployed worker. The server is responding to a specific operation call with UNIMPLEMENTED after the worker has already established connectivity.

## Why it matters

Workers receiving UNIMPLEMENTED on operational calls — poll, respond, heartbeat — will fail those calls and potentially stop processing tasks entirely. Unlike RESOURCE_EXHAUSTED (which the SDK retries), UNIMPLEMENTED is a non-retriable error in most SDK versions — the worker will surface this as a fatal error and may shut down.

## Triage steps

**1. Check your server deployment.**
This is the most common cause. Check whether the correct server binary is running — a wrong binary deployed to one or more pods, a corrupted deployment, or a service routing issue sending requests to the wrong handler can all produce UNIMPLEMENTED responses on valid operations. Check pod versions across all history, matching, and frontend pods.

**2. Check the Service Panics panel.**
Check the [**Service Panics** panel (§5 — Service Requests and Errors)](../../../dashboards/server/temporal-server-readme.md) on the server dashboard — a wrong binary or corrupted deployment often produces panics alongside UNIMPLEMENTED responses. Any non-zero value here is a critical signal.

**3. Check the Service Errors panel.**
Check the [**Service Errors by Namespace** panel (§5)](../../../dashboards/server/temporal-server-readme.md) — a spike in errors correlated with the UNIMPLEMENTED alert confirms the server is not processing requests normally.

**4. Check SDK version compatibility.**
If the deployment looks correct, check whether you are running a very old SDK version that is calling an API that has been removed or significantly changed on the server. Review the Temporal SDK changelog for breaking API changes between your SDK version and your server version.

## Related alerts

- [#5 — Request Failure: UNIMPLEMENTED](../alerts-index.md#alert-5--request-failure-unimplemented)
- [#6 — Request Failure: INTERNAL](./06-internal.md)
- [#15 — All Pollers Disconnected](./15-all-pollers-disconnected.md)
