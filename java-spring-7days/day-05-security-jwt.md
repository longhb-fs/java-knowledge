# Day 5: Security + JWT + Registration

> **Mục tiêu:** Hệ thống Authentication hoàn chỉnh — Spring Security + JWT + Register/Login/Logout + Redis Blacklist.
> **Output:** Auth system chạy end-to-end, protect API, role-based access control.

---

## 1. Spring Security Architecture

Khi HTTP request đến app, nó phải đi qua **Security Filter Chain** trước khi vào Controller.

```
HTTP Request → DelegatingFilterProxy → FilterChainProxy →
  CorsFilter → CsrfFilter → JwtAuthFilter (CUSTOM) →
  UsernamePasswordAuthFilter → ExceptionTranslation →
  AuthorizationFilter → DispatcherServlet → Controller
```

- **JwtAuthFilter** (custom): Extract JWT, validate, set authentication vào context.
- **AuthorizationFilter**: Kiểm tra quyền truy cập URL.

### Authentication Flow

```
Request → JwtAuthFilter → extract token → check blacklist (Redis) →
  validate (signature + expiry) → load UserDetails → set SecurityContextHolder → Controller
```

### 4 Khái niệm cốt lõi

| Concept | Giải thích |
|---------|-----------|
| **Authentication** | Object xác thực (`UsernamePasswordAuthenticationToken`) |
| **Principal** | "Ai đăng nhập?" — User entity |
| **GrantedAuthority** | Quyền: `ROLE_ADMIN`, `ROLE_USER` |
| **SecurityContext** | Lưu Authentication, bound theo thread |

---

## 2. SecurityConfig

**File:** `config/SecurityConfig.java`

```java
@Configuration
@EnableWebSecurity
@EnableMethodSecurity    // Bật @PreAuthorize
@RequiredArgsConstructor
public class SecurityConfig {
    private final JwtAuthenticationFilter jwtAuthFilter;
    private final AuthenticationProvider authenticationProvider;

    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
        http
            .csrf(AbstractHttpConfigurer::disable)
            .sessionManagement(s -> s.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/api/v1/auth/**").permitAll()
                .requestMatchers("/swagger-ui/**", "/v3/api-docs/**").permitAll()
                .requestMatchers(HttpMethod.GET, "/api/v1/products/**",
                                                  "/api/v1/categories/**").permitAll()
                .requestMatchers("/api/v1/admin/**").hasRole("ADMIN")
                .anyRequest().authenticated()
            )
            .authenticationProvider(authenticationProvider)
            .addFilterBefore(jwtAuthFilter, UsernamePasswordAuthenticationFilter.class);
        return http.build();
    }

    @Bean
    public AuthenticationProvider authenticationProvider(
            UserDetailsService uds, PasswordEncoder pe) {
        DaoAuthenticationProvider p = new DaoAuthenticationProvider();
        p.setUserDetailsService(uds);
        p.setPasswordEncoder(pe);
        return p;
    }

    @Bean
    public PasswordEncoder passwordEncoder() { return new BCryptPasswordEncoder(); }

    @Bean
    public AuthenticationManager authenticationManager(AuthenticationConfiguration c)
            throws Exception { return c.getAuthenticationManager(); }
}
```

| Config | Tại sao? |
|--------|----------|
| `csrf.disable()` | JWT stateless, CSRF chỉ cần cho cookie-based session |
| `STATELESS` | Server không lưu session, mỗi request tự mang token |
| `hasRole("ADMIN")` | Spring tự thêm prefix `ROLE_`, entity phải trả `ROLE_ADMIN` |
| `addFilterBefore` | JWT filter chạy TRƯỚC filter mặc định |

---

## 3. User implements UserDetails

**File:** `entity/User.java`

```java
@Entity @Table(name = "users")
@Getter @Setter @NoArgsConstructor @AllArgsConstructor @Builder
public class User extends BaseEntity implements UserDetails {
    @Column(nullable = false, length = 100) private String fullName;
    @Column(nullable = false, unique = true) private String email;
    @Column(nullable = false) private String password;
    @Column(length = 15) private String phone;
    @Enumerated(EnumType.STRING)
    @Column(nullable = false, length = 20) private Role role;
    private boolean active = true;
    private boolean emailVerified = false;
    private boolean locked = false;
    private int failedLoginAttempts = 0;
    private Instant lockedAt;
    private String verificationToken;
    private String resetPasswordToken;
    private Instant resetPasswordExpiry;

    // ── UserDetails ─────────────────────────────
    @Override
    public Collection<? extends GrantedAuthority> getAuthorities() {
        return List.of(new SimpleGrantedAuthority("ROLE_" + role.name()));
    }
    @Override public String getUsername() { return email; }
    @Override public boolean isAccountNonExpired() { return true; }
    @Override public boolean isAccountNonLocked() { return !locked; }
    @Override public boolean isCredentialsNonExpired() { return true; }
    @Override public boolean isEnabled() { return active && emailVerified; }
}
```

