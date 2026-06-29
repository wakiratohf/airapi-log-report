# Điều tra: Spike gọi `airquality_iqair.php` lúc 20:00 / 12:00 / 08:00 (giờ VN)

> Báo cáo truy nguyên trong MÃ NGUỒN CLIENT (Android). Mọi kết luận có `file:line` thật trong repo.

| Trường | Giá trị |
|---|---|
| applicationId | `com.droidteam.weather` — `app/build.gradle.kts:21` |
| Branch / Commit | `develop_v1.41` / `6e831f9` |
| okhttp | `5.0.0-alpha.4` — `gradle/libs.versions.toml:42` (khớp User-Agent `okhttp/5.0.0-alpha.4`) |
| Endpoint | `airquality_iqair.php` (type=9) — `AirVisualApiService.java:12` |
| Ngày điều tra | 2026-06-29 |

---

## 1. KẾT LUẬN

**Nguyên nhân là (A) — Lịch chạy ngầm trong app**, độ tin cậy **rất cao (~95%)**.

Cụ thể: **thông báo định kỳ "Daily" + "Tomorrow"** dùng **`AlarmManager` exact alarm** theo **giờ địa phương**, với **giá trị giờ MẶC ĐỊNH chính là 08:00 / 12:00 / 20:00**, và **cả hai loại đều BẬT MẶC ĐỊNH**. Khi alarm nổ, service fetch dữ liệu chất lượng không khí → gọi đúng `airquality_iqair.php`.

Loại trừ:
- **(B) Push FCM** — handler push **chỉ hiển thị thông báo**, KHÔNG fetch AQI (`MyFirebaseMessagingService.kt:138-180`). → loại.
- **(C) Widget / SyncAdapter** — widget chỉ fetch *thời tiết* (không AQI) và `updatePeriodMillis = 24h`; không có SyncAdapter. → loại.
- **(D) User mở app** — có đóng góp nền nhưng KHÔNG thể tạo mốc đồng bộ sắc nét theo giờ.

> ⚠️ **Mâu thuẫn với `AIRAPI_AUDIT__com.droidteam.weather__2026-06-29.md`** (kết luận "spike :00 do server"). Lý do: audit đó trả lời cho triệu chứng "*spike mỗi giờ tròn HH:00*" và **đã bỏ sót việc giải mã GIÁ TRỊ MẶC ĐỊNH** của giờ thông báo. Triệu chứng thật chỉ là **3 mốc/ngày (08/12/20)** — trùng khít với 3 default time hard-code trong app. Đây chính là "client-side anchor" mà audit kết luận "NOT PRESENT".

---

## 2. BẰNG CHỨNG — TỪNG MẮT XÍCH

### Mắt xích 1 — Nơi đặt lịch (ra quyết định THỜI ĐIỂM)
`app/src/main/java/com/droidteam/weather/utils/NotificationCenter.kt`
- `scheduleDailyAlarm()` (dòng **304–346**) và `scheduleTomorrowAlarm()` (dòng **348–392**): đặt **exact alarm** bằng `setAlarmClock(...)` (API≥24) / `setExactAndAllowWhileIdle` (API23) — dòng **336 / 382**.
- Giờ alarm lấy theo **giờ địa phương máy**: `calendar.set(Calendar.HOUR_OF_DAY, timeModel.hours)` (dòng **321 / 365**).
- **Jitter giây**: `seconds = Random().nextInt(59)` (dòng **407** Daily, dòng **487** Tomorrow).

### Mắt xích 2 — Giờ MẶC ĐỊNH (mấu chốt)
`app/src/main/java/com/droidteam/weather/data/local/preference/PreferencesHelper.kt`

| Loại | Hàm / dòng | Default (epoch ms) | Giải mã |
|---|---|---|---|
| Daily **sáng** | `getDailyNotificationMorningTime()` :**643** | `1691542800000` | **08:00 VN** = 01:00 UTC |
| Daily **chiều** | `getDailyNotificationAfternoonTime()` :**647** | `1691557200000` | **12:00 VN** = 05:00 UTC |
| **Tomorrow** | `getTomorrowNotificationTime()` :**431** | `1691586000000` | **20:00 VN** = 13:00 UTC |

Và **bật mặc định**:
- `isDailyNotificationEnable()` :**408** → mặc định `true` *(comment: "From v13.6, default ON like WF3")*
- `isTomorrowNotificationEnable()` :**419** → mặc định `true`

