# Day 17: JWT Authentication

## Mục tiêu học tập
- Hiểu cấu trúc và cách hoạt động của JWT
- Implement JWT Token generation
- Tạo JWT Authentication Filter
- Secure REST API với JWT

---

## 1. JWT (JSON Web Token) Overview

### 1.1 JWT là gì?

```
┌─────────────────────────────────────────────────────────────────┐
│                    JWT STRUCTURE                                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.                         │
│  eyJzdWIiOiIxMjM0NTY3ODkwIiwibmFtZSI6IkpvaG4iLCJpYXQiOjE1MTY.  │
│  SflKxwRJSMeKKF2QT4fwpMeJf36POk6yJV_adQssw5c                   │
│                                                                  │
│  ┌──────────────────┬─────────────────────┬──────────────────┐  │
│  │     HEADER       │      PAYLOAD        │    SIGNATURE     │  │
│  │   (Algorithm)    │     (Claims)        │   (Verify)       │  │
│  └──────────────────┴─────────────────────┴──────────────────┘  │
│         RED              PURPLE                 CYAN             │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 1.2 JWT Flow

```
┌─────────────────────────────────────────────────────────────────┐
│                    JWT AUTHENTICATION FLOW                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│    CLIENT                                           SERVER       │
│       │                                                │         │
│       │  1. POST /auth/login                          │         │
│       │     {email, password}                         │         │
│       │ ─────────────────────────────────────────────▶│         │
│       │                                                │         │
│       │               ┌──────────────────────────┐    │         │
│       │               │ Validate Credentials     │    │         │
│       │               │ Generate JWT Token       │    │         │
│       │               │ Return Access + Refresh  │    │         │
│       │               └──────────────────────────┘    │         │
│       │                                                │         │
│       │  2. Response: {accessToken, refreshToken}     │         │
│       │◀─────────────────────────────────────────────│         │
│       │                                                │         │
│ ┌─────────────────────┐                               │         │
│ │ Store tokens in     │                               │         │
│ │ localStorage/cookie │                               │         │
│ └─────────────────────┘                               │         │
│       │                                                │         │
│       │  3. GET /api/orders                           │         │
│       │     Header: Authorization: Bearer <token>     │         │
│       │ ─────────────────────────────────────────────▶│         │
│       │                                                │         │
│       │               ┌──────────────────────────┐    │         │
│       │               │ Validate JWT Signature   │    │         │
│       │               │ Check Expiration         │    │         │
│       │               │ Extract User from Token  │    │         │
│       │               │ Set SecurityContext      │    │         │
│       │               └──────────────────────────┘    │         │
│       │                                                │         │
│       │  4. Response: Order data                      │         │
│       │◀─────────────────────────────────────────────│         │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## 2. JWT Dependencies

### 2.1 pom.xml

```xml
<dependencies>
    <!-- JWT Library - JJWT -->
    <dependency>
        <groupId>io.jsonwebtoken</groupId>
        <artifactId>jjwt-api</artifactId>
        <version>0.12.5</version>
    </dependency>
    <dependency>
        <groupId>io.jsonwebtoken</groupId>
        <artifactId>jjwt-impl</artifactId>
        <version>0.12.5</version>
        <scope>runtime</scope>
    </dependency>
    <dependency>
        <groupId>io.jsonwebtoken</groupId>
        <artifactId>jjwt-jackson</artifactId>
        <version>0.12.5</version>
        <scope>runtime</scope>
    </dependency>
</dependencies>
```

---

## 3. JWT Configuration

### 3.1 application.yml

```yaml
app:
  jwt:
    secret: ${JWT_SECRET:your-256-bit-secret-key-must-be-at-least-32-chars-long}
    access-token:
      expiration: 900000      # 15 minutes in milliseconds
    refresh-token:
      expiration: 604800000   # 7 days in milliseconds
    issuer: ecommerce-api
    audience: ecommerce-client
```

### 3.2 JwtProperties

