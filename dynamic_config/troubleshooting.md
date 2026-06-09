# Temporal Dynamic Config — Troubleshooting Patterns

## Table of Contents

- [RESOURCE_EXHAUSTED errors](#resource_exhausted-errors)
- [History cache misses / memory pressure](#history-cache-misses--memory-pressure)
- [ES / Visibility indexing backlog](#es--visibility-indexing-backlog)
- [Too many outstanding task appends](#too-many-outstanding-task-appends)
- [Idle cluster hitting DB too often](#idle-cluster-hitting-db-too-often)
- [History growing too large](#history-growing-too-large)
- [Schedules too slow / rate limited / resource issues](#schedules-too-slow--rate-limited--resource-issues)
- [approximate_backlog_count showing stale / stuck value](#approximate_backlog_count-showing-stale--stuck-value)

---

## Common Troubleshooting Patterns

### RESOURCE_EXHAUSTED errors
- `RpsLimit` cause → adjust `frontend.rps`, `frontend.namespaceRPS`, `frontend.globalNamespaceRPS`
- `ConcurrentLimit` cause → adjust `frontend.namespaceCount`, `frontend.globalNamespaceCount`
- Visibility `list*` operations → adjust `frontend.namespaceRPS.visibility` (default only 10!)
- Persistence overloaded → adjust `frontend.persistenceMaxQPS`, `history.persistenceMaxQPS`, `matching.persistenceMaxQPS`

### History cache misses / memory pressure
- Check `history.hostLevelCacheMaxSize` (entry count, default 128K) or `history.hostLevelCacheMaxSizeBytes` (~1GB, when `cacheSizeBasedLimit=true`)
- Enable `history.cacheBackgroundEvict` for proactive cleanup
- Tune `history.cacheTTL` / `history.eventsCacheTTL` (default 1h)

### ES / Visibility indexing backlog
- Reduce `history.visibilityProcessorMaxPollRPS` and `history.visibilityProcessorSchedulerWorkerCount` to throttle indexing speed
- Reduce `history.visibilityProcessorSchedulerWorkerCount` to `0` to fully pause indexing during reindex (set via dynamic config, **requires restart**)
- Set `system.secondaryVisibilityWritingMode: dual` when migrating to ES

### Too many outstanding task appends
- Increase `matching.outstandingTaskAppendsThreshold` (restart required)
- Consider increasing `matching.numTaskqueueWritePartitions` / `matching.numTaskqueueReadPartitions`

### Idle cluster hitting DB too often
- Increase `history.transferProcessorMaxPollInterval`, `history.timerProcessorMaxPollInterval`, `history.visibilityProcessorMaxPollInterval`, `history.shardUpdateMinInterval`

### History growing too large
- Reduce `limit.historySize.suggestContinueAsNew` and `limit.historyCount.suggestContinueAsNew` to encourage continue-as-new earlier

### Schedules too slow / rate limited / resource issues

These three configs **always go together** when tuning schedules:

- `worker.schedulerNamespaceStartWorkflowRPS` — the RPS limit is **per namespace as a whole** (divided evenly among workers if more than one). Default 30. Increase as needed, up to 400–500 is fine. If `schedule_rate_limited` metric is non-zero, you're being throttled here.
- `worker.perNamespaceWorkerCount` — default is 1, meaning **all schedule work for a namespace runs on a single pod**. This is a very common cause of disproportionate CPU/memory on one worker pod. Set to your number of worker pods (or slightly higher).
- `worker.perNamespaceWorkerOptions` — set `MaxConcurrentWorkflowTaskPollers` to **1 per 4 "schedule RPS"** you want. Going higher doesn't hurt. Also set `MaxConcurrentActivityTaskPollers`.

**Example config for ~100 schedules RPS:**
```yaml
worker.schedulerNamespaceStartWorkflowRPS:
  - value: 100
worker.perNamespaceWorkerOptions:
  - value:
      MaxConcurrentWorkflowTaskPollers: 25
      MaxConcurrentActivityTaskPollers: 10
worker.perNamespaceWorkerCount:
  - value: 3
```

**Example config for ~400 schedules RPS:**
```yaml
worker.schedulerNamespaceStartWorkflowRPS:
  - value: 400
worker.perNamespaceWorkerOptions:
  - value:
      MaxConcurrentWorkflowTaskPollers: 100
      MaxConcurrentActivityTaskPollers: 10
worker.perNamespaceWorkerCount:
  - value: 4
```

> **Note:** `worker.perNamespaceWorkerOptions` is a map-valued setting. `DescribeSchedule` slowness can also be a symptom — it relies on `QueryWorkflow` on the scheduler workflow, so check `schedule_to_start` latency on your server worker if queries are slow.

---

### `approximate_backlog_count` showing stale / stuck value

**Symptom:** The `approximate_backlog_count` gauge for a task queue shows a non-zero value in Grafana (e.g. 500) long after the backlog was resolved — hours or days later. The value never changes or drops to zero.

**Root cause:** When a task queue partition unloads (due to idle timeout or explicit unload), the matching service emits a zero value for the gauge to clear it in Prometheus. This zero-on-unload is gated on both `metrics.breakdownByTaskQueue` and `metrics.breakdownByPartition` being `true`. If either flag is `false`, the zero is never emitted. Prometheus keeps the last scraped value indefinitely (until series retention expires), and Grafana draws a flat line at the stale value.

**How to confirm:**
1. Run `temporal task-queue describe --task-queue <name> --namespace <ns>` — if it reports 0 backlog, the gauge is stale and the actual queue is healthy.
2. Check your dynamic config for `metrics.breakdownByTaskQueue` and `metrics.breakdownByPartition` — if either is set to `false`, that is the cause.

**Fix:**
- Do not disable `metrics.breakdownByTaskQueue` or `metrics.breakdownByPartition` globally. Both default to `true` and should stay that way. These flags are the gate for all per-task-queue backlog gauges (`approximate_backlog_count`, `approximate_backlog_age_seconds`, `physical_approximate_backlog_count`) and for the zero-on-unload cleanup that prevents stale values.
- If you disabled them to reduce label cardinality (too many task queues in Prometheus), use the `constraints` field to scope the disable to specific high-cardinality namespaces or task queues rather than turning them off globally.
- As an immediate remediation for a customer seeing a stuck value: restart the matching pods. This forces all task queue partitions to unload and re-emit zeros on shutdown, clearing the stale series.

**What not to change:** These flags have no meaningful operational tuning surface. Do not adjust them unless you have a confirmed Prometheus cardinality problem. The correct default for any production cluster is `true` for both.

> **Pre-1.31.0 warning:** On server versions before 1.31.0, `approximate_backlog_count` can get permanently stuck at a non-zero value after a matching service restart even when no real backlog exists. This is a known bug in the classic matcher fixed in 1.31.0 (PR #9731). On affected versions, do not alert on this metric — use `schedule_to_start` latency (SDK metrics) and `sum(rate(persistence_requests{operation="CreateTasks",namespace="$namespace"}[$__rate_interval]))` (the **Tasks Persisted to DB** panel in the server dashboard) as the reliable backlog signals instead. Upgrade to 1.31.0 to get the fix.