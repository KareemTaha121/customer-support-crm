# Story 35 — AI assistant and KB chatbot (Story: AI-01)

> As-built plan: written after implementation. The feature shipped in `customer-support-crm-api` commit `a67763f` (feat: add AI features); the `ai_suggestions` table, the feature toggles and `SettingsReader` came with `678ea67`; follow-ups are `889dfcf` (any-term KB matching for AI context), `0936711` (en/ar messages) and `e3e5d3d` (`/ai/status` honours `ai.agent_assist_enabled`). **Paths and line numbers refer to `2956767`** (current `develop`); `AiSlices.cs` has not changed since `e3e5d3d`; in `a67763f` its lines after the `using` block are 2 lower (two `using` lines were added later).

## Prerequisites

- Ticket Management (02): `Ticket`, `TicketMessage`, `TicketCategory`, `TicketQueries.NotFound()` (`TICKET_NOT_FOUND`), `AccessScope` / `IAccessScopeProvider` / `WhereInScope`.
- Knowledge Base (06): `KnowledgeArticle`, `ArticleStatus.Published`, `IKnowledgeSearch` (PostgreSQL full text).
- Platform / Settings (feature 12, `678ea67`): `SettingsReader`, `SystemSettings` catalog, `FeatureToggleBehavior`.
- Phase 2: permission policies (one policy per code), `ICurrentUser`, `ApiResults`, `GlobalExceptionHandler`.
- All paths below are relative to the **`customer-support-crm-api/`** repo root.

---

## Story Goal

1. An agent with `ai.use` asks for a **summary**, a **suggested reply** (AR/EN, tone, optional instructions), a **categorization** (category, priority, tags, confidence) or **suggested KB solutions** for a ticket they can see.
2. The web app can ask `GET /ai/status` whether to show the AI panel at all.
3. The agent rates a suggestion (accepted / rejected); every AI call is logged with model and token usage.
4. An anonymous customer chats with a stateless **chatbot** that answers only from published public articles and signals `handoff` when a human is needed.

Guards: customer text fenced as untrusted data, nothing applied automatically, per-user rate limit, admin toggles.

**Deviations from the intake (the code is authoritative):**

| Intake / feature spec | As built |
|---|---|
| `IAiService` at the Application boundary | `IAiCompletionClient` (generic completion: system, messages, JSON schema, effort) in `Application/Abstractions/Ai`; the task logic is the scoped `AiAssistant` class in Application. The SDK is only in Infrastructure; Domain has no provider types |
| One slice folder per use case (`SummarizeTicket/`, `SuggestReply/`, …) | One file `Features/Ai/AiSlices.cs` holding all commands, validators, handlers and endpoints; contracts in `Contracts/Ai/AiContracts.cs` |
| `AiSuggestion(Type, Input ref, Output, Confidence, Status: Pending/Accepted/Rejected)` | `Domain/Tickets/AiSuggestion.cs`: `Feature`, `TicketId?`, `RequestedBy?`, `Model`, `Output` (jsonb), `InputTokens`, `OutputTokens`, `Accepted` (`bool?`: null = no feedback), `CreatedAt`. No confidence column (it stays inside `Output` for categorize); no input reference beyond the ticket id |
| `AiConversation` (chatbot sessions) | **Not built.** Chatbot is stateless: the client sends up to 20 recent turns |
| Hand-off creates/updates a ticket | **Not built.** The chatbot only returns `handoff: true`; the client then offers live chat / web form (frontend story 17) |
| New tickets arrive pre-categorized | **Not automatic.** Categorization runs only on demand (`POST /ai/tickets/{id}/categorize`); the agent applies the result |
| Suggested solutions include similar resolved tickets | KB articles only (search candidates re-ranked by the model) |
| Feature flags per capability, per organization | Two global system settings: `ai.agent_assist_enabled` (all four ticket actions together) and `chatbot.enabled`. Single-organization deployment, no per-capability switch |
| PII handling / retention documented and configurable | **Not built.** Ticket text and customer name go to the provider; `ai_suggestions` rows are kept forever; nothing in `docs/security.md` |
| Timeouts, rate limits, cost tracking | Rate limit `ai` (20/min per user or IP). Tokens stored per call; **no cost calculation, no explicit timeout** (SDK default) |
| Feedback endpoint | `POST /ai/suggestions/{id}/feedback`; only the requesting user may rate; chatbot answers have no requester and no `suggestionId` in the response, so they cannot be rated |

