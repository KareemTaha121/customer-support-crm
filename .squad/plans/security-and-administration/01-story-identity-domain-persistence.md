# Story 01 — Identity domain & persistence (Story: P2-01)

## Prerequisites

- Phase 1 (Backend Platform) completed in `customer-support-crm-api` (commit `c3c7815 feat: add Phase 1 API foundation`).
- Local PostgreSQL reachable at the `Database:ConnectionString` in `src/CustomerSupportCrm.Api/appsettings.Development.json` (only needed for the migration/seed verification steps).
- All paths below are relative to the **`customer-support-crm-api/`** repo root.

---

## Story Goal

Add the identity domain model and its persistence so later Phase 2 stories (login, authorization, user/role management, audit) have something to build on:

1. `User`, `Role`, `UserRole`, `RolePermission`, `RefreshToken` in `Domain/Users`, plus a code-defined `Permissions` catalog.
2. EF Core configurations, DbSets and the first migration **`AddIdentity`**.
3. An idempotent **`IdentitySeeder`** that creates the system roles and, when configured, a bootstrap administrator.
4. `IPasswordHasher` abstraction + implementation (the seeder needs it; P2-02 reuses it).

**Not in scope:** HTTP endpoints, JWT, login, audit log, organization/branch/department scoping. **Do not touch** `docker-compose.yml`, `deploy/`, `.github/` or anything under `tests/`.

---

## Context — Read These Files First

1. `src/CustomerSupportCrm.Domain/Common/DomainException.cs` — lines 1–10. `DomainException(string code, string message)`; every domain invariant below throws this with a stable UPPER_SNAKE code.
2. `src/CustomerSupportCrm.Infrastructure/Persistence/ApplicationDbContext.cs` — whole file (11 lines). `OnModelCreating` already calls `ApplyConfigurationsFromAssembly`, so new `IEntityTypeConfiguration<T>` classes are picked up automatically. Snake_case naming is applied in DI, not here.
3. `src/CustomerSupportCrm.Infrastructure/DependencyInjection.cs` — lines 13–41. `AddInfrastructure` → private `AddPersistence`; follow the `AddOptions<T>().BindConfiguration(...).ValidateDataAnnotations().ValidateOnStart()` pattern at lines 24–27 for new options.
4. `src/CustomerSupportCrm.Infrastructure/Persistence/DatabaseOptions.cs` — lines 1–17. Template for an options class (`SectionName` const, `init` properties, DataAnnotations).
5. `src/CustomerSupportCrm.Api/Program.cs` — lines 11–42. Host setup; the seed command is added between `builder.Build()` (line 22) and the middleware (line 26).
6. `src/CustomerSupportCrm.Application/Common/Exceptions/AppException.cs` — lines 1–10. Application exceptions are separate from `DomainException`; do not mix them in Domain.
7. `Directory.Packages.props` — `ItemGroup Label="Persistence"` and `Label="Application"`. Package versions are central; `.csproj` files use `PackageReference` **without** `Version`.
8. `Directory.Build.props` — lines 1–12. `TreatWarningsAsErrors`, `Nullable`, `AnalysisLevel latest-recommended`: every new file must compile warning-free (seal classes, no unused usings, pass `CancellationToken`).
9. `tests/CustomerSupportCrm.Api.Tests/ApiFactory.cs` — lines 14–30 (**read only, do not edit**). The API test host runs in the **Development** environment against an **unreachable** database. Anything that touches the DB at startup would break these tests — that is why seeding is an explicit command, not a hosted service.
10. `tests/CustomerSupportCrm.IntegrationTests/PostgresApiFactory.cs` — lines 23–27 (**read only**). Integration host uses environment `Test` against an empty, un-migrated database — same constraint.
11. `dotnet-tools.json` — `dotnet-ef` 10.0.12 is the local tool used to create the migration.

---

## Backend Tasks

### 1 — Package

File: `Directory.Packages.props` — add to the `Label="Persistence"` group:

```xml
<PackageVersion Include="Microsoft.Extensions.Identity.Core" Version="10.0.12" />
```

File: `src/CustomerSupportCrm.Infrastructure/CustomerSupportCrm.Infrastructure.csproj` — add `<PackageReference Include="Microsoft.Extensions.Identity.Core" />` to the existing package `ItemGroup` (keep alphabetical order).

### 2 — Permission catalog

Create file: `src/CustomerSupportCrm.Domain/Users/Permissions.cs`

