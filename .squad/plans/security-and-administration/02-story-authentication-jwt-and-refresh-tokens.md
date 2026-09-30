# Story 02 — Authentication: JWT access tokens and rotating refresh tokens (Story: P2-02)

> As-built plan: written after implementation in `customer-support-crm-api` commit `0f87e2d`; paths and line numbers refer to that commit.

## Prerequisites

- Story 01 completed: [01-story-identity-domain-persistence.md](01-story-identity-domain-persistence.md) — `User`, `RefreshToken`, `EmailAddress`, `IApplicationDbContext`, `IPasswordHasher` / `IdentityPasswordHasher`, the `InitialIdentity` migration and `DatabaseInitializer` (bootstrap admin) exist.
- Phase 1 platform (`c3c7815`): `ApiResponse` envelope, `GlobalExceptionHandler`, `ErrorResponseWriter.Localize`, `IEndpoint` discovery, `Messages.resx` / `Messages.ar.resx`.
- Follow-up stories build on this one: [03-story-current-user-and-permission-authorization.md](03-story-current-user-and-permission-authorization.md) (`ICurrentUser`, policies, `/auth/me`, OpenAPI bearer scheme), [04-story-user-management.md](04-story-user-management.md) (change password), [06-story-audit-logging.md](06-story-audit-logging.md) (audit calls in these handlers), [07-story-security-hardening.md](07-story-security-hardening.md) (rate limiting, CORS).
- All paths are relative to the **`customer-support-crm-api/`** repo root.

---

## Story Goal

Sign-in for staff with short-lived JWT access tokens and rotated, revocable, hashed refresh tokens:

1. `POST /api/v1/auth/login` verifies credentials, enforces lockout, starts a session and returns an access token plus the user profile; the refresh token is set as an **HttpOnly cookie**.
2. `POST /api/v1/auth/refresh` rotates the refresh token (same session); presenting a spent token revokes the whole session.
3. `POST /api/v1/auth/logout` revokes the session and clears the cookie (idempotent).
4. JwtBearer validation (issuer, audience, lifetime, signing key, 30 s skew, no inbound claim mapping) and `UseAuthentication` / `UseAuthorization` in the pipeline.

**Deviations from the intake (follow the code):**

- Refresh token travels in the `crm_refresh` cookie (`HttpOnly; Secure; SameSite=Strict; Path=/api/v1/auth`), **not** in the JSON body. Refresh and logout therefore require the `X-CSRF-Protection` header; logout is `AllowAnonymous` (the access token may have expired).
- "Family" is `RefreshToken.SessionId` (Story 01). Reuse returns `INVALID_REFRESH_TOKEN`; there is no `REFRESH_TOKEN_REUSED` code.
- Options are `JwtOptions.AccessTokenLifetimeMinutes` (15) / `RefreshTokenLifetimeDays` (**14**), `SigningKey` `[MinLength(32)]` characters. No `LockoutOptions`: lockout constants live on `User` (Story 01).
- Claims: `sub`, `email`, `name`, `sid`, `jti`, `role`, `permission` (not `perm`); no `culture` claim.
- Response is `AccessTokenResponse(AccessToken, ExpiresAt, User)` with `CurrentUserResponse(Id, Email, DisplayName, Roles, Permissions)`.
- Error constants are `AuthenticationErrors` (not `AuthErrorCodes`). A disabled user with the correct password gets `ACCOUNT_DISABLED`.
- JwtBearer is wired in `Infrastructure/DependencyInjection.cs`, not `Api/Authentication/`. CSRF/cookie rationale is documented in `docs/security.md` and ADR 0002.

**Not in scope:** audit calls (Story 06), `.RequireRateLimiting(...)` and CORS (Story 07), `ICurrentUser`, permission policies, `RequireAuthorization()` on the `/api/v1` group, `/auth/me`, OpenAPI bearer scheme (Story 03), change password (Story 04). **Do not touch** Docker, `deploy/`, `.github/` or `tests/`.

---

## Context — Read These Files First

