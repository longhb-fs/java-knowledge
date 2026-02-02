# Ngày 01: Java Ecosystem & Spring Boot Tổng Quan

## Mục tiêu hôm nay
- Hiểu hệ sinh thái Java và vị trí của Spring Boot
- Phân biệt JDK, JRE, JVM
- Hiểu kiến trúc Spring Framework & Spring Boot
- Cài đặt môi trường phát triển hoàn chỉnh
- Chạy được ứng dụng Spring Boot đầu tiên

---

## 1. HỆ SINH THÁI JAVA

### 1.1. Java là gì? Tại sao vẫn phổ biến?

Java ra đời năm 1995 bởi Sun Microsystems (nay thuộc Oracle). Tính đến nay, Java vẫn nằm trong **top 3 ngôn ngữ phổ biến nhất thế giới** vì:

| Đặc điểm | Giải thích |
|-----------|------------|
| **Write Once, Run Anywhere** | Code Java chạy trên mọi OS có JVM (Windows, Linux, Mac) |
| **Strongly Typed** | Phát hiện lỗi tại compile time, giảm bug runtime |
| **Garbage Collection** | Tự động quản lý memory, không cần free/malloc |
| **Enterprise Grade** | Hệ sinh thái khổng lồ cho enterprise (banking, logistics, e-commerce) |
| **Backward Compatible** | Code Java 8 vẫn chạy được trên JDK 21 |
| **Community** | Hàng triệu developer, tài liệu phong phú |

### 1.2. JVM, JRE, JDK - Phân biệt rõ ràng

```
┌─────────────────────────────────────────────────┐
│                    JDK (Java Development Kit)     │
│  ┌─────────────────────────────────────────────┐ │
│  │              JRE (Java Runtime Environment)  │ │
│  │  ┌─────────────────────────────────────────┐│ │
│  │  │          JVM (Java Virtual Machine)      ││ │
│  │  │                                          ││ │
│  │  │  ┌──────────┐  ┌──────────┐  ┌────────┐││ │
│  │  │  │Class     │  │Execution │  │Garbage │││ │
│  │  │  │Loader    │  │Engine    │  │Collect.│││ │
│  │  │  └──────────┘  └──────────┘  └────────┘││ │
│  │  │                                          ││ │
│  │  └─────────────────────────────────────────┘│ │
│  │                                               │ │
│  │  + Java Class Libraries (rt.jar, etc.)        │ │
│  │  + Java Runtime (java command)                │ │
│  └─────────────────────────────────────────────┘ │
│                                                   │
│  + javac (compiler)                               │
│  + jar (archiver)                                 │
│  + javadoc (documentation)                        │
│  + jshell (REPL - từ Java 9)                     │
│  + jlink, jpackage (từ Java 14+)                 │
└─────────────────────────────────────────────────┘
```

**Tóm lại:**
- **JVM**: Máy ảo chạy bytecode `.class` - trái tim của Java
- **JRE**: JVM + thư viện chuẩn - đủ để CHẠY ứng dụng
- **JDK**: JRE + công cụ phát triển (compiler, debugger) - cần để PHÁT TRIỂN

### 1.3. Java Versions - Chọn version nào?

```
Java 8  (2014)  ─── LTS ─── Lambda, Streams, Optional, Date/Time API
    │                        ↑ Vẫn được dùng rất nhiều trong enterprise
    ▼
Java 11 (2018)  ─── LTS ─── var keyword, HTTP Client, Module System
    │
    ▼
Java 17 (2021)  ─── LTS ─── Records, Sealed Classes, Pattern Matching
    │
    ▼
Java 21 (2023)  ─── LTS ─── Virtual Threads, Pattern Matching switch
                             ↑ CHÚNG TA DÙNG VERSION NÀY
```

**Tại sao Java 21?**
- LTS (Long-Term Support) - được hỗ trợ lâu dài
- **Virtual Threads** - xử lý hàng triệu request concurrent
- **Pattern Matching** - code gọn gàng hơn
- Spring Boot 3.2+ yêu cầu tối thiểu Java 17, tối ưu cho Java 21

