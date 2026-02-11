# Day 7: Collections Basics (Bộ Sưu Tập — Cấu Trúc Dữ Liệu)

## Mục tiêu hôm nay

Sau khi học xong Day 7, bạn sẽ:
- Hiểu **Collections Framework** (khung bộ sưu tập) là gì và tại sao cần
- Sử dụng **List** (danh sách): ArrayList, LinkedList
- Sử dụng **Set** (tập hợp — không trùng lặp): HashSet, TreeSet, LinkedHashSet
- Sử dụng **Map** (bản đồ key-value): HashMap, TreeMap, LinkedHashMap
- Biết cách **duyệt** (iterate) qua các collection
- Biết cách **sắp xếp** (sort) với Comparable và Comparator

---

## Tại sao cần học Collections?

### Ví dụ đời thường

Bạn quản lý một danh sách học sinh. Bạn cần:
- **Lưu danh sách** tên học sinh → Dùng **List** (có thứ tự, cho phép trùng tên)
- **Lưu danh sách mã học sinh** → Dùng **Set** (không cho phép trùng mã)
- **Tra cứu điểm theo tên** → Dùng **Map** (key = tên, value = điểm)

Nếu chỉ dùng **mảng (array)** thì:
- ❌ Kích thước cố định — không thể thêm/xóa phần tử linh hoạt
- ❌ Không có sẵn method tìm kiếm, sắp xếp, xóa trùng
- ❌ Không có kiểu "key-value" để tra cứu nhanh

**Collections** giải quyết TẤT CẢ vấn đề trên!

```
Array (Mảng):
  ┌───┬───┬───┬───┬───┐
  │ A │ B │ C │ ? │ ? │  ← Kích thước CỐ ĐỊNH = 5
  └───┴───┴───┴───┴───┘    Muốn thêm phần tử thứ 6? KHÔNG ĐƯỢC!

ArrayList (Collection):
  ┌───┬───┬───┐
  │ A │ B │ C │  ← Kích thước TỰ ĐỘNG tăng
  └───┴───┴───┘    Thêm D? OK! → [A, B, C, D]
                   Thêm E? OK! → [A, B, C, D, E]
```

---

## 1. Collections Framework Overview (Tổng quan)

### Sơ đồ phân cấp

```
Collection (interface — gốc)
│   → "Một nhóm các phần tử"
│
├── List (interface) — Danh sách CÓ THỨ TỰ, CHO PHÉP trùng lặp
│   │   → Giống "danh sách học sinh" (có số thứ tự, cho phép trùng tên)
│   │
│   ├── ArrayList    → Mảng tự giãn nở (dùng NHIỀU NHẤT, ~90%)
│   ├── LinkedList   → Danh sách liên kết (thêm/xóa đầu-cuối nhanh)
│   └── Vector       → Giống ArrayList + thread-safe (LEGACY, ít dùng)
│
├── Set (interface) — Tập hợp KHÔNG trùng lặp
│   │   → Giống "danh sách mã số" (mỗi mã chỉ xuất hiện 1 lần)
│   │
│   ├── HashSet        → Nhanh nhất, KHÔNG giữ thứ tự
│   ├── LinkedHashSet  → Giữ thứ tự thêm vào
│   └── TreeSet        → TỰ ĐỘNG sắp xếp
│
└── Queue (interface) — Hàng đợi (FIFO: vào trước ra trước)
    │   → Giống "xếp hàng mua vé" (ai đến trước được phục vụ trước)
    │
    ├── PriorityQueue  → Hàng đợi ưu tiên (ưu tiên cao ra trước)
    └── Deque          → Hàng đợi 2 đầu (thêm/xóa ở cả 2 đầu)
        └── ArrayDeque → Triển khai nhanh nhất của Deque

Map (interface) — Bản đồ KEY-VALUE (KHÔNG thuộc Collection!)
│   → Giống "từ điển" (tra từ = key → ra nghĩa = value)
│
├── HashMap        → Nhanh nhất, KHÔNG giữ thứ tự
├── LinkedHashMap  → Giữ thứ tự thêm vào
├── TreeMap        → TỰ ĐỘNG sắp xếp theo key
└── Hashtable      → Giống HashMap + thread-safe (LEGACY, ít dùng)
```

### Bảng so sánh nhanh — Chọn cái nào?

| Cần gì? | Dùng gì? | Ví dụ |
|---------|----------|-------|
| Danh sách có thứ tự, truy cập theo index | **ArrayList** | Danh sách sản phẩm |
| Thêm/xóa đầu-cuối nhiều | **LinkedList** | Hàng đợi tin nhắn |
| Loại bỏ trùng lặp, không cần thứ tự | **HashSet** | Tập hợp email unique |
| Loại bỏ trùng lặp, giữ thứ tự thêm | **LinkedHashSet** | Lịch sử tìm kiếm |
| Loại bỏ trùng lặp, tự sắp xếp | **TreeSet** | Bảng xếp hạng |
| Tra cứu theo key nhanh | **HashMap** | Config settings |
| Tra cứu theo key, giữ thứ tự thêm | **LinkedHashMap** | Cache LRU |
| Tra cứu theo key, sắp xếp | **TreeMap** | Từ điển A-Z |

---

## 2. List Interface (Danh sách)

### Tại sao cần List?

List giống như **danh sách** trong đời thường:
- Có **thứ tự** (phần tử 1, 2, 3...)
- Cho phép **trùng lặp** (2 học sinh cùng tên cũng OK)
- Truy cập bằng **index** (vị trí)

