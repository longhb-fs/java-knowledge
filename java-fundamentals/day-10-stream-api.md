# Day 10: Stream API

## Mục tiêu
- Hiểu Stream là gì
- Intermediate operations
- Terminal operations
- Collectors
- Parallel streams

---

## 1. Stream Basics

### 1.1. Stream là gì?

```java
// Stream = sequence of elements + operations
// KHÔNG phải data structure, mà là abstraction cho processing

List<String> names = Arrays.asList("John", "Jane", "Bob", "Alice");

// Imperative style (cách cũ)
List<String> result = new ArrayList<>();
for (String name : names) {
    if (name.length() > 3) {
        result.add(name.toUpperCase());
    }
}

// Stream style (declarative)
List<String> result2 = names.stream()
    .filter(name -> name.length() > 3)
    .map(String::toUpperCase)
    .collect(Collectors.toList());
```

### 1.2. Tạo Stream

```java
// Từ Collection
List<String> list = Arrays.asList("a", "b", "c");
Stream<String> stream1 = list.stream();

// Từ Array
String[] array = {"a", "b", "c"};
Stream<String> stream2 = Arrays.stream(array);

// Stream.of()
Stream<String> stream3 = Stream.of("a", "b", "c");

// Stream.empty()
Stream<String> empty = Stream.empty();

// Stream.generate() - infinite
Stream<Double> randoms = Stream.generate(Math::random).limit(5);

// Stream.iterate() - infinite
Stream<Integer> evens = Stream.iterate(0, n -> n + 2).limit(10);
// Java 9+: với predicate
Stream<Integer> evens2 = Stream.iterate(0, n -> n < 20, n -> n + 2);

// IntStream, LongStream, DoubleStream
IntStream intStream = IntStream.range(1, 10);      // 1-9
IntStream intStream2 = IntStream.rangeClosed(1, 10); // 1-10

// From String
IntStream chars = "Hello".chars();

// Files
Stream<String> lines = Files.lines(Path.of("file.txt"));
```

### 1.3. Stream Operations

```
Source → Intermediate Operations → Terminal Operation
         (lazy)                    (triggers execution)
```

```java
List<String> names = Arrays.asList("John", "Jane", "Bob", "Alice");

names.stream()                    // Source
    .filter(n -> n.length() > 3)  // Intermediate
    .map(String::toUpperCase)     // Intermediate
    .sorted()                     // Intermediate
    .forEach(System.out::println); // Terminal
```

---

## 2. Intermediate Operations

### 2.1. filter()

```java
List<Integer> numbers = Arrays.asList(1, 2, 3, 4, 5, 6, 7, 8, 9, 10);

// Filter even numbers
List<Integer> evens = numbers.stream()
    .filter(n -> n % 2 == 0)
    .collect(Collectors.toList());
// [2, 4, 6, 8, 10]

// Multiple filters
List<Integer> result = numbers.stream()
    .filter(n -> n > 3)
    .filter(n -> n < 8)
    .collect(Collectors.toList());
// [4, 5, 6, 7]
```

### 2.2. map()

```java
List<String> names = Arrays.asList("john", "jane", "bob");

// Transform elements
List<String> upperNames = names.stream()
    .map(String::toUpperCase)
    .collect(Collectors.toList());
// [JOHN, JANE, BOB]

// Map to different type
List<Integer> lengths = names.stream()
    .map(String::length)
    .collect(Collectors.toList());
// [4, 4, 3]

// mapToInt, mapToLong, mapToDouble
int totalLength = names.stream()
    .mapToInt(String::length)
    .sum();
// 11
```

### 2.3. flatMap()

```java
// Flatten nested structures
List<List<Integer>> nested = Arrays.asList(
    Arrays.asList(1, 2, 3),
    Arrays.asList(4, 5, 6),
    Arrays.asList(7, 8, 9)
);

List<Integer> flat = nested.stream()
    .flatMap(List::stream)
    .collect(Collectors.toList());
// [1, 2, 3, 4, 5, 6, 7, 8, 9]

// Split strings
List<String> sentences = Arrays.asList("Hello World", "Java Stream");
List<String> words = sentences.stream()
    .flatMap(s -> Arrays.stream(s.split(" ")))
    .collect(Collectors.toList());
// [Hello, World, Java, Stream]
```

