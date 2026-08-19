# Bronto Agent Skills

Agent Skills to help AI coding assistants investigate Bronto telemetry, manage saved searches and monitors, and audit coverage and usage.

## Installation

### As a Platform Plugin

The Cursor plugin includes the Bronto investigation skill and hosted MCP
configuration. The Claude Code marketplace currently exposes only the separate
`bronto-logging` plugin for logging statement IDs; it does not package the
Bronto investigation skill.

| Platform    | Available components                              | Setup guide                                             |
| ----------- | ------------------------------------------------- | ------------------------------------------------------- |
| Cursor      | Bronto investigation skill and hosted MCP config | [Cursor setup](.cursor-plugin/README.md)                 |
| Claude Code | `bronto-logging` statement-ID plugin only        | [Claude Code setup](.claude-plugin/README.md)            |

The Cursor plugin is not currently listed in the public Marketplace. The
[Cursor setup guide](.cursor-plugin/README.md) covers direct Git installation
and local development until Marketplace installation is available.

## Usage

After installing the Cursor plugin, the Bronto investigation skill is available
automatically when Cursor detects a Bronto-related task. Example prompts:

### Investigate production behavior

`Use Bronto to find the services with the highest error rate in the last hour`

`Find recent logs for checkout timeouts and show representative examples`

`Use Bronto's error summary to triage organization-wide errors before searching logs`

### Explore telemetry data

`List available Bronto datasets for this organization`

`Show the top routes by p95 latency over the last 24 hours`

`Find the fields available for this log dataset before writing a query`

### Reuse and monitor queries

`Find and run the saved search for checkout errors`

`Dry-run a muted monitor for API 5xx spikes, then validate its query`

`Show datasets that are ingesting but are neither monitored nor searched`

## Skill Structure

The Cursor Bronto investigation skill follows the Agent Skills format:

- `skills/bronto/SKILL.md` - Skill manifest, routing table, workflow, and guardrails
- `skills/bronto/references/` - Bronto query shapes, MCP tool patterns, and investigation guidance
- `mcp.json` - Bronto hosted MCP server definitions for US and EU regions

## Cursor Plugin Prerequisites

- Bronto account in the correct data region, US or EU.
- MCP access enabled by a Bronto administrator under **Settings →
  Authentication**.
- Either OAuth access or an API key with permission to search the organization's
  data.
- For API key authentication, configure `US_BRONTO_API_KEY` or
  `EU_BRONTO_API_KEY` on the installed plugin for the organization's region.

## Quick Verification

After configuring the Bronto MCP connection:

1. Open **Customize → MCPs**, enable the selected region, and disable the other
   regional server.
2. Select **Login** to use OAuth, or configure the selected region's API-key
   plugin variable.
3. Call `get_datasets` and confirm results include `active` and `last_heartbeat_at`.
4. Call `get_error_summary` for a short window to verify precomputed error triage.

## Links

- [Bronto docs](https://docs.bronto.io)
- [Hosted MCP guide](https://docs.bronto.io/ai-features/hosted-mcp.md)
- [API key docs](https://docs.bronto.io/Account-Management/API-Keys.md)

