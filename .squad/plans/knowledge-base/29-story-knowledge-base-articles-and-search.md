# Story 29 — Knowledge base articles, categories, search and public help center (Story: KB-01)

> As-built plan: written after implementation in `customer-support-crm-api` commit `0027cd1` (feat: add knowledge base), with follow-ups `2de665a` (search moved onto `SqlRunner`), `678ea67` (migration `AddSupportOperations` creates the tables and the GIN index) and `889dfcf` (match-any for ticket suggestions). Line numbers refer to `develop` HEAD `2956767`. For `KnowledgeBaseSlices.cs`, `KnowledgeArticle.cs`, `KnowledgeBaseContracts.cs` and `KnowledgeBaseConfiguration.cs` they are identical to `0027cd1` (`889dfcf` changed one line in place).

## Prerequisites

- Security & Administration Phase 2 completed: [../security-and-administration/00-overview.md](../security-and-administration/00-overview.md) — permission policies (policy name = code), `ICurrentUser`, `IAuditTrail`, `GlobalExceptionHandler`.
- Platform route groups in `src/CustomerSupportCrm.Api/Endpoints/EndpointExtensions.cs`: staff `/api/v1` (`IEndpoint`) and anonymous `/api/v1/public` (`IPublicEndpoint`).
- Ticket Management (02) for `db.Tickets`, `IAccessScopeProvider` / `WhereInScope` and `TicketErrors.TicketNotFound` (used by suggestions).
- All paths below are relative to the **`customer-support-crm-api/`** repo root.

---

## Story Goal

1. KB editors manage hierarchical, bilingual-named categories.
2. KB editors write entries (FAQ, article, guide, solution) in English or Arabic as drafts, link translations, and publish, unpublish, archive, restore or delete them through domain actions.
3. Staff list and full-text search articles with filters; agents get article suggestions for a ticket.
4. Anonymous visitors browse public categories and published public articles, read an article by slug (view counted) and send "Was this helpful?" feedback.

**Deviations from the intake (the code is authoritative):**

| Intake / feature spec | As built in `0027cd1` (+ follow-ups) |
|---|---|
| Slices in folders `Articles/`, `Categories/`, `Faqs/`, `Search/`, `Feedback/` | One file `Features/KnowledgeBase/KnowledgeBaseSlices.cs`; FAQs are articles with `Type = Faq`, no separate FAQ slice |
| Localized title/body per language with fallback rules | One language per article row (`en`/`ar`), translations linked by `TranslationOfId`; **no fallback** — the response lists sibling translations only |
| PostgreSQL full-text with Arabic + English configs | Single `'simple'` config (no stemming) with prefix terms; one GIN expression index |
| Article versioning | **Not built** — only `CreatedAt/By`, `UpdatedAt/By`, `PublishedAt/By` |
| `ArticleFeedback` entity | **Not built** — `HelpfulCount` / `NotHelpfulCount` counters on the article; anonymous, not de-duplicated |
| Articles linked to tickets and ticket categories | **Not built** — KB categories are separate from ticket categories; suggestions are computed from the ticket subject only |
| Help articles with images and attachments | **Not built** — Markdown body only |
| Separate `Create`/`Update`/`Publish`/... commands | `SaveKnowledgeArticleCommand(Guid? ArticleId, …)` for create+update; `ChangeKnowledgeArticleCommand(ArticleId, Action)` for publish/unpublish/archive/restore/delete |
| Archive from Published only | `Archive()` works from any state; `Restore()` on a non-archived article is a no-op |

**Not in scope:** article attachments, versions, per-visitor feedback. **Do not touch** `docker-compose.yml`, `deploy/`, `.github/` or anything under `tests/`.

---

## Context — Read These Files First

