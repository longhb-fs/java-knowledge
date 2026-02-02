# Day 7: Resilience Patterns

## Mục tiêu
- Circuit Breaker pattern
- Retry pattern
- Bulkhead pattern
- Rate Limiter
- Resilience4j implementation

---

## 1. Why Resilience Matters

### 1.1. Cascade Failure Problem

```
┌─────────────────────────────────────────────────────────────────┐
│                    CASCADE FAILURE                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Normal Flow:                                                   │
│   User → Gateway → Order → Payment → Inventory                  │
│    ✓       ✓        ✓        ✓          ✓                       │
│                                                                  │
│   Payment Service Fails:                                        │
│   User → Gateway → Order → Payment ✗                            │
│    ↓       ↓        ↓        ↓                                  │
│   Wait   Wait     Wait    Timeout                               │
│                                                                  │
│   Without Resilience:                                           │
│   - Thread pools exhausted                                      │
│   - Memory consumption increases                                │
│   - All services slow down                                      │
│   - Complete system failure                                     │
│                                                                  │
│   ┌─────┐   ┌─────┐   ┌─────┐   ┌─────┐   ┌─────┐             │
│   │ OK  │ → │Slow │ → │Slow │ → │Down │ → │ OK  │             │
│   └─────┘   └─────┘   └─────┘   └─────┘   └─────┘             │
│                                     ↑                           │
│                              Root Cause                         │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 1.2. Resilience Patterns Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                    RESILIENCE PATTERNS                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   ┌──────────────────┐     ┌──────────────────┐                │
│   │  Circuit Breaker │     │      Retry       │                │
│   │  ─────────────── │     │  ─────────────── │                │
│   │  Stop calling    │     │  Retry transient │                │
│   │  failing service │     │  failures        │                │
│   └──────────────────┘     └──────────────────┘                │
│                                                                  │
│   ┌──────────────────┐     ┌──────────────────┐                │
│   │    Bulkhead      │     │   Rate Limiter   │                │
│   │  ─────────────── │     │  ─────────────── │                │
│   │  Isolate failures│     │  Limit request   │                │
│   │  limit resources │     │  rate            │                │
│   └──────────────────┘     └──────────────────┘                │
│                                                                  │
│   ┌──────────────────┐     ┌──────────────────┐                │
│   │   Time Limiter   │     │    Fallback      │                │
│   │  ─────────────── │     │  ─────────────── │                │
│   │  Limit execution │     │  Default value   │                │
│   │  time            │     │  on failure      │                │
│   └──────────────────┘     └──────────────────┘                │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## 2. Resilience4j Setup

### 2.1. Dependencies

```xml
<dependency>
    <groupId>io.github.resilience4j</groupId>
    <artifactId>resilience4j-spring-boot3</artifactId>
    <version>2.2.0</version>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-aop</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
```

### 2.2. Basic Configuration

```yaml
# application.yml
resilience4j:
  circuitbreaker:
    configs:
      default:
        slidingWindowSize: 10
        failureRateThreshold: 50
        waitDurationInOpenState: 10000
        permittedNumberOfCallsInHalfOpenState: 3
        automaticTransitionFromOpenToHalfOpenEnabled: true
        recordExceptions:
          - java.io.IOException
          - java.net.ConnectException
          - org.springframework.web.client.HttpServerErrorException

  retry:
    configs:
      default:
        maxAttempts: 3
        waitDuration: 500ms
        retryExceptions:
          - java.io.IOException
          - java.net.ConnectException

  bulkhead:
    configs:
      default:
        maxConcurrentCalls: 10
        maxWaitDuration: 100ms

  ratelimiter:
    configs:
      default:
        limitForPeriod: 10
        limitRefreshPeriod: 1s
        timeoutDuration: 500ms

  timelimiter:
    configs:
      default:
        timeoutDuration: 3s
        cancelRunningFuture: true
