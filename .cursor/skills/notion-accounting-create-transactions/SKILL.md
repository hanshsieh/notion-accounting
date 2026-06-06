---
name: notion-accounting-create-transactions
description: Reads transaction details or receipts, extracts structured fields, maps them to Notion transaction schema, and creates one or more records in Notion database. Use when the user provides bank statements, transaction history, or receipts (screenshots, CSV, or other formats) and asks to log transactions into Notion.
---
## Prerequisites

- Use `{THIS_SKILL_FOLDER}/../notion-accounting-schema/SKILL.md` to decide the Notion database location and schema.
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

   Search corresponding accounts in Notion `accountsDataSourceId`.  
   If unclear, stop and confirm with the user before proceeding.

1. **Read recent transactions of the account(s) for habit analysis**
   For each account, read recent 60 transactions from `transactionsDataSourceId`.  
   If you Notion MCP failed with authentication errors, stop and ask the user the configure the `.env` with the Notion token.  
   Query transactions sorted by `Date` descending, and focus on records related by `From` or `To`.  
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
   - Use a query for the whole extracted batch: compute min/max extracted datetime, then query once with  `Date` between `(min - 24h)` and `(max + 24h)`.
   - In that same query, limit records to the target account direction (`From Account` -> `To Account`) for this  import.
   - From that candidate set, check whether existing transactions have the same amount around neighboring time.
   - Do not reuse the "recent 30 transactions" habit-analysis query result for duplicate detection.
 
   For duplicate transactions, skip choosing store and category.  

1. **Choose store**
   For transfer, no store.  
   For expense and income, choose store based on extracted transaction and user habit.  
   First, find an appropriate store based on history. If there is no direct match, search for a potential existing store. If no suitable existing store is found, create a new store. If the appropriate store is still unclear, leave it empty.  

1. **Choose category**
   Each non-duplicate transaction must have a category.  
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
     Store it in the `body` of Notion transaction page.  
     Do not include PII such as credit card number.
   - Status
     Status of transaction, one of:
     - ✅ OK: everything is good
     - ❌ Duplicate: transaction already exists
     - 🤔 Need Confirm: include additional text for details requiring confirmation

   If user asks for changes, apply changes and show above summary again before creating transactions.  
   Only create transactions after getting explicit user approval.

1. **Create the transactions**  
   Create transactions in Notion.  
   Map fields to `transactionsDataSourceId` schema (for example: `Date`, `From`, `To`, `From Amount`, `To Amount`, `Category`, `Title`, `Store`), and write long notes to page body/content.  
   Summarize result, such as:
   ```
   Success: 10
   Error: 0
   ```
