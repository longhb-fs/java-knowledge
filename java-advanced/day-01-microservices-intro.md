# Day 1: Microservices Introduction

## Mục tiêu
- Hiểu Monolith vs Microservices
- Microservices benefits và challenges
- Domain-Driven Design basics
- Service decomposition strategies

---

## 1. Monolith vs Microservices

### 1.1. Monolithic Architecture

```
┌─────────────────────────────────────────────────────────┐
│                    MONOLITH APPLICATION                  │
├─────────────────────────────────────────────────────────┤
│  ┌─────────┐  ┌─────────┐  ┌─────────┐  ┌─────────┐    │
│  │  User   │  │  Order  │  │ Product │  │ Payment │    │
│  │ Module  │  │ Module  │  │ Module  │  │ Module  │    │
│  └────┬────┘  └────┬────┘  └────┬────┘  └────┬────┘    │
│       │            │            │            │          │
│  ┌────┴────────────┴────────────┴────────────┴────┐    │
│  │              Shared Database                    │    │
│  └─────────────────────────────────────────────────┘    │
│                                                          │
│  Single Deployment Unit (WAR/JAR)                       │
└─────────────────────────────────────────────────────────┘
```

**Đặc điểm:**
- Tất cả code trong 1 codebase
- Shared database
- Single deployment
- Tightly coupled

**Ưu điểm:**
- Dễ develop ban đầu
- Dễ test end-to-end
- Dễ deploy (1 artifact)
- Không có network latency giữa modules

**Nhược điểm:**
- Khó scale từng phần
- Một lỗi nhỏ có thể crash toàn bộ
- Tech stack bị lock
- Team lớn khó làm việc song song

### 1.2. Microservices Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                      MICROSERVICES                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   ┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐ │
│   │  User    │    │  Order   │    │ Product  │    │ Payment  │ │
│   │ Service  │    │ Service  │    │ Service  │    │ Service  │ │
│   └────┬─────┘    └────┬─────┘    └────┬─────┘    └────┬─────┘ │
│        │               │               │               │        │
│   ┌────┴────┐    ┌────┴────┐    ┌────┴────┐    ┌────┴────┐    │
│   │ User DB │    │Order DB │    │Product  │    │Payment  │    │
│   │ (MySQL) │    │(Postgres)│   │DB(Mongo)│    │DB(MySQL)│    │
│   └─────────┘    └─────────┘    └─────────┘    └─────────┘    │
│                                                                  │
│   Each service: Independent deployment, Own database            │
└─────────────────────────────────────────────────────────────────┘
```

**Đặc điểm:**
- Mỗi service là 1 codebase riêng
- Database per service
- Independent deployment
- Loosely coupled
- Communicate qua API/Message

---

## 2. Microservices Benefits & Challenges

### 2.1. Benefits

| Benefit | Giải thích |
|---------|------------|
| **Independent Scaling** | Scale từng service theo nhu cầu |
| **Technology Freedom** | Mỗi service có thể dùng tech khác nhau |
| **Fault Isolation** | Lỗi 1 service không crash toàn bộ |
| **Team Autonomy** | Teams làm việc độc lập |
| **Faster Deployment** | Deploy từng service, không cần deploy all |
| **Easier to Understand** | Codebase nhỏ, dễ hiểu |

### 2.2. Challenges

| Challenge | Giải thích |
|-----------|------------|
| **Distributed Complexity** | Network calls, latency, failures |
| **Data Consistency** | No ACID across services |
| **Operational Overhead** | Nhiều services cần monitor |
| **Testing Complexity** | Integration tests khó hơn |
| **Debugging** | Trace request across services |
| **Initial Setup** | Cần infrastructure phức tạp hơn |

### 2.3. When to Use Microservices?

```
✅ NÊN dùng Microservices khi:
- Team lớn (>10 developers)
- Cần scale từng phần độc lập
- Các domain rõ ràng, ít phụ thuộc
- Cần deploy thường xuyên
- Có đủ infrastructure/DevOps support

