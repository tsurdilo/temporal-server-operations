## Replication Stream Status Messages Stopped

**Severity:** Warning
**Component:** history
**Dashboard panels:** [Replication Streams Received from Each Remote Cluster](../../../dashboards/server/temporal-server-readme.md#18-cluster-replication) (3202), [Replication Task Cleanup Runs and Failures](../../../dashboards/server/temporal-server-readme.md#18-cluster-replication) (3203), [Replication Tasks Sent per Second by Task Type](../../../dashboards/server/temporal-server-readme.md#18-cluster-replication) (3201), [Stream Health](../../../dashboards/server/temporal-server-readme.md#18-cluster-replication) (2001)

### What this alert detects

Each replication stream a cluster receives sends a status message back about every second
(`history.ReplicationStreamSyncStatusDuration`, default 1s), whether or not anything is being
replicated. This alert fires when those messages have stopped for 10 minutes for one remote
cluster. The `to_cluster` label names that remote cluster by its initial failover version
(`initialFailoverVersion` in its cluster metadata), not by its name.

### Why it matters

The most common cause is the remote cluster being down or unreachable. While that lasts, this
cluster cannot deliver replication tasks to it either. It keeps every replication task the remote
cluster has not acknowledged, for every namespace replicated to it, and its replication task table
grows with every workflow update in those namespaces.

**Nothing is lost while the remote cluster is only away.** When it reconnects, the kept tasks are
sent and the table shrinks again.

### Check this first

- **[Replication Task Cleanup Runs and Failures](../../../dashboards/server/temporal-server-readme.md#18-cluster-replication)**:
  cleanup runs at zero while workflows are being updated confirms that replication tasks are
  being kept.
- **[Stream Health](../../../dashboards/server/temporal-server-readme.md#18-cluster-replication)**:
  `replication_stream_error` rising while this cluster tries to reconnect.
- Is the remote cluster up? `temporal operator cluster describe` against its frontend.
- On SQL persistence, the size of the backlog is the row count of `replication_tasks` in this
  cluster's database.

### What to do

**Follow the playbook. It is the full procedure for this alert, and this runbook does not
repeat it:**

> **[Multi-Cluster Replication — Load and Settings on Each Cluster](../../../../playbooks/replication-load-on-each-cluster.md)**

In short:

1. **Bring the remote cluster back if you can.** Nothing else is needed; the kept tasks drain.
2. **If it will be away for a long time and the table growth is a problem,** remove it from this
   cluster's cluster list with `temporal operator cluster remove`.
   `temporal operator cluster upsert --enable-connection=false` does not work while the remote
   cluster cannot be reached, because it contacts the remote cluster first.
3. **After a removal, when the remote cluster is back:** add it again with
   `temporal operator cluster upsert`, then run force-replication for every workflow changed
   while it was removed. Removing it let this cluster delete every replication task the remote
   cluster had not received, so those changes reach it only through force-replication.

### What not to do

**Do not remove the remote cluster just because this alert fired.** A short outage drains by
itself; removal turns it into a force-replication job.

### Gaps in this alert

- If a history service instance restarts while the remote cluster is unreachable, its series is
  not recreated, so the alert can go quiet although the stream is still down. Read the panel.
- It fires per remote cluster, on the sum over all shards. A single shard whose stream has stopped
  does not bring the sum to zero; the planned stream-stuck alert (64) is the one for that.
- A single cluster with no remote cluster connected has no series and never fires.
