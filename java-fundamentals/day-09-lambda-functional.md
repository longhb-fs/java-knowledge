# Day 9: Lambda & Functional Programming

## Mục tiêu
- Lambda expressions
- Functional interfaces
- Method references
- Built-in functional interfaces

---

## 1. Lambda Expressions

### 1.1. Syntax

```java
// Cú pháp cơ bản
(parameters) -> expression
(parameters) -> { statements; }

// Ví dụ
() -> System.out.println("Hello")           // No params
x -> x * 2                                   // One param
(x, y) -> x + y                              // Multiple params
(String s) -> s.length()                     // With type
(x, y) -> { return x + y; }                  // Block body
```

### 1.2. So sánh với Anonymous Class

```java
// Anonymous class
Comparator<String> comp1 = new Comparator<String>() {
    @Override
    public int compare(String s1, String s2) {
        return s1.compareTo(s2);
    }
};

// Lambda expression
Comparator<String> comp2 = (s1, s2) -> s1.compareTo(s2);

// Runnable
Runnable r1 = new Runnable() {
    @Override
    public void run() {
        System.out.println("Running");
    }
};

Runnable r2 = () -> System.out.println("Running");
```

### 1.3. Effectively Final

```java
int multiplier = 5;  // Effectively final

// Lambda có thể access outer variables
Function<Integer, Integer> func = x -> x * multiplier;

// ❌ Error nếu modify
// multiplier = 10;  // Cannot modify
```

---

## 2. Functional Interfaces

### 2.1. Definition

```java
// Functional interface = interface có duy nhất 1 abstract method
@FunctionalInterface
public interface Calculator {
    int calculate(int a, int b);

    // Có thể có default methods
    default void printInfo() {
        System.out.println("Calculator interface");
    }

    // Có thể có static methods
    static Calculator createAdder() {
        return (a, b) -> a + b;
    }
}

// Sử dụng
Calculator add = (a, b) -> a + b;
Calculator subtract = (a, b) -> a - b;
Calculator multiply = (a, b) -> a * b;

System.out.println(add.calculate(5, 3));       // 8
System.out.println(subtract.calculate(5, 3));  // 2
```

### 2.2. Common Functional Interfaces

| Interface | Method | Description |
|-----------|--------|-------------|
| `Function<T,R>` | `R apply(T t)` | T → R |
| `Consumer<T>` | `void accept(T t)` | T → void |
| `Supplier<T>` | `T get()` | () → T |
| `Predicate<T>` | `boolean test(T t)` | T → boolean |
| `BiFunction<T,U,R>` | `R apply(T t, U u)` | (T,U) → R |
| `BiConsumer<T,U>` | `void accept(T t, U u)` | (T,U) → void |
| `BiPredicate<T,U>` | `boolean test(T t, U u)` | (T,U) → boolean |
| `UnaryOperator<T>` | `T apply(T t)` | T → T |
| `BinaryOperator<T>` | `T apply(T t1, T t2)` | (T,T) → T |

---

## 3. Function Interface

### 3.1. Basic Usage

```java
import java.util.function.Function;

// String → Integer
Function<String, Integer> strLength = s -> s.length();
System.out.println(strLength.apply("Hello"));  // 5

// Integer → String
Function<Integer, String> intToStr = i -> "Number: " + i;
System.out.println(intToStr.apply(42));  // "Number: 42"
```

### 3.2. Chaining Functions

```java
Function<String, String> toUpper = s -> s.toUpperCase();
Function<String, String> addPrefix = s -> ">>> " + s;

// andThen: toUpper THEN addPrefix
Function<String, String> combined1 = toUpper.andThen(addPrefix);
System.out.println(combined1.apply("hello"));  // ">>> HELLO"

// compose: addPrefix THEN toUpper
Function<String, String> combined2 = toUpper.compose(addPrefix);
System.out.println(combined2.apply("hello"));  // ">>> HELLO"

// identity
Function<String, String> identity = Function.identity();
System.out.println(identity.apply("test"));  // "test"
```

