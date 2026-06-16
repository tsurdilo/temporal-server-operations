# Dual Visibility Operations Runbook

Operational reference for self-hosted Temporal clusters running dual visibility
(`system.secondaryVisibilityWritingMode = dual`). Covers failure detection,
diagnosis, recovery, and write-mode management for all three failure scenarios.

> **Scope:** This runbook covers **SQL-backed dual visibility only** (both stores
> are PostgreSQL). It has not been validated against Elasticsearch-backed
> configurations — ES coverage will be added in a future iteration.

---

## References

**Dashboard:** [Temporal Server Dashboard](../metrics/dashboards/server/temporal-server-readme.md) — v2.5.0+, Visibility group

| Panel ID | Panel Name | What to look for |
|----------|------------|------------------|
| 2117 | Visibility Write Request Rate per Store | Flat line on one store = writes stopped (store down or dynconfig changed) |
| 2118 | Visibility Write Error Rate per Store | **Primary alert signal.** Which store's line is spiking identifies the failure |
| 2119 | Visibility Write Latency per Store | Divergence between stores = one recovering or under pressure |

**Alerts:** [`metrics/alerts/server/temporal-server-alerts.yaml`](../metrics/alerts/server/temporal-server-alerts.yaml)

| Alert UID | Alert Name | Fires when |
|-----------|------------|------------|
| `temporal-alert-059a` | Visibility Store Write Errors (Warning) | `visibility_persistence_errors` > 0.1 req/s for 2m on either store |
| `temporal-alert-059b` | Visibility Store Write Errors (Critical) | `visibility_persistence_errors` > 1 req/s for 1m on either store |
| `temporal-alert-059c` | Visibility Store Write Latency High | p99 `visibility_persistence_latency` > 3s for 5m on either store |

Alert runbook: [`metrics/alerts/server/runbooks/59a-visibility-store-write-errors.md`](../metrics/alerts/server/runbooks/59a-visibility-store-write-errors.md)

**Related dynamic config keys:**

| Key | Values | Effect |
|-----|--------|--------|
| `system.secondaryVisibilityWritingMode` | `off` / `on` / `dual` | Controls whether secondary store receives writes |

---

## Background

**Dual visibility is a last-resort migration tool, not an HA mechanism.** The
right approach is to plan your visibility store choice upfront and set it up
correctly from the start. Dual visibility exists for cases where that wasn't
possible — migrating a running cluster from one store to another without
downtime. It does not improve availability or reliability: a primary store
failure still degrades reads, there is no automatic failover to secondary, and
secondary receives no read traffic under any circumstances. For read HA, use
Elasticsearch — the ES cluster handles shard-level failover transparently
without any Temporal-level routing logic.

Dual visibility writes every visibility event (workflow open/close/update) to two
PostgreSQL stores in parallel via `dualWriteWrapper`
(`visibility_manager_dual.go:241`). Reads go to the primary only (`dualReadWrapper`,
`visibility_manager_dual.go:276`). History retries failed visibility tasks
indefinitely with exponential backoff (1s initial, 1.1× coefficient, 3-minute
cap, no retry limit) — workflows are never affected and no data is lost as long
as at least one store is reachable.

**The only metric family that identifies which store failed:**
`visibility_persistence_requests`, `visibility_persistence_errors`,
`visibility_persistence_latency` — all carry the `visibility_index_name` label
(`temporal_visibility` for primary, `temporal_visibility_secondary` for secondary).

**Log-only diagnosis is insufficient.** Logs always show `dualWriteWrapper` /
`visibility_manager_dual.go:241` in the stack trace regardless of which store
failed — the label in the metric is the authoritative signal.

---

## Metrics Reference

| Metric | Label(s) | Notes |
|--------|----------|-------|
| `visibility_persistence_requests` | `visibility_index_name`, `operation` | No `namespace` label — do not filter by namespace |
| `visibility_persistence_errors` | `visibility_index_name`, `operation` | Primary alert signal for store failure |
| `visibility_persistence_latency` | `visibility_index_name`, `operation`, `le` | Latency divergence = one store struggling |

