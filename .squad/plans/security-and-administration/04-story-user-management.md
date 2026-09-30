# Story 04 — User management (admin) (Story: P2-04)

> As-built plan: written after implementation in `customer-support-crm-api` commit `0f87e2d`; paths and line numbers refer to that commit.

## Prerequisites

- Phase 1 (Backend Platform) completed in `customer-support-crm-api` (commit `c3c7815`).
- Story 01 completed: [01-story-identity-domain-persistence.md](01-story-identity-domain-persistence.md) — `User`, `UserRole`, `Role`, `EmailAddress`, `IApplicationDbContext`, `IPasswordHasher`.
- Story 02 completed: [02-story-authentication-jwt-and-refresh-tokens.md](02-story-authentication-jwt-and-refresh-tokens.md) — `UserSessionService`, `RefreshTokenRevocationReason`, `AuthenticationErrors`, `AuthenticationHttp`, `RateLimitPolicies`.
- Story 03 completed: [03-story-current-user-and-permission-authorization.md](03-story-current-user-and-permission-authorization.md) — `ICurrentUser`, one authorization policy per permission code, `/api/v1` group requiring authentication.
- `IAuditTrail` / `AuditActions` are used by the handlers below; the audit table and writer belong to [06-story-audit-logging.md](06-story-audit-logging.md). In `0f87e2d` all seven stories shipped together, so the handlers call `audit.Record(...)` directly.
- All paths below are relative to the **`customer-support-crm-api/`** repo root.

---

## Story Goal

Administrators manage staff accounts through `/api/v1/users`:

1. Create a user with email, display name, password and roles.
2. Get one user (with roles and lockout state) and list users (paged, searchable, filterable, sortable).
3. Replace a user's role set.
4. Disable a user (revokes all their refresh-token sessions) and enable a user (clears lockout).
5. Any signed-in user changes their own password (revokes their other sessions).

Guards: unique email, no self-disable, never leave zero active administrators. Responses never expose `PasswordHash`.

**Deviations from the intake (the code is authoritative):**

| Intake | As built in `0f87e2d` |
|---|---|
| `fullName`, `culture` on the user | `DisplayName` only; `User` has no culture property (Story 01 domain) |
| `PUT /users/{id}` (update) | **Not implemented** — there is no Update slice |
| `POST /users/{id}/deactivate` / `activate` | `POST /users/{id}/disable` / `enable` (`UserStatus.Active` / `Disabled`) |
| `PUT /users/{id}/roles` "AssignRoles" | Same route, slice named `SetRoles` |
| `POST /users/me/change-password` | `POST /auth/change-password`, slice in `Features/Authentication/ChangePassword` |
| `users.view` for Get/List | Catalog has no `users.view`; every Users endpoint requires `users.manage` |
| `EMAIL_ALREADY_EXISTS` | `EMAIL_TAKEN` (409) |
| `CANNOT_DEACTIVATE_SELF` / `LAST_ADMINISTRATOR` → 422 | `CANNOT_DISABLE_SELF` / `LAST_ADMINISTRATOR` → **409** (`ConflictException`) |
| Unknown role → validation error | `UNKNOWN_ROLE` field error on `roleIds` (400) |
| List filters `isActive`, `roleId`; sort `fullName` | Filter `status` (`Active`/`Disabled`) only; sort `displayName` (default), `email`, `createdAt`, `lastLoginAt` |
| Password: min 10, letter + digit, ≠ email | `CommonRules.ValidPassword`: length 12–128, no composition rules (NIST SP 800-63B) |

**Not in scope:** invitations, email password reset, avatars, branch/department assignment (Phase 3). **Do not touch** `docker-compose.yml`, `deploy/`, `.github/` or anything under `tests/`.

---

## Context — Read These Files First

