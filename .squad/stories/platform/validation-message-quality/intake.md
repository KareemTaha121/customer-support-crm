# Story intake

- Folder: `.squad/stories/platform/validation-message-quality/intake.md`

---

## Feature

- **Feature name (display):** Platform
- **Feature slug (folder under `plans/`):** `platform`

## Tracker (metadata only)

- **Tracker type:** `none`
- **Work item id:** `BUG-21`
- **Work item type:** `Bug`
- **Status:** `To do`
- **Assignee:** ``
- **Labels:** `backend`, `frontend`, `bug`, `validation`, `localization`

---

## Title

```
Specific, localized validation messages, shown inline on the customer and ticket forms
```

---

## Description

```
Manual QA (2026-10-01) findings L3, L4, L7 and L8:

L7  POST /public/chatbot/messages with a bad body returns INVALID_VALUE
    "The specified condition was not met for 'Messages'." ChatbotValidator
    (Features/Ai/AiSlices.cs ~397) uses Must() without a message. Across the
    API, 9 Must() rules have no code and no message (collection size limits,
    one of them in the chatbot) and 35 more have INVALID_VALUE and no message,
    so they all return the same generic text.

L8  In Arabic, FluentValidation messages keep the English property name
    ("'Customer Id' لا يجب أن يكون فارغاً."). There is no DisplayNameResolver.
    FluentValidation splits the PascalCase property name.

L3  The staff customer form validates email and phone only on the server. The
    contact format is checked in the domain (CustomerContact.NormalizeValue), not
    in CreateCustomerValidator, so the API returns a single 422 without a field
    for the first bad contact. The form shows it in a snackbar, one at a time.

L4  The "Customer" field on New ticket has no required marker (*). Its only
    validator is a custom function, so Angular Material does not detect it.

Fix:
- A ValidatorOptions.Global.DisplayNameResolver backed by Field_<PropertyName>
  entries in Messages.resx / Messages.ar.resx.
- A custom LanguageManager: en/ar text for PredicateValidator (every Must()
  without a message), plus a MaxItems rule with en/ar text for the collection
  limits.
- Chatbot: a specific code CHATBOT_LAST_MESSAGE_NOT_USER (en/ar).
- CreateCustomerValidator checks each contact's email / E.164 phone format, so
  all contact errors come back at once as field errors (contacts[i].value).
- Customer form: client-side Validators.email / E.164 check per contact type;
  server field errors are mapped to the right contact row.
- Ticket create: the Customer control carries Validators.required.
```

---

## Acceptance criteria

```
- [ ] The chatbot with a last turn from "assistant" returns 400 CHATBOT_LAST_MESSAGE_NOT_USER on `messages` with a specific en/ar message.
- [ ] No Must() rule returns "The specified condition was not met …" any more; collection limits name the limit.
- [ ] With Accept-Language: ar, field messages use Arabic field names for the listed properties (e.g. 'العميل', 'الموضوع').
- [ ] Unlisted properties keep FluentValidation's default name (no regression in English).
- [ ] Creating a customer with a bad email and a bad phone returns both errors at once as field errors, shown under the two contact fields.
- [ ] The customer form rejects a bad email / non-E.164 phone before sending.
- [ ] The New ticket "Customer" field shows the required marker.
- [ ] `dotnet build` passes with zero warnings; `npx ng build` passes with zero errors and zero warnings.
```

---

## Attachments

None.

---

## Dependencies

- **Blocked by / related ids:** PL-01 (backend foundation, plan 20), BUG-09 (resx convention, plan 45), FE-03 (customers UI, plan 10), FE-04 (tickets UI, plan 11), AI-01 (chatbot, plan 35). Run last in the QA roadmap; BUG-12 (plan 48) edits `AiSlices.cs` next to the chatbot validator.
- **Depends on code areas or other stories:** source findings L3, L4, L7, L8 in `.squad/qa/2026-10-01-manual-qa-report.md`; `.squad/HANDOFF.md` "Known gaps" (FluentValidation property names).

## Technical hints (optional)

- Repo roots: `customer-support-crm-api/` (branch `develop`, .NET 10, FluentValidation 12.1.1) and `customer-support-crm-web/` (branch `main`, Angular 22).

## Out of scope

- Do NOT touch Docker, docker-compose.yml, deploy/ or .github/ (CI/CD).
- Do NOT create or modify anything under tests/.
- No e2e tests.
- Domain exception texts (422 codes keep their resx messages).
- Server-side format rules for the contact add/edit endpoints (the contact dialog already shows their 422 under the value field).
