# Day 13: Multithreading Basics (Đa Luồng Cơ Bản)

## Mục tiêu hôm nay
- Hiểu Thread (luồng) là gì và tại sao cần đa luồng
- 2 cách tạo Thread: extends Thread vs implements Runnable
- Thread States (trạng thái) và Lifecycle (vòng đời)
- Synchronization (đồng bộ hóa) - giải quyết race condition
- Thread Communication (giao tiếp giữa các luồng) - wait/notify
- Thread-safe Collections và Atomic Classes

---

## 🤔 Tại sao cần học Multithreading?

### Ví dụ đời thường
> Hãy tưởng tượng **1 quán ăn**:
> - **Single-threaded** (1 luồng): Chỉ có **1 nhân viên** - tiếp khách → nấu ăn → phục vụ → rửa bát → tiếp khách tiếp. Khách phải chờ rất lâu!
> - **Multi-threaded** (đa luồng): Có **nhiều nhân viên** - 1 người tiếp khách, 1 người nấu, 1 người phục vụ, 1 người rửa bát → phục vụ nhanh hơn!
>
> Trong lập trình cũng vậy:
> - Download file + hiển thị UI → 2 threads
> - Server xử lý nhiều request đồng thời → mỗi request 1 thread
> - Tính toán nặng → chia cho nhiều cores xử lý song song

### Process vs Thread

```
┌─────────────────────────────────────────────────────────┐
│                  PROCESS vs THREAD                      │
├──────────────────────────┬──────────────────────────────┤
│ Process (Tiến trình)     │ Thread (Luồng)               │
├──────────────────────────┼──────────────────────────────┤
│ Chương trình đang chạy   │ Đơn vị nhỏ nhất trong       │
│ (Chrome, IntelliJ...)    │ Process                      │
│                          │                              │
│ Có bộ nhớ RIÊNG          │ CHIA SẺ bộ nhớ với           │
│ (address space riêng)    │ các Thread khác trong        │
│                          │ cùng Process                 │
│                          │                              │
│ Nặng (tốn tài nguyên    │ Nhẹ (tạo nhanh, ít tài      │
│ để tạo)                  │ nguyên)                      │
│                          │                              │
│ Giao tiếp khó (IPC)     │ Giao tiếp dễ (chung bộ nhớ) │
├──────────────────────────┼──────────────────────────────┤
│ Ví dụ: Mỗi cửa sổ      │ Ví dụ: Trong 1 cửa sổ       │
│ Chrome = 1 Process       │ Chrome → 1 thread render,   │
│                          │ 1 thread tải, 1 thread JS   │
└──────────────────────────┴──────────────────────────────┘
```

---

## 1. Tạo Thread

### 1.1. Cách 1: Extends Thread

```java
// Bước 1: Tạo class kế thừa Thread
public class MyThread extends Thread {
    @Override
    public void run() {
        // Code chạy trong thread mới
        for (int i = 0; i < 5; i++) {
            System.out.println("Thread " + getName() + ": " + i);
            try {
                Thread.sleep(500);       // Tạm dừng 500ms (0.5 giây)
            } catch (InterruptedException e) {
                break;
            }
        }
    }
}

// Bước 2: Tạo object và START
MyThread thread = new MyThread();
thread.start();   // ← ĐÚNG: Tạo thread mới và gọi run()
// thread.run(); // ← SAI! Chạy run() trên thread HIỆN TẠI, KHÔNG tạo thread mới!
```

### 1.2. Cách 2: Implements Runnable (KHUYÊN DÙNG)

```java
// Bước 1: Tạo class implements Runnable
public class MyRunnable implements Runnable {
    @Override
    public void run() {
        System.out.println("Runnable chạy trên: " + Thread.currentThread().getName());
    }
}

// Bước 2: Wrap vào Thread và start
Thread thread = new Thread(new MyRunnable());
thread.start();

// Hoặc dùng Lambda (ngắn gọn nhất)
Thread thread2 = new Thread(() -> {
    System.out.println("Lambda thread chạy trên: " + Thread.currentThread().getName());
});
thread2.start();
```

### So sánh 2 cách

