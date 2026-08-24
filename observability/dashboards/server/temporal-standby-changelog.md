# Changelog — Temporal Standby Cluster — Replication Health Dashboard

## v2.1.0 — 2026-08-24

- Added **Replication Latency — Time Behind (End-to-End)** panel (id 1017) to the 📉 Replication Lag row. Dedicated view of `replication_latency` (create→apply wall-clock, p50 and p99) — the single best "how many seconds is the standby behind" signal. Red threshold at 20s aligns with alert `FAILOVER-PRE-07`. The metric was previously only visible as one of five series in the combined **Replication Latencies** panel.

## v2.0.0 — 2026-05-12

First versioned release. Prior changes were unversioned.
