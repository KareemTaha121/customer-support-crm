# Story 33 — Live chat (Story: CH-02)

> As-built plan: written after implementation in `customer-support-crm-api` commit `e921626` (feat: add communication channels) with follow-up `f6074d2` (staff chat transcript). Paths and line numbers refer to `f6074d2`; every cited file is unchanged at `develop` HEAD `2956767` except where another commit is named. The SignalR hubs themselves were added earlier, in `65c74a3` (platform).

## Prerequisites

- Story 32 completed: [32-story-inbound-channels-and-outbound-messaging.md](32-story-inbound-channels-and-outbound-messaging.md) — `CustomerResolver`, `ChannelDeliveryHandlers` (pushes `chatMessage` for chat tickets), `CustomerMessenger` (skips chat tickets).
- platform (NN 20–21, `65c74a3`): `StaffHub`, `ChatHub`, `IRealtimeNotifier`, `RealtimeGroups`, `RealtimeEvents`, `IChatAccessValidator`, `IAccessScopeProvider`.
- ticket-management (NN 25–26): `TicketFactory`, `TicketMessageWriter`, `TicketQueries.LoadAsync` / `EnsureAccessibleAsync` / `GetMessagesAsync`, `TicketMessageResponse`, `Ticket.AssignTo`, `Ticket.ChangeStatus`.
- All paths below are relative to the **`customer-support-crm-api/`** repo root.

---

## Story Goal

1. A website visitor starts a chat (name, email, first message); a `Chat` ticket and a `ChatConversation` are created and the visitor receives a one-time access token.
2. The visitor reads the transcript, sends messages and closes the chat with that token, and receives live updates on `/hubs/chat`.
3. Agents with `chat.handle` see the queue live on `/hubs/staff`, list chats by state, read the transcript, accept, reply and close — limited to their branch/department scope.
4. The transcript is the ticket's public messages, so every chat stays searchable and reportable as a ticket.

**Deviations from the intake (the code is authoritative):**

| Feature spec (`03-communication-channels.md`) | As built in `e921626` |
|---|---|
| Slices `LiveChat/StartSession, SendMessage, EndSession` | One file `Features/Channels/LiveChat.cs`: `StartChat`, `VisitorChatMessage`, `GetVisitorChat`, `CloseChat` (visitor and staff), `ListChats`, `AcceptChat`, `AgentChatMessage`, `GetAgentChatMessages` |
| Embeddable widget configured per branch/department | No widget configuration entity; chats land in the oldest active branch (`CustomerResolver`), a global on/off setting `chat.enabled` came in `678ea67` |
| Staff transcript | Not in `e921626` (agents had to use `GET /tickets/{id}`, needing `tickets.view`); added as `GET /chat/conversations/{id}/messages` in `f6074d2` |
| SignalR adapter in `Infrastructure/Channels` | Hubs live in `Infrastructure/Realtime` (`65c74a3`); this story only uses them |
| `StartChatRequest.Language` | Accepted but not used or validated by the handler |
| Migration with the feature | `chat_conversations` table created by `20260930104602_AddSupportOperations` in `678ea67` |

**Not in scope:** typing indicators, transfer between agents, chat attachments, offline routing. **Do not touch** `docker-compose.yml`, `deploy/`, `.github/` or anything under `tests/`.

---

## Context — Read These Files First

