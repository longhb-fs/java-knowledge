# Day 22: Order Processing

## Mục tiêu học tập
- Thiết kế Order lifecycle và state machine
- Implement checkout flow
- Handle inventory reservation
- Implement order status transitions

---

## 1. Order State Machine

```
┌─────────────────────────────────────────────────────────────────┐
│                    ORDER STATE MACHINE                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│                    ┌─────────────┐                              │
│                    │   PENDING   │                              │
│                    │  (Created)  │                              │
│                    └──────┬──────┘                              │
│                           │                                      │
│            ┌──────────────┼──────────────┐                      │
│            │              │              │                      │
│            ▼              ▼              ▼                      │
│    ┌───────────┐  ┌───────────┐  ┌───────────┐                 │
│    │ CANCELLED │  │ CONFIRMED │  │  EXPIRED  │                 │
│    │(By User)  │  │(Payment OK)│ │(Timeout)  │                 │
│    └───────────┘  └─────┬─────┘  └───────────┘                 │
│                         │                                        │
│                         ▼                                        │
│                  ┌───────────┐                                  │
│                  │PROCESSING │                                  │
│                  │(Preparing)│                                  │
│                  └─────┬─────┘                                  │
│                        │                                         │
│                        ▼                                         │
│                  ┌───────────┐                                  │
│                  │  SHIPPED  │                                  │
│                  │(In Transit│                                  │
│                  └─────┬─────┘                                  │
│                        │                                         │
│            ┌───────────┼───────────┐                            │
│            │           │           │                            │
│            ▼           ▼           ▼                            │
│    ┌───────────┐┌───────────┐┌───────────┐                     │
│    │ DELIVERED ││ RETURNED  ││  FAILED   │                     │
│    │(Completed)││(Refund)   ││(Delivery) │                     │
│    └───────────┘└───────────┘└───────────┘                     │
│                                                                  │
│  Legend:                                                        │
│  ───────── Valid transition                                     │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## 2. Order Entities

### 2.1 Order Entity

```java
package com.ecommerce.domain.entity;

import com.ecommerce.domain.enums.OrderStatus;
import com.ecommerce.domain.enums.PaymentMethod;
import com.ecommerce.domain.enums.PaymentStatus;
import com.ecommerce.exception.InvalidOrderStateException;
import jakarta.persistence.*;
import lombok.*;

import java.math.BigDecimal;
import java.time.LocalDateTime;
import java.util.ArrayList;
import java.util.List;
import java.util.Set;

@Entity
@Table(name = "orders")
@Getter
@Setter
@NoArgsConstructor
@AllArgsConstructor
@Builder
public class Order extends BaseEntity {

