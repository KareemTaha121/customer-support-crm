# Story intake

- Folder: `.squad/stories/frontend/customer-portal-ui/intake.md`

---

## Feature

- **Feature name (display):** Frontend — Angular web app for all 12 features
- **Feature slug (folder under `plans/`):** `frontend`

## Tracker (metadata only)

- **Tracker type:** `none`
- **Work item id:** `FE-10`
- **Work item type:** `Story`
- **Status:** `Ready`
- **Assignee:** ``
- **Labels:** `frontend`, `portal`

---

## Title

```
Customer portal UI (feature 08)
```

---

## Description

```
Portal area (/portal) using the portal token: my tickets (list, create with
category from /portal/categories, detail with messages, attachments, close,
feedback/CSAT), history; public help/FAQ link; public chatbot
(POST /public/ai/chatbot/messages), live chat widget (/public/chat/conversations +
/hubs/chat JoinConversation) and web form (/public/web-forms/tickets) gated by
GET /public/features.
```

---

## Acceptance criteria

```
- [ ] A verified customer can submit, track, reply to, close and rate tickets.
- [ ] Chatbot, live chat and web form work anonymously when enabled. en/ar + RTL.
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