---

## Scenario 1 — Primary Store Failure

### Symptom Pattern

- **Panel 2118 (Error Rate)**: `temporal_visibility` line spikes; `temporal_visibility_secondary` stays at zero.
- **Panel 2119 (Latency)**: both lines rise as task backlog grows.
- **`temporal workflow list`**: degrades once primary connection pool exhausts (~30-60s after pod goes down).
- **History logs** (`component: visibility-queue-processor`):
  ```json
  {"msg":"sql handle: unable to refresh database connection pool",
   "error":"dial tcp <primary-ip>:5432: connect: connection refused"}
  {"msg":"Critical error processing task, retrying.",
   "error":"no usable database connection found",
   "error-type":"serviceerror.Unavailable",
   "attempt":N,
   "task-category":"visibility",
   "stacktrace":"...dualWriteWrapper...visibility_manager_dual.go:241"}
  ```
  Attempt counter increments every retry. At the 3-minute backoff cap,
  `unexpected-error-attempts` equals `attempt - 1`.

### Impact

- Workflow execution: **unaffected**
- `temporal workflow list` / `describe`: **degrades** — reads go to primary only
- Visibility task queue: builds up; no data loss — tasks retry until store recovers
- Temporal Web UI workflow list: will return errors or show incomplete results

### Recovery

1. Check primary pod:
   ```bash
   kubectl get pods -n temporal | grep "visibility-0"
   ```
2. Scale up if down:
   ```bash
   kubectl scale statefulset temporal-stack-postgresql-visibility -n temporal --replicas=1
   kubectl wait pod/temporal-stack-postgresql-visibility-0 -n temporal \
     --for=condition=Ready --timeout=5m
   ```
3. Wait 30-60s for connection pool refresh — no server restart required.
4. Verify panel 2118 error rate returns to zero.
5. Verify panel 2119 latency drains back to baseline (may take several minutes).

### Emergency: pause secondary writes to reduce pressure during recovery

```bash
kubectl get configmap temporal-dynconfig -n temporal -o json | \
  jq '.data["system.secondaryVisibilityWritingMode"] = "off"' | \
  kubectl apply -f -
```

Revert immediately after primary recovers:
```bash
kubectl get configmap temporal-dynconfig -n temporal -o json | \
  jq '.data["system.secondaryVisibilityWritingMode"] = "dual"' | \
  kubectl apply -f -
```

---

## Scenario 2 — Secondary Store Failure

### Symptom Pattern

- **Panel 2118 (Error Rate)**: `temporal_visibility_secondary` line spikes; `temporal_visibility` stays at zero.
- **`temporal workflow list`**: **fully functional** throughout — reads from primary only.
- **History logs**: identical pattern to Scenario 1 (`dualWriteWrapper` in stack). Only panel 2118 label distinguishes this from Scenario 1.

### Impact

- Workflow execution: **unaffected**
- `temporal workflow list` / `describe`: **fully functional**
- Visibility task queue: builds up for secondary writes; no data loss
- Lower severity than Scenario 1 — self-healing once store recovers

### Recovery

```bash
kubectl scale statefulset temporal-stack-postgresql-visibility-secondary -n temporal --replicas=1
kubectl wait pod/temporal-stack-postgresql-visibility-secondary-0 -n temporal \
  --for=condition=Ready --timeout=5m
```

Wait 30-60s for pool refresh. Verify panel 2118 secondary error line returns to zero.

### Emergency: disable secondary writes for extended maintenance

```bash
kubectl get configmap temporal-dynconfig -n temporal -o json | \
  jq '.data["system.secondaryVisibilityWritingMode"] = "off"' | \
  kubectl apply -f -
```

After disabling, history stops retrying secondary tasks on the next config poll
(~10s). **Note:** workflows that ran during the outage will be missing from the
secondary store. Backfill or reconciliation required if secondary is later
promoted to primary.

---

## Scenario 3 — Both Stores Fail

### Symptom Pattern

