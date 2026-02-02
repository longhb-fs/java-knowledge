# Ngày 15: Swagger/OpenAPI & Logging

## Mục tiêu hôm nay
- Cài đặt SpringDoc OpenAPI (Swagger)
- Document API endpoints với annotations
- Customize Swagger UI
- Cấu hình Logging với SLF4J + Logback
- Structured logging best practices

---

## 1. SPRINGDOC OPENAPI SETUP

### 1.1. Dependency

```xml
<dependency>
    <groupId>org.springdoc</groupId>
    <artifactId>springdoc-openapi-starter-webmvc-ui</artifactId>
    <version>2.5.0</version>
</dependency>
```

### 1.2. Configuration

```yaml
# application.yml
springdoc:
  api-docs:
    path: /api-docs         # OpenAPI JSON endpoint
    enabled: true
  swagger-ui:
    path: /swagger-ui.html  # Swagger UI URL
    enabled: true
    tags-sorter: alpha      # Sort tags alphabetically
    operations-sorter: method
    display-request-duration: true
    filter: true            # Enable search filter
  show-actuator: false
  packages-to-scan: com.example.ecommerce.controller
  paths-to-match: /api/**
```

### 1.3. OpenAPI Configuration Class

```java
package com.example.ecommerce.config;

import io.swagger.v3.oas.models.Components;
import io.swagger.v3.oas.models.OpenAPI;
import io.swagger.v3.oas.models.info.Contact;
import io.swagger.v3.oas.models.info.Info;
import io.swagger.v3.oas.models.info.License;
import io.swagger.v3.oas.models.security.SecurityRequirement;
import io.swagger.v3.oas.models.security.SecurityScheme;
import io.swagger.v3.oas.models.servers.Server;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

import java.util.List;

@Configuration
public class OpenApiConfig {

    @Value("${spring.application.name:E-Commerce API}")
    private String appName;

    @Bean
    public OpenAPI customOpenAPI() {
        return new OpenAPI()
                .info(apiInfo())
                .servers(servers())
                .addSecurityItem(new SecurityRequirement().addList("Bearer Authentication"))
                .components(new Components()
                        .addSecuritySchemes("Bearer Authentication", securityScheme()));
    }

    private Info apiInfo() {
        return new Info()
                .title(appName)
                .version("1.0.0")
                .description("""
                    REST API cho hệ thống E-Commerce.

                    ## Tính năng chính:
                    - Quản lý sản phẩm (CRUD)
                    - Quản lý danh mục
                    - Giỏ hàng & Đặt hàng
                    - Authentication với JWT

                    ## Authentication:
                    Sử dụng JWT Bearer token. Nhấn nút **Authorize** và nhập token:
                    ```
                    Bearer eyJhbGciOiJIUzI1NiJ9...
                    ```
                    """)
                .contact(new Contact()
                        .name("Dev Team")
                        .email("dev@example.com")
                        .url("https://example.com"))
                .license(new License()
                        .name("MIT License")
                        .url("https://opensource.org/licenses/MIT"));
    }

    private List<Server> servers() {
        return List.of(
                new Server().url("http://localhost:8080").description("Development"),
                new Server().url("https://staging-api.example.com").description("Staging"),
                new Server().url("https://api.example.com").description("Production")
        );
    }

    private SecurityScheme securityScheme() {
        return new SecurityScheme()
                .type(SecurityScheme.Type.HTTP)
                .scheme("bearer")
                .bearerFormat("JWT")
                .description("Nhập JWT token (không cần prefix 'Bearer ')");
    }
}
```

### 1.4. Truy cập Swagger UI

```
Swagger UI: http://localhost:8080/swagger-ui.html
OpenAPI JSON: http://localhost:8080/api-docs
OpenAPI YAML: http://localhost:8080/api-docs.yaml
```

---

## 2. DOCUMENT CONTROLLERS

### 2.1. Controller với OpenAPI Annotations

