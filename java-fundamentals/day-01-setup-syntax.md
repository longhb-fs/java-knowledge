# Day 1: Setup & Syntax Cơ Bản

## Mục tiêu
- Cài đặt JDK và IDE
- Viết chương trình Java đầu tiên
- Hiểu cấu trúc chương trình Java
- Variables và Data Types

---

## 1. Cài đặt môi trường

### 1.1. Cài đặt JDK 21

**Windows:**
1. Tải JDK 21 từ [Oracle](https://www.oracle.com/java/technologies/downloads/) hoặc [Adoptium](https://adoptium.net/)
2. Chạy installer
3. Thiết lập JAVA_HOME:
   - Mở System Properties → Environment Variables
   - Thêm `JAVA_HOME = C:\Program Files\Java\jdk-21`
   - Thêm `%JAVA_HOME%\bin` vào Path

**Kiểm tra cài đặt:**
```bash
java --version
# java 21.0.x 2024-xx-xx LTS

javac --version
# javac 21.0.x
```

### 1.2. Cài đặt IDE

**IntelliJ IDEA Community (Khuyến nghị):**
1. Tải từ [jetbrains.com/idea](https://www.jetbrains.com/idea/download/)
2. Chọn Community Edition (miễn phí)
3. Cài đặt và khởi động

**VS Code (Alternative):**
1. Cài Extension Pack for Java
2. Cài Language Support for Java

---

## 2. Hello World

### 2.1. Tạo project đầu tiên

1. IntelliJ → New Project → Java
2. Chọn JDK 21
3. Đặt tên: `JavaFundamentals`

### 2.2. Chương trình đầu tiên

```java
// File: HelloWorld.java
public class HelloWorld {
    public static void main(String[] args) {
        System.out.println("Hello, World!");
    }
}
```

**Giải thích:**
- `public class HelloWorld` - Tên class phải trùng với tên file
- `public static void main(String[] args)` - Entry point của chương trình
- `System.out.println()` - In ra console

### 2.3. Compile và Run

**Command line:**
```bash
# Compile
javac HelloWorld.java

# Run
java HelloWorld
```

**IDE:** Nhấn nút Run (Shift + F10 trong IntelliJ)

---

## 3. Cấu trúc chương trình Java

```java
// 1. Package declaration (optional)
package com.example.demo;

// 2. Import statements
import java.util.Scanner;

// 3. Class declaration
public class MyProgram {

    // 4. Fields (biến instance)
    private String name;

    // 5. Constructor
    public MyProgram() {
        this.name = "Default";
    }

    // 6. Methods
    public void sayHello() {
        System.out.println("Hello, " + name);
    }

    // 7. Main method - entry point
    public static void main(String[] args) {
        MyProgram program = new MyProgram();
        program.sayHello();
    }
}
```

---

## 4. Variables (Biến)

### 4.1. Khai báo biến

```java
// Cú pháp: dataType variableName = value;

int age = 25;
String name = "John";
double salary = 50000.50;
boolean isActive = true;
```

### 4.2. Quy tắc đặt tên

```java
// ✅ Hợp lệ
int myAge;
int _count;
int $price;
int camelCase;

// ❌ Không hợp lệ
int 2fast;      // Không bắt đầu bằng số
int my-age;     // Không dùng dấu gạch ngang
int my age;     // Không có khoảng trắng
int class;      // Không dùng từ khóa
```

### 4.3. Constants (Hằng số)

```java
// Dùng final để khai báo hằng số
final double PI = 3.14159;
final int MAX_SIZE = 100;

// Convention: UPPER_SNAKE_CASE
```

---

## 5. Data Types

### 5.1. Primitive Types (Kiểu nguyên thủy)

| Type | Size | Range | Default | Ví dụ |
|------|------|-------|---------|-------|
| `byte` | 1 byte | -128 to 127 | 0 | `byte b = 100;` |
| `short` | 2 bytes | -32,768 to 32,767 | 0 | `short s = 1000;` |
| `int` | 4 bytes | -2^31 to 2^31-1 | 0 | `int i = 100000;` |
| `long` | 8 bytes | -2^63 to 2^63-1 | 0L | `long l = 100000L;` |
| `float` | 4 bytes | ~6-7 decimal digits | 0.0f | `float f = 3.14f;` |
| `double` | 8 bytes | ~15 decimal digits | 0.0d | `double d = 3.14159;` |
| `char` | 2 bytes | 0 to 65,535 | '\u0000' | `char c = 'A';` |
| `boolean` | 1 bit | true/false | false | `boolean b = true;` |

### 5.2. Ví dụ Primitive Types

```java
public class DataTypesDemo {
    public static void main(String[] args) {
        // Integer types
        byte age = 25;
        short year = 2024;
        int population = 1000000;
        long worldPopulation = 8000000000L;  // Phải có L suffix

        // Floating-point types
        float price = 19.99f;  // Phải có f suffix
        double pi = 3.14159265359;

        // Character
        char grade = 'A';
        char unicode = '\u0041';  // 'A' in Unicode

        // Boolean
        boolean isJavaFun = true;
        boolean isFishTasty = false;

        // In ra
        System.out.println("Age: " + age);
        System.out.println("Price: " + price);
        System.out.println("Grade: " + grade);
        System.out.println("Is Java fun? " + isJavaFun);
    }
}
```

### 5.3. Reference Types (Kiểu tham chiếu)

```java
// String - chuỗi ký tự
String message = "Hello Java";
String empty = "";
String nullString = null;

// Arrays - mảng
int[] numbers = {1, 2, 3, 4, 5};
String[] names = new String[3];

// Objects - đối tượng
Scanner scanner = new Scanner(System.in);
```

### 5.4. Type Casting (Ép kiểu)

```java
// Widening (tự động) - nhỏ → lớn
int myInt = 100;
long myLong = myInt;      // int → long
double myDouble = myLong;  // long → double

// Narrowing (thủ công) - lớn → nhỏ
double d = 9.78;
int i = (int) d;  // i = 9 (mất phần thập phân)

long l = 1000L;
int j = (int) l;  // OK nếu giá trị trong range

// Cẩn thận với overflow
int big = 130;
byte b = (byte) big;  // b = -126 (overflow!)
```

---

## 6. Nhập xuất cơ bản

### 6.1. Output (Xuất)

```java
// println - in và xuống dòng
System.out.println("Hello");
System.out.println("World");
// Output:
// Hello
// World

// print - in không xuống dòng
System.out.print("Hello ");
System.out.print("World");
// Output: Hello World

// printf - formatted output
String name = "John";
int age = 25;
double salary = 50000.50;

System.out.printf("Name: %s%n", name);
System.out.printf("Age: %d years old%n", age);
System.out.printf("Salary: $%.2f%n", salary);
// Output:
// Name: John
// Age: 25 years old
// Salary: $50000.50
```

**Format specifiers:**
| Specifier | Mô tả |
|-----------|-------|
| `%s` | String |
| `%d` | Integer |
| `%f` | Float/Double |
| `%.2f` | Float với 2 decimal |
| `%n` | New line |
| `%b` | Boolean |

### 6.2. Input (Nhập)

```java
import java.util.Scanner;

public class InputDemo {
    public static void main(String[] args) {
        Scanner scanner = new Scanner(System.in);

        // Đọc String
        System.out.print("Enter your name: ");
        String name = scanner.nextLine();

        // Đọc int
        System.out.print("Enter your age: ");
        int age = scanner.nextInt();

        // Đọc double
        System.out.print("Enter your salary: ");
        double salary = scanner.nextDouble();

        // In kết quả
        System.out.printf("Hello %s, you are %d years old%n", name, age);
        System.out.printf("Your salary is $%.2f%n", salary);

        // Đóng scanner
        scanner.close();
    }
}
```

**Lưu ý quan trọng:**
```java
// Vấn đề khi đọc int rồi đọc String
System.out.print("Enter age: ");
int age = scanner.nextInt();

System.out.print("Enter name: ");
String name = scanner.nextLine();  // ⚠️ Bị skip!

// Giải pháp: thêm nextLine() sau nextInt()
int age = scanner.nextInt();
scanner.nextLine();  // Đọc bỏ ký tự xuống dòng
String name = scanner.nextLine();  // ✅ OK
```

---

## 7. Comments (Chú thích)

```java
// Single-line comment - chú thích 1 dòng

/*
 * Multi-line comment
 * Chú thích nhiều dòng
 */

/**
 * Javadoc comment
 * Dùng để tạo documentation
 * @param args command line arguments
 * @return nothing
 */
public static void main(String[] args) {
    // code here
}
```

---

## 8. Bài tập thực hành

### Bài 1: Thông tin cá nhân
Viết chương trình nhập và hiển thị thông tin cá nhân:
- Họ tên
- Tuổi
- Chiều cao (m)
- Cân nặng (kg)
- Tính BMI = cân nặng / (chiều cao ^ 2)

```java
// Gợi ý cấu trúc
public class PersonalInfo {
    public static void main(String[] args) {
        Scanner scanner = new Scanner(System.in);

        // Nhập thông tin
        // ...

        // Tính BMI
        // ...

        // Hiển thị kết quả với printf
        // ...
    }
}
```

**Expected output:**
```
Enter your name: John Doe
Enter your age: 25
Enter your height (m): 1.75
Enter your weight (kg): 70

=== Personal Information ===
Name: John Doe
Age: 25 years old
Height: 1.75 m
Weight: 70.0 kg
BMI: 22.86
```

---

### Bài 2: Đổi tiền
Viết chương trình nhập số tiền (VND) và đổi sang USD, EUR, JPY.
- Tỷ giá: 1 USD = 24,500 VND, 1 EUR = 26,500 VND, 1 JPY = 165 VND

```java
// Gợi ý
final double USD_RATE = 24500;
final double EUR_RATE = 26500;
final double JPY_RATE = 165;
```

**Expected output:**
```
Enter amount in VND: 1000000

=== Currency Exchange ===
VND: 1,000,000
USD: 40.82
EUR: 37.74
JPY: 6,060.61
```

---

### Bài 3: Swap hai số
Viết chương trình hoán đổi giá trị 2 biến **không dùng biến tạm**.

```java
int a = 5;
int b = 10;

// Swap không dùng biến tạm
// Gợi ý: dùng phép cộng trừ hoặc XOR

System.out.println("Before: a = " + a + ", b = " + b);
// Your code here
System.out.println("After: a = " + a + ", b = " + b);
```

**Expected output:**
```
Before: a = 5, b = 10
After: a = 10, b = 5
```

---

### Bài 4: Temperature Converter
Viết chương trình chuyển đổi nhiệt độ:
- Nhập nhiệt độ Celsius
- Đổi sang Fahrenheit: F = C × 9/5 + 32
- Đổi sang Kelvin: K = C + 273.15

---

### Bài 5: Circle Calculator
Viết chương trình tính chu vi và diện tích hình tròn:
- Nhập bán kính
- Chu vi = 2 × π × r
- Diện tích = π × r²
- Sử dụng `Math.PI` cho giá trị π

---

## 9. Đáp án tham khảo

<details>
<summary>Bài 1: Thông tin cá nhân</summary>

```java
import java.util.Scanner;

public class PersonalInfo {
    public static void main(String[] args) {
        Scanner scanner = new Scanner(System.in);

        System.out.print("Enter your name: ");
        String name = scanner.nextLine();

        System.out.print("Enter your age: ");
        int age = scanner.nextInt();

        System.out.print("Enter your height (m): ");
        double height = scanner.nextDouble();

        System.out.print("Enter your weight (kg): ");
        double weight = scanner.nextDouble();

        // Tính BMI
        double bmi = weight / (height * height);

        // Hiển thị kết quả
        System.out.println();
        System.out.println("=== Personal Information ===");
        System.out.printf("Name: %s%n", name);
        System.out.printf("Age: %d years old%n", age);
        System.out.printf("Height: %.2f m%n", height);
        System.out.printf("Weight: %.1f kg%n", weight);
        System.out.printf("BMI: %.2f%n", bmi);

        scanner.close();
    }
}
```
</details>

<details>
<summary>Bài 2: Đổi tiền</summary>

```java
import java.util.Scanner;

public class CurrencyExchange {
    public static void main(String[] args) {
        final double USD_RATE = 24500;
        final double EUR_RATE = 26500;
        final double JPY_RATE = 165;

        Scanner scanner = new Scanner(System.in);

        System.out.print("Enter amount in VND: ");
        double vnd = scanner.nextDouble();

        double usd = vnd / USD_RATE;
        double eur = vnd / EUR_RATE;
        double jpy = vnd / JPY_RATE;

        System.out.println();
        System.out.println("=== Currency Exchange ===");
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
<summary>Bài 3: Swap hai số</summary>

```java
public class SwapNumbers {
    public static void main(String[] args) {
        int a = 5;
        int b = 10;

        System.out.println("Before: a = " + a + ", b = " + b);

        // Cách 1: Dùng phép cộng trừ
        a = a + b;  // a = 15
        b = a - b;  // b = 5
        a = a - b;  // a = 10

        // Cách 2: Dùng XOR
        // a = a ^ b;
        // b = a ^ b;
        // a = a ^ b;

        System.out.println("After: a = " + a + ", b = " + b);
    }
}
```
</details>

---

## Navigation

- [← Overview](./00-overview.md)
- [Day 2: Operators & Control Flow →](./day-02-operators-control-flow.md)
