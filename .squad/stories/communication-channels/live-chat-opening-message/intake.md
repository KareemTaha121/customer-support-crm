# Story intake

- Folder: `.squad/stories/communication-channels/live-chat-opening-message/intake.md`

---

## Feature

- **Feature name (display):** Communication Channels
- **Feature slug (folder under `plans/`):** `communication-channels`

## Tracker (metadata only)

- **Tracker type:** `none`
- **Work item id:** `BUG-17`
- **Work item type:** `Bug`
- **Status:** `To do`
- **Assignee:** ``
- **Labels:** `backend`, `frontend`, `bug`, `channels`, `live-chat`

---

## Title

```
Show the visitor's opening message in both live chat transcripts
```

---

## Description

```
A visitor starts a live chat with POST /public/chat/conversations
(name, email, message). StartChatHandler (Features/Channels/LiveChat.cs ~131)
creates the Chat ticket with the message as its description and does not add a
ticket message. Both transcripts read only ticket messages:

- GET /chat/conversations/{id}/messages        (staff, GetAgentChatMessagesHandler)
- GET /public/chat/conversations/{id}           (visitor, GetVisitorChatHandler)

So after accepting, the agent sees "No messages yet." and does not know what the
visitor asked, and the visitor's own window does not show the first message.
The waiting queue (GET /chat/conversations) shows only the name and ticket
number, so the agent cannot triage before accepting.

Listed in .squad/HANDOFF.md "Known gaps (low priority)"; manual QA
(2026-10-01, finding M4) raises it to medium because the agent cannot answer.

Fix:
- Both transcript responses start with the ticket description as a synthetic
  first message from the customer (no new ticket message row).
- The conversation list returns a short preview of the opening message; the
  staff queue shows it.
```

---

## Acceptance criteria

```
- [ ] After a visitor starts a chat, GET /chat/conversations/{id}/messages starts with the opening message (authorType Customer).
- [ ] GET /public/chat/conversations/{id} starts with the same message; the visitor's window shows it as "You".
- [ ] The staff console no longer shows "No messages yet." for a new chat.
- [ ] GET /chat/conversations returns a preview; the waiting queue shows it under the visitor name.
- [ ] Ticket details (GET /tickets/{id}/messages) are unchanged: the description is not duplicated as a message.
- [ ] No new ticket message, timeline entry, notification or webhook is produced when a chat starts.
- [ ] `dotnet build` passes with zero warnings; `npx ng build` passes with zero errors and zero warnings.
```

---

## Attachments

None.

---

## Dependencies

- **Blocked by / related ids:** CH-02 (live chat, plan 33), FE-05 (chat console, plan 12), BUG-14 (plan 50, edits the same chat components; run it first).
- **Depends on code areas or other stories:** source finding M4 in `.squad/qa/2026-10-01-manual-qa-report.md`; `.squad/HANDOFF.md` "Known gaps".

## Technical hints (optional)

- Repo roots: `customer-support-crm-api/` (branch `develop`, .NET 10) and `customer-support-crm-web/` (branch `main`, Angular).

## Out of scope

- Do NOT touch Docker, docker-compose.yml, deploy/ or .github/ (CI/CD).
- Do NOT create or modify anything under tests/.
- No e2e tests.
- Storing the opening message as a real ticket message, or backfilling old chats (the synthetic message covers old chats too).
- The realtime `chatUpdated` payload (the queue reloads from the list endpoint).
