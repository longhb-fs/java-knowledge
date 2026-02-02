# Day 3: Service Discovery

## Mục tiêu
- Service Discovery patterns
- Eureka deep dive
- Consul integration
- Health checking & monitoring

---

## 1. Service Discovery Patterns

### 1.1. Why Service Discovery?

```
Problem without Service Discovery:
┌─────────────────────────────────────────────────────────────────┐
│                                                                  │
│   Order Service needs to call User Service                      │
│                                                                  │
│   ❌ Hardcoded: http://192.168.1.10:8081/api/users              │
│      - IP changes when container restarts                       │
│      - Port conflicts                                           │
│      - No load balancing                                        │
│      - No failover                                              │
│                                                                  │
│   ✅ With Discovery: http://user-service/api/users              │
│      - Dynamic IP resolution                                    │
│      - Automatic load balancing                                 │
│      - Health-aware routing                                     │
│      - Automatic failover                                       │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 1.2. Discovery Patterns

```
┌─────────────────────────────────────────────────────────────────┐
│              CLIENT-SIDE DISCOVERY                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Client ──► Registry ──► Get Service Instances                 │
│      │                                                           │
│      └──────────────────► Call Service Directly                 │
│                                                                  │
│   Pros: Simple, no extra hop                                    │
│   Cons: Client needs discovery logic                            │
│   Examples: Netflix Eureka, Consul                              │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│              SERVER-SIDE DISCOVERY                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Client ──► Load Balancer ──► Registry                         │
│                   │                                              │
│                   └──────────► Route to Service                  │
│                                                                  │
│   Pros: Client is simple                                        │
│   Cons: Extra network hop, LB is single point                   │
│   Examples: AWS ELB, Kubernetes Service                         │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## 2. Eureka Deep Dive

### 2.1. Eureka Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                    EUREKA ARCHITECTURE                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   ┌─────────────┐              ┌─────────────┐                  │
│   │   Eureka    │◄── Sync ───►│   Eureka    │                  │
│   │  Server 1   │              │  Server 2   │                  │
│   └──────┬──────┘              └──────┬──────┘                  │
│          │                            │                          │
│          │  Register/Heartbeat        │                          │
│          │                            │                          │
│   ┌──────┴──────────────────────────┴──────┐                   │
│   │                                          │                   │
│   ▼                  ▼                  ▼    │                   │
│ ┌─────┐          ┌─────┐          ┌─────┐   │                   │
│ │Svc A│          │Svc B│          │Svc C│   │                   │
│ │ x3  │          │ x2  │          │ x2  │   │                   │
│ └─────┘          └─────┘          └─────┘   │                   │
│                                              │                   │
│   Register: POST /eureka/apps/{appName}     │                   │
│   Heartbeat: PUT /eureka/apps/{appName}/{id}│                   │
│   Fetch: GET /eureka/apps                   │                   │
│                                              │                   │
└─────────────────────────────────────────────────────────────────┘
```

### 2.2. Eureka Server - Production Setup

```java
@SpringBootApplication
@EnableEurekaServer
public class EurekaServerApplication {
    public static void main(String[] args) {
        SpringApplication.run(EurekaServerApplication.class, args);
    }
}
```

```yaml
# application.yml - Standalone (Development)
server:
  port: 8761

spring:
  application:
    name: eureka-server

eureka:
  instance:
    hostname: localhost
  client:
    register-with-eureka: false
    fetch-registry: false
    service-url:
      defaultZone: http://${eureka.instance.hostname}:${server.port}/eureka/
  server:
    enable-self-preservation: false
    eviction-interval-timer-in-ms: 5000
```

```yaml
# application.yml - Cluster (Production)
---
spring:
  config:
    activate:
      on-profile: peer1

server:
  port: 8761

eureka:
  instance:
    hostname: eureka-peer1
  client:
    service-url:
      defaultZone: http://eureka-peer2:8762/eureka/,http://eureka-peer3:8763/eureka/

---
spring:
  config:
    activate:
      on-profile: peer2

server:
  port: 8762

