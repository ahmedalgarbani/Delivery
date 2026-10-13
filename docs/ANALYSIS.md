# وثيقة التحليل: منصة التوصيل متعددة المتاجر (صنعاء، اليمن)

> **هذه الوثيقة مرجع وحيد للوكيل (AI Agent) الذي ينفّذ المشروع.**
> كل القرارات الواردة هنا **مُتّخذة ومعتمدة من صاحب المشروع** ولا تُعاد مناقشتها.
> ما لم يُذكر صراحةً يُسجَّل تحت "افتراضات" في القسم 17 ويُنفَّذ بالخيار الأبسط، ثم يُبلَّغ عنه.

---

## 0. تعليمات للوكيل المنفّذ

1. اقرأ الوثيقة كاملة قبل كتابة أي كود. لا تبدأ بالتطبيقات قبل أن يكتمل الـ Backend لتلك الميزة.
2. **الترتيب:** `delivery-infra` ← `delivery-backend` ← اللوحات ← تطبيقات Flutter.
3. **المال مقدّس:** أي كود يمس الأرصدة أو الـ Ledger أو العمولات يُكتب مع اختبارات وحدة أولًا (TDD).
4. **المبالغ أعداد صحيحة** (`bigint`) بالريال اليمني (YER) بلا كسور. ممنوع استخدام `float`/`numeric` للمال.
5. **لا تعدّل جدولًا ماليًا بـ UPDATE.** قيود الـ Ledger غير قابلة للتعديل أو الحذف (Append-only).
6. **كل طلب يحفظ Snapshot** من الأسماء والأسعار والعنوان والعمولة وقت الإنشاء. ممنوع أن يتغير طلب قديم بتغيّر القائمة.
7. **الـ Backend هو مصدر الحقيقة الوحيد**: المنطقة، والسعر، والعمولة، والمهلات تُحسب في الخادم ولا تُصدَّق من العميل.
8. نصوص الواجهات كلها عبر ملفات ترجمة (عربي أولًا RTL، إنجليزي ثانيًا). ممنوع نص ثابت داخل الكود.
9. كل Pull Request: اختبارات خضراء + Lint + تحديث OpenAPI. لا تدمج بدونها.
10. عند أي غموض: اتخذ القرار الأبسط، دوّنه في `docs/DECISIONS.md` في مستودع الوثائق، وتابع.

---

## 1. نظرة عامة

منصة توصيل **عامة (Multi-vendor)** تخدم أنواع متاجر مختلفة (مطاعم، سوبرماركت، صيدليات، …) بنموذج واحد: "المتجر" كيان عام، والميزات الخاصة (وقت التحضير، الإضافات للمطاعم، المخزون بالكمية للسوبرماركت) تُفعَّل حسب نوع المتجر.

- **السوق:** صنعاء، اليمن. العملة: ريال يمني (YER). اللغة: عربي (RTL) ثم إنجليزي.
- **الطبيعة:** منتج حقيقي للإطلاق، ولدى صاحب المشروع اتفاقات فعلية مع متاجر وسائقين.
- **النمط التشغيلي** (مثل توصيل ون/طلقة/ناس): العميل يطلب ← يُبثّ الطلب للسائقين القريبين ← أول من يقبل يأخذه ← السائق يحرّك حالة الطلب. **التاجر مستقبِل فقط ولا يرفض ولا يغيّر الحالة.**
- **الدفع:** كاش عند الاستلام + محفظة داخلية (الشحن يدويًا من الأدمن في البداية).

---

## 2. المستودعات (Repos)

كل مشروع في مستودع مستقل. المستودع الحالي `Delivery` يُستخدم مستودعًا للوثائق.

| # | اسم المستودع | المحتوى | التقنية | المرحلة |
|---|---|---|---|---|
| 1 | `delivery-docs` (أو هذا المستودع `Delivery`) | الوثائق، القرارات، ERD، OpenAPI المنشور | Markdown | 1 |
| 2 | `delivery-infra` | Docker Compose، Nginx، Postgres+PostGIS، Redis، خادم خرائط البلاط، النسخ الاحتياطي، سكربتات النشر | Docker, Nginx, Bash | 1 |
| 3 | `delivery-backend` | الـ API وWebSocket والطوابير | NestJS, Drizzle, PostgreSQL+PostGIS, Redis, BullMQ, Socket.IO | 1 |
| 4 | `delivery-admin-web` | لوحة الأدمن | Next.js | 1 |
| 5 | `delivery-merchant-web` | لوحة التاجر | Next.js | 1 |
| 6 | `delivery-customer-app` | تطبيق العميل | Flutter (Android أولًا) | 1 |
| 7 | `delivery-driver-app` | تطبيق السائق | Flutter (Android أولًا) | 1 |
| 8 | `delivery-merchant-app` | تطبيق التاجر | Flutter (Android) | 2 *(افتراض، راجع القسم 17)* |

> يوصى بإنشاء المستودعات تحت Organization واحدة على GitHub بنفس البادئة `delivery-`.
> بما أن المستودعات منفصلة، **العقد الوحيد المشترك هو OpenAPI** المُولَّد من الـ Backend (راجع القسم 12).

---

## 3. التقنيات والمكتبات (معتمدة)

### Backend (`delivery-backend`)
- NestJS + TypeScript (strict)، **Modular Monolith** (تطبيق واحد بوحدات منفصلة، ليس Microservices).
- **Drizzle ORM** + `drizzle-kit` للهجرات. PostgreSQL 16 + **PostGIS**.
  - أنواع PostGIS (`geometry`/`geography` للنقاط والمضلعات) عبر Custom Types في Drizzle مع `sql` template للاستعلامات الجغرافية.
