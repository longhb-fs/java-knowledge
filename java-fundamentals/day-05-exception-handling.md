# Day 5: Exception Handling (Xử Lý Ngoại Lệ / Xử Lý Lỗi)

## Mục tiêu hôm nay

Sau khi học xong Day 5, bạn sẽ:
- Hiểu **Exception** (ngoại lệ) là gì và tại sao cần xử lý
- Phân biệt **Checked Exception** (lỗi bắt buộc xử lý) và **Unchecked Exception** (lỗi không bắt buộc)
- Sử dụng **try-catch-finally** để "bắt lỗi" và xử lý
- Phân biệt **throw** (ném lỗi) và **throws** (khai báo lỗi)
- Tạo **Custom Exception** (lỗi tùy chỉnh riêng)
- Nắm được **best practices** (nguyên tắc tốt nhất) khi xử lý lỗi

---

## Tại sao cần học Exception Handling?

### Ví dụ đời thường

Bạn lái xe trên đường. Bạn CÓ THỂ gặp:
- **Xe hết xăng** → Bạn cần biết cách xử lý (đổ xăng, gọi cứu hộ)
- **Lốp xe thủng** → Bạn cần biết cách thay lốp dự phòng
- **Tai nạn nghiêm trọng** → Bạn KHÔNG THỂ tự xử lý, cần gọi cứu thương

Trong lập trình cũng vậy:

```
Chương trình chạy bình thường
        ↓
   Gặp tình huống bất thường (Exception)
        ↓
   ┌─────────────────────────────────┐
   │  Có xử lý lỗi?                 │
   │  ├── CÓ  → Chương trình tiếp   │
   │  │         tục chạy bình thường │
   │  └── KHÔNG → Chương trình CRASH │
   │              (dừng đột ngột)    │
   └─────────────────────────────────┘
```

### Nếu không xử lý lỗi thì sao?

```java
public class NoExceptionHandling {
    public static void main(String[] args) {
        System.out.println("Bước 1: Bắt đầu chương trình");

        // Chia cho 0 → Lỗi ArithmeticException!
        int result = 10 / 0;  // ← CRASH tại đây!

        // Dòng này KHÔNG BAO GIỜ được chạy
        System.out.println("Bước 2: Kết quả = " + result);
        System.out.println("Bước 3: Kết thúc chương trình");
    }
}
// Output:
// Bước 1: Bắt đầu chương trình
// Exception in thread "main" java.lang.ArithmeticException: / by zero
//     at NoExceptionHandling.main(NoExceptionHandling.java:5)
```

**Kết quả**: Chương trình DỪNG ĐỘT NGỘT ở dòng lỗi. "Bước 2" và "Bước 3" không bao giờ chạy!

### Nếu CÓ xử lý lỗi

```java
public class WithExceptionHandling {
    public static void main(String[] args) {
        System.out.println("Bước 1: Bắt đầu chương trình");

        try {
            // Thử chạy code có thể lỗi
            int result = 10 / 0;
            System.out.println("Kết quả = " + result);
        } catch (ArithmeticException e) {
            // Bắt lỗi và xử lý
            System.out.println("Bước 2: Lỗi chia cho 0! Bỏ qua và tiếp tục.");
        }

        // Chương trình VẪN TIẾP TỤC chạy!
        System.out.println("Bước 3: Kết thúc chương trình");
    }
}
// Output:
// Bước 1: Bắt đầu chương trình
// Bước 2: Lỗi chia cho 0! Bỏ qua và tiếp tục.
// Bước 3: Kết thúc chương trình
```

**Kết quả**: Chương trình BẮT được lỗi, xử lý, rồi TIẾP TỤC chạy bình thường!

---

## 1. Exception Hierarchy (Cây phân cấp ngoại lệ)

### 🔥 Tại sao cần biết cây phân cấp?

Vì mỗi loại lỗi có **cách xử lý khác nhau**. Giống như bệnh viện phân loại bệnh nhân:
- **Bệnh nhẹ** (Unchecked) → Tự mua thuốc uống cũng được
- **Bệnh nặng** (Checked) → BẮT BUỘC phải đi khám bác sĩ
- **Bệnh nguy kịch** (Error) → KHÔNG TỰ chữa được, cần hệ thống y tế xử lý

```
Throwable (Gốc - tổ tiên của mọi loại lỗi)
│
├── Error (Lỗi nghiêm trọng - KHÔNG NÊN bắt)
│   │   → Giống như "nhà sập" - bạn không thể tự sửa
│   │
│   ├── OutOfMemoryError      → Hết bộ nhớ RAM
│   ├── StackOverflowError    → Gọi đệ quy quá sâu
│   └── VirtualMachineError   → Máy ảo Java lỗi
│
└── Exception (Ngoại lệ - CÓ THỂ bắt và xử lý)
    │
    ├── RuntimeException (Unchecked - KHÔNG bắt buộc xử lý)
    │   │   → Giống như "vấp ngã" - do code viết sai
    │   │   → Compiler KHÔNG ép bạn phải xử lý
    │   │
    │   ├── NullPointerException          → Gọi method trên biến null
    │   ├── ArrayIndexOutOfBoundsException → Truy cập index vượt mảng
    │   ├── ArithmeticException           → Chia cho 0
    │   ├── ClassCastException            → Ép kiểu sai
    │   ├── IllegalArgumentException      → Tham số không hợp lệ
    │   └── NumberFormatException         → Chuyển chuỗi→số sai format
    │
    └── Checked Exception (BẮT BUỘC xử lý)
        │   → Giống như "tai nạn giao thông" - bạn PHẢI có bảo hiểm
        │   → Compiler BẮT BUỘC bạn phải try-catch hoặc throws
        │
        ├── IOException            → Lỗi đọc/ghi file
        ├── FileNotFoundException  → Không tìm thấy file
        ├── SQLException           → Lỗi truy vấn database
        └── ClassNotFoundException → Không tìm thấy class
```

### 🔥 Checked vs Unchecked — Bảng so sánh

| Tiêu chí | Checked Exception (Bắt buộc xử lý) | Unchecked Exception (Không bắt buộc) |
|-----------|--------------------------------------|--------------------------------------|
| **Compiler kiểm tra?** | ✅ CÓ — Không xử lý → code không compile được | ❌ KHÔNG — Compiler không ép |
| **Bắt buộc try-catch hoặc throws?** | ✅ BẮT BUỘC | ❌ Tùy bạn |
| **Kế thừa từ đâu?** | `Exception` (trực tiếp) | `RuntimeException` |
| **Nguyên nhân thường gặp** | Yếu tố BÊN NGOÀI (file, network, DB) | Do LỖI CODE (null, sai index, chia 0) |
| **Cách phòng tránh** | Không thể tránh 100% → phải catch | Viết code cẩn thận hơn |
| **Ví dụ** | `IOException`, `SQLException` | `NullPointerException`, `ArithmeticException` |

### Ví dụ minh họa sự khác biệt

