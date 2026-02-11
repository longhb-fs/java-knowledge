# Day 1: Foundation + Project Setup

> **Mục tiêu:** Hiểu Java ecosystem, tạo Spring Boot project, chạy được Hello World API.
> **Thời gian:** ~4-6 giờ | **Output:** `curl localhost:8080/api/v1/hello` trả về JSON.

---

## 1. Java Ecosystem Tóm Tắt

### 1.1 JVM / JRE / JDK — Chúng khác nhau thế nào?

```
┌─────────────────────────────────────────────────────┐
│                      JDK (Java Development Kit)      │
│  ┌───────────────────────────────────────────────┐  │
│  │              JRE (Java Runtime Environment)    │  │
│  │  ┌─────────────────────────────────────────┐  │  │
│  │  │           JVM (Java Virtual Machine)     │  │  │
│  │  │                                          │  │  │
│  │  │  ┌──────────┐  ┌───────────────────┐    │  │  │
│  │  │  │Class     │  │ Execution Engine   │    │  │  │
│  │  │  │Loader    │  │ (JIT Compiler +    │    │  │  │
│  │  │  │System    │  │  Garbage Collector) │    │  │  │
│  │  │  └──────────┘  └───────────────────┘    │  │  │
│  │  │                                          │  │  │
│  │  │  ┌──────────────────────────────────┐   │  │  │
│  │  │  │ Runtime Memory Areas              │   │  │  │
│  │  │  │ (Heap, Stack, Metaspace)          │   │  │  │
│  │  │  └──────────────────────────────────┘   │  │  │
│  │  └─────────────────────────────────────────┘  │  │
│  │                                                │  │
│  │  + Core Libraries (java.lang, java.util, ...)  │  │
│  │  + java (runtime command)                      │  │
│  └───────────────────────────────────────────────┘  │
│                                                      │
│  + javac (compiler)     + jdb (debugger)             │
│  + jar (archiver)       + javadoc (docs generator)   │
│  + jshell (REPL)        + jlink (custom runtime)     │
└─────────────────────────────────────────────────────┘
```

### 1.2 So Sánh Nhanh

| Thành phần | Chức năng | Ai cần? |
|-----------|-----------|---------|
| **JVM** | Chạy bytecode (.class), quản lý memory, garbage collection | Mọi ứng dụng Java |
| **JRE** | JVM + thư viện chuẩn (java.lang, java.util...) | User chạy app Java |
| **JDK** | JRE + compiler (javac) + dev tools (jdb, jar, jshell) | **Developer** (chúng ta) |

> **Ghi nhớ:** Luôn cài **JDK** khi dev. Dự án này dùng **Java 21 LTS**.

### 1.3 Build Flow

```
                    javac                JVM
Source Code ───────────────▶ Bytecode ─────────▶ Machine Code
 (.java)      compile time    (.class)   runtime    (native)
                                │
                         ┌──────┴──────┐
                         │  .jar / .war │  ← Maven đóng gói
                         └─────────────┘
```

Kiểm tra đã cài đúng chưa:

```bash
java -version
# openjdk version "21.0.x" ...

mvn -version
# Apache Maven 3.9.x ...

docker --version
# Docker version 2x.x.x ...
```

---

## 2. Spring Boot Là Gì?

### 2.1 Spring Framework vs Spring Boot

| Tiêu chí | Spring Framework | Spring Boot |
|----------|-----------------|-------------|
| Config | XML hoặc Java Config **thủ công** | **Auto-configuration** |
| Server | Phải cài Tomcat riêng, deploy WAR | **Embedded Tomcat** đi kèm |
| Dependencies | Tự quản lý version từng thư viện | **Starter POMs** quản lý hết |
| Khởi tạo project | Copy template cũ, config nhiều | **Spring Initializr** 1 phút xong |
| Production | Tự viết health check, metrics | **Actuator** có sẵn |
| Learning curve | Cao — cần hiểu nhiều concept | Thấp hơn — convention over config |

**Tóm lại:** Spring Boot = Spring Framework + Auto-config + Embedded Server + Opinionated Defaults.

### 2.2 Auto-Configuration Hoạt Động Thế Nào?

