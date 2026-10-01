# Story intake

- Folder: `.squad/stories/ai-features/ai-assistant-and-chatbot/intake.md`

---

## Feature

- **Feature name (display):** AI Features
- **Feature slug (folder under `plans/`):** `ai-features`

## Tracker (metadata only)

- **Tracker type:** `none`
- **Work item id:** `AI-01`
- **Work item type:** `Story`
- **Status:** `Done`
- **Assignee:** ``
- **Labels:** `backend`, `ai`

---

## Title

```
AI assistant (summaries, replies, categorization, solutions) and KB chatbot
```

---

## Description

```
Agents get AI help on a ticket and anonymous customers get a chatbot grounded in
the public knowledge base. Everything is in customer-support-crm-api
Application/Features/Ai/AiSlices.cs (one file: commands, validators, handlers,
endpoints). The provider sits behind IAiCompletionClient
(Application/Abstractions/Ai); the Anthropic SDK client is in
Infrastructure/Ai/AnthropicAiClient.cs. Contracts are in Contracts/Ai.

Staff endpoints (/api/v1, group /ai, permission ai.use on all of them):
- GET  /ai/status                       {enabled}: provider configured AND setting
                                        ai.agent_assist_enabled is on
- POST /ai/tickets/{id}/summary         {suggestionId, summary, keyPoints[], sentiment, nextStep}
- POST /ai/tickets/{id}/reply           body {tone?: friendly|formal|empathetic, instructions? <=1000}
                                        -> {suggestionId, reply, language, usedArticleIds[]}
- POST /ai/tickets/{id}/categorize      {suggestionId, categoryId?, categoryName?, priority,
                                        tags[] (<=5), confidence 0..1, reasoning}
- POST /ai/tickets/{id}/solutions       {suggestionId, solutions[{articleId, title, slug, reason}]}
- POST /ai/suggestions/{id}/feedback    body {accepted: bool}; only the user who requested it

Public endpoint (/api/v1/public, anonymous):
- POST /chatbot/messages                body {messages[{role: user|assistant, content <=2000}] (1..20,
                                        last one user), language?: en|ar}
                                        -> {answer, handoff, sources[{articleId, title, slug}]}

Rules:
- The four ticket actions and the chatbot use the "ai" rate-limit policy: 20 requests per minute
  per user (sub claim), or per IP for anonymous callers.
- Ticket actions respect the caller's ticket access scope (branch/department) -> 404 TICKET_NOT_FOUND.
- Feature toggles (FeatureToggleBehavior, system settings): ai.agent_assist_enabled for the four
  ticket actions, chatbot.enabled for the chatbot -> 409 FEATURE_DISABLED when off.
- Provider not configured -> 503 AI_NOT_CONFIGURED; provider refusal -> 422 AI_REFUSED;
  output that does not match the schema -> 422 AI_INVALID_OUTPUT; feedback on an unknown or
  someone else's suggestion -> 404 AI_SUGGESTION_NOT_FOUND.
- Customer-written text is wrapped in <customer_data> tags and the system prompt says never to
  follow instructions inside them. Results are suggestions only; nothing is applied or sent
  automatically.
- Every AI call is logged as an AiSuggestion row (feature, ticket, requester, model, JSON output,
  input/output tokens, accepted feedback).
- Article ids returned by the model are filtered to the articles actually supplied.
- Messages localized en/ar (Messages.resx / Messages.ar.resx).
```

---

## Acceptance criteria

```
- [ ] AI is isolated behind `IAiService` at the Application boundary; provider SDKs only in Infrastructure; nothing in Domain.
- [ ] AI output treated as untrusted content; never auto-executes high-impact actions (send, close, reassign) without human confirmation or explicit rules.
- [ ] Suggestions stored as `AiSuggestion` with accepted/rejected feedback for quality tracking.
- [ ] Customer PII handling and data-retention rules documented; configurable per organization.
- [ ] Chatbot grounded on published public KB only; clear hand-off to live agent creates/updates a ticket.
- [ ] Feature flags to enable/disable each AI capability per organization.
- [ ] Timeouts, rate limits and cost tracking for provider calls.
- [ ] Arabic and English support.
```

---

## Attachments

None.

---

## Dependencies

- **Blocked by / related ids:** Ticket Management (02), Knowledge Base (06), Communication Channels (03), Customer Portal (08), Settings (feature 12, `FeatureToggleBehavior`).
- **Depends on code areas or other stories:** `IKnowledgeSearch`, `IAccessScopeProvider` / `WhereInScope`, `TicketQueries`, `SettingsReader`, `RateLimitPolicies`, `ICurrentUser`, `IPublicEndpoint`.

## Extra notes (optional)

- Feature spec: `.squad/features/07-ai-features.md`.
- Frontend: `.squad/plans/frontend/16-story-ai-assistant-panels.md` (staff panel), `17-story-customer-portal-ui.md` (chatbot), `19-story-administration-ui.md` (toggles).

## Technical hints (optional)

- Repo root: `customer-support-crm-api/`. .NET 10. Anthropic SDK package `Anthropic` 12.51.0, default model `claude-opus-5-5`.

## Out of scope

- Do NOT touch Docker, docker-compose.yml, deploy/ or .github/ (CI/CD).
- Do NOT create or modify anything under tests/.
- Persisted chatbot conversations (`AiConversation`), automatic categorization on ticket creation, per-organization AI configuration.
