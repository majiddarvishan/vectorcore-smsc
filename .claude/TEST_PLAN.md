# برنامه تست‌های Race و Integration

> این برنامه از روی گزارش `.claude/REVIEW.fa.md` نوشته شده. کدهای BUG / PERF / SEC / ARCH / CC به همون گزارش ارجاع میدن.
> **وضعیت فعلی: فقط برنامه‌ریزی شده. هنوز هیچ تستی پیاده‌سازی نشده.**

## قواعد علامت‌گذاری

- هر تسک با `- [ ]` شروع میشه. وقتی انجام شد به `- [x]` تغییر کنه.
- هر تست دو مرحله داره:
  - **تست:** تست نوشته و merge شده. اگه باگ هنوز باز باشه، تست فعلاً fail میشه و پشت گارد known-bug قرار می‌گیره (پایین‌تر توضیح دادم).
  - **فیکس:** باگ درست شده، گارد برداشته شده و تست در CI سبزه.
- ستون «الان»: ❌ یعنی انتظار داریم روی کد فعلی fail بشه (باگ تأییدشده). ✅ یعنی باید همین الان پاس بشه (گارد رگرسیون).
- «پیش‌نیاز»: 🔧 یعنی قبل از نوشتن تست یه refactor کوچیک لازمه. 🧩 یعنی اول باید فیچرش ساخته بشه (مثل auth).

## قراردادها

- تست‌های race با `go test -race` اجرا میشن و باید deterministic باشن (با hook یا کانال، نه با `time.Sleep` تصادفی).
- تست‌های integration پشت build tag `//go:build integration` قرار می‌گیرن.
- هر تست store روی **هر دو backend** (SQLite و Postgres) به صورت table-driven اجرا میشه.
- تست‌هایی که باگ‌های باز رو پوشش میدن با `testutil.KnownBug(t, "BUG-5")` شروع میشن. این تابع تست رو skip می‌کنه، مگر اینکه `SMSC_RUN_KNOWN_BUGS=1` ست شده باشه. این‌طوری CI سبز می‌مونه و باگ‌ها هم مستند باقی می‌مونن.
- برای تشخیص نشت گوروتین از `go.uber.org/goleak` استفاده میشه (وابستگی جدید).

---

## فاز ۰: زیرساخت تست

- [ ] **INF-01** تارگت `make test-race` → `go test -race -count=1 ./...`
- [ ] **INF-02** تارگت `make test-integration` → `go test -race -tags=integration -count=1 ./...`
- [ ] **INF-03** فایل `docker-compose.test.yml` با Postgres 16 و متغیر `SMSC_TEST_PG_DSN`
- [ ] **INF-04** پکیج `internal/testutil`:
  - [ ] `NewSQLiteStore(t)` روی فایل موقت (نه `:memory:`، چون WAL و چند handle هم باید تست بشه)
  - [ ] `NewPostgresStore(t)` با schema یکتا برای هر تست؛ اگه DSN نبود skip کنه
  - [ ] `ForEachBackend(t, func(t, store.Store))`
  - [ ] `KnownBug(t, id)`
  - [ ] `FailingStore` (wrapper که متدهای انتخابی رو با خطا برمی‌گردونه)
- [ ] **INF-05** fakeهای egress در `internal/testutil/fakes`:
  - [ ] `FakeISCSender`، `FakeSimpleSender`، `FakeSGdSender` با خطا، تأخیر و hook قابل تنظیم (`OnSend func()`) و شمارنده فراخوانی
  - [ ] `FakeReporter` که همه Reportها رو ثبت می‌کنه
  - [ ] 🔧 اینترفیس برای `SMPPManager` در forwarder (الان concrete `*smppClient.Manager` هست و fake کردنش ممکن نیست)
- [ ] **INF-06** سرورهای جعلی شبکه:
  - [ ] `FakeESME` / `FakeSMSC` (SMPP روی `127.0.0.1:0`)
  - [ ] `FakeDiameterPeer` (CER/CEA، DWR/DWA، پاسخ دلخواه به OFR/TFR/UDR/SRR)
  - [ ] `FakeSIPUA` با sipgo روی UDP لوکال
