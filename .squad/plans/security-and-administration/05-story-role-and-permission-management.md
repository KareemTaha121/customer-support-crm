# Story 05 — Role and permission management (admin) (Story: P2-05)

> As-built plan: written after implementation in `customer-support-crm-api` commit `0f87e2d`; paths and line numbers refer to that commit.

## Prerequisites

- [01-story-identity-domain-persistence.md](01-story-identity-domain-persistence.md) completed: `Role` aggregate, `Permissions` catalog, `RoleConfiguration`, `DatabaseInitializer` seed.
- [03-story-current-user-and-permission-authorization.md](03-story-current-user-and-permission-authorization.md) completed: one policy per permission code plus `PolicyNames.RolesRead`.
- Related: [04-story-user-management.md](04-story-user-management.md) (assigns roles, so it needs to read roles), [06-story-audit-logging.md](06-story-audit-logging.md) (`IAuditTrail`, used by the handlers below), [02-story-authentication-jwt-and-refresh-tokens.md](02-story-authentication-jwt-and-refresh-tokens.md) (permission changes reach users on refresh).
- Later stories: [07-story-security-hardening.md](07-story-security-hardening.md).
- All paths are relative to the **`customer-support-crm-api/`** repo root.

---

## Story Goal

Give administrators an API to manage roles and view the code-defined permission catalog:

1. `GET /api/v1/permissions` returns every code in `Permissions.All` together with its group.
2. Role CRUD: `POST /roles`, `GET /roles`, `GET /roles/{id}`, `PUT /roles/{id}` (name, description **and** full permission set), `DELETE /roles/{id}`.
3. Business rules: case-insensitive unique name, system roles cannot be changed, a role assigned to users cannot be deleted, and unknown permission codes are rejected. All errors use the standard envelope and have localized messages (en/ar).

**Deviations from the intake (the real code wins):**

- **No `roles.view` permission.** The catalog in `0f87e2d` has 13 codes. Read endpoints use the composite policy `PolicyNames.RolesRead`, which accepts **`roles.manage` or `users.manage`**. Write endpoints use `roles.manage`.
- **No `Features/Permissions/` slice.** The catalog endpoint lives at `Features/Roles/ListPermissions/`. It returns a flat list of `{code, group}` (group = the prefix before the first `.`), not groups with localized name/description.
- **No `PUT /roles/{id}/permissions`.** `PUT /roles/{id}` replaces name, description and the permission set in one call (`Role.Update`).
- **`GET /roles` is not paged and has no search.** It returns every role, ordered by name.
- **Error codes:** `ROLE_NAME_TAKEN` (not `ROLE_NAME_ALREADY_EXISTS`), `ROLE_IS_SYSTEM` (not `SYSTEM_ROLE_READ_ONLY`).
- **Only Administrator is a system role.** `DatabaseInitializer` seeds `Manager` and `Agent` as ordinary roles (not Supervisor), so they can be edited and deleted. A system role cannot be edited at all, including its permissions.
- Handlers already write audit entries through `IAuditTrail` (P2-06 shipped in the same commit).

**Not in scope:** adding permission codes at runtime, scoped role assignment (Phase 3), Docker, `deploy/`, `.github/`, `tests/`.

---

## Context — Read These Files First

