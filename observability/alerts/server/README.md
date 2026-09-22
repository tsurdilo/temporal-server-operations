# Temporal Server — Grafana Alerts

Grafana alerting provisioning rules for a self-hosted Temporal Server cluster.

## Writing a new alert — required condition pattern

Use **query → reduce → threshold**. Do not use `classic_conditions`.

```yaml
data:
  - refId: A          # the Prometheus query, grouped: sum(...) by (namespace)
  - refId: B          # type: reduce,    reducer: last,  expression: A
  - refId: C          # type: threshold, expression: B,  conditions: [ evaluator ]
condition: C
```

`classic_conditions` collapses every series into one boolean. Two things follow from that, and both
were live defects in this set until 2026-09-18:

- **Labels are dropped**, so `{{ $labels.namespace }}` in an annotation renders `<no value>`.
- **You get one alert instance no matter how many series matched**, so an alert grouped
  `by (visibility_index_name)` could not tell you which store had failed.

In annotations, reference the reduce step, not `$value`:

```
{{ $values.B.Value | humanize }}        correct
{{ $value | humanize }}                 renders %!f(string=) — $value is a string, not a number
```

**Expect one alert instance per series.** That is the point — you learn which pod, operation or
store is affected. It does mean a rule grouped by operation can produce several instances at once;
group them in the notification policy rather than removing the `by (...)` clause.

### Guard every `histogram_quantile` alert with `> 0`

`histogram_quantile` returns `NaN` for a series with no observations in the window — an operation
that simply had no traffic. **Grafana's threshold step treats `NaN` as breaching**, so without a
guard the alert fires on absent data.

```promql
histogram_quantile(0.99, sum by (operation, le) (rate(..._bucket[5m]))) > 0
```

`NaN` comparisons are false in PromQL, so the guard drops those series before the threshold ever
sees them. Series that genuinely measured `0` are dropped too, which is harmless — `0` never
breaches a greater-than threshold.

This applies to any alert whose condition is **greater-than** and whose query is a
`histogram_quantile`. It was a live defect in this set until 2026-09-18: two latency alerts were
firing continuously on operations that had no traffic, while the one operation with a real
measurement sat in `Normal`.

> **Do not try to fix this on the reduce step.** Setting `mode: replaceNN` with
> `replaceWithValue: 0` looks like the answer and does nothing — the Prometheus datasource drops
> `NaN` points when it builds the frame, so the series arrives empty and there is no value left to
> replace. Reducing an empty series produces `NaN` again. The guard has to be in the query.

Alerts whose condition is **less-than** (the "dropped to zero" alerts) must **not** get this guard —
it would remove the very series they exist to catch. None of them use `histogram_quantile`, so the
two rules do not collide.

---

## Alert Files

| File | When to use |
|---|---|
| [`temporal-server-alerts.yaml`](./temporal-server-alerts.yaml) | All deployments — 26 implemented alerts covering core server health, persistence, shard queues, visibility (write path, read path and dual-visibility data loss), and pollers |
| [`temporal-failover-alerts.yaml`](./temporal-failover-alerts.yaml) | Multi-cluster replication only — 9 implemented alerts for graceful handover pre-flight, drain, and post-flip health. Drop alongside the core file if you run global namespaces with active-standby replication. Single-cluster deployments can skip it. |

See [alerts-index.md](./alerts-index.md) for the full planned inventory and design decisions.

---

## Setup

### 1. Prerequisites

- Temporal Server v1.20+ emitting Prometheus metrics
- Grafana 9.0+ with a Prometheus datasource named **`Prometheus`**
- [temporal-server.json](../../dashboards/server/temporal-server.json) dashboard imported — dashboard panel links in alert notifications will not resolve without it
- *(Failover alerts only)** [namespace-failover-graceful-handover.json](../../dashboards/server/namespace-failover-graceful-handover.json) dashboard imported

### 2. Drop the file(s) into Grafana provisioning

Copy the relevant file(s) into your Grafana provisioning directory:

```
<grafana-root>/provisioning/alerting/temporal-server-alerts.yaml

# Multi-cluster only:
<grafana-root>/provisioning/alerting/temporal-failover-alerts.yaml
```

Restart Grafana (or wait for the hot-reload interval). Core alerts appear under **Alerting → Alert rules → Temporal Server**. Failover alerts appear under **Temporal Failover**.

### 3. Set the datasource UID

Grafana requires the datasource **UID** (not name) in provisioned alert rules. The source file uses `Prometheus` as a placeholder — you must substitute your actual UID before deploying.

**Step 1 — look up your UID:**

```bash
curl -s http://admin:admin@localhost:3000/api/datasources \
  | jq '.[] | {name, uid, type}'
```

Find the entry for your Prometheus datasource and copy its `uid` value (e.g. `P7AC1CE4626B85CB6`).

**Step 2 — copy with UID substituted:**

