# Session handoff (2026-10-01, fourth session)

Read this first in a new session. It replaces the old conversation.

## Standing directive from the owner

- Build all 12 features in `.squad/features/` on both the **backend and the frontend**. Work autonomously and **do not ask questions**.
- Use the **squad-kit** workflow: intake → plan (`.squad/plans/<feature>/NN-story-*.md`) → implement → update `00-overview.md` and `00-index.md`.
- **Ignore Docker and unit/e2e tests.** Verification is build-level only: `dotnet build` with 0 warnings, and `npx ng build` with 0 errors and 0 warnings.
- Commit and push each repo after each meaningful step. Use conventional commits and end each message with the `Co-Authored-By` trailer.

## Repositories

| Repo | Branch | State |
|------|--------|-------|
| `CRM` (this workspace, `.squad/`) | main | Every feature has backend plans, a frontend plan and story intakes: plans 01–46, 46 intakes (36 feature stories + 10 bug stories). Index: `.squad/plans/00-index.md`; feature → plan matrix: `.squad/features/README.md` |
| `customer-support-crm-api` | develop | Backend complete for all 12 features, pushed (HEAD `dca992c`, BUG-01 to BUG-10 fixes; migration `AddTicketSlaPolicyForeignKey` applied to the local dev database on 2026-10-01). `docs/endpoints.md` lists every endpoint |
| `customer-support-crm-web` | main | Stories 08–19 all committed (one `feat(...)` commit per feature) and pushed; HEAD `06c817a` (BUG-08 KB category dialog). `npx ng build`: 0 errors, 0 warnings |

## Frontend status (`customer-support-crm-web`)

Conventions, ownership rules and folder/route/i18n table: **`.squad/plans/frontend/00-overview.md`**. Follow it.

| NN | Story | Folder | State |
|----|-------|--------|-------|
| 08 | Core platform shell | `core/`, `shared/` | Done, pushed |
| 09 | Authentication (staff + portal) | `features/auth`, `features/customer-portal/auth` | Done, pushed |
| 10 | Customers UI | `features/customers` | Done, pushed |
| 11 | Tickets UI | `features/tickets` | Done, pushed |
| 12 | Channels & live chat console | `features/channels` | Done, pushed |
| 13 | Agent dashboard UI | `features/dashboard` | Done, pushed |
| 14 | SLA & automation admin | `features/sla` | Done, pushed |
| 15 | Knowledge base UI | `features/knowledge-base` | Done, pushed |
| 16 | AI assistant panel | `features/ai` | Done, pushed |
| 17 | Customer portal UI | `features/customer-portal` | Done, pushed |
| 18 | Reports UI | `features/reports` | Done, pushed |
| 19 | Administration UI | `features/administration` | Done, pushed |

Integration points verified: `/knowledge-base/articles/:id` route exists for the AI panel links; `/help/articles/:slug` exists for portal chatbot links; the quick-reply picker is used by the ticket conversation and the chat console; en/ar key sets are identical for every scope.

## Squad workspace (fourth session)

- Backend features 01–09, 11, 12 had no plans. **As-built** plans and intakes were written for them, NN **20–36** (one folder per feature under `plans/` and `stories/`), plus **22** (system settings + audit export) in `security-and-administration`. Line numbers refer to the commit named at the top of each plan.
- Each plan has a *Deviations* table (spec vs code) and each `00-overview.md` has *Known gaps*. Treat those as the backlog of spec items not built.
- Every feature spec has a `Status` line linking its backend and frontend plans. `dotnet build` on api `2956767`: 0 warnings, 0 errors.

## Next steps

All 12 features are built on backend and frontend. Remaining work: the probable bugs below, then the spec gaps listed in each plan's deviations table (write a new intake per fix).

**QA fixes (NN 47–57, BUG-11 to BUG-21):** a manual browser QA pass on 2026-10-01 ([report](qa/2026-10-01-manual-qa-report.md)) produced 11 planned fix stories, each with an intake and a plan. Run them in the wave order of [plans/qa-2026-10-01-fix-roadmap.md](plans/qa-2026-10-01-fix-roadmap.md), starting with 49 and then 47. Status is in `plans/00-index.md` ("QA fixes").

