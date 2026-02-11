# Day 14: Concurrency (Lập Trình Đồng Thời Nâng Cao)

## Mục tiêu hôm nay
- ExecutorService và Thread Pool (quản lý thread chuyên nghiệp)
- Callable và Future (task có kết quả trả về)
- Các loại Thread Pool và khi nào dùng
- Lock API (ReentrantLock, ReadWriteLock) - thay thế synchronized
- Synchronizers (CountDownLatch, CyclicBarrier, Semaphore)

---

## 🤔 Tại sao cần Concurrency Framework?

### Vấn đề khi tự quản lý Thread

```java
// ❌ Tự tạo thread thủ công:
for (int i = 0; i < 1000; i++) {
    new Thread(() -> processRequest()).start();
}
// Vấn đề:
// 1. Tạo 1000 thread → tốn rất nhiều tài nguyên (mỗi thread ~1MB stack)
// 2. Không giới hạn → có thể crash hệ thống (OutOfMemoryError)
// 3. Không tái sử dụng thread → tốn chi phí tạo/hủy liên tục
// 4. Không quản lý được lifecycle (shutdown, timeout...)

// ✅ Dùng ExecutorService (Thread Pool):
ExecutorService executor = Executors.newFixedThreadPool(10);  // Tối đa 10 thread
for (int i = 0; i < 1000; i++) {
    executor.submit(() -> processRequest());   // 10 thread xử lý 1000 tasks
}
executor.shutdown();
// → Tái sử dụng thread, giới hạn tài nguyên, quản lý lifecycle
```

> **Ví dụ đời thường**: Thay vì thuê **1000 nhân viên tạm thời** mỗi khi có việc, bạn thuê **10 nhân viên full-time** (Thread Pool) và phân công việc cho họ. Xong việc này → làm việc khác. Hiệu quả hơn rất nhiều!

---

## 1. ExecutorService (Quản Lý Thread Chuyên Nghiệp)

```java
import java.util.concurrent.*;

// === Tạo ExecutorService ===
ExecutorService executor = Executors.newFixedThreadPool(4);   // Pool 4 threads

// === Gửi task không cần kết quả (Runnable) ===
executor.execute(() -> {
    System.out.println("Task chạy trên: " + Thread.currentThread().getName());
});

// === Gửi task CÓ kết quả (Callable → Future) ===
Future<String> future = executor.submit(() -> {
    Thread.sleep(1000);    // Giả lập xử lý
    return "Kết quả!";    // Trả về kết quả
});

String result = future.get();   // Chờ và lấy kết quả ("Kết quả!")

// === SHUTDOWN: Dừng executor đúng cách ===
executor.shutdown();             // Không nhận task mới, chờ task hiện tại xong
// HOẶC
executor.shutdownNow();          // Cố gắng dừng ngay TẤT CẢ task (interrupt)

// ✅ Pattern chuẩn để shutdown:
executor.shutdown();                                    // Bước 1: Không nhận task mới
if (!executor.awaitTermination(60, TimeUnit.SECONDS)) { // Bước 2: Chờ 60 giây
    executor.shutdownNow();                             // Bước 3: Force stop nếu quá lâu
}
```

```
ExecutorService hoạt động thế nào:

  ┌──────────────────────────────────────────────────────┐
  │                 THREAD POOL                          │
  │                                                      │
  │  submit(task) ──► ┌─────────────┐   ┌─────────┐     │
  │  submit(task) ──► │ Task Queue  │──►│Thread 1 │     │
  │  submit(task) ──► │ (Hàng đợi)  │──►│Thread 2 │     │
  │  submit(task) ──► │             │──►│Thread 3 │     │
  │  submit(task) ──► └─────────────┘──►│Thread 4 │     │
  │                                     └─────────┘     │
  │                                                      │
  │  5 tasks, 4 threads → 1 task phải chờ trong queue   │
  │  Thread xong task → lấy task tiếp theo từ queue     │
  └──────────────────────────────────────────────────────┘
```

---

## 2. Callable và Future (Task Có Kết Quả)

### Runnable vs Callable

