# Story intake

- Folder: `.squad/stories/frontend/core-platform-shell/intake.md`

---

## Feature

- **Feature name (display):** Frontend — Angular web app for all 12 features
- **Feature slug (folder under `plans/`):** `frontend`

## Tracker (metadata only)

- **Tracker type:** `none`
- **Work item id:** `FE-01`
- **Work item type:** `Story`
- **Status:** `Ready`
- **Assignee:** ``
- **Labels:** `frontend`, `core`, `platform`

---

## Title

```
Core platform shell
```

---

## Description

```
Build the platform layer of customer-support-crm-web (Angular 22, standalone,
signals) that every feature story plugs into. No feature screens.

- Angular Material (M3 theme) + CDK; global SCSS tokens; theme colors driven by
  CSS variables so branding from GET /api/v1/public/branding (primaryColor,
  accentColor, name, logoUrl, defaultCulture) is applied at startup.
- Typed API client over HttpClient that unwraps the ApiResponse envelope
  { success, data, message, errors[{code,message,field}], meta{page,pageSize,
  totalCount,totalPages}, correlationId } into data / Paged<T>, and maps
  failures into a typed ApiError (status, code, message, fieldErrors).
- Interceptors: base URL + credentials, bearer token, single-flight refresh on
  401 (POST /auth/refresh with X-CSRF-Protection: 1, withCredentials), correlation
  id (X-Correlation-Id), Accept-Language, global error snackbar.
- i18n: runtime en/ar translation service (signals) with a `t` pipe, per-feature
  JSON files under public/i18n/<scope>/<lang>.json, dir="rtl" for Arabic, and
  language persisted in localStorage.
- Permission service (from the current user's permission codes), *appHasPermission
  structural directive, permissionGuard / authGuard / guestGuard route guards.
- Responsive staff shell: sidenav (collapses on mobile) with permission-filtered
  navigation for every feature, toolbar with language switch, user menu and a
  notification bell fed by GET /notifications, /notifications/unread-count and the
  SignalR /hubs/staff `notificationCreated` event (token via access_token query).
- Shared building blocks: page header, empty/loading/error states, confirm dialog,
  paged table helper, form error helper that maps server field errors to controls.
- Lazy route skeleton for all feature areas (each feature owns its *.routes.ts).
```

---

## Acceptance criteria

```
- [ ] `ng build` succeeds with no errors and no warnings.
- [ ] Envelope unwrapping, typed errors and paging work for any endpoint.
- [ ] A 401 triggers exactly one refresh call; queued requests replay with the new token; refresh failure logs out.
- [ ] Switching to Arabic flips the whole shell to RTL and reloads translations without a page reload.
- [ ] Navigation items and routes are hidden/blocked without the required permission.
- [ ] Branding colors/name/logo come from the public branding endpoint.
- [ ] Notification bell shows unread count and updates in realtime.
```

---

## Attachments

None.

---

## Dependencies

- **Blocked by / related ids:** none
- **Depends on code areas or other stories:** backend `customer-support-crm-api` (complete, develop branch); API contract `customer-support-crm-api/docs/api-contract.md`.

## Extra notes (optional)

- Feature specs: `.squad/features/*.md` (Frontend sections).

## Technical hints (optional)

- Repo root: `customer-support-crm-web/` (Angular 22, standalone components, signals, Angular Material 22).
- DTOs: mirror `customer-support-crm-api/src/CustomerSupportCrm.Contracts/**` and response records declared in the slice files under `Application/Features/**`.

## Out of scope

- Docker, deploy/, CI/CD, unit/e2e tests (verification is `ng build` only).
- Backend changes.
