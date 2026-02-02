# Day 27: Integration Testing

## Mục tiêu học tập
- Test REST APIs với @WebMvcTest và MockMvc
- Test full application với @SpringBootTest
- Sử dụng TestContainers cho database tests
- Test Security configurations

---

## 1. Integration Test Types

```
┌─────────────────────────────────────────────────────────────────┐
│                 INTEGRATION TEST TYPES                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │ @WebMvcTest - Controller Layer Only                      │   │
│  │ • Tests HTTP request/response                            │   │
│  │ • Mocks service layer                                    │   │
│  │ • Fast startup                                           │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                  │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │ @DataJpaTest - Repository Layer Only                     │   │
│  │ • Tests JPA queries                                      │   │
│  │ • Uses embedded/test database                            │   │
│  │ • Transaction rollback after each test                   │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                  │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │ @SpringBootTest - Full Application                       │   │
│  │ • Loads complete Spring context                          │   │
│  │ • Tests end-to-end flow                                  │   │
│  │ • Slower but more comprehensive                          │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                  │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │ @SpringBootTest + TestContainers                         │   │
│  │ • Real database in Docker                                │   │
│  │ • Production-like environment                            │   │
│  │ • Tests actual SQL queries                               │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## 2. Controller Integration Tests (@WebMvcTest)

### 2.1 ProductController Test

```java
package com.ecommerce.controller;

import com.ecommerce.dto.request.CreateProductRequest;
import com.ecommerce.dto.response.ProductDto;
import com.ecommerce.exception.ResourceNotFoundException;
import com.ecommerce.security.jwt.JwtTokenProvider;
import com.ecommerce.service.ProductService;
import com.fasterxml.jackson.databind.ObjectMapper;
import org.junit.jupiter.api.*;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.autoconfigure.web.servlet.WebMvcTest;
import org.springframework.boot.test.mock.mockito.MockBean;
import org.springframework.http.MediaType;
import org.springframework.security.test.context.support.WithMockUser;
import org.springframework.test.web.servlet.MockMvc;
import org.springframework.test.web.servlet.ResultActions;

import java.math.BigDecimal;
import java.util.List;

import static org.hamcrest.Matchers.*;
import static org.mockito.ArgumentMatchers.any;
import static org.mockito.ArgumentMatchers.eq;
import static org.mockito.BDDMockito.given;
import static org.mockito.Mockito.*;
import static org.springframework.security.test.web.servlet.request.SecurityMockMvcRequestPostProcessors.csrf;
import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.*;
import static org.springframework.test.web.servlet.result.MockMvcResultHandlers.print;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.*;

@WebMvcTest(ProductController.class)
@DisplayName("ProductController Integration Tests")
class ProductControllerTest {

    @Autowired
    private MockMvc mockMvc;

    @Autowired
    private ObjectMapper objectMapper;

    @MockBean
    private ProductService productService;

    @MockBean
    private JwtTokenProvider jwtTokenProvider;

    private ProductDto testProduct;

    @BeforeEach
    void setUp() {
        testProduct = ProductDto.builder()
                .id(1L)
                .name("Test Product")
                .slug("test-product")
                .price(new BigDecimal("99.99"))
                .stockQuantity(100)
                .active(true)
                .build();
    }

    // ========== GET /api/v1/products ==========

    @Nested
    @DisplayName("GET /api/v1/products")
    class GetAllProductsTests {

        @Test
        @DisplayName("Should return list of products")
        void getAllProducts_ShouldReturnProductList() throws Exception {
            // Given
            given(productService.getAllProducts(any()))
                    .willReturn(new org.springframework.data.domain.PageImpl<>(List.of(testProduct)));

            // When & Then
            mockMvc.perform(get("/api/v1/products")
                            .contentType(MediaType.APPLICATION_JSON))
                    .andDo(print())
                    .andExpect(status().isOk())
                    .andExpect(jsonPath("$.success", is(true)))
                    .andExpect(jsonPath("$.data.content", hasSize(1)))
                    .andExpect(jsonPath("$.data.content[0].name", is("Test Product")));
        }

