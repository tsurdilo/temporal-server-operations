---
name: temporal-metrics
description: >
  This skill should be used when the user asks about "temporal_request_failure",
  "temporal_long_request_failure", "ResourceExhausted", "SDK metrics",
  "rate limiting", "throttling", "gRPC status codes", "namespace RPS",
  "execution bucket", "visibility bucket", "poll throttling",
  "schedule to start latency", "workflow task timeout", "activity timeout",
  "heartbeat timeout", "resource exhausted cause", "server dashboard",
  "metric troubleshooting", "Temporal production incident",
  "request failure metrics", "graph patterns", "RPS quota",
  "frontend.namespaceRPS", or asks to diagnose Temporal metrics or
  troubleshoot Temporal SDK errors.
version: 0.1.0
---

# Temporal Metrics Troubleshooting Skill

You now have access to a production-focused troubleshooting knowledge base for Temporal SDK request failure metrics. This knowledge comes from 5+ years of real-world experience across hundreds of Temporal implementations — production incidents, load tests, and hard-won debugging sessions. It is not a metrics glossary. It connects dots across related signals so you can build a complete picture of what is happening.

## What This Skill Covers

- **`temporal_request_failure`** and **`temporal_long_request_failure`** — the two most useful SDK metrics for troubleshooting Temporal applications
- How the SDK distinguishes the two metrics (based on `LongPollContextKey`)
- SDK gRPC retry behavior — which status codes are retried, which are not, and what that means for business impact
- Alert severity tiers — which failures warrant immediate alerting vs. passive monitoring
- Diagnostic tables mapping every operation + status code combination to what it means operationally
- Server-side rate limit buckets (Execution, Visibility, NamespaceReplicationInducing) and priority levels
- How to correlate SDK metrics with server dashboard panels
- Visual graph pattern interpretation for RPS failure graphs

## How to Use the References

Read the appropriate reference file based on what the user needs help with:

### SDK Request Failure Metrics (the main guide)

**Read `references/sdk-request-failures.md`** for:
- Understanding what `temporal_request_failure` or `temporal_long_request_failure` means
- Diagnosing a specific operation + status code combination
- Understanding alert severity and business impact
- Server-side throttling theory (rate limit buckets, priorities, dynamic config keys)
- Mapping SDK metrics to server dashboard panels
- Understanding gRPC retry behavior

### Graph Pattern Reference

**Read `references/graph-patterns.md`** for:
- Interpreting visual patterns in RPS failure graphs
- Identifying root causes from graph shapes (e.g., sustained plateau with periodic drops)
- Distinguishing between causes using correlated signals

## Key Concepts

- **`temporal_request_failure`** covers all standard client and worker operations (starts, signals, completions, heartbeats, visibility queries)
- **`temporal_long_request_failure`** covers long-poll operations (`PollWorkflowTaskQueue`, `PollActivityTaskQueue`, `PollWorkflowExecutionUpdate`, `GetWorkflowExecutionHistory` with `wait_new_event=true`)
- The server has **three independent rate limit buckets** — Execution, Visibility, and NamespaceReplicationInducing — each with its own RPS quota and priority queue
- Within each bucket, lower-priority operations are shed first under load (Priority 0 = highest, 5 = lowest)
- Not all `ResourceExhausted` errors are throttling — can also be gRPC message size violations or `BusyWorkflow` throttle

## Companion Dashboards

These dashboards are referenced throughout the knowledge base:
- https://github.com/tsurdilo/my-temporal-dockercompose
- https://github.com/tsurdilo/my-temporal-dockercompose/tree/main/deployment/grafana/dashboards
