# Story 06 — Append-only audit log for entity changes and security events (Story: P2-06)

> As-built plan: written after implementation in `customer-support-crm-api` commit `0f87e2d`; paths and line numbers refer to that commit.

## Prerequisites

- Phase 1 (Backend Platform) completed in `customer-support-crm-api` (commit `c3c7815`): correlation id middleware, `ApiResponse` envelope, `ValidationBehavior`, `IEndpoint` scanning, `PaginationMeta`.
- [01-story-identity-domain-persistence.md](01-story-identity-domain-persistence.md) — `User`, `Role`, `UserId`, `ApplicationDbContext`, `IApplicationDbContext`, migration `InitialIdentity`.
- [02-story-authentication-jwt-and-refresh-tokens.md](02-story-authentication-jwt-and-refresh-tokens.md) — Login / Refresh / Logout / ChangePassword handlers (audit calls are added to them here).
- [03-story-current-user-and-permission-authorization.md](03-story-current-user-and-permission-authorization.md) — `ICurrentUser`, permission-code policies (`RequireAuthorization(Permissions.AuditView)`).
- [04-story-user-management.md](04-story-user-management.md) — Create / SetRoles / Disable / Enable user handlers, `PagedResult<T>` + `CommonRules` pagination rules.
- [05-story-role-and-permission-management.md](05-story-role-and-permission-management.md) — Create / Update / Delete role handlers.
- Next: [07-story-security-hardening.md](07-story-security-hardening.md).
- All paths below are relative to the **`customer-support-crm-api/`** repo root.

---

## Story Goal

Record who did what, when and from where, in the **same transaction** as the change it describes:

1. `AuditLog` entity (factory-only, no mutators) plus stable action / entity-type names in `Domain/Audit`.
2. Reuse `IRequestContext` (correlation id, IP, user agent) from Story 02 to stamp each audit row.
3. `IAuditTrail.Record(...)` that adds an `AuditLog` row to the current unit of work; handlers call it before their own `SaveChangesAsync`.
4. Security events (login succeeded/failed/locked out, logout, refresh-token reuse, password change) and user/role changes audited with explicit old/new snapshots that never contain secrets.
5. `GET /api/v1/audit-logs` (permission `audit.view`), filtered, paged (max 100), newest first.

**Deviations from the intake (follow the real code):**

- **No automatic change-capture interceptor.** The intake's `AuditSaveChangesInterceptor` / `IAuditable` / `[AuditIgnore]` redaction was not built. Auditing is **explicit** via `IAuditTrail` in each handler; redaction is by construction (handlers pass anonymous objects that only contain safe fields). The only interceptor, `AuditableEntityInterceptor`, stamps `CreatedAt/By` / `UpdatedAt/By` and writes no audit rows.
- **No `IAuditLogger.LogSecurityEventAsync`** — security events use the same `IAuditTrail.Record`.
- **Action names** are dotted strings (`auth.login.failed`, `users.roles_changed`, …) in `AuditActions`, not the intake's enum-like list; there is no generic `Created/Updated/Deleted`, and no `ActorEmail` column (the list response joins `ActorDisplayName` from `users`).
- **Append-only is not enforced in `ApplicationDbContext`** (no `InvalidOperationException` on Modified/Deleted `AuditLog` entries). It is enforced by the model: `AuditLog` has only private setters and a factory, and no update/delete endpoint or handler exists.
- **No separate `AddAuditLog` migration** — `audit_logs` is created by `InitialIdentity` together with the identity tables.
- **Slice is `Features/AuditLogs/List`** (`ListAuditLogsQuery`), not `Search`. Extra index on `action`.
- `HttpRequestContext` lives in `Infrastructure/Authentication`, not in Api.

**Not in scope:** audit export, retention, auditing reads. **Do not touch** `docker-compose.yml`, `deploy/`, `.github/` or anything under `tests/`.

---

## Context — Read These Files First

