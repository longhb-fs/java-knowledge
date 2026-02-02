# Day 29: Monitoring & Logging

## Mục tiêu học tập
- Hiểu tầm quan trọng của observability trong production
- Implement structured logging với Logback và ELK Stack
- Sử dụng Spring Boot Actuator cho health monitoring
- Tích hợp Prometheus và Grafana cho metrics
- Setup distributed tracing với Micrometer

## 1. Observability Overview

### 1.1. Three Pillars of Observability

```
┌─────────────────────────────────────────────────────────────┐
│              THREE PILLARS OF OBSERVABILITY                  │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────┐  │
│  │     LOGS        │  │    METRICS      │  │   TRACES    │  │
│  ├─────────────────┤  ├─────────────────┤  ├─────────────┤  │
│  │                 │  │                 │  │             │  │
│  │  What happened  │  │   How system    │  │  Request    │  │
│  │  at a specific  │  │   is performing │  │  flow       │  │
│  │  point in time  │  │   over time     │  │  across     │  │
│  │                 │  │                 │  │  services   │  │
│  │  • Events       │  │  • Counters     │  │             │  │
│  │  • Errors       │  │  • Gauges       │  │  • Spans    │  │
│  │  • Debug info   │  │  • Histograms   │  │  • Context  │  │
│  │                 │  │  • Summaries    │  │  • Timing   │  │
│  │                 │  │                 │  │             │  │
│  │  Tools:         │  │  Tools:         │  │  Tools:     │  │
│  │  ELK, Loki      │  │  Prometheus     │  │  Jaeger     │  │
│  │  CloudWatch     │  │  Grafana        │  │  Zipkin     │  │
│  │                 │  │  DataDog        │  │  Tempo      │  │
│  └─────────────────┘  └─────────────────┘  └─────────────┘  │
│                                                              │
│                    ┌─────────────────────┐                   │
│                    │    CORRELATION      │                   │
│                    │   TraceId links     │                   │
│                    │   logs, metrics,    │                   │
│                    │   and traces        │                   │
│                    └─────────────────────┘                   │
└─────────────────────────────────────────────────────────────┘
```

### 1.2. Monitoring Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                 MONITORING ARCHITECTURE                      │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Applications                         Visualization          │
│  ┌─────────────┐                     ┌─────────────┐        │
│  │   App 1     │────┐                │   Grafana   │        │
│  └─────────────┘    │                │  Dashboards │        │
│  ┌─────────────┐    │   Collectors   └──────▲──────┘        │
│  │   App 2     │────┼──►┌──────────┐        │               │
│  └─────────────┘    │   │Prometheus│────────┘               │
│  ┌─────────────┐    │   │ (Metrics)│                        │
│  │   App 3     │────┘   └──────────┘                        │
│  └─────────────┘                                             │
│        │                                                     │
│        │ Logs         ┌──────────┐    ┌─────────────┐       │
│        └─────────────►│ Logstash │───►│Elasticsearch│       │
│                       │  (Logs)  │    │   (Index)   │       │
│                       └──────────┘    └──────▲──────┘       │
│                                              │               │
│                                        ┌─────┴─────┐        │
│                                        │  Kibana   │        │
│                                        │  (View)   │        │
│                                        └───────────┘        │
│                                                              │
│        Traces         ┌──────────┐    ┌─────────────┐       │
│        ─────────────►│  Tempo   │───►│  Grafana    │       │
│                       │ (Traces) │    │  (Trace UI) │       │
│                       └──────────┘    └─────────────┘       │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

## 2. Spring Boot Actuator

### 2.1. Thêm Dependencies

```xml
<!-- pom.xml -->
<dependencies>
    <!-- Spring Boot Actuator -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-actuator</artifactId>
    </dependency>

    <!-- Micrometer Prometheus Registry -->
    <dependency>
        <groupId>io.micrometer</groupId>
        <artifactId>micrometer-registry-prometheus</artifactId>
    </dependency>

    <!-- Micrometer Tracing with Brave -->
    <dependency>
        <groupId>io.micrometer</groupId>
        <artifactId>micrometer-tracing-bridge-brave</artifactId>
    </dependency>

    <!-- Zipkin Reporter -->
    <dependency>
        <groupId>io.zipkin.reporter2</groupId>
        <artifactId>zipkin-reporter-brave</artifactId>
    </dependency>
</dependencies>
```

### 2.2. Actuator Configuration

```yaml
# application.yml
management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics,prometheus,loggers,env,caches
      base-path: /actuator
    enabled-by-default: true

  endpoint:
    health:
      show-details: when_authorized
      show-components: when_authorized
      probes:
        enabled: true
      group:
        readiness:
          include: db, redis, elasticsearch
        liveness:
          include: livenessState

    metrics:
      enabled: true

    prometheus:
      enabled: true

    loggers:
      enabled: true

  health:
    livenessState:
      enabled: true
    readinessState:
      enabled: true
    db:
      enabled: true
    redis:
      enabled: true
    elasticsearch:
      enabled: true

  metrics:
    export:
      prometheus:
        enabled: true
    tags:
      application: ${spring.application.name}
      environment: ${spring.profiles.active:default}
    distribution:
      percentiles-histogram:
        http.server.requests: true
      slo:
        http.server.requests: 50ms, 100ms, 200ms, 500ms, 1s

  tracing:
    sampling:
      probability: 1.0  # 100% in dev, reduce in prod

  info:
    env:
      enabled: true
    build:
      enabled: true
    git:
      enabled: true
      mode: full

# Application info
info:
  app:
    name: ${spring.application.name}
    description: E-Commerce API Service
    version: '@project.version@'
    java:
      version: ${java.version}
  build:
    artifact: '@project.artifactId@'
    group: '@project.groupId@'
    version: '@project.version@'
    time: '@maven.build.timestamp@'
```

