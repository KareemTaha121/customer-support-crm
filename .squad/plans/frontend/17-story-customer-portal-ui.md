# Story 17 — Customer portal UI (Story: FE-10)

## Prerequisites

- Story 08 completed: [08-story-core-platform-shell.md](08-story-core-platform-shell.md) (`ApiService`, `ApiError`, i18n, `BrandingService` feature flags, shared states and form helpers).
- Story 09 completed: [09-story-authentication-staff-and-portal.md](09-story-authentication-staff-and-portal.md) (portal shell, `PortalAuthService`, `portalAuthGuard`, portal i18n scope). This story **extends** `features/customer-portal/**` and keeps every Story 09 route and key.
- Conventions and ownership: [00-overview.md](00-overview.md). Edit only `src/app/features/customer-portal/**` and `public/i18n/portal/{en,ar}.json`.

---

## Story Goal

1. A signed-in, verified customer lists their tickets (open / closed / all, paged), creates a ticket (subject, message, category, attachments), opens a ticket to read the public thread, replies with attachments, downloads attachments with the customer token, closes the ticket and rates it (CSAT 1–5 stars + comment).
2. The customer sees an activity history (`/portal/history`).
3. Anonymous visitors (inside the portal shell) can use the **web form** (`/portal/contact`), the **live chat** widget (`/portal/chat`) and the **chatbot** (`/portal/assistant`) when the matching public feature flag is on; otherwise a friendly "unavailable" state with links to the other channels is shown.
4. Everything is available in English and Arabic (RTL).

Not in scope: help-center pages (Story 15 owns `/help`), staff chat console (Story 12), backend changes, tests.

---

## Context — Read These Files First

1. `customer-support-crm-api/src/CustomerSupportCrm.Application/Features/CustomerPortal/PortalTicketSlices.cs`
   - lines 56–76: `PortalListTicketsQuery(Page, PageSize, Status)` — `status` = `open` (default) | `closed` | `all`, page size clamped to 1–50.
   - lines 78–86: create validator (subject required, message ≤ 10 000); category ignored when inactive.
   - lines 126–134: `PortalAddMessageCommand` — body required, **≤ 10 attachment ids**.
   - lines 147–158: upload stores the file as a public ticket attachment; lines 160–170 download (public only).
   - lines 172–193: feedback rating 1–5, comment ≤ 2000. lines 195–206: close = active → Resolved, Resolved → Closed.
   - lines 214–292: endpoints under `/api/v1/portal`: `GET /categories`, `GET /history?page` (25 per page), `GET|POST /tickets`, `GET /tickets/{id}`, `GET|POST /tickets/{id}/messages`, `POST /tickets/{id}/attachments` (multipart `file`), `GET /tickets/{id}/attachments/{attachmentId}`, `POST /tickets/{id}/feedback`, `POST /tickets/{id}/close`.
2. `customer-support-crm-api/src/CustomerSupportCrm.Contracts/Portal/PortalContracts.cs` lines 22–50 — request/response records (`PortalTicketResponse.canReply`, `canGiveFeedback`, `satisfactionRating`).
3. `customer-support-crm-api/src/CustomerSupportCrm.Application/Features/Channels/LiveChat.cs`
   - lines 26–43: `StartChatRequest(Name, Email, Message, Language)`, `ChatStartedResponse(ConversationId, AccessToken, TicketNumber)`, `VisitorChatResponse(Id, Status, AgentName, Messages)`.
   - lines 45–65: the per-conversation token travels in the **`X-Chat-Token`** header; wrong token → 404 `CHAT_NOT_FOUND`.
   - lines 88–96: start validator (name, email, **message required**, ≤ 5000).
   - lines 197–229: `/public/chat/conversations` POST, `GET /{id}`, `POST /{id}/messages`, `POST /{id}/close`.
