# Story 21 — Organization context, branding and notifications (Story: PL-02)

> As-built plan: written after implementation in `customer-support-crm-api` commit `65c74a3` (feat: add platform services and organization context (phase 3)). Paths and line numbers refer to `65c74a3` unless another commit is named. The tables for this story were first created by migration `20260930104602_AddSupportOperations` in `678ea67`. Arabic (and most English) messages for its error codes were added in `0936711`.

## Prerequisites

- Story 20 completed: [20-story-backend-foundation.md](20-story-backend-foundation.md) — envelope, `ApiResults`, `GlobalExceptionHandler`, `ValidationBehavior`, `IEndpoint` discovery, localization.
- Phase 2 completed: [../security-and-administration/00-overview.md](../security-and-administration/00-overview.md) (`0f87e2d`) — `User`, `Role`, `IApplicationDbContext`, `IAuditTrail`, `ICurrentUser`, one policy per permission, `UserQueries`, `UserSessionService`, `PagedResult<T>` / `ToPagedResultAsync`, `CommonRules`, `ApiResults.Success()`, `AuditableEntityInterceptor`, `DatabaseInitializer`.
- All paths below are relative to the **`customer-support-crm-api/`** repo root.

---

## Story Goal

1. Model the deployment as one **Organization** with **Branches** and **Departments**; give business data an owner (`IScopedEntity`: `BranchId` + optional `DepartmentId`).
2. Let administrators manage the organization profile, branding (colors, logo), branches and departments, and restrict each user to branches/departments (`UserScope`).
3. Resolve a per-request **AccessScope** that later features use to filter reads (404 when out of scope) and guard writes (403 `OUT_OF_SCOPE`).
4. Deliver **in-app notifications** stored per user and pushed over SignalR after commit.
5. Add shared platform services the later features rely on: in-transaction domain events, soft delete, file storage with upload validation, recurring background jobs, staff/portal/public/external endpoint groups, the full permission catalog.

**Deviations from the intake (the code is authoritative):**

| Intake / feature spec | As built in `65c74a3` |
|---|---|
| Slices `Features/Branding/` (Get, Update, UploadLogo) | Branding lives in `Features/Organization/OrganizationProfile.cs` (`PUT /organization/branding`, `POST /organization/logo`) and `GET /public/branding` |
| Branding "per organization" | One organization per deployment (`OrganizationStore.GetAsync` reads the first row) |
| Branding used in email templates | Emails use the organization name and `PrimaryColor` only (`Features/Channels/CustomerMessaging.cs`, later commit `e921626`); no logo or accent color in emails |
| `Features/Localization/` (Languages, Resources) | Not built. Languages are fixed to en/ar in code (`Organization.SupportedCultures`, `LocalizationExtensions.SupportedCultures`) |
| Branches CRUD / Departments CRUD | Create, update, activate, deactivate; **no delete** (by design) and **no GET by id**. Departments are listed only nested inside `GET /branches` |
| Organization writes need `organization.manage` | Profile, branding and logo need `settings.manage`; only branches/departments need `organization.manage`. `GET /organization` and `GET /branches` need only a staff token |
| User scopes "assign scopes" slice | `PUT /users/{id}/scopes` in `Features/Users/Administration/UserAdministration.cs` (one file holds Update, SetScopes, ResetPassword, Lookup) |
| Localized messages for new codes | `65c74a3` added **no** resx keys; the codes returned English fallbacks until `0936711` |
| Database migration | None in `65c74a3`. `organizations`, `branches`, `departments`, `user_scopes` and `notifications` are created by `20260930104602_AddSupportOperations` (`678ea67`) |
| `ORGANIZATION_UNIT_INACTIVE` | Constant `OrganizationErrors.InactiveUnit` exists but nothing throws it: deactivated branches/departments can still be assigned |

**Not in scope:** system settings and audit export ([../security-and-administration/22-story-system-settings-and-audit-export.md](../security-and-administration/22-story-system-settings-and-audit-export.md)); feature entities that implement `IScopedEntity` (customers `0fd694e`, tickets `57e52f8`). **Do not touch** `docker-compose.yml`, `deploy/`, `.github/` or anything under `tests/`.

---

## Context — Read These Files First