```java
import java.io.*;

public class CheckedVsUnchecked {

    // ====== CHECKED EXCEPTION ======
    // Compiler BẮT BUỘC bạn phải xử lý IOException
    public void readFile() {
        // ❌ KHÔNG COMPILE được! Thiếu try-catch hoặc throws
        // BufferedReader reader = new BufferedReader(new FileReader("data.txt"));
        // String line = reader.readLine();

        // ✅ Cách 1: try-catch
        try {
            BufferedReader reader = new BufferedReader(new FileReader("data.txt"));
            String line = reader.readLine();
        } catch (IOException e) {
            System.out.println("Lỗi đọc file: " + e.getMessage());
        }
    }

    // ✅ Cách 2: throws (khai báo "tôi không xử lý, người gọi tự lo")
    public void readFile2() throws IOException {
        BufferedReader reader = new BufferedReader(new FileReader("data.txt"));
        String line = reader.readLine();
    }

    // ====== UNCHECKED EXCEPTION ======
    // Compiler KHÔNG ép bạn phải xử lý
    public void divideNumbers() {
        // Code này compile bình thường, nhưng CRASH lúc chạy
        int result = 10 / 0;  // ArithmeticException lúc runtime!
    }
}
```

💡 **Mẹo nhớ:**
- **Checked** = "Compiler **check** (kiểm tra) bạn" → BẮT BUỘC xử lý
- **Unchecked** = "Compiler **KHÔNG check**" → Bạn tự chịu trách nhiệm

---

## 2. try-catch-finally (Thử - Bắt lỗi - Dọn dẹp)

### Tại sao cần học try-catch-finally?

Đây là cú pháp **cơ bản nhất** để xử lý lỗi trong Java. Tất cả các dự án đều dùng.

### Ví dụ đời thường

```
try    = "THỬ" làm điều gì đó (có thể thất bại)
catch  = "BẮT" lỗi nếu xảy ra (xử lý tình huống)
finally = "LUÔN LUÔN" dọn dẹp (dù thành công hay thất bại)
```

Giống như nấu ăn:
- **try**: Thử nấu món mới
- **catch**: Nếu cháy → dập lửa, gọi đội cứu hỏa
- **finally**: DÙ GÌ ĐI NỮA → tắt bếp, rửa nồi (dọn dẹp)

### 2.1. Basic try-catch (Cơ bản)

```java
public class TryCatchBasic {
    public static void main(String[] args) {

        // ===== VÍ DỤ 1: Bắt lỗi chia cho 0 =====
        try {
            // try = "thử chạy" khối code này
            int result = 10 / 0;  // ← Lỗi ArithmeticException xảy ra ở đây!
            // Dòng dưới KHÔNG chạy vì lỗi đã xảy ra ở trên
            System.out.println("Kết quả: " + result);

        } catch (ArithmeticException e) {
            // catch = "bắt" lỗi ArithmeticException
            // Biến 'e' chứa thông tin về lỗi
            System.out.println("Lỗi: Không thể chia cho 0!");
            System.out.println("Chi tiết: " + e.getMessage());
            // Output: Chi tiết: / by zero
        }

        // Chương trình VẪN tiếp tục chạy sau try-catch!
        System.out.println("Chương trình vẫn chạy bình thường!");
    }
}
```

**Luồng chạy khi có lỗi:**

```
try {
    Dòng 1 → Chạy ✅
    Dòng 2 → LỖI! ❌ → Nhảy xuống catch ngay lập tức
    Dòng 3 → KHÔNG chạy (bị bỏ qua)
} catch (...) {
    Xử lý lỗi → Chạy ✅
}
Code tiếp theo → Chạy ✅
```

**Luồng chạy khi KHÔNG có lỗi:**

```
try {
    Dòng 1 → Chạy ✅
    Dòng 2 → Chạy ✅
    Dòng 3 → Chạy ✅
} catch (...) {
    KHÔNG chạy (vì không có lỗi)
}
Code tiếp theo → Chạy ✅
```

### 2.2. Multiple catch blocks (Bắt nhiều loại lỗi)

Một khối try có thể gặp NHIỀU loại lỗi khác nhau. Bạn có thể viết nhiều `catch` để xử lý từng loại riêng.

```java
public class MultipleCatch {
    public static void processData(String[] args) {
        try {
            // Bước 1: Chuyển chuỗi thành số
            // → Có thể lỗi NumberFormatException (nếu chuỗi không phải số)
            int index = Integer.parseInt(args[0]);

            // Bước 2: Lấy phần tử trong mảng
            // → Có thể lỗi ArrayIndexOutOfBoundsException (nếu index vượt mảng)
            int[] numbers = {1, 2, 3};
            int value = numbers[index];

            // Bước 3: Chia
            // → Có thể lỗi ArithmeticException (nếu value = 0)
            int result = 100 / value;

            System.out.println("Kết quả: " + result);

        } catch (ArrayIndexOutOfBoundsException e) {
            // Bắt lỗi: index vượt quá kích thước mảng
            System.out.println("Lỗi: Index ngoài phạm vi mảng!");

        } catch (NumberFormatException e) {
            // Bắt lỗi: chuỗi không phải số
            System.out.println("Lỗi: Chuỗi nhập vào không phải số!");

        } catch (ArithmeticException e) {
            // Bắt lỗi: chia cho 0
            System.out.println("Lỗi: Không thể chia cho 0!");

        } catch (Exception e) {
            // Catch-all: bắt TẤT CẢ lỗi còn lại
            // ⚠️ PHẢI đặt CUỐI CÙNG (vì Exception là cha của tất cả)
            System.out.println("Lỗi không xác định: " + e.getMessage());
        }
    }
}
```

⚠️ **Quy tắc quan trọng:** Các catch phải đi từ **cụ thể → tổng quát** (con → cha)

```java
// ❌ SAI: Exception (cha) đặt trước con → Compile Error!
try {
    // code
} catch (Exception e) {           // ← Cha bắt hết rồi
    System.out.println("Error");
} catch (ArithmeticException e) {  // ← Con không bao giờ được chạy!
    System.out.println("Arithmetic error");
}

// ✅ ĐÚNG: Cụ thể (con) trước, tổng quát (cha) sau
try {
    // code
} catch (ArithmeticException e) {  // ← Con (cụ thể) trước
    System.out.println("Arithmetic error");
} catch (Exception e) {            // ← Cha (tổng quát) cuối
    System.out.println("Other error");
}
```

### 2.3. Multi-catch (Bắt nhiều lỗi trong 1 catch — Java 7+)

Nếu nhiều loại lỗi xử lý **giống nhau**, bạn có thể gom lại bằng dấu `|` (pipe):

```java
try {
    // code có thể gây IOException HOẶC SQLException
    readFromDatabaseAndFile();

} catch (IOException | SQLException e) {
    // Bắt CẢ HAI loại lỗi trong 1 catch
    // Xử lý giống nhau: log lỗi và thông báo
    System.out.println("Lỗi truy xuất dữ liệu: " + e.getMessage());

} catch (Exception e) {
    // Bắt các lỗi khác
    System.out.println("Lỗi khác: " + e.getMessage());
}
```

⚠️ **Lưu ý:** Các exception trong multi-catch KHÔNG ĐƯỢC có quan hệ cha-con:
```java
// ❌ SAI: FileNotFoundException là con của IOException
catch (FileNotFoundException | IOException e) { }

// ✅ ĐÚNG: IOException và SQLException không có quan hệ cha-con
catch (IOException | SQLException e) { }
```

