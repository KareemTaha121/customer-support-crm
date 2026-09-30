# Story 07 — Security hardening: rate limiting, CORS, secure headers, request limits (Story: P2-07)

> As-built plan: written after implementation in `customer-support-crm-api` commit `0f87e2d`; paths and line numbers refer to that commit.
>
> **Follow-up:** the items `0f87e2d` left open (secure headers, HSTS, Server header, configurable body limit, CORS startup check, global rate limiter) were completed in commit `2956767`: `Api/Middleware/SecureHeadersMiddleware.cs`, `GlobalRateLimitOptions`, `RequestLimitOptions` and `AddApiRequestLimits` in `Api/Configuration/SecurityExtensions.cs`, and `Program.cs`. The CORS check exempts the `Test` environment so the integration host still starts.

## Prerequisites

- Phase 1 (Backend Platform) completed in `customer-support-crm-api` (commit `c3c7815`): `ErrorResponseWriter`, `GlobalExceptionHandler`, status-code pages, correlation id, en/ar `Messages.resx`.
- [01-story-identity-domain-persistence.md](01-story-identity-domain-persistence.md) completed.
- [02-story-authentication-jwt-and-refresh-tokens.md](02-story-authentication-jwt-and-refresh-tokens.md) completed: auth endpoints exist, the refresh token lives in the `crm_refresh` **HttpOnly cookie**, refresh/logout require the `X-CSRF-Protection` header.
- [03-story-current-user-and-permission-authorization.md](03-story-current-user-and-permission-authorization.md) completed: `UseAuthentication` / `UseAuthorization` are in the pipeline, 401/403 use the envelope.
- Siblings for reference: [04-story-user-management.md](04-story-user-management.md), [05-story-role-and-permission-management.md](05-story-role-and-permission-management.md), [06-story-audit-logging.md](06-story-audit-logging.md).
- All paths below are relative to the **`customer-support-crm-api/`** repo root.

---

## Story Goal

Harden the API host with the pieces that shipped in `0f87e2d`:

1. **Rate limiting** — ASP.NET Core built-in `RateLimiter`, one named policy `authentication`: fixed window **per client IP**, default **10 requests / 60 s**, options validated on start, applied to **login, refresh and change-password**. Rejections return **429** in the standard envelope (`RATE_LIMITED`, localized message, `correlationId`) with a `Retry-After` header.
2. **CORS** — `CorsSettings` (section `Cors`) with `AllowedOrigins[]`; a **default** policy with explicit origins only, any header/method, **credentials allowed**, exposed headers `X-Correlation-Id`, `Content-Language`, `Retry-After`. Development allows `http(s)://localhost:4200`.
3. **Pipeline order** — `UseCors` → `UseAuthentication` → `UseAuthorization` → `UseRateLimiter`, after exception handling / status-code pages / HTTPS redirection.
4. **Docs** — pipeline, configuration table and `docs/security.md` CORS / sign-in protection sections.

**Deviations from the intake (as built):**

- Options class is **`CorsSettings`**, not `CorsOptions` — `Microsoft.AspNetCore.Cors.Infrastructure.CorsOptions` is the framework type configured from it. Rate-limit section is **`RateLimiting:Authentication`**, policy name **`authentication`** (`RateLimitPolicies.Authentication`), not `auth`.
- CORS **allows credentials** and **any header**: P2-02 moved the refresh token to an HttpOnly cookie, which needs credentialed CORS (hence no wildcard). A default policy is used instead of a named one.
- Rate limiting also covers **change-password**; there is **no global per-user limiter**.
- **Not delivered in `0f87e2d`** (no code exists for them): secure-headers middleware (nosniff / X-Frame-Options / Referrer-Policy / CSP), `UseHsts`, Kestrel `AddServerHeader = false`, configurable `MaxRequestBodySize`, and failing startup when `Cors:AllowedOrigins` is empty outside Development. The 413 `PAYLOAD_TOO_LARGE` envelope already exists from Phase 1 (Kestrel default limit of 30,000,000 bytes).
- `docs/development.md` got no CORS/rate-limit notes; the configuration notes went into `docs/architecture.md` and `docs/security.md`.

