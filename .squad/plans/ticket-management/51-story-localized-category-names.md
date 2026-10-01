# Story 51 — Show the Arabic ticket category name wherever a ticket's category is displayed (Bug: BUG-15)

> Fix plan, not yet implemented. Paths and line numbers refer to customer-support-crm-api `dca992c` (`develop`) and customer-support-crm-web `06c817a` (`main`).
> Intake: [../../stories/ticket-management/localized-category-names/intake.md](../../stories/ticket-management/localized-category-names/intake.md)

## Prerequisites

- Story 25 — [25-story-ticket-lifecycle.md](25-story-ticket-lifecycle.md): `TicketQueries`, ticket history.
- Story 26 — [26-story-ticket-conversation-and-categories.md](26-story-ticket-conversation-and-categories.md): `TicketCategory` (`Name`, `NameAr`).
- Story 31 — [../customer-portal/31-story-portal-tickets-and-feedback.md](../customer-portal/31-story-portal-tickets-and-feedback.md): `PortalTicketResponse`.
- Story 34 — [../reports-and-management/34-story-reports-and-management-dashboards.md](../reports-and-management/34-story-reports-and-management-dashboards.md): `PostgresReportingQueries`, `CountByKey`.
- Story 29 — [../knowledge-base/29-story-knowledge-base-articles-and-search.md](../knowledge-base/29-story-knowledge-base-articles-and-search.md) and story 35 — [../ai-features/35-story-ai-assistant-and-chatbot.md](../ai-features/35-story-ai-assistant-and-chatbot.md): KB suggestions and AI categorize.
- Frontend stories 11, 15, 16, 17 and 18 — [../frontend/11-story-tickets-ui.md](../frontend/11-story-tickets-ui.md), [../frontend/15-story-knowledge-base-ui.md](../frontend/15-story-knowledge-base-ui.md), [../frontend/16-story-ai-assistant-panels.md](../frontend/16-story-ai-assistant-panels.md), [../frontend/17-story-customer-portal-ui.md](../frontend/17-story-customer-portal-ui.md), [../frontend/18-story-reports-ui.md](../frontend/18-story-reports-ui.md).
- Source: manual QA report `.squad/qa/2026-10-01-manual-qa-report.md`, finding M2.
- Backend paths are relative to **`customer-support-crm-api/`**, frontend paths to **`customer-support-crm-web/`**.

---

## Story Goal

Ticket categories have an optional Arabic name (`TicketCategory.NameAr`, `Domain/Tickets/TicketRecords.cs:158`). The pickers already show it: they load the full category and call `categoryLabel(category, language)` (`features/tickets/tickets.models.ts:300–302`). Every other place gets only the English name from the API, so the Arabic UI shows "Billing" instead of "الفواتير".

| Where | Before | After |
|---|---|---|
| Ticket details chip, side panel, ticket list column | `categoryName` only (English) | `categoryName` + `categoryNameAr`; the UI picks by language |
| Portal ticket detail "Category" | `categoryName` only | Same as above |
| Ticket history "Category" entry | English name stored at change time | New rows store the category id; the API returns `oldValue`/`newValue` (English) + `oldValueAr`/`newValueAr` |
| Reports "Top categories" (management dashboard), "By category" (ticket volume) | `label` only | `label` + `labelAr` |
| KB suggestions card on a ticket | KB `categoryName` only | + `categoryNameAr` |
| AI "Categorize" result | `categoryName` only | + `categoryNameAr` |
| Fallback | — | No `nameAr`, or UI in English → English name |

**Why both names, not a server-localized name.** The web app switches language in place (`TranslationService.setLanguage`, `core/localization/translation.service.ts:39–52`) without refetching data. A name chosen from `Accept-Language` would stay in the old language until the page reloads. Returning both names matches what the category pickers and `PortalCategoryResponse(Id, Name, NameAr)` (`Contracts/Portal/PortalContracts.cs:48`) already do.

