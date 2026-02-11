# Day 2: IoC/DI + Configuration + JPA Setup

> **Mục tiêu cuối ngày:** Hiểu Dependency Injection, cấu hình ứng dụng đúng cách, và tạo xong Category/Product entities trong MySQL.

---

## 1. IoC & Dependency Injection

### 1.1 IoC là gì?

**Inversion of Control** — đảo ngược quyền điều khiển.

```
Không có IoC (tự nấu):   Bạn → đi chợ → mua nguyên liệu → nấu → rửa bát → ăn
Có IoC (nhà hàng):       Bạn → gọi món → ăn (nhà hàng lo phần còn lại)
```

**Spring Container** chính là "nhà hàng" — tạo object, quản lý vòng đời, và **inject** dependencies. Bạn không cần `new` thủ công.

```
Không có IoC:  OrderService svc = new OrderService(new ProductRepo(), new EmailService());
Có IoC:        @Autowired OrderService svc;  // Spring tự tạo và inject
```

### 1.2 Dependency Injection — 3 kiểu

#### Constructor Injection (Khuyến nghị)

```java
@Service
public class ProductService {
    private final ProductRepository productRepository;
    private final CategoryRepository categoryRepository;

    // Nếu chỉ có 1 constructor → không cần @Autowired
    public ProductService(ProductRepository productRepository,
                          CategoryRepository categoryRepository) {
        this.productRepository = productRepository;
        this.categoryRepository = categoryRepository;
    }
}
```

**Tại sao Constructor Injection là best practice?**
- Fields `final` → immutable, thread-safe
- Dependencies rõ ràng → dễ test: `new ProductService(mockRepo, mockCatRepo)`
- Quên inject → lỗi **compile-time** (không phải runtime)

#### Setter Injection (Optional dependency)

```java
@Service
public class NotificationService {
    private EmailService emailService;

    @Autowired
    public void setEmailService(EmailService emailService) {
        this.emailService = emailService;
    }
}
```

#### Field Injection (Tránh dùng)

```java
@Service
public class ProductService {
    @Autowired  // ❌ KHÔNG khuyến nghị
    private ProductRepository productRepository;
}
```

**Tại sao tránh?** Không `final` được, khó unit test (cần reflection), dependencies ẩn, dễ inject quá nhiều mà không nhận ra.

### 1.3 Stereotype Annotations

| Annotation | Layer | Mục đích | Đặc biệt |
|-----------|-------|----------|-----------|
| `@Component` | General | Bean chung | Base annotation |
| `@Service` | Service | Business logic | Semantic — giống `@Component` |
| `@Repository` | Data Access | Truy vấn DB | Auto translate SQL exception |
| `@Controller` | Web | HTTP request | Kết hợp `@RequestMapping` |

> Tất cả đều **extend** từ `@Component`. Khác biệt chính là **semantic** và xử lý đặc biệt (exception translation ở `@Repository`).

### 1.4 @Bean trong @Configuration

Dùng khi tạo Bean từ class **bên ngoài** (library) mà không thể thêm `@Component`:

```java
@Configuration
public class AppConfig {
    @Bean
    public ObjectMapper objectMapper() {
        return new ObjectMapper()
                .registerModule(new JavaTimeModule())
                .disable(SerializationFeature.WRITE_DATES_AS_TIMESTAMPS);
    }
}
```

### 1.5 Khi nào dùng gì?

```
Class do bạn viết?
├── YES → @Component / @Service / @Repository / @Controller (tuỳ layer)
└── NO  → Class từ library → @Bean method trong @Configuration
```

---

## 2. Bean Lifecycle

```
┌───────────────────────────────────────────────────────┐
│                    BEAN LIFECYCLE                      │
├───────────────────────────────────────────────────────┤
│  1. Instantiation        → Spring gọi constructor     │
│           │                                           │
│           ▼                                           │
│  2. Populate Properties  → Inject dependencies        │
│           │                                           │
│           ▼                                           │
│  3. BeanPostProcessor    → AOP proxy, validation...   │
│           │                                           │
│           ▼                                           │
│  4. @PostConstruct       → Custom init logic          │
│           │                                           │
│           ▼                                           │
│  5. Ready                → Bean sẵn sàng sử dụng     │
│           │                                           │
│      (App running...)                                 │
│           │                                           │
│           ▼                                           │
│  6. @PreDestroy          → Cleanup khi shutdown       │
└───────────────────────────────────────────────────────┘
```

