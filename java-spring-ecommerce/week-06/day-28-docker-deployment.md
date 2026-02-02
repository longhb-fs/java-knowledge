# Day 28: Docker & Deployment

## Mục tiêu học tập
- Hiểu Docker containerization cho Spring Boot
- Tạo Dockerfile tối ưu với multi-stage build
- Sử dụng Docker Compose cho development
- Deploy ứng dụng lên cloud platforms
- Implement CI/CD pipeline cơ bản

## 1. Docker Fundamentals

### 1.1. Docker Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                     DOCKER ARCHITECTURE                      │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  ┌─────────────────────────────────────────────────────┐    │
│  │                   Docker Client                      │    │
│  │    docker build, run, pull, push, compose           │    │
│  └─────────────────────┬───────────────────────────────┘    │
│                        │ REST API                            │
│  ┌─────────────────────▼───────────────────────────────┐    │
│  │                   Docker Daemon                      │    │
│  │              (dockerd - background service)          │    │
│  └─────────────────────┬───────────────────────────────┘    │
│                        │                                     │
│  ┌─────────────────────▼───────────────────────────────┐    │
│  │                   Docker Objects                     │    │
│  │  ┌───────────┐  ┌───────────┐  ┌───────────────┐    │    │
│  │  │  Images   │  │ Containers│  │   Networks    │    │    │
│  │  └───────────┘  └───────────┘  └───────────────┘    │    │
│  │  ┌───────────┐  ┌───────────────────────────────┐   │    │
│  │  │  Volumes  │  │         Registries            │   │    │
│  │  └───────────┘  │   (Docker Hub, ECR, GCR)      │   │    │
│  │                 └───────────────────────────────┘   │    │
│  └─────────────────────────────────────────────────────┘    │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### 1.2. Container vs Virtual Machine

```
┌─────────────────────────────────────────────────────────────┐
│              CONTAINER vs VIRTUAL MACHINE                    │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│    Containers                    Virtual Machines            │
│  ┌─────────────────┐          ┌─────────────────┐           │
│  │   App A │ App B │          │      App A      │           │
│  ├─────────┴───────┤          ├─────────────────┤           │
│  │   Bins/Libs     │          │   Bins/Libs     │           │
│  ├─────────────────┤          ├─────────────────┤           │
│  │ Container Engine│          │   Guest OS      │           │
│  ├─────────────────┤          ├─────────────────┤           │
│  │    Host OS      │          │   Hypervisor    │           │
│  ├─────────────────┤          ├─────────────────┤           │
│  │  Infrastructure │          │    Host OS      │           │
│  └─────────────────┘          ├─────────────────┤           │
│                               │  Infrastructure │           │
│  • Lightweight (MB)           └─────────────────┘           │
│  • Fast startup (seconds)                                    │
│  • Share host kernel          • Heavyweight (GB)            │
│  • Process isolation          • Slow startup (minutes)      │
│                               • Complete isolation          │
│                               • Full OS per VM              │
└─────────────────────────────────────────────────────────────┘
```

## 2. Dockerfile cho Spring Boot

### 2.1. Simple Dockerfile

```dockerfile
# Dockerfile.simple
FROM eclipse-temurin:21-jre-alpine

LABEL maintainer="your-email@example.com"
LABEL version="1.0"
LABEL description="E-Commerce Spring Boot Application"

# Create non-root user for security
RUN addgroup -S spring && adduser -S spring -G spring

# Set working directory
WORKDIR /app

# Copy jar file
COPY target/ecommerce-*.jar app.jar

# Change ownership
RUN chown -R spring:spring /app

# Switch to non-root user
USER spring:spring

# Expose port
EXPOSE 8080

# Health check
HEALTHCHECK --interval=30s --timeout=3s --start-period=60s --retries=3 \
    CMD wget --no-verbose --tries=1 --spider http://localhost:8080/actuator/health || exit 1

# JVM tuning for containers
ENV JAVA_OPTS="-XX:+UseContainerSupport -XX:MaxRAMPercentage=75.0 -XX:InitialRAMPercentage=50.0"

# Run the application
ENTRYPOINT ["sh", "-c", "java $JAVA_OPTS -jar app.jar"]
```

