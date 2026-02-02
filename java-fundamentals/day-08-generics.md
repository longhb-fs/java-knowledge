# Day 8: Generics

## Mục tiêu
- Hiểu Generic types
- Generic classes và methods
- Bounded type parameters
- Wildcards
- Type erasure

---

## 1. Tại sao cần Generics?

### 1.1. Vấn đề không có Generics

```java
// Trước Java 5 - không có generics
List list = new ArrayList();
list.add("Hello");
list.add(123);  // Có thể add bất kỳ type

String s = (String) list.get(0);  // Phải cast
String s2 = (String) list.get(1);  // ClassCastException tại runtime!
```

### 1.2. Với Generics

```java
// Với generics - type-safe
List<String> list = new ArrayList<>();
list.add("Hello");
// list.add(123);  // Compile error!

String s = list.get(0);  // Không cần cast
```

---

## 2. Generic Classes

### 2.1. Basic Generic Class

```java
public class Box<T> {
    private T content;

    public void set(T content) {
        this.content = content;
    }

    public T get() {
        return content;
    }
}

// Sử dụng
Box<String> stringBox = new Box<>();
stringBox.set("Hello");
String s = stringBox.get();

Box<Integer> intBox = new Box<>();
intBox.set(123);
Integer i = intBox.get();
```

### 2.2. Multiple Type Parameters

```java
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

// Sử dụng
Pair<String, Integer> pair = new Pair<>("Age", 25);
System.out.println(pair.getKey());    // "Age"
System.out.println(pair.getValue());  // 25

Pair<Integer, String> reversed = new Pair<>(1, "One");
```

### 2.3. Generic Class với Inheritance

```java
public class NumberBox<T extends Number> {
    private T number;

    public NumberBox(T number) {
        this.number = number;
    }

    public double getDoubleValue() {
        return number.doubleValue();
    }
}

// Sử dụng
NumberBox<Integer> intBox = new NumberBox<>(10);
NumberBox<Double> doubleBox = new NumberBox<>(3.14);
// NumberBox<String> stringBox = new NumberBox<>("Hi");  // Error!
```

---

## 3. Generic Methods

### 3.1. Basic Generic Method

```java
public class Utils {

    // Generic method - type parameter trước return type
    public static <T> void printArray(T[] array) {
        for (T element : array) {
            System.out.print(element + " ");
        }
        System.out.println();
    }

    public static <T> T getFirst(List<T> list) {
        if (list.isEmpty()) {
            return null;
        }
        return list.get(0);
    }
}

// Sử dụng
String[] strings = {"A", "B", "C"};
Integer[] integers = {1, 2, 3};

Utils.printArray(strings);   // A B C
Utils.printArray(integers);  // 1 2 3

List<String> list = Arrays.asList("X", "Y", "Z");
String first = Utils.getFirst(list);  // "X"
```

### 3.2. Multiple Type Parameters

```java
public class Utils {

    public static <K, V> void printPair(K key, V value) {
        System.out.println(key + " = " + value);
    }

    public static <T, U> Pair<T, U> makePair(T first, U second) {
        return new Pair<>(first, second);
    }
}

// Sử dụng
Utils.printPair("Name", "John");
Utils.printPair(1, true);

Pair<String, Integer> pair = Utils.makePair("Age", 25);
```

### 3.3. Generic Method trong Generic Class

```java
public class Container<T> {
    private T item;

    public void setItem(T item) {
        this.item = item;
    }

    // Generic method với type parameter riêng
    public <U> void inspect(U data) {
        System.out.println("T: " + item.getClass().getName());
        System.out.println("U: " + data.getClass().getName());
    }
}

Container<Integer> container = new Container<>();
container.setItem(10);
container.inspect("Hello");  // U là String
```

---

## 4. Bounded Type Parameters

### 4.1. Upper Bounded (extends)

```java
// T phải là Number hoặc subclass của Number
public class MathBox<T extends Number> {
    private T value;

    public MathBox(T value) {
        this.value = value;
    }

    public double square() {
        return value.doubleValue() * value.doubleValue();
    }
}

MathBox<Integer> intBox = new MathBox<>(5);
System.out.println(intBox.square());  // 25.0

MathBox<Double> doubleBox = new MathBox<>(3.14);
System.out.println(doubleBox.square());  // 9.8596
```