1. `src/CustomerSupportCrm.Domain/Tickets/ChannelRecords.cs` — `ChatStatus` (144–149), `ChatConversation` (151–231): `Start` (190–200, status `Waiting`), `NewAccessToken` (202, 24 random bytes hex), `Accept` (204–214), `Touch` (216–224), both throw `DomainException("CHAT_CLOSED")` when closed; `Close` (226–230).
2. `src/CustomerSupportCrm.Infrastructure/Persistence/Configurations/ChannelConfiguration.cs` — `ChatConversationConfiguration` (39–55): table `chat_conversations`, index `(Status, StartedAt)`, **unique** `TicketId`, cascade FKs to tickets and customers, `xmin` concurrency token.
3. `src/CustomerSupportCrm.Infrastructure/Persistence/Migrations/20260930104602_AddSupportOperations.cs` (`678ea67`) — `chat_conversations` 832–864, indexes 1000–1013.
4. `src/CustomerSupportCrm.Application/Abstractions/Notifications/IRealtimeNotifier.cs` (`65c74a3`) — `RealtimeEvents` (13–19: `chatMessage`, `chatUpdated`), `RealtimeGroups` (21–27: `chat-agents`, `chat:{id:N}`). `IChatAccessValidator.cs` lines 3–7.
5. `src/CustomerSupportCrm.Infrastructure/Realtime/ChatHub.cs` (`65c74a3`) — `[AllowAnonymous]`, `Path = "/hubs/chat"` (14), `JoinConversation(conversationId, accessToken)` (16–24) throws `HubException` when the token is wrong.
6. `src/CustomerSupportCrm.Infrastructure/Realtime/RealtimeHubs.cs` (`65c74a3`) — `StaffHub` (16–37, `[Authorize(Policy = PolicyNames.Staff)]`, chat agents group in `OnConnectedAsync` 21–29, `JoinConversation` / `LeaveConversation` 32–36), `SignalRRealtimeNotifier` (45–78, best effort, pushes a group event to both hubs).
7. `src/CustomerSupportCrm.Api/Endpoints/EndpointExtensions.cs` lines 42–43 — `MapHub<StaffHub>`, `MapHub<ChatHub>`.
8. `src/CustomerSupportCrm.Application/Features/Channels/LiveChat.cs` — contracts (26–43), `ChatTokens` (45–65), `ChatAccessValidator` (67–82), visitor side (84–229), staff side (231–379).
9. `src/CustomerSupportCrm.Application/Features/Channels/CustomerMessaging.cs` — `ChannelDeliveryHandlers` chat push (190–201) and chat skip in `QueueAgentReplyAsync` (82–84) / resolved notice (223).
10. `src/CustomerSupportCrm.Application/DependencyInjection.cs` at `e921626` line 47 — `AddScoped<IChatAccessValidator, ChatAccessValidator>()`.
11. `src/CustomerSupportCrm.Domain/Roles/Permissions.cs` — `ChatHandle = "chat.handle"` (31), included in `AgentDefaults` (line 69).
12. `src/CustomerSupportCrm.Application/Features/Tickets/Common/TicketQueries.cs` — `LoadAsync` (28–33, 404 `TICKET_NOT_FOUND` when out of scope), `EnsureAccessibleAsync` (40–46), `GetMessagesAsync` (177).
13. `src/CustomerSupportCrm.Application/Common/Authorization/AccessScope.cs` — `AccessScope` (14), `IAccessScopeProvider` (30), `WhereInScope` (75).
14. `src/CustomerSupportCrm.Application/Features/Settings/SettingsSlices.cs` (`678ea67`) — `FeatureToggleBehavior` line 142: `StartChatCommand` → `chat.enabled`.
15. `docs/endpoints.md` — staff chat 154–156, public chat 205–207, realtime 223–228.

---

## Backend Tasks

### 1 — Domain and persistence

`ChatConversation(TicketId, CustomerId, VisitorName, AccessTokenHash, Status, AgentId?, StartedAt, AcceptedAt?, ClosedAt?, LastMessageAt)`. The token is never stored: `ChatTokens.Hash` = uppercase hex SHA-256 (line 50). `DbSet<ChatConversation> ChatConversations` on `IApplicationDbContext` / `ApplicationDbContext` (Story 32, Context 4).

### 2 — Token check

`ChatTokens.LoadForVisitorAsync(db, id, token, ct)` (52–64): empty token or unknown id or hash mismatch (`CryptographicOperations.FixedTimeEquals`) → `NotFoundException(CHAT_NOT_FOUND)` — one answer for all cases, so ids cannot be probed. Header name `X-Chat-Token` (47). `ChatAccessValidator.CanJoinAsync` wraps it for `ChatHub`.

### 3 — Visitor slices (`VisitorChatEndpoints : IPublicEndpoint`, 197–229)

Group `/chat/conversations` → `/api/v1/public/chat/conversations`, tag `Live chat`, `.RequireRateLimiting(RateLimitPolicies.Public)`.

