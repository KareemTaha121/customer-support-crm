# Story intake

- Folder: `.squad/stories/communication-channels/live-chat/intake.md`

---

## Feature

- **Feature name (display):** Communication Channels (feature 03)
- **Feature slug (folder under `plans/`):** `communication-channels`

## Tracker (metadata only)

- **Tracker type:** `none`
- **Work item id:** `CH-02`
- **Work item type:** `Story`
- **Status:** `Done`
- **Assignee:** ``
- **Labels:** `backend`, `communication-channels`

---

## Title

```
Live chat (visitor widget API, agent handling, realtime, transcript)
```

---

## Description

```
As built in customer-support-crm-api e921626 (+ f6074d2). Code in
Application/Features/Channels/LiveChat.cs, Domain/Tickets/ChannelRecords.cs (ChatConversation),
Infrastructure/Realtime/{ChatHub.cs, RealtimeHubs.cs} (hubs from 65c74a3).

Each conversation is backed by a ticket (channel Chat, tag "chat"); its public ticket
messages are the transcript. The visitor proves ownership with a per-conversation token
(24 random bytes, hex; only the SHA-256 hash is stored) sent as the X-Chat-Token header.

Visitor endpoints (/api/v1/public, rate limit "public"):
- POST /chat/conversations                 {name, email, message, language?}  toggle chat.enabled
                                           201 {conversationId, accessToken, ticketNumber}
- GET  /chat/conversations/{id}            X-Chat-Token -> {id, status, agentName, messages[]}
- POST /chat/conversations/{id}/messages   X-Chat-Token, {body}
- POST /chat/conversations/{id}/close      X-Chat-Token

Staff endpoints (/api/v1, chat.handle, branch/department scope of the ticket):
- GET  /chat/conversations?status=waiting|active|mine|closed   (default waiting, max 100)
- GET  /chat/conversations/{id}/messages   public transcript (f6074d2)
- POST /chat/conversations/{id}/accept     assigns the conversation and its ticket to the caller
- POST /chat/conversations/{id}/messages   {body}; auto-accepts a waiting chat
- POST /chat/conversations/{id}/close      resolves the ticket

Realtime (SignalR):
- /hubs/chat  (anonymous)  JoinConversation(conversationId, accessToken) -> group chat:{id}
- /hubs/staff (staff JWT)  chat.handle holders auto-join "chat-agents";
                           JoinConversation(id) / LeaveConversation(id)
- Events: chatMessage {conversationId, messageId, authorType, body, createdAt}
          chatUpdated {conversationId, status, visitorName?, ticketNumber?, agentName?}

Rules / errors:
- Wrong or missing token, unknown or out-of-scope conversation -> 404 CHAT_NOT_FOUND.
- Writing to a closed conversation -> 422 CHAT_CLOSED (DomainException).
- Out-of-scope ticket on accept/reply -> 404 TICKET_NOT_FOUND.
- Bodies max 5000 chars; chat disabled -> 409 FEATURE_DISABLED.
```

---

## Acceptance criteria

```
- [ ] Live chat uses real-time transport (e.g. SignalR); chat transcript stored as ticket messages.
- [ ] Customer can chat with support and the chat becomes a ticket automatically.
- [ ] Agent console can list, accept, reply to and close chats.
- [ ] Visitor endpoints are protected against abuse (rate limiting) and only reveal a conversation to its token holder.
- [ ] `dotnet build` passes with zero warnings.
```

---

## Attachments

None.

---

## Dependencies

- **Blocked by / related ids:** CH-01 (`CustomerResolver`, `ChannelDeliveryHandlers` push of chat messages), ticket-management (NN 25–26), platform (NN 20–21: SignalR hubs, `IRealtimeNotifier`).
- **Depends on code areas or other stories:** `TicketFactory`, `TicketMessageWriter`, `TicketQueries`, `IAccessScopeProvider`, `ICurrentUser`, `FeatureToggleBehavior` (`chat.enabled`, added in `678ea67`).

## Extra notes (optional)

- Frontend widget and agent console: `../../../plans/frontend/12-story-channels-and-live-chat-console.md` (FE-05).

## Technical hints (optional)

- Repo root: `customer-support-crm-api/`. .NET 10.

## Out of scope

- Do NOT touch Docker, docker-compose.yml, deploy/ or .github/ (CI/CD).
- Do NOT create or modify anything under tests/.
- Typing indicators, chat transfer between agents, attachments in chat, offline-message routing.
