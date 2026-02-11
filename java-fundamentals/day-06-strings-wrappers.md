# Day 6: Strings & Wrapper Classes (Chuỗi Ký Tự & Lớp Bọc)

## Mục tiêu hôm nay

Sau khi học xong Day 6, bạn sẽ:
- Hiểu **String** (chuỗi ký tự) hoạt động thế nào bên trong Java
- Biết tại sao String là **immutable** (bất biến — không thể thay đổi)
- Sử dụng thành thạo các **String methods** (phương thức xử lý chuỗi)
- Biết khi nào dùng **StringBuilder** thay vì String
- Hiểu **Wrapper Classes** (lớp bọc) và **Autoboxing/Unboxing** (đóng gói/mở gói tự động)
- Tránh được các **bẫy** phổ biến khi làm việc với String và Wrapper

---

## Tại sao cần học String & Wrapper?

### Ví dụ đời thường

Trong bất kỳ ứng dụng nào, **80% dữ liệu** bạn xử lý đều là **chuỗi ký tự**:
- Tên người dùng: `"Nguyễn Văn A"`
- Email: `"user@email.com"`
- Mật khẩu: `"P@ssw0rd"`
- JSON data: `"{ \"name\": \"John\" }"`
- Câu SQL: `"SELECT * FROM users WHERE id = 1"`

Nếu không hiểu String hoạt động thế nào → code của bạn sẽ **chậm**, **tốn bộ nhớ**, và có **bug khó tìm**.

---

## 1. String Class (Lớp Chuỗi)

### 1.1. 🔥 String là Immutable (Bất biến — Không thể thay đổi)

**Đây là kiến thức QUAN TRỌNG NHẤT về String, HAY GẶP trong phỏng vấn!**

**Immutable có nghĩa là:** Khi bạn tạo một String, nội dung của nó **KHÔNG BAO GIỜ** thay đổi. Mọi thao tác "thay đổi" đều tạo ra **String MỚI**.

### Ví dụ đời thường

```
String giống như CHỮ ĐƯỢC KHẮC TRÊN ĐÁ:
- Bạn KHÔNG THỂ xóa chữ trên đá
- Muốn "sửa" → phải lấy VIÊN ĐÁ MỚI và khắc lại

KHÔNG giống như viết bảng phấn:
- Viết bảng phấn có thể xóa rồi viết lại (mutable)
- StringBuilder hoạt động giống viết bảng phấn
```

```java
public class StringImmutableDemo {
    public static void main(String[] args) {

        // ===== String là IMMUTABLE =====
        String greeting = "Hello";
        System.out.println("Ban đầu: " + greeting);  // "Hello"

        // Tưởng đang "sửa" greeting, nhưng KHÔNG PHẢI!
        greeting = greeting + " World";
        // Thực tế: Java tạo String MỚI "Hello World"
        // và trỏ biến greeting sang String mới đó
        // String cũ "Hello" vẫn còn trong bộ nhớ!

        System.out.println("Sau khi nối: " + greeting);  // "Hello World"
    }
}
```

**Minh họa bộ nhớ:**

```
Bước 1: String greeting = "Hello";

  greeting ──→ [ "Hello" ]     ← String Pool (vùng nhớ đặc biệt)

Bước 2: greeting = greeting + " World";

  greeting ──→ [ "Hello World" ]   ← String MỚI được tạo
               [ "Hello" ]         ← String CŨ vẫn còn (sẽ bị GC dọn sau)
```

### 🔥 String Pool (Bể chứa chuỗi)

Java có một vùng nhớ đặc biệt gọi là **String Pool** (bể chứa chuỗi). Khi bạn tạo String bằng dấu ngoặc kép `""`, Java sẽ **kiểm tra xem chuỗi đó đã tồn tại trong pool chưa**:
- **Đã có** → Dùng lại (không tạo mới) → Tiết kiệm bộ nhớ
- **Chưa có** → Tạo mới và thêm vào pool

```java
public class StringPoolDemo {
    public static void main(String[] args) {

        // ===== Tạo bằng "" (literal) → Dùng String Pool =====
        String s1 = "Hello";       // Tạo "Hello" trong Pool
        String s2 = "Hello";       // Pool đã có "Hello" → DÙNG LẠI
        // s1 và s2 CÙNG TRỎ đến 1 object trong Pool

        // ===== Tạo bằng new → Tạo object MỚI trên Heap =====
        String s3 = new String("Hello");  // Object MỚI trên Heap
        // s3 trỏ đến object KHÁC (dù nội dung giống)

        // ===== SO SÁNH =====
        // == so sánh ĐỊA CHỈ (reference) trong bộ nhớ
        System.out.println(s1 == s2);      // true  (cùng address trong Pool)
        System.out.println(s1 == s3);      // false (khác address: Pool vs Heap)

        // .equals() so sánh NỘI DUNG
        System.out.println(s1.equals(s3)); // true  (nội dung giống nhau)
    }
}
```

**Sơ đồ bộ nhớ:**

```
┌──────────── STRING POOL ────────────┐
│                                      │
│   s1 ──→ [ "Hello" ] ←── s2        │  s1 và s2 cùng trỏ tới 1 object
│                                      │
└──────────────────────────────────────┘

┌──────────── HEAP ───────────────────┐
│                                      │
│   s3 ──→ [ "Hello" ]               │  s3 trỏ tới object KHÁC
│                                      │
└──────────────────────────────────────┘
```

⚠️ **BẪY KINH ĐIỂN:** Dùng `==` để so sánh String!

```java
// ❌ SAI: Dùng == để so sánh nội dung String
String input = new String("admin");
if (input == "admin") {              // false! Vì khác address!
    System.out.println("Đăng nhập thành công");
}

// ✅ ĐÚNG: Dùng .equals() để so sánh nội dung
if (input.equals("admin")) {         // true! So sánh NỘI DUNG
    System.out.println("Đăng nhập thành công");
}

// ✅ TỐT HƠN: Đặt literal trước để tránh NullPointerException
String username = null;
// username.equals("admin")  → NullPointerException! (gọi method trên null)
// "admin".equals(username)  → false (an toàn, không lỗi)
if ("admin".equals(username)) {
    System.out.println("Đăng nhập thành công");
}
```

