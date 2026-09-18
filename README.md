# fabric-sql-pools-config

A single Microsoft Fabric notebook that **inventories** — and optionally **enables** —
[custom SQL pools](https://learn.microsoft.com/en-us/rest/api/fabric/warehouse/sql-pools)
across every Fabric workspace you can see.

---

## The problem

Custom SQL pools let you control how much of a Fabric capacity a Warehouse workload is allowed to
consume, and whether that pool is optimised for reads. It is a very useful lever.

It is also a **per-workspace** setting, configured one workspace at a time. If you have three
workspaces, fine. If you have three hundred, you have a long afternoon ahead of you — and no
easy way to answer "which of my workspaces are still on the default?".

This notebook loops the whole thing, writes what it finds to Delta tables so you can query your
estate afterwards, and is deliberately hard to fire accidentally.

---

## What it does

| Phase | Notebook cell | API call | Effect |
|---|---|---|---|
| **Discover** | 3 | `GET /v1/workspaces` (paged) | Builds the in-scope workspace list from your filters |
| **Inventory** | 4 | `GET .../warehouses/sqlPoolsConfiguration?beta=True` | Read-only. Records current state to `sqlpools_configuration` |
| **Enable** | 5 | `PATCH .../warehouses/sqlPoolsConfiguration?beta=True` | **The only cell that changes anything.** Gated by `DRY_RUN` |
| **Query** | 6 | — | Registers `v_sqlpools_*` temp views and runs example queries |

API reference:
[GET sqlPoolsConfiguration (beta)](https://learn.microsoft.com/en-us/rest/api/fabric/warehouse/sql-pools/get-sql-pools-configuration(beta))
·
[PATCH sqlPoolsConfiguration (beta)](https://learn.microsoft.com/en-us/rest/api/fabric/warehouse/sql-pools/update-sql-pools-configuration(beta))

Every HTTP call — success or failure — is appended to an API log table. A `401`/`403` on a
workspace you are not an admin of is **recorded and skipped**, not fatal, so one bad workspace
cannot kill a tenant-wide run.

---

## Quick start

1. Import the notebook into a Fabric workspace.
2. **Attach a Lakehouse** to it (Explorer pane → Add data source), or set `TABLE_BASE_PATH`.
   The notebook raises a clear error at helper-load time if neither is set.
3. In the config cell, put **two or three test workspaces** in `ALLOW_LIST`.
4. Leave `DRY_RUN = True` and `FORCE_UPDATE = False`.
5. Run every cell. Read the printed plan and the `sqlpools_configuration` table.
6. Happy? Set `DRY_RUN = False` and re-run **cell 5** only.
7. Verify in the portal, *then* widen `ALLOW_LIST`.

> Just because you can loop over every workspace doesn't mean you should do it on a Friday
> afternoon.

---

## Configuration reference

Everything is in the **first code cell**. It is tagged `parameters`, so a Fabric pipeline's
notebook activity can override any of these values at run time without editing the notebook.
Nothing else in the notebook needs changing.

### The three settings that actually matter

#### `ALLOW_LIST` — what gets touched

```python
ALLOW_LIST = ["my-test-workspace"]
```

Accepts **workspace display names or workspace IDs**, matched case-insensitively (both are
trimmed and lower-cased before comparison, so `"My Workspace"` and `"my workspace"` are the same
thing).

| Value | Behaviour |
|---|---|
| `["ws-a", "ws-b"]` | **Only** those two workspaces are processed. Everything else is skipped. |
| `[]` (empty) | **Every workspace you can see** is processed. |

The shipped default is `["my-test-workspace"]` — a name that almost certainly matches nothing,
so a careless first run does nothing at all. That is on purpose.

This is your blast radius. Set it deliberately. An empty `ALLOW_LIST` combined with
`DRY_RUN = False` is a tenant-wide change.

#### `DRY_RUN` — the safety catch

```python
DRY_RUN = True   # default
```

`True` means **no `PATCH` is ever sent**. What it *still* does:

* reads the configuration of every in-scope workspace,
* builds and prints the exact JSON payload it would send,
* prints a per-workspace decision (`WOULD enable` / `already enabled - no change`),
* writes the full `sqlpools_configuration`, `sqlpools_api_log` and `sqlpools_enable_actions`
  tables, with `action = "dry-run-would-enable"` and `succeeded = null`.

So a dry run is genuinely useful on its own: it is how you get a complete inventory of your
estate without changing a single setting. **Run it like this first. Always.**

Set to `False` only when you have read the plan and agree with it.

#### `FORCE_UPDATE` — ⚠️ the one that breaks things

```python
FORCE_UPDATE = False   # default — leave it
```

| Value | What happens to a workspace that **already** has custom SQL pools |
|---|---|
| `False` | Nothing. Logged as `skipped-already-enabled`. |
| `True` | Its pool configuration is **replaced** by the single pool this notebook builds. |

That word *replaced* is the important one. The `PATCH` sends a complete `customSQLPools` array
containing exactly one pool, so any existing pools — including ones somebody sized carefully for
a production workload, with their own classifiers — are gone.

Do not enable this unless you know exactly which workspaces it will hit (i.e. you have a tight
`ALLOW_LIST`) and you are certain nobody has tuned those pools on purpose. Run it with
`DRY_RUN = True` first so you can see the `re-applied` decisions before they happen.

---

### Fabric REST API

| Setting | Default | Notes |
|---|---|---|
| `API_BASE` | `"https://api.fabric.microsoft.com/v1"` | Change only for a non-public cloud. |
| `BETA` | `True` | The SQL pools APIs are in **beta** and the `?beta=true` query string is mandatory. Setting this to `False` will make the calls fail. The API contract may change without notice. |

### Workspace scoping

| Setting | Default | Notes |
|---|---|---|
| `ALLOW_LIST` | `["my-test-workspace"]` | See above. Empty = everything. |
| `EXCLUDE_LIST` | `[]` | Names or IDs that are **always** skipped. Applied *after* the allow list, so exclude wins on a conflict. Useful for carving out known-good production workspaces from a broad run. |
| `INCLUDE_WORKSPACE_TYPES` | `["Workspace"]` | Only these workspace types are considered. `"Personal"` and `"AdminWorkspace"` cannot hold warehouses, so there is no reason to call the API for them. |
| `INCLUDE_CAPACITY_IDS` | `[]` | When non-empty, only workspaces on those capacity IDs are in scope. Empty = any capacity. Handy for rolling a change out one capacity at a time. |

**Filter precedence** (from `scope_reason()`) — the first rule that matches wins, and the reason
is recorded so you can see why something was skipped:

1. `INCLUDE_WORKSPACE_TYPES` — *note this is checked **before** the allow list*, so a Personal
   workspace named in `ALLOW_LIST` is still skipped.
2. `ALLOW_LIST` (if non-empty)
3. `EXCLUDE_LIST`
4. `INCLUDE_CAPACITY_IDS` (if non-empty)

The discovery cell prints `Workspaces visible / In scope / Skipped` plus the full in-scope list
before anything else runs. Read that number before you continue.

### Output tables

| Setting | Default | Notes |
|---|---|---|
| `INVENTORY_TABLE` | `"sqlpools_configuration"` | One row per workspace **per pool**. |
| `LOG_TABLE` | `"sqlpools_api_log"` | One row per HTTP call. |
| `ACTION_TABLE` | `"sqlpools_enable_actions"` | One row per workspace from the enable cell. |
| `TABLE_BASE_PATH` | `None` | `None` → managed tables in the notebook's **attached Lakehouse**. Or give an explicit OneLake Tables folder for external Delta tables, e.g. `"abfss://ws@onelake.dfs.fabric.microsoft.com/lh.Lakehouse/Tables"`. Either way the notebook registers `v_<table>` temp views so the example SQL works for both. |
| `WRITE_MODE` | `"append"` | `append` keeps the **full history** of every run — recommended, because each run is stamped with a `run_id` and `run_timestamp` and you can diff runs over time. `overwrite` keeps only the latest run. Writes use `mergeSchema` (and `overwriteSchema` when overwriting), so a future schema change won't break an existing table. |

### Pool settings applied by the enable cell

These build the `PATCH` payload:

| Setting | Default | Notes |
|---|---|---|
| `CUSTOM_SQL_POOLS_ENABLED` | `True` | Sets `customSQLPoolsEnabled` on the workspace. |
| `POOL_NAME` | `"defaultpool"` | Name of the single pool it creates. |
| `POOL_IS_DEFAULT` | `True` | Marks it as the workspace default pool. |
| `POOL_MAX_RESOURCE_PERCENTAGE` | `15` | The **default size** given to workspaces that had no custom pools — the share of capacity this pool may consume. |
| `POOL_OPTIMIZE_FOR_READS` | `False` | Sets `optimizeForReads`. `False` is the point of the exercise for most people. |
| `POOL_CLASSIFIER` | `None` | Optional override, e.g. `{"type": "Application Name", "value": ["ETL", "Load"]}`. |

Resulting payload:

```json
{
  "customSQLPoolsEnabled": true,
  "customSQLPools": [
    {
      "name": "defaultpool",
      "isDefault": true,
      "maxResourcePercentage": 15,
      "optimizeForReads": false,
      "classifier": { "type": "Application Name", "value": [] }
    }
  ]
}
```

The notebook prints this payload before it does anything, so you can eyeball it.

> **The empty `classifier` block is deliberate.** `build_update_payload()` always emits
> `{"type": "Application Name", "value": []}` even when `POOL_CLASSIFIER` is `None`.
> Leave it alone — set `POOL_CLASSIFIER` if you want a real classifier.

### HTTP behaviour

| Setting | Default | Notes |
|---|---|---|
| `MAX_RETRIES` | `5` | Attempts per call. |
| `RETRY_BACKOFF_SECONDS` | `5` | Base for linear backoff (`n × attempt`), used when the response has no `Retry-After` header. |
| `REQUEST_TIMEOUT` | `60` | Per-request timeout in seconds. |
| `MAX_LOGGED_BODY_CHARS` | `8000` | Response bodies are truncated to this before being stored, so one enormous error payload can't bloat the log table. |

Retry rules: `429` and `5xx` are retried (honouring `Retry-After` when present); a `401` forces a
token refresh and retries; network errors and timeouts retry with backoff. Anything else returns
immediately and is logged.

---

## What happens to each workspace

The enable cell records exactly one `action` per in-scope workspace:

| Situation | `action` | `succeeded` | API called |
|---|---|---|---|
| The `GET` failed (e.g. you're not admin) | `read-failed` | `false` | GET only |
| Pools already enabled, `FORCE_UPDATE = False` | `skipped-already-enabled` | `true` | GET only |
| Would change something, but `DRY_RUN = True` | `dry-run-would-enable` | `null` | GET only |
| Pools were **not** enabled → now enabled | `enabled` | `true` | GET + PATCH |
| Pools **were** enabled, `FORCE_UPDATE = True` | `re-applied` | `true` | GET + PATCH |
| The `PATCH` was rejected | `update-failed` | `false` | GET + PATCH |

Each row also captures `enabled_before`, `pool_count_before`, the exact `request_body`, the HTTP
status, the error code/message and the response body — so the table is a complete audit trail of
what you did and why.

---

## Output table schemas

**`sqlpools_configuration`** — one row per pool (or a single row with null pool columns when a
workspace has none):

`run_id`, `run_timestamp`, `workspace_id`, `workspace_name`, `workspace_type`, `capacity_id`,
`status_code`, `succeeded`, `custom_sql_pools_enabled`, `pool_count`, `pool_name`,
`pool_is_default`, `pool_max_resource_percentage`, `pool_optimize_for_reads`, `classifier_type`,
`classifier_values`, `error_code`, `error_message`, `raw_response`

**`sqlpools_api_log`** — one row per HTTP call:

`log_id`, `run_id`, `run_timestamp`, `logged_at`, `workspace_id`, `workspace_name`, `operation`,
`method`, `url`, `request_body`, `status_code`, `success`, `attempts`, `duration_ms`,
`error_code`, `error_message`, `response_body`

**`sqlpools_enable_actions`** — one row per workspace from the enable cell:

`run_id`, `run_timestamp`, `actioned_at`, `workspace_id`, `workspace_name`, `dry_run`,
`force_update`, `enabled_before`, `pool_count_before`, `action`, `request_body`, `status_code`,
`succeeded`, `error_code`, `error_message`, `response_body`

---

## Querying the results

Ordinary Delta tables — query them from the notebook, the Lakehouse SQL analytics endpoint, or a
semantic model. `register_views()` exposes them as `v_sqlpools_configuration`,
`v_sqlpools_api_log` and `v_sqlpools_enable_actions`.

```sql
-- Which workspaces are still on the default (no custom SQL pools)?
SELECT workspace_name, custom_sql_pools_enabled, status_code, error_code
FROM   v_sqlpools_configuration
WHERE  run_timestamp = (SELECT MAX(run_timestamp) FROM v_sqlpools_configuration)
  AND  COALESCE(custom_sql_pools_enabled, false) = false
ORDER  BY workspace_name;

-- Pools still optimised for reads
SELECT workspace_name, pool_name, pool_max_resource_percentage, run_timestamp
FROM   v_sqlpools_configuration
WHERE  pool_optimize_for_reads = true
ORDER  BY run_timestamp DESC, workspace_name;

-- Everything that failed, across all runs
SELECT logged_at, workspace_name, operation, method, status_code, error_code, error_message
FROM   v_sqlpools_api_log
WHERE  success = false
ORDER  BY logged_at DESC;
```

---

## Requirements

* A Fabric workspace and a Spark notebook (PySpark).
* A **Lakehouse attached** to the notebook, or `TABLE_BASE_PATH` pointing at a OneLake `Tables`
  folder.
* The **Admin** role on every workspace you want to read or change. The API returns `401`/`403`
  otherwise — logged, not fatal, so partial coverage is normal on a large tenant. Check the
  `read-failed` rows afterwards.
* The SQL pools APIs are **beta**. Treat the shape of the payload as subject to change.

Auth uses the notebook's own identity via `notebookutils.credentials.getToken("pbi")` (falling
back to `mssparkutils`) — **no secrets, no app registration, nothing to store.** The token is
cached for 30 minutes and refreshed automatically on a `401`.

---

## Troubleshooting

| Symptom | Cause / fix |
|---|---|
| `RuntimeError: No default Lakehouse is attached` | Attach one in the Explorer pane, or set `TABLE_BASE_PATH`. |
| `In scope: 0` | Your `ALLOW_LIST` doesn't match anything. It ships with a placeholder — change it. Remember names are matched case-insensitively against `displayName`. |
| Lots of `read-failed` / `401` / `403` | You aren't a workspace Admin on those workspaces. Expected on a big tenant. |
| A workspace in `ALLOW_LIST` was skipped anyway | `INCLUDE_WORKSPACE_TYPES` is evaluated **first** — Personal workspaces never make it through. Check the skip reason. |
| Nothing changed after setting `DRY_RUN = False` | You need to re-run **cell 5**; the config cell alone doesn't do anything. |
| `Could not list workspaces` | The very first `GET /v1/workspaces` failed — this one *is* fatal. Check the token / that you can reach the Fabric API. |

---

## Licence

MIT — see [LICENSE](LICENSE). Provided as-is; test it on workspaces you can afford to break.
