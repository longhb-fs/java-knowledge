# Day 15: CompletableFuture (Lập Trình Bất Đồng Bộ Nâng Cao)

## Mục tiêu hôm nay
- CompletableFuture là gì và tại sao cần dùng (thay vì Future)
- Tạo async task: runAsync, supplyAsync
- Chaining (chuỗi xử lý): thenApply, thenAccept, thenCompose
- Combining (kết hợp): thenCombine, allOf, anyOf
- Exception Handling: exceptionally, handle, whenComplete
- Các pattern thực tế: Parallel API Calls, Retry, Timeout

---

## 🤔 Tại sao cần CompletableFuture?

### Vấn đề với Future (Day 14)

```java
// ❌ Future hạn chế:
Future<String> future = executor.submit(() -> fetchData());

// 1. Phải BLOCK để lấy kết quả
String result = future.get();          // BLOCKING! Thread đứng đây chờ

// 2. Không thể CHAIN (nối) operations
// Muốn: fetch → transform → save → notify
// Phải: get() → transform → submit() → get() → submit() → get() 💀

// 3. Không thể combine (kết hợp) nhiều Future
// Muốn: chờ cả userFuture VÀ orderFuture rồi merge
// Phải: tự viết loop phức tạp

// 4. Không có exception handling tốt
// ExecutionException bao bọc exception thật → khó xử lý
```

```java
// ✅ CompletableFuture giải quyết TẤT CẢ:
CompletableFuture.supplyAsync(() -> fetchData())         // Chạy async
    .thenApply(data -> transform(data))                  // Chain: biến đổi
    .thenApply(transformed -> save(transformed))         // Chain: lưu
    .thenAccept(saved -> notify(saved))                  // Chain: thông báo
    .exceptionally(ex -> handleError(ex));               // Xử lý lỗi
// → KHÔNG blocking, code đọc như văn bản, xử lý lỗi tập trung
```

> **Ví dụ đời thường**: `Future` giống **đặt hàng online rồi ngồi chờ trước cửa** (blocking). `CompletableFuture` giống **đặt hàng online, để lại số điện thoại → shipper gọi khi xong → bạn tự do làm việc khác** (non-blocking callback).

```
Future vs CompletableFuture:

┌──────────────────────────┬────────────────────────────────────┐
│ Future                   │ CompletableFuture                  │
├──────────────────────────┼────────────────────────────────────┤
│ get() blocking           │ Non-blocking callbacks             │
│ Không chain được         │ thenApply, thenCompose chain       │
│ Không combine được       │ allOf, anyOf, thenCombine          │
│ Exception khó xử lý     │ exceptionally, handle              │
│ Không tự tạo được        │ Có thể tạo và complete thủ công   │
│ Không có timeout method  │ orTimeout, completeOnTimeout (9+)  │
├──────────────────────────┼────────────────────────────────────┤
│ ⚠️ Cơ bản              │ ✅ KHUYÊN DÙNG cho async           │
└──────────────────────────┴────────────────────────────────────┘
```

---

## 1. Tạo CompletableFuture

```java
import java.util.concurrent.CompletableFuture;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;

// === runAsync: Chạy task KHÔNG trả về kết quả (Runnable) ===
CompletableFuture<Void> cf1 = CompletableFuture.runAsync(() -> {
    System.out.println("Đang chạy trên: " + Thread.currentThread().getName());
    // Mặc định chạy trên ForkJoinPool.commonPool()
});

// === supplyAsync: Chạy task CÓ trả về kết quả (Supplier) ===
CompletableFuture<String> cf2 = CompletableFuture.supplyAsync(() -> {
    // Giả lập gọi API mất 1 giây
    try { Thread.sleep(1000); } catch (InterruptedException e) {}
    return "Dữ liệu từ API";
});

// === Với custom executor (khuyên dùng trong production) ===
ExecutorService executor = Executors.newFixedThreadPool(4);
CompletableFuture<String> cf3 = CompletableFuture.supplyAsync(
    () -> "Kết quả",
    executor                   // Chạy trên executor tùy chỉnh thay vì commonPool
);

// === Tạo CompletableFuture đã hoàn thành sẵn ===
CompletableFuture<String> completed = CompletableFuture.completedFuture("Done");
// Hữu ích cho testing hoặc khi đã có kết quả sẵn

// === Lấy kết quả ===
String result = cf2.get();             // Blocking (giống Future)
String result2 = cf2.join();           // Giống get() nhưng throw unchecked exception
                                       // → tiện hơn, không cần try-catch
```

