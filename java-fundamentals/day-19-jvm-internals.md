# Day 19: JVM Internals (Bên trong máy ảo Java)

## Mục tiêu hôm nay
- Hiểu JVM Architecture (kiến trúc máy ảo Java)
- Hiểu Class Loading (cơ chế nạp class)
- Hiểu Bytecode (mã trung gian)
- Hiểu JIT Compilation (biên dịch tức thì)
- Nắm được String Pool, Method Dispatch và các công cụ chẩn đoán

---

## Tại sao cần học cái này?

> Hãy tưởng tượng bạn lái xe mỗi ngày. Bạn **không cần** biết động cơ hoạt động thế nào để lái. Nhưng khi xe **hỏng giữa đường**, hiểu về động cơ giúp bạn **biết cách xử lý**.
>
> JVM cũng vậy:
> - Hiểu JVM giúp bạn **debug** các vấn đề performance, memory, class loading
> - Biết cách code Java **thực sự** chạy bên dưới → viết code tối ưu hơn
> - Đây là kiến thức **phân biệt** junior với senior developer
> - Phỏng vấn Senior Java position **luôn** hỏi về JVM internals

---

## 1. JVM Architecture (Kiến trúc JVM)

### 1.1. Bức tranh toàn cảnh

```
┌──────────────────────────────────────────────────────────────┐
│                     Java Application                         │
│                  (Code Java bạn viết)                        │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│                   JVM (Java Virtual Machine)                 │
│              ┌───── "Máy ảo" chạy trên mọi OS ─────┐       │
│              │                                       │       │
│  ┌───────────────────────────────────────────────────────┐  │
│  │          1. CLASS LOADER SUBSYSTEM                     │  │
│  │          (Bộ nạp Class)                                │  │
│  │                                                        │  │
│  │  .class files → Loading → Linking → Initialization    │  │
│  │                                                        │  │
│  │  Bootstrap ──→ Platform ──→ Application ClassLoader   │  │
│  └───────────────────────────────────────────────────────┘  │
│              │ Class đã nạp xong, sẵn sàng chạy            │
│              ▼                                               │
│  ┌───────────────────────────────────────────────────────┐  │
│  │          2. RUNTIME DATA AREAS                         │  │
│  │          (Vùng dữ liệu khi chạy)                      │  │
│  │                                                        │  │
│  │  ┌──────────┐ ┌──────┐ ┌──────┐ ┌────┐ ┌──────────┐ │  │
│  │  │Method    │ │ Heap │ │Stack │ │ PC │ │ Native   │ │  │
│  │  │Area      │ │      │ │(mỗi │ │Reg │ │ Method   │ │  │
│  │  │(Metaspace│ │(Chứa │ │thread│ │(mỗi│ │ Stack    │ │  │
│  │  │chứa info │ │objects│ │ có 1)│ │thrd│ │(cho code │ │  │
│  │  │về class) │ │      │ │      │ │)   │ │ native)  │ │  │
│  │  └──────────┘ └──────┘ └──────┘ └────┘ └──────────┘ │  │
│  └───────────────────────────────────────────────────────┘  │
│              │ Thực thi bytecode                             │
│              ▼                                               │
│  ┌───────────────────────────────────────────────────────┐  │
│  │          3. EXECUTION ENGINE                           │  │
│  │          (Bộ thực thi)                                 │  │
│  │                                                        │  │
│  │  Interpreter      │ JIT Compiler    │ Garbage         │  │
│  │  (Chạy từng dòng) │ (Biên dịch code │ Collector       │  │
│  │                    │  "nóng" → native)│ (Dọn rác)      │  │
│  └───────────────────────────────────────────────────────┘  │
│              │                                               │
│              ▼                                               │
│  ┌───────────────────────────────────────────────────────┐  │
│  │          4. NATIVE METHOD INTERFACE (JNI)              │  │
│  │          (Giao diện gọi code C/C++)                    │  │
│  └───────────────────────────────────────────────────────┘  │
│                                                              │
├──────────────────────────────────────────────────────────────┤
│               Native Libraries (libc, libm, ...)             │
│               (Thư viện hệ điều hành)                        │
└──────────────────────────────────────────────────────────────┘
```

### 1.2. Ví dụ đời thường — "Nhà máy sản xuất"

| Thành phần JVM | Ví dụ nhà máy | Vai trò |
|----------------|---------------|---------|
| **Class Loader** | Bộ phận nhập nguyên liệu | Đọc file .class từ disk vào bộ nhớ |
| **Method Area** | Kho bản vẽ kỹ thuật | Lưu thông tin class, method, field |
| **Heap** | Nhà kho sản phẩm | Chứa tất cả objects được tạo ra |
| **Stack** | Bàn làm việc mỗi công nhân | Mỗi thread có bàn riêng (biến local, method calls) |
| **Interpreter** | Công nhân làm thủ công | Chạy bytecode từng dòng, chậm nhưng bắt đầu ngay |
| **JIT Compiler** | Máy tự động hóa | Biên dịch code "nóng" thành native → chạy nhanh hơn |
| **GC** | Đội dọn dẹp | Tự dọn sản phẩm hỏng/không dùng |