### 1.4. Build Tool: Maven vs Gradle

| Tiêu chí | Maven | Gradle |
|-----------|-------|--------|
| **Cấu hình** | XML (`pom.xml`) | Groovy/Kotlin (`build.gradle`) |
| **Học** | Dễ hơn, cấu trúc cố định | Phức tạp hơn, linh hoạt hơn |
| **Tốc độ build** | Chậm hơn | Nhanh hơn (incremental build) |
| **Phổ biến** | Enterprise truyền thống | Android, modern projects |
| **Plugin** | Rất nhiều | Rất nhiều |
| **Spring Boot** | Hỗ trợ tốt | Hỗ trợ tốt |

**Khóa học này dùng Maven** vì:
- Cấu trúc rõ ràng, dễ đọc
- Phổ biến trong enterprise Java
- Spring Initializr mặc định tạo Maven project
- Tài liệu Spring hầu hết dùng Maven

---

## 2. SPRING FRAMEWORK & SPRING BOOT

### 2.1. Spring Framework là gì?

Spring Framework (ra đời 2003) là **bộ khung phát triển toàn diện** cho Java, giải quyết các vấn đề:

```
Vấn đề Java EE truyền thống          Spring Framework giải quyết
──────────────────────────────        ──────────────────────────────
Boilerplate code quá nhiều      ──>   Convention over Configuration
Tight coupling giữa classes    ──>   Dependency Injection (IoC)
Khó test (new Object() khắp nơi) ─>  Bean management, Mockable
Transaction management phức tạp ──>   Declarative Transaction (@Transactional)
JDBC boilerplate (try/catch/close) ─> Spring JDBC Template, JPA
Security phức tạp              ──>   Spring Security
```

### 2.2. Spring Ecosystem - Bức tranh toàn cảnh

```
┌─────────────────────────────────────────────────────────┐
│                    SPRING ECOSYSTEM                      │
│                                                          │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  │
│  │ Spring Boot  │  │ Spring Cloud │  │ Spring Data  │  │
│  │              │  │              │  │              │  │
│  │ Auto-config  │  │ Microservice │  │ JPA, MongoDB │  │
│  │ Embedded     │  │ Config       │  │ Redis, ES    │  │
│  │ Server       │  │ Discovery    │  │ JDBC         │  │
│  │ Starters     │  │ Gateway      │  │              │  │
│  └──────┬───────┘  └──────────────┘  └──────────────┘  │
│         │                                                │
│  ┌──────┴───────┐  ┌──────────────┐  ┌──────────────┐  │
│  │   Spring     │  │   Spring     │  │   Spring     │  │
│  │   Framework  │  │   Security   │  │   Batch      │  │
│  │              │  │              │  │              │  │
│  │  IoC/DI      │  │  Auth/Authz  │  │  Job         │  │
│  │  AOP         │  │  JWT/OAuth2  │  │  Scheduling  │  │
│  │  MVC         │  │  CORS/CSRF   │  │  Processing  │  │
│  │  WebFlux     │  │  Filters     │  │              │  │
│  └──────────────┘  └──────────────┘  └──────────────┘  │
│                                                          │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  │
│  │   Spring     │  │   Spring     │  │   Spring     │  │
│  │   WebFlux    │  │   AMQP       │  │   Session    │  │
│  │   (Reactive) │  │   (Kafka,    │  │   (Redis,    │  │
│  │              │  │    RabbitMQ)  │  │    JDBC)     │  │
│  └──────────────┘  └──────────────┘  └──────────────┘  │
└─────────────────────────────────────────────────────────┘
```

### 2.3. Spring Boot - Cái gì và tại sao?

**Spring Boot = Spring Framework + Auto Configuration + Embedded Server + Opinionated Defaults**

**Ví dụ so sánh** - Kết nối database:

**Không có Spring Boot** (Spring truyền thống):
```java
// 1. Phải tạo file applicationContext.xml
// 2. Khai báo DataSource bean
// 3. Khai báo EntityManagerFactory
// 4. Khai báo TransactionManager
// 5. Khai báo JpaVendorAdapter
// 6. Scan packages
// ... 100+ dòng XML config
```

**Với Spring Boot**:
```yaml
# application.yml - chỉ 4 dòng
spring:
  datasource:
    url: jdbc:mysql://localhost:3306/ecommerce
    username: root
    password: 1q2w3E*
```

Spring Boot **tự động cấu hình** (Auto-Configuration) dựa trên:
1. Dependencies có trong classpath (pom.xml)
2. Properties trong application.yml
3. Bean đã được định nghĩa

### 2.4. Spring Boot Starters - Gói dependency tiện lợi

Thay vì thêm từng dependency một, Spring Boot gom lại thành **Starter**:

| Starter | Bao gồm |
|---------|---------|
| `spring-boot-starter-web` | Spring MVC + Tomcat + Jackson JSON |
| `spring-boot-starter-data-jpa` | Hibernate + Spring Data JPA + HikariCP |
| `spring-boot-starter-security` | Spring Security + Auto-config |
| `spring-boot-starter-validation` | Jakarta Bean Validation + Hibernate Validator |
| `spring-boot-starter-mail` | Jakarta Mail + Spring Mail |
| `spring-boot-starter-data-redis` | Lettuce Redis client + Spring Data Redis |
| `spring-boot-starter-test` | JUnit 5 + Mockito + AssertJ + MockMvc |
| `spring-boot-starter-actuator` | Health check + Metrics + Info endpoints |

### 2.5. Kiến trúc Spring Boot Application

```
                    HTTP Request
                         │
                         ▼
              ┌─────────────────────┐
              │   Embedded Tomcat    │  ← Spring Boot tự khởi tạo
              │   (Port 8080)        │
              └──────────┬──────────┘
                         │
              ┌──────────▼──────────┐
              │  DispatcherServlet   │  ← Front Controller pattern
              └──────────┬──────────┘
                         │
              ┌──────────▼──────────┐
              │  Security Filter     │  ← Spring Security chain
              │  Chain               │
              └──────────┬──────────┘
                         │
              ┌──────────▼──────────┐
              │  @RestController      │  ← Nhận request, trả response
              │  (Controller Layer)   │
              └──────────┬──────────┘
                         │
              ┌──────────▼──────────┐
              │  @Service             │  ← Business logic
              │  (Service Layer)      │
              └──────────┬──────────┘
                         │
              ┌──────────▼──────────┐
              │  @Repository          │  ← Data access
              │  (Repository Layer)   │
              └──────────┬──────────┘
                         │
              ┌──────────▼──────────┐
              │  JPA / Hibernate      │  ← ORM mapping
              └──────────┬──────────┘
                         │
              ┌──────────▼──────────┐
              │  MySQL Database       │
              └─────────────────────┘
```

---

## 3. CÀI ĐẶT MÔI TRƯỜNG

### 3.1. Cài JDK 21

**Windows:**
```bash
# Cách 1: Dùng winget (Windows Package Manager)
winget install EclipseAdoptium.Temurin.21.JDK

# Cách 2: Tải từ https://adoptium.net/temurin/releases/
# Chọn: JDK 21 - Windows x64 - .msi installer

# Kiểm tra sau khi cài
java -version
# Kết quả mong đợi:
# openjdk version "21.0.x" 2024-xx-xx LTS
# OpenJDK Runtime Environment Temurin-21.0.x+xx (build 21.0.x+xx-LTS)

javac -version
# javac 21.0.x
```

**macOS:**
```bash
# Dùng Homebrew
brew install --cask temurin@21

# Hoặc dùng SDKMAN (khuyến khích)
curl -s "https://get.sdkman.io" | bash
sdk install java 21.0.2-tem
```

**Linux (Ubuntu/Debian):**
```bash
sudo apt update
sudo apt install -y temurin-21-jdk

# Hoặc dùng SDKMAN
sdk install java 21.0.2-tem
```