1. `src/CustomerSupportCrm.Domain/Roles/Role.cs` — lines 9–145. Constants `NameMaxLength`, `DescriptionMaxLength`, `AdministratorName`, `SystemRoleCode` (`ROLE_IS_SYSTEM`), `UnknownPermissionCode`, `InvalidNameCode` (lines 11–18). `Create(name, description, permissions)` (54–59), `Normalize` (69–73), `Update` (75–81, calls `EnsureNotSystem` first), `EnsureCanBeDeleted` (83), `PermissionCodes` (52, ordinal-sorted), `ReplacePermissions` (110–125, throws `DomainException(UNKNOWN_PERMISSION)`). `RolePermission.Permission` (lines 147–163).
2. `src/CustomerSupportCrm.Domain/Roles/Permissions.cs` — lines 7–37. Constants such as `RolesManage`, `TicketsAssign`, `All` (26–32), `IsKnown` (36).
3. `src/CustomerSupportCrm.Domain/Roles/RoleId.cs` — lines 3–8. `readonly record struct RoleId(Guid Value)`; `ToString()` returns the Guid.
4. `src/CustomerSupportCrm.Application/Abstractions/Authorization/PolicyNames.cs` — line 10, `RolesRead = "policy:roles.read"`.
5. `src/CustomerSupportCrm.Infrastructure/Authorization/AuthorizationSetup.cs` — lines 18–27. Each permission code is also a policy name, so `.RequireAuthorization(Permissions.RolesManage)` works. `RolesRead` requires a `permission` claim of `roles.manage` or `users.manage`.
6. `src/CustomerSupportCrm.Application/Abstractions/Persistence/IApplicationDbContext.cs` — lines 11–22. `Users`, `Roles`, `SaveChangesAsync`.
7. `src/CustomerSupportCrm.Application/Abstractions/Auditing/IAuditTrail.cs` — lines 10–20. `Record(action, entityType, entityId, oldValues, newValues, actorUserId)`; call it before `SaveChangesAsync`.
8. `src/CustomerSupportCrm.Domain/Audit/AuditActions.cs` — lines 18–20 (`RoleCreated`, `RoleUpdated`, `RoleDeleted`) and line 26 (`AuditEntityTypes.Role`).
9. `src/CustomerSupportCrm.Application/Abstractions/Http/ApiResults.cs` — lines 12–20. `Ok`, `Created(location, data)`, `Success()`.
10. `src/CustomerSupportCrm.Application/Abstractions/Http/IEndpoint.cs` (Phase 1) — lines 9–12. Endpoints are found by `AddEndpoints` in `src/CustomerSupportCrm.Application/DependencyInjection.cs` lines 31–39. Validators are registered at line 23 with `includeInternalTypes: true`.
11. `src/CustomerSupportCrm.Api/Endpoints/EndpointExtensions.cs` — line 15. Every slice is mapped under `/api/v1` with `RequireAuthorization()`.
12. `src/CustomerSupportCrm.Api/Middleware/GlobalExceptionHandler.cs` — lines 26–41 map exceptions to statuses (`ValidationException` → 400, `NotFoundException` → 404, `ConflictException` → 409, `DomainException` → 422, a unique-index race → 409 `CONFLICT`). Lines 71–91: a `WithErrorCode("UPPER_SNAKE")` code passes through unchanged, and the property path is camel-cased (`permissions[0]`).
13. `src/CustomerSupportCrm.Application/Common/Exceptions/ConflictException.cs` and `NotFoundException.cs` (Phase 1) — `(string code, string message)` primary constructors.
14. `src/CustomerSupportCrm.Infrastructure/Persistence/Configurations/UserConfiguration.cs` — lines 45–49. The `user_roles.role_id` FK uses `DeleteBehavior.Restrict`, so the database also blocks deleting an assigned role.
15. `src/CustomerSupportCrm.Infrastructure/Persistence/Seed/DatabaseInitializer.cs` — lines 22–35 (default `Manager`/`Agent` roles, not system) and 45–74 (the Administrator is re-synced with `GrantAllPermissions`).
16. `src/CustomerSupportCrm.Application/Features/Users/Common/UserErrors.cs` — lines 3–10. Precedent for a static error-code class.

---

## Backend Tasks

Delete the placeholders `src/CustomerSupportCrm.Application/Features/Roles/.gitkeep` and `src/CustomerSupportCrm.Application/Features/Permissions/.gitkeep`. The `Permissions` folder stays empty (see deviations).

### 1 — Contracts

Create file: `src/CustomerSupportCrm.Contracts/Roles/RoleContracts.cs` (17 lines)

```csharp
public sealed record CreateRoleRequest(string Name, string? Description, IReadOnlyList<string> Permissions);
public sealed record UpdateRoleRequest(string Name, string? Description, IReadOnlyList<string> Permissions);
public sealed record RoleResponse(Guid Id, string Name, string? Description, bool IsSystem, IReadOnlyList<string> Permissions, int UserCount);
/// <param name="Code">Permission code, e.g. tickets.assign.</param>
/// <param name="Group">Feature group, e.g. tickets.</param>
public sealed record PermissionResponse(string Code, string Group);
```

### 2 — Shared role helpers

Create file: `src/CustomerSupportCrm.Application/Features/Roles/Common/RoleQueries.cs` (42 lines)