## Backend bug stories (NN 37–46)

Each bug below has an intake; status and links in `.squad/plans/00-index.md` ("Bug fixes"). All ten are done (plans 37–46). The running local API must be restarted to pick them up.

## Probable backend bugs (found while writing plans 20–36)

- ~~`StaffHub.JoinConversation` has no permission or scope check~~ — **fixed** in api `d2563dc` (BUG-01, plan 37).
- ~~`GET /tasks?ticketId=` / `?customerId=` skips scope; unknown task links fail as 500~~ — **fixed** in api `7058b62` (BUG-02, plan 38).
- ~~`tickets.sla_policy_id` has no FK to `sla_policies`~~ — **fixed** in api `3433f13` (BUG-03, plan 39; new migration `AddTicketSlaPolicyForeignKey`).
- ~~`PUT /portal/me` returns the old name~~ — **fixed** in api `1ad5302` (BUG-04, plan 40). Staff and integration renames: **fixed** in api `dca992c` (BUG-10, plan 46).
- ~~`ORGANIZATION_UNIT_INACTIVE` is never thrown~~ — **fixed** in api `487e078` (BUG-06, plan 42; also added en messages for it and for `BRANCH_NOT_FOUND` / `DEPARTMENT_NOT_FOUND`).
- ~~Category cycles are possible~~ — **fixed** for ticket categories (api `cedce62`, BUG-07) and KB categories (api `e6bf4d7` + web `06c817a`, BUG-08).
- ~~Missing resx entries~~ — **fixed** in api `5eb50e6` (BUG-09, plan 45): `CHAT_CLOSED`, `NO_ACTIVE_BRANCH`, `OUTBOUND_MESSAGE_NOT_FOUND`, `WEBHOOK_DELIVERY_NOT_FOUND` added in en/ar. Codes that are only in the ar file (`TICKET_CLOSED`, `INVALID_STATUS_TRANSITION`, `CATEGORY_NOT_FOUND`, `INVALID_*`, …) have several English messages and are ar-only **by design**.
- ~~Revoking portal access does not end portal sessions~~ — **fixed** in api `31d6d5a` + `6587e7d` (BUG-05, plan 41; re-grant reactivates the revoked account).

## Known gaps (low priority)

- Customer portal: CSAT keeps a "Change rating" button (the backend accepts repeat feedback); the plan asked for read-only after rating.
- Error messages: codes with several/parameterized English messages (INVALID_TICKET, FILE_TOO_LARGE, ...) are Arabic-only in resx, so Arabic shows a generic message while English keeps the specific one. FluentValidation field messages keep the English property name in Arabic ("'Category Name' لا يجب أن يكون فارغاً").
- Admin integrations: scope/event label keys replace non-alphanumerics with `_` (`customers:read` → `admin.integrations.scopes.customers_read`).
- A live chat's first visitor message is the ticket description, so it is not part of the transcript (`GET /chat/conversations/{id}/messages`).

## Resolved in the third session

- KB article suggestions on ticket details (`GET /kb/suggestions`, match-any search).
- en/ar resx messages for every feature error code; feature-coded validation failures are localized.
- Staff chat transcript endpoint `GET /chat/conversations/{id}/messages` (`chat.handle`, no `tickets.view`).
- `GET /ai/status` honours `ai.agent_assist_enabled`, so the AI panel hides when agent assist is off.
- Public `GET /public/web-forms/categories`; the portal contact form has a category picker.
- Plural forms in `TranslationService` (`{ zero, one, two, few, many, other }` objects, `params.count`).
- `/knowledge-base/articles/:id` is the canonical editor URL (`/knowledge-base/:id` redirects).
- Ticket categories page shows the Arabic parent name; dashboard shows unread notifications as a KPI card; integration action errors use `admin.errors.*`.
- Dev: `ng serve` proxies `/api` and `/hubs` (`proxy.conf.json`) so the SameSite=Strict refresh cookie survives reloads.
