# Changelog — Temporal Server Dashboard

## v2.24.0 — 2026-10-05

**Both queue-lag panels had thresholds set above their own histogram's top bucket, so neither could
ever change colour.** Shipped that way since the panels were written.

The numbers came from the server's internal *log* warning levels — 3,000,000 task ids and 30
minutes. Those apply to the raw value before it is bucketed. On a histogram panel the quantile can
never exceed the highest finite bucket boundary, so a threshold above it is unreachable by
construction.

### Fixed

- **Immediate Queue Lag per Pod (2109).** `shardinfo_immediate_queue_lag` is a **Dimensionless**
  histogram; its top bucket is **100,000**. Thresholds were orange 500,000 and red 3,000,000 —
  5× and 30× above the ceiling. Now **orange 10,000, red 50,000**. Healthy is single digits, so
  both still represent a large distance while remaining reachable.
- **Scheduled Queue Lag per Pod (2110).** `shardinfo_scheduled_queue_lag` is a **Seconds** timer;
  its top bucket is **1000s (16.7 min)**. The red threshold was 30 minutes — unreachable. Now
  **15 minutes (900,000 ms)**, which also aligns the panel with alert 38's 900s threshold. Orange
  stays at 10 minutes.

### Changed

- Both descriptions now state the ceiling, say that a line sitting at it means *at least* that far
  behind rather than exactly that, and warn against re-deriving thresholds from the server's log
  values.
- **Scheduled Queue Lag per Pod** additionally documents its **floor**: the reader always reads
  ahead of now, so an idle cluster reports roughly `history.timerProcessorMaxPollInterval`
  (default 5 min; measured 494.6s p99 on an empty cluster). Between that floor and the 1000s
  ceiling the Seconds histogram offers one boundary at 500s — so the panel establishes *that* a
  queue is behind and never *how far*.
- **Immediate Queue Lag per Pod** now says plainly that it measures a task-id **distance**, not a
  count of rows or pending tasks, because **no metric exists for either**. Operators reach for
  these panels looking for a backlog size and there is none to be had.

### Added

- **Immediate Queue Backlog Age by Category (2417).** `shardinfo_immediate_queue_backlog_age` —
  the age of the oldest task at or above the last checkpointed position. The metric has existed all
  along and was on no dashboard. It is the time-based companion to **Immediate Queue Lag per Pod**
  and the panel to reach for when the question is *is the backlog growing or shrinking*, which
  nothing else here answered directly.

  **Immediate categories only** — transfer, visibility, outbound. No scheduled-queue equivalent
  exists.

### Changed

- **Task Load Latency by Task Type (2408)** now states its second use. Read at p95 it is the best
  available indicator of whether a **timer** backlog is growing or shrinking, because a timer
  loaded after its fire time reports exactly how overdue it is. The panel already carried the
  mechanism; it did not say that this is what to use it for, so nobody would find it while looking
  for a backlog. Two limits are stated with it: it is a latency and not a size, and it only records
  when a task is **loaded**, so a queue that has stopped loading goes quiet rather than high.

### Known gaps

The server emits **no count of task rows or pending tasks** for any queue — so there is no panel to
build. Every signal it does emit is an age or a distance, which is enough to trend a backlog
growing or shrinking but never to size one. Sizing means querying the database directly.

---

## v2.23.0 — 2026-10-02

One panel, closing a gap in the rejection views: **nothing answered "which namespace is being
rejected".**

The write-reject loop is diagnosed from persistence rejections, and the useful question is which
namespace they belong to — that is what tells you whose limit to change. But
**History Rejected Database Calls Total by Scope (2405)** is cluster-wide, and **Rejected Database
Calls by Operation and Scope (2401)** is filtered to the single selected `$namespace`. So the only
way to find the namespace was to flip the template variable through them one at a time, which is
not something anyone does calmly at the time.

### Added

- **Rejected Database Calls by Namespace (2416)**, in the Persistence Requests, Latencies and
  Errors row. `persistence_errors_resource_exhausted` by `namespace` and
  `resource_exhausted_scope`.

  The scope breakdown is the half that makes it actionable: `System` is the per-pod limit,
  `Namespace` the per-namespace or per-shard one, so the panel names both the namespace **and**
  which setting refused the call.

  Deliberately the mirror of **Task Scheduler Throttling by Namespace (2407)**, and the pair tells
  the two apart: 2407 is work the scheduler declined to **dispatch**, 2416 is work that was
  dispatched and then **refused** by a persistence limit. Those are different mechanisms with
  different fixes, and conflating them sends you after the wrong setting.

  Cluster-wide by design — it ignores `$namespace`, which is the entire point. Empty is normal.

---

## v2.22.0 — 2026-10-02

A new row for a failure nothing on this dashboard could see: **task tables growing without bound
because row cleanup has stopped.** The table can grow without limit while every error panel reads
clean, because the tasks themselves are completing — it is the cleanup behind them that has
stalled.

A task completing does not delete its row. Rows are removed only by a periodic range delete of
everything older than the oldest still-incomplete task in that queue. So a single task that never
completes holds the position for its whole shard and rows accumulate for **every namespace on that
shard** — while the stuck task is merely slow rather than failing, which means nothing appears on
`task_errors`, nothing reaches the dead-letter queue, and no existing panel moves.

The sequel is worse. When a long-pinned watermark finally advances, the first delete must cover the
entire stuck window. That statement is unbounded — no `LIMIT`, no batching — under a **hard-coded
5-second timeout** that no dynamic config changes. If it cannot finish it is cancelled and deletes
nothing, and because the deletion cursor only advances on success, each retry covers a **wider**
range than the last. Once a category enters that state it cannot recover on its own. On the database
side it presents as a rising wall of cancelled `DELETE` statements against the task table, with
database CPU tracking the cancellation count one-for-one.

### Added

- **History Task Cleanup** row (2410), appended last. Four panels, all on fixed `[5m]` windows
  because deletes fire on a 30-second checkpoint and shorter windows are noisy. Covers every task
  category via `operation=~"RangeComplete.*"` — the defect is identical for timer, transfer,
  visibility, archival, outbound and replication, since all six use the same unbounded range
  delete and none of the task tables has a namespace column.

- **Task Row Cleanup Latency by Category (2411).** The primary signal. Healthy is single-digit
  milliseconds — measured 4.9 ms (timer) and 2.0 ms (archival) on an idle reference cluster.
  Thresholds at 2.5s and 5s.

  **Field-tested by inducing the failure**, holding a `SHARE MODE` lock on `timer_tasks` so deletes
  block while reads continue. Two calibrations came out of it. A blocked delete reports p99 around
  **9.9s**, not 5s — the Seconds histogram steps 5 → 10, so a timed-out delete lands in the 5–10s
  band; the 5s threshold catches it either way. And the `error_type` on the failures panel is
  **`serviceerror_Unavailable`**. Both are now in the panel descriptions, because "pinned at
  exactly 5s" is what an operator would otherwise look for and not find.

- **Task Row Cleanup Failures by Category (2412).** Should be flat zero. Records that a cancelled
  delete rolls back and therefore creates **no dead tuples** — vacuum pressure comes from real
  cleanup, not from these failures, which is a question that comes up immediately with a DBA.

- **Task Row Cleanup Success Rate (2413).** The single "is it fixed" number. The `or vector(0)`
  guard is deliberate: without it the panel goes blank on recovery instead of showing 100%, which
  reads as a broken panel.

