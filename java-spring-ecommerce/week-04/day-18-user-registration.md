# Day 18: User Registration & Login Flow

## Mục tiêu học tập
- Implement User Registration với validation
- Tạo Email Verification flow
- Implement Password Reset functionality
- Xử lý các edge cases trong authentication

---

## 1. Registration Flow Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                    USER REGISTRATION FLOW                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌───────────────────┐                                          │
│  │   User submits    │                                          │
│  │   registration    │                                          │
│  │   form            │                                          │
│  └─────────┬─────────┘                                          │
│            │                                                     │
│            ▼                                                     │
│  ┌───────────────────┐     ┌───────────────────┐               │
│  │ Validate input    │────▶│ Email exists?     │               │
│  │ - Email format    │     │                   │               │
│  │ - Password rules  │     └─────────┬─────────┘               │
│  └───────────────────┘               │                          │
│                           ┌──────────┴──────────┐               │
│                          YES                    NO               │
│                           │                     │                │
│                           ▼                     ▼                │
│            ┌───────────────────┐  ┌───────────────────┐        │
│            │ Return error:     │  │ Create User       │        │
│            │ "Email exists"    │  │ (emailVerified    │        │
│            └───────────────────┘  │  = false)         │        │
│                                   └─────────┬─────────┘        │
│                                             │                   │
│                                             ▼                   │
│                                   ┌───────────────────┐        │
│                                   │ Generate verify   │        │
│                                   │ token & send      │        │
│                                   │ email             │        │
│                                   └─────────┬─────────┘        │
│                                             │                   │
│                                             ▼                   │
│                                   ┌───────────────────┐        │
│                                   │ User clicks link  │        │
│                                   │ in email          │        │
│                                   └─────────┬─────────┘        │
│                                             │                   │
│                                             ▼                   │
│                                   ┌───────────────────┐        │
│                                   │ Verify token &    │        │
│                                   │ activate account  │        │
│                                   └─────────┬─────────┘        │
│                                             │                   │
│                                             ▼                   │
│                                   ┌───────────────────┐        │
│                                   │ User can login    │        │
│                                   └───────────────────┘        │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## 2. Verification Token Entity

### 2.1 VerificationToken Entity

```java
package com.ecommerce.domain.entity;

import com.ecommerce.domain.enums.TokenType;
import jakarta.persistence.*;
import lombok.*;

import java.time.LocalDateTime;
import java.util.UUID;

@Entity
@Table(name = "verification_tokens")
@Getter
@Setter
@NoArgsConstructor
@AllArgsConstructor
@Builder
public class VerificationToken extends BaseEntity {

    @Column(nullable = false, unique = true)
    private String token;

    @Enumerated(EnumType.STRING)
    @Column(nullable = false)
    private TokenType tokenType;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "user_id", nullable = false)
    private User user;

    @Column(name = "expires_at", nullable = false)
    private LocalDateTime expiresAt;

    @Column(name = "used_at")
    private LocalDateTime usedAt;

    @Column(name = "is_used")
    private boolean used = false;

    // ========== Factory Methods ==========

    public static VerificationToken createEmailVerification(User user, int expirationHours) {
        return VerificationToken.builder()
                .token(generateToken())
                .tokenType(TokenType.EMAIL_VERIFICATION)
                .user(user)
                .expiresAt(LocalDateTime.now().plusHours(expirationHours))
                .used(false)
                .build();
    }

    public static VerificationToken createPasswordReset(User user, int expirationHours) {
        return VerificationToken.builder()
                .token(generateToken())
                .tokenType(TokenType.PASSWORD_RESET)
                .user(user)
                .expiresAt(LocalDateTime.now().plusHours(expirationHours))
                .used(false)
                .build();
    }

    // ========== Helper Methods ==========

    private static String generateToken() {
        return UUID.randomUUID().toString().replace("-", "");
    }

    public boolean isExpired() {
        return LocalDateTime.now().isAfter(expiresAt);
    }

    public boolean isValid() {
        return !used && !isExpired();
    }

    public void markAsUsed() {
        this.used = true;
        this.usedAt = LocalDateTime.now();
    }
}
```

### 2.2 TokenType Enum

```java
package com.ecommerce.domain.enums;

public enum TokenType {
    EMAIL_VERIFICATION,
    PASSWORD_RESET,
    PHONE_VERIFICATION
}
```

### 2.3 Flyway Migration

