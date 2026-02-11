# Day 3: Exception Handling + Strings + Collections

> Gộp từ bản 19 ngày: Day 5 (Exception) + Day 6 (Strings) + Day 7 (Collections)
> 📖 Đọc sâu: [day-05](../java-fundamentals/day-05-exception-handling.md) | [day-06](../java-fundamentals/day-06-strings-wrappers.md) | [day-07](../java-fundamentals/day-07-collections-basics.md)

---

## Phần A: Exception Handling (Xử lý ngoại lệ)

### 1. Phân loại Exception

```
         Throwable
         ├── Error (Lỗi hệ thống — KHÔNG bắt)
         │   ├── OutOfMemoryError
         │   └── StackOverflowError
         └── Exception
             ├── Checked Exception (BẮT BUỘC xử lý — compile-time)
             │   ├── IOException
             │   ├── SQLException
             │   └── FileNotFoundException
             └── RuntimeException (Unchecked — KHÔNG bắt buộc xử lý)
                 ├── NullPointerException
                 ├── ArrayIndexOutOfBoundsException
                 ├── IllegalArgumentException
                 └── ClassCastException
```

| Loại | Phải try-catch? | Ví dụ | Khi nào dùng |
|------|----------------|-------|-------------|
| **Checked** | ✅ Bắt buộc | IOException, SQLException | Lỗi NGOÀI tầm kiểm soát (I/O, DB) |
| **Unchecked** | ❌ Tùy chọn | NullPointerException | Lỗi DO code sai (bug) |

### 2. try-catch-finally

```java
try {
    FileReader reader = new FileReader("data.txt");
    // Đọc file...
} catch (FileNotFoundException e) {
    System.out.println("File không tồn tại: " + e.getMessage());
} catch (IOException e) {
    System.out.println("Lỗi đọc file: " + e.getMessage());
} finally {
    // LUÔN chạy dù có exception hay không — dùng để cleanup
    System.out.println("Cleanup xong");
}
```

### 3. try-with-resources (Java 7+) — Ưu tiên dùng cách này

```java
// Tự động đóng resource khi kết thúc block — KHÔNG cần finally
try (FileReader reader = new FileReader("data.txt");
     BufferedReader br = new BufferedReader(reader)) {

    String line = br.readLine();
} catch (IOException e) {
    e.printStackTrace();
}
// reader và br TỰ ĐỘNG đóng ở đây
```

### 4. Throw & Custom Exception

```java
// Throw exception
public void setAge(int age) {
    if (age < 0) throw new IllegalArgumentException("Age cannot be negative: " + age);
    this.age = age;
}

// Custom exception
public class InsufficientBalanceException extends RuntimeException {
    public InsufficientBalanceException(double amount, double balance) {
        super("Cannot withdraw " + amount + ", balance is only " + balance);
    }
}
```

💡 **Quy tắc thực tế:**
- Checked Exception → cho lỗi có thể **recover** (retry, fallback)
- Unchecked Exception → cho lỗi **không nên xảy ra** (bug, invalid input)
- **KHÔNG** catch `Exception` hoặc `Throwable` chung chung

---

## Phần B: Strings (Chuỗi)

### 1. String là Immutable (Bất biến)

```java
String s = "Hello";
s.toUpperCase();           // Trả về "HELLO" nhưng KHÔNG thay đổi s!
System.out.println(s);     // Vẫn là "Hello"

String upper = s.toUpperCase();  // ✅ Phải gán vào biến mới
```

### 2. Cheat Sheet — Methods hay dùng

