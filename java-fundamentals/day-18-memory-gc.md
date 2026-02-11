# Day 18: Memory & Garbage Collection (Bộ nhớ & Dọn rác tự động)

## Mục tiêu hôm nay
- Hiểu JVM Memory model (mô hình bộ nhớ JVM)
- Hiểu Garbage Collection (cơ chế dọn rác tự động)
- Nhận diện và phòng tránh Memory Leaks (rò rỉ bộ nhớ)
- Biết cách tuning performance (tối ưu hiệu năng) với JVM options

---

## Tại sao cần học cái này?

> Hãy tưởng tượng máy tính như một **căn phòng làm việc**:
> - **Bàn làm việc (Stack)** = nơi bạn đang xử lý công việc, nhỏ nhưng truy cập nhanh
> - **Kho hàng (Heap)** = nơi chứa đồ đạc lớn, rộng nhưng tìm kiếm chậm hơn
> - **Người dọn dẹp (GC)** = tự động dọn đồ không dùng nữa
>
> Nếu không hiểu cách quản lý bộ nhớ:
> - App chạy ngày càng **chậm** (memory leak)
> - App bị **crash** do OutOfMemoryError
> - Không biết cách **debug** khi production gặp vấn đề bộ nhớ

---

## 1. JVM Memory Structure (Cấu trúc bộ nhớ JVM)

### 1.1. Bức tranh toàn cảnh

```
┌─────────────────────────────────────────────────────────┐
│                    JVM MEMORY                           │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  ┌─── HEAP (Vùng nhớ chung - chứa objects) ──────────┐ │
│  │                                                     │ │
│  │  ┌── Young Generation (Thế hệ trẻ) ─────────────┐ │ │
│  │  │  ┌──────┐  ┌───────────┐  ┌───────────┐      │ │ │
│  │  │  │ Eden │  │Survivor 0 │  │Survivor 1 │      │ │ │
│  │  │  │      │  │   (S0)    │  │   (S1)    │      │ │ │
│  │  │  │ Mới  │  │ Sống sót  │  │ Sống sót  │      │ │ │
│  │  │  │ sinh │  │  lần 1    │  │  lần 2    │      │ │ │
│  │  │  └──────┘  └───────────┘  └───────────┘      │ │ │
│  │  └───────────────────────────────────────────────┘ │ │
│  │                                                     │ │
│  │  ┌── Old Generation (Thế hệ già) ───────────────┐ │ │
│  │  │                                               │ │ │
│  │  │  Chứa objects sống lâu (qua nhiều lần GC)    │ │ │
│  │  │                                               │ │ │
│  │  └───────────────────────────────────────────────┘ │ │
│  └─────────────────────────────────────────────────────┘ │
│                                                         │
│  ┌─── NON-HEAP (Bên ngoài Heap) ────────────────────┐  │
│  │  Metaspace  │ Class metadata, method info         │  │
│  │  Code Cache │ JIT compiled code (mã biên dịch)    │  │
│  └───────────────────────────────────────────────────┘  │
│                                                         │
│  ┌─── STACK (Mỗi thread có 1 stack riêng) ──────────┐  │
│  │  - Biến local (local variables)                   │  │
│  │  - Lời gọi method (method call frames)            │  │
│  │  - Tham chiếu đến objects trên Heap               │  │
│  └───────────────────────────────────────────────────┘  │
│                                                         │
│  ┌─── NATIVE MEMORY (Bộ nhớ hệ điều hành) ─────────┐  │
│  │  - Thread stacks, Direct ByteBuffer               │  │
│  │  - JNI (Java Native Interface)                    │  │
│  └───────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────┘
```

### 1.2. Ví dụ đời thường — "Trường học"

| Vùng nhớ | Ví dụ đời thường | Đặc điểm |
|-----------|-----------------|-----------|
| **Eden** | Lớp mẫu giáo | Object mới sinh ra ở đây |
| **Survivor** | Lớp tiểu học | Object sống sót qua "kỳ thi" (GC) |
| **Old Gen** | Trường đại học | Object "lão làng", sống rất lâu |
| **Metaspace** | Thư viện trường | Lưu thông tin về các "lớp học" (class) |
| **Stack** | Bảng ghi chú cá nhân | Mỗi học sinh (thread) có riêng |

---

## 2. Stack vs Heap — Hai vùng nhớ quan trọng nhất

### 2.1. Phân biệt bằng code

