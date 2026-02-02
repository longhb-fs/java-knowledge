# Day 2: JPA + Specification (Dynamic Query)

## Mục tiêu
- Tạo Entity và Repository
- Query methods cơ bản
- **JPA Specification** cho dynamic filter (thay thế LINQ)
- Áp dụng cho chức năng "Lọc nâng cao"

## 1. Entity - So sánh với EF Core

### 1.1. Entity Framework (.NET)

```csharp
// .NET Entity
public class MarketInfo
{
    public long Id { get; set; }
    public string IndexCode { get; set; }
    public decimal Value { get; set; }
    public DateTime CreatedAt { get; set; }
}
```

### 1.2. JPA Entity (Spring Boot 4.0)

```java
import jakarta.persistence.*;  // Jakarta namespace (không còn javax)

@Entity
@Table(name = "market_info")
@Data  // Lombok: getter, setter, toString, equals, hashCode
@NoArgsConstructor
@AllArgsConstructor
@Builder
public class MarketInfo {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(name = "index_code", nullable = false, length = 50)
    private String indexCode;  // Chỉ số: HNX30, VN30...

    @Column(name = "value", precision = 18, scale = 2)
    private BigDecimal value;  // Giá trị

    @Column(name = "trading_volume")
    private Long tradingVolume;  // Khối lượng giao dịch

    @Column(name = "trading_value", precision = 18, scale = 2)
    private BigDecimal tradingValue;  // Giá trị giao dịch

    @Column(name = "foreign_buy_volume")
    private Long foreignBuyVolume;  // Khối lượng NĐTNN mua

    @Column(name = "foreign_sell_volume")
    private Long foreignSellVolume;  // Khối lượng NĐTNN bán

    @Column(name = "change_5_sessions", precision = 10, scale = 2)
    private BigDecimal change5Sessions;  // Biến động 5 phiên

    @Column(name = "change_10_sessions", precision = 10, scale = 2)
    private BigDecimal change10Sessions;  // Biến động 10 phiên

    @Column(name = "trading_date")
    private LocalDate tradingDate;  // Ngày giao dịch

    @Column(name = "created_at")
    private LocalDateTime createdAt;

    @Column(name = "updated_at")
    private LocalDateTime updatedAt;

    @PrePersist
    protected void onCreate() {
        createdAt = LocalDateTime.now();
        updatedAt = LocalDateTime.now();
    }

    @PreUpdate
    protected void onUpdate() {
        updatedAt = LocalDateTime.now();
    }
}
```

### 1.3. So sánh Annotation

| JPA | EF Core | Mô tả |
|-----|---------|-------|
| `@Entity` | `[Table]` | Đánh dấu entity |
| `@Table(name="...")` | `ToTable("...")` | Tên table |
| `@Id` | `[Key]` | Primary key |
| `@GeneratedValue` | `.ValueGeneratedOnAdd()` | Auto increment |
| `@Column` | `[Column]` | Column mapping |
| `@ManyToOne` | Navigation property | Relationship |

## 2. Repository

### 2.1. Basic Repository

```java
// Tương đương DbSet<MarketInfo> trong EF Core
@Repository
public interface MarketInfoRepository extends JpaRepository<MarketInfo, Long> {
    // JpaRepository cung cấp sẵn:
    // - findAll()
    // - findById(id)
    // - save(entity)
    // - deleteById(id)
    // - count()
    // ... và nhiều method khác
}
```

### 2.2. Query Methods - Tương đương LINQ đơn giản

```java
@Repository
public interface MarketInfoRepository extends JpaRepository<MarketInfo, Long> {

    // SELECT * FROM market_info WHERE index_code = ?
    List<MarketInfo> findByIndexCode(String indexCode);

    // SELECT * FROM market_info WHERE index_code LIKE '%keyword%'
    List<MarketInfo> findByIndexCodeContaining(String keyword);

    // SELECT * FROM market_info WHERE value > ?
    List<MarketInfo> findByValueGreaterThan(BigDecimal value);

    // SELECT * FROM market_info WHERE trading_date BETWEEN ? AND ?
    List<MarketInfo> findByTradingDateBetween(LocalDate start, LocalDate end);

    // SELECT * FROM market_info WHERE index_code = ? AND value > ?
    List<MarketInfo> findByIndexCodeAndValueGreaterThan(String indexCode, BigDecimal value);

    // Custom query với @Query
    @Query("SELECT m FROM MarketInfo m WHERE m.indexCode LIKE %:keyword% ORDER BY m.value DESC")
    List<MarketInfo> searchByKeyword(@Param("keyword") String keyword);

    // Native SQL query
    @Query(value = "SELECT * FROM market_info WHERE MATCH(index_code) AGAINST(:keyword)", nativeQuery = true)
    List<MarketInfo> fullTextSearch(@Param("keyword") String keyword);
}
```