- **Task Row Cleanup Attempts by Category (2414).** Looks like context and is not. A delete is
  attempted only when the watermark actually moved, so **zero attempts is the pinned-watermark
  signal** — nothing being cleaned up and nothing complaining. It is the only panel here that
  separates "nothing is trying" from "trying and failing", which are different problems with
  different fixes.

### Notes

- Categories with no task traffic report `NaN` and draw nothing. Correct, not broken.
- Nothing here depends on a server change — `operation` is already present on the persistence
  metrics — so this row works on current and older servers.
- There is still **no metric for task table row count**, so none of these panels shows the backlog
  itself, only whether cleanup is working. Counting rows remains a direct database query.

---

## v2.21.0 — 2026-10-01

One panel in the Cluster Replication row that was wrong in four separate ways at once, three of
them silent. It claimed to be the history task DLQ, it was labelled Cassandra-only, one of its two
series read a metric that does not exist, and the other aggregated a counter as if it were a gauge.

### Fixed

- **Replication DLQ Non-Empty and Enqueue Failures (2005)** — was *DLQ Writes and Failures
  ⚠️ Cassandra Only*. Four corrections:

  **The metric name was wrong.** The panel read `replication_dlq_failed`, which has never existed
  in server source on any branch. The real metric is `replication_dlq_enqueue_failed`, present
  since the original Domain Replication DLQ commit — first tagged **v0.10.0**, so there is no
  version floor worth stating. One missing word meant that series had never returned a point.

  **The Cassandra-only label was false.** The replication task DLQ is implemented for SQL as well:
  `replication_tasks_dlq` exists in the PostgreSQL, MySQL and SQLite schemas with insert, read and
  delete paths in each plugin, and all of its metrics are emitted from generic replication and
  shard code with no store gate.

  **It named the wrong DLQ.** The description cited `history.TaskDLQEnabled`, which governs the
  *history task* DLQ — an unrelated mechanism. What actually sends a replication task here is
  `history.ReplicationTaskProcessorErrorRetryMaxAttempts` (default 80).

  **A counter was being read as a gauge.** `replication_dlq_non_empty` is a counter incremented by
  a periodic check each time it finds a non-empty replication DLQ. The panel did
  `max(replication_dlq_non_empty)`, which returns the cumulative count since process start — so
  once a DLQ had been non-empty even briefly, the panel showed alarm permanently until the pod
  restarted, and nothing it displayed afterwards meant anything. It is now
  `sum(increase(...[11m]))`, which falls back to zero when the DLQ drains.

  The `[11m]` window is deliberate and matches the reasoning behind panels 325, 2109 and 2110:
  the check runs on a 5-minute interval with **full jitter**, so gaps between observations on a
  given shard reach 5 minutes and any shorter window reads zero at random.

### Changed

- **Readme — the DLQ note above the Cluster Replication table** was the source of the confusion
  and now separates the two DLQs explicitly, says which setting governs each, and records that
  neither is Cassandra-only.

### Known, not fixed here

- **`replicator_dlq_enqueue_fails`** (`service/worker/replicator`) is emitted and on no dashboard.
  It is the namespace-replication counterpart to **Namespace Replication DLQ Enqueue Requests** on
  the standby dashboard, which shows enqueue requests with no failure series beside it.
- **Planned alerts 69 and 70** are specified against this panel. Alert 70 was written on the
  non-existent metric name and alert 69 on the un-rated counter; both need respecifying before
  they are built.
- The same four defects exist on the standby dashboard's Replication DLQ row and are fixed in its
  own release.

---

## v2.20.0 — 2026-09-30

One panel that could not return data, and the essential-set alert built on it that therefore
could never fire. Between them, a timer queue could stay stuck indefinitely with nothing to
surface it.

`shardinfo_scheduled_queue_lag` is emitted once per shard every ~5 minutes
(`queueMetricUpdateInterval`, with 15% jitter, so up to ~5.75 min between points). A rate window
has to span at least two consecutive points or `histogram_quantile` returns NaN and the panel
reads "No data". **Timer Task Scheduling Latency (325)** used `$__rate_interval`, which on any
normal dashboard range is a minute or two — far too short. The two per-pod lag panels, 2109 and
2110, were already fixed to a hardcoded `[11m]`; this panel was missed.

It also grouped by `operation`. That label does not exist on this metric: it is recorded with a
single `task_category` tag and nothing else. Grouping by a label that is not present collapses
everything into one unlabelled series, and the legend `{{operation}}` rendered empty.

### Fixed

- **Timer Task Scheduling Latency (325)** — rate window `$__rate_interval` → `[11m]`; grouping
  `by (operation, le)` → `by (task_category, le)`; legend `{{operation}}` → `{{task_category}}`.
  Now matches 2109 and 2110, of which it is the cluster-wide counterpart.
- **Alert 38 — Timer Task Scheduling Lag Critical** — the same two bugs. Its `> 0` guard dropped
  the NaN, so instead of erroring it simply never fired. This is an Essential Set alert and it
  has never been capable of firing on any cluster.
- **Alert 38 threshold** — 30s → **900s**, and `for` 5m → 15m. The query bug was masking a second
  defect: 30s is below the metric's floor. The value is the distance between the queue's read
  position and its ack level, and the timer reader always reads ahead of now, so an idle cluster
  with zero workflows still reports several hundred seconds — one was measured at 495s. A 30s
  threshold would have fired permanently the moment the query started returning data. The upper
  bound is fixed too: the metric is a Seconds timer, whose top bucket is 1000s, so p99 cannot
  exceed that. 900s is the usable page point between the two. `for: 15m` requires more than one
  emission cycle rather than letting a single sample satisfy it.
- **Alert 37 (planned) — Timer Task Scheduling Lag High** — same correction applied to its
  recorded condition in the alerts index, 5s → 700s.
- **Runbook 38** — rewritten. It described the metric as how late individual timers fire, which
  is not what it measures, and cited `history.timerProcessorCompleteTimerInterval`, which does
  not exist in server source. It now covers what the ack level is, why a pinned ack level stops
  row deletion for every namespace on the shard, the floor and the 1000s ceiling, and how to find
  the namespace holding the position with `tdbg shard describe`.

### Changed

- **Timer Task Scheduling Latency** readme entry — rewrote it. It said high values mean "timers
  are firing later than expected", which is the misreading the panel description already warns
  against. It now states what the measurement is, that one stuck task raises it on its own, that
  the real consequence is timer rows never being deleted, and both reading traps.

---

## v2.19.0 — 2026-09-26

Four dead-letter panels that returned nothing on any server below **v1.32.0**, and the two
essential alerts built on two of them.

`dlq_writes` is recorded on the DLQ writer's own metrics handler, which is a service-level
singleton created once at startup. `operation` is a per-task value, so it cannot come from that
handler — it has to be passed explicitly in the record call, and that only happens from server
**v1.32.0**. Every panel filtering `dlq_writes{operation=...}` therefore matched nothing on older
servers, and an empty panel reads as "nothing dead-lettered" rather than "wrong label".

`task_terminal_failures` records the same moment — a task being marked terminally failed — on a
handler that *is* rebuilt per task from the execution's own metric tags, so it carries `operation`
on every version. The filters are unchanged; only the metric name moved.

One caveat, and it is not in the reassuring direction. With `history.TaskDLQEnabled` off, neither
failure path reaches these panels. A task failing with a corrupt or otherwise non-retryable error
is dropped and marked complete — `task_errors_corruption` counts it, these panels do not. A task
failing with unexpected but retryable errors — the 70-attempt `TaskDLQUnexpectedErrorAttempts`
path that the whole stranding story is about — is never dead-lettered at all: it retries with no
limit. So with the DLQ off a flat zero on these panels does not mean nothing is being abandoned,
and alerts 80 and 83 go blind rather than quiet.

