# Story 20 — Backend foundation (API platform) (Story: PL-01)

> As-built plan: written after implementation in `customer-support-crm-api` commit `c3c7815` (feat: add Phase 1 API foundation), on top of the scaffold `84f7bb1`. Paths and line numbers refer to `c3c7815` unless another commit is named. Follow-up `0936711` (localized messages for all feature error codes) is covered in task 9.

## Prerequisites

- Phase 0 scaffold `84f7bb1` (chore: scaffold repository structure): `CustomerSupportCrm.sln`, the five `src/` projects, the four `tests/` projects, `Directory.Build.props`, `Directory.Packages.props`, `global.json`, `.editorconfig`, the empty feature-slice folders (`Application/Features/<Feature>/.gitkeep`), `deploy/docker/Dockerfile`, `docker-compose.yml`, `.github/workflows/ci.yml` and doc placeholders.
- A PostgreSQL instance for `/health/ready` and the integration tests (`appsettings.Development.json` lines 9–11 point at `localhost:5432/customer_support_crm`).
- All paths below are relative to the **`customer-support-crm-api/`** repo root.

---

## Story Goal

Give every later feature slice the same platform:

1. A Clean Architecture solution with vertical slices: Api → Application → Domain, Contracts shared, Infrastructure behind Application abstractions.
2. One response envelope for success and failure, with stable error codes and localized messages (en/ar).
3. A MediatR pipeline that validates every request with FluentValidation before its handler runs.
4. Correlation ids, structured Serilog logging, health checks, OpenAPI, and EF Core on PostgreSQL.
5. Endpoints that register themselves: a slice adds an `IEndpoint` class and nothing else.

**Deviations from the intake (the code is authoritative):**

| Intake / feature spec | As built in `c3c7815` |
|---|---|
| API versioning | URL prefix only: `EndpointExtensions.ApiV1Prefix = "/api/v1"`. There is no `Asp.Versioning` package or version negotiation; `docs/api-contract.md` says a new version is added only for a breaking change |
| Pagination helpers | Only the `PaginationMeta` contract and `ApiResults.Paged`. `PagedResult<T>`, `ToPagedResultAsync` and the `CommonRules` validators (`ValidPage`, `ValidPageSize`) came in `0f87e2d` (Phase 2) |
| `ApiResults.Success()` (no payload) | Added in `0f87e2d`, not in Phase 1 |
| Soft delete and auditing conventions | Not in Phase 1. Auditable stamps and the audit log came in `0f87e2d`; `ISoftDeletable`, the soft-delete query filter and interceptor came in `65c74a3` (see Story 21) |
| Background jobs infrastructure | Not in Phase 1. `RecurringRequestService<TRequest>` came in `65c74a3` (Story 21) |
| Database migration strategy | No migration in Phase 1 (`ApplicationDbContext` has no entities). The first migration is `20260930092037_InitialIdentity` (`0f87e2d`), applied by `DatabaseInitializer.InitializeAsync` (`db.Database.MigrateAsync`) |
| Docker, CI, environments | Only the scaffold files from `84f7bb1` (`deploy/docker/Dockerfile`, `docker-compose.yml`, `.github/workflows/ci.yml`); Phase 1 did not change them |
| Swagger in every environment | `/openapi/v1.json` and `/swagger` are mapped in Development only (`Program.cs` lines 32–35) |
| Arabic messages for feature codes | Phase 1 shipped only the 12 category codes. Feature codes were added story by story; `0936711` filled in Arabic for every business code |

**Not in scope:** authentication and authorization (Phase 2), the frontend shell (`../frontend/08-story-core-platform-shell.md`). **Do not touch** `docker-compose.yml`, `deploy/` or `.github/` in feature stories.

---

## Context — Read These Files First

