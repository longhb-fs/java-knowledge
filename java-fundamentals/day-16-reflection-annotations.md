# Day 16: Reflection & Annotations (Phản Chiếu & Chú Thích)

## Mục tiêu hôm nay
- Reflection (phản chiếu) là gì và tại sao các framework dùng nó
- Cách lấy thông tin Class, Fields, Methods, Constructors tại runtime
- Annotations (chú thích) là gì - @Override, @Deprecated...
- Tạo Custom Annotation (chú thích tùy chỉnh)
- Đọc Annotation bằng Reflection
- Ví dụ thực tế: Simple Validator

---

## 🤔 Tại sao cần học Reflection & Annotations?

### Ví dụ đời thường
> **Reflection** giống như **kính hiển vi** cho code - cho phép bạn "soi" vào bên trong bất kỳ class nào tại runtime: xem nó có những field gì, method gì, annotation gì... mà **không cần biết trước** class đó lúc viết code.
>
> **Annotation** giống như **nhãn dán** trên sản phẩm: `@NotNull` = "không được null", `@Cacheable` = "cache kết quả", `@Autowired` = "inject dependency". Chỉ là nhãn - cần Reflection để ĐỌC nhãn và HÀNH ĐỘNG.

### Ai dùng Reflection & Annotations?

```
┌─────────────────────────────────────────────────────────────────┐
│                AI DÙNG REFLECTION & ANNOTATIONS?                │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  🔧 Spring Framework:                                           │
│     @Autowired  → Reflection tìm field → inject dependency     │
│     @Controller → Reflection tìm class → đăng ký route         │
│     @Transactional → Reflection tạo proxy → quản lý transaction│
│                                                                 │
│  📦 Hibernate/JPA:                                              │
│     @Entity → Reflection map class → database table            │
│     @Column → Reflection map field → table column              │
│                                                                 │
│  ✅ Validation:                                                  │
│     @NotNull, @Size → Reflection check giá trị field           │
│                                                                 │
│  📊 Jackson (JSON):                                             │
│     @JsonProperty → Reflection serialize/deserialize JSON      │
│                                                                 │
│  🧪 JUnit:                                                      │
│     @Test → Reflection tìm method → chạy test                  │
│                                                                 │
│  💡 Bạn KHÔNG cần tự viết Reflection thường xuyên,             │
│     nhưng cần HIỂU để debug framework và viết library          │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 1. Reflection Basics (Cơ Bản Về Phản Chiếu)

### Lấy Class object - 3 cách

```java
// Mọi class trong Java đều có 1 object Class<?> đại diện cho nó
// Class object chứa TẤT CẢ thông tin về class: fields, methods, constructors...

// === Cách 1: Từ tên class (compile-time) ===
Class<?> clazz1 = String.class;              // Biết class lúc viết code

// === Cách 2: Từ object instance ===
String str = "Hello";
Class<?> clazz2 = str.getClass();            // Lấy class từ object đang có

// === Cách 3: Từ tên String (runtime - KHÔNG cần biết class lúc compile!) ===
Class<?> clazz3 = Class.forName("java.lang.String");
// ⚡ Cách này mạnh nhất: có thể load class từ config file, database...
// Đây là cách Spring, Hibernate load class!
```

### Lấy thông tin class

```java
Class<?> clazz = String.class;

clazz.getName();           // "java.lang.String" - tên đầy đủ (fully qualified)
clazz.getSimpleName();     // "String" - tên ngắn
clazz.getPackageName();    // "java.lang" - tên package
clazz.getSuperclass();     // Object.class - class cha
clazz.getInterfaces();     // [Serializable, Comparable, CharSequence] - các interface
clazz.getModifiers();      // public, final... (dưới dạng int, dùng Modifier.isPublic() để check)

// Kiểm tra loại
clazz.isInterface();       // false (String không phải interface)
clazz.isEnum();            // false
clazz.isArray();           // false
clazz.isPrimitive();       // false (String là reference type)
```

---

## 2. Fields (Truy Cập Trường Dữ Liệu)

```java
public class Person {
    private String name;        // private field
    private int age;
    public String email;        // public field
}