1. `src/CustomerSupportCrm.Application/Abstractions/Http/IEndpoint.cs` (Phase 1) — lines 9–12. Each slice has one `internal sealed class …Endpoint : IEndpoint`; discovered by `AddEndpoints` in `src/CustomerSupportCrm.Application/DependencyInjection.cs` lines 31–39.
2. `src/CustomerSupportCrm.Api/Endpoints/EndpointExtensions.cs` — lines 13–23. Endpoints are mapped on the `/api/v1` group, which already calls `.RequireAuthorization()`; routes in slices are relative (`"/users"`).
3. `src/CustomerSupportCrm.Application/Abstractions/Http/ApiResults.cs` — lines 10–24. `Ok`, `Created(location, data)`, `Success()`, `Paged(items, meta)`; always return these, never raw `Results.*`.
4. `src/CustomerSupportCrm.Application/Common/Pagination/PagedResult.cs` — lines 6–23. `PagedResult<T>.Map`, `ToPagedResultAsync(page, pageSize, ct)` (query must be ordered), `DefaultPageSize = 25`, `MaxPageSize = 100`. `PaginationMeta` is Phase 1 (`src/CustomerSupportCrm.Contracts/Common/PaginationMeta.cs` lines 3–14).
5. `src/CustomerSupportCrm.Application/Common/Validation/CommonRules.cs` — lines 7–27. `ValidPassword`, `ValidPage`, `ValidPageSize`, `ValidSortDirection`, `OneOf(...)` (emits `INVALID_VALUE`).
6. `src/CustomerSupportCrm.Domain/Users/User.cs` — lines 16–136. `User.Create(EmailAddress, displayName, passwordHash, roleIds)` (69–76), `SetRoles` (95–104), `HasRole` (106), `RoleIds` (67), `IsActive` (65), `IsLockedOut` (108), `Disable` / `Enable` (128–135), `ChangePasswordHash` (89–93), `DisplayNameMaxLength` (18).
7. `src/CustomerSupportCrm.Domain/Shared/EmailAddress.cs` — lines 9–48. `Create` trims + lower-cases and throws `DomainException(INVALID_EMAIL_ADDRESS)`; `MaxLength = 254`. Uniqueness is on the stored lower-case value (`UserConfiguration.cs` line 18, unique index).
8. `src/CustomerSupportCrm.Domain/Roles/Permissions.cs` — line 21 `UsersManage`; `src/CustomerSupportCrm.Domain/Roles/Role.cs` line 14 `AdministratorName`, line 69 `Normalize`.
9. `src/CustomerSupportCrm.Infrastructure/Authorization/AuthorizationSetup.cs` — lines 14–28. Policy name = permission code, so `.RequireAuthorization(Permissions.UsersManage)` is enough.
10. `src/CustomerSupportCrm.Application/Abstractions/Authentication/ICurrentUser.cs` — lines 10–27. `UserId`, `SessionId` (used by change-password to keep the current session).
11. `src/CustomerSupportCrm.Application/Features/Authentication/Common/UserSessionService.cs` — lines 48–60. `RevokeAsync(predicate, reason, ct)` revokes only active tokens; the caller saves.
12. `src/CustomerSupportCrm.Application/Abstractions/Auditing/IAuditTrail.cs` — lines 10–20; `src/CustomerSupportCrm.Domain/Audit/AuditActions.cs` lines 11–16 (`PasswordChanged`, `UserCreated`, `UserRolesChanged`, `UserDisabled`, `UserEnabled`) and 23–27 (`AuditEntityTypes.User`).
13. `src/CustomerSupportCrm.Application/Common/Exceptions/ConflictException.cs`, `NotFoundException.cs` (Phase 1) — `(code, message)` primary constructors.
14. `src/CustomerSupportCrm.Api/Middleware/GlobalExceptionHandler.cs` — lines 26–41 (exception → status map; `ValidationException` 400, `AppException` via `StatusFor` lines 59–66, `DomainException` 422, unique-violation race 409) and 71–86 (`ToValidationCode`: UPPER_SNAKE `ErrorCode`s pass through).
15. `src/CustomerSupportCrm.Application/Resources/Messages.resx` / `Messages.ar.resx` — lines 69–86 hold the keys this story adds.

