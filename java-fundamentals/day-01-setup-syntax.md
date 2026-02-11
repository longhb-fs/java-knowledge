# Day 1: Setup & Syntax Cơ Bản

## Mục tiêu hôm nay

Sau ngày hôm nay, bạn sẽ:
- Cài đặt được JDK (bộ công cụ Java) và IDE (phần mềm viết code)
- Viết và chạy chương trình Java đầu tiên
- Hiểu cấu trúc một chương trình Java
- Biết cách khai báo biến (variable) và các kiểu dữ liệu (data type)
- Biết cách nhập/xuất dữ liệu từ bàn phím

---

## 1. Cài đặt môi trường

### Tại sao cần cài đặt?

Để viết và chạy code Java, bạn cần 2 thứ:
- **JDK (Java Development Kit)** = "Bộ dụng cụ phát triển Java" — chứa trình biên dịch (compiler) để chuyển code bạn viết thành chương trình chạy được
- **IDE (Integrated Development Environment)** = "Môi trường phát triển tích hợp" — phần mềm giúp bạn viết code dễ dàng hơn (gợi ý code, bắt lỗi, debug)

> 💡 **Ví dụ đời thường**: JDK giống như bộ dao kéo nấu ăn, IDE giống như cái bếp hiện đại. Bạn CÓ THỂ nấu ăn không cần bếp (viết code trên Notepad), nhưng có bếp thì nấu nhanh hơn nhiều.

### 1.1. Cài đặt JDK 21

**Bước 1: Tải JDK**

