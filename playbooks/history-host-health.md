# Temporal History Host Health — Monitoring Playbook

## References

**Dashboard:** [Temporal History Host Health Dashboard](../metrics/dashboards/server/history-health-dashboard-readme.md) — v1.3.0

| Row | What to look for |
|---|---|
| **1. History Host Health** | Which pods are in state 2 (NOT_SERVING) or 3 (DECLINED_SERVING), when the flip happened, fleet size changes, metric freshness |
| **2. Service Readiness (gRPC Health)** | Frontend pods ready — if zero, poller cannot reach frontend and `host_health` goes absent |
| **3. Persistence Health** | Latency / errors / availability spiking at the same timestamp as a `host_health == 2` flip → database layer triggered the flip |
| **4. History RPC Health** | Latency / errors spiking without a DB spike → RPC check triggered the flip; check shard ownership lost and membership change panels |
| **5. Shard Acquisition Health and Movement** | Shard acquisition latency and per-pod shard count — explains why pods stay in DECLINED_SERVING longer than expected during restarts or rebalancing |

**Alerts:** [`metrics/alerts/server/alerts-index.md`](../metrics/alerts/server/alerts-index.md) — Section 0

| Alert | Severity | Fires when |
|---|---|---|
| `0a` — History Pod Disappeared | critical | A pod stops emitting `host_health` entirely — crashed or killed. Does not appear as NOT_SERVING. |
| `0b` — History Pod Degraded | warning | Any pod reports `host_health == 2`. Cluster still functional — graceful failover window is open. |
| `0b-critical` — History Fleet Majority Degraded | critical | More than 50% of pods report `host_health == 2`. Act immediately. |
| `0c` — Metric Freshness Stale | critical | `host_health` not updated in 120s — poller is down or frontend is unreachable. |

**Related dynamic config:**

| Key | Default | Notes |
|---|---|---|
| `history.healthRPCLatencyFailure` | 500ms | RPC average latency threshold for NOT_SERVING |
| `history.healthRPCErrorRatio` | 0.90 | RPC error ratio threshold (90%) for NOT_SERVING |
| `history.healthPersistenceLatencyFailure` | 500ms | Persistence average latency threshold for NOT_SERVING |
| `history.healthPersistenceErrorRatio` | 0.90 | Persistence error ratio threshold (90%) for NOT_SERVING |
| `system.persistenceHealthSignalWindowSize` | 10s | Rolling window over which latency/error averages are computed |
| `system.persistenceHealthSignalBufferSize` | 5000 | Max samples in the rolling window |
| `frontend.historyHostErrorPercentage` | 0.50 | Combined failure proportion threshold triggering cluster-level NOT_SERVING |
| `frontend.historyHostSelfErrorProportion` | 0.05 | DECLINED_SERVING proportion threshold (min 2 pods) |

---

## Table of Contents