**Not in scope:** portal/AI/upload rate limits, dependency scanning. **Do not touch** `docker-compose.yml`, `deploy/`, `.github/` or anything under `tests/`.

---

## Context — Read These Files First

1. `src/CustomerSupportCrm.Api/Program.cs` at `c3c7815` — lines 24–40. Phase 1 pipeline: `CorrelationIdMiddleware` → `UseRequestLocalization` → `UseApiRequestLogging` → `UseExceptionHandler` → `UseStatusCodePages(ErrorResponseWriter.WriteStatusCodePageAsync)` → OpenAPI (Development) → `UseHttpsRedirection` → endpoints.
2. `src/CustomerSupportCrm.Api/Middleware/ErrorResponseWriter.cs` — lines 29–58 (unchanged since Phase 1). `WriteStatusCodePageAsync` turns empty-body responses into the envelope; `CategoryFor` maps **429 → `ErrorCodes.RateLimited`** (line 55) and **413 → `ErrorCodes.PayloadTooLarge`** (line 52). This is why the limiter only sets the status code and `Retry-After`.
3. `src/CustomerSupportCrm.Api/Middleware/GlobalExceptionHandler.cs` — lines 37–38. `BadHttpRequestException` (e.g. Kestrel body too large, 413) is mapped via `CategoryFor`.
4. `src/CustomerSupportCrm.Application/Resources/Messages.resx` — lines 36–37 `PAYLOAD_TOO_LARGE`, lines 45–46 `RATE_LIMITED` (Arabic in `Messages.ar.resx` line 45). Messages already exist; no resource changes.
5. `src/CustomerSupportCrm.Application/Abstractions/Http/CorrelationIdHttpContextExtensions.cs` — line 7 `HeaderName = "X-Correlation-Id"`; reuse it for the exposed CORS header.
6. `src/CustomerSupportCrm.Application/Features/Authentication/Common/AuthenticationHttp.cs` — lines 13–18. `RefreshCookieName = "crm_refresh"`, `CsrfHeaderName = "X-CSRF-Protection"`, cookie path `/api/v1/auth`: the reason CORS needs `AllowCredentials()` and must accept the custom CSRF header.
7. `src/CustomerSupportCrm.Application/Features/Authentication/Login/LoginEndpoint.cs` — lines 14–25; `Refresh/RefreshSessionEndpoint.cs` — lines 14–26; `ChangePassword/ChangePasswordEndpoint.cs` — lines 14–23. Each already chains `.RequireRateLimiting(RateLimitPolicies.Authentication)` (lines 22, 23, 20) — added with P2-02; this story provides the policy they name.
8. `src/CustomerSupportCrm.Infrastructure/Persistence/DatabaseOptions.cs` / `Authentication/JwtOptions.cs` — options pattern (`SectionName` const, `init` properties, DataAnnotations, `ValidateOnStart`).
9. `tests/CustomerSupportCrm.Api.Tests/ApiFactory.cs` (**read only**) — `protected virtual int RateLimitPermits => 1_000` and `builder.UseSetting("RateLimiting:Authentication:PermitLimit", …)`: the tests override the limit through the configuration key, so the section name and `PermitLimit` property name are a contract.

---

## Backend Tasks

### 1 — Policy name constant

Create file: `src/CustomerSupportCrm.Application/Abstractions/Http/RateLimitPolicies.cs` (lives in Application because endpoints are in Application slices)

```csharp
namespace CustomerSupportCrm.Application.Abstractions.Http;

public static class RateLimitPolicies
{
    /// <summary>Strict per-client limit for login, refresh and password endpoints.</summary>
    public const string Authentication = "authentication";
}
```

### 2 — Options and service registration

