# Day 21: Shopping Cart Implementation

## Mục tiêu học tập
- Thiết kế Cart system (Session vs Database)
- Implement Cart operations (add, update, remove)
- Handle inventory checking
- Merge anonymous cart với user cart

---

## 1. Cart Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                    CART ARCHITECTURE OPTIONS                     │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │ OPTION 1: SESSION-BASED CART                              │   │
│  ├──────────────────────────────────────────────────────────┤   │
│  │ ✓ Simple implementation                                   │   │
│  │ ✓ No database writes for each cart update                 │   │
│  │ ✗ Lost if session expires                                 │   │
│  │ ✗ Cannot access from multiple devices                     │   │
│  │ ✗ Limited session storage                                 │   │
│  └──────────────────────────────────────────────────────────┘   │
│                                                                  │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │ OPTION 2: DATABASE CART (Recommended)                     │   │
│  ├──────────────────────────────────────────────────────────┤   │
│  │ ✓ Persistent across sessions                              │   │
│  │ ✓ Accessible from multiple devices                        │   │
│  │ ✓ Can handle large carts                                  │   │
│  │ ✓ Analytics on abandoned carts                            │   │
│  │ ✗ More database operations                                │   │
│  └──────────────────────────────────────────────────────────┘   │
│                                                                  │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │ OPTION 3: REDIS CART (Best for Scale)                     │   │
│  ├──────────────────────────────────────────────────────────┤   │
│  │ ✓ Very fast read/write                                    │   │
│  │ ✓ TTL for auto-cleanup                                    │   │
│  │ ✓ Atomic operations                                       │   │
│  │ ✓ Can handle anonymous users                              │   │
│  │ ✗ Need Redis infrastructure                               │   │
│  └──────────────────────────────────────────────────────────┘   │
│                                                                  │
│  → We'll implement: Database + Redis hybrid approach            │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## 2. Cart Entities

### 2.1 Cart Entity

```java
package com.ecommerce.domain.entity;

import jakarta.persistence.*;
import lombok.*;

import java.math.BigDecimal;
import java.time.LocalDateTime;
import java.util.ArrayList;
import java.util.List;
import java.util.Optional;

@Entity
@Table(name = "carts")
@Getter
@Setter
@NoArgsConstructor
@AllArgsConstructor
@Builder
public class Cart extends BaseEntity {

    @OneToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "user_id")
    private User user;

    @Column(name = "session_id", length = 100)
    private String sessionId;  // For anonymous users

    @OneToMany(mappedBy = "cart", cascade = CascadeType.ALL, orphanRemoval = true)
    @Builder.Default
    private List<CartItem> items = new ArrayList<>();

    @Column(name = "last_activity")
    private LocalDateTime lastActivity;

    @Column(length = 20)
    private String currency = "USD";

    // ========== Calculated Fields ==========

    @Transient
    public int getTotalItems() {
        return items.stream()
                .mapToInt(CartItem::getQuantity)
                .sum();
    }

    @Transient
    public int getUniqueItemsCount() {
        return items.size();
    }

    @Transient
    public BigDecimal getSubtotal() {
        return items.stream()
                .map(CartItem::getSubtotal)
                .reduce(BigDecimal.ZERO, BigDecimal::add);
    }

    // ========== Cart Operations ==========

    public CartItem addItem(Product product, int quantity) {
        // Check if product already in cart
        Optional<CartItem> existingItem = findItemByProductId(product.getId());

        if (existingItem.isPresent()) {
            CartItem item = existingItem.get();
            item.setQuantity(item.getQuantity() + quantity);
            return item;
        }

        // Create new item
        CartItem newItem = CartItem.builder()
                .cart(this)
                .product(product)
                .quantity(quantity)
                .priceAtAdd(product.getPrice())
                .build();

        items.add(newItem);
        updateLastActivity();
        return newItem;
    }

    public void updateItemQuantity(Long productId, int quantity) {
        findItemByProductId(productId).ifPresent(item -> {
            if (quantity <= 0) {
                removeItem(productId);
            } else {
                item.setQuantity(quantity);
            }
        });
        updateLastActivity();
    }

    public void removeItem(Long productId) {
        items.removeIf(item -> item.getProduct().getId().equals(productId));
        updateLastActivity();
    }

    public void clear() {
        items.clear();
        updateLastActivity();
    }

    public Optional<CartItem> findItemByProductId(Long productId) {
        return items.stream()
                .filter(item -> item.getProduct().getId().equals(productId))
                .findFirst();
    }

    public boolean isEmpty() {
        return items.isEmpty();
    }

    public void updateLastActivity() {
        this.lastActivity = LocalDateTime.now();
    }

    // Merge anonymous cart into user cart
    public void mergeWith(Cart anonymousCart) {
        for (CartItem item : anonymousCart.getItems()) {
            Optional<CartItem> existing = findItemByProductId(item.getProduct().getId());

            if (existing.isPresent()) {
                // Add quantities
                existing.get().setQuantity(
                        existing.get().getQuantity() + item.getQuantity()
                );
            } else {
                // Move item to this cart
                CartItem newItem = CartItem.builder()
                        .cart(this)
                        .product(item.getProduct())
                        .quantity(item.getQuantity())
                        .priceAtAdd(item.getPriceAtAdd())
                        .build();
                items.add(newItem);
            }
        }
        updateLastActivity();
    }
}
```

