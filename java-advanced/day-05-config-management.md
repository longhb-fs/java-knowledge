# Day 5: Configuration Management

## Mục tiêu
- Spring Cloud Config
- Git-based configuration
- Dynamic configuration refresh
- Encryption & Security

---

## 1. Why Centralized Configuration?

### 1.1. Problems with Local Configuration

```
┌─────────────────────────────────────────────────────────────────┐
│            LOCAL CONFIGURATION PROBLEMS                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Service A                Service B                Service C   │
│   ┌─────────┐             ┌─────────┐             ┌─────────┐  │
│   │app.yml  │             │app.yml  │             │app.yml  │  │
│   │db.url=X │             │db.url=X │             │db.url=X │  │
│   │api.key=Y│             │api.key=Y│             │api.key=Y│  │
│   └─────────┘             └─────────┘             └─────────┘  │
│                                                                  │
│   Problems:                                                      │
│   ✗ Config duplicated across services                           │
│   ✗ Hard to change config (need redeploy)                       │
│   ✗ No version control for config changes                       │
│   ✗ Environment-specific configs messy                          │
│   ✗ Secrets exposed in config files                             │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│            CENTRALIZED CONFIGURATION                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│                    ┌─────────────────┐                          │
│                    │  Config Server  │                          │
│                    │  ┌───────────┐  │                          │
│                    │  │ Git Repo  │  │                          │
│                    │  │ app.yml   │  │                          │
│                    │  │ svc-a.yml │  │                          │
│                    │  │ svc-b.yml │  │                          │
│                    │  └───────────┘  │                          │
│                    └────────┬────────┘                          │
│                             │                                    │
│              ┌──────────────┼──────────────┐                    │
│              ▼              ▼              ▼                    │
│         Service A      Service B      Service C                 │
│                                                                  │
│   Benefits:                                                      │
│   ✓ Single source of truth                                      │
│   ✓ Dynamic refresh without restart                             │
│   ✓ Git versioning and audit trail                              │
│   ✓ Environment separation                                      │
│   ✓ Encrypted secrets                                           │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## 2. Spring Cloud Config Server

### 2.1. Setup Config Server

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
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-security</artifactId>
    </dependency>
</dependencies>
```

```java
@SpringBootApplication
@EnableConfigServer
@EnableDiscoveryClient
public class ConfigServerApplication {
    public static void main(String[] args) {
        SpringApplication.run(ConfigServerApplication.class, args);
    }
}
```

### 2.2. Git Backend Configuration

```yaml
# config-server/application.yml
server:
  port: 8888

spring:
  application:
    name: config-server
  cloud:
    config:
      server:
        git:
          # Public repo
          uri: https://github.com/your-org/config-repo
          default-label: main
          clone-on-start: true
          search-paths: '{application}'
          # Skip SSL verification (not recommended for production)
          skip-ssl-validation: false

          # Private repo with SSH
          # uri: git@github.com:your-org/config-repo.git
          # private-key: |
          #   -----BEGIN RSA PRIVATE KEY-----
          #   ...
          #   -----END RSA PRIVATE KEY-----

          # Private repo with HTTPS
          # uri: https://github.com/your-org/config-repo
          # username: ${GIT_USERNAME}
          # password: ${GIT_TOKEN}

        # Refresh rate
        git:
          refresh-rate: 5

  # Security
  security:
    user:
      name: configuser
      password: ${CONFIG_SERVER_PASSWORD:configpass}

eureka:
  client:
    service-url:
      defaultZone: http://localhost:8761/eureka/
```

### 2.3. Config Repository Structure

```
config-repo/
├── application.yml                    # Shared by all services
├── application-dev.yml                # Dev environment shared
├── application-prod.yml               # Prod environment shared
│
├── user-service/
│   ├── user-service.yml               # User service default
│   ├── user-service-dev.yml           # User service dev
│   └── user-service-prod.yml          # User service prod
│
├── order-service/
│   ├── order-service.yml
│   ├── order-service-dev.yml
│   └── order-service-prod.yml
│
└── gateway-service/
    ├── gateway-service.yml
    └── gateway-service-prod.yml
```

### 2.4. Sample Config Files

```yaml
# config-repo/application.yml (shared)
spring:
  jackson:
    default-property-inclusion: non_null
    date-format: yyyy-MM-dd HH:mm:ss

logging:
  level:
    root: INFO
    org.springframework.web: INFO

management:
  endpoints:
    web:
      exposure:
        include: health,info,refresh
```

