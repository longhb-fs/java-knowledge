# Day 21: Docker Fundamentals for Java Applications

## Mục tiêu học tập
- Hiểu Docker architecture và core concepts
- Viết Dockerfile tối ưu cho Java/Spring Boot applications
- Implement multi-stage builds để giảm image size
- Cấu hình Docker Compose cho development
- Apply security best practices

## 1. Docker Architecture

### 1.1 Core Concepts

```
Docker Architecture Overview
════════════════════════════

┌─────────────────────────────────────────────────────────────────┐
│                         Docker Host                              │
├─────────────────────────────────────────────────────────────────┤
│  ┌─────────────────────────────────────────────────────────────┐│
│  │                      Docker Daemon                          ││
│  │  ┌──────────────────────────────────────────────────────┐  ││
│  │  │                   Containers                          │  ││
│  │  │  ┌─────────┐  ┌─────────┐  ┌─────────┐              │  ││
│  │  │  │   App   │  │   App   │  │   App   │              │  ││
│  │  │  │  Bins/  │  │  Bins/  │  │  Bins/  │              │  ││
│  │  │  │  Libs   │  │  Libs   │  │  Libs   │              │  ││
│  │  │  └─────────┘  └─────────┘  └─────────┘              │  ││
│  │  └──────────────────────────────────────────────────────┘  ││
│  │  ┌──────────────────────────────────────────────────────┐  ││
│  │  │                    Images                             │  ││
│  │  │  ┌─────────┐  ┌─────────┐  ┌─────────┐              │  ││
│  │  │  │ OpenJDK │  │ MySQL   │  │  Redis  │              │  ││
│  │  │  └─────────┘  └─────────┘  └─────────┘              │  ││
│  │  └──────────────────────────────────────────────────────┘  ││
│  └─────────────────────────────────────────────────────────────┘│
│                                                                  │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │                     Host OS Kernel                         │  │
│  └───────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘

Key Concepts:
┌──────────────┬────────────────────────────────────────────────┐
│ Image        │ Read-only template with instructions           │
├──────────────┼────────────────────────────────────────────────┤
│ Container    │ Runnable instance of an image                  │
├──────────────┼────────────────────────────────────────────────┤
│ Dockerfile   │ Text file with image build instructions        │
├──────────────┼────────────────────────────────────────────────┤
│ Registry     │ Storage for Docker images (Docker Hub, ECR)    │
├──────────────┼────────────────────────────────────────────────┤
│ Volume       │ Persistent data storage                        │
├──────────────┼────────────────────────────────────────────────┤
│ Network      │ Communication between containers               │
└──────────────┴────────────────────────────────────────────────┘
```

### 1.2 Image Layers

```
Docker Image Layers
═══════════════════

┌─────────────────────────────────────┐
│ Layer 5: Application JAR (R/W)      │ ← Container Layer
├─────────────────────────────────────┤
│ Layer 4: Application Dependencies   │ ← Image Layers
├─────────────────────────────────────┤   (Read-Only)
│ Layer 3: JDK Runtime                │
├─────────────────────────────────────┤
│ Layer 2: OS Packages                │
├─────────────────────────────────────┤
│ Layer 1: Base OS (Alpine/Debian)    │
└─────────────────────────────────────┘

Layer Caching:
- Each instruction creates a layer
- Layers are cached and reused
- Order instructions from least to most frequently changing
- Minimize layer changes for faster builds
```

## 2. Dockerfile for Java Applications

### 2.1 Basic Dockerfile

