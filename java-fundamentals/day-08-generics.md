# Day 8: Generics (Kiểu Tổng Quát)

## Mục tiêu hôm nay

Sau khi học xong Day 8, bạn sẽ:
- Hiểu **Generics** (kiểu tổng quát) là gì và tại sao cần
- Tạo **Generic Class** (lớp tổng quát) và **Generic Method** (phương thức tổng quát)
- Sử dụng **Bounded Type Parameters** (giới hạn kiểu) với `extends`
- Hiểu **Wildcards** (ký tự đại diện): `?`, `? extends`, `? super`
- Nắm nguyên tắc **PECS** (Producer Extends, Consumer Super)
- Biết **Type Erasure** (xóa kiểu) — cách Generics hoạt động bên trong

---

## Tại sao cần học Generics?

### Vấn đề TRƯỚC khi có Generics (Java < 5)

```java
// TRƯỚC Java 5: List không có kiểu — chứa BẤT KỲ object nào
List list = new ArrayList();
list.add("Hello");    // Thêm String ✅
list.add(123);        // Thêm Integer ✅ (nhưng VÔ TÌNH!)
list.add(true);       // Thêm Boolean ✅ (hỗn loạn!)

// Lấy ra PHẢI ép kiểu (cast) → DỄ LỖI!
String s1 = (String) list.get(0);  // OK → "Hello"
String s2 = (String) list.get(1);  // CRASH! ClassCastException!
//          ↑ Integer không thể ép thành String!
```

**Vấn đề:** Lỗi chỉ phát hiện **lúc chạy** (runtime) → khó debug, nguy hiểm!

### Giải pháp: Generics (Java 5+)

```java
// SAU Java 5: List CÓ kiểu → an toàn
List<String> list = new ArrayList<>();  // Chỉ chứa String
list.add("Hello");    // OK ✅
// list.add(123);     // LỖI NGAY LÚC VIẾT CODE! ❌ Compiler báo lỗi
// list.add(true);    // LỖI NGAY LÚC VIẾT CODE! ❌

String s = list.get(0);  // KHÔNG CẦN ép kiểu! Compiler biết là String
```

**Lợi ích:**
- Lỗi phát hiện **ngay lúc viết code** (compile-time) → an toàn hơn
- **Không cần ép kiểu** (cast) → code sạch hơn
- **Tái sử dụng** code — 1 class dùng được cho nhiều kiểu dữ liệu

### Ví dụ đời thường

```
KHÔNG có Generics:
  Một chiếc hộp KHÔNG có nhãn:
  ┌─────────┐
  │  ???     │  ← Bỏ gì vào cũng được (táo, sách, giày...)
  └─────────┘    Lấy ra phải đoán "cái gì bên trong?" → dễ nhầm!

CÓ Generics:
  Hộp CÓ NHÃN rõ ràng:
  ┌─────────┐
  │ 🍎 Táo  │  ← Chỉ bỏ được TÁO vào
  └─────────┘    Lấy ra CHẮC CHẮN là táo → không nhầm!

  Box<String>  → Hộp chỉ chứa String
  Box<Integer> → Hộp chỉ chứa Integer
  Box<User>    → Hộp chỉ chứa User
```

---

## 1. Generic Classes (Lớp tổng quát)

### 1.1. Tạo Generic Class cơ bản

```java
// <T> = Type Parameter (tham số kiểu)
// T là "biến kiểu" — sẽ được thay thế bằng kiểu thực tế khi sử dụng
public class Box<T> {
    private T content;  // content có kiểu T (chưa biết cụ thể)

    public void set(T content) {
        this.content = content;
    }

    public T get() {
        return content;
    }

    @Override
    public String toString() {
        return "Box chứa: " + content;
    }
}
```

**Sử dụng:**

```java
public class BoxDemo {
    public static void main(String[] args) {

        // Box<String> → T được thay thế bằng String
        // → content là String, set() nhận String, get() trả về String
        Box<String> stringBox = new Box<>();
        stringBox.set("Xin chào!");
        String greeting = stringBox.get();  // Không cần cast!
        System.out.println(stringBox);      // Box chứa: Xin chào!

        // Box<Integer> → T được thay thế bằng Integer
        Box<Integer> intBox = new Box<>();
        intBox.set(42);
        int number = intBox.get();  // Tự động unboxing
        System.out.println(intBox); // Box chứa: 42

        // Box<User> → T được thay thế bằng User
        Box<User> userBox = new Box<>();
        userBox.set(new User("An", 25));
        User user = userBox.get();
    }
}
```

