# تقرير اختبار يدوي — Customer Support CRM

**التاريخ:** 2026-10-01 · **البيئة:** محلي (API `https://localhost:5001` · Web `http://localhost:4200` · PostgreSQL 18)
**الأدوات:** Chrome (Claude in Chrome) + المتصفح المدمج لمحاكاة الموبايل + `curl` على الـ API
**خطة الإصلاح:** [../plans/qa-2026-10-01-fix-roadmap.md](../plans/qa-2026-10-01-fix-roadmap.md) (BUG-11 → BUG-21, plans 47–57)
**الحساب:** `admin@crm.local` (Bootstrap admin من `appsettings.Development.json`) + حساب بوابة تجريبي لعميل QA

> ملاحظة بيئة: كانت هناك جلسة أخرى تعدّل الكود أثناء الاختبار (`category-dialog.component.ts` و `KnowledgeBaseSlices.cs` غير مُلتزمين، وcommit `cedce62` لإصلاح دورات التصنيفات). هذا سبّب إعادة تحميل الصفحة عدة مرات وبناء فاشل مؤقت (`TS2304: Cannot find name 'excludeSubtree'`) ثم نجح.

---

## الملخص

| الخطورة | العدد |
|---------|-------|
| 🔴 عالية | 4 |
| 🟠 متوسطة | 5 |
| 🟡 منخفضة | 8 |

كل الـ 27 شاشة للموظفين تفتح بدون أخطاء Console وبدون أي طلب API فاشل (4xx/5xx)، ولا يوجد overflow أفقي على عرض 375px. المسارات الأساسية (عميل → تذكرة → رد → ملاحظة داخلية → إسناد → تصنيف → حل → تقييم → رد العميل يعيد فتح التذكرة → محادثة مباشرة realtime) شغالة.

---

## 🔴 عالية

### H1. لا يوجد "نسيت كلمة المرور" إطلاقاً (موظفين + بوابة العملاء)
- **المتوقع:** الـ spec يطلبها صراحة: `.squad/features/10-security-and-administration.md:64` — *"Login, forgot/reset password."*
- **الفعلي:** لا يوجد endpoint ولا شاشة ولا رابط في `/login` أو `/portal/login`. البحث في الكود عن `forgot` لا يرجع شيئاً.
- **يزيد الأمر سوءاً:** المقال العام في مركز المساعدة `/help/articles/how-to-reset-your-password` يقول للعميل *"Click Forgot password"* — زر غير موجود.
- **الحل:** story جديدة: `POST /auth/forgot-password` + `/auth/reset-password` (توكن لمرة واحدة، صلاحية قصيرة) ونفسها للبوابة، + روابط في صفحتي الدخول.

### H2. "المساعد الافتراضي" ظاهر للعملاء ويرد بـ "AI features are not configured."
- **الخطوات:** البوابة → Virtual assistant → اسأل أي سؤال.
- **الفعلي:** الرد: *"AI features are not configured."* — رسالة إعدادات داخلية تظهر للعميل.
- **السبب:** `GET /public/features` يرجع `chatbot.enabled: true` من الإعداد فقط، بدون التحقق من وجود AI provider (`AnthropicAiClient.IsConfigured` = false لأنه لا يوجد `ApiKey` / `ANTHROPIC_API_KEY`).
- **نفس المشكلة في الإدارة:** في `/admin/settings` مفتاح "AI agent assist" مفعّل، لكن `GET /ai/status` يرجع `enabled:false`، فلوحة الـ AI مختفية من التذكرة بدون أي تنبيه للأدمن.
- **الحل:** `public/features` يرجّع `chatbot.enabled = setting && ai.IsConfigured`؛ وفي صفحة Settings يظهر تحذير "لا يوجد AI provider مُعدّ" بجانب المفتاحين.

### H3. الإيميلات تفشل بصمت — الموظف والعميل يظنان أنها وصلت
- **الخطوات:** رد عام على تذكرة → snackbar "Reply sent." / نموذج "Contact us" → *"We will reply by email."*
- **الفعلي:** `GET /channels/outbox` فيه 3 رسائل `status: Failed`، `attempts: 1`، `lastError: "The Email channel is not configured."`. لا شيء في واجهة التذكرة يوضّح أن الرد لم يُرسَل للعميل.
- **الحل:** (1) مؤشر على الرسالة في المحادثة (Sent / Failed + retry)؛ (2) banner في الـ dashboard أو صفحة Channels عند وجود قناة غير مُعدّة ورسائل فاشلة؛ (3) للـ dev: provider وهمي (log/file) بدل الفشل.

