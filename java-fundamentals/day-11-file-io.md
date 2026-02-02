# Day 11: File I/O

## Mục tiêu
- java.io package
- java.nio.file (NIO.2)
- Reading và Writing files
- Working with directories

---

## 1. File Class (java.io)

```java
import java.io.File;

// Tạo File object
File file = new File("data.txt");
File file2 = new File("C:/Users/Documents/data.txt");
File dir = new File("myFolder");

// Properties
file.getName();           // "data.txt"
file.getPath();           // "data.txt"
file.getAbsolutePath();   // Full path
file.getParent();         // Parent directory
file.length();            // Size in bytes

// Checks
file.exists();
file.isFile();
file.isDirectory();
file.canRead();
file.canWrite();
file.isHidden();

// Operations
file.createNewFile();     // Create file
dir.mkdir();              // Create directory
dir.mkdirs();             // Create with parents
file.delete();            // Delete
file.renameTo(new File("newname.txt"));

// List directory
File folder = new File(".");
String[] names = folder.list();
File[] files = folder.listFiles();
File[] txtFiles = folder.listFiles((d, name) -> name.endsWith(".txt"));
```

---

## 2. Path và Files (NIO.2)

### 2.1. Path

```java
import java.nio.file.Path;
import java.nio.file.Paths;

// Tạo Path
Path path = Paths.get("data.txt");
Path path2 = Paths.get("C:", "Users", "Documents", "data.txt");
Path path3 = Path.of("data.txt");  // Java 11+

// Properties
path.getFileName();       // data.txt
path.getParent();         // Parent path
path.getRoot();           // Root (e.g., C:\)
path.toAbsolutePath();    // Full path
path.normalize();         // Remove . and ..

// Operations
path.resolve("subdir");          // Append
path.resolve("../other.txt");    // Relative
path.relativize(path2);          // Get relative path
path.startsWith("C:");
path.endsWith(".txt");

// Iterate path elements
for (Path p : path) {
    System.out.println(p);
}
```

### 2.2. Files Utility

```java
import java.nio.file.Files;

Path path = Path.of("data.txt");

// Checks
Files.exists(path);
Files.isRegularFile(path);
Files.isDirectory(path);
Files.isReadable(path);
Files.isWritable(path);
Files.size(path);  // Size in bytes

// Create
Files.createFile(path);
Files.createDirectory(Path.of("newDir"));
Files.createDirectories(Path.of("a/b/c"));
Files.createTempFile("prefix", ".tmp");

// Copy/Move/Delete
Files.copy(source, target);
Files.copy(source, target, StandardCopyOption.REPLACE_EXISTING);
Files.move(source, target);
Files.delete(path);
Files.deleteIfExists(path);

// Attributes
FileTime lastModified = Files.getLastModifiedTime(path);
Files.setLastModifiedTime(path, FileTime.fromMillis(System.currentTimeMillis()));
```

---

## 3. Reading Files

### 3.1. Read All Content

```java
// Read all lines
List<String> lines = Files.readAllLines(Path.of("data.txt"));
List<String> linesUtf8 = Files.readAllLines(Path.of("data.txt"), StandardCharsets.UTF_8);

// Read all bytes
byte[] bytes = Files.readAllBytes(Path.of("data.txt"));

// Read as String (Java 11+)
String content = Files.readString(Path.of("data.txt"));
```

### 3.2. Buffered Reading

```java
// BufferedReader
try (BufferedReader reader = Files.newBufferedReader(Path.of("data.txt"))) {
    String line;
    while ((line = reader.readLine()) != null) {
        System.out.println(line);
    }
}

// With Stream
try (Stream<String> lines = Files.lines(Path.of("data.txt"))) {
    lines.filter(line -> line.contains("keyword"))
         .forEach(System.out::println);
}
```

### 3.3. InputStream

```java
try (InputStream is = Files.newInputStream(Path.of("data.bin"))) {
    byte[] buffer = new byte[1024];
    int bytesRead;
    while ((bytesRead = is.read(buffer)) != -1) {
        // Process bytes
    }
}

// With BufferedInputStream
try (BufferedInputStream bis = new BufferedInputStream(
        Files.newInputStream(Path.of("data.bin")))) {
    // Read
}
```

---

## 4. Writing Files

### 4.1. Write All Content

```java
// Write lines
List<String> lines = Arrays.asList("Line 1", "Line 2", "Line 3");
Files.write(Path.of("output.txt"), lines);

// Write bytes
byte[] data = "Hello World".getBytes();
Files.write(Path.of("output.txt"), data);

// Write String (Java 11+)
Files.writeString(Path.of("output.txt"), "Hello World");

// Append
Files.write(Path.of("output.txt"), lines, StandardOpenOption.APPEND);
```