→ Khớp **chính xác** với 3 mốc server: **13:00 UTC (lớn nhất) / 05:00 / 01:00 UTC**.

### Mắt xích 3 — Alarm được "lên dây cót" ở đâu
`app/src/main/java/com/droidteam/weather/BaseApplication.kt` `checkAndStartBackgroundFunc()` (dòng **47–48**): mỗi lần app khởi động gọi `checkEnableDailyNotification` + `checkEnableTomorrowNotification`. Tái lập lịch cả khi **boot máy** (`BootUpReceiver`), **đổi giờ hệ thống** (`TimeChangedReceiver`), **cấp quyền exact alarm** (`ExactAlarmPermissionReceiver`). Service tự re-arm cho ngày kế (`DailyNotificationService.kt:218`, `TomorrowNotificationService.kt:98`).

### Mắt xích 4 — Service → fetch AQI
- `app/src/main/java/com/droidteam/weather/services/DailyNotificationService.kt:153` → `AirQualityModules.getInstance().getAQIDetailByLocationId(...)`
- `app/src/main/java/com/droidteam/weather/services/TomorrowNotificationService.kt:84` → `getAQIDetailByLocationId(...)`

### Mắt xích 5 — Dựng `param` mã hóa → đúng endpoint (khớp manh mối log)
`airquality/src/main/java/com/weather/airquality/network/DataManager.java`
- `getNearestMonitorPointsV2()` :**73–82** → `param.append("type=9")` (**type=9 = iqair**) :**76** → `EncodeHelper.encodeBase64(param)` :**80**.
- `EncodeHelper.encodeBase64()` :**25–31** → `compress()` dùng **`GZIPOutputStream`** :**54** → `Base64.encodeToString` → thêm salt ngẫu nhiên → `URLEncoder.encode`. ⇒ đúng dạng **`param=<base64 + gzip>`** trong log.
- `airquality/src/main/java/com/weather/airquality/v2/network/AirVisualApiService.java:12` → `@GET("airquality_iqair.php")`.

(Đường `type=1` → `getMorePollutantsAQI` → `airquality_owm.php` :**84–90** là bộ khuếch đại, không phải nguồn mốc giờ.)

**Chuỗi đầy đủ:**
```
App start / boot / đổi giờ
  → BaseApplication.checkAndStartBackgroundFunc()
  → NotificationCenter.checkEnable{Daily,Tomorrow}Notification()
  → scheduleDailyAlarm / scheduleTomorrowAlarm  (setAlarmClock, EXACT, giờ địa phương + giây random)
        08:00 / 12:00 / 20:00 local  (default, default-ON)
  → [alarm nổ] → Daily/TomorrowNotificationService
  → AirQualityModules.getAQIDetailByLocationId() ×N địa điểm
  → DataManager.getNearestMonitorPointsV2()  (type=9, gzip+base64 → param=…)
  → GET airquality_iqair.php
```

---

## 3. CẤU HÌNH CHÍNH XÁC

| Thuộc tính | Giá trị |
|---|---|
| Cơ chế | **`AlarmManager` exact** (`setAlarmClock` / `setExactAndAllowWhileIdle`), chu kỳ 24h, tự re-arm |
| Mốc giờ | **08:00, 12:00, 20:00** — *giá trị mặc định, đa số user không đổi* |
| Theo giờ nào | **Giờ địa phương** (`Calendar.HOUR_OF_DAY`). Default lưu dạng epoch tuyệt đối neo UTC+7 |
| Nhắm iqair? | **Có** — `type=9` → `airquality_iqair.php`, ×N địa điểm đã lưu |
| Jitter | **Chỉ ở GIÂY**: `Random().nextInt(59)` → rải trong 0–58s của phút :00. **KHÔNG có jitter ở phút** → phút vẫn trùng khít |
| Flex/window | **Không** (exact alarm, không `setWindow`/`flex`) |
| Backoff/Retry-After | **Không**; `retryOnConnectionFailure` mặc định `true` → tự retry khi server nghẽn (`RemoteApiService.java:75-83`) |

---

## 4. VÌ SAO ĐỒNG BỘ HÀNG VẠN MÁY & VÌ SAO ~38% CHỈ GỌI 1 LẦN