```
┌──────────────────────────┬──────────────────────────────┐
│ Runnable                 │ Callable<V>                  │
├──────────────────────────┼──────────────────────────────┤
│ void run()               │ V call() throws Exception    │
│ KHÔNG trả về kết quả    │ CÓ trả về kết quả            │
│ KHÔNG throw checked exc  │ CÓ THỂ throw exception       │
│ Dùng khi "cứ chạy đi"  │ Dùng khi "cần biết kết quả" │
├──────────────────────────┼──────────────────────────────┤
│ () -> doSomething()      │ () -> { return result; }     │
└──────────────────────────┴──────────────────────────────┘
```

```java
// === Callable: task có kết quả ===
Callable<Integer> task = () -> {
    Thread.sleep(1000);        // Giả lập xử lý 1 giây
    return 42;                 // Trả về kết quả
};

ExecutorService executor = Executors.newSingleThreadExecutor();
Future<Integer> future = executor.submit(task);

// === Future: "Phiếu hẹn" lấy kết quả ===
future.isDone();               // Task đã xong chưa? (non-blocking)
future.isCancelled();          // Task đã bị hủy chưa?

// Lấy kết quả (BLOCKING - chờ cho đến khi có kết quả)
Integer result = future.get();                          // Chờ vô thời hạn
Integer result2 = future.get(5, TimeUnit.SECONDS);      // Chờ tối đa 5 giây
// → Nếu quá 5 giây: TimeoutException

// Hủy task
future.cancel(true);           // true = interrupt thread đang chạy task
                               // false = không interrupt, chỉ hủy nếu chưa bắt đầu
```

> **Ví dụ đời thường**: `Future` giống như **phiếu nhận đồ giặt**. Bạn gửi quần áo (submit task), nhận phiếu (Future). Sau đó quay lại lấy kết quả (future.get()). Nếu chưa giặt xong → bạn phải chờ.

### invokeAll và invokeAny

```java
List<Callable<String>> tasks = Arrays.asList(
    () -> { Thread.sleep(1000); return "Task 1 - chậm"; },
    () -> { Thread.sleep(500);  return "Task 2 - nhanh"; },
    () -> { Thread.sleep(1500); return "Task 3 - rất chậm"; }
);

ExecutorService executor = Executors.newFixedThreadPool(3);

// === invokeAll: Chạy TẤT CẢ, chờ TẤT CẢ xong ===
List<Future<String>> allResults = executor.invokeAll(tasks);
// Chờ đến khi task chậm nhất (Task 3) xong → trả về danh sách Future
for (Future<String> f : allResults) {
    System.out.println(f.get());       // Lấy kết quả từng task
}

// === invokeAny: Chạy TẤT CẢ, trả về kết quả ĐẦU TIÊN xong ===
String firstResult = executor.invokeAny(tasks);
// Task 2 xong sớm nhất → trả về "Task 2 - nhanh"
// Các task còn lại bị hủy (cancel)

// 💡 invokeAny hữu ích khi:
// → Gọi nhiều server, lấy kết quả từ server nhanh nhất
// → Thử nhiều phương pháp giải, lấy kết quả đầu tiên
```

---

## 3. Thread Pool Types (Các Loại Thread Pool)

```java
// === 1. FixedThreadPool: Số thread CỐ ĐỊNH ===
ExecutorService fixed = Executors.newFixedThreadPool(4);
// 4 thread chạy đồng thời, task thừa → chờ trong queue
// 💡 Dùng khi: Biết trước số lượng task đồng thời (web server, batch processing)

// === 2. CachedThreadPool: Tạo thread khi cần, tái sử dụng khi rảnh ===
ExecutorService cached = Executors.newCachedThreadPool();
// Thread rảnh 60s → bị hủy. Task mới + không có thread rảnh → tạo thread mới
// ⚠️ CẨN THẬN: Không giới hạn thread → có thể tạo quá nhiều!
// 💡 Dùng khi: Nhiều task ngắn, không biết trước số lượng

// === 3. SingleThreadExecutor: Đúng 1 thread ===
ExecutorService single = Executors.newSingleThreadExecutor();
// Tasks chạy TUẦN TỰ (1 task xong mới chạy task tiếp)
// 💡 Dùng khi: Cần đảm bảo thứ tự xử lý (ghi log, event queue)

// === 4. ScheduledThreadPool: Chạy theo lịch ===
ScheduledExecutorService scheduled = Executors.newScheduledThreadPool(2);

// Chạy SAU 5 giây
scheduled.schedule(() ->
    System.out.println("Chạy sau 5 giây!"),
    5, TimeUnit.SECONDS
);

// Chạy LẶP LẠI mỗi 1 giây (bắt đầu ngay)
scheduled.scheduleAtFixedRate(() ->
    System.out.println("Tick " + LocalTime.now()),
    0,                         // Initial delay (chờ ban đầu)
    1, TimeUnit.SECONDS        // Period (chu kỳ lặp)
);
// 💡 Dùng khi: Cron job, health check, periodic cleanup
```