4. `customer-support-crm-api/src/CustomerSupportCrm.Infrastructure/Realtime/ChatHub.cs` lines 11–25 — anonymous hub `/hubs/chat`, `JoinConversation(conversationId, accessToken)`; `RealtimeHubs.cs` lines 63–68 send group events to both hubs.
5. `customer-support-crm-api/src/CustomerSupportCrm.Application/Features/Channels/CustomerMessaging.cs` lines 190–200 — `chatMessage` payload `{ conversationId, messageId, authorType, body, createdAt }` for every chat message (visitor and agent); `LiveChat.cs` lines 192, 283–285 — `chatUpdated` `{ conversationId, status, agentName? }`.
6. `customer-support-crm-api/src/CustomerSupportCrm.Application/Features/Channels/InboundChannels.cs` lines 215–231, 263–271 — `POST /public/web-forms/tickets` with `WebFormTicketRequest(Name, Email, Phone, Subject, Message, CategoryId, Language, Website)`; `Website` is a honeypot that must stay empty; response `{ ticketNumber }`.
7. `customer-support-crm-api/src/CustomerSupportCrm.Application/Features/Ai/AiSlices.cs` lines 383–397, 500–509 — chatbot route is **`POST /public/chatbot/messages`** (not `/public/ai/...`), stateless: ≤ 20 turns, role `user|assistant`, last turn `user`, content ≤ 2000. `Contracts/Ai/AiContracts.cs` lines 27–34 — `ChatbotResponse(Answer, Handoff, Sources[{articleId,title,slug}])`.
8. `customer-support-crm-api/src/CustomerSupportCrm.Api/Middleware/GlobalExceptionHandler.cs` lines 90–93 — validation field paths are camel-cased property paths, so wrapped requests report `form.name` (web form) and `request.name` (chat start); strip the prefix before `applyServerErrors`.
9. Enums: `Domain/Tickets/TicketEnums.cs` lines 3–12 (`TicketStatus`), 35–40 (`MessageAuthorType`); `Domain/Tickets/ChannelRecords.cs` lines 144–149 (`ChatStatus`); `Domain/Customers/CustomerRecords.cs` lines 116–128 (activity types).
10. Web: `core/http/api.service.ts` (lines 14–22 `RequestOptions.headers`, 67–89 `upload`/`download`), `core/interceptors/auth.interceptor.ts` (portal token on `/portal/*`, nothing on `/public/*`), `core/branding/feature-flags.ts`, `core/config/app-config.ts` (`API_ORIGIN`), `core/realtime/staff-hub.service.ts` (HubConnection pattern to mirror).

---

## Frontend Tasks

### 1 — Models and API (`features/customer-portal/`)

- Create `customer-portal.models.ts`: `PortalTicketListItem`, `PortalTicket`, `PortalMessage`, `PortalCategory`, `PortalHistoryItem`, `PortalTicketFilter`, `ChatStarted`, `VisitorChat`, `ChatMessage` (mirrors `TicketMessageResponse`), `ChatMessageEvent`, `ChatUpdatedEvent`, `ChatbotTurn/Response/Source`, `WebFormTicketRequest/Response`; helpers `ticketStatusTone(status)` (pill class), `historyTypeKey(type)` (`ticket.created` → `ticket_created`), `withoutFieldPrefix(error, prefix)`.
- Create `customer-portal.api.ts` — `PortalTicketsApi` (all `/portal/*` calls; `uploadAll(ticketId, files)` uploads sequentially) and `PortalPublicApi` (`/public/web-forms/tickets`, `/public/chat/conversations*` with the `X-Chat-Token` header, `/public/chatbot/messages`), every call `{ anonymous: true }` for public ones.

### 2 — Ticket pages (`tickets/`, `portalAuthGuard`)

- `portal-tickets.page.ts` — status toggle (open/closed/all), `mat-table` (number, subject, status pill, created, last agent reply) + `mat-paginator` (20, max 50), row link to detail, "New ticket" button, empty/error states.
- `portal-new-ticket.page.ts` — form (subject ≤ 200, category from `/portal/categories` with `nameAr` in Arabic, message ≤ 10 000) + `portal-file-picker.component.ts` (≤ 10 files, `model<File[]>`). Submit: create → upload files → `POST messages` with the translated "attached files" body and `attachmentIds` (attachments only surface in the portal through messages) → navigate to the detail page. `{ silent: true }` + `applyServerErrors`.
- `portal-ticket-detail.page.ts` — header with number/status, description, public message thread (customer messages aligned to the inline end), attachment chips downloading via `ApiService.download('/portal/tickets/{id}/attachments/{attId}')` + `saveBlob`; reply form when `canReply` (upload files first, then post body + ids); "Mark as resolved" / "Close ticket" via `ConfirmService`; CSAT card (5 star buttons + comment) when `canGiveFeedback && satisfactionRating === null`, read-only rating otherwise.
- `portal-history.page.ts` — paged activity list (25 per page, `page` query only), translated type + summary, link to the ticket when `ticketId`.

### 3 — Public channel pages (`channels/`, anonymous, flag-gated)

