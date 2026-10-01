# Story 52 — Isolate mixed-direction values in RTL and use one date format rule (Bug: BUG-16)

> Fix plan, not yet implemented. Paths and line numbers refer to customer-support-crm-web `06c817a` (`main`).
> Intake: [../../stories/frontend/rtl-bidi-isolation/intake.md](../../stories/frontend/rtl-bidi-isolation/intake.md)

## Prerequisites

- Story 08 — [08-story-core-platform-shell.md](08-story-core-platform-shell.md): `LocalizedDatePipe`, `TranslatePipe`, `TranslationService`, `app.config.ts`.
- Story 11 — [11-story-tickets-ui.md](11-story-tickets-ui.md): ticket history, attachments, list, side panel.
- Story 18 — [18-story-reports-ui.md](18-story-reports-ui.md): the only Material datepicker today (`report-filter-bar.component.ts`).
- Source: manual QA report `.squad/qa/2026-10-01-manual-qa-report.md`, findings M3 and L5.
- All paths are relative to `customer-support-crm-web/src/app/` unless stated otherwise.
- This story edits `core/**` and several feature folders. The ownership rule in `00-overview.md` is for feature stories; a cross-cutting bug fix may edit them. Coordinate with any story editing the same templates in parallel.

---

## Story Goal

### Part A — bidi isolation (M3)

`TranslationService` sets `<html dir="rtl">` for Arabic (`translation.service.ts:87–89`). `LocalizedDatePipe` formats Arabic dates with `ar-SA-u-nu-latn-ca-gregory` (`localized-date.pipe.ts:22`). Checked with Node 24 `Intl`: that locale returns `"01‏/10‏/2026، 6:53 ص"` for `medium`, with a U+200F RIGHT-TO-LEFT MARK after the day and after the month. Inside an RTL paragraph, next to a Latin name and the neutral ` · `, the bidi algorithm splits and reorders those runs. That is the scrambled history line.

The same "value · value" join, with no isolation, appears in about 20 templates (sweep below). Dates are also interpolated into translated sentences, e.g. `'tickets.sla.dueAt' | t: { date: (… | localDate: 'short') }` (`ticket-side-panel.component.ts:73`), where a `<bdi>` cannot be placed.

| | Before | After |
|---|---|---|
| History line (ar) | `01 · 2026/10/Administrator…` | `Administrator · 01/10/2026، 6:53 ص` |
| Dates in translated sentences | Not isolated | Isolated by the pipe |
| `{{name}}`-style params in `t` | Not isolated | Isolated by the pipe |
| Names, numbers, emails in meta lines | Not isolated | `<bdi>` |

Three layers, each small:

1. **`localDate` output is isolated:** the pipe returns `⁨…⁩` (FIRST STRONG ISOLATE … POP DIRECTIONAL ISOLATE). Every date in the app, including inside translations and tooltips, becomes one unit. FSI picks its direction from the first strong character: the Arabic string starts with digits then U+200F, so it is RTL; the English one is LTR.
2. **`t` string parameters are isolated** the same way in `TranslatePipe` (template use only; `TranslationService.translate()` is unchanged). Number parameters stay numbers, because `translate()` needs `typeof params.count === 'number'` for plurals (`translation.service.ts:67–71`).
3. **`<bdi>` around data values** in the meta lines. `<bdi>` has `unicode-bidi: isolate` in the browser's default style sheet, and so does any element with a `dir` attribute; that is why `settings.page.ts:57` (`dir="ltr"`) is already safe.

**No global CSS rule.** A rule such as `* { unicode-bidi: plaintext }` would change every paragraph's base direction; `<bdi>` and `[dir]` already isolate by default. `src/styles.scss` has no `unicode-bidi` rule today and gets none.

### Part B — one date format rule (L5)

Today the pipe offers `short`, `medium`, `shortDate`, `mediumDate`, `shortTime` and `relative` (`localized-date.pipe.ts:14`). In English, `medium` gives `1 Oct 2026, 06:53` and `short` gives `01/10/2026, 06:53`; the ticket page uses both, plus `relative`. In Arabic both are numeric (`01/10/2026، 6:53 ص` and `1/10/2026، 6:53 ص`).

