# Day 2: Spring Cloud Overview

## Mục tiêu
- Spring Cloud ecosystem
- Core components
- Spring Cloud vs Netflix OSS
- Setup Spring Cloud project

---

## 1. Spring Cloud Ecosystem

### 1.1. What is Spring Cloud?

```
┌─────────────────────────────────────────────────────────────────┐
│                    SPRING CLOUD ECOSYSTEM                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │                 Spring Cloud Components                  │   │
│   │                                                          │   │
│   │  Service Discovery    │  Config Management               │   │
│   │  (Eureka, Consul)     │  (Config Server, Vault)          │   │
│   │                       │                                  │   │
│   │  API Gateway          │  Circuit Breaker                 │   │
│   │  (Gateway, Zuul)      │  (Resilience4j, Hystrix)         │   │
│   │                       │                                  │   │
│   │  Load Balancing       │  Distributed Tracing             │   │
│   │  (LoadBalancer)       │  (Sleuth, Zipkin)                │   │
│   │                       │                                  │   │
│   │  Messaging            │  Security                        │   │
│   │  (Stream, Bus)        │  (OAuth2, Gateway)               │   │
│   └─────────────────────────────────────────────────────────┘   │
│                                                                  │
│   Built on top of Spring Boot                                   │
└─────────────────────────────────────────────────────────────────┘
```

### 1.2. Spring Cloud Components Map

```
┌─────────────────────────────────────────────────────────────────┐
│                    MICROSERVICES ARCHITECTURE                    │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Client Request                                                 │
│        │                                                         │
│        ▼                                                         │
│   ┌─────────────┐                                               │
│   │ API Gateway │◄─── Spring Cloud Gateway                      │
│   └──────┬──────┘                                               │
│          │                                                       │
│          ▼                                                       │
│   ┌─────────────────┐                                           │
│   │Load Balancer    │◄─── Spring Cloud LoadBalancer             │
│   └────────┬────────┘                                           │
│            │                                                     │
│    ┌───────┼───────┐                                            │
│    ▼       ▼       ▼                                            │
│ ┌─────┐ ┌─────┐ ┌─────┐                                        │
│ │Svc A│ │Svc B│ │Svc C│◄─── Spring Boot Microservices          │
│ └──┬──┘ └──┬──┘ └──┬──┘                                        │
│    │       │       │                                            │
│    └───────┼───────┘                                            │
│            ▼                                                     │
│   ┌─────────────────┐                                           │
│   │Service Registry │◄─── Eureka / Consul                       │
│   └─────────────────┘                                           │
│            │                                                     │
│            ▼                                                     │
│   ┌─────────────────┐                                           │
│   │ Config Server   │◄─── Spring Cloud Config                   │
│   └─────────────────┘                                           │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## 2. Core Components

### 2.1. Service Discovery

| Component | Mô tả |
|-----------|-------|
| **Eureka** | Netflix OSS, self-preservation mode |
| **Consul** | HashiCorp, health checking, KV store |
| **Zookeeper** | Apache, strong consistency |
| **Nacos** | Alibaba, service discovery + config |

### 2.2. Configuration Management

| Component | Mô tả |
|-----------|-------|
| **Config Server** | Git/SVN backed, encryption |
| **Consul KV** | Key-Value store |
| **Vault** | Secrets management |
| **Nacos** | Dynamic configuration |

### 2.3. API Gateway

| Component | Mô tả |
|-----------|-------|
| **Spring Cloud Gateway** | Reactive, recommended |
| **Zuul** | Netflix OSS, legacy |
| **Kong** | Third-party, feature-rich |

### 2.4. Circuit Breaker

| Component | Mô tả |
|-----------|-------|
| **Resilience4j** | Lightweight, recommended |
| **Hystrix** | Netflix OSS, maintenance mode |
| **Sentinel** | Alibaba, flow control |

---

## 3. Spring Cloud vs Netflix OSS

### 3.1. Evolution

```
Timeline:
2015-2018: Spring Cloud Netflix (Eureka, Zuul, Hystrix, Ribbon)
2018-2020: Netflix components entering maintenance mode
2020+:     Spring Cloud native alternatives