### Fixed

- **Visibility Tasks Dead-Lettered by Task Type (2128)** — now reads `task_terminal_failures`.
  Backs alert 83.
- **Dead-Lettered Tasks — Execution-Stranding (page-worthy) (2202)** — now reads
  `task_terminal_failures`. Backs alert 80, the page for stranded executions. This is the one that
  mattered most: with the panel empty, executions can stay frozen indefinitely and the page that
  exists to catch it never fires.
- **Dead-Lettered Tasks — Informational (2203)** — now reads `task_terminal_failures`.
- **Signal 2 — History Task DLQ Writes & Write Failures (2213)** — the `dlq_writes` series now
  reads `task_terminal_failures`. The `task_dlq_failures` series is unchanged and was never
  affected, so **alert 82 was never broken**.
- **Readme** — the claim that "`dlq_writes` carries an `operation` tag" is corrected; it does so
  only from v1.32.0.

### Added

- **Dead-Letter Queue Depth by Category (2206).** `dlq_message_count` by `task_category` — the
  number of messages **sitting** in each dead-letter queue. Every other panel in this group is a
  rate, and a rate reads zero once dead-lettering stops — so a queue that filled long ago and has
  been quiet since reads flat zero on every one of them, however many messages are parked in it.
  A rate cannot show a backlog that already exists. Note the gauge refreshes every
  three hours and only from the host owning shard 1.
- **Task Retry Depth — Stranding Types (approaching DLQ) (2207).** The `task_attempt` companion to
  the visibility panel of the same name, for the execution-stranding operation types. Its real use
  is beside **Dead-Lettered Tasks — Execution-Stranding**: a line that climbs toward the threshold
  and then resets *without* any dead-lettering means tasks are being discarded and reloaded before
  they can reach it — so they retry forever rather than being parked, the queue's position stops
  advancing, and its task table grows without bound.

### Changed

- All four panel descriptions now state which metric they read and why, including what they do
  and do not show when `TaskDLQEnabled` is false.

---

## v2.18.0 — 2026-09-24

Attributing `ResourceExhausted` to the service that raised it. Once a persistence limit is raised,
the dominant cause commonly moves to `RpsLimit` on `AddActivityTask`, `AddWorkflowTask`,
`PollActivityTaskQueue` and `PollWorkflowTaskQueue` — and the dashboard could not say whether
frontend or matching had refused them.

### Changed

- **Resource Exhausted with Cause (121)** now groups by **`service_name`** as well as operation,
  cause and scope, and the legend leads with it. That single label changes what the error means:
  an `RpsLimit` on a poll from **frontend** is a poll being squeezed at priority 4 of 0–5 against
  `frontend.rps` / `frontend.namespaceRPS` (2400 each); the same error from **matching** is polls
  and `AddWorkflowTask` / `AddActivityTask` — history pushing tasks in — sharing one priority-1
  bucket against `matching.rps` (1200 per host, and `matching.namespaceRPS` defaults to `0`, which
  falls back to that same number). The panel description carries the distinction.

### Fixed

- **Actual RPS vs Namespace Host RPS Limit (123)** — the description now says the limit line does
  not render. `namespace_host_rps_limit` is defined in the server but **emitted by nothing**, so
  only the "actual" series has ever had data; compare against `frontend.namespaceRPS` by hand. The
  description also notes that both RPS-vs-limit panels are frontend-only, because `host_rps_limit`
  is emitted by the frontend alone — matching has no limit gauge, so a matching `RpsLimit` has to
  be read off panel 121.

---

## v2.17.0 — 2026-09-23

Four panels the history task processing playbook needs, all in **9. Shard Queue Health** beside the
existing task scheduler panels. Alert **87 — History Write-Reject Loop** ships alongside them,
reading panel 2403 from v2.16.0.

### Added

- **History Task Throughput (2406).** History tasks completed per second, cluster-wide and per
  namespace. It exists because two things in that playbook depend on it and neither could be read
  off the dashboard: it is the **divisor** when working out how many database calls one task costs
  (`calls that reached the store ÷ tasks completed`), and it is how you **confirm a scheduler limit
  took effect** — with `history.taskSchedulerGlobalNamespaceMaxQPS` set to 60, the per-namespace line
  settled at **59.7 to 60.0** cluster-wide on a test cluster. **Total Timer Tasks Processed** uses the
  same metric but filters to `operation=~"TimerActive.*"`, so it could not serve either purpose.
- **Task Scheduler Throttling by Namespace (2407).** The same `task_scheduler_throttled` metric as
  the panel beside it, grouped by namespace instead of by operation — which namespace is being paced,
  rather than which task type. Above zero is the desired state once the scheduler limits are set. It
  also counts in shadow mode (`history.taskSchedulerEnableRateLimiterShadowMode: true`), reporting
  what *would* be held back while nothing is delayed, which is how the limits get sized before they
  take effect.
- **Task Load Latency by Task Type (2408).** How long tasks waited before being loaded. **It reads
  differently for the two kinds of queue**, which is why it needs a panel of its own rather than a
  raw metric: for immediate queues (transfer, visibility, outbound) it is creation-to-load; for
  scheduled queues (timer, archival) it is how *late* the load was relative to the fire time,
  floored at zero by the read-ahead window, and never includes the timer's own duration. The
  metric's description in server source says "from task generation to loading", which holds only
  for immediate queues. This is the signal for a poll rate capped too low.
- **Task Loading Rate by Queue (2409).** The `Get*Tasks` calls, by queue. They spend the same
  persistence budget as task execution. The description carries the two comparisons that make the
  number meaningful: the idle floor (`shards x queues / poll interval` — about **116/s** on a
  2048-shard cluster with no work at all) and the re-read case, where a rate far above that floor
  while tasks are not completing means work is not finishing rather than loading being too fast.

### Changed

- **Panel 2408's description** now lists the queue prefix each series name carries
  (`TransferActive*`, `TimerActive*`, `VisibilityTask*`, `OutboundActive.*`, `ArchivalTask*`), since
  the panel groups by task type and the queue is only readable off the legend.

---

## v2.16.0 — 2026-09-18

Everything in this release came out of field-testing the persistence QPS limits playbook on our own
test cluster. The short version: **the metrics that actually diagnose persistence throttling were
not on this dashboard at all**, and four panels that were on it read clean or misleading under
heavy throttling.

### Added

- **Rejected Database Calls by Operation and Scope (2401).** The gap that motivated this release.
  **Resource Exhausted with Cause** counts requests that *failed*. Most rejections never fail a
  request — the database call is turned away, the task waits and retries, and the request eventually
  succeeds. Measured on a test cluster at one moment: **Resource Exhausted with Cause showed 90/s
  while 785 database calls per second were being turned away.** Nothing on the dashboard showed the
  785. Grouped by `resource_exhausted_scope` as well as operation and cause, because that tag is the
  only thing that separates a per-pod limit (`System`) from a per-namespace or per-shard one
  (`Namespace`) — and the two `Namespace`-scoped limits can otherwise only be told apart from the
  server log message. Expect a mix of both scopes at once; different requests hit different limits.
