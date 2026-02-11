# Day 11: File I/O (Đọc Ghi File - Nhập Xuất Dữ Liệu)

## Mục tiêu hôm nay
- Hiểu java.io và java.nio.file (NIO.2) khác nhau như thế nào
- Path và Files - cách hiện đại để thao tác file
- Đọc file: readAllLines, BufferedReader, InputStream
- Ghi file: write, BufferedWriter, OutputStream
- Duyệt thư mục: walk, find, DirectoryStream
- Serialization (tuần tự hóa đối tượng) và Properties files

---

## 🤔 Tại sao cần học File I/O?

### Ví dụ đời thường
> Mọi ứng dụng đều cần **đọc/ghi dữ liệu**: file cấu hình, log, CSV, JSON, ảnh, video...
> Giống như bạn cần biết **đọc sách** và **viết nhật ký** - đó là kỹ năng cơ bản để giao tiếp với "thế giới bên ngoài" chương trình.
>
> - **Đọc file** = "Mở sách ra đọc" - lấy dữ liệu từ ổ cứng vào chương trình
> - **Ghi file** = "Viết nhật ký" - lưu dữ liệu từ chương trình ra ổ cứng
> - **Thao tác thư mục** = "Sắp xếp tủ sách" - tạo, xóa, liệt kê folder

### 2 thư viện chính trong Java

```
┌─────────────────────────────────────────────────────────────────┐
│                    JAVA FILE I/O EVOLUTION                      │
├────────────────────────────┬────────────────────────────────────┤
│ java.io (cũ - Java 1.0)   │ java.nio.file (mới - Java 7+)     │
├────────────────────────────┼────────────────────────────────────┤
│ Class chính: File          │ Class chính: Path + Files          │
│ Blocking I/O               │ Non-blocking I/O                   │
│ Ít method tiện ích          │ Rất nhiều method tiện ích          │
│ Khó xử lý lỗi              │ Exception rõ ràng                  │
│ Không hỗ trợ symbolic link │ Hỗ trợ symbolic link               │
│ Không hỗ trợ file attrs    │ Hỗ trợ file attributes đầy đủ     │
├────────────────────────────┼────────────────────────────────────┤
│ ⚠️ Legacy - vẫn gặp       │ ✅ Khuyến khích dùng trong code   │
│ trong code cũ               │ mới                                │
└────────────────────────────┴────────────────────────────────────┘

💡 Quy tắc: Viết code mới → dùng java.nio.file (Path + Files)
   Đọc code cũ → cần hiểu java.io (File) để maintain
```

---

## 1. File Class - java.io (Cách Cũ, Cần Biết Để Đọc Code Legacy)

```java
import java.io.File;

// === Tạo File object (CHỈ tạo object, CHƯA tạo file thật trên ổ cứng!) ===
File file = new File("data.txt");                          // Đường dẫn tương đối
File file2 = new File("C:/Users/Documents/data.txt");      // Đường dẫn tuyệt đối
File dir = new File("myFolder");                           // Thư mục

// === Lấy thông tin file ===
file.getName();           // "data.txt" - tên file
file.getPath();           // "data.txt" - đường dẫn đã truyền vào
file.getAbsolutePath();   // "C:/project/data.txt" - đường dẫn tuyệt đối
file.getParent();         // Thư mục cha
file.length();            // Kích thước file (đơn vị: bytes)

// === Kiểm tra file ===
file.exists();            // File có tồn tại không?
file.isFile();            // Đây có phải file (không phải folder)?
file.isDirectory();       // Đây có phải thư mục?
file.canRead();           // Có quyền đọc không?
file.canWrite();          // Có quyền ghi không?
file.isHidden();          // File bị ẩn không?

// === Thao tác file ===
file.createNewFile();     // Tạo file mới (trả về true nếu tạo thành công)
dir.mkdir();              // Tạo 1 thư mục (thư mục cha phải tồn tại)
dir.mkdirs();             // Tạo thư mục + tất cả thư mục cha nếu chưa có
file.delete();            // Xóa file/thư mục rỗng
file.renameTo(new File("newname.txt")); // Đổi tên

// === Liệt kê nội dung thư mục ===
File folder = new File(".");                               // "." = thư mục hiện tại
String[] names = folder.list();                            // Lấy tên files/folders
File[] files = folder.listFiles();                         // Lấy File objects
File[] txtFiles = folder.listFiles(
    (d, name) -> name.endsWith(".txt")                     // Lọc chỉ lấy file .txt
);
```