### @PostConstruct — Use Case

```java
@Service
public class CacheWarmupService {
    private final ProductRepository productRepository;

    public CacheWarmupService(ProductRepository productRepository) {
        this.productRepository = productRepository;
    }

    @PostConstruct  // Gọi tự động SAU khi inject xong
    public void warmupCache() {
        List<Product> featured = productRepository.findByFeaturedTrue();
        // load vào cache...
        System.out.println("Cache warmed up: " + featured.size() + " products");
    }
}
```

> Không nên đặt logic nặng trong `@PostConstruct` — nó chạy khi startup, làm chậm boot time.

---

## 3. @ConfigurationProperties — Type-Safe Configuration

### 3.1 Vấn đề với @Value

```java
@Value("${app.jwt.secert}")  // ❌ typo "secert" → null, lỗi runtime
private String jwtSecret;
```

### 3.2 Solution: @ConfigurationProperties

```java
@ConfigurationProperties(prefix = "app")
@Validated
@Getter @Setter
public class AppProperties {

    @NotBlank private String name;
    @NotBlank private String version;
    @Valid private Jwt jwt = new Jwt();
    @Valid private Security security = new Security();

    @Getter @Setter
    public static class Jwt {
        @NotBlank private String secret;
        private long accessExpiration = 86400000;   // 24h
        private long refreshExpiration = 604800000; // 7 ngày
        private String issuer = "ecommerce-api";
    }

    @Getter @Setter
    public static class Security {
        private int maxLoginAttempts = 5;
        private long lockDuration = 900000;  // 15 phút
        private String[] allowedOrigins = {"http://localhost:3000"};
    }
}
```

### 3.3 Enable và sử dụng

```java
@SpringBootApplication
@EnableConfigurationProperties(AppProperties.class)
public class EcommerceApplication { ... }
```

### 3.4 application.yml tương ứng

```yaml
app:
  name: ecommerce-api
  version: 1.0.0
  jwt:
    secret: ${JWT_SECRET}
    access-expiration: 86400000     # kebab-case → camelCase tự động
    refresh-expiration: 604800000
  security:
    max-login-attempts: 5
    lock-duration: 900000
    allowed-origins:
      - http://localhost:3000
      - http://localhost:4200
```

### 3.5 Inject trong Service

```java
@Service
public class AuthService {
    private final AppProperties appProperties;

    public AuthService(AppProperties appProperties) {
        this.appProperties = appProperties;
    }

    public String generateToken(User user) {
        long expiration = appProperties.getJwt().getAccessExpiration();
        String secret = appProperties.getJwt().getSecret();
        // ... tạo JWT token
    }
}
```

> Typo trong YAML key → app **fail fast** khi startup (nhờ `@Validated`).

---

## 4. Secrets Management

### .env file pattern

**Bước 1 — Tạo `.env`:**

```bash
# .env — KHÔNG commit file này
JWT_SECRET=my-super-secret-key-at-least-32-characters-long
DB_HOST=localhost
DB_PORT=3306
DB_NAME=ecommerce
DB_USERNAME=root
DB_PASSWORD=1q2w3E*
REDIS_HOST=localhost
REDIS_PORT=6379
```

**Bước 2 — Spring Boot đọc `.env`:**

```yaml
spring:
  config:
    import: optional:file:.env[.properties]
```

> `optional:` — nếu `.env` không tồn tại, app vẫn chạy (dùng env vars từ system).

**Bước 3 — `.gitignore`:**

```bash
.env
.env.local
.env.*.local
```

**Bước 4 — Tham chiếu:**

```yaml
spring:
  datasource:
    url: jdbc:mysql://${DB_HOST}:${DB_PORT}/${DB_NAME}
    username: ${DB_USERNAME}
    password: ${DB_PASSWORD}
```

> **Tip:** Tạo `.env.example` (không chứa giá trị thật) và commit — để team biết cần set biến nào.

---

## 5. JPA Stack Overview