```
┌────────────────────────────┬────────────────────────────┐
│ extends Thread             │ implements Runnable         │
├────────────────────────────┼────────────────────────────┤
│ Kế thừa Thread class       │ Implement Runnable interface│
│                            │                            │
│ KHÔNG thể kế thừa class    │ CÓ THỂ kế thừa class khác │
│ khác (Java single inherit) │ (linh hoạt hơn)            │
│                            │                            │
│ Gọi trực tiếp: start()    │ Phải wrap: new Thread(r)   │
│                            │                            │
│ ❌ Ít dùng                 │ ✅ KHUYÊN DÙNG             │
│                            │ (hỗ trợ Lambda, linh hoạt) │
└────────────────────────────┴────────────────────────────┘

💡 Quy tắc: LUÔN dùng Runnable (hoặc Lambda) trừ khi có lý do đặc biệt
```

### ⚠️ Bẫy kinh điển: start() vs run()

```java
Thread thread = new Thread(() -> {
    System.out.println("Chạy trên: " + Thread.currentThread().getName());
});

// ❌ SAI: Gọi run() trực tiếp
thread.run();
// Output: "Chạy trên: main"  ← Chạy trên thread MAIN, KHÔNG tạo thread mới!

// ✅ ĐÚNG: Gọi start()
thread.start();
// Output: "Chạy trên: Thread-0"  ← Chạy trên thread MỚI!

// 💡 Giải thích:
// start() → JVM tạo thread mới trong OS → gọi run() trên thread mới
// run()   → Chỉ là method thường, chạy trên thread gọi nó (main)
```

---

## 2. Thread States (Trạng Thái Luồng)

```
┌───────────────────────────────────────────────────────────────────┐
│                    VÒNG ĐỜI CỦA THREAD                          │
│                                                                   │
│  ┌─────┐   start()   ┌──────────┐                                │
│  │ NEW │ ──────────► │ RUNNABLE │ ◄──── Thread sẵn sàng/đang chạy│
│  └─────┘             └──────────┘                                │
│                        │      ▲                                   │
│          synchronized  │      │ lock acquired                     │
│          (chờ lock)    ▼      │                                   │
│                     ┌─────────┐                                   │
│                     │ BLOCKED │  ← Chờ lấy lock (synchronized)   │
│                     └─────────┘                                   │
│                        │      ▲                                   │
│            wait()      │      │ notify()/notifyAll()              │
│            join()      ▼      │                                   │
│                     ┌─────────┐                                   │
│                     │ WAITING │  ← Chờ vô thời hạn               │
│                     └─────────┘                                   │
│                        │      ▲                                   │
│        sleep(ms)       │      │ hết thời gian                     │
│        wait(ms)        ▼      │                                   │
│                 ┌──────────────┐                                  │
│                 │TIMED_WAITING │ ← Chờ có thời hạn               │
│                 └──────────────┘                                  │
│                                                                   │
│  Khi run() kết thúc:                                              │
│  ┌────────────┐                                                   │
│  │ TERMINATED │  ← Thread đã chết, KHÔNG thể start lại           │
│  └────────────┘                                                   │
└───────────────────────────────────────────────────────────────────┘
```

```java
Thread thread = new Thread(() -> { /* ... */ });

thread.getState();   // NEW - vừa tạo, chưa start
thread.start();
thread.getState();   // RUNNABLE - đang chạy hoặc sẵn sàng chạy
thread.isAlive();    // true - thread chưa kết thúc
// ... sau khi run() xong
thread.getState();   // TERMINATED - đã kết thúc
```

---

## 3. Thread Methods (Các Method Quan Trọng)