### H4. تصنيفات بدورات (A → B → A) — قيد الإصلاح
- **الفعلي (على الـ API الشغال):** `PUT /ticket-categories/{A}` بـ `parentId = B` (و B ابن A) رجع **200**.
- **الحالة:** الإصلاح `cedce62 fix(tickets): reject ticket category parents that would create a cycle` نزل بعد تشغيل الـ API بـ 3 دقائق، وجلسة أخرى تعمل حالياً على تصنيفات الـ KB (تعديلات غير مُلتزمة). **المطلوب:** إعادة تشغيل الـ API والتأكد من الحالتين، ثم commit لتعديلات الـ KB.
- بيانات الاختبار أُصلحت (أُزيلت الدورة، وتم تعطيل `QA Cycle A/B`).

---

## 🟠 متوسطة

### M1. رسالة "This field is required." تظهر بعد الإرسال الناجح
- **أين:** رد العميل في البوابة، محادثة البوابة المباشرة، وconsole المحادثة عند الموظف.
- **الفعلي:** بعد الإرسال الناجح يُفرّغ الحقل ويظهر تحته خطأ أحمر فوراً.
- **السبب:** `form.reset()` لا يصفّر حالة `submitted` في `FormGroupDirective`، فيعتبر الـ `ErrorStateMatcher` في Material أن الحقل خاطئ.
  - `features/customer-portal/tickets/portal-ticket-detail.page.ts:152`
  - `features/customer-portal/channels/portal-chat.page.ts:195`
  - `features/channels/chat-console.page.ts:195, 294`
  - على الأغلب أيضاً `portal-chatbot.page.ts:89, 107`
- **الحل:** استخدام `@ViewChild(FormGroupDirective) ... .resetForm()` بدل `form.reset()`.

### M2. بالعربي: اسم التصنيف يظهر بالإنجليزي
- **الفعلي:** تذكرة بتصنيف "Billing" تظهر بـ "Billing" وليس "الفواتير" (التصنيف عنده `nameAr`)، في: الـ chip أعلى التذكرة، اللوحة الجانبية، عمود القائمة، سجل التذكرة، وتقارير "Top categories".
- **السبب:** الـ DTO يرجّع `categoryName` فقط:
  - `features/tickets/ticket-details.page.html:62`
  - `features/tickets/ticket-side-panel.component.ts:148`
  - `features/tickets/ticket-list.page.html:134`
- **الحل:** إضافة `categoryNameAr` في `TicketSummary`/`TicketDetails` والتقارير، واختيار الاسم حسب اللغة (مثل ما يفعل `categoryName(category)` في القوائم المنسدلة).

### M3. نص مقلوب في سجل التذكرة بالعربي (Bidi)
- **الفعلي:** `Administrator · 01/10/2026، 6:53 ص` يظهر هكذا: `01 · 2026/10/Administrator`.
- **أين:** `features/tickets/ticket-history.component.ts:45`
- **الحل:** وضع اسم المنفّذ والتاريخ داخل `<bdi>` كلٌّ على حدة (أو `dir="auto"`). نفس النمط `{{ name }} · {{ date }}` موجود في `ticket-attachments.component.ts:29` ويستحق نفس الإصلاح.

### M4. أول رسالة من الزائر في المحادثة المباشرة لا تظهر لأحد
- **الفعلي:** عند الموظف: *"No messages yet."* بعد قبول المحادثة، فلا يعرف ماذا طلب العميل. وعند العميل أيضاً لا تظهر رسالته الأولى.
- مسجّلة في HANDOFF كـ "Known gap (low priority)" لكن أثرها على الموظف كبير، وأقترح رفعها.
- **الحل:** إضافة وصف التذكرة كأول رسالة في `GET /chat/conversations/{id}/messages` وفي الـ transcript العام.

### M5. سجل التدقيق (Audit) غير مقروء
- عمود "الكيان" يعرض GUID خام (`01a0f596-c0bf-...`) بدل اسم/رقم الكيان (C-000005, T-000004).
- `customers.created`، `ticket_categories.updated`، `TicketCategory` بالإنجليزي حتى في الواجهة العربية.
- **الحل:** ترجمة رموز الإجراءات وأنواع الكيانات (مفاتيح i18n)، وإرجاع `entityLabel` من الـ API.

---

## 🟡 منخفضة

