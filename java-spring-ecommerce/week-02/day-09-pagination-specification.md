# Ngày 09: Pagination, Sorting & Specification Pattern

## Mục tiêu hôm nay
- Hiểu sâu Spring Data Pagination (Page, Slice, Pageable)
- Master Sorting với nhiều fields
- Sử dụng JPA Specification cho dynamic queries
- Kết hợp Specification với Pagination
- Xây dựng Search API linh hoạt

---

## 1. PAGINATION DEEP DIVE

### 1.1. Pageable Interface

```java
// Pageable = page + size + sort
public interface Pageable {
    int getPageNumber();        // 0-based
    int getPageSize();
    long getOffset();           // = pageNumber * pageSize
    Sort getSort();
    Pageable next();
    Pageable previousOrFirst();
    Pageable first();
    boolean hasPrevious();
}

// Tạo Pageable
Pageable pageable = PageRequest.of(0, 10);                      // Page 0, size 10
Pageable pageable = PageRequest.of(0, 10, Sort.by("name"));     // Với sort
Pageable pageable = PageRequest.ofSize(20);                     // Page 0, size 20
```

### 1.2. Page vs Slice

```
┌─────────────────────────────────────────────────────────────┐
│                         PAGE<T>                              │
├─────────────────────────────────────────────────────────────┤
│  • Chứa data + metadata                                      │
│  • Thực hiện SELECT COUNT(*) → biết tổng số records          │
│  • Phù hợp: UI cần hiển thị "Trang 1/10" hoặc page numbers   │
│  • Overhead: Query thêm COUNT                                │
│                                                              │
│  Methods:                                                    │
│  - getContent()        → List<T>                             │
│  - getTotalElements()  → long (tổng records)                 │
│  - getTotalPages()     → int                                 │
│  - getNumber()         → int (page hiện tại, 0-based)        │
│  - getSize()           → int (page size)                     │
│  - hasNext()           → boolean                             │
│  - hasPrevious()       → boolean                             │
│  - isFirst()           → boolean                             │
│  - isLast()            → boolean                             │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│                        SLICE<T>                              │
├─────────────────────────────────────────────────────────────┤
│  • Chứa data + hasNext (KHÔNG có total)                      │
│  • KHÔNG thực hiện COUNT → nhẹ hơn Page                      │
│  • Phù hợp: Infinite scroll, "Load more" button              │
│  • Query thêm 1 record để check hasNext                      │
│                                                              │
│  Methods:                                                    │
│  - getContent()        → List<T>                             │
│  - getNumber()         → int                                 │
│  - getSize()           → int                                 │
│  - hasNext()           → boolean (có trang tiếp không)       │
│  - hasPrevious()       → boolean                             │
│  - isFirst()           → boolean                             │
│  - isLast()            → boolean                             │
│                                                              │
│  ❌ KHÔNG CÓ: getTotalElements(), getTotalPages()            │
└─────────────────────────────────────────────────────────────┘
```

### 1.3. Pagination Response DTO

```java
package com.example.ecommerce.dto.response;

import lombok.AllArgsConstructor;
import lombok.Builder;
import lombok.Data;
import lombok.NoArgsConstructor;
import org.springframework.data.domain.Page;

import java.util.List;
import java.util.function.Function;

@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class PagedResponse<T> {

    private List<T> content;
    private int pageNumber;
    private int pageSize;
    private long totalElements;
    private int totalPages;
    private boolean first;
    private boolean last;
    private boolean empty;

    // Factory method từ Page
    public static <T> PagedResponse<T> from(Page<T> page) {
        return PagedResponse.<T>builder()
                .content(page.getContent())
                .pageNumber(page.getNumber())
                .pageSize(page.getSize())
                .totalElements(page.getTotalElements())
                .totalPages(page.getTotalPages())
                .first(page.isFirst())
                .last(page.isLast())
                .empty(page.isEmpty())
                .build();
    }

    // Factory method với mapper (Entity → DTO)
    public static <T, R> PagedResponse<R> from(Page<T> page, Function<T, R> mapper) {
        List<R> content = page.getContent().stream()
                .map(mapper)
                .toList();

        return PagedResponse.<R>builder()
                .content(content)
                .pageNumber(page.getNumber())
                .pageSize(page.getSize())
                .totalElements(page.getTotalElements())
                .totalPages(page.getTotalPages())
                .first(page.isFirst())
                .last(page.isLast())
                .empty(page.isEmpty())
                .build();
    }
}
```

---

## 2. SORTING

### 2.1. Sort Class

