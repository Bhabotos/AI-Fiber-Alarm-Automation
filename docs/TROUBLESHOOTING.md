# Troubleshooting

Verified, current issues found by reading the live workflow JSON exports directly — not hypothetical. Each entry names the exact node(s) involved so you can go find them on the canvas.

---

## 1. The ingestion webhook accepts requests with no authentication

**Symptom:** any client that knows the webhook URL can POST alarms, with no token check.

**Cause:** WF-01's `1. Alarm Ingestion Webhook` node has no `authentication` parameter and no attached credential, despite the workflow's own sticky note describing it as "Secured with Header Auth token."

**Fix:** open the node, set **Authentication → Header Auth**, and attach (or create) a Header Auth credential with a secret token. Update any calling systems to send that header. See [SECURITY.md](../SECURITY.md).

---

## 2. All teams' Telegram notifications land in the same chat

**Symptom:** alerts routed to different teams (Transmission, Power, NOC, etc.) all arrive in one Telegram chat, regardless of `assignedTeam`.

**Cause:** WF-08's `3. Resolve Telegram Chat ID` node's `CHAT_ID_TABLE` currently maps every team key to the same literal chat ID.

**Fix:** open `3. Resolve Telegram Chat ID` and replace each team's entry with its real chat/group ID. Also check WF-07's `3. Resolve Notification Destination & Channel`, which independently maps team → chat-ID *key* (not the ID itself) — the two tables must stay in sync.

---

## 3. Duplicate alarms after a service restart / clustered n8n deployment

**Symptom:** the same alarm gets treated as a new incident instead of being deduplicated, especially after restarting n8n or when running multiple n8n workers.

**Cause:** WF-05's correlation/dedup store is `$getWorkflowStaticData('global')` — in-memory, per-instance data. It is not shared across worker processes and does not survive certain restart scenarios in clustered n8n deployments.

**Fix:** this is a known architectural limitation, not a bug to patch in place. If you need multi-worker correlation, the store would need to move to an external system (e.g. Redis or Postgres) — that is a design change to WF-05's `3. Correlation & Deduplication Engine` node, not a configuration fix.

---

## 4. An alarm doesn't merge into an existing incident even though it looks related

**Symptom:** two alarms from the same fiber route/site create two separate incidents instead of one.

**Cause:** WF-05 only merges alarms into an existing incident if the prior alarm on that `route:{fiberRoute}` (or `site:{siteId}`) key arrived within the last **5 minutes** (`WINDOW_MS`). Anything older is treated as a fresh incident. Also check whether `fiberRoute` is actually populated on both alarms — if one has it and the other doesn't, they'll group on different keys (`route:...` vs `site:...`).

**Fix:** confirm both alarms carry the same `fiberRoute` (from WF-04's site lookup) and arrived within 5 minutes of each other. If your operational need is a longer window, that constant lives in WF-05's `3. Correlation & Deduplication Engine` node.

---

## 5. WF-03 (AI Classification) execution fails outright instead of routing to an error path

**Symptom:** the whole WF-03 execution fails (n8n shows a failed execution), rather than landing on `Error Output - AI Call Failed` or `Error Output - Invalid AI Response`.

**Likely cause:** `4. Call Claude API` has `retryOnFail`/`onError: continueErrorOutput` configured and is wired to `Error Output - AI Call Failed`, and `5. Parse & Validate AI Output` explicitly handles Anthropic's own top-level error response shape — so most failure modes (timeouts after retries, rate limits, auth errors, malformed AI output) should route to an error node, not crash the execution. If you're seeing a hard failure instead, check:
- Whether the Anthropic Header Auth credential is actually attached and valid (an unattached credential fails before the HTTP node's own error handling applies).
- Whether the failure is happening in a node *other* than `4. Call Claude API` (e.g. a code node throwing on unexpected input shape from WF-02).

---

## 6. An alarm is rejected at WF-04 with "Site Not Found"

**Symptom:** alarms for a real site fail enrichment.

**Cause:** WF-04's `3. Lookup Site` looks up `siteId` against the **`Site ID`** column of a Google Sheet tab named **`SiteMetadata`**. If that sheet/tab doesn't exist, isn't shared with the OAuth2 credential's account, or simply doesn't have a row for that site, the lookup returns no match and the alarm routes to `Error Output - Site Not Found`.

**Fix:** confirm the sheet ID configured on `3. Lookup Site` points to a real, populated spreadsheet (see [INSTALLATION.md](INSTALLATION.md)), and that the site actually has a row keyed by its exact `Site ID` value.

---

## 7. A rejected alarm at WF-01 seems to vanish

**Symptom:** an alarm that fails WF-01's validation doesn't show up anywhere afterward.

**Cause:** WF-01's rejected path (`4b. Mark Alarm Rejected` → `5b. Rejected - Awaiting Audit Logger`) is a genuine dead end in the current implementation — there is no audit-log workflow wired up yet to persist rejected alarms.

**Fix:** this is expected with the current implementation, not a malfunction. If you need visibility into rejected alarms, you'll need to add persistence (e.g. a database write or log export) after `5b`.