### 2.1. ArrayList (Mảng tự giãn nở — Dùng nhiều nhất!)

ArrayList là implementation phổ biến nhất của List. Bên trong nó là một **mảng** (array), nhưng tự động **tăng kích thước** khi cần.

```java
import java.util.ArrayList;
import java.util.List;

public class ArrayListDemo {
    public static void main(String[] args) {

        // ===== TẠO ArrayList =====
        // List<String> = danh sách chứa String
        // <String> gọi là Generics — chỉ định kiểu dữ liệu
        List<String> fruits = new ArrayList<>();

        // ===== THÊM phần tử =====
        fruits.add("Táo");           // Thêm vào CUỐI → [Táo]
        fruits.add("Chuối");         // Thêm vào CUỐI → [Táo, Chuối]
        fruits.add("Cam");           // Thêm vào CUỐI → [Táo, Chuối, Cam]
        fruits.add(1, "Nho");        // Thêm vào INDEX 1 → [Táo, Nho, Chuối, Cam]
        //                                                    0    1     2      3

        // ===== TRUY CẬP phần tử =====
        String first = fruits.get(0);    // "Táo" (index 0 = đầu tiên)
        String second = fruits.get(1);   // "Nho" (index 1)
        int size = fruits.size();        // 4 (số phần tử)

        // ===== SỬA phần tử =====
        fruits.set(0, "Xoài");  // Thay index 0: "Táo" → "Xoài"
        // Giờ: [Xoài, Nho, Chuối, Cam]

        // ===== XÓA phần tử =====
        fruits.remove(0);          // Xóa theo INDEX → xóa "Xoài"
        fruits.remove("Chuối");    // Xóa theo GIÁ TRỊ → xóa "Chuối"
        // Giờ: [Nho, Cam]

        // ===== KIỂM TRA =====
        boolean hasCam = fruits.contains("Cam");  // true (có "Cam" trong list)
        int index = fruits.indexOf("Cam");         // 1 (vị trí của "Cam")
        int notFound = fruits.indexOf("Dưa");      // -1 (KHÔNG TÌM THẤY)

        // ===== XÓA TẤT CẢ =====
        fruits.clear();                   // Xóa hết → []
        boolean empty = fruits.isEmpty(); // true (list rỗng)

        // ===== IN TOÀN BỘ LIST =====
        System.out.println(fruits);  // [] (ArrayList tự có toString)
    }
}
```

### 2.2. LinkedList (Danh sách liên kết)

LinkedList hoạt động khác ArrayList: mỗi phần tử là một **node** (nút) trỏ đến node tiếp theo, giống **chuỗi xích**.

```
ArrayList (mảng liên tục trong bộ nhớ):
  ┌───┬───┬───┬───┐
  │ A │ B │ C │ D │  ← Các phần tử NẰM CẠN NHAU
  └───┴───┴───┴───┘

LinkedList (các node trỏ đến nhau):
  ┌───┐    ┌───┐    ┌───┐    ┌───┐
  │ A │───→│ B │───→│ C │───→│ D │  ← Mỗi node GIỮ LINK đến node tiếp
  └───┘    └───┘    └───┘    └───┘
```

```java
import java.util.LinkedList;

public class LinkedListDemo {
    public static void main(String[] args) {
        LinkedList<String> list = new LinkedList<>();

        // Thêm vào ĐẦU và CUỐI (đặc biệt của LinkedList)
        list.addFirst("Đầu tiên");  // Thêm vào đầu
        list.addLast("Cuối cùng");  // Thêm vào cuối
        list.add("Giữa");           // addLast() mặc định

        // Lấy phần tử ĐẦU và CUỐI
        String first = list.getFirst();  // "Đầu tiên"
        String last = list.getLast();    // "Giữa"

        // Xóa ĐẦU và CUỐI
        list.removeFirst();  // Xóa "Đầu tiên"
        list.removeLast();   // Xóa "Giữa"

        // peek() — Xem đầu tiên (KHÔNG xóa)
        String peek = list.peek();   // "Cuối cùng" (chỉ xem, không xóa)

        // poll() — Lấy đầu tiên (CÓ xóa)
        String poll = list.poll();   // "Cuối cùng" (lấy ra + xóa khỏi list)

        // Dùng như STACK (ngăn xếp — LIFO: vào sau ra trước)
        list.push("A");     // Thêm vào đầu (giống addFirst)
        list.push("B");     // Thêm vào đầu → [B, A]
        String top = list.pop();  // Lấy từ đầu → "B" (giống removeFirst)
    }
}
```

### 2.3. ArrayList vs LinkedList — Khi nào dùng gì?

| Tiêu chí | ArrayList | LinkedList |
|----------|-----------|------------|
| **Truy cập index** (get/set) | ✅ O(1) — CỰC NHANH | ❌ O(n) — Phải duyệt từ đầu |
| **Thêm/xóa CUỐI** | ✅ O(1) | ✅ O(1) |
| **Thêm/xóa GIỮA** | ❌ O(n) — Phải dịch chuyển | ✅ O(1)* — Chỉ đổi link |
| **Bộ nhớ** | ✅ Ít hơn | ❌ Nhiều hơn (mỗi node chứa 2 link) |
| **Khi nào dùng?** | **90% trường hợp** — đọc nhiều | Thêm/xóa đầu-cuối nhiều |

