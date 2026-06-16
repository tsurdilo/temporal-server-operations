# Changelog — Temporal Java SDK Dashboard (Micrometer)

## v1.2.0 — 2026-06-15

### Changed — Thresholds

Added and corrected threshold definitions across all timeseries panels. Thresholds now render as dashed lines in Grafana (previously defined threshold values were not rendering because `thresholdsStyle` was missing from the panel `custom` block).

**New latency thresholds (values grounded in Java SDK source defaults):**
- `Request Latency` — orange 2s, red 5s (anchored to default RPC timeout of 10s)
- `Workflow Task Schedule To Start Latency` — orange 1s, red 5s (anchored to default sticky schedule-to-start timeout of 5s)
- `Activity Schedule To Start Latency` — orange 10s only, no red (user-defined; wide range for LLM/batch use cases)
- `Workflow Task Replay Latency` — orange 5s, red 10s (anchored to default WFT timeout of 10s)
- `Local Activity Execution Latency` — orange 8s, red 10s (80% of WFT timeout triggers SDK heartbeat; 10s = WFT timeout)
- `Local Activity Succeed End-to-End Latency` — orange 8s, red 10s (same basis)
- `Local Activity Total Execution Latency` — orange 8s, red 10s (same basis)

**Worker Task Slots Available threshold:**
- `Worker Task Slots Available` — orange reference line at y=10, red reference line at y=0. Uses `palette-classic` color mode since this is a "lower is worse" metric (inverted from normal threshold direction); dashed lines serve as visual y-axis markers. Only emitted when using fixed-size slot suppliers (`maxConcurrentWorkflowTaskExecutionSize` etc.) — not emitted for resource-based tuner users. Corresponding alert should use `noDataState: OK` so resource-based tuner deployments do not false-fire on absent data.

**Sticky Cache Miss threshold:**
- `Sticky Cache Miss` — orange at 20/s, red at 50/s. Some cache misses are expected (first execution, worker restart, eviction pressure) but sustained high rates indicate the sticky cache is too small — workflows are being replayed from scratch on every task. Threshold basis: 50+/s is a clear signal of under-sized cache.

**Counter/rate panel threshold corrections:**
- `Request Failures` / `Long-Poll Request Failures` — changed from red at 1/s to orange at 0.1/s; per-status-code severity (`PERMISSION_DENIED`, `UNAUTHENTICATED`, `NOT_FOUND` on respond ops) is handled at the alert level
- `Workflow Task Execution Failed` — changed from red at 1/s to orange at 0.001/s (any non-zero rate of `NonDeterminismError` or `WorkflowError` is a signal worth seeing immediately)
- `Activity Execution Failed` — changed from red to orange (activities fail and retry by design; user controls retry options)
- `Local Activity Execution Failed` — changed from red to orange (user controls local activity retry options)
- `Unregistered Activity Invocation` — lowered red threshold from 1/s to 0.001/s (always a code bug; any non-zero rate requires investigation)

---

## v1.1.0 — 2026-05-23

### Added
- **Workflow Task Heartbeat** panel in the _Workflow Task Info_ group — tracks `temporal_workflow_task_heartbeat_total`, the counter incremented each time the SDK forces a new workflow task because a local activity has consumed 80% of the workflow task timeout (`RespondWorkflowTaskCompleted` with `force_create_new_workflow_task=true`).
- `temporal_workflow_task_heartbeat` → `temporal_workflow_task_heartbeat_total` added to the counter metrics naming table in the readme.

---

## v1.0.0 — 2026-05-12

Initial versioned release.
