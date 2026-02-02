# Day 26: Unit Testing

## Mục tiêu học tập
- Hiểu Testing Pyramid
- Viết Unit Tests với JUnit 5 & Mockito
- Test Service Layer
- Test Repository với @DataJpaTest

---

## 1. Testing Pyramid

```
┌─────────────────────────────────────────────────────────────────┐
│                    TESTING PYRAMID                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│                         /\                                       │
│                        /  \                                      │
│                       /    \                                     │
│                      / E2E  \     ← Few, Slow, Expensive        │
│                     /  Tests \       UI/Browser Tests            │
│                    /──────────\                                  │
│                   /            \                                 │
│                  / Integration  \   ← Some, Medium Speed        │
│                 /    Tests       \    API Tests, DB Tests        │
│                /──────────────────\                              │
│               /                    \                             │
│              /     Unit Tests       \  ← Many, Fast, Cheap       │
│             /                        \   Service, Logic Tests    │
│            /──────────────────────────\                          │
│                                                                  │
│  Best Practices:                                                │
│  • 70% Unit Tests                                               │
│  • 20% Integration Tests                                        │
│  • 10% E2E Tests                                                │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## 2. Test Dependencies

### 2.1 pom.xml

```xml
<dependencies>
    <!-- Spring Boot Starter Test -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-test</artifactId>
        <scope>test</scope>
    </dependency>

    <!-- JUnit 5 -->
    <dependency>
        <groupId>org.junit.jupiter</groupId>
        <artifactId>junit-jupiter</artifactId>
        <scope>test</scope>
    </dependency>

    <!-- Mockito -->
    <dependency>
        <groupId>org.mockito</groupId>
        <artifactId>mockito-core</artifactId>
        <scope>test</scope>
    </dependency>

    <dependency>
        <groupId>org.mockito</groupId>
        <artifactId>mockito-junit-jupiter</artifactId>
        <scope>test</scope>
    </dependency>

    <!-- AssertJ for fluent assertions -->
    <dependency>
        <groupId>org.assertj</groupId>
        <artifactId>assertj-core</artifactId>
        <scope>test</scope>
    </dependency>

    <!-- H2 Database for tests -->
    <dependency>
        <groupId>com.h2database</groupId>
        <artifactId>h2</artifactId>
        <scope>test</scope>
    </dependency>

    <!-- TestContainers for MySQL tests -->
    <dependency>
        <groupId>org.testcontainers</groupId>
        <artifactId>mysql</artifactId>
        <scope>test</scope>
    </dependency>
</dependencies>
```

---

## 3. JUnit 5 Basics

### 3.1 Test Class Structure

```java
package com.ecommerce.service.impl;

import org.junit.jupiter.api.*;
import org.junit.jupiter.api.extension.ExtendWith;
import org.mockito.junit.jupiter.MockitoExtension;

import static org.assertj.core.api.Assertions.*;
import static org.junit.jupiter.api.Assertions.*;

@ExtendWith(MockitoExtension.class)
@DisplayName("ProductService Tests")
class ProductServiceImplTest {

    @BeforeAll
    static void setUpAll() {
        // Run once before all tests
        System.out.println("Starting ProductService tests...");
    }

    @BeforeEach
    void setUp() {
        // Run before each test
    }

    @AfterEach
    void tearDown() {
        // Run after each test
    }

    @AfterAll
    static void tearDownAll() {
        // Run once after all tests
    }

    @Test
    @DisplayName("Should create product successfully")
    void createProduct_WithValidData_ShouldSucceed() {
        // Arrange
        // Act
        // Assert
    }

    @Test
    @DisplayName("Should throw exception when product not found")
    void getProduct_WhenNotFound_ShouldThrowException() {
        // Test implementation
    }

    @Nested
    @DisplayName("Product Creation Tests")
    class ProductCreationTests {

        @Test
        void shouldCreateProductWithAllFields() {
            // Nested test
        }

        @Test
        void shouldCreateProductWithMinimalFields() {
            // Nested test
        }
    }
}
```

### 3.2 Common Annotations

```java
// Test method annotations
@Test                           // Marks method as test
@DisplayName("Description")     // Custom test name
@Disabled("Reason")            // Skip test
@RepeatedTest(5)               // Repeat test N times
@Timeout(5)                    // Fail if exceeds timeout (seconds)

