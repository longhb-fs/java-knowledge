# Day 13: Multithreading Basics

## Mục tiêu
- Thread creation
- Thread states và lifecycle
- Synchronization
- Thread communication

---

## 1. Creating Threads

### 1.1. Extend Thread Class

```java
public class MyThread extends Thread {
    @Override
    public void run() {
        System.out.println("Thread: " + Thread.currentThread().getName());
    }
}

// Usage
MyThread thread = new MyThread();
thread.start();  // NOT run()!
```

### 1.2. Implement Runnable

```java
public class MyRunnable implements Runnable {
    @Override
    public void run() {
        System.out.println("Runnable: " + Thread.currentThread().getName());
    }
}

// Usage
Thread thread = new Thread(new MyRunnable());
thread.start();

// Lambda
Thread thread2 = new Thread(() -> {
    System.out.println("Lambda thread");
});
thread2.start();
```

---

## 2. Thread States

```
NEW → RUNNABLE → BLOCKED/WAITING/TIMED_WAITING → TERMINATED
```

```java
Thread thread = new Thread(() -> { /* ... */ });

thread.getState();  // NEW
thread.start();
thread.getState();  // RUNNABLE

// Check if alive
thread.isAlive();
```

---

## 3. Thread Methods

```java
// Current thread
Thread current = Thread.currentThread();
current.getName();
current.getId();
current.getPriority();

// Set properties
thread.setName("MyThread");
thread.setPriority(Thread.MAX_PRIORITY);  // 1-10
thread.setDaemon(true);  // Daemon thread

// Control
Thread.sleep(1000);  // Pause current thread
thread.join();       // Wait for thread to finish
thread.join(5000);   // Wait max 5 seconds

// Interrupt
thread.interrupt();
Thread.interrupted();     // Check & clear flag
thread.isInterrupted();   // Check only
```

---

## 4. Synchronization

### 4.1. synchronized Method

```java
public class Counter {
    private int count = 0;

    public synchronized void increment() {
        count++;
    }

    public synchronized int getCount() {
        return count;
    }
}
```

### 4.2. synchronized Block

```java
public class Counter {
    private int count = 0;
    private final Object lock = new Object();

    public void increment() {
        synchronized (lock) {
            count++;
        }
    }
}
```

### 4.3. volatile Keyword

```java
public class Flag {
    private volatile boolean running = true;

    public void stop() {
        running = false;
    }

    public void run() {
        while (running) {
            // Do work
        }
    }
}
```

---

## 5. Thread Communication

```java
public class ProducerConsumer {
    private final Object lock = new Object();
    private int data;
    private boolean hasData = false;

    public void produce(int value) throws InterruptedException {
        synchronized (lock) {
            while (hasData) {
                lock.wait();  // Wait for consumer
            }
            data = value;
            hasData = true;
            lock.notify();  // Notify consumer
        }
    }

    public int consume() throws InterruptedException {
        synchronized (lock) {
            while (!hasData) {
                lock.wait();  // Wait for producer
            }
            hasData = false;
            lock.notify();  // Notify producer
            return data;
        }
    }
}
```

---

## 6. Thread-Safe Collections

```java
// Synchronized wrappers
List<String> syncList = Collections.synchronizedList(new ArrayList<>());
Map<String, Integer> syncMap = Collections.synchronizedMap(new HashMap<>());

// Concurrent collections
ConcurrentHashMap<String, Integer> concurrentMap = new ConcurrentHashMap<>();
CopyOnWriteArrayList<String> cowList = new CopyOnWriteArrayList<>();
BlockingQueue<String> blockingQueue = new LinkedBlockingQueue<>();
```

---

## 7. Atomic Classes

```java
import java.util.concurrent.atomic.*;

AtomicInteger atomicInt = new AtomicInteger(0);
atomicInt.incrementAndGet();
atomicInt.getAndIncrement();
atomicInt.compareAndSet(expected, newValue);

AtomicBoolean atomicBool = new AtomicBoolean(false);
AtomicLong atomicLong = new AtomicLong(0L);
AtomicReference<String> atomicRef = new AtomicReference<>("Hello");
```

---

## 8. Bài tập thực hành

### Bài 1: Producer-Consumer
Implement producer-consumer với BlockingQueue.

### Bài 2: Thread Pool Simulation
Tạo simple thread pool.

### Bài 3: Bank Account
Simulate concurrent deposits/withdrawals.

### Bài 4: Countdown Latch
Implement parallel task completion.

---

## Đáp án tham khảo

<details>
<summary>Bài 3: Bank Account</summary>

```java
public class BankAccount {
    private double balance;
    private final Object lock = new Object();

    public void deposit(double amount) {
        synchronized (lock) {
            balance += amount;
            System.out.printf("Deposited %.2f. Balance: %.2f%n", amount, balance);
        }
    }

    public boolean withdraw(double amount) {
        synchronized (lock) {
            if (balance >= amount) {
                balance -= amount;
                System.out.printf("Withdrawn %.2f. Balance: %.2f%n", amount, balance);
                return true;
            }
            return false;
        }
    }
}
```
</details>

---

## Navigation

- [← Day 12: Date/Time API](./day-12-datetime-api.md)
- [Day 14: Concurrency →](./day-14-concurrency.md)
