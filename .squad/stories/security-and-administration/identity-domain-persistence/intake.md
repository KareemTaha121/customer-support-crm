# Story intake

- Folder: `.squad/stories/security-and-administration/identity-domain-persistence/intake.md`

---

## Feature

- **Feature name (display):** Security & Administration — Phase 2 Identity & Authorization
- **Feature slug (folder under `plans/`):** `security-and-administration`

## Tracker (metadata only)

- **Tracker type:** `none`
- **Work item id:** `P2-01`
- **Work item type:** `Story`
- **Status:** `Done`
- **Assignee:** ``
- **Labels:** `phase-2`, `backend`, `identity`

---

## Title

```
Identity domain & persistence
```

---

## Description

```
Add the identity domain model (users, roles, permissions, refresh tokens) to the
backend (repo: customer-support-crm-api), persist it with EF Core/PostgreSQL, and
seed default roles plus a bootstrap administrator. This is the foundation every
other Phase 2 story builds on. No HTTP endpoints in this story.

Domain (src/CustomerSupportCrm.Domain/Users/):
- User aggregate: Id (Guid.CreateVersion7), Email + NormalizedEmail (unique),
  FullName, PasswordHash, IsActive, PreferredCulture ("en"|"ar"), SecurityStamp,
  AccessFailedCount, LockoutEndsAt, LastLoginAt, CreatedAt, UpdatedAt.
  Behaviors: Create, Rename, ChangePasswordHash (rotates SecurityStamp),
  Activate, Deactivate, AssignRoles(set), RecordFailedLogin(now, maxAttempts,
  lockoutDuration), RecordSuccessfulLogin(now), IsLockedOut(now).
  Invariants throw DomainException with stable codes.
- Role aggregate: Id, Name + NormalizedName (unique), Description, IsSystem,
  permission codes (RolePermission rows). Behaviors: Create, Rename,
  SetPermissions(set), guard: system roles cannot be renamed.
- Permissions: code-defined catalog, NOT a DB table. Static class
  `Permissions` with nested groups and string constants, e.g.
  users.view, users.manage, roles.view, roles.manage, audit.view,
  tickets.view, tickets.create, tickets.update, tickets.assign, tickets.delete,
  customers.view, customers.create, customers.update, reports.view,
  settings.manage. Expose `Permissions.All` and `Permissions.IsDefined(code)`.
- UserRole (UserId, RoleId) and RolePermission (RoleId, PermissionCode) joins.
- RefreshToken entity: Id, UserId, TokenHash (SHA-256, never the raw token),
  FamilyId, CreatedAt, ExpiresAt, RevokedAt, RevokedReason, ReplacedByTokenId,
  CreatedByIp, UserAgent. Behaviors: IsActive(now), Revoke(now, reason),
  MarkReplaced(newId, now).

Persistence (src/CustomerSupportCrm.Infrastructure/Persistence/):
- DbSets on ApplicationDbContext; one IEntityTypeConfiguration per entity in
  Persistence/Configurations (snake_case is applied globally already).
- Unique indexes: users.normalized_email, roles.normalized_name,
  refresh_tokens.token_hash. Index refresh_tokens(user_id), (family_id).
- Max lengths on all strings; PreferredCulture length 5.
- Generate the first EF migration `AddIdentity` into Persistence/Migrations.

Seeding (src/CustomerSupportCrm.Infrastructure/Persistence/Seed/):
- IdentitySeeder run once at startup (hosted service or explicit call after
  build) that is idempotent:
  - Ensures system roles: Administrator (all permissions), Supervisor,
    Agent with sensible default permission sets.
  - If there are no users and `Identity:BootstrapAdmin:Email` and
    `Identity:BootstrapAdmin:Password` are configured, creates an active
    Administrator user. Never hard-code credentials; empty in appsettings.json,
    developers set them via dotnet user-secrets.
- Uses TimeProvider (BCL) for timestamps, registered as TimeProvider.System.
```

---

## Acceptance criteria

```
- [ ] Domain entities above exist under Domain/Users with no framework references.
- [ ] Permission catalog is code-defined; Administrator seeded with Permissions.All.
- [ ] EF configurations, unique indexes and `AddIdentity` migration exist; `dotnet ef database update` applies cleanly.
- [ ] Seeder is idempotent (running twice creates nothing new) and skips admin creation when config is empty.
- [ ] Password hash, token hash and security stamp are never logged.
- [ ] `dotnet build` passes with zero warnings (warnings are errors).
```

---

## Attachments

None.

---

## Dependencies

- **Blocked by / related ids:** none (Phase 1 is done)
- **Depends on code areas or other stories:** ApplicationDbContext, DomainException, DatabaseOptions from Phase 1.

## Extra notes (optional)

- Feature spec: `.squad/features/10-security-and-administration.md` (Phase 2 section).
- Implementation plan sections: §4–5 (DDD), §10 (EF Core, no generic repository / unit of work), §21 (auth), §24 (options).

## Technical hints (optional)

- Repo root for this story: `customer-support-crm-api/`. Primary language: `C#` (.NET 10).
- Package versions are central in `Directory.Packages.props`.
- `Microsoft.Extensions.Identity.Core` is acceptable in Infrastructure for `PasswordHasher<T>` (used in story P2-02); Domain stays framework-free.

## Out of scope

- Do NOT touch Docker, docker-compose.yml, deploy/ or .github/ (CI/CD).
- Do NOT create or modify anything under tests/.
- HTTP endpoints, JWT, login (P2-02); user/role management endpoints (P2-04, P2-05); audit log (P2-06).
- Organization / Branch / Department scoping (Phase 3).