- Redis 7: مواقع السائقين الحية (GEO)، الـ Heartbeat، مقابس Socket.IO، طوابير BullMQ.
- BullMQ: مهلات توسيع البث، الإشعارات، المهام المجدولة.
- Socket.IO (مع Redis Adapter).
- `@nestjs/swagger` لتوليد OpenAPI، `zod` أو `class-validator` للتحقق.
- Firebase Admin (FCM)، مزوّد واتساب (WhatsApp Business API) للـ OTP والتنبيهات.
- Pino للسجلات، Sentry للأخطاء.
- الاختبارات: Jest + Testcontainers (Postgres+PostGIS وRedis حقيقيان).

### لوحات الويب (`delivery-admin-web`, `delivery-merchant-web`)
- Next.js (App Router) + TypeScript + Tailwind + shadcn/ui + TanStack Query + next-intl (عربي/إنجليزي RTL).
- عميل API مُولَّد من OpenAPI (`orval` أو `openapi-typescript`).
- خرائط: MapLibre GL أو Leaflet + أداة رسم مضلعات (للمناطق في لوحة الأدمن).
- اختبارات E2E: Playwright.

### تطبيقات Flutter
- Riverpod (إدارة الحالة)، `go_router`، `dio` مع عميل مُولَّد من OpenAPI.
- `flutter_map` للخرائط (بلاط من خادمنا الخاص، **ممنوع** استخدام `tile.openstreetmap.org` في الإنتاج).
- `firebase_messaging`، `socket_io_client`، `flutter_localizations` (ملفات `.arb` عربي/إنجليزي).
- تطبيق السائق: `geolocator` + Foreground Service (أو `flutter_background_geolocation`) لإرسال الموقع في الخلفية.
- الاختبارات: `flutter_test` + `integration_test`.

### البنية التحتية
- **VPS واحد** (Docker Compose). نشر آلي عبر GitHub Actions (SSH).
- خادم بلاط خرائط خاص بصنعاء/اليمن: `tileserver-gl` أو Protomaps (PMTiles) من بيانات OSM.
- نسخ احتياطي يومي لـ PostgreSQL إلى تخزين خارج الـ VPS (مثل restic إلى S3-compatible) مع اختبار استعادة دوري.

---

## 4. الأدوار والصلاحيات

| الدور | الواجهة | الدخول | الإنشاء |
|---|---|---|---|
| العميل (Customer) | تطبيق Flutter | رقم هاتف + OTP واتساب | تسجيل ذاتي |
| السائق (Driver) | تطبيق Flutter | رقم هاتف + OTP واتساب | تسجيل ذاتي ← **موافقة الأدمن** |
| التاجر (Merchant) | لوحة ويب (+ تطبيق لاحقًا) | رقم هاتف + OTP واتساب | تسجيل ذاتي ← **موافقة الأدمن** |
| الأدمن (Admin) | لوحة ويب | بريد/هاتف + كلمة مرور (+2FA اختياري) | يُنشأ من أدمن رئيسي |

### صلاحيات الأدمن (RBAC مخصص)
- جدول `roles` و`permissions`؛ الأدمن الرئيسي ينشئ أدوارًا ويحدد صلاحية كل **صفحة وإجراء** (مثل `wallet.topup`, `zones.edit`, `orders.assign`, `settlement.confirm`).
- الحماية في **الـ Backend** (Guard على كل نقطة) وفي الواجهة (إخفاء).
- **Audit Log لكل إجراء** يقوم به أي أدمن (من، ماذا، قبل/بعد، متى، IP). غير قابل للتعديل.

### حالات الحساب
`PENDING_APPROVAL` → `ACTIVE` ↔ `SUSPENDED` ، و`REJECTED`. السائق والتاجر لا يعملان قبل `ACTIVE`.

---

## 5. نموذج البيانات (ERD مفصّل)

> أسماء الجداول `snake_case` بصيغة الجمع. كل جدول فيه `id` (uuid v7 أو bigint) و`created_at` و`updated_at`.
> **المال = `bigint` (ريال).** الإحداثيات: SRID 4326.

### 5.1 الهوية
- `users`: `phone` (unique)، `name`، `status`، `locale`، `fcm_token(s)`.
- `user_roles`: `user_id`، `role` (`CUSTOMER|DRIVER|MERCHANT_OWNER|MERCHANT_STAFF|ADMIN`).
- `admin_accounts`: `user_id`، `email`، `password_hash`، `two_factor_secret?`، `admin_role_id`.
- `admin_roles` + `admin_role_permissions(role_id, permission)`.
- `otp_codes`: `phone`، `code_hash`، `expires_at`، `attempts`، `channel`.
- `refresh_tokens`: `user_id`، `token_hash`، `device_id`، `revoked_at`.
- `audit_logs`: `actor_id`، `action`، `entity`، `entity_id`، `before` (jsonb)، `after` (jsonb)، `ip`، `created_at`.
- `app_versions`: `app` (`customer|driver|merchant`)، `platform`، `min_supported_version`، `latest_version`.

### 5.2 التجار والمتاجر (مصمَّم جاهزًا للفروع)
- `merchants`: `owner_user_id`، `legal_name`، `commission_percent` (**نسبة مخصصة لكل تاجر**، 0–100، تحتمل خانتين عشريتين → خزّنها `basis points` كعدد صحيح)، `status`، وثائق (`document_urls`).
- `stores` (= فرع): `merchant_id`، `store_type_id`، `name_ar`، `name_en`، `location` (point)، `zone_id` (يُحسب من الموقع)، `address_text`، `logo_url`، `cover_url`، `prep_time_minutes?` (للمطاعم)، `is_open_override` (nullable)، `override_until` (nullable)، `status`.
  > **المرحلة 1:** كل تاجر له متجر واحد، لكن النموذج يدعم فروعًا بلا تغيير مستقبلًا.
- `merchant_staff` (المرحلة 2): `store_id`، `user_id`، `role` (`MANAGER|CASHIER`).
- `store_hours`: `store_id`، `weekday` (0–6)، `open_time`، `close_time` (يدعم أكثر من فترة باليوم).
- `store_types` (على مستوى المنصة، يديرها الأدمن): `name_ar`، `name_en`، `icon`، `features` (jsonb: `{prep_time:true, addons:true, stock:true}`)، `sort_order`.
- `store_categories` (خاصة بكل متجر): `store_id`، `name`، `sort_order`.

