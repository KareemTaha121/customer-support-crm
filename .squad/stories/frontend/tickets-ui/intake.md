# Story intake

- Folder: `.squad/stories/frontend/tickets-ui/intake.md`

---

## Feature

- **Feature name (display):** Frontend — Angular web app for all 12 features
- **Feature slug (folder under `plans/`):** `frontend`

## Tracker (metadata only)

- **Tracker type:** `none`
- **Work item id:** `FE-04`
- **Work item type:** `Story`
- **Status:** `Done`
- **Assignee:** ``
- **Labels:** `frontend`, `tickets`

---

## Title

```
Tickets UI (feature 02)
```

---

## Description

```
features/tickets: list with filters (status, priority, category, assignee,
date, search), paging/sorting; create ticket (customer lookup, category,
priority); details page: conversation (public replies vs internal notes),
attachments, side panel with customer, SLA due times, assignment (user lookup
GET /users/lookup), status transitions / resolve / close / reopen / escalate
with reason, history tab. Realtime refresh on SignalR `ticketUpdated`.
Ticket category admin (GET/POST/PUT /ticket-categories).
Permissions: tickets.view/create/update/assign/escalate/delete,
tickets.categories_manage.
```

---

## Acceptance criteria

```
- [ ] Every ticket endpoint in Application/Features/Tickets has a UI entry point.
- [ ] Only valid actions for the current status are offered; domain errors are shown.
- [ ] Internal notes are visually distinct. en/ar + RTL.
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
