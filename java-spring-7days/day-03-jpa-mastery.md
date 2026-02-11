# Day 3: JPA Mastery — Full Schema + Flyway + Optimized Queries

> **Mục tiêu:** Hoàn thành toàn bộ entity cho E-Commerce, setup Flyway migration, viết query tối ưu, và xử lý N+1 problem.
>
> **Thời lượng:** ~6-8 giờ
>
> **Prerequisite:** Hoàn thành Day 2 (đã có BaseEntity, Category, Product, ProductImage entities)

---

## 1. Remaining Entities

Day 2 chúng ta đã tạo `BaseEntity`, `Category`, `Product`, `ProductImage`. Hôm nay sẽ hoàn thành phần còn lại.

### 1.1 Role Enum

Tạo file `src/main/java/com/example/ecommerce/enums/Role.java`:

```java
package com.example.ecommerce.enums;

public enum Role {
    CUSTOMER,
    ADMIN
}
```

> **Tại sao dùng enum?** Thay vì lưu String tự do ("admin", "Admin", "ADMIN"...), enum đảm bảo chỉ có đúng giá trị đã định nghĩa. Compile-time safe.

### 1.2 User Entity

Tạo file `src/main/java/com/example/ecommerce/entity/User.java`:

```java
package com.example.ecommerce.entity;

import com.example.ecommerce.enums.Role;
import jakarta.persistence.*;
import java.time.Instant;
import java.util.ArrayList;
import java.util.List;

@Entity
@Table(name = "users")  // "user" là reserved keyword trong nhiều DB
public class User extends BaseEntity {

    @Column(nullable = false, length = 100)
    private String fullName;

    @Column(nullable = false, unique = true)  // unique = DB tự tạo UNIQUE index
    private String email;

    @Column(nullable = false)
    private String password;  // Sẽ lưu BCrypt hash, KHÔNG BAO GIỜ lưu plain text

    @Column(length = 15)
    private String phone;

    @Column(name = "avatar_url")
    private String avatarUrl;

    // EnumType.STRING lưu "CUSTOMER"/"ADMIN" trong DB (đọc được)
    // EnumType.ORDINAL lưu 0, 1 (TRÁNH - thêm enum value sẽ sai thứ tự)
    @Enumerated(EnumType.STRING)
    @Column(nullable = false, length = 20)
    private Role role = Role.CUSTOMER;

    private boolean active = true;
    private boolean locked = false;

    @Column(name = "failed_login_attempts")
    private int failedLoginAttempts = 0;

    @Column(name = "lock_time")
    private Instant lockTime;  // Thời điểm bị lock, null = không bị lock

    @Column(name = "verification_token")
    private String verificationToken;  // Token xác thực email

    @Column(name = "email_verified")
    private boolean emailVerified = false;

    // mappedBy = "user" -> field "user" bên Address là owning side
    // CascadeType.ALL -> CRUD User sẽ cascade xuống Address
    @OneToMany(mappedBy = "user", cascade = CascadeType.ALL)
    private List<Address> addresses = new ArrayList<>();

    // Không cascade -> xóa User KHÔNG tự xóa Order (business rule)
    @OneToMany(mappedBy = "user")
    private List<Order> orders = new ArrayList<>();

    // Constructors
    public User() {}

    public User(String fullName, String email, String password) {
        this.fullName = fullName;
        this.email = email;
        this.password = password;
    }

    // Getters & Setters (Lombok @Data cũng được, nhưng viết tay để hiểu rõ)
    public String getFullName() { return fullName; }
    public void setFullName(String fullName) { this.fullName = fullName; }
    public String getEmail() { return email; }
    public void setEmail(String email) { this.email = email; }
    public String getPassword() { return password; }
    public void setPassword(String password) { this.password = password; }
    public String getPhone() { return phone; }
    public void setPhone(String phone) { this.phone = phone; }
    public String getAvatarUrl() { return avatarUrl; }
    public void setAvatarUrl(String avatarUrl) { this.avatarUrl = avatarUrl; }
    public Role getRole() { return role; }
    public void setRole(Role role) { this.role = role; }
    public boolean isActive() { return active; }
    public void setActive(boolean active) { this.active = active; }
    public boolean isLocked() { return locked; }
    public void setLocked(boolean locked) { this.locked = locked; }
    public int getFailedLoginAttempts() { return failedLoginAttempts; }
    public void setFailedLoginAttempts(int failedLoginAttempts) { this.failedLoginAttempts = failedLoginAttempts; }
    public Instant getLockTime() { return lockTime; }
    public void setLockTime(Instant lockTime) { this.lockTime = lockTime; }
    public String getVerificationToken() { return verificationToken; }
    public void setVerificationToken(String verificationToken) { this.verificationToken = verificationToken; }
    public boolean isEmailVerified() { return emailVerified; }
    public void setEmailVerified(boolean emailVerified) { this.emailVerified = emailVerified; }
    public List<Address> getAddresses() { return addresses; }
    public void setAddresses(List<Address> addresses) { this.addresses = addresses; }
    public List<Order> getOrders() { return orders; }
    public void setOrders(List<Order> orders) { this.orders = orders; }
}
```

**Giải thích các annotation quan trọng:**

| Annotation | Ý nghĩa |
|-----------|----------|
| `@Table(name = "users")` | Đặt tên table trong DB. "user" là reserved keyword nên dùng "users" |
| `@Column(unique = true)` | Tạo UNIQUE constraint, không cho phép email trùng |
| `@Enumerated(EnumType.STRING)` | Lưu tên enum ("CUSTOMER") thay vì số thứ tự (0) |
| `@OneToMany(mappedBy = "user")` | Quan hệ 1-N, "user" là tên field bên entity con |
| `CascadeType.ALL` | Create/Update/Delete User sẽ tự động áp dụng cho Address |

### 1.3 Address Entity

Tạo file `src/main/java/com/example/ecommerce/entity/Address.java`:

```java
package com.example.ecommerce.entity;

import jakarta.persistence.*;

@Entity
@Table(name = "addresses")
public class Address extends BaseEntity {

    @Column(nullable = false)
    private String street;

    private String ward;       // Phường/Xã
    private String district;   // Quận/Huyện

    @Column(nullable = false)
    private String city;       // Tỉnh/Thành phố

    @Column(length = 15)
    private String phone;

    @Column(name = "recipient_name", nullable = false, length = 100)
    private String recipientName;  // Tên người nhận (có thể khác tên user)

    @Column(name = "default_address")
    private boolean defaultAddress = false;

    // FetchType.LAZY = chỉ load User khi thực sự gọi getUser()
    // Đây là owning side vì có @JoinColumn (chứa FK trong table)
    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "user_id", nullable = false)
    private User user;

    // Constructors, Getters & Setters...
    public Address() {}

    public String getStreet() { return street; }
    public void setStreet(String street) { this.street = street; }
    public String getWard() { return ward; }
    public void setWard(String ward) { this.ward = ward; }
    public String getDistrict() { return district; }
    public void setDistrict(String district) { this.district = district; }
    public String getCity() { return city; }
    public void setCity(String city) { this.city = city; }
    public String getPhone() { return phone; }
    public void setPhone(String phone) { this.phone = phone; }
    public String getRecipientName() { return recipientName; }
    public void setRecipientName(String recipientName) { this.recipientName = recipientName; }
    public boolean isDefaultAddress() { return defaultAddress; }
    public void setDefaultAddress(boolean defaultAddress) { this.defaultAddress = defaultAddress; }
    public User getUser() { return user; }
    public void setUser(User user) { this.user = user; }
}
```

