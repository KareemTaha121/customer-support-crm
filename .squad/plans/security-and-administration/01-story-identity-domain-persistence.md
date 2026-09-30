# Story 01 — Identity domain & persistence (Story: P2-01)

> As-built plan: revised after implementation in `customer-support-crm-api` commit `0f87e2d`; paths and line numbers refer to that commit.

## Prerequisites

- Phase 1 (Backend Platform) completed in `customer-support-crm-api` (commit `c3c7815 feat: add Phase 1 API foundation`).
- Local PostgreSQL reachable at the `Database:ConnectionString` in `src/CustomerSupportCrm.Api/appsettings.Development.json` (only needed for the migration/seed verification steps).
- Next: [02-story-authentication-jwt-and-refresh-tokens.md](02-story-authentication-jwt-and-refresh-tokens.md).
- All paths below are relative to the **`customer-support-crm-api/`** repo root.

---

## Story Goal

Add the identity domain model and its persistence so later Phase 2 stories (login, authorization, user/role management, audit) have something to build on:

1. `User` + `UserRole` and `RefreshToken` in `Domain/Users`; `Role` + `RolePermission` and the code-defined `Permissions` catalog in `Domain/Roles`; strongly typed `UserId` / `RoleId`; the `EmailAddress` value object; `Entity<TId>` and `IAuditableEntity` base types.
2. `IApplicationDbContext`, EF Core configurations, DbSets, strongly typed id converters, the `xmin` concurrency convention, the `AuditableEntityInterceptor`, and the first migration **`InitialIdentity`**.
3. An idempotent **`DatabaseInitializer`** that migrates, seeds the Administrator system role plus Manager/Agent defaults, and creates a bootstrap administrator from `Bootstrap:*` when no users exist.
4. `IPasswordHasher` abstraction + `IdentityPasswordHasher` (the initializer needs it; P2-02 reuses it).

**Deviations from the intake (follow the real code):**

| Intake | As built (`0f87e2d`) |
|---|---|
| Everything in `Domain/Users` | `Role`, `RolePermission`, `RoleId`, `Permissions` live in `Domain/Roles`; `EmailAddress` in `Domain/Shared`; `Entity<TId>` / `IAuditableEntity` in `Domain/Common`. |
| `Guid` ids | Strongly typed `UserId` / `RoleId` (`readonly record struct`, UUIDv7), stored as `uuid` through converters in `ConfigureConventions`. `RefreshToken` uses `Entity<Guid>`. |
| Nested catalog with 15 codes (`users.view`, `roles.view`, …), `IsDefined` | 13 **flat** constants (`TicketsView`, `UsersManage`, …); no `users.view` / `roles.view`; method is `IsKnown`. |
| `Email` + `NormalizedEmail`, `FullName`, `PreferredCulture`, `SecurityStamp`, `IsActive` bool, `AccessFailedCount` | Single lower-cased `Email` (via `EmailAddress`, max 254) with unique index; `DisplayName`; `UserStatus` enum (`Active`/`Disabled`) with computed `IsActive`; `FailedLoginAttempts`. **No culture and no security stamp.** |
| `RecordFailedLogin(now, maxAttempts, lockoutDuration)` | `RecordFailedLogin(now)`; constants `MaxFailedLoginAttempts = 5`, `LockoutDuration = 15 min` on `User`. |
| `Activate` / `Deactivate` / `AssignRoles` | `Enable` / `Disable` / `SetRoles`; `Create` takes the role ids. |
| `Role.Rename` guard for system roles, `SetPermissions` | `Role.Update(name, description, permissions)` and `EnsureCanBeDeleted()`; both throw `ROLE_IS_SYSTEM` on system roles. `CreateAdministrator()` / `GrantAllPermissions()` for the system role. |
| `RefreshToken.FamilyId`, `MarkReplaced`, string `RevokedReason` | `SessionId`, `UsedAt`, `Rotate(...)` returning the successor, `IsSpent`, enum `RefreshTokenRevocationReason`; `Revoke` is idempotent (keeps first reason). |
| `IdentityErrorCodes` class | Codes are constants on the owning type (`EmailAddress.InvalidCode`, `User.InvalidDisplayNameCode`, `Role.SystemRoleCode`, …). |
| Timestamps passed as `now` into domain methods | `CreatedAt/By`, `UpdatedAt/By` (nullable) are stamped by `AuditableEntityInterceptor` from `TimeProvider` + `ICurrentUser`. |
| `SecurityStamp` concurrency token | PostgreSQL `xmin` row version on `users`, `roles`, `refresh_tokens` (`HasXminConcurrencyToken`). |
| Migration `AddIdentity` | `20260930092037_InitialIdentity` — also creates `audit_logs` (Story 06), because all stories shipped in one commit. |
| `IdentitySeeder`, roles Administrator/Supervisor/Agent, `Identity:BootstrapAdmin`, `seed` command | `DatabaseInitializer` (migrate + seed), roles **Administrator** (only system role), **Manager**, **Agent** (ordinary roles, first run only), section **`Bootstrap`** (`AdminEmail`, `AdminDisplayName`, `AdminPassword`); runs via `--init-database` or `Database:InitializeOnStartup`. |
| No credentials in `appsettings.Development.json`; use user-secrets | Dev-only `Bootstrap` values **are** committed in `appsettings.Development.json` (lines 22–25), with `InitializeOnStartup: true`. |
| `Microsoft.Extensions.Identity.Core` package | Not added: the Infrastructure `FrameworkReference Microsoft.AspNetCore.App` already supplies `PasswordHasher<T>`. |
| `PasswordVerification` enum | `PasswordVerificationResult` (Application), aliased against the Identity enum of the same name. |

