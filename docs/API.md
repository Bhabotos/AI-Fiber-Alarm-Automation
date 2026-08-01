# API Reference

This project exposes exactly **one** public HTTP endpoint (WF-01's webhook). Every other "API" in this system is an internal contract between two chained n8n sub-workflows, enforced by each workflow's own `Validate Input` node. Both are documented here.

---

## Public endpoint

### `POST /webhook/fiber-alarm-ingest`

Implemented by WF-01, node `1. Alarm Ingestion Webhook`.

> **Note:** as currently configured, this endpoint has **no authentication** — see [SECURITY.md](../SECURITY.md). Despite the workflow's own sticky note describing it as "secured with Header Auth token," the live node has no `authentication` parameter or credential attached.

**Required field:** only `source` (string). WF-01 does not validate vendor-specific fields itself — that happens in WF-02, one step later. If `source` is missing, WF-01 responds with a rejection; if `source` is present but the vendor is unrecognized or that vendor's required fields are missing, the rejection happens downstream in WF-02 rather than at this endpoint (WF-02's output does not currently propagate a response back to the original caller — see [TROUBLESHOOTING.md](TROUBLESHOOTING.md)).

**Example request — Huawei alarm:**
```bash
curl -i -X POST https://<your-n8n-host>/webhook/fiber-alarm-ingest \
  -H "Content-Type: application/json" \
  -d '{
    "source": "Huawei-U2000",
    "alarmId": "HW-ALM-20260731-001",
    "neName": "OLT-DHK-001",
    "alarmName": "Fiber Cut",
    "severity": "Critical",
    "occurTime": "2026-07-31 09:12:45"
  }'
```

**WF-01 output envelope** (built by `4a. Build Standard Alarm Envelope`, passed into WF-02):
```json
{
  "correlationId": "string (UUID)",
  "source": "string",
  "siteId": "string",
  "alarmType": "string",
  "rawSeverity": "string",
  "receivedAt": "string (ISO-8601 UTC)",
  "status": "New",
  "rawPayload": { "...": "original body" }
}
```

**Rejected envelope** (when `source` is missing — `4b. Mark Alarm Rejected`):
```json
{
  "correlationId": "string (UUID)",
  "receivedAt": "string (ISO-8601 UTC)",
  "status": "Rejected",
  "rejectionReason": ["array of strings"],
  "rawPayload": { "...": "original body" }
}
```

---

## Internal contracts (Execute Workflow chain)

Each of the following is enforced by the receiving workflow's own `2. Validate Input` (or `2. Detect Vendor & Validate`) node. A payload missing any required field is rejected to that workflow's own `Error Output - Invalid Input` node rather than proceeding.

### WF-01 → WF-02 (Alarm Normalizer)

**WF-02 required input:** vendor detected from `source`, or from a structural field signature; then that vendor's own required raw fields:

| Vendor (`source` value) | Required raw fields |
|---|---|
| `Huawei-U2000` | `neName`, `alarmName`, `severity`, `occurTime` |
| `Nokia-NFM-P` | `objectName`, `problemText`, `perceivedSeverity`, `eventTime` |
| `Cisco-EPNM` | `managedElement`, `faultDescription`, `faultSeverity`, `timeStamp` |
| `Juniper-JunosSpace` | `device-name`, `alarm-description`, `alarm-severity`, `alarm-time` |
| `FiberHome-iManagerU31` | `NEName`, `faultName`, `faultLevel`, `happenTime` |

**WF-02 output** (`5. Return Standardized JSON`):
```json
{
  "correlationId": "string",
  "vendor": "string",
  "alarmId": "string | null",
  "nodeName": "string",
  "siteId": "string",
  "alarmName": "string",
  "alarmType": "one of 12 standard types, or OTHER",
  "severity": "Critical | Major | Minor | Warning",
  "timestamp": "string (ISO-8601 UTC)",
  "region": "string | null",
  "status": "New",
  "rawPayload": { "...": "..." }
}
```

**WF-02 invalid output** (`Error Output`):
```json
{ "status": "invalid", "vendor": "string | null", "errors": ["..."], "rawPayload": {}, "receivedAt": "string" }
```

### WF-02 → WF-03 (AI Classification Engine)

**Required:** `correlationId`, `vendor`, `alarmType`, `severity`, `nodeName`, `siteId`, `region`, `timestamp`.

**Output** (all three confidence-routing branches share this shape):
```json
{
  "correlationId": "string", "vendor": "string", "alarmType": "string",
  "originalSeverity": "string", "nodeName": "string", "siteId": "string",
  "region": "string", "timestamp": "string",
  "businessRuleApplied": "boolean",
  "alarmCategory": "string", "rootCause": "string", "impact": "string",
  "priority": "Critical | Major | Minor | Warning",
  "recommendedAction": "string", "estimatedResolution": "string",
  "confidence": "number (0-1)", "fieldEngineer": "boolean",
  "customerImpact": "High | Medium | Low", "team": "string",
  "prevention": "string",
  "reviewStatus": "auto-approved | manual-review | human-validation-required",
  "aiModel": "claude-sonnet-5", "processedAt": "string (ISO-8601)"
}
```