- [ ] **INF-07** 🔧 تزریق ساعت (`Clock` interface) به forwarder، retry و expiry برای تست‌های زمانی بدون sleep
- [ ] **INF-08** GitHub Actions: job یکم unit+race، job دوم integration با Postgres service، job سوم nightly با `SMSC_RUN_KNOWN_BUGS=1` (فقط گزارش، non-blocking)
- [ ] **INF-09** `goleak.VerifyTestMain` در پکیج‌های `smpp`، `diameter`، `forwarder`، `sip`

---

## فاز ۱: تست‌های Race / همزمانی

| ID | الان | پیش‌نیاز |
|---|---|---|

- [ ] **R-01** نقشه‌های پیر Diameter در `main.go` (BUG-7) · ❌ · 🔧
  - سناریو: منطق `syncHSSPeers` و `syncSGdPeers` به یه struct به اسم `diameterRuntime` منتقل میشه. بعد ۵۰ گوروتین همزمان `Sync()`، `PeerStatuses()` و آپدیت metrics رو صدا می‌زنن.
  - انتظار: detector هیچ race ای گزارش نده و panic نشه.
  - [ ] فیکس
- [ ] **R-02** `dr.Correlator.SetS6cReporter` در برابر `Report` همزمان (BUG-30) · ❌
  - [ ] فیکس
- [ ] **R-03** `sgd.Server.onMsg` که در `handleMOForwardRequest` بدون قفل خونده میشه، همزمان با `SetOnMessage` (BUG-30) · ❌
  - [ ] فیکس
- [ ] **R-04** `routing.Engine.Reload(rules)` وقتی caller بعد از Reload اسلایس رو تغییر میده، همزمان با `RouteAll` (CC-7) · ❌
  - [ ] فیکس
- [ ] **R-05** `registry`: همزمانی `Lookup`، `Upsert`، `loadAll` و `ShRefresh` · ✅
  - [ ] فیکس (در صورت نیاز)
- [ ] **R-06** سالم ماندن frameهای Diameter زیر بار (BUG-12) · ❌
  - سناریو: `net.Pipe`؛ ۱۰۰ گوروتین `peer.Send` می‌کنن، همزمان watchdog با interval ده میلی‌ثانیه‌ای DWR می‌فرسته و طرف مقابل DWR می‌فرسته که DWA بگیره. خواننده همه frameها رو decode می‌کنه.
  - انتظار: همه frameها decode بشن و هیچ خطای length یا version نباشه.
  - [ ] فیکس
- [ ] **R-07** `smpp.Link`: همزمانی `SendAndWait`، `DispatchPending` و `Close` · ❌
  - [ ] با `Close`، همه pendingها باید فوراً با خطا برگردن (BUG-16 و بخش SMPP-07 گزارش)
  - [ ] seq رو روی `0x7FFFFFFE` بذار و ۱۰ درخواست بفرست؛ seq هیچ‌وقت صفر یا بزرگ‌تر از `0x7FFFFFFF` نشه و pending بازنویسی نشه (BUG-18)
  - [ ] فیکس
- [ ] **R-08** `smppClient.Manager`: `reload` در حالی که `SendViaPeer` و `SetOnMessage` همزمان صدا زده میشن (PERF-10، CC-7) · ❌
  - انتظار: `SendViaPeer` در طول reload بیشتر از ۱۰۰ میلی‌ثانیه block نشه؛ callback جدید روی sessionهای موجود اعمال بشه.
  - [ ] فیکس
- [ ] **R-09** Race بین expiry و تحویل (BUG-1) · ❌ · هر دو backend
  - سناریو (deterministic): پیام در `WAIT_TIMER` با `expiry_at` گذشته. `FakeSender.OnSend` وسط ارسال `sweeper.sweep()` رو صدا می‌زنه و بعد موفق برمی‌گرده.
  - انتظار: وضعیت نهایی `DELIVERED` باشه (نه `EXPIRED`) و reporter فقط یک بار صدا زده بشه (`DELIVRD`).
  - [ ] فیکس
- [ ] **R-10** `ClaimMessageForDispatch` همزمان: ۲۰ گوروتین روی یک id · ✅ · هر دو backend
  - انتظار: دقیقاً یکی `true` بگیره.
- [ ] **R-11** Alert requeue در برابر retry claim: ALR و tick اسکجولر همزمان · ✅/❌ · هر دو backend
  - انتظار: sender دقیقاً یک بار صدا زده بشه.
  - [ ] فیکس (در صورت نیاز)