        @Test
        @DisplayName("Should support pagination")
        void getAllProducts_WithPagination_ShouldReturnPagedResults() throws Exception {
            // When & Then
            mockMvc.perform(get("/api/v1/products")
                            .param("page", "0")
                            .param("size", "10")
                            .param("sort", "createdAt,desc"))
                    .andExpect(status().isOk());

            verify(productService).getAllProducts(argThat(pageable ->
                    pageable.getPageNumber() == 0 &&
                    pageable.getPageSize() == 10
            ));
        }
    }

    // ========== GET /api/v1/products/{id} ==========

    @Nested
    @DisplayName("GET /api/v1/products/{id}")
    class GetProductByIdTests {

        @Test
        @DisplayName("Should return product when exists")
        void getProductById_WhenExists_ShouldReturnProduct() throws Exception {
            // Given
            given(productService.getProductById(1L)).willReturn(testProduct);

            // When & Then
            mockMvc.perform(get("/api/v1/products/{id}", 1L))
                    .andExpect(status().isOk())
                    .andExpect(jsonPath("$.success", is(true)))
                    .andExpect(jsonPath("$.data.id", is(1)))
                    .andExpect(jsonPath("$.data.name", is("Test Product")))
                    .andExpect(jsonPath("$.data.price", is(99.99)));
        }

        @Test
        @DisplayName("Should return 404 when product not found")
        void getProductById_WhenNotFound_ShouldReturn404() throws Exception {
            // Given
            given(productService.getProductById(999L))
                    .willThrow(new ResourceNotFoundException("Product", "id", 999L));

            // When & Then
            mockMvc.perform(get("/api/v1/products/{id}", 999L))
                    .andExpect(status().isNotFound())
                    .andExpect(jsonPath("$.success", is(false)))
                    .andExpect(jsonPath("$.message", containsString("not found")));
        }
    }

    // ========== POST /api/v1/products ==========

    @Nested
    @DisplayName("POST /api/v1/products")
    class CreateProductTests {

        private CreateProductRequest createRequest;

        @BeforeEach
        void setUp() {
            createRequest = new CreateProductRequest();
            createRequest.setName("New Product");
            createRequest.setSku("SKU-001");
            createRequest.setDescription("Description");
            createRequest.setPrice(new BigDecimal("149.99"));
            createRequest.setStockQuantity(50);
            createRequest.setCategoryId(1L);
        }

        @Test
        @WithMockUser(roles = "SELLER")
        @DisplayName("Should create product with valid data")
        void createProduct_WithValidData_ShouldReturn201() throws Exception {
            // Given
            given(productService.createProduct(any())).willReturn(testProduct);

            // When & Then
            mockMvc.perform(post("/api/v1/products")
                            .with(csrf())
                            .contentType(MediaType.APPLICATION_JSON)
                            .content(objectMapper.writeValueAsString(createRequest)))
                    .andDo(print())
                    .andExpect(status().isCreated())
                    .andExpect(jsonPath("$.success", is(true)))
                    .andExpect(jsonPath("$.data.id", notNullValue()));
        }

        @Test
        @DisplayName("Should return 401 without authentication")
        void createProduct_WithoutAuth_ShouldReturn401() throws Exception {
            // When & Then
            mockMvc.perform(post("/api/v1/products")
                            .with(csrf())
                            .contentType(MediaType.APPLICATION_JSON)
                            .content(objectMapper.writeValueAsString(createRequest)))
                    .andExpect(status().isUnauthorized());
        }

        @Test
        @WithMockUser(roles = "CUSTOMER")
        @DisplayName("Should return 403 without proper role")
        void createProduct_WithWrongRole_ShouldReturn403() throws Exception {
            // When & Then
            mockMvc.perform(post("/api/v1/products")
                            .with(csrf())
                            .contentType(MediaType.APPLICATION_JSON)
                            .content(objectMapper.writeValueAsString(createRequest)))
                    .andExpect(status().isForbidden());
        }

        @Test
        @WithMockUser(roles = "SELLER")
        @DisplayName("Should return 400 with invalid data")
        void createProduct_WithInvalidData_ShouldReturn400() throws Exception {
            // Given - Invalid request (missing required fields)
            CreateProductRequest invalidRequest = new CreateProductRequest();
            invalidRequest.setName(""); // Invalid: empty name

            // When & Then
            mockMvc.perform(post("/api/v1/products")
                            .with(csrf())
                            .contentType(MediaType.APPLICATION_JSON)
                            .content(objectMapper.writeValueAsString(invalidRequest)))
                    .andExpect(status().isBadRequest())
                    .andExpect(jsonPath("$.success", is(false)))
                    .andExpect(jsonPath("$.errors", hasKey("name")));
        }
    }

