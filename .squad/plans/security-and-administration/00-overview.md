# Security & Administration — Phase 2 Identity & Authorization

Feature spec: [../../features/10-security-and-administration.md](../../features/10-security-and-administration.md) (Phase 2 section).
Repo: `customer-support-crm-api`. Out of scope for every story: Docker, deployment, CI/CD, and files under `tests/`.

| NN | Tracker id | Plan file | Title | Depends on | Status |
|----|-----------|-----------|-------|-----------|--------|
| 01 | P2-01 | [01-story-identity-domain-persistence.md](01-story-identity-domain-persistence.md) | Identity domain & persistence | Phase 1 | Done |
| 02 | P2-02 | [02-story-authentication-jwt-and-refresh-tokens.md](02-story-authentication-jwt-and-refresh-tokens.md) | Authentication: JWT + rotating refresh tokens | 01 | Done |
| 03 | P2-03 | [03-story-current-user-and-permission-authorization.md](03-story-current-user-and-permission-authorization.md) | Current user & permission authorization | 01, 02 | Done |
| 04 | P2-04 | [04-story-user-management.md](04-story-user-management.md) | User management | 01–03 | Done |
| 05 | P2-05 | [05-story-role-and-permission-management.md](05-story-role-and-permission-management.md) | Role & permission management | 01, 03 | Done |
| 06 | P2-06 | [06-story-audit-logging.md](06-story-audit-logging.md) | Audit logging | 01–05 | Done |
| 07 | P2-07 | [07-story-security-hardening.md](07-story-security-hardening.md) | Security hardening | 02, 03 | Partial |

Plans are generated one at a time, after the previous story is implemented, so each plan cites the real code the earlier stories produced.

Note: `IPasswordHasher` moved from P2-02 into Story 01 because the bootstrap-admin seeder needs it.

All seven stories were implemented in `customer-support-crm-api` commit `0f87e2d` (feat: add identity, roles, permissions and audit logging). Plan 01 was written before implementation and later revised; plans 01–07 are all **as-built** plans, and their paths and line numbers refer to `0f87e2d`. Each one lists where the code differs from its intake.

Known gaps and drift:

- **Story 07 is partial.** `0f87e2d` has auth rate limiting and strict CORS, but no secure-headers middleware (nosniff, X-Frame-Options, Referrer-Policy, CSP), no `UseHsts`, no Server-header removal, no configurable `MaxRequestBodySize`, and no startup failure when `Cors:AllowedOrigins` is empty outside Development.
- **`GET /auth/me` does not check `IsActive`.** A disabled user keeps getting 200 until the access token expires (see plan 03, Edge Cases).
- **Dev bootstrap credentials are committed** in `appsettings.Development.json` (`Bootstrap` section, `InitializeOnStartup: true`); the intake asked for user-secrets (see plan 01, Deviations).