### 2.3. Custom Health Indicators

```java
package com.ecommerce.monitoring.health;

import org.springframework.boot.actuate.health.Health;
import org.springframework.boot.actuate.health.HealthIndicator;
import org.springframework.stereotype.Component;
import org.springframework.web.client.RestTemplate;

@Component
public class PaymentGatewayHealthIndicator implements HealthIndicator {

    private final RestTemplate restTemplate;
    private final String paymentGatewayUrl;

    public PaymentGatewayHealthIndicator(
            RestTemplate restTemplate,
            @Value("${payment.gateway.health-url}") String paymentGatewayUrl) {
        this.restTemplate = restTemplate;
        this.paymentGatewayUrl = paymentGatewayUrl;
    }

    @Override
    public Health health() {
        try {
            long startTime = System.currentTimeMillis();
            var response = restTemplate.getForEntity(paymentGatewayUrl, String.class);
            long responseTime = System.currentTimeMillis() - startTime;

            if (response.getStatusCode().is2xxSuccessful()) {
                return Health.up()
                        .withDetail("status", "Payment gateway is reachable")
                        .withDetail("responseTime", responseTime + "ms")
                        .build();
            } else {
                return Health.down()
                        .withDetail("status", "Payment gateway returned error")
                        .withDetail("statusCode", response.getStatusCode().value())
                        .build();
            }
        } catch (Exception e) {
            return Health.down()
                    .withDetail("status", "Payment gateway is unreachable")
                    .withDetail("error", e.getMessage())
                    .build();
        }
    }
}
```

```java
package com.ecommerce.monitoring.health;

import org.springframework.boot.actuate.health.Health;
import org.springframework.boot.actuate.health.HealthIndicator;
import org.springframework.data.redis.connection.RedisConnectionFactory;
import org.springframework.stereotype.Component;

@Component
public class RedisCacheHealthIndicator implements HealthIndicator {

    private final RedisConnectionFactory connectionFactory;

    public RedisCacheHealthIndicator(RedisConnectionFactory connectionFactory) {
        this.connectionFactory = connectionFactory;
    }

    @Override
    public Health health() {
        try {
            var connection = connectionFactory.getConnection();
            String pong = connection.ping();
            connection.close();

            if ("PONG".equals(pong)) {
                return Health.up()
                        .withDetail("status", "Redis is responding")
                        .build();
            } else {
                return Health.unknown()
                        .withDetail("status", "Unexpected response: " + pong)
                        .build();
            }
        } catch (Exception e) {
            return Health.down()
                    .withDetail("status", "Redis connection failed")
                    .withDetail("error", e.getMessage())
                    .build();
        }
    }
}
```

### 2.4. Custom Info Contributors

```java
package com.ecommerce.monitoring.info;

import org.springframework.boot.actuate.info.Info;
import org.springframework.boot.actuate.info.InfoContributor;
import org.springframework.stereotype.Component;

import java.lang.management.ManagementFactory;
import java.lang.management.RuntimeMXBean;
import java.time.Duration;
import java.time.Instant;
import java.util.HashMap;
import java.util.Map;

@Component
public class RuntimeInfoContributor implements InfoContributor {

    @Override
    public void contribute(Info.Builder builder) {
        RuntimeMXBean runtimeMXBean = ManagementFactory.getRuntimeMXBean();

        Map<String, Object> runtimeInfo = new HashMap<>();
        runtimeInfo.put("jvmName", runtimeMXBean.getVmName());
        runtimeInfo.put("jvmVendor", runtimeMXBean.getVmVendor());
        runtimeInfo.put("jvmVersion", runtimeMXBean.getVmVersion());
        runtimeInfo.put("startTime", Instant.ofEpochMilli(runtimeMXBean.getStartTime()).toString());
        runtimeInfo.put("uptime", formatDuration(Duration.ofMillis(runtimeMXBean.getUptime())));
        runtimeInfo.put("pid", runtimeMXBean.getPid());

        // Memory info
        Runtime runtime = Runtime.getRuntime();
        Map<String, Object> memoryInfo = new HashMap<>();
        memoryInfo.put("maxMemory", formatBytes(runtime.maxMemory()));
        memoryInfo.put("totalMemory", formatBytes(runtime.totalMemory()));
        memoryInfo.put("freeMemory", formatBytes(runtime.freeMemory()));
        memoryInfo.put("usedMemory", formatBytes(runtime.totalMemory() - runtime.freeMemory()));

        builder.withDetail("runtime", runtimeInfo);
        builder.withDetail("memory", memoryInfo);
    }

    private String formatDuration(Duration duration) {
        long days = duration.toDays();
        long hours = duration.toHours() % 24;
        long minutes = duration.toMinutes() % 60;
        long seconds = duration.getSeconds() % 60;

        return String.format("%dd %dh %dm %ds", days, hours, minutes, seconds);
    }

    private String formatBytes(long bytes) {
        if (bytes < 1024) return bytes + " B";
        int exp = (int) (Math.log(bytes) / Math.log(1024));
        String pre = "KMGTPE".charAt(exp - 1) + "";
        return String.format("%.2f %sB", bytes / Math.pow(1024, exp), pre);
    }
}
```

## 3. Structured Logging

### 3.1. Logback Configuration