**منطق "المفتوح/المغلق":** المتجر مفتوح إذا (`is_open_override` غير فارغ وصالح لم ينتهِ `override_until`) فالقيمة اليدوية، وإلا حسب `store_hours` بتوقيت اليمن (UTC+3). تُحسب في الـ Backend.

### 5.3 المنتجات
- `products`: `store_id`، `category_id`، `name_ar`، `name_en`، `description`، `image_url`، `base_price`، `is_available` (توفر يدوي)، `track_stock` (bool)، `stock_quantity` (nullable)، `sort_order`.
- `product_variants`: `product_id`، `name` (مثل صغير/وسط/كبير)، `price`، `is_default`. (إن لم توجد variants يُستخدم `base_price`.)
- `addon_groups`: `product_id` (أو `store_id` لمجموعات قابلة لإعادة الاستخدام)، `name`، `min_select`، `max_select`، `is_required`.
- `addons`: `group_id`، `name`، `price_delta`، `is_available`.

**المخزون (الاثنان معًا):**
- التوفر اليدوي `is_available` للمطاعم.
- `track_stock=true` للسوبرماركت: خصم ذري عند إنشاء الطلب `UPDATE products SET stock_quantity = stock_quantity - :q WHERE id=:id AND stock_quantity >= :q` (فشل ⇒ رفض الطلب)، وإعادة الكمية عند الإلغاء.

### 5.4 الجغرافيا والأسعار
- `zones`: `name_ar`، `name_en`، `polygon` (geometry Polygon)، `is_active`.
- `delivery_fees`: `from_zone_id` (منطقة المتجر)، `to_zone_id` (منطقة العميل)، `fee` — **فريد على (from,to)**، ويُحرَّر كمصفوفة في لوحة الأدمن.
- `driver_zones`: `driver_id`، `zone_id` (المناطق المسموح للسائق العمل فيها).

### 5.5 العملاء والعناوين
- `customer_addresses`: `user_id`، `label`، `location` (point)، `zone_id` (يُحسب بالخادم)، `details_text` (وصف نصي)، `is_default`. عدة عناوين، ويمكن تعديل أي عنوان محفوظ.

### 5.6 السائقون
- `drivers`: `user_id`، `vehicle_type`، `plate`، `documents`، `status`، `is_online` (حالة منطقية)، `payout_type` (`PERCENT|FIXED|SALARY`)، `payout_value`، `cash_limit` (**حد مخصص لكل سائق**)، `withdrawals_today`، `suspended_until`.
- `driver_location_history`: `driver_id`، `order_id?`، `location`، `recorded_at` (للطلبات النشطة فقط، كل ~15 ثانية، وتُقسَّم بالتاريخ).
- `driver_withdrawals`: `driver_id`، `order_id`، `reason`، `created_at`.
- `driver_penalties`: `driver_id`، `type` (`AUTO_SUSPEND|MANUAL_DEDUCTION`)، `amount?`، `reason`، `created_by`.

> **المواقع الحية في Redis وليست في PostgreSQL:** مفتاح `drivers:geo` (GEOADD) + مفتاح `driver:{id}:hb` بـ TTL = 60 ثانية. التاريخ فقط يُحفظ في PG.

### 5.7 الطلبات
- `orders`:
  - `id`، `code` (رقم قصير مقروء)، `customer_id`، `store_id`، `driver_id?`
  - `status` (راجع القسم 6)، `payment_method` (`CASH|WALLET`)، `payment_status`
  - `subtotal`، `delivery_fee`، `discount_total` (0 في المرحلة 1)، `total`
  - `commission_percent_snapshot`، `commission_amount`
  - `driver_payout_type_snapshot`، `driver_earning`
  - `customer_location` (point)، `customer_zone_id` (محسوب)، `store_zone_id`، `address_snapshot` (jsonb: نص + وصف)
  - `notes`، `cancel_reason`، `cancelled_by`
  - `broadcast_radius_km`، `broadcast_step`
  - `delivered_lat/lng` (موقع السائق لحظة "تم التسليم")
  - طوابع زمنية: `placed_at`، `accepted_at`، `preparing_at`، `picked_up_at`، `delivered_at`، `cancelled_at`
  - `version` (للقفل المتفائل)
- `order_items`: `order_id`، `product_id`، `name_snapshot`، `variant_snapshot`، `unit_price_snapshot`، `quantity`، `line_total`، `addons_snapshot` (jsonb)، `notes`.
- `order_status_history`: `order_id`، `from`، `to`، `actor_type/id`، `at`، `meta`.
- `order_dispatch_attempts`: `order_id`، `step`، `radius_km`، `driver_ids` (من أُرسل لهم)، `at`.

**فهرس حاسم:** ضمان سائق واحد لطلب نشط واحد على الأكثر:
```sql
CREATE UNIQUE INDEX one_active_order_per_driver
  ON orders (driver_id)
  WHERE status IN ('ACCEPTED','PREPARING','ON_THE_WAY');
```

### 5.8 المال (Ledger مزدوج القيد)
- `ledger_accounts`: `id`، `type`، `owner_type`، `owner_id`، `currency='YER'`.
  - الأنواع: `CUSTOMER_WALLET`، `ESCROW` (حجز الدفع من المحفظة)، `MERCHANT_PAYABLE` (مستحقات التاجر)، `DRIVER_EARNINGS` (أرباح السائق)، `DRIVER_CASH_HELD` (كاش بحوزة السائق = التزام للمنصة)، `PLATFORM_REVENUE`، `PLATFORM_CASH`، `PROMO_EXPENSE` (المرحلة 2).
