# Day 16: Spring Security Fundamentals

## Mục tiêu học tập
- Hiểu kiến trúc Spring Security
- Cấu hình Security Filter Chain
- Implement Password Encoding
- Tạo Custom UserDetailsService

---

## 1. Spring Security Overview

### 1.1 Tại sao cần Spring Security?

```
┌─────────────────────────────────────────────────────────────────┐
│                    SECURITY CONCERNS                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐          │
│  │Authentication│  │Authorization │  │   Attack     │          │
│  │              │  │              │  │  Prevention  │          │
│  ├──────────────┤  ├──────────────┤  ├──────────────┤          │
│  │ Who are you? │  │ What can you │  │ CSRF, XSS,   │          │
│  │              │  │    do?       │  │ SQL Injection│          │
│  └──────────────┘  └──────────────┘  └──────────────┘          │
│                                                                  │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐          │
│  │   Session    │  │   Password   │  │    Audit     │          │
│  │  Management  │  │   Security   │  │   Logging    │          │
│  ├──────────────┤  ├──────────────┤  ├──────────────┤          │
│  │ Token, Cookie│  │ Hashing,Salt │  │ Who did what │          │
│  └──────────────┘  └──────────────┘  └──────────────┘          │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 1.2 Spring Security Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                     HTTP REQUEST                                 │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                   SECURITY FILTER CHAIN                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │ 1. SecurityContextPersistenceFilter                      │   │
│  │    - Load/Save SecurityContext từ Session               │   │
│  └─────────────────────────────────────────────────────────┘   │
│                              │                                   │
│                              ▼                                   │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │ 2. CsrfFilter                                            │   │
│  │    - Validate CSRF token                                 │   │
│  └─────────────────────────────────────────────────────────┘   │
│                              │                                   │
│                              ▼                                   │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │ 3. UsernamePasswordAuthenticationFilter                  │   │
│  │    - Process login form submission                       │   │
│  └─────────────────────────────────────────────────────────┘   │
│                              │                                   │
│                              ▼                                   │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │ 4. BasicAuthenticationFilter                             │   │
│  │    - Process HTTP Basic Auth header                      │   │
│  └─────────────────────────────────────────────────────────┘   │
│                              │                                   │
│                              ▼                                   │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │ 5. ExceptionTranslationFilter                            │   │
│  │    - Handle security exceptions                          │   │
│  └─────────────────────────────────────────────────────────┘   │
│                              │                                   │
│                              ▼                                   │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │ 6. FilterSecurityInterceptor                             │   │
│  │    - Authorization check (access decision)               │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                      CONTROLLER                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## 2. Thêm Dependencies

### 2.1 pom.xml

```xml
<dependencies>
    <!-- Spring Security -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-security</artifactId>
    </dependency>

    <!-- Security Test -->
    <dependency>
        <groupId>org.springframework.security</groupId>
        <artifactId>spring-security-test</artifactId>
        <scope>test</scope>
    </dependency>
</dependencies>
```

> **Lưu ý**: Khi thêm spring-boot-starter-security, tất cả endpoints sẽ được bảo vệ mặc định!

---

## 3. Security Configuration

### 3.1 SecurityConfig cơ bản

```java
package com.ecommerce.config;

import lombok.RequiredArgsConstructor;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.http.HttpMethod;
import org.springframework.security.authentication.AuthenticationManager;
import org.springframework.security.authentication.AuthenticationProvider;
import org.springframework.security.authentication.dao.DaoAuthenticationProvider;
import org.springframework.security.config.annotation.authentication.configuration.AuthenticationConfiguration;
import org.springframework.security.config.annotation.method.configuration.EnableMethodSecurity;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.config.annotation.web.configuration.EnableWebSecurity;
import org.springframework.security.config.annotation.web.configurers.AbstractHttpConfigurer;
import org.springframework.security.config.http.SessionCreationPolicy;
import org.springframework.security.core.userdetails.UserDetailsService;
import org.springframework.security.crypto.bcrypt.BCryptPasswordEncoder;
import org.springframework.security.crypto.password.PasswordEncoder;
import org.springframework.security.web.SecurityFilterChain;

