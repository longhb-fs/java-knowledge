# Day 6: Inter-Service Communication

## Mục tiêu
- Synchronous vs Asynchronous communication
- REST with RestTemplate & WebClient
- OpenFeign client
- gRPC basics

---

## 1. Communication Patterns

### 1.1. Overview

```
┌─────────────────────────────────────────────────────────────────┐
│            INTER-SERVICE COMMUNICATION PATTERNS                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   SYNCHRONOUS (Request-Response)                                │
│   ┌─────────┐         Request        ┌─────────┐               │
│   │Service A│ ───────────────────► │Service B│               │
│   │ (waits) │ ◄─────────────────── │         │               │
│   └─────────┘         Response       └─────────┘               │
│                                                                  │
│   Pros: Simple, immediate response                              │
│   Cons: Tight coupling, latency, availability dependency        │
│   Use: Read operations, real-time data needed                   │
│                                                                  │
│   ────────────────────────────────────────────────────────────  │
│                                                                  │
│   ASYNCHRONOUS (Event-Driven)                                   │
│   ┌─────────┐    Event    ┌─────────┐    Event    ┌─────────┐ │
│   │Service A│ ─────────► │ Message │ ─────────► │Service B│ │
│   │(continue)│            │  Queue  │            │(process)│ │
│   └─────────┘             └─────────┘            └─────────┘ │
│                                                                  │
│   Pros: Loose coupling, resilient, scalable                     │
│   Cons: Complex, eventual consistency                           │
│   Use: Write operations, long-running tasks                     │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 1.2. When to Use Which?

```
┌────────────────────┬────────────────────┬────────────────────┐
│     Scenario       │    Synchronous     │   Asynchronous     │
├────────────────────┼────────────────────┼────────────────────┤
│ Read user profile  │        ✓           │                    │
│ Process payment    │        ✓           │        ✓           │
│ Send notification  │                    │        ✓           │
│ Update inventory   │        ✓           │        ✓           │
│ Generate report    │                    │        ✓           │
│ Real-time search   │        ✓           │                    │
│ Audit logging      │                    │        ✓           │
│ Order placement    │        ✓           │        ✓           │
└────────────────────┴────────────────────┴────────────────────┘
```

---

## 2. RestTemplate

### 2.1. Basic Setup

```java
@Configuration
public class RestTemplateConfig {

    @Bean
    @LoadBalanced  // Enable service discovery
    public RestTemplate restTemplate(RestTemplateBuilder builder) {
        return builder
            .setConnectTimeout(Duration.ofSeconds(5))
            .setReadTimeout(Duration.ofSeconds(30))
            .build();
    }
}
```

### 2.2. CRUD Operations

```java
@Service
public class UserServiceClient {

    private final RestTemplate restTemplate;
    private static final String USER_SERVICE_URL = "http://user-service";

    public UserServiceClient(RestTemplate restTemplate) {
        this.restTemplate = restTemplate;
    }

    // GET - Single resource
    public User getUser(Long id) {
        return restTemplate.getForObject(
            USER_SERVICE_URL + "/api/users/{id}",
            User.class,
            id
        );
    }

    // GET - With response entity (access headers, status)
    public ResponseEntity<User> getUserWithDetails(Long id) {
        return restTemplate.getForEntity(
            USER_SERVICE_URL + "/api/users/{id}",
            User.class,
            id
        );
    }

    // GET - List
    public List<User> getAllUsers() {
        ResponseEntity<List<User>> response = restTemplate.exchange(
            USER_SERVICE_URL + "/api/users",
            HttpMethod.GET,
            null,
            new ParameterizedTypeReference<List<User>>() {}
        );
        return response.getBody();
    }

    // POST - Create
    public User createUser(CreateUserRequest request) {
        return restTemplate.postForObject(
            USER_SERVICE_URL + "/api/users",
            request,
            User.class
        );
    }

    // PUT - Update
    public void updateUser(Long id, UpdateUserRequest request) {
        restTemplate.put(
            USER_SERVICE_URL + "/api/users/{id}",
            request,
            id
        );
    }

