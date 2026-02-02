# Ngày 05: MySQL + Spring Data JPA Setup

## Mục tiêu hôm nay
- Cấu hình kết nối MySQL với HikariCP connection pool
- Hiểu JPA, Hibernate, Spring Data JPA và mối quan hệ
- Tạo Entity đầu tiên và ánh xạ với database table
- Tạo Repository interface để truy vấn dữ liệu
- Hiểu ddl-auto options và khi nào dùng

---

## 1. HIỂU JPA - HIBERNATE - SPRING DATA JPA

### 1.1. JPA là gì?

**JPA (Jakarta Persistence API)** = Đặc tả (specification) chuẩn cho ORM trong Java

```
┌─────────────────────────────────────────┐
│              JPA (Specification)         │
│                                          │
│  • Định nghĩa các annotation:            │
│    @Entity, @Table, @Column, @Id...      │
│  • Định nghĩa API:                       │
│    EntityManager, Query, Transaction...   │
│  • KHÔNG PHẢI implementation!            │
│                                          │
└─────────────────────────────────────────┘
          │
          │  Implementations (thực thi)
          │
    ┌─────┴──────┬───────────────┐
    │            │               │
┌───▼───┐   ┌────▼────┐   ┌──────▼─────┐
│Hibern │   │EclipseL │   │  OpenJPA   │
│ate    │   │ink      │   │            │
│       │   │         │   │            │
│ PHỔ   │   │         │   │            │
│ BIẾN  │   │         │   │            │
│ NHẤT  │   │         │   │            │
└───────┘   └─────────┘   └────────────┘
```

### 1.2. Hibernate là gì?

**Hibernate** = Implementation PHỔ BIẾN NHẤT của JPA

```
Java Object  ←──── Hibernate (ORM) ────→  Database Table

Product.java           ↔               products table
──────────────                         ────────────────
id: Long               ↔               id BIGINT
name: String           ↔               name VARCHAR(255)
price: BigDecimal      ↔               price DECIMAL(10,2)
createdAt: LocalDateTime ↔             created_at DATETIME
```

### 1.3. Spring Data JPA là gì?

**Spring Data JPA** = Layer TRỪU TƯỢNG HÓA phía trên JPA/Hibernate

```
┌─────────────────────────────────────────┐
│           Spring Data JPA                │
│  • Repository interface (JpaRepository)  │
│  • Derived Query Methods                 │
│  • @Query annotation                     │
│  • Pagination, Sorting                   │
│  • Auditing (@CreatedDate, @LastModified)│
└───────────────────┬─────────────────────┘
                    │
┌───────────────────▼─────────────────────┐
│               JPA API                     │
│  (EntityManager, Query, Transaction)      │
└───────────────────┬─────────────────────┘
                    │
┌───────────────────▼─────────────────────┐
│              Hibernate                    │
│  (SessionFactory, Session, Criteria)      │
└───────────────────┬─────────────────────┘
                    │
┌───────────────────▼─────────────────────┐
│               JDBC                        │
│  (Connection, Statement, ResultSet)       │
└───────────────────┬─────────────────────┘
                    │
                    ▼
              MySQL Database
```

**Lợi ích Spring Data JPA:**

| Không có Spring Data JPA | Với Spring Data JPA |
|------------------------|-------------------|
| 50 dòng code tạo connection | 0 dòng (auto-config) |
| 20 dòng code cho mỗi CRUD method | 0 dòng (JpaRepository có sẵn) |
| Tự viết SQL queries | Derived Query Methods (tự sinh SQL từ method name) |
| Tự xử lý pagination | Pageable + Page<T> có sẵn |
| Tự map ResultSet → Object | Tự động |

---

## 2. CẤU HÌNH MYSQL CONNECTION

### 2.1. Dependencies (đã có trong pom.xml)

```xml
<!-- Spring Data JPA + Hibernate -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-jpa</artifactId>
</dependency>

<!-- MySQL Driver -->
<dependency>
    <groupId>com.mysql</groupId>
    <artifactId>mysql-connector-j</artifactId>
    <scope>runtime</scope>
</dependency>
```

### 2.2. application.yml configuration

