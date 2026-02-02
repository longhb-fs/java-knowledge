# Ngày 10: JPA Performance Optimization

## Mục tiêu hôm nay
- Phát hiện và xử lý N+1 Problem
- Hiểu và sử dụng @EntityGraph
- Batch Fetching configuration
- Connection Pool tuning
- Query optimization techniques

---

## 1. N+1 PROBLEM

### 1.1. N+1 là gì?

```java
// Entity
@Entity
public class Product {
    @ManyToOne(fetch = FetchType.LAZY)  // LAZY mặc định cho ManyToOne
    private Category category;
}

// ❌ N+1 PROBLEM
List<Product> products = productRepository.findAll();  // 1 query
for (Product p : products) {
    System.out.println(p.getCategory().getName());     // N queries!
}

// Kết quả:
// Query 1: SELECT * FROM products                     (1 query)
// Query 2: SELECT * FROM categories WHERE id = 1     (N queries)
// Query 3: SELECT * FROM categories WHERE id = 2
// Query 4: SELECT * FROM categories WHERE id = 3
// ...
// TỔNG: 1 + N queries (nếu 100 products = 101 queries!)
```

```
VISUALIZE N+1:

Request ──> findAll() ──> 1 Query ──> 100 Products
                │
                └──> Loop through products
                      │
                      ├──> product[0].getCategory() ──> Query #2
                      ├──> product[1].getCategory() ──> Query #3
                      ├──> product[2].getCategory() ──> Query #4
                      │    ...
                      └──> product[99].getCategory() ──> Query #101

                      TOTAL: 101 Queries! 🔥
```

### 1.2. Phát hiện N+1

```yaml
# application.yml - Bật SQL logging
spring:
  jpa:
    show-sql: true
    properties:
      hibernate:
        format_sql: true
        # Thống kê queries
        generate_statistics: true
        session:
          events:
            log:
              LOG_QUERIES_SLOWER_THAN_MS: 25  # Log query > 25ms

logging:
  level:
    org.hibernate.SQL: DEBUG
    org.hibernate.type.descriptor.sql.BasicBinder: TRACE  # Log params
    org.hibernate.stat: DEBUG  # Statistics
```

**Output log khi có N+1:**
```
Hibernate: select p.* from products p
Hibernate: select c.* from categories c where c.id=?
Hibernate: select c.* from categories c where c.id=?
Hibernate: select c.* from categories c where c.id=?
... (lặp lại nhiều lần)
```

---

## 2. GIẢI PHÁP N+1

### 2.1. JOIN FETCH (JPQL)

```java
@Repository
public interface ProductRepository extends JpaRepository<Product, Long> {

    // ✅ GIẢI PHÁP 1: JOIN FETCH
    @Query("SELECT p FROM Product p JOIN FETCH p.category")
    List<Product> findAllWithCategory();

    // Multiple JOIN FETCH
    @Query("""
        SELECT DISTINCT p FROM Product p
        LEFT JOIN FETCH p.category
        LEFT JOIN FETCH p.images
        WHERE p.isActive = true
        """)
    List<Product> findAllWithCategoryAndImages();

    // JOIN FETCH với Pagination
    // ⚠️ Cần 2 queries: 1 cho IDs, 1 cho data
    @Query(value = "SELECT p.id FROM Product p WHERE p.isActive = true")
    Page<Long> findActiveProductIds(Pageable pageable);

    @Query("SELECT p FROM Product p JOIN FETCH p.category WHERE p.id IN :ids")
    List<Product> findAllWithCategoryByIds(@Param("ids") List<Long> ids);
}

// Sử dụng với pagination
public Page<Product> findActiveWithCategory(Pageable pageable) {
    Page<Long> idsPage = productRepository.findActiveProductIds(pageable);
    List<Product> products = productRepository.findAllWithCategoryByIds(idsPage.getContent());
    return new PageImpl<>(products, pageable, idsPage.getTotalElements());
}
```

### 2.2. @EntityGraph

```java
@Repository
public interface ProductRepository extends JpaRepository<Product, Long> {

    // ✅ GIẢI PHÁP 2: @EntityGraph (annotation-based)

    // Fetch category cùng lúc
    @EntityGraph(attributePaths = {"category"})
    List<Product> findByIsActiveTrue();

    // Fetch nhiều associations
    @EntityGraph(attributePaths = {"category", "images", "tags"})
    Optional<Product> findWithDetailsById(Long id);

    // Kết hợp với @Query
    @EntityGraph(attributePaths = {"category"})
    @Query("SELECT p FROM Product p WHERE p.price < :maxPrice")
    List<Product> findCheapProductsWithCategory(@Param("maxPrice") BigDecimal maxPrice);

    // Named EntityGraph (định nghĩa trên Entity)
    @EntityGraph(value = "Product.withCategoryAndImages")
    List<Product> findAll();
}
```

