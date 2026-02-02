# Day 8: Distributed Transactions

## Mục tiêu
- CAP theorem & BASE
- Saga pattern
- Eventual consistency
- Outbox pattern

---

## 1. The Transaction Problem

### 1.1. Monolith vs Microservices Transactions

```
┌─────────────────────────────────────────────────────────────────┐
│                MONOLITH - ACID TRANSACTION                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   @Transactional                                                │
│   public void placeOrder(Order order) {                         │
│       orderRepository.save(order);        // Same DB            │
│       inventoryService.reserve(items);    // Same DB            │
│       paymentService.charge(amount);      // Same DB            │
│   }                                                              │
│                                                                  │
│   ✓ All succeed or all fail                                     │
│   ✓ Consistent state guaranteed                                 │
│   ✓ Simple rollback                                             │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│            MICROSERVICES - DISTRIBUTED TRANSACTION               │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Order Service         Inventory Service      Payment Service  │
│   ┌─────────────┐       ┌─────────────┐       ┌─────────────┐  │
│   │  Order DB   │       │Inventory DB │       │ Payment DB  │  │
│   └─────────────┘       └─────────────┘       └─────────────┘  │
│         │                      │                     │          │
│         ▼                      ▼                     ▼          │
│     Create Order  →    Reserve Items    →    Charge Payment    │
│         ✓                     ✓                     ✗          │
│                                                                  │
│   Problem: Order created, inventory reserved, but payment failed│
│   No automatic rollback across services!                        │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 1.2. CAP Theorem

```
┌─────────────────────────────────────────────────────────────────┐
│                    CAP THEOREM                                   │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│                    Consistency                                   │
│                        /\                                        │
│                       /  \                                       │
│                      /    \                                      │
│                     / CA   \                                     │
│                    /  (Not  \                                    │
│                   / possible)\                                   │
│                  /    during  \                                  │
│                 /   partition  \                                 │
│                /________________\                                │
│   Availability                    Partition                      │
│                                   Tolerance                      │
│                                                                  │
│   CA: RDBMS (single node)                                       │
│   CP: MongoDB, HBase (sacrifice availability)                   │
│   AP: Cassandra, DynamoDB (sacrifice consistency)               │
│                                                                  │
│   In distributed systems, P is mandatory                        │
│   → Must choose between C and A                                 │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 1.3. ACID vs BASE

```
┌────────────────────────┬────────────────────────────────────────┐
│         ACID           │                BASE                     │
├────────────────────────┼────────────────────────────────────────┤
│ Atomicity              │ Basically Available                    │
│ Consistency            │ Soft state                             │
│ Isolation              │ Eventually consistent                  │
│ Durability             │                                        │
├────────────────────────┼────────────────────────────────────────┤
│ Strong consistency     │ Eventual consistency                   │
│ Pessimistic locking    │ Optimistic approach                    │
│ Single database        │ Distributed databases                  │
│ Lower scalability      │ High scalability                       │
│ Simpler reasoning      │ Complex failure handling               │
└────────────────────────┴────────────────────────────────────────┘
```

---

## 2. Saga Pattern

### 2.1. What is Saga?

```
┌─────────────────────────────────────────────────────────────────┐
│                    SAGA PATTERN                                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   A Saga is a sequence of local transactions where each         │
│   transaction updates data within a single service.             │
│                                                                  │
│   If a step fails, compensating transactions are executed       │
│   to undo the changes made by preceding transactions.           │
│                                                                  │
│   Order Saga Example:                                           │
│                                                                  │
│   T1: Create Order (PENDING)                                    │
│    │                                                            │
│    ▼                                                            │
│   T2: Reserve Inventory                                         │
│    │                                                            │
│    ▼                                                            │
│   T3: Process Payment                                           │
│    │                                                            │
│    ▼                                                            │
│   T4: Confirm Order (CONFIRMED)                                 │
│                                                                  │
│   If T3 fails:                                                  │
│   C2: Release Inventory (compensate T2)                         │
│   C1: Cancel Order (compensate T1)                              │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 2.2. Choreography Saga

```
┌─────────────────────────────────────────────────────────────────┐
│                CHOREOGRAPHY SAGA                                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Each service publishes events that trigger next step          │
│                                                                  │
│   ┌─────────┐      ┌─────────┐      ┌─────────┐                │
│   │ Order   │      │Inventory│      │ Payment │                │
│   │ Service │      │ Service │      │ Service │                │
│   └────┬────┘      └────┬────┘      └────┬────┘                │
│        │                │                │                      │
│   1. Create Order       │                │                      │
│        │                │                │                      │
│        ├─── OrderCreated Event ───►     │                      │
│        │                │                │                      │
│        │         2. Reserve Inventory    │                      │
│        │                │                │                      │
│        │                ├─── InventoryReserved ───►            │
│        │                │                │                      │
│        │                │         3. Process Payment            │
│        │                │                │                      │
│        │◄─── PaymentCompleted ───────────┤                      │
│        │                │                │                      │
│   4. Confirm Order      │                │                      │
│                                                                  │
│   Pros: Simple, decoupled                                       │
│   Cons: Hard to track, cyclic dependencies                      │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 2.3. Orchestration Saga