### 2.4. finally block (Khối "luôn luôn" chạy)

`finally` **LUÔN LUÔN** chạy, dù có lỗi hay không. Dùng để **dọn dẹp tài nguyên** (đóng file, đóng kết nối database...).

```java
import java.io.*;

public class FinallyDemo {
    public static void main(String[] args) {
        FileInputStream fis = null; // Luồng đọc file

        try {
            // Mở file để đọc
            fis = new FileInputStream("data.txt");
            int data = fis.read(); // Đọc 1 byte
            System.out.println("Đọc được: " + data);

        } catch (FileNotFoundException e) {
            // File không tồn tại
            System.out.println("Không tìm thấy file!");

        } catch (IOException e) {
            // Lỗi khi đọc file
            System.out.println("Lỗi đọc file!");

        } finally {
            // finally LUÔN chạy → đóng file (dọn dẹp)
            if (fis != null) {
                try {
                    fis.close(); // Đóng file
                } catch (IOException e) {
                    System.out.println("Lỗi đóng file!");
                }
            }
            System.out.println("Dọn dẹp xong!");
        }
    }
}
```

**Khi nào finally chạy?**

| Tình huống | try chạy? | catch chạy? | finally chạy? |
|------------|-----------|-------------|----------------|
| Không có lỗi | ✅ Chạy hết | ❌ Không | ✅ LUÔN chạy |
| Có lỗi + catch đúng | ✅ Chạy đến dòng lỗi | ✅ Chạy | ✅ LUÔN chạy |
| Có lỗi + catch sai | ✅ Chạy đến dòng lỗi | ❌ Không bắt được | ✅ LUÔN chạy (rồi mới crash) |
| Có return trong try | ✅ Chạy đến return | ❌ Không | ✅ LUÔN chạy (trước return) |

💡 **Mẹo nhớ:** `finally` chỉ KHÔNG chạy khi gọi `System.exit()` hoặc máy bị tắt nguồn.

### 2.5. try-with-resources (Tự động đóng tài nguyên — Java 7+)

⚠️ **Vấn đề với finally**: Viết code đóng file trong finally RẤT DÀI và XẤU (xem ví dụ 2.4 ở trên).

✅ **Giải pháp**: `try-with-resources` — Java tự động đóng tài nguyên cho bạn!

```java
import java.io.*;

public class TryWithResources {
    public static void main(String[] args) {

        // ===== CÁCH CŨ (trước Java 7): Phải tự đóng file =====
        // → Dài dòng, dễ quên đóng → rò rỉ tài nguyên (resource leak)
        FileInputStream fis = null;
        try {
            fis = new FileInputStream("data.txt");
            // ... đọc file
        } catch (IOException e) {
            e.printStackTrace();
        } finally {
            if (fis != null) {
                try { fis.close(); } catch (IOException e) { /* ignore */ }
            }
        }

        // ===== CÁCH MỚI (Java 7+): try-with-resources =====
        // → Ngắn gọn, Java TỰ ĐỘNG đóng file khi ra khỏi try
        try (FileInputStream fis2 = new FileInputStream("data.txt");
             BufferedReader reader = new BufferedReader(new InputStreamReader(fis2))) {

            // Đọc file từng dòng
            String line;
            while ((line = reader.readLine()) != null) {
                System.out.println(line);
            }

        } catch (FileNotFoundException e) {
            System.out.println("Không tìm thấy file!");
        } catch (IOException e) {
            System.out.println("Lỗi đọc file!");
        }
        // ↑ fis2 và reader TỰ ĐỘNG được đóng (close) ở đây
        //   Không cần viết finally!
    }
}
```

**Điều kiện dùng try-with-resources:** Tài nguyên phải implement interface `AutoCloseable` hoặc `Closeable`.

Các class Java phổ biến đã implement AutoCloseable:
- `FileInputStream`, `FileOutputStream`
- `BufferedReader`, `BufferedWriter`
- `Connection`, `Statement`, `ResultSet` (JDBC)
- `Scanner`

### 2.6. Nested try-catch (try-catch lồng nhau)

Đôi khi bạn cần xử lý lỗi ở nhiều tầng:

```java
public class NestedTryCatch {
    public static void main(String[] args) {
        try {
            System.out.println("Tầng ngoài: Bắt đầu");

            try {
                // Tầng trong: thử chia cho 0
                int result = 10 / 0;
            } catch (ArithmeticException e) {
                System.out.println("Tầng trong: Bắt lỗi chia cho 0");
                // Bọc (wrap) lỗi cũ trong lỗi mới rồi ném tiếp
                throw new RuntimeException("Lỗi xử lý phép tính", e);
            }

            System.out.println("Dòng này KHÔNG chạy vì throw ở trên");

        } catch (RuntimeException e) {
            System.out.println("Tầng ngoài: Bắt lỗi RuntimeException");
            System.out.println("Lỗi: " + e.getMessage());
            // Output: Lỗi: Lỗi xử lý phép tính
            System.out.println("Nguyên nhân gốc: " + e.getCause().getMessage());
            // Output: Nguyên nhân gốc: / by zero
        }
    }
}
```

💡 **Khi nào dùng nested try-catch?**
- Khi bạn muốn **bắt lỗi ở tầng trong**, xử lý một phần, rồi **ném lỗi mới lên tầng ngoài**
- Pattern phổ biến: **wrap exception** (bọc lỗi gốc vào lỗi mới có message rõ ràng hơn)

---

## 3. throw và throws (Ném lỗi và Khai báo lỗi)

### 🔥 Phân biệt throw vs throws

Đây là câu hỏi **RẤT HAY GẶP trong phỏng vấn**!

| Tiêu chí | `throw` (ném) | `throws` (khai báo) |
|-----------|---------------|---------------------|
| **Ý nghĩa** | NÉM một exception (tạo lỗi) | KHAI BÁO method có thể gây lỗi |
| **Vị trí** | Trong thân method (body) | Sau tên method (signature) |
| **Đi kèm với** | Một object exception | Danh sách loại exception |
| **Ví dụ** | `throw new IOException("lỗi")` | `void read() throws IOException` |
| **Ví dụ đời thường** | "NÉM quả bóng đi" | "Cảnh BÁO: Tôi CÓ THỂ ném bóng" |

### 3.1. throw — Ném exception (Tạo và ném lỗi)

Dùng `throw` khi bạn phát hiện điều kiện không hợp lệ → **chủ động tạo lỗi** để báo cho người gọi biết.

```java
public class ThrowDemo {

    // Hàm kiểm tra tuổi hợp lệ
    public static void validateAge(int age) {
        // Nếu tuổi < 0 → KHÔNG hợp lệ → ném lỗi
        if (age < 0) {
            throw new IllegalArgumentException(
                "Tuổi không thể là số âm! Nhận được: " + age
            );
            // ↑ Tạo một object IllegalArgumentException rồi NÉM ra ngoài
            // Code sau dòng throw KHÔNG BAO GIỜ chạy
        }

        // Nếu tuổi > 150 → KHÔNG hợp lệ → ném lỗi
        if (age > 150) {
            throw new IllegalArgumentException(
                "Tuổi không thể > 150! Nhận được: " + age
            );
        }

        // Nếu đến đây → tuổi hợp lệ
        System.out.println("Tuổi hợp lệ: " + age);
    }

    public static void main(String[] args) {
        // Người gọi phải BẮT lỗi
        try {
            validateAge(25);   // OK
            validateAge(-5);   // ← Lỗi tại đây!
            validateAge(200);  // KHÔNG chạy đến dòng này
        } catch (IllegalArgumentException e) {
            System.out.println("Lỗi validate: " + e.getMessage());
            // Output: Lỗi validate: Tuổi không thể là số âm! Nhận được: -5
        }
    }
}
```

