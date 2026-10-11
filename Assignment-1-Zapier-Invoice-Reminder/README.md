# Assignment 1 – Scheduled Invoice Payment Reminders with Reply Handling (Zapier)

> **Status: in progress.** The design, email templates and invoice sheet are ready; the Zaps are being built, and screenshots will be added once they have run.

## 1. Problem Statement

The accounts team tracks invoices in a spreadsheet. Every day someone has to check which invoices are still **Pending**, email each party a reminder, and later, when a party replies that the payment is done, find the invoice and mark it **Completed**. This is repetitive and easy to miss.

This assignment automates the full loop with Zapier:

1. **Zap 1 – Scheduled Payment Reminder:** every morning, read the invoice sheet, pick the Pending invoices, send each party a personalised reminder email, and log when the reminder was sent.
2. **Zap 2 – Reply Handling (Payment Confirmation):** when a party replies "Payment Done", extract the Invoice ID from the email subject, update that invoice to Completed with the confirmation date, and send an automatic acknowledgement reply.

## 2. Tools Used

| Tool | Purpose |
|---|---|
| Schedule by Zapier | Daily trigger at 9:00 AM IST |
| Google Sheets | Invoice tracker (data source and status log) |
| Looping by Zapier | Process each pending invoice separately |
| Gmail | Send reminders, watch for replies, send acknowledgement |
| Filter by Zapier | Only continue for genuine "Payment Done" replies on Pending invoices |
| Formatter by Zapier | Extract the Invoice ID from the subject; generate the timestamp |

**LLM used:** None. This assignment is rule-based workflow automation; no language model is involved. The email text is a fixed template with dynamic fields (see `Prompt.md`).

## 3. Input Data

`SCIT_Invoice_Tracker.xlsx` is the invoice table given in class (Sr, Invoice ID, Type, Name, Amount, Account No, Bank Name, Status) with three columns added for the automation:

| Column | Filled by | Purpose |
|---|---|---|
| Email | You (test addresses) | Where the reminder is sent |
| Reminder Sent On | Zap 1 | Audit trail of when the reminder went out |
| Payment Confirmed On | Zap 2 | Date the payment confirmation was received |

The Status values were normalised to exactly `Pending` / `Completed` (the original sheet had `completed` and `Completed ` with a trailing space, which would break an exact-match filter).

| Invoice ID | Name | Amount (₹) | Bank | Status |
|---|---|---|---|---|
| 1001 | Madhav | 50,000 | SBI | Pending |
| 1002 | Ram | 28,000 | ICICI | Pending |
| 1003 | Pavan | 94,000 | HDFC | Pending |
| 1004 | Raj | 80,000 | Kotak | Completed |
| 1005 | Veeru | 85,000 | DBS | Completed |

Expected result of Zap 1: 3 reminder emails (1001, 1002, 1003); 1004 and 1005 are skipped.

## 4. Workflow Architecture

### Zap 1 – Scheduled Payment Reminder

```text
Schedule by Zapier (every day, 9:00 AM IST)
        ↓
Google Sheets – Lookup Spreadsheet Rows (Advanced)   [Status = Pending]
        ↓
Looping by Zapier – Create Loop From Line Items      [one run per invoice]
        ↓
Gmail – Send Email                                   [personalised reminder]
        ↓
Formatter by Zapier – Date/Time                      [current IST timestamp]
        ↓
Google Sheets – Update Spreadsheet Row               [Reminder Sent On]
```

### Zap 2 – Reply Handling and Payment Confirmation

```text
Gmail – New Email Matching Search                    [replies to reminders]
        ↓
Filter by Zapier                                     [body contains "Payment Done"]
        ↓
Formatter by Zapier – Text – Extract Pattern         [Invoice ID from subject]
        ↓
Google Sheets – Lookup Spreadsheet Row               [find the invoice]
        ↓
Filter by Zapier                                     [Status is still Pending]
        ↓
Formatter by Zapier – Date/Time                      [current IST timestamp]
        ↓
Google Sheets – Update Spreadsheet Row               [Status = Completed, Payment Confirmed On]
        ↓
Gmail – Reply to Email                               [acknowledgement to the sender]
```

## 5. Step-by-Step Build

### Setup
1. Upload `SCIT_Invoice_Tracker.xlsx` to Google Drive and open it with Google Sheets (**File → Save as Google Sheets**).
2. In the **Email** column, replace the placeholders with two addresses you can open (for example your SCIT mail and a personal Gmail). The Gmail account connected to Zapier is the *sender*; use different addresses as recipients so you can reply from them.