```java
package com.ecommerce.config.properties;

import lombok.Data;
import org.springframework.boot.context.properties.ConfigurationProperties;
import org.springframework.stereotype.Component;

@Component
@ConfigurationProperties(prefix = "app.jwt")
@Data
public class JwtProperties {

    private String secret;
    private String issuer;
    private String audience;
    private AccessToken accessToken = new AccessToken();
    private RefreshToken refreshToken = new RefreshToken();

    @Data
    public static class AccessToken {
        private long expiration = 900000; // 15 minutes
    }

    @Data
    public static class RefreshToken {
        private long expiration = 604800000; // 7 days
    }
}
```

---

## 4. JWT Service

### 4.1 JwtTokenProvider

```java
package com.ecommerce.security.jwt;

import com.ecommerce.config.properties.JwtProperties;
import com.ecommerce.domain.entity.User;
import io.jsonwebtoken.*;
import io.jsonwebtoken.io.Decoders;
import io.jsonwebtoken.security.Keys;
import io.jsonwebtoken.security.SignatureException;
import lombok.extern.slf4j.Slf4j;
import org.springframework.security.core.Authentication;
import org.springframework.security.core.GrantedAuthority;
import org.springframework.stereotype.Component;

import javax.crypto.SecretKey;
import java.util.Date;
import java.util.HashMap;
import java.util.Map;
import java.util.function.Function;
import java.util.stream.Collectors;

@Component
@Slf4j
public class JwtTokenProvider {

    private final JwtProperties jwtProperties;
    private final SecretKey secretKey;

    public JwtTokenProvider(JwtProperties jwtProperties) {
        this.jwtProperties = jwtProperties;
        this.secretKey = Keys.hmacShaKeyFor(
            Decoders.BASE64.decode(jwtProperties.getSecret())
        );
    }

    /**
     * Generate Access Token từ Authentication
     */
    public String generateAccessToken(Authentication authentication) {
        User user = (User) authentication.getPrincipal();
        return generateAccessToken(user);
    }

    /**
     * Generate Access Token từ User entity
     */
    public String generateAccessToken(User user) {
        Map<String, Object> claims = new HashMap<>();
        claims.put("userId", user.getId());
        claims.put("email", user.getEmail());
        claims.put("role", user.getRole().name());
        claims.put("authorities", user.getAuthorities().stream()
                .map(GrantedAuthority::getAuthority)
                .collect(Collectors.toList()));

        return buildToken(claims, user.getEmail(),
                jwtProperties.getAccessToken().getExpiration());
    }

    /**
     * Generate Refresh Token
     */
    public String generateRefreshToken(User user) {
        Map<String, Object> claims = new HashMap<>();
        claims.put("userId", user.getId());
        claims.put("tokenType", "REFRESH");

        return buildToken(claims, user.getEmail(),
                jwtProperties.getRefreshToken().getExpiration());
    }

    /**
     * Build JWT Token
     */
    private String buildToken(Map<String, Object> claims, String subject, long expiration) {
        Date now = new Date();
        Date expiryDate = new Date(now.getTime() + expiration);

        return Jwts.builder()
                .claims(claims)
                .subject(subject)
                .issuer(jwtProperties.getIssuer())
                .audience().add(jwtProperties.getAudience()).and()
                .issuedAt(now)
                .expiration(expiryDate)
                .signWith(secretKey, Jwts.SIG.HS256)
                .compact();
    }

    /**
     * Extract Username (email) từ Token
     */
    public String extractUsername(String token) {
        return extractClaim(token, Claims::getSubject);
    }

    /**
     * Extract User ID từ Token
     */
    public Long extractUserId(String token) {
        Claims claims = extractAllClaims(token);
        return claims.get("userId", Long.class);
    }

    /**
     * Extract Role từ Token
     */
    public String extractRole(String token) {
        Claims claims = extractAllClaims(token);
        return claims.get("role", String.class);
    }

    /**
     * Extract Expiration Date
     */
    public Date extractExpiration(String token) {
        return extractClaim(token, Claims::getExpiration);
    }

    /**
     * Extract specific claim
     */
    public <T> T extractClaim(String token, Function<Claims, T> claimsResolver) {
        final Claims claims = extractAllClaims(token);
        return claimsResolver.apply(claims);
    }

    /**
     * Extract all claims từ Token
     */
    private Claims extractAllClaims(String token) {
        return Jwts.parser()
                .verifyWith(secretKey)
                .build()
                .parseSignedClaims(token)
                .getPayload();
    }

    /**
     * Validate Token
     */
    public boolean validateToken(String token) {
        try {
            Jwts.parser()
                    .verifyWith(secretKey)
                    .build()
                    .parseSignedClaims(token);
            return true;
        } catch (SignatureException ex) {
            log.error("Invalid JWT signature: {}", ex.getMessage());
        } catch (MalformedJwtException ex) {
            log.error("Invalid JWT token: {}", ex.getMessage());
        } catch (ExpiredJwtException ex) {
            log.error("JWT token is expired: {}", ex.getMessage());
        } catch (UnsupportedJwtException ex) {
            log.error("JWT token is unsupported: {}", ex.getMessage());
        } catch (IllegalArgumentException ex) {
            log.error("JWT claims string is empty: {}", ex.getMessage());
        }
        return false;
    }

    /**
     * Validate Token với UserDetails
     */
    public boolean validateToken(String token, User user) {
        final String username = extractUsername(token);
        return username.equals(user.getEmail()) && !isTokenExpired(token);
    }

    /**
     * Check Token expired
     */
    public boolean isTokenExpired(String token) {
        return extractExpiration(token).before(new Date());
    }

    /**
     * Get remaining time until expiration (ms)
     */
    public long getExpirationTime(String token) {
        Date expiration = extractExpiration(token);
        return expiration.getTime() - System.currentTimeMillis();
    }

    /**
     * Check if token is refresh token
     */
    public boolean isRefreshToken(String token) {
        Claims claims = extractAllClaims(token);
        String tokenType = claims.get("tokenType", String.class);
        return "REFRESH".equals(tokenType);
    }
}
```

