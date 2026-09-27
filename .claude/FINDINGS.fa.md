# گزارش بررسی ریپازیتوری vectorcore-smsc

**ریپو:** majiddarvishan/vectorcore-smsc (فورک از پروژه‌ی VectorCore Mobile / svinson1121)
**نسخه:** 0.4.0b (بتا) · **زبان:** Go 1.25 + React (Vite) · **لایسنس:** Apache 2.0

---

## ۱. این پروژه چیه؟
یک **SMSC و IP-SM-GW** چندواسطی برای شبکه‌های LTE/IMS. پیامک رو از چند پروتکل می‌گیره، مسیریابی می‌کنه و با مکانیزم Store-and-Forward تحویل می‌ده:

| جهت | واسط‌ها |
|---|---|
| ورودی (Ingress) | SMPP Server (TCP و TLS)، SIP ISC از S-CSCF (MESSAGE / REGISTER / NOTIFY)، SIP SIMPLE، Diameter SGd (MO از MME) |
| خروجی (Egress) | SMPP Client (TCP/TLS)، SIP/3GPP ISC به UE در IMS، SIP SIMPLE به سایت‌های دیگر، Diameter SGd به MME (با کمک S6c) |
| لوک‌آپ مشترک | Diameter **Sh** (وضعیت ثبت IMS و S-CSCF از HSS)، Diameter **S6c** (IMSI و MME برای SMS-in-MME) |
| ذخیره‌سازی | PostgreSQL (با LISTEN/NOTIFY) یا SQLite (با polling) |
| عملیات | REST API روی `/api/v1` + OpenAPI، Prometheus روی `/metrics`، `/health`، UI ری‌اکت روی `/ui/` |

**چیزهایی که نداره:** SS7/MAP/SIGTRAN (یعنی برای 2G/3G مستقیم کار نمی‌کنه)، CDR/بیلینگ، HTTP API برای ارسال پیامک (فقط مدیریت)، کلاستر/HA.

## ۲. معماری کد
```
cmd/smsc/main.go          → سیم‌کشی همه اجزا (~30KB، خیلی بزرگ)
internal/
  codec/                  → Message کانونیکال + کدک‌های smpp، sip3gpp، sipsimple، sgd، tpdu (GSM 03.40)
  forwarder/              → قلب سیستم: dispatch، retry (هر ۱۵ ثانیه)، expiry (هر ۶۰ ثانیه)
  routing/                → موتور قوانین fallback با hot-reload
  registry/               → کش ثبت IMS + Sh refresh + کش S6c
  diameter/               → FSM پیر، tcp/sctp، کدک AVP، اپ‌های sh، s6c، sgd
  sip/                    → isc (SMS over IP) و simple (+IMDN)
  smpp/                   → PDU، server (bcrypt auth)، client manager
  dr/                     → تطبیق و تولید Delivery Report
  store/                  → اینترفیس Store + postgres و sqlite (هر کدوم schema.sql خودش)
  api/  metrics/  numbering/  sgdmap/  config/
web/                      → UI ری‌اکت (embed میشه داخل باینری)
```

## ۳. جریان پیام
1. دیکد به `codec.Message` و نرمال‌سازی MSISDN مقصد
2. ذخیره در DB با وضعیت `DISPATCHED` (قبل از ارسال شبکه، پس با کرش پیام گم نمیشه)
3. ارزیابی کاندیداها به ترتیب ثابت:
   - `ims-local`: اگه مشترک توی کش محلی IMS رجیستر باشه → ISC
   - `ims-sh`: کوئری زنده Sh به HSS → ISC
   - `sgd-built-in`: کوئری S6c → مپینگ نام MME → SGd OFR
   - قوانین fallback از DB (فقط smpp یا sipsimple) به ترتیب priority
4. نتیجه: `DELIVERED` / `WAIT_TIMER` / `WAIT_EVENT` / `WAIT_TIMER_EVENT` / `FAILED`، و بعداً `EXPIRED`
5. Retry: اولی ۳۰ ثانیه بعد، بعدش طبق SF Policy یا پیش‌فرض `[30, 300, 1800, 3600×5]` ثانیه. Alertها (S6c ALSC و SGd ALR) پیام‌های منتظر رو دوباره صف می‌کنن، با transition محافظت‌شده که race نشه.