### 3.2. Thiết lập JAVA_HOME

**Windows** (System Environment Variables):
```
JAVA_HOME = C:\Program Files\Eclipse Adoptium\jdk-21.0.x
Path += %JAVA_HOME%\bin
```

**macOS/Linux** (thêm vào `~/.bashrc` hoặc `~/.zshrc`):
```bash
export JAVA_HOME=$(/usr/libexec/java_home -v 21)
export PATH=$JAVA_HOME/bin:$PATH
```

Kiểm tra:
```bash
echo $JAVA_HOME
java -version
javac -version
```

### 3.3. Cài Maven

**Windows:**
```bash
winget install Apache.Maven

# Hoặc tải từ https://maven.apache.org/download.cgi
# Giải nén, thêm bin/ vào Path
```

**macOS:**
```bash
brew install maven
```

**Kiểm tra:**
```bash
mvn -version
# Apache Maven 3.9.x
# Java version: 21.0.x
```

> **Lưu ý**: Spring Boot project có sẵn Maven Wrapper (`./mvnw`), nên không bắt buộc cài Maven global. Nhưng cài sẵn tiện hơn.

### 3.4. Cài IntelliJ IDEA

Tải tại: https://www.jetbrains.com/idea/download/

| Phiên bản | Giá | Tính năng |
|-----------|-----|-----------|
| **Community** (Free) | 0đ | Java, Maven, Git, đủ cho khóa này |
| **Ultimate** (Paid) | ~$150/năm | Spring support, Database tool, HTTP client |

**Plugin nên cài thêm (Community):**
- **Lombok** - giảm boilerplate code
- **MapStruct Support** - code generation cho DTO mapping
- **Spring Boot Assistant** - hỗ trợ Spring Boot

### 3.5. Cài MySQL + Redis (Docker)

```bash
# Tạo file docker-compose.yml cho infrastructure
cat > docker-compose-infra.yml << 'EOF'
version: '3.8'
services:
  mysql:
    image: mysql:8.0
    container_name: ecommerce-mysql
    environment:
      MYSQL_ROOT_PASSWORD: 1q2w3E*
      MYSQL_DATABASE: ecommerce
    ports:
      - "3306:3306"
    volumes:
      - mysql_data:/var/lib/mysql

  redis:
    image: redis:7-alpine
    container_name: ecommerce-redis
    ports:
      - "6379:6379"
    volumes:
      - redis_data:/data

volumes:
  mysql_data:
  redis_data:
EOF

# Khởi động
docker compose -f docker-compose-infra.yml up -d

# Kiểm tra
docker ps
# Phải thấy 2 container: ecommerce-mysql, ecommerce-redis

# Test kết nối MySQL
docker exec -it ecommerce-mysql mysql -uroot -p1q2w3E* -e "SELECT VERSION();"
# 8.0.xx

# Test kết nối Redis
docker exec -it ecommerce-redis redis-cli ping
# PONG
```

---

## 4. TẠO PROJECT SPRING BOOT ĐẦU TIÊN

### 4.1. Dùng Spring Initializr

Truy cập: https://start.spring.io

Cấu hình:
```
Project:      Maven
Language:     Java
Spring Boot:  3.2.x (hoặc mới nhất stable)
Group:        com.example
Artifact:     ecommerce
Name:         ecommerce
Description:  E-Commerce API with Spring Boot
Package name: com.example.ecommerce
Packaging:    Jar
Java:         21

Dependencies (thêm ngay):
  ✅ Spring Web
  ✅ Spring Data JPA
  ✅ MySQL Driver
  ✅ Spring Boot DevTools
  ✅ Lombok
  ✅ Validation
```

Nhấn **Generate** → tải file zip → giải nén.

### 4.2. Hoặc tạo bằng command line

```bash
# Dùng Spring CLI (nếu đã cài)
spring init \
  --type=maven-project \
  --language=java \
  --boot-version=3.2.5 \
  --java-version=21 \
  --group-id=com.example \
  --artifact-id=ecommerce \
  --name=ecommerce \
  --description="E-Commerce API" \
  --dependencies=web,data-jpa,mysql,devtools,lombok,validation \
  ecommerce.zip

unzip ecommerce.zip -d ecommerce
cd ecommerce
```

