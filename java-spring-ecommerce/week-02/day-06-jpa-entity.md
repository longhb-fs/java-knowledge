# Ngày 06: JPA Entity & Relationships Chi Tiết

## Mục tiêu hôm nay
- Nắm vững tất cả JPA relationships: OneToOne, OneToMany, ManyToOne, ManyToMany
- Hiểu Bidirectional vs Unidirectional relationships
- Xử lý FetchType (LAZY vs EAGER) đúng cách
- Cascade operations và orphanRemoval
- Tạo thêm User, Order, OrderItem, Address entities

---

## 1. JPA RELATIONSHIPS TỔNG QUAN

### 1.1. Bảng tóm tắt

```
┌─────────────────────────────────────────────────────────────────────┐
│                     JPA RELATIONSHIP TYPES                          │
├───────────────┬─────────────────┬───────────────────────────────────┤
│ Relationship  │ Annotation       │ Ví dụ                            │
├───────────────┼─────────────────┼───────────────────────────────────┤
│ One-to-One    │ @OneToOne        │ User ←→ UserProfile              │
│ One-to-Many   │ @OneToMany       │ Category ←→ Products             │
│ Many-to-One   │ @ManyToOne       │ Products ←→ Category             │
│ Many-to-Many  │ @ManyToMany      │ Products ←→ Tags                 │
└───────────────┴─────────────────┴───────────────────────────────────┘
```

### 1.2. Owning Side vs Inverse Side

```
QUAN HỆ HAI CHIỀU (Bidirectional):

Category (Inverse Side)              Product (Owning Side)
┌──────────────────────┐             ┌──────────────────────┐
│ @OneToMany(          │             │ @ManyToOne           │
│   mappedBy="category"│ ←─────────→ │ @JoinColumn(         │
│ )                    │             │   name="category_id" │
│ List<Product>        │             │ )                    │
│ products             │             │ Category category    │
└──────────────────────┘             └──────────────────────┘
        │                                    │
        │                                    │
        └── KHÔNG chứa FK ───────────────────┴── CHỨA Foreign Key (category_id)

QUAN TRỌNG:
• Owning Side = bên CHỨA Foreign Key (có @JoinColumn)
• Inverse Side = bên có mappedBy
• Khi UPDATE relationship: phải SET ở Owning Side!
```

---

## 2. USER ENTITY

### 2.1. User Entity hoàn chỉnh

```java
package com.example.ecommerce.entity;

import com.example.ecommerce.entity.enums.UserRole;
import jakarta.persistence.*;
import lombok.*;

import java.util.ArrayList;
import java.util.List;

@Entity
@Table(name = "users", indexes = {
    @Index(name = "idx_user_email", columnList = "email", unique = true),
    @Index(name = "idx_user_phone", columnList = "phone")
})
@Getter
@Setter
@NoArgsConstructor
@AllArgsConstructor
@Builder
@ToString(exclude = {"orders", "addresses", "cartItems", "reviews"})
@EqualsAndHashCode(of = "id", callSuper = false)
public class User extends BaseEntity {

    @Column(nullable = false, unique = true, length = 100)
    private String email;

    @Column(nullable = false)
    private String password;  // BCrypt encoded

    @Column(name = "full_name", nullable = false, length = 100)
    private String fullName;

    @Column(length = 20)
    private String phone;

    @Column(name = "avatar_url")
    private String avatarUrl;

    @Enumerated(EnumType.STRING)
    @Column(nullable = false, length = 20)
    @Builder.Default
    private UserRole role = UserRole.CUSTOMER;

    @Column(name = "is_active", nullable = false)
    @Builder.Default
    private Boolean isActive = true;

    @Column(name = "is_email_verified", nullable = false)
    @Builder.Default
    private Boolean isEmailVerified = false;

    // ─── RELATIONSHIPS ───

    // User có nhiều Orders
    @OneToMany(mappedBy = "user", cascade = CascadeType.ALL)
    @Builder.Default
    private List<Order> orders = new ArrayList<>();

    // User có nhiều Addresses
    @OneToMany(mappedBy = "user", cascade = CascadeType.ALL, orphanRemoval = true)
    @Builder.Default
    private List<Address> addresses = new ArrayList<>();

    // User có nhiều CartItems
    @OneToMany(mappedBy = "user", cascade = CascadeType.ALL, orphanRemoval = true)
    @Builder.Default
    private List<CartItem> cartItems = new ArrayList<>();

    // User có nhiều Reviews
    @OneToMany(mappedBy = "user", cascade = CascadeType.ALL)
    @Builder.Default
    private List<Review> reviews = new ArrayList<>();

    // ─── HELPER METHODS ───

    public void addAddress(Address address) {
        addresses.add(address);
        address.setUser(this);
    }

    public void removeAddress(Address address) {
        addresses.remove(address);
        address.setUser(null);
    }

    public Address getDefaultAddress() {
        return addresses.stream()
                .filter(Address::getIsDefault)
                .findFirst()
                .orElse(null);
    }
}
```