### 2.2. Multi-Stage Dockerfile (Recommended)

```dockerfile
# Dockerfile
# ============================================
# Stage 1: Build Stage
# ============================================
FROM eclipse-temurin:21-jdk-alpine AS builder

WORKDIR /build

# Install Maven
RUN apk add --no-cache maven

# Copy pom.xml first for dependency caching
COPY pom.xml .
COPY .mvn .mvn
COPY mvnw .

# Download dependencies (cached layer)
RUN mvn dependency:go-offline -B

# Copy source code
COPY src src

# Build application (skip tests for faster build)
RUN mvn clean package -DskipTests -B

# Extract layers for better caching
RUN java -Djarmode=layertools -jar target/*.jar extract --destination extracted

# ============================================
# Stage 2: Runtime Stage
# ============================================
FROM eclipse-temurin:21-jre-alpine AS runtime

LABEL maintainer="dev@ecommerce.com"
LABEL version="1.0.0"
LABEL description="E-Commerce API Service"

# Security: Create non-root user
RUN addgroup -g 1001 spring && \
    adduser -u 1001 -G spring -s /bin/sh -D spring

WORKDIR /app

# Copy layers from builder (ordered by change frequency)
COPY --from=builder --chown=spring:spring /build/extracted/dependencies/ ./
COPY --from=builder --chown=spring:spring /build/extracted/spring-boot-loader/ ./
COPY --from=builder --chown=spring:spring /build/extracted/snapshot-dependencies/ ./
COPY --from=builder --chown=spring:spring /build/extracted/application/ ./

# Switch to non-root user
USER spring:spring

# Expose port
EXPOSE 8080

# Health check
HEALTHCHECK --interval=30s --timeout=10s --start-period=60s --retries=3 \
    CMD wget --no-verbose --tries=1 --spider http://localhost:8080/actuator/health || exit 1

# Container-optimized JVM settings
ENV JAVA_OPTS="\
    -XX:+UseContainerSupport \
    -XX:MaxRAMPercentage=75.0 \
    -XX:InitialRAMPercentage=50.0 \
    -XX:+UseG1GC \
    -XX:+UseStringDeduplication \
    -Djava.security.egd=file:/dev/./urandom"

# Use layered jar launcher
ENTRYPOINT ["sh", "-c", "java $JAVA_OPTS org.springframework.boot.loader.launch.JarLauncher"]
```

### 2.3. Enable Layered JAR trong pom.xml

```xml
<!-- pom.xml -->
<build>
    <plugins>
        <plugin>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-maven-plugin</artifactId>
            <configuration>
                <layers>
                    <enabled>true</enabled>
                </layers>
                <excludes>
                    <exclude>
                        <groupId>org.projectlombok</groupId>
                        <artifactId>lombok</artifactId>
                    </exclude>
                </excludes>
            </configuration>
        </plugin>
    </plugins>
</build>
```

### 2.4. .dockerignore

```dockerignore
# .dockerignore
.git
.gitignore
.idea
*.iml
.vscode
*.md
!README.md

# Build artifacts (we build inside container)
target/
!target/*.jar

# Test files
src/test/

# Local config
*.local
.env.local

# Logs
*.log
logs/

# Docker files (prevent recursive copy)
Dockerfile*
docker-compose*
.docker/
```

## 3. Docker Compose cho Development

### 3.1. docker-compose.yml