    // DELETE
    public void deleteUser(Long id) {
        restTemplate.delete(USER_SERVICE_URL + "/api/users/{id}", id);
    }

    // With custom headers
    public User getUserWithAuth(Long id, String token) {
        HttpHeaders headers = new HttpHeaders();
        headers.setBearerAuth(token);
        headers.setContentType(MediaType.APPLICATION_JSON);

        HttpEntity<Void> entity = new HttpEntity<>(headers);

        ResponseEntity<User> response = restTemplate.exchange(
            USER_SERVICE_URL + "/api/users/{id}",
            HttpMethod.GET,
            entity,
            User.class,
            id
        );

        return response.getBody();
    }
}
```

### 2.3. Error Handling

```java
@Service
public class UserServiceClient {

    private final RestTemplate restTemplate;

    public User getUser(Long id) {
        try {
            return restTemplate.getForObject(
                "http://user-service/api/users/{id}",
                User.class,
                id
            );
        } catch (HttpClientErrorException.NotFound e) {
            throw new UserNotFoundException("User not found: " + id);
        } catch (HttpClientErrorException e) {
            throw new ServiceException("Client error: " + e.getStatusCode());
        } catch (HttpServerErrorException e) {
            throw new ServiceException("Server error: " + e.getStatusCode());
        } catch (ResourceAccessException e) {
            throw new ServiceUnavailableException("User service unavailable");
        }
    }
}

// Custom Error Handler
@Configuration
public class RestTemplateConfig {

    @Bean
    @LoadBalanced
    public RestTemplate restTemplate() {
        RestTemplate restTemplate = new RestTemplate();
        restTemplate.setErrorHandler(new CustomResponseErrorHandler());
        return restTemplate;
    }
}

public class CustomResponseErrorHandler implements ResponseErrorHandler {

    @Override
    public boolean hasError(ClientHttpResponse response) throws IOException {
        return response.getStatusCode().isError();
    }

    @Override
    public void handleError(ClientHttpResponse response) throws IOException {
        HttpStatusCode statusCode = response.getStatusCode();

        if (statusCode.is4xxClientError()) {
            throw new ClientException("Client error: " + statusCode);
        } else if (statusCode.is5xxServerError()) {
            throw new ServerException("Server error: " + statusCode);
        }
    }
}
```

---

## 3. WebClient (Reactive)

### 3.1. Setup

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-webflux</artifactId>
</dependency>
```

```java
@Configuration
public class WebClientConfig {

    @Bean
    @LoadBalanced
    public WebClient.Builder webClientBuilder() {
        return WebClient.builder()
            .defaultHeader(HttpHeaders.CONTENT_TYPE, MediaType.APPLICATION_JSON_VALUE);
    }

    @Bean
    public WebClient webClient(WebClient.Builder builder) {
        return builder
            .baseUrl("http://user-service")
            .filter(logRequest())
            .filter(logResponse())
            .build();
    }

    private ExchangeFilterFunction logRequest() {
        return ExchangeFilterFunction.ofRequestProcessor(request -> {
            log.debug("Request: {} {}", request.method(), request.url());
            return Mono.just(request);
        });
    }

    private ExchangeFilterFunction logResponse() {
        return ExchangeFilterFunction.ofResponseProcessor(response -> {
            log.debug("Response: {}", response.statusCode());
            return Mono.just(response);
        });
    }
}
```

### 3.2. CRUD Operations

