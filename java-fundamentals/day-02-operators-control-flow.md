# Day 2: Operators & Control Flow (Toán tử & Luồng điều khiển)

## Mục tiêu hôm nay

Sau ngày hôm nay, bạn sẽ:
- Biết dùng các loại toán tử (operators) trong Java
- Viết được câu lệnh điều kiện (if/else, switch) — "nếu... thì..."
- Viết được vòng lặp (for, while, do-while) — "lặp lại cho đến khi..."
- Biết cách dùng mảng (Array) để lưu nhiều giá trị cùng lúc

### Tại sao cần học?

Mọi chương trình đều cần 3 thứ:
1. **Tính toán** → Operators (toán tử)
2. **Ra quyết định** → Conditions (điều kiện) — "Nếu user nhập sai mật khẩu thì báo lỗi"
3. **Lặp lại** → Loops (vòng lặp) — "Duyệt qua tất cả 1000 đơn hàng để tính tổng"

---

## 1. Operators (Toán tử)

### 1.1. Arithmetic Operators (Toán tử số học) — phép tính cơ bản

> 💡 **Ví dụ đời thường**: Giống máy tính bỏ túi — cộng, trừ, nhân, chia.

```java
int a = 10, b = 3;

System.out.println("a + b = " + (a + b));  // 13  — Cộng (addition)
System.out.println("a - b = " + (a - b));  // 7   — Trừ (subtraction)
System.out.println("a * b = " + (a * b));  // 30  — Nhân (multiplication)
System.out.println("a / b = " + (a / b));  // 3   — Chia (division)
System.out.println("a % b = " + (a % b));  // 1   — Chia lấy dư (modulo)
```

> ⚠️ **Bẫy phổ biến nhất: Chia nguyên**

```java
// Khi CẢ HAI đều là int → kết quả cũng là int (CẮT phần thập phân!)
int ketQua1 = 7 / 2;        // = 3  (KHÔNG phải 3.5!)
double ketQua2 = 7 / 2;     // = 3.0 (vẫn sai! vì 7/2 tính xong = 3, rồi mới chuyển sang double)

// ✅ CÁCH SỬA: Ít nhất 1 bên phải là số thực
double ketQua3 = 7.0 / 2;     // = 3.5 ✅
double ketQua4 = (double) 7 / 2;  // = 3.5 ✅ (ép kiểu 7 thành 7.0 trước khi chia)
```

> 💡 **Mẹo nhớ phép chia lấy dư (%)**: `10 % 3 = 1` → chia 10 cho 3 được 3 dư 1. Dùng để kiểm tra chẵn/lẻ: `n % 2 == 0` → số chẵn.

### 1.2. Assignment Operators (Toán tử gán) — viết tắt phép tính

```java
int x = 10;

x += 5;   // Viết tắt của: x = x + 5;  → x = 15
x -= 3;   // Viết tắt của: x = x - 3;  → x = 12
x *= 2;   // Viết tắt của: x = x * 2;  → x = 24
x /= 4;   // Viết tắt của: x = x / 4;  → x = 6
x %= 4;   // Viết tắt của: x = x % 4;  → x = 2
```

> 💡 **Mẹo nhớ**: `x += 5` đọc là "x cộng thêm 5". Viết tắt gọn hơn `x = x + 5`.

### 1.3. Increment/Decrement (Tăng/Giảm 1)

```java
int a = 5;

// ++ tăng thêm 1, -- giảm đi 1
a++;   // a = 6 (tăng 1)
a--;   // a = 5 (giảm 1, quay lại)
```

> ⚠️ **Phân biệt `a++` và `++a`** — đây là câu hỏi phỏng vấn kinh điển!

```java
int a = 5;

// a++ (Post-increment) = "dùng TRƯỚC, tăng SAU"
System.out.println(a++);  // In ra 5 (dùng giá trị cũ), rồi mới tăng a lên 6

// ++a (Pre-increment) = "tăng TRƯỚC, dùng SAU"
System.out.println(++a);  // Tăng a lên 7 trước, rồi in ra 7
```

```
Trạng thái a qua từng bước:
a = 5
a++ → in 5, a thành 6
++a → a thành 7, in 7
```