```yaml
spring:
  datasource:
    url: jdbc:mysql://localhost:3306/ecommerce?useSSL=false&serverTimezone=UTC&allowPublicKeyRetrieval=true&characterEncoding=UTF-8
    username: root
    password: 1q2w3E*
    driver-class-name: com.mysql.cj.jdbc.Driver

    # HikariCP Connection Pool (Spring Boot default)
    hikari:
      pool-name: EcommerceHikariPool
      maximum-pool-size: 10          # Max connections
      minimum-idle: 5                 # Min idle connections
      idle-timeout: 300000            # 5 phút - connection idle bị đóng
      max-lifetime: 1800000           # 30 phút - max lifetime của connection
      connection-timeout: 20000       # 20s - timeout khi lấy connection từ pool
      leak-detection-threshold: 60000 # 60s - cảnh báo connection leak

  jpa:
    # Hibernate settings
    hibernate:
      ddl-auto: update                # Tự động update schema (chỉ cho dev!)
      naming:
        physical-strategy: org.hibernate.boot.model.naming.CamelCaseToUnderscoresNamingStrategy
        # Java: productName → DB: product_name

    # SQL logging
    show-sql: true
    properties:
      hibernate:
        format_sql: true              # Format SQL cho dễ đọc
        use_sql_comments: true        # Thêm comment vào SQL
        dialect: org.hibernate.dialect.MySQLDialect

        # Batch operations (performance)
        jdbc:
          batch_size: 50
          batch_versioned_data: true
        order_inserts: true
        order_updates: true

    # Khởi tạo database
    defer-datasource-initialization: true

  sql:
    init:
      mode: never  # never | always | embedded
```

### 2.3. HikariCP Connection Pool

```
┌────────────────────────────────────────────────────┐
│             HikariCP Connection Pool                │
│                                                     │
│  ┌──────────────────────────────────────────────┐  │
│  │  Idle Connections (minimum-idle: 5)           │  │
│  │  ┌───┐ ┌───┐ ┌───┐ ┌───┐ ┌───┐              │  │
│  │  │C1 │ │C2 │ │C3 │ │C4 │ │C5 │ ← Sẵn sàng  │  │
│  │  └───┘ └───┘ └───┘ └───┘ └───┘              │  │
│  └──────────────────────────────────────────────┘  │
│                                                     │
│  ┌──────────────────────────────────────────────┐  │
│  │  Active Connections (đang được dùng)          │  │
│  │  ┌───┐ ┌───┐ ┌───┐                           │  │
│  │  │C6 │ │C7 │ │C8 │ ← Thread đang dùng       │  │
│  │  └───┘ └───┘ └───┘                           │  │
│  └──────────────────────────────────────────────┘  │
│                                                     │
│  Max Pool Size: 10                                  │
│  Có thể tạo thêm: 10 - 8 = 2 connections           │
│                                                     │
└────────────────────────────────────────────────────┘

Flow:
1. Thread cần DB → Lấy connection từ pool (idle → active)
2. Thread xong → Trả connection về pool (active → idle)
3. Pool đầy → Thread phải đợi (connection-timeout)
4. Connection quá lâu → Đóng bỏ (max-lifetime)
```

### 2.4. ddl-auto Options

| Giá trị | Hành động | Dùng khi |
|---------|-----------|----------|
| `none` | Không làm gì với schema | **Production** |
| `validate` | Chỉ kiểm tra schema khớp Entity | **Staging** |
| `update` | Thêm column/table mới, KHÔNG xóa | **Development** |
| `create` | Drop + Create lại mỗi lần start | Testing |
| `create-drop` | Như create, + drop khi shutdown | Unit Test |

**⚠️ CẢNH BÁO:** KHÔNG BAO GIỜ dùng `update`, `create`, `create-drop` trong PRODUCTION!

---

## 3. TẠO ENTITY ĐẦU TIÊN

### 3.1. BaseEntity (đã tạo)

