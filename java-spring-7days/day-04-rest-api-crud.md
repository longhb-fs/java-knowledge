# Day 4: REST API + CRUD + Validation

> **Mục tiêu:** Xây dựng Complete Product CRUD API với validation, error handling, DTO mapping, và Swagger docs.
> **Thời lượng:** ~6-8 giờ | **Prerequisites:** Hoàn thành Day 3 (JPA entities, repositories, Flyway)

---

## 1. REST Principles

### REST Là Gì?

REST (Representational State Transfer) là **kiến trúc thiết kế API** dựa trên giao thức HTTP. Không phải framework, không phải library -- chỉ là tập quy tắc để server và client giao tiếp hiệu quả.

**5 Constraints của REST:**

| # | Constraint | Ý Nghĩa |
|---|-----------|---------|
| 1 | **Client-Server** | Client và Server tách biệt, phát triển độc lập |
| 2 | **Stateless** | Mỗi request chứa đủ thông tin, server không lưu session |
| 3 | **Cacheable** | Response có thể được cache để tăng performance |
| 4 | **Uniform Interface** | Sử dụng URL + HTTP method thống nhất cho mọi resource |
| 5 | **Layered System** | Client không biết đang kết nối trực tiếp hay qua proxy |

### HTTP Methods

| Method | Mục Đích | Body? | Idempotent? | Safe? |
|--------|----------|-------|-------------|-------|
| **GET** | Lấy dữ liệu | Không | Có | Có |
| **POST** | Tạo mới resource | Có | Không | Không |
| **PUT** | Cập nhật toàn bộ resource | Có | Có | Không |
| **PATCH** | Cập nhật một phần resource | Có | **Không** | Không |
| **DELETE** | Xóa resource | Không | Có | Không |

> **QUAN TRỌNG:** PATCH **KHÔNG** idempotent. Ví dụ: PATCH với `{"stockQuantity": "increment"}` -- gọi 2 lần sẽ tăng 2 lần. PUT mới là idempotent vì gửi toàn bộ data thay thế.

### URL Design Best Practices

```
GET    /api/v1/products              -- Collection (danh sách)
GET    /api/v1/products/42           -- Single resource (1 item)
POST   /api/v1/products              -- Tạo mới
PUT    /api/v1/products/42           -- Cập nhật toàn bộ
PATCH  /api/v1/products/42           -- Cập nhật một phần
DELETE /api/v1/products/42           -- Xóa
GET    /api/v1/categories/5/products -- Nested resource
```

**Quy tắc:** Danh từ số nhiều (`/products`), lowercase + hyphen (`/order-items`), version trong URL (`/api/v1/`), không dùng động từ.

### HTTP Status Codes

| Code | Tên | Khi Nào Dùng |
|------|-----|-------------|
| **200** | OK | GET/PUT/PATCH thành công |
| **201** | Created | POST tạo resource thành công |
| **204** | No Content | DELETE thành công (không trả body) |
| **400** | Bad Request | Request sai format, thiếu field |
| **401** | Unauthorized | Chưa đăng nhập / token hết hạn |
| **403** | Forbidden | Đã đăng nhập nhưng không có quyền |
| **404** | Not Found | Resource không tồn tại |
| **409** | Conflict | Trùng dữ liệu (duplicate email, slug...) |
| **422** | Unprocessable Entity | Validation lỗi (dữ liệu đúng format nhưng sai logic) |
| **500** | Internal Server Error | Lỗi server (bug, DB down...) |

---

## 2. Controller Basics

### @RestController = @Controller + @ResponseBody

```java
// @Controller trả về VIEW (HTML) -- dùng cho web MVC
// @RestController trả về DATA (JSON) -- dùng cho REST API
@RestController
@RequestMapping("/api/v1/products")   // Base URL cho mọi method
@RequiredArgsConstructor              // Lombok: auto constructor injection
public class ProductController {
    private final ProductService productService;

    @GetMapping              // GET /api/v1/products
    @GetMapping("/{id}")     // GET /api/v1/products/42
    @PostMapping             // POST /api/v1/products
    @PutMapping("/{id}")     // PUT /api/v1/products/42
    @PatchMapping("/{id}")   // PATCH /api/v1/products/42
    @DeleteMapping("/{id}")  // DELETE /api/v1/products/42
}
```