1. `Directory.Build.props` — `net10.0`, `Nullable`, `TreatWarningsAsErrors`, `AnalysisLevel latest-recommended`, `EnforceCodeStyleInBuild`. This is why the build has zero warnings.
2. `Directory.Packages.props` — central versions: MediatR 12.5.0 (last Apache-2.0 release), FluentValidation 12.1.1, Npgsql EF Core 10.0.3, EFCore.NamingConventions 10.0.1, Serilog.AspNetCore 10.0.0, Microsoft.OpenApi 2.12.2 (pinned above a vulnerable transitive version).
3. `src/CustomerSupportCrm.Api/Program.cs` — lines 13–20 (services), 26–30 (middleware order: correlation → localization → request logging → exception handler → status code pages), 32–35 (OpenAPI in Development), 39–40 (health, endpoints).
4. `src/CustomerSupportCrm.Contracts/Common/ApiResponse.cs` lines 6–28, `ApiError.cs` line 6, `ErrorCodes.cs` lines 7–31, `PaginationMeta.cs` lines 3–14.
5. `src/CustomerSupportCrm.Application/Abstractions/Http/ApiResults.cs` lines 10–44 — `Ok`, `Created(location, data)`, `Paged(items, meta)` and `ApiResult<T>`, which stamps the correlation id into the body (line 41).
6. `src/CustomerSupportCrm.Application/Abstractions/Http/IEndpoint.cs` lines 9–12 and `src/CustomerSupportCrm.Application/DependencyInjection.cs` lines 12–37 (MediatR, `ValidationBehavior`, validators with `includeInternalTypes: true`, `AddEndpoints` scanning, `AddLocalization`).
7. `src/CustomerSupportCrm.Api/Endpoints/EndpointExtensions.cs` lines 7–20 — maps every `IEndpoint` on the `/api/v1` group.
8. `src/CustomerSupportCrm.Application/Behaviors/ValidationBehavior.cs` lines 9–30.
9. `src/CustomerSupportCrm.Api/Middleware/GlobalExceptionHandler.cs` lines 24–36 (exception → status), 54–60 (`StatusFor`), 65–80 (`ToValidationCode`, `IsStableCode`), 82–85 (camelCase field path).
10. `src/CustomerSupportCrm.Api/Middleware/ErrorResponseWriter.cs` lines 15–27 (`WriteAsync`), 30–35 (status code pages), 37–42 (`Localize`), 44–58 (`CategoryFor`).
11. `src/CustomerSupportCrm.Api/Middleware/CorrelationIdMiddleware.cs` lines 13–37 and `src/CustomerSupportCrm.Application/Abstractions/Http/CorrelationIdHttpContextExtensions.cs` lines 7–15.
12. `src/CustomerSupportCrm.Api/Localization/LocalizationExtensions.cs` lines 5–20; `src/CustomerSupportCrm.Application/Resources/Messages.cs` line 7.
13. `src/CustomerSupportCrm.Api/Configuration/LoggingExtensions.cs` lines 18–34 (sinks) and 40–64 (request logging).
14. `src/CustomerSupportCrm.Infrastructure/DependencyInjection.cs` lines 11–41 and `Persistence/DatabaseOptions.cs` lines 5–17.
15. `docs/api-contract.md`, `docs/architecture.md`, `docs/development.md`, `docs/adr/0001-vertical-slice-architecture.md`, `docs/adr/0003-api-contract.md`.

---

## Backend Tasks

### 1 — Contracts (`src/CustomerSupportCrm.Contracts/Common/`)

- `ApiResponse.cs` — `sealed record ApiResponse<T>` with `required bool Success`, `T? Data`, `string? Message`, `IReadOnlyList<ApiError> Errors = []`, `PaginationMeta? Meta`, `string? CorrelationId`; static `ApiResponse.Ok<T>(data, message, meta)` and `ApiResponse.Failure(message, errors)`.
- `ApiError.cs` — `record ApiError(string Code, string Message, string? Field = null)`.
- `ErrorCodes.cs` — category codes (`BAD_REQUEST`, `VALIDATION_ERROR`, `UNAUTHORIZED`, `FORBIDDEN`, `NOT_FOUND`, `METHOD_NOT_ALLOWED`, `CONFLICT`, `PAYLOAD_TOO_LARGE`, `UNSUPPORTED_MEDIA_TYPE`, `BUSINESS_RULE_VIOLATION`, `RATE_LIMITED`, `INTERNAL_ERROR`) and field codes (`REQUIRED`, `INVALID_LENGTH`, `INVALID_FORMAT`, `INVALID_EMAIL`, `OUT_OF_RANGE`, `INVALID_VALUE`, `INVALID`). Values never change.
- `PaginationMeta.cs` — `record PaginationMeta(int Page, int PageSize, long TotalCount, int TotalPages)`; `Create(page, pageSize, totalCount)` guards `page >= 1`, `pageSize >= 1`, `totalCount >= 0` and computes `TotalPages` by ceiling division.

### 2 — Exceptions