eureka:
  instance:
    hostname: eureka-peer2
  client:
    service-url:
      defaultZone: http://eureka-peer1:8761/eureka/,http://eureka-peer3:8763/eureka/

---
spring:
  config:
    activate:
      on-profile: peer3

server:
  port: 8763

eureka:
  instance:
    hostname: eureka-peer3
  client:
    service-url:
      defaultZone: http://eureka-peer1:8761/eureka/,http://eureka-peer2:8762/eureka/
```

### 2.3. Eureka Client Configuration

```yaml
# Full Eureka Client Configuration
eureka:
  client:
    # Enable/disable registration
    register-with-eureka: true
    # Enable/disable fetching registry
    fetch-registry: true
    # Registry fetch interval
    registry-fetch-interval-seconds: 30
    # Service URL
    service-url:
      defaultZone: http://localhost:8761/eureka/
    # Enable delta fetching (optimization)
    disable-delta: false
    # Initial instance info replication interval
    initial-instance-info-replication-interval-seconds: 40

  instance:
    # Prefer IP address over hostname
    prefer-ip-address: true
    # Instance ID (unique identifier)
    instance-id: ${spring.application.name}:${spring.cloud.client.ip-address}:${server.port}
    # Heartbeat interval
    lease-renewal-interval-in-seconds: 30
    # Expiration time (remove if no heartbeat)
    lease-expiration-duration-in-seconds: 90
    # Metadata
    metadata-map:
      zone: zone1
      version: 1.0.0
    # Health check URL
    health-check-url-path: /actuator/health
    status-page-url-path: /actuator/info
```

### 2.4. Self-Preservation Mode

```
┌─────────────────────────────────────────────────────────────────┐
│                    SELF-PRESERVATION MODE                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   When Eureka loses heartbeats from many instances:             │
│                                                                  │
│   Network Partition Scenario:                                   │
│   ┌─────────┐                      ┌─────────┐                  │
│   │ Eureka  │◄── ✗ Network ✗ ──►  │Services │                  │
│   │ Server  │      Issue          │(Healthy)│                  │
│   └─────────┘                      └─────────┘                  │
│                                                                  │
│   Without Self-Preservation:                                    │
│   - Eureka evicts all instances                                 │
│   - Services can't find each other                              │
│   - System-wide outage!                                         │
│                                                                  │
│   With Self-Preservation (enabled by default):                  │
│   - Eureka keeps the registry as-is                             │
│   - Stale data better than no data                              │
│   - Red warning banner appears                                  │
│                                                                  │
│   Trigger: When renewal percentage < 85%                        │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

```yaml
eureka:
  server:
    # Enable/disable self-preservation
    enable-self-preservation: true
    # Renewal percent threshold
    renewal-percent-threshold: 0.85
    # Expected renewals per minute
    expected-client-renewal-interval-seconds: 30
```

---

## 3. Service Registration

### 3.1. Registration Lifecycle

```java
// Custom Registration Behavior
@Component
public class CustomEurekaInstanceConfigBean extends EurekaInstanceConfigBean {

    public CustomEurekaInstanceConfigBean(InetUtils inetUtils) {
        super(inetUtils);
    }

    @Override
    public String getInstanceId() {
        return String.format("%s:%s:%d",
            getAppname(),
            getIpAddress(),
            getNonSecurePort());
    }
}

// Registration Event Listener
@Component
public class EurekaEventListener {

    private static final Logger log = LoggerFactory.getLogger(EurekaEventListener.class);

    @EventListener
    public void onEurekaStatusChange(StatusChangeEvent event) {
        log.info("Eureka status changed: {} -> {}",
            event.getPreviousStatus(), event.getStatus());
    }

    @EventListener
    public void onCacheRefreshed(CacheRefreshedEvent event) {
        log.info("Eureka cache refreshed");
    }
}
```

### 3.2. Manual Registration Control