```
┌─────────────────────────────────────────────────────────────────┐
│                ORCHESTRATION SAGA                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Orchestrator coordinates all steps                            │
│                                                                  │
│              ┌──────────────────┐                               │
│              │ Saga Orchestrator│                               │
│              │  (Order Saga)    │                               │
│              └────────┬─────────┘                               │
│                       │                                         │
│           ┌───────────┼───────────┐                            │
│           │           │           │                            │
│           ▼           ▼           ▼                            │
│      ┌─────────┐ ┌─────────┐ ┌─────────┐                      │
│      │ Order   │ │Inventory│ │ Payment │                      │
│      │ Service │ │ Service │ │ Service │                      │
│      └─────────┘ └─────────┘ └─────────┘                      │
│                                                                  │
│   Flow:                                                         │
│   1. Orchestrator → Order Service: Create Order                 │
│   2. Orchestrator → Inventory Service: Reserve                  │
│   3. Orchestrator → Payment Service: Charge                     │
│   4. Orchestrator → Order Service: Confirm                      │
│                                                                  │
│   Pros: Clear flow, easy to track, centralized logic           │
│   Cons: Orchestrator can be bottleneck                          │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## 3. Choreography Implementation

### 3.1. Event Classes

```java
// Base Event
public abstract class DomainEvent {
    private String eventId;
    private LocalDateTime occurredAt;
    private String correlationId;

    public DomainEvent() {
        this.eventId = UUID.randomUUID().toString();
        this.occurredAt = LocalDateTime.now();
    }
}

// Order Events
public class OrderCreatedEvent extends DomainEvent {
    private String orderId;
    private String customerId;
    private List<OrderItem> items;
    private BigDecimal totalAmount;
}

public class OrderConfirmedEvent extends DomainEvent {
    private String orderId;
}

public class OrderCancelledEvent extends DomainEvent {
    private String orderId;
    private String reason;
}

// Inventory Events
public class InventoryReservedEvent extends DomainEvent {
    private String orderId;
    private List<ReservedItem> items;
}

public class InventoryReservationFailedEvent extends DomainEvent {
    private String orderId;
    private String reason;
}

// Payment Events
public class PaymentCompletedEvent extends DomainEvent {
    private String orderId;
    private String paymentId;
    private BigDecimal amount;
}

public class PaymentFailedEvent extends DomainEvent {
    private String orderId;
    private String reason;
}
```

### 3.2. Order Service

```java
@Service
public class OrderService {

    private final OrderRepository orderRepository;
    private final KafkaTemplate<String, Object> kafkaTemplate;

    @Transactional
    public Order createOrder(CreateOrderRequest request) {
        // Create order in PENDING state
        Order order = Order.builder()
            .id(UUID.randomUUID().toString())
            .customerId(request.getCustomerId())
            .items(request.getItems())
            .totalAmount(calculateTotal(request.getItems()))
            .status(OrderStatus.PENDING)
            .build();

        orderRepository.save(order);

        // Publish event
        OrderCreatedEvent event = new OrderCreatedEvent(order);
        kafkaTemplate.send("order-events", order.getId(), event);

        return order;
    }

