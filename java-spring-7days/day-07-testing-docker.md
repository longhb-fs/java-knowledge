# Day 7: Testing + Docker

> **Mục tiêu:** Viết unit test, integration test cho các layer, đóng gói app bằng Docker, chạy full stack với Docker Compose.
> **Thời gian:** ~6-8 giờ | **Output:** Test suite xanh + `docker compose up` chạy cả hệ thống.

---

## 1. Testing Pyramid

### 1.1 Kim Tự Tháp Testing

```
            /\
           /  \
          / E2E \            (10%) — Full system tests
         /  Tests \          Chạy trên browser/API thực, chậm nhất
        /──────────\
       / Integration \       (20%) — DB + API tests
      /    Tests      \      Test với database thật, HTTP request thật
     /────────────────\
    /    Unit Tests     \    (70%) — Fast, isolated
   /                     \   Mock dependencies, chạy trong < 1 giây
  /───────────────────────\
```

### 1.2 Khi nào viết loại nào?

| Loại | Khi nào | Tốc độ | Độ tin cậy |
|------|---------|--------|------------|
| **Unit Test** | Business logic, service methods, utility functions | Cực nhanh (ms) | Trung bình |
| **Integration Test** | Repository queries, API endpoints, external services | Chậm (giây) | Cao |
| **E2E Test** | Critical user flows (checkout, payment) | Rất chậm (phút) | Rất cao |

### 1.3 Dependencies đã có sẵn (từ Day 1 pom.xml)

```xml
<!-- spring-boot-starter-test bao gồm: JUnit 5, Mockito, AssertJ, MockMvc, JSONPath -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-test</artifactId>
    <scope>test</scope>
</dependency>

<!-- Spring Security Test -->
<dependency>
    <groupId>org.springframework.security</groupId>
    <artifactId>spring-security-test</artifactId>
    <scope>test</scope>
</dependency>

<!-- TestContainers — chạy MySQL/Redis thật trong Docker khi test -->
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
```

> **Không cần thêm gì mới** — tất cả đã được khai báo từ Day 1. Chỉ việc viết test.

---

## 2. JUnit 5 Basics

### 2.1 Annotations Quan Trọng

| Annotation | Chức năng |
|-----------|-----------|
| `@Test` | Đánh dấu method là test case |
| `@BeforeEach` | Chạy trước **mỗi** test method (setup data) |
| `@AfterEach` | Chạy sau **mỗi** test method (cleanup) |
| `@BeforeAll` | Chạy 1 lần trước **tất cả** tests (static method) |
| `@DisplayName("...")` | Đặt tên test dễ đọc cho dễ hiểu |
| `@Nested` | Nhóm các test liên quan thành inner class |
| `@Disabled` | Tạm tắt test này |
| `@ParameterizedTest` | Chạy test với nhiều bộ dữ liệu khác nhau |

### 2.2 Assertions Cơ Bản

```java
import static org.junit.jupiter.api.Assertions.*;

assertEquals(expected, actual);              // So sánh bằng
assertTrue(condition);                        // Kiểm tra true
assertFalse(condition);                       // Kiểm tra false
assertNotNull(object);                        // Kiểm tra không null
assertThrows(Exception.class, () -> {...});   // Kiểm tra ném exception
assertAll(                                    // Chạy nhiều assertion, báo cáo tất cả lỗi
    () -> assertEquals("a", result.getA()),
    () -> assertEquals("b", result.getB())
);
```

### 2.3 Test Naming Convention

Format: `methodName_scenario_expectedResult`

```java
@Test
void getById_existingId_returnsProduct() { }

@Test
void getById_nonExistingId_throwsException() { }

@Test
void create_duplicateSlug_throwsDuplicateException() { }

@Test
void delete_existingId_success() { }
```

> **Quy tắc:** Tên test phải đọc hiểu như 1 câu tiếng Anh — khi test fail, bạn biết ngay lỗi ở đâu mà không cần đọc code.

---

## 3. Mockito Basics

### 3.1 Core Annotations

```java
@ExtendWith(MockitoExtension.class)  // Bật Mockito cho JUnit 5
class ProductServiceTest {

    @Mock                              // Tạo object giả (mock) — không gọi logic thật
    private ProductRepository productRepository;

    @InjectMocks                       // Tạo object thật + inject các @Mock vào
    private ProductService productService;
}
```

### 3.2 Stubbing — Định nghĩa hành vi giả

```java
// Khi gọi findById(1L) → trả về testProduct
when(productRepository.findById(1L)).thenReturn(Optional.of(testProduct));

// Khi gọi findById(99L) → trả về empty
when(productRepository.findById(99L)).thenReturn(Optional.empty());

// Khi gọi save(bất kỳ Product nào) → trả về savedProduct
when(productRepository.save(any(Product.class))).thenReturn(savedProduct);

// Khi gọi delete → ném exception
when(productRepository.delete(any())).thenThrow(new RuntimeException("DB error"));
```

