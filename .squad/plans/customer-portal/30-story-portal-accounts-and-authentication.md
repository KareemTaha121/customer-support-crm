# Story 30 — Portal accounts and authentication (Story: CP-01)

> As-built plan: written after implementation in `customer-support-crm-api` commit `00d3f35` (feat: add customer portal). `PortalAccountSlices.cs`, `PortalContracts.cs` and `CustomerAccount.cs` have not changed since, so line numbers are the same at `00d3f35` and `develop` HEAD `2956767`; line numbers for other files refer to `2956767`.

## Prerequisites

- Security & Administration Phase 2 completed: [../security-and-administration/00-overview.md](../security-and-administration/00-overview.md) — `IPasswordHasher`, `CommonRules.ValidPassword`, `RateLimitPolicies.Authentication`, `IAuditTrail`.
- Platform (phase 3, `65c74a3`): customer token and policy infrastructure already exists — `ITokenService.CreateCustomerAccessToken`, `ICurrentCustomer` / `HttpCurrentCustomer`, `PolicyNames.Customer`, the `/api/v1/portal` and `/api/v1/public` route groups.
- Customer Management (01, `0fd694e`): `Customer`, `CustomerAccount` entity and `customer_accounts` mapping, `CustomerQueries`, `CustomerTimeline`.
- Communication Channels (03, `e921626`): `CustomerResolver`, `CustomerMessenger.QueueVerificationCodeAsync`.
- All paths below are relative to the **`customer-support-crm-api/`** repo root.

---

## Story Goal

1. A visitor registers with name, email, password (and optional phone/language); the account stays inactive until the emailed 6-digit code is verified, which signs them in.
2. A customer signs in with email + password and gets an 8-hour, non-refreshable customer access token.
3. A signed-in customer reads and updates their profile (name, language) and changes their password.
4. Staff with `customers.update` grant portal access to a known customer or revoke it.

Registration never reveals whether an email is known. Customer tokens carry no permissions, so they cannot call staff endpoints.

**Deviations from the intake (the code is authoritative):**

| Intake / feature spec | As built in `00d3f35` |
|---|---|
| Sign in with email **or phone OTP** | Email + password only; OTP is a 6-digit **email** code used once for verification. No phone/SMS sign-in |
| `Features/Portal/Auth/(Register, SignIn, VerifyOtp)` | `Features/CustomerPortal/PortalAccountSlices.cs` (one file): `PortalRegister`, `PortalVerify`, `PortalResendVerification`, `PortalLogin` commands; profile endpoints are inline lambdas, not MediatR slices |
| `CustomerUser` entity | `CustomerAccount` (Domain/Customers, from feature 01); one customer can have several accounts |
| Customer roles (company visibility) | Not built — no customer roles; token has `actor=customer`, `sub`, `cid` only |
| Toggle for self-registration | Not in `00d3f35`; `FeatureToggleBehavior` maps `PortalRegisterCommand` → `portal.registration_enabled` from `678ea67` |
| Change password uses the shared policy | Only a 12–128 length check inline (same bounds as `ValidPassword`); does not reject new == current |
| Profile update changes the portal name | `PUT /me` updates the **Customer** record name/language; the returned `name` is `CustomerAccount.DisplayName`, which is not changed |
| Revoking access ends sessions | Accounts are deactivated (`SetActive(false)`); issued tokens stay valid until expiry (≤ 8 h) — nothing checks the account per request |

**Not in scope:** refresh tokens for customers, social login, password reset by email. **Do not touch** `docker-compose.yml`, `deploy/`, `.github/` or anything under `tests/`.

---

## Context — Read These Files First

