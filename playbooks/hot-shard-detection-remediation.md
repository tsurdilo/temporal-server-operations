# Hot Shard — Detection & Remediation Playbook

**Audience**

Operators of self-hosted Temporal clusters. A hot shard can happen on **any** cluster — even one running a single namespace. On a **multi-tenant** cluster (many namespaces on one cluster) the impact can be wider, because a single shard processes executions from different namespaces, so one tenant's hot workload can also affect others that share the shard. This playbook calls out that case specifically. Applies to both SQL (PostgreSQL / MySQL) and Cassandra, except where a step is called out as SQL-only.

**When to use**

Use this playbook when you see any of these:

- Workflows are taking longer than usual to make progress, and the slowness is not spread evenly — some workflows run normally while others on the same cluster are slow.
- One history host is using far more CPU, memory, or database connections than the others, even though shards are meant to be spread evenly across hosts.
- On a multi-tenant cluster, one namespace's workflows slow down at the same time another namespace is running a heavy workload.

You can also run the detection steps below as a routine check, before any of these show up.

**What a hot shard is**

Every workflow is assigned to one history shard by hashing its namespace and workflow ID together. That assignment is fixed: the same workflow ID always maps to the same shard. A shard is a shared resource — one lock, one write pipeline, and one set of queue processors serve **every** execution on it. So when one workload overloads a shard, **every other workflow on that same shard slows down too**, even though it did nothing wrong.

This happens on any cluster: on a single-namespace cluster, one workload can slow down other workflows in the same namespace. On a multi-tenant cluster the impact can be wider, because the hash mixes in the namespace, so **a single shard processes executions from different namespaces** — and then one tenant's hot workload can also affect unrelated tenants that share the shard. That is why a hot shard is a cluster problem, not a single-user one.

The effects go beyond slow workflows. The one history host that owns a hot shard has to do far more work than the other hosts — using more CPU, memory, and database connections — so load across the history hosts becomes uneven. And it can escalate: a history host keeps ownership of a shard by regularly updating that shard's record in the database, and when the database is overloaded that update can fail or keep timing out. The host then loses ownership, the shard is dropped and loaded again — on the same or another host — and no workflow on it runs until it is loaded. So a hot shard can go from just slow to briefly stopped.

**Scope**

This playbook covers detecting a hot shard and remediating it. It does not cover general database tuning or shard count sizing for a new cluster.

**Dashboards for this playbook**