---

## 2. Class Loading (Cơ chế nạp Class)

### 2.1. Tại sao cần hiểu Class Loading?

```
Khi bạn viết: new MyClass()
JVM phải trả lời: "MyClass ở đâu? Nạp thế nào?"

  .java file                     JVM Memory
  ┌──────────┐    javac     ┌──────────┐    ClassLoader    ┌──────────────┐
  │MyClass   │ ──────────→  │MyClass   │ ───────────────→  │ Class object │
  │.java     │   (biên dịch)│.class    │   (nạp vào JVM)   │ trong memory │
  └──────────┘              └──────────┘                    └──────────────┘
     Code bạn viết       Bytecode (mã trung gian)        Sẵn sàng sử dụng
```

### 2.2. ClassLoader Hierarchy (Phân cấp ClassLoader)

```
┌─────────────────────────────────────────────────────────────┐
│  BOOTSTRAP ClassLoader (Cha cao nhất)                       │
│  - Viết bằng C/C++ (không phải Java)                       │
│  - Nạp java.lang.*, java.util.*, ... (core Java)           │
│  - getClassLoader() trả về null                             │
├─────────────────────────────────────────────────────────────┤
│         ↓ Nếu không tìm thấy, chuyển xuống                 │
├─────────────────────────────────────────────────────────────┤
│  PLATFORM ClassLoader (trước Java 9 gọi là Extension)      │
│  - Nạp các module mở rộng của JDK                          │
│  - javax.*, java.sql.*, ...                                 │
├─────────────────────────────────────────────────────────────┤
│         ↓ Nếu không tìm thấy, chuyển xuống                 │
├─────────────────────────────────────────────────────────────┤
│  APPLICATION ClassLoader (System ClassLoader)               │
│  - Nạp class từ classpath (code bạn viết + thư viện)       │
│  - -cp hoặc CLASSPATH                                       │
├─────────────────────────────────────────────────────────────┤
│         ↓ Có thể mở rộng thêm                              │
├─────────────────────────────────────────────────────────────┤
│  CUSTOM ClassLoader (do developer tự viết)                  │
│  - Plugin systems, hot reload, encryption class...          │
└─────────────────────────────────────────────────────────────┘
```

### 2.3. Delegation Model (Mô hình ủy quyền)

> **Nguyên tắc quan trọng:** Khi cần nạp class, ClassLoader **hỏi cha trước** (parent-first).

```
App cần nạp "com.example.MyClass"

  Application ClassLoader: "Cha ơi, cha có MyClass không?"
       ↑
  Platform ClassLoader:    "Cha ơi, cha có MyClass không?"
       ↑
  Bootstrap ClassLoader:   "Không có! Con tự tìm đi."
       ↓
  Platform ClassLoader:    "Tôi cũng không có! Con tự tìm."
       ↓
  Application ClassLoader: "OK, tìm trong classpath... CÓ RỒI! Nạp!"

💡 Tại sao làm vậy?
  → Đảm bảo java.lang.String LUÔN được Bootstrap nạp
  → Không ai có thể "giả mạo" core Java classes
  → Bảo mật + nhất quán
```

### 2.4. Quá trình nạp Class — 3 bước

```
  ┌────────────────┐     ┌──────────────────────────┐     ┌─────────────────┐
  │   1. LOADING   │ ──→ │       2. LINKING          │ ──→ │3. INITIALIZATION│
  │  (Nạp file)    │     │                            │     │ (Khởi tạo)      │
  │                │     │ ┌──────────────┐           │     │                 │
  │ Đọc .class     │     │ │ Verification │ Kiểm tra  │     │ Chạy static     │
  │ từ disk/network│     │ │ (Xác minh)   │ bytecode  │     │ initializers    │
  │ vào memory     │     │ ├──────────────┤ hợp lệ    │     │ và static {}    │
  │                │     │ │ Preparation  │ Cấp phát  │     │ blocks          │
  │                │     │ │ (Chuẩn bị)   │ memory    │     │                 │
  │                │     │ ├──────────────┤ cho static │     │                 │
  │                │     │ │ Resolution   │ Giải quyết│     │                 │
  │                │     │ │ (Phân giải)  │ references│     │                 │
  │                │     │ └──────────────┘           │     │                 │
  └────────────────┘     └──────────────────────────────┘     └─────────────────┘
```

```java
public class ClassLoadingDemo {
    // Bước 3 (Initialization): static block chạy ở đây
    static {
        System.out.println("Class đang được khởi tạo!");
    }

    // Bước 2 (Preparation): staticVar được gán = 0 (default)
    // Bước 3 (Initialization): staticVar được gán = 42
    static int staticVar = 42;
}
```

### 2.5. Xem thông tin ClassLoader

