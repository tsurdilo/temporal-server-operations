# Temporal Namespace Failover — Graceful Handover Playbook

> **Scope.** This playbook covers **planned graceful handover only** using the `namespace-handover-v2` workflow. Both clusters must be reachable and healthy before starting.

## Before You Start

- [Recommendations](#recommendations)
- [Dashboard and workflow steps](#dashboard-and-workflow-steps)
- [What your clients and workers experience](#what-your-clients-and-workers-experience)
- [Setting dcRedirectionPolicy](#setting-dcredirectionpolicy)
- [Forwarding policy reference](#forwarding-policy-reference)
- [API forwarding to the active cluster: cluster policy and per-namespace settings](#api-forwarding-to-the-active-cluster-cluster-policy-and-per-namespace-settings)
- [Dynamic config](#dynamic-config)
- [Alerts](#alerts)
- [Temporal Schedules](#temporal-schedules)
- [Workflow Deletion](#workflow-deletion)
- [Namespace Deletion](#namespace-deletion)
- [Search Attributes](#search-attributes)
- [Archival](#archival)
- [Starting the Handover Workflow](#starting-the-handover-workflow)

---

### Recommendations

Quick reference — all recommendations below are explained in detail in the sections that follow.

- Think carefully about `dcRedirectionPolicy` — it is a static YAML setting that requires a restart and controls how each cluster handles traffic when it is not the active. Get it right before the first handover. See [Setting dcRedirectionPolicy](#setting-dcredirectionpolicy).
- Verify `numHistoryShards` is a clean multiple between clusters (e.g. 512 and 1024 is valid; 512 and 768 is not). A non-multiple ratio causes the replication stream to fail at connection time with no recovery path — this must be correct when replication is first configured.
- Never run two handover workflows for the same namespace at the same time.
- Set `history.EnableReplicationTaskTieredProcessing` to the **same value on both clusters**. Dynamic config, no restart needed; both clusters must be on v1.25.0 or later. A mismatch makes the replication stream restart over and over. Setting it to `true` does three things:
  - **Send speed:** `true` caps how fast each stream sends, at `history.ReplicationStreamSenderHighPriorityQPS`, default 100 tasks per second per stream.
  - **Pause:** `true` lets the standby pause the active when the standby falls behind.
  - **Force-replication backlog:** `true` stops a force-replication backlog of closed workflows from holding up the handover. This only works when `history.enableTransitionHistory` is `false`, which is the default up to v1.29.x. From v1.30.0 that setting is `true` by default, and a force-replication backlog holds up the handover whether tiered processing is on or off. Let force-replication finish before you start the handover.

  See [Dynamic config](#dynamic-config).
- Set `system.enableNamespaceHandoverWait: true` on **both clusters**. Dynamic config, no restart needed. Eliminates the `Unavailable` error during the HANDOVER drain window — clients see latency instead.
- Confirm `system.enableNamespaceNotActiveAutoForwarding` is not set to `false` on either cluster, for this namespace or for all namespaces. The setting is namespace-scoped and defaults to `true`. If it is `false` for a namespace, nothing is forwarded for that namespace regardless of `dcRedirectionPolicy`. A value of `true` does not turn forwarding on by itself, because forwarding also needs `dcRedirectionPolicy` to be set. See [Setting dcRedirectionPolicy](#setting-dcredirectionpolicy) and [API forwarding to the active cluster: cluster policy and per-namespace settings](#api-forwarding-to-the-active-cluster-cluster-policy-and-per-namespace-settings).
- Do not lower `history.standbyTaskMissingEventsDiscardDelay` below its default (15m) on the standby — lower values increase the risk of workflows getting stuck post-flip.
- Before every handover, including one that hands a namespace back to the cluster it came from, check [Replication DLQ Writes](../observability/dashboards/server/namespace-failover-graceful-handover-readme.md#panel-replication-dlq-writes-stat). If it is non-zero, list the standby's replication DLQ and merge it first. An update left in that DLQ means the standby's copy of a workflow is out of date, and after the flip that copy is the one that runs. See [section 1.2](#12-is-the-replication-stream-between-clusters-healthy).
- If a namespace was a local namespace before it was made global, and still has workflows that were started before then, do not raise `frontend.keepAliveMaxConnectionAge` (default 5 minutes) on either cluster, and do not lower `history.ReplicationExecutableTaskErrorRetryMaxAttempts` (default 80) or `history.ReplicationExecutableTaskErrorRetryExpiration` (default 10 minutes). A longer reconnect interval or a lower retry limit makes it more likely that replication updates for those workflows end up in the old active cluster's replication DLQ after a flip. See [Replication errors on the old active cluster after the flip](#replication-errors-on-the-old-active-cluster-after-the-flip).
- Use `AllowedLaggingSeconds: 10`, `AllowedLaggingTasks: 500`, `HandoverTimeoutSeconds: 30` as starting values. If WaitReplication stalls, check Replication Latency p99 first. A larger `AllowedLaggingTasks` does not help while the standby has not yet reached the point recorded when the handover workflow started. See [section 1.3](#13-is-replication-lag-low-enough-to-proceed).
- Use a consistent workflow ID convention — `handover-<namespace>` — so running handovers are easy to spot before starting a new one.
- Tune history task processing on **both clusters**, sized for the load each would carry if its namespaces became active. A cluster holding only passive copies looks cheap to run, so its limits are often left low or unset — and then throttle the workload the moment it becomes active. See [History Task Processing — Tuning and Troubleshooting](./history-task-processing-tuning.md#63-if-you-run-more-than-one-cluster).
- Use `system.forceNamespaceSelectedAPIAutoForwarding` (namespace-scoped dynamic config, no restart) to control which cluster's workers actively participate in workflow execution before and after a handover — see [API forwarding to the active cluster: cluster policy and per-namespace settings](#api-forwarding-to-the-active-cluster-cluster-policy-and-per-namespace-settings).
- If you use Schedules, confirm `worker.enableScheduler` is `true` (or not explicitly set to `false`) on the cluster you are failing over to — if `false`, schedules will not fire after the handover. See [Temporal Schedules — Pre-handover](#pre-handover).
- If you use Schedules, ensure each schedule's `catchup_window` (a per-schedule spec field, default 365 days) is longer than the expected handover duration — fires outside that window are not backfilled after the handover.

### Dashboard and workflow steps

**Dashboard:** [Temporal — Namespace Failover: Graceful Handover](../observability/dashboards/server/namespace-failover-graceful-handover-readme.md) — v1.7.0

| Row | What you are watching for |
|---|---|
| **1. Pre-Flight** | Is the replication stream healthy and caught up? Nothing should be red before you start. |
| **2a. WaitReplication** | Is the standby catching up? Progress rises steadily with no client impact. |
| **2b. HANDOVER Drain** | Is the drain completing in time? Writes are briefly blocked — this window is at most 30 seconds. |
| **3. Flip Confirmed** | Did the flip happen cleanly? UNAVAILABLE errors on the old active return to 0. Confirm the flip itself with `temporal operator namespace describe` on both clusters. |
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

> **There is no dynamic config equivalent for upgrading to `all-apis-forwarding`.** The only runtime knob is `system.forceNamespaceSelectedAPIAutoForwarding`, which can downgrade a cluster running `all-apis-forwarding` to whitelist-only for a specific namespace — but it cannot upgrade `selected-apis-forwarding` to `all-apis-forwarding`. The base policy must be set in YAML. See [API forwarding to the active cluster: cluster policy and per-namespace settings](#api-forwarding-to-the-active-cluster-cluster-policy-and-per-namespace-settings) for the full breakdown.

### Forwarding policy reference

What each policy forwards — and what it does not:

| Operation | `all-apis-forwarding` | `selected-apis-forwarding` |
|---|---|---|
| Starting and signalling workflows | ✓ | ✓ |
| Worker polls — picking up workflow and activity tasks | ✓ | ✗ |
| Workers completing tasks (`RespondWorkflowTaskCompleted`, `RespondActivityTask*`) | ✓ | ✗ |
| Activity heartbeats (`RecordActivityTaskHeartbeat`) | ✓ | ✗ |
| `UpdateWorkflowExecution`, `ExecuteMultiOperation` | ✓ | ✗ Not in the whitelist in any release up to v1.32.1 (see the note below the whitelist) |

**The `selected-apis-forwarding` whitelist, by release.** A cluster forwards an API in this list if its release is the one shown or later.

| API | Forwarded from |
|---|---|
| `StartWorkflowExecution`, `SignalWithStartWorkflowExecution`, `SignalWorkflowExecution`, `RequestCancelWorkflowExecution`, `TerminateWorkflowExecution`, `QueryWorkflow` | v1.20.0 or earlier |
| `DeleteWorkflowExecution`, `StartActivityExecution`, `RequestCancelActivityExecution`, `TerminateActivityExecution`, `DeleteActivityExecution` | v1.31.0 |
| `PauseActivityExecution`, `UnpauseActivityExecution`, `ResetActivityExecution`, `UpdateActivityExecutionOptions`, `StartNexusOperationExecution`, `RequestCancelNexusOperationExecution`, `TerminateNexusOperationExecution`, `DeleteNexusOperationExecution` | v1.32.0 |

The whitelist has 6 APIs up to v1.30.x, 11 on v1.31.x and 19 on v1.32.x. Polls, `Respond*` calls and heartbeats are not in the whitelist in any release, so `selected-apis-forwarding` never lets workers on the standby pick up work.

Everything else — polls, Respond*, heartbeats, Update, ExecuteMultiOperation — is not forwarded and will be served locally by the standby (which means no work for workers, and errors for anything that requires the active).

**Batch operations are not in the whitelist.** `StartBatchOperation` — which covers every batch job (batch delete, terminate, signal, cancel, reset) — is not whitelisted. So under `selected-apis-forwarding` (and the default no-forwarding setup) a batch command sent to the standby is not forwarded and fails with `NamespaceNotActive`. Only `all-apis-forwarding` forwards it. Either way, the safe practice is to run batch operations against the active cluster. This matters for cleanup jobs like batch delete — see [Workflow Deletion](#workflow-deletion).

> `UpdateWorkflowExecution` and `ExecuteMultiOperation` are not in the whitelist in any release up to and including v1.32.1. They have been added in the server's development branch, but no release includes that change yet. Until then, Update calls to a cluster running `selected-apis-forwarding` while it is not the active will fail with `NamespaceNotActive`.

`system.forceNamespaceSelectedAPIAutoForwarding` is only meaningful if the static policy is `all-apis-forwarding`. Setting `all-apis-forwarding` in static config once gives you the flexibility to downgrade per-namespace at runtime without ever needing another restart.

**Using `system.forceNamespaceSelectedAPIAutoForwarding` to control worker participation across a handover — without restarting clusters:**

This requires both clusters to have `dcRedirectionPolicy: all-apis-forwarding` set in static config (one-time restart). After that, the dynamic config becomes the only knob you need.

*Before handover:*
- **Active cluster** — leave `system.forceNamespaceSelectedAPIAutoForwarding: false` (default). Full forwarding. Workers on the active poll and complete tasks normally.
- **Standby cluster** — set `system.forceNamespaceSelectedAPIAutoForwarding: true` (namespace-scoped). Downgrades to whitelist-only. Workers deployed on the standby do not receive polls and cannot complete tasks — they are effectively idle. Only relevant if you have workers pointed at both clusters; if workers only point at the active, this is a no-op.

*After handover (flip):*
- **New active (was standby)** — set `system.forceNamespaceSelectedAPIAutoForwarding: false`. Full forwarding restored. Workers on the new active start picking up tasks immediately.
- **Old active (now standby)** — do not set `true` immediately. Workers still pointed here may have in-flight WFTs they are completing — with `all-apis-forwarding` and `false`, those Respond* calls are forwarded to the new active transparently. Wait until in-flight work has drained and workers have been re-pointed to the new active, then set `true` to cut off remaining connections cleanly.

### API forwarding to the active cluster: cluster policy and per-namespace settings

Forwarding is controlled at two levels: the cluster's static config sets what forwarding is possible at all, and two dynamic config settings narrow it per namespace.

**Static config: `dcRedirectionPolicy`.** This is one value for the whole cluster, in the server's static config YAML, and changing it needs a server restart.

| `dcRedirectionPolicy` | What the cluster forwards |
|---|---|
| not set (the default) or `noop` | Nothing, for any namespace. The dynamic config settings below have no effect. |
| `selected-apis-forwarding` | Only the APIs in the [selected-APIs whitelist](#forwarding-policy-reference). Polls, `Respond*` calls and heartbeats are not forwarded. |
| `all-apis-forwarding` | All APIs, including polls, `Respond*` calls and heartbeats. |

Any other value stops the frontend service from starting. For multi-cluster setups this playbook recommends `all-apis-forwarding` on every cluster; see [Setting `dcRedirectionPolicy`](#setting-dcredirectionpolicy). Only global (replicated) namespaces are ever forwarded, whatever the policy.

**Dynamic config: two namespace-scoped settings.** With forwarding enabled in static config, these two settings narrow it for a single namespace, or for every namespace on the cluster. Changing either one needs no restart.

| Setting | Default | Effect when changed from the default | Available in |
|---|---|---|---|
| `system.enableNamespaceNotActiveAutoForwarding` | `true` | `false` turns forwarding off for the namespace. Calls are handled by the cluster the client connected to. | v1.20.0 and later |
| `system.forceNamespaceSelectedAPIAutoForwarding` | `false` | `true` limits the namespace to the whitelisted APIs, even when the cluster runs `all-apis-forwarding`. | v1.31.0 and later |

Dynamic config can only narrow what the static policy allows. It cannot turn forwarding on, and it cannot make a `selected-apis-forwarding` cluster forward all APIs.

**Setting a value for one namespace.** Add an entry with a `namespace` constraint for that namespace. An entry with no constraint applies to every other namespace on the cluster. For a namespace, the entry with its name is used whatever the order of the entries; a namespace with no matching entry gets the setting's default.

```yaml
system.forceNamespaceSelectedAPIAutoForwarding:
  - value: true
    constraints:
      namespace: "my-namespace"
  - value: false
```

With these two entries, `my-namespace` is limited to the whitelisted APIs, and every other namespace keeps whatever forwarding the cluster's `dcRedirectionPolicy` allows.

`system.enableNamespaceNotActiveAutoForwarding` and `system.forceNamespaceSelectedAPIAutoForwarding` apply only to the cluster whose dynamic config they are set in. To change forwarding for a namespace on both clusters, set the value in the dynamic config of both clusters.

**What gets forwarded for a namespace**, from the policy and the two settings together:

| `dcRedirectionPolicy` | `system.enableNamespaceNotActiveAutoForwarding` | `system.forceNamespaceSelectedAPIAutoForwarding` | Forwarded for the namespace |
|---|---|---|---|
| not set (default) or `noop` | any | any | Nothing. Neither setting has any effect. |
| `selected-apis-forwarding` or `all-apis-forwarding` | `false` | any | Nothing. |
| `all-apis-forwarding` | `true` (default) | `false` (default) | All APIs, including polls |
| `all-apis-forwarding` | `true` (default) | `true` | Whitelisted APIs only |
| `selected-apis-forwarding` | `true` (default) | any | Whitelisted APIs only. `system.forceNamespaceSelectedAPIAutoForwarding` has no effect, because the policy is already whitelist-only. |

**A handover does not reset the forwarding settings.** A value set for a namespace in `system.enableNamespaceNotActiveAutoForwarding` or `system.forceNamespaceSelectedAPIAutoForwarding` stays in that cluster's dynamic config until someone removes it. A handover changes which cluster is active for the namespace; it does not change dynamic config. Forwarding only happens for calls that reach a cluster that is not the namespace's active cluster, so a value left over from an earlier handover takes effect again each time that cluster becomes the standby. Before each handover, check `system.enableNamespaceNotActiveAutoForwarding` and `system.forceNamespaceSelectedAPIAutoForwarding` for the namespace on both clusters, as the [Recommendations](#recommendations) say.

The next section, [Dynamic config](#dynamic-config), lists the dynamic config settings to have in place on both clusters before the first handover.

### Dynamic config

These must be set on a running cluster before the first handover — no restart required. Treat them as permanent standing configuration for any multi-cluster deployment.

| Config | Cluster | Required value | How to check |
|---|---|---|---|
| `system.enableNamespaceNotActiveAutoForwarding` | Both | `true` (default) | Namespace-scoped. Confirm it is not set to `false` for the namespace being handed over, or for all namespaces. If it is `false`, nothing is forwarded for that namespace, whatever `dcRedirectionPolicy` says. See [API forwarding to the active cluster: cluster policy and per-namespace settings](#api-forwarding-to-the-active-cluster-cluster-policy-and-per-namespace-settings). |
| `history.EnableReplicationTaskTieredProcessing` | Both | Same value on both clusters | When `true`, caps how fast each stream sends and lets the standby pause the active. See the notes below this table. |
| `history.standbyTaskMissingEventsDiscardDelay` | Standby | Default (15m) or higher | Lower values increase discard risk during elevated lag |
| `history.enableDeleteWorkflowExecutionReplication` | Both | **`true`** (v1.31.x only; default `false`) | Turns on delete replication so deletions on the active reach the standby. On v1.32.0+ it is always on — nothing to set. On v1.30.x and earlier the feature does not exist. See [Workflow Deletion](#workflow-deletion). |

**`history.EnableReplicationTaskTieredProcessing` is safe to change on a running cluster.** Change the setting on **both clusters** before the handover, so the two match.

- **Restart on a change:** the cluster sending the replication stream checks the setting each time the other cluster reports its progress, and restarts the stream when the value has changed.
- **After the flip:** the new active sends its own replication stream to the old active, and that stream needs the same setting on both clusters too.
- **Versions:** both clusters must be on v1.25.0 or later. A mismatch, or a cluster older than v1.25.0, makes the stream restart over and over.
- **Force-replication backlog:** `true` keeps a force-replication backlog of closed workflows from holding up the handover only when `history.enableTransitionHistory` is `false`, the default up to v1.29.x. From v1.30.0, a force-replication backlog holds up the handover whether tiered processing is on or off.

### Alerts

Full reference: [`observability/alerts/server/alerts-index.md`](../observability/alerts/server/alerts-index.md) — Section 20

All alerts below link to the [Namespace Failover: Graceful Handover dashboard](../observability/dashboards/server/namespace-failover-graceful-handover-readme.md) and fire annotations directly onto it — alert state is visible in context alongside the panels they monitor.

| Alert | Severity | Workflow step | Dashboard row | Fires when |
|---|---|---|---|---|
| `FAILOVER-PRE-01` — Replication Stream Stuck | critical | Pre-flight (before Step 1) | Row 1 | Any stream stopped making progress — hard blocker for handover |
| `FAILOVER-PRE-02` — Replication DLQ Enqueue Failing | critical | Pre-flight (before Step 1) | Row 1 | Tasks failing into DLQ on stream path — hard blocker for handover |
| `FAILOVER-PRE-03` — Replication Stream Reconnect Rate Elevated | warning | Pre-flight (before Step 1) | Row 1 | `replication_stream_error` sustained for 2m — advisory. A steady non-zero rate is expected from connection recycling; only a concern if the stream is also stuck or lag is rising (see `FAILOVER-PRE-01`) |
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

### Workflow Deletion

This describes how workflow deletion works for a namespace that is replicated across clusters.

#### How workflows get deleted

Deleting a workflow execution removes it from both the primary store (its history and state) and the visibility store (the record you see in the UI and in list queries). There are three ways to do it:

| Operation | How you run it | What it is for |
|---|---|---|
| `DeleteWorkflowExecution` | `temporal workflow delete --workflow-id <id>` | Delete one execution by ID. |
| Batch delete | `temporal workflow delete --query <query>` | Delete every execution matching a visibility query. Runs as a batch job. |
| Force delete (admin) | `tdbg workflow delete` | Last-resort removal that bypasses the normal path. Only for support or recovery situations. |

Deleting workflows is not a routine operation. Most workflows are cleaned up automatically by retention once they close. Explicit delete is for special cases — clearing out bad or stuck runs, or removing large batches of old executions ahead of time.

#### Workflow deletion for a global namespace

For a global namespace, delete operations are cluster-specific by default — a delete runs only on the cluster you run it against and is not copied to the other cluster. This can be turned on (and becomes the default in v1.32); see [How to enable replication for workflow delete operations](#how-to-enable-replication-for-workflow-delete-operations) below.

**When delete replication is enabled**, an explicit delete on the active cluster is copied to the standby, so both clusters stay in sync. This covers both ways of running an explicit delete:

- Single-workflow delete (`temporal workflow delete --workflow-id`).
- Batch delete (`temporal workflow delete --query`). A batch job deletes each matched workflow one at a time, so every one of those deletions is replicated the same way.

**These are never replicated, regardless of the flag:**

- Retention cleanup — each cluster expires closed workflows on its own schedule, so both sides clean up on their own. No action needed and no resurrection risk.
- The `tdbg` admin force-delete — it bypasses the normal delete path.

**When delete replication is off** (the default today, or v1.30.x where the feature does not exist), a workflow you delete on the active cluster still exists on the standby. After a failover the standby becomes active and those workflows reappear. This is called **resurrection**.

#### How to enable replication for workflow delete operations

Whether you can turn on delete replication depends on your server version:

| Server version | What to do |
|---|---|
| v1.30.x and earlier | Feature does not exist. Deletes never replicate. Nothing you can set. |
| **v1.31.0 – v1.31.x** | Set `history.enableDeleteWorkflowExecutionReplication` to `true` on **both** clusters. Off by default. |
| v1.32.0 and later | Always on. Nothing to set. |

#### Where to run delete operations

Always run delete operations against the currently active cluster for the namespace. This holds for every version. On v1.31.0+ the deletion is replicated to the standby, but the command itself still runs on the active — batch delete in particular runs as a workflow, which only runs on the active cluster.

What changes with the version is whether you have to repeat the delete after a handover:

- **Replication on** (v1.31.x with the flag enabled, or v1.32.0+): one run on the active cluster removes the workflows from both clusters. Nothing to repeat.
- **Replication off or unavailable** (v1.30.x and earlier, or the flag left off): the delete only affects the active cluster. After a handover, run it again on the new active cluster if those workflows still need to be removed.

---

### Namespace Deletion

This section describes namespace deletion for global (replicated) namespaces — for example, deleting one with the CLI:

```bash
temporal operator namespace delete --namespace <ns>
```

#### What namespace deletion does

Deleting a namespace starts a background system workflow that:

1. Deletes every workflow execution in the namespace — both running and closed.
2. Removes the namespace record itself.

It must be run on the **active** cluster. On a passive cluster it is rejected:

> namespace X is passive in current cluster Y: remove cluster Y from cluster list or make namespace active in this cluster and retry

#### What reaches the passive cluster — and what does not

Two different things happen, and they replicate differently:

- **The namespace record is never replicated.** Deleting the namespace only removes it on the active cluster. The passive keeps its own copy of the namespace record — always, on every version.
- **The per-execution deletes follow the normal workflow-delete rule.** Each execution the background workflow deletes is a regular `DeleteWorkflowExecution`, so it replicates to the passive only when delete replication is enabled (same version and flag as [Workflow Deletion](#workflow-deletion)).

So the namespace itself always survives on the passive. Whether its workflows also survive there depends on your version and the flag.

#### What happens by version

The table below shows the state you are left with **if you only run `namespace delete` on the active and do nothing else** — this is what motivates the full procedure that follows.

| Version / config | Active cluster | Passive cluster is left with |
|---|---|---|
| **Before v1.31.0** | Namespace and all executions deleted | The namespace **and all its executions** |
| **v1.31.x, flag off** (default) | Namespace and all executions deleted | The namespace **and all its executions** (the flag does nothing until you turn it on) |
| **v1.31.x, flag on** | Namespace and all executions deleted | The namespace, now **empty** — its executions were deleted via replication |
| **v1.32.0 and later** | Namespace and all executions deleted | The namespace, now **empty** — replication is always on |

The flag is `history.enableDeleteWorkflowExecutionReplication` (see [Dynamic config](#dynamic-config)).

In every case the namespace record itself survives on the passive. If you leave it there, it comes back on an unplanned failover. So to fully remove a global namespace, follow the procedure below — it is the same regardless of version.

#### How to fully delete a global namespace

Delete it on both clusters. **Do the steps in this order — disconnect first, before either delete.** Two reasons: you cannot delete on the passive until it is out of the cluster list, and you cannot change the cluster list once the namespace is deleted on the active (there is nothing left to update). So disconnect must come before both deletes.

**Step 1 — On the active cluster, shrink the cluster list to just the active cluster:**

```bash
temporal operator namespace update \
  --namespace <ns> \
  --cluster <active-cluster> \
  --address <active-frontend>
```

`--cluster` replaces the whole list, so passing only the active cluster removes the standby. The active cluster must stay in the list. This stops replicating the namespace to the standby; the standby keeps its (now orphaned) copy.

**Step 2 — Delete on the active cluster:**

```bash
temporal operator namespace delete --namespace <ns> --address <active-frontend>
```

**Step 3 — Delete the leftover copy on the standby.** This is now allowed, because Step 1 removed that cluster from the list:

```bash
temporal operator namespace delete --namespace <ns> --address <standby-frontend>
```

Confirm with `temporal operator namespace describe` against each cluster — both should return not found.

#### Notes

- This is a namespace-level change, not a cluster-level one. Do **not** use `operator cluster remove` — that disconnects every namespace between the two clusters.
- Removing a cluster from the list does not clean up the standby on its own. The standby stops receiving updates but keeps the namespace, which is why Step 3 is needed.

---

### Search Attributes

Custom search attributes must exist on every cluster in the replication group. Whether a change on the active reaches the passive depends on the visibility store.

| Visibility store | SA changes on the active reach the passive? | What you must do |
|---|---|---|
| SQL (MySQL / PostgreSQL) | **Yes** — the SA is stored as a namespace-config alias, which replicates | Add it on the active; it replicates to the passive. |
| Elasticsearch | **No** — the SA lives in each cluster's own metadata and ES index | Add it on **every** cluster. |

The same applies when you promote a namespace to global after it already has search attributes:

- **SQL:** the existing aliases are part of the namespace config, which replicates in full when the namespace is first replicated. Nothing to do — they come along.
- **Elasticsearch:** the existing search attributes are not replicated. **Add them all on the passive's Elasticsearch before the handover.** Otherwise, after the flip, workflows that use them will fail visibility upserts on the new active.

**Recommendation:** use the same visibility store type on every cluster in a replication group. A mixed setup — for example SQL on the active and Elasticsearch on the passive — is not a supported or validated configuration; search attributes are stored differently in each and will not stay consistent.

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
| `AllowedLaggingTasks` | No bounds | Gap in task IDs per shard, not a task count. `0` is the strictest value; a very large number makes it non-binding. [Section 1.3](#13-is-replication-lag-low-enough-to-proceed) explains how it combines with `AllowedLaggingSeconds`. |
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
| `AllowedLaggingTasks` | int64 | No bounds | Maximum gap in task IDs per shard, not a task count. `0` is the strictest value, not off; a very large number makes it non-binding. [Section 1.3](#13-is-replication-lag-low-enough-to-proceed) explains how it combines with `AllowedLaggingSeconds`. |
| `HandoverTimeoutSeconds` | int | Max 30s (enforced) | How long WaitHandover (Step 5) waits for the drain to complete before rolling back automatically. Values above 30 are clamped to 30 — this cap is hardcoded in the server |

> **Always set `HandoverTimeoutSeconds` explicitly — never omit it or set it to `0`.** A zero value causes the drain to time out immediately and roll back. **Recommended value: 30s** — the maximum the server allows, giving shards the full drain window.

---

## Running the Handover

This section is the operational playbook for running a graceful handover end-to-end — from pre-flight signal checks through the handover workflow execution and into post-flip health verification. It is structured around the six internal activity steps of `namespace-handover-v2` and maps directly to the [Namespace Failover: Graceful Handover dashboard](../observability/dashboards/server/namespace-failover-graceful-handover-readme.md). The dashboard is an integral part of this playbook — each section below corresponds to a dashboard row.

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
4. [Is the standby making extra round-trips to fetch missing history?](#14-is-the-standby-making-extra-round-trips-to-fetch-missing-history)
5. [Is the standby keeping up with incoming replication tasks?](#15-is-the-standby-keeping-up-with-incoming-replication-tasks)
6. [Are there workflow executions on the standby that will be stuck after the flip?](#16-are-there-workflow-executions-on-the-standby-that-will-be-stuck-after-the-flip)

#### 1.1. Do you need to run the force-replication workflow?

If required, the `force-replication-v2` workflow must be run and complete **before** you start the handover workflow — for the same namespace you intend to hand over.

> **`force-replication-v2` is a data-plane operation only — it does not fail over anything.** It backfills the target cluster with history/data it is missing; it never changes which cluster the namespace is active on. It does **not** call `UpdateActiveCluster` or `UpdateNamespaceState`, does not enter HANDOVER state, and does not bump the failover version — active stays active and passive stays passive throughout. Flipping the active cluster is done only by the handover workflow (its final `UpdateActiveCluster` step) or by a direct `UpdateNamespace` force failover. Run `force-replication-v2` to make the standby a complete replica; run the handover workflow to actually move traffic.

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
| `Query` | string | Required | Visibility query to scope which executions to backfill. Empty string backfills all executions in the namespace, open and closed |
| `ConcurrentActivityCount` | int | `1` | Number of replication activities to run in parallel. Raise for faster backfill; lower if receiver backlog builds up |
| `OverallRps` | float64 | `= ConcurrentActivityCount` | RPS cap for enqueuing replication tasks across all concurrent activities |
| `GetParentInfoRPS` | float64 | `= ConcurrentActivityCount` | RPS cap for parent/child relationship lookups during backfill |
| `ListWorkflowsPageSize` | int | `1000` | Page size for listing workflows via visibility |
| `PageCountPerExecution` | int | `200` (max `1000`) | Number of pages processed before the workflow continues-as-new to avoid history size limits |
| `EnableVerification` | bool | `false` | After backfill, verify executions were replicated to the target cluster |
| `TargetClusterEndpoint` | string | — | Required when `EnableVerification=true` if `TargetClusterName` is not set |
| `TargetClusterName` | string | — | Required when `EnableVerification=true` if `TargetClusterEndpoint` is not set |
| `VerifyIntervalInSeconds` | int | `5` | Polling interval for verification checks |

**Closed executions are backfilled too.** `force-replication-v2` copies every execution its `Query` returns, whatever its status. It has no running-only filter. An empty `Query` therefore copies completed, failed, canceled, terminated, timed-out and continued-as-new executions as well as running ones. Use the `Query` to narrow the backfill:

| To backfill | `Query` |
|---|---|
| Every execution in the namespace, open and closed | `""` |
| Running executions only | `ExecutionStatus = 'Running'` |
| Closed executions only | `ExecutionStatus != 'Running'` |
| Executions closed after a given time | `CloseTime > '2026-01-01T00:00:00Z'` |

Two limits apply to closed executions, both caused by namespace retention:

- **An execution that retention has already deleted on the active cluster cannot be backfilled.** Visibility no longer returns it, so `force-replication-v2` never sees it. An execution that retention deletes after it was listed but before it was copied is skipped, and `force-replication-v2` carries on without failing.
- **Verification does not fail on an execution that is past retention.** With `EnableVerification` set to `true`, a closed execution that is missing on the standby is counted as skipped, not as failed, when its close time plus the namespace retention is already in the past.

Wait for the workflow to complete before starting the handover workflow.

**If the handover has already run without `force-replication-v2`**, the executions it would have
copied can still be copied afterwards. See
[The handover completed but `force-replication-v2` was not run first](#the-handover-completed-but-force-replication-v2-was-not-run-first).

**If the namespace was a local namespace before it was made global**, the old active cluster logs
replication errors for some workflows after the flip. The errors clear on their own in most cases.
[Replication errors on the old active cluster after the flip](#replication-errors-on-the-old-active-cluster-after-the-flip)
covers what to check, and what to do before any later handover of the namespace.

#### 1.2. Is the replication stream between clusters healthy?

Alerts [`FAILOVER-PRE-01` through `FAILOVER-PRE-07`](../observability/alerts/server/alerts-index.md) fire on the blockers covered here. If any are already firing, you have your answer before opening the dashboard — but check the dashboard anyway. Alerts tell you something is wrong; the dashboard panels below tell you how bad it is, whether it is recovering, and what to do about it.

Open [Row 1 — Pre-Flight](../observability/dashboards/server/namespace-failover-graceful-handover-readme.md#row-1--pre-flight-go--no-go) in the dashboard.

| Panel | Blocker | Advisory |
|---|---|---|
| [Stream Stuck](../observability/dashboards/server/namespace-failover-graceful-handover-readme.md#panel-stream-stuck-stat) | Any non-zero rate → **blocker** | The standby has a built-in liveness monitor — if the stream stops making progress it reconnects automatically within ~3 minutes. Wait before acting. If it does not clear, see section [5](#5-something-went-wrong--rollback-and-recovery). |
| [DLQ Enqueue Failed](../observability/dashboards/server/namespace-failover-graceful-handover-readme.md#panel-dlq-enqueue-failed-stat) | Any non-zero rate → **blocker** | Fires on the standby. The DLQ is the last resort for replication tasks the standby cannot apply — if even that is failing, those tasks are permanently lost. Running a handover in this state makes no sense. Investigate and fix before proceeding. |
| [Replication DLQ Writes](../observability/dashboards/server/namespace-failover-graceful-handover-readme.md#panel-replication-dlq-writes-stat) | Any non-zero count → **blocker until the DLQ has been listed and merged** | Fires on the standby, for all namespaces on it. A write means the standby holds an out-of-date copy of a workflow, which runs after the flip. See the note below the table. |
| [Receiver Backlog](../observability/dashboards/server/namespace-failover-graceful-handover-readme.md#panel-receiver-backlog-depth-stat) | ≥ 500 → **blocker**, 400–499 → **do not start** | Replication tasks are backing up on the standby faster than it can apply them. Starting the handover now risks the backlog reaching its limit while WaitReplication runs. See section [1.5](#15-is-the-standby-keeping-up-with-incoming-replication-tasks). |
| [Stream Errors (gRPC)](../observability/dashboards/server/namespace-failover-graceful-handover-readme.md#panel-stream-errors--grpc-stat) | Elevated **and** paired with stalled progress → **investigate** | A steady non-zero rate here is **normal** — it is the connection recycling reconnecting the stream every few minutes (see the note below). On its own it is not a blocker. It only matters if it comes with **Stream Stuck** non-zero or **Replication Lag / Latency** rising — that combination means the stream is reconnecting *and* failing to catch up. Judge readiness by Stream Stuck and lag, not by this count. |
| [Stream Service Errors](../observability/dashboards/server/namespace-failover-graceful-handover-readme.md#panel-stream-service-errors-stat) | Sustained unexplained rate → **investigate before proceeding** | Fires on the standby. A brief spike when the standby restarts is normal and clears on its own. A sustained rate with no obvious cause means the standby is struggling to process the replication stream — do not start until you understand why. |
| [Replication Lag](../observability/dashboards/server/namespace-failover-graceful-handover-readme.md#panel-replication-lag-gauge) | Trending up → **do not start** | Fires on the active. The gap in task IDs, not a count of tasks, between the newest task ID of any kind on the shard and the last replication task the standby confirmed. Rising means the standby is falling behind. Section [1.3](#13-is-replication-lag-low-enough-to-proceed) gives the time behind. |
| [Backfill Activity](../observability/dashboards/server/namespace-failover-graceful-handover-readme.md#panel-backfill-activity-stat) | — | The standby is receiving replication tasks but is missing some of the history behind them — it has to fetch that history from the active before it can apply each affected task. This slows down catchup. See section [1.4](#14-is-the-standby-making-extra-round-trips-to-fetch-missing-history). |
| [Receive Buffer Full](../observability/dashboards/server/namespace-failover-graceful-handover-readme.md#panel-receive-buffer-full-standby-stat) | Sustained during steady load → **do not start** | Fires on the standby. The standby's buffer of received replication tasks is full, so the standby applies them more slowly than the active sends. Short bursts while the standby catches up are normal. See the note below the table. |
| [Replication Tasks Sent](../observability/dashboards/server/namespace-failover-graceful-handover-readme.md#panel-replication-tasks-sent-per-second-active-stat) | — | Fires on the active. How many replication tasks the active sends per second. Lag rising while Replication Tasks Sent is flat has two causes; see the note below the table. |
| [Standby Task Discards](../observability/dashboards/server/namespace-failover-graceful-handover-readme.md#panel-standby-task-discards-stat) | Any non-zero rate → **blocker** | Fires on the standby. Workflow executions on the standby had their tasks discarded because the history events they depended on never arrived in time. After the flip those workflows will be stuck on the new active with no pending task — direct business impact. Identify and understand the scope before proceeding. See section [1.6](#16-are-there-workflow-executions-on-the-standby-that-will-be-stuck-after-the-flip). |

> If any blocker above is unresolved, our recommendation is not to proceed with the handover. Resolve the issue first and re-run this check before continuing.

**Replication DLQ writes on the standby.** The **Replication DLQ Writes** count is the number of replication updates the standby could not apply and wrote to its replication DLQ (its queue of replication updates it could not apply) since its history pods last started. Each write leaves the standby's copy of a workflow out of date, and after the flip that copy is the one that runs. A non-zero count is the prompt to list the DLQ. The count does not go down after a merge, so an empty DLQ listing means the writes were already merged and the handover can go ahead. [Replication errors on the old active cluster after the flip](#replication-errors-on-the-old-active-cluster-after-the-flip) gives the commands to list and merge the DLQ.

**Stream errors are usually normal — here is how to tell.** A steady, non-zero **Stream Errors (gRPC)** rate (often between 10s and 20s on a busy cluster) is expected. It is the connection between clusters being recycled every few minutes; the stream drops and re-establishes within a couple of seconds and resumes where it left off — no data is lost. 
So looking at just raw error count is not really indication of a non-healthy stream. It's also important to look at two things:

- **Stream Stuck** (`replication_stream_stuck`) — this only goes non-zero when the stream stops making forward progress (its acknowledged position stops advancing). This, not the error count, is the real "the stream is broken" signal.
- **Replication Lag / Replication Latency** — if these are flat and low, the stream is keeping up despite the reconnects.

Errors + healthy Stream Stuck and lag = benign, proceed. Errors + Stream Stuck non-zero *or* rising lag = a real problem, do not proceed.

> **Why these reconnects happen — and should I change a setting to stop them?**
>
> These reconnects come from Temporal periodically recycling its cross-cluster connections (`frontend.keepAliveMaxConnectionAge`, default 5 minutes). 
> The recycling keeps replication connections spread evenly across your frontend pods, so one pod doesn't end up carrying most of them. 
> It is not a replication-performance setting.
>
> You can raise `frontend.keepAliveMaxConnectionAge` (up to around 24 hours) to make the reconnects, and the stream errors they cause, much less frequent.
> Connections that recycle less often can build up unevenly, with some frontend pods holding far more replication connections than others. So keep the setting short only if you actually see that imbalance. If load across your frontend pods is already even, a longer value quiets the stream errors.
>
> Keep `frontend.keepAliveMaxConnectionAge` at its default (5 minutes) on both clusters if any namespace was a local namespace before it was made global and still has workflows that were started before then. After a flip, the first update that adds history events to each such workflow is applied only at the next reconnect. If the update's retries run out first, the update is written to the replication DLQ instead, and a longer reconnect interval makes that the usual outcome (reasoned from the retry limits, not measured at a raised value). See [Replication errors on the old active cluster after the flip](#replication-errors-on-the-old-active-cluster-after-the-flip).
>
> If you run a proxy or load balancer in front of your clusters, it may enforce its own connection age limit that also forces these reconnects — in that case you have to raise it there too, not just in Temporal.
>
> Bottom line: don't change this just to silence the errors. Decide based on whether replication is actually falling behind — check **Stream Stuck** and the lag panels.

**Receive Buffer Full that stays on under steady load means the standby is not catching up.** Wait for it to drop to zero before starting the handover. Check the standby's history pod CPU, its database latency, and **Receiver Backlog**, and see section [1.5](#15-is-the-standby-keeping-up-with-incoming-replication-tasks).

> **Test cluster result:** after a 4-minute standby outage with load still running, Receive Buffer Full stayed at about 130 per second and Replication Latency p99 stayed at 16.7 minutes. A handover started then stayed in WaitReplication for over 15 minutes, with the namespace still in normal state, and was stopped. With the load paused until Receive Buffer Full was zero, the same handover completed.
> - **Setup:** two clusters on v1.31.0, 4 history shards each, PostgreSQL, `AllowedLaggingSeconds` 10.

**Lag rising while Replication Tasks Sent stays flat has two causes.** If Receive Buffer Full is non-zero, the standby is the limit, and raising how fast the active sends does not help. If Receive Buffer Full is zero, the active is not sending any faster. With `history.EnableReplicationTaskTieredProcessing` set to `true`, the cap is `history.ReplicationStreamSenderHighPriorityQPS`, default 100 tasks per second per stream. With it `false` the active has no cap on how fast it sends.

> **Test cluster result:** catching up after a standby outage, the active sent 400 tasks per second with tiered processing on (4 streams × 100) and 550–680 per second with it off.
> - **Setup:** same two clusters.

#### 1.3. Is replication lag low enough to proceed?

> Assumes the replication stream between the two clusters is healthy, so complete section [1.2](#12-is-the-replication-stream-between-clusters-healthy) first.

**Replication lag matters because the handover's drain step has at most 30 seconds.** The handover workflow puts the namespace into HANDOVER state. It then waits up to `HandoverTimeoutSeconds`, which is capped at 30, for the standby to confirm every replication task up to that point. If the standby has not confirmed them in time, the handover rolls back and the namespace stays active on the current cluster.

The dashboard shows lag in two ways:

- **Time behind:** [Replication Latency p99](../observability/dashboards/server/namespace-failover-graceful-handover-readme.md#panel-replication-latency-p99-time-series) (Row 1, standby cluster) shows how long a replication task takes from being created on the active to being applied on the standby. Decide by this panel, because the drain step is limited by time. Alert `FAILOVER-PRE-07` fires when p99 reaches 20 seconds.
- **Direction:** [Replication Lag Trend](../observability/dashboards/server/namespace-failover-graceful-handover-readme.md#panel-replication-lag-trend-time-series) and the [Replication Lag gauge](../observability/dashboards/server/namespace-failover-graceful-handover-readme.md#panel-replication-lag-gauge) show a gap in task IDs, not a count of tasks. All of a shard's tasks take their IDs from one sequence, so the gap grows with every task the shard writes, not only with replication tasks. Use these panels to see whether lag is rising, falling or flat. Values above 100,000 show as exactly 100K.

| What the lag panels show | What it means | What to do |
|---|---|---|
| Declining toward zero | The standby is catching up | Wait for the decline to flatten and for Replication Latency p99 to settle before starting. |
| Rising | The standby is falling further behind | Do not start. Check the [Receiver Backlog Depth panel](../observability/dashboards/server/namespace-failover-graceful-handover-readme.md#panel-receiver-backlog-depth-stat) and the stream health panels from section [1.2](#12-is-the-replication-stream-between-clusters-healthy). Wait until the trend reverses. |
| Flat at a non-zero value | The standby is keeping pace with the active's current load | Decide by Replication Latency p99. A p99 close to 30 seconds means a task sent just before HANDOVER can take as long as the whole drain step. |

**How WaitReplication uses `AllowedLaggingTasks` and `AllowedLaggingSeconds`.** When the handover workflow starts, it records each shard's newest replication task ID. WaitReplication first waits until the standby has confirmed every shard up to that recorded task. After that, a shard counts as caught up when either of these is true:

- **Tasks:** the gap between the shard's newest replication task ID and the last one the standby confirmed is at most `AllowedLaggingTasks`. The gap is in task IDs, per shard, not a count of tasks.
- **Seconds:** the time between the shard's newest replication task and the last one the standby confirmed is at most `AllowedLaggingSeconds`. Values below 5 are raised to 5, and values above 120 are lowered to 120.

**Do not copy the gauge value into `AllowedLaggingTasks`.** The gauge measures from the newest task ID of any kind on the shard. WaitReplication measures from the newest replication task ID, so the gauge reads the same or higher. Keep the starting values, `AllowedLaggingSeconds: 10` and `AllowedLaggingTasks: 500`. A larger `AllowedLaggingTasks` only lets WaitReplication finish sooner; the drain step still has to clear what is left within `HandoverTimeoutSeconds`.

> If lag is rising, or Replication Latency p99 is close to 30 seconds, do not proceed. Wait for the trend to reverse and latency to settle.

Section [1.4](#14-is-the-standby-making-extra-round-trips-to-fetch-missing-history) checks whether the standby is slowed down by fetching missing history from the active.

#### 1.4. Is the standby making extra round-trips to fetch missing history?

**Dashboard:** [Backfill Activity panel](../observability/dashboards/server/namespace-failover-graceful-handover-readme.md#panel-backfill-activity-stat) (Row 1, standby cluster)

When the standby applies a replication task, it sometimes finds that the history events that task depends on have not arrived yet. When that happens, it makes a synchronous call back to the active to fetch them before it can continue. This is not an error — the task eventually completes — but each of these extra round-trips makes that task take longer to apply. A high rate of this means WaitReplication will take longer than the raw lag number alone suggests.

| What you see | What to do |
|---|---|
| Zero or low rate | Proceed — no impact on catchup timing |
| Elevated rate | WaitReplication will take longer than the lag number alone suggests — each affected task needs an extra round-trip to the active. It will drop naturally as the stream catches up. If you need the handover to complete quickly, wait for it to subside first. |

> This check is advisory only — there is no value here that blocks you from proceeding. Continue to section 1.5 regardless of what you see.

#### 1.5. Is the standby keeping up with incoming replication tasks?

**Dashboard:** [Receiver Backlog Depth panel](../observability/dashboards/server/namespace-failover-graceful-handover-readme.md#panel-receiver-backlog-depth-stat) (Row 1, standby cluster)

Replication tasks travel from the active to the standby over a stream. When they arrive on the standby they land in a receive buffer before being applied. The Receiver Backlog Depth panel shows how many tasks are sitting in that buffer waiting to be processed.

**What happens when the receive buffer reaches its limit depends on tiered processing.** The limit is `history.ReplicationReceiverMaxOutstandingTaskCount`, default 500 tasks per shard.

- **`history.EnableReplicationTaskTieredProcessing` set to `true`:** the standby tells the active to pause sending. The stream stays open, and the active sends again once the backlog drops below the limit.
- **`history.EnableReplicationTaskTieredProcessing` set to `false`:** the standby sends no pause, and the active does not pause at 500. What limits the backlog in that case has not been measured.

**A handover started while the standby is behind waits in WaitReplication.** WaitReplication is Step 3 of the handover workflow. It waits for the standby to catch up before the namespace goes into HANDOVER state. The namespace stays in its normal state, so traffic is not affected, but no flip happens. Each WaitReplication attempt runs for up to 1 hour, and the handover workflow then retries it with no limit on attempts.

| Backlog depth (p99) | What it means | What to do |
|---|---|---|
| < 100 | Healthy | Safe to proceed |
| 100–399 | Elevated | Resolve the backlog before starting. Check standby history pod CPU, database latency, and Receive Buffer Full in section [1.2](#12-is-the-replication-stream-between-clusters-healthy). |
| 400–499 | Close to the 500-task limit | Do not start. The workload keeps adding tasks while WaitReplication runs. |
| ≥ 500 | At the 500-task limit. With tiered processing `true`, the active has paused sending. | Hard blocker. Resolve the receive backlog first. |

> If backlog depth is 100 or above, resolve the backlog before starting the handover workflow.

Once backlog depth is below 100, section [1.6](#16-are-there-workflow-executions-on-the-standby-that-will-be-stuck-after-the-flip) checks for workflow executions on the standby that will be stuck after the flip.

#### 1.6. Are there workflow executions on the standby that will be stuck after the flip?

**Dashboard:** [Standby Task Discards panel](../observability/dashboards/server/namespace-failover-graceful-handover-readme.md#panel-standby-task-discards-stat) (Row 1, standby cluster) · Alert `FAILOVER-PRE-06`

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

**Watch [Row 2a — WaitReplication](../observability/dashboards/server/namespace-failover-graceful-handover-readme.md#row-2a--waitreplication-steps-13--no-client-impact) in the dashboard.**

The key panel is the [Catchup Progress % gauge](../observability/dashboards/server/namespace-failover-graceful-handover-readme.md#panel-catchup-progress--gauge). The gauge shows the percentage of shards that WaitReplication counts as caught up, by the rule in section [1.3](#13-is-replication-lag-low-enough-to-proceed). You want this to climb steadily toward 100%. The [Handover Progress — Catchup + Drain Combined](../observability/dashboards/server/namespace-failover-graceful-handover-readme.md#panel-handover-progress--catchup--drain-combined-time-series) panel shows both catchup and drain progress together. During this phase only catchup rises; drain stays at 0 until the flip begins.

> **Catchup progress covers every replicated namespace, not only the one being handed over.** WaitReplication checks each shard's replication stream, and that stream carries every replicated namespace on the shard.

**What normal looks like:** catchup progress climbs steadily. How fast depends on tiered processing:

- **`history.EnableReplicationTaskTieredProcessing` set to `true`:** the active sends at most `history.ReplicationStreamSenderHighPriorityQPS` tasks per second per stream, default 100. Raising that limit before you start speeds up catch-up only while Receive Buffer Full is zero; see the note on lag rising while Replication Tasks Sent stays flat in section [1.2](#12-is-the-replication-stream-between-clusters-healthy).
- **`history.EnableReplicationTaskTieredProcessing` set to `false`:** the active has no send cap, and catch-up speed depends on how fast the standby applies the tasks.

**What to watch for:**

| Signal | What it means | What to do |
|---|---|---|
| Catchup progress rising steadily | Normal — standby is catching up | Wait for it to reach 100% |
| Catchup progress stalled for more than 2 minutes and lag is flat | Stream may be stuck | Check the [Stream Stuck panel](../observability/dashboards/server/namespace-failover-graceful-handover-readme.md#panel-stream-stuck-stat) in Row 1 — a non-zero rate confirms it. See section [5](#5-something-went-wrong--rollback-and-recovery) for recovery steps. |
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

**Watch [Row 2b — HANDOVER Drain](../observability/dashboards/server/namespace-failover-graceful-handover-readme.md#row-2b--handover-drain-steps-45) in the dashboard.**

| What you see | What it means | What to do |
|---|---|---|
| [Handover Drain Progress %](../observability/dashboards/server/namespace-failover-graceful-handover-readme.md#panel-handover-drain-progress--gauge) rising toward 100% | Shards are confirming catchup — drain is proceeding normally | Wait for it to reach 100% |
| [UNAVAILABLE Error Burst](../observability/dashboards/server/namespace-failover-graceful-handover-readme.md#panel-unavailable-error-burst-expected-time-series) spiking | Expected — this is the active cluster returning `Unavailable` during HANDOVER. The SDK retries automatically. | Normal — watch for it to clear once drain completes |
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
- The receiver backlog on the standby reached its limit during the drain. With `history.EnableReplicationTaskTieredProcessing` set to `true`, the active then pauses sending. Check the Receiver Backlog Depth panel; if it was elevated going in, that is the likely cause. See section [1.5](#15-is-the-standby-keeping-up-with-incoming-replication-tasks).
- A force-replication backlog was still being replicated when HANDOVER started. Backfill for running executions is part of the drain in every release. Backfill for closed executions is kept out of the drain only when `history.EnableReplicationTaskTieredProcessing` is `true` and `history.enableTransitionHistory` is `false`, the default up to v1.29.x. Let force-replication finish before re-running, including its activities. Terminating the workflow does not stop activities that are already running.
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

Step 6 ran `UpdateActiveCluster`, and the namespace is now active on the new cluster. Watch [Row 3 — Flip Confirmed](../observability/dashboards/server/namespace-failover-graceful-handover-readme.md#row-3--flip-confirmed-step-6) in the dashboard. Of its three panels, only **UNAVAILABLE Errors Gone** is a pass or fail signal. **Forwarding Active** and **Version Fence Fired** describe the flip from the old active cluster's side, and 0 is a normal reading for both. Confirm the flip itself with `temporal operator namespace describe`, as described below the table.

| Panel (on the old active) | What to look for |
|---|---|
| [UNAVAILABLE Errors Gone](../observability/dashboards/server/namespace-failover-graceful-handover-readme.md#panel-unavailable-errors-gone-stat) (`client_redirection_errors`, `error_type="Unavailable"`) | Returns to 0. The write block is lifted, and clients are no longer being rejected. |
| [Forwarding Active](../observability/dashboards/server/namespace-failover-graceful-handover-readme.md#panel-forwarding-active-stat) (`client_redirection_requests`) | Rises when clients or workers still call the old active and it forwards those calls to the new active. Zero, shown in blue, is normal when all of them already point at the new active. |
| [Version Fence Fired](../observability/dashboards/server/namespace-failover-graceful-handover-readme.md#panel-version-fence-fired-stat) (`task_errors_version_mismatch`) | May spike, which is expected, see below. Zero, shown in blue, is also a valid outcome. |

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

The namespace is now active on the new cluster and the handover workflow has completed. Watch [Row 4 — Post-Handover Health](../observability/dashboards/server/namespace-failover-graceful-handover-readme.md#row-4--post-handover-health) in the dashboard for 5–10 minutes. The first minute or two may be noisy — queue processors are activating, workers are reconnecting, and the reverse replication stream is establishing. Give signals time to settle before acting on them.

**What normal looks like in the first few minutes:**

- If the old active is running `dcRedirectionPolicy=all-apis-forwarding`, workers that were polling the old active will have their Respond calls forwarded to the new active — you will see [Forwarding Success Rate](../observability/dashboards/server/namespace-failover-graceful-handover-readme.md#panel-forwarding-success-rate-time-series) on the old active rise and then gradually decline as workers are re-pointed. If you are running `selected-apis-forwarding`, Respond calls are not in the forwarding whitelist — workers must already be pointed at the new active or those calls will fail.
- [WFT Schedule-to-Start](../observability/dashboards/server/namespace-failover-graceful-handover-readme.md#panel-wft-schedule-to-start-new-active-time-series) on the new active may spike briefly while its queue processors warm up — this is expected and should normalize on its own.
- Some workflows may take up to 5 seconds longer to get their first task picked up after the flip. This is because workers that were running on the old active had a dedicated fast-path (sticky queue) for picking up tasks. After the flip that fast-path no longer exists on the new active, so the server waits 5 seconds (the default sticky timeout) before falling back to the normal queue. Once that happens the workflow resumes normally — this is not a stuck state.
- The old active (now standby) will show timer queue lag of approximately `history.standbyClusterDelay` (default 5 minutes) — this is normal standby behavior.

**What to watch for:**

| Panel | Cluster | What it means if something looks wrong |
|---|---|---|
| [Version Mismatch Decay](../observability/dashboards/server/namespace-failover-graceful-handover-readme.md#panel-version-mismatch-decay-time-series) | Old active | The old active is working through its backlog of pre-flip tasks — each one is version-checked, found to belong to the new active, and safely dropped. This should trend toward 0 as the backlog drains. If it stays non-zero for more than a few minutes, a queue processor on the old active may be stuck — check history pod logs. |
| [Forwarding Error Rate by Type](../observability/dashboards/server/namespace-failover-graceful-handover-readme.md#panel-forwarding-error-rate-by-type-time-series) | Old active | Should be 0. `Unavailable` = new active unreachable. `ResourceExhausted` = new active throttling, raise `frontend.globalNamespaceRPS` if set, else `frontend.namespaceRPS`. `FailedPrecondition` = workers completing tasks on old active without forwarding — see dynamic config section below. |
| [WFT Schedule-to-Start](../observability/dashboards/server/namespace-failover-graceful-handover-readme.md#panel-wft-schedule-to-start-new-active-time-series) | New active | Still elevated after 2 minutes = workers have not reconnected to the new active yet. Check that workers are polling the new active. |
| [Replication Lag — Reverse Stream](../observability/dashboards/server/namespace-failover-graceful-handover-readme.md#panel-replication-lag--reverse-stream-time-series) | New active | Gap in task IDs, for direction only. Rising means the old active, now the standby, cannot keep up with the new active. Right after the flip, p99 can sit at the 100K cap; read p50. |
| [Reverse Replication Active (New Active → Old Active)](../observability/dashboards/server/namespace-failover-graceful-handover-readme.md#panel-reverse-replication-active-time-series) | Old active (now standby) | Rises shortly after the flip while the namespace has traffic on the new active. Flat while there is traffic means the reverse stream has not been established. Check [Stream Stuck](../observability/dashboards/server/namespace-failover-graceful-handover-readme.md#panel-stream-stuck-stat) on the old active. |

**Re-check task processing on the new active.** That namespace's task load has just changed from
verifying replicated work to doing it, so a task scheduler limit sized while the namespace was
passive can now be the thing holding it back. Take the three readings in
[History Task Processing — Tuning and Troubleshooting](./history-task-processing-tuning.md#53-step-3--set-the-numbers-then-take-it-out-of-shadow-mode)
once traffic has settled.

#### Check for stuck workflows and recover them

If all pre-flight checks were clean — in particular if the Standby Task Discards panel in section 1.6 was zero — stuck workflows are very unlikely after a graceful handover. If section 1.6 was non-zero and you proceeded anyway, this is where those workflows will surface.

The [WFT Schedule-to-Start](../observability/dashboards/server/namespace-failover-graceful-handover-readme.md#panel-wft-schedule-to-start-new-active-time-series) panel tells you whether tasks are queued but workers are slow to pick them up. It does not detect workflows whose tasks were never re-scheduled on the new active at all — those are invisible to this metric.

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

#### Replication errors on the old active cluster after the flip

Read this subsection only if the namespace was a local namespace before it was made global
(promoted with `temporal operator namespace update --promote-global`). It covers replication errors
the old active cluster logs after the flip, what they mean for its copy of the affected workflows, and
the replication DLQ check to make before any later handover of the namespace.

**What you see:** for some workflows, the old active cluster logs the message
`SyncVersionedTransition replication task encountered error` at log level ERROR, with
`event slices are not consecutive Expected N, but got 1` as its error. The message repeats every 1 to
5 seconds by default for the same workflow, as the old active cluster retries the update. The repeats for one workflow end in one of two ways:

- **The update is applied.** The old active cluster logs the message
  `VerifyVersionedTransition replication task encountered error` at log level ERROR, with
  `mutable state not up to date` as its error, once. Nothing more is logged for that workflow.
- **The update is written to the DLQ.** The old active cluster logs the message
  `Enqueued replication task to DLQ` at log level ERROR for that workflow, and
  [Replication DLQ Writes (Previously Active Cluster)](../observability/dashboards/server/namespace-failover-graceful-handover-readme.md#panel-replication-dlq-writes-previously-active-cluster-time-series)
  shows a write. The update is now in the old active cluster's replication DLQ (its queue of
  replication updates it could not apply).

**Why the old active cluster rejects the update:** the affected workflows are the ones started while
the namespace was still local, so their first history events were written before the namespace had a
failover version. The first time the new active cluster adds history events to such a workflow, the
old active cluster asks the new active cluster for the events it is missing. The reply starts at the
workflow's first event instead of where the old active cluster's copy ends, so the old active cluster
rejects it and retries the same update. Only that first update is affected. Once one update for a
workflow has been applied, later updates for it apply normally.

**Why the repeated errors for one workflow stop:**

- **The replication connection is re-established.** Temporal re-establishes the connection between
  the clusters about every five minutes by default (`frontend.keepAliveMaxConnectionAge`, default
  `5m`). After the reconnect the update is sent again in a form the old active cluster applies.
- **The update's retries run out first.** The old active cluster stops retrying the update after
  `history.ReplicationExecutableTaskErrorRetryMaxAttempts` attempts (default `80`, about five and a
  half minutes at the retry pace measured on a test cluster) or after `history.ReplicationExecutableTaskErrorRetryExpiration` (default `10m`),
  whichever comes first, and writes the update to its replication DLQ.

At their default values the reconnect interval and the retry limit are close, so either outcome is
possible. Raising `frontend.keepAliveMaxConnectionAge`, or lowering either retry setting, makes the
DLQ outcome more likely.

**Impact:** none on the new active cluster. Workflows run normally there. On the old active cluster:

- **An update applied at the reconnect needs nothing.** Measured on a test cluster, every such
  workflow caught up and ended with the same state on both clusters, nothing was written to the DLQ,
  and each workflow's errors cleared within five minutes.
- **An update in the DLQ is repaired by the next update to the same workflow.** If no next update
  comes, for example because the update in the DLQ was the workflow's termination or completion, the
  old active cluster's copy stays out of date until the DLQ is merged. If the namespace is handed back
  to the old active cluster before the merge, the out-of-date copy is the one that runs. Measured on a test
  cluster, a workflow terminated after the flip was running again once the namespace was handed back, ran a
  workflow task and completed, and the termination no longer showed on either cluster. Merging the DLQ before
  handing the namespace back, as described below, brings the copy up to date first.

**What to do:**

1. Watch [Replication DLQ Writes (Previously Active Cluster)](../observability/dashboards/server/namespace-failover-graceful-handover-readme.md#panel-replication-dlq-writes-previously-active-cluster-time-series)
   for as long as workflows started before the namespace was made global are still running. An
   affected workflow's first update can come long after the flip, for example when a long timer fires.
2. If the panel shows writes, list the old active cluster's DLQs, then merge its replication DLQ:

   ```bash
   tdbg --address <old-active-cluster-frontend> dlq list

   tdbg --address <old-active-cluster-frontend> dlq merge \
     --dlq-type 3 \
     --cluster <new-active-cluster-name> \
     --last-message-id <LASTMESSAGEID of the replication DLQ>
   ```

   The replication DLQ is the line in the list whose name starts with `3_`. `--dlq-type 3` selects
   it, and `--cluster` names the cluster the updates came from. Measured on a test cluster, the merge
   brought every out-of-date copy up to date, and merging an update that had already been repaired did
   no harm.
   [Reading, merging and purging the DLQ with tdbg](./visibility-archival-tasks-dlq.md#7-reading-merging-and-purging-the-dlq-with-tdbg)
   explains these `tdbg dlq` commands in full, including what the list shows, how to see which
   workflows the updates belong to, why to always give `--last-message-id`, and how to check that a
   merge finished. The same commands work for the replication DLQ with `--dlq-type 3` and
   `--cluster <new-active-cluster-name>` added.
3. Before any later handover of this namespace, including one that hands it back to the old active
   cluster, check
   [Replication DLQ Writes](../observability/dashboards/server/namespace-failover-graceful-handover-readme.md#panel-replication-dlq-writes-stat)
   in [section 1.2](#12-is-the-replication-stream-between-clusters-healthy).

**Replication errors on the old active cluster are now covered.**
[Dynamic config changes after the flip](#dynamic-config-changes-after-the-flip) covers when to stop
forwarding API calls from the old active cluster.

#### Dynamic config changes after the flip

Now that the flip has happened, the old active is serving as standby and forwarding write calls to the new active. You need to decide when to tighten forwarding. **Order matters** — workers do two things: they poll for tasks, and they respond when done (completing a workflow task, finishing an activity, etc). As long as a worker is still polling the old active, it may pick up a task and try to respond on the old active too. With `all-apis-forwarding` active, that respond call is forwarded to the new active transparently. If you tighten forwarding before all workers are re-pointed, those respond calls stop being forwarded and the old active rejects them with an error the SDK cannot retry — the task fails.

1. **Verify all workers are fully re-pointed to the new active first.** Workers completing tasks or sending heartbeats through the old active depend on `all-apis-forwarding` being active. If you tighten forwarding before workers are re-pointed, those calls will fail with a non-retryable error.
2. **Set `system.forceNamespaceSelectedAPIAutoForwarding=true` on the old active** (scoped to the namespace, no restart). This switches the old active from forwarding all APIs, including polls, to forwarding only the APIs in the [selected-APIs whitelist](#forwarding-policy-reference), which depends on your release. Workers can no longer poll through the old active, so they must be pointed at the new active directly.
3. **Once no forwarding is needed at all**, set `system.enableNamespaceNotActiveAutoForwarding=false` on the old active, scoped to the namespace (no restart). This turns forwarding off for this namespace only; other namespaces on the old active keep forwarding. The value stays in place until you remove it. Before any later handover that makes this cluster the standby again, set it back to `true`, as the [pre-handover check](#recommendations) says. See [API forwarding to the active cluster: cluster policy and per-namespace settings](#api-forwarding-to-the-active-cluster-cluster-policy-and-per-namespace-settings).

Here is a summary of the config changes to make and when:

| Config | Cluster | When to apply |
|---|---|---|
| `system.forceNamespaceSelectedAPIAutoForwarding` → `true` (namespace-scoped) | Old active | After all workers are re-pointed to new active |
| `system.enableNamespaceNotActiveAutoForwarding` → `false` (namespace-scoped) | Old active | After forwarding is no longer needed at all. Set it back to `true` before any later handover that makes this cluster the standby again. |
| `frontend.namespaceRPS` → raise (or `frontend.globalNamespaceRPS` if your cluster sets it) | New active | If `ResourceExhausted` errors appear on Forwarding Error Rate panel |

> If you set `system.forceNamespaceSelectedAPIAutoForwarding=true` too early and workers are still polling the old active, you will see `FailedPrecondition` errors on the [Forwarding Error Rate by Type panel](../observability/dashboards/server/namespace-failover-graceful-handover-readme.md#panel-forwarding-error-rate-by-type-time-series). Alert `FAILOVER-POST-01` detects this. To recover: set `system.forceNamespaceSelectedAPIAutoForwarding=false` immediately, wait 2 seconds for propagation, and verify the errors clear.

> If all signals in Row 4 look healthy, no stuck workflows were found, and [Replication DLQ Writes (Previously Active Cluster)](../observability/dashboards/server/namespace-failover-graceful-handover-readme.md#panel-replication-dlq-writes-previously-active-cluster-time-series) shows no writes, the handover is complete. If the handover workflow rolled back, the drain timed out, you need to force a failover, or executions are missing on the new active cluster, see [section 5](#5-something-went-wrong--rollback-and-recovery).

---

## 5. Something went wrong — rollback and recovery

This section is a reference for situations where something did not go as expected. Each scenario below describes what you will see, why it happened, and what to do. Earlier sections in this playbook point here when they detect a problem.

In most cases: **the namespace is safe**. The handover workflow is designed so that if anything goes wrong before the flip completes, the namespace is automatically returned to its original state. No flip means no data loss and no stuck active cluster. Two situations below come after a completed flip, so that automatic return does not apply to them. A [passive-passive state](#the-handover-completed-but-the-namespace-is-in-a-passive-passive-state) is rare. [A handover that completed before `force-replication-v2` was run](#the-handover-completed-but-force-replication-v2-was-not-run-first) leaves the new active cluster without the executions the standby never received, and those executions have to be copied over afterwards.

---

#### The handover workflow rolled back — no flip occurred

**What you see:** [Handover Drain Progress %](../observability/dashboards/server/namespace-failover-graceful-handover-readme.md#panel-handover-drain-progress--gauge) was climbing during section 2b, then dropped back to 0. `Unavailable` errors cleared. Alert `FAILOVER-HANDOVER-01` may have fired. The namespace is back on the original active cluster.

**Why it happened:** the drain window expired before all shards confirmed catchup. The workflow rolled back automatically — no operator action was needed and no flip occurred.

**Common causes:**

| Cause | Where to check |
|---|---|
| Too much replication lag when HANDOVER started | Go back to section [1.3](#13-is-replication-lag-low-enough-to-proceed) — wait for lag to be near-zero before re-running |
| Receiver backlog hit its limit during the drain | Check [Receiver Backlog Depth](../observability/dashboards/server/namespace-failover-graceful-handover-readme.md#panel-receiver-backlog-depth-stat); see section [1.5](#15-is-the-standby-keeping-up-with-incoming-replication-tasks) |
| A force-replication backlog was still being replicated when HANDOVER started | Let force-replication finish, including activities that are already running, before re-running. See the rollback reasons in section [2b](#2b-namespace-is-in-handover--monitor-the-drain-window-steps-45) |
| Migration worker pod restarted during the drain window | Re-run once the pod is stable |

**What to do:** go back to section 1 and repeat all pre-flight checks before re-running the handover workflow.

---

#### The standby is not catching up — WaitReplication is stalled

**What you see:** you started the handover workflow and are in section 2a. [Catchup Progress %](../observability/dashboards/server/namespace-failover-graceful-handover-readme.md#panel-catchup-progress--gauge) has not moved for several minutes. The namespace is still serving traffic normally — no client impact yet.

**Why it happened:** the replication stream between the clusters is not delivering tasks fast enough, or has stopped entirely.

**What to do — check in this order:**

1. **Check [Stream Stuck](../observability/dashboards/server/namespace-failover-graceful-handover-readme.md#panel-stream-stuck-stat)** — a non-zero rate means one or more streams have stopped. A stuck stream will reconnect automatically after `history.ReplicationStreamSendEmptyTaskDuration` × `history.ReplicationReceiverLivenessMultiplier` (defaults: 1 min × 3 = **3 minutes**). Wait and watch — if the panel clears on its own, WaitReplication will resume. If it does not clear, restart the affected standby history pods (check history pod logs to find which pod is serving the stuck shard first).
2. **Check [Receiver Backlog Depth](../observability/dashboards/server/namespace-failover-graceful-handover-readme.md#panel-receiver-backlog-depth-stat) p99** — if it is near 500, the standby receiver is saturated and the sender has paused. See section [1.5](#15-is-the-standby-keeping-up-with-incoming-replication-tasks) for how to resolve it.
3. **Check [Stream Errors (gRPC)](../observability/dashboards/server/namespace-failover-graceful-handover-readme.md#panel-stream-errors--grpc-stat) and [Stream Service Errors](../observability/dashboards/server/namespace-failover-graceful-handover-readme.md#panel-stream-service-errors-stat)** — look for `"ReplicationStreamError"` in standby history pod logs for the specific cause.

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

---

#### The handover completed but `force-replication-v2` was not run first

**What you see:** the handover workflow completed and the new active cluster is serving the
namespace, but some executions are missing on the new active cluster. Whether an execution is there
depends on when it last changed, compared with when the standby was added to the namespace:

| Execution | On the new active cluster after the handover? |
|---|---|
| Closed before the standby was added to the namespace | **Missing** |
| Running, and unchanged since the standby was added (for example, waiting for a signal, a long timer, or a workflow task no SDK worker has picked up) | **Missing** |
| Running, and changed at least once since the standby was added | Present. That first change brought the whole execution over, with its full event history |
| Started after the standby was added | Present |

A describe or signal for a missing execution sent to the new active cluster returns
`workflow not found`. When the old active cluster has `dcRedirectionPolicy` set to
`all-apis-forwarding`, the same call sent to the old active cluster returns the same error, because
the old active cluster forwards it to the new active cluster.

**Why it happened:** live replication only carries changes made after the standby was added to the
namespace, so an execution with no change since then never reaches the standby. The only way to copy
such an execution to the standby is `force-replication-v2`
(see [Do you need to run the force-replication workflow?](#11-do-you-need-to-run-the-force-replication-workflow)).
The handover workflow does not copy executions either.

**Impact:** a missing closed execution does not exist on the new active cluster, so describing or
listing it there finds nothing. A missing running execution makes no progress. Its timers, activities and workflow tasks do not run on
either cluster, and signals to it fail with `workflow not found`.

**How to recover:** wait until the handover workflow has finished. While the namespace is in
HANDOVER, a backlog from `force-replication-v2` can block the drain and roll the handover back,
whatever `history.EnableReplicationTaskTieredProcessing` is set to, because backfill for running
executions is part of the drain in every release. Once the handover workflow has finished,
start `force-replication-v2` on the **old active cluster**, which is now the standby. Start it as soon as
you can, because running executions make no progress until they are copied.

```bash
temporal workflow start \
  --address <old-active-cluster-frontend> \
  --namespace temporal-system \
  --type force-replication-v2 \
  --task-queue default-worker-tq \
  --input '{
    "Namespace": "<your-namespace>",
    "Query": "",
    "ConcurrentActivityCount": 5,
    "EnableVerification": true,
    "TargetClusterName": "<new-active-cluster-name>"
  }'
```

An empty `Query` copies running and closed executions. To copy only one kind, use the `Query` values
in [Do you need to run the force-replication workflow?](#11-do-you-need-to-run-the-force-replication-workflow).

- **Run it on the old active cluster, not the new one.** `force-replication-v2` copies the executions
  it finds in the visibility store of the cluster it runs on. The new active cluster's visibility store
  does not have the missing executions, so started there it finds nothing to copy.
- **Copying works although the old active cluster is now the standby.** Copying executions does not
  depend on which cluster is active for the namespace. Measured on a test cluster with
  `force-replication-v2` started on the old active cluster after the flip, every closed execution was copied with its full event history, and
  every running execution carried on normally on the new active cluster. Pending workflow tasks and
  activities were delivered to workers, timers fired at their original time, and signals were
  accepted. No further step was needed.
- **Ignore `TotalWorkflowCount` in the workflow's progress query.** When the standby has
  `dcRedirectionPolicy` set to `all-apis-forwarding`, the count comes from the new active cluster, so it shows
  what the new active cluster already has, not what will be copied. Use `ReplicatedWorkflowCount`,
  which counts executions confirmed on the new active cluster when `EnableVerification` is `true`.
- **Check the result against the new active cluster's frontend.** When the standby has
  `dcRedirectionPolicy` set to `all-apis-forwarding`, a describe or list sent to it is answered by the active cluster.
- **Executions that namespace retention has already deleted on the old active cluster cannot be
  copied**, the same limit as in
  [Do you need to run the force-replication workflow?](#11-do-you-need-to-run-the-force-replication-workflow).
- **Executions started before the namespace was made global** can cause replication errors on the
  old active cluster afterwards. See
  [Replication errors on the old active cluster after the flip](#replication-errors-on-the-old-active-cluster-after-the-flip).