```java
// === Lấy thông tin thread hiện tại ===
Thread current = Thread.currentThread();
current.getName();      // "main" (hoặc "Thread-0", "Thread-1"...)
current.getId();        // ID duy nhất
current.getPriority();  // Độ ưu tiên (1-10, mặc định 5)

// === Cài đặt thread ===
thread.setName("Worker-1");                    // Đặt tên (giúp debug dễ hơn)
thread.setPriority(Thread.MAX_PRIORITY);       // 10 - ưu tiên cao nhất
                                               // MIN_PRIORITY = 1, NORM_PRIORITY = 5
thread.setDaemon(true);                        // Daemon thread (giải thích bên dưới)
// ⚠️ Phải gọi setDaemon() TRƯỚC khi start()!

// === Điều khiển thread ===
Thread.sleep(1000);    // Tạm dừng thread HIỆN TẠI 1 giây
                       // ⚠️ Phải try-catch InterruptedException

thread.join();         // Chờ thread này chết rồi mới tiếp tục
thread.join(5000);     // Chờ tối đa 5 giây

// === Ngắt thread ===
thread.interrupt();            // Gửi tín hiệu ngắt → thread nên dừng lại
thread.isInterrupted();        // Kiểm tra flag ngắt (KHÔNG clear flag)
Thread.interrupted();          // Kiểm tra + CLEAR flag ngắt của thread hiện tại
```

### Daemon Thread là gì?

```java
// Daemon thread = "thread nền" - tự chết khi tất cả non-daemon thread đã chết
// Ví dụ: Garbage Collector là daemon thread

Thread daemon = new Thread(() -> {
    while (true) {
        System.out.println("Daemon đang chạy...");
        try { Thread.sleep(1000); } catch (InterruptedException e) { break; }
    }
});
daemon.setDaemon(true);    // Đánh dấu là daemon
daemon.start();

// Khi main thread (non-daemon) kết thúc → daemon tự động bị kill
// → Không cần lo về vòng lặp vô hạn trong daemon thread

// 💡 Mẹo nhớ:
// Non-daemon (User thread): "Nhân viên chính" - công ty (JVM) phải chờ họ xong việc
// Daemon: "Bảo vệ" - khi nhân viên về hết, bảo vệ cũng về (JVM tắt)
```

### Thread.join() - Chờ thread khác

```java
// Ví dụ: Thread main chờ worker xong mới tiếp tục
Thread worker = new Thread(() -> {
    System.out.println("Worker bắt đầu...");
    try { Thread.sleep(3000); } catch (InterruptedException e) {}
    System.out.println("Worker xong!");
});

worker.start();
System.out.println("Main đang chờ worker...");
worker.join();     // ← Main ĐỨNG ĐÂY chờ worker kết thúc
System.out.println("Main tiếp tục sau khi worker xong!");

// Output:
// Main đang chờ worker...
// Worker bắt đầu...
// Worker xong!                    ← Sau 3 giây
// Main tiếp tục sau khi worker xong!  ← Main chạy tiếp
```

---

## 4. Synchronization (Đồng Bộ Hóa)

### 🚨 Vấn đề: Race Condition (Điều Kiện Tranh Chấp)

> **Ví dụ đời thường**: 2 người cùng rút tiền từ **1 tài khoản ATM** đồng thời. Tài khoản có 1000đ, cả 2 rút 800đ. Nếu không có cơ chế khóa → cả 2 đều thấy "còn 1000đ" → cả 2 đều rút → mất 600đ!

```java
// ❌ CODE KHÔNG AN TOÀN: Race condition!
public class UnsafeCounter {
    private int count = 0;

    public void increment() {
        count++;    // KHÔNG atomic! Thực ra gồm 3 bước:
                    // 1. Đọc count (ví dụ: 5)
                    // 2. Tính count + 1 (= 6)
                    // 3. Ghi count = 6
                    // → 2 thread đọc cùng lúc → cả 2 thấy 5 → cả 2 ghi 6 → MẤT 1 lần tăng!
    }
}

// Test:
UnsafeCounter counter = new UnsafeCounter();
Thread t1 = new Thread(() -> { for (int i = 0; i < 10000; i++) counter.increment(); });
Thread t2 = new Thread(() -> { for (int i = 0; i < 10000; i++) counter.increment(); });
t1.start(); t2.start();
t1.join(); t2.join();
System.out.println(counter.getCount());
// Kỳ vọng: 20000
// Thực tế: 18XXX (mỗi lần chạy ra kết quả khác nhau!) 💥
```

### 4.1. synchronized Method - Khóa toàn bộ method

