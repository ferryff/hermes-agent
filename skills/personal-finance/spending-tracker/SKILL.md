---
name: spending-tracker
description: Log, query, find, and edit expenses in Notion Spending DB
---

## Overview

This skill manages the Notion Spending DB (`36de1afd-dd9c-819b-aea3-e0d9710d8fa8`). All operations use `mcp_notion_API_*` tools directly — no terminal Python.

## Fields

| Field | Type | Valid values |
|---|---|---|
| Merchant | title | payee name |
| Date | date | YYYY-MM-DD |
| Amount (SGD) | number | always in SGD |
| Card | select | DBS CC, DBS Debit, PayLah, Citi, Amaze, Revolut |
| Category | select | Food, Transport, Shopping, Entertainment, Health, Travel, Bills, Education, Personal Care, Other |
| Source | select | Manual, Email |
| Original Currency | rich_text | 3-letter code e.g. MYR, USD |
| Original Amount | number | FX amount |
| Notes | rich_text | free text |
| Gmail Message ID | rich_text | used for email deduplication |

## Log a new expense

Parse from user message: merchant, amount, card, category, date. Ask if any required field is ambiguous.

**Always confirm before writing.** Show: merchant, amount, card, category, date, notes. Wait for "yes".

Call `mcp_notion_API_post_page`:
```json
{
  "parent": {"database_id": "36de1afd-dd9c-819b-aea3-e0d9710d8fa8"},
  "properties": {
    "Merchant":     {"title": [{"text": {"content": "Grab"}}]},
    "Date":         {"date": {"start": "2026-06-21"}},
    "Amount (SGD)": {"number": 8.50},
    "Card":         {"select": {"name": "PayLah"}},
    "Category":     {"select": {"name": "Transport"}},
    "Source":       {"select": {"name": "Manual"}}
  }
}
```

## Query expenses by date range

Call `mcp_notion_API_post_database_query`:
```json
{
  "database_id": "36de1afd-dd9c-819b-aea3-e0d9710d8fa8",
  "filter": {
    "and": [
      {"property": "Date", "date": {"on_or_after": "2026-06-01"}},
      {"property": "Date", "date": {"on_or_before": "2026-06-21"}}
    ]
  },
  "sorts": [{"property": "Date", "direction": "ascending"}]
}
```

Format each result as: `{date} | {card} | {merchant} ${amount:.2f} | {category}` (include FX if `Original Currency` set). Sum `Amount (SGD)` for totals.

## Get month total

Query `Date on_or_after` first of current month, `on_or_before` today. Sum `Amount (SGD)`.

## Find expenses by merchant

Call `mcp_notion_API_post_database_query`:
```json
{
  "database_id": "36de1afd-dd9c-819b-aea3-e0d9710d8fa8",
  "filter": {
    "and": [
      {"property": "Merchant", "title": {"contains": "QUERY"}},
      {"property": "Date", "date": {"on_or_after": "60-days-ago"}}
    ]
  },
  "sorts": [{"property": "Date", "direction": "descending"}]
}
```

Return numbered list: `1. {date} | {card} | {merchant} ${amount:.2f} | {category} | id:{page.id}`

## Update category

Require page_id (from find result). Confirm before writing.

Call `mcp_notion_API_patch_page`:
```json
{
  "page_id": "PAGE_ID",
  "properties": {"Category": {"select": {"name": "Food"}}}
}
```

## Add/update note

Require page_id (from find result). Confirm before writing.

Call `mcp_notion_API_patch_page`:
```json
{
  "page_id": "PAGE_ID",
  "properties": {"Notes": {"rich_text": [{"text": {"content": "Business expense"}}]}}
}
```

## Cron: Daily summary

Schedule: `0 22 * * *` (10pm SGT)

Query today's expenses + month total. Send to Telegram:
- No spending: `No spending logged today ✓\nMonth so far: $XX.XX`
- Has spending:
  ```
  💳 Today's spending:

  {card}  — {merchant} ${amount}
  ...

  Total today: $XX.XX
  Month so far: $XX.XX
  ```

## Cron: Weekly summary

Schedule: `0 22 * * 5` (Friday 10pm SGT)

Query last week (Mon–Sun). Group by category, sort descending. One data-grounded insight.

```
📊 Week {start} → {end}:

{category}      $XX.XX
...

Total: $XX.XX

💡 {insight}
```
