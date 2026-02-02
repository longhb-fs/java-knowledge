# Ngày 04: Cấu hình Application (Properties, YAML, Profiles)

## Mục tiêu hôm nay
- Hiểu hệ thống cấu hình của Spring Boot (externalized configuration)
- Phân biệt application.properties vs application.yml
- Sử dụng @Value và @ConfigurationProperties
- Quản lý cấu hình theo môi trường (Profiles)
- Bảo mật thông tin nhạy cảm (secrets management)

---

## 1. EXTERNALIZED CONFIGURATION

### 1.1. Thứ tự ưu tiên cấu hình (từ cao → thấp)

Spring Boot đọc cấu hình từ nhiều nguồn, nguồn ưu tiên CAO hơn sẽ GHI ĐÈ nguồn thấp:

```
Ưu tiên CAO nhất
    │
    ├── 1. Command line arguments          (--server.port=9090)
    ├── 2. SPRING_APPLICATION_JSON          (JSON trong env var)
    ├── 3. OS Environment variables        (SERVER_PORT=9090)
    ├── 4. application-{profile}.yml        (profile-specific)
    ├── 5. application.yml                  (default config)
    ├── 6. @PropertySource trên @Configuration
    └── 7. Default properties (SpringApplication.setDefaultProperties)
    │
Ưu tiên THẤP nhất
```

**Ví dụ thực tế:**
```bash
# application.yml đặt port = 8080
# Nhưng chạy với command line → port = 9090
java -jar ecommerce.jar --server.port=9090

# Hoặc qua environment variable
export SERVER_PORT=9090
java -jar ecommerce.jar
```

### 1.2. Properties vs YAML

```properties
# application.properties (flat key-value)
server.port=8080
spring.datasource.url=jdbc:mysql://localhost:3306/ecommerce
spring.datasource.username=root
spring.datasource.password=1q2w3E*
spring.jpa.hibernate.ddl-auto=update
spring.jpa.show-sql=true
```

```yaml
# application.yml (hierarchical - dễ đọc hơn)
server:
  port: 8080

spring:
  datasource:
    url: jdbc:mysql://localhost:3306/ecommerce
    username: root
    password: 1q2w3E*
  jpa:
    hibernate:
      ddl-auto: update
    show-sql: true
```

| Tiêu chí | .properties | .yml |
|-----------|------------|------|
| **Cấu trúc** | Flat (key=value) | Hierarchical (indented) |
| **Đọc** | Khó đọc khi nhiều nested | Dễ đọc, trực quan |
| **List** | `list[0]=a`, `list[1]=b` | `- a`, `- b` |
| **Multi-document** | Không hỗ trợ | Hỗ trợ (`---` separator) |
| **Comment** | `# comment` | `# comment` |
| **Dùng khi** | Project đơn giản | **Khuyến khích cho Spring Boot** |

---

## 2. @VALUE - ĐỌC GIÁ TRỊ ĐƠN

### 2.1. Cơ bản

```yaml
# application.yml
app:
  name: E-Commerce API
  version: 1.0.0
  upload:
    max-file-size: 10MB
    allowed-types: jpg,png,gif,webp
    path: /uploads
  pagination:
    default-page-size: 20
    max-page-size: 100
```

```java
@Service
@Slf4j
public class FileUploadService {

    @Value("${app.upload.max-file-size}")
    private String maxFileSize;  // "10MB"

    @Value("${app.upload.path}")
    private String uploadPath;   // "/uploads"

    @Value("${app.upload.allowed-types}")
    private String allowedTypes; // "jpg,png,gif,webp"

    // Giá trị mặc định nếu property không tồn tại
    @Value("${app.upload.max-files:5}")
    private int maxFiles;  // 5 (default)

    // Inject từ environment variable
    @Value("${DATABASE_PASSWORD:defaultPass}")
    private String dbPassword;

    // SpEL (Spring Expression Language)
    @Value("#{${app.pagination.default-page-size} * 2}")
    private int doublePageSize;  // 40

    @PostConstruct
    public void init() {
        log.info("Upload config: maxSize={}, path={}, types={}",
                maxFileSize, uploadPath, allowedTypes);
    }
}
```