### 1.2. Các cách tạo String

```java
// Cách 1: Literal (phổ biến nhất) → dùng String Pool
String s1 = "Hello";

// Cách 2: Constructor (ít dùng) → tạo object mới trên Heap
String s2 = new String("Hello");

// Cách 3: Từ mảng ký tự (char array)
char[] chars = {'H', 'e', 'l', 'l', 'o'};
String s3 = new String(chars);  // "Hello"

// Cách 4: Từ mảng byte (byte array)
byte[] bytes = {72, 101, 108, 108, 111}; // Mã ASCII của H,e,l,l,o
String s4 = new String(bytes);  // "Hello"

// Cách 5: Từ StringBuilder
StringBuilder sb = new StringBuilder("Hello");
String s5 = sb.toString();  // "Hello"
```

---

## 2. String Methods (Các phương thức xử lý chuỗi)

### Tại sao cần biết String Methods?

Vì bạn sẽ dùng chúng **hàng ngày** khi lập trình. Đây là bảng tóm tắt nhanh:

```
┌─────────────────────────────────────────────────────────┐
│  NHÓM CHỨC NĂNG           │  METHODS                   │
├────────────────────────────┼────────────────────────────┤
│  Độ dài & truy cập        │  length(), charAt()        │
│  Kiểm tra rỗng            │  isEmpty(), isBlank()      │
│  So sánh                  │  equals(), compareTo()     │
│  Tìm kiếm                 │  indexOf(), contains()     │
│  Cắt chuỗi                │  substring()               │
│  Chuyển đổi               │  toUpperCase(), trim()     │
│  Thay thế                 │  replace(), replaceAll()   │
│  Tách & nối               │  split(), join()           │
│  Định dạng                │  format(), formatted()     │
└────────────────────────────┴────────────────────────────┘
```

### 2.1. Độ dài và truy cập ký tự

```java
String str = "Hello World";

// length() — Lấy độ dài chuỗi (số ký tự)
int len = str.length();  // 11 (đếm cả khoảng trắng)

// charAt(index) — Lấy ký tự tại vị trí (index bắt đầu từ 0)
char first = str.charAt(0);   // 'H' (ký tự đầu tiên)
char sixth = str.charAt(6);   // 'W' (ký tự thứ 7)
char last = str.charAt(str.length() - 1);  // 'd' (ký tự cuối)

// ⚠️ Cẩn thận: index vượt phạm vi → StringIndexOutOfBoundsException
// char oops = str.charAt(20);  // CRASH!

// isEmpty() — Kiểm tra chuỗi rỗng (length == 0)
"".isEmpty();     // true  (chuỗi rỗng, 0 ký tự)
" ".isEmpty();    // false (có 1 ký tự: khoảng trắng)

// isBlank() (Java 11+) — Kiểm tra chuỗi rỗng HOẶC chỉ có khoảng trắng
"".isBlank();     // true  (chuỗi rỗng)
"  ".isBlank();   // true  (chỉ có khoảng trắng)
" Hi".isBlank();  // false (có chữ "Hi")
```

💡 **Mẹo nhớ:** `isEmpty()` chỉ check độ dài = 0. `isBlank()` check cả khoảng trắng (dùng trong validate form).

### 2.2. So sánh chuỗi

```java
String s1 = "Hello";
String s2 = "hello";
String s3 = "Hello";

// equals() — So sánh nội dung (PHÂN BIỆT hoa/thường)
s1.equals(s2);  // false ("Hello" ≠ "hello" → vì H ≠ h)
s1.equals(s3);  // true  ("Hello" = "Hello")

// equalsIgnoreCase() — So sánh nội dung (KHÔNG phân biệt hoa/thường)
s1.equalsIgnoreCase(s2);  // true ("Hello" = "hello" khi bỏ qua hoa/thường)

// compareTo() — So sánh thứ tự "từ điển" (lexicographic)
// Trả về: 0 (bằng), < 0 (nhỏ hơn), > 0 (lớn hơn)
"apple".compareTo("banana");  // < 0 (vì 'a' < 'b' trong bảng ASCII)
"banana".compareTo("apple");  // > 0 (vì 'b' > 'a')
"apple".compareTo("apple");   // 0 (bằng nhau)

// compareToIgnoreCase() — So sánh thứ tự bỏ qua hoa/thường
"Hello".compareToIgnoreCase("hello");  // 0 (coi như bằng nhau)
```

### 2.3. Tìm kiếm trong chuỗi

```java
String str = "Hello World, Hello Java";
//            0123456789...

// indexOf() — Tìm vị trí XUẤT HIỆN ĐẦU TIÊN
str.indexOf('o');         // 4  (ký tự 'o' đầu tiên ở index 4)
str.indexOf("Hello");     // 0  (chuỗi "Hello" bắt đầu ở index 0)
str.indexOf("Hello", 1);  // 13 (tìm "Hello" bắt đầu từ index 1 → thấy ở index 13)
str.indexOf("Python");    // -1 (KHÔNG TÌM THẤY → trả về -1)

// lastIndexOf() — Tìm vị trí XUẤT HIỆN CUỐI CÙNG
str.lastIndexOf('o');      // 20 (ký tự 'o' cuối cùng)
str.lastIndexOf("Hello");  // 13 (chuỗi "Hello" cuối cùng)

// contains() — Kiểm tra CÓ CHỨA chuỗi con hay không
str.contains("World");  // true  (có chứa "World")
str.contains("world");  // false (PHÂN BIỆT hoa/thường!)

// startsWith() — Kiểm tra BẮT ĐẦU bằng
str.startsWith("Hello");     // true
str.startsWith("World", 6);  // true (bắt đầu kiểm tra từ index 6)

// endsWith() — Kiểm tra KẾT THÚC bằng
str.endsWith("Java");   // true
str.endsWith("World");  // false
```