```sql
-- V7__Create_verification_tokens_table.sql

CREATE TABLE verification_tokens (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    token VARCHAR(100) NOT NULL UNIQUE,
    token_type VARCHAR(30) NOT NULL,
    user_id BIGINT NOT NULL,
    expires_at DATETIME NOT NULL,
    used_at DATETIME,
    is_used BOOLEAN DEFAULT FALSE,
    created_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at DATETIME,
    created_by BIGINT,
    updated_by BIGINT,
    deleted BOOLEAN DEFAULT FALSE,

    CONSTRAINT fk_verification_token_user
        FOREIGN KEY (user_id) REFERENCES users(id)
        ON DELETE CASCADE,

    INDEX idx_token (token),
    INDEX idx_user_token_type (user_id, token_type)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
```

---

## 3. Registration DTOs

### 3.1 RegisterRequest

```java
package com.ecommerce.dto.request;

import com.ecommerce.validation.PasswordMatch;
import com.ecommerce.validation.UniqueEmail;
import jakarta.validation.constraints.*;
import lombok.Data;

@Data
@PasswordMatch(
    password = "password",
    confirmPassword = "confirmPassword",
    message = "Passwords do not match"
)
public class RegisterRequest {

    @NotBlank(message = "Email is required")
    @Email(message = "Invalid email format")
    @Size(max = 100, message = "Email must be less than 100 characters")
    @UniqueEmail(message = "Email already registered")
    private String email;

    @NotBlank(message = "Password is required")
    @Size(min = 8, max = 100, message = "Password must be between 8 and 100 characters")
    @Pattern(
        regexp = "^(?=.*[a-z])(?=.*[A-Z])(?=.*\\d)(?=.*[@$!%*?&])[A-Za-z\\d@$!%*?&]+$",
        message = "Password must contain at least one uppercase, one lowercase, one digit and one special character"
    )
    private String password;

    @NotBlank(message = "Confirm password is required")
    private String confirmPassword;

    @NotBlank(message = "First name is required")
    @Size(min = 2, max = 50, message = "First name must be between 2 and 50 characters")
    private String firstName;

    @NotBlank(message = "Last name is required")
    @Size(min = 2, max = 50, message = "Last name must be between 2 and 50 characters")
    private String lastName;

    @Pattern(regexp = "^\\+?[0-9]{10,15}$", message = "Invalid phone number")
    private String phone;

    @AssertTrue(message = "You must accept the terms and conditions")
    private boolean acceptTerms;
}
```

### 3.2 Custom Validators

```java
package com.ecommerce.validation;

import jakarta.validation.Constraint;
import jakarta.validation.Payload;

import java.lang.annotation.*;

@Target({ElementType.FIELD})
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Constraint(validatedBy = UniqueEmailValidator.class)
public @interface UniqueEmail {
    String message() default "Email already exists";
    Class<?>[] groups() default {};
    Class<? extends Payload>[] payload() default {};
}
```

```java
package com.ecommerce.validation;

import com.ecommerce.repository.UserRepository;
import jakarta.validation.ConstraintValidator;
import jakarta.validation.ConstraintValidatorContext;
import lombok.RequiredArgsConstructor;
import org.springframework.stereotype.Component;

@Component
@RequiredArgsConstructor
public class UniqueEmailValidator implements ConstraintValidator<UniqueEmail, String> {

    private final UserRepository userRepository;

    @Override
    public boolean isValid(String email, ConstraintValidatorContext context) {
        if (email == null || email.isBlank()) {
            return true; // Let @NotBlank handle this
        }
        return !userRepository.existsByEmail(email);
    }
}
```

```java
package com.ecommerce.validation;

import jakarta.validation.Constraint;
import jakarta.validation.Payload;

import java.lang.annotation.*;

@Target({ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Constraint(validatedBy = PasswordMatchValidator.class)
public @interface PasswordMatch {
    String message() default "Passwords do not match";
    String password();
    String confirmPassword();
    Class<?>[] groups() default {};
    Class<? extends Payload>[] payload() default {};
}
```

```java
package com.ecommerce.validation;

import jakarta.validation.ConstraintValidator;
import jakarta.validation.ConstraintValidatorContext;
import org.springframework.beans.BeanWrapperImpl;

public class PasswordMatchValidator implements ConstraintValidator<PasswordMatch, Object> {

    private String passwordField;
    private String confirmPasswordField;

    @Override
    public void initialize(PasswordMatch constraintAnnotation) {
        this.passwordField = constraintAnnotation.password();
        this.confirmPasswordField = constraintAnnotation.confirmPassword();
    }

    @Override
    public boolean isValid(Object value, ConstraintValidatorContext context) {
        Object password = new BeanWrapperImpl(value).getPropertyValue(passwordField);
        Object confirmPassword = new BeanWrapperImpl(value).getPropertyValue(confirmPasswordField);

        if (password == null || confirmPassword == null) {
            return true; // Let other validators handle null
        }

        boolean isValid = password.equals(confirmPassword);

        if (!isValid) {
            // Add error to confirmPassword field instead of class level
            context.disableDefaultConstraintViolation();
            context.buildConstraintViolationWithTemplate(context.getDefaultConstraintMessageTemplate())
                    .addPropertyNode(confirmPasswordField)
                    .addConstraintViolation();
        }

        return isValid;
    }
}
```