> ⚠️ **Hạn chế của java.io.File:**
> - `delete()` trả về `false` nếu xóa thất bại → KHÔNG biết lý do tại sao
> - `mkdir()` trả về `false` nếu thất bại → KHÔNG có exception
> - Không hỗ trợ symbolic links (liên kết tắt)

---

## 2. Path và Files - java.nio.file (Cách Hiện Đại, NÊN Dùng)

### 2.1. Path - Đại diện cho đường dẫn file/folder

> **Ví dụ đời thường**: `Path` giống như **địa chỉ nhà**. Nó chỉ là địa chỉ, KHÔNG phải ngôi nhà. Bạn dùng địa chỉ để tìm đến ngôi nhà (file thật).

```java
import java.nio.file.Path;
import java.nio.file.Paths;

// === Tạo Path ===
Path path1 = Paths.get("data.txt");                       // Cách 1: Paths.get()
Path path2 = Paths.get("C:", "Users", "Documents", "data.txt"); // Nối nhiều phần
Path path3 = Path.of("data.txt");                         // Cách 2: Path.of() (Java 11+, ưu tiên dùng)

// === Lấy thông tin ===
path1.getFileName();       // data.txt - tên file
path1.getParent();         // Thư mục cha (null nếu không có)
path1.getRoot();           // C:\ (trên Windows) hoặc / (trên Linux)
path1.toAbsolutePath();    // Đường dẫn tuyệt đối đầy đủ
path1.normalize();         // Chuẩn hóa: loại bỏ . và ..
                           // Ví dụ: "a/b/../c" → "a/c"

// === Thao tác đường dẫn ===
// resolve() = "nối thêm" đường dẫn con
path1.resolve("subdir");           // data.txt/subdir
// Ví dụ thực tế:
Path baseDir = Path.of("C:/project");
Path configFile = baseDir.resolve("config").resolve("app.properties");
// → C:/project/config/app.properties

// relativize() = "tìm đường đi" từ path A đến path B
Path pathA = Path.of("C:/project");
Path pathB = Path.of("C:/project/src/Main.java");
Path relative = pathA.relativize(pathB);
// → src/Main.java (đường đi từ A đến B)

// Kiểm tra
path1.startsWith("C:");              // Bắt đầu bằng C: ?
path1.endsWith(".txt");              // Kết thúc bằng .txt ?
```

```
Path vs String - Tại sao dùng Path thay vì String?

┌─────────────────────────────────────────────────────┐
│ String path = "C:\\Users\\data.txt";                │
│ → Chỉ là text, không có method xử lý               │
│ → Phải tự xử lý / vs \\ giữa OS                    │
│ → Dễ sai khi nối đường dẫn                          │
├─────────────────────────────────────────────────────┤
│ Path path = Path.of("C:/Users/data.txt");           │
│ → Có method: getFileName(), getParent()...          │
│ → Tự xử lý separator cho từng OS                   │
│ → resolve() nối đường dẫn an toàn                   │
│ → normalize() dọn dẹp đường dẫn rác                │
└─────────────────────────────────────────────────────┘
```

### 2.2. Files - Bộ công cụ thao tác file mạnh mẽ

```java
import java.nio.file.Files;
import java.nio.file.StandardCopyOption;
import java.nio.file.StandardOpenOption;

Path path = Path.of("data.txt");

// === Kiểm tra ===
Files.exists(path);              // Tồn tại không?
Files.isRegularFile(path);       // Là file thường (không phải folder, symlink)?
Files.isDirectory(path);         // Là thư mục?
Files.isReadable(path);          // Có thể đọc?
Files.isWritable(path);          // Có thể ghi?
Files.size(path);                // Kích thước (bytes)

// === Tạo file/thư mục ===
Files.createFile(path);                              // Tạo file mới (throw nếu đã tồn tại)
Files.createDirectory(Path.of("newDir"));            // Tạo 1 thư mục
Files.createDirectories(Path.of("a/b/c"));           // Tạo cả chuỗi thư mục a → b → c
Files.createTempFile("prefix", ".tmp");              // Tạo file tạm thời

// === Sao chép / Di chuyển / Xóa ===
Path source = Path.of("original.txt");
Path target = Path.of("copy.txt");

Files.copy(source, target);                          // Sao chép (throw nếu target đã tồn tại)
Files.copy(source, target,
    StandardCopyOption.REPLACE_EXISTING);             // Sao chép, ghi đè nếu đã tồn tại

Files.move(source, target);                          // Di chuyển (đổi tên)
Files.move(source, target,
    StandardCopyOption.REPLACE_EXISTING);             // Di chuyển, ghi đè

Files.delete(path);                                  // Xóa (throw nếu không tồn tại)
Files.deleteIfExists(path);                          // Xóa nếu tồn tại (an toàn hơn)
```

