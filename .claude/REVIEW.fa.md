# بررسی عمیق کد vectorcore-smsc

> بازبینی سورس روی برنچ `main` (نسخه 0.4.0b). همه فایل‌های غیرتستی پکیج‌های `forwarder`, `routing`, `registry`, `dr`, `smpp`, `codec/*`, `diameter/*`, `sip/*`, `store/*`, `api`, `config` و `cmd/smsc` خونده شده. شماره خط‌ها تقریبی‌ان. این بررسی استاتیکه؛ تست race یا تست یکپارچه اجرا نشده.
>
> سطح‌بندی: 🔴 بحرانی · 🟠 بالا · 🟡 متوسط · ⚪ پایین

---

## خلاصه مدیریتی

کد تمیز و ماژولاره و پایه پروتکلی خوبی داره (FSM دیامتر، fuzz test برای کدک‌ها، claim اتمیک پیام‌ها). ولی **ماشین حالت پیام‌ها (state machine) و مرز اعتماد (trust boundary) دو نقطه ضعف اصلی‌ان**:

1. انتقال‌های وضعیت (DELIVERED / EXPIRED / FAILED) شرطی نیستن و خطاهاشون نادیده گرفته میشه، پس هم **از دست رفتن پیام** ممکنه هم **ارسال تکراری**.
2. هیچ‌کدوم از ورودی‌های مدیریتی و سیگنالینگ (REST API، Diameter، SIP REGISTER/NOTIFY) احراز هویت یا allow-list ندارن.
3. مسیر ورودی همگامه و کوئری‌های زنده HSS داخلش انجام میشه، پس زیر بار یا وقتی HSS کند میشه کل سیستم کند میشه.
4. برای استقرار چندنمونه‌ای (multi-instance) آماده نیست.

## ۱۰ کار اول (به ترتیب)

| # | کار | ارجاع |
|---|---|---|
| 1 | احراز هویت برای REST API + bind پیش‌فرض روی loopback | SEC-1 |
| 2 | allow-list برای پیرهای Diameter و S-CSCFهای SIP | SEC-2, SEC-3 |
| 3 | انتقال‌های وضعیت شرطی (compare-and-set) + بررسی خطا در همه‌شون | BUG-1..4, ARCH-1 |
| 4 | اگه persist شکست خورد، ارسال نکن | BUG-2 |
| 5 | درست کردن مقایسه زمان در SQLite (retry و expiry کار نمی‌کنه!) | BUG-5 |
| 6 | قبل از `ESME_ROK` اعتبارسنجی و decode کن | BUG-9 |
| 7 | سریال کردن همه writeهای Diameter در یک writer | BUG-12 |
| 8 | قفل گذاشتن روی mapهای hot-reload در main.go | BUG-7 |
| 9 | timeout برای bind در SMPP + سقف کوچیک‌تر command_length | SEC-5 |
| 10 | retention برای پیام‌های نهایی + سقف limit در API | PERF-5, PERF-6 |

---

## 🐞 باگ‌ها

### BUG-1 🔴 Expiry می‌تونه پیام تحویل‌شده رو EXPIRED کنه
- **فایل:** `internal/forwarder/expiry.go` → `sweep`؛ `store/*/messages.go` → `UpdateMessageStatus`
- اول `ListExpiredMessages` خونده میشه، بعد بدون شرط `UpdateMessageStatus(..., EXPIRED)`. در این فاصله retry ممکنه پیام رو claim و تحویل بده؛ نتیجه: پیام تحویل‌شده EXPIRED میشه و DR اشتباه (EXPIRED) هم فرستاده میشه.
- **راه حل:** `UPDATE ... SET status='EXPIRED' WHERE id=? AND status IN (...) AND expiry_at<=now` و بررسی `RowsAffected`.

### BUG-2 🟠 خطای persist نادیده گرفته میشه و پیام باز هم ارسال میشه
- **فایل:** `forwarder.go` → `Dispatch` / `persistMessage`
- `persistMessage` خطا برنمی‌گردونه؛ فقط لاگ می‌کنه. اگه DB قطع باشه پیامک ارسال میشه ولی هیچ رکوردی نداره (نه DR، نه retry، نه آمار). این با ادعای "persist before send" تناقض داره.
- **راه حل:** خطا برگردون و در صورت شکست ارسال نکن؛ به ingress خطا بده (مثلاً SMPP `ESME_RSYSERR`).