### 3.3 Verification — Xác nhận method được gọi

```java
// Xác nhận findById được gọi đúng 1 lần với argument = 1L
verify(productRepository).findById(1L);
verify(productRepository, times(1)).findById(1L);

// Xác nhận delete KHÔNG được gọi
verify(productRepository, never()).delete(any());

// Xác nhận save được gọi ít nhất 1 lần
verify(productRepository, atLeastOnce()).save(any());
```

### 3.4 ArgumentCaptor — Bắt argument truyền vào

```java
ArgumentCaptor<Product> captor = ArgumentCaptor.forClass(Product.class);
verify(productRepository).save(captor.capture());

Product savedProduct = captor.getValue();
assertThat(savedProduct.getName()).isEqualTo("iPhone 15");
assertThat(savedProduct.getCategory()).isEqualTo(testCategory);
```

> **ArgumentCaptor** rất hữu ích khi bạn muốn kiểm tra object được truyền vào repository.save() có đúng không — đặc biệt khi service modify object trước khi save.

---

## 4. Unit Test: ProductService

Đây là test quan trọng nhất — test business logic mà không cần database, không cần Spring context.

```java
package com.example.ecommerce.service;

import com.example.ecommerce.dto.request.ProductCreateRequest;
import com.example.ecommerce.dto.response.ProductResponse;
import com.example.ecommerce.entity.Category;
import com.example.ecommerce.entity.Product;
import com.example.ecommerce.exception.ResourceNotFoundException;
import com.example.ecommerce.mapper.ProductMapper;
import com.example.ecommerce.repository.CategoryRepository;
import com.example.ecommerce.repository.ProductRepository;
import org.junit.jupiter.api.*;
import org.junit.jupiter.api.extension.ExtendWith;
import org.mockito.ArgumentCaptor;
import org.mockito.InjectMocks;
import org.mockito.Mock;
import org.mockito.junit.jupiter.MockitoExtension;

import java.math.BigDecimal;
import java.util.Optional;

import static org.assertj.core.api.Assertions.*;
import static org.mockito.ArgumentMatchers.*;
import static org.mockito.Mockito.*;

@ExtendWith(MockitoExtension.class)
@DisplayName("ProductService Tests")
class ProductServiceTest {

    @Mock private ProductRepository productRepository;
    @Mock private CategoryRepository categoryRepository;
    @Mock private ProductMapper productMapper;
    @InjectMocks private ProductService productService;

    private Product testProduct;
    private Category testCategory;
    private ProductCreateRequest createRequest;
    private ProductResponse expectedResponse;

    @BeforeEach
    void setUp() {
        // Chuẩn bị data dùng chung cho mỗi test
        testCategory = new Category();
        testCategory.setId(1L);
        testCategory.setName("Electronics");

        testProduct = new Product();
        testProduct.setId(1L);
        testProduct.setName("iPhone 15");
        testProduct.setSlug("iphone-15");
        testProduct.setPrice(new BigDecimal("999.99"));
        testProduct.setCategory(testCategory);

        createRequest = new ProductCreateRequest();
        createRequest.setName("iPhone 15");
        createRequest.setPrice(new BigDecimal("999.99"));
        createRequest.setCategoryId(1L);

        expectedResponse = new ProductResponse();
        expectedResponse.setId(1L);
        expectedResponse.setName("iPhone 15");
    }

    @Nested
    @DisplayName("getById")
    class GetById {

        @Test
        @DisplayName("should return product when found")
        void getById_existingId_returnsProduct() {
            // Given (Arrange) — chuẩn bị mock
            when(productRepository.findById(1L)).thenReturn(Optional.of(testProduct));
            when(productMapper.toResponse(testProduct)).thenReturn(expectedResponse);

            // When (Act) — gọi method cần test
            ProductResponse result = productService.getById(1L);

            // Then (Assert) — kiểm tra kết quả
            assertThat(result).isNotNull();
            assertThat(result.getName()).isEqualTo("iPhone 15");
            verify(productRepository).findById(1L);
        }

        @Test
        @DisplayName("should throw exception when not found")
        void getById_nonExistingId_throwsException() {
            // Given
            when(productRepository.findById(99L)).thenReturn(Optional.empty());

            // When & Then
            assertThatThrownBy(() -> productService.getById(99L))
                    .isInstanceOf(ResourceNotFoundException.class)
                    .hasMessageContaining("Product not found");
        }
    }

    @Nested
    @DisplayName("create")
    class Create {

        @Test
        @DisplayName("should create product successfully")
        void create_validRequest_returnsProduct() {
            // Given
            when(categoryRepository.findById(1L)).thenReturn(Optional.of(testCategory));
            when(productMapper.toEntity(createRequest)).thenReturn(testProduct);
            when(productRepository.existsBySlug(anyString())).thenReturn(false);
            when(productRepository.save(any(Product.class))).thenReturn(testProduct);
            when(productMapper.toResponse(testProduct)).thenReturn(expectedResponse);

            // When
            ProductResponse result = productService.create(createRequest);

            // Then
            assertThat(result).isNotNull();
            assertThat(result.getName()).isEqualTo("iPhone 15");

            // Bắt argument truyền vào save() để kiểm tra
            ArgumentCaptor<Product> captor = ArgumentCaptor.forClass(Product.class);
            verify(productRepository).save(captor.capture());
            assertThat(captor.getValue().getCategory()).isEqualTo(testCategory);
        }

        @Test
        @DisplayName("should throw when category not found")
        void create_invalidCategory_throwsException() {
            when(categoryRepository.findById(99L)).thenReturn(Optional.empty());
            createRequest.setCategoryId(99L);

            assertThatThrownBy(() -> productService.create(createRequest))
                    .isInstanceOf(ResourceNotFoundException.class);
        }
    }

    @Nested
    @DisplayName("delete")
    class Delete {

        @Test
        @DisplayName("should delete product successfully")
        void delete_existingId_success() {
            when(productRepository.findById(1L)).thenReturn(Optional.of(testProduct));

            productService.delete(1L);

            verify(productRepository).delete(testProduct);
        }

        @Test
        @DisplayName("should throw when product not found")
        void delete_nonExistingId_throwsException() {
            when(productRepository.findById(99L)).thenReturn(Optional.empty());

            assertThatThrownBy(() -> productService.delete(99L))
                    .isInstanceOf(ResourceNotFoundException.class);
            verify(productRepository, never()).delete(any());
        }
    }
}
```