1. `src/CustomerSupportCrm.Api/Endpoints/EndpointExtensions.cs` — lines 16–46. Four groups (`/api/v1` staff, `/api/v1/portal`, `/api/v1/public`, `/api/v1/external`) and the two hubs.
2. `src/CustomerSupportCrm.Application/Abstractions/Http/IEndpoint.cs` — lines 9–30 (`IEndpoint`, `IPortalEndpoint`, `IPublicEndpoint`, `IExternalEndpoint`); `src/CustomerSupportCrm.Application/DependencyInjection.cs` lines 28–45 (scoped services and multi-interface endpoint scanning).
3. `src/CustomerSupportCrm.Application/Abstractions/Authorization/PolicyNames.cs` lines 7–20; `src/CustomerSupportCrm.Infrastructure/Authorization/AuthorizationSetup.cs` lines 16–38 (`RequireStaff` = authenticated + `actor=staff`); `Abstractions/Authentication/CrmClaimTypes.cs` lines 4–24.
4. `src/CustomerSupportCrm.Domain/Roles/Permissions.cs` — lines 9–49 (catalog), 51–62 (`All`), 65–81 (`AgentDefaults`, `ManagerDefaults`). `OrganizationManage` line 44, `SettingsManage` 45, `DataAllBranches` 49.
5. `src/CustomerSupportCrm.Domain/Organizations/Organization.cs` lines 10–117, `Branch.cs` lines 6–70, `Department.cs` lines 6–68.
6. `src/CustomerSupportCrm.Domain/Users/User.cs` — `Scopes` (59), `Rename` (82), `SetScopes` (111), `UserScope` (157).
7. `src/CustomerSupportCrm.Domain/Common/AggregateRoot.cs` lines 4–37, `IScopedEntity.cs` lines 7–12, `ISoftDeletable.cs` lines 7–14.
8. `src/CustomerSupportCrm.Application/Common/Authorization/AccessScope.cs` — `AccessScope` (14–28), `AccessScopeProvider` (36–70), `WhereInScope` / `EnsureAccess` / `EnsureCanAssign` (75–109), `OrganizationErrors` (112–121).
9. `src/CustomerSupportCrm.Application/Common/Files/FileUploadRules.cs` — limits and codes (16–23), type table (34–50), `ValidateAsync` (57–99), `CreateStorageKey` (102–103).
10. `src/CustomerSupportCrm.Infrastructure/Persistence/ApplicationDbContext.cs` — `SaveChangesAsync` (45–66: dispatch events, save, push notifications), soft-delete filters (80–98), `DispatchDomainEventsAsync` (100–120, max 10 rounds).
11. `src/CustomerSupportCrm.Infrastructure/Persistence/Interceptors/AuditableEntityInterceptor.cs` lines 38–44 (delete → soft delete).
12. `src/CustomerSupportCrm.Infrastructure/Persistence/Configurations/OrganizationConfiguration.cs` lines 10–91.
13. `src/CustomerSupportCrm.Application/Features/Users/Common/UserQueries.cs` — `GetResponseAsync` now projects scopes (37–45, 57); `ResolveScopesAsync` (86–109).

---

## Backend Tasks

### 1 — Domain

Create `src/CustomerSupportCrm.Domain/Organizations/`:

- `Organization.cs` — `Entity<Guid>, IAuditableEntity`; `Name` (max 200), `SupportEmail`, `SupportPhone`, `DefaultCulture` (`SupportedCultures = ["en", "ar"]`), `TimeZone` (max 64), `PrimaryColor` (default `#1f6feb`), `AccentColor` (default `#0e9f6e`), `LogoStorageKey`, `LogoContentType`. `Create(name)` (65–70), `UpdateProfile` (72–95), `UpdateBranding` (upper-cases, `#RRGGBB` regex, 97–106), `SetLogo` (108–113). Invariant failures → `DomainException("INVALID_ORGANIZATION")`.
- `Branch.cs` — `Code` (max 20, `NormalizeCode` = trim + upper), `Name` (max 200), `Address`, `Phone`, `IsActive`; `Create`, `Update`, `Activate`, `Deactivate`; `INVALID_BRANCH`. Doc comment: "Deactivated, never deleted."
- `Department.cs` — `BranchId`, `Code`, `Name`, `Email` (normalized through `EmailAddress.Create`; used by the email channel for routing), `IsActive`; `INVALID_DEPARTMENT`.