```
┌─────────────────────────────────────────────────────────────┐
│                   Spring Boot Startup Flow                   │
│                                                              │
│  1. @SpringBootApplication                                   │
│     │                                                        │
│     ├── @SpringBootConfiguration  (= @Configuration)         │
│     ├── @EnableAutoConfiguration  (★ magic ở đây)            │
│     └── @ComponentScan            (scan package hiện tại)    │
│                                                              │
│  2. Spring Boot đọc META-INF/spring/                         │
│     org.springframework.boot.autoconfigure.AutoConfiguration │
│     → Tìm thấy 100+ AutoConfiguration classes               │
│                                                              │
│  3. Kiểm tra @Conditional:                                   │
│     ├── @ConditionalOnClass(DataSource.class)                │
│     │   → Có mysql-connector trong classpath? → CÓ           │
│     │   → Auto-config DataSource ✓                           │
│     │                                                        │
│     ├── @ConditionalOnClass(RedisTemplate.class)             │
│     │   → Có spring-data-redis? → CÓ                        │
│     │   → Auto-config Redis ✓                                │
│     │                                                        │
│     └── @ConditionalOnMissingBean(SecurityFilterChain.class) │
│         → Chưa có custom bean? → Auto-config default ✓       │
│                                                              │
│  4. Kết quả: App chạy với đầy đủ config mà không cần        │
│     viết XML hay Java Config thủ công!                       │
└─────────────────────────────────────────────────────────────┘
```

**Nguyên tắc cốt lõi:**
- Nếu có library trong classpath → Spring Boot **tự cấu hình**
- Nếu bạn define bean riêng → Spring Boot **nhường** cho bạn
- Bạn chỉ cần override khi muốn custom

---

## 3. Tạo Project

### 3.1 Spring Initializr

