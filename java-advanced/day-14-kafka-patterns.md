# Day 14: Kafka Patterns

## Mục tiêu
- Event Sourcing với Kafka
- CQRS pattern
- Outbox pattern
- Dead Letter Queue pattern

---

## 1. Event Sourcing

### 1.1. Concept

```
┌─────────────────────────────────────────────────────────────────┐
│                    EVENT SOURCING                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Traditional (State-based):                                    │
│   ┌─────────────────────────────────────────────────────────┐  │
│   │  Order Table                                             │  │
│   │  ┌────────┬────────┬─────────┬───────────┐              │  │
│   │  │   ID   │ Status │  Total  │ Updated   │              │  │
│   │  ├────────┼────────┼─────────┼───────────┤              │  │
│   │  │ ORD-1  │ SHIPPED│  100.00 │ 2024-01-15│              │  │
│   │  └────────┴────────┴─────────┴───────────┘              │  │
│   │  Only current state stored                               │  │
│   └─────────────────────────────────────────────────────────┘  │
│                                                                  │
│   Event Sourcing:                                               │
│   ┌─────────────────────────────────────────────────────────┐  │
│   │  Order Events Stream                                     │  │
│   │  ┌──────────────────────────────────────────────────┐   │  │
│   │  │ OrderCreated(ORD-1, items, $100)    │ offset: 0  │   │  │
│   │  │ OrderConfirmed(ORD-1)               │ offset: 1  │   │  │
│   │  │ PaymentReceived(ORD-1, $100)        │ offset: 2  │   │  │
│   │  │ OrderShipped(ORD-1, tracking)       │ offset: 3  │   │  │
│   │  └──────────────────────────────────────────────────┘   │  │
│   │  Complete history stored                                 │  │
│   └─────────────────────────────────────────────────────────┘  │
│                                                                  │
│   Benefits:                                                      │
│   ├── Complete audit trail                                      │
│   ├── Temporal queries (state at any point in time)             │
│   ├── Event replay                                              │
│   └── Debugging and analysis                                    │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 1.2. Implementation

```java
// Domain Events
public sealed interface OrderEvent permits
        OrderCreated, OrderConfirmed, OrderShipped, OrderCancelled {

    String orderId();
    Instant occurredAt();
}

public record OrderCreated(
    String orderId,
    String customerId,
    List<OrderItem> items,
    BigDecimal totalAmount,
    Instant occurredAt
) implements OrderEvent {}

public record OrderConfirmed(
    String orderId,
    Instant occurredAt
) implements OrderEvent {}

public record OrderShipped(
    String orderId,
    String trackingNumber,
    Instant occurredAt
) implements OrderEvent {}

public record OrderCancelled(
    String orderId,
    String reason,
    Instant occurredAt
) implements OrderEvent {}

// Event Store
@Service
@RequiredArgsConstructor
public class KafkaEventStore {

    private final KafkaTemplate<String, OrderEvent> kafkaTemplate;
    private static final String TOPIC = "order-events";

    public void append(OrderEvent event) {
        kafkaTemplate.send(TOPIC, event.orderId(), event)
            .whenComplete((result, ex) -> {
                if (ex != null) {
                    throw new EventStoreException("Failed to append event", ex);
                }
            });
    }

    public void appendAll(String orderId, List<OrderEvent> events) {
        kafkaTemplate.executeInTransaction(ops -> {
            events.forEach(event -> ops.send(TOPIC, orderId, event));
            return null;
        });
    }
}

// Order Aggregate
public class Order {

    private String id;
    private String customerId;
    private List<OrderItem> items = new ArrayList<>();
    private BigDecimal totalAmount;
    private OrderStatus status;
    private String trackingNumber;
    private List<OrderEvent> uncommittedEvents = new ArrayList<>();

    // Reconstruct from events
    public static Order fromEvents(List<OrderEvent> events) {
        Order order = new Order();
        events.forEach(order::apply);
        return order;
    }

    // Apply event to state
    private void apply(OrderEvent event) {
        switch (event) {
            case OrderCreated e -> {
                this.id = e.orderId();
                this.customerId = e.customerId();
                this.items = new ArrayList<>(e.items());
                this.totalAmount = e.totalAmount();
                this.status = OrderStatus.PENDING;
            }
            case OrderConfirmed e -> {
                this.status = OrderStatus.CONFIRMED;
            }
            case OrderShipped e -> {
                this.status = OrderStatus.SHIPPED;
                this.trackingNumber = e.trackingNumber();
            }
            case OrderCancelled e -> {
                this.status = OrderStatus.CANCELLED;
            }
        }
    }