1. `src/CustomerSupportCrm.Domain/Users/RefreshToken.cs` (0f87e2d) — lines 5–11 `RefreshTokenRevocationReason`; line 65 `Issue(userId, tokenHash, now, lifetime, ip, userAgent)` (new `SessionId`); line 68 `IsActive(now)`; line 71 `IsSpent`; lines 74–85 `Rotate(...)` (sets `UsedAt`, `ReplacedByTokenId`, returns successor in the same session, throws `REFRESH_TOKEN_INACTIVE`); lines 87–96 `Revoke(now, reason)` (keeps the first reason).
2. `src/CustomerSupportCrm.Domain/Users/User.cs` (0f87e2d) — lines 19 and 23 `MaxFailedLoginAttempts = 5`, `LockoutDuration = 15 min`; line 65 `IsActive`; line 89 `ChangePasswordHash`; line 108 `IsLockedOut(now)`; lines 111–119 `RecordFailedLogin(now)`; lines 121–126 `RecordSuccessfulLogin(now)`.
3. `src/CustomerSupportCrm.Domain/Shared/EmailAddress.cs` (0f87e2d) — line 11 `MaxLength = 254`; line 32 `Normalize(string?)` — `User.Email` is stored normalized, so login looks up by `EmailAddress.Normalize(request.Email)`.
4. `src/CustomerSupportCrm.Infrastructure/Persistence/Configurations/RefreshTokenConfiguration.cs` (0f87e2d) — line 16 unique `TokenHash`; line 17 index on `SessionId`; line 32 `HasXminConcurrencyToken()` — concurrent rotations of one token are serialized by this.
5. `src/CustomerSupportCrm.Application/Abstractions/Authentication/IPasswordHasher.cs` (0f87e2d, Story 01) — lines 3–10 `PasswordVerificationResult { Failed, Success, SuccessRehashNeeded }`; lines 12–17 `Hash` / `Verify(passwordHash, providedPassword)`.
6. `src/CustomerSupportCrm.Application/Abstractions/Persistence/IApplicationDbContext.cs` (0f87e2d) — lines 13–21: `Users`, `Roles`, `RefreshTokens`, `SaveChangesAsync`. Handlers use EF Core directly.
7. `src/CustomerSupportCrm.Application/Abstractions/Http/IEndpoint.cs` (c3c7815) — lines 9–12; endpoints are discovered by `AddEndpoints` in `Application/DependencyInjection.cs` (0f87e2d lines 31–39) and mapped under `/api/v1`.
8. `src/CustomerSupportCrm.Application/Common/Exceptions/AppException.cs` and `ForbiddenException.cs` (c3c7815) — pattern for the new `UnauthorizedException`.
9. `src/CustomerSupportCrm.Api/Middleware/GlobalExceptionHandler.cs` (0f87e2d) — lines 30–31 map `AppException` via `StatusFor` and localize with `ErrorResponseWriter.Localize` by code; lines 59–66 `StatusFor`. Only the category code is logged (line 49), never the message or request body.
10. `src/CustomerSupportCrm.Api/Program.cs` (c3c7815) — lines 24–40: middleware order; authentication goes after `UseHttpsRedirection` (line 37) and before `MapApiHealthChecks` (line 39).
11. `src/CustomerSupportCrm.Api/Configuration/LoggingExtensions.cs` (0f87e2d) — lines 42–66 `UseApiRequestLogging`: logs method, path, status, route and `UserId` only — no bodies, headers or cookies.
12. `src/CustomerSupportCrm.Infrastructure/DependencyInjection.cs` (0f87e2d) — lines 24–35 `AddInfrastructure` → `AddIdentityServices()` at line 32; lines 65–103 the as-built `AddIdentityServices`.

---

## Backend Tasks

### 1 — Package

File: `Directory.Packages.props` — in the first `ItemGroup` (next to `Swashbuckle.AspNetCore.SwaggerUI`) add `<PackageVersion Include="Microsoft.AspNetCore.Authentication.JwtBearer" Version="10.0.12" />`.