### 4.3. Cấu trúc project sau khi tạo

```
ecommerce/
├── pom.xml                              # Maven config (quan trọng nhất)
├── mvnw                                 # Maven wrapper (Unix)
├── mvnw.cmd                             # Maven wrapper (Windows)
├── .mvn/wrapper/
│   └── maven-wrapper.properties
├── src/
│   ├── main/
│   │   ├── java/com/example/ecommerce/
│   │   │   └── EcommerceApplication.java   # Entry point
│   │   └── resources/
│   │       ├── application.properties       # Config (đổi sang .yml)
│   │       ├── static/                      # Static files
│   │       └── templates/                   # Template files
│   └── test/
│       └── java/com/example/ecommerce/
│           └── EcommerceApplicationTests.java
└── .gitignore
```

### 4.4. Phân tích pom.xml

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0
         https://maven.apache.org/xsd/maven-4.0.0.xsd">

    <modelVersion>4.0.0</modelVersion>

    <!-- Spring Boot Parent: quản lý version tất cả dependencies -->
    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>3.2.5</version>
        <relativePath/>
    </parent>

    <!-- Thông tin project -->
    <groupId>com.example</groupId>
    <artifactId>ecommerce</artifactId>
    <version>0.0.1-SNAPSHOT</version>
    <name>ecommerce</name>
    <description>E-Commerce API with Spring Boot</description>

    <!-- Java version -->
    <properties>
        <java.version>21</java.version>
    </properties>

    <dependencies>
        <!-- Spring Web: REST API + Embedded Tomcat -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
            <!-- Không cần khai version - parent quản lý -->
        </dependency>

        <!-- Spring Data JPA: Hibernate + JPA -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-data-jpa</artifactId>
        </dependency>

        <!-- Bean Validation: @NotBlank, @Email, @Size... -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-validation</artifactId>
        </dependency>

        <!-- MySQL JDBC Driver -->
        <dependency>
            <groupId>com.mysql</groupId>
            <artifactId>mysql-connector-j</artifactId>
            <scope>runtime</scope> <!-- Chỉ cần lúc chạy -->
        </dependency>

        <!-- Lombok: giảm boilerplate (@Getter, @Setter, @Builder) -->
        <dependency>
            <groupId>org.projectlombok</groupId>
            <artifactId>lombok</artifactId>
            <optional>true</optional>
        </dependency>

        <!-- DevTools: auto-restart khi code thay đổi -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-devtools</artifactId>
            <scope>runtime</scope>
            <optional>true</optional>
        </dependency>

        <!-- Test: JUnit 5 + Mockito + MockMvc -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>
    </dependencies>

    <build>
        <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
                <configuration>
                    <!-- Exclude Lombok from final JAR -->
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
</project>
```

### 4.5. Phân tích Entry Point

```java
package com.example.ecommerce;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication  // = @Configuration + @EnableAutoConfiguration + @ComponentScan
public class EcommerceApplication {

    public static void main(String[] args) {
        // Khởi tạo Spring ApplicationContext
        // Scan tất cả @Component, @Service, @Repository, @Controller
        // Auto-configure dựa trên classpath
        // Khởi động Embedded Tomcat trên port 8080
        SpringApplication.run(EcommerceApplication.class, args);
    }
}
```

**`@SpringBootApplication`** là tổ hợp của 3 annotation:

| Annotation | Tác dụng |
|------------|----------|
| `@Configuration` | Đánh dấu class là nguồn cấu hình Bean |
| `@EnableAutoConfiguration` | Bật auto-config dựa trên dependencies |
| `@ComponentScan` | Scan package hiện tại + sub-packages tìm @Component |

### 4.6. Cấu hình application.yml

Đổi `application.properties` thành `application.yml` (dễ đọc hơn):

```yaml
# src/main/resources/application.yml
server:
  port: 8080