    // Command handlers
    public void confirm() {
        if (status != OrderStatus.PENDING) {
            throw new IllegalStateException("Can only confirm pending orders");
        }
        raiseEvent(new OrderConfirmed(id, Instant.now()));
    }

    public void ship(String trackingNumber) {
        if (status != OrderStatus.CONFIRMED) {
            throw new IllegalStateException("Can only ship confirmed orders");
        }
        raiseEvent(new OrderShipped(id, trackingNumber, Instant.now()));
    }

    private void raiseEvent(OrderEvent event) {
        uncommittedEvents.add(event);
        apply(event);
    }

    public List<OrderEvent> getUncommittedEvents() {
        return Collections.unmodifiableList(uncommittedEvents);
    }

    public void markEventsAsCommitted() {
        uncommittedEvents.clear();
    }
}

// Order Service
@Service
@RequiredArgsConstructor
public class OrderService {

    private final KafkaEventStore eventStore;
    private final OrderEventReader eventReader;

    public String createOrder(CreateOrderCommand command) {
        String orderId = UUID.randomUUID().toString();

        OrderCreated event = new OrderCreated(
            orderId,
            command.customerId(),
            command.items(),
            calculateTotal(command.items()),
            Instant.now()
        );

        eventStore.append(event);
        return orderId;
    }

    public void confirmOrder(String orderId) {
        Order order = loadOrder(orderId);
        order.confirm();
        eventStore.appendAll(orderId, order.getUncommittedEvents());
    }

    public void shipOrder(String orderId, String trackingNumber) {
        Order order = loadOrder(orderId);
        order.ship(trackingNumber);
        eventStore.appendAll(orderId, order.getUncommittedEvents());
    }

    private Order loadOrder(String orderId) {
        List<OrderEvent> events = eventReader.readEvents(orderId);
        if (events.isEmpty()) {
            throw new OrderNotFoundException(orderId);
        }
        return Order.fromEvents(events);
    }
}
```

### 1.3. Snapshot Pattern

```java
// Snapshot for performance
@Entity
@Table(name = "order_snapshots")
public class OrderSnapshot {

    @Id
    private String orderId;
    private String customerId;
    private BigDecimal totalAmount;
    private OrderStatus status;
    private String trackingNumber;
    private long lastEventOffset;
    private Instant snapshotAt;
}

@Service
public class SnapshotOrderService {

    private final SnapshotRepository snapshotRepository;
    private final OrderEventReader eventReader;

    public Order loadOrder(String orderId) {
        // Try to load snapshot
        Optional<OrderSnapshot> snapshot = snapshotRepository.findById(orderId);

        if (snapshot.isPresent()) {
            // Load events since snapshot
            List<OrderEvent> events = eventReader.readEventsSince(
                orderId, snapshot.get().getLastEventOffset());

            Order order = Order.fromSnapshot(snapshot.get());
            events.forEach(order::apply);
            return order;
        }

        // No snapshot - load all events
        List<OrderEvent> events = eventReader.readEvents(orderId);
        return Order.fromEvents(events);
    }

    // Create snapshot every N events
    @Scheduled(fixedDelay = 60000)
    public void createSnapshots() {
        List<String> ordersNeedingSnapshot = findOrdersWithManyEvents();

        for (String orderId : ordersNeedingSnapshot) {
            Order order = loadOrderFull(orderId);
            snapshotRepository.save(order.toSnapshot());
        }
    }
}
```

---

## 2. CQRS Pattern

### 2.1. Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                    CQRS PATTERN                                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Command Query Responsibility Segregation                      │
│                                                                  │
│   ┌─────────────────┐          ┌─────────────────┐             │
│   │   Commands      │          │    Queries      │             │
│   │  (Write Side)   │          │  (Read Side)    │             │
│   └────────┬────────┘          └────────┬────────┘             │
│            │                            │                       │
│            ▼                            ▼                       │
│   ┌─────────────────┐          ┌─────────────────┐             │
│   │ Command Handler │          │  Query Handler  │             │
│   └────────┬────────┘          └────────┬────────┘             │
│            │                            │                       │
│            ▼                            ▼                       │
│   ┌─────────────────┐          ┌─────────────────┐             │
│   │  Event Store    │──Events──│  Read Model     │             │
│   │  (Kafka)        │          │  (Database)     │             │
│   └─────────────────┘          └─────────────────┘             │
│                                                                  │
│   Benefits:                                                      │
│   ├── Optimized read/write models                               │
│   ├── Independent scaling                                       │
│   ├── Complex queries without affecting writes                  │
│   └── Different databases for read/write                        │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 2.2. Implementation

```java
// Commands
public sealed interface OrderCommand permits
        CreateOrderCommand, ConfirmOrderCommand, ShipOrderCommand {
}