    // ========== PUT /api/v1/products/{id} ==========

    @Nested
    @DisplayName("PUT /api/v1/products/{id}")
    class UpdateProductTests {

        @Test
        @WithMockUser(roles = "ADMIN")
        @DisplayName("Should update product successfully")
        void updateProduct_WithValidData_ShouldReturn200() throws Exception {
            // Given
            var updateRequest = new com.ecommerce.dto.request.UpdateProductRequest();
            updateRequest.setName("Updated Name");
            updateRequest.setPrice(new BigDecimal("199.99"));

            given(productService.updateProduct(eq(1L), any())).willReturn(testProduct);

            // When & Then
            mockMvc.perform(put("/api/v1/products/{id}", 1L)
                            .with(csrf())
                            .contentType(MediaType.APPLICATION_JSON)
                            .content(objectMapper.writeValueAsString(updateRequest)))
                    .andExpect(status().isOk())
                    .andExpect(jsonPath("$.success", is(true)));
        }
    }

    // ========== DELETE /api/v1/products/{id} ==========

    @Nested
    @DisplayName("DELETE /api/v1/products/{id}")
    class DeleteProductTests {

        @Test
        @WithMockUser(roles = "ADMIN")
        @DisplayName("Should delete product successfully")
        void deleteProduct_AsAdmin_ShouldReturn200() throws Exception {
            // Given
            doNothing().when(productService).deleteProduct(1L);

            // When & Then
            mockMvc.perform(delete("/api/v1/products/{id}", 1L)
                            .with(csrf()))
                    .andExpect(status().isOk())
                    .andExpect(jsonPath("$.success", is(true)));

            verify(productService).deleteProduct(1L);
        }

        @Test
        @WithMockUser(roles = "CUSTOMER")
        @DisplayName("Should return 403 for non-admin")
        void deleteProduct_AsCustomer_ShouldReturn403() throws Exception {
            // When & Then
            mockMvc.perform(delete("/api/v1/products/{id}", 1L)
                            .with(csrf()))
                    .andExpect(status().isForbidden());

            verify(productService, never()).deleteProduct(any());
        }
    }
}
```

---

## 3. Full Integration Tests (@SpringBootTest)

### 3.1 Order Flow Integration Test

```java
package com.ecommerce.integration;

import com.ecommerce.dto.request.AddToCartRequest;
import com.ecommerce.dto.request.CheckoutRequest;
import com.ecommerce.dto.response.CartDto;
import com.ecommerce.dto.response.OrderDto;
import com.ecommerce.service.CartService;
import com.ecommerce.service.OrderService;
import com.ecommerce.service.ProductService;
import com.fasterxml.jackson.databind.ObjectMapper;
import org.junit.jupiter.api.*;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.autoconfigure.web.servlet.AutoConfigureMockMvc;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.http.MediaType;
import org.springframework.security.test.context.support.WithUserDetails;
import org.springframework.test.context.ActiveProfiles;
import org.springframework.test.web.servlet.MockMvc;
import org.springframework.transaction.annotation.Transactional;

import static org.assertj.core.api.Assertions.assertThat;
import static org.springframework.security.test.web.servlet.request.SecurityMockMvcRequestPostProcessors.csrf;
import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.*;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.*;

@SpringBootTest
@AutoConfigureMockMvc
@ActiveProfiles("test")
@Transactional
@DisplayName("Order Flow Integration Tests")
class OrderFlowIntegrationTest {

    @Autowired
    private MockMvc mockMvc;

    @Autowired
    private ObjectMapper objectMapper;

    @Autowired
    private CartService cartService;

    @Autowired
    private OrderService orderService;

    @Autowired
    private ProductService productService;

