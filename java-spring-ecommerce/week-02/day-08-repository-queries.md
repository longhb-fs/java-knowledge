# Ngày 08: Repository & Custom Queries

## Mục tiêu hôm nay
- Hiểu sâu Spring Data JPA Repository hierarchy
- Thành thạo Derived Query Methods (query từ method name)
- Viết JPQL và Native SQL với @Query
- Sử dụng Named Parameters và Projections
- Modifying queries (UPDATE, DELETE)

---

## 1. SPRING DATA JPA REPOSITORY HIERARCHY

### 1.1. Repository interface tree

```
                    Repository<T, ID>
                         │
                         │ (marker interface)
                         │
              CrudRepository<T, ID>
                         │
                         │ + save, findById, delete, count, existsById
                         │
           ListCrudRepository<T, ID>
                         │
                         │ + findAll() returns List instead of Iterable
                         │
         PagingAndSortingRepository<T, ID>
                         │
                         │ + findAll(Sort), findAll(Pageable)
                         │
             JpaRepository<T, ID>
                         │
                         │ + flush, saveAndFlush, deleteAllInBatch
                         │ + findAll returns List
                         │ + JPA-specific methods
                         │
                         ├──────────────────────────┐
                         │                          │
      JpaSpecificationExecutor<T>          QueryByExampleExecutor<T>
                         │                          │
      + findAll(Specification)             + findOne(Example)
      + count(Specification)               + findAll(Example)
      + Dynamic queries!                   + Query by Example
```

### 1.2. Các method có sẵn của JpaRepository

```java
public interface ProductRepository extends JpaRepository<Product, Long> {
    // ═══ Inherited from CrudRepository ═══
    // <S extends T> S save(S entity);
    // <S extends T> List<S> saveAll(Iterable<S> entities);
    // Optional<T> findById(ID id);
    // boolean existsById(ID id);
    // List<T> findAll();
    // List<T> findAllById(Iterable<ID> ids);
    // long count();
    // void deleteById(ID id);
    // void delete(T entity);
    // void deleteAllById(Iterable<? extends ID> ids);
    // void deleteAll(Iterable<? extends T> entities);
    // void deleteAll();

    // ═══ Inherited from PagingAndSortingRepository ═══
    // List<T> findAll(Sort sort);
    // Page<T> findAll(Pageable pageable);

    // ═══ JPA-specific ═══
    // void flush();
    // <S extends T> S saveAndFlush(S entity);
    // <S extends T> List<S> saveAllAndFlush(Iterable<S> entities);
    // void deleteAllInBatch(Iterable<T> entities);
    // void deleteAllByIdInBatch(Iterable<ID> ids);
    // void deleteAllInBatch();
    // T getReferenceById(ID id);  // Lazy proxy
}
```

---

## 2. DERIVED QUERY METHODS

### 2.1. Cách Spring Data sinh query từ method name

```java
// Method name = Subject + Predicate
// findBy + PropertyName + Operator

findByName(String name)
│    │    └── Property trong Entity
│    └── Keyword bắt đầu (By)
└── Subject (find, read, get, query, stream, count, exists, delete)
```

### 2.2. Bảng Keywords

| Keyword | SQL | Ví dụ |
|---------|-----|-------|
| `And` | AND | `findByNameAndPrice` |
| `Or` | OR | `findByNameOrDescription` |
| `Is`, `Equals` | = | `findByName`, `findByNameIs`, `findByNameEquals` |
| `Between` | BETWEEN | `findByPriceBetween(min, max)` |
| `LessThan` | < | `findByPriceLessThan(price)` |
| `LessThanEqual` | <= | `findByPriceLessThanEqual(price)` |
| `GreaterThan` | > | `findByPriceGreaterThan(price)` |
| `GreaterThanEqual` | >= | `findByPriceGreaterThanEqual(price)` |
| `After` | > | `findByCreatedAtAfter(date)` |
| `Before` | < | `findByCreatedAtBefore(date)` |
| `IsNull` | IS NULL | `findBySalePriceIsNull()` |
| `IsNotNull`, `NotNull` | IS NOT NULL | `findBySalePriceIsNotNull()` |
| `Like` | LIKE | `findByNameLike("%phone%")` |
| `NotLike` | NOT LIKE | `findByNameNotLike("%test%")` |
| `StartingWith` | LIKE 'x%' | `findByNameStartingWith("iPhone")` |
| `EndingWith` | LIKE '%x' | `findByNameEndingWith("Pro")` |
| `Containing` | LIKE '%x%' | `findByNameContaining("phone")` |
| `OrderBy` | ORDER BY | `findByActiveOrderByCreatedAtDesc()` |
| `Not` | != | `findByStatusNot(status)` |
| `In` | IN | `findByIdIn(List<Long> ids)` |
| `NotIn` | NOT IN | `findByIdNotIn(List<Long> ids)` |
| `True` | = true | `findByIsActiveTrue()` |
| `False` | = false | `findByIsActiveFalse()` |
| `IgnoreCase` | UPPER() | `findByNameIgnoreCase(name)` |