**Why the history stores ids.** `TicketFieldChangeHandlers` writes the English name at change time (`Features/Tickets/TicketEventHandlers.cs:104–106`). The Arabic name cannot be recovered from that reliably: names can be renamed or duplicated. Ticket categories are never hard-deleted (there is no delete endpoint; `TicketCategorySlices.cs:97` only calls `SetActive`), so an id can always be resolved later. No migration is needed: `old_value`/`new_value` are `varchar(500)` (`Infrastructure/Persistence/Configurations/TicketConfiguration.cs:78–79`).

**Branches and departments:** `Branch` (`Domain/Organizations/Branch.cs:27`) and `Department` (`Domain/Organizations/Department.cs:29`) have only `Name`, with no Arabic column, so "Head Office" stays English. That is out of scope (it needs a migration, admin form fields and contract changes) and is listed in the intake.

**KB categories** also have `NameAr` (`Domain/KnowledgeBase/KnowledgeArticle.cs:244`). The KB pages already localize by looking up the loaded category list (`categoryLabel` in `features/knowledge-base/knowledge-base.models.ts:132–134`, used by `article-list.page.ts:311–314` and `help-article.page.ts:203–206`). Only the ticket KB suggestions card lacks the category list, so it gets `categoryNameAr` from the API.

**Deviation from the intake:** none. Note that `src/app/core/**` is edited (new pipe). `frontend/00-overview.md` reserves core for platform stories, but this helper is used by six features. It sits next to `localDate`, the only other localization pipe.

---

## Context — Read These Files First

Backend:

1. `src/CustomerSupportCrm.Contracts/Tickets/TicketContracts.cs`: `TicketListItemResponse` (35–57, `CategoryName` at 45), `TicketResponse` (72–99, `CategoryName` at 84), `TicketHistoryResponse` (113).
2. `src/CustomerSupportCrm.Application/Features/Tickets/Common/TicketQueries.cs`: `ProjectListAsync` (55–115). The category subquery is at line 69 and is mapped at 101. `GetResponseAsync` (117–174) has the `names` projection at 126–136 (category at 130), mapped at 158.
3. `src/CustomerSupportCrm.Application/Features/Tickets/TicketEventHandlers.cs`: `TicketHistoryRecorder.Record` (20–28, truncates to 500), category handler (101–107).
4. `src/CustomerSupportCrm.Domain/Tickets/TicketRecords.cs`: `TicketHistory.Record` (112–122), `TicketHistoryActions.CategoryChanged = "category"` (131).
5. `src/CustomerSupportCrm.Application/Features/Tickets/TicketQuerySlices.cs`: `GetTicketHistoryHandler` (200–223).
6. `src/CustomerSupportCrm.Contracts/Portal/PortalContracts.cs`: `PortalTicketResponse` (29–42, `CategoryName` at 35).
7. `src/CustomerSupportCrm.Application/Features/CustomerPortal/PortalTicketSlices.cs`: `ToResponseAsync` (34–52). The category query is at line 36.
8. `src/CustomerSupportCrm.Contracts/Reports/ReportContracts.cs`: `CountByKey(Key, Label, Count)` (3), used by `TicketVolumeReport.ByCategory` (17) and `ManagementDashboard.TopCategories30d` (76).
9. `src/CustomerSupportCrm.Infrastructure/Reporting/PostgresReportingQueries.cs`: category breakdowns at 41 (ticket volume) and 179 (management dashboard), `BreakdownAsync` (219–232).
10. `src/CustomerSupportCrm.Contracts/KnowledgeBase/KnowledgeBaseContracts.cs`: `KnowledgeArticleListItemResponse` (22–37, `CategoryName` at 30). `src/CustomerSupportCrm.Application/Features/KnowledgeBase/KnowledgeBaseSlices.cs`: `ProjectList` (35–51, category at 44), which `SuggestArticlesQuery` (341–354) uses.
11. `src/CustomerSupportCrm.Contracts/Ai/AiContracts.cs`: `CategorizationResponse` (12–19). `src/CustomerSupportCrm.Application/Features/Ai/AiSlices.cs` loads categories at line 288 and builds the response at 320.

Frontend:

12. `src/app/features/tickets/tickets.models.ts`: `TicketListItem` (15–38), `Ticket` (62–90), `TicketHistoryEntry` (105–112), `categoryLabel` (300–302).
13. `src/app/features/tickets/ticket-details.page.html` lines 61–63 (chip), `ticket-side-panel.component.ts` line 148, `ticket-list.page.html` line 134.
14. `src/app/features/tickets/ticket-history.component.ts`: template 35–43, `value()` (77–86).
15. `src/app/features/tickets/ticket-kb-suggestions.component.ts` line 52.
16. `src/app/features/customer-portal/customer-portal.models.ts`: `PortalTicket` (15–), `categoryName` at 22. `tickets/portal-ticket-detail.page.html` line 82. Inline `categoryName()` helpers: `channels/portal-contact.page.ts:147–149` and `tickets/portal-new-ticket.page.ts:93–95`.
17. `src/app/features/reports/reports.models.ts`: `CountByKey` (11–15). `report-format.ts` `keyLabel` (83–96). It is used by `management-dashboard.page.ts` (108, 129) and `ticket-volume.page.ts` (153).
18. `src/app/features/ai/ai.models.ts`: `Categorization` (37–, `categoryName` at 40). `ticket-ai-panel.component.html` line 127.
19. `src/app/features/knowledge-base/knowledge-base.models.ts`: `KbArticleListItem.categoryName` (67), `categoryLabel` (132–134). `src/app/features/sla/sla-lookups.service.ts` line 24 has a third copy of the same rule.
20. `src/app/core/localization/localized-date.pipe.ts`: the pattern for an impure, language-aware pipe.

---

## Backend Tasks

### 1 — Ticket contracts and projections

- `TicketListItemResponse`: add `string? CategoryNameAr` right after `CategoryName`.
- `TicketResponse`: add `string? CategoryNameAr` right after `CategoryName`.
- `TicketQueries.ProjectListAsync`: next to line 69 add
  `CategoryNameAr = db.TicketCategories.Where(c => c.Id == t.CategoryId).Select(c => c.NameAr).FirstOrDefault(),` and pass `t.CategoryNameAr` after `t.CategoryName` (101).
- `TicketQueries.GetResponseAsync`: add `CategoryAr = ...Select(c => c.NameAr)...` to the `names` projection (130) and pass `names.CategoryAr` after `names.Category` (158).

These are the only constructor call sites (`TicketListItemResponse` is also returned by the agent dashboard through `ProjectListAsync`).

### 2 — Portal ticket

- `PortalTicketResponse`: add `string? CategoryNameAr` right after `CategoryName`.
- `PortalTicketSlices.ToResponseAsync` line 36: select `new { c.Name, c.NameAr }` and pass both.

### 3 — Ticket history

**Write side.** In `TicketFieldChangeHandlers.Handle(TicketCategoryChangedDomainEvent)` (101–107), drop the name lookup and record the ids:

```csharp
public Task Handle(DomainEventNotification<TicketCategoryChangedDomainEvent> notification, CancellationToken cancellationToken)
{
    var e = notification.DomainEvent;
    history.Record(e.TicketId, TicketHistoryActions.CategoryChanged, e.PreviousCategoryId?.ToString(), e.CategoryId?.ToString());
    return Task.CompletedTask;
}
```

**Contract.** `TicketHistoryResponse(Guid Id, string Action, string? OldValue, string? NewValue, string? ActorName, DateTimeOffset OccurredAt, string? OldValueAr = null, string? NewValueAr = null)`. The two new fields are set only for `category` rows.

**Read side.** In `GetTicketHistoryHandler`, after `rows` is loaded (219):

```csharp
var categoryValues = rows.Where(h => h.Action == TicketHistoryActions.CategoryChanged)
    .SelectMany(h => new[] { h.OldValue, h.NewValue })
    .OfType<string>()
    .Distinct()
    .ToList();

var names = new Dictionary<string, (string Name, string? NameAr)>();
if (categoryValues.Count > 0)
{
    var ids = categoryValues.Select(v => Guid.TryParse(v, out var id) ? id : (Guid?)null).OfType<Guid>().ToList();
    var legacy = categoryValues.Where(v => !Guid.TryParse(v, out _)).ToList();
    var categories = await db.TicketCategories.AsNoTracking()
        .Where(c => ids.Contains(c.Id) || legacy.Contains(c.Name))
        .Select(c => new { c.Id, c.Name, c.NameAr })
        .ToListAsync(cancellationToken);

    foreach (var c in categories)
    {
        names[c.Id.ToString()] = (c.Name, c.NameAr);
    }

    // Rows written before this fix hold the English name: localize only when it matches exactly one category.
    foreach (var group in categories.Where(c => legacy.Contains(c.Name)).GroupBy(c => c.Name).Where(g => g.Count() == 1))
    {
        names[group.Key] = (group.Key, group.Single().NameAr);
    }
}
```

