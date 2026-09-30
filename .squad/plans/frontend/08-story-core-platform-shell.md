# Story 08 — Core platform shell (Story: FE-01)

## Prerequisites

- Backend complete on `customer-support-crm-api` `develop` (all 12 features).
- API conventions: `customer-support-crm-api/docs/api-contract.md` (envelope, error codes, paging, correlation id).
- Web repo scaffold: `customer-support-crm-web` (Angular 22.2, zoneless, no Material yet).

---

## Story Goal

Give every feature story a platform to plug into, without building any feature screen:

1. A typed API client that unwraps the `ApiResponse` envelope and raises typed errors.
2. Staff session handling with an in-memory access token and a single-flight cookie refresh.
3. Runtime en/ar translations with RTL, loaded per feature.
4. Permission checks for templates and routes.
5. A responsive staff shell with permission-filtered navigation and a realtime notification bell.
6. Branding (name, colors, logo) and public feature flags from the anonymous endpoints.
7. Lazy route placeholders for every feature so stories 10–19 can run in parallel.

Not in scope: login and portal pages (Story 09), feature screens (Stories 10–19).

---

## Context — Read These Files First

1. `customer-support-crm-api/docs/api-contract.md` — envelope fields `success`, `data`, `message`, `errors[{code,message,field}]`, `meta{page,pageSize,totalCount,totalPages}`, `correlationId`; status → category table.
2. `customer-support-crm-api/src/CustomerSupportCrm.Contracts/Common/ApiResponse.cs`, `ApiError.cs`, `PaginationMeta.cs`, `AttachmentResponse.cs` — the shapes mirrored in TypeScript.
3. `customer-support-crm-api/src/CustomerSupportCrm.Application/Features/Authentication/Common/AuthenticationHttp.cs` — refresh cookie `crm_refresh` (path `/api/v1/auth`, SameSite Strict) and required header `X-CSRF-Protection`.
4. `customer-support-crm-api/src/CustomerSupportCrm.Contracts/Authentication/AuthenticationContracts.cs` — `AccessTokenResponse(accessToken, expiresAt, user)`, `CurrentUserResponse(id, email, displayName, roles, permissions)`.
5. `customer-support-crm-api/src/CustomerSupportCrm.Domain/Roles/Permissions.cs` — permission codes.
6. `customer-support-crm-api/src/CustomerSupportCrm.Infrastructure/Realtime/StaffHub.cs` and `Application/Abstractions/Notifications/IRealtimeNotifier.cs` — hub path `/hubs/staff`, events `notificationCreated`, `chatMessage`, `chatUpdated`, `ticketUpdated`; token via `access_token` query (`Infrastructure/DependencyInjection.cs` ~line 174).
7. `customer-support-crm-api/src/CustomerSupportCrm.Application/Features/Notifications/NotificationSlices.cs` — `GET /notifications` (paged), `GET /notifications/unread-count`, `POST /notifications/{id}/read`, `POST /notifications/read-all`.
8. `customer-support-crm-api/src/CustomerSupportCrm.Application/Features/Organization/OrganizationProfile.cs` (~lines 167–196) and `Features/Settings/SettingsSlices.cs` (~lines 123–132) — `GET /public/branding`, `GET /public/branding/logo`, `GET /public/features`.

---

## Frontend Tasks

### 1 — Packages and compiler settings

- Install `@angular/material@~22.2`, `@angular/cdk@~22.2`, `@microsoft/signalr@^10`, `chart.js@^4` (charts used by Story 18).
- File: `tsconfig.json` — add `"strict": true`, `strictTemplates`, and `extendedDiagnostics.defaultCategory = "error"` so warnings fail the build.
- File: `angular.json` — budgets: initial 1 MB warning / 2 MB error, component style 8 kB / 16 kB.

### 2 — HTTP layer (`src/app/core/http/`)

- Create `api.models.ts`: `ApiEnvelope<T>`, `ApiErrorItem`, `PaginationMeta`, `Paged<T>`, `PageQuery`, `QueryParams`, `AttachmentResponse`.
- Create `api-error.ts`: `ApiError(status, code, message, errors, correlationId)` with `fieldErrors`, `isValidation`, `hasCode()`, and `ApiError.from(unknown)` mapping `HttpErrorResponse` (feature code = first error with `field === null`, else category by status).
- Create `api.service.ts`: `ApiService` (`get`, `getPaged`, `post`, `put`, `patch`, `delete`, `upload`, `download`), `HttpContextToken`s `SILENT_ERRORS` and `SKIP_AUTH`, options `{ params, silent, anonymous, headers, withCredentials }`.
- Create `src/app/core/config/app-config.ts`: `API_BASE_URL`, `API_ORIGIN`, `apiUrl()`, `isApiUrl()`, `SUPPORTED_LANGUAGES`.

### 3 — Sessions (`src/app/core/auth/`)

- `auth.service.ts`: signals `currentUser`, `isAuthenticated`, `permissions`; `login`, `refresh` (single-flight via `shareReplay`, CSRF header, `withCredentials`), `restoreSession`, `reloadCurrentUser` (`GET /auth/me`), `changePassword`, `logout`, `expireSession`.
- `portal-auth.service.ts`: customer session in `sessionStorage` (`crm.portal.session`), `register`, `verify`, `resendVerification`, `login` under `/public/portal/*`, `loadProfile` (`GET /portal/me`), `logout`.