---

## 2. Chaining - Chuỗi Xử Lý (Phần Quan Trọng Nhất!)

> **Ví dụ đời thường**: Giống **dây chuyền pha chế trà sữa**:
> Nhận order → Pha trà → Thêm sữa → Thêm topping → Giao cho khách
> Mỗi bước nhận kết quả bước trước → xử lý → truyền cho bước sau

### 3 method chaining cơ bản

```
┌──────────────────┬──────────────────┬──────────────────────────┐
│ Method           │ Input → Output   │ Giải thích               │
├──────────────────┼──────────────────┼──────────────────────────┤
│ thenApply()      │ T → U            │ Biến đổi kết quả        │
│ (giống map)      │ Nhận T, trả U    │ Function<T,U>           │
├──────────────────┼──────────────────┼──────────────────────────┤
│ thenAccept()     │ T → void         │ Tiêu thụ kết quả       │
│ (giống forEach)  │ Nhận T, không    │ Consumer<T>             │
│                  │ trả gì           │                          │
├──────────────────┼──────────────────┼──────────────────────────┤
│ thenRun()        │ void → void      │ Chạy sau khi xong       │
│                  │ Không nhận,      │ Runnable                 │
│                  │ không trả        │                          │
└──────────────────┴──────────────────┴──────────────────────────┘
```

```java
CompletableFuture<String> future = CompletableFuture.supplyAsync(() -> "hello");

// === thenApply: Biến đổi kết quả (T → U) ===
CompletableFuture<String> upper = future.thenApply(s -> s.toUpperCase());
// "hello" → "HELLO"

CompletableFuture<Integer> length = future.thenApply(s -> s.length());
// "hello" → 5

// === thenAccept: Tiêu thụ kết quả (T → void) ===
future.thenAccept(s -> System.out.println("Kết quả: " + s));
// In: "Kết quả: hello" (không trả về gì)

// === thenRun: Chạy sau khi xong (void → void) ===
future.thenRun(() -> System.out.println("Đã hoàn thành!"));
// In: "Đã hoàn thành!" (không nhận kết quả, không trả gì)

// === 🔥 CHAINING: Nối nhiều bước thành pipeline ===
CompletableFuture<String> result = CompletableFuture
    .supplyAsync(() -> "hello")                  // Bước 1: Tạo "hello"
    .thenApply(s -> s.toUpperCase())            // Bước 2: → "HELLO"
    .thenApply(s -> s + " WORLD")               // Bước 3: → "HELLO WORLD"
    .thenApply(s -> s + "!");                    // Bước 4: → "HELLO WORLD!"

System.out.println(result.join());              // "HELLO WORLD!"
```

### thenCompose - Chain các Future phụ thuộc (giống flatMap)

```java
// Khi bước sau CẦN kết quả bước trước để TẠO CompletableFuture mới

// ❌ thenApply → CompletableFuture<CompletableFuture<String>> (lồng!)
CompletableFuture<CompletableFuture<String>> nested =
    getUserId().thenApply(id -> fetchUserName(id));  // fetchUserName trả về CF<String>

// ✅ thenCompose → CompletableFuture<String> (phẳng!)
CompletableFuture<String> flat =
    getUserId().thenCompose(id -> fetchUserName(id)); // Tự "gỡ lồng" (flatMap)

// === Ví dụ thực tế: Gọi API tuần tự ===
CompletableFuture<OrderDetails> orderFlow =
    authenticate(credentials)                        // Bước 1: Đăng nhập → token
    .thenCompose(token -> fetchUser(token))          // Bước 2: Dùng token lấy user
    .thenCompose(user -> fetchOrders(user.getId()))  // Bước 3: Dùng userId lấy orders
    .thenCompose(orders -> enrichOrders(orders));    // Bước 4: Bổ sung chi tiết orders
// Mỗi bước cần kết quả bước trước → dùng thenCompose
```

