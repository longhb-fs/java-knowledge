# Day 5: Stream API + File I/O + DateTime

> Gộp từ bản 19 ngày: Day 10 (Stream) + Day 11 (File I/O) + Day 12 (DateTime)
> 📖 Đọc sâu: [day-10](../java-fundamentals/day-10-stream-api.md) | [day-11](../java-fundamentals/day-11-file-io.md) | [day-12](../java-fundamentals/day-12-datetime-api.md)

---

## Phần A: Stream API

### 1. Stream là gì?

```
Nguồn dữ liệu → [filter] → [map] → [sorted] → [collect] → Kết quả
   (List)       trung gian   trung gian  trung gian   kết thúc    (List/Map/...)

💡 Stream = "dây chuyền xử lý" — KHÔNG lưu dữ liệu, dùng 1 lần rồi bỏ
💡 Lazy: Intermediate ops KHÔNG chạy cho đến khi gặp Terminal op
```

### 2. Tạo Stream

```java
list.stream()                           // Từ Collection
Arrays.stream(array)                    // Từ Array
Stream.of("a", "b", "c")               // Trực tiếp
IntStream.range(1, 10)                  // 1→9 (primitive stream)
IntStream.rangeClosed(1, 10)            // 1→10
Files.lines(Path.of("file.txt"))        // Từ file
```

### 3. Intermediate Operations (Trung gian) — Cheat Sheet

| Op | Tác dụng | Ví dụ |
|----|----------|-------|
| `filter(Predicate)` | Lọc | `.filter(n -> n > 5)` |
| `map(Function)` | Biến đổi 1→1 | `.map(String::toUpperCase)` |
| `flatMap(Function)` | Gỡ lồng | `.flatMap(List::stream)` |
| `sorted()` | Sắp xếp | `.sorted(Comparator.reverseOrder())` |
| `distinct()` | Loại trùng | `.distinct()` |
| `limit(n)` | Lấy n đầu | `.limit(5)` |
| `skip(n)` | Bỏ n đầu | `.skip(3)` |
| `peek(Consumer)` | Debug | `.peek(System.out::println)` |

### 4. Terminal Operations (Kết thúc) — Cheat Sheet

| Op | Tác dụng | Trả về |
|----|----------|--------|
| `collect(Collector)` | Thu thập kết quả | List, Set, Map |
| `toList()` (Java 16+) | Thu thập nhanh | Unmodifiable List |
| `forEach(Consumer)` | Xử lý từng phần tử | void |
| `count()` | Đếm | long |
| `reduce(identity, op)` | Gộp thành 1 giá trị | T |
| `findFirst()` | Phần tử đầu tiên | Optional |
| `anyMatch(Pred)` | Có ít nhất 1 thỏa? | boolean |
| `allMatch(Pred)` | Tất cả thỏa? | boolean |
| `min(Comp)` / `max(Comp)` | Min / Max | Optional |

### 5. Ví dụ nhanh

```java
List<String> names = List.of("John", "Jane", "Bob", "Alice", "Tom");

// Lọc + biến đổi + thu thập
List<String> result = names.stream()
    .filter(n -> n.length() > 3)      // Giữ tên > 3 ký tự
    .map(String::toUpperCase)         // Chuyển hoa
    .sorted()                         // Sắp xếp A-Z
    .toList();                        // [ALICE, JANE, JOHN]

// Đếm
long count = names.stream().filter(n -> n.startsWith("J")).count();  // 2

// Tìm
Optional<String> first = names.stream()
    .filter(n -> n.length() > 3)
    .findFirst();                     // Optional["John"]

// Reduce
int totalLength = names.stream()
    .mapToInt(String::length)
    .sum();                           // 19
```

### 6. Collectors nâng cao