```

---

## 3. Circuit Breaker

### 3.1. How It Works

```
┌─────────────────────────────────────────────────────────────────┐
│                 CIRCUIT BREAKER STATES                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│                    ┌─────────────┐                              │
│        Success     │   CLOSED    │   Failure                    │
│    ┌───────────────│  (Normal)   │───────────────┐              │
│    │               └──────┬──────┘               │              │
│    │                      │                      │              │
│    │         Failure threshold exceeded          │              │
│    │                      │                      │              │
│    │                      ▼                      │              │
│    │               ┌─────────────┐               │              │
│    │               │    OPEN     │               │              │
│    │               │  (Failing)  │               │              │
│    │               └──────┬──────┘               │              │
│    │                      │                      │              │
│    │         Wait duration expires               │              │
│    │                      │                      │              │
│    │                      ▼                      │              │
│    │               ┌─────────────┐               │              │
│    └───────────────│ HALF-OPEN   │───────────────┘              │
│       Success      │  (Testing)  │    Failure                   │
│                    └─────────────┘                              │
│                                                                  │
│   CLOSED: All requests pass through                             │
│   OPEN: All requests fail fast (no call to service)             │
│   HALF-OPEN: Limited requests to test if service recovered      │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 3.2. Configuration

```yaml
resilience4j:
  circuitbreaker:
    instances:
      user-service:
        # Sliding window for failure calculation
        slidingWindowType: COUNT_BASED
        slidingWindowSize: 10

        # Open circuit when 50% of calls fail
        failureRateThreshold: 50

        # Also consider slow calls
        slowCallRateThreshold: 80
        slowCallDurationThreshold: 2s

        # Wait before transitioning to half-open
        waitDurationInOpenState: 10s

        # Calls allowed in half-open state
        permittedNumberOfCallsInHalfOpenState: 3

        # Minimum calls before calculating failure rate
        minimumNumberOfCalls: 5

        # Auto transition from open to half-open
        automaticTransitionFromOpenToHalfOpenEnabled: true

        # Record specific exceptions
        recordExceptions:
          - java.io.IOException
          - feign.FeignException.ServiceUnavailable

        # Ignore specific exceptions
        ignoreExceptions:
          - com.example.exception.BusinessException
```

### 3.3. Usage

```java
@Service
public class UserService {

    private final UserClient userClient;

    // Annotation-based
    @CircuitBreaker(name = "user-service", fallbackMethod = "getUserFallback")
    public User getUser(Long id) {
        return userClient.getUser(id);
    }

    private User getUserFallback(Long id, Exception ex) {
        log.error("Circuit breaker fallback for user {}: {}", id, ex.getMessage());
        return User.builder()
            .id(id)
            .name("Unknown")
            .status("SERVICE_UNAVAILABLE")
            .build();
    }

    // Different fallback for different exceptions
    @CircuitBreaker(name = "user-service", fallbackMethod = "getUserFallback")
    public User getUserById(Long id) {
        return userClient.getUser(id);
    }

    private User getUserFallback(Long id, CallNotPermittedException ex) {
        log.warn("Circuit is OPEN for user service");
        return User.cached(id);
    }

    private User getUserFallback(Long id, TimeoutException ex) {
        log.warn("Timeout calling user service");
        return User.unknown(id);
    }

    private User getUserFallback(Long id, Exception ex) {
        log.error("General error: {}", ex.getMessage());
        return User.unknown(id);
    }
}
```

### 3.4. Programmatic Usage