### 1.4 OrderStatus Enum

Tạo file `src/main/java/com/example/ecommerce/enums/OrderStatus.java`:

```java
package com.example.ecommerce.enums;

public enum OrderStatus {
    PENDING,      // Vừa đặt, chờ xác nhận
    CONFIRMED,    // Shop đã xác nhận
    PROCESSING,   // Đang chuẩn bị hàng
    SHIPPED,      // Đã giao cho vận chuyển
    DELIVERED,    // Đã giao thành công
    CANCELLED,    // Đã hủy
    REFUNDED      // Đã hoàn tiền
}
```

### 1.5 Order Entity

Tạo file `src/main/java/com/example/ecommerce/entity/Order.java`:

```java
package com.example.ecommerce.entity;

import com.example.ecommerce.enums.OrderStatus;
import jakarta.persistence.*;
import java.math.BigDecimal;
import java.util.ArrayList;
import java.util.List;

@Entity
@Table(name = "orders")  // "order" là reserved keyword trong SQL
public class Order extends BaseEntity {

    @Column(name = "order_number", nullable = false, unique = true, length = 20)
    private String orderNumber;  // VD: "ORD-20240115-001"

    @Enumerated(EnumType.STRING)
    @Column(nullable = false, length = 20)
    private OrderStatus status = OrderStatus.PENDING;

    // precision = tổng số chữ số, scale = số chữ số sau dấu phẩy
    // 12,2 -> max 9,999,999,999.99
    @Column(name = "total_amount", nullable = false, precision = 12, scale = 2)
    private BigDecimal totalAmount;

    @Column(name = "shipping_fee", precision = 10, scale = 2)
    private BigDecimal shippingFee = BigDecimal.ZERO;

    @Column(length = 500)
    private String note;

    // Lưu JSON snapshot của địa chỉ tại thời điểm đặt hàng
    // Vì user có thể thay đổi/xóa address sau khi đặt hàng
    @Column(name = "shipping_address", columnDefinition = "TEXT")
    private String shippingAddress;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "user_id", nullable = false)
    private User user;

    // orphanRemoval = true -> xóa OrderItem khỏi list sẽ xóa luôn trong DB
    @OneToMany(mappedBy = "order", cascade = CascadeType.ALL, orphanRemoval = true)
    private List<OrderItem> items = new ArrayList<>();

    // 1-1 với Payment, cascade để tạo Payment cùng lúc tạo Order
    @OneToOne(mappedBy = "order", cascade = CascadeType.ALL)
    private Payment payment;

    // Constructors
    public Order() {}

    // Helper method - thêm item vào order (đảm bảo bidirectional sync)
    public void addItem(OrderItem item) {
        items.add(item);
        item.setOrder(this);
    }

    public void removeItem(OrderItem item) {
        items.remove(item);
        item.setOrder(null);
    }

    // Getters & Setters
    public String getOrderNumber() { return orderNumber; }
    public void setOrderNumber(String orderNumber) { this.orderNumber = orderNumber; }
    public OrderStatus getStatus() { return status; }
    public void setStatus(OrderStatus status) { this.status = status; }
    public BigDecimal getTotalAmount() { return totalAmount; }
    public void setTotalAmount(BigDecimal totalAmount) { this.totalAmount = totalAmount; }
    public BigDecimal getShippingFee() { return shippingFee; }
    public void setShippingFee(BigDecimal shippingFee) { this.shippingFee = shippingFee; }
    public String getNote() { return note; }
    public void setNote(String note) { this.note = note; }
    public String getShippingAddress() { return shippingAddress; }
    public void setShippingAddress(String shippingAddress) { this.shippingAddress = shippingAddress; }
    public User getUser() { return user; }
    public void setUser(User user) { this.user = user; }
    public List<OrderItem> getItems() { return items; }
    public void setItems(List<OrderItem> items) { this.items = items; }
    public Payment getPayment() { return payment; }
    public void setPayment(Payment payment) { this.payment = payment; }
}
```

### 1.6 OrderItem Entity

Tạo file `src/main/java/com/example/ecommerce/entity/OrderItem.java`:

```java
package com.example.ecommerce.entity;

import jakarta.persistence.*;
import java.math.BigDecimal;

@Entity
@Table(name = "order_items")
public class OrderItem extends BaseEntity {

    // === Snapshot fields ===
    // Giữ thông tin sản phẩm TẠI THỜI ĐIỂM đặt hàng
    // Vì sau này product có thể đổi tên, đổi giá, hoặc bị xóa
    @Column(name = "product_name", nullable = false)
    private String productName;

    @Column(name = "product_price", nullable = false, precision = 12, scale = 2)
    private BigDecimal productPrice;

    @Column(nullable = false)
    private int quantity;

    @Column(precision = 12, scale = 2)
    private BigDecimal subtotal;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "order_id", nullable = false)
    private Order order;

    // product_id có thể NULL nếu product bị xóa sau khi đặt hàng
    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "product_id")
    private Product product;

    // @PrePersist chạy TRƯỚC KHI Hibernate INSERT vào DB
    // Tự động tính subtotal = price * quantity
    @PrePersist
    public void calculateSubtotal() {
        this.subtotal = productPrice.multiply(BigDecimal.valueOf(quantity));
    }

    // Constructors
    public OrderItem() {}

    // Getters & Setters
    public String getProductName() { return productName; }
    public void setProductName(String productName) { this.productName = productName; }
    public BigDecimal getProductPrice() { return productPrice; }
    public void setProductPrice(BigDecimal productPrice) { this.productPrice = productPrice; }
    public int getQuantity() { return quantity; }
    public void setQuantity(int quantity) { this.quantity = quantity; }
    public BigDecimal getSubtotal() { return subtotal; }
    public void setSubtotal(BigDecimal subtotal) { this.subtotal = subtotal; }
    public Order getOrder() { return order; }
    public void setOrder(Order order) { this.order = order; }
    public Product getProduct() { return product; }
    public void setProduct(Product product) { this.product = product; }
}
```

> **Tại sao snapshot?** Khi khách hàng đặt mua "iPhone 15 - 25,000,000d", sau 1 tháng shop tăng giá lên 27,000,000d. Nếu không snapshot, hóa đơn cũ sẽ hiển thị sai giá. Đây là pattern phổ biến trong mọi hệ thống e-commerce.

### 1.7 Payment Enums & Entity

Tạo file `src/main/java/com/example/ecommerce/enums/PaymentMethod.java`:

```java
package com.example.ecommerce.enums;

public enum PaymentMethod {
    COD,            // Thanh toán khi nhận hàng
    STRIPE,         // Thanh toán online qua Stripe
    BANK_TRANSFER   // Chuyển khoản ngân hàng
}
```

Tạo file `src/main/java/com/example/ecommerce/enums/PaymentStatus.java`:

```java
package com.example.ecommerce.enums;

public enum PaymentStatus {
    PENDING,    // Chờ thanh toán
    COMPLETED,  // Đã thanh toán thành công
    FAILED,     // Thanh toán thất bại
    REFUNDED    // Đã hoàn tiền
}
```