File: `src/CustomerSupportCrm.Infrastructure/CustomerSupportCrm.Infrastructure.csproj` — add `<PackageReference Include="Microsoft.AspNetCore.Authentication.JwtBearer" />` (line 9, alphabetical). The project already has `FrameworkReference Microsoft.AspNetCore.App` (line 4), which supplies `PasswordHasher<T>`; no `Microsoft.Extensions.Identity.Core` package was added.

### 2 — Abstractions

Create file: `src/CustomerSupportCrm.Application/Abstractions/Authentication/CrmClaimTypes.cs` — `public static class CrmClaimTypes` with `Subject = "sub"`, `Email = "email"`, `Name = "name"`, `Role = "role"`, `Permission = "permission"`, `SessionId = "sid"`.

Create file: `src/CustomerSupportCrm.Application/Abstractions/Authentication/ITokenService.cs`

```csharp
public sealed record AccessToken(string Token, DateTimeOffset ExpiresAt);

/// <param name="Token">The opaque value handed to the client. Never stored or logged.</param>
/// <param name="Hash">The value persisted server-side.</param>
public sealed record GeneratedRefreshToken(string Token, string Hash);

public interface ITokenService
{
    TimeSpan RefreshTokenLifetime { get; }
    AccessToken CreateAccessToken(User user, Guid sessionId, IReadOnlyCollection<string> roles, IReadOnlyCollection<string> permissions);
    GeneratedRefreshToken GenerateRefreshToken();
    string HashRefreshToken(string token);
}
```

Create file: `src/CustomerSupportCrm.Application/Abstractions/Http/IRequestContext.cs` — `string? CorrelationId`, `string? IpAddress`, `string? UserAgent` (used to stamp `RefreshToken.CreatedByIp` / `UserAgent`).

Create file: `src/CustomerSupportCrm.Application/Common/Exceptions/UnauthorizedException.cs`

```csharp
public sealed class UnauthorizedException(string code = ErrorCodes.Unauthorized, string message = "Authentication is required.")
    : AppException(code, message);
```

File: `src/CustomerSupportCrm.Application/Abstractions/Http/ApiResults.cs` — add `public static ApiResult<object?> Success(string? message = null)` returning 200 with `ApiResponse.Ok<object?>(null, message)` (lines 18–20; used by logout).

### 3 — Infrastructure

Create file: `src/CustomerSupportCrm.Infrastructure/Authentication/JwtOptions.cs` — `SectionName = "Jwt"`; `[Required] Issuer`, `[Required] Audience`, `[Required][MinLength(32)] SigningKey`, `[Range(1, 60)] AccessTokenLifetimeMinutes = 15`, `[Range(1, 90)] RefreshTokenLifetimeDays = 14` (all `init`).

Create file: `src/CustomerSupportCrm.Infrastructure/Authentication/TokenService.cs` — `internal sealed class TokenService(IOptions<JwtOptions> options, TimeProvider time) : ITokenService`:

- `private static readonly JsonWebTokenHandler Handler = new();` (`Microsoft.IdentityModel.JsonWebTokens`).
- `public static SymmetricSecurityKey CreateSigningKey(JwtOptions options)` — UTF-8 bytes of `SigningKey`; reused by the bearer setup (task 5).
- `CreateAccessToken` (lines 23–53): `now = time.GetUtcNow()`, `expiresAt = now + AccessTokenLifetimeMinutes`; claims `sub` = `user.Id.Value`, `email`, `name` = `user.DisplayName`, `sid`, `jti` = new Guid, one `role` per role, one `permission` per permission; `SecurityTokenDescriptor` with issuer, audience, `IssuedAt`/`NotBefore` = now, `Expires`, `HmacSha256` signing credentials.
- `GenerateRefreshToken` (lines 56–60): `Base64Url.EncodeToString(RandomNumberGenerator.GetBytes(32))`, paired with its hash.
- `HashRefreshToken` (lines 63–64): `Convert.ToHexString(SHA256.HashData(Encoding.UTF8.GetBytes(token)))` — the only form ever persisted.

Create file: `src/CustomerSupportCrm.Infrastructure/Authentication/HttpRequestContext.cs` — `internal sealed class HttpRequestContext(IHttpContextAccessor accessor) : IRequestContext`; `CorrelationId => accessor.HttpContext?.GetCorrelationId()`, `IpAddress` from `Connection.RemoteIpAddress`, `UserAgent` from the `User-Agent` header or `null`.

