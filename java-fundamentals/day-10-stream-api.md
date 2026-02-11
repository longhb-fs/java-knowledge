# Day 10: Stream API (Xử Lý Dữ Liệu Kiểu Dây Chuyền)

## Mục tiêu hôm nay
- Hiểu Stream (luồng dữ liệu) là gì và tại sao cần dùng
- Intermediate Operations (thao tác trung gian) - filter, map, flatMap, sorted...
- Terminal Operations (thao tác kết thúc) - collect, reduce, forEach...
- Collectors (bộ thu thập) - groupingBy, partitioningBy, toMap...
- Parallel Streams (luồng song song) và khi nào nên/không nên dùng

---

## 🤔 Tại sao cần học Stream API?

### Ví dụ đời thường
> Hãy tưởng tượng bạn có **1000 hồ sơ xin việc** và cần:
> 1. Lọc ra những người có kinh nghiệm > 3 năm
> 2. Sắp xếp theo điểm phỏng vấn giảm dần
> 3. Lấy top 10 người
>
> **Cách thủ công**: Bạn phải đọc từng hồ sơ, tạo danh sách tạm, sort, rồi chọn...
> **Cách dây chuyền (Stream)**: Bạn thiết lập "dây chuyền xử lý" - hồ sơ đi vào → lọc → sắp xếp → lấy 10 → ra kết quả!

### So sánh code TRƯỚC và SAU khi có Stream

```java
List<String> names = Arrays.asList("John", "Jane", "Bob", "Alice", "Tom");

// ❌ CÁCH CŨ: Imperative (mệnh lệnh) - phải nói "LÀM THẾ NÀO"
List<String> result = new ArrayList<>();
for (String name : names) {
    if (name.length() > 3) {                    // Bước 1: Lọc
        result.add(name.toUpperCase());          // Bước 2: Chuyển hoa
    }
}
Collections.sort(result);                        // Bước 3: Sắp xếp
// Cần 6 dòng code, phải tạo list tạm, khó đọc

// ✅ CÁCH MỚI: Declarative (khai báo) - chỉ cần nói "MUỐN GÌ"
List<String> result2 = names.stream()            // Tạo dây chuyền
    .filter(name -> name.length() > 3)           // Lọc tên > 3 ký tự
    .map(String::toUpperCase)                    // Chuyển thành chữ HOA
    .sorted()                                    // Sắp xếp A-Z
    .collect(Collectors.toList());               // Thu thập kết quả
// 1 câu lệnh, đọc từ trên xuống như đọc văn bản
```

### 💡 Mẹo nhớ
> **Stream giống như DÂY CHUYỀN SẢN XUẤT trong nhà máy:**
> ```
> Nguyên liệu → [Lọc tạp] → [Cắt hình] → [Sơn màu] → [Đóng gói] → Thành phẩm
>     ↑              ↑            ↑            ↑            ↑            ↑
>   Source     Intermediate  Intermediate  Intermediate  Terminal     Result
>   (Nguồn)   (Trung gian)  (Trung gian)  (Trung gian)  (Kết thúc)  (Kết quả)
> ```
> - **Source**: Nguyên liệu đầu vào (List, Set, Array...)
> - **Intermediate**: Các khâu xử lý giữa chừng (filter, map, sorted...)
> - **Terminal**: Khâu cuối cùng để lấy thành phẩm (collect, forEach, count...)

---

## 1. Stream Basics (Kiến Thức Cơ Bản)

### 1.1. Stream là gì?

```
┌─────────────────────────────────────────────────────────┐
│                    STREAM LÀ GÌ?                        │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  ✅ Stream LÀ:                                          │
│  • Một "ống dẫn" để xử lý dữ liệu                     │
│  • Abstraction (lớp trừu tượng) cho data processing    │
│  • Hỗ trợ xử lý tuần tự hoặc song song                │
│                                                         │
│  ❌ Stream KHÔNG PHẢI:                                   │
│  • Data structure (cấu trúc dữ liệu) - không lưu data │
│  • Collection - không thể add/remove phần tử           │
│  • Reusable - dùng xong là bỏ, KHÔNG dùng lại được    │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

### 1.2. Các cách tạo Stream

```java
// === CÁCH 1: Từ Collection (phổ biến nhất) ===
List<String> list = Arrays.asList("a", "b", "c");
Stream<String> stream1 = list.stream();           // Stream tuần tự
Stream<String> stream1p = list.parallelStream();  // Stream song song

// === CÁCH 2: Từ Array ===
String[] array = {"a", "b", "c"};
Stream<String> stream2 = Arrays.stream(array);    // Chuyển mảng thành Stream

// === CÁCH 3: Stream.of() - Tạo trực tiếp ===
Stream<String> stream3 = Stream.of("a", "b", "c");

// === CÁCH 4: Stream rỗng ===
Stream<String> empty = Stream.empty();            // Hữu ích khi cần trả về Stream rỗng

// === CÁCH 5: Stream.generate() - Tạo vô hạn ===
// Tạo 5 số ngẫu nhiên (phải dùng limit() để giới hạn!)
Stream<Double> randoms = Stream.generate(Math::random).limit(5);

// === CÁCH 6: Stream.iterate() - Tạo dãy số ===
// Bắt đầu từ 0, mỗi lần +2 → tạo dãy số chẵn
Stream<Integer> evens = Stream.iterate(0, n -> n + 2).limit(10);
// Kết quả: 0, 2, 4, 6, 8, 10, 12, 14, 16, 18

// Java 9+: iterate() với điều kiện dừng (không cần limit nữa)
Stream<Integer> evens2 = Stream.iterate(0, n -> n < 20, n -> n + 2);
// Tham số: giá trị đầu, điều kiện tiếp tục, bước nhảy

// === CÁCH 7: IntStream, LongStream, DoubleStream (Stream cho kiểu nguyên thủy) ===
IntStream range1 = IntStream.range(1, 10);        // 1 đến 9 (không bao gồm 10)
IntStream range2 = IntStream.rangeClosed(1, 10);   // 1 đến 10 (bao gồm 10)

// === CÁCH 8: Từ String ===
IntStream chars = "Hello".chars();                // Stream các mã ký tự