**Not in scope:** HTTP endpoints, JWT, login, `ICurrentUser` (Story 03), `AuditLog` / `IAuditTrail` (Story 06), organization/branch/department scoping. **Do not touch** `docker-compose.yml`, `deploy/`, `.github/` or anything under `tests/`.

---

## Context — Read These Files First

1. `src/CustomerSupportCrm.Domain/Common/DomainException.cs` (c3c7815) — lines 1–10. `DomainException(string code, string message)`; every domain invariant below throws this with a stable UPPER_SNAKE code.
2. `src/CustomerSupportCrm.Infrastructure/Persistence/ApplicationDbContext.cs` (c3c7815) — whole file (11 lines). `OnModelCreating` already calls `ApplyConfigurationsFromAssembly` (line 9), so new `IEntityTypeConfiguration<T>` classes are picked up automatically. Snake_case naming is applied in DI, not here.
3. `src/CustomerSupportCrm.Infrastructure/DependencyInjection.cs` (c3c7815) — lines 13–41. `AddInfrastructure` → private `AddPersistence`; the `AddOptions<T>().BindConfiguration(...).ValidateDataAnnotations().ValidateOnStart()` pattern is at lines 24–27; `AddDbContext` at 29–37.
4. `src/CustomerSupportCrm.Infrastructure/Persistence/DatabaseOptions.cs` (c3c7815) — template for an options class (`SectionName` const, `init` properties, DataAnnotations); `EnableSensitiveDataLogging` already exists.
5. `src/CustomerSupportCrm.Api/Program.cs` (c3c7815) — lines 11–30. Host setup; the initializer hook goes directly after `var app = builder.Build();` (line 22), before the middleware (line 26).
6. `Directory.Packages.props` — `ItemGroup Label="Persistence"`. Package versions are central; `.csproj` files use `PackageReference` **without** `Version`.
7. `Directory.Build.props` — `TreatWarningsAsErrors`, `Nullable`, `AnalysisLevel latest-recommended`: every new file must compile warning-free.
8. `tests/CustomerSupportCrm.Api.Tests/ApiFactory.cs` (**read only**) — the API test host runs in Development against an **unreachable** database, so it must force `Database:InitializeOnStartup=false` (0f87e2d line 25).
9. `tests/CustomerSupportCrm.IntegrationTests/PostgresApiFactory.cs` (**read only**) — 0f87e2d lines 69–74: environment `Test`, `InitializeOnStartup=true`, `Bootstrap:AdminEmail` / `AdminPassword` set per run.
10. `dotnet-tools.json` — `dotnet-ef` is the local tool used to create the migration.

---

## Backend Tasks

### 1 — Packages

File: `Directory.Packages.props` — in `Label="Persistence"` add `Microsoft.EntityFrameworkCore` and `Microsoft.EntityFrameworkCore.Relational`, both `10.0.12` (lines 22–23).

File: `src/CustomerSupportCrm.Application/CustomerSupportCrm.Application.csproj` — `<PackageReference Include="Microsoft.EntityFrameworkCore" />` (line 12; `IApplicationDbContext` exposes `DbSet<T>`).

File: `src/CustomerSupportCrm.Infrastructure/CustomerSupportCrm.Infrastructure.csproj` — `<PackageReference Include="Microsoft.EntityFrameworkCore.Relational" />` (line 10). Line 9 (`JwtBearer`) is Story 02. **Do not** add `Microsoft.Extensions.Identity.Core`; `FrameworkReference Microsoft.AspNetCore.App` (line 4, Phase 1) provides `PasswordHasher<T>`.

