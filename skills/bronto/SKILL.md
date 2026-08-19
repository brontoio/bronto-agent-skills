---
name: bronto
description: Investigate Bronto logs, errors, latency, trends, and raw events; manage saved searches and monitors; and audit telemetry coverage and usage. Use when the user mentions Bronto, datasets, log search, timeseries, monitors, saved searches, production debugging, or telemetry investigation.
---

# Bronto

## Route The Request

| User needs                  | Read first                             | Prefer                                                         |
| --------------------------- | -------------------------------------- | -------------------------------------------------------------- |
| Find datasets or fields     | `references/mcp-tool-patterns.md`      | `get_datasets`, then `get_keys` / `get_key_values`             |
| Cross-dataset schema        | `references/mcp-tool-patterns.md`      | `get_all_datasets_keys`                                       |
| Error or warning triage     | `references/investigation-patterns.md` | `get_error_summary` before dataset queries                     |
| Raw log investigation       | `references/investigation-patterns.md` | `timeseries` for broad scope; `search_logs` for exact evidence |
| Saved searches              | `references/mcp-tool-patterns.md`      | Find or run existing searches before creating one              |
| Monitors                    | `references/mcp-tool-patterns.md`      | Search, dry-run creation, then validate                         |
| Coverage, blind spots, usage | `references/investigation-patterns.md` | `get_coverage`, `get_usage`                                    |
| Build query payloads        | `references/bronto-query-shapes.md`    | MCP argument names for MCP tools; API fields for REST requests |


## Workflow

1. Inventory Bronto MCP tools and read their current schemas. If authentication is required, call the region's Cursor-injected `mcp_auth` tool for browser OAuth.
2. For error or warning investigations, call `get_error_summary` first. It provides org-wide totals and affected datasets without consuming search quota.
3. Discover the dataset unless the user gave exact dataset IDs or `get_error_summary` already identified them:
  - `get_datasets` for broad discovery. Prefer datasets whose `active` and `last_heartbeat_at` overlap the requested time window; narrow with `from_expr` or `from` IDs when useful.
  - `get_datasets_by_name` when collection and dataset names are known.
4. Inspect fields before guessing:
  - `get_keys` for one dataset.
  - `get_key_values` for candidate dimensions like service, status, host, route, and error type.
  - `get_all_datasets_keys` for a schema overview across accessible datasets; still resolve fields per dataset before querying because schemas differ.
5. Use `timeseries` before `search_logs` for operational questions:
  - counts, rates, percentiles, group-bys, regressions, top offenders.
6. Use `search_logs` for evidence:
  - raw examples, exact event IDs, pasted error messages, event payloads, `@raw`, pagination/drill-down.
7. Keep first queries narrow:
  - small time range, precise `search_filter`, small `limit`, explicit `group_by_keys`.
8. For reusable queries, use `get_saved_searches` to find existing searches, `run_saved_search` to execute one by ID, and `create_saved_search` only when asked. Creation supports up to 20 searches, optional visualizations, and `dry_run`.
9. For alerting, use `search_monitors` before creating duplicates. `create_monitor` supports up to 20 `PATTERN`, `USAGE`, or `CHANGE_DETECTION` monitors and `dry_run`; every created monitor starts muted. Use `validate_monitors` to health-check existing monitor queries and data.
10. Use `get_coverage` to find unmonitored, unsearched, or non-ingesting datasets. Use `get_usage` for org ingestion and search activity.
11. Report findings evidence-first:
  - Start with the direct answer or current best hypothesis.
  - Include the exact dataset selection, time window, filters, aggregations, and group-bys used.
  - Summarize grouped results with counts/rates/percentiles and call out the top offenders.
  - Include representative raw examples only after aggregation, with timestamps, IDs, key fields, and message snippets.
  - Separate facts from inference. State uncertainty, missing data, and the next query that would reduce it.

## Guardrails

- Do not query every dataset by default. Discover, filter by tags/collection/name, then query.
- Prefer `from_expr`/dataset selection expressions when the user asks by collection/tag. Prefer `log_ids` when exact datasets are known.
- Prefer `get_error_summary` before ad hoc queries for open-ended error and warning questions.
- Prefer `timeseries` for rates, trends, top offenders, and latency percentiles after selecting datasets.
- Prefer `search_logs` after aggregation identifies where to drill down, except when the user already provides exact evidence such as an event ID, unique error string, or pasted log message.
- Before creating saved searches or monitors, check for existing equivalents and use `dry_run`. Do not create either unless the user asked for a persistent change.
- Treat newly created monitors as inactive until the user reviews and unmutes them in Bronto.
- Bronto's query language is a subset of SQL. Preserve its filter syntax: quote keys with special characters, e.g. `"http.status_code" >= 500`.

