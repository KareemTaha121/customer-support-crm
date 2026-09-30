# Story intake

- Folder: `.squad/stories/security-and-administration/security-hardening/intake.md`

---

## Feature

- **Feature name (display):** Security & Administration — Phase 2 Identity & Authorization
- **Feature slug (folder under `plans/`):** `security-and-administration`

## Tracker (metadata only)

- **Tracker type:** `none`
- **Work item id:** `P2-07`
- **Work item type:** `Story`
- **Status:** `Done`
- **Assignee:** ``
- **Labels:** `phase-2`, `backend`, `security`

---

## Title

```
Security hardening: rate limiting, CORS, secure headers, request limits
```

---

## Description

```
Harden the API host (customer-support-crm-api, Api project) per implementation
plan §23.

Rate limiting (ASP.NET Core built-in RateLimiter):
- RateLimitOptions (section "RateLimiting"), validated on start.
- Policy "auth": fixed window per client IP, default 10 requests / minute,
  applied to POST /auth/login and POST /auth/refresh.
- Global limiter: per authenticated user id (fallback IP), default
  300 requests / minute; health endpoints excluded.
- Rejections return 429 with the standard envelope (RATE_LIMITED, localized
  message, correlationId) and a Retry-After header.

CORS:
- CorsOptions (section "Cors"): AllowedOrigins[] (required outside
  Development). Development default http://localhost:4200.
- Named policy with explicit origins only, any method, headers Authorization,
  Content-Type, Accept-Language, X-Correlation-Id; exposed headers
  X-Correlation-Id, Content-Language, Retry-After. No AllowAnyOrigin, no
  credentials (tokens are bearer, not cookies).

Secure headers middleware:
- X-Content-Type-Options: nosniff, X-Frame-Options: DENY,
  Referrer-Policy: no-referrer, Content-Security-Policy for API responses
  (default-src 'none'; frame-ancestors 'none') except the Swagger UI path in
  Development. HSTS (UseHsts) outside Development.
- Remove the Server header (Kestrel AddServerHeader = false).

Request limits:
- Kestrel MaxRequestBodySize from config (default 10 MB); oversized requests
  already map to 413 PAYLOAD_TOO_LARGE — verify the envelope.

Pipeline order in Program.cs must stay correct: correlation -> localization ->
logging -> exception handler -> status code pages -> HSTS/HTTPS -> secure
headers -> CORS -> authentication -> authorization -> rate limiter -> endpoints.
Update docs/architecture.md request-pipeline section and docs/development.md
configuration notes.
```

---

## Acceptance criteria

```
- [ ] 11th login attempt within a minute from the same IP returns 429 RATE_LIMITED envelope with Retry-After.
- [ ] CORS allows only configured origins; preflight from an unknown origin gets no CORS headers.
- [ ] Secure headers present on API responses; Swagger UI still works in Development.
- [ ] Server header removed; request body limit configurable.
- [ ] Startup fails outside Development when Cors:AllowedOrigins is empty.
- [ ] Docs updated.
- [ ] `dotnet build` passes with zero warnings.
```

---

## Attachments

None.

---

## Dependencies

- **Blocked by / related ids:** P2-02, P2-03
- **Depends on code areas or other stories:** Program.cs pipeline, ErrorResponseWriter, auth endpoints.

## Extra notes (optional)

- Implementation plan §23 (security strategy, CORS, rate limiting).

## Technical hints (optional)

- Repo root: `customer-support-crm-api/`. .NET 10.

## Out of scope

- Do NOT touch Docker, docker-compose.yml, deploy/ or .github/ (CI/CD).
- Do NOT create or modify anything under tests/.
- Dependency / container image scanning (CI concern).
- Rate limits for portal, AI, uploads (added with those features).
