# Day 6: Cart + Order + Payment

> **Mục tiêu:** Xây dựng Shopping Cart, Order Processing, và Payment Integration (Stripe + COD) — chạy end-to-end.
> **Thời gian:** ~6-8 giờ | **Output:** Checkout flow: Add to Cart -> Place Order -> Pay -> Order Confirmed.

---

## 1. Shopping Cart

### 1.1 Tổng Quan

```
LOGGED-IN user  → CartService → MySQL (tồn tại vĩnh viễn)
ANONYMOUS user  → AnonymousCartService → Redis (TTL 7 ngày)
```

Phần này tập trung vào **DB Cart** cho logged-in user trước, Redis Cart ở cuối section.

### 1.2 CartItem Entity (Reference từ Day 3)

```java
@Entity
@Table(name = "cart_items", uniqueConstraints = {
    @UniqueConstraint(columnNames = {"user_id", "product_id"})
})
@Getter @Setter @NoArgsConstructor
public class CartItem extends BaseEntity {
    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "user_id", nullable = false)
    private User user;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "product_id", nullable = false)
    private Product product;

    @Column(nullable = false) @Min(1)
    private int quantity;
}
```

> **`uniqueConstraints`** đảm bảo mỗi user chỉ có 1 record/product. Add cùng product → **update quantity** thay vì tạo mới.

### 1.3 Cart DTOs

```java
// === REQUEST DTOs ===
@Data
public class AddToCartRequest {
    @NotNull(message = "Product ID is required")
    private Long productId;
    @Min(value = 1, message = "Quantity must be at least 1")
    private int quantity = 1;
}

@Data
public class UpdateCartRequest {
    @Min(value = 1, message = "Quantity must be at least 1")
    private int quantity;
}

// === RESPONSE DTOs ===
@Data @Builder
public class CartItemResponse {
    private Long id;
    private Long productId;
    private String productName;
    private String productImage;
    private BigDecimal productPrice;
    private int quantity;
    private BigDecimal subtotal;    // = productPrice * quantity
    private boolean inStock;        // product.stockQuantity >= quantity
}

@Data @Builder
public class CartResponse {
    private List<CartItemResponse> items;
    private int totalItems;
    private BigDecimal totalAmount;
}
```

> **Tại sao `inStock`?** Giữa lúc add to cart và checkout, hàng có thể hết. Frontend hiển thị warning mà không cần gọi thêm API.

### 1.4 CartItemRepository

```java
public interface CartItemRepository extends JpaRepository<CartItem, Long> {
    @Query("SELECT ci FROM CartItem ci JOIN FETCH ci.product p " +
           "LEFT JOIN FETCH p.images WHERE ci.user.id = :userId")
    List<CartItem> findByUserIdWithProduct(Long userId);

    Optional<CartItem> findByUserIdAndProductId(Long userId, Long productId);
    void deleteAllByUserId(Long userId);
    void deleteByIdAndUserId(Long id, Long userId);
}
```

### 1.5 CartService