---

## 4. User Registration Service

### 4.1 UserService Interface

```java
package com.ecommerce.service;

import com.ecommerce.dto.request.RegisterRequest;
import com.ecommerce.dto.request.ResendVerificationRequest;
import com.ecommerce.dto.request.VerifyEmailRequest;
import com.ecommerce.dto.response.UserDto;

public interface UserService {

    UserDto register(RegisterRequest request);

    void verifyEmail(String token);

    void resendVerificationEmail(String email);
}
```

### 4.2 UserServiceImpl

```java
package com.ecommerce.service.impl;

import com.ecommerce.config.properties.AppProperties;
import com.ecommerce.domain.entity.User;
import com.ecommerce.domain.entity.VerificationToken;
import com.ecommerce.domain.enums.Role;
import com.ecommerce.domain.enums.TokenType;
import com.ecommerce.dto.request.RegisterRequest;
import com.ecommerce.dto.response.UserDto;
import com.ecommerce.exception.BadRequestException;
import com.ecommerce.exception.ResourceNotFoundException;
import com.ecommerce.mapper.UserMapper;
import com.ecommerce.repository.UserRepository;
import com.ecommerce.repository.VerificationTokenRepository;
import com.ecommerce.service.EmailService;
import com.ecommerce.service.UserService;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.security.crypto.password.PasswordEncoder;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

@Service
@RequiredArgsConstructor
@Slf4j
public class UserServiceImpl implements UserService {

    private final UserRepository userRepository;
    private final VerificationTokenRepository tokenRepository;
    private final PasswordEncoder passwordEncoder;
    private final EmailService emailService;
    private final UserMapper userMapper;
    private final AppProperties appProperties;

    @Override
    @Transactional
    public UserDto register(RegisterRequest request) {
        log.info("Registering new user with email: {}", request.getEmail());

        // 1. Check email exists (backup check, validator should catch this)
        if (userRepository.existsByEmail(request.getEmail())) {
            throw new BadRequestException("Email already registered");
        }

        // 2. Create user entity
        User user = User.builder()
                .email(request.getEmail().toLowerCase().trim())
                .password(passwordEncoder.encode(request.getPassword()))
                .firstName(request.getFirstName().trim())
                .lastName(request.getLastName().trim())
                .phone(request.getPhone())
                .role(Role.CUSTOMER)
                .emailVerified(false)
                .accountLocked(false)
                .failedLoginAttempts(0)
                .build();

        // 3. Save user
        user = userRepository.save(user);
        log.info("User created with id: {}", user.getId());

        // 4. Generate verification token
        VerificationToken verificationToken = VerificationToken.createEmailVerification(
                user,
                appProperties.getVerification().getEmailExpirationHours()
        );
        tokenRepository.save(verificationToken);

        // 5. Send verification email
        sendVerificationEmail(user, verificationToken.getToken());

        return userMapper.toDto(user);
    }

    @Override
    @Transactional
    public void verifyEmail(String token) {
        log.info("Verifying email with token");

        // 1. Find token
        VerificationToken verificationToken = tokenRepository
                .findByTokenAndTokenType(token, TokenType.EMAIL_VERIFICATION)
                .orElseThrow(() -> new BadRequestException("Invalid verification token"));

        // 2. Validate token
        if (verificationToken.isUsed()) {
            throw new BadRequestException("Token has already been used");
        }

        if (verificationToken.isExpired()) {
            throw new BadRequestException("Token has expired. Please request a new one.");
        }

        // 3. Mark token as used
        verificationToken.markAsUsed();
        tokenRepository.save(verificationToken);

        // 4. Verify user email
        User user = verificationToken.getUser();
        user.setEmailVerified(true);
        userRepository.save(user);

        log.info("Email verified for user: {}", user.getEmail());
    }

    @Override
    @Transactional
    public void resendVerificationEmail(String email) {
        log.info("Resending verification email to: {}", email);

        // 1. Find user
        User user = userRepository.findByEmail(email.toLowerCase().trim())
                .orElseThrow(() -> new ResourceNotFoundException("User", "email", email));

        // 2. Check if already verified
        if (user.isEmailVerified()) {
            throw new BadRequestException("Email is already verified");
        }

        // 3. Invalidate old tokens
        tokenRepository.invalidateAllTokensForUser(user.getId(), TokenType.EMAIL_VERIFICATION);

        // 4. Create new token
        VerificationToken newToken = VerificationToken.createEmailVerification(
                user,
                appProperties.getVerification().getEmailExpirationHours()
        );
        tokenRepository.save(newToken);

        // 5. Send email
        sendVerificationEmail(user, newToken.getToken());

        log.info("Verification email resent to: {}", email);
    }

    private void sendVerificationEmail(User user, String token) {
        String verificationUrl = appProperties.getBaseUrl() +
                "/api/v1/auth/verify-email?token=" + token;

        emailService.sendEmail(
                user.getEmail(),
                "Verify Your Email - E-Commerce",
                "email/verify-email",
                java.util.Map.of(
                        "name", user.getFirstName(),
                        "verificationUrl", verificationUrl,
                        "expirationHours", appProperties.getVerification().getEmailExpirationHours()
                )
        );
    }
}
```