The rule:

| Use | Format |
|---|---|
| Absolute date and time | `'medium'` (the default) |
| Date only | `'mediumDate'` |
| Time inside a chat bubble, and "last updated" | `'shortTime'` |
| Lists, queues, feeds, conversation threads | `'relative'`, with the absolute `medium` value as `[title]` or `[matTooltip]` |
| Supplement after an absolute due date | `'relative'` in parentheses (side panel 74, 88; unchanged) |

`'short'` and `'shortDate'` are removed from the pipe's format type, so a missed call site fails the build.

### Part C — date inputs

The five `<input type="date">` fields (tickets list 66, 71; audit 65, 69; API key expiry 41) are native browser inputs. Their `mm/dd/yyyy` placeholder follows the browser's own locale; the page's `lang`/`dir` does not change it, so it cannot be fixed by an attribute. They are replaced with the Material datepicker, which formats through a `DateAdapter`.

The only datepicker today (`reports/report-filter-bar.component.ts`) provides its own `provideNativeDateAdapter()` (27) and sets the locale in an effect (133–135). This story moves that to one app-wide provider. `NativeDateAdapter.parse` uses `Date.parse`, which reads a typed `01/10/2026` as January 10; the app adapter parses `d/m/yyyy` first.

**Deviation from the intake:** none. The intake's QA note suggested `dir="auto"` as an alternative; `<bdi>` is used everywhere for data values, because `dir="auto"` on an inline element only isolates too and `<bdi>` says so in the markup. Existing `dir="ltr"` / `dir="auto"` attributes are left alone (one exception, `branches.component.ts:74`, below).

---

## Context — Read These Files First

1. `core/localization/localized-date.pipe.ts`: format union (14), locale (22), `Intl.DateTimeFormat` (34), fallback (36), `relative()` (40–57).
2. `core/localization/translate.pipe.ts`: `transform` (12–14).
3. `core/localization/translation.service.ts`: `translate()` plural handling and `{{ }}` replacement (66–75), `applyDocumentLanguage()` (86–90).
4. `core/config/app-config.ts:10`: `Language`.
5. `app.config.ts`: providers (14–35).
6. `features/reports/report-filter-bar.component.ts`: `provideNativeDateAdapter()` (5, 27), `DateAdapter` injection (102), locale effect (133–135), the datepicker markup (33–39).
7. `features/tickets/ticket-history.component.ts:45`: the reported line.
8. `features/tickets/ticket-list.page.html:64–72` and `ticket-list.page.ts`: filters (118–126), `buildQuery` (204–226, `dayToIso` 221–222), `readQueryParams` (244–245), `writeQueryParams` (275–276); `dayToIso` in `tickets.models.ts:305`.
9. `features/administration/audit/audit.page.ts`: date inputs (65, 69), `filters` (174), `apply()` (201–216), `startOfDay` / `endOfDay` (265–271).
10. `features/administration/integrations/api-key-dialog.component.ts`: input (41), `today` (74), `expiresAt` control (82), request (112).

### Sweep A — joins and adjacent values (`grep -rn "·\| — " src/app --include=*.ts --include=*.html`)