Class<?> clazz = Person.class;

// === Lấy danh sách fields ===
Field[] allFields = clazz.getDeclaredFields();   // TẤT CẢ fields (kể cả private)
Field[] publicFields = clazz.getFields();        // Chỉ PUBLIC fields (bao gồm kế thừa)

// === Lấy 1 field cụ thể ===
Field nameField = clazz.getDeclaredField("name"); // Tìm field "name"

// === Đọc/ghi giá trị field (kể cả private!) ===
Person person = new Person("John", 25);

nameField.setAccessible(true);                   // "Bẻ khóa" private → cho phép truy cập
String name = (String) nameField.get(person);    // Đọc: "John"
nameField.set(person, "Jane");                   // Ghi: đổi thành "Jane"

// ⚠️ setAccessible(true) bỏ qua private → vi phạm encapsulation
// → Chỉ dùng trong framework, testing, hoặc khi thực sự cần thiết!
```

```
getDeclaredFields() vs getFields():

  class Animal { public String type; }
  class Dog extends Animal { private String name; public int age; }

  Dog.class.getDeclaredFields()  → [name, age]      (chỉ field TRONG class Dog)
  Dog.class.getFields()          → [age, type]       (chỉ PUBLIC, bao gồm kế thừa)

  💡 getDeclared* = "của riêng class này" (private + public)
     get*          = "public + kế thừa"
```

---

## 3. Methods (Gọi Method Động)

```java
Class<?> clazz = Person.class;

// === Lấy danh sách methods ===
Method[] allMethods = clazz.getDeclaredMethods();   // Tất cả methods của class
Method[] publicMethods = clazz.getMethods();         // Public methods (bao gồm kế thừa)

// === Lấy 1 method cụ thể ===
Method getter = clazz.getMethod("getName");                    // getName() - không có tham số
Method setter = clazz.getMethod("setName", String.class);     // setName(String) - có 1 tham số String

// === Gọi method (invoke) ===
Person person = new Person("John", 25);

String name = (String) getter.invoke(person);                  // Gọi person.getName() → "John"
setter.invoke(person, "Jane");                                 // Gọi person.setName("Jane")

// === Gọi private method ===
Method privateMethod = clazz.getDeclaredMethod("secretMethod");
privateMethod.setAccessible(true);                             // "Bẻ khóa" private
privateMethod.invoke(person);                                  // Gọi method private

// === Lấy thông tin method ===
getter.getName();                    // "getName"
getter.getReturnType();              // String.class
getter.getParameterTypes();          // [] (không có tham số)
setter.getParameterTypes();          // [String.class]
getter.getModifiers();               // public
```

---

## 4. Constructors (Tạo Object Động)

```java
Class<?> clazz = Person.class;

// === Lấy constructors ===
Constructor<?>[] constructors = clazz.getConstructors();       // Public constructors

// === Lấy constructor cụ thể ===
Constructor<?> fullCtor = clazz.getConstructor(String.class, int.class);  // Person(String, int)
Constructor<?> noCtor = clazz.getDeclaredConstructor();                    // Person() - no-arg

// === Tạo object ===
Person person1 = (Person) fullCtor.newInstance("John", 25);    // new Person("John", 25)
Person person2 = (Person) noCtor.newInstance();                 // new Person()

// 💡 Đây là cách Spring tạo Bean: đọc config → Class.forName() → newInstance()
```

---

## 5. Annotations (Chú Thích)

### Annotations có sẵn trong Java

```java
@Override                    // Đánh dấu method ghi đè từ class cha
@Deprecated                  // Đánh dấu code không nên dùng nữa
@SuppressWarnings("unchecked") // Tắt warning cụ thể
@FunctionalInterface         // Đánh dấu interface chỉ có 1 abstract method
```

### Tạo Custom Annotation

```java
import java.lang.annotation.*;

