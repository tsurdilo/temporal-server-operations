## Namespace Deletion Failed

**Severity:** Warning
**Component:** worker
**Dashboard panels:** [Namespace Deletions Failed](../../../dashboards/server/temporal-server-readme.md#25-namespace-deletion) (2604), [Namespace Deletions Completed](../../../dashboards/server/temporal-server-readme.md#25-namespace-deletion) (2603), [Namespace Deletion — Executions Deleted per Second](../../../dashboards/server/temporal-server-readme.md#25-namespace-deletion) (2601)

### What this alert detects

A reclaim workflow run ended without deleting the namespace. Every run that ends writes either
`reclaim_resources_namespace_delete_failure` or `reclaim_resources_namespace_delete_success`,
never both.

### Check this first

**One point on this counter is one failed attempt, not an abandoned deletion.** A namespace
deletion gets three workflow attempts, and the first thing to establish is whether a later
attempt already finished the job:

```
sum by (namespace) (reclaim_resources_namespace_delete_success)
```

**If that shows the same namespace, the deletion completed and there is nothing to do.**

**This happens on healthy deletions.** On Elasticsearch visibility the remaining-execution
count comes from an index that lags behind the deletes themselves, so a deletion that is
working can report the same count twice, trip the no-progress check, and then be completed by
its own retry. Both deletions observed failing on a test cluster recovered this way.

### Why it matters when it is real

If no success has followed, the deletion has stopped and **nothing will restart it**. The
`DeleteNamespace` call that started it returned success long before, so no client is told. The
namespace is left renamed, marked deleted and partly emptied — it cannot serve traffic, cannot
be recovered, and its remaining executions cannot be listed.

### What to do

**Follow the playbook — it is the full procedure for this alert, and this runbook deliberately
does not repeat it:**

> **[Namespace Deletion — System Workflow Playbook](../../../../playbooks/namespace-deletion.md)**

In order:

| Step | Section |
|---|---|
| Tell a refused, a running and a stopped deletion apart | [6. Following a namespace deletion in progress](../../../../playbooks/namespace-deletion.md#6-following-a-namespace-deletion-in-progress) |
| Read why the run ended, and what the message means | [7. When a namespace deletion stops making progress](../../../../playbooks/namespace-deletion.md#7-when-a-namespace-deletion-stops-making-progress) |
| Fix the cause, then restart the deletion | [Restarting a namespace deletion](../../../../playbooks/namespace-deletion.md#restarting-a-namespace-deletion) |

### What not to do

**Do not terminate the reclaim workflow.** Terminating it undoes nothing and throws away the
retries still left on the three-attempt policy.

**Do not restart the deletion before fixing what stopped it.** A restart on an unchanged
cluster spends three more attempts reaching the same point.

### Label note

The `namespace` label is the **renamed** namespace with hyphens replaced by underscores — a
deletion of `orders-prod` appears as `orders_prod_deleted_a3f91`. Searching for the original
name matches nothing.