Tạo file `src/main/java/com/example/ecommerce/entity/Payment.java`:

```java
package com.example.ecommerce.entity;

import com.example.ecommerce.enums.PaymentMethod;
import com.example.ecommerce.enums.PaymentStatus;
import jakarta.persistence.*;
import java.math.BigDecimal;

@Entity
@Table(name = "payments")
public class Payment extends BaseEntity {

    @Column(nullable = false, precision = 12, scale = 2)
    private BigDecimal amount;

    @Enumerated(EnumType.STRING)
    @Column(nullable = false, length = 20)
    private PaymentMethod method;

    @Enumerated(EnumType.STRING)
    @Column(nullable = false, length = 20)
    private PaymentStatus status = PaymentStatus.PENDING;

    @Column(name = "transaction_id", length = 100)
    private String transactionId;  // Mã giao dịch từ payment gateway

    @Column(name = "stripe_payment_intent_id", length = 100)
    private String stripePaymentIntentId;  // Stripe-specific ID

    // @OneToOne + @JoinColumn = bảng payments chứa cột order_id
    @OneToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "order_id", nullable = false)
    private Order order;

    // Constructors
    public Payment() {}

    // Getters & Setters
    public BigDecimal getAmount() { return amount; }
    public void setAmount(BigDecimal amount) { this.amount = amount; }
    public PaymentMethod getMethod() { return method; }
    public void setMethod(PaymentMethod method) { this.method = method; }
    public PaymentStatus getStatus() { return status; }
    public void setStatus(PaymentStatus status) { this.status = status; }
    public String getTransactionId() { return transactionId; }
    public void setTransactionId(String transactionId) { this.transactionId = transactionId; }
    public String getStripePaymentIntentId() { return stripePaymentIntentId; }
    public void setStripePaymentIntentId(String stripePaymentIntentId) { this.stripePaymentIntentId = stripePaymentIntentId; }
    public Order getOrder() { return order; }
    public void setOrder(Order order) { this.order = order; }
}
```

### 1.8 CartItem Entity

Tạo file `src/main/java/com/example/ecommerce/entity/CartItem.java`:

```java
package com.example.ecommerce.entity;

import jakarta.persistence.*;

@Entity
@Table(name = "cart_items", uniqueConstraints = {
    // Mỗi user chỉ có 1 record cho mỗi product trong giỏ hàng
    // Thêm cùng product -> tăng quantity, KHÔNG tạo record mới
    @UniqueConstraint(columnNames = {"user_id", "product_id"})
})
public class CartItem extends BaseEntity {

    @Column(nullable = false)
    private int quantity;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "user_id", nullable = false)
    private User user;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "product_id", nullable = false)
    private Product product;

    // Constructors
    public CartItem() {}

    // Getters & Setters
    public int getQuantity() { return quantity; }
    public void setQuantity(int quantity) { this.quantity = quantity; }
    public User getUser() { return user; }
    public void setUser(User user) { this.user = user; }
    public Product getProduct() { return product; }
    public void setProduct(Product product) { this.product = product; }
}
```

### 1.9 Review Entity

Tạo file `src/main/java/com/example/ecommerce/entity/Review.java`:

```java
package com.example.ecommerce.entity;

import jakarta.persistence.*;

@Entity
@Table(name = "reviews", uniqueConstraints = {
    // Mỗi user chỉ được review 1 lần cho mỗi product
    @UniqueConstraint(columnNames = {"user_id", "product_id"})
})
public class Review extends BaseEntity {

    @Column(nullable = false)
    private int rating;  // 1-5 sao, validate ở service layer

    @Column(columnDefinition = "TEXT")
    private String comment;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "user_id", nullable = false)
    private User user;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "product_id", nullable = false)
    private Product product;

    // Constructors
    public Review() {}

    // Getters & Setters
    public int getRating() { return rating; }
    public void setRating(int rating) { this.rating = rating; }
    public String getComment() { return comment; }
    public void setComment(String comment) { this.comment = comment; }
    public User getUser() { return user; }
    public void setUser(User user) { this.user = user; }
    public Product getProduct() { return product; }
    public void setProduct(Product product) { this.product = product; }
}
```

---

## 2. Relationships Diagram

### 2.1 Full Entity Relationship Map

```
  ┌─────────────┐       ┌─────────────────┐       ┌───────────────┐
  │  categories  │ 1───N │    products      │ 1───N │ product_images│
  │              │       │                  │       │               │
  │ id (PK)      │       │ id (PK)          │       │ id (PK)       │
  │ name         │       │ category_id (FK) │       │ product_id(FK)│
  │ slug         │       │ name, slug       │       │ url           │
  │ parent_id(FK)│◄──┐   │ price            │       │ is_primary    │
  └──────────────┘   │   │ stock_quantity   │       └───────────────┘
         │           │   │ active           │
         └───────────┘   └────────┬─────────┘
        (self-reference)          │
                                  │ 1
                    ┌─────────────┼─────────────┐
                    │ N           │ N            │ N
              ┌───────────┐ ┌──────────┐  ┌───────────┐
              │ cart_items │ │ reviews  │  │order_items│
              │           │ │          │  │           │
              │ user_id   │ │ user_id  │  │ order_id  │
              │product_id │ │product_id│  │product_id │
              │ quantity   │ │ rating   │  │ prod_name │
              └─────┬─────┘ │ comment  │  │ prod_price│
                    │       └────┬─────┘  │ quantity  │
                    │ N          │ N      └─────┬─────┘
                    │            │              │ N
                    ▼            ▼              ▼
              ┌──────────────────────┐   ┌──────────┐    ┌──────────┐
              │        users         │   │  orders  │1──1│ payments │
              │                      │   │          │    │          │
              │ id (PK)              │   │ id (PK)  │    │ id (PK)  │
              │ email (UNIQUE)       │   │ user_id  │    │ order_id │
              │ password             │1─N│order_num │    │ amount   │
              │ role (ENUM)          │   │ status   │    │ method   │
              │ active, locked       │   │ total    │    │ status   │
              └──────────┬───────────┘   └──────────┘    └──────────┘
                         │ 1
                         │
                         ▼ N
                   ┌───────────┐
                   │ addresses │
                   │           │
                   │ user_id   │
                   │ street    │
                   │ city      │
                   │ is_default│
                   └───────────┘
```

### 2.2 Owning Side Explanation

Trong JPA, **owning side** là entity **chứa foreign key** trong database table.

| Relationship | Owning Side | Inverse Side | FK Column |
|-------------|-------------|-------------|-----------|
| Category - Product | `Product` (@ManyToOne) | `Category` (mappedBy) | `products.category_id` |
| Product - ProductImage | `ProductImage` (@ManyToOne) | `Product` (mappedBy) | `product_images.product_id` |
| User - Address | `Address` (@ManyToOne) | `User` (mappedBy) | `addresses.user_id` |
| User - Order | `Order` (@ManyToOne) | `User` (mappedBy) | `orders.user_id` |
| Order - OrderItem | `OrderItem` (@ManyToOne) | `Order` (mappedBy) | `order_items.order_id` |
| Order - Payment | `Payment` (@OneToOne + @JoinColumn) | `Order` (mappedBy) | `payments.order_id` |
| User - CartItem | `CartItem` (@ManyToOne) | n/a | `cart_items.user_id` |
| User - Review | `Review` (@ManyToOne) | n/a | `reviews.user_id` |
| Category - Category (self) | Child Category (@ManyToOne) | Parent (mappedBy) | `categories.parent_id` |