Vào một trong hai link sau:
- [Oracle JDK](https://www.oracle.com/java/technologies/downloads/) — bản chính thức từ Oracle
- [Adoptium (Eclipse Temurin)](https://adoptium.net/) — bản miễn phí, mã nguồn mở (khuyên dùng)

Chọn phiên bản **JDK 21** (LTS = Long Term Support, nghĩa là được hỗ trợ lâu dài).

**Bước 2: Cài đặt**

Chạy file installer vừa tải, nhấn Next liên tục.

**Bước 3: Thiết lập biến môi trường (Environment Variables)**

Bước này để máy tính biết Java được cài ở đâu:

1. Nhấn **Windows + R** → gõ `sysdm.cpl` → nhấn Enter
2. Chọn tab **Advanced** → nhấn **Environment Variables**
3. Ở phần **System variables**, nhấn **New**:
   - Variable name: `JAVA_HOME`
   - Variable value: `C:\Program Files\Java\jdk-21` (đường dẫn nơi bạn cài JDK)
4. Tìm biến **Path** → nhấn **Edit** → nhấn **New** → thêm `%JAVA_HOME%\bin`

**Bước 4: Kiểm tra cài đặt thành công**

Mở **Command Prompt** (nhấn Windows + R → gõ `cmd`) và gõ:

```bash
java --version
# Kết quả mong muốn: java 21.0.x 2024-xx-xx LTS

javac --version
# Kết quả mong muốn: javac 21.0.x
```

> ⚠️ **Nếu báo lỗi "java is not recognized"**: Kiểm tra lại bước 3, có thể bạn gõ sai đường dẫn JAVA_HOME hoặc quên thêm vào Path. Đóng cmd và mở lại sau khi sửa.

### 1.2. Cài đặt IDE

**IntelliJ IDEA Community (Khuyến nghị cho người mới):**
1. Vào [jetbrains.com/idea](https://www.jetbrains.com/idea/download/)
2. Chọn **Community Edition** (miễn phí, đủ dùng)
3. Cài đặt và khởi động

**VS Code (Lựa chọn thay thế — nhẹ hơn):**
1. Cài [VS Code](https://code.visualstudio.com/)
2. Mở VS Code → vào Extensions (Ctrl+Shift+X)
3. Tìm và cài "Extension Pack for Java"

> 💡 **Nên dùng IntelliJ** nếu bạn mới học Java. IntelliJ được thiết kế riêng cho Java nên gợi ý code tốt hơn, bắt lỗi nhanh hơn.

---

## 2. Hello World — Chương trình đầu tiên

### Tại sao bắt đầu với Hello World?

Đây là truyền thống trong lập trình từ năm 1978. Mục đích là kiểm tra xem môi trường đã cài đúng chưa, và làm quen cú pháp cơ bản nhất.

### 2.1. Tạo project đầu tiên (IntelliJ)

1. Mở IntelliJ → chọn **New Project**
2. Chọn Language: **Java**, Build system: **IntelliJ**
3. Chọn JDK: **21**
4. Đặt tên project: `JavaFundamentals`
5. Nhấn **Create**

### 2.2. Viết chương trình đầu tiên

Tạo file `HelloWorld.java` trong thư mục `src` và gõ:

```java
// File: HelloWorld.java
// Đây là chương trình Java đầu tiên của bạn!

public class HelloWorld {
    public static void main(String[] args) {
        System.out.println("Xin chào, tôi đang học Java!");
    }
}
```

### 2.3. Giải thích từng dòng

```java
public class HelloWorld {
//│      │     └── Tên class (lớp) - PHẢI trùng với tên file (HelloWorld.java)
//│      └── "class" = khai báo 1 lớp (mọi code Java đều nằm trong class)
//└── "public" = ai cũng truy cập được (sẽ học kỹ hơn ở Day 3)

    public static void main(String[] args) {
    //│      │      │    │          └── tham số đầu vào (sẽ hiểu sau)
    //│      │      │    └── "main" = tên method đặc biệt, Java bắt đầu chạy từ đây
    //│      │      └── "void" = method này không trả về giá trị gì
    //│      └── "static" = không cần tạo object để gọi (sẽ hiểu ở Day 3)
    //└── "public" = ai cũng gọi được

        System.out.println("Xin chào, tôi đang học Java!");
        // └── In dòng chữ ra màn hình console rồi xuống dòng mới
    }
}
```

> 💡 **Mẹo nhớ**: `public static void main(String[] args)` — cứ gõ y hệt dòng này. Đây là "cửa vào" của mọi chương trình Java. Cứ nhớ "**psvm**" rồi nhấn Tab trong IntelliJ, nó sẽ tự sinh ra dòng này.

> 🔥 **Quy tắc BẮT BUỘC**: Tên file phải TRÙNG KHỚP với tên class. File `HelloWorld.java` thì class phải là `HelloWorld`. Sai tên → lỗi biên dịch!

### 2.4. Chạy chương trình

**Cách 1: Trong IDE**
- IntelliJ: Nhấn nút ▶️ (Run) bên cạnh dòng `main` → hoặc nhấn **Shift + F10**
- VS Code: Nhấn nút **Run** phía trên method `main`

**Cách 2: Command line**
```bash
# Bước 1: Biên dịch (compile) — chuyển .java thành .class
javac HelloWorld.java

# Bước 2: Chạy (run) — thực thi file .class
java HelloWorld
```

**Kết quả:**
```
Xin chào, tôi đang học Java!
```

> 💡 **Compile là gì?** Java là ngôn ngữ biên dịch. Code bạn viết (.java) → compiler chuyển thành bytecode (.class) → JVM (Java Virtual Machine) chạy bytecode đó. Nhờ vậy, code Java chạy được trên mọi hệ điều hành có cài JVM.

### 2.5. Sai lầm thường gặp

```java
// ❌ SAI: Tên class không trùng tên file
// File tên: HelloWorld.java nhưng class tên: Hello
public class Hello {  // → Lỗi! Class name phải là HelloWorld
    // ...
}

// ❌ SAI: Thiếu dấu chấm phẩy (;) cuối dòng
System.out.println("Hello")  // → Lỗi! Thiếu dấu ;

// ❌ SAI: Viết sai tên method main
public static void Main(String[] args) {  // → Lỗi! Phải là "main" chữ m thường
    // Java sẽ không tìm thấy "cửa vào" để chạy
}

// ❌ SAI: Dùng ngoặc đơn thay ngoặc nhọn
public class HelloWorld (  // → Lỗi! Phải dùng { }
    // ...
)
```

---

## 3. Cấu trúc chương trình Java

Mỗi chương trình Java đều có cấu trúc giống nhau. Hãy nhớ thứ tự:

```java
// ① Package declaration (khai báo gói) — "địa chỉ nhà" của class
//    Giống như thư mục chứa file, giúp tổ chức code
package com.example.demo;

// ② Import statements (nhập thư viện) — "mượn đồ" từ thư viện Java
//    Giống như import nguyên liệu trước khi nấu ăn
import java.util.Scanner;

// ③ Class declaration (khai báo lớp) — "bản thiết kế"
public class MyProgram {

    // ④ Fields (thuộc tính) — "đặc điểm" của class
    private String name;

    // ⑤ Constructor (hàm khởi tạo) — "cách tạo" đối tượng
    public MyProgram() {
        this.name = "Default";
    }

    // ⑥ Methods (phương thức) — "hành động" class có thể làm
    public void sayHello() {
        System.out.println("Hello, " + name);
    }

    // ⑦ Main method — "cửa vào" chương trình, Java bắt đầu chạy từ đây
    public static void main(String[] args) {
        MyProgram program = new MyProgram();
        program.sayHello();
    }
}
```

> 💡 **Ví dụ đời thường**: Class giống **bản vẽ nhà**, Object giống **căn nhà thực tế** xây từ bản vẽ đó. Fields là đặc điểm nhà (màu sơn, số phòng), Methods là hành động (mở cửa, bật đèn), Constructor là quá trình xây nhà.

> ⚠️ **Chưa hiểu hết cũng không sao!** Day 3 sẽ giải thích chi tiết về Class, Object, Constructor. Hôm nay chỉ cần nhớ cấu trúc tổng quan.

---

## 4. Variables (Biến)

### Tại sao cần biến?

Biến là **"hộp chứa"** để lưu trữ dữ liệu. Giống như bạn có nhiều hộp khác nhau: hộp đựng số, hộp đựng chữ, hộp đựng đúng/sai...

```
┌─────────┐    ┌──────────┐    ┌─────────────┐
│ age = 25│    │name="An" │    │isActive=true│
│ (int)   │    │ (String) │    │ (boolean)   │
└─────────┘    └──────────┘    └─────────────┘
   Hộp số       Hộp chữ         Hộp đúng/sai
```

### 4.1. Khai báo biến

```java
// Cú pháp (syntax):  kiểuDữLiệu  tênBiến = giáTrị;
//                     dataType    varName = value;

int age = 25;                // Số nguyên (integer) — không có phần thập phân
String name = "Nguyen Van A"; // Chuỗi ký tự (text) — luôn đặt trong dấu ""
double salary = 15000000.50;  // Số thực (decimal) — có phần thập phân
boolean isActive = true;      // Đúng/Sai (true/false)
```

> 💡 **Mẹo nhớ cú pháp**: "**Kiểu Tên Bằng Giá trị Chấm phẩy**" → `int age = 25;`

### 4.2. Quy tắc đặt tên biến

```java
// ✅ ĐÚNG — Các tên hợp lệ
int myAge;         // bắt đầu bằng chữ cái, dùng camelCase (chữ đầu tiên thường, các từ sau viết hoa)
int _count;        // bắt đầu bằng dấu gạch dưới
int $price;        // bắt đầu bằng dấu $
int studentCount;  // camelCase — CÁCH ĐẶT TÊN CHUẨN trong Java

// ❌ SAI — Các tên KHÔNG hợp lệ
int 2fast;      // ❌ Không được bắt đầu bằng SỐ
int my-age;     // ❌ Không được dùng dấu gạch ngang (-)
int my age;     // ❌ Không được có khoảng trắng
int class;      // ❌ Không được dùng từ khóa của Java (class, int, public, static...)
```

> 🔥 **Quy ước đặt tên trong Java (Naming Convention)**:
> - **Biến và method**: `camelCase` — viết thường từ đầu, viết hoa chữ cái đầu của các từ sau → `studentName`, `calculateTotal`
> - **Class**: `PascalCase` — viết hoa chữ cái đầu mỗi từ → `StudentManager`, `HelloWorld`
> - **Hằng số**: `UPPER_SNAKE_CASE` — viết HOA hết, dùng gạch dưới → `MAX_SIZE`, `PI`

### 4.3. Constants (Hằng số)

Hằng số là biến **KHÔNG THỂ thay đổi** sau khi gán giá trị. Dùng từ khóa `final`.

```java
// final = "khóa lại", không cho thay đổi
final double PI = 3.14159;        // Số pi — không bao giờ thay đổi
final int MAX_STUDENTS = 50;      // Số sinh viên tối đa
final String COMPANY = "PortX";   // Tên công ty

PI = 3.14;  // ❌ Lỗi biên dịch! Không thể thay đổi hằng số
```

> 💡 **Khi nào dùng hằng số?** Khi giá trị KHÔNG BAO GIỜ thay đổi trong suốt chương trình: tỷ giá cố định, số pi, tên công ty, giới hạn kích thước...

---

## 5. Data Types (Kiểu dữ liệu)

### Tại sao cần kiểu dữ liệu?

Java là ngôn ngữ **"strongly typed"** (kiểu mạnh) — mỗi biến PHẢI khai báo kiểu dữ liệu. Giống như mỗi hộp chứa chỉ đựng được một loại đồ: hộp số chỉ đựng số, hộp chữ chỉ đựng chữ.

### 5.1. Primitive Types (Kiểu nguyên thủy) — 8 kiểu cơ bản

Đây là các kiểu dữ liệu "gốc" của Java, lưu trực tiếp giá trị trong bộ nhớ:

#### Nhóm số nguyên (Integer — không có phần thập phân)

| Kiểu | Kích thước | Phạm vi | Khi nào dùng? | Ví dụ |
|------|-----------|---------|---------------|-------|
| `byte` | 1 byte | -128 → 127 | Hiếm dùng, tiết kiệm bộ nhớ | `byte tuoi = 25;` |
| `short` | 2 bytes | -32,768 → 32,767 | Hiếm dùng | `short nam = 2024;` |
| `int` | 4 bytes | ~-2.1 tỷ → ~2.1 tỷ | **DÙNG NHIỀU NHẤT** cho số nguyên | `int luong = 15000000;` |
| `long` | 8 bytes | Rất lớn (~±9.2 × 10^18) | Số rất lớn (dân số, tiền tệ lớn) | `long danSo = 8000000000L;` |

#### Nhóm số thực (Floating-point — có phần thập phân)

| Kiểu | Kích thước | Độ chính xác | Khi nào dùng? | Ví dụ |
|------|-----------|-------------|---------------|-------|
| `float` | 4 bytes | ~6-7 chữ số | Hiếm dùng | `float gia = 19.99f;` |
| `double` | 8 bytes | ~15 chữ số | **DÙNG NHIỀU NHẤT** cho số thực | `double pi = 3.14159;` |

#### Kiểu ký tự và logic

| Kiểu | Kích thước | Mô tả | Ví dụ |
|------|-----------|-------|-------|
| `char` | 2 bytes | MỘT ký tự, đặt trong dấu nháy đơn `' '` | `char xepLoai = 'A';` |
| `boolean` | 1 bit | Chỉ có 2 giá trị: `true` hoặc `false` | `boolean daThanhToan = true;` |

> 💡 **Mẹo nhớ**: 90% trường hợp bạn chỉ cần nhớ 4 kiểu: **int** (số nguyên), **double** (số thực), **boolean** (đúng/sai), **String** (chuỗi chữ — kiểu tham chiếu, không phải primitive).

### 5.2. Ví dụ thực tế

```java
public class DataTypesDemo {
    public static void main(String[] args) {
        // === SỐ NGUYÊN ===
        byte tuoi = 25;                    // Tuổi (0-127 là đủ)
        short namSinh = 1999;              // Năm sinh
        int luongThang = 15000000;         // Lương tháng (VND)
        long danSoTheGioi = 8000000000L;   // ⚠️ Phải có chữ L ở cuối!

        // === SỐ THỰC ===
        float diemTrungBinh = 8.5f;        // ⚠️ Phải có chữ f ở cuối!
        double pi = 3.14159265359;         // Số pi

        // === KÝ TỰ ===
        char xepLoai = 'A';               // ⚠️ Dùng nháy đơn ' ' cho char
        char kyTuUnicode = '\u0041';       // Cũng là 'A' (mã Unicode)

        // === LOGIC ===
        boolean dangHocJava = true;        // Đúng!
        boolean daRanh = false;            // Chưa rảnh!

        // In ra console để kiểm tra
        System.out.println("Tuổi: " + tuoi);
        System.out.println("Lương: " + luongThang + " VND");
        System.out.println("Dân số thế giới: " + danSoTheGioi);
        System.out.println("Điểm TB: " + diemTrungBinh);
        System.out.println("Xếp loại: " + xepLoai);
        System.out.println("Đang học Java? " + dangHocJava);
    }
}
```

### 5.3. Sai lầm thường gặp với kiểu dữ liệu

```java
// ❌ SAI: Quên chữ L khi dùng long
long soLon = 8000000000;   // Lỗi! Java hiểu đây là int, nhưng giá trị vượt quá int
long soLon = 8000000000L;  // ✅ ĐÚNG — thêm L

// ❌ SAI: Quên chữ f khi dùng float
float gia = 19.99;    // Lỗi! Java hiểu 19.99 là double, không tự chuyển thành float
float gia = 19.99f;   // ✅ ĐÚNG — thêm f

// ❌ SAI: Dùng nháy kép cho char
char c = "A";    // Lỗi! Nháy kép "" là String, không phải char
char c = 'A';    // ✅ ĐÚNG — nháy đơn '' cho char

// ❌ SAI: Dùng nháy đơn cho String
String ten = 'An';     // Lỗi!
String ten = "An";     // ✅ ĐÚNG — nháy kép "" cho String
```

### 5.4. Reference Types (Kiểu tham chiếu)

Khác với primitive (lưu trực tiếp giá trị), reference type lưu **"địa chỉ"** trỏ đến vùng nhớ chứa dữ liệu.

```java
// String — chuỗi ký tự (DÙNG RẤT NHIỀU)
String hoTen = "Nguyễn Văn A";    // Chuỗi có nội dung
String chuoiRong = "";             // Chuỗi rỗng (có tồn tại nhưng không có ký tự)
String chuoiNull = null;           // null = "không có gì cả" (chưa trỏ đến đâu)

// Array — mảng (danh sách có kích thước cố định)
int[] danhSachDiem = {8, 9, 7, 10, 6};       // Mảng 5 phần tử
String[] danhSachTen = new String[3];          // Mảng 3 ô trống, chưa gán giá trị

// Object — đối tượng (sẽ học kỹ ở Day 3)
Scanner scanner = new Scanner(System.in);      // Tạo đối tượng Scanner để đọc input
```

> 💡 **Primitive vs Reference — khác nhau ở đâu?**
> - `int a = 5;` → Biến `a` chứa **trực tiếp** giá trị 5
> - `String s = "Hello";` → Biến `s` chứa **địa chỉ** trỏ đến nơi lưu "Hello" trong bộ nhớ

### 5.5. Type Casting (Ép kiểu) — chuyển đổi kiểu dữ liệu

```java
// === WIDENING (Mở rộng) — TỰ ĐỘNG — nhỏ → lớn ===
// Không mất dữ liệu, Java tự chuyển
int soNguyen = 100;
long soLon = soNguyen;       // int → long: OK tự động
double soThuc = soLon;       // long → double: OK tự động
// Thứ tự: byte → short → int → long → float → double

// === NARROWING (Thu hẹp) — THỦ CÔNG — lớn → nhỏ ===
// CÓ THỂ mất dữ liệu, phải cast rõ ràng
double pi = 9.78;
int soNguyen2 = (int) pi;    // soNguyen2 = 9 (mất phần .78!)
System.out.println(soNguyen2); // In ra: 9

// ⚠️ CẨN THẬN: Overflow (tràn số)
int soLon2 = 130;
byte soBe = (byte) soLon2;   // soBe = -126 (!!) vì byte chỉ chứa -128 đến 127
System.out.println(soBe);     // In ra: -126 — KẾT QUẢ SAI!
```

> ⚠️ **Cẩn thận khi ép kiểu từ lớn sang nhỏ**: Luôn kiểm tra giá trị có nằm trong phạm vi cho phép không. Overflow là bug khó phát hiện!

---

## 6. Nhập xuất cơ bản (Input/Output)

### Tại sao cần?

Chương trình cần **giao tiếp** với người dùng: nhận dữ liệu từ bàn phím (Input) và hiển thị kết quả ra màn hình (Output).

### 6.1. Output (Xuất — hiển thị ra màn hình)

```java
// println = "print line" — in ra rồi XUỐNG DÒNG
System.out.println("Dòng 1");
System.out.println("Dòng 2");
// Kết quả:
// Dòng 1
// Dòng 2

// print — in ra KHÔNG xuống dòng
System.out.print("Xin ");
System.out.print("chào!");
// Kết quả: Xin chào!

// printf = "print formatted" — in có ĐỊNH DẠNG
String ten = "An";
int tuoi = 25;
double luong = 15000000.5;

System.out.printf("Tên: %s%n", ten);           // %s = chỗ chèn String
System.out.printf("Tuổi: %d tuổi%n", tuoi);    // %d = chỗ chèn số nguyên
System.out.printf("Lương: %,.2f VND%n", luong); // %,.2f = số thực có dấu phẩy ngăn và 2 số thập phân
// Kết quả:
// Tên: An
// Tuổi: 25 tuổi
// Lương: 15,000,000.50 VND
```

**Bảng format specifier (ký tự định dạng):**

| Ký tự | Dùng cho | Ví dụ | Kết quả |
|-------|---------|-------|---------|
| `%s` | String (chuỗi) | `printf("%s", "An")` | `An` |
| `%d` | int, long (số nguyên) | `printf("%d", 100)` | `100` |
| `%f` | float, double (số thực) | `printf("%f", 3.14)` | `3.140000` |
| `%.2f` | Số thực, 2 chữ số thập phân | `printf("%.2f", 3.14159)` | `3.14` |
| `%,.2f` | Số thực có dấu phẩy phân cách | `printf("%,.2f", 1000.5)` | `1,000.50` |
| `%n` | Xuống dòng mới | — | — |
| `%b` | boolean | `printf("%b", true)` | `true` |

### 6.2. Input (Nhập — đọc từ bàn phím)

Java dùng class `Scanner` để đọc input từ bàn phím:

```java
import java.util.Scanner;  // ① Phải import Scanner ở đầu file

public class InputDemo {
    public static void main(String[] args) {
        // ② Tạo đối tượng Scanner, nối với bàn phím (System.in)
        Scanner scanner = new Scanner(System.in);

        // ③ Đọc String (chuỗi — cả dòng)
        System.out.print("Nhập họ tên: ");
        String hoTen = scanner.nextLine();    // Đọc NGUYÊN DÒNG

        // ④ Đọc int (số nguyên)
        System.out.print("Nhập tuổi: ");
        int tuoi = scanner.nextInt();         // Đọc 1 số nguyên

        // ⑤ Đọc double (số thực)
        System.out.print("Nhập lương: ");
        double luong = scanner.nextDouble();  // Đọc 1 số thực

        // ⑥ Hiển thị kết quả
        System.out.printf("Chào %s, bạn %d tuổi, lương %.0f VND%n", hoTen, tuoi, luong);

        // ⑦ Đóng scanner (giải phóng tài nguyên)
        scanner.close();
    }
}
```

### 6.3. Bẫy kinh điển: nextInt() rồi nextLine()

Đây là lỗi **99% người mới** gặp phải:

```java
Scanner scanner = new Scanner(System.in);

System.out.print("Nhập tuổi: ");
int tuoi = scanner.nextInt();       // Bạn gõ: 25 rồi nhấn Enter
// ⚠️ nextInt() đọc số 25, NHƯNG BỎ LẠI ký tự Enter (\n) trong bộ đệm!

System.out.print("Nhập tên: ");
String ten = scanner.nextLine();    // ❌ BỊ SKIP! Vì nextLine() đọc luôn ký tự \n còn sót
// ten = "" (chuỗi rỗng, không phải tên bạn gõ!)

// ✅ CÁCH SỬA: Thêm 1 dòng nextLine() để "dọn rác"
System.out.print("Nhập tuổi: ");
int tuoi2 = scanner.nextInt();
scanner.nextLine();                 // 🧹 Đọc bỏ ký tự \n còn sót
System.out.print("Nhập tên: ");
String ten2 = scanner.nextLine();   // ✅ Giờ mới đọc đúng tên bạn gõ!
```

> 💡 **Mẹo nhớ**: Sau mỗi lần dùng `nextInt()`, `nextDouble()`, `nextFloat()`... mà tiếp theo cần `nextLine()`, LUÔN thêm `scanner.nextLine();` ở giữa để dọn ký tự Enter thừa.

---

## 7. Comments (Chú thích)

### Tại sao cần comment?

Comment là ghi chú cho **con người đọc**, Java hoàn toàn **bỏ qua** khi chạy. Giúp giải thích code cho đồng đội (hoặc chính bạn 3 tháng sau).

```java
// ① Comment 1 dòng — dùng 2 dấu gạch //
// Đây là comment, Java sẽ bỏ qua dòng này
int tuoi = 25; // Comment cũng có thể đặt cuối dòng code

// ② Comment nhiều dòng — dùng /* ... */
/*
 * Đây là comment nhiều dòng
 * Thường dùng để giải thích đoạn code phức tạp
 * hoặc tạm thời vô hiệu hóa (disable) một đoạn code
 */

// ③ Javadoc comment — dùng /** ... */
// Dùng để tạo tài liệu (documentation) tự động
/**
 * Tính tổng 2 số.
 * @param a số thứ nhất
 * @param b số thứ hai
 * @return tổng của a và b
 */
public static int tinhTong(int a, int b) {
    return a + b;
}
```

> 💡 **Phím tắt trong IntelliJ**: Chọn đoạn code → nhấn **Ctrl + /** để comment/uncomment nhanh.

---

## 8. Tóm tắt cuối ngày

| Khái niệm | Giải thích | Ví dụ |
|-----------|-----------|-------|
| JDK | Bộ công cụ để biên dịch và chạy Java | `javac`, `java` |
| IDE | Phần mềm viết code thông minh | IntelliJ, VS Code |
| Class | "Bản thiết kế" — mọi code Java nằm trong class | `public class Hello {}` |
| main method | "Cửa vào" chương trình — Java bắt đầu chạy từ đây | `public static void main(String[] args)` |
| Variable | "Hộp chứa" dữ liệu | `int age = 25;` |
| Primitive type | 8 kiểu cơ bản: byte, short, int, long, float, double, char, boolean | `int`, `double`, `boolean` |
| Reference type | Kiểu tham chiếu: String, Array, Object | `String name = "An";` |
| Type casting | Chuyển đổi kiểu dữ liệu | `(int) 3.14` → `3` |
| Scanner | Đọc input từ bàn phím | `scanner.nextLine()` |
| Comment | Ghi chú cho người đọc, Java bỏ qua | `// ghi chú` |

---

## 9. Bài tập thực hành

### Bài 1: Thông tin cá nhân
Viết chương trình nhập và hiển thị thông tin cá nhân + tính BMI:

**Yêu cầu:**
- Nhập: Họ tên, tuổi, chiều cao (m), cân nặng (kg)
- Tính: BMI = cân nặng / (chiều cao × chiều cao)
- Xuất: Hiển thị đẹp với printf

```java
// Gợi ý cấu trúc — BẠN TỰ VIẾT, đừng copy đáp án!
import java.util.Scanner;

public class PersonalInfo {
    public static void main(String[] args) {
        Scanner scanner = new Scanner(System.in);

        // 1. Nhập thông tin (dùng nextLine, nextInt, nextDouble)
        // Nhớ xử lý bẫy nextInt → nextLine!

        // 2. Tính BMI

        // 3. Hiển thị kết quả với printf

        scanner.close();
    }
}
```

**Kết quả mong muốn:**
```
Nhập họ tên: Nguyễn Văn A
Nhập tuổi: 25
Nhập chiều cao (m): 1.75
Nhập cân nặng (kg): 70

=== THÔNG TIN CÁ NHÂN ===
Họ tên: Nguyễn Văn A
Tuổi: 25
Chiều cao: 1.75 m
Cân nặng: 70.0 kg
BMI: 22.86
```

---

### Bài 2: Đổi tiền
Viết chương trình nhập số tiền VND và đổi sang USD, EUR, JPY.

**Gợi ý:**
```java
// Khai báo tỷ giá bằng hằng số (final)
final double TY_GIA_USD = 24500;   // 1 USD = 24,500 VND
final double TY_GIA_EUR = 26500;   // 1 EUR = 26,500 VND
final double TY_GIA_JPY = 165;     // 1 JPY = 165 VND
```

**Kết quả mong muốn:**
```
Nhập số tiền (VND): 1000000

=== ĐỔI TIỀN ===
VND: 1,000,000
USD: 40.82
EUR: 37.74
JPY: 6,060.61
```

---

### Bài 3: Swap hai số không dùng biến tạm
Hoán đổi giá trị 2 biến mà KHÔNG tạo biến thứ 3.

```java
int a = 5;
int b = 10;
System.out.println("Trước: a = " + a + ", b = " + b);

// Viết code swap ở đây (gợi ý: dùng phép cộng trừ hoặc XOR)

System.out.println("Sau: a = " + a + ", b = " + b);
// Kết quả: Trước: a = 5, b = 10 → Sau: a = 10, b = 5
```

---

### Bài 4: Chuyển đổi nhiệt độ
- Nhập nhiệt độ Celsius
- Đổi sang Fahrenheit: `F = C × 9/5 + 32`
- Đổi sang Kelvin: `K = C + 273.15`

---

### Bài 5: Tính chu vi và diện tích hình tròn
- Nhập bán kính
- Chu vi = `2 × π × r`
- Diện tích = `π × r²`
- Dùng `Math.PI` cho giá trị π, dùng `Math.pow(r, 2)` cho r²

---

## 10. Đáp án tham khảo

> ⚠️ **Hãy tự làm trước, chỉ xem đáp án khi đã thử ít nhất 15 phút!**

<details>
<summary>Bài 1: Thông tin cá nhân (click để mở)</summary>

```java
import java.util.Scanner;

public class PersonalInfo {
    public static void main(String[] args) {
        Scanner scanner = new Scanner(System.in);

        // Nhập họ tên
        System.out.print("Nhập họ tên: ");
        String hoTen = scanner.nextLine();

        // Nhập tuổi (nextInt đọc số, để lại \n)
        System.out.print("Nhập tuổi: ");
        int tuoi = scanner.nextInt();

        // Nhập chiều cao
        System.out.print("Nhập chiều cao (m): ");
        double chieuCao = scanner.nextDouble();

        // Nhập cân nặng
        System.out.print("Nhập cân nặng (kg): ");
        double canNang = scanner.nextDouble();

        // Tính BMI
        double bmi = canNang / (chieuCao * chieuCao);

        // Hiển thị kết quả
        System.out.println();
        System.out.println("=== THÔNG TIN CÁ NHÂN ===");
        System.out.printf("Họ tên: %s%n", hoTen);
        System.out.printf("Tuổi: %d%n", tuoi);
        System.out.printf("Chiều cao: %.2f m%n", chieuCao);
        System.out.printf("Cân nặng: %.1f kg%n", canNang);
        System.out.printf("BMI: %.2f%n", bmi);

        scanner.close();
    }
}
```
</details>

<details>
<summary>Bài 2: Đổi tiền (click để mở)</summary>

```java
import java.util.Scanner;

public class DoiTien {
    public static void main(String[] args) {
        // Khai báo tỷ giá bằng hằng số
        final double TY_GIA_USD = 24500;
        final double TY_GIA_EUR = 26500;
        final double TY_GIA_JPY = 165;

        Scanner scanner = new Scanner(System.in);

        // Nhập số tiền VND
        System.out.print("Nhập số tiền (VND): ");
        double vnd = scanner.nextDouble();

        // Tính đổi
        double usd = vnd / TY_GIA_USD;
        double eur = vnd / TY_GIA_EUR;
        double jpy = vnd / TY_GIA_JPY;

        // Hiển thị kết quả
        System.out.println();
        System.out.println("=== ĐỔI TIỀN ===");
        System.out.printf("VND: %,.0f%n", vnd);
        System.out.printf("USD: %.2f%n", usd);
        System.out.printf("EUR: %.2f%n", eur);
        System.out.printf("JPY: %,.2f%n", jpy);

        scanner.close();
    }
}
```
</details>

<details>
<summary>Bài 3: Swap hai số (click để mở)</summary>

```java
public class SwapNumbers {
    public static void main(String[] args) {
        int a = 5;
        int b = 10;

        System.out.println("Trước: a = " + a + ", b = " + b);

        // Cách 1: Dùng phép cộng trừ
        a = a + b;  // a = 15 (5 + 10)
        b = a - b;  // b = 5  (15 - 10)
        a = a - b;  // a = 10 (15 - 5)

        System.out.println("Sau: a = " + a + ", b = " + b);

        // Cách 2: Dùng XOR (phép toán bit — nâng cao hơn)
        // a = a ^ b;  // a = 5 XOR 10
        // b = a ^ b;  // b = (5 XOR 10) XOR 10 = 5
        // a = a ^ b;  // a = (5 XOR 10) XOR 5 = 10
    }
}
```
</details>

---

## Navigation

- [← Overview](./00-overview.md)
- [Day 2: Operators & Control Flow →](./day-02-operators-control-flow.md)