- `src/CustomerSupportCrm.Domain/Common/DomainException.cs` — `DomainException(string code, string message)` with `Code`. Raised for invariants and state transitions → 422.
- `src/CustomerSupportCrm.Application/Common/Exceptions/AppException.cs` — abstract base `(code, message)`; the message is a fallback, the API localizes by `Code`.
- `NotFoundException`, `ConflictException`, `ForbiddenException` — sealed, primary constructors with default codes `NOT_FOUND` / `CONFLICT` / `FORBIDDEN` and default English messages.

### 3 — HTTP adapter for slices

- `Abstractions/Http/IEndpoint.cs` — `void MapEndpoint(IEndpointRouteBuilder app)`.
- `Abstractions/Http/ApiResults.cs` — `ApiResult<T> : IResult, IStatusCodeHttpResult, IValueHttpResult<ApiResponse<T>>`; `ExecuteAsync` sets status, `Location` (for `Created`) and writes `response with { CorrelationId = httpContext.GetCorrelationId() }`. Slices always return these, never raw `Results.*`.
- `Abstractions/Http/CorrelationIdHttpContextExtensions.cs` — `HeaderName = "X-Correlation-Id"`, `GetCorrelationId` / `SetCorrelationId` on `HttpContext.Items`.
- `Application/DependencyInjection.cs` — `AddApplication()` registers MediatR from the assembly with `AddOpenBehavior(typeof(ValidationBehavior<,>))`, validators (`includeInternalTypes: true`), `AddEndpoints(assembly)` (every non-abstract `IEndpoint` as a transient `IEndpoint` via `TryAddEnumerable`) and `AddLocalization()`.
- `Api/Endpoints/EndpointExtensions.cs` — `MapApiEndpoints()` creates `app.MapGroup("/api/v1")` and calls `MapEndpoint` on every registered `IEndpoint`. (Phase 2 adds `.RequireAuthorization()` to this group; Phase 3 adds the portal, public and external groups.)

### 4 — Validation pipeline

`Behaviors/ValidationBehavior.cs` — `IPipelineBehavior<TRequest, TResponse>` over `IEnumerable<IValidator<TRequest>>`. Each validator gets its own `ValidationContext` (line 20); failures from all validators are collected and thrown as one `FluentValidation.ValidationException` (lines 24–27). With no validators, the handler runs directly.

### 5 — Error handling

- `Api/Middleware/GlobalExceptionHandler.cs` — `IExceptionHandler`. A client abort (`OperationCanceledException` with `RequestAborted`) is logged at Information and swallowed. The switch (lines 24–36) produces `(status, category, errors)`; validation failures become one `ApiError` per failure with the mapped code and camelCase field path (`items[0].quantity` style via `JsonNamingPolicy.CamelCase`). 5xx logs the exception at Error; everything else logs only the category code at Information. Exception details never reach the body.
- `Api/Middleware/ErrorResponseWriter.cs` — the only place error bodies are written: localized category message, `errors` defaulting to `[ApiError(category, message)]`, correlation id, and `Content-Language` re-set because the exception handler clears headers (line 25).
- `Program.cs` line 29 `UseExceptionHandler(_ => { })` and line 30 `UseStatusCodePages(ErrorResponseWriter.WriteStatusCodePageAsync)` (envelope for unknown route, 405 and auth challenges).

### 6 — Localization

- `Api/Localization/LocalizationExtensions.cs` — `DefaultCulture = "en"`, `SupportedCultures = ["en", "ar"]`, `ApplyCurrentCultureToResponseHeaders = true`. The default request-culture providers are used (query string, cookie, `Accept-Language`).
- `Application/Resources/Messages.cs` — marker class for `IStringLocalizer<Messages>`; keys are error codes.
- `Application/Resources/Messages.resx` and `Messages.ar.resx` — one `<data>` per category code (lines 15–50 in both).

### 7 — Correlation id and logging

- `Api/Middleware/CorrelationIdMiddleware.cs` — reads `X-Correlation-Id`; keeps it only when it matches `^[A-Za-z0-9._-]{1,64}$` (generated regex, line 36), else `Guid.NewGuid().ToString("N")`. Sets the response header in `OnStarting` (survives the exception handler) and pushes `CorrelationId` into the Serilog `LogContext`.
- `Api/Configuration/LoggingExtensions.cs` — `AddApiLogging()` reads the `Serilog` section, enriches from log context, writes text in Development and `RenderedCompactJsonFormatter` elsewhere. `UseApiRequestLogging()` logs `HTTP {RequestMethod} {RequestPath} responded {StatusCode}`, Error for exceptions/5xx, Verbose for `/health`, and adds `RequestId`, `Route` and `UserId` (from `NameIdentifier`) to the event.
- `appsettings.json` lines 2–14 — `Serilog` levels (`Microsoft.AspNetCore`, `Microsoft.EntityFrameworkCore`, `System.Net.Http.HttpClient` at Warning) and the `Application` property.