### 2.4. Cắt chuỗi con (Substring)

```java
String str = "Hello World";
//            01234567890
//                  ↑index 6

// substring(beginIndex) — Cắt từ beginIndex đến HẾT
str.substring(6);     // "World" (từ index 6 đến cuối)

// substring(beginIndex, endIndex) — Cắt từ begin đến end (KHÔNG bao gồm end)
str.substring(0, 5);  // "Hello" (index 0,1,2,3,4 → KHÔNG lấy index 5)
str.substring(6, 11); // "World" (index 6,7,8,9,10)
```

⚠️ **Bẫy thường gặp:** `endIndex` là **exclusive** (không bao gồm)!

```java
String str = "ABCDEF";
//            012345

str.substring(1, 3);  // "BC" (KHÔNG phải "BCD"!)
// Lấy index 1 và 2, KHÔNG lấy index 3
```

💡 **Mẹo nhớ:** Số ký tự được cắt = endIndex - beginIndex. Ví dụ: `(1, 3)` → 3 - 1 = 2 ký tự.

### 2.5. Chuyển đổi chuỗi

```java
String str = "  Hello World  ";

// toUpperCase() — Chuyển thành CHỮ HOA
str.toUpperCase();  // "  HELLO WORLD  "

// toLowerCase() — Chuyển thành chữ thường
str.toLowerCase();  // "  hello world  "

// trim() — Xóa khoảng trắng ĐẦU và CUỐI
str.trim();         // "Hello World" (xóa 2 space đầu + 2 space cuối)

// strip() (Java 11+) — Giống trim() nhưng xử lý tốt hơn ký tự Unicode
str.strip();          // "Hello World"
str.stripLeading();   // "Hello World  " (chỉ xóa khoảng trắng ĐẦU)
str.stripTrailing();  // "  Hello World" (chỉ xóa khoảng trắng CUỐI)

// replace() — Thay thế TẤT CẢ ký tự/chuỗi
"Hello".replace('l', 'x');     // "Hexxo" (thay 'l' → 'x')
"Hello".replace("ll", "LL");   // "HeLLo" (thay "ll" → "LL")

// replaceAll() — Thay thế dùng REGEX (biểu thức chính quy)
"a1b2c3".replaceAll("\\d", "X");     // "aXbXcX" (\\d = bất kỳ chữ số nào)
"  Hello   World  ".replaceAll("\\s+", " ").trim();  // "Hello World"

// replaceFirst() — Chỉ thay thế KẾT QUẢ ĐẦU TIÊN
"a1b2c3".replaceFirst("\\d", "X");   // "aXb2c3" (chỉ thay số đầu tiên)
```

⚠️ **Nhớ:** Tất cả method này đều TRẢ VỀ String MỚI. String gốc **KHÔNG BỊ THAY ĐỔI** (vì String là immutable!).

```java
String name = "hello";
name.toUpperCase();              // Trả về "HELLO" nhưng KHÔNG ai lưu!
System.out.println(name);        // Vẫn là "hello"!

name = name.toUpperCase();       // ✅ Phải GÁN LẠI vào biến!
System.out.println(name);        // "HELLO"
```

### 2.6. Tách và Nối chuỗi (Split & Join)

```java
// ===== SPLIT (Tách chuỗi thành mảng) =====

// Tách theo dấu phẩy
String csv = "apple,banana,cherry";
String[] fruits = csv.split(",");
// fruits = ["apple", "banana", "cherry"]

// Tách theo khoảng trắng (regex: \\s+ = 1 hoặc nhiều khoảng trắng)
String text = "Hello   World   Java";
String[] words = text.split("\\s+");
// words = ["Hello", "World", "Java"]

// Tách với giới hạn số phần
"a,b,c,d,e".split(",", 3);
// ["a", "b", "c,d,e"]  → Chỉ tách 3 phần, phần cuối giữ nguyên

// ===== JOIN (Nối mảng thành chuỗi) =====

// Nối bằng dấu phẩy + khoảng trắng
String joined = String.join(", ", "apple", "banana", "cherry");
// "apple, banana, cherry"

// Nối mảng
String[] arr = {"Nguyễn", "Văn", "A"};
String fullName = String.join(" ", arr);
// "Nguyễn Văn A"
```

### 2.7. Chuyển đổi kiểu dữ liệu

```java
// ===== Chuyển MỌI THỨ thành String =====
String s1 = String.valueOf(123);       // "123" (int → String)
String s2 = String.valueOf(3.14);      // "3.14" (double → String)
String s3 = String.valueOf(true);      // "true" (boolean → String)
String s4 = String.valueOf('A');       // "A" (char → String)

// Cách khác: nối với chuỗi rỗng
String s5 = "" + 123;    // "123"
String s6 = "" + 3.14;   // "3.14"

// ===== Chuyển String thành mảng =====
char[] chars = "Hello".toCharArray();
// chars = ['H', 'e', 'l', 'l', 'o']

byte[] bytes = "Hello".getBytes();
// bytes = [72, 101, 108, 108, 111]  (mã ASCII)

// ===== Kiểm tra bằng regex =====
"hello123".matches("[a-z]+\\d+");  // true (chữ thường + số)
"user@email.com".matches(".*@.*\\..*");  // true (có @ và .)
```

### 2.8. Repeat và Indent (Java 11+)

```java
// repeat() — Lặp lại chuỗi N lần
"Ha".repeat(3);     // "HaHaHa"
"-".repeat(20);     // "--------------------" (vẽ đường kẻ)
"=".repeat(50);     // Vẽ đường kẻ dài

// indent() (Java 12+) — Thêm khoảng trắng đầu mỗi dòng
String text = "Hello\nWorld";
text.indent(4);
// "    Hello\n    World\n"  (thêm 4 space đầu mỗi dòng)
```