> 💡 **So sánh với java.io.File:**
> - `File.delete()` → trả về `false` nếu thất bại, KHÔNG biết lý do
> - `Files.delete()` → throw `IOException` với message rõ ràng: "file not found", "permission denied"...
> → **NIO.2 giúp debug dễ hơn rất nhiều!**

---

## 3. Đọc File (Reading Files)

### 3.1. Đọc toàn bộ nội dung (File nhỏ)

> ⚠️ **Chú ý:** Các method này đọc TOÀN BỘ file vào bộ nhớ. Chỉ phù hợp với file nhỏ (vài MB). File lớn → dùng BufferedReader hoặc Stream.

```java
// === Đọc tất cả dòng → List<String> ===
List<String> lines = Files.readAllLines(Path.of("data.txt"));
// Mỗi dòng trong file = 1 phần tử trong List

// Chỉ định encoding (mã hóa ký tự)
List<String> linesUtf8 = Files.readAllLines(
    Path.of("data.txt"),
    StandardCharsets.UTF_8                   // UTF-8 cho tiếng Việt
);

// === Đọc tất cả bytes → byte[] ===
byte[] bytes = Files.readAllBytes(Path.of("image.png"));
// Dùng cho file binary (ảnh, video, PDF...)

// === Đọc thành 1 String (Java 11+) - TIỆN NHẤT ===
String content = Files.readString(Path.of("data.txt"));
// Đọc toàn bộ file thành 1 chuỗi duy nhất
```

```
Chọn method đọc file nào?

┌────────────────────────────┬──────────────────────────────────┐
│ Method                     │ Khi nào dùng?                    │
├────────────────────────────┼──────────────────────────────────┤
│ Files.readString()         │ File text nhỏ, cần toàn bộ nội  │
│ (Java 11+)                 │ dung dưới dạng 1 String          │
├────────────────────────────┼──────────────────────────────────┤
│ Files.readAllLines()       │ File text nhỏ, cần xử lý        │
│                            │ từng dòng (List<String>)         │
├────────────────────────────┼──────────────────────────────────┤
│ Files.readAllBytes()       │ File binary (ảnh, PDF...)        │
├────────────────────────────┼──────────────────────────────────┤
│ BufferedReader              │ File text LỚN, đọc từng dòng    │
│                            │ để tiết kiệm bộ nhớ             │
├────────────────────────────┼──────────────────────────────────┤
│ Files.lines() (Stream)     │ File text LỚN + cần Stream API  │
│                            │ (filter, map, collect...)        │
├────────────────────────────┼──────────────────────────────────┤
│ InputStream                │ File binary LỚN, đọc từng block │
└────────────────────────────┴──────────────────────────────────┘
```

### 3.2. Đọc từng dòng với BufferedReader (File lớn)

> **Ví dụ đời thường**: Thay vì **đọc hết quyển sách rồi mới xử lý**, ta đọc **từng trang** để không bị quá tải bộ nhớ.

```java
// === BufferedReader: đọc từng dòng, tiết kiệm bộ nhớ ===
try (BufferedReader reader = Files.newBufferedReader(Path.of("data.txt"))) {
    String line;
    while ((line = reader.readLine()) != null) {   // readLine() trả null khi hết file
        System.out.println(line);
    }
}
// try-with-resources tự động đóng reader khi xong

// === Kết hợp với Stream API (cách hiện đại nhất cho file lớn) ===
try (Stream<String> lines = Files.lines(Path.of("data.txt"))) {
    lines.filter(line -> line.contains("ERROR"))   // Lọc dòng chứa "ERROR"
         .map(String::trim)                        // Xóa khoảng trắng đầu/cuối
         .forEach(System.out::println);            // In ra
}
// ⚠️ PHẢI dùng try-with-resources vì Stream từ file cần được đóng!

// === Ví dụ thực tế: Đếm dòng có chứa từ khóa ===
try (Stream<String> lines = Files.lines(Path.of("access.log"))) {
    long errorCount = lines
        .filter(line -> line.contains("500"))      // HTTP 500 errors
        .count();
    System.out.println("Số lỗi 500: " + errorCount);
}
```