**Quá trình thay thế kiểu:**

```
Box<T> (khai báo)        Box<String> (sử dụng)
─────────────────        ────────────────────
private T content;   →   private String content;
void set(T content)  →   void set(String content)
T get()              →   String get()
```

### Quy ước đặt tên Type Parameter

| Ký hiệu | Ý nghĩa | Ví dụ |
|----------|---------|-------|
| `T` | **T**ype — kiểu chung | `Box<T>`, `List<T>` |
| `E` | **E**lement — phần tử | `List<E>`, `Set<E>` |
| `K` | **K**ey — khóa | `Map<K, V>` |
| `V` | **V**alue — giá trị | `Map<K, V>` |
| `N` | **N**umber — số | `MathBox<N>` |
| `R` | **R**eturn — kiểu trả về | `Function<T, R>` |

### 1.2. Generic Class với nhiều Type Parameter

```java
// Pair có 2 type parameters: K (Key) và V (Value)
public class Pair<K, V> {
    private K key;
    private V value;

    public Pair(K key, V value) {
        this.key = key;
        this.value = value;
    }

    public K getKey() { return key; }
    public V getValue() { return value; }

    @Override
    public String toString() {
        return "(" + key + ", " + value + ")";
    }
}

// Sử dụng:
// K = String, V = Integer
Pair<String, Integer> nameAge = new Pair<>("An", 25);
System.out.println(nameAge.getKey());    // "An"
System.out.println(nameAge.getValue());  // 25

// K = Integer, V = String
Pair<Integer, String> idName = new Pair<>(1, "Bình");

// K = String, V = List<String>
Pair<String, List<String>> config = new Pair<>("hosts", List.of("server1", "server2"));
```

### 1.3. Generic Class với giới hạn kiểu (Bounded)

```java
// <T extends Number> = T CHỈ được là Number hoặc con của Number
// → Integer, Double, Long... OK
// → String, Boolean... KHÔNG OK!
public class NumberBox<T extends Number> {
    private T number;

    public NumberBox(T number) {
        this.number = number;
    }

    // Vì T extends Number → có thể gọi method của Number
    public double getDoubleValue() {
        return number.doubleValue();  // Method của Number
    }

    public boolean isPositive() {
        return number.doubleValue() > 0;
    }
}

// Sử dụng:
NumberBox<Integer> intBox = new NumberBox<>(10);
System.out.println(intBox.getDoubleValue());  // 10.0

NumberBox<Double> doubleBox = new NumberBox<>(3.14);
System.out.println(doubleBox.isPositive());   // true

// NumberBox<String> sBox = new NumberBox<>("Hi");
// ❌ COMPILE ERROR! String không phải Number!
```

---

## 2. Generic Methods (Phương thức tổng quát)

### Tại sao cần Generic Method?

Đôi khi bạn không cần TOÀN BỘ class là generic, chỉ cần **1 method** hoạt động với nhiều kiểu.

### 2.1. Cú pháp

```java
//     ↓ Type parameter đặt TRƯỚC return type
public static <T> void printArray(T[] array) {
    for (T element : array) {
        System.out.print(element + " ");
    }
    System.out.println();
}
```

**Phân tích:**
```
public static <T>  void  printArray(T[] array)
               ↑    ↑                ↑
               │    │                └── Tham số: mảng kiểu T
               │    └── Kiểu trả về: void
               └── Khai báo type parameter T
```

### 2.2. Ví dụ