```
┌─────────────────────────────────────────┐
│  Your Code:  productRepo.save(p)        │  ← Bạn chỉ viết ở tầng này
├─────────────────────────────────────────┤
│  Spring Data JPA                        │  ← Auto-generate implementation
│  (JpaRepository, Pageable, Auditing)    │     từ interface methods
├─────────────────────────────────────────┤
│  JPA (Jakarta Persistence API)          │  ← Specification (chỉ interface)
│  (@Entity, @Table, EntityManager, JPQL) │     Defines API contract
├─────────────────────────────────────────┤
│  Hibernate 6 (Implementation)           │  ← Caching, Dirty Checking,
│  (Session, L1/L2 Cache, Lazy Loading)   │     Lazy Loading
├─────────────────────────────────────────┤
│  JDBC                                   │  ← Connection, PreparedStatement
├─────────────────────────────────────────┤
│  MySQL Driver → MySQL Server            │  ← Lưu trữ data
└─────────────────────────────────────────┘
```

**Tại sao Spring Data JPA?** Không cần viết implementation, derived queries từ tên method, pagination built-in, auditing tự động, Specification cho dynamic queries. Loại bỏ 80-90% boilerplate so với raw Hibernate.

---

## 6. HikariCP — Connection Pool

Mở connection tới DB rất tốn kém (TCP handshake, auth). HikariCP giữ sẵn pool connections đã mở, tái sử dụng.

```yaml
spring:
  datasource:
    url: jdbc:mysql://${DB_HOST}:${DB_PORT}/${DB_NAME}?useSSL=false&serverTimezone=UTC&allowPublicKeyRetrieval=true
    username: ${DB_USERNAME}
    password: ${DB_PASSWORD}
    driver-class-name: com.mysql.cj.jdbc.Driver
    hikari:
      maximum-pool-size: 10       # Max connections
      minimum-idle: 5             # Min idle connections sẵn sàng
      connection-timeout: 30000   # 30s — timeout chờ lấy connection
      idle-timeout: 600000        # 10 phút — đóng connection idle
      max-lifetime: 1800000       # 30 phút — max tuổi thọ 1 connection
      pool-name: EcommercePool
```

| Setting | Default | Gợi ý |
|---------|---------|-------|
| `maximum-pool-size` | 10 | `(core_count * 2) + disk_spindles` |
| `minimum-idle` | = max | Set bằng max cho ổn định |
| `connection-timeout` | 30000 | Quá → `SQLException` |
| `max-lifetime` | 1800000 | Nên < MySQL `wait_timeout` |

---

## 7. BaseEntity — Entity cha dùng chung

```java
@MappedSuperclass                                // Không tạo table, chỉ kế thừa fields
@EntityListeners(AuditingEntityListener.class)    // Auto-fill audit fields
@Getter @Setter
public abstract class BaseEntity {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @CreatedDate
    @Column(name = "created_at", nullable = false, updatable = false)
    private Instant createdAt;

    @LastModifiedDate
    @Column(name = "updated_at")
    private Instant updatedAt;
}
```

**Enable JPA Auditing:**

```java
@SpringBootApplication
@EnableJpaAuditing
@EnableConfigurationProperties(AppProperties.class)
public class EcommerceApplication { ... }
```

> Dùng `Instant` thay vì `LocalDateTime` — luôn UTC, tránh lỗi timezone khi deploy.

---

## 8. Entity: Category

Category hỗ trợ **cấu trúc cây** (parent-child): `Electronics → Phones → Smartphones`.

```java
@Entity
@Table(name = "categories")
@Getter @Setter @NoArgsConstructor
public class Category extends BaseEntity {

    @Column(nullable = false, length = 100)
    private String name;

    @Column(nullable = false, unique = true, length = 120)
    private String slug;

    @Column(columnDefinition = "TEXT")
    private String description;

    @Column(name = "image_url")
    private String imageUrl;

    private boolean active = true;

    @Column(name = "sort_order")
    private int sortOrder = 0;

    // Self-referencing: cây danh mục
    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "parent_id")
    private Category parent;

    @OneToMany(mappedBy = "parent")
    private List<Category> children = new ArrayList<>();

    @OneToMany(mappedBy = "category")
    private List<Product> products = new ArrayList<>();
}
```

**Annotation cheatsheet:**

