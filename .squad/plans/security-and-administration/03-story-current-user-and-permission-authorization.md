# Story 03 — Current user abstraction and permission-based authorization (Story: P2-03)

> As-built plan: written after implementation in `customer-support-crm-api` commit `0f87e2d`; paths and line numbers refer to that commit.

## Prerequisites

- [01-story-identity-domain-persistence.md](01-story-identity-domain-persistence.md) completed: `Domain/Roles/Permissions.cs` catalog, `UserId`, `IApplicationDbContext` with `Users` / `Roles`.
- [02-story-authentication-jwt-and-refresh-tokens.md](02-story-authentication-jwt-and-refresh-tokens.md) completed: `CrmClaimTypes`, `TokenService`, JwtBearer wiring (`MapInboundClaims = false`), `UnauthorizedException`, `UserAccessProfile`, `CurrentUserResponse`, the `/auth` slices.
- Followed by [04-story-user-management.md](04-story-user-management.md), [05-story-role-and-permission-management.md](05-story-role-and-permission-management.md), [06-story-audit-logging.md](06-story-audit-logging.md) and [07-story-security-hardening.md](07-story-security-hardening.md). All of them consume `ICurrentUser` and the permission policies from this story.
- All paths are relative to the **`customer-support-crm-api/`** repo root.

---

## Story Goal

Give the Application layer a framework-free view of the caller, and enforce permission-based authorization on every endpoint:

1. `ICurrentUser` in `Application/Abstractions/Authentication`, implemented by `HttpCurrentUser` in Infrastructure from the JWT claims.
2. One authorization policy per permission code (policy name = code), plus composite policies in `PolicyNames`. No role-name checks anywhere.
3. Secure by default: the `/api/v1` group requires an authenticated user; login, refresh and logout opt out with `AllowAnonymous`.
4. 401 / 403 responses use the standard envelope (`UNAUTHORIZED` / `FORBIDDEN`) with `correlationId`, via the Phase 1 status-code pages writer.
5. `GET /api/v1/auth/me` returns the caller's profile, roles and permissions read from the database.
6. OpenAPI declares the Bearer scheme so Swagger UI has an **Authorize** button.

**Deviations from the intake (as built):**

- No custom `IAuthorizationPolicyProvider` and no `perm:<code>` prefix. Policies are registered up front in `AuthorizationSetup.AddPermissionPolicies`, one per `Permissions.All` code, and the policy name is the bare code.
- No `RequirePermission` extension. Endpoints call `.RequireAuthorization(Permissions.UsersManage)` directly.
- The claim is `permission`, not `perm`. There is no `culture` claim: `Culture` comes from `CultureInfo.CurrentUICulture` (request localization).
- `ICurrentUser` has no `Email` / `FullName`. `UserId` is a non-nullable `UserId` that throws `UnauthorizedException` when anonymous. `SessionId` was added so later stories can target the caller's session.
- The adapter lives in **Infrastructure** (`Infrastructure/Authentication/HttpCurrentUser.cs`), not Api.
- `CurrentUserResponse` has `DisplayName` and no `Culture`. `/me` returns 401 when the user row no longer exists. It does **not** check `IsActive`: disabled users are cut off at refresh (see Edge Cases).
- There are no JwtBearer `OnChallenge` / `OnForbidden` handlers. The empty-body 401/403 from the auth middleware is wrapped by the existing `UseStatusCodePages(ErrorResponseWriter.WriteStatusCodePageAsync)`, so the body is written once.
- Permission freshness is documented in `docs/security.md` ("Revocation latency"), not `docs/architecture.md`.

**Not in scope:** branch/department scoped authorization (Phase 3), user/role endpoints (P2-04, P2-05). **Do not touch** `docker-compose.yml`, `deploy/`, `.github/` or anything under `tests/`.

---

## Context — Read These Files First