`src/CustomerSupportCrm.Domain/CustomerSupportCrm.Domain.csproj` stays empty — Domain has no package or framework references.

### 2 — Common and shared domain types

Create file: `src/CustomerSupportCrm.Domain/Common/Entity.cs` (lines 1–12) — `public abstract class Entity<TId> where TId : notnull`; `protected Entity(TId id)`; protected parameterless ctor for EF (`Id = default!`); `public TId Id { get; private init; }`.

Create file: `src/CustomerSupportCrm.Domain/Common/IAuditableEntity.cs` (lines 1–15) — `DateTimeOffset CreatedAt`, `Guid? CreatedBy`, `DateTimeOffset? UpdatedAt`, `Guid? UpdatedBy` (getters only). Doc comment: set by persistence, never by domain code.

Create file: `src/CustomerSupportCrm.Domain/Shared/EmailAddress.cs` (lines 1–48; delete `Domain/Shared/.gitkeep`)

- `public sealed record EmailAddress`; `MaxLength = 254`; `InvalidCode = "INVALID_EMAIL_ADDRESS"`.
- `Create(string? value)` (lines 19–29): normalize, then reject empty, over-long or malformed with `DomainException(InvalidCode, ...)`.
- `Normalize(string? value)` (lines 32–35): `Trim().ToLowerInvariant()`, `CA1308` suppressed locally; used for lookups without validation.
- `IsWellFormed` (lines 39–47): exactly one `@`, not first/last, a `.` after it, no whitespace. Deliverability is not checked.

### 3 — Permission catalog and roles

Create file: `src/CustomerSupportCrm.Domain/Roles/Permissions.cs` (lines 1–37)

- 13 flat `const string`s (lines 9–24): `TicketsView/Create/Update/Assign/Delete`, `CustomersView/Create/Update`, `ReportsView`, `UsersManage`, `RolesManage`, `SettingsManage`, `AuditView` → `tickets.view` … `audit.view`.
- `All` (lines 26–32), private `HashSet<string> Known` (line 34), `IsKnown(string)` (line 36). Doc comment: add codes, never rename existing ones.

Create file: `src/CustomerSupportCrm.Domain/Roles/RoleId.cs` — `public readonly record struct RoleId(Guid Value)`; `New()` → `Guid.CreateVersion7()`; `ToString()` → `Value.ToString()`.

Create file: `src/CustomerSupportCrm.Domain/Roles/Role.cs` (lines 1–163)

- `public sealed class Role : Entity<RoleId>, IAuditableEntity` (line 9). Constants: `NameMaxLength = 100`, `DescriptionMaxLength = 500`, `AdministratorName = "Administrator"`, codes `ROLE_IS_SYSTEM`, `UNKNOWN_PERMISSION`, `INVALID_ROLE_NAME` (lines 11–18).
- Properties (lines 34–52): `Name`, `NormalizedName`, `Description?`, `IsSystem`, `Permissions` (backed by `List<RolePermission> _permissions`), audit stamps, computed `PermissionCodes` (ordinal-sorted).
- `Create(name, description, permissions)` (lines 54–59) — non-system role. `CreateAdministrator()` (lines 62–67) — system role holding every permission.
- `Normalize(name)` (lines 69–73) → `Trim().ToUpperInvariant()`.
- `Update(...)` (lines 75–81) and `EnsureCanBeDeleted()` (line 83) call `EnsureNotSystem()` → `ROLE_IS_SYSTEM`.
- `GrantAllPermissions()` (lines 86–94) — `InvalidOperationException` for non-system roles; replaces with `Permissions.All`.
- `ReplacePermissions` (lines 110–125) — distinct, first unknown code → `DomainException(UNKNOWN_PERMISSION)`, removes dropped rows and adds new ones (existing rows are kept).
- Name/description validation (lines 98–108, 135–144) → `INVALID_ROLE_NAME`; empty description becomes `null`.
- `RolePermission` in the same file (lines 147–163): `RoleId RoleId`, `string Permission`; internal ctor, private EF ctor.

### 4 — Users

Create file: `src/CustomerSupportCrm.Domain/Users/UserId.cs` — same shape as `RoleId`. Delete `Domain/Users/.gitkeep`.

Create file: `src/CustomerSupportCrm.Domain/Users/User.cs` (lines 1–153)

