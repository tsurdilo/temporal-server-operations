## History Write-Reject Loop

**Severity:** Critical
**Component:** history
**Store types:** All
**Dashboard panel:** [Write-Reject Loop Indicator (cleared / cache miss)](https://github.com/tsurdilo/temporal-metrics/blob/main/observability/dashboards/server/temporal-server-readme.md) — Persistence group. The panel plots the ratio on its own; this alert adds a second condition, so the panel can show a high ratio while the alert stays quiet.
**Playbook:** [History Task Processing — Tuning and Troubleshooting Playbook](https://github.com/tsurdilo/temporal-metrics/blob/main/playbooks/history-task-processing-tuning.md)

### What this alert detects

Two conditions at once, both sustained for 10 minutes:

| | |
|---|---|
| `workflow_context_cleared` ÷ `cache_miss{cache_type="mutablestate"}` | **above 5** |
| `persistence_errors_resource_exhausted{service_name="history",resource_exhausted_cause="PersistenceLimit"}` | **above 10/s** |

In words: the history service is throwing away cached workflow state far faster than that state is genuinely missing from the cache, while Temporal's own rate limiter is refusing its database calls.

### What is happening

1. A task's `UpdateWorkflowExecution` is refused by the persistence rate limiter.
2. The pod cannot know how far the write got, so it discards the cached mutable state — that is the `workflow_context_cleared` count.
3. The task retries. With no cached state, the retry must begin with `GetWorkflowExecution`, which on a SQL store is **nine** queries and on Cassandra **one** — but the limiter counts **one token per operation** either way.
4. Those reads spend the same budget the writes need, so more writes are refused, and step 1 repeats.

The loop sustains itself: the work it generates is the reason it keeps going.

### What it is not

- **Not a cache that is too small.** That shows up as a high genuine cache-miss rate. This alert requires the opposite — misses low while clearing is high. Resizing `history.hostLevelCacheMaxSize` will not help.
- **Not a slow database.** The limiter turned the calls away before the store saw them. Check **Persistence Latencies**: flat latency through the event is the normal picture here.
- **Not a read problem, even though reads dominate.** `GetWorkflowExecution` will be at the top of the rejected-operations list, well ahead of `UpdateWorkflowExecution`. The reads are the consequence; the refused writes are the cause. An attempt whose read is refused never gets as far as its write.

### The fix is not to raise the persistence limit

Raising it lets more of the same loop through. The fix is to admit fewer tasks so the tasks that are admitted can finish — the `history.taskScheduler*` rate limits, which are **off by default**. Sizing them, rolling them out in shadow mode first, and checking the result are what the playbook covers.

### If you raise the limit anyway

Sometimes the limit really is too low for the cluster, and section 5 of the playbook says how to tell. Check **Persistence Latencies** first: flat means the limiter was the only constraint and there is headroom; climbing means the database was already the constraint and raising the limit moves the problem into the store.

### Relationship to alert 86

Alert 86 (*History Database Calls Rejected*) fires on the refusals alone, at the same 10/s. Rejections are not always a loop — a burst that drains is the limiter working. **87 fires when the refusals have become self-sustaining.** Expect 86 to be firing whenever 87 is.

### Tuning this alert

The ratio threshold of 5 comes from measurement on a test cluster driven into the loop deliberately: the ratio climbed 5.1 → 6.7 → 9.3 → 10.9 and peaked at 18.2, against a healthy reading below 1. The 10/s rejection gate keeps the alert off clusters whose ratio is noisy simply because both numbers are tiny.

**Do not remove the `> 0` guards from the query.** On an idle cluster both rates are `0`, the bare ratio evaluates to `NaN`, and Grafana's threshold step treats `NaN` as breaching — the alert would fire continuously on a cluster doing nothing. The guards drop the series instead, and `noDataState: OK` keeps it silent.