```yaml
# config-repo/user-service/user-service.yml
server:
  port: 8081

spring:
  datasource:
    url: jdbc:mysql://localhost:3306/user_db
    username: root
    driver-class-name: com.mysql.cj.jdbc.Driver
  jpa:
    hibernate:
      ddl-auto: update
    show-sql: true

app:
  jwt:
    expiration: 86400000
  feature:
    email-verification: true
```

```yaml
# config-repo/user-service/user-service-prod.yml
spring:
  datasource:
    url: jdbc:mysql://prod-db.example.com:3306/user_db
    username: ${DB_USERNAME}
    password: '{cipher}encrypted-password'
    hikari:
      maximum-pool-size: 20
  jpa:
    show-sql: false

app:
  jwt:
    expiration: 3600000
  feature:
    email-verification: true
```

---

## 3. Config Client

### 3.1. Client Setup

```xml
<!-- user-service/pom.xml -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-config</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
```

```yaml
# user-service/application.yml
spring:
  application:
    name: user-service
  profiles:
    active: dev
  config:
    import: optional:configserver:http://localhost:8888
  cloud:
    config:
      # Fail if can't connect (recommended for production)
      fail-fast: true
      # Retry configuration
      retry:
        initial-interval: 1000
        max-attempts: 6
        max-interval: 2000
        multiplier: 1.1
      # Auth for config server
      username: configuser
      password: ${CONFIG_SERVER_PASSWORD:configpass}
      # Request specific label
      label: main

eureka:
  client:
    service-url:
      defaultZone: http://localhost:8761/eureka/
```

### 3.2. Config Discovery via Eureka

```yaml
# user-service/application.yml
spring:
  config:
    import: optional:configserver:
  cloud:
    config:
      discovery:
        enabled: true
        service-id: config-server
      fail-fast: true

eureka:
  client:
    service-url:
      defaultZone: http://localhost:8761/eureka/
```

### 3.3. Using Configuration Properties

```java
@Component
@ConfigurationProperties(prefix = "app")
@Validated
public class AppProperties {

    @NotNull
    private Jwt jwt = new Jwt();

    @NotNull
    private Feature feature = new Feature();

    // Nested class for jwt.*
    public static class Jwt {
        private String secret;
        private long expiration = 86400000;

        // Getters and setters
    }

    // Nested class for feature.*
    public static class Feature {
        private boolean emailVerification = true;
        private boolean smsNotification = false;

        // Getters and setters
    }

    // Getters and setters for main class
}

// Usage
@Service
public class AuthService {

    private final AppProperties appProperties;

    public AuthService(AppProperties appProperties) {
        this.appProperties = appProperties;
    }

    public String generateToken(User user) {
        long expiration = appProperties.getJwt().getExpiration();
        // Generate token...
    }
}
```

---

## 4. Dynamic Refresh

### 4.1. Enable Refresh

```yaml
# Enable refresh endpoint
management:
  endpoints:
    web:
      exposure:
        include: health,info,refresh,bus-refresh
```

```java
// Mark beans for refresh
@Component
@RefreshScope
public class DynamicConfigBean {

    @Value("${app.feature.email-verification}")
    private boolean emailVerification;

    public boolean isEmailVerificationEnabled() {
        return emailVerification;
    }
}

// ConfigurationProperties are automatically refreshable
@ConfigurationProperties(prefix = "app")
@RefreshScope  // Optional, properties are refreshed by default
public class AppProperties {
    // ...
}
```

### 4.2. Trigger Refresh

```bash
# Manual refresh for single instance
curl -X POST http://localhost:8081/actuator/refresh

# Response shows changed keys
["app.feature.email-verification", "app.jwt.expiration"]
```

### 4.3. Spring Cloud Bus (Broadcast Refresh)

```xml
<!-- Add to all services -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-bus-amqp</artifactId>
</dependency>
```

```yaml
spring:
  rabbitmq:
    host: localhost
    port: 5672
    username: guest
    password: guest

management:
  endpoints:
    web:
      exposure:
        include: health,info,refresh,bus-refresh,bus-env
```

```bash
# Broadcast refresh to all instances
curl -X POST http://localhost:8888/actuator/bus-refresh

# Refresh specific service
curl -X POST http://localhost:8888/actuator/bus-refresh/user-service:**
```

### 4.4. Webhook Auto-Refresh

```yaml
# config-server/application.yml
spring:
  cloud:
    config:
      server:
        git:
          uri: https://github.com/your-org/config-repo
          webhooks:
            enabled: true
```

