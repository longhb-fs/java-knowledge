# Day 7: CompletableFuture + Design Patterns + JVM Overview

> Gộp từ bản 19 ngày: Day 15 (CompletableFuture) + Day 17 (Design Patterns) + Day 16,18,19 (Reflection, Memory, JVM)
> 📖 Đọc sâu: [day-15](../java-fundamentals/day-15-completable-future.md) | [day-17](../java-fundamentals/day-17-design-patterns.md) | [day-18](../java-fundamentals/day-18-memory-gc.md) | [day-19](../java-fundamentals/day-19-jvm-internals.md)

---

## Phần A: CompletableFuture (Lập trình bất đồng bộ)

### 1. Tại sao cần?

```
Future (cũ):     future.get() → BLOCK thread → chờ đợi → lãng phí
CompletableFuture: .thenApply() → callback → KHÔNG block → chain được
```

### 2. Tạo CompletableFuture

```java
// Chạy task async, CÓ kết quả
CompletableFuture<String> cf = CompletableFuture.supplyAsync(() -> {
    // Chạy trên thread khác (ForkJoinPool.commonPool())
    return callExternalAPI();
});

// Chạy task async, KHÔNG kết quả
CompletableFuture<Void> cf2 = CompletableFuture.runAsync(() -> {
    sendEmail();
});

// Tạo từ giá trị sẵn
CompletableFuture<String> ready = CompletableFuture.completedFuture("Done");
```

### 3. Chaining — Nối chuỗi xử lý

| Method | Input → Output | Ví dụ |
|--------|---------------|-------|
| `thenApply(Function)` | T → R (có kết quả) | `.thenApply(s -> s.toUpperCase())` |
| `thenAccept(Consumer)` | T → void (xử lý) | `.thenAccept(System.out::println)` |
| `thenRun(Runnable)` | void → void | `.thenRun(() -> log("Done"))` |
| `thenCompose(Function)` | T → CF\<R\> (flatMap) | `.thenCompose(id -> fetchUser(id))` |

```java
CompletableFuture.supplyAsync(() -> fetchUserId("an@email.com"))  // Bước 1: lấy userId
    .thenApply(id -> fetchUserProfile(id))    // Bước 2: lấy profile (đồng bộ)
    .thenCompose(id -> fetchUserAsync(id))    // Bước 2b: lấy profile (bất đồng bộ)
    .thenAccept(user -> System.out.println("Got: " + user))  // Bước 3: xử lý
    .exceptionally(ex -> { System.err.println("Error: " + ex); return null; });
```

💡 `thenApply` vs `thenCompose`:
- `thenApply` = `map` — Function trả về **giá trị thường** → wrap vào CF
- `thenCompose` = `flatMap` — Function trả về **CompletableFuture** → không wrap lồng

### 4. Kết hợp nhiều CF

```java
CompletableFuture<String> cf1 = CompletableFuture.supplyAsync(() -> fetchFromAPI1());
CompletableFuture<String> cf2 = CompletableFuture.supplyAsync(() -> fetchFromAPI2());

// thenCombine: Kết hợp 2 kết quả
CompletableFuture<String> combined = cf1.thenCombine(cf2,
    (result1, result2) -> result1 + " + " + result2);

// allOf: Chờ TẤT CẢ hoàn thành
CompletableFuture<Void> all = CompletableFuture.allOf(cf1, cf2);
all.thenRun(() -> System.out.println("All done!"));

// anyOf: Lấy kết quả ĐẦU TIÊN hoàn thành
CompletableFuture<Object> fastest = CompletableFuture.anyOf(cf1, cf2);
```

### 5. Exception Handling

```java
CompletableFuture.supplyAsync(() -> riskyOperation())
    .thenApply(result -> process(result))
    .exceptionally(ex -> {                    // Catch exception
        System.err.println("Error: " + ex.getMessage());
        return "default";                     // Giá trị mặc định
    })
    .thenAccept(System.out::println);

// handle: Xử lý CẢ success và error
.handle((result, ex) -> {
    if (ex != null) return "Error: " + ex.getMessage();
    return "Success: " + result;
});
```

### 6. Pattern: Parallel API Calls