### Parameter Annotations

```java
@GetMapping("/{id}")
public ProductResponse getById(@PathVariable Long id) { ... }
// GET /api/v1/products/42 → id = 42

@GetMapping
public List<ProductResponse> search(@RequestParam(required = false) String keyword) { ... }
// GET /api/v1/products?keyword=laptop → keyword = "laptop"

@PostMapping
public ProductResponse create(@RequestBody ProductCreateRequest request) { ... }
// POST body JSON → bind vào object

@GetMapping
public Page<ProductResponse> getAll(@ModelAttribute ProductFilterRequest filter) { ... }
// GET ?keyword=laptop&minPrice=100 → bind tất cả query params vào object
```

### ResponseEntity -- Kiểm Soát HTTP Response

```java
return ResponseEntity.ok(data);                                   // 200 OK
return ResponseEntity.status(HttpStatus.CREATED).body(data);      // 201 Created
return ResponseEntity.noContent().build();                         // 204 No Content
return ResponseEntity.notFound().build();                          // 404 Not Found
```

---

## 3. DTOs (Data Transfer Objects)

> **Tại sao cần DTO?** Không bao giờ trả Entity trực tiếp. DTO giúp: ẩn thông tin nhạy cảm, định dạng data cho client, tách DB logic khỏi API contract, validation riêng cho từng operation.

### Request DTOs

```java
// === ProductCreateRequest ===
public class ProductCreateRequest {
    @NotBlank(message = "Product name is required")
    @Size(max = 200)
    private String name;

    @Size(max = 250)
    private String slug; // auto-generate từ name nếu để trống

    private String description;

    @NotNull(message = "Price is required")
    @DecimalMin(value = "0.01", message = "Price must be at least 0.01")
    private BigDecimal price;

    private BigDecimal originalPrice;

    @Min(value = 0, message = "Stock quantity cannot be negative")
    private int stockQuantity = 0;

    private boolean active = true;
    private boolean featured = false;

    @NotNull(message = "Category is required")
    private Long categoryId;

    private List<String> imageUrls;
}

// === ProductUpdateRequest (dùng wrapper types để phân biệt "không gửi" vs "gửi null") ===
public class ProductUpdateRequest {
    @Size(max = 200)
    private String name;          // optional
    @Size(max = 250)
    private String slug;
    private String description;
    @DecimalMin("0.01")
    private BigDecimal price;
    private BigDecimal originalPrice;
    @Min(0)
    private Integer stockQuantity; // Integer, không phải int!
    private Boolean active;        // Boolean, không phải boolean!
    private Boolean featured;
    private Long categoryId;
    private List<String> imageUrls;
}

// === ProductFilterRequest (cho search/filter) ===
public class ProductFilterRequest {
    private String keyword;
    private Long categoryId;
    private BigDecimal minPrice;
    private BigDecimal maxPrice;
    private Boolean active;
    private Boolean featured;
}
```

> **Lưu ý:** `ProductUpdateRequest` dùng **wrapper types** (`Integer`, `Boolean`) vì primitive luôn có giá trị mặc định (0, false) -- không phân biệt "client gửi false" vs "client không gửi field này".

### Response DTOs