// === CÁCH 9: Từ File ===
Stream<String> lines = Files.lines(Path.of("file.txt")); // Đọc từng dòng file
```

### 1.3. Vòng đời của Stream

```
┌──────────────────────────────────────────────────────────────┐
│                     VÒNG ĐỜI CỦA STREAM                     │
│                                                              │
│  ① Source (Nguồn)                                            │
│     │  List, Set, Array, File...                             │
│     ▼                                                        │
│  ② Intermediate Operations (Thao tác trung gian)            │
│     │  filter(), map(), sorted(), distinct()...              │
│     │  ⚡ LAZY! Chỉ chạy khi có Terminal Operation          │
│     ▼                                                        │
│  ③ Terminal Operation (Thao tác kết thúc)                    │
│     │  collect(), forEach(), count(), reduce()...            │
│     │  🔥 TRIGGER! Kích hoạt toàn bộ pipeline               │
│     ▼                                                        │
│  ④ Stream bị đóng, KHÔNG thể tái sử dụng                   │
└──────────────────────────────────────────────────────────────┘
```

```java
List<String> names = Arrays.asList("John", "Jane", "Bob", "Alice");

names.stream()                          // ① Tạo Stream từ List
    .filter(n -> n.length() > 3)        // ② Trung gian: Lọc tên > 3 ký tự
    .map(String::toUpperCase)           // ② Trung gian: Chuyển thành chữ hoa
    .sorted()                           // ② Trung gian: Sắp xếp A-Z
    .forEach(System.out::println);      // ③ Kết thúc: In ra từng phần tử

// ⚠️ SAU KHI Terminal chạy xong → Stream bị đóng vĩnh viễn
```

### 💡 Mẹo nhớ: Lazy Evaluation (Đánh Giá Lười)
```java
// Intermediate operations KHÔNG chạy cho đến khi có Terminal operation
Stream<String> stream = names.stream()
    .filter(n -> {
        System.out.println("Đang lọc: " + n);    // Dòng này CHƯA chạy!
        return n.length() > 3;
    })
    .map(n -> {
        System.out.println("Đang map: " + n);     // Dòng này CHƯA chạy!
        return n.toUpperCase();
    });

// ↑ Chưa in gì hết! Vì chưa có Terminal operation

System.out.println("--- Bắt đầu Terminal ---");
stream.forEach(System.out::println);  // ← BÂY GIỜ mới chạy filter + map!

// Output:
// --- Bắt đầu Terminal ---
// Đang lọc: John        ← filter chạy từng phần tử
// Đang map: John         ← map chạy ngay sau filter (không chờ filter xong hết)
// JOHN
// Đang lọc: Jane
// Đang map: Jane
// JANE
// Đang lọc: Bob          ← Bob bị lọc ra (length = 3), KHÔNG chạy map
// Đang lọc: Alice
// Đang map: Alice
// ALICE
```

> **Tại sao Lazy?** Vì nếu bạn chỉ cần `findFirst()`, Stream sẽ DỪNG NGAY khi tìm thấy kết quả đầu tiên, không cần xử lý hết toàn bộ danh sách. Tiết kiệm hiệu suất!

---

## 2. Intermediate Operations (Thao Tác Trung Gian)

### Bảng tổng hợp

| Operation | Tác dụng | Ví dụ |
|-----------|----------|-------|
| `filter(Predicate)` | Lọc phần tử theo điều kiện | `.filter(n -> n > 5)` |
| `map(Function)` | Biến đổi từng phần tử | `.map(String::toUpperCase)` |
| `flatMap(Function)` | Gỡ lồng (flatten) | `.flatMap(List::stream)` |
| `sorted()` | Sắp xếp | `.sorted(Comparator.reverseOrder())` |
| `distinct()` | Loại bỏ trùng lặp | `.distinct()` |
| `limit(n)` | Lấy n phần tử đầu | `.limit(5)` |
| `skip(n)` | Bỏ qua n phần tử đầu | `.skip(3)` |
| `peek(Consumer)` | Nhìn trộm (debug) | `.peek(System.out::println)` |
| `takeWhile(Predicate)` | Lấy khi còn đúng (Java 9+) | `.takeWhile(n -> n < 5)` |
| `dropWhile(Predicate)` | Bỏ khi còn đúng (Java 9+) | `.dropWhile(n -> n < 5)` |

### 2.1. filter() - Lọc phần tử

> **Ví dụ đời thường**: Giống như lọc cà phê - chỉ cho nước qua, giữ lại bã.

```java
List<Integer> numbers = Arrays.asList(1, 2, 3, 4, 5, 6, 7, 8, 9, 10);

// Lọc số chẵn (chỉ giữ lại phần tử thỏa điều kiện)
List<Integer> evens = numbers.stream()
    .filter(n -> n % 2 == 0)             // Giữ lại nếu chia 2 dư 0
    .collect(Collectors.toList());
// Kết quả: [2, 4, 6, 8, 10]

// Lọc nhiều điều kiện (chaining - nối nhiều filter)
List<Integer> result = numbers.stream()
    .filter(n -> n > 3)                  // Điều kiện 1: lớn hơn 3
    .filter(n -> n < 8)                  // Điều kiện 2: nhỏ hơn 8
    .collect(Collectors.toList());
// Kết quả: [4, 5, 6, 7]

// 💡 Cũng có thể viết gộp bằng && (AND)
List<Integer> result2 = numbers.stream()
    .filter(n -> n > 3 && n < 8)         // Gộp 2 điều kiện
    .collect(Collectors.toList());
// Kết quả: [4, 5, 6, 7] - giống nhau
```

### 2.2. map() - Biến đổi phần tử

> **Ví dụ đời thường**: Giống như máy ép trái cây - đưa trái cây vào → ra nước ép. Mỗi trái cây biến thành nước ép tương ứng.

```java
List<String> names = Arrays.asList("john", "jane", "bob");

// Biến đổi: chữ thường → chữ HOA
List<String> upperNames = names.stream()
    .map(String::toUpperCase)            // Mỗi String → gọi toUpperCase()
    .collect(Collectors.toList());
// Kết quả: [JOHN, JANE, BOB]

// Biến đổi kiểu: String → Integer (lấy độ dài)
List<Integer> lengths = names.stream()
    .map(String::length)                 // Mỗi String → lấy length()
    .collect(Collectors.toList());
// Kết quả: [4, 4, 3]

// mapToInt() - dùng khi muốn tính toán số (tránh autoboxing)
int totalLength = names.stream()
    .mapToInt(String::length)            // Trả về IntStream (hiệu quả hơn)
    .sum();                              // IntStream có sẵn method sum()