@Configuration
@EnableWebSecurity
@EnableMethodSecurity  // Cho phép @PreAuthorize, @PostAuthorize
@RequiredArgsConstructor
public class SecurityConfig {

    private final UserDetailsService userDetailsService;

    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
        http
            // Disable CSRF cho REST API (sẽ dùng JWT)
            .csrf(AbstractHttpConfigurer::disable)

            // Session management - STATELESS cho JWT
            .sessionManagement(session ->
                session.sessionCreationPolicy(SessionCreationPolicy.STATELESS)
            )

            // Authorization rules
            .authorizeHttpRequests(auth -> auth
                // Public endpoints
                .requestMatchers("/api/v1/auth/**").permitAll()
                .requestMatchers("/api/v1/products/**").permitAll()
                .requestMatchers("/api/v1/categories/**").permitAll()

                // Swagger endpoints
                .requestMatchers(
                    "/swagger-ui/**",
                    "/v3/api-docs/**",
                    "/swagger-resources/**"
                ).permitAll()

                // Actuator health check
                .requestMatchers("/actuator/health").permitAll()

                // Admin only
                .requestMatchers("/api/v1/admin/**").hasRole("ADMIN")

                // Authenticated users
                .requestMatchers("/api/v1/users/me").authenticated()
                .requestMatchers("/api/v1/orders/**").authenticated()
                .requestMatchers("/api/v1/cart/**").authenticated()

                // Specific HTTP methods
                .requestMatchers(HttpMethod.POST, "/api/v1/reviews/**").authenticated()
                .requestMatchers(HttpMethod.DELETE, "/api/v1/reviews/**").hasRole("ADMIN")

                // All other requests require authentication
                .anyRequest().authenticated()
            )

            // Authentication provider
            .authenticationProvider(authenticationProvider());

        return http.build();
    }

    @Bean
    public AuthenticationProvider authenticationProvider() {
        DaoAuthenticationProvider authProvider = new DaoAuthenticationProvider();
        authProvider.setUserDetailsService(userDetailsService);
        authProvider.setPasswordEncoder(passwordEncoder());
        return authProvider;
    }

    @Bean
    public AuthenticationManager authenticationManager(
            AuthenticationConfiguration config) throws Exception {
        return config.getAuthenticationManager();
    }

    @Bean
    public PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder(12); // Strength 12
    }
}
```

### 3.2 Authorization Flow

```
┌─────────────────────────────────────────────────────────────────┐
│                  REQUEST: POST /api/v1/orders                    │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│              Check: .requestMatchers("/api/v1/orders/**")        │
│                     .authenticated()                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   ┌─────────────────┐          ┌─────────────────┐             │
│   │  Has valid JWT? │──YES───▶│   ALLOW ACCESS   │             │
│   └─────────────────┘          └─────────────────┘             │
│           │                                                      │
│          NO                                                      │
│           │                                                      │
│           ▼                                                      │
│   ┌─────────────────┐                                           │
│   │  401 Unauthorized│                                           │
│   └─────────────────┘                                           │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## 4. User Entity & UserDetails

### 4.1 User Entity với Security

