# Day 1: Setup + Syntax + Operators + Control Flow

> Gộp từ bản 19 ngày: Day 1 (Setup) + Day 2 (Operators & Control Flow)
> 📖 Đọc sâu: [day-01-setup-syntax.md](../java-fundamentals/day-01-setup-syntax.md) | [day-02-operators-control-flow.md](../java-fundamentals/day-02-operators-control-flow.md)

---

## 1. Setup nhanh

### Cài đặt

```bash
# 1. Cài JDK 17+ (LTS)
#    Download: https://adoptium.net/ hoặc sdkman (Linux/Mac)
sdk install java 17.0.9-tem

# 2. Kiểm tra
java -version    # Runtime
javac -version   # Compiler

# 3. IDE: IntelliJ IDEA (Community Edition miễn phí)
```

### Hello World

```java
// File: HelloWorld.java (tên file PHẢI trùng tên class)
public class HelloWorld {
    public static void main(String[] args) {  // Entry point — JVM bắt đầu chạy từ đây
        System.out.println("Xin chào Java!");
    }
}
```

```bash
javac HelloWorld.java   # Compile → HelloWorld.class (bytecode)
java HelloWorld         # Chạy → "Xin chào Java!"
```

---

## 2. Biến & Kiểu dữ liệu

### 2.1. Primitive Types (8 kiểu nguyên thủy)

| Kiểu | Kích thước | Phạm vi | Dùng khi |
|------|-----------|---------|----------|
| `byte` | 1 byte | -128 → 127 | Tiết kiệm memory |
| `short` | 2 bytes | -32,768 → 32,767 | Ít dùng |
| `int` | 4 bytes | ±2.1 tỷ | **Số nguyên mặc định** |
| `long` | 8 bytes | ±9.2 × 10¹⁸ | Số rất lớn (ID, timestamp) |
| `float` | 4 bytes | ~7 chữ số thập phân | Ít dùng (dùng double) |
| `double` | 8 bytes | ~15 chữ số thập phân | **Số thực mặc định** |
| `char` | 2 bytes | Unicode character | Ký tự đơn |
| `boolean` | ~1 bit | `true` / `false` | Điều kiện |

```java
int age = 25;
long id = 123456789L;        // Suffix L cho long
double salary = 15_000_000;  // Dấu _ cho dễ đọc
boolean active = true;
char grade = 'A';            // Dấu nháy đơn
```

### 2.2. Reference Types (Kiểu tham chiếu)

```java
String name = "Nguyễn Văn A";   // String là class, KHÔNG phải primitive
int[] numbers = {1, 2, 3};      // Mảng
List<String> list = new ArrayList<>();  // Collection
```

### 2.3. var (Java 10+) — Type Inference

```java
var count = 10;              // Compiler tự suy ra: int
var name = "Hello";          // Compiler tự suy ra: String
var list = new ArrayList<String>();  // ArrayList<String>

// ⚠️ Chỉ dùng cho biến local, KHÔNG dùng cho field/parameter
```

### 2.4. Hằng số

```java
final double PI = 3.14159;         // Không thể gán lại
static final int MAX_SIZE = 100;   // Hằng số class-level (VIẾT HOA + UNDERSCORE)
```

💡 **Primitive vs Reference:**
- Primitive lưu **giá trị trực tiếp** trên Stack
- Reference lưu **địa chỉ** trỏ đến object trên Heap

---

## 3. Operators (Toán tử)

### Bảng cheat sheet

| Nhóm | Toán tử | Ví dụ |
|------|---------|-------|
| **Số học** | `+  -  *  /  %` | `10 / 3 = 3` (int chia int = int!) |
| **So sánh** | `==  !=  >  <  >=  <=` | `5 > 3 → true` |
| **Logic** | `&&  \|\|  !` | `true && false → false` |
| **Gán** | `=  +=  -=  *=  /=` | `x += 5` tương đương `x = x + 5` |
| **Tăng/giảm** | `++  --` | `i++` (tăng sau) vs `++i` (tăng trước) |
| **Bit** | `&  \|  ^  ~  <<  >>` | `5 & 3 = 1` |
| **Ternary** | `? :` | `x > 0 ? "dương" : "không dương"` |
| **instanceof** | `instanceof` | `obj instanceof String` |

### Những cái hay nhầm

```java
// ❌ Chia số nguyên → mất phần thập phân
int result = 10 / 3;          // = 3 (không phải 3.33)

// ✅ Ép kiểu để có số thực
double result2 = 10.0 / 3;    // = 3.333...
double result3 = (double) 10 / 3;  // = 3.333...

// ❌ So sánh String bằng ==
String a = new String("hello");
String b = new String("hello");
a == b;         // false! (so sánh địa chỉ)

// ✅ So sánh String bằng equals()
a.equals(b);    // true (so sánh nội dung)

// Short-circuit: && và || dừng sớm
if (obj != null && obj.getValue() > 0) { }
//  ↑ Nếu null → DỪNG, không gọi getValue() → tránh NullPointerException
```