> 💡 **Mẹo nhớ**: Dấu `++` đứng SAU (`a++`) = dùng SAU. Dấu `++` đứng TRƯỚC (`++a`) = dùng TRƯỚC.

### 1.4. Comparison Operators (Toán tử so sánh)

Kết quả luôn là `true` hoặc `false`.

```java
int a = 10, b = 20;

System.out.println(a == b);  // false — Bằng nhau?
System.out.println(a != b);  // true  — Khác nhau?
System.out.println(a > b);   // false — a lớn hơn b?
System.out.println(a < b);   // true  — a nhỏ hơn b?
System.out.println(a >= b);  // false — a lớn hơn hoặc bằng b?
System.out.println(a <= b);  // true  — a nhỏ hơn hoặc bằng b?
```

> 🔥 **So sánh String — BẪY KINH ĐIỂN**

```java
String s1 = "hello";
String s2 = "hello";
String s3 = new String("hello");

System.out.println(s1 == s2);      // true  — cùng "địa chỉ" trong String pool
System.out.println(s1 == s3);      // false — KHÁC "địa chỉ" (s3 tạo ở nơi khác)
System.out.println(s1.equals(s3)); // true  — SO SÁNH NỘI DUNG → đúng!

// ⚠️ QUY TẮC: LUÔN dùng .equals() khi so sánh String
// == so sánh ĐỊA CHỈ bộ nhớ (reference)
// .equals() so sánh NỘI DUNG (value)
```

> 💡 **Ví dụ đời thường**: `==` giống hỏi "Hai cuốn sách này có phải CÙNG MỘT cuốn không?" (cùng vật thể). `.equals()` giống hỏi "Hai cuốn sách này có NỘI DUNG giống nhau không?" (cùng nội dung dù khác cuốn).

### 1.5. Logical Operators (Toán tử logic) — kết hợp nhiều điều kiện

```java
boolean troi_dep = true;
boolean co_tien = false;

// && (AND) — "VÀ" — cả hai ĐÚNG thì mới ĐÚNG
System.out.println(troi_dep && co_tien);  // false — trời đẹp VÀ có tiền? Không!

// || (OR) — "HOẶC" — một trong hai ĐÚNG thì ĐÚNG
System.out.println(troi_dep || co_tien);  // true — trời đẹp HOẶC có tiền? Có!

// ! (NOT) — "KHÔNG" — đảo ngược
System.out.println(!troi_dep);            // false — KHÔNG trời đẹp? Sai!
```

> 💡 **Bảng tóm tắt**:
> - `&&` (AND): `true && true` = true, còn lại false
> - `||` (OR): `false || false` = false, còn lại true
> - `!` (NOT): `!true` = false, `!false` = true

> 🔥 **Short-circuit evaluation (Đánh giá ngắn mạch)**:

```java
int x = 5;

// && → Nếu vế trái FALSE, Java KHÔNG kiểm tra vế phải (vì kết quả chắc chắn false)
boolean r1 = (x > 10) && (++x > 5);
// x > 10 = false → dừng luôn, KHÔNG chạy ++x
// x vẫn = 5!

// || → Nếu vế trái TRUE, Java KHÔNG kiểm tra vế phải (vì kết quả chắc chắn true)
boolean r2 = (x < 10) || (++x > 5);
// x < 10 = true → dừng luôn, KHÔNG chạy ++x
// x vẫn = 5!
```

### 1.6. Ternary Operator (Toán tử 3 ngôi) — if/else gọn

```java
// Cú pháp: điều_kiện ? giá_trị_nếu_đúng : giá_trị_nếu_sai

int tuoi = 20;
String trangThai = (tuoi >= 18) ? "Người lớn" : "Trẻ em";
System.out.println(trangThai);  // Người lớn

// Tương đương với if/else:
// String trangThai;
// if (tuoi >= 18) {
//     trangThai = "Người lớn";
// } else {
//     trangThai = "Trẻ em";
// }
```

> ⚠️ **Không nên lồng ternary**: `a ? b : c ? d : e` → khó đọc! Dùng if/else cho trường hợp phức tạp.