```java
// ─── Single field ───
Sort sort = Sort.by("name");                         // ASC mặc định
Sort sort = Sort.by("name").ascending();
Sort sort = Sort.by("name").descending();
Sort sort = Sort.by(Sort.Direction.DESC, "name");

// ─── Multiple fields ───
Sort sort = Sort.by("category", "name");             // category ASC, name ASC
Sort sort = Sort.by(
    Sort.Order.desc("isFeatured"),
    Sort.Order.desc("createdAt"),
    Sort.Order.asc("name")
);

// ─── Null handling ───
Sort sort = Sort.by(Sort.Order.asc("salePrice").nullsLast());   // NULL cuối
Sort sort = Sort.by(Sort.Order.desc("salePrice").nullsFirst()); // NULL đầu

// ─── Ignore case ───
Sort sort = Sort.by(Sort.Order.asc("name").ignoreCase());

// ─── Combine với Pageable ───
Pageable pageable = PageRequest.of(0, 10, Sort.by("createdAt").descending());
```

### 2.2. Dynamic Sort từ Request

```java
@RestController
@RequestMapping("/api/products")
@RequiredArgsConstructor
public class ProductController {

    private final ProductService productService;

    @GetMapping
    public PagedResponse<ProductResponse> getProducts(
            @RequestParam(defaultValue = "0") int page,
            @RequestParam(defaultValue = "10") int size,
            @RequestParam(defaultValue = "createdAt") String sortBy,
            @RequestParam(defaultValue = "desc") String sortDir
    ) {
        return productService.getProducts(page, size, sortBy, sortDir);
    }
}

@Service
@RequiredArgsConstructor
public class ProductService {

    private final ProductRepository productRepository;

    // Whitelist allowed sort fields (bảo mật)
    private static final Set<String> ALLOWED_SORT_FIELDS = Set.of(
            "id", "name", "price", "createdAt", "stockQuantity"
    );

    public PagedResponse<ProductResponse> getProducts(
            int page, int size, String sortBy, String sortDir
    ) {
        // Validate sort field
        if (!ALLOWED_SORT_FIELDS.contains(sortBy)) {
            sortBy = "createdAt";  // Default
        }

        // Limit page size
        size = Math.min(size, 100);

        Sort sort = sortDir.equalsIgnoreCase("asc")
                ? Sort.by(sortBy).ascending()
                : Sort.by(sortBy).descending();

        Pageable pageable = PageRequest.of(page, size, sort);
        Page<Product> productPage = productRepository.findByIsActiveTrue(pageable);

        return PagedResponse.from(productPage, this::toResponse);
    }

    private ProductResponse toResponse(Product product) {
        return ProductResponse.builder()
                .id(product.getId())
                .name(product.getName())
                .price(product.getPrice())
                .build();
    }
}
```

---

## 3. JPA SPECIFICATION PATTERN

### 3.1. Tại sao cần Specification?

```java
// ❌ VẤN ĐỀ: Phải tạo nhiều method cho mỗi combination filter
interface ProductRepository {
    List<Product> findByCategory(Category category);
    List<Product> findByCategoryAndPriceLessThan(Category category, BigDecimal price);
    List<Product> findByCategoryAndPriceBetween(Category category, BigDecimal min, BigDecimal max);
    List<Product> findByNameContaining(String name);
    List<Product> findByNameContainingAndCategory(String name, Category category);
    List<Product> findByNameContainingAndCategoryAndPriceBetween(...);
    // Explosion của methods! N filters → 2^N combinations
}

// ✅ GIẢI PHÁP: Specification Pattern
// Mỗi filter = 1 Specification
// Combine bằng and(), or(), not()
```

### 3.2. Cách implement Specification

```java
// ─── Repository phải extends JpaSpecificationExecutor ───
@Repository
public interface ProductRepository extends JpaRepository<Product, Long>,
        JpaSpecificationExecutor<Product> {
    // Tự động có: findAll(Specification), count(Specification), exists(Specification)
}
```

