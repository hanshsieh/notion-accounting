# Notion Accounting

This repository provides Cursor skills and MCP configuration for Notion-based personal accounting workflows.

## What this project includes

- Skills under `.cursor/skills/`
- Notion MCP server config in `.cursor/mcp.json`

## Use in Cursor

1. Clone this repository and open it as a Cursor project.
2. Copy `.env.example` to `.env` and set your Notion integration token.
3. Follow the Notion data source setup guide in [`docs/notion-data-sources-setup.md`](docs/notion-data-sources-setup.md).
4. Restart Cursor, or run **Developer: Reload Window**, so skills and MCP servers are picked up.

## Required setup

### Notion data source setup

Follow the step-by-step guide in:

- [`docs/notion-data-sources-setup.md`](docs/notion-data-sources-setup.md)

### Notion MCP

Do **not** use the global hosted Notion MCP server setup described in
[Connecting to Notion MCP](https://developers.notion.com/guides/mcp/get-started-with-mcp).

This project depends on filtering data source pages by property values. The
global hosted Notion MCP server does not support that capability yet, which
would greatly reduce its effectiveness.