- [ ] **R-12** چند نمونه روی یک Postgres (BUG-4، ARCH-6) · ❌
  - سناریو: store A یه پیام رو `DISPATCHED` می‌کنه، بعد store B با `Open` روی همون DB باز میشه.
  - انتظار: پیام A دست نخوره (بعد از فیکس: فقط leaseهای منقضی recover بشن).
  - [ ] فیکس
- [ ] **R-13** SMPP server زیر بار: ESME با ۵۰۰۰ `submit_sm` در ثانیه (PERF-4) · ❌
  - انتظار: تعداد گوروتین‌ها سقف داشته باشه (مثلاً کمتر از ۲۰۰)؛ اگه صف پر شد `ESME_RTHROTTLED` برگرده.
  - [ ] فیکس

---

## فاز ۲: Integration، لایه Store (هر دو backend)

- [ ] **I-S01** SQLite: retry و expiry پیامی که سررسیدش «همین الانه» (BUG-5) · ❌
  - `next_retry_at = now-1s` → باید در `ListRetryableMessages` باشه. `expiry_at = now-1s` → باید در `ListExpiredMessages` باشه.
  - [ ] فیکس
- [ ] **I-S02** Round-trip همه فیلدهای همه مدل‌ها در هر دو backend (BUG-26، BUG-30) · ❌
  - [ ] `IMSRegistration.IMSI` در Postgres
  - [ ] `Message.TPMR`
  - [ ] بقیه مدل‌ها (SMPP account و client، SIP peer، Diameter peer، rule، policy، subscriber، MME mapping)
  - [ ] فیکس
- [ ] **I-S03** Migration (BUG-27) · ❌
  - [ ] `Open` روی DB خالی موفق باشه؛ دو بار `Open` پشت سر هم idempotent باشه
  - [ ] migration خراب (با تزریق) باید خطا برگردونه، نه اینکه ادامه بده
  - [ ] جدول نسخه schema وجود داشته باشه
  - [ ] فیکس
- [ ] **I-S04** Postgres: LISTEN/NOTIFY بعد از قطع اتصال (BUG-24) · ❌
  - سناریو: `pg_terminate_backend` روی connection مربوط به listener، بعد insert یه routing rule.
  - انتظار: subscriber ظرف ۱۰ ثانیه event بگیره.
  - [ ] فیکس
- [ ] **I-S05** SQLite polling (BUG-25) · ❌
  - [ ] حذف یه rule باید event بده
  - [ ] دو آپدیت در یک ثانیه باید هر دو دیده بشن (یا حداقل reload نهایی درست باشه)
  - [ ] فیکس
- [ ] **I-S06** انتقال شرطی وضعیت (ARCH-1) · ❌ · 🧩
  - `DELIVERED → EXPIRED` رد بشه؛ `FAILED → DELIVERED` رد بشه؛ خروجی نشون بده تغییر اعمال شده یا نه.
  - [ ] فیکس
- [ ] **I-S07** Recovery بعد از کرش (BUG-4) · ❌ · 🧩
  - `DISPATCHED` با lease منقضی → `QUEUED`. `DISPATCHED` با lease معتبر → دست نخوره.
  - [ ] فیکس
- [ ] **I-S08** SQLite: دو handle روی یک فایل با نوشتن همزمان (BUG-8 گزارش store، PERF-9) · ❌
  - انتظار: خطای `SQLITE_BUSY` نگیریم (با busy_timeout).
  - [ ] فیکس
- [ ] **I-S09** کوئری صف با فیلتر و ۱۰۰ هزار ردیف: زیر ۲۰۰ میلی‌ثانیه (Postgres، PERF-7) · ❌
  - [ ] فیکس
- [ ] **I-S10** Retention: پیام‌های نهایی قدیمی‌تر از N روز purge بشن (PERF-5) · ❌ · 🧩
  - [ ] فیکس

---

## فاز ۳: Integration، Forwarder (store واقعی + fakeها)

- [ ] **I-F01** شکست persist یعنی ارسالی انجام نشه (BUG-2) · ❌
  - `FailingStore.SaveMessage` خطا بده → تعداد فراخوانی sender صفر باشه و `Dispatch` خطا برگردونه.
  - [ ] فیکس