### 2.2. Address Entity (One-to-Many)

```java
package com.example.ecommerce.entity;

import jakarta.persistence.*;
import lombok.*;

@Entity
@Table(name = "addresses")
@Getter
@Setter
@NoArgsConstructor
@AllArgsConstructor
@Builder
@ToString(exclude = "user")
@EqualsAndHashCode(of = "id", callSuper = false)
public class Address extends BaseEntity {

    @Column(name = "full_name", nullable = false, length = 100)
    private String fullName;

    @Column(nullable = false, length = 20)
    private String phone;

    @Column(name = "address_line", nullable = false)
    private String addressLine;

    @Column(length = 100)
    private String ward;  // Phường/Xã

    @Column(length = 100)
    private String district;  // Quận/Huyện

    @Column(nullable = false, length = 100)
    private String city;  // Tỉnh/Thành phố

    @Column(name = "postal_code", length = 20)
    private String postalCode;

    @Column(name = "is_default", nullable = false)
    @Builder.Default
    private Boolean isDefault = false;

    // ─── RELATIONSHIP ───

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "user_id", nullable = false)
    private User user;

    // ─── HELPER ───
    public String getFullAddress() {
        StringBuilder sb = new StringBuilder();
        sb.append(addressLine);
        if (ward != null) sb.append(", ").append(ward);
        if (district != null) sb.append(", ").append(district);
        sb.append(", ").append(city);
        return sb.toString();
    }
}
```

---

## 3. ORDER ENTITY

### 3.1. Order Entity (Complex Relationships)