### 2.3. So sánh với EF Core LINQ

```csharp
// EF Core LINQ
var results = context.MarketInfos
    .Where(m => m.IndexCode.Contains(keyword))
    .Where(m => m.Value > minValue)
    .OrderByDescending(m => m.Value)
    .ToList();
```

```java
// Spring Query Method (đơn giản)
List<MarketInfo> findByIndexCodeContainingAndValueGreaterThanOrderByValueDesc(
    String keyword, BigDecimal minValue);

// Nhưng tên method quá dài! -> Dùng Specification
```

## 3. JPA Specification - Dynamic Query (Thay thế LINQ)

### 3.1. Vấn đề: Filter động như trong SRS

SRS yêu cầu filter theo nhiều tiêu chí **TÙY CHỌN**:
- Chỉ số (dropdown)
- Giá trị (text)
- Biến động so với n phiên trước (text)

User có thể nhập **bất kỳ combination nào** → Cần dynamic query.

### 3.2. DTO cho Search Request

```java
@Data
public class MarketInfoSearchRequest {
    private String indexCode;           // Lọc theo chỉ số
    private String valueKeyword;        // Tìm kiếm giá trị
    private BigDecimal minValue;        // Giá trị từ
    private BigDecimal maxValue;        // Giá trị đến
    private LocalDate fromDate;         // Từ ngày
    private LocalDate toDate;           // Đến ngày
    private String changeKeyword;       // Biến động
}
```

### 3.3. Enable JpaSpecificationExecutor

```java
@Repository
public interface MarketInfoRepository extends
        JpaRepository<MarketInfo, Long>,
        JpaSpecificationExecutor<MarketInfo> {  // Thêm dòng này!
}
```

### 3.4. Tạo Specification Class

```java
public class MarketInfoSpecification {

    // Filter theo index code (exact match)
    public static Specification<MarketInfo> hasIndexCode(String indexCode) {
        return (root, query, cb) -> {
            if (indexCode == null || indexCode.isEmpty()) {
                return null;  // Bỏ qua điều kiện này
            }
            return cb.equal(root.get("indexCode"), indexCode);
        };
    }

    // Filter theo keyword (LIKE)
    public static Specification<MarketInfo> indexCodeContains(String keyword) {
        return (root, query, cb) -> {
            if (keyword == null || keyword.isEmpty()) {
                return null;
            }
            return cb.like(
                cb.lower(root.get("indexCode")),
                "%" + keyword.toLowerCase() + "%"
            );
        };
    }

    // Filter theo range giá trị
    public static Specification<MarketInfo> valueBetween(BigDecimal min, BigDecimal max) {
        return (root, query, cb) -> {
            if (min == null && max == null) {
                return null;
            }
            if (min != null && max != null) {
                return cb.between(root.get("value"), min, max);
            }
            if (min != null) {
                return cb.greaterThanOrEqualTo(root.get("value"), min);
            }
            return cb.lessThanOrEqualTo(root.get("value"), max);
        };
    }

    // Filter theo date range
    public static Specification<MarketInfo> tradingDateBetween(LocalDate from, LocalDate to) {
        return (root, query, cb) -> {
            if (from == null && to == null) {
                return null;
            }
            if (from != null && to != null) {
                return cb.between(root.get("tradingDate"), from, to);
            }
            if (from != null) {
                return cb.greaterThanOrEqualTo(root.get("tradingDate"), from);
            }
            return cb.lessThanOrEqualTo(root.get("tradingDate"), to);
        };
    }

    // Combine all filters
    public static Specification<MarketInfo> buildSearchSpec(MarketInfoSearchRequest request) {
        return Specification
            .where(hasIndexCode(request.getIndexCode()))
            .and(indexCodeContains(request.getValueKeyword()))
            .and(valueBetween(request.getMinValue(), request.getMaxValue()))
            .and(tradingDateBetween(request.getFromDate(), request.getToDate()));
    }
}
```