| Location | Content | Action |
|---|---|---|
| `features/tickets/ticket-history.component.ts:45` | `actorName · date` | `<bdi>` actor; date via pipe (**reported**) |
| `features/tickets/ticket-attachments.component.ts:29` | `size · uploader · date` | `<bdi>` size and uploader |
| `features/tickets/ticket-list.page.html:113` | `customerName · channel` | `<bdi>` customer |
| `features/tickets/ticket-side-panel.component.ts:170` | `escalationLevel {level} · date` | Pipes only (param and date) |
| `features/tickets/ticket-side-panel.component.ts:174` | `rating / 5 — comment` | `<bdi>` comment |
| `features/tickets/ticket-side-panel.component.ts:71, 73, 85, 87` | date inside translated sentence | Pipe only |
| `features/tickets/agent-picker.component.ts:43` | `displayName · email` | `<bdi>` email |
| `features/tickets/ticket-create.page.ts:76` | `· number email/phone` | `<bdi>` number, `<bdi>` email/phone |
| `features/tickets/ticket-kb-suggestions.component.ts:52` | `categoryName · visibility` | `<bdi>` category |
| `features/tickets/ticket-details.page.html:2` | `[title]="number + ' · ' + subject"` | No change: a single string input to `app-page-header`; a Latin code then a subject keeps a correct order |
| `features/customers/customer-attachments.component.ts:62–64` | `size · date · uploader` | `<bdi>` size and uploader |
| `features/customers/customer-contacts.component.ts:41` | `value · label` | `<bdi>` label (value already has `.ltr`) |
| `features/customers/customer-notes.component.ts:77–80` | `author date · edited` | `<bdi>` author |
| `features/customers/customer-profile-tab.component.ts:66` | `externalSystem · externalId` | `<bdi>` each |
| `features/customers/customer-details.page.ts:56` | `[subtitle]="number + ' · ' + company"` | No change (same reason as the ticket title) |
| `features/customers/customer-form.page.ts:207` | `number — name` (duplicate link) | `<bdi>` each |
| `features/channels/chat-console.page.html:77–80` | `ticketNumber · status · handledBy {name}` | `<bdi>` number; param via pipe |
| `features/channels/chat-console.page.html:119–120` | `authorName` then time | `<bdi>` author |
| `features/customer-portal/channels/portal-chat.page.html:44` | `status · reference {number}` | Pipe only |
| `features/customer-portal/channels/portal-chat.page.html:67–68` | `author · time` | `<bdi>` author |
| `features/dashboard/dashboard.page.html:128` | `number · relative` | `<bdi>` number |
| `features/dashboard/task-dialog.component.ts:93` | `number · subject` | `<bdi>` each |
| `features/knowledge-base/help/help-article-list.component.ts:25` | `category · type` | `<bdi>` category |
| `features/administration/organization/branches.component.ts:74` | `<span dir="ltr">· email</span>` | The dot is inside the LTR span, so in Arabic it lands on the wrong side. Becomes `<span class="crm-muted">· <bdi>{{ department.email }}</bdi></span>` |
| `features/administration/settings/settings.page.ts:57–58` | `<span dir="ltr">key</span> · default {value}` | Already isolated; param via pipe |
| `features/sla/sla-policies.page.ts:97` | `<bdi>start–end</bdi>` | Already isolated |
| `features/channels/channels-admin.page.html:104` | `<bdi>to</bdi>` | Already isolated |
| `features/reports/report-filter-bar.component.ts:126`, `features/sla/sla-lookups.service.ts:49` | `` `${a} — ${b}` `` built in TypeScript for `mat-option` labels | No change: both parts are names; isolating them needs FSI/PDI in code, not worth it here |

Lines where the name and the date are separate elements of a flex header (`portal-ticket-detail.page.html:21–22, 30–31`, `ticket-conversation.component.ts:53–54`) only depend on the date pipe.

### Sweep B — `localDate` call sites that change (`grep -rn "localDate" src/app`)

`'short'` → `'medium'` (15):
`features/administration/audit/audit.page.ts:114`; `features/administration/integrations/deliveries-dialog.component.ts:42`; `features/channels/channels-admin.page.html:94, 115`; `features/dashboard/dashboard.page.html:85`; `features/dashboard/tasks.page.ts:93, 97, 103`; `features/tickets/ticket-attachments.component.ts:29`; `features/tickets/ticket-list.page.html:156`; `features/tickets/ticket-side-panel.component.ts:71, 73, 85, 87, 170`.

`'shortDate'` → `'mediumDate'` (5):
`features/administration/integrations/integrations.page.ts:90, 98`; `features/administration/users/users.page.ts:124`; `features/customers/customer-list.page.ts:185`; `features/knowledge-base/staff/article-list.page.ts:185`.

`'relative'` without an absolute tooltip → add `[title]="value | localDate"` (8; `[title]` follows `customer-history.component.ts:83` and needs no import):
`core/layout/notification-bell.component.ts:55`; `features/administration/integrations/integrations.page.ts:94`; `features/administration/users/users.page.ts:119`; `features/channels/chat-console.page.html:46`; `features/customer-portal/tickets/portal-tickets.page.ts:91`; `features/customers/customer-profile-tab.component.ts:26`; `features/dashboard/dashboard-ticket-list.component.ts:41`; `features/dashboard/dashboard.page.html:128`.