**Chạy test:**

```bash
# Chạy tất cả tests
./mvnw test

# Chạy 1 class cụ thể
./mvnw test -Dtest=ProductServiceTest

# Chạy 1 method cụ thể
./mvnw test -Dtest="ProductServiceTest#getById_existingId_returnsProduct"
```

> **Pattern Given-When-Then** (hay Arrange-Act-Assert) là quy tắc vàng khi viết test. Mỗi test chỉ nên kiểm tra **1 hành vi duy nhất**.

---

## 5. AssertJ — Fluent Assertions

AssertJ cung cấp cách viết assertion đọc như tiếng Anh, thay thế JUnit Assertions mặc định.

### 5.1 So Sánh Cơ Bản

```java
import static org.assertj.core.api.Assertions.*;

// String assertions
assertThat(product.getName()).isEqualTo("iPhone 15");
assertThat(product.getName()).startsWith("iPhone");
assertThat(product.getName()).contains("Phone");
assertThat(product.getName()).isNotBlank();

// Number assertions
assertThat(product.getPrice()).isGreaterThan(BigDecimal.ZERO);
assertThat(product.getStockQuantity()).isBetween(0, 1000);

// Boolean assertions
assertThat(product.isActive()).isTrue();

// Null check
assertThat(result).isNotNull();
assertThat(result.getDescription()).isNull();
```

### 5.2 Exception Assertions

```java
// Kiểm tra exception type và message
assertThatThrownBy(() -> productService.getById(99L))
        .isInstanceOf(ResourceNotFoundException.class)
        .hasMessageContaining("Product not found")
        .hasMessageContaining("99");

// Kiểm tra KHÔNG ném exception
assertThatCode(() -> productService.delete(1L))
        .doesNotThrowAnyException();
```

### 5.3 Collection Assertions

```java
List<ProductResponse> products = productService.getAll();

assertThat(products)
        .hasSize(3)
        .isNotEmpty()
        .extracting(ProductResponse::getName)      // Lấy tên từ mỗi element
        .containsExactly("iPhone 15", "Samsung S24", "Pixel 8");

// Filter và kiểm tra
assertThat(products)
        .filteredOn(p -> p.getPrice().compareTo(new BigDecimal("500")) > 0)
        .hasSize(2)
        .extracting(ProductResponse::getName)
        .contains("iPhone 15", "Samsung S24");
```

> **Tip:** AssertJ có IDE auto-complete tuyệt vời — chỉ viết `assertThat(x).` rồi IDE gợi ý tất cả assertion phù hợp với kiểu dữ liệu của `x`.

---

## 6. Repository Test: @DataJpaTest

Test repository với **database thật** (H2 in-memory hoặc TestContainers MySQL).