### 4.2 JWT Token Structure

```
┌─────────────────────────────────────────────────────────────────┐
│                    ACCESS TOKEN CLAIMS                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  {                                                               │
│    "sub": "user@example.com",         // Subject (email)        │
│    "userId": 123,                     // User ID                │
│    "email": "user@example.com",       // Email                  │
│    "role": "CUSTOMER",                // Role                   │
│    "authorities": ["ROLE_CUSTOMER"],  // Authorities            │
│    "iss": "ecommerce-api",            // Issuer                 │
│    "aud": "ecommerce-client",         // Audience               │
│    "iat": 1704067200,                 // Issued At              │
│    "exp": 1704068100                  // Expiration (15 min)    │
│  }                                                               │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│                    REFRESH TOKEN CLAIMS                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  {                                                               │
│    "sub": "user@example.com",         // Subject (email)        │
│    "userId": 123,                     // User ID                │
│    "tokenType": "REFRESH",            // Token Type             │
│    "iss": "ecommerce-api",            // Issuer                 │
│    "aud": "ecommerce-client",         // Audience               │
│    "iat": 1704067200,                 // Issued At              │
│    "exp": 1704672000                  // Expiration (7 days)    │
│  }                                                               │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## 5. JWT Authentication Filter

### 5.1 JwtAuthenticationFilter

```java
package com.ecommerce.security.jwt;

import com.ecommerce.service.impl.CustomUserDetailsService;
import jakarta.servlet.FilterChain;
import jakarta.servlet.ServletException;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.lang.NonNull;
import org.springframework.security.authentication.UsernamePasswordAuthenticationToken;
import org.springframework.security.core.context.SecurityContextHolder;
import org.springframework.security.core.userdetails.UserDetails;
import org.springframework.security.web.authentication.WebAuthenticationDetailsSource;
import org.springframework.stereotype.Component;
import org.springframework.util.StringUtils;
import org.springframework.web.filter.OncePerRequestFilter;

import java.io.IOException;

@Component
@RequiredArgsConstructor
@Slf4j
public class JwtAuthenticationFilter extends OncePerRequestFilter {

    private final JwtTokenProvider jwtTokenProvider;
    private final CustomUserDetailsService userDetailsService;

    private static final String AUTHORIZATION_HEADER = "Authorization";
    private static final String BEARER_PREFIX = "Bearer ";

