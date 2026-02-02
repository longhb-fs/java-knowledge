# Day 23: Payment Integration (Stripe)

## Mục tiêu học tập
- Hiểu Payment Gateway concepts
- Integrate Stripe Payment
- Handle Payment webhooks
- Implement refund functionality

---

## 1. Payment Flow Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                    STRIPE PAYMENT FLOW                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌────────────┐    ┌────────────┐    ┌────────────────────┐    │
│  │  Frontend  │    │  Backend   │    │      Stripe        │    │
│  └─────┬──────┘    └─────┬──────┘    └─────────┬──────────┘    │
│        │                 │                     │                │
│        │ 1. Checkout     │                     │                │
│        │────────────────▶│                     │                │
│        │                 │                     │                │
│        │                 │ 2. Create           │                │
│        │                 │    PaymentIntent    │                │
│        │                 │────────────────────▶│                │
│        │                 │                     │                │
│        │                 │ 3. client_secret    │                │
│        │                 │◀────────────────────│                │
│        │                 │                     │                │
│        │ 4. client_secret│                     │                │
│        │◀────────────────│                     │                │
│        │                 │                     │                │
│        │ 5. Confirm Payment (Stripe.js)        │                │
│        │──────────────────────────────────────▶│                │
│        │                 │                     │                │
│        │ 6. 3D Secure (if required)            │                │
│        │◀─────────────────────────────────────▶│                │
│        │                 │                     │                │
│        │                 │ 7. Webhook:         │                │
│        │                 │    payment_intent   │                │
│        │                 │    .succeeded       │                │
│        │                 │◀────────────────────│                │
│        │                 │                     │                │
│        │                 │ 8. Update Order     │                │
│        │                 │    Status           │                │
│        │                 │                     │                │
│        │ 9. Success page │                     │                │
│        │◀────────────────│                     │                │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## 2. Stripe Dependencies

### 2.1 pom.xml

```xml
<dependencies>
    <!-- Stripe Java SDK -->
    <dependency>
        <groupId>com.stripe</groupId>
        <artifactId>stripe-java</artifactId>
        <version>24.16.0</version>
    </dependency>
</dependencies>
```

### 2.2 application.yml

```yaml
stripe:
  api-key: ${STRIPE_SECRET_KEY}
  publishable-key: ${STRIPE_PUBLISHABLE_KEY}
  webhook-secret: ${STRIPE_WEBHOOK_SECRET}
  currency: usd

app:
  payment:
    success-url: ${APP_BASE_URL}/payment/success
    cancel-url: ${APP_BASE_URL}/payment/cancel
```

---

## 3. Stripe Configuration

```java
package com.ecommerce.config;

import com.stripe.Stripe;
import jakarta.annotation.PostConstruct;
import lombok.Data;
import org.springframework.boot.context.properties.ConfigurationProperties;
import org.springframework.context.annotation.Configuration;

@Configuration
@ConfigurationProperties(prefix = "stripe")
@Data
public class StripeConfig {

    private String apiKey;
    private String publishableKey;
    private String webhookSecret;
    private String currency = "usd";

    @PostConstruct
    public void init() {
        Stripe.apiKey = apiKey;
    }
}
```

---

## 4. Payment Entities

### 4.1 Payment Entity

