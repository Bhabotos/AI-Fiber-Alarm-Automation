# Workflow Reference

Detailed, node-by-node reference for each of the 8 workflows in `workflows/`. Input/output schemas are in [API.md](API.md); business-rule detail (scoring formulas, thresholds) is in [ARCHITECTURE.md](ARCHITECTURE.md). This document focuses on structure: nodes, triggers, error paths, and retry configuration, read directly from each workflow's JSON export.

---

## WF-01 — Alarm Ingestion Webhook

- **id:** `y31iGY3hrG7LjdMI` · **active:** true
- **Trigger:** `n8n-nodes-base.webhook`, `POST /fiber-alarm-ingest`. No authentication configured (see [SECURITY.md](../SECURITY.md)).

**Nodes:**
1. `1. Alarm Ingestion Webhook` (webhook)
2. `2. Extract & Validate Payload` (code) — checks only that `source` is present
3. `3. Is Payload Valid?` (if)
4. `4a. Build Standard Alarm Envelope` (set) → `5a. Ready for WF-02 Normalizer` (no-op) → `6. Execute WF-02 Normalizer` (executeWorkflow → WF-02)
5. `4b. Mark Alarm Rejected` (set) → `5b. Rejected - Awaiting Audit Logger` (no-op) — **dead end**, not persisted anywhere yet

**Error handling:** no `retryOnFail` on any node; no dedicated `Error Output` node — rejection is a reachable but terminal branch.

---

## WF-02 — Alarm Normalizer

- **id:** `lpJzpskuA7RO06vm` · **active:** true
- **Trigger:** `Execute Workflow Trigger` (`inputSource: passthrough`) — invoked only by WF-01.

**Nodes:**
1. `1. Execute Workflow Trigger`
2. `2. Detect Vendor & Validate` (code) — vendor detection + per-vendor required-field check
3. `Is Valid?` (if)
4. `3. Normalize Fields` (code, **`retryOnFail: true, maxTries: 2`**) — vendor field mapping, alarm-type keyword detection, severity normalization, UTC timestamp conversion
5. `4. Ensure Correlation ID` (code)
6. `5. Return Standardized JSON` (no-op) → `6. Execute WF-03 Classification` (executeWorkflow → WF-03)
7. `Error Output` (set) — reachable from `Is Valid?` false branch

---

## WF-03 — AI Classification Engine

- **id:** `rl8C5Lsc7z4fbv5X` · **active:** true
- **Trigger:** `Execute Workflow Trigger` — invoked only by WF-02.

**Nodes:**
1. `1. Execute Workflow Trigger`
2. `2. Validate Input` (code) → `Is Input Valid?` (if) → `Error Output - Invalid Input` (set)
3. `3. Build Claude Prompt` (code) — assembles system + user prompt with the business-rule guardrails and strict JSON output contract
4. `4. Call Claude API` (httpRequest, **`retryOnFail: true, maxTries: 3, waitBetweenTries: 2000`**, **`onError: continueErrorOutput`**) → on failure, `Error Output - AI Call Failed` (set)
5. `5. Parse & Validate AI Output` (code) — strips markdown fencing, parses JSON, explicitly detects Anthropic's own top-level `{type: "error", error: {...}}` response shape (surfaces the real Anthropic error message rather than a generic parse failure)
6. `Is AI Output Valid?` (if) → `Error Output - Invalid AI Response` (set)
7. `6. Apply Business Rules` (code) — enforces the 5 deterministic overrides (see ARCHITECTURE.md)
8. `7. Compute Confidence Routing` (code)
9. `Confidence > 90%?` (if) → `8a. Auto-Approved` (no-op)
10. `Confidence 70-90%?` (if) → `8b. Manual Review` (no-op) / `8c. Human Validation Required` (no-op)
11. All three of `8a`/`8b`/`8c` → `9. Execute WF-04 Enrichment` (executeWorkflow → WF-04)

**Error handling:** three distinct error outputs — invalid input, AI call failure, invalid AI response — all reachable, none orphaned.

---

## WF-04 — Site Metadata Enrichment

- **id:** `E51WXHZ6qEUC1f56` · **active:** true
- **Trigger:** `Execute Workflow Trigger` — invoked only by WF-03.

**Nodes:**
1. `1. Execute Workflow Trigger`
2. `2. Validate Input` (code) → `Is Input Valid?` (if) → `Error Output - Invalid Input` (set)
3. `3. Lookup Site` (Google Sheets, **`retryOnFail: true, maxTries: 3, waitBetweenTries: 2000`**, `alwaysOutputData: true`, **`onError: continueErrorOutput`**) → on failure, `Error Output - Lookup Failed` (set)
4. `4. Merge Metadata` (code)
5. `Is Site Found?` (if) → `Error Output - Site Not Found` (set)
6. `5. Calculate Customer Impact` (code)
7. `6. Calculate Business Priority` (code)
8. `7. Return Enriched Alarm` (no-op) → `8. Execute WF-05 Correlation` (executeWorkflow → WF-05)