```xml
<!-- src/main/resources/logback-spring.xml -->
<?xml version="1.0" encoding="UTF-8"?>
<configuration scan="true" scanPeriod="30 seconds">

    <!-- Include Spring Boot defaults -->
    <include resource="org/springframework/boot/logging/logback/defaults.xml"/>

    <!-- Properties -->
    <springProperty scope="context" name="APP_NAME" source="spring.application.name" defaultValue="ecommerce"/>
    <springProperty scope="context" name="LOG_PATH" source="logging.file.path" defaultValue="logs"/>

    <!-- Console Appender for Development -->
    <appender name="CONSOLE" class="ch.qos.logback.core.ConsoleAppender">
        <encoder>
            <pattern>%d{yyyy-MM-dd HH:mm:ss.SSS} %highlight(%-5level) [%thread] [%X{traceId:-},%X{spanId:-}] %cyan(%logger{36}) - %msg%n</pattern>
            <charset>UTF-8</charset>
        </encoder>
    </appender>

    <!-- JSON Appender for Production (ELK) -->
    <appender name="JSON_FILE" class="ch.qos.logback.core.rolling.RollingFileAppender">
        <file>${LOG_PATH}/${APP_NAME}.json</file>
        <rollingPolicy class="ch.qos.logback.core.rolling.SizeAndTimeBasedRollingPolicy">
            <fileNamePattern>${LOG_PATH}/${APP_NAME}-%d{yyyy-MM-dd}.%i.json.gz</fileNamePattern>
            <maxFileSize>100MB</maxFileSize>
            <maxHistory>30</maxHistory>
            <totalSizeCap>3GB</totalSizeCap>
        </rollingPolicy>
        <encoder class="net.logstash.logback.encoder.LogstashEncoder">
            <includeMdcKeyName>traceId</includeMdcKeyName>
            <includeMdcKeyName>spanId</includeMdcKeyName>
            <includeMdcKeyName>userId</includeMdcKeyName>
            <includeMdcKeyName>requestId</includeMdcKeyName>
            <customFields>{"application":"${APP_NAME}","environment":"${ENVIRONMENT:-local}"}</customFields>
            <timeZone>UTC</timeZone>
        </encoder>
    </appender>

    <!-- Async Appender for better performance -->
    <appender name="ASYNC_JSON" class="ch.qos.logback.classic.AsyncAppender">
        <appender-ref ref="JSON_FILE"/>
        <queueSize>512</queueSize>
        <discardingThreshold>0</discardingThreshold>
        <neverBlock>true</neverBlock>
    </appender>

    <!-- Rolling File Appender for Application logs -->
    <appender name="FILE" class="ch.qos.logback.core.rolling.RollingFileAppender">
        <file>${LOG_PATH}/${APP_NAME}.log</file>
        <rollingPolicy class="ch.qos.logback.core.rolling.SizeAndTimeBasedRollingPolicy">
            <fileNamePattern>${LOG_PATH}/${APP_NAME}-%d{yyyy-MM-dd}.%i.log.gz</fileNamePattern>
            <maxFileSize>100MB</maxFileSize>
            <maxHistory>30</maxHistory>
            <totalSizeCap>3GB</totalSizeCap>
        </rollingPolicy>
        <encoder>
            <pattern>%d{yyyy-MM-dd HH:mm:ss.SSS} %-5level [%thread] [%X{traceId:-},%X{spanId:-}] %logger{36} - %msg%n</pattern>
        </encoder>
    </appender>

    <!-- Error File Appender -->
    <appender name="ERROR_FILE" class="ch.qos.logback.core.rolling.RollingFileAppender">
        <file>${LOG_PATH}/${APP_NAME}-error.log</file>
        <filter class="ch.qos.logback.classic.filter.ThresholdFilter">
            <level>ERROR</level>
        </filter>
        <rollingPolicy class="ch.qos.logback.core.rolling.SizeAndTimeBasedRollingPolicy">
            <fileNamePattern>${LOG_PATH}/${APP_NAME}-error-%d{yyyy-MM-dd}.%i.log.gz</fileNamePattern>
            <maxFileSize>50MB</maxFileSize>
            <maxHistory>30</maxHistory>
            <totalSizeCap>1GB</totalSizeCap>
        </rollingPolicy>
        <encoder>
            <pattern>%d{yyyy-MM-dd HH:mm:ss.SSS} %-5level [%thread] [%X{traceId:-}] %logger{36} - %msg%n%ex</pattern>
        </encoder>
    </appender>

    <!-- Logger configurations -->
    <logger name="com.ecommerce" level="DEBUG"/>
    <logger name="org.springframework" level="INFO"/>
    <logger name="org.hibernate.SQL" level="DEBUG"/>
    <logger name="org.hibernate.type.descriptor.sql" level="TRACE"/>

    <!-- Profile-specific configurations -->
    <springProfile name="dev,local">
        <root level="INFO">
            <appender-ref ref="CONSOLE"/>
            <appender-ref ref="FILE"/>
        </root>
    </springProfile>

    <springProfile name="prod,production">
        <root level="INFO">
            <appender-ref ref="ASYNC_JSON"/>
            <appender-ref ref="ERROR_FILE"/>
        </root>
    </springProfile>

</configuration>
```

### 3.2. Structured Logging Service