### BUG-3 🟠 خطای آپدیت وضعیت نهایی بعد از ارسال نادیده گرفته میشه
- **فایل:** `forwarder.go` → `finalizeRoutingPass`؛ `retry.go` → `dispatch`
- `_ = f.st.UpdateMessageStatus(ctx, msg.ID, DELIVERED)`. اگه این شکست بخوره، پیام در `DISPATCHED` می‌مونه و با ری‌استارت بعدی دوباره ارسال میشه. همچنین آپدیت route و status دو statement جدان؛ کرش بینشون هم همین اثر رو داره.
- **راه حل:** یک متد اتمیک `CompleteDelivery(id, expectedStatus, egress, peer)`؛ retry/outbox برای writeهای نهایی.

### BUG-4 🟠 ری‌استارت همه پیام‌های `DISPATCHED` رو دوباره صف می‌کنه
- **فایل:** `store/postgres/db.go` و `store/sqlite/db.go` → `Open`
- `UPDATE messages SET status='QUEUED' ... WHERE status='DISPATCHED'` بدون owner/lease. توی ری‌استارت پیام‌هایی که ممکنه تحویل شده باشن دوباره میرن. اگه دو نمونه روی یک DB باشن، نمونه دوم پیام‌های در حال ارسال نمونه اول رو دوباره می‌فرسته.
- نکته: بدون این reset، پیامی که بعد از persist و قبل از finalize کرش کنه برای همیشه در `DISPATCHED` گیر می‌کرد، چون `ListRetryableMessages` این وضعیت رو نمی‌بینه. پس reset لازمه ولی باید هوشمند باشه.
- **راه حل:** ستون‌های `claimed_by` و `lease_until`؛ فقط leaseهای منقضی رو recover کن.

### BUG-5 🟠 در SQLite، retry و expiry به‌موقع اجرا نمیشن
- **فایل:** `store/sqlite/messages.go` → `ListRetryableMessages`, `ListExpiredMessages`
- اپلیکیشن زمان رو به صورت `2026-09-28T12:00:00Z` (RFC3339) می‌نویسه ولی کوئری با `datetime('now')` مقایسه می‌کنه که `2026-09-28 12:00:00` هست. مقایسه TEXT لغوی انجام میشه و `T` بعد از space میاد؛ پس پیام‌هایی که **امروز** سررسیدشونه تا فردا انتخاب نمیشن.
- **راه حل:** `julianday(next_retry_at) <= julianday('now')` یا یک فرمت واحد. تست "due now" اضافه کن.

### BUG-6 🟠 قوانین مسیریابی غیرفعال هنوز اعمال میشن
- **فایل:** `internal/routing/engine.go` → `matches`؛ `loader.go` → `load`
- `matches` هیچ‌وقت `r.Enabled` رو چک نمی‌کنه و loader همه ردیف‌ها رو به engine میده. غیرفعال کردن rule از UI عملاً هیچ اثری نداره.
- **راه حل:** موقع load، ruleهای `!Enabled` و egressهای نامعتبر رو فیلتر کن.

### BUG-7 🟠 Race روی mapهای Diameter در main.go
- **فایل:** `cmd/smsc/main.go` → `syncHSSPeers`, `syncSGdPeers`، گوروتین metrics و callback `SetPeerStatusFunc`
- `activeHSSPeers`, `activeSGd` و بقیه mapهای معمولی‌ان. گوروتین hot-reload می‌نویسه و همزمان API و metrics روشون iterate می‌کنن، که می‌تونه به `concurrent map iteration and map write` و **panic کل پروسس** برسه.
- **راه حل:** `sync.RWMutex` یا snapshot تغییرناپذیر با `atomic.Value`.