```java
// Config Server webhook endpoint
@RestController
public class WebhookController {

    private final ApplicationEventPublisher publisher;

    @PostMapping("/webhook/github")
    public ResponseEntity<String> handleGithubWebhook(
            @RequestHeader("X-Hub-Signature") String signature,
            @RequestBody String payload) {

        // Verify webhook signature
        if (!verifySignature(signature, payload)) {
            return ResponseEntity.status(HttpStatus.FORBIDDEN).build();
        }

        // Trigger refresh
        publisher.publishEvent(new RefreshRemoteApplicationEvent(
            this, "config-server", "**"));

        return ResponseEntity.ok("Refresh triggered");
    }
}
```

---

## 5. Encryption

### 5.1. Symmetric Encryption

```yaml
# config-server/application.yml
encrypt:
  key: ${ENCRYPT_KEY:my-secret-encryption-key}
```

```bash
# Encrypt a value
curl -X POST http://localhost:8888/encrypt -d 'my-secret-password'
# Returns: d3f4e5a6b7c8d9e0f1a2b3c4d5e6f7a8b9c0d1e2f3a4b5c6d7e8f9

# Decrypt a value
curl -X POST http://localhost:8888/decrypt -d 'd3f4e5a6b7c8d9e0f1a2b3c4d5e6f7a8b9c0d1e2f3a4b5c6d7e8f9'
# Returns: my-secret-password
```

```yaml
# Use encrypted value in config
spring:
  datasource:
    password: '{cipher}d3f4e5a6b7c8d9e0f1a2b3c4d5e6f7a8b9c0d1e2f3a4b5c6d7e8f9'
```

### 5.2. Asymmetric Encryption (RSA)

```bash
# Generate keystore
keytool -genkeypair -alias config-server-key \
  -keyalg RSA -keysize 2048 \
  -keystore config-server.jks \
  -storepass changeit \
  -validity 365 \
  -dname "CN=Config Server,OU=Dev,O=Company,L=City,ST=State,C=US"
```

```yaml
# config-server/application.yml
encrypt:
  key-store:
    location: classpath:/config-server.jks
    password: changeit
    alias: config-server-key
    secret: changeit
```

### 5.3. Vault Integration

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-config-server</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-vault-config</artifactId>
</dependency>
```

```yaml
# config-server/application.yml
spring:
  cloud:
    config:
      server:
        vault:
          host: localhost
          port: 8200
          scheme: http
          authentication: TOKEN
          token: ${VAULT_TOKEN}
          kv-version: 2
          backend: secret
          default-key: application
```

---

## 6. Multiple Backends

### 6.1. Composite Backend

```yaml
spring:
  profiles:
    active: composite

  cloud:
    config:
      server:
        composite:
          # Git for application config
          - type: git
            uri: https://github.com/your-org/app-config
            search-paths: '{application}'
            order: 1

          # Vault for secrets
          - type: vault
            host: localhost
            port: 8200
            token: ${VAULT_TOKEN}
            order: 2

          # Native filesystem for local overrides
          - type: native
            search-locations: file:/config
            order: 3
```

### 6.2. Profile-based Backend Selection

```yaml
# Git for dev/staging
spring:
  config:
    activate:
      on-profile: dev,staging
  cloud:
    config:
      server:
        git:
          uri: https://github.com/your-org/config-repo
          default-label: develop

---
# Vault for production secrets
spring:
  config:
    activate:
      on-profile: prod
  cloud:
    config:
      server:
        composite:
          - type: git
            uri: https://github.com/your-org/config-repo
            default-label: main
          - type: vault
            host: vault.example.com
            port: 8200
```

---

## 7. Config Server API

### 7.1. Endpoints

```bash
# Get config for service
GET /{application}/{profile}
GET /{application}/{profile}/{label}

# Examples
GET /user-service/dev
GET /user-service/prod/main
GET /order-service/default

# Get specific file
GET /{application}/{profile}/{label}/{path}
GET /user-service/prod/main/application.yml

# Response format
/{application}-{profile}.yml
/{application}-{profile}.json
/{application}-{profile}.properties
```

### 7.2. Example Responses

```bash
# Request
curl http://localhost:8888/user-service/dev