---

## 5. Email Service

### 5.1 Email Dependencies

```xml
<dependencies>
    <!-- Spring Mail -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-mail</artifactId>
    </dependency>

    <!-- Thymeleaf for email templates -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-thymeleaf</artifactId>
    </dependency>
</dependencies>
```

### 5.2 application.yml

```yaml
spring:
  mail:
    host: ${MAIL_HOST:smtp.gmail.com}
    port: ${MAIL_PORT:587}
    username: ${MAIL_USERNAME}
    password: ${MAIL_PASSWORD}
    properties:
      mail:
        smtp:
          auth: true
          starttls:
            enable: true
            required: true
          connectiontimeout: 5000
          timeout: 5000
          writetimeout: 5000

app:
  base-url: ${APP_BASE_URL:http://localhost:8080}
  mail:
    from-address: ${MAIL_FROM:noreply@ecommerce.com}
    from-name: ${MAIL_FROM_NAME:E-Commerce}
  verification:
    email-expiration-hours: 24
    password-reset-expiration-hours: 1
```

### 5.3 AppProperties

```java
package com.ecommerce.config.properties;

import lombok.Data;
import org.springframework.boot.context.properties.ConfigurationProperties;
import org.springframework.stereotype.Component;

@Component
@ConfigurationProperties(prefix = "app")
@Data
public class AppProperties {

    private String baseUrl;
    private Mail mail = new Mail();
    private Verification verification = new Verification();

    @Data
    public static class Mail {
        private String fromAddress;
        private String fromName;
    }

    @Data
    public static class Verification {
        private int emailExpirationHours = 24;
        private int passwordResetExpirationHours = 1;
    }
}
```

### 5.4 EmailService

```java
package com.ecommerce.service;

import java.util.Map;

public interface EmailService {

    void sendEmail(String to, String subject, String templateName, Map<String, Object> variables);

    void sendSimpleEmail(String to, String subject, String text);
}
```

```java
package com.ecommerce.service.impl;

import com.ecommerce.config.properties.AppProperties;
import com.ecommerce.service.EmailService;
import jakarta.mail.MessagingException;
import jakarta.mail.internet.MimeMessage;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.mail.SimpleMailMessage;
import org.springframework.mail.javamail.JavaMailSender;
import org.springframework.mail.javamail.MimeMessageHelper;
import org.springframework.scheduling.annotation.Async;
import org.springframework.stereotype.Service;
import org.thymeleaf.TemplateEngine;
import org.thymeleaf.context.Context;

import java.util.Map;

@Service
@RequiredArgsConstructor
@Slf4j
public class EmailServiceImpl implements EmailService {

    private final JavaMailSender mailSender;
    private final TemplateEngine templateEngine;
    private final AppProperties appProperties;

    @Override
    @Async("emailExecutor")
    public void sendEmail(String to, String subject, String templateName,
                          Map<String, Object> variables) {
        try {
            // 1. Prepare context
            Context context = new Context();
            context.setVariables(variables);

            // 2. Process template
            String htmlContent = templateEngine.process(templateName, context);

            // 3. Create message
            MimeMessage message = mailSender.createMimeMessage();
            MimeMessageHelper helper = new MimeMessageHelper(message, true, "UTF-8");

            helper.setFrom(appProperties.getMail().getFromAddress(),
                    appProperties.getMail().getFromName());
            helper.setTo(to);
            helper.setSubject(subject);
            helper.setText(htmlContent, true); // true = HTML

            // 4. Send
            mailSender.send(message);
            log.info("Email sent to: {}", to);

        } catch (MessagingException e) {
            log.error("Failed to send email to {}: {}", to, e.getMessage());
            throw new RuntimeException("Failed to send email", e);
        } catch (Exception e) {
            log.error("Email sending error: {}", e.getMessage());
            throw new RuntimeException("Email sending failed", e);
        }
    }

    @Override
    @Async("emailExecutor")
    public void sendSimpleEmail(String to, String subject, String text) {
        try {
            SimpleMailMessage message = new SimpleMailMessage();
            message.setFrom(appProperties.getMail().getFromAddress());
            message.setTo(to);
            message.setSubject(subject);
            message.setText(text);

            mailSender.send(message);
            log.info("Simple email sent to: {}", to);

        } catch (Exception e) {
            log.error("Failed to send simple email to {}: {}", to, e.getMessage());
            throw new RuntimeException("Failed to send email", e);
        }
    }
}
```