```java
package com.example.ecommerce.entity;

import jakarta.persistence.*;
import lombok.Getter;
import lombok.Setter;
import org.hibernate.annotations.CreationTimestamp;
import org.hibernate.annotations.UpdateTimestamp;

import java.time.LocalDateTime;

@Getter
@Setter
@MappedSuperclass
public abstract class BaseEntity {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @CreationTimestamp
    @Column(name = "created_at", nullable = false, updatable = false)
    private LocalDateTime createdAt;

    @UpdateTimestamp
    @Column(name = "updated_at")
    private LocalDateTime updatedAt;
}
```

### 3.2. Category Entity

```java
package com.example.ecommerce.entity;

import jakarta.persistence.*;
import lombok.*;

import java.util.ArrayList;
import java.util.List;

@Entity
@Table(name = "categories", indexes = {
    @Index(name = "idx_category_slug", columnList = "slug", unique = true),
    @Index(name = "idx_category_parent", columnList = "parent_id")
})
@Getter
@Setter
@NoArgsConstructor
@AllArgsConstructor
@Builder
@ToString(exclude = {"parent", "children", "products"})
@EqualsAndHashCode(of = "id", callSuper = false)
public class Category extends BaseEntity {

    @Column(nullable = false, length = 100)
    private String name;

    @Column(nullable = false, unique = true, length = 100)
    private String slug;

    @Column(columnDefinition = "TEXT")
    private String description;

    @Column(name = "image_url")
    private String imageUrl;

    @Column(name = "is_active", nullable = false)
    @Builder.Default
    private Boolean isActive = true;

    @Column(name = "sort_order")
    @Builder.Default
    private Integer sortOrder = 0;

    // Self-referencing relationship (Category tree)
    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "parent_id")
    private Category parent;

    @OneToMany(mappedBy = "parent", cascade = CascadeType.ALL)
    @Builder.Default
    private List<Category> children = new ArrayList<>();

    // Relationship with Product (sẽ thêm sau)
    @OneToMany(mappedBy = "category", cascade = CascadeType.ALL)
    @Builder.Default
    private List<Product> products = new ArrayList<>();
}
```

### 3.3. Product Entity

```java
package com.example.ecommerce.entity;

import jakarta.persistence.*;
import lombok.*;

import java.math.BigDecimal;
import java.util.ArrayList;
import java.util.List;

@Entity
@Table(name = "products", indexes = {
    @Index(name = "idx_product_slug", columnList = "slug", unique = true),
    @Index(name = "idx_product_sku", columnList = "sku", unique = true),
    @Index(name = "idx_product_category", columnList = "category_id"),
    @Index(name = "idx_product_price", columnList = "price"),
    @Index(name = "idx_product_active", columnList = "is_active")
})
@Getter
@Setter
@NoArgsConstructor
@AllArgsConstructor
@Builder
@ToString(exclude = {"category", "images", "reviews"})
@EqualsAndHashCode(of = "id", callSuper = false)
public class Product extends BaseEntity {

    @Column(nullable = false, length = 200)
    private String name;

    @Column(nullable = false, unique = true, length = 200)
    private String slug;

    @Column(nullable = false, unique = true, length = 50)
    private String sku;  // Stock Keeping Unit

    @Column(columnDefinition = "TEXT")
    private String description;

    @Column(nullable = false, precision = 12, scale = 2)
    private BigDecimal price;

    @Column(name = "sale_price", precision = 12, scale = 2)
    private BigDecimal salePrice;

    @Column(name = "stock_quantity", nullable = false)
    @Builder.Default
    private Integer stockQuantity = 0;

    @Column(name = "is_active", nullable = false)
    @Builder.Default
    private Boolean isActive = true;

    @Column(name = "is_featured", nullable = false)
    @Builder.Default
    private Boolean isFeatured = false;

    @Column(name = "main_image")
    private String mainImage;

    // Relationship with Category
    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "category_id")
    private Category category;

    // Relationship with ProductImage
    @OneToMany(mappedBy = "product", cascade = CascadeType.ALL, orphanRemoval = true)
    @Builder.Default
    private List<ProductImage> images = new ArrayList<>();

    // Helper methods
    public void addImage(ProductImage image) {
        images.add(image);
        image.setProduct(this);
    }

    public void removeImage(ProductImage image) {
        images.remove(image);
        image.setProduct(null);
    }

    public BigDecimal getEffectivePrice() {
        return salePrice != null && salePrice.compareTo(BigDecimal.ZERO) > 0
                ? salePrice
                : price;
    }

    public boolean isInStock() {
        return stockQuantity > 0;
    }
}
```

