# Story intake

- Folder: `.squad/stories/communication-channels/staff-hub-conversation-access/intake.md`

---

## Feature

- **Feature name (display):** Communication Channels
- **Feature slug (folder under `plans/`):** `communication-channels`

## Tracker (metadata only)

- **Tracker type:** `none`
- **Work item id:** `BUG-01`
- **Work item type:** `Bug`
- **Status:** `Done`
- **Assignee:** ``
- **Labels:** `backend`, `bug`, `chat`

---

## Title

```
Staff hub: check permission and scope before joining a chat conversation
```

---

## Description

```
StaffHub.JoinConversation(conversationId) (src/CustomerSupportCrm.Infrastructure/Realtime/RealtimeHubs.cs)
adds any signed-in staff connection to RealtimeGroups.ChatConversation(id) without any check.
Any staff user who knows (or guesses) a conversation id receives that conversation's live
messages, even without chat.handle or outside their branch/department.

The HTTP routes already enforce this: /chat/conversations/* require chat.handle and filter
the linked ticket by the caller's AccessScope (GetAgentChatMessagesHandler, LiveChat.cs).

Fix:
- JoinConversation requires the chat.handle permission claim on the connection.
- The conversation's ticket must be in the caller's AccessScope (same rule as the
  staff transcript endpoint). Users with data.all_branches see everything.
- Failure throws HubException("Conversation not found.") so existence is not disclosed
  (same message as ChatHub).
- Scope is computed from the hub's ClaimsPrincipal (Context.User), not IHttpContextAccessor.
- LeaveConversation is unchanged (leaving a group is harmless).
```

---

## Acceptance criteria

```
- [ ] A staff connection without chat.handle cannot join any conversation (HubException).
- [ ] A chat agent cannot join a conversation whose ticket is outside their branch/department scope.
- [ ] A chat agent in scope (or with data.all_branches) joins as before; the chat console keeps working.
- [ ] The scope rule matches GET /chat/conversations/{id}/messages (shared code, not a copy).
- [ ] `dotnet build` passes with zero warnings.
```

---

## Attachments

None.

---

## Dependencies

- **Blocked by / related ids:** none
- **Depends on code areas or other stories:** found while writing the as-built plans 20–36 (see `.squad/HANDOFF.md`, "Probable backend bugs").

## Technical hints (optional)

- Repo root: `customer-support-crm-api/` (branch `develop`). .NET 10.

## Out of scope

- Do NOT touch Docker, docker-compose.yml, deploy/ or .github/ (CI/CD).
- Do NOT create or modify anything under tests/.