**Not in scope:** auto-apply of any AI output, admin UI (frontend stories 16, 17, 19). **Do not touch** `docker-compose.yml`, `deploy/`, `.github/` or anything under `tests/`.

---

## Context — Read These Files First

1. `src/CustomerSupportCrm.Application/Abstractions/Ai/IAiCompletionClient.cs` — lines 5–35. `AiRole`, `AiMessage`, `AiRequest(Feature, System, Messages, JsonSchema, MaxTokens, Effort)`, `AiCompletion(Text, Refused, RefusalCategory, Model, InputTokens, OutputTokens)`, `IAiCompletionClient { IsConfigured; CompleteAsync }`.
2. `src/CustomerSupportCrm.Infrastructure/Ai/AnthropicAiClient.cs` — lines 8–21 `AiOptions` (section `Ai`: `Enabled`, `ApiKey`, `Model = "claude-opus-5-5"`, `UseRefusalFallback = true`); lines 27–86 `AnthropicAiClient`: client created only when `Enabled` and a key exists (option or `ANTHROPIC_API_KEY`, 34–42); `Beta.Messages.Create` with effort and `BetaJsonOutputFormat` (54–74); `StopReason == "refusal"` → `Refused` (76–79); server-side fallback beta `server-side-fallback-2026-07-01` (29, 70–71).
3. `src/CustomerSupportCrm.Infrastructure/DependencyInjection.cs` — lines 135–136: `AddOptions<AiOptions>().BindConfiguration("Ai")`, `AddSingleton<IAiCompletionClient, AnthropicAiClient>()`.
4. `src/CustomerSupportCrm.Application/DependencyInjection.cs` — line 33 `FeatureToggleBehavior<,>` pipeline behaviour, line 52 `AddScoped<AiAssistant>()`, line 53 `AddScoped<SettingsReader>()`.
5. `src/CustomerSupportCrm.Application/Features/Ai/AiSlices.cs` — the whole feature (sections below).
6. `src/CustomerSupportCrm.Domain/Tickets/AiSuggestion.cs` — lines 10–61; `Record(...)` (47–58), `RecordFeedback(bool)` (60).
7. `src/CustomerSupportCrm.Infrastructure/Persistence/Configurations/AiSuggestionConfiguration.cs` — lines 9–19: table `ai_suggestions`, `Feature` ≤ 30, `Model` ≤ 100, `Output` jsonb, indexes `(Feature, CreatedAt)` and `TicketId`. DbSet: `IApplicationDbContext.Ai.cs` line 8 / `ApplicationDbContext.Ai.cs` line 8.
8. `src/CustomerSupportCrm.Infrastructure/Persistence/Migrations/20260930104602_AddSupportOperations.cs` — line 22 creates `ai_suggestions`, lines 964–972 its indexes (migration from `678ea67`; `a67763f` shipped no migration).
9. `src/CustomerSupportCrm.Application/Features/Settings/SettingsSlices.cs` — lines 32–55 `SettingsReader` (`IsEnabledAsync`, `EnsureEnabledAsync` → 409 `FEATURE_DISABLED`); lines 136–160 `FeatureToggleBehavior` mapping `ChatbotCommand` → `chatbot.enabled` and the four ticket commands → `ai.agent_assist_enabled` (144–148).
10. `src/CustomerSupportCrm.Domain/Integrations/Integrations.cs` — lines 297–315 `SystemSettings` (`ChatbotEnabled` 302, `AiAgentAssistEnabled` 303; both default `"true"`, chatbot public, agent assist not public).
11. `src/CustomerSupportCrm.Domain/Roles/Permissions.cs` — line 38 `AiUse = "ai.use"`; in `AgentDefaults` (line 69).
12. `src/CustomerSupportCrm.Api/Configuration/SecurityExtensions.cs` — lines 154–157 policy `RateLimitPolicies.Ai` (fixed window 20/min, partition `sub` or IP). Constant in `Application/Abstractions/Http/RateLimitPolicies.cs` line 12.
13. `src/CustomerSupportCrm.Application/Common/Exceptions/ServiceUnavailableException.cs` — `ServiceUnavailableException` (503) and `UnprocessableException` (422); mapped in `src/CustomerSupportCrm.Api/Middleware/GlobalExceptionHandler.cs` lines 65–66. `ErrorCodes.ServiceUnavailable = "SERVICE_UNAVAILABLE"` in `Contracts/Common/ErrorCodes.cs`.
14. `src/CustomerSupportCrm.Application/Abstractions/Search/IKnowledgeSearch.cs` — `KnowledgeSearchRequest(..., MatchAny = false)`; `src/CustomerSupportCrm.Infrastructure/Search/PostgresKnowledgeSearch.cs` line 48 joins terms with `|` when `MatchAny` (`889dfcf`).
15. `src/CustomerSupportCrm.Application/Resources/Messages.resx` lines 210–220 and `Messages.ar.resx` lines 282–292 — `AI_NOT_CONFIGURED`, `AI_REFUSED`, `AI_INVALID_OUTPUT`, `AI_SUGGESTION_NOT_FOUND` (added in `0936711`); `SERVICE_UNAVAILABLE` (111), `FEATURE_DISABLED` (114).