---

## 4. Consumer Interface

```java
import java.util.function.Consumer;

// Print
Consumer<String> printer = s -> System.out.println(s);
printer.accept("Hello");  // Hello

// Modify object
Consumer<List<String>> addItem = list -> list.add("New Item");
List<String> myList = new ArrayList<>();
addItem.accept(myList);

// andThen
Consumer<String> print = s -> System.out.print(s);
Consumer<String> printUpperCase = s -> System.out.print(s.toUpperCase());

Consumer<String> combined = print.andThen(s -> System.out.print(" - "))
                                 .andThen(printUpperCase);
combined.accept("hello");  // hello - HELLO

// forEach với Consumer
List<String> names = Arrays.asList("John", "Jane", "Bob");
names.forEach(name -> System.out.println("Hello, " + name));
```

---

## 5. Supplier Interface

```java
import java.util.function.Supplier;

// Basic
Supplier<String> greeting = () -> "Hello World";
System.out.println(greeting.get());  // Hello World

// Random number
Supplier<Double> randomValue = () -> Math.random();
System.out.println(randomValue.get());

// Lazy initialization
Supplier<Connection> connectionSupplier = () -> {
    System.out.println("Creating connection...");
    return createDatabaseConnection();
};
// Connection chỉ được tạo khi gọi get()
Connection conn = connectionSupplier.get();

// Factory pattern
Supplier<List<String>> listFactory = ArrayList::new;
List<String> list1 = listFactory.get();
List<String> list2 = listFactory.get();  // New instance
```

---

## 6. Predicate Interface

```java
import java.util.function.Predicate;

// Basic
Predicate<Integer> isPositive = n -> n > 0;
System.out.println(isPositive.test(5));   // true
System.out.println(isPositive.test(-5));  // false

// Chaining
Predicate<Integer> isEven = n -> n % 2 == 0;
Predicate<Integer> isPositiveEven = isPositive.and(isEven);
System.out.println(isPositiveEven.test(4));   // true
System.out.println(isPositiveEven.test(-4));  // false

// or
Predicate<String> isEmpty = s -> s.isEmpty();
Predicate<String> isNull = s -> s == null;
Predicate<String> isNullOrEmpty = isNull.or(isEmpty);

// negate
Predicate<String> isNotEmpty = isEmpty.negate();

// isEqual
Predicate<String> equalsHello = Predicate.isEqual("Hello");
System.out.println(equalsHello.test("Hello"));  // true

// Filter với Predicate
List<Integer> numbers = Arrays.asList(1, 2, 3, 4, 5, 6, 7, 8, 9, 10);
numbers.stream()
    .filter(isPositive.and(isEven))
    .forEach(System.out::println);  // 2 4 6 8 10
```

---

## 7. BiFunction và Variations

```java
import java.util.function.*;

// BiFunction<T, U, R>
BiFunction<Integer, Integer, Integer> add = (a, b) -> a + b;
System.out.println(add.apply(5, 3));  // 8

BiFunction<String, Integer, String> repeat = (s, n) -> s.repeat(n);
System.out.println(repeat.apply("Ha", 3));  // HaHaHa

// BiConsumer<T, U>
BiConsumer<String, Integer> printPair = (k, v) ->
    System.out.println(k + " = " + v);
printPair.accept("Age", 25);  // Age = 25

// Map.forEach dùng BiConsumer
Map<String, Integer> map = Map.of("A", 1, "B", 2);
map.forEach((k, v) -> System.out.println(k + ": " + v));

// BiPredicate<T, U>
BiPredicate<String, Integer> lengthCheck = (s, n) -> s.length() > n;
System.out.println(lengthCheck.test("Hello", 3));  // true

// BinaryOperator<T> (BiFunction<T,T,T>)
BinaryOperator<Integer> sum = (a, b) -> a + b;
BinaryOperator<Integer> max = Integer::max;

// UnaryOperator<T> (Function<T,T>)
UnaryOperator<String> toUpper = String::toUpperCase;
System.out.println(toUpper.apply("hello"));  // HELLO
```