    @Override
    protected void doFilterInternal(
            @NonNull HttpServletRequest request,
            @NonNull HttpServletResponse response,
            @NonNull FilterChain filterChain) throws ServletException, IOException {

        try {
            // 1. Extract JWT from request
            String jwt = extractJwtFromRequest(request);

            // 2. Validate token và set authentication
            if (StringUtils.hasText(jwt) && jwtTokenProvider.validateToken(jwt)) {

                // 3. Extract username from token
                String username = jwtTokenProvider.extractUsername(jwt);

                // 4. Load user details
                UserDetails userDetails = userDetailsService.loadUserByUsername(username);

                // 5. Validate token với user
                if (jwtTokenProvider.validateToken(jwt, (com.ecommerce.domain.entity.User) userDetails)) {

                    // 6. Create authentication object
                    UsernamePasswordAuthenticationToken authentication =
                            new UsernamePasswordAuthenticationToken(
                                    userDetails,
                                    null,  // No credentials needed
                                    userDetails.getAuthorities()
                            );

                    // 7. Set request details
                    authentication.setDetails(
                            new WebAuthenticationDetailsSource().buildDetails(request)
                    );

                    // 8. Set authentication in SecurityContext
                    SecurityContextHolder.getContext().setAuthentication(authentication);

                    log.debug("Authenticated user: {}", username);
                }
            }
        } catch (Exception ex) {
            log.error("Cannot set user authentication: {}", ex.getMessage());
        }

        // Continue filter chain
        filterChain.doFilter(request, response);
    }

    /**
     * Extract JWT token từ Authorization header
     */
    private String extractJwtFromRequest(HttpServletRequest request) {
        String bearerToken = request.getHeader(AUTHORIZATION_HEADER);

        if (StringUtils.hasText(bearerToken) && bearerToken.startsWith(BEARER_PREFIX)) {
            return bearerToken.substring(BEARER_PREFIX.length());
        }

        return null;
    }

    /**
     * Skip filter cho các paths không cần authentication
     */
    @Override
    protected boolean shouldNotFilter(HttpServletRequest request) {
        String path = request.getServletPath();
        return path.startsWith("/api/v1/auth/") ||
               path.startsWith("/swagger-ui/") ||
               path.startsWith("/v3/api-docs/") ||
               path.equals("/actuator/health");
    }
}
```

### 5.2 Filter Flow Diagram

```
┌─────────────────────────────────────────────────────────────────┐
│                  JWT FILTER FLOW                                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  REQUEST: GET /api/v1/orders                                     │
│  Header: Authorization: Bearer eyJhbGciOiJIUzI1NiIs...          │
│                              │                                   │
│                              ▼                                   │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │ 1. Extract JWT from Authorization header                 │   │
│  │    jwt = "eyJhbGciOiJIUzI1NiIs..."                       │   │
│  └─────────────────────────────────────────────────────────┘   │
│                              │                                   │
│                              ▼                                   │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │ 2. Validate JWT Signature & Expiration                   │   │
│  │    jwtTokenProvider.validateToken(jwt)                   │   │
│  └─────────────────────────────────────────────────────────┘   │
│                              │                                   │
│                   ┌──────────┴──────────┐                       │
│                   │                     │                        │
│                 VALID                INVALID                     │
│                   │                     │                        │
│                   ▼                     ▼                        │
│  ┌───────────────────────────┐  ┌────────────────────────────┐ │
│  │ 3. Extract username       │  │ Continue without auth      │ │
│  │ 4. Load UserDetails       │  │ (Will fail at endpoint)    │ │
│  │ 5. Create Authentication  │  └────────────────────────────┘ │
│  │ 6. Set SecurityContext    │                                  │
│  └───────────────────────────┘                                  │
│                              │                                   │
│                              ▼                                   │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │ 7. filterChain.doFilter(request, response)               │   │
│  │    → Continue to Controller                              │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## 6. Update SecurityConfig

### 6.1 Thêm JWT Filter vào Filter Chain

