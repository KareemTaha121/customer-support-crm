# Story 09 — Authentication: staff and portal (Story: FE-02)

## Prerequisites

- Story 08 completed: [08-story-core-platform-shell.md](08-story-core-platform-shell.md) (`AuthService`, `PortalAuthService`, guards, i18n, shared form helpers).

---

## Story Goal

1. Staff sign in at `/login`, stay signed in across reloads (refresh cookie) and sign out from the user menu.
2. Lockout, disabled account and wrong credentials show distinct localized messages.
3. Staff change their password at `/profile`.
4. Customers sign in, self-register (when `portal.registration_enabled` is on), verify their email with a code and edit their profile under `/portal`, in a separate portal layout and session.

Not in scope: portal tickets, chat widget, web form (Story 17).

---

## Context — Read These Files First

1. `customer-support-crm-api/src/CustomerSupportCrm.Application/Features/Authentication/Login/LoginEndpoint.cs`, `Refresh/RefreshSessionEndpoint.cs`, `Logout/LogoutEndpoint.cs`, `ChangePassword/ChangePasswordEndpoint.cs` — routes under `/api/v1/auth`.
2. `customer-support-crm-api/src/CustomerSupportCrm.Application/Features/Authentication/Common/AuthenticationErrors.cs` — `INVALID_CREDENTIALS`, `ACCOUNT_LOCKED`, `ACCOUNT_DISABLED`, `INVALID_CURRENT_PASSWORD`.
3. `customer-support-crm-api/src/CustomerSupportCrm.Application/Features/CustomerPortal/PortalAccountSlices.cs` (~lines 230–300) — `/public/portal/register|verify|resend-verification|login`, `/portal/me`, `/portal/me/change-password`.
4. `customer-support-crm-api/src/CustomerSupportCrm.Contracts/Portal/PortalContracts.cs` — `PortalSessionResponse`, `PortalProfileResponse`, `PortalRegisterRequest`.
5. `customer-support-crm-api/src/CustomerSupportCrm.Application/Common/Validation/CommonRules.cs` — password 12–128 characters.
6. `customer-support-crm-web/src/app/core/auth/auth.service.ts` and `portal-auth.service.ts` — the session APIs these pages call.

---

## Frontend Tasks

### 1 — Staff pages (`src/app/features/auth/`)

- Create `login.page.ts` + `login.page.html` + `auth-pages.scss`: reactive form (email, password), show/hide password, progress bar, error banner mapped from the codes above, `returnUrl` support (same-origin paths only), link to the portal.
- Create `profile.page.ts`: account details (name, email, roles, permission count) and change-password form (`matchFields`, `passwordValidators` from `shared/password.ts`); `INVALID_CURRENT_PASSWORD` sets a field error.
- File: `auth.routes.ts` — `LOGIN_ROUTES` (`guestGuard`, `translationResolver('auth')`), `PROFILE_ROUTES`.
- Create `public/i18n/auth/en.json` and `ar.json`.

### 2 — Portal layout and pages (`src/app/features/customer-portal/`)

- Create `portal-shell.component.ts` (brand bar, `portal-navigation.ts` items, language switch, account menu or sign-in button).
- Create `auth/portal-login.page.ts`, `auth/portal-register.page.ts` (feature-flagged), `auth/portal-verify.page.ts` (prefills `?email=`, resend code), `auth/portal-profile.page.ts` (name, language — also switches the UI language — and password), shared `auth/portal-auth.scss`.
- File: `customer-portal.routes.ts` — `PORTAL_ROUTES` with the shell, `translationResolver('portal')`, `login`/`register` (`portalGuestGuard`), `verify`, `profile` (`portalAuthGuard`), a `tickets` placeholder for Story 17.
- Create `public/i18n/portal/en.json` and `ar.json` (`nav`, `fields`, `actions`, `auth`, `profile`).

---

## Edge Cases & Failure Modes

- `returnUrl` pointing off-site (`//evil`) → ignored, go to `/dashboard` (`login.page.ts` `submit`).
- Rate limit (429) on login → generic localized message via `describeError`.
- Registration disabled → the register page shows an info banner; the login page hides the link (`BrandingService.isEnabled`).
- Register response is deliberately generic (no account enumeration) → always continue to `/portal/verify?email=…`.
- Portal token expired while browsing → the auth interceptor signs the customer out and redirects to `/portal/login?returnUrl=…`.

## Test Plan

Out of scope (build-level verification only). Manual smoke: wrong password, lockout after repeated failures, reload keeps the session, portal register → verify → profile.

## Verification Steps

1. **Frontend builds:** `cd customer-support-crm-web && npx ng build` with no warnings.

## Done Criteria

- [x] Staff can sign in, stay signed in across reloads, and sign out.
- [x] Error codes show localized messages; lockout and disabled are distinct.
- [x] Customers can register, verify, sign in and edit their profile in the portal.
- [x] Portal and staff sessions are independent; guards redirect correctly.