```java
package com.example.ecommerce.entity;

import com.example.ecommerce.entity.enums.OrderStatus;
import com.example.ecommerce.entity.enums.PaymentMethod;
import com.example.ecommerce.entity.enums.PaymentStatus;
import jakarta.persistence.*;
import lombok.*;

import java.math.BigDecimal;
import java.time.LocalDateTime;
import java.util.ArrayList;
import java.util.List;

@Entity
@Table(name = "orders", indexes = {
    @Index(name = "idx_order_number", columnList = "order_number", unique = true),
    @Index(name = "idx_order_user", columnList = "user_id"),
    @Index(name = "idx_order_status", columnList = "status"),
    @Index(name = "idx_order_created", columnList = "created_at")
})
@Getter
@Setter
@NoArgsConstructor
@AllArgsConstructor
@Builder
@ToString(exclude = {"user", "items", "payment"})
@EqualsAndHashCode(of = "id", callSuper = false)
public class Order extends BaseEntity {

    @Column(name = "order_number", nullable = false, unique = true, length = 50)
    private String orderNumber;  // ORD-20240101-001

    @Enumerated(EnumType.STRING)
    @Column(nullable = false, length = 20)
    @Builder.Default
    private OrderStatus status = OrderStatus.PENDING;

    // ─── PRICE BREAKDOWN ───

    @Column(name = "subtotal", nullable = false, precision = 12, scale = 2)
    private BigDecimal subtotal;  // Tổng tiền sản phẩm

    @Column(name = "shipping_fee", precision = 12, scale = 2)
    @Builder.Default
    private BigDecimal shippingFee = BigDecimal.ZERO;

    @Column(name = "discount_amount", precision = 12, scale = 2)
    @Builder.Default
    private BigDecimal discountAmount = BigDecimal.ZERO;

    @Column(name = "total_amount", nullable = false, precision = 12, scale = 2)
    private BigDecimal totalAmount;  // subtotal + shippingFee - discountAmount

    // ─── SHIPPING INFO ───

    @Column(name = "shipping_name", nullable = false, length = 100)
    private String shippingName;

    @Column(name = "shipping_phone", nullable = false, length = 20)
    private String shippingPhone;

    @Column(name = "shipping_address", nullable = false)
    private String shippingAddress;

    // ─── PAYMENT INFO ───

    @Enumerated(EnumType.STRING)
    @Column(name = "payment_method", nullable = false, length = 20)
    private PaymentMethod paymentMethod;

    @Enumerated(EnumType.STRING)
    @Column(name = "payment_status", nullable = false, length = 20)
    @Builder.Default
    private PaymentStatus paymentStatus = PaymentStatus.PENDING;

    // ─── NOTES ───

    @Column(columnDefinition = "TEXT")
    private String note;

    @Column(name = "cancel_reason")
    private String cancelReason;

    @Column(name = "cancelled_at")
    private LocalDateTime cancelledAt;

    // ─── RELATIONSHIPS ───

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "user_id", nullable = false)
    private User user;

    // Order có nhiều OrderItems
    @OneToMany(mappedBy = "order", cascade = CascadeType.ALL, orphanRemoval = true)
    @Builder.Default
    private List<OrderItem> items = new ArrayList<>();

    // Order có 1 Payment (optional)
    @OneToOne(mappedBy = "order", cascade = CascadeType.ALL, fetch = FetchType.LAZY)
    private Payment payment;

    // ─── HELPER METHODS ───

    public void addItem(OrderItem item) {
        items.add(item);
        item.setOrder(this);
    }

    public void removeItem(OrderItem item) {
        items.remove(item);
        item.setOrder(null);
    }

    public void calculateTotals() {
        this.subtotal = items.stream()
                .map(OrderItem::getSubtotal)
                .reduce(BigDecimal.ZERO, BigDecimal::add);
        this.totalAmount = subtotal
                .add(shippingFee != null ? shippingFee : BigDecimal.ZERO)
                .subtract(discountAmount != null ? discountAmount : BigDecimal.ZERO);
    }

    public int getTotalItems() {
        return items.stream()
                .mapToInt(OrderItem::getQuantity)
                .sum();
    }

    @PrePersist
    public void prePersist() {
        if (orderNumber == null) {
            // Generate order number: ORD-yyyyMMdd-xxxxx
            orderNumber = "ORD-" + java.time.LocalDate.now().toString().replace("-", "")
                    + "-" + System.currentTimeMillis() % 100000;
        }
    }
}
```

### 3.2. OrderItem Entity

```java
package com.example.ecommerce.entity;

import jakarta.persistence.*;
import lombok.*;

import java.math.BigDecimal;

@Entity
@Table(name = "order_items")
@Getter
@Setter
@NoArgsConstructor
@AllArgsConstructor
@Builder
@ToString(exclude = {"order", "product"})
@EqualsAndHashCode(of = "id", callSuper = false)
public class OrderItem extends BaseEntity {

    @Column(nullable = false)
    private Integer quantity;

    @Column(name = "unit_price", nullable = false, precision = 12, scale = 2)
    private BigDecimal unitPrice;  // Giá tại thời điểm đặt hàng

    @Column(nullable = false, precision = 12, scale = 2)
    private BigDecimal subtotal;  // quantity * unitPrice

    // Lưu thông tin sản phẩm tại thời điểm đặt hàng (denormalized)
    @Column(name = "product_name", nullable = false)
    private String productName;

    @Column(name = "product_sku", nullable = false, length = 50)
    private String productSku;

    @Column(name = "product_image")
    private String productImage;

    // ─── RELATIONSHIPS ───

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "order_id", nullable = false)
    private Order order;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "product_id")  // nullable = có thể sản phẩm bị xóa
    private Product product;

    // ─── HELPER ───

    @PrePersist
    @PreUpdate
    public void calculateSubtotal() {
        this.subtotal = unitPrice.multiply(BigDecimal.valueOf(quantity));
    }
}
```

---

