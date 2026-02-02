# Day 10: Microservices Testing

## Mục tiêu
- Testing pyramid for microservices
- Unit testing với Mockito
- Integration testing
- Contract testing với Pact
- End-to-end testing

---

## 1. Testing Pyramid

### 1.1. Testing Levels

```
┌─────────────────────────────────────────────────────────────────┐
│                    TESTING PYRAMID                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│                          /\                                      │
│                         /  \                                     │
│                        / E2E\        Few, slow, expensive       │
│                       /──────\                                   │
│                      /        \                                  │
│                     / Contract \     Integration points         │
│                    /────────────\                                │
│                   /              \                               │
│                  /  Integration   \  Service + dependencies     │
│                 /──────────────────\                             │
│                /                    \                            │
│               /        Unit          \  Many, fast, cheap       │
│              /────────────────────────\                          │
│                                                                  │
│   Microservices Testing Considerations:                         │
│   ├── Each service has its own pyramid                          │
│   ├── Contract tests verify service boundaries                  │
│   ├── Integration tests with real dependencies (containers)     │
│   └── E2E tests span multiple services                          │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 1.2. Test Categories

```
┌────────────────────┬────────────────────────────────────────────┐
│     Test Type      │              Description                    │
├────────────────────┼────────────────────────────────────────────┤
│ Unit Tests         │ Test isolated classes with mocks           │
│ Component Tests    │ Test service in isolation (mocked deps)    │
│ Integration Tests  │ Test with real dependencies (DB, Kafka)    │
│ Contract Tests     │ Verify API contracts between services      │
│ E2E Tests          │ Test entire flow across services           │
└────────────────────┴────────────────────────────────────────────┘
```

---

## 2. Unit Testing

### 2.1. Dependencies

```xml
<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-test</artifactId>
        <scope>test</scope>
    </dependency>
    <dependency>
        <groupId>org.mockito</groupId>
        <artifactId>mockito-junit-jupiter</artifactId>
        <scope>test</scope>
    </dependency>
</dependencies>
```

### 2.2. Service Layer Tests

```java
@ExtendWith(MockitoExtension.class)
class OrderServiceTest {

    @Mock
    private OrderRepository orderRepository;

    @Mock
    private UserClient userClient;

    @Mock
    private InventoryClient inventoryClient;

    @Mock
    private KafkaTemplate<String, Object> kafkaTemplate;

    @InjectMocks
    private OrderService orderService;

    @Captor
    private ArgumentCaptor<Order> orderCaptor;

    @Test
    @DisplayName("Should create order successfully")
    void createOrder_Success() {
        // Given
        CreateOrderRequest request = CreateOrderRequest.builder()
            .userId("user-123")
            .items(List.of(
                new OrderItem("product-1", 2),
                new OrderItem("product-2", 1)
            ))
            .build();

        User user = new User("user-123", "John Doe", "john@example.com");
        when(userClient.getUser("user-123")).thenReturn(user);

        when(inventoryClient.checkAvailability(anyList()))
            .thenReturn(new AvailabilityResult(true, List.of()));

        when(orderRepository.save(any(Order.class)))
            .thenAnswer(invocation -> {
                Order order = invocation.getArgument(0);
                order.setId("order-123");
                return order;
            });

        // When
        Order result = orderService.createOrder(request);

        // Then
        assertThat(result.getId()).isEqualTo("order-123");
        assertThat(result.getUserId()).isEqualTo("user-123");
        assertThat(result.getStatus()).isEqualTo(OrderStatus.PENDING);

        verify(orderRepository).save(orderCaptor.capture());
        Order savedOrder = orderCaptor.getValue();
        assertThat(savedOrder.getItems()).hasSize(2);

        verify(kafkaTemplate).send(eq("order-events"), anyString(), any(OrderCreatedEvent.class));
    }