// Kết quả: 11
```

```
map() hoạt động thế nào:

  Input:    ["john",  "jane",  "bob" ]
              │         │        │
  map():    toUpper  toUpper  toUpper
              │         │        │
  Output:   ["JOHN",  "JANE",  "BOB" ]

  → Mỗi phần tử ĐỘC LẬP đi qua hàm biến đổi
  → Số lượng phần tử KHÔNG thay đổi
```

### 2.3. flatMap() - Gỡ lồng (Flatten)

> **Ví dụ đời thường**: Bạn có **3 hộp quà**, mỗi hộp chứa nhiều kẹo. `flatMap` = **đổ hết kẹo từ tất cả hộp ra 1 đống**.

```java
// Vấn đề: Có list lồng trong list
List<List<Integer>> nested = Arrays.asList(
    Arrays.asList(1, 2, 3),       // Hộp 1: [1, 2, 3]
    Arrays.asList(4, 5, 6),       // Hộp 2: [4, 5, 6]
    Arrays.asList(7, 8, 9)        // Hộp 3: [7, 8, 9]
);

// Nếu dùng map() → vẫn còn lồng!
// nested.stream().map(List::stream) → Stream<Stream<Integer>> (lồng 2 lớp)

// flatMap() = map + flatten (gỡ lồng)
List<Integer> flat = nested.stream()
    .flatMap(List::stream)               // Mỗi List → stream → gộp lại
    .collect(Collectors.toList());
// Kết quả: [1, 2, 3, 4, 5, 6, 7, 8, 9]
```

```
flatMap() hoạt động thế nào:

  Input:    [ [1,2,3],  [4,5,6],  [7,8,9] ]
                │           │          │
  flatMap:   stream()    stream()   stream()
                │           │          │
              1,2,3       4,5,6      7,8,9
                \           |          /
                 \__________|_________/
                            |
  Output:   [1, 2, 3, 4, 5, 6, 7, 8, 9]

  → "Mở tung" từng phần tử và gộp lại thành 1 Stream
```

```java
// Ví dụ thực tế: Tách câu thành từ
List<String> sentences = Arrays.asList("Hello World", "Java Stream API");

List<String> words = sentences.stream()
    .flatMap(s -> Arrays.stream(s.split(" ")))  // Mỗi câu → tách thành mảng từ → stream
    .collect(Collectors.toList());
// Kết quả: [Hello, World, Java, Stream, API]

// Ví dụ thực tế: Lấy tất cả skills từ danh sách nhân viên
// Mỗi nhân viên có List<String> skills
List<String> allSkills = employees.stream()
    .flatMap(emp -> emp.getSkills().stream())    // Gỡ lồng skills
    .distinct()                                  // Loại bỏ trùng
    .collect(Collectors.toList());
```

### 2.4. sorted() - Sắp xếp

```java
List<Integer> numbers = Arrays.asList(5, 2, 8, 1, 9);

// Sắp xếp tự nhiên (tăng dần)
List<Integer> asc = numbers.stream()
    .sorted()                                    // Mặc định: tăng dần
    .collect(Collectors.toList());
// Kết quả: [1, 2, 5, 8, 9]

// Sắp xếp giảm dần
List<Integer> desc = numbers.stream()
    .sorted(Comparator.reverseOrder())           // Đảo ngược thứ tự
    .collect(Collectors.toList());
// Kết quả: [9, 8, 5, 2, 1]

// Sắp xếp Object theo 1 trường
List<Person> sortedByAge = people.stream()
    .sorted(Comparator.comparing(Person::getAge))     // Sắp xếp theo tuổi
    .collect(Collectors.toList());

// Sắp xếp theo nhiều tiêu chí
List<Person> sortedMulti = people.stream()
    .sorted(Comparator.comparing(Person::getAge)       // Tiêu chí 1: tuổi
                      .thenComparing(Person::getName))  // Tiêu chí 2: tên (khi tuổi bằng nhau)
    .collect(Collectors.toList());

// Sắp xếp giảm dần theo tuổi
List<Person> sortedByAgeDesc = people.stream()
    .sorted(Comparator.comparing(Person::getAge).reversed())  // reversed() = đảo ngược
    .collect(Collectors.toList());
```

### 2.5. distinct() - Loại bỏ trùng lặp

```java
List<Integer> numbers = Arrays.asList(1, 2, 2, 3, 3, 3, 4);

List<Integer> unique = numbers.stream()
    .distinct()                          // Giữ lại phần tử duy nhất (dựa trên equals())
    .collect(Collectors.toList());
// Kết quả: [1, 2, 3, 4]

// ⚠️ Với Object: distinct() dùng equals() để so sánh
// → Phải override equals() và hashCode() trong class!
```

### 2.6. limit() và skip() - Cắt và bỏ qua

> **Ví dụ đời thường**: `skip(5).limit(3)` giống như nói: "Bỏ qua 5 người đầu tiên trong hàng, lấy 3 người tiếp theo."

```java
List<Integer> numbers = Arrays.asList(1, 2, 3, 4, 5, 6, 7, 8, 9, 10);

// limit(n): Lấy n phần tử đầu tiên
List<Integer> first5 = numbers.stream()
    .limit(5)
    .collect(Collectors.toList());
// Kết quả: [1, 2, 3, 4, 5]

// skip(n): Bỏ qua n phần tử đầu tiên
List<Integer> after5 = numbers.stream()
    .skip(5)
    .collect(Collectors.toList());
// Kết quả: [6, 7, 8, 9, 10]

// 🔥 Ứng dụng thực tế: PHÂN TRANG (Pagination)
int page = 2;        // Trang thứ 2 (đánh số từ 1)
int pageSize = 3;    // Mỗi trang 3 phần tử

List<Integer> pageData = numbers.stream()
    .skip((page - 1) * pageSize)         // Bỏ qua (2-1)*3 = 3 phần tử đầu
    .limit(pageSize)                     // Lấy 3 phần tử tiếp theo
    .collect(Collectors.toList());
// Kết quả: [4, 5, 6]  ← Trang 2 với 3 phần tử/trang
```

### 2.7. peek() - Nhìn trộm (Debug)

```java
// peek() dùng để DEBUG - xem dữ liệu đang chảy qua pipeline như thế nào
List<Integer> result = Stream.of(1, 2, 3, 4, 5)
    .filter(n -> n > 2)
    .peek(n -> System.out.println("Sau filter: " + n))    // Debug sau bước filter
    .map(n -> n * 2)
    .peek(n -> System.out.println("Sau map: " + n))       // Debug sau bước map
    .collect(Collectors.toList());

// Output:
// Sau filter: 3
// Sau map: 6
// Sau filter: 4
// Sau map: 8
// Sau filter: 5
// Sau map: 10