### 3.3. InputStream - Đọc file binary (ảnh, video, PDF...)

```java
// === Đọc file binary từng block ===
try (InputStream is = Files.newInputStream(Path.of("image.png"))) {
    byte[] buffer = new byte[1024];     // Buffer 1KB
    int bytesRead;
    while ((bytesRead = is.read(buffer)) != -1) {  // -1 = hết file
        // Xử lý bytesRead bytes từ buffer
        // bytesRead có thể < 1024 ở lần đọc cuối
    }
}

// === Thêm BufferedInputStream để tăng hiệu suất ===
try (BufferedInputStream bis = new BufferedInputStream(
        Files.newInputStream(Path.of("large-file.bin")))) {
    // BufferedInputStream tự tạo buffer nội bộ
    // → Giảm số lần đọc ổ cứng → nhanh hơn
    byte[] data = bis.readAllBytes();   // Java 9+: đọc tất cả
}
```

> 💡 **Mẹo nhớ: Text vs Binary**
> - **Text file** (.txt, .csv, .json, .xml, .html) → dùng **Reader** (xử lý ký tự)
> - **Binary file** (.png, .jpg, .pdf, .zip, .exe) → dùng **InputStream** (xử lý bytes)

---

## 4. Ghi File (Writing Files)

### 4.1. Ghi toàn bộ nội dung (File nhỏ)

```java
// === Ghi danh sách dòng ===
List<String> lines = Arrays.asList("Dòng 1", "Dòng 2", "Dòng 3");
Files.write(Path.of("output.txt"), lines);
// Tạo file mới hoặc GHI ĐÈ nếu đã tồn tại

// === Ghi bytes ===
byte[] data = "Hello World".getBytes(StandardCharsets.UTF_8);
Files.write(Path.of("output.txt"), data);

// === Ghi String trực tiếp (Java 11+) - TIỆN NHẤT ===
Files.writeString(Path.of("output.txt"), "Xin chào Việt Nam!");

// === GHI THÊM (Append) thay vì ghi đè ===
Files.write(Path.of("output.txt"), lines,
    StandardOpenOption.APPEND);                    // Thêm vào cuối file

// === Tạo file mới + ghi thêm (không ghi đè file cũ) ===
Files.write(Path.of("log.txt"), lines,
    StandardOpenOption.CREATE,                     // Tạo nếu chưa có
    StandardOpenOption.APPEND);                    // Thêm vào cuối
```

```
StandardOpenOption - Các tùy chọn mở file:

┌─────────────────────────┬──────────────────────────────────────┐
│ Option                  │ Ý nghĩa                             │
├─────────────────────────┼──────────────────────────────────────┤
│ CREATE                  │ Tạo file nếu chưa tồn tại           │
│ CREATE_NEW              │ Tạo file mới (lỗi nếu đã tồn tại)  │
│ TRUNCATE_EXISTING       │ Xóa nội dung cũ khi mở (mặc định)  │
│ APPEND                  │ Ghi thêm vào cuối file              │
│ WRITE                   │ Mở để ghi                           │
│ READ                    │ Mở để đọc                           │
│ DELETE_ON_CLOSE         │ Xóa file khi đóng (file tạm)       │
└─────────────────────────┴──────────────────────────────────────┘
```

### 4.2. Ghi từng phần với BufferedWriter (File lớn / nhiều lần ghi)