```
thenApply vs thenCompose:

  thenApply: Khi function trả về GIÁ TRỊ THƯỜNG
  supplyAsync(() -> "hello")
      .thenApply(s -> s.toUpperCase())     // String → String
      // CompletableFuture<String>  ✅

  thenCompose: Khi function trả về COMPLETABLEFUTURE
  getUserId()
      .thenCompose(id -> fetchUser(id))    // id → CompletableFuture<User>
      // CompletableFuture<User>  ✅ (tự gỡ lồng)

  💡 Mẹo: thenApply = map(), thenCompose = flatMap()
  Nếu bạn hiểu Stream: map vs flatMap → y hệt!
```

---

## 3. Combining - Kết Hợp Nhiều Future

### 3.1. thenCombine - Kết hợp 2 Future

```java
CompletableFuture<String> nameFuture = CompletableFuture.supplyAsync(() -> {
    sleep(1000);
    return "John";
});

CompletableFuture<Integer> ageFuture = CompletableFuture.supplyAsync(() -> {
    sleep(800);
    return 25;
});

// Chạy SONG SONG, khi CẢ 2 xong → kết hợp kết quả
CompletableFuture<String> combined = nameFuture.thenCombine(
    ageFuture,
    (name, age) -> name + " (" + age + " tuổi)"    // Kết hợp 2 kết quả
);

System.out.println(combined.join());  // "John (25 tuổi)"
// Tổng thời gian: ~1 giây (chạy song song, không phải 1.8 giây)
```

### 3.2. allOf - Chờ TẤT CẢ xong

```java
CompletableFuture<String> cf1 = CompletableFuture.supplyAsync(() -> fetchUser());
CompletableFuture<String> cf2 = CompletableFuture.supplyAsync(() -> fetchOrders());
CompletableFuture<String> cf3 = CompletableFuture.supplyAsync(() -> fetchPayments());

// allOf: Chạy song song, chờ TẤT CẢ xong
CompletableFuture<Void> allDone = CompletableFuture.allOf(cf1, cf2, cf3);
// ⚠️ allOf trả về Void! Phải dùng join() từng future để lấy kết quả

allDone.thenRun(() -> {
    String user = cf1.join();       // Đã xong, join() trả về ngay
    String orders = cf2.join();
    String payments = cf3.join();
    System.out.println("Tất cả xong: " + user + ", " + orders + ", " + payments);
});
```

### 3.3. anyOf - Lấy kết quả ĐẦU TIÊN xong

```java
CompletableFuture<String> server1 = CompletableFuture.supplyAsync(() -> {
    sleep(3000);
    return "Kết quả từ Server 1";
});
CompletableFuture<String> server2 = CompletableFuture.supplyAsync(() -> {
    sleep(1000);
    return "Kết quả từ Server 2";   // Server 2 nhanh nhất
});
CompletableFuture<String> server3 = CompletableFuture.supplyAsync(() -> {
    sleep(2000);
    return "Kết quả từ Server 3";
});

// anyOf: Trả về kết quả ĐẦU TIÊN hoàn thành
CompletableFuture<Object> fastest = CompletableFuture.anyOf(server1, server2, server3);
System.out.println(fastest.join());  // "Kết quả từ Server 2" (1 giây)

// 💡 Dùng khi:
// → Gọi nhiều server, lấy response nhanh nhất
// → Fallback: thử cách A và cách B, lấy cách nào xong trước
```