---

## Backend Tasks

### 1 — Contracts

Create file: `src/CustomerSupportCrm.Contracts/Users/UserContracts.cs` (lines 1–27):

```csharp
public sealed record CreateUserRequest(string Email, string DisplayName, string Password, IReadOnlyList<Guid> RoleIds);
public sealed record SetUserRolesRequest(IReadOnlyList<Guid> RoleIds);
public sealed record RoleReference(Guid Id, string Name);
public sealed record UserListItemResponse(Guid Id, string Email, string DisplayName, string Status,
    IReadOnlyList<string> Roles, DateTimeOffset? LastLoginAt, DateTimeOffset CreatedAt);
public sealed record UserResponse(Guid Id, string Email, string DisplayName, string Status, bool IsLockedOut,
    IReadOnlyList<RoleReference> Roles, DateTimeOffset? LastLoginAt, DateTimeOffset CreatedAt, DateTimeOffset? UpdatedAt);
```

**No** `PasswordHash`, failed-attempt count or token data in any contract. `ChangePasswordRequest(string CurrentPassword, string NewPassword)` lives in `src/CustomerSupportCrm.Contracts/Authentication/AuthenticationContracts.cs` line 5.

### 2 — Shared error codes and queries

Create file: `src/CustomerSupportCrm.Application/Features/Users/Common/UserErrors.cs` — `public static class UserErrors`: `UserNotFound = "USER_NOT_FOUND"`, `EmailTaken = "EMAIL_TAKEN"`, `UnknownRole = "UNKNOWN_ROLE"`, `CannotDisableSelf = "CANNOT_DISABLE_SELF"`, `LastAdministrator = "LAST_ADMINISTRATOR"`. Delete `Features/Users/.gitkeep`.

Create file: `src/CustomerSupportCrm.Application/Features/Users/Common/UserQueries.cs` — `internal static class UserQueries`:

```csharp
public static Task<UserResponse> GetResponseAsync(IApplicationDbContext db, UserId userId, DateTimeOffset now, CancellationToken ct);
public static Task<IReadOnlyList<RoleId>> ResolveRoleIdsAsync(IApplicationDbContext db, IReadOnlyList<Guid> roleIds, IStringLocalizer<Messages> localizer, CancellationToken ct);
public static Task<RoleId?> GetAdministratorRoleIdAsync(IApplicationDbContext db, CancellationToken ct);
public static Task EnsureAnotherActiveAdministratorAsync(IApplicationDbContext db, RoleId administratorRoleId, UserId excluding, CancellationToken ct);
```

- `GetResponseAsync` (17–51): `AsNoTracking()`, projects to an anonymous row with a correlated `db.Roles` sub-select ordered by `Name`; throws `NotFoundException(USER_NOT_FOUND)`; `IsLockedOut = row.LockoutEndsAt > now`; `Status = row.Status.ToString()`.
- `ResolveRoleIdsAsync` (54–73): loads all role ids into a `HashSet` (small set; avoids translating a converted-id `IN` list), `Distinct()`s the request; any unknown id → `FluentValidation.ValidationException` with one `ValidationFailure("RoleIds", localizer[UNKNOWN_ROLE]) { ErrorCode = UNKNOWN_ROLE }`.
- `GetAdministratorRoleIdAsync` (75–82): `IsSystem && NormalizedName == Role.Normalize(Role.AdministratorName)`.
- `EnsureAnotherActiveAdministratorAsync` (85–97): if no *other* `Active` user holds the admin role → `ConflictException(LAST_ADMINISTRATOR)`.

### 3 — Create

Create files in `src/CustomerSupportCrm.Application/Features/Users/Create/`:

- `CreateUserCommand.cs` — `record CreateUserCommand(string Email, string DisplayName, string Password, IReadOnlyList<Guid> RoleIds) : IRequest<UserResponse>`.
- `CreateUserValidator.cs` — `Email` `NotEmpty().MaximumLength(EmailAddress.MaxLength).EmailAddress()`; `DisplayName` `NotEmpty().MaximumLength(User.DisplayNameMaxLength)`; `Password` `.ValidPassword()`; `RoleIds` `NotNull()`.
- `CreateUserHandler.cs` (lines 17–46) — deps `IApplicationDbContext, IPasswordHasher, IAuditTrail, IStringLocalizer<Messages>, TimeProvider`. Steps: `EmailAddress.Create` → `AnyAsync(u => u.Email == email.Value)` → `ConflictException(EMAIL_TAKEN)`; `ResolveRoleIdsAsync`; `User.Create(email, displayName, passwordHasher.Hash(password), roleIds)`; `db.Users.Add`; `audit.Record(UserCreated, User, id, newValues: new { user.Email, user.DisplayName, roleIds })` (**no password/hash**); `SaveChangesAsync`; return `GetResponseAsync`.
- `CreateUserEndpoint.cs` — `MapPost("/users", ...)`, maps `request.RoleIds ?? []`, returns `ApiResults.Created($"/api/v1/users/{user.Id}", user)`; `.RequireAuthorization(Permissions.UsersManage).WithName("CreateUser").WithTags("Users").Produces<ApiResponse<UserResponse>>(201)`.

### 4 — Get by id

Create files in `Features/Users/GetById/`: `GetUserByIdQuery(Guid UserId) : IRequest<UserResponse>`; `GetUserByIdHandler(IApplicationDbContext db, TimeProvider time)` → `UserQueries.GetResponseAsync(db, new UserId(id), time.GetUtcNow(), ct)`; `GetUserByIdEndpoint` → `MapGet("/users/{id:guid}")`, `ApiResults.Ok`, `users.manage`, name `GetUserById`.

### 5 — List

Create files in `Features/Users/List/`:

```csharp
public sealed record ListUsersQuery(
    int Page = 1,
    int PageSize = PaginationExtensions.DefaultPageSize,
    string? Search = null,
    string? Status = null,
    string? SortBy = null,
    string? SortDirection = null)
    : IRequest<PagedResult<UserListItemResponse>>;
```

- `ListUsersValidator.cs` — `ValidPage()`, `ValidPageSize()` (1–100), `Search` max 200, `Status.OneOf("Active", "Disabled")`, `SortBy.OneOf("displayName", "email", "createdAt", "lastLoginAt")`, `ValidSortDirection()`.
- `ListUsersHandler.cs` (lines 12–66): search upper-cases the term and matches `u.Email.ToUpper().Contains(term) || u.DisplayName.ToUpper().Contains(term)` inside `#pragma warning disable CA1304, CA1311, CA1862` (translated to SQL `upper()`); status via `Enum.Parse<UserStatus>`; `switch` on `SortBy` (default `DisplayName`), then `.ThenBy(u => u.Id)` for a stable page; `Select` projection with role **names** (no tracking needed for a projection); `ToPagedResultAsync` then `Map` to `UserListItemResponse`.
- `ListUsersEndpoint.cs` — `MapGet("/users", ([AsParameters] ListUsersQuery query, ...) => ApiResults.Paged(result.Items, result.Meta))`, `users.manage`, `Produces<ApiResponse<IReadOnlyList<UserListItemResponse>>>()`.

### 6 — Set roles

Create files in `Features/Users/SetRoles/`: `SetUserRolesCommand(Guid UserId, IReadOnlyList<Guid> RoleIds)`, `SetUserRolesValidator` (`RoleIds` `NotNull()`), endpoint `MapPut("/users/{id:guid}/roles")` mapping `request.RoleIds ?? []`.