```dockerfile
# Basic Dockerfile for Spring Boot
FROM eclipse-temurin:21-jre-alpine

# Labels for metadata
LABEL maintainer="dev@company.com"
LABEL version="1.0"
LABEL description="Order Service"

# Create non-root user for security
RUN addgroup -g 1001 appgroup && \
    adduser -u 1001 -G appgroup -s /bin/sh -D appuser

# Set working directory
WORKDIR /app

# Copy application
COPY target/order-service-*.jar app.jar

# Change ownership
RUN chown -R appuser:appgroup /app

# Switch to non-root user
USER appuser

# Expose port
EXPOSE 8080

# Health check
HEALTHCHECK --interval=30s --timeout=3s --start-period=60s --retries=3 \
    CMD wget --no-verbose --tries=1 --spider http://localhost:8080/actuator/health || exit 1

# Entry point with JVM options
ENTRYPOINT ["java", \
    "-XX:+UseContainerSupport", \
    "-XX:MaxRAMPercentage=75.0", \
    "-Djava.security.egd=file:/dev/./urandom", \
    "-jar", "app.jar"]
```

### 2.2 Multi-Stage Build (Optimized)

```dockerfile
# ===========================================
# Stage 1: Build
# ===========================================
FROM eclipse-temurin:21-jdk-alpine AS builder

WORKDIR /build

# Install Maven
RUN apk add --no-cache maven

# Copy POM first (dependency caching)
COPY pom.xml .
COPY .mvn .mvn
COPY mvnw .

# Download dependencies (cached unless pom.xml changes)
RUN mvn dependency:go-offline -B

# Copy source and build
COPY src ./src
RUN mvn package -DskipTests -B

# Extract layers for better caching
RUN java -Djarmode=layertools -jar target/*.jar extract --destination extracted

# ===========================================
# Stage 2: Runtime
# ===========================================
FROM eclipse-temurin:21-jre-alpine AS runtime

# Security: Create non-root user
RUN addgroup -g 1001 spring && \
    adduser -u 1001 -G spring -s /bin/sh -D spring

WORKDIR /app

# Copy extracted layers (optimizes Docker layer caching)
COPY --from=builder --chown=spring:spring /build/extracted/dependencies/ ./
COPY --from=builder --chown=spring:spring /build/extracted/spring-boot-loader/ ./
COPY --from=builder --chown=spring:spring /build/extracted/snapshot-dependencies/ ./
COPY --from=builder --chown=spring:spring /build/extracted/application/ ./

USER spring

EXPOSE 8080

# Use the Spring Boot layer launcher
ENTRYPOINT ["java", \
    "-XX:+UseContainerSupport", \
    "-XX:MaxRAMPercentage=75.0", \
    "-XX:InitialRAMPercentage=50.0", \
    "-XX:+ExitOnOutOfMemoryError", \
    "org.springframework.boot.loader.launch.JarLauncher"]
```

### 2.3 Spring Boot Layer Configuration

```xml
<!-- pom.xml - Enable layer tools -->
<build>
    <plugins>
        <plugin>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-maven-plugin</artifactId>
            <configuration>
                <layers>
                    <enabled>true</enabled>
                </layers>
            </configuration>
        </plugin>
    </plugins>
</build>
```

```yaml
# src/main/resources/layers.idx - Custom layers (optional)
- "dependencies":
  - "BOOT-INF/lib/"
- "spring-boot-loader":
  - "org/"
- "snapshot-dependencies":
  - "BOOT-INF/lib/*SNAPSHOT*.jar"
- "company-dependencies":
  - "BOOT-INF/lib/company-*.jar"
- "application":
  - "BOOT-INF/classes/"
  - "BOOT-INF/classpath.idx"
  - "BOOT-INF/layers.idx"
  - "META-INF/"
```

### 2.4 GraalVM Native Image

```dockerfile
# Native image with GraalVM
FROM ghcr.io/graalvm/graalvm-ce:ol9-java21 AS builder

WORKDIR /build

# Install native-image
RUN gu install native-image

# Copy Maven wrapper and pom
COPY mvnw .
COPY .mvn .mvn
COPY pom.xml .

# Download dependencies
RUN ./mvnw dependency:go-offline

# Copy source
COPY src ./src

# Build native image
RUN ./mvnw -Pnative native:compile -DskipTests

# ===========================================
# Runtime - Minimal base image
# ===========================================
FROM gcr.io/distroless/base-debian12

WORKDIR /app

COPY --from=builder /build/target/order-service /app/order-service

EXPOSE 8080

ENTRYPOINT ["/app/order-service"]
```

