# Story 19 — Administration UI (Story: FE-12)

## Prerequisites

- Story 08 completed: [08-story-core-platform-shell.md](08-story-core-platform-shell.md) (`ApiService`, `ApiError`, i18n, permissions, guards, shared components, sidenav entries under `/admin/*`).
- Story 09 completed: [09-story-authentication-staff-and-portal.md](09-story-authentication-staff-and-portal.md) (`AuthService.reloadCurrentUser`, the `profile.page.ts` form pattern this story copies: `{ silent: true }` + `applyServerErrors` + `describeError`).
- Conventions and ownership: [00-overview.md](00-overview.md). This story edits only `customer-support-crm-web/src/app/features/administration/**` and `customer-support-crm-web/public/i18n/admin/{en,ar}.json`.

---

## Story Goal

1. **Users** (`/admin/users`, `users.manage`): paged, sortable list with search and status filter; create (password 12–128), edit display name, assign roles, set branch/department scopes, reset password, activate/deactivate with confirmation.
2. **Roles** (`/admin/roles`, `roles.manage`): list with user counts; create/edit with a permission checklist grouped by feature (`GET /permissions`); delete with confirmation; system roles are read-only.
3. **Organization** (`/admin/organization`, `organization.manage` or `settings.manage`): profile and branding colors + logo upload (writes need `settings.manage`), branches with departments (writes need `organization.manage`). Saving branding re-applies it live via `BrandingService.apply`.
4. **Settings** (`/admin/settings`, `settings.manage`): toggles/number/text inputs typed by `SettingResponse.kind`.
5. **Integrations** (`/admin/integrations`, `integrations.manage`): API keys (secret shown once with copy, revoke), webhooks CRUD (signing secret shown once, rotate), send test, deliveries list with retry.
6. **Audit log** (`/admin/audit`, `audit.view`): filters (date range, user, action, entity type/id), paging, detail dialog with old/new values JSON, CSV export (`audit.export`).
7. Backend rule errors (`LAST_ADMINISTRATOR`, `CANNOT_DISABLE_SELF`, `ROLE_IS_SYSTEM`, `ROLE_IN_USE`, `EMAIL_TAKEN`, `ROLE_NAME_TAKEN`, `BRANCH_CODE_TAKEN`, ...) show localized messages keyed by code. Full en/ar, RTL-safe.

Not in scope: backend changes, tests. `GET /users` has no role filter (see Edge Cases).

---

## Context — Read These Files First