```java
// ─── Specification Class ───
package com.example.ecommerce.repository.specification;

import com.example.ecommerce.entity.Product;
import org.springframework.data.jpa.domain.Specification;
import jakarta.persistence.criteria.*;

import java.math.BigDecimal;

public class ProductSpecification {

    // ─── BASIC SPECS ───

    public static Specification<Product> hasCategory(Long categoryId) {
        return (root, query, cb) -> {
            if (categoryId == null) {
                return cb.conjunction();  // Không filter (WHERE 1=1)
            }
            return cb.equal(root.get("category").get("id"), categoryId);
        };
    }

    public static Specification<Product> isActive() {
        return (root, query, cb) -> cb.isTrue(root.get("isActive"));
    }

    public static Specification<Product> isFeatured() {
        return (root, query, cb) -> cb.isTrue(root.get("isFeatured"));
    }

    public static Specification<Product> inStock() {
        return (root, query, cb) -> cb.greaterThan(root.get("stockQuantity"), 0);
    }

    // ─── PRICE RANGE ───

    public static Specification<Product> priceBetween(BigDecimal min, BigDecimal max) {
        return (root, query, cb) -> {
            if (min == null && max == null) {
                return cb.conjunction();
            }
            if (min == null) {
                return cb.lessThanOrEqualTo(root.get("price"), max);
            }
            if (max == null) {
                return cb.greaterThanOrEqualTo(root.get("price"), min);
            }
            return cb.between(root.get("price"), min, max);
        };
    }

    public static Specification<Product> priceGreaterThan(BigDecimal price) {
        return (root, query, cb) -> {
            if (price == null) return cb.conjunction();
            return cb.greaterThan(root.get("price"), price);
        };
    }

    public static Specification<Product> priceLessThan(BigDecimal price) {
        return (root, query, cb) -> {
            if (price == null) return cb.conjunction();
            return cb.lessThan(root.get("price"), price);
        };
    }

    // ─── TEXT SEARCH ───

    public static Specification<Product> nameContains(String keyword) {
        return (root, query, cb) -> {
            if (keyword == null || keyword.isBlank()) {
                return cb.conjunction();
            }
            String likePattern = "%" + keyword.toLowerCase() + "%";
            return cb.like(cb.lower(root.get("name")), likePattern);
        };
    }

    public static Specification<Product> searchKeyword(String keyword) {
        return (root, query, cb) -> {
            if (keyword == null || keyword.isBlank()) {
                return cb.conjunction();
            }
            String likePattern = "%" + keyword.toLowerCase() + "%";
            return cb.or(
                    cb.like(cb.lower(root.get("name")), likePattern),
                    cb.like(cb.lower(root.get("description")), likePattern),
                    cb.like(cb.lower(root.get("sku")), likePattern)
            );
        };
    }

    // ─── DATE FILTERS ───

    public static Specification<Product> createdAfter(LocalDateTime date) {
        return (root, query, cb) -> {
            if (date == null) return cb.conjunction();
            return cb.greaterThanOrEqualTo(root.get("createdAt"), date);
        };
    }

    public static Specification<Product> createdBefore(LocalDateTime date) {
        return (root, query, cb) -> {
            if (date == null) return cb.conjunction();
            return cb.lessThanOrEqualTo(root.get("createdAt"), date);
        };
    }

    // ─── WITH JOIN (tránh N+1) ───

    public static Specification<Product> withCategory() {
        return (root, query, cb) -> {
            // Fetch join để load category cùng lúc
            if (Long.class != query.getResultType()) {  // Không fetch khi COUNT
                root.fetch("category", JoinType.LEFT);
            }
            return cb.conjunction();
        };
    }

    // ─── COLLECTION FILTER ───

    public static Specification<Product> hasTag(String tagSlug) {
        return (root, query, cb) -> {
            if (tagSlug == null) return cb.conjunction();
            Join<Object, Object> tags = root.join("tags", JoinType.INNER);
            return cb.equal(tags.get("slug"), tagSlug);
        };
    }
}
```

### 3.3. Combine Specifications

```java
@Service
@RequiredArgsConstructor
public class ProductService {

    private final ProductRepository productRepository;

    public PagedResponse<ProductResponse> searchProducts(ProductSearchRequest request) {
        // Build Specification từ request
        Specification<Product> spec = Specification.where(ProductSpecification.isActive())
                .and(ProductSpecification.hasCategory(request.getCategoryId()))
                .and(ProductSpecification.priceBetween(request.getMinPrice(), request.getMaxPrice()))
                .and(ProductSpecification.searchKeyword(request.getKeyword()))
                .and(ProductSpecification.hasTag(request.getTag()));

        // Add stock filter if requested
        if (Boolean.TRUE.equals(request.getInStockOnly())) {
            spec = spec.and(ProductSpecification.inStock());
        }

        // Add featured filter
        if (Boolean.TRUE.equals(request.getFeaturedOnly())) {
            spec = spec.and(ProductSpecification.isFeatured());
        }

        // Pageable với sort
        Sort sort = buildSort(request.getSortBy(), request.getSortDir());
        Pageable pageable = PageRequest.of(
                request.getPage(),
                Math.min(request.getSize(), 100),
                sort
        );

        // Execute query
        Page<Product> productPage = productRepository.findAll(spec, pageable);

        return PagedResponse.from(productPage, this::toResponse);
    }

    private Sort buildSort(String sortBy, String sortDir) {
        if (sortBy == null) sortBy = "createdAt";
        if (sortDir == null) sortDir = "desc";

        return sortDir.equalsIgnoreCase("asc")
                ? Sort.by(sortBy).ascending()
                : Sort.by(sortBy).descending();
    }
}
```