### 4 — Interceptors (`src/app/core/interceptors/`), registered in `app.config.ts` in this order

1. `request-headers.interceptor.ts` — `Accept-Language`, `X-Correlation-Id` (32 hex chars).
2. `auth.interceptor.ts` — `/public/*` anonymous; `/portal/*` portal token (401 → portal logout and redirect); staff token otherwise, proactive refresh when expiring, one refresh + replay on 401, `expireSession()` when refresh fails.
3. `error.interceptor.ts` — snackbar via `core/layout/notification-toast.service.ts` unless `SILENT_ERRORS`; exports `describeError(apiError, translations)`.

### 5 — Localization (`src/app/core/localization/`)

- `translation.service.ts` — loads `public/i18n/<scope>/<lang>.json` through `HttpBackend`, flattens to `<scope>.<path>`, `setLanguage()` reloads loaded scopes and sets `<html lang dir>`, persists `crm.language`.
- `translate.pipe.ts` (`t`), `translation.resolver.ts` (`translationResolver(...scopes)`), `localized-date.pipe.ts` (`localDate`), `paginator-intl.ts` (`MatPaginatorIntl`).
- `src/app/app.ts` wraps the router outlet in `[dir]` (CDK `Dir`) so Material follows language switches.
- Create `public/i18n/core/en.json` and `ar.json` (nav, actions, states, validation, error categories, status pages, paginator).

### 6 — Permissions and guards

- `core/permissions/permissions.ts` (codes), `permission.service.ts` (`has`, `hasAny`, `hasAll`), `has-permission.directive.ts` (`*appHasPermission`).
- `core/guards/auth.guards.ts` — `authGuard` (lazy `restoreSession()` once), `guestGuard`, `permissionGuard` (route data), `requirePermission(...)`, `portalAuthGuard`, `portalGuestGuard`.

### 7 — Branding and realtime

- `core/branding/branding.service.ts` — loads branding + features in the app initializer, sets `--mat-sys-primary`/`--mat-sys-tertiary` (+ on-colors), document title, `logoSrc()`, `isEnabled(flag)`; `feature-flags.ts` holds the public keys.
- `core/realtime/staff-hub.service.ts` — connects while signed in, `on<T>(event)`, `invoke(method, ...args)`, reconnects, retries a failed start every 15 s.

### 8 — Shell and shared UI

- `core/layout/staff-shell.component.ts` (+ `.scss`): sidenav (`over` below 960 px), sections from `core/layout/navigation.ts` filtered by permission, toolbar with `language-switcher.component.ts`, `notification-bell.component.ts`, user menu (profile, sign out).
- `core/layout/status-pages.component.ts`: `NotFoundComponent`, `ForbiddenComponent`, `ComingSoonComponent`.
- `src/app/shared/`: `page-header.component.ts`, `state.components.ts`, `confirm-dialog.component.ts` (`ConfirmService`), `form-errors.ts` + `form-error.pipe.ts`, `file-utils.ts` + `file-size.pipe.ts`, `password.ts`.
- `src/styles.scss`: M3 theme, fonts (Roboto / IBM Plex Sans Arabic via `src/index.html`), shared `crm-*` helpers.

### 9 — Routes

- File: `src/app/app.routes.ts` — `/login`, `/portal`, `/help`, and the staff shell (`authGuard`) with lazy children `dashboard`, `tickets`, `customers`, `chat`, `channels`, `knowledge-base`, `sla`, `reports`, `admin`, `profile`, `forbidden`, `**`.
- Create placeholder `features/<feature>/<feature>.routes.ts` files (export names listed in `00-overview.md`) and the contract placeholders `features/ai/ticket-ai-panel.component.ts`, `features/dashboard/quick-reply-picker.component.ts`.

---

## Edge Cases & Failure Modes

- Many requests fail with 401 at once → one `POST /auth/refresh` (`AuthService.refresh` shares the in-flight observable), all replay with the new token.
- Refresh fails (cookie expired, reuse detected) → `expireSession(returnUrl)` clears state and routes to `/login?returnUrl=…`; no refresh loop because the replay is outside the `catchError`.
- Entering through `/portal` or `/help` then opening a staff URL → `authGuard` tries the cookie once (`hasAttemptedRestore`).
- Branding endpoint down or invalid colors → fallback palette (`safeColor`).
- Translation file missing → key rendered as-is and a console warning; the app keeps working.
- Hub unreachable → warning, retry in 15 s; HTTP features unaffected.
- `localStorage`/`sessionStorage` blocked → language and portal session live in memory only.

## Test Plan

Out of scope for this project phase (build-level verification only). Manual smoke: sign in, reload (session restored), switch language (RTL), open the bell.

## Verification Steps

1. **Frontend builds:** `cd customer-support-crm-web && npx ng build` → "Application bundle generation complete" with no warnings.

## Done Criteria

- [x] `ng build` succeeds with no errors and no warnings.
- [x] Envelope unwrapping, typed errors and paging work for any endpoint.
- [x] A 401 triggers exactly one refresh call; queued requests replay; refresh failure signs out.
- [x] Arabic flips the shell to RTL and reloads translations without a page reload.
- [x] Navigation items and routes are hidden/blocked without the required permission.
- [x] Branding colors/name/logo come from the public branding endpoint.
- [x] Notification bell shows the unread count and updates in realtime.