```java
@Service
public class UserServiceClient {

    private final WebClient webClient;

    public UserServiceClient(WebClient.Builder builder) {
        this.webClient = builder.baseUrl("http://user-service").build();
    }

    // GET - Single (reactive)
    public Mono<User> getUser(Long id) {
        return webClient.get()
            .uri("/api/users/{id}", id)
            .retrieve()
            .bodyToMono(User.class);
    }

    // GET - Single (blocking)
    public User getUserBlocking(Long id) {
        return webClient.get()
            .uri("/api/users/{id}", id)
            .retrieve()
            .bodyToMono(User.class)
            .block();  // Block for result
    }

    // GET - List
    public Flux<User> getAllUsers() {
        return webClient.get()
            .uri("/api/users")
            .retrieve()
            .bodyToFlux(User.class);
    }

    // POST
    public Mono<User> createUser(CreateUserRequest request) {
        return webClient.post()
            .uri("/api/users")
            .bodyValue(request)
            .retrieve()
            .bodyToMono(User.class);
    }

    // PUT
    public Mono<User> updateUser(Long id, UpdateUserRequest request) {
        return webClient.put()
            .uri("/api/users/{id}", id)
            .bodyValue(request)
            .retrieve()
            .bodyToMono(User.class);
    }

    // DELETE
    public Mono<Void> deleteUser(Long id) {
        return webClient.delete()
            .uri("/api/users/{id}", id)
            .retrieve()
            .bodyToMono(Void.class);
    }

    // With custom headers
    public Mono<User> getUserWithAuth(Long id, String token) {
        return webClient.get()
            .uri("/api/users/{id}", id)
            .header(HttpHeaders.AUTHORIZATION, "Bearer " + token)
            .retrieve()
            .bodyToMono(User.class);
    }
}
```

### 3.3. Error Handling

```java
@Service
public class UserServiceClient {

    private final WebClient webClient;

    public Mono<User> getUser(Long id) {
        return webClient.get()
            .uri("/api/users/{id}", id)
            .retrieve()
            .onStatus(HttpStatusCode::is4xxClientError, response -> {
                if (response.statusCode() == HttpStatus.NOT_FOUND) {
                    return Mono.error(new UserNotFoundException("User not found: " + id));
                }
                return response.bodyToMono(String.class)
                    .flatMap(body -> Mono.error(new ClientException(body)));
            })
            .onStatus(HttpStatusCode::is5xxServerError, response ->
                Mono.error(new ServiceUnavailableException("User service error")))
            .bodyToMono(User.class)
            .timeout(Duration.ofSeconds(5))
            .retryWhen(Retry.backoff(3, Duration.ofMillis(500))
                .filter(ex -> ex instanceof ServiceUnavailableException));
    }

    // With fallback
    public Mono<User> getUserWithFallback(Long id) {
        return webClient.get()
            .uri("/api/users/{id}", id)
            .retrieve()
            .bodyToMono(User.class)
            .onErrorResume(WebClientResponseException.class, ex -> {
                log.error("Error calling user service: {}", ex.getMessage());
                return Mono.just(User.unknown());
            });
    }
}
```

---

## 4. OpenFeign

### 4.1. Setup

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-openfeign</artifactId>
</dependency>
```

```java
@SpringBootApplication
@EnableFeignClients
public class OrderServiceApplication {
    public static void main(String[] args) {
        SpringApplication.run(OrderServiceApplication.class, args);
    }
}
```

### 4.2. Feign Client Interface

```java
@FeignClient(
    name = "user-service",
    path = "/api/users",
    configuration = FeignConfig.class,
    fallbackFactory = UserClientFallbackFactory.class
)
public interface UserClient {

    @GetMapping("/{id}")
    User getUser(@PathVariable("id") Long id);

    @GetMapping
    List<User> getAllUsers();

    @GetMapping
    List<User> getUsersByIds(@RequestParam("ids") List<Long> ids);

    @PostMapping
    User createUser(@RequestBody CreateUserRequest request);

    @PutMapping("/{id}")
    User updateUser(@PathVariable("id") Long id, @RequestBody UpdateUserRequest request);

    @DeleteMapping("/{id}")
    void deleteUser(@PathVariable("id") Long id);

    // With headers
    @GetMapping("/{id}")
    User getUserWithAuth(
        @PathVariable("id") Long id,
        @RequestHeader("Authorization") String token
    );

    // Query parameters
    @GetMapping("/search")
    Page<User> searchUsers(
        @RequestParam("name") String name,
        @RequestParam(value = "page", defaultValue = "0") int page,
        @RequestParam(value = "size", defaultValue = "20") int size
    );
}
```

### 4.3. Feign Configuration

```java
@Configuration
public class FeignConfig {