Then map each row. For `category` rows, `OldValue`/`NewValue` become `names[value].Name` (or the raw value when it is not found) and `OldValueAr`/`NewValueAr` become `names[value].NameAr`. Other rows are unchanged. `Guid.ToString()` gives the same lowercase "D" format that is stored, so the dictionary keys match.

Older clients keep working: `oldValue`/`newValue` still carry the English name, never a raw id (unless the id is unknown, which cannot happen because categories are not deleted).

### 4 — Report rows

- `CountByKey(string Key, string? Label, long Count, string? LabelAr = null)`. A trailing optional parameter, so the status/priority/channel/department breakdowns compile unchanged and send `labelAr: null`.
- `PostgresReportingQueries.BreakdownAsync` (221): add a `string? labelArExpression = null` parameter. Select `{labelArExpression ?? "NULL"}` as a fourth column, add it to `GROUP BY 1, 2, 3`, and map it with `r.GetNullableString(3)`. The `GROUP BY` stays correct because the label columns depend only on the key.
- Lines 41 and 179: pass `"(SELECT c.name_ar FROM ticket_categories c WHERE c.id = t.category_id)"` (the snake_case column, as in the migration snapshot).

### 5 — KB suggestions and AI categorize

- `KnowledgeArticleListItemResponse`: add `string? CategoryNameAr` after `CategoryName`. In `KnowledgeQueries.ProjectList`, add the matching `db.KnowledgeCategories...Select(c => c.NameAr)` subquery after line 44. `KnowledgeArticleResponse` is left as is, because the KB pages resolve names from the loaded category list.
- `CategorizationResponse`: add `string? CategoryNameAr` after `CategoryName`. In `AiSlices.cs`, line 288 selects `new { c.Id, c.Name, c.NameAr }` and line 320 passes `category?.NameAr`. The prompt (289) keeps the English names.

**No changes to:** domain, migrations, resx, docs or the CSV exports (none of them export category labels; `ReportSlices.cs:114–139`).

---

## Frontend Tasks

### 1 — Shared helper and pipe

Create `src/app/core/localization/localized-name.pipe.ts`:

```ts
/** The Arabic name when the UI is Arabic and one is set; otherwise the default name. */
export function localizedName(name: string | null | undefined, nameAr: string | null | undefined, language: string): string {
  return language === 'ar' && nameAr ? nameAr : (name ?? '');
}

/** `{{ tk.categoryName | localName: tk.categoryNameAr }}`. Impure so it follows the language signal (like `localDate`). */
@Pipe({ name: 'localName', pure: false })
export class LocalizedNamePipe implements PipeTransform {
  private readonly translations = inject(TranslationService);
  transform(name: string | null | undefined, nameAr?: string | null): string {
    return localizedName(name, nameAr, this.translations.language());
  }
}
```

The existing copies of the same rule should delegate to `localizedName`, so there is one source of truth: `categoryLabel` in `tickets.models.ts:300` and `knowledge-base.models.ts:132`, `portal-contact.page.ts:147`, `portal-new-ticket.page.ts:93`, and the `label` lambda in `sla-lookups.service.ts:24`.

### 2 — Models

Add the new fields:

- `TicketListItem.categoryNameAr`, `Ticket.categoryNameAr`, `TicketHistoryEntry.oldValueAr`/`newValueAr` (`string | null`, optional on the history entry).
- `PortalTicket.categoryNameAr`.
- `CountByKey.labelAr?: string | null`.
- `KbArticleListItem.categoryNameAr`.
- `Categorization.categoryNameAr`.

### 3 — Apply at every site

