---
name: bronto-post-deploy-check
description: Check a service after a deploy using Bronto anomaly detection, guided by the code that was deployed. Use when someone asks whether a release looks OK, wants a post-deploy check, or asks if anything changed after a deploy.
---

# Post-deploy check with Bronto

Checks whether error rates, traffic and latency for a service have moved away from what's normal for that time of day and week since a deploy. It uses the Bronto MCP server's `detect_anomalies` tool with seasonality on, so the normal daily and weekly swings don't get flagged.

It also reads the code that went out, so it can check the things the change actually touched (new endpoints, new or changed log messages, changed error handling), not just the generic service-level metrics.

This is a sanity check, not a root cause analysis. Report what changed and offer to dig further. Don't speculate about causes unless asked.

## What you need

- **Service**: which service or datasets to check. If the user names a service, find its datasets (see step 2) and use only active ones.
- **Deploy time**: when the release went out. If the user doesn't give it, get it from the CI system that deployed it. With GitHub Actions, find the deploy workflow run for the deployed commit (`gh run list --workflow <deploy workflow> --json databaseId,headSha,createdAt`) and use the `completedAt` of the job that deploys to the target environment (`gh run view <id> --json jobs`). Other CI systems have an equivalent run or job history. Note which environments the pipeline actually deploys to. If you can't find it, ask. Don't guess it from the logs.
- **What was deployed**: a commit SHA, tag, PR or release. If the user doesn't say, see "Reading the deployed code" for how to work it out, and ask if it's still unclear.
- **Key metrics** (optional): anything the user specifically cares about.

If the deploy was less than about 15 minutes ago, there won't be much data after it. Say so, run the check anyway, and suggest running it again later.

## Reading the deployed code

The aim is the diff between what was running before and what's running now. The deployed commit is the source of truth, not whatever happens to be checked out locally.

**Where to read it from**

- **Local repo first** if the working directory is a clone of the service. It's faster and doesn't need extra access. Run `git fetch --tags` first, then check the deployed commit exists locally (`git cat-file -e <sha>`). Never read the working tree as "the deployed code": it may have uncommitted changes or be on another branch.
- **The code host** if there's no local clone, the deployed commit isn't in it, or the user gave a PR or release link. On GitHub, use the `gh` CLI or a GitHub connector, e.g. `gh api repos/{owner}/{repo}/compare/{base}...{head}` or `gh pr view <n> --json files,title,body`. The PR description is often the quickest summary of intent.

Only read. Don't check out branches, commit, push or comment on PRs.

**Working out the range**

1. Deployed commit: from the user, the release tag, the CI run, or a version/commit field in the service's own logs if one exists (check with `get_keys`).
2. Previous commit: the previous release tag, or the last deploy the user names. If you can't establish it confidently, ask rather than diffing against the wrong base.
3. `git log --oneline <base>..<head>` and `git diff --stat <base>..<head>` for the overview, then read the relevant files in full.

**What to look for**

- Endpoints, routes, handlers or consumers that were added or changed
- New or changed log messages, especially error and warning ones, and the fields they log
- Changed error handling, retries, timeouts, limits or batch sizes
- Config, feature flag or dependency changes
- Database queries or external calls that were added or changed

Keep it brief. If the diff is large, focus on files under the service's runtime code and skip tests, docs and generated files.

## Steps

1. **Read the deployed code** as above and note what to check: the routes or components touched, and any new log messages or error strings.

2. **Find the datasets.** Don't call `get_datasets` unfiltered: on a large org the response is too big to read. Use `get_datasets_by_name`, or `get_datasets` with a `from_expr` on the service's collection (e.g. `"collection" IN ('<your-collection>')`) and `minimal_output: true`. Pick the active datasets: usually the application log for errors and traffic, and a traces dataset for latency if the service sends traces. If more than a handful match, confirm which ones matter.

3. **Quick error overview.** Call `get_error_summary` from about an hour before the deploy to now. It doesn't use search quota, so do it first. Note any datasets whose errors jump after the deploy.

