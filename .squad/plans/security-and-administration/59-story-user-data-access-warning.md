# Story 59 — Warn when a staff user would see no data (Bug: BUG-23)

> Fix plan, implemented in `customer-support-crm-api` commit `613e607` (`develop`) and `customer-support-crm-web` commit `6652669` (`main`). Paths and line numbers refer to those commits.
> Intake: [../../stories/security-and-administration/user-data-access-warning/intake.md](../../stories/security-and-administration/user-data-access-warning/intake.md)

## Prerequisites

- Story 03 ([03-story-current-user-and-permission-authorization.md](03-story-current-user-and-permission-authorization.md)): `GET /auth/me`, `UserAccessProfile`.
- Platform story 21 ([../platform/21-story-organization-context-and-notifications.md](../platform/21-story-organization-context-and-notifications.md)): branch/department scopes and `AccessScopeProvider`.
- Frontend story 19 ([../frontend/19-story-administration-ui.md](../frontend/19-story-administration-ui.md)): the user dialog. Frontend story 13: the dashboard.

---

## Story Goal

A user's data scope is `AccessScope.Everything` with `data.all_branches`, otherwise the user's `UserScope` rows (`AccessScopeProvider.LoadAsync`). With neither, every ticket and customer query returns nothing. That is a safe default, but before this fix nothing told the admin or the user.

| Where | Before | After |
|---|---|---|
| `CurrentUserResponse` (login, refresh, `/auth/me`) | Id, email, name, roles, permissions | Also `HasDataAccess` |
| User dialog | Hint "Leave empty when a role grants access to all branches" only | Warning when roles are selected, none grants `data.all_branches`, and no scope is set |
| Dashboard | Empty KPIs and lists, no explanation | Notice: "Your account has no branch or department access yet…" |

**Deviation from the intake:** none. The ticket and customer list empty states are unchanged (out of scope); the dashboard is the landing page after sign-in.

---

## Context — Read These Files First

1. API `src/CustomerSupportCrm.Application/Features/Authentication/Common/UserAccessProfile.cs`: record at 11, `hasDataAccess` at 23–24.
2. API `src/CustomerSupportCrm.Contracts/Authentication/AuthenticationContracts.cs`: `CurrentUserResponse` (13–23).
3. API `Features/Authentication/Common/UserSessionService.cs:70` (login/refresh) and `GetCurrentUser/GetCurrentUserHandler.cs:34` (`/auth/me`).
4. Web `src/app/core/auth/auth.models.ts:9`: `hasDataAccess` on `CurrentUser`.
5. Web `src/app/features/administration/users/user-dialog.component.ts`: template 115–117, `noDataAccess` 213–217, styles 169–170.
6. Web `src/app/features/dashboard/dashboard.page.ts:83`, `dashboard.page.html` 40–45, `dashboard.page.scss` 110–118.

---

## Backend Tasks

1. `UserAccessProfile` gets a third member, `HasDataAccess`:
   ```csharp
   var hasDataAccess = permissions.Contains(Domain.Roles.Permissions.DataAllBranches, StringComparer.Ordinal)
       || await db.Users.AnyAsync(user => user.Id == userId && user.Scopes.Any(), cancellationToken);
   ```
   The `Any` query runs only for users without `data.all_branches`.
2. `CurrentUserResponse` gets `bool HasDataAccess` as its last parameter (XML doc on the record). Both builders pass `profile.HasDataAccess`.

The JWT is unchanged, and so is the access rule itself.

---

## Frontend Tasks

1. `CurrentUser.hasDataAccess: boolean`.
2. Dashboard: `noDataScope = computed(() => auth.currentUser()?.hasDataAccess === false)`. The strict `false` check means a session restored before this field existed shows nothing. A `.scope-warning` card (`domain_disabled` icon, warning border) goes above the KPIs.
3. User dialog:
   ```ts
   readonly noDataAccess = computed(() => {
     const selected = this.selectedRoles();
     const allBranches = this.data.roles.some((role) => selected.has(role.id) && role.permissions.includes(Permissions.dataAllBranches));
     return selected.size > 0 && !allBranches && this.scopes().length === 0;
   });
   ```
   Shown under the scopes hint with `role="status"`. It does not block saving, because scopes can be added later.
4. i18n: `dashboard.home.noDataScope` and `admin.users.noScopeWarning` (en + ar).

---

## Test Plan

Test projects are out of scope.

---

## Verification Steps

1. `dotnet build src/CustomerSupportCrm.Api/CustomerSupportCrm.Api.csproj -o <temp>`: 0 warnings, 0 errors. `npx ng build`: 0 errors, 0 warnings.
2. API: admin login gives `hasDataAccess: true`; agent with a Head Office scope gives `true`. After `PUT /users/{id}/scopes {"scopes":[]}` the agent's `/auth/me` gives `false`.
3. Web, as the unscoped agent: the dashboard shows the notice.
4. Web, New user dialog: no roles → no warning; Agent only → warning; add a scope → gone; remove it → back; Agent plus Administrator → gone.

All passed on 2026-10-01.

---

## Done Criteria

- [x] `hasDataAccess` is returned by login, refresh and `/auth/me` with the right value.
- [x] The user dialog warns only for roles without all-branches and no scope.
- [x] The dashboard explains an empty workspace (en/ar).
- [x] Nothing changed in `tests/`, Docker, `deploy/` or `.github/`.
- [x] `dotnet build` 0 warnings; `npx ng build` 0 errors, 0 warnings.

**STOP HERE. Report to the user.**