// Lifecycle annotations
@BeforeAll                     // Before all tests (static)
@BeforeEach                    // Before each test
@AfterEach                     // After each test
@AfterAll                      // After all tests (static)

// Conditional execution
@EnabledOnOs(OS.LINUX)
@DisabledOnOs(OS.WINDOWS)
@EnabledIfEnvironmentVariable(named = "ENV", matches = "dev")
@EnabledIf("customCondition")
```

---

## 4. Testing Service Layer

### 4.1 ProductServiceImpl Test

```java
package com.ecommerce.service.impl;

import com.ecommerce.domain.entity.Category;
import com.ecommerce.domain.entity.Product;
import com.ecommerce.dto.request.CreateProductRequest;
import com.ecommerce.dto.request.UpdateProductRequest;
import com.ecommerce.dto.response.ProductDto;
import com.ecommerce.exception.ResourceNotFoundException;
import com.ecommerce.mapper.ProductMapper;
import com.ecommerce.repository.CategoryRepository;
import com.ecommerce.repository.ProductRepository;
import org.junit.jupiter.api.*;
import org.junit.jupiter.api.extension.ExtendWith;
import org.mockito.*;
import org.mockito.junit.jupiter.MockitoExtension;

import java.math.BigDecimal;
import java.util.Optional;

import static org.assertj.core.api.Assertions.*;
import static org.mockito.ArgumentMatchers.*;
import static org.mockito.BDDMockito.*;

@ExtendWith(MockitoExtension.class)
@DisplayName("ProductServiceImpl Tests")
class ProductServiceImplTest {

    @Mock
    private ProductRepository productRepository;

    @Mock
    private CategoryRepository categoryRepository;

    @Mock
    private ProductMapper productMapper;

    @InjectMocks
    private ProductServiceImpl productService;

    @Captor
    private ArgumentCaptor<Product> productCaptor;

    private Product testProduct;
    private Category testCategory;
    private ProductDto testProductDto;

    @BeforeEach
    void setUp() {
        testCategory = Category.builder()
                .id(1L)
                .name("Electronics")
                .slug("electronics")
                .build();

        testProduct = Product.builder()
                .id(1L)
                .name("Test Product")
                .slug("test-product")
                .sku("SKU-001")
                .description("Test description")
                .price(new BigDecimal("99.99"))
                .stockQuantity(100)
                .category(testCategory)
                .active(true)
                .build();

        testProductDto = ProductDto.builder()
                .id(1L)
                .name("Test Product")
                .slug("test-product")
                .price(new BigDecimal("99.99"))
                .build();
    }

    // ========== getProductById Tests ==========

    @Nested
    @DisplayName("getProductById")
    class GetProductByIdTests {

        @Test
        @DisplayName("Should return product when found")
        void getProductById_WhenExists_ShouldReturnProduct() {
            // Given
            given(productRepository.findByIdAndActiveTrue(1L))
                    .willReturn(Optional.of(testProduct));
            given(productMapper.toDto(testProduct))
                    .willReturn(testProductDto);

            // When
            ProductDto result = productService.getProductById(1L);

            // Then
            assertThat(result).isNotNull();
            assertThat(result.getId()).isEqualTo(1L);
            assertThat(result.getName()).isEqualTo("Test Product");

            then(productRepository).should(times(1)).findByIdAndActiveTrue(1L);
        }

        @Test
        @DisplayName("Should throw exception when product not found")
        void getProductById_WhenNotFound_ShouldThrowException() {
            // Given
            given(productRepository.findByIdAndActiveTrue(999L))
                    .willReturn(Optional.empty());

            // When & Then
            assertThatThrownBy(() -> productService.getProductById(999L))
                    .isInstanceOf(ResourceNotFoundException.class)
                    .hasMessageContaining("Product")
                    .hasMessageContaining("999");

            then(productMapper).shouldHaveNoInteractions();
        }
    }

    // ========== createProduct Tests ==========

    @Nested
    @DisplayName("createProduct")
    class CreateProductTests {

        private CreateProductRequest createRequest;

        @BeforeEach
        void setUp() {
            createRequest = new CreateProductRequest();
            createRequest.setName("New Product");
            createRequest.setSku("SKU-NEW");
            createRequest.setDescription("New description");
            createRequest.setPrice(new BigDecimal("149.99"));
            createRequest.setStockQuantity(50);
            createRequest.setCategoryId(1L);
        }