| Annotation | Mô tả |
|-----------|-------|
| `@Entity` | JPA entity → Hibernate quản lý |
| `@Table(name)` | Tên table. Không set → dùng tên class |
| `@Column(...)` | Tuỳ chỉnh: nullable, unique, length, columnDefinition |
| `@ManyToOne(LAZY)` | Nhiều-một. LAZY = chỉ load khi gọi getter |
| `@OneToMany(mappedBy)` | Một-nhiều. `mappedBy` = field ở owning side |
| `@JoinColumn` | FK column. Đặt ở **owning side** (phía có FK) |

> **Luôn dùng `FetchType.LAZY`** cho `@ManyToOne` và `@OneToMany` để tránh N+1 problem.

---

## 9. Entity: Product

```java
@Entity
@Table(name = "products", indexes = {
    @Index(name = "idx_product_slug", columnList = "slug", unique = true),
    @Index(name = "idx_product_category", columnList = "category_id")
})
@Getter @Setter @NoArgsConstructor
public class Product extends BaseEntity {

    @Column(nullable = false, length = 200)
    private String name;

    @Column(nullable = false, unique = true, length = 250)
    private String slug;

    @Column(columnDefinition = "TEXT")
    private String description;

    @Column(nullable = false, precision = 12, scale = 2)    // DECIMAL(12,2)
    private BigDecimal price;

    @Column(name = "original_price", precision = 12, scale = 2)
    private BigDecimal originalPrice;

    @Column(name = "stock_quantity", nullable = false)
    private int stockQuantity = 0;

    private boolean active = true;
    private boolean featured = false;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "category_id", nullable = false)
    private Category category;

    @OneToMany(mappedBy = "product", cascade = CascadeType.ALL, orphanRemoval = true)
    private List<ProductImage> images = new ArrayList<>();

    // Helper: đồng bộ bidirectional relationship
    public void addImage(ProductImage image) {
        images.add(image);
        image.setProduct(this);
    }

    public void removeImage(ProductImage image) {
        images.remove(image);
        image.setProduct(null);
    }
}
```

| Khái niệm | Giải thích |
|-----------|-----------|
| `@Index` | Database index → tăng tốc query trên slug, category_id |
| `precision=12, scale=2` | DECIMAL(12,2). **Luôn dùng BigDecimal cho tiền** |
| `cascade = ALL` | Save/delete Product → tự động save/delete images |
| `orphanRemoval` | Remove image khỏi list → DELETE record trong DB |

> **BigDecimal cho tiền:** `double 0.1 + 0.2 = 0.30000000000000004`. BigDecimal: `0.3` chính xác.

---

## 10. Entity: ProductImage

```java
@Entity
@Table(name = "product_images")
@Getter @Setter @NoArgsConstructor
public class ProductImage extends BaseEntity {

    @Column(name = "image_url", nullable = false)
    private String imageUrl;

    @Column(name = "is_primary")
    private boolean primary = false;

    @Column(name = "sort_order")
    private int sortOrder = 0;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "product_id", nullable = false)
    private Product product;
}
```

> `ProductImage` là **owning side** (có `@JoinColumn`). Hibernate chỉ track thay đổi từ owning side — luôn dùng `product.addImage()`.

---

## 11. Repositories

### CategoryRepository

```java
@Repository
public interface CategoryRepository extends JpaRepository<Category, Long> {

    Optional<Category> findBySlug(String slug);           // WHERE slug = ?
    List<Category> findByActiveTrue();                    // WHERE active = true
    List<Category> findByParentIsNull();                  // WHERE parent_id IS NULL
    boolean existsBySlug(String slug);                    // COUNT(*) > 0
    List<Category> findByActiveTrueOrderBySortOrderAsc(); // active + ORDER BY
}
```

### ProductRepository

```java
@Repository
public interface ProductRepository extends JpaRepository<Product, Long>,
        JpaSpecificationExecutor<Product> {

    Optional<Product> findBySlug(String slug);
    List<Product> findByCategoryId(Long categoryId);
    List<Product> findByFeaturedTrue();

    @Query("SELECT p FROM Product p WHERE p.price BETWEEN :min AND :max AND p.active = true")
    List<Product> findByPriceRange(@Param("min") BigDecimal min, @Param("max") BigDecimal max);

    long countByCategoryIdAndActiveTrue(Long categoryId);
}
```

> `JpaSpecificationExecutor` — dynamic queries dùng `Specification`. Dùng cho filter/search nhiều điều kiện (Day 4).