Create file: `src/CustomerSupportCrm.Api/Configuration/SecurityExtensions.cs` (91 lines). Usings: `System.ComponentModel.DataAnnotations`, `System.Globalization`, `System.Threading.RateLimiting`, `CustomerSupportCrm.Application.Abstractions.Http`, `Microsoft.AspNetCore.Cors.Infrastructure`, `Microsoft.AspNetCore.RateLimiting`, `Microsoft.Extensions.Options`. Namespace `CustomerSupportCrm.Api.Configuration`.

Options (lines 11–28):

```csharp
public sealed class CorsSettings
{
    public const string SectionName = "Cors";

    /// <summary>Exact frontend origins, e.g. https://crm.example.com. Never "*".</summary>
    public string[] AllowedOrigins { get; init; } = [];
}

public sealed class RateLimitOptions
{
    public const string SectionName = "RateLimiting:Authentication";

    [Range(1, 10_000)]
    public int PermitLimit { get; init; } = 10;

    [Range(1, 3_600)]
    public int WindowSeconds { get; init; } = 60;
}
```

`internal static class SecurityExtensions`:

**`AddApiCors`** (lines 36–48) — bind `CorsSettings` (no validation), `services.AddCors()`, then configure the framework `CorsOptions` from the bound settings so config overrides (tests, env vars) apply:

```csharp
services.AddOptions<CorsOptions>()
    .Configure<IOptions<CorsSettings>>((cors, settings) => cors.AddDefaultPolicy(policy => policy
        .WithOrigins(settings.Value.AllowedOrigins)
        .AllowAnyHeader()
        .AllowAnyMethod()
        .AllowCredentials()
        .WithExposedHeaders(CorrelationIdHttpContextExtensions.HeaderName, "Content-Language", "Retry-After")));
```

**`AddApiRateLimiting`** (lines 54–90):

- `AddOptions<RateLimitOptions>().BindConfiguration(...).ValidateDataAnnotations().ValidateOnStart()` (lines 56–59).
- `services.AddRateLimiter(options => …)`:
  - `options.RejectionStatusCode = StatusCodes.Status429TooManyRequests;` — **no body written**, so status-code pages produce the `RATE_LIMITED` envelope.
  - `OnRejected` (lines 64–73): when `context.Lease.TryGetMetadata(MetadataName.RetryAfter, out var retryAfter)`, set `Response.Headers.RetryAfter` to `Math.Ceiling(retryAfter.TotalSeconds)` formatted with `CultureInfo.InvariantCulture`; return `ValueTask.CompletedTask`.
  - `options.AddPolicy(RateLimitPolicies.Authentication, context => …)` (lines 75–86): resolve `IOptions<RateLimitOptions>` from `context.RequestServices`, return `RateLimitPartition.GetFixedWindowLimiter(context.Connection.RemoteIpAddress?.ToString() ?? "unknown", _ => new FixedWindowRateLimiterOptions { PermitLimit, Window = TimeSpan.FromSeconds(WindowSeconds), QueueLimit = 0 })`.

No new package: rate limiting and CORS are in the ASP.NET Core shared framework.

### 3 — Host wiring and pipeline order

File: `src/CustomerSupportCrm.Api/Program.cs`

- Service chain (lines 16–25): add `.AddApiCors()` and `.AddApiRateLimiting()` after `.AddApiOpenApi()` (lines 20–21).
- Middleware (lines 54–59), after `app.UseHttpsRedirection();`:

```csharp
app.UseCors();
app.UseAuthentication();
app.UseAuthorization();
app.UseRateLimiter();
```

`UseRateLimiter` **after** `UseAuthorization` and before `MapApiHealthChecks` / `MapApiEndpoints` (lines 61–62). Health endpoints carry no rate-limit metadata, so they are never limited. Resulting order: correlation → localization → logging → exception handler → status-code pages → (OpenAPI, Development) → HTTPS redirection → CORS → authentication → authorization → rate limiter → endpoints.

### 4 — Endpoint metadata