Truy cập [start.spring.io](https://start.spring.io) và chọn:

| Setting | Value |
|---------|-------|
| Project | **Maven** |
| Language | **Java** |
| Spring Boot | **3.3.x** (latest stable) |
| Group | `com.example` |
| Artifact | `ecommerce` |
| Name | `ecommerce` |
| Package name | `com.example.ecommerce` |
| Packaging | **Jar** |
| Java | **21** |

Dependencies chọn trên UI: `Spring Web`, `Spring Data JPA`, `Spring Security`, `Validation`, `MySQL Driver`, `Lombok`, `Spring Boot DevTools`, `Flyway Migration`.

> Sau khi generate, ta sẽ thêm thủ công các dependency còn lại vào `pom.xml`.

### 3.2 Full pom.xml

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0
         https://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <!-- ============================== -->
    <!-- Parent: Spring Boot BOM        -->
    <!-- ============================== -->
    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>3.3.5</version>
        <relativePath/>
    </parent>

    <!-- ============================== -->
    <!-- Project Info                   -->
    <!-- ============================== -->
    <groupId>com.example</groupId>
    <artifactId>ecommerce</artifactId>
    <version>0.0.1-SNAPSHOT</version>
    <name>ecommerce</name>
    <description>E-Commerce API - Spring Boot 7-Day Speed Run</description>

    <!-- ============================== -->
    <!-- Properties                     -->
    <!-- ============================== -->
    <properties>
        <java.version>21</java.version>
        <mapstruct.version>1.5.5.Final</mapstruct.version>
        <lombok-mapstruct-binding.version>0.2.0</lombok-mapstruct-binding.version>
        <jjwt.version>0.12.6</jjwt.version>
        <springdoc.version>2.6.0</springdoc.version>
    </properties>

    <!-- ============================== -->
    <!-- Dependencies                   -->
    <!-- ============================== -->
    <dependencies>

        <!-- ── Web (Embedded Tomcat + REST) ── -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>

        <!-- ── JPA + Hibernate (ORM) ── -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-data-jpa</artifactId>
        </dependency>

        <!-- ── Security (Authentication + Authorization) ── -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-security</artifactId>
        </dependency>

        <!-- ── Validation (Bean Validation / Hibernate Validator) ── -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-validation</artifactId>
        </dependency>

        <!-- ── MySQL Driver ── -->
        <dependency>
            <groupId>com.mysql</groupId>
            <artifactId>mysql-connector-j</artifactId>
            <scope>runtime</scope>
        </dependency>

        <!-- ── Lombok (giảm boilerplate code) ── -->
        <dependency>
            <groupId>org.projectlombok</groupId>
            <artifactId>lombok</artifactId>
            <optional>true</optional>
        </dependency>

        <!-- ── MapStruct (object mapping) ── -->
        <dependency>
            <groupId>org.mapstruct</groupId>
            <artifactId>mapstruct</artifactId>
            <version>${mapstruct.version}</version>
        </dependency>

        <!-- ── JWT (JSON Web Token) ── -->
        <dependency>
            <groupId>io.jsonwebtoken</groupId>
            <artifactId>jjwt-api</artifactId>
            <version>${jjwt.version}</version>
        </dependency>
        <dependency>
            <groupId>io.jsonwebtoken</groupId>
            <artifactId>jjwt-impl</artifactId>
            <version>${jjwt.version}</version>
            <scope>runtime</scope>
        </dependency>
        <dependency>
            <groupId>io.jsonwebtoken</groupId>
            <artifactId>jjwt-jackson</artifactId>
            <version>${jjwt.version}</version>
            <scope>runtime</scope>
        </dependency>

        <!-- ── Flyway (Database Migration) ── -->
        <dependency>
            <groupId>org.flywaydb</groupId>
            <artifactId>flyway-core</artifactId>
        </dependency>
        <dependency>
            <groupId>org.flywaydb</groupId>
            <artifactId>flyway-mysql</artifactId>
        </dependency>

        <!-- ── SpringDoc OpenAPI (Swagger UI) ── -->
        <dependency>
            <groupId>org.springdoc</groupId>
            <artifactId>springdoc-openapi-starter-webmvc-ui</artifactId>
            <version>${springdoc.version}</version>
        </dependency>

        <!-- ── Redis (Caching) ── -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-data-redis</artifactId>
        </dependency>

        <!-- ── DevTools (auto-restart khi dev) ── -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-devtools</artifactId>
            <scope>runtime</scope>
            <optional>true</optional>
        </dependency>

        <!-- ── Testing ── -->
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

        <!-- ── TestContainers (DB test thật trong Docker) ── -->
        <dependency>
            <groupId>org.testcontainers</groupId>
            <artifactId>junit-jupiter</artifactId>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>org.testcontainers</groupId>
            <artifactId>mysql</artifactId>
            <scope>test</scope>
        </dependency>

    </dependencies>

    <!-- ============================== -->
    <!-- Build Configuration            -->
    <!-- ============================== -->
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

            <!-- ── Maven Compiler: Lombok + MapStruct ── -->
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-compiler-plugin</artifactId>
                <configuration>
                    <source>${java.version}</source>
                    <target>${java.version}</target>
                    <annotationProcessorPaths>
                        <!-- Lombok PHẢI trước MapStruct -->
                        <path>
                            <groupId>org.projectlombok</groupId>
                            <artifactId>lombok</artifactId>
                            <version>${lombok.version}</version>
                        </path>
                        <!-- Binding giữa Lombok và MapStruct -->
                        <path>
                            <groupId>org.projectlombok</groupId>
                            <artifactId>lombok-mapstruct-binding</artifactId>
                            <version>${lombok-mapstruct-binding.version}</version>
                        </path>
                        <!-- MapStruct processor -->
                        <path>
                            <groupId>org.mapstruct</groupId>
                            <artifactId>mapstruct-processor</artifactId>
                            <version>${mapstruct.version}</version>
                        </path>
                    </annotationProcessorPaths>
                    <compilerArgs>
                        <arg>-Amapstruct.defaultComponentModel=spring</arg>
                    </compilerArgs>
                </configuration>
            </plugin>
        </plugins>
    </build>

</project>
```

> **Lưu ý quan trọng:** Trong `annotationProcessorPaths`, thứ tự **Lombok -> lombok-mapstruct-binding -> MapStruct** là bắt buộc. Nếu đảo thứ tự, MapStruct sẽ không nhận được getter/setter mà Lombok generate.

---

## 4. Clean Architecture Layout

### 4.1 Full Package Structure

```
src/main/java/com/example/ecommerce/
├── EcommerceApplication.java          # Entry point - @SpringBootApplication
│
├── config/                            # Cấu hình app (Security, Redis, CORS, OpenAPI)
│   ├── SecurityConfig.java
│   ├── RedisConfig.java
│   ├── OpenApiConfig.java
│   └── WebConfig.java
│
├── controller/                        # REST Controllers - nhận request, trả response
│   ├── AuthController.java
│   ├── CategoryController.java
│   ├── ProductController.java
│   ├── CartController.java
│   └── OrderController.java
│
├── dto/                               # Data Transfer Objects - không expose entity ra ngoài
│   ├── request/                       # Input DTOs (client gửi lên)
│   │   ├── LoginRequest.java
│   │   ├── RegisterRequest.java
│   │   └── ProductCreateRequest.java
│   └── response/                      # Output DTOs (server trả về)
│       ├── ApiResponse.java
│       ├── PagedResponse.java
│       ├── AuthResponse.java
│       └── ProductResponse.java
│
├── entity/                            # JPA Entities - map trực tiếp với database tables
│   ├── BaseEntity.java
│   ├── User.java
│   ├── Category.java
│   ├── Product.java
│   ├── ProductImage.java
│   ├── CartItem.java
│   ├── Order.java
│   ├── OrderItem.java
│   └── Payment.java
│
├── enums/                             # Enum constants - trạng thái, vai trò
│   ├── Role.java
│   ├── OrderStatus.java
│   └── PaymentMethod.java
│
├── exception/                         # Exception handling - lỗi thống nhất format
│   ├── ResourceNotFoundException.java
│   ├── DuplicateResourceException.java
│   ├── BadRequestException.java
│   └── GlobalExceptionHandler.java
│
├── mapper/                            # MapStruct mappers - convert Entity <-> DTO
│   ├── ProductMapper.java
│   ├── CategoryMapper.java
│   └── UserMapper.java
│
├── repository/                        # Spring Data JPA repos - truy vấn database
│   ├── UserRepository.java
│   ├── CategoryRepository.java
│   ├── ProductRepository.java
│   └── OrderRepository.java
│
├── security/                          # JWT + Spring Security components
│   ├── JwtTokenProvider.java
│   ├── JwtAuthenticationFilter.java
│   └── CustomUserDetailsService.java
│
├── service/                           # Business logic - xử lý nghiệp vụ
│   ├── AuthService.java
│   ├── CategoryService.java
│   ├── ProductService.java
│   ├── CartService.java
│   └── OrderService.java
│
├── specification/                     # JPA Specifications - dynamic query filters
│   └── ProductSpecification.java
│
└── util/                              # Utility classes - helper functions
    └── SlugUtils.java
```

### 4.2 Request Flow

```
Client Request
    │
    ▼
┌──────────────┐   validate    ┌──────────┐   logic    ┌────────────┐   query   ┌────────────┐
│  Controller  │──────────────▶│  Service  │──────────▶│ Repository │─────────▶│  Database  │
│  (REST API)  │               │ (Business)│           │  (JPA)     │          │  (MySQL)   │
└──────────────┘               └──────────┘            └────────────┘          └────────────┘
    │  ▲                           │  ▲
    │  │                           │  │
    ▼  │         DTO               ▼  │        Entity
┌──────────┐  ◄──────────▶   ┌──────────┐
│   DTO    │   MapStruct     │  Entity  │
│ (Request/│                 │  (JPA)   │
│ Response)│                 └──────────┘
└──────────┘
```

**Quy tắc vàng:** Controller KHÔNG bao giờ truy cập Repository trực tiếp. Entity KHÔNG bao giờ lộ ra ngoài Service layer.

---

## 5. Lombok Essentials

### 5.1 Top 5 Annotations Cần Biết

```java
import lombok.*;

// 1. @Getter - tự generate getter cho tất cả fields
@Getter
public class Product {
    private String name;    // → getName()
    private BigDecimal price; // → getPrice()
}

// 2. @Setter - tự generate setter cho tất cả fields
@Setter
public class Product {
    private String name;    // → setName(String name)
}

// 3. @NoArgsConstructor - constructor không tham số
@NoArgsConstructor
public class Product {
    // → public Product() {}
}

// 4. @AllArgsConstructor - constructor với tất cả fields
@AllArgsConstructor
public class Product {
    private String name;
    private BigDecimal price;
    // → public Product(String name, BigDecimal price) {}
}

// 5. @Builder - Builder pattern cho khởi tạo object
@Builder
public class Product {
    private String name;
    private BigDecimal price;
}
// Sử dụng:
// Product p = Product.builder().name("iPhone").price(new BigDecimal("999")).build();
```

### 5.2 WARNING: @Data Trên JPA Entity

`@Data` = `@Getter` + `@Setter` + `@ToString` + `@EqualsAndHashCode` + `@RequiredArgsConstructor`

**Vấn đề:** `@EqualsAndHashCode` dùng tất cả fields, bao gồm các relationship (lazy-loaded). Điều này gây ra:
- **LazyInitializationException** khi so sánh entity ngoài transaction
- **StackOverflowError** khi 2 entity có quan hệ 2 chiều (circular reference)
- **Performance issue** khi load tất cả fields chỉ để so sánh

### BAD - Không làm thế này

```java
@Data  // ← NGUY HIỂM với JPA Entity!
@Entity
@Table(name = "products")
public class Product {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String name;

    @ManyToOne(fetch = FetchType.LAZY)
    private Category category;  // ← @Data sẽ gọi category.hashCode()
                                //    → LazyInitializationException!
}
```

### GOOD - Làm thế này

```java
@Getter
@Setter
@NoArgsConstructor
@AllArgsConstructor
@Builder
@Entity
@Table(name = "products")
public class Product {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String name;

    @ManyToOne(fetch = FetchType.LAZY)
    private Category category;

    // equals/hashCode chỉ dùng id - an toàn với JPA
    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (!(o instanceof Product other)) return false;
        return id != null && id.equals(other.getId());
    }

    @Override
    public int hashCode() {
        return getClass().hashCode(); // constant hashCode - an toàn cho Set/Map
    }
}
```

> **Quy tắc:** Trên JPA Entity, LUÔN dùng `@Getter @Setter` riêng lẻ, KHÔNG dùng `@Data`. Tự viết `equals/hashCode` chỉ dùng ID.

---

## 6. Base Classes

### 6.1 BaseEntity

```java
package com.example.ecommerce.entity;

