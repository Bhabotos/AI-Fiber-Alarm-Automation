# Architecture

## Overview

The system is a chain of 8 independent n8n workflows, each with a single responsibility, each invoked by the previous stage via an `Execute Workflow` node. This is confirmed directly in the workflow JSON: every `Execute Workflow` node's target `workflowId` matches the next workflow's own `id`, and every stage but WF-01 is entered exclusively through an `Execute Workflow Trigger` (`inputSource: passthrough`) — there is no other way to invoke WF-02 through WF-08.

```
Alarm Source (5 vendor systems)
      │  POST /fiber-alarm-ingest  →  WF-01 (id y31iGY3hrG7LjdMI)
      ▼
WF-01  Alarm Ingestion Webhook
      │  Execute Workflow → WF-02 (lpJzpskuA7RO06vm)
      ▼
WF-02  Alarm Normalizer
      │  Execute Workflow → WF-03 (rl8C5Lsc7z4fbv5X)
      ▼
WF-03  AI Classification Engine
      │  Execute Workflow → WF-04 (E51WXHZ6qEUC1f56)
      ▼
WF-04  Site Metadata Enrichment
      │  Execute Workflow → WF-05 (PHTsuwVrf4KR5wLd)
      ▼
WF-05  Alarm Correlation & Deduplication Engine
      │  Execute Workflow → WF-06 (T1CGeiqzkL8XDFEr)      [not taken on the "duplicate" outcome — see below]
      ▼
WF-06  Intelligent Routing & Decision Engine
      │  Execute Workflow → WF-07 (YQyl59k4lSA0fTKA)
      ▼
WF-07  Notification Dispatcher
      │  Execute Workflow → WF-08 (SgKj749jIq3b0Q6D)
      ▼
WF-08  Telegram Notifier   [terminal — no outbound Execute Workflow node]
```

All 8 workflows are marked `active: true` in their exports.

**One deliberate branch does not continue the chain:** WF-05's `4a. Duplicate Ignored` outcome has no outgoing connection at all — an exact-duplicate alarm updates the existing incident's counters and stops there. It never reaches WF-06/07/08, so duplicates do not generate additional notifications.

---

## Stage-by-stage architecture

### WF-01 — Alarm Ingestion Webhook
Public entry point: `POST /fiber-alarm-ingest`. Validates only that a `source` field is present — it deliberately does **not** perform vendor-specific validation (that's WF-02's job). Builds a minimal envelope (`correlationId`, `source`, `siteId`, `alarmType`, `rawSeverity`, `receivedAt`, `status`, `rawPayload`) and hands off to WF-02. Invalid payloads are marked `Rejected` and dead-end at a `No-Op` node — there is no persistence/audit log for rejected alarms yet.

### WF-02 — Alarm Normalizer
Detects the sending vendor either from an explicit `source` tag or by matching the payload's field signature against 5 known vendor shapes, then maps that vendor's field names into one standard schema. See [API.md](API.md) for the exact per-vendor field table. Alarm type is derived from free-text keyword matching into 12 standard categories or `OTHER`. Severity is normalized to `Critical`/`Major`/`Minor`/`Warning` (unrecognized → `Warning`, never silently escalated to `Critical`). Timestamps without timezone info are assumed Asia/Dhaka (UTC+6) and converted to UTC.

### WF-03 — AI Classification Engine
Builds a system + user prompt from the normalized alarm and calls Claude (`claude-sonnet-5`) via the Anthropic Messages API, requesting a strict JSON object (`alarmCategory`, `rootCause`, `impact`, `priority`, `recommendedAction`, `estimatedResolution`, `confidence`, `fieldEngineer`, `customerImpact`, `team`, `prevention`).

Five alarm types have **deterministic business-rule overrides** that are applied after the AI call, regardless of what Claude proposed:

| Alarm type | Priority | Team | Field engineer |
|---|---|---|---|
| `FIBER_CUT` | Critical | Transmission | Required |
| `LOS` | Major | NOC | — |
| `HIGH_BER` | Minor | Transmission Monitoring | — |
| `POWER_FAILURE` | Critical | Power | — |
| `MUX_FAILURE` | Critical | Transmission | — |

For every other alarm type, Claude's own judgment is used as-is.

**Confidence-based review routing** (from `7. Compute Confidence Routing`):
- Confidence **> 0.90** → `auto-approved`
- Confidence **0.70 – 0.90** → `manual-review`
- Confidence **< 0.70** → `human-validation-required`

All three routes converge back into the same output shape and continue to WF-04.

**Anthropic error handling:** the HTTP node retries up to 3 times (2s between tries) and routes network/HTTP-level failures to `Error Output - AI Call Failed` via `onError: continueErrorOutput`. Separately, `5. Parse & Validate AI Output` explicitly checks for Anthropic's own top-level API error shape (`{ type: "error", error: {...} }` — e.g. rate-limit, billing, auth errors) and surfaces the real Anthropic error message rather than a generic JSON-parse failure.

