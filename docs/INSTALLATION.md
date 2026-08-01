# Installation

This project is a set of 8 chained n8n workflows. There is no separate application server to install — you need an n8n instance (self-hosted or n8n Cloud) and three external credentials.

## 1. Prerequisites

- An n8n instance you can import workflows into and create credentials on.
- An Anthropic (Claude) API key.
- A Google account with access to Google Sheets and permission to create an OAuth2 credential (or an existing Google Sheets OAuth2 credential in your n8n instance).
- A Telegram bot token (create one via [@BotFather](https://t.me/BotFather)) and the chat/group IDs of the teams you want to notify.

## 2. Create the required credentials in n8n

Create these three credentials in n8n's **Credentials** section before importing the workflows, so you can attach them to the relevant nodes right after import:

| Credential | Type | Used by | Notes |
|---|---|---|---|
| Anthropic API | Generic **Header Auth** | WF-03, node `4. Call Claude API` | Header name `x-api-key`, value = your Anthropic API key. The node calls `https://api.anthropic.com/v1/messages` with model `claude-sonnet-5`. |
| Google Sheets | **Google Sheets OAuth2 API** | WF-04, node `3. Lookup Site` | Standard Google OAuth2 credential with Sheets access. |
| Telegram Bot | **Telegram API** | WF-08, node `5. Send Telegram Notification` | The bot token from BotFather. |

## 3. Prepare the SiteMetadata Google Sheet

WF-04's `3. Lookup Site` node reads a sheet named **`SiteMetadata`**, looking up rows by a **`Site ID`** column. Create a spreadsheet with at least these columns (WF-04's `4. Merge Metadata` and later nodes read these fields, defaulting gracefully — see [ARCHITECTURE.md](ARCHITECTURE.md) — if a column is missing or a row isn't found):

- `Site ID`
- Site name, division, district, latitude, longitude, node type
- Customer count
- SLA tier (`Gold` / `Silver` / `Bronze`)
- Site owner, engineer name, engineer phone, backup engineer
- Nearest POP, fiber route

Populate it with real site rows before testing WF-04 end-to-end — an unpopulated sheet will cause every lookup to land on WF-04's `Error Output - Site Not Found` path.

## 4. Import the workflows

Import all 8 files from [`workflows/`](../workflows/) into your n8n instance, in this order (each one references the next by workflow ID, so importing WF-08 first avoids any transient "workflow not found" warnings on the Execute Workflow nodes, though n8n will resolve them regardless of import order once all 8 exist):

1. `WF-08 Telegram Notifier.json`
2. `WF-07 Notification Dispatcher.json`
3. `WF-06 - Intelligent Routing & Decision Engine.json`
4. `WF-05 - Alarm Correlation & Deduplication Engine.json`
5. `WF-04 - Site Metadata Enrichment.json`
6. `WF-03 - AI Classification Engine.json`
7. `WF-02 - Alarm Normalizer.json`
8. `WF-01 - Alarm Ingestion Webhook.json`

**Workflows → Import from File** for each, in n8n.

## 5. Attach credentials to imported nodes

After import, open each of these nodes and attach the credential you created in step 2 (a fresh import does not automatically re-link credentials across n8n instances):

- WF-03 → `4. Call Claude API` → Anthropic Header Auth credential
- WF-04 → `3. Lookup Site` → Google Sheets OAuth2 credential, and set `documentId` to your own spreadsheet
- WF-08 → `5. Send Telegram Notification` → Telegram Bot credential

## 6. Configure Telegram chat routing

WF-07's `3. Resolve Notification Destination & Channel` and WF-08's `3. Resolve Telegram Chat ID` both contain a team-to-chat-ID table. As imported, every team currently resolves to the same placeholder chat ID — replace each entry with the real chat/group ID for that team before relying on differentiated routing. See [SECURITY.md](../SECURITY.md) and [TROUBLESHOOTING.md](TROUBLESHOOTING.md).

## 7. Activate the workflows

Toggle **Active** on all 8 imported workflows. Only WF-01 needs to be active for the chain to be externally triggerable — but WF-02 through WF-08 must also be active for `Execute Workflow` calls to reach them, since inactive workflows cannot be invoked this way in n8n.

## 8. Test

Send a test POST request to WF-01's webhook:

```bash
curl -i -X POST https://<your-n8n-host>/webhook/fiber-alarm-ingest \
  -H "Content-Type: application/json" \
  -d '{
    "source": "Huawei-U2000",
    "alarmId": "HW-ALM-TEST-001",
    "neName": "OLT-DHK-001",
    "alarmName": "Fiber Cut",
    "severity": "Critical",
    "occurTime": "2026-08-01 09:12:45"
  }'
```

See [API.md](API.md) for the full request/response contract and [WORKFLOW.md](WORKFLOW.md) for each stage's exact input requirements if you want to test a workflow in isolation instead of the full chain.
