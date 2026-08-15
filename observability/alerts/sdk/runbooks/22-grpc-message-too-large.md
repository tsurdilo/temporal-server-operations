# WFT Execution Failed: GrpcMessageTooLarge

**Severity:** Critical
**Component:** wft
**Metric:** `temporal_workflow_task_execution_failed` — `failure_reason=GrpcMessageTooLarge`
**Dashboard panel:** [WFT Execution Failed](../../../dashboards/sdk/temporal-sdk-go-readme.md) — panel 47
**Full alert definition:** [Alert #22 — alerts-index.md](../alerts-index.md#alert-22--wft-execution-failed-grpcmessagetoolarge)

---

## Table of Contents

- [What this alert detects](#what-this-alert-detects)
- [Why it matters](#why-it-matters)
- [Triage steps](#triage-steps)
- [Related alerts](#related-alerts)

---

## What this alert detects

The WFT response payload exceeded the gRPC message size limit. The SDK worker attempted to send `RespondWorkflowTaskCompleted` but the response was rejected as too large — either by the gRPC library on the SDK side, by a proxy or load balancer in the path (e.g. Envoy), or by the gRPC library on the server side on receive. Because the Temporal server history service never saw the original request, the SDK explicitly sends a follow-up `RespondWorkflowTaskFailed` with cause `WORKFLOW_TASK_FAILED_CAUSE_GRPC_MESSAGE_TOO_LARGE`. The server then terminates the workflow execution in response — ending it with `TERMINATED` status with no retry.

## Why it matters

Affected workflow executions are terminated immediately and permanently — they cannot be retried automatically. Any in-progress work in those executions is lost. Terminated executions must be restarted manually if needed. At scale, cross-check the [**Workflow Terminate** panel](../../../dashboards/server/temporal-server-readme.md) on the server dashboard — it tracks the rate of terminated executions and will show a spike when this is happening at volume.

## Triage steps

**1. Identify the affected workflow IDs.**
This metric does not carry `workflow_id` — check worker logs for the affected workflow IDs and run IDs. Look at the `WorkflowTaskFailed` event and the `WorkflowExecutionTerminated` event in the history of affected executions to confirm the cause.

**2. Investigate which part of the WFT response is oversized.**
The fix depends entirely on what is making the response too large. Common causes:
- **Oversized activity inputs or outputs** — activity arguments or return values that are too large. Move large payloads out of band (e.g. store in blob storage and pass a reference) instead of passing them directly through Temporal history.
- **Excessive signal or update accumulation in history** — a large number of signals or updates buffered in a single WFT. Consider rate-limiting signal senders or batching signals.
- **Too many commands batched in a single WFT response** — a workflow scheduling a very large number of activities or child workflows in a single execution step. Break large fan-outs into smaller batches across multiple WFTs.

**3. Check for terminated executions on the server.**
Check the [**Workflow Terminate** panel](../../../dashboards/server/temporal-server-readme.md) on the server dashboard — a spike in terminated executions alongside this alert confirms the server is terminating executions in response to this failure.

**4. Fix the root cause and redeploy first.**
Terminated executions do not retry automatically. Before restarting any executions, fix the underlying cause — oversized payloads, excessive signal accumulation, or large fan-outs — and deploy the corrected worker. Restarting executions without fixing the code will cause them to hit the same limit and be terminated again. Once the fix is deployed and verified, restart affected executions manually via the Temporal UI or CLI if they need to continue.

## Related alerts

- [#21 — WFT Execution Failed: NonDeterminismError](./21-nde.md)
- [#32 — WFT Execution Latency Critical](./32-wft-execution-latency-critical.md)
