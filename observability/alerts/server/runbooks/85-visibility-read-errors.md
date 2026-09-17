## Visibility Read Errors

**Severity:** Critical
**Component:** frontend (also fires for matching and worker)
**Dashboard panel:** [Visibility Read Error Rate per Store](https://github.com/tsurdilo/temporal-metrics/blob/main/observability/dashboards/server/temporal-server-readme.md) — Visibility group
**Playbook:** [Dual Visibility Operations](https://github.com/tsurdilo/temporal-metrics/blob/main/playbooks/dual-visibility.md)

### What this alert detects

`visibility_persistence_errors` above 0.1/s for 2 minutes on a visibility **read** operation — `ListWorkflowExecutions`, `CountWorkflowExecutions`, `GetWorkflowExecution`, or their CHASM equivalents — broken down by store and by the service making the call.

### Why this alert exists

Alerts **59a / 59b / 59c** all filter `service_name="history"`. History is the only service that **writes** visibility records and never reads them, so those three alerts can only ever see the write path. A visibility store that is failing **reads** fires nothing at all.

This alert deliberately omits that filter. Reads are issued by **frontend**, **worker** and **matching**.

### Why it matters

This is the user-facing visibility failure. These operations back `temporal workflow list`, `count` and `describe`, from the CLI, the Web UI and every SDK. When they fail, users see errors or timeouts on the workflow list while their workflows are running perfectly well.

**Under dual visibility there is no read fallback.** A visibility read goes to exactly one store, chosen by `system.enableReadFromSecondaryVisibility`, and if that store is failing the read fails — even when the other store holds the same records. Verified on a live cluster: `workflow list` returned `context deadline exceeded` while the secondary store held every record.

### Read the `service_name` label first

It changes the urgency:

| `service_name` | What is failing | Urgency |
|---|---|---|
| `frontend` | User queries — CLI, Web UI, SDKs | **Highest.** Users are seeing it now |
| `worker` | Background work that reads visibility: schedules, batch operations, deletion | Lower, same root cause |
| `matching` | Worker versioning only — build ID reachability, and reviving a build ID being removed from a task queue | Lowest, and only if you use worker versioning |

`matching` appearing alone is unusual and points at worker versioning rather than a general store problem.

### Triage steps

1. Read `visibility_index_name` and compare it against your persistence config to identify **which** store is failing and whether it is the primary or secondary.
2. Confirm which store reads are actually pointed at — open **Visibility Read Request Rate per Store**. By default only the primary reports reads. If the secondary is reporting, a namespace has `system.enableReadFromSecondaryVisibility` set. If **both** report, shadow read mode is on.
3. Check **Visibility Errors by Type per Store** for the `error_type`: `serviceerror_Unavailable` for a store that is unreachable or rejecting, `persistence_TimeoutError` for an Elasticsearch store not answering, `serviceerror_ResourceExhausted` for throttling.
4. If the cause is throttling, check **Visibility Rate Limit Rejections per Store**. Visibility `list*` operations have a default namespace RPS of only **10** (`frontend.namespaceRPS.visibility`), which is a common and easily-missed cause of read failures that has nothing to do with store health.
5. Check whether writes are failing too. If the write-side alerts are quiet and only reads are failing, suspect read-path rate limits or query shape rather than an unhealthy store.

### Remediation

**1. Give users a working workflow list first.** If the failing store is the one serving reads and the other store is healthy, move reads:

```yaml
system.enableReadFromSecondaryVisibility:
  - value: true
```

Dynamic config, effective within one poll interval, **no restart**. Confirm on **Visibility Read Request Rate per Store** that the line moves to the other store.

Two caveats before you do this:

- The other store only holds records written since dual writing was enabled. Anything that started **and** finished before that is not in it, so the list may be shorter than expected — with no error, just missing rows. During an outage an incomplete list is usually better than a failing one, but that is your call to make, not the server's.
- Add a `namespace` constraint to move one namespace at a time, or set it with no constraints to move the whole cluster at once.

**2. Fix the failing store.** Follow the matching scenario in the playbook.

**3. If the cause was rate limiting, not store health**, raise the read-path limits rather than touching the store: `frontend.namespaceRPS.visibility` (default 10) and `system.visibilityPersistenceMaxReadQPS`.

### Tuning this alert

0.1/s for 2 minutes. The window is deliberately shorter than the write-side alerts because this is user-facing and there is no retry behind it — a failed read is a failed user request, whereas a failed write is retried for roughly 70 minutes before anything is lost.

If you run shadow read mode continuously, note that the shadowed store's read errors are recorded under its own `visibility_index_name` even though the results are discarded and no user ever sees them. That can fire this alert for a store nobody is relying on. Shadow read is intended as a short migration test, not a permanent setting.

### Relevant dynamic config

| Key | Default | Scope | Effect |
|---|---|---|---|
| `system.enableReadFromSecondaryVisibility` | `false` | Whole cluster, or per namespace | Which store answers visibility queries |
| `system.visibilityEnableShadowReadMode` | `false` | Whole cluster | Also sends a discarded copy of every read to the other store |
| `frontend.namespaceRPS.visibility` | `10` | Per namespace | Rate limit on `list*` operations — a common cause of read failures |
| `system.visibilityPersistenceMaxReadQPS` | `9000` | Per store | Read rate limit at the persistence layer. Each store gets its own budget, and shadow reads spend the shadowed store's |
