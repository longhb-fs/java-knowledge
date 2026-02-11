# Day 12: Date/Time API (Xử Lý Ngày Giờ Hiện Đại)

## Mục tiêu hôm nay
- LocalDate, LocalTime, LocalDateTime - ngày giờ KHÔNG có múi giờ
- ZonedDateTime và ZoneId - ngày giờ CÓ múi giờ
- Instant - mốc thời gian tuyệt đối (timestamp)
- Period và Duration - khoảng cách thời gian
- DateTimeFormatter - định dạng và parse ngày giờ
- TemporalAdjusters - điều chỉnh ngày giờ thông minh
- Chuyển đổi từ API cũ (Date, Calendar) sang API mới

---

## 🤔 Tại sao cần học Date/Time API mới (java.time)?

### Vấn đề với API cũ (java.util.Date, Calendar)

```java
// ❌ API CŨ: Rối loạn, dễ sai, khó dùng
Date date = new Date(2024, 3, 15);      // Tháng bắt đầu từ 0! → thực ra là tháng 4
                                         // Năm tính từ 1900! → thực ra là năm 3924

Calendar cal = Calendar.getInstance();
cal.set(Calendar.MONTH, 3);              // Tháng 3 hay tháng 4? → tháng 4 (vì đếm từ 0)
date.setHours(14);                       // Mutable! Bất kỳ ai cũng sửa được → nguy hiểm

// ✅ API MỚI (Java 8+): Rõ ràng, immutable, an toàn
LocalDate date = LocalDate.of(2024, 3, 15);  // 15/03/2024 - rõ ràng!
LocalTime time = LocalTime.of(14, 30);       // 14:30 - không nhầm lẫn
```

```
┌──────────────────────────────────────────────────────────────┐
│              CŨ vs MỚI: SO SÁNH DATE/TIME API               │
├──────────────────────────┬───────────────────────────────────┤
│ java.util.Date (cũ)     │ java.time.* (mới - Java 8+)      │
├──────────────────────────┼───────────────────────────────────┤
│ Mutable (có thể sửa)    │ Immutable (không thể sửa)        │
│ Tháng đếm từ 0 (0-11)   │ Tháng đếm từ 1 (1-12)           │
│ Năm tính từ 1900         │ Năm đúng giá trị thật           │
│ Không thread-safe        │ Thread-safe                       │
│ Thiếu timezone support   │ Hỗ trợ timezone đầy đủ           │
│ 1 class làm mọi thứ     │ Tách rõ: Date, Time, DateTime    │
│ API khó đọc              │ API fluent, dễ đọc               │
├──────────────────────────┼───────────────────────────────────┤
│ ⚠️ KHÔNG dùng cho code  │ ✅ LUÔN dùng cho code mới        │
│ mới                      │                                   │
└──────────────────────────┴───────────────────────────────────┘
```

### Bản đồ các class chính trong java.time

```
┌─────────────────────────────────────────────────────────────────────┐
│                     java.time PACKAGE                               │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  KHÔNG có múi giờ (Local):                                          │
│  ┌─────────────┐  ┌─────────────┐  ┌──────────────────┐            │
│  │ LocalDate   │  │ LocalTime   │  │ LocalDateTime    │            │
│  │ 2024-03-15  │  │ 14:30:00    │  │ 2024-03-15T14:30 │            │
│  │ (chỉ ngày)  │  │ (chỉ giờ)   │  │ (ngày + giờ)     │            │
│  └─────────────┘  └─────────────┘  └──────────────────┘            │
│                                                                     │
│  CÓ múi giờ:                                                        │
│  ┌──────────────────────────────────────────────┐                   │
│  │ ZonedDateTime                                │                   │
│  │ 2024-03-15T14:30+07:00[Asia/Ho_Chi_Minh]    │                   │
│  │ (ngày + giờ + múi giờ)                       │                   │
│  └──────────────────────────────────────────────┘                   │
│                                                                     │
│  Mốc thời gian tuyệt đối:                                          │
│  ┌────────────────────────────┐                                     │
│  │ Instant                    │                                     │
│  │ 1710495000 (epoch seconds) │                                     │
│  │ (timestamp toàn cầu)       │                                     │
│  └────────────────────────────┘                                     │
│                                                                     │
│  Khoảng cách thời gian:                                             │
│  ┌────────────────────┐  ┌────────────────────┐                     │
│  │ Period             │  │ Duration           │                     │
│  │ 1 năm 2 tháng 3   │  │ 2 giờ 30 phút     │                     │
│  │ ngày (date-based)  │  │ (time-based)       │                     │
│  └────────────────────┘  └────────────────────┘                     │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘

💡 Khi nào dùng gì?
• Ngày sinh, ngày hết hạn     → LocalDate
• Giờ mở cửa, giờ đóng cửa   → LocalTime
• Thời gian tạo log           → LocalDateTime
• Lịch họp quốc tế            → ZonedDateTime
• Timestamp trong database    → Instant
• "Còn bao lâu" (ngày)       → Period
• "Còn bao lâu" (giờ/phút)   → Duration
```