---

## 4. SEARCH REQUEST DTO

### 4.1. ProductSearchRequest

```java
package com.example.ecommerce.dto.request;

import jakarta.validation.constraints.Max;
import jakarta.validation.constraints.Min;
import jakarta.validation.constraints.PositiveOrZero;
import lombok.Data;

import java.math.BigDecimal;

@Data
public class ProductSearchRequest {

    private String keyword;

    private Long categoryId;

    @PositiveOrZero
    private BigDecimal minPrice;

    @PositiveOrZero
    private BigDecimal maxPrice;

    private String tag;

    private Boolean inStockOnly;

    private Boolean featuredOnly;

    @Min(0)
    private int page = 0;

    @Min(1)
    @Max(100)
    private int size = 10;

    private String sortBy = "createdAt";

    private String sortDir = "desc";
}
```

### 4.2. Controller

```java
@RestController
@RequestMapping("/api/products")
@RequiredArgsConstructor
public class ProductController {

    private final ProductService productService;

    // GET /api/products?keyword=iphone&categoryId=1&minPrice=1000000&page=0&size=10&sortBy=price&sortDir=asc
    @GetMapping
    public ApiResponse<PagedResponse<ProductResponse>> searchProducts(
            @Valid ProductSearchRequest request
    ) {
        PagedResponse<ProductResponse> result = productService.searchProducts(request);
        return ApiResponse.success(result);
    }

    // Alternative: POST với body
    @PostMapping("/search")
    public ApiResponse<PagedResponse<ProductResponse>> searchProductsPost(
            @Valid @RequestBody ProductSearchRequest request
    ) {
        PagedResponse<ProductResponse> result = productService.searchProducts(request);
        return ApiResponse.success(result);
    }
}
```

---

## 5. SPECIFICATION BUILDER PATTERN

### 5.1. Fluent Builder cho complex queries

```java
package com.example.ecommerce.repository.specification;

import com.example.ecommerce.entity.Product;
import org.springframework.data.jpa.domain.Specification;

import java.math.BigDecimal;
import java.time.LocalDateTime;

public class ProductSpecBuilder {

    private Specification<Product> spec;

    private ProductSpecBuilder() {
        this.spec = Specification.where(null);
    }

    public static ProductSpecBuilder builder() {
        return new ProductSpecBuilder();
    }

    public ProductSpecBuilder active() {
        spec = spec.and(ProductSpecification.isActive());
        return this;
    }

    public ProductSpecBuilder featured() {
        spec = spec.and(ProductSpecification.isFeatured());
        return this;
    }

    public ProductSpecBuilder inStock() {
        spec = spec.and(ProductSpecification.inStock());
        return this;
    }

    public ProductSpecBuilder categoryId(Long categoryId) {
        if (categoryId != null) {
            spec = spec.and(ProductSpecification.hasCategory(categoryId));
        }
        return this;
    }

    public ProductSpecBuilder priceBetween(BigDecimal min, BigDecimal max) {
        spec = spec.and(ProductSpecification.priceBetween(min, max));
        return this;
    }

    public ProductSpecBuilder keyword(String keyword) {
        if (keyword != null && !keyword.isBlank()) {
            spec = spec.and(ProductSpecification.searchKeyword(keyword));
        }
        return this;
    }

    public ProductSpecBuilder tag(String tagSlug) {
        if (tagSlug != null) {
            spec = spec.and(ProductSpecification.hasTag(tagSlug));
        }
        return this;
    }

    public ProductSpecBuilder createdAfter(LocalDateTime date) {
        if (date != null) {
            spec = spec.and(ProductSpecification.createdAfter(date));
        }
        return this;
    }

    public ProductSpecBuilder withCategory() {
        spec = spec.and(ProductSpecification.withCategory());
        return this;
    }

    public Specification<Product> build() {
        return spec;
    }
}
```

### 5.2. Sử dụng Builder

