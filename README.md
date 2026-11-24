# Invoice Follow-Up Bot (n8n)

Loom Link: -Soon-

A daily payment-reminder automation for a small design studio. It sends staged reminders around each invoice's due date, never sends the same stage twice, never chases a paid or disputed invoice, and ends every run with one summary message to the owner.

> Portfolio project built from a fictional client brief. All client names, emails and amounts in the sample data are made up (emails use the reserved `.example` domain).

## The brief

Imran runs a 3-person design studio in Karachi and sends about 40 invoices a month to clients abroad, tracked in a Google Sheet. Roughly a quarter are paid late. He chases irregularly, writes awkward emails, and once chased a client who had already paid. He wants to stop thinking about it and never be embarrassed again.

## What it does

- Runs once a day on a schedule, with nobody touching it.
- Sends reminders in stages: a heads-up before the due date, then Friendly, Firm and Final as the invoice ages.
- Skips Paid and Disputed invoices, always.
- Sends each stage once per invoice, even if the workflow runs twice or is re-run by hand.
- Gives a late-added invoice the stage it has actually reached, not every stage in a row.
- Records every reminder back in the sheet.
- Alerts Imran immediately on a Final notice (so he can call the client) and immediately if a Final notice fails to send.
- Sends Imran one daily summary, including on quiet days, so silence means the workflow is broken.

### Reminder stages

| Stage | Sent when | Tone | Extra |
|---|---|---|---|
| Heads-up | 3 days before the due date | Informational | |
| Friendly | 3 days overdue | Light, assumes it slipped through | |
| Firm | 10 days overdue | Direct, asks for a payment date | |
| Final | 21 days overdue | Professional but clear, asks for payment within 2 working days | Discord alert to Imran to handle it personally |

Thresholds live in one table at the top of the decision code, so changing them is a one-line edit.

## How it works

```
Daily Run (schedule trigger, Asia/Karachi time)
  -> Read invoice sheet (Google Sheets)
  -> Decide action per invoice (Code)
        |-- Needs a reminder? -- true --> Build email (Code) --> Send reminder email (Gmail)
        |                                     |-- success --> Record reminder in sheet
        |                                     |             \-> Final sent? --> Discord alert to Imran
        |                                     \-- error   --> Final failed? --> Discord alert to Imran
        \-- (all rows) --> Build summary (Code) --> Send summary to Imran (Discord)
```

Stack: n8n 2.10.3 (self-hosted in Docker), Google Sheets, Gmail, Discord webhook.

### What the decision code returns

Every row produces exactly one result with a fixed shape (`action`, `reason`, `needsAttention`, plus the invoice fields), so nothing disappears silently.

| `action` | Meaning | What happens |
|---|---|---|
| `send_headsup` / `send_friendly` / `send_firm` / `send_final` | A reminder is due | Email sent, then recorded in the sheet |
| `already_sent` | The stage it has reached was already sent | Nothing |
| `not_due` | More than 3 days before the due date | Nothing |
| `skip_paid` | Status is Paid | Nothing |
| `skip_disputed` | Status is Disputed | Nothing, flagged for Imran |
| `skip_duplicate` | Same invoice number appeared earlier in the sheet | First row wins, duplicate is reported |
| `error` | Unreadable date, amount, status or email | Flagged in the summary, batch continues |

## Design decisions

**Where does the workflow's memory live, and why is a re-run harmless?**
In the sheet's `Last Reminder Stage` and `Last Reminder Date` columns. For each invoice the code works out which stage it *should* be at today (the highest threshold reached) and compares it with the last stage sent. It only sends if the target stage is later than the last one sent. The sheet is updated only after Gmail reports success, so a failed send is retried on the next run and a repeated run sends nothing.

**How are dates compared when the sheet returns strings?**
Dates are converted to whole day numbers (days since 1970) and subtracted, so time zones and daylight saving cannot shift the result. "Today" is the Asia/Karachi calendar day. The parser accepts `2026-11-13`, `4 Nov 2026` and Google Sheets serial numbers, and rejects anything ambiguous (is `04-11-26` 4 Nov or 11 Apr?) instead of guessing, which turns that row into a flagged error.

**How does one bad row or one failed send avoid stopping the rest?**
Every row is processed inside its own `try/catch` in the decision code, so a bad row becomes an `error` item instead of crashing the batch. The Gmail node uses "continue using error output", so a failed send goes down its own path while the others carry on.

**What if a run is missed?**
Stage thresholds use "greater than or equal", not "equal to". If the workflow skips a day, an invoice that jumped from 2 to 4 days overdue still gets its Friendly reminder.

**Messy data.** Status is trimmed and lowercased, amounts like `1,200.50` are parsed, duplicate invoice numbers are collapsed, and an email is only validated when a send would actually happen (a bad email on a not-yet-due invoice is not an error yet).