// === @Retention: Annotation tồn tại đến khi nào? ===
// RetentionPolicy.SOURCE  → Biến mất sau compile (chỉ cho compiler)
// RetentionPolicy.CLASS   → Tồn tại trong .class file (mặc định)
// RetentionPolicy.RUNTIME → Tồn tại tại runtime → Reflection ĐỌC ĐƯỢC

// === @Target: Annotation dùng ở đâu? ===
// ElementType.FIELD       → Trên field (thuộc tính)
// ElementType.METHOD      → Trên method
// ElementType.TYPE         → Trên class/interface
// ElementType.PARAMETER   → Trên tham số method
// ElementType.CONSTRUCTOR → Trên constructor

// === Ví dụ 1: @NotNull - kiểm tra field không được null ===
@Retention(RetentionPolicy.RUNTIME)    // Tồn tại tại runtime → Reflection đọc được
@Target(ElementType.FIELD)             // Chỉ dùng trên field
public @interface NotNull {
    String message() default "Trường này không được null";  // Thuộc tính với giá trị mặc định
}

// === Ví dụ 2: @Cacheable - đánh dấu method cần cache ===
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.METHOD)
public @interface Cacheable {
    int ttlSeconds() default 300;      // Time-to-live mặc định 5 phút
}

// === Ví dụ 3: @Size - giới hạn độ dài ===
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.FIELD)
public @interface Size {
    int min() default 0;
    int max() default Integer.MAX_VALUE;
    String message() default "Độ dài không hợp lệ";
}
```

### Sử dụng Custom Annotation

```java
public class User {
    @NotNull                                  // Field name không được null
    @Size(min = 2, max = 50, message = "Tên phải từ 2-50 ký tự")
    private String name;

    @NotNull
    private String email;

    @Cacheable(ttlSeconds = 60)              // Cache kết quả 60 giây
    public List<Order> getOrders() {
        return orderRepository.findByUser(this);
    }
}

// ⚠️ Annotation chỉ là NHÃN DÁN - tự nó KHÔNG làm gì cả!
// Cần viết code (dùng Reflection) để ĐỌC annotation và THỰC HIỆN hành động
```

---

## 6. Đọc Annotations Bằng Reflection

```java
Class<?> clazz = User.class;

// === Kiểm tra annotation trên class ===
if (clazz.isAnnotationPresent(Entity.class)) {
    Entity entity = clazz.getAnnotation(Entity.class);
    System.out.println("Entity name: " + entity.name());
}

// === Đọc annotation trên fields ===
for (Field field : clazz.getDeclaredFields()) {
    if (field.isAnnotationPresent(NotNull.class)) {
        NotNull notNull = field.getAnnotation(NotNull.class);
        System.out.println(field.getName() + " - message: " + notNull.message());
    }

    if (field.isAnnotationPresent(Size.class)) {
        Size size = field.getAnnotation(Size.class);
        System.out.println(field.getName() + " - min: " + size.min() + ", max: " + size.max());
    }
}

// === Đọc annotation trên methods ===
for (Method method : clazz.getDeclaredMethods()) {
    Cacheable cacheable = method.getAnnotation(Cacheable.class);
    if (cacheable != null) {
        System.out.println(method.getName() + " cached " + cacheable.ttlSeconds() + "s");
    }
}
```

---

## 7. Ví Dụ Thực Tế: Simple Validator

> Kết hợp Reflection + Annotation để tạo validator tự động - giống Bean Validation trong Spring!

```java
public class SimpleValidator {

