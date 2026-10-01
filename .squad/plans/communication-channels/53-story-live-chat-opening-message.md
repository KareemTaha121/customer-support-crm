# Story 53 — Show the visitor's opening message in both live chat transcripts (Bug: BUG-17)

> Fix plan, not yet implemented. Paths and line numbers refer to customer-support-crm-api `dca992c` (`develop`) and customer-support-crm-web `06c817a` (`main`).
> Intake: [../../stories/communication-channels/live-chat-opening-message/intake.md](../../stories/communication-channels/live-chat-opening-message/intake.md)

## Prerequisites

- Story 33 — [33-story-live-chat.md](33-story-live-chat.md): `StartChatHandler`, the visitor and agent transcript endpoints, `ListChatsHandler`.
- Frontend story 12 — [../frontend/12-story-channels-and-live-chat-console.md](../frontend/12-story-channels-and-live-chat-console.md): the staff chat console.
- Frontend story 17 — [../frontend/17-story-customer-portal-ui.md](../frontend/17-story-customer-portal-ui.md): the portal live chat page.
- Story 50 — [../frontend/50-story-composer-error-state-after-send.md](../frontend/50-story-composer-error-state-after-send.md) edits `chat-console.page.ts` and `portal-chat.page.ts`. Run it first ([roadmap](../qa-2026-10-01-fix-roadmap.md), wave 3 before wave 4). Re-check the line numbers below after rebasing on it.
- Source: manual QA report `.squad/qa/2026-10-01-manual-qa-report.md`, finding M4; `.squad/HANDOFF.md` "Known gaps (low priority)".
- Backend paths are relative to **`customer-support-crm-api/`**, frontend paths to **`customer-support-crm-web/`**.

---

## Story Goal

`StartChatHandler` creates the Chat ticket with the visitor's first message as its **description** (`LiveChat.cs` 131–134). It does not add a ticket message. Both transcripts return only ticket messages (`TicketQueries.GetMessagesAsync`), so the opening message is missing from both:

| View | Before | After |
|---|---|---|
| Staff console, after opening a new chat (`GET /chat/conversations/{id}/messages`) | "No messages yet." | The visitor's opening message first, then the rest |
| Visitor window (`GET /public/chat/conversations/{id}`) | Only messages after the first | The opening message first, shown as "You" |
| Waiting queue (`GET /chat/conversations`) | Name, ticket number, status | Plus a one-line preview of the opening message |
| Ticket details (`GET /tickets/{id}`, `/messages`) | Description plus messages | Unchanged |

### Decision: synthetic first message (option b), not a stored message (option a)

**Option (a)**, storing the opening message as a real inbound `TicketMessage`, was rejected for these reasons:

1. **Every other channel works the same way as chat today.** The first customer text is the ticket description, never a message. See email/WhatsApp/SMS (`InboundChannels.cs` 130–131), web form (254–255), portal (`PortalTicketSlices.cs` 96–97) and the external API (`IntegrationSlices.cs` 410). Chat would be the only channel that stores it twice.
2. **Duplicate display.** The staff ticket page and the portal ticket page show the description and then the messages. The same text would appear twice.
3. **Side effects of `TicketMessageAddedDomainEvent`.** `TicketMessageWriter.AddAsync` calls `Ticket.RecordMessage` (`Ticket.cs` 317–348), which raises that event. Its handlers would produce, at chat start:
   - a second customer-timeline entry "T-…: Customer message", next to the ticket-created one (`TicketEventHandlers.cs` 169–173);
   - a "Customer replied on T-…" notification if an assignment rule already set an assignee on `TicketCreated` (`TicketEventHandlers.cs` 158–161);
   - a `ticket.message_added` webhook in addition to `ticket.created` (`IntegrationSlices.cs` 217–227);
   - a `chatMessage` realtime push to a group nobody has joined yet (`CustomerMessaging.cs` 189–201). This one is harmless.
4. **SLA:** no difference either way. `Ticket.Create` already sets `LastCustomerMessageAt` for non-agent channels (`Ticket.cs` 168–171). `FirstRespondedAt` is set only by agent messages (326–327).
5. Option (a) fixes only new chats. Option (b) also fixes chats already in the database.

**Option (b)**, chosen: both transcript handlers prepend one synthetic `TicketMessageResponse` built from the ticket description:

- `id` = the **ticket id**. It is stable across reloads, so `track message.id` works. It never equals a `TicketMessage` id, so realtime de-duplication (`chat-console.page.ts` 333, `portal-chat.page.ts` 258) never drops a real message. Neither client calls an API with a chat message id.
- `authorType` = `Customer` and `authorCustomerId` = `ticket.CustomerId`. `authorName` follows the customer-name rule of `GetMessagesAsync` (`TicketQueries.cs` 211–213).
- `body` = `ticket.Description`, `isInternal` = false, `channel` = `Chat`, `createdAt` = `ticket.CreatedAt`, `attachments` = [].
- Skipped when the description is empty. Staff can edit it through `PUT /tickets/{id}`; the start-chat validator requires it non-empty (`LiveChat.cs` 115).