### 4 — Contracts

Create file: `src/CustomerSupportCrm.Contracts/Authentication/AuthenticationContracts.cs` (delete `Contracts/Authentication/.gitkeep`):

```csharp
public sealed record LoginRequest(string Email, string Password);

/// Returned by login and refresh. The refresh token is never in the body; it is set as an
/// HttpOnly cookie scoped to the auth endpoints.
public sealed record AccessTokenResponse(string AccessToken, DateTimeOffset ExpiresAt, CurrentUserResponse User);

public sealed record CurrentUserResponse(Guid Id, string Email, string DisplayName, IReadOnlyList<string> Roles, IReadOnlyList<string> Permissions);
```

(`ChangePasswordRequest` in the same file belongs to Story 04.)

### 5 — DI and JwtBearer

File: `src/CustomerSupportCrm.Infrastructure/DependencyInjection.cs`

- In `AddInfrastructure` add `services.AddHttpContextAccessor();` (line 29) and call `services.AddIdentityServices();` (line 32).
- `AddIdentityServices` (lines 65–103) — the parts owned by this story:

```csharp
services.AddOptions<JwtOptions>()
    .BindConfiguration(JwtOptions.SectionName)
    .ValidateDataAnnotations()
    .ValidateOnStart();

services.AddAuthentication(JwtBearerDefaults.AuthenticationScheme).AddJwtBearer();
services.AddOptions<JwtBearerOptions>(JwtBearerDefaults.AuthenticationScheme)
    .Configure<IOptions<JwtOptions>>((bearer, jwtOptions) =>
    {
        var jwt = jwtOptions.Value;
        bearer.MapInboundClaims = false; // keep "sub", "role", "permission"
        bearer.TokenValidationParameters = new TokenValidationParameters
        {
            ValidIssuer = jwt.Issuer, ValidAudience = jwt.Audience,
            IssuerSigningKey = TokenService.CreateSigningKey(jwt),
            ValidAlgorithms = [SecurityAlgorithms.HmacSha256],
            ValidateIssuer = true, ValidateAudience = true, ValidateIssuerSigningKey = true, ValidateLifetime = true,
            ClockSkew = TimeSpan.FromSeconds(30),
            NameClaimType = CrmClaimTypes.Name, RoleClaimType = CrmClaimTypes.Role,
        };
    });

services.AddSingleton<ITokenService, TokenService>();
services.AddScoped<IRequestContext, HttpRequestContext>();
```

Line 98 (`IPasswordHasher`) comes from Story 01; lines 96, 100 and 102 (`AddPermissionPolicies`, `ICurrentUser`, `IAuditTrail`) come from Stories 03 and 06.

File: `src/CustomerSupportCrm.Application/DependencyInjection.cs` — `services.AddScoped<UserSessionService>();` (line 26).

### 6 — Feature common (`Application/Features/Authentication/Common/`)

Delete `Features/Authentication/.gitkeep`.

