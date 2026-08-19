# Bronto Plugin for Cursor

Investigate Bronto telemetry, reuse queries, manage monitors, and audit coverage directly from Cursor using Bronto's hosted MCP servers.

The plugin provides Bronto MCP configuration and the Bronto investigation
skill. It exposes both regional MCP servers so users can authenticate to the
region that hosts their Bronto organization.

## Prerequisites

- A Bronto account in the correct data region, US or EU.
- MCP access enabled by a Bronto administrator under **Settings → Authentication**.
- Optional: a Bronto API key with permission to search your organization's data.
- Cursor Desktop or Cursor Agent CLI.

Create or manage API keys in Bronto:

[https://docs.bronto.io/Account-Management/API-Keys.md](https://docs.bronto.io/Account-Management/API-Keys.md)

## Installation

### Direct Git URL

Cursor Agent CLI can install the plugin directly from GitHub:

1. Start the CLI with `agent`.
2. Run `/plugin`.
3. Paste `https://github.com/brontoio/bronto-agent-skills.git` into plugin
  search, then install it at user or project scope.



### Local Development

Clone the repository, then symlink it into Cursor's local plugin directory:

```bash
git clone https://github.com/brontoio/bronto-agent-skills.git /absolute/path/to/bronto-agent-skills
mkdir -p ~/.cursor/plugins/local
ln -s /absolute/path/to/bronto-agent-skills ~/.cursor/plugins/local/bronto-cursor-plugin
```

Alternatively, load the checkout for one Cursor Agent CLI session:

```bash
agent --plugin-dir /absolute/path/to/bronto-agent-skills
```

After adding the local plugin, restart Cursor or open the Command Palette
(`Cmd+Shift+P` on macOS; `Ctrl+Shift+P` on Windows/Linux) and run
**Developer: Reload Window**.

## Configure Authentication



### OAuth

OAuth is the default:

1. Open **Customize → MCPs**.
2. Select `bronto-us` or `bronto-eu` for your organization's region.
3. Select **Login** and complete the browser authentication flow.
4. Disable the unused regional server.



### API Key

Open the installed Bronto plugin in **Customize**, select **Configure**, and set
`US_BRONTO_API_KEY` or `EU_BRONTO_API_KEY`. The `bronto-us` server reads only
`US_BRONTO_API_KEY`; the `bronto-eu` server reads only `EU_BRONTO_API_KEY`.
Leave both unset and complete the OAuth login flow to use OAuth.

## MCP Servers

This plugin declares two hosted Bronto MCP servers:


| Name        | Endpoint                       |
| ----------- | ------------------------------ |
| `bronto-us` | `https://mcp.us.bronto.io/mcp` |
| `bronto-eu` | `https://mcp.eu.bronto.io/mcp` |


Use the server that matches your Bronto organization region. Disable the other
server under **Customize → MCPs**.

## Verify The Plugin

1. Open **Customize → MCPs** and confirm the server matching your region is
  enabled and the other regional server is disabled.
2. For OAuth, select **Login** and complete browser authentication. For API-key
  authentication, configure the matching plugin variable.
3. Confirm the selected server connects without authentication or region errors.
4. Ask Cursor to call `get_datasets` and confirm results include `active` and
  `last_heartbeat_at`.
5. Ask Cursor to call `get_error_summary` for a short window.

For connection or authentication errors, open Cursor's Output panel
(`Cmd+Shift+U` on macOS; `Ctrl+Shift+U` on Windows/Linux) and select
**MCP Logs**. Use **Customize → MCPs** or `agent mcp list` to check server
status.

## What's Included


| File                         | What it does                                                           |
| ---------------------------- | ---------------------------------------------------------------------- |
| `.cursor-plugin/plugin.json` | Cursor plugin manifest with name, description, homepage, and keywords  |
| `mcp.json`                   | Bronto hosted MCP server configuration for US and EU regions           |
| `skills/bronto/SKILL.md`     | Bronto investigation workflow, routing table, and guardrails           |
| `skills/bronto/references/`  | MCP tool guidance, query payload examples, and investigation workflows |


The MCP surface covers dataset and schema discovery, raw and aggregate queries,
precomputed error triage, saved searches, monitors, coverage, usage, and
regional browser authentication.

## Links

- [Bronto docs](https://docs.bronto.io)
- [Hosted MCP guide](https://docs.bronto.io/ai-features/hosted-mcp.md)
- [API key docs](https://docs.bronto.io/Account-Management/API-Keys.md)
- [Cursor plugin docs](https://cursor.com/docs/plugins)