### Derived Query — Naming Convention

| Keyword | Ví dụ | SQL tương đương |
|---------|-------|----------------|
| `findBy` | `findByName(String)` | `WHERE name = ?` |
| `And` | `findByNameAndActive(...)` | `WHERE name = ? AND active = ?` |
| `Or` | `findByNameOrSlug(...)` | `WHERE name = ? OR slug = ?` |
| `IsNull` | `findByParentIsNull()` | `WHERE parent_id IS NULL` |
| `True/False` | `findByActiveTrue()` | `WHERE active = true` |
| `Between` | `findByPriceBetween(a, b)` | `WHERE price BETWEEN ? AND ?` |
| `LessThan` | `findByPriceLessThan(p)` | `WHERE price < ?` |
| `Containing` | `findByNameContaining(s)` | `WHERE name LIKE %?%` |
| `OrderBy` | `findByActiveTrueOrderByNameAsc()` | `... ORDER BY name ASC` |
| `Top/First` | `findTop5ByOrderByPriceDesc()` | `... LIMIT 5` |
| `countBy` | `countByActiveTrue()` | `SELECT COUNT(*)` |
| `existsBy` | `existsBySlug(slug)` | `SELECT COUNT(*) > 0` |

> **Rule:** Tên method 4+ conditions → chuyển sang `@Query` JPQL hoặc `Specification`.

---

## 12. JPA Configuration

```yaml
spring:
  jpa:
    hibernate:
      ddl-auto: update              # Tự tạo/update table (dev only!)
    show-sql: true
    properties:
      hibernate:
        format_sql: true
        dialect: org.hibernate.dialect.MySQLDialect
    open-in-view: false              # Tắt OSIV — best practice
```

**`ddl-auto` options:**

| Giá trị | Mô tả | Dùng khi |
|---------|-------|---------|
| `none` | Không làm gì | Production |
| `validate` | Check schema khớp entity | Staging |
| `update` | Tạo/thêm column, KHÔNG xoá | Development |
| `create` | DROP + CREATE mỗi lần start | Testing |
| `create-drop` | CREATE lúc start, DROP lúc stop | Unit test |

> **`open-in-view: false`** — Mặc định Spring bật true → giữ Hibernate session mở tới Controller → N+1 risk. **Luôn tắt**, xử lý lazy loading trong Service layer.

---

## 13. Verify

### Chạy ứng dụng

```bash
docker compose up -d mysql
./mvnw spring-boot:run -Dspring-boot.run.profiles=dev
```

### Kiểm tra MySQL

```bash
docker exec -it ecommerce-mysql mysql -uroot -p1q2w3E* ecommerce
```

```sql
SHOW TABLES;
-- Expected: categories, products, product_images

DESCRIBE products;
DESCRIBE categories;
```

### Kiểm tra Hibernate SQL logs

```sql
Hibernate:
    create table categories (
        id bigint not null auto_increment,
        active bit not null,
        created_at datetime(6) not null,
        name varchar(100) not null,
        slug varchar(120) not null,
        parent_id bigint,
        primary key (id)
    ) engine=InnoDB
```

**Checklist:**
- [ ] App start thành công
- [ ] 3 tables: `categories`, `products`, `product_images`
- [ ] FK: `products.category_id` → `categories.id`
- [ ] FK: `product_images.product_id` → `products.id`
- [ ] Self-ref FK: `categories.parent_id` → `categories.id`
- [ ] Indexes: `idx_product_slug`, `idx_product_category`
- [ ] Audit columns: `created_at`, `updated_at` trong tất cả tables

---

## Tóm tắt Day 2

```
IoC/DI          → Constructor injection, @Service/@Repository/@Component
Bean Lifecycle  → @PostConstruct, vòng đời 5 giai đoạn
Configuration   → @ConfigurationProperties type-safe, .env secrets
JPA Stack       → Spring Data JPA → JPA → Hibernate → JDBC → MySQL
HikariCP        → Connection pooling config
Entities        → Category (self-ref tree), Product, ProductImage
Repositories    → JpaRepository, derived queries, @Query JPQL
```

> **Ngày tiếp theo** → [Day 3: JPA Mastery](./day-03-jpa-mastery.md) — Flyway migrations, Specification, Pagination, N+1 problem, query optimization.
