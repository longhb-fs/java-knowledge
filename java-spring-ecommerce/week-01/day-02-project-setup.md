# Ngày 02: Khởi tạo Project & Clean Architecture

## Mục tiêu hôm nay
- Hiểu Clean Architecture / Layered Architecture trong Spring Boot
- Tổ chức project theo package structure chuẩn
- Tạo base classes dùng chung (BaseEntity, ApiResponse)
- Cấu hình Lombok để giảm boilerplate code
- Hiểu Maven dependency management chuyên sâu

---

## 1. CLEAN ARCHITECTURE TRONG JAVA

### 1.1. Tại sao cần kiến trúc rõ ràng?

Không có kiến trúc → code lộn xộn → không test được → khó maintain:

```
❌ Code không có kiến trúc:
Controller gọi thẳng Database
    → Business logic trộn lẫn trong Controller
    → Không thể unit test
    → Thay database = sửa toàn bộ code

✅ Code có kiến trúc:
Controller → Service → Repository → Database
    → Mỗi layer có trách nhiệm rõ ràng
    → Unit test từng layer riêng
    → Thay database chỉ cần sửa Repository
```

### 1.2. Layered Architecture (kiến trúc phân lớp)

```
┌─────────────────────────────────────────────────┐
│                 PRESENTATION LAYER               │
│          (Controller / REST API / DTO)           │
│                                                   │
│  Trách nhiệm:                                    │
│  • Nhận HTTP request                              │
│  • Validate input cơ bản                          │
│  • Gọi Service                                    │
│  • Trả HTTP response (JSON)                       │
├─────────────────────────────────────────────────┤
│                  SERVICE LAYER                    │
│         (Business Logic / Use Cases)              │
│                                                   │
│  Trách nhiệm:                                    │
│  • Xử lý business logic                          │
│  • Orchestrate nhiều Repository                  │
│  • Transaction management                         │
│  • Validation business rules                      │
│  • Gọi external services (email, payment)         │
├─────────────────────────────────────────────────┤
│                REPOSITORY LAYER                   │
│          (Data Access / Persistence)              │
│                                                   │
│  Trách nhiệm:                                    │
│  • CRUD operations                                │
│  • Custom queries (JPQL, Native SQL)              │
│  • Pagination, Sorting                            │
│  • Database-specific logic                        │
├─────────────────────────────────────────────────┤
│                  ENTITY LAYER                     │
│             (Domain Model / JPA)                  │
│                                                   │
│  Trách nhiệm:                                    │
│  • Định nghĩa cấu trúc dữ liệu                  │
│  • JPA mapping (table, column, relationship)      │
│  • Domain validation rules                        │
│  • Audit fields (createdAt, updatedAt)            │
└─────────────────────────────────────────────────┘
```

### 1.3. Quy tắc phụ thuộc (Dependency Rule)

```
Controller ──depends on──> Service ──depends on──> Repository ──depends on──> Entity

KHÔNG ĐƯỢC:
  Controller ──❌──> Repository  (bỏ qua Service)
  Service    ──❌──> Controller  (phụ thuộc ngược)
  Repository ──❌──> Service     (phụ thuộc ngược)
```

**Ví dụ thực tế:**
```java
// ✅ ĐÚNG: Controller gọi Service
@RestController
public class ProductController {
    private final ProductService productService;  // Service

    @GetMapping("/products/{id}")
    public ProductResponse getProduct(@PathVariable Long id) {
        return productService.getById(id);  // Gọi Service
    }
}

// ❌ SAI: Controller gọi thẳng Repository
@RestController
public class ProductController {
    private final ProductRepository productRepository;  // Repository trực tiếp

    @GetMapping("/products/{id}")
    public Product getProduct(@PathVariable Long id) {
        return productRepository.findById(id).orElseThrow();  // Bypass Service!
    }
}
```

---

## 2. PACKAGE STRUCTURE

### 2.1. Package by Layer (chúng ta dùng cách này)

