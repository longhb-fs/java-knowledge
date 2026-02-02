# Day 20: Refresh Token & OAuth2 Social Login

## Mục tiêu học tập
- Implement Refresh Token rotation
- Hiểu OAuth2 flow
- Tích hợp Google/Facebook login
- Handle token revocation

---

## 1. Refresh Token Strategy

```
┌─────────────────────────────────────────────────────────────────┐
│                 REFRESH TOKEN ROTATION                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │ ACCESS TOKEN                                             │   │
│  │ • Short-lived (15 minutes)                              │   │
│  │ • Stored in memory (frontend)                           │   │
│  │ • Contains user claims                                  │   │
│  │ • Sent with every request                               │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                  │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │ REFRESH TOKEN                                            │   │
│  │ • Long-lived (7 days)                                   │   │
│  │ • Stored in HTTP-only cookie or secure storage          │   │
│  │ • Only contains user ID + token type                    │   │
│  │ • Used ONLY to get new access token                     │   │
│  │ • ROTATED on each use (one-time use)                   │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                  │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │ TOKEN ROTATION FLOW                                      │   │
│  │                                                          │   │
│  │  1. Client calls /refresh with refresh_token_v1         │   │
│  │  2. Server validates refresh_token_v1                   │   │
│  │  3. Server issues new access_token + refresh_token_v2   │   │
│  │  4. Server invalidates refresh_token_v1                 │   │
│  │  5. Client stores refresh_token_v2                      │   │
│  │                                                          │   │
│  │  Benefits:                                               │   │
│  │  • If token stolen, attacker can only use once          │   │
│  │  • Original user's next refresh will fail               │   │
│  │  • Indicates possible token theft                       │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## 2. Refresh Token Entity

### 2.1 RefreshToken Entity

```java
package com.ecommerce.domain.entity;

import jakarta.persistence.*;
import lombok.*;

import java.time.LocalDateTime;
import java.util.UUID;

@Entity
@Table(name = "refresh_tokens")
@Getter
@Setter
@NoArgsConstructor
@AllArgsConstructor
@Builder
public class RefreshToken extends BaseEntity {

    @Column(nullable = false, unique = true)
    private String token;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "user_id", nullable = false)
    private User user;

    @Column(name = "expires_at", nullable = false)
    private LocalDateTime expiresAt;

    @Column(name = "device_info")
    private String deviceInfo;

    @Column(name = "ip_address", length = 45)
    private String ipAddress;

    @Column(name = "user_agent")
    private String userAgent;

    @Column(name = "is_revoked")
    private boolean revoked = false;

    @Column(name = "revoked_at")
    private LocalDateTime revokedAt;

    @Column(name = "replaced_by_token")
    private String replacedByToken;

    @Column(name = "family_id")
    private String familyId;  // Token family for rotation tracking

    // Factory method
    public static RefreshToken create(User user, long expirationMs, String deviceInfo,
                                       String ipAddress, String userAgent) {
        String familyId = UUID.randomUUID().toString();
        return create(user, expirationMs, deviceInfo, ipAddress, userAgent, familyId);
    }

    public static RefreshToken create(User user, long expirationMs, String deviceInfo,
                                       String ipAddress, String userAgent, String familyId) {
        return RefreshToken.builder()
                .token(generateToken())
                .user(user)
                .expiresAt(LocalDateTime.now().plusSeconds(expirationMs / 1000))
                .deviceInfo(deviceInfo)
                .ipAddress(ipAddress)
                .userAgent(userAgent)
                .revoked(false)
                .familyId(familyId)
                .build();
    }

    private static String generateToken() {
        return UUID.randomUUID().toString().replace("-", "") +
               UUID.randomUUID().toString().replace("-", "");
    }

    public boolean isExpired() {
        return LocalDateTime.now().isAfter(expiresAt);
    }

    public boolean isValid() {
        return !revoked && !isExpired();
    }

    public void revoke() {
        this.revoked = true;
        this.revokedAt = LocalDateTime.now();
    }

    public void rotateWith(String newToken) {
        this.revoke();
        this.replacedByToken = newToken;
    }
}
```

### 2.2 Flyway Migration

```sql
-- V10__Create_refresh_tokens_table.sql

