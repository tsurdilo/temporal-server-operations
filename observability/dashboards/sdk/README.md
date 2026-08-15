# Temporal SDK Grafana Dashboards

Grafana dashboards for monitoring Temporal SDK workers and clients.

---

## Dashboards

### Temporal Go SDK Dashboard — v1.0.0

Monitors Temporal Go SDK workers and clients using the default Prometheus metrics reporter.

| File | Description |
|---|---|
| [temporal-sdk-go.json](./temporal-sdk-go.json) | Grafana dashboard JSON — import this into Grafana |
| [temporal-sdk-go-readme.md](./temporal-sdk-go-readme.md) | Full panel reference and setup guide |
| [temporal-sdk-go-changelog.md](./temporal-sdk-go-changelog.md) | Version history |

---

### Temporal Java SDK Dashboard (Micrometer) — v1.0.0

Monitors Temporal Java SDK workers configured with `MicrometerClientStatsReporter` and a Prometheus meter registry.

| File | Description |
|---|---|
| [temporal-sdk-java-micrometer.json](./temporal-sdk-java-micrometer.json) | Grafana dashboard JSON — import this into Grafana |
| [temporal-sdk-java-micrometer-readme.md](./temporal-sdk-java-micrometer-readme.md) | Full panel reference and setup guide |
| [temporal-sdk-java-micrometer-changelog.md](./temporal-sdk-java-micrometer-changelog.md) | Version history |

---

### Temporal Java SDK Dashboard (OTel) — v1.0.0

Monitors Temporal Java SDK workers configured with an OpenTelemetry-based metrics reporter and a Prometheus exporter.

| File | Description |
|---|---|
| [temporal-sdk-java-otel.json](./temporal-sdk-java-otel.json) | Grafana dashboard JSON — import this into Grafana |
| [temporal-sdk-java-otel-readme.md](./temporal-sdk-java-otel-readme.md) | Full panel reference and setup guide |
| [temporal-sdk-java-otel-changelog.md](./temporal-sdk-java-otel-changelog.md) | Version history |

---

### Temporal Core SDK Dashboard (TypeScript, Python, .NET, Ruby) — v1.0.0

Monitors Temporal workers built on the Core SDK — the shared Rust library that powers the TypeScript, Python, .NET, and Ruby SDKs.

| File | Description |
|---|---|
| [temporal-sdk-core.json](./temporal-sdk-core.json) | Grafana dashboard JSON — import this into Grafana |
| [temporal-sdk-core-readme.md](./temporal-sdk-core-readme.md) | Full panel reference and setup guide |
| [temporal-sdk-core-changelog.md](./temporal-sdk-core-changelog.md) | Version history |

---

## Updating to a new version

When a new version is available, reimport the updated JSON into Grafana via **Dashboards → Import**. Because the dashboard UID is stable across versions, Grafana will update your existing dashboard in place rather than creating a duplicate. Check the relevant changelog before updating to see what changed.