### 2.3. Ví dụ thực tế

```java
@Repository
public interface ProductRepository extends JpaRepository<Product, Long> {

    // ─── BASIC QUERIES ───

    // SELECT * FROM products WHERE slug = ?
    Optional<Product> findBySlug(String slug);

    // SELECT * FROM products WHERE sku = ?
    Optional<Product> findBySku(String sku);

    // SELECT * FROM products WHERE is_active = true
    List<Product> findByIsActiveTrue();

    // SELECT * FROM products WHERE is_active = true AND is_featured = true
    List<Product> findByIsActiveTrueAndIsFeaturedTrue();

    // ─── COMPARISON ───

    // SELECT * FROM products WHERE price < ?
    List<Product> findByPriceLessThan(BigDecimal price);

    // SELECT * FROM products WHERE price BETWEEN ? AND ?
    List<Product> findByPriceBetween(BigDecimal minPrice, BigDecimal maxPrice);

    // SELECT * FROM products WHERE stock_quantity > 0
    List<Product> findByStockQuantityGreaterThan(Integer quantity);

    // ─── TEXT SEARCH ───

    // SELECT * FROM products WHERE name LIKE '%keyword%'
    List<Product> findByNameContaining(String keyword);

    // Case-insensitive: UPPER(name) LIKE UPPER('%keyword%')
    List<Product> findByNameContainingIgnoreCase(String keyword);

    // SELECT * FROM products WHERE name LIKE '%keyword%' OR description LIKE '%keyword%'
    List<Product> findByNameContainingIgnoreCaseOrDescriptionContainingIgnoreCase(
            String name, String description);

    // ─── RELATIONSHIP QUERIES ───

    // SELECT * FROM products WHERE category_id = ?
    List<Product> findByCategoryId(Long categoryId);

    // SELECT * FROM products WHERE category_id = ? AND is_active = true
    List<Product> findByCategoryIdAndIsActiveTrue(Long categoryId);

    // SELECT * FROM products p JOIN categories c ON p.category_id = c.id WHERE c.slug = ?
    List<Product> findByCategorySlug(String categorySlug);

    // ─── DATE QUERIES ───

    // SELECT * FROM products WHERE created_at > ?
    List<Product> findByCreatedAtAfter(LocalDateTime date);

    // SELECT * FROM products WHERE created_at BETWEEN ? AND ?
    List<Product> findByCreatedAtBetween(LocalDateTime start, LocalDateTime end);

    // ─── NULL CHECKS ───

    // SELECT * FROM products WHERE sale_price IS NOT NULL
    List<Product> findBySalePriceIsNotNull();

    // ─── ORDERING ───

    // SELECT * FROM products ORDER BY price ASC
    List<Product> findAllByOrderByPriceAsc();

    // SELECT * FROM products WHERE is_active = true ORDER BY created_at DESC
    List<Product> findByIsActiveTrueOrderByCreatedAtDesc();

    // ─── LIMITING RESULTS ───

    // SELECT * FROM products ORDER BY created_at DESC LIMIT 10
    List<Product> findTop10ByOrderByCreatedAtDesc();

    // SELECT * FROM products WHERE is_featured = true ORDER BY id DESC LIMIT 1
    Optional<Product> findFirstByIsFeaturedTrueOrderByIdDesc();

    // ─── COUNT & EXISTS ───

    // SELECT COUNT(*) FROM products WHERE category_id = ?
    long countByCategoryId(Long categoryId);

    // SELECT COUNT(*) FROM products WHERE is_active = true
    long countByIsActiveTrue();

    // SELECT EXISTS(SELECT 1 FROM products WHERE slug = ?)
    boolean existsBySlug(String slug);

    boolean existsBySku(String sku);

    // ─── DELETE ───

    // DELETE FROM products WHERE is_active = false
    void deleteByIsActiveFalse();

    // DELETE FROM products WHERE id IN (?)
    void deleteByIdIn(List<Long> ids);
}
```

---

## 3. @QUERY ANNOTATION

### 3.1. JPQL (Java Persistence Query Language)

