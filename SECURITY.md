# Security Policy

## Reporting a vulnerability

If you discover a security issue in this project (credential exposure, injection risk in a code node, an access-control gap), please report it privately rather than opening a public issue. Open a GitHub issue marked clearly as security-sensitive with minimal detail and request a private channel to share specifics, or contact the repository owner directly.

Please do not include real credentials, tokens, chat IDs, or spreadsheet contents in any report.

## Known current gaps (as of this repository's initial documentation pass)

These are verified facts read directly from the live workflow JSON exports in `workflows/` — not hypothetical risks.

### 1. WF-01's public webhook has no authentication configured

WF-01 (`Alarm Ingestion Webhook`) exposes `POST /webhook/fiber-alarm-ingest`. Its own sticky note describes the endpoint as "Secured with Header Auth token," but the webhook node itself currently has **no `authentication` parameter and no `credentials` block** — meaning the endpoint is reachable by anyone who has the URL, with no token check.

**Recommendation:** add a Header Auth (or equivalent) credential to the webhook node before exposing this endpoint publicly in production. See [docs/TROUBLESHOOTING.md](docs/TROUBLESHOOTING.md) for detail.

### 2. All notification teams currently resolve to one Telegram chat

WF-08's `CHAT_ID_TABLE` (`3. Resolve Telegram Chat ID`) currently maps every team key (`TELEGRAM_CHAT_TRANSMISSION_TEAM`, `_POWER_TEAM`, `_NOC`, etc.) to the same literal chat ID. This is not a credential leak, but it does mean team-based access separation for incident notifications is not currently enforced at delivery time — anyone with access to that one chat sees every team's alerts regardless of routing decision.

**Recommendation:** assign distinct, real chat/group IDs per team before relying on this for access-segmented incident response.

### 3. Credentials referenced by ID only

Workflow JSON exports in this repo reference credentials by n8n-internal ID and display name only (e.g. `"googleSheetsOAuth2Api": {"id": "H7TRzfxIzawI7UIO", "name": "Google Sheets account"}`). These IDs are meaningless outside the originating n8n instance and contain no secret material — but always double-check before committing a fresh export that no credential's actual key/token was pasted into a node's parameters instead of using the Credential Manager.

## Supported versions

This project does not yet use version tags; see [CHANGELOG.md](CHANGELOG.md) for the current state. Security fixes will be applied to the latest state on `main`.
