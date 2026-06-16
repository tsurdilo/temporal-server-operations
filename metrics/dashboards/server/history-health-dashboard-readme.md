# Temporal History Host Health Dashboard

A Grafana dashboard for monitoring the deep health of a self-hosted Temporal Server history fleet using Prometheus metrics.

> **Compatibility:** Temporal Server v1.20+ · Grafana 9.0+ · Prometheus · Kubernetes

> **Current version:** v1.2.0 — see [CHANGELOG](./history-health-dashboard-changelog.md)

---

## Table of Contents

- [Overview](#overview)
- [Relationship to Temporal Server Overview Dashboard](#relationship-to-temporal-server-overview-dashboard)
- [Prerequisites](#prerequisites)
- [Setup](#setup)
- [Template Variables](#template-variables)
- [Groups and Panels](#groups-and-panels)
  - [History Host Health](#1-history-host-health)
  - [Service Readiness (gRPC Health)](#2-service-readiness-grpc-health)
  - [Persistence Health](#3-persistence-health)
  - [History RPC Health](#4-history-rpc-health)
  - [Shard Acquisition Health and Movement](#5-shard-acquisition-health-and-movement)
- [Threshold Reference](#threshold-reference)
- [Panels Duplicated from Server Overview](#panels-duplicated-from-server-overview)
- [Known Limitations and Adjustments](#known-limitations-and-adjustments)
- [Related Resources](#related-resources)

---

## Overview

This dashboard provides deep health monitoring for the Temporal history service fleet, focused on the signals needed to detect degradation and make informed failover decisions. It is built on Prometheus metrics emitted by Temporal Server and Kubernetes (`kube-state-metrics`).

The dashboard is organized around a single question: **is the history fleet healthy, and if not, why?**

It is intentionally cluster-level, not namespace-level. There is no Temporal namespace variable — all panels show fleet-wide state.

---

## Relationship to Temporal Server Overview Dashboard

This dashboard is a **companion to** the [Temporal Server Overview](../temporal-server/README.md) dashboard, not a replacement for it. They serve different purposes:

| Dashboard | Purpose | Primary audience |
|---|---|---|
| **Temporal Server Overview** | Cluster throughput, per-namespace workload health, workflow stats, matching, replication | Day-to-day operations, capacity planning |
| **Temporal History Host Health** (this dashboard) | History fleet deep health, failover signal, incident root cause | Incident response, SRE on-call |

Some panels in this dashboard intentionally duplicate panels from the Server Overview (specifically persistence latency and history RPC latency). This is by design — during an incident you should not need to switch dashboards to correlate `host_health` state with its root cause. The duplicates are noted in the [Panels Duplicated from Server Overview](#panels-duplicated-from-server-overview) section.

---

## Prerequisites

### 1. Temporal Server Prometheus metrics

Ensure your Temporal Server is configured to expose Prometheus metrics:

```yaml
metrics:
  prometheus:
    timerType: "histogram"
    listenAddress: "0.0.0.0:8000"
```

### 2. `kube-state-metrics`

The Service Readiness row requires `kube-state-metrics` to be running in your cluster, which provides `kube_pod_status_ready`. This is standard in most Kubernetes monitoring setups (Prometheus Operator, kube-prometheus-stack, etc.).

### 3. The `host_health` poller

> ⚠️ **The `host_health` metric is absent by default.** It is only emitted when `AdminHandler.DeepHealthCheck` is called explicitly on the Temporal frontend. If nothing in your infrastructure calls this endpoint, the entire History Host Health row will show no data.

You must run a poller that calls `AdminHandler.DeepHealthCheck` on a schedule. The polling interval determines metric freshness — set it based on your alerting SLA requirements. The call itself is what triggers the server to emit the `host_health` gauge; your poller does not need to parse or expose the response, though logging it is useful for debugging.

**Minimal Go poller:**

```go
package main

import (
	"context"
	"log"
	"os"
	"os/signal"
	"syscall"
	"time"

	adminservice "go.temporal.io/server/api/adminservice/v1"
	"google.golang.org/grpc"
	"google.golang.org/grpc/credentials/insecure"
)

func main() {
	frontendAddr := getenv("FRONTEND_ADDR", "temporal-loadbalancing:7233")
	pollInterval := mustParseDuration(getenv("POLL_INTERVAL", "15s"))

	conn, err := grpc.NewClient(frontendAddr, grpc.WithTransportCredentials(insecure.NewCredentials()))
	if err != nil {
		log.Fatalf("failed to connect: %v", err)
	}
	defer conn.Close()

	client := adminservice.NewAdminServiceClient(conn)

	ctx, cancel := signal.NotifyContext(context.Background(), os.Interrupt, syscall.SIGTERM)
	defer cancel()

	ticker := time.NewTicker(pollInterval)
	defer ticker.Stop()

	log.Printf("starting DeepHealthCheck poller against %s, interval %s", frontendAddr, pollInterval)

	for {
		select {
		case <-ctx.Done():
			log.Println("poller stopped")
			return
		case <-ticker.C:
			pollCtx, pollCancel := context.WithTimeout(ctx, 10*time.Second)
			resp, err := client.DeepHealthCheck(pollCtx, &adminservice.DeepHealthCheckRequest{})
			pollCancel()
			if err != nil {
				log.Printf("DeepHealthCheck error: %v", err)
				continue
			}
			log.Printf("cluster state: %s", resp.State)
			for _, svc := range resp.Services {
				for _, host := range svc.Hosts {
					log.Printf("  service=%s host=%s state=%s", svc.Service, host.Address, host.State)
				}
			}
		}
	}
}

func getenv(key, fallback string) string {
	if v := os.Getenv(key); v != "" {
		return v
	}
	return fallback
}

func mustParseDuration(s string) time.Duration {
	d, err := time.ParseDuration(s)
	if err != nil {
		log.Fatalf("invalid duration %q: %v", s, err)
	}
	return d
}
```

**`go.mod`:**
```
module temporal-health-poller

go 1.26.2

require (
	go.temporal.io/server v1.31.0
	google.golang.org/grpc v1.79.3
)
```

Pin `go.temporal.io/server` to the same minor version as your Temporal cluster. The gRPC version is specified in the server's `go.mod`.

**Dockerfile:**
```dockerfile
FROM golang:1.26-alpine AS builder
RUN apk add --no-cache git
WORKDIR /app
COPY go.mod go.sum main.go ./
RUN CGO_ENABLED=0 GOOS=linux go build -o poller .

FROM alpine:latest
RUN apk add --no-cache ca-certificates
COPY --from=builder /app/poller /poller
ENTRYPOINT ["/poller"]
```

**For TLS**, replace `insecure.NewCredentials()` with `credentials.NewTLS(tlsConfig)` from `google.golang.org/grpc/credentials`.

**Deployment notes:**
- Run this as a sidecar or dedicated Deployment in the same Kubernetes namespace as Temporal.
- One poller instance is sufficient regardless of how many history pods you have — the frontend fans out the check to all history hosts internally.
- If the frontend is unreachable, the poller logs the error and retries on the next tick; `host_health` will go stale and the Metric Freshness panel will alert at 120s.

---

## Setup

1. Import `temporal-history-health.json` into Grafana via **Dashboards → Import**.
2. Select your Prometheus datasource when prompted.
3. Set the `K8s Namespace` variable to the Kubernetes namespace where Temporal is deployed.
4. Verify the History Host Health row shows data — if all panels are empty, the poller is not running.

---

## Template Variables

| Variable | Description | Source |
|---|---|---|
| **Datasource** | Prometheus datasource to use | — |
| **K8s Namespace** | Kubernetes namespace where Temporal is deployed | `label_values(kube_pod_status_ready, namespace)` |
| **Percentile** | Histogram quantile for latency panels | Custom: 0.50 / 0.75 / 0.90 / 0.95 / 0.99 / 0.999, default 0.99 |

Note: there is no Temporal namespace variable. This dashboard is cluster-level, not per-namespace.

---

## Groups and Panels

---

### 1. History Host Health

The primary row. All other rows exist to explain what you see here.

> ⚠️ All panels in this row require the `host_health` poller to be running. See [Prerequisites](#prerequisites).

**Health state values** (from `enumsspb.HEALTH_STATE_*`):

| Value | State | Meaning |
|---|---|---|
| `0` | `UNSPECIFIED` | Pod not reporting — treat as missing data |
| `1` | `SERVING` | All 5 DeepHealthCheck checks passed |
| `2` | `NOT_SERVING` | RPC or persistence threshold exceeded, or gRPC health not SERVING after the 60s init window |

> **Note:** the per-pod `host_health` gauge only ever emits values `0`, `1`, or `2`. Value `3` (DECLINED_SERVING) is a fleet-level aggregate state returned by the frontend's `healthCheckerImpl.Check` — it is never written to the per-pod gauge. Queries filtering `host_health == 3` on individual pods will always return no data.

| Panel | Type | Description |
|---|---|---|
| **Pods NOT_SERVING** | Stat | Count of history pods in state 2. Red at any value ≥ 1. |
| **Pods DECLINED_SERVING** | Stat | Always 0 — per-pod `host_health` never emits value 3. This panel is retained for completeness but will not show data. |
| **Pods SERVING** | Stat | Count of history pods in state 1. Red if zero. |
| **Total Pods Reporting** | Stat | Count of history pods emitting `host_health`. A drop here means pods disappeared or the poller stopped. |
| **Fleet Size Change** | Stat | Pods that stopped reporting `host_health` compared to peak count in the last hour. Non-zero means pods disappeared entirely — a stopped or crashed pod does not emit `host_health` and will not appear as NOT_SERVING. Red at ≥ 1. |
| **Metric Freshness** | Stat | Seconds since `host_health` was last updated. Red at 120s — poller is broken or frontend is unreachable. |
| **Pod Count by Health State** | Time series | All three states over time on one chart. Color coded green/red/yellow. |
| **Pod Health State Percentage** | Time series | Percentage of pods in each state. The `not_serving %` line has a threshold at 50% — the frontend `historyHostErrorPercentage` default that triggers cluster-level NOT_SERVING. |
| **Per-Pod Health State** | Table | Current health state per individual pod. Color coded by state value. Use this to identify which specific pods are degraded during an incident. |

---

### 2. Service Readiness (gRPC Health)

Tracks Kubernetes pod readiness for all three Temporal services, driven by gRPC health probes.

**Important:** these probes tell you pods are alive and past their startup conditions. They are **not** equivalent to `host_health` and are **not** sufficient for a failover decision on their own. A history pod can pass its readiness probe and still be in a degraded state if persistence latency or RPC error ratios exceed thresholds.

| Service | Flips to SERVING when | Notes |
|---|---|---|
| Frontend | `membershipMonitor.WaitUntilInitialized()` completes | Prerequisite for `host_health` poller to reach frontend |
| History | `InitialShardsAcquired()` + 5s stabilization | Most meaningful probe — requires real work |
| Matching | Immediately after `handler.Start()` | Weakest probe — no readiness guarantee |

| Panel | Type | Description |
|---|---|---|
| **Frontend Pods Ready** | Stat | Count of frontend pods with readiness passing. If this is zero, the poller cannot reach frontend and `host_health` will go absent. |
| **History Pods Ready** | Stat | Count of history pods with readiness passing. |
| **Matching Pods Ready** | Stat | Count of matching pods with readiness passing. |
| **Ready Pod Count Over Time** | Time series | All three services on one chart over time. A drop to zero on any service is a critical signal. |

---

### 3. Persistence Health

Tracks DB latency, errors, connection pool state, and session refresh activity for the history service. These are the signals that trigger `NOT_SERVING` via DeepHealthCheck checks 4 and 5.

**DeepHealthCheck persistence thresholds (dynamic config, defaults):**
- `history.healthPersistenceLatencyFailure` = 500ms average
- `history.healthPersistenceErrorRatio` = 0.90 (90%)

| Panel | Type | Description |
|---|---|---|
| **Persistence Latency (History)** | Time series | By operation at selected percentile. Threshold: 300ms orange, 1s red. |
| **Persistence Errors by Type (History)** | Time series | By operation and error type. Any sustained rate is a serious signal. |
| **Persistence Availability (History)** | Gauge | % of persistence requests that succeeded. 99% green, 95% orange. |
| **SQL DB Connection Pool** | Time series | max_open (configured ceiling), open, idle, in_use. SQL backends only — not emitted for Cassandra. |
| **Persistence Session Refresh Attempts** | Time series | Connection churn signal. Spikes indicate the pool is dropping and re-establishing connections. Absent in normal operation. |
| **Persistence Session Refresh Failures** | Time series | Failed reconnects. Red at any value — the pool cannot re-establish DB connections. |

---

### 4. History RPC Health

Tracks RPC latency, errors, shard ownership lost events, membership changes, and service panics on the history service. High RPC latency or error ratios trigger `NOT_SERVING` via DeepHealthCheck checks 2 and 3.

**DeepHealthCheck RPC thresholds (dynamic config, defaults):**
- `history.healthRPCLatencyFailure` = 500ms average
- `history.healthRPCErrorRatio` = 0.90 (90%)

| Panel | Type | Description |
|---|---|---|
| **History Service Latency** | Time series | By operation at selected percentile. Threshold: 400ms orange, 2s red. |
| **History Service Errors by Type** | Time series | By error type. Sustained rate can push error ratio past the 90% threshold. |
| **Shard Ownership Lost** | Time series | Red at any value. Indicates active shard rebalancing — explains transient DECLINED_SERVING. |
| **Membership Changes (History)** | Time series | Spikes correlate with shard rebalancing events and explain DECLINED_SERVING bursts. |
| **Service Panics (History)** | Time series | Red at any value. An unrecoverable goroutine error — the pod will restart. |

---

### 5. Shard Acquisition Health and Movement

Tracks shard ownership per pod, acquisition latency, and shard creation, removal, closing, and service restarts. This row explains both the duration and cause of `DECLINED_SERVING` on history pods.

A history pod stays in `DECLINED_SERVING` until all three `InitialShardsAcquired` conditions pass (joined membership ring, acquired all expected shards, per-shard engine and ownership checks), followed by a 5-second stabilization delay. For a cluster with 2048 shards across 14 pods (~146 shards per pod), this takes meaningful time during startup or rebalancing.

Shard movement most commonly occurs during cluster restarts and scaling of history hosts, but can also be triggered by elevated DB latency. Frequent history service restarts cause repeated shard movement — correlate the Service Restarts panel with spikes in shard creation and removal.

| Panel | Type | Description |
|---|---|---|
| **Shards Per Pod** | Time series | Shard count per individual pod. A drop on a specific pod indicates shard loss on that pod. |
| **Shard Acquisition Latency** | Time series | Latency to acquire shards at selected percentile. High values explain why pods stay in DECLINED_SERVING longer than expected during restarts or rebalancing. |
| **Shards Created** | Time series | Rate of shard items created. Spikes typically correlate with a cluster restart or a new history host coming online. |
| **Shards Removed** | Time series | Rate of shard items removed. Combined with Shards Created, shows the full shard rebalancing picture. |
| **Shards Closed** | Time series | Rate of shard items closed. Shards are closed when a history host loses ownership, typically during a restart or scale-down. |
| **Service Restarts** | Time series | Rate of history service restarts. Frequent restarts cause repeated shard movement and elevated latencies — correlate with Shards Created/Removed/Closed above. |

---

## Threshold Reference

| Row | Panel | Orange | Red | Notes |
|---|---|---|---|---|
| History Host Health | Pods NOT_SERVING | — | ≥ 1 | Any degraded pod warrants investigation |
| History Host Health | Pods DECLINED_SERVING | — | — | Always 0 — per-pod metric never emits state 3 |
| History Host Health | Fleet Size Change | — | ≥ 1 | Any missing pod; a dropped pod goes absent, not NOT_SERVING |
| History Host Health | Metric Freshness | 60s | 120s | 120s = poller likely broken |
| History Host Health | Pod Health % (not_serving) | — | 50% | Frontend cluster threshold |
| Persistence Health | Persistence Latency | 300ms | 1s | DeepHealthCheck check 4 fires at 500ms average |
| Persistence Health | Session Refresh Failures | — | Any | No expected baseline |
| Persistence Health | Persistence Availability | 95% | — | Below 99% investigate |
| History RPC Health | History Service Latency | 400ms | 2s | DeepHealthCheck check 2 fires at 500ms average |
| History RPC Health | Shard Ownership Lost | — | Any | Indicates active rebalancing |
| History RPC Health | Service Panics | — | Any | Always critical |

---

## Panels Duplicated from Server Overview

The following panels appear in both this dashboard and the [Temporal Server Overview](../temporal-server/README.md) dashboard. The duplication is intentional — these panels are needed here for incident correlation without switching dashboards.

| Panel | Server Overview row | Notes |
|---|---|---|
| History Service Latency | Service Latencies | Server Overview breaks down all 3 services; this dashboard shows history only |
| Persistence Latency | Persistence Requests, Latencies and Errors | Server Overview includes namespace filter; this dashboard does not |
| Persistence Errors by Type | Persistence Requests, Latencies and Errors | Same difference — no namespace filter here |
| Persistence Availability | Persistence Requests, Latencies and Errors | Identical query scoped to history service only |
| SQL DB Connection Pool | Persistence Requests, Latencies and Errors | Server Overview shows total `sum()`; this dashboard also shows `sum()` scoped to history |
| Shards Created | Shard Movement | Identical query scoped to `service_name="history"` |
| Shards Removed | Shard Movement | Identical query scoped to `service_name="history"` |
| Shards Closed | Shard Movement | Identical query scoped to `service_name="history"` |
| Service Restarts | Shard Movement | Server Overview shows all services; this dashboard scopes to `service_name="history"` only |


---

## Known Limitations and Adjustments

**`pod` vs `instance` label:**
Panels that show per-pod data use the `pod` label, which is standard when Prometheus scrapes via pod monitors (Prometheus Operator). If your setup scrapes via a service monitor instead, individual pods may appear as `instance`. Adjust the `legendFormat` in those panels if needed.

**Pod name regex:**
Service Readiness panels use regex patterns based on Helm chart defaults:
- Frontend: `.*frontend.*`
- History: `.*history.*`
- Matching: `.*matching.*`

If your Helm release name is not `temporal`, adjust these patterns. For example if your release is named `my-temporal`, the patterns would be `.*my-temporal.*frontend.*` etc.

**Cassandra backends:**
SQL connection pool panels (`persistence_sql_*`) are only emitted by SQL backends (PostgreSQL, MySQL). These panels will show no data on Cassandra deployments. This is expected.

**`host_health` label availability:**
All `host_health` queries assume a `service_name` label. This is emitted by OSS Temporal Server. Temporal Cloud uses different label naming (`temporal_service_type`) and is out of scope for this dashboard.

---

## Related Resources

- [Temporal Server Overview Dashboard](../temporal-server/README.md) — full cluster overview, per-namespace workload health