    @KafkaListener(topics = "payment-events")
    public void handlePaymentEvent(Object event) {
        if (event instanceof PaymentCompletedEvent completed) {
            confirmOrder(completed.getOrderId());
        } else if (event instanceof PaymentFailedEvent failed) {
            cancelOrder(failed.getOrderId(), failed.getReason());
        }
    }

    @KafkaListener(topics = "inventory-events")
    public void handleInventoryEvent(Object event) {
        if (event instanceof InventoryReservationFailedEvent failed) {
            cancelOrder(failed.getOrderId(), failed.getReason());
        }
    }

    @Transactional
    private void confirmOrder(String orderId) {
        Order order = orderRepository.findById(orderId)
            .orElseThrow();
        order.setStatus(OrderStatus.CONFIRMED);
        orderRepository.save(order);

        kafkaTemplate.send("order-events", orderId, new OrderConfirmedEvent(orderId));
    }

    @Transactional
    private void cancelOrder(String orderId, String reason) {
        Order order = orderRepository.findById(orderId)
            .orElseThrow();
        order.setStatus(OrderStatus.CANCELLED);
        order.setCancellationReason(reason);
        orderRepository.save(order);

        kafkaTemplate.send("order-events", orderId, new OrderCancelledEvent(orderId, reason));
    }
}
```

### 3.3. Inventory Service

```java
@Service
public class InventoryService {

    private final InventoryRepository inventoryRepository;
    private final ReservationRepository reservationRepository;
    private final KafkaTemplate<String, Object> kafkaTemplate;

    @KafkaListener(topics = "order-events")
    public void handleOrderEvent(Object event) {
        if (event instanceof OrderCreatedEvent created) {
            reserveInventory(created);
        } else if (event instanceof OrderCancelledEvent cancelled) {
            releaseInventory(cancelled.getOrderId());
        }
    }

    @Transactional
    private void reserveInventory(OrderCreatedEvent event) {
        try {
            List<ReservedItem> reservedItems = new ArrayList<>();

            for (OrderItem item : event.getItems()) {
                Inventory inventory = inventoryRepository
                    .findByProductId(item.getProductId())
                    .orElseThrow(() -> new ProductNotFoundException(item.getProductId()));

                if (inventory.getAvailableQuantity() < item.getQuantity()) {
                    throw new InsufficientInventoryException(item.getProductId());
                }

                inventory.reserve(item.getQuantity());
                inventoryRepository.save(inventory);

                Reservation reservation = Reservation.builder()
                    .orderId(event.getOrderId())
                    .productId(item.getProductId())
                    .quantity(item.getQuantity())
                    .build();
                reservationRepository.save(reservation);

                reservedItems.add(new ReservedItem(item.getProductId(), item.getQuantity()));
            }

            // Success - publish event
            kafkaTemplate.send("inventory-events", event.getOrderId(),
                new InventoryReservedEvent(event.getOrderId(), reservedItems));

        } catch (Exception e) {
            // Failure - publish event
            kafkaTemplate.send("inventory-events", event.getOrderId(),
                new InventoryReservationFailedEvent(event.getOrderId(), e.getMessage()));
        }
    }

    @Transactional
    private void releaseInventory(String orderId) {
        List<Reservation> reservations = reservationRepository.findByOrderId(orderId);

        for (Reservation reservation : reservations) {
            Inventory inventory = inventoryRepository
                .findByProductId(reservation.getProductId())
                .orElseThrow();

            inventory.release(reservation.getQuantity());
            inventoryRepository.save(inventory);
        }

        reservationRepository.deleteByOrderId(orderId);
    }
}
```

### 3.4. Payment Service

```java
@Service
public class PaymentService {

    private final PaymentRepository paymentRepository;
    private final PaymentGateway paymentGateway;
    private final KafkaTemplate<String, Object> kafkaTemplate;

    @KafkaListener(topics = "inventory-events")
    public void handleInventoryEvent(Object event) {
        if (event instanceof InventoryReservedEvent reserved) {
            processPayment(reserved);
        }
    }

    @KafkaListener(topics = "order-events")
    public void handleOrderEvent(Object event) {
        if (event instanceof OrderCancelledEvent cancelled) {
            refundPayment(cancelled.getOrderId());
        }
    }