1. `customer-support-crm-web/src/app/app.routes.ts` line 35 — `admin` lazy-loads `ADMINISTRATION_ROUTES` (export name fixed).
2. `customer-support-crm-web/src/app/core/layout/navigation.ts` lines 38–48 — sidenav links and their permissions for the six child routes.
3. `customer-support-crm-web/src/app/core/http/api.service.ts` — `get`/`getPaged`/`post`/`put`/`delete`, `upload` (~line 68, field `file`), `download` (~line 78) returning `{ blob, fileName }`.
4. `customer-support-crm-web/src/app/core/http/api-error.ts` — `ApiError.from`, `hasCode`, `isValidation`.
5. `customer-support-crm-web/src/app/core/interceptors/error.interceptor.ts` line 33 — `describeError(error, translations)`.
6. `customer-support-crm-web/src/app/shared/form-errors.ts` (`applyServerErrors`), `form-error.pipe.ts` (`formError`), `password.ts` (`passwordValidators`, `PASSWORD_MIN_LENGTH`), `file-utils.ts` (`saveBlob`), `confirm-dialog.component.ts` (`ConfirmService.ask`), `page-header.component.ts`, `state.components.ts`.
7. `customer-support-crm-web/src/app/core/permissions/permission.service.ts` (`has`, `hasAny`) and `core/guards/auth.guards.ts` (`requirePermission(...codes)` = any-of).
8. `customer-support-crm-web/src/app/core/auth/auth.service.ts` line 84 — `reloadCurrentUser()` (call after changing the signed-in user's roles or editing a role they hold).
9. `customer-support-crm-web/src/app/core/branding/branding.service.ts` line 55 — `apply(PublicBranding)`.
10. `customer-support-crm-web/src/app/features/auth/profile.page.ts` — reference form/error handling.
11. Backend users: `customer-support-crm-api/src/CustomerSupportCrm.Application/Features/Users/List/ListUsersQuery.cs` (search, status `Active|Disabled`, sortBy `displayName|email|createdAt|lastLoginAt`), `Create/CreateUserEndpoint.cs`, `SetRoles/SetUserRolesEndpoint.cs`, `Enable/*`, `Disable/DisableUserHandler.cs` (lines 23–46: `CANNOT_DISABLE_SELF`, last admin check), `Administration/UserAdministration.cs` lines 141–172 (`PUT /users/{id}`, `PUT /users/{id}/scopes`, `POST /users/{id}/reset-password`, `GET /users/lookup`), `Common/UserErrors.cs`.
12. `customer-support-crm-api/src/CustomerSupportCrm.Contracts/Users/UserContracts.cs` — `UserListItemResponse`, `UserResponse`, `CreateUserRequest`, `UserScopeRequest`, `UserLookupResponse`.
13. Backend roles: `Features/Roles/Common/RoleQueries.cs` (`RoleErrors`), `ListPermissions/ListPermissionsEndpoint.cs` (`GET /permissions` → `{ code, group }`), `Delete/DeleteRoleHandler.cs`; `Contracts/Roles/RoleContracts.cs`; `Domain/Roles/Role.cs` line 16 `ROLE_IS_SYSTEM`.
14. `Features/Organization/OrganizationProfile.cs` lines 134–165 — `GET /organization`, `PUT /organization`, `PUT /organization/branding`, `POST /organization/logo` (all writes `settings.manage`); cultures `en|ar`; colors `^#[0-9A-Fa-f]{6}$`; logo ≤ 2 MB png/jpg/gif/webp (`Common/Files/FileUploadRules.cs`).
15. `Features/Branches/BranchEndpoints.cs` lines 121–159 (list `?includeInactive=true`, create/update/activate/deactivate, `organization.manage`) and `Features/Departments/DepartmentEndpoints.cs` ~lines 98–125 (`POST/PUT /branches/{branchId}/departments[/{id}]`, `/departments/{id}/activate|deactivate`); codes in `Common/Authorization/AccessScope.cs` lines 112–121.
16. `Features/AuditLogs/List/ListAuditLogsQuery.cs` (action, entityType, entityId, actorUserId, from, to) and `ExportAuditLogs.cs` (`GET /audit-logs/export.csv?from&to&action`, `audit.export`); `Contracts/Audit/AuditContracts.cs`.
17. `Features/Settings/SettingsSlices.cs` lines 4–6, 52–104 — `SettingResponse(key, kind, value, defaultValue, isPublic)`, `PUT /settings { values }`; kinds `Boolean|Number(0..3650)|Text(≤2000)`.
18. `Features/Integrations/IntegrationSlices.cs` lines 23–39 (records) and 55–170 (catalog, api-keys, webhooks, rotate-secret, test, deliveries, retry).
19. `customer-support-crm-api/docs/api-contract.md` "Feature error codes (Phase 2)".

---

## Frontend Tasks

All paths below are under `customer-support-crm-web/src/app/features/administration/`.

### 1 — Models and API

- Create file: `administration.models.ts` — interfaces mirroring the records above (`UserListItem`, `UserDetail`, `RoleResponse`, `PermissionResponse`, `OrganizationResponse`, `BranchResponse`, `DepartmentResponse`, `AuditLogResponse` with `oldValues/newValues: unknown`, `SettingResponse`, `ApiKeyResponse`, `CreatedApiKeyResponse`, `WebhookResponse`, `WebhookWithSecretResponse`, `WebhookDeliveryResponse`, `IntegrationCatalog`), plus `AdminErrorCodes`.
- Create file: `administration.api.ts` — `AdministrationApi` (`providedIn: 'root'`) with one method per endpoint; writes accept `silent` where forms render errors.
- Create file: `admin-errors.ts` — `adminErrorMessage(error, translations)`: returns `admin.errors.<CODE>` when translated for any code in `error.errors`, else `describeError`.

### 2 — Routes

- File: `administration.routes.ts` — `ADMINISTRATION_ROUTES`: root `{ path: '', resolve: { i18n: translationResolver('admin') }, children }`; children `users`, `roles`, `organization` (`requirePermission(organizationManage, settingsManage)`), `settings`, `integrations`, `audit`, each with `requirePermission` and lazy `loadComponent`; `''` uses `redirectTo: () => …` computing the first accessible page with `PermissionService` (fallback `/forbidden`).

### 3 — Pages and dialogs

- `users/users.page.ts` + `users/user-dialog.component.ts` (create/edit: name, email, password on create, role checkboxes, scopes rows, reset password section on edit).
- `roles/roles.page.ts` + `roles/role-dialog.component.ts` (grouped permission checklist with per-group "select all"; read-only when `isSystem`).
- `organization/organization.page.ts` (profile + branding/logo cards) + `organization/branches.component.ts` + `organization/unit-dialog.component.ts` (branch or department form).
- `settings/settings.page.ts`.
- `integrations/integrations.page.ts` + `integrations/api-key-dialog.component.ts` + `integrations/webhook-dialog.component.ts` + `integrations/secret-dialog.component.ts` (shows a secret once, copy via `navigator.clipboard`) + `integrations/deliveries-dialog.component.ts`.
- `audit/audit.page.ts` + `audit/audit-detail-dialog.component.ts` (pretty-printed JSON in `<pre dir="ltr">`).
- `admin.scss` shared styles kept small; component styles < 8 kB.

### 4 — i18n

- Create `customer-support-crm-web/public/i18n/admin/en.json` and `ar.json` with identical keys: `users`, `roles`, `organization`, `settings`, `integrations`, `audit`, `status`, `permissionGroups`, `errors.<CODE>`.

---

## Edge Cases & Failure Modes

- `GET /users` has no role filter: the list filters by status only; roles are shown as chips. Documented gap.
- Disabling yourself → `CANNOT_DISABLE_SELF`; disabling/unassigning the last admin → `LAST_ADMINISTRATOR`; both show localized snackbar text and the row is left unchanged.
- Changing roles of the signed-in user or editing a role they hold → `AuthService.reloadCurrentUser()` so the sidenav and guards update.
- `ROLE_IS_SYSTEM` (422) for system roles: the UI hides edit/delete, and still maps the code if the API returns it.
- `ROLE_IN_USE` on delete → localized message suggesting to unassign users first.
- Organization page with `organization.manage` only: profile/branding are read-only (writes need `settings.manage`); with `settings.manage` only: branches are read-only.
- API key and webhook secrets are returned once; the secret dialog cannot be dismissed by backdrop click and warns the value will not be shown again.
- Audit CSV export only supports `from`, `to`, `action`; other filters are not applied to the file (hint text says so).
- Audit `oldValues/newValues` may be null → "none".

## Test Plan

Out of scope (build-level verification only). Manual smoke: create user, assign roles, disable self (error), delete role in use (error), change brand color (applies live), create API key (secret once), test webhook then view deliveries, filter audit and export.

## Verification Steps

1. **Frontend builds:** `cd customer-support-crm-web && npx ng build` with no errors or warnings in `features/administration/**`.

## Done Criteria

- [ ] Every admin endpoint listed above has a screen; destructive actions confirm first.
- [ ] Rule errors show localized messages keyed by code.
- [ ] en/ar translations with identical keys; RTL-safe styles.
