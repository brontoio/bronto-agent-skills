# Bronto Investigation Patterns

Use this reference when a coding agent is investigating production issues, telemetry,
logs, errors, latency, or traffic changes through Bronto MCP.

## Default Investigation Flow

1. Discover the relevant datasets with `get_datasets` unless the user already gave exact dataset IDs.
2. Inspect schema with `get_keys` before guessing field names.
3. Inspect common values with `get_key_values` for fields you plan to filter or group by.
4. Use `timeseries` first for broad scope, counts, rates, trends, percentiles, and top offenders.
5. Use `search_logs` after narrowing the question to fetch representative raw evidence.
  Use `search_logs` directly when the user provides an exact event ID, unique error string, or pasted log message.
6. Report the dataset selection, time window, filter, grouped findings, and raw examples used.

Do not start by querying every dataset. Pick candidate datasets by name, collection, tags, or exact ID.

## Raw Logs

For "what happened" questions:

1. Identify dataset(s).
2. Use `timeseries` grouped by high-signal dimensions.
3. Drill into the worst group with `search_logs`.
4. Return examples with timestamps, key fields, and raw message snippets.

For exact evidence requests, such as an event ID, unique error string, or
pasted log message, start with a narrow `search_logs` query instead of aggregating first.

Good group-by candidates (always use keys that we know exist in the datasets from the get_keys tool call):

- `service`, `host`, `path`, `route`, `status`, `status_code`
- `error`, `error_type`, `exception`, `message`
- `client_ip`, `user_agent`, `region`, `pod`, `container`

## Error Spikes

Start with a grouped `timeseries` query:

```json
{
  "log_ids": ["dataset-id"],
  "metric_functions": ["COUNT(*)"],
  "search_filter": "\"status\" >= 500",
  "group_by_keys": ["service", "path", "status"],
  "time_range": "Last 30 minutes",
  "limit": 20
}
```

Then run `search_logs` on the top group:

```json
{
  "log_ids": ["dataset-id"],
  "time_range": "Last 30 minutes",
  "search_filter": "\"status\" >= 500 AND \"service\"='api'",
  "limit": 20,
  "order_by": "\"@timestamp\" DESC"
}
```

If the status field is unknown, inspect keys first and try likely variants:
`status`, `status_code`, `http.status_code`.

## Latency

Prefer percentiles over averages:

```json
{
  "log_ids": ["dataset-id"],
  "metric_functions": [
    "P50(duration_ms)",
    "P95(duration_ms)",
    "P99(duration_ms)"
  ],
  "group_by_keys": ["service", "path"],
  "time_range": "Last 1 hour",
  "limit": 20
}
```

