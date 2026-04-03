# Temporal Metrics — Graph Pattern Reference

This is a living playbook for interpreting common visual patterns in `temporal_request_failure` and `temporal_long_request_failure` RPS graphs. Each entry maps a visual shape to its likely causes, business impact, and how to distinguish between root causes using correlated signals.

For background on the metrics, operations, and rate limit buckets referenced here, see the main [README](./README.md).

---

## Table of Contents

1. [Pattern 1: Sustained High Plateau with Periodic Sharp Drops and Recoveries](#pattern-1-sustained-high-plateau-with-periodic-sharp-drops-and-recoveries)

---

## Pattern 1: Sustained High Plateau with Periodic Sharp Drops and Recoveries

**What it looks like:**
One series (most commonly `PollWorkflowTaskQueue`) climbs to a very high RPS of `ResourceExhausted` responses — often in the tens of thousands — and stays there as a sustained plateau. Periodically, the line drops sharply to near-zero before spiking back up to the same ceiling. Other series (completions, starts, signals) remain near-zero or at low rates throughout.

**What it means:**
The namespace execution bucket RPS quota is saturated. Because `PollWorkflowTaskQueue` is Priority 4 — the lowest priority in the Execution bucket — it is the first operation to be shed under load. The server is correctly protecting higher-priority operations (workflow completions, starts, signals) while throttling polls. The periodic drops are **not necessarily worker restarts** — see distinguishing causes below.

**Impact:**
This pattern is severely disruptive to workflow workers and should be treated as high priority:

- **Schedule-to-start latency increases** for workflow tasks — workers cannot pick up tasks fast enough because their polls are being rejected
- **Sync match rate drops** — tasks cannot be handed off directly to a waiting poller within the sync match window (~500ms) and are instead written to the database, adding latency to every workflow task dispatch
- **Event delivery delays** — signals, activity completions, and timers firing all experience delays getting delivered to workers, since workers are not polling successfully
- **Prolonged end-to-end workflow execution** — the compounding effect of all of the above means workflows take significantly longer to complete than they should
- **Potential significant business impact** — depending on what workflows power in your application, this can directly affect SLAs, user-facing latency, or critical business processes

**Possible causes of the periodic sharp drops:**

| Cause | Description |
|---|---|
| **Workload-driven (most common)** | A large batch of workflows was executing activities or local activities — during which workers do not need to poll for workflow tasks. Poll pressure drops naturally. When those activities finish and workflows need to progress, workers flood back to polling and immediately re-hit the quota ceiling. |
| **Workflow executions completing** | A large number of executions completed, temporarily reducing the number of workers actively polling. Once new work arrives, polling resumes at full pressure. |
| **Worker restarts or scaling events** | Workers restarted or new instances came online, briefly reducing in-flight poll connections before resuming at the same rate. |

**How to distinguish between causes:**

| Signal to check | What it tells you |
|---|---|
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