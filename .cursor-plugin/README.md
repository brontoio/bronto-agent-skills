# Bronto Plugin for Cursor

Investigate Bronto logs, datasets, and timeseries directly from Cursor using Bronto's hosted MCP servers.

The plugin provides Bronto MCP configuration and the Bronto investigation
skill. It exposes both regional MCP servers so users can authenticate to the
region that hosts their Bronto organization.

## Prerequisites

- A Bronto account in the correct data region, US or EU.
- Optional: a Bronto API key with permission to search your organization's data.
- Cursor with third-party plugins, skills, and configs enabled.

Create or manage API keys in Bronto:

[https://docs.bronto.io/Account-Management/API-Keys.md](https://docs.bronto.io/Account-Management/API-Keys.md)

## Configure Authentication

For API key authentication, set the environment variable for the region used by
your Bronto organization. Set one or both variables as needed:

```bash
export US_BRONTO_API_KEY="your_us_api_key"
export EU_BRONTO_API_KEY="your_eu_api_key"
```

The `bronto-us` server reads only `US_BRONTO_API_KEY`; the `bronto-eu` server
reads only `EU_BRONTO_API_KEY`. Restart Cursor after changing environment
variables. For a server whose regional API key is not set, complete its
OAuth-based authentication flow instead.

## Installation

### Manual Local Installation

Clone this repository into Cursor's local plugin directory:

```bash
mkdir -p ~/.cursor/plugins/local
git clone https://github.com/brontoio/bronto-agent-skills.git ~/.cursor/plugins/local/bronto-cursor-plugin
```

Then reload Cursor with `Cmd+Shift+P` and `Developer: Reload Window`.

Cursor's local plugin loader expects a real directory under `~/.cursor/plugins/local`. A symlink at `~/.cursor/plugins/local/<name>` can be skipped by the loader, so clone or copy the repository there directly.

### Keep a Dev Checkout Elsewhere

If you want the active plugin to live under Cursor's plugin directory but still appear in your normal development folder, keep the real directory under `~/.cursor/plugins/local` and create the symlink in the other direction:

```bash
mkdir -p ~/.cursor/plugins/local
mv ~/dev/projects/bronto-agent-skills ~/.cursor/plugins/local/bronto-cursor-plugin
ln -s ~/.cursor/plugins/local/bronto-cursor-plugin ~/dev/projects/bronto-agent-skills
```

## MCP Servers

This plugin declares two hosted Bronto MCP servers:


| Name        | Endpoint                       |
| ----------- | ------------------------------ |
| `bronto-us` | `https://mcp.us.bronto.io/mcp` |
| `bronto-eu` | `https://mcp.eu.bronto.io/mcp` |


Use the server that matches your Bronto organization region. Disable or ignore the other server in Cursor's MCP settings.

## Verify The Plugin

1. Open Cursor settings and confirm third-party plugins, skills, and configs are enabled.
2. If using an API key, confirm `US_BRONTO_API_KEY` or `EU_BRONTO_API_KEY` is
   available to the Cursor process. Otherwise, complete OAuth for the server
   matching your region.
3. Open Cursor's MCP status and check that `bronto-us` or `bronto-eu` connects without authentication or region errors.
4. Ask Cursor to list Bronto datasets or inspect available keys for a dataset.

For local plugin loading issues, inspect Cursor's plugin log under:

```text
~/Library/Application Support/Cursor/logs/<latest>/window*/exthost/anysphere.cursor-agent-exec/Cursor Plugins.log
```

## What's Included


| File                         | What it does                                                          |
| ---------------------------- | --------------------------------------------------------------------- |
| `.cursor-plugin/plugin.json` | Cursor plugin manifest with name, description, homepage, and keywords |
| `mcp.json`                   | Bronto hosted MCP server configuration for US and EU regions          |
| `skills/bronto/SKILL.md`     | Bronto investigation workflow, routing table, and guardrails          |
| `skills/bronto/references/`  | Detailed query, schema discovery, and investigation patterns          |


## Links

- [Bronto docs](https://docs.bronto.io)
- [Hosted MCP guide](https://docs.bronto.io/ai-features/hosted-mcp.md)
- [API key docs](https://docs.bronto.io/Account-Management/API-Keys.md)
- [Cursor plugin docs](https://cursor.com/docs/plugins)