```java
package com.ecommerce.domain.entity;

import com.ecommerce.domain.enums.Role;
import jakarta.persistence.*;
import lombok.*;
import org.springframework.security.core.GrantedAuthority;
import org.springframework.security.core.authority.SimpleGrantedAuthority;
import org.springframework.security.core.userdetails.UserDetails;

import java.time.LocalDateTime;
import java.util.Collection;
import java.util.List;

@Entity
@Table(name = "users")
@Getter
@Setter
@NoArgsConstructor
@AllArgsConstructor
@Builder
public class User extends BaseEntity implements UserDetails {

    @Column(nullable = false, unique = true, length = 100)
    private String email;

    @Column(nullable = false)
    private String password;

    @Column(name = "first_name", length = 50)
    private String firstName;

    @Column(name = "last_name", length = 50)
    private String lastName;

    @Column(length = 20)
    private String phone;

    @Enumerated(EnumType.STRING)
    @Column(nullable = false)
    private Role role = Role.CUSTOMER;

    @Column(name = "email_verified")
    private boolean emailVerified = false;

    @Column(name = "account_locked")
    private boolean accountLocked = false;

    @Column(name = "failed_login_attempts")
    private int failedLoginAttempts = 0;

    @Column(name = "lock_time")
    private LocalDateTime lockTime;

    @Column(name = "last_login")
    private LocalDateTime lastLogin;

    // ========== UserDetails Implementation ==========

    @Override
    public Collection<? extends GrantedAuthority> getAuthorities() {
        // Prefix "ROLE_" cho hasRole() check
        return List.of(new SimpleGrantedAuthority("ROLE_" + role.name()));
    }

    @Override
    public String getUsername() {
        return email; // Dùng email làm username
    }

    @Override
    public boolean isAccountNonExpired() {
        return true; // Account không hết hạn
    }

    @Override
    public boolean isAccountNonLocked() {
        // Tự động unlock sau 30 phút
        if (accountLocked && lockTime != null) {
            if (LocalDateTime.now().isAfter(lockTime.plusMinutes(30))) {
                return true;
            }
        }
        return !accountLocked;
    }

    @Override
    public boolean isCredentialsNonExpired() {
        return true; // Password không hết hạn
    }

    @Override
    public boolean isEnabled() {
        return emailVerified; // Phải verify email mới enabled
    }

    // ========== Helper Methods ==========

    public String getFullName() {
        return firstName + " " + lastName;
    }

    public void incrementFailedAttempts() {
        this.failedLoginAttempts++;
        if (this.failedLoginAttempts >= 5) {
            this.accountLocked = true;
            this.lockTime = LocalDateTime.now();
        }
    }

    public void resetFailedAttempts() {
        this.failedLoginAttempts = 0;
        this.accountLocked = false;
        this.lockTime = null;
    }

    public void updateLastLogin() {
        this.lastLogin = LocalDateTime.now();
    }
}
```

### 4.2 Role Enum

```java
package com.ecommerce.domain.enums;

public enum Role {
    CUSTOMER,    // Khách hàng thông thường
    SELLER,      // Người bán
    ADMIN,       // Quản trị viên
    SUPER_ADMIN  // Super admin
}
```

---

## 5. Custom UserDetailsService

### 5.1 Implementation

```java
package com.ecommerce.service.impl;

import com.ecommerce.domain.entity.User;
import com.ecommerce.exception.ResourceNotFoundException;
import com.ecommerce.repository.UserRepository;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.security.core.userdetails.UserDetails;
import org.springframework.security.core.userdetails.UserDetailsService;
import org.springframework.security.core.userdetails.UsernameNotFoundException;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

@Service
@RequiredArgsConstructor
@Slf4j
public class CustomUserDetailsService implements UserDetailsService {

    private final UserRepository userRepository;

    @Override
    @Transactional(readOnly = true)
    public UserDetails loadUserByUsername(String email) throws UsernameNotFoundException {
        log.debug("Loading user by email: {}", email);

        return userRepository.findByEmail(email)
                .orElseThrow(() -> {
                    log.warn("User not found with email: {}", email);
                    return new UsernameNotFoundException(
                        "User not found with email: " + email
                    );
                });
    }

    @Transactional(readOnly = true)
    public User loadUserById(Long id) {
        return userRepository.findById(id)
                .orElseThrow(() -> new ResourceNotFoundException("User", "id", id));
    }
}
```

### 5.2 UserRepository

```java
package com.ecommerce.repository;

import com.ecommerce.domain.entity.User;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.data.jpa.repository.Modifying;
import org.springframework.data.jpa.repository.Query;
import org.springframework.data.repository.query.Param;
import org.springframework.stereotype.Repository;

import java.time.LocalDateTime;
import java.util.Optional;

@Repository
public interface UserRepository extends JpaRepository<User, Long> {

    Optional<User> findByEmail(String email);

    boolean existsByEmail(String email);

    @Query("SELECT u FROM User u WHERE u.email = :email AND u.deleted = false")
    Optional<User> findActiveByEmail(@Param("email") String email);

    @Modifying
    @Query("UPDATE User u SET u.failedLoginAttempts = :attempts WHERE u.id = :id")
    void updateFailedAttempts(@Param("id") Long id, @Param("attempts") int attempts);

    @Modifying
    @Query("UPDATE User u SET u.lastLogin = :time WHERE u.id = :id")
    void updateLastLogin(@Param("id") Long id, @Param("time") LocalDateTime time);

    @Modifying
    @Query("UPDATE User u SET u.accountLocked = false, " +
           "u.failedLoginAttempts = 0, u.lockTime = null WHERE u.id = :id")
    void unlockAccount(@Param("id") Long id);
}
```