public record CreateOrderCommand(
    String customerId,
    List<OrderItem> items
) implements OrderCommand {}

public record ConfirmOrderCommand(String orderId) implements OrderCommand {}

public record ShipOrderCommand(String orderId, String trackingNumber) implements OrderCommand {}

// Command Handler
@Service
@RequiredArgsConstructor
public class OrderCommandHandler {

    private final KafkaEventStore eventStore;
    private final OrderEventReader eventReader;

    public String handle(CreateOrderCommand command) {
        String orderId = UUID.randomUUID().toString();

        OrderCreated event = new OrderCreated(
            orderId,
            command.customerId(),
            command.items(),
            calculateTotal(command.items()),
            Instant.now()
        );

        eventStore.append(event);
        return orderId;
    }

    public void handle(ConfirmOrderCommand command) {
        Order order = loadOrder(command.orderId());
        order.confirm();
        eventStore.appendAll(order.getId(), order.getUncommittedEvents());
    }

    public void handle(ShipOrderCommand command) {
        Order order = loadOrder(command.orderId());
        order.ship(command.trackingNumber());
        eventStore.appendAll(order.getId(), order.getUncommittedEvents());
    }
}

// Read Model
@Entity
@Table(name = "order_read_model")
public class OrderReadModel {

    @Id
    private String id;
    private String customerId;
    private String customerName;  // Denormalized
    private String customerEmail; // Denormalized
    private BigDecimal totalAmount;
    private int itemCount;
    private OrderStatus status;
    private String trackingNumber;
    private Instant createdAt;
    private Instant updatedAt;
}

// Event Projector - Updates Read Model
@Service
@RequiredArgsConstructor
public class OrderProjector {

    private final OrderReadModelRepository readRepository;
    private final CustomerClient customerClient;

    @KafkaListener(topics = "order-events", groupId = "order-projector")
    public void project(OrderEvent event, Acknowledgment ack) {
        switch (event) {
            case OrderCreated e -> handleOrderCreated(e);
            case OrderConfirmed e -> handleOrderConfirmed(e);
            case OrderShipped e -> handleOrderShipped(e);
            case OrderCancelled e -> handleOrderCancelled(e);
        }
        ack.acknowledge();
    }

    private void handleOrderCreated(OrderCreated event) {
        // Fetch customer details for denormalization
        Customer customer = customerClient.getCustomer(event.customerId());

        OrderReadModel model = new OrderReadModel();
        model.setId(event.orderId());
        model.setCustomerId(event.customerId());
        model.setCustomerName(customer.getName());
        model.setCustomerEmail(customer.getEmail());
        model.setTotalAmount(event.totalAmount());
        model.setItemCount(event.items().size());
        model.setStatus(OrderStatus.PENDING);
        model.setCreatedAt(event.occurredAt());
        model.setUpdatedAt(event.occurredAt());

        readRepository.save(model);
    }

    private void handleOrderConfirmed(OrderConfirmed event) {
        readRepository.findById(event.orderId()).ifPresent(model -> {
            model.setStatus(OrderStatus.CONFIRMED);
            model.setUpdatedAt(event.occurredAt());
            readRepository.save(model);
        });
    }

    private void handleOrderShipped(OrderShipped event) {
        readRepository.findById(event.orderId()).ifPresent(model -> {
            model.setStatus(OrderStatus.SHIPPED);
            model.setTrackingNumber(event.trackingNumber());
            model.setUpdatedAt(event.occurredAt());
            readRepository.save(model);
        });
    }

    private void handleOrderCancelled(OrderCancelled event) {
        readRepository.findById(event.orderId()).ifPresent(model -> {
            model.setStatus(OrderStatus.CANCELLED);
            model.setUpdatedAt(event.occurredAt());
            readRepository.save(model);
        });
    }
}

// Query Handler
@Service
@RequiredArgsConstructor
public class OrderQueryHandler {

    private final OrderReadModelRepository readRepository;