```java
public class MemoryExample {
    // instanceVar → nằm trên HEAP (thuộc về object)
    private int instanceVar = 42;

    // staticVar → nằm trên METASPACE
    private static String staticVar = "shared";

    public void method() {
        // localVar → nằm trên STACK (biến cục bộ)
        int localVar = 10;

        // str (biến tham chiếu) → STACK
        // "Hello" (giá trị String) → HEAP (String Pool)
        String str = "Hello";

        // obj (biến tham chiếu) → STACK
        // new Object() (đối tượng thực) → HEAP
        Object obj = new Object();
    }
    // Khi method() kết thúc:
    // - localVar, str, obj (references) bị XÓA khỏi Stack
    // - Objects trên Heap vẫn còn → chờ GC dọn
}
```

### 2.2. Minh họa trực quan

```
         STACK (Thread-1)              HEAP
        ┌───────────────┐      ┌─────────────────────┐
        │ method() frame│      │                     │
        │  localVar: 10 │      │  ┌───────────────┐  │
        │  str: ─────────┼─────┼─→│ "Hello"       │  │
        │  obj: ─────────┼──┐  │  └───────────────┘  │
        ├───────────────┤  │  │                     │
        │ main() frame  │  │  │  ┌───────────────┐  │
        │  args: ────────┼──┼──┼─→│ String[]      │  │
        └───────────────┘  │  │  └───────────────┘  │
                           │  │                     │
                           │  │  ┌───────────────┐  │
                           └──┼─→│ Object@abc    │  │
                              │  └───────────────┘  │
                              │                     │
                              │  ┌───────────────┐  │
                              │  │MemoryExample  │  │
                              │  │ instanceVar:42│  │
                              │  └───────────────┘  │
                              └─────────────────────┘
```

### 2.3. Bảng so sánh chi tiết

| Tiêu chí | Stack | Heap |
|-----------|-------|------|
| **Chứa gì** | Biến local, tham chiếu, method frames | Objects, instance variables |
| **Cấu trúc** | LIFO (vào sau ra trước) | Không có thứ tự cố định |
| **Phạm vi** | Mỗi thread có **riêng** | **Chia sẻ** giữa tất cả threads |
| **Dọn dẹp** | Tự động khi method kết thúc | GC dọn khi không còn tham chiếu |
| **Tốc độ** | **Rất nhanh** (truy cập trực tiếp) | Chậm hơn (cần tìm kiếm) |
| **Kích thước** | Nhỏ (~512KB - 1MB/thread) | Lớn (vài trăm MB - vài GB) |
| **Lỗi khi đầy** | `StackOverflowError` | `OutOfMemoryError` |
| **Ví dụ lỗi** | Đệ quy vô hạn | Tạo quá nhiều objects |

### 2.4. Khi nào bị StackOverflowError?

```java
// ❌ Đệ quy vô hạn → Stack đầy → StackOverflowError
public static void infiniteRecursion() {
    infiniteRecursion(); // Mỗi lần gọi = thêm 1 frame vào Stack
}

// ✅ Luôn có điều kiện dừng (base case)
public static int factorial(int n) {
    if (n <= 1) return 1;          // Điều kiện dừng
    return n * factorial(n - 1);   // Stack sẽ được giải phóng khi quay về
}
```

---

## 3. Garbage Collection (GC — Bộ dọn rác tự động)

### 3.1. Ví dụ đời thường — "Dọn phòng tự động"

> Tưởng tượng bạn có **robot dọn phòng (GC):**
> - Bạn bày đồ ra dùng (tạo objects)
> - Khi bạn không cần nữa (không còn reference), robot sẽ dọn
> - Bạn **KHÔNG** cần tự dọn (không cần gọi `free()` như C/C++)
> - Nhưng robot cần **tạm dừng** mọi hoạt động để dọn (Stop-the-World)

### 3.2. GC Roots — Object nào KHÔNG bị dọn?

```
GC bắt đầu từ "gốc" (roots) và đi theo references:

   GC ROOTS (điểm bắt đầu)
   ├── Biến local trên Stack          ← đang dùng
   ├── Active threads (luồng đang chạy) ← đang chạy
   ├── Static fields (biến static)     ← luôn tồn tại
   └── JNI references                  ← code native

   Object REACHABLE (đi tới được từ Root) → GIỮ LẠI ✅
   Object UNREACHABLE (không ai trỏ tới)  → DỌN ĐI  🗑️
```

```java
public class GCRootsDemo {
    // static field → là GC Root → KHÔNG bị GC
    private static List<String> globalList = new ArrayList<>();

    public void demo() {
        // obj là biến local → là GC Root
        Object obj = new Object();  // Object@123 REACHABLE ✅

        obj = null;
        // Object@123 bây giờ UNREACHABLE → sẽ bị GC dọn 🗑️
    }
}
```

### 3.3. Vòng đời của Object trong Heap

