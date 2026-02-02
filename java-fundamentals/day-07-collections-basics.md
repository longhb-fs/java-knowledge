# Day 7: Collections Basics

## Mục tiêu
- Hiểu Collections Framework
- List: ArrayList, LinkedList
- Set: HashSet, TreeSet, LinkedHashSet
- Map: HashMap, TreeMap, LinkedHashMap
- Iteration và common operations

---

## 1. Collections Framework Overview

```
Collection (interface)
├── List (interface) - ordered, allows duplicates
│   ├── ArrayList
│   ├── LinkedList
│   └── Vector (legacy)
│
├── Set (interface) - no duplicates
│   ├── HashSet
│   ├── LinkedHashSet
│   └── TreeSet (sorted)
│
└── Queue (interface)
    ├── PriorityQueue
    └── Deque
        └── ArrayDeque

Map (interface) - key-value pairs
├── HashMap
├── LinkedHashMap
├── TreeMap (sorted)
└── Hashtable (legacy)
```

---

## 2. List Interface

### 2.1. ArrayList

```java
import java.util.ArrayList;
import java.util.List;

// Tạo ArrayList
List<String> fruits = new ArrayList<>();

// Thêm phần tử
fruits.add("Apple");
fruits.add("Banana");
fruits.add("Cherry");
fruits.add(1, "Blueberry");  // Thêm vào index 1

// Truy cập
String first = fruits.get(0);  // "Apple"
int size = fruits.size();      // 4

// Sửa đổi
fruits.set(0, "Avocado");  // Thay thế tại index 0

// Xóa
fruits.remove(0);           // Xóa theo index
fruits.remove("Banana");    // Xóa theo giá trị

// Kiểm tra
boolean has = fruits.contains("Cherry");  // true
int index = fruits.indexOf("Cherry");     // Vị trí hoặc -1

// Clear
fruits.clear();  // Xóa tất cả
boolean empty = fruits.isEmpty();  // true
```

### 2.2. LinkedList

```java
import java.util.LinkedList;

LinkedList<String> list = new LinkedList<>();

// Thêm vào đầu/cuối
list.addFirst("First");
list.addLast("Last");
list.add("Middle");  // addLast by default

// Lấy đầu/cuối
String first = list.getFirst();
String last = list.getLast();

// Xóa đầu/cuối
list.removeFirst();
list.removeLast();

// Peek (không xóa) và Poll (xóa)
String peek = list.peek();      // Xem phần tử đầu
String poll = list.poll();      // Lấy và xóa phần tử đầu

// Có thể dùng như Stack
list.push("New");   // addFirst
String top = list.pop();  // removeFirst
```

### 2.3. ArrayList vs LinkedList

| Feature | ArrayList | LinkedList |
|---------|-----------|------------|
| Access by index | O(1) ✅ | O(n) |
| Add/Remove at end | O(1) | O(1) |
| Add/Remove at middle | O(n) | O(1) * |
| Memory | Less | More (nodes) |
| Best for | Random access | Frequent insertions |

```java
// Khi nào dùng gì?

// ArrayList - truy cập nhiều, ít insert/delete
List<String> products = new ArrayList<>();
products.get(100);  // O(1) - nhanh

// LinkedList - thêm/xóa đầu/cuối nhiều
LinkedList<String> queue = new LinkedList<>();
queue.addFirst(item);  // O(1)
queue.removeLast();    // O(1)
```

### 2.4. List Operations

```java
List<Integer> numbers = new ArrayList<>(Arrays.asList(3, 1, 4, 1, 5, 9, 2, 6));

// Sort
Collections.sort(numbers);  // [1, 1, 2, 3, 4, 5, 6, 9]
numbers.sort(Comparator.reverseOrder());  // Descending

// Shuffle
Collections.shuffle(numbers);

// Reverse
Collections.reverse(numbers);

// Min/Max
int min = Collections.min(numbers);
int max = Collections.max(numbers);

// Binary search (list phải được sort)
Collections.sort(numbers);
int index = Collections.binarySearch(numbers, 5);

// Sublist
List<Integer> subList = numbers.subList(0, 3);  // View, không phải copy

// Copy
List<Integer> copy = new ArrayList<>(numbers);

// Unmodifiable list
List<Integer> readOnly = Collections.unmodifiableList(numbers);
// readOnly.add(10);  // UnsupportedOperationException

// List.of() - immutable (Java 9+)
List<String> immutable = List.of("a", "b", "c");
```