### 3.2. throws — Khai báo exception (Cảnh báo)

Dùng `throws` trong **khai báo method** để nói rằng: "Method này CÓ THỂ ném ra loại lỗi này, người gọi tự lo xử lý nhé!"

```java
import java.io.*;

public class ThrowsDemo {

    // Khai báo: method này CÓ THỂ ném IOException
    // → Ai gọi method này PHẢI xử lý IOException
    public static String readFile(String path) throws IOException {
        BufferedReader reader = new BufferedReader(new FileReader(path));
        StringBuilder content = new StringBuilder();
        String line;

        while ((line = reader.readLine()) != null) {
            content.append(line).append("\n");
        }
        reader.close();

        return content.toString();
    }

    // Khai báo: method này CÓ THỂ ném NHIỀU loại lỗi
    public static void processFile(String path)
            throws IOException, IllegalArgumentException {

        if (path == null || path.isEmpty()) {
            throw new IllegalArgumentException("Đường dẫn file không được rỗng!");
        }
        String content = readFile(path); // ← Có thể ném IOException
        System.out.println("Nội dung file: " + content);
    }

    public static void main(String[] args) {
        // Người gọi PHẢI xử lý lỗi (vì readFile khai báo throws)
        try {
            String content = readFile("data.txt");
            System.out.println(content);
        } catch (IOException e) {
            System.out.println("Lỗi đọc file: " + e.getMessage());
        }
    }
}
```

### 3.3. Re-throwing exceptions (Ném lại lỗi)

Đôi khi bạn muốn **bắt lỗi → log → rồi ném lại** cho tầng trên xử lý tiếp:

```java
public class RethrowDemo {

    // Cách 1: Ném lại lỗi gốc (re-throw)
    public void processFile(String path) throws IOException {
        try {
            readFile(path);
        } catch (IOException e) {
            // Log lỗi trước
            System.out.println("LOG: Lỗi đọc file " + path + " - " + e.getMessage());
            // Ném LẠI lỗi gốc cho tầng trên
            throw e;
        }
    }

    // Cách 2: Bọc (wrap) trong exception khác
    // → Thêm ngữ cảnh (context) cho lỗi rõ ràng hơn
    public void processFile2(String path) {
        try {
            readFile(path);
        } catch (IOException e) {
            // Bọc IOException vào RuntimeException
            // Tham số thứ 2 'e' = giữ lại nguyên nhân gốc (cause)
            throw new RuntimeException("Không thể xử lý file: " + path, e);
        }
    }
}
```

💡 **Khi nào ném lại lỗi?**
- Khi tầng hiện tại **KHÔNG THỂ** xử lý lỗi hoàn toàn
- Khi muốn **log** lỗi trước rồi để tầng trên quyết định xử lý
- Khi muốn **bọc** lỗi kỹ thuật thành lỗi có ý nghĩa business

---

## 4. Custom Exceptions (Tạo lỗi tùy chỉnh riêng)

### Tại sao cần tạo Custom Exception?

Các exception có sẵn trong Java (IOException, NullPointerException...) đôi khi **không đủ rõ ràng** cho logic nghiệp vụ (business logic) của bạn.

Ví dụ: Khi rút tiền ATM thất bại, thay vì ném `Exception("error")` chung chung, bạn nên ném `InsufficientFundsException("Số dư không đủ")` → RÕ RÀNG hơn!

### Quy tắc tạo Custom Exception

```
Muốn tạo Checked Exception (BẮT BUỘC xử lý)?
→ extends Exception

Muốn tạo Unchecked Exception (KHÔNG bắt buộc)?
→ extends RuntimeException
```

### 4.1. Checked Custom Exception

```java
// Lỗi: Số dư không đủ (dùng cho hệ thống ngân hàng)
// extends Exception → Checked → Người gọi BẮT BUỘC xử lý
public class InsufficientFundsException extends Exception {

    private double amount;   // Số tiền muốn rút
    private double balance;  // Số dư hiện tại

    // Constructor 1: Chỉ truyền message
    public InsufficientFundsException(String message) {
        super(message); // Gọi constructor của Exception để set message
    }

    // Constructor 2: Truyền số tiền và số dư → tự tạo message chi tiết
    public InsufficientFundsException(double amount, double balance) {
        super(String.format(
            "Số dư không đủ: muốn rút %.2f nhưng chỉ còn %.2f",
            amount, balance
        ));
        this.amount = amount;
        this.balance = balance;
    }

    // Getter để người gọi lấy thông tin chi tiết
    public double getAmount() { return amount; }
    public double getBalance() { return balance; }
}
```

### 4.2. Unchecked Custom Exception

```java
// Lỗi: Email không hợp lệ
// extends RuntimeException → Unchecked → Người gọi KHÔNG bị ép xử lý
public class InvalidEmailException extends RuntimeException {

    private String email; // Email bị lỗi

    // Constructor 1: Truyền email sai
    public InvalidEmailException(String email) {
        super("Email không hợp lệ: " + email);
        this.email = email;
    }

    // Constructor 2: Truyền message + nguyên nhân gốc (cause)
    public InvalidEmailException(String message, Throwable cause) {
        super(message, cause);
    }

    public String getEmail() { return email; }
}
```

### 4.3. Sử dụng Custom Exception trong thực tế

```java
public class BankAccount {
    private String accountNumber; // Số tài khoản
    private double balance;       // Số dư

    public BankAccount(String accountNumber, double initialBalance) {
        this.accountNumber = accountNumber;
        this.balance = initialBalance;
    }

    // Rút tiền: CÓ THỂ ném InsufficientFundsException
    // → Checked → người gọi BẮT BUỘC try-catch hoặc throws
    public void withdraw(double amount) throws InsufficientFundsException {
        // Validate: số tiền phải > 0
        if (amount <= 0) {
            // IllegalArgumentException là Unchecked → không cần khai báo throws
            throw new IllegalArgumentException("Số tiền rút phải > 0");
        }

        // Validate: số dư phải đủ
        if (amount > balance) {
            // Ném Custom Checked Exception
            throw new InsufficientFundsException(amount, balance);
        }

        // Thực hiện rút tiền
        balance -= amount;
        System.out.printf("Đã rút: $%.2f. Số dư mới: $%.2f%n", amount, balance);
    }

    public static void main(String[] args) {
        BankAccount account = new BankAccount("123-456", 1000);

        try {
            account.withdraw(500);   // OK → Đã rút: $500.00. Số dư mới: $500.00
            account.withdraw(800);   // LỖI → Số dư không đủ!
        } catch (InsufficientFundsException e) {
            System.out.println("Lỗi: " + e.getMessage());
            // Output: Lỗi: Số dư không đủ: muốn rút 800.00 nhưng chỉ còn 500.00
            System.out.printf("Muốn rút: $%.2f, Chỉ còn: $%.2f%n",
                e.getAmount(), e.getBalance());
        }
    }
}
```

