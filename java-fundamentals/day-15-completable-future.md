# Day 15: CompletableFuture

## Mục tiêu
- Async programming với CompletableFuture
- Chaining và combining
- Exception handling
- Real-world patterns

---

## 1. Creating CompletableFuture

```java
import java.util.concurrent.CompletableFuture;

// Run async task
CompletableFuture<Void> cf1 = CompletableFuture.runAsync(() -> {
    System.out.println("Running async");
});

// Supply async value
CompletableFuture<String> cf2 = CompletableFuture.supplyAsync(() -> {
    return "Hello";
});

// Already completed
CompletableFuture<String> completed = CompletableFuture.completedFuture("Done");

// With custom executor
ExecutorService executor = Executors.newFixedThreadPool(4);
CompletableFuture<String> cf3 = CompletableFuture.supplyAsync(() -> "Result", executor);
```

---

## 2. Transformations (thenApply, thenAccept, thenRun)

```java
CompletableFuture<String> future = CompletableFuture.supplyAsync(() -> "Hello");

// thenApply - transform result (Function)
CompletableFuture<Integer> lengthFuture = future.thenApply(s -> s.length());

// thenAccept - consume result (Consumer)
future.thenAccept(s -> System.out.println("Result: " + s));

// thenRun - run after completion (Runnable)
future.thenRun(() -> System.out.println("Done!"));

// Chaining
CompletableFuture<String> result = CompletableFuture
    .supplyAsync(() -> "hello")
    .thenApply(String::toUpperCase)
    .thenApply(s -> s + " WORLD")
    .thenApply(s -> s + "!");
// "HELLO WORLD!"
```

---

## 3. Combining Futures

```java
CompletableFuture<String> cf1 = CompletableFuture.supplyAsync(() -> "Hello");
CompletableFuture<String> cf2 = CompletableFuture.supplyAsync(() -> "World");

// thenCombine - combine two results
CompletableFuture<String> combined = cf1.thenCombine(cf2, (s1, s2) -> s1 + " " + s2);
// "Hello World"

// thenCompose - chain dependent futures (flatMap)
CompletableFuture<String> composed = cf1.thenCompose(s ->
    CompletableFuture.supplyAsync(() -> s + " World")
);

// allOf - wait for all
CompletableFuture<Void> all = CompletableFuture.allOf(cf1, cf2);

// anyOf - wait for first
CompletableFuture<Object> any = CompletableFuture.anyOf(cf1, cf2);
```

---

## 4. Exception Handling

```java
CompletableFuture<String> future = CompletableFuture.supplyAsync(() -> {
    if (true) throw new RuntimeException("Error!");
    return "Success";
});

// exceptionally - handle exception
CompletableFuture<String> handled = future.exceptionally(ex -> {
    System.out.println("Error: " + ex.getMessage());
    return "Default";
});

// handle - handle both success and failure
CompletableFuture<String> handled2 = future.handle((result, ex) -> {
    if (ex != null) {
        return "Error: " + ex.getMessage();
    }
    return result;
});

// whenComplete - perform action on completion
future.whenComplete((result, ex) -> {
    if (ex != null) {
        System.out.println("Failed: " + ex.getMessage());
    } else {
        System.out.println("Success: " + result);
    }
});
```

---

## 5. Async Variants

```java
// Sync (same thread)
future.thenApply(s -> s.toUpperCase());

// Async (different thread)
future.thenApplyAsync(s -> s.toUpperCase());

// Async with executor
future.thenApplyAsync(s -> s.toUpperCase(), executor);

// All methods have async variants:
// thenApplyAsync, thenAcceptAsync, thenRunAsync
// thenCombineAsync, thenComposeAsync
// handleAsync, whenCompleteAsync
```

---

## 6. Practical Patterns

### 6.1. Parallel API Calls

```java
public CompletableFuture<UserProfile> getUserProfile(String userId) {
    CompletableFuture<User> userFuture = fetchUser(userId);
    CompletableFuture<List<Order>> ordersFuture = fetchOrders(userId);
    CompletableFuture<Preferences> prefsFuture = fetchPreferences(userId);

    return CompletableFuture.allOf(userFuture, ordersFuture, prefsFuture)
        .thenApply(v -> new UserProfile(
            userFuture.join(),
            ordersFuture.join(),
            prefsFuture.join()
        ));
}
```

### 6.2. Retry Pattern

```java
public <T> CompletableFuture<T> retry(Supplier<CompletableFuture<T>> task, int maxRetries) {
    return task.get().exceptionallyCompose(ex -> {
        if (maxRetries > 0) {
            return retry(task, maxRetries - 1);
        }
        return CompletableFuture.failedFuture(ex);
    });
}
```

### 6.3. Timeout

```java
CompletableFuture<String> future = CompletableFuture.supplyAsync(() -> {
    Thread.sleep(5000);
    return "Result";
});

// Java 9+
future.orTimeout(2, TimeUnit.SECONDS);
future.completeOnTimeout("Default", 2, TimeUnit.SECONDS);
```

---

## 7. Bài tập thực hành

### Bài 1: Parallel Service Calls
Gọi 3 services song song và combine kết quả.

### Bài 2: Chain with Retry
Implement retry logic cho failed operations.

### Bài 3: Batch Processing
Xử lý batch items với rate limiting.

---

## Navigation

- [← Day 14: Concurrency](./day-14-concurrency.md)
- [Day 16: Reflection & Annotations →](./day-16-reflection-annotations.md)