💡 **Mẹo:** Nếu không biết chọn gì → **dùng ArrayList**. Nó nhanh hơn trong hầu hết trường hợp thực tế.

### 2.4. List Operations (Thao tác hữu ích)

```java
import java.util.*;

public class ListOperations {
    public static void main(String[] args) {
        List<Integer> numbers = new ArrayList<>(Arrays.asList(3, 1, 4, 1, 5, 9, 2, 6));

        // ===== SẮP XẾP =====
        Collections.sort(numbers);  // Tăng dần: [1, 1, 2, 3, 4, 5, 6, 9]
        numbers.sort(Comparator.reverseOrder());  // Giảm dần: [9, 6, 5, 4, 3, 2, 1, 1]

        // ===== TRỘN NGẪU NHIÊN =====
        Collections.shuffle(numbers);  // Xáo trộn random

        // ===== ĐẢO NGƯỢC =====
        Collections.reverse(numbers);  // Đảo ngược thứ tự

        // ===== TÌM MIN/MAX =====
        int min = Collections.min(numbers);  // Phần tử nhỏ nhất
        int max = Collections.max(numbers);  // Phần tử lớn nhất

        // ===== TÌM KIẾM NHỊ PHÂN (list phải sorted trước!) =====
        Collections.sort(numbers);
        int index = Collections.binarySearch(numbers, 5);
        // Trả về index nếu tìm thấy, số âm nếu không

        // ===== CẮT DANH SÁCH CON =====
        List<Integer> subList = numbers.subList(0, 3);
        // ⚠️ subList là VIEW (xem), không phải bản sao!
        // Sửa subList sẽ ảnh hưởng đến numbers gốc!

        // ===== SAO CHÉP =====
        List<Integer> copy = new ArrayList<>(numbers);  // Bản sao ĐỘC LẬP

        // ===== TẠO LIST KHÔNG THỂ SỬA (Immutable) =====
        // Cách 1: Collections.unmodifiableList (Java cũ)
        List<Integer> readOnly = Collections.unmodifiableList(numbers);
        // readOnly.add(10);  // UnsupportedOperationException!

        // Cách 2: List.of() (Java 9+) — ngắn gọn hơn
        List<String> immutable = List.of("a", "b", "c");
        // immutable.add("d");  // UnsupportedOperationException!
    }
}
```

---

## 3. Set Interface (Tập hợp — Không trùng lặp)

### Tại sao cần Set?

Set giống **tập hợp** trong toán học:
- **KHÔNG** cho phép phần tử trùng lặp
- Thêm phần tử đã có → **bị bỏ qua** (không lỗi)

### Ví dụ đời thường

```
Set giống DANH SÁCH EMAIL ĐĂNG KÝ:
  - user@email.com  → Thêm ✅
  - admin@email.com → Thêm ✅
  - user@email.com  → Đã có → BỎ QUA (không thêm 2 lần)
```

### 3.1. HashSet (Nhanh nhất, không giữ thứ tự)

```java
import java.util.HashSet;
import java.util.Set;

public class HashSetDemo {
    public static void main(String[] args) {
        Set<String> emails = new HashSet<>();

        // ===== THÊM phần tử =====
        emails.add("a@email.com");   // Thêm OK → true
        emails.add("b@email.com");   // Thêm OK → true
        emails.add("a@email.com");   // Đã có → BỎ QUA → false
        System.out.println(emails.size());  // 2 (chỉ có 2 email unique)

        // ===== KIỂM TRA =====
        boolean hasA = emails.contains("a@email.com");  // true

        // ===== XÓA =====
        emails.remove("b@email.com");  // Xóa 1 phần tử

        // ===== DUYỆT =====
        // ⚠️ Thứ tự KHÔNG cố định (có thể khác mỗi lần chạy!)
        for (String email : emails) {
            System.out.println(email);
        }
    }
}
```

### 3.2. LinkedHashSet (Giữ thứ tự thêm vào)

```java
import java.util.LinkedHashSet;
import java.util.Set;

// Giống HashSet nhưng GIỮ THỨ TỰ thêm vào
Set<String> set = new LinkedHashSet<>();
set.add("C");
set.add("A");
set.add("B");

for (String s : set) {
    System.out.print(s + " ");  // C A B (đúng thứ tự thêm vào!)
}
// HashSet có thể in: A B C hoặc C B A (ngẫu nhiên)
```

### 3.3. TreeSet (Tự động sắp xếp)

```java
import java.util.TreeSet;
import java.util.Set;

// TreeSet TỰ ĐỘNG sắp xếp phần tử
Set<Integer> scores = new TreeSet<>();
scores.add(85);
scores.add(92);
scores.add(78);
scores.add(95);
scores.add(88);

for (int score : scores) {
    System.out.print(score + " ");  // 78 85 88 92 95 (đã sắp xếp!)
}

// TreeSet với String → sắp xếp theo bảng chữ cái
TreeSet<String> names = new TreeSet<>();
names.add("Charlie");
names.add("Alice");
names.add("Bob");
System.out.println(names);  // [Alice, Bob, Charlie]

// ===== Navigation methods (method đặc biệt của TreeSet) =====
TreeSet<Integer> ts = new TreeSet<>(Set.of(10, 20, 30, 40, 50));

ts.first();        // 10 (phần tử NHỎ NHẤT)
ts.last();         // 50 (phần tử LỚN NHẤT)
ts.lower(30);      // 20 (phần tử LỚN NHẤT mà < 30)
ts.higher(30);     // 40 (phần tử NHỎ NHẤT mà > 30)
ts.floor(35);      // 30 (phần tử LỚN NHẤT mà <= 35)
ts.ceiling(35);    // 40 (phần tử NHỎ NHẤT mà >= 35)
ts.headSet(30);    // [10, 20] (các phần tử < 30)
ts.tailSet(30);    // [30, 40, 50] (các phần tử >= 30)
```