1. `src/CustomerSupportCrm.Domain/KnowledgeBase/KnowledgeArticle.cs` — enums `ArticleType` (6–12), `ArticleStatus` (14–19), `ArticleVisibility` (21–28); `KnowledgeArticle : AggregateRoot<Guid>, IAuditableEntity, ISoftDeletable` (34–223): limits and codes (36–42), `CreateDraft` (112–127), `Slugify` (130–136), `Edit` (138–166), `Publish` (168–183), `Unpublish` (185–193), `Archive` (195), `Restore` (197–203), `RecordView` (205), `RecordFeedback` (207–217), `IsVisibleToCustomers` (219). `KnowledgeCategory : Entity<Guid>, IAuditableEntity` (226–280), `Update` (265–279).
2. `src/CustomerSupportCrm.Application/Features/KnowledgeBase/KnowledgeBaseSlices.cs` — `KnowledgeErrors` (23–29), `KnowledgeQueries` (31–97), categories (99–166), staff articles (168–322), suggestions (324–341), `KnowledgeBaseEndpoints : IEndpoint` (343–428), public queries (430–499), `PublicKnowledgeBaseEndpoints : IPublicEndpoint` (501–528).
3. `src/CustomerSupportCrm.Contracts/KnowledgeBase/KnowledgeBaseContracts.cs` — lines 1–63.
4. `src/CustomerSupportCrm.Application/Abstractions/Search/IKnowledgeSearch.cs` — `KnowledgeSearchRequest` (7–13, `MatchAny` from `889dfcf`), `IKnowledgeSearch` (19–22).
5. `src/CustomerSupportCrm.Infrastructure/Search/PostgresKnowledgeSearch.cs` — `SearchVector` (15), SQL (17–28), term splitting (34–40), `&` vs `|` join (48), parameters via `SqlRunner` (50–60).
6. `src/CustomerSupportCrm.Infrastructure/Persistence/Configurations/KnowledgeBaseConfiguration.cs` — articles (7–31: unique slug 16, enum-as-string 19–21, `text[]` tags 23, indexes 25–27, category FK `SetNull` 28, `xmin` 29); categories (33–45: parent FK `Restrict` 43).
7. `src/CustomerSupportCrm.Infrastructure/Persistence/Migrations/20260930104602_AddSupportOperations.cs` — `knowledge_categories` (181), `knowledge_articles` (373), indexes (1118–1142), GIN index `ix_knowledge_articles_search` (1272–1275), dropped in `Down` (1281).
8. `src/CustomerSupportCrm.Domain/Roles/Permissions.cs` — `KnowledgeView`/`KnowledgeManage`/`KnowledgePublish` (24–26); agents get `kb.view` (69), managers `kb.manage` + `kb.publish` (78).
9. `src/CustomerSupportCrm.Api/Endpoints/EndpointExtensions.cs` — staff group (18–22), anonymous `/api/v1/public` group (30–34).
10. `src/CustomerSupportCrm.Application/Abstractions/Http/RateLimitPolicies.cs` — `Public` (9); policy registered in `src/CustomerSupportCrm.Api/Configuration/SecurityExtensions.cs` (`PublicRateLimitOptions` 31, policy 141; `RateLimiting:Public` 30/60 s in `appsettings.json`).
11. `src/CustomerSupportCrm.Infrastructure/Persistence/Interceptors/AuditableEntityInterceptor.cs` — line 38–41 turns `Remove` of an `ISoftDeletable` into `IsDeleted = true`; global filter in `ApplicationDbContext.cs` 84–95.

---

## Backend Tasks

### 1 — Domain

Create file: `src/CustomerSupportCrm.Domain/KnowledgeBase/KnowledgeArticle.cs` (as listed above).

