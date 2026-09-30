# Story 12 — Channels and live chat console (Story: FE-05)

## Prerequisites

- Story 08 completed: [08-story-core-platform-shell.md](08-story-core-platform-shell.md) (`ApiService`, `StaffHubService`, i18n, guards, shared states).
- Story 09 completed: [09-story-authentication-staff-and-portal.md](09-story-authentication-staff-and-portal.md) (staff session; realtime connects when signed in).
- Cross-feature contract: `<app-quick-reply-picker (selected)>` in `features/dashboard/quick-reply-picker.component.ts` is owned by Story 13. Use it as-is; **do not edit** it (it renders nothing until Story 13 lands).

---

## Story Goal

1. Agents with `chat.handle` open `/chat`: a two-pane console (queue on the start side, conversation on the end side; stacked on narrow screens).
2. The queue filters by `waiting` (default), `mine`, `active` and `closed` (backend `ListChatsHandler` values) and refreshes live when `chatUpdated` arrives.
3. Opening a conversation loads the transcript, joins the SignalR conversation group, appends `chatMessage` pushes and auto-scrolls. Agents accept, reply (quick replies insert text) and close.
4. Admins with `channels.manage` open `/channels`: provider status cards, a "send test message" form and the outbox table (status filter, server paging, retry for failed rows).
5. Everything is translated (en/ar) and RTL-safe.

Not in scope: the visitor chat widget (Story 17), inbound channel webhooks, backend changes.

---

## Context — Read These Files First

1. `customer-support-crm-api/src/CustomerSupportCrm.Application/Features/Channels/LiveChat.cs`
   - ~lines 32–41 `ChatConversationResponse` (id, ticketId, ticketNumber, visitorName, status, agentId, agentName, startedAt, lastMessageAt).
   - ~lines 233–266 `ListChatsQuery`/`ListChatsHandler`: `status` = `waiting` (default) | `active` | `mine` | `closed`, max 100 rows, not paged.
   - ~lines 268–287 `AcceptChatHandler` — pushes `chatUpdated` `{ conversationId, status, agentName }` to the agents group and the conversation group.
   - ~lines 289–315 `AgentChatMessageHandler` — body 1–5000 chars; auto-accepts a waiting chat. `CHAT_CLOSED` when closed.
   - ~lines 163–195 `CloseChatHandler` — resolves the ticket, pushes `chatUpdated` `{ conversationId, status }`.
   - ~lines 120–124 `StartChatHandler` — new chat pushes `chatUpdated` `{ conversationId, status, visitorName, ticketNumber }` to agents.
   - ~lines 317–350 `AgentChatEndpoints` — `GET /chat/conversations?status=`, `POST /chat/conversations/{id}/accept|messages|close` (`chat.handle`).
   - **There is no staff "get conversation messages" endpoint**: the transcript is the backing ticket's messages.
2. `customer-support-crm-api/src/CustomerSupportCrm.Application/Features/Tickets/TicketQuerySlices.cs` ~lines 245–248 — `GET /tickets/{id}/messages` (`tickets.view`) returns `TicketMessageResponse[]` (`Contracts/Tickets/TicketContracts.cs` ~lines 101–111).
3. `customer-support-crm-api/src/CustomerSupportCrm.Application/Features/Channels/CustomerMessaging.cs`
   - ~lines 190–200 — `chatMessage` payload `{ conversationId, messageId, authorType, body, createdAt }`, sent **only to the conversation group** (so the agent must `JoinConversation`).
   - ~lines 287–291 — `ChannelStatusResponse(channel, configured)`, `OutboundMessageResponse`, `TestChannelRequest(channel, to)`.
   - ~lines 295–302 — test channel must be `Email` | `WhatsApp` | `Sms`, `to` ≤ 320.
   - ~lines 314–361 — `GET /channels/status`, `POST /channels/test`, `GET /channels/outbox?status=&page=` (fixed page size 50), `POST /channels/outbox/{id}/retry` (`OUTBOUND_MESSAGE_NOT_FOUND`).