```java
package com.ecommerce.config;

import com.ecommerce.security.CustomAccessDeniedHandler;
import com.ecommerce.security.JwtAuthenticationEntryPoint;
import com.ecommerce.security.jwt.JwtAuthenticationFilter;
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
import org.springframework.security.web.authentication.UsernamePasswordAuthenticationFilter;
import org.springframework.web.cors.CorsConfigurationSource;

@Configuration
@EnableWebSecurity
@EnableMethodSecurity
@RequiredArgsConstructor
public class SecurityConfig {

    private final UserDetailsService userDetailsService;
    private final JwtAuthenticationFilter jwtAuthFilter;
    private final JwtAuthenticationEntryPoint authEntryPoint;
    private final CustomAccessDeniedHandler accessDeniedHandler;
    private final CorsConfigurationSource corsConfigurationSource;

    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
        http
            // Enable CORS
            .cors(cors -> cors.configurationSource(corsConfigurationSource))

            // Disable CSRF (using JWT)
            .csrf(AbstractHttpConfigurer::disable)

            // Stateless session
            .sessionManagement(session ->
                session.sessionCreationPolicy(SessionCreationPolicy.STATELESS)
            )

            // Exception handling
            .exceptionHandling(ex -> ex
                .authenticationEntryPoint(authEntryPoint)
                .accessDeniedHandler(accessDeniedHandler)
            )

            // Authorization rules
            .authorizeHttpRequests(auth -> auth
                // Auth endpoints - Public
                .requestMatchers("/api/v1/auth/**").permitAll()

                // Public read endpoints
                .requestMatchers(HttpMethod.GET, "/api/v1/products/**").permitAll()
                .requestMatchers(HttpMethod.GET, "/api/v1/categories/**").permitAll()
                .requestMatchers(HttpMethod.GET, "/api/v1/reviews/**").permitAll()

                // Swagger
                .requestMatchers(
                    "/swagger-ui/**",
                    "/swagger-ui.html",
                    "/v3/api-docs/**",
                    "/swagger-resources/**",
                    "/webjars/**"
                ).permitAll()

                // Actuator
                .requestMatchers("/actuator/health", "/actuator/info").permitAll()

                // Admin endpoints
                .requestMatchers("/api/v1/admin/**").hasRole("ADMIN")

                // All other requests require authentication
                .anyRequest().authenticated()
            )

            // Authentication provider
            .authenticationProvider(authenticationProvider())

            // Add JWT filter before UsernamePasswordAuthenticationFilter
            .addFilterBefore(jwtAuthFilter, UsernamePasswordAuthenticationFilter.class);

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
        return new BCryptPasswordEncoder(12);
    }
}
```

---

## 7. Auth DTOs

### 7.1 Request DTOs

```java
package com.ecommerce.dto.request;

import jakarta.validation.constraints.Email;
import jakarta.validation.constraints.NotBlank;
import jakarta.validation.constraints.Size;
import lombok.Data;

@Data
public class LoginRequest {

    @NotBlank(message = "Email is required")
    @Email(message = "Invalid email format")
    private String email;

    @NotBlank(message = "Password is required")
    private String password;
}
```

```java
package com.ecommerce.dto.request;

import jakarta.validation.constraints.NotBlank;
import lombok.Data;

@Data
public class RefreshTokenRequest {

    @NotBlank(message = "Refresh token is required")
    private String refreshToken;
}
```

### 7.2 Response DTOs

```java
package com.ecommerce.dto.response;

import com.fasterxml.jackson.annotation.JsonInclude;
import lombok.AllArgsConstructor;
import lombok.Builder;
import lombok.Data;
import lombok.NoArgsConstructor;

@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
@JsonInclude(JsonInclude.Include.NON_NULL)
public class AuthResponse {

    private String accessToken;
    private String refreshToken;
    private String tokenType;
    private Long expiresIn;
    private UserInfo user;

    @Data
    @Builder
    @NoArgsConstructor
    @AllArgsConstructor
    public static class UserInfo {
        private Long id;
        private String email;
        private String firstName;
        private String lastName;
        private String role;
    }

    public static AuthResponse of(String accessToken, String refreshToken,
                                   Long expiresIn, UserInfo user) {
        return AuthResponse.builder()
                .accessToken(accessToken)
                .refreshToken(refreshToken)
                .tokenType("Bearer")
                .expiresIn(expiresIn)
                .user(user)
                .build();
    }
}
```

---

## 8. Auth Service

### 8.1 AuthService Interface

```java
package com.ecommerce.service;

import com.ecommerce.dto.request.LoginRequest;
import com.ecommerce.dto.request.RefreshTokenRequest;
import com.ecommerce.dto.response.AuthResponse;

public interface AuthService {

    AuthResponse login(LoginRequest request);

    AuthResponse refreshToken(RefreshTokenRequest request);

    void logout(String token);
}
```

