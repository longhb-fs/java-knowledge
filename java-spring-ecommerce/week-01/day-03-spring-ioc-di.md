# Ngày 03: Spring IoC & Dependency Injection

## Mục tiêu hôm nay
- Hiểu Inversion of Control (IoC) và tại sao cần nó
- Hiểu Dependency Injection (DI) - 3 kiểu inject
- Nắm vững Bean Lifecycle trong Spring
- Phân biệt @Component, @Service, @Repository, @Controller
- Hiểu Bean Scope (singleton, prototype, request, session)
- Sử dụng @Configuration và @Bean

---

## 1. INVERSION OF CONTROL (IoC)

### 1.1. Vấn đề: Tight Coupling

```java
// ❌ Tight Coupling - class tự tạo dependency
public class OrderService {

    // OrderService tự quyết định dùng implementation nào
    private EmailService emailService = new GmailEmailService();  // hardcode!
    private PaymentService paymentService = new StripePaymentService();  // hardcode!

    public void createOrder(Order order) {
        // xử lý order...
        paymentService.charge(order.getTotal());
        emailService.sendConfirmation(order);
    }
}

// Vấn đề:
// 1. Muốn đổi sang SES Email → phải sửa OrderService
// 2. Test: không thể mock EmailService → phải gửi email thật khi test!
// 3. OrderService biết quá nhiều: biết cả implementation cụ thể
```

### 1.2. Giải pháp: IoC - Đảo ngược quyền kiểm soát

```java
// ✅ Loose Coupling - dependency được INJECT từ bên ngoài
public class OrderService {

    // OrderService CHỈ biết interface, KHÔNG biết implementation
    private final EmailService emailService;        // interface
    private final PaymentService paymentService;    // interface

    // Dependency được truyền vào qua constructor
    public OrderService(EmailService emailService, PaymentService paymentService) {
        this.emailService = emailService;
        this.paymentService = paymentService;
    }

    public void createOrder(Order order) {
        paymentService.charge(order.getTotal());
        emailService.sendConfirmation(order);
    }
}

// Lợi ích:
// 1. Đổi implementation: chỉ cần đổi config, KHÔNG sửa OrderService
// 2. Test: inject MockEmailService → KHÔNG gửi email thật
// 3. OrderService chỉ biết interface → loose coupling
```

### 1.3. IoC Container = Spring ApplicationContext

```
┌────────────────────────────────────────────────────┐
│            SPRING IoC CONTAINER                     │
│         (ApplicationContext)                         │
│                                                     │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐          │
│  │ Bean A   │  │ Bean B   │  │ Bean C   │  ...     │
│  │(Service) │  │(Repo)    │  │(Config)  │          │
│  └──────────┘  └──────────┘  └──────────┘          │
│       │              │              │                │
│       └──────────────┼──────────────┘                │
│                      │                               │
│              Quản lý vòng đời:                      │
│              • Tạo (Instantiate)                    │
│              • Inject dependencies                   │
│              • Initialize (@PostConstruct)           │
│              • Sử dụng                              │
│              • Destroy (@PreDestroy)                │
└────────────────────────────────────────────────────┘
```

**IoC Container chịu trách nhiệm:**
1. **Tạo** các object (Bean)
2. **Quản lý** vòng đời của Bean
3. **Inject** dependency giữa các Bean
4. **Cấu hình** Bean (scope, lazy init, etc.)

---

## 2. DEPENDENCY INJECTION - 3 KIỂU

### 2.1. Constructor Injection (KHUYẾN KHÍCH NHẤT)

```java
@Service
public class ProductService {

    private final ProductRepository productRepository;
    private final CategoryRepository categoryRepository;

    // Spring tự inject qua constructor
    // Từ Spring 4.3: nếu chỉ có 1 constructor → không cần @Autowired
    public ProductService(ProductRepository productRepository,
                          CategoryRepository categoryRepository) {
        this.productRepository = productRepository;
        this.categoryRepository = categoryRepository;
    }
}

// Với Lombok - gọn hơn:
@Service
@RequiredArgsConstructor  // Tự tạo constructor cho các final field
public class ProductService {
    private final ProductRepository productRepository;
    private final CategoryRepository categoryRepository;
    // Constructor được Lombok tự sinh
}
```