### 4.2. Multiple Bounds

```java
// T phải extend Number VÀ implement Comparable
public class SortableBox<T extends Number & Comparable<T>> {
    private T value;

    public SortableBox(T value) {
        this.value = value;
    }

    public boolean isGreaterThan(T other) {
        return value.compareTo(other) > 0;
    }
}

// Class trước, interfaces sau
// <T extends SomeClass & Interface1 & Interface2>
```

### 4.3. Generic Method với Bounds

```java
public class Utils {

    // Tìm max trong list
    public static <T extends Comparable<T>> T findMax(List<T> list) {
        if (list.isEmpty()) {
            return null;
        }

        T max = list.get(0);
        for (T item : list) {
            if (item.compareTo(max) > 0) {
                max = item;
            }
        }
        return max;
    }

    // Tính tổng
    public static <T extends Number> double sum(List<T> list) {
        double sum = 0;
        for (T num : list) {
            sum += num.doubleValue();
        }
        return sum;
    }
}

List<Integer> numbers = Arrays.asList(3, 1, 4, 1, 5);
System.out.println(Utils.findMax(numbers));  // 5
System.out.println(Utils.sum(numbers));      // 14.0
```

---

## 5. Wildcards

### 5.1. Unbounded Wildcard (?)

```java
// Chấp nhận List của bất kỳ type nào
public static void printList(List<?> list) {
    for (Object item : list) {
        System.out.println(item);
    }
}

List<String> strings = Arrays.asList("A", "B", "C");
List<Integer> integers = Arrays.asList(1, 2, 3);

printList(strings);   // OK
printList(integers);  // OK
```

### 5.2. Upper Bounded Wildcard (? extends)

```java
// Chấp nhận List của Number hoặc subclass
public static double sumOfList(List<? extends Number> list) {
    double sum = 0;
    for (Number num : list) {
        sum += num.doubleValue();
    }
    return sum;
}

List<Integer> integers = Arrays.asList(1, 2, 3);
List<Double> doubles = Arrays.asList(1.1, 2.2, 3.3);

System.out.println(sumOfList(integers));  // 6.0
System.out.println(sumOfList(doubles));   // 6.6

// Chỉ có thể READ, không thể ADD
// list.add(1);  // Error! Không biết chính xác type
```

### 5.3. Lower Bounded Wildcard (? super)

```java
// Chấp nhận List của Integer hoặc superclass (Number, Object)
public static void addNumbers(List<? super Integer> list) {
    list.add(1);
    list.add(2);
    list.add(3);
}

List<Integer> intList = new ArrayList<>();
List<Number> numList = new ArrayList<>();
List<Object> objList = new ArrayList<>();

addNumbers(intList);  // OK
addNumbers(numList);  // OK
addNumbers(objList);  // OK

// Có thể ADD Integer, nhưng READ chỉ trả về Object
// Integer i = list.get(0);  // Error!
Object obj = list.get(0);    // OK
```

### 5.4. PECS: Producer Extends, Consumer Super

```java
// Copy từ source sang destination
public static <T> void copy(List<? extends T> source, List<? super T> dest) {
    for (T item : source) {
        dest.add(item);
    }
}

List<Integer> source = Arrays.asList(1, 2, 3);
List<Number> dest = new ArrayList<>();

copy(source, dest);  // Integer extends Number
System.out.println(dest);  // [1, 2, 3]

// Producer (source) - extends - đọc từ đây
// Consumer (dest) - super - ghi vào đây
```

---

## 6. Type Erasure

### 6.1. Concept

```java
// Compile time
List<String> list = new ArrayList<>();

// Runtime (after erasure)
List list = new ArrayList();

// Generics chỉ tồn tại tại compile time
// JVM không biết generic type tại runtime
```

### 6.2. Implications