```java
// === BufferedWriter: ghi hiệu quả với buffer ===
try (BufferedWriter writer = Files.newBufferedWriter(Path.of("output.txt"))) {
    writer.write("Dòng 1");           // Ghi text
    writer.newLine();                  // Xuống dòng (tự dùng \n hoặc \r\n tùy OS)
    writer.write("Dòng 2");
    writer.newLine();
    writer.write("Dòng 3");
}
// try-with-resources tự động flush() và close()

// === PrintWriter: tiện hơn với println(), printf() ===
try (PrintWriter pw = new PrintWriter(
        Files.newBufferedWriter(Path.of("output.txt")))) {
    pw.println("Dòng 1");                              // Tự xuống dòng
    pw.println("Dòng 2");
    pw.printf("Tên: %s, Tuổi: %d%n", "John", 25);     // Format giống C
}

// 💡 PrintWriter vs BufferedWriter:
// PrintWriter: Có println(), printf() → tiện khi ghi text có format
// BufferedWriter: Có newLine() → tương thích tốt hơn giữa các OS
```

### 4.3. OutputStream - Ghi file binary

```java
// === Ghi file binary ===
try (OutputStream os = Files.newOutputStream(Path.of("output.bin"))) {
    os.write("Hello".getBytes());                      // Ghi bytes
}

// === Ghi thêm vào file binary ===
try (OutputStream os = Files.newOutputStream(Path.of("output.bin"),
        StandardOpenOption.CREATE,
        StandardOpenOption.APPEND)) {
    os.write("More data".getBytes());
}
```

---

## 5. Duyệt Thư Mục (Working with Directories)

### 5.1. Liệt kê nội dung thư mục

```java
// === DirectoryStream: Liệt kê file/folder trong 1 thư mục ===
try (DirectoryStream<Path> stream = Files.newDirectoryStream(Path.of("."))) {
    for (Path entry : stream) {
        System.out.println(entry.getFileName());       // In tên từng file/folder
    }
}

// === Lọc theo pattern (glob) ===
try (DirectoryStream<Path> stream = Files.newDirectoryStream(
        Path.of("."), "*.txt")) {                      // Chỉ lấy file .txt
    for (Path entry : stream) {
        System.out.println(entry);
    }
}
// Glob patterns:
// "*.txt"     → file .txt trong thư mục hiện tại
// "*.{txt,csv}" → file .txt hoặc .csv
// "data*"     → file bắt đầu bằng "data"
```

### 5.2. Duyệt đệ quy (Recursive) - Files.walk()

> **Ví dụ đời thường**: `Files.walk()` giống như đi vào **mê cung thư mục**, mở từng cánh cửa, vào từng phòng con để liệt kê tất cả.

```java
// === walk(): Duyệt TẤT CẢ file/folder trong cây thư mục ===
try (Stream<Path> walk = Files.walk(Path.of("."))) {
    walk.filter(Files::isRegularFile)                  // Chỉ lấy file (bỏ folder)
        .forEach(System.out::println);
}
// Duyệt toàn bộ từ thư mục hiện tại xuống tận cùng

// === walk() với giới hạn độ sâu ===
try (Stream<Path> walk = Files.walk(Path.of("."), 2)) { // Tối đa 2 cấp
    walk.forEach(System.out::println);
}

// === find(): Tìm kiếm có điều kiện ===
try (Stream<Path> found = Files.find(
        Path.of("."),                                   // Thư mục bắt đầu
        10,                                             // Độ sâu tối đa
        (path, attr) -> path.toString().endsWith(".java")  // Điều kiện
                        && attr.size() > 1000           // File > 1KB
    )) {
    found.forEach(System.out::println);
}

// === Ví dụ thực tế: Tính tổng dung lượng thư mục ===
try (Stream<Path> walk = Files.walk(Path.of("project"))) {
    long totalSize = walk
        .filter(Files::isRegularFile)                  // Chỉ lấy file
        .mapToLong(path -> {
            try {
                return Files.size(path);               // Lấy kích thước
            } catch (IOException e) {
                return 0L;
            }
        })
        .sum();                                        // Tổng dung lượng

    System.out.printf("Tổng dung lượng: %.2f MB%n", totalSize / 1_048_576.0);
}

// === Copy thư mục đệ quy ===
public static void copyDirectory(Path source, Path target) throws IOException {
    try (Stream<Path> walk = Files.walk(source)) {
        walk.forEach(sourcePath -> {
            Path targetPath = target.resolve(source.relativize(sourcePath));
            try {
                if (Files.isDirectory(sourcePath)) {
                    Files.createDirectories(targetPath);   // Tạo folder
                } else {
                    Files.copy(sourcePath, targetPath,
                        StandardCopyOption.REPLACE_EXISTING); // Copy file
                }
            } catch (IOException e) {
                throw new RuntimeException("Lỗi copy: " + sourcePath, e);
            }
        });
    }
}
```