    @Test
    @WithUserDetails("customer@test.com")
    @DisplayName("Complete order flow: Add to cart -> Checkout -> Confirm")
    void completeOrderFlow() throws Exception {
        Long productId = 1L; // Assume product exists from test data

        // Step 1: Add product to cart
        AddToCartRequest addToCartRequest = new AddToCartRequest();
        addToCartRequest.setProductId(productId);
        addToCartRequest.setQuantity(2);

        String cartResponse = mockMvc.perform(post("/api/v1/cart/items")
                        .with(csrf())
                        .contentType(MediaType.APPLICATION_JSON)
                        .content(objectMapper.writeValueAsString(addToCartRequest)))
                .andExpect(status().isOk())
                .andExpect(jsonPath("$.data.totalItems").value(2))
                .andReturn()
                .getResponse()
                .getContentAsString();

        // Step 2: Get cart and verify
        mockMvc.perform(get("/api/v1/cart"))
                .andExpect(status().isOk())
                .andExpect(jsonPath("$.data.items").isNotEmpty());

        // Step 3: Checkout
        CheckoutRequest checkoutRequest = new CheckoutRequest();
        checkoutRequest.setShippingAddress(createShippingAddress());
        checkoutRequest.setPaymentMethod(com.ecommerce.domain.enums.PaymentMethod.CREDIT_CARD);

        String orderResponse = mockMvc.perform(post("/api/v1/orders/checkout")
                        .with(csrf())
                        .contentType(MediaType.APPLICATION_JSON)
                        .content(objectMapper.writeValueAsString(checkoutRequest)))
                .andExpect(status().isCreated())
                .andExpect(jsonPath("$.data.orderNumber").isNotEmpty())
                .andExpect(jsonPath("$.data.status").value("PENDING"))
                .andReturn()
                .getResponse()
                .getContentAsString();

        // Extract order ID from response
        OrderDto order = objectMapper.readValue(
                objectMapper.readTree(orderResponse).path("data").toString(),
                OrderDto.class
        );

        // Step 4: Verify cart is empty
        mockMvc.perform(get("/api/v1/cart"))
                .andExpect(status().isOk())
                .andExpect(jsonPath("$.data.totalItems").value(0));

        // Step 5: View order
        mockMvc.perform(get("/api/v1/orders/{id}", order.getId()))
                .andExpect(status().isOk())
                .andExpect(jsonPath("$.data.orderNumber").value(order.getOrderNumber()));
    }

    @Test
    @WithUserDetails("customer@test.com")
    @DisplayName("Should prevent checkout with empty cart")
    void checkout_WithEmptyCart_ShouldFail() throws Exception {
        // Given: Empty cart

        CheckoutRequest request = new CheckoutRequest();
        request.setShippingAddress(createShippingAddress());
        request.setPaymentMethod(com.ecommerce.domain.enums.PaymentMethod.CREDIT_CARD);

        // When & Then
        mockMvc.perform(post("/api/v1/orders/checkout")
                        .with(csrf())
                        .contentType(MediaType.APPLICATION_JSON)
                        .content(objectMapper.writeValueAsString(request)))
                .andExpect(status().isBadRequest())
                .andExpect(jsonPath("$.message").value("Cart is empty"));
    }

    private CheckoutRequest.ShippingAddressDto createShippingAddress() {
        CheckoutRequest.ShippingAddressDto address = new CheckoutRequest.ShippingAddressDto();
        address.setRecipientName("John Doe");
        address.setPhone("+1234567890");
        address.setAddressLine1("123 Main St");
        address.setCity("New York");
        address.setState("NY");
        address.setPostalCode("10001");
        address.setCountry("USA");
        return address;
    }
}
```

---

## 4. TestContainers for MySQL

### 4.1 Base Test with TestContainers

```java
package com.ecommerce.integration;

import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.test.context.DynamicPropertyRegistry;
import org.springframework.test.context.DynamicPropertySource;
import org.testcontainers.containers.MySQLContainer;
import org.testcontainers.junit.jupiter.Container;
import org.testcontainers.junit.jupiter.Testcontainers;

@SpringBootTest
@Testcontainers
public abstract class BaseContainerTest {

    @Container
    static MySQLContainer<?> mysql = new MySQLContainer<>("mysql:8.0")
            .withDatabaseName("ecommerce_test")
            .withUsername("test")
            .withPassword("test")
            .withReuse(true);

    @DynamicPropertySource
    static void configureProperties(DynamicPropertyRegistry registry) {
        registry.add("spring.datasource.url", mysql::getJdbcUrl);
        registry.add("spring.datasource.username", mysql::getUsername);
        registry.add("spring.datasource.password", mysql::getPassword);
        registry.add("spring.flyway.enabled", () -> "true");
    }
}
```

### 4.2 Repository Test with Real MySQL

```java
package com.ecommerce.repository;

