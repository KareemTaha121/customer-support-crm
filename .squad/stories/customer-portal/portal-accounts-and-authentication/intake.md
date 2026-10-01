# Story intake

- Folder: `.squad/stories/customer-portal/portal-accounts-and-authentication/intake.md`

---

## Feature

- **Feature name (display):** Customer Portal
- **Feature slug (folder under `plans/`):** `customer-portal`

## Tracker (metadata only)

- **Tracker type:** `none`
- **Work item id:** `CP-01`
- **Work item type:** `Story`
- **Status:** `Done`
- **Assignee:** ``
- **Labels:** `backend`, `customer-portal`

---

## Title

```
Portal accounts and authentication
```

---

## Description

```
Customers get a portal login (CustomerAccount, linked to a Customer record) either by
self-registration with an emailed 6-digit verification code or by staff granting access.
Signed-in customers receive a customer access token (JWT, 8 h, not refreshable) and manage
their profile and password. Slices in Application/Features/CustomerPortal/PortalAccountSlices.cs;
contracts in Contracts/Portal/PortalContracts.cs; domain Domain/Customers/CustomerAccount.cs.

Anonymous endpoints (/api/v1/public, "authentication" rate limit):
- POST /portal/register             {name, email, password, phone?, language?}  Always 200 with the same
                                    message (no account enumeration); behind portal.registration_enabled
- POST /portal/verify               {email, code}  -> PortalSessionResponse {accessToken, expiresAt, profile}
- POST /portal/resend-verification  {email}  Always 200
- POST /portal/login                {email, password} -> PortalSessionResponse

Customer endpoints (/api/v1/portal, customer policy):
- GET  /me                          PortalProfileResponse {accountId, customerId, customerNumber, name, email, language}
- PUT  /me                          {name, language (en|ar)}
- POST /me/change-password          {currentPassword, newPassword}  ("authentication" rate limit)

Staff endpoints (/api/v1):
- POST   /customers/{id}/portal-access   customers.update  {email, password}  Grant (active + verified)
- DELETE /customers/{id}/portal-access   customers.update  Revoke (deactivates all the customer's accounts)

Rules:
- Registration resolves or creates the Customer by email (CustomerResolver), adds the phone as a
  contact, stores SHA-256 of the code; code valid 24 h. Re-registering an unverified email
  restarts verification with the new password/name; a verified email is silently ignored.
- Wrong or expired code -> 400 field error code / INVALID_VERIFICATION_CODE.
- Login: unknown email or wrong password -> 401 INVALID_CREDENTIALS (constant-time equalizer hash);
  5 failures -> 15-minute lock -> 401 ACCOUNT_LOCKED; unverified -> 401 EMAIL_NOT_VERIFIED;
  inactive -> 401 INVALID_CREDENTIALS. Successful login records a portal.sign_in timeline entry.
- Customer tokens carry actor=customer, sub=account id, cid=customer id; no permission claims, so
  staff policies never accept them.
- Password policy: CommonRules.ValidPassword (12-128) on register/grant; change-password checks length
  only; wrong current password -> 400 INVALID_CURRENT_PASSWORD.
- Grant: email already used by any portal account -> 409 PORTAL_ACCOUNT_EXISTS; customer outside
  the staff member's scope -> 404. Grant/revoke are audited.
- Registration disabled -> 409 FEATURE_DISABLED.
- Messages localized (en/ar).
```

---

## Acceptance criteria

```
- [ ] As a customer, I register / sign in (email or phone OTP) to the portal.
- [ ] Reuses the same backend domain and API contracts; separate customer authorization policies.
- [ ] Portal identity (CustomerUser) linked to Customer.
- [ ] Agent-only data is never exposed through portal tokens.
- [ ] AR/EN messages.
- [ ] `dotnet build` passes with zero warnings.
```

---

## Attachments

None.

---

## Dependencies

- **Blocked by / related ids:** P2-01 – P2-07 (identity, tokens, password hasher, rate limiting), Customer Management (01: `Customer`, `CustomerAccount`, `CustomerTimeline`), Communication Channels (03: `CustomerMessenger`, `CustomerResolver`).
- **Depends on code areas or other stories:** `ITokenService.CreateCustomerAccessToken`, `ICurrentCustomer`, `PolicyNames.Customer`, `IPasswordHasher`, `FeatureToggleBehavior` (Settings), `IAccessScopeProvider`.

## Extra notes (optional)

- Feature spec: `.squad/features/08-customer-portal.md` (Auth slices).
- Frontend: `.squad/plans/frontend/09-story-authentication-staff-and-portal.md` (FE-02) and `17-story-customer-portal-ui.md` (FE-10).

## Technical hints (optional)

- Repo root: `customer-support-crm-api/`. .NET 10.
- Implemented in commit `00d3f35`; the registration toggle was wired later in `678ea67`.

## Out of scope

- Do NOT touch Docker, docker-compose.yml, deploy/ or .github/ (CI/CD).
- Do NOT create or modify anything under tests/.
- Phone/SMS OTP sign-in, social login, refresh tokens for customers, company-level customer roles.
