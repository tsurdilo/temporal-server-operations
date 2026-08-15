# Sticky Cache Disabled

**Severity:** Warning
**Component:** cache
**Metric:** `temporal_sticky_cache_size` (gauge) — value == 0
**Dashboard panel:** [Sticky Cache Size](../../../dashboards/sdk/temporal-sdk-go-readme.md) — panel 54
**Full alert definition:** [Alert #34 — alerts-index.md](../alerts-index.md#alert-34--sticky-cache-disabled)

---

## Table of Contents

- [What this alert detects](#what-this-alert-detects)
- [Why it matters](#why-it-matters)
- [Triage steps](#triage-steps)
- [Related alerts](#related-alerts)

---

## What this alert detects

The sticky cache size gauge has been zero for 5 minutes — the worker's sticky execution cache is disabled. No workflow executions are being held in memory between workflow tasks. Every WFT requires a full cold replay from the beginning of history.

## Why it matters

With the sticky cache disabled, every workflow task for every execution on this worker requires fetching all history pages from the server and re-executing every command from scratch. At any meaningful scale this causes significant and sustained pressure on the frontend and persistence layers — every WFT is equivalent to a cache miss. WFT execution latency will be elevated for all executions on this worker. Cross-check alert [#32 — WFT Execution Latency Critical](./32-wft-execution-latency-critical.md) and the [**Persistence Latencies** panel (§3 — Persistence Requests, Latencies and Errors)](../../../dashboards/server/temporal-server-readme.md#3-persistence-requests-latencies-and-errors).

## Triage steps

**1. Check your worker configuration.**
This is almost always a misconfiguration. Check the worker options in your application code:
- **Go SDK** — `maxWorkflowCacheSize` set to `0` disables the cache entirely. The default is 10,000. Restore a non-zero value.
- **Java SDK** — `WorkerFactoryOptions.Builder.setWorkflowCacheSize(int)` controls the cache size, default is 600. Setting it to `0` or negative resets to the default — the Java SDK does not allow disabling the cache this way. If this alert fires on a Java worker, check `WorkerFactoryOptions.Builder.setMaxWorkflowThreadCount(int)` — if the thread pool is set too low it can starve workflow execution and prevent the cache from being utilized effectively.

**2. Verify the fix takes effect.**
After correcting the worker configuration and redeploying, monitor `temporal_sticky_cache_size` on the SDK dashboard — it should climb from zero as the worker begins caching workflow executions. Also monitor `temporal_workflow_task_execution_latency` — it should decrease as cold replays are replaced by warm cache hits.

**3. Check the impact on persistence and WFT latency.**
While the cache is disabled, check the [**Persistence Latencies** panel (§3)](../../../dashboards/server/temporal-server-readme.md#3-persistence-requests-latencies-and-errors) filtered to `GetWorkflowExecution` — sustained high latency there confirms the server is under pressure from repeated history reads. Cross-check alert [#32 — WFT Execution Latency Critical](./32-wft-execution-latency-critical.md) and the `GetWorkflowExecutionHistory` long-poll rate on the SDK dashboard.

## Related alerts

- [#15 — All Pollers Disconnected](./15-all-pollers-disconnected.md)
- [#32 — WFT Execution Latency Critical](./32-wft-execution-latency-critical.md)
- [#35 — WFT Replay Latency High](../alerts-index.md#alert-35--wft-replay-latency-high)