❌ KHÔNG NÊN dùng khi:
- Team nhỏ (<5 developers)
- Startup mới, domain chưa rõ
- Không có DevOps experience
- Monolith vẫn đang hoạt động tốt
```

---

## 3. Domain-Driven Design (DDD) Basics

### 3.1. Core Concepts

```
┌─────────────────────────────────────────────────────────┐
│                    DDD CONCEPTS                          │
├─────────────────────────────────────────────────────────┤
│                                                          │
│  STRATEGIC DDD (High-level)                             │
│  ├── Domain: Lĩnh vực business                          │
│  ├── Subdomain: Phần nhỏ của domain                     │
│  ├── Bounded Context: Ranh giới của model               │
│  └── Ubiquitous Language: Ngôn ngữ chung                │
│                                                          │
│  TACTICAL DDD (Low-level)                               │
│  ├── Entity: Object có identity                         │
│  ├── Value Object: Object không có identity             │
│  ├── Aggregate: Cluster of entities                     │
│  ├── Repository: Data access abstraction                │
│  └── Domain Service: Business logic                     │
│                                                          │
└─────────────────────────────────────────────────────────┘
```

### 3.2. Bounded Context

```
E-Commerce Domain
├── User Context          → User Service
│   ├── User
│   ├── Profile
│   └── Authentication
│
├── Catalog Context       → Product Service
│   ├── Product
│   ├── Category
│   └── Inventory
│
├── Order Context         → Order Service
│   ├── Order
│   ├── OrderItem
│   └── Shipping
│
└── Payment Context       → Payment Service
    ├── Payment
    ├── Transaction
    └── Refund
```

**Key Insight:** Mỗi Bounded Context = 1 Microservice

### 3.3. Context Mapping

```
┌─────────────┐         ┌─────────────┐
│   Order     │◄───────►│   Payment   │
│   Context   │ Partner │   Context   │
└──────┬──────┘         └─────────────┘
       │
       │ Customer/
       │ Supplier
       ▼
┌─────────────┐
│  Inventory  │
│   Context   │
└─────────────┘
```

**Relationship patterns:**
- **Partnership**: Hai teams cooperate
- **Customer/Supplier**: Upstream/Downstream relationship
- **Conformist**: Downstream follows upstream
- **Anti-Corruption Layer**: Translate between contexts

---

## 4. Service Decomposition Strategies

### 4.1. Decompose by Business Capability

```java
// Identify business capabilities
// E-commerce capabilities:
// - User Management
// - Product Catalog
// - Order Processing
// - Payment Processing
// - Shipping
// - Notification

// Each capability = 1 Service
@SpringBootApplication
public class UserServiceApplication { }

@SpringBootApplication
public class OrderServiceApplication { }

@SpringBootApplication
public class PaymentServiceApplication { }
```

### 4.2. Decompose by Subdomain

```
Core Subdomain (competitive advantage)
├── Order Processing      → Custom development
└── Recommendation Engine → Custom development

Supporting Subdomain (necessary but not core)
├── User Management       → Can use existing solution
└── Notification          → Can use existing solution

Generic Subdomain (commodity)
├── Payment Gateway       → Use third-party
└── Email Service         → Use third-party (SendGrid)
```

### 4.3. Strangler Fig Pattern (Migration)

```
Phase 1: Identify module to extract
┌─────────────────────────────────────┐
│           MONOLITH                   │
│  ┌─────┐ ┌─────┐ ┌─────┐ ┌─────┐   │
│  │User │ │Order│ │ Pay │ │Notif│   │
│  └─────┘ └─────┘ └─────┘ └─────┘   │
└─────────────────────────────────────┘

Phase 2: Create new service, route traffic
┌─────────────────────────────────────┐
│           MONOLITH                   │
│  ┌─────┐ ┌─────┐         ┌─────┐   │
│  │User │ │Order│ ──────► │Notif│   │
│  └─────┘ └─────┘         └─────┘   │
└─────────────────────────────────────┘
                    │
                    ▼
              ┌──────────┐
              │ Payment  │
              │ Service  │
              └──────────┘

Phase 3: Remove old module, fully migrate
┌─────────────────────────────────────┐
│           MONOLITH                   │
│  ┌─────┐ ┌─────┐         ┌─────┐   │
│  │User │ │Order│         │Notif│   │
│  └─────┘ └─────┘         └─────┘   │
└─────────────────────────────────────┘
        │       │               │
        ▼       ▼               ▼
   ┌────────┐ ┌────────┐ ┌──────────┐
   │Payment │ │Shipping│ │Notif Svc │
   │Service │ │Service │ │(new)     │
   └────────┘ └────────┘ └──────────┘