```java
@Repository
public interface ProductRepository extends JpaRepository<Product, Long> {

    // ─── BASIC JPQL ───
    @Query("SELECT p FROM Product p WHERE p.isActive = true")
    List<Product> findAllActive();

    // ─── NAMED PARAMETERS ───
    @Query("SELECT p FROM Product p WHERE p.category.id = :categoryId AND p.isActive = true")
    List<Product> findActiveByCategory(@Param("categoryId") Long categoryId);

    // ─── POSITIONAL PARAMETERS ───
    @Query("SELECT p FROM Product p WHERE p.price BETWEEN ?1 AND ?2")
    List<Product> findByPriceRange(BigDecimal minPrice, BigDecimal maxPrice);

    // ─── JOIN FETCH (tránh N+1) ───
    @Query("SELECT p FROM Product p JOIN FETCH p.category WHERE p.id = :id")
    Optional<Product> findByIdWithCategory(@Param("id") Long id);

    @Query("SELECT p FROM Product p LEFT JOIN FETCH p.images WHERE p.id = :id")
    Optional<Product> findByIdWithImages(@Param("id") Long id);

    @Query("""
        SELECT DISTINCT p FROM Product p
        LEFT JOIN FETCH p.category
        LEFT JOIN FETCH p.images
        WHERE p.isActive = true
        """)
    List<Product> findAllWithCategoryAndImages();

    // ─── AGGREGATION ───
    @Query("SELECT AVG(p.price) FROM Product p WHERE p.category.id = :categoryId")
    BigDecimal findAveragePriceByCategory(@Param("categoryId") Long categoryId);

    @Query("SELECT COUNT(p) FROM Product p WHERE p.stockQuantity = 0")
    long countOutOfStock();

    @Query("SELECT SUM(p.stockQuantity) FROM Product p WHERE p.isActive = true")
    Long getTotalStock();

    // ─── COMPLEX CONDITIONS ───
    @Query("""
        SELECT p FROM Product p
        WHERE (:categoryId IS NULL OR p.category.id = :categoryId)
        AND (:minPrice IS NULL OR p.price >= :minPrice)
        AND (:maxPrice IS NULL OR p.price <= :maxPrice)
        AND (:keyword IS NULL OR LOWER(p.name) LIKE LOWER(CONCAT('%', :keyword, '%')))
        AND p.isActive = true
        ORDER BY p.createdAt DESC
        """)
    Page<Product> searchProducts(
            @Param("categoryId") Long categoryId,
            @Param("minPrice") BigDecimal minPrice,
            @Param("maxPrice") BigDecimal maxPrice,
            @Param("keyword") String keyword,
            Pageable pageable);

    // ─── CASE WHEN ───
    @Query("""
        SELECT p FROM Product p
        ORDER BY
        CASE WHEN p.isFeatured = true THEN 0 ELSE 1 END,
        p.createdAt DESC
        """)
    List<Product> findAllFeaturedFirst();
}
```

### 3.2. Native SQL

```java
@Repository
public interface ProductRepository extends JpaRepository<Product, Long> {

    // ─── NATIVE QUERY ───
    @Query(
        value = "SELECT * FROM products WHERE is_active = 1 ORDER BY RAND() LIMIT :limit",
        nativeQuery = true)
    List<Product> findRandomProducts(@Param("limit") int limit);

    // ─── FULLTEXT SEARCH (MySQL) ───
    @Query(
        value = "SELECT * FROM products WHERE MATCH(name, description) AGAINST(:keyword IN BOOLEAN MODE)",
        nativeQuery = true)
    List<Product> fullTextSearch(@Param("keyword") String keyword);

    // ─── COMPLEX NATIVE ───
    @Query(
        value = """
            SELECT p.*, c.name as category_name,
                   (SELECT COUNT(*) FROM order_items oi WHERE oi.product_id = p.id) as order_count
            FROM products p
            LEFT JOIN categories c ON p.category_id = c.id
            WHERE p.is_active = 1
            ORDER BY order_count DESC
            LIMIT :limit
            """,
        nativeQuery = true)
    List<Object[]> findBestSellingProducts(@Param("limit") int limit);

    // ─── COUNT NATIVE ───
    @Query(
        value = "SELECT COUNT(*) FROM products WHERE DATE(created_at) = CURDATE()",
        nativeQuery = true)
    long countProductsCreatedToday();
}
```

### 3.3. JPQL vs Native SQL