---

## 6. Password Encoding

### 6.1 BCrypt Explained

```
┌─────────────────────────────────────────────────────────────────┐
│                    PASSWORD HASHING                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Plain Password: "MyPassword123"                                 │
│                              │                                   │
│                              ▼                                   │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │                    BCrypt Algorithm                       │   │
│  │                                                           │   │
│  │  1. Generate random salt (22 chars)                      │   │
│  │  2. Combine password + salt                              │   │
│  │  3. Hash với cost factor (2^12 iterations)               │   │
│  │  4. Output: $2a$12$salt...hash                           │   │
│  └─────────────────────────────────────────────────────────┘   │
│                              │                                   │
│                              ▼                                   │
│  Hashed: "$2a$12$LQv3c1yqBWVHxkd0LHAkCOYz6TtxMQJqhN8/X4MnlYV"  │
│                                                                  │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  $2a$  │   12   │   LQv3c1yqBWVH   │   xkd0LHAkCOY...   │   │
│  │ Version│  Cost  │      Salt        │       Hash         │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 6.2 Password Service

```java
package com.ecommerce.service.impl;

import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.security.crypto.password.PasswordEncoder;
import org.springframework.stereotype.Service;

@Service
@RequiredArgsConstructor
@Slf4j
public class PasswordService {

    private final PasswordEncoder passwordEncoder;

    /**
     * Hash password với BCrypt
     */
    public String hashPassword(String rawPassword) {
        return passwordEncoder.encode(rawPassword);
    }

    /**
     * Verify password
     */
    public boolean verifyPassword(String rawPassword, String hashedPassword) {
        return passwordEncoder.matches(rawPassword, hashedPassword);
    }

    /**
     * Check password strength
     */
    public PasswordStrength checkStrength(String password) {
        int score = 0;

        if (password.length() >= 8) score++;
        if (password.length() >= 12) score++;
        if (password.matches(".*[A-Z].*")) score++;  // Uppercase
        if (password.matches(".*[a-z].*")) score++;  // Lowercase
        if (password.matches(".*\\d.*")) score++;     // Digit
        if (password.matches(".*[!@#$%^&*(),.?\":{}|<>].*")) score++; // Special

        if (score <= 2) return PasswordStrength.WEAK;
        if (score <= 4) return PasswordStrength.MEDIUM;
        return PasswordStrength.STRONG;
    }

    public enum PasswordStrength {
        WEAK, MEDIUM, STRONG
    }
}
```

---

## 7. Security Utility

### 7.1 SecurityUtils - Lấy thông tin User hiện tại

```java
package com.ecommerce.util;

import com.ecommerce.domain.entity.User;
import org.springframework.security.core.Authentication;
import org.springframework.security.core.context.SecurityContextHolder;
import org.springframework.security.core.userdetails.UserDetails;
import org.springframework.stereotype.Component;

import java.util.Optional;

@Component
public class SecurityUtils {

    /**
     * Lấy Authentication object hiện tại
     */
    public static Optional<Authentication> getCurrentAuthentication() {
        Authentication authentication = SecurityContextHolder
                .getContext()
                .getAuthentication();

        if (authentication == null || !authentication.isAuthenticated()) {
            return Optional.empty();
        }

        return Optional.of(authentication);
    }

    /**
     * Lấy email của user đang đăng nhập
     */
    public static Optional<String> getCurrentUserEmail() {
        return getCurrentAuthentication()
                .map(auth -> {
                    Object principal = auth.getPrincipal();
                    if (principal instanceof UserDetails) {
                        return ((UserDetails) principal).getUsername();
                    }
                    return principal.toString();
                });
    }

