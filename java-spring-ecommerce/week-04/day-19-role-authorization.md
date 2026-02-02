# Day 19: Role-Based Authorization

## Mục tiêu học tập
- Implement Role & Permission system
- Hiểu sự khác biệt Role vs Permission
- Tạo dynamic authorization
- Implement resource-based authorization

---

## 1. RBAC (Role-Based Access Control) Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                    RBAC ARCHITECTURE                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌─────────────┐                                                │
│  │    USER     │                                                │
│  │  (john@...)│                                                │
│  └──────┬──────┘                                                │
│         │ has                                                    │
│         ▼                                                        │
│  ┌─────────────┐      ┌─────────────┐      ┌─────────────┐     │
│  │   ROLE(S)   │      │   ROLE(S)   │      │   ROLE(S)   │     │
│  │  CUSTOMER   │      │   SELLER    │      │    ADMIN    │     │
│  └──────┬──────┘      └──────┬──────┘      └──────┬──────┘     │
│         │ grants             │ grants             │ grants      │
│         ▼                    ▼                    ▼             │
│  ┌─────────────┐      ┌─────────────┐      ┌─────────────┐     │
│  │ PERMISSIONS │      │ PERMISSIONS │      │ PERMISSIONS │     │
│  ├─────────────┤      ├─────────────┤      ├─────────────┤     │
│  │ product:read│      │ product:read│      │ product:*   │     │
│  │ order:create│      │ product:write│     │ order:*     │     │
│  │ order:read  │      │ order:read  │      │ user:*      │     │
│  │ review:write│      │ seller:*    │      │ admin:*     │     │
│  └─────────────┘      └─────────────┘      └─────────────┘     │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 1.1 Role vs Permission

```
┌────────────────────────────────────────────────────────────────┐
│                  ROLE vs PERMISSION                             │
├────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ROLE (Vai trò)              PERMISSION (Quyền)                │
│  ───────────────             ──────────────────                │
│                                                                 │
│  • High-level grouping       • Granular access control         │
│  • "Who you are"             • "What you can do"               │
│  • Examples:                 • Examples:                       │
│    - ADMIN                     - user:read                     │
│    - SELLER                    - user:write                    │
│    - CUSTOMER                  - product:delete                │
│                                - order:approve                 │
│                                                                 │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │ Best Practice: Use Roles for coarse-grained checks,      │  │
│  │                Use Permissions for fine-grained control  │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                 │
└────────────────────────────────────────────────────────────────┘
```

---

## 2. Permission & Role Entities

### 2.1 Permission Entity

```java
package com.ecommerce.domain.entity;

import jakarta.persistence.*;
import lombok.*;

import java.util.HashSet;
import java.util.Set;

@Entity
@Table(name = "permissions")
@Getter
@Setter
@NoArgsConstructor
@AllArgsConstructor
@Builder
public class Permission extends BaseEntity {

    @Column(nullable = false, unique = true, length = 50)
    private String name;  // e.g., "product:read", "order:create"

    @Column(length = 100)
    private String displayName;  // e.g., "Read Products"

    @Column(length = 255)
    private String description;

    @Column(length = 50)
    private String module;  // e.g., "product", "order", "user"

    @ManyToMany(mappedBy = "permissions")
    private Set<Role> roles = new HashSet<>();

    // Factory method for common permission format
    public static Permission of(String module, String action) {
        return Permission.builder()
                .name(module + ":" + action)
                .module(module)
                .displayName(capitalize(action) + " " + capitalize(module))
                .build();
    }

    private static String capitalize(String str) {
        return str.substring(0, 1).toUpperCase() + str.substring(1);
    }
}
```

### 2.2 Role Entity (Updated)

```java
package com.ecommerce.domain.entity;

import jakarta.persistence.*;
import lombok.*;

import java.util.HashSet;
import java.util.Set;

@Entity
@Table(name = "roles")
@Getter
@Setter
@NoArgsConstructor
@AllArgsConstructor
@Builder
public class Role extends BaseEntity {

    @Column(nullable = false, unique = true, length = 50)
    private String name;  // e.g., "ADMIN", "SELLER", "CUSTOMER"

    @Column(length = 100)
    private String displayName;

    @Column(length = 255)
    private String description;

    @Column(name = "is_default")
    private boolean defaultRole = false;  // Default role for new users

    @Column(name = "is_system")
    private boolean systemRole = false;   // System role (cannot be deleted)

    @ManyToMany(fetch = FetchType.EAGER)
    @JoinTable(
            name = "role_permissions",
            joinColumns = @JoinColumn(name = "role_id"),
            inverseJoinColumns = @JoinColumn(name = "permission_id")
    )
    private Set<Permission> permissions = new HashSet<>();

    @ManyToMany(mappedBy = "roles")
    private Set<User> users = new HashSet<>();

    // Helper methods
    public void addPermission(Permission permission) {
        permissions.add(permission);
        permission.getRoles().add(this);
    }

    public void removePermission(Permission permission) {
        permissions.remove(permission);
        permission.getRoles().remove(this);
    }

    public boolean hasPermission(String permissionName) {
        return permissions.stream()
                .anyMatch(p -> p.getName().equals(permissionName));
    }

    public boolean hasAnyPermission(String... permissionNames) {
        Set<String> permSet = Set.of(permissionNames);
        return permissions.stream()
                .anyMatch(p -> permSet.contains(p.getName()));
    }
}
```

