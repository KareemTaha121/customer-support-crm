# Story intake

- Folder: `.squad/stories/security-and-administration/authentication-jwt-and-refresh-tokens/intake.md`

---

## Feature

- **Feature name (display):** Security & Administration — Phase 2 Identity & Authorization
- **Feature slug (folder under `plans/`):** `security-and-administration`

## Tracker (metadata only)

- **Tracker type:** `none`
- **Work item id:** `P2-02`
- **Work item type:** `Story`
- **Status:** `Done`
- **Assignee:** ``
- **Labels:** `phase-2`, `backend`, `identity`

---

## Title

```
Authentication: JWT access tokens and rotating refresh tokens
```

---

## Description

```
Implement login, refresh and logout in customer-support-crm-api using short-lived
JWT access tokens and rotated, revocable, hashed refresh tokens.

Configuration:
- JwtOptions (section "Jwt"): Issuer, Audience, SigningKey (>= 32 bytes,
  required), AccessTokenMinutes (default 15), RefreshTokenDays (default 7).
  Validated with ValidateOnStart. SigningKey empty in appsettings.json, set via
  user-secrets in Development.
- LockoutOptions (section "Identity:Lockout"): MaxFailedAttempts (5),
  LockoutMinutes (15).

Abstractions (Application/Abstractions/Authentication/):
- IPasswordHasher (Hash, Verify -> Success | Failed | SuccessRehashNeeded).
- ITokenService (CreateAccessToken(user, roles, permissions),
  CreateRefreshToken() -> raw + hash, HashRefreshToken(raw)).
Implementations in Infrastructure/Authentication using ASP.NET Core Identity
PasswordHasher<T> and System.IdentityModel.Tokens.Jwt / JsonWebTokenHandler.

Access-token claims: sub (user id), email, name, jti, role (one per role),
perm (one per permission), culture.

Feature slices (Application/Features/Authentication/), endpoints under
/api/v1/auth, all AllowAnonymous except logout:
- POST /auth/login {email, password} -> 200 {accessToken, accessTokenExpiresAt,
  refreshToken, refreshTokenExpiresAt, user {id, email, fullName, culture,
  roles, permissions}}.
  Wrong email, wrong password, or inactive user -> 401 INVALID_CREDENTIALS
  (same response, no user enumeration). Locked -> 401 ACCOUNT_LOCKED.
  Failed attempt increments counter and may lock; success resets it and sets
  LastLoginAt; rehash password when SuccessRehashNeeded.
- POST /auth/refresh {refreshToken} -> same response shape. Rotation: old token
  is revoked and linked to the new one (same FamilyId). Presenting an already
  revoked token revokes the whole family -> 401 REFRESH_TOKEN_REUSED. Expired /
  unknown -> 401 INVALID_REFRESH_TOKEN. Inactive user -> 401.
- POST /auth/logout {refreshToken} (authenticated) -> revokes that token's
  family; 204-style success envelope.

Api host (Api/Authentication/): AddAuthentication().AddJwtBearer with issuer,
audience, lifetime and signing-key validation, ClockSkew 30s, MapInboundClaims
false. UseAuthentication/UseAuthorization in the correct place in Program.cs
(after correlation/localization, before endpoints).

Tokens travel in the JSON body and the Authorization header only (no cookies),
so cookie CSRF protection is not required; note this in docs/architecture.md.
Contracts in Contracts/Authentication. Error codes are feature constants
(AuthErrorCodes) with en/ar messages in Messages.resx / Messages.ar.resx.
```

---

## Acceptance criteria

```
- [ ] Login returns access + refresh tokens and the user summary for valid credentials.
- [ ] Invalid credentials and inactive users get the identical 401 INVALID_CREDENTIALS response.
- [ ] Account locks after MaxFailedAttempts and unlocks after LockoutMinutes.
- [ ] Refresh rotates the token; the old one cannot be reused; reuse revokes the whole family.
- [ ] Only SHA-256 hashes of refresh tokens are stored.
- [ ] Logout revokes the refresh-token family.
- [ ] App fails to start when Jwt:SigningKey is missing or shorter than 32 bytes.
- [ ] Passwords and tokens never appear in logs (check Serilog request logging and exception handler).
- [ ] Endpoints appear in OpenAPI with request/response schemas.
- [ ] `dotnet build` passes with zero warnings.
```

---

## Attachments

None.

---

## Dependencies

- **Blocked by / related ids:** P2-01
- **Depends on code areas or other stories:** User / RefreshToken entities and seeder from P2-01; ApiResponse, GlobalExceptionHandler, Messages resources from Phase 1.

## Extra notes (optional)

- Implementation plan §21.1–21.2, §23.1, §24.
- Security events (login, failed login, lockout, logout) will be audited in P2-06; leave a clear seam (e.g. call sites in handlers) but do not implement audit here.

## Technical hints (optional)

- Repo root: `customer-support-crm-api/`. .NET 10, central package versions in `Directory.Packages.props` (add `Microsoft.AspNetCore.Authentication.JwtBearer`, `Microsoft.Extensions.Identity.Core`).
- Use TimeProvider for all time.

## Out of scope

- Do NOT touch Docker, docker-compose.yml, deploy/ or .github/ (CI/CD).
- Do NOT create or modify anything under tests/.
- ICurrentUser and permission policies (P2-03); rate limiting (P2-07).
- Password reset by email, SSO, MFA.