- **History Rejected Database Calls Total by Scope (2405).** The same metric as 2401 for the
  history service across **all** namespaces, ignoring the `$namespace` variable, grouped by scope
  and cause. 2401 is filtered by namespace, and a history pod's own internal work — queue loading,
  checkpointing, shard updates — is tagged `namespace="system"`, so selecting a namespace drops
  exactly the `System`-scope rejections that indicate the per-pod limit is the one biting.
  Measured during heavy throttling: with `default` selected, 2401 showed System scope at
  **271.9/s** against a true **472.2/s** — **42 per cent hidden**; a second run on the same
  cluster read **313.5/s against 589.4/s**. `Namespace`-scope rejections read identically on both
  panels, since those do belong to the selected namespace. The expression is identical to what
  alert 86 evaluates, so the panel and the alert can never disagree. 2401 stays as it is — it is
  the panel that answers *which namespace* and *which operation*.
- **Database Calls That Reached the Database (2402).** Attempts minus rejections. **Persistence
  Requests Total** counts rejected attempts, because the metrics client wraps *outside* the rate
  limiter, so during throttling it overstates real database load. The query fills missing operations
  with zero deliberately: the naive subtraction silently drops every operation that never had a
  rejection — validated against live data, 17 series instead of 27.
- **Adaptive Rate Limit Multiplier (2404).** The only view of adaptive backoff
  (`history.persistenceDynamicRateLimitingParams`) short of the pod log, which carries it at INFO
  and is therefore invisible on a cluster running at `warn`. **Empty does not mean the setting is
  off:** the gauge is recorded only in the backoff and recovery branches of
  `health_request_rate_limiter.go`, so an enabled limiter on a healthy database emits no series at
  all. Measured: at the default `rateMultiMin: 0.8` the multiplier fell to 0.8 in one step and
  never went lower; with `rateMultiMin: 0.2` the same cluster stepped 0.8, 0.5, 0.2 and held.
- **Write-Reject Loop Indicator (2403).** Separates two causes of a `GetWorkflowExecution` spike
  that look identical on every other panel. When a write is rejected the server empties that
  workflow's cached state, so the retry reads it from the database again — throttling becomes a read
  storm that feeds itself. Ordinary cache pressure produces a read spike too. The ratio
  `workflow_context_cleared / cache_miss` tells them apart: measured **0.5 healthy, 0.9 with a cache
  too small, 1.3 partly throttled, 9.2 in the loop**. Not namespace-filtered, because neither metric
  carries a `namespace` label — `cache_miss` has `namespace_id` (a UUID), `workflow_context_cleared`
  has no namespace dimension at all.

### Changed

- **Resource Exhausted with Cause now groups by `resource_exhausted_scope`.** The tag was already
  being emitted and the dashboard was throwing it away. Without it there is no way to tell which of
  the three persistence limits rejected the request.
- **Matching Service Latency now excludes `GetTaskQueueUserData` and `PollNexusTaskQueue`.** Both
  are long polls. `GetTaskQueueUserData` blocks for up to `matching.getUserDataLongPollTimeout`
  (4m50s by default) and read around **485s** on a test cluster, pinning the y-axis and making
  `AddWorkflowTask` and `AddActivityTask` unreadable. The panel already excluded the *client*-side
  `MatchingClientGetTaskQueueUserData` but not the service-side one — so the exclusion looked
  deliberate and complete, and was neither.

### Documentation

Seven descriptions corrected. All of these were measured during real throttling; each panel read in
a way that would send an operator down the wrong path.

- **Persistence Availability** read **100% throughout heavy throttling**. Its error count excludes
  resource-exhausted, timeouts, shard-ownership-lost, not-found and condition failures. Now says so.
- **Total Timer Tasks Errors** read **flat 0 while 470 timer tasks per second were being throttled**.
  Throttled tasks are counted in `task_errors_throttled`, not here.
- **Timer Task Scheduling Latency** — three separate ways to misread it, now all documented. It
  measures the queue's **ack level**, not fire-time delay, so one stuck task holds it up. It is
  emitted once per shard every ~5 minutes, so the line steps and drops on its own and a drop is not
  recovery. It saturates at the histogram's top bucket, 1000s. And it has a high floor: **measured
  at 495s on a completely idle cluster with zero workflows**, and it did not move under load or
  under heavy throttling. Take a quiet-cluster reading as your zero.
- **Total Timer Tasks Processed** counts attempts including retries, so it inflates under
  throttling — and it has no namespace filter, unlike its neighbours.
- **Timer Task Processing Latency** is per-attempt.
- **Per-Shard Persistence RPS Distribution** is a **30-second average** on a hard-coded interval, so
  it cannot resolve a burst shorter than that — while the limiter it gets compared against is
  instantaneous. It also counts rejected attempts, so it shows demand rather than served rate.
- **Hottest Shard RPS** now says to judge it against one pod's persistence budget rather than
  against zero. Measured on a cluster deliberately built with **no hot shard in it**: the
  distribution shape read `max` 10x `p50` while the busiest shard was doing **5 req/s — 1.7% of a
  pod's budget**. Shape alone is not evidence of a hot shard.

### Known gaps

- There is **no metric for the configured persistence limit**, so an "actual vs limit" panel is not
  possible for persistence the way it is for frontend RPS. `host_rps_limit` is frontend-only.
- `persistence_errors_resource_exhausted` has **no alert**. It is a better signal than the
  service-level one currently alerted on. Planned as a follow-up.

## v2.15.2 — 2026-09-16

### Changed
- **Visibility row description rewritten.** It said the row "tracks your Temporal Visibility store latencies and availability", which stopped being true in v2.15.0 when the row grew from 8 panels to 19. It now names what the row actually covers — write and read paths, per-store health under dual visibility, how close visibility tasks are to being dropped, and Elasticsearch bulk write health — and states the store-attribution rule up front, because that is the thing operators get wrong: only the `visibility_persistence_*` panels carry a store label.

### Documentation
- **Store-blindness now noted on all three ES panels, not just one.** v2.15.1 documented it on **ES Bulk Processor Errors by HTTP Status** only. It is structural and applies to the whole family: `NewProcessor` tags its metrics handler with the operation alone, and the `visibility_index_name` tag is applied around the visibility *manager*, not the store's bulk processor. **ES Bulk Processor Queue Depth** and **ES Write Confirm Latency vs Ack Timeout** carry the same caveat now.
- **Readme Visibility section opens with an attribution note** explaining which metric family can identify a store and which cannot, so the per-panel caveats have context.
- No panel, query, threshold or layout changes. A dashboard on v2.15.1 shows identical data; upgrade only for the corrected descriptions.

## v2.15.1 — 2026-09-15

### Fixed
All six corrections below came from running the v2.15.0 panels against a live dual-Elasticsearch cluster (one ES cluster, two indices). Every one of them was a claim that survived source review and PromQL validation but turned out to be wrong, or incomplete, against real data.