```java
@Service
@RequiredArgsConstructor
@Transactional(readOnly = true)
public class CartService {
    private final CartItemRepository cartItemRepository;
    private final ProductRepository productRepository;

    public CartResponse getCart(Long userId) {
        List<CartItem> items = cartItemRepository.findByUserIdWithProduct(userId);
        List<CartItemResponse> itemResponses = items.stream()
                .map(this::toCartItemResponse).toList();
        BigDecimal totalAmount = itemResponses.stream()
                .map(CartItemResponse::getSubtotal)
                .reduce(BigDecimal.ZERO, BigDecimal::add);
        return CartResponse.builder()
                .items(itemResponses).totalItems(itemResponses.size())
                .totalAmount(totalAmount).build();
    }

    @Transactional
    public CartItemResponse addToCart(Long userId, AddToCartRequest request) {
        // 1. Tìm product, throw 404 nếu không có
        Product product = productRepository.findById(request.getProductId())
                .orElseThrow(() -> new ResourceNotFoundException("Product not found"));
        // 2. Check stock
        if (product.getStockQuantity() < request.getQuantity())
            throw new BadRequestException("Insufficient stock");
        // 3. Đã có trong cart? → update quantity. Chưa có? → tạo mới
        CartItem cartItem = cartItemRepository
                .findByUserIdAndProductId(userId, request.getProductId())
                .map(existing -> {
                    int newQty = existing.getQuantity() + request.getQuantity();
                    if (product.getStockQuantity() < newQty)
                        throw new BadRequestException("Insufficient stock");
                    existing.setQuantity(newQty);
                    return existing;
                })
                .orElseGet(() -> {
                    CartItem newItem = new CartItem();
                    newItem.setProduct(product);
                    newItem.setQuantity(request.getQuantity());
                    return newItem;
                });
        cartItemRepository.save(cartItem);
        return toCartItemResponse(cartItem);
    }

    @Transactional
    public CartItemResponse updateQuantity(Long userId, Long cartItemId,
                                           UpdateCartRequest request) {
        CartItem cartItem = cartItemRepository.findById(cartItemId)
                .filter(item -> item.getUser().getId().equals(userId))
                .orElseThrow(() -> new ResourceNotFoundException("Cart item not found"));
        if (cartItem.getProduct().getStockQuantity() < request.getQuantity())
            throw new BadRequestException("Insufficient stock");
        cartItem.setQuantity(request.getQuantity());
        return toCartItemResponse(cartItemRepository.save(cartItem));
    }

    @Transactional
    public void removeItem(Long userId, Long cartItemId) {
        cartItemRepository.deleteByIdAndUserId(cartItemId, userId);
    }

    @Transactional
    public void clearCart(Long userId) {
        cartItemRepository.deleteAllByUserId(userId);
    }

    private CartItemResponse toCartItemResponse(CartItem item) {
        Product p = item.getProduct();
        String primaryImage = p.getImages().stream()
                .filter(ProductImage::getIsPrimary).findFirst()
                .map(ProductImage::getUrl).orElse(null);
        BigDecimal subtotal = p.getPrice().multiply(BigDecimal.valueOf(item.getQuantity()));
        return CartItemResponse.builder()
                .id(item.getId()).productId(p.getId())
                .productName(p.getName()).productImage(primaryImage)
                .productPrice(p.getPrice()).quantity(item.getQuantity())
                .subtotal(subtotal).inStock(p.getStockQuantity() >= item.getQuantity())
                .build();
    }
}
```

> **Lưu ý:** `@Transactional(readOnly = true)` ở class level tối ưu read. Method thay đổi data override bằng `@Transactional`. Luôn check **userId** để chống truy cập trái phép.

### 1.6 CartController

```java
@RestController
@RequestMapping("/api/v1/cart")
@RequiredArgsConstructor
@Tag(name = "Shopping Cart")
public class CartController {
    private final CartService cartService;

    @GetMapping
    @Operation(summary = "Get current user's cart")
    public ResponseEntity<ApiResponse<CartResponse>> getCart(@CurrentUser Long userId) {
        return ResponseEntity.ok(ApiResponse.success(cartService.getCart(userId)));
    }

    @PostMapping
    @Operation(summary = "Add product to cart")
    public ResponseEntity<ApiResponse<CartItemResponse>> addToCart(
            @CurrentUser Long userId, @Valid @RequestBody AddToCartRequest request) {
        return ResponseEntity.ok(ApiResponse.success(cartService.addToCart(userId, request)));
    }

    @PutMapping("/{itemId}")
    @Operation(summary = "Update cart item quantity")
    public ResponseEntity<ApiResponse<CartItemResponse>> updateQuantity(
            @CurrentUser Long userId, @PathVariable Long itemId,
            @Valid @RequestBody UpdateCartRequest request) {
        return ResponseEntity.ok(
            ApiResponse.success(cartService.updateQuantity(userId, itemId, request)));
    }

    @DeleteMapping("/{itemId}")
    @Operation(summary = "Remove item from cart")
    public ResponseEntity<Void> removeItem(@CurrentUser Long userId, @PathVariable Long itemId) {
        cartService.removeItem(userId, itemId);
        return ResponseEntity.noContent().build();
    }

    @DeleteMapping
    @Operation(summary = "Clear entire cart")
    public ResponseEntity<Void> clearCart(@CurrentUser Long userId) {
        cartService.clearCart(userId);
        return ResponseEntity.noContent().build();
    }
}
```