### 5.5 Async Configuration

```java
package com.ecommerce.config;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.scheduling.annotation.EnableAsync;
import org.springframework.scheduling.concurrent.ThreadPoolTaskExecutor;

import java.util.concurrent.Executor;

@Configuration
@EnableAsync
public class AsyncConfig {

    @Bean(name = "emailExecutor")
    public Executor emailExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(2);
        executor.setMaxPoolSize(5);
        executor.setQueueCapacity(100);
        executor.setThreadNamePrefix("email-");
        executor.initialize();
        return executor;
    }
}
```

### 5.6 Email Template (Thymeleaf)

```html
<!-- src/main/resources/templates/email/verify-email.html -->
<!DOCTYPE html>
<html xmlns:th="http://www.thymeleaf.org">
<head>
    <meta charset="UTF-8">
    <title>Verify Your Email</title>
    <style>
        body {
            font-family: Arial, sans-serif;
            line-height: 1.6;
            color: #333;
            max-width: 600px;
            margin: 0 auto;
            padding: 20px;
        }
        .header {
            background-color: #4CAF50;
            color: white;
            padding: 20px;
            text-align: center;
        }
        .content {
            padding: 20px;
            background-color: #f9f9f9;
        }
        .button {
            display: inline-block;
            padding: 12px 24px;
            background-color: #4CAF50;
            color: white;
            text-decoration: none;
            border-radius: 4px;
            margin: 20px 0;
        }
        .footer {
            text-align: center;
            padding: 20px;
            color: #666;
            font-size: 12px;
        }
    </style>
</head>
<body>
    <div class="header">
        <h1>E-Commerce</h1>
    </div>

    <div class="content">
        <h2>Hello, <span th:text="${name}">User</span>!</h2>

        <p>Thank you for registering with E-Commerce. Please verify your email address by clicking the button below:</p>

        <p style="text-align: center;">
            <a th:href="${verificationUrl}" class="button">Verify Email</a>
        </p>

        <p>Or copy and paste this link into your browser:</p>
        <p style="word-break: break-all;" th:text="${verificationUrl}"></p>

        <p><strong>This link will expire in <span th:text="${expirationHours}">24</span> hours.</strong></p>

        <p>If you didn't create an account, you can safely ignore this email.</p>
    </div>

    <div class="footer">
        <p>&copy; 2024 E-Commerce. All rights reserved.</p>
        <p>This is an automated message. Please do not reply.</p>
    </div>
</body>
</html>
```

---

## 6. Password Reset Flow

```
┌─────────────────────────────────────────────────────────────────┐
│                    PASSWORD RESET FLOW                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌───────────────────┐                                          │
│  │ User clicks       │                                          │
│  │ "Forgot Password" │                                          │
│  └─────────┬─────────┘                                          │
│            │                                                     │
│            ▼                                                     │
│  ┌───────────────────┐     ┌───────────────────┐               │
│  │ Enter email       │────▶│ Email exists?     │               │
│  └───────────────────┘     └─────────┬─────────┘               │
│                                      │                          │
│                         ┌────────────┴────────────┐             │
│                        YES                        NO             │
│                         │                         │              │
│                         ▼                         ▼              │
│  ┌────────────────────────────┐  ┌─────────────────────────┐   │
│  │ Generate reset token       │  │ Return generic message  │   │
│  │ Send email with link       │  │ (don't reveal email     │   │
│  └────────────┬───────────────┘  │  exists or not)         │   │
│               │                  └─────────────────────────┘   │
│               ▼                                                 │
│  ┌────────────────────────────┐                                │
│  │ User clicks link           │                                │
│  │ (within 1 hour)            │                                │
│  └────────────┬───────────────┘                                │
│               │                                                 │
│               ▼                                                 │
│  ┌────────────────────────────┐                                │
│  │ Validate token             │                                │
│  │ Show password reset form   │                                │
│  └────────────┬───────────────┘                                │
│               │                                                 │
│               ▼                                                 │
│  ┌────────────────────────────┐                                │
│  │ User enters new password   │                                │
│  │ Confirm password           │                                │
│  └────────────┬───────────────┘                                │
│               │                                                 │
│               ▼                                                 │
│  ┌────────────────────────────┐                                │
│  │ Update password in DB      │                                │
│  │ Invalidate all tokens      │                                │
│  │ Send confirmation email    │                                │
│  └────────────────────────────┘                                │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 6.1 Password Reset DTOs

```java
package com.ecommerce.dto.request;