```java
public class ClassLoaderInfo {
    public static void main(String[] args) {
        // Xem ClassLoader của các class khác nhau
        System.out.println("String loader: " + String.class.getClassLoader());
        // → null (Bootstrap ClassLoader, viết bằng C++)

        System.out.println("MyClass loader: " + ClassLoaderInfo.class.getClassLoader());
        // → jdk.internal.loader.ClassLoaders$AppClassLoader (Application)

        // Duyệt chuỗi parent
        ClassLoader loader = ClassLoaderInfo.class.getClassLoader();
        while (loader != null) {
            System.out.println("ClassLoader: " + loader.getClass().getName());
            loader = loader.getParent();
        }
        System.out.println("ClassLoader: Bootstrap (null)");

        // Nạp class động (Dynamic loading)
        try {
            Class<?> clazz = Class.forName("java.util.ArrayList");
            System.out.println("Đã nạp: " + clazz.getName());
        } catch (ClassNotFoundException e) {
            System.out.println("Không tìm thấy class!");
        }
    }
}
```

### 2.6. Custom ClassLoader — Khi nào cần?

```java
// Ví dụ: Nạp class từ thư mục tùy chỉnh (plugin system)
public class PluginClassLoader extends ClassLoader {

    private final String pluginDir; // Thư mục chứa plugins

    public PluginClassLoader(String pluginDir) {
        this.pluginDir = pluginDir;
    }

    @Override
    protected Class<?> findClass(String name) throws ClassNotFoundException {
        // Chuyển tên class → đường dẫn file
        // com.example.MyPlugin → com/example/MyPlugin.class
        String path = pluginDir + "/" + name.replace('.', '/') + ".class";

        try {
            // Đọc file .class thành byte array
            byte[] classData = java.nio.file.Files.readAllBytes(
                java.nio.file.Path.of(path)
            );

            // Chuyển bytes → Class object trong JVM
            return defineClass(name, classData, 0, classData.length);
        } catch (java.io.IOException e) {
            throw new ClassNotFoundException("Không tìm thấy: " + name, e);
        }
    }
}

// Sử dụng:
// PluginClassLoader loader = new PluginClassLoader("/opt/plugins");
// Class<?> pluginClass = loader.loadClass("com.plugin.MyPlugin");
// Object plugin = pluginClass.getDeclaredConstructor().newInstance();
```

> 💡 **Ứng dụng thực tế của Custom ClassLoader:**
> - **Tomcat/Jetty:** Mỗi webapp có ClassLoader riêng → isolation
> - **OSGi:** Module system với ClassLoader riêng cho mỗi bundle
> - **Hot Reload:** Spring DevTools nạp lại class khi code thay đổi
> - **Encryption:** Decrypt .class file trước khi nạp (bảo vệ source code)

---

## 3. Bytecode (Mã trung gian)

### 3.1. Bytecode là gì?

```
  Code Java bạn viết             Bytecode                    Chạy trên mọi OS
  ┌──────────────┐   javac    ┌──────────────┐    JVM     ┌─────────────────┐
  │ MyClass.java │ ────────→  │ MyClass.class│ ────────→  │ Windows ✅      │
  │              │  (biên dịch)│ (bytecode)   │  (thực thi)│ Linux   ✅      │
  │ int a = 1+2;│             │ iconst_1     │            │ macOS   ✅      │
  │              │             │ iconst_2     │            │                 │
  │              │             │ iadd         │            │ "Write Once,    │
  │              │             │ istore_1     │            │  Run Anywhere"  │
  └──────────────┘             └──────────────┘            └─────────────────┘

💡 Bytecode KHÔNG phải mã máy (native code)
   Bytecode là "ngôn ngữ" mà JVM hiểu
   JVM sẽ dịch bytecode → native code cho từng OS
```

### 3.2. Xem Bytecode bằng javap

```bash
# Bước 1: Biên dịch file .java → .class
javac Simple.java

# Bước 2: Xem bytecode
javap -c Simple.class          # Xem instructions cơ bản

# Bước 3: Xem chi tiết (constant pool, flags...)
javap -v Simple.class          # Verbose output
```

### 3.3. Ví dụ — Đọc hiểu Bytecode

```java
// Code Java
public class Calculator {
    public int add(int a, int b) {
        return a + b;
    }
}
```

```
// Bytecode (javap -c Calculator.class)
public int add(int, int);
  Code:
     0: iload_1      // Đẩy tham số 'a' (vị trí 1) lên Stack
     1: iload_2      // Đẩy tham số 'b' (vị trí 2) lên Stack
     2: iadd         // Lấy 2 số trên Stack, cộng lại, đẩy kết quả lên
     3: ireturn      // Trả về số int trên đỉnh Stack

// Minh họa Stack trong quá trình thực thi:
// Bước 0: iload_1    Stack: [a]
// Bước 1: iload_2    Stack: [a, b]
// Bước 2: iadd       Stack: [a+b]
// Bước 3: ireturn    → Trả về giá trị a+b
```

### 3.4. Bảng Bytecode Instructions thường gặp