┌─────────────────────────────────────────────────────────────────┐
│           NETFLIX OSS  →  SPRING CLOUD NATIVE                   │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Hystrix        →  Resilience4j                                │
│   Ribbon         →  Spring Cloud LoadBalancer                   │
│   Zuul 1.x       →  Spring Cloud Gateway                        │
│   Archaius       →  Spring Cloud Config                         │
│                                                                  │
│   Eureka         →  Still supported (Eureka 2.x)                │
│                     Or use Consul/Nacos                         │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 3.2. Comparison Table

| Feature | Netflix OSS | Spring Cloud Native |
|---------|-------------|---------------------|
| Circuit Breaker | Hystrix (deprecated) | Resilience4j |
| Load Balancer | Ribbon (deprecated) | LoadBalancer |
| API Gateway | Zuul 1.x | Spring Cloud Gateway |
| Service Discovery | Eureka | Eureka / Consul |
| Configuration | Archaius | Config Server |

---

## 4. Project Setup

### 4.1. Parent POM (BOM)

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0">
    <modelVersion>4.0.0</modelVersion>

    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>3.2.0</version>
    </parent>

    <groupId>com.example</groupId>
    <artifactId>microservices-parent</artifactId>
    <version>1.0.0</version>
    <packaging>pom</packaging>

    <properties>
        <java.version>21</java.version>
        <spring-cloud.version>2023.0.0</spring-cloud.version>
    </properties>

    <modules>
        <module>service-discovery</module>
        <module>config-server</module>
        <module>api-gateway</module>
        <module>user-service</module>
        <module>order-service</module>
    </modules>

    <dependencyManagement>
        <dependencies>
            <dependency>
                <groupId>org.springframework.cloud</groupId>
                <artifactId>spring-cloud-dependencies</artifactId>
                <version>${spring-cloud.version}</version>
                <type>pom</type>
                <scope>import</scope>
            </dependency>
        </dependencies>
    </dependencyManagement>

    <dependencies>
        <!-- Common dependencies for all modules -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-actuator</artifactId>
        </dependency>
        <dependency>
            <groupId>org.projectlombok</groupId>
            <artifactId>lombok</artifactId>
            <optional>true</optional>
        </dependency>
    </dependencies>
</project>
```

### 4.2. Version Compatibility

```
Spring Cloud Version    │  Spring Boot Version
────────────────────────┼─────────────────────
2023.0.x (Leyton)       │  3.2.x
2022.0.x (Kilburn)      │  3.0.x, 3.1.x
2021.0.x (Jubilee)      │  2.6.x, 2.7.x
2020.0.x (Ilford)       │  2.4.x, 2.5.x
```

---

## 5. Service Discovery (Eureka)

### 5.1. Eureka Server

```xml
<!-- service-discovery/pom.xml -->
<dependencies>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-netflix-eureka-server</artifactId>
    </dependency>
</dependencies>
```

```java
// EurekaServerApplication.java
@SpringBootApplication
@EnableEurekaServer
public class EurekaServerApplication {
    public static void main(String[] args) {
        SpringApplication.run(EurekaServerApplication.class, args);
    }
}
```

```yaml
# application.yml
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
    enable-self-preservation: false  # For development
```

### 5.2. Eureka Client

```xml
<!-- user-service/pom.xml -->
<dependencies>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
</dependencies>
```

```java
// UserServiceApplication.java
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
server:
  port: 8081

spring:
  application:
    name: user-service

eureka:
  client:
    service-url:
      defaultZone: http://localhost:8761/eureka/
  instance:
    prefer-ip-address: true
```

---

## 6. Config Server

### 6.1. Config Server Setup

```xml
<!-- config-server/pom.xml -->
<dependencies>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-config-server</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
    </dependency>
</dependencies>
```

```java
// ConfigServerApplication.java
@SpringBootApplication
@EnableConfigServer
@EnableDiscoveryClient
public class ConfigServerApplication {
    public static void main(String[] args) {
        SpringApplication.run(ConfigServerApplication.class, args);
    }
}
```

```yaml
# application.yml
server:
  port: 8888

