# Bronto Investigation Patterns

Use this reference when a coding agent is investigating production issues, telemetry,
logs, traces, errors, latency, or traffic changes through Bronto MCP.

## Default Investigation Flow

1. Discover the relevant datasets with `get_datasets` unless the user already gave exact dataset IDs.
2. Inspect schema with `get_keys` before guessing field names.
3. Inspect common values with `get_key_values` for fields you plan to filter or group by.
4. Use `timeseries` first for broad scope, counts, rates, trends, percentiles, and top offenders.
5. Use `search_logs` after narrowing the question to fetch representative raw evidence.
  Use `search_logs` directly when the user provides an exact trace ID, event ID, unique error string, or pasted log message.
6. Report the dataset selection, time window, filter, grouped findings, and raw examples used.

Do not start by querying every dataset. Pick candidate datasets by name, collection, tags, or exact ID.

## Raw Logs

For "what happened" questions:

1. Identify production dataset(s).
2. Use `timeseries` grouped by high-signal dimensions.
3. Drill into the worst group with `search_logs`.
4. Return examples with timestamps, key fields, and raw message snippets.

For exact evidence requests, such as a trace ID, event ID, unique error string, or
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
`status`, `status_code`, `http.status_code`, `$span.status_code`.

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

If the field is OpenTelemetry span duration, use `$span.duration_nano` and convert ns to ms
by dividing by `1_000_000`.

```json
{
  "from_expr": "(\"collection\" IN ('.traces'))",
  "metric_functions": [
    "P50($span.duration_nano)",
    "P95($span.duration_nano)",
    "P99($span.duration_nano)"
  ],
  "group_by_keys": ["$span.name", "$service.name"],
  "time_range": "Last 1 hour",
  "limit": 20
}
```

## Traces

Trace datasets are usually in collection `.traces`.

Common OTel fields:

- `$span.trace_id`
- `$span.span_id`
- `$span.parent_span_id`
- `$span.name`
- `$service.name`
- `$span.status_code`
- `$span.duration_nano`
- `$span.kind`

Root span filter:

```text
"$span.parent_span_id"='0000000000000000'
```

Trace list / operation overview:

```json
{
  "from_expr": "(\"collection\" IN ('.traces'))",
  "metric_functions": ["COUNT(*)"],
  "search_filter": "\"$span.parent_span_id\"='0000000000000000'",
  "group_by_keys": ["$span.name", "$service.name", "$span.status_code"],
  "time_range": "Last 10 minutes",
  "limit": 1000
}
```

Trace latency percentiles:

```json
{
  "from_expr": "(\"collection\" IN ('.traces'))",
  "metric_functions": [
    "P50($span.duration_nano)",
    "P95($span.duration_nano)",
    "P99($span.duration_nano)"
  ],
  "search_filter": "\"$span.parent_span_id\"='0000000000000000'",
  "time_range": "Last 10 minutes"
}
```

Error traces:

```json
{
  "from_expr": "(\"collection\" IN ('.traces'))",
  "metric_functions": ["COUNT(*)"],
  "search_filter": "\"$span.parent_span_id\"='0000000000000000' AND \"$span.status_code\"='STATUS_CODE_ERROR'",
  "group_by_keys": ["$span.name", "$service.name"],
  "time_range": "Last 10 minutes",
  "limit": 50
}
```

Trace drill-down:

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

If trace ID fields vary, OR them:

```text
"$span.trace_id"='TRACE_ID' OR "$trace_id"='TRACE_ID' OR "trace_id"='TRACE_ID'
```

For span-specific drill-down, also OR span ID variants:

```text
("$span.trace_id"='TRACE_ID' OR "$trace_id"='TRACE_ID' OR "trace_id"='TRACE_ID')
AND ("$span.span_id"='SPAN_ID' OR "$span_id"='SPAN_ID' OR "span_id"='SPAN_ID')
```

