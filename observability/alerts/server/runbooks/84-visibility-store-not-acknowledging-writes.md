## Visibility Store Not Acknowledging Writes

**Severity:** Warning
**Component:** history
**Store types:** Elasticsearch only
**Dashboard panel:** [Visibility Errors by Type per Store](https://github.com/tsurdilo/temporal-metrics/blob/main/observability/dashboards/server/temporal-server-readme.md) — Visibility group
**Playbook:** [Dual Visibility Operations](https://github.com/tsurdilo/temporal-metrics/blob/main/playbooks/dual-visibility.md)

### What this alert detects

`visibility_persistence_error_with_type` with `error_type="persistence_TimeoutError"` above 0.1/s for 5 minutes, on the store named by `visibility_index_name`.

An Elasticsearch visibility write was handed to the bulk processor and never confirmed inside `worker.ESProcessorAckTimeout` (default 30s). The store is reachable enough to accept the connection but is not completing the write.

### Why this alert exists

**Nothing else catches this failure.** That is the whole reason for it.

`persistence.TimeoutError` sits in the no-op arm of the server's visibility error classifier, so it is **not** counted by `visibility_persistence_errors`. That is the metric behind alerts **59a / 59b / 59c**, so none of them fire. And because the bulk request never completes, Elasticsearch never returns an HTTP status, so the **ES Bulk Processor Errors by HTTP Status** panel records nothing either.

Confirmed on a live cluster: pausing Elasticsearch (reachable, not responding) left the write-error panel **and** the bulk-processor error panel completely flat, while this metric was the only thing that moved.

Two consequences worth internalising:

- A flat **Visibility Write Error Rate per Store** panel is not proof that visibility is healthy.
- An `http 0` on the bulk processor panel means Elasticsearch was **gone**, not slow. A hung store shows nothing there.

### Why it matters

Every failing attempt counts toward `history.TaskDLQUnexpectedErrorAttempts`, so the **same 70-minute data-loss clock** is running as for a store that is fully down. A store that hangs for an hour drops visibility records exactly like one that is unreachable for an hour — see alert **83**.

Treat a sustained firing of this alert with the same urgency as alert 59b, despite the lower severity label. The severity is warning because a brief timeout spike under load is not itself an incident; a sustained one is.

### Triage steps

1. Read `visibility_index_name` from the alert. That is your Elasticsearch index name — compare it against your persistence config to tell whether the **primary** store is affected (workflow list and describe degrade for anyone reading from it) or the **secondary** (nothing user-facing breaks, but the clock is still running).
2. Open **ES Bulk Processor Queue Depth**. Climbing means the processor cannot keep up. Because documents are handed to the processor over a channel with no queue behind it, a saturated processor **blocks** history's visibility workers rather than rejecting writes — so this shows as visibility work slowing down, not as errors.
3. Open **ES Write Confirm Latency vs Ack Timeout**. Anything reaching the 30s line is failing as a timeout.
4. Check **Visibility Task Retry Depth** to see how close tasks are to the 70-attempt limit, and therefore how much time is left.
5. Check Elasticsearch itself: cluster health, heap and garbage collection, and whether the index has been flipped to read-only.

### Common causes

- **Elasticsearch overloaded or garbage collecting.** The cluster accepts connections but cannot commit bulk requests fast enough.
- **An index flipped to read-only by a disk watermark.** A cluster low on disk sets `index.blocks.read_only_allow_delete`, and writes stop completing. Check the flood-stage watermark; this is a common cause on small or single-node clusters and looks like a Temporal bug from the outside.
- **A bulk processor that cannot keep up with write volume.** Tune `worker.ESProcessorBulkActions`, `worker.ESProcessorBulkSize`, `worker.ESProcessorFlushInterval` and `worker.ESProcessorNumOfWorkers` — but note all four are read once at history startup and need a **history restart** to take effect. Only `worker.ESProcessorAckTimeout` is read live.

### Remediation

**1. Relieve Elasticsearch.** Restore headroom, clear the disk watermark, or reduce write volume. Nothing else fixes the underlying condition.

**2. Buy time if the clock is close.** `history.TaskDLQUnexpectedErrorAttempts` is dynamic config and can be raised **during** the incident to extend the 70-minute window before records start being dropped.

**3. Consider moving reads** if the affected store is the one serving reads and queries are timing out for users — `system.enableReadFromSecondaryVisibility`, effective in seconds with no restart. This does not fix writes; it just stops users seeing the failure. See the playbook.

**4. Do not** raise `worker.ESProcessorAckTimeout` as a first move. It is live-reloadable, so it is tempting, but a longer timeout means each failing attempt holds a visibility worker for longer, which slows the whole visibility pipeline. It is a tuning knob for a genuinely slow cluster, not an outage remedy.

### Why SQL clusters never fire this

SQL visibility writes are synchronous: history issues the statement and waits, and an unreachable database fails immediately with an unavailable error, which alerts 59a and 59b do catch. The asynchronous hand-off-and-wait-for-acknowledgement path that produces `persistence_TimeoutError` exists only for Elasticsearch.

### Relevant dynamic config

| Key | Default | Live-reloadable | Effect |
|---|---|---|---|
| `worker.ESProcessorAckTimeout` | `30s` | **Yes** | How long the store waits for a write to be confirmed |
| `worker.ESProcessorBulkActions` | `500` | No — history restart | Documents per bulk flush |
| `worker.ESProcessorBulkSize` | `16777216` | No — history restart | Bytes per bulk flush |
| `worker.ESProcessorFlushInterval` | `1s` | No — history restart | Maximum time before a flush |
| `worker.ESProcessorNumOfWorkers` | `2` | No — history restart | Concurrent bulk workers |
| `history.TaskDLQUnexpectedErrorAttempts` | `70` | **Yes** | Attempts before records are dropped |

All of these are read by the **history** service despite the `worker.` prefix.
