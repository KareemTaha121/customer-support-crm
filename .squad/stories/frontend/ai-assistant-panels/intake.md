# Story intake

- Folder: `.squad/stories/frontend/ai-assistant-panels/intake.md`

---

## Feature

- **Feature name (display):** Frontend — Angular web app for all 12 features
- **Feature slug (folder under `plans/`):** `frontend`

## Tracker (metadata only)

- **Tracker type:** `none`
- **Work item id:** `FE-09`
- **Work item type:** `Story`
- **Status:** `Ready`
- **Assignee:** ``
- **Labels:** `frontend`, `ai`

---

## Title

```
AI assistant panels (feature 07)
```

---

## Description

```
Reusable AI panel embedded in ticket details (ai.use, gated by GET /ai/status):
summary, suggested reply (insert into reply box), categorize (apply category/
priority), suggested solutions (KB links), and feedback on suggestions
(POST /ai/suggestions/{id}/feedback).
```

---

## Acceptance criteria

```
- [ ] Panel hidden when AI is disabled or the user lacks ai.use.
- [ ] Each action shows loading/error states and feedback can be sent. en/ar + RTL.
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