import jakarta.validation.constraints.Email;
import jakarta.validation.constraints.NotBlank;
import lombok.Data;

@Data
public class ForgotPasswordRequest {

    @NotBlank(message = "Email is required")
    @Email(message = "Invalid email format")
    private String email;
}
```

```java
package com.ecommerce.dto.request;

import com.ecommerce.validation.PasswordMatch;
import jakarta.validation.constraints.NotBlank;
import jakarta.validation.constraints.Pattern;
import jakarta.validation.constraints.Size;
import lombok.Data;

@Data
@PasswordMatch(password = "newPassword", confirmPassword = "confirmPassword")
public class ResetPasswordRequest {

    @NotBlank(message = "Token is required")
    private String token;

    @NotBlank(message = "New password is required")
    @Size(min = 8, max = 100)
    @Pattern(
        regexp = "^(?=.*[a-z])(?=.*[A-Z])(?=.*\\d)(?=.*[@$!%*?&])[A-Za-z\\d@$!%*?&]+$",
        message = "Password must contain uppercase, lowercase, digit and special character"
    )
    private String newPassword;

    @NotBlank(message = "Confirm password is required")
    private String confirmPassword;
}
```

### 6.2 Password Reset Service

```java
package com.ecommerce.service;

import com.ecommerce.dto.request.ChangePasswordRequest;
import com.ecommerce.dto.request.ForgotPasswordRequest;
import com.ecommerce.dto.request.ResetPasswordRequest;

public interface PasswordService {

    void forgotPassword(ForgotPasswordRequest request);

    void resetPassword(ResetPasswordRequest request);

    void changePassword(ChangePasswordRequest request);
}
```

```java
package com.ecommerce.service.impl;

import com.ecommerce.config.properties.AppProperties;
import com.ecommerce.domain.entity.User;
import com.ecommerce.domain.entity.VerificationToken;
import com.ecommerce.domain.enums.TokenType;
import com.ecommerce.dto.request.ChangePasswordRequest;
import com.ecommerce.dto.request.ForgotPasswordRequest;
import com.ecommerce.dto.request.ResetPasswordRequest;
import com.ecommerce.exception.BadRequestException;
import com.ecommerce.repository.UserRepository;
import com.ecommerce.repository.VerificationTokenRepository;
import com.ecommerce.service.EmailService;
import com.ecommerce.service.PasswordService;
import com.ecommerce.util.SecurityUtils;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.security.crypto.password.PasswordEncoder;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.util.Map;

@Service
@RequiredArgsConstructor
@Slf4j
public class PasswordServiceImpl implements PasswordService {

    private final UserRepository userRepository;
    private final VerificationTokenRepository tokenRepository;
    private final PasswordEncoder passwordEncoder;
    private final EmailService emailService;
    private final AppProperties appProperties;

    @Override
    @Transactional
    public void forgotPassword(ForgotPasswordRequest request) {
        String email = request.getEmail().toLowerCase().trim();
        log.info("Password reset requested for: {}", email);

        // Find user (don't reveal if email exists or not)
        userRepository.findByEmail(email).ifPresent(user -> {
            // Invalidate existing reset tokens
            tokenRepository.invalidateAllTokensForUser(user.getId(), TokenType.PASSWORD_RESET);

            // Create new token
            VerificationToken resetToken = VerificationToken.createPasswordReset(
                    user,
                    appProperties.getVerification().getPasswordResetExpirationHours()
            );
            tokenRepository.save(resetToken);

            // Send email
            sendPasswordResetEmail(user, resetToken.getToken());
        });

        // Always log success (don't reveal if email exists)
        log.info("Password reset email sent if user exists");
    }

    @Override
    @Transactional
    public void resetPassword(ResetPasswordRequest request) {
        log.info("Processing password reset");

        // 1. Find and validate token
        VerificationToken token = tokenRepository
                .findByTokenAndTokenType(request.getToken(), TokenType.PASSWORD_RESET)
                .orElseThrow(() -> new BadRequestException("Invalid or expired token"));

        if (!token.isValid()) {
            throw new BadRequestException("Token has expired or already been used");
        }

        // 2. Update password
        User user = token.getUser();
        user.setPassword(passwordEncoder.encode(request.getNewPassword()));
        user.resetFailedAttempts(); // Unlock account if locked
        userRepository.save(user);

        // 3. Mark token as used
        token.markAsUsed();
        tokenRepository.save(token);

        // 4. Invalidate all other reset tokens
        tokenRepository.invalidateAllTokensForUser(user.getId(), TokenType.PASSWORD_RESET);

        // 5. Send confirmation email
        sendPasswordChangedEmail(user);

        log.info("Password reset successful for: {}", user.getEmail());
    }