```java
// Nối chuỗi
String joined = names.stream().collect(Collectors.joining(", "));
// "John, Jane, Bob, Alice, Tom"

// Nhóm theo tiêu chí
Map<Integer, List<String>> byLength = names.stream()
    .collect(Collectors.groupingBy(String::length));
// {3=[Bob, Tom], 4=[John, Jane], 5=[Alice]}

// Chia 2 nhóm true/false
Map<Boolean, List<String>> partition = names.stream()
    .collect(Collectors.partitioningBy(n -> n.length() > 3));

// Thu thập thành Map
Map<String, Integer> nameToLength = names.stream()
    .collect(Collectors.toMap(n -> n, String::length));
// {John=4, Jane=4, Bob=3, Alice=5, Tom=3}
```

### 7. flatMap — Gỡ lồng

```java
List<List<Integer>> nested = List.of(List.of(1,2), List.of(3,4), List.of(5,6));

List<Integer> flat = nested.stream()
    .flatMap(List::stream)        // [[1,2],[3,4],[5,6]] → [1,2,3,4,5,6]
    .toList();
```

---

## Phần B: File I/O

### 1. Đọc/Ghi file — java.nio.file (Cách hiện đại)

```java
import java.nio.file.*;

// === ĐỌC FILE ===

// Đọc toàn bộ thành String
String content = Files.readString(Path.of("data.txt"));

// Đọc thành List<String> (từng dòng)
List<String> lines = Files.readAllLines(Path.of("data.txt"));

// Đọc bằng Stream (tốt cho file lớn — đọc lazy từng dòng)
try (Stream<String> stream = Files.lines(Path.of("data.txt"))) {
    stream.filter(line -> !line.isBlank())
          .forEach(System.out::println);
}

// === GHI FILE ===

// Ghi String
Files.writeString(Path.of("output.txt"), "Hello World");

// Ghi List<String>
Files.write(Path.of("output.txt"), List.of("Line 1", "Line 2"));

// Ghi thêm (append)
Files.writeString(Path.of("log.txt"), "New entry\n", StandardOpenOption.APPEND);
```

### 2. Thao tác File/Directory

```java
// Kiểm tra
Files.exists(Path.of("data.txt"));
Files.isDirectory(Path.of("/tmp"));
Files.size(Path.of("data.txt"));       // Kích thước bytes

// Tạo / Xóa / Copy / Move
Files.createDirectories(Path.of("a/b/c"));
Files.delete(Path.of("temp.txt"));
Files.copy(source, target, StandardCopyOption.REPLACE_EXISTING);
Files.move(source, target);

// Duyệt thư mục
try (Stream<Path> paths = Files.list(Path.of("."))) {         // Chỉ level 1
    paths.forEach(System.out::println);
}
try (Stream<Path> paths = Files.walk(Path.of("."))) {          // Đệ quy
    paths.filter(p -> p.toString().endsWith(".java"))
         .forEach(System.out::println);
}
```

### 3. BufferedReader/Writer (File lớn)

```java
// Đọc file lớn từng dòng — không load toàn bộ vào RAM
try (BufferedReader br = Files.newBufferedReader(Path.of("big.csv"))) {
    String line;
    while ((line = br.readLine()) != null) {
        // Xử lý từng dòng
    }
}

// Ghi file lớn
try (BufferedWriter bw = Files.newBufferedWriter(Path.of("output.csv"))) {
    bw.write("Name,Age");
    bw.newLine();
    bw.write("An,25");
}
```

💡 **Quy tắc chọn:**
- File nhỏ → `Files.readString()` / `Files.readAllLines()`
- File lớn → `Files.lines()` (Stream) hoặc `BufferedReader`
- **Luôn** dùng try-with-resources để tự động đóng

---

## Phần C: DateTime API (java.time)

### 1. Các class chính

