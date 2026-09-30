# Story intake

- Folder: `.squad/stories/frontend/agent-dashboard-ui/intake.md`

---

## Feature

- **Feature name (display):** Frontend — Angular web app for all 12 features
- **Feature slug (folder under `plans/`):** `frontend`

## Tracker (metadata only)

- **Tracker type:** `none`
- **Work item id:** `FE-06`
- **Work item type:** `Story`
- **Status:** `Done`
- **Assignee:** ``
- **Labels:** `frontend`, `dashboard`

---

## Title

```
Agent dashboard UI (feature 04)
```

---

## Description

```
Default landing page: agent dashboard widgets from GET /dashboard/agent
(assigned/open/overdue counts, my tickets, due soon, etc.), tasks and reminders
(CRUD, complete/reopen), quick replies (list/CRUD, quickreplies.manage for
shared ones) usable from the ticket reply box (exported picker component).
```

---

## Acceptance criteria

```
- [ ] Dashboard shows the agent's live numbers and lists with links to tickets.
- [ ] Tasks and quick replies can be managed. en/ar + RTL.
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