1. `src/CustomerSupportCrm.Application/Abstractions/Authentication/CrmClaimTypes.cs` — lines 1–12 (P2-02). Claim names `sub`, `email`, `name`, `role`, `permission`, `sid`. Use these constants and never hard-code the strings.
2. `src/CustomerSupportCrm.Infrastructure/Authentication/TokenService.cs` — lines 23–53 (P2-02). `CreateAccessToken` writes one `role` claim per role and one `permission` claim per permission (lines 37–38). This story reads exactly those claims.
3. `src/CustomerSupportCrm.Infrastructure/DependencyInjection.cs` — lines 65–103. `AddIdentityServices`: the JwtBearer config (lines 72–94, P2-02) sets `MapInboundClaims = false` and `RoleClaimType = CrmClaimTypes.Role`. You add lines 96 and 100 (task 4).
4. `src/CustomerSupportCrm.Domain/Roles/Permissions.cs` — lines 1–37 (P2-01). Flat constants (`UsersManage`, `RolesManage`, `AuditView`, …) and `All` (13 codes, lines 26–32). **Not** nested classes, and not in `Domain/Users`.
5. `src/CustomerSupportCrm.Application/Features/Authentication/Common/UserAccessProfile.cs` — lines 8–23 (P2-02). `LoadAsync(db, userId, ct)` returns sorted role names and the distinct union of their permissions. Refresh already uses it through `UserSessionService.BuildAsync` (`UserSessionService.cs` lines 62–73), so refresh re-reads roles/permissions from the DB.
6. `src/CustomerSupportCrm.Application/Common/Exceptions/UnauthorizedException.cs` — lines 1–6 (P2-02). The default code is `ErrorCodes.Unauthorized`. `GlobalExceptionHandler.StatusFor` maps it to 401 at line 61, and `ForbiddenException` to 403 at line 64 (Phase 1).
7. `src/CustomerSupportCrm.Api/Middleware/ErrorResponseWriter.cs` — lines 29–35 and 44–58 (Phase 1, unchanged). `WriteStatusCodePageAsync` produces the envelope for empty-body responses, and `CategoryFor` maps 401 → `UNAUTHORIZED` and 403 → `FORBIDDEN`. The localized messages already exist in `Messages.resx` lines 21–26.
8. `src/CustomerSupportCrm.Api/Program.cs` at `c3c7815` — lines 26–40. `UseStatusCodePages` (line 30) runs **before** the endpoint pipeline, so auth challenges get the envelope with no extra code.
9. `src/CustomerSupportCrm.Api/Endpoints/EndpointExtensions.cs` at `c3c7815` — lines 10–20. `MapApiEndpoints` creates the `/api/v1` group at line 12.
10. `src/CustomerSupportCrm.Api/OpenApi/OpenApiExtensions.cs` at `c3c7815` — lines 7–13. This is the document transformer to extend.
11. `src/CustomerSupportCrm.Application/Features/Authentication/Login/LoginEndpoint.cs` — lines 14–25 (P2-02). This is the endpoint pattern: `AuthenticationHttp.RoutePrefix` (`"/auth"`), `.AllowAnonymous()`, `.WithName/WithTags/Produces<ApiResponse<T>>`.

---

## Backend Tasks

### 1 — `ICurrentUser`

Create file: `src/CustomerSupportCrm.Application/Abstractions/Authentication/ICurrentUser.cs` (27 lines)

```csharp
/// Organization, branch and department context is added in Phase 3.
public interface ICurrentUser
{
    bool IsAuthenticated { get; }
    UserId UserId { get; }            // throws when anonymous
    Guid? SessionId { get; }
    IReadOnlyCollection<string> Roles { get; }
    IReadOnlyCollection<string> Permissions { get; }
    CultureInfo Culture { get; }
    bool HasPermission(string permission);
}
```

Phase 3 adds scope members as new properties, so existing callers keep compiling.

### 2 — `HttpCurrentUser` adapter

Create file: `src/CustomerSupportCrm.Infrastructure/Authentication/HttpCurrentUser.cs` (35 lines): `internal sealed class HttpCurrentUser(IHttpContextAccessor accessor) : ICurrentUser`.

- `Principal => accessor.HttpContext?.User` (line 13). This is the **only** place that reads `HttpContext.User` outside the Api logging enricher.
- `UserId` (lines 17–20): `Guid.TryParse(Principal?.FindFirstValue(CrmClaimTypes.Subject), …)` → `new UserId(id)`, otherwise `throw new UnauthorizedException()`.
- `SessionId` (lines 22–23): parse `CrmClaimTypes.SessionId`, `null` on failure.
- `Roles` / `Permissions` (lines 25–27): all values of `CrmClaimTypes.Role` / `CrmClaimTypes.Permission` through the private `ValuesOf` (lines 33–34), which returns `[]` when there is no principal.
- `Culture => CultureInfo.CurrentUICulture` (line 29).
- `HasPermission(p) => Principal?.HasClaim(CrmClaimTypes.Permission, p) == true` (line 31).

