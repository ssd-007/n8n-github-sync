# AI Invoice Processor

Automatically detects invoice emails in Gmail, extracts structured data using AI, organises the inbox, and logs everything to Google Sheets.

---

## What It Does

This workflow monitors a Gmail inbox for new unread emails. Each email is classified by Claude AI as an invoice (or not). Invoices are labelled, archived out of the inbox, and their data — vendor, amount, currency, date, invoice number — is extracted from the PDF attachment or email body and appended to a Google Sheets expense tracker. A daily Telegram summary is sent at 9am.

Two entry points: the Gmail trigger polls every minute in production; a manual trigger fetches up to 100 unread emails for bulk first-run processing.

---

## Flow

Manual Trigger → Get Many Messages (up to 100 unread) ──┐
├──► AI Agent (Claude)
Watch Gmail Inbox (every minute) ────────────────────────┘        ↓
Is Invoice Email?
/         

YES          NO
↓          (stop)
Get Email with Attachment
↓
Extract PDF Text
↓
Label as Expenses
↓
Remove from Inbox
↓
Mark as Read
↓
Prepare AI Input
↓
AI: Extract Invoice Data
↓
Format Invoice Row
↓
Google Sheets

Daily Summary Trigger (9am) + Manual Trigger
↓
Read Expense Tracker     ← get all rows from Google Sheet
↓
Build Daily Summary        ← filter yesterday + today, totals by currency
↓
Telegram              ← send summary to private chat



---


---

## Nodes

- **Watch Gmail Inbox** — polls every minute for new unread emails (production trigger)
- **Get many messages** — fetches up to 100 unread emails for initial bulk processing (manual trigger)
- **AI Agent** — Claude Sonnet classifies each email: invoice, receipt, bill, or order confirmation? Replies YES or NO
- **Is Invoice Email?** — routes YES to processing, NO stops here
- **Get Email with Attachment** — re-fetches the full email with PDF binary (Simplify OFF required for attachments)
- **Extract PDF Text** — extracts text from PDF; continues gracefully if no PDF attached
- **Label as Expenses** — adds the Expenses label to the email in Gmail
- **Remove from Inbox** — removes INBOX label, effectively moving the email to the Expenses folder
- **Mark as Read** — marks the email as read to prevent re-processing
- **Prepare AI Input** — combines PDF text (or email body fallback) with email metadata into structured input
- **AI: Extract Invoice Data** — Claude extracts vendor, date, amount, currency, invoice number, description
- **Format Invoice Row** — maps AI output to Google Sheet column names
- **Append or update row in sheet** — writes to the expense tracker; deduplicates by Gmail Message ID
- **Daily Summary Trigger** — fires at 9am daily
- **Read Expense Tracker** — reads all rows from the Google Sheet
- **Build Daily Summary** — filters to recent invoices, calculates totals by currency, lists vendors
- **Send Daily Summary** — sends the summary to Telegram with HTML formatting

---

## Credentials

All credentials are stored in the n8n credential store — no keys are in the workflow JSON.

| Service   | Used By                                                                                              |
|-----------|------------------------------------------------------------------------------------------------------|
| Gmail     | Watch Gmail Inbox, Get many messages, Get Email with Attachment, Label as Expenses, Remove from Inbox, Mark as Read |
| Anthropic | Anthropic Chat Model, Anthropic Chat Model1                                                          |
| Google Sheets | Append or update row in sheet, Read Expense Tracker                                              |
| Telegram  | Send Daily Summary                                                                                   |

---

## Why This Was Built

Manual invoice triaging is repetitive and error-prone — sorting through inboxes, identifying which emails are invoices, extracting amounts and vendor details, filing them away. This workflow eliminates that manual work entirely. AI handles the classification and data extraction, Gmail stays organised automatically, and every invoice lands in a structured spreadsheet ready for bookkeeping or reconciliation.
