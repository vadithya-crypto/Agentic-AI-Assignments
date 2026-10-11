# Assignment 1 – Templates and Rules Used

**LLM used:** None (rule-based Zapier automation). Fields in `{{ }}` are mapped from the Google Sheets / Looping steps.

## Template 1 – Payment Reminder (Zap 1, Gmail – Send Email)

**Subject**
```text
Payment Reminder - Invoice {{Invoice ID}}
```

**Body**
```text
Dear {{Name}},

This is a friendly reminder that the following invoice is pending payment:

  Invoice ID   : {{Invoice ID}}
  Invoice Type : {{Type}}
  Amount       : Rs. {{Amount}}
  Account No   : {{Account No}}
  Bank Name    : {{Bank Name}}

Please complete the payment at the earliest. Once paid, simply reply to this email with "Payment Done" and our records will be updated automatically.

If you have already made the payment, please reply "Payment Done" so we can close the invoice.

Regards,
Accounts Team
```

> Keep "Invoice {{Invoice ID}}" in the subject: Zap 2 reads the Invoice ID back from the subject of the reply ("Re: Payment Reminder - Invoice 1001").

## Template 2 – Acknowledgement (Zap 2, Gmail – Reply to Email)

```text
Dear {{Name}},

Thank you. We have recorded your payment for Invoice {{Invoice ID}} of Rs. {{Amount}}. The invoice status has been updated to Completed on {{Payment Confirmed On}}.

Regards,
Accounts Team
```

## Rules

| Where | Rule | Why |
|---|---|---|
| Zap 1, Lookup | Status = `Pending` | Only chase unpaid invoices |
| Zap 2, Gmail search | `subject:"Payment Reminder - Invoice" -from:me` | Only replies to our reminders, not our own sent mail |
| Zap 2, Filter 1 | Body contains `Payment Done` (case-insensitive) | Ignore replies such as "will pay next week" |
| Zap 2, Extract Pattern | `Invoice\s*(\d+)` on Subject | Pulls `1001` out of `Re: Payment Reminder - Invoice 1001` |
| Zap 2, Filter 2 | Status exactly matches `Pending` | Don't re-process an invoice that is already Completed |
