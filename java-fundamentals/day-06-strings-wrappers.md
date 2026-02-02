# Day 6: Strings & Wrapper Classes

## Mục tiêu
- String class và methods
- StringBuilder và StringBuffer
- String formatting
- Wrapper classes (Integer, Double, ...)
- Autoboxing và Unboxing

---

## 1. String Class

### 1.1. String là immutable

```java
String str = "Hello";
str = str + " World";  // Tạo String MỚI, không modify cũ

// String pool
String s1 = "Hello";        // Từ String pool
String s2 = "Hello";        // Cùng reference từ pool
String s3 = new String("Hello");  // Object mới trên heap

System.out.println(s1 == s2);      // true (cùng reference)
System.out.println(s1 == s3);      // false (khác reference)
System.out.println(s1.equals(s3)); // true (cùng nội dung)
```

### 1.2. Tạo String

```java
// Literal
String s1 = "Hello";

// Constructor
String s2 = new String("Hello");

// Từ char array
char[] chars = {'H', 'e', 'l', 'l', 'o'};
String s3 = new String(chars);

// Từ byte array
byte[] bytes = {72, 101, 108, 108, 111};
String s4 = new String(bytes);

// Từ StringBuilder
StringBuilder sb = new StringBuilder("Hello");
String s5 = sb.toString();
```

---

## 2. String Methods

### 2.1. Độ dài và truy cập

```java
String str = "Hello World";

// Độ dài
int len = str.length();  // 11

// Ký tự tại vị trí
char c = str.charAt(0);  // 'H'
char c2 = str.charAt(6); // 'W'

// isEmpty() và isBlank() (Java 11+)
"".isEmpty();    // true
"".isBlank();    // true
"  ".isEmpty();  // false
"  ".isBlank();  // true (chỉ whitespace)
```

### 2.2. So sánh

```java
String s1 = "Hello";
String s2 = "hello";
String s3 = "Hello";

// equals - case sensitive
s1.equals(s2);        // false
s1.equals(s3);        // true

// equalsIgnoreCase - case insensitive
s1.equalsIgnoreCase(s2);  // true

// compareTo - so sánh thứ tự từ điển
s1.compareTo(s3);     // 0 (equal)
s1.compareTo(s2);     // -32 (H < h)
"apple".compareTo("banana");  // -1 (a < b)

// compareToIgnoreCase
s1.compareToIgnoreCase(s2);  // 0

// contentEquals - so sánh với CharSequence
s1.contentEquals(new StringBuilder("Hello"));  // true
```

### 2.3. Tìm kiếm

```java
String str = "Hello World, Hello Java";

// indexOf - vị trí xuất hiện đầu tiên
str.indexOf('o');        // 4
str.indexOf("Hello");    // 0
str.indexOf("Hello", 1); // 13 (bắt đầu từ index 1)

// lastIndexOf - vị trí xuất hiện cuối cùng
str.lastIndexOf('o');    // 20
str.lastIndexOf("Hello"); // 13

// contains
str.contains("World");   // true
str.contains("world");   // false

// startsWith, endsWith
str.startsWith("Hello"); // true
str.endsWith("Java");    // true
str.startsWith("World", 6); // true (bắt đầu từ index 6)
```

### 2.4. Substring

```java
String str = "Hello World";

// substring(beginIndex)
str.substring(6);      // "World"

// substring(beginIndex, endIndex) - endIndex exclusive
str.substring(0, 5);   // "Hello"
str.substring(6, 11);  // "World"

// subSequence (trả về CharSequence)
CharSequence cs = str.subSequence(0, 5);  // "Hello"
```

### 2.5. Chuyển đổi

```java
String str = "  Hello World  ";

// toUpperCase, toLowerCase
str.toUpperCase();     // "  HELLO WORLD  "
str.toLowerCase();     // "  hello world  "

// trim - xóa whitespace đầu cuối
str.trim();            // "Hello World"

// strip (Java 11+) - xóa Unicode whitespace
str.strip();           // "Hello World"
str.stripLeading();    // "Hello World  "
str.stripTrailing();   // "  Hello World"

// replace
"Hello".replace('l', 'x');     // "Hexxo"
"Hello".replace("ll", "LL");   // "HeLLo"

// replaceAll - regex
"a1b2c3".replaceAll("\\d", "X");  // "aXbXcX"

// replaceFirst - regex, chỉ thay thế đầu tiên
"a1b2c3".replaceFirst("\\d", "X");  // "aXb2c3"
```

### 2.6. Split và Join