- **Đồng bộ:** Default time hard-code = 08/12/20 giờ VN + **bật mặc định** → mọi máy giữ mặc định nổ alarm đúng cùng giờ địa phương. Vì user base tập trung ở VN (UTC+7, không DST) → cùng dồn về **01:00 / 05:00 / 13:00 UTC**.
  - *Củng cố thêm:* default lưu dạng **epoch tuyệt đối**; code trích giờ ở TZ máy rồi đặt lại ở chính TZ đó → với vùng không-DST, alarm rơi đúng **cùng một mốc UTC** bất kể máy ở múi giờ nào → mốc UTC càng sắc.
- **Rải đều 60 giây:** đúng do `Random().nextInt(59)` ở phần giây — phút thì exact, giây thì ngẫu nhiên.
- **20:00 là đỉnh lớn nhất:** Tomorrow (20:00) là feature riêng, buổi tối máy **mở/online/sạc nhiều nhất**; còn 08:00 nhiều máy còn ngủ/Doze hoặc đang di chuyển → tỉ lệ gọi mạng thành công thấp hơn *(suy luận, hợp triệu chứng)*.
- **~38% gọi đúng 1 request rồi dừng:** máy chỉ có **1 địa điểm**; alarm nổ → fetch iqair 1 lần → service xong → máy ngủ lại. (TB **2,7 req/máy** = vài địa điểm, hoặc iqair + owm bổ sung.)
- **~62% cả ngày chỉ xuất hiện đúng giờ đó:** user **không mở app**; thứ duy nhất đánh thức & gọi mạng là **alarm thông báo** → log server chỉ thấy máy đó đúng 1 lần/ngày tại mốc giờ.
- **Đuôi 10–12 phút:** retry (`retryOnConnectionFailure`), đường `owm` nối tiếp, vòng lặp N địa điểm, máy offline kết nối lại.

---

## 5. KHUYẾN NGHỊ SỬA (theo nguyên nhân A)

**Phía client (gốc rễ — phá đồng bộ):**
1. **Thêm jitter ở PHÚT, không chỉ giây.** Hiện `Random().nextInt(59)` chỉ ở giây ⇒ trải ±0–30 phút quanh mốc (`NotificationCenter.scheduleDailyAlarm/scheduleTomorrowAlarm`).
2. **Đổi default time lệch nhau theo thiết bị** (vd hash theo `ANDROID_ID`/installId ra phút offset cố định cho mỗi máy) — giữ "khoảng sáng/trưa/tối" nhưng không dồn đúng :00.
3. **Cân nhắc dùng inexact** cho thông báo không cần chính xác tuyệt đối: `setWindow(...)` thay `setAlarmClock` để OS gom-rải.

**Giảm khuếch đại (các điểm audit đã chỉ ra):**
4. Gate đường `airquality_owm.php` (`type=1`) bằng TTL + dedup + negative-cache (`AQIHelperDataV2.java:233,270`).
5. Đặt `retryOnConnectionFailure(false)` hoặc thêm backoff/tôn trọng `Retry-After` (`RemoteApiService.java:75–83`).

**Phía server (giảm áp tức thời):**
6. **Cache `iqair.php` theo (lat,lon, làm tròn thời gian)** + trả `Cache-Control`/`ETag` để hấp thụ burst đầu phút :00.

---

## 6. PHẠM VI ĐÃ GREP & GHI CHÚ

Đã grep toàn repo (trừ `/build/`) các module `app, airquality, weather-sdk, utilities, TOH-Ad`: `PeriodicWorkRequest/OneTimeWorkRequest/setInitialDelay`, `AlarmManager/setExact*/setAlarmClock/setWindow`, `JobScheduler`, `Handler/postDelayed/Timer/ScheduledExecutorService`, `HOUR_OF_DAY/LocalTime/ZoneId/TimeZone`, `FirebaseMessagingService/onMessageReceived`, `AppWidgetProvider/updatePeriodMillis`, `SyncAdapter`, `GZIPOutputStream/Base64`, `airquality_iqair/param/type=9`.

Các scheduler khác (Realtime 15', BadWeather 30', FcmInfo 1h) dùng **interval tương đối + jitter, tự re-arm → trôi pha**, KHÔNG neo giờ tròn → là nhiễu nền, không tạo mốc sắc. **Mốc 08/12/20 chỉ Daily/Tomorrow alarm tạo ra.**