### 2.2 CartItem Entity

```java
package com.ecommerce.domain.entity;

import jakarta.persistence.*;
import lombok.*;

import java.math.BigDecimal;

@Entity
@Table(name = "cart_items", uniqueConstraints = {
    @UniqueConstraint(columnNames = {"cart_id", "product_id"})
})
@Getter
@Setter
@NoArgsConstructor
@AllArgsConstructor
@Builder
public class CartItem extends BaseEntity {

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "cart_id", nullable = false)
    private Cart cart;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "product_id", nullable = false)
    private Product product;

    @Column(nullable = false)
    private int quantity;

    @Column(name = "price_at_add", nullable = false, precision = 10, scale = 2)
    private BigDecimal priceAtAdd;  // Price khi add vào cart

    // ========== Calculated ==========

    @Transient
    public BigDecimal getCurrentPrice() {
        return product.getPrice();
    }

    @Transient
    public BigDecimal getSubtotal() {
        return getCurrentPrice().multiply(BigDecimal.valueOf(quantity));
    }

    @Transient
    public boolean isPriceChanged() {
        return !getCurrentPrice().equals(priceAtAdd);
    }

    @Transient
    public BigDecimal getPriceDifference() {
        return getCurrentPrice().subtract(priceAtAdd);
    }

    // ========== Validation ==========

    @Transient
    public boolean isAvailable() {
        return product.isActive() && product.getStockQuantity() >= quantity;
    }

    @Transient
    public int getMaxAvailableQuantity() {
        return product.getStockQuantity();
    }

    public void adjustQuantityToAvailable() {
        if (quantity > getMaxAvailableQuantity()) {
            this.quantity = getMaxAvailableQuantity();
        }
    }
}
```

### 2.3 Flyway Migration

```sql
-- V11__Create_cart_tables.sql

CREATE TABLE carts (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    user_id BIGINT UNIQUE,
    session_id VARCHAR(100),
    last_activity DATETIME,
    currency VARCHAR(20) DEFAULT 'USD',
    created_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at DATETIME,
    deleted BOOLEAN DEFAULT FALSE,

    CONSTRAINT fk_cart_user
        FOREIGN KEY (user_id) REFERENCES users(id)
        ON DELETE CASCADE,

    INDEX idx_cart_session (session_id),
    INDEX idx_cart_last_activity (last_activity)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;

CREATE TABLE cart_items (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    cart_id BIGINT NOT NULL,
    product_id BIGINT NOT NULL,
    quantity INT NOT NULL DEFAULT 1,
    price_at_add DECIMAL(10, 2) NOT NULL,
    created_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at DATETIME,
    deleted BOOLEAN DEFAULT FALSE,

    CONSTRAINT fk_cart_item_cart
        FOREIGN KEY (cart_id) REFERENCES carts(id)
        ON DELETE CASCADE,

    CONSTRAINT fk_cart_item_product
        FOREIGN KEY (product_id) REFERENCES products(id)
        ON DELETE CASCADE,

    UNIQUE KEY uk_cart_product (cart_id, product_id),
    INDEX idx_cart_item_product (product_id)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
```