---

## 3. StringBuilder và StringBuffer (Chuỗi có thể thay đổi)

### 3.1. Tại sao cần StringBuilder?

**Vấn đề:** Mỗi lần nối String bằng `+`, Java tạo **String MỚI**. Nếu nối trong vòng lặp → tạo **hàng nghìn** String tạm → **cực kỳ chậm và tốn bộ nhớ**!

```
Nối String bằng + trong vòng lặp:

Lần 1: "" + "a"     → tạo "a"          (1 object mới)
Lần 2: "a" + "a"    → tạo "aa"         (1 object mới, bỏ "a" cũ)
Lần 3: "aa" + "a"   → tạo "aaa"        (1 object mới, bỏ "aa" cũ)
...
Lần 1000: tạo chuỗi 1000 ký tự         (1 object mới, bỏ chuỗi 999 ký tự)

→ Tổng cộng tạo 1000 String tạm thời → LÃNG PHÍ!
```

**Giải pháp:** Dùng `StringBuilder` — chuỗi **có thể thay đổi** (mutable). Nó sửa trực tiếp trên 1 object, KHÔNG tạo object mới.

```java
// ❌ CHẬM: Nối String bằng + trong vòng lặp
String result = "";
for (int i = 0; i < 100000; i++) {
    result += i;  // Tạo String MỚI mỗi lần lặp! → ~8500ms
}

// ✅ NHANH: Dùng StringBuilder
StringBuilder sb = new StringBuilder();
for (int i = 0; i < 100000; i++) {
    sb.append(i);  // Sửa trực tiếp trên 1 object → ~5ms
}
String result2 = sb.toString();

// → StringBuilder NHANH HƠN ~1700 LẦN trong trường hợp này!
```

### Ví dụ đời thường

```
String + trong vòng lặp:
  Giống như viết thư trên giấy. Mỗi lần viết sai 1 chữ
  → phải LẤY TỜ GIẤY MỚI và chép lại TOÀN BỘ + thêm chữ mới.

StringBuilder:
  Giống như viết trên bảng trắng. Muốn thêm chữ
  → viết TIẾP vào cuối bảng. Không cần chép lại gì cả.
```

### 3.2. StringBuilder Methods (Các phương thức)

```java
StringBuilder sb = new StringBuilder("Hello");

// append() — Thêm vào CUỐI (dùng nhiều nhất)
sb.append(" ");       // "Hello "
sb.append("World");   // "Hello World"
sb.append(123);       // "Hello World123"
sb.append(true);      // "Hello World123true"

// Chaining (nối chuỗi method) — Vì append trả về chính sb
sb.append(" ").append("Java").append("!");
// "Hello World123true Java!"

// insert() — Chèn vào VỊ TRÍ chỉ định
sb.insert(0, ">>> ");  // ">>> Hello World123true Java!"
//         ↑ index 0 = đầu chuỗi

// delete() — Xóa từ beginIndex đến endIndex (exclusive)
sb.delete(0, 4);       // Xóa ">>> " (index 0,1,2,3)

// deleteCharAt() — Xóa 1 ký tự tại vị trí
sb.deleteCharAt(0);    // Xóa ký tự đầu tiên

// replace() — Thay thế đoạn từ begin đến end
sb.replace(0, 5, "Hi"); // Thay 5 ký tự đầu bằng "Hi"

// reverse() — Đảo ngược chuỗi
new StringBuilder("Hello").reverse().toString();  // "olleH"

// setCharAt() — Thay ký tự tại vị trí
sb.setCharAt(0, 'h');  // Thay ký tự đầu thành 'h'

// length() — Số ký tự hiện tại
sb.length();

// capacity() — Dung lượng bộ đệm (buffer capacity)
// Mặc định = 16 + độ dài chuỗi ban đầu
sb.capacity();

// toString() — Chuyển thành String (khi muốn dùng kết quả)
String result = sb.toString();
```

### 3.3. StringBuilder vs StringBuffer — Bảng so sánh

| Tiêu chí | StringBuilder | StringBuffer |
|----------|---------------|--------------|
| **Thread-safe?** (an toàn đa luồng) | ❌ KHÔNG | ✅ CÓ (synchronized) |
| **Tốc độ** | ✅ NHANH hơn | ❌ CHẬM hơn (vì phải đồng bộ) |
| **Khi nào dùng?** | 1 luồng (99% trường hợp) | Nhiều luồng cùng sửa 1 chuỗi |

💡 **Mẹo:** Hầu hết trường hợp, dùng **StringBuilder**. Chỉ dùng StringBuffer khi bạn chắc chắn có multi-threading.

### 3.4. Khi nào dùng String vs StringBuilder?

| Tình huống | Dùng gì? | Lý do |
|-----------|----------|-------|
| Nối 2-3 chuỗi đơn giản | `String +` | Java compiler tự tối ưu |
| Nối chuỗi trong **vòng lặp** | `StringBuilder` | Tránh tạo hàng nghìn String tạm |
| Xây dựng chuỗi phức tạp | `StringBuilder` | Có insert, delete, replace |
| Chuỗi **không thay đổi** | `String` | Immutable = an toàn |

```java
// ✅ OK: Nối đơn giản (compiler tự tối ưu thành StringBuilder)
String fullName = firstName + " " + lastName;

// ✅ PHẢI dùng StringBuilder: Nối trong vòng lặp
StringBuilder html = new StringBuilder();
html.append("<ul>\n");
for (String item : items) {
    html.append("  <li>").append(item).append("</li>\n");
}
html.append("</ul>");
String result = html.toString();
```

---

## 4. String Formatting (Định dạng chuỗi)

### 4.1. printf() và String.format()