```java
// === ProductResponse ===
public class ProductResponse {
    private Long id;
    private String name;
    private String slug;
    private String description;
    private BigDecimal price;
    private BigDecimal originalPrice;
    private int stockQuantity;
    private boolean active;
    private boolean featured;
    private String categoryName;  // chỉ trả tên, không trả toàn bộ category
    private Long categoryId;
    private List<ProductImageResponse> images;
    private Instant createdAt;
    private Instant updatedAt;
}

// === ProductImageResponse ===
public class ProductImageResponse {
    private Long id;
    private String imageUrl;
    private boolean primary;
    private int sortOrder;
}

// === CategoryResponse ===
public class CategoryResponse {
    private Long id;
    private String name;
    private String slug;
    private String description;
    private String imageUrl;
    private boolean active;
    private int sortOrder;
    private Long parentId;
    private String parentName;
    private int productCount;
}

// === ApiResponse<T> -- Wrapper chung ===
public class ApiResponse<T> {
    private boolean success;
    private String message;
    private T data;
    private Instant timestamp = Instant.now();

    public static <T> ApiResponse<T> success(T data) { ... }
    public static <T> ApiResponse<T> success(String message, T data) { ... }
    public static <T> ApiResponse<T> error(String message) { ... }
}

// === PagedResponse<T> -- Cho phân trang ===
public class PagedResponse<T> {
    private List<T> content;
    private int page;
    private int size;
    private long totalElements;
    private int totalPages;
    private boolean last;

    public static <T> PagedResponse<T> from(Page<T> springPage) { ... }
}
```

---

## 4. SlugUtils -- Tạo Slug Từ Tiếng Việt

Slug là phiên bản URL-friendly của tên: "Áo Thun Nam Cao Cấp" → `ao-thun-nam-cao-cap`

```java
public class SlugUtils {
    private static final Pattern DIACRITICS = Pattern.compile("\\p{InCombiningDiacriticalMarks}+");
    private static final Pattern NON_ALPHANUMERIC = Pattern.compile("[^a-z0-9\\s-]");
    private static final Pattern WHITESPACE = Pattern.compile("[\\s]+");
    private static final Pattern MULTIPLE_HYPHENS = Pattern.compile("-{2,}");

    public static String removeDiacritics(String input) {
        if (input == null) return null;
        String normalized = Normalizer.normalize(input, Normalizer.Form.NFD);
        normalized = normalized.replace('\u0111', 'd').replace('\u0110', 'D');
        return DIACRITICS.matcher(normalized).replaceAll("");
    }

    public static String toSlug(String input) {
        if (input == null || input.isBlank()) return "";
        String slug = removeDiacritics(input);
        slug = slug.toLowerCase();
        slug = NON_ALPHANUMERIC.matcher(slug).replaceAll("");
        slug = WHITESPACE.matcher(slug).replaceAll("-");
        slug = MULTIPLE_HYPHENS.matcher(slug).replaceAll("-");
        slug = slug.replaceAll("^-|-$", "");
        return slug;
    }
}
```

```java
SlugUtils.toSlug("Ao Thun Nam Cao Cap");       // → "ao-thun-nam-cao-cap"
SlugUtils.toSlug("iPhone 15 Pro Max - 256GB");  // → "iphone-15-pro-max-256gb"
```

---

## 5. MapStruct -- DTO Mapping Tự Động

### MapStruct Là Gì?

MapStruct là **code generator** -- tự động tạo code chuyển đổi Entity ↔ DTO tại **compile time**. Không dùng reflection nên nhanh hơn ModelMapper. Entity 15 fields → MapStruct generate hết, type-safe, báo lỗi lúc compile.

### pom.xml -- Thứ Tự Annotation Processor QUAN TRỌNG

```xml
<annotationProcessorPaths>
    <!-- THỨ TỰ: Lombok TRƯỚC → binding → MapStruct SAU -->
    <path>
        <groupId>org.projectlombok</groupId>
        <artifactId>lombok</artifactId>
        <version>${lombok.version}</version>
    </path>
    <path>
        <groupId>org.projectlombok</groupId>
        <artifactId>lombok-mapstruct-binding</artifactId>
        <version>0.2.0</version>
    </path>
    <path>
        <groupId>org.mapstruct</groupId>
        <artifactId>mapstruct-processor</artifactId>
        <version>1.5.5.Final</version>
    </path>
</annotationProcessorPaths>
```

> **CẢNH BÁO:** Đặt MapStruct trước Lombok → MapStruct không thấy getter/setter → build fail. Luôn để **Lombok trước**.