Create `src/CustomerSupportCrm.Domain/Notifications/Notification.cs` — `RecipientId` (`UserId`), `Type` (max 64), `Title` (max 300, truncated), `Body`, `Link` (max 500), `Data` (JSON string), `CreatedAt`, `ReadAt`; `MarkRead` is idempotent (`ReadAt ??= now`). `NotificationTypes` (65–75): `ticket.assigned`, `ticket.replied`, `ticket.mentioned`, `ticket.escalated`, `sla.warning`, `sla.breached`, `task.reminder`, `chat.waiting`.

Create in `src/CustomerSupportCrm.Domain/Common/`: `AggregateRoot<TId>` with `IDomainEvent`, `IHasDomainEvents`, `Raise`, `DequeueDomainEvents`; `IScopedEntity { Guid BranchId; Guid? DepartmentId; }`; `ISoftDeletable { IsDeleted; DeletedAt; DeletedBy; }`.

Update `src/CustomerSupportCrm.Domain/Users/User.cs` — `_scopes` list, `Scopes` navigation, `SetScopes(IEnumerable<(Guid BranchId, Guid? DepartmentId)>)` (distinct, removes missing, adds new), and `UserScope(UserId, BranchId, DepartmentId?)` with a v7 `Id`.

Update `src/CustomerSupportCrm.Domain/Roles/Permissions.cs` — full catalog for all features (32 codes incl. `audit.export`, `organization.manage`, `integrations.manage`, `data.all_branches`) and `AgentDefaults` / `ManagerDefaults`, used by `DatabaseInitializer.DefaultRoles`.

### 2 — Persistence

- `IApplicationDbContext.cs` — becomes `partial` (each feature adds `IApplicationDbContext.<Feature>.cs`); adds `Organizations`, `Branches`, `Departments`, `Notifications`.
- `ApplicationDbContext.cs` — `partial`, constructor takes `IPublisher` and `IRealtimeNotifier`. `SaveChangesAsync`: dispatch domain events (wrapped by `DomainEventNotification.Wrap`, handlers run in the same unit of work and must not save), capture added `Notification`s, save, then `SendToUserAsync(recipient, "notificationCreated", { id, type, title, link, createdAt })`. `ApplySoftDeleteFilters` adds `!EF.Property<bool>(e, "IsDeleted")` to every root `ISoftDeletable` type.
- `AuditableEntityInterceptor.cs` — `Deleted` `ISoftDeletable` entries become `Modified` with `IsDeleted = true`, `DeletedAt`, `DeletedBy`.
- `Configurations/OrganizationConfiguration.cs` — tables `organizations`, `branches` (unique `code`), `departments` (unique `(branch_id, code)`, index on `email`, FK restrict), `user_scopes` (unique `(user_id, branch_id, department_id)` with `AreNullsDistinct(false)`, FKs restrict), `notifications` (`data` jsonb, index `(recipient_id, read_at, created_at)`, cascade on user delete). `xmin` concurrency on organization, branch, department.
- `Configurations/UserConfiguration.cs` — `HasMany(Scopes)` cascade, field access mode.
- `Seed/BootstrapOptions.cs` — `OrganizationName = "Customer Support"`. `Seed/DatabaseInitializer.cs` `SeedOrganizationAsync` (39–56): first run creates the organization, branch `HQ` "Head Office" and department `SUPPORT` "Customer Support".
- **Migration:** none in this commit (see Deviations). Created later by `20260930104602_AddSupportOperations` (`678ea67`).

### 3 — Contracts