    @Override
    @Transactional
    public void changePassword(ChangePasswordRequest request) {
        // 1. Get current user
        User user = SecurityUtils.getCurrentUser()
                .orElseThrow(() -> new BadRequestException("User not authenticated"));

        // 2. Verify current password
        if (!passwordEncoder.matches(request.getCurrentPassword(), user.getPassword())) {
            throw new BadRequestException("Current password is incorrect");
        }

        // 3. Check new password is different
        if (passwordEncoder.matches(request.getNewPassword(), user.getPassword())) {
            throw new BadRequestException("New password must be different from current password");
        }

        // 4. Update password
        user.setPassword(passwordEncoder.encode(request.getNewPassword()));
        userRepository.save(user);

        // 5. Send notification email
        sendPasswordChangedEmail(user);

        log.info("Password changed for user: {}", user.getEmail());
    }

    private void sendPasswordResetEmail(User user, String token) {
        String resetUrl = appProperties.getBaseUrl() +
                "/reset-password?token=" + token;

        emailService.sendEmail(
                user.getEmail(),
                "Reset Your Password - E-Commerce",
                "email/reset-password",
                Map.of(
                        "name", user.getFirstName(),
                        "resetUrl", resetUrl,
                        "expirationHours", appProperties.getVerification().getPasswordResetExpirationHours()
                )
        );
    }

    private void sendPasswordChangedEmail(User user) {
        emailService.sendEmail(
                user.getEmail(),
                "Password Changed - E-Commerce",
                "email/password-changed",
                Map.of(
                        "name", user.getFirstName()
                )
        );
    }
}
```

---

## 7. Verification Token Repository

```java
package com.ecommerce.repository;

import com.ecommerce.domain.entity.VerificationToken;
import com.ecommerce.domain.enums.TokenType;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.data.jpa.repository.Modifying;
import org.springframework.data.jpa.repository.Query;
import org.springframework.data.repository.query.Param;
import org.springframework.stereotype.Repository;

import java.time.LocalDateTime;
import java.util.Optional;

@Repository
public interface VerificationTokenRepository extends JpaRepository<VerificationToken, Long> {

    Optional<VerificationToken> findByToken(String token);

    Optional<VerificationToken> findByTokenAndTokenType(String token, TokenType tokenType);

    @Query("SELECT vt FROM VerificationToken vt WHERE vt.user.id = :userId " +
           "AND vt.tokenType = :tokenType AND vt.used = false AND vt.expiresAt > :now")
    Optional<VerificationToken> findValidTokenForUser(
            @Param("userId") Long userId,
            @Param("tokenType") TokenType tokenType,
            @Param("now") LocalDateTime now
    );

    @Modifying
    @Query("UPDATE VerificationToken vt SET vt.used = true WHERE vt.user.id = :userId " +
           "AND vt.tokenType = :tokenType AND vt.used = false")
    void invalidateAllTokensForUser(
            @Param("userId") Long userId,
            @Param("tokenType") TokenType tokenType
    );

    @Modifying
    @Query("DELETE FROM VerificationToken vt WHERE vt.expiresAt < :now")
    int deleteExpiredTokens(@Param("now") LocalDateTime now);
}
```

---

## 8. Auth Controller Updates

```java
package com.ecommerce.controller;

import com.ecommerce.dto.request.*;
import com.ecommerce.dto.response.ApiResponse;
import com.ecommerce.dto.response.AuthResponse;
import com.ecommerce.dto.response.UserDto;
import com.ecommerce.service.AuthService;
import com.ecommerce.service.PasswordService;
import com.ecommerce.service.UserService;
import io.swagger.v3.oas.annotations.Operation;
import io.swagger.v3.oas.annotations.tags.Tag;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.validation.Valid;
import lombok.RequiredArgsConstructor;
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

@RestController
@RequestMapping("/api/v1/auth")
@RequiredArgsConstructor
@Tag(name = "Authentication", description = "Authentication and user management APIs")
public class AuthController {

    private final AuthService authService;
    private final UserService userService;
    private final PasswordService passwordService;

    // ========== Registration ==========