```yaml
# docker-compose.yml
version: '3.9'

services:
  # ============================================
  # Application Service
  # ============================================
  app:
    build:
      context: .
      dockerfile: Dockerfile
    container_name: ecommerce-app
    restart: unless-stopped
    ports:
      - "8080:8080"
    environment:
      - SPRING_PROFILES_ACTIVE=docker
      - SPRING_DATASOURCE_URL=jdbc:mysql://mysql:3306/ecommerce?useSSL=false&allowPublicKeyRetrieval=true
      - SPRING_DATASOURCE_USERNAME=ecommerce
      - SPRING_DATASOURCE_PASSWORD=${DB_PASSWORD:-ecommerce123}
      - SPRING_DATA_REDIS_HOST=redis
      - SPRING_DATA_REDIS_PORT=6379
      - SPRING_ELASTICSEARCH_URIS=http://elasticsearch:9200
      - JWT_SECRET=${JWT_SECRET:-your-super-secret-jwt-key-that-is-at-least-256-bits}
      - STRIPE_API_KEY=${STRIPE_API_KEY}
    depends_on:
      mysql:
        condition: service_healthy
      redis:
        condition: service_healthy
    networks:
      - ecommerce-network
    healthcheck:
      test: ["CMD", "wget", "--no-verbose", "--tries=1", "--spider", "http://localhost:8080/actuator/health"]
      interval: 30s
      timeout: 10s
      retries: 5
      start_period: 60s

  # ============================================
  # MySQL Database
  # ============================================
  mysql:
    image: mysql:8.0
    container_name: ecommerce-mysql
    restart: unless-stopped
    ports:
      - "3306:3306"
    environment:
      - MYSQL_ROOT_PASSWORD=${MYSQL_ROOT_PASSWORD:-root123}
      - MYSQL_DATABASE=ecommerce
      - MYSQL_USER=ecommerce
      - MYSQL_PASSWORD=${DB_PASSWORD:-ecommerce123}
    volumes:
      - mysql-data:/var/lib/mysql
      - ./docker/mysql/init:/docker-entrypoint-initdb.d
      - ./docker/mysql/conf:/etc/mysql/conf.d
    networks:
      - ecommerce-network
    healthcheck:
      test: ["CMD", "mysqladmin", "ping", "-h", "localhost", "-u", "root", "-p${MYSQL_ROOT_PASSWORD:-root123}"]
      interval: 10s
      timeout: 5s
      retries: 5
      start_period: 30s

  # ============================================
  # Redis Cache
  # ============================================
  redis:
    image: redis:7-alpine
    container_name: ecommerce-redis
    restart: unless-stopped
    ports:
      - "6379:6379"
    command: redis-server --appendonly yes --maxmemory 256mb --maxmemory-policy allkeys-lru
    volumes:
      - redis-data:/data
    networks:
      - ecommerce-network
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 5s
      retries: 5

  # ============================================
  # Elasticsearch
  # ============================================
  elasticsearch:
    image: elasticsearch:8.11.0
    container_name: ecommerce-elasticsearch
    restart: unless-stopped
    ports:
      - "9200:9200"
      - "9300:9300"
    environment:
      - discovery.type=single-node
      - xpack.security.enabled=false
      - ES_JAVA_OPTS=-Xms512m -Xmx512m
    volumes:
      - elasticsearch-data:/usr/share/elasticsearch/data
    networks:
      - ecommerce-network
    healthcheck:
      test: ["CMD-SHELL", "curl -s http://localhost:9200/_cluster/health | grep -vq '\"status\":\"red\"'"]
      interval: 30s
      timeout: 10s
      retries: 5

  # ============================================
  # Nginx Reverse Proxy
  # ============================================
  nginx:
    image: nginx:alpine
    container_name: ecommerce-nginx
    restart: unless-stopped
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./docker/nginx/nginx.conf:/etc/nginx/nginx.conf:ro
      - ./docker/nginx/ssl:/etc/nginx/ssl:ro
    depends_on:
      - app
    networks:
      - ecommerce-network

# ============================================
# Networks
# ============================================
networks:
  ecommerce-network:
    driver: bridge
    ipam:
      config:
        - subnet: 172.28.0.0/16

# ============================================
# Volumes
# ============================================
volumes:
  mysql-data:
    driver: local
  redis-data:
    driver: local
  elasticsearch-data:
    driver: local
```

### 3.2. Docker Compose Override cho Development

