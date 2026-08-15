## Unexpected Shard Movement

**Severity:** Critical
**Component:** history
**Dashboard panel:** [Shards Created](https://github.com/tsurdilo/temporal-metrics/blob/main/observability/dashboards/server/temporal-server-readme.md) — panel ID 60

### What this alert detects

Any non-zero rate of shard creation (`sharditem_created_count`), shard removal (`sharditem_removed_count`), or shard closing (`shard_closed_count`) on the history service without a corresponding history service restart in the same 8-minute window. Shard movement during a restart is expected and suppressed.

### Why it matters

Unexpected shard movement means the history service is redistributing shard ownership without a pod restart. This typically indicates DB pressure causing shard ownership loss, membership instability, or a history pod that is alive but unable to maintain its shard leases. Shard movement causes in-flight workflow tasks to fail and be retried on the new owner, adding latency spikes across all affected workflows.

### Triage steps

1. Open the **Shards Created** (panel 60), **Shards Removed**, and **Shards Closed** panels in the Temporal Server dashboard — confirm which type of movement is occurring and when
2. Check **Service Restarts** (panel 63) alongside — if restarts are present the alert should not have fired; investigate if the compound condition did not suppress correctly
3. Check **Persistence Latencies** (panel 71) — shard ownership loss under DB pressure is the most common cause; check if alert 12 is also firing
4. Check **Shard Lock Latency** (panel 14) — elevated lock latency on the losing pod can cause it to miss lease renewals
5. Check history pod logs for shard ownership transfer messages

### Relevant dynamic config

- `history.acquireShardInterval` — how frequently history pods attempt to acquire unowned shards (default 1m); lower values mean faster rebalancing after a legitimate restart but more DB load
- `history.shardUpdateMinInterval` — minimum interval between shard info updates (default 5m); if too high, pods may miss lease renewals under load