## 3. Docker Compose for Development

### 3.1 Complete Development Stack

```yaml
# docker-compose.yml
version: '3.8'

services:
  # ===========================================
  # Application Services
  # ===========================================
  order-service:
    build:
      context: ./order-service
      dockerfile: Dockerfile
    container_name: order-service
    ports:
      - "8081:8080"
    environment:
      - SPRING_PROFILES_ACTIVE=docker
      - SPRING_DATASOURCE_URL=jdbc:mysql://mysql:3306/orders?useSSL=false&allowPublicKeyRetrieval=true
      - SPRING_DATASOURCE_USERNAME=root
      - SPRING_DATASOURCE_PASSWORD=secret
      - SPRING_RABBITMQ_HOST=rabbitmq
      - SPRING_REDIS_HOST=redis
      - EUREKA_CLIENT_SERVICE_URL_DEFAULTZONE=http://eureka:8761/eureka/
    depends_on:
      mysql:
        condition: service_healthy
      rabbitmq:
        condition: service_healthy
      redis:
        condition: service_healthy
    networks:
      - app-network
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8080/actuator/health"]
      interval: 30s
      timeout: 10s
      retries: 5
      start_period: 60s

  payment-service:
    build:
      context: ./payment-service
      dockerfile: Dockerfile
    container_name: payment-service
    ports:
      - "8082:8080"
    environment:
      - SPRING_PROFILES_ACTIVE=docker
      - SPRING_DATASOURCE_URL=jdbc:mysql://mysql:3306/payments?useSSL=false&allowPublicKeyRetrieval=true
      - SPRING_DATASOURCE_USERNAME=root
      - SPRING_DATASOURCE_PASSWORD=secret
      - SPRING_RABBITMQ_HOST=rabbitmq
    depends_on:
      mysql:
        condition: service_healthy
      rabbitmq:
        condition: service_healthy
    networks:
      - app-network

  # ===========================================
  # Service Discovery
  # ===========================================
  eureka:
    image: order-service-eureka:latest
    build:
      context: ./eureka-server
      dockerfile: Dockerfile
    container_name: eureka
    ports:
      - "8761:8761"
    networks:
      - app-network
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8761/actuator/health"]
      interval: 30s
      timeout: 10s
      retries: 3

  # ===========================================
  # API Gateway
  # ===========================================
  gateway:
    build:
      context: ./api-gateway
      dockerfile: Dockerfile
    container_name: gateway
    ports:
      - "8080:8080"
    environment:
      - SPRING_PROFILES_ACTIVE=docker
      - EUREKA_CLIENT_SERVICE_URL_DEFAULTZONE=http://eureka:8761/eureka/
    depends_on:
      eureka:
        condition: service_healthy
    networks:
      - app-network

  # ===========================================
  # Infrastructure Services
  # ===========================================
  mysql:
    image: mysql:8.0
    container_name: mysql
    ports:
      - "3306:3306"
    environment:
      - MYSQL_ROOT_PASSWORD=secret
      - MYSQL_DATABASE=orders
    volumes:
      - mysql-data:/var/lib/mysql
      - ./init-db:/docker-entrypoint-initdb.d
    networks:
      - app-network
    healthcheck:
      test: ["CMD", "mysqladmin", "ping", "-h", "localhost", "-uroot", "-psecret"]
      interval: 10s
      timeout: 5s
      retries: 10

  redis:
    image: redis:7-alpine
    container_name: redis
    ports:
      - "6379:6379"
    volumes:
      - redis-data:/data
    networks:
      - app-network
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 5s
      retries: 5

  rabbitmq:
    image: rabbitmq:3.12-management-alpine
    container_name: rabbitmq
    ports:
      - "5672:5672"
      - "15672:15672"
    environment:
      - RABBITMQ_DEFAULT_USER=guest
      - RABBITMQ_DEFAULT_PASS=guest
    volumes:
      - rabbitmq-data:/var/lib/rabbitmq
    networks:
      - app-network
    healthcheck:
      test: ["CMD", "rabbitmq-diagnostics", "check_port_connectivity"]
      interval: 30s
      timeout: 10s
      retries: 5

  # ===========================================
  # Monitoring Stack
  # ===========================================
  prometheus:
    image: prom/prometheus:latest
    container_name: prometheus
    ports:
      - "9090:9090"
    volumes:
      - ./prometheus/prometheus.yml:/etc/prometheus/prometheus.yml
      - prometheus-data:/prometheus
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.path=/prometheus'
    networks:
      - app-network

  grafana:
    image: grafana/grafana:latest
    container_name: grafana
    ports:
      - "3000:3000"
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=admin
    volumes:
      - grafana-data:/var/lib/grafana
      - ./grafana/dashboards:/etc/grafana/provisioning/dashboards
      - ./grafana/datasources:/etc/grafana/provisioning/datasources
    depends_on:
      - prometheus
    networks:
      - app-network

  # ===========================================
  # Logging Stack (ELK)
  # ===========================================
  elasticsearch:
    image: docker.elastic.co/elasticsearch/elasticsearch:8.11.0
    container_name: elasticsearch
    environment:
      - discovery.type=single-node
      - xpack.security.enabled=false
      - ES_JAVA_OPTS=-Xms512m -Xmx512m
    ports:
      - "9200:9200"
    volumes:
      - elasticsearch-data:/usr/share/elasticsearch/data
    networks:
      - app-network

  kibana:
    image: docker.elastic.co/kibana/kibana:8.11.0
    container_name: kibana
    ports:
      - "5601:5601"
    environment:
      - ELASTICSEARCH_HOSTS=http://elasticsearch:9200
    depends_on:
      - elasticsearch
    networks:
      - app-network

volumes:
  mysql-data:
  redis-data:
  rabbitmq-data:
  prometheus-data:
  grafana-data:
  elasticsearch-data:

networks:
  app-network:
    driver: bridge
```

