# Story 58 — Quick replies fill customer and ticket placeholders (Bug: BUG-22)

> Fix plan, implemented in `customer-support-crm-web` commit `48d8533` (`main`). Paths and line numbers refer to that commit.
> Intake: [../../stories/frontend/quick-reply-ticket-context/intake.md](../../stories/frontend/quick-reply-ticket-context/intake.md)

## Prerequisites

- Frontend story 13 ([13-story-agent-dashboard-ui.md](13-story-agent-dashboard-ui.md)): `QuickReplyPickerComponent`.
- Frontend stories 11 ([11-story-tickets-ui.md](11-story-tickets-ui.md)) and 12 ([12-story-channels-and-live-chat-console.md](12-story-channels-and-live-chat-console.md)): the two composers that host the picker.
- Backend story 27 ([../agent-dashboard/27-story-agent-workspace.md](../agent-dashboard/27-story-agent-workspace.md)): `POST /quick-replies/{id}/render?ticketId=`.

---

## Story Goal

`POST /quick-replies/{id}/render` fills `{{agent.name}}` from the caller. It fills `{{customer.name}}`, `{{customer.number}}`, `{{ticket.number}}` and `{{ticket.subject}}` only when it gets a `ticketId` that is in the caller's scope (`AgentWorkspaceSlices.cs` 402–425, api `613e607`).

The picker already takes the id: `readonly ticketId = input<string | null>(null)` (`quick-reply-picker.component.ts:127`), passed at line 205. Neither host bound it, so the render request never had `?ticketId=`.

| Composer | Before | After |
|---|---|---|
| Ticket details (`ticket-conversation.component.ts:103`) | `<app-quick-reply-picker (selected)=…>` | `[ticketId]="ticket().id"` |
| Live chat console (`chat-console.page.html:149`) | same | `[ticketId]="current()?.ticketId ?? null"` (`ChatConversation.ticketId`, `channels.models.ts:12`) |

**Deviation from the intake:** none. No backend change was needed.

---

## Context — Read These Files First

1. `src/app/features/dashboard/quick-reply-picker.component.ts`: input at 127, `pick()` at 200–210 (falls back to the raw body on error).
2. `src/app/features/dashboard/dashboard.api.ts:70`: `renderQuickReply(id, ticketId)` adds `ticketId` as a query param.
3. `src/app/features/tickets/ticket-conversation.component.ts:103`: `ticket` is `input.required<Ticket>()` (171).
4. `src/app/features/channels/chat-console.page.html:149`: `current` is `signal<ChatConversation | null>` (`chat-console.page.ts:101`).

---

## Frontend Tasks

1. Bind `[ticketId]` in both hosts as shown in the table. No i18n changes.

---

## Test Plan

Test projects are out of scope.

---

## Verification Steps

1. `npx ng build`: 0 errors, 0 warnings.
2. As admin, create a personal quick reply with body `Hi {{customer.name}} ({{customer.number}}), about {{ticket.number}} "{{ticket.subject}}" - {{agent.name}}`.
3. On T-000004, pick it. Composer: `Hi QA Test Customer Renamed (C-000005), about T-000004 "QA: Invoice shows wrong amount" - Administrator`. The request is `/quick-replies/{id}/render?ticketId=…` (200).
4. In `/chat`, open the active conversation T-000005 and pick it. Composer: `… about T-000005 "Chat: QA Test Customer" - Administrator`.

All three passed on 2026-10-01.

---

## Done Criteria

- [x] Ticket details render with the ticket id; all placeholders are filled.
- [x] Live chat console renders with the open conversation's ticket id.
- [x] `npx ng build` passes with 0 errors and 0 warnings.

**STOP HERE. Report to the user.**