```java
package com.ecommerce.logging;

import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.slf4j.MDC;
import org.springframework.stereotype.Component;

import java.util.HashMap;
import java.util.Map;
import java.util.function.Supplier;

@Component
public class StructuredLogger {

    private final Logger logger;

    public StructuredLogger() {
        this.logger = LoggerFactory.getLogger(StructuredLogger.class);
    }

    public static StructuredLogger getLogger(Class<?> clazz) {
        return new StructuredLogger(clazz);
    }

    private StructuredLogger(Class<?> clazz) {
        this.logger = LoggerFactory.getLogger(clazz);
    }

    public void info(String message, Map<String, Object> context) {
        withContext(context, () -> logger.info(formatMessage(message, context)));
    }

    public void error(String message, Throwable throwable, Map<String, Object> context) {
        withContext(context, () -> logger.error(formatMessage(message, context), throwable));
    }

    public void warn(String message, Map<String, Object> context) {
        withContext(context, () -> logger.warn(formatMessage(message, context)));
    }

    public void debug(String message, Map<String, Object> context) {
        withContext(context, () -> logger.debug(formatMessage(message, context)));
    }

    // Business event logging
    public void logBusinessEvent(String eventType, String entityType,
                                  String entityId, Map<String, Object> details) {
        Map<String, Object> context = new HashMap<>();
        context.put("eventType", eventType);
        context.put("entityType", entityType);
        context.put("entityId", entityId);
        context.putAll(details);

        info("Business event: " + eventType, context);
    }

    // Performance logging
    public <T> T logPerformance(String operation, Supplier<T> action) {
        long startTime = System.currentTimeMillis();
        try {
            T result = action.get();
            long duration = System.currentTimeMillis() - startTime;

            Map<String, Object> context = new HashMap<>();
            context.put("operation", operation);
            context.put("duration", duration);
            context.put("status", "success");

            if (duration > 1000) {
                warn("Slow operation detected", context);
            } else {
                debug("Operation completed", context);
            }

            return result;
        } catch (Exception e) {
            long duration = System.currentTimeMillis() - startTime;

            Map<String, Object> context = new HashMap<>();
            context.put("operation", operation);
            context.put("duration", duration);
            context.put("status", "error");
            context.put("errorType", e.getClass().getSimpleName());

            error("Operation failed", e, context);
            throw e;
        }
    }

    private void withContext(Map<String, Object> context, Runnable action) {
        try {
            context.forEach((key, value) -> {
                if (value != null) {
                    MDC.put(key, value.toString());
                }
            });
            action.run();
        } finally {
            context.keySet().forEach(MDC::remove);
        }
    }

    private String formatMessage(String message, Map<String, Object> context) {
        if (context == null || context.isEmpty()) {
            return message;
        }
        StringBuilder sb = new StringBuilder(message);
        sb.append(" | ");
        context.forEach((key, value) ->
            sb.append(key).append("=").append(value).append(" "));
        return sb.toString().trim();
    }
}
```

### 3.3. Request Logging Filter

```java
package com.ecommerce.logging;

import jakarta.servlet.FilterChain;
import jakarta.servlet.ServletException;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.slf4j.MDC;
import org.springframework.core.Ordered;
import org.springframework.core.annotation.Order;
import org.springframework.stereotype.Component;
import org.springframework.web.filter.OncePerRequestFilter;
import org.springframework.web.util.ContentCachingRequestWrapper;
import org.springframework.web.util.ContentCachingResponseWrapper;

import java.io.IOException;
import java.util.UUID;

@Component
@Order(Ordered.HIGHEST_PRECEDENCE)
public class RequestLoggingFilter extends OncePerRequestFilter {

    private static final Logger log = LoggerFactory.getLogger(RequestLoggingFilter.class);

    private static final String REQUEST_ID_HEADER = "X-Request-ID";
    private static final String REQUEST_ID_MDC_KEY = "requestId";
    private static final String USER_ID_MDC_KEY = "userId";

    @Override
    protected void doFilterInternal(HttpServletRequest request,
                                    HttpServletResponse response,
                                    FilterChain filterChain)
            throws ServletException, IOException {

        // Skip actuator endpoints
        if (request.getRequestURI().startsWith("/actuator")) {
            filterChain.doFilter(request, response);
            return;
        }

        // Generate or extract request ID
        String requestId = request.getHeader(REQUEST_ID_HEADER);
        if (requestId == null || requestId.isEmpty()) {
            requestId = UUID.randomUUID().toString().substring(0, 8);
        }

        // Wrap request and response for logging body
        ContentCachingRequestWrapper wrappedRequest =
            new ContentCachingRequestWrapper(request);
        ContentCachingResponseWrapper wrappedResponse =
            new ContentCachingResponseWrapper(response);

        long startTime = System.currentTimeMillis();

        try {
            // Set MDC context
            MDC.put(REQUEST_ID_MDC_KEY, requestId);

            // Add request ID to response header
            response.setHeader(REQUEST_ID_HEADER, requestId);

            // Log request
            logRequest(wrappedRequest, requestId);

            // Execute request
            filterChain.doFilter(wrappedRequest, wrappedResponse);

        } finally {
            long duration = System.currentTimeMillis() - startTime;

            // Log response
            logResponse(wrappedRequest, wrappedResponse, requestId, duration);

            // Copy body to response
            wrappedResponse.copyBodyToResponse();

            // Clear MDC
            MDC.remove(REQUEST_ID_MDC_KEY);
            MDC.remove(USER_ID_MDC_KEY);
        }
    }

    private void logRequest(ContentCachingRequestWrapper request, String requestId) {
        String method = request.getMethod();
        String uri = request.getRequestURI();
        String queryString = request.getQueryString();
        String clientIp = getClientIp(request);
        String userAgent = request.getHeader("User-Agent");

        log.info(">>> REQUEST [{}] {} {} | IP: {} | UA: {}",
                requestId, method, uri + (queryString != null ? "?" + queryString : ""),
                clientIp, userAgent);
    }

    private void logResponse(ContentCachingRequestWrapper request,
                            ContentCachingResponseWrapper response,
                            String requestId, long duration) {
        int status = response.getStatus();
        String method = request.getMethod();
        String uri = request.getRequestURI();

        String logLevel = status >= 500 ? "ERROR" : status >= 400 ? "WARN" : "INFO";

        if (status >= 400) {
            log.warn("<<< RESPONSE [{}] {} {} | Status: {} | Duration: {}ms",
                    requestId, method, uri, status, duration);
        } else {
            log.info("<<< RESPONSE [{}] {} {} | Status: {} | Duration: {}ms",
                    requestId, method, uri, status, duration);
        }

        // Log slow requests
        if (duration > 3000) {
            log.warn("SLOW REQUEST [{}] {} {} took {}ms",
                    requestId, method, uri, duration);
        }
    }

    private String getClientIp(HttpServletRequest request) {
        String ip = request.getHeader("X-Forwarded-For");
        if (ip == null || ip.isEmpty() || "unknown".equalsIgnoreCase(ip)) {
            ip = request.getHeader("X-Real-IP");
        }
        if (ip == null || ip.isEmpty() || "unknown".equalsIgnoreCase(ip)) {
            ip = request.getRemoteAddr();
        }
        // Get first IP if multiple
        if (ip != null && ip.contains(",")) {
            ip = ip.split(",")[0].trim();
        }
        return ip;
    }
}
```