### 2.3 User Entity (Updated)

```java
package com.ecommerce.domain.entity;

import jakarta.persistence.*;
import lombok.*;
import org.springframework.security.core.GrantedAuthority;
import org.springframework.security.core.authority.SimpleGrantedAuthority;
import org.springframework.security.core.userdetails.UserDetails;

import java.util.*;
import java.util.stream.Collectors;

@Entity
@Table(name = "users")
@Getter
@Setter
@NoArgsConstructor
@AllArgsConstructor
@Builder
public class User extends BaseEntity implements UserDetails {

    // ... existing fields ...

    @ManyToMany(fetch = FetchType.EAGER)
    @JoinTable(
            name = "user_roles",
            joinColumns = @JoinColumn(name = "user_id"),
            inverseJoinColumns = @JoinColumn(name = "role_id")
    )
    @Builder.Default
    private Set<Role> roles = new HashSet<>();

    // Direct permissions (bypass role)
    @ManyToMany(fetch = FetchType.EAGER)
    @JoinTable(
            name = "user_permissions",
            joinColumns = @JoinColumn(name = "user_id"),
            inverseJoinColumns = @JoinColumn(name = "permission_id")
    )
    @Builder.Default
    private Set<Permission> directPermissions = new HashSet<>();

    // ========== UserDetails Implementation ==========

    @Override
    public Collection<? extends GrantedAuthority> getAuthorities() {
        Set<GrantedAuthority> authorities = new HashSet<>();

        // Add role authorities (ROLE_xxx format)
        roles.forEach(role -> {
            authorities.add(new SimpleGrantedAuthority("ROLE_" + role.getName()));

            // Add permission authorities from role
            role.getPermissions().forEach(permission ->
                    authorities.add(new SimpleGrantedAuthority(permission.getName()))
            );
        });

        // Add direct permission authorities
        directPermissions.forEach(permission ->
                authorities.add(new SimpleGrantedAuthority(permission.getName()))
        );

        return authorities;
    }

    // ... rest of UserDetails implementation ...

    // ========== Helper Methods ==========

    public void addRole(Role role) {
        roles.add(role);
        role.getUsers().add(this);
    }

    public void removeRole(Role role) {
        roles.remove(role);
        role.getUsers().remove(this);
    }

    public boolean hasRole(String roleName) {
        return roles.stream()
                .anyMatch(r -> r.getName().equals(roleName));
    }

    public boolean hasPermission(String permissionName) {
        // Check direct permissions
        if (directPermissions.stream().anyMatch(p -> p.getName().equals(permissionName))) {
            return true;
        }

        // Check role permissions
        return roles.stream()
                .flatMap(r -> r.getPermissions().stream())
                .anyMatch(p -> p.getName().equals(permissionName));
    }

    public Set<String> getAllPermissions() {
        Set<String> permissions = new HashSet<>();

        // Add role permissions
        roles.forEach(role ->
                role.getPermissions().forEach(p -> permissions.add(p.getName()))
        );

        // Add direct permissions
        directPermissions.forEach(p -> permissions.add(p.getName()));

        return permissions;
    }
}
```

---

## 3. Database Migrations