| # | المشكلة | المكان / الحل |
|---|---------|----------------|
| L1 | في الـ Dashboard، كارت "SLA at risk" يعرض **"SLA breached"** وبجانبه **"in 15 hours"**. السبب أن الرد الأول متأخر، لكن الوقت المعروض هو موعد الحل. ونفس الـ KPI يعدّ التذاكر المتأخرة ضمن "at risk". | اعرض الموعد المتأخر نفسه ("الرد الأول متأخر منذ 7 ساعات") وافصل breached عن at-risk |
| L2 | شريط تمرير عمودي (▲▼) في هيدر البوابة بجانب "Help center". | `portal-shell.component.ts:69`: أضف `overflow-y: hidden` (لأن `overflow-x:auto` يجعل y = auto، والأزرار 44px داخل nav ارتفاعه 40px) |
| L3 | التحقق من الإيميل والجوال في نموذج العميل يتم على السيرفر فقط، ويظهر خطأ واحد كل مرة في snackbar بدل ظهوره تحت الحقل. | أضف `Validators.email` / pattern للـ E.164 في الفرونت، واربط `errors[].field` من الـ API بالحقل |
| L4 | حقل "Customer" في New ticket غير معلَّم كإجباري (*) رغم أن الـ API يطلبه. | `ticket-create.page.ts` |
| L5 | تنسيقات التاريخ مختلطة في نفس الصفحة: `1 Oct 2026, 06:53` / `01/10/2026, 06:53` / "14 seconds ago"، وحقول التاريخ `mm/dd/yyyy` بينما العرض `dd/mm/yyyy`. | توحيد `localDate` pipe |
| L6 | في المحادثة: "Administrator Agent **[Agent]**"، أي أن "Agent" مكرر (نص + chip). | قالب الرسالة في ticket conversation |
| L7 | رسالة الـ API للـ chatbot: *"The specified condition was not met for 'Messages'."* رسالة عامة غير مفيدة. | رسالة مخصصة في validator الـ chatbot |
| L8 | رسائل FluentValidation بالعربي تبقي اسم الحقل بالإنجليزي (`'Subject' لا يجب أن يكون فارغاً`). مذكورة في HANDOFF. | `WithName(...)` مترجم أو `DisplayNameResolver` |

---

## ✅ ما تم اختباره ويعمل

- **الدخول:** التحقق من صيغة الإيميل، رسالة "كلمة مرور خاطئة"، الدخول، `auth/refresh` بعد التحديث، تسجيل الخروج، والـ guard يعيد توجيه `/tickets` إلى `/login?returnUrl=...`.
- **التذاكر:** البحث بالموضوع وبالرقم (مع debounce)، الفلاتر، الإنشاء، Assign to me، رد عام (New → Open)، ملاحظة داخلية، تغيير التصنيف، Resolve، السجل، حساب الـ SLA.
- **العملاء:** إنشاء العميل وجهات الاتصال، الملاحظات، منح الوصول للبوابة.
- **البوابة:** الدخول، القائمة (Resolved تُعدّ ضمن Closed)، **الملاحظة الداخلية مخفية عن العميل**، التقييم 4/5، رد العميل يعيد فتح التذكرة ويرسل notification للموظف، نموذج Contact us (T-000006 مرتبطة بنفس العميل).
- **المحادثة المباشرة:** الطابور، القبول، والرسائل في الاتجاهين **realtime عبر SignalR**.
- **مركز المساعدة:** المقالات العامة تظهر، والمقال الداخلي يرجع "not found" للزوار.
- **التقارير:** الأرقام صحيحة (6 مفتوحة، 4 بدون إسناد، 3 أُنشئت اليوم، رضا 4/5).
- **العربي/RTL:** `dir=rtl` واتجاه الواجهة صحيحان، ولا توجد مفاتيح ترجمة ناقصة ظاهرة.
- **الموبايل (375px):** لا يوجد overflow أفقي في 12 شاشة.
- **الـ API:** 404 بشكل صحيح للـ GUID غير الصالح أو غير الموجود، envelope موحّد مع `correlationId`، و`/health/live` و`/health/ready` = 200.

---

## بيانات اختبار أُنشئت في قاعدة dev

- عميل `C-000005 QA Test Customer` (`qa.customer@example.test`) مع وصول للبوابة وملاحظة.
- تذاكر `T-000004` (مفتوحة، تقييم 4)، `T-000005` (محادثة مباشرة نشطة)، `T-000006` (web form).
- تصنيفات `QA Cycle A` و `QA Cycle B` (معطّلة).
- 3 رسائل Failed في الـ outbox.