- `ledger_transactions`: `id`، `type`، `order_id?`، `idempotency_key` (unique)، `created_by`، `created_at`، `description`.
- `ledger_entries`: `transaction_id`، `account_id`، `direction` (`DEBIT|CREDIT`)، `amount` (>0). **مجموع المدين = مجموع الدائن لكل معاملة** (قيد CHECK/Trigger + اختبار).
- `account_balances` (مادّي، يُحدَّث داخل نفس المعاملة): `account_id`، `balance`، `updated_at`. ويجب ألا يتعارض مع مجموع القيود (مهمة تدقيق ليلية).
- `wallet_topups`: `customer_id`، `amount`، `admin_id`، `note`، `ledger_transaction_id`. (الشحن اليدوي من الأدمن.)
- `cash_settlements`: `driver_id`، `amount`، `received_by_admin_id`، `at`، `note`، `ledger_transaction_id`. (تسليم الكاش اليومي.)
- `merchant_payouts`: `merchant_id`، `amount`، `paid_by_admin_id`، `at`، `ledger_transaction_id`. (تسوية مستحقات التاجر.)

### 5.9 الإشعارات
- `notifications`: `user_id`، `type`، `payload`، `channel`، `status`، `sent_at`.

### 5.10 جداول المرحلة الثانية (تُحجز ولا تُنفَّذ الآن)
`ratings` (متجر وسائق منفصلان)، `tickets` + `ticket_messages` (العميل والسائق والتاجر)، `coupons`، `promotions`، `subscriptions` (منح يدوي من الأدمن)، `merchant_staff`.

---

## 6. آلة حالات الطلب (State Machine)

**حالات داخلية ← ما يظهر للعميل:**

| داخلي | يظهر للعميل | المعنى | من يحرّكها |
|---|---|---|---|
| `PENDING` | قيد المراجعة | أُنشئ وفي مرحلة البث | النظام عند إنشاء الطلب |
| `ACCEPTED` | قيد المراجعة | سائق قبل الطلب (داخلية) | السائق (قبول ذري) |
| `PREPARING` | قيد التحضير | السائق وصل للمتجر | **السائق** (زر "وصلت للمتجر") |
| `ON_THE_WAY` | في الطريق إليك | السائق استلم الطلب | السائق |
| `DELIVERED` | تم التوصيل | سُلِّم | السائق (ضغطة "تم التسليم") |
| `CANCELLED` | ملغي | | العميل / الأدمن |

**الانتقالات المسموحة فقط:**
```
PENDING   → ACCEPTED      (سائق، قبول ذري)
PENDING   → CANCELLED     (العميل / الأدمن)
ACCEPTED  → PREPARING     (السائق عند وصوله للمتجر)
ACCEPTED  → PENDING       (انسحاب السائق، يُعاد للبث)
ACCEPTED  → CANCELLED     (العميل / الأدمن)
PREPARING → ON_THE_WAY    (السائق)
PREPARING → CANCELLED     (الأدمن فقط)
ON_THE_WAY→ DELIVERED     (السائق)
ON_THE_WAY→ CANCELLED     (الأدمن فقط)
```
- **إلغاء العميل مسموح فقط في `PENDING` و`ACCEPTED`**، أي حتى ضغط السائق "وصلت للمتجر".
- **الانسحاب:** للسائق فقط في `ACCEPTED` (قبل وصوله للمتجر)، ويُسجَّل ويُعالَج بعقوبات (القسم 8.4).
- **التاجر لا يرفض ولا يحرّك الحالة.**
- كل انتقال: داخل معاملة DB، مع فحص `from` الحالي (WHERE status = :expected)، وتسجيل في `order_status_history`، ثم بثّ حدث.
- ممنوع أي انتقال خارج الجدول (يُرجع 409).
- العميل لا يستطيع تعديل العنوان بعد إرسال الطلب (يلغي ويعيد).
- طلب العميل من **متجر واحد فقط**؛ تبديل المتجر يُفرِّغ السلة.

---

## 7. إنشاء الطلب والتسعير

### 7.1 خطوات `POST /orders` (معاملة واحدة، مع Idempotency-Key)
1. تحقق أن المتجر **مفتوح** (القسم 5.2) وأن العميل `ACTIVE`.
2. أعد حساب سعر كل بند من قاعدة البيانات (تجاهل أي سعر من العميل)، وتحقق من توفر المنتج/الإضافات/المخزون وخصم المخزون ذريًا.
3. **حدد منطقة العميل من الدبوس:** `ST_Contains(zones.polygon, point)`. **المنطقة التي يختارها العميل في القائمة للعرض فقط ويتجاهلها الخادم.** إن وقع الدبوس خارج كل المناطق الفعّالة ← رفض (خارج نطاق الخدمة).
4. رسوم التوصيل = `delivery_fees[store.zone_id → customer_zone_id]`. إن لم يوجد سطر ← رفض.
5. العمولة = `subtotal × merchant.commission_percent`. (المرحلة 2: الخصم يؤثر حسب من يتحمله.)
6. أجر السائق وفق `payout_type` (يُحسب وقت التسليم لكن يُخزَّن snapshot لنوعه عند القبول).
7. إن كان الدفع بالمحفظة: تحقق من الرصيد وانقل `total` إلى `ESCROW` (حجز).
8. أنشئ الطلب بحالة `PENDING`، بلّغ التاجر فورًا، وابدأ البث للسائقين.

**البث يبدأ فور الإنشاء** (لا ينتظر التاجر).

---

## 8. خوارزمية البث والإسناد (Dispatch)

### 8.1 المرشّح المؤهل
سائق يحقق **كل** الآتي:
- `status=ACTIVE` وغير موقوف (`suspended_until` فارغ/منتهٍ).
- **متصل**: سائق ضغط "متصل" **و**`driver:{id}:hb` موجود (آخر موقع خلال 60 ثانية). وإن انقطع الإرسال يُعتبر غير متصل تلقائيًا.
- **ليس لديه طلب نشط** (طلب واحد فقط في الوقت).
- ضمن نصف القطر: `GEOSEARCH drivers:geo FROMLONLAT <store> BYRADIUS <r> km`.
- منطقة المتجر ضمن `driver_zones` الخاصة به.
- `DRIVER_CASH_HELD` أقل من `cash_limit` المخصص له.