`SetUserRolesHandler.cs` (lines 22–48): load user `Include(u => u.Roles)` or 404; `ResolveRoleIdsAsync`; if the admin role exists **and** the user is active, holds it, and the new set drops it → `EnsureAnotherActiveAdministratorAsync`; capture `previous = user.RoleIds`; `user.SetRoles(roleIds)`; audit `UserRolesChanged` with old/new role ids; save; return `GetResponseAsync`.

### 7 — Disable / Enable

Create files in `Features/Users/Disable/` — `DisableUserCommand(Guid UserId) : IRequest<UserResponse>`; endpoint `MapPost("/users/{id:guid}/disable")`. `DisableUserHandler.cs` (lines 27–53), deps include `ICurrentUser` and `UserSessionService`:

```csharp
if (userId == currentUser.UserId) throw new ConflictException(UserErrors.CannotDisableSelf, "You cannot disable your own account.");
// load with Include(u => u.Roles) or 404
if (user.IsActive)
{
    // last-admin guard when the user holds the admin role
    user.Disable();
    await sessions.RevokeAsync(token => token.UserId == userId, RefreshTokenRevocationReason.UserDisabled, cancellationToken);
    audit.Record(AuditActions.UserDisabled, AuditEntityTypes.User, user.Id.ToString());
    await db.SaveChangesAsync(cancellationToken);
}
```

Disabling an already-disabled user is a no-op that returns 200. Existing access tokens stay valid until expiry; refresh is refused immediately (doc comment lines 15–18).

Create files in `Features/Users/Enable/` — `EnableUserCommand(Guid UserId)`; endpoint `MapPost("/users/{id:guid}/enable")`. `EnableUserHandler.cs` (lines 16–32): acts only when `!IsActive || IsLockedOut(now)`; `user.Enable()` (clears failures + lockout); audit `UserEnabled` with `oldValues: new { wasActive, wasLockedOut }`; save; return `GetResponseAsync`.

All four Users endpoints: `.RequireAuthorization(Permissions.UsersManage).WithTags("Users").Produces<ApiResponse<UserResponse>>()`.

### 8 — Change own password

Create files in `src/CustomerSupportCrm.Application/Features/Authentication/ChangePassword/`:

- `ChangePasswordCommand(string CurrentPassword, string NewPassword) : IRequest`.
- `ChangePasswordValidator.cs` — `CurrentPassword` `NotEmpty().MaximumLength(CommonRules.PasswordMaxLength)`; `NewPassword` `.ValidPassword().NotEqual(c => c.CurrentPassword)` — the **same** `ValidPassword` rule as Create.
- `ChangePasswordHandler.cs` (lines 28–54): load caller by `currentUser.UserId` or `UnauthorizedException`; `Verify(...) == Failed` → `ValidationException` with field `CurrentPassword`, code `INVALID_CURRENT_PASSWORD`; `ChangePasswordHash(Hash(new))`; `sessions.RevokeAsync(t => t.UserId == userId && t.SessionId != currentUser.SessionId, PasswordChanged)`; audit `PasswordChanged`; save.
- `ChangePasswordEndpoint.cs` — `MapPost($"{AuthenticationHttp.RoutePrefix}/change-password")` → `/api/v1/auth/change-password`; `ApiResults.Success()`; `.RequireRateLimiting(RateLimitPolicies.Authentication)`; no permission, only the group's authenticated-user requirement.

### 9 — Localization

File: `src/CustomerSupportCrm.Application/Resources/Messages.resx` and `Messages.ar.resx` — add `INVALID_CURRENT_PASSWORD` (line 69), `USER_NOT_FOUND` (72), `EMAIL_TAKEN` (75), `UNKNOWN_ROLE` (78), `CANNOT_DISABLE_SELF` (81), `LAST_ADMINISTRATOR` (84). English values match the exception fallback messages (e.g. "At least one active administrator is required."). `INVALID_EMAIL_ADDRESS` / `INVALID_DISPLAY_NAME` (102, 105) cover the domain codes.

### 10 — Docs

