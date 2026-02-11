# Day 17: Design Patterns (Mẫu Thiết Kế)

## Mục tiêu hôm nay
- Creational Patterns (mẫu khởi tạo): Singleton, Factory, Builder
- Structural Patterns (mẫu cấu trúc): Adapter, Decorator
- Behavioral Patterns (mẫu hành vi): Strategy, Observer, Template Method
- Mỗi pattern: Vấn đề gì nó giải quyết? Khi nào dùng? Ví dụ thực tế?

---

## 🤔 Tại sao cần học Design Patterns?

> **Design Patterns** (mẫu thiết kế) là **giải pháp đã được kiểm chứng** cho các vấn đề thiết kế phần mềm lặp đi lặp lại. Giống như **công thức nấu ăn** - bạn không cần phát minh lại từ đầu.

```
┌─────────────────────────────────────────────────────────────────┐
│                 3 NHÓM DESIGN PATTERNS                          │
├──────────────────────┬──────────────┬───────────────────────────┤
│ Creational           │ Structural   │ Behavioral                │
│ (Khởi tạo)          │ (Cấu trúc)  │ (Hành vi)                 │
├──────────────────────┼──────────────┼───────────────────────────┤
│ Cách TẠO object     │ Cách TỔ CHỨC│ Cách TƯƠNG TÁC            │
│                      │ class/object│ giữa objects               │
├──────────────────────┼──────────────┼───────────────────────────┤
│ • Singleton          │ • Adapter   │ • Strategy                │
│ • Factory            │ • Decorator │ • Observer                │
│ • Builder            │ • Proxy     │ • Template Method         │
│ • Prototype          │ • Facade    │ • Command                 │
└──────────────────────┴──────────────┴───────────────────────────┘
```

---

## 1. Singleton (Chỉ Có 1 Instance Duy Nhất)

### Vấn đề nó giải quyết
> Có những thứ chỉ cần **đúng 1 cái** trong toàn bộ ứng dụng: database connection pool, logger, configuration manager. Nếu tạo nhiều → lãng phí tài nguyên hoặc xung đột.

### 3 cách implement

```java
// === Cách 1: Eager Initialization (tạo ngay khi class được load) ===
public class EagerSingleton {
    // Tạo instance ngay khi class load → đơn giản, thread-safe
    private static final EagerSingleton INSTANCE = new EagerSingleton();

    private EagerSingleton() {}    // Constructor private → không ai tạo được instance mới

    public static EagerSingleton getInstance() {
        return INSTANCE;           // Luôn trả về cùng 1 instance
    }
}
// 💡 Ưu: Đơn giản, thread-safe
// ⚠️ Nhược: Tạo instance ngay cả khi chưa dùng (tốn bộ nhớ nếu khởi tạo nặng)

// === Cách 2: Lazy Initialization + Double-Check Locking ===
public class LazySingleton {
    // volatile đảm bảo visibility giữa các thread
    private static volatile LazySingleton instance;

    private LazySingleton() {}

    public static LazySingleton getInstance() {
        if (instance == null) {                    // Check 1: Nhanh, không lock
            synchronized (LazySingleton.class) {   // Lock chỉ khi instance chưa tạo
                if (instance == null) {            // Check 2: Sau khi lock, kiểm tra lại
                    instance = new LazySingleton(); // Chỉ tạo 1 lần
                }
            }
        }
        return instance;
    }
}
// 💡 Ưu: Lazy (chỉ tạo khi cần), thread-safe
// ⚠️ Nhược: Code phức tạp

// === Cách 3: Enum Singleton (KHUYÊN DÙNG) ===
public enum EnumSingleton {
    INSTANCE;                      // Java enum đảm bảo chỉ có 1 instance

    private int counter = 0;

    public void doSomething() {
        counter++;
        System.out.println("Counter: " + counter);
    }
}

// Sử dụng:
EnumSingleton.INSTANCE.doSomething();
// 💡 Ưu: Thread-safe, serialization-safe, đơn giản nhất
// ⚠️ Nhược: Không thể lazy initialization, không kế thừa class khác
```

> **💡 Mẹo nhớ**: Singleton = "Vua chỉ có 1 người". Private constructor = "Cấm ai tự xưng vua". getInstance() = "Chỉ có 1 cách gặp vua".

---

## 2. Factory (Nhà Máy Sản Xuất Object)

