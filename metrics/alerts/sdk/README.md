# Temporal SDK — Grafana Alerts

Grafana alerting provisioning rules for Temporal SDK workers on a self-hosted Temporal Server cluster.

> **Current scope:** Essential Alert Set — 23 alerts covering the most impactful SDK failure modes across all four reporters (Java Micrometer, Java OTel, Go SDK, Core SDK). See [alerts-index.md](./alerts-index.md) for the full inventory of all 36 defined alerts, and [planning.md](./planning.md) for design decisions and working notes.

---

## Table of Contents

- [Setup](#setup)
  - [1. Prerequisites](#1-prerequisites)
  - [2. Choose your reporter YAML](#2-choose-your-reporter-yaml)
  - [3. Drop the file into Grafana provisioning](#3-drop-the-file-into-grafana-provisioning)
  - [4. Set the datasource UID](#4-set-the-datasource-uid)
  - [5. Configure notification policies](#5-configure-notification-policies)
- [Essential Alert Set](#essential-alert-set)
- [Thresholds](#thresholds)
- [Runbooks](#runbooks)
- [Related Resources](#related-resources)

---

## Setup

### 1. Prerequisites

- Temporal Server v1.20+ emitting Prometheus metrics
- Grafana 9.0+ with a Prometheus datasource
- SDK workers instrumented with one of the four supported reporters — Java Micrometer, Java OTel, Go SDK, or Core SDK
- The relevant SDK dashboard imported — dashboard panel links in alert notifications will not resolve without it:
  - [temporal-sdk-go.json](../../dashboards/sdk/temporal-sdk-go.json)
  - [temporal-sdk-java-micrometer.json](../../dashboards/sdk/temporal-sdk-java-micrometer.json)
  - [temporal-sdk-java-otel.json](../../dashboards/sdk/temporal-sdk-java-otel.json)
  - [temporal-sdk-core.json](../../dashboards/sdk/temporal-sdk-core.json)

### 2. Choose your reporter YAML

Each SDK reporter emits metrics with different naming conventions. Use the YAML file that matches your reporter:

| Reporter | YAML file | Counter suffix | Histogram suffix |
|---|---|---|---|
| Java Micrometer | `temporal-sdk-java-micrometer-alerts.yaml` | `_total` | `_seconds_bucket` |
| Java OTel | `temporal-sdk-java-otel-alerts.yaml` | `_total` | `_seconds_bucket` |
| Go SDK | `temporal-sdk-go-alerts.yaml` | `_total` | `_seconds_bucket` |
| Core SDK (Python/.NET/Ruby) | `temporal-sdk-core-alerts.yaml` | `_total` | `_seconds_bucket` |

> If you run multiple SDK reporters in the same Prometheus instance, import multiple YAML files. Each file uses a unique alert group name to avoid conflicts.

### 3. Drop the file into Grafana provisioning

Copy the relevant YAML file into your Grafana provisioning directory:

```
<grafana-root>/provisioning/alerting/temporal-sdk-<reporter>-alerts.yaml
```

Restart Grafana (or wait for the hot-reload interval). The alerts will appear under **Alerting → Alert rules → Temporal SDK**.

### 4. Set the datasource UID

Grafana requires the datasource **UID** (not name) in provisioned alert rules. Each YAML file uses `Prometheus` as a placeholder — substitute your actual UID before deploying.

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
  temporal-sdk-go-alerts.yaml \
  > /path/to/grafana/provisioning/alerting/temporal-sdk-go-alerts.yaml
```

The UID is stable for the lifetime of the Grafana instance. Re-run only if you wipe Grafana's database and recreate the datasource from scratch.

### 5. Configure notification policies

The alerts ship with labels but no notification policy. Wire them up in **Alerting → Notification policies** based on your routing needs. All Essential Set alerts carry:

```
severity: critical | warning
service: temporal-sdk
component: worker | poller | cache | wft | activity | local-activity | request
```

---

## Essential Alert Set

| # | Alert | Severity | Component | `for` | Runbook |
|---|---|---|---|---|---|
| 1a | [NOT_FOUND on WFT Respond Operations](./runbooks/01a-not-found-wft-respond.md) | 🔴 Critical | wft | 5m |
| 1b | [NOT_FOUND on Activity Respond Operations](./runbooks/01b-not-found-activity-respond.md) | 🔴 Critical | activity | 5m |
| 1c | [NOT_FOUND on RecordActivityTaskHeartbeat](./runbooks/01c-not-found-heartbeat.md) | ⚠️ Warning | activity | 5m |
| 2a | [RESOURCE_EXHAUSTED on Critical User-Facing Operations](./runbooks/02a-resource-exhausted-critical-ops.md) | 🔴 Critical | request | 1m |
| 2b | [RESOURCE_EXHAUSTED on Respond Operations](./runbooks/02b-resource-exhausted-respond-ops.md) | 🔴 Critical | request | 5m |
| 5 | [Request Failure: UNIMPLEMENTED](./runbooks/05-unimplemented.md) | 🔴 Critical | request | 2m |
| 6 | [Request Failure: INTERNAL](./runbooks/06-internal.md) | 🔴 Critical | request | 2m |
| 12 | [RespondWorkflowTaskCompleted Rate Dropped to Zero](./runbooks/12-wft-completions-zero.md) | 🔴 Critical | wft | 5m |
| 13 | [RespondActivityTaskCompleted Rate Dropped to Zero](./runbooks/13-activity-completions-zero.md) | 🔴 Critical | activity | 5m |
| 14 | [Worker Task Slots Exhausted](./runbooks/14-worker-slots-exhausted.md) | 🔴 Critical | worker | 2m |
| 15 | [All Pollers Disconnected](./runbooks/15-all-pollers-disconnected.md) | 🔴 Critical | poller | 5m |
| 19 | [WFT Schedule-To-Start Latency Critical](./runbooks/19-wft-schedule-to-start-critical.md) | 🔴 Critical | wft | 5m |
| 19b | [WFT Schedule-To-Start Latency Severely Elevated](./runbooks/19b-wft-schedule-to-start-severe.md) | 🔴 Critical | wft | 5m |
| 20b | [Activity Schedule-To-Start Latency Severely Elevated](./runbooks/20b-activity-schedule-to-start-severe.md) | 🔴 Critical | activity | 5m |
| 21 | [WFT Execution Failed: NonDeterminismError](./runbooks/21-nde.md) | 🔴 Critical | wft | 1m |
| 22 | [WFT Execution Failed: GrpcMessageTooLarge](./runbooks/22-grpc-message-too-large.md) | 🔴 Critical | wft | 1m |
| 26 | [Activity Execution Failed Rate Elevated](./runbooks/26-activity-execution-failed.md) | ⚠️ Warning | activity | 5m |
| 27 | [Unregistered Activity Invocation](./runbooks/27-unregistered-activity.md) | 🔴 Critical | activity | 1m |
| 29 | [Local Activity Execution Latency Exceeds WFT Heartbeat Timeout](./runbooks/29-la-execution-latency-timeout.md) | 🔴 Critical | local-activity | 5m |
| 30 | [Local Activity Total Execution Latency Exceeds WFT Heartbeat Timeout](./runbooks/30-la-total-latency-timeout.md) | 🔴 Critical | local-activity | 5m |
| 31 | [Request Latency High on Critical User-Facing Operations](./runbooks/31-request-latency-critical-ops.md) | 🔴 Critical | request | 5m |
| 32 | [WFT Execution Latency Critical](./runbooks/32-wft-execution-latency-critical.md) | 🔴 Critical | wft | 5m |
| 34 | [Sticky Cache Disabled](./runbooks/34-sticky-cache-disabled.md) | ⚠️ Warning | cache | 5m |

---

## Thresholds

All thresholds are starting points. Adjust to match your workload, namespace SLOs, and cluster size.

| # | Threshold | Basis |
|---|---|---|
| 1a | Any NOT_FOUND | Binary — always unexpected |
| 1b | Any NOT_FOUND | Binary — always unexpected |
| 1c | Any NOT_FOUND | Binary — always unexpected |
| 2a | Any RESOURCE_EXHAUSTED | Binary — server throttling critical ops |
| 2b | Any RESOURCE_EXHAUSTED | Binary — server throttling respond ops |
| 5 | Any UNIMPLEMENTED | Binary — deployment or routing issue |
| 6 | Any INTERNAL | Binary — server-side infrastructure issue |
| 12 | Rate == 0 per task queue | Zero WFT completions |
| 13 | Rate == 0 per task queue | Zero activity completions |
| 14 | Available slots == 0 | All slots occupied |
| 15 | Pollers == 0 | No active pollers |
| 19 | p99 > 5s | Severe WFT latency |
| 19b | p99 > 1800s (30m) | Critical WFT latency — large backlog risk |
| 20b | p99 > 1800s (30m) | Critical activity latency — large backlog risk |
| 21 | Any NDE | Binary — executions stuck |
| 22 | Any GrpcMessageTooLarge | Binary — executions terminated |
| 26 | rate > 20/s | Sustained activity failure churn |
| 27 | Any unregistered invocation | Binary — deployment bug |
| 29 | p99 > 1800s (30m) | Single LA attempt exceeds WFT heartbeat timeout |
| 30 | p99 > 1800s (30m) | LA retry chain exceeds WFT heartbeat timeout |
| 31 | p99 > 2s | User-facing SDK call latency |
| 32 | p99 > 10s | At or past default WFT timeout (10s) |
| 34 | Size == 0 | Cache disabled — full replay on every WFT |

---

## Runbooks

Each Essential Set alert has a runbook in [runbooks/](./runbooks/) with triage steps, remediation guidance, and cross-references to related alerts and server dashboard panels.

---

## Related Resources

- [Full Alert Index](./alerts-index.md) — all 36 alerts with descriptions and per-SDK PromQL
- [Alert Planning Document](./planning.md) — design decisions and working notes
- [Temporal SDK Go Dashboard README](../../dashboards/sdk/temporal-sdk-go-readme.md)
- [Temporal SDK Java Micrometer Dashboard README](../../dashboards/sdk/temporal-sdk-java-micrometer-readme.md)
- [Temporal SDK Java OTel Dashboard README](../../dashboards/sdk/temporal-sdk-java-otel-readme.md)
- [Temporal SDK Core Dashboard README](../../dashboards/sdk/temporal-sdk-core-readme.md)
- [Temporal Server Alerts](../server/README.md)
- [Temporal Dynamic Config Reference](../../../dynamic_config/README.md)