import jakarta.persistence.*;
import lombok.Getter;
import lombok.Setter;
import java.time.Instant;

@Getter
@Setter
@MappedSuperclass  // Không tạo table riêng, chỉ để các entity khác kế thừa
public abstract class BaseEntity {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(name = "created_at", nullable = false, updatable = false)
    private Instant createdAt;

    @Column(name = "updated_at", nullable = false)
    private Instant updatedAt;

    /**
     * Tự động set createdAt và updatedAt khi INSERT
     */
    @PrePersist
    protected void onCreate() {
        Instant now = Instant.now();
        this.createdAt = now;
        this.updatedAt = now;
    }

    /**
     * Tự động update updatedAt khi UPDATE
     */
    @PreUpdate
    protected void onUpdate() {
        this.updatedAt = Instant.now();
    }

    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (!(o instanceof BaseEntity other)) return false;
        return id != null && id.equals(other.getId());
    }

    @Override
    public int hashCode() {
        return getClass().hashCode();
    }
}
```

### 6.2 ApiResponse<T>

```java
package com.example.ecommerce.dto.response;

import com.fasterxml.jackson.annotation.JsonInclude;
import lombok.AllArgsConstructor;
import lombok.Builder;
import lombok.Getter;
import lombok.NoArgsConstructor;