---

## 1. LocalDate (Chỉ Ngày, Không Có Giờ)

> **Ví dụ đời thường**: Giống như **ngày trên tờ lịch** - chỉ có ngày/tháng/năm, không quan tâm mấy giờ.

```java
import java.time.LocalDate;
import java.time.DayOfWeek;
import java.time.Month;

// === Tạo LocalDate ===
LocalDate today = LocalDate.now();                     // Ngày hôm nay
LocalDate date = LocalDate.of(2024, 3, 15);           // 15/03/2024
LocalDate date2 = LocalDate.of(2024, Month.MARCH, 15);// Dùng enum Month rõ ràng hơn
LocalDate parsed = LocalDate.parse("2024-03-15");     // Parse từ String (format ISO)

// === Lấy thông tin ===
int year = today.getYear();               // 2024
int month = today.getMonthValue();        // 3 (tháng 3)  ← Đếm từ 1, KHÔNG phải 0!
Month monthEnum = today.getMonth();       // MARCH (enum)
int day = today.getDayOfMonth();          // 15
DayOfWeek dow = today.getDayOfWeek();     // SATURDAY (enum)
int dayOfYear = today.getDayOfYear();     // 75 (ngày thứ 75 trong năm)

// === Thao tác (Immutable - trả về object MỚI, KHÔNG sửa object cũ) ===
LocalDate tomorrow = today.plusDays(1);        // Ngày mai
LocalDate nextMonth = today.plusMonths(1);     // Tháng sau
LocalDate nextYear = today.plusYears(1);       // Năm sau
LocalDate lastWeek = today.minusWeeks(1);      // Tuần trước
LocalDate lastYear = today.minusYears(1);      // Năm trước

// ⚠️ Immutable nghĩa là:
LocalDate today = LocalDate.now();
today.plusDays(1);                             // ← KHÔNG thay đổi today!
LocalDate tomorrow = today.plusDays(1);        // ← Phải gán vào biến mới

// === So sánh ===
today.isBefore(tomorrow);                     // true: hôm nay trước ngày mai
today.isAfter(yesterday);                     // true: hôm nay sau hôm qua
today.isEqual(LocalDate.now());               // true (nếu cùng ngày)

// === Kiểm tra ===
today.isLeapYear();                           // Năm nhuận? (2024 → true)
int daysInMonth = today.lengthOfMonth();      // Số ngày trong tháng (28/29/30/31)
int daysInYear = today.lengthOfYear();        // Số ngày trong năm (365 hoặc 366)
```

---

## 2. LocalTime (Chỉ Giờ, Không Có Ngày)

> **Ví dụ đời thường**: Giống như **đồng hồ treo tường** - chỉ hiển thị giờ:phút:giây, không biết hôm nay ngày mấy.

```java
import java.time.LocalTime;

// === Tạo LocalTime ===
LocalTime now = LocalTime.now();                       // Giờ hiện tại
LocalTime time = LocalTime.of(14, 30);                // 14:30 (2:30 PM)
LocalTime time2 = LocalTime.of(14, 30, 45);           // 14:30:45
LocalTime parsed = LocalTime.parse("14:30:00");       // Parse từ String

// === Lấy thông tin ===
int hour = now.getHour();                  // 14
int minute = now.getMinute();              // 30
int second = now.getSecond();              // 45
int nano = now.getNano();                  // Nano giây

// === Thao tác ===
LocalTime later = now.plusHours(2).plusMinutes(30);    // Cộng 2 giờ 30 phút
LocalTime earlier = now.minusMinutes(15);              // Trừ 15 phút

// === Hằng số tiện ích ===
LocalTime.MIDNIGHT;  // 00:00:00 - nửa đêm
LocalTime.NOON;      // 12:00:00 - giữa trưa
LocalTime.MIN;       // 00:00:00 - giá trị nhỏ nhất
LocalTime.MAX;       // 23:59:59.999999999 - giá trị lớn nhất

// === Ví dụ thực tế: Kiểm tra giờ làm việc ===
LocalTime openTime = LocalTime.of(8, 0);
LocalTime closeTime = LocalTime.of(17, 30);
LocalTime current = LocalTime.now();

boolean isWorkingHour = !current.isBefore(openTime) && current.isBefore(closeTime);
// true nếu current >= 08:00 VÀ current < 17:30
```

