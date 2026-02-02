# Ngày 11: REST Controller & HTTP Methods

## Mục tiêu hôm nay
- Hiểu REST principles và HTTP methods
- Sử dụng @RestController, @RequestMapping
- Xử lý Path Variables, Query Parameters, Request Body
- ResponseEntity và HTTP Status Codes
- Exception handling cơ bản

---

## 1. REST PRINCIPLES

### 1.1. REST là gì?

**REST (Representational State Transfer)** = Kiến trúc cho web services

```
┌─────────────────────────────────────────────────────────────┐
│                    REST PRINCIPLES                           │
├─────────────────────────────────────────────────────────────┤
│  1. Client-Server     : Tách biệt UI và Data                │
│  2. Stateless         : Mỗi request chứa đủ thông tin       │
│  3. Cacheable         : Response có thể cache               │
│  4. Uniform Interface : URL + HTTP methods chuẩn            │
│  5. Layered System    : Client không biết server layers     │
│  6. Code on Demand    : (Optional) Server gửi code          │
└─────────────────────────────────────────────────────────────┘
```

### 1.2. HTTP Methods (CRUD mapping)

| HTTP Method | CRUD | Mô tả | Idempotent |
|-------------|------|-------|------------|
| `GET` | Read | Lấy resource | ✅ Yes |
| `POST` | Create | Tạo resource mới | ❌ No |
| `PUT` | Update | Cập nhật toàn bộ resource | ✅ Yes |
| `PATCH` | Update | Cập nhật một phần | ✅ Yes |
| `DELETE` | Delete | Xóa resource | ✅ Yes |

### 1.3. RESTful URL Design

```
RESOURCE-BASED URLs (danh từ, số nhiều):

✅ ĐÚNG:
GET    /api/products          → Lấy danh sách products
GET    /api/products/123      → Lấy product ID 123
POST   /api/products          → Tạo product mới
PUT    /api/products/123      → Update product 123
DELETE /api/products/123      → Xóa product 123

GET    /api/products/123/reviews     → Reviews của product 123
POST   /api/products/123/reviews     → Tạo review cho product 123

❌ SAI (dùng động từ):
GET    /api/getProducts
POST   /api/createProduct
GET    /api/deleteProduct/123
```

---

## 2. @RESTCONTROLLER

### 2.1. Annotations cơ bản

```java
package com.example.ecommerce.controller;

import org.springframework.web.bind.annotation.*;
import org.springframework.http.ResponseEntity;
import lombok.RequiredArgsConstructor;

@RestController  // = @Controller + @ResponseBody
@RequestMapping("/api/products")  // Base path
@RequiredArgsConstructor
public class ProductController {

    private final ProductService productService;

    // Các endpoints ở đây...
}
```

### 2.2. Mapping Annotations

```java
@RestController
@RequestMapping("/api/products")
public class ProductController {

    // ─── GET requests ───

    // GET /api/products
    @GetMapping
    public List<ProductResponse> getAllProducts() {
        return productService.getAll();
    }

    // GET /api/products/123
    @GetMapping("/{id}")
    public ProductResponse getProduct(@PathVariable Long id) {
        return productService.getById(id);
    }

    // GET /api/products/slug/iphone-15
    @GetMapping("/slug/{slug}")
    public ProductResponse getBySlug(@PathVariable String slug) {
        return productService.getBySlug(slug);
    }

    // ─── POST requests ───

    // POST /api/products
    @PostMapping
    public ProductResponse createProduct(@RequestBody ProductCreateRequest request) {
        return productService.create(request);
    }

    // ─── PUT requests ───

    // PUT /api/products/123
    @PutMapping("/{id}")
    public ProductResponse updateProduct(
            @PathVariable Long id,
            @RequestBody ProductUpdateRequest request) {
        return productService.update(id, request);
    }

    // ─── PATCH requests ───

    // PATCH /api/products/123
    @PatchMapping("/{id}")
    public ProductResponse patchProduct(
            @PathVariable Long id,
            @RequestBody Map<String, Object> updates) {
        return productService.patch(id, updates);
    }

    // ─── DELETE requests ───

    // DELETE /api/products/123
    @DeleteMapping("/{id}")
    public void deleteProduct(@PathVariable Long id) {
        productService.delete(id);
    }
}
```

