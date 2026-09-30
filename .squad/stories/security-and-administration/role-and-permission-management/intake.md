# Story intake

- Folder: `.squad/stories/security-and-administration/role-and-permission-management/intake.md`

---

## Feature

- **Feature name (display):** Security & Administration — Phase 2 Identity & Authorization
- **Feature slug (folder under `plans/`):** `security-and-administration`

## Tracker (metadata only)

- **Tracker type:** `none`
- **Work item id:** `P2-05`
- **Work item type:** `Story`
- **Status:** `Done`
- **Assignee:** ``
- **Labels:** `phase-2`, `backend`, `roles`

---

## Title

```
Role and permission management (admin)
```

---

## Description

```
Administrators create roles and assign permissions from the code-defined
catalog (customer-support-crm-api). Slices in Application/Features/Roles/ and
Application/Features/Permissions/, contracts in Contracts/Roles.

Endpoints and permissions:
- GET    /permissions               roles.view    Catalog grouped by module:
                                                  [{group, permissions:[{code, name, description}]}]
                                                  name/description localized (en/ar) from resources
- POST   /roles                     roles.manage  Create {name, description, permissions[]}
- GET    /roles/{id}                roles.view    Get with permission codes and user count
- GET    /roles                     roles.view    List (paged, search by name)
- PUT    /roles/{id}                roles.manage  Update {name, description}
- PUT    /roles/{id}/permissions    roles.manage  AssignPermissions {permissions[]} (replace set)
- DELETE /roles/{id}                roles.manage  Delete

Rules:
- Role name unique (case-insensitive) -> 409 ROLE_NAME_ALREADY_EXISTS.
- Unknown role -> 404 ROLE_NOT_FOUND.
- Unknown permission code -> 400 validation error with code UNKNOWN_PERMISSION on
  the offending permissions[i] field.
- System roles (IsSystem) cannot be renamed or deleted -> 422 SYSTEM_ROLE_READ_ONLY.
  The Administrator role's permissions are always Permissions.All and cannot be
  changed -> 422 SYSTEM_ROLE_READ_ONLY. Supervisor/Agent permissions may be edited.
- Role assigned to any user cannot be deleted -> 409 ROLE_IN_USE.
- Permission changes take effect on the affected users' next token refresh
  (see P2-03); no extra work needed here.
- All messages localized (en/ar).
```

---

## Acceptance criteria

```
- [ ] Permission catalog endpoint returns every code from Permissions.All, grouped and localized.
- [ ] Role CRUD and AssignPermissions work with the standard envelope and listed permissions.
- [ ] System-role, duplicate-name, unknown-permission and role-in-use rules return the listed codes.
- [ ] `dotnet build` passes with zero warnings.
```

---

## Attachments

None.

---

## Dependencies

- **Blocked by / related ids:** P2-01, P2-03
- **Depends on code areas or other stories:** Role entity + Permissions catalog (P2-01), RequirePermission (P2-03).

## Extra notes (optional)

- Implementation plan §21.3.
- Permission-change audit entries come in P2-06.

## Technical hints (optional)

- Repo root: `customer-support-crm-api/`. .NET 10.

## Out of scope

- Do NOT touch Docker, docker-compose.yml, deploy/ or .github/ (CI/CD).
- Do NOT create or modify anything under tests/.
- Adding new permission codes at runtime (catalog stays code-defined).
- Scoped (branch/department) role assignment (Phase 3).