### 2.3 FetchType Rule of Thumb

```
@ManyToOne  -> mặc định EAGER  -> LUÔN đổi thành LAZY
@OneToOne   -> mặc định EAGER  -> LUÔN đổi thành LAZY
@OneToMany  -> mặc định LAZY   -> Giữ nguyên LAZY
@ManyToMany -> mặc định LAZY   -> Giữ nguyên LAZY
```

**Quy tắc vàng:** Mọi quan hệ đều nên `FetchType.LAZY`. Khi cần data liên quan, dùng `JOIN FETCH` trong query hoặc `@EntityGraph`. Lý do: nếu dùng EAGER, mỗi lần load 1 entity sẽ tự động load tất cả entity liên quan, gây ra vô số query không cần thiết.

---

## 3. Flyway Migrations

### 3.1 Tại sao cần Flyway?

Flyway là **version control cho database**. Giống như Git quản lý code, Flyway quản lý schema changes.

**Không có Flyway:**
- Developer A thêm column, Developer B không biết -> crash
- Lên production quên chạy ALTER TABLE -> crash
- Không biết DB đang ở version nào

**Có Flyway:**
- Mỗi schema change = 1 file SQL có version number
- Flyway tự track đã chạy đến version nào (bảng `flyway_schema_history`)
- Startup app -> Flyway tự chạy các migration chưa chạy
- Mọi người luôn có cùng DB schema

### 3.2 Setup Flyway

Thêm dependency trong `pom.xml` (nếu chưa có):

```xml
<dependency>
    <groupId>org.flywaydb</groupId>
    <artifactId>flyway-core</artifactId>
</dependency>
<dependency>
    <groupId>org.flywaydb</groupId>
    <artifactId>flyway-mysql</artifactId>
</dependency>
```

### 3.3 Naming Convention

```
V{version}__{description}.sql

V1__create_categories_table.sql       (OK)
V2__create_products_tables.sql        (OK)
V10__add_phone_to_users.sql           (OK - sort theo số)

R__seed_data.sql                      (Repeatable migration - chạy lại mỗi khi thay đổi)
```

> **Lưu ý:** Dấu gạch dưới là **HAI dấu** (`__`), không phải một. Đây là convention bắt buộc của Flyway.

### 3.4 Migration Scripts

Tạo folder `src/main/resources/db/migration/`.

**V1__create_categories_table.sql:**

```sql
CREATE TABLE categories (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    slug VARCHAR(120) NOT NULL UNIQUE,
    description TEXT,
    image_url VARCHAR(500),
    active BOOLEAN DEFAULT TRUE,
    parent_id BIGINT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    CONSTRAINT fk_category_parent FOREIGN KEY (parent_id) REFERENCES categories(id) ON DELETE SET NULL,
    INDEX idx_category_slug (slug),
    INDEX idx_category_parent (parent_id)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
```

**V2__create_products_tables.sql:**

```sql
CREATE TABLE products (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(255) NOT NULL,
    slug VARCHAR(300) NOT NULL UNIQUE,
    description TEXT,
    price DECIMAL(12,2) NOT NULL,
    sale_price DECIMAL(12,2),
    stock_quantity INT NOT NULL DEFAULT 0,
    sku VARCHAR(50) UNIQUE,
    active BOOLEAN DEFAULT TRUE,
    featured BOOLEAN DEFAULT FALSE,
    category_id BIGINT NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    CONSTRAINT fk_product_category FOREIGN KEY (category_id) REFERENCES categories(id),
    INDEX idx_product_slug (slug),
    INDEX idx_product_category (category_id),
    INDEX idx_product_active_featured (active, featured)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;

CREATE TABLE product_images (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    url VARCHAR(500) NOT NULL,
    alt_text VARCHAR(255),
    is_primary BOOLEAN DEFAULT FALSE,
    sort_order INT DEFAULT 0,
    product_id BIGINT NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    CONSTRAINT fk_image_product FOREIGN KEY (product_id) REFERENCES products(id) ON DELETE CASCADE,
    INDEX idx_image_product (product_id)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
```

**V3__create_users_table.sql:**

```sql
CREATE TABLE users (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    full_name VARCHAR(100) NOT NULL,
    email VARCHAR(255) NOT NULL UNIQUE,
    password VARCHAR(255) NOT NULL,
    phone VARCHAR(15),
    avatar_url VARCHAR(500),
    role VARCHAR(20) NOT NULL DEFAULT 'CUSTOMER',
    active BOOLEAN DEFAULT TRUE,
    locked BOOLEAN DEFAULT FALSE,
    failed_login_attempts INT DEFAULT 0,
    lock_time TIMESTAMP NULL,
    verification_token VARCHAR(255),
    email_verified BOOLEAN DEFAULT FALSE,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    INDEX idx_user_email (email),
    INDEX idx_user_role (role)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
```

**V4__create_addresses_cart_items_tables.sql:**

```sql
CREATE TABLE addresses (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    street VARCHAR(255) NOT NULL,
    ward VARCHAR(100),
    district VARCHAR(100),
    city VARCHAR(100) NOT NULL,
    phone VARCHAR(15),
    recipient_name VARCHAR(100) NOT NULL,
    default_address BOOLEAN DEFAULT FALSE,
    user_id BIGINT NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    CONSTRAINT fk_address_user FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE,
    INDEX idx_address_user (user_id)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;

CREATE TABLE cart_items (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    quantity INT NOT NULL DEFAULT 1,
    user_id BIGINT NOT NULL,
    product_id BIGINT NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    CONSTRAINT fk_cart_user FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE,
    CONSTRAINT fk_cart_product FOREIGN KEY (product_id) REFERENCES products(id) ON DELETE CASCADE,
    UNIQUE KEY uk_cart_user_product (user_id, product_id),
    INDEX idx_cart_user (user_id)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
```

**V5__create_orders_tables.sql:**

```sql
CREATE TABLE orders (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    order_number VARCHAR(20) NOT NULL UNIQUE,
    status VARCHAR(20) NOT NULL DEFAULT 'PENDING',
    total_amount DECIMAL(12,2) NOT NULL,
    shipping_fee DECIMAL(10,2) DEFAULT 0.00,
    note VARCHAR(500),
    shipping_address TEXT,
    user_id BIGINT NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    CONSTRAINT fk_order_user FOREIGN KEY (user_id) REFERENCES users(id),
    INDEX idx_order_user (user_id),
    INDEX idx_order_number (order_number),
    INDEX idx_order_status (status)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;

CREATE TABLE order_items (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    product_name VARCHAR(255) NOT NULL,
    product_price DECIMAL(12,2) NOT NULL,
    quantity INT NOT NULL,
    subtotal DECIMAL(12,2),
    order_id BIGINT NOT NULL,
    product_id BIGINT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    CONSTRAINT fk_orderitem_order FOREIGN KEY (order_id) REFERENCES orders(id) ON DELETE CASCADE,
    CONSTRAINT fk_orderitem_product FOREIGN KEY (product_id) REFERENCES products(id) ON DELETE SET NULL,
    INDEX idx_orderitem_order (order_id)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;

CREATE TABLE payments (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    amount DECIMAL(12,2) NOT NULL,
    method VARCHAR(20) NOT NULL,
    status VARCHAR(20) NOT NULL DEFAULT 'PENDING',
    transaction_id VARCHAR(100),
    stripe_payment_intent_id VARCHAR(100),
    order_id BIGINT NOT NULL UNIQUE,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    CONSTRAINT fk_payment_order FOREIGN KEY (order_id) REFERENCES orders(id),
    INDEX idx_payment_order (order_id),
    INDEX idx_payment_status (status)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
```

