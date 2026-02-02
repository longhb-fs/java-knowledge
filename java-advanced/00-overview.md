# Java Advanced - Từ Developer đến Architect

## Tổng quan khóa học

Khóa học nâng cao dành cho Java Developer có 1-2 năm kinh nghiệm, hướng tới Senior Developer / Architect level.

## Yêu cầu đầu vào

- ✅ Đã hoàn thành Java Fundamentals
- ✅ Có kinh nghiệm với Spring Boot
- ✅ Hiểu biết cơ bản về Docker
- ✅ Đã làm dự án thực tế

---

## Cấu trúc khóa học

### Module 01: Microservices Architecture (Day 1-10)

| Day | Chủ đề |
|-----|--------|
| 1 | [Microservices Introduction](./day-01-microservices-intro.md) |
| 2 | [Spring Cloud Overview](./day-02-spring-cloud-overview.md) |
| 3 | [Service Discovery](./day-03-service-discovery.md) |
| 4 | [API Gateway](./day-04-api-gateway.md) |
| 5 | [Configuration Management](./day-05-config-management.md) |
| 6 | [Inter-Service Communication](./day-06-inter-service-communication.md) |
| 7 | [Resilience Patterns](./day-07-resilience-patterns.md) |
| 8 | [Distributed Transactions](./day-08-distributed-transactions.md) |
| 9 | [Security in Microservices](./day-09-microservices-security.md) |
| 10 | [Microservices Testing](./day-10-microservices-testing.md) |

### Module 02: Message Queues & Event-Driven (Day 11-20)

| Day | Chủ đề |
|-----|--------|
| 11 | [Kafka Fundamentals](./day-11-kafka-fundamentals.md) |
| 12 | [Spring Kafka](./day-12-spring-kafka.md) |
| 13 | [Kafka Advanced](./day-13-kafka-advanced.md) |
| 14 | [Kafka Patterns](./day-14-kafka-patterns.md) |
| 15 | [Kafka Operations](./day-15-kafka-operations.md) |
| 16 | [RabbitMQ Fundamentals](./day-16-rabbitmq-fundamentals.md) |
| 17 | [Spring AMQP](./day-17-spring-amqp.md) |
| 18 | [RabbitMQ Patterns](./day-18-rabbitmq-patterns.md) |
| 19 | [Comparing Message Brokers](./day-19-comparing-brokers.md) |
| 20 | [Event-Driven Architecture](./day-20-event-driven-architecture.md) |

### Module 03: Cloud Native & Kubernetes (Day 21-30)

| Day | Chủ đề |
|-----|--------|
| 21 | [Kubernetes Introduction](./day-21-kubernetes-intro.md) |
| 22 | [Kubernetes Workloads](./day-22-kubernetes-workloads.md) |
| 23 | [Kubernetes Networking](./day-23-kubernetes-networking.md) |
| 24 | [Kubernetes Configuration](./day-24-kubernetes-config.md) |
| 25 | [Kubernetes Storage](./day-25-kubernetes-storage.md) |
| 26 | [Helm & GitOps](./day-26-helm-gitops.md) |
| 27 | [Service Mesh (Istio)](./day-27-service-mesh.md) |
| 28 | [Cloud Platforms](./day-28-cloud-platforms.md) |
| 29 | [Kubernetes Observability](./day-29-kubernetes-observability.md) |
| 30 | [Production Best Practices](./day-30-k8s-best-practices.md) |

### Module 04: Performance & Optimization (Day 31-35)

| Day | Chủ đề |
|-----|--------|
| 31 | [JVM Tuning](./day-31-jvm-tuning.md) |
| 32 | [Application Profiling](./day-32-application-profiling.md) |
| 33 | [Database Performance](./day-33-database-performance.md) |
| 34 | [Caching Strategies](./day-34-caching-strategies.md) |
| 35 | [Load Testing](./day-35-load-testing.md) |

### Module 05: Advanced Patterns & Architecture (Day 36-40)

| Day | Chủ đề |
|-----|--------|
| 36 | [Design Patterns in Enterprise](./day-36-enterprise-patterns.md) |
| 37 | [Domain-Driven Design](./day-37-ddd.md) |
| 38 | [CQRS & Event Sourcing](./day-38-cqrs-event-sourcing.md) |
| 39 | [Clean Architecture](./day-39-clean-architecture.md) |
| 40 | [API Design](./day-40-api-design.md) |

### Module 06: DevOps & SRE (Day 41-45)

| Day | Chủ đề |
|-----|--------|
| 41 | [CI/CD Advanced](./day-41-cicd-advanced.md) |
| 42 | [Infrastructure as Code](./day-42-iac.md) |
| 43 | [Observability Deep Dive](./day-43-observability.md) |
| 44 | [Security & Compliance](./day-44-security-compliance.md) |
| 45 | [Site Reliability Engineering](./day-45-sre.md) |

---

## Tech Stack

| Category | Technologies |
|----------|--------------|
| Framework | Spring Boot 3.x, Spring Cloud 2023.x |
| Messaging | Apache Kafka, RabbitMQ |
| Container | Docker, Kubernetes |
| Service Mesh | Istio |
| Observability | Prometheus, Grafana, Jaeger |
| CI/CD | GitHub Actions, ArgoCD |
| Cloud | AWS/GCP/Azure basics |

---

## Learning Path

```
┌─────────────────────────────────────────────────────────────┐
│                    JAVA ADVANCED PATH                        │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│   Module 1-2: Microservices + Messaging (20 days)           │
│   ├── Break monolith into services                          │
│   ├── Async communication with Kafka/RabbitMQ               │
│   └── Resilience patterns                                   │
│                        ↓                                     │
│   Module 3: Kubernetes (10 days)                            │
│   ├── Deploy services to K8s                                │
│   ├── Service mesh with Istio                               │
│   └── Cloud-native patterns                                 │
│                        ↓                                     │
│   Module 4-5: Performance + Architecture (10 days)          │
│   ├── JVM tuning, profiling                                 │
│   ├── DDD, CQRS, Event Sourcing                             │
│   └── Clean Architecture                                    │
│                        ↓                                     │
│   Module 6: DevOps & SRE (5 days)                           │
│   ├── CI/CD pipelines                                       │
│   ├── Infrastructure as Code                                │
│   └── Site Reliability Engineering                          │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## Bắt đầu

➡️ [Day 1: Microservices Introduction](./day-01-microservices-intro.md)