- `public static class RoleErrors` with `RoleNotFound = "ROLE_NOT_FOUND"`, `RoleNameTaken = "ROLE_NAME_TAKEN"` and `RoleInUse = "ROLE_IN_USE"` (lines 9–14).
- `internal static class RoleQueries` (lines 16–42):

```csharp
public static IQueryable<RoleResponse> ProjectToResponse(this IQueryable<Role> roles, IApplicationDbContext db);
// anonymous projection: Permissions = r.Permissions.Select(p => p.Permission).OrderBy(p => p).ToList(),
// UserCount = db.Users.Count(u => u.Roles.Any(m => m.RoleId == r.Id)); then new RoleResponse(r.Id.Value, ...)
public static async Task<RoleResponse> GetResponseAsync(IApplicationDbContext db, RoleId roleId, CancellationToken ct); // AsNoTracking, ?? throw NotFound()
public static NotFoundException NotFound() => new(RoleErrors.RoleNotFound, "The role was not found.");
public static Task<bool> NameTakenAsync(IApplicationDbContext db, string name, RoleId? excluding, CancellationToken ct); // compares Role.Normalize(name) to NormalizedName
```

Create file: `src/CustomerSupportCrm.Application/Features/Roles/Common/RoleDefinitionValidator.cs` (27 lines)

```csharp
public interface IRoleDefinition { string Name { get; } string? Description { get; } IReadOnlyList<string> Permissions { get; } }

internal abstract class RoleDefinitionValidator<T> : AbstractValidator<T> where T : IRoleDefinition
{
    protected RoleDefinitionValidator()
    {
        RuleFor(role => role.Name).NotEmpty().MaximumLength(Role.NameMaxLength);
        RuleFor(role => role.Description).MaximumLength(Role.DescriptionMaxLength);
        RuleFor(role => role.Permissions).NotNull();
        RuleForEach(role => role.Permissions).Must(Permissions.IsKnown).WithErrorCode(Role.UnknownPermissionCode);
    }
}
```

### 3 — Create

- Create file: `Features/Roles/Create/CreateRoleCommand.cs`. `public sealed record CreateRoleCommand(string Name, string? Description, IReadOnlyList<string> Permissions) : IRequest<RoleResponse>, IRoleDefinition;`
- Create file: `Features/Roles/Create/CreateRoleValidator.cs`. `internal sealed class CreateRoleValidator : RoleDefinitionValidator<CreateRoleCommand>;`
- Create file: `Features/Roles/Create/CreateRoleHandler.cs`. `(IApplicationDbContext db, IAuditTrail audit)`:
  1. If `NameTakenAsync(db, request.Name, excluding: null, ct)`, throw `new ConflictException(RoleErrors.RoleNameTaken, "A role with this name already exists.")`.
  2. Call `Role.Create(...)`, then `db.Roles.Add(role)`.
  3. Call `audit.Record(AuditActions.RoleCreated, AuditEntityTypes.Role, role.Id.ToString(), newValues: new { role.Name, role.Description, permissions = role.PermissionCodes })`.
  4. Call `SaveChangesAsync`, then return `RoleQueries.GetResponseAsync(db, role.Id, ct)`.
- Create file: `Features/Roles/Create/CreateRoleEndpoint.cs`:

```csharp
app.MapPost("/roles", async (CreateRoleRequest request, ISender sender, CancellationToken cancellationToken) =>
    {
        var role = await sender.Send(new CreateRoleCommand(request.Name, request.Description, request.Permissions ?? []), cancellationToken);
        return ApiResults.Created($"/api/v1/roles/{role.Id}", role);
    })
    .RequireAuthorization(Permissions.RolesManage)
    .WithName("CreateRole").WithTags("Roles")
    .Produces<ApiResponse<RoleResponse>>(StatusCodes.Status201Created);
```

### 4 — Get by id and list

