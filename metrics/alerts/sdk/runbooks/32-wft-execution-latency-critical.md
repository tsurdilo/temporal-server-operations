# WFT Execution Latency Critical

**Severity:** Critical
**Component:** wft
**Metric:** `temporal_workflow_task_execution_latency` — p99 > 10s
**Dashboard panel:** [WFT Execution Latency](../../../dashboards/sdk/temporal-sdk-go-readme.md) — panel 44
**Full alert definition:** [Alert #32 — alerts-index.md](../alerts-index.md#alert-32--wft-execution-latency-critical)

---

## Table of Contents

- [What this alert detects](#what-this-alert-detects)
- [Why it matters](#why-it-matters)
- [Triage steps](#triage-steps)
- [Related alerts](#related-alerts)

---

## What this alert detects

p99 workflow task execution latency has been above 10 seconds for 5 minutes on this namespace and task queue. The default WFT timeout is 10 seconds (`history.defaultWorkflowTaskTimeout`) — at this latency level the server is actively timing out workflow tasks.

## Why it matters

The server writes `WorkflowTaskTimedOut` events to history and reschedules timed-out tasks on the normal task queue. Each timeout forces a sticky cache eviction on the worker that held the execution — the next WFT for that execution requires a full cold replay. If you are running local activities, a WFT timeout causes local activities to be re-executed from scratch on the retried task since their results are not checkpointed between WFT heartbeats. If local activities are not idempotent, re-execution can cause duplicate side effects with real business impact. At scale, a high volume of WFT timeouts creates a compounding problem — more cold replays drive higher WFT execution latency, causing more timeouts.

## Triage steps

**1. Check WFT replay latency.**
Check `temporal_workflow_task_replay_latency` on the SDK dashboard — if replay latency is high, the time is being spent re-executing history rather than running new commands. High replay latency is typically caused by large history size, slow data converter execution during replay, or high sticky cache eviction rate forcing frequent cold replays. Cross-check alert [#35 — WFT Replay Latency High](../alerts-index.md#alert-35--wft-replay-latency-high) and alert [#34 — Sticky Cache Disabled](./34-sticky-cache-disabled.md).

**2. Check sticky cache eviction rate.**
Cross-check alert [#16 — Sticky Cache Forced Evictions High](../alerts-index.md#alert-16--sticky-cache-forced-evictions-high) — a high eviction rate forces cold replays on every WFT, directly driving up execution latency. This can be a compounding loop: timeouts cause evictions, evictions cause slow replays, slow replays cause more timeouts.

**3. Check worker pod CPU.**
If replay latency is normal but WFT execution latency is high, the time is being spent in new command execution — check worker pod CPU utilization. High CPU slows all code execution on the worker including WFT processing. Also check for blocking calls or heavy computation inside workflow functions — workflow code must be deterministic and fast.

**4. Check for blocking workflow code.**
Workflow code must not perform blocking I/O, heavy computation, or call non-Temporal APIs synchronously. Any blocking call inside a workflow function holds the WFT slot and inflates execution latency. For Python SDK workers, verify that no `async def` workflow code is blocking the event loop.

**5. Check for RESOURCE_EXHAUSTED on respond operations.**
Cross-check alert [#2b — RESOURCE_EXHAUSTED on Respond Operations](./02b-resource-exhausted-respond-ops.md) — if the server is throttling `RespondWorkflowTaskCompleted`, the SDK holds the slot until the respond call succeeds, inflating WFT execution latency even when the workflow code itself completed quickly.

## Related alerts

- [#2b — RESOURCE_EXHAUSTED on Respond Operations](./02b-resource-exhausted-respond-ops.md)
- [#14 — Worker Task Slots Exhausted](./14-worker-slots-exhausted.md)
- [#16 — Sticky Cache Forced Evictions High](../alerts-index.md#alert-16--sticky-cache-forced-evictions-high)
- [#21 — WFT Execution Failed: NonDeterminismError](./21-nde.md)
- [#34 — Sticky Cache Disabled](./34-sticky-cache-disabled.md)
- [#35 — WFT Replay Latency High](../alerts-index.md#alert-35--wft-replay-latency-high)