    @Transactional
    private void processPayment(InventoryReservedEvent event) {
        try {
            // Get order details for payment
            BigDecimal amount = calculateAmount(event);

            PaymentResult result = paymentGateway.charge(
                event.getOrderId(),
                amount
            );

            Payment payment = Payment.builder()
                .id(UUID.randomUUID().toString())
                .orderId(event.getOrderId())
                .amount(amount)
                .status(PaymentStatus.COMPLETED)
                .transactionId(result.getTransactionId())
                .build();
            paymentRepository.save(payment);

            // Success
            kafkaTemplate.send("payment-events", event.getOrderId(),
                new PaymentCompletedEvent(event.getOrderId(), payment.getId(), amount));

        } catch (Exception e) {
            // Failure
            kafkaTemplate.send("payment-events", event.getOrderId(),
                new PaymentFailedEvent(event.getOrderId(), e.getMessage()));
        }
    }

    @Transactional
    private void refundPayment(String orderId) {
        Payment payment = paymentRepository.findByOrderId(orderId)
            .orElse(null);

        if (payment != null && payment.getStatus() == PaymentStatus.COMPLETED) {
            paymentGateway.refund(payment.getTransactionId());
            payment.setStatus(PaymentStatus.REFUNDED);
            paymentRepository.save(payment);
        }
    }
}
```

---

## 4. Orchestration Implementation

### 4.1. Saga Orchestrator

```java
@Service
public class OrderSagaOrchestrator {

    private final OrderServiceClient orderClient;
    private final InventoryServiceClient inventoryClient;
    private final PaymentServiceClient paymentClient;
    private final SagaStateRepository sagaStateRepository;

    @Transactional
    public SagaResult executeOrderSaga(CreateOrderRequest request) {
        String sagaId = UUID.randomUUID().toString();
        SagaState state = new SagaState(sagaId, SagaStatus.STARTED);
        sagaStateRepository.save(state);

        try {
            // Step 1: Create Order
            state.setCurrentStep("CREATE_ORDER");
            Order order = orderClient.createOrder(request);
            state.setOrderId(order.getId());
            sagaStateRepository.save(state);

            // Step 2: Reserve Inventory
            state.setCurrentStep("RESERVE_INVENTORY");
            ReservationResult reservation = inventoryClient.reserve(
                order.getId(), order.getItems());
            state.setReservationId(reservation.getId());
            sagaStateRepository.save(state);

            // Step 3: Process Payment
            state.setCurrentStep("PROCESS_PAYMENT");
            PaymentResult payment = paymentClient.charge(
                order.getId(), order.getTotalAmount());
            state.setPaymentId(payment.getId());
            sagaStateRepository.save(state);

            // Step 4: Confirm Order
            state.setCurrentStep("CONFIRM_ORDER");
            orderClient.confirmOrder(order.getId());

            // Success
            state.setStatus(SagaStatus.COMPLETED);
            sagaStateRepository.save(state);

            return SagaResult.success(order.getId());

        } catch (Exception e) {
            log.error("Saga failed at step {}: {}", state.getCurrentStep(), e.getMessage());
            compensate(state, e);
            return SagaResult.failed(e.getMessage());
        }
    }