```java
package com.example.ecommerce.repository;

import com.example.ecommerce.entity.Category;
import com.example.ecommerce.entity.Product;
import org.junit.jupiter.api.*;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.autoconfigure.jdbc.AutoConfigureTestDatabase;
import org.springframework.boot.test.autoconfigure.orm.jpa.DataJpaTest;
import org.springframework.boot.test.autoconfigure.orm.jpa.TestEntityManager;

import java.math.BigDecimal;
import java.util.List;
import java.util.Optional;

import static org.assertj.core.api.Assertions.*;

@DataJpaTest  // Chỉ load JPA layer (Repository + EntityManager), KHÔNG load full context
@AutoConfigureTestDatabase(replace = AutoConfigureTestDatabase.Replace.NONE)
// ↑ Dùng database thực (MySQL qua TestContainers) thay vì H2 in-memory
@DisplayName("ProductRepository Tests")
class ProductRepositoryTest {

    @Autowired
    private TestEntityManager entityManager;  // Helper để persist test data

    @Autowired
    private ProductRepository productRepository;

    private Category category;

    @BeforeEach
    void setUp() {
        // Tạo test data trực tiếp qua EntityManager (không qua repository)
        category = new Category();
        category.setName("Test Category");
        category.setSlug("test-category");
        category.setActive(true);
        entityManager.persist(category);

        Product product1 = new Product();
        product1.setName("Test Product");
        product1.setSlug("test-product");
        product1.setPrice(new BigDecimal("100.00"));
        product1.setStockQuantity(10);
        product1.setCategory(category);
        product1.setActive(true);
        entityManager.persist(product1);

        Product product2 = new Product();
        product2.setName("Expensive Product");
        product2.setSlug("expensive-product");
        product2.setPrice(new BigDecimal("500.00"));
        product2.setStockQuantity(5);
        product2.setCategory(category);
        product2.setActive(true);
        entityManager.persist(product2);

        entityManager.flush();  // Force SQL INSERT ngay lập tức
    }

    @Test
    @DisplayName("should find product by slug")
    void findBySlug_existingSlug_returnsProduct() {
        Optional<Product> result = productRepository.findBySlug("test-product");

        assertThat(result).isPresent();
        assertThat(result.get().getName()).isEqualTo("Test Product");
        assertThat(result.get().getPrice()).isEqualByComparingTo(new BigDecimal("100.00"));
    }

    @Test
    @DisplayName("should return empty when slug not found")
    void findBySlug_nonExistingSlug_returnsEmpty() {
        Optional<Product> result = productRepository.findBySlug("non-existing");

        assertThat(result).isEmpty();
    }

    @Test
    @DisplayName("should find products by category")
    void findByCategoryId_existingCategory_returnsProducts() {
        List<Product> results = productRepository.findByCategoryId(category.getId());

        assertThat(results).hasSize(2);
        assertThat(results)
                .extracting(Product::getName)
                .containsExactlyInAnyOrder("Test Product", "Expensive Product");
    }

    @Test
    @DisplayName("should find products in price range")
    void findByPriceRange_validRange_returnsProducts() {
        List<Product> results = productRepository.findByPriceRange(
                new BigDecimal("50"), new BigDecimal("150"));

        assertThat(results).hasSize(1);
        assertThat(results.get(0).getName()).isEqualTo("Test Product");
    }

    @Test
    @DisplayName("should count products by category")
    void countByCategoryIdAndActiveTrue_returnsCorrectCount() {
        long count = productRepository.countByCategoryIdAndActiveTrue(category.getId());

        assertThat(count).isEqualTo(2);
    }
}
```

**Giải thích `@DataJpaTest`:**

| Đặc điểm | Mô tả |
|----------|-------|
| Chỉ load JPA beans | Repository, EntityManager, DataSource — KHÔNG load Service, Controller |
| Transactional | Mỗi test tự động rollback sau khi chạy xong |
| Auto-config | Tự động cấu hình embedded DB hoặc test DB |
| Nhanh hơn `@SpringBootTest` | Vì chỉ load 1 phần context |

> **Lưu ý về @MockBean deprecation:** Từ Spring Boot 3.4+, `@MockBean` đã bị deprecated. Thay vào đó, sử dụng `@MockitoBean` (cùng chức năng, tên mới). Với unit test (không cần Spring context), vẫn dùng `@Mock` + `@InjectMocks` như bình thường.

---

## 7. Controller Test: @WebMvcTest

Test REST API endpoints với **MockMvc** — gửi HTTP request giả, kiểm tra response.

