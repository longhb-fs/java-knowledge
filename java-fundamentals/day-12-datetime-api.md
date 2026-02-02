# Day 12: Date/Time API

## Mục tiêu
- LocalDate, LocalTime, LocalDateTime
- ZonedDateTime và time zones
- Period và Duration
- Formatting và parsing

---

## 1. LocalDate

```java
import java.time.LocalDate;

// Tạo
LocalDate today = LocalDate.now();
LocalDate date = LocalDate.of(2024, 3, 15);
LocalDate parsed = LocalDate.parse("2024-03-15");

// Get
int year = today.getYear();
int month = today.getMonthValue();
int day = today.getDayOfMonth();
DayOfWeek dayOfWeek = today.getDayOfWeek();
int dayOfYear = today.getDayOfYear();

// Modify (immutable - returns new object)
LocalDate tomorrow = today.plusDays(1);
LocalDate nextMonth = today.plusMonths(1);
LocalDate lastYear = today.minusYears(1);

// Compare
today.isBefore(tomorrow);
today.isAfter(yesterday);
today.isEqual(LocalDate.now());

// Check
today.isLeapYear();
int daysInMonth = today.lengthOfMonth();
```

---

## 2. LocalTime

```java
import java.time.LocalTime;

LocalTime now = LocalTime.now();
LocalTime time = LocalTime.of(14, 30, 0);  // 14:30:00
LocalTime parsed = LocalTime.parse("14:30:00");

int hour = now.getHour();
int minute = now.getMinute();
int second = now.getSecond();

LocalTime later = now.plusHours(2).plusMinutes(30);
LocalTime earlier = now.minusMinutes(15);

// Constants
LocalTime.MIDNIGHT;  // 00:00
LocalTime.NOON;      // 12:00
LocalTime.MIN;       // 00:00
LocalTime.MAX;       // 23:59:59.999999999
```

---

## 3. LocalDateTime

```java
import java.time.LocalDateTime;

LocalDateTime now = LocalDateTime.now();
LocalDateTime dt = LocalDateTime.of(2024, 3, 15, 14, 30, 0);
LocalDateTime dt2 = LocalDateTime.of(LocalDate.now(), LocalTime.NOON);

// Convert
LocalDate date = now.toLocalDate();
LocalTime time = now.toLocalTime();

// Modify
LocalDateTime future = now.plusWeeks(2).plusHours(5);

// Compare
now.isBefore(future);
now.isAfter(past);
```

---

## 4. ZonedDateTime

```java
import java.time.ZonedDateTime;
import java.time.ZoneId;

// Current time with zone
ZonedDateTime now = ZonedDateTime.now();
ZonedDateTime tokyo = ZonedDateTime.now(ZoneId.of("Asia/Tokyo"));

// Available zones
Set<String> zones = ZoneId.getAvailableZoneIds();

// Convert between zones
ZonedDateTime newYork = tokyo.withZoneSameInstant(ZoneId.of("America/New_York"));

// From LocalDateTime
LocalDateTime ldt = LocalDateTime.now();
ZonedDateTime zdt = ldt.atZone(ZoneId.of("Asia/Ho_Chi_Minh"));
```

---

## 5. Instant

```java
import java.time.Instant;

// Timestamp (epoch seconds)
Instant now = Instant.now();
long epochSecond = now.getEpochSecond();
long epochMilli = now.toEpochMilli();

// Create from epoch
Instant fromEpoch = Instant.ofEpochSecond(1234567890);
Instant fromMilli = Instant.ofEpochMilli(System.currentTimeMillis());

// Convert
ZonedDateTime zdt = now.atZone(ZoneId.systemDefault());
LocalDateTime ldt = LocalDateTime.ofInstant(now, ZoneId.systemDefault());
```

---

## 6. Period và Duration

```java
import java.time.Period;
import java.time.Duration;

// Period (years, months, days)
Period period = Period.of(1, 2, 3);  // 1 year, 2 months, 3 days
Period between = Period.between(date1, date2);
period.getYears();
period.getMonths();
period.getDays();

LocalDate future = today.plus(period);

// Duration (hours, minutes, seconds, nanos)
Duration duration = Duration.ofHours(2);
Duration d = Duration.between(time1, time2);
duration.toMinutes();
duration.toSeconds();
duration.toMillis();

LocalTime later = now.plus(Duration.ofMinutes(30));
```

---

## 7. Formatting

```java
import java.time.format.DateTimeFormatter;

LocalDateTime now = LocalDateTime.now();

// Predefined formatters
String iso = now.format(DateTimeFormatter.ISO_LOCAL_DATE_TIME);

// Custom pattern
DateTimeFormatter formatter = DateTimeFormatter.ofPattern("dd/MM/yyyy HH:mm:ss");
String formatted = now.format(formatter);

// With locale
DateTimeFormatter vn = DateTimeFormatter.ofPattern("dd 'tháng' MM 'năm' yyyy",
    new Locale("vi", "VN"));

// Parse
LocalDateTime parsed = LocalDateTime.parse("15/03/2024 14:30:00", formatter);
LocalDate date = LocalDate.parse("2024-03-15");
```

---

## 8. TemporalAdjusters

```java
import java.time.temporal.TemporalAdjusters;

LocalDate today = LocalDate.now();

// Built-in adjusters
today.with(TemporalAdjusters.firstDayOfMonth());
today.with(TemporalAdjusters.lastDayOfMonth());
today.with(TemporalAdjusters.firstDayOfYear());
today.with(TemporalAdjusters.next(DayOfWeek.MONDAY));
today.with(TemporalAdjusters.previousOrSame(DayOfWeek.FRIDAY));

// Custom adjuster
TemporalAdjuster nextWorkingDay = temporal -> {
    DayOfWeek dow = DayOfWeek.from(temporal);
    int daysToAdd = switch (dow) {
        case FRIDAY -> 3;
        case SATURDAY -> 2;
        default -> 1;
    };
    return temporal.plus(daysToAdd, ChronoUnit.DAYS);
};

LocalDate nextWork = today.with(nextWorkingDay);
```

---

## 9. Legacy Conversion

```java
// Date ↔ Instant
Date date = new Date();
Instant instant = date.toInstant();
Date backToDate = Date.from(instant);

// Calendar ↔ ZonedDateTime
Calendar calendar = Calendar.getInstance();
ZonedDateTime zdt = calendar.toInstant().atZone(calendar.getTimeZone().toZoneId());

// Timestamp ↔ LocalDateTime
java.sql.Timestamp ts = Timestamp.valueOf(LocalDateTime.now());
LocalDateTime ldt = ts.toLocalDateTime();
```

---

## 10. Bài tập thực hành

### Bài 1: Age Calculator
Tính tuổi từ ngày sinh.

### Bài 2: Working Days
Đếm số ngày làm việc giữa 2 dates.

### Bài 3: Meeting Scheduler
Chuyển đổi meeting time giữa các time zones.

### Bài 4: Calendar Generator
Tạo calendar của tháng hiện tại.

---

## Đáp án tham khảo

<details>
<summary>Bài 1: Age Calculator</summary>

```java
public static int calculateAge(LocalDate birthDate) {
    return Period.between(birthDate, LocalDate.now()).getYears();
}

public static String getAgeDetails(LocalDate birthDate) {
    Period period = Period.between(birthDate, LocalDate.now());
    return String.format("%d years, %d months, %d days",
        period.getYears(), period.getMonths(), period.getDays());
}
```
</details>

---

## Navigation

- [← Day 11: File I/O](./day-11-file-io.md)
- [Day 13: Multithreading Basics →](./day-13-multithreading-basics.md)
