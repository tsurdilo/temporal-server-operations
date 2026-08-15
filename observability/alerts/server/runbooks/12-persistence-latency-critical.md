## Persistence Latency Critical

**Severity:** Critical
**Component:** persistence
**Dashboard panel:** [Persistence Latencies](https://github.com/tsurdilo/temporal-metrics/blob/main/observability/dashboards/server/temporal-server-readme.md) — panel ID 71

### What this alert detects

p99 persistence latency exceeding 1s for 5 consecutive minutes, scoped to critical-path DB operations: `CreateWorkflowExecution`, `UpdateWorkflowExecution`, `ConflictResolveWorkflowExecution`, `GetWorkflowExecution`, `GetCurrentExecution`, `AppendHistoryNodes`, `ReadHistoryBranch`, `CreateTasks`, `GetTasks`, `GetTransferTasks`, `GetTimerTasks`, `CompleteTransferTask`, `CompleteTimerTask`.

### Why it matters

Persistence latencies can cause delays writing and reading from the database, causing slowness for critical operations that support forward progress of workflow executions.

### Triage steps

1. Open the **Persistence Latencies** panel (panel 71) in the Temporal Server dashboard — identify which operation(s) are slow and since when
2. Check **Persistence Errors by Namespace** — if errors are also elevated the DB may be rejecting requests, not just slow
3. Check DB host metrics (CPU, IOPS, connection count) — persistence latency spikes often originate at the DB layer, not the Temporal service
4. Check if alert 8 (Shard Lock Latency Critical) is also firing — confirms the DB slowness is cascading into history service contention
5. Check **SQL DB Connection Pool** panel if using a SQL backend — a saturated connection pool causes queuing before requests even reach the DB

### Relevant dynamic config

- `history.persistenceMaxQPS` — max persistence QPS per history host (default 9,000); if the cluster is hitting this limit, requests queue before reaching the DB
- `history.persistencePerShardNamespaceMaxQPS` — per-shard per-namespace persistence QPS cap (default 0, disabled); useful for isolating a noisy namespace saturating DB connections
- `history.shardIOConcurrency` — concurrency of persistence operations per shard (default 1); too low on high-throughput SQL clusters can cause artificial queuing
- `frontend.persistenceMaxQPS` — frontend-side persistence QPS limit (default 2,000)
- `matching.persistenceMaxQPS` — matching-side persistence QPS limit (default 9,000)