```java
package com.example.ecommerce.controller;

import com.example.ecommerce.dto.request.ProductCreateRequest;
import com.example.ecommerce.dto.request.ProductSearchRequest;
import com.example.ecommerce.dto.request.ProductUpdateRequest;
import com.example.ecommerce.dto.response.ApiResponse;
import com.example.ecommerce.dto.response.PagedResponse;
import com.example.ecommerce.dto.response.ProductResponse;
import com.example.ecommerce.service.ProductService;
import io.swagger.v3.oas.annotations.Operation;
import io.swagger.v3.oas.annotations.Parameter;
import io.swagger.v3.oas.annotations.media.Content;
import io.swagger.v3.oas.annotations.media.Schema;
import io.swagger.v3.oas.annotations.responses.ApiResponses;
import io.swagger.v3.oas.annotations.security.SecurityRequirement;
import io.swagger.v3.oas.annotations.tags.Tag;
import jakarta.validation.Valid;
import lombok.RequiredArgsConstructor;
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

@RestController
@RequestMapping("/api/products")
@RequiredArgsConstructor
@Tag(name = "Products", description = "API quản lý sản phẩm")
public class ProductController {

    private final ProductService productService;

    @Operation(
            summary = "Tìm kiếm sản phẩm",
            description = "Tìm kiếm sản phẩm với các bộ lọc: category, giá, keyword, phân trang"
    )
    @ApiResponses({
            @io.swagger.v3.oas.annotations.responses.ApiResponse(
                    responseCode = "200",
                    description = "Thành công",
                    content = @Content(schema = @Schema(implementation = PagedResponse.class))
            )
    })
    @GetMapping
    public ApiResponse<PagedResponse<ProductResponse>> searchProducts(
            @Valid ProductSearchRequest request) {
        return ApiResponse.success(productService.search(request));
    }

    @Operation(
            summary = "Lấy chi tiết sản phẩm",
            description = "Lấy thông tin chi tiết sản phẩm theo ID"
    )
    @ApiResponses({
            @io.swagger.v3.oas.annotations.responses.ApiResponse(
                    responseCode = "200",
                    description = "Thành công"
            ),
            @io.swagger.v3.oas.annotations.responses.ApiResponse(
                    responseCode = "404",
                    description = "Không tìm thấy sản phẩm"
            )
    })
    @GetMapping("/{id}")
    public ApiResponse<ProductResponse> getProduct(
            @Parameter(description = "ID sản phẩm", required = true, example = "1")
            @PathVariable Long id) {
        return ApiResponse.success(productService.getById(id));
    }

    @Operation(
            summary = "Tạo sản phẩm mới",
            description = "Tạo sản phẩm mới. Yêu cầu quyền ADMIN.",
            security = @SecurityRequirement(name = "Bearer Authentication")
    )
    @ApiResponses({
            @io.swagger.v3.oas.annotations.responses.ApiResponse(
                    responseCode = "201",
                    description = "Tạo thành công"
            ),
            @io.swagger.v3.oas.annotations.responses.ApiResponse(
                    responseCode = "400",
                    description = "Dữ liệu không hợp lệ"
            ),
            @io.swagger.v3.oas.annotations.responses.ApiResponse(
                    responseCode = "401",
                    description = "Chưa xác thực"
            ),
            @io.swagger.v3.oas.annotations.responses.ApiResponse(
                    responseCode = "409",
                    description = "SKU đã tồn tại"
            )
    })
    @PostMapping
    public ResponseEntity<ApiResponse<ProductResponse>> createProduct(
            @Valid @RequestBody ProductCreateRequest request) {
        ProductResponse created = productService.create(request);
        return ResponseEntity
                .status(HttpStatus.CREATED)
                .body(ApiResponse.created(created));
    }

    @Operation(
            summary = "Cập nhật sản phẩm",
            description = "Cập nhật thông tin sản phẩm. Yêu cầu quyền ADMIN.",
            security = @SecurityRequirement(name = "Bearer Authentication")
    )
    @PutMapping("/{id}")
    public ApiResponse<ProductResponse> updateProduct(
            @Parameter(description = "ID sản phẩm", required = true)
            @PathVariable Long id,
            @Valid @RequestBody ProductUpdateRequest request) {
        return ApiResponse.success("Cập nhật thành công", productService.update(id, request));
    }

    @Operation(
            summary = "Xóa sản phẩm",
            description = "Soft delete sản phẩm. Yêu cầu quyền ADMIN.",
            security = @SecurityRequirement(name = "Bearer Authentication")
    )
    @ApiResponses({
            @io.swagger.v3.oas.annotations.responses.ApiResponse(responseCode = "200", description = "Xóa thành công"),
            @io.swagger.v3.oas.annotations.responses.ApiResponse(responseCode = "404", description = "Không tìm thấy")
    })
    @DeleteMapping("/{id}")
    public ApiResponse<Void> deleteProduct(@PathVariable Long id) {
        productService.delete(id);
        return ApiResponse.success("Xóa thành công", null);
    }
}
```