- Create file: `AuthenticationErrors.cs` — `InvalidCredentials = "INVALID_CREDENTIALS"`, `AccountLocked = "ACCOUNT_LOCKED"`, `AccountDisabled = "ACCOUNT_DISABLED"`, `InvalidRefreshToken = "INVALID_REFRESH_TOKEN"`, `CsrfValidationFailed = "CSRF_VALIDATION_FAILED"` (`InvalidCurrentPassword` at line 10 is Story 04).
- Create file: `AuthenticatedSession.cs` — `public sealed record AuthenticatedSession(AccessTokenResponse Response, string RefreshToken, DateTimeOffset RefreshTokenExpiresAt);`
- Create file: `AuthenticationHttp.cs` — `RoutePrefix = "/auth"`, `RefreshCookieName = "crm_refresh"`, `CsrfHeaderName = "X-CSRF-Protection"`, `RefreshCookiePath = "/api/v1/auth"`; `AppendRefreshCookie(HttpResponse, AuthenticatedSession)`, `DeleteRefreshCookie(HttpResponse)`, `ReadRefreshCookie(HttpRequest)`; `RequireCsrfHeader(this RouteHandlerBuilder)` endpoint filter throwing `ForbiddenException(CsrfValidationFailed, ...)` when the header is absent (lines 44–53); cookie options (lines 55–63) `HttpOnly = true, Secure = true, SameSite = Strict, Path = RefreshCookiePath, Expires, IsEssential = true`.
- Create file: `UserAccessProfile.cs` — `record UserAccessProfile(IReadOnlyList<string> Roles, IReadOnlyList<string> Permissions)` with `static LoadAsync(IApplicationDbContext db, UserId userId, CancellationToken)` (lines 10–22): role names of the user's roles and the distinct union of their permission codes, both ordinal-sorted.
- Create file: `UserSessionService.cs` — `public sealed class UserSessionService(IApplicationDbContext db, ITokenService tokens, IRequestContext request, TimeProvider time)`; **never saves** (the handler does):

```csharp
public Task<AuthenticatedSession> StartAsync(User user, CancellationToken ct);                 // RefreshToken.Issue + db.RefreshTokens.Add
public Task<AuthenticatedSession> RotateAsync(User user, RefreshToken current, CancellationToken ct); // current.Rotate + Add successor
public Task RevokeAsync(Expression<Func<RefreshToken, bool>> predicate, RefreshTokenRevocationReason reason, CancellationToken ct);
    // loads tokens matching predicate with UsedAt == null && RevokedAt == null, calls Revoke(now, reason)
```

`BuildAsync` (lines 62–73) loads `UserAccessProfile`, calls `CreateAccessToken(user, stored.SessionId, roles, permissions)` and returns the session with `stored.ExpiresAt`.

### 7 — Login slice (`Features/Authentication/Login/`)

- Create file: `LoginCommand.cs` — `public sealed record LoginCommand(string Email, string Password) : IRequest<AuthenticatedSession>;`
- Create file: `LoginValidator.cs` — `Email` `NotEmpty().MaximumLength(EmailAddress.MaxLength)`; `Password` `NotEmpty().MaximumLength(CommonRules.PasswordMaxLength)` (128; `CommonRules` is in `Application/Common/Validation/CommonRules.cs`, created in this commit and shared with Story 04).
- Create file: `LoginEndpoint.cs` — `MapPost($"{AuthenticationHttp.RoutePrefix}/login", (LoginRequest, ISender, HttpContext, CancellationToken) => ...)`: send command, `AppendRefreshCookie`, `ApiResults.Ok(session.Response)`; `.AllowAnonymous().WithName("Login").WithTags("Authentication").Produces<ApiResponse<AccessTokenResponse>>()`. (Line 22 `.RequireRateLimiting(...)` is Story 07.)
- Create file: `LoginHandler.cs` — `internal sealed class LoginHandler(IApplicationDbContext db, IPasswordHasher passwordHasher, UserSessionService sessions, TimeProvider time)`. Order (lines 29–77):
  1. Look up `db.Users.SingleOrDefaultAsync(u => u.Email == EmailAddress.Normalize(request.Email))`.
  2. Unknown → verify against a static `_timingEqualizerHash` (lazily `Hash(Guid.NewGuid().ToString())`), save, throw `INVALID_CREDENTIALS`.
  3. `user.IsLockedOut(now)` → save, throw `UnauthorizedException(ACCOUNT_LOCKED, ...)` (password not checked).
  4. `Verify` fails → `user.RecordFailedLogin(now)`, **`SaveChangesAsync` before throwing** `INVALID_CREDENTIALS`.
  5. `!user.IsActive` → save, throw `ACCOUNT_DISABLED`.
  6. `SuccessRehashNeeded` → `user.ChangePasswordHash(passwordHasher.Hash(request.Password))`.
  7. `RecordSuccessfulLogin(now)`, `sessions.StartAsync`, save, return.

  The as-built file also has an `IAuditTrail audit` constructor parameter and `audit.Record(...)` before each save (lines 38, 45, 54, 61, 73). Those are Story 06; the save-before-throw calls are the seam it hooks into.