```java
// ✅ AN TOÀN: Dùng synchronized
public class SafeCounter {
    private int count = 0;

    // synchronized = "1 lần chỉ 1 thread được vào method này"
    public synchronized void increment() {
        count++;    // An toàn vì chỉ 1 thread chạy tại 1 thời điểm
    }

    public synchronized int getCount() {
        return count;
    }
}

// 💡 Cơ chế: Mỗi object có 1 "ổ khóa" (intrinsic lock / monitor)
// Thread 1 vào synchronized method → KHÓA object
// Thread 2 gọi synchronized method → PHẢI CHỜ (BLOCKED)
// Thread 1 xong → MỞ KHÓA → Thread 2 được vào
```

### 4.2. synchronized Block - Khóa một phần code

```java
public class SafeCounter {
    private int count = 0;
    private final Object lock = new Object();  // Object dùng làm khóa

    public void increment() {
        // Code ngoài synchronized: nhiều thread chạy song song OK
        synchronized (lock) {
            // Code trong synchronized: chỉ 1 thread tại 1 thời điểm
            count++;
        }
        // Code ngoài synchronized: nhiều thread chạy song song OK
    }
}

// 💡 synchronized block tốt hơn synchronized method vì:
// → Chỉ khóa đoạn code CẦN thiết (ít blocking hơn, nhanh hơn)
// → Có thể dùng object khác nhau làm lock (fine-grained locking)
```

### 4.3. volatile Keyword - Đảm bảo visibility (tính nhìn thấy)

```java
// Vấn đề: Mỗi thread có thể cache biến trong CPU cache riêng
// → Thread 1 sửa biến nhưng Thread 2 KHÔNG THẤY (đọc từ cache cũ)

public class StopFlag {
    // volatile = "luôn đọc/ghi từ Main Memory, KHÔNG cache"
    private volatile boolean running = true;

    public void stop() {
        running = false;     // Thread 1 ghi → Main Memory cập nhật ngay
    }

    public void doWork() {
        while (running) {    // Thread 2 đọc → lấy từ Main Memory (giá trị mới nhất)
            // Làm việc...
        }
        System.out.println("Đã dừng!");
    }
}

// ⚠️ volatile CHỈ đảm bảo visibility, KHÔNG đảm bảo atomicity!
// volatile phù hợp cho: flag boolean, read-only sau khi write
// KHÔNG phù hợp cho: count++ (vẫn cần synchronized)
```

```
volatile vs synchronized:

┌────────────────────────┬───────────────────────────────────┐
│ volatile               │ synchronized                      │
├────────────────────────┼───────────────────────────────────┤
│ Đảm bảo: Visibility   │ Đảm bảo: Visibility + Atomicity  │
│ (nhìn thấy giá trị    │ (nhìn thấy + thao tác nguyên tử) │
│ mới nhất)              │                                   │
├────────────────────────┼───────────────────────────────────┤
│ Nhanh (không blocking) │ Chậm hơn (có blocking)           │
├────────────────────────┼───────────────────────────────────┤
│ Dùng cho: flag, đọc    │ Dùng cho: count++, complex       │
│ /ghi đơn giản          │ operations, critical sections    │
└────────────────────────┴───────────────────────────────────┘
```

---

## 5. Thread Communication (Giao Tiếp Giữa Các Luồng)

### wait() / notify() / notifyAll()

> **Ví dụ đời thường**: **Producer-Consumer** (Người sản xuất - Người tiêu dùng)
> - Đầu bếp (Producer) nấu xong → đặt lên quầy → báo hiệu "Có đồ ăn!" (notify)
> - Phục vụ (Consumer) thấy quầy trống → đợi (wait) → được báo hiệu → lấy đồ ăn