spring:
  application:
    name: ecommerce-api

  # Database
  datasource:
    url: jdbc:mysql://localhost:3306/ecommerce?useSSL=false&serverTimezone=UTC&allowPublicKeyRetrieval=true
    username: root
    password: 1q2w3E*
    driver-class-name: com.mysql.cj.jdbc.Driver

  # JPA / Hibernate
  jpa:
    hibernate:
      ddl-auto: update    # Tự tạo/update table (chỉ dùng cho dev)
    show-sql: true         # Log SQL queries
    properties:
      hibernate:
        format_sql: true   # Format SQL cho dễ đọc
        dialect: org.hibernate.dialect.MySQLDialect

  # DevTools
  devtools:
    restart:
      enabled: true        # Auto-restart khi code thay đổi
    livereload:
      enabled: true

# Logging
logging:
  level:
    root: INFO
    com.example.ecommerce: DEBUG
    org.hibernate.SQL: DEBUG
    org.hibernate.type.descriptor.sql.BasicBinder: TRACE
```

**Giải thích `ddl-auto`:**

| Giá trị | Tác dụng | Dùng khi |
|---------|----------|----------|
| `none` | Không làm gì với schema | Production |
| `validate` | Chỉ kiểm tra schema khớp Entity | Staging |
| `update` | Tự tạo/update table, KHÔNG xóa column | Development |
| `create` | Xóa sạch + tạo lại mỗi lần start | Testing |
| `create-drop` | Như `create` + xóa khi shutdown | Unit test |

---

## 5. VIẾT API ĐẦU TIÊN

### 5.1. Hello World Controller

Tạo file `src/main/java/com/example/ecommerce/controller/HelloController.java`:

```java
package com.example.ecommerce.controller;

import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RequestParam;
import org.springframework.web.bind.annotation.RestController;

import java.time.LocalDateTime;
import java.util.Map;

@RestController  // = @Controller + @ResponseBody (trả JSON tự động)
public class HelloController {

    // GET http://localhost:8080/hello
    @GetMapping("/hello")
    public String hello() {
        return "Xin chào từ Spring Boot E-Commerce API!";
    }

    // GET http://localhost:8080/hello/json
    @GetMapping("/hello/json")
    public Map<String, Object> helloJson() {
        return Map.of(
            "message", "Xin chào từ Spring Boot!",
            "timestamp", LocalDateTime.now().toString(),
            "version", "1.0.0",
            "java", System.getProperty("java.version")
        );
    }

    // GET http://localhost:8080/hello/greet?name=Hùng
    @GetMapping("/hello/greet")
    public Map<String, String> greet(
            @RequestParam(defaultValue = "Bạn") String name) {
        return Map.of(
            "greeting", "Xin chào, " + name + "! Chào mừng đến với E-Commerce API."
        );
    }
}
```

### 5.2. Chạy ứng dụng

```bash
# Cách 1: Maven wrapper
./mvnw spring-boot:run

# Cách 2: Maven global
mvn spring-boot:run

# Cách 3: IntelliJ IDEA
# Click nút ▶ (Run) bên cạnh main() method
```

**Output mong đợi:**
```
  .   ____          _            __ _ _
 /\\ / ___'_ __ _ _(_)_ __  __ _ \ \ \ \
( ( )\___ | '_ | '_| | '_ \/ _` | \ \ \ \
 \\/  ___)| |_)| | | | | || (_| |  ) ) ) )
  '  |____| .__|_| |_|_| |_\__, | / / / /
 =========|_|==============|___/=/_/_/_/

 :: Spring Boot ::                (v3.2.5)

2024-xx-xx INFO  --- Started EcommerceApplication in 3.456 seconds
2024-xx-xx INFO  --- Tomcat initialized with port 8080 (http)
2024-xx-xx INFO  --- Hibernate: create table ...
```

### 5.3. Test các endpoint