    private void compensate(SagaState state, Exception originalError) {
        state.setStatus(SagaStatus.COMPENSATING);
        sagaStateRepository.save(state);

        List<String> compensationErrors = new ArrayList<>();

        // Compensate in reverse order
        switch (state.getCurrentStep()) {
            case "CONFIRM_ORDER":
            case "PROCESS_PAYMENT":
                // Compensate payment
                if (state.getPaymentId() != null) {
                    try {
                        paymentClient.refund(state.getPaymentId());
                    } catch (Exception e) {
                        compensationErrors.add("Payment refund failed: " + e.getMessage());
                    }
                }
                // Fall through

            case "RESERVE_INVENTORY":
                // Compensate inventory
                if (state.getReservationId() != null) {
                    try {
                        inventoryClient.releaseReservation(state.getReservationId());
                    } catch (Exception e) {
                        compensationErrors.add("Inventory release failed: " + e.getMessage());
                    }
                }
                // Fall through

            case "CREATE_ORDER":
                // Compensate order
                if (state.getOrderId() != null) {
                    try {
                        orderClient.cancelOrder(state.getOrderId(), originalError.getMessage());
                    } catch (Exception e) {
                        compensationErrors.add("Order cancel failed: " + e.getMessage());
                    }
                }
                break;
        }

        if (compensationErrors.isEmpty()) {
            state.setStatus(SagaStatus.COMPENSATED);
        } else {
            state.setStatus(SagaStatus.COMPENSATION_FAILED);
            state.setCompensationErrors(compensationErrors);
        }
        sagaStateRepository.save(state);
    }
}
```

### 4.2. Saga State Machine

```java
// Using Spring State Machine
@Configuration
@EnableStateMachine
public class OrderSagaStateMachineConfig
    extends StateMachineConfigurerAdapter<SagaState, SagaEvent> {

    @Override
    public void configure(StateMachineStateConfigurer<SagaState, SagaEvent> states)
            throws Exception {
        states
            .withStates()
            .initial(SagaState.STARTED)
            .state(SagaState.ORDER_CREATED)
            .state(SagaState.INVENTORY_RESERVED)
            .state(SagaState.PAYMENT_PROCESSED)
            .end(SagaState.COMPLETED)
            .end(SagaState.COMPENSATED);
    }

    @Override
    public void configure(StateMachineTransitionConfigurer<SagaState, SagaEvent> transitions)
            throws Exception {
        transitions
            // Happy path
            .withExternal()
                .source(SagaState.STARTED).target(SagaState.ORDER_CREATED)
                .event(SagaEvent.ORDER_CREATE_SUCCESS)
                .action(createOrderAction())
            .and()
            .withExternal()
                .source(SagaState.ORDER_CREATED).target(SagaState.INVENTORY_RESERVED)
                .event(SagaEvent.INVENTORY_RESERVE_SUCCESS)
                .action(reserveInventoryAction())
            .and()
            .withExternal()
                .source(SagaState.INVENTORY_RESERVED).target(SagaState.PAYMENT_PROCESSED)
                .event(SagaEvent.PAYMENT_SUCCESS)
                .action(processPaymentAction())
            .and()
            .withExternal()
                .source(SagaState.PAYMENT_PROCESSED).target(SagaState.COMPLETED)
                .event(SagaEvent.ORDER_CONFIRM_SUCCESS)

            // Compensation path
            .and()
            .withExternal()
                .source(SagaState.ORDER_CREATED).target(SagaState.COMPENSATED)
                .event(SagaEvent.INVENTORY_RESERVE_FAILED)
                .action(compensateOrderAction())
            .and()
            .withExternal()
                .source(SagaState.INVENTORY_RESERVED).target(SagaState.COMPENSATED)
                .event(SagaEvent.PAYMENT_FAILED)
                .action(compensateInventoryAndOrderAction());
    }
}
```

---

## 5. Outbox Pattern

### 5.1. Why Outbox?

```
┌─────────────────────────────────────────────────────────────────┐
│                    DUAL WRITE PROBLEM                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   @Transactional                                                │
│   public void createOrder(Order order) {                        │
│       orderRepository.save(order);     // ✓ Committed           │
│       kafkaTemplate.send(event);       // ✗ Failed!             │
│   }                                                              │
│                                                                  │
│   Problem: Data in DB but event not published                   │
│   Other services don't know about the order                     │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│                    OUTBOX PATTERN                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   @Transactional                                                │
│   public void createOrder(Order order) {                        │
│       orderRepository.save(order);                              │
│       outboxRepository.save(new OutboxEvent(order));            │
│   }                                                              │
│                                                                  │
│   // Separate process publishes events                          │
│   @Scheduled(fixedDelay = 1000)                                 │
│   public void publishEvents() {                                 │
│       List<OutboxEvent> events = outboxRepository.findPending();│
│       for (OutboxEvent event : events) {                        │
│           kafkaTemplate.send(event);                            │
│           event.markPublished();                                │
│       }                                                          │
│   }                                                              │
│                                                                  │
│   ✓ Atomic: both order and event in same transaction           │
│   ✓ Reliable: events always published                           │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 5.2. Implementation