- `Features/Roles/GetById/GetRoleByIdQuery.cs`: `public sealed record GetRoleByIdQuery(Guid RoleId) : IRequest<RoleResponse>;`
- `Features/Roles/GetById/GetRoleByIdHandler.cs`: returns `RoleQueries.GetResponseAsync(db, new RoleId(request.RoleId), ct)`.
- `Features/Roles/GetById/GetRoleByIdEndpoint.cs`: `MapGet("/roles/{id:guid}", ...)` → `ApiResults.Ok(...)`, `.RequireAuthorization(PolicyNames.RolesRead)`, name `GetRoleById`, tag `Roles`.
- `Features/Roles/List/ListRolesQuery.cs`: `public sealed record ListRolesQuery : IRequest<IReadOnlyList<RoleResponse>>;` with the XML comment "All roles, unpaged: the set is small and curated by administrators."
- `Features/Roles/List/ListRolesHandler.cs`: `db.Roles.AsNoTracking().OrderBy(r => r.Name).ProjectToResponse(db).ToListAsync(ct)`.
- `Features/Roles/List/ListRolesEndpoint.cs`: `MapGet("/roles", ...)`, `.RequireAuthorization(PolicyNames.RolesRead)`, name `ListRoles`, `.Produces<ApiResponse<IReadOnlyList<RoleResponse>>>()`.

### 5 — Update

- `Features/Roles/Update/UpdateRoleCommand.cs`: `public sealed record UpdateRoleCommand(Guid RoleId, string Name, string? Description, IReadOnlyList<string> Permissions) : IRequest<RoleResponse>, IRoleDefinition;`
- `Features/Roles/Update/UpdateRoleValidator.cs`: `internal sealed class UpdateRoleValidator : RoleDefinitionValidator<UpdateRoleCommand>;`
- `Features/Roles/Update/UpdateRoleHandler.cs` (43 lines). Put this XML comment on the class: "Permission changes reach signed-in users on their next token refresh":
  1. Load the role with `db.Roles.Include(r => r.Permissions).SingleOrDefaultAsync(r => r.Id == roleId, ct)`, or `throw RoleQueries.NotFound()`.
  2. If `NameTakenAsync(db, request.Name, excluding: roleId, ct)`, throw `ConflictException(RoleNameTaken)`.
  3. Capture `before = new { role.Name, role.Description, permissions = role.PermissionCodes }`, then call `role.Update(request.Name, request.Description, request.Permissions)`. For a system role this throws `ROLE_IS_SYSTEM`.
  4. Call `audit.Record(AuditActions.RoleUpdated, ..., oldValues: before, newValues: ...)`, then `SaveChangesAsync`, then return `GetResponseAsync`.
- `Features/Roles/Update/UpdateRoleEndpoint.cs`: `MapPut("/roles/{id:guid}", (Guid id, UpdateRoleRequest request, ...) => ApiResults.Ok(await sender.Send(new UpdateRoleCommand(id, request.Name, request.Description, request.Permissions ?? []), ct)))`, `.RequireAuthorization(Permissions.RolesManage)`, name `UpdateRole`.

### 6 — Delete

- `Features/Roles/Delete/DeleteRoleCommand.cs`: `public sealed record DeleteRoleCommand(Guid RoleId) : IRequest;`
- `Features/Roles/Delete/DeleteRoleHandler.cs` (36 lines). Class comment: "Deletes an unassigned, non-system role. Unassign users first."
  1. Load the role with `Include(r => r.Permissions)`, or throw `NotFound()`.
  2. Call `role.EnsureCanBeDeleted()`. This throws `ROLE_IS_SYSTEM` (422).
  3. If `db.Users.AnyAsync(u => u.Roles.Any(m => m.RoleId == roleId), ct)`, throw `new ConflictException(RoleErrors.RoleInUse, "The role is assigned to users.")`.
  4. Call `db.Roles.Remove(role)`, then `audit.Record(AuditActions.RoleDeleted, ..., oldValues: new { role.Name, role.Description, permissions = role.PermissionCodes })`, then `SaveChangesAsync`.
- `Features/Roles/Delete/DeleteRoleEndpoint.cs`: `MapDelete("/roles/{id:guid}", ...)` → `ApiResults.Success()`, `.RequireAuthorization(Permissions.RolesManage)`, `.Produces<ApiResponse<object?>>()`.

### 7 — Permission catalog

Create file: `src/CustomerSupportCrm.Application/Features/Roles/ListPermissions/ListPermissionsEndpoint.cs` (28 lines). Do not create a query or handler. The class comment explains that MediatR would only add indirection for static data.

```csharp
private static readonly IReadOnlyList<PermissionResponse> Catalog =
    [.. Permissions.All.Select(code => new PermissionResponse(code, code[..code.IndexOf('.', StringComparison.Ordinal)]))];

public void MapEndpoint(IEndpointRouteBuilder app) =>
    app.MapGet("/permissions", () => ApiResults.Ok(Catalog))
        .RequireAuthorization(PolicyNames.RolesRead)
        .WithName("ListPermissions").WithTags("Roles")
        .Produces<ApiResponse<IReadOnlyList<PermissionResponse>>>();
```

