# تقرير اختبار يدوي — الجولة الثانية

**التاريخ:** 2026-10-01 (بعد الظهر) · **البيئة:** محلي، API `dca992c` (`develop`، شغّال من 11:31 بعد آخر commit) · Web `06c817a` (`main`)
**الجولة الأولى:** [2026-10-01-manual-qa-report.md](2026-10-01-manual-qa-report.md) · **خطة الإصلاح:** [../plans/qa-2026-10-01-fix-roadmap.md](../plans/qa-2026-10-01-fix-roadmap.md)
**الأدوات:** Chrome (Claude in Chrome) + `curl` على الـ API
**الحسابات:** Admin (`admin@crm.local`)، و **Agent جديد** بدور "Agent" الافتراضي (`qa.agent.r2@example.test`، فرع Head Office)، وحساب البوابة التجريبي من الجولة الأولى.

> **مهم:** لم يُنفَّذ أي من plans 47–57 بعد (كلها `To do` في `plans/00-index.md`)، والكود هو نفسه الذي اختُبر في الجولة الأولى. لذلك هذه الجولة: (1) تتحقق من إصلاحات 43–46 على build حديث، (2) تؤكّد أن مشاكل الجولة الأولى ما زالت موجودة، (3) **تغطي مناطق لم تُختبر في الجولة الأولى**.

---

## الملخص

| | العدد |
|---|---|
| إصلاحات سابقة تم التحقق منها (43، 44، 45، 46) | 4 / 4 ✅ |
| مشاكل الجولة الأولى ما زالت موجودة (كما هو متوقع) | 16 / 16 (H4 أُصلحت = 15 مفتوحة) |
| **مشاكل جديدة** | **6** (1 عالية، 3 متوسطة، 2 منخفضة) |
| أخطاء Console / أخطاء سيرفر الـ Web | 0 |

---

## 1. التحقق من الإصلاحات السابقة ✅

| Plan | الاختبار | النتيجة |
|------|----------|---------|
| 43 BUG-07 | `PUT /ticket-categories/{A}` بأب = B (و B ابن A) | ✅ 400 `CATEGORY_CYCLE` على `parentId` |
| 44 BUG-08 | نفس الاختبار على `/kb/categories` + أب غير موجود | ✅ 400 `CATEGORY_CYCLE`، و 404 `KB_CATEGORY_NOT_FOUND` |
| 45 BUG-09 | `CHAT_NOT_FOUND` بـ `Accept-Language: ar` | ✅ "المحادثة غير موجودة." |
| 46 BUG-10 | تغيير اسم العميل من الموظف → `GET /portal/me` | ✅ الاسم الجديد يظهر في البوابة |

ملاحظة صغيرة على 44: رسالة الـ 404 للأب غير الموجود بالإنجليزي هي "The category was not found." وليست "The parent category was not found." كما في الخطة، لأن مدخل `KB_CATEGORY_NOT_FOUND` في `Messages.resx` يغطي على رسالة مكان الرمي. (منخفضة، يمكن ضمّها لـ plan 57)

---

## 2. مشاكل الجولة الأولى — ما زالت موجودة

تم التأكد فعلياً (API أو المتصفح) من:

| المشكلة | الدليل في هذه الجولة | Plan |
|---------|----------------------|------|
| H1 لا يوجد نسيت كلمة المرور | `POST /auth/forgot-password` → 404، `/public/portal/forgot-password` → 404، لا رابط في `/portal/login` | 47 |
| H2 المساعد الافتراضي بدون AI | `/public/features` → `chatbot.enabled: true` بينما `/ai/status` → `false`؛ "Virtual assistant" ما زال في شريط البوابة | 48 |
| H3 فشل الإيميل بصمت | الـ outbox: 3 رسائل `Failed` + 2 `Pending` جديدة | 49 |
| L1 "SLA breached" + "in 10 hours" | ظهرت في Dashboard الـ Agent لـ 4 تذاكر | 55 |
| L2 scrollbar في هيدر البوابة | `.portal-bar__nav` scrollHeight 44 > clientHeight 40 | 56 |
| L6 "Agent Agent" مكرر | ما زال في المحادثة | 56 |
| L7 رسالة الـ chatbot العامة | `INVALID_VALUE` "The specified condition was not met for 'Messages'." | 57 |
| M2 اسم التصنيف بالإنجليزي | الـ chip ما زال "Billing" | 51 |

باقي المشاكل (M1، M3، M4، M5، L3، L4، L5، L8) في نفس الكود الذي لم يتغير؛ لم يُعَد اختبارها بالتفصيل.