### 3.4. ProductImage Entity

```java
package com.example.ecommerce.entity;

import jakarta.persistence.*;
import lombok.*;

@Entity
@Table(name = "product_images")
@Getter
@Setter
@NoArgsConstructor
@AllArgsConstructor
@Builder
@ToString(exclude = "product")
@EqualsAndHashCode(of = "id", callSuper = false)
public class ProductImage extends BaseEntity {

    @Column(nullable = false)
    private String url;

    @Column(name = "alt_text")
    private String altText;

    @Column(name = "sort_order")
    @Builder.Default
    private Integer sortOrder = 0;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "product_id", nullable = false)
    private Product product;
}
```

---

## 4. JPA ANNOTATIONS GIẢI THÍCH

### 4.1. Entity & Table

```java
@Entity                              // Đánh dấu là JPA Entity
@Table(
    name = "products",               // Tên table trong DB
    indexes = {                       // Các index
        @Index(name = "idx_slug", columnList = "slug", unique = true)
    },
    uniqueConstraints = {             // Unique constraints
        @UniqueConstraint(columnNames = {"sku"})
    }
)
public class Product extends BaseEntity { }
```

### 4.2. ID & Generation Strategy

```java
@Id                                   // Primary key
@GeneratedValue(strategy = GenerationType.IDENTITY)  // Auto-increment
private Long id;

// Các strategy:
// IDENTITY - Database auto-increment (MySQL, PostgreSQL)
// SEQUENCE - Database sequence (PostgreSQL, Oracle)
// TABLE    - Dùng table riêng để sinh ID
// UUID     - UUID (không cần @GeneratedValue)
// AUTO     - Hibernate tự chọn (không khuyến khích)
```

### 4.3. Column Mapping

```java
@Column(
    name = "product_name",           // Tên column trong DB
    nullable = false,                 // NOT NULL
    unique = true,                    // UNIQUE constraint
    length = 200,                     // VARCHAR(200)
    updatable = false,                // Không cho UPDATE
    insertable = true,                // Cho INSERT
    columnDefinition = "TEXT"         // Override toàn bộ DDL
)
private String name;

// Số với precision + scale
@Column(precision = 12, scale = 2)   // DECIMAL(12,2)
private BigDecimal price;
```

### 4.4. Relationships

```java
// ─── ONE TO MANY ───
// 1 Category có nhiều Products
@OneToMany(
    mappedBy = "category",           // Field ở phía Many
    cascade = CascadeType.ALL,        // Cascade operations
    fetch = FetchType.LAZY            // LAZY by default
)
private List<Product> products;

// ─── MANY TO ONE ───
// Nhiều Products thuộc 1 Category
@ManyToOne(fetch = FetchType.LAZY)   // EAGER by default, nên set LAZY
@JoinColumn(
    name = "category_id",             // Foreign key column
    nullable = true                   // Có thể null (no category)
)
private Category category;

// ─── MANY TO MANY ───
@ManyToMany
@JoinTable(
    name = "product_tags",            // Join table name
    joinColumns = @JoinColumn(name = "product_id"),
    inverseJoinColumns = @JoinColumn(name = "tag_id")
)
private Set<Tag> tags;

// ─── ONE TO ONE ───
@OneToOne(cascade = CascadeType.ALL, fetch = FetchType.LAZY)
@JoinColumn(name = "profile_id", unique = true)
private UserProfile profile;
```

### 4.5. Cascade Types

```
CascadeType.ALL     = Tất cả operations
CascadeType.PERSIST = Khi save parent → save children
CascadeType.MERGE   = Khi update parent → update children
CascadeType.REMOVE  = Khi delete parent → delete children
CascadeType.REFRESH = Khi refresh parent → refresh children
CascadeType.DETACH  = Khi detach parent → detach children

Ví dụ:
Category category = new Category("Electronics");
category.getProducts().add(new Product("iPhone"));
category.getProducts().add(new Product("MacBook"));
categoryRepository.save(category);  // → CŨNG save 2 products (CASCADE.PERSIST)
```

