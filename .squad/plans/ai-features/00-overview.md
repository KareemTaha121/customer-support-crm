# AI Features

Feature spec: [../../features/07-ai-features.md](../../features/07-ai-features.md).
Repo: `customer-support-crm-api`. Out of scope for every story: Docker, deployment, CI/CD, and files under `tests/`.

| NN | Tracker id | Plan file | Title | Depends on | Status |
|----|-----------|-----------|-------|-----------|--------|
| 35 | AI-01 | [35-story-ai-assistant-and-chatbot.md](35-story-ai-assistant-and-chatbot.md) | AI assistant (summary, reply, categorization, solutions) and KB chatbot | Tickets (02), KB (06), Settings (12) | Done |
| 48 | BUG-12 | [48-story-hide-ai-when-unconfigured.md](48-story-hide-ai-when-unconfigured.md) | Hide the chatbot and warn admins when no AI provider is configured (QA H2) | 35 | To do |

Intake: [../../stories/ai-features/ai-assistant-and-chatbot/intake.md](../../stories/ai-features/ai-assistant-and-chatbot/intake.md).

The feature was implemented directly on `develop` without a plan in `customer-support-crm-api` commit `a67763f` (feat: add AI features (feature 07)). The `ai_suggestions` migration, the `FeatureToggleBehavior` toggles and `SettingsReader` came with `678ea67`; follow-ups: `889dfcf` (KB context matches any term), `0936711` (en/ar messages for the AI error codes), `e3e5d3d` (`GET /ai/status` reports `false` when `ai.agent_assist_enabled` is off). Plan 35 is an **as-built** plan; its paths and line numbers refer to `2956767`.

Frontend counterparts: [../frontend/16-story-ai-assistant-panels.md](../frontend/16-story-ai-assistant-panels.md) (ticket AI panel), [../frontend/17-story-customer-portal-ui.md](../frontend/17-story-customer-portal-ui.md) (chatbot at `/portal/assistant`), [../frontend/19-story-administration-ui.md](../frontend/19-story-administration-ui.md) (settings toggles).

Known gaps and drift:

- **No `AiConversation`; chatbot is stateless.** The client sends up to 20 turns; nothing is stored apart from the `ai_suggestions` row.
- **Hand-off does not create or update a ticket.** The chatbot only returns `handoff: true`.
- **No automatic categorization** of new tickets; categorize runs only on demand and is never applied by the server.
- **Suggested solutions use KB articles only**, not similar resolved tickets.
- **Toggles are global and coarse:** `ai.agent_assist_enabled` covers all four ticket actions; `chatbot.enabled` covers the chatbot. No per-capability or per-organization configuration.
- **PII / retention not addressed:** ticket text and customer name are sent to the provider; `ai_suggestions` rows are never purged; `docs/security.md` has no AI section.
- **No cost tracking or explicit timeout:** tokens are stored per call, but no cost is calculated and the SDK default timeout applies.
- **Feedback** works only for the requesting staff user; chatbot answers cannot be rated (no requester, no `suggestionId` in the response). `AiSuggestion` uses `Accepted bool?` instead of a `Pending/Accepted/Rejected` status and has no confidence column.
- **No tests** cover the AI feature.
- Code layout differs from the spec: one `Features/Ai/AiSlices.cs` file instead of per-use-case folders, and `IAiCompletionClient` + `AiAssistant` instead of `IAiService`.