### 8.2 AuthServiceImpl

```java
package com.ecommerce.service.impl;

import com.ecommerce.config.properties.JwtProperties;
import com.ecommerce.domain.entity.User;
import com.ecommerce.dto.request.LoginRequest;
import com.ecommerce.dto.request.RefreshTokenRequest;
import com.ecommerce.dto.response.AuthResponse;
import com.ecommerce.exception.UnauthorizedException;
import com.ecommerce.repository.UserRepository;
import com.ecommerce.security.jwt.JwtTokenProvider;
import com.ecommerce.service.AuthService;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.security.authentication.AuthenticationManager;
import org.springframework.security.authentication.BadCredentialsException;
import org.springframework.security.authentication.DisabledException;
import org.springframework.security.authentication.LockedException;
import org.springframework.security.authentication.UsernamePasswordAuthenticationToken;
import org.springframework.security.core.Authentication;
import org.springframework.security.core.AuthenticationException;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

@Service
@RequiredArgsConstructor
@Slf4j
public class AuthServiceImpl implements AuthService {

    private final AuthenticationManager authenticationManager;
    private final JwtTokenProvider jwtTokenProvider;
    private final JwtProperties jwtProperties;
    private final UserRepository userRepository;
    private final TokenBlacklistService tokenBlacklistService;

    @Override
    @Transactional
    public AuthResponse login(LoginRequest request) {
        log.info("Login attempt for email: {}", request.getEmail());

        try {
            // 1. Authenticate with email/password
            Authentication authentication = authenticationManager.authenticate(
                    new UsernamePasswordAuthenticationToken(
                            request.getEmail(),
                            request.getPassword()
                    )
            );

            // 2. Get authenticated user
            User user = (User) authentication.getPrincipal();

            // 3. Reset failed login attempts on success
            user.resetFailedAttempts();
            user.updateLastLogin();
            userRepository.save(user);

            // 4. Generate tokens
            String accessToken = jwtTokenProvider.generateAccessToken(user);
            String refreshToken = jwtTokenProvider.generateRefreshToken(user);

            log.info("User {} logged in successfully", user.getEmail());

            // 5. Return auth response
            return AuthResponse.of(
                    accessToken,
                    refreshToken,
                    jwtProperties.getAccessToken().getExpiration() / 1000, // seconds
                    mapToUserInfo(user)
            );

        } catch (BadCredentialsException e) {
            // Handle failed login attempt
            handleFailedLogin(request.getEmail());
            throw new UnauthorizedException("Invalid email or password");

        } catch (LockedException e) {
            log.warn("Account locked for email: {}", request.getEmail());
            throw new UnauthorizedException(
                "Account is locked. Please try again after 30 minutes."
            );

        } catch (DisabledException e) {
            log.warn("Account not verified for email: {}", request.getEmail());
            throw new UnauthorizedException(
                "Account is not verified. Please verify your email first."
            );

        } catch (AuthenticationException e) {
            log.error("Authentication failed for email: {}", request.getEmail());
            throw new UnauthorizedException("Authentication failed");
        }
    }

    @Override
    @Transactional
    public AuthResponse refreshToken(RefreshTokenRequest request) {
        String refreshToken = request.getRefreshToken();

        // 1. Validate refresh token
        if (!jwtTokenProvider.validateToken(refreshToken)) {
            throw new UnauthorizedException("Invalid refresh token");
        }

        // 2. Check if it's actually a refresh token
        if (!jwtTokenProvider.isRefreshToken(refreshToken)) {
            throw new UnauthorizedException("Token is not a refresh token");
        }

        // 3. Check if token is blacklisted
        if (tokenBlacklistService.isBlacklisted(refreshToken)) {
            throw new UnauthorizedException("Refresh token has been revoked");
        }

        // 4. Extract user info and generate new tokens
        String email = jwtTokenProvider.extractUsername(refreshToken);
        User user = userRepository.findByEmail(email)
                .orElseThrow(() -> new UnauthorizedException("User not found"));

        // 5. Generate new tokens
        String newAccessToken = jwtTokenProvider.generateAccessToken(user);
        String newRefreshToken = jwtTokenProvider.generateRefreshToken(user);

        // 6. Blacklist old refresh token
        tokenBlacklistService.blacklist(refreshToken,
                jwtTokenProvider.getExpirationTime(refreshToken));

        log.info("Token refreshed for user: {}", email);

        return AuthResponse.of(
                newAccessToken,
                newRefreshToken,
                jwtProperties.getAccessToken().getExpiration() / 1000,
                mapToUserInfo(user)
        );
    }

    @Override
    public void logout(String token) {
        if (token != null && token.startsWith("Bearer ")) {
            String jwt = token.substring(7);
            long expirationTime = jwtTokenProvider.getExpirationTime(jwt);
            tokenBlacklistService.blacklist(jwt, expirationTime);
            log.info("User logged out, token blacklisted");
        }
    }

    private void handleFailedLogin(String email) {
        userRepository.findByEmail(email).ifPresent(user -> {
            user.incrementFailedAttempts();
            userRepository.save(user);
            log.warn("Failed login attempt {} for email: {}",
                    user.getFailedLoginAttempts(), email);
        });
    }

    private AuthResponse.UserInfo mapToUserInfo(User user) {
        return AuthResponse.UserInfo.builder()
                .id(user.getId())
                .email(user.getEmail())
                .firstName(user.getFirstName())
                .lastName(user.getLastName())
                .role(user.getRole().name())
                .build();
    }
}
```

