# Changelog — Temporal Core SDK Dashboard (TypeScript, Python, .NET, Ruby)

## v1.1.0 — 2026-06-15

### Changed — Thresholds

Added and corrected threshold definitions across all timeseries panels. Thresholds now render as dashed lines in Grafana (previously defined threshold values were not rendering because `thresholdsStyle` was missing from the panel `custom` block).

**New latency thresholds:**
- `Request Latency` — orange 2s, red 5s (anchored to default RPC timeout of 10s)
- `Workflow Task Schedule To Start Latency` — orange 1s, red 5s (anchored to default sticky schedule-to-start timeout of 5s)
- `Activity Schedule To Start Latency` — orange 10s only, no red (user-defined; wide range for LLM/batch use cases)
- `Workflow Task Replay Latency` — orange 5s, red 10s (anchored to default WFT timeout of 10s)
- `Local Activity Execution Latency` — orange 8s, red 10s (80% of WFT timeout triggers SDK heartbeat; 10s = WFT timeout)
- `Local Activity Succeed End-to-End Latency` — orange 8s, red 10s (same basis)

**Worker Task Slots Available threshold:**
- `Worker Task Slots Available` — orange reference line at y=10, red reference line at y=0. Uses `palette-classic` color mode (inverted metric — lower is worse). Only emitted when using fixed-size slot suppliers; corresponding alert should use `noDataState: OK` for resource-based tuner deployments.

**Sticky Cache Miss threshold:**
- `Sticky Cache Miss` — orange at 20/s, red at 50/s. Sustained high rates indicate the sticky cache is too small — workflows are being replayed from scratch on every task.

**Counter/rate panel threshold corrections:**
- `Request Failures` / `Long-Poll Request Failures` — changed from red at 1/s to orange at 0.1/s; per-status-code severity handled at the alert level
- `Workflow Task Execution Failed` — changed to orange at 0.001/s (any non-zero rate of `NonDeterminismError` or `WorkflowError` is a signal)
- `Activity Execution Failed` — changed from red to orange (activities fail and retry by design)
- `Local Activity Execution Failed` — changed from red to orange (user controls local activity retry options)
- `Unregistered Activity Invocation` — lowered red threshold to 0.001/s (always a code bug)

---

## v1.0.0 — 2026-05-12

Initial versioned release.