```csharp
namespace CustomerSupportCrm.Domain.Users;

/// <summary>
/// Code-defined permission catalog. Codes are stable and stored in role_permissions;
/// never rename or remove a published code.
/// </summary>
public static class Permissions
{
    public static class Users
    {
        public const string View = "users.view";
        public const string Manage = "users.manage";
    }

    public static class Roles
    {
        public const string View = "roles.view";
        public const string Manage = "roles.manage";
    }

    public static class Audit
    {
        public const string View = "audit.view";
    }

    public static class Tickets
    {
        public const string View = "tickets.view";
        public const string Create = "tickets.create";
        public const string Update = "tickets.update";
        public const string Assign = "tickets.assign";
        public const string Delete = "tickets.delete";
    }

    public static class Customers
    {
        public const string View = "customers.view";
        public const string Create = "customers.create";
        public const string Update = "customers.update";
    }

    public static class Reports
    {
        public const string View = "reports.view";
    }

    public static class Settings
    {
        public const string Manage = "settings.manage";
    }

    public static IReadOnlyList<string> All { get; } =
    [
        Users.View, Users.Manage,
        Roles.View, Roles.Manage,
        Audit.View,
        Tickets.View, Tickets.Create, Tickets.Update, Tickets.Assign, Tickets.Delete,
        Customers.View, Customers.Create, Customers.Update,
        Reports.View,
        Settings.Manage,
    ];

    private static readonly HashSet<string> Defined = new(All, StringComparer.Ordinal);

    public static bool IsDefined(string code) => Defined.Contains(code);
}
```

### 3 — Error codes

Create file: `src/CustomerSupportCrm.Domain/Users/IdentityErrorCodes.cs` — `public static class IdentityErrorCodes` with `const string`s:

| Constant | Value | Thrown when |
|---|---|---|
| `InvalidEmail` | `USER_INVALID_EMAIL` | email empty, > 256 chars, or no `@` |
| `InvalidFullName` | `USER_INVALID_FULL_NAME` | full name empty/whitespace or > 200 chars |
| `UnsupportedCulture` | `USER_UNSUPPORTED_CULTURE` | culture not `en` / `ar` |
| `InvalidRoleName` | `ROLE_INVALID_NAME` | role name empty or > 100 chars |
| `SystemRoleReadOnly` | `SYSTEM_ROLE_READ_ONLY` | rename of a system role |
| `UnknownPermission` | `UNKNOWN_PERMISSION` | `Permissions.IsDefined` is false |
| `RefreshTokenAlreadyRevoked` | `REFRESH_TOKEN_ALREADY_REVOKED` | revoke/replace on a revoked token |

### 4 — User aggregate

Create file: `src/CustomerSupportCrm.Domain/Users/User.cs`

- `public sealed class User` with **private setters** and a private parameterless constructor for EF.
- Properties: `Guid Id`, `string Email`, `string NormalizedEmail`, `string FullName`, `string PasswordHash`, `bool IsActive`, `string PreferredCulture`, `string SecurityStamp`, `int AccessFailedCount`, `DateTimeOffset? LockoutEndsAt`, `DateTimeOffset? LastLoginAt`, `DateTimeOffset CreatedAt`, `DateTimeOffset UpdatedAt`.
- Roles collection: `private readonly List<UserRole> _roles = [];` exposed as `IReadOnlyCollection<UserRole> Roles => _roles;`.
- Constants: `EmailMaxLength = 256`, `FullNameMaxLength = 200`, `CultureMaxLength = 5`, `PasswordHashMaxLength = 512`, `SecurityStampMaxLength = 64`, `SupportedCultures = ["en", "ar"]`.

Behaviors (all take `DateTimeOffset now` where time matters; never call `DateTime.UtcNow`):

```csharp
public static User Create(string email, string fullName, string passwordHash, string culture, DateTimeOffset now);
public static string Normalize(string email) => email.Trim().ToUpperInvariant();
public void Rename(string fullName, DateTimeOffset now);
public void ChangeCulture(string culture, DateTimeOffset now);
public void ChangePasswordHash(string passwordHash, DateTimeOffset now);   // also rotates SecurityStamp
public void Activate(DateTimeOffset now);                                   // clears AccessFailedCount + LockoutEndsAt
public void Deactivate(DateTimeOffset now);
public void AssignRoles(IEnumerable<Guid> roleIds, DateTimeOffset now);     // replace set, distinct, keeps existing rows that stay
public void RecordFailedLogin(DateTimeOffset now, int maxAttempts, TimeSpan lockoutDuration);
public void RecordSuccessfulLogin(DateTimeOffset now);                     // resets counter, clears lockout, sets LastLoginAt
public bool IsLockedOut(DateTimeOffset now) => LockoutEndsAt is { } end && end > now;
```