    @Column(name = "order_number", nullable = false, unique = true, length = 20)
    private String orderNumber;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "user_id", nullable = false)
    private User user;

    @Enumerated(EnumType.STRING)
    @Column(nullable = false)
    private OrderStatus status = OrderStatus.PENDING;

    @OneToMany(mappedBy = "order", cascade = CascadeType.ALL, orphanRemoval = true)
    @Builder.Default
    private List<OrderItem> items = new ArrayList<>();

    // ========== Pricing ==========

    @Column(nullable = false, precision = 10, scale = 2)
    private BigDecimal subtotal;

    @Column(name = "shipping_fee", precision = 10, scale = 2)
    private BigDecimal shippingFee = BigDecimal.ZERO;

    @Column(precision = 10, scale = 2)
    private BigDecimal tax = BigDecimal.ZERO;

    @Column(name = "discount_amount", precision = 10, scale = 2)
    private BigDecimal discountAmount = BigDecimal.ZERO;

    @Column(name = "total_amount", nullable = false, precision = 10, scale = 2)
    private BigDecimal totalAmount;

    @Column(length = 10)
    private String currency = "USD";

    // ========== Payment ==========

    @Enumerated(EnumType.STRING)
    @Column(name = "payment_method")
    private PaymentMethod paymentMethod;

    @Enumerated(EnumType.STRING)
    @Column(name = "payment_status")
    private PaymentStatus paymentStatus = PaymentStatus.PENDING;

    @Column(name = "payment_id")
    private String paymentId;

    @Column(name = "paid_at")
    private LocalDateTime paidAt;

    // ========== Shipping ==========

    @Embedded
    private ShippingAddress shippingAddress;

    @Column(name = "shipping_method", length = 50)
    private String shippingMethod;

    @Column(name = "tracking_number", length = 100)
    private String trackingNumber;

    @Column(name = "shipped_at")
    private LocalDateTime shippedAt;

    @Column(name = "delivered_at")
    private LocalDateTime deliveredAt;

    @Column(name = "estimated_delivery")
    private LocalDateTime estimatedDelivery;

    // ========== Timestamps ==========

    @Column(name = "confirmed_at")
    private LocalDateTime confirmedAt;

    @Column(name = "cancelled_at")
    private LocalDateTime cancelledAt;

    @Column(name = "cancellation_reason")
    private String cancellationReason;

    @Column(name = "expires_at")
    private LocalDateTime expiresAt;

    @Column(columnDefinition = "TEXT")
    private String notes;

    // ========== State Transitions ==========

    private static final Set<OrderStatus> CANCELLABLE_STATUSES = Set.of(
            OrderStatus.PENDING, OrderStatus.CONFIRMED
    );

    public boolean canCancel() {
        return CANCELLABLE_STATUSES.contains(status);
    }

    public void confirm() {
        validateTransition(OrderStatus.CONFIRMED);
        this.status = OrderStatus.CONFIRMED;
        this.confirmedAt = LocalDateTime.now();
    }

    public void markProcessing() {
        validateTransition(OrderStatus.PROCESSING);
        this.status = OrderStatus.PROCESSING;
    }

    public void ship(String trackingNumber, LocalDateTime estimatedDelivery) {
        validateTransition(OrderStatus.SHIPPED);
        this.status = OrderStatus.SHIPPED;
        this.trackingNumber = trackingNumber;
        this.estimatedDelivery = estimatedDelivery;
        this.shippedAt = LocalDateTime.now();
    }

    public void deliver() {
        validateTransition(OrderStatus.DELIVERED);
        this.status = OrderStatus.DELIVERED;
        this.deliveredAt = LocalDateTime.now();
    }

    public void cancel(String reason) {
        if (!canCancel()) {
            throw new InvalidOrderStateException(
                    "Order cannot be cancelled in status: " + status);
        }
        this.status = OrderStatus.CANCELLED;
        this.cancelledAt = LocalDateTime.now();
        this.cancellationReason = reason;
    }

    public void expire() {
        validateTransition(OrderStatus.EXPIRED);
        this.status = OrderStatus.EXPIRED;
    }

    public void markFailed(String reason) {
        this.status = OrderStatus.FAILED;
        this.notes = reason;
    }

    public void initiateReturn() {
        validateTransition(OrderStatus.RETURNED);
        this.status = OrderStatus.RETURNED;
    }

    private void validateTransition(OrderStatus newStatus) {
        if (!isValidTransition(newStatus)) {
            throw new InvalidOrderStateException(
                    String.format("Invalid transition from %s to %s", status, newStatus));
        }
    }

    private boolean isValidTransition(OrderStatus newStatus) {
        return switch (status) {
            case PENDING -> Set.of(OrderStatus.CONFIRMED, OrderStatus.CANCELLED,
                    OrderStatus.EXPIRED).contains(newStatus);
            case CONFIRMED -> Set.of(OrderStatus.PROCESSING, OrderStatus.CANCELLED)
                    .contains(newStatus);
            case PROCESSING -> newStatus == OrderStatus.SHIPPED;
            case SHIPPED -> Set.of(OrderStatus.DELIVERED, OrderStatus.RETURNED,
                    OrderStatus.FAILED).contains(newStatus);
            case DELIVERED -> newStatus == OrderStatus.RETURNED;
            default -> false;
        };
    }

    // ========== Calculations ==========

    public void calculateTotals() {
        this.subtotal = items.stream()
                .map(OrderItem::getSubtotal)
                .reduce(BigDecimal.ZERO, BigDecimal::add);

        this.totalAmount = subtotal
                .add(shippingFee != null ? shippingFee : BigDecimal.ZERO)
                .add(tax != null ? tax : BigDecimal.ZERO)
                .subtract(discountAmount != null ? discountAmount : BigDecimal.ZERO);
    }

    public void addItem(OrderItem item) {
        items.add(item);
        item.setOrder(this);
    }

    // ========== Order Number Generation ==========

    public static String generateOrderNumber() {
        // Format: ORD-YYYYMMDD-XXXXX (random 5 digits)
        String date = java.time.LocalDate.now()
                .format(java.time.format.DateTimeFormatter.BASIC_ISO_DATE);
        String random = String.format("%05d", (int) (Math.random() * 100000));
        return "ORD-" + date + "-" + random;
    }
}
```

### 2.2 OrderItem Entity

```java
package com.ecommerce.domain.entity;

import jakarta.persistence.*;
import lombok.*;

import java.math.BigDecimal;

@Entity
@Table(name = "order_items")
@Getter
@Setter
@NoArgsConstructor
@AllArgsConstructor
@Builder
public class OrderItem extends BaseEntity {

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "order_id", nullable = false)
    private Order order;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "product_id", nullable = false)
    private Product product;

    @Column(name = "product_name", nullable = false)
    private String productName;  // Snapshot at order time

    @Column(name = "product_sku", length = 50)
    private String productSku;

    @Column(name = "product_image")
    private String productImage;

    @Column(nullable = false)
    private int quantity;

    @Column(name = "unit_price", nullable = false, precision = 10, scale = 2)
    private BigDecimal unitPrice;  // Price at order time

    @Column(precision = 10, scale = 2)
    private BigDecimal discount = BigDecimal.ZERO;

    @Transient
    public BigDecimal getSubtotal() {
        return unitPrice.multiply(BigDecimal.valueOf(quantity))
                .subtract(discount != null ? discount : BigDecimal.ZERO);
    }

    // Factory method
    public static OrderItem fromCartItem(com.ecommerce.domain.entity.CartItem cartItem) {
        Product product = cartItem.getProduct();
        return OrderItem.builder()
                .product(product)
                .productName(product.getName())
                .productSku(product.getSku())
                .productImage(product.getThumbnailUrl())
                .quantity(cartItem.getQuantity())
                .unitPrice(product.getPrice())
                .build();
    }
}
```

### 2.3 ShippingAddress Embeddable

```java
package com.ecommerce.domain.entity;