- `enum UserStatus { Active, Disabled }` (lines 7–11).
- `public sealed class User : Entity<UserId>, IAuditableEntity` (line 16). Constants: `DisplayNameMaxLength = 200`, `MaxFailedLoginAttempts = 5`, `InvalidDisplayNameCode = "INVALID_DISPLAY_NAME"`, `LockoutDuration = 15 min` (lines 18–23).
- Properties (lines 41–67): `Email` (normalized), `DisplayName`, `PasswordHash`, `Status`, `FailedLoginAttempts`, `LockoutEndsAt`, `LastLoginAt`, `Roles`, audit stamps, computed `IsActive` and `RoleIds`.
- `Create(EmailAddress, displayName, passwordHash, IEnumerable<RoleId>)` (lines 69–76) — starts `Active`.
- `Rename` (lines 78–87) → `INVALID_DISPLAY_NAME`; `ChangePasswordHash` (lines 89–93) → `ArgumentException` on blank (programming error).
- `SetRoles` (lines 95–104) — distinct replace; `HasRole` (line 106).
- `IsLockedOut(now)` (line 108); `RecordFailedLogin(now)` (lines 111–119) locks at the 5th failure and resets the counter; `RecordSuccessfulLogin(now)` (lines 121–126).
- `Disable()` (line 128); `Enable()` (lines 130–135) also clears failures and lockout.
- `UserRole` in the same file (lines 138–153): `UserId`, `RoleId`; internal ctor, private EF ctor.
- No `ToString()` override: `PasswordHash` is never formatted.

Create file: `src/CustomerSupportCrm.Domain/Users/RefreshToken.cs` (lines 1–108)

- `enum RefreshTokenRevocationReason { Logout, ReuseDetected, UserDisabled, PasswordChanged }` (lines 5–11).
- `public sealed class RefreshToken : Entity<Guid>` (line 18); `UserAgentMaxLength = 512`, `IpAddressMaxLength = 64`, `InactiveCode = "REFRESH_TOKEN_INACTIVE"` (lines 20–23).
- Properties (lines 42–62): `UserId`, `SessionId`, `TokenHash`, `CreatedAt`, `ExpiresAt`, `UsedAt`, `ReplacedByTokenId`, `RevokedAt`, `RevokedReason`, `CreatedByIp`, `UserAgent`.
- `Issue(userId, tokenHash, now, lifetime, ip, userAgent)` (lines 65–66) starts a new session; `IsActive(now)` (line 68); `IsSpent` (line 71); `Rotate(...)` (lines 74–85) throws `REFRESH_TOKEN_INACTIVE` when not active; `Revoke(now, reason)` (lines 87–96) is a no-op when already revoked.
- Private `Create` (lines 98–104) validates hash and positive lifetime; `Truncate` (lines 106–107) caps IP/user agent and turns empty into `null`. The domain only ever sees the hash.

### 5 — Persistence abstraction and model

Create file: `src/CustomerSupportCrm.Application/Abstractions/Persistence/IApplicationDbContext.cs` (delete `Abstractions/Persistence/.gitkeep`) — `DbSet<User> Users`, `DbSet<Role> Roles`, `DbSet<RefreshToken> RefreshTokens`, `SaveChangesAsync` (lines 13–17, 21). `AuditLogs` (line 19) and its using (line 1) are Story 06.

File: `src/CustomerSupportCrm.Infrastructure/Persistence/ApplicationDbContext.cs`

- Implement `IApplicationDbContext` (line 10); DbSets `Users`, `Roles`, `RefreshTokens` (lines 12–16). `AuditLogs` (line 18) is Story 06.
- `ConfigureConventions` (lines 25–29): `Properties<UserId>()` / `Properties<RoleId>()` `.HaveConversion<StronglyTypedIdConverters.…>()`.

Create file: `src/CustomerSupportCrm.Infrastructure/Persistence/Configurations/StronglyTypedIdConverters.cs` (lines 1–13) — nested `UserIdConverter` / `RoleIdConverter` : `ValueConverter<T, Guid>`.

Create file: `src/CustomerSupportCrm.Infrastructure/Persistence/Configurations/PostgresConventions.cs` (lines 1–15) — `HasXminConcurrencyToken<T>()`: shadow `uint Version` → column `xmin`, type `xid`, `IsRowVersion()`.

Create file: `src/CustomerSupportCrm.Infrastructure/Persistence/Configurations/UserConfiguration.cs`