    @Test
    @DisplayName("Should throw exception when user not found")
    void createOrder_UserNotFound() {
        // Given
        CreateOrderRequest request = CreateOrderRequest.builder()
            .userId("invalid-user")
            .items(List.of(new OrderItem("product-1", 1)))
            .build();

        when(userClient.getUser("invalid-user"))
            .thenThrow(new UserNotFoundException("User not found"));

        // When & Then
        assertThatThrownBy(() -> orderService.createOrder(request))
            .isInstanceOf(UserNotFoundException.class)
            .hasMessage("User not found");

        verify(orderRepository, never()).save(any());
        verify(kafkaTemplate, never()).send(anyString(), anyString(), any());
    }

    @Test
    @DisplayName("Should throw exception when inventory not available")
    void createOrder_InsufficientInventory() {
        // Given
        CreateOrderRequest request = CreateOrderRequest.builder()
            .userId("user-123")
            .items(List.of(new OrderItem("product-1", 100)))
            .build();

        when(userClient.getUser("user-123")).thenReturn(new User("user-123", "John", "john@example.com"));
        when(inventoryClient.checkAvailability(anyList()))
            .thenReturn(new AvailabilityResult(false, List.of("product-1")));

        // When & Then
        assertThatThrownBy(() -> orderService.createOrder(request))
            .isInstanceOf(InsufficientInventoryException.class);
    }
}
```

### 2.3. Controller Layer Tests

```java
@WebMvcTest(OrderController.class)
class OrderControllerTest {

    @Autowired
    private MockMvc mockMvc;

    @MockBean
    private OrderService orderService;

    @Autowired
    private ObjectMapper objectMapper;

    @Test
    @DisplayName("POST /api/orders - Should create order")
    @WithMockUser(roles = "USER")
    void createOrder_Success() throws Exception {
        // Given
        CreateOrderRequest request = CreateOrderRequest.builder()
            .userId("user-123")
            .items(List.of(new OrderItem("product-1", 2)))
            .build();

        Order order = Order.builder()
            .id("order-123")
            .userId("user-123")
            .status(OrderStatus.PENDING)
            .build();

        when(orderService.createOrder(any(CreateOrderRequest.class)))
            .thenReturn(order);

        // When & Then
        mockMvc.perform(post("/api/orders")
                .contentType(MediaType.APPLICATION_JSON)
                .content(objectMapper.writeValueAsString(request)))
            .andExpect(status().isCreated())
            .andExpect(jsonPath("$.id").value("order-123"))
            .andExpect(jsonPath("$.status").value("PENDING"));

        verify(orderService).createOrder(any(CreateOrderRequest.class));
    }

    @Test
    @DisplayName("GET /api/orders/{id} - Should return order")
    @WithMockUser(roles = "USER")
    void getOrder_Success() throws Exception {
        // Given
        Order order = Order.builder()
            .id("order-123")
            .userId("user-123")
            .status(OrderStatus.CONFIRMED)
            .build();

        when(orderService.getOrder("order-123")).thenReturn(order);

        // When & Then
        mockMvc.perform(get("/api/orders/order-123"))
            .andExpect(status().isOk())
            .andExpect(jsonPath("$.id").value("order-123"))
            .andExpect(jsonPath("$.status").value("CONFIRMED"));
    }

    @Test
    @DisplayName("GET /api/orders/{id} - Should return 404 when not found")
    @WithMockUser(roles = "USER")
    void getOrder_NotFound() throws Exception {
        when(orderService.getOrder("invalid-id"))
            .thenThrow(new OrderNotFoundException("Order not found"));

        mockMvc.perform(get("/api/orders/invalid-id"))
            .andExpect(status().isNotFound());
    }

    @Test
    @DisplayName("POST /api/orders - Should return 400 for invalid request")
    @WithMockUser(roles = "USER")
    void createOrder_ValidationError() throws Exception {
        CreateOrderRequest request = new CreateOrderRequest();  // Missing required fields

        mockMvc.perform(post("/api/orders")
                .contentType(MediaType.APPLICATION_JSON)
                .content(objectMapper.writeValueAsString(request)))
            .andExpect(status().isBadRequest());
    }

    @Test
    @DisplayName("POST /api/orders - Should return 401 without authentication")
    void createOrder_Unauthorized() throws Exception {
        mockMvc.perform(post("/api/orders")
                .contentType(MediaType.APPLICATION_JSON)
                .content("{}"))
            .andExpect(status().isUnauthorized());
    }
}
```

---

## 3. Integration Testing

### 3.1. Testcontainers Setup

```xml
<dependency>
    <groupId>org.testcontainers</groupId>
    <artifactId>testcontainers</artifactId>
    <scope>test</scope>