---

## 3. Set Interface

### 3.1. HashSet

```java
import java.util.HashSet;
import java.util.Set;

Set<String> fruits = new HashSet<>();

// Thêm (không có duplicates)
fruits.add("Apple");
fruits.add("Banana");
fruits.add("Apple");  // Không thêm được, đã tồn tại
System.out.println(fruits.size());  // 2

// Kiểm tra
boolean has = fruits.contains("Apple");  // true

// Xóa
fruits.remove("Banana");

// Iteration (không có thứ tự cố định)
for (String fruit : fruits) {
    System.out.println(fruit);
}
```

### 3.2. LinkedHashSet

```java
// Giữ thứ tự thêm vào
Set<String> set = new LinkedHashSet<>();
set.add("C");
set.add("A");
set.add("B");

for (String s : set) {
    System.out.print(s + " ");  // C A B (thứ tự thêm)
}
```

### 3.3. TreeSet

```java
import java.util.TreeSet;

// Tự động sort
Set<Integer> numbers = new TreeSet<>();
numbers.add(5);
numbers.add(2);
numbers.add(8);
numbers.add(1);

for (int n : numbers) {
    System.out.print(n + " ");  // 1 2 5 8 (đã sort)
}

// TreeSet với custom Comparator
TreeSet<String> names = new TreeSet<>(String.CASE_INSENSITIVE_ORDER);
names.add("John");
names.add("alice");
names.add("Bob");
// Sorted case-insensitively

// Navigation methods
TreeSet<Integer> ts = new TreeSet<>(Arrays.asList(10, 20, 30, 40, 50));
ts.first();        // 10
ts.last();         // 50
ts.lower(30);      // 20 (< 30)
ts.higher(30);     // 40 (> 30)
ts.floor(35);      // 30 (<= 35)
ts.ceiling(35);    // 40 (>= 35)
ts.headSet(30);    // [10, 20] (< 30)
ts.tailSet(30);    // [30, 40, 50] (>= 30)
```

### 3.4. Set Operations

```java
Set<Integer> set1 = new HashSet<>(Arrays.asList(1, 2, 3, 4, 5));
Set<Integer> set2 = new HashSet<>(Arrays.asList(4, 5, 6, 7, 8));

// Union (hợp)
Set<Integer> union = new HashSet<>(set1);
union.addAll(set2);  // [1, 2, 3, 4, 5, 6, 7, 8]

// Intersection (giao)
Set<Integer> intersection = new HashSet<>(set1);
intersection.retainAll(set2);  // [4, 5]

// Difference (hiệu)
Set<Integer> difference = new HashSet<>(set1);
difference.removeAll(set2);  // [1, 2, 3]

// Symmetric difference (đối xứng)
Set<Integer> symDiff = new HashSet<>(set1);
symDiff.addAll(set2);
Set<Integer> tmp = new HashSet<>(set1);
tmp.retainAll(set2);
symDiff.removeAll(tmp);  // [1, 2, 3, 6, 7, 8]
```

---

## 4. Map Interface

### 4.1. HashMap

```java
import java.util.HashMap;
import java.util.Map;

Map<String, Integer> ages = new HashMap<>();

// Put
ages.put("John", 25);
ages.put("Jane", 30);
ages.put("Bob", 35);
ages.put("John", 26);  // Update existing key

// Get
int johnAge = ages.get("John");        // 26
int unknownAge = ages.get("Unknown");  // null
int safeAge = ages.getOrDefault("Unknown", 0);  // 0

// Check
boolean hasJohn = ages.containsKey("John");    // true
boolean has25 = ages.containsValue(25);        // false (đã update)

// Remove
ages.remove("Bob");
ages.remove("John", 26);  // Remove only if value matches

// Size
int size = ages.size();

// putIfAbsent - chỉ put nếu key chưa tồn tại
ages.putIfAbsent("Alice", 28);

// computeIfAbsent - compute value nếu key chưa tồn tại
ages.computeIfAbsent("Charlie", key -> key.length() * 10);

// computeIfPresent
ages.computeIfPresent("Jane", (key, value) -> value + 1);

// merge
ages.merge("Jane", 1, Integer::sum);  // Cộng thêm 1
```

