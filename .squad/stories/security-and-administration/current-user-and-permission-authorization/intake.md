# Story intake

- Folder: `.squad/stories/security-and-administration/current-user-and-permission-authorization/intake.md`

---

## Feature

- **Feature name (display):** Security & Administration — Phase 2 Identity & Authorization
- **Feature slug (folder under `plans/`):** `security-and-administration`

## Tracker (metadata only)

- **Tracker type:** `none`
- **Work item id:** `P2-03`
- **Work item type:** `Story`
- **Status:** `Ready`
- **Assignee:** ``
- **Labels:** `phase-2`, `backend`, `authorization`

---

## Title

```
Current user abstraction and permission-based authorization
```

---

## Description

```
Give the Application layer a framework-free view of the caller and enforce
permission-based authorization on every endpoint (customer-support-crm-api).

ICurrentUser (Application/Abstractions/Authentication/):
- IsAuthenticated, UserId (Guid?), Email, FullName, Roles, Permissions,
  Culture, HasPermission(code). Application code must never use
  HttpContext.User directly.
- Implementation HttpContextCurrentUser in Api (or Infrastructure) reading the
  JWT claims from P2-02 (sub, email, name, role, perm, culture). Registered
  scoped.

Authorization (Api/Authorization + Application/Abstractions/Authorization):
- Permission policies named "perm:<code>" created on demand by a custom
  IAuthorizationPolicyProvider; PermissionAuthorizationHandler checks the
  "perm" claims. No role-name checks anywhere.
- Endpoint extension in Application/Abstractions/Http:
  `builder.RequirePermission(Permissions.Users.Manage)`.
- Secure by default: the /api/v1 group requires an authenticated user
  (RequireAuthorization on the group in EndpointExtensions.MapApiEndpoints);
  anonymous endpoints (login, refresh) opt out with AllowAnonymous.
- 401 and 403 responses must use the standard ApiResponse envelope with
  UNAUTHORIZED / FORBIDDEN (JwtBearer OnChallenge / OnForbidden or the existing
  StatusCodePages writer — make sure the body is not empty and not duplicated).
- ForbiddenException thrown by handlers keeps mapping to 403.

New slice (Application/Features/Authentication/Me):
- GET /api/v1/auth/me -> current user summary (id, email, fullName, culture,
  roles, permissions) loaded from the database, 401 if user no longer active.

Permission freshness: permissions are embedded in the access token and refresh
on the next token refresh (<= AccessTokenMinutes). Document this trade-off in
docs/architecture.md. Refresh (P2-02) must re-read roles/permissions from the DB.

OpenAPI: add the Bearer security scheme so Swagger UI has an Authorize button,
and mark anonymous endpoints accordingly.
```

---

## Acceptance criteria

```
- [ ] ICurrentUser is available in handlers; no Application code references HttpContext.User.
- [ ] Every /api/v1 endpoint requires authentication unless explicitly AllowAnonymous.
- [ ] RequirePermission(code) returns 403 FORBIDDEN envelope when the permission is missing.
- [ ] Missing/invalid/expired token returns 401 UNAUTHORIZED envelope with correlationId.
- [ ] GET /api/v1/auth/me returns the caller's profile, roles and permissions.
- [ ] Swagger UI can authorize with a bearer token.
- [ ] `dotnet build` passes with zero warnings.
```

---

## Attachments

None.

---

## Dependencies

- **Blocked by / related ids:** P2-01, P2-02
- **Depends on code areas or other stories:** Permissions catalog (P2-01), JWT claims (P2-02), ErrorResponseWriter / StatusCodePages and OpenApiExtensions (Phase 1).

## Extra notes (optional)

- Implementation plan §21.3, §22.
- OrganizationId / BranchId / DepartmentIds on ICurrentUser come in Phase 3 — design the interface so they can be added without breaking callers.

## Technical hints (optional)

- Repo root: `customer-support-crm-api/`. .NET 10.

## Out of scope

- Do NOT touch Docker, docker-compose.yml, deploy/ or .github/ (CI/CD).
- Do NOT create or modify anything under tests/.
- Resource / branch / department scoped authorization (Phase 3).
- User and role management endpoints (P2-04, P2-05).
