# Temporal Metrics Troubleshooting Knowledge Base

Everything in this repository comes from my own experience — 5+ years working with Temporal across hundreds of implementations, from small services to large-scale, mission-critical systems. This is not official documentation. It's not a summary of what's written somewhere else. It's knowledge built through real production incidents, load tests, and hard-won debugging sessions.

If you're here to understand what a metric actually means when things are going sideways, you're in the right place.

---

This isn't a metrics glossary. It's a troubleshooting guide built around how problems actually present themselves in production. A few things that make it different:

- **It connects the dots.** Rather than explaining metrics in isolation, it guides you across related signals so you can build a complete picture of what's happening — not just identify a single data point.
- **It covers real impact.** Every entry is grounded in what the failure actually means for your system and your users, not just what the metric technically represents.
- **It's practical beyond incidents.** The same knowledge applies to cluster and database sizing, capacity planning, and understanding system behavior under load testing.
- **It works as a Claude skill.** The content is structured to be used directly with Claude for interactive troubleshooting.

This is a living document. It will keep growing as I work through new patterns and scenarios.

---

Useful dashboards to use alongside this knowledge base:
- https://github.com/tsurdilo/my-temporal-dockercompose
- https://github.com/tsurdilo/my-temporal-dockercompose/tree/main/deployment/grafana/dashboards

## Table of Contents

1. [Troubleshooting SDK Request Failure Metrics](./SDK_METRICS_REQUEST_FAILURES.md)