## 4. Custom Metrics

### 4.1. Custom Metrics Configuration

```java
package com.ecommerce.monitoring.metrics;

import io.micrometer.core.instrument.Counter;
import io.micrometer.core.instrument.MeterRegistry;
import io.micrometer.core.instrument.Timer;
import io.micrometer.core.instrument.binder.MeterBinder;
import org.springframework.stereotype.Component;

import java.util.concurrent.atomic.AtomicInteger;

@Component
public class BusinessMetrics implements MeterBinder {

    private Counter ordersCreated;
    private Counter ordersCompleted;
    private Counter ordersCancelled;
    private Counter paymentSuccessful;
    private Counter paymentFailed;
    private Timer orderProcessingTime;
    private Timer paymentProcessingTime;
    private AtomicInteger activeUsers;
    private AtomicInteger cartItems;

    @Override
    public void bindTo(MeterRegistry registry) {
        // Order metrics
        ordersCreated = Counter.builder("ecommerce.orders.created")
                .description("Number of orders created")
                .tag("type", "order")
                .register(registry);

        ordersCompleted = Counter.builder("ecommerce.orders.completed")
                .description("Number of orders completed")
                .tag("type", "order")
                .register(registry);

        ordersCancelled = Counter.builder("ecommerce.orders.cancelled")
                .description("Number of orders cancelled")
                .tag("type", "order")
                .register(registry);

        // Payment metrics
        paymentSuccessful = Counter.builder("ecommerce.payments.success")
                .description("Number of successful payments")
                .tag("type", "payment")
                .register(registry);

        paymentFailed = Counter.builder("ecommerce.payments.failed")
                .description("Number of failed payments")
                .tag("type", "payment")
                .register(registry);

        // Timing metrics
        orderProcessingTime = Timer.builder("ecommerce.orders.processing.time")
                .description("Time taken to process orders")
                .register(registry);

        paymentProcessingTime = Timer.builder("ecommerce.payments.processing.time")
                .description("Time taken to process payments")
                .register(registry);

        // Gauge metrics
        activeUsers = registry.gauge("ecommerce.users.active", new AtomicInteger(0));
        cartItems = registry.gauge("ecommerce.cart.items.total", new AtomicInteger(0));
    }

    // Public methods to record metrics
    public void recordOrderCreated() {
        ordersCreated.increment();
    }

    public void recordOrderCompleted() {
        ordersCompleted.increment();
    }

    public void recordOrderCancelled() {
        ordersCancelled.increment();
    }

    public void recordPaymentSuccess() {
        paymentSuccessful.increment();
    }

    public void recordPaymentFailure() {
        paymentFailed.increment();
    }

    public Timer.Sample startOrderProcessing() {
        return Timer.start();
    }

    public void stopOrderProcessing(Timer.Sample sample) {
        sample.stop(orderProcessingTime);
    }

    public Timer.Sample startPaymentProcessing() {
        return Timer.start();
    }

    public void stopPaymentProcessing(Timer.Sample sample) {
        sample.stop(paymentProcessingTime);
    }

    public void setActiveUsers(int count) {
        activeUsers.set(count);
    }

    public void setCartItems(int count) {
        cartItems.set(count);
    }
}
```

### 4.2. Metrics Aspect