```java
// Outbox Entity
@Entity
@Table(name = "outbox_events")
public class OutboxEvent {

    @Id
    private String id;

    private String aggregateType;
    private String aggregateId;
    private String eventType;

    @Column(columnDefinition = "TEXT")
    private String payload;

    @Enumerated(EnumType.STRING)
    private OutboxStatus status;

    private LocalDateTime createdAt;
    private LocalDateTime publishedAt;
    private int retryCount;

    public static OutboxEvent create(String aggregateType, String aggregateId,
            String eventType, Object event) {
        OutboxEvent outbox = new OutboxEvent();
        outbox.setId(UUID.randomUUID().toString());
        outbox.setAggregateType(aggregateType);
        outbox.setAggregateId(aggregateId);
        outbox.setEventType(eventType);
        outbox.setPayload(JsonUtils.toJson(event));
        outbox.setStatus(OutboxStatus.PENDING);
        outbox.setCreatedAt(LocalDateTime.now());
        outbox.setRetryCount(0);
        return outbox;
    }
}

// Repository
public interface OutboxRepository extends JpaRepository<OutboxEvent, String> {

    @Query("SELECT e FROM OutboxEvent e WHERE e.status = 'PENDING' " +
           "AND e.retryCount < :maxRetries ORDER BY e.createdAt")
    List<OutboxEvent> findPendingEvents(@Param("maxRetries") int maxRetries, Pageable pageable);

    @Modifying
    @Query("DELETE FROM OutboxEvent e WHERE e.status = 'PUBLISHED' " +
           "AND e.publishedAt < :before")
    int deletePublishedBefore(@Param("before") LocalDateTime before);
}

// Service with Outbox
@Service
public class OrderService {

    private final OrderRepository orderRepository;
    private final OutboxRepository outboxRepository;

    @Transactional
    public Order createOrder(CreateOrderRequest request) {
        Order order = Order.builder()
            .id(UUID.randomUUID().toString())
            .customerId(request.getCustomerId())
            .items(request.getItems())
            .status(OrderStatus.PENDING)
            .build();

        orderRepository.save(order);

        // Create outbox event in same transaction
        OutboxEvent outboxEvent = OutboxEvent.create(
            "Order",
            order.getId(),
            "OrderCreated",
            new OrderCreatedEvent(order)
        );
        outboxRepository.save(outboxEvent);

        return order;
    }
}

// Outbox Publisher
@Component
public class OutboxPublisher {

    private final OutboxRepository outboxRepository;
    private final KafkaTemplate<String, String> kafkaTemplate;

    @Scheduled(fixedDelay = 1000)
    @Transactional
    public void publishPendingEvents() {
        List<OutboxEvent> events = outboxRepository.findPendingEvents(
            3, PageRequest.of(0, 100));

        for (OutboxEvent event : events) {
            try {
                String topic = event.getAggregateType().toLowerCase() + "-events";

                kafkaTemplate.send(topic, event.getAggregateId(), event.getPayload())
                    .get(5, TimeUnit.SECONDS);  // Wait for confirmation

                event.setStatus(OutboxStatus.PUBLISHED);
                event.setPublishedAt(LocalDateTime.now());

            } catch (Exception e) {
                log.error("Failed to publish event {}: {}", event.getId(), e.getMessage());
                event.setRetryCount(event.getRetryCount() + 1);
                if (event.getRetryCount() >= 3) {
                    event.setStatus(OutboxStatus.FAILED);
                }
            }
            outboxRepository.save(event);
        }
    }

    // Cleanup old published events
    @Scheduled(cron = "0 0 * * * *")  // Every hour
    @Transactional
    public void cleanupOldEvents() {
        LocalDateTime before = LocalDateTime.now().minusDays(7);
        int deleted = outboxRepository.deletePublishedBefore(before);
        log.info("Cleaned up {} old outbox events", deleted);
    }
}
```

### 5.3. Debezium CDC Alternative