### 3 — Permission policies

Create file: `src/CustomerSupportCrm.Application/Abstractions/Authorization/PolicyNames.cs` (11 lines). Delete `Abstractions/Authorization/.gitkeep`.

```csharp
public static class PolicyNames
{
    /// <summary>Read roles: needed both to manage roles and to assign them to users.</summary>
    public const string RolesRead = "policy:roles.read";
}
```

Create file: `src/CustomerSupportCrm.Infrastructure/Authorization/AuthorizationSetup.cs` (29 lines)

```csharp
internal static class AuthorizationSetup
{
    public static void AddPermissionPolicies(this IServiceCollection services)
    {
        var builder = services.AddAuthorizationBuilder();
        foreach (var permission in Permissions.All)
        {
            builder.AddPolicy(permission, policy => policy
                .RequireAuthenticatedUser()
                .RequireClaim(CrmClaimTypes.Permission, permission));
        }

        builder.AddPolicy(PolicyNames.RolesRead, policy => policy
            .RequireAuthenticatedUser()
            .RequireClaim(CrmClaimTypes.Permission, Permissions.RolesManage, Permissions.UsersManage));
    }
}
```

`RequireClaim` with several values means **any of** them. Composite policies go in `PolicyNames` with a `policy:` prefix, so their names cannot clash with a permission code.

### 4 — DI registration

File: `src/CustomerSupportCrm.Infrastructure/DependencyInjection.cs`

- Line 29 in `AddInfrastructure`: `services.AddHttpContextAccessor();`.
- Line 96 in `AddIdentityServices`, after the JwtBearer options: `services.AddPermissionPolicies();`.
- Line 100: `services.AddScoped<ICurrentUser, HttpCurrentUser>();`.
- Usings: `CustomerSupportCrm.Infrastructure.Authorization` (line 7).

### 5 — Secure-by-default route group

File: `src/CustomerSupportCrm.Api/Endpoints/EndpointExtensions.cs` — line 15:

```csharp
var v1 = app.MapGroup(ApiV1Prefix).RequireAuthorization();
```

Update the summary (lines 9–12) to say that endpoints opt out with `AllowAnonymous`. Anonymous endpoints at `0f87e2d`: `LoginEndpoint` (line 21), `RefreshSessionEndpoint` (line 21) and `LogoutEndpoint` (line 21), which are cookie + CSRF-header authenticated. Feature endpoints add `.RequireAuthorization(Permissions.X)` or `.RequireAuthorization(PolicyNames.RolesRead)` on top of the group requirement (e.g. `ListPermissionsEndpoint.cs` line 24).

### 6 — Pipeline

File: `src/CustomerSupportCrm.Api/Program.cs` — lines 57–58, after `UseHttpsRedirection` (line 54) and before `MapApiHealthChecks` (line 61):

```csharp
app.UseAuthentication();
app.UseAuthorization();
```

`UseCors` (line 56) and `UseRateLimiter` (line 59) belong to P2-07. Keep `UseStatusCodePages` (line 47) ahead of them. Do **not** add JwtBearer `OnChallenge` / `OnForbidden` handlers: they would write a second body.

### 7 — Log enricher claim

File: `src/CustomerSupportCrm.Api/Configuration/LoggingExtensions.cs` — line 60: replace `ClaimTypes.NameIdentifier` with `CrmClaimTypes.Subject` (add using `CustomerSupportCrm.Application.Abstractions.Authentication`, line 3). With inbound claim mapping disabled, `NameIdentifier` is never present.

### 8 — `GET /api/v1/auth/me` slice

Create folder `src/CustomerSupportCrm.Application/Features/Authentication/GetCurrentUser/` with these files:

- `GetCurrentUserQuery.cs` (6 lines): `public sealed record GetCurrentUserQuery : IRequest<CurrentUserResponse>;`
- `GetCurrentUserHandler.cs` (28 lines):

```csharp
internal sealed class GetCurrentUserHandler(IApplicationDbContext db, ICurrentUser currentUser)
    : IRequestHandler<GetCurrentUserQuery, CurrentUserResponse>
{
    public async Task<CurrentUserResponse> Handle(GetCurrentUserQuery request, CancellationToken cancellationToken)
    {
        var userId = currentUser.UserId;
        var user = await db.Users.AsNoTracking()
            .Where(u => u.Id == userId)
            .Select(u => new { u.Email, u.DisplayName })
            .SingleOrDefaultAsync(cancellationToken)
            ?? throw new UnauthorizedException();

        var profile = await UserAccessProfile.LoadAsync(db, userId, cancellationToken);
        return new CurrentUserResponse(userId.Value, user.Email, user.DisplayName, profile.Roles, profile.Permissions);
    }
}
```