> **`@CurrentUser`** là custom annotation từ Day 5 để lấy userId từ JWT token.

### 1.7 Redis Cart cho Anonymous Users

```
Key:    cart:session:{sessionId}
Value:  JSON của List<AnonymousCartItem>
TTL:    7 ngày (604800 seconds)
```

```java
@Data @AllArgsConstructor @NoArgsConstructor
public class AnonymousCartItem {
    private Long productId;
    private int quantity;
}

@Service @RequiredArgsConstructor
public class AnonymousCartService {
    private final RedisTemplate<String, String> redisTemplate;
    private final ObjectMapper objectMapper;
    private static final String CART_PREFIX = "cart:session:";

    public void addToCart(String sessionId, Long productId, int quantity) {
        String key = CART_PREFIX + sessionId;
        // Get existing → upsert item → save with TTL
        redisTemplate.opsForValue().set(key, json, 7, TimeUnit.DAYS);
    }

    public List<AnonymousCartItem> getCart(String sessionId) {
        String json = redisTemplate.opsForValue().get(CART_PREFIX + sessionId);
        if (json == null) return List.of();
        return objectMapper.readValue(json, new TypeReference<>() {});
    }
}
```

**Cart Merge khi Login** — Gọi trong AuthService sau khi xác thực, trước khi trả JWT:

```java
@Transactional
public void mergeCart(String sessionId, Long userId) {
    List<AnonymousCartItem> redisItems = anonymousCartService.getCart(sessionId);
    for (AnonymousCartItem item : redisItems) {
        try {
            AddToCartRequest req = new AddToCartRequest();
            req.setProductId(item.getProductId());
            req.setQuantity(item.getQuantity());
            cartService.addToCart(userId, req);
        } catch (Exception e) {
            log.warn("Skip merge: productId={}", item.getProductId(), e);
        }
    }
    anonymousCartService.clearCart(sessionId);
}
```

---

## 2. Order Processing

### 2.1 Order State Machine

```
                        Order State Machine
    PENDING ──────▶ CONFIRMED ──────▶ PROCESSING ──────▶ SHIPPED ──────▶ DELIVERED
      │                │                  │                                  │
      ▼                ▼                  ▼                                  ▼
   CANCELLED       CANCELLED          CANCELLED                          REFUNDED
```

| Trạng thái hiện tại | Cho phép chuyển sang |
|---------------------|----------------------|
| `PENDING` | `CONFIRMED`, `CANCELLED` |
| `CONFIRMED` | `PROCESSING`, `CANCELLED` |
| `PROCESSING` | `SHIPPED`, `CANCELLED` |
| `SHIPPED` | `DELIVERED` |
| `DELIVERED` | `REFUNDED` |
| `CANCELLED` / `REFUNDED` | *(không chuyển tiếp)* |

```java
public enum OrderStatus {
    PENDING, CONFIRMED, PROCESSING, SHIPPED, DELIVERED, CANCELLED, REFUNDED;

    private static final Map<OrderStatus, Set<OrderStatus>> TRANSITIONS = Map.of(
        PENDING,     Set.of(CONFIRMED, CANCELLED),
        CONFIRMED,   Set.of(PROCESSING, CANCELLED),
        PROCESSING,  Set.of(SHIPPED, CANCELLED),
        SHIPPED,     Set.of(DELIVERED),
        DELIVERED,   Set.of(REFUNDED),
        CANCELLED,   Set.of(),
        REFUNDED,    Set.of()
    );

    public boolean canTransitionTo(OrderStatus target) {
        return TRANSITIONS.getOrDefault(this, Set.of()).contains(target);
    }
}
```