No new edits here: `LoginEndpoint.cs` line 22, `RefreshSessionEndpoint.cs` line 23 and `ChangePasswordEndpoint.cs` line 20 already call `.RequireRateLimiting(RateLimitPolicies.Authentication)` (Context item 7). If implementing from scratch, add that call to those three endpoints.

### 5 — Configuration

File: `src/CustomerSupportCrm.Api/appsettings.json` — lines 27–35:

```json
"Cors": {
  "AllowedOrigins": []
},
"RateLimiting": {
  "Authentication": {
    "PermitLimit": 10,
    "WindowSeconds": 60
  }
},
```

File: `src/CustomerSupportCrm.Api/appsettings.Development.json` — lines 16–21:

```json
"Cors": {
  "AllowedOrigins": [
    "http://localhost:4200",
    "https://localhost:4200"
  ]
},
```

### 6 — Docs

File: `docs/architecture.md`

- Request pipeline block (lines 26–39): `StatusCodePages` row now lists `401, 403, 429`; add rows `Cors` ("configured frontend origins only, with credentials"), `Authentication`, `Authorization`, `RateLimiter` ("per-IP limit on sign-in endpoints") before `Endpoint`.
- Configuration table (lines 95–103): rows `Cors` → `CorsSettings`, `RateLimiting:Authentication` → `RateLimitOptions`.

File: `docs/security.md`

- "Sign-in protection" (line 36), bullet at line 41: login, refresh and change-password share a fixed-window per-IP limit (default 10/min) → `429 RATE_LIMITED` with `Retry-After`.
- "## CORS" (lines 51–53): configured origins only, with credentials, empty by default, never wildcard; exposed headers.
- Secrets table line 82 (`Cors:AllowedOrigins`) and deployment note line 87 (configure forwarded headers behind a proxy so the limiter sees the client IP).

---

## Edge Cases & Failure Modes

- **11th login in a window from one IP** — fixed-window permit exhausted → `RejectionStatusCode` 429, `OnRejected` adds `Retry-After` (`SecurityExtensions.cs` 63–73), envelope written by `ErrorResponseWriter.WriteStatusCodePageAsync` / `CategoryFor` line 55. Message localized by `Accept-Language` (`RATE_LIMITED` in both resx files).
- **Invalid body still consumes a permit** — the limiter runs before the endpoint, so 400 validation failures count (the test relies on this).
- **Refresh / change-password share the same partition** — one IP bucket per policy across the three endpoints; a burst of refreshes can block login from the same IP for the rest of the window.
- **Unknown remote IP** (`RemoteIpAddress` null, e.g. some test hosts) — partition key `"unknown"`; all such clients share one bucket (line 79).
- **Behind a reverse proxy** — without forwarded headers every client has the proxy IP and shares one bucket; documented in `docs/security.md` line 87, not configured in code.
- **Invalid limits in config** (`PermitLimit = 0`, `WindowSeconds = 99999`) — `ValidateDataAnnotations().ValidateOnStart()` fails startup with `OptionsValidationException` (lines 56–59).
- **Unknown origin preflight** — `WithOrigins` does not match → no `Access-Control-Allow-Origin` header; browser blocks the call.
- **Empty `Cors:AllowedOrigins` outside Development** — **not validated in `0f87e2d`**: startup succeeds and every cross-origin browser request is refused. Same-origin and non-browser clients are unaffected.
- **`"*"` in `AllowedOrigins`** — `WithOrigins("*")` combined with `AllowCredentials()` makes the CORS service throw when the policy is built; do not configure a wildcard.
- **Oversized body** — Kestrel default limit (30,000,000 bytes, not configurable via app config in this commit) raises `BadHttpRequestException` 413 → `PAYLOAD_TOO_LARGE` envelope (`GlobalExceptionHandler.cs` 37–38, `ErrorResponseWriter.cs` 52).
- **Health endpoints** — no `RequireRateLimiting` and no global limiter → never 429.

---

## Test Plan