        @Test
        @DisplayName("Should create product with valid data")
        void createProduct_WithValidData_ShouldSucceed() {
            // Given
            given(categoryRepository.findById(1L))
                    .willReturn(Optional.of(testCategory));
            given(productRepository.existsBySku(anyString()))
                    .willReturn(false);
            given(productRepository.save(any(Product.class)))
                    .willAnswer(invocation -> {
                        Product p = invocation.getArgument(0);
                        p.setId(2L);
                        return p;
                    });
            given(productMapper.toDto(any(Product.class)))
                    .willReturn(testProductDto);

            // When
            ProductDto result = productService.createProduct(createRequest);

            // Then
            assertThat(result).isNotNull();

            // Verify save was called with correct data
            then(productRepository).should().save(productCaptor.capture());
            Product savedProduct = productCaptor.getValue();

            assertThat(savedProduct.getName()).isEqualTo("New Product");
            assertThat(savedProduct.getSku()).isEqualTo("SKU-NEW");
            assertThat(savedProduct.getCategory()).isEqualTo(testCategory);
        }

        @Test
        @DisplayName("Should throw exception when category not found")
        void createProduct_WhenCategoryNotFound_ShouldThrowException() {
            // Given
            given(categoryRepository.findById(999L))
                    .willReturn(Optional.empty());

            createRequest.setCategoryId(999L);

            // When & Then
            assertThatThrownBy(() -> productService.createProduct(createRequest))
                    .isInstanceOf(ResourceNotFoundException.class)
                    .hasMessageContaining("Category");

            then(productRepository).shouldHaveNoInteractions();
        }

        @Test
        @DisplayName("Should throw exception when SKU already exists")
        void createProduct_WhenSkuExists_ShouldThrowException() {
            // Given
            given(categoryRepository.findById(1L))
                    .willReturn(Optional.of(testCategory));
            given(productRepository.existsBySku("SKU-NEW"))
                    .willReturn(true);

            // When & Then
            assertThatThrownBy(() -> productService.createProduct(createRequest))
                    .isInstanceOf(IllegalArgumentException.class)
                    .hasMessageContaining("SKU already exists");
        }

        @Test
        @DisplayName("Should generate slug from name")
        void createProduct_ShouldGenerateSlug() {
            // Given
            createRequest.setName("Amazing Product 123!");

            given(categoryRepository.findById(1L))
                    .willReturn(Optional.of(testCategory));
            given(productRepository.existsBySku(anyString()))
                    .willReturn(false);
            given(productRepository.save(any(Product.class)))
                    .willAnswer(invocation -> invocation.getArgument(0));
            given(productMapper.toDto(any(Product.class)))
                    .willReturn(testProductDto);

            // When
            productService.createProduct(createRequest);

            // Then
            then(productRepository).should().save(productCaptor.capture());
            Product savedProduct = productCaptor.getValue();

            assertThat(savedProduct.getSlug())
                    .isEqualTo("amazing-product-123")
                    .doesNotContain("!")
                    .isLowerCase();
        }
    }

    // ========== updateProduct Tests ==========

    @Nested
    @DisplayName("updateProduct")
    class UpdateProductTests {

        @Test
        @DisplayName("Should update product price")
        void updateProduct_Price_ShouldUpdate() {
            // Given
            UpdateProductRequest updateRequest = new UpdateProductRequest();
            updateRequest.setPrice(new BigDecimal("199.99"));

            given(productRepository.findById(1L))
                    .willReturn(Optional.of(testProduct));
            given(productRepository.save(any(Product.class)))
                    .willAnswer(invocation -> invocation.getArgument(0));
            given(productMapper.toDto(any(Product.class)))
                    .willReturn(testProductDto);

            // When
            productService.updateProduct(1L, updateRequest);

            // Then
            then(productRepository).should().save(productCaptor.capture());
            Product updatedProduct = productCaptor.getValue();

            assertThat(updatedProduct.getPrice())
                    .isEqualByComparingTo(new BigDecimal("199.99"));
        }