- `POST /` — `StartChatCommand(StartChatRequest(Name, Email, Message, Language?))`; validator (88–96): name ≤ `Customer.NameMaxLength`, email valid ≤ 254, message ≤ 5000. Handler (98–128): `CustomerResolver.ResolveAsync(Email, …)`; `TicketFactory.CreateAsync(new NewTicket(customer, "Chat: {name}", message, null, Medium, TicketChannel.Chat, null, null, ["chat"], normalizedEmail))`; `ChatConversation.Start(...)`; save; push `chatUpdated {conversationId, status, visitorName, ticketNumber}` to `chat-agents`; 201 `ChatStartedResponse(ConversationId, AccessToken, TicketNumber)`.
- `GET /{id:guid}` — `GetVisitorChatQuery` (150–161) → `VisitorChatResponse(Id, Status, AgentName, Messages)`; messages via `GetMessagesAsync(publicOnly: true)` (internal notes never reach the visitor; attachment URLs blanked).
- `POST /{id:guid}/messages` — `VisitorChatMessageCommand` (130–148): `Touch` (throws `CHAT_CLOSED`), `TicketMessageWriter.AddAsync(Customer, …, TicketChannel.Chat)`; the resulting `TicketMessageAddedDomainEvent` pushes `chatMessage` to `chat:{id}`.
- `POST /{id:guid}/close` — `CloseChatCommand(id, headerValue)`.

### 4 — Staff slices (`AgentChatEndpoints : IEndpoint`, 340–379)

Group `/chat/conversations` → `/api/v1/chat/conversations`, `.RequireAuthorization(Permissions.ChatHandle)`.

- `GET /?status=` — `ListChatsHandler` (236–266): conversations whose ticket is `WhereInScope(scope)`; `active`, `mine` (active and `AgentId == me`), `closed`, anything else → `waiting`; ordered by `StartedAt`, `Take(100)`; projects `ChatConversationResponse(Id, TicketId, TicketNumber, VisitorName, Status, AgentId, AgentName, StartedAt, LastMessageAt)` with correlated sub-selects.
- `GET /{id:guid}/messages` (`f6074d2`) — `GetAgentChatMessagesHandler` (295–310): conversation must belong to an in-scope ticket else 404 `CHAT_NOT_FOUND`; returns public ticket messages. Needs only `chat.handle`, not `tickets.view`.
- `POST /{id:guid}/accept` — `AcceptChatHandler` (270–287): `TicketQueries.LoadAsync` (scope) → `conversation.Accept(me)` + `ticket.AssignTo(me)`; push `chatUpdated {conversationId, status, agentName}` to the conversation and to `chat-agents`. Accepting an already active chat re-assigns it to the caller.
- `POST /{id:guid}/messages` — `AgentChatMessageHandler` (319–338): auto-accept when `Waiting`; `Touch`; `AddAsync(Agent, me, …, TicketChannel.Chat)`; body ≤ 5000.
- `POST /{id:guid}/close` — `CloseChatCommand(id, null)`.

### 5 — Close (shared, 163–195)

Staff path when `currentUser.IsAuthenticated && VisitorToken is null` (scope check via `EnsureAccessibleAsync`), else visitor token path. `Close(now)`; ticket → `Resolved` unless already `Resolved`/`Closed` (no "resolved" email for chat tickets); save; push `chatUpdated {conversationId, status}` to the conversation and `chat-agents`. Closing twice is idempotent.

### 6 — Realtime wiring

- Visitor: connect `/hubs/chat`, invoke `JoinConversation(conversationId, accessToken)`; receives `chatMessage` and `chatUpdated` for that conversation.
- Staff: connect `/hubs/staff` with the JWT (`access_token` query string, `src/CustomerSupportCrm.Infrastructure/DependencyInjection.cs` line 171 at `2956767`); `chat.handle` holders join `chat-agents` automatically; `JoinConversation(id)` to follow one chat.
- Pushes are best effort (`SignalRRealtimeNotifier` logs and swallows failures); the HTTP API remains the source of truth.

### 7 — Localization

`CHAT_NOT_FOUND` exists in `src/CustomerSupportCrm.Application/Resources/Messages.resx` line 183 and `Messages.ar.resx` line 231 (added by `0936711`, not by this story). `CHAT_CLOSED` has no resx entry.

