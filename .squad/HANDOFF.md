# Session handoff (2026-09-30, second session)

Read this first in a new session. It replaces the old conversation.

## Standing directive from the owner

- Build all 12 features in `.squad/features/` on both the **backend and the frontend**. Work autonomously and **do not ask questions**.
- Use the **squad-kit** workflow: intake → plan (`.squad/plans/<feature>/NN-story-*.md`) → implement → update `00-overview.md` and `00-index.md`.
- **Ignore Docker and unit/e2e tests.** Verification is build-level only: `dotnet build` with 0 warnings, and `npx ng build` with 0 errors and 0 warnings.
- Commit and push each repo after each meaningful step. Use conventional commits and end each message with the `Co-Authored-By` trailer.

## Repositories

| Repo | Branch | State |
|------|--------|-------|
| `CRM` (this workspace, `.squad/`) | main | Frontend intakes (`.squad/stories/frontend/`) and **all plans 08–19** written |
| `customer-support-crm-api` | develop | Backend complete for all 12 features, pushed. `docs/endpoints.md` lists every endpoint |
| `customer-support-crm-web` | main | Stories 08 and 09 committed and pushed. Stories 10–19 are **uncommitted work in progress** in the working tree |

## Frontend status (`customer-support-crm-web`)

Conventions, ownership rules and folder/route/i18n table: **`.squad/plans/frontend/00-overview.md`**. Follow it.

| NN | Story | Folder | State |
|----|-------|--------|-------|
| 08 | Core platform shell | `core/`, `shared/` | Done, pushed |
| 09 | Authentication (staff + portal) | `features/auth`, `features/customer-portal/auth` | Done, pushed |
| 12 | Channels & live chat console | `features/channels` | Done (uncommitted), own files compile |
| 16 | AI assistant panel | `features/ai` | Done (uncommitted), own files compile |
| 10 | Customers UI | `features/customers` | Partial: stopped while writing the contacts component |
| 11 | Tickets UI | `features/tickets` | Partial: routes and agent picker not finished |
| 13 | Agent dashboard UI | `features/dashboard` | Nearly done: i18n key check and build pending |
| 14 | SLA & automation admin | `features/sla` | Partial: translations not written |
| 15 | Knowledge base UI | `features/knowledge-base` | Partial: categories page/dialog missing |
| 17 | Customer portal UI | `features/customer-portal` | Partial: routes done, contact page and more missing |
| 18 | Reports UI | `features/reports` | Partial: export button and shared styles missing |
| 19 | Administration UI | `features/administration` | Partial |

The partial stories were cut off by a usage limit, not by design problems. Their plan files are complete.

## Next steps

1. For each partial story, read its plan and finish the missing files. Do not rewrite existing work.
2. Run `cd customer-support-crm-web && npx ng build` and fix every error until it builds with 0 errors and 0 warnings.
3. Check integration points:
   - The AI panel links articles to `/knowledge-base/articles/{id}`. Make the KB routes match.
   - The quick-reply picker (`features/dashboard/quick-reply-picker.component.ts`) is used by tickets and chat.
   - en/ar JSON key sets must be identical per scope.
4. Commit each feature separately (`feat(<feature>): ...`) and push.
5. Mark stories Done in `.squad/plans/frontend/00-overview.md`, then commit and push the CRM repo.

## Known gaps (low priority)

- The staff live chat transcript is read from the linked ticket (`GET /tickets/{id}/messages`), so it needs `tickets.view`. There is no staff chat-messages endpoint.
- `ai.agent_assist_enabled` is not a public setting, so the AI panel only hides when it is explicitly "false". Otherwise the actions show `FEATURE_DISABLED`.
- en/ar resx entries are missing for some newer backend error codes; the English fallback messages are used.
