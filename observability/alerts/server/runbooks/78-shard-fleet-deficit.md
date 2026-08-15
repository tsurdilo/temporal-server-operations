## Shard Fleet Deficit

**Severity:** Critical
**Component:** history
**Dashboard panel:** [Owned Shards (Total)](https://github.com/tsurdilo/temporal-metrics/blob/main/observability/dashboards/server/temporal-server-readme.md) — panel ID 2120, Shard Movement group

### What this alert detects

The sum of `numshards_gauge` across every history pod has stayed below the cluster's configured total shard count (`NumberOfShards`) for 15 minutes straight. This is a direct coverage check: it fires when at least one shard has **no owner at all**, regardless of whether that shard is still generating churn events.

### Why it matters

Alert 34 (Unexpected Shard Movement) watches `sharditem_created_count` / `removed_count` / `shard_closed_count` — rate metrics. A shard that is actively bouncing between hosts shows up there. But a shard that gets **permanently stuck** — every acquisition attempt failing an identical conditional check — often stops generating churn once it settles into that failure loop, because the same host keeps retrying against the same rejected precondition rather than continuing to move. That can go quiet on the churn-rate panels while still being completely unowned. `numshards_gauge` catches this directly: it's a coverage count, not a rate, so a permanently missing shard shows up as a permanent dip that never recovers, independent of whether churn is happening.

A brief dip (seconds to low minutes) during a rolling restart or scale event is expected and normal — pods release shards and other pods pick them back up. The 15-minute `for` window is set well beyond that recovery time specifically so this alert doesn't fire on routine deploys.

### Triage steps

1. Confirm which shard(s) are missing. Check history pod logs for repeating `Failed to update shard` / `Couldn't acquire shard` messages — these name the specific shard ID(s) failing to acquire.
2. Run `tdbg shard describe --shard-id <ID>` for each affected shard. This works even while the shard is stuck (it's a direct persistence read, not gated on the shard having an active engine). Look for an `Owner` that is a history pod no longer in the cluster, and a stale `UpdateTime`.
3. Check whether Alert 79 (Shard Ownership Loss Persisting) is also firing — the two conditions frequently co-occur, since a permanently unacquirable shard produces both a coverage deficit and a sustained `ShardOwnershipLostError` rate.
4. If the shard is genuinely stuck (not just mid-handoff), and **if Cassandra persistence is used**, the most common root cause is a `range_id` divergence: the shard row's `range_id` column and the `RangeId` embedded in its serialized blob get out of sync, so every acquisition attempt's conditional `UPDATE ... IF range_id = ?` fails against the same mismatch forever. **No `tdbg` command or AdminService RPC can repair this** — `tdbg shard close-shard` only evicts an in-memory context and writes nothing to Cassandra; there is no shard-repair capability in the tooling at all. Fixing it requires a direct, conditional Cassandra `UPDATE` to the `range_id` column. Full diagnosis, exact repair statements, guardrails, and post-fix verification steps: [shard-range-id-divergence-stuck-child-workflows.md](https://github.com/tsurdilo/temporal-metrics/blob/main/tmp/shard-range-id-divergence-stuck-child-workflows.md). For SQL-backed persistence (PostgreSQL/MySQL), the shard store implementation differs and this playbook's diagnosis does not directly apply — investigate the equivalent shard-row state in your SQL backend separately.
5. Once the shard is confirmed re-acquired (owner flips to a live pod, `UpdateTime` becomes current), this metric should recover to the full configured count within one `acquireShards` sweep. If it doesn't, check for a second, independently stuck shard rather than assuming the same fix needs repeating.
6. Downstream impact check: if this incident ran for an extended period (hours to months), assume the affected shard's transfer/timer task backlog needs separate attention after the fix — including any tasks that reached the history task DLQ during the outage. The linked playbook covers this in its "scope the real blast radius" and "resuming stuck parent workflows" sections.

### Relevant dynamic config

- `history.acquireShardInterval` — how frequently history pods sweep for unowned shards to acquire (default 1m). Does not help a shard stuck on a genuine `range_id` mismatch — the sweep will keep retrying and keep failing identically.
- `history.TaskDLQEnabled` / `history.TaskDLQUnexpectedErrorAttempts` (default `true` / `70`, ~75-80 minutes of backoff) — governs when tasks queued against the stuck shard get moved to the DLQ instead of continuing to retry. Relevant when assessing the downstream blast radius after the shard is fixed.