---

## Backend Tasks

### 1 — Provider abstraction and Anthropic client

- `IAiCompletionClient.cs` (Application) — provider-neutral request/response records above. Output is untrusted text.
- `AnthropicAiClient.cs` (Infrastructure) — `internal sealed`, `IDisposable`, singleton. Effort `low|medium|high` → `Effort.*` (default medium); JSON schema passed as structured-output format; text = concatenated text blocks; refusal returns `Refused = true` with empty text.
- Package: `Directory.Packages.props` `<PackageVersion Include="Anthropic" Version="12.51.0" />`; `PackageReference` in `CustomerSupportCrm.Infrastructure.csproj`.
- Config: `src/CustomerSupportCrm.Api/appsettings.json` lines 87–92 (`"Ai": { "Enabled": false, "ApiKey": "", "Model": "claude-opus-5-5", "UseRefusalFallback": true }`). Disabled by default.

### 2 — `AiAssistant` and prompt helpers (`AiSlices.cs`)

- `AiErrors` (30–36): `AI_NOT_CONFIGURED`, `AI_REFUSED`, `AI_INVALID_OUTPUT`, `AI_SUGGESTION_NOT_FOUND`.
- `AiAssistant(IAiCompletionClient, IApplicationDbContext, ICurrentUser, TimeProvider)` (42–107):
  - `RunAsync<T>(feature, ticketId, system, userContent, schema, effort, maxTokens, ct)` (51–98): not configured → `ServiceUnavailableException(AI_NOT_CONFIGURED)`; refused → `UnprocessableException(AI_REFUSED)`; `JsonSerializer.Deserialize<T>` failure → `UnprocessableException(AI_INVALID_OUTPUT)`; then `AiSuggestion.Record(...)` (requester = current user when authenticated) and `SaveChangesAsync`; returns `(result, suggestion.Id)`.
  - `UntrustedDataRule` (46–47), `Fence(value)` → `<customer_data>…</customer_data>` (106), `ToSchema` / `StringArray` helpers (101–104). `#pragma warning disable CA1861` for the inline schema arrays (26).
- `TicketContext.BuildAsync(db, scope, ticketId, includeInternal, ct)` (110–151): ticket loaded with `WhereInScope` or `TICKET_NOT_FOUND`; customer name + preferred language; last 40 messages (3 000 chars each, 60 000 total); subject/description and customer messages fenced.
- `KnowledgeContext.LoadAsync(db, search, query, publicOnly, ct, language, limit = 5)` (253–274): `IKnowledgeSearch` with `PublishedOnly: true, MatchAny: true` (259), re-checks `ArticleStatus.Published`, bodies clipped to 4 000 chars, returned as `<article id=… title=…>` blocks (trusted).

### 3 — Ticket actions