# Response
{
  "name": "user-service",
  "profiles": ["dev"],
  "label": null,
  "version": "abc123",
  "state": null,
  "propertySources": [
    {
      "name": "https://github.com/.../user-service-dev.yml",
      "source": {
        "server.port": 8081,
        "spring.datasource.url": "jdbc:mysql://localhost:3306/user_db"
      }
    },
    {
      "name": "https://github.com/.../application.yml",
      "source": {
        "logging.level.root": "INFO"
      }
    }
  ]
}
```

---

## 8. High Availability

### 8.1. Multiple Config Server Instances

```yaml
# Client configuration with multiple servers
spring:
  cloud:
    config:
      uri:
        - http://config-server-1:8888
        - http://config-server-2:8888
        - http://config-server-3:8888
      fail-fast: true
      retry:
        initial-interval: 1000
        max-attempts: 6
```

### 8.2. Config Server with Discovery

```yaml
# Config clients find config server via Eureka
spring:
  config:
    import: optional:configserver:
  cloud:
    config:
      discovery:
        enabled: true
        service-id: config-server
```

### 8.3. Local Fallback

```yaml
# Fallback to local config if server unavailable
spring:
  config:
    import:
      - optional:configserver:http://localhost:8888
      - optional:file:./config/

  cloud:
    config:
      fail-fast: false  # Don't fail, use local
```

---

## 9. Best Practices

### 9.1. Security

```yaml
# Config Server Security
spring:
  security:
    user:
      name: ${CONFIG_USER:admin}
      password: ${CONFIG_PASSWORD}

# Enable HTTPS
server:
  ssl:
    key-store: classpath:keystore.p12
    key-store-password: ${SSL_PASSWORD}
    key-store-type: PKCS12
    key-alias: config-server
```

### 9.2. Config Organization

```
config-repo/
├── shared/
│   ├── application.yml           # Global defaults
│   ├── application-dev.yml       # Dev environment
│   └── application-prod.yml      # Prod environment
│
├── services/
│   ├── user-service/
│   │   └── user-service.yml
│   ├── order-service/
│   │   └── order-service.yml
│   └── gateway/
│       └── gateway.yml
│
├── secrets/                       # Encrypted secrets
│   ├── user-service-secrets.yml
│   └── order-service-secrets.yml
│
└── README.md                      # Documentation
```

### 9.3. Version Control

```bash
# Tag config releases
git tag -a config-v1.0.0 -m "Config release 1.0.0"
git push origin config-v1.0.0

# Use specific version
spring:
  cloud:
    config:
      label: config-v1.0.0
```

---

## 10. Troubleshooting

### 10.1. Common Issues

```java
// Issue: Config not loading
// Check 1: Is config server running?
curl http://localhost:8888/actuator/health

// Check 2: Can you fetch config?
curl http://localhost:8888/user-service/dev

// Check 3: Bootstrap vs application.yml
// Spring Boot 2.4+ uses spring.config.import
// Older versions need bootstrap.yml

// Issue: Encryption not working
// Check: Encrypt key is set
curl http://localhost:8888/encrypt/status
// Response: {"status":"OK"}

// Issue: Refresh not working
// Check 1: @RefreshScope annotation
// Check 2: Actuator endpoint enabled
curl -X POST http://localhost:8081/actuator/refresh
```

### 10.2. Debug Logging

```yaml
logging:
  level:
    org.springframework.cloud.config: DEBUG
    org.springframework.cloud.bus: DEBUG
    org.springframework.cloud.context.refresh: DEBUG
```

---

## 11. Bài tập thực hành

### Bài 1: Basic Config Server
Setup Config Server với Git backend, 2 services đọc config.

### Bài 2: Dynamic Refresh
Implement dynamic refresh với Spring Cloud Bus và RabbitMQ.

### Bài 3: Encryption
Encrypt database passwords, test decrypt trên client.

### Bài 4: High Availability
Setup 2 Config Server instances, test failover.

---

## 12. Key Takeaways

| Feature | Description |
|---------|-------------|
| Centralized Config | Single source of truth |
| Git Backend | Version control, audit trail |
| Dynamic Refresh | Update without restart |
| Encryption | Secure sensitive data |
| Spring Cloud Bus | Broadcast refresh |
| Discovery | Find config server via Eureka |

```
Config Resolution Order (highest priority first):
1. Command line arguments
2. Java System properties
3. OS environment variables
4. Profile-specific application properties (remote)
5. Application properties (remote)
6. Profile-specific application properties (local)
7. Application properties (local)
8. @PropertySource annotations
9. Default properties
```

---

## Navigation

- [← Day 4: API Gateway](./day-04-api-gateway.md)
- [Day 6: Inter-Service Communication →](./day-06-inter-service-communication.md)