### ProductMapper

```java
@Mapper(componentModel = "spring",
        nullValuePropertyMappingStrategy = NullValuePropertyMappingStrategy.IGNORE)
public interface ProductMapper {

    // Entity ← Request
    @Mapping(target = "id", ignore = true)
    @Mapping(target = "slug", ignore = true)       // set trong service
    @Mapping(target = "category", ignore = true)   // set trong service
    @Mapping(target = "images", ignore = true)
    @Mapping(target = "createdAt", ignore = true)
    @Mapping(target = "updatedAt", ignore = true)
    Product toEntity(ProductCreateRequest request);

    // Update entity -- chỉ update fields không null (nhờ IGNORE strategy)
    @Mapping(target = "id", ignore = true)
    @Mapping(target = "category", ignore = true)
    @Mapping(target = "images", ignore = true)
    @Mapping(target = "createdAt", ignore = true)
    @Mapping(target = "updatedAt", ignore = true)
    void updateEntity(ProductUpdateRequest request, @MappingTarget Product product);

    // Response ← Entity
    @Mapping(source = "category.name", target = "categoryName")
    @Mapping(source = "category.id", target = "categoryId")
    ProductResponse toResponse(Product product);

    List<ProductResponse> toResponseList(List<Product> products);
    ProductImageResponse toImageResponse(ProductImage image);
}
```

**Key concepts:**
- `componentModel = "spring"` → tạo Spring Bean, inject được
- `NullValuePropertyMappingStrategy.IGNORE` → field null không ghi đè entity
- `@MappingTarget` → map vào entity đã tồn tại (cho update)
- `source = "category.name"` → tự động gọi `product.getCategory().getName()`

### CategoryMapper

```java
@Mapper(componentModel = "spring")
public interface CategoryMapper {
    @Mapping(target = "parentName", source = "parent.name")
    @Mapping(target = "parentId", source = "parent.id")
    @Mapping(target = "productCount", ignore = true)
    CategoryResponse toResponse(Category category);
    List<CategoryResponse> toResponseList(List<Category> categories);
}
```

---

## 6. Product Service Layer