### Vấn đề nó giải quyết
> Khi bạn cần tạo object nhưng **không muốn client biết** chính xác class nào được tạo. Client nói "tôi muốn hình tròn" → Factory tạo Circle. Logic tạo object được **tập trung 1 chỗ**.

### Simple Factory

```java
// Interface chung
public interface Shape {
    void draw();
}

// Các implementation cụ thể
public class Circle implements Shape {
    public void draw() { System.out.println("Vẽ hình tròn ⭕"); }
}
public class Rectangle implements Shape {
    public void draw() { System.out.println("Vẽ hình chữ nhật 🟦"); }
}

// Factory: "Nhà máy" sản xuất Shape
public class ShapeFactory {
    public static Shape create(String type) {
        return switch (type) {
            case "circle" -> new Circle();
            case "rectangle" -> new Rectangle();
            default -> throw new IllegalArgumentException("Không biết loại: " + type);
        };
    }
}

// Sử dụng:
Shape shape = ShapeFactory.create("circle");   // Client KHÔNG cần biết Circle class
shape.draw();                                   // "Vẽ hình tròn ⭕"

// 💡 Lợi ích:
// → Thêm Triangle? Chỉ sửa ShapeFactory, client code KHÔNG thay đổi
// → Logic tạo object tập trung 1 chỗ → dễ maintain
```

### Factory Method (Mỗi subclass tự quyết định tạo gì)

```java
// Ví dụ: Dialog framework - mỗi OS tạo button khác nhau
public abstract class Dialog {
    public void render() {
        Button button = createButton();        // Gọi factory method
        button.render();
    }

    protected abstract Button createButton();  // Subclass quyết định tạo button nào
}

public class WindowsDialog extends Dialog {
    @Override
    protected Button createButton() {
        return new WindowsButton();            // Windows tạo WindowsButton
    }
}

public class MacDialog extends Dialog {
    @Override
    protected Button createButton() {
        return new MacButton();                // Mac tạo MacButton
    }
}

// Sử dụng:
Dialog dialog = isWindows ? new WindowsDialog() : new MacDialog();
dialog.render();   // Tự động dùng button đúng OS
```

---

## 3. Builder (Xây Dựng Object Phức Tạp Từng Bước)

### Vấn đề nó giải quyết
> Object có **quá nhiều tham số** (6-7 tham số trong constructor) → dễ nhầm thứ tự, khó đọc. Builder cho phép **xây từng bước**, chỉ set những field cần thiết.

```java
// ❌ KHÔNG có Builder: Constructor dài, dễ nhầm tham số
User user = new User("John", 25, "john@email.com", "0123456789", "VN", true);
//                    ↑       ↑     ↑                ↑             ↑     ↑
//                   name   age    email             phone         country active
// Dễ nhầm phone với country! Không biết true là gì!

// ✅ CÓ Builder: Rõ ràng, đọc như tiếng Anh
User user = User.builder()
    .name("John")
    .age(25)
    .email("john@email.com")
    .phone("0123456789")
    .build();
// Rõ ràng từng field, bỏ qua field không cần (country, active dùng default)
```

### Implementation

```java
public class User {
    private final String name;         // final → immutable sau khi build
    private final int age;
    private final String email;
    private final String phone;

    private User(Builder builder) {    // Constructor PRIVATE → chỉ Builder tạo được
        this.name = builder.name;
        this.age = builder.age;
        this.email = builder.email;
        this.phone = builder.phone;
    }

    public static Builder builder() {
        return new Builder();
    }

    public static class Builder {
        private String name;
        private int age;
        private String email;
        private String phone;

        public Builder name(String name) {
            this.name = name;
            return this;               // return this → cho phép chaining: .name().age()
        }
        public Builder age(int age) {
            this.age = age;
            return this;
        }
        public Builder email(String email) {
            this.email = email;
            return this;
        }
        public Builder phone(String phone) {
            this.phone = phone;
            return this;
        }
        public User build() {
            // Có thể thêm validation ở đây
            if (name == null) throw new IllegalStateException("Name is required");
            return new User(this);     // Tạo User từ Builder
        }
    }
}

// 💡 Trong thực tế: Dùng @Builder annotation từ Lombok → tự generate toàn bộ code trên!
// @Builder
// public class User { String name; int age; String email; }
```

---

## 4. Adapter (Bộ Chuyển Đổi)