```java
@Service
public class UserService {

    private final CircuitBreakerRegistry circuitBreakerRegistry;
    private final UserClient userClient;

    public User getUser(Long id) {
        CircuitBreaker circuitBreaker = circuitBreakerRegistry
            .circuitBreaker("user-service");

        // Decorate the call
        Supplier<User> decoratedSupplier = CircuitBreaker
            .decorateSupplier(circuitBreaker, () -> userClient.getUser(id));

        // Execute with fallback
        return Try.ofSupplier(decoratedSupplier)
            .recover(throwable -> User.fallback(id))
            .get();
    }

    // Reactive style
    public Mono<User> getUserReactive(Long id) {
        CircuitBreaker circuitBreaker = circuitBreakerRegistry
            .circuitBreaker("user-service");

        return webClient.get()
            .uri("/api/users/{id}", id)
            .retrieve()
            .bodyToMono(User.class)
            .transformDeferred(CircuitBreakerOperator.of(circuitBreaker))
            .onErrorResume(CallNotPermittedException.class,
                ex -> Mono.just(User.fallback(id)));
    }
}
```

---

## 4. Retry Pattern

### 4.1. Configuration

```yaml
resilience4j:
  retry:
    instances:
      user-service:
        maxAttempts: 3
        waitDuration: 500ms

        # Exponential backoff
        enableExponentialBackoff: true
        exponentialBackoffMultiplier: 2

        # Randomization
        enableRandomizedWait: true
        randomizedWaitFactor: 0.5

        # Retry specific exceptions
        retryExceptions:
          - java.io.IOException
          - java.net.ConnectException
          - org.springframework.web.client.ResourceAccessException

        # Don't retry on
        ignoreExceptions:
          - com.example.exception.ValidationException
          - com.example.exception.NotFoundException
```

### 4.2. Usage

```java
@Service
public class UserService {

    @Retry(name = "user-service", fallbackMethod = "getUserFallback")
    public User getUser(Long id) {
        return userClient.getUser(id);
    }

    private User getUserFallback(Long id, Exception ex) {
        log.error("All retries failed for user {}: {}", id, ex.getMessage());
        throw new ServiceUnavailableException("User service unavailable");
    }
}

// Programmatic
@Service
public class UserService {

    private final RetryRegistry retryRegistry;

    public User getUser(Long id) {
        Retry retry = retryRegistry.retry("user-service");

        Supplier<User> decoratedSupplier = Retry
            .decorateSupplier(retry, () -> userClient.getUser(id));

        return decoratedSupplier.get();
    }

    // Custom retry config
    public User getUserWithCustomRetry(Long id) {
        RetryConfig config = RetryConfig.custom()
            .maxAttempts(5)
            .waitDuration(Duration.ofMillis(100))
            .retryOnResult(response -> response == null)
            .retryExceptions(IOException.class, TimeoutException.class)
            .ignoreExceptions(BusinessException.class)
            .build();

        Retry retry = Retry.of("custom-retry", config);

        return Retry.decorateSupplier(retry, () -> userClient.getUser(id)).get();
    }
}
```

### 4.3. Retry with Circuit Breaker

```java
@Service
public class UserService {

    // Order matters: Retry -> CircuitBreaker
    @Retry(name = "user-service")
    @CircuitBreaker(name = "user-service", fallbackMethod = "getUserFallback")
    public User getUser(Long id) {
        return userClient.getUser(id);
    }

    // Flow: Try 3 times → if all fail → circuit breaker records failure
    // When circuit opens → no more retries, fail fast
}

// Configure order explicitly
resilience4j:
  circuitbreaker:
    circuitBreakerAspectOrder: 1
  retry:
    retryAspectOrder: 2  # Higher number = inner (executes first)
```

---

## 5. Bulkhead Pattern

### 5.1. Types