---

## 5. TẠO REPOSITORY

### 5.1. ProductRepository

```java
package com.example.ecommerce.repository;

import com.example.ecommerce.entity.Product;
import org.springframework.data.domain.Page;
import org.springframework.data.domain.Pageable;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.data.jpa.repository.JpaSpecificationExecutor;
import org.springframework.data.jpa.repository.Query;
import org.springframework.data.repository.query.Param;
import org.springframework.stereotype.Repository;

import java.math.BigDecimal;
import java.util.List;
import java.util.Optional;

@Repository
public interface ProductRepository extends JpaRepository<Product, Long>,
        JpaSpecificationExecutor<Product> {

    // ─── DERIVED QUERY METHODS ───
    // Spring Data tự sinh SQL từ method name

    // SELECT * FROM products WHERE slug = ?
    Optional<Product> findBySlug(String slug);

    // SELECT * FROM products WHERE sku = ?
    Optional<Product> findBySku(String sku);

    // SELECT * FROM products WHERE is_active = true
    List<Product> findByIsActiveTrue();

    // SELECT * FROM products WHERE is_featured = true AND is_active = true
    List<Product> findByIsFeaturedTrueAndIsActiveTrue();

    // SELECT * FROM products WHERE category_id = ? AND is_active = true
    List<Product> findByCategoryIdAndIsActiveTrue(Long categoryId);

    // SELECT * FROM products WHERE price BETWEEN ? AND ?
    List<Product> findByPriceBetween(BigDecimal minPrice, BigDecimal maxPrice);

    // SELECT * FROM products WHERE name LIKE '%keyword%' OR description LIKE '%keyword%'
    List<Product> findByNameContainingIgnoreCaseOrDescriptionContainingIgnoreCase(
            String name, String description);

    // SELECT COUNT(*) FROM products WHERE category_id = ?
    long countByCategoryId(Long categoryId);

    // SELECT EXISTS(SELECT 1 FROM products WHERE slug = ?)
    boolean existsBySlug(String slug);

    // ─── @QUERY - JPQL ───
    @Query("SELECT p FROM Product p WHERE p.isActive = true ORDER BY p.createdAt DESC")
    List<Product> findLatestProducts(Pageable pageable);

    @Query("SELECT p FROM Product p WHERE p.category.id = :categoryId AND p.isActive = true")
    Page<Product> findActiveProductsByCategory(
            @Param("categoryId") Long categoryId,
            Pageable pageable);

    @Query("""
        SELECT p FROM Product p
        WHERE (:categoryId IS NULL OR p.category.id = :categoryId)
        AND (:minPrice IS NULL OR p.price >= :minPrice)
        AND (:maxPrice IS NULL OR p.price <= :maxPrice)
        AND p.isActive = true
        """)
    Page<Product> searchProducts(
            @Param("categoryId") Long categoryId,
            @Param("minPrice") BigDecimal minPrice,
            @Param("maxPrice") BigDecimal maxPrice,
            Pageable pageable);

    // ─── @QUERY - NATIVE SQL ───
    @Query(
        value = "SELECT * FROM products WHERE MATCH(name, description) AGAINST(:keyword IN BOOLEAN MODE)",
        nativeQuery = true)
    List<Product> fullTextSearch(@Param("keyword") String keyword);

    // ─── MODIFYING QUERIES ───
    @Modifying
    @Query("UPDATE Product p SET p.isActive = false WHERE p.id = :id")
    void softDelete(@Param("id") Long id);

    @Modifying
    @Query("UPDATE Product p SET p.stockQuantity = p.stockQuantity - :quantity WHERE p.id = :id AND p.stockQuantity >= :quantity")
    int decreaseStock(@Param("id") Long id, @Param("quantity") int quantity);
}
```

### 5.2. CategoryRepository