---

## 9. Token Blacklist Service

### 9.1 Redis-based Blacklist

```java
package com.ecommerce.service.impl;

import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.data.redis.core.RedisTemplate;
import org.springframework.stereotype.Service;

import java.util.concurrent.TimeUnit;

@Service
@RequiredArgsConstructor
@Slf4j
public class TokenBlacklistService {

    private final RedisTemplate<String, String> redisTemplate;
    private static final String BLACKLIST_PREFIX = "token:blacklist:";

    /**
     * Add token to blacklist
     * @param token JWT token
     * @param expirationTimeMs remaining time until token expires (ms)
     */
    public void blacklist(String token, long expirationTimeMs) {
        String key = BLACKLIST_PREFIX + token;

        // Store in Redis with TTL = remaining token lifetime
        redisTemplate.opsForValue().set(
                key,
                "blacklisted",
                expirationTimeMs,
                TimeUnit.MILLISECONDS
        );

        log.debug("Token blacklisted, expires in {} ms", expirationTimeMs);
    }

    /**
     * Check if token is blacklisted
     */
    public boolean isBlacklisted(String token) {
        String key = BLACKLIST_PREFIX + token;
        return Boolean.TRUE.equals(redisTemplate.hasKey(key));
    }

    /**
     * Remove token from blacklist (if needed)
     */
    public void removeFromBlacklist(String token) {
        String key = BLACKLIST_PREFIX + token;
        redisTemplate.delete(key);
    }
}
```

### 9.2 Update JWT Filter để check blacklist

```java
@Component
@RequiredArgsConstructor
@Slf4j
public class JwtAuthenticationFilter extends OncePerRequestFilter {

    private final JwtTokenProvider jwtTokenProvider;
    private final CustomUserDetailsService userDetailsService;
    private final TokenBlacklistService tokenBlacklistService;

    @Override
    protected void doFilterInternal(...) {
        try {
            String jwt = extractJwtFromRequest(request);

            if (StringUtils.hasText(jwt)) {
                // Check blacklist FIRST
                if (tokenBlacklistService.isBlacklisted(jwt)) {
                    log.warn("Blacklisted token used");
                    filterChain.doFilter(request, response);
                    return;
                }

                if (jwtTokenProvider.validateToken(jwt)) {
                    // ... rest of authentication logic
                }
            }
        } catch (Exception ex) {
            log.error("Cannot set user authentication: {}", ex.getMessage());
        }

        filterChain.doFilter(request, response);
    }
}
```

---

## 10. Auth Controller