```java
// split
String csv = "apple,banana,cherry";
String[] fruits = csv.split(",");
// ["apple", "banana", "cherry"]

String text = "Hello   World";
String[] words = text.split("\\s+");  // Regex: 1 hoặc nhiều whitespace
// ["Hello", "World"]

// split với limit
"a,b,c,d".split(",", 2);  // ["a", "b,c,d"]

// join (Java 8+)
String joined = String.join(", ", "apple", "banana", "cherry");
// "apple, banana, cherry"

String[] arr = {"a", "b", "c"};
String joined2 = String.join("-", arr);  // "a-b-c"
```

### 2.7. Kiểm tra và chuyển đổi

```java
// toCharArray
char[] chars = "Hello".toCharArray();
// ['H', 'e', 'l', 'l', 'o']

// getBytes
byte[] bytes = "Hello".getBytes();
byte[] utf8 = "Hello".getBytes(StandardCharsets.UTF_8);

// valueOf - chuyển thành String
String s1 = String.valueOf(123);      // "123"
String s2 = String.valueOf(3.14);     // "3.14"
String s3 = String.valueOf(true);     // "true"
String s4 = String.valueOf(new char[]{'H', 'i'});  // "Hi"

// matches - regex
"hello123".matches("[a-z]+\\d+");  // true
```

### 2.8. Repeat và Indent (Java 11+)

```java
// repeat
"Ha".repeat(3);  // "HaHaHa"

// indent (Java 12+)
String text = "Hello\nWorld";
text.indent(4);
// "    Hello\n    World\n"
```

---

## 3. StringBuilder và StringBuffer

### 3.1. Tại sao cần StringBuilder?

```java
// ❌ Không hiệu quả - tạo nhiều String objects
String result = "";
for (int i = 0; i < 1000; i++) {
    result += i;  // Tạo String mới mỗi lần
}

// ✅ Hiệu quả - dùng StringBuilder
StringBuilder sb = new StringBuilder();
for (int i = 0; i < 1000; i++) {
    sb.append(i);
}
String result = sb.toString();
```

### 3.2. StringBuilder Methods

```java
StringBuilder sb = new StringBuilder("Hello");

// append - thêm vào cuối
sb.append(" ");
sb.append("World");
sb.append(123);
// "Hello World123"

// Chaining
sb.append(" ").append("Java").append("!");
// "Hello World123 Java!"

// insert - chèn vào vị trí
sb.insert(0, ">>> ");
// ">>> Hello World123 Java!"

// delete - xóa
sb.delete(0, 4);      // Xóa từ index 0 đến 3
sb.deleteCharAt(0);   // Xóa ký tự tại index 0

// replace
sb.replace(0, 5, "Hi");

// reverse
new StringBuilder("Hello").reverse();  // "olleH"

// setCharAt
sb.setCharAt(0, 'h');

// length và capacity
sb.length();    // Số ký tự hiện tại
sb.capacity();  // Dung lượng buffer

// setLength
sb.setLength(5);  // Cắt còn 5 ký tự

// toString
String result = sb.toString();
```

### 3.3. StringBuilder vs StringBuffer

| Feature | StringBuilder | StringBuffer |
|---------|--------------|--------------|
| Thread-safe | ❌ No | ✅ Yes (synchronized) |
| Performance | ✅ Faster | ❌ Slower |
| When to use | Single-threaded | Multi-threaded |

```java
// StringBuilder - không thread-safe, nhanh hơn
StringBuilder sb = new StringBuilder();

// StringBuffer - thread-safe, chậm hơn
StringBuffer sbf = new StringBuffer();
```

---

## 4. String Formatting

### 4.1. printf và format

```java
String name = "John";
int age = 25;
double salary = 50000.5;

// printf
System.out.printf("Name: %s, Age: %d%n", name, age);
System.out.printf("Salary: $%.2f%n", salary);

// String.format
String formatted = String.format("Name: %s, Age: %d", name, age);

// formatted() method (Java 15+)
String result = "Name: %s, Age: %d".formatted(name, age);
```

### 4.2. Format Specifiers

| Specifier | Type | Example |
|-----------|------|---------|
| `%s` | String | `"Hello"` |
| `%d` | Integer | `123` |
| `%f` | Float/Double | `3.14` |
| `%.2f` | 2 decimal places | `3.14` |
| `%e` | Scientific notation | `1.23e+02` |
| `%n` | New line | |
| `%b` | Boolean | `true` |
| `%c` | Character | `'A'` |
| `%x` | Hexadecimal | `ff` |
| `%o` | Octal | `77` |

### 4.3. Format với width và alignment

