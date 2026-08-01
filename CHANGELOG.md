# Changelog

All notable changes to this project are documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/).

## [Unreleased]

### Added
- `WF-01 - Alarm Ingestion Webhook` — public webhook entry point (`POST /fiber-alarm-ingest`), lightweight envelope validation, hands off to WF-02.
- `WF-02 - Alarm Normalizer` — multi-vendor detection and field mapping for Huawei-U2000, Nokia-NFM-P, Cisco-EPNM, Juniper-JunosSpace, and FiberHome-iManagerU31 into one standard alarm schema.
- `WF-03 - AI Classification Engine` — Claude-based (Anthropic API) severity/root-cause/priority classification, with deterministic business-rule overrides for 5 alarm types and confidence-based review routing (auto-approved / manual-review / human-validation-required).
- `WF-04 - Site Metadata Enrichment` — Google Sheets site lookup, blended customer-impact calculation, and weighted business-priority scoring (P1–P4).
- `WF-05 - Alarm Correlation & Deduplication Engine` — exact-duplicate detection, 5-minute correlation window grouping by fiber route/site, 24-hour retention pruning, using n8n workflow static data as the store.
- `WF-06 - Intelligent Routing & Decision Engine` — alarm-type-to-team assignment matrix, priority/notification-policy derivation, escalation level (L1/L2/L3) and SLA deadline computation.
- `WF-07 Notification Dispatcher` — resolves destination channel/team and builds the notification payload and message text.
- `WF-08 Telegram Notifier` — resolves the Telegram chat ID and sends the message via the Telegram Bot API, with retry and a structured failure record on delivery failure.
- Full end-to-end `Execute Workflow` chaining from WF-01 through WF-08.
- Project documentation: README, installation guide, architecture reference, API reference, per-workflow reference, and troubleshooting guide.
- Workflow JSON exports (`workflows/`) and canvas/Telegram delivery screenshots (`screenshots/`) added to the repository.
- `demo/README.md` added, pointing to the full pipeline run recording, hosted externally rather than tracked in Git (see below).

### Known issues
- WF-01's webhook has no authentication configured (see [SECURITY.md](SECURITY.md)).
- WF-08's per-team chat-ID table currently resolves every team to the same Telegram chat.
- WF-05's correlation/deduplication store is in-memory and single-instance only.
- The demo video (~128 MB) is intentionally not tracked in Git — it exceeds GitHub's 100 MB per-file push limit. `demo/README.md` currently has a placeholder pending the external hosting link (YouTube/Drive/GitHub Release).