### 8 — Persistence, health and OpenAPI

- `Infrastructure/Persistence/DatabaseOptions.cs` — section `Database`: `[Required] ConnectionString`, `[Range(1, 600)] CommandTimeoutSeconds = 30`, `EnableSensitiveDataLogging` (Development only).
- `Infrastructure/DependencyInjection.cs` — `AddInfrastructure` → `AddPersistence`: options bound with `ValidateDataAnnotations().ValidateOnStart()`; `AddDbContext<ApplicationDbContext>` with `UseNpgsql(..., CommandTimeout)`, `UseSnakeCaseNamingConvention()`; `AddHealthChecks().AddDbContextCheck<ApplicationDbContext>("database", tags: ["ready"])`. `ReadinessTag = "ready"` (line 11).
- `Infrastructure/Persistence/ApplicationDbContext.cs` — `ApplyConfigurationsFromAssembly` only; no entities yet.
- `Api/Health/HealthCheckExtensions.cs` — `/health/live` (`Predicate = _ => false`) and `/health/ready` (checks tagged `ready`); both return only the overall status.
- `Api/OpenApi/OpenApiExtensions.cs` — `AddOpenApi("v1")` with title "Customer Support CRM API"; `MapApiOpenApi()` serves `/openapi/v1.json` and Swagger UI.
- `Program.cs` line 18 — `JsonStringEnumConverter` for enums as strings.

### 9 — Follow-up `0936711`: localized messages for every feature error code

- `Api/Middleware/GlobalExceptionHandler.cs` (at `0936711`) — `ToApiError(HttpContext, ValidationFailure)` localizes feature codes on validation failures (`UNKNOWN_SETTING`, `INVALID_VERIFICATION_CODE`, `DEPARTMENT_NOT_IN_BRANCH`, file rules, …). The generic field codes in `GenericFieldCodes` (`REQUIRED`, `INVALID_LENGTH`, `INVALID_EMAIL`, `INVALID_FORMAT`, `OUT_OF_RANGE`, `INVALID_VALUE`, `INVALID`) keep the validator's own message, which names the field and limit.
- `Messages.ar.resx` grows from 32 to 101 entries (every business code). `Messages.resx` grows from 32 to 74; codes thrown with several or parameterized English messages (for example `INVALID_TICKET`, `FILE_TOO_LARGE`) are only in the Arabic file, so English keeps the throw-site message (doc comment on `Messages.cs`).
- Adds `SERVICE_UNAVAILABLE`; before this, 503 responses returned the raw code as the message.

### 10 — Docs

`docs/api-contract.md` (base path and versioning, envelope, status/category table, field codes, pagination, correlation id, serialization), `docs/architecture.md` (project dependencies, request pipeline), `docs/development.md`, ADRs 0001 and 0003. `.gitattributes` and `.editorconfig` set LF line endings.

---

## Edge Cases & Failure Modes