    @Bean
    public Logger.Level feignLoggerLevel() {
        return Logger.Level.FULL;  // NONE, BASIC, HEADERS, FULL
    }

    @Bean
    public ErrorDecoder errorDecoder() {
        return new CustomErrorDecoder();
    }

    @Bean
    public Request.Options requestOptions() {
        return new Request.Options(
            5, TimeUnit.SECONDS,   // Connect timeout
            30, TimeUnit.SECONDS,  // Read timeout
            true                   // Follow redirects
        );
    }

    @Bean
    public Retryer retryer() {
        return new Retryer.Default(100, 1000, 3);
    }

    // Request interceptor for auth
    @Bean
    public RequestInterceptor authInterceptor() {
        return template -> {
            // Add common headers
            template.header("X-Request-Source", "order-service");
            // Add auth token from context
            String token = SecurityContextHolder.getContext()
                .getAuthentication().getCredentials().toString();
            template.header("Authorization", "Bearer " + token);
        };
    }
}

// Custom Error Decoder
public class CustomErrorDecoder implements ErrorDecoder {

    @Override
    public Exception decode(String methodKey, Response response) {
        return switch (response.status()) {
            case 400 -> new BadRequestException("Bad request");
            case 404 -> new NotFoundException("Resource not found");
            case 500 -> new ServiceException("Internal server error");
            default -> new Exception("Generic error");
        };
    }
}
```

### 4.4. Fallback & Circuit Breaker

```java
// Fallback Factory
@Component
public class UserClientFallbackFactory implements FallbackFactory<UserClient> {

    private static final Logger log = LoggerFactory.getLogger(UserClientFallbackFactory.class);

    @Override
    public UserClient create(Throwable cause) {
        log.error("User service fallback triggered: {}", cause.getMessage());

        return new UserClient() {
            @Override
            public User getUser(Long id) {
                return User.builder()
                    .id(id)
                    .name("Unknown User")
                    .status("UNAVAILABLE")
                    .build();
            }

            @Override
            public List<User> getAllUsers() {
                return Collections.emptyList();
            }

            @Override
            public User createUser(CreateUserRequest request) {
                throw new ServiceUnavailableException("User service unavailable");
            }

            // Implement other methods...
        };
    }
}

// Enable Circuit Breaker
// application.yml
spring:
  cloud:
    openfeign:
      circuitbreaker:
        enabled: true

resilience4j:
  circuitbreaker:
    instances:
      user-service:
        slidingWindowSize: 10
        failureRateThreshold: 50
        waitDurationInOpenState: 10000
```

---

## 5. gRPC

### 5.1. Setup

```xml
<!-- pom.xml -->
<dependencies>
    <dependency>
        <groupId>net.devh</groupId>
        <artifactId>grpc-spring-boot-starter</artifactId>
        <version>2.15.0.RELEASE</version>
    </dependency>
</dependencies>

<build>
    <extensions>
        <extension>
            <groupId>kr.motd.maven</groupId>
            <artifactId>os-maven-plugin</artifactId>
            <version>1.7.1</version>
        </extension>
    </extensions>
    <plugins>
        <plugin>
            <groupId>org.xolstice.maven.plugins</groupId>
            <artifactId>protobuf-maven-plugin</artifactId>
            <version>0.6.1</version>
            <configuration>
                <protocArtifact>com.google.protobuf:protoc:3.24.0:exe:${os.detected.classifier}</protocArtifact>
                <pluginId>grpc-java</pluginId>
                <pluginArtifact>io.grpc:protoc-gen-grpc-java:1.58.0:exe:${os.detected.classifier}</pluginArtifact>
            </configuration>
            <executions>
                <execution>
                    <goals>
                        <goal>compile</goal>
                        <goal>compile-custom</goal>
                    </goals>
                </execution>
            </executions>
        </plugin>
    </plugins>