---

## 4. Exception Handling (Xử Lý Lỗi)

### 3 cách xử lý exception

```
┌──────────────────┬───────────────────────────────────────────┐
│ Method           │ Giải thích                                │
├──────────────────┼───────────────────────────────────────────┤
│ exceptionally()  │ Chỉ xử lý khi có EXCEPTION               │
│                  │ Trả về giá trị mặc định (recovery)        │
├──────────────────┼───────────────────────────────────────────┤
│ handle()         │ Xử lý CẢ thành công VÀ thất bại          │
│                  │ Nhận (result, exception)                   │
├──────────────────┼───────────────────────────────────────────┤
│ whenComplete()   │ Giống handle nhưng KHÔNG thay đổi kết quả│
│                  │ Dùng cho logging, cleanup                 │
└──────────────────┴───────────────────────────────────────────┘
```

```java
CompletableFuture<String> future = CompletableFuture.supplyAsync(() -> {
    if (true) throw new RuntimeException("Lỗi kết nối!");
    return "Thành công";
});

// === exceptionally: Xử lý lỗi, trả về giá trị thay thế ===
CompletableFuture<String> recovered = future.exceptionally(ex -> {
    System.out.println("Lỗi: " + ex.getMessage());
    return "Giá trị mặc định";        // Recovery value
});
System.out.println(recovered.join());  // "Giá trị mặc định"

// === handle: Xử lý cả 2 trường hợp (thành công/thất bại) ===
CompletableFuture<String> handled = future.handle((result, ex) -> {
    if (ex != null) {
        return "Lỗi: " + ex.getMessage();    // Khi có exception
    }
    return "OK: " + result;                   // Khi thành công
});

// === whenComplete: Log/cleanup, KHÔNG thay đổi kết quả ===
future.whenComplete((result, ex) -> {
    if (ex != null) {
        System.out.println("FAILED: " + ex.getMessage());  // Log lỗi
    } else {
        System.out.println("SUCCESS: " + result);          // Log thành công
    }
    // ⚠️ Kết quả gốc KHÔNG bị thay đổi (exception vẫn propagate)
});
```

```
Exception handling flow:

  supplyAsync() ──► thenApply() ──► thenApply() ──► thenAccept()
       │                                                │
       │ Exception xảy ra ở đây?                        │
       │                                                │
       └───── skip ───── skip ───── skip ────► exceptionally()
                                                         │
                                                    Recovery value
                                                         │
                                                    tiếp tục pipeline

  💡 Exception sẽ "nhảy" qua tất cả thenApply/thenAccept
  → đến exceptionally/handle đầu tiên gặp
  → giống try-catch bao bọc toàn bộ pipeline
```

---

## 5. Async Variants (Biến Thể Bất Đồng Bộ)

```java
CompletableFuture<String> future = CompletableFuture.supplyAsync(() -> "hello");

// === Sync: Chạy trên CÙNG thread với bước trước ===
future.thenApply(s -> s.toUpperCase());            // Thread hoàn thành bước trước sẽ chạy tiếp

// === Async: Chạy trên thread KHÁC (từ ForkJoinPool) ===
future.thenApplyAsync(s -> s.toUpperCase());       // Submit vào pool → thread khác xử lý

// === Async với custom executor ===
ExecutorService myPool = Executors.newFixedThreadPool(4);
future.thenApplyAsync(s -> s.toUpperCase(), myPool); // Chạy trên myPool

// 💡 Khi nào dùng Async variant?
// thenApply:      Bước tiếp theo NHANH (vài ms) → chạy trên cùng thread OK
// thenApplyAsync: Bước tiếp theo CHẬM (I/O, tính toán nặng) → dùng thread riêng

// Tất cả method đều có Async variant:
// thenApplyAsync, thenAcceptAsync, thenRunAsync
// thenCombineAsync, thenComposeAsync
// handleAsync, whenCompleteAsync
```

