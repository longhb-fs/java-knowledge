# Day 6: Multithreading + Concurrency

> Gộp từ bản 19 ngày: Day 13 (Multithreading) + Day 14 (Concurrency)
> 📖 Đọc sâu: [day-13](../java-fundamentals/day-13-multithreading-basics.md) | [day-14](../java-fundamentals/day-14-concurrency.md)

---

## Phần A: Multithreading Basics

### 1. Thread là gì?

```
Process (Tiến trình) = 1 chương trình đang chạy (VD: Chrome, IntelliJ)
Thread (Luồng)       = 1 đơn vị thực thi BÊN TRONG process

  Process (App Java)
  ├── Main thread (chạy main())
  ├── Thread-1 (đọc file)
  ├── Thread-2 (gọi API)
  └── Thread-3 (xử lý data)
  → Chạy song song → App nhanh hơn
```

### 2. Tạo Thread — 2 cách

```java
// Cách 1: implements Runnable (KHUYẾN KHÍCH — vì Java chỉ cho extends 1 class)
Runnable task = () -> System.out.println("Running in: " + Thread.currentThread().getName());
Thread t1 = new Thread(task, "Worker-1");
t1.start();   // start() tạo thread mới VÀ chạy run()
              // ⚠️ KHÔNG gọi run() trực tiếp — run() chạy trên thread HIỆN TẠI

// Cách 2: extends Thread (ít dùng)
class MyThread extends Thread {
    @Override
    public void run() { System.out.println("Running"); }
}
new MyThread().start();
```

### 3. Thread Methods

```java
Thread t = new Thread(task);

t.start();                    // Bắt đầu thread mới
t.join();                     // Chờ thread t kết thúc rồi mới tiếp tục
t.join(5000);                 // Chờ tối đa 5 giây
Thread.sleep(1000);           // Dừng thread hiện tại 1 giây
t.interrupt();                // Gửi tín hiệu ngắt
t.isAlive();                  // Thread còn chạy?
t.setDaemon(true);            // Daemon thread: tự chết khi main thread kết thúc
Thread.currentThread();       // Lấy thread hiện tại
```

### 4. Synchronization — Vấn đề Race Condition

```java
// ❌ Race Condition: 2 threads cùng sửa 1 biến → kết quả sai
class Counter {
    private int count = 0;

    public void increment() { count++; }    // count++ KHÔNG atomic!
    // count++ = đọc count → +1 → ghi count (3 bước, thread có thể chen vào)
}

// ✅ Fix 1: synchronized
class SafeCounter {
    private int count = 0;

    public synchronized void increment() { count++; }  // Chỉ 1 thread vào cùng lúc
}

// ✅ Fix 2: AtomicInteger (nhanh hơn synchronized)
class AtomicCounter {
    private AtomicInteger count = new AtomicInteger(0);

    public void increment() { count.incrementAndGet(); }  // Thread-safe, lock-free
}
```

### 5. volatile — Đảm bảo visibility

```java
// ❌ Không có volatile: Thread khác có thể không thấy thay đổi (do CPU cache)
private boolean running = true;

// ✅ volatile: Đảm bảo MỌI thread đọc giá trị MỚI NHẤT từ main memory
private volatile boolean running = true;

// 💡 volatile CHỈ đảm bảo visibility, KHÔNG đảm bảo atomicity
// count++ vẫn cần synchronized/Atomic dù có volatile
```

---

## Phần B: Concurrency Utilities (java.util.concurrent)

### 1. ExecutorService — Thread Pool

> Thay vì tạo/hủy Thread thủ công → dùng Thread Pool: nhóm threads tái sử dụng.

```java
// Tạo thread pool
ExecutorService executor = Executors.newFixedThreadPool(4);  // 4 threads

// Gửi task
executor.submit(() -> System.out.println("Task 1: " + Thread.currentThread().getName()));
executor.submit(() -> System.out.println("Task 2: " + Thread.currentThread().getName()));

// Kết thúc
executor.shutdown();                  // Chờ tasks hoàn thành rồi tắt
executor.awaitTermination(10, TimeUnit.SECONDS);  // Chờ tối đa 10s
```

### 2. Các loại Thread Pool

| Pool | Cách tạo | Đặc điểm | Dùng khi |
|------|---------|-----------|----------|
| **Fixed** | `newFixedThreadPool(n)` | n threads cố định | Biết trước workload |
| **Cached** | `newCachedThreadPool()` | Tạo thread khi cần, tái dùng | Nhiều task ngắn |
| **Single** | `newSingleThreadExecutor()` | 1 thread duy nhất | Đảm bảo thứ tự |
| **Scheduled** | `newScheduledThreadPool(n)` | Chạy định kỳ | Cron-like tasks |

```java
// Scheduled: chạy task mỗi 5 giây
ScheduledExecutorService scheduler = Executors.newScheduledThreadPool(1);
scheduler.scheduleAtFixedRate(
    () -> System.out.println("Heartbeat"),
    0,        // delay ban đầu
    5,        // interval
    TimeUnit.SECONDS
);
```

### 3. Callable & Future — Task có kết quả trả về

