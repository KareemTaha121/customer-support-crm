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
| 07 | P2-07 | [07-story-security-hardening.md](07-story-security-hardening.md) | Security hardening | 02, 03 | Done |
| 22 | P3-01 | [22-story-system-settings-and-audit-export.md](22-story-system-settings-and-audit-export.md) | System settings and audit log export (Phase 3) | 03, 06, platform 21 | Done |
| 47 | BUG-11 | [47-story-password-reset.md](47-story-password-reset.md) | Staff and portal password reset by email (QA H1) | 49, 03, 06 | Done (api `ba569b9`, web `5ab9998`) |
| 54 | BUG-18 | [54-story-readable-audit-log.md](54-story-readable-audit-log.md) | Readable audit log: entity labels, translated actions and entity types (QA M5) | 06, 22, 47 | To do |
| 59 | BUG-23 | [59-story-user-data-access-warning.md](59-story-user-data-access-warning.md) | Warn when a staff user would see no data; `hasDataAccess` on the current user (QA round 2, N3) | 03, platform 21 | Done (api `613e607`, web `6652669`) |

Plans are generated one at a time, after the previous story is implemented, so each plan cites the real code the earlier stories produced.

Note: `IPasswordHasher` moved from P2-02 into Story 01 because the bootstrap-admin seeder needs it.

All seven stories were implemented in `customer-support-crm-api` commit `0f87e2d` (feat: add identity, roles, permissions and audit logging). Plan 01 was written before implementation and later revised; plans 01–07 are all **as-built** plans, and their paths and line numbers refer to `0f87e2d`. Each one lists where the code differs from its intake.

Known gaps and drift:

- **Story 07 was partial in `0f87e2d`.** Secure headers, HSTS, Server-header removal, the configurable body limit, the CORS startup check and the global rate limiter were added in `2956767`.
- **`GET /auth/me` now refuses disabled accounts** (401 `ACCOUNT_DISABLED`, `2956767`). Other endpoints still accept an already-issued access token until it expires (≤ 15 minutes), as documented in `docs/security.md`.
- **Dev bootstrap credentials are committed** in `appsettings.Development.json` (`Bootstrap` section, with `Database:InitializeOnStartup: true`); the intake asked for user-secrets (see plan 01, Deviations).

Phase 3 addition (Story 22): **system settings and audit export** shipped in `678ea67`, in the same commit as integrations (a separate story). Plan 22 is as-built against `678ea67`. Its gaps:

- `GET /settings` needs only a staff token; only `PUT /settings` requires `settings.manage`. The update runs inline in the endpoint (no command or validator), and the audit entry records only the new value.
- `GET /audit-logs/export.csv` filters only by `from`/`to`/`action`, caps at 100,000 rows, and is not itself audited.
- No tests cover settings, feature toggles or the export.
- Branches, departments, user scopes and branding (feature 10's Phase 3 part) are planned in [../platform/21-story-organization-context-and-notifications.md](../platform/21-story-organization-context-and-notifications.md).