`getUsername()` trả về `email`. `isEnabled()` kết hợp `active` + `emailVerified`.

### CustomUserDetailsService

```java
@Service
@RequiredArgsConstructor
public class CustomUserDetailsService implements UserDetailsService {
    private final UserRepository userRepository;

    @Override
    public UserDetails loadUserByUsername(String email) throws UsernameNotFoundException {
        return userRepository.findByEmail(email)
                .orElseThrow(() -> new UsernameNotFoundException("User not found: " + email));
    }
}
```

---

## 4. JWT Token Service

### Dependencies: `jjwt-api`, `jjwt-impl`, `jjwt-jackson` version `0.12.6` (pom.xml)

### application.yml + AppProperties

```yaml
app:
  jwt:
    # Base64-encoded, min 256 bits. Tạo: echo -n "secret-key-min-32-chars!" | base64
    secret: "dGhpcyBpcyBhIHZlcnkgc2VjdXJlIHNlY3JldCBrZXkgZm9yIGp3dCB0b2tlbg=="
    access-expiration: 86400000    # 24h
    refresh-expiration: 604800000  # 7 days
```

```java
@Component
@ConfigurationProperties(prefix = "app")
@Getter @Setter
public class AppProperties {
    private Jwt jwt = new Jwt();
    @Getter @Setter
    public static class Jwt {
        private String secret;
        private long accessExpiration;
        private long refreshExpiration;
    }
}
```

### JwtService

**File:** `security/JwtService.java`

```java
@Service @Slf4j
public class JwtService {
    private final SecretKey secretKey;
    private final long accessExpiration;
    private final long refreshExpiration;

    public JwtService(AppProperties props) {
        // FIX: Dùng Decoders.BASE64, không dùng string trực tiếp
        byte[] keyBytes = Decoders.BASE64.decode(props.getJwt().getSecret());
        this.secretKey = Keys.hmacShaKeyFor(keyBytes);
        this.accessExpiration = props.getJwt().getAccessExpiration();
        this.refreshExpiration = props.getJwt().getRefreshExpiration();
    }

    public String generateAccessToken(UserDetails ud) {
        return buildToken(ud, accessExpiration, Map.of("type", "access"));
    }

    public String generateRefreshToken(UserDetails ud) {
        return buildToken(ud, refreshExpiration, Map.of("type", "refresh"));
    }

    private String buildToken(UserDetails ud, long exp, Map<String, Object> claims) {
        return Jwts.builder()
                .subject(ud.getUsername())
                .claims(claims)
                .issuedAt(new Date())
                .expiration(new Date(System.currentTimeMillis() + exp))
                .signWith(secretKey)
                .compact();
    }

    public String extractUsername(String token) {
        return extractClaim(token, Claims::getSubject);
    }

    public boolean isTokenValid(String token, UserDetails ud) {
        return extractUsername(token).equals(ud.getUsername()) && !isTokenExpired(token);
    }

    private boolean isTokenExpired(String token) {
        return extractClaim(token, Claims::getExpiration).before(new Date());
    }

    public long getRemainingExpiration(String token) {
        return extractClaim(token, Claims::getExpiration).getTime() - System.currentTimeMillis();
    }

    private <T> T extractClaim(String token, Function<Claims, T> resolver) {
        Claims claims = Jwts.parser().verifyWith(secretKey).build()
                .parseSignedClaims(token).getPayload();
        return resolver.apply(claims);
    }
}
```

**Lỗi hay gặp:**

| Lỗi | Fix |
|-----|-----|
| `WeakKeyException` | Secret key < 256 bits → dùng key dài hơn |
| `SignatureException` | Key không khớp → kiểm tra secret config |
| `ExpiredJwtException` | Token hết hạn → dùng refresh token |

---

## 5. JWT Authentication Filter

**File:** `security/JwtAuthenticationFilter.java`

