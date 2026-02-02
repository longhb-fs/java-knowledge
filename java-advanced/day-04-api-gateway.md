# Day 4: API Gateway

## Mục tiêu
- API Gateway patterns
- Spring Cloud Gateway
- Routing & Filters
- Rate limiting & Security

---

## 1. API Gateway Pattern

### 1.1. Why API Gateway?

```
┌─────────────────────────────────────────────────────────────────┐
│                WITHOUT API GATEWAY                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Mobile App ──────────────────────►  User Service :8081        │
│        │                                                         │
│        ├───────────────────────────►  Order Service :8082       │
│        │                                                         │
│        ├───────────────────────────►  Product Service :8083     │
│        │                                                         │
│        └───────────────────────────►  Payment Service :8084     │
│                                                                  │
│   Problems:                                                      │
│   ✗ Client needs to know all service endpoints                  │
│   ✗ CORS issues                                                 │
│   ✗ No centralized authentication                               │
│   ✗ No rate limiting                                            │
│   ✗ Different protocols per service                             │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│                WITH API GATEWAY                                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Mobile App ─────► API Gateway :8080 ─────► User Service       │
│                          │                                       │
│                          ├─────────────────► Order Service      │
│                          │                                       │
│                          ├─────────────────► Product Service    │
│                          │                                       │
│                          └─────────────────► Payment Service    │
│                                                                  │
│   Benefits:                                                      │
│   ✓ Single entry point                                          │
│   ✓ Centralized auth, logging, monitoring                       │
│   ✓ Rate limiting, circuit breaking                             │
│   ✓ Protocol translation                                        │
│   ✓ Request aggregation                                         │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 1.2. Gateway Responsibilities

```
┌─────────────────────────────────────────────────────────────────┐
│                    API GATEWAY FEATURES                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Request Flow:                                                  │
│                                                                  │
│   Client Request                                                 │
│        │                                                         │
│        ▼                                                         │
│   ┌──────────────┐                                              │
│   │   SSL/TLS    │  ◄── HTTPS termination                       │
│   └──────┬───────┘                                              │
│          ▼                                                       │
│   ┌──────────────┐                                              │
│   │Authentication│  ◄── JWT validation, OAuth2                  │
│   └──────┬───────┘                                              │
│          ▼                                                       │
│   ┌──────────────┐                                              │
│   │ Rate Limiting│  ◄── Throttle requests                       │
│   └──────┬───────┘                                              │
│          ▼                                                       │
│   ┌──────────────┐                                              │
│   │   Routing    │  ◄── Path-based, header-based                │
│   └──────┬───────┘                                              │
│          ▼                                                       │
│   ┌──────────────┐                                              │
│   │Load Balancing│  ◄── Round-robin, weighted                   │
│   └──────┬───────┘                                              │
│          ▼                                                       │
│   ┌──────────────┐                                              │
│   │Circuit Break │  ◄── Fallback on failure                     │
│   └──────┬───────┘                                              │
│          ▼                                                       │
│   Backend Service                                                │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## 2. Spring Cloud Gateway

### 2.1. Setup

```xml
<!-- pom.xml -->
<dependencies>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-gateway</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-actuator</artifactId>
    </dependency>
</dependencies>
```

```java
@SpringBootApplication
@EnableDiscoveryClient
public class ApiGatewayApplication {
    public static void main(String[] args) {
        SpringApplication.run(ApiGatewayApplication.class, args);
    }
}
```

### 2.2. Route Configuration (YAML)

```yaml
server:
  port: 8080

spring:
  application:
    name: api-gateway
  cloud:
    gateway:
      # Enable service discovery routes
      discovery:
        locator:
          enabled: true
          lower-case-service-id: true

      # Define routes
      routes:
        # User Service
        - id: user-service
          uri: lb://user-service
          predicates:
            - Path=/api/users/**
          filters:
            - StripPrefix=1
            - name: CircuitBreaker
              args:
                name: userServiceCB
                fallbackUri: forward:/fallback/users

        # Order Service
        - id: order-service
          uri: lb://order-service
          predicates:
            - Path=/api/orders/**
          filters:
            - StripPrefix=1

        # Product Service with weighted routing
        - id: product-service-v1
          uri: lb://product-service
          predicates:
            - Path=/api/products/**
            - Weight=group1, 80
          filters:
            - StripPrefix=1

        - id: product-service-v2
          uri: lb://product-service-v2
          predicates:
            - Path=/api/products/**
            - Weight=group1, 20
          filters:
            - StripPrefix=1

      # Global filters
      default-filters:
        - AddResponseHeader=X-Gateway-Version, 1.0.0
        - name: RequestRateLimiter
          args:
            redis-rate-limiter.replenishRate: 10
            redis-rate-limiter.burstCapacity: 20

eureka:
  client:
    service-url:
      defaultZone: http://localhost:8761/eureka/
```

