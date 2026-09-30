# Session handoff (2026-09-30)

Read this first in a new session. It replaces the old conversation.

## Standing directive from the owner

- Build all 12 features in `.squad/features/` on both the **backend and the frontend**. Work autonomously and **do not ask questions**. Report only when everything is done.
- Use the **squad-kit** workflow (`squad` CLI is installed globally):
  1. intake: `squad new-story <feature-slug>`
  2. plan: write the `NN-story-*.md` plan under `.squad/plans/<feature>/`
  3. implement the plan
  4. update `.squad/plans/00-index.md` and each feature's `00-overview.md`
- Ignore Docker and tests. Verification is build-level only: `dotnet build` with 0 warnings, and `ng build`.
- Commit and push each repo after each meaningful step. Use conventional commits and end each message with the `Co-Authored-By` trailer.

## Repositories

| Repo | Branch | State |
|------|--------|-------|
| `CRM` (this workspace, `.squad/`) | main | planning only |
| `customer-support-crm-api` | develop | **Backend complete for all 12 features**, pushed (678ea67) |
| `customer-support-crm-web` | main | Angular 22 scaffold only (`core/`, `shared/`, `features/` folders empty) |

## Backend summary (what the frontend talks to)

- .NET 10 minimal APIs, vertical slices in `Application/Features/*` (single-file slices such as `TicketCommandSlices.cs`). MediatR, FluentValidation, EF Core + PostgreSQL.
- Every response uses the `ApiResponse` envelope `{ success, data, error: { code, message, details } , meta }`. Paged lists come through `ApiResults.Paged`. See `customer-support-crm-api/docs/api-contract.md` and the Swagger UI at `/swagger`.
- Route groups:
  - `/api/v1` for staff (JWT, permission policies named by code, e.g. `tickets.manage`)
  - `/api/v1/portal` for customers (customer JWT, 8h, no refresh)
  - `/api/v1/public` for anonymous callers (branding, features, KB, web form, chat start, chatbot)
  - `/api/v1/external` for API-key clients
- Staff auth:
  - An access token plus an httpOnly refresh cookie `crm_refresh`. The refresh call must send the header `X-CSRF-Protection: 1` with credentials.
  - Permissions arrive as `permission` claims and from `GET /api/v1/me`.
- Realtime runs over SignalR: `/hubs/staff` (notifications, ticket updates) and `/hubs/chat` (live chat). The token is passed via the `access_token` query parameter.
- Localization: en/ar via `Accept-Language`, and error messages come back localized.
- Contract DTOs live in `src/CustomerSupportCrm.Contracts`, plus response records declared inside the slice files. Mirror them in TypeScript.

## Remaining work: the frontend (all 12 features)

Suggested squad feature folder is `frontend`, with one story per row (global NN continues after 07):

1. Core platform:
   - Angular Material, a typed ApiClient and envelope handling
   - Interceptors: auth, single-flight refresh, correlation id, Accept-Language, errors
   - i18n en/ar with RTL, and branding theming from `/public/branding`
   - Permission service, directive and guards
   - Responsive staff shell with a SignalR notification bell
2. Auth: staff login/logout/refresh, and portal login/register/verify.
3. Customers (feature 01): list/search, profile, contacts, notes, timeline.
4. Tickets (02): list/filters, detail, messages, attachments, assignment, status, history.
5. Channels and live chat console (03): templates, outbox, channel admin, agent chat console.
6. Agent dashboard (04): widgets, tasks, reminders, quick replies.
7. SLA and automation admin (05).
8. Knowledge base (06): staff editor, public FAQ.
9. AI panels (07): ticket summary, suggested reply, categorize, solutions.
10. Customer portal (08):
    - Tickets and feedback, FAQ
    - Chatbot, chat widget, web form
11. Reports (09): charts and CSV export.
12. Administration (10, 11, 12):
    - Users, roles, branches, departments, audit log and export
    - Settings toggles, API keys, webhooks

## Known gaps (optional, low priority)

- The backend docs (`docs/architecture.md`, `api-contract.md`, `security.md`) describe Phase 2 only. Extend them with the feature endpoints.
- en/ar resx entries are missing for some of the newer error codes; the English fallback messages are used for now.
