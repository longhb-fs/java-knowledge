# Day 9: Lambda & Functional Programming (Lambda & Lập Trình Hàm)

## Mục tiêu hôm nay

Sau khi học xong Day 9, bạn sẽ:
- Hiểu **Lambda Expression** (biểu thức lambda) là gì và cách viết
- Hiểu **Functional Interface** (interface hàm) — nền tảng của lambda
- Sử dụng các **Built-in Functional Interfaces**: Function, Consumer, Supplier, Predicate
- Biết cách dùng **Method References** (tham chiếu method) — ngắn gọn hơn lambda
- Áp dụng lambda trong **thực tế**: sắp xếp, lọc, xử lý sự kiện

---

## Tại sao cần học Lambda?

### Trước Java 8: Code DÀI DÒNG

```java
// Muốn sắp xếp danh sách tên → phải viết cả class ẩn danh (anonymous class)!
List<String> names = Arrays.asList("Châu", "An", "Bình");

Collections.sort(names, new Comparator<String>() {
    @Override
    public int compare(String s1, String s2) {
        return s1.compareTo(s2);
    }
});
// 5 dòng code chỉ để nói: "sắp xếp theo bảng chữ cái"!
```

### Từ Java 8: Lambda NGẮN GỌN

```java
// Cùng kết quả nhưng CHỈ 1 DÒNG!
Collections.sort(names, (s1, s2) -> s1.compareTo(s2));

// Hoặc ngắn hơn nữa: method reference
Collections.sort(names, String::compareTo);

// Hoặc đơn giản nhất:
names.sort(String::compareTo);
```

### Ví dụ đời thường

```
TRƯỚC Lambda (anonymous class):
  Giống viết một BỨC THƯ đầy đủ:
  ┌───────────────────────────────────┐
  │ Kính gửi: Bưu điện               │
  │ Tôi tên là: Comparator<String>   │
  │ Tôi muốn thực hiện:              │
  │   Phương thức compare:            │
  │     So sánh s1 với s2             │
  │ Trân trọng!                       │
  └───────────────────────────────────┘

SAU Lambda:
  Giống gửi TIN NHẮN ngắn:
  ┌───────────────────────────┐
  │ (s1, s2) -> s1.compareTo(s2) │
  └───────────────────────────┘
  → Cùng nội dung, nhưng BỎ HẾT phần thừa!
```

---

## 1. Lambda Expressions (Biểu thức Lambda)

### 1.1. Cú pháp Lambda

```java
// CÚ PHÁP CHUNG:
// (tham số) -> biểu thức
// (tham số) -> { các câu lệnh; }

// ===== Không có tham số =====
() -> System.out.println("Xin chào!")
//↑     ↑
//│     └── Thân: in ra "Xin chào!"
//└── Không có tham số

// ===== 1 tham số (không cần ngoặc tròn) =====
x -> x * 2
//↑    ↑
//│    └── Trả về x nhân 2
//└── Tham số x (Java tự suy luận kiểu)

// ===== 2+ tham số (BẮT BUỘC ngoặc tròn) =====
(x, y) -> x + y
// ↑         ↑
// │         └── Trả về x + y
// └── 2 tham số x và y

// ===== Có khai báo kiểu (tùy chọn) =====
(String s) -> s.length()
//  ↑             ↑
//  │             └── Trả về độ dài chuỗi
//  └── Tham số s kiểu String

// ===== Nhiều câu lệnh (PHẢI có {} và return) =====
(x, y) -> {
    int sum = x + y;
    System.out.println("Tổng: " + sum);
    return sum;  // Phải có return nếu có giá trị trả về
}
```

### 1.2. So sánh Lambda vs Anonymous Class

```java
// ===== TRƯỚC: Anonymous class (lớp ẩn danh) =====
// Viết 1 Comparator để sắp xếp
Comparator<String> comp1 = new Comparator<String>() {
    @Override
    public int compare(String s1, String s2) {
        return s1.compareTo(s2);  // So sánh bảng chữ cái
    }
};
// → 5 dòng code!

// ===== SAU: Lambda expression =====
Comparator<String> comp2 = (s1, s2) -> s1.compareTo(s2);
// → 1 dòng code! Cùng kết quả!

// ===== VÍ DỤ 2: Runnable (chạy tác vụ) =====

// Anonymous class:
Runnable r1 = new Runnable() {
    @Override
    public void run() {
        System.out.println("Đang chạy tác vụ...");
    }
};

// Lambda:
Runnable r2 = () -> System.out.println("Đang chạy tác vụ...");

// ===== VÍ DỤ 3: ActionListener (xử lý sự kiện) =====

// Anonymous class:
button.addActionListener(new ActionListener() {
    @Override
    public void actionPerformed(ActionEvent e) {
        System.out.println("Nút được click!");
    }
});

// Lambda:
button.addActionListener(e -> System.out.println("Nút được click!"));
```