@Getter
@Builder
@NoArgsConstructor
@AllArgsConstructor
@JsonInclude(JsonInclude.Include.NON_NULL)  // Không trả field null trong JSON
public class ApiResponse<T> {

    private boolean success;
    private String message;
    private T data;

    // ── Factory Methods ──

    /**
     * Thành công với data
     * ApiResponse.success("OK", productDto)
     */
    public static <T> ApiResponse<T> success(String message, T data) {
        return ApiResponse.<T>builder()
                .success(true)
                .message(message)
                .data(data)
                .build();
    }

    /**
     * Thành công chỉ có message
     * ApiResponse.success("Deleted successfully")
     */
    public static <T> ApiResponse<T> success(String message) {
        return ApiResponse.<T>builder()
                .success(true)
                .message(message)
                .build();
    }

    /**
     * Thất bại với message lỗi
     * ApiResponse.error("Product not found")
     */
    public static <T> ApiResponse<T> error(String message) {
        return ApiResponse.<T>builder()
                .success(false)
                .message(message)
                .build();
    }
}
```

### 6.3 PagedResponse<T>

```java
package com.example.ecommerce.dto.response;

import lombok.AllArgsConstructor;
import lombok.Builder;
import lombok.Getter;
import lombok.NoArgsConstructor;
import org.springframework.data.domain.Page;

import java.util.List;
import java.util.function.Function;