File: `docs/api-contract.md` — "Feature error codes (Phase 2)" table (lines 86–104): `INVALID_CURRENT_PASSWORD` 400, `EMAIL_TAKEN` 409, `UNKNOWN_ROLE` 400, `USER_NOT_FOUND` 404, `CANNOT_DISABLE_SELF` 409, `LAST_ADMINISTRATOR` 409.

No DI changes: MediatR handlers, validators (`includeInternalTypes: true`) and endpoints are picked up by assembly scanning (`DependencyInjection.cs` lines 17–24); `UserSessionService` is already registered (line 26).

---

## Edge Cases & Failure Modes

- **Duplicate email with different casing / spaces** — `EmailAddress.Create` lower-cases and trims, then `CreateUserHandler` lines 27–31 return 409 `EMAIL_TAKEN`. A parallel insert that beats the check hits the unique index (`UserConfiguration.cs` line 18) → generic 409 `CONFLICT` (`GlobalExceptionHandler.cs` lines 35–36).
- **Malformed email passing FluentValidation** — `EmailAddress.Create` throws `DomainException(INVALID_EMAIL_ADDRESS)` → 422.
- **Unknown or duplicate role ids** — unknown → 400 field error `roleIds` / `UNKNOWN_ROLE` (`UserQueries.cs` 64–70); duplicates are removed by `Distinct()` (line 62) and `User.SetRoles`.
- **`roleIds` omitted in JSON** — endpoints coalesce `null` to `[]`; a user can be created or left with no roles.
- **Unknown user id** — 404 `USER_NOT_FOUND` from Get, SetRoles, Disable, Enable. A non-GUID id does not match the `{id:guid}` route → 404 `NOT_FOUND`.
- **Self-disable** — checked before loading (`DisableUserHandler.cs` 30–33) → 409 `CANNOT_DISABLE_SELF`.
- **Last active administrator** — disabling them (40–44) or removing the Administrator role (`SetUserRolesHandler.cs` 30–34) → 409 `LAST_ADMINISTRATOR`. A disabled admin losing the role is allowed. If no Administrator role exists, the guard is skipped.
- **Disable / enable twice** — idempotent: no state change, no audit entry, 200 with the current user.
- **Enable an active but locked-out user** — clears lockout (`EnableUserHandler.cs` 22–29).
- **Disabled user's sessions** — all active refresh tokens revoked with `UserDisabled`; access tokens live until expiry.
- **Wrong current password** — 400 field error `currentPassword` / `INVALID_CURRENT_PASSWORD`; nothing saved. New password equal to current → validation error on `newPassword`.
- **Password policy** — shorter than 12 or longer than 128 → `INVALID_LENGTH` on both Create and change-password (`CommonRules.cs` 13–14).
- **List paging** — `pageSize` > 100 or < 1, `page` < 1 → `OUT_OF_RANGE`; invalid `status` / `sortBy` / `sortDirection` → `INVALID_VALUE`; a page past the end returns an empty list with the correct `totalCount`/`totalPages`.
- **Concurrent edits on the same user** — `xmin` concurrency token (`UserConfiguration.cs` line 33) → 409 `CONFLICT`.
- **Missing permission / anonymous** — 403 / 401 envelopes from Story 03.
- **Secrets** — responses are projected DTOs; audit values contain email, display name and role ids only.

---

## Test Plan

Test projects are **out of scope** (`tests/` must not be modified). No tests are added, changed or removed by this story. Existing tests in `0f87e2d` that cover it (read-only references):