- **`error_type` label values were wrong in every panel description that named one.** The metrics exporter renders the Go error type with dots replaced by underscores, so the queryable values are **`persistence_TimeoutError`** and **`serviceerror_Unavailable`** — not `TimeoutError` and `Unavailable`. Anyone following the old descriptions was querying values that match nothing. Corrected on Visibility Errors by Type per Store, and the readme now states the naming rule rather than just listing examples.
- **ES Bulk Processor Errors by HTTP Status: `0` means Elasticsearch was *gone*, not slow — and the panel is empty when it is hung.** The bulk processor only records a status once Elasticsearch answers. A container that is paused (reachable, not responding) never completes the bulk, so no status is recorded and the panel stays flat. Confirmed both ways: `docker pause` → this panel empty, Write Error Rate empty, only Visibility Errors by Type moved (`persistence_TimeoutError`); `docker stop` → this panel `http_status=0` and Write Error Rate fired. The description now says a flat panel here is not an all-clear.
- **Visibility Tasks Dead-Lettered by Task Type over-reports data loss on Elasticsearch.** The bulk processor holds documents in memory. The store gives up at `worker.ESProcessorAckTimeout` and fails the task — possibly into the DLQ — but the document is still buffered and gets flushed once Elasticsearch recovers. Observed directly: twelve tasks dead-lettered during an outage (`dlq_writes` = 12, `tdbg dlq list` = 12 messages), and after recovery **all twelve records were present in both indices**. The description now says to verify against the store before assuming loss, and notes that SQL has no such buffer so a DLQ'd task there really did lose the write.
- **Visibility Task Failures & Internal Errors only moves while tasks are still retrying.** It counts failing attempts, so once tasks exhaust `history.TaskDLQUnexpectedErrorAttempts` and are set aside the line returns to zero while writes remain completely broken. Observed with the attempt limit at 3: `task_errors_internal` back to zero within a minute while not one of five new workflows reached either store. The description now names the durable signals instead — Write Request Rate flat on both stores, and new workflows never appearing.
- **Visibility Task Retry Depth: corrected in v2.15.0's own entry.** `task_attempt` is recorded by every task on completion (`Ack`), not only past 30 attempts, so a healthy cluster reads about 1 rather than being empty. Restated here because the live run confirmed it: the top bucket read 5 for tasks that had actually made ~3 attempts, which is the histogram bucket rounding in action.
- **Readme: `--dlq-type` takes the numeric category id, not a name.** `tdbg dlq read --dlq-type visibility` fails with `strconv.Atoi: parsing "visibility": invalid syntax`; the history task DLQ wants **4** for visibility. Also noted: without `--last-message-id` the command prompts, and with no terminal attached that prompt panics on EOF.

## v2.15.0 — 2026-09-15

### Added
- **Visibility group — 11 panels for running two visibility stores (dual visibility), and for Elasticsearch write-path health generally.** Backs the rewritten **[Dual Visibility playbook](../../../playbooks/dual-visibility.md)**. Before this release the group had three store-aware panels (2117–2119), all filtered `service_name="history"` — so the dashboard could only show **writes**, and there was no per-store view of the read path anywhere in the repo. Every metric name, tag and default below was verified against server source.
  - **Read path (new — previously no coverage at all).** Reads are issued by frontend, matching and worker, never by history, so nothing in the old group saw them. All three split by `visibility_index_name` and `service_name`, matching `operation=~"ListWorkflowExecutions|CountWorkflowExecutions|GetWorkflowExecution|ListChasmExecutions|CountChasmExecutions"`.
    - **Visibility Read Request Rate per Store** — `sum(rate(visibility_persistence_requests{…}[$__rate_interval])) by (visibility_index_name, service_name)`. Doubles as a read-routing readout: only the primary reports by default; a namespace with `system.enableReadFromSecondaryVisibility` set moves to the secondary; **both** stores reporting means `system.visibilityEnableShadowReadMode` is on.
    - **Visibility Read Error Rate per Store** — read failures per store. Alerts 059a/059b/059c filter `service_name="history"` and therefore **never fire on read failures**; this panel is the only per-store view of a broken workflow list. Orange 0.1 / red 1 req/s.
    - **Visibility Read Latency per Store** — selected percentile per store. Pairs with shadow read mode, whose results are discarded and so are observable only through metrics. Orange 3s / red 5s.
  - **Complete error picture — closes a structural blind spot in panel 2118.** `visibility_persistence_errors` is the `default` arm of the server's visibility error classifier and **skips** `TimeoutError`, `ResourceExhausted`, `NotFound`, `InvalidArgument` and `ConditionFailedError` (verified in `updateErrorMetric`). A flat 2118 is therefore not proof of health.
    - **Visibility Errors by Type per Store** — `sum(rate(visibility_persistence_error_with_type[$__rate_interval])) by (visibility_index_name, error_type, service_name)`. Counts **every** error. `error_type="TimeoutError"` is the only signal for an Elasticsearch write never confirmed inside `worker.ESProcessorAckTimeout` (30s default) — invisible on 2118. Orange 0.1 / red 1 req/s.
    - **Visibility Rate Limit Rejections per Store** — `visibility_persistence_resource_exhausted` by store and cause. `system.visibilityPersistenceMaxWriteQPS` / `MaxReadQPS` (both 9000) are applied **per store**, each store getting its own budget from the same setting. Also absent from 2118.
  - **Data-loss window.** Visibility tasks do **not** retry forever: a store error, an ES ack timeout or an ES rejection counts as an unexpected error, and at `history.TaskDLQUnexpectedErrorAttempts` (70, default-on via `history.TaskDLQEnabled=true`) the task is dead-lettered. With default retry gaps (1s, ×1.1, 180s cap, jittered to 80–100%) that is roughly **70 minutes**.
    - **Visibility Task Retry Depth (approaching DLQ)** — `histogram_quantile(1.00, sum by (operation, le) (rate(task_attempt_bucket{operation=~"VisibilityTask.*"}[$__rate_interval])))`. `task_attempt` is a dimensionless **histogram**, so the top bucket stands in for "deepest attempt count" and the reading is coarse (buckets step 1/2/5/10/20/50/100, so 35 attempts reads as 50 and 70 reads as 100, and the thresholds trigger on the bucket above) — same caveat as the hot-shard `max` panels from v2.14.0. Note it is recorded by **every** task on completion (`Ack`), not only past 30 attempts — the >30 gate is an additional in-flight emission — so the panel is populated on a healthy cluster at about 1 and the signal is the line climbing, not the presence of data. Thresholds mark 30 and 70 (records start being dropped).
    - **Visibility Tasks Dead-Lettered by Task Type** — `dlq_writes{operation=~"VisibilityTask.*"}` by operation. Narrower and more actionable than panels 2202/2203: 2202 matches only timer and transfer operations and will **never** show visibility, while 2203 blends visibility with retention and workflow-task-timeout. Task type decides whether the loss is permanent — a dropped `VisibilityTaskStartExecution` or `…UpsertExecution` is rebuilt by the workflow's next visibility write (every write is a full snapshot versioned on the task id), but a dropped `…CloseExecution` is the last task that workflow ever emits, and a dropped `…DeleteExecution` leaves a record that outlives its workflow.
    - **Visibility Task Failures & Internal Errors** — `task_errors` and `task_errors_internal` on `VisibilityTask.*` (note the emitted name is `task_errors`; the Go constant is `TaskFailures`). `task_errors_internal` rising while panel 2117 sits at zero on **both** stores is the signature of an invalid `system.secondaryVisibilityWritingMode`: only `off`, `on`, `dual` are accepted, anything else is rejected *above* the per-store metrics layer, so **no** `visibility_persistence_*` metric fires and the dashboard merely looks quiet. The config loader accepts any string with no warning and `temporal-server validate-dynamic-config` passes it. Reads keep working throughout.
  - **Elasticsearch write path (new — previously no coverage at all).** ES writes are asynchronous through a per-store bulk processor, built only by the history service. All three panels are labelled **(Elasticsearch only)** in the title and are empty on SQL clusters. They carry **no** `visibility_index_name`, so two ES stores collapse into one series — use Visibility Errors by Type per Store to attribute per store.
    - **ES Bulk Processor Errors by HTTP Status (Elasticsearch only)** — `elasticsearch_bulk_processor_errors` by `http_status`. `0` = cluster unreachable, `429` = overloaded, `400` = mapping, `404` = index missing (never self-recovers; dead-letters everything after ~70 min).
    - **ES Bulk Processor Queue Depth (Elasticsearch only)** — `elasticsearch_bulk_processor_queued_requests`, also a dimensionless histogram, read as the top bucket. Handing a document over **blocks** once the processor is busy, stalling history's visibility workers. Tuning knobs `worker.ESProcessorBulkActions` (500), `BulkSize` (16MB), `FlushInterval` (1s), `NumOfWorkers` (2) are read by **history** despite the `worker.` prefix, and unlike `ESProcessorAckTimeout` they are snapshotted at store construction — **a history restart is required** for them to take effect.
    - **ES Write Confirm Latency vs Ack Timeout (Elasticsearch only)** — `elasticsearch_bulk_processor_request_latency`, red line at 30s = `worker.ESProcessorAckTimeout`. Reaching it fails the visibility task as a timeout that 2118 does not record but which still counts toward the 70-attempt DLQ threshold.