### BUG-8 🟠 Registry بعد از reload، رکوردهای حذف‌شده رو نگه می‌داره
- **فایل:** `internal/registry/cache.go` → `loadAll`
- فقط `r.cache[msisdn] = ...` انجام میشه و map پاک یا جایگزین نمیشه. مشترکی که deregister شده تا زمان expiry قبلیش هنوز به S-CSCF قبلی route میشه.
- **راه حل:** map جدید بساز و اتمیک جایگزین کن.

### BUG-9 🟠 `submit_sm` قبل از اعتبارسنجی با `ESME_ROK` جواب داده میشه
- **فایل:** `internal/smpp/server/session.go` → `handleSubmit`
- کامنت خود کد: `// Always acknowledge the ESME`. اول ROK و message_id برمی‌گرده، بعد `DecodeSM` اجرا میشه. اگه decode شکست بخوره پیام بی‌صدا دور ریخته میشه ولی ESME فکر می‌کنه قبول شده.
- **راه حل:** اول decode، اعتبارسنجی و persist، بعد پاسخ؛ در صورت خطا کد مناسب (`ESME_RINVMSGLEN` و غیره).

### BUG-10 🟠 وضعیت bind در SMPP server اعمال نمیشه
- **فایل:** `smpp/server/session.go` → read loop
- bind از نوع receiver می‌تونه `submit_sm` بفرسته؛ bind دوم یا دستور ناشناخته `generic_nack` نمی‌گیره. `deliver_sm` از ESME (مثلاً DR) هندل نمیشه.
- **راه حل:** FSM صریح (Unbound, BoundTX, BoundRX, BoundTRX) و رد دستورات غیرمجاز.

### BUG-11 🟠 آدرس‌های الفبایی (Alphanumeric) در TPDU خراب دیکد میشن
- **فایل:** `codec/tpdu/decode.go` → `decodeAddress`؛ `encode.go` → `encodeAlphaAddress`
- برای TON=alphanumeric از فرمول BCD `(n+1)/2` استفاده میشه، در حالی که باید `ceil(n*7/8)` اکتت و n سپتت باشه. فرستنده‌هایی مثل `MyBank` قطع یا رد میشن.

### BUG-12 🟠 writeهای Diameter می‌تونن در هم تنیده بشن
- **فایل:** `internal/diameter/peer.go` → `writeLoop` در برابر `watchdogLoop`, `handleDWR`, `handleCER`, `handleDPR`
- پیام‌های اپلیکیشن از `writeCh` میرن ولی DWR/DWA/CEA/DPA مستقیم `writeFull(conn, ...)` می‌کنن. دو write همزمان می‌تونن بایت‌هاشون قاطی بشه و framing خراب بشه، که یعنی قطع اتصال با MME/HSS.
- **راه حل:** همه writeها از یک writer واحد.

### BUG-13 🟠 End-to-End ID دیامتر هر ۲۵۶ پیام تکرار میشه
- **فایل:** `diameter/codec/message.go` → `NextEndToEnd`
- `atomic.AddUint32(&c, 1) & 0xFF`؛ فقط ۸ بیت پایین تغییر می‌کنه. با RFC 6733 که یکتایی رو الزامی می‌کنه نمی‌خونه، و پاسخ‌های قدیمی/retransmit ممکنه اشتباه تطبیق داده بشن.

### BUG-14 🟠 SGd پیام MT رو ممکنه به MME اشتباه بفرسته
- **فایل:** `diameter/sgd/server.go` → `selectPeerForMMELocked`
- اگه peer دقیق MME پیدا نشه، **اولین peer فعال** برگردونده میشه. اگه اون peer یک DRA باشه این رفتار درسته، ولی اگه یک MME دیگه باشه پیام با Destination-Host اشتباه به MME نامربوط میره.
- **راه حل:** فقط peerهایی که صریحاً به عنوان proxy/DRA کانفیگ شدن fallback باشن.

### BUG-15 🟡 SF Policy با `retry_schedule` خالی باعث panic میشه
- **فایل:** `forwarder/retry.go` → `nextRetry`
- `idx = len(schedule)-1` که میشه `-1` و بعد `schedule[-1]`. API هم این رو validate نمی‌کنه.