### 2.4. sorted()

```java
List<Integer> numbers = Arrays.asList(5, 2, 8, 1, 9);

// Natural order
List<Integer> sorted = numbers.stream()
    .sorted()
    .collect(Collectors.toList());
// [1, 2, 5, 8, 9]

// Custom comparator
List<Integer> descending = numbers.stream()
    .sorted(Comparator.reverseOrder())
    .collect(Collectors.toList());
// [9, 8, 5, 2, 1]

// Sort objects
List<Person> people = Arrays.asList(
    new Person("John", 25),
    new Person("Jane", 30),
    new Person("Bob", 20)
);

List<Person> sortedByAge = people.stream()
    .sorted(Comparator.comparing(Person::getAge))
    .collect(Collectors.toList());

// Multiple criteria
List<Person> sorted = people.stream()
    .sorted(Comparator.comparing(Person::getAge)
                      .thenComparing(Person::getName))
    .collect(Collectors.toList());
```

### 2.5. distinct()

```java
List<Integer> numbers = Arrays.asList(1, 2, 2, 3, 3, 3, 4);

List<Integer> unique = numbers.stream()
    .distinct()
    .collect(Collectors.toList());
// [1, 2, 3, 4]
```

### 2.6. limit() và skip()

```java
List<Integer> numbers = Arrays.asList(1, 2, 3, 4, 5, 6, 7, 8, 9, 10);

// First 5 elements
List<Integer> first5 = numbers.stream()
    .limit(5)
    .collect(Collectors.toList());
// [1, 2, 3, 4, 5]

// Skip first 5 elements
List<Integer> after5 = numbers.stream()
    .skip(5)
    .collect(Collectors.toList());
// [6, 7, 8, 9, 10]

// Pagination
int page = 2;
int pageSize = 3;
List<Integer> pageData = numbers.stream()
    .skip((page - 1) * pageSize)
    .limit(pageSize)
    .collect(Collectors.toList());
// [4, 5, 6]
```

### 2.7. peek()

```java
// Debug/logging
List<Integer> result = Stream.of(1, 2, 3, 4, 5)
    .filter(n -> n > 2)
    .peek(n -> System.out.println("After filter: " + n))
    .map(n -> n * 2)
    .peek(n -> System.out.println("After map: " + n))
    .collect(Collectors.toList());
```

### 2.8. takeWhile() và dropWhile() (Java 9+)

```java
List<Integer> numbers = Arrays.asList(1, 2, 3, 4, 5, 4, 3, 2, 1);

// Take while condition is true
List<Integer> taken = numbers.stream()
    .takeWhile(n -> n < 4)
    .collect(Collectors.toList());
// [1, 2, 3]

// Drop while condition is true
List<Integer> dropped = numbers.stream()
    .dropWhile(n -> n < 4)
    .collect(Collectors.toList());
// [4, 5, 4, 3, 2, 1]
```

---

## 3. Terminal Operations

### 3.1. forEach()

```java
List<String> names = Arrays.asList("John", "Jane", "Bob");

names.stream()
    .forEach(System.out::println);

// forEachOrdered (for parallel streams)
names.parallelStream()
    .forEachOrdered(System.out::println);
```

### 3.2. count()

```java
long count = Stream.of(1, 2, 3, 4, 5)
    .filter(n -> n > 2)
    .count();
// 3
```

### 3.3. collect()

```java
List<String> names = Arrays.asList("John", "Jane", "Bob");

// To List
List<String> list = names.stream()
    .collect(Collectors.toList());

// To Set
Set<String> set = names.stream()
    .collect(Collectors.toSet());

// To specific collection
LinkedList<String> linkedList = names.stream()
    .collect(Collectors.toCollection(LinkedList::new));
```

