# Archival Backend Outage — Detection & Recovery Playbook

**Audience**

Operators of self-hosted Temporal clusters that use the Archival feature (S3, GCP/GCS, or a custom archival provider).

**When to use**

A running cluster is serving a production workload, and the configured archival backend becomes **unavailable and does not recover quickly** — for example the archival endpoint is unreachable, misconfigured, or persistently erroring while a large workload keeps running. Use this playbook to detect that condition early and stop it from destabilizing the cluster.

This playbook is specifically for a **sustained** archival outage — one that is not resolving on its own. It is **not** for brief or intermittent archival failures (occasional timeouts, transient throttling): those self-heal via retries and need no operator action. The concern is a **prolonged** failure — one that keeps failing under load instead of recovering on its own.

**Related dashboard & alerts**

- **Dashboard:** use the **Archival Health** row of the [Temporal Server Dashboard](../metrics/dashboards/server/temporal-server-readme.md#21-archival-health). Two panels there drive this playbook: **Signal 1 — Archival Attempt Error Rate** (detect the failure early) and **Signal 2 — History Task DLQ Writes & Write Failures** (see it escalate).
- **Alert 81 — Archival Backend Failing** ([runbook](../metrics/alerts/server/runbooks/81-archival-backend-failing.md)): the earliest alert — fires on a sustained archival error rate, ~1h before any DLQ pressure. This is the one that should trigger this playbook.
- **Alert 82 — History Task DLQ Write Failures** ([runbook](../metrics/alerts/server/runbooks/82-history-task-dlq-write-failures.md)): the later alert — the DLQ writes themselves are failing. If this fires, the situation has already progressed; go straight to the [recommended action](#5-recommended-action--pause-archival).

Both alerts are part of the server Essential Alert Set.

---

## Contents

1. [How the history task DLQ works](#1-how-the-history-task-dlq-works)
2. [Impact of a sustained archival failure on a production cluster](#2-impact-of-a-sustained-archival-failure-on-a-production-cluster)
3. [Detection](#3-detection)
4. [Alert on archival failure](#4-alert-on-archival-failure)
5. [Recommended action — pause archival](#5-recommended-action--pause-archival)
6. [Recover — when the backend is healthy again](#6-recover--when-the-backend-is-healthy-again)

---

## 1. How the history task DLQ works

The **history task DLQ (dead-letter queue)** is a built-in Temporal mechanism, present on **both Cassandra and SQL (PostgreSQL / MySQL)** persistence — it is **not** Cassandra-specific. It lives entirely in Temporal's **persistence layer**: it is a set of **internal database tables** — `queue_messages` (the dead-lettered task records) and `queues` (queue metadata) — where failed history tasks are parked for an operator to inspect or replay later. It is part of the server's storage; **workers never poll it or see it** (it is unrelated to the task queues your workers use).

It is designed for the **occasional** bad task — a "poison pill" here and there — **not for a flood**, because it is a **single, unsharded queue.** Picture it as **one checkout lane** (a single mailbox slot): all of a queue's dead-lettered records live under **one partition key**, in the same physical spot, and must be written **one at a time, in order** — there is no spreading the writes across lanes or database nodes. Each write is also a slow, coordinated write: on **Cassandra** a *lightweight transaction* (LWT / Paxos); on **SQL** a *transaction that locks the queue row* (`SELECT … FOR UPDATE`). Fine for a trickle; a bottleneck at volume. *(This "partition" is a database-storage concept — unrelated to Temporal **task-queue** partitions, which spread worker load across hosts. The DLQ does no such spreading; it always uses a single partition.)*

**Sending a task to the DLQ** removes it from active processing and parks it there. It is **not retried automatically** — an operator must later **redrive** it (`tdbg dlq merge`) or purge it. For an archival task specifically, this means the workflow is **neither archived nor deleted** (retention cleanup) until the task is redriven.

This playbook concerns **archival** tasks specifically. (The same DLQ mechanism also handles Temporal's other internal history tasks — transfer, timer, visibility, and outbound — but those are outside the scope of this playbook; replication tasks use a separate replication DLQ.)

**The two DLQ dynamic configs relevant here** (defaults shown — you do **not** need to change either for the DLQ to work):

| Dynamic config | Default | What it controls |
|---|---|---|
| `history.TaskDLQEnabled` | `true` | Master switch for the history task DLQ. Must be **true** (the default) for Temporal to dead-letter tasks at all — **on by default; no action needed**. |
| `history.TaskDLQUnexpectedErrorAttempts` | `70` | Number of *unexpected-error* attempts a task makes before it is sent to the DLQ (≈ 1 hour at the default backoff). Archival's backend failures count as unexpected errors, so they follow this path. |

> These are **service-wide** settings — they apply to all history task types, not per task type.

---

## 2. Impact of a Sustained Archival Failure on a Production Cluster

When the archival backend is unreachable, every closed workflow's archival task fails and **retries**. Each failure counts as an *unexpected error*, and after `history.TaskDLQUnexpectedErrorAttempts` such attempts (default **70** — roughly an hour per task) the task is sent to the **[history task DLQ](#1-how-the-history-task-dlq-works)**. Because that DLQ is a **single lane** and a failed DLQ write **keeps retrying until it succeeds** (the task is never dropped), a sustained archival outage under load **can turn** the DLQ into a **self-amplifying write bottleneck that back-pressures the whole database**.

**How severe this gets depends on the workload — specifically, how many executions close around the same time.** Some use cases complete a large number of executions in bursts (many workflows finishing close together); others complete them steadily at a low rate. In the bursty case, all of those archival tasks begin failing and retrying on roughly the same schedule, so they also reach the DLQ threshold (`history.TaskDLQUnexpectedErrorAttempts`, default **70** attempts) at roughly the same time — producing, perhaps ~70–90 minutes into the outage, a **large spike of archival tasks that all need to be written to the DLQ at once**. A workload that completes executions steadily, at a low rate, produces a much smaller and more spread-out effect.

Nothing throttles or spreads these DLQ writes: unlike archival's *reads* (which are rate-limited), the DLQ *writes* are **not rate-limited, batched, or jittered** — so a burst all hits the single-lane DLQ together.

If that burst is large enough — again, this depends on the use case — and **especially if the database is already busy serving the production workload**, the flood of single-lane DLQ writes can add significant pressure and start to **overwhelm the database**.

If the database does get overloaded, the **DLQ writes themselves can start to fail** (time out). A failed DLQ write is **never given up on**: Temporal retries it on a backoff — starting at ~1 second and growing to at most ~3 minutes between attempts (this backoff is fixed and **not currently tunable**) — and keeps retrying **until it succeeds**. With many tasks stuck retrying their DLQ writes, this **adds still more load to the already-struggling database**. On **Cassandra** these failures appear as LWT/Paxos timeouts (`received only 0 responses`); on **SQL** as row-lock contention and transaction/connection-pool exhaustion.

This pressure is **not limited to the DLQ writes themselves.** When the database is contended, persistence latency can rise for **all history task processing** on the cluster — not just archival. In effect, the database now has to serve, at the same time:
- the **normal production workload** — ongoing history task processing across the cluster;
- a **potentially large burst of DLQ writes** — the failed archival tasks being dead-lettered; and
- a **potentially large burst of DLQ-write failures that are continuously retried** — per the backoff described above.

(Archival's own tasks also keep retrying throughout, and each attempt reloads the workflow's state and history — adding read load as well.) All together, this situation can potentially drive up **history host CPU utilization**.

> **Persistence rate limits do not help here — and `ResourceExhausted` alerts will not catch it.** Temporal's persistence QPS limits (`history.persistenceMaxQPS`, `persistencePerShardNamespaceMaxQPS`, …) only cap how fast history *sends* its **normal** persistence requests. They do not protect against this scenario, because **(1)** the **DLQ writes bypass them entirely** — that path is not rate-limited — and **(2)** rate limits cap request *rate*, not database *latency*; once the database is contended, every operation slows down regardless of the limit. As a result, the problem surfaces as **rising persistence latency and timeouts** (the DLQ-write failures are timeouts, e.g. `received only 0 responses`) — **not** as `ResourceExhausted` rate-limit rejections. So alerting on `ResourceExhausted` will not reliably detect it; detection should watch **archival error rate** and **persistence latency** instead (see [Detection](#3-detection)).

**Key point:** archival errors are *retryable*, so you **cannot "drop" the failing tasks** via any DLQ config — the fix is to **pause archival processing** while the backend is down. This playbook detects the condition early (well before the meltdown) and pauses.

**Another key point:** as it stands today, the Temporal DLQ mechanism is **not designed to handle a very large burst of tasks being dead-lettered at once** — it is a single, serialized lane ([how the DLQ works](#1-how-the-history-task-dlq-works)), and there is no configuration that makes it absorb such a burst gracefully. This is one of the main reasons this playbook's recommendations focus on **detecting an archival failure early and pausing archival — so the burst never forms** — rather than trying to deal with it after the fact (tuning the DLQ, dropping tasks, or adding database capacity, none of which help here).

---

## 3. Detection

Detection is driven from the **Archival Health** row of the [Temporal Server Dashboard](../metrics/dashboards/server/temporal-server-readme.md#21-archival-health). The signals below are ordered by how early they appear; **Signal 1 is the one to alert on** (see [Alert on archival failure](#4-alert-on-archival-failure)).

### Signal 1 — Is archival failing?

Archival normally succeeds, so a sustained error rate means the backend is failing. Three panels answer this:

- **Archival Attempt Error Rate** — the main signal. Any sustained non-zero rate means archival is failing.
- **Archival Attempts by Status** — confirms it. `ok` should drop toward zero; `err` is a real backend failure; `rate_limit_exceeded` is just archival rate-limiting, not an outage.
- **Archival Errors by Type** — tells you if it will recover on its own. `non-retryable` (bad endpoint, DNS NXDOMAIN) means a hard outage that needs action; `transient` (timeouts, dropped connections) may clear by itself.

If the errors are sustained and non-retryable, treat it as a real outage and move to the [recommended action](#5-recommended-action--pause-archival).

### Signal 2 — Is the DLQ backlog building?

Once archival tasks start failing repeatedly, they get sent to the DLQ. Two panels show this:

- **History Task DLQ Writes & Write Failures** — two series:
  - **archival DLQ writes** rising = archival tasks are now being dead-lettered (they hit the `history.TaskDLQUnexpectedErrorAttempts` limit).
  - **DLQ write failures** rising = the DLQ writes themselves are failing, which means the database is under pressure (see [Impact on the cluster](#2-impact-of-a-sustained-archival-failure-on-a-production-cluster)).
- **Archival DLQ Depth** — the total number of archival tasks parked in the DLQ. Expect it to climb during the outage and drain back to zero after [recovery](#6-recover--when-the-backend-is-healthy-again).

### Signal 3 — Is the database under pressure?

This is the late stage — the archival problem has started to affect the database. These panels are elsewhere on the dashboard (not the Archival Health row):

- **Persistence** row — look for rising persistence latency.
- **Service Restarts** panel — look for the `history` service restarting.

Also check **your own infrastructure dashboards** for history pods with high CPU or memory (the server dashboard does not track pod CPU/memory).

In all cases, the [recommended action](#5-recommended-action--pause-archival) helps — apply it now.

---

## 4. Alert on archival failure

Alert on **Signal 1** so you get roughly an hour of lead time before the DLQ backlog builds. Archival normally produces almost no errors, so a sustained error rate is an unambiguous trigger.

You do not need to write this rule yourself — it ships in this repo:

- **[Alert 81 — Archival Backend Failing](../metrics/alerts/server/runbooks/81-archival-backend-failing.md)** — fires on a sustained archival error rate, severity critical, `for: 10m`. This is the alert that should trigger this playbook.
- **[Alert 82 — History Task DLQ Write Failures](../metrics/alerts/server/runbooks/82-history-task-dlq-write-failures.md)** — the escalation: the DLQ writes themselves are failing (the database is already under pressure). If this fires, [pause immediately](#5-recommended-action--pause-archival).

Both are part of the server Essential Alert Set.

**Tuning:** if your cluster sees occasional harmless archival blips (e.g. rare S3 throttling), make Alert 81 less sensitive — raise its `for` window, or switch to a ratio condition (errors as a fraction of total attempts) instead of any-error. See the runbook.

---

## 5. Recommended action — pause archival

**This is the section the whole playbook leads to.** Everything above is about spotting a sustained archival outage early enough to take this one action. As we've seen, the history task DLQ is not built to absorb a high rate of dead-lettered tasks or the DLQ write failures that follow — so trying to tune or force the DLQ is not the answer. The best action during a sustained archival outage is to **pause** archival, and resume once the backend is healthy again (see [Recover](#6-recover--when-the-backend-is-healthy-again)).

Once you've confirmed the backend is unreachable, set these two dynamic config values (no restart needed):

```
history.archivalProcessorSchedulerWorkerCount = 0     # stop processing already-loaded archival tasks (default: 512)
history.archivalProcessorMaxPollRPS         = 1       # stop loading new archival tasks (default: 20)
```

**What this does:** archival tasks stop running, so there are no more calls to the backend, no failures, and nothing gets written to the DLQ. The archival tasks simply wait, idle, until you resume — nothing is lost.

**Confirm it worked** (on the Archival Health row): the Signal 1 error rate drops to ~0, the Signal 2 DLQ writes and write-failures stop, and the DLQ Depth stops growing. History pod CPU / memory and persistence latency should settle back to normal.

**One thing to know: this pauses archival for the whole host.** These two settings are host-wide — they pause archival for **every** namespace on the host, not just one.

- If all your namespaces archive to the **same backend** (one shared bucket — the usual setup), that backend being down means every namespace's archival is failing anyway, so pausing everything is exactly right.
- If only **some** namespaces are affected (they use different backends), pausing also stops the healthy ones — there is no per-namespace pause today. As a workaround you can raise `history.TaskDLQUnexpectedErrorAttempts` to a very high number so the failing tasks keep retrying quietly instead of piling into the DLQ; their retries still add some load to the database, so it's a trade-off.

**Two things not to do instead:**

- **Don't turn archival off on the namespace.** That setting replicates to all your clusters (you'd remove archival everywhere), and it doesn't stop archival tasks that were already created.
- **Turning off the DLQ won't help.** You might be tempted to set `history.TaskDLQEnabled = false` so failing archival tasks stop being written to the DLQ. That doesn't reduce the load — the archival tasks still fail and keep retrying against the down backend, still hitting the database. Pausing archival (the step above) is what actually stops the failing work.

---

## 6. Recover — when the backend is healthy again

Once the backend is reachable again, restore the two values to **whatever they were before you paused** — the defaults if you never changed them, or your own previously configured values if you had customized them:

```
history.archivalProcessorSchedulerWorkerCount = 512   # restore to your previous value (default: 512)
history.archivalProcessorMaxPollRPS         = 20      # restore to your previous value (default: 20)
```

The queued-up archival backlog drains and archives complete. **Retention cleanup also catches up:** while archival was paused, closed workflows could not be deleted yet — Temporal deletes a workflow only after it has been archived — so they piled up in the database. That is expected; they get cleaned up now that archival is running again.

On the **Archival Health** row of the [Temporal Server Dashboard](../metrics/dashboards/server/temporal-server-readme.md#21-archival-health), watch two panels:
- **Signal 1 — Archival Attempt Error Rate** — archival is succeeding again, so the error rate stays ~0.
- **Archival DLQ Depth** — stops growing and drains back toward zero.

To check whether any archival tasks actually reached the DLQ during the outage, look at the **Archival DLQ Depth** panel — a non-zero depth means tasks are parked in the archival DLQ and need to be redriven. If so, redrive them after the backend recovers:
```
tdbg dlq --dlq-version v2 list
tdbg dlq --dlq-version v2 merge --dlq-type 5
```