### 1.3. Effectively Final — Biến ngoài trong Lambda

Lambda có thể truy cập biến bên ngoài, nhưng biến đó phải **không thay đổi** (effectively final):

```java
int multiplier = 5;  // Effectively final (gán 1 lần, không đổi)

// Lambda truy cập biến multiplier bên ngoài
Function<Integer, Integer> multiply = x -> x * multiplier;
System.out.println(multiply.apply(3));  // 15

// ❌ SAI: Không thể thay đổi biến mà lambda đang dùng
// multiplier = 10;  // COMPILE ERROR!
// Vì lambda "bắt" (capture) giá trị biến → biến không được thay đổi

// ✅ Workaround: Dùng mảng hoặc AtomicInteger
int[] counter = {0};  // Mảng có thể thay đổi nội dung
Runnable increment = () -> counter[0]++;  // Sửa phần tử, không sửa biến mảng
increment.run();
System.out.println(counter[0]);  // 1
```

💡 **Mẹo nhớ:** Lambda chỉ "chụp ảnh" giá trị biến ngoài. Nếu biến thay đổi → ảnh bị sai → Java không cho phép.

---

## 2. Functional Interface (Interface Hàm)

### 2.1. Định nghĩa

**Functional Interface** = Interface có **DUY NHẤT 1 abstract method** (phương thức trừu tượng).

Lambda chỉ hoạt động với Functional Interface!

```java
// @FunctionalInterface = annotation đánh dấu "đây là Functional Interface"
// → Compiler sẽ kiểm tra: nếu có > 1 abstract method → báo lỗi
@FunctionalInterface
public interface Calculator {
    // DUY NHẤT 1 abstract method
    int calculate(int a, int b);

    // ✅ Có thể có default methods (method có thân)
    default void printInfo() {
        System.out.println("Tôi là Calculator");
    }

    // ✅ Có thể có static methods
    static Calculator createAdder() {
        return (a, b) -> a + b;
    }
}
```

**Sử dụng:**

```java
// Lambda được gán cho Functional Interface
Calculator add      = (a, b) -> a + b;      // Phép cộng
Calculator subtract = (a, b) -> a - b;      // Phép trừ
Calculator multiply = (a, b) -> a * b;      // Phép nhân
Calculator divide   = (a, b) -> a / b;      // Phép chia
Calculator max      = (a, b) -> Math.max(a, b); // Lấy max

System.out.println(add.calculate(10, 3));       // 13
System.out.println(subtract.calculate(10, 3));  // 7
System.out.println(multiply.calculate(10, 3));  // 30
```

### 2.2. Các Functional Interface có sẵn trong Java (Built-in)

Java cung cấp sẵn nhiều Functional Interface phổ biến trong package `java.util.function`:

```
┌─────────────────────────────────────────────────────────┐
│  TÊN              │  NHẬN VÀO  │  TRẢ VỀ   │ VÍ DỤ   │
├────────────────────┼────────────┼───────────┼─────────┤
│  Function<T, R>    │  T         │  R        │ T → R   │
│  Consumer<T>       │  T         │  void     │ T → ❌  │
│  Supplier<T>       │  (không)   │  T        │ ❌ → T  │
│  Predicate<T>      │  T         │  boolean  │ T → ✅❌│
│  BiFunction<T,U,R> │  T, U      │  R        │ T,U → R │
│  UnaryOperator<T>  │  T         │  T        │ T → T   │
│  BinaryOperator<T> │  T, T      │  T        │ T,T → T │
└────────────────────┴────────────┴───────────┴─────────┘
```

💡 **Mẹo nhớ từng tên:**
- **Function** = "Hàm" — nhận 1 thứ → trả ra thứ khác
- **Consumer** = "Tiêu thụ" — nhận 1 thứ → không trả gì (chỉ xử lý)
- **Supplier** = "Cung cấp" — không nhận gì → trả ra 1 thứ
- **Predicate** = "Kiểm tra" — nhận 1 thứ → trả true/false

---

## 3. Function\<T, R\> (Hàm: nhận T → trả R)

### 3.1. Cơ bản

```java
import java.util.function.Function;

// Function<String, Integer> = nhận String → trả Integer
Function<String, Integer> getLength = s -> s.length();
System.out.println(getLength.apply("Hello"));  // 5
//                            ↑
//                     Gọi method apply() để thực thi

// Function<Integer, String> = nhận Integer → trả String
Function<Integer, String> numberToText = n -> {
    if (n == 1) return "Một";
    if (n == 2) return "Hai";
    if (n == 3) return "Ba";
    return "Số " + n;
};
System.out.println(numberToText.apply(2));  // "Hai"
System.out.println(numberToText.apply(5));  // "Số 5"
```

