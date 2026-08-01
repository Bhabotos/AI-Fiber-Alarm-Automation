# Contributing

Thanks for your interest in improving the AI Fiber Alarm Automation pipeline. This is an n8n-based project — most "code" changes are edits to workflow JSON exports, not application source code, so the process below is tailored to that.

## Before you start

- Read [docs/ARCHITECTURE.md](docs/ARCHITECTURE.md) and [docs/WORKFLOW.md](docs/WORKFLOW.md) to understand the 8-workflow chain (`WF-01` → `WF-08`) and how each stage's output must satisfy the next stage's `Validate Input` node.
- Open an issue describing the change before starting on anything non-trivial (a new alarm type, a new vendor mapping, a routing-rule change), so the approach can be agreed on first.

## Making a workflow change

1. Make your change in n8n directly (import the relevant workflow from `workflows/`, edit it on the canvas, test it).
2. Re-export the workflow as JSON from n8n (**Download** from the workflow's menu) and overwrite the corresponding file in `workflows/`.
3. **Never export or commit real credential values, chat IDs, API keys, or spreadsheet IDs that aren't already placeholders.** n8n workflow exports store credential *references* (an ID and a display name), not secrets — verify this before committing.
4. If your change affects a workflow's required input fields or output shape, update the corresponding section in [docs/WORKFLOW.md](docs/WORKFLOW.md) and [docs/API.md](docs/API.md) in the same PR. Documentation and workflow JSON must not drift apart.
5. Test the workflow independently using its own `Execute Workflow Trigger` (or, for WF-01, its webhook) with the minimum valid payload documented in `docs/WORKFLOW.md` before opening a PR.

## Adding a new vendor or alarm type

- Vendor mappings live in WF-02's `2. Detect Vendor & Validate` and `3. Normalize Fields` code nodes.
- Alarm-type keyword detection lives in WF-02's `3. Normalize Fields` (`detectAlarmType`).
- Team-routing rules for a new alarm type live in WF-06's `3. Determine Team Assignment` (`ALARM_TYPE_TEAM_MAP`).
- Update the vendor/alarm-type tables in `docs/ARCHITECTURE.md` and `docs/WORKFLOW.md` to match.

## Commit conventions

- Use clear, imperative commit messages (`Add Juniper vendor mapping`, not `updates`).
- Keep workflow-JSON changes and documentation changes in the same commit/PR when they describe the same change.
- Do not use `git commit --amend` on commits that have already been pushed/reviewed.

## Pull requests

- Describe what changed and why, not just what.
- Note any manual n8n-side testing you did (which workflow, what payload, what result).
- Flag any credential or external-service requirement your change introduces.
