# Bronto MCP Tool Patterns

## Tool Metadata

Tool names, descriptions, and argument schemas are provided by the Bronto MCP server. Treat MCP tool descriptors as the source of truth; this file only documents agent-side investigation flow and known-good query shapes.

## Tool Catalog

| Category | Tools and purpose |
| -------- | ----------------- |
| Authentication | `mcp_auth` starts browser OAuth for the selected regional MCP server. Cursor injects this tool. |
| Discovery | `get_datasets` lists datasets with `active` and `last_heartbeat_at`; `get_datasets_by_name` filters by dataset and collection; `get_keys` and `get_key_values` inspect one dataset; `get_all_datasets_keys` returns common keys keyed by dataset ID. |
| Query | `timeseries` returns counts, rates, group-bys, percentiles, and trends; `search_logs` returns raw events; `get_error_summary` returns precomputed org-wide error/warning totals and affected datasets without consuming search quota. |
| Saved searches | `get_saved_searches` lists or fetches by ID; `run_saved_search` executes by ID; `create_saved_search` creates one or up to 20 with optional visualization and `dry_run`. |
| Monitors | `search_monitors` lists, filters, or inspects; `create_monitor` creates one or up to 20 muted `PATTERN`, `USAGE`, or `CHANGE_DETECTION` monitors with `dry_run`; `validate_monitors` checks whether existing queries work and have data. |
| Coverage and usage | `get_coverage` reports monitored, searched, and ingesting status per dataset; `get_usage` summarizes org ingestion and search activity. |

## Common Sequence

Default investigation:

1. `get_datasets`, or `get_datasets_by_name` when both names are known.
2. Pick active candidate dataset IDs or build `from_expr`.
3. `get_keys` on each dataset that will be queried.
4. `get_key_values` for dimensions before filtering.
5. `timeseries` with grouped aggregation.
6. `search_logs` for representative raw events.

For errors and warnings:

1. `get_error_summary` over the requested window.
2. Pick the affected dataset IDs from its per-dataset breakdown.
3. Continue with `get_keys` → `get_key_values` → `timeseries` → `search_logs`.

Use `get_all_datasets_keys` only for cross-dataset schema discovery. Field names and types must still be verified with `get_keys` for every dataset used in a field-specific query.

## Dataset Discovery

`get_datasets` accepts optional `from_expr` or `from` dataset IDs. These are mutually exclusive. Its `active` and `last_heartbeat_at` fields help exclude dormant datasets that cannot contain events in the requested window.

Use `get_datasets_by_name` when the collection and dataset names are already known.

## Saved Search Lifecycle

1. Call `get_saved_searches` to search existing entries or fetch one by ID.
2. Call `run_saved_search` to execute an existing query without rebuilding it.
3. Call `create_saved_search` only when persistence is requested. Use `dry_run` first.

Creation accepts one saved search or a batch of up to 20. A saved search can render as `list`, `timeseries`, `toplist`, `table`, `piechart`, `treemap`, `geomap`, or `queryvalue`. Verify dataset keys before adding filters, groups, or visualization dimensions.

## Monitor Lifecycle

1. Call `search_monitors` to find an existing equivalent or inspect one by ID.
2. Call `create_monitor` with `dry_run` before creating one or a batch of up to 20.
3. Call `validate_monitors` to distinguish broken queries from valid queries that currently have no data.

Choose the monitor type deliberately:

- `PATTERN` — threshold over a query.
- `USAGE` — volume or usage signal.
- `CHANGE_DETECTION` — anomalous change signal.

Every created monitor is muted. Tell the user it must be reviewed and unmuted in Bronto before it can alert.

## Coverage and Usage

- `get_coverage` finds live blind spots: datasets that are ingesting but unmonitored and unsearched.
- `get_usage` summarizes ingestion and search activity across the organization.
- Use `search_monitors` when monitor identity or configuration is needed after a coverage check.

## `timeseries` Shape

Use for summaries:
```json
{
  "log_ids": ["dataset-id"],
  "time_range": "Last 20 minutes",
  "metric_functions": ["COUNT(*)"],
  "search_filter": "\"status\" >= 500",
  "group_by_keys": ["host", "path", "status"],
  "limit": 20,
  "num_of_slices": 20
}
```

Use absolute time when comparing exact incident windows:

```json
{
  "log_ids": ["dataset-id"],
  "timerange_start": 1714900000000,
  "timerange_end": 1714901800000,
  "metric_functions": ["P95(duration_ms)", "COUNT(*)"],
  "group_by_keys": ["service"],
  "search_filter": "\"env\"='prod'"
}
```

## `search_logs` Shape

Use for raw evidence after narrowing:

```json
{
  "log_ids": ["dataset-id"],
  "time_range": "Last 20 minutes",
  "search_filter": "\"status\" >= 500 AND \"service\"='api'",
  "limit": 20,
  "order_by": "\"@timestamp\" DESC"
}
```

Use from_expr when dataset selection is tag/collection based:

```json
{
  "from_expr": "(\"collection\" IN ('prod-api'))",
  "time_range": "Last 10 minutes",
  "search_filter": "\"status\" >= 500",
  "limit": 50
}
```