```java
@Service
@RequiredArgsConstructor
@Transactional(readOnly = true)
@Slf4j
public class ProductService {
    private final ProductRepository productRepository;
    private final CategoryRepository categoryRepository;
    private final ProductMapper productMapper;

    // GET ALL -- pagination + dynamic filter (Specification pattern từ Day 3)
    public PagedResponse<ProductResponse> getAll(ProductFilterRequest filter, Pageable pageable) {
        Specification<Product> spec = Specification.where(null);
        if (filter.getKeyword() != null && !filter.getKeyword().isBlank())
            spec = spec.and(ProductSpecification.hasKeyword(filter.getKeyword()));
        if (filter.getCategoryId() != null)
            spec = spec.and(ProductSpecification.hasCategory(filter.getCategoryId()));
        if (filter.getMinPrice() != null)
            spec = spec.and(ProductSpecification.hasPriceGreaterThan(filter.getMinPrice()));
        if (filter.getMaxPrice() != null)
            spec = spec.and(ProductSpecification.hasPriceLessThan(filter.getMaxPrice()));
        if (filter.getActive() != null)
            spec = spec.and(ProductSpecification.isActive(filter.getActive()));
        if (filter.getFeatured() != null)
            spec = spec.and(ProductSpecification.isFeatured(filter.getFeatured()));

        Page<Product> page = productRepository.findAll(spec, pageable);
        return PagedResponse.from(page.map(productMapper::toResponse));
    }

    // GET BY ID
    public ProductResponse getById(Long id) {
        Product product = productRepository.findById(id)
                .orElseThrow(() -> new ResourceNotFoundException("Product", "id", id));
        return productMapper.toResponse(product);
    }

    // GET BY SLUG
    public ProductResponse getBySlug(String slug) {
        Product product = productRepository.findBySlug(slug)
                .orElseThrow(() -> new ResourceNotFoundException("Product", "slug", slug));
        return productMapper.toResponse(product);
    }

    // CREATE
    @Transactional
    public ProductResponse create(ProductCreateRequest request) {
        // 1. Tìm category
        Category category = categoryRepository.findById(request.getCategoryId())
                .orElseThrow(() -> new ResourceNotFoundException("Category", "id", request.getCategoryId()));
        // 2. Generate slug
        String slug = (request.getSlug() != null && !request.getSlug().isBlank())
                ? SlugUtils.toSlug(request.getSlug())
                : SlugUtils.toSlug(request.getName());
        // 3. Check slug unique
        if (productRepository.existsBySlug(slug))
            throw new DuplicateResourceException("Product", "slug", slug);
        // 4. Map → entity + set fields
        Product product = productMapper.toEntity(request);
        product.setSlug(slug);
        product.setCategory(category);
        // 5. Add images
        if (request.getImageUrls() != null) {
            for (int i = 0; i < request.getImageUrls().size(); i++) {
                ProductImage image = new ProductImage();
                image.setImageUrl(request.getImageUrls().get(i));
                image.setPrimary(i == 0);
                image.setSortOrder(i);
                product.addImage(image);
            }
        }
        // 6. Save
        Product saved = productRepository.save(product);
        log.info("Created product: id={}, slug={}", saved.getId(), slug);
        return productMapper.toResponse(saved);
    }

    // UPDATE
    @Transactional
    public ProductResponse update(Long id, ProductUpdateRequest request) {
        Product product = productRepository.findById(id)
                .orElseThrow(() -> new ResourceNotFoundException("Product", "id", id));
        // Update category nếu thay đổi
        if (request.getCategoryId() != null
                && !request.getCategoryId().equals(product.getCategory().getId())) {
            Category category = categoryRepository.findById(request.getCategoryId())
                    .orElseThrow(() -> new ResourceNotFoundException("Category", "id", request.getCategoryId()));
            product.setCategory(category);
        }
        // Update slug nếu thay đổi
        if (request.getSlug() != null && !request.getSlug().isBlank()) {
            String newSlug = SlugUtils.toSlug(request.getSlug());
            if (!newSlug.equals(product.getSlug()) && productRepository.existsBySlug(newSlug))
                throw new DuplicateResourceException("Product", "slug", newSlug);
            product.setSlug(newSlug);
        }
        // Map các field còn lại (chỉ update non-null)
        productMapper.updateEntity(request, product);
        Product saved = productRepository.save(product);
        return productMapper.toResponse(saved);
    }

    // DELETE
    @Transactional
    public void delete(Long id) {
        if (!productRepository.existsById(id))
            throw new ResourceNotFoundException("Product", "id", id);
        productRepository.deleteById(id);
        log.info("Deleted product: id={}", id);
    }
}
```

**Key points:** `@Transactional(readOnly = true)` trên class → mặc định read-only (Hibernate tối ưu). `@Transactional` trên method ghi đè → cho phép write.

---

## 7. Product Controller

