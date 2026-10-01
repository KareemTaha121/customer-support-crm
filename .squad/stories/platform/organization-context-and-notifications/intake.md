# Story intake

- Folder: `.squad/stories/platform/organization-context-and-notifications/intake.md`

---

## Feature

- **Feature name (display):** Platform — Phase 3 Organization Context
- **Feature slug (folder under `plans/`):** `platform`

## Tracker (metadata only)

- **Tracker type:** `none`
- **Work item id:** `PL-02`
- **Work item type:** `Story`
- **Status:** `Done`
- **Assignee:** ``
- **Labels:** `phase-3`, `backend`, `platform`

---

## Title

```
Organization context, branding and notifications
```

---

## Description

```
The Organization -> Branch -> Department model, per-user branch/department scopes, branding,
in-app notifications with realtime push, and the shared platform services later features
use (domain events, soft delete, file storage, recurring jobs, endpoint groups, the full
permission catalog). customer-support-crm-api, commit 65c74a3.

Endpoint groups:
- /api/v1           staff (policy:staff = authenticated + actor=staff)
- /api/v1/portal    customers (policy:customer)
- /api/v1/public    anonymous
- /api/v1/external  API keys (policy:api-client)
- /hubs/staff       SignalR, staff policy; /hubs/chat SignalR, anonymous + per-conversation token

Endpoints and permissions:
- GET  /organization                                 staff        Profile + branding + logoUrl
- PUT  /organization                                 settings.manage  {name, supportEmail, supportPhone, defaultCulture (en|ar), timeZone}
- PUT  /organization/branding                        settings.manage  {primaryColor, accentColor} (#RRGGBB)
- POST /organization/logo                            settings.manage  multipart "file"; png/jpg/jpeg/gif/webp, max 2 MB, magic bytes checked
- GET  /public/branding                              anonymous    {name, defaultCulture, primaryColor, accentColor, logoUrl}
- GET  /public/branding/logo                         anonymous    logo bytes, Cache-Control public 1 day
- GET  /branches?includeInactive=                    staff        Branches with their departments (active only by default)
- POST /branches                                     organization.manage  {code, name, address, phone}
- PUT  /branches/{id}                                organization.manage
- POST /branches/{id}/activate | /deactivate         organization.manage
- POST /branches/{branchId}/departments              organization.manage  {code, name, email}
- PUT  /branches/{branchId}/departments/{id}         organization.manage
- POST /departments/{id}/activate | /deactivate      organization.manage
- PUT  /users/{id}                                   users.manage  {displayName}
- PUT  /users/{id}/scopes                            users.manage  {scopes: [{branchId, departmentId?}]} (replace set)
- POST /users/{id}/reset-password                    users.manage  {newPassword}; revokes all the user's sessions
- GET  /users/lookup?search=&departmentId=&permission=  staff     Active users for pickers (max 50)
- POST /users                                        users.manage  now also accepts optional scopes[]
- GET  /notifications?page=&pageSize=&unreadOnly=    staff (own)  Paged, newest first
- GET  /notifications/unread-count                   staff (own)
- POST /notifications/{id}/read                      staff (own)  Returns the new unread count
- POST /notifications/read-all                       staff (own)

Rules:
- One organization per deployment, created by database initialization with a HQ branch and a
  SUPPORT department -> 404 ORGANIZATION_NOT_CONFIGURED if missing.
- Branch code unique (upper-cased) -> 409 BRANCH_CODE_TAKEN; department code unique within its
  branch -> 409 DEPARTMENT_CODE_TAKEN; unknown -> 404 BRANCH_NOT_FOUND / DEPARTMENT_NOT_FOUND.
- Branches and departments are deactivated, never deleted.
- Scope rows: departmentId null = the whole branch. Unknown branch or department not in its
  branch -> 400 DEPARTMENT_NOT_IN_BRANCH on "scopes".
- Data access: data.all_branches sees everything; otherwise reads are filtered to the user's
  scopes (out-of-scope records return 404) and writes into a unit the user does not own
  return 403 OUT_OF_SCOPE.
- Domain invariants -> 422 INVALID_ORGANIZATION / INVALID_BRANCH / INVALID_DEPARTMENT.
- Upload rules -> 400 FILE_EMPTY / FILE_TOO_LARGE / FILE_TYPE_NOT_ALLOWED / FILE_SIGNATURE_MISMATCH.
- Every organization, branch, department and user-administration change is audited.
```

---

## Acceptance criteria

```
- [ ] Organization -> Branch -> Department context model; every business entity declares ownership; no single-branch assumption.
- [ ] As an admin, I set up branches and departments and assign users to them.
- [ ] As an admin, I restrict a user to specific branches/departments (context-aware authorization).
- [ ] Branding settings (logo, colors, name) exposed for the agent app, portal and emails.
- [ ] Soft delete and auditing conventions.
- [ ] Background jobs infrastructure.
- [ ] In-app notifications with realtime push.
- [ ] All new messages localized (en/ar).
- [ ] `dotnet build` passes with zero warnings.
```

---

## Attachments

None.

---

## Dependencies

- **Blocked by / related ids:** PL-01, P2-01 … P2-07
- **Depends on code areas or other stories:** `IAuditTrail`, `ICurrentUser`, permission policies, `UserQueries`, `UserSessionService`, `ApiResults`, `PagedResult`.

## Extra notes (optional)

- Feature specs: `features/12-platform.md` (multi-department, multi-branch, branding) and `features/10-security-and-administration.md` (branches/departments CRUD, user scopes).
- Frontend: `plans/frontend/08-story-core-platform-shell.md` (branding theme loader, notifications) and `plans/frontend/19-story-administration-ui.md` (organization screens).

## Technical hints (optional)

- Repo root: `customer-support-crm-api/`. .NET 10.

## Out of scope

- Do NOT touch Docker, docker-compose.yml, deploy/ or .github/ (CI/CD).
- Do NOT create or modify anything under tests/.
- System settings and audit export (P3-01, Story 22).
- Feature-specific scoped entities (customers, tickets) — their own features.