### 3.4. So sánh 3 loại Set

| Tiêu chí | HashSet | LinkedHashSet | TreeSet |
|----------|---------|---------------|---------|
| **Thứ tự** | ❌ Không | ✅ Thứ tự thêm | ✅ Tự sắp xếp |
| **Tốc độ** | ✅ O(1) Nhanh nhất | ✅ O(1) | ⚠️ O(log n) Chậm hơn |
| **null** | ✅ Cho phép 1 null | ✅ Cho phép 1 null | ❌ Không cho null |
| **Khi nào?** | Loại trùng nhanh | Giữ thứ tự thêm | Cần tự sắp xếp |

### 3.5. Set Operations (Phép toán tập hợp)

```java
Set<Integer> setA = new HashSet<>(Set.of(1, 2, 3, 4, 5));
Set<Integer> setB = new HashSet<>(Set.of(4, 5, 6, 7, 8));

// ===== HỢP (Union): A ∪ B — Tất cả phần tử của cả 2 set =====
Set<Integer> union = new HashSet<>(setA);
union.addAll(setB);
// union = [1, 2, 3, 4, 5, 6, 7, 8]

// ===== GIAO (Intersection): A ∩ B — Phần tử CHUNG =====
Set<Integer> intersection = new HashSet<>(setA);
intersection.retainAll(setB);
// intersection = [4, 5]

// ===== HIỆU (Difference): A - B — Phần tử chỉ có trong A =====
Set<Integer> difference = new HashSet<>(setA);
difference.removeAll(setB);
// difference = [1, 2, 3]
```

**Minh họa:**

```
Set A: {1, 2, 3, 4, 5}     Set B: {4, 5, 6, 7, 8}

Hợp (Union):        {1, 2, 3, 4, 5, 6, 7, 8}  ← Tất cả
Giao (Intersection): {4, 5}                     ← Phần chung
Hiệu (A - B):       {1, 2, 3}                  ← Chỉ có trong A
```

---

## 4. Map Interface (Bản đồ Key-Value)

### Tại sao cần Map?

Map giống **từ điển** hoặc **danh bạ điện thoại**:
- **Key** (khóa) = từ cần tra / tên người
- **Value** (giá trị) = nghĩa / số điện thoại
- Mỗi key chỉ có **1 value** (tra 1 từ → ra 1 nghĩa)
- Key **KHÔNG được trùng** (không có 2 từ giống nhau trong từ điển)

```
Map giống DANH BẠ ĐIỆN THOẠI:
  ┌─────────────────┬──────────────┐
  │ Key (Tên)       │ Value (SĐT)  │
  ├─────────────────┼──────────────┤
  │ "Nguyễn Văn A"  │ "0901234567" │
  │ "Trần Thị B"    │ "0912345678" │
  │ "Lê Văn C"      │ "0923456789" │
  └─────────────────┴──────────────┘
  Tra tên → ra số điện thoại (rất nhanh!)
```

### 4.1. HashMap (Nhanh nhất, dùng nhiều nhất)

```java
import java.util.HashMap;
import java.util.Map;

public class HashMapDemo {
    public static void main(String[] args) {
        // Map<Key, Value> = Map<String, Integer>
        // Key = tên (String), Value = tuổi (Integer)
        Map<String, Integer> ages = new HashMap<>();

        // ===== THÊM cặp key-value =====
        ages.put("An", 25);     // Thêm An: 25 tuổi
        ages.put("Bình", 30);   // Thêm Bình: 30 tuổi
        ages.put("Châu", 35);   // Thêm Châu: 35 tuổi
        ages.put("An", 26);     // Key "An" đã có → GHI ĐÈ value: 25 → 26
        //                         ⚠️ put KHÔNG thêm key mới, mà UPDATE value!

        // ===== LẤY value theo key =====
        int anAge = ages.get("An");          // 26
        Integer unknown = ages.get("Xyz");   // null (key không tồn tại)
        int safe = ages.getOrDefault("Xyz", 0);  // 0 (giá trị mặc định nếu không có)

        // ===== KIỂM TRA =====
        boolean hasAn = ages.containsKey("An");     // true (có key "An")
        boolean has25 = ages.containsValue(25);      // false (25 đã bị update thành 26)

        // ===== XÓA =====
        ages.remove("Châu");            // Xóa key "Châu"
        ages.remove("An", 99);          // KHÔNG xóa — vì value không khớp (26 ≠ 99)
        ages.remove("An", 26);          // Xóa — vì value khớp (26 = 26)

        // ===== KÍCH THƯỚC =====
        int size = ages.size();          // Số cặp key-value

        // ===== CÁC METHOD NÂNG CAO =====

        // putIfAbsent — Chỉ thêm nếu key CHƯA CÓ
        ages.putIfAbsent("Dung", 28);   // Thêm vì "Dung" chưa có
        ages.putIfAbsent("Bình", 99);   // KHÔNG thêm — "Bình" đã có → giữ 30

        // getOrDefault — Lấy value, nếu không có trả giá trị mặc định
        int dungAge = ages.getOrDefault("Dung", 0);  // 28

        // merge — Gộp value nếu key đã có
        ages.merge("Bình", 1, Integer::sum);
        // Key "Bình" đã có (30) → gộp: 30 + 1 = 31
        // Nếu key chưa có → đặt value = 1

        System.out.println(ages);
    }
}
```