---

## 2. Conditional Statements (Câu lệnh điều kiện)

### Tại sao cần?

Chương trình cần **ra quyết định**: "Nếu mật khẩu đúng → cho đăng nhập, sai → báo lỗi". Không có điều kiện, chương trình chỉ chạy từ trên xuống dưới không phân nhánh.

### 2.1. if / else if / else

```java
// ① if đơn giản — "Nếu... thì..."
int tuoi = 20;
if (tuoi >= 18) {
    System.out.println("Bạn đủ tuổi bầu cử");
}

// ② if-else — "Nếu... thì..., nếu không thì..."
if (tuoi >= 18) {
    System.out.println("Người lớn");
} else {
    System.out.println("Trẻ em");
}

// ③ if - else if - else — "Nếu A thì..., nếu B thì..., còn lại thì..."
int diem = 85;
if (diem >= 90) {
    System.out.println("Xếp loại: A (Xuất sắc)");
} else if (diem >= 80) {
    System.out.println("Xếp loại: B (Giỏi)");      // ← Chạy dòng này vì 85 >= 80
} else if (diem >= 70) {
    System.out.println("Xếp loại: C (Khá)");
} else if (diem >= 60) {
    System.out.println("Xếp loại: D (Trung bình)");
} else {
    System.out.println("Xếp loại: F (Không đạt)");
}
```

> ⚠️ **Sai lầm thường gặp**:

```java
// ❌ SAI: Quên dấu ngoặc nhọn {} khi có nhiều dòng
if (tuoi >= 18)
    System.out.println("Người lớn");
    System.out.println("Được bầu cử");  // ⚠️ Dòng này LUÔN chạy! Không nằm trong if!

// ✅ ĐÚNG: Luôn dùng {} cho rõ ràng
if (tuoi >= 18) {
    System.out.println("Người lớn");
    System.out.println("Được bầu cử");
}

// ❌ SAI: Dùng = thay vì ==
if (tuoi = 18) {  // Lỗi! Đây là phép GÁN, không phải SO SÁNH
}
if (tuoi == 18) { // ✅ ĐÚNG: == là so sánh
}
```

### 2.2. switch — khi có nhiều trường hợp cụ thể

Khi bạn cần kiểm tra 1 biến với NHIỀU giá trị cụ thể, `switch` gọn hơn nhiều `if/else if`:

```java
int thu = 3; // 1 = Thứ 2, 2 = Thứ 3, ...

switch (thu) {
    case 1:
        System.out.println("Thứ Hai");
        break;     // ← BẮT BUỘC có break! Không có → chạy luôn case tiếp theo
    case 2:
        System.out.println("Thứ Ba");
        break;
    case 3:
        System.out.println("Thứ Tư");  // ← Chạy dòng này
        break;
    case 4:
        System.out.println("Thứ Năm");
        break;
    case 5:
        System.out.println("Thứ Sáu");
        break;
    case 6:
    case 7:        // 2 case dùng chung 1 xử lý
        System.out.println("Cuối tuần!");
        break;
    default:       // Tương tự else — nếu không match case nào
        System.out.println("Ngày không hợp lệ!");
}
```

> ⚠️ **Bẫy kinh điển: QUÊN break**

```java
int x = 1;
switch (x) {
    case 1:
        System.out.println("Một");
        // ❌ QUÊN break → "rơi" xuống case 2!
    case 2:
        System.out.println("Hai");
        break;
}
// Kết quả: In ra CẢ "Một" VÀ "Hai"! (fall-through)
```

### 2.3. Switch Expression (Java 14+) — cách viết hiện đại

```java
int thu = 3;

// Dùng -> thay cho case/break, gọn hơn nhiều
String tenThu = switch (thu) {
    case 1 -> "Thứ Hai";
    case 2 -> "Thứ Ba";
    case 3 -> "Thứ Tư";
    case 4 -> "Thứ Năm";
    case 5 -> "Thứ Sáu";
    case 6, 7 -> "Cuối tuần";   // Nhiều case 1 dòng
    default -> "Không hợp lệ";
};
// Không cần break! Không bị fall-through!

System.out.println(tenThu);  // Thứ Tư
```