💡 **Mẹo nhớ chọn Checked hay Unchecked:**
- Lỗi do **yếu tố bên ngoài** mà code không thể ngăn chặn (file, network, DB) → **Checked** (extends Exception)
- Lỗi do **code viết sai** hoặc input không hợp lệ → **Unchecked** (extends RuntimeException)

---

## 5. Exception Methods (Các method hữu ích của Exception)

Mỗi exception object đều có sẵn các method này (kế thừa từ `Throwable`):

```java
try {
    int result = 10 / 0;
} catch (ArithmeticException e) {

    // 1. getMessage() — Lấy thông báo lỗi (message)
    System.out.println(e.getMessage());
    // Output: / by zero

    // 2. toString() — Lấy tên class + message
    System.out.println(e.toString());
    // Output: java.lang.ArithmeticException: / by zero

    // 3. printStackTrace() — In ra "dấu vết" lỗi (stack trace)
    //    Cho biết lỗi xảy ra ở file nào, dòng mấy
    e.printStackTrace();
    // Output:
    // java.lang.ArithmeticException: / by zero
    //     at MyClass.main(MyClass.java:3)

    // 4. getStackTrace() — Lấy stack trace dạng mảng
    //    Để tự xử lý (ví dụ: gửi vào hệ thống log)
    StackTraceElement[] stackTrace = e.getStackTrace();
    for (StackTraceElement element : stackTrace) {
        System.out.println("  tại " + element.getClassName()
            + "." + element.getMethodName()
            + " (dòng " + element.getLineNumber() + ")");
    }

    // 5. getCause() — Lấy nguyên nhân gốc (exception bên trong)
    //    Dùng khi exception được "bọc" (wrap)
    Throwable cause = e.getCause();
    // Nếu không có cause → trả về null
}
```

**Khi nào dùng method nào?**

| Method | Khi nào dùng? |
|--------|---------------|
| `getMessage()` | Hiển thị thông báo lỗi cho **người dùng** |
| `toString()` | Ghi log ngắn gọn (class + message) |
| `printStackTrace()` | Debug — in ra đầy đủ vị trí lỗi xảy ra |
| `getStackTrace()` | Gửi thông tin lỗi vào **hệ thống log** (Kibana, Sentry...) |
| `getCause()` | Truy tìm **nguyên nhân gốc** khi lỗi bị bọc nhiều lớp |

---

## 6. Common Exceptions (Các lỗi thường gặp nhất)

### 6.1. NullPointerException (NPE) — "Vua" của các lỗi Java

**Nguyên nhân**: Gọi method hoặc truy cập thuộc tính trên biến `null` (biến chưa được gán giá trị).

```java
String name = null; // Biến name chưa trỏ tới object nào

// ❌ SAI: Gọi .length() trên null → CRASH!
// int length = name.length(); // NullPointerException!

// ✅ Cách 1: Kiểm tra null trước
if (name != null) {
    int length = name.length(); // An toàn
}

// ✅ Cách 2: Dùng Objects.requireNonNull() — ném lỗi rõ ràng
import java.util.Objects;
String safeName = Objects.requireNonNull(name, "Tên không được null");
// ↑ Ném NullPointerException với message rõ ràng ngay lập tức

// ✅ Cách 3: Dùng Optional (Java 8+) — cách hiện đại
import java.util.Optional;
Optional<String> optName = Optional.ofNullable(name);
int length = optName.map(String::length).orElse(0);
// Nếu name = null → trả về 0 (giá trị mặc định)
// Nếu name = "Java" → trả về 4
```

💡 **Mẹo tránh NPE:** Luôn kiểm tra null TRƯỚC khi gọi method. Trong dự án thực tế, dùng `@NonNull`, `@Nullable` annotations để đánh dấu.

### 6.2. ArrayIndexOutOfBoundsException — Truy cập vượt mảng

**Nguyên nhân**: Truy cập mảng với index < 0 hoặc >= mảng.length.

```java
int[] arr = {10, 20, 30}; // Index hợp lệ: 0, 1, 2

// ❌ SAI: Index = 5 nhưng mảng chỉ có 3 phần tử (index 0-2)
// int value = arr[5]; // ArrayIndexOutOfBoundsException!

// ❌ SAI: Index = -1 (index âm)
// int value = arr[-1]; // ArrayIndexOutOfBoundsException!

// ✅ ĐÚNG: Kiểm tra index trước khi truy cập
int index = 5;
if (index >= 0 && index < arr.length) {
    int value = arr[index]; // An toàn
} else {
    System.out.println("Index " + index + " ngoài phạm vi (0 đến " + (arr.length - 1) + ")");
}
```

### 6.3. NumberFormatException — Chuyển chuỗi→số sai format

**Nguyên nhân**: Gọi `Integer.parseInt()` hoặc `Double.parseDouble()` với chuỗi không phải số.

```java
String input = "abc"; // Không phải số

// ❌ SAI: Chuyển "abc" thành int → CRASH!
// int num = Integer.parseInt(input); // NumberFormatException!

// ✅ Cách 1: try-catch
try {
    int num = Integer.parseInt(input);
    System.out.println("Số: " + num);
} catch (NumberFormatException e) {
    System.out.println("'" + input + "' không phải là số hợp lệ!");
}

// ✅ Cách 2: Validate bằng regex trước khi parse
if (input.matches("-?\\d+")) {
    // -? = có thể có dấu trừ (số âm)
    // \\d+ = 1 hoặc nhiều chữ số
    int num = Integer.parseInt(input);
} else {
    System.out.println("Chuỗi không hợp lệ");
}
```

### 6.4. ClassCastException — Ép kiểu sai

**Nguyên nhân**: Ép (cast) một object sang kiểu mà nó KHÔNG PHẢI.

```java
Object obj = "Hello"; // obj thực tế là String

// ❌ SAI: Ép String thành Integer → CRASH!
// Integer num = (Integer) obj; // ClassCastException!

// ✅ Cách 1: Kiểm tra bằng instanceof trước khi ép
if (obj instanceof Integer) {
    Integer num = (Integer) obj;
    System.out.println("Số: " + num);
} else {
    System.out.println("Object không phải Integer, mà là " + obj.getClass().getName());
}

// ✅ Cách 2: Pattern matching (Java 16+) — vừa kiểm tra vừa ép
if (obj instanceof Integer num) {
    // Nếu obj là Integer → tự động ép và gán vào biến 'num'
    System.out.println("Số: " + num);
} else if (obj instanceof String str) {
    // Nếu obj là String → tự động ép và gán vào biến 'str'
    System.out.println("Chuỗi: " + str);
}
```

---

## 7. Best Practices (Nguyên tắc tốt nhất khi xử lý lỗi)

### 7.1. Bắt lỗi CỤ THỂ, không bắt chung chung