> **Map trong Enum** thay vì if-else: định nghĩa **tất cả luật** 1 chỗ. Thêm trạng thái mới? Thêm 1 dòng.

### 2.2 Order DTOs

```java
@Data
public class CreateOrderRequest {
    @NotNull private Long addressId;
    private String note;
    @NotNull private PaymentMethod paymentMethod;  // STRIPE, COD
}

@Data @Builder
public class OrderResponse {
    private Long id;
    private String orderNumber;
    private OrderStatus status;
    private BigDecimal totalAmount;
    private BigDecimal shippingFee;
    private String note;
    private String shippingAddress;      // Snapshot
    private List<OrderItemResponse> items;
    private PaymentResponse payment;
    private Instant createdAt;
}

@Data @Builder
public class OrderItemResponse {
    private Long id;
    private String productName;          // Snapshot
    private BigDecimal productPrice;     // Snapshot
    private int quantity;
    private BigDecimal subtotal;
}
```

### 2.3 OrderItem Snapshot Pattern

```
VẤN ĐỀ:
  User đặt Product A giá 100,000 VND. Hôm sau admin tăng giá 150,000.
  → Đơn hàng cũ hiện giá nào? Product B bị xóa → hiện tên thế nào?

GIẢI PHÁP: SNAPSHOT — OrderItem lưu TRỰC TIẾP info tại thời điểm đặt:
  ┌──────────────────────────────────────────────┐
  │ OrderItem                                    │
  │   product_name  = "Áo thun nam"  (snapshot)  │
  │   product_price = 100000         (snapshot)  │
  │   quantity      = 2                          │
  │   subtotal      = 200000                     │
  │   product_id    = 42  (reference, NULLABLE)  │
  └──────────────────────────────────────────────┘
  Dù Product sửa giá, đổi tên, hay xóa → đơn hàng vẫn ĐÚNG.
```

```java
@Entity @Table(name = "order_items")
@Getter @Setter @NoArgsConstructor
public class OrderItem extends BaseEntity {
    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "order_id", nullable = false)
    private Order order;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "product_id")       // NULLABLE — product có thể bị xóa
    private Product product;

    // === SNAPSHOT FIELDS ===
    @Column(name = "product_name", nullable = false)
    private String productName;
    @Column(name = "product_price", nullable = false, precision = 12, scale = 2)
    private BigDecimal productPrice;
    @Column(nullable = false)
    private int quantity;
    @Column(nullable = false, precision = 12, scale = 2)
    private BigDecimal subtotal;
}
```

### 2.4 OrderService