```
Chọn Thread Pool nào?

┌─────────────────────────┬────────────────────────────────────────┐
│ Loại                    │ Khi nào dùng?                          │
├─────────────────────────┼────────────────────────────────────────┤
│ FixedThreadPool(n)      │ Biết trước workload, giới hạn tài     │
│                         │ nguyên. Web server, batch processing   │
├─────────────────────────┼────────────────────────────────────────┤
│ CachedThreadPool        │ Nhiều task ngắn, burst traffic.        │
│                         │ ⚠️ Cẩn thận không giới hạn thread     │
├─────────────────────────┼────────────────────────────────────────┤
│ SingleThreadExecutor    │ Cần xử lý tuần tự: ghi log, event    │
│                         │ queue, update UI                       │
├─────────────────────────┼────────────────────────────────────────┤
│ ScheduledThreadPool(n)  │ Task theo lịch: cron job, heartbeat,  │
│                         │ periodic cleanup                       │
├─────────────────────────┼────────────────────────────────────────┤
│ Custom ThreadPoolExecutor│ Cần kiểm soát chi tiết: queue size,  │
│                         │ rejection policy, core/max size        │
└─────────────────────────┴────────────────────────────────────────┘
```

### Custom ThreadPoolExecutor

```java
// Khi cần kiểm soát chi tiết hơn:
ThreadPoolExecutor custom = new ThreadPoolExecutor(
    2,                                     // Core pool size: 2 thread luôn sẵn sàng
    10,                                    // Max pool size: tối đa 10 thread khi busy
    60L, TimeUnit.SECONDS,                // Keep alive: thread thừa sống 60s rồi hủy
    new LinkedBlockingQueue<>(100),        // Queue: tối đa 100 task chờ
    new ThreadPoolExecutor.CallerRunsPolicy()  // Rejection policy khi queue đầy
);

// Rejection Policies (khi queue đầy + đã đạt max thread):
// AbortPolicy (mặc định): Throw RejectedExecutionException
// CallerRunsPolicy: Thread gửi task tự chạy task đó (giảm tốc)
// DiscardPolicy: Bỏ task mới, không báo lỗi
// DiscardOldestPolicy: Bỏ task cũ nhất trong queue, thêm task mới
```

---

## 4. Lock API (Khóa Nâng Cao)

> `synchronized` đơn giản nhưng hạn chế. `Lock` API mạnh hơn: tryLock (thử lock), timeout, fairness, đọc/ghi riêng biệt.

### 4.1. ReentrantLock - Thay thế synchronized

```java
import java.util.concurrent.locks.*;

ReentrantLock lock = new ReentrantLock();

// === Cách dùng cơ bản ===
lock.lock();           // Lấy lock (blocking nếu thread khác đang giữ)
try {
    // Critical section (vùng code cần bảo vệ)
    count++;
} finally {
    lock.unlock();     // ⚠️ LUÔN unlock trong finally! Tránh deadlock nếu exception
}

// === tryLock: Thử lấy lock KHÔNG blocking ===
if (lock.tryLock()) {                           // Thử lấy lock ngay lập tức
    try {
        // Lấy được lock → xử lý
    } finally {
        lock.unlock();
    }
} else {
    // Không lấy được → làm việc khác thay vì chờ
    System.out.println("Không lấy được lock, thử lại sau");
}

// === tryLock với timeout ===
if (lock.tryLock(1, TimeUnit.SECONDS)) {        // Thử chờ tối đa 1 giây
    try {
        // Lấy được lock → xử lý
    } finally {
        lock.unlock();
    }
} else {
    // Chờ 1 giây vẫn không lấy được → bỏ qua
}
```

