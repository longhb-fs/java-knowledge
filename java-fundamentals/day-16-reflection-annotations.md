# Day 16: Reflection & Annotations

## Mục tiêu
- Java Reflection API
- Custom annotations
- Annotation processing

---

## 1. Reflection Basics

```java
// Get Class object
Class<?> clazz = String.class;
Class<?> clazz2 = "Hello".getClass();
Class<?> clazz3 = Class.forName("java.lang.String");

// Class info
clazz.getName();           // "java.lang.String"
clazz.getSimpleName();     // "String"
clazz.getPackageName();    // "java.lang"
clazz.getSuperclass();     // Object.class
clazz.getInterfaces();     // Serializable, Comparable, CharSequence
clazz.getModifiers();      // public, final, etc.
```

---

## 2. Fields

```java
Class<?> clazz = Person.class;

// All fields (including private)
Field[] allFields = clazz.getDeclaredFields();

// Public fields only
Field[] publicFields = clazz.getFields();

// Specific field
Field nameField = clazz.getDeclaredField("name");

// Access private field
nameField.setAccessible(true);
Person person = new Person("John", 25);
String name = (String) nameField.get(person);
nameField.set(person, "Jane");
```

---

## 3. Methods

```java
Class<?> clazz = Person.class;

// All methods
Method[] methods = clazz.getDeclaredMethods();

// Specific method
Method getter = clazz.getMethod("getName");
Method setter = clazz.getMethod("setName", String.class);

// Invoke
Person person = new Person("John", 25);
String name = (String) getter.invoke(person);
setter.invoke(person, "Jane");
```

---

## 4. Constructors

```java
Class<?> clazz = Person.class;

// Get constructors
Constructor<?>[] constructors = clazz.getConstructors();

// Specific constructor
Constructor<?> ctor = clazz.getConstructor(String.class, int.class);

// Create instance
Person person = (Person) ctor.newInstance("John", 25);

// No-arg constructor
Person person2 = clazz.getDeclaredConstructor().newInstance();
```

---

## 5. Creating Annotations

```java
// Define annotation
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.FIELD)
public @interface NotNull {
    String message() default "Field cannot be null";
}

@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.METHOD)
public @interface Cacheable {
    int ttlSeconds() default 300;
}

// Use annotation
public class User {
    @NotNull
    private String name;

    @Cacheable(ttlSeconds = 60)
    public List<Order> getOrders() { ... }
}
```

---

## 6. Reading Annotations

```java
Class<?> clazz = User.class;

// Check if annotation present
if (clazz.isAnnotationPresent(Entity.class)) {
    Entity entity = clazz.getAnnotation(Entity.class);
    System.out.println(entity.name());
}

// Read field annotations
for (Field field : clazz.getDeclaredFields()) {
    if (field.isAnnotationPresent(NotNull.class)) {
        NotNull notNull = field.getAnnotation(NotNull.class);
        System.out.println(field.getName() + ": " + notNull.message());
    }
}

// Read method annotations
for (Method method : clazz.getDeclaredMethods()) {
    Cacheable cacheable = method.getAnnotation(Cacheable.class);
    if (cacheable != null) {
        System.out.println(method.getName() + " cached for " + cacheable.ttlSeconds() + "s");
    }
}
```

---

## 7. Simple Validator

```java
public class Validator {
    public static List<String> validate(Object obj) throws Exception {
        List<String> errors = new ArrayList<>();
        Class<?> clazz = obj.getClass();

        for (Field field : clazz.getDeclaredFields()) {
            field.setAccessible(true);
            Object value = field.get(obj);

            // Check @NotNull
            if (field.isAnnotationPresent(NotNull.class)) {
                if (value == null) {
                    NotNull ann = field.getAnnotation(NotNull.class);
                    errors.add(field.getName() + ": " + ann.message());
                }
            }

            // Check @Size
            if (field.isAnnotationPresent(Size.class)) {
                Size size = field.getAnnotation(Size.class);
                if (value instanceof String) {
                    String str = (String) value;
                    if (str.length() < size.min() || str.length() > size.max()) {
                        errors.add(field.getName() + ": must be between " +
                            size.min() + " and " + size.max());
                    }
                }
            }
        }
        return errors;
    }
}
```

---

## 8. Bài tập thực hành

### Bài 1: Simple DI Container
Tạo dependency injection container với @Inject annotation.

### Bài 2: JSON Serializer
Serialize object thành JSON với @JsonProperty annotation.

### Bài 3: Bean Mapper
Map properties giữa 2 objects.

---

## Navigation

- [← Day 15: CompletableFuture](./day-15-completable-future.md)
- [Day 17: Design Patterns →](./day-17-design-patterns.md)