</dependency>
<dependency>
    <groupId>org.testcontainers</groupId>
    <artifactId>mysql</artifactId>
    <scope>test</scope>
</dependency>
<dependency>
    <groupId>org.testcontainers</groupId>
    <artifactId>kafka</artifactId>
    <scope>test</scope>
</dependency>
<dependency>
    <groupId>org.testcontainers</groupId>
    <artifactId>junit-jupiter</artifactId>
    <scope>test</scope>
</dependency>
```

### 3.2. Repository Integration Tests

```java
@DataJpaTest
@AutoConfigureTestDatabase(replace = AutoConfigureTestDatabase.Replace.NONE)
@Testcontainers
class OrderRepositoryTest {

    @Container
    static MySQLContainer<?> mysql = new MySQLContainer<>("mysql:8.0")
        .withDatabaseName("testdb")
        .withUsername("test")
        .withPassword("test");

    @DynamicPropertySource
    static void configureProperties(DynamicPropertyRegistry registry) {
        registry.add("spring.datasource.url", mysql::getJdbcUrl);
        registry.add("spring.datasource.username", mysql::getUsername);
        registry.add("spring.datasource.password", mysql::getPassword);
    }

    @Autowired
    private OrderRepository orderRepository;

    @Autowired
    private TestEntityManager entityManager;

    @Test
    @DisplayName("Should find orders by user ID")
    void findByUserId() {
        // Given
        Order order1 = Order.builder()
            .userId("user-123")
            .status(OrderStatus.PENDING)
            .build();
        Order order2 = Order.builder()
            .userId("user-123")
            .status(OrderStatus.CONFIRMED)
            .build();
        Order order3 = Order.builder()
            .userId("user-456")
            .status(OrderStatus.PENDING)
            .build();

        entityManager.persist(order1);
        entityManager.persist(order2);
        entityManager.persist(order3);
        entityManager.flush();

        // When
        List<Order> result = orderRepository.findByUserId("user-123");

        // Then
        assertThat(result).hasSize(2);
        assertThat(result).allMatch(o -> o.getUserId().equals("user-123"));
    }

    @Test
    @DisplayName("Should find orders by status")
    void findByStatus() {
        // Given
        Order order1 = Order.builder()
            .userId("user-123")
            .status(OrderStatus.PENDING)
            .build();
        Order order2 = Order.builder()
            .userId("user-456")
            .status(OrderStatus.PENDING)
            .build();

        entityManager.persist(order1);
        entityManager.persist(order2);
        entityManager.flush();

        // When
        List<Order> result = orderRepository.findByStatus(OrderStatus.PENDING);

        // Then
        assertThat(result).hasSize(2);
    }
}
```

### 3.3. Full Integration Test

```java
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
@Testcontainers
@ActiveProfiles("test")
class OrderServiceIntegrationTest {

    @Container
    static MySQLContainer<?> mysql = new MySQLContainer<>("mysql:8.0")
        .withDatabaseName("testdb");

    @Container
    static KafkaContainer kafka = new KafkaContainer(
        DockerImageName.parse("confluentinc/cp-kafka:7.4.0"));

    @DynamicPropertySource
    static void configureProperties(DynamicPropertyRegistry registry) {
        registry.add("spring.datasource.url", mysql::getJdbcUrl);
        registry.add("spring.datasource.username", mysql::getUsername);
        registry.add("spring.datasource.password", mysql::getPassword);
        registry.add("spring.kafka.bootstrap-servers", kafka::getBootstrapServers);
    }

    @Autowired
    private TestRestTemplate restTemplate;

    @Autowired
    private OrderRepository orderRepository;

    @MockBean
    private UserClient userClient;

    @MockBean
    private InventoryClient inventoryClient;

    @BeforeEach
    void setUp() {
        orderRepository.deleteAll();
    }