| Tiêu chí | JPQL | Native SQL |
|----------|------|------------|
| **Portable** | ✅ DB-independent | ❌ DB-specific |
| **Type-safe** | ✅ Entity names | ❌ Table names |
| **Join Fetch** | ✅ | ❌ Khó hơn |
| **DB Functions** | ⚠️ Hạn chế | ✅ Full access |
| **Performance** | ⚠️ Có overhead | ✅ Direct |
| **Maintenance** | ✅ Refactor-friendly | ❌ Table rename = sửa query |

**Quy tắc:** Ưu tiên JPQL. Dùng Native SQL khi cần DB-specific features.

---

## 4. MODIFYING QUERIES

### 4.1. UPDATE và DELETE

```java
@Repository
public interface ProductRepository extends JpaRepository<Product, Long> {

    // ─── UPDATE ───
    @Modifying
    @Query("UPDATE Product p SET p.isActive = false WHERE p.id = :id")
    int softDelete(@Param("id") Long id);

    @Modifying
    @Query("UPDATE Product p SET p.stockQuantity = p.stockQuantity - :quantity WHERE p.id = :id AND p.stockQuantity >= :quantity")
    int decreaseStock(@Param("id") Long id, @Param("quantity") int quantity);

    @Modifying
    @Query("UPDATE Product p SET p.stockQuantity = p.stockQuantity + :quantity WHERE p.id = :id")
    int increaseStock(@Param("id") Long id, @Param("quantity") int quantity);

    @Modifying
    @Query("UPDATE Product p SET p.price = p.price * :rate WHERE p.category.id = :categoryId")
    int updatePriceByCategory(@Param("categoryId") Long categoryId, @Param("rate") BigDecimal rate);

    // ─── BULK UPDATE ───
    @Modifying
    @Query("UPDATE Product p SET p.isFeatured = false WHERE p.isFeatured = true")
    int clearAllFeatured();

    @Modifying
    @Query("UPDATE Product p SET p.isFeatured = true WHERE p.id IN :ids")
    int setFeatured(@Param("ids") List<Long> ids);

    // ─── DELETE ───
    @Modifying
    @Query("DELETE FROM Product p WHERE p.isActive = false AND p.createdAt < :date")
    int deleteInactiveOlderThan(@Param("date") LocalDateTime date);

    // ─── NATIVE MODIFYING ───
    @Modifying
    @Query(value = "UPDATE products SET view_count = view_count + 1 WHERE id = :id", nativeQuery = true)
    void incrementViewCount(@Param("id") Long id);
}
```

### 4.2. Lưu ý với @Modifying

```java
@Service
@RequiredArgsConstructor
@Transactional(readOnly = true)
public class ProductService {

    private final ProductRepository productRepository;

    // ⚠️ QUAN TRỌNG: @Modifying cần @Transactional
    @Transactional  // Override readOnly
    public int deactivateProduct(Long id) {
        return productRepository.softDelete(id);
    }

    // ⚠️ Clear persistence context sau khi modifying
    @Transactional
    @Modifying(clearAutomatically = true, flushAutomatically = true)
    public void bulkUpdate() {
        productRepository.clearAllFeatured();
        // Persistence context đã được clear
        // findAll() sẽ query lại từ DB
    }
}
```

---

## 5. PROJECTIONS

### 5.1. Interface Projection (Closed Projection)

```java
// ─── PROJECTION INTERFACE ───
public interface ProductSummary {
    Long getId();
    String getName();
    String getSlug();
    BigDecimal getPrice();
    BigDecimal getSalePrice();

    // Computed property (SpEL)
    @Value("#{target.salePrice != null ? target.salePrice : target.price}")
    BigDecimal getEffectivePrice();
}

// ─── REPOSITORY ───
@Repository
public interface ProductRepository extends JpaRepository<Product, Long> {

    // Trả về Projection thay vì Entity
    List<ProductSummary> findByIsActiveTrue();

    Optional<ProductSummary> findSummaryById(Long id);

    // JPQL với Projection
    @Query("SELECT p FROM Product p WHERE p.category.id = :categoryId")
    List<ProductSummary> findSummaryByCategory(@Param("categoryId") Long categoryId);

    Page<ProductSummary> findByIsActiveTrue(Pageable pageable);
}

// ─── SỬ DỤNG ───
List<ProductSummary> summaries = productRepository.findByIsActiveTrue();
summaries.forEach(s -> {
    System.out.println(s.getName() + " - " + s.getEffectivePrice());
});
```

### 5.2. DTO Projection (Class-based)