```
                      Object mới sinh
                           │
                           ▼
               ┌───────────────────┐
               │     EDEN SPACE    │  ← Object được tạo ở đây
               └───────────────────┘
                           │
                     Minor GC chạy
                     (GC thế hệ trẻ)
                           │
              ┌────────────┴────────────┐
              │                         │
         Còn sống?                 Đã chết?
              │                         │
              ▼                         ▼
    ┌──────────────────┐          🗑️ Bị xóa
    │  SURVIVOR SPACE  │
    │  (Tuổi tăng +1)  │
    └──────────────────┘
              │
        Tuổi >= 15?
        (mặc định)
              │
    ┌─────────┴──────────┐
    │                    │
   Chưa                 Rồi
    │                    │
    ▼                    ▼
  Ở lại            ┌──────────────────┐
  Survivor         │  OLD GENERATION  │
                   │  (Thế hệ già)    │
                   └──────────────────┘
                           │
                     Major GC / Full GC
                     (chậm hơn, tốn hơn)
                           │
                      Còn sống?
                     ┌─────┴──────┐
                    Có           Không
                     │             │
                  Giữ lại       🗑️ Xóa
```

> 💡 **Mẹo nhớ:** Object như con người — sinh ra (Eden), lớn lên (Survivor), trưởng thành (Old Gen), cuối cùng bị "dọn" nếu không ai cần.

### 3.4. Các loại GC

| GC Algorithm | Đặc điểm | Khi nào dùng |
|-------------|-----------|-------------|
| **Serial GC** | 1 thread, tạm dừng app (Stop-the-World) | App nhỏ, ít RAM (<100MB) |
| **Parallel GC** | Nhiều threads, vẫn tạm dừng app | App cần throughput cao (batch processing) |
| **G1 GC** ⭐ | Chia Heap thành regions, dọn song song | **Mặc định từ Java 9**, đa năng |
| **ZGC** | Pause cực thấp (<10ms), concurrent | App cần low latency (< vài ms) |
| **Shenandoah** | Tương tự ZGC, bởi RedHat | Tương tự ZGC, có trong OpenJDK |

### 3.5. Minor GC vs Major GC vs Full GC

| Loại GC | Dọn vùng nào | Tốc độ | Khi nào xảy ra |
|---------|-------------|--------|----------------|
| **Minor GC** | Young Gen (Eden + Survivor) | **Nhanh** (vài ms) | Eden đầy |
| **Major GC** | Old Generation | **Chậm** (vài trăm ms) | Old Gen đầy |
| **Full GC** | **Toàn bộ** Heap + Metaspace | **Rất chậm** (vài giây!) | Hệ thống cần bộ nhớ gấp |

> ⚠️ **Full GC** là nguyên nhân chính gây **"đơ" ứng dụng**. Nếu thấy Full GC xảy ra liên tục → có vấn đề cần xử lý!

---

## 4. Memory Leaks (Rò rỉ bộ nhớ)

### 4.1. Memory Leak là gì?

> **Memory Leak** = Object không được dùng nữa nhưng **vẫn có reference** trỏ tới → GC không thể dọn → bộ nhớ bị chiếm mãi.

```
Bình thường:
  [Bạn dùng Object] → hết dùng → [bỏ reference] → [GC dọn] ✅

Memory Leak:
  [Bạn dùng Object] → hết dùng → [QUÊN bỏ reference] → [GC KHÔNG dọn được] ❌
                                                          ↓
                                                   Bộ nhớ tăng dần
                                                          ↓
                                                   OutOfMemoryError! 💥
```

### 4.2. Bốn nguyên nhân phổ biến

#### Nguyên nhân 1: Static Collection không bao giờ dọn

```java
public class UserCache {
    // ❌ MEMORY LEAK: List static → sống mãi → objects bên trong cũng sống mãi
    private static List<User> cache = new ArrayList<>();

    public void addUser(User user) {
        cache.add(user);
        // Không bao giờ remove → list chỉ tăng, không giảm!
    }
}

// ✅ FIX: Giới hạn size hoặc dùng cache có expiry
public class UserCache {
    private static final int MAX_SIZE = 1000;
    private static LinkedHashMap<Long, User> cache = new LinkedHashMap<>() {
        @Override
        protected boolean removeEldestEntry(Map.Entry<Long, User> eldest) {
            return size() > MAX_SIZE; // Tự xóa entry cũ nhất khi quá size
        }
    };
}
```

#### Nguyên nhân 2: Quên đóng Resources

