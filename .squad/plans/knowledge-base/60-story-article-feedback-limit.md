# Story 60 — Limit knowledge base article votes per visitor and article (Bug: BUG-24)

> Fix plan, implemented in `customer-support-crm-api` commit `93d98b1` (`develop`) and `customer-support-crm-web` commit `79460bd` (`main`). Paths and line numbers refer to those commits.
> Intake: [../../stories/knowledge-base/article-feedback-limit/intake.md](../../stories/knowledge-base/article-feedback-limit/intake.md)

## Prerequisites

- Story 29 ([29-story-knowledge-base-articles-and-search.md](29-story-knowledge-base-articles-and-search.md)): public KB endpoints and `KnowledgeArticle.RecordFeedback`.
- Security story 07 ([../security-and-administration/07-story-security-hardening.md](../security-and-administration/07-story-security-hardening.md)): rate limiter setup.
- Frontend story 15 ([../frontend/15-story-knowledge-base-ui.md](../frontend/15-story-knowledge-base-ui.md)): help center article page (vote kept in `localStorage`, key `crm.kb.feedback.<id>`).

---

## Story Goal

`POST /api/v1/public/kb/articles/{id}/feedback` is anonymous. Before the fix:

- It used the shared `public` policy: 30 requests per minute per IP across all public endpoints. One script could add hundreds of votes to one article per hour. QA added 5 "not helpful" votes in a row, all counted.
- The handler checked `Status == Published` but not `Visibility == Public`, so internal articles could be voted on by id. `GET /public/kb/articles/{slug}` already hides them.

After the fix:

| Case | Result |
|---|---|
| Votes 1–3 from one IP on one article in a day | 200, counted |
| Vote 4 and later | 429 (`Retry-After`), not counted |
| Vote on an internal, draft or archived article | 404 `KB_ARTICLE_NOT_FOUND` |
| Help center gets a 429 | Shows "Thanks for your feedback!" and stores the vote (it was counted earlier) |

**Deviation from the intake:** none.

---

## Context — Read These Files First

1. API `src/CustomerSupportCrm.Application/Abstractions/Http/RateLimitPolicies.cs:12`: `ArticleFeedback = "article-feedback"`.
2. API `src/CustomerSupportCrm.Api/Configuration/SecurityExtensions.cs` 154–158: the policy.
3. API `src/CustomerSupportCrm.Application/Features/KnowledgeBase/KnowledgeBaseSlices.cs`: handler 506–516 (visibility check at 512) and endpoint 538–545 (`RequireRateLimiting(RateLimitPolicies.ArticleFeedback)` at 543).
4. Web `src/app/features/knowledge-base/help/help-article.page.ts` `vote()`: the `RATE_LIMITED` branch at 227. `ApiError` maps status 429 to `RATE_LIMITED` (`core/http/api-error.ts:77`).

---

## Backend Tasks

1. New policy: a fixed window keyed by client IP **and** request path (the path holds the article id), 3 permits per day, no queue:
   ```csharp
   options.AddPolicy(RateLimitPolicies.ArticleFeedback, context => RateLimitPartition.GetFixedWindowLimiter(
       $"{context.Connection.RemoteIpAddress?.ToString() ?? "unknown"}|{context.Request.Path.Value}",
       _ => new FixedWindowRateLimiterOptions { PermitLimit = 3, Window = TimeSpan.FromDays(1), QueueLimit = 0 }));
   ```
   Three, not one, so that a few people behind one office NAT can still vote. The global limiter still applies, which caps how many partitions a single client can create.
2. The feedback endpoint uses the new policy instead of `Public`.
3. `SubmitArticleFeedbackHandler` adds `a.Visibility == ArticleVisibility.Public` to its lookup.

---

## Frontend Tasks

1. In `vote()`'s error handler, `RATE_LIMITED` is treated as success: set `lastVote`, set `feedback` to `'sent'`, and call `writeVote`. Any other error still shows the message.

No i18n changes.

---

## Test Plan

Test projects are out of scope.

---

## Verification Steps

1. `dotnet build … -o <temp>`: 0 warnings. `npx ng build`: 0 errors, 0 warnings.
2. Create and publish a public test article. Five `POST …/feedback {"helpful":true}` calls return `200 200 200 429 429`, and `helpfulCount` is 3.
3. Voting on a published **internal** article returns 404.
4. Help center, after clearing `crm.kb.feedback.*` from localStorage: "Yes" gets 429 and the page shows "Thanks for your feedback!".

All passed on 2026-10-01; the test article was archived afterwards.

---

## Known limits

- Limiter partitions live in memory: an API restart resets the counts, and with several API instances each one counts separately. A durable per-visitor record would need a table; that is out of scope.

---

## Done Criteria

- [x] At most 3 votes per IP and article per day; the 4th returns 429.
- [x] Votes on internal articles return 404.
- [x] The help center treats a rate-limited vote as already counted.
- [x] `dotnet build` 0 warnings; `npx ng build` 0 errors, 0 warnings.

**STOP HERE. Report to the user.**
