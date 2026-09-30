# 10 — Security & Administration

> **Source:** AZM Squad Customer Support CRM — Core Features §10
> **Implementation phase:** Phase 2 — Identity & Authorization (+ Phase 3 — Organization Context)
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
