# Notion Accounting

This repository provides Cursor skills for Notion-based personal accounting workflows.

## What this project includes

- Skills under `.cursor/skills/`
- Setup docs for Notion data sources in `docs/`

## Use in Cursor

1. Clone this repository and open it as a Cursor project.
2. Connect **Notion** in Cursor (**Settings → MCP** or **Integrations**).
3. Follow the Notion data source setup guide in [`docs/notion-data-sources-setup.md`](docs/notion-data-sources-setup.md).
4. Restart Cursor, or run **Developer: Reload Window**, so skills are picked up.

## Required setup

### Notion integration

This project uses Cursor's **Notion** workspace plugin MCP (`notion-fetch`, `notion-query-data-sources`, `notion-create-pages`, etc.).

- Connect Notion in Cursor and grant access to the workspace that contains your accounting databases.
- You do **not** need a local `NOTION_TOKEN` or project-level MCP server config.

### Notion data source setup

Follow the step-by-step guide in:

- [`docs/notion-data-sources-setup.md`](docs/notion-data-sources-setup.md)

Save the four data source IDs to:

- `.cursor/skills/notion-accounting-schema/assets/config.json`

Start from:

- `.cursor/skills/notion-accounting-schema/assets/config.example.json`

## Skills

| Skill | Purpose |
| --- | --- |
| `notion-accounting-schema` | Database schema, config, and Notion MCP read/write patterns |
| `notion-accounting-create-transactions` | Import receipts/statements into the Transactions database |