- `src/CustomerSupportCrm.Contracts/Organization/OrganizationContracts.cs` — `OrganizationResponse`, `UpdateOrganizationRequest`, `UpdateBrandingRequest`, `PublicBrandingResponse`, `BranchRequest`, `DepartmentRequest`, `DepartmentResponse`, `BranchResponse` (with `Departments`).
- `src/CustomerSupportCrm.Contracts/Notifications/NotificationContracts.cs` — `NotificationResponse(Id, Type, Title, Body, Link, JsonElement? Data, CreatedAt, ReadAt)`, `UnreadCountResponse(int Count)`.
- `src/CustomerSupportCrm.Contracts/Users/UserContracts.cs` — `CreateUserRequest` gains `Scopes`; new `UpdateUserRequest`, `ResetPasswordRequest`, `UserScopeRequest(BranchId, DepartmentId?)`, `SetUserScopesRequest`, `UserScopeResponse(BranchId, BranchName, DepartmentId?, DepartmentName?)`, `UserLookupResponse(Id, DisplayName, Email)`; `UserResponse` gains `Scopes`.

### 4 — Organization profile and branding

File: `src/CustomerSupportCrm.Application/Features/Organization/OrganizationProfile.cs`.

- `OrganizationStore` (20–40): `NotConfigured = "ORGANIZATION_NOT_CONFIGURED"`; `ToResponse` builds `LogoUrl = "/api/v1/public/branding/logo?v=<UpdatedAt unix seconds>"` for cache busting.
- `GetOrganizationQuery` / handler (42–48).
- `UpdateOrganizationCommand` + validator (name max 200, email, phone max 32, culture `en|ar` → `INVALID_VALUE`, `TimeZoneInfo.TryFindSystemTimeZoneById` → `INVALID_VALUE`) + handler auditing `organization.updated` with before/after (50–79).
- `UpdateBrandingCommand` + validator (`^#[0-9A-Fa-f]{6}$`) + handler auditing `organization.branding_updated` (81–105).
- `UploadLogoCommand` + handler (107–132): `FileUploadRules.ValidateAsync(..., MaxImageBytes, Images)`; key `branding/yyyy/MM/<guid><ext>`; save, set logo, audit `organization.logo_updated` (file name + size), save, then delete the previous object.
- `OrganizationEndpoints` (134–166) under `/organization`, tag `Organization`: `GET /` (staff), `PUT /`, `PUT /branding`, `POST /logo` (`IFormFile file`, `.DisableAntiforgery()`) — writes require `Permissions.SettingsManage`.
- `PublicBrandingEndpoints : IPublicEndpoint` (169–195): `GET /branding` → `PublicBrandingResponse`; `GET /branding/logo` → `Results.Stream` with the stored content type and `Cache-Control: public, max-age=86400`, or 404.

### 5 — Branches and departments

File: `src/CustomerSupportCrm.Application/Features/Branches/BranchEndpoints.cs`.

- `BranchQueries.ListAsync` (21–46): `AsNoTracking`, branches by name, departments loaded in one query and nested; `GetAsync` (50–52) → 404 `BRANCH_NOT_FOUND`.
- `ListBranchesQuery(IncludeInactive)`; `SaveBranchCommand(BranchId?, Code, Name, Address, Phone)` with validator (code `^[A-Za-z0-9_-]+$` max 20) and one handler for create and update (77–106): unique normalized code → 409 `BRANCH_CODE_TAKEN`; audit `branches.created` / `branches.updated`.
- `SetBranchActiveCommand` (108–134): no-op when unchanged; audit `branches.activated` / `branches.deactivated`.
- Endpoints (136–175): `GET /branches?includeInactive` (staff), `POST /branches` (201, `Location: /api/v1/branches/{id}`), `PUT /branches/{id:guid}`, `POST /branches/{id:guid}/activate|deactivate` — writes require `Permissions.OrganizationManage`.

File: `src/CustomerSupportCrm.Application/Features/Departments/DepartmentEndpoints.cs`.

- `SaveDepartmentCommand(DepartmentId?, BranchId, Code, Name, Email)` + validator + handler (20–66): branch must exist (404 `BRANCH_NOT_FOUND`); code unique within the branch (409 `DEPARTMENT_CODE_TAKEN`); update requires the department to belong to `branchId` (404 `DEPARTMENT_NOT_FOUND`); audit `departments.created` / `departments.updated`.
- `SetDepartmentActiveCommand` (68–94).
- Endpoints (96–128): `POST /branches/{branchId:guid}/departments` (201, `Location` points to the branch), `PUT /branches/{branchId:guid}/departments/{id:guid}`, `POST /departments/{id:guid}/activate|deactivate`; all `organization.manage`.