- `GetCurrentUserEndpoint.cs` (20 lines): `internal sealed class GetCurrentUserEndpoint : IEndpoint` → `app.MapGet($"{AuthenticationHttp.RoutePrefix}/me", …)` returning `ApiResults.Ok(await sender.Send(new GetCurrentUserQuery(), ct))`, `.WithName("GetCurrentUser")`, `.WithTags("Authentication")`, `.Produces<ApiResponse<CurrentUserResponse>>()`. There is **no** extra `RequireAuthorization`: the group requirement is enough.

`CurrentUserResponse` already exists in `src/CustomerSupportCrm.Contracts/Authentication/AuthenticationContracts.cs` lines 13–18 (P2-02): `Id`, `Email`, `DisplayName`, `Roles`, `Permissions`. No validator: the query has no input.

### 9 — OpenAPI Bearer scheme

File: `src/CustomerSupportCrm.Api/OpenApi/OpenApiExtensions.cs` — add `using Microsoft.OpenApi;` (line 1) and `private const string BearerScheme = "Bearer";` (line 8). Inside the document transformer (lines 16–31):

```csharp
document.Components ??= new OpenApiComponents();
document.Components.SecuritySchemes ??= new Dictionary<string, IOpenApiSecurityScheme>();
document.Components.SecuritySchemes[BearerScheme] = new OpenApiSecurityScheme
{
    Type = SecuritySchemeType.Http, Scheme = "bearer", BearerFormat = "JWT",
    Description = "Access token from POST /api/v1/auth/login.",
};
document.Security ??= [];
document.Security.Add(new OpenApiSecurityRequirement
{
    [new OpenApiSecuritySchemeReference(BearerScheme, document)] = [],
});
```

As built, the requirement is document-level. Anonymous endpoints are **not** marked individually, and Swagger UI still calls them without a token.

### 10 — Docs

- `docs/security.md` — "Revocation latency" (lines 43–45): permissions and disablement take effect at the next refresh, and access tokens stay valid for up to 15 min. "Authorization" (lines 55–69): secure by default, permission code = policy name, `PolicyNames`, permission table. Endpoints table row for `GET /api/v1/auth/me` (line 19).
- `docs/architecture.md` — pipeline rows for Authentication / Authorization, `ICurrentUser` → `HttpCurrentUser` in the "Application abstractions" table, `UnauthorizedException` in the handler exceptions sentence.

---

## Edge Cases & Failure Modes

- **No / malformed / expired / wrongly signed token on a protected route**: JwtBearer challenges with an empty 401, and `ErrorResponseWriter.WriteStatusCodePageAsync` (lines 30–35) writes `UNAUTHORIZED` with `correlationId`. Validation parameters are in `DependencyInjection.cs` lines 80–93 (`ClockSkew` 30 s).
- **Valid token without the permission**: the policy fails, the empty 403 goes through the same writer, and the response is `FORBIDDEN`.
- **Handler throws `ForbiddenException` / `UnauthorizedException`**: `GlobalExceptionHandler.StatusFor` lines 61–64 maps it to 403 / 401 with the exception's own code (e.g. `ACCOUNT_LOCKED`).
- **`ICurrentUser.UserId` read on an anonymous request** (e.g. an `AllowAnonymous` handler): `UnauthorizedException` → 401. Check `IsAuthenticated` first where anonymity is legal. `AuditableEntityInterceptor` and `AuditTrail` (P2-06) must do the same.
- **`AllowAnonymous` on a parent group**: it overrides child `RequireAuthorization`. See the comment in `tests/CustomerSupportCrm.Api.Tests/TestEndpoints.cs` lines 69–72.
- **Unknown policy name** (a typo in `RequireAuthorization("...")`): ASP.NET throws `InvalidOperationException` at request time → 500. Always pass `Permissions.*` / `PolicyNames.*` constants.
- **Stale permissions**: a token keeps its `permission` claims until it expires (≤ `Jwt:AccessTokenLifetimeMinutes`, 15). `/me` reads the DB, so it can show fewer permissions than the token grants until refresh. Refresh re-reads through `UserAccessProfile`.
- **User disabled while holding a token**: `/me` still returns 200 until expiry. The as-built handler only checks that the row exists. Refresh fails because P2-04 revokes the user's refresh tokens on disable.
- **User row deleted**: `/me` → 401 `UNAUTHORIZED` (`GetCurrentUserHandler.cs` line 23).
- **Claim-name mapping**: if `MapInboundClaims` were re-enabled, `sub` would become `NameIdentifier`, and `HttpCurrentUser` and the log enricher would silently lose the user id. Keep it `false`.