---

## 8. Method References

### 8.1. Types of Method References

```java
// 1. Static method reference
// ClassName::staticMethod
Function<String, Integer> parseInt = Integer::parseInt;
System.out.println(parseInt.apply("123"));  // 123

// 2. Instance method of particular object
// object::instanceMethod
String str = "Hello";
Supplier<Integer> length = str::length;
System.out.println(length.get());  // 5

// 3. Instance method of arbitrary object
// ClassName::instanceMethod
Function<String, String> toUpper = String::toUpperCase;
System.out.println(toUpper.apply("hello"));  // HELLO

// 4. Constructor reference
// ClassName::new
Supplier<ArrayList<String>> listSupplier = ArrayList::new;
Function<Integer, int[]> arrayCreator = int[]::new;
```

### 8.2. Examples

```java
List<String> names = Arrays.asList("John", "Jane", "Bob");

// Lambda
names.forEach(name -> System.out.println(name));
// Method reference
names.forEach(System.out::println);

// Lambda
names.sort((s1, s2) -> s1.compareToIgnoreCase(s2));
// Method reference
names.sort(String::compareToIgnoreCase);

// Lambda
names.stream().map(s -> s.toUpperCase());
// Method reference
names.stream().map(String::toUpperCase);

// Constructor reference
Supplier<List<String>> listFactory = ArrayList::new;
Function<String, Person> personFactory = Person::new;
```

---

## 9. Practical Examples

### 9.1. Filtering and Transforming

```java
List<Person> people = Arrays.asList(
    new Person("John", 25),
    new Person("Jane", 30),
    new Person("Bob", 20),
    new Person("Alice", 28)
);

// Filter adults (age >= 21)
Predicate<Person> isAdult = p -> p.getAge() >= 21;

// Get names
Function<Person, String> getName = Person::getName;

// To uppercase
Function<String, String> toUpper = String::toUpperCase;

// Combined
List<String> adultNames = people.stream()
    .filter(isAdult)
    .map(getName)
    .map(toUpper)
    .collect(Collectors.toList());

System.out.println(adultNames);  // [JOHN, JANE, ALICE]
```

### 9.2. Strategy Pattern với Lambda

```java
public class PaymentProcessor {
    public void process(double amount, Consumer<Double> paymentStrategy) {
        System.out.println("Processing payment of $" + amount);
        paymentStrategy.accept(amount);
    }
}

// Strategies as lambdas
Consumer<Double> creditCard = amount ->
    System.out.println("Paid $" + amount + " with Credit Card");

Consumer<Double> paypal = amount ->
    System.out.println("Paid $" + amount + " with PayPal");

Consumer<Double> crypto = amount ->
    System.out.println("Paid $" + amount + " with Crypto");

// Usage
PaymentProcessor processor = new PaymentProcessor();
processor.process(100, creditCard);
processor.process(50, paypal);
processor.process(200, crypto);
```

### 9.3. Builder với Functions

```java
public class PersonBuilder {
    private String name;
    private int age;
    private String email;

    public PersonBuilder with(Consumer<PersonBuilder> builderFunction) {
        builderFunction.accept(this);
        return this;
    }

    public PersonBuilder name(String name) {
        this.name = name;
        return this;
    }

    public PersonBuilder age(int age) {
        this.age = age;
        return this;
    }

    public PersonBuilder email(String email) {
        this.email = email;
        return this;
    }

    public Person build() {
        return new Person(name, age, email);
    }
}

// Usage
Person person = new PersonBuilder()
    .with(b -> {
        b.name("John");
        b.age(25);
        b.email("john@email.com");
    })
    .build();
```

---

## 10. Bài tập thực hành

### Bài 1: Custom Functional Interface
Tạo functional interface `Transformer<T>` với method `transform(T input)` và các default methods:
- `andThen(Transformer<T> other)`
- `compose(Transformer<T> other)`

---

