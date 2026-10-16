# Namespace Deletion — System Workflow Playbook

**Deleting a namespace on a running cluster is a single API call** —
[`DeleteNamespace`](https://github.com/temporalio/api/blob/main/temporal/api/operatorservice/v1/service.proto),
part of the Operator service API.

**The `DeleteNamespace` call starts an internal system workflow.**
[Section 2](#2-how-the-server-runs-a-namespace-deletion) covers what that workflow runs, and where.

> **A namespace deletion cannot be paused or undone once it has started**, and it terminates
> every running execution in the namespace.
> [Section 4](#4-when-not-to-delete-a-namespace) covers when a namespace deletion is the wrong
> thing to reach for, and what to use instead.

**A namespace deletion is a process, not a single administrative action, and it can run for a
long time** — how long depends on how much the namespace holds. While it runs, it is deleting
executions against the same database and the same history service that every other namespace on
the cluster uses. And if it cannot finish, it stops — without that ever reaching the client that
made the call.

**Which is what this playbook is for: the confidence to run a namespace deletion knowing exactly
what it does and how to watch it.** How to follow a deletion while it runs and tell whether it
has finished or stopped, what to alert on so that one which gives up does not go unnoticed, what
to do when one stops making progress, and how to tell whether it is costing your production
namespaces anything — and keep it from doing so.

**This playbook also covers when a namespace deletion is the wrong thing to reach for.**
[Section 4](#4-when-not-to-delete-a-namespace) goes through those cases — a history task table
that has stopped shrinking, for example, will not be fixed by deleting the namespace, and
starting a deletion in that state adds work to a pipeline that is already struggling. The
problem itself belongs to [History Task Table Growth](./history-task-table-growth.md).

**Audience**

Operators of self-hosted Temporal clusters.

**This playbook does not cover**

- **Explicit deletion of workflow executions**, not through the `DeleteNamespace` API. Temporal
  has APIs for deleting a single execution or a batch of them.
- **Workflow execution deletion triggered by namespace retention**, which removes closed
  executions once their retention period expires.
- **The other namespace APIs.** Registering, describing and updating a namespace are separate
  operations, as is moving a global namespace between clusters —
  [Namespace Failover — Graceful Handover](./namespace-failover-graceful-handover.md).

**Dashboard**

This playbook is written against the
[Temporal Server dashboard](../observability/dashboards/server/temporal-server-readme.md#groups-and-panels),
and refers to panels by name. Every panel it uses, with the dashboard group it sits in:

| Dashboard group | Panels this playbook uses |
|---|---|
| [5. Service Requests and Errors](../observability/dashboards/server/temporal-server-readme.md#5-service-requests-and-errors) | **Server Errors by Type**<br>**Service Errors by Namespace**<br>**Service Errors by Namespace and Operation** |
| [25. Namespace Deletion](../observability/dashboards/server/temporal-server-readme.md#25-namespace-deletion) | **Namespace Deletion — Executions Deleted per Second**<br>**Namespace Deletion — Execution Deletion Failures per Second**<br>**Namespace Deletions Completed**<br>**Namespace Deletions Failed** |

**Tracking the progress of a namespace deletion is not done from the dashboard.** The
remaining count comes from a query against the deletion's own workflow rather than from a
metric — [section 6](#6-following-a-namespace-deletion-in-progress) covers it.

**Alerts**

| Alert | Fires when |
|---|---|
| [92 — Namespace Deletion Failed](../observability/alerts/server/runbooks/92-namespace-deletion-failed.md) | a reclaim workflow run ended without deleting the namespace. Documented, not in the Essential Set — [section 7](#7-when-a-namespace-deletion-stops-making-progress) covers why one firing is often self-recovering |

**Applies to**

- **All persistence stores.** The deletion works the same way on SQL and on Cassandra.
- **Both visibility stores**, with the differences called out where they exist — what happens to
  a namespace's search attributes is one, and depends on whether visibility is SQL or
  Elasticsearch.
- **Single- and multi-cluster setups.** On a multi-cluster setup, a namespace that is passive in
  the current cluster cannot be deleted there —
  [section 5](#5-four-checks-a-deletenamespace-request-must-pass) covers that along with the other refusals.

---

## Contents

1. [Starting a namespace deletion](#1-starting-a-namespace-deletion)
2. [How the server runs a namespace deletion](#2-how-the-server-runs-a-namespace-deletion)
3. [Why an OK response from the `DeleteNamespace` API does not mean the namespace deletion is done](#3-why-an-ok-response-from-the-deletenamespace-api-does-not-mean-the-namespace-deletion-is-done)
4. [When not to delete a namespace](#4-when-not-to-delete-a-namespace)
5. [Four checks a `DeleteNamespace` request must pass](#5-four-checks-a-deletenamespace-request-must-pass)
6. [Following a namespace deletion in progress](#6-following-a-namespace-deletion-in-progress)
7. [When a namespace deletion stops making progress](#7-when-a-namespace-deletion-stops-making-progress)
8. [Deleting more than one namespace at a time](#8-deleting-more-than-one-namespace-at-a-time)
9. [Every setting named in this playbook](#9-every-setting-named-in-this-playbook)

---

## 1. Starting a namespace deletion

**A namespace deletion is started either from the CLI or by calling the `DeleteNamespace` API
directly.** The CLI is the usual way, shown here against an example namespace called
`orders-prod` — every command in this playbook uses that same example name, and it should be
replaced with the namespace being deleted:

```bash
temporal operator namespace delete --namespace orders-prod
```

**which calls `DeleteNamespace` on the Operator service API.** That request has three fields, and
the CLI exposes one of them:

| Request field | Settable from the CLI? | What it is for |
|---|---|---|
| `namespace` | **yes**, `--namespace` | the namespace to delete, by name |
| `namespace_id` | no | identifies the same namespace by id instead of by name |
| `namespace_delete_delay` | no | holds the namespace record for a set period after its executions have gone, instead of removing it straight away. **Omitted from the request, the cluster default applies** |

**The server fills in `namespace_id` and `namespace_delete_delay` itself, which is why a name
on its own is enough.** `namespace_id` is resolved for you: the request must carry a name or an
id, and the very first thing the deletion does is look up whichever one is missing.
`namespace_delete_delay` falls back to the cluster default.

**The cluster default for `namespace_delete_delay` is a dynamic config setting,
`frontend.deleteNamespaceNamespaceDeleteDelay`, and it ships as `0`.** With no delay, the
namespace record is removed as soon as the last of its workflow executions has been deleted.
Raising the setting holds that final step back on every namespace the cluster deletes; putting a
value in the request field holds it back for one namespace only, and the CLI cannot send that
field.

> **Leave `namespace_delete_delay` at `0` unless something specific requires otherwise.** A
> delay does not make a deletion safer or more recoverable, and a large one set by mistake
> leaves a finished deletion sitting in a timer, looking stuck —
> [section 7](#7-when-a-namespace-deletion-stops-making-progress) covers how to tell that apart from a
> real stall, and how to clear it.
>
> **If a delay is wanted, set it cluster-wide rather than per caller.** A single cluster-wide
> setting cannot be mistyped by a client.
>
> **`namespace_delete_delay` does not delay deleting the executions.** It applies once they are
> all gone, and holds back only the removal of the namespace record.

**Calling the `DeleteNamespace` API directly means having to look up the namespace id first.**
`DescribeNamespace` is what returns it — call it with the namespace **name**, and the id comes
back on the response, in the namespace info. The CLI wraps the same call:

```bash
temporal operator namespace describe --namespace orders-prod
```

> **Note the namespace id down before starting a deletion, whichever way you start it.** The
> deletion renames the namespace, so shortly after the call the name stops resolving. The id
> never changes, and looking a namespace up by id keeps working for as long as its record
> exists.

Now that starting a deletion is covered,
[section 2](#2-how-the-server-runs-a-namespace-deletion) is what the server does once one has
started.

---

## 2. How the server runs a namespace deletion

**The `DeleteNamespace` API starts three workflows, nested one inside the next.** The three
have different names, different lifetimes, and only one of them deletes anything. Knowing which
is which is what makes a deletion possible to follow while it runs, and
[section 6](#6-following-a-namespace-deletion-in-progress) builds on it.

**All three workflows run in the `temporal-system` namespace**, not in the namespace being
deleted — that is where to look them up. The workflow ids below are what a namespace called
`orders-prod` would produce, and
[section 2.2](#22-which-workers-run-a-namespace-deletion) covers which
workers run them.

| | Workflow type | Workflow id |
|---|---|---|
| Started by the `DeleteNamespace` API | `temporal-sys-delete-namespace-workflow` | `temporal-sys-delete-namespace-workflow/orders-prod` |
| Started by the delete namespace workflow, as its child | `temporal-sys-reclaim-namespace-resources-workflow` | `temporal-sys-reclaim-namespace-resources-workflow/orders-prod-deleted-a3f91` |
| Started by the reclaim workflow, as its child | `temporal-sys-delete-executions-workflow` | `temporal-sys-delete-executions-workflow/orders-prod-deleted-a3f91` |

**A workflow id here is the workflow type, a slash, and a namespace name.** A workflow id can
therefore be worked out rather than searched for, and it is the same id every time the same
namespace is deleted. **That is what stops two deletions of one namespace running at once** —
the second start is given a workflow id that is already in use and is rejected.

**The namespace name in the workflow id is not the same for all three workflows.** The first
workflow starts before anything has changed, so it carries the original name, `orders-prod`.
Both child workflows start after step 3 below has renamed the namespace, so they carry the new
name instead: the original name, then `-deleted-`, then the first five characters of the
namespace id — `orders-prod-deleted-a3f91`. **Finding either child workflow means knowing that
renamed form** — [section 2.1](#21-what-each-stage-of-a-namespace-deletion-changes) covers the
rename and why it happens.

**Every step is carried out by activities.** The steps in order, the activities that carry each
one out, and the workflow each runs in:

| Step | Activities | Workflow |
|---|---|---|
| **1. Check that the namespace may be deleted** | `GetNamespaceInfoActivity`, `ValidateProtectedNamespacesActivity`, `ValidateNexusEndpointsActivity` | `temporal-sys-delete-namespace-workflow` |
| **2. Mark the namespace deleted** | `MarkNamespaceDeletedActivity` | `temporal-sys-delete-namespace-workflow` |
| **3. Rename the namespace** | `GenerateDeletedNamespaceNameActivity`, `RenameNamespaceActivity` | `temporal-sys-delete-namespace-workflow` |
| **4. Delete every workflow execution in the namespace** | `DeleteExecutionsActivity` | `temporal-sys-delete-executions-workflow` |
| **5. Remove the namespace record** | `DeleteNamespaceActivity` | `temporal-sys-reclaim-namespace-resources-workflow` |

**Most of the time a namespace deletion takes is spent on step 4, deleting the executions.**

**Step 4, deleting the executions, is also the only one that does not run as a single
workflow.** A workflow execution is limited in how many events its history may hold and how
large that history may grow, so
deleting millions of executions in one run would reach those limits. The delete-executions
workflow avoids that: after a fixed number of pages it closes itself and starts a fresh run in
its place, carrying its position and its running counts across — what Temporal calls
continue-as-new. **The size of a namespace therefore changes how long its deletion takes and
how many runs it takes, never whether it can finish.**

### 2.1 What each stage of a namespace deletion changes

Deleting a namespace runs the same five stages in order, and only the last of them removes the
namespace itself.

| Stage | What changes |
|---|---|
| **Validate** | nothing. Four checks run, and a failure here leaves the namespace exactly as it was — [section 5](#5-four-checks-a-deletenamespace-request-must-pass) lists them |
| **Mark deleted** | the namespace state becomes `DELETED` |
| **Rename** | the namespace is renamed to `<name>-deleted-<first five characters of its id>` — `orders-prod` becomes `orders-prod-deleted-a3f91` |
| **Delete executions** | every workflow execution in the namespace is deleted, one at a time. **Effectively all of the time goes here** |
| **Delete the namespace** | the namespace record is removed from the metadata store |

**Each stage has consequences worth knowing before a deletion is started** — how long it
takes, what else it removes, and what cannot be undone. They are grouped below.

#### Where a namespace deletion spends its time

**Removing the namespace record is a single row delete from the cluster's main database, and it
is fast.** Deleting the executions is the stage that takes the time, so a namespace that is
taking a long time to disappear is almost always still deleting executions.
[Section 6](#6-following-a-namespace-deletion-in-progress) covers how to confirm that and how far
along it has got, and [section 7](#7-when-a-namespace-deletion-stops-making-progress) covers a deletion
that has stopped making progress altogether.

**A namespace with no executions skips the delete-executions stage entirely.** The count comes
back zero, the delete-executions workflow is never started, and the deletion goes straight to
removing the namespace record.

#### What happens to custom search attributes

**On SQL visibility, the custom search attributes are deleted along with the namespace record.**
A SQL search attribute is an alias held in the namespace's own configuration, so it is removed
when that configuration is removed. **Only the namespace being deleted is affected** — every
namespace holds its own aliases, so another namespace using a search attribute of the same name
keeps it. Recreating the deleted namespace later means defining its attributes again.

**On Elasticsearch the custom search attributes are not stored on the namespace at all.** They
are registered against the index for the whole cluster, so a namespace deletion leaves them
where they are — still defined, still available to every other namespace on that index, and
still there if the namespace is recreated.

#### Why a namespace deletion cannot be undone

**Renaming the namespace is where a deletion becomes irreversible, and it exists so that the
original namespace name can be used again.** Namespace names are unique — the database enforces
it — so while a namespace exists, nothing else on the cluster can take its name. Renaming
`orders-prod` to `orders-prod-deleted-a3f91` releases `orders-prod` the moment it commits, which
is long before the deletion has finished. A replacement namespace can be created under
`orders-prod` straight away, while the namespace being deleted still has every one of its
executions to get through.

> **Keep the deleted namespace's id if a replacement is created under the original name.**
> Operations using the original name no longer reach the namespace being deleted, so following
> the deletion means knowing the renamed form — and that form is built from the id, as the
> workflow ids above show. [Section 1](#1-starting-a-namespace-deletion) recommends noting the
> id down before starting a deletion for this reason.

**Once renamed and marked deleted, a namespace cannot be brought back:**

- **Operations using the original namespace name no longer reach it.** The namespace is under its
  renamed form now, so a request naming `orders-prod` gets "not found" — and `orders-prod` is
  free for a new namespace to take.
- **`DELETED` is the end of the line for namespace state.** A namespace can move to `DELETED`, but
  nothing moves it back out — the server rejects every transition away from it.

#### What a namespace deletion does on a multi-cluster setup

**On a multi-cluster setup, deleting a namespace is a single-cluster operation.** Once the
namespace is marked deleted, changes to it are no longer sent to the other clusters — the
replication handler skips any namespace in the `DELETED` state. The namespace record on every
other cluster is therefore left exactly as it was, and a global namespace is only fully gone
once the deletion has been run on each of its clusters.

**The executions are a separate question from the namespace record.** Depending on the server
version and whether workflow-delete replication is enabled, the executions deleted on the active
cluster may also be removed on the other clusters, leaving the namespace there present but
empty.

**A namespace can only be deleted on the cluster where it is active.** Calling
`DeleteNamespace` on a cluster where the namespace is passive is refused —
[section 5](#5-four-checks-a-deletenamespace-request-must-pass) covers that with the other refusals.

> **Deleting a global namespace on every one of its clusters has a required order.**
> [Namespace Failover — Graceful
> Handover](./namespace-failover-graceful-handover.md#namespace-deletion) has the procedure, and
> the version matrix for what reaches the passive cluster.

**What each stage changes is now covered.**
[Section 2.2](#22-which-workers-run-a-namespace-deletion) covers where that work actually runs.

### 2.2 Which workers run a namespace deletion

**The three namespace deletion workflows run in the `temporal-system` namespace, not in the
namespace being deleted.** They run the way any Temporal workflow runs — on SDK workers polling
a task queue — and the cluster hosts those workers itself, inside the worker service.

| What runs | Which worker runs it | Task queue |
|---|---|---|
| The three workflows | the cluster's shared default worker | `default-worker-tq` |
| Their activities | a worker kept for namespace deletion | `temporal-sys-delete-namespace-activity-tq` |

> **Two different things are called a worker here.** The **worker service** is one of Temporal's
> four server roles, alongside frontend, history and matching — a process an operator deploys
> and scales. The **workers** it hosts are ordinary Temporal Go SDK workers, the same kind an
> application runs, except that the cluster runs them against its own `temporal-system`
> namespace. So the server is acting as its own client here.

**Every worker service instance runs a default worker and a namespace deletion activity
worker, and all of them poll the same two task queues.** There is no leader instance and no
assignment step: a cluster with four worker service instances has four default workers and four
namespace deletion activity workers, and each activity is picked up by whichever one takes it. A
single deletion's work therefore spreads across the whole worker service rather than being pinned
to one instance, and adding worker service instances adds capacity for it.

**`default-worker-tq` and `temporal-sys-delete-namespace-activity-tq` are per cluster, not per
namespace.** The worker service runs some components once per namespace and others once for the
whole cluster, and namespace deletion is the second kind. Deleting one namespace and deleting
five both go to the same two queues and the same set of workers —
[section 8](#8-deleting-more-than-one-namespace-at-a-time) is about what that means when more
than one is running.

**The deletes themselves do not go through a frontend.** The activity that deletes executions
calls the **history service** directly, one execution at a time. What does go through a
frontend is everything the SDK workers do to run the workflows at all — polling both task
queues, and recording each workflow's progress.

**Which frontend that polling lands on depends on whether the cluster has an internal
frontend.** If the static config defines an `internal-frontend` service, the worker
service's SDK client connects there. If it does not, the same polling goes to the ordinary
frontend, the one serving application traffic.

> **Run an internal frontend if the cluster does not already have one.** It keeps the polling
> that system workflows do — namespace deletion among them — off the frontend that
> applications call, so a long deletion cannot consume frontend capacity production clients
> need. It also separates frontend rate limits and metrics cleanly into cluster traffic and
> application traffic.
>
> **The `internal-frontend` service and `publicClient.hostPort` are static configuration, not
> dynamic config.** They live in the server's configuration file and are read once at startup,
> so adding an internal frontend, or pointing `publicClient.hostPort` somewhere else, means a
> configuration change and a server redeploy. Neither can be switched on while a deletion is
> running, and neither is something to reach for in the hour before one starts. **Have it in
> place beforehand, or accept for this deletion that the polling lands on the application
> frontend.**

**A deletion makes progress only while the worker service is running.** Nothing else drives it:
the frontend that accepted the call is long finished, and the deletion has no deadline of its
own. A worker service that is stopped, scaled to zero or unable to poll leaves the deletion
exactly where it is, for as long as that lasts, and reports nothing.
[Section 7](#7-when-a-namespace-deletion-stops-making-progress) starts there.

**Deleting the executions is the lowest-priority work on the database, and the server arranges
that on its own.** When the worker service builds the worker that runs these activities, it
labels that worker `preemptable`. The label travels as a header on every call the activities
make, so when the history service turns one of those calls into database work, its rate limiter
can see where the work came from. Temporal sorts database work into seven priority levels, and
`preemptable` is the bottom one, below all ordinary API traffic.

**The `preemptable` label is set in server code, not by configuration, and it does not depend on
which operation is being called.** Every database call made while deleting a namespace's
executions carries it. So production traffic wins whenever the database is at its limit: the
deletion is held back first.
[History Persistence QPS Limits](./history-persistence-qps-limits.md#31-seven-buckets-one-per-priority)
has the levels and what else lands in each.

> **Low priority protects production, but it does not slow a deletion down on its own.** With
> database capacity to spare, a deletion uses it. What one deletion asks for is bounded by its
> own settings, listed in [section 9](#9-every-setting-named-in-this-playbook); what several
> ask for together is not bounded at all, which is
> [section 8](#8-deleting-more-than-one-namespace-at-a-time).

You now know what runs, where, and in what order. **What none of that tells you is when the
operation is actually finished** — and the `DeleteNamespace` API deliberately does not wait for
it.
[Section 3](#3-why-an-ok-response-from-the-deletenamespace-api-does-not-mean-the-namespace-deletion-is-done)
covers the distance between the call returning and the deletion finishing, which misleads more
operators than anything else here.

---

## 3. Why an OK response from the `DeleteNamespace` API does not mean the namespace deletion is done

**An OK response from the `DeleteNamespace` API means the namespace has been renamed, not that
it has been deleted.** Of the five steps in
[section 2](#2-how-the-server-runs-a-namespace-deletion), the response is sent once step 3,
renaming the namespace, has finished: the namespace has been checked, marked `DELETED`, and
renamed. **Step 4, deleting every workflow execution, has not started. Step 5, removing the
namespace record, is further off still.**

**The `DeleteNamespace` response carries one field, and it is the renamed namespace:**

```
DeleteNamespaceResponse{
  deleted_namespace: "orders-prod-deleted-a3f91"
}
```

**The `deleted_namespace` value is the new name the rename step gave the namespace.** The API
definition calls it a temporary name used during the reclaim step — the step that has only just
started when the response is sent. Nothing in the response says anything about executions,
because none have been deleted yet.

**The namespace id is unchanged.** Renaming changes the name and nothing else, so the id still
identifies the same namespace, and the name in this response is the one built from it.

> **This is the response to keep, if the namespace id was not written down already.** It carries
> the only name the namespace now answers to.
> [Section 1](#1-starting-a-namespace-deletion) recommends noting the id down before starting a
> deletion for exactly this reason — with the id, the renamed form can be worked out at any
> time; without either, finding the deletion again means searching `temporal-system` for it.

**`DeleteNamespace` responds after the rename because it waits on the delete namespace
workflow, and that workflow is built to finish early.** The delete namespace workflow carries
out the three steps it owns — checking that the namespace may be deleted, marking it deleted,
renaming it — then starts the reclaim workflow as a child. That child is started in
**abandoned** mode, meaning it is deliberately not tied to its parent and keeps running after
the parent completes. The parent waits only for confirmation that the child has started, then
completes, and its completion is what the API returns on — never the child's.

| Done when `DeleteNamespace` responds | Not done when `DeleteNamespace` responds |
|---|---|
| The namespace passed every check in [section 5](#5-four-checks-a-deletenamespace-request-must-pass) | No workflow execution has been deleted |
| The namespace state is `DELETED` | The namespace record is still in the database |
| It has been renamed, and the original name is free | The reclaim workflow has done no work beyond starting |

**There is no continuous progress signal once the `DeleteNamespace` API has responded.** A
deletion that later slows down, runs out of attempts or stops entirely does so with nobody to
tell. So an OK response is worth exactly one conclusion — a deletion started — and whether it
is still progressing, has finished, or has stopped is read from the two child workflows under
the renamed namespace.
[Section 6](#6-following-a-namespace-deletion-in-progress) covers how to read them.

**What an OK response from `DeleteNamespace` means, and what it does not, is now covered.**
Whether to start a namespace deletion at all is a separate question.
[Section 4](#4-when-not-to-delete-a-namespace) covers the situations where a namespace deletion
is the wrong tool for the problem, and what to reach for instead in each.

---

## 4. When not to delete a namespace

**A namespace deletion is the right tool for one job: removing a namespace nobody needs any
more.** Three other situations can look like a reason to delete one, and in each it is the
wrong tool — in the first two, using it leaves the original problem worse than before.

#### 1. Deleting a namespace to fix a history task table that has stopped shrinking

**Deleting the namespace looks like a clean way out, and the reasoning behind it is sound as
far as it goes.** When a namespace no longer exists, the history service drops every task
belonging to it: the task cannot resolve its namespace, the server treats that as safe to
ignore, acknowledges the task, and the deletion watermark finally moves past it. Remove the
namespace and the tasks holding cleanup up have nothing left to act on, so the rows start
being deleted again.

**The server only drops a namespace's tasks once the namespace record has been removed from
the database altogether**, and that removal is the last step of a namespace deletion — it runs
only after every workflow execution in the namespace has already been deleted.

**The rename step of a namespace deletion does not bring that moment forward, because it
renames and nothing more.** Renaming is step 3 of the deletion, not a separate operation on the
namespace: the record stays exactly where it was and the namespace id does not change. Tasks
are keyed by namespace id rather than by name, so a renamed namespace that is marked deleted
still resolves for every one of its tasks, and those tasks are refused exactly as they were
before.

**Deleting every workflow execution in the namespace needs the same task processing that has
already stopped.** The deletion's own database work runs at the lowest priority the server
has, which puts it first in line to be refused on a cluster that is already refusing work. It
gives up once the count of remaining executions stops falling between attempts, and the server
names a task processor that has stopped as one of the two causes of exactly that.

**A namespace deletion delivers none of the hoped-for benefit until the whole process has
finished, and one started in this state may never finish.** For as long as it runs without
completing, the task table is exactly as it was. Meanwhile the namespace was marked deleted and
renamed in the first seconds, before a single execution was touched — so a deletion that never
completes leaves the original problem untouched and the namespace unrecoverable.

> **Get the task rows deleting again before reaching for a namespace deletion.**
> [History Task Table Growth section 6](./history-task-table-growth.md#6-why-a-deleted-namespace-keeps-its-tasks-alive)
> covers this exact case, and
> [its section 6.4](./history-task-table-growth.md#64-what-to-do-instead) covers what to do
> instead, including what to do when a namespace deletion is already running. Where the limit
> is task processing itself rather than one namespace,
> [History Task Processing Tuning](./history-task-processing-tuning.md) covers getting it
> moving again.

#### 2. Deleting a namespace to stop workflows that are running

**A namespace deletion does terminate every running execution, and termination is not
cancellation.** A cancellation is delivered into the workflow: it lands in the history, workflow
code can see it, and cleanup or compensation logic gets a chance to run. A termination closes
the execution from the outside and delivers nothing. No workflow code runs in response, no
compensating activity fires, and nothing downstream is told.

**The terminations are not written to visibility either.** When the server terminates an
execution as a step in deleting it, it deliberately skips the close record that would normally
mark that execution as Terminated — there is no point writing a record that is about to be
removed. So the executions do not turn up in visibility as terminated and then disappear; they
only disappear. Searching afterwards finds nothing, rather than a list of what was stopped.

**A namespace deletion takes every execution in the namespace, whatever its status.** There is
no option to limit it to the closed ones: the server has a flag for skipping open executions
and namespace deletion does not set it, so running executions are terminated and deleted along
with the rest.

> **To remove closed executions and keep the namespace, use the workflow delete API.**
> `DeleteWorkflowExecution` deletes one execution by id, and a batch operation deletes every
> execution matching a query — closed ones only, if the query says so. Both leave the namespace
> and everything still running untouched. This is the tool for a history table that has grown
> large under a long retention period, which is a problem a namespace deletion solves only by
> destroying the namespace along with it.
>
> **Shortening the namespace's retention period does not clear executions that have already
> closed.** Each execution's cleanup is scheduled when it closes, from the retention period in
> force at that moment, so a shorter period applies to executions that close after the change
> and to no others. Lowering retention is worth doing to stop the problem growing; it will not
> undo it.

> **To stop workflows and keep what they did, terminate or cancel them instead.** A batch
> operation terminates or cancels every execution matching a query, leaves the namespace in
> place, and leaves the histories intact to look at afterwards. A namespace deletion deletes
> those histories within moments of terminating the executions, so nothing is left to inspect
> once the incident is over.

#### 3. Deleting a namespace before the decision is final

**A namespace deletion cannot be undone once it has started.** The namespace is marked deleted
and renamed within seconds, before a single execution is touched, and neither step reverses —
[section 2.1](#21-what-each-stage-of-a-namespace-deletion-changes) covers why.

**A namespace deletion has no grace period.** `namespace_delete_delay` holds back only the
removal of the namespace record, and it starts after every execution has already been deleted —
[section 1](#1-starting-a-namespace-deletion) covers the setting.

> **Settle what the goal actually is before starting a namespace deletion.** Cases 1 and 2 come
> down to the same mistake: treating a namespace deletion as a way to clear up a problem inside
> the namespace. It is not — it removes the namespace, and everything inside it goes with it.
> **Start one only when the namespace itself is no longer wanted and everything in it can go.**
>
> **For the namespaces where the answer is always no, put that in configuration rather than in
> a convention.** `worker.protectedNamespaces` is a list of namespace names the server will
> refuse to delete, so a mistaken call against a production namespace fails instead of
> succeeding. It is the only control in this playbook that prevents a wrong deletion rather
> than explaining one afterwards —
> [section 5](#5-four-checks-a-deletenamespace-request-must-pass) covers it.

**All three cases above are decisions for whoever runs the cluster.** Some namespaces the
server refuses to delete at all, whatever the operator has decided.
[Section 5](#5-four-checks-a-deletenamespace-request-must-pass) covers the four checks a `DeleteNamespace`
request has to pass before a deletion starts, and what each one rejects.

---

## 5. Four checks a `DeleteNamespace` request must pass

**Four checks run before a namespace deletion changes anything, and failing any one of them
ends the request.** All four run inside step 1, checking that the namespace may be deleted —
the first row of the step table in
[section 2](#2-how-the-server-runs-a-namespace-deletion) — so they finish before the namespace
is marked deleted and before it is renamed.

**A refused `DeleteNamespace` request gets an error back, not an OK response.** The
`DeleteNamespace` API hands the failed precondition straight to the caller, the delete namespace
workflow ends as failed, and no child workflow is ever started. The namespace is exactly as it
was — nothing has to be undone, because nothing was done. That also makes a refusal easy to tell
apart from a deletion that started and then stopped, which [section
6](#6-following-a-namespace-deletion-in-progress) covers.

**Each check is a rule about which namespaces may be deleted at all.** Two are fixed in the
server, and two are decided by cluster configuration:

| The rule being enforced | The error message the caller gets back | What to do about it |
|---|---|---|
| **Temporal's own `temporal-system` namespace cannot be deleted**, by anyone, under any configuration | `unable to delete system namespace` | nothing — the rule is fixed in the server and has no setting |
| **A namespace can only be deleted on the cluster where it is active**, because executions keep replicating into a passive cluster and a deletion there could never finish | `namespace <name> is passive in current cluster <cluster>: remove cluster <cluster> from cluster list or make namespace active in this cluster and retry` | run the deletion on the active cluster, or take the receiving cluster out of the namespace's cluster list first. [Namespace Failover — Graceful Handover](./namespace-failover-graceful-handover.md#namespace-deletion) has the full procedure |
| **A namespace named in `worker.protectedNamespaces` cannot be deleted** | `namespace <name> is protected from deletion` | remove the name from `worker.protectedNamespaces`, a dynamic config setting that takes effect without a restart |
| **A namespace that a Nexus endpoint points at cannot be deleted** | `cannot delete a namespace that is a target of a Nexus endpoint <endpoint>` | delete or repoint the named endpoint, or set the dynamic config setting `frontend.allowDeleteNamespaceIfNexusEndpointTarget` to `true`, which lets a deletion go ahead despite the endpoint |

**A refused `DeleteNamespace` request is counted in `service_error_with_type`**, emitted by
the frontend with two tags worth knowing:

| Tag | Value for a refused namespace deletion |
|---|---|
| `operation` | `OperatorDeleteNamespace` |
| `error_type` | `serviceerror_FailedPrecondition`, for all four checks — note the underscore, not a dot |

**The `Operator` prefix on the operation tag is easy to miss.** Temporal prepends `Operator` to
the operation tag of every Operator service API, so a query written against
`operation="DeleteNamespace"` matches nothing at all.

**Refusals are not counted in `service_errors`.** The server classifies a failed precondition
as an expected error — caused by the request rather than by the cluster — and `service_errors`
counts only the unexpected ones. Anything built on `service_errors` will not see a refused
namespace deletion.

**`service_error_with_type` and `service_errors` are both already on the server dashboard**,
in the
[Service Requests and Errors](../observability/dashboards/server/temporal-server-readme.md#5-service-requests-and-errors)
row:

| Panel | What it shows for a refused namespace deletion |
|---|---|
| **Server Errors by Type** | a `serviceerror_FailedPrecondition` series. The panel groups by error type only, so it confirms a refusal happened without saying which namespace or which API |
| **Service Errors by Namespace** and **Service Errors by Namespace and Operation** | nothing — both are built on `service_errors`, which refusals never reach |

#### Protected namespaces are the one check an operator sets up in advance

**`worker.protectedNamespaces` is a list of namespace names that cannot be deleted**, and it is
empty by default. Adding a name to it makes every `DeleteNamespace` request for that namespace
fail, including one sent by mistake and one sent by automation.

**The list holds namespace names, matched exactly.** There is no pattern matching and no way
to protect a namespace by its id, so each name has to be written as it is registered.

**A name can be protected before the namespace exists, and protecting it does not reserve it.**
The setting is read only when a deletion is validated — nothing consults it when a namespace is
registered. So a name already on the list can still be used to create a namespace, and that
namespace is then undeletable until the name comes off the list again.

> **Put production namespaces in `worker.protectedNamespaces`.** It is the only guard in this
> playbook that stops a mistaken deletion before it starts, rather than explaining the damage
> afterwards. The setting is dynamic config, so it can be changed without a restart — which
> also means a deliberate deletion only needs the name taken out of the list first.

**Passing all four checks is the last moment at which the namespace is still untouched.**
The rest of the deletion runs asynchronously, after the OK response has already gone back to
the caller.
[Section 6](#6-following-a-namespace-deletion-in-progress) covers where to look: how to find the
deletion, how many executions it has left, how fast it is working through them, and how to
tell one that is still going from one that has stopped.

---

## 6. Following a namespace deletion in progress

**Once a `DeleteNamespace` request has passed its checks and the OK response has gone back,
the rest of the namespace deletion runs asynchronously on the server.** Nothing further is
reported to the caller, so following it means reading the two child workflows in the
`temporal-system` namespace, named in [section 2](#2-how-the-server-runs-a-namespace-deletion).

> **The visibility API cannot be used to watch a namespace deletion.** The rename in step 3
> leaves the namespace with two names an operator might try, and neither works for listing or
> counting executions:
>
> - **The renamed form**, `orders-prod-deleted-a3f91`, is refused outright and stays refused:
>   `Namespace has invalid state: Deleted. Must be Registered or Deprecated.`
> - **The original name**, `orders-prod`, returns `Namespace orders-prod is not found.` once
>   the namespace registry has refreshed, which takes about two seconds.
>
> **For those first couple of seconds the original name does still answer**, from a registry
> cache that has not caught up yet, and the count it gives back is real. It stops working
> moments later, so it is not something to follow a deletion with. Everything below reads the
> system workflows instead.

#### Which state a namespace deletion is in

**One command lists all three workflows**, and it takes the namespace name as it was typed —
no need to work out the renamed form, because the renamed namespace starts with the original
name. `orders-prod` is the example name again, to be replaced with the namespace being
deleted:

```bash
temporal workflow list --namespace temporal-system \
  --query "WorkflowId STARTS_WITH 'temporal-sys-delete-namespace-workflow/orders-prod' \
        OR WorkflowId STARTS_WITH 'temporal-sys-reclaim-namespace-resources-workflow/orders-prod' \
        OR WorkflowId STARTS_WITH 'temporal-sys-delete-executions-workflow/orders-prod'"
```

**Three outcomes are distinguishable from that list of workflows**, and they are
distinguishable because a refused request never starts a child at all:

| What the workflows show | What it means |
|---|---|
| Delete namespace workflow **failed**, no child workflow exists | a check refused the request — [section 5](#5-four-checks-a-deletenamespace-request-must-pass). The namespace was never touched |
| Delete namespace workflow **completed**, children **running** | the deletion is under way |
| Delete namespace workflow **completed**, a child **failed** | the deletion started and then stopped — [section 7](#7-when-a-namespace-deletion-stops-making-progress) |

**The delete namespace workflow completes within seconds in all three cases**, so its own
state says only whether the deletion was accepted. Progress lives in the children.

**To read why a workflow ended as it did, show its history:**

```bash
temporal workflow show --namespace temporal-system \
  --workflow-id "temporal-sys-reclaim-namespace-resources-workflow/orders-prod-deleted-a3f91"
```

**A failed workflow carries its reason in the `WorkflowExecutionFailed` event**, and
[section 7](#7-when-a-namespace-deletion-stops-making-progress) covers what each reason means.

#### Reading how much is left

**The delete-executions workflow answers a `stats` query**, and it is the only place that
gives a number:

```bash
temporal workflow query \
  --namespace temporal-system \
  --workflow-id "temporal-sys-delete-executions-workflow/<namespace>-deleted-<5 hex>" \
  --type stats
```

**A reply from a namespace of 3000 executions:**

```json
{"TotalExecutionsCount":3000,"SuccessCount":3000,"ErrorCount":0,
 "RemainingExecutionsCount":0,"AverageRPS":333,"ContinueAsNewCount":0,
 "StartTime":"2026-10-16T14:44:19Z","ApproximateEndTime":"2026-10-16T14:44:29Z",
 "ApproximateTimeLeft":0}
```

| Field | What to read from it |
|---|---|
| `RemainingExecutionsCount` | how much is left. **Falling means progress; unchanged between readings is how a stuck deletion shows itself** |
| `AverageRPS` | how fast executions are being deleted, averaged over the run |
| `ApproximateTimeLeft` and `ApproximateEndTime` | an estimate, derived from the deletion rate so far |
| `ErrorCount` | executions the deletion failed to remove |
| `ContinueAsNewCount` | how many times the workflow has rolled over, which rises with namespace size |

**Two `stats` readings a minute apart are worth more than any single one.** A falling
`RemainingExecutionsCount` settles the question on its own, and a count that has not moved is
the same condition the server itself watches for.

#### Nothing happens for the first few seconds, by design

**A deletion waits before it deletes its first execution.** The reclaim workflow sleeps for the
namespace cache refresh interval plus two seconds, so that no new executions can be created in
a namespace already marked deleted. On a cluster running the default
`system.namespaceCacheRefreshInterval` of two seconds, that is a pause of about four seconds.

**A deletion that appears to be doing nothing in its first seconds is therefore normal.**
Judging progress needs a reading taken after the pause, not during it.

#### What the server metrics show

**Four of these counters are on the
[Namespace Deletion](../observability/dashboards/server/temporal-server-readme.md#25-namespace-deletion)
row of the server dashboard.**

**A namespace deletion emits two families of counter, and they arrive at different times.**
Which one to use depends on whether the question is "is it moving" or "did it finish":

| Metric | Panel | When it is recorded |
|---|---|---|
| `delete_executions_success` | **Namespace Deletion — Executions Deleted per Second** | **once per execution, as the deletion runs.** The only counter to move while a deletion is still working |
| `delete_executions_failure` | **Namespace Deletion — Execution Deletion Failures per Second** | once per execution the deletion could not remove |
| `reclaim_resources_delete_executions_success` | none | **once, at the end**, carrying the total the run deleted |
| `reclaim_resources_delete_executions_failure` | none | once, at the end, carrying the total that errored |
| `reclaim_resources_namespace_delete_success` | **Namespace Deletions Completed** | `1` when the namespace record has been removed |
| `reclaim_resources_namespace_delete_failure` | **Namespace Deletions Failed** | `1` when the reclaim workflow ended **without** deleting the namespace |

**The `reclaim_resources_` family says nothing until a deletion ends.** All four are written
from a deferred block when the reclaim workflow completes, so a deletion that is halfway
through has emitted none of them. Watching those for progress shows a flat line on a perfectly
healthy deletion.

> **The `namespace` label is not the name that was typed.** Every counter above carries the
> **renamed** namespace, and Prometheus replaces hyphens with underscores — so deleting
> `orders-prod` produces `namespace="orders_prod_deleted_a3f91"`. A query for either
> `orders-prod` or `orders-prod-deleted-a3f91` matches nothing.

**Every reclaim workflow run that ends writes exactly one of
`reclaim_resources_namespace_delete_success` or
`reclaim_resources_namespace_delete_failure`**, which makes the failure counter the one to
alert on.

**This section has covered how to track a namespace deletion that passed its checks and is
running.** [Section 7](#7-when-a-namespace-deletion-stops-making-progress) covers one that has stopped:
what makes a deletion stop, what state it leaves the namespace in, and how to get it moving
again.

---

## 7. When a namespace deletion stops making progress

**A namespace deletion stops when its reclaim workflow runs out of retry attempts.** The reclaim
workflow is the one that owns everything after the rename — it starts the delete-executions
workflow, waits for every execution in the namespace to be gone, and then removes the
namespace record, which is steps 4 and 5 of the five in
[section 2](#2-how-the-server-runs-a-namespace-deletion).

**The delete namespace workflow starts the reclaim workflow with a retry policy of three
attempts.** If a reclaim run fails, Temporal starts a fresh run of it a minute later, and the
third failure ends the deletion. Nothing restarts it afterwards.

**Neither the reclaim workflow's attempt count nor its retry interval is configurable.** Both are
fixed in the server, so a deletion that keeps failing cannot be given more attempts — it has to be
restarted, which [restarting a namespace deletion](#restarting-a-namespace-deletion) covers.

#### Why a namespace deletion stops

**A namespace deletion finds what to delete by listing the namespace's executions from
visibility**, one page at a time, and deleting each one. Visibility is eventually consistent,
so a pass can run to the end of the list without having seen everything — an execution that
closed moments before the page was read may not be in the index yet.

**A completed pass is therefore not taken as proof the namespace is empty, and the reclaim
workflow checks for itself once the delete-executions workflow reports back.** Counting the
executions still in the namespace is an activity in its own right, and it retries while any
remain. What it counts decides whether the deletion carries on, retries, or ends:

**Each outcome is reported as an activity error**, which appears in the reclaim workflow's
history — printed by the `temporal workflow show` command above — and in the worker service
logs alongside a `Some workflow executions still exist.` or `No progress was made.` line:

| Error text | What happens next |
|---|---|
| `<n> executions are still exist` | **the counting activity retries, inside the same reclaim run.** Executions remain and their number is still falling — normal on a large namespace |
| `no progress was made: <n> executions are still exist` | **the reclaim run fails**, because the number of remaining executions has not changed since the previous attempt. The reclaim workflow's retry starts a fresh run |
| `<n> not deleted executions are still exist` | **the reclaim run fails.** Executions were asked to go and did not |

**Only the no-progress and not-deleted errors also appear as the reclaim workflow's
`WorkflowExecutionFailed` message.** An `executions are still exist` error stays an activity
error inside a run that is still going.

**An activity failure is what ends a reclaim run, and the workflow arranges for it to be
retried.** The counting activity does not run out of retries — it returns a **non-retryable**
error, which stops it there and then. The reclaim workflow catches that error and deliberately
re-raises it as a **retryable** one so the server will retry the whole run. Without that
conversion the run would simply fail, and the three-attempt policy would never come into play.

**The worker service writes four log lines as a deletion struggles, at three levels:**

| Log line | Level | Carries |
|---|---|---|
| `Some workflow executions still exist.` | warn | the remaining count, as `counter` |
| `No progress was made.` | warn | the attempt number and the remaining count |
| `Activity error.` | error | the typed error message, the workflow id and the run id |
| `Unable to delete workflow executions.` | **info** | the remaining count |

> **A cluster logging at warn or above sees the first three and not the fourth.** Measured on
> a test cluster: the two warnings and the error were all present for a failed deletion, and
> the info line was absent. Do not build a log search on
> `Unable to delete workflow executions.` without checking the cluster's log level first.

**A fresh reclaim run does not start the deletion over.** It begins by counting the executions
still in the namespace and finishes immediately if that count is zero, so a retry works
through the remainder rather than repeating what is already done. It does **not** re-run the
four checks in [section 5](#5-four-checks-a-deletenamespace-request-must-pass) — those belong
to the delete namespace workflow, which completed long before.

**The counting activity and the reclaim workflow each have their own attempt count:**

| | Retries | How many |
|---|---|---|
| The counting activity | while the number of remaining executions keeps falling | **no limit.** From its **eighth** attempt onwards it also compares the remaining count with the previous attempt's, and ends the run when both are equal |
| The reclaim workflow | when one of its runs ends in failure | **three runs** |

**The counting activity backs off from one second to a two-minute ceiling**, bounded only by a
ten-hour schedule-to-close timeout. A reclaim run can sit in it for hours while executions are
still being removed, and still be working normally.

**A reclaim run is slow to declare a namespace deletion stopped.** The counting activity has to
get as far as its eighth attempt and find the remaining count unchanged from the attempt before.
Only then does the reclaim run end, and only then is one of the three workflow attempts used.

**No workflow timeout caps a reclaim run.** None of the three workflows sets a run or an execution
timeout, so the ten-hour activity timeout is the only limit there is. A namespace deletion
that is going to fail can therefore take three runs of up to ten hours each — well over a day
— before it ends and stops retrying.

**A namespace deletion that is merely slow never ends on its own at all.** While the remaining
count keeps falling, however slowly, the no-progress check does not fire, nothing fails, and
the deletion carries on. **"Running for a long time" and "failing" look the same from the
outside, and the `stats` query in
[section 6](#6-following-a-namespace-deletion-in-progress) is what tells them apart.**

**A reclaim run that ends without deleting the namespace is what
[alert 92](../observability/alerts/server/runbooks/92-namespace-deletion-failed.md) fires on**,
and it shows on the **Namespace Deletions Failed** panel.

> **A stopped deletion is not always a broken one.** On Elasticsearch visibility the remaining
> count comes from an index that lags behind the deletes themselves, so a healthy deletion can
> report the same count twice and trip the no-progress check. Measured on a test cluster: a
> deletion of 8000 executions failed with `no progress was made: 616 executions are still
> exist`, and the workflow's own retry then completed it successfully, deleting the remaining
> 616. **Check whether a retry has already recovered it before intervening.**

#### What a stopped namespace deletion leaves behind

**Everything the deletion had already done stays done.** A deletion that gives up part way
leaves the namespace in the state it reached, and none of it rolls back:

| | State after a deletion stops |
|---|---|
| The namespace record | still in the database, still marked `DELETED` |
| The namespace name | still the renamed form; the original name is still free for reuse |
| Executions already deleted | gone, permanently |
| Executions not yet reached | still there, inside a namespace that cannot be queried or used |
| The namespace id | unchanged, and still the way to address what is left |

**A half-deleted namespace is neither usable nor recoverable.** It cannot serve traffic,
it cannot be renamed back, and [section 2.1](#21-what-each-stage-of-a-namespace-deletion-changes)
covers why neither step reverses. The only way forward is to finish the deletion.

#### Restarting a namespace deletion

**A stopped deletion is restarted by calling `DeleteNamespace` again**, against the renamed
namespace rather than the original name:

```bash
temporal operator namespace delete --namespace orders-prod-deleted-a3f91
```

**Restarting a namespace deletion is safe to repeat.** The rename step recognises a name that
already ends in its own `-deleted-` suffix and returns it unchanged, so a restart does not rename
the namespace a second time. The reclaim workflow is started with a policy that terminates an
existing run of itself, so a restart also cannot leave two reclaim workflows working on the same
namespace.

**A restarted namespace deletion picks up what is left rather than starting over.**
Executions already deleted are gone, so the restarted deletion works through the remainder.

> **Do not terminate a reclaim workflow that is still running.** Terminating it undoes
> nothing — the namespace stays renamed and marked deleted, and the executions already removed
> stay removed. What it does cost is the remaining retries: a terminated workflow is not
> retried, so the attempts left on the three-attempt policy are thrown away. Restarting the
> deletion, as above, is the way to intervene.

> **Fix the cause before restarting, or the restart stops in the same place.** A deletion that
> ended on no progress did so because the executions were not being removed, and
> [section 4](#4-when-not-to-delete-a-namespace) covers the condition that most often causes
> that. A restart on an unchanged cluster spends three more attempts reaching the same point.

#### A delay that looks like a stall

**A deletion can also sit doing nothing because it was told to.** With
`namespace_delete_delay` set, the reclaim workflow sleeps after every execution has been
deleted and before the namespace record is removed — so the workflow is running, healthy, and
deliberately idle. [Section 1](#1-starting-a-namespace-deletion) covers the setting.

**A `stats` query tells a delay apart from a stall.** A `RemainingExecutionsCount` of zero
with the reclaim workflow still running is a delay timer, not a stall.

**The delay on a running deletion can be changed or removed without restarting it:**

```bash
temporal workflow update execute \
  --namespace temporal-system \
  --name update_namespace_delete_delay \
  --workflow-id "temporal-sys-reclaim-namespace-resources-workflow/orders-prod-deleted-a3f91" \
  --input '"0"'
```

**An input of `"0"` removes the delay** and lets the deletion finish immediately. Any other
duration replaces the remaining wait, counted from the moment the update is accepted rather
than from when the deletion started.

**This section has covered a namespace deletion that stopped, and how to finish it.**
[Section 8](#8-deleting-more-than-one-namespace-at-a-time) covers what changes when several
namespace deletions run at once.

---

## 8. Deleting more than one namespace at a time

**Nothing prevents several namespace deletions running at once.** Each one gets its own set of
three workflows, keyed on its own namespace, so deleting five namespaces starts five
independent workflow trees that neither wait for nor notice each other.

#### What running several namespace deletions at once costs

**Each deletion works at its own full rate, and the load on the database adds up.** Measured on
a test cluster at server defaults, deleting two namespaces of 6000 executions at the same time:

| | Executions | Rate reported by `stats` | Elapsed |
|---|---|---|---|
| First namespace | 6000 | 333/s | 18.3s |
| Second namespace | 6000 | 333/s | 18.3s |
| **Together** | **12000** | **656/s against the cluster** | 18.3s |

**Neither deletion slowed its neighbour down.** A single deletion of 3000 executions on the same
cluster also reported 333/s, so running two did not divide a budget between them — it doubled the
total.

**Two settings multiplied together decide how fast one deletion works.**
`frontend.deleteNamespaceDeleteActivityRPS` is the rate per parallel activity, default
**100**, and `frontend.deleteNamespaceConcurrentDeleteExecutionsActivities` is how many run in
parallel, default **4** — a ceiling of **400 executions a second per deletion**. The measured
333/s is that ceiling less overhead.

**So the arithmetic for N deletions is N times that.** Three concurrent deletions ask for up to
1200 executions a second, five for 2000, and nothing in the deletion path caps the sum.

#### What the deletions share, and what they do not

| | Shared between concurrent deletions? |
|---|---|
| The per-deletion rate settings | **no** — each deletion gets its own `RPS x concurrency` budget |
| The activity worker pool and its task queue | **yes** — one set per worker service instance, all polling `temporal-sys-delete-namespace-activity-tq` |
| Database capacity | **yes**, and the combined load lands here |

**Worker capacity is the one part that scales by adding hardware.** Every worker service
instance runs its own delete-namespace activity worker, as
[section 2.2](#22-which-workers-run-a-namespace-deletion) covers, so more instances means more
workers polling the same queue. Database capacity does not scale that way.

**The deletes run at the lowest database priority, which protects production traffic without
limiting the deletions.** A database at its limit sheds their work before anything else. On a
cluster with spare capacity, as in the measurement above, nothing holds the deletions back and
the full combined rate arrives.

#### Recommendations

> **Delete namespaces one at a time unless there is a reason not to.** The deletions do not
> help each other finish sooner — each runs at its own rate regardless — so running several
> concurrently buys nothing except a larger total load on the database.

> **If several must run at once, cap the group rather than each deletion.**
> `worker.deleteNamespaceActivityLimitsConfig` applies to the shared activity worker and is
> therefore the only setting that bounds all deletions together.
> `frontend.deleteNamespaceDeleteActivityRPS` only slows each deletion individually, which on
> its own does nothing to bound five of them. [Section 9](#9-every-setting-named-in-this-playbook)
> lists both.

> **Start every namespace deletion by name, not by id.** The CLI only accepts a name, so
> using it avoids this entirely — the situation can only arise from a direct API call.
> A `DeleteNamespace` request carrying a namespace id and no name is given a workflow id with
> an empty name, `temporal-sys-delete-namespace-workflow/`, and that id is identical for every
> such request. Two of them overlapping therefore collide on one workflow id, even though they
> are deleting different namespaces.

**This section has covered running several namespace deletions at once.**
[Section 9](#9-every-setting-named-in-this-playbook) lists every setting this playbook names,
with its default and what it changes.

---

## 9. Every setting named in this playbook

**Most of these settings carry a `frontend.` prefix and are read by the worker service.** The
prefix is historical and does not say which service uses the setting — Temporal's own
description of the rate setting says so outright. All of them are dynamic config.

#### How hard a namespace deletion works

| Setting | Default | What it changes | Takes effect |
|---|---|---|---|
| `frontend.deleteNamespaceDeleteActivityRPS` | `100` | executions deleted per second, per parallel activity | **while a deletion is running** |
| `frontend.deleteNamespaceConcurrentDeleteExecutionsActivities` | `4` | how many delete activities run in parallel. Must not exceed 256, or the number of worker cores | read once, when the deletion starts |
| `frontend.deleteNamespaceDeletePageSize` | `1000` | executions read from visibility per page | read once, when the deletion starts |
| `frontend.deleteNamespacePagesPerExecution` | `256` | pages worked through before the delete-executions workflow continues as new | read once, when the deletion starts |
| `worker.deleteNamespaceActivityLimitsConfig` | empty | limits on the shared activity worker: `MaxConcurrentActivityExecutionSize`, `TaskQueueActivitiesPerSecond`, `WorkerActivitiesPerSecond`, `MaxConcurrentActivityTaskPollers` | worker service restart |

**`deleteNamespaceDeleteActivityRPS` and `deleteNamespaceConcurrentDeleteExecutionsActivities`
multiply.** `RPS x concurrency` is the rate one deletion asks for — **400
executions a second on defaults**, as [section 8](#8-deleting-more-than-one-namespace-at-a-time)
measures.

**Only `worker.deleteNamespaceActivityLimitsConfig` bounds several deletions together.** Every
other setting here applies per deletion, so lowering one of them does nothing about the total
when more than one is running.

#### Whether a namespace deletion may start at all

| Setting | Default | What it changes |
|---|---|---|
| `worker.protectedNamespaces` | empty | namespace names, matched exactly, that the server refuses to delete |
| `frontend.allowDeleteNamespaceIfNexusEndpointTarget` | `false` | set `true` to allow deleting a namespace a Nexus endpoint points at |

[Section 5](#5-four-checks-a-deletenamespace-request-must-pass) covers both, along with the two
refusals that no setting controls.

#### What happens at the end of a namespace deletion

| Setting | Default | What it changes |
|---|---|---|
| `frontend.deleteNamespaceNamespaceDeleteDelay` | `0` | how long the namespace record is kept after every execution has been deleted |

**Leave the delay at `0`** — [section 1](#1-starting-a-namespace-deletion) covers why, and
[section 7](#7-when-a-namespace-deletion-stops-making-progress) covers clearing one that is
already set on a running deletion.

#### Fixed in the server, with no setting

| Behaviour | Value |
|---|---|
| Reclaim workflow retry attempts | **3**, first retry after 60 seconds |
| Counting activity retries | **no limit**; one second backing off to a two-minute ceiling |
| Counting activity timeout | **10 hours** schedule-to-close |
| When the no-progress check starts applying | the activity's **eighth** attempt |
| Renamed namespace format | the original name, `-deleted-`, and five characters taken from the namespace id |

#### The namespace cache refresh interval

| Setting | Default | Why it matters here |
|---|---|---|
| `system.namespaceCacheRefreshInterval` | `2s` | a deletion waits for one refresh interval plus two seconds before deleting its first execution, so that no new executions can be created. [Section 6](#6-following-a-namespace-deletion-in-progress) covers the pause |

#### Static configuration, which needs a redeploy

| Setting | What it changes |
|---|---|
| an `internal-frontend` service defined in the server's config file | the system workers poll the internal frontend, keeping their traffic off the frontend applications call |
| `publicClient.hostPort` | overrides that choice and sends the polling to whatever address it names |

**The internal frontend and `publicClient.hostPort` are read once at startup rather than from
dynamic config**, so changing either means editing the configuration file and redeploying —
[section 2.2](#22-which-workers-run-a-namespace-deletion) covers what they affect.