### 3.2 Environment-Specific Overrides

```yaml
# docker-compose.override.yml (development)
version: '3.8'

services:
  order-service:
    build:
      target: builder  # Use builder stage for dev
    volumes:
      - ./order-service/src:/app/src
      - ./order-service/target:/app/target
    environment:
      - SPRING_DEVTOOLS_RESTART_ENABLED=true
      - SPRING_DEVTOOLS_LIVERELOAD_ENABLED=true
      - LOGGING_LEVEL_ROOT=DEBUG
    ports:
      - "8081:8080"
      - "5005:5005"  # Debug port
    command: >
      java -agentlib:jdwp=transport=dt_socket,server=y,suspend=n,address=*:5005
      -jar /app/target/*.jar
```

```yaml
# docker-compose.prod.yml (production)
version: '3.8'

services:
  order-service:
    deploy:
      replicas: 3
      resources:
        limits:
          cpus: '2'
          memory: 2G
        reservations:
          cpus: '1'
          memory: 1G
      restart_policy:
        condition: on-failure
        delay: 5s
        max_attempts: 3
    environment:
      - SPRING_PROFILES_ACTIVE=prod
      - JAVA_OPTS=-XX:+UseG1GC -XX:MaxGCPauseMillis=200
    logging:
      driver: json-file
      options:
        max-size: "10m"
        max-file: "3"
```

### 3.3 Database Initialization Script