@Getter
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class PagedResponse<T> {

    private List<T> content;
    private int page;
    private int size;
    private long totalElements;
    private int totalPages;

    /**
     * Convert từ Page<Entity> sang PagedResponse<DTO>.
     *
     * Sử dụng:
     *   Page<Product> productPage = productRepo.findAll(pageable);
     *   PagedResponse<ProductResponse> response =
     *       PagedResponse.from(productPage, productMapper::toResponse);
     *
     * @param page    Spring Data Page result
     * @param mapper  Function convert Entity -> DTO (thường là MapStruct method reference)
     */
    public static <E, R> PagedResponse<R> from(Page<E> page, Function<E, R> mapper) {
        List<R> content = page.getContent()
                .stream()
                .map(mapper)
                .toList();

        return PagedResponse.<R>builder()
                .content(content)
                .page(page.getNumber())
                .size(page.getSize())
                .totalElements(page.getTotalElements())
                .totalPages(page.getTotalPages())
                .build();
    }
}
```

**Kết quả JSON khi dùng PagedResponse:**

```json
{
  "content": [
    { "id": 1, "name": "iPhone 15", "price": 999.00 },
    { "id": 2, "name": "Samsung S24", "price": 899.00 }
  ],
  "page": 0,
  "size": 10,
  "totalElements": 52,
  "totalPages": 6
}
```

---

## 7. Docker Compose

Tạo file `docker-compose.yml` ở root project:

```yaml
version: "3.8"

services:
  # ── MySQL 8.0 ──
  mysql:
    image: mysql:8.0
    container_name: ecommerce-mysql
    restart: unless-stopped
    environment:
      MYSQL_ROOT_PASSWORD: root123
      MYSQL_DATABASE: ecommerce_db
      MYSQL_USER: ecommerce
      MYSQL_PASSWORD: ecommerce123
    ports:
      - "3306:3306"
    volumes:
      - mysql_data:/var/lib/mysql
    command: >
      --character-set-server=utf8mb4
      --collation-server=utf8mb4_unicode_ci
      --default-authentication-plugin=mysql_native_password
    healthcheck:
      test: ["CMD", "mysqladmin", "ping", "-h", "localhost", "-u", "root", "-proot123"]
      interval: 10s
      timeout: 5s
      retries: 5
      start_period: 30s
    networks:
      - ecommerce-net

  # ── Redis 7 ──
  redis:
    image: redis:7-alpine
    container_name: ecommerce-redis
    restart: unless-stopped
    ports:
      - "6379:6379"
    volumes:
      - redis_data:/data
    command: redis-server --appendonly yes --maxmemory 256mb --maxmemory-policy allkeys-lru
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 5s
      retries: 5
      start_period: 10s
    networks:
      - ecommerce-net

# ── Persistent Volumes ──
volumes:
  mysql_data:
    driver: local
  redis_data:
    driver: local

# ── Network ──
networks:
  ecommerce-net:
    driver: bridge
```

Khởi động infrastructure:

```bash
# Start MySQL + Redis (chạy background)
docker compose up -d

# Kiểm tra trạng thái
docker compose ps

# Kết quả mong đợi:
# NAME               STATUS                  PORTS
# ecommerce-mysql    Up (healthy)            0.0.0.0:3306->3306/tcp
# ecommerce-redis    Up (healthy)            0.0.0.0:6379->6379/tcp

# Test kết nối MySQL
docker exec -it ecommerce-mysql mysql -u ecommerce -pecommerce123 -e "SELECT 1"

# Test kết nối Redis
docker exec -it ecommerce-redis redis-cli ping
# → PONG
```

---

## 8. Application Configuration

### 8.1 application.yml (cấu hình chung)

```yaml
# ============================================
# application.yml - Shared Configuration
# ============================================

spring:
  application:
    name: ecommerce-api

  # ── Profiles ──
  profiles:
    active: dev  # Default profile, override bằng biến môi trường

  # ── JPA / Hibernate ──
  jpa:
    open-in-view: false  # Tắt OSIV - best practice cho production
    hibernate:
      ddl-auto: validate  # Production: chỉ validate schema, không tự tạo/sửa
    properties:
      hibernate:
        dialect: org.hibernate.dialect.MySQLDialect
        format_sql: true
        jdbc:
          time_zone: UTC

  # ── Flyway Migration ──
  flyway:
    enabled: true
    locations: classpath:db/migration
    baseline-on-migrate: true

  # ── Jackson (JSON serialization) ──
  jackson:
    serialization:
      write-dates-as-timestamps: false  # ISO-8601 format cho dates
    default-property-inclusion: non_null

# ── Server ──
server:
  port: 8080
  servlet:
    context-path: /

# ── SpringDoc OpenAPI ──
springdoc:
  api-docs:
    path: /api-docs
  swagger-ui:
    path: /swagger-ui.html
    operations-sorter: method