```java
package com.ecommerce.domain.entity;

import com.ecommerce.domain.enums.PaymentMethod;
import com.ecommerce.domain.enums.PaymentStatus;
import jakarta.persistence.*;
import lombok.*;

import java.math.BigDecimal;
import java.time.LocalDateTime;

@Entity
@Table(name = "payments")
@Getter
@Setter
@NoArgsConstructor
@AllArgsConstructor
@Builder
public class Payment extends BaseEntity {

    @OneToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "order_id", nullable = false)
    private Order order;

    @Column(name = "stripe_payment_intent_id", unique = true)
    private String stripePaymentIntentId;

    @Column(name = "stripe_charge_id")
    private String stripeChargeId;

    @Column(name = "stripe_customer_id")
    private String stripeCustomerId;

    @Enumerated(EnumType.STRING)
    @Column(nullable = false)
    private PaymentStatus status = PaymentStatus.PENDING;

    @Enumerated(EnumType.STRING)
    @Column(name = "payment_method")
    private PaymentMethod paymentMethod;

    @Column(nullable = false, precision = 10, scale = 2)
    private BigDecimal amount;

    @Column(length = 10)
    private String currency = "USD";

    @Column(name = "card_last4", length = 4)
    private String cardLast4;

    @Column(name = "card_brand", length = 20)
    private String cardBrand;

    @Column(name = "paid_at")
    private LocalDateTime paidAt;

    @Column(name = "failed_at")
    private LocalDateTime failedAt;

    @Column(name = "failure_code")
    private String failureCode;

    @Column(name = "failure_message")
    private String failureMessage;

    @Column(name = "refunded_at")
    private LocalDateTime refundedAt;

    @Column(name = "refund_amount", precision = 10, scale = 2)
    private BigDecimal refundAmount;

    @Column(name = "refund_reason")
    private String refundReason;

    @Column(columnDefinition = "JSON")
    private String metadata;

    // ========== Status Methods ==========

    public void markSuccessful(String chargeId, String cardLast4, String cardBrand) {
        this.status = PaymentStatus.COMPLETED;
        this.stripeChargeId = chargeId;
        this.cardLast4 = cardLast4;
        this.cardBrand = cardBrand;
        this.paidAt = LocalDateTime.now();
    }

    public void markFailed(String failureCode, String failureMessage) {
        this.status = PaymentStatus.FAILED;
        this.failureCode = failureCode;
        this.failureMessage = failureMessage;
        this.failedAt = LocalDateTime.now();
    }

    public void markRefunded(BigDecimal amount, String reason) {
        this.status = PaymentStatus.REFUNDED;
        this.refundAmount = amount;
        this.refundReason = reason;
        this.refundedAt = LocalDateTime.now();
    }

    public boolean canRefund() {
        return status == PaymentStatus.COMPLETED && refundedAt == null;
    }
}
```

### 4.2 Flyway Migration

```sql
-- V12__Create_payments_table.sql

CREATE TABLE payments (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    order_id BIGINT NOT NULL UNIQUE,
    stripe_payment_intent_id VARCHAR(100) UNIQUE,
    stripe_charge_id VARCHAR(100),
    stripe_customer_id VARCHAR(100),
    status VARCHAR(30) NOT NULL,
    payment_method VARCHAR(30),
    amount DECIMAL(10, 2) NOT NULL,
    currency VARCHAR(10) DEFAULT 'USD',
    card_last4 VARCHAR(4),
    card_brand VARCHAR(20),
    paid_at DATETIME,
    failed_at DATETIME,
    failure_code VARCHAR(100),
    failure_message TEXT,
    refunded_at DATETIME,
    refund_amount DECIMAL(10, 2),
    refund_reason VARCHAR(255),
    metadata JSON,
    created_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at DATETIME,
    deleted BOOLEAN DEFAULT FALSE,

    CONSTRAINT fk_payment_order
        FOREIGN KEY (order_id) REFERENCES orders(id)
        ON DELETE CASCADE,

    INDEX idx_payment_intent (stripe_payment_intent_id),
    INDEX idx_payment_status (status)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
```

---

## 5. Payment DTOs

### 5.1 Request DTOs

```java
package com.ecommerce.dto.request;

import jakarta.validation.constraints.NotNull;
import lombok.Data;

@Data
public class CreatePaymentRequest {

    @NotNull(message = "Order ID is required")
    private Long orderId;
}
```

