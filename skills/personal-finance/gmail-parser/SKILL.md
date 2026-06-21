---
name: gmail-parser
description: Scan Gmail for bank transaction emails and log new ones to the Notion Spending DB
---

## Overview

When Ferry says "scan emails", "scan my email", "sync spending", or "check for new transactions": search Gmail for bank alert emails using `mcp_gmail_autoauth_search_emails`, read each one with `mcp_gmail_autoauth_read_email`, parse the transaction, deduplicate against Notion, and log new ones.

## Step 1 — Search Gmail

Call `mcp_gmail_autoauth_search_emails` with:
```json
{
  "query": "from:(ibanking.alert@dbs.com OR paylah.alert@dbs.com OR alerts@citibank.com.sg OR donotreply@instarem.com OR dbsindonesia@1bank.dbs.com) after:YYYY/MM/DD",
  "maxResults": 30
}
```

Default lookback: 7 days. Use `after:YYYY/MM/DD` where the date is today minus 7 days.

The response is a list of messages. Each has an `id` (Gmail message ID) and metadata.

## Step 2 — Read each email

For each message, call `mcp_gmail_autoauth_read_email` with the message `id`.

The response is a string in this format:
```
Thread ID: {thread_id}
Subject: {subject}
From: {sender}
To: {recipient}
Date: {date}

[Note: This email is HTML-formatted. Plain text version not available.]

{raw HTML body}
```

Extract:
- `subject` — from the Subject header line
- `from` — from the From header line  
- `message_id` — the `id` from the search result (use this for deduplication)
- body text — parse from the raw HTML using the rules below

## Step 3 — Skip non-transactions

Skip the email if the subject contains any of:
- "successful bill payment"
- "manage card alert"
- "payment received"
- "balance"
- "limit"

## Step 4 — Parse the transaction

The HTML body contains the transaction details embedded in table cells. Use regex on the full response string.

### DBS Card Transaction Alert (`ibanking.alert@dbs.com`)

The body contains text like:
```
Date & Time: 21 Jun 17:41 (SGT)
Amount: SGD33.35
From: DBS/POSB card ending 6413
To: WELCIA SG (GUOCO TOWER
```

Extract with regex (run on the full read_email response string):
```
date_match    = re.search(r'Date\s*&amp;\s*Time:\s*(\d+\s+\w+)\s+\d+:\d+', body)
              # also try without &amp;: r'Date\s*&\s*Time:\s*(\d+\s+\w+)\s+\d+:\d+'
amount_match  = re.search(r'Amount:\s*SGD\s*(\d+\.?\d*)', body)
amount_fx     = re.search(r'Amount:\s*([A-Z]{3})\s*(\d+\.?\d*)', body)  # foreign currency
card_match    = re.search(r'card ending\s+(\d+)', body, re.IGNORECASE)
merchant_match = re.search(r'To:\s+([^\n<]+?)(?:\s+If unauthorized|\s*$)', body)
```

Card endings: `7468` → DBS CC, `6413` → DBS Debit.

Date format: `"21 Jun"` + current year → `2026-06-21`.

If `amount_fx` matches a non-SGD currency, store it in `Original Currency` + `Original Amount`.

### PayLah (`paylah.alert@dbs.com`)

Card = "PayLah". Same HTML table format as DBS card alerts. Three subtypes — all parsed identically:

| Ref prefix | Email says | `To:` field |
|---|---|---|
| PLPE... | "PayLah! Scan & Pay Transfer" | merchant name (all caps) |
| MT... | "PayLah! Bill Payment Transfer" | merchant name |
| TF... | "PayLah! Transfer" | "Person Name (Mobile ending XXXX)" |

PayLah body is an HTML table, so target the `<td>` structure to avoid matching the email `To:` header:
```
date_match     = re.search(r'Date\s*&amp;\s*Time:\s*(\d+\s+\w+)\s+\d+:\d+', body)
amount_match   = re.search(r'Amount:</td>\s*<td>\s*SGD\s*(\d+\.?\d*)', body)
merchant_match = re.search(r'To:</td>\s*<td>(.*?)</td>', body)
```