### BUG-16 🟡 گوروتین خواندن SMPP client نشت می‌کنه
- **فایل:** `smpp/client/session.go` → `readLoop`
- `pduCh <- pdu` هیچ case لغوی نداره. اگه main loop برگرده یا کانال پر باشه گوروتین برای همیشه block می‌مونه و با هر reconnect یکی اضافه میشه.

### BUG-17 🟡 `throughput_limit` فقط در کانفیگ هست و هیچ‌جا اعمال نمیشه
- **فایل:** `smpp/client/manager.go`, `session.go`
- این مقدار فقط برای تشخیص تغییر کانفیگ مقایسه میشه و هیچ rate limiterی وجود نداره. ممکنه از سقف قراردادی SMSC مقصد بیشتر ارسال کنی و throttle بخوری.

### BUG-18 🟡 Sequence number در SMPP از محدوده مجاز خارج میشه
- **فایل:** `smpp/conn.go` → `NextSeq`
- محدوده مجاز `1..0x7FFFFFFF` هست ولی کد تا `0xFFFFFFFF` و بعد صفر میره. موقع wrap، `pending[seq]` قبلی بازنویسی میشه.

### BUG-19 🟡 پاسخ bind در کلاینت SMPP با sequence تطبیق داده نمیشه
- **فایل:** `smpp/client/session.go` → `connect`؛ همین مشکل برای `enquire_link_resp` هم هست.

### BUG-20 🟡 کدگذاری VP نسبی برای ۲۲ تا ۳۰ روز غلطه
- **فایل:** `codec/tpdu/encode.go` → `encodeVPRelative`
- `mins <= 30240` یعنی ۲۱ روز، نه ۳۰ روز. ۲۲ روز به صورت ۲۸ روز encode میشه.

### BUG-21 🟡 طول UD در TPDU سرریز می‌کنه
- **فایل:** `codec/tpdu/encode.go` → `append(out, byte(udl))` بدون بررسی سقف ۲۵۵. در SMPP هم سقف `short_message` باید ۲۵۴ باشه نه ۲۵۵.

### BUG-22 🟡 UDH غیر concat در خروجی SMPP حذف میشه
- **فایل:** `codec/smpp/encode.go` → `encodeSM`
- `msg.UDH.Raw` (مثلاً port addressing، EMS، national language shift) فقط وقتی `Concat != nil` باشه لحاظ میشه، و حتی اون موقع فقط concat بازسازی میشه.

### BUG-23 🟡 DCS فشرده (compressed) مثل متن عادی دیکد میشه
- **فایل:** `codec/tpdu/dcs.go` → `ParseDCS`. یا پیاده‌سازیش کن یا صریحاً رد کن.

### BUG-24 🟡 LISTEN/NOTIFY پستگرس با اولین خطای شبکه برای همیشه می‌میره
- **فایل:** `store/postgres/notify.go` → `loop` که در خطا `return` می‌کنه. بعد از failover دیتابیس، hot-reload بی‌صدا از کار میفته و سرویس هنوز سالم به نظر میاد.
- **راه حل:** reconnect با backoff، دوباره LISTEN، و reload کامل.

### BUG-25 🟡 polling در SQLite حذف‌ها و آپدیت‌های هم‌ثانیه رو نمی‌بینه
- **فایل:** `store/sqlite/notify.go` → `checkTable` بر اساس `MAX(updated_at)`. حذف یک rule یا peer هیچ‌وقت reload نمی‌کنه.

### BUG-26 🟡 schema پستگرس و SQLite یکی نیستن
- جدول `ims_registrations` در پستگرس ستون `imsi` نداره ولی در SQLite داره. IMSI در پستگرس گم میشه.

### BUG-27 🟡 migrationها نسخه‌بندی ندارن و خطاهاشون نادیده گرفته میشه
- **فایل:** `store/*/db.go` → `Open`
- `pool.Exec(...)` بدون بررسی خطا؛ فولدر `migrations/` اصلاً اجرا نمیشه. یک ALTER شکست‌خورده بی‌صدا رد میشه.