### 3.2. Chuỗi Function (Chaining)

Nối nhiều Function lại với nhau:

```java
Function<String, String> toUpper = s -> s.toUpperCase();
Function<String, String> addBrackets = s -> "[" + s + "]";
Function<String, Integer> getLength = s -> s.length();

// andThen: Thực hiện THEO THỨ TỰ
// toUpper → rồi addBrackets
Function<String, String> format = toUpper.andThen(addBrackets);
System.out.println(format.apply("hello"));
// Bước 1: toUpper("hello") → "HELLO"
// Bước 2: addBrackets("HELLO") → "[HELLO]"
// Kết quả: "[HELLO]"

// compose: Thực hiện NGƯỢC THỨ TỰ
// addBrackets trước → rồi toUpper
Function<String, String> format2 = toUpper.compose(addBrackets);
System.out.println(format2.apply("hello"));
// Bước 1: addBrackets("hello") → "[hello]"
// Bước 2: toUpper("[hello]") → "[HELLO]"
// Kết quả: "[HELLO]"

// Nối dài:
Function<String, Integer> pipeline = toUpper
    .andThen(addBrackets)
    .andThen(getLength);
System.out.println(pipeline.apply("hello"));
// "hello" → "HELLO" → "[HELLO]" → 7

// identity: Function "không làm gì" (trả về nguyên input)
Function<String, String> doNothing = Function.identity();
System.out.println(doNothing.apply("test"));  // "test"
```

💡 **Mẹo nhớ:** `andThen` = "rồi làm tiếp", `compose` = "nhưng trước đó hãy".

---

## 4. Consumer\<T\> (Tiêu thụ: nhận T → không trả gì)

Consumer **chỉ nhận vào, không trả về**. Dùng khi bạn muốn **xử lý** dữ liệu (in, log, gửi email...) mà không cần kết quả.

```java
import java.util.function.Consumer;

// Consumer<String> = nhận String → không trả gì (void)
Consumer<String> print = s -> System.out.println(s);
print.accept("Xin chào!");  // In: Xin chào!
//        ↑
//  Gọi accept() để thực thi Consumer

// Xử lý object
Consumer<User> sendWelcomeEmail = user ->
    System.out.println("Gửi email chào mừng tới: " + user.getEmail());

Consumer<User> logUserCreated = user ->
    System.out.println("LOG: Đã tạo user " + user.getName());

// ===== Chuỗi Consumer: andThen =====
Consumer<User> onUserCreated = logUserCreated.andThen(sendWelcomeEmail);
// Khi user được tạo → log trước → rồi gửi email

onUserCreated.accept(new User("An", "an@email.com"));
// Output:
// LOG: Đã tạo user An
// Gửi email chào mừng tới: an@email.com

// ===== Dùng trong forEach =====
List<String> names = List.of("An", "Bình", "Châu");

// Lambda
names.forEach(name -> System.out.println("Xin chào " + name));

// Method reference (ngắn hơn)
names.forEach(System.out::println);
```

---

## 5. Supplier\<T\> (Cung cấp: không nhận → trả T)

Supplier **không nhận gì, chỉ trả về**. Dùng khi bạn muốn **tạo** hoặc **cung cấp** dữ liệu.

```java
import java.util.function.Supplier;

// Supplier<String> = không nhận gì → trả String
Supplier<String> greeting = () -> "Xin chào!";
System.out.println(greeting.get());  // "Xin chào!"
//                           ↑
//                    Gọi get() để lấy giá trị

// Supplier số ngẫu nhiên
Supplier<Double> randomNumber = () -> Math.random();
System.out.println(randomNumber.get());  // 0.7463...
System.out.println(randomNumber.get());  // 0.2891... (khác mỗi lần)

// Supplier ngày giờ hiện tại
Supplier<String> currentTime = () -> java.time.LocalDateTime.now().toString();

// ===== Lazy Initialization (Khởi tạo lười) =====
// Object CHỈ được tạo khi gọi get() → tiết kiệm tài nguyên
Supplier<List<String>> listFactory = () -> {
    System.out.println("Đang tạo danh sách mới...");
    return new ArrayList<>();
};

// Chưa tạo gì cả (lazy)
// ...
// Chỉ khi CẦN mới gọi:
List<String> list1 = listFactory.get();  // "Đang tạo danh sách mới..."
List<String> list2 = listFactory.get();  // "Đang tạo danh sách mới..." (instance MỚI)

// ===== Factory Pattern =====
Supplier<ArrayList<String>> newArrayList = ArrayList::new;
// Mỗi lần get() → tạo ArrayList mới
```