```java
package com.ecommerce.dto.request;

import jakarta.validation.constraints.NotBlank;
import jakarta.validation.constraints.NotNull;
import jakarta.validation.constraints.Positive;
import lombok.Data;

import java.math.BigDecimal;

@Data
public class RefundRequest {

    @NotNull(message = "Payment ID is required")
    private Long paymentId;

    @Positive(message = "Amount must be positive")
    private BigDecimal amount;  // null = full refund

    @NotBlank(message = "Reason is required")
    private String reason;
}
```

### 5.2 Response DTOs

```java
package com.ecommerce.dto.response;

import com.ecommerce.domain.enums.PaymentStatus;
import lombok.AllArgsConstructor;
import lombok.Builder;
import lombok.Data;
import lombok.NoArgsConstructor;

import java.math.BigDecimal;
import java.time.LocalDateTime;

@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class PaymentIntentDto {

    private Long paymentId;
    private String clientSecret;
    private String paymentIntentId;
    private BigDecimal amount;
    private String currency;
    private PaymentStatus status;
    private String publishableKey;
}
```

```java
package com.ecommerce.dto.response;

import com.ecommerce.domain.enums.PaymentStatus;
import lombok.AllArgsConstructor;
import lombok.Builder;
import lombok.Data;
import lombok.NoArgsConstructor;

import java.math.BigDecimal;
import java.time.LocalDateTime;

@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class PaymentDto {

    private Long id;
    private Long orderId;
    private String orderNumber;
    private PaymentStatus status;
    private BigDecimal amount;
    private String currency;
    private String cardLast4;
    private String cardBrand;
    private LocalDateTime paidAt;
    private LocalDateTime refundedAt;
    private BigDecimal refundAmount;
}
```

---

## 6. Stripe Service

### 6.1 StripeService Interface

```java
package com.ecommerce.service;

import com.ecommerce.dto.request.RefundRequest;
import com.ecommerce.dto.response.PaymentDto;
import com.ecommerce.dto.response.PaymentIntentDto;

public interface StripeService {

    PaymentIntentDto createPaymentIntent(Long orderId);

    PaymentDto confirmPayment(String paymentIntentId);

    PaymentDto refundPayment(RefundRequest request);

    void handleWebhook(String payload, String signature);

    String getOrCreateCustomer(Long userId, String email);
}
```

### 6.2 StripeServiceImpl