spring:
  application:
    name: config-server
  cloud:
    config:
      server:
        git:
          uri: https://github.com/your-org/config-repo
          default-label: main
          search-paths: '{application}'
        # Or use native (local file system)
        # native:
        #   search-locations: classpath:/config

eureka:
  client:
    service-url:
      defaultZone: http://localhost:8761/eureka/
```

### 6.2. Config Client

```xml
<!-- user-service/pom.xml (add this) -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-config</artifactId>
</dependency>
```

```yaml
# application.yml
spring:
  application:
    name: user-service
  config:
    import: optional:configserver:http://localhost:8888
  cloud:
    config:
      fail-fast: true
      retry:
        initial-interval: 1000
        max-attempts: 6
```

### 6.3. Config Repository Structure

```
config-repo/
├── application.yml              # Shared config for all services
├── user-service.yml             # User service specific
├── user-service-dev.yml         # User service dev profile
├── user-service-prod.yml        # User service prod profile
├── order-service.yml
└── order-service-prod.yml
```

---

## 7. API Gateway

### 7.1. Gateway Setup

```xml
<!-- api-gateway/pom.xml -->
<dependencies>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-gateway</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
    </dependency>
</dependencies>
```

```java
// ApiGatewayApplication.java
@SpringBootApplication
@EnableDiscoveryClient
public class ApiGatewayApplication {
    public static void main(String[] args) {
        SpringApplication.run(ApiGatewayApplication.class, args);
    }
}
```

```yaml
# application.yml
server:
  port: 8080