1. `src/CustomerSupportCrm.Application/Abstractions/Http/CorrelationIdHttpContextExtensions.cs` (Phase 1) — lines 5–16. `GetCorrelationId(this HttpContext)` reads the id that `CorrelationIdMiddleware` stored; reuse it, do not re-parse the header.
2. `src/CustomerSupportCrm.Api/Middleware/CorrelationIdMiddleware.cs` (Phase 1) — lines 11–34. Validates/generates the id (`Resolve`) and calls `SetCorrelationId` before the rest of the pipeline, so it is available to every handler.
3. `src/CustomerSupportCrm.Domain/Common/Entity.cs` — lines 3–12. `Entity<TId>` with a protected `(TId id)` ctor and a protected parameterless EF ctor; `AuditLog` derives from `Entity<Guid>`.
4. `src/CustomerSupportCrm.Application/Abstractions/Authentication/ICurrentUser.cs` — lines 10–27. `IsAuthenticated` and `UserId` (throws when anonymous — always check `IsAuthenticated` first).
5. `src/CustomerSupportCrm.Infrastructure/Persistence/ApplicationDbContext.cs` — lines 10–30. DbSets at 12–18; `ApplyConfigurationsFromAssembly` at 22; `UserId` value conversion at 27 (needed for `AuditLog.ActorUserId`).
6. `src/CustomerSupportCrm.Application/Abstractions/Persistence/IApplicationDbContext.cs` — lines 11–22. Handlers query through this interface.
7. `src/CustomerSupportCrm.Infrastructure/DependencyInjection.cs` — `AddPersistence` lines 37–63 and `AddIdentityServices` lines 65–103; the audit registrations sit at 101–102. `TimeProvider.System` and `AddHttpContextAccessor` are registered at 28–29.
8. `src/CustomerSupportCrm.Infrastructure/Persistence/Interceptors/AuditableEntityInterceptor.cs` — lines 9–50. Stamps `IAuditableEntity` fields only; **not** an audit-log writer. Do not add audit-row logic here.
9. `src/CustomerSupportCrm.Application/Common/Pagination/PagedResult.cs` — lines 6–23. `PagedResult<T>.Map`, `PaginationExtensions.DefaultPageSize = 25`, `MaxPageSize = 100`, `ToPagedResultAsync` (query must be ordered).
10. `src/CustomerSupportCrm.Application/Common/Validation/CommonRules.cs` — lines 16–20. `ValidPage()` / `ValidPageSize()`.
11. `src/CustomerSupportCrm.Application/Abstractions/Http/ApiResults.cs` — lines 22–23. `ApiResults.Paged(items, meta)` returns the envelope with `meta`.
12. `src/CustomerSupportCrm.Api/Endpoints/EndpointExtensions.cs` — lines 13–23. Every `IEndpoint` is mapped under `/api/v1` with `RequireAuthorization()`; `src/CustomerSupportCrm.Application/DependencyInjection.cs` lines 31–39 registers endpoints by assembly scan (no manual registration).
13. `src/CustomerSupportCrm.Domain/Roles/Permissions.cs` — line 24 `AuditView = "audit.view"`; permission codes double as policy names.
14. `src/CustomerSupportCrm.Application/Features/Users/List/ListUsersHandler.cs` / `ListUsersQuery.cs` — precedent for a paged `[AsParameters]` list slice.

---

## Backend Tasks

### 1 — Domain: `AuditLog` and action names

Create file: `src/CustomerSupportCrm.Domain/Audit/AuditLog.cs` (delete `Domain/Audit/.gitkeep`)

- `public sealed class AuditLog : Entity<Guid>`; constants `ActionMaxLength = 100`, `EntityTypeMaxLength = 100`, `EntityIdMaxLength = 100`, `CorrelationIdMaxLength = 64`, `IpAddressMaxLength = 64`, `UserAgentMaxLength = 512`.
- Two private ctors (parameterless for EF, `(Guid id) : base(id)`), both initialise `Action`/`EntityType` to `string.Empty`.
- Properties, all `{ get; private set; }`: `DateTimeOffset OccurredAt`, `UserId? ActorUserId`, `string Action`, `string EntityType`, `string? EntityId`, `string? CorrelationId`, `string? OldValues`, `string? NewValues` (JSON text), `string? IpAddress`, `string? UserAgent`.

```csharp
public static AuditLog Record(
    DateTimeOffset occurredAt, UserId? actorUserId, string action, string entityType, string? entityId,
    string? oldValues, string? newValues, string? correlationId, string? ipAddress, string? userAgent);
```