---

## 3. Loops (Vòng lặp)

### Tại sao cần vòng lặp?

Thay vì viết 100 dòng `System.out.println()`, bạn viết 1 vòng lặp chạy 100 lần. Vòng lặp = "lặp lại một hành động cho đến khi điều kiện thỏa mãn".

### 3.1. for Loop — biết trước số lần lặp

> 💡 **Ví dụ đời thường**: "Làm 10 bài tập" → biết trước phải làm 10 bài.

```java
// Cú pháp: for (khởi_tạo; điều_kiện; thay_đổi) { ... }
//           for (int i = 0;  i < 5;    i++)     { ... }
//                    ①          ②        ③
// ① Chạy 1 lần đầu: tạo biến i = 0
// ② Kiểm tra mỗi vòng: i < 5 đúng? Đúng → chạy code trong {}, Sai → thoát
// ③ Sau mỗi vòng: tăng i lên 1

for (int i = 0; i < 5; i++) {
    System.out.println("Lần lặp thứ " + i);
}
// Kết quả: 0, 1, 2, 3, 4 (5 lần, từ 0 đến 4)
```

```
Quá trình chạy chi tiết:
┌──────────┬──────────┬───────────┬──────────┐
│ Vòng     │ i = ?    │ i < 5?    │ In ra    │
├──────────┼──────────┼───────────┼──────────┤
│ 1        │ 0        │ true ✅   │ 0        │
│ 2        │ 1        │ true ✅   │ 1        │
│ 3        │ 2        │ true ✅   │ 2        │
│ 4        │ 3        │ true ✅   │ 3        │
│ 5        │ 4        │ true ✅   │ 4        │
│ 6        │ 5        │ false ❌  │ THOÁT    │
└──────────┴──────────┴───────────┴──────────┘
```

```java
// Đếm ngược
for (int i = 5; i > 0; i--) {
    System.out.println(i);  // 5, 4, 3, 2, 1
}

// Bước nhảy 2
for (int i = 0; i <= 10; i += 2) {
    System.out.println(i);  // 0, 2, 4, 6, 8, 10
}
```

### 3.2. while Loop — chưa biết trước số lần lặp

> 💡 **Ví dụ đời thường**: "Ăn cho đến khi no" → không biết trước ăn bao nhiêu bát.

```java
// Kiểm tra điều kiện TRƯỚC, đúng thì mới chạy
int count = 0;
while (count < 5) {         // Còn nhỏ hơn 5 không? Đúng → chạy tiếp
    System.out.println("Count: " + count);
    count++;                  // Nhớ tăng biến, không thì lặp vô tận!
}

// Ví dụ thực tế: Đọc lệnh từ user cho đến khi gõ "quit"
Scanner scanner = new Scanner(System.in);
String lenh = "";
while (!lenh.equals("quit")) {    // Chưa gõ "quit" → tiếp tục
    System.out.print("Nhập lệnh: ");
    lenh = scanner.nextLine();
    System.out.println("Bạn đã gõ: " + lenh);
}
System.out.println("Thoát chương trình!");
```

> ⚠️ **Sai lầm: Vòng lặp vô tận (infinite loop)**

```java
// ❌ SAI: Quên tăng biến đếm
int i = 0;
while (i < 5) {
    System.out.println(i);
    // Quên i++ → i mãi = 0 → vòng lặp chạy mãi không dừng!
}
// Nhấn Ctrl + C trong terminal hoặc nút Stop trong IDE để dừng
```

### 3.3. do-while Loop — chạy ít nhất 1 lần rồi mới kiểm tra

> 💡 **Ví dụ đời thường**: "Nếm thử món ăn, rồi mới quyết định ăn tiếp hay không" → ít nhất nếm 1 lần.