**Trade-off (accepted):** if staff edit the ticket description, the chat transcript shows the edited text. Chat tickets are rarely edited. The ticket page shows the same edited description anyway.

**Deviation from the intake:** none.

---

## Context — Read These Files First

1. `src/CustomerSupportCrm.Application/Features/Channels/LiveChat.cs`:
   - `ChatConversationResponse` (33–42) and `VisitorChatResponse` (44);
   - `StartChatValidator` (109–117) and `StartChatHandler` (119–149), where the ticket is created with the message as description (131–134);
   - `GetVisitorChatHandler` (173–182), messages at 179;
   - `ListChatsHandler` (257–287), projection at 274–285;
   - `GetAgentChatMessagesHandler` (316–327), messages at 325.
2. `src/CustomerSupportCrm.Application/Features/Tickets/Common/TicketQueries.cs`: `GetMessagesAsync` (177–221), with the author-name switch at 209–214.
3. `src/CustomerSupportCrm.Contracts/Tickets/TicketContracts.cs`: `TicketMessageResponse` (101–111).
4. `src/CustomerSupportCrm.Domain/Tickets/Ticket.cs`: `Create` (134–175), `RecordMessage` (317–348).
5. `src/CustomerSupportCrm.Application/Features/Tickets/TicketMessageSlices.cs`: `TicketMessageWriter.AddAsync` (26–61), read only. It shows why option (a) raises events.
6. `customer-support-crm-web/src/app/features/channels/channels.models.ts`: `ChatConversation` (9–20), `ChatMessage` (24–32).
7. `customer-support-crm-web/src/app/features/channels/chat-console.page.html`: the queue item (33–58) and the transcript `@for … @empty` with `channels.chat.noMessages` (116–126).
8. `customer-support-crm-web/src/app/features/channels/chat-console.page.ts`: `loadMessages()` (207–231), `appendMessage()` (328–346). No change needed.
9. `customer-support-crm-web/src/app/features/customer-portal/channels/portal-chat.page.ts`: `start()` (124–151) calls `resume()` (153–176). `applyChat()` (252–255) shows `chat.messages` as returned. `isMine()` treats `authorType === 'Customer'` as "You". No change needed.
10. `customer-support-crm-api/docs/endpoints.md` line 155: the staff transcript row.

---

## Backend Tasks

### 1 — Transcript helper (`LiveChat.cs`)

Add an internal static class next to `AgentChats`:

```csharp
/// <summary>
/// Chat transcripts: the ticket description (the visitor's opening message, see StartChatHandler)
/// as a synthetic first message, then the ticket's public messages.
/// </summary>
internal static class ChatTranscript
{
    public static async Task<IReadOnlyList<TicketMessageResponse>> LoadAsync(IApplicationDbContext db, Guid ticketId, CancellationToken cancellationToken)
    {
        var messages = await TicketQueries.GetMessagesAsync(db, ticketId, publicOnly: true, _ => string.Empty, cancellationToken);

        var opening = await db.Tickets.AsNoTracking()
            .Where(t => t.Id == ticketId)
            .Select(t => new
            {
                t.CustomerId,
                t.Description,
                t.CreatedAt,
                CustomerName = db.Customers.IgnoreQueryFilters().Where(c => c.Id == t.CustomerId).Select(c => c.Name).FirstOrDefault(),
            })
            .SingleOrDefaultAsync(cancellationToken);

        if (opening is null || string.IsNullOrWhiteSpace(opening.Description))
        {
            return messages;
        }

        return
        [
            new TicketMessageResponse(
                ticketId,                       // stable, never a TicketMessage id
                nameof(MessageAuthorType.Customer),
                null,
                opening.CustomerId,
                opening.CustomerName ?? "Customer",
                opening.Description,
                false,
                nameof(TicketChannel.Chat),
                opening.CreatedAt,
                []),
            .. messages,
        ];
    }
}
```

- The `"Customer"` fallback copies `GetMessagesAsync` (`TicketQueries.cs` 212).
- `MessageAuthorType` and `TicketChannel` are in `CustomerSupportCrm.Domain.Tickets`, which is already imported (line 16).

### 2 — Use it in both transcript handlers

- `GetVisitorChatHandler` (line 179): `var messages = await ChatTranscript.LoadAsync(db, conversation.TicketId, cancellationToken);`
- `GetAgentChatMessagesHandler` (line 325): `return await ChatTranscript.LoadAsync(db, ticketId, cancellationToken);`

The ticket query filters (soft delete) apply as before. A deleted ticket returns no opening message.

### 3 — Queue preview (`ChatConversationResponse`, `ListChatsHandler`)

- Add `string? Preview` as the **last** positional parameter of `ChatConversationResponse` (after `LastMessageAt`). JSON is camelCase, so it appears as `preview`.
- Add `internal const int PreviewLength = 140;` to `ChatTranscript`.
- In the `ListChatsHandler` projection (274–285), add a ticket subquery like the `Number` one at 278:

```csharp
db.Tickets.Where(t => t.Id == c.TicketId)
    .Select(t => t.Description.Length > ChatTranscript.PreviewLength ? t.Description.Substring(0, ChatTranscript.PreviewLength) : t.Description)
    .FirstOrDefault()
```

  Npgsql translates `Length` and `Substring`. The client adds the ellipsis with CSS, so the server does not append "…".
- `ChatConversationResponse` is built only in `ListChatsHandler`. Search for `new ChatConversationResponse(` to confirm before changing the record.

### 4 — Docs

`docs/endpoints.md` line 155: change the note to "public transcript of the linked ticket, starting with the visitor's opening message (the ticket description); staff scope".

**No changes to:** domain, migrations, `StartChatHandler`, event handlers, `TicketQueries`, resx (no new error codes) or the realtime payloads.

## Frontend Tasks

### 1 — Model (`features/channels/channels.models.ts`)

Add `preview: string | null;` to `ChatConversation`, after `lastMessageAt`.

### 2 — Queue item (`features/channels/chat-console.page.html`, 43–56)

After the second `queue__row` (the one with the ticket number and status, 47–50), add:

```html
@if (conversation.preview) {
  <span class="crm-muted queue__preview" dir="auto">{{ conversation.preview }}</span>
}
```

The visitor writes in either language, whatever the UI language is, so `dir="auto"` isolates the preview's direction.

In `chat-console.page.scss`, next to `.queue__agent` (36), add:

```scss
.queue__preview { overflow: hidden; text-overflow: ellipsis; white-space: nowrap; font-size: 12px; }
```

`.queue__body` already has `min-width: 0` (line 32), so the ellipsis works inside the flex item.

### 3 — Transcripts

No code change. Both pages render the returned list as is:

- The staff console calls `loadMessages()` → `messages.set(items.filter((m) => !m.isInternal))` at 219. The synthetic message has `isInternal: false` and renders with the `message--Customer` style.
- The portal calls `applyChat()` → `messages.set(chat.messages)` at 254. `isMine()` makes the opening message "You".

The `channels.chat.noMessages` empty state stays for transcripts that really are empty. That only happens to an old chat whose description was cleared.

**No i18n changes.**

---

## Test Plan

Test projects are **out of scope**: `tests/` must not be modified. Neither repo has tests covering live chat transcripts.

---

## Verification Steps

1. **Backend build:** `dotnet build src/CustomerSupportCrm.Api/CustomerSupportCrm.Api.csproj` → 0 warnings, 0 errors. If a running API locks `bin/`, build with `-o <temp dir>`.
2. **Frontend build:** `cd customer-support-crm-web && npx ng build` → 0 errors, 0 warnings.
3. **Start a chat:** `POST /api/v1/public/chat/conversations` with `{ "name": "QA", "email": "qa@example.com", "message": "My order #123 has not arrived" }`. Keep `conversationId` and `accessToken`.
4. **Visitor transcript:** `GET /api/v1/public/chat/conversations/{id}` with header `X-Chat-Token: <accessToken>` → `messages[0]` has `id` equal to the ticket id, `authorType: "Customer"` and the body above.
5. **Queue:** `GET /api/v1/chat/conversations?status=waiting` (`chat.handle`) → the item has `preview: "My order #123 has not arrived"`. Start another chat with a 300-character message → `preview` is 140 characters long.
6. **Staff transcript:** `GET /api/v1/chat/conversations/{id}/messages` → the same first message. Accept the chat, then send an agent reply and a visitor reply → they follow it in time order.
7. **No side effects:** after step 3, `GET /api/v1/tickets/{ticketId}/messages` returns `[]`. The customer's history (`GET /api/v1/customers/{id}/history`) has a `ticket.created` entry for the chat ticket and no `ticket.message` entry. No `ticket.message_added` webhook delivery is listed for that ticket.
8. **UI:** in `/chat`, the waiting item shows the preview under the name. Opening it shows the opening message instead of "No messages yet.". In the portal (`/portal/chat`), after starting a chat, the first bubble is the visitor's own message, marked "You". Reload the page: the message is still shown once, not twice.
9. **Arabic:** start a chat with an Arabic message while the staff UI is in English. The preview and the bubble render right to left without mixing up the surrounding layout.

---

## Done Criteria

- [ ] Both transcript endpoints start with the visitor's opening message (synthetic, `id` = ticket id, `authorType` Customer).
- [ ] The staff console and the portal chat window show it; the staff console no longer shows "No messages yet." for a new chat.
- [ ] `GET /chat/conversations` returns `preview` (≤ 140 characters) and the waiting queue shows it.
- [ ] No ticket message, timeline entry, notification or webhook is added when a chat starts; ticket details are unchanged.
- [ ] `docs/endpoints.md` describes the transcript change.
- [ ] Nothing changed in `tests/`, `docker-compose.yml`, `deploy/` or `.github/`.
- [ ] `dotnet build` and `npx ng build` pass with zero warnings.

**STOP HERE. Report to the user.**
