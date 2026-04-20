# Temporal On-Prem Server Dynamic Config Reference

---

## Table of Contents

- [Baseline: Configs to Consider for Every Cluster](#baseline-configs-to-consider-for-every-cluster)
- [All Available Configs](#all-available-configs)
    - [Frontend](#frontend)
    - [History](#history)
    - [Matching](#matching)
    - [System](#system)
    - [Limits](#limits)
    - [Worker](#worker)

---

## Baseline: Configs to Consider for Every Cluster

These are the configs you should review and consciously set (or decide to leave at default) for any production cluster.

| Config Key | Default | Example Value | Description / When to Use |
|---|---|---|---|
| `frontend.rps` | 2400 | — | Overall RPS limit per frontend pod. One of the 3 core configs to adjust when hitting `service_errors_resource_exhausted`. |
| `frontend.globalRPS` | 0 | `6000` | RPS limit for the whole frontend cluster. |
| `frontend.namespaceRPS` | 2400 | — | Per-namespace RPS limit per frontend instance. Adjust when getting "namespace rate limit exceeded". |
| `frontend.globalNamespaceRPS` | 0 | `1600` | Per-namespace RPS distributed evenly across the whole frontend cluster. Use with server 1.18.0+. Overwrites per-instance limit when set. |
| `frontend.namespaceCount` | 1200 | `2000` | Concurrent poller limit per namespace per frontend host. Increase when seeing `RESOURCE_EXHAUSTED: namespace count limit exceeded` or "namespace concurrent poller limit exceeded". |
| `frontend.globalNamespaceCount` | 0 | `20000` | Global concurrent long-running requests limit per namespace across all frontend instances. |
| `frontend.persistenceMaxQPS` | 2000 | `2400` | Frontend persistence max QPS per host. Increase when hitting "Persistence Max QPS Reached". |
| `frontend.persistenceGlobalMaxQPS` | 0 | — | Frontend persistence max QPS for the whole frontend cluster. |
| `frontend.namespaceRPS.visibility` | 10 | `50` | Per-namespace RPS limit for visibility APIs. **Default is very low — very commonly needs increasing.** Increase when getting `RESOURCE_EXHAUSTED: namespace rate limit exceeded` on `list*` operations. |
| `frontend.globalNamespaceRPS.visibility` | 0 | `500` | Cluster-wide per-namespace RPS for visibility APIs. Alternative to per-instance limit. |
| `frontend.WorkerHeartbeatsEnabled` | true | `true` | Allows workers to send periodic heartbeats to the server. |
| `frontend.ListWorkersEnabled` | true | `true` | Allows clients to retrieve worker heartbeat information. |
| `frontend.WorkerCommandsEnabled` | false | `true` | Allows clients to send commands to workers. |
| `frontend.keepAliveMinTime` | 10s | `10s` | Minimum time a client should wait before sending a keepalive ping. |
| `frontend.keepAlivePermitWithoutStream` | true | `true` | If true, server allows keepalive pings even when there are no active streams. |
| `frontend.visibilityArchivalQueryMaxPageSize` | 10000 | — | Max page size for archival queries. Reduce when UI archival queries cause issues (no restart required). |
| `frontend.MaxConcurrentBatchOperationPerNamespace` | 1 | — | Max concurrent batch operation jobs per namespace. Increase to allow parallel batch jobs. |
| `history.rps` | 3000 | — | RPS limit per history pod. |
| `history.cacheSizeBasedLimit` | false | `true` | If true, limits history cache by bytes instead of entry count. |
| `history.hostLevelCacheMaxSizeBytes` | ~1GB | `100000000` | Max bytes for host-level cache (~100MB in example). Used when `cacheSizeBasedLimit=true`. |
| `history.cacheMaxSizeBytes` | — | `2000000` | Per-shard cache size in bytes. Used alongside `cacheSizeBasedLimit`. |
| `history.persistenceMaxQPS` | 9000 | `16000` | History persistence max QPS per host. Increase when hitting persistence max QPS errors. |
| `history.persistenceGlobalMaxQPS` | 0 | `36000` | History persistence max QPS for whole cluster. |
| `history.persistencePerShardNamespaceMaxQPS` | 0 | `500` | Per-shard per-namespace persistence max QPS. Useful to cap noisy-neighbor namespaces. |
| `history.shardIOConcurrency` | 1 | — | Concurrency of persistence operations per shard. For SQL persistence, increase to reduce shard lock contention. **Requires restart.** |
| `history.MaxBufferedQueryCount` | 1 | — | Max consistent queries buffered while a workflow task is in-flight. Increase to 3–5 to fix "buffered query cleared, please retry" errors. |
| `matching.rps` | 1200 | — | RPS limit per matching pod. |
| `matching.persistenceMaxQPS` | 3000 | `2400` | Matching persistence max QPS per host. Increase when hitting persistence max QPS errors. |
| `matching.numTaskqueueWritePartitions` | 1 | `4` | Number of write partitions per task queue. **Increase for high-throughput task queues. Set to 1 for low-throughput queues.** Can be set per task queue via constraints. |
| `matching.numTaskqueueReadPartitions` | 1 | `4` | Number of read partitions per task queue. Should match write partitions. Can be set per task queue via constraints. |
| `matching.outstandingTaskAppendsThreshold` | 250 | `500` | Buffer size for outstanding task appends before backpressure kicks in. Increase when seeing "Too many outstanding appends to the task queue". **Requires node restart to take effect.** |
| `matching.priorityLevels` | 5 | `10` | Number of simple priority levels. Set higher for richer priority use cases. Requires new matcher. |
| `matching.enableFairness` | false | `true` (per-TQ) | Enable fairness for task dispatching. Can be set per task queue via constraints. |
| `system.secondaryVisibilityWritingMode` | `"off"` | `"dual"` | Controls dual-write to standard and enhanced (ES) visibility. Set to `"dual"` when migrating to ES. Remove or set to `"off"` after migration is complete. |
| `system.enableReadFromSecondaryVisibility` | false | `false` | Whether to read from secondary visibility store. Enable after ES migration is stable. |
| `worker.schedulerNamespaceStartWorkflowRPS` | 30 | `100` | RPS limit for starting workflows from schedules, per namespace as a whole. Increase for high-schedule-volume namespaces. |
| `worker.batcherRPS` | 50 | — | RPS limit for batch operations per namespace. |
| `worker.batcherConcurrency` | 5 | — | Number of concurrent workflow operations within a single batch job per namespace. |
| `worker.perNamespaceWorkerCount` | 1 | `4` | Number of per-namespace workers (scheduler, batcher, etc.) running per namespace. Set to number of worker pods. |
| `worker.perNamespaceWorkerOptions` | — | `MaxConcurrentWorkflowTaskPollers: 25, MaxConcurrentActivityTaskPollers: 10` | SDK worker options for per-namespace workers. Rule of thumb: 1 workflow task poller per 4 schedule RPS desired. |

---

## All Available Configs

## Frontend

| Config Key | Default | Example Value | Description / When to Use |
|---|---|---|---|
| `frontend.rps` | 2400 | — | Overall RPS limit per frontend pod. One of the 3 core configs to adjust when hitting `service_errors_resource_exhausted`. |
| `frontend.globalRPS` | 0 | `6000` | RPS limit for the whole frontend cluster. |
| `frontend.namespaceRPS` | 2400 | — | Per-namespace RPS limit per frontend instance. Adjust when getting "namespace rate limit exceeded". |
| `frontend.globalNamespaceRPS` | 0 | `1600` | Per-namespace RPS distributed evenly across the whole frontend cluster. Use with server 1.18.0+. Overwrites per-instance limit when set. |
| `frontend.namespaceCount` | 1200 | `2000` | Concurrent poller limit per namespace per frontend host. Increase when seeing `RESOURCE_EXHAUSTED: namespace count limit exceeded` or "namespace concurrent poller limit exceeded". |
| `frontend.globalNamespaceCount` | 0 | `20000` | Global concurrent long-running requests limit per namespace across all frontend instances. |
| `frontend.persistenceMaxQPS` | 2000 | `2400` | Frontend persistence max QPS per host. Increase when hitting "Persistence Max QPS Reached". |
| `frontend.persistenceGlobalMaxQPS` | 0 | — | Frontend persistence max QPS for the whole frontend cluster. |
| `frontend.namespaceRPS.visibility` | 10 | `50` | Per-namespace RPS limit for visibility APIs. **Default is very low — very commonly needs increasing.** Increase when getting `RESOURCE_EXHAUSTED: namespace rate limit exceeded` on `list*` operations. |
| `frontend.globalNamespaceRPS.visibility` | 0 | `500` | Cluster-wide per-namespace RPS for visibility APIs. Alternative to per-instance limit. |
| `frontend.namespaceBurstRatio.visibility` | 1 | — | Burst ratio for visibility namespace RPS. Must be >= 1. |
| `frontend.keepAliveMinTime` | 10s | `10s` | Minimum time a client should wait before sending a keepalive ping. |
| `frontend.keepAlivePermitWithoutStream` | true | `true` | If true, server allows keepalive pings even when there are no active streams (RPCs). |
| `frontend.WorkerHeartbeatsEnabled` | true | `true` | Allows workers to send periodic heartbeats to the server. |
| `frontend.ListWorkersEnabled` | true | `true` | Allows clients to retrieve worker heartbeat information. |
| `frontend.WorkerCommandsEnabled` | false | `true` | Allows clients to send commands to workers. |
| `frontend.visibilityMaxPageSize` | 1000 | — | Max page size for `ListWorkflowExecutions`. Increase if needed. |
| `frontend.historyMaxPageSize` | — | — | Max page size for `GetWorkflowExecutionHistory`. Increase via dynamic config when needed. |
| `frontend.visibilityArchivalQueryMaxPageSize` | 10000 | — | Max page size for archival queries. Reduce when UI archival queries cause issues (no restart required). |
| `frontend.searchAttributesNumberOfKeysLimit` | 100 | — | Limits the number of search attributes that can be set on a workflow. |
| `frontend.searchAttributesSizeOfValueLimit` | 2KB | — | Size limit per search attribute value. Can be set per-namespace via constraints. |
| `frontend.searchAttributesTotalSizeLimit` | 40KB | — | Total size limit of all search attributes map. |
| `frontend.MaxConcurrentBatchOperationPerNamespace` | 1 | — | Max concurrent batch operation jobs per namespace. Increase to allow parallel batch jobs. |
| `frontend.namespaceBurstRatio` | 2 | — | Burst ratio for regular namespace RPS (not just visibility). Must be >= 1. The RPS used is the effective RPS from global and per-instance limits. |
| `frontend.shutdownDrainDuration` | 0s | — | Duration of traffic drain during frontend shutdown. Set to allow in-flight requests to complete before pod terminates. |
| `frontend.shutdownFailHealthCheckDuration` | 0s | — | Duration to fail health checks before shutdown begins, allowing load balancers to drain connections. |
| `frontend.enableUpdateWorkflowExecution` | true | — | Feature flag to enable the `UpdateWorkflowExecution` API. |
| `frontend.enableSchedules` | true | — | Enables schedule-related RPCs in the frontend. |
| `frontend.WorkflowPauseEnabled` | false | — | Feature flag to allow clients to pause workflows. |
| `frontend.enableBatcher` | true | — | Feature flag to enable batch operation RPCs in the frontend. |
| `frontend.maxBadBinaries` | 10 | — | Max number of bad binaries in namespace config. |
| `frontend.maskInternalErrorDetails` | true | — | Whether to mask internal error details in responses to clients. |

---

## History

| Config Key | Default | Example Value | Description / When to Use |
|---|---|---|---|
| `history.rps` | 3000 | — | RPS limit per history pod. |
| `history.persistenceMaxQPS` | 9000 | `16000` | History persistence max QPS per host. Increase when hitting persistence max QPS errors. |
| `history.persistenceGlobalMaxQPS` | 0 | `36000` | History persistence max QPS for whole cluster. |
| `history.persistencePerShardNamespaceMaxQPS` | 0 | `500` | Per-shard per-namespace persistence max QPS. Useful to cap noisy-neighbor namespaces. |
| `history.cacheMaxSize` | 512 | — | Number of mutable state entries in the **per-shard** cache. Increase if history pods have cache misses under load. |
| `history.cacheMaxSizeBytes` | — | `2000000` | Per-shard cache size in bytes (used alongside `cacheSizeBasedLimit`). |
| `history.cacheInitialSize` | 128 | — | Initial size of per-shard history cache. |
| `history.hostLevelCacheMaxSize` | 128000 | — | Size of **host-level** mutable state cache (entries). Default since newer versions — controls memory usage of history service. Adjust based on available memory. |
| `history.hostLevelCacheMaxSizeBytes` | ~1GB | `100000000` | Max bytes for host-level cache (used when `cacheSizeBasedLimit=true`). ~100MB in example. |
| `history.cacheSizeBasedLimit` | false | `true` | If true, limits history cache by bytes instead of entry count. |
| `history.enableHostHistoryCache` | true | — | Whether to use host-level cache instead of per-shard cache. Default true in newer versions. |
| `history.cacheTTL` | 1h | — | TTL for history cache entries. Entry is evicted if not accessed for this duration. Usually no need to change. |
| `history.cacheBackgroundEvict` | disabled | — | Background eviction of expired cache entries. Enable with `Enabled: true, LoopInterval: 1m, MaxEntryPerCall: 1024`. |
| `history.cacheNonUserContextLockTimeout` | 500ms | — | How long non-user calls wait for workflow lock. Increase when seeing `ResourceExhausted on BusyWorkflow`. **Requires service restart.** |
| `history.eventsCacheMaxSize` | 512 | — | Max size of shard-level events cache (entries). |
| `history.eventsCacheInitialSize` | 128 | — | Initial size of events cache. |
| `history.eventsCacheTTL` | 1h | — | TTL of events cache. |
| `history.shardIOConcurrency` | 1 | — | Concurrency of persistence operations per shard. For SQL persistence, increase to reduce shard lock contention. **Requires restart.** |
| `history.shardUpdateMinInterval` | 5m | — | Min interval between shard info updates. Increase to reduce background DB writes on idle clusters. |
| `history.transferProcessorMaxPollInterval` | 1m | — | How often TransferTaskProcessor reads from DB on an idle cluster. Increase to reduce idle DB reads. |
| `history.transferProcessorMaxPollHostRPS` | 0 | — | Max poll rate for all transfer processors on a host. Tune to limit DB load during recovery. |
| `history.transferProcessorSchedulerWorkerCount` | 512 | — | Workers in host-level task scheduler for transfer processor. Increase when transfer processing is a bottleneck. **Requires restart.** |
| `history.timerProcessorMaxPollInterval` | 5m | — | How often TimerTaskProcessor reads from DB on idle cluster. |
| `history.timerProcessorMaxPollHostRPS` | 0 | — | Max poll rate for all timer processors on a host. |
| `history.timerProcessorSchedulerWorkerCount` | 512 | — | Workers for timer task scheduler. Increase under load. **Requires restart.** |
| `history.visibilityProcessorMaxPollInterval` | 1m | — | How often VisibilityTaskProcessor reads from DB on idle cluster. |
| `history.visibilityProcessorMaxPollRPS` | 20 | — | Max poll rate for visibility queue processor per host. **Reduce** to throttle ES indexing during backlog drain (e.g. after AWS outage). |
| `history.visibilityProcessorMaxPollHostRPS` | 0 | — | Host-level max poll rate for visibility processor. Tune to limit ES load. |
| `history.visibilityProcessorSchedulerWorkerCount` | 512 | — | Workers for visibility task scheduler. Reduce (e.g. to 128) to slow down ES indexing during recovery. **Requires restart.** |
| `history.visibilityTaskWorkerCount` | — | `0` | Set to `0` to completely block the visibility queue processor (e.g. during ES reindexing to prevent conflicts with new documents). |
| `history.MaxBufferedQueryCount` | 1 | — | Max consistent queries buffered while a workflow task is in-flight. Increase to 3–5 to fix "buffered query cleared, please retry" errors. |
| `history.workflowTaskCriticalAttempt` | 10 | — | Number of consecutive workflow task attempts before flagging as critical. |
| `history.workflowTaskRetryMaxInterval` | 10m | — | Maximum interval added to workflow task `startToClose` timeout for retry backoff. |
| `history.maxTotalUpdates` | 2000 | — | Max number of updates a workflow execution can receive. Set to 0 to disable. |
| `history.maxTotalUpdates.suggestContinueAsNewThreshold` | 0.9 | — | Percentage of `maxTotalUpdates` before suggesting continue-as-new. |
| `history.archivalProcessorArchiveDelay` | 5m | — | Delay before archival queue processor starts processing tasks. |
| `history.archivalTaskBatchSize` | 100 | — | Batch size for archival queue processor. Reduce to slow down archival under load. |
| `history.archivalProcessorMaxPollHostRPS` | 0 | — | Host-level max poll rate for archival processor. Reduce to limit archival I/O. |
| `history.maximumSignalsPerExecution` | 10000 | — | Max number of signals a single workflow execution can receive. |
| `history.maximumBufferedEventsBatch` | 100 | — | Max number of buffered events for any given mutable state. |
| `history.maxInFlightUpdates` | 10 | — | Max number of in-flight updates (admitted but not yet completed) for a workflow execution. Set to 0 to disable. |
| `history.maxInFlightUpdatePayloads` | 20MB | — | Max total payload size of in-flight updates for a workflow execution. Set to 0 to disable. |
| `history.defaultActivityRetryPolicy` | — | — | Default retry policy for activities where user has not specified one. Configurable per namespace. |
| `history.defaultWorkflowRetryPolicy` | — | — | Default retry policy for workflows where user has set a retry policy but not all fields. Configurable per namespace. |
| `history.enableWorkflowIdReuseStartTimeValidation` | false | — | If true, validates old workflow start time is older than `workflowIdReuseMinimalInterval` before allowing ID reuse. Can cause throttling with `BUSY_WORKFLOW` cause if enabled. |
| `history.workflowIdReuseMinimalInterval` | 1s | — | Minimum interval between workflow starts with same ID. Used when `enableWorkflowIdReuseStartTimeValidation` is true. |

---

## Matching

| Config Key | Default | Example Value | Description / When to Use |
|---|---|---|---|
| `matching.rps` | 1200 | — | RPS limit per matching pod. |
| `matching.persistenceMaxQPS` | 3000 | `2400` | Matching persistence max QPS per host. Increase when hitting persistence max QPS errors. |
| `matching.numTaskqueueWritePartitions` | 1 | `4` | Number of write partitions per task queue. **Increase for high-throughput task queues. Set to 1 for low-throughput queues.** Can be set per task queue via constraints. |
| `matching.numTaskqueueReadPartitions` | 1 | `4` | Number of read partitions per task queue. Should match write partitions. Can be set per task queue via constraints. |
| `matching.outstandingTaskAppendsThreshold` | 250 | `500` | Buffer size for outstanding task appends before backpressure kicks in. Increase when seeing "Too many outstanding appends to the task queue". **Requires node restart to take effect.** |
| `matching.getTasksBatchSize` | 1000 | — | How many backlog tasks to read from persistence at once. |
| `matching.maxTaskQueueIdleTime` | 5m | — | Time after which an idle task queue will be unloaded. Should be greater than `longPollExpirationInterval`. |
| `matching.backlogNegligibleAge` | 5s | — | Threshold for negligible vs significant backlog age. If head of backlog is older than this, sync match and forwarding stop to ensure more equal dispatch order across partitions. |
| `matching.maxWaitForPollerBeforeFwd` | 200ms | — | In presence of a non-negligible backlog, resume forwarding tasks if no poll has been seen within this duration. |
| `matching.useNewMatcher` | false | — | Enable the new priority-enabled task matcher. Required for `priorityLevels` and `enableFairness` to take effect. |
| `matching.enableFairness` | false | `true` (per-TQ) | Enable fairness for task dispatching. Implies `useNewMatcher`. Can be set per task queue via constraints. |
| `matching.priorityLevels` | 5 | `10` | Number of simple priority levels (requires new matcher). Set to 10 for richer priority use cases. |
| `matching.longPollExpirationInterval` | 1m | — | Long poll expiration interval in the matching service. |
| `matching.syncMatchWaitDuration` | 200ms | — | Wait time for sync match before falling back to async. |
| `matching.updateAckInterval` | 1m | — | How frequently the `task_queues` table is updated. Increase to reduce DB write frequency for task queues. |

---

## System

| Config Key | Default | Example Value | Description / When to Use |
|---|---|---|---|
| `system.secondaryVisibilityWritingMode` | `"off"` | `"dual"` | Controls dual-write to standard and enhanced (ES) visibility. Set to `"dual"` when migrating to ES. Remove or set to `"off"` after migration is complete. |
| `system.enableReadFromSecondaryVisibility` | false | `false` | Whether to read from secondary visibility store. Enable after ES migration is stable. |
| `system.visibilityDisableOrderByClause` | true | — | Disable `ORDER BY` clause for Elasticsearch queries. |
| `system.forceSearchAttributesCacheRefreshOnRead` | false | — | Bypass search attribute cache. Useful right after adding a new search attribute. Available in server 1.20.3+. **Do not enable in production long-term.** |
| `system.namespaceCacheRefreshInterval` | 2s | — | Interval for namespace cache refresh. Tune if namespace config updates are slow to propagate. |
| `system.enableEagerWorkflowStart` | true | — | Skips the trip through matching by returning the first workflow task inline in the `StartWorkflowExecution` response. Enabled by default. |
| `system.enableActivityEagerExecution` | false | — | Enables eager activity execution per namespace — activities can be returned inline in the workflow task response without going through matching. |
| `system.enableStickyQuery` | true | — | Whether sticky query is enabled per namespace. |
| `system.operatorRPSRatio` | 0.2 | — | Percentage of rate limit reserved for operator API calls (highest priority). Value between 0 and 1. Default 20%. |
| `system.enableDataLossMetrics` | false | — | Emit metrics when data loss errors are encountered. Useful for detecting persistence issues. |
| `system.enableNexus` | true | — | Master toggle for all Nexus functionality on the server. **Requires restart to take effect.** |
| `system.transactionSizeLimit` | — | — | Largest allowed transaction size to persistence. |
| `system.enableCrossNamespaceCommands` | false | — | Disabled since server 1.30. Affects `SignalExternalWorkflow` across namespaces. Enable cautiously for OSS users needing cross-namespace signaling. |

---

## Limits

| Config Key | Default | Example Value | Description / When to Use |
|---|---|---|---|
| `limit.maxIDLength` | 1000 | — | Max length for task queue names, namespace names, workflow IDs, activity names, signal names, etc. (was 100 in older versions). |
| `limit.blobSize.error` | 2MB | — | Per-event blob size hard limit. Can be increased per-namespace via constraints. |
| `limit.blobSize.warn` | 512KB | — | Per-event blob size warning threshold. |
| `limit.memoSize.error` | 2MB | — | Per-event memo size hard limit. |
| `limit.memoSize.warn` | 2KB | — | Memo size warning threshold. |
| `limit.historySize.error` | 50MB | — | Per-workflow history size hard limit. |
| `limit.historySize.warn` | 10MB | — | Per-workflow history size warning threshold. |
| `limit.historySize.suggestContinueAsNew` | 4MB | — | History size at which the server suggests continue-as-new to the worker. Reduce to proactively prompt CAN before hitting hard limits. |
| `limit.historyCount.error` | 50K | — | Per-workflow history event count hard limit. |
| `limit.historyCount.warn` | 10K | — | History event count warning threshold. |
| `limit.historyCount.suggestContinueAsNew` | 4K | — | History event count at which the server suggests continue-as-new. Set to 2000 or lower to shorten schedule workflow histories. |
| `limit.numPendingChildExecutions.error` | 2000 | — | Max pending child workflows per workflow execution. |
| `limit.numPendingActivities.error` | 2000 | — | Max pending activities per workflow execution. |
| `limit.numPendingSignals.error` | 2000 | — | Max pending signals per workflow execution. |
| `limit.numPendingCancelRequests.error` | 2000 | — | Max pending cancel requests per workflow execution. |

---

## Worker

| Config Key | Default | Example Value | Description / When to Use |
|---|---|---|---|
| `worker.taskQueueScannerEnabled` | true | — | Enable/disable the internal task queue scanner workflow. Disable if it causes issues. |
| `worker.historyScannerEnabled` | true | — | Enable/disable history scanner. |
| `worker.historyScannerDataMinAge` | 60d | — | Minimum age of history data before scanner considers it for cleanup. Reduce to clean up stale history faster (e.g. set to `4d` for aggressive cleanup). |
| `worker.historyScannerVerifyRetention` | true | — | Whether scanner verifies data retention. If archival is enabled, set to double the data retention period. |
| `worker.executionsScannerEnabled` | false | — | Enable executions scanner. No effect when SQL persistence is used. |
| `worker.executionDataDurationBuffer` | 90d | — | TTL buffer for execution data. |
| `worker.persistenceMaxQPS` | 500 | — | Worker persistence max QPS per host. |
| `worker.scannerPersistenceMaxQPS` | 100 | — | Max persistence QPS specifically from the worker scanner. |
| `worker.ESProcessorNumOfWorkers` | 2 | — | Number of workers for the ES (Elasticsearch) processor. Increase to ~8 to help drain ES indexing backlog. **Requires history service pod restart.** |
| `worker.ESProcessorBulkActions` | 500 | — | Max number of requests in a single ES bulk operation. Do not set above 512. Decrease to reduce ES load. **Requires restart.** |
| `worker.ESProcessorBulkSize` | 16MB | — | Max total size of an ES bulk request in bytes. **Requires restart.** |
| `worker.ESProcessorFlushInterval` | 1s | — | How often the ES processor flushes its bulk buffer. **Requires restart.** |
| `worker.ESProcessorAckTimeout` | 30s | — | Timeout waiting for ack from ES processor. Should be at least `FlushInterval` + processing time. **Requires restart.** |
| `worker.stickyCacheSize` | 0 | — | Sticky workflow cache size for SDK workers on worker service nodes. Shared between all workers in the process. **Cannot be changed after startup.** |
| `worker.enableScheduler` | true | — | Toggle the schedule worker per namespace. Disable for namespaces that do not use schedules. |
| `worker.batcherRPS` | 50 | — | RPS limit for batch operations per namespace. |
| `worker.batcherConcurrency` | 5 | — | Number of concurrent workflow operations within a single batch job per namespace. |
| `worker.perNamespaceWorkerCount` | 1 | `4` | Number of per-namespace workers (scheduler, batcher, etc.) running per namespace. Set to number of worker pods. |
| `worker.perNamespaceWorkerOptions` | — | `MaxConcurrentWorkflowTaskPollers: 25, MaxConcurrentActivityTaskPollers: 10` | SDK worker options for per-namespace workers. Rule of thumb: 1 workflow task poller per 4 schedule RPS desired. |
| `worker.schedulerNamespaceStartWorkflowRPS` | 30 | `100` | RPS limit for starting workflows from schedules, per namespace as a whole. Increase for high-schedule-volume namespaces. |