import jakarta.persistence.Column;
import jakarta.persistence.Embeddable;
import lombok.*;

@Embeddable
@Getter
@Setter
@NoArgsConstructor
@AllArgsConstructor
@Builder
public class ShippingAddress {

    @Column(name = "recipient_name", length = 100)
    private String recipientName;

    @Column(name = "phone", length = 20)
    private String phone;

    @Column(name = "address_line1")
    private String addressLine1;

    @Column(name = "address_line2")
    private String addressLine2;

    @Column(name = "city", length = 100)
    private String city;

    @Column(name = "state", length = 100)
    private String state;

    @Column(name = "postal_code", length = 20)
    private String postalCode;

    @Column(name = "country", length = 100)
    private String country;

    public String getFullAddress() {
        StringBuilder sb = new StringBuilder();
        sb.append(addressLine1);
        if (addressLine2 != null && !addressLine2.isBlank()) {
            sb.append(", ").append(addressLine2);
        }
        sb.append(", ").append(city);
        if (state != null && !state.isBlank()) {
            sb.append(", ").append(state);
        }
        sb.append(" ").append(postalCode);
        sb.append(", ").append(country);
        return sb.toString();
    }
}
```

### 2.4 Enums

```java
package com.ecommerce.domain.enums;

public enum OrderStatus {
    PENDING,      // Chờ thanh toán
    CONFIRMED,    // Đã xác nhận (thanh toán thành công)
    PROCESSING,   // Đang xử lý
    SHIPPED,      // Đã giao cho vận chuyển
    DELIVERED,    // Đã giao hàng
    CANCELLED,    // Đã hủy
    RETURNED,     // Đã hoàn trả
    EXPIRED,      // Hết hạn thanh toán
    FAILED        // Giao hàng thất bại
}
```

```java
package com.ecommerce.domain.enums;

public enum PaymentStatus {
    PENDING,
    PROCESSING,
    COMPLETED,
    FAILED,
    REFUNDED,
    CANCELLED
}
```

```java
package com.ecommerce.domain.enums;

public enum PaymentMethod {
    CREDIT_CARD,
    DEBIT_CARD,
    PAYPAL,
    BANK_TRANSFER,
    COD,  // Cash on Delivery
    WALLET
}
```

---

## 3. Checkout DTOs

### 3.1 Request DTOs

```java
package com.ecommerce.dto.request;

import com.ecommerce.domain.enums.PaymentMethod;
import jakarta.validation.Valid;
import jakarta.validation.constraints.NotBlank;
import jakarta.validation.constraints.NotNull;
import lombok.Data;

@Data
public class CheckoutRequest {

    @Valid
    @NotNull(message = "Shipping address is required")
    private ShippingAddressDto shippingAddress;

    @NotNull(message = "Payment method is required")
    private PaymentMethod paymentMethod;

    private String shippingMethod = "STANDARD";

    private String couponCode;

    private String notes;

    @Data
    public static class ShippingAddressDto {
        @NotBlank(message = "Recipient name is required")
        private String recipientName;

        @NotBlank(message = "Phone is required")
        private String phone;

        @NotBlank(message = "Address is required")
        private String addressLine1;

        private String addressLine2;

        @NotBlank(message = "City is required")
        private String city;

        private String state;

        @NotBlank(message = "Postal code is required")
        private String postalCode;

        @NotBlank(message = "Country is required")
        private String country;
    }
}
```

### 3.2 Response DTOs

```java
package com.ecommerce.dto.response;

import com.ecommerce.domain.enums.OrderStatus;
import com.ecommerce.domain.enums.PaymentMethod;
import com.ecommerce.domain.enums.PaymentStatus;
import lombok.AllArgsConstructor;
import lombok.Builder;
import lombok.Data;
import lombok.NoArgsConstructor;

import java.math.BigDecimal;
import java.time.LocalDateTime;
import java.util.List;

@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class OrderDto {

    private Long id;
    private String orderNumber;
    private OrderStatus status;
    private List<OrderItemDto> items;

    // Pricing
    private BigDecimal subtotal;
    private BigDecimal shippingFee;
    private BigDecimal tax;
    private BigDecimal discountAmount;
    private BigDecimal totalAmount;
    private String currency;

    // Payment
    private PaymentMethod paymentMethod;
    private PaymentStatus paymentStatus;
    private LocalDateTime paidAt;

    // Shipping
    private ShippingAddressDto shippingAddress;
    private String shippingMethod;
    private String trackingNumber;
    private LocalDateTime estimatedDelivery;

    // Timestamps
    private LocalDateTime createdAt;
    private LocalDateTime confirmedAt;
    private LocalDateTime shippedAt;
    private LocalDateTime deliveredAt;
    private LocalDateTime cancelledAt;
    private String cancellationReason;

    private String notes;

    @Data
    @Builder
    @NoArgsConstructor
    @AllArgsConstructor
    public static class OrderItemDto {
        private Long id;
        private Long productId;
        private String productName;
        private String productSku;
        private String productImage;
        private int quantity;
        private BigDecimal unitPrice;
        private BigDecimal subtotal;
    }

    @Data
    @Builder
    @NoArgsConstructor
    @AllArgsConstructor
    public static class ShippingAddressDto {
        private String recipientName;
        private String phone;
        private String addressLine1;
        private String addressLine2;
        private String city;
        private String state;
        private String postalCode;
        private String country;
        private String fullAddress;
    }
}
```

---

## 4. Order Service

### 4.1 OrderService Interface

```java
package com.ecommerce.service;