Unchanged: every default (`medium`) and `'medium'` / `'mediumDate'` call; `'shortTime'` at `chat-console.page.html:120`, `portal-chat.page.html:68`, `dashboard.page.html:141`; `'relative'` with a tooltip at `ticket-conversation.component.ts:76`, `ticket-list.page.html:167`, `customer-history.component.ts:83`; the side-panel supplements at 74 and 88.

On the reported ticket page this gives: description and side-panel dates `medium`, SLA dates `medium` with `(relative)`, attachments `medium`, messages `relative` with a `medium` tooltip.

---

## Frontend Tasks

### 1 — Shared date locale and adapter (new `core/localization/date-locale.ts`)

```ts
import { EnvironmentProviders, Provider, effect, inject, makeEnvironmentProviders, provideEnvironmentInitializer } from '@angular/core';
import { DateAdapter, MAT_DATE_FORMATS, MAT_NATIVE_DATE_FORMATS, NativeDateAdapter } from '@angular/material/core';
import { Language } from '../config/app-config';
import { TranslationService } from './translation.service';

/** Intl locale for both the localDate pipe and the datepicker (Latin digits, Gregorian calendar). */
export function dateLocale(language: Language): string {
  return language === 'ar' ? 'ar-SA-u-nu-latn-ca-gregory' : 'en-GB';
}

/** Native adapter that reads typed dates as day/month/year (both app locales are day-first). */
export class AppDateAdapter extends NativeDateAdapter {
  override parse(value: unknown, parseFormat?: unknown): Date | null {
    if (typeof value === 'string') {
      const text = value.replace(/[‎‏؜⁨⁩]/g, '').trim();
      const match = /^(\d{1,2})[./-](\d{1,2})[./-](\d{4})$/.exec(text);
      if (match) {
        const [day, month, year] = [Number(match[1]), Number(match[2]), Number(match[3])];
        const date = new Date(year, month - 1, day);
        return date.getFullYear() === year && date.getMonth() === month - 1 && date.getDate() === day ? date : this.invalid();
      }
    }
    return super.parse(value, parseFormat);
  }
}

/** App-wide DateAdapter whose locale follows the UI language. */
export function provideAppDateAdapter(): EnvironmentProviders {
  const providers: Provider[] = [
    { provide: DateAdapter, useClass: AppDateAdapter },
    { provide: MAT_DATE_FORMATS, useValue: MAT_NATIVE_DATE_FORMATS },
  ];
  return makeEnvironmentProviders([
    ...providers,
    provideEnvironmentInitializer(() => {
      const adapter = inject<DateAdapter<Date>>(DateAdapter);
      const translations = inject(TranslationService);
      effect(() => adapter.setLocale(dateLocale(translations.language())));
    }),
  ]);
}

/** Local calendar day as `yyyy-mm-dd`, or '' (the format the filters and URLs already use). */
export function dayKey(date: Date | null): string {
  if (!date || Number.isNaN(date.getTime())) {
    return '';
  }
  const pad = (n: number) => String(n).padStart(2, '0');
  return `${date.getFullYear()}-${pad(date.getMonth() + 1)}-${pad(date.getDate())}`;
}

/** `yyyy-mm-dd` → local Date, or null. */
export function fromDayKey(key: string): Date | null {
  const match = /^(\d{4})-(\d{2})-(\d{2})$/.exec(key);
  return match ? new Date(Number(match[1]), Number(match[2]) - 1, Number(match[3])) : null;
}
```

The native formats display `{ year: 'numeric', month: 'numeric', day: 'numeric' }`, i.e. `01/10/2026` (en-GB) and `1‏/10‏/2026` (ar); the parser strips the marks.

### 2 — `app.config.ts`

Add `provideAppDateAdapter()` to `providers`. Check while implementing that `provideEnvironmentInitializer` runs after `TranslationService` has its initial language (it reads it from storage in the signal initialiser, `translation.service.ts:29`), so the first locale is right.

### 3 — `localized-date.pipe.ts`

