# Story 57 — Specific, localized validation messages, shown inline on the customer and ticket forms (Bug: BUG-21)

> Fix plan, not yet implemented. Paths and line numbers refer to customer-support-crm-api `dca992c` (`develop`) and customer-support-crm-web `06c817a` (`main`).
> Intake: [../../stories/platform/validation-message-quality/intake.md](../../stories/platform/validation-message-quality/intake.md)

## Prerequisites

- Story 20 — [20-story-backend-foundation.md](20-story-backend-foundation.md): `ValidationBehavior`, `GlobalExceptionHandler`, the en/ar resx files.
- Story 45 — [45-story-missing-error-messages.md](45-story-missing-error-messages.md): the resx convention. `Messages.ar.resx` has every code; `Messages.resx` has only codes with one fixed English message. Generic field codes keep the validator's message.
- Story 35 — [../ai-features/35-story-ai-assistant-and-chatbot.md](../ai-features/35-story-ai-assistant-and-chatbot.md): `ChatbotValidator`.
- Frontend stories 10 and 11 — [../frontend/10-story-customers-ui.md](../frontend/10-story-customers-ui.md), [../frontend/11-story-tickets-ui.md](../frontend/11-story-tickets-ui.md): the customer form and the ticket create page.
- **Run last** ([roadmap](../qa-2026-10-01-fix-roadmap.md), wave 7). Story 48 changes `ChatbotHandler`, right below `ChatbotValidator` in `AiSlices.cs`. Other stories add resx entries. Re-check line numbers after rebasing.
- Source: manual QA report `.squad/qa/2026-10-01-manual-qa-report.md`, findings L3, L4, L7, L8; `.squad/HANDOFF.md` "Known gaps (low priority)".
- Backend paths are relative to **`customer-support-crm-api/`**, frontend paths to **`customer-support-crm-web/`**.

---

## Story Goal

### How validation messages are produced today

1. `ValidationBehavior` (`Behaviors/ValidationBehavior.cs` 9–31) runs every validator and throws `ValidationException`.
2. `GlobalExceptionHandler.ToApiError` (`Middleware/GlobalExceptionHandler.cs` 70–77) maps each failure:
   - `ToValidationCode` (85–97) turns the FluentValidation code into an API code. `NotEmptyValidator` → `REQUIRED`, …, an UPPER_SNAKE `WithErrorCode` passes through, anything else → `INVALID`.
   - Generic field codes (`GenericFieldCodes`, 79–83: `REQUIRED`, `INVALID_LENGTH`, `INVALID_EMAIL`, `INVALID_FORMAT`, `OUT_OF_RANGE`, `INVALID_VALUE`, `INVALID`) keep FluentValidation's own `ErrorMessage`.
   - Feature codes go through `ErrorResponseWriter.Localize` (resx; `ErrorResponseWriter.cs` 37–42).
3. FluentValidation's built-in `LanguageManager` translates its message per `CultureInfo.CurrentUICulture`. `app.UseRequestLocalization()` (`Program.cs` 45) sets that culture from the request, `en` or `ar`.
4. **Nothing configures FluentValidation globally.** There is no `ValidatorOptions.Global.*`, no `DisplayNameResolver`, no custom `LanguageManager` and no `WithName(...)` in any validator; all `.WithName(...)` hits in `src/` are endpoint names. So `{PropertyName}` is FluentValidation's default: the property name split at capitals ("Customer Id", "Category Name"). The Arabic template is translated, the name is not (L8).

### The four findings

| # | Before | After |
|---|---|---|
| L7 | Chatbot with a last turn from `assistant` → 400 `INVALID_VALUE` "The specified condition was not met for 'Messages'." (`AiSlices.cs` 397) | 400 `CHATBOT_LAST_MESSAGE_NOT_USER` "The last message must be from the user." / "يجب أن تكون آخر رسالة من المستخدم." |
| L7 sweep | 9 `Must()` rules without code or message → `INVALID` + the same generic text; 35 `Must()` rules with `INVALID_VALUE` and no message → the same generic text | Collection limits → `OUT_OF_RANGE` "'Tags' can have at most 20 items."; allowed-value checks → "'Language' has a value that is not allowed." (en/ar) |
| L8 | `ar`: "'Customer Id' لا يجب أن يكون فارغاً." | `ar`: "'العميل' لا يجب أن يكون فارغاً."; `en`: "'Customer' must not be empty." |
| L3 | Customer form: a bad email or phone is caught by the domain on the first bad contact (`Customer.cs` 178 → `NormalizeValue` 278–285 → `DomainException` 422, no `field`). `reportFormError` puts it in a snackbar, one at a time | 400 with one field error per bad contact (`contacts[i].value`), shown under each field. Bad formats are also caught on the client before sending |
| L4 | "Customer" on New ticket has no `*`. Its only validator is `customerSelected`, a custom function (`ticket-create.page.ts` 37–40, 184). Material shows `*` only when the control `hasValidator(Validators.required)` | `Validators.required` is added; the `*` is shown |