### BUG-28 🟡 آپدیت SIP peer پسورد رو پاک می‌کنه
- **فایل:** `api/handlers.go` → `update-sip-peer`
- اگه `auth_pass` در PUT نباشه، `NULLIF('', '')` پسورد رو NULL می‌کنه. برعکس مسیر SMPP که پسورد قبلی رو حفظ می‌کنه.

### BUG-29 🟡 Shutdown منتظر گوروتین‌ها نمی‌مونه
- **فایل:** `cmd/smsc/main.go` → `run`
- `_ = shutdownCtx; return nil` و بعد `defer st.Close()`. store ممکنه وسط کار retry یا SMPP بسته بشه. errgroup یا WaitGroup نیست.

### BUG-30 ⚪ موارد کوچیک‌تر
- `TPMR` موقع persist ذخیره نمیشه، پس بعد از retry گم میشه (`persistMessage`).
- `msg.Expiry` (مطلق) نادیده گرفته میشه؛ فقط `ValidityPeriod` خونده میشه.
- `Dispatch(nil)` قبل از nil check panic می‌کنه.
- مسیر alert در SGd (`tryRouteAt`) از `applyRoutePolicy` رد میشه، پس `vp_override` اعمال نمیشه.
- `req.Recipient` در هندلرهای SIP بدون nil check استفاده میشه (panic با درخواست خراب).
- `SetS6cReporter` و `sgd.Server.onMsg` بدون قفل خونده میشن (data race).
- `StopGraceful` برای peer ورودی (inbound) ممکنه برای همیشه block بشه، چون `doneCh` فقط در outbound بسته میشه.
- خطای `rand.Read` در `newUUID` چک نمیشه.
- `/api/v1/status` وضعیت‌های `WAIT_*` رو نمی‌شمره؛ `/health` همیشه ok میده حتی اگه DB مرده باشه.

---

## ⚡ پرفورمنس

### PERF-1 🟠 کوئری زنده Sh/S6c در مسیر ورودی
- `resolveCandidateAt` برای هر پیامی که مشترکش در کش محلی نیست `ShRefresh` زنده صدا می‌زنه، و برای SGd هم `GetSubscriber` و S6c. این در هر retry هم تکرار میشه. کند شدن HSS مستقیم ظرفیت ورودی رو می‌خوره و cascade failure درست می‌کنه.
- **راه حل:** کش مثبت و منفی با TTL + `singleflight` + deadline کوتاه.

### PERF-2 🟠 Dispatch همگام در مسیر ورودی
- SGd، SIP و SMPP client مستقیم `fwd.Dispatch` رو صدا می‌زنن که شامل persist، کوئری Diameter و ارسال شبکه‌ست. در SMPP client callback داخل read loop اجرا میشه، پس کند شدنش enquire_link و DR رو هم block می‌کنه و لینک قطع میشه.
- **راه حل:** "پذیرش و ذخیره" رو از "تلاش برای تحویل" جدا کن: صف durable + worker pool با محدودیت per-peer.

### PERF-3 🟠 Retry scheduler و expiry سریالی‌ان
- `for _, m := range msgs { r.dispatch(ctx, m) }`. یک ارسال کند (مثلاً OFA timeout پنج ثانیه‌ای) کل صف رو نگه می‌داره. با ۱۰۰۰ پیام منتظر برای MME قطع، هر tick چند دقیقه طول می‌کشه.
- **راه حل:** claim اتمیک و بعد worker pool محدود.

### PERF-4 🟡 گوروتین بی‌حد در SMPP server
- `dispatchMessage` برای هر پیام `go func(){...}` بدون سقف راه میندازه. یک ESME پرسرعت می‌تونه حافظه رو تموم کنه. backpressure وجود نداره.

### PERF-5 🟠 رشد بی‌پایان جدول‌ها
- هیچ purge یا retention برای پیام‌های `DELIVERED/FAILED/EXPIRED` و `delivery_reports` نیست. شمارش‌ها، ایندکس‌ها و بکاپ به مرور کند میشن.

### PERF-6 🟠 `limit` در API سقف نداره
- `?limit=10000000` مستقیم به SQL میره، اون هم با payload. بقیه منابع هم pagination ندارن. با API بدون auth این یک DoS ساده‌ست.