### Vấn đề nó giải quyết
> Bạn có **2 interface không tương thích** nhưng cần làm việc cùng nhau. Giống như **ổ chuyển điện** - ổ Mỹ (2 chân dẹt) cắm vào ổ Việt (2 chân tròn) không được → cần adapter.

```java
// Interface mà hệ thống của bạn dùng
public interface MediaPlayer {
    void play(String filename);
}

// Class bên thứ 3 (không thể sửa code) - interface KHÁC
public class VlcPlayer {
    public void playVlc(String filename) {
        System.out.println("Playing VLC: " + filename);
    }
}

// Adapter: Chuyển đổi interface VlcPlayer → MediaPlayer
public class VlcAdapter implements MediaPlayer {
    private VlcPlayer vlcPlayer = new VlcPlayer();     // Chứa object cần adapt

    @Override
    public void play(String filename) {
        vlcPlayer.playVlc(filename);                    // Gọi method đúng tên
    }
}

// Sử dụng: Code client chỉ biết MediaPlayer interface
MediaPlayer player = new VlcAdapter();
player.play("movie.vlc");   // Adapter chuyển thành vlcPlayer.playVlc()

// 💡 Ví dụ thực tế trong Java:
// Arrays.asList() → Adapter: Array → List
// InputStreamReader → Adapter: InputStream (bytes) → Reader (chars)
```

---

## 5. Decorator (Bổ Sung Tính Năng)

### Vấn đề nó giải quyết
> Muốn **thêm tính năng** cho object mà **không sửa class gốc**. Giống như **thêm topping** cho kem: kem vanilla + chocolate sauce + sprinkles.

```java
// Interface gốc
public interface Coffee {
    double getCost();
    String getDescription();
}

// Class cơ bản
public class SimpleCoffee implements Coffee {
    public double getCost() { return 2.0; }
    public String getDescription() { return "Cà phê đen"; }
}

// Base Decorator
public abstract class CoffeeDecorator implements Coffee {
    protected Coffee coffee;               // Chứa object được decorate
    public CoffeeDecorator(Coffee coffee) { this.coffee = coffee; }
}

// Decorator: Thêm sữa
public class MilkDecorator extends CoffeeDecorator {
    public MilkDecorator(Coffee coffee) { super(coffee); }
    public double getCost() { return coffee.getCost() + 0.5; }              // Cộng thêm giá sữa
    public String getDescription() { return coffee.getDescription() + " + Sữa"; }
}

// Decorator: Thêm đường
public class SugarDecorator extends CoffeeDecorator {
    public SugarDecorator(Coffee coffee) { super(coffee); }
    public double getCost() { return coffee.getCost() + 0.3; }
    public String getDescription() { return coffee.getDescription() + " + Đường"; }
}

// Sử dụng: "Bọc" nhiều lớp decorator
Coffee coffee = new SimpleCoffee();                    // Cà phê đen: $2.0
coffee = new MilkDecorator(coffee);                    // + Sữa: $2.5
coffee = new SugarDecorator(coffee);                   // + Đường: $2.8

System.out.println(coffee.getDescription());           // "Cà phê đen + Sữa + Đường"
System.out.println("Giá: $" + coffee.getCost());       // "Giá: $2.8"

// 💡 Ví dụ thực tế trong Java:
// BufferedReader(new FileReader(file))  → Decorator thêm buffer cho FileReader
// Collections.synchronizedList(list)    → Decorator thêm thread-safety cho List
```

```
Decorator hoạt động thế nào (bọc lồng):

  ┌─────────────────────────────────────┐
  │ SugarDecorator     getCost()        │
  │  ┌─────────────────────────────┐    │
  │  │ MilkDecorator   getCost()   │    │   sugar.getCost()
  │  │  ┌─────────────────────┐    │    │   = milk.getCost() + 0.3
  │  │  │ SimpleCoffee        │    │    │   = (simple.getCost() + 0.5) + 0.3
  │  │  │ getCost() = 2.0    │    │    │   = (2.0 + 0.5) + 0.3
  │  │  └─────────────────────┘    │    │   = 2.8
  │  └─────────────────────────────┘    │
  └─────────────────────────────────────┘
```

---

## 6. Strategy (Chiến Lược Thay Đổi Được)

### Vấn đề nó giải quyết
> Cùng 1 hành động nhưng có **nhiều cách thực hiện**. Thay vì dùng if/else để chọn → tách mỗi cách thành 1 class riêng, có thể **thay đổi tại runtime**.