- `KnowledgeArticle` fields: `Title` (≤ 300), `Slug` (≤ 200, unique), `Summary` (≤ 1000), `Body` (Markdown, ≤ 200 000), `Type`, `Language` (`en`/`ar`), `CategoryId`, `Tags` (trimmed, 1–50 chars, distinct case-insensitive, max 20), `Visibility`, `Status`, `TranslationOfId`, `PublishedAt/By`, `ViewCount`, `HelpfulCount`, `NotHelpfulCount`, audit + soft-delete fields.
- `Edit` throws `DomainException(INVALID_ARTICLE_STATE)` when archived and `INVALID_ARTICLE` for bad title/slug/body/summary/language.
- `Publish(now, publishedBy)` refuses archived or empty-body articles (`INVALID_ARTICLE_STATE`); `Unpublish` only from `Published`.
- `Slugify`: lower-case invariant, `[^\p{L}\p{N}]+` → `-`, trimmed, cut to 200 — Arabic letters stay.
- `KnowledgeCategory.Update(name, nameAr, description, parentId, sortOrder, isPublic)`; name ≤ 150; `parentId == Id` → `INVALID_KB_CATEGORY`. Only direct self-parenting is blocked (no cycle check deeper down).

### 2 — Persistence

- `src/CustomerSupportCrm.Application/Abstractions/Persistence/IApplicationDbContext.KnowledgeBase.cs` — `DbSet<KnowledgeArticle> KnowledgeArticles`, `DbSet<KnowledgeCategory> KnowledgeCategories`; implemented in `src/CustomerSupportCrm.Infrastructure/Persistence/ApplicationDbContext.KnowledgeBase.cs` (8, 10).
- `KnowledgeBaseConfiguration.cs` — tables `knowledge_articles`, `knowledge_categories`.
- Migration: none in `0027cd1`; the tables, indexes and the raw-SQL GIN index ship in `AddSupportOperations` (`678ea67`).

### 3 — Search abstraction

- `IKnowledgeSearch.SearchAsync(KnowledgeSearchRequest, ct)` returns article ids by relevance.
- `PostgresKnowledgeSearch(SqlRunner sql)` registered in `src/CustomerSupportCrm.Infrastructure/DependencyInjection.cs` line 84 (`SqlRunner` line 81). Splits the text on whitespace/punctuation, keeps letters/digits only (no tsquery injection), max 10 terms, each `term:*`; joins with ` & ` (all terms) or ` | ` when `MatchAny`. SQL excludes deleted and archived rows, applies `publishedOnly` / `publicOnly` / `language`, orders by `ts_rank` then `view_count`, `LIMIT` clamped 1–100.

### 4 — Categories

- `ListKnowledgeCategoriesQuery(bool PublicOnly)` (101–113): ordered by `SortOrder`, `Name`; `ArticleCount` counts all articles (staff) or published public ones (public).
- `SaveKnowledgeCategoryCommand(Guid? CategoryId, KnowledgeCategoryRequest)` + validator (115–148): 404 `KB_CATEGORY_NOT_FOUND` on update of an unknown id; returns `ArticleCount = 0`. The parent id is not checked for existence (FK violation → 409 `CONFLICT`).
- `DeleteKnowledgeCategoryCommand` (150–166): children → 409 `KB_CATEGORY_IN_USE`; articles keep existing with `CategoryId = null` (FK `SetNull`).

### 5 — Staff articles

- `ListKnowledgeArticlesQuery` + validator (170–192): `ValidPage`, `ValidPageSize`, `Search` ≤ 200, enum names for `Status`/`Type`/`Visibility` (`INVALID_VALUE`), `Language.OneOf("en","ar")`.
- `KnowledgeQueries.SearchAsync` (72–96): no text → newest first (`PublishedAt ?? CreatedAt`, then `Id`) with `ToPagedResultAsync`; with text → top 100 ids from `IKnowledgeSearch` (`PublicOnly = PublishedOnly = publicOnly`), filtered, kept in rank order, paged in memory.
- `GetKnowledgeArticleQuery` (232–238) → `KnowledgeQueries.GetResponseAsync` (54–69): article + translation siblings (`TranslationOfId` group) + category name.
- `SaveKnowledgeArticleCommand` + validator + handler (240–288): slug from slug or title; uniqueness checked with `IgnoreQueryFilters()` → 409 `ARTICLE_SLUG_TAKEN`; unknown category → 404 `KB_CATEGORY_NOT_FOUND`; create → `CreateDraft`, update → `Edit`; audit `kb.article_created` / `kb.article_updated` (`KnowledgeArticle`, `{Title, Slug, Language}`).
- `ChangeKnowledgeArticleCommand(ArticleId, Action)` (291–322): `publish` (with `currentUser.UserId`), `unpublish`, `archive`, `restore`, `delete` (`Remove` → soft delete); audit `kb.article_{action}`; returns the article (null for delete).