---

## 3. LocalDateTime (Ngày + Giờ, Không Có Múi Giờ)

```java
import java.time.LocalDateTime;

// === Tạo LocalDateTime ===
LocalDateTime now = LocalDateTime.now();                                // Ngày giờ hiện tại
LocalDateTime dt = LocalDateTime.of(2024, 3, 15, 14, 30, 0);         // 15/03/2024 14:30:00
LocalDateTime dt2 = LocalDateTime.of(LocalDate.now(), LocalTime.NOON); // Ghép Date + Time

// === Chuyển đổi qua lại ===
LocalDate date = now.toLocalDate();        // Lấy phần ngày
LocalTime time = now.toLocalTime();        // Lấy phần giờ

// === Thao tác ===
LocalDateTime future = now.plusWeeks(2).plusHours(5);   // Cộng 2 tuần 5 giờ
LocalDateTime past = now.minusDays(3).minusMinutes(30); // Trừ 3 ngày 30 phút

// === So sánh ===
now.isBefore(future);  // true
now.isAfter(past);     // true

// 💡 Khi nào dùng LocalDateTime?
// → Khi múi giờ KHÔNG quan trọng (cùng 1 hệ thống, cùng 1 quốc gia)
// → Ví dụ: log timestamp, thời gian tạo record trong DB local
```

---

## 4. ZonedDateTime (Ngày Giờ Có Múi Giờ)

> **Ví dụ đời thường**: Khi bạn đặt **lịch họp online với đối tác Mỹ**, bạn nói "14:30 giờ Việt Nam" = "2:30 AM giờ New York". ZonedDateTime giữ thông tin múi giờ!

```java
import java.time.ZonedDateTime;
import java.time.ZoneId;

// === Giờ hiện tại với múi giờ ===
ZonedDateTime now = ZonedDateTime.now();
// 2024-03-15T14:30:00+07:00[Asia/Ho_Chi_Minh]

ZonedDateTime tokyo = ZonedDateTime.now(ZoneId.of("Asia/Tokyo"));
// 2024-03-15T16:30:00+09:00[Asia/Tokyo]  (Nhật Bản sớm hơn VN 2 giờ)

// === Xem tất cả múi giờ có sẵn ===
Set<String> zones = ZoneId.getAvailableZoneIds();
// Asia/Ho_Chi_Minh, America/New_York, Europe/London, Asia/Tokyo...

// === Chuyển đổi múi giờ (cùng 1 thời điểm, khác cách hiển thị) ===
ZonedDateTime vietnam = ZonedDateTime.now(ZoneId.of("Asia/Ho_Chi_Minh"));
// 14:30 giờ Việt Nam

ZonedDateTime newYork = vietnam.withZoneSameInstant(ZoneId.of("America/New_York"));
// 02:30 giờ New York (cùng 1 thời điểm, nhưng hiển thị khác)

// === Từ LocalDateTime → ZonedDateTime (thêm múi giờ) ===
LocalDateTime ldt = LocalDateTime.of(2024, 3, 15, 14, 30);
ZonedDateTime zdt = ldt.atZone(ZoneId.of("Asia/Ho_Chi_Minh"));
// Gắn múi giờ vào LocalDateTime
```

```
Chuyển đổi múi giờ:

  14:30 VN (GMT+7)     =     16:30 Tokyo (GMT+9)     =     02:30 New York (GMT-5)
  ┌──────────────┐           ┌──────────────┐              ┌──────────────┐
  │ 🇻🇳 Việt Nam │   ═══►   │ 🇯🇵 Nhật Bản │    ═══►    │ 🇺🇸 Mỹ       │
  │ +07:00       │           │ +09:00       │              │ -05:00       │
  └──────────────┘           └──────────────┘              └──────────────┘
       CÙNG MỘT THỜI ĐIỂM → chỉ khác cách hiển thị theo múi giờ

  withZoneSameInstant() = "Cùng thời điểm, đổi cách hiển thị"
```