import com.ecommerce.dto.request.CheckoutRequest;
import com.ecommerce.dto.response.OrderDto;
import org.springframework.data.domain.Page;
import org.springframework.data.domain.Pageable;

public interface OrderService {

    OrderDto checkout(CheckoutRequest request);

    OrderDto getOrderById(Long id);

    OrderDto getOrderByNumber(String orderNumber);

    Page<OrderDto> getMyOrders(Pageable pageable);

    Page<OrderDto> getAllOrders(Pageable pageable);

    OrderDto confirmOrder(Long orderId);

    OrderDto cancelOrder(Long orderId, String reason);

    OrderDto updateStatus(Long orderId, String status);

    OrderDto shipOrder(Long orderId, String trackingNumber);

    OrderDto deliverOrder(Long orderId);
}
```

### 4.2 OrderServiceImpl

```java
package com.ecommerce.service.impl;

import com.ecommerce.domain.entity.*;
import com.ecommerce.domain.enums.OrderStatus;
import com.ecommerce.domain.enums.PaymentStatus;
import com.ecommerce.dto.request.CheckoutRequest;
import com.ecommerce.dto.response.OrderDto;
import com.ecommerce.event.OrderCreatedEvent;
import com.ecommerce.event.OrderStatusChangedEvent;
import com.ecommerce.exception.BadRequestException;
import com.ecommerce.exception.InsufficientStockException;
import com.ecommerce.exception.ResourceNotFoundException;
import com.ecommerce.mapper.OrderMapper;
import com.ecommerce.repository.CartRepository;
import com.ecommerce.repository.OrderRepository;
import com.ecommerce.repository.ProductRepository;
import com.ecommerce.service.InventoryService;
import com.ecommerce.service.OrderService;
import com.ecommerce.service.ShippingService;
import com.ecommerce.util.SecurityUtils;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.context.ApplicationEventPublisher;
import org.springframework.data.domain.Page;
import org.springframework.data.domain.Pageable;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.math.BigDecimal;
import java.time.LocalDateTime;
import java.util.ArrayList;
import java.util.List;

@Service
@RequiredArgsConstructor
@Slf4j
public class OrderServiceImpl implements OrderService {

    private final OrderRepository orderRepository;
    private final CartRepository cartRepository;
    private final ProductRepository productRepository;
    private final InventoryService inventoryService;
    private final ShippingService shippingService;
    private final OrderMapper orderMapper;
    private final ApplicationEventPublisher eventPublisher;

    private static final int ORDER_EXPIRATION_MINUTES = 30;

    @Override
    @Transactional
    public OrderDto checkout(CheckoutRequest request) {
        User currentUser = SecurityUtils.getCurrentUser()
                .orElseThrow(() -> new BadRequestException("User not authenticated"));

        // 1. Get cart
        Cart cart = cartRepository.findByUserIdWithItems(currentUser.getId())
                .orElseThrow(() -> new BadRequestException("Cart is empty"));

        if (cart.isEmpty()) {
            throw new BadRequestException("Cart is empty");
        }

        // 2. Validate stock và reserve inventory
        List<OrderItem> orderItems = new ArrayList<>();
        for (CartItem cartItem : cart.getItems()) {
            Product product = cartItem.getProduct();

            // Check availability
            if (!product.isActive()) {
                throw new BadRequestException(product.getName() + " is no longer available");
            }

            // Reserve inventory (throws exception if insufficient)
            inventoryService.reserveStock(product.getId(), cartItem.getQuantity());

            // Create order item
            OrderItem orderItem = OrderItem.fromCartItem(cartItem);
            orderItems.add(orderItem);
        }

        // 3. Calculate shipping
        BigDecimal shippingFee = shippingService.calculateShippingFee(
                request.getShippingMethod(),
                request.getShippingAddress()
        );

        // 4. Create order
        Order order = Order.builder()
                .orderNumber(Order.generateOrderNumber())
                .user(currentUser)
                .status(OrderStatus.PENDING)
                .paymentMethod(request.getPaymentMethod())
                .paymentStatus(PaymentStatus.PENDING)
                .shippingAddress(mapShippingAddress(request.getShippingAddress()))
                .shippingMethod(request.getShippingMethod())
                .shippingFee(shippingFee)
                .tax(calculateTax(cart.getSubtotal()))
                .discountAmount(BigDecimal.ZERO)
                .notes(request.getNotes())
                .expiresAt(LocalDateTime.now().plusMinutes(ORDER_EXPIRATION_MINUTES))
                .items(new ArrayList<>())
                .build();

        // Add items
        for (OrderItem item : orderItems) {
            order.addItem(item);
        }

        // Calculate totals
        order.calculateTotals();

        // 5. Save order
        order = orderRepository.save(order);

        // 6. Clear cart
        cart.clear();
        cartRepository.save(cart);

        log.info("Order {} created for user {}", order.getOrderNumber(), currentUser.getEmail());

        // 7. Publish event
        eventPublisher.publishEvent(new OrderCreatedEvent(this, order));

        return orderMapper.toDto(order);
    }