```java
public class GenericMethodDemo {

    // Generic method: in mảng bất kỳ kiểu nào
    public static <T> void printArray(T[] array) {
        System.out.print("[ ");
        for (T element : array) {
            System.out.print(element + " ");
        }
        System.out.println("]");
    }

    // Generic method: lấy phần tử đầu tiên
    public static <T> T getFirst(List<T> list) {
        if (list == null || list.isEmpty()) {
            return null;
        }
        return list.get(0);  // Trả về kiểu T
    }

    // Generic method: hoán đổi 2 phần tử trong mảng
    public static <T> void swap(T[] array, int i, int j) {
        T temp = array[i];
        array[i] = array[j];
        array[j] = temp;
    }

    public static void main(String[] args) {
        // Java TỰ ĐỘNG suy luận kiểu T từ tham số truyền vào

        // T = String (suy từ String[])
        String[] names = {"An", "Bình", "Châu"};
        printArray(names);    // [ An Bình Châu ]

        // T = Integer (suy từ Integer[])
        Integer[] numbers = {1, 2, 3};
        printArray(numbers);  // [ 1 2 3 ]

        // T = String (suy từ List<String>)
        List<String> list = List.of("X", "Y", "Z");
        String first = getFirst(list);  // "X"

        // Hoán đổi
        swap(names, 0, 2);
        printArray(names);    // [ Châu Bình An ]
    }
}
```

### 2.3. Generic Method với nhiều Type Parameter

```java
public class Utils {

    // Tạo Pair từ 2 giá trị bất kỳ
    public static <T, U> Pair<T, U> makePair(T first, U second) {
        return new Pair<>(first, second);
    }

    // In cặp key-value
    public static <K, V> void printPair(K key, V value) {
        System.out.println(key + " = " + value);
    }

    public static void main(String[] args) {
        Pair<String, Integer> pair = makePair("Tuổi", 25);
        // T = String, U = Integer (tự suy luận)

        printPair("Tên", "An");    // Tên = An
        printPair(1, true);         // 1 = true
    }
}
```

### 2.4. Generic Method với Bounded Type

```java
public class MathUtils {

    // T phải là Comparable → có thể so sánh được
    public static <T extends Comparable<T>> T findMax(List<T> list) {
        if (list == null || list.isEmpty()) {
            return null;
        }

        T max = list.get(0);  // Giả sử phần tử đầu là lớn nhất
        for (T item : list) {
            if (item.compareTo(max) > 0) {  // item > max?
                max = item;                  // Cập nhật max
            }
        }
        return max;
    }

    // T phải là Number → có thể gọi doubleValue()
    public static <T extends Number> double sum(List<T> list) {
        double total = 0;
        for (T num : list) {
            total += num.doubleValue();  // Method của Number
        }
        return total;
    }

    public static void main(String[] args) {
        List<Integer> numbers = List.of(3, 1, 4, 1, 5, 9);
        System.out.println("Max: " + findMax(numbers));   // Max: 9
        System.out.println("Tổng: " + sum(numbers));      // Tổng: 23.0

        List<String> names = List.of("Châu", "An", "Bình");
        System.out.println("Max: " + findMax(names));      // Max: Châu (theo bảng chữ cái)

        // sum(names);  // ❌ ERROR! String không phải Number
    }
}
```

---

## 3. Bounded Type Parameters (Giới hạn kiểu)

### 3.1. Upper Bounded: `<T extends X>` (T phải là X hoặc con của X)

```java
// T PHẢI là Number hoặc con của Number (Integer, Double, Long...)
public class Calculator<T extends Number> {
    private T value;

    public Calculator(T value) {
        this.value = value;
    }

    public double square() {
        // Vì T extends Number → chắc chắn có method doubleValue()
        return value.doubleValue() * value.doubleValue();
    }
}

Calculator<Integer> calc1 = new Calculator<>(5);
System.out.println(calc1.square());  // 25.0

Calculator<Double> calc2 = new Calculator<>(3.14);
System.out.println(calc2.square());  // 9.8596

// Calculator<String> calc3 = new Calculator<>("Hi");
// ❌ COMPILE ERROR! String không extends Number
```

### 3.2. Multiple Bounds: `<T extends A & B & C>`

```java
// T phải VỪA là Number VỪA implement Comparable
// → Có thể tính toán VÀ so sánh
public class SortableNumber<T extends Number & Comparable<T>> {
    private T value;

    public SortableNumber(T value) {
        this.value = value;
    }

    public boolean isGreaterThan(T other) {
        return value.compareTo(other) > 0;  // Từ Comparable
    }

    public double toDouble() {
        return value.doubleValue();          // Từ Number
    }
}

// Integer extends Number ✅ VÀ implements Comparable<Integer> ✅
SortableNumber<Integer> num = new SortableNumber<>(10);
System.out.println(num.isGreaterThan(5));   // true
System.out.println(num.toDouble());          // 10.0
```