💡 **Khi nào dùng Supplier?**
- Lazy initialization (khởi tạo khi cần)
- Factory pattern (tạo object)
- Cung cấp giá trị mặc định

---

## 6. Predicate\<T\> (Kiểm tra: nhận T → trả boolean)

Predicate dùng để **kiểm tra điều kiện**. Trả về `true` hoặc `false`.

```java
import java.util.function.Predicate;

// Predicate<Integer> = nhận Integer → trả boolean
Predicate<Integer> isPositive = n -> n > 0;
System.out.println(isPositive.test(5));   // true
System.out.println(isPositive.test(-5));  // false
//                          ↑
//                   Gọi test() để kiểm tra

Predicate<String> isNotEmpty = s -> s != null && !s.isEmpty();
System.out.println(isNotEmpty.test("Hello"));  // true
System.out.println(isNotEmpty.test(""));       // false
```

### Kết hợp Predicate (Chaining)

```java
Predicate<Integer> isPositive = n -> n > 0;
Predicate<Integer> isEven = n -> n % 2 == 0;
Predicate<Integer> isLessThan100 = n -> n < 100;

// ===== AND: cả 2 điều kiện đều đúng =====
Predicate<Integer> isPositiveEven = isPositive.and(isEven);
System.out.println(isPositiveEven.test(4));   // true  (> 0 VÀ chẵn)
System.out.println(isPositiveEven.test(-4));  // false (không > 0)
System.out.println(isPositiveEven.test(3));   // false (không chẵn)

// ===== OR: ít nhất 1 điều kiện đúng =====
Predicate<Integer> isPositiveOrEven = isPositive.or(isEven);
System.out.println(isPositiveOrEven.test(-4));  // true (chẵn dù không > 0)

// ===== NEGATE: phủ định (đảo ngược) =====
Predicate<Integer> isNegative = isPositive.negate();  // NOT isPositive
System.out.println(isNegative.test(-5));  // true

// ===== Kết hợp nhiều điều kiện =====
Predicate<Integer> isValid = isPositive
    .and(isEven)
    .and(isLessThan100);
// Phải > 0 VÀ chẵn VÀ < 100
System.out.println(isValid.test(42));   // true
System.out.println(isValid.test(200));  // false (>= 100)

// ===== isEqual: so sánh bằng =====
Predicate<String> isAdmin = Predicate.isEqual("admin");
System.out.println(isAdmin.test("admin"));  // true
System.out.println(isAdmin.test("user"));   // false

// ===== Dùng trong filter =====
List<Integer> numbers = List.of(1, -2, 3, -4, 5, 6, -7, 8);
List<Integer> positiveEvens = numbers.stream()
    .filter(isPositive.and(isEven))
    .toList();
System.out.println(positiveEvens);  // [6, 8]
```

---

## 7. BiFunction và các biến thể (Nhận 2 tham số)

Khi cần **2 tham số đầu vào**, dùng các interface có prefix "Bi":

```java
import java.util.function.*;

// ===== BiFunction<T, U, R>: nhận T và U → trả R =====
BiFunction<Integer, Integer, Integer> add = (a, b) -> a + b;
System.out.println(add.apply(5, 3));  // 8

BiFunction<String, Integer, String> repeat = (s, n) -> s.repeat(n);
System.out.println(repeat.apply("Ha", 3));  // "HaHaHa"

// ===== BiConsumer<T, U>: nhận T và U → void =====
BiConsumer<String, Integer> printPair = (key, value) ->
    System.out.println(key + " = " + value);
printPair.accept("Tuổi", 25);  // Tuổi = 25

// Map.forEach dùng BiConsumer
Map<String, Integer> scores = Map.of("Toán", 9, "Lý", 8);
scores.forEach((subject, score) ->
    System.out.println(subject + ": " + score));

// ===== BiPredicate<T, U>: nhận T và U → boolean =====
BiPredicate<String, Integer> isLongerThan = (s, n) -> s.length() > n;
System.out.println(isLongerThan.test("Hello", 3));  // true (5 > 3)
System.out.println(isLongerThan.test("Hi", 3));     // false (2 > 3 = false)

// ===== UnaryOperator<T>: nhận T → trả T (cùng kiểu) =====
// Giống Function<T, T>
UnaryOperator<String> toUpper = s -> s.toUpperCase();
System.out.println(toUpper.apply("hello"));  // "HELLO"

// Dùng trong replaceAll
List<String> names = new ArrayList<>(List.of("an", "bình", "châu"));
names.replaceAll(String::toUpperCase);  // [AN, BÌNH, CHÂU]

// ===== BinaryOperator<T>: nhận T, T → trả T (cùng kiểu) =====
// Giống BiFunction<T, T, T>
BinaryOperator<Integer> sum = (a, b) -> a + b;
BinaryOperator<Integer> max = Integer::max;
System.out.println(sum.apply(10, 20));  // 30
System.out.println(max.apply(10, 20));  // 20
```