### 2.2. Document DTOs với @Schema

```java
package com.example.ecommerce.dto.request;

import io.swagger.v3.oas.annotations.media.Schema;
import jakarta.validation.constraints.*;
import lombok.Data;

import java.math.BigDecimal;
import java.util.List;

@Data
@Schema(description = "Request tạo sản phẩm mới")
public class ProductCreateRequest {

    @Schema(description = "Tên sản phẩm", example = "iPhone 15 Pro Max", required = true)
    @NotBlank(message = "Tên sản phẩm không được trống")
    @Size(min = 2, max = 200)
    private String name;

    @Schema(description = "Mã SKU (unique)", example = "IP15PM-256", required = true)
    @NotBlank
    @Pattern(regexp = "^[A-Z0-9-]+$")
    private String sku;

    @Schema(description = "Giá bán (VND)", example = "34990000", required = true, minimum = "0")
    @NotNull
    @Positive
    private BigDecimal price;

    @Schema(description = "Giá khuyến mãi (VND)", example = "32990000", nullable = true)
    @PositiveOrZero
    private BigDecimal salePrice;

    @Schema(description = "Mô tả sản phẩm", example = "Điện thoại Apple iPhone 15 Pro Max...")
    @Size(max = 5000)
    private String description;

    @Schema(description = "Số lượng tồn kho", example = "100", defaultValue = "0")
    @PositiveOrZero
    private Integer stockQuantity = 0;

    @Schema(description = "ID danh mục", example = "1")
    private Long categoryId;

    @Schema(description = "URL hình ảnh chính", example = "https://example.com/images/product.jpg")
    private String mainImage;

    @Schema(description = "Danh sách URL hình ảnh phụ")
    private List<String> images;

    @Schema(description = "Danh sách ID tags", example = "[1, 2, 3]")
    private List<Long> tagIds;

    @Schema(description = "Trạng thái active", defaultValue = "true")
    private Boolean isActive = true;

    @Schema(description = "Sản phẩm nổi bật", defaultValue = "false")
    private Boolean isFeatured = false;
}
```

```java
@Data
@Schema(description = "Response thông tin sản phẩm")
public class ProductResponse {

    @Schema(description = "ID sản phẩm", example = "1")
    private Long id;

    @Schema(description = "Tên sản phẩm", example = "iPhone 15 Pro Max")
    private String name;

    @Schema(description = "Slug URL-friendly", example = "iphone-15-pro-max")
    private String slug;

    @Schema(description = "Mã SKU", example = "IP15PM-256")
    private String sku;

    @Schema(description = "Giá gốc (VND)", example = "34990000")
    private BigDecimal price;

    @Schema(description = "Giá khuyến mãi (VND)", example = "32990000")
    private BigDecimal salePrice;

    @Schema(description = "Giá thực tế (sale hoặc gốc)", example = "32990000")
    private BigDecimal effectivePrice;

    @Schema(description = "Còn hàng", example = "true")
    private Boolean inStock;

    @Schema(description = "Tên danh mục", example = "Điện thoại")
    private String categoryName;

    @Schema(description = "Thời gian tạo")
    private LocalDateTime createdAt;
}
```

---

## 3. LOGGING CONFIGURATION

### 3.1. Logback Configuration

```yaml
# application.yml
logging:
  level:
    root: INFO
    com.example.ecommerce: DEBUG
    org.springframework.web: INFO
    org.springframework.security: DEBUG
    org.hibernate.SQL: DEBUG
    org.hibernate.type.descriptor.sql.BasicBinder: TRACE

  pattern:
    console: "%d{HH:mm:ss.SSS} %clr(%-5level) [%thread] %cyan(%logger{36}) - %msg%n"
    file: "%d{yyyy-MM-dd HH:mm:ss.SSS} %-5level [%thread] %logger{36} - %msg%n"

  file:
    name: logs/ecommerce.log

  logback:
    rollingpolicy:
      max-file-size: 10MB
      max-history: 30
      total-size-cap: 1GB
```

### 3.2. Custom logback-spring.xml