| Method | Tác dụng | Ví dụ |
|--------|----------|-------|
| `length()` | Độ dài | `"Hello".length()` → 5 |
| `charAt(i)` | Ký tự tại vị trí i | `"Hello".charAt(0)` → 'H' |
| `substring(from, to)` | Cắt chuỗi | `"Hello".substring(1, 4)` → "ell" |
| `contains(s)` | Chứa chuỗi con? | `"Hello".contains("ell")` → true |
| `indexOf(s)` | Vị trí chuỗi con | `"Hello".indexOf("lo")` → 3 |
| `startsWith(s)` | Bắt đầu bằng? | `"Hello".startsWith("He")` → true |
| `toUpperCase()` | Chuyển hoa | `"hello".toUpperCase()` → "HELLO" |
| `toLowerCase()` | Chuyển thường | `"HELLO".toLowerCase()` → "hello" |
| `trim()` | Xóa khoảng trắng 2 đầu | `" Hi ".trim()` → "Hi" |
| `strip()` | Như trim nhưng Unicode-aware (Java 11+) | `" Hi ".strip()` → "Hi" |
| `replace(a, b)` | Thay thế | `"abc".replace("a", "x")` → "xbc" |
| `split(regex)` | Tách chuỗi | `"a,b,c".split(",")` → ["a","b","c"] |
| `join(sep, arr)` | Nối chuỗi | `String.join("-", "a", "b")` → "a-b" |
| `isBlank()` | Rỗng hoặc toàn khoảng trắng? (Java 11+) | `"  ".isBlank()` → true |
| `formatted()` | Format (Java 15+) | `"Hi %s".formatted("An")` → "Hi An" |

### 3. String vs StringBuilder

```java
// ❌ SAI: String + trong vòng lặp → tạo N objects → chậm
String result = "";
for (int i = 0; i < 10000; i++) {
    result += i;  // Mỗi += tạo String MỚI!
}

// ✅ ĐÚNG: StringBuilder — 1 object, nhanh O(n)
StringBuilder sb = new StringBuilder();
for (int i = 0; i < 10000; i++) {
    sb.append(i);
}
String result = sb.toString();
```

| Loại | Mutable? | Thread-safe? | Dùng khi |
|------|----------|-------------|----------|
| **String** | ❌ | ✅ | Chuỗi cố định, ít thay đổi |
| **StringBuilder** | ✅ | ❌ | Nối chuỗi trong vòng lặp (99% cases) |
| **StringBuffer** | ✅ | ✅ | Multi-thread (hiếm dùng) |

### 4. Text Block (Java 13+)

```java
String json = """
        {
            "name": "An",
            "age": 25
        }
        """;
```

---

## Phần C: Collections Framework

### 1. Bản đồ Collections

```
         Collection                          Map
         ├── List (có thứ tự, cho phép trùng)   ├── HashMap (nhanh, không thứ tự)
         │   ├── ArrayList ⭐                    ├── LinkedHashMap (giữ thứ tự insert)
         │   └── LinkedList                      ├── TreeMap (sorted theo key)
         ├── Set (không trùng)                   └── ConcurrentHashMap (thread-safe)
         │   ├── HashSet ⭐
         │   ├── LinkedHashSet
         │   └── TreeSet (sorted)
         └── Queue/Deque
             ├── PriorityQueue
             └── ArrayDeque ⭐
```

### 2. List — Danh sách có thứ tự

```java
// Tạo List
List<String> names = new ArrayList<>();           // Mutable
List<String> fixed = List.of("A", "B", "C");     // Immutable (Java 9+)
List<String> copy = new ArrayList<>(fixed);       // Mutable copy

// Thao tác cơ bản
names.add("An");                 // Thêm cuối
names.add(0, "Bình");           // Thêm vị trí 0
names.get(0);                    // Lấy phần tử
names.set(0, "Châu");           // Thay thế
names.remove(0);                 // Xóa theo index
names.remove("An");             // Xóa theo giá trị
names.size();                    // Kích thước
names.contains("An");           // Có chứa?
names.isEmpty();                 // Rỗng?

// Sắp xếp
Collections.sort(names);                          // Tự nhiên A-Z
names.sort(Comparator.reverseOrder());            // Giảm dần
names.sort(Comparator.comparing(String::length)); // Theo độ dài
```

### 3. Set — Không trùng lặp