```
┌─────────────────────────────────────────────────────────────────┐
│                    BULKHEAD PATTERNS                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   SEMAPHORE BULKHEAD (Default)                                  │
│   ─────────────────────────────                                 │
│   ┌────────────────────────────┐                                │
│   │      Service Call          │                                │
│   │  ┌───┬───┬───┬───┬───┐    │                                │
│   │  │ 1 │ 2 │ 3 │ 4 │ 5 │    │ maxConcurrentCalls: 5          │
│   │  └───┴───┴───┴───┴───┘    │                                │
│   │     Wait Queue: [6,7,8]   │ maxWaitDuration: 100ms         │
│   └────────────────────────────┘                                │
│                                                                  │
│   Pros: Low overhead, simple                                    │
│   Cons: Shared thread pool                                      │
│                                                                  │
│   ─────────────────────────────────────────────────────────────│
│                                                                  │
│   THREAD POOL BULKHEAD                                          │
│   ────────────────────                                          │
│   ┌────────────────────────────┐                                │
│   │   Dedicated Thread Pool    │                                │
│   │  ┌───┬───┬───┬───┬───┐    │                                │
│   │  │ T1│ T2│ T3│ T4│ T5│    │ maxThreadPoolSize: 5           │
│   │  └───┴───┴───┴───┴───┘    │                                │
│   │     Queue: [task,task]    │ queueCapacity: 10              │
│   └────────────────────────────┘                                │
│                                                                  │
│   Pros: Complete isolation                                      │
│   Cons: More resource usage                                     │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 5.2. Configuration

```yaml
resilience4j:
  bulkhead:
    instances:
      user-service:
        # Semaphore bulkhead
        maxConcurrentCalls: 10
        maxWaitDuration: 100ms

  thread-pool-bulkhead:
    instances:
      payment-service:
        # Thread pool bulkhead
        maxThreadPoolSize: 10
        coreThreadPoolSize: 5
        queueCapacity: 20
        keepAliveDuration: 20ms
```

### 5.3. Usage

```java
@Service
public class UserService {

    // Semaphore bulkhead
    @Bulkhead(name = "user-service", fallbackMethod = "getUserFallback")
    public User getUser(Long id) {
        return userClient.getUser(id);
    }

    // Thread pool bulkhead (async)
    @Bulkhead(name = "user-service", type = Bulkhead.Type.THREADPOOL)
    public CompletableFuture<User> getUserAsync(Long id) {
        return CompletableFuture.supplyAsync(() -> userClient.getUser(id));
    }

    private User getUserFallback(Long id, BulkheadFullException ex) {
        log.warn("Bulkhead full, rejecting request for user {}", id);
        throw new ServiceBusyException("Service busy, try again later");
    }
}

// Programmatic
@Service
public class UserService {

    private final BulkheadRegistry bulkheadRegistry;

    public User getUser(Long id) {
        Bulkhead bulkhead = bulkheadRegistry.bulkhead("user-service");

        Supplier<User> decoratedSupplier = Bulkhead
            .decorateSupplier(bulkhead, () -> userClient.getUser(id));

        return Try.ofSupplier(decoratedSupplier)
            .recover(BulkheadFullException.class, ex -> {
                log.warn("Bulkhead full");
                return User.fallback(id);
            })
            .get();
    }
}
```

---

## 6. Rate Limiter

### 6.1. Configuration

```yaml
resilience4j:
  ratelimiter:
    instances:
      user-service:
        # 10 requests per second
        limitForPeriod: 10
        limitRefreshPeriod: 1s

        # Wait up to 500ms for permission
        timeoutDuration: 500ms

        # Register for health indicator
        registerHealthIndicator: true

      api-rate-limit:
        # 100 requests per minute
        limitForPeriod: 100
        limitRefreshPeriod: 60s
        timeoutDuration: 0  # Fail immediately if limit reached
```

### 6.2. Usage

```java
@Service
public class UserService {

    @RateLimiter(name = "user-service", fallbackMethod = "rateLimitFallback")
    public User getUser(Long id) {
        return userClient.getUser(id);
    }

    private User rateLimitFallback(Long id, RequestNotPermitted ex) {
        log.warn("Rate limit exceeded for user request");
        throw new TooManyRequestsException("Rate limit exceeded, try again later");
    }
}

// In controller
@RestController
@RequestMapping("/api/users")
public class UserController {