```
com.example.ecommerce/
├── EcommerceApplication.java
│
├── config/                    # Cấu hình (Security, Redis, OpenAPI...)
│   ├── SecurityConfig.java
│   ├── RedisConfig.java
│   ├── OpenApiConfig.java
│   └── AsyncConfig.java
│
├── entity/                    # JPA Entities (Domain Model)
│   ├── BaseEntity.java
│   ├── User.java
│   ├── Product.java
│   ├── Category.java
│   ├── Order.java
│   ├── OrderItem.java
│   ├── CartItem.java
│   └── enums/
│       ├── OrderStatus.java
│       ├── PaymentStatus.java
│       └── UserRole.java
│
├── repository/                # Spring Data JPA Repositories
│   ├── UserRepository.java
│   ├── ProductRepository.java
│   ├── CategoryRepository.java
│   ├── OrderRepository.java
│   └── specification/
│       └── ProductSpecification.java
│
├── service/                   # Business Logic
│   ├── AuthService.java
│   ├── UserService.java
│   ├── ProductService.java
│   ├── CategoryService.java
│   ├── CartService.java
│   ├── OrderService.java
│   ├── PaymentService.java
│   └── EmailService.java
│
├── controller/                # REST Controllers
│   ├── AuthController.java
│   ├── UserController.java
│   ├── ProductController.java
│   ├── CategoryController.java
│   ├── CartController.java
│   └── OrderController.java
│
├── dto/                       # Data Transfer Objects
│   ├── request/
│   │   ├── LoginRequest.java
│   │   ├── RegisterRequest.java
│   │   ├── ProductCreateRequest.java
│   │   ├── ProductUpdateRequest.java
│   │   └── CartItemRequest.java
│   ├── response/
│   │   ├── ApiResponse.java
│   │   ├── PagedResponse.java
│   │   ├── JwtResponse.java
│   │   ├── ProductResponse.java
│   │   └── OrderResponse.java
│   └── mapper/
│       ├── ProductMapper.java
│       └── UserMapper.java
│
├── security/                  # Spring Security (JWT)
│   ├── JwtTokenProvider.java
│   ├── JwtAuthFilter.java
│   ├── CustomUserDetailsService.java
│   └── SecurityUtils.java
│
├── exception/                 # Exception Handling
│   ├── GlobalExceptionHandler.java
│   ├── ResourceNotFoundException.java
│   ├── BadRequestException.java
│   ├── DuplicateResourceException.java
│   └── UnauthorizedException.java
│
└── util/                      # Utilities
    ├── SlugUtils.java
    └── FileUploadUtils.java
```

### 2.2. Package by Feature (tham khảo - dùng cho project lớn)

```
com.example.ecommerce/
├── product/
│   ├── ProductController.java
│   ├── ProductService.java
│   ├── ProductRepository.java
│   ├── Product.java
│   └── dto/
│       ├── ProductCreateRequest.java
│       └── ProductResponse.java
├── order/
│   ├── OrderController.java
│   ├── OrderService.java
│   ├── OrderRepository.java
│   ├── Order.java
│   └── dto/
├── user/
│   ├── UserController.java
│   ├── UserService.java
│   └── ...
└── shared/
    ├── BaseEntity.java
    ├── ApiResponse.java
    └── GlobalExceptionHandler.java
```

**So sánh:**

| Tiêu chí | Package by Layer | Package by Feature |
|-----------|-----------------|-------------------|
| **Tìm file** | Theo role (controller/service) | Theo business domain |
| **Dễ học** | Dễ hơn (cấu trúc quen thuộc) | Phức tạp hơn |
| **Scale** | Khó khi > 20 entity | Tốt hơn |
| **Coupling** | Cao hơn (cross-package refs) | Thấp hơn |
| **Phù hợp** | Project nhỏ-trung | Project lớn, microservices |

---

## 3. TẠO BASE CLASSES

### 3.1. BaseEntity - Entity cơ sở

Mọi entity trong project đều kế thừa từ đây:

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
@MappedSuperclass  // Không tạo table riêng, chỉ cho kế thừa
public abstract class BaseEntity {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)  // Auto-increment
    private Long id;

    @CreationTimestamp  // Hibernate tự set khi INSERT
    @Column(name = "created_at", nullable = false, updatable = false)
    private LocalDateTime createdAt;

    @UpdateTimestamp    // Hibernate tự set khi UPDATE
    @Column(name = "updated_at")
    private LocalDateTime updatedAt;
}
```

**Giải thích annotation:**

| Annotation | Mục đích |
|------------|----------|
| `@MappedSuperclass` | Cho phép entity con kế thừa các field, nhưng KHÔNG tạo table riêng |
| `@Id` | Đánh dấu primary key |
| `@GeneratedValue(IDENTITY)` | Auto-increment (MySQL) |
| `@CreationTimestamp` | Tự gán thời gian khi tạo record mới |
| `@UpdateTimestamp` | Tự cập nhật thời gian khi update record |
| `@Column(updatable = false)` | Không cho phép update field này |

### 3.2. ApiResponse - Response wrapper chuẩn

Tất cả API đều trả về format thống nhất:

```java
package com.example.ecommerce.dto.response;