### 2.2. Inject List và Map

```yaml
# application.yml
app:
  cors:
    allowed-origins:
      - http://localhost:4200
      - http://localhost:3000
      - https://ecommerce.example.com
  payment:
    providers:
      stripe: sk_test_xxx
      vnpay: VNPAY_SANDBOX_KEY
```

```java
@Configuration
public class AppConfig {

    // Inject List
    @Value("${app.cors.allowed-origins}")
    private List<String> allowedOrigins;
    // ["http://localhost:4200", "http://localhost:3000", "https://ecommerce.example.com"]

    // Inject từ comma-separated string
    @Value("#{'${app.upload.allowed-types}'.split(',')}")
    private List<String> allowedFileTypes;
    // ["jpg", "png", "gif", "webp"]
}
```

---

## 3. @CONFIGURATIONPROPERTIES - CÁCH TỐT HƠN

### 3.1. Tại sao tốt hơn @Value?

| @Value | @ConfigurationProperties |
|--------|------------------------|
| Inject từng field riêng lẻ | Bind toàn bộ group vào 1 class |
| Không type-safe | Type-safe (compile-time check) |
| Không validate được | Hỗ trợ @Validated |
| Khó refactor | Dễ refactor (1 class) |
| Dùng cho 1-2 property | Dùng cho group nhiều property |

### 3.2. Tạo Configuration Properties class

```yaml
# application.yml
app:
  name: E-Commerce API
  version: 1.0.0
  jwt:
    secret: mySecretKeyHere12345678901234567890
    expiration: 86400000       # 1 ngày (milliseconds)
    refresh-expiration: 604800000  # 7 ngày
  upload:
    max-file-size: 10485760    # 10MB (bytes)
    allowed-types:
      - jpg
      - png
      - gif
      - webp
    path: /uploads
  pagination:
    default-size: 20
    max-size: 100
  cors:
    allowed-origins:
      - http://localhost:4200
      - http://localhost:3000
```

```java
package com.example.ecommerce.config;

import jakarta.validation.constraints.Min;
import jakarta.validation.constraints.NotBlank;
import jakarta.validation.constraints.NotEmpty;
import lombok.Data;
import org.springframework.boot.context.properties.ConfigurationProperties;
import org.springframework.stereotype.Component;
import org.springframework.validation.annotation.Validated;

import java.util.List;

@Data
@Component
@Validated  // Bật validation cho properties
@ConfigurationProperties(prefix = "app")  // Bind tất cả property bắt đầu bằng "app."
public class AppProperties {

    @NotBlank
    private String name;

    private String version;

    private Jwt jwt = new Jwt();
    private Upload upload = new Upload();
    private Pagination pagination = new Pagination();
    private Cors cors = new Cors();

    @Data
    public static class Jwt {
        @NotBlank
        private String secret;

        @Min(60000)  // Tối thiểu 1 phút
        private long expiration = 86400000;  // Default 1 ngày

        private long refreshExpiration = 604800000;  // Default 7 ngày
    }

    @Data
    public static class Upload {
        private long maxFileSize = 10485760;  // 10MB default

        @NotEmpty
        private List<String> allowedTypes = List.of("jpg", "png", "gif");

        private String path = "/uploads";
    }

    @Data
    public static class Pagination {
        @Min(1)
        private int defaultSize = 20;

        @Min(1)
        private int maxSize = 100;
    }

    @Data
    public static class Cors {
        private List<String> allowedOrigins = List.of("http://localhost:4200");
    }
}
```

### 3.3. Sử dụng trong Service