---

## 6. Patterns Thực Tế

### Pattern 1: Parallel API Calls (Gọi nhiều API song song)

```java
// 🔥 Pattern phổ biến nhất: Gọi nhiều service song song → merge kết quả
public CompletableFuture<UserProfile> getUserProfile(String userId) {
    // 3 API calls chạy ĐỒNG THỜI (không chờ nhau)
    CompletableFuture<User> userFuture = CompletableFuture.supplyAsync(
        () -> userService.getUser(userId));

    CompletableFuture<List<Order>> ordersFuture = CompletableFuture.supplyAsync(
        () -> orderService.getOrders(userId));

    CompletableFuture<Preferences> prefsFuture = CompletableFuture.supplyAsync(
        () -> prefService.getPreferences(userId));

    // Chờ TẤT CẢ xong → combine kết quả
    return CompletableFuture.allOf(userFuture, ordersFuture, prefsFuture)
        .thenApply(v -> new UserProfile(
            userFuture.join(),             // Đã xong, join() trả về ngay
            ordersFuture.join(),
            prefsFuture.join()
        ));
}

// Nếu gọi tuần tự: 1s + 0.5s + 0.3s = 1.8s
// Gọi song song:   max(1s, 0.5s, 0.3s) = 1s  ← Nhanh hơn 80%!
```

### Pattern 2: Retry (Thử lại khi lỗi)

```java
// Retry tối đa N lần khi task thất bại
public <T> CompletableFuture<T> retryAsync(
        Supplier<CompletableFuture<T>> taskSupplier,
        int maxRetries) {

    return taskSupplier.get()                    // Chạy task lần đầu
        .exceptionallyCompose(ex -> {            // Nếu lỗi:
            if (maxRetries > 0) {
                System.out.println("Lỗi: " + ex.getMessage() +
                    ". Thử lại... (còn " + maxRetries + " lần)");
                return retryAsync(taskSupplier, maxRetries - 1);  // Thử lại
            }
            return CompletableFuture.failedFuture(ex);  // Hết lần thử → fail
        });
}

// Sử dụng:
CompletableFuture<String> result = retryAsync(
    () -> CompletableFuture.supplyAsync(() -> callUnstableAPI()),
    3    // Thử tối đa 3 lần
);
```

### Pattern 3: Timeout (Java 9+)

```java
CompletableFuture<String> future = CompletableFuture.supplyAsync(() -> {
    sleep(5000);        // Task chạy 5 giây
    return "Kết quả";
});

// === orTimeout: Ném exception nếu quá timeout ===
future.orTimeout(2, TimeUnit.SECONDS);
// Sau 2 giây → TimeoutException nếu chưa xong

// === completeOnTimeout: Trả về giá trị mặc định nếu quá timeout ===
future.completeOnTimeout("Timeout - giá trị mặc định", 2, TimeUnit.SECONDS);
// Sau 2 giây → trả "Timeout - giá trị mặc định" thay vì chờ tiếp

// 💡 Pattern thực tế:
CompletableFuture<String> apiCall = CompletableFuture.supplyAsync(() -> callExternalAPI())
    .completeOnTimeout("Cache value", 3, TimeUnit.SECONDS)   // Fallback nếu timeout
    .exceptionally(ex -> "Error: " + ex.getMessage());        // Fallback nếu lỗi
```

---

## 7. Sai Lầm Thường Gặp

### ❌ Sai lầm 1: Dùng get() thay vì join()

```java
// ❌ SAI: get() throw checked exceptions → phải try-catch
try {
    String result = future.get();
} catch (InterruptedException | ExecutionException e) {
    // Xử lý lỗi
}

// ✅ TỐT HƠN: join() throw unchecked exception → code sạch hơn
String result = future.join();

// 💡 Dùng get() khi cần timeout: future.get(5, TimeUnit.SECONDS)
// Dùng join() cho các trường hợp khác
```

### ❌ Sai lầm 2: Block trong async pipeline