```bash
GRAFANA_DS_UID="<your-uid-here>"
PROVISIONING_DIR="/path/to/grafana/provisioning/alerting"

sed "s/datasourceUid: Prometheus/datasourceUid: ${GRAFANA_DS_UID}/g" \
  temporal-server-alerts.yaml \
  > "${PROVISIONING_DIR}/temporal-server-alerts.yaml"

# Multi-cluster only:
sed "s/datasourceUid: Prometheus/datasourceUid: ${GRAFANA_DS_UID}/g" \
  temporal-failover-alerts.yaml \
  > "${PROVISIONING_DIR}/temporal-failover-alerts.yaml"
```

The UID is stable for the lifetime of the Grafana instance. Re-run this command only if you wipe Grafana's database and recreate the datasource from scratch.

### 4. Configure notification policies

The alerts ship with labels but no notification policy. Wire them up in **Alerting → Notification policies** based on your routing needs. All Essential Set alerts carry:

```
severity: critical
service: temporal
component: <frontend | history | persistence | server | matching>
```

---

## Essential Alert Set

| # | Alert | Component | Panel | `for` |
|---|---|---|---|---|
| 1 | [Total RPS Drops to Zero](./runbooks/01-total-rps-zero.md) | frontend | [Total RPS](../../dashboards/server/temporal-server-readme.md) | 5m |
| 1b | [Frontend Metrics Absent](./runbooks/01b-frontend-metrics-absent.md) | frontend | [Total RPS](../../dashboards/server/temporal-server-readme.md) | 2m |
| 4 | [Namespace RPS Drops to Zero](./runbooks/04-namespace-rps-zero.md) | frontend | [RPS per Namespace](../../dashboards/server/temporal-server-readme.md) | 5m |
| 8 | [Shard Lock Latency Critical](./runbooks/08-shard-lock-latency-critical.md) | history | [Shard Lock Latency](../../dashboards/server/temporal-server-readme.md) | 5m |
| 12 | [Persistence Latency Critical](./runbooks/12-persistence-latency-critical.md) | persistence | [Persistence Latencies](../../dashboards/server/temporal-server-readme.md) | 5m |
| 25 | [Service Panic Detected](./runbooks/25-service-panic-detected.md) | server | [Service Panics](../../dashboards/server/temporal-server-readme.md) | 1m |
| 27 | [Service Error Rate Critical](./runbooks/27-service-error-rate-critical.md) | frontend | [Service Errors by Namespace](../../dashboards/server/temporal-server-readme.md) | 2m |
| 30 | [System Overload Throttling](./runbooks/30-system-overload-throttling.md) | persistence | [Resource Exhausted with Cause](../../dashboards/server/temporal-server-readme.md) | 1m |
| 34 | [Unexpected Shard Movement](./runbooks/34-unexpected-shard-movement.md) | history | [Shards Created](../../dashboards/server/temporal-server-readme.md) | 10m |
| 34b | [Immediate Queue Lag Critical](./runbooks/34b-immediate-queue-lag-critical.md) | history | [Immediate Queue Lag per Pod](../../dashboards/server/temporal-server-readme.md) | 15m |
| 34f | [Shard Deadlock Detected](./runbooks/34f-shard-deadlock-detected.md) | history | [Suspected Deadlocks per Pod](../../dashboards/server/temporal-server-readme.md) | 1m |
| 34j | [Shard IO Semaphore Deadlock Approaching](./runbooks/34j-shard-io-semaphore-deadlock-approaching.md) | history | [Shard IO Concurrency Dashboard](../../dashboards/server/shard-io-concurrency-readme.md) | 5m |
| 38 | [Timer Task Scheduling Lag Critical](./runbooks/38-timer-scheduling-lag-critical.md) | history | [Timer Task Scheduling Latency](../../dashboards/server/temporal-server-readme.md) | 5m |
| 78 | [Shard Fleet Deficit](./runbooks/78-shard-fleet-deficit.md) | history | [Owned Shards (Total)](../../dashboards/server/temporal-server-readme.md) | 15m |
| 79 | [Shard Ownership Loss Persisting](./runbooks/79-shard-ownership-loss-persisting.md) | history | [Persistence Errors Total by Operation](../../dashboards/server/temporal-server-readme.md) | 10m |
| 57 | [All Pollers Disconnected](./runbooks/57-all-pollers-disconnected.md) | frontend | [Total Concurrent Pollers](../../dashboards/server/temporal-server-readme.md) | 1m |
| 59a | [Visibility Store Write Errors (Warning)](./runbooks/59a-visibility-store-write-errors.md) | history | [Visibility Write Error Rate per Store](../../dashboards/server/temporal-server-readme.md) | 2m |
| 59b | [Visibility Store Write Errors (Critical)](./runbooks/59a-visibility-store-write-errors.md) | history | [Visibility Write Error Rate per Store](../../dashboards/server/temporal-server-readme.md) | 1m |
| 59c | [Visibility Store Write Latency High](./runbooks/59a-visibility-store-write-errors.md) | history | [Visibility Write Latency per Store](../../dashboards/server/temporal-server-readme.md) | 5m |
| 80 | [History Task DLQ Stranding](./runbooks/80-history-task-dlq-stranding.md) | history | [Dead-Lettered Tasks — Execution-Stranding](../../dashboards/server/temporal-server-readme.md) | 10m |
| 81 | [Archival Backend Failing](./runbooks/81-archival-backend-failing.md) | history | [Archival Health — Signal 1](../../dashboards/server/temporal-server-readme.md) | 10m |
| 82 | [History Task DLQ Write Failures](./runbooks/82-history-task-dlq-write-failures.md) | history | [Archival Health — Signal 2](../../dashboards/server/temporal-server-readme.md) | 5m |
| 83 | [Visibility Tasks Dead-Lettered](./runbooks/83-visibility-tasks-dead-lettered.md) | history | [Visibility Tasks Dead-Lettered by Task Type](../../dashboards/server/temporal-server-readme.md) | 5m |
| 84 | [Visibility Store Not Acknowledging Writes](./runbooks/84-visibility-store-not-acknowledging-writes.md) | history | [Visibility Errors by Type per Store](../../dashboards/server/temporal-server-readme.md) | 5m |
| 85 | [Visibility Read Errors](./runbooks/85-visibility-read-errors.md) | frontend | [Visibility Read Error Rate per Store](../../dashboards/server/temporal-server-readme.md) | 2m |
| 86 | [History Database Calls Rejected](./runbooks/86-history-database-calls-rejected.md) | history | [History Rejected Database Calls Total by Scope](../../dashboards/server/temporal-server-readme.md) | 10m |