### 8 — Refresh slice (`Features/Authentication/Refresh/`)

- Create file: `RefreshSessionCommand.cs` — `record RefreshSessionCommand(string? RefreshToken) : IRequest<AuthenticatedSession>`.
- Create file: `RefreshSessionEndpoint.cs` — `MapPost("/auth/refresh")` reads the cookie, sends, re-appends the cookie, returns `ApiResults.Ok(session.Response)`; `.AllowAnonymous().RequireCsrfHeader().WithName("RefreshSession").WithTags("Authentication").Produces<ApiResponse<AccessTokenResponse>>()`.
- Create file: `RefreshSessionHandler.cs` (lines 25–70): empty token → `INVALID_REFRESH_TOKEN`; hash and look up (tracked) by `TokenHash` → unknown → invalid; `stored.IsSpent` → `RevokeAsync(t => t.SessionId == stored.SessionId, ReuseDetected)`, save, invalid; `!IsActive(now)` (expired) → invalid; user not active → `RevokeAsync(t => t.UserId == user.Id, UserDisabled)`, save, invalid; else `RotateAsync`, then `SaveChangesAsync` inside `try` and map `DbUpdateConcurrencyException` to `INVALID_REFRESH_TOKEN`. Audit call at line 39 is Story 06.

### 9 — Logout slice (`Features/Authentication/Logout/`)

- Create file: `LogoutCommand.cs` — `record LogoutCommand(string? RefreshToken) : IRequest`.
- Create file: `LogoutEndpoint.cs` — `MapPost("/auth/logout")`: send with `ReadRefreshCookie`, `DeleteRefreshCookie`, `ApiResults.Success()`; `.AllowAnonymous().RequireCsrfHeader().WithName("Logout").WithTags("Authentication").Produces<ApiResponse<object?>>()`.
- Create file: `LogoutHandler.cs` — no token or unknown hash (`AsNoTracking`) → return silently; else `RevokeAsync(t => t.SessionId == stored.SessionId, Logout)` and save. Audit call at line 35 is Story 06.

### 10 — Host, errors, logging

File: `src/CustomerSupportCrm.Api/Program.cs` — after `app.UseHttpsRedirection();` add `app.UseAuthentication();` and `app.UseAuthorization();` (0f87e2d lines 57–58; `UseCors` at 56 and `UseRateLimiter` at 59 are Story 07). Change `app.Run()` to `await app.RunAsync();` (line 64).

File: `src/CustomerSupportCrm.Api/Middleware/GlobalExceptionHandler.cs` — in `StatusFor` add `UnauthorizedException => StatusCodes.Status401Unauthorized,` as the first arm (line 61).

File: `src/CustomerSupportCrm.Api/Configuration/LoggingExtensions.cs` — read the user id from `CrmClaimTypes.Subject` instead of `ClaimTypes.NameIdentifier` (line 60; required because `MapInboundClaims = false`).

### 11 — Configuration and resources

File: `src/CustomerSupportCrm.Api/appsettings.json` — `Jwt` section (lines 20–26): `Issuer "customer-support-crm-api"`, `Audience "customer-support-crm-web"`, `SigningKey ""`, `AccessTokenLifetimeMinutes 15`, `RefreshTokenLifetimeDays 14`.

File: `src/CustomerSupportCrm.Api/appsettings.Development.json` — `Jwt:SigningKey` with a `dev-only-` prefixed value of at least 32 characters (lines 13–15). Production supplies `Jwt__SigningKey` from a secret store.

Files: `src/CustomerSupportCrm.Application/Resources/Messages.resx` and `Messages.ar.resx` — add `INVALID_CREDENTIALS`, `ACCOUNT_LOCKED`, `ACCOUNT_DISABLED`, `INVALID_REFRESH_TOKEN`, `REFRESH_TOKEN_INACTIVE`, `CSRF_VALIDATION_FAILED` (lines 51–68 in both files). English values: "The email or password is incorrect.", "The account is temporarily locked. Try again later.", "The account is disabled.", "The session has expired. Please sign in again." (both refresh codes), "The request could not be verified."