| Nhóm | Instruction | Ý nghĩa | Ví dụ |
|------|------------|---------|-------|
| **Load** | `iload_N` | Đẩy biến int vị trí N lên Stack | `iload_1` → đẩy param 1 |
| | `aload_N` | Đẩy biến reference vị trí N | `aload_0` → đẩy `this` |
| | `iconst_N` | Đẩy hằng số int (0-5) | `iconst_3` → đẩy 3 |
| | `ldc` | Đẩy constant từ pool | `ldc "hello"` |
| **Store** | `istore_N` | Lưu int từ Stack vào biến N | `istore_1` → lưu vào var 1 |
| | `astore_N` | Lưu reference vào biến N | `astore_2` |
| **Toán học** | `iadd` | Cộng 2 int trên Stack | Stack: [3,5] → [8] |
| | `isub` | Trừ | Stack: [5,3] → [2] |
| | `imul` | Nhân | Stack: [3,4] → [12] |
| **Object** | `new` | Tạo object mới | `new #2` (class index) |
| | `invokespecial` | Gọi constructor / private method | `invokespecial <init>` |
| | `invokevirtual` | Gọi instance method (đa hình) | `invokevirtual toString` |
| | `invokestatic` | Gọi static method | `invokestatic Math.max` |
| **Control** | `if_icmpge` | If so sánh int >= thì nhảy | Branch nếu true |
| | `goto` | Nhảy vô điều kiện | Loop / branch |
| **Return** | `return` | Return void | Kết thúc method |
| | `ireturn` | Return int | Trả về int |
| | `areturn` | Return reference | Trả về object |

### 3.5. Ví dụ phức tạp hơn — Vòng lặp

```java
// Code Java
public int sum(int n) {
    int total = 0;
    for (int i = 1; i <= n; i++) {
        total += i;
    }
    return total;
}
```

```
// Bytecode
  Code:
     0: iconst_0          // Đẩy 0 lên Stack
     1: istore_2          // total = 0  (lưu vào biến local 2)
     2: iconst_1          // Đẩy 1 lên Stack
     3: istore_3          // i = 1  (lưu vào biến local 3)
     4: iload_3           // Đẩy i lên Stack      ┐
     5: iload_1           // Đẩy n lên Stack       │ Kiểm tra
     6: if_icmpgt 19      // Nếu i > n → nhảy 19   │ i <= n
     9: iload_2           // Đẩy total lên Stack   │
    10: iload_3           // Đẩy i lên Stack       │ Thân vòng lặp
    11: iadd              // total + i              │
    12: istore_2          // total = total + i      │
    13: iinc 3, 1         // i++                    │
    16: goto 4            // Quay lại bước 4        ┘
    19: iload_2           // Đẩy total lên Stack
    20: ireturn           // Return total
```

---

## 4. JIT Compilation (Biên dịch tức thì)

### 4.1. Vấn đề: Interpreter chạy chậm

```
Code Java → javac → Bytecode → Interpreter → Chạy từng dòng bytecode
                                                    ↑
                                              CHẬM! Mỗi dòng bytecode
                                              phải dịch → native code
                                              mỗi lần thực thi

Ví dụ: Vòng lặp 1 triệu lần
  → Interpreter dịch cùng 1 đoạn bytecode... 1 triệu lần!
  → Lãng phí!
```

### 4.2. Giải pháp: JIT Compiler

```
  Lần 1-999: Interpreter chạy, JVM đếm số lần gọi
                    │
                    ▼
  Lần 1000: "Method này HOT rồi!" (gọi nhiều lần)
                    │
                    ▼
         ┌─────────────────────┐
         │   JIT COMPILER      │
         │   Biên dịch bytecode│
         │   → Native code     │
         │   (mã máy thật sự)  │
         └─────────────────────┘
                    │
                    ▼
  Lần 1001+: Chạy native code trực tiếp → NHANH HƠN NHIỀU!
                    │
  ┌─────────────────┴──────────────────┐
  │ Code Cache (bộ nhớ lưu native code)│
  │ Lần sau gọi → dùng lại, không cần │
  │ biên dịch lại                      │
  └────────────────────────────────────┘
```

> 💡 **Mẹo nhớ:** JIT như "học thuộc lòng". Lần đầu đọc sách (Interpreter), nhưng nếu đọc đi đọc lại cùng 1 trang → thuộc lòng (native code) → không cần mở sách nữa.

### 4.3. Tiered Compilation (Biên dịch phân tầng)

Từ Java 8+, JVM dùng **2 compiler** kết hợp:

```
┌─────────────────────────────────────────────────────────────┐
│                   TIERED COMPILATION                         │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Tier 0: INTERPRETER (Thông dịch)                           │
│  ├── Chạy bytecode từng dòng                                │
│  ├── Không tối ưu, chậm nhất                                │
│  └── Nhưng bắt đầu NGAY lập tức (không cần chờ compile)    │
│            │                                                 │
│            ▼ (Khi method "ấm" lên)                          │
│                                                              │
│  Tier 1-3: C1 COMPILER (Client Compiler)                    │
│  ├── Biên dịch nhanh, tối ưu đơn giản                      │
│  ├── Tier 1: Không profiling                                │
│  ├── Tier 2: Profiling cơ bản                               │
│  └── Tier 3: Profiling đầy đủ (thu thập data cho C2)       │
│            │                                                 │
│            ▼ (Khi method "nóng" = gọi rất nhiều lần)        │
│                                                              │
│  Tier 4: C2 COMPILER (Server Compiler)                      │
│  ├── Biên dịch chậm hơn, nhưng tối ưu MẠNH                │
│  ├── Inlining (nhúng method nhỏ vào method lớn)            │
│  ├── Loop unrolling (mở vòng lặp)                          │
│  ├── Dead code elimination (xóa code không chạy)           │
│  └── Escape analysis (tối ưu object allocation)            │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### 4.4. JIT Optimizations — Các kỹ thuật tối ưu

#### Method Inlining (Nhúng method)

```java
// Trước inlining: gọi method → overhead
public int calculate(int x) {
    return square(x) + 1;   // Gọi method square()
}
private int square(int n) {
    return n * n;
}

// Sau inlining: JIT "copy-paste" code vào → nhanh hơn
// (JVM tự làm, bạn không cần làm gì)
public int calculate(int x) {
    return x * x + 1;       // Không cần gọi method nữa
}
```

#### Escape Analysis (Phân tích thoát)

```java
// JVM phát hiện object KHÔNG "thoát" ra khỏi method
public int getLength() {
    // point chỉ dùng trong method này → "not escaped"
    Point point = new Point(1, 2);
    return point.getX() + point.getY();
}

// JIT tối ưu: KHÔNG tạo object trên Heap, dùng Stack thay
// (gọi là "scalar replacement" — thay object bằng biến đơn)
public int getLength() {
    int x = 1;   // Trực tiếp trên Stack
    int y = 2;   // Không cần new Point()
    return x + y;
}
// → Không cần GC dọn, nhanh hơn nhiều!
```

### 4.5. JIT Options (Tham số cấu hình)

```bash
# Xem method nào được JIT compile
-XX:+PrintCompilation

# Output mẫu:
#  125   1       3       java.lang.String::hashCode (55 bytes)
#  │     │       │       │
#  │     │       │       └── Method name
#  │     │       └── Tier level (1-4)
#  │     └── Compilation ID
#  └── Timestamp (ms)

# Điều chỉnh ngưỡng compile (mặc định 10000 cho C2)
-XX:CompileThreshold=5000

# Tắt JIT (debug — app sẽ rất chậm!)
-Djava.compiler=NONE

# Chỉ dùng C1 (compile nhanh hơn, ít tối ưu hơn)
-XX:TieredStopAtLevel=1
```

---

## 5. String Pool (Vùng nhớ đặc biệt cho String)

### 5.1. String Pool hoạt động thế nào?

```
  String a = "Hello";     // Tìm trong Pool → chưa có → tạo mới trong Pool
  String b = "Hello";     // Tìm trong Pool → CÓ RỒI → trỏ đến cùng object
  String c = new String("Hello"); // LUÔN tạo object MỚI trên Heap (bỏ qua Pool)
  String d = c.intern();  // Thêm vào Pool (hoặc tìm nếu có) → trả ref trong Pool

         HEAP
  ┌──────────────────────────────────────┐
  │                                      │
  │   String Pool                        │
  │   ┌────────────────┐                 │
  │   │ "Hello" @100   │ ←── a, b, d    │
  │   │ "World" @200   │                 │
  │   └────────────────┘                 │
  │                                      │
  │   ┌────────────────┐                 │
  │   │ "Hello" @300   │ ←── c          │
  │   │ (bản copy riêng)                 │
  │   └────────────────┘                 │
  └──────────────────────────────────────┘

  a == b   → true  ✅ (cùng reference trong Pool)
  a == c   → false ❌ (khác reference: Pool vs Heap)
  a == d   → true  ✅ (d = c.intern() → trỏ về Pool)
  a.equals(c) → true ✅ (nội dung giống nhau)
```

### 5.2. Tại sao String Pool quan trọng?

```java
// Không có Pool: 10,000 "ERROR" → 10,000 objects → tốn memory
for (int i = 0; i < 10_000; i++) {
    log("ERROR", messages[i]); // "ERROR" tạo object mới mỗi lần?
}

// Có Pool: 10,000 "ERROR" → CHỈ 1 object → tiết kiệm memory
// Java tự làm điều này cho String literals ("...")
```

> 💡 **Từ Java 7+**, String Pool nằm trong **Heap** (không còn trong PermGen). Điều này có nghĩa Pool có thể mở rộng và objects trong Pool có thể bị **GC** khi không còn reference.

---

## 6. Method Dispatch (Cơ chế gọi Method)

### 6.1. Static Dispatch vs Dynamic Dispatch

```java
// STATIC DISPATCH (quyết định lúc COMPILE)
// → Method overloading (cùng tên, khác tham số)
public class Printer {
    public void print(String s) { System.out.println("String: " + s); }
    public void print(Integer i) { System.out.println("Integer: " + i); }
}