| Command (lines) | Feature key | Context | Effort / max tokens | Response |
|---|---|---|---|---|
| `SummarizeTicketCommand(TicketId)` (155–189) | `summary` | transcript incl. internal notes | low / 2000 | `TicketSummaryResponse(SuggestionId, Summary, KeyPoints, Sentiment, NextStep)`; sentiment enum `positive|neutral|negative|frustrated` |
| `SuggestReplyCommand(TicketId, Tone, Instructions)` (193–251) + `SuggestReplyValidator` (195–202: tone `friendly|formal|empathetic` → `INVALID_VALUE`, instructions ≤ 1000) | `reply` | transcript incl. internal notes (prompt forbids revealing them) + 5 staff-visible published articles by subject | medium / 4000 | `SuggestedReplyResponse(SuggestionId, Reply, Language, UsedArticleIds)`; language = customer's preferred (`ar` → Arabic, else English); agent display name in prompt; used ids filtered to supplied articles (248) |
| `CategorizeTicketCommand(TicketId)` (278–322) | `categorize` | public transcript + active categories list | low / 1500 | `CategorizationResponse(SuggestionId, CategoryId?, CategoryName?, Priority, Tags, Confidence, Reasoning)`; unknown category → null, unparsable priority → `Medium`, tags ≤ 5, confidence clamped 0–1 (318–320) |
| `SuggestSolutionsCommand(TicketId)` (326–381) | `solutions` | public transcript + up to 8 candidate articles (subject + description) | low / 2000 | `SolutionSuggestionsResponse(SuggestionId, Solutions[ArticleId, Title, Slug, Reason])`; no candidates → `(Guid.Empty, [])` without calling the provider (339–342) |

All four handlers take `IAccessScopeProvider` and pass `await scopes.GetAsync(ct)` to `TicketContext`.

### 4 — Chatbot

- `ChatbotCommand(Messages, Language)` (385) and `ChatbotValidator` (387–400): 1–20 turns, role `user|assistant` (`INVALID_VALUE`), content 1–2000, last turn must be `user`, language `en|ar|null`.
- `ChatbotHandler` (406–453): KB context from the last user message with `publicOnly: true` and the requested language; organization name from `db.Organizations` (fallback "our company"); customer turns fenced; system prompt: answer only from the articles, hand off for people/account/refund/complaint requests, never ask for passwords or payment details. `feature = "chatbot"`, `ticketId = null`, effort low / 1500. `Handoff = output.Handoff || no articles found` (451); sources filtered to supplied articles.

### 5 — Endpoints

- `AiEndpoints : IEndpoint` (457–502) — `MapGroup("/ai").WithTags("AI").RequireAuthorization(Permissions.AiUse)`:
  - `GET /status` (464–467) → `AiStatusResponse(ai.IsConfigured && settings.IsEnabledAsync(ai.agent_assist_enabled))` (`e3e5d3d`; before it only `IsConfigured`). Name `GetAiStatus`.
  - `POST /tickets/{id:guid}/summary|reply|categorize|solutions` (469–488) — `.RequireRateLimiting(RateLimitPolicies.Ai)`, names `SummarizeTicket`, `SuggestReply`, `CategorizeTicket`, `SuggestSolutions`, `ApiResults.Ok`.
  - `POST /suggestions/{id:guid}/feedback` (490–500) — inline handler: loads `AiSuggestions` where `Id == id && RequestedBy == currentUser` or `NotFoundException(AI_SUGGESTION_NOT_FOUND)`; `RecordFeedback(request.Accepted)`; `ApiResults.Success()`. No rate limit, no toggle.
- `ChatbotEndpoints : IPublicEndpoint` (504–513) — `POST /chatbot/messages` under `/api/v1/public` (`EndpointExtensions.cs` line 30), `messages ?? []`, rate limit `ai`, name `ChatbotMessage`.

### 6 — Error mapping and localization

- `GlobalExceptionHandler.StatusFor`: `ServiceUnavailableException` → 503, `UnprocessableException` → 422; `ErrorResponseWriter` maps a bare 503 to `SERVICE_UNAVAILABLE`.
- Messages (en/ar) for the four AI codes — see Context item 15.

### 7 — Docs

- `docs/endpoints.md` lines 158–164 (AI staff routes, `ai.use`, toggle note) and line 210 (`POST /chatbot/messages`, toggle `chatbot.enabled`).

No other DI changes: MediatR handlers and validators are assembly-scanned; `IEndpoint` / `IPublicEndpoint` implementations are discovered by `AddEndpoints`.

---

## Edge Cases & Failure Modes

