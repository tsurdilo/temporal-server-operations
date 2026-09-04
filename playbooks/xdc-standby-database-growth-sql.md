# XDC Standby Database Growth on SQL — Playbook

**Audience**

Operators of self-hosted Temporal clusters running **multi-cluster replication (XDC)** on SQL persistence (PostgreSQL / MySQL, including Aurora).

This playbook is for **SQL persistence** — the detect and remediate steps are SQL queries. A similar problem can happen on Cassandra, but the steps here don't apply to it. A Cassandra-specific playbook will be added later.

**Applies to**

This playbook applies to all Temporal server versions in use. The problem and how you investigate it are the same across versions, but some fixes differ before and after server release **1.31.0** — so check your server version to know which path applies to you. The differences are spelled out in [Server version differences](#server-version-differences).

**When to use**

Use this playbook when a standby cluster's database has grown much larger than the active for a **global namespace**. Replication itself is healthy — workflows replicate fine — but under some conditions the standby's **cleanup** falls behind, so its database keeps growing. The extra size is data that was removed on the active but not on the standby: completed workflows and their leftover history.

Because active and standby are **per-namespace roles**, one cluster can be active for some namespaces and standby for others. This growth builds up only for the namespaces the cluster is standby for.

**Scope**

This playbook covers **one specific cause**: the cleanup of **completed and deleted executions** falling behind on the standby. A standby's database can be a different size for other reasons — for example the two clusters hosting a different set of namespaces — and those are out of scope here.

It is **not** a retention mismatch. For a global namespace, retention is replicated configuration: as long as it is only ever changed on the active cluster, it is identical on the active and the standby. The size gap is cleanup falling behind, not a shorter retention on the standby.

<a id="which-database-tables"></a>
**Which database tables this playbook covers**

This playbook is about a few tables in Temporal's **main database** getting large on the standby. Both groups below can grow — the history tables usually account for most of the excess:

- `history_node` and `history_tree` — the workflow's **event history** (all the events it recorded); usually most of the excess size
- `executions` and `current_executions` — the **execution's own state**, separate from the events: `executions` holds each run's state (one row per run), and `current_executions` points to the current run (one row per workflow id); usually much smaller than the history tables

If those are the tables that are much larger on your standby, you're in the right place. If the large tables are something else — the **visibility** store (`executions_visibility`, or Elasticsearch), or the task or replication tables — that's a different problem and this playbook won't help.

**What this playbook answers**

1. **[Detect](#1-detect--is-the-standbys-database-growing-larger-than-the-actives)** — how do I confirm the standby's database has grown larger than the active for the situation this playbook covers?
2. **[Remediate](#2-remediate--it-already-happened)** — it already happened; how do I safely clear the excess data off the standby?
3. **[Prevent](#3-prevent--stop-it-recurring)** — how do I stop it from happening in the first place, and from returning after I've cleaned it up?

The cleanup this playbook is about is controlled by a couple of dynamic config settings whose defaults are deliberately cautious. [Prevent](#3-prevent--stop-it-recurring) explains what they do and how to set them for your situation.

**Dashboards and alert for this playbook**

Two dashboards carry the signals used here. On each, expand the History Scavenger row — it holds the same two panels, **Scavenger Activity — Skipped vs Handled** and **Scavenger Errors**:

- **[Temporal Server Dashboard](../observability/dashboards/server/temporal-server-readme.md#22-history-scavenger)** — expand the **History Scavenger** row.
- **[Temporal Standby Cluster — Replication Health](../observability/dashboards/server/temporal-standby-readme.md#8-history-scavenger)** — expand the **🧹 History Scavenger** row. Use this one when the cluster is the standby for a global namespace.

There's also an optional **[History Scavenger Errors](../observability/alerts/server/alerts-index.md#alert-85--history-scavenger-errors)** alert (`temporal-alert-085`). It's documented but **not** shipped in the essential alert set — provision it yourself if you run global namespaces on SQL.

---

## Contents

- [How cleanup works, and why the standby can grow larger](#how-cleanup-works-and-why-the-standby-can-grow-larger)
  - [The `executions` gap](#the-executions-gap)
  - [The history gap](#the-history-gap)
- [1. Detect — is the standby's database growing larger than the active's?](#1-detect--is-the-standbys-database-growing-larger-than-the-actives)
  - [1.1 Watch the scavenger's own metrics](#11-watch-the-scavengers-own-metrics-the-history-gap)
  - [1.2 Confirm deletions aren't being replicated](#12-confirm-deletions-arent-being-replicated-the-executions-gap)
  - [1.3 Compare the largest tables](#13-compare-the-largest-tables)
  - [1.4 Count leftover history branches](#14-count-leftover-history-branches-the-history-gap)
  - [1.5 Compare branches per workflow](#15-compare-branches-per-workflow-the-history-gap)
- [2. Remediate — it already happened](#2-remediate--it-already-happened)
  - [2.1 Clear the left-behind workflows on the standby](#21-clear-the-left-behind-workflows-on-the-standby-the-executions-gap)
  - [2.2 Let the scavenger remove the leftover history](#22-let-the-scavenger-remove-the-leftover-history-the-history-gap)
- [3. Prevent — stop it recurring](#3-prevent--stop-it-recurring)
  - [3.1 Copy workflow deletions to the standby](#31-copy-workflow-deletions-to-the-standby-the-executions-gap)
  - [3.2 Keep the scavenger's wait short enough](#32-keep-the-scavengers-wait-short-enough-the-history-gap)
  - [3.3 What a healthy steady state looks like](#33-what-a-healthy-steady-state-looks-like)
- [Server version differences](#server-version-differences)
- [Config & metric reference](#config--metric-reference)

---

## How cleanup works, and why the standby can grow larger

A Temporal cluster's database fills up as workflows run: every workflow execution leaves behind a record and its event history. Temporal deletes this over time so the database doesn't grow forever. When an execution **closes** (completes, fails, is canceled, times out, is terminated, or continues as new), the **namespace's retention period** sets how long it's kept — once that's up, Temporal deletes the execution. Removing the execution's record and removing its event history are two separate deletes, so if the record is removed but the history delete keeps failing (for example, the database erroring on it repeatedly), the history is left behind. Temporal runs a background job that finds and removes this left-behind history afterward. That ongoing work — deleting closed executions, and removing left-behind history — is what "cleanup" means here.

**Each cluster runs this cleanup for every namespace it hosts — including the global namespaces where it is the standby.** There are two cleanup jobs:

- **Retention cleanup.** As a workflow runs on the active, its history events are sent to the standby through replication, and the standby writes those same events into its own history tables. From them, the standby keeps its own copy of the workflow — both the execution's record and its event history. The workflow's completion is just another such event. When the standby applies that completion event, it sets its **own** deletion timer for that workflow (same close time, same retention period), and deletes the workflow itself when the timer fires. Nothing is handed over as a deletion instruction; both clusters reach the deletion independently, at about the same time. (Retention is the "keep closed workflows for N days" setting on the namespace.)

- **The history scavenger** — a background job that removes **leftover history**: `history_tree` / `history_node` rows whose execution has already been deleted but whose history was left behind. It's an internal Temporal workflow that runs on every cluster about every 12 hours; changing that cadence isn't possible.

To understand what the scavenger cleans up — and why it can be slow — it helps to see how history is stored and deleted.

**How an execution is stored.** An execution is stored as two parts:

- its **record** — one row in the `executions` table, holding the execution's current state plus a pointer to its history;
- its **history** — a **branch**, the ordered line of events the execution produced. (Normally an execution has exactly one branch.)

A branch isn't a single database row; it's spread across two tables:

- **`history_node`** — the events themselves. As the workflow runs, its events are appended in batches, one row per batch, so a single branch has **many** `history_node` rows (a longer history just means more of them).
- **`history_tree`** — **one row per branch**: a small record that the branch exists.

So a normal execution's history is **one branch = one `history_tree` row + many `history_node` rows**, however long the history.

**How an execution is deleted.** Deleting an execution (by retention, or on demand) removes its parts in a fixed order, with the branch **last**: first the visibility entry (in the separate visibility store), then the `current_executions` pointer, then the `executions` record, and finally the branch — its `history_tree` row and `history_node` rows. The branch is deleted last on purpose: an execution should never be left pointing at history that's already gone.

**What a leftover is.** A **leftover** is what remains when that last step doesn't finish: the `executions` record is deleted, but the branch's `history_tree` and `history_node` rows are still there.

**Why leftovers happen.** The deletion is carried out by an internal **delete task** — a background work item. If its history-delete step keeps hitting a database error, the task is retried, but not forever: after about 70 attempts (~1 hour, `history.TaskDLQUnexpectedErrorAttempts`) Temporal stops retrying and parks the task in a dead-letter queue (on by default, `history.TaskDLQEnabled`) for an operator to inspect. It's the *task* that gets parked — not the data. The `history_tree` / `history_node` rows just stay in their tables, undeleted, and those undeleted rows are the leftover.

**What the scavenger does.** The scavenger goes branch by branch and checks whether each branch's workflow still exists:

- **workflow already gone** → it deletes the leftover branch (its `history_tree` row and `history_node` rows). This is the leftover cleanup this playbook is mostly about.
- **workflow still exists** → it normally leaves the branch alone. But with `worker.historyScannerVerifyRetention` **on (the default)**, if that workflow is completed and older than its namespace's **retention plus a buffer** (`worker.executionDataDurationBuffer`, default **90 days**), the scavenger deletes the whole workflow. This is a late backstop for retention — it only catches workflows that outlived retention by a wide margin, e.g. after a normal retention delete kept failing.

**The one setting that gates all of this:** the scavenger skips any branch until it is older than `worker.historyScannerDataMinAge` — **60 days by default** — measured from when the branch was created (roughly when the workflow started). So with the default, a leftover can sit in the database undeleted for up to about **2 months** before the scavenger will even consider it.

So the standby's database isn't growing because cleanup is switched off — cleanup runs on every cluster. A cluster's database can still grow for two reasons, and this playbook covers both:

- **A cluster's own deletes are failing.** Under real database stress, a cluster's history-deletes can keep failing, leaving history behind (**leftovers**) for the scavenger to clean up later. This can happen on the **active or the standby**.
- **The standby never received a delete.** Some operators delete completed workflows on the active early — through Temporal's delete API — instead of waiting out the retention period. By default in 1.31, that delete isn't copied to the standby, so the standby keeps the whole workflow (its record and its history) until its own retention removes it later. This affects only the standby, and it has nothing to do with the scavenger or with database errors.

In this playbook, a **gap** means the extra data the standby is holding that the active has already removed — the size difference between the two clusters. There are two, one for each table group above:

- **[The `executions` gap](#the-executions-gap)** — extra rows in `executions` / `current_executions`: workflows the active has deleted but the standby still has.
- **[The history gap](#the-history-gap)** — extra rows in `history_tree` / `history_node`: the event history of workflows the standby is still holding, plus any leftover history.

These usually appear **together**, not one or the other. When the standby is still holding a workflow the active already deleted, both its `executions` rows and its history are extra — so both gaps are present at once, from the same cause. The one difference is that the history gap has an extra source the executions gap doesn't: **leftovers** — the record deleted but the history left behind (from a failed delete, or after you clear the `executions` rows during cleanup) — which only the scavenger removes. The next two sections explain each.

### The `executions` gap

*The gap in the `executions` and `current_executions` tables.*

Alongside the automatic retention cleanup, Temporal provides an API — `DeleteWorkflowExecution` — to delete a workflow execution on demand, whether it is still running or already closed.

Why an operator would use it: the `executions` table keeps one row per execution — running and closed alike — and retention only deletes **closed** executions, and only once the retention period has passed. A namespace that needs a long retention, or that runs many long-lived workflows, can build up a large `executions` table. To keep the database size down, some operators run a job (a "janitor") that uses this API to delete closed executions early, rather than waiting out the full retention period. On the active, that keeps the `executions` table smaller than retention alone would.

`current_executions` grows on the standby the same way, but stays smaller. A single workflow (one `workflow_id`) can run as a chain of **runs** — each continue-as-new, retry, or reset starts a new run with its own `run_id`. `executions` keeps a row for **every run**, while `current_executions` keeps **one row per `workflow_id`**, holding just a pointer to that workflow's current run. So it has far fewer rows, and it's deleted along with the workflow.

**The catch: in Temporal 1.31, these API deletes are not replicated to the standby by default.** The setting that turns this on, `history.enableDeleteWorkflowExecutionReplication`, is **off by default.** (On 1.30 and earlier the setting doesn't exist, so these deletes can't be replicated at all; in a future Temporal version it becomes automatic — full breakdown in [Server version differences](#server-version-differences).) So the active deletes a workflow soon after it closes, but with the flag off the standby doesn't receive a delete event for it. The standby then keeps that workflow until its **own** retention removes it, at the workflow's close time plus the retention period. How far behind the standby falls depends on how early the janitor deleted the workflow on the active: delete it right after it closes and the standby holds it for nearly the whole retention period; delete it close to when retention would fire anyway and the gap is small. That lag is the `executions` gap — it clears on its own eventually, but with aggressive early deletion it can grow large and look permanent.

### The history gap

*The gap in `history_tree` / `history_node`.*

The extra history comes in two kinds.

**Most of it belongs to workflows the standby still has.** These are the same workflows behind the executions gap: the standby hasn't deleted them yet, so it still stores their event history. This history is **not** a leftover — the workflows still exist. It is removed when the workflows themselves are removed: when the standby's own retention deletes them, or once their deletes are replicated from the active. (The scavenger normally leaves this history alone — it only steps in as the late retention backstop described above, for workflows that outlive retention by a wide margin.)

**The rest is leftover history** — the workflow's record (its `executions` row) has been deleted, but its history (the `history_tree` and `history_node` rows) is still there.

Normally the record and its history are removed together. Deleting a workflow isn't a single delete — it's a sequence of separate steps (remove the visibility entry, then the current-run pointer, then the record, and **last** the history), and Temporal finds the history to remove by following a pointer that is stored inside the record. A leftover appears when the earlier steps run — so the record is gone — but that final history step does not. And once the record is gone, the pointer to its history is gone too, so nothing can find that history the normal way anymore; only the scavenger, which scans the history tables directly, can still find and remove it.

There are three ways the record gets removed but the history doesn't:

- **The delete couldn't read the record, so it couldn't find the history to remove.** `tdbg workflow delete` reads the workflow's record first to find where its history is, then removes both. If it can't read the record — the read fails, or the record is already gone — it has no way to find the history, so it removes what it can and skips the history. The record disappears; the history stays. The history service logs this at **`WARN`** — `Unable to load mutable state. Skipping workflow history deletion.` (tagged with the namespace, workflow, and run id) — and `tdbg` also prints it in the command's own output, so you see it as you run the delete. There is **no metric** for this skip, so if you want to alert on it, it has to be log-based (e.g. Loki / your log pipeline).
- **A cluster's own automatic delete kept failing and Temporal gave up on it.** Retention deletes (and the normal delete API) don't run inline — they run as an internal **delete task** that Temporal retries whenever the database returns an error. It doesn't retry forever: after about **70 attempts** (roughly an hour, `history.TaskDLQUnexpectedErrorAttempts`) Temporal stops retrying that task and moves it to the **history task dead-letter queue (the DLQ)** for an operator to review (the DLQ is on by default — `history.TaskDLQEnabled`). The delete task removes the record before it removes the history, so a task that got the record deleted and then kept failing on the history step — until it landed in the DLQ — leaves the record gone but the history behind.
- **Someone removed the record rows directly in the database.** Deleting `executions` / `current_executions` rows by hand (raw SQL) skips Temporal's delete entirely, so Temporal never removes the matching history — and with the record gone, nothing points to that history anymore. Don't do this; it is exactly how history gets stranded (see [1.4](#14-count-leftover-history-branches-the-history-gap)).

Leftover history is what the scavenger removes. But by default the scavenger waits until a branch is 60 days old before it will touch it (`worker.historyScannerDataMinAge`), so leftovers can sit for weeks. Clearing them promptly means lowering that wait — see [Remediate](#2-remediate--it-already-happened) and [Prevent](#3-prevent--stop-it-recurring). Leftover history can build up on the active or the standby, which is why the **History Scavenger** panels are on both dashboards.

---

## 1. Detect — is the standby's database growing larger than the active's?

Run the checks below on both clusters and compare. Together they tell you which of the two gaps you have — and both can be present at once:

- **The executions gap** — the standby is holding workflows the active already deleted. You confirm it from the **table sizes** ([1.3](#13-compare-the-largest-tables)) being much larger on the standby, plus **delete-replication being off** ([1.2](#12-confirm-deletions-arent-being-replicated-the-executions-gap)).
- **The history gap** — history whose workflow is already gone (leftovers). You confirm it from the **branch checks** ([1.4](#14-count-leftover-history-branches-the-history-gap), [1.5](#15-compare-branches-per-workflow-the-history-gap)) and the **scavenger metrics** ([1.1](#11-watch-the-scavengers-own-metrics-the-history-gap)). Leftovers mostly appear *after* you start clearing executions, or when a cluster's own deletes are failing — so in a fresh case these checks can look normal even while the tables are large.

> **A note on namespaces:** these counts cover the whole cluster — all the namespaces it hosts — not one namespace at a time. The `executions` table has a `namespace_id` column, so you can group by it to see which namespaces contribute most. `history_tree` has no `namespace_id` column, so history growth can't be broken down by namespace in SQL. To narrow it down, look at the namespaces this cluster is standby for whose workflows are short-lived and high-volume — those generate the most history to replicate here.

### 1.1 Watch the scavenger's own metrics (the history gap)

Start here — this is the quick, non-invasive check, and the one you can alert on. The scavenger's **Prometheus metrics** show whether it's running, whether it's erroring, and — after you act — whether it's making progress. They're pre-built as the **History Scavenger** row on the [Temporal Server Dashboard](../observability/dashboards/server/temporal-server-readme.md#22-history-scavenger) and the [Temporal Standby Cluster dashboard](../observability/dashboards/server/temporal-standby-readme.md#8-history-scavenger); the raw counters, all tagged `operation="HistoryScavenger"`:

| Metric | What it actually counts |
|---|---|
| `scavenger_skips` | branches skipped — **only** because they're younger than the 60-day wait (`historyScannerDataMinAge`) |
| `scavenger_success` | branches **handled without error** — this means **kept OR deleted**, not "deleted" |
| `scavenger_errors` | branches that errored (couldn't be read, or the lookup / delete failed) |

> **Metric-name note:** these names are what Temporal's default (tally) Prometheus reporter emits, and the dashboard panels ship with them. If your cluster uses the OpenTelemetry / OpenMetrics reporter, the same counters appear with a `_total` suffix — `scavenger_skips_total`, `scavenger_success_total`, `scavenger_errors_total` — so you'll need to adjust the panel queries (and any direct queries) to match what your Prometheus actually has.

> **Don't read `scavenger_success` as "history deleted."** It counts every branch the scavenger handled without error — both the ones it **kept** (their workflow still exists) and the ones it **deleted**. So it can climb steadily even while nothing is being removed, and no metric separates out the actual deletions. To confirm the database is really shrinking, watch the database checks trend down: [leftover-branch count](#14-count-leftover-history-branches-the-history-gap), [branches per workflow](#15-compare-branches-per-workflow-the-history-gap), or [table size](#13-compare-the-largest-tables). (If you collect worker logs, each real deletion also writes one `deleted history garbage` line, which you can count as a cross-check.)

**What this panel can and can't tell you.** When *skipped* is much higher than *handled*, the scavenger is skipping most of the branches it looks at because they're younger than the 60-day wait. That alone is **not proof of a problem** — a busy cluster with many recent workflows looks the same. Use it as a reason to check the database (the steps below), not as confirmation on its own. What the panel **does** tell you reliably is whether the scavenger is running and whether it is erroring.

On the **Scavenger Errors** panel, the count should stay near zero. If it's climbing, the scavenger is hitting errors while working through branches — for example it can't read a branch, can't check whether a branch's workflow still exists, or a delete fails. This stalls cleanup too, but the cause is these errors, not the 60-day wait — so look into what's failing (usually a database problem).

**If `scavenger_skips`, `scavenger_success`, and `scavenger_errors` are all missing or flat at zero,** the scavenger probably isn't running. It runs as a workflow on the **Worker service** (the `worker` role process), on a 12-hour schedule. It's on by default but can be turned off (`worker.historyScannerEnabled`), and it never fires if the Worker service isn't running or isn't polling the `temporal-system` namespace. Two checks confirm it:

**Is the scanner workflow there and running on schedule?**

```bash
temporal workflow describe \
    --address <cluster> \
    --namespace temporal-system \
    --workflow-id temporal-sys-history-scanner
```

Expect the workflow to be **found**, with a run that completed within roughly the last 12 hours (it runs on a 12-hour schedule). Signs of trouble: **not found** (the scanner never started — the Worker service is down, or the scavenger is disabled); the most recent run **failed**; or the last run finished **much longer than 12 hours ago** (it has stopped firing).

**Is a worker actually polling the scanner's task queue?**

```bash
temporal task-queue describe \
    --address <cluster> \
    --namespace temporal-system \
    --task-queue temporal-sys-history-scanner-taskqueue-0
```

Expect at least one **poller** listed — that's the Worker service. No pollers means nothing is there to run the scavenger: the Worker service isn't up, or it isn't polling this namespace.

### 1.2 Confirm deletions aren't being replicated (the executions gap)

Check the setting on the active:

- Look at dynamic config for `history.enableDeleteWorkflowExecutionReplication`. In 1.31 it defaults to `false`.

If it's off **and** you run a cleanup job that deletes closed workflows early on the active, the executions gap is likely present. Confirm it directly by checking whether workflows the active has already deleted are still on the standby:

1. On the **standby**, list some workflows for an affected namespace. Send the `xdc-redirection: false` header so the request is answered by the standby itself rather than forwarded to the active:

   ```bash
   temporal workflow list \
       --address <standby-cluster> --namespace <ns> \
       --grpc-meta xdc-redirection=false
   ```

2. Pick a few closed workflows from that list and look each one up on the **active**:

   ```bash
   temporal workflow describe \
       --address <active-cluster> --namespace <ns> --workflow-id <workflow-id>
   ```

If a workflow is present on the standby but the active returns **workflow not found**, the active deleted it and the delete never reached the standby — that's the executions gap.

### 1.3 Compare the largest tables

List the largest tables on each side (PostgreSQL example):

```sql
SELECT relname AS table_name, pg_size_pretty(pg_total_relation_size(relid)) AS size
FROM pg_catalog.pg_statio_user_tables
ORDER BY pg_total_relation_size(relid) DESC
LIMIT 10;
```

Run this in the main database. (Temporal's visibility store is a separate database, so it won't appear here.)

What you're looking for, and what it points at:

| Table | If it's much larger on the standby | Which gap |
|---|---|---|
| `history_node`, `history_tree` | history of workflows the standby is still holding (plus any leftover history) | [the history gap](#the-history-gap) |
| `executions`, `current_executions` | workflows deleted on the active but still present on the standby | [the executions gap](#the-executions-gap) |

If instead the table that's growing is in the **visibility** store (`executions_visibility` / Elasticsearch), this is a different problem — see [Which database tables this playbook covers](#which-database-tables).

### 1.4 Count leftover history branches (the history gap)

This estimates leftovers by matching each history tree to a workflow in `executions`. It relies on a tree's `tree_id` being the `run_id` of the run that created it — which holds for new workflows and for restarts (continue-as-new). Run on both clusters:

```sql
SELECT
    COUNT(*)                                                   AS total_branches,
    COUNT(*) FILTER (WHERE e.run_id IS NULL)                   AS orphan_branches,
    ROUND(100.0 * COUNT(*) FILTER (WHERE e.run_id IS NULL)
                  / NULLIF(COUNT(*), 0), 2)                    AS pct_orphan
FROM history_tree ht
LEFT JOIN executions e ON e.run_id = ht.tree_id;
```

Read it carefully:
- The reliable signal is `total_branches` — a plain count of branches that doesn't depend on the `tree_id`-to-`run_id` match (unlike the other two columns). **Much larger on the standby** than on the active is the confirmation.
- A **low `pct_orphan` does not mean the standby is fine.** Most of the standby's extra branches belong to workflows it hasn't deleted yet — those branches still match an execution, so they don't count as orphans. The orphan percentage can stay low while `total_branches` is very large. (A branch becomes an orphan — a leftover — only after its execution is deleted; see [1.5](#15-compare-branches-per-workflow-the-history-gap).)
- **`orphan_branches` and `pct_orphan` are only approximate.** A reset workflow reuses the same `tree_id` across its runs. If the original run was deleted but a later run still exists, that branch has no matching `run_id` in `executions`, so the query counts it as an orphan even though the workflow is still live. If you reset workflows often, treat these two numbers as rough.
- **Use this query only to compare the two clusters and confirm the problem — never to hand-pick `history_tree` rows from it to delete.** Because the orphan count can be wrong (previous point), deleting history that way could remove a live workflow's history. Let the scavenger remove leftover history instead (see [Let the scavenger remove the leftover history](#22-let-the-scavenger-remove-the-leftover-history-the-history-gap)). (Deleting whole stranded *workflows* on the standby is a separate, safe step done through the admin API — see [2.1](#21-clear-the-left-behind-workflows-on-the-standby-the-executions-gap).)

### 1.5 Compare branches per workflow (the history gap)

```sql
SELECT
    COUNT(*)                                        AS workflow_rows,
    ROUND(
      (SELECT COUNT(*) FROM history_tree)::numeric
      / NULLIF(COUNT(*), 0), 2)                     AS branches_per_workflow
FROM executions;
```

This divides the number of `history_tree` rows by the number of `executions` rows. On a healthy active it's close to **1.0**. When it climbs to several, there are far more history branches than live executions — that's leftover history: branches whose workflow is already gone. The scavenger brings it back down once its wait is lowered (see [2.2](#22-let-the-scavenger-remove-the-leftover-history-the-history-gap)).

---

## 2. Remediate — it already happened

There are two cleanups here, one for each gap. They are independent — do whichever gaps you have:

- **[2.1](#21-clear-the-left-behind-workflows-on-the-standby-the-executions-gap) clears the executions gap** — admin-delete the stranded workflows on the standby. Each delete removes that workflow's record, its history, and its visibility entry together.
- **[2.2](#22-let-the-scavenger-remove-the-leftover-history-the-history-gap) clears the history gap** — let the scavenger remove any leftover history (history whose workflow is already gone). Leftovers come from earlier deletes that failed under database stress, or from branches the admin delete in 2.1 had to skip.

> Any bulk delete on the standby must be **throttled** — large manual deletes can overwhelm the database. See the cautions in [2.1](#21-clear-the-left-behind-workflows-on-the-standby-the-executions-gap) before you start.

### 2.1 Clear the left-behind workflows on the standby (the executions gap)

You need to delete the left-behind workflows **directly on the standby**, not on the active. The `tdbg` admin delete below does exactly that: it acts on whichever cluster you point `--address` at and is never redirected to the active, so it needs no special header — that's why the command has none. (The normal `temporal workflow delete` CLI is different: it *is* redirected to the active by default, so with it you'd have to send `xdc-redirection: false` — see the note at the end of this section.)

**Small batches — `tdbg`:**

```bash
tdbg --address <standby-cluster> \
    workflow delete \
    --namespace <ns> --workflow-id <wid> --run-id <rid>
```

`tdbg workflow delete` uses Temporal's **admin** delete, which removes the workflow's record, its history, **and** its visibility entry on whatever cluster you point it at. The admin delete isn't redirected and doesn't check which cluster is active, so it acts on the standby. Use it for **small batches only** — each call deletes one workflow.

**Large backlogs — script the admin delete:**

For millions of workflows, write a small tool that runs the same **admin** delete against the standby in **throttled batches** — for example: list the targets with `temporal workflow list` (with `xdc-redirection: false`, limited to workflows older than a cutoff), then delete a few hundred at a time with a short pause between batches. Throttling is not optional — the next two cautions explain why.

> **Throttle bulk deletes — database wraparound risk (PostgreSQL).** Deleting hundreds of millions of rows by hand creates a huge amount of cleanup work for PostgreSQL's autovacuum. If the deletes outrun autovacuum, PostgreSQL can approach **transaction-ID (XID) wraparound**, which forces a disruptive emergency vacuum. This is a PostgreSQL behavior, not a Temporal one — see the PostgreSQL docs on routine vacuuming and wraparound prevention. In practice: delete in small batches (a few hundred at a time), pause between batches, and give autovacuum room to keep up. If you hit wraparound, stop the cleanup until the database recovers.

> **Expect a replication backlog while the standby catches up.** During and after a large cleanup, the standby has a lot of replication to catch up on; the backlog may grow (watch the **Replication Lag** and **Task Pipeline Health** rows on the [Temporal Standby Cluster dashboard](../observability/dashboards/server/temporal-standby-readme.md), or gauge it directly with `SELECT count(*) FROM replication_tasks` on the active — these are tasks the standby hasn't confirmed yet). The standby's history logs may fill with repeated `VerifyVersionedTransition` errors ("mutable state not up to date"). **This is normal catch-up, not a fault:** when the standby is behind, it fetches the missing data from the active and retries. It clears as the standby catches up — watch that a very large backlog is trending **down**.

> **Why not Temporal's built-in batch delete?** OSS does have one (`temporal workflow delete --query …`, i.e. `StartBatchOperation`), but it can't do this cleanup. Both `StartBatchOperation` and the per-workflow delete it runs are redirected to the namespace's **active** cluster, so the batch job executes on the active and its deletes never touch the standby — and there's no way to make it send `xdc-redirection: false`. The admin delete is the only one that acts locally on the standby, so you have to drive it yourself.

> If plain `temporal workflow delete` seems to do nothing on the standby, the usual cause is the CLI **not sending the `xdc-redirection: false` header**, so the request goes to the active instead. The server-side delete itself *does* work on a standby for **finished** workflows. Check that your CLI actually sends `--grpc-meta xdc-redirection=false`; older versions may not.

> **Visibility deletes are asynchronous.** The visibility entry is removed by a background task, so right after a large delete run the visibility store lags the real deletions and its row count catches up over time — don't read an immediate row count as "visibility isn't being deleted." To check whether those background tasks are keeping up, watch the **Visibility** row on the [Temporal Server Dashboard](../observability/dashboards/server/temporal-server-readme.md#16-visibility) — the **Visibility Task End-to-End Latencies** panel (how long tasks wait before they're processed) and **Visibility Task Processing by Operation**.

### 2.2 Let the scavenger remove the leftover history (the history gap)

The scavenger is what removes **leftover history** — history whose workflow has already been deleted. You have leftover history in two cases:

- a cluster's own deletes failed earlier under database stress; or
- the admin delete in [2.1](#21-clear-the-left-behind-workflows-on-the-standby-the-executions-gap) couldn't remove some workflows' history — it skips a workflow's history when it can't read that workflow's record (so it can't find the history) or when a history delete returns an error (it logs `Failed to delete history branch, skip` at `WARN` and moves on). These skips are logged only — there's no metric for them.

By default the scavenger won't touch a branch until it is 60 days old, so leftovers can sit for weeks. Lower that wait so it clears them promptly:

```yaml
worker.historyScannerDataMinAge:
  - value: "1h"
```

On its next scan, the scavenger stops skipping the older branches and starts checking them: any whose workflow is already gone (the leftovers) are deleted, and any whose workflow still exists are kept. As it works through them, the leftover history is removed. You should see, in order:

- on the [scavenger metrics](#11-watch-the-scavengers-own-metrics-the-history-gap), `scavenger_skips` fall (it is no longer skipping those branches for age);
- the [branches-per-workflow](#15-compare-branches-per-workflow-the-history-gap) ratio drop back toward **1.0** (the healthy level, where history branches roughly match live executions);
- the [table sizes](#13-compare-the-largest-tables) shrink.

A full pass over every shard can take **days** on a large cluster, and it can take several passes (the scavenger scans about every 12 hours) to reach a steady size.

> **Let the scavenger clear leftover history — don't delete `history_tree` / `history_node` rows by hand.** The manual delete in [2.1](#21-clear-the-left-behind-workflows-on-the-standby-the-executions-gap) is for whole stranded workflows (the `executions` rows). For leftover *history*, the scavenger is safer: it only removes a branch once it has confirmed the branch's workflow is truly gone, and it keeps clearing new leftovers on later scans instead of being a one-time cleanup.

---

## 3. Prevent — stop it recurring

The two fixes above clean up what has already built up. To stop the gap forming again:

### 3.1 Copy workflow deletions to the standby (the executions gap)

On **1.31**, turn on the setting so future deletes on the active are replicated to the standby:

```yaml
history.enableDeleteWorkflowExecutionReplication:
  - value: true
```

After this, a delete on the active is copied to the standby, so both clusters remove the workflow. The setting only replicates deletes for namespaces that are **active in the cluster where you set it**, so **set it on every cluster that hosts global namespaces — active or standby.** (Set it on only one cluster and you cover only the namespaces active there.)

> This only affects deletes **from now on**. Workflows already stranded on the standby (deleted on the active before you turned this on) are not removed by enabling it — clear those with [2.1](#21-clear-the-left-behind-workflows-on-the-standby-the-executions-gap), or leave them to age out through the standby's own retention.

> This setting only addresses the executions gap (deletes that never reach the standby). It does **not** cover leftover history from failed deletes: if a cluster's own history-deletes keep failing under database stress, history is left behind regardless of this setting — on the active or the standby. That is the scavenger's job; keep its wait short as in [Keep the scavenger's wait short enough](#32-keep-the-scavengers-wait-short-enough-the-history-gap).

> In a future Temporal version, `history.enableDeleteWorkflowExecutionReplication` is removed and delete replication is always on, so there's no setting to turn on. See [Server version differences](#server-version-differences).

### 3.2 Keep the scavenger's wait short enough (the history gap)

The scavenger only has work to do when leftover history exists — after a cleanup like the one above, or if a cluster's own deletes fail. Its default **60-day wait** means any leftover sits for up to two months before the scavenger will touch it. Lower `worker.historyScannerDataMinAge` so leftovers are cleared promptly:

- `1h` is fine and safe, including for a one-time heavy cleanup.
- A value **up to about one week** is a reasonable everyday setting: enough margin that the scavenger never touches a branch that was just created, while still clearing leftovers quickly.

This is a **single cluster-wide setting** — no per-namespace version — and it applies whether the cluster is the active or the standby for a namespace.

> **When the change takes effect.** The scavenger applies this value while it scans, and it scans about every 12 hours. Changing the config does **not** start a scan — so a new value takes effect on the **next scheduled scan** (up to ~12 hours later), not the moment you set it. If the scavenger just finished a scan, expect up to a 12-hour wait before the lower value does anything.
>
> **Don't terminate the scanner to speed it up.** It's a **cron** workflow — terminating it *stops the schedule* (no more automatic scans). The Worker service only re-creates it on restart, and a fresh cron workflow still waits for the next cron time, so terminating gives you no faster run and risks leaving the scavenger stopped.
>
> **Check how long until the next scan.** The scanner is a cron workflow, so between runs its current execution is waiting out a start-delay until the next 12-hour tick. See when that is:
>
> ```bash
> temporal workflow describe \
>     --address <cluster> --namespace temporal-system \
>     --workflow-id temporal-sys-history-scanner \
>     --output json
> ```
>
> In the output, `executionTime` is when the next scan is scheduled to start (the run's start time plus its remaining backoff). Compare it to now to see how long you'd wait.
>
> **If that's too long to wait, trigger the scan with SignalWithStart — not a plain Signal.** During the start-delay a plain Signal is only buffered and won't wake the run, but SignalWithStart skips the delay and schedules the scan immediately. Target the scanner's own identifiers: workflow id `temporal-sys-history-scanner`, workflow type `temporal-sys-history-scanner-workflow`, task queue `temporal-sys-history-scanner-taskqueue-0` (the signal name doesn't matter — the workflow ignores it). SignalWithStart isn't exposed in every `temporal` CLI version, so you may need `tctl` or an SDK call.

### 3.3 What a healthy steady state looks like

After the fixes above, don't expect the standby's database to exactly match the active's. Each cluster runs its own cleanup independently and on its own timing, so the two are never identical moment to moment — a **small, steady gap is normal**. What matters is that the gap stays roughly stable rather than growing: if the standby keeps pulling further ahead of the active, one of the two gaps is still open — go back to [Detect](#1-detect--is-the-standbys-database-growing-larger-than-the-actives). (A burst of workflow resets or a recent failover can widen the gap for a while before it settles.)

---

## Server version differences

**`history.enableDeleteWorkflowExecutionReplication`** (the executions gap):

| Version | Behavior |
|---|---|
| 1.30 and earlier | Setting does **not exist** — deletes on the active are never replicated to the standby, and you can't turn replication on. |
| 1.31 (current) | Setting exists, **defaults to `false`**. Early deletes are **not** replicated unless you set it `true`. |
| A future version | Setting **removed** — replicating deletes is **always on** (for namespaces active in the cluster). No action needed. |

**`worker.historyScannerDataMinAge`** (the history gap): default **60 days**, and this default is the same on all versions here (1.30 through current). It's a setting you adjust, not something tied to a version. Lower it as in [Keep the scavenger's wait short enough](#32-keep-the-scavengers-wait-short-enough-the-history-gap).

---

## Config & metric reference

**Dynamic config**

| Key | Default | Effect |
|---|---|---|
| `history.enableDeleteWorkflowExecutionReplication` | `false` (1.31; removed / always-on in a future version) | When true, a workflow deleted on the active is also deleted on the standby. |
| `worker.historyScannerDataMinAge` | `60` days | The scavenger skips any history branch created more recently than this. Lower it (e.g. `1h`) so leftovers are removed promptly. |
| `worker.historyScannerEnabled` | `true` | Whether the history scavenger runs. On by default on every cluster, including the standby. |
| `worker.historyScannerVerifyRetention` | `true` | When true, the scavenger also deletes completed workflows older than retention plus `executionDataDurationBuffer` — a late retention backstop. |
| `worker.executionDataDurationBuffer` | `90` days | The buffer added to retention for the check above. A workflow is only deleted by that backstop once it is older than retention + this buffer. |

**Scavenger workflow / queue**

| Name | Value |
|---|---|
| Workflow ID | `temporal-sys-history-scanner` |
| Queue | `temporal-sys-history-scanner-taskqueue-0` (note the `-0`) |
| Namespace | `temporal-system` |

**Metrics** (all tagged `operation="HistoryScavenger"`)

| Metric | What it counts |
|---|---|
| `scavenger_skips` | branches skipped because of the 60-day wait (`historyScannerDataMinAge`) |
| `scavenger_success` | branches handled without error — **kept or deleted** (not "deleted") |
| `scavenger_errors` | branches that errored (not deleted) |

These counters are graphed on the **History Scavenger** row of the [Server](../observability/dashboards/server/temporal-server-readme.md#22-history-scavenger) and [Standby](../observability/dashboards/server/temporal-standby-readme.md#8-history-scavenger) dashboards. No metric counts actual deletions; confirm real cleanup with the branch-count SQL ([1.4](#14-count-leftover-history-branches-the-history-gap) / [1.5](#15-compare-branches-per-workflow-the-history-gap)) or table size ([1.3](#13-compare-the-largest-tables)).

**Alert**

| Alert | Fires when |
|---|---|
| `temporal-alert-085` — History Scavenger Errors | `scavenger_errors` exceeds ~5 branch errors over 24h — the scavenger is failing to process branches (a different problem than the 60-day wait), so leftover history isn't being cleared. Investigate as in [1.1](#11-watch-the-scavengers-own-metrics-the-history-gap). |

This alert is **documented but not in the essential alert set** — it's multi-cluster-specific and warning-level, so provision it yourself if you run global namespaces on SQL (see [Alert 85 in the alert index](../observability/alerts/server/alerts-index.md#alert-85--history-scavenger-errors)). It's also the only metrics-based alert for this scenario: the core growth signal — the standby's tables outgrowing the active's — is a **database size** measurement, not a Prometheus metric, so it isn't alertable from server metrics; watch that with the SQL checks in [Detect](#1-detect--is-the-standbys-database-growing-larger-than-the-actives).

**Handy queries during an incident**

- Largest tables per cluster — [1.3](#13-compare-the-largest-tables)
- Leftover-branch count — [1.4](#14-count-leftover-history-branches-the-history-gap)
- Branches per workflow — [1.5](#15-compare-branches-per-workflow-the-history-gap)
- Replication backlog on the active — `SELECT count(*) FROM replication_tasks`