- `ArgumentException.ThrowIfNullOrWhiteSpace` on `action` and `entityType`; id `Guid.CreateVersion7()`; every string with a max length goes through a private `Truncate(value, max)` that returns `null` for empty strings (lines 84–85). JSON values are stored as given. No other public members.

Create file: `src/CustomerSupportCrm.Domain/Audit/AuditActions.cs` — two static classes (lines 4–27):

| `AuditActions` constant | Value |
|---|---|
| `LoginSucceeded` / `LoginFailed` / `LoginLockedOut` | `auth.login.succeeded` / `auth.login.failed` / `auth.login.locked_out` |
| `Logout` / `RefreshTokenReuseDetected` / `PasswordChanged` | `auth.logout` / `auth.refresh.reuse_detected` / `auth.password.changed` |
| `UserCreated` / `UserRolesChanged` / `UserDisabled` / `UserEnabled` | `users.created` / `users.roles_changed` / `users.disabled` / `users.enabled` |
| `RoleCreated` / `RoleUpdated` / `RoleDeleted` | `roles.created` / `roles.updated` / `roles.deleted` |

`AuditEntityTypes.User = "User"`, `AuditEntityTypes.Role = "Role"`. Doc comment: never rename an existing value.

### 2 — Request context

**Already created by [Story 02](02-story-authentication-jwt-and-refresh-tokens.md)** — do not create again. Reuse `IRequestContext` (`src/CustomerSupportCrm.Application/Abstractions/Http/IRequestContext.cs`, lines 4–11: `CorrelationId`, `IpAddress`, `UserAgent`) and its implementation `HttpRequestContext` (`src/CustomerSupportCrm.Infrastructure/Authentication/HttpRequestContext.cs`), which reads the correlation id via `GetCorrelationId()`, the IP from `Connection.RemoteIpAddress` and the `User-Agent` header. `AuditTrail` below takes it as a constructor dependency.

Doc comment: behind a reverse proxy, forwarded headers must be configured so `IpAddress` is the client's.

### 3 — Audit trail

Create file: `src/CustomerSupportCrm.Application/Abstractions/Auditing/IAuditTrail.cs`

```csharp
public interface IAuditTrail
{
    /// <param name="actorUserId">Overrides the current user, e.g. during sign-in.</param>
    void Record(string action, string entityType, string? entityId,
        object? oldValues = null, object? newValues = null, UserId? actorUserId = null);
}
```

Create file: `src/CustomerSupportCrm.Infrastructure/Auditing/AuditTrail.cs` — `internal sealed class AuditTrail(ApplicationDbContext db, ICurrentUser currentUser, IRequestContext request, TimeProvider time) : IAuditTrail`:

- Actor = `actorUserId ?? (currentUser.IsAuthenticated ? currentUser.UserId : null)` (line 28).
- `db.AuditLogs.Add(AuditLog.Record(time.GetUtcNow(), actor, ..., Serialize(oldValues), Serialize(newValues), request.CorrelationId, request.IpAddress, request.UserAgent))`.
- `Serialize` uses a static `JsonSerializerOptions(JsonSerializerDefaults.Web)` (camelCase) and returns `null` for `null`.
- **Does not save.** The caller's `SaveChangesAsync` commits the audit row with the change.

### 4 — Persistence

Create file: `src/CustomerSupportCrm.Infrastructure/Persistence/Configurations/AuditLogConfiguration.cs` (lines 7–30): table `audit_logs`; key `Id` `ValueGeneratedNever()`; `HasMaxLength` from the `AuditLog` constants; `OldValues`/`NewValues` `HasColumnType("jsonb")`; **no FK to users** (history outlives rows); indexes on `OccurredAt`, `(EntityType, EntityId)`, `ActorUserId`, `Action`.

File: `src/CustomerSupportCrm.Infrastructure/Persistence/ApplicationDbContext.cs` — add `public DbSet<AuditLog> AuditLogs => Set<AuditLog>();` (line 18).

File: `src/CustomerSupportCrm.Application/Abstractions/Persistence/IApplicationDbContext.cs` — add `DbSet<AuditLog> AuditLogs { get; }` (line 19).