```java
package com.ecommerce.service.impl;

import com.ecommerce.config.StripeConfig;
import com.ecommerce.domain.entity.Order;
import com.ecommerce.domain.entity.Payment;
import com.ecommerce.domain.entity.User;
import com.ecommerce.domain.enums.PaymentStatus;
import com.ecommerce.dto.request.RefundRequest;
import com.ecommerce.dto.response.PaymentDto;
import com.ecommerce.dto.response.PaymentIntentDto;
import com.ecommerce.exception.BadRequestException;
import com.ecommerce.exception.PaymentException;
import com.ecommerce.exception.ResourceNotFoundException;
import com.ecommerce.mapper.PaymentMapper;
import com.ecommerce.repository.OrderRepository;
import com.ecommerce.repository.PaymentRepository;
import com.ecommerce.repository.UserRepository;
import com.ecommerce.service.OrderService;
import com.ecommerce.service.StripeService;
import com.stripe.exception.SignatureVerificationException;
import com.stripe.exception.StripeException;
import com.stripe.model.*;
import com.stripe.net.Webhook;
import com.stripe.param.*;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.math.BigDecimal;
import java.util.HashMap;
import java.util.Map;

@Service
@RequiredArgsConstructor
@Slf4j
public class StripeServiceImpl implements StripeService {

    private final StripeConfig stripeConfig;
    private final PaymentRepository paymentRepository;
    private final OrderRepository orderRepository;
    private final UserRepository userRepository;
    private final OrderService orderService;
    private final PaymentMapper paymentMapper;

    @Override
    @Transactional
    public PaymentIntentDto createPaymentIntent(Long orderId) {
        Order order = orderRepository.findById(orderId)
                .orElseThrow(() -> new ResourceNotFoundException("Order", "id", orderId));

        // Check if payment already exists
        if (paymentRepository.existsByOrderId(orderId)) {
            Payment existing = paymentRepository.findByOrderId(orderId).get();
            if (existing.getStatus() == PaymentStatus.COMPLETED) {
                throw new BadRequestException("Order already paid");
            }
            // Return existing payment intent
            return buildPaymentIntentDto(existing, existing.getStripePaymentIntentId());
        }

        try {
            // Get or create Stripe customer
            String customerId = getOrCreateCustomer(
                    order.getUser().getId(),
                    order.getUser().getEmail()
            );

            // Convert amount to cents
            long amountInCents = order.getTotalAmount()
                    .multiply(new BigDecimal("100"))
                    .longValue();

            // Create PaymentIntent
            PaymentIntentCreateParams params = PaymentIntentCreateParams.builder()
                    .setAmount(amountInCents)
                    .setCurrency(stripeConfig.getCurrency())
                    .setCustomer(customerId)
                    .setAutomaticPaymentMethods(
                            PaymentIntentCreateParams.AutomaticPaymentMethods.builder()
                                    .setEnabled(true)
                                    .build()
                    )
                    .putMetadata("order_id", orderId.toString())
                    .putMetadata("order_number", order.getOrderNumber())
                    .setDescription("Order " + order.getOrderNumber())
                    .build();

            PaymentIntent paymentIntent = PaymentIntent.create(params);

            // Save payment record
            Payment payment = Payment.builder()
                    .order(order)
                    .stripePaymentIntentId(paymentIntent.getId())
                    .stripeCustomerId(customerId)
                    .amount(order.getTotalAmount())
                    .currency(stripeConfig.getCurrency().toUpperCase())
                    .status(PaymentStatus.PENDING)
                    .build();

            payment = paymentRepository.save(payment);

            log.info("Created PaymentIntent {} for order {}",
                    paymentIntent.getId(), order.getOrderNumber());

            return buildPaymentIntentDto(payment, paymentIntent.getClientSecret());

        } catch (StripeException e) {
            log.error("Stripe error creating payment intent: {}", e.getMessage());
            throw new PaymentException("Failed to create payment: " + e.getMessage());
        }
    }

    @Override
    @Transactional
    public PaymentDto confirmPayment(String paymentIntentId) {
        Payment payment = paymentRepository.findByStripePaymentIntentId(paymentIntentId)
                .orElseThrow(() -> new ResourceNotFoundException(
                        "Payment", "paymentIntentId", paymentIntentId));

        try {
            PaymentIntent paymentIntent = PaymentIntent.retrieve(paymentIntentId);

            if ("succeeded".equals(paymentIntent.getStatus())) {
                // Get charge details
                Charge charge = null;
                if (paymentIntent.getLatestCharge() != null) {
                    charge = Charge.retrieve(paymentIntent.getLatestCharge());
                }

                String cardLast4 = null;
                String cardBrand = null;

                if (charge != null && charge.getPaymentMethodDetails() != null
                        && charge.getPaymentMethodDetails().getCard() != null) {
                    cardLast4 = charge.getPaymentMethodDetails().getCard().getLast4();
                    cardBrand = charge.getPaymentMethodDetails().getCard().getBrand();
                }

                payment.markSuccessful(
                        paymentIntent.getLatestCharge(),
                        cardLast4,
                        cardBrand
                );
                paymentRepository.save(payment);

                // Confirm order
                orderService.confirmOrder(payment.getOrder().getId());

                log.info("Payment {} confirmed for order {}",
                        paymentIntentId, payment.getOrder().getOrderNumber());
            }

            return paymentMapper.toDto(payment);

        } catch (StripeException e) {
            log.error("Error confirming payment: {}", e.getMessage());
            throw new PaymentException("Failed to confirm payment");
        }
    }

    @Override
    @Transactional
    public PaymentDto refundPayment(RefundRequest request) {
        Payment payment = paymentRepository.findById(request.getPaymentId())
                .orElseThrow(() -> new ResourceNotFoundException(
                        "Payment", "id", request.getPaymentId()));

        if (!payment.canRefund()) {
            throw new BadRequestException("Payment cannot be refunded");
        }

        try {
            BigDecimal refundAmount = request.getAmount() != null
                    ? request.getAmount()
                    : payment.getAmount();

            long amountInCents = refundAmount.multiply(new BigDecimal("100")).longValue();

            RefundCreateParams params = RefundCreateParams.builder()
                    .setPaymentIntent(payment.getStripePaymentIntentId())
                    .setAmount(amountInCents)
                    .setReason(RefundCreateParams.Reason.REQUESTED_BY_CUSTOMER)
                    .putMetadata("reason", request.getReason())
                    .build();

            Refund refund = Refund.create(params);

            if ("succeeded".equals(refund.getStatus())) {
                payment.markRefunded(refundAmount, request.getReason());
                paymentRepository.save(payment);

                // Update order status
                Order order = payment.getOrder();
                order.initiateReturn();
                orderRepository.save(order);

                log.info("Refund {} processed for payment {}",
                        refund.getId(), payment.getId());
            }

            return paymentMapper.toDto(payment);

        } catch (StripeException e) {
            log.error("Refund failed: {}", e.getMessage());
            throw new PaymentException("Failed to process refund: " + e.getMessage());
        }
    }

    @Override
    @Transactional
    public void handleWebhook(String payload, String signature) {
        Event event;

        try {
            event = Webhook.constructEvent(
                    payload,
                    signature,
                    stripeConfig.getWebhookSecret()
            );
        } catch (SignatureVerificationException e) {
            log.error("Invalid webhook signature");
            throw new BadRequestException("Invalid webhook signature");
        }

        log.info("Received Stripe webhook: {}", event.getType());

        switch (event.getType()) {
            case "payment_intent.succeeded" -> handlePaymentSucceeded(event);
            case "payment_intent.payment_failed" -> handlePaymentFailed(event);
            case "charge.refunded" -> handleChargeRefunded(event);
            default -> log.debug("Unhandled event type: {}", event.getType());
        }
    }

    @Override
    public String getOrCreateCustomer(Long userId, String email) {
        // Check if user already has Stripe customer ID
        User user = userRepository.findById(userId)
                .orElseThrow(() -> new ResourceNotFoundException("User", "id", userId));

        if (user.getStripeCustomerId() != null) {
            return user.getStripeCustomerId();
        }

        try {
            // Search for existing customer by email
            CustomerSearchParams searchParams = CustomerSearchParams.builder()
                    .setQuery("email:'" + email + "'")
                    .build();

            CustomerSearchResult result = Customer.search(searchParams);

            if (!result.getData().isEmpty()) {
                String customerId = result.getData().get(0).getId();
                user.setStripeCustomerId(customerId);
                userRepository.save(user);
                return customerId;
            }

            // Create new customer
            CustomerCreateParams params = CustomerCreateParams.builder()
                    .setEmail(email)
                    .setName(user.getFullName())
                    .putMetadata("user_id", userId.toString())
                    .build();

            Customer customer = Customer.create(params);

            user.setStripeCustomerId(customer.getId());
            userRepository.save(user);

            log.info("Created Stripe customer {} for user {}", customer.getId(), email);

            return customer.getId();

        } catch (StripeException e) {
            log.error("Error creating Stripe customer: {}", e.getMessage());
            throw new PaymentException("Failed to create customer");
        }
    }

    // ========== Webhook Handlers ==========

    private void handlePaymentSucceeded(Event event) {
        PaymentIntent paymentIntent = (PaymentIntent) event.getDataObjectDeserializer()
                .getObject().orElse(null);

        if (paymentIntent == null) return;

        paymentRepository.findByStripePaymentIntentId(paymentIntent.getId())
                .ifPresent(payment -> {
                    if (payment.getStatus() != PaymentStatus.COMPLETED) {
                        confirmPayment(paymentIntent.getId());
                    }
                });
    }

    private void handlePaymentFailed(Event event) {
        PaymentIntent paymentIntent = (PaymentIntent) event.getDataObjectDeserializer()
                .getObject().orElse(null);

        if (paymentIntent == null) return;

        paymentRepository.findByStripePaymentIntentId(paymentIntent.getId())
                .ifPresent(payment -> {
                    String failureCode = paymentIntent.getLastPaymentError() != null
                            ? paymentIntent.getLastPaymentError().getCode()
                            : "unknown";
                    String failureMessage = paymentIntent.getLastPaymentError() != null
                            ? paymentIntent.getLastPaymentError().getMessage()
                            : "Payment failed";

                    payment.markFailed(failureCode, failureMessage);
                    paymentRepository.save(payment);

                    log.warn("Payment failed for intent {}: {}",
                            paymentIntent.getId(), failureMessage);
                });
    }

    private void handleChargeRefunded(Event event) {
        Charge charge = (Charge) event.getDataObjectDeserializer()
                .getObject().orElse(null);

        if (charge == null) return;

        // Handle if refund was initiated from Stripe dashboard
        paymentRepository.findByStripeChargeId(charge.getId())
                .ifPresent(payment -> {
                    if (charge.getRefunded() && payment.getRefundedAt() == null) {
                        BigDecimal refundAmount = new BigDecimal(charge.getAmountRefunded())
                                .divide(new BigDecimal("100"));
                        payment.markRefunded(refundAmount, "Refunded via Stripe");
                        paymentRepository.save(payment);
                    }
                });
    }

    // ========== Helper Methods ==========

    private PaymentIntentDto buildPaymentIntentDto(Payment payment, String clientSecret) {
        return PaymentIntentDto.builder()
                .paymentId(payment.getId())
                .paymentIntentId(payment.getStripePaymentIntentId())
                .clientSecret(clientSecret)
                .amount(payment.getAmount())
                .currency(payment.getCurrency())
                .status(payment.getStatus())
                .publishableKey(stripeConfig.getPublishableKey())
                .build();
    }
}
```