---

## 8. Method References (Tham chiếu phương thức)

### Tại sao cần Method Reference?

Khi lambda **chỉ gọi 1 method** và truyền tham số thẳng vào, bạn có thể viết ngắn hơn bằng method reference.

```java
// Lambda: gọi 1 method, truyền tham số thẳng
names.forEach(name -> System.out.println(name));
//                     ↑ Chỉ gọi println với name

// Method reference: ngắn hơn, bỏ phần thừa
names.forEach(System.out::println);
//                       ↑↑ Hai dấu :: = method reference
```

### 8.1. Bốn loại Method Reference

```java
// ===== LOẠI 1: Static method reference =====
// ClassName::staticMethod
// Lambda:            s -> Integer.parseInt(s)
// Method reference:  Integer::parseInt
Function<String, Integer> parse = Integer::parseInt;
System.out.println(parse.apply("123"));  // 123

// ===== LOẠI 2: Instance method của object CỤ THỂ =====
// object::instanceMethod
String greeting = "Xin chào";
// Lambda:            () -> greeting.length()
// Method reference:  greeting::length
Supplier<Integer> getLength = greeting::length;
System.out.println(getLength.get());  // 8

// ===== LOẠI 3: Instance method của object BẤT KỲ =====
// ClassName::instanceMethod
// Lambda:            s -> s.toUpperCase()
// Method reference:  String::toUpperCase
Function<String, String> toUpper = String::toUpperCase;
System.out.println(toUpper.apply("hello"));  // "HELLO"

// Lambda:            (s1, s2) -> s1.compareToIgnoreCase(s2)
// Method reference:  String::compareToIgnoreCase
Comparator<String> comp = String::compareToIgnoreCase;

// ===== LOẠI 4: Constructor reference =====
// ClassName::new
// Lambda:            () -> new ArrayList<>()
// Method reference:  ArrayList::new
Supplier<List<String>> newList = ArrayList::new;
List<String> list = newList.get();  // Tạo ArrayList mới

// Lambda:            s -> new Person(s)
// Method reference:  Person::new
Function<String, Person> createPerson = Person::new;
Person person = createPerson.apply("An");
```

### 8.2. Bảng tóm tắt 4 loại

| Loại | Cú pháp | Lambda tương đương | Ví dụ |
|------|---------|-------------------|-------|
| Static method | `ClassName::staticMethod` | `x -> ClassName.staticMethod(x)` | `Integer::parseInt` |
| Instance method (object cụ thể) | `object::method` | `() -> object.method()` | `str::length` |
| Instance method (object bất kỳ) | `ClassName::method` | `x -> x.method()` | `String::toUpperCase` |
| Constructor | `ClassName::new` | `() -> new ClassName()` | `ArrayList::new` |

### 8.3. Ví dụ thực tế

```java
List<String> names = List.of("Châu", "An", "Bình", "Dung");

// ===== Sắp xếp =====
// Lambda:
names.sort((s1, s2) -> s1.compareToIgnoreCase(s2));
// Method reference:
names.sort(String::compareToIgnoreCase);

// ===== Chuyển đổi =====
// Lambda:
List<String> upperNames = names.stream()
    .map(s -> s.toUpperCase())
    .toList();
// Method reference:
List<String> upperNames2 = names.stream()
    .map(String::toUpperCase)
    .toList();

// ===== In ra =====
// Lambda:
names.forEach(name -> System.out.println(name));
// Method reference:
names.forEach(System.out::println);

// ===== Parse =====
List<String> numberStrings = List.of("1", "2", "3", "4");
// Lambda:
List<Integer> numbers = numberStrings.stream()
    .map(s -> Integer.parseInt(s))
    .toList();
// Method reference:
List<Integer> numbers2 = numberStrings.stream()
    .map(Integer::parseInt)
    .toList();
```

💡 **Khi nào dùng method reference?** Khi lambda **chỉ gọi đúng 1 method** mà không có logic thêm. Nếu lambda có logic phức tạp → giữ nguyên lambda.

```java
// ✅ Dùng method reference (chỉ gọi 1 method)
names.forEach(System.out::println);

// ❌ KHÔNG dùng method reference (có logic thêm)
names.forEach(name -> System.out.println("Xin chào " + name));
// ↑ Có nối chuỗi → không thể rút gọn thành method reference
```

---

## 9. Ví dụ thực tế (Practical Examples)

### 9.1. Lọc và chuyển đổi danh sách