```java
@Service
public class ProductService {

    public PagedResponse<ProductResponse> searchProducts(ProductSearchRequest request) {
        // Fluent builder - dễ đọc!
        Specification<Product> spec = ProductSpecBuilder.builder()
                .active()
                .categoryId(request.getCategoryId())
                .keyword(request.getKeyword())
                .priceBetween(request.getMinPrice(), request.getMaxPrice())
                .tag(request.getTag())
                .build();

        if (Boolean.TRUE.equals(request.getInStockOnly())) {
            spec = spec.and(ProductSpecification.inStock());
        }

        Pageable pageable = PageRequest.of(
                request.getPage(),
                request.getSize(),
                Sort.by(request.getSortDir().equalsIgnoreCase("asc")
                        ? Sort.Direction.ASC
                        : Sort.Direction.DESC,
                        request.getSortBy())
        );

        Page<Product> page = productRepository.findAll(spec, pageable);
        return PagedResponse.from(page, this::toResponse);
    }
}
```

---

## 6. ADVANCED: CRITERIA API

### 6.1. Hiểu Criteria API (nền tảng của Specification)

```java
public static Specification<Product> complexSearch(
        String keyword, Long categoryId, BigDecimal minPrice, BigDecimal maxPrice
) {
    return (Root<Product> root, CriteriaQuery<?> query, CriteriaBuilder cb) -> {
        List<Predicate> predicates = new ArrayList<>();

        // Always active
        predicates.add(cb.isTrue(root.get("isActive")));

        // Keyword search (name OR description)
        if (keyword != null && !keyword.isBlank()) {
            String pattern = "%" + keyword.toLowerCase() + "%";
            Predicate namePredicate = cb.like(cb.lower(root.get("name")), pattern);
            Predicate descPredicate = cb.like(cb.lower(root.get("description")), pattern);
            predicates.add(cb.or(namePredicate, descPredicate));
        }

        // Category
        if (categoryId != null) {
            predicates.add(cb.equal(root.get("category").get("id"), categoryId));
        }

        // Price range
        if (minPrice != null) {
            predicates.add(cb.greaterThanOrEqualTo(root.get("price"), minPrice));
        }
        if (maxPrice != null) {
            predicates.add(cb.lessThanOrEqualTo(root.get("price"), maxPrice));
        }

        // Distinct để tránh duplicate khi join
        query.distinct(true);

        // Order by featured DESC, createdAt DESC
        if (!Long.class.equals(query.getResultType())) {  // Skip for COUNT query
            query.orderBy(
                    cb.desc(root.get("isFeatured")),
                    cb.desc(root.get("createdAt"))
            );
        }

        return cb.and(predicates.toArray(new Predicate[0]));
    };
}
```

### 6.2. Subquery trong Specification

```java
// Tìm products có ít nhất 1 order (best sellers)
public static Specification<Product> hasSales() {
    return (root, query, cb) -> {
        Subquery<Long> subquery = query.subquery(Long.class);
        Root<OrderItem> orderItemRoot = subquery.from(OrderItem.class);

        subquery.select(cb.literal(1L))
                .where(cb.equal(orderItemRoot.get("product"), root));

        return cb.exists(subquery);
    };
}

// Tìm products có rating >= 4
public static Specification<Product> hasGoodRating(double minRating) {
    return (root, query, cb) -> {
        Subquery<Double> subquery = query.subquery(Double.class);
        Root<Review> reviewRoot = subquery.from(Review.class);

        subquery.select(cb.avg(reviewRoot.get("rating")))
                .where(cb.equal(reviewRoot.get("product"), root))
                .groupBy(reviewRoot.get("product"));

        return cb.greaterThanOrEqualTo(subquery, minRating);
    };
}
```

---

## 7. BÀI TẬP & CHECKLIST

### Bài tập thực hành

- [ ] Tạo `PagedResponse` DTO
- [ ] Tạo `ProductSearchRequest` DTO với validation
- [ ] Implement `ProductSpecification` với 10+ specs
- [ ] Tạo `ProductSpecBuilder` fluent builder
- [ ] Implement `ProductController.searchProducts()`
- [ ] Test với các filter combinations
- [ ] Test pagination (page, size)
- [ ] Test sorting (sortBy, sortDir)
- [ ] Validate sort field (whitelist)
- [ ] Test với Postman/curl

### Kiến thức cần nắm

| Khái niệm | Hiểu chưa? |
|-----------|-------------|
| Page vs Slice | ☐ |
| Pageable interface | ☐ |
| Sort với multiple fields | ☐ |
| JpaSpecificationExecutor | ☐ |
| Specification<T> interface | ☐ |
| CriteriaBuilder, Root, Predicate | ☐ |
| Combine specs với and(), or() | ☐ |
| Specification Builder pattern | ☐ |
| Sort field whitelist (security) | ☐ |

---

**Trước đó:** [← Ngày 08 - Repository Queries](day-08-repository-queries.md)

**Tiếp theo:** [Ngày 10 - JPA Performance Optimization →](day-10-jpa-performance.md)