```java
@RestController
@RequestMapping("/admin")
public class EurekaAdminController {

    @Autowired
    private EurekaClient eurekaClient;

    @Autowired
    private ApplicationInfoManager applicationInfoManager;

    @PostMapping("/status/out-of-service")
    public String setOutOfService() {
        applicationInfoManager.setInstanceStatus(InstanceStatus.OUT_OF_SERVICE);
        return "Status set to OUT_OF_SERVICE";
    }

    @PostMapping("/status/up")
    public String setUp() {
        applicationInfoManager.setInstanceStatus(InstanceStatus.UP);
        return "Status set to UP";
    }

    @GetMapping("/instances/{serviceName}")
    public List<String> getInstances(@PathVariable String serviceName) {
        Application application = eurekaClient.getApplication(serviceName);
        if (application == null) {
            return Collections.emptyList();
        }
        return application.getInstances().stream()
            .map(InstanceInfo::getHomePageUrl)
            .collect(Collectors.toList());
    }
}
```

---

## 4. Consul Integration

### 4.1. Consul Setup

```yaml
# docker-compose.yml
version: '3.8'
services:
  consul:
    image: consul:1.15
    ports:
      - "8500:8500"
      - "8600:8600/udp"
    command: agent -server -ui -bootstrap-expect=1 -client=0.0.0.0
```

### 4.2. Spring Cloud Consul

```xml
<!-- pom.xml -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-consul-discovery</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-consul-config</artifactId>
</dependency>
```

```java
@SpringBootApplication
@EnableDiscoveryClient
public class UserServiceApplication {
    public static void main(String[] args) {
        SpringApplication.run(UserServiceApplication.class, args);
    }
}
```

```yaml
# application.yml
spring:
  application:
    name: user-service
  cloud:
    consul:
      host: localhost
      port: 8500
      discovery:
        # Service name in Consul
        service-name: ${spring.application.name}
        # Prefer IP
        prefer-ip-address: true
        # Health check
        health-check-path: /actuator/health
        health-check-interval: 10s
        # Instance ID
        instance-id: ${spring.application.name}:${random.value}
        # Tags for filtering
        tags:
          - version=1.0.0
          - environment=dev
      config:
        enabled: true
        format: YAML
        prefix: config
        default-context: application
```

### 4.3. Consul vs Eureka

| Feature | Eureka | Consul |
|---------|--------|--------|
| Consistency | AP (Eventually Consistent) | CP (Strongly Consistent) |
| Health Check | Client heartbeat | Agent-based |
| Config Store | No | Yes (KV Store) |
| Multi-DC | Limited | Built-in |
| DNS Interface | No | Yes |
| ACL | No | Yes |
| UI | Basic | Rich |

---

## 5. Health Checking

### 5.1. Actuator Health Endpoint

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
```

```yaml
management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics
  endpoint:
    health:
      show-details: always
      show-components: always
  health:
    defaults:
      enabled: true
```

### 5.2. Custom Health Indicator

```java
@Component
public class DatabaseHealthIndicator implements HealthIndicator {

    @Autowired
    private DataSource dataSource;

    @Override
    public Health health() {
        try (Connection conn = dataSource.getConnection()) {
            if (conn.isValid(1)) {
                return Health.up()
                    .withDetail("database", "MySQL")
                    .withDetail("status", "Connected")
                    .build();
            }
        } catch (SQLException e) {
            return Health.down()
                .withDetail("error", e.getMessage())
                .build();
        }
        return Health.down().build();
    }
}

// External Service Health Check
@Component
public class ExternalApiHealthIndicator implements HealthIndicator {

    private final RestTemplate restTemplate;

    public ExternalApiHealthIndicator(RestTemplate restTemplate) {
        this.restTemplate = restTemplate;
    }

    @Override
    public Health health() {
        try {
            ResponseEntity<String> response = restTemplate.getForEntity(
                "http://external-api/health", String.class);

            if (response.getStatusCode().is2xxSuccessful()) {
                return Health.up()
                    .withDetail("external-api", "Available")
                    .build();
            }
        } catch (Exception e) {
            return Health.down()
                .withDetail("external-api", "Unavailable")
                .withDetail("error", e.getMessage())
                .build();
        }
        return Health.unknown().build();
    }
}
```

### 5.3. Health Groups

```yaml
management:
  endpoint:
    health:
      group:
        readiness:
          include: db, redis
          show-details: always
        liveness:
          include: ping
          show-details: always