### 6 — Access scope

File: `src/CustomerSupportCrm.Application/Common/Authorization/AccessScope.cs`, registered scoped as `IAccessScopeProvider` (`Application/DependencyInjection.cs` line 29).

- `AccessScopeProvider.GetAsync` caches per request: anonymous → empty scope; `data.all_branches` → `AccessScope.Everything`; otherwise branch ids from scope rows without a department, department ids from the rest.
- `WhereInScope<T>` filters `IScopedEntity` queries in SQL; `EnsureAccess` throws `NotFoundException(notFoundCode)` (hides existence); `EnsureCanAssign` throws `ForbiddenException("OUT_OF_SCOPE")`.
- Used later by customers, tickets, dashboard, live chat, knowledge base and AI slices.

### 7 — User administration (branch/department assignment)

File: `src/CustomerSupportCrm.Application/Features/Users/Administration/UserAdministration.cs`.

- `UpdateUserCommand(UserId, DisplayName)` (23–45) → `user.Rename`, audit `users.updated`.
- `SetUserScopesCommand(UserId, Scopes)` (47–76) → `UserQueries.ResolveScopesAsync` (400 `DEPARTMENT_NOT_IN_BRANCH` on field `scopes`), `user.SetScopes`, audit `users.scopes_changed` with old/new lists.
- `ResetUserPasswordCommand(UserId, NewPassword)` (79–104) → `CommonRules.ValidPassword`, rehash, revoke **all** the user's refresh tokens (`RefreshTokenRevocationReason.PasswordChanged`), audit `users.password_reset` (no password in values).
- `LookupUsersQuery(Search, DepartmentId, Permission)` (107–139) → active users, upper-case search on name/email, department filter (scope on the department or on its whole branch), permission filter through role permissions; ordered by name, `Take(50)`.
- `UserAdministrationEndpoints` (141–173): `PUT /users/{id:guid}`, `PUT /users/{id:guid}/scopes`, `POST /users/{id:guid}/reset-password` (`ApiResults.Success()`) — `users.manage`; `GET /users/lookup` — staff only.
- `Features/Users/Create/*` — `CreateUserCommand` takes optional `Scopes`; the handler calls `ResolveScopesAsync` + `SetScopes` when any are given.

### 8 — Notifications

File: `src/CustomerSupportCrm.Application/Features/Notifications/NotificationSlices.cs`.

- `NotificationSender` (24–45, scoped): `Notify(recipient, type, title, link, data, body)` adds a row in the caller's unit of work (data serialized with web JSON options); `NotifyMany(..., except)` skips duplicates and the actor. Domain-event handlers in later features call it.
- `ListNotificationsQuery(Page, PageSize, UnreadOnly)` + validator (`ValidPage`, `ValidPageSize`) + handler (47–85): own notifications, newest first, `ToPagedResultAsync`, `Data` parsed back to `JsonElement`.
- `GetUnreadCountQuery` (87–96); `MarkNotificationsReadCommand(NotificationId?)` (98–120): marks one or all unread, returns the new count; an unknown or foreign id simply changes nothing.
- `NotificationEndpoints` (122–148) under `/notifications`, tag `Notifications`; no permission (own data only).

### 9 — Realtime, files, jobs, customer actor