```java
Set<String> uniqueNames = new HashSet<>();
uniqueNames.add("An");
uniqueNames.add("An");    // Không thêm được — đã tồn tại
uniqueNames.size();        // 1

// Set operations
Set<Integer> a = Set.of(1, 2, 3);
Set<Integer> b = Set.of(2, 3, 4);

Set<Integer> union = new HashSet<>(a);
union.addAll(b);              // Hợp: {1, 2, 3, 4}

Set<Integer> intersection = new HashSet<>(a);
intersection.retainAll(b);    // Giao: {2, 3}

Set<Integer> diff = new HashSet<>(a);
diff.removeAll(b);            // Hiệu: {1}
```

### 4. Map — Cặp Key-Value

```java
Map<String, Integer> scores = new HashMap<>();
scores.put("Toán", 9);
scores.put("Lý", 8);
scores.put("Toán", 10);       // Ghi đè — key "Toán" → 10

scores.get("Toán");            // 10
scores.getOrDefault("Hóa", 0); // 0 (không có → default)
scores.containsKey("Lý");     // true
scores.size();                 // 2
scores.remove("Lý");          // Xóa

// Duyệt Map
for (Map.Entry<String, Integer> entry : scores.entrySet()) {
    System.out.println(entry.getKey() + ": " + entry.getValue());
}
// Hoặc:
scores.forEach((k, v) -> System.out.println(k + ": " + v));

// Methods hữu ích
scores.putIfAbsent("Hóa", 7);               // Chỉ put nếu chưa có
scores.computeIfAbsent("Sinh", k -> 8);     // Tính value nếu chưa có
scores.merge("Toán", 1, Integer::sum);       // Toán: 10 + 1 = 11
```

### 5. Queue & Deque

```java
// Queue — FIFO (vào trước ra trước)
Queue<String> queue = new LinkedList<>();
queue.offer("A");    // Thêm cuối
queue.offer("B");
queue.poll();        // Lấy + xóa đầu → "A"
queue.peek();        // Xem đầu (không xóa) → "B"

// Deque — Hàng đợi 2 đầu (dùng làm Stack hoặc Queue)
Deque<Integer> stack = new ArrayDeque<>();
stack.push(1);       // Stack: thêm đầu
stack.push(2);
stack.pop();         // Stack: lấy + xóa đầu → 2
```

### 6. Chọn Collection nào? — Decision Guide

```
Cần lưu trữ gì?
│
├── Danh sách có thứ tự, cho phép trùng?
│   └── ArrayList ⭐ (99% cases)
│       └── LinkedList (chỉ khi insert/delete giữa list nhiều)
│
├── Tập hợp KHÔNG trùng lặp?
│   ├── Không cần thứ tự → HashSet ⭐
│   ├── Giữ thứ tự insert → LinkedHashSet
│   └── Tự sắp xếp → TreeSet
│
├── Cặp Key → Value?
│   ├── Không cần thứ tự → HashMap ⭐
│   ├── Giữ thứ tự insert → LinkedHashMap
│   ├── Sorted theo key → TreeMap
│   └── Multi-thread → ConcurrentHashMap
│
├── Hàng đợi FIFO?
│   └── ArrayDeque ⭐ (nhanh hơn LinkedList)
│
└── Stack LIFO?
    └── ArrayDeque ⭐ (KHÔNG dùng class Stack cũ)
```

### 7. Collections utility

```java
Collections.unmodifiableList(list);    // Tạo bản read-only
Collections.synchronizedList(list);    // Tạo bản thread-safe
Collections.singletonList("A");       // List 1 phần tử
Collections.emptyList();              // List rỗng immutable
Collections.frequency(list, "A");     // Đếm số lần xuất hiện
Collections.swap(list, 0, 1);         // Đổi chỗ 2 phần tử
```

---

## Bài tập

1. **Word Counter**: Đọc 1 câu, đếm số lần xuất hiện mỗi từ (dùng HashMap)
2. **Remove Duplicates**: Loại bỏ trùng lặp từ List nhưng giữ thứ tự (dùng LinkedHashSet)
3. **Custom Exception**: Tạo BankAccount với method withdraw(). Throw `InsufficientFundsException` nếu rút quá số dư

---

## Navigation

- [← Day 2: OOP](./day-2-oop.md)
- [Day 4: Generics + Lambda →](./day-4-generics-lambda.md)
