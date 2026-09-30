# Story intake

- Folder: `.squad/stories/security-and-administration/user-management/intake.md`

---

## Feature

- **Feature name (display):** Security & Administration — Phase 2 Identity & Authorization
- **Feature slug (folder under `plans/`):** `security-and-administration`

## Tracker (metadata only)

- **Tracker type:** `none`
- **Work item id:** `P2-04`
- **Work item type:** `Story`
- **Status:** `Ready`
- **Assignee:** ``
- **Labels:** `phase-2`, `backend`, `users`

---

## Title

```
User management (admin)
```

---

## Description

```
Administrators manage internal users (agents, supervisors, admins) through
/api/v1/users in customer-support-crm-api. One vertical slice per use case in
Application/Features/Users/<UseCase>/ (Command/Query, Validator, Handler,
Endpoint, Response). Contracts in Contracts/Users.

Endpoints and permissions:
- POST   /users                     users.manage  Create {email, fullName, password, culture, roleIds[]}
- GET    /users/{id}                users.view    Get (with roles)
- GET    /users                     users.view    List: page, pageSize (max 100), search (name/email),
                                                  isActive, roleId, sortBy (fullName|email|createdAt|lastLoginAt),
                                                  sortDirection; returns PaginationMeta
- PUT    /users/{id}                users.manage  Update {fullName, culture}
- PUT    /users/{id}/roles          users.manage  AssignRoles {roleIds[]} (replace set)
- POST   /users/{id}/deactivate     users.manage  Deactivate (revokes all active refresh tokens)
- POST   /users/{id}/activate       users.manage  Activate (clears lockout)
- POST   /users/me/change-password  authenticated ChangePassword {currentPassword, newPassword}
                                                  (revokes the caller's other refresh-token families)

Rules:
- Email unique (case-insensitive) -> 409 EMAIL_ALREADY_EXISTS.
- Unknown user -> 404 USER_NOT_FOUND; unknown role id -> 400 validation error on roleIds.
- Admin cannot deactivate themselves -> 422 CANNOT_DEACTIVATE_SELF.
- The last active user holding the Administrator role cannot be deactivated or
  lose that role -> 422 LAST_ADMINISTRATOR.
- Wrong current password -> 400 INVALID_CURRENT_PASSWORD.
- Shared password policy validator (reuse for Create and ChangePassword):
  min 10 chars, at least one letter and one digit, max 128, not equal to email.
- Responses never include PasswordHash or SecurityStamp.
- Queries use AsNoTracking and project directly to response DTOs.
- All messages localized (en/ar) in Messages.resx / Messages.ar.resx.
```

---

## Acceptance criteria

```
- [ ] All eight endpoints exist, use the standard envelope, and enforce the listed permissions.
- [ ] List supports paging, search, filters and sorting with correct PaginationMeta.
- [ ] Duplicate email, self-deactivation and last-administrator rules return the listed codes.
- [ ] Deactivation and password change revoke refresh tokens as described.
- [ ] Password policy is enforced identically on create and change-password.
- [ ] No password hash / security stamp in any response or log.
- [ ] `dotnet build` passes with zero warnings.
```

---

## Attachments

None.

---

## Dependencies

- **Blocked by / related ids:** P2-01, P2-02, P2-03
- **Depends on code areas or other stories:** User/Role entities, IPasswordHasher, ICurrentUser, RequirePermission, ApiResults, PaginationMeta.

## Extra notes (optional)

- Implementation plan §7 (vertical slices), §8 (CQRS), §16 (endpoint conventions), §76 (pagination).
- Audit entries for these actions come in P2-06 via the SaveChanges interceptor + security events.

## Technical hints (optional)

- Repo root: `customer-support-crm-api/`. .NET 10.

## Out of scope

- Do NOT touch Docker, docker-compose.yml, deploy/ or .github/ (CI/CD).
- Do NOT create or modify anything under tests/.
- User invitations by email, password reset by email, avatars.
- Branch / department assignment (Phase 3).