---

## 3. Cart DTOs

### 3.1 Request DTOs

```java
package com.ecommerce.dto.request;

import jakarta.validation.constraints.Min;
import jakarta.validation.constraints.NotNull;
import lombok.Data;

@Data
public class AddToCartRequest {

    @NotNull(message = "Product ID is required")
    private Long productId;

    @Min(value = 1, message = "Quantity must be at least 1")
    private int quantity = 1;
}
```

```java
package com.ecommerce.dto.request;

import jakarta.validation.constraints.Min;
import jakarta.validation.constraints.NotNull;
import lombok.Data;

@Data
public class UpdateCartItemRequest {

    @NotNull(message = "Product ID is required")
    private Long productId;

    @Min(value = 0, message = "Quantity cannot be negative")
    private int quantity;  // 0 = remove
}
```

### 3.2 Response DTOs

```java
package com.ecommerce.dto.response;

import lombok.AllArgsConstructor;
import lombok.Builder;
import lombok.Data;
import lombok.NoArgsConstructor;

import java.math.BigDecimal;
import java.util.List;

@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class CartDto {

    private Long id;
    private List<CartItemDto> items;
    private int totalItems;
    private int uniqueItemsCount;
    private BigDecimal subtotal;
    private String currency;
    private List<CartWarning> warnings;

    @Data
    @Builder
    @NoArgsConstructor
    @AllArgsConstructor
    public static class CartItemDto {
        private Long id;
        private ProductSummaryDto product;
        private int quantity;
        private BigDecimal priceAtAdd;
        private BigDecimal currentPrice;
        private BigDecimal subtotal;
        private boolean priceChanged;
        private BigDecimal priceDifference;
        private boolean available;
        private int maxAvailable;
    }

    @Data
    @Builder
    @NoArgsConstructor
    @AllArgsConstructor
    public static class ProductSummaryDto {
        private Long id;
        private String name;
        private String slug;
        private String imageUrl;
        private BigDecimal price;
        private int stockQuantity;
        private boolean active;
    }

    @Data
    @Builder
    @NoArgsConstructor
    @AllArgsConstructor
    public static class CartWarning {
        private String type;  // PRICE_CHANGED, OUT_OF_STOCK, LOW_STOCK, QUANTITY_ADJUSTED
        private Long productId;
        private String message;
    }
}
```

---

## 4. Cart Repository

```java
package com.ecommerce.repository;

import com.ecommerce.domain.entity.Cart;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.data.jpa.repository.Modifying;
import org.springframework.data.jpa.repository.Query;
import org.springframework.data.repository.query.Param;
import org.springframework.stereotype.Repository;

import java.time.LocalDateTime;
import java.util.List;
import java.util.Optional;

@Repository
public interface CartRepository extends JpaRepository<Cart, Long> {

    Optional<Cart> findByUserId(Long userId);

    Optional<Cart> findBySessionId(String sessionId);

    @Query("SELECT c FROM Cart c LEFT JOIN FETCH c.items ci " +
           "LEFT JOIN FETCH ci.product WHERE c.user.id = :userId")
    Optional<Cart> findByUserIdWithItems(@Param("userId") Long userId);

    @Query("SELECT c FROM Cart c LEFT JOIN FETCH c.items ci " +
           "LEFT JOIN FETCH ci.product WHERE c.sessionId = :sessionId")
    Optional<Cart> findBySessionIdWithItems(@Param("sessionId") String sessionId);

    @Query("SELECT c FROM Cart c WHERE c.user IS NULL " +
           "AND c.lastActivity < :before")
    List<Cart> findAbandonedAnonymousCarts(@Param("before") LocalDateTime before);

    @Query("SELECT c FROM Cart c WHERE c.user IS NOT NULL " +
           "AND c.lastActivity < :before AND SIZE(c.items) > 0")
    List<Cart> findAbandonedUserCarts(@Param("before") LocalDateTime before);

    @Modifying
    @Query("DELETE FROM Cart c WHERE c.user IS NULL AND c.lastActivity < :before")
    int deleteOldAnonymousCarts(@Param("before") LocalDateTime before);

    boolean existsByUserId(Long userId);
}
```