- [1. What `host_health` actually is](#1-what-host_health-actually-is)
- [2. Startup and shutdown health transitions](#2-startup-and-shutdown-health-transitions)
- [3. Dynamic config thresholds](#3-dynamic-config-thresholds)
- [4. Grafana dashboard](#4-grafana-dashboard)
- [5. Alerts](#5-alerts)
- [6. Diagnosing what triggered NOT_SERVING](#6-diagnosing-what-triggered-not_serving)
- [7. Cluster failover](#7-cluster-failover)
- [8. Setting up the DeepHealthCheck poller](#8-setting-up-the-deephealthcheck-poller)
- [9. Quick reference](#9-quick-reference)

---

## 1. What `host_health` actually is

The `host_health` gauge is emitted by each history pod when `DeepHealthCheck` runs on it. `DeepHealthCheck` is an admin RPC called on the frontend, which fans out to every history pod in parallel — each pod runs its own checks and records its result as a `host_health` gauge value. It is a numeric value representing that pod's health state at the time the check ran.

> **Multi-cluster replication note**: in a multi-cluster setup, `host_health` is the primary signal for making namespace failover decisions. A sustained degradation of the history fleet on the active cluster — visible as `host_health == 2` or `== 3` on a majority of pods — is the key indicator that failing over to a standby cluster should be considered. See section 7 for the full failover signal stack.

> **⚠️ This metric is absent by default.** `host_health` is only emitted when `DeepHealthCheck` is explicitly called on each history pod. There is no internal background polling loop in the Temporal server that drives this. If nothing in your infrastructure is calling `AdminHandler.DeepHealthCheck` on the frontend, this metric will not exist in your metrics store and all queries and alerts based on it will produce no data. See section 8 for how to set up the required poller.

### Health state values

| Value | State | Meaning |
|---|---|---|
| `0` | `UNSPECIFIED` | Not emitted explicitly — treat as missing data if observed |
| `1` | `SERVING` | All checks passed — pod is healthy and ready |
| `2` | `NOT_SERVING` | RPC or persistence threshold exceeded, or gRPC health server unreachable — pod is degraded |
| `3` | `DECLINED_SERVING` | gRPC health server responded non-SERVING — pod is starting up, shutting down, or stuck in shard acquisition |
| `4` | `INTERNAL_ERROR` | Frontend has no membership ring registered for the history service — this is a misconfiguration, not a transient state. No per-pod checks were attempted. Treat as a deployment problem requiring immediate investigation. |

The `host_health` Prometheus gauge only ever records `1`, `2`, or `3`. Value `4` is never written to the gauge — it is only returned in the `DeepHealthCheck` RPC response to the caller when the frontend cannot resolve history pod membership.

### What each history pod checks when polled

When the frontend fans out `DeepHealthCheck` to a history pod, that pod runs 5 checks against itself and records the result as `host_health`. Check 1 has two distinct failure paths:

Each history pod has an in-process gRPC health component that tracks whether the pod is SERVING or NOT_SERVING — this is what gets flipped during startup and shutdown (see section 2). Check 1 reads its status. There are two paths:

- **`Check()` returns an error** — the in-process health component itself errored, which is a degenerate case and should not happen under normal conditions. Records `NOT_SERVING (2)` immediately and returns. Checks 2–5 are skipped.
- **`Check()` succeeds but status is non-SERVING** — the pod has marked itself not ready (startup or shutdown in progress). Sets `overallState = DECLINED_SERVING (3)` and continues running checks 2–5. Checks 2–5 can still detect their own failures, but cannot change `overallState` from `3` to `2` — `DECLINED_SERVING` always wins.

In practice, path 1b is the one you will see during normal startup and shutdown. Path 1a is a degenerate failure.

If all checks pass, the pod records `host_health = 1` (SERVING). The table below shows what each check tests and what `host_health` value gets recorded if that check fails:

| Order | Check | Failure condition | `host_health` recorded |
|---|---|---|---|
| 1a | gRPC health — call error | In-process health component errored (degenerate) | `2` (NOT_SERVING) — remaining checks skipped |
| 1b | gRPC health — non-SERVING status | Pod marked itself not ready (startup/shutdown) | `3` (DECLINED_SERVING) — remaining checks still run |
| 2 | RPC average latency | Rolling average latency of history RPC calls exceeds `history.healthRPCLatencyFailure` (default **500ms**) | `2` (NOT_SERVING) |
| 3 | RPC error ratio | Rolling error ratio of history RPC calls exceeds `history.healthRPCErrorRatio` (default **0.90**, i.e. 90% of calls failing) | `2` (NOT_SERVING) |
| 4 | Persistence average latency | Rolling average latency of persistence calls exceeds `history.healthPersistenceLatencyFailure` (default **500ms**) | `2` (NOT_SERVING) |
| 5 | Persistence error ratio | Rolling error ratio of persistence calls exceeds `history.healthPersistenceErrorRatio` (default **0.90**, i.e. 90% of calls failing) | `2` (NOT_SERVING) |

The rolling window for checks 2–5 defaults to **10 seconds** (`system.persistenceHealthSignalWindowSize`), buffering up to **5000 samples** (`system.persistenceHealthSignalBufferSize`). Averages are computed over this window at the time `DeepHealthCheck` is called.

**Important**: the default thresholds are deliberately conservative — 90% error ratio and 500ms average latency. These are designed to catch severe degradation, not transient spikes. If a pod flips to `NOT_SERVING` (2) under these defaults, something is genuinely wrong.

### When is `DeepHealthCheck` invoked?

**`DeepHealthCheck` has no internal background polling loop.** It is an on-demand RPC only. The `host_health` metric is emitted only when something explicitly calls `DeepHealthCheck` on a history pod.

The `HealthInterceptor` — which guards every frontend `WorkflowService` and `OperatorService` request — is completely separate. It only controls whether the frontend itself accepts requests and has nothing to do with history health or `host_health`.

**`healthCheckerImpl.Check`** — the fan-out to all history hosts — is only wired to `AdminHandler.DeepHealthCheck`. It runs when something calls that admin API endpoint explicitly.

**Practical implication**: if `host_health` data is present in your metrics, something in your environment is calling `AdminHandler.DeepHealthCheck` on a schedule (monitoring infrastructure, Kubernetes probes, or similar). If nothing calls it, the metric is absent or stale. There is no Temporal server component that polls this internally on a timer — confirm with your infrastructure what is driving the calls and at what interval, as this determines how fresh the metric is for alerting.

**Recommended polling interval**: **15 seconds** is a good default. The key rule is: keep the polling interval at or below the signal window size (`system.persistenceHealthSignalWindowSize`, default 10s) so you don't miss a degradation event that comes and goes between polls. 15s is slightly above the 10s default window, which is fine in practice — genuine degradation persists long enough to be caught. Polling faster than the window size gives diminishing returns since consecutive polls read largely overlapping data. If you widen the signal window (e.g. to 60s to reduce flapping), increase the polling interval to match — a 30–60s poll pairs well with a 60s window. The other constraint is your alerting SLA — polling every 15s with a 2-cycle `for` condition on your alert gives you detection within ~30 seconds of sustained degradation.

---

## 2. Startup and shutdown health transitions

Each history pod tracks its own readiness state internally — it starts as not ready when it boots, and only marks itself ready once it has fully acquired its shards. This is what drives `host_health = 3` during startup and `host_health = 3` during shutdown. This section explains exactly when and why those transitions happen.

### The asymmetric health transition pattern

The history service deliberately uses different rules for becoming healthy vs unhealthy:

- **Becoming unhealthy**: immediate — any failure triggers it instantly
- **Becoming healthy**: requires sustained proof — multiple conditions must be met plus a stabilization delay

This prevents a pod from briefly passing a health check before it has fully initialized, which would cause it to receive traffic it cannot handle.

### Startup sequence

```
Pod starts → gRPC health = NOT_SERVING → host_health = 3
    ↓
InitialShardsAcquired() completes:
  - pod has joined membership ring (owns at least one shard)
  - per-shard readiness check passes for all owned shards
    (GetShardByID + GetEngine + AssertOwnership per shard)
    ↓
5 second stabilization delay
    ↓
gRPC health = SERVING → host_health = 1
```

During the entire startup phase — however long shard acquisition takes — the pod emits `host_health = 3`. There is no suppression window and no timeout on shard acquisition. For a large cluster (e.g. 2048 shards across 14 history pods, ~146 shards per pod), acquiring all expected shards takes non-trivial time. The pod stays at `3` until acquisition completes and the 5s stabilization passes.

**Important for failover decisions**: `host_health = 3` on a pod is the same value whether the pod is in normal startup, gracefully shutting down, or stuck unable to acquire shards. The metric alone cannot tell you which. Before treating `host_health = 3` as a reason to failover, cross-reference:
- **Is a deploy or rolling restart in progress?** — expected on some pods.
- **How long has the pod been at `3`?** — normal startup on a large cluster can take several minutes. A pod stuck at `3` for 10+ minutes outside of a deploy window needs investigation.
- **`history.acquireShardConcurrency`** (default **10**) — controls how many shards are acquired in parallel. If startup is consistently slow, increasing this can help.
- **`acquire_shards_latency` and `acquire_shards_count` metrics** — track shard acquisition timing directly and are a better signal for "is this pod actually making progress" than `host_health` alone.

### Shutdown sequence

Shutdown is immediate — no conditions, no delay. The pod flips gRPC health to `NOT_SERVING` instantly. Any `DeepHealthCheck` call during or after shutdown sees Check 1b fire and records `host_health = 3`.

### Practical implications for alerting

| Scenario | `host_health` value | Expected? |
|---|---|---|
| Pod starting up, shard acquisition in progress | `3` | Yes — normal until `InitialShardsAcquired` + 5s completes |
| Pod fully started, all shards acquired + 5s passed | `1` | Yes — healthy |
| Pod shutting down gracefully | `3` | Yes — normal during rolling restarts |
| Pod in crashloopbackoff | `3` | Abnormal — pod never reaches `SERVING` |
| Pod healthy but gRPC health server unreachable | `2` | Abnormal — investigate pod |
| Pod healthy but persistence degraded | `2` | Abnormal — investigate DB |
| Pod healthy but RPC latency/errors high | `2` | Abnormal — investigate traffic or shard contention |

**Distinguishing crashloop from normal startup**: a pod stuck at `host_health = 3` for more than a few minutes outside of a deploy window is likely crashlooping or stuck in shard acquisition. Check pod restart count in Kubernetes alongside the `host_health` timeline.

- **Crashlooping** (restart count increasing) — investigate pod logs for the root cause first. If a majority of pods are affected and not recovering, consider namespace failover to the standby cluster.
- **Stuck in shard acquisition** (restart count stable, pod running but not progressing) — failover is unlikely to help if the cause is infrastructure-level (e.g. database connectivity issues), since the standby cluster persistence layer may be affected too. Investigate `acquire_shards_latency` and DB health before failing over.

---

## 3. Dynamic config thresholds (all tunable at runtime without restart)

### Per-host thresholds (control when a single pod flips to NOT_SERVING)

| Dynamic config key | Default | Description |
|---|---|---|
| `history.healthRPCLatencyFailure` | `500` (ms) | RPC average latency threshold |
| `history.healthRPCErrorRatio` | `0.90` | RPC error ratio threshold |
| `history.healthPersistenceLatencyFailure` | `500` (ms) | Persistence average latency threshold |
| `history.healthPersistenceErrorRatio` | `0.90` | Persistence error ratio threshold |

### Signal window config (control what the averages in checks 2–5 are computed over)

| Dynamic config key | Default | Description |
|---|---|---|
| `system.persistenceHealthSignalWindowSize` | `10s` | Time window over which RPC and persistence latency and error averages are computed |
| `system.persistenceHealthSignalBufferSize` | `5000` | Maximum number of data points buffered per signal key within that window |

Tuning guidance: the default 10s window means `host_health` reacts quickly to sudden spikes but can also trip on short bursts. If you are seeing flapping (pods briefly going to `NOT_SERVING` and recovering within seconds), consider widening the window to `30s` or `60s` to smooth out transient spikes before tuning the thresholds themselves. Increase the buffer size if you have very high RPC or persistence call rates and want the averages to be statistically representative within the window.

**If you widen the signal window, increase your polling interval to match.** The key rule is: keep the polling interval at or below the window size so you don't miss a degradation event that comes and goes between polls. For example, a 60s window pairs well with a 30–60s poll interval — polling faster than that just re-reads overlapping data with little added value. See the polling interval guidance in section 8 for the full reasoning.

### Fleet-level aggregation thresholds

When you call `DeepHealthCheck` on the frontend, it fans out to every history pod, collects each pod's individual `host_health` result, and then evaluates the fleet as a whole — returning a single aggregated state to the caller. These thresholds control when that fleet-level verdict flips from healthy to degraded. They are separate from the per-pod thresholds above: the per-pod thresholds determine what each pod reports, while these thresholds determine what the frontend concludes about the fleet overall.

**Check 1 — DECLINED_SERVING proportion (runs first):**

```
hostDeclinedServingProportion = DECLINED_SERVING_count / total_hosts
if hostDeclinedServingProportion > hostDeclinedServingProportion_threshold:
    → frontend returns DECLINED_SERVING
    → logs: "health check exceeded host declined serving proportion threshold"
```

| Dynamic config key | Default | Notes |
|---|---|---|
| `frontend.historyHostSelfErrorProportion` | `0.05` (5%) | Minimum of 2 hosts must be `DECLINED_SERVING` regardless of proportion — enforced by `ensureMinimumProportionOfHosts`. For a 14-pod fleet, effective minimum threshold is `2/14 = 14.3%` |

**Check 2 — Combined failure proportion (runs second):**

```
failedHostProportion = (NOT_SERVING_count + UNSPECIFIED_count + INTERNAL_ERROR_count + other_count) / total_hosts
combinedProportion = failedHostProportion + DECLINED_SERVING_proportion
if combinedProportion > hostFailurePercentage_threshold:
    → frontend returns NOT_SERVING
    → logs: "health check exceeded host failure percentage threshold"
```

| Dynamic config key | Default | Notes |
|---|---|---|
| `frontend.historyHostErrorPercentage` | `0.50` (50%) | Counts all non-SERVING, non-DECLINED_SERVING pods (`NOT_SERVING`, `UNSPECIFIED`, `INTERNAL_ERROR`, other) plus `DECLINED_SERVING` pods combined. Unreachable pods are set to `NOT_SERVING` and counted in the non-SERVING bucket |

**Important nuances from the code:**
- If the membership resolver returns zero history hosts, the frontend immediately returns `NOT_SERVING` — this happens before any pings are attempted and produces no log message, just the state
- `UNSPECIFIED` (value 0) counts as a failure — it falls into the `default:` case of the aggregation switch alongside `NOT_SERVING` and `INTERNAL_ERROR`
- `failed to ping deep health check` log → that pod's response is explicitly set to `NOT_SERVING` (not UNSPECIFIED) → counts toward `failedHostCount`
- The combined 50% check includes pods in `DECLINED_SERVING` — so a cluster doing a large rolling restart where many pods are simultaneously draining can trip the `NOT_SERVING` threshold even if no individual pod has a health problem

**For a 14-pod history fleet:**

| Scenario | Pods in state | Proportion | Threshold crossed? |
|---|---|---|---|
| 1 pod `DECLINED_SERVING` | 1/14 = 7.1% | Below minimum-2 rule | No |
| 2 pods `DECLINED_SERVING` | 2/14 = 14.3% | > 5% AND ≥ 2 pods | Yes → frontend `DECLINED_SERVING` |
| 7 pods any unhealthy state | 7/14 = 50.0% | = 50%, not > 50% | No — condition is strictly `>` |
| 8 pods any unhealthy state | 8/14 = 57.1% | > 50% | Yes → frontend `NOT_SERVING` |

To tighten thresholds (catch degradation earlier), update dynamic config. The values below are examples only — set them based on your cluster's observed baseline latency and error rates, not as fixed targets:

```yaml
history.healthPersistenceLatencyFailure:
  - value: 200   # example only — adjust to your baseline

history.healthPersistenceErrorRatio:
  - value: 0.50  # example only — adjust to your baseline

history.healthRPCLatencyFailure:
  - value: 200

history.healthRPCErrorRatio:
  - value: 0.50
```

Tradeoff: tighter thresholds mean earlier signal but higher sensitivity to transient spikes. Start by observing your cluster's normal latency and error rate ranges before changing these.

---

## 4. Grafana dashboard

A pre-built Grafana dashboard covering all `host_health` panels — pod counts by state, percentage breakdown, per-pod state, metric freshness, and fleet size change — is available in the [Temporal History Host Health Dashboard](../metrics/dashboards/server/history-health-dashboard.json). A full panel-by-panel explanation is in the [dashboard readme](../metrics/dashboards/server/history-health-dashboard-readme.md).

Use the dashboard as your starting point. It includes a **Metric Freshness** panel that shows how long ago `host_health` was last updated — this is your signal that the poller has stopped calling `DeepHealthCheck`, which is a silent failure mode that leaves all other panels showing stale data.

---

## 5. Alerts

Alerting definitions for `host_health` are maintained in the [server alerts index](../metrics/alerts/server/alerts-index.md) under **Section 0 — History Host Health**. Three alerts are defined:

| Alert | Severity | What it catches |
|---|---|---|
| **0a — History Pod Disappeared** | critical | A pod stopped emitting `host_health` entirely — crashed or killed. Does not show as NOT_SERVING; requires its own alert. |
| **0b — History Pod Degraded** | warning | Any pod reporting `host_health == 2` (NOT_SERVING). Early signal — cluster still functional, graceful failover window is open. Act now. |
| **0b-critical — History Fleet Majority Degraded** | critical | More than 50% of pods reporting `NOT_SERVING`. Cluster severely degraded — initiate failover for all global namespaces immediately. Graceful failover may no longer be possible if the cluster stops responding. |
| **0c — Metric Freshness Stale** | critical | `host_health` hasn't been updated in 120s — the poller is down or can't reach the frontend. |

All alerts are required. The warning/critical split on 0b is intentional — 0b-warning gives you the window to initiate a graceful namespace failover (handover) before the cluster degrades further. Waiting for 0b-critical means you may only have forced failover available, with potential replication task loss.

---

## 6. Diagnosing what triggered NOT_SERVING

**Typical flow**: Alert 0b warning fires → open the [History Host Health Dashboard](../metrics/dashboards/server/history-health-dashboard.json) → the dashboard shows you which pods are affected and when the flip happened → use the rows below to identify the cause. If 0b-critical fires instead, skip straight to section 7 (failover).

`host_health == 2` tells you a pod is degraded but not which check fired. **The dashboard is your primary diagnostic tool** — it has dedicated rows for each failure category. Logs are a follow-up when the dashboard doesn't show a clear cause.

### Step 1 — Open the History Host Health Dashboard

Note the timestamp when `host_health` flipped to `2` on the affected pods. Then check each row:

**Database issues (checks 4–5):** Open the **Persistence Health** row. If **Persistence Latency**, **Persistence Errors by Type**, or **Persistence Availability** spiked at the same timestamp → database layer triggered the flip. Investigate your DB and connection pool at that time.

**History RPC issues (checks 2–3):** Open the **History RPC Health** row. If **History Service Latency** or **History Service Errors by Type** spiked without any DB spike → RPC check fired. Check the **Shard Acquisition Health** row for concurrent shard movement or membership changes.

**No spike in either row:** gRPC health check fired (check 1a — degenerate pod failure). Investigate the pod directly via Kubernetes logs.

### Step 2 — Check frontend logs if cause is still unclear

These log messages are emitted at **WARN** level by the **frontend** service. Ensure your frontend log level is set to `warn` or lower to see them.

| Log message | Meaning |
|---|---|
| `failed to ping deep health check` | Frontend could not reach a history pod during the fan-out. That pod's response is set to `NOT_SERVING` (2) and counts toward the combined failure proportion. |
| `health check exceeded host declined serving proportion threshold` | `DECLINED_SERVING` pod count / total pods exceeded `frontend.historyHostSelfErrorProportion` (default 5%, minimum 2 pods) |
| `health check exceeded host failure percentage threshold` | Combined (`NOT_SERVING` + `UNSPECIFIED` + `DECLINED_SERVING`) / total pods exceeded `frontend.historyHostErrorPercentage` (default 50%) |

### Diagnosing host_health = 3 (DECLINED_SERVING)

`host_health = 3` means the pod marked itself not ready. This is normal during startup and shutdown. Three situations produce it:

**1. Normal startup** — pod still acquiring shards. Expected from pod start until `InitialShardsAcquired` + 5s. Check the **Shard Acquisition Health** row in the dashboard to see acquisition progress.

**2. Normal shutdown / rolling restart** — pod received shutdown signal. Expected during deploys. Confirm via deploy timing.

**3. Crashloopbackoff** — pod restarting repeatedly, never completes `InitialShardsAcquired`. Check Kubernetes pod restart count — if increasing, it's a crashloop. Check pod logs for panic or shard acquisition errors.

---

## 7. Cluster failover

`host_health` is a cluster-wide metric — it reflects the health of the history fleet as a whole, not any individual namespace. A degraded history pod affects all namespaces running on that cluster equally. When `host_health` signals fleet-level degradation, the decision is which global namespaces to fail over to a standby cluster — not whether a specific namespace is affected.

A history pod can show as ready in Kubernetes (shards acquired, gRPC health `SERVING`) and still have degraded history health if persistence latency or RPC error ratios exceed thresholds. Pod readiness alone is not sufficient for a failover decision. The `host_health` metric captures this deeper degradation.

**What pod readiness actually means per service:**

| Service | Becomes ready when | What readiness tells you |
|---|---|---|
| **Frontend** | Joined membership ring | Pod is up and can accept requests |
| **History** | All expected shards acquired + 5s stabilization | Pod is up and owns its full shard set |
| **Matching** | Immediately on startup — no conditions | Pod is up and handler started — **no deeper readiness guarantee** |

Note: a matching pod showing ready does not mean it has loaded any task queues. History readiness is the most meaningful of the three for operational decisions.

**Failover is a manual decision.** There is no automatic cross-cluster failover triggered by any of these health signals.

**Failover granularity is always per namespace.** Temporal has no single "fail over the entire cluster" operation — even when the whole history fleet is down, you initiate failover namespace by namespace via `UpdateNamespace`. Only **global namespaces** (those with replication configured to a standby cluster) can be failed over. Local namespaces have no standby and cannot be failed over. If your entire cluster is affected, you should fail over **all global namespaces** — not just one.

### The complete signal stack for a failover decision

| Layer | Signal | Meaning |
|---|---|---|
| 1 | Frontend `kube_pod_status_ready` | Frontend is reachable — prerequisite for everything else |
| 2 | `host_health` metric present and fresh | Poller is reaching frontend, `DeepHealthCheck` fan-out is working |
| 3 | `host_health == 2` or `== 3` on majority of history pods | History fleet is degraded — `2` means RPC/persistence threshold breach, `3` means gRPC health non-SERVING (startup/shutdown/crashloop) |

If layer 1 fails (frontend pods down), layer 2 will also fail (no metric emission). Absence of `host_health` data combined with frontend pods not ready is itself a failover signal. Alert 0c in the [server alerts index](../metrics/alerts/server/alerts-index.md) covers metric staleness detection.

### Graceful vs forced failover — why timing matters

Temporal supports two failover modes and the distinction is critical:

**Graceful failover (namespace handover)**: the namespace enters `REPLICATION_STATE_HANDOVER` on the active cluster. New workflow tasks are blocked, the active cluster drains all pending replication tasks to the standby, and once the standby has caught up the active role transfers. No in-flight replication tasks are lost.

**Forced failover**: the standby cluster immediately takes over without waiting for replication drain. Any replication tasks that had not yet reached the standby are lost.

**If you wait until the history fleet is completely down, graceful failover is no longer possible** — there is no active cluster left to drain tasks. You are forced to do a forced failover and accept potential data loss.

This is why `host_health` monitoring matters for failover: it gives you early warning while the cluster is degraded but still running, giving you the window to initiate a graceful handover before the situation becomes a forced failover.

### When to consider initiating failover

**Alert 0b fires (warning — any pod NOT_SERVING)**: the cluster is still functional. This is your window for a graceful failover. Investigate the root cause — if it is confirmed infrastructure and not recovering, initiate graceful failover (handover) for all global namespaces now, before the situation escalates.

**Alert 0b-critical fires (> 50% of pods NOT_SERVING)**: the cluster is severely degraded. Initiate failover for all global namespaces immediately. If the cluster is still partially responsive, graceful failover may still be possible — attempt it first. If the cluster is no longer responding, forced failover is the only option and some replication tasks may be lost.

**Alert 0a fires (pod disappeared) or 0c fires (metric stale)** alongside other degradation signals: treat as equivalent to 0b-critical — assume the worst and act accordingly.

See the [server alerts index](../metrics/alerts/server/alerts-index.md) section 0 for full alert definitions.

---

## 8. Setting up the DeepHealthCheck poller

Because `host_health` is only emitted when `AdminHandler.DeepHealthCheck` is called explicitly (no internal polling loop), you need something in your infrastructure calling it on a regular schedule for the metric to be useful.

A single call to `AdminHandler.DeepHealthCheck` on the **frontend** fans out to all history hosts via `healthCheckerImpl.Check` and returns an aggregated state. You do not need to call individual history pods.

The admin service is exposed on the same port as the frontend (default `7233`).

### Example Go poller

```go
package main

import (
	"context"
	"log"
	"os"
	"os/signal"
	"syscall"
	"time"

	adminservice "go.temporal.io/server/api/adminservice/v1"
	"google.golang.org/grpc"
	"google.golang.org/grpc/credentials/insecure"
)

func main() {
	frontendAddr := getenv("FRONTEND_ADDR", "temporal-loadbalancing:7233")
	pollInterval := mustParseDuration(getenv("POLL_INTERVAL", "15s"))

	conn, err := grpc.NewClient(frontendAddr, grpc.WithTransportCredentials(insecure.NewCredentials()))
	if err != nil {
		log.Fatalf("failed to connect: %v", err)
	}
	defer conn.Close()

	client := adminservice.NewAdminServiceClient(conn)

	ctx, cancel := signal.NotifyContext(context.Background(), os.Interrupt, syscall.SIGTERM)
	defer cancel()

	ticker := time.NewTicker(pollInterval)
	defer ticker.Stop()

	log.Printf("starting DeepHealthCheck poller against %s, interval %s", frontendAddr, pollInterval)

	for {
		select {
		case <-ctx.Done():
			log.Println("poller stopped")
			return
		case <-ticker.C:
			pollCtx, pollCancel := context.WithTimeout(ctx, 10*time.Second)
			resp, err := client.DeepHealthCheck(pollCtx, &adminservice.DeepHealthCheckRequest{})
			pollCancel()
			if err != nil {
				log.Printf("DeepHealthCheck error: %v", err)
				continue
			}
			log.Printf("cluster state: %s", resp.State)
			for _, svc := range resp.Services {
				for _, host := range svc.Hosts {
					log.Printf("  service=%s host=%s state=%s", svc.Service, host.Address, host.State)
				}
			}
		}
	}
}

func getenv(key, fallback string) string {
	if v := os.Getenv(key); v != "" {
		return v
	}
	return fallback
}

func mustParseDuration(s string) time.Duration {
	d, err := time.ParseDuration(s)
	if err != nil {
		log.Fatalf("invalid duration %q: %v", s, err)
	}
	return d
}
```

**`go.mod`:**
```
module temporal-health-poller

go 1.26.2

require (
	go.temporal.io/server v1.31.0
	google.golang.org/grpc v1.79.3
)
```

**Dockerfile:**
```dockerfile
FROM golang:1.26-alpine AS builder
RUN apk add --no-cache git
WORKDIR /app
COPY go.mod go.sum main.go ./
RUN CGO_ENABLED=0 GOOS=linux go build -o poller .

FROM alpine:latest
RUN apk add --no-cache ca-certificates
COPY --from=builder /app/poller /poller
ENTRYPOINT ["/poller"]
```

**Notes:**
- If TLS is required, replace `insecure.NewCredentials()` with your cluster's TLS configuration
- The polling interval determines metric freshness for alerting — set it based on your alerting SLA requirements; configure via `POLL_INTERVAL` env var
- `resp.State` is the aggregated fleet-level state evaluated by `healthCheckerImpl.Check` logic documented in section 3; the per-host detail is in `resp.Services[].Hosts[]`
- Before relying on `host_health` alerts, confirm what is currently calling `DeepHealthCheck` in your environment and at what interval

---

## 9. Quick reference — what to check when `host_health` alerts

**First action for any alert: open the [History Host Health Dashboard](../metrics/dashboards/server/history-health-dashboard.json).** The dashboard tells you which pods are affected, when it happened, and which row (Persistence Health, History RPC Health, Shard Acquisition) shows the cause. Sections 6 and 7 of this playbook have the full diagnosis and failover guidance.

| Alert | Severity | Immediate action |
|---|---|---|
| **0b** — any pod `host_health == 2` | warning | Open dashboard → identify cause row → if infrastructure confirmed and not recovering, initiate graceful failover for all global namespaces now (see section 7) |
| **0b-critical** — >50% pods `host_health == 2` | critical | Initiate failover for all global namespaces immediately — graceful if cluster still responding, forced if not (see section 7) |
| **0a** — pod stopped emitting `host_health` | critical | Check Kubernetes pod status — if multiple pods disappeared treat as 0b-critical |
| **0c** — metric stale >120s | critical | Poller is down or frontend unreachable — check frontend pods first, then treat as 0b-critical if frontend is down |

> **Failover procedures** (graceful namespace handover and forced failover) will be covered in dedicated playbooks — coming soon.