### WF-03 → WF-04 (Site Metadata Enrichment)

**Required:** `correlationId`, `siteId`, `alarmType`, `priority`.

**Output** (`7. Return Enriched Alarm`) — all WF-03 fields plus:
```json
{
  "isSiteFound": "boolean", "siteName": "string", "division": "string", "district": "string",
  "latitude": "number", "longitude": "number", "nodeType": "string", "customerCount": "number",
  "slaTier": "Gold | Silver | Bronze", "siteOwner": "string", "transmissionRegion": "string",
  "engineerName": "string", "engineerPhone": "string", "backupEngineer": "string",
  "nearestPOP": "string", "fiberRoute": "string",
  "computedCustomerImpact": "High | Medium | Low", "customerImpact": "High | Medium | Low",
  "customerImpactScore": "number", "businessPriorityScore": "number",
  "businessPriority": "P1 - Urgent | P2 - High | P3 - Normal | P4 - Low",
  "enrichedAt": "string (ISO-8601)"
}
```

### WF-04 → WF-05 (Alarm Correlation & Deduplication Engine)

**Required:** `correlationId`, `siteId`, `nodeName`, `alarmType`, `timestamp`, `region`, `vendor`.

**Output** (all 3 outcomes share this shape):
```json
{
  "outcome": "duplicate_ignored | merged_child | new_master_incident",
  "incidentId": "string",
  "rootAlarm": { "correlationId": "string", "alarmType": "string", "nodeName": "string", "priority": "string", "timestamp": "string" },
  "childAlarms": [ { "correlationId": "string", "alarmType": "string", "nodeName": "string", "priority": "string", "timestamp": "string" } ],
  "occurrenceCount": "number", "duplicateCount": "number",
  "impact": "string", "affectedCustomers": "number", "businessPriority": "string",
  "siteId": "string", "fiberRoute": "string", "vendor": "string", "region": "string",
  "createdAt": "string (ISO-8601)", "lastAlarmAt": "string (ISO-8601)"
}
```

> `duplicate_ignored` outcomes are not forwarded to WF-06 (see [ARCHITECTURE.md](ARCHITECTURE.md)).

### WF-05 → WF-06 (Intelligent Routing & Decision Engine)

**Required:** `incidentId`, `siteId`, `vendor`, `region`, plus a nested `rootAlarm.alarmType`.

**Output** (`7. Return Routing Decision`):
```json
{
  "incidentId": "string", "correlationId": "string | null", "siteId": "string",
  "assignedTeam": "string", "priority": "Critical | Major | Minor | Warning",
  "notificationPolicy": "immediate | urgent | standard | digest",
  "escalationLevel": "L1 | L2 | L3", "immediateEscalation": "boolean",
  "slaDeadline": "string (ISO-8601)", "businessImpact": "string",
  "affectedCustomers": "number", "vendor": "string", "region": "string",
  "routedAt": "string (ISO-8601)", "nextWorkflow": "notification-dispatcher"
}
```

### WF-06 → WF-07 (Notification Dispatcher)

**Required:** `incidentId`, `siteId`, `assignedTeam`, `priority`, `notificationPolicy`.

**Output** (`4. Build Notification Payload`):
```json
{
  "incidentId": "string", "correlationId": "string", "siteId": "string",
  "channel": "telegram",
  "destination": { "team": "string", "chatId": "string", "destinationMatched": "boolean" },
  "priority": "string", "notificationPolicy": "string",
  "escalationLevel": "string", "immediateEscalation": "boolean",
  "messageTitle": "string", "messageBody": "string", "slaDeadline": "string",
  "businessImpact": "string", "affectedCustomers": "number",
  "dispatchStatus": "ready_for_dispatch", "preparedAt": "string",
  "nextWorkflow": "telegram-notifier"
}
```

### WF-07 → WF-08 (Telegram Notifier)

**Required:** `incidentId`, `destination.chatId`, `destination.team`, `priority`.

**Output — success** (`6. Return Dispatch Result`):
```json
{ "status": "sent", "incidentId": "string", "chatId": "string", "assignedTeam": "string", "telegramMessageId": "string", "sentAt": "string" }
```

**Output — delivery failure** (`Error Output - Delivery Failed`):
```json
{ "status": "failed", "incidentId": "string", "chatId": "string", "assignedTeam": "string", "errorMessage": "string", "attemptsMade": "number", "failedAt": "string" }
```

**Output — invalid input** (`Error Output - Invalid Input`):
```json
{ "status": "invalid_input", "incidentId": "string", "missingFields": ["..."], "receivedAt": "string" }
```

---

## External API calls made by this pipeline

| Workflow | Node | External API | Purpose |
|---|---|---|---|
| WF-03 | `4. Call Claude API` | `POST https://api.anthropic.com/v1/messages` (model `claude-sonnet-5`) | Alarm classification |
| WF-04 | `3. Lookup Site` | Google Sheets API | Site metadata lookup |
| WF-08 | `5. Send Telegram Notification` | Telegram Bot API | Message delivery |