### 2.3. Route Configuration (Java)

```java
@Configuration
public class GatewayConfig {

    @Bean
    public RouteLocator customRouteLocator(RouteLocatorBuilder builder) {
        return builder.routes()
            // Simple route
            .route("user-service", r -> r
                .path("/api/users/**")
                .filters(f -> f
                    .stripPrefix(1)
                    .addRequestHeader("X-Request-Source", "gateway"))
                .uri("lb://user-service"))

            // Route with predicates
            .route("order-service", r -> r
                .path("/api/orders/**")
                .and()
                .method(HttpMethod.GET, HttpMethod.POST)
                .and()
                .header("Authorization", "Bearer.*")
                .filters(f -> f
                    .stripPrefix(1)
                    .retry(config -> config
                        .setRetries(3)
                        .setStatuses(HttpStatus.INTERNAL_SERVER_ERROR)))
                .uri("lb://order-service"))

            // Route with custom filter
            .route("product-service", r -> r
                .path("/api/products/**")
                .filters(f -> f
                    .stripPrefix(1)
                    .filter(new LoggingGatewayFilter()))
                .uri("lb://product-service"))

            // Redirect route
            .route("legacy-redirect", r -> r
                .path("/old-api/**")
                .filters(f -> f
                    .rewritePath("/old-api/(?<segment>.*)", "/api/${segment}"))
                .uri("lb://api-gateway"))

            .build();
    }
}
```

---

## 3. Predicates

### 3.1. Built-in Predicates

```yaml
spring:
  cloud:
    gateway:
      routes:
        - id: example-route
          uri: lb://example-service
          predicates:
            # Path matching
            - Path=/api/**,/v1/**

            # HTTP Method
            - Method=GET,POST,PUT

            # Header matching
            - Header=X-Request-Id, \d+

            # Cookie matching
            - Cookie=sessionId, .+

            # Query parameter
            - Query=name, john.*

            # Host matching
            - Host=**.example.com

            # Time-based
            - After=2024-01-01T00:00:00+07:00[Asia/Ho_Chi_Minh]
            - Before=2025-01-01T00:00:00+07:00[Asia/Ho_Chi_Minh]
            - Between=2024-01-01T00:00:00+07:00,2024-12-31T23:59:59+07:00

            # Remote address
            - RemoteAddr=192.168.1.0/24

            # Weight for canary deployment
            - Weight=group1, 80
```

### 3.2. Custom Predicate

```java
@Component
public class ApiKeyRoutePredicateFactory
    extends AbstractRoutePredicateFactory<ApiKeyRoutePredicateFactory.Config> {

    public ApiKeyRoutePredicateFactory() {
        super(Config.class);
    }

    @Override
    public Predicate<ServerWebExchange> apply(Config config) {
        return exchange -> {
            String apiKey = exchange.getRequest()
                .getHeaders()
                .getFirst("X-API-Key");

            return config.getValidKeys().contains(apiKey);
        };
    }

    @Override
    public List<String> shortcutFieldOrder() {
        return List.of("validKeys");
    }

    @Validated
    public static class Config {
        @NotEmpty
        private List<String> validKeys;

        public List<String> getValidKeys() {
            return validKeys;
        }

        public void setValidKeys(List<String> validKeys) {
            this.validKeys = validKeys;
        }
    }
}

// Usage in YAML
// predicates:
//   - ApiKey=key1,key2,key3
```

---

## 4. Filters

### 4.1. Built-in Filters