---

## 7. Payment Controller

```java
package com.ecommerce.controller;

import com.ecommerce.dto.request.CreatePaymentRequest;
import com.ecommerce.dto.request.RefundRequest;
import com.ecommerce.dto.response.ApiResponse;
import com.ecommerce.dto.response.PaymentDto;
import com.ecommerce.dto.response.PaymentIntentDto;
import com.ecommerce.security.annotation.AdminOnly;
import com.ecommerce.service.StripeService;
import io.swagger.v3.oas.annotations.Operation;
import io.swagger.v3.oas.annotations.security.SecurityRequirement;
import io.swagger.v3.oas.annotations.tags.Tag;
import jakarta.validation.Valid;
import lombok.RequiredArgsConstructor;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

@RestController
@RequestMapping("/api/v1/payments")
@RequiredArgsConstructor
@Tag(name = "Payments", description = "Payment management APIs")
public class PaymentController {

    private final StripeService stripeService;

    @PostMapping("/create-intent")
    @SecurityRequirement(name = "bearerAuth")
    @Operation(summary = "Create payment intent for order")
    public ResponseEntity<ApiResponse<PaymentIntentDto>> createPaymentIntent(
            @Valid @RequestBody CreatePaymentRequest request) {

        PaymentIntentDto result = stripeService.createPaymentIntent(request.getOrderId());
        return ResponseEntity.ok(ApiResponse.success(result));
    }

    @PostMapping("/confirm/{paymentIntentId}")
    @SecurityRequirement(name = "bearerAuth")
    @Operation(summary = "Confirm payment (called after client-side confirmation)")
    public ResponseEntity<ApiResponse<PaymentDto>> confirmPayment(
            @PathVariable String paymentIntentId) {

        PaymentDto payment = stripeService.confirmPayment(paymentIntentId);
        return ResponseEntity.ok(ApiResponse.success(payment, "Payment confirmed"));
    }

    @PostMapping("/refund")
    @AdminOnly
    @SecurityRequirement(name = "bearerAuth")
    @Operation(summary = "Refund payment (Admin only)")
    public ResponseEntity<ApiResponse<PaymentDto>> refundPayment(
            @Valid @RequestBody RefundRequest request) {

        PaymentDto payment = stripeService.refundPayment(request);
        return ResponseEntity.ok(ApiResponse.success(payment, "Refund processed"));
    }

    @PostMapping("/webhook")
    @Operation(summary = "Stripe webhook endpoint")
    public ResponseEntity<String> handleWebhook(
            @RequestBody String payload,
            @RequestHeader("Stripe-Signature") String signature) {

        stripeService.handleWebhook(payload, signature);
        return ResponseEntity.ok("Webhook received");
    }
}
```