```java
// Chạy code TRƯỚC, kiểm tra điều kiện SAU
Scanner scanner = new Scanner(System.in);
int luaChon;

do {
    System.out.println("=== MENU ===");
    System.out.println("1. Xem danh sách");
    System.out.println("2. Thêm mới");
    System.out.println("0. Thoát");
    System.out.print("Chọn: ");
    luaChon = scanner.nextInt();

    switch (luaChon) {
        case 1 -> System.out.println("→ Đang hiển thị danh sách...");
        case 2 -> System.out.println("→ Đang thêm mới...");
        case 0 -> System.out.println("→ Tạm biệt!");
        default -> System.out.println("→ Lựa chọn không hợp lệ!");
    }
    System.out.println();
} while (luaChon != 0);  // Lặp cho đến khi chọn 0
```

> 💡 **Khi nào dùng loại nào?**
> - `for`: Biết trước số lần lặp → "Lặp 10 lần", "Duyệt mảng 100 phần tử"
> - `while`: Chưa biết số lần, kiểm tra điều kiện trước → "Đọc file cho đến hết"
> - `do-while`: Chạy ít nhất 1 lần → "Hiện menu, hỏi user chọn, lặp lại"

### 3.4. for-each (Enhanced for) — duyệt mảng/danh sách

```java
// Cú pháp: for (kiểu phần_tử : mảng) { ... }

int[] diemSo = {8, 9, 7, 10, 6};
for (int diem : diemSo) {
    System.out.println("Điểm: " + diem);
}
// Tương đương:
// for (int i = 0; i < diemSo.length; i++) {
//     int diem = diemSo[i];
//     System.out.println("Điểm: " + diem);
// }

String[] monAn = {"Phở", "Bún chả", "Bánh mì"};
for (String mon : monAn) {
    System.out.println("Món: " + mon);
}
```

> 💡 **Khi nào dùng for-each?** Khi bạn chỉ cần **đọc** từng phần tử, không cần biết index (vị trí). Nếu cần index hoặc muốn thay đổi phần tử → dùng for thường.

### 3.5. break và continue

```java
// break = "THOÁT" khỏi vòng lặp ngay lập tức
for (int i = 0; i < 10; i++) {
    if (i == 5) {
        break;  // Gặp 5 → thoát luôn, không chạy 6, 7, 8, 9
    }
    System.out.println(i);
}
// Kết quả: 0, 1, 2, 3, 4

// continue = "BỎ QUA" vòng hiện tại, nhảy sang vòng tiếp theo
for (int i = 0; i < 10; i++) {
    if (i % 2 == 0) {
        continue;  // Số chẵn → bỏ qua, không in
    }
    System.out.println(i);
}
// Kết quả: 1, 3, 5, 7, 9 (chỉ in số lẻ)
```

> 💡 **Mẹo nhớ**: `break` = đập vỡ vòng lặp (thoát). `continue` = tiếp tục sang vòng kế.

---

## 4. Arrays (Mảng)

### Tại sao cần mảng?

Nếu cần lưu điểm của 100 học sinh, bạn sẽ tạo 100 biến riêng lẻ? Không! Mảng = **"dãy hộp"** có đánh số thứ tự, chứa cùng kiểu dữ liệu.

```
Mảng diem[] gồm 5 phần tử:
┌──────┬──────┬──────┬──────┬──────┐
│  8   │  9   │  7   │  10  │  6   │
├──────┼──────┼──────┼──────┼──────┤
│ [0]  │ [1]  │ [2]  │ [3]  │ [4]  │  ← Index (chỉ số) bắt đầu từ 0!
└──────┴──────┴──────┴──────┴──────┘
```

### 4.1. Khai báo và khởi tạo mảng

```java
// Cách 1: Tạo mảng rỗng (chưa có giá trị)
int[] diem = new int[5];  // Mảng 5 ô, mỗi ô mặc định = 0

// Cách 2: Tạo mảng với giá trị sẵn
int[] diem2 = {8, 9, 7, 10, 6};  // 5 phần tử, có giá trị luôn

// Cách 3: Tạo tường minh hơn
int[] diem3 = new int[]{8, 9, 7, 10, 6};
```

**Giá trị mặc định khi tạo mảng rỗng:**

| Kiểu | Giá trị mặc định |
|------|-----------------|
| `int[]` | 0 |
| `double[]` | 0.0 |
| `boolean[]` | false |
| `String[]` | null (chưa có gì) |

