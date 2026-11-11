# bronto-post-deploy-check

A Claude Code skill that answers "did that deploy break anything?" using Bronto's MCP server. It reads the code that went out, works out what the change touched, and then runs Bronto's `detect_anomalies` over errors, traffic and latency, plus up to three checks aimed at the changed code, comparing the time since the deploy against what's normal for that time of day and week.

It is a sanity check, not a root-cause tool: it reports what moved, links to Bronto for anything flagged, and offers one next step.

## What it shows off

- **`detect_anomalies` with seasonality.** Each metric is scored against a rolling baseline, and with `seasonality: compare` every episode says whether the usual daily or weekly pattern explains it. A Monday-morning traffic ramp doesn't get reported as a regression.
- **Code-aware checks.** The skill reads the diff between the previous and deployed commit and turns it into targeted checks: errors on a changed route, the rate of a new log message, traffic to a downstream call the change touched.
- **Cheap triage first.** `get_error_summary` runs before any anomaly query because it doesn't use search quota.

## Prerequisites

- A Bronto account with logs from the service you want to check.
- The Bronto MCP server connected to Claude Code. It uses OAuth, so there is no API key to manage:

  ```bash
  # EU
  claude mcp add --transport http bronto https://mcp.eu.bronto.io/mcp
  # US
  claude mcp add --transport http bronto https://mcp.us.bronto.io/mcp
  ```

  Then run `/mcp` in Claude Code and sign in to Bronto.
- Optional: a local clone of the service and the [`gh`](https://cli.github.com/) CLI, so the skill can read the deployed diff and find the deploy time from GitHub Actions. Without them it asks you for the deploy time and commit range.

## Install

Copy or symlink the directory into a skills path Claude Code reads:

```bash
# available in every project
ln -s "$PWD" ~/.claude/skills/bronto-post-deploy-check

# or scoped to one repo
ln -s "$PWD" /path/to/repo/.claude/skills/bronto-post-deploy-check
```

Claude picks it up on the next session. Ask for it by name (`/bronto-post-deploy-check`) or describe the task:

> Did the checkout-api release that went out at 14:00 look OK?

> Run a post-deploy check on checkout-api for v2.14.0.

Running it from inside a clone of the service gives the best results, because the skill can read the diff directly.

## Example report

Abridged, from a run against a small change to a monitor-update API:

> **Looks clean.** No new error or latency anomalies since the deploy, and the changed endpoint is serving successfully.
>
> **What was deployed:** `v1.252` → `v1.253`, one commit: the monitor update path now always writes the `aux` field, so an empty map clears the previous value. Deployed to staging at 11:25; the version field confirms all hosts are on `v1.253`. About 30 minutes of data so far, so worth re-running later.
>
> | Check | From code? | Result |
> |---|---|---|
> | Errors | | No episodes |
> | Traffic | | ALERT at 09:17, 10:17 and 11:17 (~14.0k vs ~13.4k expected). Recurs hourly and started before the deploy: a periodic job, not this change |
> | Latency (P95) | | No episodes |
> | `UpdateMonitor` calls | yes | 3 calls since the deploy, all successful, 360–470 ms. No calls in the 2 h before, so no baseline |
>
> Seasonal profiles were still generating for these queries, so seasonality wasn't applied this time.

## How it uses the MCP tools

| Step | Tool | Why |
|---|---|---|
| Find datasets | `get_datasets_by_name` / `get_datasets` with `from_expr` | Scope to the service's active datasets |
| Error overview | `get_error_summary` | Free first pass over the whole org |
| Field names and values | `get_keys`, grouped `timeseries` | Map code concepts (routes, log messages) to real fields |
| Rollout check | `timeseries` grouped by version | Confirm the new version is the one logging |
| Anomaly checks | `detect_anomalies` | Defaults plus up to three code-driven checks |
| Links | `webapp_url` from results, `get_webapp_link` | Jump to anything flagged in Bronto |

### Seasonal profiles

A seasonal profile is built per query: the dataset, filter, aggregate and group-by keys. The first time a query is run with seasonality on, Bronto starts backfilling its history and the response reports `PROFILE_GENERATING`; that run is scored without seasonality, and the skill says so. Later runs of the same query use the trained profile. That is why the skill keeps its default queries identical between deploys: the first check of a service warms the profiles, and every check after that benefits.

## Developing on it

- **`SKILL.md` is the interface.** It carries the judgement: which checks to run, what counts as relevant, and how to report. A change in how the MCP tools behave usually needs a matching change there.
- **Keep it generic.** No org IDs, hostnames, dataset or collection names from a real account. Examples are illustrative.
- **Search quota matters.** Every `detect_anomalies` and `timeseries` call counts towards it. Keep the default check count low and make extra checks opt-in.