    @Override
    @Transactional(readOnly = true)
    public OrderDto getOrderById(Long id) {
        Order order = findOrderById(id);
        validateOrderAccess(order);
        return orderMapper.toDto(order);
    }

    @Override
    @Transactional(readOnly = true)
    public OrderDto getOrderByNumber(String orderNumber) {
        Order order = orderRepository.findByOrderNumber(orderNumber)
                .orElseThrow(() -> new ResourceNotFoundException("Order", "orderNumber", orderNumber));
        validateOrderAccess(order);
        return orderMapper.toDto(order);
    }

    @Override
    @Transactional(readOnly = true)
    public Page<OrderDto> getMyOrders(Pageable pageable) {
        Long userId = SecurityUtils.getCurrentUserId()
                .orElseThrow(() -> new BadRequestException("User not authenticated"));

        return orderRepository.findByUserIdOrderByCreatedAtDesc(userId, pageable)
                .map(orderMapper::toDto);
    }

    @Override
    @Transactional(readOnly = true)
    public Page<OrderDto> getAllOrders(Pageable pageable) {
        return orderRepository.findAllByOrderByCreatedAtDesc(pageable)
                .map(orderMapper::toDto);
    }

    @Override
    @Transactional
    public OrderDto confirmOrder(Long orderId) {
        Order order = findOrderById(orderId);

        // Validate payment (in real app, integrate with payment gateway)
        // For now, just confirm
        order.confirm();
        order.setPaymentStatus(PaymentStatus.COMPLETED);
        order.setPaidAt(LocalDateTime.now());

        orderRepository.save(order);

        log.info("Order {} confirmed", order.getOrderNumber());

        publishStatusChange(order, OrderStatus.PENDING, OrderStatus.CONFIRMED);

        return orderMapper.toDto(order);
    }

    @Override
    @Transactional
    public OrderDto cancelOrder(Long orderId, String reason) {
        Order order = findOrderById(orderId);

        OrderStatus previousStatus = order.getStatus();

        order.cancel(reason);

        // Release reserved inventory
        for (OrderItem item : order.getItems()) {
            inventoryService.releaseStock(item.getProduct().getId(), item.getQuantity());
        }

        orderRepository.save(order);

        log.info("Order {} cancelled: {}", order.getOrderNumber(), reason);

        publishStatusChange(order, previousStatus, OrderStatus.CANCELLED);

        return orderMapper.toDto(order);
    }

    @Override
    @Transactional
    public OrderDto updateStatus(Long orderId, String status) {
        Order order = findOrderById(orderId);
        OrderStatus previousStatus = order.getStatus();
        OrderStatus newStatus = OrderStatus.valueOf(status.toUpperCase());

        switch (newStatus) {
            case CONFIRMED -> order.confirm();
            case PROCESSING -> order.markProcessing();
            case DELIVERED -> order.deliver();
            case RETURNED -> order.initiateReturn();
            default -> throw new BadRequestException("Invalid status transition");
        }

        orderRepository.save(order);

        publishStatusChange(order, previousStatus, newStatus);

        return orderMapper.toDto(order);
    }

    @Override
    @Transactional
    public OrderDto shipOrder(Long orderId, String trackingNumber) {
        Order order = findOrderById(orderId);
        OrderStatus previousStatus = order.getStatus();

        // Calculate estimated delivery (3-5 business days)
        LocalDateTime estimatedDelivery = LocalDateTime.now().plusDays(5);

        order.ship(trackingNumber, estimatedDelivery);

        // Deduct actual inventory
        for (OrderItem item : order.getItems()) {
            inventoryService.confirmDeduction(item.getProduct().getId(), item.getQuantity());
        }

        orderRepository.save(order);

        log.info("Order {} shipped with tracking: {}", order.getOrderNumber(), trackingNumber);

        publishStatusChange(order, previousStatus, OrderStatus.SHIPPED);

        return orderMapper.toDto(order);
    }

    @Override
    @Transactional
    public OrderDto deliverOrder(Long orderId) {
        Order order = findOrderById(orderId);
        OrderStatus previousStatus = order.getStatus();

        order.deliver();
        orderRepository.save(order);

        log.info("Order {} delivered", order.getOrderNumber());

        publishStatusChange(order, previousStatus, OrderStatus.DELIVERED);

        return orderMapper.toDto(order);
    }

    // ========== Private Methods ==========

    private Order findOrderById(Long id) {
        return orderRepository.findByIdWithItems(id)
                .orElseThrow(() -> new ResourceNotFoundException("Order", "id", id));
    }