---

## 6. Serialization (Tuần Tự Hóa Đối Tượng)

> **Ví dụ đời thường**: Giống như **đông lạnh thức ăn** để bảo quản (serialize), rồi sau đó **rã đông** để dùng lại (deserialize).
>
> - **Serialize**: Chuyển Object trong bộ nhớ → bytes để lưu file hoặc gửi qua mạng
> - **Deserialize**: Chuyển bytes → Object để dùng lại

```java
// === Bước 1: Class phải implements Serializable ===
public class Person implements Serializable {
    // serialVersionUID = "phiên bản" của class
    // Nếu thay đổi class → thay đổi UID → tránh lỗi khi deserialize bản cũ
    private static final long serialVersionUID = 1L;

    private String name;
    private int age;
    private transient String password;  // transient = KHÔNG serialize trường này
                                        // (thông tin nhạy cảm, không cần lưu)
}

// === Bước 2: Serialize - "Đông lạnh" object → file ===
Person person = new Person("John", 25);
try (ObjectOutputStream oos = new ObjectOutputStream(
        Files.newOutputStream(Path.of("person.dat")))) {
    oos.writeObject(person);                           // Ghi object → file binary
}

// === Bước 3: Deserialize - "Rã đông" file → object ===
try (ObjectInputStream ois = new ObjectInputStream(
        Files.newInputStream(Path.of("person.dat")))) {
    Person loaded = (Person) ois.readObject();         // Đọc object từ file
    System.out.println(loaded.getName());              // "John"
    System.out.println(loaded.getPassword());          // null! (vì transient)
}
```

```
Serialization Flow:

  ┌─────────────┐     writeObject()     ┌──────────────┐
  │ Java Object │  ──────────────────►  │  File (.dat)  │
  │ name="John" │     Serialize         │  [binary data]│
  │ age=25      │     (đông lạnh)       │               │
  └─────────────┘                       └──────────────┘
                                               │
                                         readObject()
                                         Deserialize
                                         (rã đông)
                                               │
                                               ▼
                                        ┌─────────────┐
                                        │ Java Object │
                                        │ name="John" │
                                        │ age=25      │
                                        └─────────────┘

  ⚠️ Ngày nay, Serialization ít dùng. Thay vào đó dùng JSON/XML:
  → Jackson (ObjectMapper) cho JSON
  → JAXB cho XML
  → Protobuf cho binary hiệu quả
```

---

## 7. Properties Files (File Cấu Hình)

> **Ví dụ đời thường**: Giống như **danh sách cài đặt** của ứng dụng. Thay vì hard-code giá trị trong source code, ta lưu trong file `.properties` để dễ thay đổi mà KHÔNG cần compile lại.

### File config.properties:
```properties
# Đây là comment
app.name=PortX
app.version=1.0
db.host=localhost
db.port=3306
db.user=admin
db.password=secret123
```

### Đọc và ghi Properties:

```java
Properties props = new Properties();

// === Đọc (Load) ===
try (InputStream is = Files.newInputStream(Path.of("config.properties"))) {
    props.load(is);                                    // Load toàn bộ properties
}

// === Lấy giá trị ===
String appName = props.getProperty("app.name");                  // "PortX"
String dbPort = props.getProperty("db.port");                    // "3306"
String missing = props.getProperty("not.exist");                 // null
String withDefault = props.getProperty("not.exist", "default");  // "default"

// === Đặt giá trị ===
props.setProperty("app.version", "2.0");             // Cập nhật giá trị
props.setProperty("new.key", "new.value");           // Thêm key mới

// === Ghi (Save) ===
try (OutputStream os = Files.newOutputStream(Path.of("config.properties"))) {
    props.store(os, "Updated Configuration");        // Comment header
}

// === Đọc từ classpath (trong JAR) ===
// Cách phổ biến trong Spring/enterprise app
try (InputStream is = getClass().getClassLoader()
        .getResourceAsStream("config.properties")) {
    if (is != null) {
        props.load(is);
    }
}
```