- **[Temporal Server Dashboard](../observability/dashboards/server/temporal-server-readme.md#3-persistence-requests-latencies-and-errors)** — in the **Persistence Requests, Latencies and Errors** row, two panels drive detection:
  - **Per-Shard Persistence RPS Distribution (Hot-Shard Detector)**
  - **Hottest Shard RPS**
- **[Shard IO Concurrency Dashboard](../observability/dashboards/server/shard-io-concurrency-readme.md)** — the **Shard IO Semaphore Latency** and **Shard IO Semaphore Failures** panels confirm write contention on a shard (SQL only).

---

## Contents

1. [What causes a hot shard, and what doesn't](#1-what-causes-a-hot-shard-and-what-doesnt)
2. [Detect](#2-detect)
3. [Remediate](#3-remediate)
4. [Configuration reference](#4-configuration-reference)

---

## 1. What causes a hot shard, and what doesn't

There are two kinds of hot shard. They look different and need different fixes.

**Kind 1 — many different workflow IDs crowding a shard.**
When lots of *different* workflow IDs are active at once and land on the same shard, they all compete for that shard's single lock and its writes to the database. Because many workflows are involved, this kind has the wider-spread impact. It usually comes down to **too few shards for the amount of work happening at once**: with few shards, many active workflows land on each shard no matter what their IDs are. This is the most common cause.

**Kind 2 — a single hot workflow ID.**
Because a workflow ID always maps to the same shard, pushing a lot of work through one ID concentrates it on that one shard. This shows up two ways:

- **The same workflow ID is hit hard at a high rate** — for example, restarting the same ID in a tight loop, or repeatedly calling `DescribeWorkflowExecution` on it to check whether it has finished. Every one of those operations lands on the one shard that ID maps to. (Temporal spreads load across shards by workflow ID, so a high rate is only spread out when it targets many *different* IDs.)
- **One workflow ID builds up over time** — a workflow ID that keeps calling ContinueAsNew stays on one shard, run after run. Because it only does one thing at a time, this is milder than a shard crowded with many IDs; it mainly becomes a problem during a migration of that namespace to another cluster, when all of that ID's built-up history has to move onto a single shard on the new cluster.

**What does not cause a hot shard**

These are single-workflow patterns people often suspect, but on their own they do not create a hot shard — as long as the cluster has enough shards for its workload. If you are seeing a hot shard, look to the two causes above (too few shards, or a single hot workflow ID), not these.

- **Setting many user timers in one workflow** — the durable timers you start from workflow code, such as a workflow sleep. The server keeps only one user timer outstanding per workflow at a time; it creates the next one after the current one fires. So a workflow with a thousand of these does not flood a shard. (This is about user timers specifically. Other timers, such as activity timeouts, are separate — scheduling many activities in parallel is a different pattern and is not what this covers.)
- **One large workflow doing lots of updates.** Each workflow execution is processed one operation at a time by its own lock, so a single workflow cannot flood its shard with writes. It may hold its lock longer (large state, more reloads from the database), but that shows up as workflow-lock latency, not as a hot shard.

---

## 2. Detect

### Step 1 — Confirm a hot shard exists

Open the **Per-Shard Persistence RPS Distribution (Hot-Shard Detector)** panel on the [Temporal Server Dashboard](../observability/dashboards/server/temporal-server-readme.md#3-persistence-requests-latencies-and-errors). It shows three lines — **the typical shard (p50)**, **many-shards-hot (p99)**, and **the single hottest shard (max)**.

- If **`max` sits far above `p50`**, at least one shard is doing much more work than the rest — you have a hot shard.
- If **`p99` is also far above `p50`**, it is not just one — **many** shards are hot at once (broad concentration, usually too few shards).
- If **all three lines are close together**, load is evenly spread — this is not a hot-shard problem.

> **This panel needs a busy fleet of many active shards to be reliable.** The hottest-shard line uses **`max`** on purpose: a single hot shard sits above the 99.9th percentile on a large cluster, so a p999 line would land on a normal shard and miss it. But `max` is **coarse** — it rounds up to the metric's histogram bucket, so read it as "one shard is way up there," not an exact rate. And the metric only counts shards that had traffic in the window, so on a small cluster (few shards) or a lightly-loaded one (few shards active) there are too few data points and the lines get noisy — the detector is meant for a cluster with hundreds-plus active shards.

The **Hottest Shard RPS** panel next to it shows one number — the single busiest shard's request rate right now. Compare it to a typical shard (the `p50` line on the panel to its left): if the hottest shard is doing far more than typical, you have a hot shard. (Same caveats: `max` is coarse — a ballpark, not an exact rate — and it needs a busy fleet to be meaningful.)

### Step 2 — Read the shape: one hot shard, many hot shards, or the whole cluster crowded?

The shape of the distribution narrows the cause:

| What you see | What it means | Next |
|---|---|---|
| `max` far above `p50`, but `p99` still near `p50` | **One (or a few) shards** are hot; the rest are fine. Usually a single hot workflow ID being hit hard (Kind 2). | [Find what is driving it](#find-driver) |
| `p99` **and** `max` far above `p50` | **Many** shards are hot — broad concentration or too few shards for the load (Kind 1). | [Add shards](#add-shards) |
| The whole distribution is shifted up — even `p50` is high | Every shard is crowded: too few shards for the total workload (Kind 1). | [Add shards](#add-shards) |

One case will not show here: a single workflow ID that is slowly **building up** over time — rather than being hit at a high rate — keeps a low request rate, so it does not stand out on this panel. You find that one from the workflow ID itself or at migration time.

### Step 3 — Confirm write contention (SQL)

On the [Shard IO Concurrency Dashboard](../observability/dashboards/server/shard-io-concurrency-readme.md), check **Shard IO Semaphore Latency** and **Shard IO Semaphore Failures**. Elevated latency or any failures confirm that writes are queuing inside shards — the symptom of Kind 1 contention.

<a id="find-driver"></a>
### Step 4 — Find what is driving it (the namespace or workflow ID)

You usually do not need the shard number itself. If the whole distribution is shifted up (every shard busy), there is no single culprit to chase — go to [add shards](#add-shards). If the load is concentrated, find what is aimed at that shard:

- **Which namespace:** the **Persistence Requests per Namespace and Operation** panel breaks database requests down by namespace. Find the namespace whose request rate went up when the hot shard appeared — that is the one putting load on the shard. One thing to know: the server's own internal work is labeled with the namespace name `system`, so if the busiest name is `system`, the load is coming from Temporal itself rather than from one of your namespaces.
- **Which workflow ID:** this is usually a recognizable application pattern — a loop that keeps polling `DescribeWorkflowExecution` on one workflow ID, a tight ContinueAsNew or terminate-and-restart loop, or a job that keeps restarting the same few IDs. Identify the workflow ID (or small set of IDs) behind the traffic.

### Step 5 — Get the exact shard, if you need it

Once you know a hot workflow ID, you can get its shard directly — no waiting for a log. The `tdbg` debug CLI runs the same assignment the server uses:

```bash
tdbg history-host get-shardid --namespace-id <namespace-id> --workflow-id <workflow-id> --number-of-shards <your numHistoryShards>
```

With the shard in hand, `tdbg shard describe` and `tdbg shard list-tasks` let you inspect what is on it — but note these hang on a shard that is fully stuck, so they help while the shard is busy but still moving.

A shard that has fallen far behind will also name itself in the logs, which is useful when a shard is stuck or badly behind rather than just busy. Turn the shard lag log on (no restart needed):

```yaml
history.emitShardLagLog:
  - value: true
```

The history service then writes a warning whose message is exactly `Shard queue lag exceeds warn threshold.`, with the shard ID attached as a field. A fully stuck shard shows up in the deadlock detector log at error level with the message `potential deadlock detected` (all lowercase); the shard is carried in a separate `name` field, formatted as `Shard(<id>)-shard-lock` or `Shard(<id>)-io-semaphore` — for example `Shard(64)-shard-lock`.

---

## 3. Remediate

Do two things: **limit the impact first** so the hot shard stops slowing down other work that shares it, then **fix the root cause**.

### Limit the impact first

A single overloaded shard slows down every execution that shares it — other workflows of the same or different types, and, on a cluster running more than one namespace, workflows of other namespaces too. While you work on the root cause, two server-side guardrails can cap the damage without any application change. You can also turn either one on ahead of time, as a standing guardrail, to prevent this class of hot shard before it appears.

**Guardrail 1 — slow repeated starts of the same workflow ID.** This targets the single hot workflow ID case directly. Turn on `history.enableWorkflowIdReuseStartTimeValidation`. With it on, if a new run is started on a workflow ID whose current run started less than the minimum interval ago, the server rejects the start with a resource-exhausted error (cause `BusyWorkflow`). The interval defaults to 1 second — you only set `history.workflowIdReuseMinimalInterval` if you want a different value. It covers reusing a finished workflow ID and terminate-and-restart of a running one — the fast repeated starts behind a single hot workflow ID — and it does not touch a first start, or a SignalWithStart that just signals a run that is already alive. Both settings are per namespace and take effect without a restart.

Turn it on for every namespace (uses the 1-second default):

```yaml
history.enableWorkflowIdReuseStartTimeValidation:
  - value: true
```

or for just one namespace:

```yaml
history.enableWorkflowIdReuseStartTimeValidation:
  - value: true
    constraints:
      namespace: "orders-namespace"
```

To use a spacing other than 1 second, set the interval too (also per namespace):

```yaml
history.workflowIdReuseMinimalInterval:
  - value: 2s
    constraints:
      namespace: "orders-namespace"
```

- **Off by default.** The rejected caller gets an error — that is the point: it forces the caller to slow its starts down. An application that legitimately restarts the same workflow ID faster than the interval will start seeing these errors, so set the interval to match what your workloads actually need (per namespace where they differ).
- **No burst allowance today.** This setting enforces a strict minimum spacing — it does not permit a short burst of starts closer together than the interval, so even a brief, legitimate burst of new runs on the same workflow ID is rejected. A future Temporal release adds an option to allow a limited burst above the base rate; this playbook will cover that once it ships.
- **Good to set ahead of time.** A modest interval does not affect normal workloads, so you can turn it on before any hot shard appears to keep any one workflow ID from being restarted too fast.
- The rejections appear on the **Resource Exhausted with Cause** panel (Temporal Server dashboard) as cause `BusyWorkflow` on the operations that start a run: `StartWorkflowExecution`, `SignalWithStartWorkflowExecution`, and update-with-start (`ExecuteMultiOperation`). After you turn the throttle on, a rise in `BusyWorkflow` on those operations is the throttle doing its job. (On these operations `BusyWorkflow` otherwise only shows up when the same workflow ID's current run is itself too busy to act on — the run is locked by other work or is closing — which is also a sign of a hot workflow ID.)

**Guardrail 2 — cap one namespace's database rate on a shard.** Cap how many database requests per second a single namespace is allowed to make on a single shard, using the `history.persistencePerShardNamespaceMaxQPS` dynamic config. It is set per namespace and takes effect without a restart, so a namespace running a heavy workload can no longer take over a shard's capacity and the other work on that shard recovers.

It is **off by default** (`0` means no limit). Set the same cap for every namespace:

```yaml
history.persistencePerShardNamespaceMaxQPS:
  - value: 500
```

or a different cap for one namespace:

```yaml
history.persistencePerShardNamespaceMaxQPS:
  - value: 500
    constraints:
      namespace: "orders-namespace"
```

**Choosing the value.** No metric reads a namespace's per-shard request rate back directly, so pick a starting number and tune it:

- **500 requests per second per namespace per shard is a reasonable starting point** — high enough that a normal workload never notices it, low enough that one namespace cannot take over a shard.
- After setting it, watch two panels on the **Temporal Server dashboard**, both in the **Persistence Requests, Latencies and Errors** row: the **Per-Shard Persistence RPS Distribution (Hot-Shard Detector)** panel (does the hottest shard's rate come down?) and the **Persistence Errors by Namespace and Operation** panel (set the dashboard's namespace filter to the busy namespace — when the cap starts limiting it, resource-exhausted errors appear for it here).
- Lower the value if the busy namespace still drives a shard too hard; raise it if a normal namespace starts getting throttled (its resource-exhausted errors climb and its workflows start retrying).

**Setting `history.persistencePerShardNamespaceMaxQPS` ahead of time is also a good preventative measure.** Because a well-chosen cap does not affect normal workloads, it works well as a standing guardrail so no single namespace can ever take over a shard — you do not have to wait for a hot shard to appear before setting it. If you set it ahead of time, keep the value high enough to clear your busiest legitimate namespace's needs, and use the same two panels above to confirm it is not throttling real work.

Two things to keep in mind. This is a **guardrail, not a cure** — it slows the busy namespace down to protect the rest, but it does not make the hot workload itself run any faster, so fix the root cause below as well. And the cap **limits each namespace on its own**, so it only helps when a single namespace is the heavy one. If a shard is busy because many namespaces are each doing a moderate amount, every one of them can stay under the cap while their combined load still overloads the shard — the cap never kicks in. For that situation the fix is more shards, to spread those namespaces across more shard contexts.

### Kind 1 — many different workflow IDs crowding a shard

<a id="add-shards"></a>
**Add shards (the real fix for too-few-shards).**
More shards spread the workload across more shard contexts, so fewer active workflows share each one. This is the correct fix when the whole distribution is shifted up, or when many different workflow IDs concentrate on too few shards. Changing shard count is a migration and is disruptive — plan it as a maintenance operation, not a quick tweak.

**Raise the per-shard write concurrency (SQL only).**
On PostgreSQL / MySQL, `history.shardIOConcurrency` controls how many writes a shard can have in flight at once. The default is `1` (fully serialized). Raising it eases write contention **when the database has spare capacity**. Cassandra cannot use more than `1`. Work through the [Shard IO Concurrency playbook](./shard-io-concurrency.md) before changing it — raising it when the database is already saturated makes things worse, and it does not redistribute a hot shard's load, it only lets each shard write faster.

**If the load is aimed at one or a few specific workflow IDs**, that is a hot workflow ID, not a shard-count problem — see [Kind 2 — a single hot workflow ID](#kind-2--a-single-hot-workflow-id) below for the patterns behind it and their fixes.

### Kind 2 — a single hot workflow ID

A single workflow ID always maps to one shard, so the more often you start a new run on that same ID, the more work piles onto that one shard. Adding shards does not help — that ID still maps to the same shard. The fix in every case below is the same idea: **lower the rate at which new runs are started on the same workflow ID** (and, for the read case, how often the same ID is re-read). If you are using one or a few IDs where you could use many, the simplest fix is to **spread the work across more workflow IDs**.

**Frequent ContinueAsNew.** Calling ContinueAsNew after only a little work starts a new run each time — more starts on the same ID.
**Instead:** continue only when the run's history has actually grown. Temporal sets a `continueAsNewSuggested` flag on the workflow (readable in workflow code) once history is large enough; keep the run going and continue only when that flag is true. Fewer, larger runs means fewer starts.

**Terminate-and-restart loops.** Repeatedly starting the same workflow ID with the `TerminateIfRunning` policy, or terminating and then starting again, begins a new run every cycle. This is an anti-pattern.
**Instead:** keep one long-running workflow and send it new input rather than replacing it — use SignalWithStart or UpdateWithStart to deliver work to the running run (either starts a run only if none is running). When the run needs to reset, let it ContinueAsNew on the `continueAsNewSuggested` flag.

**SignalWithStart on very short-lived runs.** SignalWithStart starts a new run only if none is running; if a run is alive, it just signals it. The problem appears only when runs are so short that each call finds none running and starts a fresh one.
**Instead:** keep the workflow running long enough that SignalWithStart signals the existing run instead of starting a new one — a long-running workflow that stays up and handles incoming signals, continuing on the `continueAsNewSuggested` flag.

**A workflow retry policy on a fast-failing workflow.** A retry starts a new run under the same ID. Setting a retry policy on a workflow is usually an anti-pattern — which is why workflows have none by default, unlike activities. A workflow that fails almost immediately and retries with little spacing starts new runs quickly.
**Instead:** usually, do not set a workflow retry policy — handle errors where they happen (activities already retry by default). If you truly need "on any error, start the steps over from the beginning," set a **backoff coefficient** so retries space out. Temporal's defaults already back off (first retry about 1 second, doubling each time); the risk is only if you override them to near-zero.

**Polling `DescribeWorkflowExecution` to wait for a result.** The read-side version: each poll reads that workflow from its shard, so polling one ID in a loop concentrates reads on its shard.
**Instead:** attach to the running workflow and wait for its result. In code, get a handle to the workflow and call its get-result method (for example `client.GetWorkflow(...).Get(...)` in Go); on the command line, run `temporal workflow result --workflow-id <workflow-id>`. One long-lived request replaces the repeated reads. Watch one thing after switching: these waits are long polls, and each namespace has a cap on how many it can hold open at once (`frontend.namespaceCount`, default 1200, per API method). If you reach it, it appears on the Temporal Server dashboard's **Resource Exhausted with Cause** panel as `ConcurrentLimit` on the `PollWorkflowExecutionHistory` operation.

### What will not help: raising `history.shardIOConcurrency`

Raising `history.shardIOConcurrency` does not help a single hot workflow ID (Kind 2). That setting lets a shard run more database writes at the same time — which is useful only when many *different* workflows on a shard are queued up waiting to write (the Kind 1 case, and only if the database has spare capacity). A single hot workflow ID does its work one step at a time no matter what, so giving its shard more write slots changes nothing for it. And in no case does this setting move load off a hot shard — it only lets each shard write faster — so it will not even out load that is unevenly spread across shards either.

### If you cannot change the workload quickly

The Kind 2 fixes are application changes, and the team that owns the workload may not be able to make them soon enough while the hot shard is affecting the rest of the cluster. As a temporary measure, consider moving that workload — its namespace — to a cluster of its own. On its own cluster it can still run hot, but it no longer shares shards with anyone else, so it stops slowing down other work. Treat this as a way to buy time to fix the pattern properly, not as the fix itself.

---

## 4. Configuration reference

| Setting | Default (self-hosted) | What it does |
|---|---|---|
| `history.enableWorkflowIdReuseStartTimeValidation` | `false` (off) | Master switch for the same-workflow-ID start throttle. On its own it enforces the default 1-second spacing; pair it with `workflowIdReuseMinimalInterval` only to change that. Per namespace. |
| `history.workflowIdReuseMinimalInterval` | `1s` | Minimum time between starts of the same workflow ID (only enforced when the switch above is on). A faster repeat start is rejected with a `BusyWorkflow` error. Per namespace. |
| `history.persistencePerShardNamespaceMaxQPS` | `0` (off) | Caps one namespace's database requests (reads + writes) on one shard. Turn it on to stop one namespace taking over a shard's capacity. |
| `history.persistenceNamespaceMaxQPS` | `0` (falls back to the per-host limit) | Caps one namespace's database requests per history host. |
| `history.persistenceMaxQPS` | `9000` | Caps all database requests per history host. |
| `history.shardIOConcurrency` | `1` | Writes in flight per shard (SQL only; Cassandra is fixed at 1). Raising it needs a history host restart. See the [Shard IO Concurrency playbook](./shard-io-concurrency.md). |
| `history.emitShardLagLog` | `false` | Turns on the per-shard warning log that names a backed-up shard. |

> On a self-hosted cluster these limits are off or generous by default, so the ones above are yours to set.