```java
@RestController
@RequestMapping("/api/v1/products")
@RequiredArgsConstructor
@Tag(name = "Products", description = "Product management APIs")
public class ProductController {
    private final ProductService productService;

    @GetMapping
    @Operation(summary = "Get all products with filter + pagination")
    public ResponseEntity<ApiResponse<PagedResponse<ProductResponse>>> getAll(
            @ModelAttribute ProductFilterRequest filter,
            @PageableDefault(size = 20, sort = "createdAt", direction = Sort.Direction.DESC)
            Pageable pageable) {
        return ResponseEntity.ok(ApiResponse.success(productService.getAll(filter, pageable)));
    }

    @GetMapping("/{id}")
    @Operation(summary = "Get product by ID")
    public ResponseEntity<ApiResponse<ProductResponse>> getById(@PathVariable Long id) {
        return ResponseEntity.ok(ApiResponse.success(productService.getById(id)));
    }

    @GetMapping("/slug/{slug}")
    @Operation(summary = "Get product by slug")
    public ResponseEntity<ApiResponse<ProductResponse>> getBySlug(@PathVariable String slug) {
        return ResponseEntity.ok(ApiResponse.success(productService.getBySlug(slug)));
    }

    @PostMapping
    @Operation(summary = "Create a new product")
    public ResponseEntity<ApiResponse<ProductResponse>> create(
            @Valid @RequestBody ProductCreateRequest request) {
        return ResponseEntity.status(HttpStatus.CREATED)
                .body(ApiResponse.success("Product created", productService.create(request)));
    }

    @PutMapping("/{id}")
    @Operation(summary = "Update product")
    public ResponseEntity<ApiResponse<ProductResponse>> update(
            @PathVariable Long id, @Valid @RequestBody ProductUpdateRequest request) {
        return ResponseEntity.ok(ApiResponse.success("Product updated",
                productService.update(id, request)));
    }

    @DeleteMapping("/{id}")
    @Operation(summary = "Delete product")
    public ResponseEntity<Void> delete(@PathVariable Long id) {
        productService.delete(id);
        return ResponseEntity.noContent().build(); // 204
    }
}
```

**Giải thích:** `@Valid` trước `@RequestBody` → kích hoạt Bean Validation. `@PageableDefault` → giá trị mặc định. DELETE → `204 No Content`. POST → `201 Created`.

---

## 8. Bean Validation

### Built-in Annotations

| Annotation | Áp Dụng Cho | Ý Nghĩa |
|-----------|------------|---------|
| `@NotNull` | Mọi kiểu | Không được null |
| `@NotBlank` | String | Không null, không rỗng, không chỉ whitespace |
| `@NotEmpty` | String, Collection | Không null và không rỗng |
| `@Size(min, max)` | String, Collection | Độ dài trong khoảng |
| `@Min` / `@Max` | Number | Giá trị tối thiểu / tối đa |
| `@DecimalMin` / `@DecimalMax` | BigDecimal | Hỗ trợ số thập phân |
| `@Email` | String | Định dạng email |
| `@Pattern(regexp)` | String | Phải match regex |
| `@Past` / `@Future` | Date/Time | Quá khứ / tương lai |
| `@Positive` / `@PositiveOrZero` | Number | > 0 / >= 0 |

### @Valid -- Kích Hoạt Validation

```java
@PostMapping
public ResponseEntity<?> create(@Valid @RequestBody ProductCreateRequest request) {
    // Validation fail → Spring throw MethodArgumentNotValidException tự động
    return ResponseEntity.ok(service.create(request));
}
```

### Custom Validator -- @UniqueEmail

**Bước 1: Annotation**

```java
@Target({ElementType.FIELD})
@Retention(RetentionPolicy.RUNTIME)
@Constraint(validatedBy = UniqueEmailValidator.class)
@Documented
public @interface UniqueEmail {
    String message() default "Email already exists";
    Class<?>[] groups() default {};
    Class<? extends Payload>[] payload() default {};
}
```

**Bước 2: Validator**

```java
@RequiredArgsConstructor
public class UniqueEmailValidator implements ConstraintValidator<UniqueEmail, String> {
    private final UserRepository userRepository;

    @Override
    public boolean isValid(String email, ConstraintValidatorContext context) {
        if (email == null || email.isBlank()) return true; // @NotBlank xử lý
        return !userRepository.existsByEmail(email);
    }
}
```

**Bước 3: Sử dụng**

```java
public class UserRegisterRequest {
    @NotBlank @Email @UniqueEmail
    private String email;

    @NotBlank @Size(min = 8, max = 100)
    private String password;

    @NotBlank @Size(max = 100)
    private String fullName;

    @Pattern(regexp = "^\\+?[0-9]{10,15}$", message = "Invalid phone number")
    private String phone;
}
```

> **Phone regex:** Dùng `^\\+?[0-9]{10,15}$` -- đơn giản, chấp nhận quốc tế. Không nên regex phức tạp quá vì mỗi nước khác nhau.