```
ReentrantLock vs synchronized:

┌──────────────────────────┬───────────────────────────────────┐
│ synchronized             │ ReentrantLock                     │
├──────────────────────────┼───────────────────────────────────┤
│ Tự unlock khi ra block   │ Phải TỰ unlock (trong finally)   │
│ Không có tryLock          │ Có tryLock + timeout              │
│ Không fair                │ Có fair mode (FIFO)              │
│ Không có Condition        │ Có Condition (thay wait/notify)  │
│ Đơn giản, ít sai          │ Linh hoạt hơn, dễ sai hơn       │
├──────────────────────────┼───────────────────────────────────┤
│ ✅ Dùng khi đơn giản     │ ✅ Dùng khi cần tryLock, timeout │
│ đủ                        │ hoặc fair                        │
└──────────────────────────┴───────────────────────────────────┘
```

### 4.2. ReadWriteLock - Phân biệt đọc/ghi

> **Ví dụ đời thường**: **Thư viện** - nhiều người có thể ĐỌC sách cùng lúc, nhưng khi ai đó VIẾT (sửa sách) thì chỉ 1 người được viết và không ai đọc.

```java
ReadWriteLock rwLock = new ReentrantReadWriteLock();
Lock readLock = rwLock.readLock();
Lock writeLock = rwLock.writeLock();

// === Đọc: Nhiều thread đọc ĐỒNG THỜI (không block nhau) ===
public String read() {
    readLock.lock();
    try {
        return data;            // Nhiều thread đọc song song OK
    } finally {
        readLock.unlock();
    }
}

// === Ghi: Chỉ 1 thread ghi, block tất cả đọc ===
public void write(String newData) {
    writeLock.lock();
    try {
        data = newData;         // Chỉ 1 thread ghi tại 1 thời điểm
    } finally {                 // Các thread đọc phải chờ ghi xong
        writeLock.unlock();
    }
}

// 💡 Khi nào dùng ReadWriteLock?
// → Khi ĐỌC NHIỀU, GHI ÍT (cache, config, shared data)
// → Hiệu quả hơn synchronized vì cho phép đọc song song
```

---

## 5. Synchronizers (Bộ Đồng Bộ Hóa)

### 5.1. CountDownLatch - "Đếm ngược"

> **Ví dụ đời thường**: Giống như **đếm ngược phóng tên lửa**: 3... 2... 1... Phóng! Phải chờ tất cả điều kiện sẵn sàng mới bắt đầu.

```java
import java.util.concurrent.CountDownLatch;

// Tạo latch đếm ngược từ 3
CountDownLatch latch = new CountDownLatch(3);

// 3 worker thread, mỗi thread xong → countDown()
for (int i = 0; i < 3; i++) {
    final int id = i;
    new Thread(() -> {
        System.out.println("Service " + id + " đang khởi động...");
        try { Thread.sleep(1000 * (id + 1)); } catch (InterruptedException e) {}
        System.out.println("Service " + id + " sẵn sàng! ✅");
        latch.countDown();     // Giảm đếm: 3→2→1→0
    }).start();
}

// Main thread chờ tất cả service khởi động
System.out.println("Chờ tất cả services...");
latch.await();                 // BLOCK cho đến khi count = 0
System.out.println("Tất cả services đã sẵn sàng! 🚀 Bắt đầu xử lý!");

// Output:
// Chờ tất cả services...
// Service 0 đang khởi động...
// Service 1 đang khởi động...
// Service 2 đang khởi động...
// Service 0 sẵn sàng! ✅
// Service 1 sẵn sàng! ✅
// Service 2 sẵn sàng! ✅
// Tất cả services đã sẵn sàng! 🚀 Bắt đầu xử lý!

// ⚠️ CountDownLatch dùng 1 LẦN, không reset được!
```