### 4.2. Duyệt Map (Iteration)

Map có **3 cách duyệt** — mỗi cách lấy dữ liệu khác nhau:

```java
Map<String, Integer> map = new HashMap<>();
map.put("Toán", 9);
map.put("Lý", 8);
map.put("Hóa", 7);

// ===== Cách 1: Duyệt KEYS (chỉ lấy key) =====
System.out.println("--- Các môn học ---");
for (String key : map.keySet()) {
    System.out.println("Môn: " + key);
}

// ===== Cách 2: Duyệt VALUES (chỉ lấy value) =====
System.out.println("--- Các điểm số ---");
for (Integer value : map.values()) {
    System.out.println("Điểm: " + value);
}

// ===== Cách 3: Duyệt ENTRIES (lấy CẢ key và value) — PHỔBIẾN NHẤT =====
System.out.println("--- Bảng điểm ---");
for (Map.Entry<String, Integer> entry : map.entrySet()) {
    System.out.println(entry.getKey() + ": " + entry.getValue());
}
// Output:
// Toán: 9
// Lý: 8
// Hóa: 7

// ===== Cách 4: forEach (Java 8+) — NGẮN GỌN NHẤT =====
map.forEach((subject, score) -> {
    System.out.println(subject + " = " + score);
});
```

### 4.3. LinkedHashMap (Giữ thứ tự thêm vào)

```java
import java.util.LinkedHashMap;
import java.util.Map;

// Giống HashMap nhưng GIỮ THỨ TỰ thêm vào
Map<String, Integer> map = new LinkedHashMap<>();
map.put("C", 3);
map.put("A", 1);
map.put("B", 2);

map.forEach((k, v) -> System.out.print(k + " "));
// Output: C A B (đúng thứ tự thêm vào!)
// HashMap có thể in: A B C hoặc B C A (ngẫu nhiên)
```

### 4.4. TreeMap (Tự động sắp xếp theo key)

```java
import java.util.TreeMap;
import java.util.Map;

// TreeMap TỰ ĐỘNG sắp xếp theo KEY
Map<String, Integer> map = new TreeMap<>();
map.put("Châu", 3);
map.put("An", 1);
map.put("Bình", 2);

map.forEach((k, v) -> System.out.print(k + " "));
// Output: An Bình Châu (sắp xếp theo key A-Z!)

// ===== Navigation methods =====
TreeMap<Integer, String> treeMap = new TreeMap<>();
treeMap.put(1, "Một");
treeMap.put(3, "Ba");
treeMap.put(5, "Năm");

treeMap.firstKey();     // 1 (key nhỏ nhất)
treeMap.lastKey();      // 5 (key lớn nhất)
treeMap.lowerKey(3);    // 1 (key lớn nhất < 3)
treeMap.higherKey(3);   // 5 (key nhỏ nhất > 3)
treeMap.headMap(3);     // {1=Một} (các entry có key < 3)
treeMap.tailMap(3);     // {3=Ba, 5=Năm} (các entry có key >= 3)
```

### 4.5. So sánh 3 loại Map

| Tiêu chí | HashMap | LinkedHashMap | TreeMap |
|----------|---------|---------------|---------|
| **Thứ tự** | ❌ Không | ✅ Thứ tự thêm | ✅ Sắp xếp theo key |
| **Tốc độ** | ✅ O(1) | ✅ O(1) | ⚠️ O(log n) |
| **null key** | ✅ 1 null | ✅ 1 null | ❌ Không |
| **Khi nào?** | Tra cứu nhanh (90%) | Giữ thứ tự, cache | Cần sort theo key |

---

## 5. Iteration Methods (Các cách duyệt Collection)

### 5.1. For-each Loop (Cách đơn giản nhất)

```java
List<String> list = List.of("An", "Bình", "Châu");

// Duyệt từng phần tử
for (String name : list) {
    System.out.println("Xin chào " + name);
}
```

### 5.2. Iterator (Có thể XÓA phần tử khi đang duyệt)

```java
List<String> list = new ArrayList<>(List.of("An", "Bình", "Châu", "An"));
Iterator<String> iterator = list.iterator();

while (iterator.hasNext()) {        // Còn phần tử tiếp theo?
    String name = iterator.next();  // Lấy phần tử tiếp
    if (name.equals("An")) {
        iterator.remove();          // ✅ AN TOÀN: Xóa khi đang duyệt
    }
}
System.out.println(list);  // [Bình, Châu]
```

⚠️ **BẪY NGUY HIỂM:** Xóa phần tử trong for-each → `ConcurrentModificationException`!

```java
List<String> list = new ArrayList<>(List.of("An", "Bình", "Châu"));

// ❌ SAI: Xóa trong for-each → CRASH!
for (String name : list) {
    if (name.equals("An")) {
        list.remove(name);  // ConcurrentModificationException!
    }
}

// ✅ ĐÚNG: Dùng Iterator
Iterator<String> it = list.iterator();
while (it.hasNext()) {
    if (it.next().equals("An")) {
        it.remove();  // OK!
    }
}

// ✅ ĐÚNG: Hoặc dùng removeIf (Java 8+)
list.removeIf(name -> name.equals("An"));
```