**V6__create_reviews_and_seed_data.sql:**

```sql
-- Reviews table
CREATE TABLE reviews (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    rating INT NOT NULL,
    comment TEXT,
    user_id BIGINT NOT NULL,
    product_id BIGINT NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    CONSTRAINT fk_review_user FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE,
    CONSTRAINT fk_review_product FOREIGN KEY (product_id) REFERENCES products(id) ON DELETE CASCADE,
    UNIQUE KEY uk_review_user_product (user_id, product_id),
    INDEX idx_review_product (product_id),
    INDEX idx_review_rating (rating)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;

-- Seed categories
INSERT INTO categories (name, slug, description, active) VALUES
('Điện thoại', 'dien-thoai', 'Smartphone và phụ kiện', TRUE),
('Laptop', 'laptop', 'Laptop văn phòng và gaming', TRUE),
('Phụ kiện', 'phu-kien', 'Phụ kiện công nghệ', TRUE);

-- Seed products
INSERT INTO products (name, slug, description, price, stock_quantity, sku, active, featured, category_id) VALUES
('iPhone 15 Pro Max', 'iphone-15-pro-max', 'Apple iPhone 15 Pro Max 256GB', 34990000.00, 50, 'IP15PM-256', TRUE, TRUE, 1),
('Samsung Galaxy S24 Ultra', 'samsung-galaxy-s24-ultra', 'Samsung Galaxy S24 Ultra 512GB', 33990000.00, 30, 'SS-S24U-512', TRUE, TRUE, 1),
('MacBook Pro 14 M3', 'macbook-pro-14-m3', 'Apple MacBook Pro 14 inch M3 chip', 49990000.00, 20, 'MBP14-M3', TRUE, TRUE, 2),
('Tai nghe AirPods Pro 2', 'tai-nghe-airpods-pro-2', 'Apple AirPods Pro 2nd gen USB-C', 6790000.00, 100, 'APP2-USBC', TRUE, FALSE, 3),
('Sạc nhanh Anker 65W', 'sac-nhanh-anker-65w', 'Anker 65W GaN USB-C Charger', 890000.00, 200, 'ANK-65W', TRUE, FALSE, 3);
```

### 3.5 Flyway Configuration

Cấu hình trong `application.yml`:

```yaml
spring:
  jpa:
    # QUAN TRỌNG: Khi dùng Flyway, đổi ddl-auto thành "validate"
    # "validate" = Hibernate chỉ KIỂM TRA schema khớp với entities, KHÔNG tạo/sửa table
    # Flyway chịu trách nhiệm tạo/sửa table
    hibernate:
      ddl-auto: validate
    show-sql: true

  flyway:
    enabled: true
    locations: classpath:db/migration
    baseline-on-migrate: true  # Cho phép migrate trên DB đã có data
    # baseline-version: 0      # Uncomment nếu DB đã tồn tại trước khi dùng Flyway
```

> **Lưu ý quan trọng:** Trước Day 3, chúng ta dùng `ddl-auto: update` để Hibernate tự tạo table. Bây giờ chuyển sang Flyway, phải đổi thành `validate`. Nếu để `update`, Hibernate và Flyway sẽ "đánh nhau" -- cả hai đều cố gắng thay đổi schema.

---

## 4. Repository Queries

### 4.1 Derived Query Methods (Spring Data Magic)

Spring Data JPA tự động tạo SQL query từ tên method. Bạn chỉ cần đặt tên theo convention.

| Method Name | Generated SQL |
|-------------|---------------|
| `findByEmail(String email)` | `WHERE email = ?` |
| `findByRole(Role role)` | `WHERE role = ?` |
| `findByActiveTrue()` | `WHERE active = TRUE` |
| `findByPriceBetween(BigDecimal min, BigDecimal max)` | `WHERE price BETWEEN ? AND ?` |
| `findByNameContainingIgnoreCase(String name)` | `WHERE LOWER(name) LIKE LOWER('%?%')` |
| `findByCategoryIdAndActiveTrue(Long catId)` | `WHERE category_id = ? AND active = TRUE` |
| `countByCategoryId(Long categoryId)` | `SELECT COUNT(*) WHERE category_id = ?` |
| `existsByEmail(String email)` | `SELECT EXISTS(... WHERE email = ?)` |
| `deleteByUserId(Long userId)` | `DELETE WHERE user_id = ?` |
| `findTop5ByOrderByCreatedAtDesc()` | `ORDER BY created_at DESC LIMIT 5` |
| `findByStatusIn(List<OrderStatus> statuses)` | `WHERE status IN (?, ?, ...)` |

**ProductRepository ví dụ:**

```java
package com.example.ecommerce.repository;

import com.example.ecommerce.entity.Product;
import org.springframework.data.domain.Page;
import org.springframework.data.domain.Pageable;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.data.jpa.repository.JpaSpecificationExecutor;
import org.springframework.data.jpa.repository.Modifying;
import org.springframework.data.jpa.repository.Query;
import org.springframework.data.repository.query.Param;
import org.springframework.stereotype.Repository;

import java.util.List;
import java.util.Optional;

@Repository
public interface ProductRepository extends JpaRepository<Product, Long>,
                                           JpaSpecificationExecutor<Product> {

    // === Derived queries ===
    Optional<Product> findBySlug(String slug);
    List<Product> findByCategoryIdAndActiveTrue(Long categoryId);
    boolean existsBySlug(String slug);
    long countByCategoryId(Long categoryId);

    // === JPQL queries ===

    // JOIN FETCH = load Product + images trong 1 query (giải quyết N+1)
    @Query("SELECT p FROM Product p LEFT JOIN FETCH p.images WHERE p.id = :id")
    Optional<Product> findByIdWithImages(@Param("id") Long id);

    // Tìm products theo category, kèm theo images
    @Query("SELECT DISTINCT p FROM Product p " +
           "LEFT JOIN FETCH p.images " +
           "WHERE p.category.id = :categoryId AND p.active = true")
    List<Product> findByCategoryWithImages(@Param("categoryId") Long categoryId);

    // Search theo tên (case-insensitive)
    @Query("SELECT p FROM Product p WHERE LOWER(p.name) LIKE LOWER(CONCAT('%', :keyword, '%'))")
    Page<Product> searchByName(@Param("keyword") String keyword, Pageable pageable);

    // === Native SQL query ===
    // Dùng khi JPQL không đủ mạnh (VD: MySQL-specific functions)
    @Query(value = "SELECT p.*, AVG(r.rating) as avg_rating " +
                   "FROM products p LEFT JOIN reviews r ON p.id = r.product_id " +
                   "WHERE p.active = TRUE " +
                   "GROUP BY p.id " +
                   "ORDER BY avg_rating DESC " +
                   "LIMIT :limit",
           nativeQuery = true)
    List<Product> findTopRatedProducts(@Param("limit") int limit);

    // === UPDATE/DELETE queries ===
    // @Modifying BẮT BUỘC cho UPDATE/DELETE queries
    // @Transactional đặt ở SERVICE layer, KHÔNG đặt ở đây
    @Modifying
    @Query("UPDATE Product p SET p.active = false WHERE p.id = :id")
    int softDeleteById(@Param("id") Long id);

    @Modifying
    @Query("UPDATE Product p SET p.stockQuantity = p.stockQuantity - :qty WHERE p.id = :id AND p.stockQuantity >= :qty")
    int decreaseStock(@Param("id") Long id, @Param("qty") int quantity);
}
```