### 5.2. CyclicBarrier - "Điểm tập kết"

> **Ví dụ đời thường**: Giống như **tour du lịch nhóm** - tất cả phải đến điểm tập kết rồi mới đi tiếp cùng nhau. Và có thể tập kết nhiều lần (cyclic = lặp lại).

```java
import java.util.concurrent.CyclicBarrier;

// Barrier cho 3 thread, khi tất cả đến → chạy action
CyclicBarrier barrier = new CyclicBarrier(3, () -> {
    System.out.println("=== Tất cả đã đến! Tiếp tục phase tiếp theo ===");
});

for (int i = 0; i < 3; i++) {
    final int id = i;
    new Thread(() -> {
        try {
            System.out.println("Worker " + id + " xong phase 1");
            barrier.await();     // Chờ tất cả đến điểm tập kết 1

            System.out.println("Worker " + id + " xong phase 2");
            barrier.await();     // Chờ tất cả đến điểm tập kết 2 (REUSABLE!)
        } catch (Exception e) {}
    }).start();
}

// 💡 CyclicBarrier vs CountDownLatch:
// CountDownLatch: Dùng 1 lần. Thread A chờ các thread B,C,D xong
// CyclicBarrier: Dùng lại được. Các thread chờ LẪN NHAU tại 1 điểm
```

### 5.3. Semaphore - "Giới hạn vé vào"

> **Ví dụ đời thường**: Giống như **bãi đỗ xe** có 3 chỗ. Khi đầy → xe mới phải chờ. Xe ra → có chỗ → xe mới vào.

```java
import java.util.concurrent.Semaphore;

// Tạo Semaphore với 3 permits (3 "vé vào")
Semaphore semaphore = new Semaphore(3);

// 10 thread cùng muốn vào, nhưng chỉ 3 thread vào đồng thời
for (int i = 0; i < 10; i++) {
    final int id = i;
    new Thread(() -> {
        try {
            semaphore.acquire();         // Lấy 1 permit (nếu hết → chờ)
            System.out.println("Thread " + id + " vào vùng giới hạn");
            Thread.sleep(2000);          // Giả lập xử lý
            System.out.println("Thread " + id + " ra khỏi vùng giới hạn");
        } catch (InterruptedException e) {
        } finally {
            semaphore.release();         // Trả lại permit
        }
    }).start();
}

// 💡 Semaphore dùng khi:
// → Giới hạn số kết nối database đồng thời
// → Giới hạn concurrent requests đến external API
// → Rate limiting
```

```
Tóm tắt Synchronizers:

┌──────────────────┬──────────────────────────────────────────┐
│ CountDownLatch   │ "Chờ N sự kiện xảy ra rồi mới tiếp"    │
│                  │ Dùng 1 lần. Ví dụ: chờ services khởi    │
│                  │ động xong                                │
├──────────────────┼──────────────────────────────────────────┤
│ CyclicBarrier    │ "Tất cả chờ nhau tại điểm tập kết"     │
│                  │ Dùng lại được. Ví dụ: xử lý theo phases │
├──────────────────┼──────────────────────────────────────────┤
│ Semaphore        │ "Giới hạn N thread vào cùng lúc"        │
│                  │ Ví dụ: connection pool, rate limiting    │
└──────────────────┴──────────────────────────────────────────┘
```

---

## 6. Ví Dụ Thực Tế

### Ví dụ: Hệ thống xử lý đơn hàng song song