⚠️ **Quy tắc Multiple Bounds:** Class phải đặt **trước**, interface đặt **sau**:
```java
// ✅ ĐÚNG: Class trước, interface sau
<T extends SomeClass & InterfaceA & InterfaceB>

// ❌ SAI: Interface trước class
<T extends InterfaceA & SomeClass>
```

---

## 4. Wildcards (Ký tự đại diện `?`)

### Tại sao cần Wildcards?

```java
// Bạn muốn viết method in bất kỳ List nào
public static void printList(List<Object> list) {
    for (Object item : list) {
        System.out.println(item);
    }
}

List<String> names = List.of("An", "Bình");
// printList(names);  // ❌ ERROR!
// List<String> KHÔNG PHẢI là List<Object>!
// Dù String là con của Object, nhưng List<String> KHÔNG phải con của List<Object>
```

⚠️ **Bẫy quan trọng:** `List<String>` **KHÔNG** phải là subtype của `List<Object>`!

```
Object ← String (String là con của Object ✅)
List<Object> ← List<String> (KHÔNG phải! ❌)
```

**Giải pháp:** Dùng Wildcard `?`

### 4.1. Unbounded Wildcard: `?` (Bất kỳ kiểu nào)

```java
// List<?> = "List của BẤT KỲ kiểu nào"
public static void printList(List<?> list) {
    for (Object item : list) {  // Lấy ra dưới dạng Object
        System.out.println(item);
    }
}

List<String> names = List.of("An", "Bình");
List<Integer> numbers = List.of(1, 2, 3);

printList(names);    // ✅ OK!
printList(numbers);  // ✅ OK!

// ⚠️ Nhưng KHÔNG THỂ thêm phần tử vào List<?>
// list.add("Hello");  // ❌ ERROR! Không biết kiểu chính xác
```

### 4.2. Upper Bounded Wildcard: `? extends X` (X hoặc con của X)

**Dùng khi:** Bạn muốn **ĐỌC** (read) từ collection.

```java
// List<? extends Number> = "List của Number HOẶC con của Number"
// → Chấp nhận: List<Number>, List<Integer>, List<Double>, List<Long>...
public static double sumOfList(List<? extends Number> list) {
    double sum = 0;
    for (Number num : list) {       // Lấy ra dưới dạng Number
        sum += num.doubleValue();
    }
    return sum;
}

List<Integer> integers = List.of(1, 2, 3);
List<Double> doubles = List.of(1.1, 2.2, 3.3);
List<Long> longs = List.of(100L, 200L, 300L);

System.out.println(sumOfList(integers));  // 6.0  ✅
System.out.println(sumOfList(doubles));   // 6.6  ✅
System.out.println(sumOfList(longs));     // 600.0 ✅

// ⚠️ CHỈ ĐỌC được, KHÔNG GHI được!
// list.add(1);  // ❌ ERROR! Vì không biết chính xác kiểu bên trong
```

### 4.3. Lower Bounded Wildcard: `? super X` (X hoặc cha của X)

**Dùng khi:** Bạn muốn **GHI** (write) vào collection.

```java
// List<? super Integer> = "List của Integer HOẶC cha của Integer"
// → Chấp nhận: List<Integer>, List<Number>, List<Object>
public static void addNumbers(List<? super Integer> list) {
    list.add(1);    // ✅ Thêm Integer vào được!
    list.add(2);
    list.add(3);
}

List<Integer> intList = new ArrayList<>();
List<Number> numList = new ArrayList<>();
List<Object> objList = new ArrayList<>();

addNumbers(intList);  // ✅ Integer super Integer
addNumbers(numList);  // ✅ Number super Integer
addNumbers(objList);  // ✅ Object super Integer

// ⚠️ ĐỌC chỉ trả về Object (vì không biết kiểu chính xác)
// Integer i = list.get(0);  // ❌ ERROR!
Object obj = intList.get(0);  // ✅ OK nhưng phải dùng Object
```