---

## 8. Frontend Integration

### 8.1 React/JavaScript Example

```javascript
// Install: npm install @stripe/stripe-js @stripe/react-stripe-js

import { loadStripe } from '@stripe/stripe-js';
import { Elements, CardElement, useStripe, useElements } from '@stripe/react-stripe-js';

// Initialize Stripe
const stripePromise = loadStripe('pk_test_...');

// Payment Form Component
function CheckoutForm({ orderId }) {
  const stripe = useStripe();
  const elements = useElements();
  const [loading, setLoading] = useState(false);
  const [error, setError] = useState(null);

  const handleSubmit = async (e) => {
    e.preventDefault();
    setLoading(true);
    setError(null);

    try {
      // 1. Create PaymentIntent on backend
      const response = await fetch('/api/v1/payments/create-intent', {
        method: 'POST',
        headers: {
          'Content-Type': 'application/json',
          'Authorization': `Bearer ${accessToken}`,
        },
        body: JSON.stringify({ orderId }),
      });

      const { data } = await response.json();
      const { clientSecret } = data;

      // 2. Confirm payment with Stripe.js
      const { error: stripeError, paymentIntent } = await stripe.confirmCardPayment(
        clientSecret,
        {
          payment_method: {
            card: elements.getElement(CardElement),
          },
        }
      );

      if (stripeError) {
        setError(stripeError.message);
        return;
      }

      if (paymentIntent.status === 'succeeded') {
        // 3. Confirm on backend
        await fetch(`/api/v1/payments/confirm/${paymentIntent.id}`, {
          method: 'POST',
          headers: {
            'Authorization': `Bearer ${accessToken}`,
          },
        });

        // Redirect to success page
        window.location.href = '/payment/success';
      }

    } catch (err) {
      setError('Payment failed. Please try again.');
    } finally {
      setLoading(false);
    }
  };

  return (
    <form onSubmit={handleSubmit}>
      <CardElement options={cardStyle} />

      {error && <div className="error">{error}</div>}

      <button type="submit" disabled={!stripe || loading}>
        {loading ? 'Processing...' : 'Pay Now'}
      </button>
    </form>
  );
}

// Wrapper Component
function PaymentPage({ orderId }) {
  return (
    <Elements stripe={stripePromise}>
      <CheckoutForm orderId={orderId} />
    </Elements>
  );
}
```