```java
@Service
@RequiredArgsConstructor
@Slf4j
public class JwtTokenProvider {

    private final AppProperties appProperties;  // Inject cả config group

    public String generateToken(String username) {
        AppProperties.Jwt jwtConfig = appProperties.getJwt();

        return Jwts.builder()
                .subject(username)
                .issuedAt(new Date())
                .expiration(new Date(System.currentTimeMillis() + jwtConfig.getExpiration()))
                .signWith(getSigningKey())
                .compact();
    }

    private SecretKey getSigningKey() {
        byte[] keyBytes = Decoders.BASE64.decode(appProperties.getJwt().getSecret());
        return Keys.hmacShaKeyFor(keyBytes);
    }
}
```

```java
@Service
@RequiredArgsConstructor
public class FileUploadService {

    private final AppProperties appProperties;

    public void validateFile(MultipartFile file) {
        AppProperties.Upload uploadConfig = appProperties.getUpload();

        // Check file size
        if (file.getSize() > uploadConfig.getMaxFileSize()) {
            throw new BadRequestException("File quá lớn. Tối đa: " +
                    uploadConfig.getMaxFileSize() / 1024 / 1024 + "MB");
        }

        // Check file type
        String extension = getExtension(file.getOriginalFilename());
        if (!uploadConfig.getAllowedTypes().contains(extension)) {
            throw new BadRequestException("Loại file không được phép. Cho phép: " +
                    String.join(", ", uploadConfig.getAllowedTypes()));
        }
    }
}
```

---

## 4. PROFILES NÂNG CAO

### 4.1. Multi-document YAML

Gom tất cả profiles vào 1 file (dùng `---` separator):

```yaml
# application.yml - config chung
spring:
  application:
    name: ecommerce-api
  profiles:
    active: dev  # Profile mặc định

app:
  name: E-Commerce API

---
# Profile: dev
spring:
  config:
    activate:
      on-profile: dev
  datasource:
    url: jdbc:mysql://localhost:3306/ecommerce
    username: root
    password: 1q2w3E*
  jpa:
    hibernate:
      ddl-auto: update
    show-sql: true

logging:
  level:
    com.example: DEBUG

---
# Profile: test
spring:
  config:
    activate:
      on-profile: test
  datasource:
    url: jdbc:mysql://localhost:3306/ecommerce_test
    username: root
    password: 1q2w3E*
  jpa:
    hibernate:
      ddl-auto: create-drop

---
# Profile: prod
spring:
  config:
    activate:
      on-profile: prod
  datasource:
    url: ${DATABASE_URL}
    username: ${DATABASE_USERNAME}
    password: ${DATABASE_PASSWORD}
  jpa:
    hibernate:
      ddl-auto: none
    show-sql: false

logging:
  level:
    com.example: WARN
```

### 4.2. Kích hoạt Profile

```bash
# Cách 1: application.yml
spring.profiles.active: dev

# Cách 2: Command line
java -jar app.jar --spring.profiles.active=prod

# Cách 3: Environment variable
export SPRING_PROFILES_ACTIVE=prod
java -jar app.jar

# Cách 4: Maven
mvn spring-boot:run -Dspring-boot.run.profiles=prod

# Cách 5: IntelliJ IDEA
# Run Configuration → Active Profiles: prod

# Cách 6: Kết hợp nhiều profiles
java -jar app.jar --spring.profiles.active=prod,redis,email
```

### 4.3. Profile-specific Beans

```java
@Configuration
public class DataSourceConfig {

    // Dev: dùng HikariCP với settings thoải mái
    @Bean
    @Profile("dev")
    public HikariDataSource devDataSource() {
        HikariConfig config = new HikariConfig();
        config.setMaximumPoolSize(5);
        config.setMinimumIdle(2);
        config.setConnectionTimeout(10000);
        return new HikariDataSource(config);
    }

    // Prod: dùng HikariCP với settings chặt chẽ
    @Bean
    @Profile("prod")
    public HikariDataSource prodDataSource() {
        HikariConfig config = new HikariConfig();
        config.setMaximumPoolSize(20);
        config.setMinimumIdle(5);
        config.setConnectionTimeout(5000);
        config.setMaxLifetime(1800000);  // 30 phút
        config.setLeakDetectionThreshold(60000);  // Phát hiện connection leak
        return new HikariDataSource(config);
    }
}
```

