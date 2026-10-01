# Story intake

- Folder: `.squad/stories/ticket-management/localized-category-names/intake.md`

---

## Feature

- **Feature name (display):** Ticket Management
- **Feature slug (folder under `plans/`):** `ticket-management`

## Tracker (metadata only)

- **Tracker type:** `none`
- **Work item id:** `BUG-15`
- **Work item type:** `Bug`
- **Status:** `To do`
- **Assignee:** ``
- **Labels:** `backend`, `frontend`, `bug`, `tickets`, `i18n`, `reports`, `portal`

---

## Title

```
Show the Arabic ticket category name wherever a ticket's category is displayed
```

---

## Description

```
Manual QA (2026-10-01, finding M2): in the Arabic UI, a ticket in category
"Billing" (nameAr "الفواتير") still shows "Billing" in:

- the category chip on ticket details   (features/tickets/ticket-details.page.html:62)
- the side panel                         (features/tickets/ticket-side-panel.component.ts:148)
- the ticket list column                 (features/tickets/ticket-list.page.html:134)
- the ticket history "Category" entry
- the portal ticket detail "Category"    (features/customer-portal/tickets/portal-ticket-detail.page.html:82)
- reports "Top categories" (management dashboard) and "By category" (ticket volume)

Cause: the API returns only the English name (CategoryName / Label). The
category pickers already localize, because they hold the full category
(name + nameAr) and use categoryLabel(...).

Two more sites have the same gap: the KB suggestions card on a ticket
(KB article categoryName) and the AI "Categorize" result (categoryName).

The ticket history stores the English category NAME at the time of the
change (TicketEventHandlers.cs:105-106), so it cannot be localized later.

Fix:
- API: add CategoryNameAr next to every CategoryName that describes a ticket
  category or KB category in a ticket screen, and LabelAr to report rows.
- History: new "category" rows store the category id; the history query
  resolves ids (and, best effort, old name rows) to Name + NameAr.
- Web: one shared helper + pipe that picks the Arabic name when the UI is
  Arabic and one is set, applied at every site above.
```

---

## Acceptance criteria

```
- [ ] With the UI in Arabic, a ticket whose category has nameAr shows the Arabic name on ticket details (chip + side panel), in the ticket list, and in the portal ticket detail.
- [ ] A category change made after the fix shows the Arabic names in the ticket history; switching to English shows the English names.
- [ ] Reports "Top categories" (management dashboard) and "By category" (ticket volume) show Arabic labels in the chart and the table.
- [ ] The KB suggestions card and the AI categorize result show the Arabic category name.
- [ ] A category without nameAr falls back to the English name everywhere.
- [ ] Switching the language updates the names without reloading.
- [ ] `dotnet build` passes with zero warnings; `ng build` passes with zero errors and zero warnings.
```

---

## Attachments

None.

---

## Dependencies

- **Blocked by / related ids:** none.
- **Depends on code areas or other stories:** stories 25/26 (tickets, categories), 31 (portal tickets), 34 (reports), 29 (KB), 35 (AI); frontend stories 11, 15, 16, 17, 18.
- **Source:** `.squad/qa/2026-10-01-manual-qa-report.md`, finding M2.

## Technical hints (optional)

- Repo roots: `customer-support-crm-api/` (branch `develop`, .NET 10) and `customer-support-crm-web/` (branch `main`, Angular).
- Existing helpers: `categoryLabel` in `features/tickets/tickets.models.ts:300` and `features/knowledge-base/knowledge-base.models.ts:132`.

## Out of scope

- Do NOT touch Docker, docker-compose.yml, deploy/ or .github/ (CI/CD).
- Do NOT create or modify anything under tests/.
- No e2e tests.
- Arabic names for branches and departments: `Branch` and `Department` have no `NameAr` column, so "Head Office" stays English. Adding one needs a migration and admin UI changes; that is a separate story.
- Rewriting existing ticket history rows (old rows keep their English name; they are localized only when the name still matches exactly one category).
- CSV exports (none of them contain category labels).
