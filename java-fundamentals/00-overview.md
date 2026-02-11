# Java Fundamentals - Từ Cơ Bản Đến Nâng Cao

## Tổng quan khóa học

Khóa học **Java Core** hoàn chỉnh trong **19 ngày**, dành cho dev mới bắt đầu hoặc muốn nắm vững nền tảng Java.

### Tại sao học Java?

- **Phổ biến nhất thế giới**: Java liên tục nằm top 3 ngôn ngữ được dùng nhiều nhất (cùng Python, JavaScript)
- **Việc làm nhiều**: Hầu hết doanh nghiệp lớn (ngân hàng, logistics, e-commerce) đều dùng Java cho backend
- **Nền tảng vững chắc**: Học Java giúp bạn hiểu sâu về OOP (lập trình hướng đối tượng), memory management (quản lý bộ nhớ), multi-threading (đa luồng) - những kiến thức áp dụng được cho mọi ngôn ngữ khác
- **Hệ sinh thái khổng lồ**: Spring Boot, Hibernate, Maven, Gradle... rất nhiều framework và tool hỗ trợ

### Khóa học này dạy gì?

Bạn sẽ đi từ **"chưa biết gì về Java"** đến **"hiểu sâu cách Java hoạt động bên trong"**:

```
Tuần 1 (Day 1-7):  Nền tảng     → Biết viết chương trình Java cơ bản
Tuần 2 (Day 8-13): Trung cấp    → Biết dùng các tính năng mạnh của Java
Tuần 3 (Day 14-19): Nâng cao    → Hiểu cách Java chạy bên trong, viết code chuyên nghiệp
```

---

## Cấu trúc khóa học

### Phần 1: Java Cơ Bản (Day 1-7)

> Giai đoạn này bạn sẽ học nền tảng: cài đặt, viết code đầu tiên, hiểu OOP, xử lý lỗi, làm quen với các kiểu dữ liệu và collections.

| Day | Chủ đề | Bạn sẽ học được gì? |
|-----|--------|---------------------|
| 1 | [Setup & Syntax](./day-01-setup-syntax.md) | Cài Java, viết chương trình "Hello World" đầu tiên, biến (variable), kiểu dữ liệu (data type) |
| 2 | [Operators & Control Flow](./day-02-operators-control-flow.md) | Phép tính (+, -, *, /), câu điều kiện (if/switch), vòng lặp (for, while), mảng (array) |
| 3 | [OOP Basics](./day-03-oop-basics.md) | Lớp (Class), Đối tượng (Object), Hàm khởi tạo (Constructor), từ khóa `this`, thành viên tĩnh (static) |
| 4 | [OOP Pillars](./day-04-oop-pillars.md) | 4 trụ cột OOP: Kế thừa, Đa hình, Đóng gói, Trừu tượng |
| 5 | [Exception Handling](./day-05-exception-handling.md) | Xử lý lỗi (try/catch), ném lỗi (throw), tạo lỗi riêng (custom exception) |
| 6 | [Strings & Wrappers](./day-06-strings-wrappers.md) | Chuỗi ký tự (String), StringBuilder, đóng gói kiểu nguyên thủy (Autoboxing) |
| 7 | [Collections Basics](./day-07-collections-basics.md) | Danh sách (List), Tập hợp (Set), Bản đồ key-value (Map), duyệt phần tử (Iterator) |

### Phần 2: Java Trung Cấp (Day 8-13)

> Giai đoạn này bạn sẽ học các tính năng mạnh mẽ: Generics, Lambda, Stream API, File I/O, Date/Time, và bắt đầu với đa luồng.

| Day | Chủ đề | Bạn sẽ học được gì? |
|-----|--------|---------------------|
| 8 | [Generics](./day-08-generics.md) | Kiểu tổng quát - viết code dùng được cho nhiều kiểu dữ liệu khác nhau |
| 9 | [Lambda & Functional](./day-09-lambda-functional.md) | Biểu thức Lambda - viết hàm ngắn gọn kiểu "hàm ẩn danh" |
| 10 | [Stream API](./day-10-stream-api.md) | Xử lý dữ liệu dạng "dòng chảy": lọc (filter), biến đổi (map), gom (collect) |
| 11 | [File I/O](./day-11-file-io.md) | Đọc/ghi file, làm việc với thư mục, NIO.2 API hiện đại |
| 12 | [Date/Time API](./day-12-datetime-api.md) | Xử lý ngày giờ: LocalDate, LocalDateTime, tính khoảng cách thời gian |
| 13 | [Multithreading Basics](./day-13-multithreading-basics.md) | Đa luồng cơ bản: Thread, Runnable, đồng bộ hóa (synchronized) |

