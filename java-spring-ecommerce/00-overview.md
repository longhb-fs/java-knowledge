# JAVA SPRING BOOT E-COMMERCE - KHÓA HỌC TOÀN DIỆN

## Mục tiêu khóa học

Xây dựng **hệ thống E-Commerce hoàn chỉnh** với Java Spring Boot từ con số 0, bao gồm:
- REST API backend với Spring Boot 3.x
- Database layer với JPA/Hibernate + MySQL
- Authentication/Authorization với Spring Security + JWT
- Business logic: Product, Cart, Order, Payment, Email
- Testing: Unit Test, Integration Test
- DevOps: Docker, CI/CD, Performance Tuning

---

## Technology Stack

| Lớp | Công nghệ | Phiên bản |
|-----|-----------|-----------|
| **Language** | Java | 21 (LTS) |
| **Framework** | Spring Boot | 3.2+ |
| **Build Tool** | Maven | 3.9+ |
| **ORM** | Spring Data JPA + Hibernate | 6.x |
| **Database** | MySQL | 8.0 |
| **Cache** | Redis | 7.x |
| **Security** | Spring Security + JWT | 6.x |
| **API Docs** | SpringDoc OpenAPI (Swagger) | 2.x |
| **Mapping** | MapStruct | 1.5+ |
| **Validation** | Jakarta Bean Validation | 3.x |
| **Migration** | Flyway | 9.x |
| **Testing** | JUnit 5 + Mockito + TestContainers | 5.x |
| **DevOps** | Docker + Docker Compose + GitHub Actions | latest |
| **Logging** | SLF4J + Logback | 1.4+ |

---

## Cấu trúc khóa học: 30 ngày / 6 tuần

### TUẦN 1: SPRING BOOT FOUNDATION (Nền tảng)

| Ngày | Chủ đề | Nội dung chính |
|------|--------|----------------|
| 01 | Java Ecosystem & Spring Boot Tổng Quan | JDK 21, Maven, Spring Boot architecture, Auto-configuration |
| 02 | Khởi tạo Project | Spring Initializr, cấu trúc project, Clean Architecture |
| 03 | Spring IoC & Dependency Injection | Bean Lifecycle, @Component, @Service, @Repository, @Configuration |
| 04 | Cấu hình Application | application.yml, Profiles (dev/staging/prod), @Value, @ConfigurationProperties |
| 05 | MySQL + Spring Data JPA | DataSource config, JPA setup, first Entity + Repository |

### TUẦN 2: DATABASE & JPA (Cơ sở dữ liệu)

| Ngày | Chủ đề | Nội dung chính |
|------|--------|----------------|
| 06 | JPA Entity & Relationships | @Entity, @Table, @OneToMany, @ManyToOne, @ManyToMany, cascade, fetch |
| 07 | Flyway Migration & Data Seeding | Version-based migration, baseline, CommandLineRunner, seed data |
| 08 | Repository & Custom Queries | JpaRepository, @Query (JPQL + Native SQL), Derived Query Methods |
| 09 | Pagination, Sorting & Specification | Pageable, Sort, JpaSpecificationExecutor, dynamic filtering |
| 10 | JPA Performance | N+1 Problem, @EntityGraph, Lazy vs Eager, Batch Fetching, Query Optimization |

### TUẦN 3: REST API DEVELOPMENT (Phát triển API)

| Ngày | Chủ đề | Nội dung chính |
|------|--------|----------------|
| 11 | REST Controller & HTTP Methods | @RestController, @GetMapping, @PostMapping, ResponseEntity, Status Codes |
| 12 | CRUD API hoàn chỉnh | Product CRUD, Service Layer, Exception Flow, Response Wrapper |
| 13 | DTO Pattern & MapStruct | Request/Response DTO, MapStruct mapping, nested mapping, custom converter |
| 14 | Bean Validation & Error Handling | @Valid, @NotBlank, Custom Validator, @ControllerAdvice, ProblemDetail |
| 15 | Swagger/OpenAPI & Logging | SpringDoc OpenAPI, Swagger UI, SLF4J + Logback, structured logging |