```java
record Person(String name, int age) {}

List<Person> people = List.of(
    new Person("An", 17),
    new Person("Bình", 25),
    new Person("Châu", 30),
    new Person("Dung", 16),
    new Person("Em", 22)
);

// Tạo các Functional Interface
Predicate<Person> isAdult = p -> p.age() >= 18;          // Kiểm tra >= 18 tuổi
Function<Person, String> getName = Person::name;           // Lấy tên
Function<String, String> toUpper = String::toUpperCase;    // Viết hoa

// Kết hợp: Lấy tên người lớn, viết hoa
List<String> adultNames = people.stream()
    .filter(isAdult)                // Lọc: chỉ >= 18 tuổi
    .map(getName)                   // Chuyển: Person → tên
    .map(toUpper)                   // Chuyển: tên → HOA
    .toList();

System.out.println(adultNames);  // [BÌNH, CHÂU, EM]
```

### 9.2. Strategy Pattern với Lambda

```java
// TRƯỚC: Phải tạo 3 class riêng cho 3 chiến lược thanh toán
// SAU: Dùng lambda → 3 dòng code!

public class PaymentProcessor {
    public void process(double amount, Consumer<Double> strategy) {
        System.out.println("Đang xử lý " + amount + " VNĐ...");
        strategy.accept(amount);
    }
}

// Các chiến lược thanh toán = các lambda
Consumer<Double> creditCard = amount ->
    System.out.println("Thanh toán " + amount + " VNĐ bằng Thẻ tín dụng");

Consumer<Double> momo = amount ->
    System.out.println("Thanh toán " + amount + " VNĐ bằng MoMo");

Consumer<Double> bankTransfer = amount ->
    System.out.println("Thanh toán " + amount + " VNĐ bằng Chuyển khoản");

// Sử dụng:
PaymentProcessor processor = new PaymentProcessor();
processor.process(500000, creditCard);    // Thanh toán 500000 VNĐ bằng Thẻ tín dụng
processor.process(200000, momo);          // Thanh toán 200000 VNĐ bằng MoMo
processor.process(1000000, bankTransfer); // Thanh toán 1000000 VNĐ bằng Chuyển khoản
```

### 9.3. Validate dữ liệu với Predicate

```java
// Tạo các rule validate
Predicate<String> notNull = s -> s != null;
Predicate<String> notEmpty = s -> !s.isEmpty();
Predicate<String> hasAtSign = s -> s.contains("@");
Predicate<String> longEnough = s -> s.length() >= 5;

// Kết hợp rule
Predicate<String> isValidEmail = notNull
    .and(notEmpty)
    .and(hasAtSign)
    .and(longEnough);

// Kiểm tra
System.out.println(isValidEmail.test("user@email.com"));  // true
System.out.println(isValidEmail.test("user"));             // false (thiếu @)
System.out.println(isValidEmail.test("a@b"));              // false (< 5 ký tự)
System.out.println(isValidEmail.test(""));                 // false (rỗng)

// Lọc email hợp lệ từ danh sách
List<String> emails = List.of("an@email.com", "invalid", "b@c", "binh@gmail.com");
List<String> validEmails = emails.stream()
    .filter(isValidEmail)
    .toList();
System.out.println(validEmails);  // [an@email.com, binh@gmail.com]
```

---

## 10. Sai lầm thường gặp

### Sai lầm 1: Quên rằng Lambda cần Functional Interface

```java
// ❌ SAI: Không thể gán lambda cho class thường
// String greeting = () -> "Hello";  // ERROR!

// ✅ ĐÚNG: Lambda phải gán cho Functional Interface
Supplier<String> greeting = () -> "Hello";
Function<String, Integer> getLen = s -> s.length();
```

### Sai lầm 2: Nhầm lẫn `andThen` và `compose`

```java
Function<Integer, Integer> doubleIt = x -> x * 2;
Function<Integer, Integer> addTen = x -> x + 10;

// andThen: doubleIt TRƯỚC → addTen SAU
doubleIt.andThen(addTen).apply(5);
// 5 * 2 = 10 → 10 + 10 = 20 ✅

// compose: addTen TRƯỚC → doubleIt SAU
doubleIt.compose(addTen).apply(5);
// 5 + 10 = 15 → 15 * 2 = 30
// ⚠️ Thứ tự NGƯỢC lại!

// 💡 Mẹo: Dùng andThen cho dễ đọc (theo thứ tự tự nhiên)
```

### Sai lầm 3: Sửa biến ngoài trong Lambda

```java
int count = 0;

// ❌ SAI: Không thể sửa biến ngoài
// names.forEach(name -> count++);  // ERROR! count phải effectively final

// ✅ Cách 1: Dùng mảng
int[] counter = {0};
names.forEach(name -> counter[0]++);

// ✅ Cách 2: Dùng AtomicInteger
AtomicInteger atomicCount = new AtomicInteger(0);
names.forEach(name -> atomicCount.incrementAndGet());
```

