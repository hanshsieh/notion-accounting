# Notion Accounting Cursor Plugin

This repository provides a Cursor plugin for Notion-based personal accounting workflows, including reusable skills, plugin metadata, and MCP integration configuration.

## What this plugin includes

- Skills under `skills/`
- Plugin manifest at `.cursor-plugin/plugin.json`
- Notion MCP server config in `mcp.json`

## Use as a local plugin

1. Clone this repository.
2. Copy (or symlink) this repository to:
   - Linux/macOS: `~/.cursor/plugins/local/notion-accounting`
   - Windows: `%USERPROFILE%\.cursor\plugins\local\notion-accounting`
3. Restart Cursor, or run **Developer: Reload Window**.
4. Open Cursor Settings -> Plugins, and confirm `Notion Accounting` appears as installed.

## Required setup

### Notion data source setup
Follow the step-by-step guide in:

- [`docs/notion-data-sources-setup.md`](docs/notion-data-sources-setup.md)

### Notion MCP
Set up the MCP server by following the instructions in
[`makenotion/notion-mcp-server`](https://github.com/makenotion/notion-mcp-server).

Do **not** use the global hosted Notion MCP server setup described in
[Connecting to Notion MCP](https://developers.notion.com/guides/mcp/get-started-with-mcp).

This plugin depends on filtering data source pages by property values. The
global hosted Notion MCP server does not support that capability yet, which
would greatly reduce this plugin's effectiveness.