import com.ecommerce.domain.entity.Product;
import com.ecommerce.integration.BaseContainerTest;
import org.junit.jupiter.api.*;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.autoconfigure.jdbc.AutoConfigureTestDatabase;
import org.springframework.boot.test.autoconfigure.orm.jpa.DataJpaTest;
import org.springframework.data.domain.Page;
import org.springframework.data.domain.PageRequest;
import org.springframework.test.context.ActiveProfiles;
import org.testcontainers.junit.jupiter.Testcontainers;

import static org.assertj.core.api.Assertions.assertThat;

@DataJpaTest
@AutoConfigureTestDatabase(replace = AutoConfigureTestDatabase.Replace.NONE)
@Testcontainers
@ActiveProfiles("test-container")
@DisplayName("ProductRepository MySQL Tests")
class ProductRepositoryMySQLTest extends BaseContainerTest {

    @Autowired
    private ProductRepository productRepository;

    @Test
    @DisplayName("Should perform full-text search on MySQL")
    void fullTextSearch_ShouldWork() {
        // Given - Test data loaded via Flyway

        // When
        Page<Product> results = productRepository.fullTextSearch(
                "laptop gaming",
                PageRequest.of(0, 10)
        );

        // Then
        assertThat(results.getContent()).isNotEmpty();
    }

    @Test
    @DisplayName("Should handle MySQL specific features")
    void mysqlSpecificQuery_ShouldWork() {
        // Test MySQL-specific queries that H2 can't handle
    }
}
```

### 4.3 Redis TestContainer

```java
package com.ecommerce.integration;

import org.springframework.test.context.DynamicPropertyRegistry;
import org.springframework.test.context.DynamicPropertySource;
import org.testcontainers.containers.GenericContainer;
import org.testcontainers.junit.jupiter.Container;
import org.testcontainers.junit.jupiter.Testcontainers;
import org.testcontainers.utility.DockerImageName;

@Testcontainers
public abstract class BaseRedisTest {

    @Container
    static GenericContainer<?> redis = new GenericContainer<>(
            DockerImageName.parse("redis:7-alpine"))
            .withExposedPorts(6379);

    @DynamicPropertySource
    static void redisProperties(DynamicPropertyRegistry registry) {
        registry.add("spring.data.redis.host", redis::getHost);
        registry.add("spring.data.redis.port", redis::getFirstMappedPort);
    }
}
```

---

## 5. Security Integration Tests

### 5.1 Authentication Tests

```java
package com.ecommerce.security;

import com.ecommerce.dto.request.LoginRequest;
import com.ecommerce.dto.request.RegisterRequest;
import com.fasterxml.jackson.databind.ObjectMapper;
import org.junit.jupiter.api.*;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.autoconfigure.web.servlet.AutoConfigureMockMvc;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.http.MediaType;
import org.springframework.test.context.ActiveProfiles;
import org.springframework.test.web.servlet.MockMvc;
import org.springframework.test.web.servlet.MvcResult;
import org.springframework.transaction.annotation.Transactional;

import static org.springframework.security.test.web.servlet.request.SecurityMockMvcRequestPostProcessors.csrf;
import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.*;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.*;

@SpringBootTest
@AutoConfigureMockMvc
@ActiveProfiles("test")
@Transactional
@DisplayName("Authentication Integration Tests")
class AuthenticationIntegrationTest {

    @Autowired
    private MockMvc mockMvc;

    @Autowired
    private ObjectMapper objectMapper;

    @Nested
    @DisplayName("Login Tests")
    class LoginTests {

        @Test
        @DisplayName("Should login with valid credentials")
        void login_WithValidCredentials_ShouldReturnTokens() throws Exception {
            // Given
            LoginRequest loginRequest = new LoginRequest();
            loginRequest.setEmail("admin@test.com");
            loginRequest.setPassword("password123");

            // When & Then
            mockMvc.perform(post("/api/v1/auth/login")
                            .with(csrf())
                            .contentType(MediaType.APPLICATION_JSON)
                            .content(objectMapper.writeValueAsString(loginRequest)))
                    .andExpect(status().isOk())
                    .andExpect(jsonPath("$.data.accessToken").isNotEmpty())
                    .andExpect(jsonPath("$.data.refreshToken").isNotEmpty())
                    .andExpect(jsonPath("$.data.tokenType").value("Bearer"));
        }