- **Malformed JSON body / unbindable parameter** — `BadHttpRequestException` → 400 `BAD_REQUEST` envelope.
- **Unknown route / wrong method** — status code pages → 404 `NOT_FOUND` / 405 `METHOD_NOT_ALLOWED` envelope.
- **Unhandled exception** — 500 `INTERNAL_ERROR`, generic localized message, details only in the log.
- **Client aborted the request** — no response body; logged at Information.
- **Malformed or overlong `X-Correlation-Id`** — replaced by a generated id; never echoed back.
- **Unsupported `Accept-Language`** (for example `fr`) — falls back to `en`.
- **Missing message key** — `Localize` returns the fallback (the exception's English message or the code itself).
- **Missing connection string** — `ValidateOnStart` fails the host at startup.
- **Database down** — `/health/ready` reports Unhealthy; `/health/live` stays Healthy.
- **Several validators on one request** — all failures are returned together; the handler does not run.

---

## Test Plan

Squad plans do not add or change tests (`tests/` is out of scope). Tests that shipped with `c3c7815`, listed here for reference only:

1. **API** — `tests/CustomerSupportCrm.Api.Tests/ApiContractTests.cs`: `SuccessResponseUsesStandardEnvelope`, `PagedResponseIncludesPaginationMeta`, `CreatedResponseReturns201WithLocation`, `ValidationFailureReturnsFieldErrorsWithStableCodes`, `MessagesAreLocalizedFromAcceptLanguage`, `NotFoundExceptionReturnsFeatureSpecificCode`, `DomainExceptionReturnsBusinessRuleViolation`, `UnhandledExceptionReturnsGenericErrorWithoutDetails`, `UnknownRouteReturnsEnvelope`, `MalformedJsonReturnsBadRequestEnvelope`, `OpenApiDocumentIsServedInDevelopment` (against `TestEndpoints.cs`).
2. **API** — `CorrelationIdTests.cs`: `EchoesWellFormedIncomingCorrelationId`, `GeneratesCorrelationIdWhenMissing`, `ReplacesMalformedCorrelationId`.
3. **API** — `HealthCheckTests.cs`: `LivenessIsHealthyWithoutDependencies`, `ReadinessIsUnhealthyWhenDatabaseIsUnreachable`.
4. **Application** — `tests/CustomerSupportCrm.Application.Tests/Behaviors/ValidationBehaviorTests.cs` (3 tests) and `Contracts/PaginationMetaTests.cs` (`CalculatesTotalPages`, `RejectsInvalidArguments`).
5. **Integration (Testcontainers PostgreSQL)** — `tests/CustomerSupportCrm.IntegrationTests/DatabaseTests.cs`: `ReadinessIsHealthyWhenDatabaseIsReachable`, `DbContextConnectsToPostgres`.

No test covers the `0936711` localization change.

---

## Verification Steps

1. **Build:** in `customer-support-crm-api/` run `dotnet build` — 0 warnings, 0 errors (warnings are errors).
2. **Tests:** `dotnet test --filter "FullyQualifiedName!~IntegrationTests"` passes; the integration tests need Docker for Testcontainers.
3. **Run:** `dotnet run --project src/CustomerSupportCrm.Api --launch-profile https`.
4. **Health:** `curl -k https://localhost:5001/health/live` → `Healthy`; `curl -k https://localhost:5001/health/ready` → `Healthy` with PostgreSQL up, `Unhealthy` without.
5. **Envelope on unknown route:** `curl -k -i https://localhost:5001/api/v1/does-not-exist -H "X-Correlation-Id: probe-1"` → 404, `X-Correlation-Id: probe-1`, body `success: false`, `errors[0].code = "NOT_FOUND"`, `correlationId: "probe-1"`.
6. **Localization:** repeat with `-H "Accept-Language: ar"` → Arabic `message`, `Content-Language: ar`.
7. **Bad correlation id:** `-H "X-Correlation-Id: bad id!"` → a generated 32-char hex id comes back.
8. **OpenAPI:** `curl -k https://localhost:5001/openapi/v1.json` → document titled "Customer Support CRM API"; `/swagger` loads in Development.
9. **Logs:** the console shows one `HTTP GET /api/v1/does-not-exist responded 404` event with `CorrelationId`; no headers or bodies.

---

## Done Criteria

- [x] Solution layout, central build settings and package versions; Clean Architecture project references (`docs/architecture.md`).
- [x] Standard envelope on every `/api/v1` response and framework error; stable category and field codes.
- [x] `GlobalExceptionHandler` maps exceptions to codes and never returns exception details.
- [x] `ValidationBehavior` runs FluentValidation before every handler.
- [x] `PaginationMeta` contract and `ApiResults.Paged`. (Paging helpers arrived in `0f87e2d`.)
- [x] en/ar resource messages; `Accept-Language` honored; `Content-Language` returned; feature codes localized in `0936711`.
- [x] Correlation id middleware, Serilog structured logging, `/health/live` and `/health/ready`, OpenAPI + Swagger UI (Development).
- [x] EF Core + Npgsql, snake_case, validated `DatabaseOptions`.
- [x] Endpoints discovered from `IEndpoint` implementations and mapped under `/api/v1`.
- [ ] API versioning beyond the `/api/v1` URL prefix — not built.
- [ ] Soft delete, auditing, background jobs — not in this story (Phase 2 / Story 21).
- [ ] Docker, CI, environments, migration strategy — scaffold files only; first migration in `0f87e2d`.
- [x] `dotnet build` passes with zero warnings.

**STOP HERE. Report to the user and wait for confirmation before proceeding to Story 21.**
