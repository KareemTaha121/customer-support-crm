# Story intake

- Folder: `.squad/stories/frontend/channels-and-live-chat-console/intake.md`

---

## Feature

- **Feature name (display):** Frontend — Angular web app for all 12 features
- **Feature slug (folder under `plans/`):** `frontend`

## Tracker (metadata only)

- **Tracker type:** `none`
- **Work item id:** `FE-05`
- **Work item type:** `Story`
- **Status:** `Done`
- **Assignee:** ``
- **Labels:** `frontend`, `channels`, `chat`

---

## Title

```
Channels and live chat console (feature 03)
```

---

## Description

```
Agent live chat console (chat.handle): queue of conversations
(GET /chat/conversations), accept, send message, close; realtime via /hubs/staff
(`chatMessage`, `chatUpdated`, JoinConversation/LeaveConversation). Channel admin
(channels.manage): channel status, send test message, outbox list with retry.
```

---

## Acceptance criteria

```
- [ ] Agents see new chats and messages without reloading and can accept, reply and close.
- [ ] Channel status, test send and outbox retry work. en/ar + RTL.
```

---

## Attachments

None.

---

## Dependencies

- **Blocked by / related ids:** FE-01, FE-02
- **Depends on code areas or other stories:** backend `customer-support-crm-api` (complete, develop branch); API contract `customer-support-crm-api/docs/api-contract.md`.

## Extra notes (optional)

- Feature specs: `.squad/features/*.md` (Frontend sections).

## Technical hints (optional)

- Repo root: `customer-support-crm-web/` (Angular 22, standalone components, signals, Angular Material 22).
- DTOs: mirror `customer-support-crm-api/src/CustomerSupportCrm.Contracts/**` and response records declared in the slice files under `Application/Features/**`.

## Out of scope

- Docker, deploy/, CI/CD, unit/e2e tests (verification is `ng build` only).
- Backend changes.
