---
name: notion-accounting-schema
description: Defines how the Notion databases should be structured and used for personal accounting. Use when user wants to read or write transactions in the Notion databases.
---

## Purpose

This skill illustrates how to use Notion databases for personal accounting.  
Use this skill whenever the user asks to read, write, or update the Notion databases for transactions.

## Config

1. Read database URLs from `assets/config.json`.
2. Expect this structure:

```json
{
  "currenciesDb": "https://www.notion.so/{db_id}?v={view_id}",
  "accountsDb": "https://www.notion.so/{db_id}?v={view_id}",
  "transactionsDb": "https://www.notion.so/{db_id}?v={view_id}",
  "categoriesDb": "https://www.notion.so/{db_id}?v={view_id}"
}
```

3. Validate all four keys exist and each value is a non-empty URL string.
4. If `assets/config.json` does not exist or any URL is missing/invalid, stop and prompt the user to provide the database URLs before continuing.

## Currencies Database (`currenciesDb`)

Required property:
- `Name` (`title`): currency code like `USD`.
- `Exchange Rate` (`number`): Exchange rate. E.g. USD: 1, TWD: 31

## Accounts Database (`accountsDb`)

Required properties:
- `Name` (`title`): unique account name identifier.
- `Type` (`select`): account class.
- `Currency` (`relation` -> `Currencies`): currency reference for this account.
- `Initial Balance` (`number`): opening balance value.

Usage requirements:
- `Type` should be one of: `General`, `Bank`, `Credit Card`, `Cash`, `Liability`.
- `Currency` must always relate to a page in `Currencies`.
- Use `Name` as the canonical key when linking account-related records.

## Categories Database (`categoriesDb`)

Required property:
- `Name` (`title`): category name taxonomy.

Usage requirements:
- Categories should be reused by exact name matching.

## Transactions Database (`transactionsDb`)

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
- `Store` options and `Tags` values may be created on demand.
- Long notes should be written to page content/body, not stored as a database property. Only use `Title` for one line short description.