- `Create`: `Id = Guid.CreateVersion7(now)`, trims email/full name, validates (codes from task 3), `IsActive = true`, `SecurityStamp = Guid.NewGuid().ToString("N")`, `CreatedAt = UpdatedAt = now`.
- `RecordFailedLogin`: increments `AccessFailedCount`; when it reaches `maxAttempts`, set `LockoutEndsAt = now + lockoutDuration` and reset `AccessFailedCount = 0`.
- `ChangePasswordHash`: throws `ArgumentException` on empty hash (programming error, not a domain rule).
- Override `ToString()` is **not** added — do not expose `PasswordHash` / `SecurityStamp` through any formatting.

Create file: `src/CustomerSupportCrm.Domain/Users/UserRole.cs` — `public sealed class UserRole` with `Guid UserId`, `Guid RoleId`; internal constructor used by `User.AssignRoles`; private parameterless ctor for EF.

### 5 — Role aggregate

Create file: `src/CustomerSupportCrm.Domain/Users/Role.cs`

- Properties: `Guid Id`, `string Name`, `string NormalizedName`, `string? Description`, `bool IsSystem`, `DateTimeOffset CreatedAt`, `DateTimeOffset UpdatedAt`; `private readonly List<RolePermission> _permissions = [];` → `IReadOnlyCollection<RolePermission> Permissions`.
- Constants: `NameMaxLength = 100`, `DescriptionMaxLength = 500`.
- System role names as constants: `public const string Administrator = "Administrator"; Supervisor = "Supervisor"; Agent = "Agent";`

```csharp
public static Role Create(string name, string? description, bool isSystem, DateTimeOffset now);
public static string Normalize(string name) => name.Trim().ToUpperInvariant();
public void Rename(string name, string? description, DateTimeOffset now); // throws SYSTEM_ROLE_READ_ONLY when IsSystem and name changes
public void SetPermissions(IEnumerable<string> codes, DateTimeOffset now);  // validates each with Permissions.IsDefined → UNKNOWN_PERMISSION; replace set, distinct
public bool HasPermission(string code);
```

Create file: `src/CustomerSupportCrm.Domain/Users/RolePermission.cs` — `Guid RoleId`, `string PermissionCode` (max 100); internal ctor + private EF ctor.

### 6 — RefreshToken

Create file: `src/CustomerSupportCrm.Domain/Users/RefreshToken.cs`

- Properties: `Guid Id`, `Guid UserId`, `string TokenHash` (64 hex chars, SHA-256), `Guid FamilyId`, `DateTimeOffset CreatedAt`, `DateTimeOffset ExpiresAt`, `DateTimeOffset? RevokedAt`, `string? RevokedReason` (max 100), `Guid? ReplacedByTokenId`, `string? CreatedByIp` (max 45), `string? UserAgent` (max 512, truncate longer values).

```csharp
public static RefreshToken Issue(Guid userId, string tokenHash, Guid familyId, DateTimeOffset now, TimeSpan lifetime, string? ip, string? userAgent);
public bool IsActive(DateTimeOffset now) => RevokedAt is null && ExpiresAt > now;
public void Revoke(DateTimeOffset now, string reason);                // throws REFRESH_TOKEN_ALREADY_REVOKED when already revoked
public void MarkReplaced(Guid newTokenId, DateTimeOffset now);        // sets ReplacedByTokenId and revokes with reason "Rotated"
```

The domain never sees the raw token — only the hash.

### 7 — EF configurations

Create one file per entity in `src/CustomerSupportCrm.Infrastructure/Persistence/Configurations/`, each `internal sealed class XConfiguration : IEntityTypeConfiguration<X>`:

- **`UserConfiguration.cs`** — table `users`; key `Id` with `ValueGeneratedNever()`; `HasMaxLength` from the `User` constants; unique index on `NormalizedEmail`; `HasMany(u => u.Roles).WithOne().HasForeignKey(r => r.UserId).OnDelete(DeleteBehavior.Cascade)`; `Navigation(u => u.Roles).UsePropertyAccessMode(PropertyAccessMode.Field)`; `Property(u => u.SecurityStamp).IsConcurrencyToken()`.
- **`UserRoleConfiguration.cs`** — table `user_roles`; composite key `(UserId, RoleId)`; FK to `Role` on `RoleId` with `OnDelete(DeleteBehavior.Restrict)`; index on `RoleId`.
- **`RoleConfiguration.cs`** — table `roles`; unique index on `NormalizedName`; `HasMany(r => r.Permissions).WithOne().HasForeignKey(p => p.RoleId).OnDelete(DeleteBehavior.Cascade)`; field access mode on the navigation.
- **`RolePermissionConfiguration.cs`** — table `role_permissions`; composite key `(RoleId, PermissionCode)`; `PermissionCode` max 100.
- **`RefreshTokenConfiguration.cs`** — table `refresh_tokens`; unique index on `TokenHash`; indexes on `UserId` and `FamilyId`; FK to `User` on `UserId` with `OnDelete(DeleteBehavior.Cascade)` (no navigation on `User`).

File: `src/CustomerSupportCrm.Infrastructure/Persistence/ApplicationDbContext.cs` — add:

```csharp
public DbSet<User> Users => Set<User>();
public DbSet<Role> Roles => Set<Role>();
public DbSet<RefreshToken> RefreshTokens => Set<RefreshToken>();
```

### 8 — Password hasher

Create file: `src/CustomerSupportCrm.Application/Abstractions/Authentication/IPasswordHasher.cs`

```csharp
namespace CustomerSupportCrm.Application.Abstractions.Authentication;

public interface IPasswordHasher
{
    string Hash(string password);
    PasswordVerification Verify(string passwordHash, string password);
}

public enum PasswordVerification
{
    Failed,
    Success,
    SuccessRehashNeeded,
}
```

Delete the placeholder `src/CustomerSupportCrm.Application/Abstractions/Authentication/.gitkeep`.

Create file: `src/CustomerSupportCrm.Infrastructure/Authentication/IdentityPasswordHasher.cs`:

```csharp
using CustomerSupportCrm.Application.Abstractions.Authentication;
using Microsoft.AspNetCore.Identity;

namespace CustomerSupportCrm.Infrastructure.Authentication;

/// <summary>ASP.NET Core Identity's PBKDF2 (V3) hasher. The user argument is unused by the algorithm.</summary>
internal sealed class IdentityPasswordHasher : IPasswordHasher
{
    private static readonly object Subject = new();
    private readonly PasswordHasher<object> _inner = new();

    public string Hash(string password) => _inner.HashPassword(Subject, password);

    public PasswordVerification Verify(string passwordHash, string password) =>
        _inner.VerifyHashedPassword(Subject, passwordHash, password) switch
        {
            PasswordVerificationResult.Success => PasswordVerification.Success,
            PasswordVerificationResult.SuccessRehashNeeded => PasswordVerification.SuccessRehashNeeded,
            _ => PasswordVerification.Failed,
        };
}
```

Delete `src/CustomerSupportCrm.Infrastructure/Authentication/.gitkeep`.

### 9 — Seeder

Create file: `src/CustomerSupportCrm.Infrastructure/Persistence/Seed/BootstrapAdminOptions.cs`

```csharp
public sealed class BootstrapAdminOptions
{
    public const string SectionName = "Identity:BootstrapAdmin";

    public string? Email { get; init; }
    public string? Password { get; init; }
    public string FullName { get; init; } = "System Administrator";
}
```

Create file: `src/CustomerSupportCrm.Infrastructure/Persistence/Seed/IdentitySeeder.cs` — `internal sealed class IdentitySeeder(ApplicationDbContext db, IPasswordHasher hasher, TimeProvider time, IOptions<BootstrapAdminOptions> admin, ILogger<IdentitySeeder> logger)` with `public async Task SeedAsync(CancellationToken cancellationToken)`:

1. For each system role, load by `NormalizedName`; create when missing (`isSystem: true`).
   - **Administrator** → `Permissions.All` (re-applied every run so new codes reach admins).
   - **Supervisor** → all `tickets.*`, all `customers.*`, `reports.view`, `users.view`, `audit.view` — **only on creation** (admins may edit it later in P2-05).
   - **Agent** → `tickets.view`, `tickets.create`, `tickets.update`, `customers.view`, `customers.create`, `customers.update` — **only on creation**.
2. `SaveChangesAsync`.
3. If `await db.Users.AnyAsync(ct)` is true → log "Users exist; bootstrap admin skipped" and return.
4. If `Email` or `Password` is null/whitespace → log a **warning** "Identity:BootstrapAdmin not configured; no administrator created" and return.
5. If `Password.Length < 10` → throw `InvalidOperationException("Identity:BootstrapAdmin:Password must be at least 10 characters.")`.
6. Create the user (`culture "en"`), `AssignRoles([administrator.Id])`, save. Log `"Bootstrap administrator {Email} created"` — **never** log the password or hash.

