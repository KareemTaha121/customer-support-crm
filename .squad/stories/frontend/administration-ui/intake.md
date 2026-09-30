# Story intake

- Folder: `.squad/stories/frontend/administration-ui/intake.md`

---

## Feature

- **Feature name (display):** Frontend — Angular web app for all 12 features
- **Feature slug (folder under `plans/`):** `frontend`

## Tracker (metadata only)

- **Tracker type:** `none`
- **Work item id:** `FE-12`
- **Work item type:** `Story`
- **Status:** `Ready`
- **Assignee:** ``
- **Labels:** `frontend`, `admin`

---

## Title

```
Administration UI (features 10, 11, 12)
```

---

## Description

```
Users (users.manage): list/search, create, edit, roles, enable/disable.
Roles (roles.manage): list, create/edit with permission picker from
GET /permissions, delete. Organization (organization.manage / settings.manage):
profile, branding colors + logo upload, branches and departments.
Audit log (audit.view) with filters and CSV export (audit.export).
Settings (settings.manage) toggles. Integrations (integrations.manage): API keys
(create shows secret once, revoke), webhooks CRUD, test, deliveries, redeliver.
```

---

## Acceptance criteria

```
- [ ] Every admin endpoint has a screen; destructive actions confirm first.
- [ ] Backend rule errors (LAST_ADMINISTRATOR, ROLE_IS_SYSTEM, ...) are shown clearly. en/ar + RTL.
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