        @Test
        @DisplayName("Should reject invalid credentials")
        void login_WithInvalidCredentials_ShouldReturn401() throws Exception {
            // Given
            LoginRequest loginRequest = new LoginRequest();
            loginRequest.setEmail("admin@test.com");
            loginRequest.setPassword("wrongpassword");

            // When & Then
            mockMvc.perform(post("/api/v1/auth/login")
                            .with(csrf())
                            .contentType(MediaType.APPLICATION_JSON)
                            .content(objectMapper.writeValueAsString(loginRequest)))
                    .andExpect(status().isUnauthorized());
        }
    }

    @Nested
    @DisplayName("Protected Endpoint Tests")
    class ProtectedEndpointTests {

        @Test
        @DisplayName("Should access protected endpoint with valid token")
        void protectedEndpoint_WithValidToken_ShouldSucceed() throws Exception {
            // Given - Login first
            String token = obtainAccessToken("admin@test.com", "password123");

            // When & Then
            mockMvc.perform(get("/api/v1/users/me")
                            .header("Authorization", "Bearer " + token))
                    .andExpect(status().isOk())
                    .andExpect(jsonPath("$.data.email").value("admin@test.com"));
        }

        @Test
        @DisplayName("Should reject request without token")
        void protectedEndpoint_WithoutToken_ShouldReturn401() throws Exception {
            mockMvc.perform(get("/api/v1/users/me"))
                    .andExpect(status().isUnauthorized());
        }

        @Test
        @DisplayName("Should reject request with invalid token")
        void protectedEndpoint_WithInvalidToken_ShouldReturn401() throws Exception {
            mockMvc.perform(get("/api/v1/users/me")
                            .header("Authorization", "Bearer invalid-token"))
                    .andExpect(status().isUnauthorized());
        }
    }

    @Nested
    @DisplayName("Token Refresh Tests")
    class TokenRefreshTests {

        @Test
        @DisplayName("Should refresh token successfully")
        void refreshToken_WithValidRefreshToken_ShouldReturnNewTokens() throws Exception {
            // Given - Login first
            MvcResult loginResult = mockMvc.perform(post("/api/v1/auth/login")
                            .with(csrf())
                            .contentType(MediaType.APPLICATION_JSON)
                            .content("""
                                {"email": "admin@test.com", "password": "password123"}
                            """))
                    .andReturn();

            String refreshToken = objectMapper.readTree(
                    loginResult.getResponse().getContentAsString()
            ).path("data").path("refreshToken").asText();

            // When & Then
            mockMvc.perform(post("/api/v1/auth/refresh")
                            .with(csrf())
                            .contentType(MediaType.APPLICATION_JSON)
                            .content("{\"refreshToken\": \"" + refreshToken + "\"}"))
                    .andExpect(status().isOk())
                    .andExpect(jsonPath("$.data.accessToken").isNotEmpty());
        }
    }

    private String obtainAccessToken(String email, String password) throws Exception {
        MvcResult result = mockMvc.perform(post("/api/v1/auth/login")
                        .with(csrf())
                        .contentType(MediaType.APPLICATION_JSON)
                        .content(String.format(
                                "{\"email\": \"%s\", \"password\": \"%s\"}",
                                email, password
                        )))
                .andReturn();

        return objectMapper.readTree(result.getResponse().getContentAsString())
                .path("data")
                .path("accessToken")
                .asText();
    }
}
```

---

## 6. API Contract Tests

### 6.1 OpenAPI Contract Test

```java
package com.ecommerce.contract;

import io.restassured.RestAssured;
import io.restassured.http.ContentType;
import org.junit.jupiter.api.*;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.boot.test.web.server.LocalServerPort;
import org.springframework.test.context.ActiveProfiles;

import static io.restassured.RestAssured.*;
import static org.hamcrest.Matchers.*;

@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
@ActiveProfiles("test")
@DisplayName("API Contract Tests")
class ApiContractTest {

    @LocalServerPort
    private int port;

    @BeforeEach
    void setUp() {
        RestAssured.port = port;
        RestAssured.basePath = "/api/v1";
    }