```java
String name = "Nguyễn Văn A";
int age = 25;
double salary = 15000000.5;

// printf() — In ra console có định dạng
System.out.printf("Tên: %s, Tuổi: %d%n", name, age);
// Output: Tên: Nguyễn Văn A, Tuổi: 25

System.out.printf("Lương: %,.2f VNĐ%n", salary);
// Output: Lương: 15,000,000.50 VNĐ

// String.format() — Tạo String có định dạng (giống printf nhưng trả về String)
String info = String.format("Tên: %s, Tuổi: %d", name, age);
// info = "Tên: Nguyễn Văn A, Tuổi: 25"

// formatted() method (Java 15+) — Gọn hơn
String info2 = "Tên: %s, Tuổi: %d".formatted(name, age);
```

### 4.2. Format Specifiers (Ký hiệu định dạng)

| Ký hiệu | Kiểu dữ liệu | Ví dụ | Kết quả |
|----------|---------------|-------|---------|
| `%s` | String (chuỗi) | `String.format("%s", "Hi")` | `"Hi"` |
| `%d` | int/long (số nguyên) | `String.format("%d", 123)` | `"123"` |
| `%f` | float/double (số thực) | `String.format("%f", 3.14)` | `"3.140000"` |
| `%.2f` | 2 chữ số thập phân | `String.format("%.2f", 3.14159)` | `"3.14"` |
| `%,d` | Số có dấu phân cách nghìn | `String.format("%,d", 1000000)` | `"1,000,000"` |
| `%n` | Xuống dòng (tùy OS) | | `\n` hoặc `\r\n` |
| `%b` | boolean | `String.format("%b", true)` | `"true"` |
| `%c` | char (ký tự) | `String.format("%c", 'A')` | `"A"` |
| `%x` | Hexadecimal (hệ 16) | `String.format("%x", 255)` | `"ff"` |

### 4.3. Căn chỉnh và độ rộng

```java
// Căn phải (mặc định): %10s = chuỗi rộng 10 ký tự, đẩy sang phải
System.out.printf("|%10s|%n", "Hi");      // |        Hi|

// Căn trái: %-10s = chuỗi rộng 10 ký tự, đẩy sang trái
System.out.printf("|%-10s|%n", "Hi");     // |Hi        |

// Đệm số 0: %08d = số rộng 8 ký tự, đệm 0 phía trước
System.out.printf("|%08d|%n", 123);       // |00000123|

// Ví dụ thực tế: In bảng đẹp
System.out.printf("%-15s | %5s | %10s%n", "Tên", "Tuổi", "Lương");
System.out.printf("%-15s | %5d | %,10.0f%n", "Nguyễn Văn A", 25, 15000000.0);
System.out.printf("%-15s | %5d | %,10.0f%n", "Trần Thị B", 30, 20000000.0);
// Output:
// Tên             | Tuổi  |     Lương
// Nguyễn Văn A    |    25 | 15,000,000
// Trần Thị B      |    30 | 20,000,000
```

### 4.4. Text Blocks (Chuỗi nhiều dòng — Java 15+)

```java
// TRƯỚC Java 15: Nối chuỗi nhiều dòng RẤT XẤU
String html = "<html>\n" +
              "    <body>\n" +
              "        <h1>Hello</h1>\n" +
              "    </body>\n" +
              "</html>";

// TỪ Java 15+: Text Block — SẠCH và ĐẸP hơn nhiều!
String html2 = """
    <html>
        <body>
            <h1>Hello</h1>
        </body>
    </html>
    """;

// JSON
String json = """
    {
        "name": "Nguyễn Văn A",
        "age": 25,
        "email": "a@email.com"
    }
    """;

// SQL
String sql = """
    SELECT u.name, u.email
    FROM users u
    WHERE u.age > 18
    ORDER BY u.name
    """;

// Text Block + format
String emailBody = """
    Xin chào %s,

    Số dư tài khoản của bạn là: %,.2f VNĐ

    Trân trọng,
    Ngân hàng XYZ
    """.formatted("Nguyễn Văn A", 15000000.50);
```

---

## 5. Wrapper Classes (Lớp Bọc)

### 5.1. Tại sao cần Wrapper Classes?

**Vấn đề:** Java có 2 loại dữ liệu:
- **Primitive** (kiểu nguyên thủy): `int`, `double`, `boolean`... → KHÔNG phải object
- **Object** (đối tượng): `String`, `Integer`, `List`... → LÀ object

Một số tính năng Java **chỉ làm việc với Object**, không chấp nhận primitive:
- `List<int>` → ❌ KHÔNG được! (List chỉ chứa Object)
- `List<Integer>` → ✅ OK! (Integer là Object)

**Giải pháp:** **Wrapper Classes** — "bọc" kiểu primitive thành object.

### Ví dụ đời thường

```
Primitive (int, double...) = Tiền mặt (tiền thật, nhẹ, nhanh)
Wrapper (Integer, Double...) = Tiền trong ví điện tử (có thêm nhiều tính năng)

Bạn cần ví điện tử khi:
- Mua hàng online (chỉ nhận ví điện tử, không nhận tiền mặt)
  → Giống List<Integer> chỉ nhận Object, không nhận primitive

- Kiểm tra lịch sử giao dịch (ví điện tử có tính năng này)
  → Giống Integer có method: parseInt, compareTo, max, min...
```

### 5.2. Bảng Primitive ↔ Wrapper

| Primitive (Nguyên thủy) | Wrapper (Lớp bọc) | Kích thước |
|--------------------------|-------------------|------------|
| `byte` | `Byte` | 1 byte |
| `short` | `Short` | 2 bytes |
| `int` | `Integer` | 4 bytes |
| `long` | `Long` | 8 bytes |
| `float` | `Float` | 4 bytes |
| `double` | `Double` | 8 bytes |
| `char` | `Character` | 2 bytes |
| `boolean` | `Boolean` | ~1 byte |