    /**
     * Lấy User entity của user đang đăng nhập
     */
    public static Optional<User> getCurrentUser() {
        return getCurrentAuthentication()
                .map(Authentication::getPrincipal)
                .filter(principal -> principal instanceof User)
                .map(principal -> (User) principal);
    }

    /**
     * Lấy User ID của user đang đăng nhập
     */
    public static Optional<Long> getCurrentUserId() {
        return getCurrentUser().map(User::getId);
    }

    /**
     * Kiểm tra user có role cụ thể
     */
    public static boolean hasRole(String role) {
        return getCurrentAuthentication()
                .map(auth -> auth.getAuthorities().stream()
                        .anyMatch(a -> a.getAuthority().equals("ROLE_" + role)))
                .orElse(false);
    }

    /**
     * Kiểm tra user có phải admin
     */
    public static boolean isAdmin() {
        return hasRole("ADMIN") || hasRole("SUPER_ADMIN");
    }

    /**
     * Kiểm tra user có phải chủ sở hữu resource
     */
    public static boolean isOwner(Long ownerId) {
        return getCurrentUserId()
                .map(currentId -> currentId.equals(ownerId))
                .orElse(false);
    }

    /**
     * Kiểm tra có quyền modify resource (owner hoặc admin)
     */
    public static boolean canModify(Long ownerId) {
        return isOwner(ownerId) || isAdmin();
    }
}
```

### 7.2 Sử dụng trong Service

```java
@Service
@RequiredArgsConstructor
public class OrderService {

    private final OrderRepository orderRepository;

    public Order getOrderById(Long orderId) {
        Order order = orderRepository.findById(orderId)
                .orElseThrow(() -> new ResourceNotFoundException("Order", "id", orderId));

        // Chỉ owner hoặc admin mới xem được order
        if (!SecurityUtils.canModify(order.getUser().getId())) {
            throw new AccessDeniedException("You don't have permission to view this order");
        }

        return order;
    }

    public Order createOrder(CreateOrderRequest request) {
        // Lấy current user
        User currentUser = SecurityUtils.getCurrentUser()
                .orElseThrow(() -> new UnauthorizedException("User not authenticated"));

        Order order = Order.builder()
                .user(currentUser)
                .status(OrderStatus.PENDING)
                // ... other fields
                .build();

        return orderRepository.save(order);
    }
}
```

---

## 8. Method-Level Security

### 8.1 @PreAuthorize Annotations

```java
package com.ecommerce.controller;

import com.ecommerce.dto.UserDto;
import com.ecommerce.service.UserService;
import lombok.RequiredArgsConstructor;
import org.springframework.security.access.prepost.PreAuthorize;
import org.springframework.security.access.prepost.PostAuthorize;
import org.springframework.web.bind.annotation.*;

import java.util.List;

@RestController
@RequestMapping("/api/v1/users")
@RequiredArgsConstructor
public class UserController {

    private final UserService userService;

    // Chỉ ADMIN mới access được
    @GetMapping
    @PreAuthorize("hasRole('ADMIN')")
    public List<UserDto> getAllUsers() {
        return userService.getAllUsers();
    }

    // Chỉ ADMIN hoặc chính user đó
    @GetMapping("/{id}")
    @PreAuthorize("hasRole('ADMIN') or #id == authentication.principal.id")
    public UserDto getUserById(@PathVariable Long id) {
        return userService.getUserById(id);
    }

    // Phải authenticated và email verified
    @PostMapping("/profile")
    @PreAuthorize("isAuthenticated() and authentication.principal.emailVerified")
    public UserDto updateProfile(@RequestBody UpdateProfileRequest request) {
        return userService.updateProfile(request);
    }

    // Check quyền SAU khi lấy data
    @GetMapping("/{id}/orders")
    @PostAuthorize("returnObject.userId == authentication.principal.id or hasRole('ADMIN')")
    public OrderListDto getUserOrders(@PathVariable Long id) {
        return userService.getUserOrders(id);
    }

    // Sử dụng SpEL expression phức tạp
    @DeleteMapping("/{id}")
    @PreAuthorize("""
        hasRole('SUPER_ADMIN') or
        (hasRole('ADMIN') and #id != authentication.principal.id)
    """)
    public void deleteUser(@PathVariable Long id) {
        userService.deleteUser(id);
    }
}
```

### 8.2 Custom Security Expressions

```java
package com.ecommerce.security;