- `Application/Abstractions/Notifications/IRealtimeNotifier.cs` — `SendToUserAsync`, `SendToGroupAsync`; `RealtimeEvents` (`notificationCreated`, `chatMessage`, `chatUpdated`, `ticketUpdated`); `RealtimeGroups.ChatAgents`, `ChatConversation(id)`.
- `Infrastructure/Realtime/RealtimeHubs.cs` — `StaffHub` (`/hubs/staff`, staff policy; joins `chat-agents` when the token has `chat.handle`; `JoinConversation` / `LeaveConversation`), `SubjectUserIdProvider` (SignalR user = `sub`), `SignalRRealtimeNotifier` (best effort, logs failures at Warning). `Realtime/ChatHub.cs` — `/hubs/chat`, anonymous, `JoinConversation(id, accessToken)` checked by `IChatAccessValidator`.
- `Infrastructure/DependencyInjection.cs` — `TimeProvider.System`, `AddHttpContextAccessor`, `AddPlatformServices` (lines 72–82: `StorageOptions` + `LocalFileStorage`, SignalR, `BackgroundJobOptions`); JWT `OnMessageReceived` reads `access_token` from the query for `/hubs` (114–127).
- `Application/Abstractions/Files/IFileStorage.cs` + `Infrastructure/Files/LocalFileStorage.cs` — section `Storage:RootPath` (default `App_Data/storage`); `Resolve` rejects keys that escape the root; `FileMode.CreateNew`.
- `Infrastructure/BackgroundJobs/RecurringRequestService.cs` — `RecurringRequestService<TRequest>` sends a MediatR request on a `PeriodicTimer` in a fresh scope, logs failures and keeps going; `BackgroundJobs:Enabled`; `AddRecurringRequest<TRequest>(interval)`. No job is registered in this commit.
- Customer actor: `ICurrentCustomer` / `HttpCurrentCustomer`, `ITokenService.CreateCustomerAccessToken` (`actor=customer`, `cid`, no permissions), `JwtOptions` portal lifetime; staff tokens now carry `actor=staff`.

### 10 — Localization (follow-up `0936711`)

`Messages.ar.resx` lines 111–150 and 186–195 (at `0936711`): `SERVICE_UNAVAILABLE`, `FEATURE_DISABLED`, `OUT_OF_SCOPE`, `ORGANIZATION_NOT_CONFIGURED`, `ORGANIZATION_UNIT_INACTIVE`, `INVALID_ORGANIZATION`, `BRANCH_NOT_FOUND`, `BRANCH_CODE_TAKEN`, `INVALID_BRANCH`, `DEPARTMENT_NOT_FOUND`, `DEPARTMENT_CODE_TAKEN`, `DEPARTMENT_NOT_IN_BRANCH`, `INVALID_DEPARTMENT`, `UNKNOWN_SETTING`, `FILE_EMPTY`, `FILE_TOO_LARGE`, `FILE_TYPE_NOT_ALLOWED`, `FILE_SIGNATURE_MISMATCH`. `Messages.resx` has the fixed-message subset (lines 111–135, 159–165); `BRANCH_NOT_FOUND`, `DEPARTMENT_NOT_FOUND`, `INVALID_ORGANIZATION`, `FILE_TOO_LARGE` keep their throw-site English.

### 11 — Docs

`docs/endpoints.md` (added in `fdd93cd`) lists these routes: users (lines 38–41), organization/branches/departments (47–60), notifications (122–123), public `/branding`, `/branding/logo` (200) and hubs (227).

---

## Edge Cases & Failure Modes

- **No organization row** (initializer not run) — every organization and branding call → 404 `ORGANIZATION_NOT_CONFIGURED`.
- **Duplicate branch code with different case/spaces** — normalized to upper case → 409 `BRANCH_CODE_TAKEN`; race past the check hits the unique index → 409 `CONFLICT`.
- **Same department code in two branches** — allowed; within one branch → 409 `DEPARTMENT_CODE_TAKEN`.
- **Update a department under the wrong branch id** — 404 `DEPARTMENT_NOT_FOUND`.
- **Activate/deactivate twice** — no change, no audit entry, 200 with the current state.
- **Deactivated branch or department** — still returned with `includeInactive=true`; still accepted in user scopes and data writes (`ORGANIZATION_UNIT_INACTIVE` unused).
- **Scope with a department from another branch, or an unknown id** — 400 `DEPARTMENT_NOT_IN_BRANCH` (field `scopes`); duplicate rows are removed.
- **User with no scopes and no `data.all_branches`** — sees no scoped data at all.
- **Logo**: empty → `FILE_EMPTY`; > 2 MB → `FILE_TOO_LARGE`; `.svg`/`.pdf` → `FILE_TYPE_NOT_ALLOWED`; a renamed `.exe` → `FILE_SIGNATURE_MISMATCH`; client MIME type ignored; file name sanitized; previous logo deleted only after commit.
- **No logo uploaded** — `logoUrl` is null; `GET /public/branding/logo` → 404 envelope.
- **Invalid colors or time zone** — 400 validation (`INVALID_FORMAT` / `INVALID_VALUE`); the domain re-checks → 422.
- **Mark read for an unknown or another user's notification** — 200, count unchanged.
- **SignalR push fails** — logged at Warning; the notification is already committed.
- **Domain events that keep raising events** — `InvalidOperationException` after 10 rounds → 500.
- **Concurrent edits** — `xmin` → 409 `CONFLICT`.
- **Customer token on a staff route** — 403 (staff policy requires `actor=staff`).