```sql
-- init-db/01-init.sql
CREATE DATABASE IF NOT EXISTS orders;
CREATE DATABASE IF NOT EXISTS payments;

USE orders;

CREATE TABLE IF NOT EXISTS orders (
    id VARCHAR(36) PRIMARY KEY,
    customer_id VARCHAR(36) NOT NULL,
    status VARCHAR(20) NOT NULL,
    total_amount DECIMAL(10,2) NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
);

CREATE TABLE IF NOT EXISTS order_items (
    id VARCHAR(36) PRIMARY KEY,
    order_id VARCHAR(36) NOT NULL,
    product_id VARCHAR(36) NOT NULL,
    quantity INT NOT NULL,
    price DECIMAL(10,2) NOT NULL,
    FOREIGN KEY (order_id) REFERENCES orders(id)
);

-- Create indexes
CREATE INDEX idx_orders_customer ON orders(customer_id);
CREATE INDEX idx_orders_status ON orders(status);
CREATE INDEX idx_order_items_order ON order_items(order_id);
```

## 4. Docker Best Practices

### 4.1 Security Best Practices

```dockerfile
# Secure Dockerfile example
FROM eclipse-temurin:21-jre-alpine AS runtime

# 1. Use specific version tags, not 'latest'
# BAD:  FROM eclipse-temurin:latest
# GOOD: FROM eclipse-temurin:21-jre-alpine

# 2. Run as non-root user
RUN addgroup -g 1001 appgroup && \
    adduser -u 1001 -G appgroup -H -D appuser

# 3. Remove unnecessary packages and clean up
RUN apk add --no-cache curl && \
    rm -rf /var/cache/apk/*

# 4. Use COPY instead of ADD (ADD can unpack archives and fetch URLs)
COPY --chown=appuser:appgroup target/app.jar /app/app.jar

# 5. Don't store secrets in images
# Use environment variables or secret management
# BAD:  ENV DB_PASSWORD=secret123
# GOOD: Use Docker secrets or external config

# 6. Set file permissions explicitly
RUN chmod 400 /app/app.jar

# 7. Use read-only root filesystem where possible
# docker run --read-only ...

# 8. Drop all capabilities and add only what's needed
# docker run --cap-drop ALL ...

# 9. Set security options
USER appuser
WORKDIR /app

# 10. Health check for orchestration
HEALTHCHECK --interval=30s --timeout=3s \
    CMD curl -f http://localhost:8080/actuator/health || exit 1

ENTRYPOINT ["java", "-jar", "app.jar"]
```

### 4.2 Image Optimization

```dockerfile
# Optimized Dockerfile for minimal size
# ===========================================
# Stage 1: Build with full JDK
# ===========================================
FROM eclipse-temurin:21-jdk-alpine AS builder

WORKDIR /build

# Copy only what's needed for dependency resolution
COPY pom.xml .
COPY .mvn .mvn
COPY mvnw .

# Download dependencies (cached layer)
RUN ./mvnw dependency:go-offline -B

# Copy source
COPY src ./src

# Build without tests
RUN ./mvnw package -DskipTests -B && \
    # Extract layered jar
    java -Djarmode=layertools -jar target/*.jar extract

# ===========================================
# Stage 2: Create minimal JRE with jlink
# ===========================================
FROM eclipse-temurin:21-jdk-alpine AS jre-builder

# Create custom JRE
RUN $JAVA_HOME/bin/jlink \
    --add-modules java.base,java.logging,java.xml,java.naming,java.desktop,java.management,java.security.jgss,java.instrument,jdk.unsupported \
    --strip-debug \
    --no-man-pages \
    --no-header-files \
    --compress=2 \
    --output /customjre

# ===========================================
# Stage 3: Final minimal image
# ===========================================
FROM alpine:3.19

# Install minimal dependencies
RUN apk add --no-cache ca-certificates

# Copy custom JRE
ENV JAVA_HOME=/opt/java
COPY --from=jre-builder /customjre $JAVA_HOME
ENV PATH="${JAVA_HOME}/bin:${PATH}"

# Create user
RUN addgroup -g 1001 spring && \
    adduser -u 1001 -G spring -H -D spring

WORKDIR /app

# Copy application layers
COPY --from=builder --chown=spring:spring /build/dependencies/ ./
COPY --from=builder --chown=spring:spring /build/spring-boot-loader/ ./
COPY --from=builder --chown=spring:spring /build/snapshot-dependencies/ ./
COPY --from=builder --chown=spring:spring /build/application/ ./

USER spring

EXPOSE 8080

ENTRYPOINT ["java", \
    "-XX:+UseContainerSupport", \
    "-XX:MaxRAMPercentage=75.0", \
    "org.springframework.boot.loader.launch.JarLauncher"]
```