import com.fasterxml.jackson.annotation.JsonInclude;
import lombok.AllArgsConstructor;
import lombok.Builder;
import lombok.Data;
import lombok.NoArgsConstructor;

@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
@JsonInclude(JsonInclude.Include.NON_NULL)  // Bỏ field null khỏi JSON
public class ApiResponse<T> {

    private boolean success;
    private String message;
    private T data;
    private Object errors;

    // Factory methods - tiện dùng
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

**Kết quả JSON:**
```json
// Thành công
{
  "success": true,
  "message": "Thành công",
  "data": {
    "id": 1,
    "name": "iPhone 15",
    "price": 25000000
  }
}

// Lỗi
{
  "success": false,
  "message": "Không tìm thấy sản phẩm",
  "errors": {
    "id": "Sản phẩm với ID 999 không tồn tại"
  }
}
```

### 3.3. PagedResponse - Phân trang chuẩn

```java
package com.example.ecommerce.dto.response;

import lombok.AllArgsConstructor;
import lombok.Builder;
import lombok.Data;
import lombok.NoArgsConstructor;

import java.util.List;

@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class PagedResponse<T> {

    private List<T> content;       // Dữ liệu trang hiện tại
    private int page;              // Trang hiện tại (0-based)
    private int size;              // Số phần tử mỗi trang
    private long totalElements;    // Tổng số phần tử
    private int totalPages;        // Tổng số trang
    private boolean last;          // Có phải trang cuối không
    private boolean first;         // Có phải trang đầu không

    // Factory method từ Spring Page
    public static <T> PagedResponse<T> from(org.springframework.data.domain.Page<T> page) {
        return PagedResponse.<T>builder()
                .content(page.getContent())
                .page(page.getNumber())
                .size(page.getSize())
                .totalElements(page.getTotalElements())
                .totalPages(page.getTotalPages())
                .last(page.isLast())
                .first(page.isFirst())
                .build();
    }
}
```

---

## 4. LOMBOK CHI TIẾT

### 4.1. Lombok là gì?

Lombok là thư viện **tự sinh code** tại compile time, giúp giảm boilerplate code trong Java.

```
Không có Lombok:                    Có Lombok:
──────────────────                  ──────────────
100 dòng code                      10 dòng code
(constructor, getter,              (@Data, @Builder)
 setter, equals, hashCode,
 toString, builder...)
```

### 4.2. Các annotation thường dùng

```java
// ──────────────────────────────────────
// @Getter / @Setter - Tự sinh getter/setter
// ──────────────────────────────────────
@Getter
@Setter
public class Product {
    private String name;
    private BigDecimal price;
    // Lombok tự sinh: getName(), setName(), getPrice(), setPrice()
}

// ──────────────────────────────────────
// @ToString - Tự sinh toString()
// ──────────────────────────────────────
@ToString
public class Product {
    private String name;
    private BigDecimal price;
    // toString() → "Product(name=iPhone, price=25000000)"
}

@ToString(exclude = "password")  // Ẩn field nhạy cảm
public class User {
    private String email;
    private String password;
    // toString() → "User(email=a@b.com)"  (không hiện password)
}

// ──────────────────────────────────────
// @EqualsAndHashCode - Tự sinh equals() + hashCode()
// ──────────────────────────────────────
@EqualsAndHashCode(of = "id")  // So sánh chỉ theo id
public class Product {
    private Long id;
    private String name;
}

// ──────────────────────────────────────
// @NoArgsConstructor / @AllArgsConstructor
// ──────────────────────────────────────
@NoArgsConstructor   // Constructor không tham số: new Product()
@AllArgsConstructor  // Constructor đầy đủ: new Product(id, name, price)
public class Product {
    private Long id;
    private String name;
    private BigDecimal price;
}

// ──────────────────────────────────────
// @Data = @Getter + @Setter + @ToString + @EqualsAndHashCode + @RequiredArgsConstructor
// ──────────────────────────────────────
@Data
public class ProductResponse {
    private Long id;
    private String name;
    private BigDecimal price;
    // Tự có tất cả: getter, setter, toString, equals, hashCode
}

// ──────────────────────────────────────
// @Builder - Builder pattern
// ──────────────────────────────────────
@Builder
public class Product {
    private String name;
    private BigDecimal price;
    private String description;
}

// Sử dụng:
Product product = Product.builder()
    .name("iPhone 15")
    .price(new BigDecimal("25000000"))
    .description("Điện thoại Apple")
    .build();

// ──────────────────────────────────────
// @Slf4j - Logger tự động
// ──────────────────────────────────────
@Slf4j
@Service
public class ProductService {
    public void createProduct() {
        log.info("Tạo sản phẩm mới");        // INFO level
        log.debug("Chi tiết: {}", product);   // DEBUG level
        log.error("Lỗi: {}", e.getMessage()); // ERROR level
    }
}
// Lombok tự sinh: private static final Logger log = LoggerFactory.getLogger(ProductService.class);

// ──────────────────────────────────────
// @RequiredArgsConstructor - Constructor Injection (quan trọng nhất!)
// ──────────────────────────────────────
@Service
@RequiredArgsConstructor  // Tự tạo constructor cho các final field
public class ProductService {
    private final ProductRepository productRepository;  // final = bắt buộc inject
    private final CategoryRepository categoryRepository;

    // Lombok tự sinh:
    // public ProductService(ProductRepository productRepository, CategoryRepository categoryRepository) {
    //     this.productRepository = productRepository;
    //     this.categoryRepository = categoryRepository;
    // }
}
```

### 4.3. Lombok cho Entity - Cẩn thận!

```java
// ⚠️ KHÔNG dùng @Data cho JPA Entity
// Lý do: @Data sinh equals/hashCode dựa trên TẤT CẢ field
//         → Gây vấn đề với Lazy Loading (Hibernate proxy)
//         → Infinite loop khi có quan hệ hai chiều

// ✅ ĐÚNG: Dùng @Getter + @Setter cho Entity
@Entity
@Table(name = "products")
@Getter
@Setter
@NoArgsConstructor
@AllArgsConstructor
@Builder
@ToString(exclude = {"category", "orderItems"})  // Tránh lazy loading
@EqualsAndHashCode(of = "id")                     // Chỉ so sánh theo id
public class Product extends BaseEntity {
    private String name;
    private BigDecimal price;

    @ManyToOne(fetch = FetchType.LAZY)
    private Category category;
}

// ✅ ĐÚNG: Dùng @Data cho DTO (không phải Entity)
@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class ProductResponse {
    private Long id;
    private String name;
    private BigDecimal price;
    private String categoryName;
}
```

---

## 5. MAVEN DEPENDENCY MANAGEMENT

### 5.1. Maven Lifecycle

```
┌──────────────────────────────────────────────┐
│              MAVEN BUILD LIFECYCLE            │
│                                               │
│  validate  ──>  compile  ──>  test            │
│      │              │            │             │
│      │              │            ▼             │
│      │              │         package          │
│      │              │            │             │
│      │              │            ▼             │
│      │              │         verify           │
│      │              │            │             │
│      │              │            ▼             │
│      │              │         install          │
│      │              │            │             │
│      │              │            ▼             │
│      │              │          deploy          │
│      │              │                          │
│      ▼              ▼                          │
│   clean         site (docs)                   │
└──────────────────────────────────────────────┘
```

```bash
# Các lệnh Maven thường dùng
mvn clean                  # Xóa thư mục target/
mvn compile                # Compile source code
mvn test                   # Chạy unit tests
mvn package                # Đóng gói thành JAR/WAR
mvn clean package          # Clean + Build (phổ biến nhất)
mvn clean package -DskipTests  # Build bỏ qua test
mvn install                # Install vào local repository (~/.m2)
mvn dependency:tree        # Hiện dependency tree
mvn spring-boot:run        # Chạy Spring Boot app
```

### 5.2. Dependency Scope

```xml
<dependency>
    <groupId>com.mysql</groupId>
    <artifactId>mysql-connector-j</artifactId>
    <scope>runtime</scope>  <!-- Scope quan trọng! -->
</dependency>
```

| Scope | Compile | Test | Runtime | Ví dụ |
|-------|---------|------|---------|-------|
| **compile** (default) | ✅ | ✅ | ✅ | spring-boot-starter-web |
| **runtime** | ❌ | ✅ | ✅ | mysql-connector-j |
| **test** | ❌ | ✅ | ❌ | spring-boot-starter-test |
| **provided** | ✅ | ✅ | ❌ | lombok |
| **optional** | ✅ | ✅ | ✅ (không transitive) | devtools |

### 5.3. Dependency Tree - Kiểm tra conflict

```bash
mvn dependency:tree

# Output ví dụ:
[INFO] com.example:ecommerce:jar:0.0.1-SNAPSHOT
[INFO] +- org.springframework.boot:spring-boot-starter-web:jar:3.2.5
[INFO] |  +- org.springframework.boot:spring-boot-starter:jar:3.2.5
[INFO] |  |  +- org.springframework.boot:spring-boot:jar:3.2.5
[INFO] |  |  +- org.springframework.boot:spring-boot-autoconfigure:jar:3.2.5
[INFO] |  |  +- org.springframework.boot:spring-boot-starter-logging:jar:3.2.5
[INFO] |  |  |  +- ch.qos.logback:logback-classic:jar:1.4.14
[INFO] |  |  |  +- org.apache.logging.log4j:log4j-to-slf4j:jar:2.21.1
[INFO] |  +- org.springframework:spring-webmvc:jar:6.1.6
[INFO] |  +- org.apache.tomcat.embed:tomcat-embed-core:jar:10.1.20
[INFO] |  +- com.fasterxml.jackson.core:jackson-databind:jar:2.15.4
```

### 5.4. Thêm dependencies cho khóa học

Cập nhật `pom.xml` với tất cả dependencies cần thiết:

```xml
<properties>
    <java.version>21</java.version>
    <mapstruct.version>1.5.5.Final</mapstruct.version>
</properties>

<dependencies>
    <!-- ======= CORE ======= -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>

    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-data-jpa</artifactId>
    </dependency>

    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-validation</artifactId>
    </dependency>

    <!-- ======= SECURITY ======= -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-security</artifactId>
    </dependency>

    <!-- JWT -->
    <dependency>
        <groupId>io.jsonwebtoken</groupId>
        <artifactId>jjwt-api</artifactId>
        <version>0.12.5</version>
    </dependency>
    <dependency>
        <groupId>io.jsonwebtoken</groupId>
        <artifactId>jjwt-impl</artifactId>
        <version>0.12.5</version>
        <scope>runtime</scope>
    </dependency>
    <dependency>
        <groupId>io.jsonwebtoken</groupId>
        <artifactId>jjwt-jackson</artifactId>
        <version>0.12.5</version>
        <scope>runtime</scope>
    </dependency>

    <!-- ======= DATABASE ======= -->
    <dependency>
        <groupId>com.mysql</groupId>
        <artifactId>mysql-connector-j</artifactId>
        <scope>runtime</scope>
    </dependency>

    <dependency>
        <groupId>org.flywaydb</groupId>
        <artifactId>flyway-core</artifactId>
    </dependency>
    <dependency>
        <groupId>org.flywaydb</groupId>
        <artifactId>flyway-mysql</artifactId>
    </dependency>

    <!-- ======= CACHE ======= -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-data-redis</artifactId>
    </dependency>

    <!-- ======= EMAIL ======= -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-mail</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-thymeleaf</artifactId>
    </dependency>

    <!-- ======= MAPPING ======= -->
    <dependency>
        <groupId>org.mapstruct</groupId>
        <artifactId>mapstruct</artifactId>
        <version>${mapstruct.version}</version>
    </dependency>

    <!-- ======= API DOCS ======= -->
    <dependency>
        <groupId>org.springdoc</groupId>
        <artifactId>springdoc-openapi-starter-webmvc-ui</artifactId>
        <version>2.5.0</version>
    </dependency>

    <!-- ======= MONITORING ======= -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-actuator</artifactId>
    </dependency>

    <!-- ======= DEV TOOLS ======= -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-devtools</artifactId>
        <scope>runtime</scope>
        <optional>true</optional>
    </dependency>

    <dependency>
        <groupId>org.projectlombok</groupId>
        <artifactId>lombok</artifactId>
        <optional>true</optional>
    </dependency>

    <!-- ======= TEST ======= -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-test</artifactId>
        <scope>test</scope>
    </dependency>
    <dependency>
        <groupId>org.springframework.security</groupId>
        <artifactId>spring-security-test</artifactId>
        <scope>test</scope>
    </dependency>
    <dependency>
        <groupId>org.testcontainers</groupId>
        <artifactId>mysql</artifactId>
        <scope>test</scope>
    </dependency>
    <dependency>
        <groupId>org.testcontainers</groupId>
        <artifactId>junit-jupiter</artifactId>
        <scope>test</scope>
    </dependency>
</dependencies>

<build>
    <plugins>
        <plugin>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-maven-plugin</artifactId>
            <configuration>
                <excludes>
                    <exclude>
                        <groupId>org.projectlombok</groupId>
                        <artifactId>lombok</artifactId>
                    </exclude>
                </excludes>
            </configuration>
        </plugin>

        <!-- MapStruct + Lombok compiler plugin -->
        <plugin>
            <groupId>org.apache.maven.plugins</groupId>
            <artifactId>maven-compiler-plugin</artifactId>
            <configuration>
                <annotationProcessorPaths>
                    <path>
                        <groupId>org.projectlombok</groupId>
                        <artifactId>lombok</artifactId>
                    </path>
                    <path>
                        <groupId>org.mapstruct</groupId>
                        <artifactId>mapstruct-processor</artifactId>
                        <version>${mapstruct.version}</version>
                    </path>
                    <!-- Lombok + MapStruct binding -->
                    <path>
                        <groupId>org.projectlombok</groupId>
                        <artifactId>lombok-mapstruct-binding</artifactId>
                        <version>0.2.0</version>
                    </path>
                </annotationProcessorPaths>
                <compilerArgs>
                    <arg>-Amapstruct.defaultComponentModel=spring</arg>
                </compilerArgs>
            </configuration>
        </plugin>
    </plugins>
</build>
```

---

## 6. TẠO CẤU TRÚC THƯ MỤC

### 6.1. Tạo tất cả packages

Trong IntelliJ IDEA, tạo các package sau trong `src/main/java/com/example/ecommerce/`:

```bash
# Tạo bằng terminal (hoặc dùng IDE)
cd src/main/java/com/example/ecommerce

mkdir -p config
mkdir -p entity/enums
mkdir -p repository/specification
mkdir -p service
mkdir -p controller
mkdir -p dto/request
mkdir -p dto/response
mkdir -p dto/mapper
mkdir -p security
mkdir -p exception
mkdir -p util
```

### 6.2. Tạo Enum classes

```java
// entity/enums/UserRole.java
package com.example.ecommerce.entity.enums;

public enum UserRole {
    CUSTOMER,    // Khách hàng
    ADMIN,       // Quản trị viên
    STAFF        // Nhân viên
}
```

```java
// entity/enums/OrderStatus.java
package com.example.ecommerce.entity.enums;

public enum OrderStatus {
    PENDING,       // Chờ xử lý
    CONFIRMED,     // Đã xác nhận
    PROCESSING,    // Đang xử lý
    SHIPPING,      // Đang giao hàng
    DELIVERED,     // Đã giao hàng
    CANCELLED,     // Đã hủy
    REFUNDED       // Đã hoàn tiền
}
```

```java
// entity/enums/PaymentStatus.java
package com.example.ecommerce.entity.enums;

public enum PaymentStatus {
    PENDING,     // Chờ thanh toán
    PAID,        // Đã thanh toán
    FAILED,      // Thanh toán thất bại
    REFUNDED     // Đã hoàn tiền
}
```

```java
// entity/enums/PaymentMethod.java
package com.example.ecommerce.entity.enums;

public enum PaymentMethod {
    COD,           // Thanh toán khi nhận hàng
    BANK_TRANSFER, // Chuyển khoản ngân hàng
    CREDIT_CARD,   // Thẻ tín dụng
    E_WALLET,      // Ví điện tử (MoMo, ZaloPay)
    VNPAY          // VNPay
}
```

---

## 7. CẤU HÌNH PROFILES

### 7.1. Tách cấu hình theo môi trường

```yaml
# src/main/resources/application.yml (config chung)
spring:
  application:
    name: ecommerce-api
  profiles:
    active: dev  # Profile mặc định

server:
  port: 8080

# Logging chung
logging:
  pattern:
    console: "%d{yyyy-MM-dd HH:mm:ss} [%thread] %-5level %logger{36} - %msg%n"
```

```yaml
# src/main/resources/application-dev.yml (Development)
spring:
  datasource:
    url: jdbc:mysql://localhost:3306/ecommerce?useSSL=false&serverTimezone=UTC&allowPublicKeyRetrieval=true
    username: root
    password: 1q2w3E*
  jpa:
    hibernate:
      ddl-auto: update
    show-sql: true
    properties:
      hibernate:
        format_sql: true
  data:
    redis:
      host: localhost
      port: 6379

logging:
  level:
    com.example.ecommerce: DEBUG
    org.hibernate.SQL: DEBUG
```

```yaml
# src/main/resources/application-prod.yml (Production)
spring:
  datasource:
    url: ${DATABASE_URL}
    username: ${DATABASE_USERNAME}
    password: ${DATABASE_PASSWORD}
  jpa:
    hibernate:
      ddl-auto: none  # KHÔNG tự động thay đổi schema
    show-sql: false
  data:
    redis:
      host: ${REDIS_HOST}
      port: ${REDIS_PORT}
      password: ${REDIS_PASSWORD}

logging:
  level:
    com.example.ecommerce: INFO
    org.hibernate.SQL: WARN
```

### 7.2. Chạy với profile cụ thể

```bash
# Development (mặc định)
./mvnw spring-boot:run

# Production
./mvnw spring-boot:run -Dspring-boot.run.profiles=prod

# Hoặc qua biến môi trường
SPRING_PROFILES_ACTIVE=prod java -jar ecommerce-0.0.1-SNAPSHOT.jar
```

---

## 8. BÀI TẬP & CHECKLIST

### Bài tập thực hành

- [ ] Tạo tất cả packages theo cấu trúc ở mục 2.1
- [ ] Tạo `BaseEntity.java` với id, createdAt, updatedAt
- [ ] Tạo `ApiResponse.java` với factory methods
- [ ] Tạo `PagedResponse.java`
- [ ] Tạo 4 Enum classes (UserRole, OrderStatus, PaymentStatus, PaymentMethod)
- [ ] Cập nhật `pom.xml` với tất cả dependencies
- [ ] Tạo `application.yml`, `application-dev.yml`, `application-prod.yml`
- [ ] Chạy `mvn clean compile` - không có lỗi
- [ ] Chạy ứng dụng - kết nối MySQL thành công

### Kiến thức cần nắm

| Khái niệm | Hiểu chưa? |
|-----------|-------------|
| Layered Architecture (4 layers) | ☐ |
| Dependency Rule (hướng phụ thuộc) | ☐ |
| Package by Layer vs Package by Feature | ☐ |
| @MappedSuperclass vs @Entity | ☐ |
| Lombok annotations (@Data, @Builder, @Slf4j) | ☐ |
| Tại sao KHÔNG dùng @Data cho Entity | ☐ |
| Maven lifecycle (clean, compile, test, package) | ☐ |
| Dependency scope (compile, runtime, test) | ☐ |
| Spring Profiles (dev, prod) | ☐ |
| @RequiredArgsConstructor (Constructor Injection) | ☐ |

### Lỗi thường gặp

| Lỗi | Nguyên nhân | Cách sửa |
|-----|-------------|----------|
| `Lombok annotation not working` | IDE chưa enable annotation processing | IntelliJ: Settings → Build → Compiler → Annotation Processors → Enable |
| `MapStruct not generating` | Thiếu annotation processor trong pom.xml | Thêm mapstruct-processor trong maven-compiler-plugin |
| `Cannot resolve symbol @Slf4j` | Chưa cài Lombok plugin | IntelliJ: Install Lombok plugin + restart |
| `application.yml not recognized` | File nằm sai vị trí | Phải nằm trong src/main/resources/ |
| `Profile not switching` | Thiếu spring.profiles.active | Check application.yml hoặc set qua command line |

---

**Trước đó:** [← Ngày 01 - Spring Boot Overview](day-01-spring-boot-overview.md)

**Tiếp theo:** [Ngày 03 - Spring IoC & Dependency Injection →](day-03-spring-ioc-di.md)