```java
// Strategy interface: Định nghĩa hành động chung
public interface PaymentStrategy {
    void pay(double amount);
}

// Concrete Strategy 1: Thanh toán bằng thẻ tín dụng
public class CreditCardPayment implements PaymentStrategy {
    private String cardNumber;

    public CreditCardPayment(String cardNumber) {
        this.cardNumber = cardNumber;
    }

    @Override
    public void pay(double amount) {
        System.out.println("Thanh toán $" + amount + " bằng thẻ: " + cardNumber);
    }
}

// Concrete Strategy 2: Thanh toán bằng MoMo
public class MomoPayment implements PaymentStrategy {
    private String phoneNumber;

    public MomoPayment(String phoneNumber) {
        this.phoneNumber = phoneNumber;
    }

    @Override
    public void pay(double amount) {
        System.out.println("Thanh toán $" + amount + " bằng MoMo: " + phoneNumber);
    }
}

// Context: Giỏ hàng sử dụng strategy
public class ShoppingCart {
    private PaymentStrategy paymentStrategy;     // Có thể thay đổi bất cứ lúc nào

    public void setPaymentStrategy(PaymentStrategy strategy) {
        this.paymentStrategy = strategy;
    }

    public void checkout(double amount) {
        paymentStrategy.pay(amount);
    }
}

// Sử dụng:
ShoppingCart cart = new ShoppingCart();
cart.setPaymentStrategy(new CreditCardPayment("1234-5678"));
cart.checkout(100);       // "Thanh toán $100 bằng thẻ: 1234-5678"

cart.setPaymentStrategy(new MomoPayment("0901234567"));
cart.checkout(50);        // "Thanh toán $50 bằng MoMo: 0901234567"

// 💡 Với Lambda (Java 8+), Strategy có thể đơn giản hơn:
cart.setPaymentStrategy(amount ->
    System.out.println("Thanh toán $" + amount + " bằng Bitcoin"));
cart.checkout(200);
```

> **💡 Mẹo nhớ**: Strategy = "Nhà hàng cho chọn cách thanh toán". Thêm cách mới (VNPay) → tạo class mới, KHÔNG sửa code cũ. Tuân thủ **Open-Closed Principle** (mở cho mở rộng, đóng cho sửa đổi).

---

## 7. Observer (Người Quan Sát)

### Vấn đề nó giải quyết
> Khi 1 object thay đổi → **tự động thông báo** cho tất cả object đang "theo dõi". Giống như **subscribe YouTube**: khi YouTuber đăng video mới → tất cả subscriber nhận thông báo.

```java
// Observer: Interface cho "người theo dõi"
public interface Observer {
    void update(String message);
}

// Subject: "Nguồn tin" - quản lý danh sách observers
public class NewsAgency {
    private List<Observer> observers = new ArrayList<>();
    private String latestNews;

    // Đăng ký theo dõi
    public void subscribe(Observer observer) {
        observers.add(observer);
    }

    // Hủy theo dõi
    public void unsubscribe(Observer observer) {
        observers.remove(observer);
    }

    // Khi có tin mới → thông báo tất cả observers
    public void publishNews(String news) {
        this.latestNews = news;
        System.out.println("📰 Tin mới: " + news);
        notifyAllObservers();
    }

    private void notifyAllObservers() {
        for (Observer observer : observers) {
            observer.update(latestNews);       // Gửi thông báo cho từng observer
        }
    }
}

// Concrete Observer: Kênh tin tức
public class NewsChannel implements Observer {
    private String channelName;

    public NewsChannel(String name) { this.channelName = name; }

    @Override
    public void update(String message) {
        System.out.println("  📺 " + channelName + " nhận tin: " + message);
    }
}

// Sử dụng:
NewsAgency agency = new NewsAgency();

NewsChannel vtv = new NewsChannel("VTV");
NewsChannel bbc = new NewsChannel("BBC");

agency.subscribe(vtv);      // VTV đăng ký
agency.subscribe(bbc);      // BBC đăng ký

agency.publishNews("Java 22 ra mắt!");
// Output:
// 📰 Tin mới: Java 22 ra mắt!
//   📺 VTV nhận tin: Java 22 ra mắt!
//   📺 BBC nhận tin: Java 22 ra mắt!

agency.unsubscribe(bbc);     // BBC hủy đăng ký

agency.publishNews("Spring Boot 4.0 released!");
// Output:
// 📰 Tin mới: Spring Boot 4.0 released!
//   📺 VTV nhận tin: Spring Boot 4.0 released!
// (BBC không nhận vì đã hủy)

// 💡 Ví dụ thực tế:
// → Event listeners trong GUI (button click → nhiều handlers)
// → Spring Events (@EventListener)
// → Message Queue (publish/subscribe pattern)
```