```java
// Gọi 3 API song song, tổng hợp kết quả
public UserDashboard loadDashboard(String userId) {
    var profileFuture = CompletableFuture.supplyAsync(() -> fetchProfile(userId));
    var ordersFuture  = CompletableFuture.supplyAsync(() -> fetchOrders(userId));
    var settingsFuture = CompletableFuture.supplyAsync(() -> fetchSettings(userId));

    // Chờ tất cả xong rồi tổng hợp
    CompletableFuture.allOf(profileFuture, ordersFuture, settingsFuture).join();

    return new UserDashboard(
        profileFuture.join(),   // join() = get() nhưng throw unchecked exception
        ordersFuture.join(),
        settingsFuture.join()
    );
}
// 3 API chạy song song → thời gian = max(api1, api2, api3) thay vì api1 + api2 + api3
```

---

## Phần B: Design Patterns (Mẫu thiết kế phổ biến)

### 1. Creational Patterns — Tạo object

#### Singleton — Chỉ 1 instance duy nhất

```java
// Cách tốt nhất: Enum Singleton
public enum DatabaseConnection {
    INSTANCE;

    public void query(String sql) { /* ... */ }
}
// Sử dụng: DatabaseConnection.INSTANCE.query("SELECT ...");
// Thread-safe, chống serialization, chống reflection — miễn phí!
```

#### Builder — Tạo object phức tạp từng bước

```java
// Thay vì constructor 10 tham số → Builder pattern
User user = User.builder()
    .name("An")
    .email("an@email.com")
    .age(25)
    .role("ADMIN")
    .build();

// Nếu dùng Lombok: @Builder trên class → tự generate code trên
```

#### Factory — Tạo object mà không lộ logic khởi tạo

```java
public class NotificationFactory {
    public static Notification create(String type) {
        return switch (type) {
            case "email" -> new EmailNotification();
            case "sms"   -> new SmsNotification();
            case "push"  -> new PushNotification();
            default      -> throw new IllegalArgumentException("Unknown: " + type);
        };
    }
}
// Notification n = NotificationFactory.create("email");
```

### 2. Structural Patterns — Cấu trúc

#### Adapter — Chuyển đổi interface

```java
// Interface cũ
class OldPaymentGateway {
    void makePayment(String xml) { /* xử lý XML */ }
}

// Interface mới
interface PaymentService {
    void pay(Map<String, Object> data);
}

// Adapter: "dịch" interface mới → interface cũ
class PaymentAdapter implements PaymentService {
    private OldPaymentGateway gateway = new OldPaymentGateway();

    @Override
    public void pay(Map<String, Object> data) {
        String xml = convertToXml(data);  // Chuyển đổi format
        gateway.makePayment(xml);
    }
}
```

#### Decorator — Thêm chức năng mà không sửa class gốc

```java
// Base
interface Logger { void log(String msg); }
class ConsoleLogger implements Logger {
    public void log(String msg) { System.out.println(msg); }
}

// Decorator: thêm timestamp
class TimestampLogger implements Logger {
    private Logger inner;
    TimestampLogger(Logger inner) { this.inner = inner; }

    public void log(String msg) {
        inner.log("[" + LocalDateTime.now() + "] " + msg);  // Thêm chức năng
    }
}

// Sử dụng: xếp chồng decorators
Logger logger = new TimestampLogger(new ConsoleLogger());
logger.log("Hello");  // [2026-02-09T14:30] Hello
```

### 3. Behavioral Patterns — Hành vi

#### Strategy — Chọn thuật toán lúc runtime

```java
// Với Lambda → Strategy pattern siêu ngắn gọn
public class Sorter {
    public <T> void sort(List<T> list, Comparator<T> strategy) {
        list.sort(strategy);
    }
}

Sorter sorter = new Sorter();
sorter.sort(users, Comparator.comparing(User::getName));    // Sắp xếp theo tên
sorter.sort(users, Comparator.comparing(User::getAge));     // Sắp xếp theo tuổi
sorter.sort(users, (a, b) -> b.getSalary().compareTo(a.getSalary())); // Theo lương giảm
```

#### Observer — Subscribe/Notify

```java
// Java có sẵn: PropertyChangeSupport
class UserService {
    private PropertyChangeSupport pcs = new PropertyChangeSupport(this);

    public void addListener(PropertyChangeListener l) { pcs.addPropertyChangeListener(l); }

    public void createUser(String name) {
        // ... tạo user ...
        pcs.firePropertyChange("userCreated", null, name);  // Notify tất cả listeners
    }
}

UserService service = new UserService();
service.addListener(evt -> System.out.println("User created: " + evt.getNewValue()));
service.addListener(evt -> sendWelcomeEmail(evt.getNewValue()));
service.createUser("An");
```

### 4. Cheat Sheet — Khi nào dùng pattern nào?