## 4. ONE-TO-ONE RELATIONSHIP

### 4.1. Payment Entity (One-to-One with Order)

```java
package com.example.ecommerce.entity;

import com.example.ecommerce.entity.enums.PaymentMethod;
import com.example.ecommerce.entity.enums.PaymentStatus;
import jakarta.persistence.*;
import lombok.*;

import java.math.BigDecimal;
import java.time.LocalDateTime;

@Entity
@Table(name = "payments", indexes = {
    @Index(name = "idx_payment_transaction", columnList = "transaction_id", unique = true)
})
@Getter
@Setter
@NoArgsConstructor
@AllArgsConstructor
@Builder
@ToString(exclude = "order")
@EqualsAndHashCode(of = "id", callSuper = false)
public class Payment extends BaseEntity {

    @Column(nullable = false, precision = 12, scale = 2)
    private BigDecimal amount;

    @Enumerated(EnumType.STRING)
    @Column(nullable = false, length = 20)
    private PaymentMethod method;

    @Enumerated(EnumType.STRING)
    @Column(nullable = false, length = 20)
    @Builder.Default
    private PaymentStatus status = PaymentStatus.PENDING;

    @Column(name = "transaction_id", unique = true, length = 100)
    private String transactionId;  // ID từ payment provider (Stripe, VNPay)

    @Column(length = 50)
    private String provider;  // stripe, vnpay, momo

    @Column(name = "paid_at")
    private LocalDateTime paidAt;

    @Column(name = "refunded_at")
    private LocalDateTime refundedAt;

    @Column(name = "failure_reason")
    private String failureReason;

    @Column(name = "raw_response", columnDefinition = "TEXT")
    private String rawResponse;  // JSON response từ provider

    // ─── ONE-TO-ONE RELATIONSHIP ───
    // Payment là Owning Side (chứa FK)

    @OneToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "order_id", nullable = false, unique = true)
    private Order order;

    // ─── HELPER ───

    public void markAsPaid(String transactionId) {
        this.transactionId = transactionId;
        this.status = PaymentStatus.PAID;
        this.paidAt = LocalDateTime.now();
    }

    public void markAsFailed(String reason) {
        this.status = PaymentStatus.FAILED;
        this.failureReason = reason;
    }
}
```

### 4.2. One-to-One: Owning Side vs Inverse Side

```java
// ─── ORDER ENTITY (Inverse Side) ───
@Entity
public class Order {
    // Order KHÔNG chứa FK
    // Dùng mappedBy để chỉ ra Payment là owning side
    @OneToOne(mappedBy = "order", cascade = CascadeType.ALL, fetch = FetchType.LAZY)
    private Payment payment;
}

// ─── PAYMENT ENTITY (Owning Side) ───
@Entity
public class Payment {
    // Payment CHỨA FK (order_id)
    @OneToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "order_id", nullable = false, unique = true)
    private Order order;
}

// Khi tạo mới:
Order order = new Order();
Payment payment = new Payment();
payment.setOrder(order);  // SET ở Owning Side!
order.setPayment(payment); // SET ở cả 2 bên (bidirectional)
orderRepository.save(order); // Cascade sẽ save payment
```

---

## 5. MANY-TO-MANY RELATIONSHIP

### 5.1. Product ↔ Tag (Many-to-Many)

```java
// ─── TAG ENTITY ───
package com.example.ecommerce.entity;

import jakarta.persistence.*;
import lombok.*;

import java.util.HashSet;
import java.util.Set;

@Entity
@Table(name = "tags")
@Getter
@Setter
@NoArgsConstructor
@AllArgsConstructor
@Builder
@ToString(exclude = "products")
@EqualsAndHashCode(of = "id", callSuper = false)
public class Tag extends BaseEntity {

    @Column(nullable = false, unique = true, length = 50)
    private String name;

    @Column(nullable = false, unique = true, length = 50)
    private String slug;

    // ─── MANY-TO-MANY (Inverse Side) ───
    @ManyToMany(mappedBy = "tags")
    @Builder.Default
    private Set<Product> products = new HashSet<>();
}
```