### 4.4. 🔥 PECS: Producer Extends, Consumer Super

Đây là nguyên tắc VÀNG khi dùng Wildcards, **RẤT HAY GẶP trong phỏng vấn**!

```
PECS = Producer Extends, Consumer Super

Producer (Nguồn — ĐỌC dữ liệu ra):
  → Dùng <? extends T>
  → "Tôi SẢN XUẤT dữ liệu cho bạn đọc"
  → CHỈ ĐỌC, không ghi

Consumer (Đích — GHI dữ liệu vào):
  → Dùng <? super T>
  → "Tôi TIÊU THỤ dữ liệu bạn ghi vào"
  → CHỈ GHI, đọc ra Object
```

**Ví dụ kinh điển: Copy danh sách**

```java
// Copy từ nguồn (source) sang đích (dest)
public static <T> void copy(
        List<? extends T> source,  // Producer — đọc từ đây → extends
        List<? super T> dest       // Consumer — ghi vào đây → super
) {
    for (T item : source) {   // ĐỌC từ source (Producer)
        dest.add(item);       // GHI vào dest (Consumer)
    }
}

// Sử dụng:
List<Integer> source = List.of(1, 2, 3);   // Producer
List<Number> dest = new ArrayList<>();       // Consumer

copy(source, dest);
// Integer extends Number ✅
// Number super Integer ✅

System.out.println(dest);  // [1, 2, 3]
```

**Bảng tóm tắt PECS:**

| Tình huống | Dùng gì? | Có thể làm gì? | Ví dụ |
|-----------|----------|-----------------|-------|
| Chỉ **ĐỌC** từ collection | `? extends T` | ĐỌC ✅ GHI ❌ | `sumOfList(List<? extends Number>)` |
| Chỉ **GHI** vào collection | `? super T` | ĐỌC ❌* GHI ✅ | `addNumbers(List<? super Integer>)` |
| **Đọc VÀ Ghi** | `T` (không wildcard) | ĐỌC ✅ GHI ✅ | `process(List<T> list)` |
| Không cần biết kiểu | `?` | ĐỌC Object ✅ GHI ❌ | `printList(List<?>)` |

*Đọc chỉ trả về Object

---

## 5. Type Erasure (Xóa kiểu — Cách Generics hoạt động bên trong)

### 5.1. Generics chỉ tồn tại lúc COMPILE, không tồn tại lúc RUNTIME

```java
// Lúc bạn VIẾT code (compile-time):
List<String> strings = new ArrayList<>();
List<Integer> numbers = new ArrayList<>();

// Lúc Java CHẠY code (runtime) — Generics bị XÓA:
List strings = new ArrayList();   // ← String biến mất!
List numbers = new ArrayList();   // ← Integer biến mất!

// JVM KHÔNG BIẾT generic type lúc runtime!
```

**Tại sao Java làm vậy?** Để đảm bảo **backward compatibility** (tương thích ngược) với code Java cũ (trước Java 5).

### 5.2. Những gì KHÔNG THỂ làm với Generics

```java
public class MyClass<T> {
    // ❌ Không thể tạo object từ type parameter
    // T obj = new T();
    // → Vì runtime không biết T là gì!

    // ❌ Không thể tạo mảng từ type parameter
    // T[] arr = new T[10];

    // ❌ Không thể dùng instanceof với type parameter
    // if (obj instanceof T) { }

    // ❌ Không thể tạo generic exception
    // class MyException<T> extends Exception { }

    // ✅ Workaround: truyền Class<T> để tạo object
    public T createInstance(Class<T> clazz) throws Exception {
        return clazz.getDeclaredConstructor().newInstance();
    }
}

// Sử dụng workaround:
MyClass<String> mc = new MyClass<>();
String s = mc.createInstance(String.class);
```

### 5.3. Hệ quả thực tế

```java
// 2 List khác kiểu nhưng runtime CÙNG class!
List<String> strings = new ArrayList<>();
List<Integer> numbers = new ArrayList<>();

// Runtime: cả 2 đều là ArrayList (không có thông tin generic)
System.out.println(strings.getClass() == numbers.getClass());
// true! Cùng class ArrayList
```