spring:
  application:
    name: api-gateway
  cloud:
    gateway:
      discovery:
        locator:
          enabled: true
          lower-case-service-id: true
      routes:
        - id: user-service
          uri: lb://user-service
          predicates:
            - Path=/api/users/**
          filters:
            - StripPrefix=1

        - id: order-service
          uri: lb://order-service
          predicates:
            - Path=/api/orders/**
          filters:
            - StripPrefix=1

eureka:
  client:
    service-url:
      defaultZone: http://localhost:8761/eureka/
```

### 7.2. Gateway Filters

```java
// Custom Filter
@Component
public class LoggingFilter implements GlobalFilter, Ordered {

    private static final Logger log = LoggerFactory.getLogger(LoggingFilter.class);

    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        log.info("Request path: {}", exchange.getRequest().getPath());
        log.info("Request method: {}", exchange.getRequest().getMethod());

        return chain.filter(exchange)
            .then(Mono.fromRunnable(() -> {
                log.info("Response status: {}",
                    exchange.getResponse().getStatusCode());
            }));
    }

    @Override
    public int getOrder() {
        return -1;  // Execute first
    }
}
```

---

## 8. Load Balancing

### 8.1. Spring Cloud LoadBalancer

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-loadbalancer</artifactId>
</dependency>
```

```java
// Using @LoadBalanced RestTemplate
@Configuration
public class RestTemplateConfig {

    @Bean
    @LoadBalanced
    public RestTemplate restTemplate() {
        return new RestTemplate();
    }
}

// Usage
@Service
public class OrderService {

    @Autowired
    private RestTemplate restTemplate;

    public User getUser(Long userId) {
        // Uses service name instead of host:port
        return restTemplate.getForObject(
            "http://user-service/api/users/" + userId,
            User.class
        );
    }
}
```

### 8.2. WebClient with LoadBalancer

```java
@Configuration
public class WebClientConfig {

    @Bean
    @LoadBalanced
    public WebClient.Builder webClientBuilder() {
        return WebClient.builder();
    }
}

// Usage
@Service
public class OrderService {

    private final WebClient webClient;

    public OrderService(WebClient.Builder webClientBuilder) {
        this.webClient = webClientBuilder
            .baseUrl("http://user-service")
            .build();
    }

    public Mono<User> getUser(Long userId) {
        return webClient.get()
            .uri("/api/users/{id}", userId)
            .retrieve()
            .bodyToMono(User.class);
    }
}
```

---

## 9. Complete Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                    COMPLETE SPRING CLOUD SETUP                       │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│   Browser/Client                                                     │
│        │                                                             │
│        ▼  :8080                                                      │
│   ┌──────────────┐                                                  │
│   │ API Gateway  │──────────────────────┐                           │
│   │ (Gateway)    │                      │                           │
│   └──────┬───────┘                      │                           │
│          │                              │                           │
│   ┌──────┴───────┐              ┌───────┴────────┐                  │
│   ▼              ▼              │                │                  │
│ :8081         :8082             ▼  :8761         ▼  :8888           │
│ ┌──────────┐ ┌──────────┐   ┌──────────┐   ┌──────────┐            │
│ │  User    │ │  Order   │   │  Eureka  │   │  Config  │            │
│ │ Service  │ │ Service  │   │  Server  │   │  Server  │            │
│ └────┬─────┘ └────┬─────┘   └──────────┘   └────┬─────┘            │
│      │            │                              │                  │
│      ▼            ▼                              ▼                  │
│ ┌─────────┐  ┌─────────┐                 ┌─────────────┐           │
│ │ User DB │  │Order DB │                 │ Git Config  │           │
│ │ (MySQL) │  │(Postgres)│                │    Repo     │           │
│ └─────────┘  └─────────┘                 └─────────────┘           │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘

Startup Order:
1. Eureka Server (8761)
2. Config Server (8888)
3. API Gateway (8080)
4. User Service (8081)
5. Order Service (8082)
```

---

## 10. Docker Compose

```yaml
# docker-compose.yml
version: '3.8'

services:
  eureka-server:
    build: ./service-discovery
    ports:
      - "8761:8761"
    networks:
      - microservices-net
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8761/actuator/health"]
      interval: 10s
      timeout: 5s
      retries: 5

  config-server:
    build: ./config-server
    ports:
      - "8888:8888"
    depends_on:
      eureka-server:
        condition: service_healthy
    environment:
      - EUREKA_CLIENT_SERVICEURL_DEFAULTZONE=http://eureka-server:8761/eureka/
    networks:
      - microservices-net

  api-gateway:
    build: ./api-gateway
    ports:
      - "8080:8080"
    depends_on:
      - eureka-server
      - config-server
    environment:
      - EUREKA_CLIENT_SERVICEURL_DEFAULTZONE=http://eureka-server:8761/eureka/
      - SPRING_CONFIG_IMPORT=optional:configserver:http://config-server:8888
    networks:
      - microservices-net

  user-service:
    build: ./user-service
    ports:
      - "8081:8081"
    depends_on:
      - eureka-server
      - config-server
      - mysql
    environment:
      - EUREKA_CLIENT_SERVICEURL_DEFAULTZONE=http://eureka-server:8761/eureka/
      - SPRING_DATASOURCE_URL=jdbc:mysql://mysql:3306/user_db
    networks:
      - microservices-net

  mysql:
    image: mysql:8.0
    environment:
      MYSQL_ROOT_PASSWORD: password
      MYSQL_DATABASE: user_db
    ports:
      - "3306:3306"
    volumes:
      - mysql-data:/var/lib/mysql
    networks:
      - microservices-net

networks:
  microservices-net:
    driver: bridge

volumes:
  mysql-data:
```

---

## 11. Bài tập thực hành

### Bài 1: Setup Basic Infrastructure
Tạo Eureka Server và 2 services (User, Order) register vào Eureka.

### Bài 2: Add Config Server
Setup Config Server với Git repository, services đọc config từ Config Server.

### Bài 3: API Gateway
Setup Spring Cloud Gateway với routes tới các services.

---

## 12. Key Takeaways

| Component | Purpose | Port (Default) |
|-----------|---------|----------------|
| Eureka Server | Service Discovery | 8761 |
| Config Server | Centralized Configuration | 8888 |
| API Gateway | Single Entry Point | 8080 |
| Microservice | Business Logic | 8081, 8082... |

```
Spring Cloud =
  Service Discovery (Eureka) +
  Config Management (Config Server) +
  API Gateway (Gateway) +
  Load Balancing (LoadBalancer) +
  Resilience (Resilience4j)
```

---

## Navigation

- [← Day 1: Microservices Introduction](./day-01-microservices-intro.md)
- [Day 3: Service Discovery →](./day-03-service-discovery.md)