// ⚠️ CẢNH BÁO: peek() chỉ dùng để debug, KHÔNG nên dùng để thay đổi dữ liệu!
```

### 2.8. takeWhile() và dropWhile() (Java 9+)

```java
List<Integer> numbers = Arrays.asList(1, 2, 3, 4, 5, 4, 3, 2, 1);

// takeWhile: LẤY phần tử KHI điều kiện còn ĐÚNG, DỪNG khi gặp SAI
List<Integer> taken = numbers.stream()
    .takeWhile(n -> n < 4)               // Lấy khi < 4, dừng khi gặp 4
    .collect(Collectors.toList());
// Kết quả: [1, 2, 3]  ← Dừng ngay khi gặp 4, KHÔNG xét tiếp

// dropWhile: BỎ QUA phần tử KHI điều kiện còn ĐÚNG, LẤY phần còn lại
List<Integer> dropped = numbers.stream()
    .dropWhile(n -> n < 4)               // Bỏ qua khi < 4, bắt đầu lấy từ 4
    .collect(Collectors.toList());
// Kết quả: [4, 5, 4, 3, 2, 1]  ← Lấy TẤT CẢ sau khi ngừng bỏ

// 💡 Khác với filter():
// filter() xét TẤT CẢ phần tử
// takeWhile/dropWhile DỪNG ngay khi điều kiện thay đổi
```

---

## 3. Terminal Operations (Thao Tác Kết Thúc)

> **Nhắc lại:** Terminal Operation là bước cuối cùng, kích hoạt toàn bộ pipeline chạy.

### Bảng tổng hợp

| Operation | Tác dụng | Kiểu trả về |
|-----------|----------|-------------|
| `forEach(Consumer)` | Xử lý từng phần tử | `void` |
| `count()` | Đếm số phần tử | `long` |
| `collect(Collector)` | Thu thập kết quả | Tùy Collector |
| `reduce(BinaryOperator)` | Gộp tất cả thành 1 giá trị | `Optional<T>` / `T` |
| `findFirst()` | Tìm phần tử đầu tiên | `Optional<T>` |
| `findAny()` | Tìm phần tử bất kỳ | `Optional<T>` |
| `anyMatch(Predicate)` | Có ít nhất 1 thỏa? | `boolean` |
| `allMatch(Predicate)` | Tất cả đều thỏa? | `boolean` |
| `noneMatch(Predicate)` | Không có ai thỏa? | `boolean` |
| `min(Comparator)` | Tìm giá trị nhỏ nhất | `Optional<T>` |
| `max(Comparator)` | Tìm giá trị lớn nhất | `Optional<T>` |
| `toArray()` | Chuyển thành mảng | `Object[]` / `T[]` |

### 3.1. forEach() - Xử lý từng phần tử

```java
List<String> names = Arrays.asList("John", "Jane", "Bob");

// In ra từng phần tử
names.stream()
    .forEach(System.out::println);       // Equivalent: .forEach(n -> System.out.println(n))

// ⚠️ Với parallel stream: forEach KHÔNG đảm bảo thứ tự!
names.parallelStream()
    .forEach(System.out::println);       // Có thể in: Bob, John, Jane (lộn xộn)

// ✅ Dùng forEachOrdered nếu cần giữ thứ tự
names.parallelStream()
    .forEachOrdered(System.out::println); // Luôn in đúng: John, Jane, Bob
```

### 3.2. collect() - Thu thập kết quả

```java
List<String> names = Arrays.asList("John", "Jane", "Bob");

// Thu thập vào List
List<String> list = names.stream()
    .collect(Collectors.toList());

// Thu thập vào Set (tự động loại trùng)
Set<String> set = names.stream()
    .collect(Collectors.toSet());

// Thu thập vào Collection cụ thể
LinkedList<String> linkedList = names.stream()
    .collect(Collectors.toCollection(LinkedList::new));

// 💡 Java 16+: Có thể dùng toList() ngắn gọn hơn
List<String> list2 = names.stream()
    .filter(n -> n.length() > 3)
    .toList();                           // Ngắn hơn collect(Collectors.toList())
    // ⚠️ Nhưng trả về List KHÔNG thể sửa đổi (unmodifiable)
```

### 3.3. reduce() - Gộp thành 1 giá trị

> **Ví dụ đời thường**: Giống như **cộng dồn tiền trong ví**. Bắt đầu từ 0đ, cộng từng tờ tiền vào.

```java
List<Integer> numbers = Arrays.asList(1, 2, 3, 4, 5);

// Tính tổng: bắt đầu từ 0, cộng dồn
//   0 + 1 = 1
//   1 + 2 = 3
//   3 + 3 = 6
//   6 + 4 = 10
//  10 + 5 = 15
int sum = numbers.stream()
    .reduce(0, (a, b) -> a + b);         // identity=0, accumulator=cộng dồn
// Kết quả: 15

// Tính tích: bắt đầu từ 1, nhân dồn
int product = numbers.stream()
    .reduce(1, (a, b) -> a * b);         // 1*1*2*3*4*5 = 120
// Kết quả: 120

// Không có identity → trả về Optional (vì list có thể rỗng)
Optional<Integer> max = numbers.stream()
    .reduce(Integer::max);               // Tìm max bằng cách so sánh từng cặp
max.ifPresent(System.out::println);      // 5

// Nối chuỗi
List<String> words = Arrays.asList("Hello", " ", "World");
String sentence = words.stream()
    .reduce("", String::concat);         // "" + "Hello" + " " + "World"
// Kết quả: "Hello World"
```

```
reduce() hoạt động thế nào (tính tổng):

  identity = 0  (giá trị khởi đầu)

  Bước 1:  0 + 1 = 1
  Bước 2:  1 + 2 = 3
  Bước 3:  3 + 3 = 6
  Bước 4:  6 + 4 = 10
  Bước 5: 10 + 5 = 15  ← Kết quả cuối cùng

  → "Gộp" (reduce) danh sách [1,2,3,4,5] thành 1 giá trị: 15
```

### 3.4. findFirst() và findAny()

```java
List<Integer> numbers = Arrays.asList(1, 2, 3, 4, 5);

// findFirst(): Tìm phần tử ĐẦU TIÊN thỏa điều kiện
Optional<Integer> first = numbers.stream()
    .filter(n -> n > 3)                  // Lọc: [4, 5]
    .findFirst();                        // Lấy phần tử đầu tiên
// Kết quả: Optional[4]