```java
// ─── DTO CLASS ───
@Data
@AllArgsConstructor
public class ProductDTO {
    private Long id;
    private String name;
    private BigDecimal price;
    private String categoryName;  // Từ join
}

// ─── REPOSITORY ───
@Repository
public interface ProductRepository extends JpaRepository<Product, Long> {

    // Constructor Expression trong JPQL
    @Query("""
        SELECT new com.example.ecommerce.dto.response.ProductDTO(
            p.id, p.name, p.price, c.name
        )
        FROM Product p
        LEFT JOIN p.category c
        WHERE p.isActive = true
        """)
    List<ProductDTO> findAllProductDTOs();

    @Query("""
        SELECT new com.example.ecommerce.dto.response.ProductDTO(
            p.id, p.name, p.price, c.name
        )
        FROM Product p
        LEFT JOIN p.category c
        WHERE p.id = :id
        """)
    Optional<ProductDTO> findProductDTOById(@Param("id") Long id);
}
```

### 5.3. Dynamic Projection

```java
// ─── REPOSITORY ───
@Repository
public interface ProductRepository extends JpaRepository<Product, Long> {

    // Dynamic: type được quyết định runtime
    <T> List<T> findByIsActiveTrue(Class<T> type);

    <T> Optional<T> findById(Long id, Class<T> type);
}

// ─── SỬ DỤNG ───
// Trả về Entity đầy đủ
List<Product> products = productRepository.findByIsActiveTrue(Product.class);

// Trả về Projection
List<ProductSummary> summaries = productRepository.findByIsActiveTrue(ProductSummary.class);

// Trả về DTO
Optional<ProductDTO> dto = productRepository.findById(1L, ProductDTO.class);
```

---

## 6. PAGINATION & SORTING

### 6.1. Sử dụng Pageable

```java
@Repository
public interface ProductRepository extends JpaRepository<Product, Long> {

    Page<Product> findByIsActiveTrue(Pageable pageable);

    Page<Product> findByCategoryId(Long categoryId, Pageable pageable);

    @Query("SELECT p FROM Product p WHERE p.isActive = true")
    Page<Product> findAllActive(Pageable pageable);

    // Slice (không count total - nhẹ hơn Page)
    Slice<Product> findByIsFeaturedTrue(Pageable pageable);
}

// ─── SỬ DỤNG ───
@Service
public class ProductService {

    public Page<ProductResponse> getProducts(int page, int size, String sortBy, String direction) {
        Sort sort = direction.equalsIgnoreCase("desc")
                ? Sort.by(sortBy).descending()
                : Sort.by(sortBy).ascending();

        Pageable pageable = PageRequest.of(page, size, sort);

        Page<Product> productPage = productRepository.findByIsActiveTrue(pageable);

        return productPage.map(this::toResponse);
    }

    // Multiple sort fields
    public Page<Product> getProductsSorted(int page, int size) {
        Sort sort = Sort.by(
            Sort.Order.desc("isFeatured"),
            Sort.Order.desc("createdAt")
        );
        return productRepository.findByIsActiveTrue(PageRequest.of(page, size, sort));
    }
}
```

### 6.2. Page vs Slice vs List

| Return Type | Có total count? | Use case |
|-------------|----------------|----------|
| `Page<T>` | ✅ SELECT COUNT(*) | Hiển thị tổng số trang |
| `Slice<T>` | ❌ | Infinite scroll, "Load more" |
| `List<T>` | ❌ | Không cần pagination |

---

## 7. BÀI TẬP & CHECKLIST

### Bài tập thực hành

- [ ] Tạo `UserRepository` với 5+ derived query methods
- [ ] Tạo `OrderRepository` với @Query (JPQL + Native)
- [ ] Implement `ProductRepository.searchProducts()` với dynamic filters
- [ ] Tạo `ProductSummary` projection interface
- [ ] Tạo `ProductDTO` và query với constructor expression
- [ ] Implement pagination cho product listing
- [ ] Viết modifying query cho soft delete
- [ ] Test tất cả queries với Postman/curl

### Kiến thức cần nắm

| Khái niệm | Hiểu chưa? |
|-----------|-------------|
| JpaRepository hierarchy | ☐ |
| Derived Query Method keywords | ☐ |
| @Query với JPQL | ☐ |
| @Query với Native SQL | ☐ |
| Named Parameters vs Positional | ☐ |
| @Modifying cho UPDATE/DELETE | ☐ |
| Interface Projection | ☐ |
| DTO Projection (constructor expression) | ☐ |
| Page vs Slice | ☐ |
| Pageable và Sort | ☐ |

---

**Trước đó:** [← Ngày 07 - Flyway Migration](day-07-flyway-migration.md)

**Tiếp theo:** [Ngày 09 - Pagination, Sorting & Specification →](day-09-pagination-specification.md)