| File:line | Change |
|---|---|
| `tickets/ticket-details.page.html:62` | `{{ tk.categoryName \| localName: tk.categoryNameAr }}` |
| `tickets/ticket-side-panel.component.ts:148` | `{{ tk.categoryName ? (tk.categoryName \| localName: tk.categoryNameAr) : '—' }}` |
| `tickets/ticket-list.page.html:134` | same pattern as the side panel |
| `customer-portal/tickets/portal-ticket-detail.page.html:82` | same, with the `portal.ticket.noCategory` fallback |
| `tickets/ticket-kb-suggestions.component.ts:52` | `article.categoryName \| localName: article.categoryNameAr`, with the existing `noCategory` fallback |
| `ai/ticket-ai-panel.component.html:127` | `data.categoryName \| localName: data.categoryNameAr`, with the existing fallback |
| `tickets/ticket-history.component.ts` | see below |
| `reports/report-format.ts:84–87` | see below |

Add `LocalizedNamePipe` to each component's `imports`.

**History.** Change the template (38 and 41) to call `value(entry.action, entry.oldValue, entry.oldValueAr)` and `value(entry.action, entry.newValue, entry.newValueAr)`. In `value()`, start with:

```ts
if (action === 'category') {
  return localizedName(raw, rawAr, this.translations.language());
}
```

**Reports.** In `ReportFormatter.keyLabel`, replace the `item.label` branch with:

```ts
const label = localizedName(item.label, item.labelAr, this.translations.language());
if (label) {
  return label;
}
```

Widen the parameter type to `Pick<CountByKey, 'key' | 'label' | 'labelAr'>`. The chart labels (`management-dashboard.page.ts:108`) are `computed()` values that call `keyLabel`, so they now read the language signal and update on switch. The table columns call `keyLabel` per render.

**i18n:** no new keys.

---

## Test Plan

Test projects are **out of scope**: `tests/` must not be modified. No e2e.

---

## Verification Steps

1. **Backend build:** `dotnet build src/CustomerSupportCrm.Api/CustomerSupportCrm.Api.csproj` gives 0 warnings and 0 errors. Use `-o <temp dir>` if a running API locks `bin/`.
2. **Frontend build:** `cd customer-support-crm-web && npx ng build` gives 0 errors and 0 warnings.
3. **Set up:** the seeded category "Billing" has `nameAr` "الفواتير" (`Infrastructure/Persistence/Seed/DatabaseInitializer.cs:31–37`). Also create a category "NoArabic" without `nameAr`.
4. **Ticket screens:** switch the UI to Arabic. A Billing ticket shows "الفواتير" in the details chip, the side panel and the `/tickets` list. A "NoArabic" ticket shows "NoArabic". Switch to English: "Billing" appears without a reload.
5. **History:** change a ticket's category from Billing to another category. The history shows the Arabic names in Arabic and the English names in English. `GET /api/v1/tickets/{id}/history` returns English `oldValue`/`newValue` plus `oldValueAr`/`newValueAr`. In the database the new row holds ids. A row from before the fix (a name) still shows its name and, when unambiguous, the Arabic name.
6. **Portal:** sign in as the ticket's customer, set the portal to Arabic, and open the ticket. "Category" shows "الفواتير".
7. **Reports:** `/reports` management dashboard "Top categories" chart and table, and ticket volume "By category", show Arabic labels in Arabic. Status, priority and channel breakdowns are unchanged.
8. **KB / AI:** on a ticket, the KB suggestions card shows the Arabic KB category name when one is set. "Categorize" (with AI configured) shows the Arabic category name.

---

## Done Criteria

- [ ] Ticket details (chip and side panel), the ticket list and the portal ticket detail show the Arabic category name in the Arabic UI.
- [ ] New category history rows store ids. The history shows Arabic names in Arabic and English in English, and old rows still render.
- [ ] Report category rows carry `labelAr`, and the charts and tables use it.
- [ ] The KB suggestions card and the AI categorize result are localized.
- [ ] A missing `nameAr` falls back to the English name; switching language needs no reload.
- [ ] One helper (`localizedName` / `localName` pipe); the old copies delegate to it.
- [ ] Nothing changed in `tests/`, `docker-compose.yml`, `deploy/` or `.github/`.
- [ ] `dotnet build` and `ng build` pass with zero warnings.

**STOP HERE. Report to the user.**