    @Test
    @DisplayName("GET /products should return array of products")
    void getProducts_ShouldReturnArrayOfProducts() {
        given()
            .contentType(ContentType.JSON)
        .when()
            .get("/products")
        .then()
            .statusCode(200)
            .body("success", equalTo(true))
            .body("data.content", isA(java.util.List.class))
            .body("data.content[0].id", notNullValue())
            .body("data.content[0].name", notNullValue())
            .body("data.content[0].price", notNullValue());
    }

    @Test
    @DisplayName("GET /products/{id} should return product schema")
    void getProduct_ShouldMatchSchema() {
        given()
            .contentType(ContentType.JSON)
        .when()
            .get("/products/1")
        .then()
            .statusCode(200)
            .body("data.id", isA(Integer.class))
            .body("data.name", isA(String.class))
            .body("data.slug", isA(String.class))
            .body("data.price", isA(Number.class))
            .body("data.stockQuantity", isA(Integer.class));
    }

    @Test
    @DisplayName("POST /products should require authentication")
    void createProduct_WithoutAuth_ShouldReturn401() {
        given()
            .contentType(ContentType.JSON)
            .body("{\"name\": \"Test\"}")
        .when()
            .post("/products")
        .then()
            .statusCode(401);
    }
}
```

---

## 7. Test Data Setup

### 7.1 Test Data SQL

```sql
-- src/test/resources/test-data.sql

-- Test categories
INSERT INTO categories (id, name, slug, created_at) VALUES
(1, 'Electronics', 'electronics', NOW()),
(2, 'Clothing', 'clothing', NOW());

-- Test products
INSERT INTO products (id, name, slug, sku, description, price, stock_quantity, category_id, active, created_at) VALUES
(1, 'iPhone 15', 'iphone-15', 'SKU-IP15', 'Latest iPhone', 999.99, 100, 1, true, NOW()),
(2, 'MacBook Pro', 'macbook-pro', 'SKU-MBP', 'Professional laptop', 2499.99, 50, 1, true, NOW()),
(3, 'T-Shirt', 't-shirt', 'SKU-TS', 'Cotton T-Shirt', 29.99, 200, 2, true, NOW());

-- Test users
INSERT INTO users (id, email, password, first_name, last_name, email_verified, created_at) VALUES
(1, 'admin@test.com', '$2a$10$encrypted_password', 'Admin', 'User', true, NOW()),
(2, 'customer@test.com', '$2a$10$encrypted_password', 'Customer', 'User', true, NOW());

-- Test roles
INSERT INTO user_roles (user_id, role_id) VALUES
(1, 1), -- Admin role
(2, 2); -- Customer role
```

### 7.2 Test Configuration

```java
package com.ecommerce.config;

import org.springframework.boot.test.context.TestConfiguration;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Primary;
import org.springframework.security.crypto.bcrypt.BCryptPasswordEncoder;
import org.springframework.security.crypto.password.PasswordEncoder;

@TestConfiguration
public class TestConfig {

    @Bean
    @Primary
    public PasswordEncoder testPasswordEncoder() {
        return new BCryptPasswordEncoder(4); // Lower strength for faster tests
    }
}
```

---

## 8. Bài tập thực hành

### Bài tập 1: Controller Tests
- [x] Test ProductController với @WebMvcTest
- [x] Test authentication scenarios
- [x] Test validation errors

### Bài tập 2: Full Integration
- [x] Test order flow end-to-end
- [x] Test with authenticated users
- [x] Verify database state

### Bài tập 3: TestContainers
- [ ] Setup MySQL container
- [ ] Setup Redis container
- [ ] Test with real database

---

## 9. Checklist

- [ ] Setup @WebMvcTest for controllers
- [ ] Create MockMvc tests for all endpoints
- [ ] Test authentication và authorization
- [ ] Setup @SpringBootTest for integration
- [ ] Configure TestContainers
- [ ] Create test data setup
- [ ] Test complete user flows
- [ ] Add API contract tests
- [ ] Run integration tests in CI

---

## Điều hướng

- [← Day 26: Unit Testing](./day-26-unit-testing.md)
- [Day 28: Docker & Deployment →](./day-28-docker-deployment.md)
- [Về trang chính](../00-overview.md)