- `UserConfiguration` (lines 9–35): table `users`; `Id` `ValueGeneratedNever()`; `Email` max `EmailAddress.MaxLength` + unique index (lines 17–18); `DisplayName` 200; `PasswordHash` 512; `Status` stored as string (20); `HasMany(Roles)` cascade with field access (lines 24–28); ignore `IsActive`, `RoleIds`; `HasXminConcurrencyToken()` (line 33).
- `UserRoleConfiguration` (lines 37–51): table `user_roles`; key `(UserId, RoleId)`; index on `RoleId`; FK to `Role` with `DeleteBehavior.Restrict`.

Create file: `src/CustomerSupportCrm.Infrastructure/Persistence/Configurations/RoleConfiguration.cs`

- `RoleConfiguration` (lines 7–30): table `roles`; unique index on `NormalizedName` (line 17); `Permissions` cascade with field access (lines 20–24); ignore `PermissionCodes`; `HasXminConcurrencyToken()` (line 28).
- `RolePermissionConfiguration` (lines 32–40): table `role_permissions`; key `(RoleId, Permission)`; `Permission` max 100.

Create file: `src/CustomerSupportCrm.Infrastructure/Persistence/Configurations/RefreshTokenConfiguration.cs` (lines 7–34): table `refresh_tokens`; `TokenHash` 128 + unique index; indexes on `SessionId` and `UserId`; `RevokedReason` as string (32); FK to `User` cascade, no navigation; ignore `IsSpent`; `HasXminConcurrencyToken()` (line 32).

Create file: `src/CustomerSupportCrm.Infrastructure/Persistence/Interceptors/AuditableEntityInterceptor.cs` (lines 1–50) — `SaveChangesInterceptor` taking `TimeProvider` and `ICurrentUser` (Story 03); on `Added` sets `CreatedAt/By`, on `Modified` **or when a child collection changed** sets `UpdatedAt/By` (lines 35–48). Actor is `null` when unauthenticated (line 33). It writes no audit rows (see Story 06).

### 6 — Password hasher

Create file: `src/CustomerSupportCrm.Application/Abstractions/Authentication/IPasswordHasher.cs` (lines 1–17; delete `Abstractions/Authentication/.gitkeep`)

```csharp
public enum PasswordVerificationResult { Failed, Success, SuccessRehashNeeded }

public interface IPasswordHasher
{
    string Hash(string password);
    PasswordVerificationResult Verify(string passwordHash, string providedPassword);
}
```

Create file: `src/CustomerSupportCrm.Infrastructure/Authentication/IdentityPasswordHasher.cs` (lines 1–27) — `internal sealed`, wraps `PasswordHasher<object>` with a static unused subject; `using` aliases `IdentityResult` / `PasswordVerificationResult` (lines 3–4) resolve the name clash; `Verify` maps `Success` / `SuccessRehashNeeded`, everything else → `Failed`. The `Infrastructure/Authentication/.gitkeep` placeholder was left in place.

### 7 — Seeding

Create file: `src/CustomerSupportCrm.Infrastructure/Persistence/Seed/BootstrapOptions.cs` (lines 1–16) — `SectionName = "Bootstrap"`; `AdminEmail?`, `AdminDisplayName = "Administrator"`, `AdminPassword?`. No validation (optional).

Create file: `src/CustomerSupportCrm.Infrastructure/Persistence/Seed/DatabaseInitializer.cs` (lines 1–115) — `internal sealed partial class DatabaseInitializer(ApplicationDbContext db, IPasswordHasher passwordHasher, IOptions<BootstrapOptions> bootstrap, ILogger<DatabaseInitializer> logger)`:

1. `InitializeAsync` (lines 37–43): `db.Database.MigrateAsync`, then roles, then administrator.
2. `SeedRolesAsync` (lines 45–74): `firstRun = !Roles.Any()`; load the system Administrator by `NormalizedName` with permissions; create via `CreateAdministrator()` or re-sync via `GrantAllPermissions()` every run. On first run only, add `DefaultRoles` (lines 22–35): **Manager** (all tickets, all customers, `reports.view`) and **Agent** (tickets view/create/update, customers view/create/update). Save.
3. `SeedAdministratorAsync` (lines 76–99): return if any user exists; if `AdminEmail` or `AdminPassword` is blank → warning `LogBootstrapSkipped` and return; else `User.Create(EmailAddress.Create(...), AdminDisplayName, passwordHasher.Hash(...), [administrator.Id])`, save, log `"Created initial administrator {Email}"`. Password and hash are never logged (source-generated `[LoggerMessage]`, lines 101–105).
4. `DatabaseInitializerExtensions.InitializeDatabaseAsync(IServiceProvider, ct)` (lines 108–115) creates an async scope and runs the initializer.