---

## Edge Cases & Failure Modes

- **Wrong / missing token, unknown id** — 404 `CHAT_NOT_FOUND` (visitor) and `HubException("Conversation not found.")` (hub).
- **Message after close** — 422 `CHAT_CLOSED` for visitor and agent.
- **Out-of-scope chat** — list hides it; transcript and close → 404 `CHAT_NOT_FOUND` / `TICKET_NOT_FOUND`; accept / reply → 404 `TICKET_NOT_FOUND`.
- **Two agents accept at once** — `xmin` token on `chat_conversations` → one gets 409 `CONFLICT`.
- **Chat disabled** — `POST /public/chat/conversations` → 409 `FEATURE_DISABLED`; existing conversations keep working.
- **Abuse** — visitor routes share the per-IP `public` limiter (30 / minute) → 429.
- **Staff hub group join** — `StaffHub.JoinConversation` does not check `chat.handle` or scope; any staff connection can follow any conversation id it knows (see overview Known gaps).
- **Same email starts several chats** — each start creates a new ticket and conversation for the matched customer.

---

## Test Plan

Test projects are **out of scope** (`tests/` must not be modified). No tests were added for this story, and no existing test in `tests/` covers live chat or the hubs (searched at `2956767`).

---

## Verification Steps

1. **Backend builds:** in `customer-support-crm-api/` run `dotnet build` — 0 warnings, 0 errors.
2. **Start:** `curl -i -X POST https://localhost:<port>/api/v1/public/chat/conversations -H "Content-Type: application/json" -d '{"name":"Jane","email":"jane@example.com","message":"Hello"}'` → 201 with `conversationId`, `accessToken`, `ticketNumber`; export `CID` and `CTOKEN`.
3. **Visitor read / write:** `curl …/public/chat/conversations/$CID -H "X-Chat-Token: $CTOKEN"` → status `Waiting`, one message; without the header → 404 `CHAT_NOT_FOUND`; `POST …/$CID/messages -d '{"body":"Anyone?"}'` → 200.
4. **Staff queue:** with an agent token `curl …/api/v1/chat/conversations -H "Authorization: Bearer $TOKEN"` → the chat; `?status=mine` → empty.
5. **Accept / reply:** `POST …/api/v1/chat/conversations/$CID/accept` → 200; ticket assigned to the agent; `POST …/$CID/messages -d '{"body":"Hi Jane"}'` → visitor GET shows `agentName` and the reply.
6. **Transcript:** `GET …/api/v1/chat/conversations/$CID/messages` → public messages only.
7. **Close:** `POST …/public/chat/conversations/$CID/close -H "X-Chat-Token: $CTOKEN"` → 200; ticket status `Resolved`; another visitor message → 422 `CHAT_CLOSED`.
8. **Realtime:** connect a SignalR client to `/hubs/chat`, `JoinConversation(CID, CTOKEN)`, send an agent reply → `chatMessage` received; wrong token → hub error.
9. **Permissions:** a token without `chat.handle` on `GET /api/v1/chat/conversations` → 403.

---

## Done Criteria

- [x] Live chat uses SignalR (`/hubs/chat` for visitors, `/hubs/staff` for agents); transcript stored as ticket messages on a `Chat` ticket.
- [x] Visitor start / read / send / close endpoints, token-protected (hash only stored, constant-time compare) and rate-limited.
- [x] Agent list (waiting / active / mine / closed), accept, reply (auto-accept), close, scoped to branch/department.
- [x] Staff transcript endpoint `GET /chat/conversations/{id}/messages` (`f6074d2`).
- [x] Closing a chat resolves its ticket and notifies both sides.
- [ ] Widget configuration per branch/department — not built (global `chat.enabled` toggle only, `678ea67`).
- [ ] `CHAT_CLOSED` localized — no resx entry.
- [x] Nothing changed in `tests/`, `docker-compose.yml`, `deploy/` or `.github/` by this story.
- [x] `dotnet build` passes with zero warnings.

**STOP HERE. Report to the user and wait for confirmation before proceeding to the next feature.**
