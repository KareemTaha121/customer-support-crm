# Story intake

- Folder: `.squad/stories/frontend/authentication-staff-and-portal/intake.md`

---

## Feature

- **Feature name (display):** Frontend — Angular web app for all 12 features
- **Feature slug (folder under `plans/`):** `frontend`

## Tracker (metadata only)

- **Tracker type:** `none`
- **Work item id:** `FE-02`
- **Work item type:** `Story`
- **Status:** `Done`
- **Assignee:** ``
- **Labels:** `frontend`, `auth`

---

## Title

```
Authentication (staff and portal)
```

---

## Description

```
Staff: login page (POST /api/v1/auth/login {email,password} -> {accessToken,
expiresAt, user{id,email,displayName,roles,permissions}}; refresh cookie crm_refresh),
silent session restore on app start via POST /auth/refresh, logout (POST
/auth/logout with CSRF header), profile page with change password (POST
/auth/change-password {currentPassword,newPassword}). Map error codes
INVALID_CREDENTIALS, ACCOUNT_LOCKED, ACCOUNT_DISABLED to localized messages.
Access token kept in memory only.

Portal (customers): register, verify email, resend verification, login under
/api/v1/public/portal/*; portal token (8h, no refresh) kept in sessionStorage;
portal profile (GET/PUT /portal/me) and change password; portal guard and a
separate portal shell/layout. Respect the public feature flag for registration
(GET /public/features).
```

---

## Acceptance criteria

```
- [ ] Staff can sign in, stay signed in across reloads (cookie refresh), and sign out.
- [ ] Error codes show localized messages; lockout and disabled are distinct.
- [ ] Customers can register, verify, sign in and edit their profile in the portal.
- [ ] Portal and staff sessions are independent; guards redirect correctly.
```

---

## Attachments

None.

---

## Dependencies

- **Blocked by / related ids:** FE-01
- **Depends on code areas or other stories:** backend `customer-support-crm-api` (complete, develop branch); API contract `customer-support-crm-api/docs/api-contract.md`.

## Extra notes (optional)

- Feature specs: `.squad/features/*.md` (Frontend sections).

## Technical hints (optional)

- Repo root: `customer-support-crm-web/` (Angular 22, standalone components, signals, Angular Material 22).
- DTOs: mirror `customer-support-crm-api/src/CustomerSupportCrm.Contracts/**` and response records declared in the slice files under `Application/Features/**`.

## Out of scope

- Docker, deploy/, CI/CD, unit/e2e tests (verification is `ng build` only).
- Backend changes.