// findAny(): Tìm phần tử BẤT KỲ thỏa điều kiện
Optional<Integer> any = numbers.parallelStream()
    .filter(n -> n > 3)                  // Lọc: [4, 5]
    .findAny();                          // Lấy bất kỳ phần tử nào
// Kết quả: Optional[4] hoặc Optional[5] (không đoán trước được trong parallel)

// 💡 Khi nào dùng findAny?
// → Khi bạn KHÔNG quan tâm thứ tự, chỉ cần 1 kết quả bất kỳ
// → findAny nhanh hơn findFirst trong parallel stream
```

### 3.5. anyMatch(), allMatch(), noneMatch()

```java
List<Integer> numbers = Arrays.asList(1, 2, 3, 4, 5);

// anyMatch: CÓ ÍT NHẤT 1 phần tử thỏa điều kiện không?
boolean hasEven = numbers.stream()
    .anyMatch(n -> n % 2 == 0);          // Có số chẵn không?
// Kết quả: true (vì có 2, 4)

// allMatch: TẤT CẢ phần tử đều thỏa điều kiện không?
boolean allPositive = numbers.stream()
    .allMatch(n -> n > 0);               // Tất cả > 0 không?
// Kết quả: true

// noneMatch: KHÔNG CÓ phần tử nào thỏa điều kiện?
boolean noNegative = numbers.stream()
    .noneMatch(n -> n < 0);              // Không có số âm?
// Kết quả: true

// 💡 Mẹo nhớ:
// anyMatch  = "có ai... không?"  (tìm thấy 1 là dừng, trả true)
// allMatch  = "tất cả đều...?"   (tìm thấy 1 sai là dừng, trả false)
// noneMatch = "không ai... đúng không?" (tìm thấy 1 đúng là dừng, trả false)
```

### 3.6. min() và max()

```java
List<Integer> numbers = Arrays.asList(3, 1, 4, 1, 5, 9);

Optional<Integer> min = numbers.stream()
    .min(Comparator.naturalOrder());     // Tìm giá trị nhỏ nhất
// Kết quả: Optional[1]

Optional<Integer> max = numbers.stream()
    .max(Comparator.naturalOrder());     // Tìm giá trị lớn nhất
// Kết quả: Optional[9]

// Tìm người trẻ nhất
Optional<Person> youngest = people.stream()
    .min(Comparator.comparing(Person::getAge));

// Tìm người già nhất
Optional<Person> oldest = people.stream()
    .max(Comparator.comparing(Person::getAge));
```

### 3.7. toArray() - Chuyển thành mảng

```java
// Chuyển Stream thành mảng String[]
String[] array = Stream.of("a", "b", "c")
    .toArray(String[]::new);             // Cần truyền constructor reference

// Chuyển thành mảng Integer[]
Integer[] intArray = Stream.of(1, 2, 3)
    .toArray(Integer[]::new);
```

---

## 4. Collectors (Bộ Thu Thập Nâng Cao)

### 4.1. joining() - Nối chuỗi

```java
List<String> names = Arrays.asList("John", "Jane", "Bob", "Alice");

// Nối không có dấu phân cách
String joined1 = names.stream()
    .collect(Collectors.joining());
// Kết quả: "JohnJaneBobAlice"

// Nối với dấu phẩy
String joined2 = names.stream()
    .collect(Collectors.joining(", "));
// Kết quả: "John, Jane, Bob, Alice"

// Nối với dấu phẩy + ngoặc vuông bao quanh
String joined3 = names.stream()
    .collect(Collectors.joining(", ", "[", "]"));
//                                 ↑       ↑     ↑
//                           separator  prefix  suffix
// Kết quả: "[John, Jane, Bob, Alice]"
```

### 4.2. Thống kê (Statistics)

```java
List<Integer> numbers = Arrays.asList(1, 2, 3, 4, 5);

// Đếm
long count = numbers.stream()
    .collect(Collectors.counting());             // 5

// Tính tổng
int sum = numbers.stream()
    .collect(Collectors.summingInt(Integer::intValue));    // 15

// Tính trung bình
double avg = numbers.stream()
    .collect(Collectors.averagingInt(Integer::intValue));  // 3.0

// 🔥 Tổng hợp tất cả thống kê trong 1 lần
IntSummaryStatistics stats = numbers.stream()
    .collect(Collectors.summarizingInt(Integer::intValue));

System.out.println("Số lượng: " + stats.getCount());     // 5
System.out.println("Tổng: " + stats.getSum());           // 15
System.out.println("Nhỏ nhất: " + stats.getMin());       // 1
System.out.println("Lớn nhất: " + stats.getMax());       // 5
System.out.println("Trung bình: " + stats.getAverage()); // 3.0
```

### 4.3. groupingBy() - Nhóm theo tiêu chí

> **Ví dụ đời thường**: Giống như **chia học sinh thành các nhóm** theo lớp. Mỗi lớp là 1 key, danh sách học sinh là value.

```java
List<Person> people = Arrays.asList(
    new Person("John", 25, "IT"),
    new Person("Jane", 30, "HR"),
    new Person("Bob", 25, "IT"),
    new Person("Alice", 28, "HR")
);

// === Nhóm cơ bản: theo phòng ban ===
Map<String, List<Person>> byDept = people.stream()
    .collect(Collectors.groupingBy(Person::getDepartment));
// {
//   "IT"  → [John(25), Bob(25)],
//   "HR"  → [Jane(30), Alice(28)]
// }

// === Nhóm + đếm: số người mỗi phòng ===
Map<String, Long> countByDept = people.stream()
    .collect(Collectors.groupingBy(
        Person::getDepartment,           // Key: tên phòng ban
        Collectors.counting()            // Value: đếm số người
    ));
// {"IT"=2, "HR"=2}

// === Nhóm + biến đổi: lấy tên thay vì object ===
Map<String, List<String>> namesByDept = people.stream()
    .collect(Collectors.groupingBy(
        Person::getDepartment,           // Key: tên phòng ban
        Collectors.mapping(              // Value: chỉ lấy tên
            Person::getName,
            Collectors.toList()
        )
    ));
// {"IT"=["John", "Bob"], "HR"=["Jane", "Alice"]}

// === Nhóm lồng nhau: phòng ban → tuổi → danh sách ===
Map<String, Map<Integer, List<Person>>> byDeptAndAge = people.stream()
    .collect(Collectors.groupingBy(
        Person::getDepartment,           // Nhóm lớp 1: phòng ban
        Collectors.groupingBy(           // Nhóm lớp 2: tuổi
            Person::getAge
        )
    ));
