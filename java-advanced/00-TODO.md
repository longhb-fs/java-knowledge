# Java Advanced - TODO Outline

## Mục tiêu khóa học
Nâng cao kiến thức sau khi đã học Spring Boot, hướng tới Senior Developer / Architect level.

## Yêu cầu đầu vào
- Đã hoàn thành Java Core
- Đã hoàn thành Spring Boot E-Commerce course
- Có kinh nghiệm làm dự án thực tế

---

## Module 01: Microservices Architecture (2 weeks)

### Week 01: Microservices Fundamentals

#### Day 01: Microservices Introduction
- [ ] Monolith vs Microservices
- [ ] Microservices benefits và challenges
- [ ] Domain-Driven Design (DDD) basics
- [ ] Bounded contexts
- [ ] Service decomposition strategies

#### Day 02: Spring Cloud Overview
- [ ] Spring Cloud ecosystem
- [ ] Spring Cloud Netflix (legacy)
- [ ] Spring Cloud Gateway
- [ ] Spring Cloud Config
- [ ] Spring Cloud Sleuth/Micrometer

#### Day 03: Service Discovery
- [ ] Service registry pattern
- [ ] Netflix Eureka
- [ ] Consul
- [ ] Service registration
- [ ] Client-side load balancing

#### Day 04: API Gateway
- [ ] Gateway pattern
- [ ] Spring Cloud Gateway
- [ ] Route configuration
- [ ] Filters (Pre, Post, Global)
- [ ] Rate limiting
- [ ] Circuit breaker integration

#### Day 05: Configuration Management
- [ ] Spring Cloud Config Server
- [ ] Config repository (Git)
- [ ] Environment-specific configs
- [ ] Config encryption
- [ ] Dynamic config refresh

### Week 02: Microservices Patterns

#### Day 06: Inter-Service Communication
- [ ] Synchronous (REST, gRPC)
- [ ] Asynchronous (Message Queue)
- [ ] OpenFeign client
- [ ] WebClient reactive
- [ ] gRPC basics

#### Day 07: Resilience Patterns
- [ ] Circuit Breaker (Resilience4j)
- [ ] Retry pattern
- [ ] Bulkhead pattern
- [ ] Rate limiter
- [ ] Timeout handling
- [ ] Fallback strategies

#### Day 08: Distributed Transactions
- [ ] SAGA pattern
- [ ] Choreography vs Orchestration
- [ ] Compensating transactions
- [ ] Eventual consistency
- [ ] Outbox pattern

#### Day 09: Security in Microservices
- [ ] OAuth2 với Keycloak
- [ ] Token propagation
- [ ] Service-to-service auth
- [ ] API Gateway security
- [ ] Zero Trust architecture

#### Day 10: Microservices Testing
- [ ] Contract testing (Pact)
- [ ] Consumer-driven contracts
- [ ] Integration testing strategies
- [ ] Service virtualization
- [ ] Chaos engineering basics

---

## Module 02: Message Queues & Event-Driven (2 weeks)

### Week 03: Apache Kafka

#### Day 11: Kafka Fundamentals
- [ ] Kafka architecture
- [ ] Topics, Partitions, Offsets
- [ ] Producers và Consumers
- [ ] Consumer Groups
- [ ] Kafka cluster setup (Docker)

#### Day 12: Spring Kafka
- [ ] Spring Kafka configuration
- [ ] KafkaTemplate
- [ ] @KafkaListener
- [ ] Message serialization (JSON, Avro)
- [ ] Error handling

#### Day 13: Kafka Advanced
- [ ] Exactly-once semantics
- [ ] Transactions
- [ ] Kafka Streams basics
- [ ] KSQL introduction
- [ ] Schema Registry (Avro)

#### Day 14: Kafka Patterns
- [ ] Event sourcing với Kafka
- [ ] CQRS pattern
- [ ] Event-driven saga
- [ ] Dead letter queues
- [ ] Retry topics

#### Day 15: Kafka Operations
- [ ] Monitoring với JMX
- [ ] Kafka Manager / UI
- [ ] Performance tuning
- [ ] Partition strategies
- [ ] Data retention policies

### Week 04: RabbitMQ & Other Messaging

#### Day 16: RabbitMQ Fundamentals
- [ ] AMQP protocol
- [ ] Exchanges, Queues, Bindings
- [ ] Exchange types (Direct, Topic, Fanout)
- [ ] Message acknowledgment
- [ ] RabbitMQ setup (Docker)

#### Day 17: Spring AMQP
- [ ] RabbitTemplate
- [ ] @RabbitListener
- [ ] Message converters
- [ ] Error handling
- [ ] Dead letter exchanges

#### Day 18: RabbitMQ Patterns
- [ ] Work queues
- [ ] Publish/Subscribe
- [ ] Routing
- [ ] Topics
- [ ] RPC pattern