```java
@Service @RequiredArgsConstructor @Slf4j
public class OrderService {
    private final OrderRepository orderRepository;
    private final OrderItemRepository orderItemRepository;
    private final CartItemRepository cartItemRepository;
    private final ProductRepository productRepository;
    private final AddressRepository addressRepository;
    private final PaymentRepository paymentRepository;

    private String generateOrderNumber() {
        return "ORD-" + Instant.now().toEpochMilli() + "-"
               + String.format("%04d", new Random().nextInt(10000));
    }

    @Transactional
    public OrderResponse checkout(Long userId, CreateOrderRequest request) {
        // 1. Lấy cart items
        List<CartItem> cartItems = cartItemRepository.findByUserIdWithProduct(userId);
        if (cartItems.isEmpty())
            throw new BadRequestException("Cart is empty");

        // 2. Lấy shipping address
        Address address = addressRepository.findById(request.getAddressId())
                .filter(a -> a.getUser().getId().equals(userId))
                .orElseThrow(() -> new ResourceNotFoundException("Address not found"));

        // 3. Tạo Order
        Order order = new Order();
        order.setOrderNumber(generateOrderNumber());
        order.setStatus(OrderStatus.PENDING);
        order.setNote(request.getNote());
        order.setShippingAddress(address.getStreet() + ", " + address.getCity());
        order.setShippingFee(new BigDecimal("30000"));
        orderRepository.save(order);

        // 4. Tạo OrderItems + Reserve stock
        BigDecimal itemsTotal = BigDecimal.ZERO;
        for (CartItem cartItem : cartItems) {
            Product product = cartItem.getProduct();
            reserveStock(product.getId(), cartItem.getQuantity());

            BigDecimal subtotal = product.getPrice()
                    .multiply(BigDecimal.valueOf(cartItem.getQuantity()))
                    .setScale(2, RoundingMode.HALF_UP);

            OrderItem orderItem = new OrderItem();
            orderItem.setOrder(order);
            orderItem.setProduct(product);
            orderItem.setProductName(product.getName());       // snapshot
            orderItem.setProductPrice(product.getPrice());     // snapshot
            orderItem.setQuantity(cartItem.getQuantity());
            orderItem.setSubtotal(subtotal);
            orderItemRepository.save(orderItem);
            itemsTotal = itemsTotal.add(subtotal);
        }

        // 5. Tổng tiền = items + shipping
        BigDecimal totalAmount = itemsTotal.add(order.getShippingFee())
                .setScale(2, RoundingMode.HALF_UP);
        order.setTotalAmount(totalAmount);

        // 6. Tạo Payment
        Payment payment = new Payment();
        payment.setOrder(order);
        payment.setAmount(totalAmount);
        payment.setMethod(request.getPaymentMethod());
        payment.setStatus(PaymentStatus.PENDING);
        paymentRepository.save(payment);

        // 7. Xóa cart
        cartItemRepository.deleteAllByUserId(userId);
        log.info("Order created: {}, total: {}", order.getOrderNumber(), totalAmount);
        return toOrderResponse(order);
    }

    private void reserveStock(Long productId, int quantity) {
        Product product = productRepository.findByIdWithLock(productId)
                .orElseThrow(() -> new ResourceNotFoundException("Product not found"));
        if (product.getStockQuantity() < quantity)
            throw new BadRequestException("Insufficient stock for: " + product.getName());
        product.setStockQuantity(product.getStockQuantity() - quantity);
        productRepository.save(product);
    }

    @Transactional
    public OrderResponse updateStatus(Long orderId, OrderStatus newStatus) {
        Order order = orderRepository.findById(orderId)
                .orElseThrow(() -> new ResourceNotFoundException("Order not found"));
        if (!order.getStatus().canTransitionTo(newStatus))
            throw new BadRequestException(
                "Cannot transition from " + order.getStatus() + " to " + newStatus);
        if (newStatus == OrderStatus.CANCELLED) restoreStock(order);
        order.setStatus(newStatus);
        orderRepository.save(order);
        return toOrderResponse(order);
    }

    private void restoreStock(Order order) {
        orderItemRepository.findByOrderId(order.getId()).forEach(item -> {
            if (item.getProduct() != null) {
                Product p = item.getProduct();
                p.setStockQuantity(p.getStockQuantity() + item.getQuantity());
                productRepository.save(p);
            }
        });
    }

    public PagedResponse<OrderResponse> getUserOrders(Long userId, Pageable pageable) {
        Page<Order> page = orderRepository.findByUserIdOrderByCreatedAtDesc(userId, pageable);
        return new PagedResponse<>(page.map(this::toOrderResponse));
    }

    public OrderResponse getOrder(Long userId, Long orderId) {
        return orderRepository.findById(orderId)
                .filter(o -> o.getUser().getId().equals(userId))
                .map(this::toOrderResponse)
                .orElseThrow(() -> new ResourceNotFoundException("Order not found"));
    }
}
```