### 8.2 التوسيع التدريجي
- خطوات افتراضية قابلة للضبط من الأدمن: `[2km, 4km, 6km]` وكل خطوة مهلتها `60s`.
- الخطوة الأولى فورًا عند الإنشاء. كل مهلة تُجدول بـ **BullMQ delayed job**؛ عند انتهائها وبقاء الطلب `PENDING` ننتقل للخطوة التالية.
- في كل خطوة نُرسل لمن لم يستلم الطلب سابقًا: Socket + FCM (عالي الأولوية) ونسجّلهم في `order_dispatch_attempts`.
- بعد آخر خطوة بلا قبول: يوسَم الطلب `UNASSIGNED_ALERT` ويظهر في لوحة الأدمن بتنبيه بارز للإسناد اليدوي، **ويستمر البث بأكبر نصف قطر** إلى أن يُقبل أو يُلغى.

### 8.3 القبول الذري (Race-free)
```sql
UPDATE orders
SET status='ACCEPTED', driver_id=:driver, accepted_at=now(), version=version+1
WHERE id=:order AND status='PENDING' AND driver_id IS NULL
RETURNING *;
```
- صفر صفوف ← `409 ORDER_ALREADY_TAKEN`.
- فهرس `one_active_order_per_driver` يمنع تعدد الطلبات لسائق واحد حتى لو تزامنت طلبات القبول.
- عند النجاح: إلغاء بث الطلب للآخرين (حدث `order.taken`)، إشعار العميل والتاجر.
- **اختبار إلزامي:** 50 طلب قبول متزامن لطلب واحد ⇒ فائز واحد بالضبط.

### 8.4 انسحاب السائق والعقوبات
- مسموح فقط في `ACCEPTED`. يعود الطلب `PENDING` ويُعاد البث (يُستثنى السائق المنسحب).
- كل انسحاب يُسجَّل في `driver_withdrawals`.
- **عقوبات مزيج:**
  1. **تسجيل** عدّاد يظهر للأدمن.
  2. **إيقاف مؤقت تلقائي** عند تجاوز عتبة (افتراضي: 3 انسحابات/يوم ← إيقاف 2 ساعة)، قابل للضبط.
  3. **خصم مالي يدوي** يقرره الأدمن (يُنفَّذ كقيد Ledger مع سبب).

### 8.5 إسناد يدوي
الأدمن يستطيع إسناد طلب `PENDING` لسائق معيّن مباشرة (تتجاوز شروط الأهلية ما عدا "طلب نشط واحد")، ويُسجَّل في الـ Audit Log.

---

## 9. المال: المحفظة والكاش والعمولات

### 9.1 المبادئ
- Ledger مزدوج القيد غير قابل للتعديل. الأرصدة تُشتق منه.
- كل معاملة ذات `idempotency_key`.
- لا رصيد سالب على `CUSTOMER_WALLET` (قيد DB + فحص).

### 9.2 تسجيل الطلب عند التسليم (`DELIVERED`) — معاملة واحدة
**طلب كاش** (العميل يدفع `total` نقدًا للسائق):
- مدين `DRIVER_CASH_HELD` بـ `total` (السائق يحمل نقدًا يخص المنصة).
- دائن `MERCHANT_PAYABLE` بـ `subtotal − commission`.
- دائن `PLATFORM_REVENUE` بـ `commission`.
- دائن `DRIVER_EARNINGS` بـ `driver_earning`.
- دائن `PLATFORM_REVENUE` بـ `delivery_fee − driver_earning` (حصة المنصة من التوصيل).
  > المجموع الدائن = `subtotal + delivery_fee` = `total`. تأكد أن `driver_earning ≤ delivery_fee` وإلا ينعكس الفارق على المنصة (يُسجَّل كخسارة صريحة لا يُخفى).

**طلب محفظة:** عند الإنشاء يُنقل `total` من `CUSTOMER_WALLET` إلى `ESCROW`. عند التسليم يُفرَّغ `ESCROW` بالتوزيع نفسه (بدون `DRIVER_CASH_HELD`). عند الإلغاء يعود `ESCROW` إلى `CUSTOMER_WALLET`.

### 9.3 أنواع أجر السائق (قابلة للضبط لكل سائق)
- `PERCENT`: نسبة من `delivery_fee` (مثل 80%).
- `FIXED`: مبلغ ثابت لكل طلب (اختياريًا يُستبدل حسب زوج المناطق).
- `SALARY`: راتب شهري؛ لا يُقيَّد أجر لكل طلب، ويُسجَّل الراتب يدويًا بدفعة من الأدمن، لكن تُسجَّل الطلبات للإحصاء.
نوع الأجر يُحفَظ snapshot على الطلب عند القبول فلا يتغير بتغيّر الإعدادات لاحقًا.

### 9.4 تسليم الكاش اليومي
- السائق يسلّم الكاش للأدمن نهاية اليوم. الأدمن يسجّل **`cash_settlement`** فيُدين `DRIVER_CASH_HELD` ويدين `PLATFORM_CASH` (أو ما يوافقه محاسبيًا).
- **حد الكاش المخصص:** إن بلغ `DRIVER_CASH_HELD ≥ cash_limit` يُمنع السائق من استقبال طلبات جديدة حتى التسليم.
- تقرير يومي: كل سائق وما عليه من كاش وما سلّمه.

### 9.5 شحن المحفظة (يدوي)
الأدمن يختار العميل ويدخل المبلغ والملاحظة (إثبات الاستلام) ← `wallet_topups` + قيد (مدين `PLATFORM_CASH`، دائن `CUSTOMER_WALLET`) + Audit Log. تصميم الـ Ledger يسمح لاحقًا بربط بوابات المحافظ المحلية دون إعادة بناء (إضافة `provider` و`external_ref` فقط).

### 9.6 تسوية التاجر
الأدمن يدفع مستحقات التاجر (نقدًا/تحويل) ويسجّل `merchant_payout` ← مدين `MERCHANT_PAYABLE`، دائن `PLATFORM_CASH`.