---

## 8. Template Method (Phương Thức Mẫu)

### Vấn đề nó giải quyết
> Nhiều class có **quy trình giống nhau** nhưng **chi tiết khác nhau**. Template Method định nghĩa **bộ khung** (skeleton), subclass **điền chi tiết**.

> **Ví dụ đời thường**: Nấu mì gói luôn theo quy trình: Đun nước → Cho mì vào → Thêm gia vị → Bày ra tô. Nhưng mỗi loại mì có gia vị khác nhau.

```java
// Abstract class: Định nghĩa quy trình (template)
public abstract class DataProcessor {

    // Template Method: Quy trình CỐ ĐỊNH (final → subclass không ghi đè được!)
    public final void process() {
        readData();           // Bước 1: Đọc dữ liệu (subclass tự định nghĩa)
        processData();        // Bước 2: Xử lý dữ liệu (subclass tự định nghĩa)
        saveData();           // Bước 3: Lưu dữ liệu (có sẵn default, có thể override)
    }

    protected abstract void readData();       // Subclass BẮT BUỘC implement
    protected abstract void processData();    // Subclass BẮT BUỘC implement

    protected void saveData() {               // Default implementation (có thể override)
        System.out.println("Lưu vào database...");
    }
}

// Subclass 1: Xử lý CSV
public class CsvProcessor extends DataProcessor {
    @Override
    protected void readData() { System.out.println("Đọc file CSV..."); }

    @Override
    protected void processData() { System.out.println("Parse CSV columns..."); }
}

// Subclass 2: Xử lý JSON
public class JsonProcessor extends DataProcessor {
    @Override
    protected void readData() { System.out.println("Đọc file JSON..."); }

    @Override
    protected void processData() { System.out.println("Parse JSON objects..."); }

    @Override
    protected void saveData() { System.out.println("Lưu vào MongoDB..."); }  // Override default
}

// Sử dụng:
DataProcessor csv = new CsvProcessor();
csv.process();
// Output:
// Đọc file CSV...
// Parse CSV columns...
// Lưu vào database...

DataProcessor json = new JsonProcessor();
json.process();
// Output:
// Đọc file JSON...
// Parse JSON objects...
// Lưu vào MongoDB...    ← Override default saveData()
```

---

## 9. Sai Lầm Thường Gặp

### ❌ Sai lầm 1: Lạm dụng Singleton

```java
// ❌ SAI: Biến mọi thứ thành Singleton
public enum UserServiceSingleton {   // UserService KHÔNG nên là Singleton!
    INSTANCE;
}

// 💡 Chỉ dùng Singleton khi:
// → Resource shared duy nhất (connection pool, thread pool)
// → Config/Settings đọc 1 lần dùng chung
// → Logger
// KHÔNG dùng cho: Service, Repository, DTO (dùng DI framework thay thế)
```

### ❌ Sai lầm 2: Dùng if/else thay vì Strategy

```java
// ❌ SAI: Mỗi lần thêm cách thanh toán → sửa if/else
public void pay(String method, double amount) {
    if (method.equals("credit")) {
        // Thanh toán thẻ...
    } else if (method.equals("momo")) {
        // Thanh toán MoMo...
    } else if (method.equals("vnpay")) {
        // Thanh toán VNPay... (cứ thêm mãi)
    }
}

// ✅ ĐÚNG: Strategy pattern → thêm cách mới = thêm class mới, KHÔNG sửa code cũ
cart.setPaymentStrategy(new VnPayPayment());  // Thêm class VnPayPayment → xong!
```

### ❌ Sai lầm 3: Over-engineering (Áp dụng pattern khi không cần)

```java
// ❌ SAI: Tạo Factory cho 1 class duy nhất
public class UserFactory {
    public static User create() { return new User(); }
}
// Chỉ có 1 loại User → Factory thừa, gọi new User() là đủ!

// 💡 Quy tắc: Chỉ áp dụng pattern khi:
// → Có vấn đề CỤ THỂ cần giải quyết
// → Có > 2 biến thể (nhiều loại Shape → Factory, nhiều cách pay → Strategy)
// → Code sẽ mở rộng trong tương lai
```

