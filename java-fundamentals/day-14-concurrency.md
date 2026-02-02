# Day 14: Concurrency

## Mục tiêu
- ExecutorService
- Callable và Future
- Thread pools
- Locks

---

## 1. ExecutorService

```java
import java.util.concurrent.*;

// Create executor
ExecutorService executor = Executors.newFixedThreadPool(4);
ExecutorService single = Executors.newSingleThreadExecutor();
ExecutorService cached = Executors.newCachedThreadPool();
ScheduledExecutorService scheduled = Executors.newScheduledThreadPool(2);

// Submit tasks
executor.execute(() -> System.out.println("Task 1"));  // Runnable
Future<String> future = executor.submit(() -> "Result");  // Callable

// Shutdown
executor.shutdown();
executor.shutdownNow();
executor.awaitTermination(60, TimeUnit.SECONDS);
```

---

## 2. Callable và Future

```java
// Callable - returns value
Callable<Integer> task = () -> {
    Thread.sleep(1000);
    return 42;
};

ExecutorService executor = Executors.newSingleThreadExecutor();
Future<Integer> future = executor.submit(task);

// Future methods
future.isDone();
future.isCancelled();
future.cancel(true);

// Get result (blocking)
Integer result = future.get();
Integer result2 = future.get(5, TimeUnit.SECONDS);  // With timeout
```

---

## 3. invokeAll và invokeAny

```java
List<Callable<String>> tasks = Arrays.asList(
    () -> { Thread.sleep(1000); return "Task 1"; },
    () -> { Thread.sleep(500); return "Task 2"; },
    () -> { Thread.sleep(1500); return "Task 3"; }
);

// invokeAll - wait for all
List<Future<String>> results = executor.invokeAll(tasks);

// invokeAny - return first completed
String first = executor.invokeAny(tasks);
```

---

## 4. Thread Pool Types

```java
// Fixed - n threads
ExecutorService fixed = Executors.newFixedThreadPool(4);

// Cached - create as needed, reuse idle
ExecutorService cached = Executors.newCachedThreadPool();

// Single - 1 thread, sequential
ExecutorService single = Executors.newSingleThreadExecutor();

// Scheduled - delayed/periodic tasks
ScheduledExecutorService scheduled = Executors.newScheduledThreadPool(2);

scheduled.schedule(() -> System.out.println("Delayed"), 5, TimeUnit.SECONDS);
scheduled.scheduleAtFixedRate(() -> System.out.println("Periodic"), 0, 1, TimeUnit.SECONDS);

// Custom ThreadPoolExecutor
ThreadPoolExecutor custom = new ThreadPoolExecutor(
    2,    // core pool size
    10,   // max pool size
    60L,  // keep alive time
    TimeUnit.SECONDS,
    new LinkedBlockingQueue<>(100)  // work queue
);
```

---

## 5. Locks

```java
import java.util.concurrent.locks.*;

// ReentrantLock
ReentrantLock lock = new ReentrantLock();

lock.lock();
try {
    // Critical section
} finally {
    lock.unlock();
}

// Try lock
if (lock.tryLock(1, TimeUnit.SECONDS)) {
    try {
        // Critical section
    } finally {
        lock.unlock();
    }
}

// ReadWriteLock
ReadWriteLock rwLock = new ReentrantReadWriteLock();
Lock readLock = rwLock.readLock();
Lock writeLock = rwLock.writeLock();
```

---

## 6. Synchronizers

```java
// CountDownLatch
CountDownLatch latch = new CountDownLatch(3);
latch.countDown();  // Decrease count
latch.await();      // Wait until count = 0

// CyclicBarrier
CyclicBarrier barrier = new CyclicBarrier(3, () -> System.out.println("All arrived"));
barrier.await();  // Wait for others

// Semaphore
Semaphore semaphore = new Semaphore(3);  // 3 permits
semaphore.acquire();
try {
    // Use resource
} finally {
    semaphore.release();
}
```

---

## 7. Bài tập thực hành

### Bài 1: Parallel File Processing
Xử lý nhiều files song song với ExecutorService.

### Bài 2: Web Scraper
Fetch multiple URLs concurrently.

### Bài 3: Task Coordinator
Dùng CountDownLatch để coordinate startup tasks.

---

## Đáp án tham khảo

<details>
<summary>Bài 2: Web Scraper</summary>

```java
public class WebScraper {
    private final ExecutorService executor = Executors.newFixedThreadPool(10);

    public Map<String, String> fetchAll(List<String> urls) throws Exception {
        List<Callable<Map.Entry<String, String>>> tasks = urls.stream()
            .map(url -> (Callable<Map.Entry<String, String>>) () -> {
                String content = fetchUrl(url);
                return Map.entry(url, content);
            })
            .collect(Collectors.toList());

        return executor.invokeAll(tasks).stream()
            .map(future -> {
                try { return future.get(); }
                catch (Exception e) { return Map.entry("error", e.getMessage()); }
            })
            .collect(Collectors.toMap(Map.Entry::getKey, Map.Entry::getValue));
    }
}
```
</details>

---

## Navigation

- [← Day 13: Multithreading](./day-13-multithreading-basics.md)
- [Day 15: CompletableFuture →](./day-15-completable-future.md)