```java
package com.ecommerce.monitoring.metrics;

import io.micrometer.core.instrument.MeterRegistry;
import io.micrometer.core.instrument.Timer;
import org.aspectj.lang.ProceedingJoinPoint;
import org.aspectj.lang.annotation.Around;
import org.aspectj.lang.annotation.Aspect;
import org.aspectj.lang.reflect.MethodSignature;
import org.springframework.stereotype.Component;

import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

@Aspect
@Component
public class MetricsAspect {

    private final MeterRegistry registry;

    public MetricsAspect(MeterRegistry registry) {
        this.registry = registry;
    }

    @Around("@annotation(timed)")
    public Object measureExecutionTime(ProceedingJoinPoint joinPoint, Timed timed)
            throws Throwable {

        String metricName = timed.value().isEmpty() ?
                buildDefaultMetricName(joinPoint) : timed.value();

        Timer.Sample sample = Timer.start(registry);
        String status = "success";

        try {
            return joinPoint.proceed();
        } catch (Exception e) {
            status = "error";
            throw e;
        } finally {
            sample.stop(Timer.builder(metricName)
                    .tag("class", joinPoint.getTarget().getClass().getSimpleName())
                    .tag("method", joinPoint.getSignature().getName())
                    .tag("status", status)
                    .description(timed.description())
                    .register(registry));
        }
    }

    @Around("@annotation(counted)")
    public Object countMethodCalls(ProceedingJoinPoint joinPoint, Counted counted)
            throws Throwable {

        String metricName = counted.value().isEmpty() ?
                buildDefaultMetricName(joinPoint) + ".count" : counted.value();

        try {
            Object result = joinPoint.proceed();
            registry.counter(metricName,
                    "class", joinPoint.getTarget().getClass().getSimpleName(),
                    "method", joinPoint.getSignature().getName(),
                    "status", "success"
            ).increment();
            return result;
        } catch (Exception e) {
            registry.counter(metricName,
                    "class", joinPoint.getTarget().getClass().getSimpleName(),
                    "method", joinPoint.getSignature().getName(),
                    "status", "error"
            ).increment();
            throw e;
        }
    }

    private String buildDefaultMetricName(ProceedingJoinPoint joinPoint) {
        MethodSignature signature = (MethodSignature) joinPoint.getSignature();
        return signature.getDeclaringTypeName() + "." + signature.getName();
    }
}

// Custom annotations
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
public @interface Timed {
    String value() default "";
    String description() default "";
}

@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
public @interface Counted {
    String value() default "";
}
```

### 4.3. Using Custom Metrics

```java
package com.ecommerce.order.service;

import com.ecommerce.monitoring.metrics.BusinessMetrics;
import com.ecommerce.monitoring.metrics.Timed;
import com.ecommerce.monitoring.metrics.Counted;
import io.micrometer.core.instrument.Timer;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

@Service
public class OrderService {

    private final BusinessMetrics metrics;
    private final OrderRepository orderRepository;
    private final PaymentService paymentService;

    public OrderService(BusinessMetrics metrics,
                       OrderRepository orderRepository,
                       PaymentService paymentService) {
        this.metrics = metrics;
        this.orderRepository = orderRepository;
        this.paymentService = paymentService;
    }

    @Transactional
    @Timed(value = "ecommerce.order.create", description = "Time to create order")
    @Counted(value = "ecommerce.order.create.count")
    public Order createOrder(CreateOrderRequest request) {
        Timer.Sample processingTimer = metrics.startOrderProcessing();

        try {
            Order order = new Order();
            // ... order creation logic

            Order savedOrder = orderRepository.save(order);

            // Record success metric
            metrics.recordOrderCreated();

            return savedOrder;
        } finally {
            metrics.stopOrderProcessing(processingTimer);
        }
    }

    @Transactional
    public void processPayment(Long orderId, PaymentRequest request) {
        Timer.Sample paymentTimer = metrics.startPaymentProcessing();

        try {
            PaymentResult result = paymentService.processPayment(request);

            if (result.isSuccessful()) {
                metrics.recordPaymentSuccess();
                updateOrderStatus(orderId, OrderStatus.PAID);
            } else {
                metrics.recordPaymentFailure();
                updateOrderStatus(orderId, OrderStatus.PAYMENT_FAILED);
            }
        } finally {
            metrics.stopPaymentProcessing(paymentTimer);
        }
    }

    @Transactional
    public void completeOrder(Long orderId) {
        Order order = orderRepository.findById(orderId)
                .orElseThrow(() -> new OrderNotFoundException(orderId));

        order.setStatus(OrderStatus.COMPLETED);
        orderRepository.save(order);

        metrics.recordOrderCompleted();
    }

    @Transactional
    public void cancelOrder(Long orderId, String reason) {
        Order order = orderRepository.findById(orderId)
                .orElseThrow(() -> new OrderNotFoundException(orderId));

        order.setStatus(OrderStatus.CANCELLED);
        order.setCancellationReason(reason);
        orderRepository.save(order);

        metrics.recordOrderCancelled();
    }
}
```

## 5. Prometheus & Grafana Setup

### 5.1. Prometheus Configuration

```yaml
# docker/prometheus/prometheus.yml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

alerting:
  alertmanagers:
    - static_configs:
        - targets:
            - alertmanager:9093

rule_files:
  - /etc/prometheus/rules/*.yml

scrape_configs:
  # Prometheus self-monitoring
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']

  # Spring Boot applications
  - job_name: 'ecommerce-api'
    metrics_path: '/actuator/prometheus'
    scrape_interval: 10s
    static_configs:
      - targets: ['app:8080']
    relabel_configs:
      - source_labels: [__address__]
        target_label: instance
        regex: '([^:]+):\d+'
        replacement: '${1}'

  # MySQL Exporter
  - job_name: 'mysql'
    static_configs:
      - targets: ['mysql-exporter:9104']

  # Redis Exporter
  - job_name: 'redis'
    static_configs:
      - targets: ['redis-exporter:9121']

  # Node Exporter (host metrics)
  - job_name: 'node'
    static_configs:
      - targets: ['node-exporter:9100']
```

### 5.2. Alert Rules