### Fixed
- **Readme: corrected "History retries indefinitely" on Visibility Write Error Rate per Store.** It does not. Visibility tasks are dead-lettered after `history.TaskDLQUnexpectedErrorAttempts` unexpected attempts (70 by default, ~70 minutes), after which the record is no longer written to that store. The readme now states the real behaviour and the per-task-type consequences.
- **Readme: Visibility Write Error Rate per Store no longer described as "the primary alert signal."** It counts only a subset of visibility errors; the readme now says a flat line is not proof of health and points at Visibility Errors by Type per Store.
- **Readme: dropped `ScanWorkflowExecutions` from the Visibility Availability description** — that operation scope is defined in server source but never emitted.
- **Readme: every panel in the Visibility group now carries an explicit store-type column** (`SQL + ES` or `Elasticsearch only`), so it is clear which panels are empty on a SQL cluster.

### Known gaps
- Panels 2130–2132 carry no `visibility_index_name` label, so with **two** Elasticsearch stores they cannot be attributed to one store. This is a server-side metric limitation, not a dashboard one.
- No alert exists for visibility tasks being dead-lettered (`dlq_writes{operation=~"VisibilityTask.*"}`) or for `visibility_persistence_error_with_type`. Alert 080 matches only timer and transfer operations.
- The Elasticsearch panels are derived from server source but have not yet been confirmed against a live Elasticsearch outage.

## v2.14.0 — 2026-09-02

### Fixed
- **Hot-shard detector corrected to use `max`, not p999** (both Persistence-row panels from v2.13.0). Empirical testing on a 2048-shard cluster exposed that the v2.13.0 detector **missed a single hot shard** — the primary "one hot workflow id → one hot shard" case. A single hot shard sits at percentile `(N−1)/N` among `N` active shards, which on a large fleet is *above* the 99.9th percentile (1 of 2048 = the 99.95th), so `p999` landed on a normal shard and the skew read ~1 while one shard was genuinely on fire.
  - **Per-Shard Persistence RPS Distribution** — the hottest-shard line changed from `p999` → **`max` (`histogram_quantile(1.00, …)`)**, relabeled "max (hottest single shard)"; the `p99` line relabeled "p99 (many shards hot)" (it catches the *broad* Kind-1 case). p50 = typical shard.
  - **Hot-Shard Skew → replaced with "Hottest Shard RPS".** The v2.13.0 skew *ratio* (`max ÷ clamp_min(p50, 1)`) was unintuitive: because a typical shard is usually below 1 req/s, the divide-by-zero guard floored the denominator to 1, so the "ratio" collapsed to just `max` and read a meaningless "1.0" when idle. Replaced it with a plain **`max`** stat — the single busiest shard's req/s — titled **Hottest Shard RPS**, unit req/s, thresholds orange 200 / red 500 (illustrative, cluster-tunable). It answers "how hot is the hottest shard?" directly; compare against p50 on the distribution panel.
  - Panel descriptions and the readme now document: `max` is **coarse** (rounds to the histogram bucket boundary, so magnitude is approximate); both panels need **enough active shards** (busy fleet of hundreds-plus) to be meaningful and are noisy on tiny/idle clusters; `p99` is the many-shards-hot signal. The hot-shard alert (index entry 89, then numbered 84) updated in lockstep to use `max`.

## v2.13.0 — 2026-08-31

### Added
- **Persistence Requests, Latencies and Errors** group — two panels for **hot-shard detection**, backing the recurring "how do I find a hot shard?" question. There is no per-shard-tagged metric in Temporal (shard id is a log dimension only, by design), so hot-shard detection is a distribution read, not a single series. Both panels use `persistence_shard_rps` — a histogram of per-shard persistence RPS, computed per shard but recorded with **no `shard_id` or `namespace` label**; default-on (`system.persistenceHealthSignalMetricsEnabled`, default `true`, verified `common/dynamicconfig/constants.go`), emitted every 30s per history host from `common/persistence/health_signal_aggregator.go`, backend-agnostic (Cassandra and SQL alike). Both panels are cluster-wide and ignore `$namespace`.
  - **Per-Shard Persistence RPS Distribution (Hot-Shard Detector)** — p50 / p99 / p999 of `persistence_shard_rps_bucket{service_name="history"}` on one timeseries. `histogram_quantile(0.50|0.99|0.999, sum by (le) (rate(...[$__rate_interval])))`. p99/p999 far above p50 = a hot shard exists; all three together = balanced.
  - **Hot-Shard Skew (hottest ÷ typical shard RPS)** — a stat panel: `histogram_quantile(0.999, ...) / clamp_min(histogram_quantile(0.50, ...), 1)`. One-number skew glance; orange (3) / red (10) thresholds are illustrative and cluster-tunable.
- These panels detect **that** a hot shard exists, not **which** — the shard id comes from the `"Shard queue lag exceeds warn threshold."` WARN log (`history.emitShardLagLog`). Readme section 3 documents the full detect-then-localize flow and cross-links the Shard IO Concurrency playbook (serialization vs hot-shard distinction).

## v2.12.0 — 2026-08-24

### Added
- **History Scavenger** group (new, appended as the last row): detection surface for **unbounded history growth when the history scavenger falls behind** (most often on an XDC standby). The scavenger clears leftover history (`history_tree` / `history_node` rows whose execution is already deleted) but skips any branch younger than `worker.historyScannerDataMinAge` (default 60 days); for short-lived, high-volume workflows that is nearly every branch, so history piles up. Metric names verified against server source — `scavenger_success` / `scavenger_skips` / `scavenger_errors` are recorded only by `service/worker/scanner/history/scavenger.go`, always tagged `operation="HistoryScavenger"`. `scavenger_success` counts branches **handled** (kept or deleted), not deletions. Panels (each `$__rate_interval`, `$DS_PROMETHEUS` only — these metrics carry no `namespace` label):
  - **Scavenger Activity — Skipped vs Handled** — `sum(rate(scavenger_skips{operation="HistoryScavenger"}[$__rate_interval]))` versus `sum(rate(scavenger_success{operation="HistoryScavenger"}[$__rate_interval]))`. When skipped dwarfs handled, the 60-day wait is blocking cleanup.
  - **Scavenger Errors** — `sum(rate(scavenger_errors{operation="HistoryScavenger"}[$__rate_interval]))`. Should sit at ~0; a sustained non-zero rate is a distinct problem from the 60-day wait.
- Backs the new **[XDC Standby Database Growth on SQL playbook](../../../playbooks/xdc-standby-database-growth-sql.md)**.

### Fixed
- Table of Contents now lists **21. Archival Health** (omitted when that group was added in v2.11.0) and **22. History Scavenger**.

## v2.11.0 — 2026-08-12