### 9.7 تدقيق دوري
مهمة ليلية تتحقق: (1) كل معاملة متوازنة. (2) `account_balances` = مجموع القيود. أي اختلاف ← تنبيه فوري.

---

## 10. الوقت الحقيقي والإشعارات

### 10.1 Socket.IO (مصادقة JWT، غرف Rooms)
غرف: `customer:{id}`، `driver:{id}`، `store:{id}`، `admin`.

| الحدث | الاتجاه | الغرض |
|---|---|---|
| `order.new` | → store, admin | طلب جديد للتاجر |
| `order.offer` | → drivers المؤهلين | عرض طلب على السائقين (مع المسافة والرسوم) |
| `order.taken` | → drivers | الطلب أُخذ، أزِل العرض |
| `order.status_changed` | → customer, store, admin | تغير الحالة |
| `driver.location` | driver → server | موقع السائق |
| `driver.location_update` | → customer (طلبه النشط فقط) | تتبع حي |
| `order.unassigned_alert` | → admin | لم يقبل أي سائق |

### 10.2 مواقع السائق (تكيفية)
- **5 ثوانٍ** أثناء طلب نشط. **30 ثانية** وهو متاح بلا طلب.
- الخادم يحدّث `GEOADD` ومفتاح الـ Heartbeat. التاريخ يُحفظ في PG أثناء الطلب فقط (~كل 15 ثانية).
- العميل لا يرى إلا السائق المُسند لطلبه.
- **الخلفية:** تطبيق السائق يستخدم Foreground Service مع إشعار دائم "أنت متصل". يجب اختبار الأجهزة التي تقتل الخلفية (Xiaomi/Huawei) وإرشاد المستخدم لإعدادات توفير الطاقة.

### 10.3 الإشعارات
- **FCM + واتساب** (واتساب لـ OTP وللأحداث المهمة مثل "طلب جديد للتاجر").
- كل إشعار يُسجَّل في `notifications` مع الحالة.
- (اختياري، انظر القسم 17) تصعيد: طلب جديد لم يفتحه التاجر خلال X دقيقة ← واتساب ← تنبيه للأدمن.

---

## 11. وحدات الـ Backend (NestJS Modules)

`auth`، `users`، `admin` (RBAC + audit)، `merchants`، `stores`، `catalog` (منتجات/خيارات/إضافات/مخزون)، `zones` (+ `delivery-fees`)، `addresses`، `orders`، `dispatch`، `drivers`، `wallet`/`ledger`، `settlements`، `notifications`، `realtime` (gateway)، `reports`، `app-version`، `health`.

### نقاط API الرئيسية (REST تحت `/api/v1`، موثَّقة بـ OpenAPI)

**Auth:** `POST /auth/otp/request`، `POST /auth/otp/verify`، `POST /auth/refresh`، `POST /auth/logout`، `POST /admin/auth/login`.

**العميل:** `GET /stores?type=&zone=` (قائمة المتاجر مع حالة مفتوح/مغلق)، `GET /stores/:id`، `GET /stores/:id/menu`، `POST /orders`، `GET /orders` (سجل)، `GET /orders/:id`، `POST /orders/:id/cancel`، `POST /orders/:id/reorder` (تكرار الطلب)، `GET/POST/PATCH/DELETE /me/addresses`، `GET /me/wallet`، `GET /me/wallet/transactions`.

**السائق:** `POST /driver/online`، `POST /driver/offline`، `GET /driver/offers`، `POST /driver/orders/:id/accept`، `POST /driver/orders/:id/withdraw`، `POST /driver/orders/:id/arrived` (← PREPARING)، `POST /driver/orders/:id/picked-up` (← ON_THE_WAY)، `POST /driver/orders/:id/delivered` (مع الإحداثيات)، `GET /driver/earnings`، `GET /driver/cash-balance`.

**التاجر:** `GET/PATCH /merchant/store`، `PUT /merchant/store/hours`، `POST /merchant/store/open|close` (إغلاق يدوي مؤقت)، `CRUD /merchant/categories|products|variants|addon-groups|addons`، `PATCH /merchant/products/:id/availability`، `PATCH /merchant/products/:id/stock`، `GET /merchant/orders` (+ Socket)، `GET /merchant/orders/:id/print`، `GET /merchant/reports/sales`، `GET /merchant/balance`.

**الأدمن:** إدارة المستخدمين والتجار والسائقين (`approve|reject|suspend`)، `CRUD /admin/store-types|zones|delivery-fees|admin-roles`، `PATCH /admin/merchants/:id/commission`، `PATCH /admin/drivers/:id/payout|cash-limit|zones`، `POST /admin/orders/:id/assign`، `POST /admin/orders/:id/cancel`، `POST /admin/wallet/topup`، `POST /admin/settlements/cash`، `POST /admin/payouts/merchant`، `GET /admin/audit-logs`، `GET /admin/reports/*` (أساسية ومالية مع تصدير Excel/PDF)، `GET/PUT /admin/settings` (خطوات البث ومهلاتها وعتبات العقوبات).

**عامة:** `GET /app/version-check`، `GET /health`.

### قواعد عامة للـ API
- ترويسات: `X-App`, `X-App-Version`, `X-Platform`. إن كانت النسخة أقل من `min_supported_version` ← `426 Upgrade Required` مع رابط التحديث.
- إصدار API: `/v1` ثابت، ويُضاف `/v2` عند التغييرات الكاسرة مع إبقاء v1 فترة انتقالية.
- ترقيم صفحات موحّد (cursor)، وصيغة أخطاء موحّدة `{code, message, details}`.
- `Idempotency-Key` إلزامي على `POST /orders` وكل العمليات المالية.
- Rate limiting على OTP وتسجيل الدخول.

---

## 12. مشاركة العقد بين المستودعات (OpenAPI)