> **BigDecimal bắt buộc cho tiền:** `0.1 + 0.2 = 0.30000000000000004` (double) vs `0.3` (BigDecimal). Luôn dùng `setScale(2, RoundingMode.HALF_UP)`.

### 2.5 Inventory Management — Race Condition

**Vấn đề:** 2 user đặt cùng sản phẩm chỉ còn 1 cái:

```
Thời gian   User A                     User B
──────────────────────────────────────────────────
  T1        Read stock = 1
  T2                                   Read stock = 1
  T3        stock >= 1? YES            stock >= 1? YES
  T4        stock = 0                  stock = 0   ← SAI! Đã hết rồi
```

**Giải pháp: Pessimistic Locking**

```java
// Trong ProductRepository
@Lock(LockModeType.PESSIMISTIC_WRITE)
@Query("SELECT p FROM Product p WHERE p.id = :id")
Optional<Product> findByIdWithLock(@Param("id") Long id);
```

```
Với PESSIMISTIC_WRITE:
  T1  User A: LOCK row       User B: WAIT (bị block)
  T2  User A: stock=1→0, SAVE, RELEASE
  T3                          User B: LOCK → Read stock=0 → ERROR ✓
```

> **Pessimistic vs Optimistic:** Flash sale → Pessimistic (an toàn). CRUD thường → Optimistic (`@Version`, nhanh hơn nhưng phải handle retry).

---

## 3. Payment Integration — Stripe

### 3.1 Payment Flow

```
Client → Backend: Create PaymentIntent → Stripe: Return clientSecret → Client: Show Stripe form
                                                                            │
Client nhập card (TRỰC TIẾP với Stripe, không qua server ta)               │
                                                                            ▼
Stripe Process → Webhook Event (async) → Backend: Update Order → CONFIRMED
```

> Webhook là **event-driven**: Stripe báo ta khi xong, không cần poll.

### 3.2 Setup

```xml
<!-- pom.xml -->
<dependency>
    <groupId>com.stripe</groupId>
    <artifactId>stripe-java</artifactId>
    <version>26.0.0</version>
</dependency>
```

```yaml
# application.yml — KHÔNG BAO GIỜ commit secret key lên Git!
stripe:
  secret-key: ${STRIPE_SECRET_KEY}
  webhook-secret: ${STRIPE_WEBHOOK_SECRET}
```

```java
@Configuration
public class StripeConfig {
    @Value("${stripe.secret-key}")
    private String secretKey;

    @PostConstruct  // Chạy 1 lần duy nhất sau khi bean được tạo
    public void init() { Stripe.apiKey = secretKey; }
}
```

### 3.3 PaymentService

