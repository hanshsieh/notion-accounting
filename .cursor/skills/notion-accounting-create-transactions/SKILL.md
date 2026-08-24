---
name: notion-accounting-create-transactions
description: Reads transaction details or receipts, extracts structured fields, maps them to Notion transaction schema, and creates one or more records in Notion database. Use when the user provides bank statements, transaction history, or receipts (screenshots, CSV, or other formats) and asks to log transactions into Notion.
---
## Prerequisites

- Use `{THIS_SKILL_FOLDER}/../notion-accounting-schema/SKILL.md` to decide the Notion database location, schema, and MCP tool usage.
- Use connected **Notion** integration MCP (see schema skill).
- If image text is unclear, ask user for a clearer crop or missing fields before creating records.

## Workflow

1. **Read the transactions**  
   From user-provided transactions (image, CSV, ...), extract everything visible: merchant/payee, statement note, amount, currency, date and time, tax, tips, category labels, and payment method hints.  
   Refer to these as **extracted transactions**.

1. **Identify the account for the transactions**
   Identify corresponding account(s) for the transactions.  
   Examples:
   - Bank statements: the bank account.
   - Credit card receipts: the credit card account.
   - Points deduction: the points account, such as LINE points or credit card bonus points.

   Search corresponding accounts in Notion using `notion-query-data-sources` against `accountsDataSourceId` (SQL `WHERE` on the title/`Name` property), or use `notion-search` when name matching is fuzzy.  
   Keep each matched account's page URL/UUID for relation filters and writes.  
   If unclear, stop and confirm with the user before proceeding.

1. **Read recent transactions of the account(s) for habit analysis**
   For each account, query the latest 60 transactions from `transactionsDataSourceId` with `notion-query-data-sources` in SQL mode.  
   If Notion MCP fails with authentication or permission errors, stop and ask the user to reconnect Notion in Cursor settings.  
   Sort by `Date` descending, and filter to records where the account appears in `From` or `To` (use the account page URL/UUID in the SQL `WHERE` clause).  
   This query is for learning user recording habits (title style, category/store choices, transfer split patterns).  
   This is NOT the duplicate-detection time window as mentioned below.

1. **Split transactions when needed**
   Some extracted transactions need to be split.  
   If one extracted transaction uses multiple accounts, split it into multiple transactions.  
   For example, a payment using both credit card and points deduction should be split into 2 transactions.  
   If a user paid 2 loans of the same bank in one transaction, it may need to be split into two transfer transactions. Use recent transactions to decide how to split.

1. **Choose transaction type**
   For each extracted transaction, choose the transaction type.  
   Examples:
   - Purchase / fee / outgoing payment -> `Expense`
   - Salary / refund to account -> `Income`
   - Move between user's own accounts -> `Transfer`
   - Paying credit card bills -> `Transfer`
   - Paying loan -> `Transfer`
   - Loan disbursement -> `Transfer`

   This `Type` label is for workflow and summary only.  
   Do not write a `Type` property to Notion `transactionsDataSourceId`; the effective type is determined by `From`/`To` and amount fields per schema skill.

1. **Identify duplicate transactions**
   Decide whether extracted transactions already exist in Notion.  
   Time in Notion may not exactly match extracted transactions.  
   Steps:
   - Use one SQL query for the whole extracted batch: compute min/max extracted datetime, then query once with `Date` between `(min - 24h)` and `(max + 24h)` using `notion-query-data-sources`.
   - In that same query, limit records to the target account direction (`From` account and/or `To` account) for this import.
   - From that candidate set, check whether existing transactions have the same amount around neighboring time.
   - Do not reuse the "recent 60 transactions" habit-analysis query result for duplicate detection.
 
   For duplicate transactions, skip choosing store and category.  

1. **Choose store**
   For transfer, no store.  
   For expense and income, choose store based on extracted transaction and user habit.  
   First, find an appropriate store based on history (query distinct `Store` values from recent transactions or categories). If there is no direct match, search for a potential existing store.  
   Include the chosen store name in the summary even when it is not in the fetched `Store` select options — do not skip `Store` or leave it empty just because the option is missing. New options are added automatically when the transaction row is written (see **Create the transactions**).  
   If the appropriate store is still unclear after searching habits, leave it empty.  

1. **Choose category**
   Each non-duplicate transaction must have a category.  
   Resolve categories by querying `categoriesDataSourceId` (exact `Name` match) or from recent transaction habits.  
   Choose category based on transaction and user habit. If unclear, use a generic category like `Other` if available. If not available, leave it empty and mark `🤔 Need Confirm`.
   Exception: transactions marked as duplicate do not require a final category because they will be skipped.

1. **Summarize transactions to create**
   Summarize the extracted transactions (from newest to oldest, after spliting, don't group by status) for user review.  
   DO NOT directly create transactions.  
   Template for each transaction (localize for user's language):
   ```
   Time: 2026/04/29 08:00 AM
   Type: Income
   Account: City Bank(USD)
   Amount: 12 USD
   Category: ❓
   Store: Walmart
   Title: Bread and milk
   Note:
     Bread: $5
     Milk: $7
   Status: 🤔 Need Confirm
     Cannot find an appropriate category
   ```

   - Time
     Date and time of transaction. For credit card bill, use posting date.
   - Type
     `Income`, `Expense`, `Transfer` (summary only, not a Notion property)
   - Account
     For `Income` and `Expense`, should be the Notion account.  
     For `Transfer`, should be 2 Notion accounts for transfer. Example: `City Bank(USD) -> Cash`
   - Amount
     For `Income`, `Expense`, and `Transfer` with same amount and currency, use format like `100 USD`.  
     For `Transfer` with different amount or currency, use format like `10 USD -> 300 TWD`
   - Category
     Notion category for transaction.  
     This must be resolved before creation, except duplicates that are skipped.
   - Store
     (Optional) Store of transaction. For transfer, skip this field.
     If unclear, it can be left empty without changing status from `✅ OK`.
   - Title
     One-line short description of transaction.
   - Note
     (Optional) Put supplemental info from extracted transaction.  
     Store it in the page `content` when creating/updating the Notion transaction page.  
     Do not include PII such as credit card number.
   - Status
     Status of transaction, one of:
     - ✅ OK: everything is good
     - ❌ Duplicate: transaction already exists
     - 🤔 Need Confirm: include additional text for details requiring confirmation

   If user asks for changes, apply changes and show above summary again before creating transactions.  
   Only create transactions after getting explicit user approval.

1. **Create the transactions**  
   Create approved rows with `notion-create-pages`, parent set to `transactionsDataSourceId`.  
   Map fields to the fetched schema (for example: `Date`, `From`, `To`, `From Amount`, `To Amount`, `Category`, `Title`, `Store`), using relation arrays of page URLs/UUIDs and date split properties per the schema skill.  
   Set `Store` to the chosen store name string even when that name is not in the current schema options; Notion creates missing select options automatically on page create/update.  
   Put long notes in each page's `content`.  
   **Forbidden**: Do not call `notion-update-data-source` or `ALTER COLUMN "Store" SET SELECT(...)` to add `Store` options. That replaces the entire option list and can delete existing stores if the fetched schema is incomplete or stale.  
   If `notion-create-pages` rejects an unknown `Store` value (rare), create the row without `Store`, then set `Store` with `notion-update-page`. If that also fails (for example the 100-option limit), leave `Store` empty and note it in the result summary for manual follow-up.  
   Summarize result, such as:
   ```
   Success: 10
   Error: 0
   ```