---

## 5. Cart Service

### 5.1 CartService Interface

```java
package com.ecommerce.service;

import com.ecommerce.dto.request.AddToCartRequest;
import com.ecommerce.dto.request.UpdateCartItemRequest;
import com.ecommerce.dto.response.CartDto;

public interface CartService {

    CartDto getCart();

    CartDto addToCart(AddToCartRequest request);

    CartDto updateCartItem(UpdateCartItemRequest request);

    CartDto removeFromCart(Long productId);

    CartDto clearCart();

    void mergeAnonymousCart(String sessionId);

    int getCartItemCount();
}
```

### 5.2 CartServiceImpl

```java
package com.ecommerce.service.impl;

import com.ecommerce.domain.entity.Cart;
import com.ecommerce.domain.entity.CartItem;
import com.ecommerce.domain.entity.Product;
import com.ecommerce.domain.entity.User;
import com.ecommerce.dto.request.AddToCartRequest;
import com.ecommerce.dto.request.UpdateCartItemRequest;
import com.ecommerce.dto.response.CartDto;
import com.ecommerce.exception.BadRequestException;
import com.ecommerce.exception.ResourceNotFoundException;
import com.ecommerce.mapper.CartMapper;
import com.ecommerce.repository.CartRepository;
import com.ecommerce.repository.ProductRepository;
import com.ecommerce.service.CartService;
import com.ecommerce.util.SecurityUtils;
import com.ecommerce.util.SessionUtils;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.util.ArrayList;
import java.util.List;
import java.util.Optional;

@Service
@RequiredArgsConstructor
@Slf4j
public class CartServiceImpl implements CartService {

    private final CartRepository cartRepository;
    private final ProductRepository productRepository;
    private final CartMapper cartMapper;
    private final SessionUtils sessionUtils;

    @Override
    @Transactional(readOnly = true)
    public CartDto getCart() {
        Cart cart = getOrCreateCart();
        return buildCartDtoWithWarnings(cart);
    }

    @Override
    @Transactional
    public CartDto addToCart(AddToCartRequest request) {
        Cart cart = getOrCreateCart();

        // Get product
        Product product = productRepository.findById(request.getProductId())
                .orElseThrow(() -> new ResourceNotFoundException("Product", "id", request.getProductId()));

        // Validate product
        if (!product.isActive()) {
            throw new BadRequestException("Product is not available");
        }

        // Check stock
        Optional<CartItem> existingItem = cart.findItemByProductId(product.getId());
        int currentQty = existingItem.map(CartItem::getQuantity).orElse(0);
        int requestedTotal = currentQty + request.getQuantity();

        if (requestedTotal > product.getStockQuantity()) {
            throw new BadRequestException(
                    String.format("Only %d items available in stock", product.getStockQuantity())
            );
        }

        // Add to cart
        cart.addItem(product, request.getQuantity());
        cartRepository.save(cart);

        log.info("Added {} x {} to cart", request.getQuantity(), product.getName());

        return buildCartDtoWithWarnings(cart);
    }

    @Override
    @Transactional
    public CartDto updateCartItem(UpdateCartItemRequest request) {
        Cart cart = getOrCreateCart();

        // Find cart item
        CartItem item = cart.findItemByProductId(request.getProductId())
                .orElseThrow(() -> new ResourceNotFoundException(
                        "Cart item", "productId", request.getProductId()));

        if (request.getQuantity() == 0) {
            // Remove item
            cart.removeItem(request.getProductId());
        } else {
            // Validate stock
            Product product = item.getProduct();
            if (request.getQuantity() > product.getStockQuantity()) {
                throw new BadRequestException(
                        String.format("Only %d items available in stock", product.getStockQuantity())
                );
            }
            cart.updateItemQuantity(request.getProductId(), request.getQuantity());
        }

        cartRepository.save(cart);

        return buildCartDtoWithWarnings(cart);
    }

    @Override
    @Transactional
    public CartDto removeFromCart(Long productId) {
        Cart cart = getOrCreateCart();
        cart.removeItem(productId);
        cartRepository.save(cart);

        log.info("Removed product {} from cart", productId);

        return buildCartDtoWithWarnings(cart);
    }

    @Override
    @Transactional
    public CartDto clearCart() {
        Cart cart = getOrCreateCart();
        cart.clear();
        cartRepository.save(cart);

        log.info("Cart cleared");

        return cartMapper.toDto(cart);
    }

    @Override
    @Transactional
    public void mergeAnonymousCart(String sessionId) {
        Optional<User> currentUser = SecurityUtils.getCurrentUser();
        if (currentUser.isEmpty()) {
            return;
        }

        // Find anonymous cart
        Optional<Cart> anonymousCart = cartRepository.findBySessionIdWithItems(sessionId);
        if (anonymousCart.isEmpty() || anonymousCart.get().isEmpty()) {
            return;
        }

        // Get or create user cart
        Cart userCart = cartRepository.findByUserIdWithItems(currentUser.get().getId())
                .orElseGet(() -> {
                    Cart newCart = new Cart();
                    newCart.setUser(currentUser.get());
                    return newCart;
                });

        // Merge carts
        userCart.mergeWith(anonymousCart.get());
        cartRepository.save(userCart);

        // Delete anonymous cart
        cartRepository.delete(anonymousCart.get());

        log.info("Merged anonymous cart {} into user cart", sessionId);
    }

    @Override
    @Transactional(readOnly = true)
    public int getCartItemCount() {
        Cart cart = getOrCreateCart();
        return cart.getTotalItems();
    }

    // ========== Private Methods ==========

    private Cart getOrCreateCart() {
        Optional<User> currentUser = SecurityUtils.getCurrentUser();

        if (currentUser.isPresent()) {
            // Logged in user
            return cartRepository.findByUserIdWithItems(currentUser.get().getId())
                    .orElseGet(() -> createNewCart(currentUser.get(), null));
        } else {
            // Anonymous user
            String sessionId = sessionUtils.getCurrentSessionId();
            return cartRepository.findBySessionIdWithItems(sessionId)
                    .orElseGet(() -> createNewCart(null, sessionId));
        }
    }

    private Cart createNewCart(User user, String sessionId) {
        Cart cart = Cart.builder()
                .user(user)
                .sessionId(sessionId)
                .items(new ArrayList<>())
                .build();
        cart.updateLastActivity();
        return cartRepository.save(cart);
    }

    private CartDto buildCartDtoWithWarnings(Cart cart) {
        CartDto dto = cartMapper.toDto(cart);

        // Build warnings
        List<CartDto.CartWarning> warnings = new ArrayList<>();

        for (CartItem item : cart.getItems()) {
            Product product = item.getProduct();

            // Check if product is still available
            if (!product.isActive()) {
                warnings.add(CartDto.CartWarning.builder()
                        .type("OUT_OF_STOCK")
                        .productId(product.getId())
                        .message(product.getName() + " is no longer available")
                        .build());
            }
            // Check if price changed
            else if (item.isPriceChanged()) {
                warnings.add(CartDto.CartWarning.builder()
                        .type("PRICE_CHANGED")
                        .productId(product.getId())
                        .message(String.format("%s price has changed from $%.2f to $%.2f",
                                product.getName(),
                                item.getPriceAtAdd(),
                                item.getCurrentPrice()))
                        .build());
            }
            // Check if quantity exceeds stock
            else if (item.getQuantity() > product.getStockQuantity()) {
                warnings.add(CartDto.CartWarning.builder()
                        .type("QUANTITY_ADJUSTED")
                        .productId(product.getId())
                        .message(String.format("Only %d of %s available",
                                product.getStockQuantity(),
                                product.getName()))
                        .build());
            }
            // Check low stock
            else if (product.getStockQuantity() <= 5) {
                warnings.add(CartDto.CartWarning.builder()
                        .type("LOW_STOCK")
                        .productId(product.getId())
                        .message(String.format("Only %d left in stock",
                                product.getStockQuantity()))
                        .build());
            }
        }

        dto.setWarnings(warnings);
        return dto;
    }
}
```