**What the daily summary reports.** Planned vs actually sent, per-stage counts, total overdue per currency, the invoices needing Imran's attention, and two reconciliation lines (every row counted once, and rows in the sheet equal rows decided) confirming every row was counted in exactly one outcome. If planned and sent differ, a warning with the unconfirmed invoice numbers goes at the top. The summary runs once per day whatever happens, because it hangs off the decision step rather than the send path (it must stay the lowest branch on the canvas, since n8n runs branches top to bottom).

**Which parts use AI?** None, on purpose. Every decision is a deterministic rule on structured data, which makes it testable and repeatable. An LLM would add cost and non-determinism, and client-facing wording is exactly where Imran does not want surprises, so emails come from fixed, reviewed templates.

## Test data

`sample-data/invoice-followup-sample-data-v2.xlsx` holds a 16-row invoice sheet plus the client's notes and my expected results. Due dates are formulas (`=TODAY() - n`), so every scenario stays valid on any day.

| Invoice | Case | Days overdue | Expected action |
|---|---|---|---|
| INV-2041 | Due in 3 days | -3 | `send_headsup` |
| INV-2042 | Due today | 0 | `send_headsup` |
| INV-2043 | 3 days overdue, no history | 3 | `send_friendly` |
| INV-2043 (2nd row) | Duplicate | 3 | `skip_duplicate` |
| INV-2044 | Friendly already sent | 10 | `send_firm` |
| INV-2045 | Firm already sent | 21 | `send_final` + alert |
| INV-2046 | Paid | n/a | `skip_paid` |
| INV-2047 | Disputed | n/a | `skip_disputed` |
| INV-2048 | Missing email | 5 | `error` |
| INV-2049 | Malformed email | 8 | `error` |
| INV-2050 | Amount typed as text `1,200.50` | 11 | `send_firm` |
| INV-2051 | Added late, no history | 12 | `send_firm` only |
| INV-2052 | Friendly sent yesterday | 4 | `already_sent` |
| INV-2053 | Not due for 25 days | -25 | `not_due` |
| INV-2054 | Final already sent | 41 | `already_sent`, flagged |
| INV-2055 | Due date typed as text, status `unpaid ` | 6 | `send_friendly` |

## Test results

Tested 22-23 Nov 2026. During tests, all client emails were redirected to the author's own inbox.

| # | Test | Expected | Result |
|---|---|---|---|
| 1 | Clean run on the 16-row sheet | 8 reminders (2 Heads-up, 2 Friendly, 3 Firm, 1 Final); 2 error rows flagged; 5 invoices need attention; overdue USD 25,660.50 and EUR 2,400.00 | Pass. Summary matched exactly and all 8 emails arrived with the right stage |
| 2 | Double run, same day, no changes | No emails sent | Pass. "Sent: 0 of 0 planned", inbox unchanged |
| 3 | Quiet day (due dates moved far ahead) | A summary still arrives | Pass. Summary sent with 0 planned |
| 4 | One send fails (Final notice given an invalid address) | Other 7 sent, failure alert to Imran, summary warns, sheet not updated for that invoice | Pass. Alert fired and summary read "PLANNED 8, SENT 7, FAILED 1. Not confirmed: INV-2045" |
| 5 | Next run after a failure | Only the failed invoice is retried | Pass. "PLANNED 1", the other 7 stayed "already sent" |
| 6 | Whole workflow crashes (Google Sheets credential broken) | Alert to Imran from a separate error workflow | Pass. Sheet read failed, and the separate error workflow alerted Imran with the node name and error. |
| 7 | Rows read from the sheet equal rows decided | Counts match | Pass (16 / 16) |

## Known limitations and trade-offs

- **Send, then record.** If the sheet update failed after a successful send, the next run could send that stage again. I chose this order because the opposite (record first) can silently skip a reminder, which is worse for the client.
- **Needs n8n to be running.** The schedule only fires while the container and the machine are up. A missed day is not caught up. The container has a restart policy, but a sleeping laptop at run time means no reminders and no summary.
- **Ambiguous dates are rejected, not guessed.** They show up as flagged errors for the owner to fix.
- **Currencies are totalled separately**, with no conversion.
- **Disputed and Final invoices are deliberately left to a human.**

## Setup

1. Create a Google Sheet with the columns: Invoice No, Client Name, Client Email, Amount, Currency, Issue Date, Due Date, Status, Last Reminder Stage, Last Reminder Date.
2. In n8n, import `workflow.json` and connect your own Google Sheets, Gmail and Discord webhook credentials.
3. Point the read and update nodes at your sheet, and set the schedule time and time zone.
4. Optional: edit the `STAGES` table at the top of the decision code to change thresholds, and the templates in `Build email` to change the wording.
5. To test safely, temporarily change the Gmail "To" field to your own address.

## Possible next steps

- Weekly aging report (0-30, 30-60, 60+ days)
- A "mark as paid" link in each email
- Currency conversion for a single overdue total

## Repo contents

- `workflow.json`: the exported n8n workflow
- `sample-data/`: the test sheet and expected results
- `screenshots/`: canvas, summary message, failed-send alert, retry run, quiet-day run