        @Test
        @DisplayName("Should not update null fields")
        void updateProduct_WithNullFields_ShouldNotChange() {
            // Given
            BigDecimal originalPrice = testProduct.getPrice();
            String originalName = testProduct.getName();

            UpdateProductRequest updateRequest = new UpdateProductRequest();
            // All fields are null

            given(productRepository.findById(1L))
                    .willReturn(Optional.of(testProduct));
            given(productRepository.save(any(Product.class)))
                    .willAnswer(invocation -> invocation.getArgument(0));
            given(productMapper.toDto(any(Product.class)))
                    .willReturn(testProductDto);

            // When
            productService.updateProduct(1L, updateRequest);

            // Then
            then(productRepository).should().save(productCaptor.capture());
            Product updatedProduct = productCaptor.getValue();

            assertThat(updatedProduct.getPrice()).isEqualByComparingTo(originalPrice);
            assertThat(updatedProduct.getName()).isEqualTo(originalName);
        }
    }

    // ========== deleteProduct Tests ==========

    @Nested
    @DisplayName("deleteProduct")
    class DeleteProductTests {

        @Test
        @DisplayName("Should soft delete product")
        void deleteProduct_ShouldSoftDelete() {
            // Given
            given(productRepository.findById(1L))
                    .willReturn(Optional.of(testProduct));
            given(productRepository.save(any(Product.class)))
                    .willAnswer(invocation -> invocation.getArgument(0));

            // When
            productService.deleteProduct(1L);

            // Then
            then(productRepository).should().save(productCaptor.capture());
            Product deletedProduct = productCaptor.getValue();

            assertThat(deletedProduct.isActive()).isFalse();
            assertThat(deletedProduct.isDeleted()).isTrue();
        }
    }
}
```

---

## 5. Testing Repository Layer

### 5.1 ProductRepository Test

```java
package com.ecommerce.repository;

import com.ecommerce.domain.entity.Category;
import com.ecommerce.domain.entity.Product;
import org.junit.jupiter.api.*;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.autoconfigure.orm.jpa.DataJpaTest;
import org.springframework.boot.test.autoconfigure.orm.jpa.TestEntityManager;
import org.springframework.data.domain.Page;
import org.springframework.data.domain.PageRequest;
import org.springframework.test.context.ActiveProfiles;

import java.math.BigDecimal;
import java.util.List;
import java.util.Optional;

import static org.assertj.core.api.Assertions.*;

@DataJpaTest
@ActiveProfiles("test")
@DisplayName("ProductRepository Tests")
class ProductRepositoryTest {

    @Autowired
    private TestEntityManager entityManager;

    @Autowired
    private ProductRepository productRepository;

    private Category testCategory;

    @BeforeEach
    void setUp() {
        testCategory = Category.builder()
                .name("Electronics")
                .slug("electronics")
                .build();
        entityManager.persist(testCategory);
        entityManager.flush();
    }

    @Nested
    @DisplayName("findByIdAndActiveTrue")
    class FindByIdAndActiveTrueTests {

        @Test
        @DisplayName("Should find active product")
        void shouldFindActiveProduct() {
            // Given
            Product product = createProduct("Active Product", true);
            entityManager.persist(product);
            entityManager.flush();

            // When
            Optional<Product> found = productRepository.findByIdAndActiveTrue(product.getId());

            // Then
            assertThat(found).isPresent();
            assertThat(found.get().getName()).isEqualTo("Active Product");
        }

        @Test
        @DisplayName("Should not find inactive product")
        void shouldNotFindInactiveProduct() {
            // Given
            Product product = createProduct("Inactive Product", false);
            entityManager.persist(product);
            entityManager.flush();

            // When
            Optional<Product> found = productRepository.findByIdAndActiveTrue(product.getId());

            // Then
            assertThat(found).isEmpty();
        }
    }

    @Nested
    @DisplayName("searchByKeyword")
    class SearchByKeywordTests {

        @BeforeEach
        void setUpProducts() {
            entityManager.persist(createProduct("iPhone 15 Pro", true));
            entityManager.persist(createProduct("Samsung Galaxy S24", true));
            entityManager.persist(createProduct("iPhone 14", true));
            entityManager.persist(createProduct("Pixel 8", true));
            entityManager.flush();
        }