### 4.2. Iteration

```java
Map<String, Integer> map = new HashMap<>();
map.put("A", 1);
map.put("B", 2);
map.put("C", 3);

// Iterate keys
for (String key : map.keySet()) {
    System.out.println(key);
}

// Iterate values
for (Integer value : map.values()) {
    System.out.println(value);
}

// Iterate entries
for (Map.Entry<String, Integer> entry : map.entrySet()) {
    System.out.println(entry.getKey() + " = " + entry.getValue());
}

// forEach (Java 8+)
map.forEach((key, value) -> {
    System.out.println(key + " = " + value);
});
```

### 4.3. LinkedHashMap

```java
// Giữ thứ tự thêm vào
Map<String, Integer> map = new LinkedHashMap<>();
map.put("C", 3);
map.put("A", 1);
map.put("B", 2);

map.forEach((k, v) -> System.out.print(k + " "));  // C A B

// Access order (LRU cache)
Map<String, Integer> lruCache = new LinkedHashMap<>(16, 0.75f, true);
lruCache.put("A", 1);
lruCache.put("B", 2);
lruCache.put("C", 3);
lruCache.get("A");  // Access A
// Order now: B C A (A moved to end)
```

### 4.4. TreeMap

```java
// Sorted by keys
Map<String, Integer> map = new TreeMap<>();
map.put("Charlie", 3);
map.put("Alice", 1);
map.put("Bob", 2);

map.forEach((k, v) -> System.out.print(k + " "));  // Alice Bob Charlie

// Navigation methods
TreeMap<Integer, String> treeMap = new TreeMap<>();
treeMap.put(1, "One");
treeMap.put(3, "Three");
treeMap.put(5, "Five");

treeMap.firstKey();     // 1
treeMap.lastKey();      // 5
treeMap.lowerKey(3);    // 1
treeMap.higherKey(3);   // 5
treeMap.headMap(3);     // {1=One}
treeMap.tailMap(3);     // {3=Three, 5=Five}
```

### 4.5. Map Comparison

| Feature | HashMap | LinkedHashMap | TreeMap |
|---------|---------|---------------|---------|
| Order | None | Insertion order | Sorted |
| null keys | ✅ 1 null | ✅ 1 null | ❌ No |
| Performance | O(1) | O(1) | O(log n) |
| Use case | General | Order matters | Sorted keys |

---

## 5. Iteration Methods

### 5.1. For-each Loop

```java
List<String> list = Arrays.asList("A", "B", "C");

for (String item : list) {
    System.out.println(item);
}
```

### 5.2. Iterator

```java
List<String> list = new ArrayList<>(Arrays.asList("A", "B", "C"));
Iterator<String> iterator = list.iterator();

while (iterator.hasNext()) {
    String item = iterator.next();
    if (item.equals("B")) {
        iterator.remove();  // Safe removal during iteration
    }
}
```

### 5.3. ListIterator

```java
List<String> list = new ArrayList<>(Arrays.asList("A", "B", "C"));
ListIterator<String> li = list.listIterator();

// Forward
while (li.hasNext()) {
    System.out.println(li.next());
}

// Backward
while (li.hasPrevious()) {
    System.out.println(li.previous());
}

// Modify during iteration
li = list.listIterator();
while (li.hasNext()) {
    String item = li.next();
    li.set(item.toLowerCase());  // Replace current
    if (item.equals("b")) {
        li.add("B2");  // Add after current
    }
}
```

### 5.4. forEach Method (Java 8+)

```java
List<String> list = Arrays.asList("A", "B", "C");
list.forEach(item -> System.out.println(item));
list.forEach(System.out::println);  // Method reference
```

---

## 6. Utility Classes

### 6.1. Collections

