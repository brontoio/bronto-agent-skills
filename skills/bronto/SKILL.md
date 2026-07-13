---
name: bronto
description: Investigate logs, latency, errors, grouped trends, and raw events using Bronto MCP. Use when the user mentions Bronto, log search, datasets, timeseries, production debugging, or telemetry investigation.
---

# Bronto

## Route The Request

| User needs            | Read first                             | Prefer                                                         |
| --------------------- | -------------------------------------- | -------------------------------------------------------------- |
| Find datasets         | `references/mcp-tool-patterns.md`      | `get_datasets`, then `get_keys`                                |
| Raw log investigation | `references/investigation-patterns.md` | `timeseries` for broad scope; `search_logs` for exact evidence |
| Unknown field names   | `references/mcp-tool-patterns.md`      | `get_keys`, then `get_key_values`                              |
| Build query payloads  | `references/bronto-query-shapes.md`    | MCP argument names for MCP tools; API fields for REST requests |


## Workflow

1. Inventory Bronto MCP tools. Read tool schemas before calling tools. If an auth tool exists, authenticate first.
2. Discover the dataset unless the user gave exact dataset IDs:
  - `get_datasets` for broad discovery.
  - `get_datasets_by_name` when collection and dataset names are known.
3. Inspect fields before guessing:
  - `get_keys` for one dataset.
  - `get_key_values` for candidate dimensions like service, status, host, route, and error type.
4. Use `timeseries` before `search_logs` for operational questions:
  - counts, rates, percentiles, group-bys, regressions, top offenders.
5. Use `search_logs` for evidence:
  - raw examples, exact event IDs, pasted error messages, event payloads, `@raw`, pagination/drill-down.
6. Keep first queries narrow:
  - small time range, precise `search_filter`, small `limit`, explicit `group_by_keys`.
7. Report findings evidence-first:
  - Start with the direct answer or current best hypothesis.
  - Include the exact dataset selection, time window, filters, aggregations, and group-bys used.
  - Summarize grouped results with counts/rates/percentiles and call out the top offenders.
  - Include representative raw examples only after aggregation, with timestamps, IDs, key fields, and message snippets.
  - Separate facts from inference. State uncertainty, missing data, and the next query that would reduce it.

## Guardrails

- Do not query every dataset by default. Discover, filter by tags/collection/name, then query.
- Prefer `from_expr`/dataset selection expressions when the user asks by collection/tag. Prefer `log_ids` when exact datasets are known.
- Prefer `timeseries` for error rates, trends, top offenders, and latency percentiles.
- Prefer `search_logs` after aggregation identifies where to drill down, except when the user already provides exact evidence such as an event ID, unique error string, or pasted log message.
- Preserve Bronto filter syntax: quote keys with special characters, e.g. `"http.status_code" >= 500`.