### TUẦN 4: SECURITY & AUTHENTICATION (Bảo mật)

| Ngày | Chủ đề | Nội dung chính |
|------|--------|----------------|
| 16 | Spring Security Fundamentals | SecurityFilterChain, Authentication, Authorization, CORS, CSRF |
| 17 | JWT Authentication | JWT token generation, JwtAuthFilter, login/logout flow |
| 18 | User Registration & Login | BCrypt password encoding, UserDetailsService, signup/signin API |
| 19 | Role-Based Authorization | @PreAuthorize, Method Security, Role hierarchy, Permission model |
| 20 | Refresh Token & OAuth2 | Refresh token rotation, Google/GitHub OAuth2 login, SecurityContext |

### TUẦN 5: E-COMMERCE FEATURES (Tính năng thương mại)

| Ngày | Chủ đề | Nội dung chính |
|------|--------|----------------|
| 21 | Product & Category Management | Category tree, Product variants, Image upload (S3/local), Search |
| 22 | Shopping Cart với Redis | Cart service, Redis session store, add/remove/update items |
| 23 | Order & Checkout Flow | Order entity, Checkout process, Order status state machine, Stock management |
| 24 | Payment Integration | Stripe/VNPay integration, Webhook handling, Payment verification |
| 25 | Email & Notifications | Spring Mail, Thymeleaf email template, Async email, Event-driven notification |

### TUẦN 6: TESTING, DEVOPS & ADVANCED (Kiểm thử & Triển khai)

| Ngày | Chủ đề | Nội dung chính |
|------|--------|----------------|
| 26 | Unit Testing | JUnit 5, Mockito, @MockBean, Service layer testing, TDD approach |
| 27 | Integration Testing | @SpringBootTest, TestContainers (MySQL, Redis), MockMvc, REST Assured |
| 28 | Caching & Async Processing | @Cacheable với Redis, @Async, @Scheduled, CompletableFuture |
| 29 | Docker & Docker Compose | Dockerfile multi-stage, Docker Compose (App + MySQL + Redis), Health checks |
| 30 | CI/CD, Performance & Security Review | GitHub Actions pipeline, JMeter basics, Security checklist, Production readiness |

---

## Database Schema (E-Commerce)

```
┌─────────────┐     ┌─────────────────┐     ┌──────────────┐
│  categories  │     │    products      │     │   product_   │
│─────────────│     │─────────────────│     │   images     │
│ id (PK)     │────<│ id (PK)         │────<│ id (PK)      │
│ name        │     │ category_id (FK)│     │ product_id   │
│ slug        │     │ name            │     │ url          │
│ parent_id   │     │ slug            │     │ sort_order   │
│ description │     │ description     │     └──────────────┘
│ image_url   │     │ price           │
└─────────────┘     │ sale_price      │     ┌──────────────┐
                    │ sku             │     │   reviews    │
                    │ stock_quantity  │     │──────────────│
                    │ is_active       │────<│ id (PK)      │
                    │ created_at      │     │ product_id   │
                    │ updated_at      │     │ user_id      │
                    └─────────────────┘     │ rating       │
                                            │ comment      │
┌─────────────┐     ┌─────────────────┐     └──────────────┘
│   users     │     │    orders       │
│─────────────│     │─────────────────│     ┌──────────────┐
│ id (PK)     │────<│ id (PK)         │────<│ order_items  │
│ email       │     │ user_id (FK)    │     │──────────────│
│ password    │     │ order_number    │     │ id (PK)      │
│ full_name   │     │ status          │     │ order_id     │
│ phone       │     │ total_amount    │     │ product_id   │
│ role        │     │ shipping_addr   │     │ quantity     │
│ is_active   │     │ payment_method  │     │ unit_price   │
│ created_at  │     │ payment_status  │     │ subtotal     │
└─────────────┘     │ created_at      │     └──────────────┘
                    └─────────────────┘
                            │
┌─────────────┐             │               ┌──────────────┐
│ cart_items   │     ┌──────┴────────┐     │  payments    │
│─────────────│     │  addresses    │     │──────────────│
│ id (PK)     │     │──────────────│     │ id (PK)      │
│ user_id     │     │ id (PK)      │     │ order_id     │
│ product_id  │     │ user_id      │     │ amount       │
│ quantity    │     │ full_name    │     │ method       │
│ created_at  │     │ phone        │     │ status       │
└─────────────┘     │ address_line │     │ transaction_ │
                    │ city         │     │   id         │
                    │ district     │     │ provider     │
                    │ ward         │     │ created_at   │
                    │ is_default   │     └──────────────┘
                    └──────────────┘
```