Printer p = new Printer();
p.print("hello"); // Compiler biết chắc gọi print(String) → Static
p.print(42);      // Compiler biết chắc gọi print(Integer) → Static
```

```java
// DYNAMIC DISPATCH (quyết định lúc RUNTIME)
// → Method overriding (polymorphism - đa hình)
class Animal {
    public void speak() { System.out.println("..."); }
}
class Dog extends Animal {
    @Override
    public void speak() { System.out.println("Gâu gâu!"); }
}
class Cat extends Animal {
    @Override
    public void speak() { System.out.println("Meo meo!"); }
}

Animal animal = new Dog();
animal.speak(); // Compiler chỉ biết kiểu Animal
                // JVM lúc runtime mới biết → Dog.speak() → "Gâu gâu!"
                // → Dynamic dispatch (qua vtable)
```

### 6.2. vtable — Bảng method ảo

```
  Khi JVM gọi method trên object:
  1. Lấy KIỂU THỰC SỰ của object (Dog, không phải Animal)
  2. Tìm trong vtable (bảng method) của kiểu đó
  3. Gọi method tương ứng

  Dog vtable:
  ┌──────────┬───────────────────┐
  │ speak()  │ → Dog.speak()    │ ← Override
  │ toString │ → Object.toString│ ← Thừa kế
  │ equals   │ → Object.equals  │ ← Thừa kế
  └──────────┴───────────────────┘
```

---

## 7. Runtime Data Areas (Chi tiết vùng dữ liệu)

### 7.1. Tổng hợp 5 vùng nhớ

| Vùng | Thuộc về | Chứa gì | Lỗi khi đầy |
|------|---------|---------|-------------|
| **Method Area** (Metaspace) | Chung (tất cả threads) | Class metadata, constant pool, method data | `OutOfMemoryError: Metaspace` |
| **Heap** | Chung | Objects, arrays | `OutOfMemoryError: Java heap space` |
| **Stack** | Mỗi thread riêng | Local vars, method frames, operand stack | `StackOverflowError` |
| **PC Register** | Mỗi thread riêng | Địa chỉ instruction đang thực thi | Không có lỗi riêng |
| **Native Method Stack** | Mỗi thread riêng | Cho native methods (C/C++) | `StackOverflowError` |

### 7.2. Stack Frame chi tiết

```
  Mỗi khi gọi method → tạo 1 Stack Frame mới

  Stack (Thread-1)
  ┌─────────────────────────────┐
  │  Frame: method3()           │ ← Đang thực thi (top)
  │  ├── Local Variables        │    Biến local của method3
  │  ├── Operand Stack          │    Stack tính toán (push/pop)
  │  └── Frame Data             │    Return address, exception table
  ├─────────────────────────────┤
  │  Frame: method2()           │ ← Đang chờ method3 return
  │  ├── Local Variables        │
  │  ├── Operand Stack          │
  │  └── Frame Data             │
  ├─────────────────────────────┤
  │  Frame: main()              │ ← Đang chờ method2 return
  │  ├── Local Variables        │
  │  ├── Operand Stack          │
  │  └── Frame Data             │
  └─────────────────────────────┘

  Khi method3() return → frame bị pop → method2() tiếp tục
```

---

## 8. Diagnostic Tools (Công cụ chẩn đoán)

### 8.1. JDK Built-in Tools

| Công cụ | Mục đích | Lệnh ví dụ |
|---------|---------|-------------|
| `jps` | Liệt kê Java process đang chạy | `jps -l` |
| `jstat` | GC statistics real-time | `jstat -gc <pid> 1000` |
| `jmap` | Memory map, heap dump | `jmap -histo <pid>` |
| `jstack` | Thread dump (debug deadlock) | `jstack <pid>` |
| `jcmd` | Đa năng, thay thế các tool trên | `jcmd <pid> VM.flags` |
| `jfr` | Flight Recorder (profiling chi tiết) | Xem bên dưới |

### 8.2. Java Flight Recorder (JFR) — "Hộp đen máy bay"

```bash
# Bật JFR khi khởi động app (ghi lại 60 giây)
java -XX:StartFlightRecording=duration=60s,filename=recording.jfr \
     -jar myapp.jar

# Hoặc bật trên app đang chạy
jcmd <pid> JFR.start duration=60s filename=recording.jfr

# Mở file .jfr bằng JDK Mission Control (JMC) để phân tích
# → Xem CPU, Memory, Threads, I/O, GC events
```

### 8.3. Native Memory Tracking

```bash
# Bật theo dõi native memory (ngoài Heap)
java -XX:NativeMemoryTracking=summary -jar myapp.jar

# Xem report
jcmd <pid> VM.native_memory summary