**Định nghĩa Named EntityGraph trên Entity:**
```java
@Entity
@NamedEntityGraph(
    name = "Product.withCategoryAndImages",
    attributeNodes = {
        @NamedAttributeNode("category"),
        @NamedAttributeNode("images")
    }
)
@NamedEntityGraph(
    name = "Product.full",
    attributeNodes = {
        @NamedAttributeNode("category"),
        @NamedAttributeNode("images"),
        @NamedAttributeNode(value = "tags", subgraph = "tags-subgraph")
    },
    subgraphs = {
        @NamedSubgraph(name = "tags-subgraph",
            attributeNodes = @NamedAttributeNode("name"))
    }
)
public class Product {
    // ...
}
```

### 2.3. Batch Fetching (Hibernate)

```yaml
# application.yml
spring:
  jpa:
    properties:
      hibernate:
        default_batch_fetch_size: 50  # Fetch 50 entities per batch
```

```
WITHOUT Batch Fetching (N+1):
Query 1: SELECT * FROM products
Query 2: SELECT * FROM categories WHERE id = 1
Query 3: SELECT * FROM categories WHERE id = 2
...
Query 101: SELECT * FROM categories WHERE id = 100

WITH Batch Fetching (size=50):
Query 1: SELECT * FROM products
Query 2: SELECT * FROM categories WHERE id IN (1,2,3,...50)
Query 3: SELECT * FROM categories WHERE id IN (51,52,...100)

RESULT: 101 queries → 3 queries!
```

**Hoặc annotate trên Entity:**
```java
@Entity
@BatchSize(size = 50)  // Batch fetch this entity
public class Category {
    // ...
}

@Entity
public class Product {
    @OneToMany(mappedBy = "product")
    @BatchSize(size = 50)  // Batch fetch this collection
    private List<ProductImage> images;
}
```

### 2.4. So sánh các giải pháp

| Giải pháp | Ưu điểm | Nhược điểm |
|-----------|---------|------------|
| **JOIN FETCH** | 1 query, explicit | Không dùng với pagination trực tiếp |
| **@EntityGraph** | Clean, declarative | Có thể over-fetch |
| **Batch Fetching** | Global, transparent | Vẫn nhiều queries (tốt hơn N+1) |
| **DTO Projection** | Chỉ lấy cần | Không có entity, complex mapping |

---

## 3. LAZY VS EAGER LOADING

### 3.1. Best Practices

```java
// ❌ EAGER mặc định cho ToOne
@ManyToOne  // FetchType.EAGER by default!
private Category category;

// ✅ LUÔN dùng LAZY
@ManyToOne(fetch = FetchType.LAZY)
private Category category;

@OneToOne(fetch = FetchType.LAZY)  // OneToOne cũng LAZY!
private UserProfile profile;

@OneToMany(mappedBy = "product", fetch = FetchType.LAZY)  // OK, default LAZY
private List<ProductImage> images;
```

### 3.2. LazyInitializationException

```java
// ❌ Lỗi: Session đã đóng khi access lazy field
@Service
@Transactional(readOnly = true)
public class ProductService {
    public Product getProduct(Long id) {
        return productRepository.findById(id).orElseThrow();
    }
}

@RestController
public class ProductController {
    @GetMapping("/{id}")
    public ProductResponse getProduct(@PathVariable Long id) {
        Product product = productService.getProduct(id);
        // ❌ LazyInitializationException!
        // Session đã đóng sau khi Service method kết thúc
        return new ProductResponse(product.getName(), product.getCategory().getName());
    }
}

// ✅ GIẢI PHÁP 1: Fetch trong transaction
@Service
@Transactional(readOnly = true)
public class ProductService {
    public ProductResponse getProduct(Long id) {
        Product product = productRepository.findWithCategoryById(id).orElseThrow();
        return new ProductResponse(product.getName(), product.getCategory().getName());
    }
}

// ✅ GIẢI PHÁP 2: Open Session in View (KHÔNG khuyến khích cho API)
spring.jpa.open-in-view: false  # DISABLE này!

// ✅ GIẢI PHÁP 3: DTO Projection
@Query("""
    SELECT new ProductDTO(p.id, p.name, c.name)
    FROM Product p LEFT JOIN p.category c
    WHERE p.id = :id
    """)
Optional<ProductDTO> findProductDTOById(@Param("id") Long id);
```

