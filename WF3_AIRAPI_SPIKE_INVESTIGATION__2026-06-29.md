# Truy nguyên spike gọi `/airquality_iqair.php` lúc 08:00 / 12:00 / 20:00 (giờ VN)

> Báo cáo điều tra READ-ONLY (chỉ phân tích, không sửa code). Bổ trợ cho `AIRAPI_AUDIT__com.wftab.weather.forecast__2026-06-29.md` (repo root) — tài liệu này tập trung trả lời **vì sao spike rơi đúng 3 mốc 08:00/12:00/20:00 giờ địa phương**.

## Context (vì sao có tài liệu này)

Log server cho thấy traffic gọi endpoint AQI `/airquality_iqair.php` (User-Agent `okhttp/5.0.0-alpha.4`, ~99% Android) vọt lên đúng 3 mốc giờ địa phương mỗi ngày: **20:00 VN (13:00 UTC, lớn nhất +31–40%)**, **12:00 VN (05:00 UTC)**, **08:00 VN (01:00 UTC)**. Đặc trưng: đánh thức đồng bộ cả dàn máy, ~38% máy gọi đúng 1 request rồi tắt, request rải đều trong ~60 giây đầu giờ (có jitter).

Nhiệm vụ: tìm ĐOẠN CODE và CƠ CHẾ kích hoạt. Đã điều tra READ-ONLY toàn repo (branch `phongnx/v1.71`, app `com.wftab.weather.forecast`, OkHttp `5.0.0-alpha.4` — khớp User-Agent server). Tài liệu này ghi lại kết luận + bằng chứng + khuyến nghị sửa.

---

## 1. KẾT LUẬN

**Nguyên nhân = (A) Lịch chạy ngầm trong app** — chính xác là **AlarmManager exact alarm** của **Thông báo hằng ngày (Daily)** và **Thông báo ngày mai (Tomorrow)**, đặt theo **giờ địa phương** với **GIÁ TRỊ MẶC ĐỊNH đúng 08:00 / 12:00 / 20:00**, và **bật sẵn mặc định**. Mỗi lần alarm nổ → worker fetch AQI (`iqair.php`) cho từng địa điểm đã lưu.

**Độ tin cậy: rất cao (~95%).** Ba giờ mặc định giải mã ra **đúng 3 mốc trong log**, cơ chế jitter giải thích việc "rải đều 60 giây", và đã **loại trừ** giả thuyết B (push) và C (widget).

| Giả thuyết | Phán quyết |
|---|---|
| (A) Lịch ngầm theo giờ địa phương | ✅ **XÁC NHẬN** — Daily + Tomorrow notification |
| (B) Push FCM tự gọi iqair | ❌ Loại — handler push chỉ hiển thị notification, không fetch AQI |
| (C) Widget / sync định kỳ | ❌ Loại — widget `updatePeriodMillis=86400000` (24h), refresh UI-only, không chạm network AQI |
| (D) Người dùng mở app thật | ❌ Không khớp pattern đồng bộ + 38% gọi-1-lần |

---

## 2. BẰNG CHỨNG — CHUỖI MẮT XÍCH ĐẦY ĐỦ

### Mắt xích 1 — Giá trị giờ MẶC ĐỊNH = đúng 08:00 / 12:00 / 20:00 VN  ⭐ (chứng cứ quyết định)

`app/.../data/local/preference/PreferencesHelper.kt`
- `getDailyNotificationMorningTime()` :691 → default `1691542800000L` = **08:00 VN** (01:00 UTC)
- `getDailyNotificationAfternoonTime()` :699 → default `1691557200000L` = **12:00 VN** (05:00 UTC)
- `getTomorrowNotificationTime()` :491 → default `1691586000000L` = **20:00 VN** (13:00 UTC)

Giải mã (xác minh bằng `date`): 3 epoch này là 01:00 / 05:00 / 13:00 UTC = 08:00 / 12:00 / 20:00 giờ VN — **khớp 100% với 3 mốc spike**.

### Mắt xích 2 — Bật MẶC ĐỊNH
`app/.../data/models/config/FuncStateConfig.kt`
- `isDailyNotifyEnable = true` :9 · `isTomorrowNotifyEnable = true` :11 → **bật sẵn**
- (đối chiếu) `isRealtimeNotifyEnable = false` :17 · `isBadWeatherWarningEnable = false` :15 → tắt sẵn ⇒ không phải nguồn spike chính
- Đọc qua `PreferencesHelper.isDailyNotificationEnable()` :462 / `isTomorrowNotificationEnable()` :476 (default = giá trị remote-config, gated bởi `mustShowWelcomeScreen()`).