```

---

## 5. Sample Microservices Structure

### 5.1. Project Structure

```
ecommerce-microservices/
├── api-gateway/
│   └── src/main/java/...
├── service-discovery/
│   └── src/main/java/...
├── config-server/
│   └── src/main/java/...
├── user-service/
│   ├── src/main/java/
│   │   └── com/example/user/
│   │       ├── UserServiceApplication.java
│   │       ├── controller/
│   │       ├── service/
│   │       ├── repository/
│   │       └── model/
│   └── src/main/resources/
│       └── application.yml
├── order-service/
│   └── ...
├── product-service/
│   └── ...
├── payment-service/
│   └── ...
└── docker-compose.yml
```

### 5.2. Simple Service Example

```java
// user-service/UserServiceApplication.java
@SpringBootApplication
@EnableDiscoveryClient
public class UserServiceApplication {
    public static void main(String[] args) {
        SpringApplication.run(UserServiceApplication.class, args);
    }
}

// User Entity
@Entity
@Table(name = "users")
public class User {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    private String username;
    private String email;
    private String password;
    // getters, setters
}

// User Controller
@RestController
@RequestMapping("/api/users")
public class UserController {

    @Autowired
    private UserService userService;

    @GetMapping("/{id}")
    public ResponseEntity<User> getUser(@PathVariable Long id) {
        return ResponseEntity.ok(userService.findById(id));
    }

    @PostMapping
    public ResponseEntity<User> createUser(@RequestBody User user) {
        return ResponseEntity.status(HttpStatus.CREATED)
            .body(userService.create(user));
    }
}
```

### 5.3. application.yml

```yaml
spring:
  application:
    name: user-service
  datasource:
    url: jdbc:mysql://localhost:3306/user_db
    username: root
    password: password

server:
  port: 8081

eureka:
  client:
    service-url:
      defaultZone: http://localhost:8761/eureka/
```

---

## 6. Communication Patterns

### 6.1. Synchronous (REST/gRPC)

```java
// Order Service calls User Service
@Service
public class OrderService {

    @Autowired
    private RestTemplate restTemplate;

    public Order createOrder(OrderRequest request) {
        // Call User Service to validate user
        User user = restTemplate.getForObject(
            "http://user-service/api/users/" + request.getUserId(),
            User.class
        );

        if (user == null) {
            throw new UserNotFoundException();
        }

        // Create order
        return orderRepository.save(new Order(request, user));
    }
}
```

### 6.2. Asynchronous (Message Queue)

```java
// Order Service publishes event
@Service
public class OrderService {

    @Autowired
    private KafkaTemplate<String, OrderEvent> kafkaTemplate;

    public Order createOrder(OrderRequest request) {
        Order order = orderRepository.save(new Order(request));

        // Publish event
        kafkaTemplate.send("order-created", new OrderCreatedEvent(order));

        return order;
    }
}

// Payment Service subscribes to event
@Service
public class PaymentService {

    @KafkaListener(topics = "order-created")
    public void handleOrderCreated(OrderCreatedEvent event) {
        // Process payment
        processPayment(event.getOrderId(), event.getAmount());
    }
}
```

---

## 7. Bài tập thực hành

### Bài 1: Identify Bounded Contexts
Phân tích một hệ thống booking hotel và xác định các bounded contexts.

### Bài 2: Service Decomposition
Cho một monolith e-commerce, đề xuất cách decompose thành microservices.

### Bài 3: Create Simple Services
Tạo 2 Spring Boot services (User và Order) communicate qua REST.

---

## 8. Key Takeaways

| Concept | Ghi nhớ |
|---------|---------|
| Microservices | Small, independent, loosely coupled |
| Bounded Context | 1 Context ≈ 1 Service |
| Database per Service | Không share database |
| Communication | Sync (REST/gRPC) hoặc Async (Kafka/RabbitMQ) |
| Decomposition | By business capability hoặc subdomain |

---

## Navigation

- [← Overview](./00-overview.md)
- [Day 2: Spring Cloud Overview →](./day-02-spring-cloud-overview.md)