        @Test
        @DisplayName("Should find products by name keyword")
        void shouldFindByNameKeyword() {
            // When
            Page<Product> results = productRepository.searchByKeyword(
                    "iPhone",
                    PageRequest.of(0, 10)
            );

            // Then
            assertThat(results.getContent())
                    .hasSize(2)
                    .extracting(Product::getName)
                    .containsExactlyInAnyOrder("iPhone 15 Pro", "iPhone 14");
        }

        @Test
        @DisplayName("Should be case insensitive")
        void shouldBeCaseInsensitive() {
            // When
            Page<Product> results = productRepository.searchByKeyword(
                    "IPHONE",
                    PageRequest.of(0, 10)
            );

            // Then
            assertThat(results.getContent()).hasSize(2);
        }

        @Test
        @DisplayName("Should return empty for no matches")
        void shouldReturnEmptyForNoMatches() {
            // When
            Page<Product> results = productRepository.searchByKeyword(
                    "Xiaomi",
                    PageRequest.of(0, 10)
            );

            // Then
            assertThat(results.getContent()).isEmpty();
        }
    }

    @Nested
    @DisplayName("findByPriceBetween")
    class FindByPriceBetweenTests {

        @BeforeEach
        void setUpProducts() {
            entityManager.persist(createProductWithPrice("Cheap", new BigDecimal("50")));
            entityManager.persist(createProductWithPrice("Medium", new BigDecimal("100")));
            entityManager.persist(createProductWithPrice("Expensive", new BigDecimal("200")));
            entityManager.flush();
        }

        @Test
        @DisplayName("Should find products in price range")
        void shouldFindInPriceRange() {
            // When
            Page<Product> results = productRepository.findByActiveAndPriceBetween(
                    true,
                    new BigDecimal("50"),
                    new BigDecimal("100"),
                    PageRequest.of(0, 10)
            );

            // Then
            assertThat(results.getContent())
                    .hasSize(2)
                    .extracting(Product::getName)
                    .containsExactlyInAnyOrder("Cheap", "Medium");
        }
    }

    @Nested
    @DisplayName("existsBySku")
    class ExistsBySkuTests {

        @Test
        @DisplayName("Should return true for existing SKU")
        void shouldReturnTrueForExistingSku() {
            // Given
            Product product = createProduct("Test", true);
            product.setSku("UNIQUE-SKU");
            entityManager.persist(product);
            entityManager.flush();

            // When
            boolean exists = productRepository.existsBySku("UNIQUE-SKU");

            // Then
            assertThat(exists).isTrue();
        }

        @Test
        @DisplayName("Should return false for non-existing SKU")
        void shouldReturnFalseForNonExistingSku() {
            // When
            boolean exists = productRepository.existsBySku("NON-EXISTING");

            // Then
            assertThat(exists).isFalse();
        }
    }

    // ========== Helper Methods ==========

    private Product createProduct(String name, boolean active) {
        return Product.builder()
                .name(name)
                .slug(name.toLowerCase().replace(" ", "-"))
                .sku("SKU-" + System.currentTimeMillis())
                .description("Description for " + name)
                .price(new BigDecimal("99.99"))
                .stockQuantity(100)
                .category(testCategory)
                .active(active)
                .build();
    }

    private Product createProductWithPrice(String name, BigDecimal price) {
        Product product = createProduct(name, true);
        product.setPrice(price);
        return product;
    }
}
```

---

## 6. Testing with Mockito

### 6.1 Mockito Annotations

```java
// Basic annotations
@Mock                  // Create mock object
@InjectMocks          // Inject mocks into tested class
@Spy                  // Partial mock (real method calls by default)
@Captor               // Capture method arguments

// Argument matchers
any()                 // Any value
anyLong()            // Any long value
anyString()          // Any string
eq(value)            // Exact value
argThat(predicate)   // Custom matcher

// Stubbing
when(mock.method()).thenReturn(value)
when(mock.method()).thenThrow(exception)
when(mock.method()).thenAnswer(invocation -> { ... })

// BDD style (given/when/then)
given(mock.method()).willReturn(value)
then(mock).should().method()
then(mock).shouldHaveNoInteractions()