### 4.2. Truy cập và thay đổi phần tử

```java
int[] diem = {8, 9, 7, 10, 6};

// Đọc phần tử (index bắt đầu từ 0!)
System.out.println(diem[0]);  // 8  — phần tử đầu tiên
System.out.println(diem[2]);  // 7  — phần tử thứ 3
System.out.println(diem[4]);  // 6  — phần tử cuối cùng

// Thay đổi phần tử
diem[1] = 10;  // Sửa phần tử thứ 2 từ 9 → 10

// Lấy kích thước mảng
System.out.println(diem.length);  // 5

// ⚠️ Truy cập ngoài phạm vi → LỖI RUNTIME!
// System.out.println(diem[5]);   // ❌ ArrayIndexOutOfBoundsException!
// System.out.println(diem[-1]);  // ❌ Lỗi!
```

> 🔥 **Lỗi phổ biến nhất với mảng**: `ArrayIndexOutOfBoundsException` — truy cập index không tồn tại. Mảng 5 phần tử → index hợp lệ là 0, 1, 2, 3, 4. Index 5 → lỗi!

### 4.3. Duyệt mảng

```java
int[] diem = {8, 9, 7, 10, 6};

// Cách 1: for thường (khi cần index)
for (int i = 0; i < diem.length; i++) {
    System.out.println("Học sinh " + (i + 1) + ": " + diem[i]);
}

// Cách 2: for-each (khi chỉ cần giá trị, gọn hơn)
for (int d : diem) {
    System.out.println("Điểm: " + d);
}
```

### 4.4. Mảng 2 chiều — "bảng" có hàng và cột

> 💡 **Ví dụ đời thường**: Bảng điểm lớp học — hàng = học sinh, cột = môn học.

```java
// Tạo bảng 3 hàng × 3 cột
int[][] bangDiem = {
    {8, 9, 7},    // Hàng 0: học sinh 1
    {6, 8, 9},    // Hàng 1: học sinh 2
    {10, 7, 8}    // Hàng 2: học sinh 3
};

// Truy cập: bangDiem[hàng][cột]
System.out.println(bangDiem[0][0]);  // 8  (HS1, Môn 1)
System.out.println(bangDiem[1][2]);  // 9  (HS2, Môn 3)

// Duyệt mảng 2 chiều
for (int hang = 0; hang < bangDiem.length; hang++) {
    for (int cot = 0; cot < bangDiem[hang].length; cot++) {
        System.out.print(bangDiem[hang][cot] + "\t");
    }
    System.out.println();  // Xuống dòng sau mỗi hàng
}
// Kết quả:
// 8    9    7
// 6    8    9
// 10   7    8
```

### 4.5. Arrays utilities — công cụ xử lý mảng

```java
import java.util.Arrays;  // Phải import!

int[] so = {5, 2, 8, 1, 9};

// In mảng đẹp (thay vì in địa chỉ bộ nhớ)
System.out.println(Arrays.toString(so));  // [5, 2, 8, 1, 9]

// Sắp xếp tăng dần
Arrays.sort(so);
System.out.println(Arrays.toString(so));  // [1, 2, 5, 8, 9]

// Tìm kiếm (mảng PHẢI được sort trước!)
int viTri = Arrays.binarySearch(so, 5);
System.out.println("Số 5 ở vị trí: " + viTri);  // 2

// Điền cùng 1 giá trị
int[] arr = new int[5];
Arrays.fill(arr, 10);
System.out.println(Arrays.toString(arr));  // [10, 10, 10, 10, 10]

// Copy mảng
int[] banSao = Arrays.copyOf(so, 3);  // Copy 3 phần tử đầu
System.out.println(Arrays.toString(banSao));  // [1, 2, 5]

// So sánh 2 mảng
int[] a = {1, 2, 3};
int[] b = {1, 2, 3};
System.out.println(a == b);              // false (khác địa chỉ)
System.out.println(Arrays.equals(a, b)); // true  (cùng nội dung)
```

> ⚠️ **Không dùng `==` để so sánh mảng!** Giống String, phải dùng `Arrays.equals()`.

---

## 5. Tóm tắt cuối ngày