### 3.4. reduce()

```java
List<Integer> numbers = Arrays.asList(1, 2, 3, 4, 5);

// Sum
int sum = numbers.stream()
    .reduce(0, (a, b) -> a + b);
// 15

// Product
int product = numbers.stream()
    .reduce(1, (a, b) -> a * b);
// 120

// Max (with Optional)
Optional<Integer> max = numbers.stream()
    .reduce(Integer::max);
max.ifPresent(System.out::println);  // 5

// Concatenate strings
List<String> words = Arrays.asList("Hello", " ", "World");
String sentence = words.stream()
    .reduce("", String::concat);
// "Hello World"
```

### 3.5. findFirst() và findAny()

```java
List<Integer> numbers = Arrays.asList(1, 2, 3, 4, 5);

Optional<Integer> first = numbers.stream()
    .filter(n -> n > 3)
    .findFirst();
// Optional[4]

Optional<Integer> any = numbers.parallelStream()
    .filter(n -> n > 3)
    .findAny();
// Could be 4 or 5 (non-deterministic in parallel)
```

### 3.6. anyMatch(), allMatch(), noneMatch()

```java
List<Integer> numbers = Arrays.asList(1, 2, 3, 4, 5);

boolean anyEven = numbers.stream()
    .anyMatch(n -> n % 2 == 0);
// true

boolean allPositive = numbers.stream()
    .allMatch(n -> n > 0);
// true

boolean noneNegative = numbers.stream()
    .noneMatch(n -> n < 0);
// true
```

### 3.7. min() và max()

```java
List<Integer> numbers = Arrays.asList(3, 1, 4, 1, 5, 9);

Optional<Integer> min = numbers.stream()
    .min(Comparator.naturalOrder());
// Optional[1]

Optional<Integer> max = numbers.stream()
    .max(Comparator.naturalOrder());
// Optional[9]

// With objects
Optional<Person> youngest = people.stream()
    .min(Comparator.comparing(Person::getAge));
```

### 3.8. toArray()

```java
String[] array = Stream.of("a", "b", "c")
    .toArray(String[]::new);

Integer[] intArray = Stream.of(1, 2, 3)
    .toArray(Integer[]::new);
```

---

## 4. Collectors

### 4.1. Basic Collectors

```java
List<String> names = Arrays.asList("John", "Jane", "Bob", "Alice");

// toList, toSet
List<String> list = names.stream().collect(Collectors.toList());
Set<String> set = names.stream().collect(Collectors.toSet());

// toCollection
TreeSet<String> treeSet = names.stream()
    .collect(Collectors.toCollection(TreeSet::new));

// joining
String joined = names.stream()
    .collect(Collectors.joining());
// "JohnJaneBobAlice"

String joinedComma = names.stream()
    .collect(Collectors.joining(", "));
// "John, Jane, Bob, Alice"

String joinedBrackets = names.stream()
    .collect(Collectors.joining(", ", "[", "]"));
// "[John, Jane, Bob, Alice]"
```

### 4.2. Counting and Statistics

```java
List<Integer> numbers = Arrays.asList(1, 2, 3, 4, 5);

// counting
long count = numbers.stream()
    .collect(Collectors.counting());

// summingInt, summingDouble
int sum = numbers.stream()
    .collect(Collectors.summingInt(Integer::intValue));

// averagingInt, averagingDouble
double avg = numbers.stream()
    .collect(Collectors.averagingInt(Integer::intValue));

// summarizingInt
IntSummaryStatistics stats = numbers.stream()
    .collect(Collectors.summarizingInt(Integer::intValue));
System.out.println("Count: " + stats.getCount());
System.out.println("Sum: " + stats.getSum());
System.out.println("Min: " + stats.getMin());
System.out.println("Max: " + stats.getMax());
System.out.println("Avg: " + stats.getAverage());
```

### 4.3. Grouping

