# Ngày 14: Bean Validation & Error Handling

## Mục tiêu hôm nay
- Hiểu Jakarta Bean Validation (JSR 380)
- Sử dụng built-in constraints (@NotBlank, @Size, @Email...)
- Tạo Custom Validators
- Xử lý validation errors thống nhất
- Validation Groups cho các scenarios khác nhau

---

## 1. BEAN VALIDATION OVERVIEW

### 1.1. Jakarta Bean Validation

```
┌─────────────────────────────────────────────────────────────┐
│              JAKARTA BEAN VALIDATION (JSR 380)              │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Specification: jakarta.validation (trước là javax.validation)│
│  Implementation: Hibernate Validator                         │
│  Spring Boot: spring-boot-starter-validation                 │
│                                                              │
│  Validate:                                                   │
│  ├── Request DTOs (@Valid @RequestBody)                      │
│  ├── Path Variables (@Validated @PathVariable)               │
│  ├── Query Parameters (@RequestParam)                        │
│  ├── Entity fields (before persist)                          │
│  └── Method parameters & return values                       │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### 1.2. Built-in Constraints

| Constraint | Mô tả | Ví dụ |
|------------|-------|-------|
| `@NotNull` | Không null | `private Long id;` |
| `@NotEmpty` | Không null và không rỗng (String, Collection) | `private List<Long> ids;` |
| `@NotBlank` | Không null, không rỗng, có ký tự không phải whitespace | `private String name;` |
| `@Size(min, max)` | Độ dài/kích thước | `@Size(min=2, max=100)` |
| `@Min(value)` | Giá trị tối thiểu (số) | `@Min(0)` |
| `@Max(value)` | Giá trị tối đa | `@Max(100)` |
| `@Positive` | Số dương (> 0) | `private BigDecimal price;` |
| `@PositiveOrZero` | Số không âm (>= 0) | `private Integer quantity;` |
| `@Negative` | Số âm (< 0) | |
| `@Email` | Format email hợp lệ | `private String email;` |
| `@Pattern(regexp)` | Khớp regex | `@Pattern(regexp="^[A-Z0-9-]+$")` |
| `@Past` | Ngày trong quá khứ | `private LocalDate birthDate;` |
| `@Future` | Ngày trong tương lai | `private LocalDate deliveryDate;` |
| `@Digits(integer, fraction)` | Số chữ số | `@Digits(integer=10, fraction=2)` |

---

## 2. VALIDATION TRONG REQUEST DTOs

### 2.1. ProductCreateRequest với Validation

```java
package com.example.ecommerce.dto.request;

import jakarta.validation.constraints.*;
import lombok.Data;

import java.math.BigDecimal;
import java.util.List;

@Data
public class ProductCreateRequest {

    @NotBlank(message = "Tên sản phẩm không được trống")
    @Size(min = 2, max = 200, message = "Tên sản phẩm từ 2-200 ký tự")
    private String name;

    @NotBlank(message = "SKU không được trống")
    @Size(min = 2, max = 50, message = "SKU từ 2-50 ký tự")
    @Pattern(regexp = "^[A-Z0-9-]+$", message = "SKU chỉ chứa chữ in hoa, số và dấu -")
    private String sku;

    @NotNull(message = "Giá không được trống")
    @Positive(message = "Giá phải lớn hơn 0")
    @Digits(integer = 10, fraction = 2, message = "Giá không hợp lệ")
    private BigDecimal price;

    @PositiveOrZero(message = "Giá khuyến mãi phải >= 0")
    private BigDecimal salePrice;

    @Size(max = 5000, message = "Mô tả tối đa 5000 ký tự")
    private String description;

    @PositiveOrZero(message = "Số lượng tồn kho phải >= 0")
    private Integer stockQuantity;

    @Positive(message = "Category ID phải lớn hơn 0")
    private Long categoryId;

    @Size(max = 10, message = "Tối đa 10 hình ảnh")
    private List<@URL(message = "URL hình ảnh không hợp lệ") String> images;

    @Size(max = 20, message = "Tối đa 20 tags")
    private List<@Positive(message = "Tag ID phải lớn hơn 0") Long> tagIds;
}
```

### 2.2. UserRegisterRequest

```java
@Data
public class UserRegisterRequest {

    @NotBlank(message = "Email không được trống")
    @Email(message = "Email không hợp lệ")
    @Size(max = 100, message = "Email tối đa 100 ký tự")
    private String email;

    @NotBlank(message = "Mật khẩu không được trống")
    @Size(min = 8, max = 100, message = "Mật khẩu từ 8-100 ký tự")
    @Pattern(
        regexp = "^(?=.*[a-z])(?=.*[A-Z])(?=.*\\d)(?=.*[@$!%*?&])[A-Za-z\\d@$!%*?&]{8,}$",
        message = "Mật khẩu phải có chữ hoa, chữ thường, số và ký tự đặc biệt"
    )
    private String password;