    @PostMapping("/register")
    @Operation(summary = "Register new user")
    public ResponseEntity<ApiResponse<UserDto>> register(
            @Valid @RequestBody RegisterRequest request) {

        UserDto user = userService.register(request);
        return ResponseEntity
                .status(HttpStatus.CREATED)
                .body(ApiResponse.success(user, "Registration successful. Please verify your email."));
    }

    @GetMapping("/verify-email")
    @Operation(summary = "Verify email address")
    public ResponseEntity<ApiResponse<Void>> verifyEmail(
            @RequestParam String token) {

        userService.verifyEmail(token);
        return ResponseEntity.ok(ApiResponse.success(null, "Email verified successfully"));
    }

    @PostMapping("/resend-verification")
    @Operation(summary = "Resend verification email")
    public ResponseEntity<ApiResponse<Void>> resendVerification(
            @RequestParam String email) {

        userService.resendVerificationEmail(email);
        return ResponseEntity.ok(ApiResponse.success(null, "Verification email sent"));
    }

    // ========== Login/Logout ==========

    @PostMapping("/login")
    @Operation(summary = "User login")
    public ResponseEntity<ApiResponse<AuthResponse>> login(
            @Valid @RequestBody LoginRequest request) {

        AuthResponse response = authService.login(request);
        return ResponseEntity.ok(ApiResponse.success(response, "Login successful"));
    }

    @PostMapping("/refresh")
    @Operation(summary = "Refresh access token")
    public ResponseEntity<ApiResponse<AuthResponse>> refreshToken(
            @Valid @RequestBody RefreshTokenRequest request) {

        AuthResponse response = authService.refreshToken(request);
        return ResponseEntity.ok(ApiResponse.success(response, "Token refreshed"));
    }

    @PostMapping("/logout")
    @Operation(summary = "Logout user")
    public ResponseEntity<ApiResponse<Void>> logout(HttpServletRequest request) {
        String authHeader = request.getHeader("Authorization");
        authService.logout(authHeader);
        return ResponseEntity.ok(ApiResponse.success(null, "Logged out successfully"));
    }

    // ========== Password Reset ==========

    @PostMapping("/forgot-password")
    @Operation(summary = "Request password reset")
    public ResponseEntity<ApiResponse<Void>> forgotPassword(
            @Valid @RequestBody ForgotPasswordRequest request) {

        passwordService.forgotPassword(request);
        return ResponseEntity.ok(ApiResponse.success(null,
                "If your email is registered, you will receive a password reset link"));
    }

    @PostMapping("/reset-password")
    @Operation(summary = "Reset password with token")
    public ResponseEntity<ApiResponse<Void>> resetPassword(
            @Valid @RequestBody ResetPasswordRequest request) {

        passwordService.resetPassword(request);
        return ResponseEntity.ok(ApiResponse.success(null, "Password reset successful"));
    }
}
```

---

## 9. Scheduled Token Cleanup

```java
package com.ecommerce.scheduler;

import com.ecommerce.repository.VerificationTokenRepository;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.scheduling.annotation.Scheduled;
import org.springframework.stereotype.Component;
import org.springframework.transaction.annotation.Transactional;

import java.time.LocalDateTime;

@Component
@RequiredArgsConstructor
@Slf4j
public class TokenCleanupScheduler {

    private final VerificationTokenRepository tokenRepository;

    /**
     * Clean up expired tokens every day at 2 AM
     */
    @Scheduled(cron = "0 0 2 * * ?")
    @Transactional
    public void cleanupExpiredTokens() {
        log.info("Starting expired token cleanup");

        int deletedCount = tokenRepository.deleteExpiredTokens(LocalDateTime.now());

        log.info("Deleted {} expired tokens", deletedCount);
    }
}
```

---

## 10. Bài tập thực hành

### Bài tập 1: Complete Registration
- [x] RegisterRequest với full validation
- [x] Custom validators (UniqueEmail, PasswordMatch)
- [x] Email verification flow

### Bài tập 2: Password Management
- [x] Forgot password flow
- [x] Reset password với token
- [ ] Change password (authenticated)

### Bài tập 3: Email Templates
- [x] Verification email template
- [ ] Password reset template
- [ ] Password changed notification

---

## 11. Checklist

- [ ] Create VerificationToken entity và migration
- [ ] Implement registration với email verification
- [ ] Create custom validators
- [ ] Setup email service với Thymeleaf templates
- [ ] Implement forgot/reset password flow
- [ ] Add token cleanup scheduler
- [ ] Handle edge cases (expired tokens, resend limits)
- [ ] Test complete registration flow
- [ ] Test password reset flow

---

## Điều hướng

- [← Day 17: JWT Authentication](./day-17-jwt-authentication.md)
- [Day 19: Role-Based Authorization →](./day-19-role-authorization.md)
- [Về trang chính](../00-overview.md)