### 8.2 Card Element Styling

```javascript
const cardStyle = {
  style: {
    base: {
      color: '#32325d',
      fontFamily: '"Helvetica Neue", Helvetica, sans-serif',
      fontSmoothing: 'antialiased',
      fontSize: '16px',
      '::placeholder': {
        color: '#aab7c4',
      },
    },
    invalid: {
      color: '#fa755a',
      iconColor: '#fa755a',
    },
  },
};
```

---

## 9. Stripe Checkout Session (Alternative)

### 9.1 Hosted Checkout

```java
@Override
@Transactional
public String createCheckoutSession(Long orderId) {
    Order order = orderRepository.findById(orderId)
            .orElseThrow(() -> new ResourceNotFoundException("Order", "id", orderId));

    try {
        List<SessionCreateParams.LineItem> lineItems = order.getItems().stream()
                .map(item -> SessionCreateParams.LineItem.builder()
                        .setPriceData(
                                SessionCreateParams.LineItem.PriceData.builder()
                                        .setCurrency(stripeConfig.getCurrency())
                                        .setUnitAmount(
                                                item.getUnitPrice()
                                                        .multiply(new BigDecimal("100"))
                                                        .longValue()
                                        )
                                        .setProductData(
                                                SessionCreateParams.LineItem.PriceData.ProductData.builder()
                                                        .setName(item.getProductName())
                                                        .build()
                                        )
                                        .build()
                        )
                        .setQuantity((long) item.getQuantity())
                        .build()
                )
                .toList();

        SessionCreateParams params = SessionCreateParams.builder()
                .setMode(SessionCreateParams.Mode.PAYMENT)
                .setCustomerEmail(order.getUser().getEmail())
                .setSuccessUrl(appProperties.getPayment().getSuccessUrl() + "?session_id={CHECKOUT_SESSION_ID}")
                .setCancelUrl(appProperties.getPayment().getCancelUrl())
                .addAllLineItem(lineItems)
                .putMetadata("order_id", orderId.toString())
                .build();

        Session session = Session.create(params);

        log.info("Created Checkout Session {} for order {}",
                session.getId(), order.getOrderNumber());

        return session.getUrl();

    } catch (StripeException e) {
        log.error("Error creating checkout session: {}", e.getMessage());
        throw new PaymentException("Failed to create checkout session");
    }
}
```