# Kubernetes probes
# GET /actuator/health/readiness
# GET /actuator/health/liveness
```

---

## 6. Service Discovery in Action

### 6.1. DiscoveryClient Usage

```java
@Service
public class ServiceDiscoveryService {

    @Autowired
    private DiscoveryClient discoveryClient;

    public List<String> getServiceInstances(String serviceName) {
        return discoveryClient.getInstances(serviceName)
            .stream()
            .map(ServiceInstance::getUri)
            .map(URI::toString)
            .collect(Collectors.toList());
    }

    public List<String> getAllServices() {
        return discoveryClient.getServices();
    }

    public ServiceInstance getFirstInstance(String serviceName) {
        List<ServiceInstance> instances = discoveryClient.getInstances(serviceName);
        if (instances.isEmpty()) {
            throw new ServiceNotFoundException("No instances found for: " + serviceName);
        }
        return instances.get(0);
    }
}
```

### 6.2. Calling Services

```java
@Service
public class OrderService {

    private final RestTemplate restTemplate;
    private final WebClient.Builder webClientBuilder;

    public OrderService(
            @LoadBalanced RestTemplate restTemplate,
            @LoadBalanced WebClient.Builder webClientBuilder) {
        this.restTemplate = restTemplate;
        this.webClientBuilder = webClientBuilder;
    }

    // Using RestTemplate (blocking)
    public User getUserBlocking(Long userId) {
        return restTemplate.getForObject(
            "http://user-service/api/users/{id}",
            User.class,
            userId
        );
    }

    // Using WebClient (reactive)
    public Mono<User> getUserReactive(Long userId) {
        return webClientBuilder.build()
            .get()
            .uri("http://user-service/api/users/{id}", userId)
            .retrieve()
            .bodyToMono(User.class);
    }
}
```

### 6.3. OpenFeign Client

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

// Feign Client Interface
@FeignClient(name = "user-service", path = "/api/users")
public interface UserClient {

    @GetMapping("/{id}")
    User getUser(@PathVariable("id") Long id);

    @GetMapping
    List<User> getAllUsers();

    @PostMapping
    User createUser(@RequestBody CreateUserRequest request);

    @PutMapping("/{id}")
    User updateUser(@PathVariable("id") Long id, @RequestBody UpdateUserRequest request);

    @DeleteMapping("/{id}")
    void deleteUser(@PathVariable("id") Long id);
}

// Usage
@Service
public class OrderService {

    @Autowired
    private UserClient userClient;

    public Order createOrder(CreateOrderRequest request) {
        // Call user service via Feign
        User user = userClient.getUser(request.getUserId());

        if (user == null) {
            throw new UserNotFoundException(request.getUserId());
        }

        // Create order...
        return new Order(request, user);
    }
}
```

---

## 7. Monitoring & Dashboard

### 7.1. Eureka Dashboard

```
http://localhost:8761

┌─────────────────────────────────────────────────────────────────┐
│                    EUREKA DASHBOARD                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   System Status                                                  │
│   ├── Environment: test                                         │
│   ├── Data center: default                                      │
│   └── Current time: 2024-01-15 10:30:00 UTC                    │
│                                                                  │
│   DS Replicas                                                    │
│   └── eureka-peer2 (available)                                  │
│                                                                  │
│   Instances currently registered with Eureka                    │
│   ┌────────────────┬────────────────────────┬────────┐         │
│   │ Application    │ AMIs                    │ Status │         │
│   ├────────────────┼────────────────────────┼────────┤         │
│   │ USER-SERVICE   │ user-service:8081      │   UP   │         │
│   │                │ user-service:8082      │   UP   │         │
│   │ ORDER-SERVICE  │ order-service:8083     │   UP   │         │
│   │ API-GATEWAY    │ api-gateway:8080       │   UP   │         │
│   └────────────────┴────────────────────────┴────────┘         │
│                                                                  │
│   General Info                                                   │
│   ├── Total services: 3                                         │
│   ├── Total instances: 4                                        │
│   └── Renewal threshold: 3                                      │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 7.2. Prometheus Metrics

```yaml
management:
  endpoints:
    web:
      exposure:
        include: prometheus,health,info,metrics
  metrics:
    tags:
      application: ${spring.application.name}
    export:
      prometheus:
        enabled: true
