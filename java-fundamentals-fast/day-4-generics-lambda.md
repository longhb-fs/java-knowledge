# Day 4: Generics + Lambda + Functional Programming

> Gộp từ bản 19 ngày: Day 8 (Generics) + Day 9 (Lambda & Functional)
> 📖 Đọc sâu: [day-08](../java-fundamentals/day-08-generics.md) | [day-09](../java-fundamentals/day-09-lambda-functional.md)

---

## Phần A: Generics (Kiểu tổng quát)

### 1. Tại sao cần Generics?

```java
// ❌ Không có Generics → phải cast, dễ sai lúc runtime
List list = new ArrayList();
list.add("Hello");
list.add(123);                    // Thêm bất kỳ kiểu → nguy hiểm
String s = (String) list.get(1);  // 💥 ClassCastException!

// ✅ Có Generics → compiler kiểm tra kiểu lúc compile
List<String> list = new ArrayList<>();
list.add("Hello");
// list.add(123);                 // ❌ COMPILE ERROR — phát hiện sai sớm
String s = list.get(0);           // Không cần cast
```

### 2. Tạo Generic Class / Method

```java
// Generic class — T là "type parameter" (placeholder cho kiểu dữ liệu)
public class Box<T> {
    private T value;

    public Box(T value) { this.value = value; }
    public T getValue() { return value; }
}

Box<String> strBox = new Box<>("Hello");   // T = String
Box<Integer> intBox = new Box<>(42);       // T = Integer

// Generic method
public static <T> T firstOrNull(List<T> list) {
    return list.isEmpty() ? null : list.get(0);
}

String first = firstOrNull(List.of("A", "B"));  // Compiler tự suy ra T = String
```

### 3. Bounded Types (Giới hạn kiểu)

```java
// T phải là Number hoặc subclass (Integer, Double, Long...)
public static <T extends Number> double sum(List<T> list) {
    return list.stream().mapToDouble(Number::doubleValue).sum();
}

sum(List.of(1, 2, 3));        // ✅ Integer extends Number
sum(List.of(1.5, 2.5));       // ✅ Double extends Number
// sum(List.of("a", "b"));    // ❌ String không extends Number

// Multiple bounds
<T extends Comparable<T> & Serializable>  // T phải implement cả 2
```

### 4. Wildcards — `?`

```java
// ? extends T — "covariant" — chỉ ĐỌC (producer)
public static double sum(List<? extends Number> list) {
    double total = 0;
    for (Number n : list) total += n.doubleValue();  // ✅ Đọc OK
    // list.add(1);                                   // ❌ Không thể add
    return total;
}
sum(List.of(1, 2, 3));       // List<Integer> → OK
sum(List.of(1.5, 2.5));      // List<Double> → OK

// ? super T — "contravariant" — chỉ GHI (consumer)
public static void addNumbers(List<? super Integer> list) {
    list.add(1);              // ✅ Ghi OK
    list.add(2);
    // Integer n = list.get(0); // ❌ Đọc ra kiểu Object, không phải Integer
}
addNumbers(new ArrayList<Number>());   // OK
addNumbers(new ArrayList<Object>());   // OK
```

💡 **PECS = Producer Extends, Consumer Super**
- **Đọc** từ collection → `? extends T`
- **Ghi** vào collection → `? super T`
- **Cả hai** → dùng `T` cụ thể

---

## Phần B: Lambda & Functional Programming

### 1. Lambda Expression

```java
// Cú pháp: (params) -> expression
//     hoặc (params) -> { statements; }

// Trước Java 8: anonymous class
Comparator<String> comp = new Comparator<String>() {
    @Override
    public int compare(String a, String b) { return a.compareTo(b); }
};

// Từ Java 8: lambda — 1 dòng
Comparator<String> comp = (a, b) -> a.compareTo(b);

// Ví dụ khác
Runnable task = () -> System.out.println("Running");       // Không tham số
Consumer<String> print = s -> System.out.println(s);       // 1 tham số
BiFunction<Integer, Integer, Integer> add = (a, b) -> a + b; // 2 tham số
```

### 2. Functional Interfaces — 4 loại chính

| Interface | Input → Output | Method | Dùng khi |
|-----------|---------------|--------|----------|
| `Function<T,R>` | T → R | `apply()` | **Biến đổi** dữ liệu |
| `Consumer<T>` | T → void | `accept()` | **Xử lý** (print, log, save) |
| `Supplier<T>` | () → T | `get()` | **Tạo/cung cấp** dữ liệu |
| `Predicate<T>` | T → boolean | `test()` | **Kiểm tra** điều kiện |