### 8 — Localized messages

File: `src/CustomerSupportCrm.Application/Resources/Messages.resx`, lines 87–101. Add `ROLE_NOT_FOUND`, `ROLE_NAME_TAKEN`, `ROLE_IN_USE`, `ROLE_IS_SYSTEM` ("System roles cannot be modified or deleted.") and `UNKNOWN_PERMISSION` ("One or more permissions do not exist."). Lines 108–110 hold `INVALID_ROLE_NAME` (the domain code from `Role.Rename`). Add the same keys at the same lines in `Messages.ar.resx` with Arabic values. `GlobalExceptionHandler.Single` localizes `AppException`/`DomainException` codes by key through `ErrorResponseWriter.Localize` (`ErrorResponseWriter.cs` line 37).

### 9 — Docs

File: `docs/api-contract.md`, lines 98 and 101–104 in the "Feature error codes (Phase 2)" table: add `ROLE_NOT_FOUND` 404, `ROLE_NAME_TAKEN` 409, `ROLE_IN_USE` 409, `ROLE_IS_SYSTEM` 422 and `UNKNOWN_PERMISSION` 400 ("Field error on `permissions`").

---

## Edge Cases & Failure Modes

- **Name differing only in case or whitespace** (`" reviewers "` vs `"REVIEWERS"`): `NameTakenAsync` compares `Role.Normalize` output, so the request gets 409 `ROLE_NAME_TAKEN` (`RoleQueries.cs` 37–41). If two requests race past the check, the unique index `NormalizedName` (`RoleConfiguration.cs` line 17) raises a unique violation, which becomes 409 `CONFLICT` (`GlobalExceptionHandler.cs` line 35).
- **Update keeps the same name**: `excluding: roleId` stops the role from clashing with itself (`UpdateRoleHandler.cs` line 25).
- **Unknown permission code**: `RuleForEach ... WithErrorCode(UNKNOWN_PERMISSION)` returns 400 with field `permissions[i]` (`RoleDefinitionValidator.cs` line 25). The field message is FluentValidation's default predicate text, not the resx value. If validation is bypassed, `Role.ReplacePermissions` throws `DomainException` → 422 (`Role.cs` 114–118).
- **`permissions` omitted or null in JSON**: the endpoints coerce it to `[]` (`request.Permissions ?? []`), so a role can have no permissions.
- **Duplicate codes in the request**: removed by `Distinct(StringComparer.Ordinal)` (`Role.cs` line 112).
- **System role update or delete** (Administrator): `Role.Update`/`EnsureCanBeDeleted` → `EnsureNotSystem` returns 422 `ROLE_IS_SYSTEM` (`Role.cs` 127–133). In the update handler the name-uniqueness check runs first, so renaming Administrator to an existing name returns 409 before 422.
- **Delete an assigned role**: 409 `ROLE_IN_USE` (`DeleteRoleHandler.cs` 23–26). The `Restrict` FK (`UserConfiguration.cs` 45–49) is the database-level backstop.
- **Unknown or malformed id**: a well-formed Guid that does not exist returns 404 `ROLE_NOT_FOUND`. A non-Guid does not match the `{id:guid}` constraint, so the request gets 404 `NOT_FOUND`.
- **Concurrent edits**: `HasXminConcurrencyToken` (`RoleConfiguration.cs` line 28) raises `DbUpdateConcurrencyException`, which becomes 409 `CONFLICT`.
- **Caller has `users.manage` only**: can read roles and the catalog (`RolesRead`) but gets 403 on POST/PUT/DELETE.
- **Permission change propagation**: existing access tokens keep their old claims until refresh (the class comment on `UpdateRoleHandler`). No invalidation is added here.
- **Catalog growth**: `Catalog` is built once from `Permissions.All`. `DatabaseInitializer.GrantAllPermissions` gives new codes to the Administrator on the next start.

---

## Test Plan

Tests are out of scope under the standing directive: **no tests are added, changed or removed**. The existing tests in `0f87e2d` that cover this story are read-only references:

1. `tests/CustomerSupportCrm.Domain.Tests/Roles/RoleTests.cs` (unit): `CreateTrimsNameAndNormalizes`, `RejectsUnknownPermission`, `UpdateReplacesPermissions`, `AdministratorHoldsEveryPermission`, `SystemRoleCannotBeUpdatedOrDeleted`, `PermissionCatalogCodesAreUniqueAndGrouped`.
2. `tests/CustomerSupportCrm.IntegrationTests/RoleManagementTests.cs` (integration, PostgreSQL): `CreateUpdateAndDeleteRole` (lines 13–36), `RoleNamesAreUniqueCaseInsensitively` (38–51), `SystemRoleIsImmutable` (53–65), `AssignedRoleCannotBeDeleted` (67–78), `UnknownPermissionIsAValidationError` (80–89), `PermissionChangesReachUsersOnRefresh` (91–110), `PermissionCatalogIsListed` (112–121). The commit message says the integration suite had not yet been run.

---

## Verification Steps

1. **Backend builds:** from `customer-support-crm-api/`, run `dotnet build`. Expect 0 warnings and 0 errors.
2. **Regression:** run `dotnet test --filter "FullyQualifiedName!~IntegrationTests"`. Expect all tests to pass.
3. **Run the API:** `dotnet run --project src/CustomerSupportCrm.Api --launch-profile http` (listens on `http://localhost:5000`). You need a bootstrap admin (see [01-story-identity-domain-persistence.md](01-story-identity-domain-persistence.md)).
4. **Log in:** `curl -s -X POST http://localhost:5000/api/v1/auth/login -H "Content-Type: application/json" -d '{"email":"<admin>","password":"<pwd>"}'` and copy `data.accessToken` into `$T`.
5. **Catalog:** `curl -s http://localhost:5000/api/v1/permissions -H "Authorization: Bearer $T"`. Expect 13 items, including `{"code":"tickets.assign","group":"tickets"}`.
6. **Create:** `curl -s -i -X POST http://localhost:5000/api/v1/roles -H "Authorization: Bearer $T" -H "Content-Type: application/json" -d '{"name":"Reviewers","permissions":["tickets.view"]}'`. Expect 201, a `Location: /api/v1/roles/{id}` header and `userCount: 0`. Repeat with `"REVIEWERS"` and expect 409 `ROLE_NAME_TAKEN`.
7. **Unknown code:** POST with `"permissions":["tickets.teleport"]`. Expect 400 with `errors[0].code = UNKNOWN_PERMISSION` and `field = permissions[0]`.
8. **Update:** `PUT /api/v1/roles/{id}` with `{"name":"Reviewers","permissions":["tickets.view","reports.view"]}`. Expect 200 and sorted permissions.
9. **System role:** `PUT` and `DELETE` on the Administrator id (from `GET /api/v1/roles`). Expect 422 `ROLE_IS_SYSTEM`.
10. **In use / delete:** assign the role to a user through `POST /api/v1/users`, then `DELETE`. Expect 409 `ROLE_IN_USE`. On an unassigned role, `DELETE` returns 200, and a following `GET` returns 404 `ROLE_NOT_FOUND`.
11. **Localization:** repeat step 9 with `-H "Accept-Language: ar"`. Expect an Arabic `errors[0].message`.

---

## Done Criteria

- [x] Permission catalog endpoint returns every code from `Permissions.All` with its group. (Deviation: flat `{code, group}` list, no localized name/description.)
- [x] Role CRUD works with the standard envelope and the listed permissions. Permission assignment is part of `PUT /roles/{id}` (deviation: no separate AssignPermissions endpoint). Reads use `policy:roles.read`, writes use `roles.manage`.
- [x] System-role, duplicate-name, unknown-permission and role-in-use rules return `ROLE_IS_SYSTEM` (422), `ROLE_NAME_TAKEN` (409), `UNKNOWN_PERMISSION` (400, `permissions[i]`) and `ROLE_IN_USE` (409). `ROLE_NOT_FOUND` returns 404.
- [x] Messages for every role code exist in `Messages.resx` and `Messages.ar.resx`.
- [x] Nothing changed in `tests/`, `docker-compose.yml`, `deploy/` or `.github/`.
- [x] `dotnet build` passes with zero warnings.

**STOP HERE. Report to the user and wait for confirmation before proceeding to Story 06.**
