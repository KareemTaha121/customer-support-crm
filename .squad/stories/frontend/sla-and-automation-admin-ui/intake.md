# Story intake

- Folder: `.squad/stories/frontend/sla-and-automation-admin-ui/intake.md`

---

## Feature

- **Feature name (display):** Frontend — Angular web app for all 12 features
- **Feature slug (folder under `plans/`):** `frontend`

## Tracker (metadata only)

- **Tracker type:** `none`
- **Work item id:** `FE-07`
- **Work item type:** `Story`
- **Status:** `Ready`
- **Assignee:** ``
- **Labels:** `frontend`, `sla`, `automation`

---

## Title

```
SLA and automation admin UI (feature 05)
```

---

## Description

```
Admin screens: SLA policies (sla.manage) CRUD, assignment rules and
escalation rules (automation.manage) CRUD under /sla-policies and
/automation/*.
```

---

## Acceptance criteria

```
- [ ] SLA policies, assignment rules and escalation rules can be listed, created, edited and deleted.
- [ ] Validation errors are shown per field. en/ar + RTL.
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