- **Provider not configured** (`Ai:Enabled` false or no key) — every action → 503 `AI_NOT_CONFIGURED`; `/ai/status` → `enabled: false` (the panel hides itself).
- **Toggle off** — ticket actions → 409 `FEATURE_DISABLED` (behaviour runs after validation); `/ai/status` → `false`. `chatbot.enabled` off → chatbot 409 `FEATURE_DISABLED`.
- **Ticket outside the caller's scope or unknown** → 404 `TICKET_NOT_FOUND`; non-GUID id → route miss 404.
- **Prompt injection in customer text** — fenced in `<customer_data>`; output is only returned, never executed or saved to the ticket.
- **Model invents article ids / category ids** — dropped (reply 248, categorize 318, solutions 373–378, chatbot 445–450).
- **Malformed model JSON** → 422 `AI_INVALID_OUTPUT`; nothing logged. **Refusal** → 422 `AI_REFUSED` (with fallback beta enabled the API may first retry on a fallback model).
- **Solutions with no candidate articles** — 200 with empty list and `suggestionId = 00000000-…`; feedback on that id → 404.
- **Chatbot with no matching public article** — `handoff: true` regardless of the model's answer.
- **Feedback on another agent's suggestion** → 404 `AI_SUGGESTION_NOT_FOUND`; feedback can be changed by posting again.
- **Rate limit** — 21st AI call within a minute → 429 `RATE_LIMITED` with `Retry-After`.
- **Long tickets** — transcript capped (40 messages / 60 000 chars) so token use stays bounded.

---

## Test Plan

Test projects are **out of scope** (`tests/` must not be modified). No tests are added, changed or removed by this story. There are **no existing tests** in `2956767` that cover the AI feature (no AI, chatbot or `ai_suggestions` references under `tests/`).

---

## Verification Steps

1. **Backend builds:** in `customer-support-crm-api/` run `dotnet build` — 0 warnings, 0 errors.
2. **Run:** set `Ai__Enabled=true` and `ANTHROPIC_API_KEY` (or `Ai__ApiKey`), migrate, start the API, sign in as an agent and export `TOKEN`.
3. **Status:** `curl https://localhost:<port>/api/v1/ai/status -H "Authorization: Bearer $TOKEN"` → `{"data":{"enabled":true}}`. Without the key → `false`.
4. **Summary:** `curl -X POST …/api/v1/ai/tickets/{ticketId}/summary -H "Authorization: Bearer $TOKEN"` → `suggestionId`, `summary`, `keyPoints`, `sentiment`.
5. **Reply:** `curl -X POST …/api/v1/ai/tickets/{ticketId}/reply -H "Content-Type: application/json" -d '{"tone":"formal"}'` → reply in the customer's language; `{"tone":"angry"}` → 400 `INVALID_VALUE`.
6. **Categorize / solutions:** `POST …/categorize` → `priority`, `confidence` 0–1; `POST …/solutions` → article list.
7. **Feedback:** `curl -X POST …/api/v1/ai/suggestions/{suggestionId}/feedback -d '{"accepted":true}'` → 200; check `ai_suggestions.accepted`, `input_tokens`, `output_tokens`.
8. **Toggle:** `PUT /api/v1/settings` with `{"values":{"ai.agent_assist_enabled":"false"}}` (settings.manage) → `/ai/status` `false`, summary → 409 `FEATURE_DISABLED`.
9. **Chatbot:** `curl -X POST …/api/v1/public/chatbot/messages -H "Content-Type: application/json" -d '{"messages":[{"role":"user","content":"How do I reset my password?"}],"language":"en"}'` → `answer`, `handoff`, `sources`.
10. **Localization:** a failing call with `Accept-Language: ar` → Arabic message. **Permissions:** a user without `ai.use` → 403 on `/ai/*`.

---

## Done Criteria

- [x] Provider isolated behind `IAiCompletionClient` (Application); Anthropic SDK only in Infrastructure; Domain holds only `AiSuggestion`. (Named differently from `IAiService` — see deviations.)
- [x] AI output treated as untrusted (fenced input, filtered ids) and never applied or sent automatically.
- [x] Every call stored as `AiSuggestion` with model, tokens and accepted/rejected feedback.
- [ ] PII handling and retention documented and configurable — not built.
- [x] Chatbot grounded on published public KB only, with a `handoff` signal. [ ] Hand-off does not create or update a ticket — not built.
- [x] Toggles `ai.agent_assist_enabled` and `chatbot.enabled`. [ ] Per-capability / per-organization flags — not built.
- [x] Rate limit (`ai`, 20/min) and token tracking. [ ] Explicit provider timeout and cost calculation — not built.
- [x] Arabic and English: reply/chatbot language, en/ar messages for all AI error codes.
- [x] Nothing changed in `tests/`, `docker-compose.yml`, `deploy/` or `.github/` by this story.
- [x] `dotnet build` passes with zero warnings.

**STOP HERE. Report to the user and wait for confirmation before proceeding to Story 36.**
