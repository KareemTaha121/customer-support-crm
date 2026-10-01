# Frontend — Angular web app for all 12 features

Repo: `customer-support-crm-web` (Angular 22, standalone components, signals, zoneless, Angular Material 22 M3).
Backend: `customer-support-crm-api` (complete on `develop`). Intakes: `.squad/stories/frontend/*/intake.md`.
Out of scope for every story: Docker, `deploy/`, CI/CD, unit and e2e tests. Verification is `ng build` with **0 errors and 0 warnings**.

| NN | Tracker id | Plan file | Title | Depends on | Status |
|----|-----------|-----------|-------|-----------|--------|
| 08 | FE-01 | [08-story-core-platform-shell.md](08-story-core-platform-shell.md) | Core platform shell | backend | Done |
| 09 | FE-02 | [09-story-authentication-staff-and-portal.md](09-story-authentication-staff-and-portal.md) | Authentication (staff and portal) | 08 | Done |
| 10 | FE-03 | [10-story-customers-ui.md](10-story-customers-ui.md) | Customers UI (feature 01) | 08, 09 | Done |
| 11 | FE-04 | [11-story-tickets-ui.md](11-story-tickets-ui.md) | Tickets UI (feature 02) | 08, 09 | Done |
| 12 | FE-05 | [12-story-channels-and-live-chat-console.md](12-story-channels-and-live-chat-console.md) | Channels and live chat console (feature 03) | 08, 09 | Done |
| 13 | FE-06 | [13-story-agent-dashboard-ui.md](13-story-agent-dashboard-ui.md) | Agent dashboard UI (feature 04) | 08, 09 | Done |
| 14 | FE-07 | [14-story-sla-and-automation-admin-ui.md](14-story-sla-and-automation-admin-ui.md) | SLA and automation admin UI (feature 05) | 08, 09 | Done |
| 15 | FE-08 | [15-story-knowledge-base-ui.md](15-story-knowledge-base-ui.md) | Knowledge base UI (feature 06) | 08, 09 | Done |
| 16 | FE-09 | [16-story-ai-assistant-panels.md](16-story-ai-assistant-panels.md) | AI assistant panels (feature 07) | 08, 09, 11 | Done |
| 17 | FE-10 | [17-story-customer-portal-ui.md](17-story-customer-portal-ui.md) | Customer portal UI (feature 08) | 08, 09 | Done |
| 18 | FE-11 | [18-story-reports-ui.md](18-story-reports-ui.md) | Reports UI (feature 09) | 08, 09 | Done |
| 19 | FE-12 | [19-story-administration-ui.md](19-story-administration-ui.md) | Administration UI (features 10, 11, 12) | 08, 09 | Done |
| 50 | BUG-14 | [50-story-composer-error-state-after-send.md](50-story-composer-error-state-after-send.md) | Composers do not show "This field is required." after a send (QA M1) | 12, 17 | To do |
| 52 | BUG-16 | [52-story-rtl-bidi-isolation-and-date-formats.md](52-story-rtl-bidi-isolation-and-date-formats.md) | RTL bidi isolation, one date format rule, Material datepicker (QA M3, L5) | 51 | To do |
| 56 | BUG-20 | [56-story-ui-polish-qa-batch.md](56-story-ui-polish-qa-batch.md) | UI polish batch: portal nav scrollbar, duplicate Agent label, form gaps, page title (QA L2, L6) | 08 | To do |
| 58 | BUG-22 | [58-story-quick-reply-ticket-context.md](58-story-quick-reply-ticket-context.md) | Quick replies fill customer and ticket placeholders (QA round 2, N2) | 13, 11, 12 | Done (`48d8533`) |
| 62 | BUG-26 | [62-story-task-past-date-warning.md](62-story-task-past-date-warning.md) | Warn when a task due date or reminder is already past (QA round 2, N6) | 13 | Done (`b91dd22`) |

Stories 10–19 are independent of each other (each owns its own folders), so they can be implemented in parallel.

---

## Conventions every feature story follows

These are the contracts Story 08 created. Read the cited files before writing feature code.

### Ownership (avoid merge conflicts)

- A feature story edits **only** `src/app/features/<feature>/**` and `public/i18n/<scope>/{en,ar}.json`.
- Never edit `src/app/core/**`, `src/app/shared/**`, `src/app/app.*`, `src/styles.scss`, or another feature's folder. If something shared is missing, create it inside your feature folder.
- The top-level route and the sidenav entry already exist: `src/app/app.routes.ts` lazy-loads `features/<feature>/<feature>.routes.ts` (export name fixed there), `src/app/core/layout/navigation.ts` holds the menu. Replace the placeholder routes file; keep the export name(s).

| Story | Folder | Routes export(s) | URL | i18n scope |
|-------|--------|------------------|-----|-----------|
| 10 | `features/customers` | `CUSTOMERS_ROUTES` | `/customers` | `customers` |
| 11 | `features/tickets` | `TICKETS_ROUTES` (incl. `categories`) | `/tickets`, `/tickets/categories` | `tickets` |
| 12 | `features/channels` | `CHAT_ROUTES`, `CHANNELS_ROUTES` | `/chat`, `/channels` | `channels` |
| 13 | `features/dashboard` | `DASHBOARD_ROUTES` | `/dashboard` (tasks, quick replies as child routes) | `dashboard` |
| 14 | `features/sla` | `SLA_ROUTES` | `/sla` | `sla` |
| 15 | `features/knowledge-base` | `KNOWLEDGE_BASE_ROUTES`, `HELP_CENTER_ROUTES` | `/knowledge-base`, `/help` (public) | `kb` |
| 16 | `features/ai` | — (component library) | embedded in ticket details | `ai` |
| 17 | `features/customer-portal` | `PORTAL_ROUTES` (extend, keep auth pages) | `/portal/*` | `portal` (extend) |
| 18 | `features/reports` | `REPORTS_ROUTES` | `/reports` | `reports` |
| 19 | `features/administration` | `ADMINISTRATION_ROUTES` | `/admin/users`, `/admin/roles`, `/admin/organization`, `/admin/settings`, `/admin/integrations`, `/admin/audit` | `admin` |