**Note:** `alwaysOutputData: true` on `3. Lookup Site` is required for the "not found" path to be reachable at all — a Google Sheets lookup that returns zero rows would otherwise not produce output for `4. Merge Metadata` to inspect.

---

## WF-05 — Alarm Correlation & Deduplication Engine

- **id:** `PHTsuwVrf4KR5wLd` · **active:** true
- **Trigger:** `Execute Workflow Trigger` — invoked only by WF-04.

**Nodes:**
1. `1. Execute Workflow Trigger`
2. `2. Validate Input` (code) → `Is Input Valid?` (if) → `Error Output - Invalid Input` (set)
3. `3. Correlation & Deduplication Engine` (code, **`retryOnFail: true, maxTries: 2, waitBetweenTries: 500`**) — stateful, uses `$getWorkflowStaticData('global')`
4. `Is Duplicate?` (if) → `4a. Duplicate Ignored` (no-op) — **no outgoing connection; chain stops here for duplicates**
5. `Is Merged Child?` (if) → `4b. Merged - Child Alarm Added` (no-op) / `4c. New Master Incident Created` (no-op)
6. Both `4b` and `4c` → `5. Execute WF-06 Routing` (executeWorkflow → WF-06)

---

## WF-06 — Intelligent Routing & Decision Engine

- **id:** `T1CGeiqzkL8XDFEr` · **active:** true
- **Trigger:** `Execute Workflow Trigger` — invoked only by WF-05.

**Nodes:**
1. `1. Execute Workflow Trigger`
2. `2. Validate Input` (code) → `Is Input Valid?` (if) → `Error Output - Invalid Input` (set)
3. `3. Determine Team Assignment` (code) — `ALARM_TYPE_TEAM_MAP`
4. `4. Determine Priority & Notification Policy` (code)
5. `5. Compute Escalation & SLA` (code)
6. `6. Build Routing Decision` (code)
7. `7. Return Routing Decision` (no-op) → `8. Execute WF-07 Notification Dispatcher` (executeWorkflow → WF-07)

**Error handling:** no `retryOnFail` anywhere in this workflow — it's pure code-node business logic with no external calls.

---

## WF-07 — Notification Dispatcher

- **id:** `YQyl59k4lSA0fTKA` · **active:** true
- **Trigger:** `Execute Workflow Trigger` — invoked only by WF-06.

**Nodes:**
1. `1. Execute Workflow Trigger`
2. `2. Validate Input` (code) → `Is Input Valid?` (if) → `Error Output - Invalid Input` (set)
3. `3. Resolve Notification Destination & Channel` (code, **`retryOnFail: true, maxTries: 2, waitBetweenTries: 500`**) — team-to-chat-ID-key lookup via `TEAM_CHAT_MAP`
4. `4. Build Notification Payload` (code, **`retryOnFail: true, maxTries: 2, waitBetweenTries: 500`**)
5. `5. Return Notification Payload` (no-op) → `6. Execute WF-08 Telegram Notifier` (executeWorkflow → WF-08)

This workflow prepares and routes a payload; it does **not** call the Telegram API itself.

---

## WF-08 — Telegram Notifier

- **id:** `SgKj749jIq3b0Q6D` · **active:** true
- **Trigger:** `Execute Workflow Trigger` — invoked only by WF-07. **Terminal workflow — contains no outbound `Execute Workflow` node.**

**Nodes:**
1. `1. Execute Workflow Trigger`
2. `2. Validate Input` (code) → `Is Input Valid?` (if) → `Error Output - Invalid Input` (set)
3. `3. Resolve Telegram Chat ID` (code, **`retryOnFail: true, maxTries: 2, waitBetweenTries: 500`**) — `CHAT_ID_TABLE` lookup (see [SECURITY.md](../SECURITY.md) — all teams currently resolve to the same chat ID)
4. `4. Format Telegram Message` (code, **`retryOnFail: true, maxTries: 2, waitBetweenTries: 500`**)
5. `5. Send Telegram Notification` (Telegram node, **`retryOnFail: true, maxTries: 3, waitBetweenTries: 1000`**, **`onError: continueErrorOutput`**)
6. On success → `6. Return Dispatch Result` (set)
7. On failure → `7. Build Failure Record` (code) → `8. Log Failure` (no-op) → `Error Output - Delivery Failed` (set)

---

## Testing each workflow in isolation

Every workflow except WF-01 can be tested independently in n8n via its `Execute Workflow Trigger`'s "Test Workflow" / pinned test-data feature, using the minimum required fields listed in [API.md](API.md) for that stage. WF-01 is tested via its webhook (see [INSTALLATION.md](INSTALLATION.md)).

Recommended order for a full manual run-through: WF-01 → WF-02 → WF-03 → WF-04 → WF-05 → WF-06 → WF-07 → WF-08, following one alarm's `correlationId` through each stage's execution log.