    // Validate bất kỳ object nào có annotation @NotNull, @Size
    public static List<String> validate(Object obj) throws IllegalAccessException {
        List<String> errors = new ArrayList<>();
        Class<?> clazz = obj.getClass();

        for (Field field : clazz.getDeclaredFields()) {
            field.setAccessible(true);                   // Truy cập private field
            Object value = field.get(obj);               // Lấy giá trị field

            // === Kiểm tra @NotNull ===
            if (field.isAnnotationPresent(NotNull.class)) {
                if (value == null) {
                    NotNull ann = field.getAnnotation(NotNull.class);
                    errors.add(field.getName() + ": " + ann.message());
                }
            }

            // === Kiểm tra @Size ===
            if (field.isAnnotationPresent(Size.class)) {
                Size size = field.getAnnotation(Size.class);
                if (value instanceof String str) {
                    if (str.length() < size.min() || str.length() > size.max()) {
                        errors.add(field.getName() + ": " + size.message()
                            + " (min=" + size.min() + ", max=" + size.max() + ")");
                    }
                }
            }
        }
        return errors;
    }
}

// === Sử dụng ===
User user = new User();
user.setName("A");           // Quá ngắn (min=2)
user.setEmail(null);          // Null!

List<String> errors = SimpleValidator.validate(user);
// errors:
// ["name: Tên phải từ 2-50 ký tự (min=2, max=50)",
//  "email: Trường này không được null"]

// 💡 Đây chính là cách Spring @Valid + Bean Validation hoạt động bên trong!
```

---

## 8. Sai Lầm Thường Gặp

### ❌ Sai lầm 1: Dùng Reflection khi không cần thiết

```java
// ❌ SAI: Dùng Reflection để gọi method thường
Method m = person.getClass().getMethod("getName");
String name = (String) m.invoke(person);

// ✅ ĐÚNG: Gọi trực tiếp nhanh hơn 10-100 lần!
String name = person.getName();

// 💡 Chỉ dùng Reflection khi:
// → Không biết class/method lúc compile (framework, plugin system)
// → Cần inspect code tại runtime (testing, serialization)
// → Viết library/framework dùng chung cho nhiều class
```

### ❌ Sai lầm 2: Quên RetentionPolicy.RUNTIME

```java
// ❌ SAI: Mặc định là CLASS → Reflection KHÔNG đọc được!
@Target(ElementType.FIELD)
public @interface MyAnnotation { }

// ✅ ĐÚNG: Phải là RUNTIME nếu muốn đọc bằng Reflection
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.FIELD)
public @interface MyAnnotation { }
```

### ❌ Sai lầm 3: Bỏ qua exception handling

```java
// ❌ SAI: Reflection method throw nhiều checked exceptions
Field f = clazz.getDeclaredField("name");    // NoSuchFieldException
f.get(obj);                                   // IllegalAccessException
Method m = clazz.getMethod("foo");           // NoSuchMethodException
m.invoke(obj);                                // InvocationTargetException