### 5.3. ListIterator (Duyệt 2 chiều — chỉ cho List)

```java
List<String> list = new ArrayList<>(List.of("A", "B", "C"));
ListIterator<String> li = list.listIterator();

// Duyệt XUÔI (forward)
while (li.hasNext()) {
    System.out.print(li.next() + " ");  // A B C
}

// Duyệt NGƯỢC (backward)
while (li.hasPrevious()) {
    System.out.print(li.previous() + " ");  // C B A
}

// SỬA khi đang duyệt
li = list.listIterator();
while (li.hasNext()) {
    String item = li.next();
    li.set(item.toLowerCase());  // Đổi thành chữ thường
}
// list = [a, b, c]
```

### 5.4. forEach Method (Java 8+ — Ngắn gọn nhất)

```java
List<String> list = List.of("An", "Bình", "Châu");

// Lambda expression
list.forEach(name -> System.out.println("Xin chào " + name));

// Method reference (ngắn hơn nữa)
list.forEach(System.out::println);
```

---

## 6. Comparable và Comparator (Sắp xếp tùy chỉnh)

### Tại sao cần?

`Collections.sort()` biết cách sắp xếp String (A-Z) và số (nhỏ→lớn). Nhưng nếu bạn có **object tự tạo** (ví dụ: Student), Java **KHÔNG BIẾT** sắp xếp theo tiêu chí nào (tên? tuổi? điểm?).

### 6.1. Comparable (Sắp xếp "mặc định" — implement vào class)

```java
// Student tự biết cách so sánh với Student khác
public class Student implements Comparable<Student> {
    private String name;   // Tên
    private int age;       // Tuổi
    private double gpa;    // Điểm trung bình

    public Student(String name, int age, double gpa) {
        this.name = name;
        this.age = age;
        this.gpa = gpa;
    }

    // compareTo: quy tắc sắp xếp MẶC ĐỊNH
    // Ở đây: sắp xếp theo GPA GIẢM DẦN (điểm cao lên trước)
    @Override
    public int compareTo(Student other) {
        // Trả về:
        //   < 0 nếu this đứng TRƯỚC other
        //   > 0 nếu this đứng SAU other
        //   = 0 nếu BẰNG nhau
        return Double.compare(other.gpa, this.gpa); // Giảm dần
    }

    @Override
    public String toString() {
        return name + " (GPA: " + gpa + ")";
    }

    // Getters...
    public String getName() { return name; }
    public int getAge() { return age; }
    public double getGpa() { return gpa; }
}

// Sử dụng:
List<Student> students = new ArrayList<>();
students.add(new Student("An", 20, 3.5));
students.add(new Student("Bình", 22, 3.8));
students.add(new Student("Châu", 21, 3.2));

Collections.sort(students);  // Dùng compareTo() → sắp theo GPA giảm dần
// Kết quả: Bình (3.8), An (3.5), Châu (3.2)
```

### 6.2. Comparator (Sắp xếp "tùy chỉnh" — tạo bên ngoài class)

Khi bạn muốn sắp xếp theo **nhiều tiêu chí khác nhau**, hoặc không thể sửa class gốc:

```java
import java.util.Comparator;

// ===== Cách 1: Lambda (ngắn gọn nhất — dùng nhiều nhất) =====

// Sắp xếp theo TÊN (A-Z)
students.sort((s1, s2) -> s1.getName().compareTo(s2.getName()));

// Sắp xếp theo TUỔI (tăng dần)
students.sort((s1, s2) -> Integer.compare(s1.getAge(), s2.getAge()));

// ===== Cách 2: Comparator.comparing() — DỄ ĐỌC NHẤT =====

// Theo tên A-Z
students.sort(Comparator.comparing(Student::getName));

// Theo GPA giảm dần
students.sort(Comparator.comparing(Student::getGpa).reversed());

// Theo tên A-Z, nếu trùng tên → theo tuổi tăng dần
students.sort(
    Comparator.comparing(Student::getName)
              .thenComparingInt(Student::getAge)
);

// Theo GPA giảm dần, nếu trùng GPA → theo tên A-Z
students.sort(
    Comparator.comparing(Student::getGpa).reversed()
              .thenComparing(Student::getName)
);

// ===== Xử lý NULL an toàn =====
// nullsFirst: phần tử null đứng ĐẦU
// nullsLast: phần tử null đứng CUỐI
students.sort(Comparator.nullsFirst(
    Comparator.comparing(Student::getName)
));
```

💡 **Mẹo nhớ:**
- **Comparable** = "Tôi TỰ BIẾT cách so sánh" → implement `compareTo()` trong class
- **Comparator** = "NGƯỜI KHÁC chỉ tôi cách so sánh" → tạo bên ngoài class

---

## 7. Sai lầm thường gặp

### Sai lầm 1: Xóa phần tử khi đang duyệt for-each

```java
List<String> list = new ArrayList<>(List.of("A", "B", "C"));

// ❌ CRASH: ConcurrentModificationException
for (String item : list) {
    if (item.equals("B")) {
        list.remove(item);  // KHÔNG ĐƯỢC xóa trong for-each!
    }
}

// ✅ Dùng Iterator
Iterator<String> it = list.iterator();
while (it.hasNext()) {
    if (it.next().equals("B")) {
        it.remove();
    }
}

// ✅ Hoặc removeIf (Java 8+)
list.removeIf(item -> item.equals("B"));
```