#### Day 19: Comparing Message Brokers
- [ ] Kafka vs RabbitMQ vs ActiveMQ
- [ ] Use case selection
- [ ] Performance comparison
- [ ] Redis Pub/Sub
- [ ] Amazon SQS/SNS basics

#### Day 20: Event-Driven Architecture
- [ ] Event storming
- [ ] Domain events
- [ ] Integration events
- [ ] Event versioning
- [ ] Event schema evolution

---

## Module 03: Cloud Native & Kubernetes (2 weeks)

### Week 05: Kubernetes Fundamentals

#### Day 21: Kubernetes Introduction
- [ ] Container orchestration
- [ ] Kubernetes architecture
- [ ] Pods, Nodes, Clusters
- [ ] kubectl basics
- [ ] Minikube/Kind setup

#### Day 22: Kubernetes Workloads
- [ ] Deployments
- [ ] ReplicaSets
- [ ] StatefulSets
- [ ] DaemonSets
- [ ] Jobs và CronJobs

#### Day 23: Kubernetes Networking
- [ ] Services (ClusterIP, NodePort, LoadBalancer)
- [ ] Ingress controllers
- [ ] Network policies
- [ ] Service mesh basics
- [ ] DNS trong Kubernetes

#### Day 24: Kubernetes Configuration
- [ ] ConfigMaps
- [ ] Secrets
- [ ] Environment variables
- [ ] Volume mounts
- [ ] External secrets

#### Day 25: Kubernetes Storage
- [ ] Volumes
- [ ] PersistentVolumes (PV)
- [ ] PersistentVolumeClaims (PVC)
- [ ] StorageClasses
- [ ] StatefulSet với storage

### Week 06: Kubernetes Advanced & Cloud

#### Day 26: Helm & GitOps
- [ ] Helm charts
- [ ] Chart development
- [ ] Helm repositories
- [ ] ArgoCD introduction
- [ ] GitOps workflow

#### Day 27: Service Mesh (Istio)
- [ ] Service mesh concepts
- [ ] Istio architecture
- [ ] Traffic management
- [ ] Security (mTLS)
- [ ] Observability

#### Day 28: Cloud Platforms
- [ ] AWS EKS basics
- [ ] Google GKE basics
- [ ] Azure AKS basics
- [ ] Managed services comparison
- [ ] Cost optimization

#### Day 29: Kubernetes Observability
- [ ] Prometheus Operator
- [ ] Grafana dashboards
- [ ] Loki for logging
- [ ] Jaeger/Tempo for tracing
- [ ] Alertmanager

#### Day 30: Production Best Practices
- [ ] Resource limits và requests
- [ ] Horizontal Pod Autoscaler
- [ ] Pod Disruption Budgets
- [ ] Security contexts
- [ ] RBAC

---

## Module 04: Performance & Optimization (1 week)

### Week 07: Performance Engineering

#### Day 31: JVM Tuning
- [ ] JVM memory model
- [ ] Garbage Collection algorithms
- [ ] GC tuning flags
- [ ] JVM profiling tools
- [ ] Memory leak detection

#### Day 32: Application Profiling
- [ ] JProfiler / VisualVM
- [ ] Async Profiler
- [ ] Flame graphs
- [ ] CPU profiling
- [ ] Memory profiling

#### Day 33: Database Performance
- [ ] Query optimization
- [ ] Index strategies
- [ ] Connection pooling tuning
- [ ] Read replicas
- [ ] Database sharding concepts

#### Day 34: Caching Strategies
- [ ] Cache patterns (Aside, Through, Behind)
- [ ] Distributed caching (Redis Cluster)
- [ ] Cache invalidation strategies
- [ ] Hibernate second-level cache
- [ ] CDN caching

#### Day 35: Load Testing
- [ ] JMeter
- [ ] Gatling
- [ ] K6
- [ ] Load testing strategies
- [ ] Performance baselines

---

## Module 05: Advanced Patterns & Architecture (1 week)

### Week 08: Design Patterns & DDD

#### Day 36: Design Patterns in Java
- [ ] Creational patterns (Factory, Builder, Singleton)
- [ ] Structural patterns (Adapter, Decorator, Proxy)
- [ ] Behavioral patterns (Strategy, Observer, Command)
- [ ] Patterns trong Spring Framework
- [ ] Anti-patterns to avoid

#### Day 37: Domain-Driven Design
- [ ] Strategic DDD
- [ ] Tactical DDD patterns
- [ ] Aggregates và Aggregate Roots
- [ ] Value Objects
- [ ] Domain Services vs Application Services

#### Day 38: CQRS & Event Sourcing
- [ ] CQRS pattern deep dive
- [ ] Event sourcing implementation
- [ ] Axon Framework
- [ ] Event store
- [ ] Projections và Read models

#### Day 39: Clean Architecture
- [ ] Hexagonal Architecture
- [ ] Onion Architecture
- [ ] Ports và Adapters
- [ ] Dependency inversion
- [ ] Use case driven development