### Sai lầm 4: Lambda quá phức tạp

```java
// ❌ SAI: Lambda quá dài, khó đọc
Function<List<Person>, Map<String, List<Person>>> groupByCity = people ->
    people.stream()
        .filter(p -> p.getAge() > 18)
        .filter(p -> p.getCity() != null)
        .collect(Collectors.groupingBy(p -> {
            String city = p.getCity().trim().toLowerCase();
            return city.substring(0, 1).toUpperCase() + city.substring(1);
        }));

// ✅ ĐÚNG: Tách thành method riêng, lambda chỉ gọi method
Function<List<Person>, Map<String, List<Person>>> groupByCity2 =
    this::groupAdultsByCity;  // Method reference → rõ ràng hơn

private Map<String, List<Person>> groupAdultsByCity(List<Person> people) {
    return people.stream()
        .filter(this::isAdult)
        .filter(this::hasCity)
        .collect(Collectors.groupingBy(this::normalizeCity));
}
```

💡 **Quy tắc:** Lambda nên ngắn (1-2 dòng). Nếu dài → tách thành method riêng.

---

## 11. Tóm tắt cuối ngày

### Bảng tổng hợp kiến thức

| Khái niệm | Giải thích tiếng Việt | Cú pháp / Ví dụ |
|-----------|----------------------|-----------------|
| **Lambda** | Hàm ẩn danh (anonymous function) | `(x, y) -> x + y` |
| **Functional Interface** | Interface có 1 abstract method | `@FunctionalInterface` |
| **Function<T,R>** | Nhận T → trả R | `s -> s.length()` |
| **Consumer<T>** | Nhận T → void (xử lý) | `s -> System.out.println(s)` |
| **Supplier<T>** | Không nhận → trả T (cung cấp) | `() -> new ArrayList<>()` |
| **Predicate<T>** | Nhận T → boolean (kiểm tra) | `n -> n > 0` |
| **BiFunction<T,U,R>** | Nhận T, U → trả R | `(a, b) -> a + b` |
| **UnaryOperator<T>** | Nhận T → trả T (cùng kiểu) | `s -> s.toUpperCase()` |
| **Method Reference** | Rút gọn lambda | `String::toUpperCase` |
| **andThen** | Nối chuỗi: A rồi B | `f1.andThen(f2)` |
| **compose** | Ngược: B trước A | `f1.compose(f2)` |
| **Effectively Final** | Biến ngoài không đổi | Lambda chỉ đọc, không sửa |

### 🔥 Câu hỏi phỏng vấn thường gặp

1. **Lambda Expression là gì?**
   → Hàm ẩn danh, viết ngắn gọn cho Functional Interface. Cú pháp: `(params) -> expression`.

2. **Functional Interface là gì? Cho ví dụ?**
   → Interface có đúng 1 abstract method. Ví dụ: Runnable, Comparator, Function, Consumer, Predicate.

3. **Function vs Consumer vs Supplier vs Predicate?**
   → Function: T→R. Consumer: T→void. Supplier: ()→T. Predicate: T→boolean.

4. **Method Reference là gì? Có mấy loại?**
   → Viết ngắn gọn thay lambda khi chỉ gọi 1 method. 4 loại: static, instance cụ thể, instance bất kỳ, constructor.

5. **Effectively final là gì?**
   → Biến không bị gán lại sau khi khởi tạo. Lambda chỉ có thể truy cập biến effectively final bên ngoài.

6. **andThen vs compose?**
   → andThen: thực hiện theo thứ tự (A rồi B). compose: thực hiện ngược (B rồi A).

---

## 12. Bài tập thực hành

### Bài 1: Custom Functional Interface

Tạo `Transformer<T>` với method `transform(T input)` và default methods `andThen`, `compose`.

### Bài 2: Validation Framework

Tạo framework validate sử dụng Predicate:

```java
Validator<String> emailValidator = Validator.<String>of()
    .addRule(s -> s != null, "Email không được null")
    .addRule(s -> s.contains("@"), "Email phải có @")
    .addRule(s -> s.length() > 5, "Email quá ngắn");

ValidationResult result = emailValidator.validate("test@email.com");
System.out.println(result.isValid());   // true
System.out.println(result.getErrors()); // []
```

### Bài 3: Function Pipeline

Tạo pipeline xử lý dữ liệu:

```java
Pipeline<String, Integer> pipeline = Pipeline.<String>start()
    .then(String::trim)
    .then(String::toLowerCase)
    .then(String::length);

int result = pipeline.execute("  Hello World  ");  // 11
```