- الـ Backend ينشر `openapi.json` في كل إصدار (artifact في CI و/أو يُدفع إلى `delivery-docs`).
- اللوحات: توليد عميل TypeScript بـ `orval`/`openapi-typescript` ضمن سكربت `npm run gen:api`.
- Flutter: توليد عميل Dart بـ `openapi_generator`/`swagger_parser` ضمن سكربت.
- أي تغيير كاسر في API يرفع الإصدار ويُحدّث `min_supported_version`.

---

## 13. شاشات كل تطبيق

### 13.1 تطبيق العميل (`delivery-customer-app`)
- دخول بالهاتف + OTP.
- الرئيسية: أنواع المتاجر، قائمة المتاجر (مفتوح/مغلق)، بحث.
- صفحة المتجر: فئات المتجر، المنتجات، خيارات وإضافات، السلة (متجر واحد).
- إتمام الطلب: اختيار عنوان محفوظ أو جديد (**دبوس على الخريطة + وصف نصي + اختيار منطقة**؛ الخادم يحسب المنطقة من الدبوس)، طريقة الدفع (كاش/محفظة)، الملخص مع رسوم التوصيل.
- تتبع الطلب: الحالات الأربع (قيد المراجعة، قيد التحضير، في الطريق إليك، تم التوصيل)، وخريطة حية للسائق أثناء التوصيل، زر إلغاء (متاح حتى "قيد التحضير").
- سجل الطلبات + **تكرار الطلب**.
- المحفظة (الرصيد وسجل المعاملات)، إدارة العناوين (إضافة/تعديل/حذف).
- (المرحلة 2: تقييم المتجر والسائق، تذاكر الدعم، أكواد الخصم.)

### 13.2 تطبيق السائق (`delivery-driver-app`)
- تسجيل بالهاتف + رفع الوثائق ← انتظار الموافقة.
- زر **متصل/غير متصل**، خريطة، عروض الطلبات الواردة (بطاقة منبثقة بمؤقت، المتجر والمسافة والرسوم) مع قبول.
- شاشة الطلب النشط: انسحاب (قبل الوصول للمتجر)، "وصلت للمتجر"، "استلمت الطلب"، "تم التسليم"، تفاصيل العميل والعنوان والدبوس وبياناته.
- الأرباح، رصيد الكاش بحوزتي وحدّه، سجل الطلبات.
- Foreground Service للموقع، وتعامل مع الصلاحيات.

### 13.3 لوحة التاجر (`delivery-merchant-web`)
- دخول OTP، لوحة الطلبات الحية مع **تنبيه صوتي** للطلب الجديد، **طباعة الطلب**.
- إدارة المتجر: بيانات، شعار/غلاف، ساعات العمل، فتح/إغلاق يدوي.
- إدارة الفئات والمنتجات والخيارات والإضافات والتوفر والمخزون.
- سجل الطلبات والتقارير والرصيد المستحق.
- (المرحلة 2: الفروع، الموظفون وصلاحياتهم.)

### 13.4 لوحة الأدمن (`delivery-admin-web`)
- لوحة رئيسية: طلبات اليوم، إيرادات، عمولات، سائقون متصلون، **طلبات بلا سائق**.
- المستخدمون: عملاء، سائقون (موافقة/رفض/إيقاف، المناطق، الأجر، حد الكاش، العقوبات)، تجار (موافقة، عمولة مخصصة).
- أنواع المتاجر، **المناطق** (رسم مضلعات على الخريطة)، **جدول أسعار التوصيل** (مصفوفة منطقة×منطقة قابلة للتحرير).
- الطلبات: قائمة وتفاصيل، إسناد يدوي، إلغاء.
- المال: شحن المحافظ، تسليم الكاش اليومي للسائقين، تسوية التجار، سجل الـ Ledger.
- الأدمنز والأدوار (RBAC مخصص)، سجل التدقيق.
- التقارير: أساسية + مالية (مستحقات، كاش السائقين) مع تصدير Excel/PDF.
- الإعدادات: خطوات البث، العتبات، الحد الأدنى لنسخ التطبيقات.

### 13.5 تطبيق التاجر (`delivery-merchant-app`) — المرحلة 2
استقبال الطلبات والتنبيهات، وفتح/إغلاق المتجر، وتوفر الأصناف. (يعتمد على نفس API.)

---

## 14. المتطلبات غير الوظيفية

- **الأمان:** JWT قصير + Refresh بتدوير، تجزئة OTP، Rate limit، تحقق مدخلات صارم، CORS مقيّد، أسرار في متغيرات بيئة، HTTPS إلزامي، حماية من التلاعب بالأسعار (حساب خادم فقط).
- **الأداء:** البحث الجغرافي من Redis. فهارس GiST على `zones.polygon` و`stores.location`. فهرس على `orders(status, placed_at)`.
- **المراقبة:** Sentry (كل التطبيقات)، سجلات Pino منظّمة، مراقبة جاهزية الخادم وتنبيه، `GET /health`.
- **النسخ الاحتياطي:** `pg_dump` يومي مضغوط إلى تخزين خارجي + استعادة تجريبية شهرية. (+ نسخ لـ Redis غير ضروري، فهو مؤقت.)
- **التحديثات:** إصدارات API + حد أدنى مفروض للتطبيقات (426).
- **المنطقة الزمنية:** تخزين UTC، عرض وحساب ساعات العمل بتوقيت اليمن (UTC+3).
- **الأتمتة (CI/CD):** GitHub Actions لكل مستودع: lint + test + build. الـ Backend واللوحات تُنشر تلقائيًا إلى الـ VPS عند الدمج في `main`. التطبيقات تُبنى (APK/AAB) آليًا.
- **المتاجر:** Google Play فقط في الإطلاق (iOS لاحقًا عبر Flutter).

---

## 15. استراتيجية الاختبار