    @NotBlank(message = "Xác nhận mật khẩu không được trống")
    private String confirmPassword;

    @NotBlank(message = "Họ tên không được trống")
    @Size(min = 2, max = 100, message = "Họ tên từ 2-100 ký tự")
    private String fullName;

    @Pattern(regexp = "^(0[3|5|7|8|9])+([0-9]{8})$", message = "Số điện thoại không hợp lệ")
    private String phone;
}
```

### 2.3. Sử dụng @Valid trong Controller

```java
@RestController
@RequestMapping("/api/products")
@RequiredArgsConstructor
public class ProductController {

    private final ProductService productService;

    @PostMapping
    public ResponseEntity<ApiResponse<ProductResponse>> createProduct(
            @Valid @RequestBody ProductCreateRequest request) {  // @Valid triggers validation
        ProductResponse created = productService.create(request);
        return ResponseEntity.status(HttpStatus.CREATED)
                .body(ApiResponse.created(created));
    }

    @PutMapping("/{id}")
    public ApiResponse<ProductResponse> updateProduct(
            @PathVariable @Positive(message = "ID phải lớn hơn 0") Long id,
            @Valid @RequestBody ProductUpdateRequest request) {
        return ApiResponse.success(productService.update(id, request));
    }
}
```

---

## 3. VALIDATION ERROR HANDLING

### 3.1. GlobalExceptionHandler cho Validation

```java
package com.example.ecommerce.exception;

import com.example.ecommerce.dto.response.ApiResponse;
import jakarta.validation.ConstraintViolationException;
import lombok.extern.slf4j.Slf4j;
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.validation.FieldError;
import org.springframework.web.bind.MethodArgumentNotValidException;
import org.springframework.web.bind.annotation.ExceptionHandler;
import org.springframework.web.bind.annotation.RestControllerAdvice;

import java.util.HashMap;
import java.util.Map;

@RestControllerAdvice
@Slf4j
public class GlobalExceptionHandler {

    // ═══ @Valid @RequestBody validation errors ═══
    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ResponseEntity<ApiResponse<Void>> handleValidationException(
            MethodArgumentNotValidException ex) {

        Map<String, String> errors = new HashMap<>();

        ex.getBindingResult().getAllErrors().forEach(error -> {
            String fieldName = ((FieldError) error).getField();
            String message = error.getDefaultMessage();
            errors.put(fieldName, message);
        });

        log.warn("Validation failed: {}", errors);

        return ResponseEntity
                .status(HttpStatus.BAD_REQUEST)
                .body(ApiResponse.error("Dữ liệu không hợp lệ", errors));
    }

    // ═══ @Validated on class + @PathVariable/@RequestParam ═══
    @ExceptionHandler(ConstraintViolationException.class)
    public ResponseEntity<ApiResponse<Void>> handleConstraintViolation(
            ConstraintViolationException ex) {

        Map<String, String> errors = new HashMap<>();

        ex.getConstraintViolations().forEach(violation -> {
            String path = violation.getPropertyPath().toString();
            String field = path.contains(".") ? path.substring(path.lastIndexOf('.') + 1) : path;
            errors.put(field, violation.getMessage());
        });

        log.warn("Constraint violation: {}", errors);

        return ResponseEntity
                .status(HttpStatus.BAD_REQUEST)
                .body(ApiResponse.error("Tham số không hợp lệ", errors));
    }

    // ═══ Other exceptions ═══
    @ExceptionHandler(ResourceNotFoundException.class)
    public ResponseEntity<ApiResponse<Void>> handleNotFound(ResourceNotFoundException ex) {
        return ResponseEntity
                .status(HttpStatus.NOT_FOUND)
                .body(ApiResponse.error(ex.getMessage()));
    }

    @ExceptionHandler(BadRequestException.class)
    public ResponseEntity<ApiResponse<Void>> handleBadRequest(BadRequestException ex) {
        return ResponseEntity
                .status(HttpStatus.BAD_REQUEST)
                .body(ApiResponse.error(ex.getMessage()));
    }