---

## 3. 🆕 مشاكل جديدة

### N1 🔴 عالية — التسجيل الذاتي في البوابة مستحيل محلياً (ومرتبط بـ H3)
- **الخطوات:** `POST /public/portal/register` → 200 "a verification code has been sent" → `POST /public/portal/login` → `EMAIL_NOT_VERIFIED`.
- **الفعلي:** رسالتا "Your verification code" في الـ outbox بحالة `Pending` ثم تفشل (قناة الإيميل غير مُعدّة). العميل الجديد لا يستطيع إكمال التسجيل بأي طريقة، ولا يرى أي رسالة توضّح المشكلة.
- **الحل:** يُغطّى بـ plan 49 (Log email provider للـ dev). يُقترح إضافة شرط في 49 أن كود التحقق يظهر في الـ log في Development، وتحذير في `/admin/settings` عند تفعيل `portal.registration_enabled` بدون قناة إيميل.

### N2 🟠 متوسطة — الردود الجاهزة لا تملأ بيانات العميل والتذكرة
- **الخطوات:** Agent ينشئ رداً جاهزاً: `Thanks {{customer.name}}, your ticket {{ticket.number}} is being handled by {{agent.name}}.` ← يفتح تذكرة ← يختار الرد من الـ picker.
- **الفعلي:** النص المُدرج: `Thanks {{customer.name}}, your ticket {{ticket.number}} is being handled by QA Round2 Agent.` — فقط `agent.name` تم ملؤه.
- **السبب:** الطلب يذهب إلى `POST /quick-replies/{id}/render` **بدون** `?ticketId=`. الـ picker يقبل `[ticketId]` (`quick-reply-picker.component.ts:127`) لكن لا أحد يمرّره:
  - `features/tickets/ticket-conversation.component.ts:103` → `<app-quick-reply-picker (selected)="insert($event)" />`
  - `features/channels/chat-console.page.html:149` → نفس الشيء
- **الحل:** تمرير `[ticketId]="ticketId()"` في الموضعين (في الـ chat console: ticket id المحادثة المفتوحة). خطر: الموظف قد يرسل `{{customer.name}}` حرفياً للعميل.

### N3 🟠 متوسطة — إنشاء Agent بدون فرع يعطي تطبيقاً فارغاً بدون أي تنبيه
- **الخطوات:** `/admin/users` → New user → دور "Agent" فقط → بدون "Branch and department access" → Create.
- **الفعلي:** الـ Agent يدخل ويرى 0 تذاكر و 0 عملاء (الـ API يرجّع `totalCount: 0`) بدون أي رسالة تشرح السبب. نص المساعدة في الـ dialog: "Leave empty when a role grants access to all branches" — لكن لا تحقق إن كان الدور المختار فعلاً يملك `data.all_branches`.
- **أين:** `features/administration/users/user-dialog.component.ts` (لا يوجد أي تحقق من `data.all_branches` قبل `createUser` في السطر 262).
- **الحل:** (1) في الـ dialog: إذا لم يملك أي دور مختار `data.all_branches` والنطاق فارغ → تحذير أو منع الحفظ؛ (2) في قائمة التذاكر/العملاء: empty state يقول "ليس لديك صلاحية على أي فرع، تواصل مع المدير" عندما يكون نطاق المستخدم فارغاً.

### N4 🟠 متوسطة — تقييم المقالات بلا حدود (يمكن التلاعب به)
- **الخطوات:** `POST /public/kb/articles/{id}/feedback` بـ `{"helpful":false}` 5 مرات متتالية من نفس المصدر.
- **الفعلي:** 5 × 200، و `notHelpfulCount` = 5. لا يوجد rate limit فعّال ولا منع تكرار لنفس الزائر.
- **الحل:** rate limit على الـ endpoint (موجود policy للـ public؟ أضف واحداً أشدّ)، و/أو cookie/fingerprint لمرة واحدة لكل زائر/مقال في الـ help center.

### N5 🟡 منخفضة — استجابة رفع المرفق ترجع `uploadedByName: null`
- `POST /tickets/{id}/attachments` يرجّع `uploadedByName: null` بينما القائمة تعرض "QA Round2 Agent". نفس الشيء في 3 أماكن تمرّر `null` صراحة:
  - `Features/Tickets/TicketMessageSlices.cs:122`
  - `Features/Customers/CustomerDetailSlices.cs:260`
  - `Features/CustomerPortal/PortalTicketSlices.cs:156`