Create file: `src/CustomerSupportCrm.Infrastructure/Persistence/Seed/SeedingExtensions.cs`

```csharp
public static class SeedingExtensions
{
    public const string SeedCommand = "seed";

    public static async Task SeedIdentityAsync(this IServiceProvider services, CancellationToken cancellationToken = default)
    {
        await using var scope = services.CreateAsyncScope();
        await scope.ServiceProvider.GetRequiredService<IdentitySeeder>().SeedAsync(cancellationToken);
    }
}
```

Delete `src/CustomerSupportCrm.Infrastructure/Persistence/Seed/.gitkeep`.

### 10 — DI registration

File: `src/CustomerSupportCrm.Infrastructure/DependencyInjection.cs`

- In `AddInfrastructure`, after `services.AddPersistence();` (line 17), call a new private extension `services.AddIdentityServices();`. **Do not** name it `AddIdentityCore` — that name clashes with ASP.NET Core Identity's own extension.
- `AddIdentityServices` body:

```csharp
services.TryAddSingleton(TimeProvider.System);
services.AddSingleton<IPasswordHasher, IdentityPasswordHasher>();
services.AddOptions<BootstrapAdminOptions>().BindConfiguration(BootstrapAdminOptions.SectionName);
services.AddScoped<IdentitySeeder>();
```

(No `ValidateOnStart` for `BootstrapAdminOptions` — it is optional.)

### 11 — Seed command in the host

File: `src/CustomerSupportCrm.Api/Program.cs` — directly after `var app = builder.Build();` (line 22) insert:

```csharp
if (args.Contains(SeedingExtensions.SeedCommand, StringComparer.OrdinalIgnoreCase))
{
    await app.Services.SeedIdentityAsync();
    return;
}
```

Add `using CustomerSupportCrm.Infrastructure.Persistence.Seed;`. Do **not** seed automatically on normal startup (see Context items 9–10).

### 12 — Configuration

File: `src/CustomerSupportCrm.Api/appsettings.json` — add (values intentionally empty):

```json
"Identity": {
  "BootstrapAdmin": {
    "Email": "",
    "Password": ""
  }
}
```

Do **not** put credentials in `appsettings.Development.json`.

### 13 — Migration

Run from the repo root (requires the solution to build):

```bash
dotnet tool restore
dotnet ef migrations add AddIdentity --project src/CustomerSupportCrm.Infrastructure --startup-project src/CustomerSupportCrm.Api --output-dir Persistence/Migrations
```

Delete `src/CustomerSupportCrm.Infrastructure/Persistence/Migrations/.gitkeep` once the migration files exist. Review the generated migration: tables `users`, `user_roles`, `roles`, `role_permissions`, `refresh_tokens`; unique indexes `ix_users_normalized_email`, `ix_roles_normalized_name`, `ix_refresh_tokens_token_hash`; `timestamp with time zone` for every `DateTimeOffset`.

Generated migration files are excluded from style fixes; if the build fails on analyzer warnings inside `Migrations/`, add to `.editorconfig`:

```ini
[src/CustomerSupportCrm.Infrastructure/Persistence/Migrations/**.cs]
generated_code = true
dotnet_analyzer_diagnostic.severity = none
```

### 14 — Docs

File: `docs/development.md` — add a **"Seed identity data"** section after "Migrations":

```bash
dotnet user-secrets --project src/CustomerSupportCrm.Api set "Identity:BootstrapAdmin:Email" "admin@example.com"
dotnet user-secrets --project src/CustomerSupportCrm.Api set "Identity:BootstrapAdmin:Password" "<min 10 chars>"
dotnet run --project src/CustomerSupportCrm.Api -- seed
```

State that seeding is idempotent and only creates the admin when no users exist.

---

## Edge Cases & Failure Modes