---

## 6. Cart Mapper

```java
package com.ecommerce.mapper;

import com.ecommerce.domain.entity.Cart;
import com.ecommerce.domain.entity.CartItem;
import com.ecommerce.domain.entity.Product;
import com.ecommerce.dto.response.CartDto;
import org.mapstruct.*;

import java.util.List;

@Mapper(componentModel = "spring")
public interface CartMapper {

    @Mapping(target = "totalItems", expression = "java(cart.getTotalItems())")
    @Mapping(target = "uniqueItemsCount", expression = "java(cart.getUniqueItemsCount())")
    @Mapping(target = "subtotal", expression = "java(cart.getSubtotal())")
    @Mapping(target = "warnings", ignore = true)
    CartDto toDto(Cart cart);

    @Mapping(target = "product", source = "product")
    @Mapping(target = "currentPrice", expression = "java(item.getCurrentPrice())")
    @Mapping(target = "subtotal", expression = "java(item.getSubtotal())")
    @Mapping(target = "priceChanged", expression = "java(item.isPriceChanged())")
    @Mapping(target = "priceDifference", expression = "java(item.getPriceDifference())")
    @Mapping(target = "available", expression = "java(item.isAvailable())")
    @Mapping(target = "maxAvailable", expression = "java(item.getMaxAvailableQuantity())")
    CartDto.CartItemDto toItemDto(CartItem item);

    @Mapping(target = "imageUrl", source = "thumbnailUrl")
    @Mapping(target = "active", source = "active")
    CartDto.ProductSummaryDto toProductSummary(Product product);

    List<CartDto.CartItemDto> toItemDtoList(List<CartItem> items);
}
```