### 4.2 @Modifying và @Transactional -- Đặt Đúng Chỗ

```
                    +-----------------+
                    |   Controller    |
                    +-----------------+
                            |
                            v
                    +-----------------+
                    |    Service      |  <-- @Transactional ĐẶT Ở ĐÂY
                    +-----------------+
                            |
                            v
                    +-----------------+
                    |   Repository    |  <-- @Modifying ĐẶT Ở ĐÂY
                    +-----------------+
```

**Sai:**
```java
// KHÔNG đặt @Transactional trên repository method
@Modifying
@Transactional  // SAI! Không nên đặt ở đây
@Query("UPDATE Product p SET p.active = false WHERE p.id = :id")
int softDeleteById(Long id);
```

**Đúng:**
```java
// Repository: chỉ có @Modifying
@Modifying
@Query("UPDATE Product p SET p.active = false WHERE p.id = :id")
int softDeleteById(Long id);

// Service: @Transactional bao trùm toàn bộ business logic
@Service
public class ProductService {
    @Transactional  // ĐÚNG! Transaction bao trùm toàn bộ method
    public void deactivateProduct(Long id) {
        int updated = productRepository.softDeleteById(id);
        if (updated == 0) {
            throw new ResourceNotFoundException("Product not found: " + id);
        }
        // Có thể làm thêm việc khác trong cùng transaction...
    }
}
```

**Lý do:** `@Transactional` ở service layer cho phép nhiều repository operations trong cùng 1 transaction. Nếu đặt ở repository, mỗi query là 1 transaction riêng, không thể rollback toàn bộ khi có lỗi.

### 4.3 Projection Interface

Khi chỉ cần một vài fields, không cần load toàn bộ entity:

```java
// Interface projection - Hibernate chỉ SELECT các column cần thiết
public interface ProductSummary {
    Long getId();
    String getName();
    BigDecimal getPrice();
    String getCategoryName();  // Tự động join với Category
}

// Trong ProductRepository
@Query("SELECT p.id as id, p.name as name, p.price as price, " +
       "p.category.name as categoryName " +
       "FROM Product p WHERE p.active = true")
Page<ProductSummary> findAllSummaries(Pageable pageable);

// Hoặc derived query cũng được:
List<ProductSummary> findByActiveTrue();
```

> **Khi nào dùng Projection?** Khi API chỉ cần trả về 3-4 fields của entity có 15+ fields. Giảm data truyền qua network và giảm memory usage.

---

## 5. Pagination + Sort

### 5.1 Pageable trong Repository

```java
// Repository method với Pageable
Page<Product> findByCategoryIdAndActiveTrue(Long categoryId, Pageable pageable);

// Spring Data tự động thêm LIMIT, OFFSET, ORDER BY vào SQL
```

### 5.2 Tạo Pageable Object

```java
// Cách 1: Cơ bản
Pageable pageable = PageRequest.of(0, 20);  // page 0, size 20

// Cách 2: Có sort
Pageable pageable = PageRequest.of(0, 20, Sort.by("createdAt").descending());

// Cách 3: Nhiều trường sort
Pageable pageable = PageRequest.of(0, 20,
    Sort.by(Sort.Direction.DESC, "createdAt")
        .and(Sort.by(Sort.Direction.ASC, "name"))
);
```

### 5.3 Controller với @PageableDefault

```java
@RestController
@RequestMapping("/api/v1/products")
public class ProductController {

    private final ProductService productService;

    public ProductController(ProductService productService) {
        this.productService = productService;
    }

    // @PageableDefault: giá trị mặc định nếu client không gửi params
    // Client gửi: GET /api/v1/products?page=0&size=10&sort=price,desc
    @GetMapping
    public ApiResponse<PagedResponse<ProductResponse>> getProducts(
            @PageableDefault(size = 20, sort = "createdAt", direction = Sort.Direction.DESC)
            Pageable pageable) {
        return ApiResponse.success(productService.getProducts(pageable));
    }
}
```

### 5.4 Convert Page sang PagedResponse

```java
// DTO chung cho tất cả paged responses
public class PagedResponse<T> {
    private List<T> content;
    private int pageNumber;
    private int pageSize;
    private long totalElements;
    private int totalPages;
    private boolean last;

    public static <T> PagedResponse<T> from(Page<T> page) {
        PagedResponse<T> response = new PagedResponse<>();
        response.setContent(page.getContent());
        response.setPageNumber(page.getNumber());
        response.setPageSize(page.getSize());
        response.setTotalElements(page.getTotalElements());
        response.setTotalPages(page.getTotalPages());
        response.setLast(page.isLast());
        return response;
    }

    // Getters & Setters...
    public List<T> getContent() { return content; }
    public void setContent(List<T> content) { this.content = content; }
    public int getPageNumber() { return pageNumber; }
    public void setPageNumber(int pageNumber) { this.pageNumber = pageNumber; }
    public int getPageSize() { return pageSize; }
    public void setPageSize(int pageSize) { this.pageSize = pageSize; }
    public long getTotalElements() { return totalElements; }
    public void setTotalElements(long totalElements) { this.totalElements = totalElements; }
    public int getTotalPages() { return totalPages; }
    public void setTotalPages(int totalPages) { this.totalPages = totalPages; }
    public boolean isLast() { return last; }
    public void setLast(boolean last) { this.last = last; }
}
```

**Sử dụng trong Service:**

```java
@Service
public class ProductService {

    private final ProductRepository productRepository;
    private final ProductMapper productMapper;  // MapStruct mapper

    // ... constructor injection ...

    public PagedResponse<ProductResponse> getProducts(Pageable pageable) {
        Page<Product> productPage = productRepository.findByActiveTrue(pageable);

        // Convert Page<Product> -> Page<ProductResponse> -> PagedResponse
        Page<ProductResponse> responsePage = productPage.map(productMapper::toResponse);
        return PagedResponse.from(responsePage);
    }
}
```

---

## 6. Specification Pattern

### 6.1 Vấn đề cần giải quyết

Trang listing products có nhiều bộ lọc: theo category, theo giá, theo tên, theo trạng thái... Không lẽ viết 1 repository method cho mỗi tổ hợp bộ lọc?

