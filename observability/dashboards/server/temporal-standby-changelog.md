# Changelog — Temporal Standby Cluster — Replication Health Dashboard

## v2.2.0 — 2026-08-24

- Added **🧹 History Scavenger** row (appended last): surfaces whether this standby's history scavenger is keeping up with leftover-history cleanup. The scavenger skips branches younger than `worker.historyScannerDataMinAge` (default 60 days), so for short-lived, high-volume workflows it can skip nearly everything and let `history_tree` / `history_node` grow. Two panels — **Scavenger Activity — Skipped vs Handled** (`scavenger_skips` vs `scavenger_success`) and **Scavenger Errors** (`scavenger_errors`), all `operation="HistoryScavenger"`. Metric names and semantics verified against server source (`service/worker/scanner/history/scavenger.go`); `scavenger_success` = branches handled (kept or deleted), not deletions. Backs the new **[XDC Standby Database Growth on SQL playbook](../../../playbooks/xdc-standby-database-growth-sql.md)**.

## v2.1.0 — 2026-08-24

- Added **Replication Latency — Time Behind (End-to-End)** panel (id 1017) to the 📉 Replication Lag row. Dedicated view of `replication_latency` (create→apply wall-clock, p50 and p99) — the single best "how many seconds is the standby behind" signal. Red threshold at 20s aligns with alert `FAILOVER-PRE-07`. The metric was previously only visible as one of five series in the combined **Replication Latencies** panel.

## v2.0.0 — 2026-05-12

First versioned release. Prior changes were unversioned.