```java
public class OrderProcessor {
    private final ExecutorService executor = Executors.newFixedThreadPool(5);

    // Xử lý nhiều đơn hàng đồng thời
    public List<OrderResult> processOrders(List<Order> orders) throws InterruptedException {
        // Chuyển mỗi đơn hàng thành 1 Callable task
        List<Callable<OrderResult>> tasks = orders.stream()
            .map(order -> (Callable<OrderResult>) () -> {
                System.out.println("Đang xử lý đơn: " + order.getId());
                validateOrder(order);          // Kiểm tra đơn hàng
                calculateShipping(order);      // Tính phí vận chuyển
                processPayment(order);         // Xử lý thanh toán
                return new OrderResult(order.getId(), "SUCCESS");
            })
            .collect(Collectors.toList());

        // Chạy tất cả song song, chờ tất cả xong
        List<Future<OrderResult>> futures = executor.invokeAll(tasks);

        // Thu thập kết quả
        List<OrderResult> results = new ArrayList<>();
        for (Future<OrderResult> future : futures) {
            try {
                results.add(future.get());
            } catch (ExecutionException e) {
                results.add(new OrderResult("UNKNOWN", "FAILED: " + e.getMessage()));
            }
        }
        return results;
    }

    // Shutdown đúng cách
    public void shutdown() {
        executor.shutdown();
        try {
            if (!executor.awaitTermination(30, TimeUnit.SECONDS)) {
                executor.shutdownNow();
            }
        } catch (InterruptedException e) {
            executor.shutdownNow();
        }
    }
}
```

---

## 7. Sai Lầm Thường Gặp

### ❌ Sai lầm 1: Quên shutdown ExecutorService

```java
// ❌ SAI: Không shutdown → thread pool sống mãi → app không tắt được!
ExecutorService executor = Executors.newFixedThreadPool(4);
executor.submit(() -> doWork());
// App kết thúc nhưng JVM KHÔNG tắt vì executor vẫn chạy!

// ✅ ĐÚNG: Luôn shutdown trong finally hoặc dùng try-with-resources (Java 19+)
ExecutorService executor = Executors.newFixedThreadPool(4);
try {
    executor.submit(() -> doWork());
} finally {
    executor.shutdown();
}
```

### ❌ Sai lầm 2: Quên unlock trong Lock

```java
// ❌ SAI: Exception xảy ra → KHÔNG BAO GIỜ unlock → deadlock!
lock.lock();
doSomething();         // Nếu throw exception ở đây...
lock.unlock();         // ...dòng này KHÔNG chạy → lock mãi mãi!

// ✅ ĐÚNG: LUÔN unlock trong finally
lock.lock();
try {
    doSomething();
} finally {
    lock.unlock();     // Luôn chạy, kể cả khi có exception
}
```

### ❌ Sai lầm 3: Dùng CachedThreadPool cho task nhiều/nặng

```java
// ❌ SAI: CachedThreadPool tạo thread không giới hạn!
ExecutorService executor = Executors.newCachedThreadPool();
for (int i = 0; i < 100000; i++) {
    executor.submit(() -> heavyComputation());  // Có thể tạo 100000 thread → OOM!
}

// ✅ ĐÚNG: Dùng FixedThreadPool để giới hạn
ExecutorService executor = Executors.newFixedThreadPool(
    Runtime.getRuntime().availableProcessors()   // Số CPU cores
);
```

### ❌ Sai lầm 4: future.get() không có timeout

```java
// ❌ SAI: get() không timeout → chờ MÃI MÃI nếu task bị treo!
Future<String> future = executor.submit(() -> infiniteLoop());
String result = future.get();    // 💀 Block vĩnh viễn

// ✅ ĐÚNG: Luôn đặt timeout
try {
    String result = future.get(30, TimeUnit.SECONDS);    // Chờ tối đa 30 giây
} catch (TimeoutException e) {
    future.cancel(true);         // Hủy task nếu quá lâu
    System.out.println("Task timeout, đã hủy");
}
```

---

## 8. Tóm Tắt Cuối Ngày