```yaml
# docker-compose.override.yml (for local development)
version: '3.9'

services:
  app:
    build:
      context: .
      dockerfile: Dockerfile.dev
    volumes:
      # Hot reload with source code
      - ./target:/app/target:ro
    environment:
      - SPRING_PROFILES_ACTIVE=dev,docker
      - SPRING_DEVTOOLS_RESTART_ENABLED=true
      - DEBUG=true
    ports:
      - "5005:5005"  # Debug port

  # Adminer for database management
  adminer:
    image: adminer:latest
    container_name: ecommerce-adminer
    restart: unless-stopped
    ports:
      - "8081:8080"
    depends_on:
      - mysql
    networks:
      - ecommerce-network

  # Redis Commander for Redis management
  redis-commander:
    image: rediscommander/redis-commander:latest
    container_name: ecommerce-redis-commander
    restart: unless-stopped
    ports:
      - "8082:8081"
    environment:
      - REDIS_HOSTS=local:redis:6379
    depends_on:
      - redis
    networks:
      - ecommerce-network

  # Mailhog for email testing
  mailhog:
    image: mailhog/mailhog:latest
    container_name: ecommerce-mailhog
    restart: unless-stopped
    ports:
      - "1025:1025"  # SMTP
      - "8025:8025"  # Web UI
    networks:
      - ecommerce-network
```

### 3.3. Production Docker Compose

```yaml
# docker-compose.prod.yml
version: '3.9'

services:
  app:
    image: ${DOCKER_REGISTRY}/ecommerce-api:${VERSION:-latest}
    deploy:
      replicas: 3
      update_config:
        parallelism: 1
        delay: 10s
        failure_action: rollback
      restart_policy:
        condition: on-failure
        delay: 5s
        max_attempts: 3
      resources:
        limits:
          cpus: '1'
          memory: 1G
        reservations:
          cpus: '0.5'
          memory: 512M
    environment:
      - SPRING_PROFILES_ACTIVE=prod
      - JAVA_OPTS=-XX:MaxRAMPercentage=75.0 -XX:+UseG1GC
    logging:
      driver: "json-file"
      options:
        max-size: "10m"
        max-file: "3"
    secrets:
      - db_password
      - jwt_secret
      - stripe_key

  mysql:
    deploy:
      resources:
        limits:
          cpus: '2'
          memory: 2G
    volumes:
      - mysql-prod-data:/var/lib/mysql

  redis:
    deploy:
      resources:
        limits:
          cpus: '0.5'
          memory: 512M

secrets:
  db_password:
    external: true
  jwt_secret:
    external: true
  stripe_key:
    external: true

volumes:
  mysql-prod-data:
    driver: local
```

## 4. Nginx Configuration

### 4.1. nginx.conf