### Bài 2: Validation Framework
Tạo validation framework sử dụng Predicate:

```java
Validator<String> emailValidator = Validator.<String>of()
    .addRule(s -> s != null, "Email cannot be null")
    .addRule(s -> s.contains("@"), "Email must contain @")
    .addRule(s -> s.length() > 5, "Email too short");

ValidationResult result = emailValidator.validate("test@email.com");
```

---

### Bài 3: Function Composition
Tạo pipeline xử lý data:

```java
Pipeline<String, Integer> pipeline = Pipeline
    .<String>start()
    .then(String::trim)
    .then(String::toLowerCase)
    .then(String::length);

int result = pipeline.execute("  Hello World  ");  // 11
```

---

### Bài 4: Event System
Tạo event system với Consumer:

```java
EventBus bus = new EventBus();
bus.subscribe("userCreated", event -> System.out.println("User created: " + event));
bus.subscribe("userCreated", event -> sendEmail(event));
bus.publish("userCreated", new UserCreatedEvent("John"));
```

---

## 11. Đáp án tham khảo

<details>
<summary>Bài 2: Validation Framework</summary>

```java
public class Validator<T> {
    private List<Pair<Predicate<T>, String>> rules = new ArrayList<>();

    public static <T> Validator<T> of() {
        return new Validator<>();
    }

    public Validator<T> addRule(Predicate<T> rule, String errorMessage) {
        rules.add(new Pair<>(rule, errorMessage));
        return this;
    }

    public ValidationResult validate(T value) {
        List<String> errors = new ArrayList<>();

        for (Pair<Predicate<T>, String> rule : rules) {
            if (!rule.getKey().test(value)) {
                errors.add(rule.getValue());
            }
        }

        return new ValidationResult(errors.isEmpty(), errors);
    }
}

public class ValidationResult {
    private boolean valid;
    private List<String> errors;

    public ValidationResult(boolean valid, List<String> errors) {
        this.valid = valid;
        this.errors = errors;
    }

    public boolean isValid() { return valid; }
    public List<String> getErrors() { return errors; }
}

// Usage
Validator<String> emailValidator = Validator.<String>of()
    .addRule(s -> s != null, "Email cannot be null")
    .addRule(s -> s.contains("@"), "Email must contain @")
    .addRule(s -> s.length() > 5, "Email too short");

ValidationResult result1 = emailValidator.validate("test@email.com");
System.out.println(result1.isValid());  // true

ValidationResult result2 = emailValidator.validate("test");
System.out.println(result2.isValid());  // false
System.out.println(result2.getErrors());  // [Email must contain @, Email too short]
```
</details>

<details>
<summary>Bài 4: Event System</summary>

```java
public class EventBus {
    private Map<String, List<Consumer<Object>>> subscribers = new HashMap<>();

    public void subscribe(String eventType, Consumer<Object> handler) {
        subscribers.computeIfAbsent(eventType, k -> new ArrayList<>())
                   .add(handler);
    }

    public void unsubscribe(String eventType, Consumer<Object> handler) {
        List<Consumer<Object>> handlers = subscribers.get(eventType);
        if (handlers != null) {
            handlers.remove(handler);
        }
    }

    public void publish(String eventType, Object event) {
        List<Consumer<Object>> handlers = subscribers.get(eventType);
        if (handlers != null) {
            handlers.forEach(handler -> handler.accept(event));
        }
    }
}

// Usage
EventBus bus = new EventBus();

bus.subscribe("userCreated", event -> {
    System.out.println("Handler 1: User created - " + event);
});

bus.subscribe("userCreated", event -> {
    System.out.println("Handler 2: Sending welcome email to " + event);
});

bus.subscribe("orderPlaced", event -> {
    System.out.println("Order placed: " + event);
});

bus.publish("userCreated", "John");
bus.publish("orderPlaced", "Order #123");
```
</details>

---

## Navigation

- [← Day 8: Generics](./day-08-generics.md)
- [Day 10: Stream API →](./day-10-stream-api.md)