    @ExceptionHandler(DuplicateResourceException.class)
    public ResponseEntity<ApiResponse<Void>> handleDuplicate(DuplicateResourceException ex) {
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

### 3.2. Error Response Format

```json
// POST /api/products với body không hợp lệ
{
  "success": false,
  "message": "Dữ liệu không hợp lệ",
  "errors": {
    "name": "Tên sản phẩm không được trống",
    "sku": "SKU chỉ chứa chữ in hoa, số và dấu -",
    "price": "Giá phải lớn hơn 0"
  },
  "timestamp": "2024-01-15T10:30:00Z"
}
```

---

## 4. CUSTOM VALIDATORS

### 4.1. Custom Annotation

```java
package com.example.ecommerce.validation;

import jakarta.validation.Constraint;
import jakarta.validation.Payload;
import java.lang.annotation.*;

@Documented
@Constraint(validatedBy = UniqueEmailValidator.class)
@Target({ElementType.FIELD})
@Retention(RetentionPolicy.RUNTIME)
public @interface UniqueEmail {
    String message() default "Email đã được sử dụng";
    Class<?>[] groups() default {};
    Class<? extends Payload>[] payload() default {};
}
```

### 4.2. Validator Implementation

```java
package com.example.ecommerce.validation;

import com.example.ecommerce.repository.UserRepository;
import jakarta.validation.ConstraintValidator;
import jakarta.validation.ConstraintValidatorContext;
import lombok.RequiredArgsConstructor;
import org.springframework.stereotype.Component;

@Component
@RequiredArgsConstructor
public class UniqueEmailValidator implements ConstraintValidator<UniqueEmail, String> {

    private final UserRepository userRepository;

    @Override
    public boolean isValid(String email, ConstraintValidatorContext context) {
        if (email == null || email.isBlank()) {
            return true;  // @NotBlank sẽ xử lý
        }
        return !userRepository.existsByEmail(email);
    }
}
```

### 4.3. Sử dụng Custom Validator

```java
@Data
public class UserRegisterRequest {

    @NotBlank(message = "Email không được trống")
    @Email(message = "Email không hợp lệ")
    @UniqueEmail  // Custom validator!
    private String email;

    // ...
}
```

### 4.4. Password Match Validator (Class-level)

```java
// Annotation
@Documented
@Constraint(validatedBy = PasswordMatchValidator.class)
@Target({ElementType.TYPE})  // Class-level
@Retention(RetentionPolicy.RUNTIME)
public @interface PasswordMatch {
    String message() default "Mật khẩu xác nhận không khớp";
    Class<?>[] groups() default {};
    Class<? extends Payload>[] payload() default {};

    String password();
    String confirmPassword();
}

// Validator
public class PasswordMatchValidator implements ConstraintValidator<PasswordMatch, Object> {

    private String passwordField;
    private String confirmPasswordField;

    @Override
    public void initialize(PasswordMatch annotation) {
        this.passwordField = annotation.password();
        this.confirmPasswordField = annotation.confirmPassword();
    }

    @Override
    public boolean isValid(Object obj, ConstraintValidatorContext context) {
        try {
            Object password = getFieldValue(obj, passwordField);
            Object confirmPassword = getFieldValue(obj, confirmPasswordField);

            if (password == null || confirmPassword == null) {
                return true;  // Other validators handle null
            }

            boolean isValid = password.equals(confirmPassword);

            if (!isValid) {
                context.disableDefaultConstraintViolation();
                context.buildConstraintViolationWithTemplate(context.getDefaultConstraintMessageTemplate())
                        .addPropertyNode(confirmPasswordField)
                        .addConstraintViolation();
            }

            return isValid;
        } catch (Exception e) {
            return false;
        }
    }

    private Object getFieldValue(Object obj, String fieldName) throws Exception {
        Field field = obj.getClass().getDeclaredField(fieldName);
        field.setAccessible(true);
        return field.get(obj);
    }
}

// Sử dụng
@Data
@PasswordMatch(password = "password", confirmPassword = "confirmPassword")
public class UserRegisterRequest {
    private String password;
    private String confirmPassword;
}
```

---

## 5. VALIDATION GROUPS

### 5.1. Define Groups

```java
package com.example.ecommerce.validation;

public interface ValidationGroups {

    interface Create {}

    interface Update {}

    interface PartialUpdate {}
}
```

### 5.2. Sử dụng Groups trong DTO

```java
@Data
public class ProductRequest {

    @Null(groups = ValidationGroups.Create.class, message = "ID phải null khi tạo mới")
    @NotNull(groups = ValidationGroups.Update.class, message = "ID không được trống khi cập nhật")
    private Long id;

    @NotBlank(groups = {ValidationGroups.Create.class}, message = "Tên không được trống")
    @Size(min = 2, max = 200, message = "Tên từ 2-200 ký tự")
    private String name;

    @NotBlank(groups = ValidationGroups.Create.class, message = "SKU không được trống")
    @Size(min = 2, max = 50)
    private String sku;

    @NotNull(groups = ValidationGroups.Create.class, message = "Giá không được trống")
    @Positive(message = "Giá phải lớn hơn 0")
    private BigDecimal price;
}
```

### 5.3. Controller với Groups

```java
@RestController
@RequestMapping("/api/products")
@Validated  // Required for @Validated at method level
public class ProductController {

    @PostMapping
    public ApiResponse<ProductResponse> create(
            @Validated(ValidationGroups.Create.class) @RequestBody ProductRequest request) {
        return ApiResponse.created(productService.create(request));
    }

    @PutMapping("/{id}")
    public ApiResponse<ProductResponse> update(
            @PathVariable Long id,
            @Validated(ValidationGroups.Update.class) @RequestBody ProductRequest request) {
        return ApiResponse.success(productService.update(id, request));
    }

    @PatchMapping("/{id}")
    public ApiResponse<ProductResponse> patch(
            @PathVariable Long id,
            @Validated(ValidationGroups.PartialUpdate.class) @RequestBody ProductRequest request) {
        return ApiResponse.success(productService.patch(id, request));
    }
}
```

---

## 6. VALIDATION TRONG SERVICE

### 6.1. Manual Validation

```java
@Service
@RequiredArgsConstructor
public class OrderService {

    private final Validator validator;  // jakarta.validation.Validator

    public OrderResponse createOrder(OrderCreateRequest request) {
        // Manual validation
        Set<ConstraintViolation<OrderCreateRequest>> violations = validator.validate(request);

        if (!violations.isEmpty()) {
            Map<String, String> errors = violations.stream()
                    .collect(Collectors.toMap(
                            v -> v.getPropertyPath().toString(),
                            ConstraintViolation::getMessage,
                            (m1, m2) -> m1
                    ));
            throw new ValidationException(errors);
        }

        // Business logic validation
        validateOrderItems(request.getItems());
        validatePaymentMethod(request.getPaymentMethod());

        // Create order...
    }

    private void validateOrderItems(List<OrderItemRequest> items) {
        if (items == null || items.isEmpty()) {
            throw new BadRequestException("Đơn hàng phải có ít nhất 1 sản phẩm");
        }

        for (OrderItemRequest item : items) {
            Product product = productRepository.findById(item.getProductId())
                    .orElseThrow(() -> new ResourceNotFoundException("Product", "id", item.getProductId()));

            if (!product.getIsActive()) {
                throw new BadRequestException("Sản phẩm '" + product.getName() + "' đã ngừng kinh doanh");
            }

            if (product.getStockQuantity() < item.getQuantity()) {
                throw new BadRequestException("Sản phẩm '" + product.getName() + "' không đủ số lượng");
            }
        }
    }
}
```

---

## 7. ERROR RESPONSE CHUẨN

### 7.1. ErrorResponse DTO

```java
@Data
@Builder
@AllArgsConstructor
@NoArgsConstructor
public class ErrorResponse {

    private Instant timestamp;
    private int status;
    private String error;
    private String message;
    private String path;
    private Map<String, String> validationErrors;

    public static ErrorResponse of(HttpStatus status, String message, String path) {
        return ErrorResponse.builder()
                .timestamp(Instant.now())
                .status(status.value())
                .error(status.getReasonPhrase())
                .message(message)
                .path(path)
                .build();
    }

    public static ErrorResponse validation(String path, Map<String, String> errors) {
        return ErrorResponse.builder()
                .timestamp(Instant.now())
                .status(HttpStatus.BAD_REQUEST.value())
                .error("Validation Failed")
                .message("Dữ liệu không hợp lệ")
                .path(path)
                .validationErrors(errors)
                .build();
    }
}
```

---

## 8. BÀI TẬP & CHECKLIST

### Bài tập thực hành

- [ ] Thêm validation annotations vào tất cả Request DTOs
- [ ] Xử lý MethodArgumentNotValidException
- [ ] Xử lý ConstraintViolationException
- [ ] Tạo custom @UniqueEmail validator
- [ ] Tạo @PasswordMatch class-level validator
- [ ] Implement Validation Groups (Create, Update)
- [ ] Test validation errors với Postman
- [ ] Verify error response format

### Kiến thức cần nắm

| Khái niệm | Hiểu chưa? |
|-----------|-------------|
| Jakarta Bean Validation annotations | ☐ |
| @Valid vs @Validated | ☐ |
| MethodArgumentNotValidException | ☐ |
| ConstraintViolationException | ☐ |
| Custom Validator annotation | ☐ |
| ConstraintValidator interface | ☐ |
| Validation Groups | ☐ |
| Class-level validation | ☐ |
| Business validation in Service | ☐ |

---

**Trước đó:** [← Ngày 13 - DTO & MapStruct](day-13-dto-mapstruct.md)

**Tiếp theo:** [Ngày 15 - Swagger/OpenAPI Documentation →](day-15-swagger.md)
