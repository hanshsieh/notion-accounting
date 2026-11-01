---
name: notion-accounting-create-transactions
description: Reads transaction details or receipts, extracts structured fields, reconciles them against Notion, and suggests creating, updating, or deleting transaction records so Notion matches the user-provided source. Use when the user provides bank statements, credit card bills, transaction history, or receipts (screenshots, CSV, Excel, or other formats) and asks to log or reconcile transactions in Notion.
---
## Prerequisites

- Use `{THIS_SKILL_FOLDER}/../notion-accounting-schema/SKILL.md` to decide the Notion database location, schema, and MCP tool usage.
- Use connected **Notion** integration MCP (see schema skill).
- If image text is unclear, ask user for a clearer crop or missing fields before proposing changes.

## Source of truth

Treat user-provided bank statements, credit card bills, and similar account exports as the **source of truth** for the covered account(s) and date range.

Your job is to **reconcile** Notion with that source:
- **Create** transactions that exist in the source but not in Notion.
- **Update** Notion transactions that match a source row but have missing or incorrect fields.
- **Delete** Notion transactions in the covered period/account that do not appear in the source and were likely entered incorrectly.

For a single receipt or one-off payment (not a full statement), focus on create vs duplicate/update for that item only. Do not suggest deletions unless the user explicitly asks to reconcile a period.

## Workflow

1. **Read the transactions**  
   From user-provided transactions (image, CSV, Excel, ...), extract everything visible: merchant/payee, statement note, amount, currency, date and time, tax, tips, category labels, and payment method hints.  
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
   This is NOT the reconciliation query described below.

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

1. **Reconcile with Notion**
   Compare extracted transactions against existing Notion records for the same account(s) and statement period.

   **Query existing Notion transactions**
   - Compute min/max datetime from the extracted batch.
   - Query once with `notion-query-data-sources`, `Date` between `(min - 7d)` and `(max + 7d)`.
   - In that same query, limit records to the target account direction (`From` account and/or `To` account) for this import.
   - Fetch fields needed for comparison: `url`, `Title`, `Date`, `From`, `To`, `From Amount`, `To Amount`, `Category`, `Store`, and page content when notes may differ.
   - Do not reuse the "recent 60 transactions" habit-analysis query result for reconciliation.

   **Match extracted rows to Notion rows**
   - Match on account direction, amount, and nearby time. Times in Notion may not exactly match the source.
   - Use merchant/title/note hints when multiple candidates share amount and date.
   - Prefer one-to-one matching. If one source row clearly maps to multiple Notion rows, treat extras as delete candidates. If multiple source rows map to one Notion row, prefer update over create and flag ambiguous cases for confirmation.

   **Decide the action for each extracted transaction**
   - **Create**: no reasonable Notion match.
   - **Update**: a Notion match exists but fields differ from the source or the source has richer information.
   - **Skip**: a Notion match exists and already matches the source. Show these only when useful; they do not need user approval.

   **Decide delete candidates from Notion**
   - Within the same queried period and account scope, any Notion transaction with no matching extracted row is a **delete** candidate.
   - Delete suggestions are destructive. Default status to `🤔 Need Confirm` and explain why the row looks orphaned or incorrect.
   - Do not suggest deleting transactions outside the source's covered date range or account unless the user explicitly asks.

   **Fields to compare for updates**
   Compare at least: `Date`, accounts (`From`/`To`), amounts (`From Amount`/`To Amount`), `Category`, `Store`, `Title`, and page `content` notes.

   Suggest an update when:
   - A core field is wrong (amount, date, account, title).
   - Notion is missing information the source provides (for example empty `Store` but the bill shows a merchant).
   - The source suggests a better `Category` or `Store` than what is currently recorded.
   - Supplemental note/content from the source should be added or corrected.

   Do not suggest cosmetic-only updates that contradict established user habits unless the source clearly overrides them.

1. **Choose store**
   For transfer, no store unless the user clearly tracks one.  
   For expense and income:
   - **Create** and **Update** actions: choose store based on extracted transaction and user habit.
   - **Skip** and **Delete** actions: do not choose a new store.

   First, find an appropriate store based on history (query distinct `Store` values from recent transactions or categories). If there is no direct match, search for a potential existing store.  
   Cross-check proposed `Store` names against the `Store` options from `notion-fetch` on `transactionsDataSourceId`. If a proposed store is not in that list, still show the intended name in the summary but flag it under **Schema additions (manual in Notion)** (see **Summarize proposed changes**).  
   If the appropriate store is still unclear after searching habits, leave it empty.

1. **Choose tags (when used)**
   Only set `Tags` when user habit or the source clearly calls for them (for example `悠遊卡加值`, `刷卡`, `蝦皮`).  
   Cross-check each proposed tag against the `Tags` options from `notion-fetch`. Tags not in the schema must be listed under **Schema additions (manual in Notion)** before apply.

1. **Choose category**
   Each **Create** action must have a category.  
   Each **Update** action should include a category only when you are proposing a category change or fill-in.  
   **Skip** and **Delete** actions do not need a proposed category.

   Resolve categories by querying `categoriesDataSourceId` (exact `Name` match) or from recent transaction habits.  
   Choose category based on transaction and user habit. If unclear, use a generic category like `Other` if available. If not available, leave it empty and mark `🤔 Need Confirm`.