```java
// ❌ MEMORY LEAK: FileInputStream không đóng → giữ native memory
public void readFile() {
    FileInputStream fis = new FileInputStream("data.txt");
    // Đọc file...
    // QUÊN gọi fis.close()!
}

// ✅ FIX: Luôn dùng try-with-resources
public void readFile() {
    try (FileInputStream fis = new FileInputStream("data.txt")) {
        // Đọc file...
    } // Tự động đóng dù có exception hay không
}
```

#### Nguyên nhân 3: Inner class giữ reference đến Outer class

```java
public class Activity {
    // Data nặng 10MB
    private byte[] heavyData = new byte[10_000_000];

    // ❌ Anonymous inner class giữ reference đến Activity
    // → heavyData 10MB cũng không bị GC
    public Runnable getTask() {
        return new Runnable() {
            @Override
            public void run() {
                System.out.println("Running...");
                // Không dùng heavyData nhưng vẫn giữ reference!
            }
        };
    }

    // ✅ FIX 1: Dùng static inner class (không giữ ref đến outer)
    private static class MyTask implements Runnable {
        @Override
        public void run() {
            System.out.println("Running...");
        }
    }

    // ✅ FIX 2: Dùng Lambda (không capture outer nếu không dùng)
    public Runnable getTaskFixed() {
        return () -> System.out.println("Running...");
    }
}
```

#### Nguyên nhân 4: HashMap với key bị thay đổi hashCode

```java
// ❌ Object làm key trong HashMap nhưng thay đổi hashCode sau khi put
class MutableKey {
    private String name;

    MutableKey(String name) { this.name = name; }

    public void setName(String name) { this.name = name; } // Thay đổi field

    @Override
    public int hashCode() { return name.hashCode(); }

    @Override
    public boolean equals(Object o) {
        return o instanceof MutableKey mk && mk.name.equals(name);
    }
}

Map<MutableKey, String> map = new HashMap<>();
MutableKey key = new MutableKey("Alice");
map.put(key, "value");

key.setName("Bob");          // hashCode thay đổi!
map.get(key);                // → null (không tìm thấy!)
map.remove(key);             // → null (không xóa được!)
// Entry ("Alice", "value") BỊ KẸT trong map → memory leak

// ✅ FIX: Dùng immutable objects làm key (String, Integer, enum...)
```

---

## 5. Reference Types (Các loại tham chiếu)

### 5.1. Bốn loại Reference

```
Mạnh ─────────────────────────────────────────────── Yếu
  Strong          Soft            Weak           Phantom
    │               │               │               │
    │               │               │               │
  GC KHÔNG       GC dọn khi      GC dọn ở       Luôn trả
  bao giờ dọn   HẾT bộ nhớ     NGAY lần GC     về null
    │               │            kế tiếp            │
    │               │               │               │
  Dùng hàng     Dùng cho        Dùng cho        Dùng để
  ngày          CACHE           theo dõi         cleanup
                                object           tracking
```

### 5.2. Code minh họa

```java
import java.lang.ref.*;

public class ReferenceDemo {
    public static void main(String[] args) {
        Object data = new Object(); // data là Strong Reference

        // 1. Strong Reference (Tham chiếu mạnh) — MẶC ĐỊNH
        // Object KHÔNG bị GC khi còn strong reference
        Object strong = data;

        // 2. Soft Reference (Tham chiếu mềm) — cho CACHE
        // GC chỉ dọn khi SẮP hết bộ nhớ (OutOfMemory sắp xảy ra)
        SoftReference<Object> soft = new SoftReference<>(data);
        // soft.get() → trả về object hoặc null nếu đã bị GC

        // 3. Weak Reference (Tham chiếu yếu) — cho theo dõi tạm
        // GC dọn NGAY khi không còn strong ref nào trỏ tới
        WeakReference<Object> weak = new WeakReference<>(data);
        // weak.get() → trả về object hoặc null

        // 4. Phantom Reference (Tham chiếu ảo) — cho cleanup
        // get() LUÔN trả về null, dùng với ReferenceQueue
        ReferenceQueue<Object> queue = new ReferenceQueue<>();
        PhantomReference<Object> phantom = new PhantomReference<>(data, queue);
        // phantom.get() → luôn null
        // Khi object bị GC → phantom được enqueue vào queue

        data = null;   // Bỏ strong reference
        strong = null; // Bỏ strong reference cuối cùng
        System.gc();   // Gợi ý GC chạy

        // Sau GC:
        // soft.get()    → có thể vẫn còn (nếu đủ memory)
        // weak.get()    → null (đã bị dọn)
        // phantom.get() → luôn null
    }
}
```

### 5.3. Bảng tóm tắt Reference Types

