# Java Spring Boot E-Commerce — Speed Run 7 Days

## Tổng Quan

Khóa học nén **30 ngày** thành **7 ngày** — tập trung **Backend Core**: JPA + REST API + Security.
Dành cho **Beginner** biết Java cơ bản, chưa dùng Spring Boot.

---

## Tech Stack

| Layer | Technology | Version |
|-------|-----------|---------|
| Language | Java | 21 (LTS) |
| Framework | Spring Boot | 3.3.x |
| ORM | Spring Data JPA + Hibernate 6 | |
| Database | MySQL | 8.0 |
| Cache | Redis | 7.x |
| Migration | Flyway | 10.x |
| Security | Spring Security + JWT (JJWT 0.12.x) | |
| Mapping | MapStruct | 1.5.x |
| Validation | Bean Validation (Hibernate Validator) | |
| Testing | JUnit 5 + Mockito + TestContainers | |
| Build | Maven | 3.9.x |
| Container | Docker + Docker Compose | |
| API Docs | SpringDoc OpenAPI | 2.x |

---

## Lộ Trình 7 Ngày

| Day | Chủ Đề | Output Chính |
|-----|--------|-------------|
| 1 | Foundation + Project Setup | Hello World API chạy được |
| 2 | IoC/DI + Config + JPA Setup | Category/Product entities trong DB |
| 3 | JPA Mastery | Full schema + Flyway + queries tối ưu |
| 4 | REST API + CRUD + Validation | Complete Product CRUD API |
| 5 | Security + JWT + Registration | Auth system hoàn chỉnh |
| 6 | Cart + Order + Payment | E-Commerce features chạy end-to-end |
| 7 | Testing + Docker | Test suite + production-ready container |

---

## Prerequisites

- Java 21 installed (`java -version`)
- Maven 3.9+ (`mvn -version`)
- Docker Desktop running
- IDE: IntelliJ IDEA (recommended) hoặc VS Code + Java Extension Pack
- Postman hoặc HTTPie để test API
- Git

---

## Database Schema

```
┌──────────┐     ┌──────────────┐     ┌───────────────┐
│  users   │     │  categories  │     │   products    │
├──────────┤     ├──────────────┤     ├───────────────┤
│ id (PK)  │     │ id (PK)      │     │ id (PK)       │
│ email    │     │ name         │     │ name          │
│ password │     │ slug         │     │ slug          │
│ name     │     │ description  │     │ description   │
│ phone    │     │ parent_id(FK)│     │ price         │
│ role     │     └──────────────┘     │ stock_quantity│
│ active   │           │              │ category_id   │
│ locked   │           └──────────────│ (FK)          │
└──────────┘                          └───────────────┘
      │                                      │
      │    ┌──────────────┐                  │
      │    │product_images│    ┌─────────────┐
      │    ├──────────────┤    │   reviews   │
      │    │ id (PK)      │    ├─────────────┤
      │    │ product_id   │    │ id (PK)     │
      │    │ url          │    │ user_id(FK) │
      │    │ is_primary   │    │ product_id  │
      │    └──────────────┘    │ rating      │
      │                        │ comment     │
      │                        └─────────────┘
      │
      │    ┌──────────────┐     ┌──────────────┐
      ├────│  addresses   │     │  cart_items   │
      │    ├──────────────┤     ├──────────────┤
      │    │ id (PK)      │     │ id (PK)      │
      │    │ user_id (FK) │     │ user_id (FK) │
      │    │ street       │     │ product_id   │
      │    │ city         │     │ quantity     │
      │    │ is_default   │     └──────────────┘
      │    └──────────────┘
      │
      │    ┌──────────────┐     ┌──────────────┐
      └────│   orders     │────▶│ order_items  │
           ├──────────────┤     ├──────────────┤
           │ id (PK)      │     │ id (PK)      │
           │ user_id (FK) │     │ order_id(FK) │
           │ order_number │     │ product_id   │
           │ status       │     │ product_name │
           │ total_amount │     │ price        │
           │ address_id   │     │ quantity     │
           └──────────────┘     └──────────────┘
                  │
           ┌──────────────┐
           │   payments   │
           ├──────────────┤
           │ id (PK)      │
           │ order_id(FK) │
           │ amount       │
           │ method       │
           │ status       │
           │ transaction_id│
           └──────────────┘
```

---

## Project Structure

```
ecommerce/
├── src/main/java/com/example/ecommerce/
│   ├── EcommerceApplication.java
│   ├── config/              # Security, Redis, App configs
│   ├── controller/          # REST Controllers
│   ├── dto/
│   │   ├── request/         # Input DTOs
│   │   └── response/        # Output DTOs
│   ├── entity/              # JPA Entities
│   ├── enums/               # Status, Role enums
│   ├── exception/           # Custom exceptions + handler
│   ├── mapper/              # MapStruct mappers
│   ├── repository/          # Spring Data JPA repos
│   ├── security/            # JWT, filters, UserDetails
│   ├── service/             # Business logic
│   ├── specification/       # JPA Specifications
│   └── util/                # SlugUtils, etc.
├── src/main/resources/
│   ├── application.yml
│   ├── application-dev.yml
│   ├── db/migration/        # Flyway SQL scripts
│   └── .env
├── src/test/java/
├── docker-compose.yml
├── Dockerfile
└── pom.xml
```

---

## Quick Start

```bash
# 1. Clone & start infrastructure
docker compose up -d mysql redis

# 2. Run the app
./mvnw spring-boot:run -Dspring-boot.run.profiles=dev

# 3. Verify
curl http://localhost:8080/api/v1/hello
# → {"success":true,"message":"Hello Spring Boot!","data":"v1.0.0"}

# 4. Swagger UI
open http://localhost:8080/swagger-ui.html
```

---

## Convention

- **Package naming:** `com.example.ecommerce`
- **API prefix:** `/api/v1/`
- **Entity:** PascalCase (`ProductImage`)
- **Table:** snake_case (`product_images`)
- **DTO suffix:** `Request`, `Response` (`ProductCreateRequest`, `ProductResponse`)
- **Date/Time:** UTC everywhere, `Instant` in entities
- **Money:** `BigDecimal` with `HALF_UP` rounding
- **ID type:** `Long` (auto-increment)

---

> **Bắt đầu từ Day 1** → [day-01-foundation-setup.md](./day-01-foundation-setup.md)