### Bài 4: Event System

Tạo hệ thống event dùng Consumer:

```java
EventBus bus = new EventBus();
bus.subscribe("userCreated", event -> System.out.println("User tạo: " + event));
bus.subscribe("userCreated", event -> sendEmail(event));
bus.publish("userCreated", "An");
```

---

## 13. Đáp án tham khảo

<details>
<summary>Bài 2: Validation Framework (Click để xem)</summary>

```java
import java.util.*;
import java.util.function.Predicate;

// Kết quả validate
class ValidationResult {
    private boolean valid;
    private List<String> errors;

    public ValidationResult(boolean valid, List<String> errors) {
        this.valid = valid;
        this.errors = errors;
    }

    public boolean isValid() { return valid; }
    public List<String> getErrors() { return errors; }
}

// Validator tổng quát
class Validator<T> {
    // Danh sách rule: mỗi rule = (điều kiện, thông báo lỗi)
    private List<Map.Entry<Predicate<T>, String>> rules = new ArrayList<>();

    public static <T> Validator<T> of() {
        return new Validator<>();
    }

    // Thêm rule: điều kiện + message lỗi nếu điều kiện SAI
    public Validator<T> addRule(Predicate<T> condition, String errorMessage) {
        rules.add(Map.entry(condition, errorMessage));
        return this; // Trả về this để chaining
    }

    // Validate: chạy tất cả rules
    public ValidationResult validate(T value) {
        List<String> errors = new ArrayList<>();

        for (var rule : rules) {
            if (!rule.getKey().test(value)) {  // Điều kiện SAI?
                errors.add(rule.getValue());    // Thêm lỗi
            }
        }

        return new ValidationResult(errors.isEmpty(), errors);
    }
}

// Sử dụng:
public class ValidatorDemo {
    public static void main(String[] args) {
        Validator<String> emailValidator = Validator.<String>of()
            .addRule(s -> s != null, "Email không được null")
            .addRule(s -> s.contains("@"), "Email phải chứa @")
            .addRule(s -> s.length() > 5, "Email quá ngắn");

        ValidationResult result1 = emailValidator.validate("test@email.com");
        System.out.println(result1.isValid());   // true
        System.out.println(result1.getErrors()); // []

        ValidationResult result2 = emailValidator.validate("test");
        System.out.println(result2.isValid());   // false
        System.out.println(result2.getErrors()); // [Email phải chứa @, Email quá ngắn]
    }
}
```
</details>

<details>
<summary>Bài 4: Event System (Click để xem)</summary>

```java
import java.util.*;
import java.util.function.Consumer;

public class EventBus {
    // Map: tên sự kiện → danh sách handler
    private Map<String, List<Consumer<Object>>> subscribers = new HashMap<>();

    // Đăng ký lắng nghe sự kiện
    public void subscribe(String eventType, Consumer<Object> handler) {
        // computeIfAbsent: nếu chưa có list → tạo mới
        subscribers.computeIfAbsent(eventType, k -> new ArrayList<>())
                   .add(handler);
    }

    // Hủy đăng ký
    public void unsubscribe(String eventType, Consumer<Object> handler) {
        List<Consumer<Object>> handlers = subscribers.get(eventType);
        if (handlers != null) {
            handlers.remove(handler);
        }
    }

    // Phát sự kiện → gọi tất cả handler đã đăng ký
    public void publish(String eventType, Object event) {
        List<Consumer<Object>> handlers = subscribers.get(eventType);
        if (handlers != null) {
            // Gọi từng handler
            handlers.forEach(handler -> handler.accept(event));
        }
    }

    public static void main(String[] args) {
        EventBus bus = new EventBus();

        // Đăng ký 2 handler cho sự kiện "userCreated"
        bus.subscribe("userCreated", event ->
            System.out.println("Handler 1: User được tạo - " + event));

        bus.subscribe("userCreated", event ->
            System.out.println("Handler 2: Gửi email chào mừng tới " + event));

        // Đăng ký handler cho sự kiện khác
        bus.subscribe("orderPlaced", event ->
            System.out.println("Đơn hàng mới: " + event));

        // Phát sự kiện → tất cả handler được gọi
        bus.publish("userCreated", "An");
        // Output:
        // Handler 1: User được tạo - An
        // Handler 2: Gửi email chào mừng tới An

        bus.publish("orderPlaced", "Đơn hàng #123");
        // Output:
        // Đơn hàng mới: Đơn hàng #123
    }
}
```
</details>

---

## Navigation

- [← Day 8: Generics (Kiểu Tổng Quát)](./day-08-generics.md)
- [Day 10: Stream API (Xử Lý Dữ Liệu Dòng) →](./day-10-stream-api.md)