| Loại | Khi nào bị GC? | Dùng để làm gì? | Class |
|------|----------------|-----------------|-------|
| **Strong** | Không bao giờ (khi còn ref) | Sử dụng bình thường | Mặc định |
| **Soft** | Khi sắp hết memory | **Cache** (tự xóa khi cần RAM) | `SoftReference<T>` |
| **Weak** | Ngay lần GC kế tiếp | **WeakHashMap**, event listeners | `WeakReference<T>` |
| **Phantom** | Sau khi finalize | **Resource cleanup** tracking | `PhantomReference<T>` |

> 💡 **Mẹo nhớ ứng dụng thực tế:**
> - **Cache ảnh:** Dùng `SoftReference` → ảnh tự xóa khỏi cache khi cần RAM
> - **WeakHashMap:** Key là Weak → khi key không còn ai dùng, entry tự mất

---

## 6. Monitoring Memory (Giám sát bộ nhớ)

### 6.1. Kiểm tra bộ nhớ bằng code

```java
public class MemoryMonitor {
    public static void printMemoryInfo() {
        Runtime runtime = Runtime.getRuntime();

        long maxMemory = runtime.maxMemory();          // Tối đa Heap có thể dùng (-Xmx)
        long totalMemory = runtime.totalMemory();      // Heap hiện tại đã cấp phát
        long freeMemory = runtime.freeMemory();        // Phần trống trong Heap đã cấp phát
        long usedMemory = totalMemory - freeMemory;    // Phần đang dùng

        System.out.println("=== Memory Info ===");
        System.out.printf("Max Memory   : %d MB%n", maxMemory / (1024 * 1024));
        System.out.printf("Total Memory : %d MB%n", totalMemory / (1024 * 1024));
        System.out.printf("Used Memory  : %d MB%n", usedMemory / (1024 * 1024));
        System.out.printf("Free Memory  : %d MB%n", freeMemory / (1024 * 1024));
    }

    public static void main(String[] args) {
        printMemoryInfo();

        // Tạo nhiều objects
        List<byte[]> list = new ArrayList<>();
        for (int i = 0; i < 100; i++) {
            list.add(new byte[1024 * 1024]); // 1MB mỗi lần
        }

        printMemoryInfo(); // Xem memory tăng

        list.clear();      // Bỏ reference
        System.gc();       // Gợi ý GC (KHÔNG đảm bảo GC sẽ chạy!)

        printMemoryInfo(); // Xem memory giảm
    }
}
```

### 6.2. Dùng MemoryMXBean (chi tiết hơn)

```java
import java.lang.management.*;

public class DetailedMemoryMonitor {
    public static void main(String[] args) {
        MemoryMXBean memoryBean = ManagementFactory.getMemoryMXBean();

        // Heap Memory (nơi chứa objects)
        MemoryUsage heapUsage = memoryBean.getHeapMemoryUsage();
        System.out.println("=== Heap Memory ===");
        System.out.printf("  Init  : %d MB%n", heapUsage.getInit() / (1024 * 1024));
        System.out.printf("  Used  : %d MB%n", heapUsage.getUsed() / (1024 * 1024));
        System.out.printf("  Max   : %d MB%n", heapUsage.getMax() / (1024 * 1024));

        // Non-Heap Memory (Metaspace, Code Cache)
        MemoryUsage nonHeapUsage = memoryBean.getNonHeapMemoryUsage();
        System.out.println("=== Non-Heap Memory ===");
        System.out.printf("  Used  : %d MB%n", nonHeapUsage.getUsed() / (1024 * 1024));
    }
}
```

---

## 7. JVM Options (Tham số cấu hình JVM)

### 7.1. Bảng tham số hay dùng

| Tham số | Ý nghĩa | Ví dụ | Ghi chú |
|---------|---------|-------|---------|
| `-Xms` | Heap khởi tạo (initial) | `-Xms512m` | Heap bắt đầu với 512MB |
| `-Xmx` | Heap tối đa (maximum) | `-Xmx2g` | Heap không vượt quá 2GB |
| `-Xss` | Stack size mỗi thread | `-Xss512k` | Tăng nếu bị StackOverflow |
| `-XX:MetaspaceSize` | Metaspace khởi tạo | `-XX:MetaspaceSize=256m` | Cho app nhiều class |
| `-XX:MaxMetaspaceSize` | Metaspace tối đa | `-XX:MaxMetaspaceSize=512m` | Giới hạn Metaspace |

### 7.2. Chọn GC Algorithm

```bash
# G1 GC — Mặc định từ Java 9, phù hợp hầu hết app
-XX:+UseG1GC

# ZGC — Pause cực thấp (<10ms), cần Java 15+
-XX:+UseZGC

# Parallel GC — Throughput cao, cho batch processing
-XX:+UseParallelGC

# Serial GC — Cho app nhỏ, ít RAM
-XX:+UseSerialGC
```