```java
@Service @RequiredArgsConstructor @Slf4j
public class PaymentService {
    private final OrderRepository orderRepository;
    private final PaymentRepository paymentRepository;
    @Value("${stripe.webhook-secret}") private String webhookSecret;

    public PaymentIntentResponse createPaymentIntent(Long orderId, Long userId)
            throws StripeException {
        Order order = orderRepository.findById(orderId)
                .filter(o -> o.getUser().getId().equals(userId))
                .orElseThrow(() -> new ResourceNotFoundException("Order not found"));
        if (order.getStatus() != OrderStatus.PENDING)
            throw new BadRequestException("Can only pay for PENDING orders");

        // Stripe amount = đơn vị nhỏ nhất. VND không có decimal, USD nhân 100
        long amount = order.getTotalAmount()
                .multiply(BigDecimal.valueOf(100)).longValue();

        PaymentIntentCreateParams params = PaymentIntentCreateParams.builder()
                .setAmount(amount).setCurrency("vnd")
                .putMetadata("order_id", orderId.toString())
                .putMetadata("user_id", userId.toString())
                .setAutomaticPaymentMethods(
                    PaymentIntentCreateParams.AutomaticPaymentMethods.builder()
                        .setEnabled(true).build())
                .build();

        PaymentIntent intent = PaymentIntent.create(params);

        // Lưu Stripe PaymentIntent ID
        Payment payment = order.getPayment();
        payment.setTransactionId(intent.getId());
        paymentRepository.save(payment);

        log.info("PaymentIntent created: {} for order: {}",
                intent.getId(), order.getOrderNumber());
        return new PaymentIntentResponse(intent.getClientSecret(), intent.getId());
    }

    @Transactional
    public void handleWebhook(String payload, String sigHeader) {
        // 1. Verify signature — chống giả mạo
        Event event;
        try {
            event = Webhook.constructEvent(payload, sigHeader, webhookSecret);
        } catch (SignatureVerificationException e) {
            throw new BadRequestException("Invalid webhook signature");
        }
        log.info("Webhook: type={}", event.getType());

        // 2. Handle event types
        switch (event.getType()) {
            case "payment_intent.succeeded" -> handlePaymentSuccess(event);
            case "payment_intent.payment_failed" -> handlePaymentFailure(event);
            default -> log.info("Unhandled event: {}", event.getType());
        }
    }

    private void handlePaymentSuccess(Event event) {
        PaymentIntent intent = (PaymentIntent) event.getDataObjectDeserializer()
                .getObject().orElseThrow();
        Payment payment = paymentRepository.findByTransactionId(intent.getId())
                .orElseThrow();

        // Idempotency: đã xử lý chưa?
        if (payment.getStatus() == PaymentStatus.COMPLETED) {
            log.info("Already processed: {}", intent.getId());
            return;
        }
        payment.setStatus(PaymentStatus.COMPLETED);
        paymentRepository.save(payment);

        Order order = payment.getOrder();
        order.setStatus(OrderStatus.CONFIRMED);
        orderRepository.save(order);
        log.info("Payment succeeded: {}", order.getOrderNumber());
    }

    private void handlePaymentFailure(Event event) {
        PaymentIntent intent = (PaymentIntent) event.getDataObjectDeserializer()
                .getObject().orElseThrow();
        Payment payment = paymentRepository.findByTransactionId(intent.getId())
                .orElseThrow();
        payment.setStatus(PaymentStatus.FAILED);
        paymentRepository.save(payment);
        // Hoàn lại stock vì đã reserve lúc checkout
        restoreStockForOrder(payment.getOrder());
        log.info("Payment failed: {}", payment.getOrder().getOrderNumber());
    }
}
```

### 3.4 PaymentController

```java
@RestController
@RequestMapping("/api/v1/payments")
@RequiredArgsConstructor
@Tag(name = "Payment")
public class PaymentController {
    private final PaymentService paymentService;

    @PostMapping("/create-intent/{orderId}")
    @Operation(summary = "Create Stripe PaymentIntent")
    public ResponseEntity<ApiResponse<PaymentIntentResponse>> createPaymentIntent(
            @PathVariable Long orderId, @CurrentUser Long userId) throws StripeException {
        return ResponseEntity.ok(
            ApiResponse.success(paymentService.createPaymentIntent(orderId, userId)));
    }

    @PostMapping("/webhook")
    @Operation(summary = "Stripe webhook handler")
    public ResponseEntity<Void> handleWebhook(
            @RequestBody String payload,
            @RequestHeader("Stripe-Signature") String sigHeader) {
        paymentService.handleWebhook(payload, sigHeader);
        return ResponseEntity.ok().build();
    }
}
```

### 3.5 Webhook Security — 3 Lớp Bảo Vệ

```
Lớp 1: SIGNATURE VERIFICATION
  Webhook.constructEvent() verify Stripe signature → reject nếu giả mạo.

Lớp 2: IDEMPOTENCY
  Stripe gửi cùng event nhiều lần (retry). Check "đã xử lý chưa?" trước khi update.

Lớp 3: HTTPS ONLY (production)
  Local dev dùng Stripe CLI forward webhook.
```

```bash
# Test webhook local với Stripe CLI
stripe login
stripe listen --forward-to localhost:8080/api/v1/payments/webhook
# → Copy whsec_xxx vào .env

stripe trigger payment_intent.succeeded  # test event
```