```java
public class ProducerConsumer {
    private final Object lock = new Object();
    private int data;
    private boolean hasData = false;    // Có dữ liệu trên "quầy" không?

    // Producer: Tạo dữ liệu
    public void produce(int value) throws InterruptedException {
        synchronized (lock) {
            while (hasData) {
                lock.wait();             // Quầy còn đồ → CHỜ consumer lấy đi
            }
            data = value;                // Đặt dữ liệu lên quầy
            hasData = true;
            System.out.println("Produced: " + value);
            lock.notify();               // Báo hiệu consumer: "Có dữ liệu rồi!"
        }
    }

    // Consumer: Lấy dữ liệu
    public int consume() throws InterruptedException {
        synchronized (lock) {
            while (!hasData) {
                lock.wait();             // Quầy trống → CHỜ producer đặt đồ
            }
            hasData = false;
            System.out.println("Consumed: " + data);
            lock.notify();               // Báo hiệu producer: "Quầy trống rồi!"
            return data;
        }
    }
}

// ⚠️ QUAN TRỌNG:
// 1. wait/notify PHẢI gọi bên trong synchronized block
// 2. LUÔN dùng while (không dùng if) để kiểm tra điều kiện trước wait()
//    → Vì "spurious wakeup" (thức dậy giả) có thể xảy ra
// 3. notify() = đánh thức 1 thread, notifyAll() = đánh thức TẤT CẢ
```

---

## 6. Thread-Safe Collections

```java
// === Cách 1: Synchronized Wrappers (bọc collection cũ) ===
List<String> syncList = Collections.synchronizedList(new ArrayList<>());
Map<String, Integer> syncMap = Collections.synchronizedMap(new HashMap<>());
Set<String> syncSet = Collections.synchronizedSet(new HashSet<>());
// ⚠️ Khi iterate vẫn cần synchronized block bên ngoài!

// === Cách 2: Concurrent Collections (thiết kế sẵn cho multi-thread) ===
// Hiệu quả hơn synchronized wrappers rất nhiều!

ConcurrentHashMap<String, Integer> concurrentMap = new ConcurrentHashMap<>();
// → Khóa theo segment (phần), không khóa toàn bộ Map → nhiều thread đọc/ghi đồng thời

CopyOnWriteArrayList<String> cowList = new CopyOnWriteArrayList<>();
// → Tạo bản sao khi ghi → đọc không cần lock
// → Tốt khi đọc nhiều, ghi ít

BlockingQueue<String> blockingQueue = new LinkedBlockingQueue<>();
// → Queue tự block khi rỗng (take) hoặc đầy (put)
// → Hoàn hảo cho Producer-Consumer pattern

ConcurrentLinkedQueue<String> clQueue = new ConcurrentLinkedQueue<>();
// → Queue không blocking, dùng CAS (Compare-And-Swap)
```

```
Chọn Collection thread-safe nào?

┌─────────────────────────┬─────────────────────────────────────┐
│ Cần gì?                │ Dùng gì?                            │
├─────────────────────────┼─────────────────────────────────────┤
│ Map đa luồng           │ ConcurrentHashMap                   │
│ List đọc nhiều/ghi ít  │ CopyOnWriteArrayList                │
│ Queue Producer-Consumer │ LinkedBlockingQueue                 │
│ Queue không blocking    │ ConcurrentLinkedQueue               │
│ Map sắp xếp đa luồng   │ ConcurrentSkipListMap               │
└─────────────────────────┴─────────────────────────────────────┘
```

---

## 7. Atomic Classes (Lớp Nguyên Tử)

> **Ví dụ đời thường**: `count++` thực ra gồm 3 bước (đọc → tính → ghi). Atomic = "làm hết 3 bước trong 1 thao tác duy nhất, KHÔNG ai xen vào được".

```java
import java.util.concurrent.atomic.*;

// === AtomicInteger: thay thế synchronized cho phép tính số nguyên ===
AtomicInteger atomicInt = new AtomicInteger(0);

atomicInt.incrementAndGet();     // ++count (tăng rồi trả về)    → 1
atomicInt.getAndIncrement();     // count++ (trả về rồi tăng)    → 1 (trả 1, sau đó thành 2)
atomicInt.addAndGet(5);          // count += 5                    → 7
atomicInt.get();                 // Đọc giá trị hiện tại         → 7

// CAS (Compare-And-Set): "Nếu giá trị hiện tại = expected thì gán newValue"
atomicInt.compareAndSet(7, 10);  // Nếu đang = 7 thì gán = 10   → true

// === Các Atomic class khác ===
AtomicBoolean atomicBool = new AtomicBoolean(false);
AtomicLong atomicLong = new AtomicLong(0L);
AtomicReference<String> atomicRef = new AtomicReference<>("Hello");

// === Ví dụ: Counter an toàn KHÔNG cần synchronized ===
public class AtomicCounter {
    private final AtomicInteger count = new AtomicInteger(0);

    public void increment() {
        count.incrementAndGet();     // Atomic! Không cần synchronized
    }

    public int getCount() {
        return count.get();
    }
}
// 💡 Atomic nhanh hơn synchronized vì dùng CAS (hardware-level)
// → Không cần lock/unlock → ít overhead hơn
```