```ts
export type LocalDateFormat = 'medium' | 'mediumDate' | 'shortTime' | 'relative';

transform(value: …, format: LocalDateFormat = 'medium'): string {
  …
  const locale = dateLocale(this.translations.language());
  const text = format === 'relative' ? relative(date, locale) : this.absolute(date, locale, format);
  return `⁨${text}⁩`;
}
```

- Remove `short` and `shortDate` from `options` (27–33) and from the type.
- The fallback at 36 (`formatDate(date, format, 'en-US')`) keeps working for the remaining names; `mediumDate` and `shortTime` are valid Angular format names, `medium` too.
- Update the doc comment (6–9): output is isolated with FSI/PDI; list the rule from Part B.
- Empty input still returns `''` (no marks).

### 4 — `translate.pipe.ts`

```ts
transform(key: string | null | undefined, params?: Record<string, string | number | null | undefined>): string {
  if (!key) {
    return '';
  }
  return this.translations.translate(key, params && isolateStrings(params));
}

/** Wraps string params in FSI…PDI so names, numbers and dates keep their order in RTL text. Numbers stay numbers (plurals). */
function isolateStrings(params: Record<string, string | number | null | undefined>): Record<string, string | number | null | undefined> {
  const result: Record<string, string | number | null | undefined> = {};
  for (const [name, value] of Object.entries(params)) {
    result[name] = typeof value === 'string' && value !== '' ? `⁨${value}⁩` : value;
  }
  return result;
}
```

`TranslationService.t()` / `translate()` in component code (toasts, `document.title`) are not changed.

### 5 — `<bdi>` in templates

Apply the "Action" column of Sweep A. Pattern, using the reported line:

```html
<small class="crm-muted"><bdi>{{ entry.actorName ?? ('tickets.history.system' | t) }}</bdi> · {{ entry.occurredAt | localDate }}</small>
```

and for attachments (`ticket-attachments.component.ts:29`, with Sweep B):

```html
<bdi>{{ file.size | fileSize }}</bdi> · <bdi>{{ file.uploadedByName ?? ('tickets.attachments.unknownUploader' | t) }}</bdi> · {{ file.createdAt | localDate }}
```

Wrap only the value, never the ` · `. Inside `mat-option` (agent picker, ticket create, task dialog) `<bdi>` is fine: the option text is a normal element.

### 6 — Date formats

Apply Sweep B. Example (`ticket-side-panel.component.ts:73`): `(tk.sla.firstResponseDueAt | localDate: 'short')` → `(tk.sla.firstResponseDueAt | localDate)`. Example tooltip (`notification-bell.component.ts:55`): `<span class="bell-item__time" [title]="item.createdAt | localDate">{{ item.createdAt | localDate: 'relative' }}</span>`.

### 7 — Date inputs to Material datepicker

For each field: import `MatDatepickerModule` in the component, change the control to `Date | null`, and convert at the edges with `dayKey` / `fromDayKey`.

**Tickets list** (`ticket-list.page.html:64–72`, `ticket-list.page.ts`):

```html
<mat-form-field>
  <mat-label>{{ 'tickets.filters.createdFrom' | t }}</mat-label>
  <input matInput [matDatepicker]="fromPicker" formControlName="createdFrom" />
  <mat-datepicker-toggle matIconSuffix [for]="fromPicker" />
  <mat-datepicker #fromPicker />
</mat-form-field>
```

(the same for `createdTo` with `toPicker`). In the TS: `createdFrom: [null as Date | null]`, `createdTo: [null as Date | null]` (124–125); `dayToIso(dayKey(value.createdFrom), false)` / `(…createdTo), true)` (221–222); `createdFrom: fromDayKey(text('from'))`, `createdTo: fromDayKey(text('to'))` (244–245); `from: dayKey(value.createdFrom) || null`, `to: dayKey(value.createdTo) || null` (275–276). URLs keep `yyyy-mm-dd`, so existing links still work. `filters.reset()` (197) resets to `null`.

**Audit** (`audit.page.ts:65, 69, 174, 207–208`): the same markup; drop `dir="ltr"`, since the adapter formats per language. `filters` becomes `group({ from: [null as Date | null], to: [null as Date | null], action: '', entityType: '', entityId: '' })`; `apply()` uses `startOfDay(dayKey(v.from))` and `endOfDay(dayKey(v.to))`.