- [ ] **I-F02** شکست آپدیت `DELIVERED` یعنی پیام نه گم بشه و نه دو بار بره (BUG-3) · ❌
  - [ ] فیکس
- [ ] **I-F03** Rule غیرفعال نباید match بشه (BUG-6) · ❌
  - [ ] فیکس
- [ ] **I-F04** Policy با `retry_schedule` خالی (BUG-15) · ❌
  - انتظار: panic نشه؛ پیام `FAILED` بشه یا از schedule پیش‌فرض استفاده بشه.
  - [ ] فیکس
- [ ] **I-F05** `TPMR` و `Expiry` مطلق بعد از retry حفظ بشن (BUG-30، ARCH-3) · ❌
  - [ ] فیکس
- [ ] **I-F06** مسیر alert در SGd باید `vp_override` رو اعمال کنه (BUG-30) · ❌
  - [ ] فیکس
- [ ] **I-F07** شبیه‌سازی کرش: بعد از persist و قبل از finalize، context لغو بشه و store دوباره باز بشه · ❌
  - انتظار: پیام در نهایت **دقیقاً یک بار** تحویل بشه.
  - [ ] فیکس
- [ ] **I-F08** Registry: ثبت‌نامی که از DB حذف شده بعد از reload از کش هم حذف بشه (BUG-8) · ❌
  - [ ] فیکس
- [ ] **I-F09** `Dispatch(nil)` نباید panic کنه (BUG-30) · ❌
  - [ ] فیکس
- [ ] **I-F10** Sender کند نباید بقیه retryها رو block کنه (PERF-3) · ❌
  - یک پیام با sender پنج ثانیه‌ای و ۱۰ پیام سالم؛ همه ۱۰ تا باید ظرف یک tick تحویل بشن.
  - [ ] فیکس
- [ ] **I-F11** Sh کند نباید ورودی رو block کنه (PERF-1، PERF-2) · ❌
  - fake Sh با تأخیر ۳ ثانیه‌ای؛ ۱۰۰ پیام ورودی باید زیر یک ثانیه persist بشن.
  - [ ] فیکس
- [ ] **I-F12** DR idempotent: دو بار finalize یعنی یک ردیف DR و یک ارسال DR (ARCH-4) · ❌
  - [ ] فیکس
- [ ] **I-F13** ترتیب کامل مسیریابی: ims-local → ims-sh → sgd → fallback، با حالت‌های 404، `UNABLE_TO_DELIVER` و timeout · ✅
- [ ] **I-F14** Retry schedule و `max_retries` و `max_ttl` با Clock تزریقی (INF-07) · ✅

---

## فاز ۴: Integration، پروتکل SMPP (TCP واقعی روی localhost)

- [ ] **I-P01** `submit_sm` با UDH خراب نباید `ESME_ROK` بگیره (BUG-9) · ❌
  - [ ] فیکس
- [ ] **I-P02** FSM bind (BUG-10) · ❌
  - [ ] bind از نوع receiver با `submit_sm` رد بشه
  - [ ] bind دوم روی همون session خطا بگیره
  - [ ] command ناشناخته `generic_nack` بگیره
  - [ ] `deliver_sm` از ESME (DR) پذیرفته و ack بشه
  - [ ] فیکس
- [ ] **I-P03** Timeout برای bind: اتصالی که چیزی نمی‌فرسته ظرف N ثانیه بسته بشه (SEC-5) · ❌
  - [ ] فیکس
- [ ] **I-P04** `command_length` بزرگ (۱۶MB) بدون بدنه: اتصال فوراً بسته بشه و allocation بزرگ انجام نشه (SEC-5) · ❌
  - [ ] فیکس
- [ ] **I-P05** اتصال‌های بدون احراز هویت نباید `max_connections` رو پر کنن و مانع bind معتبر بشن (SEC-5) · ❌
  - [ ] فیکس
- [ ] **I-P06** کلاینت SMPP (BUG-19) · ❌
  - [ ] `bind_resp` با sequence اشتباه رد بشه
  - [ ] `enquire_link_resp` قدیمی timer رو ریست نکنه
  - [ ] فیکس
- [ ] **I-P07** نشت گوروتین در ۲۰ چرخه reconnect (goleak) (BUG-16) · ❌
  - [ ] فیکس