> Webhook endpoint phải **exclude khỏi Spring Security** (Stripe gọi không có JWT):

```java
// SecurityConfig.java
.requestMatchers("/api/v1/payments/webhook").permitAll()
```

---

## 4. COD (Cash on Delivery) Flow

### 4.1 So Sánh

```
STRIPE:  Checkout → PENDING → [Stripe] → Webhook → CONFIRMED  (async)
COD:     Checkout → CONFIRMED ngay lập tức (Payment vẫn PENDING cho đến khi giao)
```

### 4.2 Implementation

Trong `OrderService.checkout()`, phân biệt theo PaymentMethod:

```java
// Sau khi tạo Order + Payment (bước 6):
if (request.getPaymentMethod() == PaymentMethod.COD) {
    order.setStatus(OrderStatus.CONFIRMED);    // COD → confirm ngay
    payment.setStatus(PaymentStatus.PENDING);  // Đợi shipper thu tiền
} else {
    order.setStatus(OrderStatus.PENDING);      // STRIPE → đợi payment
    payment.setStatus(PaymentStatus.PENDING);
}
orderRepository.save(order);
paymentRepository.save(payment);
```

**Confirm COD khi shipper giao xong:**

```java
@Transactional
public void confirmCodPayment(Long orderId) {
    Order order = orderRepository.findById(orderId)
            .orElseThrow(() -> new ResourceNotFoundException("Order not found"));
    Payment payment = order.getPayment();
    if (payment.getMethod() != PaymentMethod.COD)
        throw new BadRequestException("Not a COD order");

    payment.setStatus(PaymentStatus.COMPLETED);
    order.setStatus(OrderStatus.DELIVERED);
    paymentRepository.save(payment);
    orderRepository.save(order);
}
```

### 4.3 Enums

```java
public enum PaymentMethod { STRIPE, COD }

public enum PaymentStatus { PENDING, COMPLETED, FAILED, REFUNDED }
```

---

## 5. Tổng Kết Ngày 6

### Full Checkout Flow

```
1. BROWSING:
   GET  /api/v1/products         → Xem sản phẩm
   POST /api/v1/cart             → Add to cart
   GET  /api/v1/cart             → Xem cart

2. CHECKOUT:
   POST /api/v1/orders/checkout  → Tạo đơn hàng
     ├─ Validate cart + stock → Reserve stock
     ├─ Create Order + OrderItems (snapshot) + Payment
     └─ Clear cart

3a. STRIPE:  POST /payments/create-intent/{id} → Stripe form → Webhook → CONFIRMED
3b. COD:     CONFIRMED ngay → Admin confirm khi giao xong

4. LIFECYCLE: CONFIRMED → PROCESSING → SHIPPED → DELIVERED
```

### Checklist

- [ ] CartItem entity + unique constraint
- [ ] Cart DTOs + CartService (5 methods) + CartController (5 endpoints)
- [ ] Redis cart cho anonymous + merge khi login
- [ ] OrderStatus enum với state machine
- [ ] OrderItem snapshot pattern
- [ ] OrderService: checkout, updateStatus, getUserOrders
- [ ] Pessimistic locking cho stock reservation
- [ ] Stripe: config + PaymentService + webhook handler
- [ ] Webhook: signature verification + idempotency + permitAll
- [ ] COD flow: confirm ngay + confirmCodPayment

### Bài Tập Tự Làm

1. **Wishlist:** Tương tự Cart nhưng không có quantity.
2. **Order Cancellation:** User hủy trong 30 phút. Sau đó chỉ admin hủy.
3. **Stock Alert:** Sản phẩm < 5 items → email admin (Spring Events).
4. **Coupon:** Entity `Coupon(code, discountPercent, expiryDate)`. Apply khi checkout.

---

> **Tiếp theo:** [Day 7 — Testing + Docker](./day-07-testing-docker.md)