    @Test
    @DisplayName("Should create and retrieve order")
    void createAndRetrieveOrder() {
        // Setup mocks
        when(userClient.getUser("user-123"))
            .thenReturn(new User("user-123", "John", "john@example.com"));
        when(inventoryClient.checkAvailability(anyList()))
            .thenReturn(new AvailabilityResult(true, List.of()));

        // Create order
        CreateOrderRequest request = CreateOrderRequest.builder()
            .userId("user-123")
            .items(List.of(new OrderItem("product-1", 2)))
            .build();

        ResponseEntity<Order> createResponse = restTemplate.postForEntity(
            "/api/orders", request, Order.class);

        assertThat(createResponse.getStatusCode()).isEqualTo(HttpStatus.CREATED);
        String orderId = createResponse.getBody().getId();

        // Retrieve order
        ResponseEntity<Order> getResponse = restTemplate.getForEntity(
            "/api/orders/" + orderId, Order.class);

        assertThat(getResponse.getStatusCode()).isEqualTo(HttpStatus.OK);
        assertThat(getResponse.getBody().getUserId()).isEqualTo("user-123");
    }
}
```

---

## 4. Contract Testing with Pact

### 4.1. Consumer Side (Order Service)

```xml
<dependency>
    <groupId>au.com.dius.pact.consumer</groupId>
    <artifactId>junit5</artifactId>
    <version>4.6.5</version>
    <scope>test</scope>
</dependency>
```

```java
@ExtendWith(PactConsumerTestExt.class)
@PactTestFor(providerName = "user-service")
class UserClientContractTest {

    @Pact(consumer = "order-service")
    public V4Pact getUserPact(PactDslWithProvider builder) {
        return builder
            .given("user exists")
            .uponReceiving("get user by ID")
                .path("/api/users/user-123")
                .method("GET")
            .willRespondWith()
                .status(200)
                .headers(Map.of("Content-Type", "application/json"))
                .body(new PactDslJsonBody()
                    .stringValue("id", "user-123")
                    .stringValue("name", "John Doe")
                    .stringValue("email", "john@example.com"))
            .toPact(V4Pact.class);
    }

    @Pact(consumer = "order-service")
    public V4Pact getUserNotFoundPact(PactDslWithProvider builder) {
        return builder
            .given("user does not exist")
            .uponReceiving("get non-existent user")
                .path("/api/users/invalid-user")
                .method("GET")
            .willRespondWith()
                .status(404)
                .body(new PactDslJsonBody()
                    .stringValue("error", "User not found"))
            .toPact(V4Pact.class);
    }

    @Test
    @PactTestFor(pactMethod = "getUserPact")
    void testGetUser(MockServer mockServer) {
        // Configure client to use mock server
        UserClient userClient = new UserClient(
            WebClient.builder().baseUrl(mockServer.getUrl()).build());

        User user = userClient.getUser("user-123");

        assertThat(user.getId()).isEqualTo("user-123");
        assertThat(user.getName()).isEqualTo("John Doe");
        assertThat(user.getEmail()).isEqualTo("john@example.com");
    }

    @Test
    @PactTestFor(pactMethod = "getUserNotFoundPact")
    void testGetUserNotFound(MockServer mockServer) {
        UserClient userClient = new UserClient(
            WebClient.builder().baseUrl(mockServer.getUrl()).build());

        assertThatThrownBy(() -> userClient.getUser("invalid-user"))
            .isInstanceOf(UserNotFoundException.class);
    }
}
```

### 4.2. Provider Side (User Service)

```xml
<dependency>
    <groupId>au.com.dius.pact.provider</groupId>
    <artifactId>junit5spring</artifactId>
    <version>4.6.5</version>
    <scope>test</scope>
</dependency>
```

```java
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
@Provider("user-service")
@PactBroker(url = "${pact.broker.url}")
// Or use local pacts: @PactFolder("pacts")
class UserServiceContractVerificationTest {

    @LocalServerPort
    private int port;

    @MockBean
    private UserRepository userRepository;

    @BeforeEach
    void setUp(PactVerificationContext context) {
        context.setTarget(new HttpTestTarget("localhost", port));
    }