No minimum password length is enforced here (the intake did not require one; Story 04 validates passwords for new users).

### 8 — DI registration

File: `src/CustomerSupportCrm.Infrastructure/DependencyInjection.cs`

- `AddInfrastructure`: `services.AddSingleton(TimeProvider.System);` (line 28) and the `services.AddIdentityServices();` call (line 32). `AddHttpContextAccessor` (line 29) serves Stories 02/03.
- `AddPersistence`: `AddScoped<AuditableEntityInterceptor>()` (line 44) and `.AddInterceptors(...)` on the context (line 54); `AddScoped<IApplicationDbContext>(sp => sp.GetRequiredService<ApplicationDbContext>())` (line 56); `AddOptions<BootstrapOptions>().BindConfiguration(...)` (line 58, no `ValidateOnStart`); `AddScoped<DatabaseInitializer>()` (line 59).
- `AddIdentityServices`: `services.AddSingleton<IPasswordHasher, IdentityPasswordHasher>();` (line 98). The method name avoids ASP.NET Core Identity's `AddIdentityCore`. Its JWT, policy, current-user, request-context and audit lines belong to Stories 02, 03 and 06.
- Usings for this story: lines 2, 4, 6, 9–10 (line 8 is Phase 1).

### 9 — Host wiring

File: `src/CustomerSupportCrm.Infrastructure/Persistence/DatabaseOptions.cs` — add `bool InitializeOnStartup { get; init; }` with its doc comment (lines 15–19).

File: `src/CustomerSupportCrm.Api/Program.cs` — usings (lines 10–12); after `var app = builder.Build();` (line 27):

```csharp
// Deployment step: apply migrations and seed data, then exit.
if (args.Contains("--init-database"))
{
    await app.Services.InitializeDatabaseAsync();
    return;
}

if (app.Services.GetRequiredService<IOptions<DatabaseOptions>>().Value.InitializeOnStartup)
{
    await app.Services.InitializeDatabaseAsync();
}
```

(lines 29–39). CORS, rate limiting, authentication and `RunAsync` lines are Stories 02, 03 and 07.

### 10 — Configuration

File: `src/CustomerSupportCrm.Api/appsettings.json` — `"InitializeOnStartup": false` (line 18) and the `Bootstrap` section with empty `AdminEmail` / `AdminPassword` and `AdminDisplayName "Administrator"` (lines 36–40).

File: `src/CustomerSupportCrm.Api/appsettings.Development.json` — `"InitializeOnStartup": true` (line 11) and a dev-only `Bootstrap` admin (lines 22–25). Other environments supply `Bootstrap__AdminEmail` / `Bootstrap__AdminPassword` once for the first `--init-database`.

### 11 — Migration

```bash
dotnet tool restore
dotnet ef migrations add InitialIdentity --project src/CustomerSupportCrm.Infrastructure --startup-project src/CustomerSupportCrm.Api --output-dir Persistence/Migrations
```

Delivered files (delete `Persistence/Migrations/.gitkeep`):

- `src/CustomerSupportCrm.Infrastructure/Persistence/Migrations/20260930092037_InitialIdentity.cs` (225 lines)
- `.../20260930092037_InitialIdentity.Designer.cs` (395 lines) and `.../ApplicationDbContextModelSnapshot.cs` (392 lines), generated.

Story 01 content of `InitialIdentity.cs`: `roles` (lines 35–53), `users` (55–76), `role_permissions` (78–94), `refresh_tokens` (96–123), `user_roles` (125–147); indexes `ix_refresh_tokens_session_id`, `ix_refresh_tokens_token_hash` (unique), `ix_refresh_tokens_user_id`, `ix_roles_normalized_name` (unique), `ix_user_roles_role_id`, `ix_users_email` (unique) (lines 169–200); `Down` drops at 209–222. Every `DateTimeOffset` is `timestamp with time zone`; `users`, `roles`, `refresh_tokens` carry `xmin xid` row versions. **The same migration also creates `audit_logs` from Story 06** (lines 14–33, indexes 149–167, drop 206–207).

File: `.editorconfig` — lines 398–406: `CA1716` off (the `Domain.Shared` namespace), `CA1711` off (`RolePermission`), and `[**/Migrations/*.cs]` as `generated_code = true` with `dotnet_analyzer_diagnostic.severity = none`. The `[tests/**/*.cs]` block (lines 408–410) is test work, not this story.

### 12 — Docs

