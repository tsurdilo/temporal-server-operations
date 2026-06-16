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
server logs always show `dualWriteWrapper` in the stack trace regardless of which
store is affected.

History retries failed visibility tasks indefinitely:
- Backoff: 1s initial, 1.1× coefficient, 3-minute cap
- No retry limit — tasks queue forever until the store recovers
- Workflows are never affected by visibility failures

---

## Diagnosis

### Step 1 — Identify which store is failing

Check the Grafana panel **Visibility Write Error Rate per Store** (Visibility group).

- `temporal_visibility` spiking, `temporal_visibility_secondary` clean → **primary failure**
- `temporal_visibility_secondary` spiking, `temporal_visibility` clean → **secondary failure**
- Both spiking simultaneously → **both stores failed** (most severe)

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