# ── App Custom Properties ──
app:
  jwt:
    secret: my-super-secret-key-that-should-be-at-least-256-bits-long-for-hs256
    expiration-ms: 86400000  # 24 hours
  cors:
    allowed-origins: http://localhost:3000,http://localhost:4200
```

### 8.2 application-dev.yml (override cho development)

```yaml
# ============================================
# application-dev.yml - Development Overrides
# ============================================

spring:
  # ── DataSource (MySQL) ──
  datasource:
    url: jdbc:mysql://localhost:3306/ecommerce_db?useSSL=false&allowPublicKeyRetrieval=true&serverTimezone=UTC
    username: ecommerce
    password: ecommerce123
    driver-class-name: com.mysql.cj.jdbc.Driver
    hikari:
      maximum-pool-size: 10
      minimum-idle: 5
      connection-timeout: 30000

  # ── JPA: cho phép update schema khi dev ──
  jpa:
    hibernate:
      ddl-auto: update  # Dev: tự tạo/sửa table (KHÔNG dùng cho production!)
    show-sql: true

  # ── Flyway: tắt khi dev để dùng ddl-auto ──
  flyway:
    enabled: false  # Bật lại khi dùng Flyway migration (Day 3)

  # ── Redis ──
  data:
    redis:
      host: localhost
      port: 6379
      timeout: 60000ms

  # ── DevTools ──
  devtools:
    restart:
      enabled: true
    livereload:
      enabled: true

# ── Logging (verbose khi dev) ──
logging:
  level:
    com.example.ecommerce: DEBUG
    org.springframework.security: DEBUG
    org.hibernate.SQL: DEBUG
    org.hibernate.orm.jdbc.bind: TRACE  # Log query parameters
```

> **Giải thích profiles:** Khi chạy `--spring.profiles.active=dev`, Spring Boot sẽ load `application.yml` trước, rồi `application-dev.yml` **đè chồng** (override) lên. Production sẽ dùng `application-prod.yml` với cấu hình khác.

### 8.3 Tạm thời tắt Security (để test Hello World)

Spring Security mặc định **block tất cả** request. Để Day 1 test được, ta tạm tắt:

```java
package com.example.ecommerce.config;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.config.annotation.web.configuration.EnableWebSecurity;
import org.springframework.security.config.annotation.web.configurers.AbstractHttpConfigurer;
import org.springframework.security.config.http.SessionCreationPolicy;
import org.springframework.security.web.SecurityFilterChain;

@Configuration
@EnableWebSecurity
public class SecurityConfig {

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            .csrf(AbstractHttpConfigurer::disable)
            .sessionManagement(session ->
                session.sessionCreationPolicy(SessionCreationPolicy.STATELESS)
            )
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/api/v1/hello").permitAll()
                .requestMatchers("/swagger-ui/**", "/api-docs/**").permitAll()
                .anyRequest().authenticated()
            );

        return http.build();
    }
}
```

> Day 5 sẽ cấu hình Security đầy đủ với JWT.

---

## 9. Hello World API

### 9.1 Main Application

```java
package com.example.ecommerce;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication  // = @Configuration + @EnableAutoConfiguration + @ComponentScan
public class EcommerceApplication {

    public static void main(String[] args) {
        SpringApplication.run(EcommerceApplication.class, args);
    }
}
```

### 9.2 HelloController

```java
package com.example.ecommerce.controller;

import com.example.ecommerce.dto.response.ApiResponse;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

@RestController                    // = @Controller + @ResponseBody (trả JSON mặc định)
@RequestMapping("/api/v1")         // Prefix cho tất cả endpoints trong controller này
public class HelloController {

    @GetMapping("/hello")
    public ApiResponse<String> hello() {
        return ApiResponse.success("Hello Spring Boot!", "v1.0.0");
    }
}
```

**Giải thích annotations:**

| Annotation | Chức năng |
|-----------|-----------|
| `@RestController` | Đánh dấu class là REST controller, tự động serialize return value sang JSON |
| `@RequestMapping("/api/v1")` | Đặt base path cho tất cả endpoints trong controller |
| `@GetMapping("/hello")` | Map HTTP GET request đến method này, full path = `/api/v1/hello` |

---

## 10. Build, Run và Verify

### 10.1 Start Infrastructure

```bash
# Đảm bảo Docker đang chạy, rồi:
docker compose up -d

