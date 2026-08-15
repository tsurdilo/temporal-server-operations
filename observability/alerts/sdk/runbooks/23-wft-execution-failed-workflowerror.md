# WFT Execution Failed: WorkflowError

**Severity:** Warning
**Component:** wft
**Metric:** `temporal_workflow_task_execution_failed` — `failure_reason=WorkflowError`
**Dashboard panel:** [WFT Execution Failed](../../../dashboards/sdk/temporal-sdk-go-readme.md) — panel 47
**Full alert definition:** [Alert #23 — alerts-index.md](../alerts-index.md#alert-23--wft-execution-failed-workflowerror)

---

## Table of Contents

- [What this alert detects](#what-this-alert-detects)
- [Why it matters](#why-it-matters)
- [Triage steps](#triage-steps)
- [Related alerts](#related-alerts)

---

## What this alert detects

Workflow task failures with `failure_reason=WorkflowError` at a sustained rate above 20/s for 2 minutes. This covers unhandled exceptions and panics in workflow code that the SDK catches and reports to the server as a workflow task failure — including thread pool exhaustion (`RejectedExecutionException`), unhandled exceptions thrown inside the workflow function, and data converter errors.

## Why it matters

When a WFT fails with WorkflowError, the server retries the workflow task. If the error is deterministic — reproducible on every replay — the execution is stuck retrying indefinitely, consuming worker capacity and holding the execution in a permanently unhealthy state. At high rates, the retry pressure can saturate workflow worker slots and affect healthy executions on the same task queue. Unlike NonDeterminismError or GrpcMessageTooLarge, the server does not terminate the execution — it keeps retrying, which means the impact compounds over time if left unresolved.

## Triage steps

**1. Identify the affected workflow IDs.**
This metric does not carry `workflow_id` — check worker logs for the associated workflow ID, run ID, and the full exception stack trace. The `WorkflowTaskFailed` event in the affected execution's history will also contain the error message and type.

**2. Determine the root cause.**
`WorkflowError` covers several distinct failure modes:

- **Thread pool exhaustion (Java SDK only)** — `RejectedExecutionException` from the workflow thread pool being saturated. This happens when `setMaxWorkflowThreadCount` on `WorkerFactoryOptions` is set too low relative to the number of concurrent workflow executions — the pool runs out of threads and new WFTs are rejected before they can execute. Increase `maxWorkflowThreadCount` or reduce concurrent workflow load. Cross-check alert [#14 — Worker Task Slots Exhausted](./14-worker-slots-exhausted.md) — slot exhaustion and thread pool exhaustion often occur together under high load.

- **Unhandled exception in workflow code** — a bug or unexpected condition in the workflow function throws an exception that is not handled. The server retries the WFT — if the exception is reproducible on every replay, the execution is stuck. Look at the `WorkflowTaskFailed` event in the affected execution's history for the error message and type.

- **Data converter error** — failure to serialize or deserialize workflow inputs, outputs, or memo fields. Check data converter configuration and payload codec. This often appears as a `DataConverterException` or similar in worker logs.

**3. Check worker thread and slot pressure.**
Cross-check alert [#14 — Worker Task Slots Exhausted](./14-worker-slots-exhausted.md) for `worker_type=WorkflowWorker`. Also check worker pod CPU and memory utilization via your infrastructure observability stack — a CPU-starved worker is slower to complete WFTs, which can accelerate thread pool exhaustion under load.

**4. Fix and redeploy.**
- For thread pool exhaustion (Java SDK only): increase `maxWorkflowThreadCount` on `WorkerFactoryOptions` and redeploy. Consider also whether the workflow load exceeds what this worker pool is designed to handle — scaling worker pods horizontally may be needed alongside the thread count increase.
- For code bugs: fix the exception path and redeploy. Affected executions will resume on their next WFT retry automatically once compatible code is running.
- For data converter errors: fix the converter configuration and redeploy.

## Related alerts

- [#14 — Worker Task Slots Exhausted](./14-worker-slots-exhausted.md)
- [#21 — WFT Execution Failed: NonDeterminismError](./21-nde.md)
- [#22 — WFT Execution Failed: GrpcMessageTooLarge](./22-grpc-message-too-large.md)
- [#32 — WFT Execution Latency Critical](./32-wft-execution-latency-critical.md)
