---
name: notion-accounting-schema
description: Defines how the Notion databases should be structured and used for personal accounting. Use when user wants to read or write transactions in the Notion databases.
---

## Purpose

This skill illustrates how to use Notion databases for personal accounting.  
Use this skill whenever the user asks to read, write, or update the Notion databases for transactions.

## Notion MCP

Use Cursor's **Notion** integration (workspace plugin MCP), not the legacy `@notionhq/notion-mcp-server`.

### Setup

1. In Cursor, open **Settings → MCP** (or **Integrations**) and connect **Notion**.
2. Grant the integration access to the workspace that contains the accounting databases.
3. If a Notion MCP call fails with authentication or permission errors, stop and ask the user to reconnect Notion in Cursor before continuing.

### Tools to use

| Task | Tool |
| --- | --- |
| Resolve database/data source IDs and read property schema | `notion-fetch` |
| Search workspace by name/keyword | `notion-search` |
| Query/filter/sort database rows | `notion-query-data-sources` |
| Create transaction/account/category rows | `notion-create-pages` |
| Update properties or page body | `notion-update-page` |
| Add new `Store` or `Tags` options | **Notion UI** (user adds select/multi_select options manually). MCP does not auto-create them on write. |

Before the first read or write against a data source, call `notion-fetch` on that data source (or its parent database URL) and use the exact property names/types from the fetched schema.

### Data source URLs

Config stores bare data source UUIDs. When calling MCP tools, prefix each ID as:

```text
collection://{data_source_id}
```

Example: if `transactionsDataSourceId` is `f336d0bc-b841-465b-8045-024475c079dd`, use `collection://f336d0bc-b841-465b-8045-024475c079dd` in queries and as the table name in SQL.

If the user provides a Notion database URL instead of a data source ID, call `notion-fetch` on the URL and read the `<data-source url="collection://...">` tag for the correct ID.

### Querying (`notion-query-data-sources`)

Prefer **SQL mode** (default) for filtered reads such as recent transactions, duplicate checks, and account lookup.

General shape:

```json
{
  "data": {
    "data_source_urls": ["collection://YOUR_DATA_SOURCE_ID"],
    "query": "SELECT url, \"Title\", \"date:Date:start\" FROM \"collection://YOUR_DATA_SOURCE_ID\" ORDER BY datetime(\"date:Date:start\") DESC LIMIT 60",
    "params": []
  }
}
```

Guidelines:

- Quote property names exactly as shown in `notion-fetch` schema output.
- Date properties use split columns such as `"date:Date:start"` and `"date:Date:is_datetime"`.
- For time comparisons or ordering, normalize with `datetime(...)` or `date(...)`.
- Prefer `?` placeholders via `params` instead of string interpolation.
- Relation filters/compare values using related page URLs or UUIDs from `notion-fetch` / query results.
- Checkbox values use `"__YES__"` / `"__NO__"`.
- Paginate with `LIMIT`/`OFFSET` or follow `has_more` / `next_cursor` when using view mode.

View mode is acceptable when an existing database view already encodes the needed filters and sorts.

### Creating rows (`notion-create-pages`)

Create database rows under the target data source:

```json
{
  "parent": { "data_source_id": "YOUR_DATA_SOURCE_ID" },
  "pages": [
    {
      "properties": {
        "Title": "Bread and milk",
        "date:Date:start": "2026-04-29T08:00:00",
        "date:Date:is_datetime": 1,
        "From Amount": 12,
        "From": ["https://www.notion.so/..."],
        "Category": ["category-page-url-or-id"],
        "Store": "Walmart"
      },
      "content": "Optional long notes in Notion-flavored Markdown"
    }
  ]
}
```

Guidelines:

- Use the fetched title property name (often `Title`, not `title`).
- Put long notes in `content`, not in a database property.
- Relation properties are arrays of page URLs or UUIDs.
- **`Store` and `Tags` values must already exist in the schema** returned by `notion-fetch` before you set them on `notion-create-pages` or `notion-update-page`. The Cursor Notion MCP validates against the known option list; unknown values fail with `validation_error` (unlike typing a new option in the Notion app).
- **Adding options**: Ask the user to add missing `Store` select options or `Tags` multi_select options in the Transactions database in Notion, then re-fetch or proceed with the write. Do not use MCP to add options unless the user explicitly requests a schema migration workflow.
- **Forbidden**: Do not use `notion-update-data-source` or `ALTER COLUMN "Store" SET SELECT(...)` to add or change `Store` options unless the user explicitly asks and you have a complete, verified option list. That statement replaces the entire option list; omitted options are removed and stale fetches can delete existing stores.
- `Store` allows at most **100** options. If the limit is reached, leave `Store` empty and ask the user which existing option to reuse or which option to remove in the Notion UI.

### Updating rows (`notion-update-page`)

Use `update_properties` for property changes and `insert_content` / `update_content` for page body edits.

## Config

1. Read data source IDs from `{THIS_SKILL_FOLDER}/assets/config.json`
2. Expect this structure:

```json
{
  "currenciesDataSourceId": "{data_source_id}",
  "accountsDataSourceId": "{data_source_id}",
  "transactionsDataSourceId": "{data_source_id}",
  "categoriesDataSourceId": "{data_source_id}"
}
```

3. Validate all four keys exist and each value is a non-empty data source ID string.
4. If `assets/config.json` does not exist or any ID is missing/invalid, stop and prompt the user to provide the data source IDs interactively and create the file before continuing. If the user provides a database URL, use `notion-fetch` to find the associated data source ID.

## Currencies Database (`currenciesDataSourceId`)

Required property:
- `Name` (`title`): currency code like `USD`.
- `Exchange Rate` (`number`): Exchange rate. E.g. USD: 1, TWD: 31

## Accounts Database (`accountsDataSourceId`)

Required properties:
- `Name` (`title`): unique account name identifier.
- `Type` (`select`): account class.
- `Currency` (`relation` -> `Currencies`): currency reference for this account.
- `Initial Balance` (`number`): opening balance value.

Usage requirements:
- `Type` should be one of: `General`, `Bank`, `Credit Card`, `Cash`, `Liability`.
- `Currency` must always relate to a page in `Currencies`.
- Use `Name` as the canonical key when linking account-related records.

## Categories Database (`categoriesDataSourceId`)

Required property:
- `Name` (`title`): category name taxonomy.

Usage requirements:
- Categories should be reused by exact name matching.

## Transactions Database (`transactionsDataSourceId`)

Required properties:
- `Date` (`date`): (Required) transaction date/time.
- `From` (`relation` -> `Accounts`): (Optional) source account.
- `To` (`relation` -> `Accounts`): (Optional) destination account.
- `From Amount` (`number`): (Optional) amount debited from `From`.
- `To Amount` (`number`): (Optional) amount credited to `To`.
- `Category` (`relation` -> `Categories`): (Required) transaction category.
- `Title` (`title`): (Optional) short transaction title.
- `Store` (`select`): (Optional) merchant/store label.
- `Tags` (`multi_select`): (Optional) transaction labels.

Usage requirements:
- The type of transaction is decided by the `From`, `To`, `From Amount`, and `To Amount` properties.
  - For transfer:
    - `From`, `From Amount`, `To`, `To Amount` are non-empty.
  - For income:
    - `From`, `From Amount` are empty
    - `To`, `To Amount` are non-empty
  - For expense:
    - `From`, `From Amount` are non-empty
    - `To`, `To Amount` are empty
- `Store` and `Tags` option lists are maintained in Notion (manual add in the UI). MCP writes only values that appear in the fetched schema.
- Long notes should be written to page content/body, not stored as a database property. Only use `Title` for one line short description.