💡 **Mẹo nhớ:** Tên Wrapper giống primitive nhưng viết HOA chữ đầu. Ngoại lệ: `int` → `Integer`, `char` → `Character` (tên dài hơn).

### 5.3. Autoboxing và Unboxing (Đóng gói & Mở gói tự động)

```java
// ===== AUTOBOXING: primitive → Wrapper (tự động) =====
// Java tự động "bọc" int thành Integer
Integer num = 100;      // int 100 → Integer.valueOf(100) (tự động)
Double d = 3.14;        // double 3.14 → Double.valueOf(3.14) (tự động)
Boolean flag = true;    // boolean true → Boolean.valueOf(true) (tự động)

// ===== UNBOXING: Wrapper → primitive (tự động) =====
// Java tự động "mở bọc" Integer thành int
int n = num;            // Integer → int (tự động)
double dd = d;          // Double → double (tự động)
boolean b = flag;       // Boolean → boolean (tự động)

// ===== Trong biểu thức =====
Integer a = 10;         // Autoboxing
Integer result = a + 5; // Unbox a → int, tính 10+5=15, Autobox 15 → Integer

// ===== Trong Collections =====
List<Integer> numbers = new ArrayList<>();
numbers.add(42);        // Autoboxing: int 42 → Integer 42
int first = numbers.get(0);  // Unboxing: Integer 42 → int 42
```

⚠️ **BẪY NGUY HIỂM:** Unboxing null → NullPointerException!

```java
Integer x = null;   // Wrapper có thể là null

// ❌ CRASH! Unboxing null → NullPointerException
// int y = x;       // null.intValue() → BOOM!

// ✅ Kiểm tra null trước khi unbox
int y = (x != null) ? x : 0;  // Nếu null → dùng giá trị mặc định 0
```

### 5.4. Parse (Chuyển String → số) và các Method hữu ích

```java
// ===== PARSE: Chuyển String thành số =====
int a = Integer.parseInt("123");         // String → int
double b = Double.parseDouble("3.14");   // String → double
long c = Long.parseLong("999999999");    // String → long
boolean d = Boolean.parseBoolean("true"); // String → boolean

// Parse hệ khác (hệ 16, hệ 2...)
int hex = Integer.parseInt("FF", 16);    // Hex → int = 255
int bin = Integer.parseInt("1010", 2);   // Binary → int = 10

// ===== Hằng số hữu ích =====
System.out.println(Integer.MAX_VALUE);   // 2,147,483,647 (khoảng 2.1 tỷ)
System.out.println(Integer.MIN_VALUE);   // -2,147,483,648
System.out.println(Integer.SIZE);        // 32 (bits)
System.out.println(Integer.BYTES);       // 4 (bytes)

// ===== Chuyển đổi hệ số =====
Integer.toBinaryString(10);  // "1010" (hệ 2)
Integer.toHexString(255);    // "ff" (hệ 16)
Integer.toOctalString(8);    // "10" (hệ 8)

// ===== So sánh và tính toán =====
Integer.compare(10, 20);     // -1 (10 < 20)
Integer.max(10, 20);         // 20
Integer.min(10, 20);         // 10
Integer.sum(10, 20);         // 30

// ===== Character methods (kiểm tra ký tự) =====
Character.isLetter('A');        // true (là chữ cái)
Character.isDigit('5');         // true (là chữ số)
Character.isLetterOrDigit('A'); // true
Character.isWhitespace(' ');    // true (là khoảng trắng)
Character.isUpperCase('A');     // true (chữ hoa)
Character.isLowerCase('a');     // true (chữ thường)
Character.toUpperCase('a');     // 'A' (chuyển thành hoa)
Character.toLowerCase('A');     // 'a' (chuyển thành thường)
```

### 5.5. 🔥 Integer Cache — Bẫy phỏng vấn kinh điển

Java **cache** (lưu sẵn) các Integer từ **-128 đến 127**. Khi bạn tạo Integer trong khoảng này, Java **dùng lại** object đã cache.

```java
// ===== Trong phạm vi cache: -128 đến 127 =====
Integer a = 127;    // Lấy từ cache
Integer b = 127;    // Lấy từ cache → CÙNG object với a
System.out.println(a == b);      // true (cùng address vì cùng object cache)
System.out.println(a.equals(b)); // true (cùng giá trị)

// ===== NGOÀI phạm vi cache: > 127 hoặc < -128 =====
Integer c = 128;    // Tạo object MỚI (không có cache)
Integer d = 128;    // Tạo object MỚI KHÁC
System.out.println(c == d);      // false! (khác address vì 2 object khác nhau)
System.out.println(c.equals(d)); // true  (cùng giá trị 128)
```

**Sơ đồ Integer Cache:**

```
Phạm vi cache: -128 ─────────────────── 127
                 ↓                        ↓
Cache:  [-128][-127]...[0][1][2]...[126][127]
                                          ↑
                           Integer a = 127; ─┤ Cùng trỏ tới
                           Integer b = 127; ─┘ 1 object cache

Ngoài cache: 128, 129, 130...
             Integer c = 128; → Object MỚI #1
             Integer d = 128; → Object MỚI #2 (KHÁC #1!)
```

⚠️ **Quy tắc:** LUÔN dùng `.equals()` để so sánh Wrapper objects. KHÔNG BAO GIỜ dùng `==`!

```java
// ❌ SAI: Dùng == cho Wrapper
Integer price1 = 200;
Integer price2 = 200;
if (price1 == price2) { }        // false! (vì > 127, ngoài cache)

// ✅ ĐÚNG: Dùng .equals()
if (price1.equals(price2)) { }   // true!
```

---

## 6. Sai lầm thường gặp

### Sai lầm 1: Dùng `==` thay vì `.equals()` cho String

