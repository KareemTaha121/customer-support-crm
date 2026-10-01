# Story intake

- Folder: `.squad/stories/security-and-administration/user-data-access-warning/intake.md`

---

## Feature

- **Feature name (display):** Security & Administration
- **Feature slug (folder under `plans/`):** `security-and-administration`

## Tracker (metadata only)

- **Tracker type:** `none`
- **Work item id:** `BUG-23`
- **Work item type:** `Bug`
- **Status:** `Done`
- **Assignee:** ``
- **Labels:** `backend`, `frontend`, `bug`, `users`, `scopes`

---

## Title

```
Warn when a staff user would see no data, and tell the user why their workspace is empty
```

---

## Description

```
QA round 2 (N3). A user created with only the "Agent" role and no branch scope signs in
to an empty app: GET /tickets and GET /customers return totalCount 0 and nothing says why.
The New user dialog only has the hint "Leave empty when a role grants access to all
branches" and never checks whether a selected role grants data.all_branches
(user-dialog.component.ts, createUser call ~262). GET /auth/me has no scope information,
so the web app cannot explain the empty state.

Fix:
- API: CurrentUserResponse.HasDataAccess = data.all_branches or at least one scope;
  returned by login, refresh and /auth/me.
- Web: the user dialog warns when roles are selected, none grants data.all_branches and
  no scope is set; the dashboard shows a notice when hasDataAccess is false.
```

---

## Acceptance criteria

```
- [x] /auth/me and login return hasDataAccess (true for admin and scoped agents, false for an unscoped agent).
- [x] The user dialog shows the warning only for "roles without all-branches and no scope".
- [x] The dashboard shows the notice for an unscoped user (en/ar).
- [x] dotnet build 0 warnings; npx ng build 0 errors, 0 warnings.
```

---

## Attachments

None.

---

## Dependencies

- **Blocked by / related ids:** none
- **Depends on code areas or other stories:** found in the round 2 manual QA pass, [../../../qa/2026-10-01-manual-qa-report-round2.md](../../../qa/2026-10-01-manual-qa-report-round2.md).

## Technical hints (optional)

- Repos: `customer-support-crm-api/` (`develop`), `customer-support-crm-web/` (`main`). Scope rules: `AccessScopeProvider` (`Common/Authorization/AccessScope*.cs`).

## Out of scope

- Do NOT touch Docker, docker-compose.yml, deploy/ or .github/ (CI/CD).
- Do NOT create or modify anything under tests/ or e2e/.
- Empty states on the ticket and customer lists (the dashboard is the landing page).
