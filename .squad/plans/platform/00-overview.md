# Platform — Phase 1 Backend Platform and Phase 3 Organization Context

Feature spec: [../../features/12-platform.md](../../features/12-platform.md) (organization/branch/department parts are shared with [../../features/10-security-and-administration.md](../../features/10-security-and-administration.md)).
Repo: `customer-support-crm-api`. Out of scope for every story: Docker, deployment, CI/CD, and files under `tests/`.
Frontend counterpart: [../frontend/08-story-core-platform-shell.md](../frontend/08-story-core-platform-shell.md) (interceptors, i18n/RTL, branding theme loader, notifications) and [../frontend/19-story-administration-ui.md](../frontend/19-story-administration-ui.md) (organization, branches and departments screens).

| NN | Tracker id | Plan file | Title | Depends on | Status |
|----|-----------|-----------|-------|-----------|--------|
| 20 | PL-01 | [20-story-backend-foundation.md](20-story-backend-foundation.md) | Backend foundation (API platform) | Phase 0 scaffold | Done |
| 21 | PL-02 | [21-story-organization-context-and-notifications.md](21-story-organization-context-and-notifications.md) | Organization context, branding and notifications | 20, Phase 2 (01–07) | Done |
| 42 | BUG-06 | [42-story-inactive-organization-units.md](42-story-inactive-organization-units.md) | Reject assignment to inactive branches and departments | 21 | Done (`487e078`) |
| 45 | BUG-09 | [45-story-missing-error-messages.md](45-story-missing-error-messages.md) | Add missing en/ar resx messages for error codes | 20, 42–44 | Done (`5eb50e6`) |
| 57 | BUG-21 | [57-story-validation-message-quality.md](57-story-validation-message-quality.md) | Specific, localized validation messages, inline on the customer and ticket forms (QA L3, L4, L7, L8) | 45, 48 | To do |
| 61 | BUG-25 | [61-story-upload-response-uploader-name.md](61-story-upload-response-uploader-name.md) | Upload responses include the uploader name (QA round 2, N5) | 20 | Done (`91ec854`) |

Both plans are **as-built**: they were written after the code shipped and list where the code differs from the intake.

- Story 20 was implemented in `c3c7815` (feat: add Phase 1 API foundation) on top of the scaffold `84f7bb1`. Follow-up `0936711` added en/ar messages for every feature error code.
- Story 21 was implemented in `65c74a3` (feat: add platform services and organization context (phase 3)). Its tables were created by migration `20260930104602_AddSupportOperations` in `678ea67`.
- Phase 2 (identity, roles, audit) sits between them: [../security-and-administration/00-overview.md](../security-and-administration/00-overview.md). System settings and audit export are Story 22 in that folder.

Known gaps and drift:

- **No API versioning beyond the `/api/v1` URL prefix.** No `Asp.Versioning` package and no version negotiation.
- **Phase 1 shipped without some foundation pieces the spec lists.** Paging helpers (`PagedResult`, `ToPagedResultAsync`, `CommonRules`) and `ApiResults.Success()` came in `0f87e2d`. Soft delete and the background job host came in `65c74a3`. Docker, CI and `deploy/` are still the `84f7bb1` scaffold files.
- **`65c74a3` has no migration.** The `organizations`, `branches`, `departments`, `user_scopes` and `notifications` tables (and every feature table from 01–09) first exist in `678ea67`. At `65c74a3`…`a67763f`, database initialization cannot work: the model has tables that no migration creates, and `SeedOrganizationAsync` writes to `organizations`.
- **`65c74a3` added no resx keys.** Organization, branch, department, scope and file-upload codes returned English fallback messages until `0936711`.
- **Branding:** one organization per deployment. Emails use only the name and primary color: no logo, no accent color. There is no `Features/Branding` or `Features/Localization` slice, and languages are fixed to en/ar in code.
- **Branches/departments:** there is no GET by id, no delete (deactivate only, by design), and departments are listed only nested in `GET /branches`. ~~`ORGANIZATION_UNIT_INACTIVE` is defined but never thrown, so deactivated units can still be assigned to users and data.~~ Fixed by story 42 (`487e078`). Customers and tickets already refused inactive units, but with a 404; user scopes, categories, SLA policies and assignment rules did not check at all. [intake](../../stories/platform/inactive-organization-units/intake.md)
- **Permissions:** profile, branding and logo writes need `settings.manage`, not `organization.manage`. `GET /organization`, `GET /branches` and `GET /users/lookup` need only a staff token.
- **No tests** cover organization, branches, departments, user scopes, access scope, file uploads or notifications.