---

## 7. Cart Controller

```java
package com.ecommerce.controller;

import com.ecommerce.dto.request.AddToCartRequest;
import com.ecommerce.dto.request.UpdateCartItemRequest;
import com.ecommerce.dto.response.ApiResponse;
import com.ecommerce.dto.response.CartDto;
import com.ecommerce.service.CartService;
import io.swagger.v3.oas.annotations.Operation;
import io.swagger.v3.oas.annotations.tags.Tag;
import jakarta.validation.Valid;
import lombok.RequiredArgsConstructor;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

@RestController
@RequestMapping("/api/v1/cart")
@RequiredArgsConstructor
@Tag(name = "Shopping Cart", description = "Cart management APIs")
public class CartController {

    private final CartService cartService;

    @GetMapping
    @Operation(summary = "Get current cart")
    public ResponseEntity<ApiResponse<CartDto>> getCart() {
        CartDto cart = cartService.getCart();
        return ResponseEntity.ok(ApiResponse.success(cart));
    }

    @PostMapping("/items")
    @Operation(summary = "Add item to cart")
    public ResponseEntity<ApiResponse<CartDto>> addToCart(
            @Valid @RequestBody AddToCartRequest request) {

        CartDto cart = cartService.addToCart(request);
        return ResponseEntity.ok(ApiResponse.success(cart, "Item added to cart"));
    }

    @PutMapping("/items")
    @Operation(summary = "Update cart item quantity")
    public ResponseEntity<ApiResponse<CartDto>> updateCartItem(
            @Valid @RequestBody UpdateCartItemRequest request) {

        CartDto cart = cartService.updateCartItem(request);
        return ResponseEntity.ok(ApiResponse.success(cart, "Cart updated"));
    }

    @DeleteMapping("/items/{productId}")
    @Operation(summary = "Remove item from cart")
    public ResponseEntity<ApiResponse<CartDto>> removeFromCart(
            @PathVariable Long productId) {

        CartDto cart = cartService.removeFromCart(productId);
        return ResponseEntity.ok(ApiResponse.success(cart, "Item removed from cart"));
    }

    @DeleteMapping
    @Operation(summary = "Clear cart")
    public ResponseEntity<ApiResponse<CartDto>> clearCart() {
        CartDto cart = cartService.clearCart();
        return ResponseEntity.ok(ApiResponse.success(cart, "Cart cleared"));
    }

    @GetMapping("/count")
    @Operation(summary = "Get cart item count")
    public ResponseEntity<ApiResponse<Integer>> getCartItemCount() {
        int count = cartService.getCartItemCount();
        return ResponseEntity.ok(ApiResponse.success(count));
    }

    @PostMapping("/merge")
    @Operation(summary = "Merge anonymous cart after login")
    public ResponseEntity<ApiResponse<CartDto>> mergeCart(
            @RequestParam String sessionId) {

        cartService.mergeAnonymousCart(sessionId);
        CartDto cart = cartService.getCart();
        return ResponseEntity.ok(ApiResponse.success(cart, "Cart merged"));
    }
}
```

