# Story intake

- Folder: `.squad/stories/frontend/knowledge-base-ui/intake.md`

---

## Feature

- **Feature name (display):** Frontend — Angular web app for all 12 features
- **Feature slug (folder under `plans/`):** `frontend`

## Tracker (metadata only)

- **Tracker type:** `none`
- **Work item id:** `FE-08`
- **Work item type:** `Story`
- **Status:** `Done`
- **Assignee:** ``
- **Labels:** `frontend`, `kb`

---

## Title

```
Knowledge base UI (feature 06)
```

---

## Description

```
Staff: KB categories and articles (kb.view, kb.manage, kb.publish) with
search, editor (title, body, category, visibility/status per backend), delete.
Public: FAQ/help center pages under /help using /public/kb (categories,
articles, article by slug, feedback helpful/not).
```

---

## Acceptance criteria

```
- [ ] Staff can manage articles and categories; publish requires kb.publish.
- [ ] Anonymous users can browse/search published articles and give feedback. en/ar + RTL.
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