---

## 6. Generic Interfaces (Interface tổng quát)

### Ví dụ thực tế: Repository Pattern

Đây là pattern **RẤT PHỔ BIẾN** trong dự án thực tế (Spring Boot, ABP...).

```java
// Generic interface cho CRUD operations
// T = kiểu Entity, ID = kiểu của primary key
public interface Repository<T, ID> {
    T findById(ID id);         // Tìm theo ID
    List<T> findAll();         // Lấy tất cả
    T save(T entity);          // Lưu (tạo mới hoặc cập nhật)
    void delete(ID id);        // Xóa theo ID
    boolean existsById(ID id); // Kiểm tra tồn tại
}

// Implementation cho User (T = User, ID = Long)
public class UserRepository implements Repository<User, Long> {
    private Map<Long, User> database = new HashMap<>();

    @Override
    public User findById(Long id) {
        return database.get(id);
    }

    @Override
    public List<User> findAll() {
        return new ArrayList<>(database.values());
    }

    @Override
    public User save(User entity) {
        database.put(entity.getId(), entity);
        return entity;
    }

    @Override
    public void delete(Long id) {
        database.remove(id);
    }

    @Override
    public boolean existsById(Long id) {
        return database.containsKey(id);
    }
}

// Implementation cho Product (T = Product, ID = String)
public class ProductRepository implements Repository<Product, String> {
    // Cùng interface, khác kiểu dữ liệu!
    // T = Product, ID = String (mã sản phẩm)

    @Override
    public Product findById(String id) { /* ... */ }
    // ...
}
```

💡 **Đây là sức mạnh của Generics:** Viết 1 interface, dùng cho **HÀNG TRĂM** entity khác nhau!

---

## 7. Sai lầm thường gặp

### Sai lầm 1: Nghĩ List\<String\> là con của List\<Object\>

```java
// ❌ SAI: Đây KHÔNG phải quan hệ cha-con!
List<Object> objects = new ArrayList<String>();  // COMPILE ERROR!

// Tại sao? Vì nếu cho phép:
List<Object> objects = stringList;
objects.add(123);  // Bỏ Integer vào List<String>??? Hỗn loạn!

// ✅ ĐÚNG: Dùng wildcard nếu cần
List<?> anything = new ArrayList<String>();           // OK
List<? extends Object> anything2 = new ArrayList<String>(); // OK
```

### Sai lầm 2: Thêm phần tử vào List\<? extends X\>

```java
List<? extends Number> numbers = new ArrayList<Integer>();

// ❌ KHÔNG THỂ thêm phần tử!
// numbers.add(1);      // ERROR!
// numbers.add(1.0);    // ERROR!

// Tại sao? Vì compiler không biết list THỰC SỰ chứa kiểu gì
// Có thể là List<Integer>, List<Double>, List<Long>...
// Thêm Integer vào List<Double>? Không an toàn!

// ✅ CHỈ CÓ THỂ ĐỌC
Number n = numbers.get(0);  // OK — đọc ra Number
```

### Sai lầm 3: Dùng primitive type cho Generic

```java
// ❌ SAI: Generics KHÔNG dùng được primitive type
// List<int> numbers = new ArrayList<>();     // ERROR!
// Box<double> box = new Box<>();             // ERROR!

// ✅ ĐÚNG: Dùng Wrapper class
List<Integer> numbers = new ArrayList<>();    // OK
Box<Double> box = new Box<>();                // OK
```

### Sai lầm 4: So sánh generic type lúc runtime

```java
// ❌ Không có ý nghĩa do Type Erasure
public static <T> boolean isString(T obj) {
    // return obj instanceof T;  // ERROR! T bị xóa lúc runtime
    return obj instanceof String;  // ✅ Phải dùng class cụ thể
}
```

---

## 8. Tóm tắt cuối ngày

### Bảng tổng hợp kiến thức