### Why the customer form errors arrived one at a time (L3)

`applyServerErrors` exists (`shared/form-errors.ts` 10–26) and the customer form uses it through `reportFormError` (`customer-errors.ts` 12–19, called at `customer-form.page.ts` 413). Inline display works for field errors. Two things go wrong:

1. **No field errors for contact formats.** `CreateCustomerValidator` checks only `NotEmpty` and `MaximumLength` on `contacts[].value` (`CustomerProfileSlices.cs` 49–53). The format check is in the domain (`EmailAddress.Create`, `PhoneNumber.Create`). It throws on the first invalid contact, so the API returns a 422 with no `field`, and the form shows it in a snackbar.
2. **Index shift.** `create()` drops empty contact rows before sending (`customer-form.page.ts` 385). Server index `contacts[1]` is then not form row 1 whenever an earlier row was empty, and the default form has an empty Email row and an empty Phone row (274). The field errors added in this story must be remapped to the form rows.

### Deviation from the intake suggestion

- **Fallback name.** The suggestion was to fall back to the camelCase field name. The resolver instead returns `null` for a property without a `Field_` entry, which keeps FluentValidation's default ("Customer Id"). A camelCase fallback would change every English message for unlisted properties ("'customerId' must not be empty."), and it is no more Arabic than the default.
- **Contact dialog.** The add/edit contact endpoints (`SaveCustomerContactValidator`, `CustomerDetailSlices.cs` 31–39) keep the domain 422. The dialog already shows that 422 under the value field (`contact-dialog.component.ts` 107–112). The dialog only gains the client-side check.

---

## Context — Read These Files First

1. `src/CustomerSupportCrm.Api/Middleware/GlobalExceptionHandler.cs`: `ToApiError` (70–77), `GenericFieldCodes` (79–83), `ToValidationCode` (85–97), `ToFieldPath` (102–105).
2. `src/CustomerSupportCrm.Api/Middleware/ErrorResponseWriter.cs` lines 37–42: `Localize`.
3. `src/CustomerSupportCrm.Application/DependencyInjection.cs`: `AddApplication` (25–38). Validators are registered at line 36 with the default **scoped** lifetime, so they are created inside the request, after request localization.
4. `src/CustomerSupportCrm.Application/Resources/Messages.cs` (marker type, namespace `CustomerSupportCrm.Application.Resources`). The embedded resources are `CustomerSupportCrm.Application.Resources.Messages.resources` and `….ar.resources`.
5. `src/CustomerSupportCrm.Application/Common/Validation/CommonRules.cs`: `ValidSortDirection` (22–23), `OneOf` (25–26).
6. `src/CustomerSupportCrm.Application/Features/Ai/AiSlices.cs`: `AiErrors` (30–36), `ChatbotValidator` (387–400).
7. `src/CustomerSupportCrm.Application/Features/Customers/CustomerProfileSlices.cs`: `CreateCustomerValidator` (38–55), the contact loop in the handler (81–84), `UpdateCustomerValidator` (119–131).
8. `src/CustomerSupportCrm.Domain/Customers/Customer.cs`: `CustomerContact.NormalizeValue` (278–285). `src/CustomerSupportCrm.Domain/Shared/EmailAddress.cs`: `Create` (19–29), `IsWellFormed` (39–47). `src/CustomerSupportCrm.Domain/Shared/PhoneNumber.cs`: `TryCreate` (30–35), the E.164 regex (45–46).
9. `src/CustomerSupportCrm.Application/Features/Integrations/IntegrationSlices.cs` line 385: the only `Must()` with a message (`REQUIRED`, English only).
10. `docs/api-contract.md` lines 72–84: the field-level validation code table.
11. `customer-support-crm-web/src/app/shared/form-errors.ts`: `applyServerErrors` (10–26), `findControl` (28–37), `controlErrorMessage` (43–76).
12. `customer-support-crm-web/src/app/features/customers/customer-errors.ts` (12–19). `customer-form.page.ts`: the contact rows template (167–193) with the value `<mat-error>` at 183, `submit()` (366–377), `create()` (379–390), `save()` (392–416), `contactGroup()` (432–439).
13. `customer-support-crm-web/src/app/features/customers/contact-dialog.component.ts`: the form (81–86), the 422 handling (107–112).
14. `customer-support-crm-web/src/app/features/tickets/ticket-create.page.ts`: `customerSelected` (37–40), the input (71), `customerControl` (184).
15. `customer-support-crm-web/public/i18n/core/en.json` / `ar.json`: `validation` (83–93).