- `docs/development.md` — Development migrates and seeds at startup from `Bootstrap:*` (line 16; its sign-in sentences are Story 02); "Outside Development" `dotnet CustomerSupportCrm.Api.dll --init-database` (lines 51–55).
- `docs/architecture.md` — abstractions rows `IApplicationDbContext`, `IPasswordHasher`, `TimeProvider` (lines 73, 76, 79); Persistence bullets for strongly typed ids, `xmin`, `AuditableEntityInterceptor`, migrations, `DatabaseInitializer` (lines 85–89); configuration rows `DatabaseOptions.InitializeOnStartup` and `Bootstrap` (lines 98, 102); `TimeProvider` decision reworded (line 111).
- `docs/security.md` — `Bootstrap:AdminEmail` / `AdminPassword` secrets row (line 81) and the `--init-database` release step (line 86). The "Passwords" section is owned by Story 02.

---

## Edge Cases & Failure Modes

- **Email casing / whitespace** (`" Admin@X.com "`) — `EmailAddress.Create` trims and lower-cases; uniqueness is the unique index on `users.email` (`UserConfiguration.cs` line 18). Duplicates surface as `DbUpdateException`; Story 04 checks first and returns `EMAIL_TAKEN`.
- **Malformed email** — `DomainException(INVALID_EMAIL_ADDRESS)` → 422 through the Phase 1 `GlobalExceptionHandler`.
- **Unknown permission code** — `Role.ReplacePermissions` throws `UNKNOWN_PERMISSION` (`Role.cs` line 117).
- **Changing or deleting a system role** — `ROLE_IS_SYSTEM` (`Role.cs` line 131). Manager and Agent are ordinary roles and can be edited or deleted.
- **Lockout arithmetic** — 5th consecutive failure sets `LockoutEndsAt = now + 15 min` and resets the counter; `IsLockedOut(now)` is false once `now >= LockoutEndsAt` (`User.cs` lines 108–119).
- **Catalog grows** — the next initializer run calls `GrantAllPermissions()`, so the Administrator gets new codes; Manager/Agent are never touched after the first run.
- **Initializer run twice** — Administrator found by `NormalizedName`, defaults skipped (`firstRun` false), users exist → nothing new.
- **Empty `Bootstrap` config** — warning logged, no user created, startup continues.
- **Invalid `Bootstrap:AdminEmail`** — `EmailAddress.Create` throws `DomainException`; `--init-database` exits non-zero.
- **Initializer before database exists** — `MigrateAsync` runs first, so there is no "relation does not exist" case; an unreachable server fails startup when `InitializeOnStartup` is true.
- **API test host** — Development would migrate on startup; `ApiFactory` forces `InitializeOnStartup=false` so the unreachable DB is never touched.
- **Concurrent edits** — `xmin` makes the losing `SaveChanges` throw `DbUpdateConcurrencyException` (mapped to 409 `CONFLICT` by later stories).
- **Long user agent / IP** — truncated in `RefreshToken` (lines 38–39, 106–107) instead of failing the insert.
- **Child-only change** (only role permissions changed) — `UpdatedAt/By` still stamped (`AuditableEntityInterceptor.cs` line 42).

---

## Test Plan

Tests are out of scope under the standing directive: **no tests are added, changed or removed**. The existing tests in `0f87e2d` that cover this story are read-only references:

1. `tests/CustomerSupportCrm.Domain.Tests/Users/UserTests.cs` (unit): `CreateNormalizesEmailAndStartsActive`, `CreateRejectsBlankDisplayName`, `LocksOutAfterMaxFailedAttempts`, `SuccessfulLoginResetsFailuresAndRecordsTime`, `EnableClearsLockout`, `SetRolesReplacesMembershipWithoutDuplicates`.
2. `tests/CustomerSupportCrm.Domain.Tests/Users/RefreshTokenTests.cs` (unit): `IssuedTokenIsActiveUntilExpiry`, `RotateSpendsTokenAndKeepsSession`, `SpentTokenCannotRotateAgain`, `RevokeKeepsFirstReason`, `TruncatesLongUserAgent`.
3. `tests/CustomerSupportCrm.Domain.Tests/Roles/RoleTests.cs` (unit): `CreateTrimsNameAndNormalizes`, `RejectsUnknownPermission`, `UpdateReplacesPermissions`, `AdministratorHoldsEveryPermission`, `SystemRoleCannotBeUpdatedOrDeleted`, `PermissionCatalogCodesAreUniqueAndGrouped`.
4. `tests/CustomerSupportCrm.Domain.Tests/Shared/EmailAddressTests.cs` (unit): `NormalizesToTrimmedLowerCase`, `EqualByValue`, `RejectsMalformedAddresses`.
5. `tests/CustomerSupportCrm.IntegrationTests/DatabaseTests.cs` (integration, PostgreSQL): `ReadinessIsHealthyWhenDatabaseIsReachable` (line 12), `MigrationsAreFullyApplied` (line 23), `SeedsSystemAndDefaultRoles` (line 32). The commit message says the integration suite had not yet been run.

