# Temporal On-Prem Server Dynamic Config Reference

---

## Table of Contents

- [Baseline: Configs to Consider for Every Cluster](#baseline-configs-to-consider-for-every-cluster)
    - [Frontend](#frontend)
    - [History](#history)
    - [Matching](#matching)
    - [Worker](#worker)
- [Dynamic Configs for Different Limits](#dynamic-configs-for-different-limits)
- [All Available Configs](#all-available-configs)
    - [Frontend](#frontend-1)
    - [History](#history-1)
    - [Matching](#matching-1)
    - [System](#system-1)
    - [Limits](#limits)
    - [Worker](#worker-1)
- [Configs That Require a Host Restart](#configs-that-require-a-host-restart)
- [Preparing for Migration to Temporal Cloud](#preparing-for-migration-to-temporal-cloud)

---

## Baseline: Configs to Consider for Every Cluster

These are the configs you should review and consciously set (or decide to leave at default) for any production cluster.

### Frontend

| Config Key | Default | Description / When to Use |
|---|---|---|
| `frontend.rps` | 2400 | Overall RPS limit per frontend pod. One of the 3 core configs to adjust when hitting `service_errors_resource_exhausted`. |
| `frontend.globalRPS` | 0 | RPS limit for the whole frontend cluster. |
| `frontend.namespaceRPS` | 2400 | Per-namespace RPS limit per frontend instance. Adjust when getting "namespace rate limit exceeded". |
| `frontend.globalNamespaceRPS` | 0 | Per-namespace RPS distributed evenly across the whole frontend cluster. Use with server 1.18.0+. Overwrites per-instance limit when set. |
| `frontend.namespaceCount` | 1200 | Concurrent poller limit per namespace per frontend host. Increase when seeing `RESOURCE_EXHAUSTED: namespace count limit exceeded` or "namespace concurrent poller limit exceeded". |
| `frontend.globalNamespaceCount` | 0 | Global concurrent long-running requests limit per namespace across all frontend instances. |
| `frontend.persistenceMaxQPS` | 2000 | Frontend persistence max QPS per host. Increase when hitting "Persistence Max QPS Reached". |
| `frontend.persistenceGlobalMaxQPS` | 0 | Frontend persistence max QPS for the whole frontend cluster. |
| `frontend.namespaceRPS.visibility` | 10 | Per-namespace RPS limit for visibility APIs. **Default is very low — very commonly needs increasing.** Increase when getting `RESOURCE_EXHAUSTED: namespace rate limit exceeded` on `list*` operations. |
| `frontend.globalNamespaceRPS.visibility` | 0 | Cluster-wide per-namespace RPS for visibility APIs. Alternative to per-instance limit. |
| `frontend.WorkerCommandsEnabled` | false | Allows clients to send commands to workers. |
| `frontend.visibilityArchivalQueryMaxPageSize` | 10000 | Max page size for archival queries. Reduce when UI archival queries cause issues (no restart required). |
| `frontend.MaxConcurrentBatchOperationPerNamespace` | 1 | Max concurrent batch operation jobs per namespace. Increase to allow parallel batch jobs. |

### History

| Config Key | Default | Description / When to Use |
|---|---|---|
| `history.rps` | 3000 | RPS limit per history pod. |
| `history.cacheSizeBasedLimit` | false | If true, limits history cache by bytes instead of entry count. |
| `history.hostLevelCacheMaxSizeBytes` | ~1GB | Max bytes for host-level cache. Used when `cacheSizeBasedLimit=true`. Tune down (e.g. 100MB) if history pods are memory-constrained. |
| `history.cacheMaxSizeBytes` | — | Per-shard cache size in bytes. Used alongside `cacheSizeBasedLimit`. |
| `history.persistenceMaxQPS` | 9000 | History persistence max QPS per host. Increase when hitting persistence max QPS errors. |
| `history.persistenceGlobalMaxQPS` | 0 | History persistence max QPS for whole cluster. |
| `history.persistencePerShardNamespaceMaxQPS` | 0 | Per-shard per-namespace persistence max QPS. Useful to cap noisy-neighbor namespaces. |
| `history.shardIOConcurrency` | 1 | Concurrency of persistence operations per shard. For SQL persistence, increase to reduce shard lock contention. **Requires restart.** |

### Matching

| Config Key | Default | Description / When to Use |
|---|---|---|
| `matching.rps` | 1200 | RPS limit per matching pod. |
| `matching.persistenceMaxQPS` | 3000 | Matching persistence max QPS per host. Increase when hitting persistence max QPS errors. |
| `matching.persistenceGlobalMaxQPS` | 0 | Matching persistence max QPS for the whole matching cluster. |
| `matching.numTaskqueueWritePartitions` | 4 | Number of write partitions per task queue. Default 4 for all user task queues (internal system queues default to 1). **Set to 1 for low-throughput queues to save resources.** Can be set per task queue via constraints. |
| `matching.numTaskqueueReadPartitions` | 4 | Number of read partitions per task queue. Should match write partitions. Default 4. Can be set per task queue via constraints. |
| `matching.outstandingTaskAppendsThreshold` | 250 | Buffer size for outstanding task appends before backpressure kicks in. Increase when seeing "Too many outstanding appends to the task queue". **Requires node restart to take effect.** |
| `metrics.breakdownByTaskQueue` | true | Controls whether matching and history metrics include the actual task queue name as a label tag (`taskqueue`). When `false`, the tag value is replaced with `__omitted__`. **Disabling this disables all per-task-queue gauges including `approximate_backlog_count`, `approximate_backlog_age_seconds`, and `physical_approximate_backlog_count`.** Disable only if high task queue cardinality is overwhelming your observability stack. Can be scoped per task queue via constraints. |
| `metrics.breakdownByPartition` | true | Controls whether matching metrics include the actual partition ID as a label tag (`partition`). When `false`, normal partitions are reported as `__normal__`; sticky queues always report as `__sticky__` regardless. **Disabling this also disables all per-task-queue backlog gauges** (same gate as `metrics.breakdownByTaskQueue` — both must be `true` for backlog metrics to emit). Disable only if partition cardinality is too high. Can be scoped per task queue via constraints. |

### Worker

| Config Key | Default | Description / When to Use |
|---|---|---|
| `worker.persistenceMaxQPS` | 500 | Worker persistence max QPS per host. |
| `worker.persistenceGlobalMaxQPS` | 0 | Worker persistence max QPS for the whole worker cluster. |
| `worker.schedulerNamespaceStartWorkflowRPS` | 30 | RPS limit for starting workflows from schedules, per namespace as a whole. Increase for high-schedule-volume namespaces. |
| `worker.batcherRPS` | 50 | RPS limit for batch operations per namespace. |
| `worker.batcherConcurrency` | 5 | Number of concurrent workflow operations within a single batch job per namespace. |
| `worker.perNamespaceWorkerCount` | 1 | Number of per-namespace workers (scheduler, batcher, etc.) running per namespace. Set to number of worker pods. |
| `worker.perNamespaceWorkerOptions` | — | SDK worker options for per-namespace workers (e.g. `MaxConcurrentWorkflowTaskPollers`, `MaxConcurrentActivityTaskPollers`). Rule of thumb: 1 workflow task poller per 4 schedule RPS desired. |

---

# Dynamic Configs for Different Limits

## Signals

| Config Key | Description | Default |
|---|---|---|
| `limit.numPendingSignals.error` | Max pending signals before `SignalExternalWorkflowExecution` commands fail | 2000 |
| `history.maximumSignalsPerExecution` | Max total signals ever received by a single execution | 10000 |

---

## Activities

| Config Key | Description | Default |
|---|---|---|
| `limit.numPendingActivities.error` | Max pending activities before `ScheduleActivityTask` fails | 2000 |
| `limit.mutableStateActivityFailureSize.error` | Max per-activity failure size stored in mutable state | 4KB |
| `limit.mutableStateActivityFailureSize.warn` | Warning threshold for per-activity failure size | 2KB |

---

## Child Workflows

| Config Key | Description | Default |
|---|---|---|
| `limit.numPendingChildExecutions.error` | Max pending child workflows before `StartChildWorkflowExecution` commands fail | 2000 |

---

## Cancel Requests

| Config Key | Description | Default |
|---|---|---|
| `limit.numPendingCancelRequests.error` | Max pending cancel requests to other workflows before `RequestCancelExternalWorkflowExecution` commands fail | 2000 |

---

## Updates

| Config Key | Description | Default |
|---|---|---|
| `history.maxInFlightUpdates` | Max in-flight (admitted but not completed) updates at once | 10 |
| `history.maxInFlightUpdatePayloads` | Max total payload size of in-flight updates in bytes | 20MB |
| `history.maxTotalUpdates` | Max total updates a workflow can ever receive | 2000 |
| `history.maxTotalUpdates.suggestContinueAsNewThreshold` | % of total updates limit before continue-as-new is suggested | 0.9 (90%) |

---

## History Size / Event Count

| Config Key | Description | Default |
|---|---|---|
| `limit.historySize.error` | Max history size in bytes before failing | 50MB |
| `limit.historySize.warn` | Warning threshold for history size | 10MB |
| `limit.historySize.suggestContinueAsNew` | History size at which continue-as-new is suggested | 4MB |
| `limit.historyCount.error` | Max history event count before failing | 51,200 (50×1024) |
| `limit.historyCount.warn` | Warning threshold for history event count | 10,240 (10×1024) |
| `limit.historyCount.suggestContinueAsNew` | Event count at which continue-as-new is suggested | 4,096 (4×1024) |

---

## Mutable State

| Config Key | Description | Default |
|---|---|---|
| `limit.mutableStateSize.error` | Max mutable state size in bytes before failing | 8MB |
| `limit.mutableStateSize.warn` | Warning threshold for mutable state size | 1MB |
| `limit.mutableStateTombstoneCountLimit` | Max deleted sub state machines tracked in mutable state | 16 |

---

## Payload / Blob

| Config Key | Description | Default |
|---|---|---|
| `limit.blobSize.error` | Max per-event blob size | 2MB |
| `limit.blobSize.warn` | Warning threshold for per-event blob size | 512KB |
| `limit.memoSize.error` | Max memo size | 2MB |
| `limit.memoSize.warn` | Warning threshold for memo size | 2KB |

---

## Buffered Events

| Config Key | Description | Default |
|---|---|---|
| `history.maximumBufferedEventsBatch` | Max number of buffered events in mutable state | 100 |
| `history.maximumBufferedEventsSizeInBytes` | Max total size of all buffered events | 2MB |

---

## Miscellaneous

| Config Key | Description | Default |
|---|---|---|
| `history.MaxBufferedQueryCount` | Max buffered queries | 1 |
| `history.historyMaxAutoResetPoints` | Max auto reset points stored in mutable state | primitives constant |
| `history.workflowTaskHeartbeatTimeout` | Workflow task heartbeat timeout | 30min |


## All Available Configs

## Frontend

| Config Key | Default | Description / When to Use |
|---|---|---|
| `frontend.rps` | 2400 | Overall RPS limit per frontend pod. One of the 3 core configs to adjust when hitting `service_errors_resource_exhausted`. |
| `frontend.globalRPS` | 0 | RPS limit for the whole frontend cluster. |
| `frontend.namespaceRPS` | 2400 | Per-namespace RPS limit per frontend instance. Adjust when getting "namespace rate limit exceeded". |
| `frontend.globalNamespaceRPS` | 0 | Per-namespace RPS distributed evenly across the whole frontend cluster. Use with server 1.18.0+. Overwrites per-instance limit when set. |
| `frontend.namespaceCount` | 1200 | Concurrent poller limit per namespace per frontend host. Increase when seeing `RESOURCE_EXHAUSTED: namespace count limit exceeded` or "namespace concurrent poller limit exceeded". |
| `frontend.globalNamespaceCount` | 0 | Global concurrent long-running requests limit per namespace across all frontend instances. |
| `frontend.persistenceMaxQPS` | 2000 | Frontend persistence max QPS per host. Increase when hitting "Persistence Max QPS Reached". |
| `frontend.persistenceGlobalMaxQPS` | 0 | Frontend persistence max QPS for the whole frontend cluster. |
| `frontend.persistenceNamespaceMaxQPS` | 0 | Per-namespace persistence max QPS per frontend host. |
| `frontend.persistenceGlobalNamespaceMaxQPS` | 0 | Per-namespace persistence max QPS for the whole frontend cluster. |
| `frontend.namespaceRPS.visibility` | 10 | Per-namespace RPS limit for visibility APIs. **Default is very low — very commonly needs increasing.** Increase when getting `RESOURCE_EXHAUSTED: namespace rate limit exceeded` on `list*` operations. |
| `frontend.globalNamespaceRPS.visibility` | 0 | Cluster-wide per-namespace RPS for visibility APIs. Alternative to per-instance limit. |
| `frontend.namespaceBurstRatio.visibility` | 1 | Burst ratio for visibility namespace RPS. Must be >= 1. |
| `frontend.WorkerCommandsEnabled` | false | Allows clients to send commands to workers. |
| `frontend.visibilityMaxPageSize` | 1000 | Max page size for `ListWorkflowExecutions`. Increase if needed. |
| `frontend.historyMaxPageSize` | — | Max page size for `GetWorkflowExecutionHistory`. Increase via dynamic config when needed. |
| `frontend.visibilityArchivalQueryMaxPageSize` | 10000 | Max page size for archival queries. Reduce when UI archival queries cause issues (no restart required). |
| `frontend.searchAttributesNumberOfKeysLimit` | 100 | Limits the number of search attributes that can be set on a workflow. |
| `frontend.searchAttributesSizeOfValueLimit` | 2KB | Size limit per search attribute value. Can be set per-namespace via constraints. |
| `frontend.searchAttributesTotalSizeLimit` | 40KB | Total size limit of all search attributes map. |
| `frontend.MaxConcurrentBatchOperationPerNamespace` | 1 | Max concurrent batch operation jobs per namespace. Increase to allow parallel batch jobs. |
| `frontend.namespaceBurstRatio` | 2 | Burst ratio for regular namespace RPS (not just visibility). Must be >= 1. The RPS used is the effective RPS from global and per-instance limits. |
| `frontend.shutdownDrainDuration` | 0s | Duration of traffic drain during frontend shutdown. Set to allow in-flight requests to complete before pod terminates. |
| `frontend.shutdownFailHealthCheckDuration` | 0s | Duration to fail health checks before shutdown begins, allowing load balancers to drain connections. |
| `frontend.enableUpdateWorkflowExecution` | true | Feature flag to enable the `UpdateWorkflowExecution` API. |
| `frontend.enableSchedules` | true | Enables schedule-related RPCs in the frontend. |
| `frontend.WorkflowPauseEnabled` | false | Feature flag to allow clients to pause workflows. |
| `frontend.enableBatcher` | true | Feature flag to enable batch operation RPCs in the frontend. |
| `frontend.maxBadBinaries` | 10 | Max number of bad binaries in namespace config. |
| `frontend.maskInternalErrorDetails` | true | Whether to mask internal error details in responses to clients. |

---

## History

| Config Key | Default | Description / When to Use |
|---|---|---|
| `history.rps` | 3000 | RPS limit per history pod. |
| `history.namespaceRPS` | 0 | Per-namespace RPS limit per history pod. Falls back to `history.rps` when 0. |
| `history.enableNamespaceFairness` | false | Enable per-namespace fair-share demotion in history host RPS limiter. Requests from namespaces over their fair share are routed to a lower-priority bucket. |
| `history.namespaceFairShareMultiplier` | 1.0 | Scales the per-namespace fair share. `share(ns) = scaleFactor * FrontendGlobalNamespaceRPS(ns) * multiplier`. Used with `enableNamespaceFairness`. |
| `history.persistenceMaxQPS` | 9000 | History persistence max QPS per host. Increase when hitting persistence max QPS errors. |
| `history.persistenceGlobalMaxQPS` | 0 | History persistence max QPS for whole cluster. |
| `history.persistenceNamespaceMaxQPS` | 0 | Per-namespace persistence max QPS per history host. Falls back to `history.persistenceMaxQPS` when 0. |
| `history.persistenceGlobalNamespaceMaxQPS` | 0 | Per-namespace persistence max QPS for the whole history cluster. |
| `history.persistencePerShardNamespaceMaxQPS` | 0 | Per-shard per-namespace persistence max QPS. Useful to cap noisy-neighbor namespaces. |
| `history.cacheMaxSizeBytes` | — | Per-shard mutable state cache size in bytes. Used when `cacheSizeBasedLimit=true`. |
| `history.hostLevelCacheMaxSize` | 128000 | Size of **host-level** mutable state cache (entries). Controls memory usage of history service when `cacheSizeBasedLimit=false`. Adjust based on available memory. |
| `history.hostLevelCacheMaxSizeBytes` | ~1GB | Max bytes for host-level mutable state cache. Used when `cacheSizeBasedLimit=true`. ~100MB in example. |
| `history.cacheSizeBasedLimit` | false | If true, limits history cache by bytes instead of entry count. |
| `history.cacheTTL` | 1h | TTL for history cache entries. Entry is evicted if not accessed for this duration. Usually no need to change. |
| `history.cacheBackgroundEvict` | disabled | Background eviction of expired cache entries. Enable with `Enabled: true, LoopInterval: 1m, MaxEntryPerCall: 1024`. |
| `history.cacheNonUserContextLockTimeout` | 500ms | How long non-user calls wait for workflow lock. Increase when seeing `ResourceExhausted on BusyWorkflow`. **Requires service restart.** |
| `history.eventsCacheMaxSizeBytes` | 512KB | Max size of the **per-shard** events cache in bytes. Increase if events are frequently fetched from persistence. **Requires restart.** |
| `history.eventsHostLevelCacheMaxSizeBytes` | ~256MB | Max size of the **host-level** events cache in bytes. Used when `history.enableHostLevelEventsCache=true`. **Requires restart.** |
| `history.enableHostLevelEventsCache` | false | Enable host-level events cache instead of per-shard. **Requires restart.** |
| `history.eventsCacheTTL` | 1h | TTL of events cache. |
| `history.shardIOConcurrency` | 1 | Concurrency of persistence operations per shard. For SQL persistence, increase to reduce shard lock contention. **Requires restart.** |
| `history.shardUpdateMinInterval` | 5m | Min interval between shard info updates. Increase to reduce background DB writes on idle clusters. |
| `history.transferProcessorMaxPollInterval` | 1m | How often TransferTaskProcessor reads from DB on an idle cluster. Increase to reduce idle DB reads. |
| `history.transferProcessorMaxPollHostRPS` | 0 | Max poll rate for all transfer processors on a host. Tune to limit DB load during recovery. |
| `history.transferProcessorSchedulerWorkerCount` | 512 | Workers in host-level task scheduler for transfer processor. Increase when transfer processing is a bottleneck. **Requires restart.** |
| `history.timerProcessorMaxPollInterval` | 5m | How often TimerTaskProcessor reads from DB on idle cluster. |
| `history.timerProcessorMaxPollHostRPS` | 0 | Max poll rate for all timer processors on a host. |
| `history.timerProcessorSchedulerWorkerCount` | 512 | Workers for timer task scheduler. Increase under load. **Requires restart.** |
| `history.visibilityProcessorMaxPollInterval` | 1m | How often VisibilityTaskProcessor reads from DB on idle cluster. |
| `history.visibilityProcessorMaxPollRPS` | 20 | Max poll rate for visibility queue processor per host. **Reduce** to throttle ES indexing during backlog drain (e.g. after AWS outage). |
| `history.visibilityProcessorMaxPollHostRPS` | 0 | Host-level max poll rate for visibility processor. Tune to limit ES load. |
| `history.visibilityProcessorSchedulerWorkerCount` | 512 | Workers for visibility task scheduler. Reduce (e.g. to 128) to slow down ES indexing during recovery. **Requires restart.** |
| `history.workflowTaskCriticalAttempt` | 10 | Number of consecutive workflow task attempts before flagging as critical. |
| `history.workflowTaskRetryMaxInterval` | 10m | Maximum interval added to workflow task `startToClose` timeout for retry backoff. |
| `history.maxTotalUpdates` | 2000 | Max number of updates a workflow execution can receive. Set to 0 to disable. |
| `history.maxTotalUpdates.suggestContinueAsNewThreshold` | 0.9 | Percentage of `maxTotalUpdates` before suggesting continue-as-new. |
| `history.archivalProcessorArchiveDelay` | 5m | Delay before archival queue processor starts processing tasks. |
| `history.archivalTaskBatchSize` | 100 | Batch size for archival queue processor. Reduce to slow down archival under load. |
| `history.archivalProcessorMaxPollHostRPS` | 0 | Host-level max poll rate for archival processor. Reduce to limit archival I/O. |
| `history.maximumSignalsPerExecution` | 10000 | Max number of signals a single workflow execution can receive. |
| `history.maximumBufferedEventsBatch` | 100 | Max number of buffered events for any given mutable state. |
| `history.maxInFlightUpdates` | 10 | Max number of in-flight updates (admitted but not yet completed) for a workflow execution. Set to 0 to disable. |
| `history.maxInFlightUpdatePayloads` | 20MB | Max total payload size of in-flight updates for a workflow execution. Set to 0 to disable. |
| `history.defaultActivityRetryPolicy` | — | Default retry policy for activities where user has not specified one. Configurable per namespace. |
| `history.defaultWorkflowRetryPolicy` | — | Default retry policy for workflows where user has set a retry policy but not all fields. Configurable per namespace. |
| `history.enableWorkflowIdReuseStartTimeValidation` | false | If true, validates old workflow start time is older than `workflowIdReuseMinimalInterval` before allowing ID reuse. Can cause throttling with `BUSY_WORKFLOW` cause if enabled. |
| `history.workflowIdReuseMinimalInterval` | 1s | Minimum interval between workflow starts with same ID. Used when `enableWorkflowIdReuseStartTimeValidation` is true. |

---

## Matching

| Config Key | Default | Description / When to Use |
|---|---|---|
| `matching.rps` | 1200 | RPS limit per matching pod. |
| `matching.persistenceMaxQPS` | 3000 | Matching persistence max QPS per host. Increase when hitting persistence max QPS errors. |
| `matching.persistenceGlobalMaxQPS` | 0 | Matching persistence max QPS for the whole matching cluster. |
| `matching.persistenceNamespaceMaxQPS` | 0 | Per-namespace persistence max QPS per matching host. |
| `matching.persistenceGlobalNamespaceMaxQPS` | 0 | Per-namespace persistence max QPS for the whole matching cluster. |
| `matching.numTaskqueueWritePartitions` | 4 | Number of write partitions per task queue. Default 4 for all user task queues (internal system queues default to 1). **Set to 1 for low-throughput queues to save resources.** Can be set per task queue via constraints. |
| `matching.numTaskqueueReadPartitions` | 4 | Number of read partitions per task queue. Should match write partitions. Default 4. Can be set per task queue via constraints. |
| `matching.outstandingTaskAppendsThreshold` | 250 | Buffer size for outstanding task appends before backpressure kicks in. Increase when seeing "Too many outstanding appends to the task queue". **Requires node restart to take effect.** |
| `matching.getTasksBatchSize` | 1000 | How many backlog tasks to read from persistence at once. |
| `matching.maxTaskQueueIdleTime` | 5m | Time after which an idle task queue will be unloaded. Should be greater than `longPollExpirationInterval`. |
| `matching.backlogNegligibleAge` | 5s | Threshold for negligible vs significant backlog age. If head of backlog is older than this, sync match and forwarding stop to ensure more equal dispatch order across partitions. |
| `matching.maxWaitForPollerBeforeFwd` | 200ms | In presence of a non-negligible backlog, resume forwarding tasks if no poll has been seen within this duration. |
| `matching.useNewMatcher` | false | Enable the new priority-enabled task matcher. Required for `priorityLevels` and `enableFairness` to take effect. |
| `matching.longPollExpirationInterval` | 1m | Long poll expiration interval in the matching service. |
| `matching.syncMatchWaitDuration` | 200ms | Wait time for sync match before falling back to async. |
| `matching.updateAckInterval` | 1m | How frequently the `task_queues` table is updated. Increase to reduce DB write frequency for task queues. |
| `metrics.breakdownByTaskQueue` | true | Controls whether matching and history metrics include the actual task queue name as a label tag (`taskqueue`). When `false`, the tag value is replaced with `__omitted__`. **Disabling this disables all per-task-queue gauges including `approximate_backlog_count`, `approximate_backlog_age_seconds`, and `physical_approximate_backlog_count`.** Additionally, when a task queue unloads, the zero-value emission that clears stale gauge values in Prometheus is skipped — meaning old values can persist in dashboards until Prometheus series retention expires. Disable only if high task queue cardinality is overwhelming your observability stack. Can be scoped per task queue via constraints. |
| `metrics.breakdownByPartition` | true | Controls whether matching metrics include the actual partition ID as a label tag (`partition`). When `false`, normal partitions are reported as `__normal__`; sticky queues always report as `__sticky__` regardless. **Both `metrics.breakdownByTaskQueue` and `metrics.breakdownByPartition` must be `true` for per-task-queue backlog gauges to emit, and for zero-value cleanup to fire on task queue unload.** Disable only if partition cardinality is too high. Can be scoped per task queue via constraints. |
| `metrics.breakdownByBuildID` | true | Controls whether matching metrics include the actual worker deployment version as a label tag (`worker_version`). When `false`, versioned queues are reported as `__versioned__`; unversioned queues always report as `__unversioned__` regardless. Disabling this disables per-task-queue backlog gauges for versioned queues only. Can be scoped per task queue via constraints. |

---

## System

| Config Key | Default | Description / When to Use |
|---|---|---|
| `system.visibilityPersistenceMaxReadQPS` | 9000 | Max QPS for visibility DB read operations (system-wide). Tune if visibility reads are causing DB overload. |
| `system.visibilityPersistenceMaxWriteQPS` | 9000 | Max QPS for visibility DB write operations (system-wide). Tune to limit indexing pressure. |
| `system.secondaryVisibilityWritingMode` | `"off"` | Controls dual-write to a secondary visibility store. Set to `"dual"` during ES migration to write to both stores simultaneously. Set to `"on"` to write only to secondary. |
| `system.enableReadFromSecondaryVisibility` | false | Whether to read from the secondary visibility store. Enable after dual-write is stable and secondary is caught up. Per-namespace via constraints. |
| `system.visibilityDisableOrderByClause` | true | Disable `ORDER BY` clause for Elasticsearch queries. |
| `system.forceSearchAttributesCacheRefreshOnRead` | false | Bypass search attribute cache. Useful right after adding a new search attribute. Available in server 1.20.3+. **Do not enable in production long-term.** |
| `system.namespaceCacheRefreshInterval` | 2s | Interval for namespace cache refresh. Tune if namespace config updates are slow to propagate. |
| `system.enableEagerWorkflowStart` | true | Skips the trip through matching by returning the first workflow task inline in the `StartWorkflowExecution` response. Enabled by default. |
| `system.enableActivityEagerExecution` | false | Enables eager activity execution per namespace — activities can be dispatched directly back to the completing worker inline in the `RespondWorkflowTaskCompleted` response, bypassing matching entirely. The SDK requests eager execution by default but the server ignores it unless this flag is `true`. **Set to `true` on all production clusters** — this is enabled on Temporal Cloud and is a significant latency and throughput optimization. |
| `system.enableStickyQuery` | true | Whether sticky query is enabled per namespace. |
| `system.operatorRPSRatio` | 0.2 | Percentage of rate limit reserved for operator API calls (highest priority). Value between 0 and 1. Default 20%. |
| `system.enableDataLossMetrics` | false | Emit metrics when data loss errors are encountered. Useful for detecting persistence issues. |
| `system.enableNexus` | true | Master toggle for all Nexus functionality on the server. **Requires restart to take effect.** |
| `system.transactionSizeLimit` | — | Largest allowed transaction size to persistence. |
| `system.enableCrossNamespaceCommands` | false | Disabled since server 1.30. Affects `SignalExternalWorkflow` across namespaces. Enable cautiously for OSS users needing cross-namespace signaling. |

---

## Limits

| Config Key | Default | Description / When to Use |
|---|---|---|
| `limit.maxIDLength` | 1000 | Max length for task queue names, namespace names, workflow IDs, activity names, signal names, etc. (was 100 in older versions). |
| `limit.blobSize.error` | 2MB | Per-event blob size hard limit. Can be increased per-namespace via constraints. |
| `limit.blobSize.warn` | 512KB | Per-event blob size warning threshold. |
| `limit.memoSize.error` | 2MB | Per-event memo size hard limit. |
| `limit.memoSize.warn` | 2KB | Memo size warning threshold. |
| `limit.historySize.error` | 50MB | Per-workflow history size hard limit. |
| `limit.historySize.warn` | 10MB | Per-workflow history size warning threshold. |
| `limit.historySize.suggestContinueAsNew` | 4MB | History size at which the server suggests continue-as-new to the worker. Reduce to proactively prompt CAN before hitting hard limits. |
| `limit.historyCount.error` | 51,200 | Per-workflow history event count hard limit. |
| `limit.historyCount.warn` | 10,240 | History event count warning threshold. |
| `limit.historyCount.suggestContinueAsNew` | 4,096 | History event count at which the server suggests continue-as-new. Set to 2000 or lower to shorten schedule workflow histories. |
| `limit.numPendingChildExecutions.error` | 2000 | Max pending child workflows per workflow execution. |
| `limit.numPendingActivities.error` | 2000 | Max pending activities per workflow execution. |
| `limit.numPendingSignals.error` | 2000 | Max pending signals per workflow execution. |
| `limit.numPendingCancelRequests.error` | 2000 | Max pending cancel requests per workflow execution. |

---

## Worker

| Config Key | Default | Description / When to Use |
|---|---|---|
| `worker.taskQueueScannerEnabled` | true | Enable/disable the internal task queue scanner workflow. Disable if it causes issues. |
| `worker.historyScannerEnabled` | true | Enable/disable history scanner. |
| `worker.historyScannerDataMinAge` | 60d | Minimum age of history data before scanner considers it for cleanup. Reduce to clean up stale history faster (e.g. set to `4d` for aggressive cleanup). |
| `worker.historyScannerVerifyRetention` | true | Whether scanner verifies data retention. If archival is enabled, set to double the data retention period. |
| `worker.executionsScannerEnabled` | false | Enable executions scanner. No effect when SQL persistence is used. |
| `worker.executionDataDurationBuffer` | 90d | TTL buffer for execution data. |
| `worker.persistenceMaxQPS` | 500 | Worker persistence max QPS per host. |
| `worker.persistenceGlobalMaxQPS` | 0 | Worker persistence max QPS for the whole worker cluster. |
| `worker.persistenceNamespaceMaxQPS` | 0 | Per-namespace persistence max QPS per worker host. |
| `worker.persistenceGlobalNamespaceMaxQPS` | 0 | Per-namespace persistence max QPS for the whole worker cluster. |
| `worker.scannerPersistenceMaxQPS` | 100 | Max persistence QPS specifically from the worker scanner. |
| `worker.ESProcessorNumOfWorkers` | 2 | Number of workers for the ES (Elasticsearch) processor. Increase to ~8 to help drain ES indexing backlog. **Requires history service pod restart.** |
| `worker.ESProcessorBulkActions` | 500 | Max number of requests in a single ES bulk operation. Do not set above 512. Decrease to reduce ES load. **Requires restart.** |
| `worker.ESProcessorBulkSize` | 16MB | Max total size of an ES bulk request in bytes. **Requires restart.** |
| `worker.ESProcessorFlushInterval` | 1s | How often the ES processor flushes its bulk buffer. **Requires restart.** |
| `worker.ESProcessorAckTimeout` | 30s | Timeout waiting for ack from ES processor. Should be at least `FlushInterval` + processing time. **Requires restart.** |
| `worker.stickyCacheSize` | 0 | Sticky workflow cache size for SDK workers on worker service nodes. Shared between all workers in the process. **Cannot be changed after startup.** |
| `worker.enableScheduler` | true | Toggle the schedule worker per namespace. Disable for namespaces that do not use schedules. |
| `worker.batcherRPS` | 50 | RPS limit for batch operations per namespace. |
| `worker.batcherConcurrency` | 5 | Number of concurrent workflow operations within a single batch job per namespace. |
| `worker.perNamespaceWorkerCount` | 1 | Number of per-namespace workers (scheduler, batcher, etc.) running per namespace. Set to number of worker pods. |
| `worker.perNamespaceWorkerOptions` | — | SDK worker options for per-namespace workers (e.g. `MaxConcurrentWorkflowTaskPollers`, `MaxConcurrentActivityTaskPollers`). Rule of thumb: 1 workflow task poller per 4 schedule RPS desired. |
| `worker.schedulerNamespaceStartWorkflowRPS` | 30 | RPS limit for starting workflows from schedules, per namespace as a whole. Increase for high-schedule-volume namespaces. |

---

## Configs That Require a Host Restart

Most dynamic config changes take effect without any restart — the server polls for config changes on a short interval. The settings below are exceptions: they are read once at process startup and cached for the lifetime of the process. Changing them in dynamic config has no effect until the relevant hosts are restarted.

**Only the service whose hosts use the setting needs to be restarted.** Changing a `history.*` setting only requires rolling the history hosts; matching hosts and frontend hosts are unaffected, and vice versa. There is no need to restart the entire cluster.

### History hosts

| Config Key | Notes |
|---|---|
| `history.cacheSizeBasedLimit` | Switches the history cache between entry-count and byte-size limiting modes. |
| `history.cacheTTL` | TTL for the history workflow execution cache. |
| `history.cacheNonUserContextLockTimeout` | Timeout for acquiring workflow lock from non-user context. |
| `history.hostLevelCacheMaxSize` | Max entry count for the host-level history cache. |
| `history.hostLevelCacheMaxSizeBytes` | Max byte size for the host-level history cache (used when `cacheSizeBasedLimit=true`). |
| `history.cacheBackgroundEvict` | Enables background goroutine for proactive cache eviction. Must be enabled at startup to activate the goroutine. |
| `history.eventsCacheMaxSizeBytes` | Max byte size for the shard-level events cache. |
| `history.eventsHostLevelCacheMaxSizeBytes` | Max byte size for the host-level events cache. |
| `history.eventsCacheTTL` | TTL for the events cache. |
| `history.enableHostLevelEventsCache` | Toggles host-level events cache on or off. |
| `history.clientOwnershipCachingEnabled` | Enables caching of shard ownership in history clients. Only read when a history client is first created. |
| `history.businessIDReuseLimiterCacheSize` | Cache size for the per-(namespace, businessID, archetype) start rate limiter. |
| `history.businessIDReuseLimiterCacheTTL` | TTL for the business ID reuse rate limiter cache. |
| `history.visibilityProcessorSchedulerWorkerCount` | Number of workers in the host-level task scheduler for the visibility queue processor. |

### Matching hosts

| Config Key | Notes |
|---|---|
| `matching.workerRegistryNumBuckets` | Number of buckets partitioning the worker registry keyspace for reduced lock contention. |
| `matching.outstandingTaskAppendsThreshold` | Threshold for outstanding task appends before backpressure is applied. |

### System (all hosts)

| Config Key | Notes |
|---|---|
| `system.nexusReadThroughCacheSize` | Max entry count for the Nexus read-through cache. Requires restart on all hosts using the Nexus endpoint cache. |
| `system.nexusReadThroughCacheTTL` | TTL for the Nexus read-through cache. Requires restart on all hosts using the Nexus endpoint cache. |

### Worker hosts

| Config Key | Notes |
|---|---|
| `worker.stickyCacheSize` | Size of the sticky workflow execution cache shared across all SDK workers in the process. Cannot be changed after startup. |

---

## Preparing for Migration to Temporal Cloud

Temporal Cloud enforces fixed limits on several dimensions that are configurable on self-hosted Temporal. This section maps Cloud limits to the on-prem dynamic configs that correspond to them, so you can verify your cluster is operating within Cloud's bounds before migrating.

### Configs that already match Cloud defaults

If these settings are at their defaults and your workloads operate within those defaults, no change is needed for migration.

Most configs in this table are fixed on Cloud and cannot be changed even via support ticket. OSS users planning to migrate should not raise these values above their defaults — doing so might cause your workloads to break on Cloud.

| On-prem config | Default | Cloud limit                                                                                                                                   |
|---|---|-----------------------------------------------------------------------------------------------------------------------------------------------|
| `limit.blobSize.error` | 2 MB | 2 MB per payload                                                                                                                              |
| `system.transactionSizeLimit` | 4 MB | 4 MB event history transaction                                                                                                                |
| `limit.maxIDLength` | 1000 | 1,000 bytes for Workflow ID, Task Queue name, etc. — **exception: namespace names are capped at 39 characters on Cloud**, versus 1,000 on OSS |
| `limit.numPendingActivities.error` | 2000 | 2,000 pending activities per Workflow Execution                                                                                               |
| `limit.numPendingSignals.error` | 2000 | 2,000 pending signals per Workflow Execution                                                                                                  |
| `limit.numPendingChildExecutions.error` | 2000 | 2,000 pending child workflows per Workflow Execution                                                                                          |
| `limit.numPendingCancelRequests.error` | 2000 | 2,000 pending cancel requests per Workflow Execution                                                                                          |
| `history.maximumSignalsPerExecution` | 10,000 | 10,000 total signals per Workflow Execution                                                                                                   |
| `history.maxInFlightUpdates` | 10 | 10 in-flight updates per Workflow Execution                                                                                                   |
| `history.maxTotalUpdates` | 2000 | 2,000 total updates per Workflow Execution                                                                                                    |
| `limit.historySize.error` | 50 MB | 50 MB Event History size                                                                                                                      |
| `limit.historySize.warn` | 10 MB | 10 MB Event History warn threshold                                                                                                            |
| `limit.historyCount.error` | 51,200 | 51,200 events (50×1024)                                                                                                                       |
| `limit.historyCount.warn` | 10,240 | 10,240 events (10×1024)                                                                                                                       |
| `frontend.MaxConcurrentBatchOperationPerNamespace` | 1 | 1 batch job running at a time per namespace                                                                                                   |
| `worker.batcherRPS` | 50 | 50 Workflow Executions per second per batch job                                                                                               |
| `matching.maxDeployments` | 100 | 100 Worker Deployments per namespace                                                                                                          |
| `matching.maxVersionsInDeployment` | 100 | 100 versions per Worker Deployment                                                                                                            |
| `matching.maxTaskQueuesInDeploymentVersion` | 100 | 100 Task Queues per Worker Deployment Version                                                                                                 |
| `limit.historySize.suggestContinueAsNew` | 4 MB | 4 MB — fixed on Cloud, configurable on OSS                                                                                                    |
| `limit.historyCount.suggestContinueAsNew` | 4,096 | 4,096 events — fixed on Cloud, configurable on OSS                                                                                            |

### Configs where on-prem default is more permissive than Cloud

These settings allow higher throughput or larger values on-prem than Cloud's default limits. The Cloud values listed below are starting defaults — they can be increased by opening a support ticket with Temporal. If your workloads rely on the higher on-prem headroom, they may hit Cloud's defaults after migration until a limit increase is approved.

| On-prem config | On-prem default | Cloud limit | What to check |
|---|---|---|---|
| `frontend.namespaceRPS` | 2400 | 2,000 RPS (auto-scales up from there) | Namespaces sustaining close to 2,400 namespace RPS will hit Cloud's 2,000 floor until auto-scaling raises it. Monitor `service_errors_resource_exhausted` after migration. The 2,000 starting value can be increased via support ticket. |
| `worker.schedulerNamespaceStartWorkflowRPS` | 30 | 10 schedule RPS | Cloud's schedule RPS cap is 3× lower than the on-prem default and does not auto-scale. Namespaces with >10 schedule starts/s will be throttled. Spread schedule start times using SDK jitter, or request a limit increase before migrating. |
| `frontend.namespaceRPS.visibility` | 10 (but commonly raised) | 30 visibility RPS | If you have raised this above 30 on-prem, visibility `list*` calls will be throttled on Cloud. Cloud's cap (30) is higher than the on-prem default, so clusters that have not raised it are fine. The 30 RPS limit on Cloud is effectively fixed — increasing it is not supported. |

### Configs where on-prem is more restrictive than Cloud

On-prem defaults are lower than Cloud limits here. Migrating gives workloads more headroom — no action needed, but worth knowing.

| On-prem config | On-prem default | Cloud limit |
|---|---|---|
| `frontend.namespaceCount` | 1,200 concurrent long-running requests per-instance, per-API (long-polls, queries) | 20,000 Activity pollers + 20,000 Workflow Task pollers per namespace (default; can be increased via support ticket) |
| `system.maxCallbacksPerWorkflow` | 32 total callbacks | 2,000 total callbacks per Workflow Execution |
| `system.enableActivityEagerExecution` | `false` | Enabled on Cloud — activities can be dispatched directly back to the completing worker inline in the `RespondWorkflowTaskCompleted` response, bypassing matching entirely. **OSS users should set this to `true`** — the SDK requests eager execution by default and the server silently ignores it if this flag is off. On-prem clusters without this enabled are missing a significant latency and throughput optimization. |