1. `src/CustomerSupportCrm.Domain/Customers/CustomerAccount.cs` — constants (13–17: 5 attempts, 15-min lockout, 24-h code), `CreateByStaff` (66–72), `Register` (75–81), `RestartVerification` (84–102, added by `00d3f35`), `GenerateVerificationCode` (104), `Verify` (106–124, fixed-time compare), `ChangePasswordHash` (126–130), `IsLockedOut` / `RecordFailedLogin` / `RecordSuccessfulLogin` (132–149), `SetActive` (151).
2. `src/CustomerSupportCrm.Application/Features/CustomerPortal/PortalAccountSlices.cs` — `PortalErrors` (26–33), `PortalSessions` (35–58), register (62–120), verify (122–139), resend (141–161), login (163–226), `PortalAuthEndpoints : IPublicEndpoint` (228–264), `PortalProfileEndpoints : IPortalEndpoint` (268–313), grant/revoke (317–366), `PortalAccessEndpoints : IEndpoint` (368–392).
3. `src/CustomerSupportCrm.Contracts/Portal/PortalContracts.cs` — auth/profile records (5–20), `GrantPortalAccessRequest` (53).
4. `src/CustomerSupportCrm.Infrastructure/Authentication/TokenService.cs` — `CreateCustomerAccessToken` (42–53); lifetime `JwtOptions.PortalTokenLifetimeHours = 8` (`JwtOptions.cs` 27, `appsettings.json` `Jwt:PortalTokenLifetimeHours`).
5. `src/CustomerSupportCrm.Application/Abstractions/Authentication/ICurrentCustomer.cs` (4–13); `src/CustomerSupportCrm.Infrastructure/Authentication/HttpCurrentCustomer.cs` (8–23); registered in `src/CustomerSupportCrm.Infrastructure/DependencyInjection.cs` line 192; claim `CrmClaimTypes.CustomerId = "cid"` (`CrmClaimTypes.cs` 17).
6. `src/CustomerSupportCrm.Infrastructure/Authorization/AuthorizationSetup.cs` — `PolicyNames.Customer` requires `actor=customer` (23–25); permission policies require staff (27–30).
7. `src/CustomerSupportCrm.Api/Endpoints/EndpointExtensions.cs` — `/api/v1/portal` with the customer policy (24–28), `/api/v1/public` anonymous (30–34).
8. `src/CustomerSupportCrm.Infrastructure/Persistence/Configurations/TicketConfiguration.cs` — `CustomerAccountConfiguration` (96–111: unique email 104, FK cascade 109, `xmin` 110). Table created by `AddSupportOperations` (`678ea67`).
9. `src/CustomerSupportCrm.Application/Features/Channels/InboundChannels.cs` — `CustomerResolver` (37); `src/CustomerSupportCrm.Application/Features/Channels/CustomerMessaging.cs` — `QueueVerificationCodeAsync` (114–120, queues a branded email `OutboundMessage`).
10. `src/CustomerSupportCrm.Application/Features/Settings/SettingsSlices.cs` — `EnsureEnabledAsync` (49–55, `FEATURE_DISABLED` 409), `FeatureToggleBehavior` (135–159, `PortalRegisterCommand` entry 141).
11. `src/CustomerSupportCrm.Application/Features/Customers/Common/CustomerQueries.cs` — `LoadAsync` (27–34) and `EnsureAccessibleAsync` (36–) enforce the staff access scope; `CustomerTimeline` (177).

---

## Backend Tasks

### 1 — Domain

File: `src/CustomerSupportCrm.Domain/Customers/CustomerAccount.cs` — add `RestartVerification(codeHash, now, passwordHash?, displayName?)`: throws `DomainException(INVALID_ACCOUNT)` when already verified; resets the code and its 24-h expiry, optionally replaces password hash and name. The rest of the entity predates this story (feature 01).

### 2 — Contracts

Create file: `src/CustomerSupportCrm.Contracts/Portal/PortalContracts.cs`: `PortalRegisterRequest(Name, Email, Password, Phone?, Language?)`, `PortalVerifyRequest(Email, Code)`, `PortalEmailRequest(Email)`, `PortalLoginRequest(Email, Password)`, `PortalProfileResponse(AccountId, CustomerId, CustomerNumber, Name, Email, Language)`, `PortalSessionResponse(AccessToken, ExpiresAt, Profile)`, `PortalUpdateProfileRequest(Name, Language)`, `PortalChangePasswordRequest(CurrentPassword, NewPassword)`, `GrantPortalAccessRequest(Email, Password)`. No password hash or code in any response.

### 3 — Shared helpers

`PortalErrors`: `INVALID_CREDENTIALS`, `ACCOUNT_LOCKED`, `EMAIL_NOT_VERIFIED`, `INVALID_VERIFICATION_CODE`, `PORTAL_ACCOUNT_EXISTS`. `PortalSessions.HashCode` (SHA-256 hex of the trimmed code), `ProfileAsync` (projection with customer number and preferred language; missing → `UnauthorizedException`), `IssueAsync` (token + profile).