4. **Check the field names.** Call `get_keys` on each dataset before using a field in a filter or metric. Map what you found in the code to real fields: for example, a route in the code to the field that holds it in the logs. Don't guess field names. To see which values a field takes (e.g. the level values for the error predicate), run `timeseries` with `COUNT(*)` grouped by that field.

   Double-quote field names containing `$`, `.` or `-` everywhere they appear, including inside `metric_function`: `P95("http.duration_ms")`.

5. **Confirm the new version is live.** If the logs carry a version field (e.g. `service.version`, `version`, `release` or `git_sha`), run `timeseries` with `COUNT(*)` grouped by it (and by host) from before the deploy to now. Check the deployed version appears and the old one stops. If the old version is still logging, the rollout is incomplete; say so, because the checks below mix both versions.

6. **Run anomaly detection.** For each check below, call `detect_anomalies` with:
   - `timerange_start`: deploy time minus 2 hours (so there's a before and after)
   - `timerange_end`: now
   - `seasonality`: `compare`
   - `sensitivity`: 3 (the default) unless the user asks for something else
   - `window`: `5 minutes` if the deploy was less than about an hour ago. The automatic choice tends to be 20 minutes or more, which leaves only one or two windows after the deploy. Omit it for older deploys.

   Default checks:

   | Check | metric_function | search_filter | direction |
   |---|---|---|---|
   | Errors | `COUNT(*)` | error predicate from step 4 | `above` |
   | Traffic | `COUNT(*)` | none | `both` |
   | Latency | `P95(<duration field>)` | none | `above` |

   Then add up to three checks from the code changes, for example:
   - errors or latency filtered to a changed route, or with the route key in `group_by_keys`
   - the count of a new or changed error log message (use a bare-word full-text filter on a distinctive part of the message)
   - traffic to a downstream call or queue the change touched

   Skip latency if there's no duration field. Each call counts towards the search quota, so keep to the defaults plus at most three code-driven checks, unless the user asks for more.

   A brand-new log message has no history before the deploy, so anomaly detection can't score it. For those, use `timeseries` to report how often it's happening instead.

   Seasonal profiles are built per query: the dataset, filter, aggregate and group-by keys. The first seasonal call for a query starts a backfill of its history and returns `PROFILE_GENERATING`; later calls with the same query use the trained profile. Keep the default checks' queries identical from one deploy to the next so they keep benefiting from it. The window size isn't part of the query, so changing it doesn't start a new profile.

7. **Decide what counts.**
   - Episodes that start at or after the deploy time are relevant.
   - An episode whose window contains the deploy time is borderline. Check whether the same metric spikes at the same point in earlier hours or days before attributing it to the deploy.
   - An episode that recurs on a regular schedule (e.g. every hour at :17) is a periodic job, not a regression. Mention it but don't treat it as a problem.
   - If an episode has `explained_by_seasonality: true`, mention it but don't treat it as a problem.
   - If the response says `PROFILE_GENERATING`, seasonality wasn't applied for that metric. Say so, because the result may include normal daily swings, and suggest re-running later once the profile is ready.
   - An episode that was already running before the deploy isn't caused by it. Mention it separately.

## Report

Start with a one-line verdict:

- **Looks clean**: no relevant episodes.
- **Worth a look**: `WARN` episodes, or something small on one metric.
- **Possible regression**: `ALERT` episodes on errors or latency that started after the deploy.

Then:

- **What was deployed**: the commit range, a one or two line summary of the change, the environment, and whether the version check in step 5 confirmed the rollout.
- **Checks**: a short table with one row per check, marking which came from the code changes. For any episode, give the start time, direction, peak versus expected value, and severity. Use 24-hour times in the user's timezone.
- **Possibly related changes**: for anything flagged, the files or commits that touch the same route or log message. Describe these as possibly related, not as the cause.

Link to Bronto for anything flagged, using the `webapp_url` from the tool results or `get_webapp_link`.

Finish by offering one next step if something was flagged, such as looking at the error messages or the relevant part of the diff. If nothing was flagged, don't add anything else.

If a check failed, a dataset couldn't be queried, or you couldn't read the code, say which. Never report "looks clean" for a check that didn't run.