**Ưu điểm Constructor Injection:**
| Ưu điểm | Giải thích |
|----------|------------|
| **Immutable** | Field là `final` → không thể thay đổi sau khi tạo |
| **Rõ ràng** | Nhìn constructor biết ngay cần dependency gì |
| **Testable** | Dễ dàng truyền mock trong unit test |
| **Fail-fast** | Thiếu dependency → lỗi ngay khi start (không phải lúc runtime) |
| **Không cần @Autowired** | Spring 4.3+ tự detect constructor |

### 2.2. Setter Injection (ÍT DÙNG)

```java
@Service
public class NotificationService {

    private EmailService emailService;
    private SmsService smsService;

    @Autowired  // Bắt buộc phải có @Autowired
    public void setEmailService(EmailService emailService) {
        this.emailService = emailService;
    }

    @Autowired(required = false)  // Optional dependency
    public void setSmsService(SmsService smsService) {
        this.smsService = smsService;
    }
}
```

**Khi nào dùng:** Optional dependency (có thể null).

### 2.3. Field Injection (TRÁNH DÙNG)

```java
@Service
public class ProductService {

    @Autowired  // Inject trực tiếp vào field
    private ProductRepository productRepository;

    @Autowired
    private CategoryRepository categoryRepository;
}
```

**Tại sao tránh:**
- Không thể tạo immutable (không dùng `final` được)
- Khó test (phải dùng reflection để inject mock)
- Dependency ẩn - nhìn class không biết cần dependency gì
- Dễ tạo class có quá nhiều dependency mà không nhận ra

### 2.4. So sánh 3 kiểu

```
Constructor Injection          Setter Injection            Field Injection
──────────────────────        ──────────────────          ──────────────────
✅ Immutable (final)          ❌ Mutable                  ❌ Mutable
✅ Fail-fast                  ⚠️ Runtime error            ⚠️ Runtime error
✅ Dễ test                    ⚠️ Cần setter               ❌ Cần reflection
✅ Rõ dependency              ⚠️ Phải đọc method          ❌ Ẩn dependency
✅ Spring recommend           ⚠️ Optional deps            ❌ Tránh dùng
```

---

## 3. SPRING BEANS & ANNOTATIONS

### 3.1. Stereotype Annotations

```
┌─────────────────────────────────────────┐
│            @Component                     │
│    (annotation gốc - bean tổng quát)     │
│                                           │
│  ┌────────────┐  ┌──────────────────┐    │
│  │ @Service   │  │ @Repository      │    │
│  │            │  │                  │    │
│  │ Business   │  │ Data Access     │    │
│  │ Logic      │  │ Layer           │    │
│  │            │  │                  │    │
│  │ Không có   │  │ + Exception     │    │
│  │ logic đặc  │  │   Translation   │    │
│  │ biệt       │  │ (SQL Exception  │    │
│  │            │  │  → Spring Ex.)  │    │
│  └────────────┘  └──────────────────┘    │
│                                           │
│  ┌────────────┐  ┌──────────────────┐    │
│  │@Controller │  │@RestController   │    │
│  │            │  │                  │    │
│  │ MVC        │  │ = @Controller   │    │
│  │ Controller │  │ + @ResponseBody │    │
│  │ (trả View) │  │ (trả JSON)     │    │
│  └────────────┘  └──────────────────┘    │
└─────────────────────────────────────────┘
```

```java
// @Component - Bean tổng quát (utility, helper)
@Component
public class SlugUtils {
    public String toSlug(String text) {
        return text.toLowerCase().replaceAll("\\s+", "-");
    }
}

// @Service - Business logic layer
@Service
public class ProductService {
    private final ProductRepository repo;
    // business logic...
}

// @Repository - Data access layer
@Repository
public interface ProductRepository extends JpaRepository<Product, Long> {
    // Spring Data JPA tự tạo implementation
}

// @RestController - REST API layer
@RestController
@RequestMapping("/api/products")
public class ProductController {
    private final ProductService service;
    // handle HTTP requests...
}

// @Configuration - Class chứa cấu hình @Bean
@Configuration
public class RedisConfig {
    @Bean
    public RedisTemplate<String, Object> redisTemplate() {
        // tạo và cấu hình bean
    }
}
```