```java
// Runnable: không trả kết quả
// Callable: CÓ trả kết quả + CÓ THỂ throw exception

ExecutorService executor = Executors.newFixedThreadPool(2);

Callable<Integer> task = () -> {
    Thread.sleep(1000);  // Giả lập tính toán
    return 42;
};

Future<Integer> future = executor.submit(task);

// future.get() BLOCK cho đến khi có kết quả
Integer result = future.get();            // Chờ mãi mãi
Integer result2 = future.get(5, TimeUnit.SECONDS);  // Chờ tối đa 5s

future.isDone();     // Đã xong?
future.cancel(true); // Hủy task

executor.shutdown();
```

### 4. Locks (Khóa nâng cao)

```java
import java.util.concurrent.locks.*;

// ReentrantLock — linh hoạt hơn synchronized
ReentrantLock lock = new ReentrantLock();

public void safeMethod() {
    lock.lock();
    try {
        // Critical section — chỉ 1 thread vào
    } finally {
        lock.unlock();  // PHẢI unlock trong finally!
    }
}

// tryLock — thử lấy lock, không block
if (lock.tryLock(1, TimeUnit.SECONDS)) {
    try {
        // Lấy được lock
    } finally {
        lock.unlock();
    }
} else {
    // Không lấy được lock trong 1s → xử lý khác
}

// ReadWriteLock — cho phép nhiều reader, chỉ 1 writer
ReadWriteLock rwLock = new ReentrantReadWriteLock();
rwLock.readLock().lock();    // Nhiều thread đọc đồng thời OK
rwLock.writeLock().lock();   // Chỉ 1 thread ghi, block tất cả
```

### 5. Synchronizers — Phối hợp threads

| Class | Ví dụ đời thường | Cách dùng |
|-------|-----------------|-----------|
| **CountDownLatch** | Chờ đủ 5 người rồi xuất phát | `latch.countDown()` + `latch.await()` |
| **CyclicBarrier** | Chờ nhau ở điểm hẹn, rồi cùng đi tiếp | `barrier.await()` (có thể reset) |
| **Semaphore** | Bãi đỗ xe 3 chỗ — chỉ 3 xe vào | `sem.acquire()` + `sem.release()` |

```java
// CountDownLatch: Chờ 3 services khởi động xong
CountDownLatch latch = new CountDownLatch(3);

for (int i = 1; i <= 3; i++) {
    int serviceId = i;
    new Thread(() -> {
        System.out.println("Service " + serviceId + " started");
        latch.countDown();  // Giảm đếm
    }).start();
}

latch.await();  // Block cho đến khi count = 0
System.out.println("All services ready!");

// Semaphore: Giới hạn 3 connections đồng thời
Semaphore semaphore = new Semaphore(3);

semaphore.acquire();    // Lấy 1 permit (block nếu hết)
try {
    // Dùng connection
} finally {
    semaphore.release();  // Trả permit
}
```

### 6. Thread-safe Collections

```java
// ❌ KHÔNG thread-safe
List<String> list = new ArrayList<>();        // Dùng trong single-thread
Map<String, Integer> map = new HashMap<>();

// ✅ Thread-safe options
List<String> safeList = new CopyOnWriteArrayList<>();      // Đọc nhiều, ghi ít
Map<String, Integer> safeMap = new ConcurrentHashMap<>();   // ⭐ Dùng nhiều nhất
Queue<String> safeQueue = new ConcurrentLinkedQueue<>();    // Lock-free queue
BlockingQueue<String> bq = new LinkedBlockingQueue<>(100);  // Producer-Consumer
```

---

## Tóm tắt: Chọn cách nào?

```
Cần thread-safe?
│
├── Biến đơn (int, boolean)?
│   ├── Chỉ đọc → volatile
│   └── Đọc + ghi → AtomicInteger / AtomicBoolean
│
├── Collection?
│   ├── Map → ConcurrentHashMap ⭐
│   ├── List (đọc nhiều, ghi ít) → CopyOnWriteArrayList
│   └── Queue → ConcurrentLinkedQueue hoặc BlockingQueue
│
├── Critical section (block code)?
│   ├── Đơn giản → synchronized
│   └── Cần timeout/tryLock → ReentrantLock
│
└── Quản lý threads?
    ├── Biết trước số tasks → Executors.newFixedThreadPool(n)
    ├── Nhiều task ngắn → Executors.newCachedThreadPool()
    └── Chạy định kỳ → Executors.newScheduledThreadPool(n)
```

---

## Bài tập

1. **Parallel Download**: Dùng ExecutorService tải 5 "files" song song (giả lập bằng Thread.sleep), in tiến trình
2. **Producer-Consumer**: 1 thread tạo data, 1 thread xử lý, dùng BlockingQueue
3. **Thread-safe Counter**: Benchmark so sánh synchronized vs AtomicInteger vs ReentrantLock (1 triệu increments)

---

## Navigation

- [← Day 5: Stream + I/O + DateTime](./day-5-stream-io-datetime.md)
- [Day 7: Async + Patterns + JVM →](./day-7-async-patterns-jvm.md)