```xml
<?xml version="1.0" encoding="UTF-8"?>
<configuration>

    <!-- Console Appender -->
    <appender name="CONSOLE" class="ch.qos.logback.core.ConsoleAppender">
        <encoder>
            <pattern>%d{HH:mm:ss.SSS} %highlight(%-5level) [%thread] %cyan(%logger{36}) - %msg%n</pattern>
        </encoder>
    </appender>

    <!-- File Appender with Rolling -->
    <appender name="FILE" class="ch.qos.logback.core.rolling.RollingFileAppender">
        <file>logs/ecommerce.log</file>
        <rollingPolicy class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy">
            <fileNamePattern>logs/ecommerce.%d{yyyy-MM-dd}.%i.log</fileNamePattern>
            <timeBasedFileNamingAndTriggeringPolicy class="ch.qos.logback.core.rolling.SizeAndTimeBasedFNATP">
                <maxFileSize>10MB</maxFileSize>
            </timeBasedFileNamingAndTriggeringPolicy>
            <maxHistory>30</maxHistory>
            <totalSizeCap>1GB</totalSizeCap>
        </rollingPolicy>
        <encoder>
            <pattern>%d{yyyy-MM-dd HH:mm:ss.SSS} %-5level [%thread] %logger{36} - %msg%n</pattern>
        </encoder>
    </appender>

    <!-- JSON Appender for Production -->
    <appender name="JSON" class="ch.qos.logback.core.rolling.RollingFileAppender">
        <file>logs/ecommerce-json.log</file>
        <rollingPolicy class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy">
            <fileNamePattern>logs/ecommerce-json.%d{yyyy-MM-dd}.log</fileNamePattern>
            <maxHistory>7</maxHistory>
        </rollingPolicy>
        <encoder class="net.logstash.logback.encoder.LogstashEncoder">
            <includeMdc>true</includeMdc>
            <includeContext>false</includeContext>
        </encoder>
    </appender>

    <!-- Application Loggers -->
    <logger name="com.example.ecommerce" level="DEBUG" />
    <logger name="org.springframework.web" level="INFO" />
    <logger name="org.hibernate.SQL" level="DEBUG" />

    <!-- Root Logger -->
    <root level="INFO">
        <appender-ref ref="CONSOLE" />
        <appender-ref ref="FILE" />
    </root>

    <!-- Production Profile -->
    <springProfile name="prod">
        <root level="WARN">
            <appender-ref ref="FILE" />
            <appender-ref ref="JSON" />
        </root>
    </springProfile>

</configuration>
```

### 3.3. Sử dụng Logger trong Code

```java
package com.example.ecommerce.service.impl;

import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.stereotype.Service;

@Service
@RequiredArgsConstructor
@Slf4j  // Lombok tạo: private static final Logger log = ...
public class ProductServiceImpl implements ProductService {

    @Override
    @Transactional
    public ProductResponse create(ProductCreateRequest request) {
        log.info("Creating product: name={}, sku={}", request.getName(), request.getSku());

        try {
            // Validation
            if (productRepository.existsBySku(request.getSku())) {
                log.warn("SKU already exists: {}", request.getSku());
                throw new DuplicateResourceException("SKU đã tồn tại");
            }

            Product product = productMapper.toEntity(request);
            product.setSlug(generateUniqueSlug(request.getName()));

            product = productRepository.save(product);

            log.info("Product created successfully: id={}, name={}",
                    product.getId(), product.getName());

            return productMapper.toResponse(product);

        } catch (Exception e) {
            log.error("Error creating product: name={}, error={}",
                    request.getName(), e.getMessage(), e);
            throw e;
        }
    }

    @Override
    public ProductResponse getById(Long id) {
        log.debug("Fetching product by id: {}", id);

        Product product = productRepository.findById(id)
                .orElseThrow(() -> {
                    log.warn("Product not found: id={}", id);
                    return new ResourceNotFoundException("Product", "id", id);
                });

        log.debug("Product found: id={}, name={}", id, product.getName());
        return productMapper.toResponse(product);
    }
}
```

### 3.4. Request/Response Logging Filter