```java
package com.example.ecommerce.controller;

import com.example.ecommerce.dto.request.ProductCreateRequest;
import com.example.ecommerce.dto.response.ProductResponse;
import com.example.ecommerce.exception.ResourceNotFoundException;
import com.example.ecommerce.service.ProductService;
import com.fasterxml.jackson.databind.ObjectMapper;
import org.junit.jupiter.api.*;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.autoconfigure.web.servlet.WebMvcTest;
import org.springframework.boot.test.mock.bean.MockBean;
import org.springframework.http.MediaType;
import org.springframework.test.web.servlet.MockMvc;

import java.math.BigDecimal;

import static org.mockito.ArgumentMatchers.*;
import static org.mockito.Mockito.*;
import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.*;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.*;

@WebMvcTest(ProductController.class)  // Chỉ load ProductController + Web layer
@DisplayName("ProductController Tests")
class ProductControllerTest {

    @Autowired
    private MockMvc mockMvc;          // Gửi HTTP request giả

    @Autowired
    private ObjectMapper objectMapper; // Serialize object → JSON

    @MockBean  // Hoặc @MockitoBean trong Spring Boot 3.4+
    private ProductService productService;

    // ── GET /api/v1/products/{id} ──

    @Test
    @DisplayName("GET /api/v1/products/{id} - should return product")
    void getById_existingId_returns200() throws Exception {
        // Given
        ProductResponse response = new ProductResponse();
        response.setId(1L);
        response.setName("iPhone 15");

        when(productService.getById(1L)).thenReturn(response);

        // When & Then
        mockMvc.perform(get("/api/v1/products/1"))
                .andExpect(status().isOk())
                .andExpect(jsonPath("$.success").value(true))
                .andExpect(jsonPath("$.data.id").value(1))
                .andExpect(jsonPath("$.data.name").value("iPhone 15"));

        verify(productService).getById(1L);
    }

    @Test
    @DisplayName("GET /api/v1/products/{id} - should return 404 when not found")
    void getById_nonExistingId_returns404() throws Exception {
        when(productService.getById(99L))
                .thenThrow(new ResourceNotFoundException("Product", "id", 99L));

        mockMvc.perform(get("/api/v1/products/99"))
                .andExpect(status().isNotFound())
                .andExpect(jsonPath("$.success").value(false))
                .andExpect(jsonPath("$.message").exists());
    }

    // ── POST /api/v1/products ──

    @Test
    @DisplayName("POST /api/v1/products - should create product")
    void create_validRequest_returns201() throws Exception {
        // Given
        ProductCreateRequest request = new ProductCreateRequest();
        request.setName("New Product");
        request.setPrice(new BigDecimal("99.99"));
        request.setCategoryId(1L);

        ProductResponse response = new ProductResponse();
        response.setId(1L);
        response.setName("New Product");

        when(productService.create(any())).thenReturn(response);

        // When & Then
        mockMvc.perform(post("/api/v1/products")
                        .contentType(MediaType.APPLICATION_JSON)
                        .content(objectMapper.writeValueAsString(request)))
                .andExpect(status().isCreated())
                .andExpect(jsonPath("$.success").value(true))
                .andExpect(jsonPath("$.data.name").value("New Product"));
    }

    @Test
    @DisplayName("POST /api/v1/products - should return 422 for invalid request")
    void create_invalidRequest_returns422() throws Exception {
        ProductCreateRequest request = new ProductCreateRequest();
        // Missing required fields: name, price, categoryId

        mockMvc.perform(post("/api/v1/products")
                        .contentType(MediaType.APPLICATION_JSON)
                        .content(objectMapper.writeValueAsString(request)))
                .andExpect(status().isUnprocessableEntity());
    }

    // ── DELETE /api/v1/products/{id} ──

    @Test
    @DisplayName("DELETE /api/v1/products/{id} - should return 204")
    void delete_existingId_returns204() throws Exception {
        doNothing().when(productService).delete(1L);

        mockMvc.perform(delete("/api/v1/products/1"))
                .andExpect(status().isNoContent());

        verify(productService).delete(1L);
    }

    @Test
    @DisplayName("DELETE /api/v1/products/{id} - should return 404 when not found")
    void delete_nonExistingId_returns404() throws Exception {
        doThrow(new ResourceNotFoundException("Product", "id", 99L))
                .when(productService).delete(99L);

        mockMvc.perform(delete("/api/v1/products/99"))
                .andExpect(status().isNotFound());
    }
}
```

### 7.1 Test với Security Enabled

Nếu API yêu cầu authentication, thêm `@WithMockUser`:

```java
import org.springframework.security.test.context.support.WithMockUser;

@Test
@WithMockUser(roles = "ADMIN")  // Giả lập user đã login với role ADMIN
@DisplayName("DELETE should work for admin users")
void delete_asAdmin_returns204() throws Exception {
    doNothing().when(productService).delete(1L);

    mockMvc.perform(delete("/api/v1/products/1"))
            .andExpect(status().isNoContent());
}

@Test
@DisplayName("DELETE should return 401 for unauthenticated user")
void delete_unauthenticated_returns401() throws Exception {
    mockMvc.perform(delete("/api/v1/products/1"))
            .andExpect(status().isUnauthorized());
}
```

> **Lưu ý:** Nếu SecurityConfig chặn request trong `@WebMvcTest`, bạn cần import config:
> `@Import(SecurityConfig.class)` hoặc dùng `@AutoConfigureMockMvc(addFilters = false)` để tắt security filter trong test.

---

## 8. TestContainers — Integration Test với Database Thật

TestContainers khởi động **MySQL thật trong Docker** khi chạy test — đảm bảo query chạy đúng trên database thật, không phải H2 in-memory (có thể khác SQL syntax).