### WF-04 — Site Metadata Enrichment
Looks up the alarm's `siteId` in a Google Sheet (`SiteMetadata` tab, `Site ID` column) with retry (3 tries, 2s apart) and a dedicated `Error Output - Site Not Found` path when no match exists.

**Customer impact** blends the site's actual customer count with the AI's own judgment, taking whichever is more severe:
- Base impact from customer count: **> 1000 → High**, **200–1000 → Medium**, **< 200 → Low**
- SLA tier nudges the result: **Gold** raises impact one level (capped at High), **Bronze** lowers it one level (floored at Low), **Silver** is neutral
- Final impact = the more severe of this computed value vs. WF-03's `customerImpact` — metadata can only escalate, never downgrade, the AI's assessment

**Business priority** is a weighted score: `priority weight (Critical 4 / Major 3 / Minor 2 / Warning 1) + impact weight (High 3 / Medium 2 / Low 1) + SLA weight (Gold 2 / Silver 1 / Bronze 0)`.

| Score | Business priority |
|---|---|
| ≥ 8 | P1 - Urgent |
| 6 – 7 | P2 - High |
| 4 – 5 | P3 - Normal |
| ≤ 3 | P4 - Low |

### WF-05 — Alarm Correlation & Deduplication Engine
Stateful — uses n8n's built-in workflow static data (`$getWorkflowStaticData('global')`) as its store. **This is in-memory and single-instance only**; it is not shared across multiple n8n worker processes.

Rules, in order:
1. **Exact duplicate** — same `correlationId` seen before → `duplicate_ignored`, increments `duplicateCount` on the existing incident, does not forward downstream.
2. **Merge** — grouping key is `route:{fiberRoute}` if present, else `site:{siteId}`. If an open incident for that key had its last alarm within a **5-minute window**, the new alarm merges in as a child (`merged_child`).
3. **New incident** — otherwise, a new master incident is created (`new_master_incident`).

A retention pass runs on every execution, pruning incidents (and their index entries) older than **24 hours**.

### WF-06 — Intelligent Routing & Decision Engine
Pure business-rule engine, no external dependencies.

**Team assignment** (`ALARM_TYPE_TEAM_MAP`):

| Alarm type(s) | Team |
|---|---|
| `FIBER_CUT`, `OTN_PORT_DOWN`, `DWDM_CARD_FAILURE`, `MUX_FAILURE` | Transmission Team |
| `LOS`, `LOF` | NOC |
| `HIGH_BER`, `HIGH_ATTENUATION` | Transmission Monitoring |
| `POWER_FAILURE` | Power Team |
| `SITE_DOWN`, `TEMPERATURE_ALARM` | Site Maintenance Team |
| `LINK_DOWN` | IP Network Team |
| `OTHER` / unmapped | NOC (default) |

**Priority** uses the root alarm's own priority (set in WF-03) as the primary signal, falling back to a value derived from WF-04's `businessPriority` (P1→Critical, P2→Major, P3→Minor, else Warning) only if missing.

**Notification policy**: Critical → `immediate`, Major → `urgent`, Minor → `standard`, Warning → `digest`.

**Escalation & SLA**:
- Escalation level: **L1** if priority is Critical *or* the incident is an "alarm storm" (`occurrenceCount > 3` correlated child alarms); **L2** for Major; **L3** otherwise. `immediateEscalation` is true only at L1.
- SLA response window by priority: **Critical 30 min, Major 120 min, Minor 480 min, Warning 1440 min** — `slaDeadline` is `now + window`.

### WF-07 — Notification Dispatcher
Resolves the assigned team to a channel/destination key and builds the final message title/body and metadata. It does **not** send anything — that's WF-08's job. (Per its own sticky note: delivery was intentionally left out of this workflow.)

### WF-08 — Telegram Notifier
Resolves the destination key to an actual Telegram chat ID and sends the message via the Telegram node, with retry (3 tries, 1s apart) and a structured failure record (`Error Output - Delivery Failed`) if delivery fails after retries.

---

## Technology stack

| Layer | Technology |
|---|---|
| Automation engine | n8n |
| AI classification | Claude (`claude-sonnet-5`) via the Anthropic Messages API |
| Site metadata store | Google Sheets |
| Correlation/dedup store | n8n workflow static data (in-memory, single-instance) |
| Notification channel | Telegram Bot API |

## Design principles (as implemented)

- **Single responsibility per workflow** — confirmed by each workflow's own sticky-note description and by the fact that every stage after WF-01 only accepts input via `Execute Workflow Trigger`.
- **`correlationId` as the thread** — generated at WF-01/WF-02 and carried through every downstream stage's output.
- **Fail closed on missing required input** — every workflow's `2. Validate Input` node rejects to its own `Error Output - Invalid Input` node rather than proceeding with partial data (exact required fields per workflow are in [API.md](API.md)).
- **Escalate, never silently downgrade** — WF-04's customer-impact logic explicitly takes the more severe of the computed and AI-assessed values.