```java
String input = new String("admin");

// ❌ SAI: == so sánh ADDRESS, không phải nội dung
if (input == "admin") {
    System.out.println("OK");  // KHÔNG chạy!
}

// ✅ ĐÚNG: .equals() so sánh NỘI DUNG
if (input.equals("admin")) {
    System.out.println("OK");  // Chạy!
}
```

### Sai lầm 2: Nối String trong vòng lặp bằng `+`

```java
// ❌ SAI: Tốn 8500ms cho 100,000 lần lặp
String result = "";
for (int i = 0; i < 100000; i++) {
    result += "a";  // Tạo String MỚI mỗi lần!
}

// ✅ ĐÚNG: Tốn 5ms cho 100,000 lần lặp
StringBuilder sb = new StringBuilder();
for (int i = 0; i < 100000; i++) {
    sb.append("a");  // Sửa trực tiếp 1 object
}
String result2 = sb.toString();
```

### Sai lầm 3: Quên rằng String method trả về String MỚI

```java
String name = "hello world";

// ❌ SAI: Gọi method nhưng KHÔNG lưu kết quả
name.toUpperCase();  // Tạo "HELLO WORLD" rồi... vứt đi!
name.trim();         // Tạo chuỗi trim rồi... vứt đi!
System.out.println(name);  // Vẫn là "hello world"!

// ✅ ĐÚNG: Phải GÁN LẠI vào biến
name = name.toUpperCase();
System.out.println(name);  // "HELLO WORLD"
```

### Sai lầm 4: Unboxing null → NullPointerException

```java
// ❌ SAI: Không kiểm tra null trước khi unbox
Integer count = null;
int value = count;  // NullPointerException! null → int???

// ✅ ĐÚNG: Kiểm tra null
int value2 = (count != null) ? count : 0;
```

### Sai lầm 5: Dùng `==` cho Integer ngoài phạm vi cache

```java
Integer a = 200;
Integer b = 200;

// ❌ SAI: == trả về false vì > 127 (ngoài cache)
if (a == b) {
    System.out.println("Bằng");  // KHÔNG chạy!
}

// ✅ ĐÚNG: .equals() so sánh giá trị
if (a.equals(b)) {
    System.out.println("Bằng");  // Chạy!
}
```

---

## 7. Tóm tắt cuối ngày

### Bảng tổng hợp kiến thức

| Khái niệm | Giải thích tiếng Việt | Điểm quan trọng |
|-----------|----------------------|-----------------|
| **String** | Chuỗi ký tự | Immutable (bất biến) |
| **String Pool** | Bể chứa chuỗi trong bộ nhớ | Literal `""` dùng Pool, `new` không dùng |
| **== vs .equals()** | So sánh địa chỉ vs nội dung | LUÔN dùng `.equals()` cho String & Wrapper |
| **StringBuilder** | Chuỗi có thể thay đổi (mutable) | Dùng khi nối chuỗi trong vòng lặp |
| **StringBuffer** | Giống StringBuilder + thread-safe | Chậm hơn, chỉ dùng khi multi-thread |
| **format()** | Định dạng chuỗi | `%s` chuỗi, `%d` số nguyên, `%.2f` số thực |
| **Text Blocks** | Chuỗi nhiều dòng (Java 15+) | `"""..."""` |
| **Wrapper Classes** | Lớp bọc kiểu nguyên thủy | `int→Integer`, `double→Double` |
| **Autoboxing** | Tự động bọc primitive → Wrapper | `Integer x = 5;` |
| **Unboxing** | Tự động mở Wrapper → primitive | `int y = x;` |
| **Integer Cache** | Cache -128 đến 127 | `==` chỉ đúng trong phạm vi cache |

### 🔥 Câu hỏi phỏng vấn thường gặp

1. **String có phải immutable không? Tại sao?**
   → CÓ. Vì lý do: an toàn (security), hiệu năng (String Pool), thread-safe. Mọi method đều trả về String MỚI.

2. **`==` và `.equals()` khác nhau thế nào khi so sánh String?**
   → `==` so sánh địa chỉ bộ nhớ (reference). `.equals()` so sánh nội dung (value). Luôn dùng `.equals()`.

3. **String Pool là gì?**
   → Vùng nhớ đặc biệt lưu trữ String literal. Nếu chuỗi đã tồn tại → dùng lại, tiết kiệm bộ nhớ.

4. **StringBuilder vs StringBuffer?**
   → StringBuilder nhanh hơn nhưng không thread-safe. StringBuffer chậm hơn nhưng thread-safe. 99% dùng StringBuilder.

5. **Autoboxing là gì? Có bẫy gì?**
   → Tự động chuyển primitive → Wrapper. Bẫy: unboxing null → NullPointerException.

6. **Integer Cache hoạt động thế nào?**
   → Cache -128 đến 127. `Integer a = 127; Integer b = 127;` → `a == b` là true. Nhưng `128` thì `==` là false.

---

## 8. Bài tập thực hành

### Bài 1: String Utilities

Tạo class `StringUtils` với các method:

```java
StringUtils.reverse("hello");              // "olleh" (đảo ngược)
StringUtils.isPalindrome("radar");         // true (đọc xuôi = đọc ngược)
StringUtils.countWords("Hello World");     // 2 (đếm số từ)
StringUtils.countVowels("hello");          // 2 (đếm nguyên âm: e, o)
StringUtils.capitalize("hello world");     // "Hello World" (viết hoa đầu mỗi từ)
```

### Bài 2: Password Validator