```java
// ❌ SAI: Gọi join()/get() giữa chừng pipeline → BLOCK thread!
CompletableFuture<String> result = CompletableFuture.supplyAsync(() -> fetchA())
    .thenApply(a -> {
        String b = CompletableFuture.supplyAsync(() -> fetchB()).join(); // 💀 BLOCK!
        return a + b;
    });

// ✅ ĐÚNG: Dùng thenCombine hoặc thenCompose
CompletableFuture<String> result = CompletableFuture.supplyAsync(() -> fetchA())
    .thenCombine(
        CompletableFuture.supplyAsync(() -> fetchB()),   // Chạy song song
        (a, b) -> a + b                                   // Combine khi cả 2 xong
    );
```

### ❌ Sai lầm 3: Quên xử lý exception

```java
// ❌ SAI: Exception bị nuốt (swallowed) - không ai biết có lỗi!
CompletableFuture.supplyAsync(() -> riskyOperation())
    .thenApply(r -> transform(r))
    .thenAccept(r -> save(r));
// Nếu riskyOperation() throw exception → im lặng, không log gì!

// ✅ ĐÚNG: Luôn có exceptionally hoặc handle ở cuối chain
CompletableFuture.supplyAsync(() -> riskyOperation())
    .thenApply(r -> transform(r))
    .thenAccept(r -> save(r))
    .exceptionally(ex -> {
        System.err.println("Pipeline lỗi: " + ex.getMessage());
        // Log, alert, recovery...
        return null;
    });
```

### ❌ Sai lầm 4: Dùng ForkJoinPool.commonPool() cho I/O tasks

```java
// ❌ SAI: I/O tasks (HTTP call, DB query) trên commonPool
CompletableFuture.supplyAsync(() -> httpClient.get(url));   // Dùng commonPool
// commonPool có ít thread (= số CPU cores)
// I/O task block thread lâu → exhausted pool → toàn bộ app chậm!

// ✅ ĐÚNG: Dùng custom executor cho I/O tasks
ExecutorService ioPool = Executors.newFixedThreadPool(20);  // Nhiều thread cho I/O
CompletableFuture.supplyAsync(() -> httpClient.get(url), ioPool);

// 💡 Quy tắc:
// CPU-bound task (tính toán) → ForkJoinPool.commonPool() OK
// I/O-bound task (HTTP, DB, file) → Custom executor với nhiều thread
```

---

## 8. Tóm Tắt Cuối Ngày

| Khái niệm | Giải thích | Ví dụ |
|------------|-----------|-------|
| **runAsync()** | Task không trả về kết quả | `CF.runAsync(() -> log())` |
| **supplyAsync()** | Task có trả về kết quả | `CF.supplyAsync(() -> fetch())` |
| **thenApply()** | Biến đổi kết quả (map) | `.thenApply(String::toUpperCase)` |
| **thenAccept()** | Tiêu thụ kết quả | `.thenAccept(System.out::println)` |
| **thenRun()** | Chạy sau khi xong | `.thenRun(() -> cleanup())` |
| **thenCompose()** | Chain CF phụ thuộc (flatMap) | `.thenCompose(id -> fetchUser(id))` |
| **thenCombine()** | Kết hợp 2 CF song song | `cf1.thenCombine(cf2, (a,b) -> a+b)` |
| **allOf()** | Chờ tất cả CF xong | `CF.allOf(cf1, cf2, cf3)` |
| **anyOf()** | Lấy CF đầu tiên xong | `CF.anyOf(cf1, cf2, cf3)` |
| **exceptionally()** | Xử lý lỗi, trả recovery value | `.exceptionally(ex -> "default")` |
| **handle()** | Xử lý cả success và failure | `.handle((r, ex) -> ...)` |
| **whenComplete()** | Log/cleanup không đổi kết quả | `.whenComplete((r, ex) -> log())` |
| **orTimeout()** | Timeout → exception (Java 9+) | `.orTimeout(5, SECONDS)` |
| **completeOnTimeout()** | Timeout → default value (Java 9+) | `.completeOnTimeout("x", 5, SEC)` |
| **join()** | Lấy kết quả (unchecked exc) | `cf.join()` |