```

```java
// Custom Metrics
@Component
public class DiscoveryMetrics {

    private final DiscoveryClient discoveryClient;
    private final MeterRegistry registry;

    public DiscoveryMetrics(DiscoveryClient discoveryClient, MeterRegistry registry) {
        this.discoveryClient = discoveryClient;
        this.registry = registry;

        // Gauge for service count
        Gauge.builder("eureka.services.count", discoveryClient,
                dc -> dc.getServices().size())
            .description("Number of services registered with Eureka")
            .register(registry);
    }

    @Scheduled(fixedRate = 30000)
    public void recordInstanceCounts() {
        discoveryClient.getServices().forEach(service -> {
            int count = discoveryClient.getInstances(service).size();
            registry.gauge("eureka.instances.count",
                Tags.of("service", service),
                count);
        });
    }
}
```

---

## 8. Best Practices

### 8.1. Production Recommendations

```yaml
# Production Eureka Server
eureka:
  server:
    enable-self-preservation: true
    renewal-percent-threshold: 0.85
    eviction-interval-timer-in-ms: 60000
    response-cache-update-interval-ms: 30000

  instance:
    lease-renewal-interval-in-seconds: 30
    lease-expiration-duration-in-seconds: 90

# Production Eureka Client
eureka:
  client:
    registry-fetch-interval-seconds: 30
    eureka-connection-idle-timeout-seconds: 30
    eureka-server-connect-timeout-seconds: 5
    eureka-server-read-timeout-seconds: 8

  instance:
    prefer-ip-address: true
    lease-renewal-interval-in-seconds: 10
    lease-expiration-duration-in-seconds: 30
```

### 8.2. High Availability Setup

```
┌─────────────────────────────────────────────────────────────────┐
│                    HA EUREKA CLUSTER                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Zone A                         Zone B                          │
│   ┌─────────────┐               ┌─────────────┐                 │
│   │   Eureka    │◄── Sync ────►│   Eureka    │                 │
│   │  Server 1   │               │  Server 2   │                 │
│   └──────┬──────┘               └──────┬──────┘                 │
│          │                              │                        │
│     ┌────┴────┐                   ┌────┴────┐                   │
│     ▼         ▼                   ▼         ▼                   │
│  ┌─────┐  ┌─────┐             ┌─────┐  ┌─────┐                 │
│  │Svc A│  │Svc B│             │Svc A│  │Svc B│                 │
│  │(Z-A)│  │(Z-A)│             │(Z-B)│  │(Z-B)│                 │
│  └─────┘  └─────┘             └─────┘  └─────┘                 │
│                                                                  │
│   Zone Preference: Services prefer same-zone instances          │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## 9. Bài tập thực hành

### Bài 1: Eureka Cluster
Setup Eureka cluster với 2 nodes, verify replication.

### Bài 2: Health Checks
Implement custom health indicators cho database và external API.

### Bài 3: Service Discovery
Tạo 3 services, gọi nhau qua service discovery với Feign Client.

---

## 10. Key Takeaways

| Concept | Description |
|---------|-------------|
| Service Discovery | Dynamic service location resolution |
| Client-side | Client queries registry, calls service |
| Server-side | Load balancer queries registry |
| Eureka | AP system, self-preservation |
| Consul | CP system, health checking |
| Health Check | Verify service availability |
| Feign Client | Declarative REST client |

```
Service Discovery Flow:
1. Service starts → Registers with Eureka
2. Heartbeat every 30s → Maintains registration
3. Client needs Service B → Queries Eureka
4. Eureka returns instances → Client load balances
5. Service stops → Eureka evicts after timeout
```

---

## Navigation

- [← Day 2: Spring Cloud Overview](./day-02-spring-cloud-overview.md)
- [Day 4: API Gateway →](./day-04-api-gateway.md)