Tạo hàm kiểm tra mật khẩu mạnh:
- Độ dài tối thiểu **8 ký tự**
- Có ít nhất **1 chữ hoa** (A-Z)
- Có ít nhất **1 chữ thường** (a-z)
- Có ít nhất **1 chữ số** (0-9)
- Có ít nhất **1 ký tự đặc biệt** (!@#$%...)

### Bài 3: String Compression (Nén chuỗi)

Nén chuỗi bằng cách đếm ký tự liên tiếp:
```
"aaabbbcc" → "a3b3c2"
"aabcccccaaa" → "a2bc5a3"
"abc" → "abc" (nếu nén dài hơn gốc → trả về gốc)
```

### Bài 4: Number Formatter

Tạo class format số:
```
1234567.89  → "$1,234,567.89"    (tiền tệ)
0.1234      → "12.34%"           (phần trăm)
1234567890  → "(123) 456-7890"   (số điện thoại)
```

### Bài 5: Performance Test

So sánh tốc độ String `+` vs StringBuilder với 100,000 lần lặp. In ra thời gian mỗi cách.

---

## 9. Đáp án tham khảo

<details>
<summary>Bài 1: String Utilities (Click để xem)</summary>

```java
public class StringUtils {

    // Đảo ngược chuỗi
    public static String reverse(String str) {
        if (str == null) return null;
        // Dùng StringBuilder.reverse() cho nhanh
        return new StringBuilder(str).reverse().toString();
    }

    // Kiểm tra palindrome (đọc xuôi = đọc ngược)
    // Ví dụ: "radar", "A man a plan a canal Panama"
    public static boolean isPalindrome(String str) {
        if (str == null) return false;
        // Bỏ ký tự đặc biệt, chuyển thường → so sánh
        String cleaned = str.toLowerCase().replaceAll("[^a-z0-9]", "");
        return cleaned.equals(reverse(cleaned));
    }

    // Đếm số từ trong chuỗi
    public static int countWords(String str) {
        if (str == null || str.isBlank()) return 0;
        // Tách theo khoảng trắng → đếm mảng
        return str.trim().split("\\s+").length;
    }

    // Đếm nguyên âm (a, e, i, o, u)
    public static int countVowels(String str) {
        if (str == null) return 0;
        int count = 0;
        String vowels = "aeiouAEIOU"; // Cả hoa lẫn thường
        for (char c : str.toCharArray()) {
            if (vowels.indexOf(c) != -1) {
                count++;
            }
        }
        return count;
    }

    // Viết hoa chữ đầu mỗi từ
    public static String capitalize(String str) {
        if (str == null || str.isEmpty()) return str;

        StringBuilder result = new StringBuilder();
        boolean capitalizeNext = true; // Cờ: ký tự tiếp theo cần viết hoa?

        for (char c : str.toCharArray()) {
            if (Character.isWhitespace(c)) {
                capitalizeNext = true; // Sau khoảng trắng → viết hoa từ tiếp
                result.append(c);
            } else if (capitalizeNext) {
                result.append(Character.toUpperCase(c));
                capitalizeNext = false;
            } else {
                result.append(Character.toLowerCase(c));
            }
        }
        return result.toString();
    }

    public static void main(String[] args) {
        System.out.println(reverse("hello"));               // olleh
        System.out.println(isPalindrome("A man a plan a canal Panama")); // true
        System.out.println(countWords("Hello World"));       // 2
        System.out.println(countVowels("hello"));            // 2
        System.out.println(capitalize("hello world"));       // Hello World
    }
}
```
</details>

<details>
<summary>Bài 3: String Compression (Click để xem)</summary>

```java
public class StringCompressor {

    public static String compress(String str) {
        if (str == null || str.isEmpty()) return str;

        StringBuilder compressed = new StringBuilder();
        int count = 1; // Đếm ký tự liên tiếp giống nhau

        for (int i = 0; i < str.length(); i++) {
            // Nếu ký tự tiếp theo GIỐNG ký tự hiện tại → tăng count
            if (i + 1 < str.length() && str.charAt(i) == str.charAt(i + 1)) {
                count++;
            } else {
                // Ký tự tiếp theo KHÁC → ghi ký tự hiện tại + số đếm
                compressed.append(str.charAt(i));
                if (count > 1) {
                    compressed.append(count); // Chỉ ghi số khi > 1
                }
                count = 1; // Reset bộ đếm
            }
        }

        // Nếu chuỗi nén NGẮN HƠN gốc → trả về nén
        // Ngược lại → trả về gốc
        return compressed.length() < str.length()
            ? compressed.toString()
            : str;
    }

    public static void main(String[] args) {
        System.out.println(compress("aaabbbcc"));    // a3b3c2
        System.out.println(compress("aabcccccaaa")); // a2bc5a3
        System.out.println(compress("abc"));          // abc (không nén vì không ngắn hơn)
    }
}
```
</details>

<details>
<summary>Bài 5: Performance Test (Click để xem)</summary>

```java
public class PerformanceTest {

    public static void main(String[] args) {
        int iterations = 100000; // 100,000 lần lặp

        // ===== Test 1: Nối String bằng + =====
        long start = System.currentTimeMillis();
        String s = "";
        for (int i = 0; i < iterations; i++) {
            s += "a"; // Tạo String MỚI mỗi lần!
        }
        long stringTime = System.currentTimeMillis() - start;

        // ===== Test 2: Nối bằng StringBuilder =====
        start = System.currentTimeMillis();
        StringBuilder sb = new StringBuilder();
        for (int i = 0; i < iterations; i++) {
            sb.append("a"); // Sửa trực tiếp 1 object
        }
        String result = sb.toString();
        long sbTime = System.currentTimeMillis() - start;

        // ===== Kết quả =====
        System.out.println("String + concatenation: " + stringTime + "ms");
        System.out.println("StringBuilder:          " + sbTime + "ms");
        System.out.println("StringBuilder nhanh hơn " + (stringTime / Math.max(sbTime, 1)) + " lần!");

        // Output ví dụ:
        // String + concatenation: 8500ms
        // StringBuilder:          5ms
        // StringBuilder nhanh hơn 1700 lần!
    }
}
```
</details>

---

## Navigation

- [← Day 5: Exception Handling (Xử Lý Ngoại Lệ)](./day-05-exception-handling.md)
- [Day 7: Collections Basics (Bộ Sưu Tập) →](./day-07-collections-basics.md)