1. **Integration** — `tests/CustomerSupportCrm.IntegrationTests/UserManagementTests.cs`: `CreateThenListAndGetUser`, `DuplicateEmailIsRejectedCaseInsensitively`, `UnknownRoleIsAValidationError`, `AgentCannotManageUsersOrReadRoles`, `DisablingAUserBlocksSignInAndRefresh`, `AdministratorCannotDisableThemselves`, `LastActiveAdministratorKeepsTheRole`, `UnknownUserReturnsNotFound`.
2. **Integration** — `tests/CustomerSupportCrm.IntegrationTests/AuthenticationFlowTests.cs`: `ChangePasswordKeepsCurrentSessionAndRevokesOthers`, `ChangePasswordRejectsWrongCurrentPassword`, `AccountLocksAfterRepeatedFailuresUntilEnabled`.
3. **Unit (domain)** — `tests/CustomerSupportCrm.Domain.Tests/Users/UserTests.cs`: `CreateNormalizesEmailAndStartsActive`, `CreateRejectsBlankDisplayName`, `EnableClearsLockout`, `SetRolesReplacesMembershipWithoutDuplicates`.
4. **API** — `tests/CustomerSupportCrm.Api.Tests/SecurityTests.cs`: `FeatureEndpointsRequireAuthenticationByDefault` (lines 61–67) — anonymous `GET /api/v1/users` returns 401.

---

## Verification Steps

1. **Backend builds:** in `customer-support-crm-api/` run `dotnet build` — 0 warnings, 0 errors.
2. **Regression:** `dotnet test --filter "FullyQualifiedName!~IntegrationTests"` — passes.
3. **Run:** migrate and seed a bootstrap admin (see Story 01), start the API, sign in via `POST /api/v1/auth/login` and export `TOKEN`.
4. **Create:** `curl -i -X POST https://localhost:<port>/api/v1/users -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" -d '{"email":"Agent1@Example.com","displayName":"Agent One","password":"<12+ chars>","roleIds":["<agent role id>"]}'` → 201, `Location: /api/v1/users/{id}`, email lower-cased, no `passwordHash` field. Repeat with upper-case email → 409 `EMAIL_TAKEN`.
5. **List:** `curl "…/api/v1/users?search=agent&status=Active&sortBy=email&sortDirection=desc&pageSize=10" -H "Authorization: Bearer $TOKEN"` → `meta` with `page`, `pageSize`, `totalCount`, `totalPages`; `pageSize=101` → 400.
6. **Roles:** `curl -X PUT …/api/v1/users/{id}/roles -d '{"roleIds":[]}'` → 200 with `roles: []`; remove Administrator from the only admin → 409 `LAST_ADMINISTRATOR`.
7. **Disable / enable:** `POST …/users/{id}/disable` → `status: "Disabled"`; that user's refresh fails; `POST …/users/{ownId}/disable` → 409 `CANNOT_DISABLE_SELF`; `POST …/users/{id}/enable` → `status: "Active"`, `isLockedOut: false`.
8. **Change password:** `POST …/api/v1/auth/change-password` with a wrong `currentPassword` → 400 `INVALID_CURRENT_PASSWORD`; with the right one → 200; other sessions can no longer refresh.
9. **Localization:** repeat a failing call with `Accept-Language: ar` → Arabic message.
10. **Permissions:** an Agent token on `GET /api/v1/users` → 403.

---

## Done Criteria

- [x] Users endpoints exist (create, get, list, set roles, disable, enable) plus `POST /api/v1/auth/change-password`; all use the standard envelope; Users endpoints require `users.manage`, change-password requires authentication. (Update endpoint not built — see deviations.)
- [x] List supports paging, search, status filter and sorting with correct `PaginationMeta`.
- [x] Duplicate email (`EMAIL_TAKEN`), self-disable (`CANNOT_DISABLE_SELF`) and last-administrator (`LAST_ADMINISTRATOR`) return their codes.
- [x] Disable revokes all the user's active refresh tokens; change-password revokes every session except the current one.
- [x] Password policy (`CommonRules.ValidPassword`) is applied identically on create and change-password.
- [x] No password hash in any response, log or audit entry.
- [x] All new messages exist in `Messages.resx` and `Messages.ar.resx`.
- [x] Nothing changed in `tests/`, `docker-compose.yml`, `deploy/` or `.github/` by this story.
- [x] `dotnet build` passes with zero warnings.

**STOP HERE. Report to the user and wait for confirmation before proceeding to Story 05.**