### PERF-7 🟡 ایندکس‌ها و کوئری‌های صف
- `ILIKE '%...%'` روی `src_msisdn` و `origin_peer` از ایندکس استفاده نمی‌کنه. ایندکس ترکیبی برای `ORDER BY COALESCE(next_retry_at, submitted_at)` نیست.

### PERF-8 🟡 کارهای تکراری در هر pass
- `fallbackDecisions()` در هر pass چند بار محاسبه میشه (`candidateCount`، `runRoutingPass`، `plannedRouteAt`).
- `GetSFPolicy` چند بار برای هر تلاش از DB خونده میشه، بدون کش.
- `GetSIPPeer` دو بار (یک بار resolve، یک بار deliver).

### PERF-9 🟡 SQLite
- `SetMaxOpenConns(1)` همه خوندن و نوشتن رو سریال می‌کنه؛ `busy_timeout` تنظیم نشده. تنظیمات pool پستگرس هم پیش‌فرضه.

### PERF-10 🟡 lock طولانی در SMPP manager
- `reload` در حالی که `m.mu` رو نگه داشته `stopGraceful` (تا ۵ ثانیه) صدا می‌زنه، که `SendViaPeer` رو block می‌کنه.

### PERF-11 ⚪ `command_length` تا ۱۶ مگابایت بدون احراز هویت allocate میشه (SMPP و Diameter). بیشتر به SEC-5 مربوطه.

---

## 🧹 Clean Code

### CC-1 🟡 نام‌گذاری OFR/TFR برعکس 3GPP هست
- طبق TS 29.338، **OFR = MO**-Forward و **TFR = MT**-Forward. در کد برعکسه: `SendOFR` پیام MT می‌فرسته، `handleMOForwardRequest` از `DecodeTFR` استفاده می‌کنه، و `HandleOFR` برای **MT request** ورودی صدا زده میشه. رفتار شبکه درسته چون command codeها درستن، ولی هر کسی کد رو بخونه گیج میشه. `HandleTFR` هم dead code هست.

### CC-2 🟡 کد مرده و مسیرهای موازی
- `tryRoutes` و `selectRoute` در forwarder استفاده نمیشن. `tryRouteAt` یک مسیر موازی با `attemptRoute` هست که همین الان رفتارشون فرق داره (BUG-30).
- `pendingMO`، `trackPendingMO` و `CompleteMO` در SGd: `trackPendingMO` هیچ‌جا صدا زده نمیشه، پس `CompleteMO` همیشه false برمی‌گردونه. OFA همیشه فوری با Success میره و SM-RP-UI (submit report) هیچ‌وقت به MME برنمی‌گرده.

### CC-3 🟡 طبقه‌بندی خطا با parse کردن متن
- `sipResponseCodeFromError` با `strings.Index(msg, " returned ")` و `Sscanf` کد SIP رو درمیاره. باید typed error باشه (`SIPResponseError{Code}`) و با `errors.As` چک بشه.
- در `registry`، `err.(*sh.ErrUnknownUser)` بدون `errors.As` هست، پس با wrap شدن خطا می‌شکنه.

### CC-4 🟡 `main.go` حدود ۳۰KB
- سیم‌کشی، منطق sync پیرهای Diameter، alert retry، cleanup رجیستری و metrics همه داخل closureهای `run()` هستن. باید به پکیج‌های `runtime/diameterpeers` و `app` با lifecycle صریح منتقل بشن.

### CC-5 🟡 عددهای جادویی
- `+3` (تعداد کاندیدهای built-in) در چند جا، `30*time.Second`، `15s`/`60s` تیک‌ها، `[30,300,...]`. باید const نام‌دار یا کانفیگ باشن.

### CC-6 🟡 تکرار بین پستگرس و SQLite
- حدود ۲۰ فایل تقریباً یکسان. می‌شه با `database/sql` + لایه dialect یا sqlc کمش کرد.