### 3.2. @Bean vs @Component

```java
// @Component: Đánh dấu class DO BẠN VIẾT
@Component  // Spring scan và tạo bean tự động
public class SlugUtils {
    public String toSlug(String text) { ... }
}

// @Bean: Tạo bean từ class CỦA THƯ VIỆN KHÁC (bạn không sửa được source code)
@Configuration
public class AppConfig {

    @Bean  // Tạo bean thủ công
    public ObjectMapper objectMapper() {
        // ObjectMapper là class của Jackson (thư viện ngoài)
        // Không thể thêm @Component vào source của nó
        return new ObjectMapper()
                .registerModule(new JavaTimeModule())
                .disable(SerializationFeature.WRITE_DATES_AS_TIMESTAMPS);
    }

    @Bean
    public RestTemplate restTemplate() {
        return new RestTemplate();
    }
}
```

| Tiêu chí | @Component | @Bean |
|----------|-----------|------|
| **Đặt ở đâu** | Trên class | Trên method (trong @Configuration) |
| **Tạo bean** | Tự động (component scan) | Thủ công (bạn viết code tạo) |
| **Dùng khi** | Class do bạn viết | Class thư viện ngoài |
| **Ví dụ** | Service, Repository, Controller | ObjectMapper, RestTemplate, DataSource |

### 3.3. Component Scanning

```java
@SpringBootApplication  // Bao gồm @ComponentScan
// Mặc định scan: com.example.ecommerce và tất cả sub-packages
public class EcommerceApplication { }

// Tùy chỉnh scan:
@SpringBootApplication
@ComponentScan(basePackages = {
    "com.example.ecommerce",
    "com.example.shared"  // Scan thêm package khác
})
public class EcommerceApplication { }
```

```
Component Scan quá trình:
com.example.ecommerce/
├── EcommerceApplication.java     ← @SpringBootApplication (bắt đầu scan từ đây)
├── config/
│   └── RedisConfig.java          ← @Configuration → scan ✅
├── service/
│   └── ProductService.java       ← @Service → scan ✅
├── repository/
│   └── ProductRepository.java    ← @Repository → scan ✅
├── controller/
│   └── ProductController.java    ← @RestController → scan ✅
└── util/
    └── SlugUtils.java            ← @Component → scan ✅

com.other.package/
└── SomeClass.java                ← KHÔNG scan (khác package) ❌
```

---

## 4. BEAN LIFECYCLE

### 4.1. Vòng đời Bean

```
┌──────────────────────────────────────────────────────────┐
│                    BEAN LIFECYCLE                          │
│                                                           │
│  1. Instantiation                                         │
│     └── Spring tạo object (new hoặc factory method)       │
│                │                                          │
│  2. Populate Properties                                   │
│     └── Inject dependencies (@Autowired)                  │
│                │                                          │
│  3. BeanNameAware.setBeanName()                          │
│     └── Thông báo tên bean                                │
│                │                                          │
│  4. BeanFactoryAware.setBeanFactory()                    │
│     └── Cho phép truy cập BeanFactory                    │
│                │                                          │
│  5. ApplicationContextAware.setApplicationContext()       │
│     └── Cho phép truy cập ApplicationContext              │
│                │                                          │
│  6. @PostConstruct                                        │
│     └── ⭐ QUAN TRỌNG: Chạy sau khi inject xong          │
│                │                                          │
│  7. InitializingBean.afterPropertiesSet()                │
│     └── Alternative cho @PostConstruct                    │
│                │                                          │
│  8. ═══ BEAN SẴN SÀNG SỬ DỤNG ═══                       │
│                │                                          │
│  9. @PreDestroy                                           │
│     └── ⭐ QUAN TRỌNG: Chạy trước khi bean bị hủy        │
│                │                                          │
│  10. DisposableBean.destroy()                             │
│      └── Alternative cho @PreDestroy                      │
│                │                                          │
│  11. Bean destroyed                                       │
└──────────────────────────────────────────────────────────┘
```