---

## 8. Session Utilities

```java
package com.ecommerce.util;

import jakarta.servlet.http.Cookie;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;
import lombok.RequiredArgsConstructor;
import org.springframework.stereotype.Component;
import org.springframework.web.context.request.RequestContextHolder;
import org.springframework.web.context.request.ServletRequestAttributes;

import java.util.Arrays;
import java.util.UUID;

@Component
@RequiredArgsConstructor
public class SessionUtils {

    private static final String CART_SESSION_COOKIE = "cart_session";
    private static final int COOKIE_MAX_AGE = 30 * 24 * 60 * 60; // 30 days

    public String getCurrentSessionId() {
        HttpServletRequest request = getCurrentRequest();
        HttpServletResponse response = getCurrentResponse();

        // Check existing cookie
        Cookie[] cookies = request.getCookies();
        if (cookies != null) {
            return Arrays.stream(cookies)
                    .filter(c -> CART_SESSION_COOKIE.equals(c.getName()))
                    .findFirst()
                    .map(Cookie::getValue)
                    .orElseGet(() -> createSessionCookie(response));
        }

        return createSessionCookie(response);
    }

    private String createSessionCookie(HttpServletResponse response) {
        String sessionId = UUID.randomUUID().toString();

        Cookie cookie = new Cookie(CART_SESSION_COOKIE, sessionId);
        cookie.setMaxAge(COOKIE_MAX_AGE);
        cookie.setPath("/");
        cookie.setHttpOnly(true);
        cookie.setSecure(true); // Enable in production
        response.addCookie(cookie);

        return sessionId;
    }

    public void clearSessionCookie() {
        HttpServletResponse response = getCurrentResponse();
        Cookie cookie = new Cookie(CART_SESSION_COOKIE, "");
        cookie.setMaxAge(0);
        cookie.setPath("/");
        response.addCookie(cookie);
    }

    private HttpServletRequest getCurrentRequest() {
        ServletRequestAttributes attrs = (ServletRequestAttributes)
                RequestContextHolder.currentRequestAttributes();
        return attrs.getRequest();
    }

    private HttpServletResponse getCurrentResponse() {
        ServletRequestAttributes attrs = (ServletRequestAttributes)
                RequestContextHolder.currentRequestAttributes();
        return attrs.getResponse();
    }
}
```

---

## 9. Redis Cart (Optional - High Performance)

### 9.1 Redis Cart Structure

```java
package com.ecommerce.dto.cache;

import lombok.AllArgsConstructor;
import lombok.Builder;
import lombok.Data;
import lombok.NoArgsConstructor;

import java.io.Serializable;
import java.math.BigDecimal;
import java.util.HashMap;
import java.util.Map;

@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class RedisCart implements Serializable {

    private String cartId;
    private Long userId;
    private String sessionId;

    @Builder.Default
    private Map<Long, RedisCartItem> items = new HashMap<>();

    private long lastUpdated;

    @Data
    @Builder
    @NoArgsConstructor
    @AllArgsConstructor
    public static class RedisCartItem implements Serializable {
        private Long productId;
        private String productName;
        private String imageUrl;
        private int quantity;
        private BigDecimal price;
        private BigDecimal priceAtAdd;
        private int stockQuantity;
    }

    public void addItem(RedisCartItem item) {
        Long productId = item.getProductId();
        if (items.containsKey(productId)) {
            RedisCartItem existing = items.get(productId);
            existing.setQuantity(existing.getQuantity() + item.getQuantity());
        } else {
            items.put(productId, item);
        }
        updateTimestamp();
    }

    public void updateQuantity(Long productId, int quantity) {
        if (quantity <= 0) {
            items.remove(productId);
        } else if (items.containsKey(productId)) {
            items.get(productId).setQuantity(quantity);
        }
        updateTimestamp();
    }

    public void removeItem(Long productId) {
        items.remove(productId);
        updateTimestamp();
    }

    public void clear() {
        items.clear();
        updateTimestamp();
    }

    private void updateTimestamp() {
        this.lastUpdated = System.currentTimeMillis();
    }

    public BigDecimal getSubtotal() {
        return items.values().stream()
                .map(item -> item.getPrice().multiply(BigDecimal.valueOf(item.getQuantity())))
                .reduce(BigDecimal.ZERO, BigDecimal::add);
    }

    public int getTotalItems() {
        return items.values().stream()
                .mapToInt(RedisCartItem::getQuantity)
                .sum();
    }
}
```