    public OrderReadModel getOrder(String orderId) {
        return readRepository.findById(orderId)
            .orElseThrow(() -> new OrderNotFoundException(orderId));
    }

    public Page<OrderReadModel> getOrdersByCustomer(String customerId, Pageable pageable) {
        return readRepository.findByCustomerId(customerId, pageable);
    }

    public Page<OrderReadModel> getOrdersByStatus(OrderStatus status, Pageable pageable) {
        return readRepository.findByStatus(status, pageable);
    }

    public List<OrderReadModel> searchOrders(OrderSearchCriteria criteria) {
        return readRepository.search(criteria);
    }

    public OrderStatistics getStatistics() {
        return new OrderStatistics(
            readRepository.countByStatus(OrderStatus.PENDING),
            readRepository.countByStatus(OrderStatus.CONFIRMED),
            readRepository.countByStatus(OrderStatus.SHIPPED),
            readRepository.sumTotalAmount()
        );
    }
}
```

---

## 3. Outbox Pattern

### 3.1. Implementation với Kafka

```java
// Outbox Entity
@Entity
@Table(name = "outbox_events")
public class OutboxEvent {

    @Id
    @GeneratedValue(strategy = GenerationType.UUID)
    private String id;

    private String aggregateType;
    private String aggregateId;
    private String eventType;

    @Column(columnDefinition = "TEXT")
    private String payload;

    @Enumerated(EnumType.STRING)
    private OutboxStatus status = OutboxStatus.PENDING;

    private LocalDateTime createdAt = LocalDateTime.now();
    private LocalDateTime publishedAt;
    private int retryCount = 0;
}

// Service with Outbox
@Service
@RequiredArgsConstructor
public class OrderService {

    private final OrderRepository orderRepository;
    private final OutboxRepository outboxRepository;

    @Transactional
    public Order createOrder(CreateOrderCommand command) {
        // Create order
        Order order = new Order(command);
        orderRepository.save(order);

        // Create outbox event (same transaction)
        OutboxEvent outbox = new OutboxEvent();
        outbox.setAggregateType("Order");
        outbox.setAggregateId(order.getId());
        outbox.setEventType("OrderCreated");
        outbox.setPayload(toJson(new OrderCreatedEvent(order)));
        outboxRepository.save(outbox);

        return order;
    }
}

// Outbox Publisher (Polling)
@Component
@RequiredArgsConstructor
public class OutboxPublisher {

    private final OutboxRepository outboxRepository;
    private final KafkaTemplate<String, String> kafkaTemplate;

    @Scheduled(fixedDelay = 1000)
    @Transactional
    public void publishEvents() {
        List<OutboxEvent> events = outboxRepository
            .findByStatusAndRetryCountLessThan(OutboxStatus.PENDING, 3);

        for (OutboxEvent event : events) {
            try {
                String topic = event.getAggregateType().toLowerCase() + "-events";

                kafkaTemplate.send(topic, event.getAggregateId(), event.getPayload())
                    .get(5, TimeUnit.SECONDS);

                event.setStatus(OutboxStatus.PUBLISHED);
                event.setPublishedAt(LocalDateTime.now());

            } catch (Exception e) {
                log.error("Failed to publish event: {}", event.getId(), e);
                event.setRetryCount(event.getRetryCount() + 1);
                if (event.getRetryCount() >= 3) {
                    event.setStatus(OutboxStatus.FAILED);
                }
            }
            outboxRepository.save(event);
        }
    }

    // Cleanup old events
    @Scheduled(cron = "0 0 * * * *")
    @Transactional
    public void cleanup() {
        LocalDateTime threshold = LocalDateTime.now().minusDays(7);
        outboxRepository.deleteByStatusAndPublishedAtBefore(
            OutboxStatus.PUBLISHED, threshold);
    }
}
```

### 3.2. Change Data Capture with Debezium

```yaml
# docker-compose.yml
services:
  debezium:
    image: debezium/connect:2.4
    ports:
      - "8083:8083"
    environment:
      BOOTSTRAP_SERVERS: kafka:9092
      GROUP_ID: debezium-connect
      CONFIG_STORAGE_TOPIC: debezium_configs
      OFFSET_STORAGE_TOPIC: debezium_offsets
      STATUS_STORAGE_TOPIC: debezium_statuses
