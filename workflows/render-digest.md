# render-digest

Turns a **structured digest JSON** into the finished HTML and delivers it. This is
the "machinery" half of the digest: OpenClaw decides *what* goes in the digest and
emits JSON; this workflow decides *how it looks* and sends it. The layout is fixed
in code, so the digest looks identical every day and only the data changes.

## Flow

```
POST /webhook/render-digest (JSON)
  -> Validate      (Code)  : checks shape/enums, outputs { valid, errors, digest }
  -> Valid?        (IF)
       true  -> Render HTML (Code) -> Send to Telegram (sendDocument) -> Respond OK (200)
       false -> Respond Invalid (400 + the list of errors)
```

- **Validation is the backstop.** A malformed payload gets a `400` with a precise
  error list instead of a broken document. OpenClaw is told to fix and resend once.
- **Escaping happens in the Render node, in code** — every content field is
  HTML-escaped before insertion, so it's a guarantee, not something the model has to
  remember.
- **Delivery is in n8n**, via the Telegram node's `sendDocument`. No base64 / data-URL
  juggling by the agent — n8n sends the binary `.html` natively.

## Import

1. n8n -> Workflows -> Import from File -> `workflows/render-digest.json`.
2. Attach a **Telegram** credential to the *Send to Telegram* node, and set the
   `chatId` field on that node to your own Telegram chat ID.
3. Activate. The endpoint is `POST http://localhost:5678/webhook/render-digest`.

## JSON contract

`severity` is one of: **`urgent` | `attention` | `info` | `done`** (maps to chip color).
Always send every key; use `[]` for empty arrays so the quiet empty-states render.

| Path | Type | Notes |
|---|---|---|
| `date` | string | `YYYY-MM-DD`. Drives title + filename. |
| `weekday` | string (optional) | Pretty header line, e.g. `Wednesday, June 24, 2026`. Falls back to `date`. |
| `metrics.todayEvents` | number | Top summary cards. |
| `metrics.overdueTasks` | number | |
| `metrics.priorityEmails` | number | |
| `metrics.upcomingP1P2` | number | |
| `schedule.allDay[]` | `{ title, meta? }` | All-day events. |
| `schedule.timed[]` | `{ time, title, meta? }` | `time` shown as the chip. |
| `deadlines[]` | `{ severity, chip, title, meta?, snippet? }` | |
| `emails[]` | `{ severity, chip, title, meta?, snippet? }` | |
| `fyi[]` | `{ severity, chip, title, meta?, snippet? }` | |
| `todoist.dueToday[]` | string[] | One line per task. |
| `todoist.overdue[]` | string[] | |
| `todoist.upcoming[]` | string[] | |
| `actions[]` | string[] | Suggested Actions bullets. |

## Filled example (the gold-standard payload)

```json
{
  "date": "2026-06-24",
  "weekday": "Wednesday, June 24, 2026",
  "metrics": { "todayEvents": 1, "overdueTasks": 10, "priorityEmails": 9, "upcomingP1P2": 1 },
  "schedule": {
    "allDay": [],
    "timed": [
      { "time": "10:00 AM-11:00 AM ET", "title": "Weekly Team Sync",
        "meta": "Calendar: Work - Outlook · Location: meeting link" }
    ]
  },
  "deadlines": [
    { "severity": "urgent", "chip": "Due today", "title": "Auto Insurance payment due today",
      "meta": "From billing@example-insurance.com · Project A · 12:38 PM ET",
      "snippet": "Payment reminder says the payment is due today." },
    { "severity": "urgent", "chip": "Final notice", "title": "Utility Co: Final Notice to Avoid Disconnection",
      "meta": "From notifications@example-utility.com · Project B · 10:27 AM ET",
      "snippet": "Requests immediate action on a past-due balance to avoid disconnection." },
    { "severity": "attention", "chip": "Next 14 days", "title": "Todoist: Do Continuing Education",
      "meta": "P1 · Deadline July 3, 2026" }
  ],
  "emails": [
    { "severity": "urgent", "chip": "Security", "title": "Security alert for n8n access",
      "meta": "From no-reply@example.com · Project A · Jun 23, 5:19 PM ET",
      "snippet": "Alert says n8n was allowed access to some account data. Review if unexpected." }
  ],
  "todoist": {
    "dueToday": [],
    "overdue": ["P1: Plugging in missing Due Date for AI", "P2: Finalize VIP Boxes", "Apply & get the EIN"],
    "upcoming": ["P1, deadline Jul 3: Do Continuing Education"]
  },
  "fyi": [
    { "severity": "done", "chip": "Receipt", "title": "Carrier payment completed",
      "meta": "From Example Carrier · Project C · 12:24 PM ET", "snippet": "Completed payment of $390.86." }
  ],
  "actions": [
    "Pay or verify the insurance and utility balances first; the utility notice is the clearest service-risk item.",
    "Confirm the n8n access alert was expected."
  ]
}
```

## Error response shape (HTTP 400)

```json
{ "valid": false, "errors": [
  "metrics.overdueTasks must be a number",
  "deadlines[0].severity must be one of urgent|attention|info|done"
] }
```