```java
package com.example.ecommerce.config;

import jakarta.servlet.*;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;
import lombok.extern.slf4j.Slf4j;
import org.springframework.core.Ordered;
import org.springframework.core.annotation.Order;
import org.springframework.stereotype.Component;
import org.springframework.web.util.ContentCachingRequestWrapper;
import org.springframework.web.util.ContentCachingResponseWrapper;

import java.io.IOException;
import java.util.UUID;

@Component
@Order(Ordered.HIGHEST_PRECEDENCE)
@Slf4j
public class RequestLoggingFilter implements Filter {

    @Override
    public void doFilter(ServletRequest request, ServletResponse response, FilterChain chain)
            throws IOException, ServletException {

        HttpServletRequest httpRequest = (HttpServletRequest) request;
        HttpServletResponse httpResponse = (HttpServletResponse) response;

        // Wrap for reading body
        ContentCachingRequestWrapper requestWrapper = new ContentCachingRequestWrapper(httpRequest);
        ContentCachingResponseWrapper responseWrapper = new ContentCachingResponseWrapper(httpResponse);

        // Generate request ID
        String requestId = UUID.randomUUID().toString().substring(0, 8);
        httpResponse.setHeader("X-Request-Id", requestId);

        long startTime = System.currentTimeMillis();

        try {
            // Log request
            log.info("[{}] {} {} - Start",
                    requestId,
                    httpRequest.getMethod(),
                    httpRequest.getRequestURI());

            chain.doFilter(requestWrapper, responseWrapper);

        } finally {
            long duration = System.currentTimeMillis() - startTime;

            // Log response
            log.info("[{}] {} {} - {} ({}ms)",
                    requestId,
                    httpRequest.getMethod(),
                    httpRequest.getRequestURI(),
                    responseWrapper.getStatus(),
                    duration);

            // Copy content to response
            responseWrapper.copyBodyToResponse();
        }
    }
}
```

---

## 4. MDC (Mapped Diagnostic Context)

### 4.1. MDC Filter

```java
@Component
@Order(Ordered.HIGHEST_PRECEDENCE + 1)
public class MDCFilter implements Filter {

    @Override
    public void doFilter(ServletRequest request, ServletResponse response, FilterChain chain)
            throws IOException, ServletException {

        HttpServletRequest httpRequest = (HttpServletRequest) request;

        try {
            // Add context to all logs in this request
            MDC.put("requestId", UUID.randomUUID().toString().substring(0, 8));
            MDC.put("method", httpRequest.getMethod());
            MDC.put("uri", httpRequest.getRequestURI());
            MDC.put("ip", httpRequest.getRemoteAddr());

            // Add user if authenticated
            String authHeader = httpRequest.getHeader("Authorization");
            if (authHeader != null && authHeader.startsWith("Bearer ")) {
                // Extract user from token...
                MDC.put("userId", "extracted-user-id");
            }

            chain.doFilter(request, response);

        } finally {
            MDC.clear();  // Important: clear after request
        }
    }
}
```

### 4.2. Log Pattern với MDC

```xml
<pattern>%d{HH:mm:ss} [%X{requestId}] %-5level %logger{36} - %msg%n</pattern>

<!-- Output: 10:30:45 [abc123] INFO ProductService - Creating product... -->
```

---

## 5. BÀI TẬP & CHECKLIST

### Bài tập thực hành

- [ ] Cài đặt springdoc-openapi
- [ ] Tạo OpenApiConfig với JWT security
- [ ] Document ProductController với annotations
- [ ] Document DTOs với @Schema
- [ ] Truy cập Swagger UI và test
- [ ] Cấu hình logging levels
- [ ] Tạo RequestLoggingFilter
- [ ] Thêm MDC context
- [ ] Test log output

### Kiến thức cần nắm

| Khái niệm | Hiểu chưa? |
|-----------|-------------|
| SpringDoc OpenAPI vs Springfox | ☐ |
| @Operation, @ApiResponses | ☐ |
| @Schema cho DTOs | ☐ |
| @SecurityRequirement | ☐ |
| @Slf4j (Lombok) | ☐ |
| Log levels (TRACE, DEBUG, INFO, WARN, ERROR) | ☐ |
| Logback configuration | ☐ |
| MDC (Mapped Diagnostic Context) | ☐ |
| Request logging filter | ☐ |

---

**Trước đó:** [← Ngày 14 - Bean Validation](day-14-validation.md)

**Tiếp theo:** [Ngày 16 - Spring Security Fundamentals →](../week-04/day-16-spring-security.md)