```

```json
// Register Debezium connector
// POST http://localhost:8083/connectors
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
    "tombstones.on.delete": "false",
    "transforms": "outbox",
    "transforms.outbox.type": "io.debezium.transforms.outbox.EventRouter",
    "transforms.outbox.table.field.event.key": "aggregate_id",
    "transforms.outbox.table.field.event.payload": "payload",
    "transforms.outbox.table.field.event.type": "event_type",
    "transforms.outbox.route.topic.replacement": "${routedByValue}.events"
  }
}
```

---

## 4. Dead Letter Queue Pattern

### 4.1. DLQ Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                    DEAD LETTER QUEUE PATTERN                     │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   ┌───────────┐     ┌───────────┐     ┌───────────┐            │
│   │   Main    │────►│  Consumer │────►│  Process  │            │
│   │   Topic   │     │           │     │           │            │
│   └───────────┘     └─────┬─────┘     └───────────┘            │
│                           │                                     │
│                           │ Error (after retries)               │
│                           ▼                                     │
│                     ┌───────────┐                               │
│                     │   DLQ     │                               │
│                     │  Topic    │                               │
│                     └─────┬─────┘                               │
│                           │                                     │
│              ┌────────────┼────────────┐                       │
│              ▼            ▼            ▼                       │
│         Manual       Auto-Retry    Alert &                     │
│         Review       Scheduled     Monitor                     │
│                                                                  │
│   Flow:                                                          │
│   1. Message fails processing                                   │
│   2. Retry N times with backoff                                 │
│   3. If still failing, send to DLQ                              │
│   4. Alert team for investigation                               │
│   5. Fix issue and replay from DLQ                              │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 4.2. Implementation

```java
// DLQ Configuration
@Configuration
public class DlqConfig {

    @Bean
    public DefaultErrorHandler errorHandler(KafkaTemplate<String, Object> kafkaTemplate) {
        // Dead letter recoverer
        DeadLetterPublishingRecoverer recoverer = new DeadLetterPublishingRecoverer(
            kafkaTemplate,
            (record, ex) -> {
                // Determine DLQ topic
                String originalTopic = record.topic();
                String dlqTopic = originalTopic + ".DLQ";

                log.error("Sending to DLQ: {} -> {}, error: {}",
                    originalTopic, dlqTopic, ex.getMessage());

                return new TopicPartition(dlqTopic, record.partition());
            }
        );

        // Exponential backoff
        ExponentialBackOff backOff = new ExponentialBackOff(1000L, 2.0);
        backOff.setMaxElapsedTime(60000L);

        DefaultErrorHandler handler = new DefaultErrorHandler(recoverer, backOff);

        // Don't retry for specific exceptions
        handler.addNotRetryableExceptions(
            DeserializationException.class,
            ValidationException.class
        );

        return handler;
    }
}

// DLQ Consumer
@Service
@Slf4j
public class DlqConsumer {

    private final AlertService alertService;
    private final DlqRepository dlqRepository;

    @KafkaListener(
        topics = "order-events.DLQ",
        groupId = "dlq-processors"
    )
    public void processDlq(
            ConsumerRecord<String, Object> record,
            @Header(KafkaHeaders.DLT_EXCEPTION_MESSAGE) String errorMessage,
            @Header(KafkaHeaders.DLT_EXCEPTION_STACKTRACE) String stackTrace,
            @Header(KafkaHeaders.DLT_ORIGINAL_TOPIC) String originalTopic,
            @Header(KafkaHeaders.DLT_ORIGINAL_OFFSET) long originalOffset,
            Acknowledgment ack) {

        log.error("DLQ message received: topic={}, offset={}, error={}",
            originalTopic, originalOffset, errorMessage);

        // Store for analysis
        DlqRecord dlqRecord = new DlqRecord();
        dlqRecord.setOriginalTopic(originalTopic);
        dlqRecord.setOriginalOffset(originalOffset);
        dlqRecord.setKey(record.key() != null ? record.key().toString() : null);
        dlqRecord.setValue(record.value().toString());
        dlqRecord.setErrorMessage(errorMessage);
        dlqRecord.setStackTrace(stackTrace);
        dlqRecord.setReceivedAt(LocalDateTime.now());
        dlqRepository.save(dlqRecord);

        // Alert team
        alertService.sendAlert(
            "DLQ Message Received",
            String.format("Topic: %s, Error: %s", originalTopic, errorMessage)
        );

        ack.acknowledge();
    }
}

// DLQ Replay Service
@Service
@RequiredArgsConstructor
public class DlqReplayService {

    private final DlqRepository dlqRepository;
    private final KafkaTemplate<String, Object> kafkaTemplate;

