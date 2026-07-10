# billPay_IT Stuck Activities — Incident Analysis

> We analyzed the server logs, metric screenshots, dynamic config, and workflow event histories
> shared with us so far. The sections below are the concrete things we were able to establish from
> that data, cross-checked against the Temporal 1.29.6 server source. Times are shown in UTC with
> MST alongside (batch start: **3:00 PM MST = 22:00 UTC**).

## Contents

- [Problem statement](#problem-statement)
- [Business impact](#business-impact)
- [Remediation](#remediation)
- [What we looked at and ruled out](#what-we-looked-at-and-ruled-out)
- [1. Environment & configuration](#1-environment--configuration)
- [2. What we observed (cluster and database side)](#2-what-we-observed-cluster-and-database-side)
- [3. Our analysis — part 1 (history side)](#3-our-analysis--part-1-history-side-the-activity-dispatch-step-was-starved)
- [4. Investigating why a small number stayed stuck](#4-investigating-why-a-small-number-stayed-stuck)
- [5. Recommendations](#5-recommendations)

---

## Problem statement

On **Jun 30 2026, ~3:00 PM MST (22:00 UTC)**, a large batch of pre-existing workflows (~170K)
advanced at the same time. A subset of them ended up with an activity stuck in
**"Pending / Scheduled, Attempt 1"** — recorded by the server as *scheduled*, but never handed to a
worker. The affected activities did not recover on their own.

## Business impact

- **~70 workflow executions stuck** (exact count to confirm — cited as ~70 in one account and ~700 in
  another), out of the ~170K in the fan-out.
- Each stuck execution could not progress past the scheduled activity until it was cleared manually;
  the longest sat for **~8 hours**.
- **No data loss** — the workflows and their histories stayed intact throughout. The activities were
  delayed, not dropped.

## Remediation

- The operator **paused and then unpaused each stuck activity** manually.
- Unpausing caused Temporal to **re-attempt the dispatch** (a fresh hand-off of the activity to
  matching). By then the database was no longer saturated, so the hand-off went through and the
  workflows progressed normally.

## What we looked at and ruled out

Before landing on the explanation below, we checked several other possible causes against the
customer's own logs and metrics and ruled them out:

| Considered | Ruled out because |
|---|---|
| Workers were down or not polling | Workers were **up and polling** the task queue during the window. |
| A matching-service bug | For the workflows we traced, the activity **never reached matching** — the history→matching hand-off failed first, so the stall did not originate in matching. *(One subset did reach matching and is tracked as an open item, in the "why a subset did not recover" section below.)* |
| Task-queue partition misconfiguration | Read and write partition counts match (**12 / 12**); the read==write invariant holds and no partition was orphaned. |
| Fairness / new matcher behavior | Both are **off** (classic matcher, the 1.29.6 default), consistent with the logs — neither feature was in play. |

---

## 1. Environment & configuration

### Platform / versions
| | |
|---|---|
| **Server version** | **Temporal `1.29.6`** — from the container image `docker.io/temporalio/server:1.29.6` on every pod. (Note: the k8s label `app.kubernetes.io/version: 1.27.2` is a stale Helm-chart label; the running image is 1.29.6.) |
| **Web UI** | `2.48.1` |
| **Deployment** | EKS; Helm chart `temporal-0.62.0`; ArgoCD-managed; k8s namespace `temporal` |
| **Database** | **Aurora PostgreSQL**, reached via an **RDS Proxy** (`aexp-bank-e3-rds-proxy.proxy-cbw8qugmxztm.us-east-1.rds.amazonaws.com`, port 5432, db `temporaldatabase`, user `temporldb_app`, SCRAM/SASL auth) |
| **Matcher mode** (from matching logs) | **Classic matcher** (`backlog: classic`) — this is the 1.29.6 default (`matching.useNewMatcher` defaults `false` in 1.29.6). **Fairness OFF**, **priority matcher OFF**. |

### Dynamic configuration (from ConfigMap `temporalmonitoring-dynamic-config`)
Values as shared, with the 1.29.6 default and a note. **Bold** = differs from default / notable.

| Setting | Value (customer) | Default (1.29.6) | Note |
|---|---|---|---|
| **`history.persistenceGlobalMaxQPS`** | **8000** | **0 (off)** | Cluster-wide cap on history→DB ops/sec. Customer-enabled. **Central to this incident.** |
| **`history.persistenceMaxQPS`** | **per-ns: DE 2500, IT 2500, AT 1000, MX 2000, US 2000** | 9000 (per host) | Per-namespace caps. **Sum = 10,000 > the 8,000 global cap** → namespaces contend at the global level. |
| `matching.numTaskqueueReadPartitions` | 12 (DE/IT/AT/MX/US) | 4 | read == write (invariant holds) |
| `matching.numTaskqueueWritePartitions` | 12 (DE/IT/AT/MX/US) | 4 | |
| **`matching.backlogNegligibleAge`** | **300s** | 5s | Tuning; not implicated in this incident |
| **`matching.maxTaskBatchSize`** | **200** | 100 | task-writer batch size |
| **`matching.outstandingTaskAppendsThreshold`** | **500** | 250 | write-buffer depth (write path only) |
| **`frontend.keepAliveMaxConnectionAge`** | **48h** | 5m | |
| `frontend.namespaceBurst` | 4000 (US) | — | |
| **`frontend.namespaceCount`** | **50000** | 1200 | |
| `system.enableEagerWorkflowStart` | true | true | same as default |
| `matching.enableFairness` | *(not set)* | false | → fairness off (consistent with logs) |
| `matching.useNewMatcher` | *(not set)* | false | → classic matcher (consistent with logs) |

### Scope observed during the incident

The database errors we observed (detailed in [Section 2](#2-what-we-observed-cluster-and-database-side)) were **cluster-wide, not specific to billPay** —
they came from **1,000+ shards across at least four namespaces**. billPay (`IT`) was just one of
them, and **not even the biggest** contributor to the database load: another namespace
(`8576ffb0-cd2b-44e0-8a07-fbdf11468bb0`) hit the shared per-second database limit even harder than
billPay did.

**Namespaces seen in the persistence errors.** (Counts are from sampled windows, so treat them as
*relative* size, not exact totals.)

| Namespace ID | Name | Role in this incident |
|---|---|---|
| `8576ffb0-cd2b-44e0-8a07-fbdf11468bb0` | *not yet resolved* | **Largest** contributor at the 22:00 peak — more errors than IT |
| `047ed8be-83d5-465d-903f-b70d8c010cc4` | **IT** (billPay) | The namespace in this ticket |
| `1973f596-ec95-45f6-a3a1-eb02c131f68c` | *not yet resolved* | Also affected |
| `96e2c8a3-4e5b-4530-8584-5400e34ede53` | *not yet resolved* | Also affected |

**What was affected inside billPay (`IT`):**

| Item | Value |
|---|---|
| Task queue | `taskQueue_billPay_IT` (Activity type), 12 partitions |
| Workers | Deployment `billpay-core-deployment` |
| Activity that got stuck | `VERIFY: Payment Instrument` |

**That activity's timeouts** (these matter later — see the activity-timeouts section):

| Timeout | Value | Set explicitly? | What it means |
|---|---|---|---|
| Schedule-To-Start | 36h | **No** — inherited from the workflow run timeout | Max time the activity can wait for a worker to pick it up |
| Schedule-To-Close | 36h | **No** — inherited from the workflow run timeout | Max total time for the activity, including retries |
| Start-To-Close | 4s | Yes | Max time for a single execution attempt, once a worker has started it |

---

## 2. What we observed (cluster and database side)

This section is **observations only** — what the logs and metrics show actually happened, on both the
**cluster/server side** (Temporal's own per-second rate limits) and the **database side** (connection
and query failures). Our *analysis* of why this made activities stick starts in [Section 3](#3-our-analysis--part-1-history-side-the-activity-dispatch-step-was-starved).

For roughly **1 hour and 40 minutes** starting at the batch time, the Temporal **history** service was
failing large numbers of its database operations — **cluster-wide**, across **1,000+ shards and
multiple namespaces** (billPay was one of several affected; another namespace was actually the largest
contributor at the peak). So this was not specific to billPay.

### 2a. The three kinds of errors we saw

Three distinct kinds. The first is a **cluster-wide rate limit Temporal enforces on itself** (not a
database fault); the other two are the **database itself failing**:

**1 — Rate-limit rejections.** `System Persistence Max QPS Reached` / `Namespace Persistence Max QPS
Reached`. Temporal caps how many database operations per second it will issue, to protect the DB
(setting: `history.persistenceGlobalMaxQPS = 8000` cluster-wide; per-namespace cap for `IT` = 2,500).
When demand exceeded the cap, Temporal **rejected its own operations before they ever reached the
database.** *(This is Temporal throttling itself — not a database failure.)*

**2 — Connection failures.** The history pods **could not reach the database.** Two forms:
DNS timeouts resolving the RDS-Proxy hostname (the in-cluster DNS, CoreDNS at `10.100.0.10`, timing
out — e.g. `dial udp 10.100.0.10:53: i/o timeout`), and connection/authentication timeouts to the
RDS Proxy itself.

**3 — Database-operation timeouts** (`context deadline exceeded`) — the database was reachable but
too slow to finish the operation. The specific operations we saw time out in the logs:
- `GetWorkflowExecution` — loading a workflow's state (by far the most common)
- `GetTimerTasks` / `GetTransferTasks` — reading the internal lists of pending tasks

### 2b. How many, and when

Count of failed history→database operations per 5-minute bucket:

| Time (UTC) | Time (MST) | Errors |
|---|---|---|
| **22:00 – 22:05** | **3:00 – 3:05 PM** | **149,028** |
| 22:05 – 22:10 | 3:05 PM | 198 |
| 22:10 – 22:30 | 3:10 – 3:30 PM | ~30–200 each |
| 22:30 – 22:35 | 3:30 PM | 197 |
| 22:45 – 23:10 | 3:45 – 4:10 PM | ~150–400 each |
| **23:15 – 23:20** | **4:15 PM** | **1,471** (secondary bump) |
| 23:20 – 23:40 | 4:20 – 4:40 PM | ~60–450 each |
| after 23:40 | after 4:40 PM | ~0 (subsided) |

- **~153,600 errors total**, and **~97% of them in the first 5 minutes.**
- The very first errors landed at `22:00:00.09` — **~10,000 in the first 6 seconds.** A near-instant
  wall, not a gradual ramp.

**Breakdown by error kind.** At the peak (a 10,000-error sample from the first ~6 seconds), all three
kinds were happening at roughly the same time:

| Error kind | Share at the peak |
|---|---|
| **1 — Rate-limit rejections** | ~37% |
| **2 — Connection failures** (DNS + proxy) | ~36% |
| **3 — Database-operation timeouts** | ~27% |

**At the peak, two separate things were happening at the same time:**
- **#1** — Temporal hit its own per-second cap and rejected operations. *(The database was not at fault here — this is Temporal's own limit.)*
- **#2 + #3** — the database itself was failing: either unreachable (#2) or too slow to respond (#3).

**By ~3:30 PM MST it was ~98% #1.** The database had recovered (no more #2 or #3), but demand was
**still above the cap**, so Temporal kept rejecting operations.

---

## 3. Our analysis — part 1 (history side): the activity-dispatch step was starved

> **If you already know Temporal's persistence priority model, skip ahead.** Subsections 3a–3c explain
> in detail how the database-operation priority rate limiter works. If that's familiar to you (e.g. the
> server team), jump straight to [**3d — what we saw in the logs**](#3d-what-we-saw-in-the-logs--the-evidence-everything-above-points-to):
> that's the actual evidence from this incident and why the priority model is the relevant lens here.
> Everything in 3a–3c is background that leads to 3d.

This section is our **analysis** of the observations in [Section 2](#2-what-we-observed-cluster-and-database-side). Here we focus on the **history service's**
role: the step where history hands a scheduled activity off to the matching service for a worker to
pick up. (The matching service's role is covered in Part 2.)

Temporal applies a **priority-based rate limiter to its database operations**: when the database
can't keep up, higher-priority operations are served first and lower-priority ones are held back. Its
purpose is to protect the database and keep the most important work (serving API calls, advancing
workflows) responsive under pressure.

This limiter is **central to the history-side analysis**, because [Section 2](#2-what-we-observed-cluster-and-database-side) shows the exact conditions where
it kicks in — the database was overloaded and the per-second limits were actively being hit. And
**activity dispatch is in the low-priority group**, so it was the work that got held back.

### 3a. The priority order of database operations

Each database operation is assigned a priority based on **what triggered it**, and when the database
is saturated, higher-priority operations are served first. These are the actual priority levels in
1.29.6 (0 = served first; source references in the appendix):

| Priority | Operations at this level | What they are |
|---|---|---|
| 0 | Operator / admin calls | manual administrative operations |
| 1 | `StartWorkflowExecution`, `SignalWorkflowExecution`, `SignalWithStartWorkflowExecution`, `RequestCancelWorkflowExecution`, `TerminateWorkflowExecution`, `UpdateWorkflowExecution`, `GetWorkflowExecutionHistory`; `RangeCompleteHistoryTasks` | starting / steering workflows; checkpointing queue progress |
| **2** | **`RespondWorkflowTaskCompleted`** (this is the call that writes `ActivityTaskScheduled`); all other API calls | a worker reporting workflow-task results — including scheduling activities |
| 3 | `GetHistoryTasks` | the history queue reader loading batches of pending tasks |
| **4** | **`TransferActivityTask`**, `TransferWorkflowTask` | **dispatching a scheduled activity/workflow-task to matching** ← *the step that got starved* |
| 5 | activity / workflow-task / run / execution **timeout** tasks; worker commands | firing timeouts |
| 6 | `DeleteHistoryEvent`, delete-execution, archival; standby & unknown tasks | background cleanup |

**The two rows that matter here:** writing an activity as *scheduled* runs at **priority 2**
(`RespondWorkflowTaskCompleted`), but **dispatching that activity to a worker runs at priority 4**
(`TransferActivityTask`). Scheduling outranks dispatch — so once the persistence rate limiter was
saturated (database operations exceeding the configured `persistenceGlobalMaxQPS` of 8,000/sec, and
being throttled with `Max QPS Reached`), scheduling kept succeeding while dispatch was starved.

### 3b. Why is dispatch lower-priority than scheduling?

Scheduling is part of directly advancing your workflow and answering an API call, so it's protected.
Dispatch is a background hand-off Temporal expects to finish a millisecond later — and under normal
load it does. It only becomes a problem when the database stays saturated for a sustained period.

### 3c. Low-priority work has no reserved share

Picture the database budget as a shared pool of tickets, refilling at the configured rate
(~8,000/second). High-priority work takes tickets **first**; low-priority work gets only what's left.
**There is no minimum reserved for low-priority work.** So when high-priority work alone drains the
whole pool, activity dispatch gets **zero** tickets — and is rejected (`Max QPS Reached`) over and
over, for as long as the high-priority flood continues.

*(This is by design: under sustained throttling the priority order protects the database and keeps
workflows and API calls responsive, at the cost of delaying background dispatch.)*

### 3d. What we saw in the logs — the evidence everything above points to

**This is the key finding of this section.** Everything above (3a–3c) is the model; this is what the
incident logs actually show — low-priority dispatch rejected and high-priority work served, at the
same time.

**Dispatch was being rejected.** For stuck workflow `01KVYNFQTR8F8WNX3146B9J1TM-execute` at 22:30:04:

- Its dispatch task `TransferActivityTask` ("move this activity to matching") failed to process:
  `context deadline exceeded`.
- The reads and writes that task depends on (`GetWorkflowExecution`, `UpdateWorkflowExecution`) were
  rejected: `System Persistence Max QPS Reached`.

**High-priority work was not.** In the same history logs, over the same window, there are essentially
no failures on high-priority operations — starting workflows, advancing them, writing "activity
scheduled." Those kept going through.

In short: under the high database load, the low-priority dispatch operations were rejected while the
high-priority operations continued to go through.

---

## 4. Investigating why a small number stayed stuck

Section 3 explained why activity dispatch was *delayed* under the load. But delay was not the whole
story: as noted in the business impact, **~70 executions never progressed at all.** They stayed in
**Running** status with an activity shown as **Pending / Scheduled, Attempt 1** — the
`ActivityTaskScheduled` event was written to history, but the activity was never delivered to a
worker. This section is about that residue: why the overwhelming majority recovered on their own, and
why a small number did not.

*(Timestamps below are in UTC. For reference: 22:00 UTC = 3:00 PM MST = 5:00 PM CDT.)*

### 4a. The workers were present and busy the whole time

A worker problem is the first thing to rule out — and it wasn't one. Three independent signals from
the customer's own metrics show the workers on `taskQueue_billPay_IT` were healthy and actively
working throughout:

- **Poll success** (server metric `poll_success`). From the batch launch at 22:00 UTC, activity
  poll-success jumped to **~1,000–1,500/sec** — a flood of activities being handed to workers — and
  the Activity and Workflow poll lines moved together, so **both worker types were polling.**
- **Poll timeouts** (server metric `poll_timeouts`). During that burst these dropped to ~0 (every
  poll got a task), then returned to their idle baseline of ~40–50/sec of empty long-polls — normal
  for workers that are up and long-polling whether or not there is work to hand them.
- **Worker task slots** (SDK metric `temporal_worker_task_slots_available`). The ActivityWorker's
  free execution slots repeatedly dipped from ~1.1K toward 0 and recovered — the sawtooth of a worker
  grabbing bursts of (short, 4-second) activities and completing them. It stayed busy well past the
  initial spike.

So this was **not** workers being down, unregistered, or not polling. There was ample idle poll
capacity on this task queue.

### 4b. The overwhelming majority recovered because dispatch kept flowing

Those same signals show that, for the bulk of the workload, dispatch **kept working.** The slot metric
shows the ActivityWorker executing burst after burst for the hours following the batch, and the
starvation from Section 3 was *intermittent* — it bit only in the moments demand exceeded the cap. So
the vast majority of the ~170K activities dispatched and completed normally; only ~70 were left
behind — a tiny fraction of the batch.

The event history of one stuck workflow shows this precisely. At **22:30:03 UTC** it scheduled
**three activities in parallel** within the same workflow execution:

| Activity | Outcome |
|---|---|
| `RetrieveAccountHolderAddress` | Started and **completed within ~1 second** |
| `Fetch account information from GAR` | Started and **completed within ~1 second** |
| **`VERIFY: Payment Instrument`** | **Never started — stuck as Pending / Scheduled, Attempt 1** |

Two of the three activities were dispatched and finished within about a second; the third was not —
even though all three were scheduled at the same moment on the same task queue. (We do not know which
task-queue partitions the two completed activities were dispatched from.) So the stall affected
individual activities, not the task queue as a whole.

A worker-side latency metric corroborates the broad delay. The customer's p99 activity
schedule-to-start latency (`temporal_activity_schedule_to_start_latency_seconds`, Java/Micrometer) ran
with a tail of **≥30 seconds across the whole incident window** — a large share of activities waited
tens of seconds to be picked up, then were. Two caveats keep this as *corroboration only*, not proof
about the stuck ones: p99 is the slowest ~1%, but the ~70 stranded activities are ~0.04% of the batch —
far past p99 — and the histogram saturates at its top bucket (~30s), so the multi-hour stragglers are
absorbed into it rather than shown. This metric confirms dispatch was broadly delayed; it cannot
isolate the stranded subset. For those, the event history above is the reliable evidence.

### 4c. Why did the stranded ones not self-recover?

The one thing the data lets us state with confidence: **the stranded activity was never delivered to
a worker, even though workers were available.** The reasoning, step by step:

| What we observe | What it tells us |
|---|---|
| Between bursts the ActivityWorker had free slots and was long-polling, returning empty (~40–50/sec timeouts, 4a) | Idle poll capacity was available on the task queue |
| The stuck `VERIFY: Payment Instrument` activity sat for **~8 hours** | Had it been in matching's deliverable queue, an idle poller would have picked it up in milliseconds |
| → So the activity was **not available to matching's pollers** | It was stranded **upstream** — the Section 3 dispatch step (`TransferActivityTask`) never completed, seen from the worker's side |
| A failed `TransferActivityTask` is normally **retried, not dropped**, and load eased by ~23:40 UTC | The retry should have completed and dispatched the activity on its own |
| ~70 executions did **not** recover — they sat until an operator intervened, ~7 hours after load cleared | **Why they stopped retrying is the central open question** (see 4d) |
| `ScheduleToStart` was effectively inherited as **36 hours** (see [Recommendations](#5-recommendations)) | No near-term timeout fired, so the stuck activity never failed and the workflow got no signal to act on — it simply sat for the workflow's 36-hour lifetime |

### 4d. What is confirmed and what is still open

| Point | Status |
|---|---|
| Workers were up, polling, and dispatching the bulk of the workload | **Confirmed** (poll + slot metrics) |
| The stuck activities were scheduled at the saturation peak (~22:30 UTC) and never delivered to a worker | **Confirmed** (event history + poll metrics) |
| The stall was *upstream* of matching's pollers — not "sitting in the queue while workers were busy" | **Confirmed** — pollers were idle and available, yet the task sat ~8h |
| Whether the task never reached matching, or reached a backlog matching could not surface | **Open** — both look the same from the poller side (idle pollers, no task); needs more data (the schedule-to-start latencies and the 23:15 UTC logs) |
| Why the stranded transfer tasks stopped retrying for ~8h | **Open** — the key remaining question; not explained by load alone, since load eased by ~23:40 UTC |

### 4e. One observation that may be useful for further investigation

Both of the two stuck workflows we were able to trace had their activity task later dispatched from
**task-queue partition 4** specifically (of 12 partitions). This is a very small sample and is most
likely coincidence — partition 4 is an ordinary, valid, polled partition, and nothing in the
configuration singles it out. We note it only because it is a real pattern in the little data we have,
and it may be worth investigating whether a single partition can lag this way while the others drain.

---

## 5. Recommendations

These recommendations address the parts of the incident we were able to **confirm** (Sections 1–3).
The remaining question — why the ~70 stranded activities did not retry on their own — is **still under
investigation** (Section 4). Recommendation 2 is important regardless of how that open question
resolves: it lets the **workflow itself** detect and recover a stranded activity instead of relying on
manual pause/unpause — though, as the note below it explains, it requires a small change in workflow
code, not just a config value.

| # | Recommendation | Why it helps |
|---|---|---|
| **1** | **Smooth the batch — the primary fix for billPay.** Spread the ~170K workflow progressions over time instead of releasing them in the same few seconds. | Removes billPay's contribution to the load spike that saturated the persistence rate limiter and starved activity dispatch. Necessary — but note the saturation was cluster-wide (Recommendation 3), so this is not sufficient on its own if other namespaces can spike at the same time. |
| **2** | **Set a short `ScheduleToStart` timeout — and handle it in workflow code** (see the note below this table). | Today `ScheduleToStart` is effectively inherited as **36h** from the workflow run timeout (not set explicitly), so a stranded activity never times out and the workflow gets no signal — which is why manual pause/unpause was needed. A short `ScheduleToStart` surfaces the stall to the workflow quickly. **It does not auto-recover on its own** — see the note. |
| **3** | **Assess cluster and database capacity for concurrent peak load — this incident was cluster-wide, not just billPay.** Multiple namespaces hit the shared persistence cap at the same moment (`8576ffb0…` harder than `IT` — see [Section 1](#1-environment--configuration)). Revisit `history.persistenceGlobalMaxQPS` = 8,000 only *alongside* database / RDS-Proxy / CoreDNS headroom. | The 8,000 cap is **shared across all namespaces** and is **below the sum of the per-namespace caps (10,000)** — so namespaces already contend at the global level. If several can spike together, the cluster and database need headroom for the *aggregate* peak; raising the cap alone, without matching DB and connection-layer capacity, just moves the bottleneck. |

> ⚠️ **This is a situational suggestion, not a general best practice.** We do **not** normally
> recommend setting a `ScheduleToStart` timeout on activities — most workloads are better off without
> it. We raise it here only as a defensive mitigation for *this specific situation*, where a small
> number of activity tasks got stuck on scheduled and **we do not yet fully understand why** (Section 4
> is still open). If the underlying cause is resolved, this mitigation may not be needed. Treat it as a
> safety net to consider while that investigation continues, not a default to apply broadly.

**How it behaves — important if you apply it.** A `ScheduleToStart` timeout is **not** retried by the
server: when it fires, the activity fails and the workflow receives an `ActivityFailure` whose **cause**
is a `TimeoutFailure` of type `ScheduleToStart`. Setting the timeout alone therefore does **not** recover
the activity — the **workflow code must catch the `ActivityFailure`, confirm the cause is a
`ScheduleToStart` `TimeoutFailure`, and re-invoke the activity.** Applied *without*
that handling it only makes things worse (the activity fails instead of continuing to wait). Value: we
cannot prescribe it — observed schedule-to-start latencies were already 30s+, so keep it in the
**minutes** range (~2 min as a rough starting point), tuned not to fire under normal load bursts.