```java
// ❌ SAI: Bắt Exception quá chung chung → không biết lỗi gì!
try {
    readFile("data.txt");
} catch (Exception e) {
    System.out.println("Có lỗi gì đó..."); // Lỗi gì? Không biết!
}

// ✅ ĐÚNG: Bắt từng loại lỗi cụ thể → xử lý phù hợp
try {
    readFile("data.txt");
} catch (FileNotFoundException e) {
    // Lỗi cụ thể: file không tồn tại → tạo file mới
    System.out.println("File không tồn tại, đang tạo mới...");
    createNewFile("data.txt");
} catch (IOException e) {
    // Lỗi cụ thể: lỗi đọc/ghi → thông báo cho user
    System.out.println("Lỗi đọc file: " + e.getMessage());
}
```

### 7.2. KHÔNG BAO GIỜ bỏ trống catch block (Nuốt lỗi)

```java
// ❌ SAI: "Nuốt" exception → lỗi xảy ra nhưng KHÔNG AI BIẾT!
// Đây là sai lầm NGUY HIỂM NHẤT khi xử lý lỗi
try {
    processPayment();
} catch (Exception e) {
    // Không làm gì cả → LỖI BỊ "NUỐT" (swallowed)
    // Thanh toán thất bại nhưng chương trình vẫn chạy
    // → Khách hàng mất tiền mà không biết!
}

// ✅ ĐÚNG: Ít nhất phải LOG lỗi
try {
    processPayment();
} catch (Exception e) {
    // Log lỗi để developer có thể debug
    logger.error("Lỗi thanh toán", e);
    // HOẶC ném lại lỗi
    throw new RuntimeException("Thanh toán thất bại", e);
}
```

💡 **Quy tắc vàng:** Nếu bạn catch một exception, bạn PHẢI làm ít nhất 1 trong 3 việc:
1. **Log** lỗi (ghi vào hệ thống log)
2. **Throw lại** (ném lại cho tầng trên)
3. **Xử lý** (khôi phục, retry, thông báo user...)

### 7.3. Dọn dẹp tài nguyên (Resource cleanup)

```java
// ❌ SAI: Mở file nhưng không đóng → rò rỉ tài nguyên (resource leak)
FileInputStream fis = new FileInputStream("data.txt");
// Nếu lỗi xảy ra ở đây → fis KHÔNG được đóng!
int data = fis.read();

// ✅ ĐÚNG: Dùng try-with-resources → tự động đóng
try (FileInputStream fis2 = new FileInputStream("data.txt")) {
    int data2 = fis2.read();
} // ← fis2 tự động đóng ở đây, dù có lỗi hay không
```

### 7.4. Throw early, Catch late (Ném sớm, Bắt muộn)

**Nguyên tắc:**
- **Throw early**: Validate đầu vào NGAY ĐẦU method → phát hiện lỗi sớm
- **Catch late**: Xử lý lỗi ở tầng **phù hợp** (thường là tầng giao diện/controller)

```java
// ===== THROW EARLY: Validate đầu vào ngay đầu method =====
public void processUser(User user) {
    // Kiểm tra NGAY → không để code chạy sâu rồi mới phát hiện lỗi
    if (user == null) {
        throw new IllegalArgumentException("User không được null!");
    }
    if (user.getName() == null || user.getName().isEmpty()) {
        throw new IllegalArgumentException("Tên user bắt buộc nhập!");
    }
    if (user.getAge() < 0 || user.getAge() > 150) {
        throw new IllegalArgumentException("Tuổi không hợp lệ: " + user.getAge());
    }

    // Nếu đến đây → tất cả đầu vào hợp lệ → an toàn để xử lý
    saveToDatabase(user);
    sendWelcomeEmail(user);
}

// ===== CATCH LATE: Xử lý lỗi ở tầng phù hợp =====
// Tầng Controller (giao diện) — nơi PHẢI trả lời cho user
public void handleRequest() {
    try {
        User user = parseUserFromRequest();
        service.processUser(user);        // ← Có thể ném lỗi
        showSuccessMessage("Đã tạo user thành công!");
    } catch (IllegalArgumentException e) {
        // Tầng Controller bắt lỗi → hiển thị cho user
        showErrorToUser(e.getMessage());
    }
}
```

### 7.5. Dùng Standard Exception có sẵn

Java đã cung cấp sẵn nhiều exception phổ biến. **Hãy dùng chúng** thay vì tạo custom exception không cần thiết.

```java
// ✅ IllegalArgumentException — Tham số không hợp lệ
public void setAge(int age) {
    if (age < 0 || age > 150) {
        throw new IllegalArgumentException("Tuổi phải từ 0 đến 150, nhận được: " + age);
    }
    this.age = age;
}

// ✅ IllegalStateException — Trạng thái object không hợp lệ
public void start() {
    if (isRunning) {
        throw new IllegalStateException("Server đã đang chạy, không thể start lại!");
    }
    isRunning = true;
}

// ✅ UnsupportedOperationException — Chức năng chưa được hỗ trợ
public void export() {
    throw new UnsupportedOperationException("Chức năng export chưa được triển khai!");
}

// ✅ Objects.requireNonNull() — Kiểm tra null nhanh
public void setName(String name) {
    this.name = Objects.requireNonNull(name, "Tên không được null");
}
```

---

## 8. Exception trong thực tế (Real-world patterns)

### Tại sao cần biết phần này?

Trong dự án thực tế, code được chia thành nhiều **tầng** (layers). Mỗi tầng xử lý lỗi **khác nhau**.

```
┌─────────────────────────────────────────────┐
│  CONTROLLER Layer (Tầng giao diện)          │
│  → Bắt lỗi → Trả HTTP response cho user    │
│  → Ví dụ: 404 Not Found, 400 Bad Request   │
├─────────────────────────────────────────────┤
│  SERVICE Layer (Tầng nghiệp vụ / logic)     │
│  → Validate → Ném lỗi cụ thể               │
│  → Ví dụ: UserNotFoundException             │
├─────────────────────────────────────────────┤
│  REPOSITORY Layer (Tầng dữ liệu)           │
│  → Truy vấn database                       │
│  → Ném lỗi khi không tìm thấy              │
└─────────────────────────────────────────────┘
```

### 8.1. Service Layer (Tầng nghiệp vụ)

```java
public class UserService {
    private UserRepository repository; // Truy vấn database

    // Tìm user theo ID → ném lỗi nếu không thấy
    public User findById(Long id) throws UserNotFoundException {
        return repository.findById(id)
            .orElseThrow(() -> new UserNotFoundException(
                "Không tìm thấy user với ID: " + id
            ));
        // orElseThrow: nếu kết quả rỗng → tạo và ném exception
    }

    // Tạo user mới → validate trước
    public User create(UserCreateRequest request) {
        // Bước 1: Validate đầu vào (Throw early!)
        validateRequest(request);

        // Bước 2: Kiểm tra email đã tồn tại chưa?
        if (repository.existsByEmail(request.getEmail())) {
            throw new DuplicateEmailException(request.getEmail());
        }

        // Bước 3: Tạo user
        User user = new User(request);
        return repository.save(user);
    }

    // Validate đầu vào
    private void validateRequest(UserCreateRequest request) {
        if (request.getName() == null || request.getName().isBlank()) {
            throw new ValidationException("Tên bắt buộc nhập!");
        }
        if (request.getEmail() == null || !request.getEmail().contains("@")) {
            throw new ValidationException("Email không hợp lệ!");
        }
    }
}
```