---

## Test Plan

No tests are added, changed or removed (`tests/` is out of scope by standing directive). These tests already exist in `0f87e2d` and cover this story (read-only references):

1. `tests/CustomerSupportCrm.Api.Tests/SecurityTests.cs` (unit / host level): `ProtectedEndpointWithoutTokenReturns401Envelope` (line 19), `TokenWithBadSignatureIsRejected` (27), `ValidTokenReachesAuthenticatedEndpoint` (38), `MissingPermissionReturns403Envelope` (46), `GrantedPermissionIsAuthorized` (54), `FeatureEndpointsRequireAuthenticationByDefault` (62), `OpenApiDeclaresBearerScheme` (103). `AssertErrorAsync` (lines 136–143) checks `success == false` and a non-empty `correlationId`.
2. `tests/CustomerSupportCrm.Api.Tests/TestEndpoints.cs` lines 69–72: the `/_test-secure/authenticated` and `/_test-secure/users-manage` fixtures used above.
3. `tests/CustomerSupportCrm.IntegrationTests/AuthenticationFlowTests.cs`: `LoginSetsHardenedRefreshCookieAndReturnsProfile` (line 14) exercises the profile shape shared with `/me`.

---

## Verification Steps

1. **Backend builds:** in `customer-support-crm-api/` run `dotnet build`. Expect 0 warnings and 0 errors.
2. **Regression:** `dotnet test tests/CustomerSupportCrm.Api.Tests` — `SecurityTests` pass.
3. **Run:** `dotnet run --project src/CustomerSupportCrm.Api --launch-profile http` (Development, `InitializeOnStartup: true` migrates and seeds the bootstrap admin from `appsettings.Development.json`).
4. **401 envelope:** `curl -i http://localhost:5000/api/v1/auth/me` → `401`, body `{"success":false,…,"errors":[{"code":"UNAUTHORIZED",…}],"correlationId":"…"}`, exactly one JSON body.
5. **Token:** `curl -s -X POST http://localhost:5000/api/v1/auth/login -H "Content-Type: application/json" -d '{"email":"<Bootstrap:AdminEmail>","password":"<Bootstrap:AdminPassword>"}'` and copy `data.accessToken`.
6. **Me:** `curl -s http://localhost:5000/api/v1/auth/me -H "Authorization: Bearer $TOKEN"` → `200`, `data` has `id`, `email`, `displayName`, `roles: ["Administrator"]` and all 13 permission codes.
7. **403 envelope:** sign in as a user whose role lacks `audit.view`, then `curl -i http://localhost:5000/api/v1/audit-logs -H "Authorization: Bearer $TOKEN"` → `403`, `errors[0].code == "FORBIDDEN"`.
8. **Swagger:** open `http://localhost:5000/swagger`. The **Authorize** button is present, and after pasting the token, `GET /api/v1/auth/me` succeeds.
9. **No `HttpContext.User` in Application:** `git grep -n "HttpContext.User\|\.User\.FindFirst" -- src/CustomerSupportCrm.Application` → no matches.

---

## Done Criteria

- [x] `ICurrentUser` is available in handlers, and no Application code references `HttpContext.User`.
- [x] Every `/api/v1` endpoint requires authentication unless it is explicitly `AllowAnonymous` (login, refresh, logout).
- [x] A missing permission returns the 403 `FORBIDDEN` envelope (as built: `.RequireAuthorization(Permissions.X)` instead of `RequirePermission`).
- [x] A missing, invalid or expired token returns the 401 `UNAUTHORIZED` envelope with `correlationId`.
- [x] `GET /api/v1/auth/me` returns the caller's profile, roles and permissions.
- [x] Swagger UI can authorize with a bearer token.
- [x] `dotnet build` passes with zero warnings.
- [x] Nothing changed in `tests/`, `docker-compose.yml`, `deploy/` or `.github/` by this story.

**STOP HERE. Report to the user and wait for confirmation before proceeding to Story 04.**