---

## 3. REQUEST PARAMETERS

### 3.1. @PathVariable

```java
// URL: /api/products/123
@GetMapping("/{id}")
public ProductResponse getProduct(@PathVariable Long id) {
    return productService.getById(id);
}

// URL: /api/products/123/reviews/456
@GetMapping("/{productId}/reviews/{reviewId}")
public ReviewResponse getReview(
        @PathVariable Long productId,
        @PathVariable Long reviewId) {
    return reviewService.getByProductAndId(productId, reviewId);
}

// Tên khác với parameter
@GetMapping("/{product-id}")
public ProductResponse getProduct(
        @PathVariable("product-id") Long productId) {
    return productService.getById(productId);
}

// Optional path variable
@GetMapping({"/", "/{id}"})
public Object getProducts(@PathVariable(required = false) Long id) {
    if (id != null) {
        return productService.getById(id);
    }
    return productService.getAll();
}
```

### 3.2. @RequestParam

```java
// URL: /api/products?category=1&minPrice=100000
@GetMapping
public List<ProductResponse> searchProducts(
        @RequestParam(required = false) Long category,
        @RequestParam(required = false) BigDecimal minPrice,
        @RequestParam(required = false) BigDecimal maxPrice,
        @RequestParam(defaultValue = "") String keyword,
        @RequestParam(defaultValue = "0") int page,
        @RequestParam(defaultValue = "10") int size,
        @RequestParam(defaultValue = "createdAt") String sortBy,
        @RequestParam(defaultValue = "desc") String sortDir
) {
    return productService.search(category, minPrice, maxPrice, keyword, page, size, sortBy, sortDir);
}

// List parameter: /api/products?ids=1,2,3
@GetMapping("/batch")
public List<ProductResponse> getByIds(@RequestParam List<Long> ids) {
    return productService.getByIds(ids);
}

// Map tất cả params
@GetMapping("/search")
public List<ProductResponse> search(@RequestParam Map<String, String> params) {
    return productService.searchByParams(params);
}
```

### 3.3. @RequestBody

```java
// POST với JSON body
@PostMapping
public ProductResponse createProduct(
        @RequestBody ProductCreateRequest request) {
    return productService.create(request);
}

// DTO class
@Data
public class ProductCreateRequest {
    private String name;
    private String sku;
    private BigDecimal price;
    private String description;
    private Long categoryId;
}

// Request JSON:
// {
//   "name": "iPhone 15",
//   "sku": "IP15-128",
//   "price": 22990000,
//   "description": "Điện thoại Apple",
//   "categoryId": 1
// }
```

### 3.4. @RequestHeader

```java
@GetMapping
public ProductResponse getProduct(
        @RequestHeader("Authorization") String authHeader,
        @RequestHeader(value = "X-Request-Id", required = false) String requestId,
        @RequestHeader(value = "Accept-Language", defaultValue = "vi") String language
) {
    // authHeader = "Bearer eyJhbGc..."
    // requestId = "abc-123" hoặc null
    // language = "vi" hoặc "en"
}

// Lấy tất cả headers
@GetMapping
public void logHeaders(@RequestHeader Map<String, String> headers) {
    headers.forEach((key, value) -> log.info("{}: {}", key, value));
}
```

---

## 4. RESPONSE ENTITY

### 4.1. Cơ bản

```java
@RestController
@RequestMapping("/api/products")
public class ProductController {

    // Trả về với status code cụ thể
    @GetMapping("/{id}")
    public ResponseEntity<ProductResponse> getProduct(@PathVariable Long id) {
        ProductResponse product = productService.getById(id);
        return ResponseEntity.ok(product);  // 200 OK
    }

    // Created với Location header
    @PostMapping
    public ResponseEntity<ProductResponse> createProduct(
            @RequestBody ProductCreateRequest request) {
        ProductResponse created = productService.create(request);

        URI location = URI.create("/api/products/" + created.getId());

        return ResponseEntity
                .created(location)  // 201 Created + Location header
                .body(created);
    }

    // No Content
    @DeleteMapping("/{id}")
    public ResponseEntity<Void> deleteProduct(@PathVariable Long id) {
        productService.delete(id);
        return ResponseEntity.noContent().build();  // 204 No Content
    }

    // Custom status + headers
    @PutMapping("/{id}")
    public ResponseEntity<ProductResponse> updateProduct(
            @PathVariable Long id,
            @RequestBody ProductUpdateRequest request) {
        ProductResponse updated = productService.update(id, request);

        return ResponseEntity
                .status(HttpStatus.OK)
                .header("X-Updated-By", "admin")
                .header("X-Updated-At", Instant.now().toString())
                .body(updated);
    }
}
```

