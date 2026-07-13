# Bronto Agent Skills

Agent Skills to help AI coding assistants investigate logs, latency, errors, grouped trends, and raw events with Bronto.

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

## Usage

After installing the Cursor plugin, the Bronto investigation skill is available
automatically when Cursor detects a Bronto-related task. Example prompts:

### Investigate production behavior

`Use Bronto to find the services with the highest error rate in the last hour`

`Find recent logs for checkout timeouts and show representative examples`

### Explore telemetry data

`List available Bronto datasets for this organization`

`Show the top routes by p95 latency over the last 24 hours`

`Find the fields available for this log dataset before writing a query`

## Skill Structure

The Cursor Bronto investigation skill follows the Agent Skills format:

- `skills/bronto/SKILL.md` - Skill manifest, routing table, workflow, and guardrails
- `skills/bronto/references/` - Bronto query shapes, MCP tool patterns, and investigation guidance
- `mcp.json` - Bronto hosted MCP server definitions for US and EU regions

## Cursor Plugin Prerequisites

- Bronto account in the correct data region, US or EU.
- Either OAuth access or an API key with permission to search the organization's
  data.
- For API key authentication, expose the key as `US_BRONTO_API_KEY` or
  `EU_BRONTO_API_KEY` for the organization's region.

## Quick Verification

After configuring the Bronto MCP connection, ask your agent to list available datasets. A successful response confirms that the agent can reach Bronto and authenticate with the configured MCP server.

## Links

- [Bronto docs](https://docs.bronto.io)
- [Hosted MCP guide](https://docs.bronto.io/ai-features/hosted-mcp.md)
- [API key docs](https://docs.bronto.io/Account-Management/API-Keys.md)