// {
//   "IT" → {25 → [John, Bob]},
//   "HR" → {30 → [Jane], 28 → [Alice]}
// }
```

```
groupingBy() trực quan:

  Input: [John(IT), Jane(HR), Bob(IT), Alice(HR)]

  groupingBy(getDepartment):

  ┌─────────────────────────┐
  │ Key: "IT"               │
  │ Value: [John, Bob]      │
  ├─────────────────────────┤
  │ Key: "HR"               │
  │ Value: [Jane, Alice]    │
  └─────────────────────────┘
```

### 4.4. partitioningBy() - Chia 2 nhóm (true/false)

> **Ví dụ đời thường**: Giống như **chia học sinh thành 2 nhóm**: ĐẬU và TRƯỢT.

```java
List<Integer> numbers = Arrays.asList(1, 2, 3, 4, 5, 6, 7, 8, 9, 10);

// Chia thành 2 nhóm: chẵn và lẻ
Map<Boolean, List<Integer>> partition = numbers.stream()
    .collect(Collectors.partitioningBy(n -> n % 2 == 0));
// {
//   true  → [2, 4, 6, 8, 10],     ← Chẵn (thỏa điều kiện)
//   false → [1, 3, 5, 7, 9]       ← Lẻ (không thỏa điều kiện)
// }

// Chia + đếm
Map<Boolean, Long> partitionCount = numbers.stream()
    .collect(Collectors.partitioningBy(
        n -> n % 2 == 0,
        Collectors.counting()            // Đếm mỗi nhóm
    ));
// {true=5, false=5}

// 💡 partitioningBy vs groupingBy:
// partitioningBy: Luôn chia ĐÚNG 2 nhóm (true/false), kể cả khi 1 nhóm rỗng
// groupingBy: Chia nhiều nhóm tùy ý, nhóm rỗng sẽ KHÔNG xuất hiện
```

### 4.5. toMap() - Thu thập thành Map

```java
List<Person> people = Arrays.asList(
    new Person("John", 25),
    new Person("Jane", 30),
    new Person("Bob", 20)
);

// === Cơ bản: tên → tuổi ===
Map<String, Integer> nameToAge = people.stream()
    .collect(Collectors.toMap(
        Person::getName,                 // Key: tên
        Person::getAge                   // Value: tuổi
    ));
// {"John"=25, "Jane"=30, "Bob"=20}

// === Xử lý trùng key ===
// ⚠️ Nếu có 2 người cùng tên → EXCEPTION!
// ✅ Cần merge function để xử lý
Map<String, Integer> nameToAge2 = people.stream()
    .collect(Collectors.toMap(
        Person::getName,
        Person::getAge,
        (existing, replacement) -> existing  // Nếu trùng key → giữ giá trị cũ
    ));

// === Chỉ định kiểu Map ===
LinkedHashMap<String, Integer> linkedMap = people.stream()
    .collect(Collectors.toMap(
        Person::getName,
        Person::getAge,
        (e, r) -> e,                     // Merge function (bắt buộc khi chỉ định Map type)
        LinkedHashMap::new               // Kiểu Map muốn dùng
    ));
```

---

## 5. Primitive Streams (Stream Cho Kiểu Nguyên Thủy)

> **Tại sao cần?** `Stream<Integer>` phải autoboxing/unboxing liên tục (int ↔ Integer), tốn bộ nhớ và chậm hơn. `IntStream` xử lý trực tiếp kiểu `int`, nhanh hơn!

```java
// === IntStream ===
IntStream intStream = IntStream.range(1, 10);      // 1 đến 9
int sum = intStream.sum();                          // Có sẵn sum() - không cần reduce

OptionalDouble avg = IntStream.of(1, 2, 3).average(); // Có sẵn average()

// === LongStream ===
LongStream longStream = LongStream.rangeClosed(1, 100); // 1 đến 100
long total = longStream.sum();

// === DoubleStream ===
DoubleStream doubleStream = DoubleStream.of(1.1, 2.2, 3.3);

// === Chuyển đổi giữa các loại Stream ===
// Stream<Integer> → IntStream
IntStream ints = Stream.of(1, 2, 3).mapToInt(Integer::intValue);

// IntStream → Stream<Integer>
Stream<Integer> boxed = IntStream.of(1, 2, 3).boxed();  // boxed() = đóng hộp

// === Thống kê nhanh ===
IntSummaryStatistics stats = IntStream.range(1, 100).summaryStatistics();
// Có getCount(), getSum(), getMin(), getMax(), getAverage()
```

```
Khi nào dùng Primitive Stream?

┌──────────────────────────┬──────────────────────────┐
│ Stream<Integer>          │ IntStream                │
├──────────────────────────┼──────────────────────────┤
│ Mỗi phần tử là Object   │ Mỗi phần tử là int      │
│ Autoboxing/Unboxing      │ Không cần boxing         │
│ Tốn bộ nhớ hơn           │ Tiết kiệm bộ nhớ        │
│ Không có sum(), avg()    │ Có sẵn sum(), average()  │
│ Dùng khi cần flexibility │ Dùng khi xử lý số nhiều │
└──────────────────────────┴──────────────────────────┘

💡 Quy tắc: Khi xử lý số (tính tổng, trung bình...) → ưu tiên IntStream/LongStream/DoubleStream
```

---

## 6. Parallel Streams (Luồng Song Song)

> **Ví dụ đời thường**: Thay vì **1 người rửa 100 cái bát**, ta chia cho **4 người, mỗi người rửa 25 cái** → xong nhanh hơn.

```java
List<Integer> numbers = Arrays.asList(1, 2, 3, 4, 5, 6, 7, 8, 9, 10);

// === Cách 1: Tạo parallel stream từ đầu ===
long count = numbers.parallelStream()        // parallelStream() thay vì stream()
    .filter(n -> n % 2 == 0)
    .count();

// === Cách 2: Chuyển stream thường → parallel ===
long count2 = numbers.stream()
    .parallel()                              // Chuyển sang parallel giữa chừng
    .filter(n -> n % 2 == 0)
    .count();
```

### ⚠️ CẢNH BÁO: Parallel Stream KHÔNG phải lúc nào cũng nhanh hơn!

```java
// ❌ SAI: Dùng parallel với side effects (tác dụng phụ)
List<Integer> result = new ArrayList<>();
numbers.parallelStream()
    .forEach(result::add);       // ArrayList KHÔNG thread-safe!
// → Có thể mất phần tử, exception, kết quả sai