```bash
# Test 1: Hello text
curl http://localhost:8080/hello
# Output: Xin chào từ Spring Boot E-Commerce API!

# Test 2: Hello JSON
curl http://localhost:8080/hello/json
# Output: {"message":"Xin chào từ Spring Boot!","timestamp":"2024-...","version":"1.0.0","java":"21.0.2"}

# Test 3: Hello với tham số
curl "http://localhost:8080/hello/greet?name=Hùng"
# Output: {"greeting":"Xin chào, Hùng! Chào mừng đến với E-Commerce API."}

# Test 4: Actuator health (nếu thêm starter-actuator)
curl http://localhost:8080/actuator/health
# Output: {"status":"UP"}
```

---

## 6. SPRING BOOT AUTO-CONFIGURATION - CÁCH NÓ HOẠT ĐỘNG

### 6.1. Quy trình Auto-Configuration

```
Application Start
      │
      ▼
@SpringBootApplication
      │
      ├── @ComponentScan
      │   └── Scan com.example.ecommerce.**
      │       tìm @Component, @Service, @Repository, @Controller
      │
      ├── @EnableAutoConfiguration
      │   └── Đọc META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports
      │       │
      │       ├── DataSourceAutoConfiguration
      │       │   └── Thấy mysql-connector-j trong classpath
      │       │       └── Tự tạo DataSource bean từ spring.datasource.*
      │       │
      │       ├── JpaRepositoriesAutoConfiguration
      │       │   └── Thấy spring-data-jpa trong classpath
      │       │       └── Tự tạo EntityManagerFactory, TransactionManager
      │       │
      │       ├── WebMvcAutoConfiguration
      │       │   └── Thấy spring-webmvc trong classpath
      │       │       └── Tự cấu hình DispatcherServlet, Jackson, etc.
      │       │
      │       └── ... (100+ auto-configurations)
      │
      └── @Configuration
          └── Cho phép khai báo thêm @Bean methods
```

### 6.2. Xem Auto-Configuration đang active

Thêm vào `application.yml`:
```yaml
debug: true  # Hiện report chi tiết khi start
```

Hoặc chạy:
```bash
./mvnw spring-boot:run --debug
```

Output sẽ hiện:
```
============================
CONDITIONS EVALUATION REPORT
============================

Positive matches:  (những config được kích hoạt)
-----------------
   DataSourceAutoConfiguration matched
   JpaRepositoriesAutoConfiguration matched
   WebMvcAutoConfiguration matched

Negative matches:  (những config KHÔNG kích hoạt)
-----------------
   RedisAutoConfiguration: @ConditionalOnClass did not find required class
   SecurityAutoConfiguration: @ConditionalOnClass did not find required class
```

---

## 7. JAVA 21 - TÍNH NĂNG MỚI SỬ DỤNG TRONG KHÓA HỌC

### 7.1. Records (Java 16+) - Thay thế DTO class

```java
// Trước đây: phải viết cả trang code
public class ProductResponse {
    private Long id;
    private String name;
    private BigDecimal price;

    // Constructor, getters, setters, equals, hashCode, toString
    // ... 50+ dòng boilerplate
}

// Java 16+: 1 dòng
public record ProductResponse(Long id, String name, BigDecimal price) {}

// Tự động có: constructor, getters, equals, hashCode, toString
// Immutable (không thể thay đổi sau khi tạo)
```

### 7.2. Pattern Matching (Java 21)

```java
// Trước đây
if (obj instanceof String) {
    String s = (String) obj;
    System.out.println(s.length());
}

// Java 16+: Pattern Matching for instanceof
if (obj instanceof String s) {
    System.out.println(s.length());  // s đã được cast tự động
}

// Java 21: Pattern Matching for switch
String describe(Object obj) {
    return switch (obj) {
        case Integer i when i > 0 -> "Số dương: " + i;
        case Integer i            -> "Số không dương: " + i;
        case String s             -> "Chuỗi: " + s;
        case null                 -> "Null";
        default                   -> "Kiểu khác: " + obj.getClass();
    };
}
```

### 7.3. Sealed Classes (Java 17+)

