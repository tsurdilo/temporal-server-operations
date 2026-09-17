# Dual Visibility Operations Playbook

Dual visibility writes every visibility record to two stores in parallel. Its
primary purpose is moving a running cluster from one visibility store to another
without downtime.

**Store types covered:** both stores on SQL (PostgreSQL or MySQL), or both on
Elasticsearch. The two stores must be the **same type** — a SQL store paired with
an Elasticsearch one is rejected and the server will not start. Where behavior
differs between SQL and Elasticsearch, the section says so.

**Dashboard panels:** this playbook names panels rather than showing queries.
They are all on the
[Temporal Server dashboard](../observability/dashboards/server/temporal-server-readme.md)
— in its **Visibility** section unless the text says otherwise — and you need
**v2.15.0 or later**, which is where most of them were added. Grafana does not
display panel ID numbers, so find them by name. The full list, with each panel's
exact title and the section it sits in, is in
[11.3](#113-dashboards-alerts-and-runbooks) — the text below shortens the longer
titles (dropping trailing qualifiers like "(Elasticsearch only)"), so search on
the distinctive part of the name rather than the whole string.

This playbook covers:

- [How dual visibility behaves in normal operation](#1-how-dual-visibility-works)
- [How to tell which of the two stores is failing](#2-finding-out-which-store-failed)
- How to recover from each kind of failure — the
  [primary store](#4-scenario-1--primary-store-failure), the
  [secondary store](#5-scenario-2--secondary-store-failure), or
  [both at once](#6-scenario-3--both-stores-fail)
- [How to move reads and writes from one store to the other](#7-moving-reads-and-writes-between-stores)

Recovery and moving traffic are both done through dynamic config and take effect
in seconds — **neither needs a cluster restart.**

## What dual visibility gives you

Dual visibility **does** give you a manual failover path. If one store starts
failing you can move reads to the other store and move writes with a single
setting. Reads can be moved for every namespace at once or only for the
namespaces you choose. See
[7. Moving reads and writes between stores](#7-moving-reads-and-writes-between-stores).

Dual visibility does **not** give you:

- **Automatic failover.** The server does not move reads or writes between the
  two stores on its own. Switching either one is a dynamic config change you
  make.
- **Read redundancy.** A read goes to exactly one store. If that store is
  failing, the read fails. There is no automatic retry against the other store,
  even when the same record is sitting in it.
- **Two interchangeable copies.** The secondary store only holds records written
  since you turned dual writing on. Anything that started and finished before
  that moment is not in it, and nothing puts those records there on its own.
  Moving reads to an incomplete store means serving an incomplete workflow list —
  with no error, just missing rows. Closing that gap is real work, not a switch;
  see [7.7 Backfilling a store that is missing records](#77-backfilling-a-store-that-is-missing-records).

So it is fair to treat dual visibility as a manual failover option. What you get
when you use it depends on how complete the secondary is:

- **Dual writing has been on continuously** — the two stores hold the same
  records, and moving reads costs you nothing.
- **Dual writing was turned on recently** — the secondary is missing everything
  that started and finished before that, so moving reads serves a shorter
  workflow list.

Either one is a valid thing to do. During an outage a partial list is usually
better than a failed one, and that call is yours to make. What matters is knowing
which of the two you are handing your users, because nothing in the server tells
them: an incomplete store returns fewer rows, not an error. If you need the gap
closed rather than accepted, see
[7.7 Backfilling a store that is missing records](#77-backfilling-a-store-that-is-missing-records).

What dual visibility is not is a way to make a store itself resilient. Surviving
the loss of a single machine is something you get from the store — replica shards
in an Elasticsearch cluster, or a replicated SQL setup — not from dual
visibility.

If a store stays unreachable long enough, the **history service gives up
retrying** the visibility writes it owes that store and sets those records aside
instead. Some of them come back on their own later; some never do. Which is which
depends on what the workflow was doing at the time — see
[3. The data loss clock](#3-the-data-loss-clock).

**The two stores must be the same type.** A SQL primary with an Elasticsearch
secondary, or the reverse, is rejected while the server validates its config,
which happens before any service starts. The process prints this to **stderr**
and exits with status **1**:

```
Unable to create server. Error: config validation error: persistence config error: cannot set visibilityStore and secondaryVisibilityStore with different datastore types.
```

That is a plain command-line print, not a log line, so it will not show up in
whatever you ship logs to.

This also means dual visibility can only ever move you between two stores of the
**same** kind — one SQL database to another, or one Elasticsearch index to
another. It is not a way to get from SQL visibility to Elasticsearch.

---

## Contents

- [What dual visibility gives you](#what-dual-visibility-gives-you)
1. [How dual visibility works](#1-how-dual-visibility-works)
   - [1.1 Choosing the two stores](#11-choosing-the-two-stores)
   - [1.2 The three settings that decide which store is used](#12-the-three-settings-that-decide-which-store-is-used)
2. [Finding out which store failed](#2-finding-out-which-store-failed)
   - [2.1 Which panels to check first](#21-which-panels-to-check-first)
   - [2.3 Why a flat Write Error Rate panel is not proof of health](#23-why-a-flat-write-error-rate-panel-is-not-proof-of-health)
   - [2.4 Detection by store type](#24-detection-by-store-type)
3. [The data loss clock](#3-the-data-loss-clock)
   - [3.1 What is actually lost, and for how long](#31-what-is-actually-lost-and-for-how-long)
4. [Scenario 1 — primary store failure](#4-scenario-1--primary-store-failure)
   - [4.1 What breaks when the primary fails](#41-what-breaks-when-the-primary-fails)
   - [4.2 How to detect a failing primary](#42-how-to-detect-a-failing-primary)
   - [4.5 When the primary comes back](#45-when-the-primary-comes-back)
5. [Scenario 2 — secondary store failure](#5-scenario-2--secondary-store-failure)
   - [5.1 What breaks when the secondary fails](#51-what-breaks-when-the-secondary-fails)
   - [5.2 How to detect a failing secondary](#52-how-to-detect-a-failing-secondary)
   - [5.3 What a dead secondary costs the healthy primary](#53-what-a-dead-secondary-costs-the-healthy-primary)
6. [Scenario 3 — both stores fail](#6-scenario-3--both-stores-fail)
7. [Moving reads and writes between stores](#7-moving-reads-and-writes-between-stores)
   - [7.5 What each change costs you](#75-what-each-change-costs-you)
   - [7.6 Migration order](#76-migration-order)
   - [7.7 Backfilling a store that is missing records](#77-backfilling-a-store-that-is-missing-records)
8. [Why errors keep coming after the store is back](#8-why-errors-keep-coming-after-the-store-is-back)
9. [Search attributes while dual visibility is on](#9-search-attributes-while-dual-visibility-is-on)
10. [After the outage — checking the stores agree](#10-after-the-outage--checking-the-stores-agree)
11. [Reference](#11-reference)

---

## 1. How dual visibility works

This section is the mental model: which stores you can pair, which settings
decide where records go, and what a write and a read actually do. It is worth
reading once before you need it.

**If a store is failing right now**, skip to
[2. Finding out which store failed](#2-finding-out-which-store-failed) and come
back afterwards.

### 1.1 Choosing the two stores

You pick the two stores in static server config, not dynamic config. Changing
either one needs a config change and a **restart of the Temporal services** —
not a restart of the visibility store itself.

`visibilityStore` and `secondaryVisibilityStore` each hold a **name**, and each
name is a key in the `datastores` map below it. The names themselves mean
nothing to the server — `vis-primary` and `vis-secondary` in the example below
are labels, not keywords, and any two names work as long as they match the
datastore entries.

The examples below show only the keys involved in visibility. A working config
also needs `defaultStore` and its own datastore entry — that is the main
persistence store holding workflow history, it is required, and nothing in this
playbook changes it.

Read the path for the store type you run:

- [If both stores are SQL](#if-both-stores-are-sql)
- [If both stores are Elasticsearch](#if-both-stores-are-elasticsearch)

Then read [Pointing both stores at the same place](#pointing-both-stores-at-the-same-place),
which is a mistake you can make with either store type.

#### If both stores are SQL

There is only one config shape — a second datastore named by
`secondaryVisibilityStore`:

```yaml
persistence:
  visibilityStore: vis-primary
  secondaryVisibilityStore: vis-secondary
  datastores:
    vis-primary:
      sql:
        pluginName: "postgres12"
        databaseName: "temporal_visibility"
        connectAddr: "pg-primary:5432"
        # ...
    vis-secondary:
      sql:
        pluginName: "postgres12"
        databaseName: "temporal_visibility_secondary"
        connectAddr: "pg-secondary:5432"
        # ...
```

Every connection setting is per datastore — `connectAddr`, `databaseName`, user
and password, plugin, and the connection pool sizes `maxConns`, `maxIdleConns`
and `maxConnLifetime`. Unlike Elasticsearch, whether the two stores share a
server is not a structural choice; it is just what you point them at. So two
valid arrangements exist:

- **Separate database servers** — different `connectAddr` on each, as above.
- **One server, two databases** — the same `connectAddr` on both, different
  `databaseName`.

**Recommend separate servers.** One server means one failure domain: when it
goes, both stores go, so you are always in
[scenario 3](#6-scenario-3--both-stores-fail) and there is nothing to fail over
to. It also means both stores draw connections, CPU and disk from the same
machine, and dual writing roughly doubles the visibility write load — the two
connection pools above are per datastore, so a shared server sees both. And if
the migration you are actually doing is onto different hardware, a newer engine
version or a managed service, separate servers is the only arrangement that gets
you there.

**One server with two databases is still legitimate** when you are migrating in
place — a schema or table layout change on hardware you intend to keep. It is
simpler to stand up and needs one set of credentials. Just know you are giving
up the failure isolation.

#### If both stores are Elasticsearch

There are two supported ways to write this, and the choice constrains what you
can migrate to later.

**Option A — two separate Elasticsearch datastores.** Each store carries its own
complete Elasticsearch config, so the two can be different clusters:

```yaml
persistence:
  visibilityStore: es-vis-1
  secondaryVisibilityStore: es-vis-2
  datastores:
    es-vis-1:
      elasticsearch:
        version: "v7"
        url:
          scheme: "http"
          host: "es-primary:9200"
        indices:
          visibility: temporal_visibility_v1
    es-vis-2:
      elasticsearch:
        version: "v7"
        url:
          scheme: "http"
          host: "es-secondary:9200"
        indices:
          visibility: temporal_visibility_v2
```

**Option B — one datastore, two indices on the same cluster.** There is no
`secondaryVisibilityStore` key at all here. The second index is declared with the
reserved `secondary_visibility` name inside the primary datastore's `indices` map:

```yaml
persistence:
  visibilityStore: es-vis
  datastores:
    es-vis:
      elasticsearch:
        version: "v7"
        url:
          scheme: "http"
          host: "elasticsearch:9200"
        indices:
          visibility: temporal_visibility_v1
          secondary_visibility: temporal_visibility_v2
```

The server resolves that into a full second visibility store — its own client,
its own bulk processor — pointed at the second index on the same cluster. Every
dynamic config key in this playbook works identically. This is the shape to use
when you are reindexing inside one ES cluster.

The two options are mutually exclusive. Setting `secondaryVisibilityStore` while
the primary datastore also declares `secondary_visibility` is rejected at startup,
and so is declaring `secondary_visibility` on the secondary datastore.

**Which option to use.** The difference comes down to what is shared. Option A
gives each store a full, independent Elasticsearch config. Option B clones the
primary's config and changes nothing but the index name, so **everything else is
shared** — the same cluster, the same major version, the same credentials, the
same TLS and request signing.

| | Option A | Option B |
|---|---|---|
| Two different ES clusters | yes | no |
| Different ES major versions, e.g. v7 → v8 | yes | no |
| Separate credentials, TLS, AWS request signing | yes | no |
| One store can fail while the other serves | yes, if they are separate clusters | no |
| Separate client and bulk processor per store | yes | yes |
| Config size | the ES block written twice | one block plus one line |

**Default to option A.** It can do everything option B does — point both datastores
at the same cluster with different indices — and it is the only one that can
later move you to a different cluster or a newer Elasticsearch version. It also
matches the shape of the two-SQL-store config, so the playbook reads the same way
whichever store type you run.

**Use option B when** the move you are making is to a **new index on the cluster
you already have.** The usual reasons are a mapping change, a new analyzer, or a
different **primary** shard count — none of which can be changed on an existing
index. Primary shard count is fixed when the index is created, so the only way
to change it is to build a second index and move onto it, which is exactly the
shape option B is for. (Replica count is different — that one you can change on
a live index, and it needs none of this.)

The other reason to reach for option B is that it saves you copy-pasting a long
Elasticsearch block with TLS or AWS request signing in it. That duplication is a
genuine maintenance hazard, and avoiding it is option B's real advantage.

**The trap in option B** is that it locks you into one cluster. If you start
there and later need to move clusters or upgrade major versions, you have to
restructure the config into option A and restart the Temporal services. Starting
with option A and pointing both stores at one cluster costs you only a
duplicated block.

**What option B means for the failure scenarios.** Both indices sit on the same
ES cluster, so there is no such thing as one store being down and the other up,
and there is nothing to fail over to. Cluster-level trouble is always
[scenario 3](#6-scenario-3--both-stores-fail). What you can still see per index
is index-level failure — a missing index, a mapping conflict, a per-index write
block — and `visibility_index_name` still tells those apart. If you want dual
visibility to double as a manual failover option, option B cannot give you that.

#### Pointing both stores at the same place

Nothing stops you doing this. The server never checks that the two stores are
actually different:

- **SQL** — the same `connectAddr` and the same `databaseName` on both datastores.
- **Elasticsearch** — the same index name on both.

Either way the cluster starts up normally, and you are left with a single
visibility store that the server treats as two. Three things follow, in
increasing order of how much they should worry you.

**Your records are safe.** Both writes carry the same record with the same
version number, because they come from one visibility task. The store keeps the
first and **rejects** the second rather than layering it on top: a new workflow
is skipped if its row already exists, and an update or close is applied only if
its version is higher than the version already stored. Elasticsearch behaves the
same way, treating an equal version as a conflict, which the server counts as a
success. So nothing is corrupted or double-counted.

**You pay for every write twice.** Each visibility event now costs two
statements instead of one, two sets of connections to the same machine, and two
round trips — and only one of them changes anything. On a busy cluster that is
real, pointless load.

**You lose the ability to tell the two stores apart, and this is the damage that
matters.** The store label in the metrics is just the database or index name, so
two stores sharing a name report as one. Everything in
[2. Finding out which store failed](#2-finding-out-which-store-failed) quietly
stops working — no error, no warning, just one line where you expect two.

**If your dashboards show one store where you expect two, check your config for
this before investigating anything else.**

### 1.2 The three settings that decide which store is used

The config above only names the two stores. It does not decide which of them is
actually used. That is controlled by three dynamic config settings, which you
change while the cluster is running:

- **Which store visibility records are written to.** Every workflow that starts,
  updates its search attributes, or closes produces a visibility record, and
  these settings decide which store or stores receive it.
- **Which store visibility queries are answered from.** `temporal workflow list`,
  `count` — and the same queries from the Web UI and the SDKs — are served by
  one store, and these settings decide which one.

| Setting | Applies to | Values | Default | What it decides |
|---------|-----------|--------|---------|-----------------|
| `system.secondaryVisibilityWritingMode` | whole cluster | `"off"` / `"on"` / `"dual"` | `"off"` | Whether the **secondary** store receives visibility records: `"off"` = primary only, `"dual"` = both stores, `"on"` = secondary only. Despite the name it also decides whether the **primary** keeps receiving them — see the value table below |
| `system.enableReadFromSecondaryVisibility` | whole cluster, or per namespace | `true` / `false` | `false` | `true` answers visibility queries from the secondary store **instead of** the primary. Set it with no constraints and it applies to every namespace; add a `namespace` constraint to move one at a time — see [7.3](#73-read-switch) |
| `system.visibilityEnableShadowReadMode` | whole cluster | `true` / `false` | `false` | `true` also sends a copy of every visibility query to the store that is **not** serving reads, and throws that answer away. Which store that is follows `enableReadFromSecondaryVisibility` — normally the secondary is the shadowed one, but for a namespace whose reads have already moved, the primary is. A migration test tool — see [7.4](#74-shadow-read) |

`system.secondaryVisibilityWritingMode` is a **three-position switch, not an
on/off toggle**, and its values do not mean what they look like:

| Value | Primary gets writes | Secondary gets writes |
|-------|--------------------|-----------------------|
| `"off"` | yes | no |
| `"on"` | **no** | yes |
| `"dual"` | yes | yes |

**`"on"` means the secondary store instead of the primary, not as well as it.**
Writing to both is `"dual"`. This is the most commonly misread setting in dual
visibility: `"on"` sounds like you are switching the secondary on while leaving
the primary alone, but it silently stops all writes to the primary. Anything
written while it is set will be missing from the primary, and there is nothing to
replay afterwards. Full detail in [7.2 Write mode](#72-write-mode).

With all three at their defaults, the secondary store you configured receives
nothing at all — no writes and no reads. Dual visibility is set up but switched
off until you change `system.secondaryVisibilityWritingMode` away from its
default `"off"` — to `"dual"` to write to both stores, or `"on"` to write to the
secondary alone.

The rest of this section explains what writes and reads actually do under those
settings. For how to change them safely, and in what order during a migration,
see [7. Moving reads and writes between stores](#7-moving-reads-and-writes-between-stores).

#### A typo in this setting stops all visibility writes

Any value other than those three is rejected, and every visibility write fails
for as long as it is set. There is no fallback to a default. A misspelling such as
`"duel"` or `"dual "` with a stray space is enough.

Nothing warns you when you make the mistake. The value is a valid string as far as
the config loader is concerned, so it loads with no error and no warning, and
`temporal-server validate-dynamic-config` passes it too — that command only checks
that keys exist and types match, not that a value is one the server will accept.

**The detection signature is unusual and worth knowing, because the obvious
places to look show nothing.** The rejection happens above the layer that records
per-store metrics, so:

- `visibility_persistence_requests` drops to **zero on both stores**. The
  **Visibility Write Request Rate**, **Write Error Rate** and **Write Latency**
  panels all go flat or empty.
- **No error metric fires.** `visibility_persistence_errors` and
  `visibility_persistence_error_with_type` stay silent, because the write never
  reaches either store. The **Visibility Write Error Rate per Store** panel
  being clean means nothing here.

So on the dashboard it looks like visibility simply went quiet. The
**Visibility Write Request Rate per Store** panel flat on both stores is your
first clue, but it cannot tell you why. What identifies the cause is the
**Visibility Task Failures & Internal Errors** panel: its `task_errors_internal`
line rising while **Visibility Write Request Rate per Store** sits at zero on
both stores is the distinctive combination.

**One catch on Visibility Task Failures & Internal Errors.** It counts failing
*attempts*, so it only moves while tasks are still retrying. Once they exhaust
`history.TaskDLQUnexpectedErrorAttempts` and are set aside, its line falls back
to zero — while writes are still completely broken. So a quiet
`task_errors_internal` does not mean the problem went away. The durable signals
are **Visibility Write Request Rate per Store** flat on both stores, and new
workflows simply never appearing in either store. Verified on a test cluster:
with the attempt limit lowered to 3, `task_errors_internal` returned to zero
within a minute while not one of five new workflows reached either store.

**To confirm it, search the history service logs.** They name the bad value
outright, which makes this the one signal that identifies the cause beyond
doubt:

```
Unknown secondary visibility writing mode: duel
```

Search for `Unknown secondary visibility writing mode` and nothing else — the
message it is attached to changes as tasks keep retrying. It starts under
`Fail to process task` and switches to `Critical error processing task,
retrying.` once a task passes 30 attempts, which for this failure is about an
hour in. Searching for either message alone will miss part of the window.

Two things make this easy to miss for longer than it should be:

- **Reads keep working perfectly.** Nothing about reading consults this setting,
  so `temporal workflow list` still responds and the Web UI still loads. It just
  stops showing anything new. The cluster looks healthy.
- **The clock from [section 3](#3-the-data-loss-clock) is running, but it runs
  slower here.** These are *internal* errors, and the server deliberately retries
  those on a slower schedule than a store failure — 3 seconds to start, growing
  by half each time, capped at 3 minutes. So the 70-attempt dead letter queue
  limit is reached after roughly **two and three-quarter hours**, not the ~70
  minutes that a store outage gives you. More time than you might expect, but
  the ending is the same.

  The exception is `history.TaskDLQInternalErrors`. It is **off** by default; if
  it has been turned on, these tasks are dead-lettered on the **first** attempt,
  with no grace period at all.

The fix is to correct the value; writes resume on the next config poll without
restarting anything. Then check the **Dead-Lettered Tasks — Informational**
panel for anything that was dropped while
it was wrong.

### 1.3 How writes work

Only the **history** service writes visibility records. Frontend, worker and
matching read visibility but never write it, so the write mode key has no effect
on them.

Matching's reads are narrow and easy to overlook. It queries visibility only for
**worker versioning** — to work out whether a build ID is still reachable, and to
decide whether to revive a build ID that is being removed from a task queue's
versioning data. Both are workflow counts. If you do not use worker versioning,
matching never reads visibility at all.

In `dual` mode, history sends both writes **at the same time** and waits for
both to finish. The write counts as successful only if **both** stores accepted
it.

That has one consequence worth being clear about, because it is easy to
misread: **if either store fails, the visibility task fails — even though the
other store stored the record perfectly well.** The record is not missing from
the healthy store. What failed is the task, not that store's write.

The task is then retried from the beginning, which sends the record to **both**
stores again, including the one that already has it. Retrying does not go on
forever: after roughly 70 attempts the task is set aside and its record is
dropped — see [3. The data loss clock](#3-the-data-loss-clock).

**Sending the same record twice is harmless.** Every visibility record is a
complete snapshot of the workflow, stamped with a version number taken from the
visibility task that produced it. A retry carries the identical snapshot and the
identical version number, and a store will only apply a write whose version is
**newer** than what it already holds. (On SQL there is a further wrinkle worth
knowing for backfills: a *start* record is an insert that does nothing at all if
a row already exists, regardless of version — so a refresh can add a missing
start row but cannot correct one that is already there. Updates and closes are
version-guarded and do overwrite.) So the store that already stored the
record keeps exactly what it has, and the retry changes nothing there. It is the
same mechanism described in
[Pointing both stores at the same place](#pointing-both-stores-at-the-same-place),
and it works the same way on SQL and on Elasticsearch.

What retries do cost is load. Every attempt is a real write against the healthy
store as well as the broken one, so a store that stays down for a while
multiplies the write traffic against the store that is still up — covered in
[5.3](#53-what-a-dead-secondary-costs-the-healthy-primary).

### 1.4 How reads work

A read goes to exactly one store. By default that is the primary. With
`system.enableReadFromSecondaryVisibility` set for a namespace, every read for
that namespace goes to the secondary instead. The other store gets no read
traffic at all. Reads are never sent to both stores and never combined.

Shadow read mode is the exception. It sends the real read to the normal target
and fires a second copy at the other store on its own separate deadline. The
shadow result is thrown away; only errors on the real read reach the caller. It
exists to warm and exercise a store before you trust reads to it.

Reads cover workflow **list** and **count**, and the equivalent CHASM
operations. Note what is *not* here: at default config `temporal workflow
describe` does **not** read visibility. It is answered from the workflow's
mutable state in the main persistence store, so it keeps working normally when a
visibility store fails.

There is one setting that changes that. `history.visibilityProcessorEnableCloseWorkflowCleanup`
(**off** by default) makes the close task strip a closed workflow's memo and
search attributes out of mutable state, after which anything needing them has to
read visibility instead — **including `describe`**. With it enabled, `describe`
on a closed workflow in that namespace also fails when visibility is down.
Archival reads visibility for the same reason. So: if you have turned that
setting on, treat `describe` as a visibility read too.

### 1.5 How each store type writes

This difference matters for detection, so it is worth knowing before you read
the scenarios.

**SQL** writes are synchronous. History issues the statement and waits. If the
database is unreachable, the call fails immediately with an unavailable error.

**Elasticsearch** writes are asynchronous. Each history pod runs its own bulk
processor per ES store. History hands the document to the processor and then
waits on an acknowledgement:

- Handing the document over blocks if the processor is busy flushing. A backed-up
  processor stalls history's visibility workers.
- History then waits up to `worker.ESProcessorAckTimeout` (default 30s) for the
  processor to confirm the document was committed. That wait deliberately ignores
  the caller's own shorter deadline.
- The processor flushes when it hits `worker.ESProcessorBulkActions` (500 docs),
  `worker.ESProcessorBulkSize` (16 MB), or `worker.ESProcessorFlushInterval` (1s),
  whichever comes first.

**There is no maximum queue length.** Those two size settings are *flush
thresholds*, not a cap on how much the processor will hold, and nothing rejects a
document because the buffer is full. What stops it growing without limit is
backpressure: documents are handed to the processor's workers over a channel with
no queue behind it, so when every worker
(`worker.ESProcessorNumOfWorkers`, default 2) is busy, handing over the next
document simply **blocks** until one is free. History's visibility workers wait
there rather than piling documents up.

The practical size of the buffer, then, is roughly workers × bulk actions —
about 1,000 documents at the defaults — plus whatever is already in flight to
Elasticsearch. A separate map tracks one entry per document still awaiting
acknowledgement, and it has no size limit either. Entries leave it only when the
bulk request **completes** — an expired ack timeout does not release anything. The
store gives up waiting and reports a timeout while the entry and the buffered
document stay exactly where they are, which is why a dead-lettered task on
Elasticsearch so often turns out not to be lost data.

Watch **ES Bulk Processor Queue Depth** for this. Because the real limit is a
blocking hand-off rather than a queue length, a processor falling behind shows up
as history's visibility work slowing down, not as rejected writes.

Two things about these `worker.ESProcessor*` keys are easy to get wrong.

**The name is misleading.** Despite the `worker.` prefix, all five are read by
the **history** service. The worker service does not use them at all, so setting
them for the worker service has no effect.

**Only one of them takes effect while the cluster is running.**
`worker.ESProcessorAckTimeout` is read fresh on every write, so changing it
applies immediately. The other four are read once, when history builds the
store at startup — so **changing bulk actions, bulk size, flush interval or
worker count needs a history restart before it does anything.**

Three outcomes, and they do **not** all look the same on the dashboard:

| Outcome | Error returned | Shows on the **Visibility Write Error Rate per Store** panel? | Where to look instead |
|---------|----------------|----------------------|-----------------------|
| ES rejects the bulk, or rejects individual documents (429, mapping error, missing index) | unavailable | **Yes** | **ES Bulk Processor Errors by HTTP Status** for the HTTP status |
| ES unreachable | unavailable | **Yes** | **ES Bulk Processor Errors by HTTP Status** shows `http 0` — but only when Elasticsearch is *gone*, not when it is merely hung |
| No acknowledgement inside the ack timeout | timeout | **No** | **Visibility Errors by Type per Store** (`error_type=persistence_TimeoutError`) — and *only* that panel. Every `elasticsearch_bulk_processor_*` panel stays **empty**, because all of them are recorded when a bulk completes and a hung cluster never completes one. See below |

That last row is the trap. See
[2. Finding out which store failed](#2-finding-out-which-store-failed).

---

## 2. Finding out which store failed

Both stores emit the same metrics under the same names. What separates them is a
single label, and if you do not know that, nothing on the dashboard makes sense
during an incident. This section goes in the order you need it: the panels to
open, how to read which store is which, the one panel that lies to you, and what
differs between SQL and Elasticsearch.

### 2.1 Which panels to check first

Open these, in this order:

| Panel | What it tells you |
|-------|-------------------|
| **Visibility Write Request Rate per Store** | A line dropping flat means writes to that store stopped — store gone, or write mode changed |
| **Visibility Write Error Rate per Store** | Which store is erroring |
| **Visibility Write Latency per Store** | Stores drifting apart means one is struggling |
| **Visibility Task End-to-End Latencies** | How far behind the visibility queue has fallen |
| **Dead-Lettered Tasks — Informational** (in the **History Task DLQ / Terminal Failures** section) | Visibility tasks have already been dropped — see [section 3](#3-the-data-loss-clock) |

### 2.2 The one label that matters

Five metrics carry a `visibility_index_name` label, and they are the only way to
tell the two stores apart. Every one of them has a panel:

| Metric | Panel |
|--------|-------|
| `visibility_persistence_requests` | **Visibility Write Request Rate per Store**, **Visibility Read Request Rate per Store** |
| `visibility_persistence_errors` | **Visibility Write Error Rate per Store**, **Visibility Read Error Rate per Store** |
| `visibility_persistence_error_with_type` | **Visibility Errors by Type per Store** |
| `visibility_persistence_resource_exhausted` | **Visibility Rate Limit Rejections per Store** |
| `visibility_persistence_latency` | **Visibility Write Latency per Store**, **Visibility Read Latency per Store** |

Anything else you might reach for — including every
`elasticsearch_bulk_processor_*` metric and every history task metric — has no
store label, so it cannot tell you which of the two stores is involved.

The label value is **not** a fixed string. It is whatever you configured:

- SQL — the database name from `databaseName`
- Elasticsearch — the index name from `indices.visibility`

Open the **Visibility Write Request Rate per Store** panel once while things are
healthy and note the two label values in its legend, so you recognise them under
pressure. You should see exactly two; if you see one, check whether both stores
were accidentally pointed at the same database or index — see
[1.1](#11-choosing-the-two-stores).

None of these metrics carry a `namespace` label, so the dashboard's `$namespace`
selector has no effect on any of the per-store panels.

### 2.3 Why a flat Write Error Rate panel is not proof of health

The **Visibility Write Error Rate per Store** panel does not show every error.
The metric behind it, `visibility_persistence_errors`, deliberately skips
several error types — and two of the skipped ones are failures you very much
want to know about:

- **Timeouts.** An Elasticsearch write that is never acknowledged inside the ack
  timeout is not counted at all. So the panel sits flat while the store is
  failing to keep up.
- **Rate limit rejections.** Visibility writes throttled by
  `system.visibilityPersistenceMaxWriteQPS` are counted on
  `visibility_persistence_resource_exhausted` instead, which is a different
  panel.

**So a flat line on that panel does not mean visibility is healthy — it may mean
the failure is one of the kinds it cannot see.** The complete per-store error
signal is the **Visibility Errors by Type per Store** panel, which counts every
error and splits them by cause.

Make that panel your first stop whenever you suspect visibility and the Write
Error Rate panel looks clean. Its `error_type` value names the failure:
`serviceerror_Unavailable` for a store that is unreachable or rejecting,
`persistence_TimeoutError` for an Elasticsearch store that is not acknowledging
in time, and `serviceerror_ResourceExhausted` for throttling.

These values are the Go error type with dots turned into underscores, which is
what the metrics exporter does to them. Query `persistence_TimeoutError`, not
`TimeoutError` — the short form matches nothing.

One caveat if you do not run the default metrics setup: that dots-to-underscores
substitution is applied by the **tally** reporter, which is what you get unless
you have set `framework: opentelemetry`. On the OpenTelemetry path the values
keep their dots — `persistence.TimeoutError` — so any query or alert written
against the underscore form will silently match nothing. Check one value in your
own Prometheus before trusting the spelling.

### 2.4 Detection by store type

| Symptom | SQL store | Elasticsearch store |
|---------|-----------|---------------------|
| Store unreachable | **Visibility Write Error Rate per Store** rises; log `sql handle: unable to refresh database connection pool` | **Visibility Write Error Rate per Store** rises; **ES Bulk Processor Errors by HTTP Status** shows `http 0`; log `Unable to commit bulk ES request.` |
| Store hung rather than gone | **Write Error Rate rises**, with `error_type=context_DeadlineExceeded` — each attempt is cut off by the 3-second visibility task timeout, so a hung SQL store is *not* invisible | **Write Error Rate and every ES bulk processor panel stay flat.** Only **Visibility Errors by Type per Store** moves, as `persistence_TimeoutError`. This is the failure with no signal on any error panel — and it is Elasticsearch-specific |
| Store up but rejecting | n/a | **ES Bulk Processor Errors by HTTP Status** shows the real status — `429` overloaded, `400` mapping problem |
| Store up but too slow — still completing bulks, just late | latency climbs on **Visibility Write Latency per Store** | **Write Error Rate stays flat**; **Visibility Errors by Type per Store** shows `error_type=persistence_TimeoutError` for anything past the ack timeout; **ES Bulk Processor Queue Depth** climbs and **ES Write Confirm Latency vs Ack Timeout** approaches the 30s line. This is the case those two panels *do* catch — unlike a fully hung store |
| Index or database missing | connection or query error | **ES Bulk Processor Errors by HTTP Status** shows `http 404`; every task fails and never recovers until the index exists |

**Why a hung Elasticsearch store shows nothing on any bulk processor panel.**
Every `elasticsearch_bulk_processor_*` metric — errors, queue depth and confirm
latency alike — is recorded at the point a bulk request **completes**. The
Elasticsearch client has no request timeout of its own, so a cluster that
accepts the connection and then goes silent never completes the bulk, and none
of those three metrics is ever written. They do not climb; they stay empty.

Two consequences:

- An `http 0` on **ES Bulk Processor Errors by HTTP Status** means Elasticsearch
  was **gone** — the address stopped resolving or refused the connection — not
  that it was slow.
- **Queue Depth climbing and confirm latency approaching 30s are signatures of a
  store that is slow but still answering**, not of one that has hung. If those
  two are empty while `persistence_TimeoutError` is rising, the store is not
  slow — it is not answering at all.

For a fully hung Elasticsearch store, **Visibility Errors by Type per Store** is
the only panel that moves.

Two notes on the Elasticsearch metrics. They only exist on `service_name="history"`,
because history is the only service that runs a bulk processor. And they carry
**no** `visibility_index_name` label — with two ES stores, both processors report
into the same series. You cannot tell primary from secondary at the bulk
processor layer. For that, go back to `visibility_persistence_error_with_type`.

### 2.5 Why the logs cannot tell you

Every dual-visibility write failure produces the same stack trace, naming the same
internal dual-write code, whichever of the two stores actually failed. The log
line tells you a visibility write failed.
Only the `visibility_index_name` label tells you where.

---

## 3. The data loss clock

So you have found which store is failing. Before you start fixing it, read this
section — because **a clock started running the moment the store broke**, and
how much time is left decides whether you are doing a repair or a repair plus a
backfill.

The short version: workflows keep running normally and nothing user-facing
breaks on the healthy store, so it is tempting to treat a failing visibility
store as low urgency. For a little over an hour, that is true. After that, the
server stops trying to write the records it owes that store, and they are gone
unless you put them back by hand.

**This is the most important section in this playbook.**

Visibility tasks do not retry forever. Failing to reach a store, an Elasticsearch
write that is never confirmed, and an Elasticsearch rejection all count as
unexpected errors. After 70 unexpected attempts on the same task, history gives up
retrying and moves the task to the **history task dead letter queue** — a holding
area for work the server has stopped trying to do.

The gap between retries starts at 1 second, grows by 10% each attempt, and stops
growing at 3 minutes. Each gap is then shortened by a random amount, to somewhere
between 80% and 100% of that, which keeps every shard from retrying in lockstep.
Adding it up, **70 attempts takes roughly 70 minutes.**

**A visibility store that stays broken for a little over an hour starts dropping
visibility records.** The workflows themselves are completely unaffected — they
keep running and completing normally. What is lost is the visibility record in
that one store.

### 3.1 What is actually lost, and for how long

Not every dropped task means a permanently missing record, and the difference
matters when you are deciding how hard to push on recovery.

Every visibility write contains the **whole record**, not just the part that
changed. Whether a workflow is starting, being updated or closing, the server
builds the complete row or document from scratch and writes all of it, stamped
with a sequence number that only ever increases. That means a later write can
fully rebuild a record that an earlier dropped write left missing.

| Task dropped | What you see | Comes back on its own? |
|---|---|---|
| **Start**, workflow still running | The workflow is missing from list and count in that store | **Yes.** The workflow's next visibility task — a search attribute or memo update, or its close — writes the complete record. A workflow that changes no search attributes stays invisible until it closes. |
| **Update** | That particular search attribute or memo change is missing | **Yes.** The next update, or the close, rewrites every field. |
| **Close** | If the start landed, the workflow keeps showing as Running and keeps appearing in open-workflow queries. If the start was dropped too, the workflow is absent entirely. | **No.** The close is the last visibility task a workflow ever emits, so nothing rebuilds it — but see the retention note below: the stale record does not last forever, it is eventually **deleted** rather than corrected. |
| **Delete** | A record that should be gone stays in the store, outliving the workflow and its retention | **No.** Nothing retries the delete. |

So the lasting damage is concentrated in workflows that **closed while the store
was down**, plus records that should have been deleted. Long-running workflows
that merely started during the outage repair themselves the moment they next
touch visibility.

That has a practical consequence: **a stuck-Running workflow list is a symptom of
dropped close tasks**, not a workflow problem. If you see completed workflows
listed as running in one store only, check the **Visibility Tasks Dead-Lettered by
Task Type** panel for that period.

#### What retention does to a dropped close

A stale "Running" record is not permanent, and the reason matters for how much
time you have.

When retention expires for a workflow, the server generates a **delete**
visibility task, which removes the record from both stores. By default that
delete does **not** wait for the close to have succeeded
(`history.visibilityProcessorEnsureCloseBeforeDelete`, default **false**), so it
removes the stale record regardless.

So the real sequence for a workflow that closed during the outage is: wrongly
listed as Running from its close until its retention expires, then **deleted**.
It is never corrected to show as closed unless you rebuild it yourself.

Two consequences:

- **The repair window is the namespace's retention period, measured from the
  workflow's close time — not from when you noticed.** A namespace with a short
  retention and an outage longer than it can age workflows out while the store is
  still broken. In that case the close record was never written, the record is
  deleted, and the workflow's mutable state is gone too — so there is nothing
  left to rebuild from and nothing to rebuild it into.
- **Replaying an old dead-lettered close after retention has deleted the
  workflow resurrects it.** The delete already ran and will not run again, so a
  replayed close writes a record for a workflow that should no longer exist and
  nothing will ever remove it. Check whether the workflows are still within
  retention before merging old visibility dead letter queue messages — see
  [3.6](#36-replaying-what-was-dropped).

#### On Elasticsearch, a dead-lettered task often is not lost data

Everything above describes what happens when a write genuinely never lands. On
Elasticsearch there is a wrinkle that makes the dead letter queue **over-report**
the damage.

Writes go through a bulk processor that holds documents in memory. The server
stops waiting after `worker.ESProcessorAckTimeout` and fails the task — but the
document is **still sitting in the processor's buffer**. When Elasticsearch comes
back, the processor flushes and the document is indexed, even though the task that
produced it already failed and may already have been dead-lettered.

This is not theoretical. On a test cluster, twelve workflows were dead-lettered
during an Elasticsearch outage — `dlq_writes` counted twelve and `tdbg dlq list`
showed twelve messages — and after recovery **all twelve records were present in
both stores**.

So on Elasticsearch, treat the **Visibility Tasks Dead-Lettered by Task Type** panel as *"tasks gave up"*, not *"records are
missing"*. Confirm which it is by looking for the records themselves:

```bash
curl -s "http://<es-host>:9200/<index>/_count" -H 'Content-Type: application/json' \
  -d '{"query":{"prefix":{"WorkflowId":"<prefix>"}}}'
```

If the records are there, the dead-lettered tasks are noise and replaying them is
harmless but unnecessary. If they are genuinely absent, replay per
[3.6](#36-replaying-what-was-dropped). On SQL there is no such buffer, so a
dead-lettered task there does mean the write never landed.

### 3.2 Replaying is safe in any order

Because each write carries the task ID as its version, replaying dropped tasks
cannot corrupt newer data. A stale start replayed after the close has already
landed is simply ignored — SQL skips the insert because the row exists, and
Elasticsearch rejects it as a version conflict, which the server treats as
success. You do not need to work out the right replay order.

This is on by default. The two keys:

| Key | Default | Effect |
|-----|---------|--------|
| `history.TaskDLQEnabled` | `true` | Dead letter queue is active |
| `history.TaskDLQUnexpectedErrorAttempts` | `70` | Attempts before a task is dropped |

`history.TaskDLQEnabled` carries a warning in the server source that the dead
letter queue is Cassandra-only. That comment is out of date. It is implemented
for SQL backends too and it will fire on them.

### 3.3 Seeing records about to be dropped

The **Visibility Task Retry Depth** panel is the early warning.
It shows the deepest attempt count visibility tasks have reached.

**A healthy cluster reads about 1, not empty.** Every visibility task records its
attempt count when it finishes, so the panel normally has data — it just sits at
the bottom. What matters is the line **climbing**. Tasks still retrying report in
flight as well, once they pass 30 attempts.

The threshold lines mark 30 and 70. A line climbing toward 70 means you are on the
clock: roughly 70 minutes from the first failure is when records start being
dropped. This is the panel to put on a screen during a visibility store outage.

Two things to read it correctly. The value is **coarse**, because the histogram
buckets step 1, 2, 5, 10, 20, 50, 100 — a task at 35 attempts reads as 50, and one
at 70 reads as 100, so the thresholds trigger on the bucket above them. Read it as
a magnitude, not an exact count. And an **empty** panel means no visibility tasks
completed in the window at all, which on a quiet cluster is normal and is not the
same as "nothing is retrying".

### 3.4 Confirming records were already dropped

The **Visibility Tasks Dead-Lettered by Task Type** panel is the one to watch.
Any value above zero means visibility tasks have already been dropped, and it
splits them by task type — `VisibilityTaskCloseExecution` and
`VisibilityTaskDeleteExecution` are the ones that do not repair themselves, per
[section 3.1](#31-what-is-actually-lost-and-for-how-long).

Two neighbouring panels in the **History Task DLQ / Terminal Failures** row look
like they cover this and do not. The **Dead-Lettered Tasks — Informational** panel blends visibility with retention
and workflow-task-timeout operations, so visibility is hard to pick out. Panel
**Dead-Lettered Tasks — Execution-Stranding**, the page-worthy one, matches only timer and transfer tasks and will
**never** show visibility at all — reading zero there proves nothing.

The history logs carry
`Marking task as terminally failed, will send to DLQ. Maximum number of attempts
with unexpected errors`.

**Alert 83 — Visibility Tasks Dead-Lettered** covers this, at critical severity
on any sustained rate above zero. Note that the general dead letter queue alert
(80) does **not**: it matches only timer and transfer operations and excludes
visibility by design, so a quiet alert 80 tells you nothing about visibility.

See the [alert 83
runbook](../observability/alerts/server/runbooks/83-visibility-tasks-dead-lettered.md)
for the replay and rebuild procedure.

### 3.5 Buying more time

If a store will be down longer than an hour and you cannot afford to lose
records, raise the attempt limit before you hit it:

```yaml
history.TaskDLQUnexpectedErrorAttempts:
  - value: 500
```

Past about the 55th attempt every retry is 2.5 to 3 minutes apart, so 500
attempts buys roughly 20 hours. Put it back to `70` once the store is healthy — a
permanently high value means genuinely broken tasks retry for a day instead of
being set aside where you can see them.

The alternative is to stop writing to the broken store entirely. That is
[section 7](#7-moving-reads-and-writes-between-stores), and it trades the same
records away deliberately instead of by timeout.

### 3.6 Replaying what was dropped

Records in the dead letter queue are recoverable. First check whether a visibility
queue exists at all and how much is in it:

```bash
tdbg dlq list
```

Then read what is in it before changing anything:

```bash
tdbg dlq read --dlq-type visibility --max-message-count 100 --last-message-id <id>
```

**`--dlq-type` takes a number, never a name.** For the history task DLQ it is
always the numeric category id — **4** for visibility. The category name is not
accepted on any build:

```bash
tdbg dlq read --dlq-type 4 --max-message-count 100 --last-message-id <id>
```

Passing `visibility` gets you
`strconv.Atoi: parsing "visibility": invalid syntax`. (Names like `namespace` and
`history` belong to the older v1 DLQ, a different queue selected with
`--dlq-version` — they are not category names for this one.)

Two more rough edges: leave out `--last-message-id` and the command prompts for
confirmation, and without a terminal attached (a `docker exec` without `-t`) that
prompt **panics on EOF** rather than failing cleanly. Pass the global `--yes`
flag before the subcommand to skip it. `tdbg dlq list` needs no flags at all and
is the safe way to see whether a visibility queue exists and how much is in it.

Then replay, once the store is healthy:

```bash
tdbg dlq merge --dlq-type visibility --last-message-id <id>
```

Same as above — `--dlq-type 4`, not the name.

Merge re-enqueues the tasks and deletes them from the queue as it goes. It
returns a job token; check progress with:

```bash
tdbg dlq job describe --job-token <token>
```

Do not merge while the store is still broken. The tasks will fail their 70
attempts again and land straight back in the queue.

**Check the age of what you are replaying.** A dead-lettered **close** task for
a workflow that retention has since deleted will write its record back — and
because the delete visibility task has already run and will not run again,
nothing will ever remove it. You get a closed-workflow record for a workflow
that no longer exists, indefinitely.

This matters most when the dead letter queue has been sitting for a while, or
when the namespace has a short retention. Read the messages first (`dlq read`
above) and check the workflow IDs are still inside their retention window before
merging. Anything already aged out is better dropped than replayed —
[3.1](#31-what-is-actually-lost-and-for-how-long) explains why retention, not
the close task, is what finally clears those records.

---

## 4. Scenario 1 — primary store failure

This is the failure that reaches your users. By default every visibility query
is answered by the primary, so when the primary breaks, `temporal workflow list`
breaks with it — there is no automatic fallback to the secondary, even when the
secondary holds the same records.

The good news is that you can move reads yourself in seconds, and that is the
first thing to do. Fixing the store comes second.

### 4.1 What breaks when the primary fails

- Workflow execution: **unaffected**. Workflows keep running and completing.
- `temporal workflow list` / `count`: **broken**. Reads go to the primary only,
  and there is no fallback to the secondary.
- `temporal workflow describe`: **works** — it reads mutable state, not
  visibility.
- Web UI workflow list: errors or empty.
- Visibility task backlog: grows on every history shard.
- Records: safe for about 70 minutes, then tasks start being dropped. Workflows
  that close during the outage are the ones that stay missing — see
  [section 3.1](#31-what-is-actually-lost-and-for-how-long).

### 4.2 How to detect a failing primary

**The most important thing in this section:** a primary store failure does not
have one signature — it has **three**, and one of them leaves both error panels
silent. Which one you get depends on *how* the store failed, not on how badly.

| How the primary failed | Write Error Rate | ES Bulk Processor Errors | Errors by Type per Store |
|---|---|---|---|
| **Gone** — process dead, host unresolvable, connection refused | **rises** | `http 0` | `serviceerror_Unavailable`, often with `persistence_TimeoutError` alongside it |
| **Hung** — reachable but not answering, **Elasticsearch only** | **flat** | **flat** | `persistence_TimeoutError` |
| **Hung** — reachable but not answering, **SQL** | **rises** | n/a | `context_DeadlineExceeded` |
| **Index or database missing** | **rises** | `http 404` (ES) | `serviceerror_Unavailable` (ES); on SQL the raw driver error type |

A store that is fully gone usually reports **two** error types at once: the new
connections fail immediately, while writes already sitting in the Elasticsearch
bulk buffer go on to hit the write ack timeout. Seeing both together means
*gone*, not two separate problems.

**The hung case is the dangerous one, and only on Elasticsearch.** An
Elasticsearch store that accepts connections but never answers produces no error
on any error panel: the write never completes, so nothing is recorded as having
failed and every bulk processor panel stays empty. Only **Visibility Errors by
Type per Store** moves. A hung **SQL** store behaves differently — each attempt
is cut off by the 3-second visibility task timeout, which *is* counted, so Write
Error Rate rises as normal.

So: **on Elasticsearch, do not conclude visibility is healthy from a flat Write
Error Rate panel.** Make **Visibility Errors by Type per Store** your first stop,
not your second — it is the only panel that catches all of the rows above.

**On telling the two stores apart:** every `visibility_persistence_*` metric
carries `visibility_index_name`, so Write Error Rate, Write Request Rate, Read
Error Rate and Errors by Type all name the store. What does **not** is the
`elasticsearch_bulk_processor_*` family behind **ES Bulk Processor Errors by HTTP
Status**, **ES Bulk Processor Queue Depth** and **ES Write Confirm Latency vs Ack
Timeout** — those are tagged with the operation and HTTP status but not the
index, so under dual visibility both stores report into one series. Use them to
see *how* an Elasticsearch store is failing, and any per-store panel to see
*which* one.

#### What to check, in order

1. **Visibility Errors by Type per Store** — the only panel that catches all
   three failure modes. Read the `error_type` and match it against the table above.
2. **Visibility Write Request Rate per Store** — if it is flat on **both**
   stores and Errors by Type is quiet too, this is not an outage at all; it is
   an invalid
   `system.secondaryVisibilityWritingMode`. See
   [1.2](#12-the-three-settings-that-decide-which-store-is-used).
3. **Visibility Read Error Rate per Store** — the only per-store view of the
   read path, and the one that tells you users are affected.
4. **Visibility Write Latency per Store** and **Visibility Task End-to-End Latencies** — latency and end-to-end task latency, to gauge how far
   behind the queue has fallen.
5. **Visibility Task Retry Depth** — how close dropped records are.

#### Which alerts fire, and which do not

Alerts `059a` / `059b` / `059c` all filter `service_name="history"`, so they cover
the **write** path only. A primary that is failing **reads** fires none of them.
Two alerts exist specifically to close that gap:

| Alert | Covers | Severity |
|---|---|---|
| **85 — Visibility Read Errors** | Read failures on any service, so a broken workflow list pages someone | critical |
| **84 — Visibility Store Not Acknowledging Writes** | The hung-store case, which is invisible to 059a/b/c because timeouts are not counted as errors | warning |
| **83 — Visibility Tasks Dead-Lettered** | Records already dropped past the 70-minute mark | critical |

**If you have not deployed those three, nothing alerts on a failing primary's
read path or on a hung store** — and the first reliable indication is a user
reporting that workflow list is broken. Check which alerts your cluster actually
has before you rely on being paged. The
[server alert set](../observability/alerts/server/README.md) ships all three.

#### What users see

`temporal workflow list` and `count` fail, and so does the Web UI workflow
list. `describe` keeps working — it reads mutable state, not visibility. The error text depends on the failure mode: a refused connection
surfaces quickly as an unavailable error, while a hung store or a missing index
surfaces as **`context deadline exceeded`** after the client timeout.

**The data is usually still there.** If dual writing has been on, the secondary
store holds the same records and is completely healthy — reads fail anyway,
because a read goes to one store and there is no fallback. That is the whole
reason [7.3](#73-read-switch) exists. Moving reads is the fix, and it takes
seconds.

**If the primary is SQL**, history logs the connection failure and then the task
failure:

```json
{"msg":"sql handle: unable to refresh database connection pool",
 "error":"dial tcp <primary-host>:5432: connect: connection refused"}
{"msg":"Critical error processing task, retrying.",
 "error":"no usable database connection found",
 "error-type":"serviceerror.Unavailable",
 "attempt":N,
 "unexpected-error-attempts":N,
 "task-category":"visibility"}
```

The first line is SQL-specific — `sql handle:` comes from the SQL connection
pool and never appears on an Elasticsearch cluster.

**If the primary is Elasticsearch**, the store-specific line is instead:

```json
{"msg":"Unable to commit bulk ES request.",
 "error":"elastic: Error 0 (): ...",
 "logging-call-at":"processor.go"}
```

That one appears when Elasticsearch is **gone** or rejecting. A store that is
merely *hung* logs nothing store-specific at all, because the bulk request never
completes — the only evidence is the task failure line below, with
`error-type":"persistence.TimeoutError`.

**The task failure line is the same on both store types.** `Fail to process
task`, `Critical error processing task, retrying.`, `attempt`,
`unexpected-error-attempts` and `task-category":"visibility` all come from the
generic history task framework, not from any store, so everything in the next
paragraph applies equally to SQL and Elasticsearch. Only `error` and
`error-type` differ by store.

That `Critical error processing task, retrying.` line only appears once a task
passes **30 attempts**, which is only about two and a half minutes in. Below that
the same failure logs at warning level as `Fail to process task`, so do not treat
a quiet error log as a quiet cluster. Watch `unexpected-error-attempts` against
the 70 limit from [section 3](#3-the-data-loss-clock). Both the 30-attempt
switch and the 70-attempt limit are store-agnostic.

### 4.3 How to remediate a failing primary

**Step zero: give users their reads back.** This takes seconds and does not wait
on the store being fixed. If dual writing has been on, the secondary holds the
same records:

```yaml
system.enableReadFromSecondaryVisibility:
  - value: true
```

That moves every namespace's reads. To limit it to one namespace while you are
still deciding, constrain it — see [7.3](#73-read-switch). Confirm the switch took
effect on the **Visibility Read Request Rate per Store** panel: the secondary's
line should start carrying the reads within seconds, with no restart.

Two things to be honest about before you do it. The secondary only holds records
written since dual writing was turned on, so anything older is still invisible —
you have traded a broken list for an incomplete one, which is usually the better
trade but is not a fix. And reads will now *stay* there until you move them back,
which is the subject of [4.5](#45-when-the-primary-comes-back).

Then, with users served:

1. Bring the primary store back. This part is your infrastructure, not
   Temporal's, and how you do it depends entirely on how you run the store —
   restoring a node, restarting a database, clearing a disk watermark that
   flipped an index to read-only, or fixing a network path. Temporal needs
   nothing from you here beyond the store answering again.

2. No server restart is needed. History reconnects on its own. Errors may keep
   appearing for a minute or two after the store is reachable — see
   [section 8](#8-why-errors-keep-coming-after-the-store-is-back).

3. Confirm the primary's error line on the **Visibility Write Error Rate per Store** panel returns to zero, and that
   `visibility_persistence_error_with_type` is clean for that store.

4. Confirm `temporal workflow list` works again.

5. Watch the **Visibility Write Latency per Store** and **Visibility Task End-to-End Latencies** panels drain back to baseline. This takes as long as
   it takes to work through the backlog.

6. Check the **Dead-Lettered Tasks — Informational** panel for dead letter queue writes during the outage. If there
   were any, replay them —
   [section 3.6](#36-replaying-what-was-dropped).

7. Check the stores agree —
   [section 10](#10-after-the-outage--checking-the-stores-agree).

### 4.4 If recovery is going to take a while

Two options, both of which give something up. Read
[section 7](#7-moving-reads-and-writes-between-stores) before using either.

**Reduce write pressure by dropping the secondary.** Sensible if the primary is
struggling rather than dead and you want to halve the write load:

```yaml
system.secondaryVisibilityWritingMode:
  - value: "off"
```

Records written while this is set will be missing from the **secondary**. Set it
back to `"dual"` as soon as the primary is healthy.

**Switch writes to the secondary.** Only if the primary will be down for hours
and the secondary is a full, current copy:

```yaml
system.secondaryVisibilityWritingMode:
  - value: "on"
```

Be clear about what this does: `"on"` means **secondary only**. Writes stop
reaching the primary entirely, and records written during this window will be
missing from the primary with nothing to replay later — no task failed, so
nothing lands in the dead letter queue. It does not redirect reads either; for
that you also need `system.enableReadFromSecondaryVisibility` per namespace.

**Which to choose comes down to how long the outage will last.** Leaving it on
`"dual"` means every visibility task keeps failing against the dead primary,
retrying, and eventually being dead-lettered — retry load and dead letter churn,
but a short outage then heals itself with no backfill. Setting `"on"` stops that
churn immediately and guarantees a gap you will have to close by hand later. Under
about an hour, leave it alone; for a multi-hour outage, `"on"` is usually worth
the backfill.

### 4.5 When the primary comes back

The primary is now missing everything written while it was down. You have two
strategies and they are not equivalent.

**Option 1 — backfill, then move reads back.** Config keeps matching reality, and
it is the right default.

1. **Leave reads on the secondary** while you do this. The backfill is driven by a
   visibility query, and that query is served by whichever store reads point at —
   so it has to be the store that still has the records. Pointing reads back too
   early is the single most common way to get stuck here.
2. **Measure the gap** before fixing it, so you know whether you succeeded.
   Compare record counts per store —
   [10.3](#103-do-the-record-counts-match).
3. **Backfill** with a query bounded to the outage window, so you are not
   refreshing the whole cluster:

   ```bash
   tdbg --namespace <ns> workflow refresh-tasks \
     --query 'StartTime > "2026-01-01T09:00:00Z"' \
     --reason "backfill primary after outage"
   ```

   Note that a `StartTime` bound deliberately includes workflows that are
   **still running** — their start records are part of the gap. Refreshing a
   running workflow is safe, but it is not free; see
   [what refreshing does to a running workflow](#what-refreshing-does-to-a-running-workflow).

   Full mechanics and caveats in
   [7.7](#77-backfilling-a-store-that-is-missing-records) — in particular that this
   refreshes **every** task category, not just visibility, and writes to **both**
   stores.
4. **Confirm the counts match**, then move reads back by removing
   `system.enableReadFromSecondaryVisibility` (or setting it to `false`).
5. **Watch Visibility Read Request Rate per Store** to confirm reads returned to
   the primary.

**Option 2 — leave reads on the secondary for good.** No backfill, no gap to
close, and it is tempting after a long outage. The cost is that the store your
config calls "secondary" is now serving production, and nobody can tell that from
the config alone — only from dynamic config. If you choose this, finish the job:
swap `visibilityStore` and `secondaryVisibilityStore` in static config, clear the
read flag, and restart the Temporal services. The restart is needed because the
store pointers are static config, not dynamic.

#### The workflows you may not get back

Refreshing a workflow's tasks only works while its **mutable state still
exists**. Retention deletes mutable state, so once a workflow ages out there is
nothing left to rebuild from.

That gives you a deadline, and it is tighter than it first looks: **the window
is the namespace's retention period measured from each workflow's close time**,
not from when you noticed the problem. A namespace with a short retention and an
outage longer than it can age workflows out *while the store is still broken* —
in which case the close was never written, retention deletes the record anyway,
and the mutable state you would have rebuilt from is gone. Nothing can recover
those.

Practically: **find what closed during the outage and deal with those first**,
shortest-retention namespaces before anything else. Workflows that merely
started during the outage and are still running repair themselves the next time
they touch visibility, so they are not urgent.

One trap when replaying instead of refreshing: if retention has already deleted
a workflow, replaying its dead-lettered close **resurrects** a record that
should not exist, and nothing will delete it again. Check the workflows are
still within retention first —
[3.1](#31-what-is-actually-lost-and-for-how-long) and
[3.6](#36-replaying-what-was-dropped).

---

## 5. Scenario 2 — secondary store failure

The mirror image of scenario 1, and much less urgent: reads are answered by the
primary, so nothing user-facing breaks. Workflows run normally and the workflow
list keeps working.

Two reasons not to ignore it. The same 70-minute clock from
[section 3](#3-the-data-loss-clock) is running against the secondary, and a
broken secondary quietly multiplies the write load on your healthy primary —
[5.3](#53-what-a-dead-secondary-costs-the-healthy-primary).

### 5.1 What breaks when the secondary fails

- Workflow execution: **unaffected**. Starting workflows does not slow down or
  fail — the visibility write happens in a background history task, not on the
  caller's path.
- `temporal workflow list` / `count`: **working**, as long as no namespace
  reads from the secondary.
- The primary store itself: **not slowed**. Its own line on the **Visibility
  Write Latency per Store** panel stays flat throughout. Both writes are issued
  in parallel, so a slow secondary does not delay the primary's own request.
- The primary's records: **still land, on the first attempt.** Anyone querying
  the primary sees no gap at all while the secondary is down.
- Visibility tasks: **failing and retrying.** The write is only reported as
  successful when *both* stores succeed, so the task fails even though the
  primary wrote fine.
- Records: safe for about 70 minutes, then tasks start being dropped from the
  secondary. See [section 3.1](#31-what-is-actually-lost-and-for-how-long).

Lower urgency than scenario 1 — nothing user-facing is broken — but the same
70-minute data loss clock is running. If the secondary is the target of a
migration you have not finished, those lost records are records you will have to
backfill later.

### 5.2 How to detect a failing secondary

A failing secondary has the same signatures as a failing primary, and the same
trap: **how it failed decides which panels move.**

| How the secondary failed | Write Error Rate | ES Bulk Processor Errors | Errors by Type per Store |
|---|---|---|---|
| **Hung** — reachable but not answering, **Elasticsearch only** | **flat** | **flat** | `persistence_TimeoutError` |
| **Hung** — reachable but not answering, **SQL** | rises | n/a | `context_DeadlineExceeded` |
| **Gone** — process dead, connection refused | rises | `http 0` (ES) | `serviceerror_Unavailable` **and** `persistence_TimeoutError` |

A hung **Elasticsearch** secondary is close to invisible: every error panel stays
flat, including all three bulk processor panels, and the only moving signal is
**Visibility Errors by Type per Store**. A hung **SQL** secondary is not
invisible — the 3-second task timeout is counted, so Write Error Rate rises.

A store that is fully gone is easier either way, but on Elasticsearch it reports
*both* error types at once, because the connection failures come back immediately
while requests already sitting in the bulk buffer go on to hit the write timeout.

**The bulk processor panels cannot tell you which store broke.** Every
`visibility_persistence_*` metric carries the `visibility_index_name` label, so
any per-store panel names the store. But every `elasticsearch_bulk_processor_*`
metric — the **ES Bulk Processor Errors by HTTP Status**, **ES Bulk Processor
Queue Depth** and **ES Write Confirm Latency vs Ack Timeout** panels — is tagged
with the operation and HTTP status but **not** the index, so under dual
visibility both stores' processors land in one series. Those panels tell you
*that* an Elasticsearch store is unhappy and *how*; pair them with any per-store
panel to work out *which one*.

Two more things to check before you assume nothing user-facing is affected:

- `temporal workflow list` keeps working, because reads go to the primary. If
  you have moved any namespace's reads to the secondary with
  `system.enableReadFromSecondaryVisibility`, that namespace's reads **will**
  break — check before assuming reads are fine.
- The **Visibility Task Failures & Internal Errors** panel (visibility task errors) rises in both cases. This is often
  the first thing anyone notices, because it moves for a hung secondary when
  the error panels do not.

### 5.3 What a dead secondary costs the healthy primary

This is the part that surprises people. A dead secondary does **not** slow the
primary store down, and the primary's records are never missing — but the
primary ends up doing the same work over and over.

The confusing bit is the word "fails", so here is the actual sequence for one
workflow, with the secondary dead:

1. The visibility task runs. It sends the record to **both** stores at the same
   time.
2. **The primary stores it successfully.** The record is there. Anyone querying
   the primary can see it.
3. The secondary fails.
4. The **task** is marked failed, because a task only succeeds when both stores
   succeeded. This is a statement about the task, **not** about the primary's
   write — that already worked.
5. The task is retried from the start, which sends the record to both stores
   again.
6. The primary receives the record a second time. It already has it, with the
   same version number, so it **rejects** the write and keeps what it has.
7. The secondary fails again. Back to step 4.

That loop continues until the store recovers or the task dead-letters at roughly
70 attempts.

The short version: the primary wrote the record correctly on the first attempt,
and everything after that is the secondary's failure dragging the primary
through the same write again and again.

Two measurable consequences:

- **The primary absorbs the retry load.** Write request rate against the primary
  rises several times over baseline while the secondary is down, for a record it
  already stored correctly on attempt one. Every one of those retries is a real
  statement and a real round trip; they change nothing, because the version is
  not newer. Your data is safe, the load is not free, and on a busy cluster it is
  load the primary was never provisioned for.
- **Each attempt is held open for as long as the failing store takes.** A hung
  Elasticsearch secondary does not refuse the write, it simply never
  acknowledges it, so the attempt runs until the write ack timeout
  (`worker.ESProcessorAckTimeout`, default 30s) expires. Visibility task
  processing latency goes from under a second to the full timeout, per attempt —
  which also means the visibility queue drains far more slowly for **every**
  workflow, not just the ones being retried.

- **It can push the healthy store into its own rate limit.** Visibility writes
  are throttled per store by `system.visibilityPersistenceMaxWriteQPS` (default
  9000, applied per history host, with each store getting its own budget). The
  retry traffic counts against the **primary's** budget, so a long enough outage
  on a busy enough cluster can start throttling writes to the store that is
  perfectly healthy. Watch **Visibility Rate Limit Rejections per Store** for
  `resource_exhausted_cause=PersistenceLimit`.

  The good news is that throttling here is **not** a data-loss path. A rate
  limit rejection is classified as an *expected retryable* error, so it does not
  count toward the 70-attempt dead letter queue threshold — the task keeps
  retrying without consuming its budget. It costs latency, not records. Note
  also that this is a separate budget from `history.persistenceMaxQPS`, which
  governs the main execution store rather than visibility.

So the risk of dual visibility is not that one bad store takes the other down.
It is that a store nobody is reading from can multiply write traffic against
the store everybody is reading from — potentially into its rate limit — while
the only clear signal that it is happening is a panel most people are not
watching.

This is symmetric: a failing **primary** does the same thing to the secondary's
write budget. It just matters less, because nothing is reading the secondary.

#### Should you raise the write rate limit?

`system.visibilityPersistenceMaxWriteQPS` is read live on every request, so it
**can** be raised mid-incident with no restart. **Do not make that your first
move.**

The reasoning: the retry traffic is *pointless* work. Those writes are rejected
by the primary because their version is not newer, so raising the limit spends
real database capacity on writes that change nothing — and the throttle you
would be lifting is the thing protecting your database. Throttling also costs
you latency rather than records, because a rate limit rejection never counts
toward the dead letter queue threshold.

**Turn secondary writes off instead.** That removes the cause rather than
widening a safety margin:

```yaml
system.secondaryVisibilityWritingMode:
  - value: "off"
```

The primary immediately returns to normal write volume, visibility tasks start
succeeding again, and the queue drains. What you accept in exchange is a **gap
in the secondary with no dead letter queue record to replay** — the tasks now
succeed against the primary alone, so nothing is preserved for later. You will
need [7.7](#77-backfilling-a-store-that-is-missing-records) to close it. See
[5.5](#55-planned-maintenance-on-the-secondary).

So the real decision is between two costs, not between a fix and a workaround:

| | Leave writes on | Turn writes off |
|---|---|---|
| Primary write load | several times baseline | back to normal |
| Visibility queue | drains slowly for everyone | drains normally |
| Secondary's missing records | preserved in the dead letter queue for ~70 min, then dropped | silently missing from the moment you switch |
| Backfill needed afterwards | only for what dead-lettered | for the whole window |

Leave writes on if the outage will be short and you would rather not backfill.
Turn them off if it will be long, or if the primary is the store your users
depend on and it is showing strain.

**Raising the limit is the fallback**, for when you cannot accept the gap — a
migration you are mid-way through, for instance — *and* you have checked the
database has the headroom to absorb the extra writes. In that case raise it
deliberately and put it back afterwards.

### 5.4 How to remediate a failing secondary


1. Bring the secondary store back, the same way as in
   [section 4.3](#43-how-to-remediate-a-failing-primary).
2. Confirm the secondary's error line on the **Visibility Write Error Rate per Store** panel returns to zero, and
   that the **Visibility Errors by Type per Store** panel stops reporting errors against the secondary's index.
   Watch Errors by Type rather than Write Error Rate — if the store was hung
   rather than gone, Write Error Rate was never moving in the first place.
3. Confirm the **Visibility Task Failures & Internal Errors** panel returns to
   zero, and that task latency comes back down on **Visibility Task Processing by
   Operation** and **Visibility Task End-to-End Latencies**. (Do not look for
   latency on **Visibility Task Retry Depth** — that panel shows attempt counts,
   not latency, and it only has data for tasks past 30 attempts.)
4. Confirm the secondary caught up on its own. Nothing needs to be replayed by
   hand for the window where tasks were still retrying: once the store answers
   again, the retrying tasks succeed and the missing records appear within
   seconds.
5. Check the **Dead-Lettered Tasks — Informational** panel and replay anything that was dropped past the 70-minute
   mark — [section 3.6](#36-replaying-what-was-dropped).
6. Check the stores agree —
   [section 10](#10-after-the-outage--checking-the-stores-agree).

### 5.5 Planned maintenance on the secondary

If you are taking the secondary down deliberately, turn writes off first rather
than letting tasks pile up and time out:

```yaml
system.secondaryVisibilityWritingMode:
  - value: "off"
```

History stops retrying secondary writes within one config poll interval. Every
workflow that starts, updates or closes during the window will be missing from
the secondary, with no dead letter queue record to replay — the tasks complete
successfully against the primary alone. If the secondary is a migration target,
plan a backfill.

---

## 6. Scenario 3 — both stores fail

When both stores fail at once there is nowhere to move reads to, so neither of
the previous two scenarios helps. Your only job is to get a store back.

Both failing together usually points at something they share: the same database
host, the same Elasticsearch cluster, the same network segment, or one set of
credentials that expired. Work out what that is before you start fixing stores
one at a time — and note that with Elasticsearch
[option B](#if-both-stores-are-elasticsearch) this is the *only* scenario a
cluster-level problem can produce, because both indices live on one cluster.

### 6.1 What breaks when both stores fail

- Workflow execution: **unaffected**. Workflows keep running.
- All visibility reads: **broken**.
- Web UI workflow list: broken.
- Visibility task backlog: grows on every shard.
- Records: safe for about 70 minutes, then tasks start being dropped from
  **both** stores. See [section 3.1](#31-what-is-actually-lost-and-for-how-long).

### 6.2 How to detect both stores failing

- The **Visibility Write Error Rate per Store** panel: **both** store lines
  rise together.
- `temporal workflow list` and `count` fail (`describe` is unaffected — it does
  not read visibility). The exact error depends on how the stores are
  unreachable: a refused connection surfaces quickly as an unavailable error, a
  blackholed network path surfaces as a deadline exceeded.
- Frontend logs:

  ```json
  {"msg":"Operation failed with an error.",
   "error":"ListWorkflowExecutions operation failed.: dial tcp <host>:5432: connect: connection refused"}
  ```

  Note this is on the **frontend**, not history. The visibility store alerts
  filter on `service_name="history"`, so a read-path failure does not trigger
  them. Check the frontend explicitly.

- History logs: the same retry pattern as scenario 1, for both stores.

### 6.3 How to remediate both stores failing

**Start with the primary.** It is the one users can see, because it answers
list and count.

1. Restore the primary. Confirm the **Visibility Write Error Rate per Store**
   panel clears for it, then confirm `temporal workflow list` works.
2. Restore the secondary, and confirm that panel clears for it too.
3. Watch the **Visibility Write Latency per Store** and **Visibility Task
   End-to-End Latencies** panels drain.
4. Check the **Dead-Lettered Tasks — Informational** panel. With both stores
   down, dead letter queue writes are more likely than in the single-store
   scenarios. Replay per [section 3.6](#36-replaying-what-was-dropped).
5. Check the stores agree —
   [section 10](#10-after-the-outage--checking-the-stores-agree).

Do not reach for a write mode change here. With both stores down there is no
healthy store to route to, and every option loses records from somewhere.

---

## 7. Moving reads and writes between stores

This is the how-to for the three settings from
[1.2](#12-the-three-settings-that-decide-which-store-is-used). You need it in two
quite different situations: moving reads off a broken store during an incident,
and staging a planned migration from one store to another.

It runs in that order: how to write the YAML, then each setting in turn, then
what each change costs you, then the order to apply them in for a migration, and
finally how to backfill a store that is missing records.

All three keys are dynamic config. No restart. Changes land within one
`dynamicConfigClient.pollInterval` — a static config setting on each server,
which cannot be set below 5 seconds.

**Check your own value before you rely on "seconds".** The development configs
ship `10s`, but the container config template most self-hosted clusters actually
run ships **`60s`**. During an incident that is the difference between a read
switch taking effect almost immediately and taking up to a minute.

### 7.1 The YAML

Dynamic config is a YAML file on each server, at the path given by
`dynamicConfigClient.filepath`. Every key is a list of entries, and each entry is
a `value` plus optional `constraints`.

**A bare scalar does more damage than you would expect.** Writing
`system.secondaryVisibilityWritingMode: dual` instead of the list form is a
decode error, and one decode error discards the **entire file** — not just that
key. At startup the server fails config validation; on a live reload it logs
`Unable to update dynamic config.` and keeps the previous values. That is how a
config change silently never lands.

```yaml
# Write to both stores
system.secondaryVisibilityWritingMode:
  - value: "dual"

# Send reads for one namespace to the secondary store
system.enableReadFromSecondaryVisibility:
  - value: true
    constraints:
      namespace: my-namespace
  - value: false
```

Order does not matter. The most specific matching entry wins, so the
unconstrained `false` above is the default for every other namespace.

To read the current state, read the file on the servers. There is no API for it:

```bash
grep -A3 -E 'secondaryVisibilityWritingMode|enableReadFromSecondaryVisibility|visibilityEnableShadowReadMode' \
  /etc/temporal/config/dynamicconfig/production.yaml
```

Quoting `"off"`, `"on"` and `"dual"` is conventional and matches the shipped
example config. Unquoted also works — the parser reads them as strings, not
booleans.

### 7.2 Write mode

```yaml
system.secondaryVisibilityWritingMode:
  - value: "dual"
```

| Value | Primary gets writes | Secondary gets writes |
|-------|--------------------|-----------------------|
| `"off"` (default) | yes | no |
| `"on"` | **no** | yes |
| `"dual"` | yes | yes |

**`"on"` means secondary only.** It is not "also write to secondary" — that is
`"dual"`. Setting `"on"` stops writes to the primary completely. This is the most
common misreading of dual visibility, and it silently stops recording anything in whichever
store you thought was still being written.

Anything other than these three values is rejected and every visibility write
fails.

**Write mode is cluster-wide.** It cannot be scoped per namespace, and you
cannot stage a write mode change one namespace at a time.

If you add a `namespace` constraint to it, the entry is ignored — the server logs
a config warning and falls back to an unconstrained entry if the file has one.
**If it does not, the key falls back to its built-in default of `"off"`**, which
silently stops all writes to the secondary. So a constrained-only entry does not
just fail to do what you wanted; it does the opposite.

### 7.3 Read switch

```yaml
system.enableReadFromSecondaryVisibility:
  - value: true
    constraints:
      namespace: my-namespace
  - value: false
```

This setting can be applied per namespace, so you choose how much moves at once.
An entry with no constraints applies to **every** namespace, so you can move the
whole cluster's reads in one edit. Add a `namespace` constraint and only that
namespace moves. Both are normal — use an unconstrained entry for a cluster-wide
failover, and constrained entries to stage a migration namespace by namespace.

`true` sends that namespace's reads to the secondary and stops sending them to
the primary. There is no fallback: if the secondary is broken the reads fail, and
if it is merely incomplete the reads succeed and quietly return fewer rows.

It is read by frontend, history, worker **and** matching, so flipping it moves
more than user-facing list calls. Background work in the worker service that
reads visibility follows it too, and so do matching's worker versioning
lookups — build ID reachability, and reviving a build ID being removed from a
task queue. Flip it on a namespace you can afford to be wrong about first.

### 7.4 Shadow read

```yaml
system.visibilityEnableShadowReadMode:
  - value: true
```

**Short answer: leave this off, except during the one step of a migration
described below.**

#### What it actually does

With this on, every visibility read is sent **twice**. The real read goes to
whichever store reads currently point at, and its answer is what the caller gets,
exactly as normal. A second, identical read is then sent in the background to
the store that is **not** serving reads, and when it comes back the server
**drops it on the floor** — both the results and any error.

Which store gets shadowed is decided by the read switch, per namespace, and it
flips:

- Reads still on the primary (the default) — the **secondary** is shadowed.
- Reads already moved to the secondary with
  `system.enableReadFromSecondaryVisibility` — the **primary** is shadowed.

So shadow read always exercises the store you are *not* relying on, whichever
one that currently is.

That is what "discards the answer" means, and it is the part worth being clear
about: **nothing compares the two answers.** The second store's results are never
checked against the first, never merged, and never shown to anyone. If the other
store returns completely different rows, or no rows at all, nothing in the server
notices or says a word.

Its *errors* are a different matter. The shadow read goes through the same
per-store metrics layer as a real one, so a failing shadow read records its
store's error and latency metrics **and logs** — `Operation failed with an error.`
at error level, plus a slow-query warning past
`system.visibilityPersistenceSlowQueryThreshold`. That is the point of the
feature, but it can be confusing: while shadow read is on you will see error logs
naming a store that nobody is reading from.

The second read gets its own context rather than the caller's, so the real read
finishing does not cancel it and it runs to completion even after the caller has
been served. It does **not** get a longer budget — it inherits the same absolute
deadline.

#### So what is the point

The answer never matters. **The metrics do.** Because the second read goes through
the same machinery as a real one, it records that store's own
`visibility_persistence_latency`, `visibility_persistence_errors` and
`visibility_persistence_error_with_type` under that store's
`visibility_index_name` label.

**To confirm it is on**, watch the **Visibility Read Request Rate per Store**
panel. Normally only one store reports reads. With shadow read on, **both**
stores report, at the same rate — that is the signal, and it is the only
straightforward way to tell the setting took effect. Each store still reports
its own read latency separately, so the extra load is visible per store rather
than blended together.

That lets you answer one specific question before you trust a store with real
traffic: *can it actually handle the queries my users run, at the rate they run
them?* Query shapes that are slow or unsupported on the other store, and errors
that only appear under real load, show up in its metrics while none of it reaches
a single user.

What it cannot tell you is whether the other store's **data** is right. For that,
compare record counts —
[10.3 Do the record counts match](#103-do-the-record-counts-match).

#### When to turn it on, and what it costs

Turn it on for one step of a migration: after dual writing is on and the newer
store has been populated, but before you move any real reads to it. Watch that
store's error and latency metrics, fix what shows up, then turn it off again.

The costs, all of which argue for keeping the window short:

- **Visibility read load roughly doubles.** Every read now hits both stores.
- **It is whole-cluster only.** There is no way to shadow-read just one namespace,
  so you cannot limit the extra load to a namespace you are comfortable with.
- **It spends the other store's read rate limit.** Each store has its own
  `system.visibilityPersistenceMaxReadQPS` budget, and shadow reads consume the
  other store's. On a busy cluster that can push that store into throttling,
  which shows up on `visibility_persistence_resource_exhausted`.

There is no reason to run it permanently. It gives you no redundancy, no
fallback, and no correctness checking — only a way to watch a store work before
you depend on it.

### 7.5 What each change costs you

| Change | What it costs |
|--------|---------------|
| `"dual"` → `"off"` | Records written from now on are missing from the secondary. Tasks currently retrying against the secondary complete against the primary alone and are not dead-lettered — there is nothing to replay. |
| `"dual"` → `"on"` | Records written from now on are missing from the primary. Unlike a dropped task, nothing ever rebuilds these — no task failed, so there is nothing to replay. |
| `"off"` → `"dual"` | Nothing lost. Write load on the secondary starts immediately. Records from before this moment are not backfilled. |
| Read switch on | If the secondary is incomplete, that namespace sees an incomplete workflow list. No error, just missing rows. |

That second row is the one to be careful with during an incident. Turning
secondary writes off to stop the noise is easy, quiet, and unrecoverable.

---

### 7.6 Migration order

The safe sequence, one namespace at a time on the read steps:

1. `system.secondaryVisibilityWritingMode: "dual"`. Both stores now receive
   writes. Only new records land in the secondary — existing workflows are not
   backfilled.
2. Close the gap for workflows that finished before step 1 — refresh their tasks,
   or wait for retention to age them out. See
   [7.7 Backfilling a store that is missing records](#77-backfilling-a-store-that-is-missing-records).
3. `system.visibilityEnableShadowReadMode: true`. Watch
   `visibility_persistence_error_with_type` and `visibility_persistence_latency`
   for the secondary's `visibility_index_name`. Fix what surfaces.
4. Turn shadow read off. Set `system.enableReadFromSecondaryVisibility: true` for
   one namespace. Confirm list and count behave. Widen namespace by
   namespace.
5. Once every namespace reads from the secondary and you are ready to stop
   writing to the old store, `system.secondaryVisibilityWritingMode: "on"`.
6. Finally, make the new store the primary in static config, set write mode back
   to `"off"`, clear `system.enableReadFromSecondaryVisibility`, and restart the
   Temporal services. Dual visibility is now off.

   How you do that swap depends on which shape you configured. With two
   datastores, exchange `visibilityStore` and `secondaryVisibilityStore`. With
   one Elasticsearch datastore and two indices there is no
   `secondaryVisibilityStore` to exchange — swap the `visibility` and
   `secondary_visibility` index names inside that one datastore instead.

Step 6 matters. Leaving a cluster on `"on"` plus per-namespace read flags works,
but it means the store labelled "secondary" in config is the one actually serving
production, and every future operator has to work that out from dynamic config
before they can read a dashboard.

### 7.7 Backfilling a store that is missing records

You need this in two situations, and the mechanism is the same for both:

- **After turning on dual writing**, the second store is missing everything that
  finished before you enabled it.
- **After an outage**, a store is missing whatever was dropped past the
  70-minute mark — [4.5](#45-when-the-primary-comes-back).

Turning on dual writing only affects visibility tasks generated from that moment
on. Workflows that had already started keep writing to both stores from then on —
their next update or their close lands in both. But a workflow that started **and
finished** before dual writing was enabled generated its last visibility task
before the secondary existed, and nothing brings it back on its own.

There is a supported way to regenerate those records: refreshing a workflow's
tasks. A refresh rebuilds the workflow's tasks from its mutable state, and the
visibility task it regenerates writes a complete record — so in `dual` mode it
lands in **both** stores.

One workflow:

```bash
tdbg --namespace <ns> workflow refresh-tasks \
  --workflow-id <wid> --run-id <rid>
```

In bulk, driven by a visibility query. This starts a batch job rather than
running inline, and it is available from **v1.30.0** onward:

```bash
tdbg --namespace <ns> workflow refresh-tasks \
  --query 'CloseTime > "2026-01-01T00:00:00Z"' \
  --reason "backfill secondary visibility store"
```

It counts the matching workflows and asks you to confirm before starting. Pass
`--job-id` if you want a stable name for the job.

To run it non-interactively, pass **`--yes`**. Note that this is a *global* tdbg
flag, so it goes before the subcommand — `tdbg --yes --namespace <ns> workflow
refresh-tasks …`, not after `refresh-tasks`. Without it and with no terminal
attached (a `docker exec` without `-t`, for instance) the confirmation prompt
**panics on EOF** rather than failing cleanly.

#### The batch form is slower than you expect

This starts an **admin batch operation**, which runs as a workflow on the
per-namespace worker. Admin batches are throttled differently from the ordinary
`temporal workflow terminate --query` style batch operations, and the setting
people reach for first is the wrong one:

| Setting | Default | What it limits |
|---|---|---|
| `worker.adminBatcherHostRPS` | **100** | Workflows per second — **this is the one that governs a refresh.** Shared across every admin batch operation on that worker host |
| `worker.adminBatcherGlobalRPS` | **0** | Cluster-wide cap. `0` means none, and each host uses the host limit above; set above zero and it is divided by the number of worker hosts |
| `worker.batcherConcurrency` | **5** | Parallel workers inside one batch job |
| `frontend.MaxConcurrentAdminBatchOperationPerNamespace` | **1** | Admin batch jobs running at once, **per namespace** |
| `worker.batcherRPS` | 50 | **Does not apply here.** This governs *user* batch operations only |

That last row matters: `worker.batcherRPS` is the documented rate limit for batch
operations and it is the natural thing to raise, but the admin path never reads
it. Raising it does nothing for a refresh.

Two consequences worth planning around:

- **You cannot parallelise by starting several jobs.** With the concurrency limit
  at 1, a second admin batch operation in the same namespace is **rejected**, not
  queued — the frontend returns a resource-exhausted error reading
  `Max concurrent admin batch operations is reached`. Run them one after another,
  or across different namespaces. Be aware the check is itself a visibility
  count, so it is eventually consistent: two jobs fired back to back can both
  slip past a limit of 1.
- **100 workflows per second per worker host sets the schedule.** That is about
  360,000 workflows an hour, so a 100,000-workflow backfill takes roughly 20
  minutes and a million takes about three hours. If your repair window is bounded
  by retention, work that out before you start rather than discovering it halfway.

All of these are dynamic config and take effect without a restart, so they can be
raised for the duration of a backfill. Raise them deliberately: every extra
workflow per second is a read and a write against your main persistence store
plus writes against **both** visibility stores, per
[what a refresh costs your main database](#what-a-refresh-costs-your-main-database).
Watch the visibility write panels and your database while you do it, and put the
values back afterwards.

**Read the caveats before you run this in bulk.**

- **It refreshes every task category, not just visibility.** Timers, activity
  tasks, child workflow tasks, pending cancel and signal requests, the close
  transfer task and the **archival task** are all regenerated too. On closed
  workflows with archival enabled, a bulk refresh can re-queue archival work.
  This is a real operation with side effects, not a reindex.
- **It writes to both stores, not just the incomplete one.** There is no way to
  target a single store. Expect the write load on your healthy store to rise by
  the same amount. Re-writes are safe — the regenerated task carries a newer
  version, so it wins cleanly and nothing is corrupted — but the load is real.
- **It can only rebuild what still has mutable state.** Workflows already aged
  out by retention are gone from history as well, so there is nothing to refresh.
  For those, the gap closes by itself once retention ages them out of the older
  store too.
- **You have to be able to query for the workflows first, twice over.** The
  batch form takes a visibility query, and that query is served by whichever
  store reads currently point at. Worse, the backfill runs **two** visibility
  counts before it does any work at all: `tdbg` counts the matching workflows to
  build its confirmation prompt, and the frontend runs its own count to enforce
  the one-job-per-namespace limit. If reads point at the broken store, the
  backfill cannot even start. This is the concrete reason to leave reads on the
  surviving store until the backfill is finished.

#### What refreshing does to a running workflow

Bounded queries usually match running workflows as well as closed ones, so this
matters. Refreshing rebuilds a workflow's tasks from its current state, and for
a running workflow that means:

- **Work already in flight is left alone.** An activity that has already
  started is skipped entirely, so a refresh never re-runs an activity a worker
  is executing right now. Paused activities are skipped too.
- **Work that is scheduled but not yet started is dispatched again.** The
  refresh generates a fresh task for it, so for a short window there can be two
  task-queue entries for the same activity or workflow task. The duplicate is
  rejected when a worker tries to start it, so it does not cause double
  execution — but it is extra task-queue traffic.
- **Timers are regenerated**, at their original fire times. Nothing fires
  earlier than it would have.
- **Each workflow is loaded from the database and written back.** This is the
  part that surprises people, so it is worth spelling out below.

So the honest summary is: **correctness is not at risk, throughput is.** Nothing
is re-executed and nothing is corrupted, but a large refresh puts load on
persistence and on your task queues at the same time as it doubles visibility
write traffic.

#### What a refresh costs your main database

A refresh works from the workflow's **mutable state**, so it has to have that
state in memory. Per workflow, that means:

1. A **read of mutable state** — a `GetWorkflowExecution` against your **main**
   persistence store, not the visibility store — but **only if the workflow is
   not already in the history cache.** A cache hit skips this entirely.
2. A **write of mutable state** back, to persist the regenerated tasks.

**On SQL, that read is not one query.** The SQL implementation of
`GetWorkflowExecution` issues **nine** statements in sequence — the executions
row, then activity infos, timer infos, child execution infos, request cancel
infos, signal infos, buffered events, CHASM nodes and signals requested. So a
cache-missed workflow costs nine round trips, not one, and at the default
backfill rate of 50 workflows per second that is on the order of **450 reads per
second** against your main database purely for the loading, before any of the
writes.

**On Cassandra it is a single query.** Mutable state lives in one partition, so
the whole thing comes back in one round trip and the maps are unpacked in
memory. A bulk refresh is therefore substantially cheaper on Cassandra than on
SQL, and the SQL numbers above are the ones to plan against.

Two consequences for a bulk backfill:

- **Closed workflows are almost never cached**, so a backfill aimed at closed
  workflows is effectively one database read plus one write per workflow, with
  no cache to soften it.
- **It evicts live workflows from the cache.** The history cache is
  **host-level** and shared across every shard on the pod
  (`history.hostLevelCacheMaxSize`, 128,000 entries by default, or
  `history.hostLevelCacheMaxSizeBytes` when `history.cacheSizeBasedLimit` is
  on). Pulling a large batch of old workflows through it pushes out the running
  ones, which then take their own cache misses and their own extra reads. The
  cost of a big refresh is therefore not only the refresh itself — it is a
  period of degraded cache hit rate for ordinary traffic afterwards.

So a bulk refresh loads **three** things at once: reads and writes against your
main persistence store, writes against **both** visibility stores, and task
queue traffic for anything re-dispatched. Size the batch accordingly, and
prefer several narrow time windows over one wide one.

If you only want closed workflows — because you are chasing dropped close
records — bound the query on `CloseTime` instead of `StartTime` and running
workflows are excluded altogether.

Start with a narrow query against a small time window, confirm the records
appear in both stores, then widen. On a large cluster the alternative is often
cheaper: leave dual writing on and wait for retention to age out everything that
predates it, at which point the two stores agree on their own.

## 8. Why errors keep coming after the store is back

Errors do not stop the moment a store becomes reachable, and the reason is how
long the server waits between retries — not the database connection pool.

**SQL.** The connection pool reconnect throttle is **1 second**. A query against a
dead store triggers a reconnect attempt, and that attempt genuinely tries to
connect, so while the store is down each one fails and logs `sql handle: unable
to refresh database connection pool`. If two attempts land inside the same second,
the second one logs `sql handle: did not refresh database connection pool because
the last refresh was too close`. That line is normal noise, not a
30-second lockout.

Reconnects only happen when something actually runs a query. That is what sets
the delay: a visibility task that has been failing for a while is sitting at the
3-minute maximum wait, so after the store recovers it can be up to 3 minutes
before that task tries again, reconnects, and succeeds. Different tasks are at
different points in their own waits, so errors tail off gradually rather than
stopping all at once.

Useful signals during this window, with the panels that show them:

| Signal | Panel | Reading it |
|--------|-------|------------|
| `persistence_session_refresh_failures{failure="error"}` | **DB Pool Refresh Failure Rate per Pod** *DB Pool Refresh Failure Rate per Pod*, and **DB Pool Refresh Failure Ratio per Pod** for the same as a ratio of attempts — both in the **Shard Queue Health** row | Reconnect attempts that genuinely failed. Drops to zero once the store is reachable. |
| `persistence_session_refresh_failures{failure="throttle"}` | same panels | The one-second race between two attempts. Noise — do not chase it. |
| `persistence_sql_open_conn` | **SQL DB Connection Pool**, in the **Persistence Requests, Latencies and Errors** row | Sampled once a minute. **Do not read this as up-or-down:** if the store dies mid-flight the gauge keeps reporting its last value and looks healthy. It only goes absent if the pool was never established since that pod started. Use `persistence_session_refresh_failures{failure="error"}` for a real up-or-down signal. |

One caveat on the **DB Pool Refresh Failure Rate per Pod**, **DB Pool Refresh Failure Ratio per Pod** and **SQL DB Connection Pool** panels: they are not filtered by visibility store,
and the **SQL DB Connection Pool** panel sums across every pool a pod has, visibility and default alike. They
tell you a pod is struggling to reach *some* database, not which one. Pair them
with the **Visibility Write Error Rate per Store** panel or the per-store error query from
[2.3](#23-why-a-flat-write-error-rate-panel-is-not-proof-of-health) to attribute it.

**Elasticsearch.** Bulk processors run inside history and are not torn down when
ES goes away. Nothing needs restarting. Recovery is bounded by the same task
wait between retries, plus one flush interval.

**Field observation:** on SQL clusters, errors have been seen continuing for
30–60 seconds after the store pod reports ready. That is consistent with tasks
working through their retry waits, but the specific 30–60 second figure is not
something the server's own timers explain — nothing in the reconnect path
produces it. Treat it as a real observation with an unconfirmed cause, and go by
the metrics above rather than by a stopwatch.

In every case: **no server restart is needed.** Restarting history during
recovery resets those retry waits and makes errors appear to stop sooner, which
only changes how it looks, and it costs you shard reloads. Not worth it.

---

## 9. Search attributes while dual visibility is on

One more thing that changes once a second store exists, and it catches people
out during an outage rather than during normal operation.

Adding or removing a custom search attribute walks **both** stores — the primary
first, then the secondary — and returns the first error it hits. It ignores the
write mode setting entirely, so it behaves the same in `"off"` mode as in `dual`.

Whether that actually touches a visibility store depends on the store type and
on whether you are adding or removing, and the difference matters during an
outage:

| | Adding | Removing |
|---|---|---|
| **SQL** | No visibility store call. Custom attributes are an alias map on the namespace, held in the main database. | No visibility store call. |
| **Elasticsearch** | **Touches the store.** Each index gets a real mapping change, and the call waits for the cluster to go yellow. | No visibility store call — Elasticsearch mappings are never un-mapped. Only the cluster's own attribute metadata is updated. |

So there is exactly one case where a down store blocks you: **adding a search
attribute on Elasticsearch.** Everything else succeeds with either or both
visibility stores unreachable, because it only needs the main database.

Two more things worth knowing:

- **The stores are updated one after the other, not together.** A failure on the
  second leaves the first already changed. Re-running after fixing the store is
  safe — an attribute that already exists is skipped, on both store types.
- **Removal does not reach the secondary Elasticsearch index.** See below; this
  is a server bug, not something you can configure around.

#### Removing a search attribute misses the secondary index

> **Open server bug** — [temporalio/temporal#12126](https://github.com/temporalio/temporal/issues/12126).
> Confirmed on v1.31.0. Check the issue for whether a fix has shipped in your
> version before relying on the workaround below.

If you run two Elasticsearch stores with **different index names**, removing a
custom search attribute silently leaves it defined on the secondary index.

The reason is that the add path looks up the index name **per store**, while the
remove path looks it up from the dual manager — which always answers with the
**primary's** index name. So the removal runs against the primary's metadata
twice: the first pass removes the attribute, and the second pass finds nothing
left to remove and returns early. **The call reports success.**

What this means in practice:

- After a removal, the attribute is gone from the primary's metadata and still
  present on the secondary's.
- If you later promote that secondary to primary, the attribute reappears.
- There is no error and nothing on any dashboard to tell you.

**Verify against cluster metadata, not the CLI.** `temporal operator
search-attribute list` reads the primary too, so it agrees with the removal and
tells you nothing. The per-index attribute maps live in the cluster metadata
record; read those directly. And expect the problem to accumulate across a long
migration — every attribute removed during the window is still registered on the
store you are moving to.

**Workaround: swap the two stores, remove again, swap back.** The API has no way
to name an index — `RemoveSearchAttributes` takes only the attribute names and a
namespace — so the only lever is which store the server considers primary:

1. Exchange `visibilityStore` and `secondaryVisibilityStore` in static config and
   restart the frontends. The store that was missed is now the primary.
2. Run the same `temporal operator search-attribute remove` again. This time it
   applies to that store.
3. Put the config back and restart the frontends again.

This works — the attribute ends up removed from both stores' metadata. The cost
is two restarts, and dual writing continues throughout, so no records are lost
while you do it. On the single-cluster two-index layout, swap the `visibility`
and `secondary_visibility` index names instead.

**If you are mid-migration, the cheaper option is to wait.** Once you promote the
new store at the end of the migration it becomes the primary anyway, so running
the removal once more at that point cleans it up with no extra restarts.

This does **not** affect two SQL stores, where the attribute lives on the
namespace rather than per index, so both passes are operating on the same record
by design.

If you are planning a visibility store migration, add every custom search
attribute to **both** stores before you start moving reads, and remember that on
Elasticsearch the attributes are per cluster rather than per namespace — so a new
cluster needs them defined on it directly, not just registered in the namespace.

---

## 10. After the outage — checking the stores agree

A store being reachable again does not mean the two stores hold the same
records. Tasks that were still retrying will have caught up on their own, but
anything dropped past the 70-minute mark did not, and errors can keep appearing
for a while after recovery for reasons covered in
[section 8](#8-why-errors-keep-coming-after-the-store-is-back).

So before you call the incident closed, confirm the stores actually agree.
Check three things, in this order.

### 10.1 Did any visibility tasks get dropped

This first, because it is the only one that needs action beyond waiting. Set the
dashboard time range to cover the outage and read the **Visibility Tasks
Dead-Lettered by Task Type** panel.

Anything above zero means visibility tasks gave up during the window. On
Elasticsearch that does not necessarily mean the records are missing — check
before replaying, per
[3.1](#31-what-is-actually-lost-and-for-how-long). Replay them —
[section 3.6](#36-replaying-what-was-dropped). Replay is safe in any order and
safe to run even if some of those records have since been rebuilt by a later
task, so when in doubt, replay.

### 10.2 Has the backlog drained

Three panels, all at rest:

- **Visibility Write Error Rate per Store** — zero on both stores.
- **Visibility Errors by Type per Store** — zero on both stores. Check this one
  too, not just Write Error Rate, for the reasons in
  [2.3](#23-why-a-flat-write-error-rate-panel-is-not-proof-of-health).
- **Visibility Task End-to-End Latencies** — back at baseline.

Until all three are at rest, any count comparison will differ for reasons that
are not a problem.

### 10.3 Do the record counts match

Only meaningful once 10.2 is clean.

SQL store:

```sql
SELECT count(*) FROM executions_visibility;
```

On a large table this is a full scan. Bound it to the outage window instead —
the interval syntax differs by engine:

```sql
-- PostgreSQL
SELECT count(*) FROM executions_visibility
WHERE start_time > now() - interval '6 hours';
```

```sql
-- MySQL
SELECT count(*) FROM executions_visibility
WHERE start_time > now() - INTERVAL 6 HOUR;
```

One note on PostgreSQL: `start_time` is a timestamp without a time zone while
`now()` carries one, so the comparison is resolved in your session's time zone.
Harmless for a rough count, but it shifts the window if your session is not UTC.

Elasticsearch store:

```bash
curl -s "http://<es-host>:9200/<visibility-index>/_count" | jq .count
```

Bounded the same way:

```bash
curl -s "http://<es-host>:9200/<visibility-index>/_count" \
  -H 'Content-Type: application/json' -d '{
    "query": {"range": {"StartTime": {"gte": "now-6h"}}}
  }' | jq .count
```

A difference is expected and fine if any of these apply:

- The backlog has not finished draining.
- The cluster ran in `"off"` or `"on"` mode at some point — one store was not
  being written to.
- Dual visibility was turned on after workflows already existed, and you have not
  backfilled.
- Retention has aged records out of one store but not the other.

A difference with none of those explanations, after the error rates are clean,
means records were dropped. Go back to 10.1.

---

## 11. Reference

Lookup tables rather than reading material: every metric this playbook uses and
what it will and will not tell you, every dynamic config key with its default
and scope, and the panels, alerts and runbooks that go with all of it.

### 11.1 Metrics

| Metric | Type | Panel | Notes |
|--------|------|-------|-------|
| `visibility_persistence_requests` | counter | Visibility Write Request Rate, Visibility Read Request Rate | Counts attempts, including ones that then fail. Labels: `visibility_index_name`, `operation`, `visibility_plugin_name`, `service_name`. |
| `visibility_persistence_errors` | counter | Visibility Write Error Rate, Visibility Read Error Rate | **Does not count timeouts, rate limit rejections, not-found, invalid-argument or version-conflict (`ConditionFailedError`) errors.** Labels: `visibility_index_name`, `operation`, `visibility_plugin_name`, `service_name`. |
| `visibility_persistence_error_with_type` | counter | Visibility Errors by Type | Counts every error. The complete signal. Adds `error_type`. |
| `visibility_persistence_resource_exhausted` | counter | Visibility Rate Limit Rejections | Rate limit rejections. Adds `resource_exhausted_cause`, `resource_exhausted_scope`. |
| `visibility_persistence_latency` | histogram | Visibility Write Latency, Visibility Read Latency | Query the `_bucket` series with `histogram_quantile`. |
| `task_attempt` | **histogram** | Visibility Task Retry Depth | Recorded by **every** task when it completes, plus in flight once a task passes 30 attempts — so it is populated on a healthy cluster, sitting at about 1. Query `task_attempt_bucket`; there is no bare `task_attempt` series. Buckets are coarse — in the range that matters for the dead letter queue they step 1, 2, 5, 10, 20, 50, 100, so a task at 35 attempts reads as 50. Labels: `operation`, `namespace`, `task_type`, `archetype`. |
| `task_errors` | counter | Visibility Task Failures & Internal Errors | Unexpected task processing errors. The emitted name is `task_errors`; the Go constant is `TaskFailures`, which is easy to mistake for the metric name. |
| `task_errors_internal` | counter | Visibility Task Failures & Internal Errors | Internal-type errors. Rising on visibility operations with Write Request Rate at zero means a bad write-mode value. |
| `dlq_writes` | counter | Visibility Tasks Dead-Lettered | Above zero means visibility tasks were dropped. Which records stay missing depends on the task type. Labels: `operation`, `task_category`, `task_type`, `namespace`, `namespace_state`, `archetype`. |
| `task_latency_queue` | histogram | Visibility Task End-to-End Latencies | End-to-end visibility task latency. |
| `elasticsearch_bulk_processor_errors` | counter | ES Bulk Processor Errors | ES only, history only. **No `visibility_index_name`** — cannot tell two ES stores apart. Adds `http_status`. |
| `elasticsearch_bulk_processor_queued_requests` | **histogram** | ES Bulk Processor Queue Depth | ES only. Query the `_bucket` series. Rising means the processor is falling behind. |
| `elasticsearch_bulk_processor_request_latency` | histogram | ES Write Confirm Latency | ES only. Time from handing over a document to it being confirmed written. |
| `persistence_session_refresh_failures` | counter | DB Pool Refresh Failure Rate, DB Pool Refresh Failure Ratio | SQL only. `failure=error` is a real failure, `failure=throttle` is noise. |
| `persistence_sql_open_conn` | gauge | SQL DB Connection Pool | SQL only, sampled once a minute. Carries a `db_kind` label (`main` / `visibility`), so it **can** be attributed to the visibility pool even though the shipped panel sums it away. Holds its last value when the store dies rather than going absent, so it is not an up-or-down signal. |

Five of these are **histograms**, not gauges or counters, so they expose only
`_bucket`, `_sum` and `_count` series — querying the bare metric name returns
nothing at all. The panels above already account for that.

`operation` values for visibility writes: `RecordWorkflowExecutionStarted`,
`RecordWorkflowExecutionClosed`, `UpsertWorkflowExecution`,
`DeleteWorkflowExecution`. For reads: `ListWorkflowExecutions`,
`CountWorkflowExecutions`, `GetWorkflowExecution`, and the CHASM equivalents
`ListChasmExecutions` and `CountChasmExecutions`.

Visibility task types on `task_attempt` and `dlq_writes`:
`VisibilityTaskStartExecution`, `VisibilityTaskUpsertExecution`,
`VisibilityTaskCloseExecution`, `VisibilityTaskDeleteExecution`.

### 11.2 Dynamic config

Routing:

| Key | Scope | Default |
|-----|-------|---------|
| `system.secondaryVisibilityWritingMode` | cluster | `"off"` |
| `system.enableReadFromSecondaryVisibility` | namespace | `false` |
| `system.visibilityEnableShadowReadMode` | cluster | `false` |

Data loss window:

| Key | Scope | Default |
|-----|-------|---------|
| `history.TaskDLQEnabled` | cluster | `true` |
| `history.TaskDLQUnexpectedErrorAttempts` | cluster | `70` |

Both are read live on every attempt, so raising the attempt limit during an
outage takes effect on tasks that have not yet crossed it. Tasks already
dead-lettered are not brought back by raising it — replay those per
[section 3.6](#36-replaying-what-was-dropped).

Rate limits, applied **per store** — each store gets its own budget from the same
key:

| Key | Scope | Default |
|-----|-------|---------|
| `system.visibilityPersistenceMaxWriteQPS` | cluster | `9000` |
| `system.visibilityPersistenceMaxReadQPS` | cluster | `9000` |
| `system.visibilityPersistenceSlowQueryThreshold` | cluster | `1s` |

Elasticsearch write path, shared by both ES stores:

| Key | Scope | Default | Live? |
|-----|-------|---------|-------|
| `worker.ESProcessorAckTimeout` | cluster | `30s` | yes |
| `worker.ESProcessorFlushInterval` | cluster | `1s` | history restart |
| `worker.ESProcessorBulkActions` | cluster | `500` | history restart |
| `worker.ESProcessorBulkSize` | cluster | `16777216` | history restart |
| `worker.ESProcessorNumOfWorkers` | cluster | `2` | history restart |

Despite the `worker.` prefix these are read by the history service, and they apply
to both ES stores identically — there is no way to tune the two bulk processors
separately.

Backfill throughput, for the batch form of `refresh-tasks` — all live, no
restart:

| Key | Scope | Default |
|-----|-------|---------|
| `frontend.MaxConcurrentAdminBatchOperationPerNamespace` | namespace | `1` |
| `worker.batcherRPS` | namespace | `50` |
| `worker.batcherConcurrency` | namespace | `5` |
| `worker.adminBatcherHostRPS` | cluster (per host) | `100` |
| `worker.adminBatcherGlobalRPS` | cluster | `0` (no global limit) |

These are what decide how long a large backfill takes — see
[7.7](#77-backfilling-a-store-that-is-missing-records).

Visibility queue speed, if the backlog is draining too slowly:

| Key | Scope | Default |
|-----|-------|---------|
| `history.visibilityTaskBatchSize` | cluster | `100` |
| `history.visibilityProcessorMaxPollRPS` | cluster | `20` |
| `history.visibilityProcessorSchedulerWorkerCount` | cluster | `512` |
| `history.visibilityQueueMaxReaderCount` | cluster | `2` |

Elasticsearch read behavior:

| Key | Scope | Default |
|-----|-------|---------|
| `system.visibilityDisableOrderByClause` | namespace | `true` |
| `system.visibilityEnableManualPagination` | namespace | `true` |
| `system.visibilityAllowList` | namespace | `true` |

`system.visibilityAllowList` only takes effect on Elasticsearch. SQL visibility
does not support list-valued search attributes and the setting is forced off
there regardless of what you configure.

`history.visibilityProcessorEnableCloseWorkflowCleanup` is **Elasticsearch only**.
Turning it on with a SQL visibility store loses search attributes and memo when a
workflow closes. Since both visibility stores are always the same type, this is
safe to enable only when both of them are Elasticsearch.

### 11.3 Dashboards, alerts and runbooks

**Dashboard:** [Temporal Server Dashboard](../observability/dashboards/server/temporal-server-readme.md#16-visibility) — **v2.15.0 or later**, the **Visibility** section. Earlier versions have only the three write-side store panels; everything else this playbook uses was added in v2.15.0, with v2.15.1 and v2.15.2 correcting panel descriptions, so prefer the latest if you are choosing.

Every panel this playbook uses, with its **exact** dashboard title. **Grafana
does not display the ID column** — it is here only to match the dashboard JSON.
The prose above shortens the longer titles; these are the strings to search for.

| ID | Name | Stores | Used in |
|----|------|--------|---------|
| **2117** | Visibility Write Request Rate per Store | SQL + ES | [1.2](#12-the-three-settings-that-decide-which-store-is-used), [2.2](#22-the-one-label-that-matters), [2.1](#21-which-panels-to-check-first) |
| **2118** | Visibility Write Error Rate per Store | SQL + ES | [2.1](#21-which-panels-to-check-first), [2.3](#23-why-a-flat-write-error-rate-panel-is-not-proof-of-health), scenarios 1–3 |
| **2119** | Visibility Write Latency per Store | SQL + ES | [2.1](#21-which-panels-to-check-first), scenarios 1–3 |
| **2122** | Visibility Read Request Rate per Store | SQL + ES | [7.3](#73-read-switch), [7.4](#74-shadow-read) |
| **2123** | Visibility Read Error Rate per Store | SQL + ES | [6.2](#62-how-to-detect-both-stores-failing) |
| **2124** | Visibility Read Latency per Store | SQL + ES | [7.4](#74-shadow-read), [7.6](#76-migration-order) |
| **2125** | Visibility Errors by Type per Store | SQL + ES | [2.3](#23-why-a-flat-write-error-rate-panel-is-not-proof-of-health), [10.2](#102-has-the-backlog-drained) |
| **2126** | Visibility Rate Limit Rejections per Store | SQL + ES | [2.3](#23-why-a-flat-write-error-rate-panel-is-not-proof-of-health), [7.4](#74-shadow-read) |
| **2127** | Visibility Task Retry Depth (approaching DLQ) | SQL + ES | [3.3](#33-seeing-records-about-to-be-dropped) |
| **2128** | Visibility Tasks Dead-Lettered by Task Type | SQL + ES | [3.4](#34-confirming-records-were-already-dropped), [10.1](#101-did-any-visibility-tasks-get-dropped) |
| **2129** | Visibility Task Failures & Internal Errors | SQL + ES | [1.2](#12-the-three-settings-that-decide-which-store-is-used) |
| **2130** | ES Bulk Processor Errors by HTTP Status (Elasticsearch only) | **ES only** | [2.4](#24-detection-by-store-type) |
| **2131** | ES Bulk Processor Queue Depth (Elasticsearch only) | **ES only** | [1.5](#15-how-each-store-type-writes), [2.4](#24-detection-by-store-type) |
| **2132** | ES Write Confirm Latency vs Ack Timeout (Elasticsearch only) | **ES only** | [1.5](#15-how-each-store-type-writes), [2.4](#24-detection-by-store-type) |
| **82** | Visibility Task End-to-End Latencies | SQL + ES | [2.1](#21-which-panels-to-check-first), [10.2](#102-has-the-backlog-drained) |

Panels used from other groups:

| ID | Name | Section | Stores | Used in |
|----|------|---------|--------|---------|
| **2111** | DB Pool Refresh Failure Rate per Pod | Shard Queue Health | **SQL only** | [8](#8-why-errors-keep-coming-after-the-store-is-back) |
| **2112** | DB Pool Refresh Failure Ratio per Pod | Shard Queue Health | **SQL only** | [8](#8-why-errors-keep-coming-after-the-store-is-back) |
| **74** | SQL DB Connection Pool | Persistence Requests, Latencies and Errors | **SQL only** | [8](#8-why-errors-keep-coming-after-the-store-is-back) |

These three are not filtered by visibility store, and **SQL DB Connection Pool**
sums every connection pool a pod has. They tell you a pod cannot reach *some*
database, not which one — pair them with **Visibility Write Error Rate per
Store** or **Visibility Errors by Type per Store** to attribute it. The **Persistence
Health** row of the
[History Health Dashboard](../observability/dashboards/server/history-health-dashboard-readme.md)
carries equivalents at ids 37 and 35 if that is the dashboard you already have
open.

**Two panels that look relevant and are not.** In the **History Task DLQ /
Terminal Failures** row, the **Dead-Lettered Tasks — Informational** panel blends visibility with retention and
workflow-task-timeout operations, and the **Dead-Lettered Tasks — Execution-Stranding** panel — the page-worthy one —
matches only timer and transfer tasks, so it will never show visibility however
many records are being dropped. Use **Visibility Tasks Dead-Lettered by Task Type** instead.

**Known dashboard gap for two Elasticsearch stores:** the **ES Bulk Processor Errors by HTTP Status**, **ES Bulk Processor Queue Depth** and **ES Write Confirm Latency vs Ack Timeout** panels
carry no store label, so both Elasticsearch stores report into the same series
and cannot be told apart there. Use **Visibility Errors by Type per Store** for per-store attribution. This is a
limitation of the server metrics, not of the dashboard.

**Alerts:** [`observability/alerts/server/temporal-server-alerts.yaml`](../observability/alerts/server/temporal-server-alerts.yaml)

| Alert UID | Name | Fires when |
|-----------|------|------------|
| `temporal-alert-059a` | Visibility Store Write Errors (Warning) | `visibility_persistence_errors` > 0.1/s for 2m on either store |
| `temporal-alert-059b` | Visibility Store Write Errors (Critical) | `visibility_persistence_errors` > 1/s for 1m on either store |
| `temporal-alert-059c` | Visibility Store Write Latency High | p99 `visibility_persistence_latency` > 3s for 5m on either store |
| `temporal-alert-083` | Visibility Tasks Dead-Lettered | `dlq_writes` on any `VisibilityTask.*` operation > 0 for 5m |
| `temporal-alert-084` | Visibility Store Not Acknowledging Writes | `visibility_persistence_error_with_type` with `error_type=persistence_TimeoutError` > 0.1/s for 5m |
| `temporal-alert-085` | Visibility Read Errors | `visibility_persistence_errors` on read operations > 0.1/s for 2m, any service |

The three `059` alerts filter on `service_name="history"`, so they cover the
**write** path only, and they do not catch an Elasticsearch acknowledgement
timeout because that does not increment `visibility_persistence_errors`. Alerts
**084** and **085** exist to cover those two gaps, and **083** covers records
that have already been dropped.

Note that the general history task DLQ alert (`temporal-alert-080`) excludes
visibility operations by design, so it never fires for visibility — a quiet 080
proves nothing about visibility records.

**Alert runbooks:**
[59a — write errors](../observability/alerts/server/runbooks/59a-visibility-store-write-errors.md) ·
[83 — tasks dead-lettered](../observability/alerts/server/runbooks/83-visibility-tasks-dead-lettered.md) ·
[84 — not acknowledging writes](../observability/alerts/server/runbooks/84-visibility-store-not-acknowledging-writes.md) ·
[85 — read errors](../observability/alerts/server/runbooks/85-visibility-read-errors.md)

**Known server issues affecting dual visibility:**

| Issue | Affects | Summary |
|-------|---------|---------|
| [temporalio/temporal#12126](https://github.com/temporalio/temporal/issues/12126) | Elasticsearch pairs | `RemoveSearchAttributes` removes the attribute from the primary only, reports success, and the attribute reappears when the secondary is promoted — [9. Search attributes](#9-search-attributes-while-dual-visibility-is-on) |

**Related:**
[`dynamic-config/README.md`](../dynamic-config/README.md) ·
[`dynamic-config/troubleshooting.md`](../dynamic-config/troubleshooting.md)