### 4.2. Ví dụ thực tế

```java
@Service
@RequiredArgsConstructor
@Slf4j
public class CacheWarmupService {

    private final ProductRepository productRepository;
    private final RedisTemplate<String, Object> redisTemplate;

    @PostConstruct  // Chạy SAU KHI bean được tạo + inject xong
    public void warmupCache() {
        log.info("🔥 Bắt đầu warm up cache...");

        // Load top products vào Redis
        List<Product> topProducts = productRepository.findTop10ByOrderBySoldDesc();
        topProducts.forEach(p ->
            redisTemplate.opsForValue().set("product:" + p.getId(), p)
        );

        log.info("✅ Đã warm up {} sản phẩm vào cache", topProducts.size());
    }

    @PreDestroy  // Chạy TRƯỚC KHI ứng dụng shutdown
    public void cleanup() {
        log.info("🧹 Dọn dẹp cache...");
        // Xóa cache keys nếu cần
    }
}
```

### 4.3. Ứng dụng thực tế

```java
@Component
@Slf4j
public class AppStartupRunner implements CommandLineRunner {

    @Override
    public void run(String... args) {
        log.info("═══════════════════════════════════════");
        log.info("  🚀 E-Commerce API đã khởi động!     ");
        log.info("  📋 Swagger: http://localhost:8080/swagger-ui.html");
        log.info("  💚 Health:  http://localhost:8080/actuator/health");
        log.info("═══════════════════════════════════════");
    }
}
```

---

## 5. BEAN SCOPE

### 5.1. Các loại Scope

```java
// ─── SINGLETON (mặc định) ───
// Chỉ tạo 1 instance duy nhất trong toàn bộ container
// Mọi nơi inject đều nhận CÙNG 1 object
@Service  // Mặc định là Singleton
public class ProductService { }

// ─── PROTOTYPE ───
// Mỗi lần inject = tạo instance MỚI
@Component
@Scope("prototype")
public class ShoppingCart {
    private List<CartItem> items = new ArrayList<>();
}

// ─── REQUEST (chỉ dùng trong Web app) ───
// 1 instance cho mỗi HTTP request
@Component
@Scope(value = WebApplicationContext.SCOPE_REQUEST, proxyMode = ScopedProxyMode.TARGET_CLASS)
public class RequestContext {
    private String requestId;
    private String userId;
}

// ─── SESSION (chỉ dùng trong Web app) ───
// 1 instance cho mỗi HTTP session
@Component
@Scope(value = WebApplicationContext.SCOPE_SESSION, proxyMode = ScopedProxyMode.TARGET_CLASS)
public class UserSession {
    private User currentUser;
}
```

### 5.2. Singleton vs Prototype

```
SINGLETON (mặc định):
┌──────────────────┐
│  IoC Container    │
│  ┌──────────────┐│      ControllerA.service ──┐
│  │ ProductService││                             ├──> CÙNG 1 object
│  │  (1 instance) ││      ControllerB.service ──┘
│  └──────────────┘│
└──────────────────┘

PROTOTYPE:
┌──────────────────┐
│  IoC Container    │
│                   │      inject lần 1 ──> ShoppingCart #1 (object mới)
│  ShoppingCart     │      inject lần 2 ──> ShoppingCart #2 (object mới)
│  (factory)        │      inject lần 3 ──> ShoppingCart #3 (object mới)
│                   │
└──────────────────┘
```

**Quy tắc chọn scope:**

| Scope | Khi nào dùng | Ví dụ |
|-------|-------------|-------|
| **Singleton** | Stateless service (không giữ state) | Service, Repository, Controller |
| **Prototype** | Mỗi lần dùng cần object mới | Form builder, Report generator |
| **Request** | Data gắn với 1 HTTP request | Request logging context |
| **Session** | Data gắn với 1 user session | Shopping cart (session-based) |