### 12 — Docs

- Create file: `docs/adr/0002-authentication-strategy.md` — own user model, 15-min HS256 access token in memory, rotating refresh cookie, CSRF header, permissions in the token.
- File: `docs/security.md` — sections "Authentication", "Endpoints", "Access token claims", "Refresh-token rotation and reuse detection", "Passwords", "Sign-in protection", "CSRF", and the `Jwt:SigningKey` row under "Secrets and configuration" (lines 5–49, 79).
- File: `docs/api-contract.md` — "Feature error codes (Phase 2)" rows for the five auth codes.

---

## Edge Cases & Failure Modes

- **Unknown email vs wrong password** — same `INVALID_CREDENTIALS` 401. Unknown emails still run a PBKDF2 verify for similar timing (`LoginHandler.cs` line 37).
- **Disabled user** — right password → `ACCOUNT_DISABLED`; wrong password → `INVALID_CREDENTIALS` (the password check runs first, lines 50–64), so disabled status is not revealed without the password.
- **Lockout** — 5th failure sets `LockoutEndsAt = now + 15 min` and resets the counter (`User.cs` lines 111–119). While locked, even the correct password returns `ACCOUNT_LOCKED`. It unlocks once `now >= LockoutEndsAt` (line 108) or on admin enable.
- **Failed attempts persist** — `SaveChangesAsync` runs before every throw. Otherwise the counter would be lost when the exception ends the request.
- **Legacy hash format** — `SuccessRehashNeeded` upgrades the stored hash on successful login (line 66–69).
- **Reuse of a rotated token** — `IsSpent` → the whole session is revoked (`ReuseDetected`); the legitimate holder's newer token stops working as well.
- **Concurrent refresh with one token** — `xmin` token (`RefreshTokenConfiguration.cs` line 32) makes the second `SaveChanges` throw `DbUpdateConcurrencyException` → `INVALID_REFRESH_TOKEN` (`RefreshSessionHandler.cs` lines 59–67). Clients must serialize refreshes.
- **Expired token** — `INVALID_REFRESH_TOKEN`, no revocation. **Missing cookie** → the same code.
- **User disabled after sign-in** — next refresh revokes all of the user's tokens (`UserDisabled`). Access tokens that were already issued stay valid until they expire (≤ 15 min).
- **Missing `X-CSRF-Protection`** on refresh/logout → 403 `CSRF_VALIDATION_FAILED` (`AuthenticationHttp.cs` lines 44–53).
- **Logout twice / without cookie** — no-op, 200 success envelope, cookie deleted.
- **Missing or short signing key** — `ValidateOnStart` on `JwtOptions` (`[Required]`, `[MinLength(32)]`) fails host startup with `OptionsValidationException`. `MinLength` counts characters, not bytes.
- **Cookie over plain HTTP** — `Secure = true`: browsers/curl will not send `crm_refresh` over `http://localhost:5000`; use the `https` profile.
- **Secrets in logs** — request logging writes no bodies, headers or cookies; `GlobalExceptionHandler` logs only the category. The raw refresh token appears only in `Set-Cookie`. Only its SHA-256 hex is stored.

---

## Test Plan

Tests are out of scope by standing directive: this story adds, changes or removes **no** tests. These existing tests in `0f87e2d` cover the story and are **read-only references**:

1. `tests/CustomerSupportCrm.Domain.Tests/Users/RefreshTokenTests.cs` — `IssuedTokenIsActiveUntilExpiry`, `RotateSpendsTokenAndKeepsSession`, `SpentTokenCannotRotateAgain`, `RevokeKeepsFirstReason`.
2. `tests/CustomerSupportCrm.Domain.Tests/Users/UserTests.cs` — `LocksOutAfterMaxFailedAttempts`, `SuccessfulLoginResetsFailuresAndRecordsTime`.
3. `tests/CustomerSupportCrm.Api.Tests/SecurityTests.cs` — `TokenWithBadSignatureIsRejected`, `ValidTokenReachesAuthenticatedEndpoint`, `RefreshWithoutCsrfHeaderIsForbidden`, `LoginValidatesInputBeforeTouchingTheDatabase`.
4. `tests/CustomerSupportCrm.IntegrationTests/AuthenticationFlowTests.cs` — `LoginSetsHardenedRefreshCookieAndReturnsProfile`, `WrongPasswordAndUnknownEmailAreIndistinguishable`, `AccountLocksAfterRepeatedFailuresUntilEnabled`, `RefreshRotatesTokenAndReuseRevokesTheSession`, `RefreshRequiresCsrfHeader`, `LogoutEndsTheSessionAndClearsTheCookie`.