| Khái niệm | Giải thích | Ví dụ |
|------------|-----------|-------|
| **ExecutorService** | Quản lý Thread Pool chuyên nghiệp | `Executors.newFixedThreadPool(4)` |
| **execute()** | Gửi Runnable (không cần kết quả) | `executor.execute(runnable)` |
| **submit()** | Gửi Callable → nhận Future | `Future<T> f = executor.submit(callable)` |
| **Future** | "Phiếu hẹn" lấy kết quả từ task | `future.get()`, `future.isDone()` |
| **invokeAll()** | Chạy tất cả, chờ tất cả xong | Batch processing |
| **invokeAny()** | Chạy tất cả, trả về đầu tiên xong | Fastest response |
| **FixedThreadPool** | N thread cố định | Biết trước workload |
| **CachedThreadPool** | Tạo khi cần, tái sử dụng | Nhiều task ngắn |
| **ScheduledThreadPool** | Chạy theo lịch | Cron job, heartbeat |
| **ReentrantLock** | Lock nâng cao (tryLock, timeout) | Thay thế synchronized |
| **ReadWriteLock** | Đọc song song, ghi exclusive | Cache, shared data |
| **CountDownLatch** | Chờ N sự kiện xảy ra | Chờ services khởi động |
| **CyclicBarrier** | Chờ nhau tại điểm tập kết | Xử lý theo phases |
| **Semaphore** | Giới hạn N thread vào cùng lúc | Connection pool |

---

## 9. Câu Hỏi Phỏng Vấn Thường Gặp

### 🔥 Câu 1: Tại sao dùng Thread Pool thay vì tạo Thread trực tiếp?
**Trả lời:**
Tạo thread trực tiếp có 3 vấn đề: (1) Tốn tài nguyên - mỗi thread ~1MB stack, (2) Không giới hạn - có thể tạo quá nhiều thread gây OOM, (3) Tốn chi phí tạo/hủy liên tục. Thread Pool giải quyết bằng cách: tái sử dụng thread, giới hạn số lượng, quản lý lifecycle (shutdown, timeout). Cải thiện hiệu suất 10-100x.

### 🔥 Câu 2: FixedThreadPool vs CachedThreadPool - khi nào dùng?
**Trả lời:**
- `FixedThreadPool`: Số thread cố định. Dùng khi biết trước workload, cần giới hạn tài nguyên. Ví dụ: web server xử lý request, batch processing
- `CachedThreadPool`: Tạo thread khi cần, hủy khi rảnh 60s. Dùng khi nhiều task ngắn, workload không đoán trước. **Cẩn thận**: không giới hạn thread → có thể OOM với workload lớn

### 🔥 Câu 3: ReentrantLock hơn synchronized ở điểm nào?
**Trả lời:**
ReentrantLock cung cấp: (1) `tryLock()` - thử lấy lock không blocking, (2) `tryLock(timeout)` - thử lấy lock có timeout, (3) Fair mode - thread chờ lâu nhất được ưu tiên, (4) Multiple Condition objects - thay thế wait/notify linh hoạt hơn, (5) `lockInterruptibly()` - có thể interrupt thread đang chờ lock. Trade-off: phải tự unlock trong finally (dễ quên → deadlock)

### 🔥 Câu 4: CountDownLatch khác CyclicBarrier thế nào?
**Trả lời:**
- `CountDownLatch`: Dùng 1 lần, không reset. Thread A chờ các thread B,C,D hoàn thành. countDown() và await() gọi ở KHÁC thread. Ví dụ: main chờ services khởi động
- `CyclicBarrier`: Dùng lại được (cyclic). Các thread chờ LẪN NHAU tại 1 điểm. Tất cả gọi await(). Ví dụ: parallel computation theo phases

### 🔥 Câu 5: Semaphore dùng để làm gì?
**Trả lời:**
Semaphore giới hạn số thread truy cập resource đồng thời. Có N permits, mỗi thread acquire 1 permit (nếu hết → chờ), xong thì release. Dùng cho: connection pool (giới hạn DB connections), rate limiting (giới hạn API requests/s), resource pool (giới hạn file handles). Khác với Lock: Lock cho phép 1 thread, Semaphore cho phép N thread.

### 🔥 Câu 6: Callable khác Runnable thế nào?
**Trả lời:**
- `Runnable`: `void run()` - không trả về kết quả, không throw checked exception
- `Callable<V>`: `V call() throws Exception` - trả về kết quả kiểu V, có thể throw exception
- Callable dùng với `executor.submit()` → nhận `Future<V>` → gọi `future.get()` lấy kết quả
- Runnable phù hợp cho fire-and-forget task, Callable cho task cần kết quả

---

## Navigation

- [← Day 13: Multithreading](./day-13-multithreading-basics.md)
- [Day 15: CompletableFuture →](./day-15-completable-future.md)