---

## 9. Câu Hỏi Phỏng Vấn Thường Gặp

### 🔥 Câu 1: CompletableFuture khác Future thế nào?
**Trả lời:**
`Future` chỉ hỗ trợ `get()` blocking. `CompletableFuture` hỗ trợ: (1) Non-blocking callbacks (thenApply, thenAccept), (2) Chaining operations thành pipeline, (3) Combining nhiều async tasks (allOf, thenCombine), (4) Exception handling (exceptionally, handle), (5) Manual completion, (6) Timeout (Java 9+). CompletableFuture basically là "Promise" trong Java, tương tự Promise trong JavaScript.

### 🔥 Câu 2: thenApply vs thenCompose - khi nào dùng?
**Trả lời:**
- `thenApply(Function<T,U>)`: Function trả về giá trị thường (U) → kết quả là `CF<U>`. Giống `map()` trong Stream
- `thenCompose(Function<T,CF<U>>)`: Function trả về `CompletableFuture<U>` → kết quả vẫn là `CF<U>` (tự gỡ lồng). Giống `flatMap()` trong Stream
- Dùng `thenCompose` khi bước tiếp theo là async operation trả về CF (gọi API, query DB). Dùng `thenApply` khi bước tiếp theo là transformation đồng bộ

### 🔥 Câu 3: allOf vs anyOf - khác nhau thế nào?
**Trả lời:**
- `allOf`: Chờ TẤT CẢ CompletableFuture hoàn thành. Trả về `CF<Void>`. Dùng khi cần tất cả kết quả để merge (fetch user + orders + payments)
- `anyOf`: Trả về ngay khi BẤT KỲ CF nào hoàn thành đầu tiên. Trả về `CF<Object>`. Dùng khi chỉ cần 1 kết quả (fastest server, first available)
- Cả 2 đều chạy tất cả CF song song

### 🔥 Câu 4: exceptionally vs handle vs whenComplete?
**Trả lời:**
- `exceptionally(Function<Throwable,T>)`: CHỈ chạy khi có exception. Trả về recovery value. Dùng cho: fallback/default
- `handle(BiFunction<T,Throwable,U>)`: LUÔN chạy (cả success và failure). Có thể thay đổi kết quả. Dùng cho: transform cả success và error
- `whenComplete(BiConsumer<T,Throwable>)`: LUÔN chạy nhưng KHÔNG thay đổi kết quả. Dùng cho: logging, cleanup, metrics

### 🔥 Câu 5: Tại sao không nên dùng ForkJoinPool.commonPool() cho I/O tasks?
**Trả lời:**
`commonPool` có số thread = CPU cores (thường 4-8). I/O tasks (HTTP, DB, file) block thread trong thời gian dài (chờ response). Nếu tất cả threads trong commonPool bị block bởi I/O → không còn thread cho CPU tasks → toàn bộ app bị ảnh hưởng. Giải pháp: tạo custom executor với nhiều threads dành riêng cho I/O tasks (`Executors.newFixedThreadPool(20)`)

### 🔥 Câu 6: CompletableFuture có thread-safe không?
**Trả lời:**
CÓ. CompletableFuture được thiết kế thread-safe. Nhiều thread có thể gọi `complete()`, `thenApply()`, etc. đồng thời mà không cần synchronization. Internally nó dùng CAS (Compare-And-Swap) và volatile fields. Tuy nhiên, code TRONG callbacks (lambda bạn viết) phải tự đảm bảo thread-safe nếu truy cập shared mutable state.

---

## Navigation

- [← Day 14: Concurrency](./day-14-concurrency.md)
- [Day 16: Reflection & Annotations →](./day-16-reflection-annotations.md)
