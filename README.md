# n8n Daily Digest

A collection of n8n workflows that build daily digests by pulling data from multiple sources, merging, cleaning, and returning a single consolidated result.

## Workflows

| File | Description |
|---|---|
| [`workflows/email-digest.json`](workflows/email-digest.json) | Consolidates recent email from multiple Gmail and Outlook accounts into a single deduplicated digest, returned via webhook. |
| [`workflows/calendar-digest.json`](workflows/calendar-digest.json) | Pulls upcoming events from all calendars on a Google account and from two Outlook accounts, merges, flattens and sorts them by start time, and returns them via webhook. |
| [`workflows/render-digest.json`](workflows/render-digest.json) | Receives a structured digest JSON, validates it, renders a self-contained styled HTML document, and delivers it to Telegram as a document attachment. See [`render-digest.md`](workflows/render-digest.md). |

The first two workflows **collect** data; `render-digest` **presents** it. The agent
(OpenClaw) sits between them: it calls the collectors, filters/summarizes, and POSTs a
structured JSON to `render-digest`, which owns rendering and delivery. The JSON schema
and the matching agent prompt are in [`workflows/render-digest.md`](workflows/render-digest.md)
and [`workflows/openclaw-cron-prompt.md`](workflows/openclaw-cron-prompt.md).

## email-digest

### What it does

1. **Webhook trigger** (`GET /webhook/digest`) kicks off the run.
2. In parallel, it pulls messages from the last 24 hours from 8 mailboxes:
   - 2 Outlook accounts (JonDoe@Outlook1, JonDoe@Outlook2)
   - 6 Gmail accounts (JonDoe@Gmail1 through JonDoe@Gmail6)
3. Each branch tags its messages with `account` and `source` (Gmail/Outlook) fields.
4. All branches are merged into a single list.
5. A Code node normalizes the mixed Gmail/Outlook payloads into a common shape (`account`, `source`, `from`, `subject`, `snippet`, `date`), sorts newest-first, and de-duplicates messages that landed in more than one mailbox (matching on sender address, subject, and the first 80 characters of the snippet).
6. The result — a count, dedupe stats, and the cleaned message list — is returned via **Respond to Webhook**.

### Import into n8n

1. Open n8n → **Workflows** → **Import from File**.
2. Select `workflows/email-digest.json`.
3. Reconnect the Gmail / Microsoft Outlook credentials for each account node (credentials are never included in exports).

## calendar-digest

### What it does

1. **Webhook trigger** (`GET /webhook/calendar`) kicks off the run.
2. In parallel, it pulls events from the next 7 days from three sources:
   - **Google**: HTTP Request lists all calendars on the account → Split Out → Get Events fetches each calendar's events (single events expanded, ordered by start time, up to 50 per calendar).
   - **Outlook account 1** (JonDoe@Outlook1) and **Outlook account 2** (JonDoe@Outlook2): each lists its calendars via the Microsoft Outlook node, then fetches events per calendar with recurring instances expanded over the same 7-day window.
3. Each branch tags its events with a `calendarName` and `account` field via a Set node.
4. All three branches are merged (append) into a single list.
5. **Flatten and Sort** (Code node) normalizes the mixed Google/Outlook event shapes into a common shape (`calendar`, `account`, `source`, `title`, `start`, `end`, `allDay`, `location`, `status`, `link`), sorts by start time, and de-duplicates events that landed twice — e.g. when an Outlook calendar is also imported into Google as a subscribed calendar (matching on normalized title and start time, rounded to the minute).
6. The result — a count, dedupe stats, and the sorted event list — is returned via **Respond to Webhook**.

### Import into n8n

1. Open n8n → **Workflows** → **Import from File**.
2. Select `workflows/calendar-digest.json`.
3. Reconnect the Google Calendar OAuth2 credential on the **HTTP Request** and **Get Events** nodes, and the Microsoft Outlook OAuth2 credential on each `Get JonDoe@OutlookN Calendars`/`Events` node pair (credentials are never included in exports).

## Notes

- These exports contain no credentials, tokens, or secrets — only node configuration.
- Mailbox/account labels in the workflows are illustrative; swap in your own accounts as needed.