---

## 5. BẢO MẬT THÔNG TIN NHẠY CẢM

### 5.1. TUYỆT ĐỐI KHÔNG commit secrets

```yaml
# ❌ SAI: hardcode secret trong application.yml (rồi commit lên Git!)
spring:
  datasource:
    password: MySecretPassword123!
app:
  jwt:
    secret: super-secret-jwt-key
  stripe:
    api-key: sk_live_xxx

# ✅ ĐÚNG: dùng environment variable
spring:
  datasource:
    password: ${DATABASE_PASSWORD}
app:
  jwt:
    secret: ${JWT_SECRET}
  stripe:
    api-key: ${STRIPE_API_KEY}
```

### 5.2. Sử dụng .env file (local development)

```bash
# .env (THÊM VÀO .gitignore!)
DATABASE_URL=jdbc:mysql://localhost:3306/ecommerce
DATABASE_USERNAME=root
DATABASE_PASSWORD=1q2w3E*
JWT_SECRET=mySecretKeyThatIsAtLeast256BitsLong12345
STRIPE_API_KEY=sk_test_xxx
REDIS_HOST=localhost
REDIS_PORT=6379
```

```
# .gitignore
.env
*.env
application-local.yml
```

### 5.3. Spring Boot + .env với dotenv

Thêm dependency:
```xml
<dependency>
    <groupId>me.paulschwarz</groupId>
    <artifactId>spring-dotenv</artifactId>
    <version>4.0.0</version>
</dependency>
```

Sau đó dùng trong application.yml:
```yaml
spring:
  datasource:
    url: ${DATABASE_URL}
    username: ${DATABASE_USERNAME}
    password: ${DATABASE_PASSWORD}
```

---

## 6. CẤU HÌNH NÂNG CAO

### 6.1. Custom Configuration với @Configuration

```java
@Configuration
public class WebConfig implements WebMvcConfigurer {

    private final AppProperties appProperties;

    public WebConfig(AppProperties appProperties) {
        this.appProperties = appProperties;
    }

    // CORS configuration
    @Override
    public void addCorsMappings(CorsRegistry registry) {
        registry.addMapping("/api/**")
                .allowedOrigins(appProperties.getCors().getAllowedOrigins().toArray(new String[0]))
                .allowedMethods("GET", "POST", "PUT", "DELETE", "PATCH")
                .allowedHeaders("*")
                .allowCredentials(true)
                .maxAge(3600);
    }

    // Custom JSON serialization
    @Bean
    public ObjectMapper objectMapper() {
        ObjectMapper mapper = new ObjectMapper();
        mapper.registerModule(new JavaTimeModule());
        mapper.disable(SerializationFeature.WRITE_DATES_AS_TIMESTAMPS);
        mapper.setSerializationInclusion(JsonInclude.Include.NON_NULL);
        mapper.configure(DeserializationFeature.FAIL_ON_UNKNOWN_PROPERTIES, false);
        return mapper;
    }
}
```

### 6.2. OpenAPI Configuration

```java
@Configuration
public class OpenApiConfig {

    @Bean
    public OpenAPI customOpenAPI() {
        return new OpenAPI()
                .info(new Info()
                        .title("E-Commerce API")
                        .version("1.0.0")
                        .description("REST API cho hệ thống E-Commerce")
                        .contact(new Contact()
                                .name("Dev Team")
                                .email("dev@example.com")))
                .addSecurityItem(new SecurityRequirement().addList("Bearer Auth"))
                .components(new Components()
                        .addSecuritySchemes("Bearer Auth",
                                new SecurityScheme()
                                        .type(SecurityScheme.Type.HTTP)
                                        .scheme("bearer")
                                        .bearerFormat("JWT")
                                        .description("Nhập JWT token")));
    }
}
```

