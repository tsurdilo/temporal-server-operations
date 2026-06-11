# Temporal Server — Grafana Alerts

Grafana alerting provisioning rules for a self-hosted Temporal Server cluster.

> **Current scope:** Essential Alert Set — 14 alerts covering the most impactful failure modes, plus 3 dual visibility store alerts (59a–59c). See [planning.md](./planning.md) for the full alert inventory (91 alerts) and the roadmap for future additions.

---

## Setup

### 1. Prerequisites

- Temporal Server v1.20+ emitting Prometheus metrics
- Grafana 9.0+ with a Prometheus datasource named **`Prometheus`**
- [temporal-server.json](../../dashboards/server/temporal-server.json) dashboard imported — dashboard panel links in alert notifications will not resolve without it

### 2. Drop the file into Grafana provisioning

Copy `temporal-server-alerts.yaml` into your Grafana provisioning directory:

```
<grafana-root>/provisioning/alerting/temporal-server-alerts.yaml
```

Restart Grafana (or wait for the hot-reload interval). The alerts will appear under **Alerting → Alert rules → Temporal Server**.

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

sed "s/datasourceUid: Prometheus/datasourceUid: ${GRAFANA_DS_UID}/g" \
  temporal-server-alerts.yaml \
  > /path/to/grafana/provisioning/alerting/temporal-server-alerts.yaml
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
| 38 | [Timer Task Scheduling Lag Critical](./runbooks/38-timer-scheduling-lag-critical.md) | history | [Timer Task Scheduling Latency](../../dashboards/server/temporal-server-readme.md) | 5m |
| 57 | [All Pollers Disconnected](./runbooks/57-all-pollers-disconnected.md) | frontend | [Total Concurrent Pollers](../../dashboards/server/temporal-server-readme.md) | 1m |
| 74 | [Matching Partition Sync Throttle Active](./runbooks/74-matching-sync-throttle-active.md) | matching | [Sync Throttle Count](../../dashboards/server/temporal-server-readme.md) | 1m |
| 59a | [Visibility Store Write Errors (Warning)](./runbooks/59a-visibility-store-write-errors.md) | history | [Visibility Write Error Rate per Store](../../dashboards/server/temporal-server-readme.md) | 2m |
| 59b | [Visibility Store Write Errors (Critical)](./runbooks/59a-visibility-store-write-errors.md) | history | [Visibility Write Error Rate per Store](../../dashboards/server/temporal-server-readme.md) | 1m |
| 59c | [Visibility Store Write Latency High](./runbooks/59a-visibility-store-write-errors.md) | history | [Visibility Write Latency per Store](../../dashboards/server/temporal-server-readme.md) | 5m |

---

## Thresholds

All thresholds are starting points based on Temporal's default dynamic config values. Adjust to match your cluster size, workload, and SLO requirements.

| # | Threshold | Basis |
|---|---|---|
| 1 | RPS < 1 | Zero traffic |
| 1b | Series absent | `absent()` returns 1 when no series match |
| 4 | RPS < 1 per namespace | Zero traffic |
| 8 | p99 shard lock latency > 300ms | Dashboard threshold reference |
| 12 | p99 persistence latency > 1s | Dashboard threshold reference |
| 25 | Any panic | Binary |
| 27 | Error rate > 30% sustained 2m | Ratio-based |
| 30 | Any `RESOURCE_EXHAUSTED_CAUSE_SYSTEM_OVERLOADED` or `RESOURCE_EXHAUSTED_CAUSE_CIRCUIT_BREAKER_OPEN` error | Binary |
| 34 | Shard creation with zero restarts in 8m window | Compound |
| 34b | p99 immediate queue lag > 3M tasks | Dashboard red threshold |
| 34f | Any `dd_current_suspected_deadlocks > 0` | Binary; noDataState OK (event-driven) |
| 38 | p99 timer lag > 30s | Planning doc threshold |
| 57 | Pollers < 1 per namespace | Zero workers |
| 74 | Any sync throttle | Binary |

---

## Runbooks

Each alert has a runbook in [runbooks/](./runbooks/). Runbook content (triage steps, remediation) is filled in after alerts have been tested and validated against a live cluster.

---

## Related Resources

- [Alert Planning Document](./planning.md) — full alert inventory, design decisions, PromQL reference
- [Temporal Server Dashboard README](../../dashboards/server/temporal-server-readme.md)
- [Temporal Dynamic Config Reference](../../../dynamic_config/README.md)
- [Temporal Server Metrics Reference](https://docs.temporal.io/references/cluster-metrics)
