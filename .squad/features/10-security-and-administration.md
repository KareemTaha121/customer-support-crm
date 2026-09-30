# 10 — Security & Administration

> **Source:** AZM Squad Customer Support CRM — Core Features §10
> **Implementation phase:** Phase 2 — Identity & Authorization (+ Phase 3 — Organization Context)
> **Status:** Phase 2 in progress — see [Phase 2 delivery plan](#phase-2--identity--authorization-delivery-plan)
> **Build priority:** 2–3

## Summary

Control who can use the system and what they can do, keep a tamper-evident record of important actions, and let administrators configure the platform without code changes.

## Scope

| Sub-feature | Description |
|-------------|-------------|
| Users and roles | Manage internal users (agents, supervisors, admins) and roles. |
| Permissions | Granular permissions assigned to roles; scoped by branch/department. |
| Audit logs | Who did what, when, from where; searchable and exportable. |
| System configuration | Organization settings, branches, departments, lookups, feature toggles. |

## User stories

- As an **admin**, I invite/create users, assign roles, and deactivate users.
- As an **admin**, I create roles and assign permissions to them.
- As an **admin**, I restrict a user to specific branches/departments.
- As an **auditor**, I search the audit log by user, entity, action and date.
- As an **admin**, I manage branches, departments and system settings.

## Acceptance criteria

- [ ] Authentication: JWT access tokens + rotating refresh tokens; secure password hashing and policy; account lockout.
- [ ] Optional SSO / MFA ready (design allows adding later).
- [ ] Permission-based authorization policies (not role-name checks in code).
- [ ] Context-aware authorization: Organization → Branch → Department.
- [ ] `ICurrentUser` abstraction available to Application layer.
- [ ] Audit log captures create/update/delete and security events (login, permission change) with old/new values and correlation id.
- [ ] Audit logs are append-only.
- [ ] Rate limiting, CORS and CSRF protections configured.
- [ ] System configuration changes are audited.

## Domain model

```text
User, Role, Permission, UserRole, RolePermission
RefreshToken
Organization, Branch, Department, UserScope
AuditLog
SystemSetting
```

## Backend slices

```text
Features/Identity/ (Login, Refresh, Logout, ChangePassword, ResetPassword)
Features/Users/ (CRUD, AssignRoles, AssignScopes, Deactivate)
Features/Roles/ (CRUD, AssignPermissions)
Features/Organization/ (Branches CRUD, Departments CRUD)
Features/AuditLogs/ (Search, Export)
Features/Settings/ (Get, Update)
```

## Frontend

- Login, forgot/reset password.
- Admin area: users, roles & permissions matrix, branches/departments, audit log viewer, settings.
- Route guards and permission directives.

## Dependencies

- Platform (12). Required by all other features.

---

## Phase 2 — Identity & Authorization (delivery plan)

Implementation plan §21–24, §13 and §81 (Phase 2). Builds on the Phase 1 platform in `customer-support-crm-api` (MediatR + validation pipeline, `ApiResponse` envelope, `GlobalExceptionHandler`, correlation id, Serilog, en/ar localization, EF Core/PostgreSQL).

### In scope

- Users, roles, permissions (`UserRole`, `RolePermission`) and a code-defined permission catalog (`tickets.view`, `users.manage`, …).
- Password hashing (ASP.NET Core Identity `PasswordHasher<T>`), password policy, account lockout.
- JWT access tokens (short-lived) + refresh tokens (rotated, revocable, stored hashed, reuse detection).
- `ICurrentUser` abstraction in Application, adapted from HTTP claims in Api/Infrastructure.
- Permission-based authorization policies (`RequirePermission("users.manage")`), never role-name checks.
- Append-only audit log for entity changes and security events (login, logout, failed login, lockout, role/permission change).
- Rate limiting on auth endpoints, strict CORS for known frontend origins.
- Bootstrap admin user + default roles seeded from configuration (no hard-coded credentials).

### Out of scope for Phase 2

- **Docker, docker-compose, deployment and CI/CD changes** — do not touch `docker-compose.yml`, `deploy/`, `.github/`.
- **Test projects** — do not create or modify files under `tests/`.
- Organization / Branch / Department and scoped authorization → Phase 3.
- System settings, audit export → Phase 3.
- SSO / MFA (design must allow adding later), email-based password reset (needs Email provider).
- Frontend (login screen, admin UI, guards) → after the web foundation.

### Stories (execution order)

| NN | Story | Slices / output |
|----|-------|-----------------|
| 01 | Identity domain & persistence | `Domain/Users`: `User`, `Role`, `Permission` catalog, `UserRole`, `RolePermission`, `RefreshToken`; EF configurations; first migration; seed default roles + bootstrap admin |
| 02 | Authentication (JWT + refresh) | `JwtOptions`, token service, password hasher, lockout; `Features/Authentication`: Login, Refresh, Logout; JwtBearer wiring |
| 03 | Current user & permission authorization | `ICurrentUser`, claims adapter, permission policies, `RequirePermission` endpoint extension, 401/403 envelope |
| 04 | User management | `Features/Users`: Create, Get, List (paged), Update, AssignRoles, Deactivate/Activate, ChangePassword (self) |
| 05 | Role & permission management | `Features/Roles`: CRUD, AssignPermissions; `Features/Permissions`: List catalog |
| 06 | Audit logging | `Domain/Audit/AuditLog` (append-only), SaveChanges interceptor (old/new values, redaction), security-event auditing, `Features/AuditLogs`: Search (paged) |
| 07 | Security hardening | Rate limiting (login/refresh), CORS options per environment, secure headers, request size limits |

### Definition of done (Phase 2)

- `dotnet build` passes with zero warnings; existing tests still pass.
- All new endpoints are under `/api/v1`, return the standard envelope, and are documented in OpenAPI.
- Every user-facing message exists in `Messages.resx` and `Messages.ar.resx`.
- No password, token or secret is ever logged or audited.