### 8.2. Controller Layer (Tầng giao diện)

```java
public class UserController {
    private UserService userService;

    // API: Lấy thông tin user
    public Response getUser(Long id) {
        try {
            User user = userService.findById(id);
            return Response.ok(user);                    // 200 OK

        } catch (UserNotFoundException e) {
            return Response.notFound(e.getMessage());    // 404 Not Found

        } catch (Exception e) {
            // Lỗi không lường trước → trả 500 Internal Server Error
            return Response.error("Lỗi hệ thống, vui lòng thử lại sau");
        }
    }

    // API: Tạo user mới
    public Response createUser(UserCreateRequest request) {
        try {
            User user = userService.create(request);
            return Response.created(user);               // 201 Created

        } catch (ValidationException e) {
            return Response.badRequest(e.getMessage());  // 400 Bad Request

        } catch (DuplicateEmailException e) {
            return Response.conflict(e.getMessage());    // 409 Conflict
        }
    }
}
```

**Tổng kết pattern:**

| Tầng | Vai trò xử lý lỗi | Hành động |
|------|---------------------|-----------|
| **Repository** | Truy vấn data | Ném lỗi nếu không tìm thấy |
| **Service** | Validate + Business logic | Ném Custom Exception cụ thể |
| **Controller** | Giao diện với user | Bắt exception → trả HTTP response phù hợp |

---

## 9. Sai lầm thường gặp

### Sai lầm 1: Bắt Exception quá chung chung

```java
// ❌ SAI: Bắt Exception → không biết lỗi gì để xử lý phù hợp
try {
    connectToDatabase();
    readFile("config.txt");
    sendEmail();
} catch (Exception e) {
    System.out.println("Có lỗi"); // Lỗi gì? DB? File? Email? Không biết!
}

// ✅ ĐÚNG: Bắt từng loại lỗi cụ thể
try {
    connectToDatabase();
    readFile("config.txt");
    sendEmail();
} catch (SQLException e) {
    System.out.println("Lỗi kết nối database: " + e.getMessage());
    reconnectDatabase();
} catch (FileNotFoundException e) {
    System.out.println("Thiếu file config, dùng config mặc định");
    useDefaultConfig();
} catch (IOException e) {
    System.out.println("Lỗi gửi email: " + e.getMessage());
    retryEmail();
}
```

### Sai lầm 2: Dùng Exception để điều khiển luồng (flow control)

```java
// ❌ SAI: Dùng exception thay cho if-else → RẤT CHẬM!
public boolean isNumber(String str) {
    try {
        Integer.parseInt(str);
        return true;
    } catch (NumberFormatException e) {
        return false;
    }
}

// ✅ ĐÚNG: Dùng if-else để kiểm tra trước
public boolean isNumber(String str) {
    if (str == null || str.isEmpty()) return false;
    return str.matches("-?\\d+");
}
```

⚠️ **Exception rất TỐN hiệu năng** (performance). Tạo một exception cần thu thập stack trace → chậm hơn if-else hàng **trăm lần**. Chỉ dùng exception cho tình huống **bất thường**, KHÔNG dùng cho logic bình thường.

### Sai lầm 3: Log rồi throw lại (log trùng)

```java
// ❌ SAI: Vừa log VỪA throw → lỗi bị log 2 lần!
try {
    readFile("data.txt");
} catch (IOException e) {
    logger.error("Lỗi đọc file", e);  // Log lần 1
    throw e;                            // Tầng trên catch → log lần 2
}

// ✅ ĐÚNG: Chọn 1 trong 2
// Cách A: Log rồi xử lý (KHÔNG throw)
try {
    readFile("data.txt");
} catch (IOException e) {
    logger.error("Lỗi đọc file", e);
    useDefaultData(); // Xử lý thay thế
}

// Cách B: Wrap rồi throw (KHÔNG log) → tầng trên sẽ log
try {
    readFile("data.txt");
} catch (IOException e) {
    throw new RuntimeException("Không thể đọc file data.txt", e);
}
```

---

## 10. Tóm tắt cuối ngày

### Bảng tổng hợp kiến thức

| Khái niệm | Giải thích tiếng Việt | Ví dụ |
|------------|----------------------|-------|
| **Exception** | Ngoại lệ / tình huống bất thường | `ArithmeticException` |
| **Checked Exception** | Lỗi BẮT BUỘC xử lý (compiler ép) | `IOException`, `SQLException` |
| **Unchecked Exception** | Lỗi KHÔNG bắt buộc xử lý | `NullPointerException` |
| **Error** | Lỗi nghiêm trọng, KHÔNG nên bắt | `OutOfMemoryError` |
| **try** | "Thử" chạy code có thể lỗi | `try { riskyCode(); }` |
| **catch** | "Bắt" lỗi và xử lý | `catch (IOException e) { ... }` |
| **finally** | Code LUÔN chạy (dọn dẹp) | `finally { file.close(); }` |
| **try-with-resources** | Tự động đóng tài nguyên | `try (var f = new FileReader()) { }` |
| **throw** | NÉM một exception | `throw new Exception("lỗi")` |
| **throws** | KHAI BÁO method có thể ném lỗi | `void read() throws IOException` |
| **Custom Exception** | Tạo lỗi tùy chỉnh riêng | `class MyError extends Exception` |

### 🔥 Câu hỏi phỏng vấn thường gặp

1. **Checked vs Unchecked Exception khác nhau thế nào?**
   → Checked: compiler bắt buộc xử lý, kế thừa Exception. Unchecked: không bắt buộc, kế thừa RuntimeException.

2. **throw vs throws khác nhau thế nào?**
   → throw: ném exception (trong body). throws: khai báo exception (trong signature).

3. **finally có luôn chạy không?**
   → CÓ, trừ khi gọi `System.exit()` hoặc JVM crash.

4. **try-with-resources là gì?**
   → Tự động đóng tài nguyên implement AutoCloseable khi ra khỏi try.

5. **Khi nào tạo Custom Exception?**
   → Khi các exception có sẵn không đủ rõ ràng cho business logic.

---

## 11. Bài tập thực hành

### Bài 1: Custom Exception cho ATM

Tạo hệ thống ATM đơn giản với các Custom Exception:
- `InsufficientBalanceException` — Số dư không đủ
- `InvalidAmountException` — Số tiền không hợp lệ
- `CardBlockedException` — Thẻ bị khóa
- `DailyLimitExceededException` — Vượt hạn mức rút tiền/ngày

```java
public class ATM {
    public void withdraw(String cardNumber, double amount)
        throws InsufficientBalanceException, InvalidAmountException,
               CardBlockedException, DailyLimitExceededException {
        // Yêu cầu:
        // 1. Kiểm tra thẻ có bị khóa không
        // 2. Kiểm tra số tiền > 0 và chia hết cho 50
        // 3. Kiểm tra hạn mức rút tiền trong ngày (max 5000)
        // 4. Kiểm tra số dư
        // 5. Thực hiện rút tiền
    }
}
```

### Bài 2: File Processor