### Added
- **Archival Health** group (new, appended as the last row): detection surface for a **sustained archival-backend (S3 / GCS / custom) outage**. A dead backend fails every closed workflow's archival task; after `history.TaskDLQUnexpectedErrorAttempts` (default 70 ≈ 1h) each is dead-lettered, and a large burst of DLQ writes can back-pressure the whole database. Two primary detection signals back new essential alerts 81 and 82. Metric name and `status` tag values verified against server source (`service/history/archival/archiver.go` — `status` ∈ {`ok`, `err`, `rate_limit_exceeded`}); `dlq_writes` `operation` tag verified against `service/history/queues/dlq_writer.go` (`OperationTag(taskType)` → `ArchivalTaskArchiveExecution`). Three panels:
  - **Signal 1 — Archival Attempt Error Rate** — `sum(rate(archiver_archive_latency_count{status="err"}[$__rate_interval]))`. `status="err"` excludes rate-limit rejections (`status="rate_limit_exceeded"`), so it reflects genuine backend failures. The **earliest** signal — fires ~1h before failing archival tasks reach the history task DLQ. Backs essential alert 81.
  - **Archival Attempts by Status (ok / err / rate_limit_exceeded)** — `sum(rate(archiver_archive_latency_count[$__rate_interval])) by (status)`. Context for Signal 1: confirms whether archival is succeeding at all and separates a genuine outage (`err`) from archival rate-limiting (`rate_limit_exceeded`).
  - **Signal 2 — History Task DLQ Writes & Write Failures** — two series: `sum(rate(dlq_writes{operation="ArchivalTaskArchiveExecution"}[$__rate_interval]))` (archival tasks reaching the DLQ after 70 failed attempts) and `sum(rate(task_dlq_failures[$__rate_interval]))` (writes to the single-partition DLQ themselves failing — database distress). Backs essential alert 82.
  - **Archival Errors by Type (non-retryable vs transient)** — `history_archiver_archive_non_retryable_error` / `_transient_error` and the `visibility_archiver_archive_*` equivalents (metric names verified against `common/metrics/metric_defs.go`). Non-retryable (bad endpoint / DNS NXDOMAIN) indicates a hard, sustained outage; transient (timeouts) may self-heal — supports the sustained-vs-intermittent judgment central to the playbook.
- Companion playbook: **[Detecting & Recovering from an Archival Backend Outage](../../../playbooks/detecting-recovering-archival-outage.md)** (full mechanism, detection queries, pause remediation, recovery).

## v2.10.0 — 2026-08-04

### Added
- **History Task DLQ / Terminal Failures** group (new, appended as the last row): surfaces history tasks being dead-lettered under database stress — a prolonged DB outage/overload drives timer/retry tasks past `history.TaskDLQUnexpectedErrorAttempts` (default 70 ≈ 1h) of unexpected errors (`context deadline exceeded` / `context canceled`) and into the history task DLQ (`history.TaskDLQEnabled`, default true), stranding activities. DB-agnostic (Cassandra and SQL alike). Four panels:
  - **Dead-Lettered Tasks — Execution-Stranding (page-worthy)** — `sum(rate(dlq_writes{operation=~"Timer(Active|Standby)TaskActivity(RetryTimer|Timeout)|Transfer(Active|Standby)Task(Activity|WorkflowTask)"}[$__rate_interval])) by (operation)`. The execution-stranding subset only; backs new essential alert 80.
  - **Dead-Lettered Tasks — Informational** — same metric filtered to `VisibilityTask.*|Timer(Active|Standby)TaskDeleteHistoryEvent|Timer(Active|Standby)TaskWorkflowTaskTimeout`; these do **not** strand a running execution (WFT timeouts are covered by alerts 56/76), so they are graphed but not paged.
  - **Task Terminal Failures (all DLQ paths)** — `sum(rate(task_terminal_failures[$__rate_interval]))`; covers the 70-attempt threshold, terminal/corruption, and `history.TaskDLQErrorPattern` paths.
  - **Leading Indicator — Unexpected Errors on Stranding Task Types** — `sum(rate(task_errors{operation=~<stranding set>}[$__rate_interval])) by (operation)`; the precursor that accumulates toward the 70-attempt threshold, so a sustained climb here precedes any DLQ write. Metric names and `operation` tag values verified against server source (`service/history/queues/dlq_writer.go`, `common/metrics/metric_defs.go`). **Note:** raw `dlq_writes` totals are dominated by non-stranding visibility/retention writes — always filter by `operation`.

## v2.9.0 — 2026-08-02

### Removed
- **Matching Task Queue Info** group: removed the **Sync Throttle Count** panel (`sync_throttle_count`). That metric is emitted only by the classic `TaskMatcher`; the priority matcher — the default since server **v1.31.0** (`matching.useNewMatcher`, added v1.28.0, defaulted on in v1.31.0) — does not emit it and has no replacement. On any default modern cluster the panel was permanently empty, which reads as false reassurance ("no throttling") on a metric that simply doesn't exist. Sync-match (path 1) saturation has no direct metric on the priority matcher; infer it from Approximate Task Backlog and Async Match Latency. Task Write Throttle Count moved into the freed grid slot. Classic-matcher operators can still alert on `sync_throttle_count` directly (alert `temporal-alert-074`); see the Changing Task Queue Partitions playbook's matcher-selection section.

## v2.8.1 — 2026-08-02

### Fixed
- **Transfer Active Task Errors Workflow Busy** panel: corrected the `resource_exhausted_cause` filter value from the full enum name `RESOURCE_EXHAUSTED_CAUSE_BUSY_WORKFLOW` to the emitted short form **`BusyWorkflow`**. Temporal's `ResourceExhaustedCause` enum defines a custom `String()` (`go.temporal.io/api` `enums/v1/failed_cause.pb.go`) that renders the short CamelCase form, so the tag value in Prometheus is `BusyWorkflow`. As shipped in v2.8.0 the filter matched nothing and the panel still read flat. Verified against source and against a live cluster.

## v2.8.0 — 2026-08-02

### Fixed
- **Busy Workflow Throttling** group: **Transfer Active Task Errors Workflow Busy** panel now queries `task_errors_throttled{operation=~"TransferActive.*",resource_exhausted_cause="RESOURCE_EXHAUSTED_CAUSE_BUSY_WORKFLOW"}` instead of `task_errors_workflow_busy`. The `task_errors_workflow_busy` counter is defined in server source (`common/metrics/metric_defs.go`) but never emitted — it has no `.Record()` call site — so the panel read flat on all versions. The busy-workflow condition is now surfaced by `task_errors_throttled` with cause `RESOURCE_EXHAUSTED_CAUSE_BUSY_WORKFLOW` (`service/history/queues/executable.go`). The `operation=~"TransferActive.*"` scope is preserved: `task_errors_throttled` carries an `operation` tag whose transfer-active values (`TransferActiveTaskActivity`, `TransferActiveTaskWorkflowTask`, …) still match the filter.

## v2.7.0 — 2026-07-27

### Added
- **SDK Workers Info** group: new **Workflow Task Schedule-to-Start Timeouts (sticky fallback)** panel — `sum(rate(schedule_to_start_timeout{operation="TimerActiveTaskWorkflowTaskTimeout",namespace="$namespace"}[$__rate_interval])) by (namespace, operation)`. Placed directly after **Workflow Task StartToClose Timeouts (sticky tq)**. Fires when a workflow task on a sticky task queue is not picked up within the sticky ScheduleToStart timeout (server default 5s, `service/history/tasks/workflow_task_timer.go`) and is rescheduled onto the normal task queue. Normal-queue workflow tasks carry no ScheduleToStart timeout, so this operation isolates the sticky fallback. The separate matching-side fast path (`StickyWorkerUnavailable`, returned after the ~10s `stickyPollerUnavailableWindow`, `service/matching/matching_engine.go`) has no dedicated counter and is not graphable.