---

## 6. CONDITIONAL BEANS & PROFILES

### 6.1. @Profile - Bean theo môi trường

```java
// Interface chung
public interface StorageService {
    String upload(MultipartFile file);
    void delete(String fileUrl);
}

// Implementation cho Development - lưu local
@Service
@Profile("dev")  // Chỉ tạo bean khi profile = dev
@Slf4j
public class LocalStorageService implements StorageService {

    @Override
    public String upload(MultipartFile file) {
        log.info("📁 Lưu file vào local: {}", file.getOriginalFilename());
        // Lưu vào thư mục uploads/
        return "/uploads/" + file.getOriginalFilename();
    }

    @Override
    public void delete(String fileUrl) {
        log.info("🗑️ Xóa file local: {}", fileUrl);
    }
}

// Implementation cho Production - lưu S3
@Service
@Profile("prod")  // Chỉ tạo bean khi profile = prod
@Slf4j
public class S3StorageService implements StorageService {

    @Override
    public String upload(MultipartFile file) {
        log.info("☁️ Upload file lên S3: {}", file.getOriginalFilename());
        // Upload lên AWS S3
        return "https://s3.amazonaws.com/bucket/" + file.getOriginalFilename();
    }

    @Override
    public void delete(String fileUrl) {
        log.info("🗑️ Xóa file S3: {}", fileUrl);
    }
}

// Service sử dụng - KHÔNG CẦN BIẾT implementation nào
@Service
@RequiredArgsConstructor
public class ProductService {
    private final StorageService storageService;  // Spring inject đúng implementation theo profile

    public void uploadProductImage(MultipartFile file) {
        String url = storageService.upload(file);  // Dev: local, Prod: S3
    }
}
```

### 6.2. @Conditional - Bean có điều kiện

```java
@Configuration
public class CacheConfig {

    // Chỉ tạo Redis cache bean khi có property cache.type=redis
    @Bean
    @ConditionalOnProperty(name = "cache.type", havingValue = "redis")
    public CacheManager redisCacheManager(RedisConnectionFactory factory) {
        return RedisCacheManager.builder(factory).build();
    }

    // Fallback: dùng in-memory cache khi không có Redis
    @Bean
    @ConditionalOnProperty(name = "cache.type", havingValue = "memory", matchIfMissing = true)
    public CacheManager inMemoryCacheManager() {
        return new ConcurrentMapCacheManager("products", "categories");
    }
}
```

---

## 7. THỰC HÀNH: XÂY DỰNG SERVICE LAYER

### 7.1. Tạo interface + implementation

```java
// service/ProductService.java (interface)
package com.example.ecommerce.service;

import com.example.ecommerce.dto.request.ProductCreateRequest;
import com.example.ecommerce.dto.response.ProductResponse;
import com.example.ecommerce.dto.response.PagedResponse;
import org.springframework.data.domain.Pageable;

public interface ProductService {
    ProductResponse create(ProductCreateRequest request);
    ProductResponse getById(Long id);
    PagedResponse<ProductResponse> getAll(Pageable pageable);
    ProductResponse update(Long id, ProductCreateRequest request);
    void delete(Long id);
}
```