---

## 9. Global Exception Handler

### Custom Exceptions

```java
// 404
public class ResourceNotFoundException extends RuntimeException {
    public ResourceNotFoundException(String resourceName, String fieldName, Object fieldValue) {
        super(String.format("%s not found with %s: '%s'", resourceName, fieldName, fieldValue));
    }
}
// 409
public class DuplicateResourceException extends RuntimeException {
    public DuplicateResourceException(String resourceName, String fieldName, Object fieldValue) {
        super(String.format("%s already exists with %s: '%s'", resourceName, fieldName, fieldValue));
    }
}
// 400
public class BadRequestException extends RuntimeException {
    public BadRequestException(String message) { super(message); }
}
```

### GlobalExceptionHandler

```java
@RestControllerAdvice
@Slf4j
public class GlobalExceptionHandler {

    @ExceptionHandler(ResourceNotFoundException.class)  // → 404
    public ResponseEntity<ApiResponse<Void>> handleNotFound(ResourceNotFoundException ex) {
        log.warn("Resource not found: {}", ex.getMessage());
        return ResponseEntity.status(HttpStatus.NOT_FOUND).body(ApiResponse.error(ex.getMessage()));
    }

    @ExceptionHandler(DuplicateResourceException.class)  // → 409
    public ResponseEntity<ApiResponse<Void>> handleDuplicate(DuplicateResourceException ex) {
        log.warn("Duplicate: {}", ex.getMessage());
        return ResponseEntity.status(HttpStatus.CONFLICT).body(ApiResponse.error(ex.getMessage()));
    }

    @ExceptionHandler(BadRequestException.class)  // → 400
    public ResponseEntity<ApiResponse<Void>> handleBadRequest(BadRequestException ex) {
        return ResponseEntity.status(HttpStatus.BAD_REQUEST).body(ApiResponse.error(ex.getMessage()));
    }

    @ExceptionHandler(MethodArgumentNotValidException.class)  // → 422 (validation @RequestBody)
    public ResponseEntity<ApiResponse<Map<String, String>>> handleValidation(
            MethodArgumentNotValidException ex) {
        Map<String, String> errors = new HashMap<>();
        ex.getBindingResult().getFieldErrors()
                .forEach(e -> errors.put(e.getField(), e.getDefaultMessage()));
        log.warn("Validation failed: {}", errors);
        ApiResponse<Map<String, String>> response = ApiResponse.error("Validation failed");
        response.setData(errors);
        return ResponseEntity.status(HttpStatus.UNPROCESSABLE_ENTITY).body(response);
    }

    @ExceptionHandler(ConstraintViolationException.class)  // → 422 (validation @PathVariable)
    public ResponseEntity<ApiResponse<Map<String, String>>> handleConstraint(
            ConstraintViolationException ex) {
        Map<String, String> errors = new HashMap<>();
        ex.getConstraintViolations()
                .forEach(v -> errors.put(v.getPropertyPath().toString(), v.getMessage()));
        ApiResponse<Map<String, String>> response = ApiResponse.error("Validation failed");
        response.setData(errors);
        return ResponseEntity.status(HttpStatus.UNPROCESSABLE_ENTITY).body(response);
    }

    @ExceptionHandler(Exception.class)  // → 500 (KHÔNG trả chi tiết cho client!)
    public ResponseEntity<ApiResponse<Void>> handleGeneric(Exception ex) {
        log.error("Unexpected error", ex); // Log FULL stack trace ở server
        return ResponseEntity.status(HttpStatus.INTERNAL_SERVER_ERROR)
                .body(ApiResponse.error("An unexpected error occurred."));
    }
}
```

**Luồng xử lý:**
```
Request → Controller (@Valid) → fail? → 422
                               → pass → Service → ResourceNotFound → 404
                                                 → Duplicate → 409
                                                 → Unknown → 500 (log, trả message chung)
```

---

## 10. Swagger / OpenAPI Setup