Test projects are **out of scope** for this story (`tests/` must not be modified). No tests are added, changed or removed by this plan. Existing tests in `0f87e2d` that cover this story (read-only references):

1. `tests/CustomerSupportCrm.Api.Tests/RateLimitingTests.cs` — `LowRateLimitApiFactory` (`Permits = 2`) and `LoginIsRateLimitedWithEnvelopeAndRetryAfter`: two 400s, then 429 with `errors[0].code == "RATE_LIMITED"` and a `Retry-After` header.
2. `tests/CustomerSupportCrm.Api.Tests/SecurityTests.cs` — `CorsAllowsConfiguredOriginWithCredentials` (preflight from `http://localhost:4200` gets `Access-Control-Allow-Origin` and `Access-Control-Allow-Credentials: true`) and `CorsIgnoresUnknownOrigin` (`https://evil.example` gets no CORS headers).
3. `tests/CustomerSupportCrm.Api.Tests/ApiFactory.cs` — raises `RateLimiting:Authentication:PermitLimit` to 1 000 so other API tests never hit the limit.

---

## Verification Steps

1. **Backend builds:** from `customer-support-crm-api/` run `dotnet build` — 0 warnings, 0 errors.
2. **Regression:** `dotnet test tests/CustomerSupportCrm.Api.Tests` — all pass (no DB needed).
3. **Rate limit:** with the API running (`dotnet run --project src/CustomerSupportCrm.Api --launch-profile https`):

```bash
for i in $(seq 1 11); do
  curl -sk -o /dev/null -w "%{http_code}\n" -X POST https://localhost:5001/api/v1/auth/login \
    -H "Content-Type: application/json" -d '{"email":"","password":""}'
done
curl -sk -i -X POST https://localhost:5001/api/v1/auth/login -H "Content-Type: application/json" -H "Accept-Language: ar" -d '{}'
```

   First 10 → `400`, then `429` with `Retry-After`, body `errors[0].code = "RATE_LIMITED"`, Arabic message, `correlationId` set.
4. **CORS allowed:** `curl -sk -i -X OPTIONS https://localhost:5001/api/v1/auth/login -H "Origin: http://localhost:4200" -H "Access-Control-Request-Method: POST" -H "Access-Control-Request-Headers: content-type,x-csrf-protection"` → `Access-Control-Allow-Origin: http://localhost:4200`, `Access-Control-Allow-Credentials: true`.
5. **CORS denied:** same request with `-H "Origin: https://evil.example"` → no `Access-Control-*` headers.
6. **Exposed headers:** a real `GET /health/live` with `Origin: http://localhost:4200` returns `Access-Control-Expose-Headers` containing `X-Correlation-Id`, `Content-Language`, `Retry-After`.
7. **Health not limited:** 20 quick `curl -sk https://localhost:5001/health/live` → all `200`.
8. **Swagger still works:** open `https://localhost:5001/swagger` in Development.

---

## Done Criteria

- [x] 11th login attempt within a minute from the same IP returns 429 `RATE_LIMITED` envelope with `Retry-After` (also refresh and change-password).
- [x] CORS allows only configured origins; preflight from an unknown origin gets no CORS headers.
- [x] Secure headers present on API responses; Swagger UI works in Development — completed in `2956767` (`SecureHeadersMiddleware`).
- [x] Server header removed; request body limit configurable — completed in `2956767` (`RequestLimits:MaxRequestBodyBytes`, default 25 MB).
- [x] Startup fails outside Development when `Cors:AllowedOrigins` is empty — completed in `2956767` (`Test` environment exempt).
- [x] Docs updated (`docs/architecture.md` pipeline + configuration table, `docs/security.md` CORS / sign-in protection).
- [x] `dotnet build` passes with zero warnings.
- [x] Nothing changed in `docker-compose.yml`, `deploy/`, `.github/` by this story; no test edits required by this plan.

**STOP HERE. Phase 2 backend stories are complete; report to the user.**