```java
package com.example.ecommerce.repository;

import com.example.ecommerce.entity.Category;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.data.jpa.repository.Query;
import org.springframework.stereotype.Repository;

import java.util.List;
import java.util.Optional;

@Repository
public interface CategoryRepository extends JpaRepository<Category, Long> {

    Optional<Category> findBySlug(String slug);

    List<Category> findByIsActiveTrueOrderBySortOrderAsc();

    // Lấy root categories (không có parent)
    List<Category> findByParentIsNullAndIsActiveTrueOrderBySortOrderAsc();

    // Lấy children của 1 category
    List<Category> findByParentIdAndIsActiveTrueOrderBySortOrderAsc(Long parentId);

    // Lấy category kèm products (tránh N+1)
    @Query("SELECT c FROM Category c LEFT JOIN FETCH c.products WHERE c.id = :id")
    Optional<Category> findByIdWithProducts(Long id);

    boolean existsBySlug(String slug);
}
```

---

## 6. TEST KẾT NỐI DATABASE

### 6.1. Chạy ứng dụng

```bash
# Đảm bảo MySQL đang chạy
docker ps  # Phải thấy ecommerce-mysql container

# Chạy ứng dụng
./mvnw spring-boot:run
```

**Output mong đợi:**
```
HikariPool-1 - Starting...
HikariPool-1 - Added connection com.mysql.cj.jdbc.ConnectionImpl@xxx
HikariPool-1 - Start completed.

Hibernate:
    create table categories (
        id bigint not null auto_increment,
        created_at datetime(6) not null,
        updated_at datetime(6),
        description TEXT,
        image_url varchar(255),
        is_active bit not null,
        name varchar(100) not null,
        slug varchar(100) not null,
        sort_order integer,
        parent_id bigint,
        primary key (id)
    ) engine=InnoDB

Hibernate:
    create table products (
        id bigint not null auto_increment,
        created_at datetime(6) not null,
        updated_at datetime(6),
        description TEXT,
        is_active bit not null,
        is_featured bit not null,
        main_image varchar(255),
        name varchar(200) not null,
        price decimal(12,2) not null,
        sale_price decimal(12,2),
        sku varchar(50) not null,
        slug varchar(200) not null,
        stock_quantity integer not null,
        category_id bigint,
        primary key (id)
    ) engine=InnoDB
```

### 6.2. Kiểm tra trong MySQL

```bash
# Kết nối MySQL
docker exec -it ecommerce-mysql mysql -uroot -p1q2w3E* ecommerce

# Kiểm tra tables
SHOW TABLES;
# +----------------------+
# | Tables_in_ecommerce  |
# +----------------------+
# | categories           |
# | product_images       |
# | products             |
# +----------------------+

# Xem cấu trúc table
DESCRIBE products;

# Xem indexes
SHOW INDEX FROM products;
```

---

## 7. BÀI TẬP & CHECKLIST

### Bài tập thực hành

- [ ] Cấu hình MySQL connection trong application.yml
- [ ] Tạo `BaseEntity` với id, createdAt, updatedAt
- [ ] Tạo `Category` entity với self-referencing (parent-child)
- [ ] Tạo `Product` entity với relationship tới Category
- [ ] Tạo `ProductImage` entity
- [ ] Tạo `ProductRepository` với derived query methods
- [ ] Tạo `CategoryRepository`
- [ ] Chạy ứng dụng → kiểm tra Hibernate tạo tables
- [ ] Kết nối MySQL → xác nhận tables + indexes

### Kiến thức cần nắm

| Khái niệm | Hiểu chưa? |
|-----------|-------------|
| JPA vs Hibernate vs Spring Data JPA | ☐ |
| HikariCP connection pool | ☐ |
| ddl-auto options (none, update, create) | ☐ |
| @Entity, @Table, @Column | ☐ |
| @Id, @GeneratedValue strategies | ☐ |
| @ManyToOne, @OneToMany, fetch type | ☐ |
| CascadeType (PERSIST, REMOVE, ALL) | ☐ |
| JpaRepository interface | ☐ |
| Derived Query Methods | ☐ |
| @Query (JPQL vs Native SQL) | ☐ |

---

**Trước đó:** [← Ngày 04 - Configuration](day-04-configuration.md)

**Tiếp theo:** [Ngày 06 - JPA Entity & Relationships →](../week-02/day-06-jpa-entity.md)