### Sai lầm 2: Dùng `==` thay vì `.equals()` cho key trong Map

```java
Map<String, Integer> map = new HashMap<>();
map.put(new String("key"), 100);

// ❌ SAI: Có thể không tìm thấy!
Integer value = map.get(new String("key"));
// Thực ra HashMap dùng .equals() nên OK trong trường hợp này
// Nhưng nếu key là object tùy chỉnh mà không override equals/hashCode → lỗi!
```

⚠️ **Quy tắc:** Nếu dùng object tùy chỉnh làm key trong HashMap/HashSet → **PHẢI override cả `equals()` và `hashCode()`**!

### Sai lầm 3: Thay đổi subList nghĩ là bản sao

```java
List<Integer> original = new ArrayList<>(List.of(1, 2, 3, 4, 5));
List<Integer> sub = original.subList(0, 3);  // [1, 2, 3]

// ⚠️ sub là VIEW, không phải bản sao!
sub.set(0, 99);  // Sửa sub → original CŨNG bị sửa!
System.out.println(original);  // [99, 2, 3, 4, 5] ← Bị thay đổi!

// ✅ Muốn bản sao độc lập:
List<Integer> copy = new ArrayList<>(original.subList(0, 3));
```

### Sai lầm 4: Nhầm lẫn List.of() là mutable

```java
// ❌ List.of() tạo list IMMUTABLE (không thể thay đổi)
List<String> list = List.of("A", "B", "C");
list.add("D");  // UnsupportedOperationException!

// ✅ Muốn list có thể thay đổi:
List<String> mutableList = new ArrayList<>(List.of("A", "B", "C"));
mutableList.add("D");  // OK!
```

---

## 8. Tóm tắt cuối ngày

### Bảng tổng hợp kiến thức

| Khái niệm | Giải thích tiếng Việt | Đặc điểm chính |
|-----------|----------------------|-----------------|
| **Collection** | Khung bộ sưu tập | Interface gốc |
| **List** | Danh sách | Có thứ tự, cho phép trùng |
| **ArrayList** | Mảng tự giãn nở | Nhanh get/set, dùng 90% |
| **LinkedList** | Danh sách liên kết | Nhanh add/remove đầu-cuối |
| **Set** | Tập hợp | KHÔNG trùng lặp |
| **HashSet** | Tập hợp dùng hash | Nhanh nhất, không thứ tự |
| **LinkedHashSet** | Tập hợp giữ thứ tự thêm | Có thứ tự, không trùng |
| **TreeSet** | Tập hợp tự sắp xếp | Tự sort, O(log n) |
| **Map** | Bản đồ key-value | Key unique, tra cứu nhanh |
| **HashMap** | Map dùng hash | Nhanh nhất, dùng nhiều nhất |
| **LinkedHashMap** | Map giữ thứ tự thêm | Có thứ tự, dùng cho cache |
| **TreeMap** | Map tự sắp xếp theo key | Key tự sort |
| **Iterator** | Bộ duyệt | An toàn khi xóa phần tử |
| **Comparable** | Interface "tự so sánh" | Override `compareTo()` |
| **Comparator** | Interface "so sánh bên ngoài" | Sắp xếp linh hoạt |

### 🔥 Câu hỏi phỏng vấn thường gặp

1. **ArrayList vs LinkedList khác nhau thế nào?**
   → ArrayList: truy cập index O(1), thêm/xóa giữa O(n). LinkedList: truy cập O(n), thêm/xóa đầu-cuối O(1).

2. **HashSet vs TreeSet?**
   → HashSet: O(1) nhanh nhất, không thứ tự. TreeSet: O(log n) chậm hơn, tự sắp xếp.

3. **HashMap vs TreeMap?**
   → HashMap: O(1), không thứ tự. TreeMap: O(log n), key sắp xếp.

4. **Làm sao xóa phần tử an toàn khi đang duyệt?**
   → Dùng `Iterator.remove()` hoặc `collection.removeIf()`.

5. **Comparable vs Comparator?**
   → Comparable: sắp xếp mặc định trong class (compareTo). Comparator: sắp xếp tùy chỉnh bên ngoài.

6. **Tại sao cần override equals và hashCode khi dùng HashMap?**
   → HashMap dùng hashCode() để tìm bucket, dùng equals() để so sánh key. Nếu không override → 2 object "giống nhau" sẽ bị coi là khác nhau.

---

## 9. Bài tập thực hành

### Bài 1: Word Counter (Đếm từ)

Đếm số lần xuất hiện của mỗi từ trong đoạn văn.

```java
String text = "the quick brown fox jumps over the lazy dog the fox";
// Output: {the=3, fox=2, quick=1, brown=1, jumps=1, over=1, lazy=1, dog=1}
```

### Bài 2: Remove Duplicates (Xóa trùng lặp)

Xóa phần tử trùng lặp từ List, **giữ nguyên thứ tự**.

```java
List<Integer> list = Arrays.asList(1, 2, 3, 2, 1, 4, 3, 5);
// Output: [1, 2, 3, 4, 5]
```

### Bài 3: First Non-Repeated Character (Ký tự không lặp đầu tiên)

```java
findFirstUnique("swiss");   // 'w'
findFirstUnique("aabbcc");  // null (tất cả đều lặp)
```