---

## 4. QUERY OPTIMIZATION

### 4.1. Select chỉ columns cần thiết

```java
// ❌ Lấy toàn bộ entity (nhiều columns không dùng)
List<Product> products = productRepository.findAll();

// ✅ Projection - chỉ lấy columns cần
public interface ProductSummary {
    Long getId();
    String getName();
    BigDecimal getPrice();
}

List<ProductSummary> summaries = productRepository.findByIsActiveTrue();

// ✅ DTO Projection
@Query("""
    SELECT new ProductListDTO(p.id, p.name, p.price, p.mainImage, c.name)
    FROM Product p LEFT JOIN p.category c
    WHERE p.isActive = true
    """)
Page<ProductListDTO> findProductList(Pageable pageable);
```

### 4.2. Avoid SELECT COUNT(*) when not needed

```java
// ❌ Page luôn chạy COUNT query
Page<Product> page = productRepository.findAll(pageable);
// Query 1: SELECT * FROM products LIMIT 10
// Query 2: SELECT COUNT(*) FROM products  (có thể chậm với large table!)

// ✅ Slice không chạy COUNT
Slice<Product> slice = productRepository.findSliceByIsActiveTrue(pageable);
// Query: SELECT * FROM products LIMIT 11  (lấy thêm 1 để check hasNext)

// ✅ Estimate count cho large tables
@Query(value = """
    SELECT TABLE_ROWS FROM information_schema.TABLES
    WHERE TABLE_SCHEMA = DATABASE() AND TABLE_NAME = 'products'
    """, nativeQuery = true)
long estimateProductCount();
```

### 4.3. Index Optimization

```sql
-- Flyway migration: V10__add_performance_indexes.sql

-- Composite index cho common queries
CREATE INDEX idx_product_category_active_price
ON products (category_id, is_active, price);

-- Covering index (include all select columns)
CREATE INDEX idx_product_listing
ON products (is_active, created_at, id, name, price, main_image);

-- Full-text search index
ALTER TABLE products ADD FULLTEXT INDEX ft_product_search (name, description);
```

### 4.4. Pagination Optimization (Keyset/Cursor)

```java
// ❌ Offset pagination chậm với large offset
// SELECT * FROM products LIMIT 10 OFFSET 100000
// MySQL phải scan 100000 rows rồi skip!

// ✅ Keyset pagination (cursor-based)
@Query("""
    SELECT p FROM Product p
    WHERE p.isActive = true
    AND (p.createdAt, p.id) < (:lastCreatedAt, :lastId)
    ORDER BY p.createdAt DESC, p.id DESC
    """)
List<Product> findNextPage(
    @Param("lastCreatedAt") LocalDateTime lastCreatedAt,
    @Param("lastId") Long lastId,
    Pageable pageable
);

// Client gửi cursor của item cuối cùng
// Không cần OFFSET, query luôn nhanh!
```

---

## 5. CONNECTION POOL TUNING

### 5.1. HikariCP Configuration

```yaml
spring:
  datasource:
    hikari:
      # ═══ Pool sizing ═══
      maximum-pool-size: 20          # Max connections
      minimum-idle: 5                 # Min idle connections

      # ═══ Timeout settings ═══
      connection-timeout: 30000       # 30s - wait for connection from pool
      idle-timeout: 600000            # 10min - close idle connections
      max-lifetime: 1800000           # 30min - max connection age

      # ═══ Leak detection ═══
      leak-detection-threshold: 60000 # 60s - log warning if connection held too long

      # ═══ Validation ═══
      connection-test-query: SELECT 1
      validation-timeout: 5000        # 5s - timeout for connection validation

      # ═══ Pool name ═══
      pool-name: EcommerceHikariPool
```

### 5.2. Sizing the Pool

```
Formula cơ bản:
pool size = (core_count * 2) + effective_spindle_count

Ví dụ:
- 4 CPU cores
- SSD (effective_spindle_count ≈ 1)
- pool size = (4 * 2) + 1 = 9-10 connections

Quy tắc:
- Bắt đầu nhỏ (10-20), monitor, tăng dần
- Quá nhiều connections = context switching overhead
- Quá ít = connection starvation
```