---

## Backend Tasks

### 1 — Global FluentValidation localization

Create `src/CustomerSupportCrm.Application/Common/Validation/ValidationLocalization.cs`:

```csharp
using System.Collections;
using System.Globalization;
using System.Resources;
using CustomerSupportCrm.Application.Resources;
using FluentValidation;
using FluentValidation.Resources;
using FluentValidation.Validators;

namespace CustomerSupportCrm.Application.Common.Validation;

/// <summary>
/// Field display names from Messages.resx / Messages.ar.resx (keys Field_&lt;PropertyName&gt;) and
/// en/ar templates for the rules FluentValidation words generically.
/// </summary>
internal static class ValidationLocalization
{
    public const string FieldKeyPrefix = "Field_";

    private static readonly ResourceManager Resources = new(typeof(Messages).FullName!, typeof(Messages).Assembly);

    public static void Configure()
    {
        // null → FluentValidation's default (the property name split at capitals).
        ValidatorOptions.Global.DisplayNameResolver = (_, member, _) =>
            member is null ? null : Resources.GetString(FieldKeyPrefix + member.Name, CultureInfo.CurrentUICulture);
        ValidatorOptions.Global.LanguageManager = new CrmLanguageManager();
    }
}

internal sealed class CrmLanguageManager : LanguageManager
{
    public CrmLanguageManager()
    {
        // Must() without WithMessage.
        AddTranslation("en", "PredicateValidator", "'{PropertyName}' has a value that is not allowed.");
        AddTranslation("ar", "PredicateValidator", "قيمة '{PropertyName}' غير مسموح بها.");
        AddTranslation("en", MaxItemsValidator.RuleName, "'{PropertyName}' can have at most {MaxItems} items.");
        AddTranslation("ar", MaxItemsValidator.RuleName, "الحد الأقصى لعدد العناصر في '{PropertyName}' هو {MaxItems}.");
    }
}

internal static class MaxItemsValidator
{
    public const string RuleName = "MaxItemsValidator";
}

/// <summary>Collection size limit; null passes (pair with NotNull).</summary>
internal sealed class MaxItemsValidator<T, TCollection>(int max) : PropertyValidator<T, TCollection>
    where TCollection : IEnumerable
{
    public override string Name => MaxItemsValidator.RuleName;

    public override bool IsValid(ValidationContext<T> context, TCollection value)
    {
        if (value is null)
        {
            return true;
        }

        var count = value is ICollection collection ? collection.Count : value.Cast<object>().Count();
        if (count <= max)
        {
            return true;
        }

        context.MessageFormatter.AppendArgument("MaxItems", max);
        return false;
    }

    protected override string GetDefaultMessageTemplate(string errorCode) => Localized(errorCode, Name);
}
```