```java
List<Integer> list = new ArrayList<>(Arrays.asList(3, 1, 4, 1, 5));

// Sort
Collections.sort(list);
Collections.sort(list, Collections.reverseOrder());

// Search
Collections.binarySearch(list, 4);  // Requires sorted list

// Shuffle
Collections.shuffle(list);

// Reverse
Collections.reverse(list);

// Swap
Collections.swap(list, 0, 1);

// Fill
Collections.fill(list, 0);  // All elements = 0

// Copy
List<Integer> dest = new ArrayList<>(Arrays.asList(0, 0, 0, 0, 0));
Collections.copy(dest, list);

// Min/Max
Collections.min(list);
Collections.max(list);

// Frequency
Collections.frequency(list, 1);  // Count of 1

// nCopies
List<String> copies = Collections.nCopies(5, "A");  // [A, A, A, A, A]

// Immutable
List<String> empty = Collections.emptyList();
List<String> singleton = Collections.singletonList("Only");
List<String> unmodifiable = Collections.unmodifiableList(list);
```

### 6.2. Arrays

```java
int[] arr = {3, 1, 4, 1, 5};

// Sort
Arrays.sort(arr);

// Binary search
Arrays.binarySearch(arr, 4);

// Fill
Arrays.fill(arr, 0);

// Copy
int[] copy = Arrays.copyOf(arr, 10);  // Larger size
int[] range = Arrays.copyOfRange(arr, 1, 4);

// Compare
Arrays.equals(arr, copy);

// toString
Arrays.toString(arr);  // [3, 1, 4, 1, 5]

// Convert to List
List<Integer> list = Arrays.asList(1, 2, 3);  // Fixed-size list
List<Integer> mutableList = new ArrayList<>(Arrays.asList(1, 2, 3));

// Stream
Arrays.stream(arr).sum();
Arrays.stream(arr).average();
```

---

## 7. Comparable và Comparator

### 7.1. Comparable

```java
public class Student implements Comparable<Student> {
    private String name;
    private int age;
    private double gpa;

    // Constructor, getters, setters...

    @Override
    public int compareTo(Student other) {
        // Sort by GPA descending
        return Double.compare(other.gpa, this.gpa);
    }
}

// Sử dụng
List<Student> students = new ArrayList<>();
students.add(new Student("John", 20, 3.5));
students.add(new Student("Jane", 22, 3.8));
students.add(new Student("Bob", 21, 3.2));

Collections.sort(students);  // Sorted by GPA descending
```

### 7.2. Comparator

```java
// Anonymous class
Comparator<Student> byName = new Comparator<Student>() {
    @Override
    public int compare(Student s1, Student s2) {
        return s1.getName().compareTo(s2.getName());
    }
};

// Lambda
Comparator<Student> byAge = (s1, s2) -> Integer.compare(s1.getAge(), s2.getAge());

// Method reference
Comparator<Student> byGpa = Comparator.comparing(Student::getGpa);

// Chaining
Comparator<Student> byNameThenAge = Comparator
    .comparing(Student::getName)
    .thenComparingInt(Student::getAge);

// Reversed
Comparator<Student> byGpaDesc = Comparator.comparing(Student::getGpa).reversed();

// Null-safe
Comparator<Student> nullSafe = Comparator.nullsFirst(
    Comparator.comparing(Student::getName)
);

// Sử dụng
Collections.sort(students, byName);
students.sort(byAge);
```

---

## 8. Bài tập thực hành

### Bài 1: Word Counter
Đếm số lần xuất hiện của mỗi từ trong một đoạn văn.

```java
String text = "the quick brown fox jumps over the lazy dog the fox";
// Output: {the=3, quick=1, brown=1, fox=2, ...}
```

---

### Bài 2: Remove Duplicates
Viết method xóa duplicates từ List, giữ thứ tự.

```java
List<Integer> list = Arrays.asList(1, 2, 3, 2, 1, 4, 3, 5);
// Output: [1, 2, 3, 4, 5]
```

---

### Bài 3: Find First Non-Repeated Character
Tìm ký tự đầu tiên không lặp lại trong chuỗi.

```java
findFirstUnique("swiss");  // 'w'
findFirstUnique("aabbcc"); // null
```

---

### Bài 4: Group Students by Grade
Nhóm học sinh theo hạng (A, B, C, D, F) dựa trên điểm.

---

### Bài 5: LRU Cache
Implement LRU (Least Recently Used) Cache với capacity cố định.