    private void validateOrderAccess(Order order) {
        if (SecurityUtils.isAdmin()) {
            return;
        }

        Long currentUserId = SecurityUtils.getCurrentUserId().orElse(null);
        if (!order.getUser().getId().equals(currentUserId)) {
            throw new BadRequestException("You don't have access to this order");
        }
    }

    private ShippingAddress mapShippingAddress(CheckoutRequest.ShippingAddressDto dto) {
        return ShippingAddress.builder()
                .recipientName(dto.getRecipientName())
                .phone(dto.getPhone())
                .addressLine1(dto.getAddressLine1())
                .addressLine2(dto.getAddressLine2())
                .city(dto.getCity())
                .state(dto.getState())
                .postalCode(dto.getPostalCode())
                .country(dto.getCountry())
                .build();
    }

    private BigDecimal calculateTax(BigDecimal subtotal) {
        // Simple 10% tax
        return subtotal.multiply(new BigDecimal("0.10"));
    }

    private void publishStatusChange(Order order, OrderStatus from, OrderStatus to) {
        eventPublisher.publishEvent(new OrderStatusChangedEvent(this, order, from, to));
    }
}
```

---

## 5. Inventory Service

```java
package com.ecommerce.service;

public interface InventoryService {

    void reserveStock(Long productId, int quantity);

    void releaseStock(Long productId, int quantity);

    void confirmDeduction(Long productId, int quantity);

    int getAvailableStock(Long productId);
}
```

```java
package com.ecommerce.service.impl;

import com.ecommerce.domain.entity.Product;
import com.ecommerce.exception.InsufficientStockException;
import com.ecommerce.exception.ResourceNotFoundException;
import com.ecommerce.repository.ProductRepository;
import com.ecommerce.service.InventoryService;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.data.redis.core.RedisTemplate;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.util.concurrent.TimeUnit;

@Service
@RequiredArgsConstructor
@Slf4j
public class InventoryServiceImpl implements InventoryService {

    private final ProductRepository productRepository;
    private final RedisTemplate<String, Integer> redisTemplate;

    private static final String RESERVED_KEY_PREFIX = "inventory:reserved:";
    private static final int RESERVATION_TTL_MINUTES = 30;

    @Override
    @Transactional
    public void reserveStock(Long productId, int quantity) {
        Product product = getProduct(productId);

        int available = getAvailableStock(productId);

        if (available < quantity) {
            throw new InsufficientStockException(
                    String.format("Insufficient stock for %s. Available: %d, Requested: %d",
                            product.getName(), available, quantity));
        }

        // Add to reserved count in Redis
        String key = RESERVED_KEY_PREFIX + productId;
        redisTemplate.opsForValue().increment(key, quantity);
        redisTemplate.expire(key, RESERVATION_TTL_MINUTES, TimeUnit.MINUTES);

        log.debug("Reserved {} units of product {}", quantity, productId);
    }

    @Override
    @Transactional
    public void releaseStock(Long productId, int quantity) {
        String key = RESERVED_KEY_PREFIX + productId;
        Integer current = redisTemplate.opsForValue().get(key);

        if (current != null && current > 0) {
            int newValue = Math.max(0, current - quantity);
            if (newValue == 0) {
                redisTemplate.delete(key);
            } else {
                redisTemplate.opsForValue().set(key, newValue);
            }
        }

        log.debug("Released {} units of product {}", quantity, productId);
    }

    @Override
    @Transactional
    public void confirmDeduction(Long productId, int quantity) {
        Product product = getProduct(productId);

        // Deduct from actual stock
        int newStock = product.getStockQuantity() - quantity;
        if (newStock < 0) {
            throw new InsufficientStockException("Stock went negative for product: " + productId);
        }

        product.setStockQuantity(newStock);
        productRepository.save(product);

        // Release reservation
        releaseStock(productId, quantity);

        log.info("Deducted {} units from product {}. New stock: {}",
                quantity, productId, newStock);
    }

    @Override
    public int getAvailableStock(Long productId) {
        Product product = getProduct(productId);
        int actualStock = product.getStockQuantity();

        // Subtract reserved quantity
        String key = RESERVED_KEY_PREFIX + productId;
        Integer reserved = redisTemplate.opsForValue().get(key);
        int reservedQty = reserved != null ? reserved : 0;

        return Math.max(0, actualStock - reservedQty);
    }

    private Product getProduct(Long productId) {
        return productRepository.findById(productId)
                .orElseThrow(() -> new ResourceNotFoundException("Product", "id", productId));
    }
}
```

---

## 6. Order Events

```java
package com.ecommerce.event;

import com.ecommerce.domain.entity.Order;
import lombok.Getter;
import org.springframework.context.ApplicationEvent;

@Getter
public class OrderCreatedEvent extends ApplicationEvent {

    private final Order order;

    public OrderCreatedEvent(Object source, Order order) {
        super(source);
        this.order = order;
    }
}
```

```java
package com.ecommerce.event;

import com.ecommerce.domain.entity.Order;
import com.ecommerce.domain.enums.OrderStatus;
import lombok.Getter;
import org.springframework.context.ApplicationEvent;

@Getter
public class OrderStatusChangedEvent extends ApplicationEvent {

