# Story intake

- Folder: `.squad/stories/platform/backend-foundation/intake.md`

---

## Feature

- **Feature name (display):** Platform — Phase 1 Backend Platform
- **Feature slug (folder under `plans/`):** `platform`

## Tracker (metadata only)

- **Tracker type:** `none`
- **Work item id:** `PL-01`
- **Work item type:** `Story`
- **Status:** `Done`
- **Assignee:** ``
- **Labels:** `phase-1`, `backend`, `platform`

---

## Title

```
Backend foundation (API platform)
```

---

## Description

```
The cross-cutting API platform every feature slice builds on, in customer-support-crm-api
(.NET 10). Solution layout: Api, Application, Domain, Infrastructure, Contracts projects plus
four test projects; central build settings (TreatWarningsAsErrors, latest-recommended
analyzers) and central package versions.

Surface:
- /api/v1                 every business endpoint; slices implement IEndpoint and are
                          discovered by assembly scanning (no hand-written registration)
- /health/live            liveness, runs no checks
- /health/ready           readiness, runs checks tagged "ready" (PostgreSQL DbContext check)
- /openapi/v1.json        OpenAPI document (Development only)
- /swagger                Swagger UI (Development only)

Conventions:
- Envelope: { success, data, message, errors[], meta, correlationId } on every /api/v1
  response, success or failure (ApiResponse<T>, ApiResults.Ok/Created/Paged).
- Errors: GlobalExceptionHandler maps ValidationException 400 VALIDATION_ERROR,
  NotFoundException 404, ConflictException 409, ForbiddenException 403, other AppException 400,
  DomainException 422 BUSINESS_RULE_VIOLATION, BadHttpRequestException -> its status,
  anything else 500 INTERNAL_ERROR (no details returned). Empty-body 4xx/5xx (unknown route,
  405, auth challenges) get the envelope through StatusCodePages.
- Field validation codes: REQUIRED, INVALID_LENGTH, INVALID_FORMAT, INVALID_EMAIL,
  OUT_OF_RANGE, INVALID_VALUE, INVALID; UPPER_SNAKE codes set with WithErrorCode pass through.
- Validation: MediatR ValidationBehavior runs every FluentValidation validator before the
  handler and throws one ValidationException with all failures.
- Pagination: PaginationMeta { page, pageSize, totalCount, totalPages } in `meta`.
- Localization: en (default) and ar; culture from query string, cookie, then
  Accept-Language; Content-Language echoed; messages keyed by error code in
  Messages.resx / Messages.ar.resx.
- Correlation id: X-Correlation-Id accepted when it matches ^[A-Za-z0-9._-]{1,64}$, else a
  32-char hex id is generated; returned in the header and body and pushed to the log context.
- Logging: Serilog, text in Development, compact JSON elsewhere; one request event with
  method, path (no query), route, status, RequestId, UserId; headers and bodies never logged.
- Persistence: EF Core + Npgsql with snake_case naming; DatabaseOptions validated on start.
- JSON: camelCase, enums as strings.
```

---

## Acceptance criteria

```
- [ ] Backend foundation: PostgreSQL + EF Core, configuration, global exception handling, standard API response contract, correlation id, Serilog, health checks, OpenAPI, validation pipeline, API versioning.
- [ ] Localization: resource-based messages; `Accept-Language` honored; errors/validation localized.
- [ ] Soft delete and auditing conventions.
- [ ] Background jobs infrastructure.
- [ ] Docker, CI pipeline, environments, database migration strategy.
- [ ] `dotnet build` passes with zero warnings.
```

---

## Attachments

None.

---

## Dependencies

- **Blocked by / related ids:** none (first backend story).
- **Depends on code areas or other stories:** Phase 0 scaffold (`84f7bb1`).

## Extra notes (optional)

- Implementation plan §7 (vertical slices), §16 (endpoint conventions), §76 (pagination), §81 (Phase 1).
- Soft delete, auditing and background jobs were delivered later (Phase 2 `0f87e2d`, Phase 3 `65c74a3`); see the plan's deviations table.
- Follow-up `0936711` localizes every feature error code in en/ar.

## Technical hints (optional)

- Repo root: `customer-support-crm-api/`. .NET 10.

## Out of scope

- Squad plans treat tests/ as read-only. The platform tests that shipped in `c3c7815` are listed in the plan for reference only.
- Authentication and authorization (Phase 2).
- Frontend (see `plans/frontend/08-story-core-platform-shell.md`).