```sql
-- V8__Create_roles_permissions_tables.sql

-- Permissions table
CREATE TABLE permissions (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(50) NOT NULL UNIQUE,
    display_name VARCHAR(100),
    description VARCHAR(255),
    module VARCHAR(50),
    created_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at DATETIME,
    deleted BOOLEAN DEFAULT FALSE,

    INDEX idx_permission_name (name),
    INDEX idx_permission_module (module)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;

-- Roles table
CREATE TABLE roles (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(50) NOT NULL UNIQUE,
    display_name VARCHAR(100),
    description VARCHAR(255),
    is_default BOOLEAN DEFAULT FALSE,
    is_system BOOLEAN DEFAULT FALSE,
    created_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at DATETIME,
    deleted BOOLEAN DEFAULT FALSE,

    INDEX idx_role_name (name)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;

-- Role-Permission junction table
CREATE TABLE role_permissions (
    role_id BIGINT NOT NULL,
    permission_id BIGINT NOT NULL,
    PRIMARY KEY (role_id, permission_id),

    CONSTRAINT fk_rp_role FOREIGN KEY (role_id) REFERENCES roles(id) ON DELETE CASCADE,
    CONSTRAINT fk_rp_permission FOREIGN KEY (permission_id) REFERENCES permissions(id) ON DELETE CASCADE
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

-- User-Role junction table
CREATE TABLE user_roles (
    user_id BIGINT NOT NULL,
    role_id BIGINT NOT NULL,
    PRIMARY KEY (user_id, role_id),

    CONSTRAINT fk_ur_user FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE,
    CONSTRAINT fk_ur_role FOREIGN KEY (role_id) REFERENCES roles(id) ON DELETE CASCADE
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

-- User-Permission (direct) junction table
CREATE TABLE user_permissions (
    user_id BIGINT NOT NULL,
    permission_id BIGINT NOT NULL,
    PRIMARY KEY (user_id, permission_id),

    CONSTRAINT fk_up_user FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE,
    CONSTRAINT fk_up_permission FOREIGN KEY (permission_id) REFERENCES permissions(id) ON DELETE CASCADE
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

```sql
-- V9__Seed_roles_permissions.sql

-- Insert default permissions
INSERT INTO permissions (name, display_name, module, created_at) VALUES
-- Product permissions
('product:read', 'View Products', 'product', NOW()),
('product:create', 'Create Products', 'product', NOW()),
('product:update', 'Update Products', 'product', NOW()),
('product:delete', 'Delete Products', 'product', NOW()),

-- Order permissions
('order:read', 'View Orders', 'order', NOW()),
('order:create', 'Create Orders', 'order', NOW()),
('order:update', 'Update Orders', 'order', NOW()),
('order:cancel', 'Cancel Orders', 'order', NOW()),
('order:approve', 'Approve Orders', 'order', NOW()),

-- User permissions
('user:read', 'View Users', 'user', NOW()),
('user:create', 'Create Users', 'user', NOW()),
('user:update', 'Update Users', 'user', NOW()),
('user:delete', 'Delete Users', 'user', NOW()),

-- Review permissions
('review:read', 'View Reviews', 'review', NOW()),
('review:create', 'Create Reviews', 'review', NOW()),
('review:update', 'Update Own Reviews', 'review', NOW()),
('review:delete', 'Delete Reviews', 'review', NOW()),
('review:moderate', 'Moderate Reviews', 'review', NOW()),

-- Admin permissions
('admin:dashboard', 'Access Admin Dashboard', 'admin', NOW()),
('admin:settings', 'Manage Settings', 'admin', NOW()),
('admin:reports', 'View Reports', 'admin', NOW());

-- Insert default roles
INSERT INTO roles (name, display_name, description, is_default, is_system, created_at) VALUES
('CUSTOMER', 'Customer', 'Regular customer with basic access', TRUE, TRUE, NOW()),
('SELLER', 'Seller', 'Seller with product management access', FALSE, TRUE, NOW()),
('ADMIN', 'Administrator', 'Full system access', FALSE, TRUE, NOW()),
('SUPER_ADMIN', 'Super Administrator', 'Ultimate system access', FALSE, TRUE, NOW());

-- Assign permissions to CUSTOMER role
INSERT INTO role_permissions (role_id, permission_id)
SELECT r.id, p.id FROM roles r, permissions p
WHERE r.name = 'CUSTOMER' AND p.name IN (
    'product:read', 'order:create', 'order:read', 'review:create', 'review:read', 'review:update'
);

-- Assign permissions to SELLER role
INSERT INTO role_permissions (role_id, permission_id)
SELECT r.id, p.id FROM roles r, permissions p
WHERE r.name = 'SELLER' AND p.name IN (
    'product:read', 'product:create', 'product:update',
    'order:read', 'order:update',
    'review:read'
);

-- Assign ALL permissions to ADMIN role
INSERT INTO role_permissions (role_id, permission_id)
SELECT r.id, p.id FROM roles r, permissions p
WHERE r.name = 'ADMIN' AND p.name NOT LIKE 'admin:settings';

-- Assign ALL permissions to SUPER_ADMIN role
INSERT INTO role_permissions (role_id, permission_id)
SELECT r.id, p.id FROM roles r, permissions p
WHERE r.name = 'SUPER_ADMIN';
```

---

## 4. Permission Constants

```java
package com.ecommerce.security;

public final class Permissions {

    private Permissions() {}

    // ========== Product Permissions ==========
    public static final String PRODUCT_READ = "product:read";
    public static final String PRODUCT_CREATE = "product:create";
    public static final String PRODUCT_UPDATE = "product:update";
    public static final String PRODUCT_DELETE = "product:delete";

    // ========== Order Permissions ==========
    public static final String ORDER_READ = "order:read";
    public static final String ORDER_CREATE = "order:create";
    public static final String ORDER_UPDATE = "order:update";
    public static final String ORDER_CANCEL = "order:cancel";
    public static final String ORDER_APPROVE = "order:approve";