### 4 — Register / verify / resend

- `PortalRegisterValidator` (64–74): name ≤ `Customer.NameMaxLength`, email, `ValidPassword()`, phone ≤ 32, language `en`/`ar`/null.
- `PortalRegisterHandler` (80–120): existing **unverified** account → `RestartVerification` with the new password/name and a new code; existing verified → return silently; new → `CustomerResolver.ResolveAsync(ContactType.Email, …)`, optional phone contact (`PhoneNumber.TryCreate`), `CustomerAccount.Register`, queue the code email, save. The endpoint always returns `ApiResults.Success("If the address can be registered, a verification code has been sent.")`.
- `PortalVerifyHandler` (124–139): unknown email or failed `Verify` → `ValidationException` field `Code`, code `INVALID_VERIFICATION_CODE`; success activates the account, records a login and returns a session.
- `PortalResendVerificationHandler` (143–161): only for unverified accounts; otherwise silent 200.

### 5 — Login

`PortalLoginHandler` (174–226): unknown email → verify against a cached dummy hash (timing equalizer) then 401 `INVALID_CREDENTIALS`; locked → 401 `ACCOUNT_LOCKED`; wrong password → `RecordFailedLogin` + save + 401 `INVALID_CREDENTIALS`; unverified → 401 `EMAIL_NOT_VERIFIED`; inactive → 401 `INVALID_CREDENTIALS`; `SuccessRehashNeeded` → rehash; success → `RecordSuccessfulLogin`, timeline `CustomerActivityTypes.PortalSignIn`, save, session.

All four anonymous routes: group `/portal` under `/api/v1/public`, `.RequireRateLimiting(RateLimitPolicies.Authentication)`, names `PortalRegister`, `PortalVerify`, `PortalResendVerification`, `PortalLogin`.

### 6 — Profile

`PortalProfileEndpoints` (268–313) under `/api/v1/portal`:

- `GET /me` → `ProfileAsync(customer.AccountId)`.
- `PUT /me` → inline check (name 1–`NameMaxLength`, language `en`/`ar`, else `INVALID` on `Name`), `Customer.UpdateProfile(type, name, companyName, language, tags)`, save, return profile.
- `POST /me/change-password` → new password length 12–128 (`INVALID_LENGTH`), wrong current → 400 `INVALID_CURRENT_PASSWORD`, `ChangePasswordHash`; rate-limited.

### 7 — Staff grant / revoke

- `GrantPortalAccessCommand(CustomerId, Email, Password)` + validator (`ValidPassword`) + handler (328–349): `CustomerQueries.LoadAsync` with the caller's scope (404 outside scope); any account with that email → 409 `PORTAL_ACCOUNT_EXISTS`; `CustomerAccount.CreateByStaff` (active, verified) using the **customer's name**; adds the email as a contact when missing; audit `customers.portal_access_granted`.
- `RevokePortalAccessCommand(CustomerId)` (351–366): scope check, `SetActive(false)` on every account of the customer, audit `customers.portal_access_revoked`.
- Routes `POST` / `DELETE /api/v1/customers/{id:guid}/portal-access`, `customers.update`, tag "Customers".

### 8 — Localization and docs

`Messages.resx` / `Messages.ar.resx`: `INVALID_CREDENTIALS` (51), `ACCOUNT_LOCKED` (54), `INVALID_CURRENT_PASSWORD` (69), `FEATURE_DISABLED` (114) shared with staff auth/settings; `EMAIL_NOT_VERIFIED` (en 222 / ar 294), `PORTAL_ACCOUNT_EXISTS` (225 / 297), `INVALID_VERIFICATION_CODE` (228 / 300), `INVALID_ACCOUNT` (ar 303). Portal keys were added in `0936711`. `docs/endpoints.md`: Customers table line 89, Customer portal (182–194), Public (201–202).

No DI changes: handlers, validators and the three endpoint classes are discovered by assembly scanning.

---

## Edge Cases & Failure Modes