    private final Order order;
    private final OrderStatus previousStatus;
    private final OrderStatus newStatus;

    public OrderStatusChangedEvent(Object source, Order order,
                                   OrderStatus previousStatus,
                                   OrderStatus newStatus) {
        super(source);
        this.order = order;
        this.previousStatus = previousStatus;
        this.newStatus = newStatus;
    }
}
```

### 6.1 Event Listeners

```java
package com.ecommerce.event.listener;

import com.ecommerce.domain.entity.Order;
import com.ecommerce.domain.enums.OrderStatus;
import com.ecommerce.event.OrderCreatedEvent;
import com.ecommerce.event.OrderStatusChangedEvent;
import com.ecommerce.service.EmailService;
import com.ecommerce.service.NotificationService;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.context.event.EventListener;
import org.springframework.scheduling.annotation.Async;
import org.springframework.stereotype.Component;

import java.util.Map;

@Component
@RequiredArgsConstructor
@Slf4j
public class OrderEventListener {

    private final EmailService emailService;
    private final NotificationService notificationService;

    @EventListener
    @Async
    public void handleOrderCreated(OrderCreatedEvent event) {
        Order order = event.getOrder();
        log.info("Handling OrderCreatedEvent for order: {}", order.getOrderNumber());

        // Send confirmation email
        emailService.sendEmail(
                order.getUser().getEmail(),
                "Order Confirmation - " + order.getOrderNumber(),
                "email/order-confirmation",
                Map.of(
                        "order", order,
                        "userName", order.getUser().getFirstName()
                )
        );

        // Send notification
        notificationService.sendOrderNotification(
                order.getUser().getId(),
                "Order Placed",
                "Your order " + order.getOrderNumber() + " has been placed successfully."
        );
    }

    @EventListener
    @Async
    public void handleOrderStatusChanged(OrderStatusChangedEvent event) {
        Order order = event.getOrder();
        OrderStatus newStatus = event.getNewStatus();

        log.info("Order {} status changed from {} to {}",
                order.getOrderNumber(),
                event.getPreviousStatus(),
                newStatus);

        String subject = switch (newStatus) {
            case CONFIRMED -> "Order Confirmed";
            case PROCESSING -> "Order Being Processed";
            case SHIPPED -> "Order Shipped";
            case DELIVERED -> "Order Delivered";
            case CANCELLED -> "Order Cancelled";
            default -> "Order Update";
        };

        // Send status update email
        emailService.sendEmail(
                order.getUser().getEmail(),
                subject + " - " + order.getOrderNumber(),
                "email/order-status-update",
                Map.of(
                        "order", order,
                        "userName", order.getUser().getFirstName(),
                        "newStatus", newStatus.name()
                )
        );

        // Send push notification
        notificationService.sendOrderNotification(
                order.getUser().getId(),
                subject,
                getStatusMessage(order, newStatus)
        );
    }

    private String getStatusMessage(Order order, OrderStatus status) {
        return switch (status) {
            case CONFIRMED -> "Your order has been confirmed and is being prepared.";
            case SHIPPED -> "Your order is on its way! Tracking: " + order.getTrackingNumber();
            case DELIVERED -> "Your order has been delivered. Enjoy!";
            case CANCELLED -> "Your order has been cancelled.";
            default -> "Your order status has been updated.";
        };
    }
}
```

---

## 7. Order Controller

```java
package com.ecommerce.controller;

import com.ecommerce.dto.request.CheckoutRequest;
import com.ecommerce.dto.response.ApiResponse;
import com.ecommerce.dto.response.OrderDto;
import com.ecommerce.security.annotation.AdminOnly;
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
import org.springframework.web.bind.annotation.*;

@RestController
@RequestMapping("/api/v1/orders")
@RequiredArgsConstructor
@Tag(name = "Orders", description = "Order management APIs")
@SecurityRequirement(name = "bearerAuth")
public class OrderController {

    private final OrderService orderService;

    @PostMapping("/checkout")
    @Operation(summary = "Create order from cart")
    public ResponseEntity<ApiResponse<OrderDto>> checkout(
            @Valid @RequestBody CheckoutRequest request) {

        OrderDto order = orderService.checkout(request);
        return ResponseEntity
                .status(HttpStatus.CREATED)
                .body(ApiResponse.success(order, "Order created successfully"));
    }

    @GetMapping
    @Operation(summary = "Get my orders")
    public ResponseEntity<ApiResponse<Page<OrderDto>>> getMyOrders(Pageable pageable) {
        Page<OrderDto> orders = orderService.getMyOrders(pageable);
        return ResponseEntity.ok(ApiResponse.success(orders));
    }

    @GetMapping("/{id}")
    @Operation(summary = "Get order by ID")
    public ResponseEntity<ApiResponse<OrderDto>> getOrder(@PathVariable Long id) {
        OrderDto order = orderService.getOrderById(id);
        return ResponseEntity.ok(ApiResponse.success(order));
    }

    @GetMapping("/number/{orderNumber}")
    @Operation(summary = "Get order by order number")
    public ResponseEntity<ApiResponse<OrderDto>> getOrderByNumber(
            @PathVariable String orderNumber) {

        OrderDto order = orderService.getOrderByNumber(orderNumber);
        return ResponseEntity.ok(ApiResponse.success(order));
    }