# Output mẫu:
# Total: reserved=4GB, committed=2GB
#   Java Heap:    reserved=2GB, committed=1.5GB
#   Class:        reserved=1GB, committed=200MB
#   Thread:       reserved=500MB, committed=100MB
#   ...
```

### 8.4. Workflow debug vấn đề production

```
  App chạy chậm / crash?
         │
         ▼
  1. jps -l → Tìm PID của Java process
         │
         ▼
  2. Vấn đề gì?
     ├── Memory cao?     → jstat -gc <pid> 1000
     │                     jmap -histo <pid> | head -20
     │                     (xem object nào chiếm nhiều nhất)
     │
     ├── CPU cao?        → jstack <pid> > threads.txt
     │                     (xem thread nào bận)
     │                     top -H -p <pid> (Linux: xem thread CPU)
     │
     ├── Deadlock?       → jstack <pid>
     │                     (tìm "Found one Java-level deadlock")
     │
     └── Crash OOM?      → Phân tích heap dump (.hprof)
                            Eclipse MAT hoặc VisualVM
                            (tìm object chiếm nhiều nhất → trace root cause)
```

---

## 9. JVM Flags hữu ích

### 9.1. Xem tất cả JVM flags

```bash
# Xem tất cả flags có thể cấu hình (hàng trăm flags!)
java -XX:+PrintFlagsFinal -version 2>&1 | head -30

# Chỉ xem flags đang active
java -XX:+PrintCommandLineFlags -version

# Output mẫu:
# -XX:InitialHeapSize=268435456     (256MB)
# -XX:MaxHeapSize=4294967296        (4GB)
# -XX:+UseCompressedOops
# -XX:+UseG1GC
```

### 9.2. Bảng flags production hay dùng

| Flag | Ý nghĩa | Ghi chú |
|------|---------|---------|
| `-Xms2g -Xmx2g` | Heap cố định 2GB | Tránh resize, nên bằng nhau |
| `-XX:+UseG1GC` | Dùng G1 GC | Mặc định Java 9+ |
| `-XX:MaxGCPauseMillis=200` | Target GC pause ≤200ms | G1 sẽ cố gắng tuân thủ |
| `-XX:+HeapDumpOnOutOfMemoryError` | Tạo heap dump khi OOM | BẮT BUỘC cho production |
| `-XX:HeapDumpPath=/var/log/` | Nơi lưu heap dump | Đảm bảo đủ disk space |
| `-Xlog:gc*:file=gc.log` | Ghi log GC | Để phân tích GC behavior |
| `-XX:+UseStringDeduplication` | Gộp String trùng lặp | Tiết kiệm memory (G1 only) |

---

## 10. Sai lầm thường gặp

### Sai lầm 1: Nghĩ rằng Bytecode = Native Code

```
❌ SAI: "Java chậm vì chạy bytecode"
✅ ĐÚNG: JIT compiler chuyển bytecode → native code cho "hot" methods
         → Java hiện đại performance gần bằng C/C++ cho nhiều workload

  Interpreter → Chạy chậm ban đầu (cold start)
  JIT compile  → Chạy nhanh sau khi "ấm" (warm up)
```

### Sai lầm 2: Không hiểu ClassLoader delegation → ClassNotFoundException

```java
// ❌ SAI: Tự tìm class trước, không hỏi parent
public class BrokenClassLoader extends ClassLoader {
    @Override
    protected Class<?> findClass(String name) throws ClassNotFoundException {
        // Nếu tự load java.lang.String → SecurityException!
        // Hoặc load class khác version → incompatible
        return loadFromCustomSource(name);
    }
}

// ✅ ĐÚNG: Luôn delegate cho parent trước (mặc định của ClassLoader)
// → Chỉ override findClass(), KHÔNG override loadClass()
// → loadClass() đã implement delegation model đúng cách
```

### Sai lầm 3: Benchmark không warm up JIT

```java
// ❌ SAI: Đo ngay lần đầu → Interpreter chạy → kết quả chậm giả tạo
long start = System.nanoTime();
result = heavyCalculation();    // Interpreter chạy, chưa JIT
long elapsed = System.nanoTime() - start;
System.out.println("Time: " + elapsed); // Chậm!

// ✅ ĐÚNG: Warm up trước, đo sau
// Warm up: chạy nhiều lần để JIT compile
for (int i = 0; i < 10_000; i++) {
    heavyCalculation();  // JIT sẽ compile method này
}

// Bây giờ mới đo
long start = System.nanoTime();
for (int i = 0; i < 1_000; i++) {
    result = heavyCalculation();  // Chạy native code → nhanh
}
long elapsed = System.nanoTime() - start;
System.out.println("Average: " + (elapsed / 1_000) + " ns");