- [ ] **I-P08** اعمال `throughput_limit` (BUG-17) · ❌
  - limit ده پیام در ثانیه، ارسال ۵۰ پیام، زمان کل باید حداقل ۴ ثانیه باشه.
  - [ ] فیکس
- [ ] **I-P09** `message_id` بعد از ری‌استارت تکراری نشه (ARCH-6) · ❌
  - [ ] فیکس
- [ ] **I-P10** End-to-end: `submit_sm` از ESME → `FakeSMSC` → `deliver_sm` DR → DR به ESME برگرده · ✅
- [ ] **I-P11** TLS خروجی بدون `verify_server_cert` صریح نباید به سرور با cert نامعتبر وصل بشه (SEC-4) · ❌
  - [ ] فیکس
- [ ] **I-P12** IP مجاز قبل از bcrypt چک بشه و پیام خطا برای system_id ناشناخته و پسورد اشتباه یکی باشه (SEC-5) · ❌
  - [ ] فیکس

---

## فاز ۵: Integration، Diameter (peer جعلی روی TCP)

- [ ] **I-D01** CER از `Origin-Host` یا IP غیرمجاز رد بشه (SEC-2) · ❌ · 🧩
  - [ ] فیکس
- [ ] **I-D02** یکتایی End-to-End ID در ۱۰ هزار درخواست (BUG-13) · ❌
  - [ ] فیکس
- [ ] **I-D03** MT برای MME ناشناخته نباید به MME نامربوط بره؛ فقط به peerهایی که صریحاً proxy یا DRA کانفیگ شدن (BUG-14) · ❌
  - [ ] فیکس
- [ ] **I-D04** `StopGraceful` روی peer ورودی ظرف timeout برگرده (BUG-30) · ❌
  - [ ] فیکس
- [ ] **I-D05** S6c: پاسخی که فقط `Experimental-Result` داره درست تفسیر بشه · ❌
  - [ ] فیکس
- [ ] **I-D06** Reconnect بعد از ری‌استارت peer؛ backoff بعد از OPEN ریست بشه · ❌
  - [ ] فیکس
- [ ] **I-D07** جریان کامل SGd: S6c SRR → MT-Forward → `UNABLE_TO_DELIVER` → `WAIT_EVENT` → ALR → تحویل دقیقاً یک بار · ✅
- [ ] **I-D08** Diameter با `length` بزرگ از peer تأییدنشده allocation بزرگ انجام نده · ❌
  - [ ] فیکس
- [ ] **I-D09** MO-Forward ورودی باید بعد از نتیجه ISC با SM-RP-UI جواب بگیره (`trackPendingMO` هیچ‌وقت صدا زده نمیشه، CC-2) · ❌
  - [ ] فیکس

---

## فاز ۶: Integration، SIP

- [ ] **I-SIP01** REGISTER از منبع غیرمجاز نباید کش IMS رو تغییر بده (SEC-3) · ❌ · 🧩
  - [ ] فیکس
- [ ] **I-SIP02** NOTIFY با `P-Asserted-Identity` جعلی نباید ثبت‌نام رو حذف کنه (SEC-3) · ❌ · 🧩
  - [ ] فیکس
- [ ] **I-SIP03** MESSAGE خراب (بدون Request-URI یا با بدنه خالی) باید 400 بگیره و panic نشه (BUG-30) · ❌
  - [ ] فیکس
- [ ] **I-SIP04** Content-Type با حروف بزرگ یا پارامتر (`Application/VND.3gpp.sms; charset=...`) درست route بشه (CC-7) · ❌
  - [ ] فیکس
- [ ] **I-SIP05** IMDN با message-id حاوی `<&"` درست escape بشه (SEC-8) · ❌
  - [ ] فیکس
- [ ] **I-SIP06** ISC MT: 200 OK بدون RP-ACK نباید پیام رو `DELIVERED` کنه (ARCH-5) · ❌
  - [ ] فیکس
- [ ] **I-SIP07** NOTIFY با بدنه reginfo که contact رو terminated می‌کنه، کش رو آپدیت کنه (ARCH-8) · ❌
  - [ ] فیکس
- [ ] **I-SIP08** SIMPLE IMDN: پاسخ 1xx یا بدون پاسخ نباید موفق حساب بشه · ❌
  - [ ] فیکس