```yaml
spring:
  cloud:
    gateway:
      routes:
        - id: filter-example
          uri: lb://example-service
          predicates:
            - Path=/api/**
          filters:
            # Path manipulation
            - StripPrefix=1
            - PrefixPath=/v1
            - RewritePath=/api/(?<segment>.*), /${segment}

            # Headers
            - AddRequestHeader=X-Request-Source, Gateway
            - AddResponseHeader=X-Response-Time, ${responseTime}
            - RemoveRequestHeader=Cookie
            - RemoveResponseHeader=Server
            - SetRequestHeader=Host, example.com

            # Query parameters
            - AddRequestParameter=debug, true
            - RemoveRequestParameter=secret

            # Response modification
            - SetStatus=200
            - DedupeResponseHeader=Access-Control-Allow-Origin

            # Retry
            - name: Retry
              args:
                retries: 3
                statuses: BAD_GATEWAY,GATEWAY_TIMEOUT
                methods: GET
                backoff:
                  firstBackoff: 50ms
                  maxBackoff: 500ms
                  factor: 2

            # Circuit Breaker
            - name: CircuitBreaker
              args:
                name: myCircuitBreaker
                fallbackUri: forward:/fallback

            # Request size limit
            - name: RequestSize
              args:
                maxSize: 5000000  # 5MB
```

### 4.2. Custom Global Filter

```java
@Component
@Order(-1)  // Execute first
public class LoggingGlobalFilter implements GlobalFilter {

    private static final Logger log = LoggerFactory.getLogger(LoggingGlobalFilter.class);

    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        long startTime = System.currentTimeMillis();
        String requestId = UUID.randomUUID().toString();

        log.info("[{}] Request: {} {}",
            requestId,
            exchange.getRequest().getMethod(),
            exchange.getRequest().getURI());

        // Add request ID header
        exchange.getRequest().mutate()
            .header("X-Request-Id", requestId);

        return chain.filter(exchange)
            .then(Mono.fromRunnable(() -> {
                long duration = System.currentTimeMillis() - startTime;
                log.info("[{}] Response: {} in {}ms",
                    requestId,
                    exchange.getResponse().getStatusCode(),
                    duration);
            }));
    }
}
```

### 4.3. Custom Gateway Filter Factory

```java
@Component
public class AuthenticationGatewayFilterFactory
    extends AbstractGatewayFilterFactory<AuthenticationGatewayFilterFactory.Config> {

    private final JwtService jwtService;

    public AuthenticationGatewayFilterFactory(JwtService jwtService) {
        super(Config.class);
        this.jwtService = jwtService;
    }

    @Override
    public GatewayFilter apply(Config config) {
        return (exchange, chain) -> {
            String authHeader = exchange.getRequest()
                .getHeaders()
                .getFirst(HttpHeaders.AUTHORIZATION);

            if (authHeader == null || !authHeader.startsWith("Bearer ")) {
                exchange.getResponse().setStatusCode(HttpStatus.UNAUTHORIZED);
                return exchange.getResponse().setComplete();
            }

            String token = authHeader.substring(7);

            try {
                Claims claims = jwtService.validateToken(token);

                // Add user info to headers
                ServerHttpRequest request = exchange.getRequest().mutate()
                    .header("X-User-Id", claims.getSubject())
                    .header("X-User-Roles", claims.get("roles", String.class))
                    .build();

                return chain.filter(exchange.mutate().request(request).build());

            } catch (Exception e) {
                exchange.getResponse().setStatusCode(HttpStatus.UNAUTHORIZED);
                return exchange.getResponse().setComplete();
            }
        };
    }

    public static class Config {
        private List<String> excludePaths = new ArrayList<>();

        public List<String> getExcludePaths() {
            return excludePaths;
        }

        public void setExcludePaths(List<String> excludePaths) {
            this.excludePaths = excludePaths;
        }
    }
}
```

---

## 5. Rate Limiting