### 4.3 Build Performance

```dockerfile
# Optimized for build cache
FROM eclipse-temurin:21-jdk-alpine AS builder

WORKDIR /build

# 1. Copy dependency files first (changes rarely)
COPY pom.xml .
COPY .mvn .mvn
COPY mvnw .

# 2. Download dependencies (cached unless pom.xml changes)
RUN --mount=type=cache,target=/root/.m2 \
    ./mvnw dependency:go-offline -B

# 3. Copy source (changes frequently)
COPY src ./src

# 4. Build with cache mount
RUN --mount=type=cache,target=/root/.m2 \
    ./mvnw package -DskipTests -B

# Runtime stage...
```

```bash
# .dockerignore - Exclude unnecessary files
.git
.gitignore
.idea
*.iml
.vscode
target/
!target/*.jar
*.md
docker-compose*.yml
Dockerfile*
.dockerignore
.env*
logs/
tmp/
```

## 5. Docker Commands Reference

### 5.1 Essential Commands

```bash
# Build Commands
docker build -t order-service:1.0 .
docker build -t order-service:1.0 -f Dockerfile.prod .
docker build --no-cache -t order-service:1.0 .
docker build --target builder -t order-service:builder .

# Run Commands
docker run -d --name order-service -p 8080:8080 order-service:1.0
docker run -d --name order-service \
    -e SPRING_PROFILES_ACTIVE=prod \
    -e DB_HOST=mysql \
    -v /data/logs:/app/logs \
    -p 8080:8080 \
    --network app-network \
    order-service:1.0

# Container Management
docker ps                        # List running containers
docker ps -a                     # List all containers
docker logs -f order-service     # Follow logs
docker exec -it order-service sh # Interactive shell
docker stop order-service        # Stop container
docker rm order-service          # Remove container
docker stats                     # Container resource usage

# Image Management
docker images                    # List images
docker rmi order-service:1.0     # Remove image
docker image prune -a            # Remove unused images
docker tag order-service:1.0 registry.example.com/order-service:1.0

# Docker Compose
docker-compose up -d             # Start all services
docker-compose up -d --build     # Build and start
docker-compose down              # Stop and remove
docker-compose logs -f           # Follow all logs
docker-compose ps                # List services
docker-compose exec order-service sh  # Shell into service

# Debugging
docker inspect order-service     # Container details
docker history order-service:1.0 # Image layer history
docker diff order-service        # File changes in container
```

### 5.2 Useful Scripts

```bash
#!/bin/bash
# build-and-push.sh - Build and push to registry

set -e

SERVICE_NAME=$1
VERSION=${2:-latest}
REGISTRY=${REGISTRY:-registry.example.com}

if [ -z "$SERVICE_NAME" ]; then
    echo "Usage: $0 <service-name> [version]"
    exit 1
fi

echo "Building $SERVICE_NAME:$VERSION..."
docker build -t $SERVICE_NAME:$VERSION .

echo "Tagging for registry..."
docker tag $SERVICE_NAME:$VERSION $REGISTRY/$SERVICE_NAME:$VERSION

echo "Pushing to registry..."
docker push $REGISTRY/$SERVICE_NAME:$VERSION

echo "Done! Pushed $REGISTRY/$SERVICE_NAME:$VERSION"
```