    @GetMapping("/{id}")
    @RateLimiter(name = "api-rate-limit")
    public ResponseEntity<User> getUser(@PathVariable Long id) {
        return ResponseEntity.ok(userService.getUser(id));
    }
}

// Return proper HTTP status
@ExceptionHandler(RequestNotPermitted.class)
public ResponseEntity<ErrorResponse> handleRateLimitException(RequestNotPermitted ex) {
    return ResponseEntity.status(HttpStatus.TOO_MANY_REQUESTS)
        .header("Retry-After", "60")
        .body(new ErrorResponse("Rate limit exceeded"));
}
```

---

## 7. Time Limiter

### 7.1. Configuration

```yaml
resilience4j:
  timelimiter:
    instances:
      user-service:
        timeoutDuration: 3s
        cancelRunningFuture: true
```

### 7.2. Usage

```java
@Service
public class UserService {

    @TimeLimiter(name = "user-service", fallbackMethod = "timeoutFallback")
    public CompletableFuture<User> getUserAsync(Long id) {
        return CompletableFuture.supplyAsync(() -> userClient.getUser(id));
    }

    private CompletableFuture<User> timeoutFallback(Long id, TimeoutException ex) {
        log.warn("Timeout getting user {}", id);
        return CompletableFuture.completedFuture(User.fallback(id));
    }
}

// Combine with Circuit Breaker
@Service
public class UserService {

    @TimeLimiter(name = "user-service")
    @CircuitBreaker(name = "user-service", fallbackMethod = "fallback")
    public CompletableFuture<User> getUser(Long id) {
        return CompletableFuture.supplyAsync(() -> userClient.getUser(id));
    }
}
```

---

## 8. Combining Patterns

### 8.1. Recommended Order

```
Request
   │
   ▼
┌──────────────┐
│ Rate Limiter │  ← First: Reject excess traffic
└──────┬───────┘
       │
       ▼
┌──────────────┐
│   Bulkhead   │  ← Second: Limit concurrent calls
└──────┬───────┘
       │
       ▼
┌──────────────┐
│    Retry     │  ← Third: Retry transient failures
└──────┬───────┘
       │
       ▼
┌──────────────┐
│Circuit Break │  ← Fourth: Prevent cascade failure
└──────┬───────┘
       │
       ▼
┌──────────────┐
│ Time Limiter │  ← Fifth: Timeout long calls
└──────┬───────┘
       │
       ▼
   Service Call
```

### 8.2. Configuration

```yaml
resilience4j:
  ratelimiter:
    rateLimiterAspectOrder: 1
  bulkhead:
    bulkheadAspectOrder: 2
  retry:
    retryAspectOrder: 3
  circuitbreaker:
    circuitBreakerAspectOrder: 4
  timelimiter:
    timeLimiterAspectOrder: 5
```

### 8.3. Combined Usage

```java
@Service
public class PaymentService {

    private final PaymentClient paymentClient;

    @RateLimiter(name = "payment-service")
    @Bulkhead(name = "payment-service")
    @Retry(name = "payment-service")
    @CircuitBreaker(name = "payment-service", fallbackMethod = "paymentFallback")
    @TimeLimiter(name = "payment-service")
    public CompletableFuture<PaymentResult> processPayment(PaymentRequest request) {
        return CompletableFuture.supplyAsync(() ->
            paymentClient.process(request));
    }

    private CompletableFuture<PaymentResult> paymentFallback(
            PaymentRequest request, Exception ex) {

        log.error("Payment processing failed: {}", ex.getMessage());

        if (ex instanceof CallNotPermittedException) {
            return CompletableFuture.completedFuture(
                PaymentResult.circuitOpen());
        }
        if (ex instanceof BulkheadFullException) {
            return CompletableFuture.completedFuture(
                PaymentResult.systemBusy());
        }
        if (ex instanceof RequestNotPermitted) {
            return CompletableFuture.completedFuture(
                PaymentResult.rateLimited());
        }

        return CompletableFuture.completedFuture(
            PaymentResult.failed(ex.getMessage()));
    }
}
```

---

## 9. Monitoring

### 9.1. Actuator Endpoints

```yaml
management:
  endpoints:
    web:
      exposure:
        include: health,circuitbreakers,circuitbreakerevents,
                 retries,retryevents,ratelimiters,bulkheads

  health:
    circuitbreakers:
      enabled: true
    ratelimiters:
      enabled: true