**API key expiry** (`api-key-dialog.component.ts:41, 74, 82, 112`): `<input matInput [matDatepicker]="expiryPicker" formControlName="expiresAt" [min]="today" />` with toggle and `<mat-datepicker #expiryPicker />`; `readonly today = new Date();`; `expiresAt: [null as Date | null]`; request: `expiresAt: value.expiresAt ? new Date(value.expiresAt.getFullYear(), value.expiresAt.getMonth(), value.expiresAt.getDate(), 23, 59, 59).toISOString() : null` (same end of the chosen local day as today).

**Reports filter bar** (`report-filter-bar.component.ts`): remove `providers: [provideNativeDateAdapter()]` (27), the `DateAdapter` injection (102) and the locale effect (133–135), and drop `DateAdapter, provideNativeDateAdapter` from the import (5). It now uses the app-wide adapter.

**No changes to:** i18n JSON files (no new keys), `styles.scss`, the backend, the `datetime-local` inputs in `task-dialog.component.ts:67, 72`.

---

## Test Plan

Test projects are **out of scope**. The web repo has no tests for these pipes or components.

---

## Verification Steps

1. **Build:** `cd customer-support-crm-web && npx ng build` → 0 errors, 0 warnings. A leftover `'short'` or `'shortDate'` fails the build (strict templates); fix it per Sweep B.
2. **Leftovers:** `grep -rn "localDate: 'short'\b\|localDate: 'shortDate'" src/app` → no results.
3. **Reported line (ar):** switch to Arabic, open a ticket → History. Each line reads actor, then ` · `, then the date as `01/10/2026، 6:53 ص` read right to left as Arabic text, with no digits next to the name.
4. **Other meta lines (ar):** ticket attachments, the tickets list subject column, agent picker options, the chat console header and messages, the portal chat, customer attachments and notes, KB suggestions, branches (department email): values keep their order, and the dot sits between them.
5. **Translated sentences (ar):** side-panel SLA "due" / "responded", dashboard task "due", help article "updated": the date is intact inside the sentence.
6. **Formats (en):** on a ticket page every absolute date reads like `1 Oct 2026, 06:53`; messages show "14 seconds ago" with that value as tooltip. The tickets list SLA column, audit table, outbox and deliveries now show the same `medium` format. Hover a relative time in the notification bell, users list, chat queue → the absolute date shows.
7. **Date inputs:** tickets list "Created from/to", audit "From/To", API key "Expires": the field shows `dd/mm/yyyy` in English and the Arabic day-first form in Arabic; the calendar opens; typing `01/10/2026` selects 1 October. Filtering by a range gives the same results as before, and the URL still holds `from=2026-10-01`. Opening an old URL with `from=2026-10-01` fills the picker.
8. **Language switch:** with a date picked, switch language → the input re-formats.
9. **Reports:** the report range picker still works and re-formats on language switch.

---

## Done Criteria

- [ ] `localDate` returns FSI/PDI-isolated text and offers only `medium`, `mediumDate`, `shortTime`, `relative`.
- [ ] The `t` pipe isolates string parameters; plural `count` still works.
- [ ] Every "Action" row of Sweep A is applied.
- [ ] Every call site of Sweep B is changed; relative times have an absolute tooltip.
- [ ] The five date inputs use the Material datepicker on one app-wide `AppDateAdapter`; the reports bar no longer provides its own.
- [ ] Nothing changed in tests, e2e, `docker-compose.yml`, `deploy/` or `.github/`.
- [ ] `npx ng build` passes with 0 errors and 0 warnings.

## Open question

- **Task due / remind (`dashboard/task-dialog.component.ts:67, 72`)** use `<input type="datetime-local">`, which also shows the browser's `mm/dd/yyyy, --:-- --`. Material 22 has `mat-timepicker`, which would work with the same adapter (one `Date` control per field, `localInputToIso` (`dashboard.api.ts:83`) and `isoToLocalInput` (92) replaced). Left out to keep this story small; a follow-up story if wanted.

**STOP HERE. Report to the user.**