---

## 8. Sai Lầm Thường Gặp

### ❌ Sai lầm 1: Gọi run() thay vì start()

```java
// ❌ SAI:
Thread thread = new Thread(() -> System.out.println("Thread: " + Thread.currentThread().getName()));
thread.run();   // In: "Thread: main" → Chạy trên main thread, KHÔNG tạo thread mới!

// ✅ ĐÚNG:
thread.start(); // In: "Thread: Thread-0" → Chạy trên thread mới
```

### ❌ Sai lầm 2: Dùng if thay vì while khi wait()

```java
// ❌ SAI: Dùng if
synchronized (lock) {
    if (!hasData) {
        lock.wait();     // Nếu spurious wakeup → tiếp tục dù chưa có data!
    }
    process(data);       // 💥 Có thể xử lý khi chưa có data
}

// ✅ ĐÚNG: Dùng while
synchronized (lock) {
    while (!hasData) {
        lock.wait();     // Nếu spurious wakeup → kiểm tra lại → wait tiếp
    }
    process(data);       // An toàn: chắc chắn có data
}
```

### ❌ Sai lầm 3: Deadlock (Bế Tắc)

```java
// ❌ DEADLOCK: 2 thread chờ nhau mãi mãi!
Object lockA = new Object();
Object lockB = new Object();

Thread t1 = new Thread(() -> {
    synchronized (lockA) {          // T1 giữ lockA
        Thread.sleep(100);
        synchronized (lockB) {      // T1 chờ lockB (T2 đang giữ)
            // Không bao giờ đến đây!
        }
    }
});

Thread t2 = new Thread(() -> {
    synchronized (lockB) {          // T2 giữ lockB
        Thread.sleep(100);
        synchronized (lockA) {      // T2 chờ lockA (T1 đang giữ)
            // Không bao giờ đến đây!
        }
    }
});

// ✅ CÁCH TRÁNH: Luôn lấy lock theo THỨ TỰ GIỐNG NHAU
// T1: lockA → lockB
// T2: lockA → lockB (CÙNG thứ tự với T1)
```

### ❌ Sai lầm 4: Dùng Thread.stop() (deprecated)

```java
// ❌ SAI: stop() bị deprecated vì có thể để object ở trạng thái inconsistent
thread.stop();   // KHÔNG dùng!

// ✅ ĐÚNG: Dùng interrupt + flag
Thread worker = new Thread(() -> {
    while (!Thread.currentThread().isInterrupted()) {  // Kiểm tra flag
        // Làm việc...
    }
    System.out.println("Thread dừng an toàn!");
});

worker.start();
// ... sau 1 lúc ...
worker.interrupt();  // Gửi tín hiệu dừng (thread tự quyết định khi nào dừng)
```

---

## 9. Tóm Tắt Cuối Ngày

| Khái niệm | Giải thích | Ví dụ |
|------------|-----------|-------|
| **Thread** | Đơn vị nhỏ nhất của xử lý song song | `new Thread(runnable).start()` |
| **Runnable** | Interface định nghĩa code chạy trong thread | `() -> { /* code */ }` |
| **start() vs run()** | start() tạo thread mới; run() chỉ gọi method | LUÔN dùng start() |
| **Thread.sleep()** | Tạm dừng thread hiện tại | `Thread.sleep(1000)` |
| **join()** | Chờ thread khác kết thúc | `worker.join()` |
| **synchronized** | Khóa code chỉ cho 1 thread vào | `synchronized(lock){...}` |
| **volatile** | Đảm bảo thread đọc giá trị mới nhất | `volatile boolean flag` |
| **wait/notify** | Giao tiếp giữa các thread | Producer-Consumer pattern |
| **Daemon thread** | Thread nền, tự chết khi app tắt | `thread.setDaemon(true)` |
| **Race condition** | 2 thread tranh nhau sửa dữ liệu | count++ không synchronized |
| **Deadlock** | 2 thread chờ nhau mãi mãi | Lock A→B vs Lock B→A |
| **AtomicInteger** | Phép tính nguyên tử không cần lock | `atomicInt.incrementAndGet()` |
| **ConcurrentHashMap** | Map thread-safe hiệu quả | Thay thế synchronized Map |