---

## 5. Instant (Mốc Thời Gian Tuyệt Đối - Timestamp)

> **Ví dụ đời thường**: `Instant` giống như **số giây trên đồng hồ bấm giờ của vũ trụ** - đếm từ 01/01/1970 00:00:00 UTC (gọi là "Epoch"). Không quan tâm bạn ở đâu, `Instant` luôn giống nhau.

```java
import java.time.Instant;

// === Thời điểm hiện tại ===
Instant now = Instant.now();               // 2024-03-15T07:30:00Z (luôn UTC)
long epochSecond = now.getEpochSecond();   // Số giây từ 01/01/1970
long epochMilli = now.toEpochMilli();      // Số milli giây từ 01/01/1970

// === Tạo từ epoch ===
Instant fromSec = Instant.ofEpochSecond(1710495000);       // Từ số giây
Instant fromMilli = Instant.ofEpochMilli(1710495000000L);  // Từ milli giây

// === Chuyển đổi sang ZonedDateTime / LocalDateTime ===
ZonedDateTime zdt = now.atZone(ZoneId.of("Asia/Ho_Chi_Minh")); // Instant → ZonedDateTime
LocalDateTime ldt = LocalDateTime.ofInstant(now, ZoneId.systemDefault()); // Instant → LocalDateTime

// 💡 Khi nào dùng Instant?
// → Lưu timestamp vào database (độc lập múi giờ)
// → Đo thời gian thực thi code
// → So sánh thời điểm giữa các server ở nhiều nơi

// === Ví dụ: Đo thời gian thực thi ===
Instant start = Instant.now();
// ... code cần đo ...
Instant end = Instant.now();
Duration elapsed = Duration.between(start, end);
System.out.println("Thời gian chạy: " + elapsed.toMillis() + " ms");
```

---

## 6. Period và Duration (Khoảng Cách Thời Gian)

```
┌────────────────────────┬────────────────────────────┐
│ Period                 │ Duration                   │
│ (Khoảng cách NGÀY)    │ (Khoảng cách THỜI GIAN)   │
├────────────────────────┼────────────────────────────┤
│ Đơn vị: năm, tháng,   │ Đơn vị: giờ, phút, giây,  │
│ ngày                   │ nano giây                  │
│                        │                            │
│ Dùng với: LocalDate    │ Dùng với: LocalTime,       │
│                        │ LocalDateTime, Instant     │
│                        │                            │
│ Ví dụ: "2 năm 3 tháng"│ Ví dụ: "5 giờ 30 phút"    │
│ "Sinh nhật còn 45      │ "Chuyến bay kéo dài        │
│ ngày nữa"              │ 13 giờ 20 phút"            │
└────────────────────────┴────────────────────────────┘
```

### 6.1. Period - Khoảng cách ngày tháng năm

```java
import java.time.Period;

// === Tạo Period ===
Period period = Period.of(1, 2, 3);        // 1 năm, 2 tháng, 3 ngày
Period oneYear = Period.ofYears(1);        // 1 năm
Period twoMonths = Period.ofMonths(2);     // 2 tháng
Period tenDays = Period.ofDays(10);        // 10 ngày

// === Tính khoảng cách giữa 2 ngày ===
LocalDate birthday = LocalDate.of(1995, 6, 20);
LocalDate today = LocalDate.now();
Period age = Period.between(birthday, today);

System.out.println("Tuổi: " + age.getYears() + " năm, "
    + age.getMonths() + " tháng, "
    + age.getDays() + " ngày");
// Ví dụ: "Tuổi: 28 năm, 8 tháng, 25 ngày"

// === Cộng Period vào ngày ===
LocalDate future = today.plus(Period.ofMonths(6));   // 6 tháng sau
LocalDate past = today.minus(Period.ofYears(2));     // 2 năm trước
```

### 6.2. Duration - Khoảng cách giờ phút giây

```java
import java.time.Duration;

// === Tạo Duration ===
Duration twoHours = Duration.ofHours(2);           // 2 giờ
Duration thirtyMin = Duration.ofMinutes(30);       // 30 phút
Duration fiveSec = Duration.ofSeconds(5);          // 5 giây

// === Tính khoảng cách giữa 2 thời điểm ===
LocalTime start = LocalTime.of(9, 0);
LocalTime end = LocalTime.of(17, 30);
Duration workDay = Duration.between(start, end);

System.out.println("Giờ làm việc: " + workDay.toHours() + " giờ "
    + (workDay.toMinutes() % 60) + " phút");
// "Giờ làm việc: 8 giờ 30 phút"

// === Chuyển đổi đơn vị ===
workDay.toHours();          // 8
workDay.toMinutes();        // 510
workDay.toSeconds();        // 30600
workDay.toMillis();         // 30600000

// === Cộng Duration vào thời gian ===
LocalTime breakTime = start.plus(Duration.ofHours(4)); // 4 giờ sau khi bắt đầu → 13:00
```