```yaml
# docker/prometheus/rules/alerts.yml
groups:
  - name: ecommerce-alerts
    rules:
      # High error rate
      - alert: HighErrorRate
        expr: |
          sum(rate(http_server_requests_seconds_count{status=~"5.."}[5m]))
          / sum(rate(http_server_requests_seconds_count[5m])) > 0.05
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "High HTTP error rate detected"
          description: "Error rate is {{ $value | humanizePercentage }} over 5%"

      # High response time
      - alert: HighResponseTime
        expr: |
          histogram_quantile(0.95,
            sum(rate(http_server_requests_seconds_bucket[5m])) by (le)
          ) > 2
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "High response time detected"
          description: "95th percentile response time is {{ $value | humanizeDuration }}"

      # Application down
      - alert: ApplicationDown
        expr: up{job="ecommerce-api"} == 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "E-Commerce API is down"
          description: "Application {{ $labels.instance }} is not responding"

      # High memory usage
      - alert: HighMemoryUsage
        expr: |
          jvm_memory_used_bytes{area="heap"}
          / jvm_memory_max_bytes{area="heap"} > 0.9
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "High JVM heap memory usage"
          description: "JVM heap usage is {{ $value | humanizePercentage }}"

      # Database connection pool exhaustion
      - alert: DatabaseConnectionPoolExhausted
        expr: |
          hikaricp_connections_active
          / hikaricp_connections_max > 0.9
        for: 2m
        labels:
          severity: critical
        annotations:
          summary: "Database connection pool nearly exhausted"
          description: "{{ $value | humanizePercentage }} of database connections are in use"

      # Order processing failures
      - alert: OrderProcessingFailures
        expr: |
          increase(ecommerce_orders_cancelled_total[1h]) > 100
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "High order cancellation rate"
          description: "{{ $value }} orders cancelled in the last hour"

      # Payment failures
      - alert: PaymentFailureSpike
        expr: |
          sum(rate(ecommerce_payments_failed_total[5m]))
          / sum(rate(ecommerce_payments_success_total[5m])) > 0.1
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "High payment failure rate"
          description: "Payment failure rate is {{ $value | humanizePercentage }}"
```

### 5.3. Grafana Dashboard JSON

```json
{
  "dashboard": {
    "title": "E-Commerce Application Dashboard",
    "panels": [
      {
        "title": "Request Rate",
        "type": "graph",
        "gridPos": { "h": 8, "w": 12, "x": 0, "y": 0 },
        "targets": [
          {
            "expr": "sum(rate(http_server_requests_seconds_count[5m])) by (uri)",
            "legendFormat": "{{uri}}"
          }
        ]
      },
      {
        "title": "Response Time (P95)",
        "type": "graph",
        "gridPos": { "h": 8, "w": 12, "x": 12, "y": 0 },
        "targets": [
          {
            "expr": "histogram_quantile(0.95, sum(rate(http_server_requests_seconds_bucket[5m])) by (le, uri))",
            "legendFormat": "{{uri}}"
          }
        ]
      },
      {
        "title": "Error Rate",
        "type": "stat",
        "gridPos": { "h": 4, "w": 6, "x": 0, "y": 8 },
        "targets": [
          {
            "expr": "sum(rate(http_server_requests_seconds_count{status=~\"5..\"}[5m])) / sum(rate(http_server_requests_seconds_count[5m])) * 100",
            "legendFormat": "Error %"
          }
        ],
        "options": {
          "colorMode": "value",
          "thresholds": {
            "steps": [
              { "color": "green", "value": 0 },
              { "color": "yellow", "value": 1 },
              { "color": "red", "value": 5 }
            ]
          }
        }
      },
      {
        "title": "Orders Created",
        "type": "stat",
        "gridPos": { "h": 4, "w": 6, "x": 6, "y": 8 },
        "targets": [
          {
            "expr": "increase(ecommerce_orders_created_total[24h])",
            "legendFormat": "Orders (24h)"
          }
        ]
      },
      {
        "title": "JVM Memory Usage",
        "type": "gauge",
        "gridPos": { "h": 4, "w": 6, "x": 12, "y": 8 },
        "targets": [
          {
            "expr": "jvm_memory_used_bytes{area=\"heap\"} / jvm_memory_max_bytes{area=\"heap\"} * 100",
            "legendFormat": "Heap Usage %"
          }
        ]
      },
      {
        "title": "Active Database Connections",
        "type": "gauge",
        "gridPos": { "h": 4, "w": 6, "x": 18, "y": 8 },
        "targets": [
          {
            "expr": "hikaricp_connections_active",
            "legendFormat": "Active"
          }
        ]
      }
    ]
  }
}
```

## 6. Docker Compose cho Monitoring Stack