### Bài 4: Group Students by Grade (Nhóm học sinh theo hạng)

Nhóm học sinh theo hạng (A ≥ 90, B ≥ 80, C ≥ 70, D ≥ 60, F < 60) dựa trên điểm.

### Bài 5: LRU Cache

Implement LRU Cache (Least Recently Used — loại bỏ phần tử ít dùng nhất) với capacity cố định.

```java
LRUCache<Integer, String> cache = new LRUCache<>(3);
cache.put(1, "Một");
cache.put(2, "Hai");
cache.put(3, "Ba");
cache.get(1);              // Truy cập 1 → 1 trở thành "recently used"
cache.put(4, "Bốn");       // Capacity đầy → loại bỏ 2 (ít dùng nhất)
```

---

## 10. Đáp án tham khảo

<details>
<summary>Bài 1: Word Counter (Click để xem)</summary>

```java
import java.util.*;

public class WordCounter {

    public static Map<String, Integer> countWords(String text) {
        Map<String, Integer> wordCount = new HashMap<>();

        // Tách chuỗi thành mảng từ (viết thường để đồng nhất)
        String[] words = text.toLowerCase().split("\\s+");

        for (String word : words) {
            // merge: nếu key chưa có → đặt value = 1
            //        nếu key đã có → value = value cũ + 1
            wordCount.merge(word, 1, Integer::sum);

            // Cách khác (dài hơn):
            // wordCount.put(word, wordCount.getOrDefault(word, 0) + 1);
        }

        return wordCount;
    }

    public static void main(String[] args) {
        String text = "the quick brown fox jumps over the lazy dog the fox";
        Map<String, Integer> result = countWords(text);

        // In kết quả sắp xếp theo số lần xuất hiện giảm dần
        result.entrySet().stream()
            .sorted(Map.Entry.<String, Integer>comparingByValue().reversed())
            .forEach(entry ->
                System.out.println(entry.getKey() + ": " + entry.getValue()));
    }
}
```
</details>

<details>
<summary>Bài 2: Remove Duplicates (Click để xem)</summary>

```java
import java.util.*;

public class RemoveDuplicates {

    // Cách 1: Dùng LinkedHashSet (giữ thứ tự + loại trùng)
    public static <T> List<T> removeDuplicates(List<T> list) {
        // LinkedHashSet tự động loại trùng + giữ thứ tự thêm vào
        return new ArrayList<>(new LinkedHashSet<>(list));
    }

    // Cách 2: Không dùng Set (thủ công)
    public static <T> List<T> removeDuplicatesManual(List<T> list) {
        List<T> result = new ArrayList<>();
        for (T item : list) {
            if (!result.contains(item)) { // Chỉ thêm nếu chưa có
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
<summary>Bài 3: First Non-Repeated Character (Click để xem)</summary>

```java
import java.util.*;

public class FirstUnique {

    public static Character findFirstUnique(String str) {
        // Dùng LinkedHashMap để GIỮ THỨ TỰ ký tự xuất hiện
        Map<Character, Integer> charCount = new LinkedHashMap<>();

        // Bước 1: Đếm số lần xuất hiện mỗi ký tự
        for (char c : str.toCharArray()) {
            charCount.merge(c, 1, Integer::sum);
        }

        // Bước 2: Tìm ký tự ĐẦU TIÊN có count = 1
        for (Map.Entry<Character, Integer> entry : charCount.entrySet()) {
            if (entry.getValue() == 1) {
                return entry.getKey(); // Trả về ký tự đầu tiên không lặp
            }
        }

        return null; // Tất cả ký tự đều lặp
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
<summary>Bài 5: LRU Cache (Click để xem)</summary>

```java
import java.util.LinkedHashMap;
import java.util.Map;

// Kế thừa LinkedHashMap với accessOrder = true
// → Phần tử ít dùng nhất nằm ĐẦU, vừa dùng nằm CUỐI
public class LRUCache<K, V> extends LinkedHashMap<K, V> {
    private final int capacity; // Sức chứa tối đa

    public LRUCache(int capacity) {
        // accessOrder = true: sắp xếp theo lần truy cập gần nhất
        super(capacity, 0.75f, true);
        this.capacity = capacity;
    }

    // Override: tự động xóa phần tử CŨ NHẤT khi vượt capacity
    @Override
    protected boolean removeEldestEntry(Map.Entry<K, V> eldest) {
        return size() > capacity; // Xóa phần tử đầu nếu quá sức chứa
    }

    public static void main(String[] args) {
        LRUCache<Integer, String> cache = new LRUCache<>(3);

        cache.put(1, "Một");
        cache.put(2, "Hai");
        cache.put(3, "Ba");
        System.out.println(cache);  // {1=Một, 2=Hai, 3=Ba}

        cache.get(1);  // Truy cập 1 → 1 được đưa lên cuối (recently used)
        System.out.println(cache);  // {2=Hai, 3=Ba, 1=Một}

        cache.put(4, "Bốn");  // Quá capacity → loại 2 (ít dùng nhất, ở đầu)
        System.out.println(cache);  // {3=Ba, 1=Một, 4=Bốn}
    }
}
```
</details>

---

## Navigation

- [← Day 6: Strings & Wrappers (Chuỗi & Lớp Bọc)](./day-06-strings-wrappers.md)
- [Day 8: Generics (Kiểu Tổng Quát) →](./day-08-generics.md)