- الأثر: أي واجهة تعرض استجابة الرفع مباشرة تعرض "Unknown uploader" حتى إعادة التحميل.

### N6 🟡 منخفضة — المهمة تقبل تاريخ استحقاق في الماضي
- إنشاء مهمة بـ `dueAt` قبل ساعة → تُحفظ بدون تحذير وتظهر "Overdue" فوراً. يُقترح تحذير (وليس منع) في `task-dialog.component.ts`.

---

## 4. ✅ مناطق جديدة تم اختبارها وتعمل

**الصلاحيات (Agent بدور "Agent"):**
- القائمة الجانبية تعرض 5 صفحات فقط (Dashboard, Tickets, Customers, Live chat, Knowledge base).
- الدخول المباشر لـ `/admin/users`, `/reports/dashboard`, `/sla/policies`, `/tickets/categories`, `/channels`, `/admin/settings` → صفحة "Access denied" (6/6).
- الـ API: 403 على users, roles, audit-logs, reports, integrations, outbox, assignment-rules، و DELETE ticket/customer، و PUT settings.
- قائمة Actions في التذكرة تعرض فقط "Move to Pending" و "Edit" (لا Escalate/Transfer/Delete)؛ الـ Admin يرى الكل.
- خيار "Shared" في الرد الجاهز مخفي عن الـ Agent.
- نطاق الفروع: بدون نطاق = 0 بيانات؛ مع Head Office = 9 تذاكر.

**التذاكر:**
- Escalate (مع سبب) → "Escalated"، مستوى 1. Transfer لفرع آخر → يُلغى إسناد الـ Agent تلقائياً لأنه خارج نطاقه، والـ Agent يحصل على 404 (صحيح، لا إسناد يتيم). Delete مع dialog تأكيد.
- رابط تذكرة غير صالح (`/tickets/ID`) → رسالة "This ticket does not exist or you no longer have access to it."
- المرفقات: رفع من الـ composer + تنزيل بـ `Content-Disposition: attachment` و `X-Content-Type-Options: nosniff`؛ `.exe`/`.html`/`.svg` مرفوضة بـ `FILE_TYPE_NOT_ALLOWED`.

**الأتمتة و SLA:** قاعدة إسناد (تصنيف Billing → Agent محدد + أولوية High) طُبّقت فوراً على تذكرة جديدة، وتم حساب الـ SLA على High (رد أول 60 دقيقة، حل 8 ساعات).

**قاعدة المعرفة:** إنشاء مقال Markdown → Draft → Publish → يظهر في `/help` والبحث (title/summary). **محاولات XSS** (`<script>`, `onerror`, `javascript:` link) تُعرض كنص آمن في المعاينة وفي مركز المساعدة العام.

**التكاملات:**
- API key بنطاقات: القراءة 200، الكتابة بدون نطاق 403، بدون مفتاح / مفتاح خاطئ / JWT موظف → 401، بعد الإلغاء → 401 فوراً.
- Webhooks: `http://` مرفوض؛ عناوين داخلية بـ https تُقبل عند الإنشاء لكن **الإرسال يُمنع** ("resolves to a private or internal address") — الفحص على IP الاتصال الفعلي، فهو آمن ضد DNS rebinding.

**أخرى:** الردود الجاهزة (إنشاء شخصي، placeholders في الـ picker)، المهام (إنشاء، إكمال، تحديث KPIs)، تسجيل الخروج والدخول بين حسابين.

---

## بيانات اختبار (هذه الجولة)

- **أُنشئت وما زالت:** مستخدم `qa.agent.r2@example.test` (دور Agent، Head Office)؛ رد جاهز شخصي "QA thanks"؛ مرفقان على T-000004؛ حساب بوابة غير مُفعّل `qa.register.r2@example.test`؛ مفاتيح API "QA round2 key" (A, B, C) **مُلغاة**؛ اسم العميل C-000005 أصبح "QA Test Customer Renamed".
- **حُذفت/نُظّفت:** قاعدة الإسناد (حُذفت)، مقال KB (أُرشف، 404 للعامة)، تصنيفا KB (حُذفا)، 6 webhooks (حُذفت)، التذكرة T-000016 (حُذفت)، المهمة (مكتملة).
- ملاحظة: قاعدة الـ dev فيها أيضاً بيانات كثيرة من جلسة التحقق الأخرى (QA Branch 1/2 × 3، 9 أدوار QA، 14 مستخدم QA).