4. `customer-support-crm-api/src/CustomerSupportCrm.Domain/Tickets/ChannelRecords.cs` ~lines 7–12 `OutboundStatus` (Pending, Sent, Failed), ~lines 144–149 `ChatStatus` (Waiting, Active, Closed), `Accept` throws `CHAT_CLOSED`.
5. `customer-support-crm-api/src/CustomerSupportCrm.Infrastructure/Realtime/RealtimeHubs.cs` ~lines 16–37 — `StaffHub` adds `chat.handle` users to the agents group on connect; `JoinConversation(Guid)` / `LeaveConversation(Guid)`.
6. `customer-support-crm-web/src/app/core/realtime/staff-hub.service.ts` — `RealtimeEvents` (lines 7–12), `on<T>()` (~line 45), `invoke()` (~line 55), `state` signal.
7. `customer-support-crm-web/src/app/core/http/api.service.ts` — `get`, `getPaged`, `post` with `{ silent: true }`; `api.models.ts` `Paged<T>`, `emptyPage`.
8. `customer-support-crm-web/src/app/shared/form-errors.ts` `applyServerErrors`; `core/interceptors/error.interceptor.ts` ~line 33 `describeError`.
9. `customer-support-crm-web/src/app/features/auth/profile.page.ts` — style reference (OnPush, signals, `NonNullableFormBuilder`, silent form errors).
10. `customer-support-crm-web/src/app/features/channels/channels.routes.ts` — placeholder exports `CHAT_ROUTES`, `CHANNELS_ROUTES` (lazy-loaded by `app.routes.ts`).

---

## Frontend Tasks

### 1 — Models and API

Create file: `customer-support-crm-web/src/app/features/channels/channels.models.ts`

```ts
export type ChatStatus = 'Waiting' | 'Active' | 'Closed';
export type ChatFilter = 'waiting' | 'mine' | 'active' | 'closed';
export interface ChatConversation { id: string; ticketId: string; ticketNumber: string; visitorName: string; status: ChatStatus; agentId: string | null; agentName: string | null; startedAt: string; lastMessageAt: string; }
export interface ChatMessage { id: string; authorType: 'Agent' | 'Customer' | 'System'; authorName: string; body: string; isInternal: boolean; createdAt: string; }
export interface ChatMessageEvent { conversationId: string; messageId: string; authorType: string; body: string; createdAt: string; }
export interface ChatUpdatedEvent { conversationId: string; status: ChatStatus; agentName?: string; visitorName?: string; ticketNumber?: string; }
export interface ChannelStatus { channel: string; configured: boolean; }
export type OutboundStatus = 'Pending' | 'Sent' | 'Failed';
export interface OutboundMessage { id: string; channel: string; to: string; subject: string | null; status: OutboundStatus; attempts: number; lastError: string | null; createdAt: string; sentAt: string | null; ticketId: string | null; }
export const TEST_CHANNELS = ['Email', 'WhatsApp', 'Sms'] as const;
```

Create file: `customer-support-crm-web/src/app/features/channels/channels.api.ts` — `ChannelsApi` (`providedIn: 'root'`): `listChats(status)`, `messages(ticketId)` (`/tickets/{id}/messages`, silent), `accept(id)`, `send(id, body)` (silent), `close(id)`, `channelStatus()`, `sendTest(req)` (silent), `outbox(status, page)` (`getPaged`), `retry(id)`.

### 2 — Chat console

Create file: `customer-support-crm-web/src/app/features/channels/chat-console.page.ts`