```java
// Không thể làm những điều này:
public class MyClass<T> {
    // T obj = new T();           // Error! Không thể instantiate type parameter
    // T[] arr = new T[10];       // Error! Không thể tạo array
    // if (obj instanceof T)     // Error! Không thể instanceof

    private T item;

    // Workaround: truyền Class
    public T createInstance(Class<T> clazz) throws Exception {
        return clazz.getDeclaredConstructor().newInstance();
    }
}
```

### 6.3. Bridge Methods

```java
public class Node<T> {
    public T data;

    public void setData(T data) {
        this.data = data;
    }
}

public class StringNode extends Node<String> {
    @Override
    public void setData(String data) {  // Override
        super.setData(data);
    }
}

// Compiler tạo bridge method:
// public void setData(Object data) {
//     setData((String) data);
// }
```

---

## 7. Generic Interfaces

```java
// Generic interface
public interface Comparable<T> {
    int compareTo(T other);
}

public interface Repository<T, ID> {
    T findById(ID id);
    List<T> findAll();
    void save(T entity);
    void delete(ID id);
}

// Implementation
public class UserRepository implements Repository<User, Long> {
    @Override
    public User findById(Long id) { /* ... */ }

    @Override
    public List<User> findAll() { /* ... */ }

    @Override
    public void save(User entity) { /* ... */ }

    @Override
    public void delete(Long id) { /* ... */ }
}
```

---

## 8. Common Generic Patterns

### 8.1. Generic Singleton Factory

```java
public class SingletonFactory {
    private static Map<Class<?>, Object> instances = new HashMap<>();

    @SuppressWarnings("unchecked")
    public static <T> T getInstance(Class<T> clazz) {
        return (T) instances.computeIfAbsent(clazz, k -> {
            try {
                return k.getDeclaredConstructor().newInstance();
            } catch (Exception e) {
                throw new RuntimeException(e);
            }
        });
    }
}
```

### 8.2. Generic Builder

```java
public class Builder<T> {
    private Map<String, Object> properties = new HashMap<>();
    private Class<T> clazz;

    public Builder(Class<T> clazz) {
        this.clazz = clazz;
    }

    public Builder<T> with(String name, Object value) {
        properties.put(name, value);
        return this;
    }

    public T build() throws Exception {
        T instance = clazz.getDeclaredConstructor().newInstance();
        for (Map.Entry<String, Object> entry : properties.entrySet()) {
            Field field = clazz.getDeclaredField(entry.getKey());
            field.setAccessible(true);
            field.set(instance, entry.getValue());
        }
        return instance;
    }
}
```

### 8.3. Generic DAO

```java
public interface GenericDao<T, ID> {
    T findById(ID id);
    List<T> findAll();
    T save(T entity);
    void delete(T entity);
    void deleteById(ID id);
}

public abstract class AbstractDao<T, ID> implements GenericDao<T, ID> {
    protected Class<T> entityClass;

    @SuppressWarnings("unchecked")
    public AbstractDao() {
        ParameterizedType type = (ParameterizedType) getClass().getGenericSuperclass();
        this.entityClass = (Class<T>) type.getActualTypeArguments()[0];
    }

    // Common implementation...
}

public class UserDao extends AbstractDao<User, Long> {
    // User-specific methods...
}
```

---

## 9. Bài tập thực hành

### Bài 1: Generic Stack
Implement Stack với generics:
- push(T item)
- pop(): T
- peek(): T
- isEmpty(): boolean
- size(): int

---

### Bài 2: Generic Pair Utils
Tạo utility class cho Pair:
- swap(Pair<K,V>): Pair<V,K>
- createFromArray(T[]): List<Pair<Integer, T>>
- toMap(List<Pair<K,V>>): Map<K,V>

---

### Bài 3: Generic Filter
Tạo generic filter method:

```java
public static <T> List<T> filter(List<T> list, Predicate<T> predicate) {
    // Return items matching predicate
}

// Usage:
List<Integer> evens = filter(numbers, n -> n % 2 == 0);
List<String> longStrings = filter(strings, s -> s.length() > 5);
```

---

### Bài 4: Generic Cache
Implement generic cache với expiration:

```java
Cache<String, User> userCache = new Cache<>(60000);  // 60s TTL
userCache.put("user1", user);
User cached = userCache.get("user1");  // null if expired
```