### 4.2. Buffered Writing

```java
// BufferedWriter
try (BufferedWriter writer = Files.newBufferedWriter(Path.of("output.txt"))) {
    writer.write("Line 1");
    writer.newLine();
    writer.write("Line 2");
}

// PrintWriter
try (PrintWriter pw = new PrintWriter(Files.newBufferedWriter(Path.of("output.txt")))) {
    pw.println("Line 1");
    pw.printf("Name: %s, Age: %d%n", "John", 25);
}
```

### 4.3. OutputStream

```java
try (OutputStream os = Files.newOutputStream(Path.of("output.bin"))) {
    os.write("Hello".getBytes());
}

// With options
try (OutputStream os = Files.newOutputStream(Path.of("output.bin"),
        StandardOpenOption.CREATE,
        StandardOpenOption.APPEND)) {
    os.write("More data".getBytes());
}
```

---

## 5. Working with Directories

```java
// List directory
try (DirectoryStream<Path> stream = Files.newDirectoryStream(Path.of("."))) {
    for (Path entry : stream) {
        System.out.println(entry.getFileName());
    }
}

// With filter
try (DirectoryStream<Path> stream = Files.newDirectoryStream(
        Path.of("."), "*.txt")) {
    for (Path entry : stream) {
        System.out.println(entry);
    }
}

// Walk directory tree
try (Stream<Path> walk = Files.walk(Path.of("."))) {
    walk.filter(Files::isRegularFile)
        .forEach(System.out::println);
}

// Find files
try (Stream<Path> found = Files.find(Path.of("."), 10,
        (path, attr) -> path.toString().endsWith(".java"))) {
    found.forEach(System.out::println);
}

// Copy directory recursively
public static void copyDirectory(Path source, Path target) throws IOException {
    Files.walk(source).forEach(sourcePath -> {
        Path targetPath = target.resolve(source.relativize(sourcePath));
        try {
            Files.copy(sourcePath, targetPath, StandardCopyOption.REPLACE_EXISTING);
        } catch (IOException e) {
            throw new RuntimeException(e);
        }
    });
}
```

---

## 6. Serialization

```java
// Implement Serializable
public class Person implements Serializable {
    private static final long serialVersionUID = 1L;
    private String name;
    private transient String password;  // Won't be serialized
}

// Serialize
try (ObjectOutputStream oos = new ObjectOutputStream(
        Files.newOutputStream(Path.of("person.dat")))) {
    oos.writeObject(new Person("John"));
}

// Deserialize
try (ObjectInputStream ois = new ObjectInputStream(
        Files.newInputStream(Path.of("person.dat")))) {
    Person person = (Person) ois.readObject();
}
```

---

## 7. Properties Files

```java
Properties props = new Properties();

// Load
try (InputStream is = Files.newInputStream(Path.of("config.properties"))) {
    props.load(is);
}

// Read
String value = props.getProperty("key");
String valueDefault = props.getProperty("key", "default");

// Set
props.setProperty("key", "value");

// Save
try (OutputStream os = Files.newOutputStream(Path.of("config.properties"))) {
    props.store(os, "Configuration");
}
```

---

## 8. Bài tập thực hành

### Bài 1: File Statistics
Đọc file và thống kê: số dòng, số từ, số ký tự.

### Bài 2: Directory Size
Tính tổng size của tất cả files trong directory (recursive).

### Bài 3: File Search
Tìm tất cả files có extension cụ thể trong directory tree.

### Bài 4: Log File Analyzer
Parse log file và extract thông tin (errors, warnings, timestamps).

### Bài 5: CSV Reader/Writer
Đọc và ghi CSV files với header.

---

## Đáp án tham khảo

<details>
<summary>Bài 1: File Statistics</summary>

```java
public class FileStats {
    public static void analyze(Path path) throws IOException {
        long lines = 0, words = 0, chars = 0;

        try (BufferedReader reader = Files.newBufferedReader(path)) {
            String line;
            while ((line = reader.readLine()) != null) {
                lines++;
                chars += line.length();
                words += line.split("\\s+").length;
            }
        }

        System.out.printf("Lines: %d, Words: %d, Characters: %d%n",
            lines, words, chars);
    }
}
```
</details>

---

## Navigation

- [← Day 10: Stream API](./day-10-stream-api.md)
- [Day 12: Date/Time API →](./day-12-datetime-api.md)