Tạo class đọc file CSV và chuyển thành danh sách object:
- Đọc file text từng dòng
- Parse từng dòng thành object Person (name, age, email)
- Nếu dòng nào lỗi → log và BỎ QUA (không crash)
- Trả về danh sách Person hợp lệ

```
File format (people.csv):
name,age,email
Nguyen Van A,25,a@email.com
Dòng sai format
Tran Thi B,abc,b@email.com
Le Van C,30,c@email.com
```

### Bài 3: Validation Framework

Tạo framework validate đơn giản dùng annotation:

```java
public class User {
    @NotNull
    private String name;     // Không được null

    @Min(0) @Max(150)
    private int age;         // Phải từ 0 đến 150

    @Email
    private String email;    // Phải có format email
}

// Gọi validate:
Validator.validate(user); // Ném ValidationException nếu sai
```

### Bài 4: Retry Mechanism (Cơ chế thử lại)

Tạo helper tự động **retry** (thử lại) khi gặp lỗi:

```java
// Gọi fetchData(), nếu lỗi → đợi 1 giây → thử lại (tối đa 3 lần)
String result = RetryHelper.retry(() -> fetchData(), 3, 1000);
```

---

## 12. Đáp án tham khảo

<details>
<summary>Bài 1: Custom Exception cho ATM (Click để xem)</summary>

```java
import java.util.*;

// ===== CUSTOM EXCEPTIONS =====

// Lỗi: Số dư không đủ
class InsufficientBalanceException extends Exception {
    public InsufficientBalanceException(double requested, double available) {
        super(String.format(
            "Số dư không đủ. Muốn rút: %.2f, Chỉ còn: %.2f",
            requested, available));
    }
}

// Lỗi: Số tiền không hợp lệ
class InvalidAmountException extends Exception {
    public InvalidAmountException(String message) {
        super(message);
    }
}

// Lỗi: Thẻ bị khóa
class CardBlockedException extends Exception {
    public CardBlockedException(String cardNumber) {
        super("Thẻ bị khóa: " + cardNumber);
    }
}

// Lỗi: Vượt hạn mức rút tiền/ngày
class DailyLimitExceededException extends Exception {
    public DailyLimitExceededException(double limit) {
        super("Đã vượt hạn mức rút tiền trong ngày: " + limit);
    }
}

// ===== CLASS ATM =====
public class ATM {
    private Map<String, Double> balances = new HashMap<>();       // Số dư mỗi thẻ
    private Map<String, Boolean> blockedCards = new HashMap<>();   // Thẻ bị khóa
    private Map<String, Double> dailyWithdrawals = new HashMap<>(); // Tổng rút trong ngày
    private static final double DAILY_LIMIT = 5000;               // Hạn mức/ngày

    public void withdraw(String cardNumber, double amount)
            throws InsufficientBalanceException, InvalidAmountException,
                   CardBlockedException, DailyLimitExceededException {

        // Bước 1: Kiểm tra thẻ có bị khóa không
        if (blockedCards.getOrDefault(cardNumber, false)) {
            throw new CardBlockedException(cardNumber);
        }

        // Bước 2: Kiểm tra số tiền hợp lệ
        if (amount <= 0) {
            throw new InvalidAmountException("Số tiền phải > 0");
        }
        if (amount % 50 != 0) {
            throw new InvalidAmountException("Số tiền phải chia hết cho 50");
        }

        // Bước 3: Kiểm tra hạn mức rút/ngày
        double todayTotal = dailyWithdrawals.getOrDefault(cardNumber, 0.0);
        if (todayTotal + amount > DAILY_LIMIT) {
            throw new DailyLimitExceededException(DAILY_LIMIT);
        }

        // Bước 4: Kiểm tra số dư
        double balance = balances.getOrDefault(cardNumber, 0.0);
        if (amount > balance) {
            throw new InsufficientBalanceException(amount, balance);
        }

        // Bước 5: Thực hiện rút tiền
        balances.put(cardNumber, balance - amount);
        dailyWithdrawals.put(cardNumber, todayTotal + amount);
        System.out.printf("Rút thành công $%.2f. Số dư mới: $%.2f%n",
            amount, balances.get(cardNumber));
    }

    public static void main(String[] args) {
        ATM atm = new ATM();
        atm.balances.put("1234", 10000.0); // Nạp 10000 cho thẻ 1234

        try {
            atm.withdraw("1234", 500);    // OK
            atm.withdraw("1234", 6000);   // Vượt hạn mức ngày!
        } catch (InsufficientBalanceException e) {
            System.out.println("Lỗi số dư: " + e.getMessage());
        } catch (InvalidAmountException e) {
            System.out.println("Lỗi số tiền: " + e.getMessage());
        } catch (CardBlockedException e) {
            System.out.println("Lỗi thẻ: " + e.getMessage());
        } catch (DailyLimitExceededException e) {
            System.out.println("Lỗi hạn mức: " + e.getMessage());
        }
    }
}
```
</details>

<details>
<summary>Bài 4: Retry Mechanism (Click để xem)</summary>

```java
import java.util.concurrent.Callable;

public class RetryHelper {

    /**
     * Thử gọi task, nếu lỗi → đợi rồi thử lại
     * @param task        Công việc cần thực hiện (kiểu Callable<T>)
     * @param maxAttempts Số lần thử tối đa
     * @param delayMs     Thời gian đợi giữa mỗi lần thử (milliseconds)
     * @return Kết quả nếu thành công
     * @throws Exception Nếu tất cả lần thử đều thất bại
     */
    public static <T> T retry(Callable<T> task, int maxAttempts, long delayMs)
            throws Exception {

        Exception lastException = null; // Lưu lỗi cuối cùng

        for (int attempt = 1; attempt <= maxAttempts; attempt++) {
            try {
                System.out.println("Lần thử " + attempt + "...");
                return task.call(); // Thử gọi task

            } catch (Exception e) {
                lastException = e; // Lưu lỗi
                System.out.println("Lần " + attempt + " thất bại: " + e.getMessage());

                // Nếu chưa hết số lần thử → đợi rồi thử lại
                if (attempt < maxAttempts) {
                    System.out.println("Đợi " + delayMs + "ms rồi thử lại...");
                    Thread.sleep(delayMs);
                }
            }
        }

        // Tất cả lần thử đều thất bại → ném lỗi
        throw new Exception(
            "Tất cả " + maxAttempts + " lần thử đều thất bại",
            lastException
        );
    }

    public static void main(String[] args) {
        try {
            // Thử gọi fetchData() tối đa 5 lần, đợi 1 giây giữa mỗi lần
            String result = retry(() -> {
                // Giả lập: 70% xác suất thất bại
                if (Math.random() < 0.7) {
                    throw new RuntimeException("Lỗi kết nối mạng");
                }
                return "Dữ liệu thành công!";
            }, 5, 1000);

            System.out.println("Kết quả: " + result);

        } catch (Exception e) {
            System.out.println("Thất bại hoàn toàn: " + e.getMessage());
        }
    }
}
```
</details>

---

## Navigation

- [← Day 4: OOP Pillars (4 Trụ cột OOP)](./day-04-oop-pillars.md)
- [Day 6: Strings & Wrappers (Chuỗi & Kiểu bọc) →](./day-06-strings-wrappers.md)