- Signals: `filter`, `conversations`, `loading`, `error`, `selectedId`, `selected` (computed), `messages`, `messagesLoading`, `sending`.
- Queue: `mat-button-toggle-group` for the filters, list of conversations (visitor, ticket number, status pill, agent, relative last-message time).
- `chatUpdated` (`StaffHubService.on`) → debounce 300 ms → reload queue silently; if it concerns the open conversation, patch its status/agent.
- Open → `LeaveConversation(previous)`, `JoinConversation(id)`, load `/tickets/{ticketId}/messages` (filter out `isInternal`). `chatMessage` for the open id → append unless `messageId` already present. On destroy → `LeaveConversation`.
- Auto-scroll: `viewChild` on the scroll container + `afterRenderEffect` reading `messages()`.
- Composer: textarea (required, max 5000), Ctrl/Cmd+Enter sends, `<app-quick-reply-picker (selected)="insertQuickReply($event)" />`. After a successful send the message is appended via the `chatMessage` push; reload the transcript if no push arrives (hub disconnected).
- Actions: Accept (Waiting), Close (not Closed, `ConfirmService.ask` destructive), link to the ticket `/tickets/{ticketId}`.
- Layout: CSS grid `minmax(260px, 340px) 1fr`; below 900 px only one pane is shown with a back button (`rtl-flip`).

### 3 — Channels admin

Create file: `customer-support-crm-web/src/app/features/channels/channels-admin.page.ts`

- Status cards from `GET /channels/status` (icon per channel, "Configured"/"Not configured" pill).
- Test form: channel select (`TEST_CHANNELS`), recipient; `{ silent: true }` + `applyServerErrors`; success toast.
- Outbox: status filter (all/Pending/Sent/Failed), `mat-table` columns created, channel, to, subject, status, attempts, last error, actions (retry when `Failed`, or `Pending` with attempts > 0); `mat-paginator` with fixed `pageSize` 50 (no size options), refresh button.

### 4 — Routes

File: `customer-support-crm-web/src/app/features/channels/channels.routes.ts` — replace placeholders:

```ts
export const CHAT_ROUTES: Routes = [{ path: '', canActivate: [requirePermission(Permissions.chatHandle)], resolve: { i18n: translationResolver('channels') }, loadComponent: () => import('./chat-console.page').then((m) => m.ChatConsolePage) }];
export const CHANNELS_ROUTES: Routes = [{ path: '', canActivate: [requirePermission(Permissions.channelsManage)], resolve: { i18n: translationResolver('channels') }, loadComponent: () => import('./channels-admin.page').then((m) => m.ChannelsAdminPage) }];
```

### 5 — Translations

Create `customer-support-crm-web/public/i18n/channels/en.json` and `ar.json` (identical keys): `chat.*`, `chatStatus.*`, `authorType.*`, `admin.*`, `outboxStatus.*`, `channel.*`, `errors.*`.

No backend changes required.

---

## Edge Cases & Failure Modes

- Agent lacks `tickets.view` → transcript request returns 403; the pane shows an error state with a localized hint (`channels.chat.transcriptForbidden`); accept/reply/close still work.
- Hub disconnected → `StaffHubService.invoke` resolves without joining after 10 s; a "live updates offline" chip shows from `hub.state()`, and after sending the transcript is reloaded.
- Duplicate `chatMessage` (reload + push race) → dedupe by message id.
- Chat closed by the visitor while open → `chatUpdated` sets status `Closed`; composer disabled. Server `CHAT_CLOSED` → toast with `channels.errors.chatClosed`.
- Another agent accepts a waiting chat → `chatUpdated` reloads the queue; the waiting filter drops it.
- Switching conversations quickly → stale transcript responses are ignored (compare `selectedId` on arrival).
- Outbox page beyond total after a filter change → filter change resets to page 1.
- Retry of a vanished row → `OUTBOUND_MESSAGE_NOT_FOUND` global snackbar; table reloads.
- Test form: server validation on `channel`/`to` → `applyServerErrors` onto controls.

## Test Plan

Out of scope (build-level verification only). Manual smoke: start a chat from the public widget endpoint, see it appear under Waiting, accept, exchange messages both ways, close; send a test email; retry a failed outbox row; switch to Arabic and check RTL.

## Verification Steps

1. **Frontend builds:** `cd customer-support-crm-web && npx ng build` — no errors or warnings in `src/app/features/channels/**`.

## Done Criteria

- [ ] Agents see new chats and messages without reloading and can accept, reply and close.
- [ ] Channel status, test send and outbox retry work.
- [ ] en/ar translations with identical keys; layout works in RTL.
