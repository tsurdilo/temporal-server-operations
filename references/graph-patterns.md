# Temporal Metrics — Graph Pattern Reference

This is a living playbook for interpreting common visual patterns in `temporal_request_failure` and `temporal_long_request_failure` RPS graphs. Each entry maps a visual shape to its likely causes, business impact, and how to distinguish between root causes using correlated signals.

For background on the metrics, operations, and rate limit buckets referenced here, see the main [SDK Request Failures guide](./sdk-request-failures.md).

---

## Table of Contents

1. [PollWorkflowTaskQueue — Sustained ResourceExhausted Saturation with Periodic Drops](#pollworkflowtaskqueue--sustained-resourceexhausted-saturation-with-periodic-drops)
2. [StartWorkflowExecution — Recurring ResourceExhausted with Intermittent Spikes](#startworkflowexecution--recurring-resourceexhausted-with-intermittent-spikes)

---

## PollWorkflowTaskQueue — Sustained ResourceExhausted Saturation with Periodic Drops

**What it looks like:**
One series (most commonly `PollWorkflowTaskQueue`) climbs to a very high RPS of `ResourceExhausted` responses — often in the tens of thousands — and stays there as a sustained plateau. Periodically, the line drops sharply to near-zero before spiking back up to the same ceiling. Other series (completions, starts, signals) remain near-zero or at low rates throughout.

**What it means:**
The namespace execution bucket RPS quota is saturated. Because `PollWorkflowTaskQueue` is Priority 4 — the lowest priority in the Execution bucket — it is the first operation to be shed under load. The server is correctly protecting higher-priority operations (workflow completions, starts, signals) while throttling polls. The periodic drops are **not necessarily worker restarts** — see distinguishing causes below.

**Impact:**
This pattern is severely disruptive to workflow workers and should be treated as high priority:

* **Schedule-to-start latency increases** for workflow tasks — workers cannot pick up tasks fast enough because their polls are being rejected
* **Sync match rate drops** — tasks cannot be handed off directly to a waiting poller within the sync match window (~500ms) and are instead written to the database, adding latency to every workflow task dispatch
* **Event delivery delays** — signals, activity completions, and timers firing all experience delays getting delivered to workers, since workers are not polling successfully
* **Prolonged end-to-end workflow execution** — the compounding effect of all of the above means workflows take significantly longer to complete than they should
* **Potential significant business impact** — depending on what workflows power in your application, this can directly affect SLAs, user-facing latency, or critical business processes

**Possible causes of the periodic sharp drops:**

| Cause | Description |
| --- | --- |
| **Workload-driven (most common)** | A large batch of workflows was executing activities or local activities — during which workers do not need to poll for workflow tasks. Poll pressure drops naturally. When those activities finish and workflows need to progress, workers flood back to polling and immediately re-hit the quota ceiling. |
| **Workflow executions completing** | A large number of executions completed, temporarily reducing the number of workers actively polling. Once new work arrives, polling resumes at full pressure. |
| **Worker restarts or scaling events** | Workers restarted or new instances came online, briefly reducing in-flight poll connections before resuming at the same rate. |

**How to distinguish between causes:**

| Signal to check | What it tells you |
| --- | --- |
| `RespondActivityTaskCompleted` or `RespondActivityTaskFailed` elevated during the drop | Workload-driven — workflows were running activities during the quiet period. This is the most common cause of the sawtooth shape. |
| `ShutdownWorker`, `DescribeNamespace`, or `GetSystemInfo` spikes around the same time as the drop | Worker restarts — these operations are emitted by the SDK during graceful worker shutdown and startup. If you see them correlating with the drops, workers were cycling. |
| No correlated signals in either of the above | Likely a batch of workflow executions completing. Check workflow completion metrics or visibility queries for the namespace around the drop times. |

**Next steps:**

1. Check the **Resource Exhausted with Cause** panel on your server dashboard — look for `RpsLimit`, `ConcurrentLimit`, or `SystemOverload` as the cause tag on `PollWorkflowTaskQueue`
2. Review `frontend.namespaceRPS` and `frontend.globalNamespaceRPS` for this namespace — they likely need to be increased
3. If the cause is `ConcurrentLimit`, review `frontend.namespaceCount` — you may have too many worker instances each holding open long-poll connections simultaneously
4. Check **Schedule to Start Latencies** on your server dashboard (`task_schedule_to_start_latency_bucket`) — if this is elevated, workers are already falling behind and the business impact is active
5. Check **Sync Match Rate** and **Approximate Task Backlog** — a growing backlog alongside this pattern confirms workflows are accumulating unprocessed tasks

---

## StartWorkflowExecution — Recurring ResourceExhausted with Intermittent Spikes

**What it looks like:**
`StartWorkflowExecution` shows a non-zero, recurring baseline of `ResourceExhausted` errors — typically in the range of 0.01–0.03 rps — punctuated by sharper spikes (e.g. 0.10 rps or higher). The pattern is not a single isolated event but a repeating signal across days. Other operations (`RespondActivityTaskCompleted`, poll operations) remain near-zero or healthy throughout.

**What it means:**
The namespace execution bucket RPS quota is being hit on workflow starts. `StartWorkflowExecution` is Priority 1 in the Execution bucket — a high-priority operation — so seeing sustained throttling here means the namespace is under genuine load pressure, not just shedding low-priority polls. The SDK retries `ResourceExhausted` automatically with backoff, so not every error in the graph necessarily maps to a dropped execution. However, the combination of a recurring baseline *and* periodic spikes means the system is regularly approaching or exceeding the SDK's ~60s retry window.

**Business Impact:**
This pattern can have direct business impact and should not be treated as informational:

* **Executions silently dropped** — if throttling persists beyond the SDK's ~60s retry window, `StartWorkflowExecution` calls will ultimately fail and return the error to the caller. Any workflow that was never created is invisible to Temporal — it will not appear in visibility, will not be retried by the server, and will not generate any timeout or failure event. The only record of it is in your client-side logs, *if* your client code captures the failure.
* **Lost work with no automatic recovery** — unlike activity failures or WFT timeouts, a failed `StartWorkflowExecution` has no built-in retry at the orchestration level. If the caller does not handle the error and re-enqueue the start, the work is permanently lost.
* **Downstream SLA impact** — depending on what these workflows power (order processing, notifications, fulfillment, etc.), dropped starts translate directly to missed business operations. The impact compounds if the caller is fire-and-forget and does not check for failures.
* **Spike severity matters** — a spike to 0.10 rps on `StartWorkflowExecution` throttling, even if brief, represents a burst of start attempts that the namespace could not absorb. If your client is batching or bursting workflow starts, the spike is likely the burst hitting the quota ceiling.

**Possible causes:**

| Cause | Description |
| --- | --- |
| **Namespace RPS quota too low** | `frontend.namespaceRPS` or `frontend.globalNamespaceRPS` is set below the actual peak start rate for this namespace. The most common cause. |
| **Burst traffic pattern** | A scheduled job, upstream event fan-out, or batch process is starting many workflows in a short window, exceeding the burst capacity (`frontend.namespaceBurstRatio`, default 2x). |
| **Oversized payload** | If the workflow input exceeds the 4 MB gRPC message size limit, the server returns `ResourceExhausted` for that specific request — *not* a throttling issue. The SDK will retry it pointlessly. If RPS metrics look healthy but `ResourceExhausted` persists, check payload sizes. |
| **`BusyWorkflow` throttle** | If `history.enableWorkflowIdReuseStartTimeValidation` is enabled, the server returns `ResourceExhausted` when the same workflow ID is being re-started very rapidly. Check the Resource Exhausted with Cause panel for `BusyWorkflow` as the cause. |

**How to distinguish between causes:**

| Signal to check | What it tells you |
| --- | --- |
| Resource Exhausted with Cause panel → `RpsLimit` | Namespace RPS quota is the bottleneck. Increase `frontend.namespaceRPS` or `frontend.globalNamespaceRPS`. |
| Resource Exhausted with Cause panel → `BusyWorkflow` | Same workflow ID is being started too rapidly. Review workflow ID generation logic or reduce re-start frequency. |
| `ResourceExhausted` on `StartWorkflowExecution` but overall RPS metrics look low | Likely a payload size issue, not throttling. Check the size of workflow input payloads being passed at start time. |
| Spikes correlate with a known scheduled job or batch process | Burst traffic pattern. Consider spreading starts over time or increasing the burst ratio. |

**Next steps:**

1. Check the **Resource Exhausted with Cause** panel on your server dashboard — filter by `StartWorkflowExecution` and look for `RpsLimit`, `SystemOverload`, or `BusyWorkflow` as the cause tag
2. Review `frontend.namespaceRPS` and `frontend.globalNamespaceRPS` for this namespace and compare against actual peak start RPS
3. **Audit your client code** — confirm that `StartWorkflowExecution` failures after SDK retries are exhausted are being caught, logged with workflow ID and input payload, and routed to a dead-letter queue or backfill process. If this is not in place, you may already have lost executions with no record of them
4. If the pattern includes sharp spikes, check whether a scheduled job or upstream fan-out is batching workflow starts — consider spreading them with a rate limiter on the client side
5. Check payload sizes on `StartWorkflowExecution` calls if the throttling pattern does not correlate with RPS metrics