- **Account enumeration** — register and resend always return 200 with the same body; login returns the same `INVALID_CREDENTIALS` for unknown email, wrong password and inactive accounts. `EMAIL_NOT_VERIFIED` is only returned after a correct password.
- **Taking over a customer record** — a self-registered account is inactive until the code sent to that address is entered.
- **Re-register an unverified email** — replaces password and name, sends a new code; old code stops working.
- **Expired / wrong code** — 400 `INVALID_VERIFICATION_CODE`; no attempt counter on codes (rate limit only, 10/60 s per IP).
- **Lockout** — 5 wrong passwords → 15 min; while locked even the right password → 401 `ACCOUNT_LOCKED`.
- **Registration disabled** — `portal.registration_enabled = false` → 409 `FEATURE_DISABLED` (`678ea67`).
- **Revoked customer** — cannot sign in (inactive → `INVALID_CREDENTIALS`), but an existing token keeps working until it expires.
- **Staff token on `/api/v1/portal`** — rejected by the customer policy (403); customer token on staff routes → 403 (no `actor=staff`).
- **Duplicate email race** — unique index on `customer_accounts.email` → 409 `CONFLICT`.

---

## Test Plan

Test projects are **out of scope** (`tests/` must not be modified). No tests are added, changed or removed by this story. There are **no existing tests** in `2956767` that cover the portal (no file under `tests/` references `portal` or `CustomerAccount`).

---

## Verification Steps

1. **Backend builds:** in `customer-support-crm-api/` run `dotnet build` — 0 warnings, 0 errors.
2. **Run:** apply migrations, start the API (the verification email lands in `outbound_messages`; read the code from the queued message or the dev mail sink).
3. **Register:** `curl -X POST https://localhost:<port>/api/v1/public/portal/register -H "Content-Type: application/json" -d '{"name":"Sara","email":"sara@example.com","password":"<12+ chars>","language":"ar"}'` → 200; repeat with the same email → identical 200.
4. **Login before verify:** `POST /api/v1/public/portal/login` → 401 `EMAIL_NOT_VERIFIED`.
5. **Verify:** `POST /api/v1/public/portal/verify -d '{"email":"sara@example.com","code":"000000"}'` → 400 `INVALID_VERIFICATION_CODE`; with the real code → 200 `{accessToken, expiresAt, profile}`; export `PTOKEN`.
6. **Profile:** `curl https://localhost:<port>/api/v1/portal/me -H "Authorization: Bearer $PTOKEN"` → profile with `customerNumber`; `PUT /me -d '{"name":"Sara A","language":"en"}'` → 200.
7. **Change password:** wrong current → 400 `INVALID_CURRENT_PASSWORD`; 8-char new → 400 `INVALID_LENGTH`; valid → 200.
8. **Lockout:** 5 wrong passwords → then 401 `ACCOUNT_LOCKED`.
9. **Staff grant:** with a staff `TOKEN`, `POST /api/v1/customers/{id}/portal-access -d '{"email":"c@example.com","password":"<12+ chars>"}'` → 200; again → 409 `PORTAL_ACCOUNT_EXISTS`; `DELETE` → 200, then that customer's login → 401 `INVALID_CREDENTIALS`.
10. **Isolation:** `$PTOKEN` on `GET /api/v1/tickets` → 403; staff `TOKEN` on `GET /api/v1/portal/me` → 403.
11. **Toggle:** set `portal.registration_enabled` to `false` (Settings) → register returns 409 `FEATURE_DISABLED`.

---

## Done Criteria

- [x] Customers self-register with an emailed verification code and sign in with email + password; staff can grant/revoke access.
- [ ] Phone OTP sign-in — not built (email code for verification only).
- [x] Separate customer authorization policy; customer tokens carry no permissions and are refused by staff routes.
- [x] Portal identity (`CustomerAccount`) linked to `Customer`; registration resolves the customer by email.
- [x] No account enumeration through register/resend/login responses; lockout after 5 failures.
- [ ] Revoking access ends existing sessions — not built; tokens live until expiry (≤ 8 h).
- [x] Error codes localized in en/ar.
- [x] Nothing changed in `tests/`, `docker-compose.yml`, `deploy/` or `.github/` by this story.
- [x] `dotnet build` passes with zero warnings.

**STOP HERE. Report to the user and wait for confirmation before proceeding to Story 31.**