// Verification
verify(mock).method()
verify(mock, times(2)).method()
verify(mock, never()).method()
verify(mock, atLeast(1)).method()
verifyNoMoreInteractions(mock)
```

### 6.2 Advanced Mockito Examples

```java
@Test
void testWithArgumentCaptor() {
    // Capture arguments
    ArgumentCaptor<Product> captor = ArgumentCaptor.forClass(Product.class);

    productService.createProduct(request);

    verify(productRepository).save(captor.capture());
    Product captured = captor.getValue();

    assertThat(captured.getName()).isEqualTo("Expected Name");
}

@Test
void testWithAnswer() {
    // Dynamic return based on input
    when(productRepository.save(any(Product.class)))
            .thenAnswer(invocation -> {
                Product product = invocation.getArgument(0);
                product.setId(1L);
                return product;
            });
}

@Test
void testWithSpy() {
    // Partial mock
    List<String> spyList = spy(new ArrayList<>());

    spyList.add("one");
    spyList.add("two");

    verify(spyList).add("one");
    assertThat(spyList).hasSize(2);

    // Override specific method
    doReturn(100).when(spyList).size();
    assertThat(spyList.size()).isEqualTo(100);
}

@Test
void testVoidMethods() {
    // Stub void method
    doNothing().when(emailService).sendEmail(anyString(), anyString());

    // Verify void method was called
    productService.createProductAndNotify(request);

    verify(emailService).sendEmail(eq("admin@example.com"), anyString());
}

@Test
void testExceptionHandling() {
    // Stub to throw exception
    when(productRepository.findById(anyLong()))
            .thenThrow(new RuntimeException("Database error"));

    assertThatThrownBy(() -> productService.getProductById(1L))
            .isInstanceOf(ServiceException.class)
            .hasCauseInstanceOf(RuntimeException.class);
}
```

---

## 7. AssertJ Assertions

```java
// Basic assertions
assertThat(actual).isEqualTo(expected);
assertThat(actual).isNotNull();
assertThat(actual).isNull();

// String assertions
assertThat(string).isNotEmpty();
assertThat(string).startsWith("prefix");
assertThat(string).contains("substring");
assertThat(string).matches("[A-Z]{3}-\\d{3}");

// Number assertions
assertThat(number).isPositive();
assertThat(number).isBetween(1, 10);
assertThat(number).isGreaterThan(5);
assertThat(price).isEqualByComparingTo(new BigDecimal("99.99"));

// Collection assertions
assertThat(list).hasSize(3);
assertThat(list).contains("item1", "item2");
assertThat(list).containsExactly("a", "b", "c");
assertThat(list).extracting(User::getName)
                .containsExactlyInAnyOrder("John", "Jane");

// Object assertions
assertThat(product)
    .hasFieldOrPropertyWithValue("name", "iPhone")
    .hasFieldOrPropertyWithValue("price", new BigDecimal("999"));

// Exception assertions
assertThatThrownBy(() -> service.method())
    .isInstanceOf(CustomException.class)
    .hasMessage("Expected message")
    .hasMessageContaining("partial");

assertThatCode(() -> service.safeMethod())
    .doesNotThrowAnyException();

// Optional assertions
assertThat(optional).isPresent();
assertThat(optional).isEmpty();
assertThat(optional).hasValue(expectedValue);

// Soft assertions (collect all failures)
SoftAssertions softly = new SoftAssertions();
softly.assertThat(product.getName()).isEqualTo("Expected");
softly.assertThat(product.getPrice()).isPositive();
softly.assertAll();
```

---

## 8. Test Configuration

### 8.1 application-test.yml

```yaml
spring:
  datasource:
    url: jdbc:h2:mem:testdb;DB_CLOSE_DELAY=-1;MODE=MySQL
    username: sa
    password:
    driver-class-name: org.h2.Driver

  jpa:
    hibernate:
      ddl-auto: create-drop
    show-sql: true
    properties:
      hibernate:
        format_sql: true

  flyway:
    enabled: false

logging:
  level:
    org.hibernate.SQL: DEBUG
    org.hibernate.type.descriptor.sql: TRACE
```

### 8.2 Test Base Class

```java
package com.ecommerce;

import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.test.context.ActiveProfiles;
import org.springframework.transaction.annotation.Transactional;