### 4.2. HTTP Status Codes thường dùng

```java
// ═══ 2xx Success ═══
ResponseEntity.ok(body);                    // 200 OK
ResponseEntity.created(uri).body(body);     // 201 Created
ResponseEntity.accepted().build();          // 202 Accepted
ResponseEntity.noContent().build();         // 204 No Content

// ═══ 3xx Redirection ═══
ResponseEntity.status(HttpStatus.MOVED_PERMANENTLY)
    .location(URI.create("/new-url"))
    .build();                               // 301 Moved Permanently

// ═══ 4xx Client Error ═══
ResponseEntity.badRequest().body(errors);   // 400 Bad Request
ResponseEntity.status(HttpStatus.UNAUTHORIZED).build();  // 401
ResponseEntity.status(HttpStatus.FORBIDDEN).build();     // 403
ResponseEntity.notFound().build();          // 404 Not Found
ResponseEntity.status(HttpStatus.CONFLICT).build();      // 409 Conflict

// ═══ 5xx Server Error ═══
ResponseEntity.internalServerError().build();  // 500 Internal Server Error
```

---

## 5. API RESPONSE WRAPPER

### 5.1. Chuẩn hóa response format

```java
package com.example.ecommerce.dto.response;

import com.fasterxml.jackson.annotation.JsonInclude;
import lombok.AllArgsConstructor;
import lombok.Builder;
import lombok.Data;
import lombok.NoArgsConstructor;

import java.time.Instant;

@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
@JsonInclude(JsonInclude.Include.NON_NULL)
public class ApiResponse<T> {

    private boolean success;
    private String message;
    private T data;
    private Object errors;

    @Builder.Default
    private Instant timestamp = Instant.now();

    // ─── Factory methods ───

    public static <T> ApiResponse<T> success(T data) {
        return ApiResponse.<T>builder()
                .success(true)
                .message("Thành công")
                .data(data)
                .build();
    }

    public static <T> ApiResponse<T> success(String message, T data) {
        return ApiResponse.<T>builder()
                .success(true)
                .message(message)
                .data(data)
                .build();
    }

    public static <T> ApiResponse<T> created(T data) {
        return ApiResponse.<T>builder()
                .success(true)
                .message("Tạo thành công")
                .data(data)
                .build();
    }

    public static <T> ApiResponse<T> error(String message) {
        return ApiResponse.<T>builder()
                .success(false)
                .message(message)
                .build();
    }

    public static <T> ApiResponse<T> error(String message, Object errors) {
        return ApiResponse.<T>builder()
                .success(false)
                .message(message)
                .errors(errors)
                .build();
    }
}
```

### 5.2. Sử dụng trong Controller

```java
@RestController
@RequestMapping("/api/products")
@RequiredArgsConstructor
public class ProductController {

    private final ProductService productService;

    @GetMapping
    public ApiResponse<PagedResponse<ProductResponse>> getProducts(
            @Valid ProductSearchRequest request) {
        PagedResponse<ProductResponse> data = productService.search(request);
        return ApiResponse.success(data);
    }

    @GetMapping("/{id}")
    public ApiResponse<ProductResponse> getProduct(@PathVariable Long id) {
        ProductResponse data = productService.getById(id);
        return ApiResponse.success(data);
    }

    @PostMapping
    public ResponseEntity<ApiResponse<ProductResponse>> createProduct(
            @Valid @RequestBody ProductCreateRequest request) {
        ProductResponse data = productService.create(request);
        return ResponseEntity
                .status(HttpStatus.CREATED)
                .body(ApiResponse.created(data));
    }

    @PutMapping("/{id}")
    public ApiResponse<ProductResponse> updateProduct(
            @PathVariable Long id,
            @Valid @RequestBody ProductUpdateRequest request) {
        ProductResponse data = productService.update(id, request);
        return ApiResponse.success("Cập nhật thành công", data);
    }

    @DeleteMapping("/{id}")
    public ApiResponse<Void> deleteProduct(@PathVariable Long id) {
        productService.delete(id);
        return ApiResponse.success("Xóa thành công", null);
    }
}
```