---

## Cấu trúc Project (Clean Architecture)

```
ecommerce-api/
├── pom.xml
├── src/
│   ├── main/
│   │   ├── java/com/example/ecommerce/
│   │   │   ├── EcommerceApplication.java
│   │   │   │
│   │   │   ├── config/                     # Cấu hình
│   │   │   │   ├── SecurityConfig.java
│   │   │   │   ├── RedisConfig.java
│   │   │   │   ├── OpenApiConfig.java
│   │   │   │   ├── MapStructConfig.java
│   │   │   │   └── AsyncConfig.java
│   │   │   │
│   │   │   ├── entity/                     # JPA Entities
│   │   │   │   ├── BaseEntity.java
│   │   │   │   ├── User.java
│   │   │   │   ├── Product.java
│   │   │   │   ├── Category.java
│   │   │   │   ├── Order.java
│   │   │   │   ├── OrderItem.java
│   │   │   │   ├── CartItem.java
│   │   │   │   ├── Address.java
│   │   │   │   ├── Payment.java
│   │   │   │   └── Review.java
│   │   │   │
│   │   │   ├── repository/                 # Spring Data JPA
│   │   │   │   ├── UserRepository.java
│   │   │   │   ├── ProductRepository.java
│   │   │   │   ├── CategoryRepository.java
│   │   │   │   ├── OrderRepository.java
│   │   │   │   └── ...
│   │   │   │
│   │   │   ├── service/                    # Business Logic
│   │   │   │   ├── UserService.java
│   │   │   │   ├── ProductService.java
│   │   │   │   ├── CartService.java
│   │   │   │   ├── OrderService.java
│   │   │   │   ├── PaymentService.java
│   │   │   │   ├── EmailService.java
│   │   │   │   └── AuthService.java
│   │   │   │
│   │   │   ├── controller/                 # REST Controllers
│   │   │   │   ├── AuthController.java
│   │   │   │   ├── ProductController.java
│   │   │   │   ├── CategoryController.java
│   │   │   │   ├── CartController.java
│   │   │   │   ├── OrderController.java
│   │   │   │   └── UserController.java
│   │   │   │
│   │   │   ├── dto/                        # Data Transfer Objects
│   │   │   │   ├── request/
│   │   │   │   │   ├── LoginRequest.java
│   │   │   │   │   ├── RegisterRequest.java
│   │   │   │   │   ├── ProductCreateRequest.java
│   │   │   │   │   └── ...
│   │   │   │   ├── response/
│   │   │   │   │   ├── ApiResponse.java
│   │   │   │   │   ├── JwtResponse.java
│   │   │   │   │   ├── ProductResponse.java
│   │   │   │   │   └── ...
│   │   │   │   └── mapper/
│   │   │   │       ├── ProductMapper.java
│   │   │   │       └── UserMapper.java
│   │   │   │
│   │   │   ├── security/                   # Spring Security
│   │   │   │   ├── JwtTokenProvider.java
│   │   │   │   ├── JwtAuthFilter.java
│   │   │   │   ├── CustomUserDetailsService.java
│   │   │   │   └── SecurityUtils.java
│   │   │   │
│   │   │   ├── exception/                  # Exception Handling
│   │   │   │   ├── GlobalExceptionHandler.java
│   │   │   │   ├── ResourceNotFoundException.java
│   │   │   │   ├── BadRequestException.java
│   │   │   │   └── UnauthorizedException.java
│   │   │   │
│   │   │   └── util/                       # Utilities
│   │   │       ├── SlugUtils.java
│   │   │       └── FileUploadUtils.java
│   │   │
│   │   └── resources/
│   │       ├── application.yml
│   │       ├── application-dev.yml
│   │       ├── application-prod.yml
│   │       ├── db/migration/               # Flyway
│   │       │   ├── V1__create_users.sql
│   │       │   ├── V2__create_categories.sql
│   │       │   ├── V3__create_products.sql
│   │       │   └── ...
│   │       └── templates/                  # Email templates
│   │           ├── order-confirmation.html
│   │           └── welcome-email.html
│   │
│   └── test/
│       └── java/com/example/ecommerce/
│           ├── service/
│           │   ├── ProductServiceTest.java
│           │   └── OrderServiceTest.java
│           ├── controller/
│           │   ├── ProductControllerTest.java
│           │   └── AuthControllerTest.java
│           └── repository/
│               └── ProductRepositoryTest.java
│
├── Dockerfile
├── docker-compose.yml
└── .github/workflows/ci.yml
```

