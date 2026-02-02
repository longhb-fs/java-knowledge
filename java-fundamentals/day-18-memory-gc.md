# Day 18: Memory & Garbage Collection

## Mục tiêu
- JVM Memory model
- Garbage Collection
- Memory leaks
- Performance tuning

---

## 1. JVM Memory Structure

```
JVM Memory
├── Heap (objects)
│   ├── Young Generation
│   │   ├── Eden Space
│   │   ├── Survivor 0
│   │   └── Survivor 1
│   └── Old Generation (Tenured)
│
├── Non-Heap
│   ├── Metaspace (class metadata)
│   └── Code Cache (JIT compiled)
│
├── Stack (per thread)
│   ├── Local variables
│   ├── Method calls
│   └── Partial results
│
└── Native Memory
```

---

## 2. Stack vs Heap

```java
public class MemoryExample {
    private int instanceVar;  // Heap

    public void method() {
        int localVar = 10;              // Stack
        String str = "Hello";           // Stack (reference), Heap (object)
        Object obj = new Object();      // Stack (reference), Heap (object)
    }
}
```

| Stack | Heap |
|-------|------|
| Local variables | Objects |
| Method calls | Instance variables |
| LIFO order | No order |
| Thread-specific | Shared |
| Auto cleanup | GC cleanup |
| Fast access | Slower access |
| Limited size | Larger size |

---

## 3. Garbage Collection

### 3.1. GC Algorithms

```
Serial GC        - Single thread, stop-the-world
Parallel GC      - Multiple threads, stop-the-world
G1 GC            - Default since Java 9, concurrent
ZGC              - Low latency (<10ms pause)
Shenandoah GC   - Low latency, RedHat
```

### 3.2. GC Roots

```java
// Objects reachable from GC roots are NOT collected
// GC Roots include:
// - Local variables on stack
// - Active threads
// - Static fields
// - JNI references
```

### 3.3. Object Lifecycle

```
1. Created in Eden
2. Minor GC: Survive → Survivor space
3. After N survivals → Old Generation
4. Major GC: Clean Old Generation
```

---

## 4. Monitoring Memory

```java
// Runtime memory info
Runtime runtime = Runtime.getRuntime();
long maxMemory = runtime.maxMemory();       // -Xmx
long totalMemory = runtime.totalMemory();   // Current heap
long freeMemory = runtime.freeMemory();     // Free in current heap
long usedMemory = totalMemory - freeMemory;

// Suggest GC (no guarantee)
System.gc();

// Memory MXBean
MemoryMXBean memoryBean = ManagementFactory.getMemoryMXBean();
MemoryUsage heapUsage = memoryBean.getHeapMemoryUsage();
System.out.println("Heap used: " + heapUsage.getUsed());
```

---

## 5. Memory Leaks

### 5.1. Common Causes

```java
// 1. Static collections
public class LeakyClass {
    private static List<Object> cache = new ArrayList<>();  // Never cleared
}

// 2. Unclosed resources
public void readFile() {
    FileInputStream fis = new FileInputStream("file.txt");
    // fis.close() never called!
}

// 3. Inner class references
public class Outer {
    private byte[] data = new byte[10_000_000];

    public Runnable getRunnable() {
        return new Runnable() {  // Holds reference to Outer
            public void run() { }
        };
    }
}

// 4. Custom hashCode/equals issues
Map<BadKey, String> map = new HashMap<>();
map.put(new BadKey(), "value");
// Cannot retrieve if hashCode changes
```

### 5.2. Prevention

```java
// 1. Use try-with-resources
try (FileInputStream fis = new FileInputStream("file.txt")) {
    // Use fis
}  // Auto-closed

// 2. Use WeakReference for caches
Map<Key, WeakReference<Value>> cache = new WeakHashMap<>();

// 3. Clear collections when done
list.clear();

// 4. Use static inner classes
public class Outer {
    private static class Inner {  // No reference to Outer
        // ...
    }
}
```

---

## 6. Reference Types

```java
// Strong reference - normal
Object obj = new Object();

// Weak reference - collected at next GC
WeakReference<Object> weak = new WeakReference<>(obj);
obj = null;
// weak.get() may return null after GC

// Soft reference - collected only when memory needed
SoftReference<Object> soft = new SoftReference<>(obj);
// Good for caches

// Phantom reference - for cleanup tracking
PhantomReference<Object> phantom = new PhantomReference<>(obj, queue);
// Always returns null, used with ReferenceQueue
```

---

## 7. JVM Options

```bash
# Heap size
-Xms512m        # Initial heap
-Xmx2g          # Maximum heap

# GC selection
-XX:+UseG1GC            # G1 (default)
-XX:+UseZGC             # ZGC
-XX:+UseParallelGC      # Parallel

# GC logging
-Xlog:gc*:file=gc.log

# Metaspace
-XX:MetaspaceSize=256m
-XX:MaxMetaspaceSize=512m

# Stack size
-Xss512k

# Heap dump on OOM
-XX:+HeapDumpOnOutOfMemoryError
-XX:HeapDumpPath=/path/to/dump
```

---

## 8. Profiling Tools

```bash
# JDK tools
jps         # List Java processes
jstat       # GC statistics
jmap        # Memory map
jstack      # Thread dump
jcmd        # Multi-purpose

# Usage
jstat -gc <pid> 1000      # GC stats every 1s
jmap -heap <pid>          # Heap summary
jmap -histo <pid>         # Object histogram
jstack <pid>              # Thread dump
jcmd <pid> GC.heap_dump file.hprof
```

---

## 9. Best Practices

```java
// 1. Minimize object creation in loops
// Bad
for (int i = 0; i < 1000000; i++) {
    String s = new String("test");  // Creates 1M objects
}

// Good
String s = "test";
for (int i = 0; i < 1000000; i++) {
    // Use s
}

// 2. Use primitives over wrappers
// Bad
List<Integer> list;  // Boxed integers

// Good
int[] array;  // Primitive array

// 3. StringBuilder for concatenation
// Bad
String result = "";
for (String s : strings) {
    result += s;  // Creates many String objects
}

// Good
StringBuilder sb = new StringBuilder();
for (String s : strings) {
    sb.append(s);
}
String result = sb.toString();

// 4. Pool objects when possible
// Use String.intern() for repeated strings
// Use object pools for expensive objects
```

---

## 10. Bài tập thực hành

### Bài 1: Memory Monitor
Tạo utility class monitor memory usage.

### Bài 2: Object Pool
Implement simple object pool pattern.

### Bài 3: Leak Detector
Find memory leaks trong sample code.

---

## Navigation

- [← Day 17: Design Patterns](./day-17-design-patterns.md)
- [Day 19: JVM Internals →](./day-19-jvm-internals.md)