### 8.1 Abstract Base Class

```java
package com.example.ecommerce;

import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.test.context.DynamicPropertyRegistry;
import org.springframework.test.context.DynamicPropertySource;
import org.testcontainers.containers.GenericContainer;
import org.testcontainers.containers.MySQLContainer;
import org.testcontainers.junit.jupiter.Container;
import org.testcontainers.junit.jupiter.Testcontainers;

@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
@Testcontainers
abstract class AbstractIntegrationTest {

    // MySQL container — tự động start trước khi test chạy
    @Container
    static MySQLContainer<?> mysql = new MySQLContainer<>("mysql:8.0")
            .withDatabaseName("ecommerce_test")
            .withUsername("test")
            .withPassword("test");

    // Redis container
    @Container
    static GenericContainer<?> redis = new GenericContainer<>("redis:7-alpine")
            .withExposedPorts(6379);

    // Inject connection properties từ container vào Spring context
    @DynamicPropertySource
    static void configureProperties(DynamicPropertyRegistry registry) {
        registry.add("spring.datasource.url", mysql::getJdbcUrl);
        registry.add("spring.datasource.username", mysql::getUsername);
        registry.add("spring.datasource.password", mysql::getPassword);
        registry.add("spring.data.redis.host", redis::getHost);
        registry.add("spring.data.redis.port", () -> redis.getMappedPort(6379));
        registry.add("spring.jpa.hibernate.ddl-auto", () -> "create-drop");
        registry.add("spring.flyway.enabled", () -> "false");
    }
}
```

### 8.2 Sử dụng trong Integration Test

```java
class ProductIntegrationTest extends AbstractIntegrationTest {

    @Autowired
    private TestRestTemplate restTemplate;  // HTTP client thật

    @Autowired
    private ProductRepository productRepository;

    @Test
    @DisplayName("Full CRUD flow should work")
    void fullCrudFlow() {
        // Create
        ProductCreateRequest request = new ProductCreateRequest();
        request.setName("Integration Test Product");
        request.setPrice(new BigDecimal("199.99"));
        request.setCategoryId(1L);

        ResponseEntity<ApiResponse> createResponse = restTemplate.postForEntity(
                "/api/v1/products", request, ApiResponse.class);
        assertThat(createResponse.getStatusCode()).isEqualTo(HttpStatus.CREATED);

        // Read
        ResponseEntity<ApiResponse> getResponse = restTemplate.getForEntity(
                "/api/v1/products/1", ApiResponse.class);
        assertThat(getResponse.getStatusCode()).isEqualTo(HttpStatus.OK);

        // Verify in database
        assertThat(productRepository.count()).isGreaterThan(0);
    }
}
```

### 8.3 TestContainers BOM trong pom.xml

Thêm BOM để quản lý version TestContainers (nếu chưa có):

```xml
<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>org.testcontainers</groupId>
            <artifactId>testcontainers-bom</artifactId>
            <version>1.20.4</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
    </dependencies>
</dependencyManagement>
```

> **Yêu cầu:** Docker Desktop phải đang chạy trên máy để TestContainers hoạt động. Nếu chạy trên CI/CD (GitHub Actions), dùng `services` để khởi động Docker.

---

## 9. Dockerfile Multi-Stage

Đóng gói Spring Boot app thành Docker image nhỏ gọn, an toàn.

```dockerfile
# ========================================
# Stage 1: Build — dùng JDK để compile
# ========================================
FROM eclipse-temurin:21-jdk AS builder
WORKDIR /app

# Copy Maven wrapper và pom.xml trước (để cache dependencies)
COPY pom.xml .
COPY .mvn .mvn
COPY mvnw .
RUN chmod +x mvnw

# Download dependencies trước — layer này được cache
# Chỉ rebuild khi pom.xml thay đổi
RUN ./mvnw dependency:go-offline -B

# Copy source code và build
COPY src ./src
RUN ./mvnw package -DskipTests -B

# ========================================
# Stage 2: Run — chỉ dùng JRE (nhẹ hơn JDK)
# ========================================
FROM eclipse-temurin:21-jre
WORKDIR /app

# Copy jar từ stage build
COPY --from=builder /app/target/*.jar app.jar

# Tạo non-root user (bảo mật)
RUN addgroup --system appgroup && \
    adduser --system appuser --ingroup appgroup
USER appuser

# Expose port (documentation, không tự mở port)
EXPOSE 8080

# Health check
HEALTHCHECK --interval=30s --timeout=10s --retries=3 \
    CMD curl -f http://localhost:8080/actuator/health || exit 1

# Start app
ENTRYPOINT ["java", \
    "-XX:+UseContainerSupport", \
    "-XX:MaxRAMPercentage=75.0", \
    "-jar", "app.jar"]
```

### 9.1 Tại sao Multi-Stage?