- **Email casing / whitespace** (`" Admin@X.com "`) — `User.Create` trims; uniqueness enforced on `NormalizedEmail` via the unique index (task 7). Duplicate insert surfaces as `DbUpdateException`; P2-04 maps it to `EMAIL_ALREADY_EXISTS`.
- **Unknown permission code** — `Role.SetPermissions` throws `DomainException(UNKNOWN_PERMISSION)` (task 5) → 422 through the existing `GlobalExceptionHandler`.
- **Renaming a system role** — throws `SYSTEM_ROLE_READ_ONLY` (task 5).
- **Lockout arithmetic** — `RecordFailedLogin` with `maxAttempts = 5`: 5th failure locks and resets the counter; `IsLockedOut(now)` false once `now >= LockoutEndsAt` (task 4).
- **Seed run twice** — roles found by `NormalizedName`, users exist → nothing created (task 9 steps 1–3).
- **Seed with empty config** — warning logged, no user, exit code 0 (task 9 step 4).
- **Seed with short password** — `InvalidOperationException`, non-zero exit (task 9 step 5).
- **Seed before migration** — `relation "roles" does not exist` from Npgsql; fix by running `dotnet ef database update` first (documented in task 14).
- **API/Integration test hosts** — no DB access at startup because seeding only runs with the `seed` argument (task 11). Health endpoints and existing tests keep working.
- **Concurrent password change** — `SecurityStamp` is a concurrency token (task 7); conflicting updates raise `DbUpdateConcurrencyException` (handled by callers in later stories).
- **Long user agent** — truncated to 512 chars in `RefreshToken.Issue` (task 6) instead of failing the insert.

---

## Test Plan

Test projects are **out of scope** for this story (`tests/` must not be modified). No tests are added, changed or removed. Existing suites must stay green:

1. `CustomerSupportCrm.Application.Tests` — unchanged, must pass.
2. `CustomerSupportCrm.Api.Tests` — unchanged, must pass (proves no DB access at startup).
3. Domain and seeder behaviour is verified manually through the Verification Steps below; automated tests for `User`, `Role`, `RefreshToken` and `IdentitySeeder` are deferred to a dedicated testing story.

---

## Migration / Rollback

- **Apply:** `dotnet ef database update --project src/CustomerSupportCrm.Infrastructure --startup-project src/CustomerSupportCrm.Api`.
- **Rollback:** `dotnet ef database update 0 --project src/CustomerSupportCrm.Infrastructure --startup-project src/CustomerSupportCrm.Api`, then `dotnet ef migrations remove ...` with the same flags.
- **Half-applied state:** EF migrations run in a transaction on PostgreSQL; a failure leaves no partial tables. If `__EFMigrationsHistory` lists `AddIdentity` but tables are missing, drop the history row and re-apply.

---

## Verification Steps

1. **Backend builds:** from `customer-support-crm-api/` run `dotnet build` — zero warnings, zero errors.
2. **Regression:** `dotnet test --filter "FullyQualifiedName!~IntegrationTests"` — all existing tests pass.
3. **Migration:** `dotnet ef database update --project src/CustomerSupportCrm.Infrastructure --startup-project src/CustomerSupportCrm.Api` — succeeds; `\dt` in psql shows the five tables.
4. **Seed (empty config):** `dotnet run --project src/CustomerSupportCrm.Api -- seed` — exits 0, logs the "not configured" warning, `roles` has 3 rows, `users` has 0.
5. **Seed (configured):** set the two user-secrets from task 14, run the seed again — one user with the Administrator role; `role_permissions` for Administrator has 15 rows.
6. **Idempotency:** run the seed a third time — row counts unchanged.
7. **No secrets in logs:** search the console output of steps 4–6 for the password value and for `AQAAAA` (Identity hash prefix) — no matches.
8. **Normal startup:** `dotnet run --project src/CustomerSupportCrm.Api --launch-profile https` — starts, `/health/ready` returns `Healthy`, no seeding log lines.

---

## Done Criteria

- [ ] `User`, `UserRole`, `Role`, `RolePermission`, `RefreshToken`, `Permissions`, `IdentityErrorCodes` exist under `src/CustomerSupportCrm.Domain/Users/` with no framework references.
- [ ] `Permissions.All` contains the 15 codes; `IsDefined` works.
- [ ] Five EF configurations exist with the listed tables, keys and unique indexes; DbSets added.
- [ ] `AddIdentity` migration exists and applies cleanly.
- [ ] `IPasswordHasher` + `IdentityPasswordHasher` registered; `TimeProvider.System` registered.
- [ ] `dotnet run ... -- seed` is idempotent, creates 3 system roles and the admin only when configured and no users exist.
- [ ] No password, hash or security stamp is logged.
- [ ] Normal startup and existing tests do not touch the database.
- [ ] Nothing changed in `tests/`, `docker-compose.yml`, `deploy/` or `.github/`.
- [ ] `dotnet build` passes with zero warnings.

**STOP HERE. Report to the user and wait for confirmation before proceeding to Story 02.**