```java
package com.ecommerce.controller;

import com.ecommerce.dto.request.LoginRequest;
import com.ecommerce.dto.request.RefreshTokenRequest;
import com.ecommerce.dto.response.ApiResponse;
import com.ecommerce.dto.response.AuthResponse;
import com.ecommerce.service.AuthService;
import io.swagger.v3.oas.annotations.Operation;
import io.swagger.v3.oas.annotations.tags.Tag;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.validation.Valid;
import lombok.RequiredArgsConstructor;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

@RestController
@RequestMapping("/api/v1/auth")
@RequiredArgsConstructor
@Tag(name = "Authentication", description = "Authentication management APIs")
public class AuthController {

    private final AuthService authService;

    @PostMapping("/login")
    @Operation(summary = "User login", description = "Authenticate user and return JWT tokens")
    public ResponseEntity<ApiResponse<AuthResponse>> login(
            @Valid @RequestBody LoginRequest request) {

        AuthResponse response = authService.login(request);
        return ResponseEntity.ok(ApiResponse.success(response, "Login successful"));
    }

    @PostMapping("/refresh")
    @Operation(summary = "Refresh token", description = "Get new access token using refresh token")
    public ResponseEntity<ApiResponse<AuthResponse>> refreshToken(
            @Valid @RequestBody RefreshTokenRequest request) {

        AuthResponse response = authService.refreshToken(request);
        return ResponseEntity.ok(ApiResponse.success(response, "Token refreshed"));
    }

    @PostMapping("/logout")
    @Operation(summary = "User logout", description = "Invalidate current token")
    public ResponseEntity<ApiResponse<Void>> logout(HttpServletRequest request) {
        String authHeader = request.getHeader("Authorization");
        authService.logout(authHeader);
        return ResponseEntity.ok(ApiResponse.success(null, "Logged out successfully"));
    }
}
```

---

## 11. Test API với cURL

```bash
# 1. Login
curl -X POST http://localhost:8080/api/v1/auth/login \
  -H "Content-Type: application/json" \
  -d '{
    "email": "user@example.com",
    "password": "password123"
  }'

# Response:
# {
#   "success": true,
#   "data": {
#     "accessToken": "eyJhbGciOiJIUzI1NiIs...",
#     "refreshToken": "eyJhbGciOiJIUzI1NiIs...",
#     "tokenType": "Bearer",
#     "expiresIn": 900,
#     "user": {
#       "id": 1,
#       "email": "user@example.com",
#       "firstName": "John",
#       "lastName": "Doe",
#       "role": "CUSTOMER"
#     }
#   }
# }

# 2. Access protected endpoint
curl -X GET http://localhost:8080/api/v1/orders \
  -H "Authorization: Bearer eyJhbGciOiJIUzI1NiIs..."

# 3. Refresh token
curl -X POST http://localhost:8080/api/v1/auth/refresh \
  -H "Content-Type: application/json" \
  -d '{
    "refreshToken": "eyJhbGciOiJIUzI1NiIs..."
  }'

# 4. Logout
curl -X POST http://localhost:8080/api/v1/auth/logout \
  -H "Authorization: Bearer eyJhbGciOiJIUzI1NiIs..."
```

---

## 12. Bài tập thực hành

### Bài tập 1: JWT Implementation
Implement đầy đủ JWT authentication:
- [x] JwtTokenProvider với generate/validate
- [x] JwtAuthenticationFilter
- [x] Token blacklist với Redis

### Bài tập 2: Auth Flow
Implement login/logout/refresh flow:
- AuthService với full error handling
- Failed login attempt tracking
- Account locking mechanism

### Bài tập 3: Testing
Test các scenarios:
- Valid login → Get tokens
- Invalid credentials → 401
- Expired token → 401
- Refresh token → New tokens
- Logout → Token blacklisted

---

## 13. Checklist

- [ ] Thêm jjwt dependencies
- [ ] Configure JWT properties (secret, expiration)
- [ ] Implement JwtTokenProvider
- [ ] Tạo JwtAuthenticationFilter
- [ ] Update SecurityConfig với JWT filter
- [ ] Implement AuthService (login, refresh, logout)
- [ ] Tạo TokenBlacklistService với Redis
- [ ] Tạo AuthController
- [ ] Handle security exceptions
- [ ] Test với Postman/cURL

---

## Điều hướng

- [← Day 16: Spring Security Fundamentals](./day-16-spring-security.md)
- [Day 18: User Registration & Login Flow →](./day-18-user-registration.md)
- [Về trang chính](../00-overview.md)