```java
List<Person> people = Arrays.asList(
    new Person("John", 25, "IT"),
    new Person("Jane", 30, "HR"),
    new Person("Bob", 25, "IT"),
    new Person("Alice", 28, "HR")
);

// Group by department
Map<String, List<Person>> byDept = people.stream()
    .collect(Collectors.groupingBy(Person::getDepartment));
// {IT=[John, Bob], HR=[Jane, Alice]}

// Group by with downstream collector
Map<String, Long> countByDept = people.stream()
    .collect(Collectors.groupingBy(
        Person::getDepartment,
        Collectors.counting()
    ));
// {IT=2, HR=2}

// Group by with mapping
Map<String, List<String>> namesByDept = people.stream()
    .collect(Collectors.groupingBy(
        Person::getDepartment,
        Collectors.mapping(Person::getName, Collectors.toList())
    ));
// {IT=[John, Bob], HR=[Jane, Alice]}

// Nested grouping
Map<String, Map<Integer, List<Person>>> byDeptAndAge = people.stream()
    .collect(Collectors.groupingBy(
        Person::getDepartment,
        Collectors.groupingBy(Person::getAge)
    ));
```

### 4.4. Partitioning

```java
List<Integer> numbers = Arrays.asList(1, 2, 3, 4, 5, 6, 7, 8, 9, 10);

// Partition by condition
Map<Boolean, List<Integer>> partition = numbers.stream()
    .collect(Collectors.partitioningBy(n -> n % 2 == 0));
// {false=[1, 3, 5, 7, 9], true=[2, 4, 6, 8, 10]}

// With downstream
Map<Boolean, Long> partitionCount = numbers.stream()
    .collect(Collectors.partitioningBy(
        n -> n % 2 == 0,
        Collectors.counting()
    ));
// {false=5, true=5}
```

### 4.5. toMap()

```java
List<Person> people = Arrays.asList(
    new Person("John", 25),
    new Person("Jane", 30),
    new Person("Bob", 20)
);

// Basic toMap
Map<String, Integer> nameToAge = people.stream()
    .collect(Collectors.toMap(
        Person::getName,
        Person::getAge
    ));
// {John=25, Jane=30, Bob=20}

// Handle duplicates
Map<String, Integer> nameToAge2 = people.stream()
    .collect(Collectors.toMap(
        Person::getName,
        Person::getAge,
        (existing, replacement) -> existing  // Merge function
    ));

// Specify map type
LinkedHashMap<String, Integer> linkedMap = people.stream()
    .collect(Collectors.toMap(
        Person::getName,
        Person::getAge,
        (e, r) -> e,
        LinkedHashMap::new
    ));
```

---

## 5. Primitive Streams

```java
// IntStream
IntStream intStream = IntStream.range(1, 10);
int sum = intStream.sum();
OptionalDouble avg = IntStream.of(1, 2, 3).average();

// LongStream
LongStream longStream = LongStream.rangeClosed(1, 100);
long total = longStream.sum();

// DoubleStream
DoubleStream doubleStream = DoubleStream.of(1.1, 2.2, 3.3);

// Convert between streams
IntStream ints = Stream.of(1, 2, 3).mapToInt(Integer::intValue);
Stream<Integer> boxed = IntStream.of(1, 2, 3).boxed();

// Statistics
IntSummaryStatistics stats = IntStream.range(1, 100).summaryStatistics();
```

---

## 6. Parallel Streams

```java
List<Integer> numbers = Arrays.asList(1, 2, 3, 4, 5, 6, 7, 8, 9, 10);

// Create parallel stream
long count = numbers.parallelStream()
    .filter(n -> n % 2 == 0)
    .count();

// Convert to parallel
long count2 = numbers.stream()
    .parallel()
    .filter(n -> n % 2 == 0)
    .count();

// Check if parallel
boolean isParallel = numbers.parallelStream().isParallel();

// Order matters? Use forEachOrdered
numbers.parallelStream()
    .forEachOrdered(System.out::println);

// ⚠️ Cẩn thận với side effects
List<Integer> result = new ArrayList<>();
numbers.parallelStream()
    .forEach(result::add);  // NOT thread-safe!

// ✅ Use collect instead
List<Integer> safeResult = numbers.parallelStream()
    .collect(Collectors.toList());
```