### Zap 1
| Step | App / Event | Configuration |
|---|---|---|
| 1 | Schedule by Zapier – Every Day | Time 9:00 AM, timezone Asia/Kolkata, trigger on weekends: No |
| 2 | Google Sheets – Lookup Spreadsheet Rows (Advanced) | Spreadsheet: SCIT_Invoice_Tracker, Worksheet: Invoices, Lookup Column: **Status**, Lookup Value: `Pending`, Row count: 100 |
| 3 | Looping by Zapier – Create Loop From Line Items | Values to loop: Row ID, Invoice ID, Name, Amount, Type, Account No, Bank Name, Email (all from step 2) |
| 4 | Gmail – Send Email | To: *Loop Email*; Subject and Body from `Prompt.md` → Template 1, mapped to loop fields |
| 5 | Formatter by Zapier – Date / Time – Format | Input: `now`, To Format: `DD/MM/YYYY HH:mm`, To Timezone: Asia/Kolkata |
| 6 | Google Sheets – Update Spreadsheet Row | Row: *Loop Row ID*, Reminder Sent On: *output of step 5* |

### Zap 2
| Step | App / Event | Configuration |
|---|---|---|
| 1 | Gmail – New Email Matching Search | Search string: `subject:"Payment Reminder - Invoice" -from:me` |
| 2 | Filter by Zapier | Only continue if *Body Plain* (Text) **contains** `Payment Done` |
| 3 | Formatter by Zapier – Text – Extract Pattern | Input: *Subject*, Pattern: `Invoice\s*(\d+)` → output `0` is the Invoice ID |
| 4 | Google Sheets – Lookup Spreadsheet Row | Lookup Column: **Invoice ID**, Lookup Value: *output of step 3* |
| 5 | Filter by Zapier | Only continue if *Status* (from step 4) **exactly matches** `Pending` (prevents double processing) |
| 6 | Formatter by Zapier – Date / Time – Format | Input: `now`, To Format: `DD/MM/YYYY`, To Timezone: Asia/Kolkata |
| 7 | Google Sheets – Update Spreadsheet Row | Row: *Row ID from step 4*, Status: `Completed`, Payment Confirmed On: *output of step 6* |
| 8 | Gmail – Reply to Email | Thread: *Thread ID from step 1*, Body from `Prompt.md` → Template 2 |

## 6. Testing

1. **Zap 1:** click *Test* on each step (or *Run* the published Zap). Three reminder emails arrive for 1001, 1002 and 1003, and their *Reminder Sent On* cells fill in. 1004 and 1005 receive nothing.
2. **Zap 2:** from the recipient inbox, reply to the reminder for invoice **1001** with `Payment Done`. Within the polling interval, the sheet shows 1001 as **Completed** with today's date, and the recipient gets the acknowledgement reply.
3. **Negative test:** reply to invoice **1002** with `Will pay next week`. The filter stops the Zap, and the status stays Pending.

## 7. Screenshots (in `screenshots/`)

| File | What it shows |
|---|---|
| `01-sheet-before.png` | Invoice sheet before running (3 Pending) |
| `02-zap1-workflow.png` | Zap 1 editor with all 6 steps |
| `03-reminder-email.png` | A reminder email as received |
| `04-sheet-after-zap1.png` | Reminder Sent On filled for the 3 Pending invoices |
| `05-zap2-workflow.png` | Zap 2 editor with all 8 steps |
| `06-payment-done-reply.png` | The "Payment Done" reply in the thread |
| `07-sheet-after-zap2.png` | Invoice 1001 changed to Completed with a date |
| `08-acknowledgement.png` | The automatic acknowledgement reply |
| `09-zap-history.png` | Zap History showing successful runs |

## 8. Outcome

The two Zaps close the loop end to end: pending invoices are identified and chased automatically every day, replies are interpreted without manual reading, the status sheet stays current, and the sender receives confirmation. Every action leaves a timestamp in the sheet, giving the accounts team an audit trail.

## 9. Files

| File | Description |
|---|---|
| `SCIT_Invoice_Tracker.xlsx` | Invoice data used by both Zaps |
| `Prompt.md` | Email templates, filter conditions and extraction pattern |
| `screenshots/` | Proof of execution |