---

## 7. DateTimeFormatter (Định Dạng Ngày Giờ)

> **Ví dụ đời thường**: Cùng 1 ngày "15 tháng 3 năm 2024" nhưng có thể viết:
> - `2024-03-15` (ISO standard)
> - `15/03/2024` (kiểu Việt Nam)
> - `March 15, 2024` (kiểu Mỹ)
> - `15 tháng 03 năm 2024` (tiếng Việt đầy đủ)

```java
import java.time.format.DateTimeFormatter;
import java.util.Locale;

LocalDateTime now = LocalDateTime.now();

// === Formatter có sẵn (Predefined) ===
String iso = now.format(DateTimeFormatter.ISO_LOCAL_DATE_TIME);
// "2024-03-15T14:30:00"

String isoDate = now.format(DateTimeFormatter.ISO_LOCAL_DATE);
// "2024-03-15"

// === Tùy chỉnh format ===
DateTimeFormatter vnFormat = DateTimeFormatter.ofPattern("dd/MM/yyyy HH:mm:ss");
String formatted = now.format(vnFormat);
// "15/03/2024 14:30:00"

DateTimeFormatter dateOnly = DateTimeFormatter.ofPattern("dd-MM-yyyy");
String dateStr = now.format(dateOnly);
// "15-03-2024"

// === Format tiếng Việt ===
DateTimeFormatter vnFull = DateTimeFormatter.ofPattern(
    "dd 'tháng' MM 'năm' yyyy, HH:mm",
    new Locale("vi", "VN")
);
String vnString = now.format(vnFull);
// "15 tháng 03 năm 2024, 14:30"

// === Parse: String → Object ===
LocalDateTime parsed = LocalDateTime.parse("15/03/2024 14:30:00", vnFormat);
LocalDate date = LocalDate.parse("2024-03-15");  // ISO format tự parse được
LocalDate date2 = LocalDate.parse("15-03-2024", dateOnly);
```

### Bảng các ký hiệu format phổ biến

| Ký hiệu | Ý nghĩa | Ví dụ |
|----------|---------|-------|
| `yyyy` | Năm 4 chữ số | 2024 |
| `yy` | Năm 2 chữ số | 24 |
| `MM` | Tháng (01-12) | 03 |
| `MMM` | Tháng viết tắt | Mar |
| `MMMM` | Tháng đầy đủ | March |
| `dd` | Ngày (01-31) | 15 |
| `HH` | Giờ 24h (00-23) | 14 |
| `hh` | Giờ 12h (01-12) | 02 |
| `mm` | Phút (00-59) | 30 |
| `ss` | Giây (00-59) | 45 |
| `a` | AM/PM | PM |
| `E` | Thứ viết tắt | Fri |
| `EEEE` | Thứ đầy đủ | Friday |
| `'text'` | Text cố định | 'tháng' → tháng |

```java
// Một số pattern phổ biến:
"dd/MM/yyyy"              // 15/03/2024
"dd/MM/yyyy HH:mm:ss"    // 15/03/2024 14:30:00
"yyyy-MM-dd"              // 2024-03-15 (ISO)
"dd MMM yyyy"             // 15 Mar 2024
"EEEE, dd MMMM yyyy"     // Friday, 15 March 2024
"hh:mm a"                 // 02:30 PM
```

---

## 8. TemporalAdjusters (Điều Chỉnh Ngày Thông Minh)

> **Ví dụ đời thường**: "Tìm ngày thứ Hai tuần tới", "Ngày cuối cùng của tháng này", "Ngày thứ Sáu trước đó"... Thay vì tự tính, dùng TemporalAdjusters!