Migration: `audit_logs` is part of `src/CustomerSupportCrm.Infrastructure/Persistence/Migrations/20260930092037_InitialIdentity.cs` — `CreateTable` lines 14–33, indexes `ix_audit_logs_action`, `ix_audit_logs_actor_user_id`, `ix_audit_logs_entity_type_entity_id`, `ix_audit_logs_occurred_at` at lines 149–167, `DropTable` at 206–207. If executing stories separately, generate it with `dotnet ef migrations add <Name> --project src/CustomerSupportCrm.Infrastructure --startup-project src/CustomerSupportCrm.Api --output-dir Persistence/Migrations` and check it matches these columns (`jsonb`, `varchar(64/100/512)`, `timestamp with time zone`).

### 5 — DI

File: `src/CustomerSupportCrm.Infrastructure/DependencyInjection.cs` — in `AddIdentityServices`, after `ICurrentUser` (line 100):

```csharp
services.AddScoped<IAuditTrail, AuditTrail>();
```

`IRequestContext` is already registered by Story 02.

Add usings `CustomerSupportCrm.Application.Abstractions.Auditing`, `...Abstractions.Http`, `CustomerSupportCrm.Infrastructure.Auditing`.

### 6 — Audit calls in handlers

Inject `IAuditTrail audit` into each handler and call `audit.Record(...)` **before** the handler's `SaveChangesAsync`. Pass only safe fields — never `PasswordHash`, token hashes, raw tokens or passwords.

| File (under `src/CustomerSupportCrm.Application/Features/`) | Line(s) | Action / values |
|---|---|---|
| `Authentication/Login/LoginHandler.cs` | 38 | `LoginFailed`, `entityId: null`, `new { email, reason = "unknown_email" }` |
| same | 45 | `LoginLockedOut`, `new { user.LockoutEndsAt }`, `actorUserId: user.Id` |
| same | 54 | `LoginFailed`, `new { reason = "invalid_password", lockedOut = user.IsLockedOut(now) }` |
| same | 61 | `LoginFailed`, `new { reason = "disabled" }` |
| same | 73 | `LoginSucceeded`, `actorUserId: user.Id` |
| `Authentication/Logout/LogoutHandler.cs` | 35 | `Logout`, `new { stored.SessionId }`, `actorUserId: stored.UserId` |
| `Authentication/Refresh/RefreshSessionHandler.cs` | 39 | `RefreshTokenReuseDetected`, `new { stored.SessionId }` |
| `Authentication/ChangePassword/ChangePasswordHandler.cs` | 52 | `PasswordChanged`, no values |
| `Users/Create/CreateUserHandler.cs` | 37–41 | `UserCreated`, `new { user.Email, user.DisplayName, roleIds }` |
| `Users/SetRoles/SetUserRolesHandler.cs` | 36–44 | `UserRolesChanged`, old `{ roleIds = previous }` / new `{ roleIds }` |
| `Users/Disable/DisableUserHandler.cs` | 48 | `UserDisabled` (only when state changes) |
| `Users/Enable/EnableUserHandler.cs` | 27 | `UserEnabled`, old `{ wasActive, wasLockedOut }` |
| `Roles/Create/CreateRoleHandler.cs` | 24–28 | `RoleCreated`, new `{ Name, Description, permissions }` |
| `Roles/Update/UpdateRoleHandler.cs` | 30–38 | `RoleUpdated`, before/after `{ Name, Description, permissions }` |
| `Roles/Delete/DeleteRoleHandler.cs` | 29–33 | `RoleDeleted`, old `{ Name, Description, permissions }` |

Failure paths in `LoginHandler` (lines 38–40, 45–47, 54–56, 61–63) and `RefreshSessionHandler` (39–41) call `SaveChangesAsync` **before** throwing `UnauthorizedException`, so the audit row persists although the response is 401.

### 7 — List slice

Create folder `src/CustomerSupportCrm.Application/Features/AuditLogs/List/`:

- **`ListAuditLogsQuery.cs`** — `public sealed record ListAuditLogsQuery(int Page = 1, int PageSize = PaginationExtensions.DefaultPageSize, string? Action = null, string? EntityType = null, string? EntityId = null, Guid? ActorUserId = null, DateTimeOffset? From = null, DateTimeOffset? To = null) : IRequest<PagedResult<AuditLogResponse>>;`
- **`ListAuditLogsValidator.cs`** — `ValidPage()`, `ValidPageSize()`, `MaximumLength` on `Action`/`EntityType`/`EntityId` from `AuditLog` constants, `To >= From` when both set (line 16).
- **`ListAuditLogsHandler.cs`** — exact-match filters (lines 18–47; `ActorUserId` wrapped in `new UserId(actor)`), `From`/`To` inclusive; `OrderByDescending(OccurredAt).ThenByDescending(Id)`; projection with correlated subquery `ActorDisplayName = db.Users.Where(u => u.Id == a.ActorUserId).Select(u => u.DisplayName).FirstOrDefault()` (line 57); `ToPagedResultAsync`; `Map` parses JSON text into `JsonElement?` via `JsonDocument.Parse(...).RootElement.Clone()` (lines 84–93).
- **`ListAuditLogsEndpoint.cs`** — `internal sealed class ListAuditLogsEndpoint : IEndpoint`; `MapGet("/audit-logs", [AsParameters] ListAuditLogsQuery ...)` → `ApiResults.Paged(result.Items, result.Meta)`; `.RequireAuthorization(Permissions.AuditView).WithName("ListAuditLogs").WithTags("Audit").Produces<ApiResponse<IReadOnlyList<AuditLogResponse>>>()`.

Create file: `src/CustomerSupportCrm.Contracts/Audit/AuditContracts.cs`

```csharp
public sealed record AuditLogResponse(
    Guid Id, DateTimeOffset OccurredAt, Guid? ActorUserId, string? ActorDisplayName,
    string Action, string EntityType, string? EntityId,
    JsonElement? OldValues, JsonElement? NewValues,
    string? CorrelationId, string? IpAddress, string? UserAgent);
```

No update or delete endpoint for audit logs.

### 8 — Docs

File: `docs/security.md` — "Audit log" section (lines 71–73): append-only, same transaction, recorded fields, actions in `Domain/Audit/AuditActions.cs`, `GET /api/v1/audit-logs` (`audit.view`), no secrets. Line 87: forwarded headers for audit IPs.
File: `docs/architecture.md` — `AuditLogs | List` slice row (line 67), `IRequestContext` / `IAuditTrail` rows (lines 75, 78), explicit auditing note (line 112).

---

## Edge Cases & Failure Modes

- **Handler throws after `Record`** — the row is only in the change tracker; nothing is saved, so no orphan audit row (`AuditTrail.cs` never calls `SaveChanges`).
- **Failed login must persist** — `LoginHandler` saves before throwing (task 6). Unknown email: `entityId` null, actor null, email stored in `newValues`; the password is never passed.
- **Anonymous caller** — `AuditTrail` checks `currentUser.IsAuthenticated` before reading `UserId` (line 28); sign-in handlers pass `actorUserId` explicitly.
- **Secrets** — no automatic redaction exists; the guarantee depends on handlers passing only the fields in the task 6 table. Never pass a `User` or `RefreshToken` entity as `oldValues`/`newValues`.
- **Oversized values** — `Action`, `EntityType`, `EntityId`, `CorrelationId`, `IpAddress`, `UserAgent` are truncated in `AuditLog.Record` (lines 73–80); empty strings become `null`.
- **Missing correlation header** — `CorrelationIdMiddleware.Resolve` generates one, so `CorrelationId` is set for every HTTP request; outside HTTP (`--init-database`) all request fields are `null`.
- **Deleted actor** — no FK (configuration line 24); `ActorDisplayName` becomes `null`, the row stays.
- **Append-only** — not guarded in `ApplicationDbContext`; mutation is only prevented by private setters and the absence of any update/delete path. Adding such a guard is a follow-up, not part of this as-built story.
- **Bad query** — `pageSize` > 100, `page` < 1, `to` < `from` → 400 `VALIDATION_ERROR` (`ValidationBehavior` → `GlobalExceptionHandler` line 29). Unknown `action` just returns an empty page.
- **No token / missing `audit.view`** — 401 / 403 envelope from P2-03.

---

## Test Plan