```nginx
# docker/nginx/nginx.conf
user nginx;
worker_processes auto;
error_log /var/log/nginx/error.log warn;
pid /var/run/nginx.pid;

events {
    worker_connections 1024;
    use epoll;
    multi_accept on;
}

http {
    include /etc/nginx/mime.types;
    default_type application/octet-stream;

    # Logging
    log_format main '$remote_addr - $remote_user [$time_local] "$request" '
                    '$status $body_bytes_sent "$http_referer" '
                    '"$http_user_agent" "$http_x_forwarded_for" '
                    'rt=$request_time uct="$upstream_connect_time" '
                    'uht="$upstream_header_time" urt="$upstream_response_time"';

    access_log /var/log/nginx/access.log main;

    # Performance
    sendfile on;
    tcp_nopush on;
    tcp_nodelay on;
    keepalive_timeout 65;
    types_hash_max_size 2048;

    # Gzip compression
    gzip on;
    gzip_vary on;
    gzip_proxied any;
    gzip_comp_level 6;
    gzip_types text/plain text/css text/xml application/json application/javascript
               application/xml application/xml+rss text/javascript application/x-javascript;

    # Rate limiting
    limit_req_zone $binary_remote_addr zone=api:10m rate=10r/s;
    limit_conn_zone $binary_remote_addr zone=conn:10m;

    # Upstream for load balancing
    upstream ecommerce_api {
        least_conn;
        server app:8080 weight=1 max_fails=3 fail_timeout=30s;
        keepalive 32;
    }

    # HTTP to HTTPS redirect
    server {
        listen 80;
        server_name _;
        return 301 https://$host$request_uri;
    }

    # Main HTTPS server
    server {
        listen 443 ssl http2;
        server_name api.ecommerce.com;

        # SSL configuration
        ssl_certificate /etc/nginx/ssl/cert.pem;
        ssl_certificate_key /etc/nginx/ssl/key.pem;
        ssl_session_timeout 1d;
        ssl_session_cache shared:SSL:50m;
        ssl_session_tickets off;

        # Modern SSL configuration
        ssl_protocols TLSv1.2 TLSv1.3;
        ssl_ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256;
        ssl_prefer_server_ciphers off;

        # HSTS
        add_header Strict-Transport-Security "max-age=63072000" always;

        # Security headers
        add_header X-Frame-Options "SAMEORIGIN" always;
        add_header X-Content-Type-Options "nosniff" always;
        add_header X-XSS-Protection "1; mode=block" always;
        add_header Referrer-Policy "strict-origin-when-cross-origin" always;

        # API endpoints
        location /api/ {
            limit_req zone=api burst=20 nodelay;
            limit_conn conn 10;

            proxy_pass http://ecommerce_api;
            proxy_http_version 1.1;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
            proxy_set_header Connection "";

            # Timeouts
            proxy_connect_timeout 60s;
            proxy_send_timeout 60s;
            proxy_read_timeout 60s;

            # Buffer settings
            proxy_buffering on;
            proxy_buffer_size 128k;
            proxy_buffers 4 256k;
            proxy_busy_buffers_size 256k;
        }

        # Health check endpoint (no rate limiting)
        location /actuator/health {
            proxy_pass http://ecommerce_api;
            proxy_http_version 1.1;
            proxy_set_header Host $host;
        }

        # Swagger UI
        location /swagger-ui/ {
            proxy_pass http://ecommerce_api;
            proxy_http_version 1.1;
            proxy_set_header Host $host;
        }

        # Deny access to sensitive paths
        location ~ ^/(actuator|env|metrics)/ {
            deny all;
            return 404;
        }
    }
}
```

## 5. Application Profiles

### 5.1. application-docker.yml

```yaml
# src/main/resources/application-docker.yml
spring:
  datasource:
    url: ${SPRING_DATASOURCE_URL:jdbc:mysql://mysql:3306/ecommerce}
    username: ${SPRING_DATASOURCE_USERNAME:ecommerce}
    password: ${SPRING_DATASOURCE_PASSWORD:ecommerce123}
    hikari:
      maximum-pool-size: 20
      minimum-idle: 5
      idle-timeout: 300000
      connection-timeout: 20000

  data:
    redis:
      host: ${SPRING_DATA_REDIS_HOST:redis}
      port: ${SPRING_DATA_REDIS_PORT:6379}

  elasticsearch:
    uris: ${SPRING_ELASTICSEARCH_URIS:http://elasticsearch:9200}

  jpa:
    hibernate:
      ddl-auto: validate
    show-sql: false

  flyway:
    enabled: true
    baseline-on-migrate: true

# Actuator endpoints for Docker health checks
management:
  endpoints:
    web:
      exposure:
        include: health,info,prometheus
  endpoint:
    health:
      show-details: when_authorized
      probes:
        enabled: true
  health:
    livenessState:
      enabled: true
    readinessState:
      enabled: true

# Logging for containerized environment
logging:
  pattern:
    console: "%d{yyyy-MM-dd HH:mm:ss} [%thread] %-5level %logger{36} - %msg%n"
  level:
    root: INFO
    com.ecommerce: INFO
    org.springframework: WARN
    org.hibernate: WARN

# Server configuration
server:
  port: 8080
  forward-headers-strategy: framework
  tomcat:
    remote-ip-header: x-forwarded-for
    protocol-header: x-forwarded-proto
```