---

## 7. LOGGING CONFIGURATION

### 7.1. Logback cấu hình trong application.yml

```yaml
logging:
  level:
    root: INFO
    com.example.ecommerce: DEBUG
    org.springframework.web: INFO
    org.springframework.security: DEBUG
    org.hibernate.SQL: DEBUG
    org.hibernate.type.descriptor.sql.BasicBinder: TRACE  # Log tham số SQL

  pattern:
    console: "%d{HH:mm:ss.SSS} %clr(%-5level) [%thread] %logger{36} - %msg%n"
    file: "%d{yyyy-MM-dd HH:mm:ss.SSS} %-5level [%thread] %logger{36} - %msg%n"

  file:
    name: logs/ecommerce.log    # Ghi log ra file

  logback:
    rollingpolicy:
      max-file-size: 10MB       # Max size mỗi file
      max-history: 30           # Giữ 30 ngày
      total-size-cap: 1GB       # Tổng max 1GB
```

### 7.2. Sử dụng Logger trong code

```java
@Service
@Slf4j  // Lombok tự tạo: private static final Logger log = ...
@RequiredArgsConstructor
public class OrderService {

    private final OrderRepository orderRepository;

    public OrderResponse createOrder(OrderCreateRequest request) {
        log.info("Tạo đơn hàng mới cho user: {}", request.getUserId());
        log.debug("Chi tiết đơn hàng: {} sản phẩm, tổng: {}",
                request.getItems().size(), request.getTotalAmount());

        try {
            Order order = processOrder(request);
            log.info("✅ Đã tạo đơn hàng #{} thành công", order.getOrderNumber());
            return toResponse(order);
        } catch (Exception e) {
            log.error("❌ Lỗi tạo đơn hàng cho user {}: {}",
                    request.getUserId(), e.getMessage(), e);
            throw e;
        }
    }
}
```

**Log levels:**
```
TRACE  →  Chi tiết nhất (parameter values, loop iterations)
DEBUG  →  Thông tin debug (method entry/exit, intermediate values)
INFO   →  Thông tin quan trọng (startup, business events)
WARN   →  Cảnh báo (deprecated API, retry, near-limit)
ERROR  →  Lỗi (exception, failure)
```

---

## 8. BÀI TẬP & CHECKLIST

### Bài tập thực hành

- [ ] Tạo `AppProperties` class với @ConfigurationProperties
- [ ] Cấu hình `application.yml` + `application-dev.yml` + `application-prod.yml`
- [ ] Tạo `.env` file và thêm vào `.gitignore`
- [ ] Tạo `WebConfig` với CORS configuration
- [ ] Tạo `OpenApiConfig` cho Swagger
- [ ] Cấu hình logging levels cho từng package
- [ ] Chạy ứng dụng với profile dev → kiểm tra log
- [ ] Test Swagger UI tại `http://localhost:8080/swagger-ui.html`
- [ ] Thử đổi port qua command line: `--server.port=9090`

### Kiến thức cần nắm

| Khái niệm | Hiểu chưa? |
|-----------|-------------|
| Thứ tự ưu tiên cấu hình | ☐ |
| .properties vs .yml | ☐ |
| @Value cơ bản + SpEL | ☐ |
| @ConfigurationProperties (group binding) | ☐ |
| Spring Profiles (dev, test, prod) | ☐ |
| Bảo mật secrets (env var, .env) | ☐ |
| CORS configuration | ☐ |
| Logging levels + pattern | ☐ |
| OpenAPI/Swagger config | ☐ |

---

**Trước đó:** [← Ngày 03 - Spring IoC & DI](day-03-spring-ioc-di.md)

**Tiếp theo:** [Ngày 05 - MySQL + Spring Data JPA Setup →](day-05-jpa-setup.md)