CREATE TABLE refresh_tokens (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    token VARCHAR(100) NOT NULL UNIQUE,
    user_id BIGINT NOT NULL,
    expires_at DATETIME NOT NULL,
    device_info VARCHAR(255),
    ip_address VARCHAR(45),
    user_agent VARCHAR(500),
    is_revoked BOOLEAN DEFAULT FALSE,
    revoked_at DATETIME,
    replaced_by_token VARCHAR(100),
    family_id VARCHAR(50),
    created_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at DATETIME,
    deleted BOOLEAN DEFAULT FALSE,

    CONSTRAINT fk_refresh_token_user
        FOREIGN KEY (user_id) REFERENCES users(id)
        ON DELETE CASCADE,

    INDEX idx_refresh_token (token),
    INDEX idx_refresh_user (user_id),
    INDEX idx_refresh_family (family_id)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
```

---

## 3. Refresh Token Service

```java
package com.ecommerce.service.impl;

import com.ecommerce.config.properties.JwtProperties;
import com.ecommerce.domain.entity.RefreshToken;
import com.ecommerce.domain.entity.User;
import com.ecommerce.exception.TokenRefreshException;
import com.ecommerce.repository.RefreshTokenRepository;
import com.ecommerce.service.RefreshTokenService;
import jakarta.servlet.http.HttpServletRequest;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.time.LocalDateTime;
import java.util.List;
import java.util.Optional;

@Service
@RequiredArgsConstructor
@Slf4j
public class RefreshTokenServiceImpl implements RefreshTokenService {

    private final RefreshTokenRepository refreshTokenRepository;
    private final JwtProperties jwtProperties;

    @Override
    @Transactional
    public RefreshToken createRefreshToken(User user, HttpServletRequest request) {
        String deviceInfo = extractDeviceInfo(request);
        String ipAddress = extractIpAddress(request);
        String userAgent = request.getHeader("User-Agent");

        RefreshToken refreshToken = RefreshToken.create(
                user,
                jwtProperties.getRefreshToken().getExpiration(),
                deviceInfo,
                ipAddress,
                userAgent
        );

        return refreshTokenRepository.save(refreshToken);
    }

    @Override
    @Transactional
    public RefreshToken rotateRefreshToken(String oldToken, HttpServletRequest request) {
        RefreshToken existingToken = findByToken(oldToken)
                .orElseThrow(() -> new TokenRefreshException("Invalid refresh token"));

        // Validate token
        if (!existingToken.isValid()) {
            // Possible token theft! Revoke entire family
            if (existingToken.isRevoked()) {
                log.warn("Refresh token reuse detected! Revoking entire token family: {}",
                        existingToken.getFamilyId());
                revokeTokenFamily(existingToken.getFamilyId());
            }
            throw new TokenRefreshException("Refresh token is expired or revoked");
        }

        User user = existingToken.getUser();
        String familyId = existingToken.getFamilyId();

        // Create new token with same family ID
        RefreshToken newToken = RefreshToken.create(
                user,
                jwtProperties.getRefreshToken().getExpiration(),
                extractDeviceInfo(request),
                extractIpAddress(request),
                request.getHeader("User-Agent"),
                familyId  // Keep same family
        );

        // Rotate old token
        existingToken.rotateWith(newToken.getToken());
        refreshTokenRepository.save(existingToken);

        return refreshTokenRepository.save(newToken);
    }

    @Override
    @Transactional(readOnly = true)
    public Optional<RefreshToken> findByToken(String token) {
        return refreshTokenRepository.findByToken(token);
    }

    @Override
    @Transactional
    public void revokeToken(String token) {
        refreshTokenRepository.findByToken(token)
                .ifPresent(refreshToken -> {
                    refreshToken.revoke();
                    refreshTokenRepository.save(refreshToken);
                    log.info("Refresh token revoked for user: {}",
                            refreshToken.getUser().getEmail());
                });
    }

    @Override
    @Transactional
    public void revokeAllUserTokens(Long userId) {
        List<RefreshToken> tokens = refreshTokenRepository
                .findAllValidTokensByUser(userId, LocalDateTime.now());

        tokens.forEach(RefreshToken::revoke);
        refreshTokenRepository.saveAll(tokens);

        log.info("Revoked {} refresh tokens for user ID: {}", tokens.size(), userId);
    }

    @Override
    @Transactional
    public void revokeTokenFamily(String familyId) {
        List<RefreshToken> familyTokens = refreshTokenRepository.findByFamilyId(familyId);

        familyTokens.forEach(RefreshToken::revoke);
        refreshTokenRepository.saveAll(familyTokens);

        log.warn("Revoked token family: {} ({} tokens)", familyId, familyTokens.size());
    }

    @Override
    @Transactional(readOnly = true)
    public List<RefreshToken> getActiveSessionsForUser(Long userId) {
        return refreshTokenRepository.findAllValidTokensByUser(userId, LocalDateTime.now());
    }

    @Override
    @Transactional
    public int cleanupExpiredTokens() {
        return refreshTokenRepository.deleteExpiredTokens(LocalDateTime.now());
    }

    private String extractDeviceInfo(HttpServletRequest request) {
        String userAgent = request.getHeader("User-Agent");
        if (userAgent == null) return "Unknown";

        // Simple device detection
        if (userAgent.contains("Mobile")) return "Mobile";
        if (userAgent.contains("Tablet")) return "Tablet";
        return "Desktop";
    }

    private String extractIpAddress(HttpServletRequest request) {
        String xForwardedFor = request.getHeader("X-Forwarded-For");
        if (xForwardedFor != null && !xForwardedFor.isEmpty()) {
            return xForwardedFor.split(",")[0].trim();
        }
        return request.getRemoteAddr();
    }
}
```

---

## 4. Updated AuthService với Token Rotation

```java
package com.ecommerce.service.impl;

import com.ecommerce.domain.entity.RefreshToken;
import com.ecommerce.domain.entity.User;
import com.ecommerce.dto.request.LoginRequest;
import com.ecommerce.dto.request.RefreshTokenRequest;
import com.ecommerce.dto.response.AuthResponse;
import com.ecommerce.exception.TokenRefreshException;
import com.ecommerce.exception.UnauthorizedException;
import com.ecommerce.repository.UserRepository;
import com.ecommerce.security.jwt.JwtTokenProvider;
import com.ecommerce.service.AuthService;
import com.ecommerce.service.RefreshTokenService;
import jakarta.servlet.http.HttpServletRequest;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.security.authentication.AuthenticationManager;
import org.springframework.security.authentication.UsernamePasswordAuthenticationToken;
import org.springframework.security.core.Authentication;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

@Service
@RequiredArgsConstructor
@Slf4j
public class AuthServiceImpl implements AuthService {

    private final AuthenticationManager authenticationManager;
    private final JwtTokenProvider jwtTokenProvider;
    private final RefreshTokenService refreshTokenService;
    private final UserRepository userRepository;

    @Override
    @Transactional
    public AuthResponse login(LoginRequest request, HttpServletRequest httpRequest) {
        log.info("Login attempt for: {}", request.getEmail());

        // Authenticate
        Authentication authentication = authenticationManager.authenticate(
                new UsernamePasswordAuthenticationToken(
                        request.getEmail(),
                        request.getPassword()
                )
        );

        User user = (User) authentication.getPrincipal();

        // Update login info
        user.resetFailedAttempts();
        user.updateLastLogin();
        userRepository.save(user);

        // Generate tokens
        String accessToken = jwtTokenProvider.generateAccessToken(user);
        RefreshToken refreshToken = refreshTokenService.createRefreshToken(user, httpRequest);

        log.info("User {} logged in successfully", user.getEmail());

        return buildAuthResponse(user, accessToken, refreshToken.getToken());
    }

    @Override
    @Transactional
    public AuthResponse refreshToken(RefreshTokenRequest request, HttpServletRequest httpRequest) {
        String requestToken = request.getRefreshToken();

        try {
            // Rotate refresh token
            RefreshToken newRefreshToken = refreshTokenService.rotateRefreshToken(
                    requestToken, httpRequest);

            User user = newRefreshToken.getUser();

            // Generate new access token
            String accessToken = jwtTokenProvider.generateAccessToken(user);

            log.info("Token refreshed for user: {}", user.getEmail());

            return buildAuthResponse(user, accessToken, newRefreshToken.getToken());

        } catch (TokenRefreshException e) {
            log.warn("Token refresh failed: {}", e.getMessage());
            throw new UnauthorizedException(e.getMessage());
        }
    }

    @Override
    @Transactional
    public void logout(String refreshToken) {
        if (refreshToken != null && !refreshToken.isEmpty()) {
            refreshTokenService.revokeToken(refreshToken);
        }
        log.info("User logged out");
    }

    @Override
    @Transactional
    public void logoutAllDevices(Long userId) {
        refreshTokenService.revokeAllUserTokens(userId);
        log.info("User {} logged out from all devices", userId);
    }

    private AuthResponse buildAuthResponse(User user, String accessToken, String refreshToken) {
        return AuthResponse.builder()
                .accessToken(accessToken)
                .refreshToken(refreshToken)
                .tokenType("Bearer")
                .expiresIn(jwtTokenProvider.getAccessTokenExpiration())
                .user(AuthResponse.UserInfo.builder()
                        .id(user.getId())
                        .email(user.getEmail())
                        .firstName(user.getFirstName())
                        .lastName(user.getLastName())
                        .role(user.getRoles().stream()
                                .findFirst()
                                .map(r -> r.getName())
                                .orElse("CUSTOMER"))
                        .build())
                .build();
    }
}
```

---

## 5. OAuth2 Social Login

### 5.1 OAuth2 Flow

```
┌─────────────────────────────────────────────────────────────────┐
│                    OAUTH2 AUTHORIZATION CODE FLOW                │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌────────┐      ┌────────────┐      ┌────────────────────┐    │
│  │  User  │      │ Our Server │      │ OAuth Provider     │    │
│  │(Browser│      │  (Backend) │      │ (Google/Facebook)  │    │
│  └───┬────┘      └─────┬──────┘      └─────────┬──────────┘    │
│      │                 │                       │                │
│      │ 1. Click        │                       │                │
│      │ "Login with     │                       │                │
│      │  Google"        │                       │                │
│      │────────────────▶│                       │                │
│      │                 │                       │                │
│      │ 2. Redirect to  │                       │                │
│      │    Google       │                       │                │
│      │◀────────────────│                       │                │
│      │                 │                       │                │
│      │ 3. Login at Google                      │                │
│      │─────────────────────────────────────────▶                │
│      │                 │                       │                │
│      │ 4. Authorize app                        │                │
│      │◀─────────────────────────────────────────                │
│      │                 │                       │                │
│      │ 5. Redirect with authorization_code     │                │
│      │─────────────────▶                       │                │
│      │                 │                       │                │
│      │                 │ 6. Exchange code      │                │
│      │                 │    for access_token   │                │
│      │                 │───────────────────────▶                │
│      │                 │                       │                │
│      │                 │ 7. Return access_token│                │
│      │                 │◀───────────────────────                │
│      │                 │                       │                │
│      │                 │ 8. Get user info      │                │
│      │                 │───────────────────────▶                │
│      │                 │                       │                │
│      │                 │ 9. Return profile     │                │
│      │                 │◀───────────────────────                │
│      │                 │                       │                │
│      │ 10. Create/     │                       │                │
│      │     Login user, │                       │                │
│      │     return JWT  │                       │                │
│      │◀────────────────│                       │                │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 5.2 OAuth2 Dependencies

```xml
<dependencies>
    <!-- OAuth2 Client -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-oauth2-client</artifactId>
    </dependency>
</dependencies>
```

### 5.3 application.yml

```yaml
spring:
  security:
    oauth2:
      client:
        registration:
          google:
            client-id: ${GOOGLE_CLIENT_ID}
            client-secret: ${GOOGLE_CLIENT_SECRET}
            scope:
              - email
              - profile
            redirect-uri: "{baseUrl}/api/v1/auth/oauth2/callback/{registrationId}"

          facebook:
            client-id: ${FACEBOOK_CLIENT_ID}
            client-secret: ${FACEBOOK_CLIENT_SECRET}
            scope:
              - email
              - public_profile
            redirect-uri: "{baseUrl}/api/v1/auth/oauth2/callback/{registrationId}"

        provider:
          google:
            authorization-uri: https://accounts.google.com/o/oauth2/v2/auth
            token-uri: https://oauth2.googleapis.com/token
            user-info-uri: https://www.googleapis.com/oauth2/v3/userinfo
            user-name-attribute: sub

          facebook:
            authorization-uri: https://www.facebook.com/v18.0/dialog/oauth
            token-uri: https://graph.facebook.com/v18.0/oauth/access_token
            user-info-uri: https://graph.facebook.com/me?fields=id,name,email,picture
            user-name-attribute: id

app:
  oauth2:
    # URL để redirect sau khi OAuth thành công
    authorized-redirect-uris:
      - http://localhost:3000/oauth2/redirect
      - https://ecommerce.example.com/oauth2/redirect
```

---

## 6. OAuth2 Entities

### 6.1 OAuth2UserInfo Classes

```java
package com.ecommerce.security.oauth2;

import java.util.Map;

public abstract class OAuth2UserInfo {

    protected Map<String, Object> attributes;

    public OAuth2UserInfo(Map<String, Object> attributes) {
        this.attributes = attributes;
    }

    public Map<String, Object> getAttributes() {
        return attributes;
    }

    public abstract String getId();
    public abstract String getName();
    public abstract String getEmail();
    public abstract String getImageUrl();
}
```

```java
package com.ecommerce.security.oauth2;

import java.util.Map;

public class GoogleOAuth2UserInfo extends OAuth2UserInfo {

    public GoogleOAuth2UserInfo(Map<String, Object> attributes) {
        super(attributes);
    }

    @Override
    public String getId() {
        return (String) attributes.get("sub");
    }

    @Override
    public String getName() {
        return (String) attributes.get("name");
    }

    @Override
    public String getEmail() {
        return (String) attributes.get("email");
    }

    @Override
    public String getImageUrl() {
        return (String) attributes.get("picture");
    }
}
```

```java
package com.ecommerce.security.oauth2;

import java.util.Map;

public class FacebookOAuth2UserInfo extends OAuth2UserInfo {

    public FacebookOAuth2UserInfo(Map<String, Object> attributes) {
        super(attributes);
    }

    @Override
    public String getId() {
        return (String) attributes.get("id");
    }

    @Override
    public String getName() {
        return (String) attributes.get("name");
    }

    @Override
    public String getEmail() {
        return (String) attributes.get("email");
    }

    @Override
    public String getImageUrl() {
        if (attributes.containsKey("picture")) {
            Map<String, Object> pictureObj = (Map<String, Object>) attributes.get("picture");
            if (pictureObj.containsKey("data")) {
                Map<String, Object> dataObj = (Map<String, Object>) pictureObj.get("data");
                if (dataObj.containsKey("url")) {
                    return (String) dataObj.get("url");
                }
            }
        }
        return null;
    }
}
```

```java
package com.ecommerce.security.oauth2;

import com.ecommerce.domain.enums.AuthProvider;
import com.ecommerce.exception.OAuth2AuthenticationProcessingException;

import java.util.Map;

public class OAuth2UserInfoFactory {

    public static OAuth2UserInfo getOAuth2UserInfo(String registrationId,
                                                    Map<String, Object> attributes) {
        if (registrationId.equalsIgnoreCase(AuthProvider.GOOGLE.name())) {
            return new GoogleOAuth2UserInfo(attributes);
        } else if (registrationId.equalsIgnoreCase(AuthProvider.FACEBOOK.name())) {
            return new FacebookOAuth2UserInfo(attributes);
        } else {
            throw new OAuth2AuthenticationProcessingException(
                    "Login with " + registrationId + " is not supported");
        }
    }
}
```

### 6.2 AuthProvider Enum

```java
package com.ecommerce.domain.enums;

public enum AuthProvider {
    LOCAL,
    GOOGLE,
    FACEBOOK,
    GITHUB
}
```

### 6.3 Update User Entity

```java
@Entity
@Table(name = "users")
public class User extends BaseEntity implements UserDetails {

    // ... existing fields ...

    @Enumerated(EnumType.STRING)
    @Column(name = "auth_provider")
    private AuthProvider authProvider = AuthProvider.LOCAL;

    @Column(name = "provider_id")
    private String providerId;

    @Column(name = "avatar_url")
    private String avatarUrl;

    // ... rest of entity ...
}
```

---

## 7. OAuth2 Service

### 7.1 CustomOAuth2UserService

```java
package com.ecommerce.security.oauth2;

import com.ecommerce.domain.entity.Role;
import com.ecommerce.domain.entity.User;
import com.ecommerce.domain.enums.AuthProvider;
import com.ecommerce.exception.OAuth2AuthenticationProcessingException;
import com.ecommerce.repository.RoleRepository;
import com.ecommerce.repository.UserRepository;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.security.authentication.InternalAuthenticationServiceException;
import org.springframework.security.oauth2.client.userinfo.DefaultOAuth2UserService;
import org.springframework.security.oauth2.client.userinfo.OAuth2UserRequest;
import org.springframework.security.oauth2.core.OAuth2AuthenticationException;
import org.springframework.security.oauth2.core.user.OAuth2User;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;
import org.springframework.util.StringUtils;

import java.util.Optional;
import java.util.Set;

@Service
@RequiredArgsConstructor
@Slf4j
public class CustomOAuth2UserService extends DefaultOAuth2UserService {

    private final UserRepository userRepository;
    private final RoleRepository roleRepository;

    @Override
    @Transactional
    public OAuth2User loadUser(OAuth2UserRequest oAuth2UserRequest)
            throws OAuth2AuthenticationException {

        OAuth2User oAuth2User = super.loadUser(oAuth2UserRequest);

        try {
            return processOAuth2User(oAuth2UserRequest, oAuth2User);
        } catch (Exception ex) {
            log.error("OAuth2 authentication failed", ex);
            throw new InternalAuthenticationServiceException(ex.getMessage(), ex);
        }
    }

    private OAuth2User processOAuth2User(OAuth2UserRequest request, OAuth2User oAuth2User) {
        String registrationId = request.getClientRegistration().getRegistrationId();

        OAuth2UserInfo oAuth2UserInfo = OAuth2UserInfoFactory.getOAuth2UserInfo(
                registrationId,
                oAuth2User.getAttributes()
        );

        // Validate email
        if (!StringUtils.hasText(oAuth2UserInfo.getEmail())) {
            throw new OAuth2AuthenticationProcessingException(
                    "Email not found from OAuth2 provider");
        }

        Optional<User> userOptional = userRepository.findByEmail(oAuth2UserInfo.getEmail());
        User user;

        if (userOptional.isPresent()) {
            user = userOptional.get();

            // Check if user signed up with different provider
            if (!user.getAuthProvider().equals(AuthProvider.valueOf(registrationId.toUpperCase()))) {
                throw new OAuth2AuthenticationProcessingException(
                        "You're signed up with " + user.getAuthProvider() +
                        ". Please use your " + user.getAuthProvider() + " account to login.");
            }

            user = updateExistingUser(user, oAuth2UserInfo);
        } else {
            user = registerNewUser(registrationId, oAuth2UserInfo);
        }

        return new OAuth2UserPrincipal(user, oAuth2User.getAttributes());
    }

    private User registerNewUser(String registrationId, OAuth2UserInfo userInfo) {
        log.info("Registering new OAuth2 user: {}", userInfo.getEmail());

        // Get default role
        Role customerRole = roleRepository.findByName("CUSTOMER")
                .orElseThrow(() -> new RuntimeException("Default role not found"));

        // Parse name
        String[] nameParts = userInfo.getName().split(" ", 2);
        String firstName = nameParts[0];
        String lastName = nameParts.length > 1 ? nameParts[1] : "";

        User user = User.builder()
                .email(userInfo.getEmail())
                .firstName(firstName)
                .lastName(lastName)
                .avatarUrl(userInfo.getImageUrl())
                .authProvider(AuthProvider.valueOf(registrationId.toUpperCase()))
                .providerId(userInfo.getId())
                .emailVerified(true)  // OAuth emails are verified
                .roles(Set.of(customerRole))
                .build();

        return userRepository.save(user);
    }

    private User updateExistingUser(User user, OAuth2UserInfo userInfo) {
        log.debug("Updating existing OAuth2 user: {}", user.getEmail());

        user.setAvatarUrl(userInfo.getImageUrl());

        return userRepository.save(user);
    }
}
```

### 7.2 OAuth2UserPrincipal

```java
package com.ecommerce.security.oauth2;

import com.ecommerce.domain.entity.User;
import lombok.Getter;
import org.springframework.security.core.GrantedAuthority;
import org.springframework.security.oauth2.core.user.OAuth2User;

import java.util.Collection;
import java.util.Map;

@Getter
public class OAuth2UserPrincipal implements OAuth2User {

    private final User user;
    private final Map<String, Object> attributes;

    public OAuth2UserPrincipal(User user, Map<String, Object> attributes) {
        this.user = user;
        this.attributes = attributes;
    }

    @Override
    public Collection<? extends GrantedAuthority> getAuthorities() {
        return user.getAuthorities();
    }

    @Override
    public String getName() {
        return String.valueOf(user.getId());
    }

    @Override
    public Map<String, Object> getAttributes() {
        return attributes;
    }
}
```

---

## 8. OAuth2 Security Configuration

### 8.1 OAuth2 Success Handler

```java
package com.ecommerce.security.oauth2;

import com.ecommerce.config.properties.AppProperties;
import com.ecommerce.domain.entity.RefreshToken;
import com.ecommerce.domain.entity.User;
import com.ecommerce.security.jwt.JwtTokenProvider;
import com.ecommerce.service.RefreshTokenService;
import com.ecommerce.util.CookieUtils;
import jakarta.servlet.http.Cookie;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.security.core.Authentication;
import org.springframework.security.web.authentication.SimpleUrlAuthenticationSuccessHandler;
import org.springframework.stereotype.Component;
import org.springframework.web.util.UriComponentsBuilder;

import java.io.IOException;
import java.net.URI;
import java.util.Optional;

@Component
@RequiredArgsConstructor
@Slf4j
public class OAuth2AuthenticationSuccessHandler extends SimpleUrlAuthenticationSuccessHandler {

    private final JwtTokenProvider jwtTokenProvider;
    private final RefreshTokenService refreshTokenService;
    private final AppProperties appProperties;
    private final HttpCookieOAuth2AuthorizationRequestRepository requestRepository;

    public static final String REDIRECT_URI_PARAM_COOKIE_NAME = "redirect_uri";

    @Override
    public void onAuthenticationSuccess(HttpServletRequest request,
                                        HttpServletResponse response,
                                        Authentication authentication) throws IOException {

        String targetUrl = determineTargetUrl(request, response, authentication);

        if (response.isCommitted()) {
            log.debug("Response has already been committed");
            return;
        }

        clearAuthenticationAttributes(request, response);
        getRedirectStrategy().sendRedirect(request, response, targetUrl);
    }

    protected String determineTargetUrl(HttpServletRequest request,
                                        HttpServletResponse response,
                                        Authentication authentication) {

        Optional<String> redirectUri = CookieUtils.getCookie(request, REDIRECT_URI_PARAM_COOKIE_NAME)
                .map(Cookie::getValue);

        if (redirectUri.isPresent() && !isAuthorizedRedirectUri(redirectUri.get())) {
            throw new RuntimeException(
                    "Unauthorized Redirect URI: " + redirectUri.get());
        }

        String targetUrl = redirectUri.orElse(getDefaultTargetUrl());

        // Get user from authentication
        OAuth2UserPrincipal principal = (OAuth2UserPrincipal) authentication.getPrincipal();
        User user = principal.getUser();

        // Generate tokens
        String accessToken = jwtTokenProvider.generateAccessToken(user);
        RefreshToken refreshToken = refreshTokenService.createRefreshToken(user, request);

        return UriComponentsBuilder.fromUriString(targetUrl)
                .queryParam("token", accessToken)
                .queryParam("refresh_token", refreshToken.getToken())
                .build().toUriString();
    }

    protected void clearAuthenticationAttributes(HttpServletRequest request,
                                                  HttpServletResponse response) {
        super.clearAuthenticationAttributes(request);
        requestRepository.removeAuthorizationRequestCookies(request, response);
    }

    private boolean isAuthorizedRedirectUri(String uri) {
        URI clientRedirectUri = URI.create(uri);

        return appProperties.getOauth2().getAuthorizedRedirectUris()
                .stream()
                .anyMatch(authorizedUri -> {
                    URI authorizedURI = URI.create(authorizedUri);
                    return authorizedURI.getHost().equalsIgnoreCase(clientRedirectUri.getHost())
                            && authorizedURI.getPort() == clientRedirectUri.getPort();
                });
    }
}
```

### 8.2 OAuth2 Failure Handler

```java
package com.ecommerce.security.oauth2;

import com.ecommerce.util.CookieUtils;
import jakarta.servlet.http.Cookie;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.security.core.AuthenticationException;
import org.springframework.security.web.authentication.SimpleUrlAuthenticationFailureHandler;
import org.springframework.stereotype.Component;
import org.springframework.web.util.UriComponentsBuilder;

import java.io.IOException;

import static com.ecommerce.security.oauth2.OAuth2AuthenticationSuccessHandler.REDIRECT_URI_PARAM_COOKIE_NAME;

@Component
@RequiredArgsConstructor
@Slf4j
public class OAuth2AuthenticationFailureHandler extends SimpleUrlAuthenticationFailureHandler {

    private final HttpCookieOAuth2AuthorizationRequestRepository requestRepository;

    @Override
    public void onAuthenticationFailure(HttpServletRequest request,
                                        HttpServletResponse response,
                                        AuthenticationException exception) throws IOException {

        String targetUrl = CookieUtils.getCookie(request, REDIRECT_URI_PARAM_COOKIE_NAME)
                .map(Cookie::getValue)
                .orElse("/");

        targetUrl = UriComponentsBuilder.fromUriString(targetUrl)
                .queryParam("error", exception.getLocalizedMessage())
                .build().toUriString();

        requestRepository.removeAuthorizationRequestCookies(request, response);

        log.error("OAuth2 authentication failed: {}", exception.getMessage());

        getRedirectStrategy().sendRedirect(request, response, targetUrl);
    }
}
```

### 8.3 Update SecurityConfig

```java
@Configuration
@EnableWebSecurity
@EnableMethodSecurity
@RequiredArgsConstructor
public class SecurityConfig {

    private final CustomOAuth2UserService customOAuth2UserService;
    private final OAuth2AuthenticationSuccessHandler oAuth2SuccessHandler;
    private final OAuth2AuthenticationFailureHandler oAuth2FailureHandler;
    private final HttpCookieOAuth2AuthorizationRequestRepository cookieAuthorizationRequestRepository;
    // ... other dependencies

    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
        http
            .cors(cors -> cors.configurationSource(corsConfigurationSource()))
            .csrf(AbstractHttpConfigurer::disable)
            .sessionManagement(session ->
                session.sessionCreationPolicy(SessionCreationPolicy.STATELESS)
            )
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/api/v1/auth/**", "/oauth2/**").permitAll()
                // ... other rules
                .anyRequest().authenticated()
            )
            .oauth2Login(oauth2 -> oauth2
                .authorizationEndpoint(authorization -> authorization
                    .baseUri("/oauth2/authorize")
                    .authorizationRequestRepository(cookieAuthorizationRequestRepository)
                )
                .redirectionEndpoint(redirection -> redirection
                    .baseUri("/api/v1/auth/oauth2/callback/*")
                )
                .userInfoEndpoint(userInfo -> userInfo
                    .userService(customOAuth2UserService)
                )
                .successHandler(oAuth2SuccessHandler)
                .failureHandler(oAuth2FailureHandler)
            )
            .exceptionHandling(ex -> ex
                .authenticationEntryPoint(authEntryPoint)
                .accessDeniedHandler(accessDeniedHandler)
            )
            .addFilterBefore(jwtAuthFilter, UsernamePasswordAuthenticationFilter.class);

        return http.build();
    }
}
```

---

## 9. Session Management API

### 9.1 SessionController

```java
package com.ecommerce.controller;

import com.ecommerce.domain.entity.RefreshToken;
import com.ecommerce.dto.response.ApiResponse;
import com.ecommerce.dto.response.SessionDto;
import com.ecommerce.service.RefreshTokenService;
import com.ecommerce.util.SecurityUtils;
import io.swagger.v3.oas.annotations.Operation;
import io.swagger.v3.oas.annotations.security.SecurityRequirement;
import io.swagger.v3.oas.annotations.tags.Tag;
import lombok.RequiredArgsConstructor;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

import java.util.List;
import java.util.stream.Collectors;

@RestController
@RequestMapping("/api/v1/users/me/sessions")
@RequiredArgsConstructor
@Tag(name = "Session Management", description = "User session management APIs")
@SecurityRequirement(name = "bearerAuth")
public class SessionController {

    private final RefreshTokenService refreshTokenService;

    @GetMapping
    @Operation(summary = "Get all active sessions")
    public ResponseEntity<ApiResponse<List<SessionDto>>> getActiveSessions() {
        Long userId = SecurityUtils.getCurrentUserId()
                .orElseThrow(() -> new RuntimeException("User not authenticated"));

        List<SessionDto> sessions = refreshTokenService.getActiveSessionsForUser(userId)
                .stream()
                .map(this::toSessionDto)
                .collect(Collectors.toList());

        return ResponseEntity.ok(ApiResponse.success(sessions));
    }

    @DeleteMapping("/{tokenId}")
    @Operation(summary = "Revoke a specific session")
    public ResponseEntity<ApiResponse<Void>> revokeSession(@PathVariable Long tokenId) {
        // Verify ownership and revoke
        refreshTokenService.revokeTokenById(tokenId, SecurityUtils.getCurrentUserId().orElse(null));
        return ResponseEntity.ok(ApiResponse.success(null, "Session revoked"));
    }

    @DeleteMapping
    @Operation(summary = "Revoke all other sessions")
    public ResponseEntity<ApiResponse<Void>> revokeAllOtherSessions(
            @RequestHeader("Authorization") String currentToken) {

        Long userId = SecurityUtils.getCurrentUserId()
                .orElseThrow(() -> new RuntimeException("User not authenticated"));

        refreshTokenService.revokeAllUserTokensExcept(userId, currentToken);
        return ResponseEntity.ok(ApiResponse.success(null, "All other sessions revoked"));
    }

    private SessionDto toSessionDto(RefreshToken token) {
        return SessionDto.builder()
                .id(token.getId())
                .deviceInfo(token.getDeviceInfo())
                .ipAddress(token.getIpAddress())
                .createdAt(token.getCreatedAt())
                .lastUsed(token.getUpdatedAt())
                .isCurrent(false)  // Will be set by frontend
                .build();
    }
}
```

---

## 10. Frontend Integration

### 10.1 OAuth2 Login Flow (Frontend)

```javascript
// 1. Initiate OAuth2 login
function loginWithGoogle() {
  const redirectUri = encodeURIComponent(window.location.origin + '/oauth2/redirect');
  window.location.href = `${API_BASE_URL}/oauth2/authorize/google?redirect_uri=${redirectUri}`;
}

function loginWithFacebook() {
  const redirectUri = encodeURIComponent(window.location.origin + '/oauth2/redirect');
  window.location.href = `${API_BASE_URL}/oauth2/authorize/facebook?redirect_uri=${redirectUri}`;
}

// 2. Handle OAuth2 callback (on /oauth2/redirect page)
function handleOAuth2Callback() {
  const urlParams = new URLSearchParams(window.location.search);
  const token = urlParams.get('token');
  const refreshToken = urlParams.get('refresh_token');
  const error = urlParams.get('error');

  if (error) {
    console.error('OAuth2 error:', error);
    // Show error to user
    return;
  }

  if (token && refreshToken) {
    // Store tokens
    localStorage.setItem('accessToken', token);
    localStorage.setItem('refreshToken', refreshToken);

    // Redirect to home/dashboard
    window.location.href = '/dashboard';
  }
}

// 3. Refresh token rotation
async function refreshAccessToken() {
  const refreshToken = localStorage.getItem('refreshToken');

  try {
    const response = await fetch(`${API_BASE_URL}/api/v1/auth/refresh`, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ refreshToken })
    });

    if (!response.ok) {
      // Refresh failed - redirect to login
      localStorage.clear();
      window.location.href = '/login';
      return null;
    }

    const data = await response.json();

    // Update both tokens (rotation)
    localStorage.setItem('accessToken', data.data.accessToken);
    localStorage.setItem('refreshToken', data.data.refreshToken);

    return data.data.accessToken;
  } catch (error) {
    console.error('Token refresh failed:', error);
    localStorage.clear();
    window.location.href = '/login';
    return null;
  }
}
```

---

## 11. Bài tập thực hành

### Bài tập 1: Refresh Token Rotation
- [x] Create RefreshToken entity
- [x] Implement token rotation
- [x] Handle token family revocation

### Bài tập 2: OAuth2 Integration
- [ ] Setup Google OAuth2
- [ ] Setup Facebook OAuth2
- [ ] Create OAuth2UserPrincipal

### Bài tập 3: Session Management
- [x] List active sessions
- [x] Revoke specific session
- [x] Revoke all other sessions

---

## 12. Checklist

- [ ] Create RefreshToken entity và migration
- [ ] Implement RefreshTokenService với rotation
- [ ] Update AuthService để dùng RefreshToken entity
- [ ] Configure OAuth2 providers (Google, Facebook)
- [ ] Create CustomOAuth2UserService
- [ ] Implement OAuth2 success/failure handlers
- [ ] Update SecurityConfig cho OAuth2
- [ ] Create Session management API
- [ ] Test refresh token rotation
- [ ] Test OAuth2 login flow

---

## Điều hướng

- [← Day 19: Role-Based Authorization](./day-19-role-authorization.md)
- [Day 21: Shopping Cart & Sessions →](../week-05/day-21-shopping-cart.md)
- [Về trang chính](../00-overview.md)