```
Cần tạo object?
├── Chỉ 1 instance → Singleton (Enum)
├── Object phức tạp, nhiều config → Builder
└── Tạo object theo điều kiện → Factory

Cần mở rộng chức năng?
├── Interface không tương thích → Adapter
└── Thêm behavior mà không sửa code gốc → Decorator

Cần thay đổi hành vi?
├── Chọn thuật toán lúc runtime → Strategy (Lambda!)
├── Notify khi state thay đổi → Observer
└── Định nghĩa skeleton, subclass tùy chỉnh → Template Method
```

---

## Phần C: JVM Overview (Kiến thức tổng quan)

### 1. Cách Java code chạy

```
.java → javac → .class (bytecode) → JVM → Native code
                                     │
                    ┌────────────────┤
                    │                │
              Interpreter       JIT Compiler
              (chậm, chạy ngay)  (nhanh, biên dịch "hot code")
```

### 2. Memory Model

```
JVM Memory
├── Heap (chia sẻ giữa threads)
│   ├── Young Gen → Minor GC (nhanh, thường xuyên)
│   └── Old Gen   → Major GC (chậm, ít khi)
│
├── Stack (mỗi thread riêng)
│   └── Local vars, method frames
│
└── Metaspace (class metadata)
```

| Vùng | Chứa gì | Lỗi khi đầy |
|------|---------|-------------|
| **Heap** | Objects, arrays | `OutOfMemoryError` |
| **Stack** | Local vars, method calls | `StackOverflowError` |
| **Metaspace** | Class info | `OutOfMemoryError: Metaspace` |

### 3. Garbage Collection (GC)

```
Object mới → Eden → Minor GC → Survivor → (sống đủ lâu) → Old Gen → Major GC

GC Roots (những gì KHÔNG bị dọn):
├── Biến local trên Stack
├── Static fields
├── Active threads
└── JNI references

Object không ai trỏ tới → UNREACHABLE → bị GC dọn
```

| GC | Đặc điểm | Dùng khi |
|----|----------|----------|
| **G1 GC** ⭐ | Mặc định Java 9+, đa năng | Hầu hết app |
| **ZGC** | Pause <10ms | App cần low latency |
| **Parallel GC** | Throughput cao | Batch processing |

### 4. JVM Options hay dùng

```bash
-Xms2g -Xmx2g                    # Heap min/max (nên bằng nhau)
-XX:+UseG1GC                     # Chọn GC (mặc định)
-XX:+HeapDumpOnOutOfMemoryError   # Tạo dump khi OOM (BẮT BUỘC production)
-XX:HeapDumpPath=/var/log/dumps/
-Xlog:gc*:file=gc.log            # Log GC
```

### 5. Memory Leaks — 4 nguyên nhân chính

| Nguyên nhân | Ví dụ | Fix |
|-------------|-------|-----|
| Static collection không clear | `static List<Object> cache` | Dùng cache có expiry/size limit |
| Quên đóng resources | `new FileInputStream()` | try-with-resources |
| Inner class giữ ref đến outer | Anonymous inner class | Static inner class hoặc Lambda |
| HashMap key thay đổi hashCode | Mutable key | Dùng immutable key (String, Integer) |

### 6. Reference Types — Tóm tắt

| Loại | Bị GC khi | Dùng cho |
|------|----------|----------|
| **Strong** (mặc định) | Không bao giờ (khi còn ref) | Sử dụng bình thường |
| **Soft** | Sắp hết memory | Cache |
| **Weak** | GC kế tiếp | WeakHashMap, listeners |
| **Phantom** | Sau finalize | Cleanup tracking |

---

## Bài tập

1. **Async Pipeline**: Dùng CompletableFuture: fetchUser → fetchOrders → calculateTotal → printResult. Thêm error handling.
2. **Builder Pattern**: Tạo `HttpRequest.Builder` cho class HttpRequest (method, url, headers, body)
3. **Memory Monitor**: Viết utility class in thông tin Heap memory (used/free/max) dùng `Runtime.getRuntime()`

---

## 🎓 Hoàn thành Crash Course!

Bạn đã học xong Java Fundamentals trong 7 ngày. Bước tiếp theo:

| Chủ đề | Tài nguyên |
|--------|-----------|
| Spring Boot | Bắt đầu xây REST API |
| JUnit + Mockito | Testing |
| Maven / Gradle | Build tools |
| JPA / Hibernate | Database ORM |
| Docker | Containerization |

📖 Muốn đọc sâu hơn → Quay lại [bản 19 ngày](../java-fundamentals/00-overview.md)

---

## Navigation

- [← Day 6: Threading](./day-6-threading.md)
- [Overview →](./00-overview.md)