    public void replayMessage(Long dlqRecordId) {
        DlqRecord record = dlqRepository.findById(dlqRecordId)
            .orElseThrow(() -> new NotFoundException("DLQ record not found"));

        // Send back to original topic
        kafkaTemplate.send(
            record.getOriginalTopic(),
            record.getKey(),
            record.getValue()
        );

        record.setReplayedAt(LocalDateTime.now());
        record.setStatus(DlqStatus.REPLAYED);
        dlqRepository.save(record);
    }

    public void replayAll(String topic) {
        List<DlqRecord> records = dlqRepository
            .findByOriginalTopicAndStatus(topic, DlqStatus.PENDING);

        for (DlqRecord record : records) {
            try {
                replayMessage(record.getId());
            } catch (Exception e) {
                log.error("Failed to replay: {}", record.getId(), e);
            }
        }
    }
}
```

---

## 5. Retry Topic Pattern

### 5.1. Multi-Level Retry

```java
@Service
public class RetryTopicConsumer {

    @RetryableTopic(
        attempts = "4",
        backoff = @Backoff(delay = 1000, multiplier = 2, maxDelay = 60000),
        topicSuffixingStrategy = TopicSuffixingStrategy.SUFFIX_WITH_INDEX_VALUE,
        dltStrategy = DltStrategy.FAIL_ON_ERROR,
        autoCreateTopics = "true"
    )
    @KafkaListener(topics = "order-events", groupId = "retry-consumers")
    public void process(OrderEvent event) {
        log.info("Processing: {}", event);

        if (shouldFail(event)) {
            throw new RetryableException("Temporary failure");
        }

        processOrder(event);
    }

    @DltHandler
    public void handleDlt(
            OrderEvent event,
            @Header(KafkaHeaders.RECEIVED_TOPIC) String topic,
            @Header(KafkaHeaders.EXCEPTION_MESSAGE) String error) {

        log.error("DLT received: topic={}, event={}, error={}", topic, event, error);
        // Store, alert, manual intervention
    }
}

// Creates topics:
// order-events (main)
// order-events-retry-0 (1st retry, 1s delay)
// order-events-retry-1 (2nd retry, 2s delay)
// order-events-retry-2 (3rd retry, 4s delay)
// order-events-dlt (dead letter)
```

### 5.2. Custom Retry Destination

```java
@Configuration
public class CustomRetryConfig {

    @Bean
    public RetryTopicConfiguration retryTopicConfiguration(
            KafkaTemplate<String, Object> kafkaTemplate) {

        return RetryTopicConfigurationBuilder
            .newInstance()
            .maxAttempts(4)
            .exponentialBackoff(1000, 2, 30000)
            .retryTopicSuffix("-retry")
            .dltSuffix("-dlt")
            .doNotAutoCreateRetryTopics()
            .setTopicSuffixingStrategy(TopicSuffixingStrategy.SUFFIX_WITH_INDEX_VALUE)
            .dltHandlerMethod("com.example.DltHandler", "handle")
            .create(kafkaTemplate);
    }
}

@Component
public class DltHandler {

    public void handle(Object message, @Header(KafkaHeaders.EXCEPTION_MESSAGE) String error) {
        log.error("DLT: {}, error: {}", message, error);
    }
}
```

---

## 6. Bài tập thực hành

### Bài 1: Event Sourcing
Implement Event Sourcing cho Order aggregate với Kafka.

### Bài 2: CQRS
Tạo separate read model với projector consumer.

### Bài 3: Outbox Pattern
Implement Outbox pattern với polling publisher.

### Bài 4: DLQ và Retry
Setup multi-level retry với DLQ và replay service.

---

## 7. Key Takeaways

| Pattern | Use Case |
|---------|----------|
| Event Sourcing | Audit trail, temporal queries |
| CQRS | Separate read/write optimization |
| Outbox | Reliable event publishing |
| DLQ | Handle failed messages |
| Retry Topics | Automatic retry with backoff |

```
Pattern Selection Guide:
├── Need audit trail? → Event Sourcing
├── Different read/write needs? → CQRS
├── DB + Kafka atomicity? → Outbox
├── Handle processing failures? → DLQ + Retry
└── All of above? → Combine patterns
```

---

## Navigation

- [← Day 13: Kafka Advanced](./day-13-kafka-advanced.md)
- [Day 15: Kafka Operations →](./day-15-kafka-operations.md)
