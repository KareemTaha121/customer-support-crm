# Story 15 — Knowledge base UI (Story: FE-08)

## Prerequisites

- Story 08 completed: [08-story-core-platform-shell.md](08-story-core-platform-shell.md) (`ApiService`, `ApiError`, i18n, permissions, shared states, `ConfirmService`, form helpers).
- Story 09 completed: [09-story-authentication-staff-and-portal.md](09-story-authentication-staff-and-portal.md) (staff session, portal layout look).
- Conventions: [00-overview.md](00-overview.md). Ownership: only `src/app/features/knowledge-base/**` and `public/i18n/kb/{en,ar}.json`.

---

## Story Goal

1. Staff with `kb.view` browse knowledge base articles at `/knowledge-base` (search, category/status/type/language/visibility filters, server paging).
2. Staff with `kb.manage` create and edit articles (title, slug, summary, Markdown body with preview, type, language, category, tags, visibility, translation link), archive/restore and delete (confirm); `kb.publish` is required to publish/unpublish.
3. Staff with `kb.manage` manage categories (name, Arabic name, description, parent, sort order, public flag) in a page with a dialog.
4. Anonymous visitors use the public help center at `/help`: home with search and categories, a category article list, an article page by slug with safe Markdown rendering and "Was this helpful?" feedback, in its own public layout (brand header, language switcher, link to `/portal`). en/ar with RTL.

Not in scope: ticket-side article suggestions (`/kb/suggestions`, used by Story 11/16), tests, backend changes.

---

## Context — Read These Files First

1. `customer-support-crm-api/src/CustomerSupportCrm.Application/Features/KnowledgeBase/KnowledgeBaseSlices.cs`
   - lines 23–29: error codes `ARTICLE_NOT_FOUND`, `KB_CATEGORY_NOT_FOUND`, `ARTICLE_SLUG_TAKEN`, `KB_CATEGORY_IN_USE`.
   - lines 101–166: categories list/save/delete (delete fails with `KB_CATEGORY_IN_USE` when sub-categories exist).
   - lines 170–230: `GET /kb/articles` filters `page`, `pageSize`, `search`, `status`, `type`, `language`, `visibility`, `categoryId`.
   - lines 240–288: save validator (title ≤ 300, slug ≤ 200 optional, summary ≤ 1000, body required ≤ 200 000, type/visibility enum names, language `en|ar`); `ARTICLE_SLUG_TAKEN` conflict.
   - lines 290–322 and 398–420: `POST /kb/articles/{id}/publish|unpublish` (`kb.publish`), `archive|restore` (`kb.manage`), `DELETE /kb/articles/{id}` (`kb.manage`).
   - lines 343–428: staff endpoint map (`/kb/categories` GET `kb.view`, POST/PUT/DELETE `kb.manage`; `/kb/articles` GET `kb.view`, POST/PUT `kb.manage`).
   - lines 432–528: public `/public/kb/categories`, `/public/kb/articles` (pageSize ≤ 50), `/public/kb/articles/{slug}` (records a view), `POST /public/kb/articles/{id}/feedback` (rate limited).
2. `customer-support-crm-api/src/CustomerSupportCrm.Contracts/KnowledgeBase/KnowledgeBaseContracts.cs` (lines 1–63) — request/response records mirrored in `knowledge-base.models.ts`.
3. `customer-support-crm-api/src/CustomerSupportCrm.Domain/KnowledgeBase/KnowledgeArticle.cs`
   - lines 6–28: `ArticleType` (Faq, Article, Guide, Solution), `ArticleStatus` (Draft, Published, Archived), `ArticleVisibility` (Public, Internal).
   - line 70: body is **Markdown**; lines 138–143: archived articles cannot be edited (restore first); lines 168–193: publish needs a body, unpublish only from Published.
   - lines 225–280: `KnowledgeCategory` (`NameAr`, `ParentId`, `IsPublic`, name ≤ 150).
4. Web core: `src/app/core/http/api.service.ts` (`getPaged`, `silent`, `anonymous`), `core/http/api-error.ts`, `shared/form-errors.ts` (`applyServerErrors`), `core/interceptors/error.interceptor.ts` (`describeError`, line 33), `core/guards/auth.guards.ts` (`requirePermission`), `core/branding/branding.service.ts`, `core/layout/language-switcher.component.ts`, `features/customer-portal/portal-shell.component.ts` (public layout look), `features/auth/profile.page.ts` (form style).
5. `src/app/app.routes.ts` — `/help` lazy-loads `HELP_CENTER_ROUTES` outside the staff shell; `/knowledge-base` loads `KNOWLEDGE_BASE_ROUTES` inside it. `app.config.ts` enables `withComponentInputBinding()` (route params → `input()`).

---

## Frontend Tasks

### 1 — Models, API, Markdown (`src/app/features/knowledge-base/`)