    // ========== User Permissions ==========
    public static final String USER_READ = "user:read";
    public static final String USER_CREATE = "user:create";
    public static final String USER_UPDATE = "user:update";
    public static final String USER_DELETE = "user:delete";

    // ========== Review Permissions ==========
    public static final String REVIEW_READ = "review:read";
    public static final String REVIEW_CREATE = "review:create";
    public static final String REVIEW_UPDATE = "review:update";
    public static final String REVIEW_DELETE = "review:delete";
    public static final String REVIEW_MODERATE = "review:moderate";

    // ========== Admin Permissions ==========
    public static final String ADMIN_DASHBOARD = "admin:dashboard";
    public static final String ADMIN_SETTINGS = "admin:settings";
    public static final String ADMIN_REPORTS = "admin:reports";
}
```

---

## 5. Custom Security Annotations

### 5.1 Permission-Based Annotations

```java
package com.ecommerce.security.annotation;

import org.springframework.security.access.prepost.PreAuthorize;

import java.lang.annotation.*;

// ========== Product Permissions ==========

@Target({ElementType.METHOD, ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
@PreAuthorize("hasAuthority('product:read')")
public @interface CanReadProducts {}

@Target({ElementType.METHOD, ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
@PreAuthorize("hasAuthority('product:create')")
public @interface CanCreateProducts {}

@Target({ElementType.METHOD, ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
@PreAuthorize("hasAuthority('product:update')")
public @interface CanUpdateProducts {}

@Target({ElementType.METHOD, ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
@PreAuthorize("hasAuthority('product:delete')")
public @interface CanDeleteProducts {}

// ========== Order Permissions ==========

@Target({ElementType.METHOD, ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
@PreAuthorize("hasAuthority('order:read')")
public @interface CanReadOrders {}

@Target({ElementType.METHOD, ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
@PreAuthorize("hasAuthority('order:create')")
public @interface CanCreateOrders {}

@Target({ElementType.METHOD, ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
@PreAuthorize("hasAuthority('order:approve')")
public @interface CanApproveOrders {}

// ========== Admin Permissions ==========

@Target({ElementType.METHOD, ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
@PreAuthorize("hasRole('ADMIN') or hasRole('SUPER_ADMIN')")
public @interface AdminOnly {}

@Target({ElementType.METHOD, ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
@PreAuthorize("hasRole('SUPER_ADMIN')")
public @interface SuperAdminOnly {}
```

### 5.2 Resource Owner Annotation

```java
package com.ecommerce.security.annotation;

import org.springframework.security.access.prepost.PreAuthorize;

import java.lang.annotation.*;

/**
 * Cho phép owner hoặc admin truy cập resource
 * Sử dụng với SpEL expression để check ownership
 */
@Target({ElementType.METHOD})
@Retention(RetentionPolicy.RUNTIME)
@PreAuthorize("@resourceAuthorizationService.isOwnerOrAdmin(#id, authentication)")
public @interface OwnerOrAdmin {}
```

---

## 6. Authorization Service

### 6.1 ResourceAuthorizationService

```java
package com.ecommerce.security;

import com.ecommerce.domain.entity.User;
import com.ecommerce.repository.OrderRepository;
import com.ecommerce.repository.ProductRepository;
import com.ecommerce.repository.ReviewRepository;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.security.core.Authentication;
import org.springframework.stereotype.Service;

@Service("resourceAuthorizationService")
@RequiredArgsConstructor
@Slf4j
public class ResourceAuthorizationService {

    private final OrderRepository orderRepository;
    private final ProductRepository productRepository;
    private final ReviewRepository reviewRepository;

    /**
     * Check if current user owns the resource or is admin
     */
    public boolean isOwnerOrAdmin(Long resourceOwnerId, Authentication auth) {
        if (auth == null || !(auth.getPrincipal() instanceof User user)) {
            return false;
        }

        // Admin can access anything
        if (user.hasRole("ADMIN") || user.hasRole("SUPER_ADMIN")) {
            return true;
        }

        // Check if user owns the resource
        return user.getId().equals(resourceOwnerId);
    }

    /**
     * Check if user owns the order
     */
    public boolean isOrderOwner(Long orderId, Authentication auth) {
        if (auth == null || !(auth.getPrincipal() instanceof User user)) {
            return false;
        }

        if (isAdmin(user)) {
            return true;
        }

        return orderRepository.existsByIdAndUserId(orderId, user.getId());
    }

    /**
     * Check if user owns the product (seller)
     */
    public boolean isProductOwner(Long productId, Authentication auth) {
        if (auth == null || !(auth.getPrincipal() instanceof User user)) {
            return false;
        }

        if (isAdmin(user)) {
            return true;
        }

        return productRepository.existsByIdAndSellerId(productId, user.getId());
    }

    /**
     * Check if user owns the review
     */
    public boolean isReviewOwner(Long reviewId, Authentication auth) {
        if (auth == null || !(auth.getPrincipal() instanceof User user)) {
            return false;
        }

        if (isAdmin(user)) {
            return true;
        }

        return reviewRepository.existsByIdAndUserId(reviewId, user.getId());
    }

    /**
     * Check if user can modify product (owner or admin)
     */
    public boolean canModifyProduct(Long productId, Authentication auth) {
        return isProductOwner(productId, auth);
    }

    /**
     * Check if user can cancel order
     */
    public boolean canCancelOrder(Long orderId, Authentication auth) {
        if (auth == null || !(auth.getPrincipal() instanceof User user)) {
            return false;
        }

        // Admin can cancel any order
        if (isAdmin(user)) {
            return true;
        }

        // Owner can cancel if order is still pending
        if (orderRepository.existsByIdAndUserId(orderId, user.getId())) {
            return orderRepository.existsByIdAndStatus(orderId, "PENDING");
        }

        return false;
    }

    /**
     * Check if user can review product (must have purchased)
     */
    public boolean canReviewProduct(Long productId, Authentication auth) {
        if (auth == null || !(auth.getPrincipal() instanceof User user)) {
            return false;
        }

        // Check if user has purchased this product
        return orderRepository.hasUserPurchasedProduct(user.getId(), productId);
    }

    private boolean isAdmin(User user) {
        return user.hasRole("ADMIN") || user.hasRole("SUPER_ADMIN");
    }
}
```

---

## 7. Controller Examples

### 7.1 ProductController với Permissions

```java
package com.ecommerce.controller;

import com.ecommerce.dto.request.CreateProductRequest;
import com.ecommerce.dto.request.UpdateProductRequest;
import com.ecommerce.dto.response.ApiResponse;
import com.ecommerce.dto.response.ProductDto;
import com.ecommerce.security.annotation.*;
import com.ecommerce.service.ProductService;
import io.swagger.v3.oas.annotations.Operation;
import io.swagger.v3.oas.annotations.security.SecurityRequirement;
import io.swagger.v3.oas.annotations.tags.Tag;
import jakarta.validation.Valid;
import lombok.RequiredArgsConstructor;
import org.springframework.data.domain.Page;
import org.springframework.data.domain.Pageable;
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.security.access.prepost.PreAuthorize;
import org.springframework.web.bind.annotation.*;

@RestController
@RequestMapping("/api/v1/products")
@RequiredArgsConstructor
@Tag(name = "Products", description = "Product management APIs")
public class ProductController {

    private final ProductService productService;

    // ========== Public endpoints ==========

    @GetMapping
    @Operation(summary = "Get all products (public)")
    public ResponseEntity<ApiResponse<Page<ProductDto>>> getAllProducts(Pageable pageable) {
        Page<ProductDto> products = productService.getAllProducts(pageable);
        return ResponseEntity.ok(ApiResponse.success(products));
    }

    @GetMapping("/{id}")
    @Operation(summary = "Get product by ID (public)")
    public ResponseEntity<ApiResponse<ProductDto>> getProduct(@PathVariable Long id) {
        ProductDto product = productService.getProductById(id);
        return ResponseEntity.ok(ApiResponse.success(product));
    }

    // ========== Protected endpoints ==========

    @PostMapping
    @CanCreateProducts  // Custom annotation
    @Operation(summary = "Create product (seller/admin)")
    @SecurityRequirement(name = "bearerAuth")
    public ResponseEntity<ApiResponse<ProductDto>> createProduct(
            @Valid @RequestBody CreateProductRequest request) {

        ProductDto product = productService.createProduct(request);
        return ResponseEntity
                .status(HttpStatus.CREATED)
                .body(ApiResponse.success(product, "Product created"));
    }

    @PutMapping("/{id}")
    @PreAuthorize("@resourceAuthorizationService.canModifyProduct(#id, authentication)")
    @Operation(summary = "Update product (owner/admin)")
    @SecurityRequirement(name = "bearerAuth")
    public ResponseEntity<ApiResponse<ProductDto>> updateProduct(
            @PathVariable Long id,
            @Valid @RequestBody UpdateProductRequest request) {

        ProductDto product = productService.updateProduct(id, request);
        return ResponseEntity.ok(ApiResponse.success(product, "Product updated"));
    }

    @DeleteMapping("/{id}")
    @PreAuthorize("hasAuthority('product:delete') and " +
                  "@resourceAuthorizationService.canModifyProduct(#id, authentication)")
    @Operation(summary = "Delete product (owner with delete permission or admin)")
    @SecurityRequirement(name = "bearerAuth")
    public ResponseEntity<ApiResponse<Void>> deleteProduct(@PathVariable Long id) {
        productService.deleteProduct(id);
        return ResponseEntity.ok(ApiResponse.success(null, "Product deleted"));
    }

    // ========== Admin only ==========

    @PostMapping("/{id}/approve")
    @AdminOnly
    @Operation(summary = "Approve product (admin only)")
    @SecurityRequirement(name = "bearerAuth")
    public ResponseEntity<ApiResponse<ProductDto>> approveProduct(@PathVariable Long id) {
        ProductDto product = productService.approveProduct(id);
        return ResponseEntity.ok(ApiResponse.success(product, "Product approved"));
    }

    @PostMapping("/{id}/feature")
    @PreAuthorize("hasAuthority('admin:settings')")
    @Operation(summary = "Feature product on homepage (admin with settings permission)")
    @SecurityRequirement(name = "bearerAuth")
    public ResponseEntity<ApiResponse<ProductDto>> featureProduct(@PathVariable Long id) {
        ProductDto product = productService.featureProduct(id);
        return ResponseEntity.ok(ApiResponse.success(product, "Product featured"));
    }
}
```

### 7.2 OrderController với Resource-Based Auth

```java
package com.ecommerce.controller;

import com.ecommerce.dto.request.CreateOrderRequest;
import com.ecommerce.dto.response.ApiResponse;
import com.ecommerce.dto.response.OrderDto;
import com.ecommerce.security.annotation.*;
import com.ecommerce.service.OrderService;
import io.swagger.v3.oas.annotations.Operation;
import io.swagger.v3.oas.annotations.security.SecurityRequirement;
import io.swagger.v3.oas.annotations.tags.Tag;
import jakarta.validation.Valid;
import lombok.RequiredArgsConstructor;
import org.springframework.data.domain.Page;
import org.springframework.data.domain.Pageable;
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.security.access.prepost.PreAuthorize;
import org.springframework.web.bind.annotation.*;

@RestController
@RequestMapping("/api/v1/orders")
@RequiredArgsConstructor
@Tag(name = "Orders", description = "Order management APIs")
@SecurityRequirement(name = "bearerAuth")
public class OrderController {

    private final OrderService orderService;

    // Customer creates order
    @PostMapping
    @CanCreateOrders
    @Operation(summary = "Create new order")
    public ResponseEntity<ApiResponse<OrderDto>> createOrder(
            @Valid @RequestBody CreateOrderRequest request) {

        OrderDto order = orderService.createOrder(request);
        return ResponseEntity
                .status(HttpStatus.CREATED)
                .body(ApiResponse.success(order, "Order created"));
    }

    // Get own orders (customer) or all orders (admin)
    @GetMapping
    @PreAuthorize("hasAuthority('order:read')")
    @Operation(summary = "Get orders (own orders for customer, all for admin)")
    public ResponseEntity<ApiResponse<Page<OrderDto>>> getOrders(Pageable pageable) {
        Page<OrderDto> orders = orderService.getOrdersForCurrentUser(pageable);
        return ResponseEntity.ok(ApiResponse.success(orders));
    }

    // Get specific order (owner or admin)
    @GetMapping("/{id}")
    @PreAuthorize("@resourceAuthorizationService.isOrderOwner(#id, authentication)")
    @Operation(summary = "Get order by ID (owner or admin)")
    public ResponseEntity<ApiResponse<OrderDto>> getOrder(@PathVariable Long id) {
        OrderDto order = orderService.getOrderById(id);
        return ResponseEntity.ok(ApiResponse.success(order));
    }

    // Cancel order (owner with pending status, or admin)
    @PostMapping("/{id}/cancel")
    @PreAuthorize("@resourceAuthorizationService.canCancelOrder(#id, authentication)")
    @Operation(summary = "Cancel order")
    public ResponseEntity<ApiResponse<OrderDto>> cancelOrder(@PathVariable Long id) {
        OrderDto order = orderService.cancelOrder(id);
        return ResponseEntity.ok(ApiResponse.success(order, "Order cancelled"));
    }

    // Admin approves order
    @PostMapping("/{id}/approve")
    @CanApproveOrders
    @Operation(summary = "Approve order (admin)")
    public ResponseEntity<ApiResponse<OrderDto>> approveOrder(@PathVariable Long id) {
        OrderDto order = orderService.approveOrder(id);
        return ResponseEntity.ok(ApiResponse.success(order, "Order approved"));
    }

    // Admin updates order status
    @PutMapping("/{id}/status")
    @AdminOnly
    @Operation(summary = "Update order status (admin)")
    public ResponseEntity<ApiResponse<OrderDto>> updateOrderStatus(
            @PathVariable Long id,
            @RequestParam String status) {

        OrderDto order = orderService.updateOrderStatus(id, status);
        return ResponseEntity.ok(ApiResponse.success(order, "Order status updated"));
    }
}
```

---

## 8. Role Management API

### 8.1 RoleController

```java
package com.ecommerce.controller;

import com.ecommerce.dto.request.CreateRoleRequest;
import com.ecommerce.dto.request.UpdateRoleRequest;
import com.ecommerce.dto.response.ApiResponse;
import com.ecommerce.dto.response.RoleDto;
import com.ecommerce.security.annotation.SuperAdminOnly;
import com.ecommerce.service.RoleService;
import io.swagger.v3.oas.annotations.Operation;
import io.swagger.v3.oas.annotations.security.SecurityRequirement;
import io.swagger.v3.oas.annotations.tags.Tag;
import jakarta.validation.Valid;
import lombok.RequiredArgsConstructor;
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

import java.util.List;

@RestController
@RequestMapping("/api/v1/admin/roles")
@RequiredArgsConstructor
@Tag(name = "Role Management", description = "Role and permission management APIs")
@SecurityRequirement(name = "bearerAuth")
@SuperAdminOnly  // All endpoints require SUPER_ADMIN
public class RoleController {

    private final RoleService roleService;

    @GetMapping
    @Operation(summary = "Get all roles")
    public ResponseEntity<ApiResponse<List<RoleDto>>> getAllRoles() {
        List<RoleDto> roles = roleService.getAllRoles();
        return ResponseEntity.ok(ApiResponse.success(roles));
    }

    @GetMapping("/{id}")
    @Operation(summary = "Get role by ID")
    public ResponseEntity<ApiResponse<RoleDto>> getRole(@PathVariable Long id) {
        RoleDto role = roleService.getRoleById(id);
        return ResponseEntity.ok(ApiResponse.success(role));
    }

    @PostMapping
    @Operation(summary = "Create new role")
    public ResponseEntity<ApiResponse<RoleDto>> createRole(
            @Valid @RequestBody CreateRoleRequest request) {

        RoleDto role = roleService.createRole(request);
        return ResponseEntity
                .status(HttpStatus.CREATED)
                .body(ApiResponse.success(role, "Role created"));
    }

    @PutMapping("/{id}")
    @Operation(summary = "Update role")
    public ResponseEntity<ApiResponse<RoleDto>> updateRole(
            @PathVariable Long id,
            @Valid @RequestBody UpdateRoleRequest request) {

        RoleDto role = roleService.updateRole(id, request);
        return ResponseEntity.ok(ApiResponse.success(role, "Role updated"));
    }

    @DeleteMapping("/{id}")
    @Operation(summary = "Delete role (non-system roles only)")
    public ResponseEntity<ApiResponse<Void>> deleteRole(@PathVariable Long id) {
        roleService.deleteRole(id);
        return ResponseEntity.ok(ApiResponse.success(null, "Role deleted"));
    }

    @PostMapping("/{roleId}/permissions/{permissionId}")
    @Operation(summary = "Add permission to role")
    public ResponseEntity<ApiResponse<RoleDto>> addPermission(
            @PathVariable Long roleId,
            @PathVariable Long permissionId) {

        RoleDto role = roleService.addPermission(roleId, permissionId);
        return ResponseEntity.ok(ApiResponse.success(role, "Permission added"));
    }

    @DeleteMapping("/{roleId}/permissions/{permissionId}")
    @Operation(summary = "Remove permission from role")
    public ResponseEntity<ApiResponse<RoleDto>> removePermission(
            @PathVariable Long roleId,
            @PathVariable Long permissionId) {

        RoleDto role = roleService.removePermission(roleId, permissionId);
        return ResponseEntity.ok(ApiResponse.success(role, "Permission removed"));
    }
}
```

---

## 9. Permission Matrix

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         PERMISSION MATRIX                                    │
├─────────────────────────┬──────────┬──────────┬──────────┬─────────────────┤
│ Permission              │ CUSTOMER │  SELLER  │  ADMIN   │  SUPER_ADMIN    │
├─────────────────────────┼──────────┼──────────┼──────────┼─────────────────┤
│ product:read            │    ✓     │    ✓     │    ✓     │       ✓         │
│ product:create          │    ✗     │    ✓     │    ✓     │       ✓         │
│ product:update          │    ✗     │   own    │    ✓     │       ✓         │
│ product:delete          │    ✗     │   own    │    ✓     │       ✓         │
├─────────────────────────┼──────────┼──────────┼──────────┼─────────────────┤
│ order:read              │   own    │   own    │    ✓     │       ✓         │
│ order:create            │    ✓     │    ✗     │    ✓     │       ✓         │
│ order:update            │    ✗     │   own    │    ✓     │       ✓         │
│ order:cancel            │   own*   │    ✗     │    ✓     │       ✓         │
│ order:approve           │    ✗     │    ✗     │    ✓     │       ✓         │
├─────────────────────────┼──────────┼──────────┼──────────┼─────────────────┤
│ user:read               │   self   │   self   │    ✓     │       ✓         │
│ user:create             │    ✗     │    ✗     │    ✓     │       ✓         │
│ user:update             │   self   │   self   │    ✓     │       ✓         │
│ user:delete             │    ✗     │    ✗     │    ✓     │       ✓         │
├─────────────────────────┼──────────┼──────────┼──────────┼─────────────────┤
│ review:read             │    ✓     │    ✓     │    ✓     │       ✓         │
│ review:create           │  buyer   │    ✗     │    ✓     │       ✓         │
│ review:update           │   own    │    ✗     │    ✓     │       ✓         │
│ review:delete           │    ✗     │    ✗     │    ✓     │       ✓         │
│ review:moderate         │    ✗     │    ✗     │    ✓     │       ✓         │
├─────────────────────────┼──────────┼──────────┼──────────┼─────────────────┤
│ admin:dashboard         │    ✗     │    ✗     │    ✓     │       ✓         │
│ admin:settings          │    ✗     │    ✗     │    ✗     │       ✓         │
│ admin:reports           │    ✗     │    ✗     │    ✓     │       ✓         │
└─────────────────────────┴──────────┴──────────┴──────────┴─────────────────┘

Legend:
  ✓    = Full access
  ✗    = No access
  own  = Only own resources
  self = Only own profile
  own* = Only own resources with status condition
  buyer= Must have purchased the product
```

---

## 10. Testing Authorization

### 10.1 Authorization Tests

```java
package com.ecommerce.controller;

import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.autoconfigure.web.servlet.AutoConfigureMockMvc;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.security.test.context.support.WithMockUser;
import org.springframework.test.web.servlet.MockMvc;

import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.*;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.*;

@SpringBootTest
@AutoConfigureMockMvc
class ProductControllerAuthorizationTest {

    @Autowired
    private MockMvc mockMvc;

    @Test
    void getProducts_shouldBePublic() throws Exception {
        mockMvc.perform(get("/api/v1/products"))
                .andExpect(status().isOk());
    }

    @Test
    void createProduct_withoutAuth_shouldReturn401() throws Exception {
        mockMvc.perform(post("/api/v1/products")
                        .contentType("application/json")
                        .content("{}"))
                .andExpect(status().isUnauthorized());
    }

    @Test
    @WithMockUser(authorities = {"product:create"})
    void createProduct_withPermission_shouldReturn201() throws Exception {
        mockMvc.perform(post("/api/v1/products")
                        .contentType("application/json")
                        .content("{\"name\":\"Test\",\"price\":100}"))
                .andExpect(status().isCreated());
    }

    @Test
    @WithMockUser(authorities = {"product:read"})  // Missing product:create
    void createProduct_withoutPermission_shouldReturn403() throws Exception {
        mockMvc.perform(post("/api/v1/products")
                        .contentType("application/json")
                        .content("{\"name\":\"Test\",\"price\":100}"))
                .andExpect(status().isForbidden());
    }

    @Test
    @WithMockUser(roles = {"ADMIN"})
    void approveProduct_asAdmin_shouldSucceed() throws Exception {
        mockMvc.perform(post("/api/v1/products/1/approve"))
                .andExpect(status().isOk());
    }

    @Test
    @WithMockUser(roles = {"CUSTOMER"})
    void approveProduct_asCustomer_shouldReturn403() throws Exception {
        mockMvc.perform(post("/api/v1/products/1/approve"))
                .andExpect(status().isForbidden());
    }
}
```

---

## 11. Bài tập thực hành

### Bài tập 1: RBAC Setup
- [x] Create Permission entity
- [x] Create Role entity with ManyToMany
- [x] Update User entity với roles
- [x] Seed default roles and permissions

### Bài tập 2: Custom Annotations
- [x] Create permission-based annotations
- [x] Create @AdminOnly annotation
- [x] Test annotations on controllers

### Bài tập 3: Resource Authorization
- [x] Implement ResourceAuthorizationService
- [x] Add owner checks cho Orders
- [x] Add seller checks cho Products

---

## 12. Checklist

- [ ] Create Permission và Role entities
- [ ] Create database migrations
- [ ] Seed default roles/permissions
- [ ] Update User entity với role relationship
- [ ] Create custom security annotations
- [ ] Implement ResourceAuthorizationService
- [ ] Apply permissions to controllers
- [ ] Create Role management API
- [ ] Write authorization tests
- [ ] Document permission matrix

---

## Điều hướng

- [← Day 18: User Registration](./day-18-user-registration.md)
- [Day 20: Refresh Token & OAuth2 →](./day-20-oauth2-refresh.md)
- [Về trang chính](../00-overview.md)