### Dependency (đã có từ Day 1)

```xml
<dependency>
    <groupId>org.springdoc</groupId>
    <artifactId>springdoc-openapi-starter-webmvc-ui</artifactId>
    <version>2.6.0</version>
</dependency>
```

### Config

```java
@Configuration
public class OpenApiConfig {
    @Bean
    public OpenAPI customOpenAPI() {
        return new OpenAPI()
                .info(new Info()
                        .title("E-Commerce API")
                        .version("1.0.0")
                        .description("Spring Boot E-Commerce REST API"));
    }
}
```

**Truy cập:** `http://localhost:8080/swagger-ui.html`

### Annotations

| Annotation | Vị Trí | Mục Đích |
|-----------|--------|---------|
| `@Tag(name, description)` | Controller class | Nhóm API |
| `@Operation(summary)` | Method | Mô tả endpoint |
| `@Parameter(description)` | Method param | Mô tả parameter |
| `@Schema(description, example)` | DTO field | Mô tả field |

```java
public class ProductCreateRequest {
    @Schema(description = "Product name", example = "iPhone 15 Pro Max")
    @NotBlank
    private String name;

    @Schema(description = "Price in VND", example = "29990000")
    @NotNull @DecimalMin("0.01")
    private BigDecimal price;
}
```

---

## 11. Test Với curl / Postman

### Create Product

```bash
curl -X POST http://localhost:8080/api/v1/products \
  -H "Content-Type: application/json" \
  -d '{"name":"Ao Thun Nam","price":299000,"stockQuantity":100,"categoryId":1,
       "imageUrls":["https://example.com/img1.jpg"]}'
```

**Response 201:**
```json
{"success":true,"message":"Product created","data":{"id":1,"name":"Ao Thun Nam",
"slug":"ao-thun-nam","price":299000,"categoryName":"Ao Nam","categoryId":1}}
```

### List + Filter + Pagination

```bash
curl "http://localhost:8080/api/v1/products?keyword=ao&minPrice=100000&page=0&size=10&sort=price,asc"
```

### Get By ID / Slug

```bash
curl http://localhost:8080/api/v1/products/1
curl http://localhost:8080/api/v1/products/slug/ao-thun-nam
```

### Update

```bash
curl -X PUT http://localhost:8080/api/v1/products/1 \
  -H "Content-Type: application/json" \
  -d '{"price":259000,"stockQuantity":80}'
```

### Delete

```bash
curl -X DELETE http://localhost:8080/api/v1/products/1
# → 204 No Content
```

### Test Validation Error (422)

```bash
curl -X POST http://localhost:8080/api/v1/products \
  -H "Content-Type: application/json" \
  -d '{"name":"","price":-5}'
```

```json
{"success":false,"message":"Validation failed",
 "data":{"name":"Product name is required","price":"Price must be at least 0.01",
         "categoryId":"Category is required"}}
```

### Test Not Found (404)

```bash
curl http://localhost:8080/api/v1/products/99999
# → {"success":false,"message":"Product not found with id: '99999'"}
```

---

## Tổng Kết Day 4

| Chủ Đề | Đã Học |
|--------|--------|
| REST Principles | 5 constraints, HTTP methods, URL design, status codes |
| Controller | @RestController, @RequestMapping, parameter annotations |
| DTOs | Request/Response separation, ApiResponse wrapper, PagedResponse |
| SlugUtils | Chuyển tiếng Việt có dấu thành slug URL-friendly |
| MapStruct | Compile-time mapping, @Mapper, @MappingTarget, IGNORE strategy |
| Service Layer | Business logic, @Transactional, Specification filtering |
| Bean Validation | Built-in annotations, @Valid, custom validator |
| Exception Handler | @RestControllerAdvice, custom exceptions, error responses |
| Swagger/OpenAPI | SpringDoc setup, annotations, Swagger UI |

**Output:** Complete Product CRUD API với validation, error handling, và Swagger docs.

**Ngày tiếp theo:** Day 5 -- Security + JWT + User Registration.