---

## 10. Tóm Tắt Cuối Ngày

| Pattern | Nhóm | Mục đích | Ví dụ thực tế |
|---------|------|----------|---------------|
| **Singleton** | Creational | Chỉ 1 instance toàn app | Logger, ConnectionPool |
| **Factory** | Creational | Tạo object không lộ class cụ thể | `ShapeFactory.create("circle")` |
| **Builder** | Creational | Xây object phức tạp từng bước | `User.builder().name("John").build()` |
| **Adapter** | Structural | Chuyển đổi interface không tương thích | `InputStreamReader` |
| **Decorator** | Structural | Thêm tính năng không sửa class gốc | `BufferedReader(FileReader)` |
| **Strategy** | Behavioral | Thay đổi thuật toán tại runtime | Nhiều cách thanh toán |
| **Observer** | Behavioral | Thông báo tự động khi có thay đổi | Event listeners, pub/sub |
| **Template Method** | Behavioral | Bộ khung quy trình cố định | `DataProcessor.process()` |

---

## 11. Câu Hỏi Phỏng Vấn Thường Gặp

### 🔥 Câu 1: Singleton dùng khi nào? Enum Singleton tốt hơn ở đâu?
**Trả lời:**
Dùng khi cần đúng 1 instance: logger, config, connection pool. Enum Singleton tốt hơn vì: (1) Thread-safe miễn phí (JVM đảm bảo), (2) Serialization-safe (tự động), (3) Chống Reflection attack (không thể tạo thêm instance qua Reflection), (4) Code đơn giản nhất. Nhược: không lazy init, không kế thừa class khác.

### 🔥 Câu 2: Factory Method khác Abstract Factory thế nào?
**Trả lời:**
- **Factory Method**: 1 method tạo 1 loại product. Subclass quyết định tạo object nào. Ví dụ: `Dialog.createButton()` → mỗi OS tạo button khác
- **Abstract Factory**: 1 factory tạo **họ sản phẩm liên quan** (button + checkbox + textfield). Ví dụ: `WindowsFactory` tạo cả bộ UI Windows
- Factory Method = 1 sản phẩm, Abstract Factory = family sản phẩm

### 🔥 Câu 3: Strategy khác Template Method thế nào?
**Trả lời:**
- **Strategy**: Toàn bộ thuật toán thay đổi. Dùng **composition** (HAS-A). Strategy object có thể thay đổi tại runtime
- **Template Method**: Bộ khung cố định, chỉ thay đổi **chi tiết** bên trong. Dùng **inheritance** (IS-A). Subclass override các bước cụ thể
- Strategy linh hoạt hơn (thay đổi runtime), Template Method đơn giản hơn khi quy trình cố định

### 🔥 Câu 4: Decorator khác Adapter thế nào?
**Trả lời:**
- **Adapter**: Chuyển interface A → interface B. **Không thêm tính năng**, chỉ "dịch" interface. Ví dụ: ổ chuyển điện
- **Decorator**: Giữ nguyên interface, **thêm tính năng** mới. Có thể bọc nhiều lớp. Ví dụ: thêm topping cho coffee
- Adapter thay đổi interface, Decorator thêm behavior

### 🔥 Câu 5: Observer pattern dùng ở đâu trong thực tế?
**Trả lời:**
Rất phổ biến: (1) **GUI events**: button click → nhiều event handlers, (2) **Spring Events**: `@EventListener` lắng nghe ApplicationEvent, (3) **Message Queue**: Kafka, RabbitMQ pub/sub pattern, (4) **Reactive programming**: RxJava Observable → Observer, (5) **MVC**: Model thay đổi → View tự cập nhật, (6) **File watcher**: WatchService theo dõi thay đổi file

### 🔥 Câu 6: Builder pattern khi nào nên dùng?
**Trả lời:**
Khi: (1) Object có **nhiều tham số** (> 4-5), (2) Có **tham số optional** (không phải lúc nào cũng cần), (3) Cần tạo **immutable object** (final fields), (4) Constructor telescope pattern (nhiều constructor overload). Trong thực tế: dùng Lombok `@Builder` để auto-generate. Ví dụ: `HttpRequest.newBuilder()`, `StringBuilder`, `Stream.builder()`.

---

## Navigation

- [← Day 16: Reflection](./day-16-reflection-annotations.md)
- [Day 18: Memory & GC →](./day-18-memory-gc.md)