```java
// service/impl/ProductServiceImpl.java (implementation)
package com.example.ecommerce.service.impl;

import com.example.ecommerce.dto.request.ProductCreateRequest;
import com.example.ecommerce.dto.response.PagedResponse;
import com.example.ecommerce.dto.response.ProductResponse;
import com.example.ecommerce.entity.Product;
import com.example.ecommerce.exception.ResourceNotFoundException;
import com.example.ecommerce.repository.ProductRepository;
import com.example.ecommerce.service.ProductService;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.data.domain.Page;
import org.springframework.data.domain.Pageable;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

@Service
@RequiredArgsConstructor
@Slf4j
@Transactional(readOnly = true)  // Mặc định read-only cho performance
public class ProductServiceImpl implements ProductService {

    private final ProductRepository productRepository;

    @Override
    @Transactional  // Override: cho phép write
    public ProductResponse create(ProductCreateRequest request) {
        log.info("Tạo sản phẩm mới: {}", request.getName());

        Product product = Product.builder()
                .name(request.getName())
                .price(request.getPrice())
                .description(request.getDescription())
                .build();

        product = productRepository.save(product);
        log.info("Đã tạo sản phẩm ID: {}", product.getId());

        return toResponse(product);
    }

    @Override
    public ProductResponse getById(Long id) {
        Product product = productRepository.findById(id)
                .orElseThrow(() -> new ResourceNotFoundException("Sản phẩm", "id", id));
        return toResponse(product);
    }

    @Override
    public PagedResponse<ProductResponse> getAll(Pageable pageable) {
        Page<Product> page = productRepository.findAll(pageable);
        Page<ProductResponse> responsePage = page.map(this::toResponse);
        return PagedResponse.from(responsePage);
    }

    @Override
    @Transactional
    public ProductResponse update(Long id, ProductCreateRequest request) {
        Product product = productRepository.findById(id)
                .orElseThrow(() -> new ResourceNotFoundException("Sản phẩm", "id", id));

        product.setName(request.getName());
        product.setPrice(request.getPrice());
        product.setDescription(request.getDescription());

        product = productRepository.save(product);
        return toResponse(product);
    }

    @Override
    @Transactional
    public void delete(Long id) {
        if (!productRepository.existsById(id)) {
            throw new ResourceNotFoundException("Sản phẩm", "id", id);
        }
        productRepository.deleteById(id);
        log.info("Đã xóa sản phẩm ID: {}", id);
    }

    // Private helper - chuyển Entity → Response DTO
    private ProductResponse toResponse(Product product) {
        return ProductResponse.builder()
                .id(product.getId())
                .name(product.getName())
                .price(product.getPrice())
                .description(product.getDescription())
                .createdAt(product.getCreatedAt())
                .build();
    }
}
```

### 7.2. Tạo Exception class

```java
// exception/ResourceNotFoundException.java
package com.example.ecommerce.exception;

import org.springframework.http.HttpStatus;
import org.springframework.web.bind.annotation.ResponseStatus;

@ResponseStatus(HttpStatus.NOT_FOUND)
public class ResourceNotFoundException extends RuntimeException {

    public ResourceNotFoundException(String resourceName, String fieldName, Object fieldValue) {
        super(String.format("%s không tìm thấy với %s: '%s'", resourceName, fieldName, fieldValue));
    }
}
```

---

## 8. BÀI TẬP & CHECKLIST

### Bài tập thực hành

- [ ] Tạo interface `StorageService` với 2 implementation (Local, Mock S3)
- [ ] Dùng `@Profile` để switch giữa dev/prod
- [ ] Tạo `ProductService` interface + implementation
- [ ] Dùng `@RequiredArgsConstructor` cho constructor injection
- [ ] Tạo 1 bean với `@PostConstruct` in ra log khi startup
- [ ] Tạo `CommandLineRunner` in banner khi ứng dụng start
- [ ] Tạo `ResourceNotFoundException` class
- [ ] Test: inject `ProductService` vào controller, gọi getById

### Kiến thức cần nắm

| Khái niệm | Hiểu chưa? |
|-----------|-------------|
| IoC là gì, tại sao cần | ☐ |
| 3 kiểu DI (Constructor, Setter, Field) | ☐ |
| Tại sao Constructor Injection tốt nhất | ☐ |
| @Component vs @Service vs @Repository vs @Controller | ☐ |
| @Bean vs @Component | ☐ |
| Component Scanning hoạt động thế nào | ☐ |
| Bean Lifecycle (PostConstruct, PreDestroy) | ☐ |
| Bean Scope (Singleton vs Prototype) | ☐ |
| @Profile cho môi trường khác nhau | ☐ |
| @Transactional (readOnly, default) | ☐ |

---

**Trước đó:** [← Ngày 02 - Project Setup](day-02-project-setup.md)

**Tiếp theo:** [Ngày 04 - Configuration & Profiles →](day-04-configuration.md)