### 7.3. Debug & Monitoring

```bash
# Bật GC logging (xem chi tiết GC hoạt động)
-Xlog:gc*:file=gc.log:time,level,tags

# Tự động tạo heap dump khi OutOfMemoryError
# → File này dùng để phân tích nguyên nhân crash
-XX:+HeapDumpOnOutOfMemoryError
-XX:HeapDumpPath=/path/to/dumps/

# In thông tin GC chi tiết
-verbose:gc
```

### 7.4. Ví dụ cấu hình cho Production

```bash
# App web trung bình (Spring Boot)
java -Xms512m -Xmx2g \
     -XX:+UseG1GC \
     -XX:+HeapDumpOnOutOfMemoryError \
     -XX:HeapDumpPath=/var/log/heapdump/ \
     -Xlog:gc*:file=/var/log/gc.log:time \
     -jar myapp.jar

# App cần low latency (Trading, Real-time)
java -Xms4g -Xmx4g \
     -XX:+UseZGC \
     -XX:+HeapDumpOnOutOfMemoryError \
     -jar trading-app.jar
```

> 💡 **Mẹo:** Đặt `-Xms` và `-Xmx` bằng nhau để tránh JVM phải resize Heap liên tục.

---

## 8. Profiling Tools (Công cụ phân tích bộ nhớ)

### 8.1. Công cụ dòng lệnh (JDK built-in)

| Công cụ | Mục đích | Lệnh ví dụ |
|---------|---------|-------------|
| `jps` | Liệt kê Java processes đang chạy | `jps -l` |
| `jstat` | Xem thống kê GC real-time | `jstat -gc <pid> 1000` (mỗi 1 giây) |
| `jmap` | Xem bản đồ bộ nhớ, tạo heap dump | `jmap -heap <pid>` |
| `jstack` | Xem thread dump (debug deadlock) | `jstack <pid>` |
| `jcmd` | Đa năng (thay thế các tool trên) | `jcmd <pid> GC.heap_dump dump.hprof` |

### 8.2. Công cụ GUI

```
┌─────────────────────────────────────────────────────────┐
│              PROFILING TOOLS PHỔ BIẾN                   │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  🆓 Miễn phí:                                          │
│  ├── VisualVM        → Xem memory, threads, CPU        │
│  ├── JConsole        → Monitor JMX, memory cơ bản      │
│  └── Eclipse MAT     → Phân tích heap dump (.hprof)    │
│                                                         │
│  💰 Trả phí:                                           │
│  ├── JProfiler       → Profiling toàn diện             │
│  ├── YourKit         → Memory + CPU profiling           │
│  └── IntelliJ Profiler → Tích hợp trong IDE            │
│                                                         │
│  📊 Workflow debug memory leak:                         │
│  1. jps → tìm PID                                      │
│  2. jstat -gc <pid> → xem GC có chạy liên tục?        │
│  3. jmap -histo <pid> → object nào chiếm nhiều nhất?   │
│  4. jcmd <pid> GC.heap_dump → tạo heap dump            │
│  5. Eclipse MAT → mở file .hprof → tìm leak           │
└─────────────────────────────────────────────────────────┘
```

---

## 9. Best Practices (Thực hành tốt cho quản lý bộ nhớ)

### 9.1. Tránh tạo Object thừa trong vòng lặp

```java
// ❌ SAI: Tạo 1 triệu String objects thừa
for (int i = 0; i < 1_000_000; i++) {
    String s = new String("test"); // Mỗi lần tạo object mới trên Heap
}

// ✅ ĐÚNG: Tái sử dụng
String s = "test"; // Dùng String Pool, chỉ 1 object
for (int i = 0; i < 1_000_000; i++) {
    // Sử dụng s
}
```

### 9.2. Dùng StringBuilder cho nối chuỗi

```java
// ❌ SAI: Mỗi += tạo String mới → N objects
String result = "";
for (String s : strings) {
    result += s;  // Tạo String mới mỗi lần! O(n²)
}

// ✅ ĐÚNG: StringBuilder — 1 object, nối nhanh O(n)
StringBuilder sb = new StringBuilder();
for (String s : strings) {
    sb.append(s);
}
String result = sb.toString();
```

### 9.3. Dùng primitive thay vì wrapper khi có thể

```java
// ❌ Mỗi Integer là 1 object trên Heap (~16 bytes)
List<Integer> list = new ArrayList<>(); // Autoboxing
for (int i = 0; i < 1_000_000; i++) {
    list.add(i); // int → Integer (tạo object)
}

// ✅ Dùng primitive array nếu không cần List API
int[] array = new int[1_000_000]; // Chỉ là vùng nhớ liên tục, không autobox
```