// ✅ ĐÚNG: Dùng collect thay vì forEach + add
List<Integer> safeResult = numbers.parallelStream()
    .collect(Collectors.toList());

// ❌ SAI: Parallel cho data nhỏ (overhead > benefit)
// 100 phần tử + phép tính đơn giản → sequential nhanh hơn!

// ✅ ĐÚNG: Parallel cho data lớn + phép tính nặng
List<Integer> bigResult = hugeList.parallelStream()   // 1 triệu phần tử
    .map(n -> expensiveCalculation(n))                 // Phép tính tốn thời gian
    .collect(Collectors.toList());
```

### Khi nào NÊN và KHÔNG NÊN dùng Parallel?

```
┌────────────────────────────────────────────────────────────────┐
│                    PARALLEL STREAM GUIDELINES                  │
├────────────────────────────┬───────────────────────────────────┤
│ ✅ NÊN dùng khi:          │ ❌ KHÔNG NÊN dùng khi:            │
├────────────────────────────┼───────────────────────────────────┤
│ • Data lớn (> 10,000)     │ • Data nhỏ (< 1,000)             │
│ • Phép tính nặng (CPU)    │ • Phép tính đơn giản             │
│ • Không có side effects   │ • Có side effects (shared state) │
│ • Nguồn dữ liệu tốt      │ • LinkedList (chia khó)          │
│   (ArrayList, Array)      │ • Cần giữ thứ tự nghiêm ngặt    │
│ • Independent operations  │ • I/O bound (đọc file, network)  │
└────────────────────────────┴───────────────────────────────────┘
```

---

## 7. Ví Dụ Thực Tế Tổng Hợp

### Ví dụ 1: Xử lý danh sách nhân viên

```java
class Employee {
    String name;
    String department;
    double salary;
    int yearsOfExp;
    // constructor, getters...
}

List<Employee> employees = getEmployees();  // Giả sử có danh sách nhân viên

// 🔍 Tìm nhân viên lương cao nhất mỗi phòng ban
Map<String, Optional<Employee>> topEarners = employees.stream()
    .collect(Collectors.groupingBy(
        Employee::getDepartment,                          // Nhóm theo phòng ban
        Collectors.maxBy(                                 // Tìm max
            Comparator.comparing(Employee::getSalary)     // So sánh theo lương
        )
    ));

// 📊 Tính lương trung bình theo phòng ban
Map<String, Double> avgSalaryByDept = employees.stream()
    .collect(Collectors.groupingBy(
        Employee::getDepartment,
        Collectors.averagingDouble(Employee::getSalary)
    ));

// 📋 Lấy danh sách tên nhân viên, phân cách bởi dấu phẩy, theo từng phòng
Map<String, String> namesByDept = employees.stream()
    .collect(Collectors.groupingBy(
        Employee::getDepartment,
        Collectors.mapping(
            Employee::getName,
            Collectors.joining(", ")                      // Nối tên bằng dấu phẩy
        )
    ));
// {"IT" → "John, Bob, Alice", "HR" → "Jane, Tom"}

// 🏆 Top 3 nhân viên lương cao nhất có kinh nghiệm > 5 năm
List<Employee> top3 = employees.stream()
    .filter(e -> e.getYearsOfExp() > 5)                  // Lọc exp > 5 năm
    .sorted(Comparator.comparing(Employee::getSalary).reversed()) // Sắp xếp giảm dần
    .limit(3)                                             // Lấy top 3
    .collect(Collectors.toList());
```

### Ví dụ 2: Xử lý đơn hàng thương mại điện tử

```java
class Order {
    String orderId;
    String customerId;
    LocalDate date;
    List<OrderItem> items;
    // getters...
}

class OrderItem {
    String productName;
    double price;
    int quantity;
    // getters...
}

List<Order> orders = getOrders();

// 💰 Tổng doanh thu
double totalRevenue = orders.stream()
    .flatMap(order -> order.getItems().stream())          // Gỡ lồng: đơn hàng → items
    .mapToDouble(item -> item.getPrice() * item.getQuantity()) // Tính tiền mỗi item
    .sum();

// 📅 Doanh thu theo tháng
Map<YearMonth, Double> revenueByMonth = orders.stream()
    .collect(Collectors.groupingBy(
        order -> YearMonth.from(order.getDate()),         // Nhóm theo tháng
        Collectors.flatMapping(                           // Java 9+
            order -> order.getItems().stream()
                .map(item -> item.getPrice() * item.getQuantity()),
            Collectors.summingDouble(Double::doubleValue)
        )
    ));

// 🛒 Sản phẩm bán chạy nhất (theo số lượng)
Optional<Map.Entry<String, Integer>> bestSeller = orders.stream()
    .flatMap(order -> order.getItems().stream())
    .collect(Collectors.groupingBy(
        OrderItem::getProductName,
        Collectors.summingInt(OrderItem::getQuantity)
    ))
    .entrySet().stream()
    .max(Map.Entry.comparingByValue());

bestSeller.ifPresent(entry ->
    System.out.println("Bán chạy nhất: " + entry.getKey() + " (" + entry.getValue() + " cái)")
);
```

---

## 8. Sai Lầm Thường Gặp

### ❌ Sai lầm 1: Dùng lại Stream đã đóng

```java
// ❌ SAI: Một Stream chỉ dùng 1 lần!
Stream<String> stream = names.stream().filter(n -> n.length() > 3);

stream.forEach(System.out::println);   // Lần 1: OK
stream.count();                         // Lần 2: 💥 IllegalStateException!
// "stream has already been operated upon or closed"

// ✅ ĐÚNG: Tạo Stream mới mỗi lần dùng
names.stream().filter(n -> n.length() > 3).forEach(System.out::println);
long count = names.stream().filter(n -> n.length() > 3).count();
```

### ❌ Sai lầm 2: Dùng forEach để tạo list kết quả

```java
// ❌ SAI: Dùng forEach + add (kiểu imperative trong stream)
List<String> result = new ArrayList<>();
names.stream()
    .filter(n -> n.length() > 3)
    .forEach(n -> result.add(n.toUpperCase()));   // Side effect!

// ✅ ĐÚNG: Dùng map + collect
List<String> result = names.stream()
    .filter(n -> n.length() > 3)
    .map(String::toUpperCase)
    .collect(Collectors.toList());