### 3.5. Sử dụng trong Service

```java
@Service
@RequiredArgsConstructor
public class MarketInfoServiceImpl implements MarketInfoService {

    private final MarketInfoRepository repository;

    public Page<MarketInfo> search(MarketInfoSearchRequest request, Pageable pageable) {
        // Build dynamic specification từ request
        Specification<MarketInfo> spec = MarketInfoSpecification.buildSearchSpec(request);

        // Execute với pagination
        return repository.findAll(spec, pageable);
    }
}
```

### 3.6. So sánh với EF Core Expression Builder

```csharp
// EF Core - Dynamic query với Expression
public IQueryable<MarketInfo> Search(MarketInfoSearchRequest request)
{
    var query = _context.MarketInfos.AsQueryable();

    if (!string.IsNullOrEmpty(request.IndexCode))
        query = query.Where(m => m.IndexCode == request.IndexCode);

    if (request.MinValue.HasValue)
        query = query.Where(m => m.Value >= request.MinValue);

    return query;
}
```

```java
// Spring Specification - tương đương nhưng pattern khác
Specification<MarketInfo> spec = Specification
    .where(hasIndexCode(request.getIndexCode()))
    .and(valueBetween(request.getMinValue(), request.getMaxValue()));

return repository.findAll(spec, pageable);
```

## 4. Ví dụ hoàn chỉnh cho SRS "Quản lý thông tin thị trường"

### 4.1. Entity

```java
@Entity
@Table(name = "market_info")
@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class MarketInfo {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(name = "index_code", length = 50)
    private String indexCode;  // HNX30, VN30

    @Column(precision = 18, scale = 3)
    private BigDecimal value;

    @Column(name = "trading_volume")
    private Long tradingVolume;

    @Column(name = "trading_value", precision = 18, scale = 2)
    private BigDecimal tradingValue;

    @Column(name = "foreign_buy_volume")
    private Long foreignBuyVolume;

    @Column(name = "foreign_buy_value", precision = 18, scale = 2)
    private BigDecimal foreignBuyValue;

    @Column(name = "foreign_sell_volume")
    private Long foreignSellVolume;

    @Column(name = "foreign_sell_value", precision = 18, scale = 2)
    private BigDecimal foreignSellValue;

    @Column(name = "net_buy_sell", precision = 18, scale = 2)
    private BigDecimal netBuySell;  // Mua/bán ròng

    @Column(name = "change_5_sessions", precision = 10, scale = 4)
    private BigDecimal change5Sessions;

    @Column(name = "change_10_sessions", precision = 10, scale = 4)
    private BigDecimal change10Sessions;

    @Column(name = "change_30_sessions", precision = 10, scale = 4)
    private BigDecimal change30Sessions;

    @Column(name = "stocks_up")
    private Integer stocksUp;  // Số CP tăng giá

    @Column(name = "stocks_down")
    private Integer stocksDown;  // Số CP giảm giá

    @Column(name = "stocks_unchanged")
    private Integer stocksUnchanged;  // Số CP đứng giá

    @Column(name = "trading_date")
    private LocalDate tradingDate;

    private LocalDateTime createdAt;
    private LocalDateTime updatedAt;
}
```

### 4.2. Search Request DTO

```java
@Data
public class MarketInfoSearchRequest {
    // Quick search
    private String keyword;  // Search cả indexCode và value

    // Advanced filters
    private String indexCode;
    private BigDecimal minValue;
    private BigDecimal maxValue;
    private BigDecimal changeNSessions;  // Biến động so với n phiên

    // Pagination (sẽ học ở Day 3)
    private Integer page = 0;
    private Integer size = 10;
    private String sortBy = "indexCode";
    private String sortDir = "asc";
}
```

### 4.3. Full Specification