### 6 — Suggestions

`SuggestArticlesQuery(TicketId, Limit = 5)` (325–341): ticket loaded inside the caller's access scope (`WhereInScope`) or 404 `TICKET_NOT_FOUND`; searches the **subject** with `PublishedOnly`, internal included, limit 1–20, `MatchAny: true` (line 337, `889dfcf`). The same match-any flag is used by `KnowledgeContext.LoadAsync` in `Features/Ai/AiSlices.cs` (line 259).

### 7 — Public help center

- `ListPublicArticlesQuery(Page = 1, PageSize = 20, Search, Type, Language, CategoryId)` + validator (432–471): page size 1–50; base filter `Published && Public`.
- `GetPublicArticleQuery(Slug)` (473–486): published public only, `RecordView()` + save, translations filtered to published public.
- `SubmitArticleFeedbackCommand(ArticleId, Helpful)` (488–499): requires `Published` (visibility not checked), increments a counter.

### 8 — Endpoints

`KnowledgeBaseEndpoints` (343–428), group `/kb`, tag "Knowledge base"; publish/unpublish/archive/restore mapped in a loop (398–411) with names `PublishKnowledgeArticle` etc. `PublicKnowledgeBaseEndpoints` (501–528) on `/api/v1/public/kb`; only feedback has `.RequireRateLimiting(RateLimitPolicies.Public)` (524). Every response uses `ApiResults` (`Ok`, `Created`, `Paged`, `Success`).

### 9 — Localization and docs

- `Messages.resx`: `ARTICLE_NOT_FOUND` (198), `ARTICLE_SLUG_TAKEN` (201), `KB_CATEGORY_NOT_FOUND` (204), `KB_CATEGORY_IN_USE` (207). `Messages.ar.resx`: the same plus `INVALID_ARTICLE`, `INVALID_ARTICLE_STATE`, `INVALID_KB_CATEGORY` (261–281). Added in `0936711`; English domain-code messages come from the throw site.
- `docs/endpoints.md` — "Knowledge base" (137–146) and Public (203–204). The staff table does not list `POST /kb/articles/{id}/restore`.

No Application DI changes: handlers, validators and endpoints are found by assembly scanning (`src/CustomerSupportCrm.Application/DependencyInjection.cs` line 63 covers `IPublicEndpoint`).

---

## Edge Cases & Failure Modes

- **Slug reuse after delete** — the uniqueness check ignores the soft-delete filter, so a deleted article's slug stays taken (409 `ARTICLE_SLUG_TAKEN`). A parallel insert hits the unique index → 409 `CONFLICT`.
- **Arabic slugs** — kept as Unicode; `/public/kb/articles/{slug}` must be URL-encoded by clients.
- **Search total capped** — with search text, at most 100 ranked ids are fetched, so `totalCount` ≤ 100.
- **Empty / punctuation-only search** — no terms → empty result (not "all articles").
- **Multi-word search box** — all terms must match (prefix); ticket suggestions match any term (`889dfcf`).
- **Archived article** — excluded from search; edit/publish → 422 `INVALID_ARTICLE_STATE` until restored.
- **Unpublish a draft** → 422 `INVALID_ARTICLE_STATE`.
- **Internal article via public routes** — list/get return 404/omit it; feedback by id on a published internal article is accepted (visibility not checked).
- **Feedback spam** — counters only, protected by the 30/60 s per-IP `public` rate limit.
- **Concurrent edits** — `xmin` token → 409 `CONFLICT`.
- **Missing permission / anonymous on staff routes** — 403 / 401.