import com.ecommerce.domain.entity.User;
import com.ecommerce.repository.OrderRepository;
import lombok.RequiredArgsConstructor;
import org.springframework.security.core.Authentication;
import org.springframework.stereotype.Component;

@Component("securityService")
@RequiredArgsConstructor
public class SecurityExpressionService {

    private final OrderRepository orderRepository;

    /**
     * Check if current user owns the order
     */
    public boolean isOrderOwner(Long orderId, Authentication auth) {
        if (auth == null || !(auth.getPrincipal() instanceof User)) {
            return false;
        }

        User user = (User) auth.getPrincipal();
        return orderRepository.existsByIdAndUserId(orderId, user.getId());
    }

    /**
     * Check if user can access seller dashboard
     */
    public boolean canAccessSellerDashboard(Authentication auth) {
        if (auth == null || !(auth.getPrincipal() instanceof User)) {
            return false;
        }

        User user = (User) auth.getPrincipal();
        return user.getRole().name().equals("SELLER") ||
               user.getRole().name().equals("ADMIN");
    }
}
```

**Sử dụng trong Controller:**

```java
@GetMapping("/orders/{orderId}")
@PreAuthorize("@securityService.isOrderOwner(#orderId, authentication) or hasRole('ADMIN')")
public OrderDto getOrder(@PathVariable Long orderId) {
    return orderService.getOrderById(orderId);
}

@GetMapping("/seller/dashboard")
@PreAuthorize("@securityService.canAccessSellerDashboard(authentication)")
public SellerDashboardDto getSellerDashboard() {
    return sellerService.getDashboard();
}
```

---

## 9. Security Exception Handling

### 9.1 Custom Security Exceptions

```java
package com.ecommerce.exception;

import org.springframework.http.HttpStatus;
import org.springframework.web.bind.annotation.ResponseStatus;

@ResponseStatus(HttpStatus.UNAUTHORIZED)
public class UnauthorizedException extends RuntimeException {
    public UnauthorizedException(String message) {
        super(message);
    }
}

@ResponseStatus(HttpStatus.FORBIDDEN)
public class ForbiddenException extends RuntimeException {
    public ForbiddenException(String message) {
        super(message);
    }
}

public class AccountLockedException extends RuntimeException {
    public AccountLockedException(String message) {
        super(message);
    }
}

public class EmailNotVerifiedException extends RuntimeException {
    public EmailNotVerifiedException(String message) {
        super(message);
    }
}
```

### 9.2 Security Exception Handler

```java
package com.ecommerce.exception;

import com.ecommerce.dto.response.ApiResponse;
import lombok.extern.slf4j.Slf4j;
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.security.access.AccessDeniedException;
import org.springframework.security.authentication.BadCredentialsException;
import org.springframework.security.authentication.DisabledException;
import org.springframework.security.authentication.LockedException;
import org.springframework.security.core.AuthenticationException;
import org.springframework.web.bind.annotation.ExceptionHandler;
import org.springframework.web.bind.annotation.RestControllerAdvice;

@RestControllerAdvice
@Slf4j
public class SecurityExceptionHandler {

    @ExceptionHandler(AuthenticationException.class)
    public ResponseEntity<ApiResponse<Void>> handleAuthenticationException(
            AuthenticationException ex) {
        log.warn("Authentication failed: {}", ex.getMessage());

        return ResponseEntity
                .status(HttpStatus.UNAUTHORIZED)
                .body(ApiResponse.error("Authentication failed: " + ex.getMessage()));
    }

    @ExceptionHandler(BadCredentialsException.class)
    public ResponseEntity<ApiResponse<Void>> handleBadCredentials(
            BadCredentialsException ex) {
        return ResponseEntity
                .status(HttpStatus.UNAUTHORIZED)
                .body(ApiResponse.error("Invalid email or password"));
    }

    @ExceptionHandler(LockedException.class)
    public ResponseEntity<ApiResponse<Void>> handleLockedException(
            LockedException ex) {
        return ResponseEntity
                .status(HttpStatus.LOCKED)
                .body(ApiResponse.error(
                    "Account is locked. Please try again after 30 minutes or contact support."
                ));
    }

