# Bronto Agent Skills

Agent Skills to help AI coding assistants investigate logs, traces, latency, errors, grouped trends, and raw events with Bronto.

## Installation

### As a Platform Plugin

This repository also includes platform plugin metadata. Configure the Bronto MCP server connection through the setup guide for your platform.


| Platform    | Setup guide                                                                                 |
| ----------- | ------------------------------------------------------------------------------------------- |
| Cursor      | `[.cursor-plugin/README.md](.cursor-plugin/README.md)`                                      |
| Claude Code | [Claude Code plugin docs](https://code.claude.com/docs/en/discover-plugins#add-from-github) |


## Usage

Once installed, the skill is available automatically when your assistant detects a Bronto-related task. Example prompts:

### Investigate production behavior

`Use Bronto to find the services with the highest error rate in the last hour`

`Look up this trace ID and summarize the failing span`

`Find recent logs for checkout timeouts and show representative examples`

### Explore telemetry data

`List available Bronto datasets for this organization`

`Show the top routes by p95 latency over the last 24 hours`

`Find the fields available for this log dataset before writing a query`

## Skill Structure

The Bronto skill follows the Agent Skills format:

- `skills/bronto/SKILL.md` - Skill manifest, routing table, workflow, and guardrails
- `skills/bronto/references/` - Bronto query shapes, MCP tool patterns, and investigation guidance
- `mcp.json` - Bronto hosted MCP server definitions for US and EU regions

## Prerequisites

- Bronto account in the correct data region, US or EU.
- API key with permission to search the organization's data.
- MCP login method enabled in Bronto when using OAuth-capable clients.

## Quick Verification

After configuring the Bronto MCP connection, ask your agent to list available datasets. A successful response confirms that the agent can reach Bronto and authenticate with the configured MCP server.

## Links

- [Bronto docs](https://docs.bronto.io)
- [Hosted MCP guide](https://docs.bronto.io/ai-features/hosted-mcp.md)
- [API key docs](https://docs.bronto.io/Account-Management/API-Keys.md)