---

## Verification Steps

1. **Backend builds:** in `customer-support-crm-api/` run `dotnet build` — 0 warnings, 0 errors.
2. **Regression:** `dotnet test tests/CustomerSupportCrm.Domain.Tests` and `dotnet test tests/CustomerSupportCrm.Api.Tests` — green.
3. **Startup guard:** `dotnet run --project src/CustomerSupportCrm.Api --launch-profile https` with `Jwt__SigningKey=short` → host fails with an options validation error for `SigningKey`.
4. **Login** (Development, database initialized, admin from `Bootstrap:AdminEmail` / `Bootstrap:AdminPassword` in `appsettings.Development.json`):
   ```bash
   curl -k -c jar.txt -H "Content-Type: application/json" \
     -d '{"email":"admin@crm.local","password":"<Bootstrap:AdminPassword>"}' \
     https://localhost:5001/api/v1/auth/login
   ```
   → 200 with `data.accessToken`, `data.expiresAt`, `data.user.permissions`; `Set-Cookie: crm_refresh=...; path=/api/v1/auth; secure; samesite=strict; httponly`.
5. **Wrong password / unknown email:** both → 401 with `errors[0].code = "INVALID_CREDENTIALS"`, identical bodies apart from the correlation id.
6. **Refresh:** `curl -k -b jar.txt -c jar.txt -X POST -H "X-CSRF-Protection: 1" https://localhost:5001/api/v1/auth/refresh` → 200 and a new cookie. Replaying the **old** cookie value → 401 `INVALID_REFRESH_TOKEN`, and the new cookie then fails too (session revoked). Without the header → 403 `CSRF_VALIDATION_FAILED`.
7. **Logout:** `curl -k -b jar.txt -X POST -H "X-CSRF-Protection: 1" https://localhost:5001/api/v1/auth/logout` → 200 envelope, cookie cleared; a following refresh → 401.
8. **Storage:** `select token_hash, revoked_reason from refresh_tokens;` — 64-char hex hashes only; reasons `ReuseDetected` / `Logout`.
9. **OpenAPI:** `GET https://localhost:5001/openapi/v1.json` lists `Login`, `RefreshSession`, `Logout` with `LoginRequest` / `ApiResponse<AccessTokenResponse>` schemas.
10. **Logs:** search the console output of steps 4–7 for the password and the cookie value — no matches.

---

## Done Criteria

- [x] Login returns an access token + user profile and sets the refresh token as the `crm_refresh` HttpOnly cookie for valid credentials.
- [x] Unknown email and wrong password get the identical 401 `INVALID_CREDENTIALS`; a disabled user gets `ACCOUNT_DISABLED` only after the correct password.
- [x] Account locks after 5 failures (`User.MaxFailedLoginAttempts`) and unlocks after 15 minutes.
- [x] Refresh rotates the token; the old one cannot be reused; reuse revokes the whole session.
- [x] Only SHA-256 hashes of refresh tokens are stored.
- [x] Logout revokes the refresh-token session and clears the cookie.
- [x] App fails to start when `Jwt:SigningKey` is missing or shorter than 32 characters.
- [x] Passwords and tokens never appear in logs (Serilog request logging and `GlobalExceptionHandler` checked).
- [x] Endpoints appear in OpenAPI with request/response schemas.
- [x] Nothing changed in `tests/`, Docker, `deploy/` or `.github/` by this story.
- [x] `dotnet build` passes with zero warnings.

**STOP HERE. Report to the user and wait for confirmation before proceeding to Story 03.**