// ✅ ĐÚNG: Phải handle hoặc wrap exceptions
try {
    Field f = clazz.getDeclaredField("name");
    f.setAccessible(true);
    return f.get(obj);
} catch (NoSuchFieldException e) {
    throw new RuntimeException("Field 'name' không tồn tại trong " + clazz.getSimpleName(), e);
} catch (IllegalAccessException e) {
    throw new RuntimeException("Không thể truy cập field 'name'", e);
}
```

---

## 9. Tóm Tắt Cuối Ngày

| Khái niệm | Giải thích | Ví dụ |
|------------|-----------|-------|
| **Reflection** | "Kính hiển vi" xem bên trong class tại runtime | `Class.forName("...")` |
| **Class<?>** | Object đại diện cho 1 class | `String.class` |
| **Field** | Đại diện cho 1 trường dữ liệu | `clazz.getDeclaredField("name")` |
| **Method** | Đại diện cho 1 method | `clazz.getMethod("getName")` |
| **Constructor** | Đại diện cho 1 constructor | `clazz.getConstructor(String.class)` |
| **invoke()** | Gọi method động | `method.invoke(object, args)` |
| **setAccessible(true)** | "Bẻ khóa" private | Truy cập private field/method |
| **@Retention** | Annotation tồn tại đến khi nào | `RUNTIME` cho Reflection |
| **@Target** | Annotation dùng ở đâu | `FIELD`, `METHOD`, `TYPE` |
| **Custom Annotation** | Tạo "nhãn dán" riêng | `@interface NotNull` |
| **isAnnotationPresent()** | Kiểm tra có annotation không | `field.isAnnotationPresent(...)` |
| **getAnnotation()** | Lấy annotation để đọc giá trị | `field.getAnnotation(Size.class)` |

---

## 10. Câu Hỏi Phỏng Vấn Thường Gặp

### 🔥 Câu 1: Reflection là gì? Tại sao cần?
**Trả lời:**
Reflection là khả năng inspect và manipulate class, field, method, constructor tại **runtime** mà không cần biết trước lúc compile. Cần dùng trong: framework (Spring DI, Hibernate ORM), serialization (Jackson JSON), testing (JUnit), plugin systems. Trade-off: chậm hơn gọi trực tiếp (10-100x), phá vỡ encapsulation, khó debug.

### 🔥 Câu 2: getDeclaredMethods() khác getMethods() thế nào?
**Trả lời:**
- `getDeclaredMethods()`: Trả về TẤT CẢ methods **khai báo trong class đó** (public + private + protected), KHÔNG bao gồm methods kế thừa
- `getMethods()`: Trả về CHỈ **public methods** của class VÀ tất cả public methods kế thừa từ superclass + interfaces
- Tương tự cho Fields: `getDeclaredFields()` vs `getFields()`

### 🔥 Câu 3: Annotation là gì? Nó tự có tác dụng không?
**Trả lời:**
Annotation là metadata (siêu dữ liệu) gắn vào code (class, method, field...). Annotation tự nó **KHÔNG có tác dụng** - nó chỉ là "nhãn dán". Cần có **processor** (thường dùng Reflection) để đọc annotation và thực hiện hành động. Ví dụ: `@Autowired` tự nó không inject gì, Spring container đọc annotation đó bằng Reflection rồi mới inject dependency.

### 🔥 Câu 4: 3 RetentionPolicy khác nhau thế nào?
**Trả lời:**
- `SOURCE`: Chỉ tồn tại trong source code, biến mất sau compile. Ví dụ: `@Override`, `@SuppressWarnings` - compiler check rồi bỏ
- `CLASS` (mặc định): Tồn tại trong .class file nhưng KHÔNG available tại runtime. Dùng cho bytecode tools
- `RUNTIME`: Tồn tại tại runtime → **Reflection đọc được**. Phải dùng RUNTIME nếu muốn đọc annotation bằng code. Ví dụ: `@Autowired`, `@Entity`

### 🔥 Câu 5: Reflection có nhược điểm gì?
**Trả lời:**
1. **Hiệu suất**: Chậm hơn gọi trực tiếp 10-100 lần (do type checking, security check)
2. **Type safety**: Mất type safety lúc compile → lỗi chỉ phát hiện tại runtime
3. **Encapsulation**: `setAccessible(true)` phá vỡ private → vi phạm nguyên tắc OOP
4. **Bảo trì**: Code khó đọc, khó debug, IDE không hỗ trợ refactor
5. **Security**: Module system (Java 9+) giới hạn Reflection access

### 🔥 Câu 6: Làm sao Spring @Autowired hoạt động?
**Trả lời:**
Spring container khi khởi động: (1) Scan tất cả class có `@Component`/`@Service`... bằng classpath scanning, (2) Tạo instance bằng `Constructor.newInstance()`, (3) Duyệt tất cả fields bằng `getDeclaredFields()`, (4) Tìm field có `@Autowired` bằng `isAnnotationPresent()`, (5) `setAccessible(true)` để truy cập private field, (6) `field.set(bean, dependency)` để inject dependency. Tất cả dựa trên Reflection!

---

## Navigation

- [← Day 15: CompletableFuture](./day-15-completable-future.md)
- [Day 17: Design Patterns →](./day-17-design-patterns.md)