### 9.4. Luôn đóng Resources

```java
// ✅ try-with-resources → tự đóng
try (
    Connection conn = dataSource.getConnection();
    PreparedStatement stmt = conn.prepareStatement(sql);
    ResultSet rs = stmt.executeQuery()
) {
    while (rs.next()) {
        // Xử lý kết quả
    }
} // conn, stmt, rs tự đóng ở đây
```

---

## 10. Ví dụ thực hành tổng hợp

### Ví dụ 1: Phát hiện Memory Leak đơn giản

```java
import java.util.*;

public class LeakDetectorDemo {

    // Giả lập một cache bị leak
    static Map<String, byte[]> cache = new HashMap<>();

    public static void main(String[] args) throws InterruptedException {
        System.out.println("Bắt đầu test memory leak...");

        for (int i = 0; i < 1000; i++) {
            // Mỗi entry chiếm ~1MB
            cache.put("key-" + i, new byte[1024 * 1024]);

            if (i % 100 == 0) {
                Runtime rt = Runtime.getRuntime();
                long used = (rt.totalMemory() - rt.freeMemory()) / (1024 * 1024);
                System.out.printf("Iteration %d: Used memory = %d MB%n", i, used);
            }
        }
        // → Memory sẽ tăng liên tục vì cache không bao giờ xóa entry

        // FIX: Dùng cache có giới hạn hoặc LRU eviction
    }
}
```

### Ví dụ 2: So sánh Soft vs Weak Reference trong Cache

```java
import java.lang.ref.*;
import java.util.*;

public class ReferenceCacheDemo {
    public static void main(String[] args) {
        // Soft Cache — giữ lại cho đến khi cần memory
        Map<String, SoftReference<byte[]>> softCache = new HashMap<>();
        for (int i = 0; i < 100; i++) {
            softCache.put("soft-" + i, new SoftReference<>(new byte[1024 * 1024]));
        }

        // Weak Cache — tự dọn ngay khi GC chạy
        Map<String, WeakReference<byte[]>> weakCache = new HashMap<>();
        for (int i = 0; i < 100; i++) {
            weakCache.put("weak-" + i, new WeakReference<>(new byte[1024 * 1024]));
        }

        System.gc(); // Gợi ý GC

        // Đếm entry còn sống
        long softAlive = softCache.values().stream()
                .filter(ref -> ref.get() != null).count();
        long weakAlive = weakCache.values().stream()
                .filter(ref -> ref.get() != null).count();

        System.out.println("Soft cache còn sống: " + softAlive + "/100");
        System.out.println("Weak cache còn sống: " + weakAlive + "/100");
        // Soft → phần lớn còn sống (nếu đủ memory)
        // Weak → hầu hết bị dọn (vì không có strong ref)
    }
}
```

---

## 11. Sai lầm thường gặp

### Sai lầm 1: Gọi System.gc() trong production

```java
// ❌ SAI: Nghĩ rằng gọi System.gc() sẽ giải quyết vấn đề memory
public void processData() {
    // Xử lý xong data...
    System.gc(); // "Dọn rác đi cho tôi!"
    // KHÔNG! System.gc() chỉ là GỢI Ý, JVM có thể bỏ qua
    // Nó có thể gây Full GC → app đơ!
}

// ✅ ĐÚNG: Để GC tự quyết định, fix root cause thay vì gọi gc()
public void processData() {
    List<Data> tempList = loadData();
    process(tempList);
    tempList = null; // Bỏ reference → GC sẽ tự dọn khi cần
}
```

### Sai lầm 2: Không đặt -Xmx phù hợp

```bash
# ❌ SAI: Để mặc định (thường chỉ 256MB)
java -jar myapp.jar
# → App dùng nhiều memory → OutOfMemoryError

# ❌ SAI: Đặt quá lớn so với RAM máy
java -Xmx32g -jar myapp.jar   # Máy chỉ có 16GB RAM!
# → OS phải swap → cực chậm

# ✅ ĐÚNG: Đặt hợp lý (~70% RAM available cho Java app)
java -Xms1g -Xmx4g -jar myapp.jar  # Máy có 8GB RAM
```

### Sai lầm 3: Giữ reference không cần thiết