    @ExceptionHandler(DisabledException.class)
    public ResponseEntity<ApiResponse<Void>> handleDisabledException(
            DisabledException ex) {
        return ResponseEntity
                .status(HttpStatus.FORBIDDEN)
                .body(ApiResponse.error(
                    "Account is not verified. Please verify your email first."
                ));
    }

    @ExceptionHandler(AccessDeniedException.class)
    public ResponseEntity<ApiResponse<Void>> handleAccessDenied(
            AccessDeniedException ex) {
        log.warn("Access denied: {}", ex.getMessage());

        return ResponseEntity
                .status(HttpStatus.FORBIDDEN)
                .body(ApiResponse.error("You don't have permission to access this resource"));
    }

    @ExceptionHandler(AccountLockedException.class)
    public ResponseEntity<ApiResponse<Void>> handleAccountLocked(
            AccountLockedException ex) {
        return ResponseEntity
                .status(HttpStatus.LOCKED)
                .body(ApiResponse.error(ex.getMessage()));
    }

    @ExceptionHandler(EmailNotVerifiedException.class)
    public ResponseEntity<ApiResponse<Void>> handleEmailNotVerified(
            EmailNotVerifiedException ex) {
        return ResponseEntity
                .status(HttpStatus.FORBIDDEN)
                .body(ApiResponse.error(ex.getMessage()));
    }
}
```

---

## 10. Entry Points

### 10.1 Custom Authentication Entry Point

```java
package com.ecommerce.security;

import com.ecommerce.dto.response.ApiResponse;
import com.fasterxml.jackson.databind.ObjectMapper;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.http.MediaType;
import org.springframework.security.core.AuthenticationException;
import org.springframework.security.web.AuthenticationEntryPoint;
import org.springframework.stereotype.Component;

import java.io.IOException;

@Component
@RequiredArgsConstructor
@Slf4j
public class JwtAuthenticationEntryPoint implements AuthenticationEntryPoint {

    private final ObjectMapper objectMapper;

    @Override
    public void commence(
            HttpServletRequest request,
            HttpServletResponse response,
            AuthenticationException authException) throws IOException {

        log.warn("Unauthorized request to: {} - {}",
                request.getRequestURI(),
                authException.getMessage());

        response.setContentType(MediaType.APPLICATION_JSON_VALUE);
        response.setStatus(HttpServletResponse.SC_UNAUTHORIZED);

        ApiResponse<Void> apiResponse = ApiResponse.error(
                "Unauthorized: " + authException.getMessage()
        );

        objectMapper.writeValue(response.getOutputStream(), apiResponse);
    }
}
```

### 10.2 Custom Access Denied Handler

```java
package com.ecommerce.security;

import com.ecommerce.dto.response.ApiResponse;
import com.fasterxml.jackson.databind.ObjectMapper;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.http.MediaType;
import org.springframework.security.access.AccessDeniedException;
import org.springframework.security.web.access.AccessDeniedHandler;
import org.springframework.stereotype.Component;

import java.io.IOException;

@Component
@RequiredArgsConstructor
@Slf4j
public class CustomAccessDeniedHandler implements AccessDeniedHandler {

    private final ObjectMapper objectMapper;

    @Override
    public void handle(
            HttpServletRequest request,
            HttpServletResponse response,
            AccessDeniedException accessDeniedException) throws IOException {

        log.warn("Access denied for request to: {} - {}",
                request.getRequestURI(),
                accessDeniedException.getMessage());

        response.setContentType(MediaType.APPLICATION_JSON_VALUE);
        response.setStatus(HttpServletResponse.SC_FORBIDDEN);

        ApiResponse<Void> apiResponse = ApiResponse.error(
                "Access Denied: You don't have permission to access this resource"
        );

        objectMapper.writeValue(response.getOutputStream(), apiResponse);
    }
}
```

### 10.3 Cập nhật SecurityConfig

```java
@Configuration
@EnableWebSecurity
@EnableMethodSecurity
@RequiredArgsConstructor
public class SecurityConfig {