```java
// Giới hạn ai có thể kế thừa
public sealed interface PaymentMethod
    permits CreditCard, BankTransfer, EWallet {
}

public record CreditCard(String number, String cvv) implements PaymentMethod {}
public record BankTransfer(String bankCode, String accountNo) implements PaymentMethod {}
public record EWallet(String provider, String phone) implements PaymentMethod {}

// Compiler đảm bảo switch xử lý hết tất cả case
String process(PaymentMethod method) {
    return switch (method) {
        case CreditCard cc    -> "Thẻ tín dụng: " + cc.number();
        case BankTransfer bt  -> "Chuyển khoản: " + bt.bankCode();
        case EWallet ew       -> "Ví điện tử: " + ew.provider();
        // Không cần default - compiler biết đã xử lý hết
    };
}
```

### 7.4. Virtual Threads (Java 21) - Game changer

```java
// Trước đây: 1 platform thread = 1 OS thread (~1MB stack)
// 10,000 request = 10,000 OS threads = ~10GB RAM

// Java 21: Virtual Thread = ~1KB, có thể tạo hàng triệu
// Cấu hình Spring Boot dùng Virtual Threads:
spring:
  threads:
    virtual:
      enabled: true  # Chỉ 1 dòng!

// Kết quả: Spring Boot dùng Virtual Thread cho mỗi request
// 1,000,000 concurrent requests mà RAM vẫn thấp
```

---

## 8. BÀI TẬP & CHECKLIST

### Bài tập thực hành

- [ ] Cài đặt JDK 21, kiểm tra `java -version`
- [ ] Cài đặt Maven, kiểm tra `mvn -version`
- [ ] Cài IntelliJ IDEA Community
- [ ] Chạy MySQL + Redis bằng Docker Compose
- [ ] Tạo project Spring Boot từ start.spring.io
- [ ] Đổi `application.properties` → `application.yml`
- [ ] Cấu hình kết nối MySQL
- [ ] Viết HelloController với 3 endpoint
- [ ] Chạy ứng dụng, test bằng curl hoặc browser
- [ ] Bật `debug: true` và đọc Auto-Configuration report

### Kiến thức cần nắm

| Khái niệm | Hiểu chưa? |
|-----------|-------------|
| JVM vs JRE vs JDK | ☐ |
| Tại sao chọn Java 21 LTS | ☐ |
| Maven vs Gradle | ☐ |
| Spring Framework vs Spring Boot | ☐ |
| @SpringBootApplication bao gồm gì | ☐ |
| Auto-Configuration hoạt động thế nào | ☐ |
| application.yml cấu hình cơ bản | ☐ |
| @RestController vs @Controller | ☐ |
| Spring Boot Starters là gì | ☐ |
| Records, Pattern Matching, Virtual Threads | ☐ |

### Lỗi thường gặp

| Lỗi | Nguyên nhân | Cách sửa |
|-----|-------------|----------|
| `JAVA_HOME not set` | Chưa set biến môi trường | Set JAVA_HOME trỏ đến JDK 21 |
| `Port 8080 already in use` | Port đang bị chiếm | Đổi `server.port` hoặc kill process |
| `Access denied for user 'root'` | Sai password MySQL | Kiểm tra password trong docker-compose |
| `Unknown database 'ecommerce'` | Chưa tạo database | Docker-compose đã tạo tự động, kiểm tra container |
| `ClassNotFoundException: com.mysql.cj.jdbc.Driver` | Thiếu MySQL driver | Kiểm tra dependency trong pom.xml |

---

## Tài liệu tham khảo

- [Spring Boot Official Docs](https://docs.spring.io/spring-boot/docs/current/reference/html/)
- [Spring Initializr](https://start.spring.io)
- [JDK 21 Release Notes](https://openjdk.org/projects/jdk/21/)
- [Maven Getting Started](https://maven.apache.org/guides/getting-started/)
- [Baeldung Spring Boot](https://www.baeldung.com/spring-boot)

---

**Tiếp theo:** [Ngày 02 - Khởi tạo Project & Clean Architecture →](day-02-project-setup.md)