- **Panel 2118 (Error Rate)**: **both** `temporal_visibility` and `temporal_visibility_secondary` lines spike simultaneously.
- **`temporal workflow list`**: **fails with `context deadline exceeded`** — reads go to primary only, so with primary down all list/describe operations return errors.
- **Frontend logs**:
  ```json
  {"msg":"Operation failed with an error.",
   "error":"ListWorkflowExecutions operation failed.: dial tcp <primary-ip>:5432: connect: connection refused",
   "stacktrace":"...dualReadWrapper...visibility_manager_dual.go:276...frontend/workflow_handler.go:2733"}
  ```
- **History logs**: same `Critical error processing task, retrying.` pattern on both stores.

### Impact

- Workflow execution: **unaffected** — workflows continue running normally
- `temporal workflow list` / `describe` / `count`: **fully broken**
- Temporal Web UI: workflow list empty or returning errors
- Visibility task queue: full backlog accumulation across all history shards

### Recovery

Restore primary first — this immediately unblocks list operations.

```bash
# 1. Restore primary
kubectl scale statefulset temporal-stack-postgresql-visibility -n temporal --replicas=1
kubectl wait pod/temporal-stack-postgresql-visibility-0 -n temporal \
  --for=condition=Ready --timeout=5m

# 2. Verify list works (30-60s after pod Ready)
temporal workflow list -n default

# 3. Restore secondary
kubectl scale statefulset temporal-stack-postgresql-visibility-secondary -n temporal --replicas=1
kubectl wait pod/temporal-stack-postgresql-visibility-secondary-0 -n temporal \
  --for=condition=Ready --timeout=5m
```

Verify both error lines on panel 2118 return to zero. Monitor panel 2119 until
backlog drains.

---

## Write Mode Management

The `system.secondaryVisibilityWritingMode` dynconfig key controls dual visibility
at runtime — no server restart required. Changes take effect within one poll
interval (~10s).

| Value | Behavior |
|-------|----------|
| `"off"` | Single visibility — writes to primary only |
| `"on"` | Writes to primary; secondary receives writes but errors are suppressed (not retried) |
| `"dual"` | Writes to both stores; errors on either store are retried indefinitely |

Inspect current value:
```bash
kubectl get configmap temporal-dynconfig -n temporal -o jsonpath='{.data}' | \
  jq 'with_entries(select(.key | startswith("system.secondaryVisibility")))'
```

Update:
```bash
kubectl get configmap temporal-dynconfig -n temporal -o json | \
  jq '.data["system.secondaryVisibilityWritingMode"] = "dual"' | \
  kubectl apply -f -
```

**Warning:** switching from `dual` to `off` while a failure is active drops the
retry queue immediately. Pending visibility tasks for the failed store are
abandoned — those records will be missing from that store permanently unless
backfilled.

---

## Connection Pool Behavior

After a visibility store pod recovers, history does NOT reconnect immediately.
The SQL connection pool has a throttled reconnect interval visible in logs as:

```
"sql handle: did not refresh database connection pool because the last refresh was too close"
```

Effective reconnect window: **30-60 seconds** from pod Ready before errors stop.
During this window panel 2118 will still show errors despite the pod showing
`Running/Ready`. This is normal — no server restart required.

---

## Post-Outage Verification

Verify both stores have converged after any recovery:

```bash
# Record count in primary
kubectl run pg-check-primary --rm -it --restart=Never --image=postgres:16-alpine -- \
  psql "postgres://temporal:temporal@temporal-stack-postgresql-visibility/temporal_visibility" \
  -c "SELECT count(*) FROM executions_visibility;"

# Record count in secondary
kubectl run pg-check-secondary --rm -it --restart=Never --image=postgres:16-alpine -- \
  psql "postgres://temporal:temporal@temporal-stack-postgresql-visibility-secondary/temporal_visibility" \
  -c "SELECT count(*) FROM executions_visibility;"
```

A difference in counts is expected during the backlog-drain window. If counts
still diverge after panel 2118 errors have cleared, allow up to 5 minutes for
the retry queue to fully drain.