```java
// ❌ SAI: Method xử lý xong nhưng List vẫn giữ data cũ
public class DataProcessor {
    private List<Record> allRecords = new ArrayList<>();

    public void processDaily(List<Record> todayRecords) {
        allRecords.addAll(todayRecords); // Chỉ tăng, không giảm!
        // Sau 1 năm → allRecords chứa hàng triệu records
    }
}

// ✅ ĐÚNG: Chỉ giữ data cần thiết
public class DataProcessor {
    public void processDaily(List<Record> todayRecords) {
        // Xử lý xong rồi thôi, không lưu lại
        todayRecords.forEach(this::process);
        // todayRecords sẽ được GC sau khi method kết thúc
    }
}
```

### Sai lầm 4: Quên unregister listeners/callbacks

```java
// ❌ SAI: Đăng ký listener nhưng không gỡ → object không bị GC
eventBus.register(myListener);  // myListener bị giữ bởi eventBus
// myListener không bao giờ bị GC vì eventBus vẫn giữ reference

// ✅ ĐÚNG: Luôn unregister khi không cần
public void onDestroy() {
    eventBus.unregister(myListener); // Gỡ listener
}

// ✅ ĐÚNG hơn: Dùng WeakReference trong event bus
// Nhiều modern framework đã xử lý điều này
```

---

## 12. Tóm tắt cuối ngày

| Chủ đề | Điểm chính cần nhớ |
|--------|-------------------|
| **Heap** | Chứa objects, chia thành Young Gen (Eden + Survivor) và Old Gen |
| **Stack** | Chứa biến local + method frames, mỗi thread có riêng |
| **GC** | Tự dọn objects UNREACHABLE từ GC Roots |
| **Minor GC** | Dọn Young Gen, nhanh (vài ms) |
| **Major/Full GC** | Dọn Old Gen/toàn bộ, chậm (có thể vài giây!) |
| **G1 GC** | Mặc định Java 9+, phù hợp hầu hết ứng dụng |
| **ZGC** | Pause <10ms, cho app cần low latency |
| **Memory Leak** | Object không dùng nhưng vẫn có reference → GC không dọn được |
| **Soft Reference** | GC dọn khi sắp hết memory → dùng cho cache |
| **Weak Reference** | GC dọn ngay lần kế → dùng cho WeakHashMap |
| **-Xms / -Xmx** | Cấu hình Heap min/max, nên đặt bằng nhau trong production |
| **HeapDump** | Dùng `-XX:+HeapDumpOnOutOfMemoryError` để debug crash |

---

## 13. Câu hỏi phỏng vấn thường gặp 🔥

**Q1: Stack và Heap khác nhau như thế nào?**
> Stack chứa biến local và method frames, mỗi thread có riêng, LIFO, tự dọn khi method kết thúc. Heap chứa objects, chia sẻ giữa tất cả threads, GC dọn. Stack nhanh nhưng nhỏ (~1MB), Heap lớn nhưng chậm hơn.

**Q2: GC hoạt động như thế nào? Có bao nhiêu loại?**
> GC bắt đầu từ GC Roots (biến local, static fields, active threads), duyệt qua tất cả references. Objects REACHABLE thì giữ, UNREACHABLE thì dọn. Có 4 GC chính: Serial (1 thread), Parallel (nhiều threads), G1 (mặc định từ Java 9, chia regions), ZGC (pause <10ms).

**Q3: Memory Leak trong Java có xảy ra không? Cho ví dụ.**
> Có! Dù có GC nhưng nếu giữ reference không cần thiết → GC không dọn được. Ví dụ: static collection không clear, quên đóng resources, inner class giữ ref đến outer, listener/callback không unregister.

**Q4: Strong, Soft, Weak, Phantom Reference khác gì nhau?**
> Strong (mặc định): GC không dọn khi còn ref. Soft: GC dọn khi sắp hết memory → dùng cho cache. Weak: GC dọn ngay lần kế → WeakHashMap. Phantom: get() luôn null, dùng với ReferenceQueue để tracking cleanup.

**Q5: Minor GC, Major GC, Full GC khác gì nhau?**
> Minor GC dọn Young Gen (Eden + Survivor), nhanh vài ms, thường xuyên. Major GC dọn Old Gen, chậm hơn. Full GC dọn toàn bộ Heap + Metaspace, chậm nhất có thể vài giây, gây đơ app.

**Q6: Giải thích -Xms, -Xmx, -Xss. Khi nào cần tuning?**
> -Xms: Heap khởi tạo, -Xmx: Heap tối đa, -Xss: Stack size per thread. Cần tuning khi: OutOfMemoryError (-Xmx quá nhỏ), StackOverflowError (-Xss quá nhỏ), GC pause dài (chọn GC phù hợp), app khởi động chậm (-Xms quá nhỏ so với -Xmx).

---

## Navigation

- [← Day 17: Design Patterns](./day-17-design-patterns.md)
- [Day 19: JVM Internals →](./day-19-jvm-internals.md)