### 5.1. Redis Rate Limiter

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-redis-reactive</artifactId>
</dependency>
```

```yaml
spring:
  data:
    redis:
      host: localhost
      port: 6379

  cloud:
    gateway:
      routes:
        - id: rate-limited-route
          uri: lb://api-service
          predicates:
            - Path=/api/**
          filters:
            - name: RequestRateLimiter
              args:
                redis-rate-limiter.replenishRate: 10
                redis-rate-limiter.burstCapacity: 20
                redis-rate-limiter.requestedTokens: 1
                key-resolver: "#{@userKeyResolver}"
```

```java
@Configuration
public class RateLimiterConfig {

    // Rate limit by user ID
    @Bean
    public KeyResolver userKeyResolver() {
        return exchange -> {
            String userId = exchange.getRequest()
                .getHeaders()
                .getFirst("X-User-Id");
            return Mono.justOrEmpty(userId)
                .defaultIfEmpty("anonymous");
        };
    }

    // Rate limit by IP address
    @Bean
    public KeyResolver ipKeyResolver() {
        return exchange -> Mono.just(
            Objects.requireNonNull(
                exchange.getRequest().getRemoteAddress()
            ).getAddress().getHostAddress()
        );
    }

    // Rate limit by API key
    @Bean
    public KeyResolver apiKeyResolver() {
        return exchange -> {
            String apiKey = exchange.getRequest()
                .getHeaders()
                .getFirst("X-API-Key");
            return Mono.justOrEmpty(apiKey)
                .defaultIfEmpty("no-api-key");
        };
    }
}
```

### 5.2. Custom Rate Limiter

```java
@Component
public class CustomRateLimiter implements RateLimiter<CustomRateLimiter.Config> {

    private final ReactiveRedisTemplate<String, String> redisTemplate;

    public CustomRateLimiter(ReactiveRedisTemplate<String, String> redisTemplate) {
        this.redisTemplate = redisTemplate;
    }

    @Override
    public Mono<Response> isAllowed(String routeId, String id) {
        String key = "rate_limit:" + routeId + ":" + id;

        return redisTemplate.opsForValue()
            .increment(key)
            .flatMap(count -> {
                if (count == 1) {
                    return redisTemplate.expire(key, Duration.ofMinutes(1))
                        .thenReturn(count);
                }
                return Mono.just(count);
            })
            .map(count -> {
                boolean allowed = count <= 100;  // 100 requests per minute
                return new Response(allowed, Map.of(
                    "X-RateLimit-Remaining", String.valueOf(100 - count),
                    "X-RateLimit-Limit", "100",
                    "X-RateLimit-Reset", "60"
                ));
            });
    }

    @Override
    public Class<Config> getConfigClass() {
        return Config.class;
    }

    @Override
    public Config newConfig() {
        return new Config();
    }

    public static class Config {
        private int requestsPerMinute = 100;

        public int getRequestsPerMinute() {
            return requestsPerMinute;
        }

        public void setRequestsPerMinute(int requestsPerMinute) {
            this.requestsPerMinute = requestsPerMinute;
        }
    }
}
```

---

## 6. Security

### 6.1. JWT Authentication

```java
@Component
public class JwtAuthenticationFilter implements GlobalFilter, Ordered {

    private final JwtService jwtService;
    private final List<String> publicPaths = List.of(
        "/api/auth/login",
        "/api/auth/register",
        "/actuator/health"
    );

    public JwtAuthenticationFilter(JwtService jwtService) {
        this.jwtService = jwtService;
    }

    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        String path = exchange.getRequest().getPath().value();

        // Skip public paths
        if (publicPaths.stream().anyMatch(path::startsWith)) {
            return chain.filter(exchange);
        }

        String authHeader = exchange.getRequest()
            .getHeaders()
            .getFirst(HttpHeaders.AUTHORIZATION);

        if (authHeader == null || !authHeader.startsWith("Bearer ")) {
            return unauthorized(exchange, "Missing or invalid Authorization header");
        }

        String token = authHeader.substring(7);

        try {
            Claims claims = jwtService.validateToken(token);

            ServerHttpRequest mutatedRequest = exchange.getRequest().mutate()
                .header("X-User-Id", claims.getSubject())
                .header("X-User-Email", claims.get("email", String.class))
                .header("X-User-Roles", String.join(",",
                    claims.get("roles", List.class)))
                .build();

            return chain.filter(exchange.mutate().request(mutatedRequest).build());

        } catch (ExpiredJwtException e) {
            return unauthorized(exchange, "Token expired");
        } catch (Exception e) {
            return unauthorized(exchange, "Invalid token");
        }
    }

    private Mono<Void> unauthorized(ServerWebExchange exchange, String message) {
        exchange.getResponse().setStatusCode(HttpStatus.UNAUTHORIZED);
        exchange.getResponse().getHeaders().add(
            HttpHeaders.CONTENT_TYPE, MediaType.APPLICATION_JSON_VALUE);

        String body = String.format("{\"error\": \"%s\"}", message);
        DataBuffer buffer = exchange.getResponse()
            .bufferFactory()
            .wrap(body.getBytes(StandardCharsets.UTF_8));

        return exchange.getResponse().writeWith(Mono.just(buffer));
    }

    @Override
    public int getOrder() {
        return -100;  // Run early
    }
}
```

### 6.2. CORS Configuration

```java
@Configuration
public class CorsConfig {

    @Bean
    public CorsWebFilter corsWebFilter() {
        CorsConfiguration corsConfig = new CorsConfiguration();
        corsConfig.setAllowedOriginPatterns(List.of(
            "http://localhost:*",
            "https://*.example.com"
        ));
        corsConfig.setAllowedMethods(List.of("GET", "POST", "PUT", "DELETE", "OPTIONS"));
        corsConfig.setAllowedHeaders(List.of("*"));
        corsConfig.setAllowCredentials(true);
        corsConfig.setMaxAge(3600L);

        UrlBasedCorsConfigurationSource source = new UrlBasedCorsConfigurationSource();
        source.registerCorsConfiguration("/**", corsConfig);

        return new CorsWebFilter(source);
    }
}
```

### 6.3. IP Whitelist Filter

```java
@Component
public class IpWhitelistFilter implements GlobalFilter, Ordered {

    private final Set<String> whitelist = Set.of(
        "127.0.0.1",
        "192.168.1.0/24",
        "10.0.0.0/8"
    );

    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        String clientIp = Objects.requireNonNull(
            exchange.getRequest().getRemoteAddress()
        ).getAddress().getHostAddress();

        if (!isAllowed(clientIp)) {
            exchange.getResponse().setStatusCode(HttpStatus.FORBIDDEN);
            return exchange.getResponse().setComplete();
        }

        return chain.filter(exchange);
    }

    private boolean isAllowed(String ip) {
        // Implement CIDR matching logic
        return whitelist.stream()
            .anyMatch(allowed -> matchesCidr(ip, allowed));
    }

    private boolean matchesCidr(String ip, String cidr) {
        // Simplified - use proper CIDR library in production
        if (!cidr.contains("/")) {
            return ip.equals(cidr);
        }
        // Implement CIDR matching
        return true;
    }

    @Override
    public int getOrder() {
        return -200;  // Run very early
    }
}
```

---

## 7. Circuit Breaker & Fallback

### 7.1. Resilience4j Circuit Breaker

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-circuitbreaker-reactor-resilience4j</artifactId>
</dependency>
```

```yaml
spring:
  cloud:
    gateway:
      routes:
        - id: user-service
          uri: lb://user-service
          predicates:
            - Path=/api/users/**
          filters:
            - name: CircuitBreaker
              args:
                name: userServiceCB
                fallbackUri: forward:/fallback/users

resilience4j:
  circuitbreaker:
    configs:
      default:
        slidingWindowSize: 10
        failureRateThreshold: 50
        waitDurationInOpenState: 10000
        permittedNumberOfCallsInHalfOpenState: 3
        automaticTransitionFromOpenToHalfOpenEnabled: true
    instances:
      userServiceCB:
        baseConfig: default
        slidingWindowSize: 5
        failureRateThreshold: 60

  timelimiter:
    configs:
      default:
        timeoutDuration: 5s
```

### 7.2. Fallback Controller

```java
@RestController
@RequestMapping("/fallback")
public class FallbackController {

    private static final Logger log = LoggerFactory.getLogger(FallbackController.class);

    @GetMapping("/users")
    public Mono<ResponseEntity<Map<String, Object>>> usersFallback() {
        log.warn("User service fallback triggered");
        return Mono.just(ResponseEntity
            .status(HttpStatus.SERVICE_UNAVAILABLE)
            .body(Map.of(
                "message", "User service is currently unavailable",
                "timestamp", Instant.now().toString()
            )));
    }

    @GetMapping("/orders")
    public Mono<ResponseEntity<Map<String, Object>>> ordersFallback() {
        log.warn("Order service fallback triggered");
        return Mono.just(ResponseEntity
            .status(HttpStatus.SERVICE_UNAVAILABLE)
            .body(Map.of(
                "message", "Order service is currently unavailable",
                "timestamp", Instant.now().toString()
            )));
    }

    @RequestMapping("/**")
    public Mono<ResponseEntity<Map<String, Object>>> defaultFallback() {
        return Mono.just(ResponseEntity
            .status(HttpStatus.SERVICE_UNAVAILABLE)
            .body(Map.of(
                "message", "Service temporarily unavailable",
                "timestamp", Instant.now().toString()
            )));
    }
}
```

---

## 8. Monitoring & Logging

### 8.1. Actuator Endpoints

```yaml
management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics,gateway
  endpoint:
    gateway:
      enabled: true
    health:
      show-details: always

# Endpoints:
# GET /actuator/gateway/routes - List all routes
# GET /actuator/gateway/routes/{id} - Get specific route
# POST /actuator/gateway/refresh - Refresh routes
# GET /actuator/gateway/globalfilters - List global filters
# GET /actuator/gateway/routefilters - List route filters
```

### 8.2. Distributed Tracing

```xml
<dependency>
    <groupId>io.micrometer</groupId>
    <artifactId>micrometer-tracing-bridge-brave</artifactId>
</dependency>
<dependency>
    <groupId>io.zipkin.reporter2</groupId>
    <artifactId>zipkin-reporter-brave</artifactId>
</dependency>
```

```yaml
management:
  tracing:
    sampling:
      probability: 1.0

spring:
  zipkin:
    base-url: http://localhost:9411
```

### 8.3. Request/Response Logging

```java
@Component
public class RequestResponseLoggingFilter implements GlobalFilter, Ordered {

    private static final Logger log = LoggerFactory.getLogger(RequestResponseLoggingFilter.class);

    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        ServerHttpRequest request = exchange.getRequest();

        // Log request
        log.info("Request: {} {} from {}",
            request.getMethod(),
            request.getURI(),
            request.getRemoteAddress());

        // Log headers (be careful with sensitive data)
        request.getHeaders().forEach((name, values) -> {
            if (!name.equalsIgnoreCase("Authorization")) {
                log.debug("Header: {} = {}", name, values);
            }
        });

        // Wrap response for logging
        ServerHttpResponseDecorator decoratedResponse =
            new ServerHttpResponseDecorator(exchange.getResponse()) {
                @Override
                public Mono<Void> writeWith(Publisher<? extends DataBuffer> body) {
                    log.info("Response: {} for {} {}",
                        getStatusCode(),
                        request.getMethod(),
                        request.getURI());
                    return super.writeWith(body);
                }
            };

        return chain.filter(exchange.mutate()
            .response(decoratedResponse)
            .build());
    }

    @Override
    public int getOrder() {
        return Ordered.HIGHEST_PRECEDENCE;
    }
}
```

---

## 9. Production Configuration

```yaml
# application-prod.yml
server:
  port: 8080
  netty:
    connection-timeout: 30s
    idle-timeout: 60s

spring:
  application:
    name: api-gateway
  cloud:
    gateway:
      httpclient:
        connect-timeout: 5000
        response-timeout: 30s
        pool:
          type: ELASTIC
          max-connections: 500
          acquire-timeout: 45000
          max-idle-time: 30s

      default-filters:
        - name: RequestSize
          args:
            maxSize: 10MB
        - name: Retry
          args:
            retries: 1
            statuses: BAD_GATEWAY,GATEWAY_TIMEOUT
            methods: GET
            backoff:
              firstBackoff: 100ms
              maxBackoff: 500ms

management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics,prometheus
  metrics:
    tags:
      application: ${spring.application.name}
```

---

## 10. Bài tập thực hành

### Bài 1: Basic Gateway
Setup API Gateway với routes tới 2 services, test routing.

### Bài 2: Authentication Filter
Implement JWT authentication filter, protect APIs.

### Bài 3: Rate Limiting
Setup Redis rate limiter, test với concurrent requests.

### Bài 4: Circuit Breaker
Configure circuit breaker với fallback responses.

---

## 11. Key Takeaways

| Feature | Description |
|---------|-------------|
| Single Entry Point | All requests go through gateway |
| Routing | Path, header, method based routing |
| Filters | Pre/post processing of requests |
| Rate Limiting | Throttle requests per user/IP |
| Circuit Breaker | Prevent cascade failures |
| Security | Centralized authentication |

```
Gateway Request Flow:
1. Client → Gateway
2. Pre-filters (auth, rate limit, logging)
3. Route matching (predicates)
4. Load balancing
5. Backend service call
6. Post-filters (response modification)
7. Gateway → Client
```

---

## Navigation

- [← Day 3: Service Discovery](./day-03-service-discovery.md)
- [Day 5: Configuration Management →](./day-05-config-management.md)