## 6. CI/CD Pipeline

### 6.1. GitHub Actions Workflow

```yaml
# .github/workflows/ci-cd.yml
name: CI/CD Pipeline

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

jobs:
  # ============================================
  # Build & Test
  # ============================================
  build:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Set up JDK 21
        uses: actions/setup-java@v4
        with:
          java-version: '21'
          distribution: 'temurin'
          cache: 'maven'

      - name: Cache Maven packages
        uses: actions/cache@v4
        with:
          path: ~/.m2
          key: ${{ runner.os }}-m2-${{ hashFiles('**/pom.xml') }}
          restore-keys: ${{ runner.os }}-m2

      - name: Run tests
        run: mvn clean verify -B

      - name: Upload test results
        uses: actions/upload-artifact@v4
        if: always()
        with:
          name: test-results
          path: target/surefire-reports/

      - name: Upload coverage report
        uses: actions/upload-artifact@v4
        with:
          name: coverage-report
          path: target/site/jacoco/

  # ============================================
  # Security Scan
  # ============================================
  security-scan:
    runs-on: ubuntu-latest
    needs: build

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Run Trivy vulnerability scanner
        uses: aquasecurity/trivy-action@master
        with:
          scan-type: 'fs'
          scan-ref: '.'
          format: 'sarif'
          output: 'trivy-results.sarif'

      - name: Upload Trivy scan results
        uses: github/codeql-action/upload-sarif@v3
        if: always()
        with:
          sarif_file: 'trivy-results.sarif'

  # ============================================
  # Build & Push Docker Image
  # ============================================
  docker:
    runs-on: ubuntu-latest
    needs: [build, security-scan]
    if: github.event_name == 'push' && github.ref == 'refs/heads/main'

    permissions:
      contents: read
      packages: write

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Set up QEMU
        uses: docker/setup-qemu-action@v3

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Log in to Container Registry
        uses: docker/login-action@v3
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Extract metadata
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}
          tags: |
            type=sha,prefix=
            type=ref,event=branch
            type=semver,pattern={{version}}
            type=raw,value=latest,enable=${{ github.ref == 'refs/heads/main' }}

      - name: Build and push Docker image
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          platforms: linux/amd64,linux/arm64
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          cache-from: type=gha
          cache-to: type=gha,mode=max

  # ============================================
  # Deploy to Staging
  # ============================================
  deploy-staging:
    runs-on: ubuntu-latest
    needs: docker
    environment: staging

    steps:
      - name: Deploy to staging
        uses: appleboy/ssh-action@v1.0.0
        with:
          host: ${{ secrets.STAGING_HOST }}
          username: ${{ secrets.STAGING_USER }}
          key: ${{ secrets.STAGING_SSH_KEY }}
          script: |
            cd /opt/ecommerce
            docker compose pull
            docker compose up -d --remove-orphans
            docker system prune -f

  # ============================================
  # Deploy to Production
  # ============================================
  deploy-production:
    runs-on: ubuntu-latest
    needs: deploy-staging
    environment: production
    if: github.ref == 'refs/heads/main'

    steps:
      - name: Deploy to production
        uses: appleboy/ssh-action@v1.0.0
        with:
          host: ${{ secrets.PROD_HOST }}
          username: ${{ secrets.PROD_USER }}
          key: ${{ secrets.PROD_SSH_KEY }}
          script: |
            cd /opt/ecommerce
            docker compose -f docker-compose.yml -f docker-compose.prod.yml pull
            docker compose -f docker-compose.yml -f docker-compose.prod.yml up -d --remove-orphans
            # Wait for health check
            sleep 30
            curl -f http://localhost:8080/actuator/health || exit 1
```

## 7. Kubernetes Deployment (Optional)

### 7.1. Kubernetes Manifests

