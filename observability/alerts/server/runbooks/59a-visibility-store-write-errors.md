# Runbook: Visibility Store Write Errors

**Alerts:** `Temporal: Visibility Store Write Errors (Warning/Critical)`,
`Temporal: Visibility Store Write Latency High`

**Component:** History — visibility queue processor  
**Metrics:** `visibility_persistence_errors`, `visibility_persistence_latency`  
**Label:** `visibility_index_name` — `temporal_visibility` (primary) or `temporal_visibility_secondary` (secondary)

---

## What This Alert Means

History emits `visibility_persistence_errors` when a visibility write to either
store fails. The `visibility_index_name` label identifies which store is failing.
This is the only reliable way to distinguish primary from secondary failure —
server logs always show the same dual-write stack trace regardless of which store
is affected.

Workflows are never affected by visibility failures — they keep running and
completing normally. But **visibility tasks are not retried forever**:

- Retry gaps: 1s initial, growing 10% per attempt, capped at 3 minutes, each gap
  jittered to 80–100% of nominal.
- At `history.TaskDLQUnexpectedErrorAttempts` unexpected attempts — **70 by
  default, roughly 70 minutes** — the task is written to the history task dead
  letter queue and stops being retried. `history.TaskDLQEnabled` defaults to
  **true**, so this is on unless you turned it off.
- Whether the record comes back on its own depends on which task was dropped: a
  dropped start or update is rebuilt by the workflow's next visibility write, but
  a dropped close is the last task that workflow ever emits and nothing rebuilds
  it. **Alert 83** fires when that happens; the **Visibility Tasks Dead-Lettered by Task Type** panel breaks it down by task type — but on
  Elasticsearch, a dead-lettered task does **not** reliably mean the record is
  missing. The bulk processor buffers documents, so a write that timed out may
  still be indexed once Elasticsearch recovers, after the task has already been
  set aside. Check the store for the records before assuming loss. On SQL there is
  no buffer, so a dead-lettered task there did lose the write.

**This alert does not catch every visibility failure.**
`visibility_persistence_errors` excludes timeouts, rate-limit rejections,
not-found and invalid-argument errors — so an Elasticsearch write that is never
confirmed within `worker.ESProcessorAckTimeout` will **not** fire this alert. Also
note this alert filters `service_name="history"`, so read-path failures never
trigger it. **Alert 84** covers this case. Check the **Visibility Errors by Type per Store** panel whenever
visibility is suspected and this alert is quiet, querying the label value
`persistence_TimeoutError` — the short form `TimeoutError` matches nothing.

---

## Diagnosis

### Step 1 — Identify which store is failing

Check the **Visibility Write Error Rate per Store** panel in the Visibility
section of the server dashboard (**v2.15.0 or later**).

The `visibility_index_name` label values are whatever you configured — the SQL
database name, or the Elasticsearch index name — not fixed strings. Read them off
the **Visibility Write Request Rate per Store** panel while healthy so you recognise them here.

- Primary store's line spiking, secondary clean → **primary failure**
- Secondary store's line spiking, primary clean → **secondary failure**
- Both spiking simultaneously → **both stores failed** (most severe)
- **Both flat at zero** while **Visibility Task Failures & Internal Errors** shows `task_errors_internal` rising →
  not an outage at all: `system.secondaryVisibilityWritingMode` holds an invalid
  value and every write is being rejected before it reaches either store. Note
  2129 counts attempts, so it falls back to zero once those tasks are
  dead-lettered even though writes stay broken — a flat 2129 is not an all-clear.
  The durable signal is 2117 flat on both stores plus new workflows never
  appearing in either.

### Step 2 — Check pod status

```bash
kubectl get pods -n temporal | grep visibility
# Expected healthy:
# temporal-stack-postgresql-visibility-0            1/1  Running
# temporal-stack-postgresql-visibility-secondary-0  1/1  Running
```

### Step 3 — Confirm with logs

```bash
# History errors (both stores look identical in logs — use metrics to distinguish)
kubectl logs -n temporal -l app.kubernetes.io/component=history --tail=50 | \
  grep -E '"msg":"(Critical error|sql handle)"'

# Primary failure: frontend will also log read errors
kubectl logs -n temporal -l app.kubernetes.io/component=frontend --since=5m | \
  grep "ListWorkflowExecutions operation failed"
```

### Step 4 — Confirm workflow list impact

```bash
temporal workflow list -n default
# Works → secondary is failing, primary is up
# context deadline exceeded → primary is down (or both)
```

---

## Recovery

### Primary store down

Restore primary first — this unblocks all list/describe/count operations.

```bash
kubectl scale statefulset temporal-stack-postgresql-visibility -n temporal --replicas=1
kubectl wait pod/temporal-stack-postgresql-visibility-0 -n temporal \
  --for=condition=Ready --timeout=5m
```

Wait 30-60 seconds for the SQL connection pool to refresh. Errors stop
automatically — no history pod restart required.

### Secondary store down

```bash
kubectl scale statefulset temporal-stack-postgresql-visibility-secondary -n temporal --replicas=1
kubectl wait pod/temporal-stack-postgresql-visibility-secondary-0 -n temporal \
  --for=condition=Ready --timeout=5m
```

Secondary failure does not affect `temporal workflow list` or workflow execution.
Lower urgency than primary failure.

### Both stores down

Restore primary first, then secondary:

```bash
kubectl scale statefulset temporal-stack-postgresql-visibility -n temporal --replicas=1
kubectl wait pod/temporal-stack-postgresql-visibility-0 -n temporal --for=condition=Ready --timeout=5m

kubectl scale statefulset temporal-stack-postgresql-visibility-secondary -n temporal --replicas=1
kubectl wait pod/temporal-stack-postgresql-visibility-secondary-0 -n temporal --for=condition=Ready --timeout=5m
```

---

## Emergency: Disable Secondary Writes

If the secondary store requires extended maintenance, disable dual writes to stop
retry noise immediately. No history restart required — takes effect within one
dynconfig poll interval (~10s).

```bash
# Disable secondary writes
kubectl get configmap temporal-dynconfig -n temporal -o json | \
  jq '.data["system.secondaryVisibilityWritingMode"] = "off"' | \
  kubectl apply -f -

# Verify
kubectl get configmap temporal-dynconfig -n temporal -o jsonpath='{.data}' | \
  jq 'with_entries(select(.key | startswith("system.secondaryVisibility")))'

# Re-enable after recovery
kubectl get configmap temporal-dynconfig -n temporal -o json | \
  jq '.data["system.secondaryVisibilityWritingMode"] = "dual"' | \
  kubectl apply -f -
```

**Warning:** disabling secondary writes abandons any pending retry tasks for the
secondary. Workflows that ran during the outage will be missing from the secondary
store until manually backfilled.

---

## Post-Recovery Verification

```bash
# Error rate should return to zero
# Check Grafana: Visibility Write Error Rate per Store

# Confirm both stores have records (counts should converge within ~5 minutes)
kubectl run pg-check-primary --rm -it --restart=Never --image=postgres:16-alpine -- \
  psql "postgres://temporal:temporal@temporal-stack-postgresql-visibility/temporal_visibility" \
  -c "SELECT count(*) FROM executions_visibility;"

kubectl run pg-check-secondary --rm -it --restart=Never --image=postgres:16-alpine -- \
  psql "postgres://temporal:temporal@temporal-stack-postgresql-visibility-secondary/temporal_visibility" \
  -c "SELECT count(*) FROM executions_visibility;"
```

For the full operational runbook including write mode management and all failure
scenarios: [`playbooks/dual-visibility.md`](../../../../playbooks/dual-visibility.md)
