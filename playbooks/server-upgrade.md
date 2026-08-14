# Temporal Server Version Upgrade Playbook

This playbook covers upgrading a self-hosted Temporal Server cluster from one version to the next.

An upgrade involves two things:

- **The database schema** — the table definitions in Temporal's two databases (primary and visibility). A newer version may add tables or columns, so the schema is updated to match.
- **The server binary** — the Temporal Server itself (the new version), rolled out across its services: frontend, history, matching, and worker.

**The schema is always updated first, then the binary.**

This playbook covers the schema update and the binary rollout — for each persistence backend (Cassandra, MySQL, PostgreSQL), for SQL or Elasticsearch visibility, and for multi-cluster deployments.

---

## Quick Reference

For a single cluster, an upgrade is just five steps, done in order.

> **Update the schema before rolling the new binary.** At startup a server refuses to run unless the database schema is at least the version it expects (equal or newer). This isn't a restriction — it's what lets you roll the binary back after a schema upgrade if you ever need to.

| Step | Action |
|------|--------|
| 0 | [Check the prerequisites](#prerequisites) |
| 1 | [Update primary DB schema](#step-1-update-primary-db-schema) |
| 2 | [Update visibility schema](#step-2-update-visibility-schema) |
| 3 | [Roll out the new version](#step-3-roll-out-the-new-version) |
| 4 | [Verify](#step-4-verify) the upgrade succeeded |

For multi-cluster setups: upgrade [passive clusters first, then active](#multi-cluster-replication).

---

## Table of Contents

- [Quick Reference](#quick-reference)
- [Upgrades and Downtime](#upgrades-and-downtime)
  - [Things to consider](#things-to-consider)
  - [Things to avoid](#things-to-avoid)
- [Prerequisites](#prerequisites)
  - [Identify your persistence backends](#identify-your-persistence-backends)
  - [Obtain the schema tools](#obtain-the-schema-tools)
- [Plan Your Upgrade Path](#plan-your-upgrade-path)
- [Before You Upgrade](#before-you-upgrade)
- [Run the Upgrade](#run-the-upgrade)
  - [Step 1: Update Primary DB Schema](#step-1-update-primary-db-schema)
    - [Cassandra](#cassandra)
    - [MySQL](#mysql)
    - [PostgreSQL](#postgresql)
  - [Step 2: Update Visibility Schema](#step-2-update-visibility-schema)
    - [SQL Visibility (MySQL / PostgreSQL)](#sql-visibility-mysql--postgresql)
    - [Elasticsearch](#elasticsearch)
    - [Dual Visibility](#dual-visibility)
      - [SQL + SQL](#sql--sql)
      - [Elasticsearch + Elasticsearch](#elasticsearch--elasticsearch)
      - [General upgrade order](#general-upgrade-order)
  - [Confirm the Schema Updated](#confirm-the-schema-updated)
  - [Step 3: Roll Out the New Version](#step-3-roll-out-the-new-version)
    - [Pre-rollout dynamic config](#pre-rollout-dynamic-config)
      - [How these settings interact](#how-these-settings-interact-during-a-history-rolling-restart)
    - [Kubernetes: RollingUpdate, not Recreate](#kubernetes-rollingupdate-not-recreate)
    - [Schema check at startup](#schema-check-at-startup)
    - [Do not change numHistoryShards](#do-not-change-numhistoryshards)
    - [No prescribed service upgrade order](#no-prescribed-service-upgrade-order)
  - [Step 4: Verify](#step-4-verify)
    - [What to watch after rolling](#what-to-watch-after-rolling)
- [Rolling Back](#rolling-back)
- [Multi-Cluster Replication](#multi-cluster-replication)
  - [Recommended upgrade order](#recommended-upgrade-order)
- [Command Reference](#command-reference)
  - [temporal-cassandra-tool](#temporal-cassandra-tool)
  - [temporal-sql-tool](#temporal-sql-tool)
  - [temporal-elasticsearch-tool](#temporal-elasticsearch-tool-experimental)

---

## Upgrades and Downtime

Upgrading Temporal Server is an **in-place rolling upgrade**: within your existing cluster, you deploy the new version to each service — frontend, history, matching, and worker — replacing their instances a few at a time so the rest keep serving. Nothing changes on the client side — your applications keep connecting to the same Temporal frontend address, with no traffic split and no second cluster to stand up.

The upgrade is designed to introduce as little disruption as possible. During the roll some callers may see brief, retryable latency — not errors, in a healthy multi-replica setup. It is also designed to be safe for your data: running workflows keep executing across the upgrade, and no workflow executions are lost or reset. (Schema migrations change table structure, not your workflow data; for edge cases and recovery, see [Rolling Back](#rolling-back).)

### Things to consider

- Run at least two replicas of each Temporal service (frontend, history, matching, worker), so healthy replicas keep serving while others restart.
- The history service is the main source of brief latency during a roll — shards move between instances as each one restarts. See [Pre-rollout dynamic config](#pre-rollout-dynamic-config) and [Kubernetes: RollingUpdate, not Recreate](#kubernetes-rollingupdate-not-recreate) for how to shrink and smooth that window.
- You can upgrade the services in any order — the server tolerates version differences between them during the roll (see [No prescribed service upgrade order](#no-prescribed-service-upgrade-order)).

### Things to avoid

- **Don't canary the server by splitting traffic across two versions.** Temporal Server is stateful infrastructure, not a stateless service — you upgrade it in place, not by routing a share of workflows to a separate new-version cluster.
- **Don't reach for multi-cluster replication to do an upgrade.** It's a different feature entirely — active-passive disaster recovery / high availability, not a way to run two server versions side by side. (If you already run multi-cluster, it has its own upgrade steps — see [Multi-Cluster Replication](#multi-cluster-replication).)

To build confidence in a target version before production, validate it on a staging cluster (see [Before You Upgrade](#before-you-upgrade)) and move one minor version at a time (see [Plan Your Upgrade Path](#plan-your-upgrade-path)).

---

## Prerequisites

### Identify your persistence backends

Before starting, confirm what storage backends your cluster is using:

```bash
temporal operator cluster describe
```

```
ClusterName  PersistenceStore  VisibilityStore
active       postgres12        postgres12
```

`PersistenceStore` tells you your primary DB backend. `VisibilityStore` tells you your visibility backend. If dual visibility is configured, `VisibilityStore` shows two comma-separated names — always the same type (two SQL, or two Elasticsearch; a mixed SQL + Elasticsearch pairing is rejected at startup). See [Dual Visibility](#dual-visibility).

Use these values to determine which schema upgrade steps apply to your cluster.

### Obtain the schema tools

Three CLI tools ship with Temporal Server for schema management:

| Tool | Purpose |
|------|---------|
| `temporal-cassandra-tool` | Cassandra primary DB schema |
| `temporal-sql-tool` | MySQL / PostgreSQL primary and visibility DB schema |
| `temporal-elasticsearch-tool` | Elasticsearch visibility schema (EXPERIMENTAL) |

**Use the release binaries (recommended).** Each GitHub release ships a `temporal_<version>_<os>_<arch>` archive (Linux, macOS, Windows; amd64/arm64) that bundles all three schema tools and the server, built at that exact version. Download the archive matching your **target** server version and extract.

**Or build from source** — if a matching release binary is not published for your platform, or you are moving to an unreleased or custom build:

```bash
git clone https://github.com/temporalio/temporal.git
cd temporal
git checkout v<target-version>

make temporal-cassandra-tool
make temporal-sql-tool
make temporal-elasticsearch-tool
```

Binaries are written to the repository root.

> Either way, use the tool version that matches your **target** server version, not your current one.

The schema tools are standalone command-line programs, separate from `temporal-server`. Each one connects to the database, applies its migration, and exits — it does not start a server. Run it as a one-off from anywhere that can reach your database. You don't deploy or restart the server for this — the new server version isn't rolled out until [Step 3](#step-3-roll-out-the-new-version).

---

## Plan Your Upgrade Path

Upgrade one minor version at a time:

1. Upgrade to the latest patch of your current minor version first
2. Then move to the next minor version
3. Repeat until you reach your target version

For example, say you're on **v1.20.0** and want **v1.22.0**. Land on the latest patch of each minor as you go:

`v1.20.0` → `v1.20.5` → `v1.21.6` → `v1.22.0`

- `v1.20.0` → `v1.20.5` — the latest v1.20 patch (skips v1.20.1–v1.20.4)
- `v1.20.5` → `v1.21.6` — the next minor, at its latest patch (skips v1.21.0–v1.21.5)
- `v1.21.6` → `v1.22.0` — your target

You skip the in-between **patch** releases, but not the in-between **minor** (v1.21) — you pass through it once. Each step is one schema-then-binary upgrade.

The schema tool applies every intermediate schema version in one run — you don't run it once per version, and it always steps through each migration in order no matter how far you jump.

Still upgrade the **binary** one minor version at a time. The schema isn't the constraint — the tool fast-forwards through every intermediate version in one run. The reason for stepping is the binary: a version can do version-gated work at startup — background data migrations, feature sequencing — that a multi-minor jump would skip past. The server doesn't enforce this; it's a recommendation, but one minor at a time is the tested, supported path.

> **Recommended: skim the release notes for the versions you skip.** Skipping patches is the norm and usually safe, but a release note occasionally calls out a required intermediate step or a version that must not be skipped. If one does, follow it — it overrides the common path here.
>
> **PostgreSQL only: one visibility schema step can take a long time on a large database.** When you upgrade a large PostgreSQL visibility database to the 1.30 or 1.31 releases, one step rebuilds a whole table and can take a long time. This was made more efficient in v1.31.1, so upgrading straight to v1.31.1 or later is easier than stopping at 1.30. See the note in [Step 2](#sql-visibility-mysql--postgresql). Keep it in mind when you pick your target version and schedule the upgrade.

---

## Before You Upgrade

The steps below are recommendations, not requirements — a standard upgrade works without them, but they reduce the risk and impact of a production upgrade.

**Test the upgrade on a non-prod cluster first.** Run the whole procedure — schema migration, binary rollout, and a period of observation under load — before touching production. If you can, use a load-testing cluster and drive representative load through it, so version-specific regressions surface early.

**Pick a low-traffic window.** The roll can introduce brief latency (see [Upgrades and Downtime](#upgrades-and-downtime)), so schedule it when your workload is quiet — steer clear of known peaks like a weekly spike or a seasonal event. The lower the load, the smaller the business impact if anything hiccups.

---

## Run the Upgrade

Everything up to here was preparation. This is the upgrade itself, run in order: migrate the primary schema, migrate the visibility schema, confirm the schema took, roll the binary, then verify the upgrade came up healthy.

### Step 1: Update Primary DB Schema

Update the schema before you roll out the new server version.

> **Keep your existing cluster running while you apply the schema update.**

Each command below runs `update-schema`, which reads the current schema version from your database and applies every migration in order, up to the version your target server binary requires.

The `--schema-name` values (`cassandra/temporal`, `mysql/v8/temporal`, `postgresql/v12/temporal`, and their `.../visibility` counterparts) name the schema embedded in the tool. The `v8`/`v12` track the **database engine** (MySQL 8, PostgreSQL 12) and match your `--plugin` — they are *not* the Temporal version, so they don't change as you upgrade Temporal. If you're ever unsure of the exact value for your tool version, pass any `--schema-name` and the tool prints the valid list.

#### Cassandra

Build the tool from your target server version, then run:

```bash
temporal-cassandra-tool \
  --ep $CASSANDRA_HOST \
  --keyspace temporal \
  update-schema \
  --schema-name cassandra/temporal
```

See [`temporal-cassandra-tool`](#temporal-cassandra-tool) in the Command Reference for the full flag list.

#### MySQL

Build the tool from your target server version, then run:

```bash
temporal-sql-tool \
  --ep $SQL_HOST -p $SQL_PORT \
  --plugin mysql8 \
  --db temporal \
  update-schema \
  --schema-name mysql/v8/temporal
```

See [`temporal-sql-tool`](#temporal-sql-tool) in the Command Reference for the full flag list.

#### PostgreSQL

PostgreSQL schemas are embedded in the tool, so you can use `--schema-name` exactly as with MySQL and Cassandra:

```bash
temporal-sql-tool \
  --ep $SQL_HOST -p $SQL_PORT \
  --plugin postgres12 \
  --db temporal \
  update-schema \
  --schema-name postgresql/v12/temporal
```

`--schema-name` uses the schema files built into the tool. If you built from source instead, `--schema-dir ./schema/postgresql/v12/temporal/versioned` points at the same files on disk and does the same thing.

See [`temporal-sql-tool`](#temporal-sql-tool) in the Command Reference for the full flag list.

---

### Step 2: Update Visibility Schema

#### SQL Visibility (MySQL / PostgreSQL)

The visibility schema lives in its own database (`temporal_visibility`) and is managed by the same `temporal-sql-tool` — run `update-schema` against it with the visibility schema name.

MySQL:
```bash
temporal-sql-tool \
  --ep $SQL_HOST -p $SQL_PORT \
  --plugin mysql8 \
  --db temporal_visibility \
  update-schema \
  --schema-name mysql/v8/visibility
```

PostgreSQL:
```bash
temporal-sql-tool \
  --ep $SQL_HOST -p $SQL_PORT \
  --plugin postgres12 \
  --db temporal_visibility \
  update-schema \
  --schema-name postgresql/v12/visibility
```

> **PostgreSQL: on a large visibility table this step can take a long time — it's working, not stuck.** Some of the columns added in the 1.30 and later releases require PostgreSQL to rebuild the entire `executions_visibility` table. On a table with tens of millions of rows this can run for a long time — potentially much longer than you'd expect — and it uses a lot of database CPU while it runs. During the rebuild the table is locked, so searching or listing workflows may be slow or fail until it finishes — but running workflows keep going and no data is lost. Don't stop the migration partway through: it's making progress, and killing it throws the work away and starts over.
>
> This got more efficient in **v1.31.1**. The older v1.30 tool makes the changes in several separate passes, each one rewriting the whole table and locking it while it builds indexes. The v1.31.1 tool does it in a single pass and builds the new indexes without blocking writes. Since the migration comes from whichever tool you run, upgrading straight to v1.31.1 or later is easier on a large table than going through v1.30. (See temporalio/temporal PR #10371.)
>
> If your visibility database is large when you run the upgrade, treat this migration like a heavy maintenance task: run it during a scheduled maintenance window, and avoid starting it while production traffic is high — especially during workload spikes. One thing to check first: while it runs, the migration temporarily needs extra free space in the database (in the worst case, up to double the size of the visibility table), so make sure there's room before you start. This only affects PostgreSQL — MySQL doesn't rewrite the table for these changes.

#### Elasticsearch

> **Note:** `temporal-elasticsearch-tool` ships from server **v1.30.0** and is marked EXPERIMENTAL — but it's still the recommended way to apply visibility schema changes; prefer it over editing the index mappings by hand. If your target version is older than v1.30.0 the tool isn't available — apply the Elasticsearch index template/mappings using that release's method (the JSON files under `schema/elasticsearch/` via the ES API, or its auto-setup). The read-only detection script in [Confirm the Schema Updated](#confirm-the-schema-updated) works on any version, since it only reads the index mapping.

The `update-schema` command applies the full current index mappings to your existing visibility index in a single operation. All mapping changes are additive — no reindexing is required.

`$ES_VISIBILITY_INDEX` is the name of your Elasticsearch visibility index — the value of `elasticsearch.indices.visibility` in your server's persistence config (e.g. `temporal_visibility_v1`).

```bash
temporal-elasticsearch-tool \
  --endpoint $ES_SERVER \
  --user $ES_USER \
  --password $ES_PWD \
  update-schema \
  --index $ES_VISIBILITY_INDEX
```

With TLS:
```bash
temporal-elasticsearch-tool \
  --endpoint $ES_SERVER \
  --tls \
  --tls-ca-file /path/to/ca.crt \
  --tls-cert-file /path/to/client.crt \
  --tls-key-file /path/to/client.key \
  --user $ES_USER \
  --password $ES_PWD \
  update-schema \
  --index $ES_VISIBILITY_INDEX
```

With AWS authentication:
```bash
# aws-sdk-default
temporal-elasticsearch-tool \
  --endpoint $ES_SERVER \
  --aws-credentials aws-sdk-default \
  update-schema \
  --index $ES_VISIBILITY_INDEX
```

#### Dual Visibility

When `secondaryVisibilityStore` is configured, both the primary and secondary visibility stores must be upgraded before rolling the binary.

Valid dual visibility combinations (from server source):

| Primary | Secondary |
|---------|-----------|
| SQL (MySQL / PostgreSQL) | SQL (MySQL / PostgreSQL) |
| Elasticsearch | Elasticsearch (via `secondaryVisibilityStore`) |
| Elasticsearch | Elasticsearch (via `elasticsearch.indices.secondary_visibility` in same datastore) |

Mixed SQL+ES combinations are rejected at server startup.

##### SQL + SQL

Both visibility stores are separate databases, each requiring its own schema upgrade. Run `update-schema` against each:

```bash
# Primary visibility DB
temporal-sql-tool \
  --ep $SQL_HOST -p $SQL_PORT \
  --plugin mysql8 \
  --db temporal_visibility \
  update-schema --schema-name mysql/v8/visibility

# Secondary visibility DB (separate database)
temporal-sql-tool \
  --ep $SQL_HOST -p $SQL_PORT \
  --plugin mysql8 \
  --db temporal_visibility_secondary \
  update-schema --schema-name mysql/v8/visibility
```

For PostgreSQL use `--plugin postgres12` and `--schema-name postgresql/v12/visibility` for both.

The schema name is the same as for a single SQL visibility store — the two stores differ only by database. Use your real database names for `--db`: the primary is the `sql.databaseName` of the datastore named by `visibilityStore` in your server config, and the secondary is the `sql.databaseName` of the datastore named by `secondaryVisibilityStore`. (`temporal_visibility` / `temporal_visibility_secondary` above are just examples.)

##### Elasticsearch + Elasticsearch

Run `update-schema` once — it applies mappings to the index specified by `--index`. If using two separate ES datastores, run it twice with each index name:

```bash
temporal-elasticsearch-tool [global flags] update-schema --index $ES_PRIMARY_INDEX
temporal-elasticsearch-tool [global flags] update-schema --index $ES_SECONDARY_INDEX
```

If using the single-datastore `elasticsearch.indices.secondary_visibility` approach, both indices live in the same ES cluster — run `update-schema` for each index name.

Where these come from in your config: `$ES_PRIMARY_INDEX` is `elasticsearch.indices.visibility`; `$ES_SECONDARY_INDEX` is either `elasticsearch.indices.secondary_visibility` (single datastore) or the `indices.visibility` of the datastore named by `secondaryVisibilityStore` (separate datastores) — whichever your setup uses.

##### General upgrade order

Upgrade **both** visibility stores before rolling the binary — the order between them doesn't matter. Each is an independent database; the server only checks at startup that every store is at the version the new binary expects.

The following dynamic config keys control dual visibility behavior. Do not change these during the schema upgrade or binary rollout:

| Key | Values | Description |
|-----|--------|-------------|
| `system.secondaryVisibilityWritingMode` | `off` / `dual` / `on` | Controls which stores receive writes |
| `system.enableReadFromSecondaryVisibility` | `true` / `false` (per namespace) | Switches reads to secondary |
| `system.visibilityEnableShadowReadMode` | `true` / `false` | Async shadow reads to secondary |

If your cluster is currently in dual-write mode (`secondaryVisibilityWritingMode: "dual"`), both stores are receiving live writes. Both schemas must be updated before the binary rolls.

---

### Confirm the Schema Updated

Before rolling the binary, confirm every schema you updated is now at the target version. SQL and Cassandra store a version number you can query directly; Elasticsearch stores none, so you confirm its mappings instead.

**Primary DB — confirm schema version:**

Cassandra:
```cql
SELECT curr_version FROM schema_version WHERE keyspace_name = 'temporal';
```

Or use the health check command:
```bash
temporal-cassandra-tool --ep $CASSANDRA_HOST --keyspace temporal validate-health
```

MySQL / PostgreSQL (in the `schema_version` table, `curr_version` is keyed by `version_partition` and `db_name`; `db_name` is the value you passed to `--db`):
```sql
SELECT curr_version FROM schema_version WHERE version_partition = 0 AND db_name = 'temporal';
```

**Visibility DB (SQL) — same check on the visibility database:**
```sql
SELECT curr_version FROM schema_version WHERE version_partition = 0 AND db_name = 'temporal_visibility';
```

**Elasticsearch.** Elasticsearch stores no schema version number (unlike SQL and Cassandra), so you verify by confirming the current mappings are applied, not by reading a version. The straightforward way is the tool itself: re-run `update-schema --index $ES_VISIBILITY_INDEX` — it's idempotent, so a clean run means your index is at the target mappings. Check connectivity with `ping`:

```bash
temporal-elasticsearch-tool --endpoint $ES_SERVER ping
```

To eyeball which mapping fields are present — for example, to see where an index stands *before* upgrading — inspect the mapping directly. Set `ES_SERVER` and `INDEX` to your values, then paste the block below into a shell (it needs `curl` and `jq`); each version added a distinct field, so the highest one present indicates the version:

```bash
ES_SERVER=http://127.0.0.1:9200
INDEX=temporal_visibility_v1

FIELDS=$(curl -s "${ES_SERVER}/${INDEX}/_mapping" \
  | jq -r '.. | .properties? | keys? | .[]' 2>/dev/null)

check() { echo "$FIELDS" | grep -q "^$1$" && echo "yes" || echo "no"; }

echo "v2  (TemporalScheduledStartTime):          $(check TemporalScheduledStartTime)"
echo "v3  (TemporalNamespaceDivision):           $(check TemporalNamespaceDivision)"
echo "v4  (HistorySizeBytes):                    $(check HistorySizeBytes)"
echo "v5  (BuildIds):                            $(check BuildIds)"
echo "v6  (ParentWorkflowId):                    $(check ParentWorkflowId)"
echo "v7  (RootWorkflowId):                      $(check RootWorkflowId)"
echo "v8  (TemporalPauseInfo):                   $(check TemporalPauseInfo)"
echo "v9/v10 (TemporalWorkerDeploymentVersion):  $(check TemporalWorkerDeploymentVersion)"
echo "v11 (TemporalBool01):                      $(check TemporalBool01)"
echo "v12 (TemporalLowCardinalityKeyword01):     $(check TemporalLowCardinalityKeyword01)"
echo "v13 (TemporalUsedWorkerDeploymentVersions): $(check TemporalUsedWorkerDeploymentVersions)"
echo "v14 (TemporalExternalPayloadSizeBytes):    $(check TemporalExternalPayloadSizeBytes)"
```

Each line prints `yes` or `no` for whether that version's field is present. Your current version is the highest line showing `yes` (fields are cumulative). v9 and v10 can't be told apart this way — v10 added no new field, only a template reformat — so an index at either reports `v9/v10`.

> This field list is a point-in-time snapshot — it covers the ES schema versions Temporal defines today (v14 is the latest). Newer server versions add fields not listed here, so an all-`yes` result really means **v14 or newer**. For a current index, rely on the tool method above (re-run `update-schema`) — it's version-agnostic and never goes stale.

**Dual visibility — verify both stores:**

SQL + SQL: run the schema version query against both the primary and secondary visibility databases (use the `db_name` each was created with):
```sql
-- primary visibility DB
SELECT curr_version FROM schema_version WHERE version_partition = 0 AND db_name = 'temporal_visibility';

-- secondary visibility DB (connect to secondary DB and run same query)
SELECT curr_version FROM schema_version WHERE version_partition = 0 AND db_name = 'temporal_visibility_secondary';
```

Both must show the expected visibility schema version before proceeding.

ES + ES: run the detection script or `update-schema` (idempotent) for each index:
```bash
temporal-elasticsearch-tool [global flags] update-schema --index $ES_PRIMARY_INDEX
temporal-elasticsearch-tool [global flags] update-schema --index $ES_SECONDARY_INDEX
```

To confirm which stores are active and their names:
```bash
temporal operator cluster describe
```

`VisibilityStore` will show both store names comma-separated when dual visibility is configured.

---

### Step 3: Roll Out the New Version

#### Pre-rollout dynamic config

For a rolling upgrade, the setting that helps most is **aligned membership changes** on the history and matching services. It's a recommendation, not a requirement — both default to `0s`, and the upgrade works without them.

When a history or matching pod restarts, the work it owns moves to other pods — shards for history, task-queue partitions for matching — and that movement is the main source of restart latency. The real cost isn't the move itself, it's the window in which different pods disagree about who owns what. Without alignment, an ownership change propagates through the membership ring, which takes on the order of `500–1000ms`; during that window pods can bounce a shard or partition back and forth. Aligning membership changes has all pods apply the change on a shared clock boundary instead, so they converge within a few milliseconds (clock skew plus scheduling latency) — shrinking that disagreement window to near zero and largely eliminating the bouncing. Coalescing several simultaneous restarts into fewer rebalances is a real but secondary benefit.

```yaml
history.alignMembershipChange:
  - value: 10s
matching.alignMembershipChange:
  - value: 10s
```

On matching this is a well-established latency win during restarts; on history it's expected to help the same way. `10s` is a reasonable starting point — reset both to `0s` after the rollout if you don't want them left on.

**Caveat for the upgrade itself:** aligned changes only take effect once *every* running pod understands the setting. On the roll that first moves the fleet onto a version supporting it, the fleet is mixed, so it won't kick in until every pod is upgraded — and if you're coming from a version too old to have the setting at all, it does nothing on that hop. Turn it on, but expect the payoff on the *next* roll, not the one that introduces it.

A couple of related settings you can leave alone:

- **`shutdownDrainDuration` (history or matching) isn't needed once aligned changes are on** — on shutdown the pod takes the aligned-eviction path and the drain value is not used (see [How these settings interact](#how-these-settings-interact-during-a-history-rolling-restart)). No point setting both.
- **Don't set `history.startupMembershipJoinDelay` alongside aligned changes** — it's redundant with them and can work against them.
- **Frontend** owns no shards or partitions, so it has no aligned-changes setting. Its only knob is `frontend.shutdownDrainDuration`, which drains in-flight requests before the pod exits — a small, optional smoothing for frontend rolls.

##### How these settings interact during a history rolling restart

On **shutdown**, a history pod follows one of two strategies, and they're mutually exclusive:

- **Aligned eviction (`history.alignMembershipChange`)** — the recommended one. The pod schedules its eviction on a shared clock boundary, so its ownership change lands on the other pods within milliseconds instead of propagating over `500–1000ms` — that's what keeps shards from bouncing during the restart. Its shutdown wait is `alignment wait + history.shardLingerTimeLimit + history.shardFinalizerTimeout` (defaults `0s` / `2s`).
- **Plain drain (`history.shutdownDrainDuration`)** — the pod evicts from the ring immediately, then sleeps this long so in-flight requests drain before exit.

> If `history.alignMembershipChange > 0`, the pod takes the aligned-eviction path and **`history.shutdownDrainDuration` is not used** — setting both does not add them together. So with aligned changes on, leave the drain unset.

On **startup**, `history.startupMembershipJoinDelay` makes a starting pod wait before it joins the ring. With aligned changes on it's redundant and can even work against them, so leave it off.

Matching works the same way for its task-queue partitions (`matching.alignMembershipChange`); frontend owns nothing that moves, so none of this applies to it.

#### Kubernetes: RollingUpdate, not Recreate

Deploy the upgrade with the **`RollingUpdate`** strategy — it replaces pods incrementally. `Recreate` terminates every pod before starting the new ones, which is a full outage for that service — it defeats the whole point of a rolling upgrade, which is to keep the service serving throughout.

RollingUpdate on its own is enough for no-downtime (healthy replicas keep serving; the SDK retries the blips). The settings below make the history-side handoff cleaner:

| Setting | Where | Guidance |
|---|---|---|
| Replicas | Deployment | **≥ 2 per service**, so healthy pods keep serving while others cycle |
| `maxUnavailable` | RollingUpdate strategy | **`1`**, especially for history — cycle one pod at a time so shard movement stays to a single handoff |
| `terminationGracePeriodSeconds` | Pod spec (Kubernetes, default `30`) | Keep it **above the pod's shutdown wait** — the aligned-eviction wait (or the drain, if you use `shutdownDrainDuration`). If the grace period is shorter, Kubernetes SIGKILLs the pod mid-shutdown and the graceful handoff is lost |
| Readiness probe | Pod spec | Kubernetes only routes requests to pods reporting ready, so a shutting-down pod stops getting new work and can drain cleanly |

The pod's shutdown wait comes from the [Pre-rollout dynamic config](#pre-rollout-dynamic-config) settings (the aligned-eviction wait, or the drain). Keep `terminationGracePeriodSeconds` comfortably above whichever you use.

#### Schema check at startup

Every service instance verifies the schema version at startup, before it joins the cluster. If the installed schema is older than the binary expects, the instance refuses to start and logs a version-mismatch error — the prefix is `cassandra` or `sql` depending on your backend:

```
cassandra schema version compatibility check failed: version mismatch for keyspace/database: "temporal". Expected version: 1.13 cannot be greater than Actual version: 1.12
```

No data is affected — fix the schema with the appropriate tool and restart the instance.

#### Do not change numHistoryShards

Keep `persistence.numHistoryShards` unchanged in your static config across the upgrade. It's set when the cluster is first initialized and can't be changed afterward — at startup the server overrides any differing config value with the count stored in the database (and logs a warning). That guardrail matters: the shard count decides which shard every workflow lives on, so a count that actually took effect would misroute workflows.

#### No prescribed service upgrade order

The Temporal server source does not define or enforce a required order for upgrading the individual services (history, matching, frontend, worker) in a multi-process deployment — which one you upgrade first is up to you.

Do roll **one service at a time**, though: take a service through its full rolling restart (all its pods) and let it settle before starting the next, rather than interleaving pods from different services. That keeps the mixed-version surface small and lets you confirm each service is healthy before moving on. The server tolerates mixed versions, so this is good practice, not a hard rule.

A commonly used order is **frontend → worker → matching → history** — history last, since it's the most stateful service and the most disruptive to roll (its shard movement), so you upgrade it once everything else is already on the new version. This is convention, not a requirement — the server doesn't care about order.

---

### Step 4: Verify

Once the new binary is rolled out, confirm the upgrade came up healthy. (If a service refuses to start, that's almost always the schema check — see [Schema check at startup](#schema-check-at-startup).)

#### What to watch after rolling

After each pod restarts, and again once the roll is complete, confirm these settle:

| Metric | Healthy after the roll |
|---|---|
| `numshards_gauge` | Back to the full shard count, steady |
| `service_errors_shard_ownership_lost` | Brief spike during the roll, then ~0 |
| `service_latency` | Back to pre-upgrade baseline |
| `service_errors_resource_exhausted` | ~0 — a rise means throttling |
| `persistence_errors` | ~0 |
| `task_schedule_to_start_latency` | Back to baseline |
| `no_poller_tasks` | 0 on active task queues — non-zero means workers have not reconnected |

A sustained rise in any of these means the roll is not clean. Pause and investigate before restarting the next pod.

---

## Rolling Back

If the new version turns out to have a problem after rollout — a regression, a spike in errors, a bug that slipped past testing — and you need to revert, the recovery path is to **roll back the binary, not the schema**:

- **Schema is forward-only.** The tools only go forward (`setup-schema`, `update-schema`) — there's no down-migration, and you shouldn't reverse one by hand. Leave the schema at its upgraded version.
- **Roll the binary back to the previous version.** This is explicitly supported — an older binary boots fine against the already-upgraded schema, precisely so you can revert a server version after a schema upgrade.
- **No data inconsistency.** A binary-version rollback doesn't modify your workflow data, so it won't introduce inconsistencies.
- **Stay within one minor version.** Compatibility holds across adjacent minors, so roll back one minor at a time (see [Plan Your Upgrade Path](#plan-your-upgrade-path)).

---

## Multi-Cluster Replication

Each cluster in a multi-cluster deployment is an independent Temporal deployment — its own database, its own binaries. Run the full upgrade (both the schema and the binary) separately on each cluster; there's no shared database, and nothing propagates across clusters.

Keep server upgrades and namespace failovers/handovers separate — don't run a handover while a cluster upgrade is in progress. They're independent operations, and doing both at once makes any problem much harder to diagnose.

### Recommended upgrade order

Upgrade the passive cluster before the active. This keeps the replication stream's receiver at least as new as its sender, so the active never sends a replication task type the passive can't understand. (A newer version can add task types an older cluster wouldn't recognize and would silently drop — passive-first avoids that.)

1. Upgrade the **passive** cluster fully — schema, then binary.
2. Confirm replication is healthy before continuing — watch replication lag/latency settle back to normal.
3. Upgrade the **active** cluster fully — schema, then binary.

The server doesn't enforce this order; it's a recommendation.

---

## Command Reference

### temporal-cassandra-tool

**Global flags:**

| Flag | Env var | Default | Description |
|------|---------|---------|-------------|
| `--endpoint, --ep` | `$CASSANDRA_HOST` | `127.0.0.1` | Cassandra host |
| `--port, -p` | `$CASSANDRA_PORT` | `9042` | Port |
| `--user, -u` | `$CASSANDRA_USER` | | Username |
| `--password, --pw` | `$CASSANDRA_PASSWORD` | | Password |
| `--keyspace, -k` | `$CASSANDRA_KEYSPACE` | `temporal` | Keyspace name |
| `--datacenter, --dc` | `$CASSANDRA_DATACENTER` | | Datacenter name (enables NetworkTopologyStrategy) |
| `--timeout, -t` | `$CASSANDRA_TIMEOUT` | `30` | Request timeout in seconds |
| `--tls` | `$CASSANDRA_ENABLE_TLS` | | Enable TLS |
| `--tls-cert-file` | `$CASSANDRA_TLS_CERT` | | TLS certificate |
| `--tls-key-file` | `$CASSANDRA_TLS_KEY` | | TLS key |
| `--tls-ca-file` | `$CASSANDRA_TLS_CA` | | TLS CA certificate |
| `--tls-server-name` | `$CASSANDRA_TLS_SERVER_NAME` | | Override target server name |
| `--tls-disable-host-verification` | `$CASSANDRA_TLS_DISABLE_HOST_VERIFICATION` | | Disable TLS host name verification |
| `--allowed-authenticators, --aa` | `$CASSANDRA_ALLOWED_AUTHENTICATORS` | | Authenticators allowed by the gocql client |
| `--disable-initial-host-lookup` | `$CASSANDRA_DISABLE_INITIAL_HOST_LOOKUP` | | Connect only to supplied hosts, skip system.peers lookup |
| `--quiet, -q` | | | Don't set exit status to 1 on error |

**update-schema** — apply all migrations from current schema version up to the highest version in the embedded schema:

```bash
temporal-cassandra-tool [global flags] update-schema \
  --schema-name cassandra/temporal \  # use embedded versioned schema
  --schema-dir <path/versioned>       # or provide path to versioned dir
```

`--version x.x` is available to stop at a specific intermediate version but is not needed for normal upgrades.

**validate-health** — verify Cassandra connectivity:

```bash
temporal-cassandra-tool [global flags] validate-health
```

---

### temporal-sql-tool

**Global flags:**

| Flag | Env var | Default | Description |
|------|---------|---------|-------------|
| `--endpoint, --ep` | `$SQL_HOST` | `127.0.0.1` | SQL host |
| `--port, -p` | `$SQL_PORT` | `3306` | Port |
| `--user, -u` | `$SQL_USER` | | Username |
| `--password, --pw` | `$SQL_PASSWORD` | | Password |
| `--database, --db` | `$SQL_DATABASE` | `temporal` | Database name |
| `--plugin, --pl` | `$SQL_PLUGIN` | `mysql8` | Plugin: `mysql8` or `postgres12` |
| `--connect-attributes, --ca` | `$SQL_CONNECT_ATTRIBUTES` | | Arbitrary SQL connect attributes |
| `--tls` | `$SQL_TLS` | | Enable TLS |
| `--tls-cert-file` | `$SQL_TLS_CERT_FILE` | | TLS certificate |
| `--tls-key-file` | `$SQL_TLS_KEY_FILE` | | TLS key |
| `--tls-ca-file` | `$SQL_TLS_CA_FILE` | | TLS CA certificate |
| `--tls-server-name` | `$SQL_TLS_SERVER_NAME` | | Override target server name |
| `--tls-disable-host-verification` | `$SQL_TLS_DISABLE_HOST_VERIFICATION` | | Disable TLS host name verification |
| `--quiet, -q` | | | Don't set exit status to 1 on error |

**update-schema** — apply all migrations from the current schema version up to the highest available:

```bash
temporal-sql-tool [global flags] update-schema --schema-name mysql/v8/temporal
```

`--schema-name` accepts any of the embedded schemas — `mysql/v8/temporal`, `mysql/v8/visibility`, `postgresql/v12/temporal`, `postgresql/v12/visibility`. As an alternative, `--schema-dir <path/versioned>` points at a schema directory from a source checkout. `--version x.x` stops at a specific intermediate version (not needed for normal upgrades).

---

### temporal-elasticsearch-tool (EXPERIMENTAL)

**Global flags:**

| Flag | Env var | Default | Description |
|------|---------|---------|-------------|
| `--endpoint` | `$ES_SERVER` | `http://127.0.0.1:9200` | Elasticsearch host |
| `--user` | `$ES_USER` | | Username (or AWS access key ID for static credentials) |
| `--password` | `$ES_PWD` | | Password (or AWS secret access key for static credentials) |
| `--aws-credentials` | `$AWS_CREDENTIALS` | | AWS credentials provider: `static`, `environment`, `aws-sdk-default` |
| `--aws-session-token` | `$AWS_SESSION_TOKEN` | | AWS session token (for static provider) |
| `--tls` | `$ES_TLS` | | Enable TLS |
| `--tls-cert-file` | `$ES_TLS_CERT_FILE` | | TLS certificate |
| `--tls-key-file` | `$ES_TLS_KEY_FILE` | | TLS key |
| `--tls-ca-file` | `$ES_TLS_CA_FILE` | | TLS CA certificate |
| `--tls-server-name` | `$ES_TLS_SERVER_NAME` | | Override target server name |
| `--tls-disable-host-verification` | `$ES_TLS_DISABLE_HOST_VERIFICATION` | | Disable TLS host name verification |
| `--quiet` | | | Don't log errors to stderr |

**update-schema** — update index template and index mappings:

```bash
temporal-elasticsearch-tool [global flags] update-schema \
  --index $ES_VISIBILITY_INDEX \  # also updates index mappings when specified
  --fail                          # fail silently on HTTP errors
```

If `--index` is omitted, only the index template is updated. Specify `--index` to also apply current mappings to the existing index (required during upgrades).

**ping** — verify connectivity:

```bash
temporal-elasticsearch-tool [global flags] ping
```