```java
// KHÔNG thể viết hết các combination:
findByCategoryId(Long catId);
findByCategoryIdAndPriceBetween(Long catId, BigDecimal min, BigDecimal max);
findByCategoryIdAndPriceBetweenAndNameContaining(...);
findByPriceBetweenAndActiveTrue(...);
// ... vô số combination
```

**Specification Pattern** cho phép tạo các điều kiện filter độc lập, rồi **ghép chúng** tùy ý.

### 6.2 ProductSpecification Class

Tạo file `src/main/java/com/example/ecommerce/specification/ProductSpecification.java`:

```java
package com.example.ecommerce.specification;

import com.example.ecommerce.entity.Product;
import org.springframework.data.jpa.domain.Specification;
import java.math.BigDecimal;

public class ProductSpecification {

    // Mỗi method trả về 1 Specification<Product>
    // Specification là functional interface: (root, query, criteriaBuilder) -> Predicate
    //   - root: entity gốc (Product)
    //   - query: CriteriaQuery (để join, group by, etc.)
    //   - cb: CriteriaBuilder (để tạo điều kiện: equal, like, between, etc.)

    public static Specification<Product> hasCategory(Long categoryId) {
        return (root, query, cb) ->
            categoryId == null ? null : cb.equal(root.get("category").get("id"), categoryId);
        // Trả về null = bỏ qua điều kiện này (không filter)
    }

    public static Specification<Product> hasPriceBetween(BigDecimal min, BigDecimal max) {
        return (root, query, cb) -> {
            if (min == null && max == null) return null;
            if (min == null) return cb.lessThanOrEqualTo(root.get("price"), max);
            if (max == null) return cb.greaterThanOrEqualTo(root.get("price"), min);
            return cb.between(root.get("price"), min, max);
        };
    }

    public static Specification<Product> hasNameContaining(String keyword) {
        return (root, query, cb) ->
            keyword == null || keyword.isBlank()
                ? null
                : cb.like(cb.lower(root.get("name")), "%" + keyword.toLowerCase() + "%");
    }

    public static Specification<Product> isActive() {
        return (root, query, cb) -> cb.isTrue(root.get("active"));
    }

    public static Specification<Product> isFeatured() {
        return (root, query, cb) -> cb.isTrue(root.get("featured"));
    }
}
```

### 6.3 ProductSpecBuilder (Fluent Builder Pattern)

```java
package com.example.ecommerce.specification;

import com.example.ecommerce.entity.Product;
import org.springframework.data.jpa.domain.Specification;
import java.math.BigDecimal;

public class ProductSpecBuilder {

    private Specification<Product> spec;

    public ProductSpecBuilder() {
        // Bắt đầu với điều kiện luôn TRUE (không filter gì)
        this.spec = Specification.where(null);
    }

    public ProductSpecBuilder category(Long categoryId) {
        spec = spec.and(ProductSpecification.hasCategory(categoryId));
        return this;
    }

    public ProductSpecBuilder priceBetween(BigDecimal min, BigDecimal max) {
        spec = spec.and(ProductSpecification.hasPriceBetween(min, max));
        return this;
    }

    public ProductSpecBuilder keyword(String keyword) {
        spec = spec.and(ProductSpecification.hasNameContaining(keyword));
        return this;
    }

    public ProductSpecBuilder activeOnly() {
        spec = spec.and(ProductSpecification.isActive());
        return this;
    }

    public ProductSpecBuilder featuredOnly() {
        spec = spec.and(ProductSpecification.isFeatured());
        return this;
    }

    public Specification<Product> build() {
        return spec;
    }
}
```

### 6.4 Sử dụng trong Service

```java
@Service
public class ProductService {

    private final ProductRepository productRepository;

    // ... constructor ...

    public PagedResponse<ProductResponse> searchProducts(
            Long categoryId,
            BigDecimal minPrice,
            BigDecimal maxPrice,
            String keyword,
            Pageable pageable) {

        // Builder pattern: ghép các điều kiện một cách sạch sẽ
        Specification<Product> spec = new ProductSpecBuilder()
                .activeOnly()
                .category(categoryId)
                .priceBetween(minPrice, maxPrice)
                .keyword(keyword)
                .build();

        // JpaSpecificationExecutor.findAll(spec, pageable) tự động tạo SQL
        Page<Product> page = productRepository.findAll(spec, pageable);
        return PagedResponse.from(page.map(productMapper::toResponse));
    }
}
```

> **Lưu ý:** Repository phải extend `JpaSpecificationExecutor<Product>` (đã khai báo ở Section 4.1) để dùng được `findAll(Specification, Pageable)`.

---

## 7. Performance Essentials

### 7.1 N+1 Problem -- Vấn đề nghiêm trọng nhất của JPA

**N+1 là gì?** Khi load 1 danh sách (1 query), rồi với mỗi entity lại chạy thêm 1 query để load dữ liệu liên quan (N queries). Tổng cộng: 1 + N queries.

**Ví dụ lỗi:**

```java
// Service method
public List<ProductResponse> getAll() {
    List<Product> products = productRepository.findAll();  // 1 query: SELECT * FROM products

    return products.stream().map(p -> {
        ProductResponse dto = new ProductResponse();
        dto.setName(p.getName());
        dto.setCategoryName(p.getCategory().getName());  // MỖI LẦN GỌI = 1 query thêm!
        return dto;
    }).toList();
}
```

**SQL log sẽ thấy:**
```sql
-- Query 1: Load tất cả products
SELECT * FROM products;

-- Query 2: Load category cho product #1
SELECT * FROM categories WHERE id = 1;

-- Query 3: Load category cho product #2
SELECT * FROM categories WHERE id = 1;  -- Cùng category nhưng vẫn query lại!

-- Query 4: Load category cho product #3
SELECT * FROM categories WHERE id = 2;

-- ... 100 products = 100 queries NỮA!
```

**Có 100 products -> 101 queries!** Thay vì 1 query với JOIN.

### 7.2 Bật SQL Logging để Phát hiện N+1

Cấu hình trong `application.yml`:

```yaml
logging:
  level:
    # Hibernate 6.x SQL logging (ĐÚNG cho Spring Boot 3.x)
    org.hibernate.SQL: DEBUG                   # Hiển thị SQL statements
    org.hibernate.orm.jdbc.bind: TRACE         # Hiển thị parameter values

    # LƯU Ý: Path cũ KHÔNG còn đúng trong Hibernate 6.x:
    # org.hibernate.type.descriptor.sql.BasicBinder: TRACE  -- SAI! Đây là Hibernate 5.x
```

> **Quan trọng:** Nếu bạn tìm tutorial cũ trên mạng, họ thường dùng `org.hibernate.type.descriptor.sql.BasicBinder`. Path này chỉ đúng cho Hibernate 5.x. Spring Boot 3.x dùng Hibernate 6.x, path đúng là `org.hibernate.orm.jdbc.bind`.

### 7.3 Giải pháp 1: JOIN FETCH trong JPQL

```java
// TRƯỚC (N+1):
@Query("SELECT p FROM Product p")
List<Product> findAll();  // category được lazy load -> N+1

// SAU (1 query duy nhất):
@Query("SELECT p FROM Product p JOIN FETCH p.category")
List<Product> findAllWithCategory();
// SQL: SELECT p.*, c.* FROM products p INNER JOIN categories c ON p.category_id = c.id

// Với collection (OneToMany) phải dùng DISTINCT:
@Query("SELECT DISTINCT p FROM Product p LEFT JOIN FETCH p.images WHERE p.active = true")
List<Product> findAllActiveWithImages();
```