1. **Summarize proposed changes**
   Summarize all proposed changes (from newest to oldest, after splitting; do not group by status) for user review.  
   DO NOT directly create, update, or delete transactions.  
   Include separate items for **Create**, **Update**, and **Delete** actions. Optionally list **Skip** items briefly in a totals section.

   Template for each item (localize for user's language):
   ```
   Action: Create
   Time: 2026/04/29 08:00 AM
   Type: Income
   Account: City Bank(USD)
   Amount: 12 USD
   Category: Salary
   Store:
   Title: Payroll deposit
   Note:
     Employer: ACME Corp
   Status: ✅ OK
   ```

   ```
   Action: Update
   Notion: https://www.notion.so/...
   Time: 2026/04/28 06:30 PM
   Type: Expense
   Account: Taishin Credit Card
   Amount: 350 TWD
   Category: Groceries
   Store: PX Mart
   Title: Grocery run
   Note:
   Changes:
     Store: (empty) -> PX Mart
     Category: Other -> Groceries
   Status: ✅ OK
   ```

   ```
   Action: Delete
   Notion: https://www.notion.so/...
   Time: 2026/04/27 01:15 PM
   Type: Expense
   Account: Taishin Credit Card
   Amount: 199 TWD
   Category: Dining
   Store:
   Title: Lunch
   Note:
   Status: 🤔 Need Confirm
     Not found on the provided statement; likely entered twice or by mistake
   ```

   Field notes:
   - **Action**
     One of `Create`, `Update`, `Delete`, or `Skip`.
   - **Notion**
     Required for `Update`, `Delete`, and `Skip`. The matched Notion page URL.
   - **Time**
     Date and time of transaction. For credit card bills, use posting date.
   - **Type**
     `Income`, `Expense`, `Transfer` (summary only, not a Notion property)
   - **Account**
     For `Income` and `Expense`, the Notion account.  
     For `Transfer`, two Notion accounts. Example: `City Bank(USD) -> Cash`
   - **Amount**
     For `Income`, `Expense`, and `Transfer` with same amount and currency, use format like `100 USD`.  
     For `Transfer` with different amount or currency, use format like `10 USD -> 300 TWD`
   - **Category**
     Proposed Notion category for `Create` and `Update`. Omit for `Delete` unless helpful context.
   - **Store**
     Optional merchant/store. For transfer, usually omit.
     If unclear, it can be left empty without changing status from `✅ OK`.
   - **Title**
     One-line short description of transaction.
   - **Note**
     Optional supplemental info from the extracted transaction.  
     Store it in the page `content` when creating/updating the Notion transaction page.  
     Do not include PII such as credit card number.
   - **Changes**
     Required for `Update`. List only fields that will change, using `old -> new`. Use `(empty)` for missing values.
   - **Status**
     One of:
     - ✅ OK: safe to apply as proposed
     - ⏭️ Skip: already matches the source; no change needed
     - 🤔 Need Confirm: include additional text for ambiguous matches, conflicting values, or destructive deletes

   End the summary with counts such as:
   ```
   Proposed: Create 8, Update 3, Delete 1, Skip 12
   ```

   **Schema additions (manual in Notion)**  
   After the change list, if any proposed **Create** or **Update** uses a `Store` or `Tags` value that is not in the schema from `notion-fetch`, add a dedicated section (do not bury it in a single line). List each missing value and which transactions need it. Ask the user to choose before you write:
   - Add the option(s) in Notion, then confirm **ready to apply** (or ask you to re-fetch and verify), or
   - Apply now **without** those properties (keep merchant in `Title` / page `content`), or
   - Map to an existing `Store` / `Tags` option they name.

   Example:
   ```
   Schema additions (manual in Notion)
   Store (select): 金興發生活百貨 — needed for Create 2026/09/13 99 TWD
   Tags (multi_select): (none)
   Please add in Notion or tell me how to proceed before I apply creates/updates.
   ```

   If the user approves apply while additions are still missing, omit only the failing properties and say so in the result summary.

   If the user asks for changes, apply them and show the summary again before writing to Notion.  
   Only apply changes after explicit user approval (including resolution of schema additions when listed).

1. **Apply approved changes**

   **Create**  
   Create approved rows with `notion-create-pages`, parent set to `transactionsDataSourceId`.  
   Map fields to the fetched schema (for example: `Date`, `From`, `To`, `From Amount`, `To Amount`, `Category`, `Title`, `Store`, `Tags`), using relation arrays of page URLs/UUIDs and date split properties per the schema skill.  
   Set `Store` and `Tags` only when each value is present in the schema from `notion-fetch` (see schema skill).  
   Put long notes in each page's `content`.  
   **Forbidden**: Do not call `notion-update-data-source` or `ALTER COLUMN "Store" SET SELECT(...)` to add `Store` options unless the user explicitly requests that migration with a verified full option list.  
   If `notion-create-pages` or `notion-update-page` rejects an unknown `Store` or `Tags` value, create or update the row without that property, keep the merchant in `Title` / `content`, and note in the result summary that the user should add the option in Notion and set the property manually (or re-run after they add it).

   **Update**  
   Update approved rows with `notion-update-page` and `command: "update_properties"` for changed properties.  
   Use `insert_content`, `update_content`, or `replace_content` only when note/content changes are approved.  
   Include only fields listed under `Changes` in the approved summary.

   **Delete**  
   For approved deletions, remove the matched Notion transaction page.  
   If the Notion MCP exposes a page trash/archive action, use it.  
   If not, give the user the list of approved Notion URLs to delete manually in Notion and do not claim the delete succeeded.

   Summarize the result, such as:
   ```
   Created: 8
   Updated: 3
   Deleted: 1
   Skipped: 12
   Errors: 0
   ```