// ✅ TỐT NHẤT: Dùng JMH (Java Microbenchmark Harness)
// JMH tự xử lý warm up, dead code elimination, etc.
```

---

## 11. Tóm tắt cuối ngày

| Chủ đề | Điểm chính cần nhớ |
|--------|-------------------|
| **JVM Architecture** | 4 thành phần: ClassLoader → Data Areas → Execution Engine → JNI |
| **ClassLoader** | 3 tầng: Bootstrap → Platform → Application. Parent-first delegation |
| **Class Loading** | 3 bước: Loading → Linking (Verify, Prepare, Resolve) → Initialization |
| **Bytecode** | Mã trung gian giữa Java và native. Dùng `javap -c` để xem |
| **JIT Compiler** | Biên dịch "hot code" → native. Tiered: Interpreter → C1 → C2 |
| **JIT Optimizations** | Inlining, Escape Analysis, Loop Unrolling, Dead Code Elimination |
| **String Pool** | Literal strings chia sẻ reference. `intern()` thêm vào Pool |
| **Method Dispatch** | Static (overloading, compile-time) vs Dynamic (overriding, runtime vtable) |
| **Stack Frame** | Mỗi method call = 1 frame (local vars + operand stack + frame data) |
| **JFR** | "Hộp đen" ghi lại CPU, Memory, GC, Threads. Dùng JMC để phân tích |

---

## 12. Câu hỏi phỏng vấn thường gặp 🔥

**Q1: JVM gồm những thành phần chính nào?**
> JVM có 4 thành phần: (1) Class Loader Subsystem — nạp .class files; (2) Runtime Data Areas — gồm Method Area, Heap, Stack, PC Register, Native Method Stack; (3) Execution Engine — gồm Interpreter, JIT Compiler, GC; (4) Native Method Interface (JNI) — giao tiếp với code C/C++.

**Q2: ClassLoader delegation model hoạt động thế nào? Tại sao cần?**
> Khi cần nạp class, ClassLoader hỏi parent trước (parent-first). Application → Platform → Bootstrap. Nếu parent không tìm thấy, child mới tự tìm. Điều này đảm bảo core classes (java.lang.String) LUÔN được Bootstrap nạp → bảo mật, không ai có thể giả mạo core classes.

**Q3: Bytecode là gì? Tại sao Java dùng bytecode?**
> Bytecode là mã trung gian giữa source code và native code. Java compile sang bytecode (.class) thay vì native code để đạt "Write Once, Run Anywhere" — cùng 1 file .class chạy trên mọi OS có JVM. JVM sẽ dịch bytecode → native code phù hợp với từng platform.

**Q4: JIT Compiler hoạt động thế nào? Giải thích Tiered Compilation.**
> JIT (Just-In-Time) biên dịch bytecode → native code cho "hot code" (code gọi nhiều lần). Tiered Compilation có 5 tầng: Tier 0 (Interpreter) → Tier 1-3 (C1 — compile nhanh, tối ưu đơn giản) → Tier 4 (C2 — compile chậm hơn, tối ưu mạnh: inlining, escape analysis, loop unrolling).

**Q5: String Pool hoạt động thế nào? `new String("abc")` tạo bao nhiêu objects?**
> String Pool lưu trữ String literals để tái sử dụng. `"abc"` → kiểm tra Pool, nếu chưa có thì tạo 1 object trong Pool. `new String("abc")` tạo **2 objects**: 1 trong Pool (cho literal "abc") + 1 trên Heap (do `new`). Từ Java 7+, Pool nằm trong Heap, có thể bị GC.

**Q6: Escape Analysis là gì? JVM tối ưu như thế nào?**
> Escape Analysis là kỹ thuật JIT phân tích xem object có "thoát" ra khỏi method/thread không. Nếu object không thoát (chỉ dùng local): (1) Scalar Replacement — thay object bằng biến đơn trên Stack; (2) Lock Elimination — bỏ synchronized nếu object chỉ dùng bởi 1 thread; (3) Stack Allocation — cấp phát trên Stack thay vì Heap → không cần GC.

---

## 🎓 Tổng kết khóa học Java Fundamentals

**Chúc mừng bạn đã hoàn thành 19 ngày học Java Fundamentals!**

### Recap toàn bộ hành trình

| Giai đoạn | Ngày | Chủ đề |
|-----------|------|--------|
| **Cơ bản** | 1-7 | Cú pháp, OOP, Collections, Exception Handling, Functional |
| **Trung cấp** | 8-13 | Generics, Lambda, Stream API, File I/O, DateTime, Threading |
| **Nâng cao** | 14-19 | Concurrency, CompletableFuture, Reflection, Design Patterns, Memory & GC, JVM Internals |

### Bước tiếp theo

```
  Java Fundamentals (DONE ✅)
         │
         ▼
  ┌──────────────────────────────────────────────┐
  │  Build Tools    : Maven / Gradle             │
  │  Testing        : JUnit 5, Mockito           │
  │  Web Framework  : Spring Boot                │
  │  Database       : JPA / Hibernate            │
  │  API Design     : REST / GraphQL             │
  │  Microservices  : Spring Cloud               │
  │  Containers     : Docker / Kubernetes        │
  │  CI/CD          : GitHub Actions / Jenkins   │
  └──────────────────────────────────────────────┘
```

---

## Navigation

- [← Day 18: Memory & GC](./day-18-memory-gc.md)
- [Overview →](./00-overview.md)