**JSON Response:**
```json
{
  "success": true,
  "message": "Thành công",
  "data": {
    "id": 1,
    "name": "iPhone 15",
    "price": 22990000
  },
  "timestamp": "2024-01-15T10:30:00Z"
}
```

---

## 6. EXCEPTION HANDLING

### 6.1. Custom Exceptions

```java
package com.example.ecommerce.exception;

import org.springframework.http.HttpStatus;
import org.springframework.web.bind.annotation.ResponseStatus;

@ResponseStatus(HttpStatus.NOT_FOUND)
public class ResourceNotFoundException extends RuntimeException {

    private String resourceName;
    private String fieldName;
    private Object fieldValue;

    public ResourceNotFoundException(String resourceName, String fieldName, Object fieldValue) {
        super(String.format("%s không tìm thấy với %s: '%s'", resourceName, fieldName, fieldValue));
        this.resourceName = resourceName;
        this.fieldName = fieldName;
        this.fieldValue = fieldValue;
    }
}
```

```java
@ResponseStatus(HttpStatus.BAD_REQUEST)
public class BadRequestException extends RuntimeException {
    public BadRequestException(String message) {
        super(message);
    }
}
```

```java
@ResponseStatus(HttpStatus.CONFLICT)
public class DuplicateResourceException extends RuntimeException {
    public DuplicateResourceException(String message) {
        super(message);
    }
}
```

### 6.2. Global Exception Handler

```java
package com.example.ecommerce.exception;

import com.example.ecommerce.dto.response.ApiResponse;
import lombok.extern.slf4j.Slf4j;
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.ExceptionHandler;
import org.springframework.web.bind.annotation.RestControllerAdvice;

@RestControllerAdvice
@Slf4j
public class GlobalExceptionHandler {

    @ExceptionHandler(ResourceNotFoundException.class)
    public ResponseEntity<ApiResponse<Void>> handleNotFound(ResourceNotFoundException ex) {
        log.warn("Resource not found: {}", ex.getMessage());
        return ResponseEntity
                .status(HttpStatus.NOT_FOUND)
                .body(ApiResponse.error(ex.getMessage()));
    }

    @ExceptionHandler(BadRequestException.class)
    public ResponseEntity<ApiResponse<Void>> handleBadRequest(BadRequestException ex) {
        log.warn("Bad request: {}", ex.getMessage());
        return ResponseEntity
                .status(HttpStatus.BAD_REQUEST)
                .body(ApiResponse.error(ex.getMessage()));
    }

    @ExceptionHandler(DuplicateResourceException.class)
    public ResponseEntity<ApiResponse<Void>> handleDuplicate(DuplicateResourceException ex) {
        log.warn("Duplicate resource: {}", ex.getMessage());
        return ResponseEntity
                .status(HttpStatus.CONFLICT)
                .body(ApiResponse.error(ex.getMessage()));
    }

    @ExceptionHandler(Exception.class)
    public ResponseEntity<ApiResponse<Void>> handleGeneral(Exception ex) {
        log.error("Unexpected error", ex);
        return ResponseEntity
                .status(HttpStatus.INTERNAL_SERVER_ERROR)
                .body(ApiResponse.error("Đã xảy ra lỗi. Vui lòng thử lại sau."));
    }
}
```

---

## 7. COMPLETE PRODUCT CONTROLLER