</build>
```

### 5.2. Proto Definition

```protobuf
// src/main/proto/user.proto
syntax = "proto3";

option java_multiple_files = true;
option java_package = "com.example.grpc.user";

package user;

service UserService {
    rpc GetUser(GetUserRequest) returns (UserResponse);
    rpc CreateUser(CreateUserRequest) returns (UserResponse);
    rpc GetUsers(GetUsersRequest) returns (stream UserResponse);
}

message GetUserRequest {
    int64 id = 1;
}

message CreateUserRequest {
    string name = 1;
    string email = 2;
}

message GetUsersRequest {
    repeated int64 ids = 1;
}

message UserResponse {
    int64 id = 1;
    string name = 2;
    string email = 3;
    string status = 4;
}
```

### 5.3. gRPC Server

```java
@GrpcService
public class UserGrpcService extends UserServiceGrpc.UserServiceImplBase {

    private final UserRepository userRepository;

    @Override
    public void getUser(GetUserRequest request, StreamObserver<UserResponse> responseObserver) {
        User user = userRepository.findById(request.getId())
            .orElseThrow(() -> new StatusRuntimeException(
                Status.NOT_FOUND.withDescription("User not found")));

        UserResponse response = UserResponse.newBuilder()
            .setId(user.getId())
            .setName(user.getName())
            .setEmail(user.getEmail())
            .setStatus(user.getStatus())
            .build();

        responseObserver.onNext(response);
        responseObserver.onCompleted();
    }

    @Override
    public void createUser(CreateUserRequest request, StreamObserver<UserResponse> responseObserver) {
        User user = new User();
        user.setName(request.getName());
        user.setEmail(request.getEmail());
        user = userRepository.save(user);

        UserResponse response = UserResponse.newBuilder()
            .setId(user.getId())
            .setName(user.getName())
            .setEmail(user.getEmail())
            .build();

        responseObserver.onNext(response);
        responseObserver.onCompleted();
    }

    @Override
    public void getUsers(GetUsersRequest request, StreamObserver<UserResponse> responseObserver) {
        // Server streaming
        request.getIdsList().forEach(id -> {
            userRepository.findById(id).ifPresent(user -> {
                UserResponse response = UserResponse.newBuilder()
                    .setId(user.getId())
                    .setName(user.getName())
                    .build();
                responseObserver.onNext(response);
            });
        });
        responseObserver.onCompleted();
    }
}
```

### 5.4. gRPC Client

```java
@Service
public class UserGrpcClient {

    @GrpcClient("user-service")
    private UserServiceGrpc.UserServiceBlockingStub userServiceStub;

    public UserResponse getUser(Long id) {
        GetUserRequest request = GetUserRequest.newBuilder()
            .setId(id)
            .build();

        try {
            return userServiceStub.getUser(request);
        } catch (StatusRuntimeException e) {
            if (e.getStatus().getCode() == Status.Code.NOT_FOUND) {
                throw new UserNotFoundException("User not found: " + id);
            }
            throw new ServiceException("gRPC error: " + e.getMessage());
        }
    }

    public List<UserResponse> getUsers(List<Long> ids) {
        GetUsersRequest request = GetUsersRequest.newBuilder()
            .addAllIds(ids)
            .build();

        List<UserResponse> users = new ArrayList<>();
        userServiceStub.getUsers(request).forEachRemaining(users::add);
        return users;
    }
}
```

```yaml
# application.yml
grpc:
  client:
    user-service:
      address: 'discovery:///user-service'  # Using Eureka
      negotiationType: PLAINTEXT
      enableKeepAlive: true
      keepAliveTime: 30s
      keepAliveTimeout: 5s