### Phần 3: Java Nâng Cao (Day 14-19)

> Giai đoạn này bạn sẽ hiểu sâu: lập trình bất đồng bộ, design patterns, quản lý bộ nhớ, và cách JVM hoạt động.

| Day | Chủ đề | Bạn sẽ học được gì? |
|-----|--------|---------------------|
| 14 | [Concurrency](./day-14-concurrency.md) | Quản lý luồng nâng cao: ExecutorService, Callable, Future |
| 15 | [CompletableFuture](./day-15-completable-future.md) | Lập trình bất đồng bộ (async) - gọi nhiều tác vụ song song |
| 16 | [Reflection & Annotations](./day-16-reflection-annotations.md) | "Soi gương" code: kiểm tra class lúc runtime, tạo annotation riêng |
| 17 | [Design Patterns](./day-17-design-patterns.md) | Mẫu thiết kế: Singleton, Factory, Builder, Strategy, Observer |
| 18 | [Memory & GC](./day-18-memory-gc.md) | Bộ nhớ Heap/Stack, Garbage Collection (dọn rác tự động), memory leak (rò rỉ bộ nhớ) |
| 19 | [JVM Internals](./day-19-jvm-internals.md) | Bên trong JVM: Classloader, Bytecode, JIT (biên dịch tức thời) |

---

## Yêu cầu trước khi bắt đầu

### Phần mềm cần cài

| Phần mềm | Phiên bản | Tại sao cần? |
|-----------|-----------|--------------|
| **JDK** (Java Development Kit) | Java 21 LTS | Bộ công cụ để biên dịch và chạy code Java |
| **IDE** (Môi trường phát triển) | IntelliJ IDEA Community (miễn phí) hoặc VS Code + Extension Pack for Java | Nơi viết, chạy và debug code |

### Kiến thức cần có trước

- **Biết dùng máy tính cơ bản**: cài phần mềm, dùng terminal/command prompt
- **Tư duy logic**: hiểu khái niệm điều kiện (nếu... thì...), lặp lại
- **Không cần biết trước ngôn ngữ nào**: khóa học bắt đầu từ zero

### Thời gian học

- **Mỗi ngày**: 2-3 giờ (đọc lý thuyết + làm bài tập)
- **Tổng**: ~19 ngày (~45-55 giờ)

---

## Cách học hiệu quả

### Quy trình mỗi ngày

```
1. Đọc phần "Tại sao cần học?"     → Hiểu MỤC ĐÍCH trước
2. Đọc lý thuyết + ví dụ đời thường → Hiểu KHÁI NIỆM
3. Tự gõ code (KHÔNG copy/paste)    → THỰC HÀNH tay
4. Chạy thử + sửa lỗi (debug)      → HIỂU cách code chạy
5. Làm bài tập cuối ngày            → KIỂM TRA kiến thức
6. Đọc phần "Sai lầm thường gặp"   → TRÁNH lỗi phổ biến
```

### Mẹo quan trọng

- **Tự gõ code, đừng copy/paste**: Gõ tay giúp não nhớ lâu hơn gấp 3 lần
- **Chạy code SAI trước**: Cố tình viết sai để xem lỗi gì xảy ra → hiểu sâu hơn
- **Debug từng dòng**: Dùng breakpoint trong IDE để xem code chạy từng bước
- **Ghi chú bằng tiếng Việt**: Comment giải thích bằng tiếng Việt cho dễ nhớ
- **Hỏi "Tại sao?"**: Mỗi khi gặp khái niệm mới, hãy tự hỏi "Tại sao Java thiết kế như vậy?"

---

## Ký hiệu trong tài liệu

Bạn sẽ gặp các ký hiệu sau xuyên suốt khóa học:

| Ký hiệu | Ý nghĩa |
|----------|---------|
| ✅ | Code đúng, nên làm theo |
| ❌ | Code sai hoặc không nên làm |
| 💡 | Mẹo nhớ / tip hữu ích |
| ⚠️ | Cẩn thận, dễ nhầm lẫn |
| 🔥 | Kiến thức quan trọng, hay gặp trong phỏng vấn |

---

## Bắt đầu thôi!

➡️ [Day 1: Setup & Syntax - Cài đặt và viết chương trình đầu tiên](./day-01-setup-syntax.md)
