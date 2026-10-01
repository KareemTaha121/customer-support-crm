# Story 41 — Revoking portal access ends the customer's portal sessions (Bug: BUG-05)

> Fix plan, implemented in `customer-support-crm-api` commits `31d6d5a` (token check) and `6587e7d` (re-grant) on `develop`. Paths and line numbers refer to `6587e7d`.
> Intake: [../../stories/customer-portal/revoke-portal-sessions-on-access-revoke/intake.md](../../stories/customer-portal/revoke-portal-sessions-on-access-revoke/intake.md)

## Prerequisites

- Story 30 — [30-story-portal-accounts-and-authentication.md](30-story-portal-accounts-and-authentication.md): `CustomerAccount`, portal sign-in, `GrantPortalAccessHandler`, `RevokePortalAccessHandler`.
- Story 21 — [../platform/21-story-organization-context-and-notifications.md](../platform/21-story-organization-context-and-notifications.md): the `/api/v1/portal` route group with the `Customer` policy.
- All paths are relative to the **`customer-support-crm-api/`** repo root.

---

## Story Goal

Before the fix, `DELETE /customers/{id}/portal-access` called `SetActive(false)` on the customer's accounts. Customer tokens are 8-hour JWTs with no refresh, and nothing on `/portal/*` checked the account. So a revoked customer kept full portal access until the token expired.

After the fix:

1. **Every** `/api/v1/portal/*` request checks that the token's account is still active. If not, it returns **401 `ACCOUNT_DISABLED`**, the same code the staff `GET /auth/me` uses for disabled users. The web app already signs the customer out on any portal 401 (`core/interceptors/auth.interceptor.ts` lines 26–33).
2. **Re-granting access works.** This was found while fixing. Before, `POST /customers/{id}/portal-access` with the same email returned 409 `PORTAL_ACCOUNT_EXISTS`, because the revoked account still held the unique email. Now a revoked account **of the same customer** is reactivated with the new password. The intake's second criterion needed this.

**Deviations from the intake:**

- The intake listed two options: a per-request status check, or a security stamp. The per-request check was built: one primary-key lookup per portal request, and no token or schema change.
- Re-grant reactivation (commit `6587e7d`) was not described in the intake's fix text. It was needed to meet the intake's acceptance criteria.

---

## Context — Read These Files First

1. `src/CustomerSupportCrm.Api/Endpoints/EndpointExtensions.cs` lines 25–32: the portal group, now `.RequireAuthorization(PolicyNames.Customer).AddEndpointFilter<ActivePortalAccountFilter>()`.
2. `src/CustomerSupportCrm.Application/Features/CustomerPortal/ActivePortalAccountFilter.cs` (new, lines 16–36).
3. `src/CustomerSupportCrm.Infrastructure/Authentication/HttpCurrentCustomer.cs`: `ICurrentCustomer.AccountId` comes from the token's `sub`.
4. `src/CustomerSupportCrm.Application/Features/Authentication/Common/AuthenticationErrors.cs` line 7: `AccountDisabled = "ACCOUNT_DISABLED"`. en/ar resx entries already exist.
5. `src/CustomerSupportCrm.Domain/Customers/CustomerAccount.cs`: `SetActive` (151) and the new `ReactivateByStaff` (157–166).
6. `src/CustomerSupportCrm.Application/Features/CustomerPortal/PortalAccountSlices.cs`: `GrantPortalAccessHandler` (332–363), `RevokePortalAccessHandler` (367–380).
7. `src/CustomerSupportCrm.Infrastructure/Persistence/Configurations/TicketConfiguration.cs` line 108: `customer_accounts.email` is unique.

---

## Backend Tasks

### 1 — Active-account filter (`31d6d5a`)

Create `Features/CustomerPortal/ActivePortalAccountFilter.cs`, a `public sealed class ActivePortalAccountFilter : IEndpointFilter`.

- Resolve `ICurrentCustomer` and `IApplicationDbContext` from `context.HttpContext.RequestServices` **inside** `InvokeAsync`. The filter instance is created once per endpoint, so it must not take scoped services through its constructor.
- Run `db.CustomerAccounts.AsNoTracking().AnyAsync(a => a.Id == accountId && a.IsActive, RequestAborted)`. When this is false, throw `UnauthorizedException(AuthenticationErrors.AccountDisabled, "The account is disabled.")`. `GlobalExceptionHandler` maps it to 401 with the envelope and the localized message.
- Otherwise call `next(context)`.

Register the filter on the portal group in `EndpointExtensions.MapApiEndpoints` (and add `using CustomerSupportCrm.Application.Features.CustomerPortal;`). Anonymous portal routes (`register`, `verify`, `resend-verification`, `login`) live under `/api/v1/public/portal` and are unaffected.

### 2 — Re-grant a revoked account (`6587e7d`)

In `CustomerAccount`:

```csharp
public void ReactivateByStaff(string passwordHash)
```

It sets the new hash, `EmailVerified = true`, `IsActive = true`, clears the verification code and expiry, and resets failed attempts and lockout. This matches `CreateByStaff`, where staff vouch for the email.

In `GrantPortalAccessHandler`, load `existing` by email:

- `existing is { IsActive: false }` **and** `existing.CustomerId == customer.Id` → `existing.ReactivateByStaff(hasher.Hash(password))`.
- Any other existing account (active, or another customer's) → 409 `PORTAL_ACCOUNT_EXISTS`, as before.
- No account → `CreateByStaff`, as before.

The email-contact and audit (`customers.portal_access_granted`) steps are unchanged.

### 3 — Docs

In `docs/endpoints.md` line 10, the `/api/v1/portal` row now says: every request checks the account is still active (401 `ACCOUNT_DISABLED` after revoke).

**No changes to:** contracts, migrations, resx (existing code), or the frontend.

---

## Test Plan

Test projects are **out of scope**: `tests/` must not be modified. No existing test covers the portal.

---

## Verification Steps

1. **Build:** in `customer-support-crm-api/` run `dotnet build`. At `6587e7d`: 0 warnings, 0 errors.
2. **Revoke:**
   - The customer signs in and exports `CTOKEN`; `GET /api/v1/portal/me` → 200.
   - Staff call `DELETE /api/v1/customers/{id}/portal-access`.
   - `GET /portal/me` with the same `CTOKEN` → **401 `ACCOUNT_DISABLED`**. With `Accept-Language: ar` the message is in Arabic.
   - In the web app, the next portal action sends the customer to `/portal/login`.
3. **Sign-in blocked:** `POST /api/v1/public/portal/login` → 401 `INVALID_CREDENTIALS` (existing rule for inactive accounts, `PortalAccountSlices.cs` 209–212).
4. **Re-grant:**
   - `POST /api/v1/customers/{id}/portal-access` with the same email and a new password → 200, not 409.
   - The customer signs in with the new password, and `/portal/me` → 200.
5. **Conflicts:** granting an email used by another customer's account, or by an active account, → 409 `PORTAL_ACCOUNT_EXISTS`.

---

## Done Criteria

- [x] After revoke, the customer's existing token gets 401 on `/portal/*` immediately.
- [x] Re-granting access lets the customer sign in again.
- [x] Staff users and anonymous `/public/portal` routes are unaffected.
- [x] `docs/endpoints.md` is updated.
- [x] Nothing changed in `tests/`, `docker-compose.yml`, `deploy/` or `.github/`.
- [x] `dotnet build` passes with zero warnings.

**STOP HERE. Report to the user and wait for confirmation before proceeding to the next bug story.**