Cross-feature component contracts (placeholders exist; the owning story replaces the body, keeping selector/inputs/outputs):

- `features/ai/ticket-ai-panel.component.ts` — `<app-ticket-ai-panel [ticketId] (replySuggested) (categorySuggested)>` (owner: Story 16, used by Story 11).
- `features/dashboard/quick-reply-picker.component.ts` — `<app-quick-reply-picker (selected)>` emits reply body text (owner: Story 13, used by Stories 11 and 12).

### HTTP

- Use `ApiService` (`src/app/core/http/api.service.ts`): `get<T>`, `getPaged<T>` (returns `Paged<T>` = `{ items, meta }`), `post`, `put`, `patch`, `delete`, `upload(path, file)`, `download(path)` (returns `{ blob, fileName }`, save with `saveBlob` from `shared/file-utils.ts`). Paths are relative to `/api/v1` (`'/tickets'`, `'/public/kb/articles'`, `'/portal/tickets'`).
- Every call fails with `ApiError` (`core/http/api-error.ts`): `status`, `code` (feature code such as `TICKET_NOT_FOUND`), `errors[]`, `fieldErrors`, `isValidation`, `hasCode()`.
- Errors show a global snackbar automatically. For forms pass `{ silent: true }` and render errors yourself: `applyServerErrors(form, error)` (`shared/form-errors.ts`) puts server field errors on controls; show the rest with `NotificationToastService.error(describeError(apiError, translations))`.
- The bearer token, refresh, `Accept-Language` and correlation id are handled by interceptors. Portal calls (`/portal/*`) use the customer token automatically; `/public/*` is anonymous.
- Model TypeScript interfaces on the C# records: `customer-support-crm-api/src/CustomerSupportCrm.Contracts/**` and records declared inside `Application/Features/**/*.cs`. JSON is camelCase, enums are strings, `Guid` → `string`, `DateTimeOffset` → ISO `string`. Put them in `features/<feature>/<feature>.models.ts`, and API calls in `features/<feature>/<feature>.api.ts` (`@Injectable({ providedIn: 'root' })`).

### Components

- Standalone, `ChangeDetectionStrategy.OnPush`, state in `signal()`/`computed()` (the app is **zoneless**: never rely on zone change detection; set signals in subscribe callbacks). Use `input()`/`output()`, `inject()`, the built-in control flow (`@if`, `@for`, `@switch`), and reactive forms (`NonNullableFormBuilder`).
- Page layout: `<app-page-header [title]="'x.y' | t">actions</app-page-header>` (`shared/page-header.component.ts`); states `<app-loading />`, `<app-empty-state />`, `<app-error-state (retry)>` (`shared/state.components.ts`); confirmation `ConfirmService.ask({ title, message, destructive })` (`shared/confirm-dialog.component.ts`).
- Global CSS helpers (`src/styles.scss`): `crm-card`, `crm-grid`, `crm-form-grid` (+ `crm-span-all`), `crm-toolbar`, `crm-actions`, `crm-table-wrap` (+ `crm-row-link`), `crm-pill crm-pill--success|warning|danger|info|primary`, `crm-muted`, `crm-danger` (destructive flat button), `rtl-flip` (mirror arrow icons). Material form fields default to `outline`.
- Tables: `mat-table` + `mat-paginator` (server paging via `page`/`pageSize`), `mat-sort` when the endpoint supports `sortBy`/`sortDirection`. Keep component styles under 8 kB.
- Dates: `{{ value | localDate }}` / `'shortDate'` / `'relative'` (`core/localization/localized-date.pipe.ts`). File sizes: `fileSize` pipe.

### Permissions

- Codes in `core/permissions/permissions.ts` (`Permissions.ticketsAssign`, ...). Hide actions with `*appHasPermission="'tickets.assign'"` (`core/permissions/has-permission.directive.ts`) or `PermissionService.has()`. Guard routes with `canActivate: [requirePermission(Permissions.x)]` (`core/guards/auth.guards.ts`).

### i18n (en/ar, RTL)

- No hard-coded user-facing text. Keys `<scope>.<path>` in `public/i18n/<scope>/en.json` and `ar.json` (same key set, real Arabic). Load the scope in the feature routes: `resolve: { i18n: translationResolver('<scope>') }` on the root route of the feature. Use `{{ 'scope.key' | t }}` (`TranslatePipe`) or `TranslationService.t()`; placeholders `{{name}}`. Common words exist under `core.actions.*`, `core.states.*`, `core.fields.*`, `core.validation.*`.
- Enum values from the API (statuses, priorities) are translated via keys like `tickets.status.Open`.
- Use logical CSS properties (`margin-inline-start`, `inset-inline-end`) so RTL works; add `rtl-flip` to directional icons.

### Realtime

- `StaffHubService` (`core/realtime/staff-hub.service.ts`): `hub.on<T>(RealtimeEvents.ticketUpdated)` returns an Observable (unsubscribe with `takeUntilDestroyed`), `hub.invoke('JoinConversation', id)`.

### Verification

- `cd customer-support-crm-web && npx ng build` must finish with no errors and no warnings (strict TypeScript, strict templates, extended diagnostics as errors).