---

## Test Plan

Squad plans do not add or change tests (`tests/` is out of scope). `65c74a3` only adjusted `tests/CustomerSupportCrm.Domain.Tests/Roles/RoleTests.cs` for the larger catalog. **No existing test covers** organization, branches, departments, user scopes, access scope, uploads or notifications.

---

## Verification Steps

1. **Build:** `dotnet build` in `customer-support-crm-api/` — 0 warnings, 0 errors.
2. **Migrate + seed:** at `678ea67` or later, start the API with `Database:InitializeOnStartup: true` (the Development default) — tables exist; one organization, branch `HQ`, department `SUPPORT`.
3. **Sign in** as the bootstrap admin and export `TOKEN`.
4. **Public branding:** `curl -k https://localhost:5001/api/v1/public/branding` (no token) → name, `defaultCulture`, colors, `logoUrl: null`.
5. **Branding:** `curl -k -X PUT …/api/v1/organization/branding -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" -d '{"primaryColor":"#112233","accentColor":"#445566"}'` → upper-cased colors. `-F "file=@logo.png"` to `POST …/organization/logo` → `logoUrl` with `?v=`; `GET` that URL → `image/png`, `Cache-Control: public, max-age=86400`.
6. **Branches:** `POST …/branches -d '{"code":"jed","name":"Jeddah"}'` → 201, `code: "JED"`; repeat → 409 `BRANCH_CODE_TAKEN`; `POST …/branches/{id}/departments -d '{"code":"billing","name":"Billing"}'` → 201; `GET …/branches` → nested departments.
7. **Scopes:** `PUT …/users/{agentId}/scopes -d '{"scopes":[{"branchId":"<JED>","departmentId":null}]}'` → `scopes[0].branchName: "Jeddah"`; a department from `HQ` under `JED` → 400 `DEPARTMENT_NOT_IN_BRANCH`.
8. **Lookup:** `GET …/users/lookup?search=agent&permission=chat.handle` with an Agent token → 200, at most 50 rows.
9. **Notifications:** `GET …/notifications?unreadOnly=true` → paged envelope with `meta`; `POST …/notifications/read-all` → `{ count: 0 }`.
10. **Permissions:** Agent token on `POST …/branches` → 403; customer portal token on `GET …/branches` → 403.
11. **Localization:** a failing call with `Accept-Language: ar` → Arabic message (after `0936711`).

---

## Done Criteria

- [x] Organization → Branch → Department model; `IScopedEntity` ownership contract; no single-branch assumption in the access model (`AccessScope`).
- [x] Admin creates/updates/activates/deactivates branches and departments (`organization.manage`).
- [x] Admin assigns users to branches/departments (`PUT /users/{id}/scopes`, scopes on create); reads filtered and writes guarded through `AccessScope`.
- [x] Organization profile, colors and logo; anonymous `GET /public/branding` for login page and portal.
- [x] Soft-delete query filter + interceptor; auditable stamps; every change in this story audited.
- [x] Recurring background job host (`RecurringRequestService<TRequest>`).
- [x] In-app notifications, unread count, mark read; SignalR push after commit; staff and chat hubs.
- [x] Staff / portal / public / external endpoint groups; full permission catalog.
- [ ] Database migration — not in `65c74a3`; tables created in `678ea67`.
- [ ] Messages localized — not in `65c74a3`; added in `0936711`.
- [ ] Branding in emails — name and primary color only; no logo.
- [ ] `ORGANIZATION_UNIT_INACTIVE` enforcement, branch/department GET by id, `Features/Localization` — not built.
- [x] Nothing changed in `docker-compose.yml`, `deploy/` or `.github/`.
- [x] `dotnet build` passes with zero warnings.

**STOP HERE. Report to the user and wait for confirmation before proceeding to Story 22.**