---

## 8. Sai Lầm Thường Gặp

### ❌ Sai lầm 1: Quên đóng resource (Resource Leak)

```java
// ❌ SAI: Không đóng reader → resource leak (rò rỉ tài nguyên)
BufferedReader reader = Files.newBufferedReader(Path.of("data.txt"));
String line = reader.readLine();
// Nếu exception xảy ra ở đây → reader KHÔNG BAO GIỜ được đóng!

// ✅ ĐÚNG: Dùng try-with-resources (TỰ ĐỘNG đóng)
try (BufferedReader reader = Files.newBufferedReader(Path.of("data.txt"))) {
    String line = reader.readLine();
}
// reader tự động đóng khi ra khỏi try block (kể cả khi có exception)

// ✅ ĐÚNG: Nhiều resources
try (InputStream is = Files.newInputStream(Path.of("input.bin"));
     OutputStream os = Files.newOutputStream(Path.of("output.bin"))) {
    // Cả 2 đều tự động đóng
    is.transferTo(os);                   // Copy nội dung (Java 9+)
}
```

### ❌ Sai lầm 2: Đọc file lớn bằng readAllLines/readString

```java
// ❌ SAI: File 2GB → OutOfMemoryError!
List<String> lines = Files.readAllLines(Path.of("huge-log.txt"));
// Đọc TOÀN BỘ file vào RAM → crash nếu file quá lớn

// ✅ ĐÚNG: Dùng Stream/BufferedReader cho file lớn
try (Stream<String> lines = Files.lines(Path.of("huge-log.txt"))) {
    long errorCount = lines
        .filter(line -> line.contains("ERROR"))
        .count();
    // Đọc từng dòng, xử lý xong giải phóng → dùng ít RAM
}

// 💡 Quy tắc:
// File < 10MB   → readAllLines(), readString() OK
// File > 10MB   → dùng Files.lines() hoặc BufferedReader
// File > 100MB  → cân nhắc xử lý song song hoặc chia nhỏ
```

### ❌ Sai lầm 3: Hard-code đường dẫn với separator sai OS

```java
// ❌ SAI: Hard-code backslash → chỉ chạy trên Windows!
Path path = Path.of("C:\\Users\\Documents\\data.txt");

// ❌ SAI: Nối String thủ công
String filePath = directory + "/" + filename;     // Có thể sai trên Windows

// ✅ ĐÚNG: Dùng Path.resolve() - tự xử lý separator
Path path = Path.of("C:", "Users", "Documents", "data.txt");  // Tự thêm separator
Path fullPath = baseDir.resolve("subdir").resolve("data.txt"); // Nối an toàn

// ✅ ĐÚNG: Dùng File.separator nếu buộc phải dùng String
String filePath = directory + File.separator + filename;
```

### ❌ Sai lầm 4: Không xử lý encoding (mã hóa ký tự)

```java
// ❌ SAI: Không chỉ định encoding → dùng encoding mặc định của OS
// Trên Windows: Cp1252, trên Linux: UTF-8 → kết quả khác nhau!
List<String> lines = Files.readAllLines(Path.of("tieng-viet.txt"));

// ✅ ĐÚNG: Luôn chỉ định UTF-8 cho file có tiếng Việt/Unicode
List<String> lines = Files.readAllLines(
    Path.of("tieng-viet.txt"),
    StandardCharsets.UTF_8
);

Files.writeString(
    Path.of("output.txt"),
    "Xin chào Việt Nam!",
    StandardCharsets.UTF_8
);
```

---

## 9. Tóm Tắt Cuối Ngày