```java
// Width
System.out.printf("|%10s|%n", "Hi");      // |        Hi|
System.out.printf("|%-10s|%n", "Hi");     // |Hi        | (left align)

// Width với numbers
System.out.printf("|%8d|%n", 123);        // |     123|
System.out.printf("|%-8d|%n", 123);       // |123     |
System.out.printf("|%08d|%n", 123);       // |00000123| (zero-padding)

// Precision với float
System.out.printf("|%10.2f|%n", 3.14159); // |      3.14|
System.out.printf("|%-10.2f|%n", 3.14159);// |3.14      |
```

### 4.4. Text Blocks (Java 15+)

```java
String html = """
    <html>
        <body>
            <h1>Hello World</h1>
        </body>
    </html>
    """;

String json = """
    {
        "name": "John",
        "age": 25
    }
    """;

// Với format
String template = """
    Hello %s,
    Your balance is $%.2f
    """.formatted("John", 1000.50);
```

---

## 5. Wrapper Classes

### 5.1. Primitive vs Wrapper

| Primitive | Wrapper | Size |
|-----------|---------|------|
| `byte` | `Byte` | 1 byte |
| `short` | `Short` | 2 bytes |
| `int` | `Integer` | 4 bytes |
| `long` | `Long` | 8 bytes |
| `float` | `Float` | 4 bytes |
| `double` | `Double` | 8 bytes |
| `char` | `Character` | 2 bytes |
| `boolean` | `Boolean` | 1 bit |

### 5.2. Tạo Wrapper Objects

```java
// Boxing - primitive to wrapper
Integer i1 = Integer.valueOf(10);  // Recommended
Integer i2 = 10;  // Autoboxing

// Unboxing - wrapper to primitive
int num = i1.intValue();
int num2 = i1;  // Auto-unboxing

// Parse từ String
int a = Integer.parseInt("123");
double b = Double.parseDouble("3.14");
boolean c = Boolean.parseBoolean("true");

// valueOf - trả về wrapper object
Integer d = Integer.valueOf("123");
Double e = Double.valueOf("3.14");
```

### 5.3. Autoboxing và Unboxing

```java
// Autoboxing - tự động chuyển primitive → wrapper
Integer num = 100;  // int → Integer
Double d = 3.14;    // double → Double

// Auto-unboxing - tự động chuyển wrapper → primitive
int n = num;        // Integer → int
double dd = d;      // Double → double

// Trong expressions
Integer a = 10;
Integer b = 20;
Integer sum = a + b;  // Unbox, add, autobox

// Cẩn thận với null
Integer x = null;
// int y = x;  // NullPointerException!

// Safe unboxing
int y = (x != null) ? x : 0;
```

### 5.4. Wrapper Methods

```java
// Integer methods
Integer.MAX_VALUE;     // 2147483647
Integer.MIN_VALUE;     // -2147483648
Integer.SIZE;          // 32 (bits)
Integer.BYTES;         // 4

Integer.parseInt("123");
Integer.parseInt("FF", 16);  // 255 (hexadecimal)
Integer.toBinaryString(10);  // "1010"
Integer.toHexString(255);    // "ff"
Integer.toOctalString(8);    // "10"

Integer.compare(10, 20);     // -1 (10 < 20)
Integer.max(10, 20);         // 20
Integer.min(10, 20);         // 10
Integer.sum(10, 20);         // 30

// Double methods
Double.MAX_VALUE;
Double.MIN_VALUE;
Double.POSITIVE_INFINITY;
Double.NEGATIVE_INFINITY;
Double.NaN;

Double.isNaN(0.0 / 0.0);       // true
Double.isInfinite(1.0 / 0.0);  // true
Double.isFinite(3.14);         // true

// Character methods
Character.isLetter('A');       // true
Character.isDigit('5');        // true
Character.isLetterOrDigit('A'); // true
Character.isWhitespace(' ');   // true
Character.isUpperCase('A');    // true
Character.isLowerCase('a');    // true
Character.toUpperCase('a');    // 'A'
Character.toLowerCase('A');    // 'a'
```

### 5.5. Integer Cache

```java
Integer a = 127;
Integer b = 127;
System.out.println(a == b);  // true (cached)

Integer c = 128;
Integer d = 128;
System.out.println(c == d);  // false (not cached)
System.out.println(c.equals(d));  // true

// Integer cache range: -128 to 127
```

---

## 6. Bài tập thực hành

### Bài 1: String Utilities
Tạo class `StringUtils` với các methods:
- `reverse(String)` - đảo ngược chuỗi
- `isPalindrome(String)` - kiểm tra palindrome
- `countWords(String)` - đếm số từ
- `countVowels(String)` - đếm nguyên âm
- `capitalize(String)` - viết hoa chữ cái đầu mỗi từ