```java
@Component
@RequiredArgsConstructor @Slf4j
public class JwtAuthenticationFilter extends OncePerRequestFilter {
    private final JwtService jwtService;
    private final UserDetailsService userDetailsService;
    private final TokenBlacklistService tokenBlacklistService;

    @Override
    protected void doFilterInternal(HttpServletRequest request,
            HttpServletResponse response, FilterChain filterChain)
            throws ServletException, IOException {

        final String authHeader = request.getHeader("Authorization");
        if (authHeader == null || !authHeader.startsWith("Bearer ")) {
            filterChain.doFilter(request, response);
            return;
        }

        final String jwt = authHeader.substring(7);
        if (tokenBlacklistService.isBlacklisted(jwt)) {
            filterChain.doFilter(request, response);
            return;
        }

        try {
            final String userEmail = jwtService.extractUsername(jwt);
            if (userEmail != null
                    && SecurityContextHolder.getContext().getAuthentication() == null) {
                UserDetails ud = userDetailsService.loadUserByUsername(userEmail);
                if (jwtService.isTokenValid(jwt, ud)) {
                    var authToken = new UsernamePasswordAuthenticationToken(
                            ud, null, ud.getAuthorities());
                    authToken.setDetails(
                            new WebAuthenticationDetailsSource().buildDetails(request));
                    SecurityContextHolder.getContext().setAuthentication(authToken);
                }
            }
        } catch (Exception e) {
            log.error("JWT auth failed: {}", e.getMessage());
        }
        filterChain.doFilter(request, response);
    }
}
```

Extends `OncePerRequestFilter` để đảm bảo chỉ chạy 1 lần/request. Catch exception mà không throw — nếu URL là public thì request vẫn đi tiếp bình thường.

---

## 6. Token Blacklist với Redis

JWT stateless → không thể "thu hồi". Giải pháp: lưu token đã logout vào Redis với TTL.

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-redis</artifactId>
</dependency>
```

**File:** `service/TokenBlacklistService.java`

```java
@Service
@RequiredArgsConstructor
public class TokenBlacklistService {
    private final StringRedisTemplate redisTemplate;
    private static final String PREFIX = "token:blacklist:";

    public void blacklist(String token, long expirationMs) {
        if (expirationMs <= 0) return;
        redisTemplate.opsForValue().set(PREFIX + token, "1", expirationMs, TimeUnit.MILLISECONDS);
    }

    public boolean isBlacklisted(String token) {
        return Boolean.TRUE.equals(redisTemplate.hasKey(PREFIX + token));
    }
}
```

Redis TTL tự động xóa key khi token hết hạn → không tốn memory vĩnh viễn.

---

## 7. Auth DTOs

```java
// dto/request/LoginRequest.java
@Getter @Setter
public class LoginRequest {
    @NotBlank @Email private String email;
    @NotBlank private String password;
}

// dto/request/RegisterRequest.java
@Getter @Setter
public class RegisterRequest {
    @NotBlank @Size(max = 100) private String fullName;
    @NotBlank @Email private String email;
    @NotBlank @Size(min = 8) private String password;
    @Size(max = 15) private String phone;
}

// dto/request/PasswordResetRequest.java
@Getter @Setter
public class PasswordResetRequest {
    @NotBlank private String token;
    @NotBlank @Size(min = 8) private String newPassword;
}

// dto/response/AuthResponse.java
@Getter @Setter @Builder @NoArgsConstructor @AllArgsConstructor
public class AuthResponse {
    private String accessToken;
    private String refreshToken;
    @Builder.Default private String tokenType = "Bearer";
    private long expiresIn;
    private UserResponse user;
}

// dto/response/UserResponse.java
@Getter @Setter @Builder @NoArgsConstructor @AllArgsConstructor
public class UserResponse {
    private Long id;
    private String fullName;
    private String email;
    private String phone;
    private Role role;
    private boolean emailVerified;
}
```

---

## 8. Auth Service

**File:** `service/AuthService.java`

```java
@Service @RequiredArgsConstructor @Slf4j
public class AuthService {
    private final UserRepository userRepository;
    private final PasswordEncoder passwordEncoder;
    private final JwtService jwtService;
    private final AuthenticationManager authenticationManager;
    private final TokenBlacklistService tokenBlacklistService;
    private final AppProperties appProperties;
    private static final int MAX_FAILED = 5;
    private static final Duration LOCK_TIME = Duration.ofMinutes(30);