    @TestTemplate
    @ExtendWith(PactVerificationInvocationContextProvider.class)
    void verifyPact(PactVerificationContext context) {
        context.verifyInteraction();
    }

    @State("user exists")
    void userExists() {
        User user = new User("user-123", "John Doe", "john@example.com");
        when(userRepository.findById("user-123")).thenReturn(Optional.of(user));
    }

    @State("user does not exist")
    void userDoesNotExist() {
        when(userRepository.findById("invalid-user")).thenReturn(Optional.empty());
    }
}
```

---

## 5. Component Testing

### 5.1. WireMock for External Services

```xml
<dependency>
    <groupId>org.wiremock</groupId>
    <artifactId>wiremock-standalone</artifactId>
    <version>3.3.1</version>
    <scope>test</scope>
</dependency>
```

```java
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
@WireMockTest(httpPort = 8081)
class OrderServiceComponentTest {

    @Autowired
    private TestRestTemplate restTemplate;

    @Test
    @DisplayName("Should create order when all external services succeed")
    void createOrder_AllServicesSucceed() {
        // Mock User Service
        stubFor(get(urlEqualTo("/api/users/user-123"))
            .willReturn(aResponse()
                .withStatus(200)
                .withHeader("Content-Type", "application/json")
                .withBody("""
                    {
                        "id": "user-123",
                        "name": "John Doe",
                        "email": "john@example.com"
                    }
                    """)));

        // Mock Inventory Service
        stubFor(post(urlEqualTo("/api/inventory/check"))
            .willReturn(aResponse()
                .withStatus(200)
                .withHeader("Content-Type", "application/json")
                .withBody("""
                    {
                        "available": true,
                        "unavailableItems": []
                    }
                    """)));

        // Test
        CreateOrderRequest request = CreateOrderRequest.builder()
            .userId("user-123")
            .items(List.of(new OrderItem("product-1", 2)))
            .build();

        ResponseEntity<Order> response = restTemplate.postForEntity(
            "/api/orders", request, Order.class);

        assertThat(response.getStatusCode()).isEqualTo(HttpStatus.CREATED);

        // Verify external calls
        verify(getRequestedFor(urlEqualTo("/api/users/user-123")));
        verify(postRequestedFor(urlEqualTo("/api/inventory/check")));
    }

    @Test
    @DisplayName("Should handle user service failure")
    void createOrder_UserServiceFailure() {
        // Mock User Service failure
        stubFor(get(urlEqualTo("/api/users/user-123"))
            .willReturn(aResponse()
                .withStatus(500)
                .withHeader("Content-Type", "application/json")
                .withBody("{\"error\": \"Internal Server Error\"}")));

        CreateOrderRequest request = CreateOrderRequest.builder()
            .userId("user-123")
            .items(List.of(new OrderItem("product-1", 2)))
            .build();

        ResponseEntity<ErrorResponse> response = restTemplate.postForEntity(
            "/api/orders", request, ErrorResponse.class);

        assertThat(response.getStatusCode()).isEqualTo(HttpStatus.SERVICE_UNAVAILABLE);
    }
}
```

---

## 6. Kafka Testing

### 6.1. Embedded Kafka

```java
@SpringBootTest
@EmbeddedKafka(
    partitions = 1,
    topics = {"order-events"},
    brokerProperties = {"listeners=PLAINTEXT://localhost:9092"}
)
@TestPropertySource(properties = {
    "spring.kafka.bootstrap-servers=${spring.embedded.kafka.brokers}"
})
class KafkaIntegrationTest {

    @Autowired
    private KafkaTemplate<String, Object> kafkaTemplate;

    @Autowired
    private EmbeddedKafkaBroker embeddedKafka;