```java
// Function: biến đổi
Function<String, Integer> toLength = s -> s.length();
toLength.apply("Hello");  // 5

// Consumer: xử lý, không trả kết quả
Consumer<String> log = s -> System.out.println("[LOG] " + s);
log.accept("Server started");  // [LOG] Server started

// Supplier: tạo dữ liệu
Supplier<LocalDate> today = LocalDate::now;
today.get();  // 2026-02-09

// Predicate: kiểm tra
Predicate<Integer> isEven = n -> n % 2 == 0;
isEven.test(4);  // true
```

### 3. Chọn Functional Interface nào? — Decision Guide

```
Bạn cần hàm làm gì?
│
├── Nhận input → trả output (biến đổi)?
│   ├── 1 input → Function<T, R>
│   ├── 2 inputs → BiFunction<T, U, R>
│   └── Input & output cùng kiểu → UnaryOperator<T>
│
├── Nhận input → không trả gì (xử lý)?
│   ├── 1 input → Consumer<T>
│   └── 2 inputs → BiConsumer<T, U>
│
├── Không nhận → trả output (tạo/cung cấp)?
│   └── Supplier<T>
│
├── Nhận input → trả true/false (kiểm tra)?
│   ├── 1 input → Predicate<T>
│   └── 2 inputs → BiPredicate<T, U>
│
└── 2 inputs cùng kiểu → cùng kiểu output?
    └── BinaryOperator<T>
```

### 4. Chaining (Nối chuỗi hàm)

```java
// Function chaining
Function<String, String> toUpper = String::toUpperCase;
Function<String, String> addBrackets = s -> "[" + s + "]";

Function<String, String> format = toUpper.andThen(addBrackets);
format.apply("hello");  // "[HELLO]"

// Predicate chaining
Predicate<Integer> isPositive = n -> n > 0;
Predicate<Integer> isEven = n -> n % 2 == 0;

Predicate<Integer> isPositiveEven = isPositive.and(isEven);
isPositiveEven.test(4);   // true
isPositiveEven.test(-4);  // false
isPositiveEven.test(3);   // false

// Negate
Predicate<Integer> isOdd = isEven.negate();
isOdd.test(3);  // true

// Consumer chaining
Consumer<User> log = u -> System.out.println("Created: " + u);
Consumer<User> sendEmail = u -> System.out.println("Email sent to: " + u);
Consumer<User> onUserCreated = log.andThen(sendEmail);
```

### 5. Method Reference — Rút gọn Lambda

```java
// Khi lambda CHỈ gọi 1 method → dùng method reference (::)

// Static method:       ClassName::staticMethod
Function<String, Integer> parse = Integer::parseInt;    // s -> Integer.parseInt(s)

// Instance method (kiểu): ClassName::instanceMethod
Function<String, String> upper = String::toUpperCase;   // s -> s.toUpperCase()

// Instance method (object): object::method
System.out::println                                     // s -> System.out.println(s)

// Constructor:         ClassName::new
Supplier<List<String>> newList = ArrayList::new;         // () -> new ArrayList<>()
```

💡 **Quy tắc:** Lambda chỉ gọi đúng 1 method, không có logic thêm → dùng method reference. Có logic → giữ lambda.

---

## Phần C: Tổng hợp — Kết hợp Generics + Lambda

```java
// Generic method + Lambda → code linh hoạt + type-safe
public static <T> List<T> filter(List<T> list, Predicate<T> condition) {
    List<T> result = new ArrayList<>();
    for (T item : list) {
        if (condition.test(item)) result.add(item);
    }
    return result;
}

// Sử dụng:
List<Integer> nums = List.of(1, -2, 3, -4, 5);
List<Integer> positives = filter(nums, n -> n > 0);      // [1, 3, 5]

List<String> names = List.of("An", "", "Bình", "", "Châu");
List<String> nonEmpty = filter(names, s -> !s.isEmpty()); // ["An", "Bình", "Châu"]
```

---

## Bài tập

1. **Generic Pair**: Tạo `Pair<A, B>` chứa 2 giá trị khác kiểu. Thêm method `map()` nhận Function để biến đổi.
2. **Validator**: Dùng `Predicate` tạo email validator: notNull AND contains("@") AND length > 5
3. **Pipeline**: Dùng `Function.andThen()` tạo pipeline: trim → lowercase → replace spaces with dashes

---

## Navigation

- [← Day 3: Exception + String + Collection](./day-3-exception-string-collection.md)
- [Day 5: Stream + I/O + DateTime →](./day-5-stream-io-datetime.md)
