# Spring Boot CRUD Enterprise - 5 Days Training

## Mục tiêu khóa học

Khóa học 5 ngày dành cho developer đã biết C#/.NET, học Spring Boot để xây dựng các chức năng **quản lý dữ liệu doanh nghiệp** như:
- Tra cứu danh sách với filter nâng cao
- Xem chi tiết bản ghi
- Export Excel/CSV
- Phân trang dữ liệu lớn

## Đối tượng

- Developer đã có kinh nghiệm C#/.NET
- Muốn chuyển sang Java Spring Boot
- Cần build các chức năng CRUD enterprise-grade

## So sánh C#/.NET vs Java Spring

| Concept | C#/.NET | Java Spring |
|---------|---------|-------------|
| Framework | ASP.NET Core | Spring Boot |
| DI Container | Built-in DI | Spring IoC |
| ORM | Entity Framework | JPA/Hibernate |
| Dynamic Query | LINQ + Expression | JPA Specification |
| DTO Mapping | AutoMapper | MapStruct |
| Validation | DataAnnotations | Jakarta Validation |

## Cấu trúc khóa học

```
Day 1: Spring Boot Setup + IoC/DI
       ├── Project structure
       ├── Configuration
       └── Dependency Injection (so sánh với .NET DI)

Day 2: JPA + Specification (Dynamic Query)
       ├── Entity & Repository
       ├── Query Methods
       └── Specification pattern (thay thế LINQ)

Day 3: REST API + Pagination
       ├── Controller & DTO
       ├── Validation
       └── Pageable response

Day 4: Export Excel/CSV
       ├── Apache POI (Excel)
       └── OpenCSV

Day 5: Angular Integration
       ├── HTTP Client
       ├── Table component
       └── Search form
```

## Tech Stack

| Layer | Technology |
|-------|------------|
| Language | Java 25 LTS |
| Framework | Spring Boot 4.0+ |
| ORM | Spring Data JPA + Hibernate 7 |
| Database | MySQL 8.0+ |
| Build | Maven |
| API Docs | SpringDoc OpenAPI 3.0 |

## What's New in Spring Boot 4.0 & Java 25

### Spring Boot 4.0 Highlights
- **Modularization**: Codebase được chia thành các jars nhỏ hơn, focused hơn
- **Null Safety**: Hỗ trợ JSpecify cho null-safety tốt hơn
- **Java 25 First-class Support**: Tối ưu cho Java 25 (vẫn tương thích Java 17+)
- **Hibernate 7**: ORM mới với performance improvements
- **Virtual Threads**: Hỗ trợ tốt hơn cho Project Loom

### Java 25 LTS Highlights
- **Pattern Matching**: Enhanced switch expressions và record patterns
- **Virtual Threads (Stable)**: Lightweight threads cho high-concurrency
- **Sequenced Collections**: Ordered collection operations
- **String Templates**: Simplified string formatting
- **Scoped Values**: Thread-local alternative cho virtual threads

## Project mẫu

Xây dựng module **"Quản lý thông tin thị trường"** với các chức năng:
- Tra cứu thông tin thị trường (search + filter)
- Xem chi tiết
- Export Excel/CSV
- Phân trang

## Navigation

- [Day 1: Spring Boot Setup](./day-01-spring-boot-setup.md)
- [Day 2: JPA Specification](./day-02-jpa-specification.md)
- [Day 3: REST API Pagination](./day-03-rest-api-pagination.md)
- [Day 4: Export Excel CSV](./day-04-export-excel-csv.md)
- [Day 5: Angular Integration](./day-05-angular-integration.md)
