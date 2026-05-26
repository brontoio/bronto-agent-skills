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

Trace collection selection:

```json
{
  "from_expr": "(\"collection\" IN ('.traces'))"
}
```

Avoid sending both `log_ids` and `from_expr` unless the tool explicitly supports combining them.
Treat them as alternative source-selection modes by default.

## Log Search

Raw event searches use this Bronto REST API request shape:

- dataset selection: `from` or `from_expr`
- filter: `where`
- time: `time_range` or `from_ts` + `to_ts`
- raw event fields, equivalent to `select: ["*", "@raw"]`
- sort: `most_recent_first`, optional `order_by`
- pagination and async polling for large searches

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
  "timeline_enabled": true,
  "async_enabled": true
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
  "timeline_enabled": true,
  "async_enabled": true
}
```

## Async API Responses

For Bronto REST API request payloads with `async_enabled: true`, the initial
response returns HTTP `202`. In the response body's `links` array, find the object
where `rel` is `status`; its `href` value is the polling URL for the async
results.

Poll that `href` until the final result is ready:

- Intermediate polling responses return HTTP `200`.
- The response body has a `status` field, usually `IN_PROGRESS` while work is
  still running.
- The final polling response returns HTTP `201` and `status: "COMPLETED"`.
- Stop polling only after receiving both HTTP `201` and `status: "COMPLETED"`.

## Traces

Tracing uses normal log search and timeseries over trace datasets.

Trace dataset expression:

```json
{
  "from_expr": "(\"collection\" IN ('.traces'))"
}
```

Root span filter:

```text
"$span.parent_span_id"='0000000000000000'
```

Span ID present filter:

```text
("$span.span_id" IS NOT NULL OR "$span_id" IS NOT NULL OR "span_id" IS NOT NULL)
```

Trace overview query:

```json
{
  "from_expr": "(\"collection\" IN ('.traces'))",
  "time_range": "Last 10 minutes",
  "metric_functions": ["COUNT(*)"],
  "search_filter": "\"$span.parent_span_id\"='0000000000000000'",
  "group_by_keys": ["$span.name", "$service.name", "$span.status_code"],
  "limit": 1000
}
```

Single trace raw query:

Reuse the incident window when one is known. Otherwise start with a narrow
relative range and widen only if the trace is not found.

```json
{
  "from_expr": "(\"collection\" IN ('.traces'))",
  "time_range": "Last 30 minutes",
  "search_filter": "\"$span.trace_id\"='TRACE_ID'",
  "limit": 1000
}
```

Known trace and span ID variants:

- Trace IDs: `$span.trace_id`, `$trace_id`, `trace_id`
- Span IDs: `$span.span_id`, `$span_id`, `span_id`

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
"$span.status_code"='STATUS_CODE_ERROR'
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
"$span.span_id" IS NOT NULL
```