```

---

## 6. REST vs gRPC Comparison

```
┌──────────────────┬────────────────────┬────────────────────┐
│     Feature      │       REST         │       gRPC         │
├──────────────────┼────────────────────┼────────────────────┤
│ Protocol         │ HTTP/1.1, HTTP/2   │ HTTP/2             │
│ Data Format      │ JSON, XML          │ Protobuf (binary)  │
│ Schema           │ OpenAPI (optional) │ Proto (required)   │
│ Streaming        │ Limited            │ Bi-directional     │
│ Performance      │ Good               │ Excellent          │
│ Browser Support  │ Native             │ Requires proxy     │
│ Learning Curve   │ Low                │ Medium             │
│ Tooling          │ Mature             │ Growing            │
│ Use Case         │ Public APIs        │ Internal services  │
└──────────────────┴────────────────────┴────────────────────┘
```

---

## 7. Best Practices

### 7.1. Retry Pattern

```java
@Configuration
public class RetryConfig {

    @Bean
    public RetryTemplate retryTemplate() {
        RetryTemplate retryTemplate = new RetryTemplate();

        // Exponential backoff
        ExponentialBackOffPolicy backOffPolicy = new ExponentialBackOffPolicy();
        backOffPolicy.setInitialInterval(500);
        backOffPolicy.setMultiplier(2.0);
        backOffPolicy.setMaxInterval(5000);
        retryTemplate.setBackOffPolicy(backOffPolicy);

        // Retry only on specific exceptions
        SimpleRetryPolicy retryPolicy = new SimpleRetryPolicy(3,
            Map.of(
                ServiceUnavailableException.class, true,
                ResourceAccessException.class, true
            ));
        retryTemplate.setRetryPolicy(retryPolicy);

        return retryTemplate;
    }
}

// Usage
@Service
public class UserServiceClient {

    private final RestTemplate restTemplate;
    private final RetryTemplate retryTemplate;

    public User getUser(Long id) {
        return retryTemplate.execute(context -> {
            log.info("Attempt {} to call user service", context.getRetryCount() + 1);
            return restTemplate.getForObject(
                "http://user-service/api/users/{id}",
                User.class,
                id
            );
        });
    }
}
```

### 7.2. Circuit Breaker Pattern

```java
@Service
public class UserServiceClient {

    private final WebClient webClient;
    private final CircuitBreakerFactory circuitBreakerFactory;

    public Mono<User> getUser(Long id) {
        CircuitBreaker circuitBreaker = circuitBreakerFactory.create("user-service");

        return webClient.get()
            .uri("/api/users/{id}", id)
            .retrieve()
            .bodyToMono(User.class)
            .transform(CircuitBreakerOperator.of(circuitBreaker))
            .onErrorResume(CallNotPermittedException.class, ex ->
                Mono.just(User.fallback()));
    }
}
```

### 7.3. Timeout Configuration

```yaml
# RestTemplate
spring:
  rest:
    connection-timeout: 5000
    read-timeout: 30000

# Feign
feign:
  client:
    config:
      default:
        connectTimeout: 5000
        readTimeout: 30000
      user-service:
        connectTimeout: 3000
        readTimeout: 10000

# WebClient configured in code
```

---

## 8. Bài tập thực hành

### Bài 1: RestTemplate
Tạo Order Service gọi User Service và Product Service via RestTemplate.

### Bài 2: OpenFeign
Chuyển đổi sang OpenFeign, implement fallback.

### Bài 3: WebClient
Implement reactive version với WebClient, parallel calls.

### Bài 4: gRPC
Setup gRPC communication giữa 2 services.

---

## 9. Key Takeaways

| Method | Best For | Pros | Cons |
|--------|----------|------|------|
| RestTemplate | Simple sync calls | Easy, well-known | Blocking, deprecated |
| WebClient | Reactive apps | Non-blocking, modern | Learning curve |
| OpenFeign | Clean interfaces | Declarative, elegant | Limited flexibility |
| gRPC | High-performance | Fast, type-safe | Browser support |

```
Communication Flow:
1. Service A needs data from Service B
2. Load Balancer selects instance
3. Circuit Breaker checks state
4. Request sent with timeout
5. Retry on transient failure
6. Fallback on circuit open
7. Response processed
```

---

## Navigation

- [← Day 5: Configuration Management](./day-05-config-management.md)
- [Day 7: Resilience Patterns →](./day-07-resilience-patterns.md)
