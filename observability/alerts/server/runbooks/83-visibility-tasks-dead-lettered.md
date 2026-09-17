## Visibility Tasks Dead-Lettered

**Severity:** Critical
**Component:** history
**Dashboard panel:** [Visibility Tasks Dead-Lettered by Task Type](https://github.com/tsurdilo/temporal-metrics/blob/main/observability/dashboards/server/temporal-server-readme.md) — Visibility group
**Playbook:** [Dual Visibility Operations](https://github.com/tsurdilo/temporal-metrics/blob/main/playbooks/dual-visibility.md)

### What this alert detects

`dlq_writes` has a non-zero rate for a `VisibilityTask.*` operation, sustained for 5 minutes. Visibility tasks have stopped being retried and are being written to the history task dead letter queue.

A visibility task reaches the DLQ after failing `history.TaskDLQUnexpectedErrorAttempts` times with unexpected errors — default **70**, which with retry jitter is roughly **70 minutes**. Failing to reach a store, an Elasticsearch write that is never acknowledged, and an Elasticsearch rejection all count as unexpected.

A dead-lettered task is removed from the active queue and is **not auto-retried**.

### Why it matters

Workflows are unaffected and keep running normally. What is lost is the visibility record — the row or document that makes a workflow show up in `workflow list`, `count` and `describe`.

Whether the damage repairs itself depends on the `operation` label, because every visibility write is a **complete snapshot** of the workflow, not a delta:

| Operation | Self-repairs? |
|---|---|
| `VisibilityTaskStartExecution` | **Yes** — the workflow's next update or close writes a full snapshot |
| `VisibilityTaskUpsertExecution` | **Yes** — same |
| `VisibilityTaskCloseExecution` | **No** — the workflow stays listed as **running forever** |
| `VisibilityTaskDeleteExecution` | **No** — a record that should have been removed stays |

A dropped close is the worst case: the store shows a workflow running that finished long ago, and nothing in normal operation ever corrects it.

### On Elasticsearch, this alert can over-report data loss

This has been confirmed on a live cluster and is important enough to check before you declare records lost.

The Elasticsearch bulk processor holds documents in memory. The store stops waiting at `worker.ESProcessorAckTimeout` (default 30s) and fails the task — possibly all the way into the DLQ — while the document is **still buffered**. When Elasticsearch recovers, the processor flushes and the document lands after all.

Observed directly: twelve tasks dead-lettered during an induced outage (`dlq_writes` = 12, `tdbg dlq list` showed 12 messages), and after recovery **all twelve records were present in both indices**.

There is a limit to this rescue. The buffer holds roughly `worker.ESProcessorNumOfWorkers` × `worker.ESProcessorBulkActions` documents — about 1,000 at defaults — plus whatever is in flight, so it cannot absorb an unbounded backlog.

**SQL has no such buffer.** On SQL, a dead-lettered visibility task really did lose the write.

### Triage steps

1. Open the **Visibility Tasks Dead-Lettered by Task Type** panel and read the `operation` breakdown. `VisibilityTaskCloseExecution` and `VisibilityTaskDeleteExecution` are the ones that do not repair themselves.
2. Establish **which store** is failing — the DLQ metric carries no store label. Use **Visibility Errors by Type per Store**, whose `visibility_index_name` label is your SQL database name or Elasticsearch index name. Do not rely on **Visibility Write Error Rate per Store** alone: it does not count timeouts, so a store that is hung rather than down leaves it flat.
3. Confirm in history logs: `Marking task as terminally failed, will send to DLQ. Maximum number of attempts with unexpected errors`.
4. Count what is in the queue: `tdbg dlq list`. It needs no flags and is the safe way to see whether a visibility queue exists and how much is in it.
5. **On Elasticsearch, check the store before assuming loss** — count documents in the index and compare against what you expect for the outage window. See the section above.

### Remediation

**1. Fix the failing store.** Nothing improves until the store accepts writes again. Follow the scenario that matches in the [dual visibility playbook](https://github.com/tsurdilo/temporal-metrics/blob/main/playbooks/dual-visibility.md).

**2. Replay the dead-lettered tasks.** Once the store is healthy:

```
tdbg dlq read --dlq-type 4 --max-message-count 100 --last-message-id <id>
tdbg dlq merge --dlq-type 4 --last-message-id <id>
```

`4` is the numeric category id for visibility. On **v1.31.0** the name form (`--dlq-type visibility`) fails with `strconv.Atoi: parsing "visibility": invalid syntax`; newer builds accept the name. Also note that omitting `--last-message-id` makes the command prompt for confirmation, and with no terminal attached (a `docker exec` without `-t`) that prompt **panics on EOF** rather than failing cleanly.

Replaying is safe in any order: each write carries its own version and a store only applies a write newer than what it holds.

**3. Rebuild anything the DLQ cannot replay.** For workflows whose close was dropped and cannot be merged, regenerate the visibility task from mutable state:

```
tdbg --namespace <ns> workflow refresh-tasks --workflow-id <wid> --run-id <rid>
```

In bulk, driven by a visibility query (available from **v1.30.0**):

```
tdbg --namespace <ns> workflow refresh-tasks \
  --query 'CloseTime > "2026-01-01T00:00:00Z"' \
  --reason "rebuild dropped visibility records"
```

Read the caveats in the playbook before running this in bulk — it refreshes **every** task category, including archival tasks, and it writes to both stores.

Refreshing a **running** workflow is safe but not free: an activity that has already started is skipped (never re-run), while one that is scheduled but not started is dispatched again and the duplicate is rejected at start. Timers are regenerated at their original fire times.

The cost is load, not correctness. A refresh works from mutable state, so per workflow it is a `GetWorkflowExecution` read against your **main** persistence store (only on a cache miss — but closed workflows are rarely cached) plus a write back. On SQL that read is **nine** sequential statements, not one, so at the default 50 workflows/second it is on the order of 450 reads/second against your main database before any writes; on Cassandra mutable state comes back in a single query, so it is much cheaper there. It also evicts live workflows from the **host-level** history cache, so ordinary traffic sees a degraded hit rate and extra reads for a while afterwards. A bulk refresh therefore loads your main database, both visibility stores, and your task queues at once — prefer several narrow time windows over one wide one.

If you only need dropped **close** records, bound the query on `CloseTime` rather than `StartTime` to exclude running workflows entirely.

The batch form is also slower than most people expect, because it runs as an **admin batch operation** with its own throttles: `frontend.MaxConcurrentAdminBatchOperationPerNamespace` is **1** (a second job in the same namespace is rejected, not queued), `worker.batcherRPS` is **50** per namespace, `worker.batcherConcurrency` is **5**, and `worker.adminBatcherHostRPS` is **100** per worker host. At 50 workflows/second a 100,000-workflow backfill takes roughly half an hour. All of these are dynamic config and can be raised for the duration, but every extra workflow per second is more load on your main store and both visibility stores.

**This only works while mutable state still exists.** Workflows already aged out by retention cannot be refreshed, and a dropped close is exactly the case that never self-heals. Prioritise short-retention namespaces.

### Tuning this alert

Fires on any rate above zero, sustained 5 minutes. Visibility DLQ writes should be zero in normal operation, so any sustained value is worth knowing about. If your cluster legitimately produces occasional visibility DLQ writes, raise the `for` window rather than the threshold — the value here is knowing it happened at all.

To buy more time before records are dropped, raise `history.TaskDLQUnexpectedErrorAttempts` (default 70). It is dynamic config and takes effect on the next poll, so it can be raised **during** an outage to extend the clock. Tasks already dead-lettered are not brought back by raising it.

### Relevant dynamic config

| Key | Default | Effect |
|---|---|---|
| `history.TaskDLQEnabled` | `true` | Master switch. If disabled, failing tasks are **dropped** instead of dead-lettered — worse, not better |
| `history.TaskDLQUnexpectedErrorAttempts` | `70` | Attempts before a task is dead-lettered, roughly 70 minutes with jitter |
| `history.TaskDLQInternalErrors` | `false` | When true, `serviceerror.Internal` failures dead-letter on the **first** attempt with no grace period |
| `worker.ESProcessorAckTimeout` | `30s` | Elasticsearch only. How long the store waits for a write to be confirmed before failing the attempt |