**Lưu ý với pagination:** `JOIN FETCH` + `Pageable` trên collection **KHÔNG HOẠT ĐỘNG** như mong đợi. Hibernate sẽ load TOÀN BỘ data rồi phân trang trong memory (rất nguy hiểm). Sử dụng `@EntityGraph` hoặc `@BatchSize` thay thế.

### 7.4 Giải pháp 2: @EntityGraph trên Repository Method

```java
public interface ProductRepository extends JpaRepository<Product, Long> {

    // @EntityGraph chỉ định load những relationship nào cùng 1 query
    // Tương đương JOIN FETCH nhưng linh hoạt hơn
    @EntityGraph(attributePaths = {"category", "images"})
    @Query("SELECT p FROM Product p WHERE p.active = true")
    Page<Product> findAllActiveWithDetails(Pageable pageable);

    // Có thể dùng với derived query
    @EntityGraph(attributePaths = {"category"})
    Page<Product> findByActiveTrue(Pageable pageable);
}
```

**Ưu điểm so với JOIN FETCH:** Hoạt động đúng với Pageable, Hibernate sẽ tạo 2 query tối ưu thay vì load hết vào memory.

### 7.5 Giải pháp 3: @BatchSize trên Collection

```java
@Entity
public class Product extends BaseEntity {

    // Thay vì load images 1-by-1 (N+1), load theo batch
    // VD: 100 products -> thay vì 100 queries, chỉ cần 100/20 = 5 queries
    @BatchSize(size = 20)
    @OneToMany(mappedBy = "product", cascade = CascadeType.ALL)
    private List<ProductImage> images = new ArrayList<>();
}
```

**Hoặc cấu hình global trong `application.yml`:**

```yaml
spring:
  jpa:
    properties:
      hibernate:
        default_batch_fetch_size: 20  # Áp dụng cho tất cả lazy collections
```

### 7.6 Tắt Open-in-View (QUAN TRỌNG)

```yaml
spring:
  jpa:
    open-in-view: false  # Mặc định là TRUE -- rất nguy hiểm
```

**Open-in-View là gì?** Khi `true`, Hibernate Session được giữ mở từ Controller đến View. Nghĩa là code trong Controller có thể lazy-load bất kỳ relationship nào mà không cần JOIN FETCH.

**Tại sao phải tắt?**
1. **Ẩn N+1 problem:** Code trông như chạy tốt, nhưng thầm thì chạy hàng chục query
2. **Database connection giữ lâu:** Connection bị giữ suốt request lifecycle, không trả về pool
3. **Không kiểm soát được:** Không biết query nào chạy khi nào
4. **Production problem:** Application chậm dần, không biết tại sao

**Khi tắt:** Nếu access lazy field ngoài transaction -> báo lỗi `LazyInitializationException`. Đây là **điều tốt** -- buộc bạn phải load dữ liệu một cách có ý thức (JOIN FETCH, EntityGraph).

### 7.7 Batch Insert Configuration

Khi insert nhiều records cùng lúc (VD: import 1000 products từ Excel):

```yaml
spring:
  jpa:
    properties:
      hibernate:
        jdbc:
          batch_size: 50           # Gom 50 INSERTs thành 1 batch
        order_inserts: true        # Sắp xếp INSERT theo entity type (tối ưu batch)
        order_updates: true        # Tương tự cho UPDATE

# BẮT BUỘC thêm vào connection string:
# jdbc:mysql://localhost:3306/ecommerce?rewriteBatchedStatements=true
# Không có param này, MySQL JDBC driver sẽ KHÔNG thực sự batch
```

> **Lưu ý:** `rewriteBatchedStatements=true` trong MySQL connection string là **bắt buộc** để batch insert thực sự hoạt động. Không có nó, driver vẫn gửi từng câu INSERT riêng lẻ.

---

## Tổng Kết Day 3

### Đã học:

| Chủ đề | Key takeaway |
|--------|-------------|
| Entities | 9 entities + 4 enums tạo thành full e-commerce schema |
| Relationships | Owning side = entity có @JoinColumn. LUÔN dùng FetchType.LAZY |
| Flyway | Version control cho DB. Đổi ddl-auto thành "validate" |
| Derived Queries | Spring Data tự tạo SQL từ tên method |
| JPQL + Native | @Query cho phép viết query phức tạp. @Modifying cho UPDATE/DELETE |
| Pagination | Pageable + PageRequest + @PageableDefault |
| Specification | Dynamic filtering không cần viết hàng chục method |
| N+1 Problem | Phát hiện bằng SQL log. Fix bằng JOIN FETCH / @EntityGraph / @BatchSize |
| open-in-view | TẮT NGAY. Buộc mình xử lý lazy loading đúng cách |

### Cấu trúc thư mục sau Day 3:

```
src/main/java/com/example/ecommerce/
├── entity/
│   ├── BaseEntity.java          (Day 2)
│   ├── Category.java            (Day 2)
│   ├── Product.java             (Day 2)
│   ├── ProductImage.java        (Day 2)
│   ├── User.java                (Day 3) ★
│   ├── Address.java             (Day 3) ★
│   ├── Order.java               (Day 3) ★
│   ├── OrderItem.java           (Day 3) ★
│   ├── Payment.java             (Day 3) ★
│   ├── CartItem.java            (Day 3) ★
│   └── Review.java              (Day 3) ★
├── enums/
│   ├── Role.java                (Day 3) ★
│   ├── OrderStatus.java         (Day 3) ★
│   ├── PaymentMethod.java       (Day 3) ★
│   └── PaymentStatus.java       (Day 3) ★
├── repository/
│   ├── CategoryRepository.java  (Day 2)
│   └── ProductRepository.java   (Day 3) ★ (enhanced)
├── specification/
│   ├── ProductSpecification.java (Day 3) ★
│   └── ProductSpecBuilder.java   (Day 3) ★
├── dto/
│   └── response/
│       └── PagedResponse.java    (Day 3) ★
└── resources/
    └── db/migration/
        ├── V1__create_categories_table.sql        (Day 3) ★
        ├── V2__create_products_tables.sql          (Day 3) ★
        ├── V3__create_users_table.sql              (Day 3) ★
        ├── V4__create_addresses_cart_items_tables.sql (Day 3) ★
        ├── V5__create_orders_tables.sql            (Day 3) ★
        └── V6__create_reviews_and_seed_data.sql    (Day 3) ★
```

### Bài tập tự làm:

1. **Tạo `UserRepository`** với derived queries: `findByEmail`, `existsByEmail`, `findByRoleAndActiveTrue`
2. **Tạo `OrderRepository`** với JPQL: tìm orders của user kèm theo items (JOIN FETCH)
3. **Tạo `ReviewRepository`** với: tính average rating của 1 product, đếm số reviews theo rating
4. **Thử bật SQL logging** và đếm số queries khi load products với category. Sau đó fix bằng JOIN FETCH và so sánh

---

> **Ngày mai (Day 4):** REST API + CRUD + Validation -- Xây dựng complete Product API với DTO mapping, Bean Validation, Exception handling, và OpenAPI documentation.