### 9.2 Redis Cart Service

```java
package com.ecommerce.service.impl;

import com.ecommerce.dto.cache.RedisCart;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.data.redis.core.RedisTemplate;
import org.springframework.stereotype.Service;

import java.util.concurrent.TimeUnit;

@Service
@RequiredArgsConstructor
@Slf4j
public class RedisCartService {

    private final RedisTemplate<String, RedisCart> redisTemplate;

    private static final String CART_KEY_PREFIX = "cart:";
    private static final long CART_TTL_DAYS = 30;

    public RedisCart getCart(String cartId) {
        String key = CART_KEY_PREFIX + cartId;
        RedisCart cart = redisTemplate.opsForValue().get(key);

        if (cart == null) {
            cart = RedisCart.builder()
                    .cartId(cartId)
                    .build();
        }

        return cart;
    }

    public void saveCart(RedisCart cart) {
        String key = CART_KEY_PREFIX + cart.getCartId();
        redisTemplate.opsForValue().set(key, cart, CART_TTL_DAYS, TimeUnit.DAYS);
    }

    public void deleteCart(String cartId) {
        String key = CART_KEY_PREFIX + cartId;
        redisTemplate.delete(key);
    }

    public void refreshTTL(String cartId) {
        String key = CART_KEY_PREFIX + cartId;
        redisTemplate.expire(key, CART_TTL_DAYS, TimeUnit.DAYS);
    }
}
```

---

## 10. Cart Cleanup Scheduler

```java
package com.ecommerce.scheduler;

import com.ecommerce.repository.CartRepository;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.scheduling.annotation.Scheduled;
import org.springframework.stereotype.Component;
import org.springframework.transaction.annotation.Transactional;

import java.time.LocalDateTime;

@Component
@RequiredArgsConstructor
@Slf4j
public class CartCleanupScheduler {

    private final CartRepository cartRepository;

    /**
     * Cleanup anonymous carts older than 30 days
     * Runs daily at 3 AM
     */
    @Scheduled(cron = "0 0 3 * * ?")
    @Transactional
    public void cleanupOldAnonymousCarts() {
        LocalDateTime threshold = LocalDateTime.now().minusDays(30);

        int deletedCount = cartRepository.deleteOldAnonymousCarts(threshold);

        log.info("Cleaned up {} old anonymous carts", deletedCount);
    }
}
```

---

## 11. Bài tập thực hành

### Bài tập 1: Basic Cart
- [x] Create Cart và CartItem entities
- [x] Implement add/update/remove operations
- [x] Handle anonymous vs authenticated carts

### Bài tập 2: Cart Warnings
- [x] Detect price changes
- [x] Check stock availability
- [x] Show low stock warnings

### Bài tập 3: Cart Merge
- [x] Merge anonymous cart on login
- [x] Handle duplicate products
- [x] Clean up merged cart

---

## 12. Checklist

- [ ] Create Cart và CartItem entities
- [ ] Create Flyway migration
- [ ] Implement CartService với CRUD operations
- [ ] Handle inventory checking
- [ ] Create cart warnings system
- [ ] Implement anonymous cart với session cookie
- [ ] Implement cart merge on login
- [ ] Add cart cleanup scheduler
- [ ] (Optional) Redis cart for high performance
- [ ] Test cart operations

---

## Điều hướng

- [← Day 20: OAuth2 & Refresh Token](../week-04/day-20-oauth2-refresh.md)
- [Day 22: Order Processing →](./day-22-order-processing.md)
- [Về trang chính](../00-overview.md)