```java
import java.time.temporal.TemporalAdjusters;
import java.time.DayOfWeek;

LocalDate today = LocalDate.now();  // Giả sử 15/03/2024 (Thứ Sáu)

// === Adjusters có sẵn ===
today.with(TemporalAdjusters.firstDayOfMonth());       // 01/03/2024 - ngày đầu tháng
today.with(TemporalAdjusters.lastDayOfMonth());        // 31/03/2024 - ngày cuối tháng
today.with(TemporalAdjusters.firstDayOfYear());        // 01/01/2024 - ngày đầu năm
today.with(TemporalAdjusters.lastDayOfYear());         // 31/12/2024 - ngày cuối năm

today.with(TemporalAdjusters.firstDayOfNextMonth());   // 01/04/2024 - ngày đầu tháng SAU
today.with(TemporalAdjusters.firstDayOfNextYear());    // 01/01/2025 - ngày đầu năm SAU

// Tìm thứ cụ thể
today.with(TemporalAdjusters.next(DayOfWeek.MONDAY));          // Thứ Hai tới (18/03)
today.with(TemporalAdjusters.previous(DayOfWeek.MONDAY));      // Thứ Hai trước (11/03)
today.with(TemporalAdjusters.nextOrSame(DayOfWeek.FRIDAY));    // Thứ Sáu tới HOẶC hôm nay (15/03)
today.with(TemporalAdjusters.previousOrSame(DayOfWeek.FRIDAY));// Thứ Sáu trước HOẶC hôm nay (15/03)

// === Custom Adjuster: Tìm ngày làm việc tiếp theo ===
TemporalAdjuster nextWorkingDay = temporal -> {
    DayOfWeek dow = DayOfWeek.from(temporal);
    int daysToAdd = switch (dow) {
        case FRIDAY -> 3;    // Thứ 6 → cộng 3 ngày → Thứ 2
        case SATURDAY -> 2;  // Thứ 7 → cộng 2 ngày → Thứ 2
        default -> 1;        // Các ngày khác → cộng 1 ngày
    };
    return temporal.plus(daysToAdd, ChronoUnit.DAYS);
};

LocalDate nextWork = today.with(nextWorkingDay);
// Nếu hôm nay là Thứ 6 → 18/03 (Thứ 2)
```

---

## 9. Chuyển Đổi Từ API Cũ (Legacy Conversion)

> Khi làm việc với code cũ hoặc thư viện bên thứ 3 dùng `java.util.Date`, bạn cần biết cách chuyển đổi.

```java
// === Date ↔ Instant ===
Date oldDate = new Date();                          // API cũ
Instant instant = oldDate.toInstant();              // Date → Instant
Date backToDate = Date.from(instant);               // Instant → Date

// === Calendar ↔ ZonedDateTime ===
Calendar calendar = Calendar.getInstance();
ZonedDateTime zdt = calendar.toInstant()
    .atZone(calendar.getTimeZone().toZoneId());     // Calendar → ZonedDateTime

// === java.sql.Timestamp ↔ LocalDateTime ===
java.sql.Timestamp ts = Timestamp.valueOf(LocalDateTime.now());  // LDT → Timestamp
LocalDateTime ldt = ts.toLocalDateTime();                         // Timestamp → LDT

// === java.sql.Date ↔ LocalDate ===
java.sql.Date sqlDate = java.sql.Date.valueOf(LocalDate.now());  // LD → sql.Date
LocalDate localDate = sqlDate.toLocalDate();                      // sql.Date → LD
```

```
Sơ đồ chuyển đổi:

  java.util.Date
       │
       ├── .toInstant() ──────► Instant
       │                           │
       │                    .atZone(ZoneId) ──► ZonedDateTime
       │                                            │
       │                                     .toLocalDateTime() ──► LocalDateTime
       │                                     .toLocalDate() ──────► LocalDate
       │                                     .toLocalTime() ──────► LocalTime
       │
       └── Date.from(instant) ◄── Instant

  💡 Lộ trình chuyển đổi: Date → Instant → ZonedDateTime → Local*
```

---

## 10. Ví Dụ Thực Tế

### Ví dụ 1: Tính tuổi chính xác

```java
public static String calculateAge(LocalDate birthDate) {
    Period age = Period.between(birthDate, LocalDate.now());
    return String.format("%d tuổi, %d tháng, %d ngày",
        age.getYears(), age.getMonths(), age.getDays());
}

// Sử dụng:
System.out.println(calculateAge(LocalDate.of(1995, 6, 20)));
// "28 tuổi, 8 tháng, 25 ngày"
```

### Ví dụ 2: Đếm ngày làm việc giữa 2 ngày