| Khái niệm | Giải thích | Ví dụ |
|-----------|-----------|-------|
| Arithmetic operators | Phép tính: +, -, *, /, % | `10 % 3` → 1 |
| `==` vs `.equals()` | `==` so sánh địa chỉ, `.equals()` so sánh nội dung | String luôn dùng `.equals()` |
| `&&`, `\|\|`, `!` | AND, OR, NOT logic | `true && false` → false |
| `a++` vs `++a` | Dùng trước tăng sau / Tăng trước dùng sau | Câu hỏi phỏng vấn kinh điển |
| if / else if / else | Ra quyết định theo điều kiện | Xếp loại điểm |
| switch | Kiểm tra nhiều giá trị cụ thể | Menu, ngày trong tuần |
| for | Lặp biết trước số lần | `for (int i = 0; i < 10; i++)` |
| while | Lặp chưa biết số lần, kiểm tra trước | Đọc input đến "quit" |
| do-while | Lặp ít nhất 1 lần | Menu chương trình |
| for-each | Duyệt mảng gọn | `for (int x : arr)` |
| break / continue | Thoát vòng lặp / Bỏ qua vòng hiện tại | |
| Array | Dãy phần tử cùng kiểu, index từ 0 | `int[] a = {1,2,3};` |

---

## 6. Bài tập thực hành

### Bài 1: Máy tính đơn giản
Viết chương trình máy tính:
- Nhập 2 số và phép tính (+, -, *, /, %)
- Dùng switch để xử lý
- Xử lý trường hợp chia cho 0

```
Nhập số thứ nhất: 10
Nhập phép tính (+, -, *, /, %): /
Nhập số thứ hai: 3
Kết quả: 10 / 3 = 3.33
```

---

### Bài 2: Kiểm tra số nguyên tố
Viết chương trình kiểm tra số nguyên tố (số chỉ chia hết cho 1 và chính nó) và in các số nguyên tố từ 1 đến n.

> 💡 **Gợi ý**: Số nguyên tố là số lớn hơn 1, không chia hết cho bất kỳ số nào từ 2 đến √n.

```
Nhập n: 20
Số nguyên tố từ 1 đến 20:
2 3 5 7 11 13 17 19
Tổng: 8 số nguyên tố
```

---

### Bài 3: Bảng cửu chương
In bảng cửu chương từ 2 đến 9.

---

### Bài 4: Tam giác sao
Vẽ tam giác sao với n dòng:

```
Nhập n: 5

*
**
***
****
*****
```

Thử thêm: tam giác ngược, kim tự tháp, hình thoi.

---

### Bài 5: Thao tác mảng
Viết chương trình:
1. Nhập mảng n phần tử
2. Tìm min, max
3. Tính tổng, trung bình
4. Đảo ngược mảng
5. Sắp xếp (không dùng `Arrays.sort`, tự viết)

---

### Bài 6: FizzBuzz
In số từ 1 đến n:
- Chia hết cho 3 → "Fizz"
- Chia hết cho 5 → "Buzz"
- Chia hết cho cả 3 và 5 → "FizzBuzz"
- Còn lại → in số

> 🔥 Đây là bài phỏng vấn kinh điển!

---

### Bài 7: Đoán số
Tạo game đoán số random 1-100, gợi ý "Cao quá" / "Thấp quá", đếm số lần đoán.

---

## 7. Đáp án tham khảo

> ⚠️ **Tự làm trước ít nhất 15 phút trước khi xem đáp án!**

<details>
<summary>Bài 1: Máy tính (click để mở)</summary>