Test projects are **out of scope** by standing directive; no tests are added, changed or removed. Existing tests in `0f87e2d` that cover this story (read-only references):

1. `tests/CustomerSupportCrm.IntegrationTests/AuthenticationFlowTests.cs` — `SignInEventsAreAuditedWithCorrelationId` (lines 156–178): failed login with `X-Correlation-Id: audit-probe-1` returns 401, then `GET /api/v1/audit-logs?action=auth.login.failed` as admin finds the entry with that correlation id, `reason = "unknown_email"`, and no password text.
2. `tests/CustomerSupportCrm.Api.Tests/SecurityTests.cs` — `ProtectedEndpointWithoutTokenReturns401Envelope` (line 19) and `MissingPermissionReturns403Envelope` (line 46): generic 401/403 envelopes (P2-03) that the audit endpoint inherits.

---

## Migration / Rollback

- **Apply:** `dotnet ef database update --project src/CustomerSupportCrm.Infrastructure --startup-project src/CustomerSupportCrm.Api` (or `--init-database`; in Development the API migrates on startup).
- **Rollback:** `audit_logs` shares `InitialIdentity` with the identity tables, so rolling back (`dotnet ef database update 0 ...`) drops everything; there is no audit-only rollback.

---

## Verification Steps

1. **Backend builds:** in `customer-support-crm-api/` run `dotnet build` — 0 warnings, 0 errors.
2. **Regression:** `dotnet test --filter "FullyQualifiedName!~IntegrationTests"` passes; with Docker available, `dotnet test tests/CustomerSupportCrm.IntegrationTests` passes.
3. **Migration:** `dotnet ef database update ...` succeeds; `\d audit_logs` shows `old_values`/`new_values` as `jsonb` and the four `ix_audit_logs_*` indexes.
4. **Failed login audited:**
   ```bash
   curl -k -X POST https://localhost:<port>/api/v1/auth/login -H "Content-Type: application/json" \
     -H "X-Correlation-Id: audit-check-1" -d '{"email":"nobody@example.com","password":"wrong-password-123"}'
   ```
   → 401. Sign in as the bootstrap admin (credentials from your local `Bootstrap` config), then:
   ```bash
   curl -k "https://localhost:<port>/api/v1/audit-logs?action=auth.login.failed&pageSize=10" -H "Authorization: Bearer $TOKEN"
   ```
   → entry with `correlationId: "audit-check-1"`, `ipAddress`, `userAgent`, `newValues.reason: "unknown_email"`.
5. **Entity change audited:** `POST /api/v1/roles` then `GET /api/v1/audit-logs?entityType=Role&entityId=<id>` → `roles.created` with `newValues.permissions`; `PUT` the role → `roles.updated` with old and new values.
6. **Paging / validation:** `?pageSize=101` → 400; `?from=2026-10-01T00:00:00Z&to=2026-09-01T00:00:00Z` → 400; response `meta` has `page`, `pageSize`, `totalCount`, `totalPages`.
7. **Authorization:** no token → 401; token of a user without `audit.view` → 403.
8. **No secrets:** `SELECT old_values, new_values FROM audit_logs` contains no `AQAAAA` (Identity hash prefix), no password and no refresh-token value.

---

## Done Criteria

- [x] Creating/updating a user or role writes `AuditLog` rows with correct old/new values and correlation id (explicit `IAuditTrail.Record` calls, task 6).
- [x] Password hash, security stamp and refresh tokens never appear in any audit row (only safe fields are passed).
- [x] Login success/failure, lockout, logout and refresh-token reuse are audited with IP and user agent; failures persist despite 401.
- [x] Audit rows cannot be modified through the domain model (private setters, factory only; no update/delete path). *Deviation: `ApplicationDbContext` does not throw on Modified/Deleted entries.*
- [x] `GET /api/v1/audit-logs` filters, pages (max 100, newest first) and requires `audit.view`.
- [x] `audit_logs` table and indexes created by migration `InitialIdentity` (no separate `AddAuditLog`), applies cleanly.
- [x] Nothing changed in `tests/`, `docker-compose.yml`, `deploy/` or `.github/`.
- [x] `dotnet build` passes with zero warnings.

**STOP HERE. Report to the user and wait for confirmation before proceeding to Story 07.**