| Khái niệm | Giải thích | Ví dụ |
|------------|-----------|-------|
| **java.io.File** | Class cũ (legacy) đại diện file/folder | `new File("data.txt")` |
| **Path** | Đường dẫn file (NIO.2, nên dùng) | `Path.of("data.txt")` |
| **Files** | Bộ công cụ thao tác file (NIO.2) | `Files.readString(path)` |
| **readAllLines()** | Đọc hết file → List<String> | File nhỏ |
| **readString()** | Đọc hết file → String (Java 11+) | File nhỏ |
| **BufferedReader** | Đọc từng dòng, tiết kiệm RAM | File lớn |
| **Files.lines()** | Đọc file → Stream<String> | File lớn + Stream API |
| **Files.write()** | Ghi List/bytes ra file | `Files.write(path, lines)` |
| **writeString()** | Ghi String ra file (Java 11+) | `Files.writeString(path, text)` |
| **BufferedWriter** | Ghi từng phần, hiệu quả | File lớn / nhiều lần ghi |
| **InputStream** | Đọc file binary (bytes) | Ảnh, PDF, ZIP |
| **OutputStream** | Ghi file binary (bytes) | Ảnh, PDF, ZIP |
| **Files.walk()** | Duyệt cây thư mục đệ quy | Tìm tất cả file .java |
| **Files.find()** | Tìm file theo điều kiện | File > 1MB, đuôi .log |
| **Serializable** | Chuyển Object ↔ bytes | Lưu/đọc object từ file |
| **Properties** | Đọc/ghi file cấu hình .properties | `props.getProperty("key")` |
| **try-with-resources** | Tự động đóng resource | BẮT BUỘC khi dùng I/O |

---

## 10. Câu Hỏi Phỏng Vấn Thường Gặp

### 🔥 Câu 1: java.io.File khác java.nio.file.Path/Files như thế nào?
**Trả lời:**
- `java.io.File` (Java 1.0): Class cũ, ít method, xử lý lỗi kém (trả boolean thay vì throw exception), không hỗ trợ symbolic links, file attributes
- `java.nio.file.Path + Files` (Java 7+): API hiện đại, nhiều method tiện ích (readString, walk, find...), exception rõ ràng, hỗ trợ symbolic links, file attributes, atomic operations
- **Khuyến khích:** Dùng NIO.2 cho code mới. Hiểu java.io.File để đọc code legacy

### 🔥 Câu 2: Byte Stream khác Character Stream như thế nào?
**Trả lời:**
- **Byte Stream** (InputStream/OutputStream): Xử lý dữ liệu dạng bytes (8-bit). Dùng cho file binary (ảnh, video, PDF). Không xử lý encoding
- **Character Stream** (Reader/Writer): Xử lý dữ liệu dạng ký tự (16-bit Unicode). Dùng cho file text. Tự động xử lý encoding (UTF-8, ASCII...)
- **Quy tắc**: Text → Reader/Writer, Binary → InputStream/OutputStream

### 🔥 Câu 3: try-with-resources hoạt động thế nào?
**Trả lời:**
Tự động đóng resource khi kết thúc try block. Resource phải implements `AutoCloseable` (có method `close()`). Compiler tự sinh code gọi `close()` trong finally block. Nếu cả try và close đều throw exception, exception trong close sẽ bị "suppressed" (đính kèm vào exception chính).

### 🔥 Câu 4: Tại sao Serializable cần serialVersionUID?
**Trả lời:**
`serialVersionUID` là "phiên bản" của class. Khi deserialize, JVM so sánh UID trong file với UID trong class hiện tại. Nếu khác nhau → `InvalidClassException`. Nếu không khai báo, JVM tự tính UID dựa trên structure của class → thay đổi nhỏ (thêm method) cũng gây incompatible. Khai báo explicit UID giúp kiểm soát backward compatibility.

### 🔥 Câu 5: Files.readAllLines() có vấn đề gì với file lớn?
**Trả lời:**
`readAllLines()` đọc TOÀN BỘ nội dung file vào bộ nhớ (RAM) dưới dạng `List<String>`. File 2GB → cần ít nhất 2GB RAM + overhead → `OutOfMemoryError`. Giải pháp: dùng `Files.lines()` (Stream, lazy loading) hoặc `BufferedReader` (đọc từng dòng) để xử lý file lớn mà không tốn nhiều RAM.

### 🔥 Câu 6: transient keyword dùng để làm gì?
**Trả lời:**
`transient` đánh dấu field KHÔNG được serialize. Khi `writeObject()`, field transient bị bỏ qua. Khi `readObject()`, field transient nhận giá trị mặc định (null cho Object, 0 cho int, false cho boolean). Dùng cho: thông tin nhạy cảm (password), giá trị tính toán được (cache), tài nguyên không thể serialize (Connection, Thread).

---

## Navigation

- [← Day 10: Stream API](./day-10-stream-api.md)
- [Day 12: Date/Time API →](./day-12-datetime-api.md)