## ۴. نقاط قوت
- جداسازی تمیز پکیج‌ها و مدل کانونیکال پیام، اضافه کردن واسط جدید راحته
- Persist قبل از ارسال، و ماشین حالت واقعی در DB
- تست خوب: تست واحد + **fuzz test** برای همه کدک‌ها (TPDU، Diameter AVP، SMPP، SIP)
- Hot-reload تنظیمات پیرها و قوانین بدون ری‌استارت
- پسوردهای SMPP با bcrypt، پسوردها از API برنمی‌گردن
- مستندات API و Routing دقیق و به‌روز

## ۵. مشکلات و ریسک‌ها (به ترتیب اهمیت)

### 🔴 بحرانی
1. **REST API و UI هیچ احراز هویتی ندارن.** در `internal/api/server.go` هیچ middleware امنیتی نیست و پیش‌فرض روی `::` (همه اینترفیس‌ها) پورت 8080 گوش میده. هر کسی به شبکه دسترسی داشته باشه می‌تونه اکانت SMPP بسازه، قوانین مسیریابی رو عوض کنه، پیر Diameter اضافه کنه یا صف پیام رو پاک کنه. پیشنهاد: حداقل Basic/Token Auth یا mTLS، و bind پیش‌فرض روی `127.0.0.1`.
2. **کلیدهای خصوصی داخل `test-certs/` کامیت شدن.** فقط برای تسته، ولی خطر استفاده اشتباهی در پروداکشن هست.

### 🟠 مهم
3. **باینری ۳۴ مگابایتی `bin/smsc` داخل گیت کامیت شده.** باید به `.gitignore` اضافه بشه و از تاریخچه پاک بشه.
4. **`make install` خرابه:** فایل `config.yaml` رو کپی می‌کنه که توی ریپو وجود نداره (فایل واقعی `config/smsc.yaml` هست). خود مستندات هم اینو قبول دارن.
5. **Failover برای HSS نداره:** فقط اولین پیر فعال Sh و اولین پیر فعال S6c استفاده میشه.
6. **لوک‌آپ Sh برای هر پیام:** برای هر مشترکی که توی کش محلی نیست، یه کوئری زنده Sh زده میشه. زیر بار بالا، هم HSS فشار می‌خوره هم تاخیر ارسال بالا میره. کش منفی (negative cache) لازمه.
7. **خطاهای DB نادیده گرفته میشن:** مثلاً `_ = f.st.UpdateMessageStatus(...)` در forwarder. اگه آپدیت DELIVERED شکست بخوره، پیام ممکنه دوباره ارسال بشه (duplicate).
8. **Dispatch همگام (synchronous) هست:** در callbackهای ورودی مستقیم `fwd.Dispatch` صدا زده میشه، یعنی ارسال شبکه و کوئری‌های Diameter داخل مسیر ورودی اتفاق میفته. برای throughput بالا صف/worker pool لازمه.

### 🟡 جزئی
9. `/api/v1/status` وضعیت‌های `WAIT_*` رو نمی‌شمره، پس عدد صف گمراه‌کننده است.
10. `routePolicy()` برای هر تلاش چند بار `GetSFPolicy` رو از DB می‌خونه، کش نداره.
11. `fallbackDecisions()` چند بار در هر pass دوباره محاسبه میشه.
12. تشخیص نوع SIP MESSAGE با مقایسه ۱۹ کاراکتر اول Content-Type (`application/vnd.3gp`) انجام میشه، شکننده است.
13. `allowed_ip` فقط یک IP دقیق قبول می‌کنه (CIDR یا لیست نه).
14. Module path هنوز `github.com/svinson1121/...` هست. اگه قراره مستقل توسعه بدی، باید عوض بشه.
15. `main.go` حدود ۳۰KB و خیلی شلوغه، منطق sync پیرهای Diameter بهتره به پکیج جدا منتقل بشه.

## ۶. پیشنهاد اولویت کار
1. اضافه کردن Auth به API + تغییر bind پیش‌فرض
2. حذف `bin/` و `test-certs` از گیت + `.gitignore`
3. درست کردن `make install`
4. هندل کردن خطاهای آپدیت وضعیت + جلوگیری از ارسال تکراری
5. کش منفی Sh و کش SF Policy
6. Failover بین چند HSS
7. اگه شبکه 2G/3G هم داری: فکر اتصال به یه SS7/MAP gateway

## ۷. اجرای سریع
```bash
make                           # UI + باینری
bin/smsc -c config/smsc.yaml   # پیش‌فرض SQLite، API روی :8080
curl localhost:8080/api/v1/status
```
پورت‌ها: SMPP 2775، SIP 5060 (UDP)، Diameter 3868، API 8080.