---

### Bài 5: Generic Tree
Implement binary tree với generics:

```java
BinaryTree<Integer> tree = new BinaryTree<>();
tree.insert(5);
tree.insert(3);
tree.insert(7);
boolean found = tree.contains(3);  // true
List<Integer> inorder = tree.inorderTraversal();
```

---

## 10. Đáp án tham khảo

<details>
<summary>Bài 1: Generic Stack</summary>

```java
public class Stack<T> {
    private List<T> items = new ArrayList<>();

    public void push(T item) {
        items.add(item);
    }

    public T pop() {
        if (isEmpty()) {
            throw new EmptyStackException();
        }
        return items.remove(items.size() - 1);
    }

    public T peek() {
        if (isEmpty()) {
            throw new EmptyStackException();
        }
        return items.get(items.size() - 1);
    }

    public boolean isEmpty() {
        return items.isEmpty();
    }

    public int size() {
        return items.size();
    }

    public static void main(String[] args) {
        Stack<String> stack = new Stack<>();
        stack.push("A");
        stack.push("B");
        stack.push("C");

        System.out.println(stack.peek());  // C
        System.out.println(stack.pop());   // C
        System.out.println(stack.size());  // 2
    }
}
```
</details>

<details>
<summary>Bài 3: Generic Filter</summary>

```java
import java.util.function.Predicate;

public class FilterUtils {

    public static <T> List<T> filter(List<T> list, Predicate<T> predicate) {
        List<T> result = new ArrayList<>();
        for (T item : list) {
            if (predicate.test(item)) {
                result.add(item);
            }
        }
        return result;
    }

    public static void main(String[] args) {
        List<Integer> numbers = Arrays.asList(1, 2, 3, 4, 5, 6, 7, 8, 9, 10);

        // Filter even numbers
        List<Integer> evens = filter(numbers, n -> n % 2 == 0);
        System.out.println(evens);  // [2, 4, 6, 8, 10]

        // Filter numbers > 5
        List<Integer> greaterThan5 = filter(numbers, n -> n > 5);
        System.out.println(greaterThan5);  // [6, 7, 8, 9, 10]

        List<String> strings = Arrays.asList("apple", "pie", "banana", "hi");
        List<String> longStrings = filter(strings, s -> s.length() > 3);
        System.out.println(longStrings);  // [apple, banana]
    }
}
```
</details>

<details>
<summary>Bài 4: Generic Cache</summary>

```java
public class Cache<K, V> {
    private Map<K, CacheEntry<V>> cache = new HashMap<>();
    private long ttlMillis;

    public Cache(long ttlMillis) {
        this.ttlMillis = ttlMillis;
    }

    public void put(K key, V value) {
        cache.put(key, new CacheEntry<>(value, System.currentTimeMillis()));
    }

    public V get(K key) {
        CacheEntry<V> entry = cache.get(key);
        if (entry == null) {
            return null;
        }
        if (isExpired(entry)) {
            cache.remove(key);
            return null;
        }
        return entry.getValue();
    }

    public void remove(K key) {
        cache.remove(key);
    }

    public void clear() {
        cache.clear();
    }

    private boolean isExpired(CacheEntry<V> entry) {
        return System.currentTimeMillis() - entry.getCreatedAt() > ttlMillis;
    }

    private static class CacheEntry<V> {
        private V value;
        private long createdAt;

        public CacheEntry(V value, long createdAt) {
            this.value = value;
            this.createdAt = createdAt;
        }

        public V getValue() { return value; }
        public long getCreatedAt() { return createdAt; }
    }

    public static void main(String[] args) throws InterruptedException {
        Cache<String, String> cache = new Cache<>(1000);  // 1 second TTL

        cache.put("key1", "value1");
        System.out.println(cache.get("key1"));  // value1

        Thread.sleep(1500);
        System.out.println(cache.get("key1"));  // null (expired)
    }
}
```
</details>

---

## Navigation

- [← Day 7: Collections Basics](./day-07-collections-basics.md)
- [Day 9: Lambda & Functional →](./day-09-lambda-functional.md)