#### Day 40: API Design
- [ ] REST maturity model
- [ ] GraphQL introduction
- [ ] gRPC deep dive
- [ ] API versioning strategies
- [ ] OpenAPI/Swagger best practices

---

## Module 06: DevOps & SRE (1 week)

### Week 09: DevOps Practices

#### Day 41: CI/CD Advanced
- [ ] GitHub Actions advanced
- [ ] GitLab CI/CD
- [ ] Jenkins pipelines
- [ ] ArgoCD workflows
- [ ] Deployment strategies (Blue/Green, Canary)

#### Day 42: Infrastructure as Code
- [ ] Terraform basics
- [ ] Terraform với AWS/GCP
- [ ] Ansible basics
- [ ] Pulumi introduction
- [ ] GitOps với IaC

#### Day 43: Observability Deep Dive
- [ ] OpenTelemetry
- [ ] Distributed tracing
- [ ] Log aggregation patterns
- [ ] Metrics cardinality
- [ ] SLOs và SLIs

#### Day 44: Security & Compliance
- [ ] OWASP Top 10
- [ ] Security scanning (SAST, DAST)
- [ ] Container security
- [ ] Secrets management (Vault)
- [ ] Compliance (SOC2, GDPR)

#### Day 45: Site Reliability Engineering
- [ ] SRE principles
- [ ] Error budgets
- [ ] Incident management
- [ ] Postmortems
- [ ] Runbooks

---

## Module 07: Specialized Topics (Electives)

### Reactive Programming
- [ ] Project Reactor
- [ ] WebFlux
- [ ] R2DBC
- [ ] Reactive Streams
- [ ] Backpressure handling

### GraphQL
- [ ] GraphQL basics
- [ ] Spring for GraphQL
- [ ] Schema design
- [ ] DataLoaders (N+1 problem)
- [ ] Subscriptions

### gRPC
- [ ] Protocol Buffers
- [ ] gRPC-Java
- [ ] Streaming types
- [ ] Error handling
- [ ] gRPC-Web

### NoSQL Databases
- [ ] MongoDB với Spring Data
- [ ] Cassandra basics
- [ ] Neo4j (Graph DB)
- [ ] DynamoDB basics

### Big Data Basics
- [ ] Apache Spark overview
- [ ] Spark với Java
- [ ] Data pipelines
- [ ] ETL patterns

---

## Certifications Path

### Recommended Certifications
- [ ] Oracle Certified Professional Java SE
- [ ] Spring Professional Certification
- [ ] AWS Solutions Architect
- [ ] Kubernetes (CKA/CKAD)
- [ ] Kafka Certification

---

## Project Ideas

### Capstone Projects
1. **E-Commerce Microservices**
   - Decompose monolith thành microservices
   - Implement SAGA pattern cho orders
   - Event sourcing cho inventory

2. **Real-time Analytics Platform**
   - Kafka Streams processing
   - Real-time dashboards
   - Anomaly detection

3. **Multi-tenant SaaS Platform**
   - Tenant isolation
   - Custom domain support
   - Usage-based billing

---

## Tiến độ

| Module | Topics | Status | Ngày hoàn thành |
|--------|--------|--------|-----------------|
| Module 01 | Microservices | ⬜ Pending | - |
| Module 02 | Message Queues | ⬜ Pending | - |
| Module 03 | Kubernetes | ⬜ Pending | - |
| Module 04 | Performance | ⬜ Pending | - |
| Module 05 | Patterns & DDD | ⬜ Pending | - |
| Module 06 | DevOps & SRE | ⬜ Pending | - |
| Module 07 | Electives | ⬜ Pending | - |

---

## Learning Path Summary

```
┌─────────────────────────────────────────────────────────────┐
│                    JAVA LEARNING PATH                        │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│   Level 1: Java Core (30 days)                              │
│   ├── OOP, Collections, Streams                             │
│   ├── Concurrency, I/O                                      │
│   └── Modern Java features                                  │
│            │                                                 │
│            ▼                                                 │
│   Level 2: Spring Boot E-Commerce (30 days)                 │
│   ├── Spring Boot, Security, JPA                            │
│   ├── REST API, Testing                                     │
│   └── Docker, Monitoring                                    │
│            │                                                 │
│            ▼                                                 │
│   Level 3: Java Advanced (45 days)          ◄── YOU ARE HERE│
│   ├── Microservices, Kafka                                  │
│   ├── Kubernetes, Cloud Native                              │
│   └── Performance, Architecture                             │
│            │                                                 │
│            ▼                                                 │
│   Level 4: Specialization                                   │
│   ├── Reactive / GraphQL / gRPC                             │
│   ├── Big Data / ML                                         │
│   └── Architect / Tech Lead path                            │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

**Ghi chú:** Course này dành cho developer đã có 1-2 năm kinh nghiệm với Java/Spring Boot muốn lên Senior level.