- `knowledge-base.models.ts`: `KbCategory`, `KbCategoryRequest`, `KbArticleListItem`, `KbArticle`, `KbArticleRequest`, `KbArticleTranslation`, `KbArticleQuery`, enum arrays `ARTICLE_TYPES`, `ARTICLE_STATUSES`, `ARTICLE_VISIBILITIES`, `KB_LANGUAGES`, `KbErrorCodes`, `categoryLabel(category, lang)`.
- `knowledge-base.api.ts` (`providedIn: 'root'`): staff `categories/saveCategory/deleteCategory`, `articles/article/saveArticle/changeStatus(id, action)/deleteArticle`; public `publicCategories/publicArticles/publicArticle(slug)/feedback(id, helpful)` (all `anonymous: true`).
- `markdown.ts`: minimal Markdown → HTML that **escapes all input first**, then supports headings, paragraphs, bold/italic, inline code, fenced code, lists, blockquotes, horizontal rules and links/images with `http(s)`, `mailto:` or relative URLs only. `kb-markdown.component.ts` binds the result with `[innerHTML]` (Angular's sanitizer runs as a second layer) and sets `dir` from the article language.
- `kb-server-errors.ts`: strips the `article.` / `category.` prefix from server field paths before `applyServerErrors`.

### 2 — Staff pages (`staff/`)

- `article-list.page.ts`: toolbar (debounced search, category, status, type, language, visibility), `mat-table` + `mat-paginator`, row click opens the editor, row menu: publish/unpublish (`kb.publish`), archive/restore, delete with confirm (`kb.manage`). Header actions: new article, categories (`kb.manage`).
- `article-editor.page.ts`: `/knowledge-base/new` (optional `?translationOf=&language=`) and `/knowledge-base/:id`. Form with `{ silent: true }` + server errors (`ARTICLE_SLUG_TAKEN` → slug field). Write/Preview toggle for the body. Side panel: status pill, stats, translations list + "Add translation", actions publish/unpublish (`kb.publish`, disabled while unsaved changes), archive/restore, delete. Read-only when the user lacks `kb.manage` or the article is archived.
- `categories.page.ts` + `category-dialog.component.ts`: table of categories, add/edit dialog, delete confirm (`KB_CATEGORY_IN_USE` shows the server message).

### 3 — Public help center (`help/`)

- `help-layout.component.ts`: brand toolbar (`BrandingService` logo/name → `/help`), language switcher, link to `/portal`; centered content.
- `help-home.page.ts`: hero search (`?q=` in the URL), results list when searching; otherwise category cards (public categories with counts) and latest articles.
- `help-category.page.ts`: `/help/categories/:id` — category title/description and paged article list.
- `help-article.page.ts`: `/help/articles/:slug` — title, category link, date, rendered body, other-language links, feedback buttons (once per article, remembered in `localStorage` best-effort; errors shown inline).
- Shared `help-article-list.component.ts` for article result rows.

### 4 — Routes and translations

- `knowledge-base.routes.ts`: `KNOWLEDGE_BASE_ROUTES` (root: `requirePermission(kb.view)`, `resolve: { i18n: translationResolver('kb') }`; children `''`, `new` + `categories` with `kb.manage`, `:id`); `HELP_CENTER_ROUTES` (root: `HelpLayoutComponent`, same resolver, children `''`, `categories/:id`, `articles/:slug`, `**` → home).
- `public/i18n/kb/en.json` and `ar.json`: identical keys, real Arabic, enum labels `kb.type.*`, `kb.status.*`, `kb.visibility.*`, `kb.language.*`.

---

## Edge Cases & Failure Modes

- Unknown slug → `ARTICLE_NOT_FOUND` (404) → help article page shows a not-found empty state with a link home (request is `silent`).
- Archived article → editor read-only with a "restore to edit" notice (domain rejects edits, `KnowledgeArticle.cs` lines 140–143).
- Publish with unsaved changes → button disabled; publish with empty body → server `INVALID_ARTICLE_STATE` message via global snackbar.
- Slug taken → slug field error; slug empty → server derives it from the title.
- Feedback rate limited (429) → inline localized error; double submit prevented.
- Untrusted Markdown (`<script>`, `javascript:` links) → escaped/dropped by `markdown.ts`, then sanitized again by Angular.
- Category delete with sub-categories → `KB_CATEGORY_IN_USE` message; deleting a category with articles leaves them uncategorized (backend behaviour).
- RTL: logical CSS properties only, `rtl-flip` on arrows; Arabic articles render `dir="rtl"` in either UI language.

## Test Plan

Out of scope (build-level verification only). Manual smoke: create draft → preview → publish → visible in `/help` → feedback; archive hides it; category CRUD; switch to Arabic.

## Verification Steps

1. **Frontend builds:** `cd customer-support-crm-web && npx ng build` with no errors or warnings in `features/knowledge-base/**`.

## Done Criteria

- [ ] Staff can manage articles and categories; publish requires `kb.publish`.
- [ ] Anonymous users can browse/search published articles and give feedback. en/ar + RTL.