### 5.3. Monitoring Pool

```yaml
# Actuator endpoint
management:
  endpoints:
    web:
      exposure:
        include: health, metrics, hikaricp
```

```bash
# Check pool metrics
curl http://localhost:8080/actuator/metrics/hikaricp.connections
curl http://localhost:8080/actuator/metrics/hikaricp.connections.active
curl http://localhost:8080/actuator/metrics/hikaricp.connections.idle
curl http://localhost:8080/actuator/metrics/hikaricp.connections.pending
```

---

## 6. HIBERNATE CACHING

### 6.1. First-Level Cache (Session Cache)

```java
// First-level cache = trong 1 Session/Transaction
// Tự động, không cần config

@Transactional
public void example() {
    Product p1 = productRepository.findById(1L);  // Query DB
    Product p2 = productRepository.findById(1L);  // Từ cache, KHÔNG query

    System.out.println(p1 == p2);  // true (same object)
}

// Khi transaction kết thúc → cache bị clear
```

### 6.2. Second-Level Cache (SessionFactory Cache)

```xml
<!-- pom.xml -->
<dependency>
    <groupId>org.hibernate.orm</groupId>
    <artifactId>hibernate-jcache</artifactId>
</dependency>
<dependency>
    <groupId>org.ehcache</groupId>
    <artifactId>ehcache</artifactId>
</dependency>
```

```yaml
# application.yml
spring:
  jpa:
    properties:
      hibernate:
        cache:
          use_second_level_cache: true
          use_query_cache: true
          region:
            factory_class: org.hibernate.cache.jcache.JCacheRegionFactory
        javax:
          cache:
            provider: org.ehcache.jsr107.EhcacheCachingProvider
```

```java
// Entity phải đánh dấu @Cacheable
@Entity
@Cacheable
@org.hibernate.annotations.Cache(usage = CacheConcurrencyStrategy.READ_WRITE)
public class Category {
    // Category ít thay đổi → cache hiệu quả
}

// Query cache
@QueryHints(@QueryHint(name = "org.hibernate.cacheable", value = "true"))
@Query("SELECT c FROM Category c WHERE c.isActive = true")
List<Category> findAllActive();
```

---

## 7. CHECKLIST PERFORMANCE

### 7.1. Quick Wins

- [ ] Set `FetchType.LAZY` cho tất cả `@ManyToOne`, `@OneToOne`
- [ ] Bật `hibernate.default_batch_fetch_size: 50`
- [ ] Disable `spring.jpa.open-in-view: false`
- [ ] Dùng Projection/DTO thay vì Entity cho read-only
- [ ] Index các columns hay được filter/sort
- [ ] Monitor với `hibernate.generate_statistics: true`

### 7.2. Performance Checklist

| Check | Status |
|-------|--------|
| Không có N+1 queries | ☐ |
| Tất cả ToOne là LAZY | ☐ |
| Batch fetching enabled | ☐ |
| open-in-view disabled | ☐ |
| DTO projection cho listings | ☐ |
| Index cho common queries | ☐ |
| Connection pool sized | ☐ |
| Query cache cho static data | ☐ |
| Slow query logging enabled | ☐ |

---

## 8. BÀI TẬP & CHECKLIST

### Bài tập thực hành

- [ ] Enable SQL logging, phát hiện N+1
- [ ] Fix N+1 bằng JOIN FETCH
- [ ] Tạo @EntityGraph cho Product
- [ ] Configure batch fetching
- [ ] Tạo DTO Projection cho product listing
- [ ] Configure HikariCP pool
- [ ] Enable Hibernate statistics
- [ ] Add indexes cho common queries
- [ ] Measure query time before/after optimization

### Kiến thức cần nắm

| Khái niệm | Hiểu chưa? |
|-----------|-------------|
| N+1 Problem là gì | ☐ |
| JOIN FETCH trong JPQL | ☐ |
| @EntityGraph annotation | ☐ |
| Batch Fetching | ☐ |
| LAZY vs EAGER loading | ☐ |
| LazyInitializationException | ☐ |
| DTO Projection benefits | ☐ |
| HikariCP configuration | ☐ |
| First vs Second level cache | ☐ |
| Query optimization techniques | ☐ |

---

**Trước đó:** [← Ngày 09 - Pagination & Specification](day-09-pagination-specification.md)

**Tiếp theo:** [Ngày 11 - REST Controller Basics →](../week-03/day-11-rest-controller.md)