```java
public class MarketInfoSpecification {

    /**
     * Quick search - tìm trong indexCode hoặc value
     */
    public static Specification<MarketInfo> quickSearch(String keyword) {
        return (root, query, cb) -> {
            if (keyword == null || keyword.trim().isEmpty()) {
                return null;
            }
            String pattern = "%" + keyword.toLowerCase() + "%";
            return cb.or(
                cb.like(cb.lower(root.get("indexCode")), pattern),
                cb.like(cb.function("CAST", String.class, root.get("value")), pattern)
            );
        };
    }

    /**
     * Filter theo index code chính xác
     */
    public static Specification<MarketInfo> hasIndexCode(String indexCode) {
        return (root, query, cb) -> {
            if (indexCode == null || indexCode.isEmpty()) {
                return null;
            }
            return cb.equal(root.get("indexCode"), indexCode);
        };
    }

    /**
     * Filter theo range giá trị
     */
    public static Specification<MarketInfo> valueBetween(BigDecimal min, BigDecimal max) {
        return (root, query, cb) -> {
            List<Predicate> predicates = new ArrayList<>();

            if (min != null) {
                predicates.add(cb.greaterThanOrEqualTo(root.get("value"), min));
            }
            if (max != null) {
                predicates.add(cb.lessThanOrEqualTo(root.get("value"), max));
            }

            return predicates.isEmpty() ? null :
                cb.and(predicates.toArray(new Predicate[0]));
        };
    }

    /**
     * Filter theo biến động
     */
    public static Specification<MarketInfo> hasChangeNSessions(BigDecimal change) {
        return (root, query, cb) -> {
            if (change == null) {
                return null;
            }
            // Tìm trong cả 3 trường change
            return cb.or(
                cb.equal(root.get("change5Sessions"), change),
                cb.equal(root.get("change10Sessions"), change),
                cb.equal(root.get("change30Sessions"), change)
            );
        };
    }

    /**
     * Build complete specification từ request
     */
    public static Specification<MarketInfo> fromRequest(MarketInfoSearchRequest request) {
        return Specification
            .where(quickSearch(request.getKeyword()))
            .and(hasIndexCode(request.getIndexCode()))
            .and(valueBetween(request.getMinValue(), request.getMaxValue()))
            .and(hasChangeNSessions(request.getChangeNSessions()));
    }
}
```

### 4.4. Service sử dụng Specification

```java
@Service
@RequiredArgsConstructor
@Slf4j
public class MarketInfoServiceImpl implements MarketInfoService {

    private final MarketInfoRepository repository;

    @Override
    public Page<MarketInfo> search(MarketInfoSearchRequest request) {
        log.info("Searching market info with request: {}", request);

        // Build specification
        Specification<MarketInfo> spec = MarketInfoSpecification.fromRequest(request);

        // Build pageable
        Sort sort = Sort.by(
            request.getSortDir().equalsIgnoreCase("desc") ?
                Sort.Direction.DESC : Sort.Direction.ASC,
            request.getSortBy()
        );
        Pageable pageable = PageRequest.of(request.getPage(), request.getSize(), sort);

        // Execute query
        return repository.findAll(spec, pageable);
    }

    @Override
    public MarketInfo findById(Long id) {
        return repository.findById(id)
            .orElseThrow(() -> new ResourceNotFoundException("MarketInfo", "id", id));
    }
}
```

## 5. Tổng kết Day 2

| Concept | C#/.NET | Java Spring |
|---------|---------|-------------|
| Entity | EF Entity | `@Entity` + `@Table` |
| DbContext | DbContext | Repository extends JpaRepository |
| LINQ Where | `.Where(x => ...)` | Query methods hoặc Specification |
| Dynamic Query | Expression Tree | `Specification<T>` |
| Include | `.Include()` | `@EntityGraph` hoặc `JOIN FETCH` |

### Key Takeaway
**JPA Specification** = **LINQ Expression Builder** trong .NET
- Dùng khi cần build query động từ nhiều điều kiện optional
- Combine với `Pageable` để pagination
- Clean, reusable, testable

## Navigation

- [← Day 1: Spring Boot Setup](./day-01-spring-boot-setup.md)
- [Day 3: REST API Pagination →](./day-03-rest-api-pagination.md)