**P2P transfer (TF... ref):** `To:` is `"Gita Asaria (Mobile ending 8749)"`. Strip the `(Mobile ending XXXX)` suffix to get the recipient name as merchant. Set Category = "Other", Notes = "PayLah transfer out".

**Merchant payment (PLPE.../MT... ref):** `To:` is the merchant name directly.

### DBS Indonesia CC (`dbsindonesia@1bank.dbs.com`)

Subject: "Transaksi Kartu Kredit digibank Anda Berhasil"

Card = "DBS ID CC" (card ending 9988). Body is simple `<p>` tags:
```
4 digit Akhir Kartu: 9988
Merchant/ATM: SCOOT CAFE_SATS APS
Tanggal Transaksi: 14-03-2026
Nominal: SGD 9.00
```

Regex:
```
merchant_match = re.search(r'Merchant/ATM:\s*([^\n<&]+)', body)
date_match     = re.search(r'Tanggal Transaksi:\s*(\d{2}-\d{2}-\d{4})', body)
nominal_match  = re.search(r'Nominal:\s*([A-Z]{3})\s*(\d+[\.,]?\d*)', body)
```

Date parse: `"14-03-2026"` → DD-MM-YYYY → `2026-03-14`.

Currency: `nominal_match.group(1)` is the currency code. If SGD → use as `amount_sgd` directly. If IDR → record as `Original Currency=IDR`, `Original Amount=X`, and set `Amount (SGD)` to 0 (user will fill in the SGD equivalent manually).

Merchant: strip trailing `&nbsp;` and whitespace.

### Citi (`alerts@citibank.com.sg`)

Card = "Citi". Date format: `"29/04/26"` → parse as `2026-04-29`. Use SGD amount directly.

### Amaze / InstaRem (`donotreply@instarem.com`)

Card = "Amaze". Use "Amount paid" SGD field as `amount_sgd`. If "Transaction amount" in foreign currency, store in `Original Currency` + `Original Amount`. Date format: `"27th May, 2026"`.

### Category — best guess from merchant name

- Food: restaurants, cafes, food delivery, CRAVE, McDonald's, Grab Food, supermarkets
- Transport: Grab rides, taxis, MRT, parking, petrol, ComfortDelGro
- Shopping: retail, clothing, electronics, Shopee, Lazada, NTUC, pharmacy (NTUC)
- Entertainment: movies, games, streaming, Netflix, Spotify
- Health: clinics, gyms, Guardian, Watsons, dental
- Other: everything else

## Step 5 — Deduplicate against Notion

Before creating any page, check if the Gmail message ID already exists.

Call `mcp_notion_API_post_database_query`:
```json
{
  "database_id": "36de1afd-dd9c-819b-aea3-e0d9710d8fa8",
  "filter": {
    "property": "Gmail Message ID",
    "rich_text": {"equals": "MESSAGE_ID"}
  }
}
```

If results are returned → already logged, skip.

## Step 6 — Log to Notion

For each new transaction, call `mcp_notion_API_post_page`:
```json
{
  "parent": {"database_id": "36de1afd-dd9c-819b-aea3-e0d9710d8fa8"},
  "properties": {
    "Merchant":         {"title": [{"text": {"content": "MERCHANT"}}]},
    "Date":             {"date": {"start": "YYYY-MM-DD"}},
    "Amount (SGD)":     {"number": 0.00},
    "Card":             {"select": {"name": "CARD"}},
    "Category":         {"select": {"name": "CATEGORY"}},
    "Source":           {"select": {"name": "Email"}},
    "Gmail Message ID": {"rich_text": [{"text": {"content": "MESSAGE_ID"}}]}
  }
}
```

Optional: `"Original Currency"`, `"Original Amount"` if FX transaction.

## Step 7 — Report

After scanning all emails, report:
```
Auto-logged {n} transaction(s) ✓
```

Then show today's expenses using `mcp_notion_API_post_database_query` for today's date.