```java
import java.util.Scanner;

public class MayTinh {
    public static void main(String[] args) {
        Scanner scanner = new Scanner(System.in);

        System.out.print("Nhập số thứ nhất: ");
        double so1 = scanner.nextDouble();

        System.out.print("Nhập phép tính (+, -, *, /, %): ");
        char phepTinh = scanner.next().charAt(0);  // Đọc 1 ký tự đầu tiên

        System.out.print("Nhập số thứ hai: ");
        double so2 = scanner.nextDouble();

        double ketQua;
        switch (phepTinh) {
            case '+':
                ketQua = so1 + so2;
                break;
            case '-':
                ketQua = so1 - so2;
                break;
            case '*':
                ketQua = so1 * so2;
                break;
            case '/':
                if (so2 == 0) {
                    System.out.println("Lỗi: Không thể chia cho 0!");
                    return;  // Thoát chương trình
                }
                ketQua = so1 / so2;
                break;
            case '%':
                if (so2 == 0) {
                    System.out.println("Lỗi: Không thể chia cho 0!");
                    return;
                }
                ketQua = so1 % so2;
                break;
            default:
                System.out.println("Phép tính không hợp lệ!");
                return;
        }

        System.out.printf("Kết quả: %.2f %c %.2f = %.2f%n", so1, phepTinh, so2, ketQua);
        scanner.close();
    }
}
```
</details>

<details>
<summary>Bài 2: Số nguyên tố (click để mở)</summary>

```java
import java.util.Scanner;

public class SoNguyenTo {
    public static void main(String[] args) {
        Scanner scanner = new Scanner(System.in);

        System.out.print("Nhập n: ");
        int n = scanner.nextInt();

        System.out.println("Số nguyên tố từ 1 đến " + n + ":");

        int dem = 0;
        for (int so = 2; so <= n; so++) {
            if (laSoNguyenTo(so)) {
                System.out.print(so + " ");
                dem++;
            }
        }

        System.out.println("\nTổng: " + dem + " số nguyên tố");
        scanner.close();
    }

    // Hàm kiểm tra số nguyên tố
    public static boolean laSoNguyenTo(int so) {
        if (so < 2) return false;      // 0 và 1 không phải số nguyên tố
        if (so == 2) return true;       // 2 là số nguyên tố duy nhất là số chẵn
        if (so % 2 == 0) return false;  // Số chẵn > 2 không phải nguyên tố

        // Chỉ cần kiểm tra đến căn bậc 2 của so
        for (int i = 3; i <= Math.sqrt(so); i += 2) {
            if (so % i == 0) {
                return false;  // Chia hết → không phải nguyên tố
            }
        }
        return true;
    }
}
```
</details>

<details>
<summary>Bài 6: FizzBuzz (click để mở)</summary>

```java
public class FizzBuzz {
    public static void main(String[] args) {
        for (int i = 1; i <= 100; i++) {
            if (i % 3 == 0 && i % 5 == 0) {
                System.out.println("FizzBuzz");  // Chia hết cả 3 VÀ 5 → kiểm tra TRƯỚC!
            } else if (i % 3 == 0) {
                System.out.println("Fizz");
            } else if (i % 5 == 0) {
                System.out.println("Buzz");
            } else {
                System.out.println(i);
            }
        }
        // ⚠️ THỨ TỰ QUAN TRỌNG: phải kiểm tra "cả 3 và 5" TRƯỚC "chỉ 3" và "chỉ 5"!
    }
}
```
</details>

<details>
<summary>Bài 7: Đoán số (click để mở)</summary>

```java
import java.util.Random;
import java.util.Scanner;

public class DoanSo {
    public static void main(String[] args) {
        Random random = new Random();
        Scanner scanner = new Scanner(System.in);

        int soBiMat = random.nextInt(100) + 1;  // Random từ 1 đến 100
        int doanSo;
        int soLanDoan = 0;

        System.out.println("Tôi đang nghĩ 1 số từ 1 đến 100. Hãy đoán thử!");

        do {
            System.out.print("Bạn đoán: ");
            doanSo = scanner.nextInt();
            soLanDoan++;

            if (doanSo < soBiMat) {
                System.out.println("Thấp quá! ⬆️");
            } else if (doanSo > soBiMat) {
                System.out.println("Cao quá! ⬇️");
            } else {
                System.out.println("Chính xác! Bạn đoán đúng sau " + soLanDoan + " lần!");
            }
        } while (doanSo != soBiMat);

        scanner.close();
    }
}
```
</details>

---

## Navigation

- [← Day 1: Setup & Syntax](./day-01-setup-syntax.md)
- [Day 3: OOP Basics →](./day-03-oop-basics.md)