### Mắt xích 3 — Lên lịch bằng AlarmManager EXACT, theo giờ ĐỊA PHƯƠNG + jitter
`app/.../utils/NotificationCenter.kt`
- `scheduleDailyNotification()` :300 → gọi `scheduleDailyAlarm()` cho Morning + Afternoon.
- `scheduleDailyAlarm()` :314–356 và `scheduleTomorrowAlarm()` :358–402: dựng `Calendar.getInstance()` (timezone máy), `set(HOUR_OF_DAY, hours)` / `MINUTE` / `SECOND` (:331–333, :375–377), rồi `setAlarmClock` / `setExactAndAllowWhileIdle` / `setExact` với `RTC_WAKEUP` (:345–351, :389+).
- **Jitter** = `java.util.Random().nextInt(59)` giây: `getDailyNotificationSettingTime()` :417, `getTomorrowNotificationSettingTime()` :527 → request rải 0–58 giây trong phút đầu. **Khớp** "rải đều cả 60 giây, không dồn giây :00".

### Mắt xích 4 — Alarm nổ → Worker → FETCH AQI (`iqair.php`)
- Receiver: `app/.../receivers/DailyTomorrowAlarmReceiver.kt` → `DailyTomorrowNotificationLauncher.startFromAlarm()` → enqueue worker.
- Worker: `app/.../helper/dailynotification/DailyTomorrowNotificationWorker.kt` → `handleNotification()` :287 → `getAqiData()` :352, gọi `AirQualityModules.getInstance().getAQIDetailByLocationId(...)` :380 (timeout `GET_AQI_TIMEOUT_MS = 45s`), lặp **×N địa điểm** đã lưu (forEach :91, tuần tự, time-budget 8').
- (Đường service song song khi không dùng worker) `services/DailyNotificationService.kt:142` & `services/TomorrowNotificationService.kt:44` cũng gọi `getAQIDetailByLocationId` — `DailyNotificationService` chạy N địa điểm **song song** (async :72) → burst.

### Mắt xích 5 — Nơi BUILD request `param` mã hóa (xác nhận đúng endpoint)
- Endpoint: `airquality/.../v2/network/AirVisualApiService.java:12` → `@GET("airquality_iqair.php")`, base `https://airapi.tohapp.com/` (`network/RemoteApiService.java:93`).
- Build param: `airquality/.../network/DataManager.java:125` `getNearestMonitorPointsV2()` → `param = "type=9&lat=..&lon=.."` → `EncodeHelper.encodeBase64()` → `mapParams.put("param", ...)`.
- Mã hóa: `airquality/.../network/helper/EncodeHelper.java:25` `encodeBase64()` = **GZIP** (`compress()` :50, `GZIPOutputStream`) → **Base64** → chèn salt ngẫu nhiên → URL-encode. **Khớp** manh mối "`param=<base64 + gzip>`". (Lưu ý: salt `SecureRandom` làm mỗi request khác nhau ⇒ server khó cache theo URL.)
- Call hierarchy lên: `DataManager` → `AQIHelperDataV2.getAQIDetailByLocationId()` (`v2/helper/AQIHelperDataV2.java:107`) → `AirQualityModules.getAQIDetailByLocationId()` (`AirQualityModules.java:154`) → các caller ở Mắt xích 4.

---

## 3. CẤU HÌNH CHÍNH XÁC

| Thuộc tính | Giá trị |
|---|---|
| Cơ chế | AlarmManager **exact** (`setAlarmClock`/`setExactAndAllowWhileIdle`/`setExact`, `RTC_WAKEUP`) → Receiver → WorkManager OneTimeWork → fetch AQI |
| Mốc giờ | Default **08:00 + 12:00 (Daily)** và **20:00 (Tomorrow)** |
| Theo giờ | **Địa phương** (đọc HOUR/MINUTE từ epoch qua `Calendar` timezone máy, re-anchor về "hôm nay HH:MM local") |
| Jitter | **Chỉ 0–58 giây** (`Random().nextInt(59)`) — đủ rải trong 1 phút, **KHÔNG** rải qua nhiều phút ⇒ vẫn là spike nhọn |
| Nhắm iqair.php | **Có** — `getAQIDetailByLocationId` → `iqair.php`, ×N địa điểm |
| Bật mặc định | Daily ✅ / Tomorrow ✅ ; Realtime/BadWeather ❌ |

---

## 4. VÌ SAO ĐỒNG BỘ CẢ DÀN MÁY & VÌ SAO ~38% GỌI 1 LẦN

- **Đồng bộ:** Giờ mặc định lưu dưới dạng **epoch tuyệt đối** (một thời điểm cố định). Khi dựng lại alarm, code đọc HOUR/MINUTE của epoch đó theo timezone máy rồi đặt "hôm nay HH:MM local". Hệ quả: **mọi máy giữ mặc định đều nổ tại cùng thời điểm UTC** (01:00/05:00/13:00 UTC). Vì app chủ yếu người dùng VN, đó chính là 08:00/12:00/20:00 VN. Đa số user không đổi giờ ⇒ hàng vạn máy nổ cùng lúc. Jitter chỉ 0–58 giây nên cả đàn vẫn dồn trong ~1 phút (đuôi 10–12 phút đến từ Doze/máy thức trễ, retry, và ×N địa điểm).
- **38% gọi đúng 1 lần:** Máy chỉ thức dậy vì alarm thông báo, có **1 địa điểm** đã lưu → fetch 1 `iqair.php` → hiển thị notification → ngủ lại. Máy có nhiều địa điểm hoặc bật nhiều loại thông báo ⇒ vài request (trung bình 2,7).
- **20:00 lớn nhất:** Tomorrow notification (dự báo ngày mai) bật mặc định và buổi tối máy thường thức/online ổn định hơn buổi sáng (08:00 nhiều máy còn Doze/tắt) ⇒ tỷ lệ alarm fire & fetch thành công cao hơn. (Tỷ lệ chính xác cần số runtime.)

---

## 5. KHUYẾN NGHỊ SỬA (đề xuất, chưa thực thi)

> Nguyên nhân là **lịch app** ⇒ hướng sửa chính là **rải tải (spread) phía client**. Khuyến nghị ưu tiên #1.

**#1 (khuyến nghị) — Mở rộng cửa sổ jitter cho alarm Daily/Tomorrow.**
Hiện jitter chỉ 0–58 **giây** (cùng phút). Thêm offset ngẫu nhiên **theo phút**, ổn định per-device (vd lưu 1 offset random ±15–30 phút trong SharedPreferences lần đầu, dùng lại mọi ngày để giờ thông báo không "nhảy" mỗi ngày). Sửa tại `NotificationCenter.getDailyNotificationSettingTime()` / `getTomorrowNotificationSettingTime()` (thêm vào `minutes`/`seconds` của `TimeModel`). Rủi ro thấp: thông báo tóm tắt hằng ngày trễ vài chục phút là chấp nhận được; spike tải dàn ra cả cửa sổ.

**#2 — Giảm số request mỗi lần fire.** Trong worker, nếu cache AQI còn hạn (`AQIHelperDataV2` TTL 30') thì **dùng cache, bỏ qua fetch**; cân nhắc bỏ nhánh fetch song song N địa điểm ở `DailyNotificationService` (đổi sang tuần tự + time-budget như worker).

**#3 — Tách push khỏi fetch tức thì:** (không áp dụng — đã xác nhận FCM **không** fetch AQI, nên không cần.)

**#4 — Phía server (ngoài repo này):** thêm cache cho `iqair.php` theo (lat,lon) làm tròn; do salt `SecureRandom` khiến URL luôn khác, cache phải dựa trên **payload đã giải mã** (sau `DecryptedPayloadInterceptor`), không theo query string.

**#5 — (Tham khảo) Khuếch đại bởi `owm.php`:** Xem `AIRAPI_AUDIT__com.wftab.weather.forecast__2026-06-29.md` ở repo root (M2/M3/M8): nhánh bổ sung `airquality_owm.php` đi vòng qua cache, không dedup, lặp khi data thiếu field — làm nặng thêm khi server chậm. Đáng sửa kèm.

---

## 6. PHẠM VI ĐÃ GREP / LOẠI TRỪ

- **Push (B):** `services/MyFirebaseMessagingService.kt` `onMessageReceived()` :167–209 — chỉ `showWeatherAlert`/`sendNotification`, **không** fetch AQI / enqueue work. Loại.
- **Widget (C):** `appwidgets/` + `res/xml/widget_*.xml` `updatePeriodMillis=86400000` (24h); `WidgetsControllerService` `ACTION_TIME_TICK` neo giây :00 nhưng chỉ vẽ lại UI (`WidgetUtils.refreshAllWidgets`), **không** chạm network AQI. Loại.
- **Anchor giờ tròn HH:00:** không có `setRepeating`/`setInexactRepeating`/`% 3600`/round-hour nào cho network (xem audit M7). Spike đến từ **mốc 8/12/20 giờ địa phương**, không phải mỗi giờ tròn.
- **Realtime / BadWeather worker:** tắt mặc định; có jitter 0–58s + chỉ chạy khung 6–9h (Realtime) ⇒ không tạo 3 spike sắc nét trên.

---

## 7. Verify (cách kiểm chứng end-to-end nếu cần)

1. **Trên thiết bị thật:** đọc SharedPreferences key `PREF_DAILY_NOTIFICATION_MORNING_TIME` / `..._AFTERNOON_TIME` / `PREF_TOMORROW_NOTIFICATION_TIME` → xác nhận nhiều máy = 3 epoch default ở trên.
2. **Logcat:** bật log tại `DailyTomorrowNotificationWorker.getAqiData()` và `DataManager.getNearestMonitorPointsV2()` (đã có `DebugLog.logd("Raw param...")`), chỉnh giờ máy tới 19:59 → quan sát alarm nổ ~20:00 và request `iqair.php` phát ra với jitter giây.
3. **Đối chiếu backend:** sau khi áp #1, theo dõi log server xem 3 spike có giãn ra thành cao nguyên không.
