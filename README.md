# منصة التوصيل متعددة المتاجر — صنعاء

هذا المستودع يحوي وثيقة التحليل الأصلية: [docs/ANALYSIS.md](docs/ANALYSIS.md). التنفيذ موزّع على المستودعات التالية:

| المستودع | الدور |
|---|---|
| [delivery-backend](https://github.com/ahmedalgarbani/delivery-backend) | الخادم: REST + Socket.IO + Workers، PostgreSQL/PostGIS، Redis، الـ Ledger، البث |
| [delivery-infra](https://github.com/ahmedalgarbani/delivery-infra) | Docker Compose، Nginx/TLS، خادم البلاط، النسخ الاحتياطي، النشر |
| [delivery-admin-web](https://github.com/ahmedalgarbani/delivery-admin-web) | لوحة الأدمن |
| [delivery-merchant-web](https://github.com/ahmedalgarbani/delivery-merchant-web) | لوحة التاجر |
| [delivery-customer-app](https://github.com/ahmedalgarbani/delivery-customer-app) | تطبيق العميل (Flutter) |
| [delivery-driver-app](https://github.com/ahmedalgarbani/delivery-driver-app) | تطبيق السائق (Flutter) |
| [delivery-merchant-app](https://github.com/ahmedalgarbani/delivery-merchant-app) | تطبيق التاجر (Flutter، المرحلة 2) |
| [delivery-docs](https://github.com/ahmedalgarbani/delivery-docs) | القرارات، المخططات، عقد الـ API، أدلة التشغيل وحالة التنفيذ |

ابدأ من `delivery-docs/STATUS.md` لمعرفة ما اكتمل وما يحتاج إجراءً منك قبل الإطلاق، ومن
`delivery-docs/runbooks/local-development.md` لتشغيل المنصة كاملة محليًا.