---

## Failover Alert Set — `temporal-failover-alerts.yaml`

Multi-cluster replication only. All five alerts link to the [Namespace Failover — Graceful Handover dashboard](../../dashboards/server/namespace-failover-graceful-handover.json) and [playbook](../../../playbooks/namespace-failover-graceful-handover.md).

| UID | Alert | Phase | Severity | `for` |
|---|---|---|---|---|
| FAILOVER-PRE-01 | Replication Stream Stuck | Pre-flight | critical | 2m |
| FAILOVER-PRE-02 | Replication DLQ Enqueue Failing | Pre-flight | critical | 2m |
| FAILOVER-PRE-03 | Replication Stream Reconnect Rate Elevated | Pre-flight | warning | 2m |
| FAILOVER-PRE-04 | Receiver Backlog At Flow Control Limit | Pre-flight | critical | 1m |
| FAILOVER-PRE-05 | Receiver Backlog Near Flow Control Limit | Pre-flight | warning | 2m |
| FAILOVER-PRE-06 | Standby Task Discards Detected | Pre-flight | warning | 2m |
| FAILOVER-HANDOVER-01 | HANDOVER Drain Stalled | Drain (Steps 4–5) | critical | 20s |
| FAILOVER-POST-01 | Forwarding FailedPrecondition Errors | Post-handover | warning | 2m |

> **Note:** `FAILOVER-HANDOVER-01` hardcodes `2048` as total shard count in its PromQL expression. Update this to match your cluster's `numHistoryShards` before deploying.

---

## Thresholds

All thresholds and `for` durations are starting points based on Temporal's default dynamic config values that should work well for most production deployments. Every cluster is different — a large multi-region deployment, a high-throughput namespace, or unusual workload patterns may need higher or lower thresholds, and shorter or longer `for` durations before an alert fires. Treat these values as a calibrated baseline and tune them to your specific setup. The `for` duration in particular controls how long a condition must hold before alerting — shorter values catch problems faster but increase noise from transient spikes; longer values reduce false positives but delay detection.

| # | Threshold | Basis |
|---|---|---|
| 1 | RPS < 1 | Zero traffic |
| 1b | Series absent | `absent()` returns 1 when no series match |
| 4 | RPS < 1 per namespace | Zero traffic |
| 8 | p99 shard lock latency > 300ms | Dashboard threshold reference |
| 12 | p99 persistence latency > 1s | Dashboard threshold reference |
| 25 | Any panic | Binary |
| 27 | Error rate > 30% sustained 2m | Ratio-based |
| 30 | Any `SystemOverloaded` or `CircuitBreakerOpen` error | Binary |
| 34 | Shard creation with zero restarts in 8m window | Compound |
| 34b | p99 immediate queue lag > 3M tasks | Dashboard red threshold |
| 34f | Any `dd_current_suspected_deadlocks > 0` | Binary; noDataState OK (event-driven) |
| 34j | `dd_shard_io_semaphore_latency` p99 > 20s | Half the 40s deadlock detector timeout (10s DB op + 30s grace, from source) — fires early enough to act before 34f |
| 38 | p99 timer lag > 30s | Planning doc threshold |
| 57 | Pollers < 1 per namespace | Zero workers |

---

## Runbooks

Each alert has a runbook in [runbooks/](./runbooks/). Runbook content (triage steps, remediation) is filled in after alerts have been tested and validated against a live cluster.

---

## Related Resources

- [Alert Planning Document](./planning.md) — full alert inventory, design decisions, PromQL reference
- [Temporal Server Dashboard README](../../dashboards/server/temporal-server-readme.md)
- [Temporal Dynamic Config Reference](../../../dynamic-config/README.md)