```yaml
# docker-compose.monitoring.yml
version: '3.9'

services:
  # Prometheus
  prometheus:
    image: prom/prometheus:v2.48.0
    container_name: prometheus
    restart: unless-stopped
    ports:
      - "9090:9090"
    volumes:
      - ./docker/prometheus/prometheus.yml:/etc/prometheus/prometheus.yml:ro
      - ./docker/prometheus/rules:/etc/prometheus/rules:ro
      - prometheus-data:/prometheus
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.path=/prometheus'
      - '--web.enable-lifecycle'
      - '--storage.tsdb.retention.time=15d'
    networks:
      - monitoring

  # Grafana
  grafana:
    image: grafana/grafana:10.2.0
    container_name: grafana
    restart: unless-stopped
    ports:
      - "3000:3000"
    environment:
      - GF_SECURITY_ADMIN_USER=admin
      - GF_SECURITY_ADMIN_PASSWORD=admin123
      - GF_USERS_ALLOW_SIGN_UP=false
    volumes:
      - grafana-data:/var/lib/grafana
      - ./docker/grafana/provisioning:/etc/grafana/provisioning:ro
      - ./docker/grafana/dashboards:/var/lib/grafana/dashboards:ro
    depends_on:
      - prometheus
    networks:
      - monitoring

  # Alertmanager
  alertmanager:
    image: prom/alertmanager:v0.26.0
    container_name: alertmanager
    restart: unless-stopped
    ports:
      - "9093:9093"
    volumes:
      - ./docker/alertmanager/alertmanager.yml:/etc/alertmanager/alertmanager.yml:ro
    command:
      - '--config.file=/etc/alertmanager/alertmanager.yml'
    networks:
      - monitoring

  # Elasticsearch (for logs)
  elasticsearch:
    image: elasticsearch:8.11.0
    container_name: elasticsearch
    restart: unless-stopped
    ports:
      - "9200:9200"
    environment:
      - discovery.type=single-node
      - xpack.security.enabled=false
      - ES_JAVA_OPTS=-Xms1g -Xmx1g
    volumes:
      - elasticsearch-data:/usr/share/elasticsearch/data
    networks:
      - monitoring

  # Logstash
  logstash:
    image: logstash:8.11.0
    container_name: logstash
    restart: unless-stopped
    ports:
      - "5044:5044"
    volumes:
      - ./docker/logstash/pipeline:/usr/share/logstash/pipeline:ro
    depends_on:
      - elasticsearch
    networks:
      - monitoring

  # Kibana
  kibana:
    image: kibana:8.11.0
    container_name: kibana
    restart: unless-stopped
    ports:
      - "5601:5601"
    environment:
      - ELASTICSEARCH_HOSTS=http://elasticsearch:9200
    depends_on:
      - elasticsearch
    networks:
      - monitoring

  # Jaeger (for distributed tracing)
  jaeger:
    image: jaegertracing/all-in-one:1.52
    container_name: jaeger
    restart: unless-stopped
    ports:
      - "16686:16686"  # UI
      - "14268:14268"  # HTTP collector
      - "6831:6831/udp"  # Thrift compact
    environment:
      - COLLECTOR_OTLP_ENABLED=true
    networks:
      - monitoring

networks:
  monitoring:
    driver: bridge

volumes:
  prometheus-data:
  grafana-data:
  elasticsearch-data:
```

## 7. Logstash Pipeline

```ruby
# docker/logstash/pipeline/logstash.conf
input {
  beats {
    port => 5044
  }

  tcp {
    port => 5000
    codec => json_lines
  }
}

filter {
  if [type] == "spring-boot" {
    json {
      source => "message"
    }

    date {
      match => ["timestamp", "ISO8601"]
      target => "@timestamp"
    }

    mutate {
      remove_field => ["timestamp", "host", "port"]
    }

    if [level] == "ERROR" {
      mutate {
        add_tag => ["error"]
      }
    }
  }

  # Extract trace information
  if [traceId] {
    mutate {
      add_field => { "trace_link" => "http://jaeger:16686/trace/%{traceId}" }
    }
  }

  # Geoip for client IP
  if [clientIp] {
    geoip {
      source => "clientIp"
      target => "geoip"
    }
  }
}

output {
  elasticsearch {
    hosts => ["elasticsearch:9200"]
    index => "ecommerce-logs-%{+YYYY.MM.dd}"
  }

  # Output errors to separate index
  if "error" in [tags] {
    elasticsearch {
      hosts => ["elasticsearch:9200"]
      index => "ecommerce-errors-%{+YYYY.MM.dd}"
    }
  }
}
```

## 8. Best Practices

```
┌─────────────────────────────────────────────────────────────┐
│           MONITORING BEST PRACTICES                         │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  ✅ Logging Best Practices                                   │
│     • Use structured JSON logging in production              │
│     • Include correlation IDs (traceId, requestId)          │
│     • Log at appropriate levels (not everything as INFO)    │
│     • Don't log sensitive data (passwords, tokens)          │
│     • Use async logging for performance                      │
│                                                              │
│  ✅ Metrics Best Practices                                   │
│     • Use meaningful metric names                            │
│     • Add relevant tags/labels for filtering                 │
│     • Set up histograms for latency (not just averages)     │
│     • Monitor RED metrics (Rate, Errors, Duration)          │
│     • Track business metrics, not just technical            │
│                                                              │
│  ✅ Alerting Best Practices                                  │
│     • Alert on symptoms, not causes                          │
│     • Set appropriate thresholds (avoid alert fatigue)      │
│     • Include runbooks in alert descriptions                 │
│     • Use multi-window alerts to reduce flapping            │
│     • Route alerts to appropriate teams                      │
│                                                              │
│  ✅ Health Check Best Practices                              │
│     • Separate liveness and readiness probes                │
│     • Check all critical dependencies                        │
│     • Set appropriate timeouts                               │
│     • Graceful degradation when deps are down               │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

## 9. Bài tập thực hành

### Bài tập 1: Structured Logging
Implement structured logging với:
- Request/Response logging
- MDC context (userId, traceId)
- Different log levels cho environments

### Bài tập 2: Custom Metrics
Tạo custom metrics cho:
- Business events (orders, payments)
- Performance monitoring
- Error tracking

### Bài tập 3: Dashboard Setup
Setup Grafana dashboard với:
- Request rate và response time
- Error rate và types
- JVM và database metrics
- Business metrics visualization

## Tổng kết

Hôm nay đã học:
- ✅ Three pillars of observability
- ✅ Spring Boot Actuator configuration
- ✅ Custom health indicators
- ✅ Structured logging với Logback
- ✅ Custom metrics với Micrometer
- ✅ Prometheus và Grafana setup
- ✅ ELK Stack cho log aggregation
- ✅ Alerting và best practices

## Navigation

- [← Day 28: Docker & Deployment](./day-28-docker-deployment.md)
- [Day 30: Project Summary →](./day-30-project-summary.md)