    private final JwtAuthenticationEntryPoint authEntryPoint;
    private final CustomAccessDeniedHandler accessDeniedHandler;
    // ... other dependencies

    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
        http
            .csrf(AbstractHttpConfigurer::disable)
            .sessionManagement(session ->
                session.sessionCreationPolicy(SessionCreationPolicy.STATELESS)
            )
            .authorizeHttpRequests(auth -> auth
                // ... authorization rules
            )
            .exceptionHandling(ex -> ex
                .authenticationEntryPoint(authEntryPoint)
                .accessDeniedHandler(accessDeniedHandler)
            )
            .authenticationProvider(authenticationProvider());

        return http.build();
    }
}
```

---

## 11. CORS Configuration

### 11.1 CORS cho REST API

```java
package com.ecommerce.config;

import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.web.cors.CorsConfiguration;
import org.springframework.web.cors.CorsConfigurationSource;
import org.springframework.web.cors.UrlBasedCorsConfigurationSource;

import java.util.Arrays;
import java.util.List;

@Configuration
public class CorsConfig {

    @Value("${app.cors.allowed-origins}")
    private List<String> allowedOrigins;

    @Bean
    public CorsConfigurationSource corsConfigurationSource() {
        CorsConfiguration configuration = new CorsConfiguration();

        // Allowed origins
        configuration.setAllowedOrigins(allowedOrigins);

        // Allowed methods
        configuration.setAllowedMethods(Arrays.asList(
                "GET", "POST", "PUT", "PATCH", "DELETE", "OPTIONS"
        ));

        // Allowed headers
        configuration.setAllowedHeaders(Arrays.asList(
                "Authorization",
                "Content-Type",
                "X-Requested-With",
                "Accept",
                "Origin",
                "Access-Control-Request-Method",
                "Access-Control-Request-Headers"
        ));

        // Exposed headers (client có thể đọc)
        configuration.setExposedHeaders(Arrays.asList(
                "Access-Control-Allow-Origin",
                "Access-Control-Allow-Credentials",
                "Authorization"
        ));

        // Allow credentials (cookies, authorization headers)
        configuration.setAllowCredentials(true);

        // Cache preflight response
        configuration.setMaxAge(3600L);

        UrlBasedCorsConfigurationSource source = new UrlBasedCorsConfigurationSource();
        source.registerCorsConfiguration("/api/**", configuration);

        return source;
    }
}
```

### 11.2 Enable CORS trong SecurityConfig

```java
@Bean
public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
    http
        .cors(cors -> cors.configurationSource(corsConfigurationSource()))
        .csrf(AbstractHttpConfigurer::disable)
        // ... rest of config
        ;
    return http.build();
}
```

### 11.3 application.yml

```yaml
app:
  cors:
    allowed-origins:
      - http://localhost:3000
      - http://localhost:4200
      - https://ecommerce.example.com
```

---

## 12. Bài tập thực hành

### Bài tập 1: Complete Security Setup
Hoàn thành cấu hình Spring Security với:
- [x] SecurityConfig với stateless session
- [x] Custom UserDetailsService
- [x] Password encoding với BCrypt
- [x] Public và protected endpoints

### Bài tập 2: Role-Based Access
Implement authorization cho các role khác nhau:
- CUSTOMER: Xem products, đặt orders
- SELLER: Quản lý sản phẩm của mình
- ADMIN: Full access

### Bài tập 3: Security Utility
Tạo SecurityUtils class để:
- Lấy current user information
- Check ownership của resources
- Validate permissions

---

## 13. Checklist

- [ ] Thêm spring-boot-starter-security dependency
- [ ] Tạo SecurityConfig với filter chain
- [ ] Implement UserDetails trên User entity
- [ ] Tạo CustomUserDetailsService
- [ ] Configure password encoder (BCrypt)
- [ ] Define public vs protected endpoints
- [ ] Implement authentication entry point
- [ ] Implement access denied handler
- [ ] Configure CORS cho frontend
- [ ] Test security với các scenarios khác nhau

---

## Điều hướng

- [← Day 15: Swagger & Logging](../week-03/day-15-swagger.md)
- [Day 17: JWT Authentication →](./day-17-jwt-authentication.md)
- [Về trang chính](../00-overview.md)