Notes:
- The resolver reads `CurrentUICulture` each time it runs. Even if FluentValidation resolved the name when the rule is built, validators are scoped (line 36), so they are built inside the request, after `UseRequestLocalization`. Verification step 4 confirms this in Arabic.
- `ResourceManager` falls back to the neutral (English) file, so an `ar` request for a key that only exists in English returns the English value.
- If the compiler reports a nullability mismatch on the resolver lambda (the delegate's return type is non-nullable), return `null!`. `null` is FluentValidation's documented "no override".
- Call `ValidationLocalization.Configure();` at the top of `AddApplication` (`DependencyInjection.cs` 25), before `AddValidatorsFromAssembly`.

### 2 — `MaxItems` rule and mapping

In `CommonRules.cs`, add:

```csharp
public static IRuleBuilderOptions<T, TCollection> MaxItems<T, TCollection>(this IRuleBuilder<T, TCollection> rule, int max)
    where TCollection : System.Collections.IEnumerable =>
    rule.SetValidator(new MaxItemsValidator<T, TCollection>(max));
```

In `GlobalExceptionHandler.ToValidationCode` (85–97), map the new rule to the existing generic range code. Put it with the comparison validators (91–92):

```csharp
"GreaterThanValidator" or … or "ExclusiveBetweenValidator" or "MaxItemsValidator" => ErrorCodes.OutOfRange,
```

`OUT_OF_RANGE` is a generic code, so the localized FluentValidation message (with the field name and the limit) is returned as is.

Replace the 9 `Must()` rules that have no code and no message:

| File:line | Before | After |
|---|---|---|
| `Features/Ai/AiSlices.cs:391` | `.NotEmpty().Must(m => m.Count <= 20)` | `.NotEmpty().MaxItems(20)` |
| `Features/Customers/CustomerProfileSlices.cs:47` | `.NotNull().Must(t => t.Count <= Customer.MaxTags)` | `.NotNull().MaxItems(Customer.MaxTags)` |
| `Features/Customers/CustomerProfileSlices.cs:48` | `.NotNull().Must(c => c.Count <= 50)` | `.NotNull().MaxItems(50)` |
| `Features/Customers/CustomerProfileSlices.cs:129` | Tags, as line 47 | `.MaxItems(Customer.MaxTags)` |
| `Features/CustomerPortal/PortalTicketSlices.cs:133` | `.NotNull().Must(a => a.Count <= 10)` | `.NotNull().MaxItems(10)` |
| `Features/Tickets/TicketCommandSlices.cs:47` | `.NotNull().Must(t => t.Count <= Ticket.MaxTags)` | `.NotNull().MaxItems(Ticket.MaxTags)` |
| `Features/Tickets/TicketCommandSlices.cs:177` | same | same |
| `Features/Tickets/TicketMessageSlices.cs:72` | `.NotNull().Must(m => m.Count <= 20)` | `.NotNull().MaxItems(20)` |
| `Features/Tickets/TicketMessageSlices.cs:73` | `.NotNull().Must(a => a.Count <= 10)` | `.NotNull().MaxItems(10)` |

Each file already has `using FluentValidation;`. Add `using CustomerSupportCrm.Application.Common.Validation;` where it is missing.

### 3 — Must() rules with `INVALID_VALUE` and no message (35, listed for the record)

These rules check allowed values (enum names, `en`/`ar`, fixed sets) and keep their code. With the `PredicateValidator` override from task 1, they now return "'X' has a value that is not allowed." / "قيمة 'X' غير مسموح بها." with the localized field name. **No code change per rule.**

- `Common/Validation/CommonRules.cs` 23, 26 (`ValidSortDirection`, `OneOf`, used by many list queries)
- `Features/Ai/AiSlices.cs` 199 (`Tone`), 394 (chatbot `Role`), 398 (chatbot `Language`)
- `Features/Channels/CustomerMessaging.cs` 299 · `Features/Channels/InboundChannels.cs` 232 · `Features/CustomerPortal/PortalAccountSlices.cs` 72
- `Features/Customers/CustomerProfileSlices.cs` 45, 126 · `Features/Dashboard/AgentWorkspaceSlices.cs` 351
- `Features/Integrations/IntegrationSlices.cs` 318, 319, 384
- `Features/KnowledgeBase/KnowledgeBaseSlices.cs` 203, 204, 205, 268, 458
- `Features/Organization/OrganizationProfile.cs` 60, 62
- `Features/Sla/SlaAdministration.cs` 189, 190, 191, 278, 279, 280
- `Features/Tickets/TicketCategorySlices.cs` 50 · `Features/Tickets/TicketCommandSlices.cs` 46, 270 · `Features/Tickets/TicketQuerySlices.cs` 52, 53, 54, 55

Do not add English `WithMessage` texts to these rules. A custom message replaces the `LanguageManager` template, so Arabic would show English.

Already specific, unchanged: `Features/Roles/Common/RoleDefinitionValidator.cs` 25 (`UNKNOWN_PERMISSION`, en/ar resx).

**`IntegrationSlices.cs` 385:** `Must(...).WithErrorCode(ErrorCodes.Required).WithMessage("Provide customerId or customerEmail.")`. `REQUIRED` is generic, so Arabic gets this English text. Change it to a feature code:

- add `public const string CustomerReferenceRequired = "CUSTOMER_REFERENCE_REQUIRED";` to `IntegrationMapping`, next to `ApiKeyNotFound` / `WebhookNotFound` (`IntegrationSlices.cs` 40–44), and use it in `WithErrorCode`;
- keep the English message;
- add resx entries: en "Provide customerId or customerEmail." / ar "أرسل customerId أو customerEmail.".

### 4 — Chatbot (L7)

In `AiErrors` (`AiSlices.cs` 30–36), add `public const string LastMessageNotUser = "CHATBOT_LAST_MESSAGE_NOT_USER";`. Rewrite `ChatbotValidator` (387–400):

```csharp
RuleFor(c => c.Messages).NotEmpty().MaxItems(20);
RuleForEach(c => c.Messages).ChildRules(turn =>
{
    turn.RuleFor(t => t.Role).Must(r => r is "user" or "assistant").WithErrorCode(ErrorCodes.InvalidValue);
    turn.RuleFor(t => t.Content).NotEmpty().MaximumLength(2000);
});
RuleFor(c => c.Messages)
    .Must(m => m[^1].Role == "user")
    .When(c => c.Messages.Count > 0)            // an empty list already fails NotEmpty
    .WithErrorCode(AiErrors.LastMessageNotUser)
    .WithMessage("The last message must be from the user.");
RuleFor(c => c.Language).Must(l => l is null or "en" or "ar").WithErrorCode(ErrorCodes.InvalidValue);
```

`Messages` is never null: the endpoint passes `request.Messages ?? []` (line 508). `CHATBOT_LAST_MESSAGE_NOT_USER` is a feature code, so `ToApiError` localizes it from resx. It has one fixed English message, so per the plan 45 convention it goes in **both** files:

| Key | en | ar |
|---|---|---|
| `CHATBOT_LAST_MESSAGE_NOT_USER` | The last message must be from the user. | يجب أن تكون آخر رسالة من المستخدم. |

Add it next to `AI_SUGGESTION_NOT_FOUND` (`Messages.resx` 240).

### 5 — Contact formats as field errors (L3)

**Domain helper (`EmailAddress.cs`):** add a public check that reuses `Create`'s rules:

```csharp
public static bool IsValid(string? value)
{
    var normalized = Normalize(value);
    return normalized.Length is > 0 and <= MaxLength && IsWellFormed(normalized);
}
```

`PhoneNumber.TryCreate` (30–35) already exists.

**`CreateCustomerValidator` (`CustomerProfileSlices.cs` 49–53):** extend the child rules:

```csharp
RuleForEach(c => c.Contacts).ChildRules(contact =>
{
    contact.RuleFor(x => x.Type).NotEmpty().IsEnumName(typeof(ContactType), caseSensitive: false);
    contact.RuleFor(x => x.Value).NotEmpty().MaximumLength(CustomerContact.ValueMaxLength);
    contact.RuleFor(x => x.Value)
        .Must(EmailAddress.IsValid)
        .When(x => IsType(x.Type, ContactType.Email) && !string.IsNullOrWhiteSpace(x.Value))
        .WithErrorCode(EmailAddress.InvalidCode)
        .WithMessage("The email address is not valid.");
    contact.RuleFor(x => x.Value)
        .Must(v => PhoneNumber.TryCreate(v, out _))
        .When(x => (IsType(x.Type, ContactType.Phone) || IsType(x.Type, ContactType.WhatsApp)) && !string.IsNullOrWhiteSpace(x.Value))
        .WithErrorCode(PhoneNumber.InvalidCode)
        .WithMessage("The phone number must be in international format, e.g. +966501234567.");
});

private static bool IsType(string? type, ContactType expected) =>
    Enum.TryParse<ContactType>(type, ignoreCase: true, out var parsed) && parsed == expected;
```

- `INVALID_EMAIL_ADDRESS` and `INVALID_PHONE_NUMBER` already have en/ar resx entries (`Messages.resx` 102, 159; `Messages.ar.resx` 102, 171). They are feature codes, so the localized resx text is used. The English messages above are the same as the domain's.
- Field paths come out as `contacts[0].value` (`ToFieldPath`, 102–105). Every bad contact is reported in one response.
- The domain checks stay as the last line of defence.
- The `Domain.Shared` and `Domain.Customers` namespaces: check the file's `using` list and add what is missing.

### 6 — Field display names (L8)

Add `Field_<PropertyName>` entries to **both** resx files, in one block at the end before `</root>`. Keep each file's encoding (`Messages.resx` ASCII/UTF-8, `Messages.ar.resx` UTF-8) and line endings. Keys use the **last member name** of the rule expression, which is what the resolver receives (`c => c.Request.Name` → `Field_Name`; child rules `x => x.Value` → `Field_Value`).

The list below covers every final member name used in a `RuleFor` / `RuleForEach` in `src/CustomerSupportCrm.Application` at `dca992c`. Re-run this sweep after rebasing:

```bash
grep -rhoE "Rule(For|ForEach)\(\s*\w+\s*=>\s*[A-Za-z_.]+" --include=*.cs src/CustomerSupportCrm.Application | sed -E 's/.*\.//; s/.*=>\s*//' | sort -u
```

| Key | en | ar |
|---|---|---|
| `Field_AccentColor` | Accent color | اللون الثانوي |
| `Field_Action` | Action | الإجراء |
| `Field_Address` | Address | العنوان |
| `Field_AfterMinutes` | Minutes after | بعد (بالدقائق) |
| `Field_Assignee` | Assignee | المسؤول |
| `Field_AttachmentIds` | Attachments | المرفقات |
| `Field_Body` | Message | نص الرسالة |
| `Field_BranchId` | Branch | الفرع |
| `Field_Category` | Category | التصنيف |
| `Field_Channel` | Channel | القناة |
| `Field_Code` | Code | الرمز |
| `Field_Comment` | Comment | التعليق |
| `Field_CompanyName` | Company name | اسم الشركة |
| `Field_Contacts` | Contacts | جهات الاتصال |
| `Field_Content` | Content | المحتوى |
| `Field_CurrentPassword` | Current password | كلمة المرور الحالية |
| `Field_CustomerEmail` | Customer email | بريد العميل |
| `Field_CustomerId` | Customer | العميل |
| `Field_DefaultCulture` | Default language | اللغة الافتراضية |
| `Field_DefaultPriority` | Default priority | الأولوية الافتراضية |
| `Field_Description` | Description | الوصف |
| `Field_DisplayName` | Display name | الاسم الظاهر |
| `Field_Email` | Email | البريد الإلكتروني |
| `Field_EntityId` | Entity ID | معرّف الكيان |
| `Field_EntityType` | Entity type | نوع الكيان |
| `Field_ExternalId` | External ID | المعرّف الخارجي |
| `Field_FirstResponseMinutes` | First response (minutes) | مدة الرد الأول (بالدقائق) |
| `Field_Instructions` | Instructions | التعليمات |
| `Field_Label` | Label | التسمية |
| `Field_Language` | Language | اللغة |
| `Field_MatchChannel` | Channel condition | شرط القناة |
| `Field_MatchKeyword` | Keyword condition | شرط الكلمة المفتاحية |
| `Field_MatchPriority` | Priority condition | شرط الأولوية |
| `Field_MentionedUserIds` | Mentioned users | المستخدمون المشار إليهم |
| `Field_Message` | Message | الرسالة |
| `Field_Messages` | Messages | الرسائل |
| `Field_Name` | Name | الاسم |
| `Field_NameAr` | Arabic name | الاسم بالعربية |
| `Field_NewPassword` | New password | كلمة المرور الجديدة |
| `Field_Notes` | Notes | الملاحظات |
| `Field_Page` | Page | الصفحة |
| `Field_PageSize` | Page size | حجم الصفحة |
| `Field_Password` | Password | كلمة المرور |
| `Field_Permissions` | Permissions | الصلاحيات |
| `Field_Phone` | Phone | رقم الهاتف |
| `Field_PreferredLanguage` | Preferred language | اللغة المفضلة |
| `Field_PrimaryColor` | Primary color | اللون الأساسي |
| `Field_Priority` | Priority | الأولوية |
| `Field_RaisePriorityTo` | Raise priority to | رفع الأولوية إلى |
| `Field_Rating` | Rating | التقييم |
| `Field_Reason` | Reason | السبب |
| `Field_ResolutionMinutes` | Resolution (minutes) | مدة الحل (بالدقائق) |
| `Field_Role` | Role | الدور |
| `Field_RoleIds` | Roles | الأدوار |
| `Field_Scopes` | Scopes | النطاقات |
| `Field_Search` | Search | البحث |
| `Field_SetPriority` | Priority to set | الأولوية الجديدة |
| `Field_Shortcut` | Shortcut | الاختصار |
| `Field_Sla` | SLA | اتفاقية مستوى الخدمة |
| `Field_Slug` | Slug | الرابط المختصر |
| `Field_SortBy` | Sort by | الترتيب حسب |
| `Field_SortDirection` | Sort direction | اتجاه الترتيب |
| `Field_Status` | Status | الحالة |
| `Field_Strategy` | Strategy | طريقة التوزيع |
| `Field_Subject` | Subject | الموضوع |
| `Field_Summary` | Summary | الملخص |
| `Field_SupportEmail` | Support email | بريد الدعم |
| `Field_SupportPhone` | Support phone | هاتف الدعم |
| `Field_System` | System | النظام |
| `Field_Tags` | Tags | الوسوم |
| `Field_Target` | Target | الهدف |
| `Field_Targets` | Targets | الأهداف |
| `Field_Ticket` | Ticket | التذكرة |
| `Field_TimeZone` | Time zone | المنطقة الزمنية |
| `Field_Title` | Title | العنوان |
| `Field_To` | Recipient | المستلم |
| `Field_Tone` | Tone | الأسلوب |
| `Field_Trigger` | Trigger | المُشغِّل |
| `Field_Type` | Type | النوع |
| `Field_Types` | Types | الأنواع |
| `Field_Value` | Value | القيمة |
| `Field_Visibility` | Visibility | مستوى الظهور |
| `Field_Website` | Website | الموقع الإلكتروني |
| `Field_WorkEnd` | Work end | نهاية الدوام |
| `Field_WorkStart` | Work start | بداية الدوام |

The English values replace FluentValidation's split names ("Customer Id" → "Customer", "Role Ids" → "Roles"). Wording is open to review; keep the keys.

`Field_*` keys are not error codes. The plan 45 sweep (error codes vs resx keys) must ignore the `Field_` prefix.

### 7 — Docs

`docs/api-contract.md`, "Field-level validation codes" (72–84):

- add `MaxItems` (collection size) to the `OUT_OF_RANGE` row;
- add one line under the table: "Field names in `message` are localized from `Field_<PropertyName>` resx entries; `field` stays the camelCase request path."

**No changes to:** migrations, contracts, `ValidationBehavior` or domain exception texts.

## Frontend Tasks

### 1 — Contact value validator (new `features/customers/contact-validators.ts`)

```ts
import { AbstractControl, ValidationErrors, ValidatorFn, Validators } from '@angular/forms';
import { ContactType } from './customers.models';

/** Domain/Shared/PhoneNumber.cs: keep digits and '+', leading 00 → +, then E.164. */
const E164 = /^\+[1-9]\d{6,14}$/;

export function normalizePhone(value: string): string {
  const digits = value.replace(/[^\d+]/g, '');
  return digits.startsWith('00') ? `+${digits.slice(2)}` : digits;
}

/** Email or E.164 phone check that follows the row's current contact type; empty passes. */
export function contactValueValidator(type: () => ContactType | null | undefined): ValidatorFn {
  return (control: AbstractControl): ValidationErrors | null => {
    const value = String(control.value ?? '').trim();
    if (!value) {
      return null;
    }
    switch (type()) {
      case 'Email':
        return Validators.email(control);
      case 'Phone':
      case 'WhatsApp':
        return E164.test(normalizePhone(value)) ? null : { phone: true };
      default:
        return null;
    }
  };
}
```

Angular's `Validators.email` accepts `a@b`; the server also requires a dot in the domain. Those cases still come back as a server field error, shown inline by task 3.

### 2 — Shared error text

- In `shared/form-errors.ts` `controlErrorMessage` (43–76), add before the `pattern` branch: `if (errors['phone']) { return t.t('core.validation.phone'); }`.
- In `public/i18n/core/en.json` `validation`, add `"phone": "Use international format, e.g. +966501234567."`.
- In `ar.json`, add `"phone": "استخدم الصيغة الدولية، مثل ‎+966501234567"`. Keep the left-to-right mark before `+`, as in `customers.contacts.phoneHint` (`public/i18n/customers/ar.json` 101).

### 3 — Customer form (`features/customers/customer-form.page.ts`)

1. **Client check.** In `contactGroup()` (432–439):

   ```ts
   const group = this.fb.group({ /* unchanged */ });
   group.controls.value.addValidators(contactValueValidator(() => group.controls.type.value));
   group.controls.type.valueChanges.pipe(takeUntilDestroyed(this.destroyRef)).subscribe(() => group.controls.value.updateValueAndValidity());
   return group;
   ```

   `destroyRef` is declared (245) before `form` (265), so it is available when the field initializer calls `contactGroup`. `submit()` already stops on `form.invalid` (367).
2. **Index remap.** In `create()` (379–390), keep the form index of each contact sent:

   ```ts
   const sent = value.contacts.map((c, index) => ({ c, index })).filter(({ c }) => c.value.trim());
   this.sentContactRows = sent.map(({ index }) => index);
   // contacts: sent.map(({ c }) => ({ type: c.type, value: c.value.trim(), label: c.label.trim() || null, isPrimary: c.isPrimary })),
   ```

   Add the field `private sentContactRows: number[] = [];`.
3. In `save()` (392–416), before `reportFormError(...)` at 413, remap: `reportFormError(this.form, remapIndexedFields(apiError, 'contacts', this.sentContactRows), this.toast, this.translations);`. The update path sends no contacts, so the remap does nothing there.
4. In `customer-errors.ts`, add:

   ```ts
   /** Rewrites `collection[i]` in field paths to `collection[indexes[i]]` (the client dropped some rows before sending). */
   export function remapIndexedFields(error: ApiError, collection: string, indexes: readonly number[]): ApiError {
     const pattern = new RegExp(`^${collection}\\[(\\d+)\\]`, 'i');
     const errors = error.errors.map((item) => {
       const match = item.field ? pattern.exec(item.field) : null;
       const target = match ? indexes[Number(match[1])] : undefined;
       return item.field && target !== undefined ? { ...item, field: item.field.replace(pattern, `${collection}[${target}]`) } : item;
     });
     return new ApiError(error.status, error.code, error.message, errors, error.correlationId);
   }
   ```

   `applyServerErrors` → `findControl` turns `contacts[1].value` into `contacts.1.value` (`form-errors.ts` 29) and `form.get` resolves the FormArray row. All errors match, so `reportFormError` shows no snackbar (`customer-errors.ts` 15).

### 4 — Contact dialog (`features/customers/contact-dialog.component.ts`)

Add the same client check. In a constructor:

```ts
constructor() {
  this.form.controls.value.addValidators(contactValueValidator(() => this.form.getRawValue().type));
  this.form.controls.type.valueChanges.pipe(takeUntilDestroyed()).subscribe(() => this.form.controls.value.updateValueAndValidity());
}
```

`getRawValue()` because `type` is disabled when editing (82). The 422 handling (107–112) stays.

### 5 — Ticket create (`features/tickets/ticket-create.page.ts`, L4)

At line 184: `new FormControl<CustomerOption | string | null>(null, { validators: [Validators.required, customerSelected] })`.

`MatInput.required` reads `control.hasValidator(Validators.required)` (Material 22, `input.mjs` line 83), so the label shows `*`. Both validators report `required`, so the `<mat-error>` text (84) is unchanged.

---

## Test Plan

Test projects are **out of scope**: `tests/` must not be modified.

---

## Verification Steps

1. **Backend build:** `dotnet build src/CustomerSupportCrm.Api/CustomerSupportCrm.Api.csproj` → 0 warnings, 0 errors. If a running API locks `bin/`, build with `-o <temp dir>`.
2. **Frontend build:** `cd customer-support-crm-web && npx ng build` → 0 errors, 0 warnings.
3. **L7:**
   - `POST /api/v1/public/chatbot/messages` with `{"messages":[{"role":"assistant","content":"hi"}]}` → 400. The error has code `CHATBOT_LAST_MESSAGE_NOT_USER`, field `messages` and message "The last message must be from the user.". With `Accept-Language: ar` → "يجب أن تكون آخر رسالة من المستخدم.".
   - 21 user turns → `OUT_OF_RANGE` "'Messages' can have at most 20 items.".
   - `"language":"fr"` → `INVALID_VALUE` "'Language' has a value that is not allowed.".
4. **L8:** with `Accept-Language: ar`:
   - `POST /api/v1/tickets` with `{"subject":"","priority":"Medium"}` → `customerId` "'العميل' لا يجب أن يكون فارغاً." and `subject` "'الموضوع' لا يجب أن يكون فارغاً.".
   - `POST /api/v1/ticket-categories` with an empty name → "'الاسم' …".
   - The same requests in English show "'Customer' must not be empty.".
5. **Unlisted names:** temporarily remove one `Field_` entry (or pick a property not in the table): the message shows FluentValidation's default name, as before. Restore the entry.
6. **L3 API:** `POST /api/v1/customers` with contacts `[{type:"Email",value:"bad"},{type:"Phone",value:"0501234567"}]` → one 400 with two errors: `contacts[0].value` `INVALID_EMAIL_ADDRESS` and `contacts[1].value` `INVALID_PHONE_NUMBER`. Before the fix: one 422.
7. **L3 UI:**
   - In `/customers/new`, type `bad` in the Email row and `0501234567` in the Phone row. Both fields show an error on blur and Create does nothing.
   - **Index remap:** leave row 1 (Email) empty, type a valid phone in row 2 (Phone), add a third row of type Email with `a@b`. Angular's email check accepts it; the server rejects it. The request sends 2 contacts, and the server error on `contacts[1].value` appears under the **third** row, with no snackbar.
8. **L4:** `/tickets/new` → the "Customer" label shows `*`. Submitting empty shows "This field is required." under it.
9. **Contact dialog:** on a customer, add a Phone contact `123` → inline client error, no request sent.

---

## Done Criteria

- [ ] The chatbot returns `CHATBOT_LAST_MESSAGE_NOT_USER` (en/ar) for a last turn not from the user.
- [ ] The 9 collection limits use `MaxItems` (`OUT_OF_RANGE`, with the limit). The 35 allowed-value rules return the en/ar `PredicateValidator` text. `IntegrationSlices.cs` 385 uses `CUSTOMER_REFERENCE_REQUIRED` (en/ar).
- [ ] `ValidatorOptions.Global.DisplayNameResolver` uses `Field_*` resx entries; Arabic messages show Arabic field names; unlisted properties keep the default.
- [ ] `POST /customers` returns all contact format errors at once as `contacts[i].value` field errors, and the form shows them under the right rows.
- [ ] The customer form and the contact dialog check email / E.164 phone on the client.
- [ ] The ticket create "Customer" field shows the required marker.
- [ ] `docs/api-contract.md` updated.
- [ ] Nothing changed in `tests/`, `docker-compose.yml`, `deploy/` or `.github/`.
- [ ] `dotnet build` and `npx ng build` pass with zero warnings.

**STOP HERE. Report to the user.**