```java
package com.example.ecommerce.controller;

import com.example.ecommerce.dto.request.ProductCreateRequest;
import com.example.ecommerce.dto.request.ProductSearchRequest;
import com.example.ecommerce.dto.request.ProductUpdateRequest;
import com.example.ecommerce.dto.response.ApiResponse;
import com.example.ecommerce.dto.response.PagedResponse;
import com.example.ecommerce.dto.response.ProductResponse;
import com.example.ecommerce.service.ProductService;
import jakarta.validation.Valid;
import lombok.RequiredArgsConstructor;
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

@RestController
@RequestMapping("/api/products")
@RequiredArgsConstructor
public class ProductController {

    private final ProductService productService;

    /**
     * Tìm kiếm sản phẩm với filters
     * GET /api/products?keyword=iphone&categoryId=1&page=0&size=10
     */
    @GetMapping
    public ApiResponse<PagedResponse<ProductResponse>> searchProducts(
            @Valid ProductSearchRequest request) {
        return ApiResponse.success(productService.search(request));
    }

    /**
     * Lấy chi tiết sản phẩm theo ID
     * GET /api/products/123
     */
    @GetMapping("/{id}")
    public ApiResponse<ProductResponse> getProduct(@PathVariable Long id) {
        return ApiResponse.success(productService.getById(id));
    }

    /**
     * Lấy sản phẩm theo slug
     * GET /api/products/slug/iphone-15-pro-max
     */
    @GetMapping("/slug/{slug}")
    public ApiResponse<ProductResponse> getProductBySlug(@PathVariable String slug) {
        return ApiResponse.success(productService.getBySlug(slug));
    }

    /**
     * Tạo sản phẩm mới
     * POST /api/products
     */
    @PostMapping
    public ResponseEntity<ApiResponse<ProductResponse>> createProduct(
            @Valid @RequestBody ProductCreateRequest request) {
        ProductResponse created = productService.create(request);
        return ResponseEntity
                .status(HttpStatus.CREATED)
                .body(ApiResponse.created(created));
    }

    /**
     * Cập nhật sản phẩm
     * PUT /api/products/123
     */
    @PutMapping("/{id}")
    public ApiResponse<ProductResponse> updateProduct(
            @PathVariable Long id,
            @Valid @RequestBody ProductUpdateRequest request) {
        return ApiResponse.success("Cập nhật thành công", productService.update(id, request));
    }

    /**
     * Xóa sản phẩm
     * DELETE /api/products/123
     */
    @DeleteMapping("/{id}")
    public ApiResponse<Void> deleteProduct(@PathVariable Long id) {
        productService.delete(id);
        return ApiResponse.success("Xóa thành công", null);
    }

    /**
     * Sản phẩm nổi bật
     * GET /api/products/featured
     */
    @GetMapping("/featured")
    public ApiResponse<java.util.List<ProductResponse>> getFeaturedProducts() {
        return ApiResponse.success(productService.getFeatured());
    }

    /**
     * Sản phẩm theo category
     * GET /api/products/category/1
     */
    @GetMapping("/category/{categoryId}")
    public ApiResponse<PagedResponse<ProductResponse>> getByCategory(
            @PathVariable Long categoryId,
            @RequestParam(defaultValue = "0") int page,
            @RequestParam(defaultValue = "10") int size) {
        return ApiResponse.success(productService.getByCategory(categoryId, page, size));
    }
}
```

---

## 8. BÀI TẬP & CHECKLIST

### Bài tập thực hành

- [ ] Tạo `ProductController` với CRUD endpoints
- [ ] Tạo `ApiResponse` wrapper class
- [ ] Tạo custom exceptions (ResourceNotFoundException, etc.)
- [ ] Tạo `GlobalExceptionHandler`
- [ ] Test các endpoints với Postman/curl
- [ ] Verify response format JSON
- [ ] Test error cases (404, 400)

### Kiến thức cần nắm

| Khái niệm | Hiểu chưa? |
|-----------|-------------|
| REST principles | ☐ |
| HTTP methods (GET, POST, PUT, DELETE) | ☐ |
| @RestController vs @Controller | ☐ |
| @PathVariable, @RequestParam, @RequestBody | ☐ |
| ResponseEntity và status codes | ☐ |
| API Response wrapper pattern | ☐ |
| @RestControllerAdvice exception handling | ☐ |
| RESTful URL design | ☐ |

---

**Trước đó:** [← Ngày 10 - JPA Performance](../week-02/day-10-jpa-performance.md)

**Tiếp theo:** [Ngày 12 - CRUD API hoàn chỉnh →](day-12-crud-api.md)