| Class | Chứa gì | Ví dụ |
|-------|---------|-------|
| `LocalDate` | Ngày (không giờ) | `2026-02-09` |
| `LocalTime` | Giờ (không ngày) | `14:30:00` |
| `LocalDateTime` | Ngày + Giờ (không timezone) | `2026-02-09T14:30:00` |
| `ZonedDateTime` | Ngày + Giờ + Timezone | `2026-02-09T14:30+07:00[Asia/Ho_Chi_Minh]` |
| `Instant` | Mốc thời gian UTC (epoch) | Dùng cho timestamp, logging |
| `Duration` | Khoảng cách giờ/phút/giây | `2 hours 30 minutes` |
| `Period` | Khoảng cách năm/tháng/ngày | `1 year 3 months` |

### 2. Tạo & Thao tác

```java
// Tạo
LocalDate today = LocalDate.now();
LocalDate birthday = LocalDate.of(2000, 1, 15);
LocalDate parsed = LocalDate.parse("2026-02-09");

LocalTime now = LocalTime.now();
LocalDateTime dateTime = LocalDateTime.of(2026, 2, 9, 14, 30);
ZonedDateTime zoned = ZonedDateTime.now(ZoneId.of("Asia/Ho_Chi_Minh"));

// Thao tác (IMMUTABLE — luôn trả về object MỚI)
LocalDate nextWeek = today.plusWeeks(1);
LocalDate lastMonth = today.minusMonths(1);
LocalDate firstDay = today.withDayOfMonth(1);

// So sánh
today.isBefore(birthday);
today.isAfter(birthday);
today.isEqual(LocalDate.of(2026, 2, 9));

// Khoảng cách
Period age = Period.between(birthday, today);    // 26 years, 0 months, 25 days
long daysBetween = ChronoUnit.DAYS.between(birthday, today);
```

### 3. Format & Parse

```java
DateTimeFormatter formatter = DateTimeFormatter.ofPattern("dd/MM/yyyy HH:mm");

// Format: DateTime → String
String text = dateTime.format(formatter);  // "09/02/2026 14:30"

// Parse: String → DateTime
LocalDateTime dt = LocalDateTime.parse("09/02/2026 14:30", formatter);
```

| Pattern | Ý nghĩa | Ví dụ |
|---------|---------|-------|
| `dd/MM/yyyy` | Ngày/Tháng/Năm | 09/02/2026 |
| `yyyy-MM-dd` | ISO format | 2026-02-09 |
| `HH:mm:ss` | Giờ:Phút:Giây (24h) | 14:30:00 |
| `hh:mm a` | Giờ:Phút AM/PM | 02:30 PM |
| `EEEE, dd MMMM yyyy` | Thứ, ngày tháng năm | Monday, 09 February 2026 |

### 4. Timezone Conversion

```java
ZonedDateTime vnTime = ZonedDateTime.now(ZoneId.of("Asia/Ho_Chi_Minh"));  // +07:00
ZonedDateTime jpTime = vnTime.withZoneSameInstant(ZoneId.of("Asia/Tokyo")); // +09:00
// VN 14:00 → JP 16:00 (chênh 2 tiếng)
```

💡 **Quy tắc chọn:**
- Chỉ cần ngày → `LocalDate`
- Ngày + giờ, KHÔNG quan tâm timezone → `LocalDateTime`
- Cần timezone (scheduling, international) → `ZonedDateTime`
- Timestamp / logging → `Instant`
- **KHÔNG dùng** `Date`, `Calendar` (legacy)

---

## Bài tập

1. **Stream**: Từ list nhân viên, tìm lương trung bình theo phòng ban (dùng `groupingBy` + `averagingDouble`)
2. **File**: Đọc file CSV, parse từng dòng, lọc theo điều kiện, ghi kết quả ra file mới
3. **DateTime**: Viết method tính số ngày làm việc giữa 2 ngày (bỏ thứ 7 chủ nhật)

---

## Navigation

- [← Day 4: Generics + Lambda](./day-4-generics-lambda.md)
- [Day 6: Threading →](./day-6-threading.md)