```
Single-stage (JDK):     ~600 MB  — chứa compiler, build tools, source code
Multi-stage (JRE only): ~300 MB  — chỉ chứa runtime + jar file

↓ Image nhỏ hơn = deploy nhanh hơn, ít surface attack hơn
```

### 9.2 Build và chạy

```bash
# Build image
docker build -t ecommerce-api:latest .

# Chạy container
docker run -p 8080:8080 \
    -e SPRING_PROFILES_ACTIVE=prod \
    -e SPRING_DATASOURCE_URL=jdbc:mysql://host.docker.internal:3306/ecommerce \
    ecommerce-api:latest

# Kiểm tra size
docker images ecommerce-api
# REPOSITORY       TAG       SIZE
# ecommerce-api    latest    ~300MB
```

### 9.3 .dockerignore

Tạo file `.dockerignore` để không copy files không cần thiết vào Docker context:

```
# .dockerignore
target/
.git/
.gitignore
.idea/
*.iml
.vscode/
*.md
docker-compose*.yml
.env
.env.*
```

---

## 10. Docker Compose — Full Stack

Chạy toàn bộ hệ thống (App + MySQL + Redis) với 1 lệnh duy nhất.

```yaml
version: "3.8"

services:
  # ── Spring Boot Application ──
  app:
    build: .
    container_name: ecommerce-app
    ports:
      - "8080:8080"
    environment:
      - SPRING_PROFILES_ACTIVE=prod
      - SPRING_DATASOURCE_URL=jdbc:mysql://mysql:3306/ecommerce
      - SPRING_DATASOURCE_USERNAME=root
      - SPRING_DATASOURCE_PASSWORD=secret
      - SPRING_DATA_REDIS_HOST=redis
      - SPRING_DATA_REDIS_PORT=6379
      - SPRING_JPA_HIBERNATE_DDL_AUTO=validate
      - SPRING_FLYWAY_ENABLED=true
      - APP_JWT_SECRET=${JWT_SECRET}
    depends_on:
      mysql:
        condition: service_healthy    # Đợi MySQL healthy mới start app
      redis:
        condition: service_healthy
    restart: unless-stopped
    networks:
      - ecommerce-net

  # ── MySQL Database ──
  mysql:
    image: mysql:8.0
    container_name: ecommerce-mysql
    environment:
      MYSQL_ROOT_PASSWORD: secret
      MYSQL_DATABASE: ecommerce
    ports:
      - "3306:3306"
    volumes:
      - mysql_data:/var/lib/mysql
    healthcheck:
      test: ["CMD", "mysqladmin", "ping", "-h", "localhost"]
      interval: 10s
      timeout: 5s
      retries: 5
      start_period: 30s
    restart: unless-stopped
    networks:
      - ecommerce-net

  # ── Redis Cache ──
  redis:
    image: redis:7-alpine
    container_name: ecommerce-redis
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
    restart: unless-stopped
    networks:
      - ecommerce-net

# ── Persistent Volumes ──
volumes:
  mysql_data:
  redis_data:

# ── Network ──
networks:
  ecommerce-net:
    driver: bridge
```

### 10.1 Các lệnh quản lý

```bash
# Build và chạy toàn bộ stack
docker compose up -d --build

# Xem logs
docker compose logs -f app

# Chỉ start infrastructure (dev mode — chạy app trên IDE)
docker compose up -d mysql redis

# Dừng tất cả
docker compose down

# Dừng và xóa data (CẢNH BÁO: mất hết data)
docker compose down -v
```

### 10.2 Lưu ý về Networking

Trong Docker Compose, các service giao tiếp với nhau qua **tên service** (không phải localhost):

```
App truy cập MySQL:  mysql:3306       (KHÔNG phải localhost:3306)
App truy cập Redis:  redis:6379       (KHÔNG phải localhost:6379)
Browser truy cập App: localhost:8080  (vì port được map ra host)
```

---

## 11. Production Checklist

Trước khi deploy, kiểm tra các mục sau:

### 11.1 Security

| # | Item | Mô tả | Status |
|---|------|-------|--------|
| 1 | CORS configured | Chỉ cho phép domain frontend | ☐ |
| 2 | CSRF protection | Bật nếu có form-based auth | ☐ |
| 3 | Rate limiting | Chống brute force login, spam API | ☐ |
| 4 | Input validation | `@Valid` trên tất cả request DTOs | ☐ |
| 5 | SQL Injection | Dùng parameterized queries (JPA default) | ☐ |
| 6 | JWT secret | Dùng env variable, >= 256-bit key | ☐ |
| 7 | HTTPS only | Redirect HTTP → HTTPS | ☐ |
| 8 | Password hashing | BCrypt với strength >= 10 | ☐ |

### 11.2 Database