    @Test
    @DisplayName("Should publish and consume order event")
    void publishAndConsumeEvent() throws Exception {
        // Create consumer
        Map<String, Object> consumerProps = KafkaTestUtils.consumerProps(
            "test-group", "true", embeddedKafka);
        consumerProps.put(ConsumerConfig.AUTO_OFFSET_RESET_CONFIG, "earliest");

        DefaultKafkaConsumerFactory<String, String> consumerFactory =
            new DefaultKafkaConsumerFactory<>(consumerProps);

        Consumer<String, String> consumer = consumerFactory.createConsumer();
        embeddedKafka.consumeFromAnEmbeddedTopic(consumer, "order-events");

        // Publish event
        OrderCreatedEvent event = new OrderCreatedEvent("order-123", "user-123");
        kafkaTemplate.send("order-events", "order-123", event).get();

        // Verify
        ConsumerRecords<String, String> records = KafkaTestUtils.getRecords(consumer);
        assertThat(records.count()).isEqualTo(1);

        consumer.close();
    }
}
```

### 6.2. Testing Kafka Listeners

```java
@SpringBootTest
@EmbeddedKafka(topics = {"inventory-events"})
@TestPropertySource(properties = {
    "spring.kafka.bootstrap-servers=${spring.embedded.kafka.brokers}"
})
class InventoryEventListenerTest {

    @Autowired
    private KafkaTemplate<String, Object> kafkaTemplate;

    @Autowired
    private OrderRepository orderRepository;

    @SpyBean
    private OrderService orderService;

    @Test
    @DisplayName("Should process inventory reserved event")
    void processInventoryReservedEvent() throws Exception {
        // Setup
        Order order = Order.builder()
            .id("order-123")
            .status(OrderStatus.PENDING)
            .build();
        orderRepository.save(order);

        // Publish event
        InventoryReservedEvent event = new InventoryReservedEvent("order-123");
        kafkaTemplate.send("inventory-events", "order-123", event).get();

        // Wait for processing
        await().atMost(5, TimeUnit.SECONDS)
            .untilAsserted(() -> {
                verify(orderService).handleInventoryReserved(any(InventoryReservedEvent.class));
            });

        // Verify
        Order updatedOrder = orderRepository.findById("order-123").orElseThrow();
        assertThat(updatedOrder.getStatus()).isEqualTo(OrderStatus.INVENTORY_RESERVED);
    }
}
```

---

## 7. End-to-End Testing

### 7.1. Docker Compose Test Setup

```yaml
# docker-compose.test.yml
version: '3.8'
services:
  user-service:
    image: user-service:test
    ports:
      - "8081:8081"
    environment:
      - SPRING_PROFILES_ACTIVE=test

  order-service:
    image: order-service:test
    ports:
      - "8082:8082"
    depends_on:
      - user-service
      - mysql
      - kafka
    environment:
      - SPRING_PROFILES_ACTIVE=test
      - USER_SERVICE_URL=http://user-service:8081

  mysql:
    image: mysql:8.0
    environment:
      MYSQL_ROOT_PASSWORD: test
      MYSQL_DATABASE: testdb

  kafka:
    image: confluentinc/cp-kafka:7.4.0
```

### 7.2. E2E Test

```java
@SpringBootTest
@Testcontainers
class OrderFlowE2ETest {

    @Container
    static DockerComposeContainer<?> environment =
        new DockerComposeContainer<>(new File("docker-compose.test.yml"))
            .withExposedService("user-service", 8081)
            .withExposedService("order-service", 8082)
            .waitingFor("order-service",
                Wait.forHttp("/actuator/health").forStatusCode(200));

    private WebClient webClient;

    @BeforeEach
    void setUp() {
        String orderServiceUrl = String.format("http://%s:%d",
            environment.getServiceHost("order-service", 8082),
            environment.getServicePort("order-service", 8082));

        webClient = WebClient.builder().baseUrl(orderServiceUrl).build();
    }

    @Test
    @DisplayName("Complete order flow")
    void completeOrderFlow() {
        // Create order
        CreateOrderRequest request = CreateOrderRequest.builder()
            .userId("user-123")
            .items(List.of(new OrderItem("product-1", 2)))
            .build();

        Order createdOrder = webClient.post()
            .uri("/api/orders")
            .bodyValue(request)
            .retrieve()
            .bodyToMono(Order.class)
            .block();

        assertThat(createdOrder.getStatus()).isEqualTo(OrderStatus.PENDING);

        // Wait for saga completion
        await().atMost(30, TimeUnit.SECONDS)
            .pollInterval(1, TimeUnit.SECONDS)
            .untilAsserted(() -> {
                Order order = webClient.get()
                    .uri("/api/orders/" + createdOrder.getId())
                    .retrieve()
                    .bodyToMono(Order.class)
                    .block();

                assertThat(order.getStatus()).isEqualTo(OrderStatus.CONFIRMED);
            });
    }
}
```

---

## 8. Test Data Management

### 8.1. Test Data Builder

```java
public class OrderTestDataBuilder {