- `portal-unavailable.component.ts` — `<app-portal-unavailable [title]>` with links to the other enabled channels and the help center.
- `portal-contact.page.ts` (`FeatureFlags.webForm`) — name, email, phone, subject, message, hidden honeypot `website`; language = UI language; prefill from the portal profile; success panel with the ticket number.
- `portal-chat.page.ts` (`FeatureFlags.liveChat`) — start form (name, email, first message); persists `{ conversationId, accessToken, ticketNumber }` in `sessionStorage` (`crm.portal.chat`); loads the transcript with the token, opens a `HubConnection` to `${API_ORIGIN}/hubs/chat` (`withAutomaticReconnect`), invokes `JoinConversation` on connect and on reconnect, appends `chatMessage` events (dedupe by id), applies `chatUpdated` (status/agent). Composer disabled when closed; "End chat" (confirm) and "Start a new chat". `CHAT_NOT_FOUND` clears storage. Stop the hub on destroy.
- `portal-chatbot.page.ts` (`FeatureFlags.chatbot`) — chat-style transcript, sends the last 20 turns + language, shows answer, suggested articles (links to `/help/articles/{slug}`) and, when `handoff`, buttons to live chat / contact form / new ticket depending on flags and sign-in.

### 4 — Routes, navigation, shell, i18n

- `customer-portal.routes.ts`: replace the `tickets` placeholder with `tickets` (`''`, `new`, `:id`), add `history` (guarded), `contact`, `chat`, `assistant` (anonymous). Keep `login`/`register`/`verify`/`profile`, the shell and the redirects.
- `portal-navigation.ts`: add `flag?: string`; entries for history, contact, chat, assistant. `portal-shell.component.ts` hides items whose flag is off.
- `public/i18n/portal/{en,ar}.json`: add `status`, `chatStatus`, `tickets`, `newTicket`, `ticket`, `feedback`, `history`, `files`, `contact`, `chat`, `chatbot`, `unavailable`; extend `nav`, `fields`, `actions`. Same key set in both files, real Arabic.

---

## Edge Cases & Failure Modes

- Portal token expired → the auth interceptor signs out and redirects to `/portal/login?returnUrl=…` (`core/interceptors/auth.interceptor.ts`); pages do nothing special.
- Ticket of another customer / unknown id → 404 `TICKET_NOT_FOUND` → detail shows `<app-error-state>` with retry and a back link.
- Attachment upload fails after the ticket was created → toast the error and still navigate to the new ticket (the ticket exists; the customer can re-attach in a reply).
- More than 10 files → the picker refuses extra files (server rule `PortalTicketSlices.cs` line 131); oversized file → 413 `PAYLOAD_TOO_LARGE` localized by `describeError`.
- Feedback submitted twice / not allowed → 422 → toast via `describeError`; the page reloads the ticket.
- Web form / chat start validation errors come back as `form.*` / `request.*` → `withoutFieldPrefix` maps them to controls.
- Honeypot `website` filled by a bot → server rejects it; the field is visually hidden and `tabindex=-1`.
- Stored chat token no longer valid (DB reset) → `CHAT_NOT_FOUND` → clear `sessionStorage`, show the start form.
- Hub connection fails (proxy without WebSockets) → SignalR falls back to other transports; if it still fails, the page keeps working over HTTP (transcript reloaded after each send; manual refresh button).
- Agent closes the chat → `chatUpdated` with `Closed` → composer disabled, "Start a new chat" offered.
- Chatbot rate limit (429) → localized `RATE_LIMITED`; the user turn stays in the transcript with a retry.
- Flags are read from `BrandingService.features()` loaded at startup; a flag turned off while a page is open takes effect on reload.
- Help-center article route: Story 15 has not fixed its URL; this story links to `/help/articles/{slug}` — adjust if Story 15 chooses another path.

## Test Plan

Out of scope (build-level verification only). Manual smoke: create ticket with 2 files → reply → download → mark resolved → rate 4 stars → close; history lists the events; web form returns a ticket number; chat start → agent reply arrives live → reload keeps the conversation → end chat; chatbot answer with articles and handoff buttons; switch to Arabic and check RTL.

## Verification Steps

1. **Frontend builds:** `cd customer-support-crm-web && npx ng build` — no errors or warnings in `src/app/features/customer-portal/**`.

## Done Criteria

- [ ] A verified customer can submit, track, reply to, close and rate tickets (with attachments).
- [ ] History page lists portal activity with links to tickets.
- [ ] Chatbot, live chat and web form work anonymously when enabled and show an unavailable state otherwise.
- [ ] en/ar translations with identical keys; layouts work in RTL.