```java
LRUCache<Integer, String> cache = new LRUCache<>(3);
cache.put(1, "One");
cache.put(2, "Two");
cache.put(3, "Three");
cache.get(1);  // Access 1
cache.put(4, "Four");  // Evict 2 (least recently used)
```

---

## 9. Đáp án tham khảo

<details>
<summary>Bài 1: Word Counter</summary>

```java
public class WordCounter {

    public static Map<String, Integer> countWords(String text) {
        Map<String, Integer> wordCount = new HashMap<>();

        String[] words = text.toLowerCase().split("\\s+");
        for (String word : words) {
            wordCount.merge(word, 1, Integer::sum);
            // Hoặc:
            // wordCount.put(word, wordCount.getOrDefault(word, 0) + 1);
        }

        return wordCount;
    }

    // Sắp xếp theo frequency
    public static Map<String, Integer> countWordsSorted(String text) {
        Map<String, Integer> wordCount = countWords(text);

        // Sort by value descending
        return wordCount.entrySet().stream()
            .sorted(Map.Entry.<String, Integer>comparingByValue().reversed())
            .collect(Collectors.toMap(
                Map.Entry::getKey,
                Map.Entry::getValue,
                (e1, e2) -> e1,
                LinkedHashMap::new
            ));
    }

    public static void main(String[] args) {
        String text = "the quick brown fox jumps over the lazy dog the fox";
        Map<String, Integer> result = countWords(text);
        result.forEach((word, count) ->
            System.out.println(word + ": " + count));
    }
}
```
</details>

<details>
<summary>Bài 2: Remove Duplicates</summary>

```java
public class RemoveDuplicates {

    public static <T> List<T> removeDuplicates(List<T> list) {
        // LinkedHashSet giữ thứ tự
        return new ArrayList<>(new LinkedHashSet<>(list));
    }

    // Không dùng Set
    public static <T> List<T> removeDuplicatesManual(List<T> list) {
        List<T> result = new ArrayList<>();
        for (T item : list) {
            if (!result.contains(item)) {
                result.add(item);
            }
        }
        return result;
    }

    public static void main(String[] args) {
        List<Integer> list = Arrays.asList(1, 2, 3, 2, 1, 4, 3, 5);
        System.out.println(removeDuplicates(list));  // [1, 2, 3, 4, 5]
    }
}
```
</details>

<details>
<summary>Bài 3: First Non-Repeated Character</summary>

```java
public class FirstUnique {

    public static Character findFirstUnique(String str) {
        Map<Character, Integer> charCount = new LinkedHashMap<>();

        for (char c : str.toCharArray()) {
            charCount.merge(c, 1, Integer::sum);
        }

        for (Map.Entry<Character, Integer> entry : charCount.entrySet()) {
            if (entry.getValue() == 1) {
                return entry.getKey();
            }
        }

        return null;
    }

    public static void main(String[] args) {
        System.out.println(findFirstUnique("swiss"));   // w
        System.out.println(findFirstUnique("aabbcc"));  // null
        System.out.println(findFirstUnique("aabbc"));   // c
    }
}
```
</details>

<details>
<summary>Bài 5: LRU Cache</summary>

```java
public class LRUCache<K, V> extends LinkedHashMap<K, V> {
    private final int capacity;

    public LRUCache(int capacity) {
        super(capacity, 0.75f, true);  // accessOrder = true
        this.capacity = capacity;
    }

    @Override
    protected boolean removeEldestEntry(Map.Entry<K, V> eldest) {
        return size() > capacity;
    }

    public static void main(String[] args) {
        LRUCache<Integer, String> cache = new LRUCache<>(3);

        cache.put(1, "One");
        cache.put(2, "Two");
        cache.put(3, "Three");
        System.out.println(cache);  // {1=One, 2=Two, 3=Three}

        cache.get(1);  // Access 1
        System.out.println(cache);  // {2=Two, 3=Three, 1=One}

        cache.put(4, "Four");  // Evict 2
        System.out.println(cache);  // {3=Three, 1=One, 4=Four}
    }
}
```
</details>

---

## Navigation

- [← Day 6: Strings & Wrappers](./day-06-strings-wrappers.md)
- [Day 8: Generics →](./day-08-generics.md)
