# Story 16 — AI assistant panels (Story: FE-09)

## Prerequisites

- Story 08 completed: [08-story-core-platform-shell.md](08-story-core-platform-shell.md) (`ApiService`, `ApiError`, i18n, permissions, toast).
- Story 09 completed: [09-story-authentication-staff-and-portal.md](09-story-authentication-staff-and-portal.md) (signed-in staff user and permission set).
- Story 11 (tickets UI) embeds the panel; it is built in parallel and only depends on the public contract below.

---

## Story Goal

1. A reusable `<app-ticket-ai-panel>` for the ticket details side column offers four AI actions: summary, suggested reply, categorization and suggested knowledge base solutions.
2. The panel renders nothing unless the user holds `ai.use` **and** `GET /ai/status` reports `enabled: true` (cached once per session in a root service).
3. "Insert" hands the suggested reply to the ticket page (`replySuggested`); "Apply" hands the suggested category/priority (`categorySuggested`). Nothing is applied automatically.
4. Every action has its own loading and inline error state; suggestions carrying a `suggestionId` can be rated with thumbs up/down (`POST /ai/suggestions/{id}/feedback`).
5. Full en/ar translations and RTL-safe styles.

Not in scope: the customer chatbot (`POST /public/chatbot/messages`, Story 17), AI admin toggles (Story 19).

---

## Context — Read These Files First