---

## 7. Bài tập thực hành

### Bài 1: Product Analysis
Cho danh sách sản phẩm:

```java
class Product {
    String name;
    String category;
    double price;
    int stock;
}
```

Viết các queries:
- Tìm sản phẩm đắt nhất trong mỗi category
- Tính tổng giá trị kho (price * stock) theo category
- Lọc sản phẩm hết hàng (stock = 0)
- Top 5 sản phẩm đắt nhất

---

### Bài 2: Text Analysis
Phân tích file text:
- Đếm số từ unique
- Tìm từ xuất hiện nhiều nhất
- Tính độ dài trung bình của từ
- Tìm các từ dài hơn 5 ký tự

---

### Bài 3: Transaction Processing
Cho danh sách giao dịch:

```java
class Transaction {
    String id;
    String type;  // "DEPOSIT" or "WITHDRAWAL"
    double amount;
    LocalDate date;
    String accountId;
}
```

Viết các queries:
- Tổng số tiền deposit theo tháng
- Account có tổng withdrawal lớn nhất
- Giao dịch lớn nhất mỗi loại
- Tất cả accounts có giao dịch trong tháng

---

### Bài 4: Stream Pipeline Builder
Tạo fluent API để build stream operations:

```java
DataProcessor<Person> processor = DataProcessor.of(people)
    .filter("age > 18")
    .sortBy("name")
    .limit(10)
    .execute();
```

---

## 8. Đáp án tham khảo

<details>
<summary>Bài 1: Product Analysis</summary>

```java
List<Product> products = Arrays.asList(
    new Product("Laptop", "Electronics", 1200, 10),
    new Product("Phone", "Electronics", 800, 0),
    new Product("Desk", "Furniture", 300, 5),
    new Product("Chair", "Furniture", 150, 20)
);

// Most expensive in each category
Map<String, Optional<Product>> expensiveByCategory = products.stream()
    .collect(Collectors.groupingBy(
        Product::getCategory,
        Collectors.maxBy(Comparator.comparing(Product::getPrice))
    ));

// Total inventory value by category
Map<String, Double> valueByCategory = products.stream()
    .collect(Collectors.groupingBy(
        Product::getCategory,
        Collectors.summingDouble(p -> p.getPrice() * p.getStock())
    ));

// Out of stock products
List<Product> outOfStock = products.stream()
    .filter(p -> p.getStock() == 0)
    .collect(Collectors.toList());

// Top 5 most expensive
List<Product> top5 = products.stream()
    .sorted(Comparator.comparing(Product::getPrice).reversed())
    .limit(5)
    .collect(Collectors.toList());
```
</details>

<details>
<summary>Bài 2: Text Analysis</summary>

```java
String text = "the quick brown fox jumps over the lazy dog the fox";

List<String> words = Arrays.asList(text.split("\\s+"));

// Unique word count
long uniqueCount = words.stream()
    .distinct()
    .count();

// Most frequent word
Map<String, Long> wordCount = words.stream()
    .collect(Collectors.groupingBy(
        Function.identity(),
        Collectors.counting()
    ));

Optional<Map.Entry<String, Long>> mostFrequent = wordCount.entrySet().stream()
    .max(Map.Entry.comparingByValue());

// Average word length
double avgLength = words.stream()
    .mapToInt(String::length)
    .average()
    .orElse(0);

// Words longer than 5 characters
List<String> longWords = words.stream()
    .filter(w -> w.length() > 5)
    .distinct()
    .collect(Collectors.toList());
```
</details>

---

## Navigation

- [← Day 9: Lambda & Functional](./day-09-lambda-functional.md)
- [Day 11: File I/O →](./day-11-file-io.md)