### CC-7 ⚪ موارد دیگه
- `routePolicy` خطای DB رو با "policy نداره" یکی می‌کنه.
- `RouteWithContext` context می‌گیره ولی نادیده‌ش می‌گیره.
- `unsafe.Pointer` در routing engine به جای `atomic.Pointer[T]`.
- `metrics.New` با `MustRegister` روی رجیستری global، پس نمونه دوم (مثلاً در تست) panic می‌کنه.
- پیام‌های لاگ `"sgd TFR received (MO)"` خودشون متناقضن.
- تشخیص نوع SIP MESSAGE با `ct[:19] == "application/vnd.3gp"` شکننده‌ست؛ باید media type parse بشه.
- `generateMsgID` در SMPP server و client کپی شده.

---

## 🏛 معماری

### ARCH-1 🔴 اینترفیس Store انتقال امن وضعیت رو پشتیبانی نمی‌کنه
- `UpdateMessageStatus`، `UpdateMessageRetry` و بقیه همه بدون شرطن؛ فقط `ClaimMessageForDispatch` شرط داره. ریشه BUG-1 تا BUG-4 همینه.
- **پیشنهاد:** متدهای `CompleteDelivery`، `Defer`، `ClaimExpired` و `Fail` با `expectedStatus` و `attempt_id`، و outbox برای DR.

### ARCH-2 🟠 Forwarder همه‌کاره است (God object)
- persist، routing، lookup HSS/S6c، policy، چهار پروتکل خروجی و DR همه در یک struct. تست‌پذیری و محدود کردن latency سخته.
- **پیشنهاد:** (۱) message state + outbox، (۲) route resolver، (۳) delivery workerها برای هر egress، (۴) DR outbox.

### ARCH-3 🟠 `store.Message` قرارداد lossless نیست
- `codec.Message` فیلدهایی مثل TON/NPI، alpha، TPMR، RPMR، SRR، Expiry و concat داره، ولی فقط بخشی ذخیره میشه. retry ممکنه پیامی متفاوت از ورودی encode کنه.
- **پیشنهاد:** یک envelope نسخه‌دار (مثلاً JSON یا protobuf) از کل پیام کانونیکال + تست round-trip.

### ARCH-4 🟠 DR همگام و غیر idempotent
- `Report` داخل finalize هم DB می‌نویسه، هم S6c RDSM و هم SMPP deliver_sm رو همگام صدا می‌زنه. کلید یکتا روی `(message_id, status)` نیست، پس DR تکراری ممکنه.

### ARCH-5 🟠 SIP 2xx به معنی تحویل پیامک گرفته میشه
- در ISC، پاسخ 200 به SIP MESSAGE فقط پذیرش تراکنشه؛ تحویل واقعی با RP-ACK/RP-ERROR جداگانه میاد (TS 24.341). همین‌طور در SIMPLE با IMDN. الان پیام با 2xx، DELIVERED میشه.

### ARCH-6 🟠 آمادگی نداشتن برای چند نمونه (HA)
- reset سراسری `DISPATCHED`، کش‌های in-memory بدون همگام‌سازی، SMPP message_id که شمارنده in-process هست و با ری‌استارت از صفر شروع میشه (DR ممکنه با پیام اشتباه تطبیق داده بشه)، و فقط اولین HSS بدون failover.

### ARCH-7 🟡 Hot-reload بدون coalescing
- هر event یک reload کامل می‌زنه با کانال‌های ۱۶ یا ۳۲ تایی. burst تغییرات باعث query‌های تکراری و ممکنه block شدن notifier بشه. debounce لازمه.

### ARCH-8 🟡 NOTIFY بدنه reginfo رو parse نمی‌کنه
- فقط `Subscription-State: terminated` دیده میشه؛ تغییرات contact و registration داخل XML نادیده گرفته میشن.

---

## 🔐 امنیت

### SEC-1 🔴 API مدیریتی بدون احراز هویت
- هیچ auth middleware در `api/server.go` نیست و پیش‌فرض روی `::` هست. CRUD روی همه چیز، حذف صف، `/metrics` و `/api/v1/docs` بازن. خطاهای خام DB هم به کلاینت برگردونده میشن (`dbErr`).