---

## 4. Control Flow (Luồng điều khiển)

### 4.1. if / else if / else

```java
int score = 85;

if (score >= 90) {
    System.out.println("Xuất sắc");
} else if (score >= 70) {
    System.out.println("Khá");      // ← Chạy dòng này
} else {
    System.out.println("Cần cải thiện");
}
```

### 4.2. switch (Classic + Enhanced)

```java
// Classic switch
switch (day) {
    case "MON": case "TUE": case "WED": case "THU": case "FRI":
        System.out.println("Ngày làm việc");
        break;                  // ⚠️ PHẢI có break, nếu không → fall-through
    case "SAT": case "SUN":
        System.out.println("Cuối tuần");
        break;
    default:
        System.out.println("Không hợp lệ");
}

// Enhanced switch (Java 14+) — KHÔNG cần break
String type = switch (day) {
    case "MON", "TUE", "WED", "THU", "FRI" -> "Ngày làm việc";
    case "SAT", "SUN" -> "Cuối tuần";
    default -> "Không hợp lệ";
};
```

### 4.3. Vòng lặp

```java
// for — biết trước số lần
for (int i = 0; i < 5; i++) {
    System.out.println(i);  // 0, 1, 2, 3, 4
}

// for-each — duyệt collection/array
String[] names = {"An", "Bình", "Châu"};
for (String name : names) {
    System.out.println(name);
}

// while — lặp khi điều kiện đúng
int count = 0;
while (count < 3) {
    System.out.println(count);
    count++;
}

// do-while — chạy ít nhất 1 lần
do {
    System.out.println("Chạy ít nhất 1 lần");
} while (false);  // Điều kiện false nhưng vẫn chạy 1 lần

// break = thoát vòng lặp, continue = bỏ qua lượt hiện tại
for (int i = 0; i < 10; i++) {
    if (i == 3) continue;  // Bỏ qua 3
    if (i == 7) break;     // Dừng khi gặp 7
    System.out.print(i + " ");  // 0 1 2 4 5 6
}
```

---

## 5. Array (Mảng)

```java
// Khai báo + khởi tạo
int[] nums = {1, 2, 3, 4, 5};
String[] names = new String[3];   // Mảng 3 phần tử, default null

// Truy cập
nums[0] = 10;               // Gán phần tử đầu tiên
int len = nums.length;      // Độ dài mảng (field, KHÔNG phải method)

// Duyệt
for (int n : nums) { System.out.println(n); }

// Mảng 2 chiều
int[][] matrix = {
    {1, 2, 3},
    {4, 5, 6}
};
int val = matrix[1][2];  // = 6 (hàng 1, cột 2)

// Array utilities
Arrays.sort(nums);                    // Sắp xếp
Arrays.fill(names, "default");       // Điền giá trị
int idx = Arrays.binarySearch(nums, 3); // Tìm kiếm (mảng phải sorted)
String str = Arrays.toString(nums);   // In đẹp: [1, 2, 3, 4, 5]
```

---

## 6. Method (Phương thức)

```java
public class Calculator {

    // Method có return value
    public int add(int a, int b) {   // access modifier + return type + name(params)
        return a + b;
    }

    // Method không return (void)
    public void greet(String name) {
        System.out.println("Xin chào " + name);
    }

    // Varargs — số lượng tham số không cố định
    public int sum(int... numbers) {  // numbers là int[]
        int total = 0;
        for (int n : numbers) total += n;
        return total;
    }
    // sum(1, 2)  → 3
    // sum(1, 2, 3, 4) → 10

    // Method overloading — cùng tên, khác tham số
    public double add(double a, double b) {
        return a + b;
    }
    // add(1, 2) → gọi add(int, int)
    // add(1.5, 2.5) → gọi add(double, double)
}
```

---

## 7. Cheat Sheet — Sơ đồ quyết định

```
Cần lưu dữ liệu gì?
├── Số nguyên → int (hoặc long nếu rất lớn)
├── Số thực → double
├── Đúng/Sai → boolean
├── Ký tự đơn → char
├── Chuỗi văn bản → String
└── Nhiều giá trị cùng kiểu → Array hoặc Collection (Day 3)

Cần rẽ nhánh?
├── 2-3 trường hợp → if/else
└── Nhiều trường hợp → switch

Cần lặp?
├── Biết số lần → for
├── Duyệt collection → for-each
├── Không biết số lần → while
└── Chạy ít nhất 1 lần → do-while
```

---

## 8. Bài tập

1. **FizzBuzz**: In số 1→100. Chia hết 3 in "Fizz", chia hết 5 in "Buzz", cả 2 in "FizzBuzz"
2. **Đảo mảng**: Viết method nhận int[] trả về mảng đảo ngược
3. **Tìm max/min**: Viết method tìm giá trị lớn nhất và nhỏ nhất trong mảng

---

## Navigation

- [← Overview](./00-overview.md)
- [Day 2: OOP →](./day-2-oop.md)