```java
// ─── PRODUCT ENTITY (thêm tags) ───
@Entity
public class Product extends BaseEntity {

    // ... các field khác ...

    // ─── MANY-TO-MANY (Owning Side) ───
    @ManyToMany(cascade = {CascadeType.PERSIST, CascadeType.MERGE})
    @JoinTable(
        name = "product_tags",  // Tên join table
        joinColumns = @JoinColumn(name = "product_id"),
        inverseJoinColumns = @JoinColumn(name = "tag_id")
    )
    @Builder.Default
    private Set<Tag> tags = new HashSet<>();

    // ─── HELPER METHODS ───
    public void addTag(Tag tag) {
        tags.add(tag);
        tag.getProducts().add(this);
    }

    public void removeTag(Tag tag) {
        tags.remove(tag);
        tag.getProducts().remove(this);
    }
}
```

### 5.2. Join Table được tạo

```sql
-- Hibernate tạo join table:
CREATE TABLE product_tags (
    product_id BIGINT NOT NULL,
    tag_id BIGINT NOT NULL,
    PRIMARY KEY (product_id, tag_id),
    FOREIGN KEY (product_id) REFERENCES products(id),
    FOREIGN KEY (tag_id) REFERENCES tags(id)
);
```

---

## 6. FETCH TYPE: LAZY vs EAGER

### 6.1. FetchType mặc định

| Relationship | Default FetchType |
|-------------|------------------|
| `@OneToOne` | **EAGER** ⚠️ |
| `@ManyToOne` | **EAGER** ⚠️ |
| `@OneToMany` | LAZY ✅ |
| `@ManyToMany` | LAZY ✅ |

**QUAN TRỌNG:** Luôn set `FetchType.LAZY` cho `@OneToOne` và `@ManyToOne`!

### 6.2. LAZY Loading hoạt động thế nào?

```java
// LAZY Loading = Load khi CẦN (proxy)
@ManyToOne(fetch = FetchType.LAZY)
private Category category;

// Khi query Product:
Product product = productRepository.findById(1L).orElseThrow();
// SQL: SELECT * FROM products WHERE id = 1
// category chưa được load (là proxy object)

// Khi access category:
String categoryName = product.getCategory().getName();  // TRIGGER QUERY
// SQL: SELECT * FROM categories WHERE id = ?
// Bây giờ mới load category từ DB
```

### 6.3. N+1 Problem

```java
// ❌ N+1 PROBLEM
List<Product> products = productRepository.findAll();  // 1 query
for (Product p : products) {
    System.out.println(p.getCategory().getName());     // N queries!
}
// Kết quả: 1 + N queries (nếu có 100 products = 101 queries!)

// ✅ GIẢI PHÁP 1: JOIN FETCH trong JPQL
@Query("SELECT p FROM Product p JOIN FETCH p.category")
List<Product> findAllWithCategory();

// ✅ GIẢI PHÁP 2: @EntityGraph
@EntityGraph(attributePaths = {"category"})
List<Product> findAll();

// ✅ GIẢI PHÁP 3: Batch Fetching
// application.yml:
spring.jpa.properties.hibernate.default_batch_fetch_size: 50
```

---

## 7. CASCADE & ORPHAN REMOVAL

### 7.1. CascadeType chi tiết

```java
// CASCADE = khi thao tác trên parent → cũng thao tác trên children

@OneToMany(mappedBy = "order", cascade = CascadeType.ALL)
private List<OrderItem> items;

// CascadeType.ALL = PERSIST + MERGE + REMOVE + REFRESH + DETACH
```

```
┌────────────────────────────────────────────────────────┐
│                    CASCADE TYPES                        │
├────────────────────┬───────────────────────────────────┤
│ PERSIST            │ save(parent) → save(children)     │
│ MERGE              │ update(parent) → update(children)  │
│ REMOVE             │ delete(parent) → delete(children)  │
│ REFRESH            │ refresh(parent) → refresh(children)│
│ DETACH             │ detach(parent) → detach(children)  │
│ ALL                │ Tất cả các loại trên              │
└────────────────────┴───────────────────────────────────┘
```

### 7.2. orphanRemoval = true