    @Transactional
    public AuthResponse register(RegisterRequest req) {
        if (userRepository.existsByEmail(req.getEmail()))
            throw new BadRequestException("Email already registered");
        User user = User.builder()
                .fullName(req.getFullName()).email(req.getEmail())
                .password(passwordEncoder.encode(req.getPassword()))
                .phone(req.getPhone()).role(Role.USER).active(true)
                .emailVerified(false)
                .verificationToken(UUID.randomUUID().toString()).build();
        userRepository.save(user);
        // TODO: Send verification email (async)
        return buildAuthResponse(user);
    }

    @Transactional
    public AuthResponse login(LoginRequest req) {
        User user = userRepository.findByEmail(req.getEmail())
                .orElseThrow(() -> new BadCredentialsException("Invalid email or password"));

        if (user.isLocked()) {
            if (user.getLockedAt() != null
                    && Instant.now().isAfter(user.getLockedAt().plus(LOCK_TIME))) {
                user.setLocked(false);
                user.setLockedAt(null);
                user.setFailedLoginAttempts(0);  // FIX: Reset counter khi auto-unlock
                userRepository.save(user);
            } else {
                throw new LockedException("Account locked. Try after 30 min.");
            }
        }

        try {
            authenticationManager.authenticate(
                new UsernamePasswordAuthenticationToken(req.getEmail(), req.getPassword()));
        } catch (BadCredentialsException e) {
            int attempts = user.getFailedLoginAttempts() + 1;
            user.setFailedLoginAttempts(attempts);
            if (attempts >= MAX_FAILED) {
                user.setLocked(true);
                user.setLockedAt(Instant.now());
            }
            userRepository.save(user);
            throw new BadCredentialsException("Invalid email or password");
        }

        if (user.getFailedLoginAttempts() > 0) {
            user.setFailedLoginAttempts(0);
            userRepository.save(user);
        }
        return buildAuthResponse(user);
    }

    public void logout(String token) {
        tokenBlacklistService.blacklist(token, jwtService.getRemainingExpiration(token));
    }

    @Transactional
    public void verifyEmail(String token) {
        User user = userRepository.findByVerificationToken(token)
                .orElseThrow(() -> new BadRequestException("Invalid verification token"));
        user.setEmailVerified(true);
        user.setVerificationToken(null);
        userRepository.save(user);
    }

    public void forgotPassword(String email) {
        // Anti-enumeration: luôn trả success, KHÔNG tiết lộ email có tồn tại không
        userRepository.findByEmail(email).ifPresent(user -> {
            user.setResetPasswordToken(UUID.randomUUID().toString());
            user.setResetPasswordExpiry(Instant.now().plus(Duration.ofHours(1)));
            userRepository.save(user);
            // TODO: Send reset email
        });
    }

    @Transactional
    public void resetPassword(String token, String newPassword) {
        User user = userRepository.findByResetPasswordToken(token)
                .orElseThrow(() -> new BadRequestException("Invalid reset token"));
        if (user.getResetPasswordExpiry() == null
                || Instant.now().isAfter(user.getResetPasswordExpiry()))
            throw new BadRequestException("Reset token expired");
        user.setPassword(passwordEncoder.encode(newPassword));
        user.setResetPasswordToken(null);
        user.setResetPasswordExpiry(null);
        userRepository.save(user);
    }

    private AuthResponse buildAuthResponse(User user) {
        return AuthResponse.builder()
                .accessToken(jwtService.generateAccessToken(user))
                .refreshToken(jwtService.generateRefreshToken(user))
                .expiresIn(appProperties.getJwt().getAccessExpiration() / 1000)
                .user(UserResponse.builder()
                    .id(user.getId()).fullName(user.getFullName())
                    .email(user.getEmail()).phone(user.getPhone())
                    .role(user.getRole()).emailVerified(user.isEmailVerified()).build())
                .build();
    }
}
```

**Highlights:**
1. **Account Locking**: 5 lần sai → khóa 30 phút → tự mở.
2. **Anti-Enumeration**: `forgotPassword()` luôn trả success.
3. **FIX**: Reset `failedLoginAttempts=0` khi auto-unlock — nếu không, sai 1 lần nữa là khóa lại.

---

## 9. Auth Controller

**File:** `controller/AuthController.java`

```java
@RestController
@RequestMapping("/api/v1/auth")
@RequiredArgsConstructor
@Tag(name = "Authentication")
public class AuthController {
    private final AuthService authService;

    @PostMapping("/register")
    public ResponseEntity<ApiResponse<AuthResponse>> register(
            @Valid @RequestBody RegisterRequest req) {
        return ResponseEntity.status(HttpStatus.CREATED)
                .body(ApiResponse.success("Registered", authService.register(req)));
    }