---

## Migration / Rollback

- **Apply:** `dotnet ef database update --project src/CustomerSupportCrm.Infrastructure --startup-project src/CustomerSupportCrm.Api`, or `dotnet CustomerSupportCrm.Api.dll --init-database` (also seeds). In Development the API migrates on startup.
- **Rollback:** `dotnet ef database update 0 --project src/CustomerSupportCrm.Infrastructure --startup-project src/CustomerSupportCrm.Api`. This also drops `audit_logs` (shared migration); there is no identity-only rollback.
- **Half-applied state:** EF migrations run in a transaction on PostgreSQL; a failure leaves no partial tables. If `__EFMigrationsHistory` lists `20260930092037_InitialIdentity` but tables are missing, delete that row and re-apply.

---

## Verification Steps

1. **Backend builds:** from `customer-support-crm-api/` run `dotnet build` — zero warnings, zero errors.
2. **Regression:** `dotnet test --filter "FullyQualifiedName!~IntegrationTests"` — all tests pass (proves the API host does not touch the DB at startup).
3. **Migration:** `dotnet ef database update --project src/CustomerSupportCrm.Infrastructure --startup-project src/CustomerSupportCrm.Api` — succeeds; `\dt` in psql shows `users`, `user_roles`, `roles`, `role_permissions`, `refresh_tokens` (and `audit_logs`).
4. **Initialize (empty config):** with `Bootstrap:AdminEmail` / `AdminPassword` cleared (e.g. empty user-secrets or env vars), run `dotnet run --project src/CustomerSupportCrm.Api -- --init-database` — exits 0, logs the "no administrator was created" warning; `roles` has 3 rows, `users` has 0.
5. **Initialize (configured):** set both `Bootstrap` values and run again — one user with the Administrator role; `role_permissions` for Administrator has 13 rows; Manager 9, Agent 6.
6. **Idempotency:** run `--init-database` a third time — row counts unchanged.
7. **No secrets in logs:** search the output of steps 4–6 for the password value and for `AQAAAA` (Identity hash prefix) — no matches.
8. **Normal startup:** `dotnet run --project src/CustomerSupportCrm.Api --launch-profile https` — Development migrates/seeds, starts, and `/health/ready` returns `Healthy`.

---

## Done Criteria

- [x] Domain entities exist with no framework references (`CustomerSupportCrm.Domain.csproj` has no references). Deviation: `User`/`UserRole`/`RefreshToken` in `Domain/Users`, `Role`/`RolePermission`/`Permissions` in `Domain/Roles`, `EmailAddress` in `Domain/Shared`.
- [x] Permission catalog is code-defined (13 codes, `Permissions.All`, `IsKnown`); the Administrator system role is seeded with `Permissions.All` and re-synced every run.
- [x] EF configurations, DbSets, strongly typed id converters, `xmin` tokens and unique indexes (`users.email`, `roles.normalized_name`, `refresh_tokens.token_hash`) exist, plus indexes on `refresh_tokens(user_id)` and `(session_id)`.
- [ ] Migration applies cleanly — delivered as `20260930092037_InitialIdentity` (not `AddIdentity`, and it also contains Story 06's `audit_logs`). `DatabaseTests.MigrationsAreFullyApplied` covers it, but the commit message says the integration suite had not been run, so a clean `dotnet ef database update` is not yet confirmed.
- [x] Initializer is idempotent and skips admin creation when `Bootstrap` is empty (`DatabaseInitializer`, not `IdentitySeeder`; runs via `--init-database` or `InitializeOnStartup`).
- [x] Password hash, token hash and security stamp are never logged (only the admin email is logged; there is no security stamp).
- [x] `IPasswordHasher` + `IdentityPasswordHasher` and `TimeProvider.System` registered.
- [x] Nothing changed in `docker-compose.yml`, `deploy/` or `.github/` by this story. (`0f87e2d` does change files under `tests/`; that is test work outside this story.)
- [x] `dotnet build` passes with zero warnings.

**STOP HERE. Report to the user and wait for confirmation before proceeding to Story 02.**