```

```bash
# Circuit breaker status
GET /actuator/circuitbreakers

# Circuit breaker events
GET /actuator/circuitbreakerevents

# Retry metrics
GET /actuator/retries

# Rate limiter status
GET /actuator/ratelimiters
```

### 9.2. Metrics with Micrometer

```yaml
resilience4j:
  circuitbreaker:
    metrics:
      enabled: true
      legacy:
        enabled: true

# Prometheus metrics exposed:
# resilience4j_circuitbreaker_state
# resilience4j_circuitbreaker_calls_total
# resilience4j_circuitbreaker_failure_rate
# resilience4j_retry_calls_total
# resilience4j_bulkhead_available_concurrent_calls
# resilience4j_ratelimiter_available_permissions
```

### 9.3. Event Listeners

```java
@Component
public class ResilienceEventListener {

    @EventListener
    public void onCircuitBreakerEvent(CircuitBreakerOnStateTransitionEvent event) {
        log.warn("Circuit Breaker '{}' transitioned from {} to {}",
            event.getCircuitBreakerName(),
            event.getStateTransition().getFromState(),
            event.getStateTransition().getToState());
    }

    @EventListener
    public void onRetryEvent(RetryOnRetryEvent event) {
        log.info("Retry #{} for '{}'",
            event.getNumberOfRetryAttempts(),
            event.getName());
    }

    @EventListener
    public void onRateLimitEvent(RateLimiterOnFailureEvent event) {
        log.warn("Rate limit exceeded for '{}'", event.getRateLimiterName());
    }
}

// Or register directly
@Configuration
public class ResilienceConfig {

    @Bean
    public RegistryEventConsumer<CircuitBreaker> circuitBreakerEventConsumer() {
        return new RegistryEventConsumer<CircuitBreaker>() {
            @Override
            public void onEntryAddedEvent(EntryAddedEvent<CircuitBreaker> event) {
                event.getAddedEntry().getEventPublisher()
                    .onStateTransition(e ->
                        log.info("CB {} state: {}", e.getCircuitBreakerName(),
                            e.getStateTransition()));
            }
        };
    }
}
```

---

## 10. Bài tập thực hành

### Bài 1: Circuit Breaker
Implement circuit breaker cho User Service client, test với failure scenarios.

### Bài 2: Retry với Backoff
Configure exponential backoff retry, verify với logs.

### Bài 3: Combined Resilience
Implement full resilience stack cho Payment Service.

### Bài 4: Monitoring Dashboard
Setup Grafana dashboard hiển thị resilience metrics.

---

## 11. Key Takeaways

| Pattern | Purpose | When to Use |
|---------|---------|-------------|
| Circuit Breaker | Prevent cascade failure | External service calls |
| Retry | Handle transient failures | Network issues, timeouts |
| Bulkhead | Isolate failures | Resource-intensive calls |
| Rate Limiter | Control request rate | API protection |
| Time Limiter | Bound execution time | Async operations |

```
Best Practices:
1. Always have fallback strategies
2. Log all resilience events
3. Monitor circuit breaker states
4. Tune thresholds based on SLAs
5. Test failure scenarios regularly
6. Use meaningful names for instances
```

---

## Navigation

- [← Day 6: Inter-Service Communication](./day-06-inter-service-communication.md)
- [Day 8: Distributed Transactions →](./day-08-distributed-transactions.md)