```yaml
# docker-compose.yml
version: '3.8'
services:
  debezium:
    image: debezium/connect:2.4
    ports:
      - "8083:8083"
    environment:
      BOOTSTRAP_SERVERS: kafka:9092
      GROUP_ID: debezium
      CONFIG_STORAGE_TOPIC: debezium_configs
      OFFSET_STORAGE_TOPIC: debezium_offsets
      STATUS_STORAGE_TOPIC: debezium_statuses

# Register connector
# POST http://localhost:8083/connectors
{
  "name": "outbox-connector",
  "config": {
    "connector.class": "io.debezium.connector.mysql.MySqlConnector",
    "database.hostname": "mysql",
    "database.port": "3306",
    "database.user": "debezium",
    "database.password": "dbz",
    "database.server.id": "1",
    "database.server.name": "order-service",
    "table.include.list": "orders.outbox_events",
    "transforms": "outbox",
    "transforms.outbox.type": "io.debezium.transforms.outbox.EventRouter",
    "transforms.outbox.table.field.event.key": "aggregate_id",
    "transforms.outbox.table.field.event.payload": "payload",
    "transforms.outbox.route.topic.replacement": "${routedByValue}"
  }
}
```

---

## 6. Idempotency

### 6.1. Idempotent Consumer

```java
@Entity
@Table(name = "processed_events")
public class ProcessedEvent {
    @Id
    private String eventId;
    private LocalDateTime processedAt;
}

@Service
public class PaymentService {

    private final ProcessedEventRepository processedEventRepository;

    @Transactional
    public void handleInventoryReserved(InventoryReservedEvent event) {
        // Check if already processed
        if (processedEventRepository.existsById(event.getEventId())) {
            log.info("Event {} already processed, skipping", event.getEventId());
            return;
        }

        // Process event
        processPayment(event);

        // Mark as processed
        processedEventRepository.save(
            new ProcessedEvent(event.getEventId(), LocalDateTime.now()));
    }
}
```

### 6.2. Idempotency Key

```java
@RestController
public class PaymentController {

    @PostMapping("/payments")
    public ResponseEntity<Payment> createPayment(
            @RequestHeader("Idempotency-Key") String idempotencyKey,
            @RequestBody PaymentRequest request) {

        // Check for existing payment with same key
        Optional<Payment> existing = paymentRepository
            .findByIdempotencyKey(idempotencyKey);

        if (existing.isPresent()) {
            return ResponseEntity.ok(existing.get());
        }

        // Create new payment
        Payment payment = paymentService.createPayment(request, idempotencyKey);
        return ResponseEntity.status(HttpStatus.CREATED).body(payment);
    }
}
```

---

## 7. Best Practices

```
┌─────────────────────────────────────────────────────────────────┐
│                    SAGA BEST PRACTICES                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   1. Design compensating transactions carefully                 │
│      - Every step needs a compensation                          │
│      - Compensations must be idempotent                         │
│                                                                  │
│   2. Handle partial failures                                    │
│      - Store saga state persistently                            │
│      - Support resume/retry from any step                       │
│                                                                  │
│   3. Implement timeouts                                         │
│      - Don't wait forever for responses                         │
│      - Trigger compensation on timeout                          │
│                                                                  │
│   4. Use correlation IDs                                        │
│      - Track events across services                             │
│      - Enable distributed tracing                               │
│                                                                  │
│   5. Monitor saga execution                                     │
│      - Track success/failure rates                              │
│      - Alert on stuck sagas                                     │
│                                                                  │
│   6. Test failure scenarios                                     │
│      - Each step failure                                        │
│      - Network partitions                                       │
│      - Service unavailability                                   │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## 8. Bài tập thực hành

### Bài 1: Choreography Saga
Implement Order Saga với Kafka events cho 3 services.

### Bài 2: Orchestration Saga
Implement Order Saga orchestrator với state machine.

### Bài 3: Outbox Pattern
Implement outbox pattern cho reliable event publishing.

### Bài 4: Idempotent Consumer
Implement idempotent event consumer với deduplication.

---

## 9. Key Takeaways

| Pattern | Use Case | Complexity |
|---------|----------|------------|
| Choreography | Simple flows, few services | Low |
| Orchestration | Complex flows, many services | Medium |
| Outbox | Reliable event publishing | Medium |
| Idempotency | Exactly-once processing | Low |

```
Distributed Transaction Guidelines:
1. Prefer eventual consistency over strong consistency
2. Design for failure and compensation
3. Use idempotent operations
4. Track saga state persistently
5. Monitor and alert on anomalies
6. Test compensation thoroughly
```

---

## Navigation

- [← Day 7: Resilience Patterns](./day-07-resilience-patterns.md)
- [Day 9: Security in Microservices →](./day-09-microservices-security.md)