    @PostMapping("/{id}/cancel")
    @Operation(summary = "Cancel order")
    public ResponseEntity<ApiResponse<OrderDto>> cancelOrder(
            @PathVariable Long id,
            @RequestParam(required = false) String reason) {

        OrderDto order = orderService.cancelOrder(id, reason);
        return ResponseEntity.ok(ApiResponse.success(order, "Order cancelled"));
    }

    // ========== Admin Endpoints ==========

    @GetMapping("/admin/all")
    @AdminOnly
    @Operation(summary = "Get all orders (Admin)")
    public ResponseEntity<ApiResponse<Page<OrderDto>>> getAllOrders(Pageable pageable) {
        Page<OrderDto> orders = orderService.getAllOrders(pageable);
        return ResponseEntity.ok(ApiResponse.success(orders));
    }

    @PostMapping("/{id}/confirm")
    @AdminOnly
    @Operation(summary = "Confirm order (Admin)")
    public ResponseEntity<ApiResponse<OrderDto>> confirmOrder(@PathVariable Long id) {
        OrderDto order = orderService.confirmOrder(id);
        return ResponseEntity.ok(ApiResponse.success(order, "Order confirmed"));
    }

    @PostMapping("/{id}/ship")
    @AdminOnly
    @Operation(summary = "Ship order (Admin)")
    public ResponseEntity<ApiResponse<OrderDto>> shipOrder(
            @PathVariable Long id,
            @RequestParam String trackingNumber) {

        OrderDto order = orderService.shipOrder(id, trackingNumber);
        return ResponseEntity.ok(ApiResponse.success(order, "Order shipped"));
    }

    @PostMapping("/{id}/deliver")
    @AdminOnly
    @Operation(summary = "Mark order as delivered (Admin)")
    public ResponseEntity<ApiResponse<OrderDto>> deliverOrder(@PathVariable Long id) {
        OrderDto order = orderService.deliverOrder(id);
        return ResponseEntity.ok(ApiResponse.success(order, "Order delivered"));
    }

    @PutMapping("/{id}/status")
    @AdminOnly
    @Operation(summary = "Update order status (Admin)")
    public ResponseEntity<ApiResponse<OrderDto>> updateStatus(
            @PathVariable Long id,
            @RequestParam String status) {

        OrderDto order = orderService.updateStatus(id, status);
        return ResponseEntity.ok(ApiResponse.success(order, "Status updated"));
    }
}
```

---

## 8. Order Expiration Scheduler

```java
package com.ecommerce.scheduler;

import com.ecommerce.domain.entity.Order;
import com.ecommerce.domain.enums.OrderStatus;
import com.ecommerce.repository.OrderRepository;
import com.ecommerce.service.InventoryService;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.scheduling.annotation.Scheduled;
import org.springframework.stereotype.Component;
import org.springframework.transaction.annotation.Transactional;

import java.time.LocalDateTime;
import java.util.List;

@Component
@RequiredArgsConstructor
@Slf4j
public class OrderExpirationScheduler {

    private final OrderRepository orderRepository;
    private final InventoryService inventoryService;

    /**
     * Expire unpaid orders every 5 minutes
     */
    @Scheduled(fixedRate = 300000)  // 5 minutes
    @Transactional
    public void expireUnpaidOrders() {
        LocalDateTime now = LocalDateTime.now();

        List<Order> expiredOrders = orderRepository
                .findByStatusAndExpiresAtBefore(OrderStatus.PENDING, now);

        for (Order order : expiredOrders) {
            order.expire();

            // Release reserved inventory
            order.getItems().forEach(item ->
                    inventoryService.releaseStock(item.getProduct().getId(), item.getQuantity())
            );

            orderRepository.save(order);

            log.info("Order {} expired", order.getOrderNumber());
        }

        if (!expiredOrders.isEmpty()) {
            log.info("Expired {} unpaid orders", expiredOrders.size());
        }
    }
}
```

---

## 9. Bài tập thực hành

### Bài tập 1: Order Entity
- [x] Create Order và OrderItem entities
- [x] Implement state machine transitions
- [x] Add validation cho state changes

### Bài tập 2: Checkout Flow
- [x] Validate cart items
- [x] Reserve inventory
- [x] Create order from cart

### Bài tập 3: Order Events
- [x] Publish events on status change
- [x] Send email notifications
- [x] Handle order expiration

---

## 10. Checklist

- [ ] Create Order, OrderItem entities
- [ ] Create ShippingAddress embeddable
- [ ] Create Flyway migrations
- [ ] Implement OrderService với checkout
- [ ] Implement InventoryService
- [ ] Create order state transitions
- [ ] Add order events và listeners
- [ ] Create OrderController
- [ ] Add order expiration scheduler
- [ ] Test checkout flow

---

## Điều hướng

- [← Day 21: Shopping Cart](./day-21-shopping-cart.md)
- [Day 23: Payment Integration →](./day-23-payment-integration.md)
- [Về trang chính](../00-overview.md)