```yaml
# k8s/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ecommerce-api
  labels:
    app: ecommerce-api
spec:
  replicas: 3
  selector:
    matchLabels:
      app: ecommerce-api
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  template:
    metadata:
      labels:
        app: ecommerce-api
    spec:
      containers:
        - name: ecommerce-api
          image: ghcr.io/your-org/ecommerce-api:latest
          ports:
            - containerPort: 8080
          env:
            - name: SPRING_PROFILES_ACTIVE
              value: "prod,kubernetes"
            - name: SPRING_DATASOURCE_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: ecommerce-secrets
                  key: db-password
            - name: JWT_SECRET
              valueFrom:
                secretKeyRef:
                  name: ecommerce-secrets
                  key: jwt-secret
          resources:
            requests:
              memory: "512Mi"
              cpu: "250m"
            limits:
              memory: "1Gi"
              cpu: "1"
          livenessProbe:
            httpGet:
              path: /actuator/health/liveness
              port: 8080
            initialDelaySeconds: 60
            periodSeconds: 10
            failureThreshold: 3
          readinessProbe:
            httpGet:
              path: /actuator/health/readiness
              port: 8080
            initialDelaySeconds: 30
            periodSeconds: 5
            failureThreshold: 3
      imagePullSecrets:
        - name: ghcr-secret

---
apiVersion: v1
kind: Service
metadata:
  name: ecommerce-api-service
spec:
  selector:
    app: ecommerce-api
  ports:
    - protocol: TCP
      port: 80
      targetPort: 8080
  type: ClusterIP

---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ecommerce-ingress
  annotations:
    kubernetes.io/ingress.class: nginx
    cert-manager.io/cluster-issuer: letsencrypt-prod
spec:
  tls:
    - hosts:
        - api.ecommerce.com
      secretName: ecommerce-tls
  rules:
    - host: api.ecommerce.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: ecommerce-api-service
                port:
                  number: 80

---
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: ecommerce-api-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: ecommerce-api
  minReplicas: 3
  maxReplicas: 10
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
    - type: Resource
      resource:
        name: memory
        target:
          type: Utilization
          averageUtilization: 80
```

## 8. Makefile cho Development

```makefile
# Makefile
.PHONY: help build test run docker-build docker-up docker-down clean

# Variables
APP_NAME := ecommerce-api
VERSION := $(shell mvn help:evaluate -Dexpression=project.version -q -DforceStdout)
DOCKER_REGISTRY := ghcr.io/your-org

help: ## Show this help
	@grep -E '^[a-zA-Z_-]+:.*?## .*$$' $(MAKEFILE_LIST) | sort | awk 'BEGIN {FS = ":.*?## "}; {printf "\033[36m%-30s\033[0m %s\n", $$1, $$2}'

# ============================================
# Development
# ============================================
install: ## Install dependencies
	mvn dependency:resolve

build: ## Build the application
	mvn clean package -DskipTests

build-full: ## Build with tests
	mvn clean verify

test: ## Run tests
	mvn test

test-integration: ## Run integration tests
	mvn verify -P integration-tests

run: ## Run locally
	mvn spring-boot:run

run-dev: ## Run with dev profile
	mvn spring-boot:run -Dspring-boot.run.profiles=dev

# ============================================
# Docker
# ============================================
docker-build: build ## Build Docker image
	docker build -t $(APP_NAME):$(VERSION) -t $(APP_NAME):latest .

docker-build-no-cache: build ## Build Docker image without cache
	docker build --no-cache -t $(APP_NAME):$(VERSION) -t $(APP_NAME):latest .

docker-up: ## Start all services
	docker compose up -d

docker-down: ## Stop all services
	docker compose down

docker-logs: ## View logs
	docker compose logs -f

docker-ps: ## List running containers
	docker compose ps

docker-clean: ## Remove all containers and volumes
	docker compose down -v --remove-orphans
	docker system prune -f

# ============================================
# Database
# ============================================
db-migrate: ## Run database migrations
	mvn flyway:migrate

db-clean: ## Clean database
	mvn flyway:clean

db-info: ## Show migration info
	mvn flyway:info

# ============================================
# Registry
# ============================================
docker-push: docker-build ## Push to registry
	docker tag $(APP_NAME):$(VERSION) $(DOCKER_REGISTRY)/$(APP_NAME):$(VERSION)
	docker tag $(APP_NAME):latest $(DOCKER_REGISTRY)/$(APP_NAME):latest
	docker push $(DOCKER_REGISTRY)/$(APP_NAME):$(VERSION)
	docker push $(DOCKER_REGISTRY)/$(APP_NAME):latest

# ============================================
# Cleanup
# ============================================
clean: ## Clean build artifacts
	mvn clean
	rm -rf target/
	rm -rf .mvn/wrapper/maven-wrapper.jar
```