```java
@OneToMany(mappedBy = "user", cascade = CascadeType.ALL, orphanRemoval = true)
private List<Address> addresses;

// orphanRemoval = true:
// Khi address bị remove khỏi list → DELETE khỏi database!

user.getAddresses().remove(address);  // address bị xóa khỏi list
userRepository.save(user);             // DELETE FROM addresses WHERE id = ?
```

**Khác biệt CASCADE.REMOVE vs orphanRemoval:**

```
CASCADE.REMOVE:
- Chỉ xóa children khi XÓA PARENT
- Remove khỏi list → KHÔNG xóa khỏi DB

orphanRemoval = true:
- Xóa children khi XÓA PARENT (như CASCADE.REMOVE)
- Remove khỏi list → CÓ xóa khỏi DB (orphan = mồ côi)
```

---

## 8. CART ITEM ENTITY

```java
package com.example.ecommerce.entity;

import jakarta.persistence.*;
import lombok.*;

@Entity
@Table(name = "cart_items", uniqueConstraints = {
    @UniqueConstraint(columnNames = {"user_id", "product_id"})
})
@Getter
@Setter
@NoArgsConstructor
@AllArgsConstructor
@Builder
@ToString(exclude = {"user", "product"})
@EqualsAndHashCode(of = "id", callSuper = false)
public class CartItem extends BaseEntity {

    @Column(nullable = false)
    @Builder.Default
    private Integer quantity = 1;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "user_id", nullable = false)
    private User user;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "product_id", nullable = false)
    private Product product;

    // ─── HELPER ───

    public void increaseQuantity(int amount) {
        this.quantity += amount;
    }

    public void decreaseQuantity(int amount) {
        this.quantity = Math.max(1, this.quantity - amount);
    }
}
```

---

## 9. REVIEW ENTITY

```java
package com.example.ecommerce.entity;

import jakarta.persistence.*;
import lombok.*;

@Entity
@Table(name = "reviews", uniqueConstraints = {
    @UniqueConstraint(columnNames = {"user_id", "product_id"})
}, indexes = {
    @Index(name = "idx_review_product", columnList = "product_id"),
    @Index(name = "idx_review_rating", columnList = "rating")
})
@Getter
@Setter
@NoArgsConstructor
@AllArgsConstructor
@Builder
@ToString(exclude = {"user", "product"})
@EqualsAndHashCode(of = "id", callSuper = false)
public class Review extends BaseEntity {

    @Column(nullable = false)
    private Integer rating;  // 1-5 stars

    @Column(columnDefinition = "TEXT")
    private String comment;

    @Column(name = "is_verified_purchase", nullable = false)
    @Builder.Default
    private Boolean isVerifiedPurchase = false;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "user_id", nullable = false)
    private User user;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "product_id", nullable = false)
    private Product product;
}
```

---

## 10. BÀI TẬP & CHECKLIST

### Bài tập thực hành

- [ ] Tạo `User` entity với relationships
- [ ] Tạo `Address` entity (One-to-Many with User)
- [ ] Tạo `Order` entity (complex relationships)
- [ ] Tạo `OrderItem` entity
- [ ] Tạo `Payment` entity (One-to-One with Order)
- [ ] Tạo `Tag` entity (Many-to-Many with Product)
- [ ] Thêm `tags` relationship vào `Product`
- [ ] Tạo `CartItem` entity
- [ ] Tạo `Review` entity
- [ ] Chạy ứng dụng → kiểm tra tất cả tables được tạo

### Kiến thức cần nắm

| Khái niệm | Hiểu chưa? |
|-----------|-------------|
| Owning Side vs Inverse Side | ☐ |
| mappedBy là gì | ☐ |
| @JoinColumn vs @JoinTable | ☐ |
| FetchType.LAZY vs EAGER | ☐ |
| N+1 Problem và cách giải quyết | ☐ |
| CascadeType (PERSIST, REMOVE, ALL) | ☐ |
| orphanRemoval = true | ☐ |
| @PrePersist, @PreUpdate hooks | ☐ |
| Bidirectional relationship maintenance | ☐ |

---

**Trước đó:** [← Ngày 05 - JPA Setup](../week-01/day-05-jpa-setup.md)

**Tiếp theo:** [Ngày 07 - Flyway Migration & Data Seeding →](day-07-flyway-migration.md)