---

## 10. Testing với Stripe

### 10.1 Test Cards

```
┌─────────────────────────────────────────────────────────────────┐
│                    STRIPE TEST CARDS                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Success:                                                        │
│  4242 4242 4242 4242 - Visa (no authentication required)        │
│  5555 5555 5555 4444 - Mastercard                               │
│                                                                  │
│  3D Secure:                                                      │
│  4000 0025 0000 3155 - Requires 3D Secure authentication        │
│                                                                  │
│  Failures:                                                       │
│  4000 0000 0000 0002 - Card declined                            │
│  4000 0000 0000 9995 - Insufficient funds                       │
│  4000 0000 0000 9987 - Lost card                                │
│                                                                  │
│  Use any future expiry date and any 3-digit CVC                 │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 10.2 Testing Webhooks Locally

```bash
# Install Stripe CLI
# https://stripe.com/docs/stripe-cli

# Login
stripe login

# Forward webhooks to local server
stripe listen --forward-to localhost:8080/api/v1/payments/webhook

# Trigger test events
stripe trigger payment_intent.succeeded
stripe trigger payment_intent.payment_failed
```

---

## 11. Bài tập thực hành

### Bài tập 1: Stripe Integration
- [x] Setup Stripe account và API keys
- [x] Implement PaymentIntent creation
- [x] Handle payment confirmation

### Bài tập 2: Webhooks
- [x] Implement webhook handler
- [x] Handle payment_intent.succeeded
- [x] Handle payment_intent.payment_failed

### Bài tập 3: Refunds
- [x] Implement refund API
- [x] Handle partial refunds
- [x] Update order status

---

## 12. Checklist

- [ ] Add Stripe dependency
- [ ] Configure Stripe API keys
- [ ] Create Payment entity
- [ ] Implement StripeService
- [ ] Create PaymentController
- [ ] Handle Stripe webhooks
- [ ] Implement refund functionality
- [ ] Frontend integration
- [ ] Test with Stripe test cards
- [ ] Test webhooks với Stripe CLI

---

## Điều hướng

- [← Day 22: Order Processing](./day-22-order-processing.md)
- [Day 24: Product Reviews & Ratings →](./day-24-reviews-ratings.md)
- [Về trang chính](../00-overview.md)
