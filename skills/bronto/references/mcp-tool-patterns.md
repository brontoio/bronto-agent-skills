# Bronto MCP Tool Patterns

## Tool Metadata

Tool names, descriptions, and argument schemas are provided by the Bronto MCP server. Treat MCP tool descriptors as the source of truth; this file only documents agent-side investigation flow and known-good query shapes.

## Common Sequence
1. `get_datasets`
2. Pick candidate dataset IDs or build `from_expr`
3. `get_keys` on the most relevant dataset
4. `get_key_values` for dimensions before filtering
5. `timeseries` with grouped aggregation
6. `search_logs` for representative raw events

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