    @PostMapping("/login")
    public ResponseEntity<ApiResponse<AuthResponse>> login(
            @Valid @RequestBody LoginRequest req) {
        return ResponseEntity.ok(ApiResponse.success("Login OK", authService.login(req)));
    }

    @PostMapping("/logout")
    public ResponseEntity<Void> logout(@RequestHeader("Authorization") String h) {
        authService.logout(h.substring(7));
        return ResponseEntity.noContent().build();
    }

    @PostMapping("/verify-email")
    public ResponseEntity<ApiResponse<Void>> verifyEmail(@RequestParam String token) {
        authService.verifyEmail(token);
        return ResponseEntity.ok(ApiResponse.success("Email verified"));
    }

    @PostMapping("/forgot-password")
    public ResponseEntity<ApiResponse<Void>> forgotPassword(@RequestParam String email) {
        authService.forgotPassword(email);
        return ResponseEntity.ok(ApiResponse.success("If email exists, reset link sent"));
    }

    @PostMapping("/reset-password")
    public ResponseEntity<ApiResponse<Void>> resetPassword(
            @Valid @RequestBody PasswordResetRequest req) {
        authService.resetPassword(req.getToken(), req.getNewPassword());
        return ResponseEntity.ok(ApiResponse.success("Password reset OK"));
    }
}
```

---

## 10. Method-Level Security với @PreAuthorize

```java
@PreAuthorize("hasRole('ADMIN')")
public void deleteProduct(Long id) { ... }

@PreAuthorize("hasRole('ADMIN') or #userId == authentication.principal.id")
public UserResponse getUser(Long userId) { ... }

@PreAuthorize("authentication.principal.emailVerified")
public OrderResponse createOrder(OrderCreateRequest req) { ... }
```

### SecurityUtils — Lấy current user

```java
public final class SecurityUtils {
    private SecurityUtils() {}

    public static User getCurrentUser() {
        Authentication auth = SecurityContextHolder.getContext().getAuthentication();
        if (auth == null || !auth.isAuthenticated())
            throw new IllegalStateException("No authenticated user");
        return (User) auth.getPrincipal();
    }

    public static Long getCurrentUserId() { return getCurrentUser().getId(); }
}

// Sử dụng:
Long userId = SecurityUtils.getCurrentUserId();
```

| SpEL Expression | Ý nghĩa |
|----------------|---------|
| `hasRole('ADMIN')` | Có role ADMIN |
| `hasAnyRole('ADMIN','MANAGER')` | Có 1 trong các role |
| `isAuthenticated()` | Đã đăng nhập |
| `#userId == authentication.principal.id` | Param = ID user đăng nhập |

---

## 11. Testing Auth APIs

```bash
docker compose up -d mysql redis
./mvnw spring-boot:run -Dspring-boot.run.profiles=dev
```

```bash
# Register
curl -X POST http://localhost:8080/api/v1/auth/register \
  -H "Content-Type: application/json" \
  -d '{"fullName":"Nguyen Van A","email":"a@test.com","password":"12345678"}'
# → 201: { accessToken, refreshToken, expiresIn, user: {id, fullName, role} }

# Login
curl -X POST http://localhost:8080/api/v1/auth/login \
  -H "Content-Type: application/json" \
  -d '{"email":"a@test.com","password":"12345678"}'
# → 200: same response format

# Protected — không token → 403 | có token → 200
curl http://localhost:8080/api/v1/cart -H "Authorization: Bearer <token>"

# Public — không cần token → 200
curl http://localhost:8080/api/v1/products

# Logout → blacklist, dùng lại token cũ → 403
curl -X POST http://localhost:8080/api/v1/auth/logout -H "Authorization: Bearer <token>"
```

### Common Errors

| Error | Fix |
|-------|-----|
| `403 Forbidden` | Token thiếu/sai/hết hạn → login lại |
| `Account is locked` | Đợi 30 phút hoặc admin unlock |
| `WeakKeyException` | JWT secret < 32 bytes → Base64 encode key dài hơn |

---

## Tổng Kết

**Files:** `SecurityConfig`, `AppProperties`, `JwtService`, `JwtAuthenticationFilter`, `CustomUserDetailsService`, `AuthService`, `TokenBlacklistService`, `AuthController`, `LoginRequest`, `RegisterRequest`, `PasswordResetRequest`, `AuthResponse`, `UserResponse`, `User.java` (modified), `Role.java`, `SecurityUtils`

---

> **Day 6** → Cart + Order + Payment: dùng auth system để protect cart/order APIs.