| # | Item | Mô tả | Status |
|---|------|-------|--------|
| 1 | Connection pool sized | HikariCP max-pool-size phù hợp | ☐ |
| 2 | Flyway migrations | Mỗi schema change = migration file | ☐ |
| 3 | ddl-auto = validate | KHÔNG dùng update/create trên production | ☐ |
| 4 | open-in-view = false | Tắt OSIV để tránh lazy loading issues | ☐ |
| 5 | Index tối ưu | Các column thường query có index | ☐ |
| 6 | Backup strategy | Scheduled backup + tested restore | ☐ |

### 11.3 Performance

| # | Item | Mô tả | Status |
|---|------|-------|--------|
| 1 | Redis caching | Cache hot data (products, categories) | ☐ |
| 2 | Query optimization | Không có N+1, dùng JOIN FETCH | ☐ |
| 3 | Pagination | Tất cả list endpoint có phân trang | ☐ |
| 4 | Gzip compression | Bật response compression | ☐ |
| 5 | Connection timeout | Config phù hợp cho mỗi service | ☐ |

### 11.4 Monitoring & Logging

| # | Item | Mô tả | Status |
|---|------|-------|--------|
| 1 | Health endpoint | `/actuator/health` return UP | ☐ |
| 2 | Structured logging | JSON log format cho production | ☐ |
| 3 | Log level | INFO cho prod, DEBUG cho dev | ☐ |
| 4 | Error tracking | Sentry, ELK, hoặc tương đương | ☐ |
| 5 | Metrics | Prometheus + Grafana (optional) | ☐ |

### 11.5 Docker

| # | Item | Mô tả | Status |
|---|------|-------|--------|
| 1 | Non-root user | Chạy app bằng user không phải root | ☐ |
| 2 | Health checks | Container tự báo cáo trạng thái | ☐ |
| 3 | .dockerignore | Không copy file thừa vào image | ☐ |
| 4 | Multi-stage build | Image nhẹ, chỉ chứa JRE | ☐ |
| 5 | Resource limits | Set CPU/memory limits | ☐ |
| 6 | Restart policy | `unless-stopped` hoặc `always` | ☐ |

---

## 12. Tổng Kết Khóa Học

### 12.1 Những gì bạn đã học trong 7 ngày

```
Day 1: Foundation        → Spring Boot project + Hello World API
Day 2: IoC/DI + JPA      → Dependency Injection + Entity design
Day 3: JPA Mastery        → Flyway + Specification + Query optimization
Day 4: REST API + CRUD    → Full ProductController + Validation + Error handling
Day 5: Security + JWT     → Authentication + Authorization + Role-based access
Day 6: Cart + Order       → Business logic phức tạp + Transaction management
Day 7: Testing + Docker   → Test pyramid + Containerization + Production readiness
```

### 12.2 Kỹ năng đạt được

| Kỹ năng | Chi tiết |
|---------|----------|
| **Spring Boot Core** | Auto-config, profiles, properties binding |
| **JPA/Hibernate** | Entity mapping, relationships, queries, N+1 fix |
| **REST API Design** | CRUD, validation, error handling, pagination |
| **Security** | JWT auth, role-based access, password hashing |
| **Testing** | Unit test (Mockito), Integration test (TestContainers), MockMvc |
| **Docker** | Multi-stage build, Docker Compose, health checks |
| **Best Practices** | DDD layers, DTO pattern, connection pooling, caching |

### 12.3 Tiếp theo — Học gì sau khóa này?

| Thứ tự | Chủ đề | Lý do |
|--------|--------|-------|
| 1 | **OAuth2 / OpenID Connect** | Authentication chuẩn công nghiệp (Google, GitHub login) |
| 2 | **CI/CD Pipeline** | GitHub Actions / GitLab CI — tự động test + deploy |
| 3 | **Elasticsearch** | Full-text search cho products (nhanh hơn LIKE %%) |
| 4 | **Monitoring Stack** | Prometheus + Grafana + ELK — quan sát hệ thống |
| 5 | **Message Queue** | RabbitMQ / Kafka — xử lý async (email, notification) |
| 6 | **Microservices** | Tách thành nhiều service nhỏ với Spring Cloud |
| 7 | **Kubernetes** | Orchestration — deploy nhiều container trên cloud |

### 12.4 Tài liệu tham khảo

- [Spring Boot Reference Documentation](https://docs.spring.io/spring-boot/docs/current/reference/htmlsingle/)
- [Baeldung](https://www.baeldung.com/) — Tutorials Spring chi tiết nhất
- [Spring Data JPA Reference](https://docs.spring.io/spring-data/jpa/docs/current/reference/html/)
- [TestContainers Documentation](https://testcontainers.com/)
- [Docker Documentation](https://docs.docker.com/)

---

> **Xin chúc mừng!** Bạn đã hoàn thành khóa học **Java Spring Boot E-Commerce Speed Run 7 Days**. Từ đây, hãy xây dựng project của riêng bạn — đó là cách học tốt nhất.
