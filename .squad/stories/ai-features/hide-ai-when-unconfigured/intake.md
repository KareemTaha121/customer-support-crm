# Story intake

- Folder: `.squad/stories/ai-features/hide-ai-when-unconfigured/intake.md`

---

## Feature

- **Feature name (display):** AI Features
- **Feature slug (folder under `plans/`):** `ai-features`

## Tracker (metadata only)

- **Tracker type:** `none`
- **Work item id:** `BUG-12`
- **Work item type:** `Bug`
- **Status:** `Done`
- **Assignee:** ``
- **Labels:** `backend`, `frontend`, `bug`, `ai`, `portal`, `settings`

---

## Title

```
Hide the AI chatbot and warn admins when no AI provider is configured
```

---

## Description

```
Manual QA 2026-10-01, finding H2 (.squad/qa/2026-10-01-manual-qa-report.md).

With no AI provider configured (AnthropicAiClient.IsConfigured is false), the
customer portal still shows the "Virtual assistant" nav item and page. Every
question then fails with "AI features are not configured.", an internal message
shown to customers.

Cause: GET /public/features returns the stored chatbot.enabled setting
("true" by default) and never asks whether a provider exists. The portal nav,
the assistant page and the "other channels" list all read that flag.

Staff side: GET /ai/status already ANDs the provider with the agent-assist
toggle, so the ticket AI panel hides itself. But /admin/settings shows the
"Chatbot" and "AI agent assist" toggles as on, with no hint that they have no
effect.

Fix:
- GET /public/features reports chatbot.enabled = setting AND provider configured.
- The chatbot endpoint keeps its 503 AI_NOT_CONFIGURED, and checks it before the
  knowledge base search.
- The portal assistant page treats AI_NOT_CONFIGURED / FEATURE_DISABLED as
  "unavailable" (friendly en/ar page) instead of printing the server message in
  the conversation.
- A staff endpoint tells the Settings page whether an AI provider is configured;
  the page shows a warning (en/ar) next to the two AI toggles.
- After saving settings, the page reloads the public features from the server
  instead of copying the raw setting values.
```

---

## Acceptance criteria

```
- [ ] Without an AI provider, GET /public/features returns chatbot.enabled "false" even when the setting is "true".
- [ ] With a provider and the setting on, it returns "true"; with the setting off, "false".
- [ ] Without a provider, the portal shows no "Virtual assistant" nav item, and /portal/assistant shows the "not available" page.
- [ ] If the chatbot call still returns AI_NOT_CONFIGURED or FEATURE_DISABLED, the portal switches to the "not available" page instead of showing the server text.
- [ ] /admin/settings shows a warning (en/ar) on the Chatbot and AI agent assist rows when no provider is configured, and nothing when one is.
- [ ] Saving settings does not re-enable the portal assistant flag when no provider is configured.
- [ ] `dotnet build` passes with zero warnings; `ng build` passes with zero errors and zero warnings.
```

---

## Attachments

None.

---

## Dependencies

- **Blocked by / related ids:** none.
- **Depends on code areas or other stories:** story 35 (AI assistant and chatbot), story 22 (system settings), frontend stories 16, 17 and 19.

## Technical hints (optional)

- API repo: `customer-support-crm-api/` (branch `develop`). .NET 10.
- Web repo: `customer-support-crm-web/` (branch `main`). Angular.
- `AnthropicAiClient.IsConfigured` is true only when `Ai:Enabled` is true **and** a key exists (`Ai:ApiKey` or `ANTHROPIC_API_KEY`).

## Out of scope

- Do NOT touch Docker, docker-compose.yml, deploy/ or .github/ (CI/CD).
- Do NOT create or modify anything under tests/.
- No e2e tests.
- Configuring a real AI provider, or changing the stored setting values.
- The live chat and web form flags.
