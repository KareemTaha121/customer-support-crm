# Story intake

- Folder: `.squad/stories/knowledge-base/article-feedback-limit/intake.md`

---

## Feature

- **Feature name (display):** Knowledge Base
- **Feature slug (folder under `plans/`):** `knowledge-base`

## Tracker (metadata only)

- **Tracker type:** `none`
- **Work item id:** `BUG-24`
- **Work item type:** `Bug`
- **Status:** `Done`
- **Assignee:** ``
- **Labels:** `backend`, `frontend`, `bug`, `kb`, `security`

---

## Title

```
Limit knowledge base article votes per visitor and article
```

---

## Description

```
QA round 2 (N4). POST /public/kb/articles/{id}/feedback with {"helpful":false} five times
in a row from one client returned 200 five times and notHelpfulCount became 5. The help
center remembers a vote in localStorage (crm.kb.feedback.<id>), but the API only has the
shared "public" limit (30 per minute per IP), so a script can stuff ratings.

Also found: the handler only checks Status == Published, not Visibility == Public, so an
anonymous caller can vote on internal articles by id (GET by slug already hides them).
```

---

## Acceptance criteria

```
- [x] At most 3 votes per client IP and article per day; the 4th returns 429.
- [x] Votes on internal articles return 404.
- [x] The help center shows "Thanks for your feedback!" when the vote is rate limited.
- [x] dotnet build 0 warnings; npx ng build 0 errors, 0 warnings.
```

---

## Attachments

None.

---

## Dependencies

- **Blocked by / related ids:** none
- **Depends on code areas or other stories:** found in the round 2 manual QA pass, [../../../qa/2026-10-01-manual-qa-report-round2.md](../../../qa/2026-10-01-manual-qa-report-round2.md).

## Technical hints (optional)

- Rate limit policies: `CustomerSupportCrm.Api/Configuration/SecurityExtensions.cs`; names in `Abstractions/Http/RateLimitPolicies.cs`.

## Out of scope

- Do NOT touch Docker, docker-compose.yml, deploy/ or .github/ (CI/CD).
- Do NOT create or modify anything under tests/ or e2e/.
- Durable per-visitor dedupe (cookie or table). In-memory limiter partitions reset on restart.