    private String id = "order-" + UUID.randomUUID().toString().substring(0, 8);
    private String userId = "user-123";
    private OrderStatus status = OrderStatus.PENDING;
    private List<OrderItem> items = new ArrayList<>();
    private BigDecimal totalAmount = BigDecimal.ZERO;

    public static OrderTestDataBuilder anOrder() {
        return new OrderTestDataBuilder();
    }

    public OrderTestDataBuilder withId(String id) {
        this.id = id;
        return this;
    }

    public OrderTestDataBuilder withUserId(String userId) {
        this.userId = userId;
        return this;
    }

    public OrderTestDataBuilder withStatus(OrderStatus status) {
        this.status = status;
        return this;
    }

    public OrderTestDataBuilder withItem(String productId, int quantity, BigDecimal price) {
        this.items.add(new OrderItem(productId, quantity, price));
        this.totalAmount = this.totalAmount.add(price.multiply(BigDecimal.valueOf(quantity)));
        return this;
    }

    public Order build() {
        return Order.builder()
            .id(id)
            .userId(userId)
            .status(status)
            .items(items)
            .totalAmount(totalAmount)
            .build();
    }
}

// Usage
@Test
void testOrder() {
    Order order = OrderTestDataBuilder.anOrder()
        .withUserId("user-456")
        .withItem("product-1", 2, BigDecimal.valueOf(10.00))
        .withItem("product-2", 1, BigDecimal.valueOf(25.00))
        .withStatus(OrderStatus.CONFIRMED)
        .build();

    assertThat(order.getTotalAmount()).isEqualByComparingTo("45.00");
}
```

---

## 9. CI/CD Pipeline Testing

```yaml
# .github/workflows/test.yml
name: Test Pipeline

on: [push, pull_request]

jobs:
  unit-tests:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Set up JDK 21
        uses: actions/setup-java@v4
        with:
          java-version: '21'
          distribution: 'temurin'
      - name: Run unit tests
        run: mvn test -Dtest=*UnitTest

  integration-tests:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Set up JDK 21
        uses: actions/setup-java@v4
        with:
          java-version: '21'
          distribution: 'temurin'
      - name: Run integration tests
        run: mvn test -Dtest=*IntegrationTest

  contract-tests:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Run contract tests
        run: mvn test -Dtest=*ContractTest
      - name: Publish pacts
        run: mvn pact:publish
        env:
          PACT_BROKER_URL: ${{ secrets.PACT_BROKER_URL }}

  e2e-tests:
    runs-on: ubuntu-latest
    needs: [unit-tests, integration-tests]
    steps:
      - uses: actions/checkout@v4
      - name: Build images
        run: docker-compose -f docker-compose.test.yml build
      - name: Run E2E tests
        run: mvn test -Dtest=*E2ETest
```

---

## 10. Key Takeaways

| Test Type | Scope | Speed | Confidence |
|-----------|-------|-------|------------|
| Unit | Class/Method | Fast | Low |
| Component | Service | Medium | Medium |
| Integration | Service + Deps | Slow | High |
| Contract | API Boundary | Fast | Medium |
| E2E | Full System | Very Slow | Very High |

```
Testing Best Practices:
1. Write tests first (TDD) when possible
2. Keep unit tests fast and isolated
3. Use Testcontainers for realistic integration tests
4. Contract tests for service boundaries
5. Minimal E2E tests for critical flows
6. Automate all tests in CI/CD
7. Monitor test coverage (aim for 80%+)
```

---

## Navigation

- [← Day 9: Security in Microservices](./day-09-microservices-security.md)
- [Day 11: Kafka Fundamentals →](./day-11-kafka-fundamentals.md)
