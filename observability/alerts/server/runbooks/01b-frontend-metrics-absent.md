## Frontend Metrics Absent

**Severity:** Critical
**Component:** frontend
**Dashboard panel:** [Total RPS](https://github.com/tsurdilo/temporal-metrics/blob/main/observability/dashboards/server/temporal-server-readme.md) — panel ID 21

### What this alert detects

Complete absence of `service_requests` metrics reported from any frontend service pod for 2 consecutive minutes.

### Why it matters

All frontend pods may have crashed or are struggling to the point where they are no longer reporting metrics. Unlike alert 1, which requires a series value to evaluate, this alert fires when the series itself is gone.

### Triage steps

1. Open the **Total RPS** panel (panel 21) in the Temporal Server dashboard — the panel will show no data or a gap in the series
2. Check if alert 1 (Total RPS Drops to Zero) is also firing — if both fire simultaneously, the series was present but at zero before disappearing
3. Check frontend pod health — are any pods running at all?
4. Check Prometheus scrape targets (`/targets` in Prometheus UI) — confirm whether the frontend scrape job is healthy or erroring

### Relevant dynamic config

None — a missing series is an infrastructure/process health issue, not a config issue.