## 9. Best Practices

### 9.1. Docker Security Checklist

```
┌─────────────────────────────────────────────────────────────┐
│              DOCKER SECURITY CHECKLIST                       │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  ✅ Image Security                                           │
│     □ Use official base images                               │
│     □ Pin specific versions (not :latest)                    │
│     □ Scan images for vulnerabilities                        │
│     □ Use multi-stage builds                                 │
│     □ Minimize image layers                                  │
│                                                              │
│  ✅ Container Security                                       │
│     □ Run as non-root user                                   │
│     □ Use read-only filesystem where possible                │
│     □ Drop unnecessary capabilities                          │
│     □ Set resource limits (CPU, memory)                      │
│     □ Use health checks                                      │
│                                                              │
│  ✅ Secrets Management                                       │
│     □ Never hardcode secrets in Dockerfile                   │
│     □ Use Docker secrets or external vault                   │
│     □ Use environment variables wisely                       │
│     □ Rotate secrets regularly                               │
│                                                              │
│  ✅ Network Security                                         │
│     □ Use custom bridge networks                             │
│     □ Expose only necessary ports                            │
│     □ Use TLS for inter-service communication                │
│     □ Implement network policies                             │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### 9.2. Deployment Checklist

```
┌─────────────────────────────────────────────────────────────┐
│              DEPLOYMENT CHECKLIST                            │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  📋 Pre-Deployment                                           │
│     □ All tests passing                                      │
│     □ Security scan completed                                │
│     □ Performance tests passed                               │
│     □ Documentation updated                                  │
│     □ Changelog updated                                      │
│                                                              │
│  📋 Configuration                                            │
│     □ Environment variables set                              │
│     □ Secrets configured                                     │
│     □ Database migrations ready                              │
│     □ Resource limits defined                                │
│     □ Health checks configured                               │
│                                                              │
│  📋 Deployment                                               │
│     □ Backup database before migration                       │
│     □ Deploy to staging first                                │
│     □ Verify staging health                                  │
│     □ Run smoke tests on staging                             │
│     □ Deploy to production                                   │
│                                                              │
│  📋 Post-Deployment                                          │
│     □ Verify production health                               │
│     □ Monitor error rates                                    │
│     □ Check performance metrics                              │
│     □ Verify critical user flows                             │
│     □ Rollback plan ready                                    │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

## 10. Bài tập thực hành

### Bài tập 1: Dockerfile Optimization
Tạo Dockerfile tối ưu cho ứng dụng với:
- Multi-stage build
- Layer caching
- Security best practices
- Health checks

### Bài tập 2: Docker Compose Setup
Thiết lập môi trường development với:
- Application + MySQL + Redis
- Volume persistence
- Network isolation
- Service dependencies

### Bài tập 3: CI/CD Pipeline
Tạo GitHub Actions workflow với:
- Build và test
- Docker image build
- Security scanning
- Deployment automation

## Tổng kết

Hôm nay đã học:
- ✅ Docker containerization fundamentals
- ✅ Multi-stage Dockerfile optimization
- ✅ Docker Compose cho development và production
- ✅ Nginx reverse proxy configuration
- ✅ CI/CD pipeline với GitHub Actions
- ✅ Kubernetes deployment basics
- ✅ Security và deployment best practices

## Navigation

- [← Day 27: Integration Testing](./day-27-integration-testing.md)
- [Day 29: Monitoring & Logging →](./day-29-monitoring.md)
