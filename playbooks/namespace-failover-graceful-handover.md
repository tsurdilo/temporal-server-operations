# Temporal Namespace Failover — Graceful Handover Playbook

> **Scope.** This playbook covers **planned graceful handover only** using the `namespace-handover-v2` workflow. Both clusters must be reachable and healthy before starting.

## Before You Start

- [Recommendations](#recommendations)
- [Dashboard and workflow steps](#dashboard-and-workflow-steps)
- [What your clients and workers experience](#what-your-clients-and-workers-experience)
- [Setting dcRedirectionPolicy](#setting-dcredirectionpolicy)
- [Forwarding policy reference](#forwarding-policy-reference)
- [Dynamic config](#dynamic-config)
- [Alerts](#alerts)
- [Temporal Schedules](#temporal-schedules)
- [Archival](#archival)
- [Starting the Handover Workflow](#starting-the-handover-workflow)

---

### Recommendations

Quick reference — all recommendations below are explained in detail in the sections that follow.

- Think carefully about `dcRedirectionPolicy` — it is a static YAML setting that requires a restart and controls how each cluster handles traffic when it is not the active. Get it right before the first handover. See [Setting dcRedirectionPolicy](#setting-dcredirectionpolicy).
- Verify `numHistoryShards` is a clean multiple between clusters (e.g. 512 and 1024 is valid; 512 and 768 is not). A non-multiple ratio causes the replication stream to fail at connection time with no recovery path — this must be correct when replication is first configured.
- Never run two handover workflows for the same namespace at the same time.
- Set `history.EnableReplicationTaskTieredProcessing: true` on **both clusters**. Dynamic config, no restart needed. Required for the handover workflow to function correctly. Both clusters must be on v1.25.0 or later before enabling — enabling it when either cluster is on an older version can cause the replication stream to get stuck.
- Set `system.enableNamespaceHandoverWait: true` on **both clusters**. Dynamic config, no restart needed. Eliminates the `Unavailable` error during the HANDOVER drain window — clients see latency instead.
- Confirm `system.enableNamespaceNotActiveAutoForwarding` is `true` on both clusters (it is the default). If it was ever set to `false`, forwarding is disabled regardless of `dcRedirectionPolicy`.
- Do not lower `history.standbyTaskMissingEventsDiscardDelay` below its default (15m) on the standby — lower values increase the risk of workflows getting stuck post-flip.
- Use `AllowedLaggingSeconds: 10`, `AllowedLaggingTasks: 500`, `HandoverTimeoutSeconds: 30` as starting values. Tune `AllowedLaggingTasks` up on high-throughput clusters if WaitReplication stalls.
- Use a consistent workflow ID convention — `handover-<namespace>` — so running handovers are easy to spot before starting a new one.
- Use `system.forceNamespaceSelectedAPIAutoForwarding` (namespace-scoped dynamic config, no restart) to control which cluster's workers actively participate in workflow execution before and after a handover — see [Forwarding policy reference](#forwarding-policy-reference).
- If you use Schedules, confirm `worker.enableScheduler` is `true` (or not explicitly set to `false`) on the cluster you are failing over to — if `false`, schedules will not fire after the handover. See [Temporal Schedules — Pre-handover](#pre-handover).
- If you use Schedules, ensure each schedule's `catchup_window` (a per-schedule spec field, default 365 days) is longer than the expected handover duration — fires outside that window are not backfilled after the handover.

### Dashboard and workflow steps

**Dashboard:** [Temporal — Namespace Failover: Graceful Handover](../metrics/dashboards/server/namespace-failover-graceful-handover-readme.md) — v1.3.0

| Row | What you are watching for |
|---|---|
| **1. Pre-Flight** | Is the replication stream healthy and caught up? Nothing should be red before you start. |
| **2a. WaitReplication** | Is the standby catching up? Progress rises steadily with no client impact. |
| **2b. HANDOVER Drain** | Is the drain completing in time? Writes are briefly blocked — this window is at most 30 seconds. |
| **3. Flip Confirmed** | Did the flip happen cleanly? Four indicators turn green at once when the namespace is live on the new active. |
| **4. Post-Handover Health** | Is the new active stable? Watch for the first 5–10 minutes after flip — traffic, latency, and replication should all normalize. |

The dashboard rows and playbook sections map directly to the six internal activity steps of the `namespace-handover-v2` workflow:

| Step | Activity | Dashboard row |
|---|---|---|
| 1 | `GetMetadata` | 2a. WaitReplication |
| 2 | `GetMaxReplicationTaskIDs` | 2a. WaitReplication |
| 3 | `WaitReplication` loop | 2a. WaitReplication |
| 4 | `UpdateNamespaceState(HANDOVER)` | 2b. HANDOVER Drain |
| 5 | `WaitHandover` loop | 2b. HANDOVER Drain |
| 6 | `UpdateActiveCluster` | 3. Flip Confirmed |

### What your clients and workers experience

What happens at each phase depends on two things: which cluster your clients and workers are pointed at, and how `dcRedirectionPolicy` is configured on that cluster. This policy is a static YAML setting — it requires a server restart to change and must be in place before any handover. Both clusters should run `all-apis-forwarding`. See [how to set it](#setting-dcredirectionpolicy) below.

**Steps 1–3 (WaitReplication) — no change from normal operation.** The workflow is running in the background. Your clients and workers see nothing different.

**Steps 4–5 (HANDOVER Drain) — brief write block on the active cluster.** When the namespace enters HANDOVER state, the active cluster's frontend returns `Unavailable` on API calls for this namespace. This is namespace-scoped — other namespaces on the same cluster are not affected.

| Who is affected | What they see |
|---|---|
| Clients calling StartWorkflow, SignalWorkflow, etc. | `Unavailable` — **SDK retries automatically.** Requests are delayed, not failed, as long as the drain completes before the SDK's retry deadline. |
| Workers calling RespondWorkflowTaskCompleted, heartbeats | `Unavailable` — **SDK retries automatically.** In-flight WFTs are safe — the worker holds the result and retries. |

> **`system.enableNamespaceHandoverWait` controls whether callers see a latency spike or an error.** When `true` (recommended), the frontend holds the connection open and releases it the moment HANDOVER exits — callers see a latency increase but no error at all. When `false` (default), `Unavailable` is returned immediately and the SDK retries. Set this to `true` on both clusters as a permanent standing configuration — no restart required. This is how Temporal Cloud operates.

**Step 6 and beyond (post-flip) — what clients and workers experience depends on `dcRedirectionPolicy` on the old active.**

After the flip, the old active becomes a standby. Workers or clients still pointing at it will get different behavior depending on its policy:

| Operation | Old active has `all-apis-forwarding` | Old active has `selected-apis-forwarding` |
|---|---|---|
| Client write APIs (StartWorkflow, Signal, Cancel, etc.) | Forwarded to new active transparently | Forwarded to new active transparently |
| Worker polls | Forwarded to new active — workers continue processing | **Not forwarded.** The poll reaches the standby frontend but the DC redirection interceptor does not proxy it to the active cluster. The call passes through to the standby's own matching service — which has no tasks for a passive namespace — and the long-poll times out returning empty. Workers stay connected and keep polling but receive no work. No SDK error is raised. |
| Worker Respond*, heartbeats, UpdateWorkflow | Forwarded to new active — in-flight work completes | **`NamespaceNotActive`** (`FailedPrecondition`) — not retried by SDK. The WFT result is not committed. The WFT stays in "started" state on the new active until its `StartToCloseTimeout` fires, at which point the new active reschedules it and a worker there re-executes it. The workflow eventually continues, but each affected WFT incurs a full timeout delay and must be re-executed from scratch. |

#### Setting `dcRedirectionPolicy`

Set this in your server static config YAML **when you first configure replication** — not as a pre-handover step. It requires a server restart to change. Both clusters should run `all-apis-forwarding`:

```yaml
# On both clusters
dcRedirectionPolicy:
  policy: "all-apis-forwarding"
```

| Cluster | Required value | Why |
|---|---|---|
| **Active (failing away from)** | `"all-apis-forwarding"` | After the flip this cluster becomes standby. Workers still pointed here need `Respond*`, heartbeats, and `UpdateWorkflowExecution` forwarded to the new active. Without it they get `NamespaceNotActive` (non-retryable) and WFTs must be re-executed. |
| **Standby (failing over to)** | `"all-apis-forwarding"` | Before the flip, any write API calls that accidentally hit the standby are forwarded to the active. After the flip it becomes the new active and serves requests directly. |

> **There is no dynamic config equivalent for upgrading to `all-apis-forwarding`.** The only runtime knob is `system.forceNamespaceSelectedAPIAutoForwarding`, which can downgrade a cluster running `all-apis-forwarding` to whitelist-only for a specific namespace — but it cannot upgrade `selected-apis-forwarding` to `all-apis-forwarding`. The base policy must be set in YAML. See [Forwarding policy reference](#forwarding-policy-reference) for the full breakdown.

### Forwarding policy reference

What each policy forwards — and what it does not:

| Operation | `all-apis-forwarding` | `selected-apis-forwarding` |
|---|---|---|
| Starting and signalling workflows | ✓ | ✓ |
| Worker polls — picking up workflow and activity tasks | ✓ | ✗ |
| Workers completing tasks (`RespondWorkflowTaskCompleted`, `RespondActivityTask*`) | ✓ | ✗ |
| Activity heartbeats (`RecordActivityTaskHeartbeat`) | ✓ | ✗ |
| `UpdateWorkflowExecution`, `ExecuteMultiOperation` | ✓ | ✗ — not yet in whitelist (tracked) |

**The `selected-apis-forwarding` whitelist — the exact 11 APIs that are forwarded:**

```
StartWorkflowExecution
SignalWithStartWorkflowExecution
SignalWorkflowExecution
RequestCancelWorkflowExecution
TerminateWorkflowExecution
DeleteWorkflowExecution
QueryWorkflow
StartActivityExecution
RequestCancelActivityExecution
TerminateActivityExecution
DeleteActivityExecution
```

Everything else — polls, Respond*, heartbeats, Update, ExecuteMultiOperation — is not forwarded and will be served locally by the standby (which means no work for workers, and errors for anything that requires the active).

> `UpdateWorkflowExecution` and `ExecuteMultiOperation` are not in the whitelist as of the current server version. A server issue has been filed to add them. Until then, Update calls to a cluster running `selected-apis-forwarding` while it is not the active will fail with `NamespaceNotActive`.

How `system.forceNamespaceSelectedAPIAutoForwarding` interacts with the static policy:

| Static `dcRedirectionPolicy` | `system.forceNamespaceSelectedAPIAutoForwarding` | Effective behavior |
|---|---|---|
| `all-apis-forwarding` | `false` (default) | Full forwarding — all operations including polls |
| `all-apis-forwarding` | `true` (namespace-scoped) | Whitelist-only for that namespace — polls and worker task completion not forwarded. No restart needed. |
| `selected-apis-forwarding` | `false` (default) | Whitelist-only |
| `selected-apis-forwarding` | `true` | No change — already whitelist-only; this setting has no effect |

The practical implication: `system.forceNamespaceSelectedAPIAutoForwarding` is only meaningful if the static policy is `all-apis-forwarding`. Setting `all-apis-forwarding` in static config once gives you the flexibility to downgrade per-namespace at runtime without ever needing another restart.

**Using `system.forceNamespaceSelectedAPIAutoForwarding` to control worker participation across a handover — without restarting clusters:**

This requires both clusters to have `dcRedirectionPolicy: all-apis-forwarding` set in static config (one-time restart). After that, the dynamic config becomes the only knob you need.

*Before handover:*
- **Active cluster** — leave `system.forceNamespaceSelectedAPIAutoForwarding: false` (default). Full forwarding. Workers on the active poll and complete tasks normally.
- **Standby cluster** — set `system.forceNamespaceSelectedAPIAutoForwarding: true` (namespace-scoped). Downgrades to whitelist-only. Workers deployed on the standby do not receive polls and cannot complete tasks — they are effectively idle. Only relevant if you have workers pointed at both clusters; if workers only point at the active, this is a no-op.

*After handover (flip):*
- **New active (was standby)** — set `system.forceNamespaceSelectedAPIAutoForwarding: false`. Full forwarding restored. Workers on the new active start picking up tasks immediately.
- **Old active (now standby)** — do not set `true` immediately. Workers still pointed here may have in-flight WFTs they are completing — with `all-apis-forwarding` and `false`, those Respond* calls are forwarded to the new active transparently. Wait until in-flight work has drained and workers have been re-pointed to the new active, then set `true` to cut off remaining connections cleanly.

### Dynamic config

These must be set on a running cluster before the first handover — no restart required. Treat them as permanent standing configuration for any multi-cluster deployment.

| Config | Cluster | Required value | How to check |
|---|---|---|---|
| `system.enableNamespaceNotActiveAutoForwarding` | Both | `true` (default) | Confirm nobody set it `false` — if `false`, forwarding is disabled regardless of static policy |
| `history.EnableReplicationTaskTieredProcessing` | Both | **`true`** | If `false`, a LOW force-replication backlog can block WaitHandover and trigger the 30s rollback; also prevents namespace-filtered progress tracking |
| `history.standbyTaskMissingEventsDiscardDelay` | Standby | Default (15m) or higher | Lower values increase discard risk during elevated lag |

**`history.EnableReplicationTaskTieredProcessing` is safe to enable on a running cluster** — the sender detects the change and triggers a controlled stream reconnect (~1s re-send window). Set it on **both clusters** before the handover: after the flip the standby becomes the new active and immediately starts sending its own replication stream — it needs tiered processing already on. Do not toggle it back off once enabled. Both clusters must be on v1.25.0 or later before enabling — enabling it when either cluster is on an older version can cause the replication stream to get stuck.

### Alerts

Full reference: [`metrics/alerts/server/alerts-index.md`](../metrics/alerts/server/alerts-index.md) — Section 20

All alerts below link to the [Namespace Failover: Graceful Handover dashboard](../metrics/dashboards/server/namespace-failover-graceful-handover-readme.md) and fire annotations directly onto it — alert state is visible in context alongside the panels they monitor.

| Alert | Severity | Workflow step | Dashboard row | Fires when |
|---|---|---|---|---|
| `FAILOVER-PRE-01` — Replication Stream Stuck | critical | Pre-flight (before Step 1) | Row 1 | Any stream stopped making progress — hard blocker for handover |
| `FAILOVER-PRE-02` — Replication DLQ Enqueue Failing | critical | Pre-flight (before Step 1) | Row 1 | Tasks failing into DLQ on stream path — hard blocker for handover |
| `FAILOVER-PRE-03` — Replication Stream Errors Sustained | critical | Pre-flight (before Step 1) | Row 1 | `replication_stream_error` sustained for 2m — stream cycling through reconnects, hard blocker for handover |
| `FAILOVER-PRE-04` — Receiver Backlog At Flow Control Limit | critical | Pre-flight (before Step 1) | Row 1 | Receiver backlog p99 ≥ 500 — flow control active, hard blocker |
| `FAILOVER-PRE-05` — Receiver Backlog Near Flow Control Limit | warning | Pre-flight (before Step 1) | Row 1 | Receiver backlog p99 ≥ 400 — approaching flow control ceiling, advisory |
| `FAILOVER-PRE-06` — Standby Task Discards Detected | warning | Pre-flight (before Step 1) | Row 1 | `task_errors_discarded` non-zero — stuck workflows will exist post-flip |
| `FAILOVER-PRE-07` — Replication Latency Too High | critical | Pre-flight (before Step 1) | Row 1 | `replication_latency` p99 ≥ 20s on standby — tasks taking too long to replicate; handover drain will fail |
| `FAILOVER-HANDOVER-01` — HANDOVER Drain Stalled | critical | Step 5 (`WaitHandover`) | Row 2b | `handover_ready_shard_count` not progressing during drain — 30s rollback clock running |
| `FAILOVER-POST-01` — Forwarding FailedPrecondition Errors | warning | Post Step 6 | Row 4 | `FailedPrecondition` errors on old active — workers or clients hitting non-forwarded APIs (Respond\*, Update, ExecuteMultiOperation) while still pointed at old active |

---

### Temporal Schedules

Schedules are a Temporal feature for running workflows on a recurring basis. **If you are not using Schedules in this namespace, skip this section — return to it if you adopt the feature later.**

- [Pre-handover](#pre-handover)
- [During handover (Steps 4–5)](#during-handover-steps-45--handover-drain)
- [After handover (Step 6+)](#after-handover-step-6-and-beyond)

Each Schedule is a durable workflow execution (`temporal-sys-scheduler-workflow`) running in the user namespace on task queue `temporal-sys-per-ns-tq`. It is replicated to the standby cluster and handled across a handover exactly like any other workflow in the namespace — no special treatment exists for schedule workflows at the replication or failover layer.

#### Pre-handover

Schedule workflows are replicated to the standby like any other workflow in the namespace. The standby does not start per-namespace SDK workers for inactive namespaces, so the scheduler never runs there — no schedule workflow tasks execute and no triggered workflow executions are started on the standby. Only the active fires schedules.

**Before running the handover workflow:**

- Confirm `worker.enableScheduler` is `true` on the new active — if `false`, schedules will not fire after the handover (see [Recommendations](#recommendations)).
- Align schedule-related dynamic configs between clusters, particularly `worker.schedulerNamespaceStartWorkflowRPS` (default 30/s per namespace). If these differ, catchup rate after handover will behave differently than on the old active.
- Ensure the worker service on the new active is scaled to match the old active. If the new active has fewer worker pods, each pod carries proportionally more scheduler load after the flip — scale up before running the handover if needed.

#### During handover (Steps 4–5 — HANDOVER drain)

During the HANDOVER drain, the scheduler workflow cannot fire new executions.

- **Scheduled fires are delayed, not lost.** Any fire due during the drain is held and retried — it will execute on the new active once the handover completes.
- **In-progress workflow starts are retried automatically.** If the scheduler was in the middle of starting a triggered workflow execution, the call is retried and will complete after the flip.

**RPC operations during drain:**
- `ListSchedules` — allowed.
- `CreateSchedule`, `UpdateSchedule`, `DeleteSchedule`, `PatchSchedule`, `DescribeSchedule` — blocked, return `Unavailable`. The SDK client retries these automatically and they will resolve once the drain completes. If you need to pause a schedule, do it before starting the handover workflow and unpause on the new active once the handover completes.

#### After handover (Step 6 and beyond)

**On the old active (now standby):** Per-namespace scheduler workers shut down gracefully.

**On the new active (formerly standby):** Per-namespace scheduler workers start and begin processing schedules.

Once started, the per-namespace scheduler workers resume schedule operations from where they left off — they do not restart from scratch.

**Catchup.** When the scheduler resumes on the new active, it backfills any fires that were due during the standby period and the handover drain. For a handover completing in seconds to low minutes, this is a small number of fires and completes quickly. By default the catchup window is 365 days — no fires are dropped for a normal handover.

The only gap where fires can be missed is Steps 4–5 (the HANDOVER drain), which is at most 30 seconds. With the default `catchup_window` of 365 days this is never an issue. It only becomes relevant if you have explicitly set a very short `catchup_window` on a schedule — shorter than the drain duration.

**Paused schedules** remain paused on the new active after the handover.

#### Verifying schedule health after handover

**Live state — `temporal schedule describe`**

The most direct check. `temporal schedule describe` issues a live query to the scheduler workflow execution — it returns real-time state from the running workflow, not a cached visibility entry. Run it on the **new active** cluster:

```bash
temporal schedule describe \
  --schedule-id <schedule-id> \
  --namespace <namespace> \
  --address <new-active-frontend>
```

Key fields to read from the output:

| Field | What to look for |
|---|---|
| `RecentActions` | Should contain recent fire timestamps — confirms the scheduler is executing |
| `FutureActionTimes` | Should contain upcoming fire times — confirms the timer is running |
| `MissedCatchupWindow` | Should be zero for a normal handover |
| `RunningWorkflows` | Lists currently executing triggered workflow runs |

If `FutureActionTimes` shows times that are all in the past and `RecentActions` is not advancing, the scheduler is not making progress. Check dynamic config (`worker.enableScheduler`) and worker service health on the new active.

**`temporal schedule list` showing schedules as `Running` is not confirmation they are making progress** — use `temporal schedule describe` to confirm a schedule is actually firing.

**Visibility queries** can give a quick namespace-level view. Run these against the new active:

```
# All running, unpaused schedules
TemporalNamespaceDivision = 'TemporalScheduler' AND TemporalSchedulePaused = false AND ExecutionStatus = 'Running'

# All paused schedules — confirm schedules paused on the old active are still paused
TemporalNamespaceDivision = 'TemporalScheduler' AND TemporalSchedulePaused = true
```

These are visibility queries (not live state) — they confirm presence and pause state but not whether the scheduler is actively firing. Use `temporal schedule describe` for live health.

**Useful metrics to monitor after handover on the new active**

| Metric | What to expect |
|---|---|
| `schedule_action_delay` | Should return to normal shortly after schedules resume |
| `schedule_action_e2e_delay` | May be temporarily elevated if many fires are being caught up |
| `schedule_action_errors` | Should be zero — any errors are worth investigating |
| `schedule_missed_catchup_window` | Should be zero for a normal handover |
| `schedule_rate_limited` | May spike briefly during catchup — if sustained, raise `worker.schedulerNamespaceStartWorkflowRPS` |
| `schedule_buffer_overruns` | Should be zero — non-zero means fires were dropped during catchup |

**Scanner workflows (optional, disabled by default)**

The server includes scanner workflows that periodically check all schedules in a namespace for health issues and emit metrics when problems are found. They are disabled by default and controlled via `worker.scheduleInvariantsScannerOptions` (global dynamic config, not per-namespace — enabling it applies to all namespaces on the cluster).

Once enabled, the key metric to watch after handover is `schedule_invariants_scanner_overdue_next_action_time` — tagged by `namespace`, so you can filter to just the namespace you failed over. A non-zero value means one or more schedules are overdue and not firing when they should be.

---

### Archival

Temporal can archive closed workflow execution history and visibility records to external storage (S3 or compatible). **If you are not using archival in this namespace, skip this section.**

Both clusters archive independently. When a workflow closes, each cluster generates and processes its own archival task — there is no active/passive gate on the archival path. If both clusters point to the same storage bucket, both attempt to upload the same workflow history. The second cluster checks whether the blob already exists and skips the upload if it does — so you get one copy, not two. However, both clusters make API calls to storage for every workflow close, including the existence check. Under sustained high load this can double the storage API request volume to S3.

**Shared bucket (both clusters point to the same storage):**

Both clusters independently generate archival tasks for every workflow close and send requests to S3 — one copy of the archive is written, but both clusters make API calls (an existence check plus upload on the first to arrive, an existence check only on the second). Under high throughput this can put significant pressure on S3.

Metrics to watch on both clusters:
- `history_archiver_blob_exists` — non-zero on either cluster means it found a blob already uploaded by the other; a sustained high rate on either side means existence checks are adding meaningful S3 request volume
- `history_archiver_archive_transient_error` — rate of retryable S3 failures (throttling, timeouts); a rising rate means S3 is under pressure

If visibility archival is also enabled, each cluster makes an additional 4 S3 PutObject calls per workflow close for index creation — no existence check is performed before each write, so under a shared bucket both clusters write the same 4 indexes independently. Watch `visibility_archiver_archive_transient_error` on both clusters for visibility-specific S3 failures. If both history and visibility archival are enabled and the transient error metrics above are rising, note that `history.archivalBackendMaxRPS` covers both types combined — you may need to raise it on both clusters to account for the higher combined request volume.

If S3 starts returning errors, archival tasks fail and retry with backoff. Retrying tasks accumulate in memory on history pods — the archival queue is separate from replication and WFT processing so it does not block workflow execution or replication tasks, but sustained S3 failures under high load can cause memory pressure on history pods over time.

If you are seeing rising values in the metrics above, options to reduce S3 pressure:
- Lower `history.archivalBackendMaxRPS` on the standby (dynamic config, default 10,000) — limits how fast the standby sends requests to S3
- Increase `history.archivalProcessorArchiveDelay` on the standby (dynamic config, default 5 minutes) — spreads standby archival tasks over a longer window, reducing burst volume; after a handover, lower it on the new active and raise it on the new standby (dynamic config, no restart needed)
- Switch to separate buckets — eliminates the double-write entirely; this is a larger configuration change that requires reconfiguring storage on both clusters — consider if the tuning options above are not sufficient

> **Do not disable archival to address S3 pressure.** Archival configuration is part of namespace metadata and replicates between clusters — disabling it on one side propagates the change to the other and removes archival on both clusters.

#### During handover

The namespace handover does not affect archival of closed workflow executions — workflows that close during this phase are archived normally.

#### After handover

If you tuned `history.archivalBackendMaxRPS` or `history.archivalProcessorArchiveDelay` on the standby before the handover, swap them after the flip — raise `history.archivalBackendMaxRPS` on the new active and lower it on the new standby; apply the same swap to `history.archivalProcessorArchiveDelay` if you changed it. Both are dynamic config, no restart needed.

Watch `history_archiver_blob_exists` and `history_archiver_archive_transient_error` on both clusters to confirm S3 pressure has normalized.

If you are on separate buckets, the new active writes to its own bucket from this point — verify your retrieval tooling accounts for the split.

---

### Starting the Handover Workflow

**Before starting — check for a running handover first:**

```bash
temporal workflow list \
  --namespace temporal-system \
  --query 'WorkflowType="namespace-handover-v2" AND ExecutionStatus="Running"'
```

If the results are empty you are clear to proceed. If you see a result for your namespace, do not start another one.

**Decide your input parameters before starting:**

| Parameter | Bounds (enforced) | Guidance |
|---|---|---|
| `AllowedLaggingSeconds` | [5, 120] | How far behind standby can be before entering HANDOVER. 5–10s for low-lag clusters. Values outside range are clamped by the server; 120 is a valid achievable maximum. |
| `AllowedLaggingTasks` | No bounds | Alternative lag gate in task count. Gate condition is OR — a shard passes when task lag ≤ this value OR time lag ≤ `AllowedLaggingSeconds`. Setting to `0` is the strictest threshold (any lagging tasks must pass via the time gate); to make this non-binding, set to a very large number. |
| `HandoverTimeoutSeconds` | Max 30s | How long WaitHandover waits before rolling back. **Recommended: 30s** — gives shards the full drain window. Only go lower if you specifically want a faster rollback at the cost of a narrower window. |

The 30-second cap on `HandoverTimeoutSeconds` is a hardcoded constant (`maximumHandoverTimeoutSeconds = 30` in `handover_workflow.go`) — not a dynamic config, not overridable.

The handover is triggered by starting `namespace-handover-v2` on the **active cluster** in the `temporal-system` namespace:

```bash
temporal workflow start \
  --namespace temporal-system \
  --type namespace-handover-v2 \
  --task-queue default-worker-tq \
  --workflow-id handover-<your-namespace> \
  --input '{
    "Namespace": "<your-namespace>",
    "RemoteCluster": "<standby-cluster-name>",
    "AllowedLaggingSeconds": 10,
    "AllowedLaggingTasks": 500,
    "HandoverTimeoutSeconds": 30
  }'
```

**Why `--workflow-id handover-<your-namespace>`:** this is a recommended convention (e.g. `handover-my-ns`). A deterministic, human-readable ID makes the workflow easy to locate with `workflow describe` and easy to check for duplicates before starting. The default workflow ID reuse policy (`ALLOW_DUPLICATE`) does not protect against concurrent runs — always run the duplicate check below before starting.

> **One handover at a time — and let it finish on its own.**
>
> Never start a second handover for the same namespace while one is already running. Always check for a running workflow first (see the duplicate check below).
>
> Do not `terminate` or `cancel` a running handover. From the outside you cannot tell which step it is on. If it is in Steps 4–5 (the HANDOVER drain), interrupting it can leave the namespace stuck in `REPLICATION_STATE_HANDOVER` — write APIs blocked indefinitely. Let it complete or roll back on its own. If it appears genuinely stuck, see [Section 5](#5-something-went-wrong--rollback-and-recovery).
>
> Using `handover-<namespace>` as the workflow ID (e.g. `handover-my-ns`) makes it easy to spot a running handover at a glance.

**Workflow input parameters (`--input` JSON):**

| Input field | Type | Bounds | Description |
|---|---|---|---|
| `Namespace` | string | Required | The global namespace to fail over |
| `RemoteCluster` | string | Required | The name of the standby cluster to become the new active — must match the cluster name in your Temporal server config |
| `AllowedLaggingSeconds` | int | [5, 120] (enforced) | Maximum replication time lag allowed on the standby before WaitReplication declares it close enough to proceed. Values outside this range are clamped by the server. 120 is a valid achievable value. |
| `AllowedLaggingTasks` | int64 | No bounds | Maximum replication task count lag allowed on the standby. The gate condition is OR: a shard passes when `laggingTasks ≤ AllowedLaggingTasks OR timeLag ≤ AllowedLaggingSeconds`. Setting to `0` means shards with any lagging tasks must pass via the time gate — it does not disable the task gate, it sets the strictest possible threshold. To make the task count non-binding, set to a very large number. |
| `HandoverTimeoutSeconds` | int | Max 30s (enforced) | How long WaitHandover (Step 5) waits for the drain to complete before rolling back automatically. Values above 30 are clamped to 30 — this cap is hardcoded in the server |

> **Always set `HandoverTimeoutSeconds` explicitly — never omit it or set it to `0`.** A zero value causes the drain to time out immediately and roll back. **Recommended value: 30s** — the maximum the server allows, giving shards the full drain window.

---

## Running the Handover

This section is the operational playbook for running a graceful handover end-to-end — from pre-flight signal checks through the handover workflow execution and into post-flip health verification. It is structured around the six internal activity steps of `namespace-handover-v2` and maps directly to the [Namespace Failover: Graceful Handover dashboard](../metrics/dashboards/server/namespace-failover-graceful-handover-readme.md). The dashboard is an integral part of this playbook — each section below corresponds to a dashboard row.

> This section assumes you have read [Before You Start](#before-you-start) and applied the recommendations there.

### Table of Contents

- [1. Check if it is safe to start the handover](#1-check-if-it-is-safe-to-start-the-handover)
- [2a. Handover workflow started — monitor standby catchup (Steps 1–3)](#2a-handover-workflow-started--monitor-standby-catchup-steps-13)
- [2b. Namespace is in HANDOVER — monitor the drain window (Steps 4–5)](#2b-namespace-is-in-handover--monitor-the-drain-window-steps-45)
- [3. Flip completed — confirm the namespace switched correctly (Step 6)](#3-flip-completed--confirm-the-namespace-switched-correctly-step-6)
- [4. Monitor the new active cluster after the handover](#4-monitor-the-new-active-cluster-after-the-handover)
- [5. Something went wrong — rollback and recovery](#5-something-went-wrong--rollback-and-recovery)

---

## 1. Check if it is safe to start the handover

**Goal:** Work through these steps to assess the replication health between your active and standby clusters and make an informed decision on whether it is safe to proceed with the handover.

1. [Do you need to run the force-replication workflow?](#11-do-you-need-to-run-the-force-replication-workflow)
2. [Is the replication stream between clusters healthy?](#12-is-the-replication-stream-between-clusters-healthy)
3. [Is replication lag low enough to proceed?](#13-is-replication-lag-low-enough-to-proceed)
4. [Is the standby still catching up on missing history?](#14-is-the-standby-still-catching-up-on-missing-history)
5. [Is the standby keeping up with incoming replication tasks?](#15-is-the-standby-keeping-up-with-incoming-replication-tasks)
6. [Are there workflow executions on the standby that will be stuck after the flip?](#16-are-there-workflow-executions-on-the-standby-that-will-be-stuck-after-the-flip)

#### 1.1. Do you need to run the force-replication workflow?

If required, the `force-replication-v2` workflow must be run and complete **before** you start the handover workflow — for the same namespace you intend to hand over.

| Situation | Action |
|---|---|
| **Brand-new standby** — cluster was recently added and has never received live replication for this namespace | **Run `force-replication-v2`.** The stream only carries tasks from the point it was connected — all executions that existed before have no replication tasks in flight and will be missing on the standby after the flip. |
| **Established standby after a replication outage** — stream was interrupted but cluster was previously in sync | **Skip.** The stream self-recovers via watermarks once connectivity is restored. Do not run force-replication — it adds unnecessary backlog. |
| **Established standby, no outage, normal operations** | **Skip.** |

If you need to run it, use the following command and wait for it to complete before starting the handover workflow:

```bash
temporal workflow start \
  --namespace temporal-system \
  --type force-replication-v2 \
  --task-queue default-worker-tq \
  --input '{
    "Namespace": "<your-namespace>",
    "Query": "",
    "ConcurrentActivityCount": 5
  }'
```

**Workflow input parameters (`--input` JSON):**

| Input field | Type | Default | Description |
|---|---|---|---|
| `Namespace` | string | Required | The namespace to backfill |
| `Query` | string | Required | Visibility query to scope which executions to backfill. Empty string backfills all executions in the namespace |
| `ConcurrentActivityCount` | int | `1` | Number of replication activities to run in parallel. Raise for faster backfill; lower if receiver backlog builds up |
| `OverallRps` | float64 | `= ConcurrentActivityCount` | RPS cap for enqueuing replication tasks across all concurrent activities |
| `GetParentInfoRPS` | float64 | `= ConcurrentActivityCount` | RPS cap for parent/child relationship lookups during backfill |
| `ListWorkflowsPageSize` | int | `1000` | Page size for listing workflows via visibility |
| `PageCountPerExecution` | int | `200` (max `1000`) | Number of pages processed before the workflow continues-as-new to avoid history size limits |
| `EnableVerification` | bool | `false` | After backfill, verify executions were replicated to the target cluster |
| `TargetClusterEndpoint` | string | — | Required when `EnableVerification=true` if `TargetClusterName` is not set |
| `TargetClusterName` | string | — | Required when `EnableVerification=true` if `TargetClusterEndpoint` is not set |
| `VerifyIntervalInSeconds` | int | `5` | Polling interval for verification checks |

Wait for the workflow to complete before starting the handover workflow.

#### 1.2. Is the replication stream between clusters healthy?

Alerts [`FAILOVER-PRE-01` through `FAILOVER-PRE-07`](../metrics/alerts/server/alerts-index.md) fire on the blockers covered here. If any are already firing, you have your answer before opening the dashboard — but check the dashboard anyway. Alerts tell you something is wrong; the dashboard panels below tell you how bad it is, whether it is recovering, and what to do about it.

Open [Row 1 — Pre-Flight](../metrics/dashboards/server/namespace-failover-graceful-handover-readme.md#row-1--pre-flight-go--no-go) in the dashboard.

| Panel | Blocker | Advisory |
|---|---|---|
| [Stream Stuck](../metrics/dashboards/server/namespace-failover-graceful-handover-readme.md#panel-stream-stuck-stat) | Any non-zero rate → **blocker** | The standby has a built-in liveness monitor — if the stream stops making progress it reconnects automatically within ~3 minutes. Wait before acting. If it does not clear, see section [5](#5-something-went-wrong--rollback-and-recovery). |
| [DLQ Enqueue Failed](../metrics/dashboards/server/namespace-failover-graceful-handover-readme.md#panel-dlq-enqueue-failed-stat) | Any non-zero rate → **blocker** | Fires on the standby. The DLQ is the last resort for replication tasks the standby cannot apply — if even that is failing, those tasks are permanently lost. Running a handover in this state makes no sense. Investigate and fix before proceeding. |
| [Receiver Backlog](../metrics/dashboards/server/namespace-failover-graceful-handover-readme.md#panel-receiver-backlog-depth-stat) | ≥ 500 → **blocker**, 400–499 → **do not start** | Replication tasks are backing up on the standby faster than it can apply them. Starting the handover now risks the backlog hitting its limit mid-flight, which causes the sender to pause and WaitReplication to stall indefinitely. See section [1.5](#15-is-the-standby-keeping-up-with-incoming-replication-tasks). |
| [Stream Errors (gRPC)](../metrics/dashboards/server/namespace-failover-graceful-handover-readme.md#panel-stream-errors--grpc-stat) | Any non-zero rate → **blocker** | The replication stream between active and standby is currently in a bad state. Starting the handover now is dangerous — the standby is not reliably receiving updates from the active. Investigate the connection between clusters before proceeding. |
| [Stream Service Errors](../metrics/dashboards/server/namespace-failover-graceful-handover-readme.md#panel-stream-service-errors-stat) | Sustained unexplained rate → **investigate before proceeding** | Fires on the standby. A brief spike when the standby restarts is normal and clears on its own. A sustained rate with no obvious cause means the standby is struggling to process the replication stream — do not start until you understand why. |
| [Replication Lag](../metrics/dashboards/server/namespace-failover-graceful-handover-readme.md#panel-replication-lag-gauge) | Trending up → **do not start** | How far behind the standby is from the active. Aim for near-zero and stable before starting — a rising lag means the standby is falling further behind and the handover drain window will not be enough time to close the gap. See section [1.3](#13-is-replication-lag-low-enough-to-proceed). |
| [Backfill Activity](../metrics/dashboards/server/namespace-failover-graceful-handover-readme.md#panel-backfill-activity-stat) | — | The standby is receiving replication tasks but is missing some of the history behind them — it has to fetch that history from the active before it can apply each affected task. This slows down catchup. See section [1.4](#14-is-the-standby-still-catching-up-on-missing-history). |
| [Send Channel Full](../metrics/dashboards/server/namespace-failover-graceful-handover-readme.md#panel-send-channel-full-active-stat) | Any non-zero rate → **investigate** | Fires on the active. The active's internal buffer for queuing outgoing replication tasks is full — it cannot dispatch tasks to the standby fast enough. This is the earliest warning sign: left unaddressed, tasks start piling up (visible as a rising Send Backlog) and replication lag grows. See section [1.3](#13-is-replication-lag-low-enough-to-proceed) for how to raise the sender rate. |
| [Send Backlog](../metrics/dashboards/server/namespace-failover-graceful-handover-readme.md#panel-send-backlog-active-stat) | — | Fires on the active. Replication tasks are piling up on the active waiting to be sent to the standby — raising `history.ReplicationStreamSenderHighPriorityQPS` will help drain it. If this panel is zero but lag is still not closing, the active is already sending as fast as it can — the standby is the problem, applying tasks too slowly. Check standby history pod CPU and the Receiver Backlog panel. See section [1.3](#13-is-replication-lag-low-enough-to-proceed). |
| [Standby Task Discards](../metrics/dashboards/server/namespace-failover-graceful-handover-readme.md#panel-standby-task-discards-stat) | Any non-zero rate → **blocker** | Fires on the standby. Workflow executions on the standby had their tasks discarded because the history events they depended on never arrived in time. After the flip those workflows will be stuck on the new active with no pending task — direct business impact. Identify and understand the scope before proceeding. See section [1.6](#16-are-there-workflow-executions-on-the-standby-that-will-be-stuck-after-the-flip). |

> If any blocker above is unresolved, our recommendation is not to proceed with the handover. Resolve the issue first and re-run this check before continuing.

#### 1.3. Is replication lag low enough to proceed?

> Assumes the replication stream between the two clusters is healthy — complete section [1.2](#12-is-the-replication-stream-between-clusters-healthy) first.

Replication lag is how far behind the standby is from the active — measured in number of replication tasks still waiting to be applied. It matters here because the handover drain window is only 30 seconds. If the standby is significantly behind when the flip starts, the stream cannot deliver all remaining tasks in time and the handover rolls back automatically. The lower the lag before you start, the higher the chance the handover completes cleanly.

Start with the [Replication Lag Trend panel](../metrics/dashboards/server/namespace-failover-graceful-handover-readme.md#panel-replication-lag-trend-time-series) (Row 1, standby cluster) — it shows the direction of lag over the last 15 minutes. Then check the [Replication Lag gauge](../metrics/dashboards/server/namespace-failover-graceful-handover-readme.md#panel-replication-lag-gauge) for the current task count.

| What the trend shows | What it means | What to do |
|---|---|---|
| Declining toward zero | The standby is catching up | Wait for it to flatten near zero before starting |
| Rising | Lag is growing — the standby is falling further behind | Do not start. This can have several causes: a burst of high load on the active, standby history pods under pressure, or stream errors slowing delivery. Check the [Receiver Backlog Depth panel](../metrics/dashboards/server/namespace-failover-graceful-handover-readme.md#panel-receiver-backlog-depth-stat) and the stream health panels from section 1.2 to narrow it down. Wait until the trend reverses before proceeding. |
| Flat at a non-zero value | Standby is keeping pace with the active's current load — there is always a small in-flight queue under normal throughput | If the value is small (a few hundred tasks or fewer), this is normal — proceed. If it is large (thousands of tasks), the drain window may not be enough to clear it. In that case, raise `history.ReplicationStreamSenderHighPriorityQPS` (default 100 tasks/s) on the **active cluster** to push tasks faster, or wait for a lower-traffic window. |

**When you are ready to start the handover workflow**

When lag looks good and you are ready to proceed, use what you observed here to set the `AllowedLaggingTasks` input for the handover workflow. This value tells WaitReplication how close the standby needs to be before the workflow moves on to the drain window — it does not fix the lag, it just sets the exit threshold. Use the lag gauge to pick a value that reflects what you actually saw, not an arbitrary number.

| What the lag gauge shows | Recommended `AllowedLaggingTasks` |
|---|---|
| Consistently below 200 tasks and stable | 500 — standby will hit this quickly |
| Occasional spikes to 1 000+ tasks | Match or slightly exceed the spike value, or wait for the cluster to settle first |
| Thousands of tasks, not converging | Do not start — setting a high value just to proceed will likely cause the handover to roll back. WaitReplication will exit the moment the threshold is met, but the 30-second drain window must then clear all remaining tasks. If thousands are still in flight when HANDOVER starts, the drain times out and no flip occurs. |

> If lag is rising or flat at a large value, do not proceed. Wait for the trend to reverse and lag to stabilize near zero before continuing to section 1.4.

**Replication latency — the time-based signal.** The panels above measure task count — how many tasks are pending. Also check the [Replication Latency p99 panel](../metrics/dashboards/server/namespace-failover-graceful-handover-readme.md#panel-replication-latency-p99-time-series) (Row 1, standby cluster). This measures how long each task takes end-to-end from generation on the active to application on the standby. If p99 is approaching 30 seconds, any task in flight when HANDOVER starts cannot drain within the 30-second window — the handover is guaranteed to roll back regardless of task count. Alert `FAILOVER-PRE-07` fires when p99 exceeds 20s.

#### 1.4. Is the standby making extra round-trips to fetch missing history?

**Dashboard:** [Backfill Activity panel](../metrics/dashboards/server/namespace-failover-graceful-handover-readme.md#panel-backfill-activity-stat) (Row 1, standby cluster)

When the standby applies a replication task, it sometimes finds that the history events that task depends on have not arrived yet. When that happens, it makes a synchronous call back to the active to fetch them before it can continue. This is not an error — the task eventually completes — but each of these extra round-trips makes that task take longer to apply. A high rate of this means WaitReplication will take longer than the raw lag number alone suggests.

| What you see | What to do |
|---|---|
| Zero or low rate | Proceed — no impact on catchup timing |
| Elevated rate | WaitReplication will take longer than the lag number alone suggests — each affected task needs an extra round-trip to the active. It will drop naturally as the stream catches up. If you need the handover to complete quickly, wait for it to subside first. |

> This check is advisory only — there is no value here that blocks you from proceeding. Continue to section 1.5 regardless of what you see.

#### 1.5. Is the standby keeping up with incoming replication tasks?

**Dashboard:** [Receiver Backlog Depth panel](../metrics/dashboards/server/namespace-failover-graceful-handover-readme.md#panel-receiver-backlog-depth-stat) (Row 1, standby cluster)

Replication tasks travel from the active to the standby over a stream. When they arrive on the standby they land in a receive buffer before being applied. The Receiver Backlog Depth panel shows how many tasks are sitting in that buffer waiting to be processed.

When the buffer fills up past its limit (500 tasks by default), the standby tells the active to stop sending. The stream stays open but the active pauses dispatching new tasks. If you start the handover workflow at this point, the WaitReplication step — the internal step that waits for the standby to catch up before triggering the flip (Steps 1–3) — will stall. The standby stops making progress and WaitReplication keeps waiting until the backlog drains. In the worst case, WaitReplication times out after 1 hour with no flip occurring.

| Backlog depth (p99) | What it means | What to do |
|---|---|---|
| < 100 | Healthy | Safe to proceed |
| 100–399 | Buffer is elevated — not a problem yet, but WaitReplication will push more tasks in and could tip it over the limit mid-flight | Resolve the backlog before starting. Check standby history pod CPU and database latency — a slow standby is the most common cause. |
| 400–499 | Close to the limit | Do not start — the buffer will almost certainly hit its limit once WaitReplication begins pushing tasks. |
| ≥ 500 | Limit reached — the active has already paused sending | Hard blocker. The handover workflow will not progress past Step 3 in this state. Resolve the backlog first. |

> If backlog depth is 400 or above, do not proceed. Resolve the backlog first, then continue to section 1.6. If it is between 100–399, resolve it before starting the handover workflow — the buffer will likely tip over once WaitReplication begins pushing tasks.

> If backlog depth is below 100, continue to section 1.6.

#### 1.6. Are there workflow executions on the standby that will be stuck after the flip?

**Dashboard:** [Standby Task Discards panel](../metrics/dashboards/server/namespace-failover-graceful-handover-readme.md#panel-standby-task-discards-stat) (Row 1, standby cluster) · Alert `FAILOVER-PRE-06`

When the standby receives a replication task it waits for the matching history events from the active before it can apply it. If those events never arrive within the timeout (default 15 minutes), the standby gives up and discards the task. That workflow execution is now in a broken state on the standby — when the flip happens and it becomes the active, that workflow will have no pending task driving it forward. It is effectively stuck and will not make progress until it is manually recovered using `RefreshWorkflowTasks`. This applies regardless of what type of task was discarded — workflow tasks, activity tasks, timers, and child workflow tasks are all regenerated by `RefreshWorkflowTasks`.

This should be zero under normal replication. A non-zero rate here means real workflows will need manual recovery after the flip — that is direct business impact.

| What you see | What to do |
|---|---|
| Zero | Safe to proceed |
| Any non-zero rate | Blocker. Identify the affected workflows (see below), understand the scope, and decide whether to proceed knowing those workflows will need recovery after the flip. Alert `FAILOVER-PRE-06` fires after 2 minutes of non-zero rate. |

**Identifying the affected workflows**

Only do this if the Standby Task Discards panel is showing a non-zero rate. The panel shows a rate but no workflow IDs — to find the specific workflows, run this Loki query against the **standby** history service:

```logql
{cluster="<standby-cluster>", k8s_component="history"}
  |= `"level":"warn"`
  |~ "Discarding standby \\w+ task due to task being pending for too long."
  | json
  | line_format "WorkflowKey: {{ .queue_task}}"
```

The output contains `NamespaceID`, `WorkflowID`, and `RunID` for each discarded task. To filter to a specific workflow, add `|= "my-workflow-id"`. To extract `WorkflowID` as a structured label, add `| regexp "WorkflowID:(?P<wfid>[^ }]+)"`.

> The `cluster` label value in Loki may be a numeric ID or a string name depending on your log shipper. Check a sample log line first to confirm the correct value.

> If this panel is non-zero, our recommendation is not to proceed until you have assessed the scope and accepted the recovery cost.

> If all six pre-flight checks are clear, you are ready to start the handover workflow. Go to [Starting the Handover Workflow](#starting-the-handover-workflow) to run it, then return to section 2a to monitor progress.

---

## 2a. Handover workflow started — monitor standby catchup (Steps 1–3)

The handover workflow is now running. During this phase — Steps 1 through 3 — the workflow is waiting for the standby to fully catch up with the active before it attempts the flip. Your namespace is serving traffic normally and your clients and workers see nothing different. This phase can take anywhere from seconds to several minutes depending on how much lag remained when you started.

**Watch [Row 2a — WaitReplication](../metrics/dashboards/server/namespace-failover-graceful-handover-readme.md#row-2a--waitreplication-steps-13--no-client-impact) in the dashboard.**

The key panel is the [Catchup Progress % gauge](../metrics/dashboards/server/namespace-failover-graceful-handover-readme.md#panel-catchup-progress--gauge) — it shows what percentage of shards have caught up within your `AllowedLaggingSeconds` threshold. You want this to climb steadily toward 100%. The [Handover Progress — Catchup + Drain Combined](../metrics/dashboards/server/namespace-failover-graceful-handover-readme.md#panel-handover-progress--catchup--drain-combined-time-series) panel shows both catchup and drain progress together — during this phase only catchup rises; drain stays at 0 until the flip begins.

> The catchup gauge does not filter by namespace. If you have multiple global namespaces replicating, it reflects all of them combined — not just the one being handed over. A server-side fix to add namespace filtering is pending.

**What normal looks like:** catchup progress climbs steadily. The speed is limited by `history.ReplicationStreamSenderHighPriorityQPS` (default 100 tasks/s per stream) — if you have a lot of lag this will take time. If you raised this value before starting, you should see faster progress.

**What to watch for:**

| Signal | What it means | What to do |
|---|---|---|
| Catchup progress rising steadily | Normal — standby is catching up | Wait for it to reach 100% |
| Catchup progress stalled for more than 2 minutes and lag is flat | Stream may be stuck | Check the [Stream Stuck panel](../metrics/dashboards/server/namespace-failover-graceful-handover-readme.md#panel-stream-stuck-stat) in Row 1 — a non-zero rate confirms it. See section [5](#5-something-went-wrong--rollback-and-recovery) for recovery steps. |
| Catchup reaches 100% | Standby is caught up — the workflow will automatically move to the drain window | Watch Row 2b next |

**Log messages you may see during this phase:**

| Message | Where | What it means |
|---|---|---|
| `"Wait catchup not ready"` (Info, every second) | Active (worker service) | Normal — the workflow is polling. Includes `ReadyShards`, `NotReadyShards`, `AllowedLagging`, `AllowedLaggingTasks`, `MaxLaggingTasks`, `MaxTimeLag`. No action needed. |
| `"GetReplicationStatus response missing expected remote cluster for shard during replication catchup"` (Info) | Active (worker service) | Transient and normal. If it persists beyond 30 seconds, verify the remote cluster name in your server config matches — run `temporal operator namespace describe`. |
| `"High replication is not making progress"` or `"Low replication is not making progress"` (Error) | Standby (history service) | The stream is stuck and will not recover on its own. Investigate immediately — see section [5](#5-something-went-wrong--rollback-and-recovery). |

---

## 2b. Namespace is in HANDOVER — monitor the drain window (Steps 4–5)

Step 4 put the namespace into HANDOVER state. The active cluster's frontend is now returning `Unavailable` for write APIs on this namespace — other namespaces on the same cluster are unaffected. The 30-second clock is running. The workflow is waiting for all shards to confirm they have applied every replication task up to the snapshot taken in Step 2. When they do, Step 6 fires and the flip completes. If they do not all confirm within `HandoverTimeoutSeconds`, the workflow rolls back automatically and the namespace returns to normal — no flip occurs.

**Watch [Row 2b — HANDOVER Drain](../metrics/dashboards/server/namespace-failover-graceful-handover-readme.md#row-2b--handover-drain-steps-45) in the dashboard.**

| What you see | What it means | What to do |
|---|---|---|
| [Handover Drain Progress %](../metrics/dashboards/server/namespace-failover-graceful-handover-readme.md#panel-handover-drain-progress--gauge) rising toward 100% | Shards are confirming catchup — drain is proceeding normally | Wait for it to reach 100% |
| [UNAVAILABLE Error Burst](../metrics/dashboards/server/namespace-failover-graceful-handover-readme.md#panel-unavailable-error-burst-expected-time-series) spiking | Expected — this is the active cluster returning `Unavailable` during HANDOVER. The SDK retries automatically. | Normal — watch for it to clear once drain completes |
| Drain Progress stalls and then drops to 0 | Drain timed out — workflow is rolling back | See rollback behaviour below |
| `Unavailable` errors persist after drain completes | HANDOVER state has not fully cleared from the shard cache | Wait one more 2-second cache cycle. If still non-zero after ~5s, check history pod logs. |

**What your clients see during HANDOVER**

Write calls to the active cluster return `Unavailable` during the drain window. What happens next depends on `system.enableNamespaceHandoverWait`:

- **`false` (default):** the active returns `Unavailable` immediately. The SDK retries with backoff. If the drain finishes before retries exhaust, the retry succeeds and nothing is lost.
- **`true` (recommended, set on both clusters):** the active holds the connection open and waits for HANDOVER state to exit. If the drain completes in time, the original call succeeds — the client never sees an error. If it does not complete in time, the server returns `Unavailable` and the SDK retries from that point.

Neither setting causes workflow starts or signals to be lost on its own — loss requires the SDK to exhaust its full retry budget, which does not happen on a drain that completes in seconds. Setting `true` reduces the chance the client sees any error at all.

Once the flip completes, `Unavailable` clears. What clients hitting the **old active** see after that depends on forwarding policy:
- **`dcRedirectionPolicy=all-apis-forwarding` on old active:** write calls are forwarded to the new active transparently — no errors.
- **Without `all-apis-forwarding`:** write calls return `NamespaceNotActive`. The SDK does not retry this automatically — clients must be re-pointed to the new active. Watch `FAILOVER-POST-01` and the Forwarding Error Rate panel in Row 4.

**What rollback looks like**

Drain Progress stops climbing and drops back to 0. The namespace returns to `REPLICATION_STATE_NORMAL`. Write operations resume on the active. No flip occurred — no recovery steps are needed.

Rollback happens when the drain window expires before all shards confirm catchup. Common reasons:

- Too much lag remained when HANDOVER started — more tasks in flight than the 30-second window could drain. Go back to section 1.3 and wait for lag to be lower before re-running.
- The receiver backlog on the standby hit its limit mid-drain and the sender paused. Check the Receiver Backlog Depth panel — if it was elevated going in, that is likely the cause. See section 1.5.
- `history.EnableReplicationTaskTieredProcessing` is not `true` — a LOW priority backlog predating the snapshot prevents drain from completing. Verify this is set on both clusters before re-running.
- The migration worker pod was restarted during the drain window — the `WaitHandover` activity heartbeat timed out. Re-run once the pod is stable.

After addressing the root cause, go back to section 1 and repeat all pre-flight checks before re-running the handover workflow. Skipping straight to re-running risks ending up in the same situation.

**Log messages you may see during this phase** (all from active worker service):

| Message | What it means |
|---|---|
| `"Wait handover not ready"` (Info, every second) | Normal — the workflow is polling. Includes `ReadyShards`, `NotReadyShards`, `MaxHandoverLag`, `MaxHandoverLagShardID`. No action needed. |
| `"Wait handover missing handover namespace info"` (Info) | Transient — shard cache has not refreshed yet. Expected in the first 1–2 seconds and clears on its own. If it persists, the namespace cache is not propagating HANDOVER state — investigate. |
| `"GetReplicationStatus response missing expected remote cluster for shard during handover"` (Info) | The remote cluster config is missing for a shard. The activity retries automatically — this does not cause immediate rollback. If it persists until `HandoverTimeoutSeconds` expires without resolving, the workflow will roll back. Verify the remote cluster name in your server config is correct. |

> The drain window is at most 30 seconds. Do not restart or redeploy the migration worker pod during this time — if the pod goes away, the handover aborts and you will need to re-run from scratch.

---

## 3. Flip completed — confirm the namespace switched correctly (Step 6)

Step 6 ran `UpdateActiveCluster` — the namespace is now active on the new cluster. Watch [Row 3 — Flip Confirmed](../metrics/dashboards/server/namespace-failover-graceful-handover-readme.md#row-3--flip-confirmed-step-6) in the dashboard. All four signals should change in quick succession.

| Signal | What to look for |
|---|---|
| [`handover_ready_shard_count`](../metrics/dashboards/server/namespace-failover-graceful-handover-readme.md#panel-handover-state-exited-stat) drops to 0 | HANDOVER state has exited — the flip is done |
| [`client_redirection_errors{error_type="Unavailable"}`](../metrics/dashboards/server/namespace-failover-graceful-handover-readme.md#panel-unavailable-errors-gone-stat) on old active returns to 0 | The write block is lifted — clients are no longer being rejected |
| [`client_redirection_requests`](../metrics/dashboards/server/namespace-failover-graceful-handover-readme.md#panel-forwarding-active-stat) on old active starts rising | The old active is now forwarding write calls to the new active |
| [`task_errors_version_mismatch`](../metrics/dashboards/server/namespace-failover-graceful-handover-readme.md#panel-version-fence-fired-stat) on old active may spike | Expected — see below. Zero is also a valid outcome. |

**Why `task_errors_version_mismatch` may spike — and why zero is fine too**

Each task the old active created is stamped with the old cluster's failover version. After the flip, the old active's executor picks up those tasks, sees the version no longer matches, and drops them. This is by design — it prevents the old cluster from executing tasks that now belong to the new active (which would cause duplicate activity executions or double-fired timers). The spike is safe and expected.

If the counter is 0, it means WaitReplication brought the standby so close to caught up that there were no pending tasks left on the old active at the moment of flip. This is the ideal outcome of a well-prepared handover — not a problem.

**Always confirm the flip using `temporal operator namespace describe`** — do not rely on the workflow showing as completed. The workflow can complete successfully even if the namespace was already pointing at the target cluster before it ran, so a completed status alone does not tell you whether a flip actually occurred. The only way to be sure is to check the namespace directly. Run this against both cluster frontends:

```bash
temporal operator namespace describe --namespace <your-namespace> --address <cluster-address>
```

Expected output on both clusters once propagated:

```
Replication Config:
  Active Cluster:    <new-active-cluster-name>
Replication State:   Normal
```

If `Active Cluster` still shows the old cluster name, the namespace cache has not refreshed yet — wait up to 2 seconds and re-run.

If `Replication State` shows `Handover` briefly after the workflow completed, that is normal — the workflow always resets the namespace back to `Normal` as its final action, but cache propagation takes up to 2 seconds. Re-run the command after waiting. You are not stuck.

If `Replication State` shows `Handover` and does not clear after 5–10 seconds, the reset did not propagate cleanly. Check the workflow execution history to see whether Step 6 (`UpdateActiveCluster`) completed — if it did, the flip happened and the state will resolve on its own. If it did not complete, the flip did not occur and the namespace will return to `Normal` automatically via the workflow's built-in rollback — no manual intervention needed.

If the two clusters disagree — one shows the new active cluster as `Active Cluster` and the other still shows the old active cluster — the namespace is in a passive-passive state. The handover completed successfully, but the namespace metadata change did not replicate to the other cluster. The namespace is live on the new active — this does not require re-running the handover. **Resolve this immediately** — in a passive-passive state both clusters treat the namespace as standby, so workers get empty polls on both sides and workflow execution stalls. See [passive-passive recovery](#the-handover-completed-but-the-namespace-is-in-a-passive-passive-state) in section 5 to resync the standby's view.

---

## 4. Monitor the new active cluster after the handover

The namespace is now active on the new cluster and the handover workflow has completed. Watch [Row 4 — Post-Handover Health](../metrics/dashboards/server/namespace-failover-graceful-handover-readme.md#row-4--post-handover-health) in the dashboard for 5–10 minutes. The first minute or two may be noisy — queue processors are activating, workers are reconnecting, and the reverse replication stream is establishing. Give signals time to settle before acting on them.

**What normal looks like in the first few minutes:**

- If the old active is running `dcRedirectionPolicy=all-apis-forwarding`, workers that were polling the old active will have their Respond calls forwarded to the new active — you will see [Forwarding Success Rate](../metrics/dashboards/server/namespace-failover-graceful-handover-readme.md#panel-forwarding-success-rate-time-series) on the old active rise and then gradually decline as workers are re-pointed. If you are running `selected-apis-forwarding`, Respond calls are not in the forwarding whitelist — workers must already be pointed at the new active or those calls will fail.
- [WFT Schedule-to-Start](../metrics/dashboards/server/namespace-failover-graceful-handover-readme.md#panel-wft-schedule-to-start-new-active-time-series) on the new active may spike briefly while its queue processors warm up — this is expected and should normalize on its own.
- Some workflows may take up to 5 seconds longer to get their first task picked up after the flip. This is because workers that were running on the old active had a dedicated fast-path (sticky queue) for picking up tasks. After the flip that fast-path no longer exists on the new active, so the server waits 5 seconds (the default sticky timeout) before falling back to the normal queue. Once that happens the workflow resumes normally — this is not a stuck state.
- The old active (now standby) will show timer queue lag of approximately `history.standbyClusterDelay` (default 5 minutes) — this is normal standby behavior.

**What to watch for:**

| Panel | Cluster | What it means if something looks wrong |
|---|---|---|
| [Version Mismatch Decay](../metrics/dashboards/server/namespace-failover-graceful-handover-readme.md#panel-version-mismatch-decay-time-series) | Old active | The old active is working through its backlog of pre-flip tasks — each one is version-checked, found to belong to the new active, and safely dropped. This should trend toward 0 as the backlog drains. If it stays non-zero for more than a few minutes, a queue processor on the old active may be stuck — check history pod logs. |
| [Forwarding Error Rate by Type](../metrics/dashboards/server/namespace-failover-graceful-handover-readme.md#panel-forwarding-error-rate-by-type-time-series) | Old active | Should be 0. `Unavailable` = new active unreachable. `ResourceExhausted` = new active throttling, raise `frontend.namespaceRPS`. `FailedPrecondition` = workers completing tasks on old active without forwarding — see dynamic config section below. |
| [WFT Schedule-to-Start](../metrics/dashboards/server/namespace-failover-graceful-handover-readme.md#panel-wft-schedule-to-start-new-active-time-series) | New active | Still elevated after 2 minutes = workers have not reconnected to the new active yet. Check that workers are polling the new active. |
| [Replication Lag — Reverse Stream](../metrics/dashboards/server/namespace-failover-graceful-handover-readme.md#panel-replication-lag--reverse-stream-time-series) | New active | Rising lag = old active (now standby) cannot keep up with replication from the new active. |
| [Reverse Replication Active](../metrics/dashboards/server/namespace-failover-graceful-handover-readme.md#panel-reverse-replication-active-time-series) | Old active (now standby) | Should go non-zero shortly after the flip — the old active recognises it is now the standby once its namespace cache refreshes (default 2s). If it stays flat, the reverse stream has not established — check [Stream Stuck](../metrics/dashboards/server/namespace-failover-graceful-handover-readme.md#panel-stream-stuck-stat) on the old active. |

#### Check for stuck workflows and recover them

If all pre-flight checks were clean — in particular if the Standby Task Discards panel in section 1.6 was zero — stuck workflows are very unlikely after a graceful handover. If section 1.6 was non-zero and you proceeded anyway, this is where those workflows will surface.

The [WFT Schedule-to-Start](../metrics/dashboards/server/namespace-failover-graceful-handover-readme.md#panel-wft-schedule-to-start-new-active-time-series) panel tells you whether tasks are queued but workers are slow to pick them up. It does not detect workflows whose tasks were never re-scheduled on the new active at all — those are invisible to this metric.

To find workflows that are genuinely stuck, run this Loki query against the **old active (now standby)** history service, scoped to the window just before and during the flip:

```logql
{cluster="<old-active-cluster>", k8s_component="history"}
  |= `"level":"warn"`
  |~ "Discarding standby \\w+ task due to task being pending for too long."
  | json
  | line_format "WorkflowKey: {{ .queue_task}}"
```

The output contains `NamespaceID`, `WorkflowID`, and `RunID` per discarded task. To filter to a specific namespace add `|= "<namespace-id>"`. To extract `WorkflowID` as a label add `| regexp "WorkflowID:(?P<wfid>[^ }]+)"`.

If results appear, verify those workflows on the new active before recovering:

```bash
temporal workflow describe --namespace <ns> --workflow-id <wfid>
```

If a workflow is stuck — no pending task, not making progress — recover it with `RefreshWorkflowTasks`. This re-schedules all pending tasks (workflow tasks, activity tasks, timers) for that execution. No history is lost.

```bash
# Single workflow
tctl admin workflow refresh-workflow-tasks --namespace <ns> --workflow-id <wfid> --run-id <runid>

# All open workflows in the namespace
tctl admin workflow refresh-workflow-tasks --namespace <ns>
```

#### Dynamic config changes after the flip

Now that the flip has happened, the old active is serving as standby and forwarding write calls to the new active. You need to decide when to tighten forwarding. **Order matters** — workers do two things: they poll for tasks, and they respond when done (completing a workflow task, finishing an activity, etc). As long as a worker is still polling the old active, it may pick up a task and try to respond on the old active too. With `all-apis-forwarding` active, that respond call is forwarded to the new active transparently. If you tighten forwarding before all workers are re-pointed, those respond calls stop being forwarded and the old active rejects them with an error the SDK cannot retry — the task fails.

1. **Verify all workers are fully re-pointed to the new active first.** Workers completing tasks or sending heartbeats through the old active depend on `all-apis-forwarding` being active. If you tighten forwarding before workers are re-pointed, those calls will fail with a non-retryable error.
2. **Set `system.forceNamespaceSelectedAPIAutoForwarding=true` on the old active** (scoped to the namespace, no restart). This switches the old active from forwarding all APIs including polls, to forwarding only the 11 write APIs in the whitelist. Workers will no longer be able to poll through the old active — they must be pointed at the new active directly.
3. **Once no forwarding is needed at all**, set `system.enableNamespaceNotActiveAutoForwarding=false` on the old active to disable forwarding entirely.

Here is a summary of the config changes to make and when:

| Config | Cluster | When to apply |
|---|---|---|
| `system.forceNamespaceSelectedAPIAutoForwarding` → `true` (namespace-scoped) | Old active | After all workers are re-pointed to new active |
| `system.enableNamespaceNotActiveAutoForwarding` → `false` | Old active | After forwarding is no longer needed at all |
| `dcRedirectionPolicy` → `"selected-apis-forwarding"` | Old active | On next restart, once polls are no longer needed through old active |
| `frontend.namespaceRPS` → raise | New active | If `ResourceExhausted` errors appear on Forwarding Error Rate panel |

> If you set `system.forceNamespaceSelectedAPIAutoForwarding=true` too early and workers are still polling the old active, you will see `FailedPrecondition` errors on the [Forwarding Error Rate by Type panel](../metrics/dashboards/server/namespace-failover-graceful-handover-readme.md#panel-forwarding-error-rate-by-type-time-series). Alert `FAILOVER-POST-01` detects this. To recover: set `system.forceNamespaceSelectedAPIAutoForwarding=false` immediately, wait 2 seconds for propagation, and verify the errors clear.

> If all signals in Row 4 look healthy and no stuck workflows were found, the handover is complete. If anything went wrong at any point during the handover — the workflow rolled back, the drain timed out, or you need to force a failover — see section 5 below.

---

## 5. Something went wrong — rollback and recovery

This section is a reference for situations where something did not go as expected. Each scenario below describes what you will see, why it happened, and what to do. Earlier sections in this playbook point here when they detect a problem.

In most cases: **the namespace is safe**. The handover workflow is designed so that if anything goes wrong before the flip completes, the namespace is automatically returned to its original state. No flip means no data loss and no stuck active cluster. There is one rare exception — see [passive-passive state](#the-handover-completed-but-the-namespace-is-in-a-passive-passive-state) below.

---

#### The handover workflow rolled back — no flip occurred

**What you see:** [Handover Drain Progress %](../metrics/dashboards/server/namespace-failover-graceful-handover-readme.md#panel-handover-drain-progress--gauge) was climbing during section 2b, then dropped back to 0. `Unavailable` errors cleared. Alert `FAILOVER-HANDOVER-01` may have fired. The namespace is back on the original active cluster.

**Why it happened:** the drain window expired before all shards confirmed catchup. The workflow rolled back automatically — no operator action was needed and no flip occurred.

**Common causes:**

| Cause | Where to check |
|---|---|
| Too much replication lag when HANDOVER started | Go back to section [1.3](#13-is-replication-lag-low-enough-to-proceed) — wait for lag to be near-zero before re-running |
| Receiver backlog hit its limit mid-drain | Check [Receiver Backlog Depth](../metrics/dashboards/server/namespace-failover-graceful-handover-readme.md#panel-receiver-backlog-depth-stat) — see section [1.5](#15-is-the-standby-keeping-up-with-incoming-replication-tasks) |
| `history.EnableReplicationTaskTieredProcessing` is not `true` | Verify on both clusters — a LOW priority backlog predating the snapshot prevents drain from completing |
| Migration worker pod restarted during the drain window | Re-run once the pod is stable |

**What to do:** go back to section 1 and repeat all pre-flight checks before re-running the handover workflow.

---

#### The standby is not catching up — WaitReplication is stalled

**What you see:** you started the handover workflow and are in section 2a. [Catchup Progress %](../metrics/dashboards/server/namespace-failover-graceful-handover-readme.md#panel-catchup-progress--gauge) has not moved for several minutes. The namespace is still serving traffic normally — no client impact yet.

**Why it happened:** the replication stream between the clusters is not delivering tasks fast enough, or has stopped entirely.

**What to do — check in this order:**

1. **Check [Stream Stuck](../metrics/dashboards/server/namespace-failover-graceful-handover-readme.md#panel-stream-stuck-stat)** — a non-zero rate means one or more streams have stopped. A stuck stream will reconnect automatically after `history.ReplicationStreamSendEmptyTaskDuration` × `history.ReplicationReceiverLivenessMultiplier` (defaults: 1 min × 3 = **3 minutes**). Wait and watch — if the panel clears on its own, WaitReplication will resume. If it does not clear, restart the affected standby history pods (check history pod logs to find which pod is serving the stuck shard first).
2. **Check [Receiver Backlog Depth](../metrics/dashboards/server/namespace-failover-graceful-handover-readme.md#panel-receiver-backlog-depth-stat) p99** — if it is near 500, the standby receiver is saturated and the sender has paused. See section [1.5](#15-is-the-standby-keeping-up-with-incoming-replication-tasks) for how to resolve it.
3. **Check [Stream Errors (gRPC)](../metrics/dashboards/server/namespace-failover-graceful-handover-readme.md#panel-stream-errors--grpc-stat) and [Stream Service Errors](../metrics/dashboards/server/namespace-failover-graceful-handover-readme.md#panel-stream-service-errors-stat)** — look for `"ReplicationStreamError"` in standby history pod logs for the specific cause.

**Important:** WaitReplication has a 1-hour timeout. If it expires, the namespace is never put into HANDOVER state — no write unavailability, no flip. It is safe to let it expire or cancel it manually and re-run. To cancel without waiting:

```bash
temporal workflow cancel --namespace temporal-system --workflow-id handover-<your-namespace>
```

Before re-running, confirm the namespace is still in NORMAL state:

```bash
temporal operator namespace describe --namespace <your-namespace> --address <active-cluster-address>
```

Then go back to section 1 and repeat pre-flight checks.

---

#### You need to force a failover — the active cluster is unreachable

This is not a graceful handover scenario. Only do this if the active cluster is down and unreachable and you cannot wait for it to recover.

```bash
# Run against the STANDBY cluster's frontend
temporal operator namespace update --namespace <ns> --active-cluster <standby-cluster-name>
```

After a forced failover, some workflow executions may be stuck — tasks that were in-flight at the time of failure may not have been replicated to the standby. See section [4 — Check for stuck workflows and recover them](#check-for-stuck-workflows-and-recover-them) for how to identify and recover them.

```bash
tctl admin workflow refresh-workflow-tasks --namespace <ns>
```

---

#### The handover completed but the namespace is in a passive-passive state

**What you see:** the handover workflow completed successfully, but when you run `temporal operator namespace describe` against both clusters, the two disagree — one shows the new active cluster as `Active Cluster` and the other still shows the old active cluster. Neither cluster is treating itself as active.

**Why it happened:** namespace updates and namespace replication are not atomic. The handover workflow completed Step 6 (`UpdateActiveCluster`) on the active cluster, but the replication of that change to the standby failed at that moment — leaving the standby unaware of the flip.

**Impact:** both clusters treat the namespace as standby. Worker polls return empty on both sides, write APIs bounce between clusters or return errors, and workflow execution stalls. Resolve this immediately — the fix is a single command.

**How to recover:** run the following against the **new active cluster's** frontend to explicitly set the active cluster:

```bash
temporal operator namespace update \
  --namespace <namespace> \
  --active-cluster <new-active-cluster-name> \
  --address <new-active-cluster-frontend>
```

After running this, verify both clusters agree by re-running `temporal operator namespace describe` on both frontends. Both should show the same `Active Cluster` value.