// 💡 Tại sao SAI?
// 1. Khó đọc: mix imperative và declarative
// 2. Không thread-safe: sẽ lỗi với parallelStream
// 3. Vi phạm nguyên tắc: Stream operations nên KHÔNG có side effects
```

### ❌ Sai lầm 3: Stream cho mọi thứ (over-use)

```java
// ❌ KHÔNG CẦN Stream: Chỉ lặp qua list và in
names.stream().forEach(System.out::println);

// ✅ TỐT HƠN: Enhanced for loop đơn giản hơn
for (String name : names) {
    System.out.println(name);
}

// ❌ KHÔNG CẦN Stream: Chỉ tìm 1 phần tử trong list nhỏ
Optional<String> found = names.stream()
    .filter(n -> n.equals("John"))
    .findFirst();

// ✅ TỐT HƠN: contains() đơn giản
boolean found = names.contains("John");

// 💡 Quy tắc: Dùng Stream khi có CHUỖI thao tác (filter + map + collect)
// Không dùng Stream cho thao tác đơn giản mà for loop làm tốt hơn
```

### ❌ Sai lầm 4: Quên xử lý Optional từ Terminal operations

```java
// ❌ SAI: Gọi get() trực tiếp → NullPointerException nếu rỗng!
String first = names.stream()
    .filter(n -> n.startsWith("Z"))
    .findFirst()
    .get();                              // 💥 NoSuchElementException! Không có tên bắt đầu Z

// ✅ ĐÚNG: Dùng orElse / orElseThrow / ifPresent
String first1 = names.stream()
    .filter(n -> n.startsWith("Z"))
    .findFirst()
    .orElse("Không tìm thấy");          // Giá trị mặc định

String first2 = names.stream()
    .filter(n -> n.startsWith("Z"))
    .findFirst()
    .orElseThrow(() -> new RuntimeException("Không tìm thấy")); // Throw cụ thể
```

---

## 9. Tóm Tắt Cuối Ngày

| Khái niệm | Giải thích | Ví dụ |
|------------|-----------|-------|
| **Stream** | Dây chuyền xử lý dữ liệu, KHÔNG lưu trữ | `list.stream()` |
| **Lazy evaluation** | Intermediate ops chỉ chạy khi có Terminal | filter/map chờ collect |
| **filter()** | Lọc phần tử theo điều kiện | `.filter(n -> n > 5)` |
| **map()** | Biến đổi từng phần tử | `.map(String::toUpperCase)` |
| **flatMap()** | Gỡ lồng + biến đổi | `.flatMap(List::stream)` |
| **sorted()** | Sắp xếp | `.sorted(Comparator.comparing(...))` |
| **distinct()** | Loại trùng lặp | `.distinct()` |
| **limit/skip** | Cắt / bỏ qua phần tử | `.skip(5).limit(10)` |
| **collect()** | Thu thập kết quả | `.collect(Collectors.toList())` |
| **reduce()** | Gộp thành 1 giá trị | `.reduce(0, Integer::sum)` |
| **groupingBy()** | Nhóm theo tiêu chí | `groupingBy(Person::getDept)` |
| **partitioningBy()** | Chia 2 nhóm true/false | `partitioningBy(n -> n > 5)` |
| **toMap()** | Thu thập thành Map | `toMap(Person::getName, Person::getAge)` |
| **Parallel Stream** | Xử lý song song | `.parallelStream()` |
| **Primitive Stream** | Stream cho int/long/double | `IntStream.range(1, 10)` |

---

## 10. Câu Hỏi Phỏng Vấn Thường Gặp

### 🔥 Câu 1: Stream khác Collection như thế nào?
**Trả lời:**
- **Collection** (List, Set): Lưu trữ dữ liệu trong bộ nhớ, có thể add/remove phần tử, truy cập nhiều lần
- **Stream**: Không lưu trữ, chỉ là pipeline xử lý dữ liệu. Dùng 1 lần rồi bỏ. Hỗ trợ lazy evaluation và parallel processing

### 🔥 Câu 2: Lazy evaluation là gì? Lợi ích?
**Trả lời:**
Lazy evaluation = các intermediate operations (filter, map...) KHÔNG chạy ngay mà CHỜ đến khi có terminal operation. Lợi ích:
1. **Hiệu suất**: Nếu dùng `findFirst()`, chỉ xử lý đến phần tử đầu tiên thỏa điều kiện rồi dừng
2. **Tối ưu**: JVM có thể gộp nhiều operations thành 1 pass (short-circuit fusion)
3. **Tiết kiệm**: Không tạo collection trung gian cho mỗi bước

### 🔥 Câu 3: map() khác flatMap() như thế nào?
**Trả lời:**
- `map()`: Biến đổi 1→1. Mỗi phần tử input → 1 phần tử output. Nếu input là List<List<T>>, output vẫn là List<List<T>>
- `flatMap()`: Biến đổi 1→N rồi gỡ lồng. Mỗi phần tử input → 1 Stream → gộp tất cả Stream lại. List<List<T>> → List<T>

### 🔥 Câu 4: Khi nào dùng reduce() vs collect()?
**Trả lời:**
- `reduce()`: Khi muốn gộp thành **1 giá trị đơn** (tổng, tích, max, min, nối chuỗi). Kết quả là immutable
- `collect()`: Khi muốn gộp thành **collection hoặc cấu trúc phức tạp** (List, Set, Map, grouping). Kết quả là mutable container

### 🔥 Câu 5: Parallel Stream có luôn nhanh hơn không?
**Trả lời:**
KHÔNG! Parallel Stream có overhead (chi phí) cho thread management, splitting, merging. Chỉ nhanh hơn khi:
- Data lớn (> 10,000 phần tử)
- Phép tính CPU-intensive (tốn thời gian tính toán)
- Data source dễ chia (ArrayList, Array) - không phải LinkedList
- Không có shared mutable state (side effects)

### 🔥 Câu 6: Tại sao Stream chỉ dùng được 1 lần?
**Trả lời:**
Stream được thiết kế theo mô hình "pipeline" - giống như băng chuyền 1 chiều. Sau khi terminal operation chạy xong, tất cả dữ liệu đã "chảy" qua pipeline và không còn gì để xử lý. Thiết kế này:
1. Tránh lưu trữ dữ liệu trung gian → tiết kiệm bộ nhớ
2. Cho phép lazy evaluation hiệu quả
3. Đơn giản hóa parallel processing
Nếu muốn dùng lại → tạo Stream mới từ source (Collection, Array...)

---

## Navigation

- [← Day 9: Lambda & Functional](./day-09-lambda-functional.md)
- [Day 11: File I/O →](./day-11-file-io.md)