### SEC-2 🔴 Diameter ورودی بدون allow-list
- `handleConn` هر اتصالی رو می‌پذیره و `handleCER` همیشه Success میده. `Origin-Host` ادعایی مستقیم برای رجیستر peer استفاده میشه، پس یک مهاجم می‌تونه خودش رو MME جا بزنه، MO جعلی تزریق کنه یا Alert-SC بفرسته.

### SEC-3 🔴 SIP REGISTER/NOTIFY بدون احراز هویت
- هر کسی می‌تونه REGISTER بفرسته و کش IMS رو مسموم کنه، یعنی پیامک‌های یک مشترک رو به S-CSCF دلخواه بفرسته (رهگیری پیامک، مثلاً OTPها). NOTIFY هم با `P-Asserted-Identity` جعلی ثبت‌نام رو حذف می‌کنه. `P-Asserted-Identity` و `From` در MESSAGE هم بدون trust boundary قبول میشن (جعل فرستنده).
- **پیشنهاد:** allow-list آی‌پی S-CSCF، TLS یا IPsec، و رد کردن PAI از منابع غیرمطمئن.

### SEC-4 🟠 TLS خروجی SMPP به صورت پیش‌فرض ناامنه
- `verify_server_cert DEFAULT false` و `InsecureSkipVerify: !VerifyServerCert`. انتخاب `tls` بدون این فلگ یعنی MITM ممکنه.

### SEC-5 🟠 DoS در SMPP
- قبل از bind هیچ read deadline نیست (slowloris)، و `max_connections` قبل از auth پر میشه. هر اتصال بدون احراز هویت می‌تونه ۱۶MB allocate کنه. `readCStr` هم سقف طول نداره. bcrypt قبل از چک IP اجرا میشه، که یعنی CPU DoS، و `unknown system_id` وجود اکانت رو لو میده.

### SEC-6 🟠 پسوردهای خروجی به صورت plaintext در DB
- `smpp_clients.password` و `sip_peers.auth_pass` رمزنگاری نشدن (فقط از JSON مخفی شدن).

### SEC-7 🟠 کلیدهای خصوصی و باینری کامیت شده
- `test-certs/*.key` و `bin/smsc` (۳۳MB). `.gitignore` وجود نداره. کانفیگ نمونه به همین کلیدها اشاره می‌کنه.

### SEC-8 🟡 تزریق XML در IMDN
- `IMDNBody` مقدار `messageID` رو بدون escape داخل XML می‌ذاره؛ parser هم با `strings.Index` کار می‌کنه.

---

## 🛠 Build و عملیات

- `make install` فایل ناموجود `config.yaml` رو کپی می‌کنه.
- کانفیگ validation نداره؛ مقادیر نامعتبر بی‌صدا به پیش‌فرض برمی‌گردن (مثلاً reconnect به `10s`).
- readiness و liveness جدا نیستن.
- metrics برای عمق صف worker، latency کوئری HSS، و خطاهای reload وجود نداره.
- تست‌ها: تست واحد و fuzz خوبی برای کدک‌ها هست، ولی تست یکپارچه store، تست race (`go test -race`)، تست FSM برای SMPP bind، تست multi-instance و تست shutdown وجود نداره. پیشنهاد: `go test -race ./...` در CI.

---

## ✅ نقاط قوت (برای انصاف)

- claim پیام با `UPDATE ... WHERE status IN (...)` اتمیکه و `RowsAffected` چک میشه.
- FSM دیامتر (CER/CEA، DWR/DWA، DPR/DPA) پیاده‌سازی شده؛ pending mapهای S6c، Sh و SGd با `defer` پاک میشن و نشت ندارن.
- decode پروتکل‌ها bounds check داره و fuzz target برای TPDU، AVP، SMPP و SIP وجود داره.
- allow-list آی‌پی در SMPP از `RemoteAddr` واقعی استفاده می‌کنه، نه هدر جعلی.
- پسورد SMPP ورودی با bcrypt ذخیره میشه؛ SQL پارامتری هست (SQL injection پیدا نشد).
- routing engine و جدول mapping به صورت immutable منتشر میشن که پایه خوبی برای خوندن همزمانه.
- مستندات `docs/ROUTING.md` دقیقاً با کد می‌خونه.
