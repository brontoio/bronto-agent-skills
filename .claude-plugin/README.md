# Bronto Plugin for Claude Code

This repository includes a Claude Code plugin marketplace for Bronto agent skills.

## Installation

Please refer to Claude Code official documentation for details on how to configure marketplaces and install the plugins that they contain:

https://code.claude.com/docs/en/discover-plugins#add-from-github

In a nutshell, the marketplace from this repository can be added with:

```shell
/plugin

> Add Marketplace

brontoio/bronto-agent-skills
```

From here, plugins can be enabled with e.g.

```shell
/plugin

> Manage and uninstall plugins
> bronto-claude-code-plugins
> bronto-logging
# check that the plugin is enabled
```

## Marketplace

This folder contains the Claude Code marketplace metadata:

- `marketplace.json` - Claude Code marketplace definition

