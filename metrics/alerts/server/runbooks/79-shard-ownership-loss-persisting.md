## Shard Ownership Loss Persisting

**Severity:** Critical
**Component:** history
**Dashboard panel:** [Persistence Errors Total by Operation](https://github.com/tsurdilo/temporal-metrics/blob/main/metrics/dashboards/server/temporal-server-readme.md) — panel ID 77 (filter to `operation="UpdateShard"`)

### What this alert detects

`persistence_error_with_type{operation="UpdateShard", error_type="persistence.ShardOwnershipLostError"}` has a non-zero rate continuously for at least 10 minutes. This is the persistence layer's conditional write (a Cassandra `IF range_id = ?` LWT, or the SQL-backend equivalent) failing on a shard's ownership-renewal attempt.

### Why it matters

A single, brief occurrence is not a problem — every normal shard ownership handoff produces exactly one of these as the losing pod's next renewal attempt fails. A **sustained** non-zero rate over 10+ minutes means a shard is failing this same conditional check over and over without ever succeeding — the shard is not just moving, it's stuck.

### Triage steps

1. Check history pod logs for `Persistent store operation failure` (tagged `operation=UpdateShard`) — this carries the exact `previous_range_id` / `columns` values from the failed CAS attempt and identifies the shard ID.
2. Check whether Alert 78 (Shard Fleet Deficit) is also firing. These two conditions typically co-occur: a shard that can't complete its CAS check also can't be acquired, so it drops out of the fleet's owned-shard count at the same time this error rate rises.
3. Run `tdbg shard describe --shard-id <ID>` for the affected shard — confirm a stale `Owner` (a host no longer in the cluster) and an old `UpdateTime`.
4. If the shard is genuinely stuck rather than mid-handoff, and **if Cassandra persistence is used**, this is most commonly a `range_id` divergence between the shard row's `range_id` column and the `RangeId` embedded in its serialized blob — a mismatch that makes every conditional `UpdateShard` attempt fail identically forever, with no path to self-heal. Full diagnosis, exact repair CQL, guardrails, and post-fix verification: [shard-range-id-divergence-stuck-child-workflows.md](https://github.com/tsurdilo/temporal-metrics/blob/main/tmp/shard-range-id-divergence-stuck-child-workflows.md).
5. If Cassandra persistence is used, there is no `tdbg`/AdminService command that can fix this directly — `tdbg shard close-shard` only evicts an in-memory context on whichever host currently holds it and writes nothing to Cassandra. The repair requires a direct, conditional Cassandra `UPDATE` to the `range_id` column (see the linked playbook for the exact statements and safety guardrails). For SQL-backed persistence (PostgreSQL/MySQL), the shard store implementation differs and this playbook's diagnosis does not directly apply — investigate the equivalent shard-row state in your SQL backend separately.
6. After the fix, this error rate should drop back to an occasional, isolated blip (normal handoff behavior) rather than a sustained rate. If it doesn't, check for a second stuck shard.

### Relevant dynamic config

- None.
