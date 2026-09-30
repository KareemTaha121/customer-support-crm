# Story intake

- Folder: `.squad/stories/frontend/customers-ui/intake.md`

---

## Feature

- **Feature name (display):** Frontend — Angular web app for all 12 features
- **Feature slug (folder under `plans/`):** `frontend`

## Tracker (metadata only)

- **Tracker type:** `none`
- **Work item id:** `FE-03`
- **Work item type:** `Story`
- **Status:** `Ready`
- **Assignee:** ``
- **Labels:** `frontend`, `customers`

---

## Title

```
Customers UI (feature 01)
```

---

## Description

```
features/customers: list with search (name/phone/email/number), paging and
sorting; create/edit form with duplicate check (GET /customers/duplicates);
details page with tabs Profile, Contacts (add/edit/remove/set primary), Notes
(internal, CRUD), Attachments (upload/download/delete), History (paged timeline
GET /customers/{id}/history); portal access grant/revoke; soft delete.
Permissions: customers.view/create/update/delete, customers.notes_manage,
customers.attachments_manage.
```

---

## Acceptance criteria

```
- [ ] All customer endpoints in Application/Features/Customers are reachable from the UI.
- [ ] Server validation errors show on the right field; duplicates are warned before save.
- [ ] Actions are hidden without the matching permission. en/ar + RTL.
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