```bash
#!/bin/bash
# clean-docker.sh - Clean up Docker resources

echo "Stopping all running containers..."
docker stop $(docker ps -q) 2>/dev/null || true

echo "Removing all containers..."
docker rm $(docker ps -aq) 2>/dev/null || true

echo "Removing unused images..."
docker image prune -af

echo "Removing unused volumes..."
docker volume prune -f

echo "Removing unused networks..."
docker network prune -f

echo "Docker cleanup complete!"
docker system df
```

## 6. Monitoring & Logging

### 6.1 Application Logging Configuration

```yaml
# application.yml - Docker-optimized logging
logging:
  pattern:
    console: "%d{yyyy-MM-dd HH:mm:ss.SSS} [%thread] %-5level %logger{36} - %msg%n"
  level:
    root: INFO
    com.example: DEBUG

# JSON logging for log aggregation
---
spring:
  config:
    activate:
      on-profile: docker

logging:
  pattern:
    console: >
      {"timestamp":"%d{yyyy-MM-dd'T'HH:mm:ss.SSSZ}",
       "level":"%level",
       "thread":"%thread",
       "logger":"%logger{36}",
       "message":"%msg",
       "exception":"%ex{full}"}%n
```

### 6.2 Prometheus Configuration

```yaml
# prometheus/prometheus.yml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

scrape_configs:
  - job_name: 'spring-boot-apps'
    metrics_path: '/actuator/prometheus'
    static_configs:
      - targets:
          - 'order-service:8080'
          - 'payment-service:8080'
    relabel_configs:
      - source_labels: [__address__]
        regex: '([^:]+):\d+'
        target_label: instance

  - job_name: 'docker-containers'
    static_configs:
      - targets: ['cadvisor:8080']
```

## 7. Hands-on Exercises

### Exercise 1: Optimize Dockerfile
Create an optimized Dockerfile for a Spring Boot app.

```bash
# Requirements:
# 1. Multi-stage build
# 2. Layer extraction for caching
# 3. Non-root user
# 4. Health check
# 5. Final image < 200MB
```

### Exercise 2: Docker Compose Stack
Build a complete microservices stack.

```bash
# Requirements:
# 1. At least 2 application services
# 2. MySQL with initialization
# 3. Redis for caching
# 4. RabbitMQ for messaging
# 5. Health checks and dependencies
```

### Exercise 3: Security Hardening
Secure a Docker deployment.

```bash
# Requirements:
# 1. Run as non-root
# 2. Read-only filesystem
# 3. Minimal capabilities
# 4. Secret management
# 5. Network isolation
```

### Exercise 4: Build Pipeline
Create a CI/CD pipeline for Docker builds.

```bash
# Requirements:
# 1. Build and test in container
# 2. Push to registry
# 3. Version tagging
# 4. Security scanning
# 5. Deployment automation
```

## 8. Key Takeaways

### Image Best Practices
1. **Multi-stage builds**: Separate build and runtime
2. **Layer ordering**: Least changing first for cache efficiency
3. **Minimal base images**: Alpine or distroless
4. **Specific tags**: Avoid `latest` in production

### Security Essentials
1. **Non-root user**: Always run as unprivileged user
2. **Read-only filesystem**: Prevent runtime modifications
3. **Minimal packages**: Reduce attack surface
4. **Secret management**: Never bake secrets into images

### Development Workflow
1. **Docker Compose**: Consistent development environment
2. **Volume mounts**: Enable hot reload
3. **Health checks**: Ensure service readiness
4. **Logging**: JSON format for aggregation

### Performance Tips
1. **Cache mounts**: Speed up builds with dependency caching
2. **.dockerignore**: Exclude unnecessary files
3. **Layer tools**: Use Spring Boot layer extraction
4. **Resource limits**: Set memory and CPU constraints

---

## Navigation

[← Day 20: Event-Driven Architecture](./day-20-event-driven-architecture.md) | [Day 22: Kubernetes Basics →](./day-22-kubernetes-basics.md)

[Back to Overview](./00-overview.md)