```java
StringUtils.reverse("hello");        // "olleh"
StringUtils.isPalindrome("radar");   // true
StringUtils.countWords("Hello World");  // 2
StringUtils.countVowels("hello");    // 2
StringUtils.capitalize("hello world");  // "Hello World"
```

---

### Bài 2: Password Validator
Tạo password validator kiểm tra:
- Độ dài tối thiểu 8 ký tự
- Có ít nhất 1 chữ hoa
- Có ít nhất 1 chữ thường
- Có ít nhất 1 số
- Có ít nhất 1 ký tự đặc biệt

---

### Bài 3: String Compression
Tạo function nén chuỗi: `"aaabbbcc" → "a3b3c2"`
Nếu chuỗi nén dài hơn gốc, trả về chuỗi gốc.

---

### Bài 4: Number Formatter
Tạo class format số:
- Format tiền tệ: `1234567.89 → $1,234,567.89`
- Format phần trăm: `0.1234 → 12.34%`
- Format số điện thoại: `1234567890 → (123) 456-7890`

---

### Bài 5: StringBuilder Performance Test
So sánh performance của String concatenation vs StringBuilder với 100,000 iterations.

---

## 7. Đáp án tham khảo

<details>
<summary>Bài 1: String Utilities</summary>

```java
public class StringUtils {

    public static String reverse(String str) {
        if (str == null) return null;
        return new StringBuilder(str).reverse().toString();
    }

    public static boolean isPalindrome(String str) {
        if (str == null) return false;
        String cleaned = str.toLowerCase().replaceAll("[^a-z0-9]", "");
        return cleaned.equals(reverse(cleaned));
    }

    public static int countWords(String str) {
        if (str == null || str.isBlank()) return 0;
        return str.trim().split("\\s+").length;
    }

    public static int countVowels(String str) {
        if (str == null) return 0;
        int count = 0;
        String vowels = "aeiouAEIOU";
        for (char c : str.toCharArray()) {
            if (vowels.indexOf(c) != -1) {
                count++;
            }
        }
        return count;
    }

    public static String capitalize(String str) {
        if (str == null || str.isEmpty()) return str;

        StringBuilder result = new StringBuilder();
        boolean capitalizeNext = true;

        for (char c : str.toCharArray()) {
            if (Character.isWhitespace(c)) {
                capitalizeNext = true;
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
        System.out.println(reverse("hello"));           // olleh
        System.out.println(isPalindrome("A man a plan a canal Panama")); // true
        System.out.println(countWords("Hello World"));  // 2
        System.out.println(countVowels("hello"));       // 2
        System.out.println(capitalize("hello world"));  // Hello World
    }
}
```
</details>

<details>
<summary>Bài 3: String Compression</summary>

```java
public class StringCompressor {

    public static String compress(String str) {
        if (str == null || str.isEmpty()) return str;

        StringBuilder compressed = new StringBuilder();
        int count = 1;

        for (int i = 0; i < str.length(); i++) {
            if (i + 1 < str.length() && str.charAt(i) == str.charAt(i + 1)) {
                count++;
            } else {
                compressed.append(str.charAt(i));
                if (count > 1) {
                    compressed.append(count);
                }
                count = 1;
            }
        }

        return compressed.length() < str.length() ?
            compressed.toString() : str;
    }

    public static void main(String[] args) {
        System.out.println(compress("aaabbbcc"));   // a3b3c2
        System.out.println(compress("aabcccccaaa")); // a2bc5a3
        System.out.println(compress("abc"));         // abc (không nén)
    }
}
```
</details>

<details>
<summary>Bài 5: Performance Test</summary>

```java
public class PerformanceTest {

    public static void main(String[] args) {
        int iterations = 100000;

        // String concatenation
        long start = System.currentTimeMillis();
        String s = "";
        for (int i = 0; i < iterations; i++) {
            s += "a";
        }
        long stringTime = System.currentTimeMillis() - start;

        // StringBuilder
        start = System.currentTimeMillis();
        StringBuilder sb = new StringBuilder();
        for (int i = 0; i < iterations; i++) {
            sb.append("a");
        }
        String result = sb.toString();
        long sbTime = System.currentTimeMillis() - start;

        System.out.println("String concatenation: " + stringTime + "ms");
        System.out.println("StringBuilder: " + sbTime + "ms");
        System.out.println("StringBuilder is " + (stringTime / sbTime) + "x faster");
    }
}
// Output example:
// String concatenation: 8500ms
// StringBuilder: 5ms
// StringBuilder is 1700x faster
```
</details>

---

## Navigation

- [← Day 5: Exception Handling](./day-05-exception-handling.md)
- [Day 7: Collections Basics →](./day-07-collections-basics.md)
