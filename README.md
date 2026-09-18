# fabric-sql-pools-config

A single Fabric notebook that **inventories** — and optionally **enables** — [custom SQL pools](https://learn.microsoft.com/en-us/rest/api/fabric/warehouse/sql-pools) across every Microsoft Fabric workspace you can see.

Custom SQL pools are a **per-workspace** setting. If you have more than a handful of workspaces,
setting them by hand in the portal gets old fast. This notebook does it in a loop, writes
everything it finds to Delta tables, and refuses to touch anything you haven't explicitly
pointed it at.

## What it does

1. **Discover** — pages `GET /v1/workspaces` and filters it with an allow list, an exclude list,
   a workspace-type filter and an optional capacity filter.
2. **Inventory** — calls
   [`GET .../warehouses/sqlPoolsConfiguration?beta=True`](https://learn.microsoft.com/en-us/rest/api/fabric/warehouse/sql-pools/get-sql-pools-configuration(beta))
   for each in-scope workspace and records whether custom SQL pools are enabled plus every pool's
   `optimizeForReads`, `maxResourcePercentage`, `isDefault` and `classifier`.
3. **Enable** *(optional)* — for workspaces that are **not** already using custom SQL pools, calls
   [`PATCH .../warehouses/sqlPoolsConfiguration?beta=True`](https://learn.microsoft.com/en-us/rest/api/fabric/warehouse/sql-pools/update-sql-pools-configuration(beta))
   to create one default pool with `optimizeForReads = false` and `maxResourcePercentage = 15`.

Workspaces that **already** have custom SQL pools configured are left alone (unless you set
`FORCE_UPDATE`). Every HTTP call — success or failure — is appended to an API log table, so a
`401`/`403` on a workspace you don't own is recorded and skipped rather than fatal.

## The settings that matter

All of them live in the first code cell (tagged `parameters`, so a Fabric pipeline can override
them).

| Setting | Default | What it does |
|---|---|---|
| `ALLOW_LIST` | `["my-test-workspace"]` | **Read this one.** Workspace names or IDs. When non-empty, these are the *only* workspaces processed. **Empty means every workspace you can see.** |
| `EXCLUDE_LIST` | `[]` | Always skipped, applied after the allow list. |
| `DRY_RUN` | `True` | **Changes nothing.** Still reads every workspace, still builds the payload, still prints the plan and still writes the inventory tables. Run it like this first. |
| `FORCE_UPDATE` | `False` | ⚠️ **Overwrites existing pool configuration** on workspaces that already have custom SQL pools. Don't enable this unless you are certain nobody has tuned those pools on purpose. |
| `POOL_OPTIMIZE_FOR_READS` | `False` | `optimizeForReads` on the pool it creates. |
| `POOL_MAX_RESOURCE_PERCENTAGE` | `15` | Default pool size applied to workspaces that had none. |
| `TABLE_BASE_PATH` | `None` | `None` = write managed tables into the attached Lakehouse. Or give an `abfss://.../Tables` folder for external Delta tables. |
| `WRITE_MODE` | `"append"` | `append` keeps run history; `overwrite` keeps only the latest run. |

## Suggested order of play

1. Attach a Lakehouse to the notebook.
2. Put **two or three test workspaces** in `ALLOW_LIST`, leave `DRY_RUN = True`, run the whole thing.
3. Read `sqlpools_configuration` and the printed plan. Check it's proposing what you expect.
4. Set `DRY_RUN = False` and run cell 5 against those same two or three workspaces.
5. Verify in the portal, *then* widen `ALLOW_LIST`.

Just because you can loop over every workspace doesn't mean you should do it on a Friday afternoon.

## Output tables

| Table | Contents |
|---|---|
| `sqlpools_configuration` | One row per workspace/pool: workspace name, enabled flag, pool details |
| `sqlpools_api_log` | One row per REST call: URL, request body, status code, error, duration |
| `sqlpools_enable_actions` | One row per workspace from the enable cell: decision + outcome |

They are ordinary Delta tables — query them from the notebook, the Lakehouse SQL analytics
endpoint, or a semantic model. The notebook also registers `v_sqlpools_*` temp views so the
example queries work for both managed tables and explicit `abfss` paths.

## Requirements

* A Fabric workspace with a **Lakehouse** attached to the notebook (or set `TABLE_BASE_PATH`).
* The **Admin** workspace role on each workspace you want to read or change.
* The SQL pools APIs are currently **beta** — `beta=True` is mandatory and the contract may change.

## Notes

* `build_update_payload()` always sends a `classifier` block
  (`{"type": "Application Name", "value": []}`) even when `POOL_CLASSIFIER` is `None`.
  **This is deliberate** — leave it alone. Set `POOL_CLASSIFIER` to override it, e.g.
  `{"type": "Application Name", "value": ["ETL", "Load"]}`.
* Auth uses the notebook identity via `notebookutils.credentials.getToken("pbi")` — no secrets,
  no app registration.
* `429` and `5xx` responses are retried with backoff (honouring `Retry-After`); `401` triggers one
  token refresh.
* Logged response bodies are truncated to `MAX_LOGGED_BODY_CHARS` so one huge error can't bloat
  the log table.

## Licence

MIT — see [LICENSE](LICENSE). Provided as-is; test it on workspaces you can afford to break.
