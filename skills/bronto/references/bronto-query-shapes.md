# Bronto Query Shapes

Use this reference to help coding agents translate between common Bronto search
concepts, Bronto REST API request payload fields, and Bronto MCP tool arguments.
MCP examples are payloads passed to MCP tools; API examples are request payloads
sent to the Bronto REST API.

## Naming Map

| Concept            | Bronto API field | Bronto MCP argument |
| ------------------ | ---------------- | ------------------- |
| Dataset IDs        | `from`           | `log_ids`           |
| Dataset expression | `from_expr`      | `from_expr`         |
| Filter string      | `where`          | `search_filter`     |
| Aggregates         | `select`         | `metric_functions`  |
| Grouping           | `groups`         | `group_by_keys`     |
| Relative time      | `time_range`     | `time_range`        |
| Absolute start     | `from_ts`        | `timerange_start`   |
| Absolute end       | `to_ts`          | `timerange_end`     |
| Result cap         | `limit`          | `limit`             |
| Time buckets       | `num_of_slices`  | `num_of_slices`     |
| Sort expression    | `order_by`       | `order_by`          |

Agent-facing examples should use MCP argument names when calling MCP tools. Use
API field names when explaining, constructing, or porting Bronto REST API request
payloads from application code.

## Dataset Selection

Use exact dataset IDs when known:

```json
{
  "log_ids": ["dataset-id-1", "dataset-id-2"]
}
```

Use `from_expr` when selecting by collection, dataset group, or tags:

```json
{
  "from_expr": "(\"collection\" IN ('prod-api'))"
}
```

Avoid sending both `log_ids` and `from_expr` unless the tool explicitly supports combining them.
Treat them as alternative source-selection modes by default.

## Discovery

`get_datasets` uses `from`, not `log_ids`, when narrowing discovery to exact dataset IDs:

```json
{
  "from": ["dataset-id-1", "dataset-id-2"],
  "minimal_output": true
}
```

Alternatively, pass `from_expr`. Do not pass both. Returned datasets include `active` and `last_heartbeat_at`; use them to avoid searching dormant datasets outside the requested window.

Use `get_datasets_by_name` when both names are known:

```json
{
  "collection_name": "production",
  "dataset_name": "api"
}
```

`get_keys` takes one `log_id` and returns common `{name, type}` entries. `get_key_values` takes `log_id` plus one exact `key`. `get_all_datasets_keys` takes no arguments and returns common keys keyed by `log_id`.

## Log Search

Raw event searches use this Bronto REST API request shape:

- dataset selection: `from` or `from_expr`
- filter: `where`
- time: `time_range` or `from_ts` + `to_ts`
- raw event fields, equivalent to `select: ["*", "@raw"]`
- sort: `most_recent_first`, optional `order_by`

MCP `search_logs` tool payload:

```json
{
  "log_ids": ["dataset-id"],
  "time_range": "Last 20 minutes",
  "search_filter": "\"status\" >= 500 AND \"service\"='api'",
  "limit": 20,
  "order_by": "\"@timestamp\" DESC"
}
```

Equivalent Bronto REST API search request payload:

```json
{
  "from": ["dataset-id"],
  "time_range": "Last 20 minutes",
  "where": "\"status\" >= 500 AND \"service\"='api'",
  "select": ["*", "@raw"],
  "limit": 20,
  "most_recent_first": true,
  "timeline_enabled": true
}
```

## Timeseries

Aggregate searches use this Bronto REST API request shape:

- `select`: aggregate expressions such as `count(*)`, `avg(field)`, `p95(field)`
- `groups`: group-by keys
- `where`: filter string
- `num_of_slices`: chart buckets
- `limit`: top group count

MCP `timeseries` tool payload:

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

Equivalent Bronto REST API aggregate request payload:

```json
{
  "from": ["dataset-id"],
  "time_range": "Last 20 minutes",
  "select": ["count(*)"],
  "where": "\"status\" >= 500",
  "groups": ["host", "path", "status"],
  "limit": 20,
  "num_of_slices": 20,
  "timeline_enabled": true
}
```

## Error Summary

Use `get_error_summary` before querying datasets for broad error or warning investigations:

```json
{
  "time_range": "Last 1 hour",
  "num_of_slices": 12
}
```

It returns precomputed org-wide totals and an affected-dataset breakdown without consuming search quota. Use the returned dataset IDs to scope subsequent `get_keys`, `timeseries`, and `search_logs` calls.

## Saved Searches

Use `get_saved_searches` without an ID to list or filter saved searches, or pass `saved_search_id` to fetch one. Execute one with:

```json
{
  "saved_search_id": "saved-search-id",
  "time_range": "Last 1 hour",
  "limit": 100
}
```

Create one with `dry_run` before persistence:

```json
{
  "name": "API errors by service",
  "dry_run": true,
  "search_details": {
    "from_expr": "(\"collection\" IN ('prod-api'))",
    "where": "\"status\" >= 500",
    "select": "count(*)",
    "groups": "service",
    "display": "toplist",
    "time_range": "Last 1 hour"
  }
}
```

`create_saved_search` accepts a `saved_searches` batch of up to 20. Supported displays are `list`, `timeseries`, `toplist`, `piechart`, `treemap`, `table`, `geomap`, and `queryvalue`.

## Monitors

Use `search_monitors` without `monitor_id` to search or filter; pass `monitor_id` to inspect one. Dry-run monitor creation:

```json
{
  "name": "API 5xx spike",
  "monitor_type": "PATTERN",
  "threshold": 100,
  "comparison_operator": "ABOVE",
  "window": "Last 15 minutes",
  "queries": [
    {
      "name": "errors",
      "from": ["dataset-id"],
      "select": "COUNT(*)",
      "where": "\"status\" >= 500"
    }
  ],
  "dry_run": true
}
```

`create_monitor` accepts a `monitors` batch of up to 20 and supports `PATTERN`, `USAGE`, and `CHANGE_DETECTION`. Every created monitor is muted. Use `validate_monitors` with optional `monitor_ids`; omit them to health-check all existing monitors.

## Coverage And Usage

`get_coverage` reports whether each dataset is monitored, searched, and ingesting. Start with its default compact output and request detail only when monitor identities, tags, metrics, or keys are needed.

`get_usage` summarizes ingestion and search activity. Narrow by activity axis and time range when the question is specific:

```json
{
  "usage_type": "search",
  "time_range": "Last 7 days",
  "group_by": "collection"
}
```

## Time Ranges

Use relative time for exploratory queries:

```json
{
  "time_range": "Last 20 minutes"
}
```

Use absolute time for incident windows:

```json
{
  "timerange_start": 1714900000000,
  "timerange_end": 1714901800000
}
```

Do not mix relative and absolute time in the same query unless the tool explicitly supports precedence.

## Filter Syntax

Quote keys with special characters:

```text
"http.status_code" >= 500
```

String values use single quotes:

```text
"service"='api'
```

Combine filters with explicit parentheses:

```text
("service"='api') AND ("status" >= 500)
```

Use `IS NULL` / `IS NOT NULL` for missing fields:

```text
"error.type" IS NOT NULL
```
