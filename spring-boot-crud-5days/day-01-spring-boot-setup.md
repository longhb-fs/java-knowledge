# Day 1: Spring Boot Setup + IoC/DI

## Mục tiêu
- Setup Spring Boot project
- Hiểu cấu trúc project
- Nắm vững IoC/DI (so sánh với .NET)
- Configuration management

## 1. Project Setup

### 1.1. Tạo project với Spring Initializr

Truy cập [start.spring.io](https://start.spring.io) hoặc dùng IDE:

```
Project: Maven
Language: Java
Spring Boot: 4.0.x
Java: 25

Dependencies:
- Spring Web
- Spring Data JPA
- MySQL Driver
- Lombok
- Validation
```

### 1.2. So sánh cấu trúc project

```
C#/.NET                          Java Spring
─────────────────────────────────────────────────
/Controllers                     /controller
/Services                        /service
/Repositories                    /repository
/Models                          /entity
/DTOs                            /dto
appsettings.json                 application.yml
Program.cs                       Application.java
```

### 1.3. Cấu trúc Spring Boot project

```
src/main/java/com/example/market/
├── MarketApplication.java          # Entry point (@SpringBootApplication)
├── config/                         # Configuration classes
│   └── AppConfig.java
├── controller/                     # REST Controllers
│   └── MarketInfoController.java
├── service/                        # Business logic
│   ├── MarketInfoService.java      # Interface
│   └── impl/
│       └── MarketInfoServiceImpl.java
├── repository/                     # Data access
│   └── MarketInfoRepository.java
├── entity/                         # JPA Entities
│   └── MarketInfo.java
├── dto/                            # Data Transfer Objects
│   ├── request/
│   │   └── MarketInfoRequest.java
│   └── response/
│       └── MarketInfoResponse.java
└── exception/                      # Custom exceptions
    └── ResourceNotFoundException.java

src/main/resources/
├── application.yml                 # Main config
├── application-dev.yml             # Dev profile
└── application-prod.yml            # Prod profile
```

## 2. IoC/DI - So sánh với .NET

### 2.1. .NET DI (để tham khảo)

```csharp
// .NET - Program.cs
builder.Services.AddScoped<IMarketService, MarketService>();

// .NET - Controller
public class MarketController : ControllerBase
{
    private readonly IMarketService _service;

    public MarketController(IMarketService service)
    {
        _service = service;
    }
}
```

### 2.2. Spring DI - Cách tương đương

```java
// Spring - Service Interface
public interface MarketInfoService {
    List<MarketInfoResponse> findAll();
    MarketInfoResponse findById(Long id);
}

// Spring - Service Implementation
@Service  // Tương đương AddScoped trong .NET
public class MarketInfoServiceImpl implements MarketInfoService {

    private final MarketInfoRepository repository;

    // Constructor injection - giống .NET
    public MarketInfoServiceImpl(MarketInfoRepository repository) {
        this.repository = repository;
    }

    @Override
    public List<MarketInfoResponse> findAll() {
        return repository.findAll().stream()
            .map(this::toResponse)
            .toList();
    }
}

// Spring - Controller
@RestController
@RequestMapping("/api/market-info")
@RequiredArgsConstructor  // Lombok tự generate constructor
public class MarketInfoController {

    private final MarketInfoService service;  // Auto-injected

    @GetMapping
    public List<MarketInfoResponse> getAll() {
        return service.findAll();
    }
}
```

### 2.3. Các annotation DI trong Spring

| Annotation | Scope | Tương đương .NET |
|------------|-------|------------------|
| `@Component` | Singleton | `AddSingleton` |
| `@Service` | Singleton | `AddScoped` (business layer) |
| `@Repository` | Singleton | `AddScoped` (data layer) |
| `@Controller` | Singleton | Controller |
| `@Scope("prototype")` | New instance mỗi lần | `AddTransient` |

### 2.4. Các cách inject dependency

```java
// ✅ Cách 1: Constructor injection (RECOMMENDED)
@Service
@RequiredArgsConstructor
public class MarketInfoServiceImpl {
    private final MarketInfoRepository repository;
    private final MarketInfoMapper mapper;
}

// ⚠️ Cách 2: Field injection (không khuyến khích)
@Service
public class MarketInfoServiceImpl {
    @Autowired
    private MarketInfoRepository repository;
}

// ✅ Cách 3: Setter injection (ít dùng)
@Service
public class MarketInfoServiceImpl {
    private MarketInfoRepository repository;

    @Autowired
    public void setRepository(MarketInfoRepository repository) {
        this.repository = repository;
    }
}
```

## 3. Configuration

### 3.1. application.yml

```yaml
# application.yml
spring:
  application:
    name: market-management

  # Database config
  datasource:
    url: jdbc:mysql://localhost:3306/market_db?useSSL=false&serverTimezone=UTC
    username: root
    password: password
    driver-class-name: com.mysql.cj.jdbc.Driver

  # JPA config
  jpa:
    hibernate:
      ddl-auto: validate  # none, validate, update, create, create-drop
    show-sql: true
    properties:
      hibernate:
        format_sql: true

  # Virtual Threads (Java 25 feature)
  threads:
    virtual:
      enabled: true  # Enable virtual threads for better concurrency

# Server config
server:
  port: 8080
  servlet:
    context-path: /api

# Custom config
app:
  pagination:
    default-page-size: 10
    max-page-size: 100
```

> **Note (Spring Boot 4.0):** Hibernate dialect được auto-detect, không cần config thủ công nữa.

### 3.2. So sánh với appsettings.json

```
.NET appsettings.json              Spring application.yml
─────────────────────────────────────────────────────────
"ConnectionStrings": {             spring:
  "Default": "..."                   datasource:
}                                      url: "..."

"Logging": {                       logging:
  "LogLevel": "Information"          level:
}                                      root: INFO
```

### 3.3. Custom Configuration Class

```java
// Đọc custom config từ application.yml
@Configuration
@ConfigurationProperties(prefix = "app.pagination")
@Data  // Lombok
public class PaginationConfig {
    private int defaultPageSize = 10;
    private int maxPageSize = 100;
}

// Sử dụng
@Service
@RequiredArgsConstructor
public class MarketInfoServiceImpl {
    private final PaginationConfig paginationConfig;

    public Page<MarketInfo> findAll(int page, int size) {
        // Validate page size
        int validSize = Math.min(size, paginationConfig.getMaxPageSize());
        return repository.findAll(PageRequest.of(page, validSize));
    }
}
```

### 3.4. Profile-based Configuration

```yaml
# application-dev.yml
spring:
  datasource:
    url: jdbc:mysql://localhost:3306/market_dev
  jpa:
    show-sql: true

# application-prod.yml
spring:
  datasource:
    url: jdbc:mysql://prod-server:3306/market_prod
  jpa:
    show-sql: false
```

Chạy với profile:
```bash
# Dev
java -jar app.jar --spring.profiles.active=dev

# Prod
java -jar app.jar --spring.profiles.active=prod
```

## 4. Entry Point

### 4.1. Main Application

```java
@SpringBootApplication  // = @Configuration + @EnableAutoConfiguration + @ComponentScan
public class MarketApplication {
    public static void main(String[] args) {
        SpringApplication.run(MarketApplication.class, args);
    }
}
```

### 4.2. So sánh với .NET Program.cs

```csharp
// .NET 6+ Program.cs
var builder = WebApplication.CreateBuilder(args);
builder.Services.AddControllers();
// ... add services
var app = builder.Build();
app.MapControllers();
app.Run();
```

```java
// Spring Boot - tất cả được auto-configure
@SpringBootApplication
public class MarketApplication {
    public static void main(String[] args) {
        SpringApplication.run(MarketApplication.class, args);
    }
}
// Controllers, Services auto-scan từ package
```

## 5. Hands-on: Tạo project cơ bản

### 5.1. pom.xml dependencies

```xml
<properties>
    <java.version>25</java.version>
    <mapstruct.version>1.6.3</mapstruct.version>
</properties>

<dependencies>
    <!-- Spring Boot Starters -->
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

    <!-- Database -->
    <dependency>
        <groupId>com.mysql</groupId>
        <artifactId>mysql-connector-j</artifactId>
        <scope>runtime</scope>
    </dependency>

    <!-- Lombok -->
    <dependency>
        <groupId>org.projectlombok</groupId>
        <artifactId>lombok</artifactId>
        <optional>true</optional>
    </dependency>

    <!-- MapStruct -->
    <dependency>
        <groupId>org.mapstruct</groupId>
        <artifactId>mapstruct</artifactId>
        <version>${mapstruct.version}</version>
    </dependency>
    <dependency>
        <groupId>org.mapstruct</groupId>
        <artifactId>mapstruct-processor</artifactId>
        <version>${mapstruct.version}</version>
        <scope>provided</scope>
    </dependency>
</dependencies>
```

### 5.2. Tạo Hello World Controller

```java
@RestController
@RequestMapping("/api/hello")
public class HelloController {

    @GetMapping
    public Map<String, String> hello() {
        return Map.of("message", "Hello Spring Boot!");
    }

    @GetMapping("/{name}")
    public Map<String, String> helloName(@PathVariable String name) {
        return Map.of("message", "Hello " + name + "!");
    }
}
```

Test: `GET http://localhost:8080/api/hello`

## 6. Tổng kết Day 1

| Học được | C#/.NET equivalent |
|----------|-------------------|
| `@SpringBootApplication` | `Program.cs` + auto-config |
| `@Service`, `@Repository` | `AddScoped<>` |
| `@RestController` | `[ApiController]` |
| `application.yml` | `appsettings.json` |
| Constructor injection | Constructor injection |
| `@ConfigurationProperties` | `IOptions<T>` |
| Virtual Threads | `async/await` (tương tự) |

### Spring Boot 4.0 Changes từ 3.x
- Hibernate dialect auto-detect (không cần config)
- Virtual threads enabled by default với `spring.threads.virtual.enabled=true`
- Improved null-safety với JSpecify annotations
- Modular jars cho faster startup

## Navigation

- [← Overview](./00-overview.md)
- [Day 2: JPA Specification →](./day-02-jpa-specification.md)