```java
public static long countWorkingDays(LocalDate start, LocalDate end) {
    return start.datesUntil(end)                    // Java 9+: Stream<LocalDate>
        .filter(date -> {
            DayOfWeek dow = date.getDayOfWeek();
            return dow != DayOfWeek.SATURDAY && dow != DayOfWeek.SUNDAY;
        })
        .count();
}

// Sử dụng:
long workDays = countWorkingDays(
    LocalDate.of(2024, 3, 1),
    LocalDate.of(2024, 3, 31)
);
System.out.println("Số ngày làm việc tháng 3: " + workDays);
```

### Ví dụ 3: Chuyển đổi giờ họp quốc tế

```java
public static void scheduleMeeting(LocalDateTime meetingTime, String fromZone, String... toZones) {
    ZonedDateTime meeting = meetingTime.atZone(ZoneId.of(fromZone));
    System.out.println("Lịch họp: " + meeting.format(
        DateTimeFormatter.ofPattern("dd/MM/yyyy HH:mm z")));

    for (String zone : toZones) {
        ZonedDateTime converted = meeting.withZoneSameInstant(ZoneId.of(zone));
        System.out.println("  " + zone + ": " + converted.format(
            DateTimeFormatter.ofPattern("dd/MM/yyyy HH:mm")));
    }
}

// Sử dụng:
scheduleMeeting(
    LocalDateTime.of(2024, 3, 15, 14, 30),
    "Asia/Ho_Chi_Minh",
    "America/New_York", "Europe/London", "Asia/Tokyo"
);
// Output:
// Lịch họp: 15/03/2024 14:30 ICT
//   America/New_York: 15/03/2024 03:30
//   Europe/London: 15/03/2024 07:30
//   Asia/Tokyo: 15/03/2024 16:30
```

---

## 11. Sai Lầm Thường Gặp

### ❌ Sai lầm 1: Quên rằng API mới là Immutable

```java
// ❌ SAI: Gọi method nhưng không gán kết quả
LocalDate date = LocalDate.of(2024, 3, 15);
date.plusDays(10);                         // Kết quả bị bỏ! date KHÔNG thay đổi!
System.out.println(date);                  // Vẫn là 2024-03-15

// ✅ ĐÚNG: Gán kết quả vào biến mới
LocalDate date = LocalDate.of(2024, 3, 15);
LocalDate newDate = date.plusDays(10);     // Gán vào biến mới
System.out.println(newDate);               // 2024-03-25
```

### ❌ Sai lầm 2: Nhầm lẫn giữa các class

```java
// ❌ SAI: Dùng LocalDateTime để lưu thời gian event quốc tế
LocalDateTime meeting = LocalDateTime.of(2024, 3, 15, 14, 30);
// → Không biết 14:30 theo múi giờ nào!

// ✅ ĐÚNG: Dùng ZonedDateTime cho sự kiện có múi giờ
ZonedDateTime meeting = ZonedDateTime.of(
    2024, 3, 15, 14, 30, 0, 0,
    ZoneId.of("Asia/Ho_Chi_Minh")
);

// 💡 Quy tắc chọn class:
// Chỉ cần ngày (sinh nhật, deadline)  → LocalDate
// Chỉ cần giờ (giờ mở cửa)           → LocalTime
// Ngày + giờ cùng hệ thống           → LocalDateTime
// Ngày + giờ khác múi giờ            → ZonedDateTime
// Timestamp lưu DB / so sánh         → Instant
```

### ❌ Sai lầm 3: Parse sai format

```java
// ❌ SAI: Parse format không khớp
LocalDate date = LocalDate.parse("15/03/2024");
// 💥 DateTimeParseException! Mặc định expect ISO format: yyyy-MM-dd

// ✅ ĐÚNG: Chỉ định formatter phù hợp
DateTimeFormatter formatter = DateTimeFormatter.ofPattern("dd/MM/yyyy");
LocalDate date = LocalDate.parse("15/03/2024", formatter);

// ❌ SAI: Nhầm MM (tháng) với mm (phút)
DateTimeFormatter wrong = DateTimeFormatter.ofPattern("dd/mm/yyyy"); // mm = phút!
// ✅ ĐÚNG:
DateTimeFormatter correct = DateTimeFormatter.ofPattern("dd/MM/yyyy"); // MM = tháng
```

---

## 12. Tóm Tắt Cuối Ngày