---

## Yêu cầu cài đặt trước khi bắt đầu

### Phần mềm cần thiết

| Phần mềm | Phiên bản | Mục đích |
|-----------|-----------|----------|
| **JDK** | 21 (LTS) | Java Development Kit |
| **IntelliJ IDEA** | Community/Ultimate | IDE chính cho Java |
| **Maven** | 3.9+ | Build tool (hoặc dùng wrapper) |
| **MySQL** | 8.0+ | Database chính |
| **Redis** | 7.x | Cache & session store |
| **Docker** | 24+ | Container hóa ứng dụng |
| **Docker Compose** | 2.x | Orchestration local |
| **Postman** | latest | Test API |
| **Git** | 2.x | Version control |

### Kiến thức nền tảng

- **Java cơ bản**: OOP, Collections, Streams, Lambda, Exception handling
- **SQL cơ bản**: SELECT, JOIN, WHERE, GROUP BY
- **HTTP/REST**: GET, POST, PUT, DELETE, status codes
- **Git cơ bản**: clone, commit, push, pull, branch

---

## Cách sử dụng tài liệu

1. **Mỗi ngày 1 bài** - đọc lý thuyết trước, rồi code theo hướng dẫn
2. **Gõ code tay** - KHÔNG copy/paste, gõ từng dòng để nhớ
3. **Chạy test** - mỗi bài đều có phần verify, chạy đầy đủ
4. **Đọc thêm** - mỗi bài có link tài liệu gốc (Spring docs)
5. **Commit thường xuyên** - mỗi bài xong thì commit 1 lần

---

## Điều hướng

| Tuần | Folder | Bắt đầu |
|------|--------|---------|
| Tuần 1 | [week-01/](week-01/) | [Ngày 01 - Java Ecosystem & Spring Boot](week-01/day-01-spring-boot-overview.md) |
| Tuần 2 | [week-02/](week-02/) | [Ngày 06 - JPA Entity & Relationships](week-02/day-06-jpa-entity.md) |
| Tuần 3 | [week-03/](week-03/) | [Ngày 11 - REST Controller](week-03/day-11-rest-controller.md) |
| Tuần 4 | [week-04/](week-04/) | [Ngày 16 - Spring Security](week-04/day-16-spring-security.md) |
| Tuần 5 | [week-05/](week-05/) | [Ngày 21 - Product & Category](week-05/day-21-product-category.md) |
| Tuần 6 | [week-06/](week-06/) | [Ngày 26 - Unit Testing](week-06/day-26-unit-testing.md) |