- **وحدة (Unit):** (إلزامي) حساب الأسعار والعمولات وأجر السائق، آلة الحالات، قيود الـ Ledger وتوازنها، حساب فتح/إغلاق المتجر، خطوات البث.
- **تكامل (Integration):** Testcontainers (Postgres+PostGIS وRedis): تدفق الطلب كاملًا، `ST_Contains` للمناطق، `GEOSEARCH`، **اختبار التزامن للقبول (50 متزامنًا ⇒ فائز واحد)**، المخزون الذري، الـ Idempotency، تدقيق الـ Ledger.
- **E2E:** Playwright للوحتين، و`integration_test` لتدفقات Flutter الأساسية (طلب كامل بين تطبيقي العميل والسائق على Staging).
- **تغطية دنيا مقترحة:** 90% للوحدات المالية، 70% عامة.

---

## 16. خطة المراحل والمهام

### المرحلة 1 — الأساس (يُشغِّل الطلب من الإنشاء إلى التسليم والحسابات)

**الترتيب والمهام (مع معيار القبول):**

1. **`delivery-infra`:** Docker Compose (Postgres+PostGIS، Redis، Nginx، Backend)، خادم البلاط لليمن/صنعاء، النسخ الاحتياطي، سكربت النشر. ✔ *يقوم `docker compose up` بتشغيل البيئة كاملة.*
2. **Backend — الأساس:** هيكل NestJS، Drizzle + الهجرات، الإعدادات، Auth (OTP واتساب + JWT)، RBAC والـ Audit Log، `app-version`. ✔ *OTP يعمل، وصلاحيات الأدمن مفروضة.*
3. **Backend — الكتالوج:** المتاجر والأنواع والفئات والمنتجات والخيارات والإضافات والمخزون وساعات العمل. ✔ *منطق مفتوح/مغلق صحيح باختبارات.*
4. **Backend — المناطق:** CRUD المناطق، `ST_Contains`، مصفوفة الأسعار. ✔ *دبوس ← منطقة ← سعر صحيح.*
5. **Backend — الطلبات والـ Ledger:** إنشاء الطلب، آلة الحالات، الحسابات المالية الكاملة، المحفظة والشحن اليدوي، التسويات. ✔ *الاختبارات المالية خضراء والـ Ledger متوازن.*
6. **Backend — البث والوقت الحقيقي:** Redis GEO، التوسع التدريجي (BullMQ)، القبول الذري، الانسحاب، الإسناد اليدوي، Socket.IO، FCM والواتساب. ✔ *اختبار التزامن يمر، وطلب بلا سائق يصل لتنبيه الأدمن.*
7. **Backend — التقارير الأساسية والمالية** + تصدير Excel/PDF.
8. **`delivery-admin-web`:** كل ما في القسم 13.4.
9. **`delivery-merchant-web`:** كل ما في القسم 13.3 (بدون الفروع/الموظفين).
10. **`delivery-driver-app`:** القسم 13.2.
11. **`delivery-customer-app`:** القسم 13.1 (بدون الميزات المؤجلة).
12. **اختبار شامل على Staging** ثم إطلاق تجريبي مع المتاجر والسائقين أصحاب الاتفاقات.

> **نصيحة تنفيذ:** ابنِ "شريحة رأسية" مبكرًا (طلب واحد من البداية للنهاية) قبل توسيع الميزات، وعرضها على أصحاب الاتفاقات.

### المرحلة 2
أكواد الخصم + الاشتراك (منح يدوي) + التوصيل المجاني/خصم المناطق البعيدة (ومن يتحمل التكلفة **يُحدَّد لكل عرض**: المنصة/التاجر/مناصفة)، التقييمات (متجر وسائق)، نظام التذاكر والشكاوى (عميل/سائق/تاجر)، التحليلات المتقدمة (ذروة، إلغاء، أداء، خريطة حرارية)، فروع التاجر وموظفوه بصلاحيات، تطبيق التاجر Flutter، العقوبات التلقائية الموسّعة، iOS.

---

## 17. افتراضات وأسئلة مفتوحة (ينفّذ الوكيل بالافتراض ويبلّغ)

1. **تطبيق التاجر (Flutter):** وُضع في المرحلة 2، والمرحلة 1 تكتفي بلوحة الويب (تنبيه صوتي + طباعة). *يمكن نقله للمرحلة 1 بقرار من صاحب المشروع.*
2. **تصعيد تأخر التاجر:** لم يُحسم، فلا يُنفَّذ في المرحلة 1 (التاجر مستقبِل فقط)، ويُترك خطاف (hook) للإضافة.
3. **دخول الأدمن:** بريد + كلمة مرور (+2FA اختياري) بدل OTP.
4. **المرحلة 1 تحوي متجرًا واحدًا لكل تاجر** لكن الجداول جاهزة للفروع.
5. **الخصم من أرباح السائق لتغطية تسليم الكاش:** غير مفعّل؛ التسليم يدوي يومي فقط.
6. **أي مدفوعات إلكترونية خارجية (محافظ محلية):** خارج النطاق الحالي، تُربَط لاحقًا فوق الـ Ledger.
7. **رسوم خرائط:** لا شيء مدفوعًا. إن لم تُغطِّ بيانات OSM حيًا ما، يُضاف الدبوس والوصف النصي.
8. **القيم الافتراضية** (نصف القطر 2/4/6 كم، المهلة 60 ثانية، عتبة الانسحاب 3/يوم، الإيقاف ساعتان، heartbeat 60 ثانية) قابلة للضبط من `settings` في لوحة الأدمن.

---

## 18. تعريف الاكتمال (Definition of Done) للمرحلة 1

- عميل ينشئ طلبًا من تطبيق Flutter، فيصل للتاجر (لوحة الويب) وللسائقين القريبين، وسائق يقبله ويحرّك حالاته، والعميل يتتبعه ويستلمه.
- الدفع كاش ومحفظة يُقيَّدان صحيحًا في الـ Ledger مع عمولة التاجر وأجر السائق.
- الأدمن يدير المناطق والأسعار والسائقين والتجار والمحافظ وتسليم الكاش والتقارير، وكل إجراء مسجَّل في Audit Log.
- اختبارات الوحدة والتكامل (ومنها التزامن والمال) خضراء في CI، والنشر آلي، والنسخ الاحتياطي يعمل ومُختبَر.