## v2.6.0 — 2026-07-02

### Added
- **Shard Movement** group: new **Owned Shards (Total)** panel — `sum(numshards_gauge{service_name="history"})`. Should equal the cluster's configured total shard count at all times; a sustained deficit means a shard has no owner. Backs new essential alerts 78/79.

## v2.5.0 — 2026-06-11

### Added
- **Visibility** group: 3 new per-store panels using `visibility_persistence_*` metrics with `visibility_index_name` label to distinguish primary from secondary store health. Only meaningful when dual visibility is enabled; with a single store both series are identical.
  - **Visibility Write Request Rate per Store** — `sum(rate(visibility_persistence_requests{service_name="history"}[$__rate_interval])) by (visibility_index_name, operation)`. A flat line on one store while the other continues indicates that store has stopped receiving writes — either it is down or `system.secondaryVisibilityWritingMode` dynconfig has changed. Note: `visibility_persistence_*` metrics carry no `namespace` label so no namespace filter is applied.
  - **Visibility Write Error Rate per Store** — `sum(rate(visibility_persistence_errors{service_name="history"}[$__rate_interval])) by (visibility_index_name, operation)`. Primary alert signal for a visibility store outage. History retries failed visibility tasks indefinitely (backoff: 1s initial, 1.1× coefficient, 3-minute cap — no retry limit). Orange > 0.1 req/s, red > 1 req/s.
  - **Visibility Write Latency per Store** — `histogram_quantile($p, sum(rate(visibility_persistence_latency_bucket{service_name="history"}[$__rate_interval])) by (visibility_index_name, operation, le))`. Divergence between primary and secondary latency indicates one store is under pressure or recovering. Orange > 3s, red > 5s.

---

## v2.4.0 — 2026-05-28

### Added
- **Shard Queue Health** group: 2 new panels for task executor scheduler health (panels 7 and 8 of the group):
  - **Task Scheduler Latency per Operation** — `histogram_quantile($p, sum by (operation, le) (rate(task_latency_schedule_bucket{service_name="history"}[$__rate_interval])))`. In-memory schedule-to-start latency: time between a task being loaded into memory and acquiring an executor worker. Rises when the `history.transferProcessorSchedulerWorkerCount` goroutine pool is saturated. Primary signal when a bulk-processing namespace (e.g. mass terminations or deletions) is starving the shared worker pool. Orange > 500ms, red > 2s. `history.transferProcessorSchedulerWorkerCount` is hot-reloadable (no restart) — reduce it via dynamic config to throttle the saturating workload.
  - **Task Scheduler Throttled Rate per Operation** — `sum by (operation) (rate(task_scheduler_throttled{service_name="history"}[$__rate_interval]))`. Rate of tasks explicitly rejected by the scheduler. Complements the latency panel: latency rising = tasks queueing for a worker; throttled rising = tasks being turned away hard. Any sustained non-zero value warrants investigation.

### Fixed
- Dashboard title corrected from `v2.3.0` to `v2.4.0` (title was not bumped in v2.3.1, which was a metadata-only fix).

---

## v2.3.1 — 2026-05-27

### Fixed
- **Immediate Queue Lag per Pod** and **Scheduled Queue Lag per Pod**: changed rate window from `[$__rate_interval]` to `[11m]` (hardcoded). `shardinfo_immediate_queue_lag` and `shardinfo_scheduled_queue_lag` are emitted by `monitorQueueMetrics()` on a fixed 5-minute timer (`queueMetricUpdateInterval = 5 * time.Minute`, `context_impl.go:73`). Grafana's `$__rate_interval` resolves to ~1 minute (4 × default 15s scrape interval), which never spans 2 consecutive emissions — `histogram_quantile` returns NaN and both panels show No Data. A fixed `[11m]` window (>2× the emission interval) always captures at least 2 data points.

---

## v2.3.0 — 2026-05-27

### Added
- New panel group **Shard Queue Health** (group 9, inserted between Shard Movement and History Timer Task Info) with 6 panels for stuck shard detection:
  - **Immediate Queue Lag per Pod** — `histogram_quantile($p, sum by (instance, task_category, le) (rate(shardinfo_immediate_queue_lag_bucket{service_name="history"}[11m])))`. Orange > 500K tasks, red > 3M tasks. Primary signal for a stuck shard — one `instance + task_category` line rising monotonically while others recover.
  - **Scheduled Queue Lag per Pod** — same structure over `shardinfo_scheduled_queue_lag_bucket`. Orange > 10 min, red > 30 min.
  - **DB Pool Refresh Failure Rate per Pod** — `sum by (instance) (rate(persistence_session_refresh_failures{service_name="history"}[$__rate_interval]))`. Earliest signal for DB-caused stuck shards; fires before queue lag builds. SQL backends only.
  - **DB Pool Refresh Failure Ratio per Pod** — failures / attempts ratio. Orange > 10%, red > 50%. SQL backends only.
  - **Suspected Deadlocks (current) per Pod** — `sum by (instance) (dd_current_suspected_deadlocks{service_name="history"})`. Event-driven gauge; absence of data is healthy. Any value > 0 requires pod restart.
  - **Deadlock Event Rate per Pod** — `sum by (instance) (rate(dd_suspected_deadlocks{service_name="history"}[$__rate_interval]))`. Complements the gauge — shows cumulative detection events after the gauge has cleared.

### Changed
- Groups 9–18 renumbered to 10–19 to accommodate the new group

---

## v2.2.0 — 2026-05-15

### Fixed
- Excluded `_unknown_` namespace from all panels that group or filter by namespace. The `_unknown_` value is emitted by Temporal for internal/system-level requests that have no namespace context and should not appear as a selectable namespace or as a series in namespace-breakdown panels.
  - Namespace template variable query updated: `label_values(service_requests{namespace!="_unknown_"}, namespace)` — `_unknown_` no longer appears in the namespace dropdown
  - Panels patched: **Actions per Namespace** (18), **RPS per Namespace** (20), **Service Requests by Namespace and Operation** (93), **Service Errors by Namespace and Operation** (100), **Actual RPS vs Namespace Host RPS Limit** (123), **Outlier Namespaces** (2004)

---

## v2.1.0 — 2026-05-13

### Added
- New panel group **Worker Registry (In-memory)** (group 16, inserted between Visibility and Cluster Replication) with 5 panels:
  - **Workers Added** — rate of new worker registrations
  - **Workers Removed** — rate of removals across all causes (shutdown, TTL eviction, capacity eviction)
  - **Percentile of Num of Cached Entries** — estimated entry count derived from `capacity_utilization × 1e6` at the selected `$p` percentile across matching instances
  - **Percentile of Cache Utilization** — utilization as a percentage at the selected `$p` percentile, with threshold lines at 80% (orange) and 100% (red)
  - **Workers - Number of Activity Slots Used** — `histogram_quantile` of `worker_registry_activity_slots_used` at the selected `$p` percentile

### Changed
- Cluster Replication renumbered from group 16 → 17
- Authorization renumbered from group 17 → 18

---

## v2.0.0 — 2026-05-12

First versioned release. Prior changes were unversioned.

### Fixed
- Corrected metric name in Shard Movement > Shards Closed panel: `sharditem_closed_count` → `shard_closed_count`
