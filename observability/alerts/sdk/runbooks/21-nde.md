# WFT Execution Failed: NonDeterminismError

**Severity:** Critical
**Component:** wft
**Metric:** `temporal_workflow_task_execution_failed` — `failure_reason=NonDeterminismError`
**Dashboard panel:** [WFT Execution Failed](../../../dashboards/sdk/temporal-sdk-go-readme.md) — panel 47
**Full alert definition:** [Alert #21 — alerts-index.md](../alerts-index.md#alert-21--wft-execution-failed-nondeterminismerror)

---

## Table of Contents

- [What this alert detects](#what-this-alert-detects)
- [Why it matters](#why-it-matters)
- [Triage steps](#triage-steps)
- [Related alerts](#related-alerts)

---

## What this alert detects

A workflow's replay produced a different command sequence than what is recorded in history — a NonDeterminismError (NDE). The worker detected that the workflow code it is running does not match the history of commands the workflow has already produced.

## Why it matters

Workflow executions affected by NDE are not progressing. The server continuously retries the workflow task, putting additional pressure on workflow workers. Affected executions remain in running status by default, prolonging their end-to-end execution time indefinitely. NDE will not resolve on its own until the root cause is addressed.

## Triage steps

**1. Identify the affected workflow IDs.**
This metric does not carry `workflow_id` — check worker logs, which will log the NDE with the associated workflow ID and run ID. In the Temporal UI, affected executions can also be found by querying the `TemporalReportedProblems` search attribute, which the server sets on executions experiencing repeated workflow task failures.

**2. Understand the error.**
Look at the `WorkflowTaskFailed` event in the history of an affected execution — the event contains the error message that indicates exactly where in the replay the mismatch occurred and what command was expected vs. what the code produced. This is the most direct signal for root cause analysis.

**3. Determine whether this is a code change or a deployment issue.**
Common causes:
- A code change added, removed, or reordered workflow commands (activity schedules, timers, signals, child workflows) without a versioning guard (`workflow.GetVersion` in Go, `Workflow.getVersion` in Java) — in-flight executions that already produced history under the old code will NDE on the new code
- A rolling restart where old and new worker versions are briefly running simultaneously — some executions may NDE transiently while the old version is still partially deployed; these typically resolve once the rollout completes
- Changing activity or timer parameters in existing workflow code without versioning

**4. Roll back if needed.**
If NDE started after a deployment and is not resolving on its own (i.e. not a transient rolling restart issue), roll back the worker to the previous version — affected executions will resume on their next WFT retry once the compatible code is running. Then fix the incompatible change by introducing a proper versioning guard before redeploying.

**5. Monitor worker pressure.**
While NDE is active, the server keeps retrying the workflow task — this puts sustained pressure on workflow workers. Cross-check alert [#14 — Worker Task Slots Exhausted](./14-worker-slots-exhausted.md) for `worker_type=WorkflowWorker` and alert [#32 — WFT Execution Latency Critical](./32-wft-execution-latency-critical.md) — a high volume of NDE retries can saturate worker capacity and affect healthy executions on the same task queue.

## Related alerts

- [#14 — Worker Task Slots Exhausted](./14-worker-slots-exhausted.md)
- [#15 — All Pollers Disconnected](./15-all-pollers-disconnected.md)
- [#22 — WFT Execution Failed: GrpcMessageTooLarge](./22-grpc-message-too-large.md)
- [#32 — WFT Execution Latency Critical](./32-wft-execution-latency-critical.md)