| Khái niệm | Giải thích tiếng Việt | Cú pháp |
|-----------|----------------------|---------|
| **Generic Class** | Lớp tổng quát | `class Box<T> { }` |
| **Generic Method** | Phương thức tổng quát | `<T> void print(T item)` |
| **Type Parameter** | Tham số kiểu (biến kiểu) | `T`, `E`, `K`, `V` |
| **Bounded Type** | Giới hạn kiểu | `<T extends Number>` |
| **Multiple Bounds** | Nhiều giới hạn | `<T extends A & B>` |
| **Unbounded Wildcard** | Bất kỳ kiểu | `List<?>` |
| **Upper Bounded Wildcard** | Kiểu X hoặc con | `List<? extends X>` |
| **Lower Bounded Wildcard** | Kiểu X hoặc cha | `List<? super X>` |
| **PECS** | Producer Extends, Consumer Super | Đọc→extends, Ghi→super |
| **Type Erasure** | Xóa kiểu lúc runtime | Generics chỉ tồn tại compile-time |

### 🔥 Câu hỏi phỏng vấn thường gặp

1. **Generics là gì? Tại sao cần?**
   → Cho phép viết code tái sử dụng cho nhiều kiểu. Type-safe lúc compile-time, không cần cast.

2. **`? extends T` vs `? super T` khác nhau thế nào?**
   → `extends`: đọc (Producer), chấp nhận T hoặc con. `super`: ghi (Consumer), chấp nhận T hoặc cha.

3. **PECS là gì?**
   → Producer Extends, Consumer Super. Đọc từ source dùng extends, ghi vào dest dùng super.

4. **Type Erasure là gì?**
   → Compiler xóa thông tin generic lúc runtime. `List<String>` trở thành `List` lúc chạy.

5. **Tại sao `List<String>` không phải subtype của `List<Object>`?**
   → Vì nếu cho phép, có thể thêm Object khác kiểu vào List<String> → phá vỡ type safety.

6. **Có thể dùng primitive type cho Generics không?**
   → Không. Phải dùng Wrapper: `List<Integer>` thay vì `List<int>`.

---

## 9. Bài tập thực hành

### Bài 1: Generic Stack (Ngăn xếp tổng quát)

Implement Stack với generics:
- `push(T item)` — Thêm phần tử lên đỉnh
- `pop(): T` — Lấy và xóa phần tử đỉnh
- `peek(): T` — Xem phần tử đỉnh (không xóa)
- `isEmpty(): boolean` — Kiểm tra rỗng
- `size(): int` — Số phần tử

### Bài 2: Generic Filter (Lọc tổng quát)

Tạo method lọc danh sách theo điều kiện bất kỳ:

```java
List<Integer> evens = filter(numbers, n -> n % 2 == 0);
List<String> longNames = filter(names, s -> s.length() > 5);
```

### Bài 3: Generic Cache (Bộ nhớ đệm tổng quát)

Implement cache có thời gian hết hạn (TTL):

```java
Cache<String, User> cache = new Cache<>(60000); // 60 giây
cache.put("user1", user);
User cached = cache.get("user1"); // null nếu đã hết hạn
```

### Bài 4: Generic Pair Utils

Tạo utility class:
- `swap(Pair<K,V>)`: Pair<V,K> — Hoán đổi key và value
- `toMap(List<Pair<K,V>>)`: Map<K,V> — Chuyển list cặp thành Map

### Bài 5: Generic Binary Tree (Cây nhị phân tổng quát)

```java
BinaryTree<Integer> tree = new BinaryTree<>();
tree.insert(5);
tree.insert(3);
tree.insert(7);
boolean found = tree.contains(3);          // true
List<Integer> sorted = tree.inorderTraversal(); // [3, 5, 7]
```

---

## 10. Đáp án tham khảo

<details>
<summary>Bài 1: Generic Stack (Click để xem)</summary>