1. `customer-support-crm-api/src/CustomerSupportCrm.Application/Features/Ai/AiSlices.cs`
   - lines 28–34 — error codes `AI_NOT_CONFIGURED` (503), `AI_REFUSED`, `AI_INVALID_OUTPUT` (422), `AI_SUGGESTION_NOT_FOUND` (404).
   - lines 151–187 — summary (`sentiment` ∈ positive|neutral|negative|frustrated, `nextStep` nullable).
   - lines 189–249 — suggested reply; `tone` ∈ friendly|formal|empathetic (validator lines 193–200), `instructions` ≤ 1000 chars; reply written in the customer's language (`language` = `en`|`ar`).
   - lines 274–320 — categorization; `categoryId`/`categoryName` null when nothing fits, `priority` ∈ Low|Medium|High|Urgent, `confidence` 0–1.
   - lines 322–379 — solutions; returns `suggestionId = Guid.Empty` and no items when there are no KB candidates (lines 336–339).
   - lines 453–497 — endpoints: group `/ai` requires `ai.use`; `GET /status`, `POST /tickets/{id}/summary|reply|categorize|solutions`, `POST /suggestions/{id}/feedback` (only the requesting user's suggestion).
2. `customer-support-crm-api/src/CustomerSupportCrm.Contracts/Ai/AiContracts.cs` lines 1–36 — response/request records mirrored in `ai.models.ts`.
3. `customer-support-crm-api/src/CustomerSupportCrm.Application/Features/Settings/SettingsSlices.cs` lines 28, 49–55, 135–149 — the `ai.agent_assist_enabled` toggle makes the four ticket actions fail with `FEATURE_DISABLED` (409). The setting is **not public** (`Domain/Integrations/Integrations.cs` lines 297–311), so `BrandingService.features()` normally lacks it; the panel only hides when the flag is explicitly `"false"` and otherwise maps `FEATURE_DISABLED` to an inline message.
4. `customer-support-crm-web/src/app/core/http/api.service.ts` lines 33–58 (`get`/`post`, `silent`), `core/http/api-error.ts` (`code`, `hasCode`), `core/interceptors/error.interceptor.ts` lines 32–45 (`describeError`).
5. `customer-support-crm-web/src/app/core/permissions/permissions.ts` (`Permissions.aiUse`), `core/permissions/permission.service.ts` (`has()` reads signals).
6. `customer-support-crm-web/src/app/core/localization/translation.service.ts` lines 33–36 (`load(scope)`), `core/branding/feature-flags.ts` (`FeatureFlags.aiAgentAssist`).
7. Placeholder being replaced: `customer-support-crm-web/src/app/features/ai/ticket-ai-panel.component.ts` (public contract, lines 1–24).

---

## Frontend Tasks

All files under `customer-support-crm-web/src/app/features/ai/` plus `public/i18n/ai/{en,ar}.json`.

### 1 — Models and API

- Create `ai.models.ts`: `AiStatus`, `TicketSummary`, `SuggestReplyRequest`, `SuggestedReply`, `Categorization`, `SolutionSuggestion`, `SolutionSuggestions`, `AiFeedbackRequest`, plus `AiTone`, `AiSentiment`, `EMPTY_GUID`.
- Create `ai.api.ts` (`providedIn: 'root'`): `status()`, `summarize(id)`, `suggestReply(id, body)`, `categorize(id)`, `solutions(id)`, `feedback(suggestionId, accepted)`. Ticket actions and feedback pass `{ silent: true }` (errors are shown inline).

### 2 — Status cache

- Create `ai-status.service.ts` (`providedIn: 'root'`): `enabled` signal (`boolean | null`), `ensureLoaded()` calls `GET /ai/status` once (silent) only when the user has `ai.use`; failures resolve to `false` for this load and allow a later retry.

### 3 — Panel component

- Replace `ticket-ai-panel.component.ts` (+ `ticket-ai-panel.component.html`, `ticket-ai-panel.component.scss`). Keep selector `app-ticket-ai-panel`, `ticketId = input.required<string>()`, `replySuggested = output<string>()`, `categorySuggested = output<AiCategorySuggestion>()`, and `AiCategorySuggestion { categoryId; priority }` (extended with optional `categoryName`, `tags`).
- Extra optional inputs: `canInsertReply` (default `true`), `canApplyCategory` (default `true`) so the ticket page can hide the buttons for read-only users.
- Constructor: `inject(TranslationService).load('ai')`, `AiStatusService.ensureLoaded()`.
- `visible = computed(() => permissions.has(aiUse) && status.enabled() === true && flag !== 'false')`.
- Layout: compact card with header and a `mat-accordion multi` of four `mat-expansion-panel`s. Each action state is a `linkedSignal` that resets when `ticketId` changes; late responses for a previous ticket are dropped.
  - Summary: sentiment pill, summary, key points, next step.
  - Reply: tone select + optional instructions (reactive form, max 1000), reply shown with `dir` from the reply language, "Insert" and "Regenerate".
  - Categorize: category (or "no match"), translated priority, tags, confidence %, reasoning, "Apply".
  - Solutions: list of links to `/knowledge-base/articles/{articleId}` with the reason; empty state.
- Create `ai-feedback.component.ts` (`<app-ai-feedback [suggestionId]>`): thumbs up/down, sends once, shows "thanks"/error inline; hidden for `EMPTY_GUID`.
- Error mapping (`ai.errors.*`): `AI_NOT_CONFIGURED`, `AI_REFUSED`, `AI_INVALID_OUTPUT`, `FEATURE_DISABLED`, `RATE_LIMITED`, `AI_SUGGESTION_NOT_FOUND`; otherwise `describeError`.

### 4 — i18n

- Create `public/i18n/ai/en.json` and `ar.json` with identical keys: `panel`, `summary`, `reply`, `tone`, `categorize`, `priority`, `sentiment`, `solutions`, `feedback`, `errors`.

---

## Edge Cases & Failure Modes

- User without `ai.use` → no `/ai/status` request (it would be 403) and nothing rendered.
- AI not configured (`enabled: false`) → nothing rendered. Status request fails → hidden for now, retried the next time a panel is created.
- `FEATURE_DISABLED` / `AI_NOT_CONFIGURED` / `AI_REFUSED` / `AI_INVALID_OUTPUT` / 429 → localized inline error with retry.
- `ticketId` changes while a request is in flight → state reset; stale response ignored.
- Solutions with `suggestionId = 00000000-…` → "no matching articles", no feedback buttons.
- Categorize with `categoryId = null` → shows "no matching category"; Apply still offers the priority.
- Reply in Arabic on an English UI (and vice versa) → reply block uses the reply's own `dir`.

## Test Plan

Out of scope (build-level verification only). Manual smoke: open a ticket with AI configured, run each action, insert/apply, rate a suggestion; switch to Arabic; sign in as a user without `ai.use`.

## Verification Steps

1. **Frontend builds:** `cd customer-support-crm-web && npx ng build` with no errors or warnings in `src/app/features/ai/**`.

## Done Criteria

- [ ] Panel hidden when AI is disabled or the user lacks `ai.use`.
- [ ] Summary, reply (Insert), categorize (Apply) and solutions each show loading/error states.
- [ ] Feedback can be sent for suggestions with an id.
- [ ] en/ar translations with identical keys; RTL-safe styles.