| Khái niệm | Giải thích | Ví dụ |
|------------|-----------|-------|
| **LocalDate** | Chỉ ngày (không giờ, không múi giờ) | `LocalDate.of(2024, 3, 15)` |
| **LocalTime** | Chỉ giờ (không ngày, không múi giờ) | `LocalTime.of(14, 30)` |
| **LocalDateTime** | Ngày + giờ (không múi giờ) | `LocalDateTime.now()` |
| **ZonedDateTime** | Ngày + giờ + múi giờ | `ZonedDateTime.now(ZoneId.of(...))` |
| **Instant** | Timestamp tuyệt đối (epoch) | `Instant.now()` |
| **Period** | Khoảng cách: năm, tháng, ngày | `Period.between(date1, date2)` |
| **Duration** | Khoảng cách: giờ, phút, giây | `Duration.between(time1, time2)` |
| **DateTimeFormatter** | Định dạng/parse ngày giờ | `.ofPattern("dd/MM/yyyy")` |
| **TemporalAdjusters** | Điều chỉnh ngày thông minh | `.with(lastDayOfMonth())` |
| **Immutable** | Tất cả class đều bất biến | `date.plusDays(1)` → object MỚI |

---

## 13. Câu Hỏi Phỏng Vấn Thường Gặp

### 🔥 Câu 1: Tại sao Java 8 giới thiệu java.time thay vì dùng java.util.Date?
**Trả lời:**
java.util.Date có nhiều vấn đề: mutable (không thread-safe), tháng đếm từ 0, năm tính từ 1900, thiếu timezone support, 1 class làm mọi thứ. java.time khắc phục tất cả: immutable, thread-safe, API rõ ràng, tách biệt Date/Time/DateTime/ZonedDateTime, hỗ trợ timezone đầy đủ. Lấy cảm hứng từ thư viện Joda-Time.

### 🔥 Câu 2: LocalDateTime khác ZonedDateTime thế nào? Khi nào dùng?
**Trả lời:**
- `LocalDateTime`: Ngày giờ KHÔNG có múi giờ. Dùng khi tất cả users cùng múi giờ, hoặc múi giờ không quan trọng (log local, thời gian tạo record)
- `ZonedDateTime`: Ngày giờ CÓ múi giờ. Dùng khi cần chuyển đổi giữa các múi giờ (ứng dụng quốc tế, scheduling, API trao đổi giữa các hệ thống)
- `Instant`: Timestamp tuyệt đối, dùng khi cần lưu/so sánh thời điểm chính xác toàn cầu

### 🔥 Câu 3: Period khác Duration thế nào?
**Trả lời:**
- `Period`: Đo khoảng cách theo **năm, tháng, ngày** (date-based). Dùng với `LocalDate`. Ví dụ: "2 năm 3 tháng" - không cần biết chính xác bao nhiêu giờ
- `Duration`: Đo khoảng cách theo **giờ, phút, giây, nano** (time-based). Dùng với `LocalTime`, `Instant`. Ví dụ: "5 giờ 30 phút" - đo chính xác đến nano giây

### 🔥 Câu 4: Tại sao các class trong java.time đều immutable?
**Trả lời:**
Immutable mang lại 3 lợi ích:
1. **Thread-safe**: Nhiều thread dùng chung mà không cần synchronize
2. **An toàn**: Không ai có thể vô tình sửa đổi giá trị (pass object vào method, method không thể thay đổi)
3. **Cacheable**: Có thể cache và tái sử dụng safely. Đây là bài học từ `java.util.Date` mutable gây ra rất nhiều bug khó tìm

### 🔥 Câu 5: Làm sao chuyển từ java.util.Date sang LocalDateTime?
**Trả lời:**
Không chuyển trực tiếp được. Phải đi qua Instant làm cầu nối:
```java
Date old = new Date();
Instant instant = old.toInstant();
LocalDateTime ldt = instant.atZone(ZoneId.systemDefault()).toLocalDateTime();
```
Lộ trình: `Date → Instant → ZonedDateTime → LocalDateTime`

### 🔥 Câu 6: DateTimeFormatter có thread-safe không?
**Trả lời:**
CÓ! `DateTimeFormatter` là **immutable và thread-safe**. Có thể khai báo static final và dùng chung giữa nhiều thread. Đây là cải tiến lớn so với `SimpleDateFormat` (cũ) vốn KHÔNG thread-safe, gây race condition trong multi-thread.

---

## Navigation

- [← Day 11: File I/O](./day-11-file-io.md)
- [Day 13: Multithreading Basics →](./day-13-multithreading-basics.md)