---

## Test Plan

Test projects are **out of scope** (`tests/` must not be modified). No tests are added, changed or removed by this story. There are **no existing tests** in `2956767` that cover the knowledge base (no file under `tests/` references `/kb`, `KnowledgeArticle` or `IKnowledgeSearch`).

---

## Verification Steps

1. **Backend builds:** in `customer-support-crm-api/` run `dotnet build` — 0 warnings, 0 errors.
2. **Run:** apply migrations (`AddSupportOperations` creates the KB tables and `ix_knowledge_articles_search`), start the API, sign in as an admin and export `TOKEN`.
3. **Category:** `curl -X POST https://localhost:<port>/api/v1/kb/categories -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" -d '{"name":"Accounts","nameAr":"الحسابات","sortOrder":1,"isPublic":true}'` → 201.
4. **Article draft:** `POST /api/v1/kb/articles` with `{"title":"Reset your password","body":"Open settings…","type":"Faq","language":"en","visibility":"Public","categoryId":"<id>"}` → 201, `status: "Draft"`, `slug: "reset-your-password"`. Repeat → 409 `ARTICLE_SLUG_TAKEN`.
5. **Publish:** `POST /api/v1/kb/articles/{id}/publish` → `status: "Published"`; `POST …/unpublish` twice → second is 422 `INVALID_ARTICLE_STATE`.
6. **Search:** `GET /api/v1/kb/articles?search=reset%20pass` → the article; `GET /api/v1/kb/suggestions?ticketId=<ticket with subject "Cannot reset my password">` → the article (match any).
7. **Public:** `curl https://localhost:<port>/api/v1/public/kb/articles?search=reset` (no token) → the article; `GET /api/v1/public/kb/articles/reset-your-password` → `viewCount` increases; `POST /api/v1/public/kb/articles/{id}/feedback -d '{"helpful":true}'` → 200; 31 rapid calls → 429.
8. **Archive / restore / delete:** `POST …/archive` → article gone from public list; `POST …/restore` → `Draft`; `DELETE /api/v1/kb/articles/{id}` → 200, then GET → 404 `ARTICLE_NOT_FOUND`.
9. **Category in use:** create a child category, `DELETE` the parent → 409 `KB_CATEGORY_IN_USE`.
10. **Permissions:** an Agent token → `POST /api/v1/kb/articles` 403; `GET /api/v1/kb/articles` 200.
11. **Localization:** a failing call with `Accept-Language: ar` → Arabic message.

---

## Done Criteria

- [x] Lifecycle `Draft → Published → Archived` (+ unpublish, restore, soft delete) through domain methods, not a status setter.
- [x] Visibility `Public` vs `Internal`; public routes return only published public articles.
- [ ] Localized title/body per language with fallback rules — per-language rows linked by `TranslationOfId`; no fallback logic (see deviations).
- [x] Hierarchical categories (`ParentId`, delete blocked while children exist) with Arabic names and a public flag.
- [x] Full-text search behind `IKnowledgeSearch` (PostgreSQL, GIN index). Arabic/English-specific configs not used (`'simple'`).
- [ ] Article versioning — not built; last-updated tracking (`UpdatedAt/By`, `PublishedAt/By`) only.
- [x] "Was this helpful?" counters and view counts (no per-visitor `ArticleFeedback` record).
- [ ] Articles linked to tickets and ticket categories — not built; ticket suggestions by subject search only.
- [x] Staff mutations audited (`kb.article_*`); error codes localized in en/ar.
- [x] Nothing changed in `tests/`, `docker-compose.yml`, `deploy/` or `.github/` by this story.
- [x] `dotnet build` passes with zero warnings.

**STOP HERE. Report to the user and wait for confirmation before proceeding to the next story.**