# Đợi MySQL healthy (khoảng 20-30 giây)
docker compose ps
```

### 10.2 Build và Run

```bash
# Cách 1: Maven wrapper (khuyên dùng)
./mvnw spring-boot:run -Dspring-boot.run.profiles=dev

# Cách 2: Build jar rồi chạy
./mvnw clean package -DskipTests
java -jar target/ecommerce-0.0.1-SNAPSHOT.jar --spring.profiles.active=dev

# Cách 3: Từ IDE (IntelliJ)
# Click nút Run bên cạnh method main() trong EcommerceApplication.java
```

Console output thành công:

```
  .   ____          _            __ _ _
 /\\ / ___'_ __ _ _(_)_ __  __ _ \ \ \ \
( ( )\___ | '_ | '_| | '_ \/ _` | \ \ \ \
 \\/  ___)| |_)| | | | | || (_| |  ) ) ) )
  '  |____| .__|_| |_|_| |_\__, | / / / /
 =========|_|==============|___/=/_/_/_/

 :: Spring Boot ::                (v3.3.5)

... Tomcat started on port 8080 (http)
... Started EcommerceApplication in 3.xxx seconds
```

### 10.3 Verify với curl

```bash
# Test Hello API
curl -s http://localhost:8080/api/v1/hello | python -m json.tool
```

**Kết quả mong đợi:**

```json
{
    "success": true,
    "message": "Hello Spring Boot!",
    "data": "v1.0.0"
}
```

```bash
# Test Swagger UI (mở trên browser)
# http://localhost:8080/swagger-ui.html

# Test endpoint không có quyền (sẽ bị 403)
curl -s -o /dev/null -w "%{http_code}" http://localhost:8080/api/v1/products
# → 403
```

### 10.4 Troubleshooting

| Vấn đề | Nguyên nhân | Cách xử lý |
|--------|------------|------------|
| `Port 3306 already in use` | MySQL local đang chạy | Tắt MySQL local hoặc đổi port trong docker-compose |
| `Access denied for user` | Sai credentials | Kiểm tra lại username/password trong application-dev.yml |
| `Connection refused` | MySQL chưa ready | Đợi 30 giây hoặc chạy `docker compose logs mysql` |
| `No qualifying bean of type 'DataSource'` | Thiếu MySQL driver | Kiểm tra pom.xml có `mysql-connector-j` |
| `403 Forbidden` cho `/api/v1/hello` | SecurityConfig chưa apply | Kiểm tra `@Configuration` và `permitAll()` |

---

## Tổng Kết Day 1

### Đã Học

| # | Kiến thức | Trạng thái |
|---|----------|-----------|
| 1 | JVM/JRE/JDK và Java ecosystem | Hiểu |
| 2 | Spring Boot vs Spring Framework | Hiểu |
| 3 | Auto-configuration mechanism | Hiểu |
| 4 | Tạo project với Spring Initializr + pom.xml | Làm được |
| 5 | Clean Architecture package layout | Hiểu |
| 6 | Lombok annotations + @Data warning | Hiểu |
| 7 | BaseEntity, ApiResponse, PagedResponse | Code xong |
| 8 | Docker Compose (MySQL + Redis) | Chạy được |
| 9 | application.yml + profiles | Config xong |
| 10 | Hello World REST API | Chạy được |

### Files Đã Tạo

```
ecommerce/
├── docker-compose.yml
├── pom.xml
└── src/main/
    ├── java/com/example/ecommerce/
    │   ├── EcommerceApplication.java
    │   ├── config/
    │   │   └── SecurityConfig.java
    │   ├── controller/
    │   │   └── HelloController.java
    │   ├── dto/response/
    │   │   ├── ApiResponse.java
    │   │   └── PagedResponse.java
    │   └── entity/
    │       └── BaseEntity.java
    └── resources/
        ├── application.yml
        └── application-dev.yml
```

### Bài Tập Tự Luyện

1. Thêm endpoint `GET /api/v1/hello/{name}` trả về `"Hello {name}!"` dùng `@PathVariable`
2. Thêm endpoint `GET /api/v1/time` trả về thời gian hiện tại (Instant.now())
3. Thử thay đổi `ddl-auto` từ `update` sang `validate` và xem lỗi gì xảy ra
4. Mở Swagger UI và thử call API từ giao diện web

---

> **Tiếp theo:** [Day 2 - IoC/DI + Config + JPA Setup](./day-02-ioc-di-jpa.md) — Học Dependency Injection, tạo Category và Product entities.