@SpringBootTest
@ActiveProfiles("test")
@Transactional
public abstract class BaseIntegrationTest {
    // Common test setup
}
```

---

## 9. Parameterized Tests

```java
@ParameterizedTest
@ValueSource(strings = {"iPhone", "Samsung", "Pixel"})
@DisplayName("Should find products by brand")
void shouldFindByBrand(String brand) {
    // Given
    Product product = createProduct(brand + " Phone");
    productRepository.save(product);

    // When
    Page<Product> results = productRepository.searchByKeyword(brand, PageRequest.of(0, 10));

    // Then
    assertThat(results.getContent()).isNotEmpty();
}

@ParameterizedTest
@CsvSource({
    "iPhone, 999.99, true",
    "Samsung, 899.99, true",
    "Unknown, 0, false"
})
@DisplayName("Should validate product prices")
void shouldValidatePrices(String name, BigDecimal price, boolean expected) {
    Product product = createProduct(name);
    product.setPrice(price);

    boolean isValid = productService.validatePrice(product);

    assertThat(isValid).isEqualTo(expected);
}

@ParameterizedTest
@MethodSource("provideProductNames")
@DisplayName("Should generate correct slugs")
void shouldGenerateSlugs(String name, String expectedSlug) {
    String slug = SlugUtils.generateSlug(name);
    assertThat(slug).isEqualTo(expectedSlug);
}

static Stream<Arguments> provideProductNames() {
    return Stream.of(
        Arguments.of("iPhone 15 Pro", "iphone-15-pro"),
        Arguments.of("Samsung Galaxy S24!", "samsung-galaxy-s24"),
        Arguments.of("Simple Name", "simple-name")
    );
}

@ParameterizedTest
@EnumSource(OrderStatus.class)
@DisplayName("Should handle all order statuses")
void shouldHandleAllStatuses(OrderStatus status) {
    // Test each enum value
}

@ParameterizedTest
@EnumSource(value = OrderStatus.class, names = {"PENDING", "CONFIRMED"})
@DisplayName("Should handle cancellable statuses")
void shouldHandleCancellableStatuses(OrderStatus status) {
    // Test specific enum values
}
```

---

## 10. Test Coverage

### 10.1 JaCoCo Configuration

```xml
<plugin>
    <groupId>org.jacoco</groupId>
    <artifactId>jacoco-maven-plugin</artifactId>
    <version>0.8.11</version>
    <executions>
        <execution>
            <goals>
                <goal>prepare-agent</goal>
            </goals>
        </execution>
        <execution>
            <id>report</id>
            <phase>test</phase>
            <goals>
                <goal>report</goal>
            </goals>
        </execution>
        <execution>
            <id>check</id>
            <goals>
                <goal>check</goal>
            </goals>
            <configuration>
                <rules>
                    <rule>
                        <element>BUNDLE</element>
                        <limits>
                            <limit>
                                <counter>LINE</counter>
                                <value>COVEREDRATIO</value>
                                <minimum>0.80</minimum>
                            </limit>
                        </limits>
                    </rule>
                </rules>
            </configuration>
        </execution>
    </executions>
</plugin>
```

### 10.2 Run Tests

```bash
# Run all tests
mvn test

# Run with coverage report
mvn test jacoco:report

# Run specific test class
mvn test -Dtest=ProductServiceImplTest

# Run specific test method
mvn test -Dtest=ProductServiceImplTest#createProduct_WithValidData_ShouldSucceed
```

---

## 11. Bài tập thực hành

### Bài tập 1: Service Tests
- [x] Test ProductService CRUD methods
- [x] Test exception scenarios
- [x] Use argument captors

### Bài tập 2: Repository Tests
- [x] Test custom queries
- [x] Test pagination
- [x] Test with test data

### Bài tập 3: Parameterized Tests
- [ ] Test slug generation with various inputs
- [ ] Test price validation
- [ ] Test enum handling

---

## 12. Checklist

- [ ] Setup test dependencies
- [ ] Configure test profile
- [ ] Write ProductService unit tests
- [ ] Write ProductRepository tests
- [ ] Use Mockito for mocking
- [ ] Use AssertJ for assertions
- [ ] Add parameterized tests
- [ ] Configure JaCoCo coverage
- [ ] Achieve 80% code coverage
- [ ] Run tests in CI/CD

---

## Điều hướng

- [← Day 25: Search & Filtering](../week-05/day-25-search-filtering.md)
- [Day 27: Integration Testing →](./day-27-integration-testing.md)
- [Về trang chính](../00-overview.md)