```java
import java.util.*;

public class Stack<T> {
    private List<T> items = new ArrayList<>(); // Dùng ArrayList bên trong

    // Thêm phần tử lên đỉnh
    public void push(T item) {
        items.add(item); // Thêm vào cuối = đỉnh stack
    }

    // Lấy và XÓA phần tử đỉnh
    public T pop() {
        if (isEmpty()) {
            throw new EmptyStackException(); // Stack rỗng → lỗi
        }
        return items.remove(items.size() - 1); // Xóa phần tử cuối
    }

    // Xem phần tử đỉnh (KHÔNG xóa)
    public T peek() {
        if (isEmpty()) {
            throw new EmptyStackException();
        }
        return items.get(items.size() - 1); // Lấy phần tử cuối
    }

    public boolean isEmpty() {
        return items.isEmpty();
    }

    public int size() {
        return items.size();
    }

    public static void main(String[] args) {
        // Stack<String>
        Stack<String> stack = new Stack<>();
        stack.push("A");
        stack.push("B");
        stack.push("C");

        System.out.println(stack.peek());  // C (đỉnh)
        System.out.println(stack.pop());   // C (lấy ra + xóa)
        System.out.println(stack.pop());   // B
        System.out.println(stack.size());  // 1 (chỉ còn A)

        // Stack<Integer>
        Stack<Integer> numStack = new Stack<>();
        numStack.push(10);
        numStack.push(20);
        System.out.println(numStack.pop());  // 20
    }
}
```
</details>

<details>
<summary>Bài 2: Generic Filter (Click để xem)</summary>

```java
import java.util.*;
import java.util.function.Predicate;

public class FilterUtils {

    // Lọc danh sách theo điều kiện (Predicate)
    public static <T> List<T> filter(List<T> list, Predicate<T> condition) {
        List<T> result = new ArrayList<>();
        for (T item : list) {
            if (condition.test(item)) { // Kiểm tra điều kiện
                result.add(item);
            }
        }
        return result;
    }

    public static void main(String[] args) {
        List<Integer> numbers = List.of(1, 2, 3, 4, 5, 6, 7, 8, 9, 10);

        // Lọc số chẵn
        List<Integer> evens = filter(numbers, n -> n % 2 == 0);
        System.out.println("Số chẵn: " + evens);  // [2, 4, 6, 8, 10]

        // Lọc số > 5
        List<Integer> big = filter(numbers, n -> n > 5);
        System.out.println("Số > 5: " + big);     // [6, 7, 8, 9, 10]

        List<String> names = List.of("An", "Bình", "Châu", "Dung", "Em");
        // Lọc tên dài > 2 ký tự
        List<String> longNames = filter(names, s -> s.length() > 2);
        System.out.println("Tên dài: " + longNames);  // [Bình, Châu, Dung]
    }
}
```
</details>

<details>
<summary>Bài 3: Generic Cache (Click để xem)</summary>

```java
import java.util.*;

public class Cache<K, V> {
    private Map<K, CacheEntry<V>> store = new HashMap<>();
    private long ttlMillis; // Thời gian sống (milliseconds)

    public Cache(long ttlMillis) {
        this.ttlMillis = ttlMillis;
    }

    // Lưu vào cache
    public void put(K key, V value) {
        store.put(key, new CacheEntry<>(value, System.currentTimeMillis()));
    }

    // Lấy từ cache (null nếu hết hạn hoặc không có)
    public V get(K key) {
        CacheEntry<V> entry = store.get(key);
        if (entry == null) return null;

        // Kiểm tra hết hạn chưa
        if (System.currentTimeMillis() - entry.createdAt > ttlMillis) {
            store.remove(key); // Xóa entry hết hạn
            return null;
        }
        return entry.value;
    }

    public void remove(K key) {
        store.remove(key);
    }

    public void clear() {
        store.clear();
    }

    public int size() {
        return store.size();
    }

    // Class lưu value + thời điểm tạo
    private static class CacheEntry<V> {
        V value;
        long createdAt;

        CacheEntry(V value, long createdAt) {
            this.value = value;
            this.createdAt = createdAt;
        }
    }

    public static void main(String[] args) throws InterruptedException {
        Cache<String, String> cache = new Cache<>(1000); // TTL = 1 giây

        cache.put("key1", "value1");
        System.out.println(cache.get("key1")); // "value1" ✅

        Thread.sleep(1500); // Đợi 1.5 giây
        System.out.println(cache.get("key1")); // null (đã hết hạn)
    }
}
```
</details>

---

## Navigation

- [← Day 7: Collections Basics (Bộ Sưu Tập)](./day-07-collections-basics.md)
- [Day 9: Lambda & Functional (Lambda & Hàm) →](./day-09-lambda-functional.md)