- [ ] **I-SIP09** جریان کامل ISC: REGISTER → MESSAGE MT → RP-ACK → DELIVERED + DR · ✅

---

## فاز ۷: Integration، REST API (`httptest`)

- [ ] **I-A01** درخواست بدون احراز هویت 401 بگیره؛ `/metrics` و `/docs` هم محافظت بشن (SEC-1) · ❌ · 🧩
  - [ ] فیکس
- [ ] **I-A02** `limit=1000000` سقف‌دار بشه یا 400 بگیره (PERF-6) · ❌
  - [ ] فیکس
- [ ] **I-A03** PUT روی SIP peer بدون `auth_pass` پسورد قبلی رو حفظ کنه (BUG-28) · ❌
  - [ ] فیکس
- [ ] **I-A04** خطای DB متن خام SQL رو به کلاینت برنگردونه (SEC-1) · ❌
  - [ ] فیکس
- [ ] **I-A05** وقتی DB بسته‌ست، `/health` یا readiness غیر 200 برگردونه · ❌
  - [ ] فیکس
- [ ] **I-A06** اعتبارسنجی ورودی: policy با schedule خالی، پورت خارج از محدوده، transport نامعتبر → 422 (BUG-15) · ❌
  - [ ] فیکس
- [ ] **I-A07** `/api/v1/status` وضعیت‌های `WAIT_*` رو هم بشمره · ❌
  - [ ] فیکس
- [ ] **I-A08** CRUD کامل همه منابع + 409 برای حذف منبعی که بهش ارجاع داده شده · ✅

---

## فاز ۸: چرخه عمر اپلیکیشن

- [ ] **I-L01** Graceful shutdown وسط ارسال: store بعد از تموم شدن workerها بسته بشه و خطای `database is closed` نیاد (BUG-29) · ❌ · 🔧
  - [ ] فیکس
- [ ] **I-L02** Start و stop کامل اپ با goleak، بدون نشت گوروتین · ❌ · 🔧
  - [ ] فیکس
- [ ] **I-L03** `make install` در کانتینر تمیز موفق باشه (Build) · ❌
  - [ ] فیکس
- [ ] **I-L04** کانفیگ نامعتبر (duration غلط، پورت منفی) باعث خطای startup بشه، نه اینکه بی‌صدا مقدار پیش‌فرض بگیره · ❌
  - [ ] فیکس

---

## پیوست: تست‌های واحد رگرسیون کدک (race یا integration نیستن ولی از گزارش درمیان)

- [ ] **U-01** آدرس الفبایی TPDU، رفت و برگشت برای طول‌های ۱ تا ۱۱ (BUG-11) · ❌
- [ ] **U-02** VP نسبی برای مرزهای 12h، 24h، 2d، 21d، 22d، 30d، 63w (BUG-20) · ❌
- [ ] **U-03** UDL بیشتر از ۲۵۵ و `short_message` با ۲۵۵ بایت → خطا (BUG-21) · ❌
- [ ] **U-04** UDH غیر concat (port یا national language) در خروجی SMPP حفظ بشه (BUG-22) · ❌
- [ ] **U-05** DCS فشرده رد بشه (BUG-23) · ❌
- [ ] **U-06** UCS2 با تعداد بایت فرد → خطا · ❌
- [ ] **U-07** UDH با IE ناقص → خطا · ❌
- [ ] **U-08** C-string بیشتر از حد مجاز SMPP → خطا · ❌
- [ ] **U-09** Fuzz: اجرای ۵ دقیقه‌ای همه fuzz targetها در nightly CI · ✅

---

## خلاصه پیشرفت

| فاز | تعداد | انجام‌شده |
|---|---|---|
| ۰ زیرساخت | 9 | 0 |
| ۱ Race | 13 | 0 |
| ۲ Store | 10 | 0 |
| ۳ Forwarder | 14 | 0 |
| ۴ SMPP | 12 | 0 |
| ۵ Diameter | 9 | 0 |
| ۶ SIP | 9 | 0 |
| ۷ API | 8 | 0 |
| ۸ Lifecycle | 4 | 0 |
| پیوست | 9 | 0 |
| **جمع** | **97** | **0** |

> هر بار که تسکی `[x]` شد، این جدول هم آپدیت بشه.