---

## 10. Câu Hỏi Phỏng Vấn Thường Gặp

### 🔥 Câu 1: Thread khác Runnable thế nào? Nên dùng cái nào?
**Trả lời:**
- `extends Thread`: Kế thừa class Thread, không thể kế thừa class khác (Java single inheritance). Thread IS-A Thread
- `implements Runnable`: Implement interface, vẫn có thể kế thừa class khác. Linh hoạt hơn, hỗ trợ Lambda
- **Nên dùng Runnable** vì: linh hoạt hơn, tách biệt "task" (Runnable) khỏi "thread mechanism" (Thread), có thể dùng với thread pool (ExecutorService)

### 🔥 Câu 2: synchronized method khác synchronized block thế nào?
**Trả lời:**
- `synchronized method`: Khóa TOÀN BỘ method, dùng `this` (hoặc class object cho static) làm lock
- `synchronized block`: Khóa CHỈ đoạn code cần thiết, có thể chỉ định lock object bất kỳ
- Nên dùng synchronized block vì ít blocking hơn → hiệu suất tốt hơn. Fine-grained locking cho phép nhiều phần code chạy song song

### 🔥 Câu 3: volatile giải quyết vấn đề gì? Có thay thế synchronized không?
**Trả lời:**
`volatile` giải quyết **visibility** (tính nhìn thấy) - đảm bảo thread đọc giá trị mới nhất từ Main Memory thay vì CPU cache. KHÔNG thay thế synchronized vì không đảm bảo **atomicity**. `count++` với volatile vẫn bị race condition vì gồm 3 bước (đọc-tính-ghi). Volatile phù hợp cho flag boolean hoặc biến chỉ có 1 thread ghi

### 🔥 Câu 4: Deadlock là gì? Làm sao tránh?
**Trả lời:**
Deadlock = 2+ thread chờ nhau mãi mãi. Xảy ra khi: có mutual exclusion, hold and wait, no preemption, circular wait. Cách tránh:
1. Lấy lock theo thứ tự cố định (consistent ordering)
2. Dùng tryLock() với timeout thay vì lock vĩnh viễn
3. Giảm scope của synchronized (ít lock hơn)
4. Dùng concurrent utilities (java.util.concurrent) thay vì tự sync

### 🔥 Câu 5: wait() khác sleep() thế nào?
**Trả lời:**
- `wait()`: Thuộc Object class. PHẢI trong synchronized block. GIẢI PHÓNG lock → thread khác vào được. Thức dậy bởi notify()/notifyAll()
- `sleep()`: Thuộc Thread class. Không cần synchronized. GIỮ lock (nếu đang trong synchronized). Thức dậy sau thời gian chỉ định
- `wait()` dùng cho thread communication (chờ điều kiện), `sleep()` dùng để tạm dừng

### 🔥 Câu 6: ConcurrentHashMap khác Collections.synchronizedMap thế nào?
**Trả lời:**
- `synchronizedMap`: Khóa TOÀN BỘ map cho mỗi operation → chỉ 1 thread truy cập tại 1 thời điểm → chậm
- `ConcurrentHashMap`: Khóa theo SEGMENT (phần nhỏ) → nhiều thread đọc/ghi đồng thời ở các segment khác nhau → nhanh hơn nhiều
- ConcurrentHashMap còn hỗ trợ atomic operations: putIfAbsent(), computeIfAbsent() - không cần lock bên ngoài

---

## Navigation

- [← Day 12: Date/Time API](./day-12-datetime-api.md)
- [Day 14: Concurrency →](./day-14-concurrency.md)
