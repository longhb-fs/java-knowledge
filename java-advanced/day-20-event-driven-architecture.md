# Day 20: Event-Driven Architecture

## Mục tiêu học tập
- Hiểu sâu về Event-Driven Architecture (EDA) principles
- Phân biệt Event types: Domain Events, Integration Events, Commands
- Implement Event-Driven Microservices với Spring
- Xây dựng Event Store và Event Sourcing
- Handle eventual consistency và event ordering

## 1. Event-Driven Architecture Overview

### 1.1 Architecture Patterns

```
Event-Driven Architecture Patterns
══════════════════════════════════

1. Event Notification
   ┌─────────┐     Event      ┌─────────┐
   │ Service │ ──────────────▶│ Service │
   │    A    │  (fire-forget) │    B    │
   └─────────┘                └─────────┘

   - Simple notification
   - No response expected
   - Loose coupling

2. Event-Carried State Transfer
   ┌─────────┐  Event + Data  ┌─────────┐
   │ Service │ ──────────────▶│ Service │
   │    A    │                │    B    │
   └─────────┘                └─────────┘

   - Event contains all needed data
   - B doesn't need to call back A
   - Reduces coupling further

3. Event Sourcing
   ┌─────────┐
   │ Command │
   └────┬────┘
        ▼
   ┌─────────┐    Append     ┌─────────────┐
   │ Service │ ─────────────▶│ Event Store │
   └────┬────┘               │ (immutable) │
        │                    │  Event 1    │
        │                    │  Event 2    │
        ▼                    │  Event 3    │
   ┌─────────┐               └─────────────┘
   │  Read   │ ◀── Projection
   │  Model  │
   └─────────┘

4. CQRS + Event Sourcing
   ┌─────────────────────────────────────────────┐
   │                                             │
   │  Commands                    Queries        │
   │     │                           │           │
   │     ▼                           ▼           │
   │  ┌───────┐                 ┌───────┐       │
   │  │ Write │                 │ Read  │       │
   │  │ Model │                 │ Model │       │
   │  └───┬───┘                 └───┬───┘       │
   │      │                         ▲           │
   │      ▼                         │           │
   │  ┌───────────┐    Projection   │           │
   │  │  Event    │ ────────────────┘           │
   │  │  Store    │                             │
   │  └───────────┘                             │
   │                                             │
   └─────────────────────────────────────────────┘
```

### 1.2 Event Types

```
Event Classification
════════════════════

┌────────────────────┬─────────────────────────────────────────────┐
│    Event Type      │              Description                    │
├────────────────────┼─────────────────────────────────────────────┤
│ Domain Event       │ Something that happened in the domain       │
│                    │ Past tense: OrderCreated, PaymentProcessed  │
│                    │ Internal to bounded context                 │
├────────────────────┼─────────────────────────────────────────────┤
│ Integration Event  │ Event published to other bounded contexts   │
│                    │ Cross-service communication                 │
│                    │ Contains minimal necessary data             │
├────────────────────┼─────────────────────────────────────────────┤
│ Command            │ Request for action (imperative)             │
│                    │ CreateOrder, ProcessPayment                 │
│                    │ May be rejected                             │
└────────────────────┴─────────────────────────────────────────────┘

Event vs Command:
┌─────────────────┬──────────────────────┬──────────────────────┐
│                 │       Event          │       Command        │
├─────────────────┼──────────────────────┼──────────────────────┤
│ Naming          │ Past tense           │ Imperative           │
│                 │ "OrderCreated"       │ "CreateOrder"        │
├─────────────────┼──────────────────────┼──────────────────────┤
│ Semantics       │ Fact (happened)      │ Request (intent)     │
├─────────────────┼──────────────────────┼──────────────────────┤
│ Handlers        │ Multiple allowed     │ Single handler       │
├─────────────────┼──────────────────────┼──────────────────────┤
│ Rejection       │ Cannot be rejected   │ Can be rejected      │
└─────────────────┴──────────────────────┴──────────────────────┘
```

## 2. Domain Events Implementation

### 2.1 Domain Event Base Classes

```java
// Base domain event
public abstract class DomainEvent {
    private final String eventId;
    private final Instant occurredAt;
    private final String aggregateId;
    private final String aggregateType;
    private final int version;

    protected DomainEvent(String aggregateId, String aggregateType, int version) {
        this.eventId = UUID.randomUUID().toString();
        this.occurredAt = Instant.now();
        this.aggregateId = aggregateId;
        this.aggregateType = aggregateType;
        this.version = version;
    }

    public abstract String getEventType();

    // Getters...
}

// Order domain events
public class OrderCreated extends DomainEvent {
    private final String customerId;
    private final List<OrderItem> items;
    private final BigDecimal totalAmount;
    private final Address shippingAddress;

    public OrderCreated(String orderId, int version, String customerId,
            List<OrderItem> items, BigDecimal totalAmount, Address shippingAddress) {
        super(orderId, "Order", version);
        this.customerId = customerId;
        this.items = items;
        this.totalAmount = totalAmount;
        this.shippingAddress = shippingAddress;
    }

    @Override
    public String getEventType() {
        return "OrderCreated";
    }

    // Getters...
}

public class OrderItemAdded extends DomainEvent {
    private final String productId;
    private final int quantity;
    private final BigDecimal price;

    public OrderItemAdded(String orderId, int version, String productId,
            int quantity, BigDecimal price) {
        super(orderId, "Order", version);
        this.productId = productId;
        this.quantity = quantity;
        this.price = price;
    }

    @Override
    public String getEventType() {
        return "OrderItemAdded";
    }
}

public class OrderConfirmed extends DomainEvent {
    private final Instant confirmedAt;
    private final String confirmedBy;

    public OrderConfirmed(String orderId, int version, String confirmedBy) {
        super(orderId, "Order", version);
        this.confirmedAt = Instant.now();
        this.confirmedBy = confirmedBy;
    }

    @Override
    public String getEventType() {
        return "OrderConfirmed";
    }
}

public class OrderShipped extends DomainEvent {
    private final String trackingNumber;
    private final String carrier;
    private final Instant shippedAt;

    public OrderShipped(String orderId, int version, String trackingNumber,
            String carrier) {
        super(orderId, "Order", version);
        this.trackingNumber = trackingNumber;
        this.carrier = carrier;
        this.shippedAt = Instant.now();
    }

    @Override
    public String getEventType() {
        return "OrderShipped";
    }
}
```

### 2.2 Event-Sourced Aggregate

```java
// Base aggregate with event sourcing
public abstract class EventSourcedAggregate {
    private final List<DomainEvent> uncommittedEvents = new ArrayList<>();
    private int version = 0;

    public List<DomainEvent> getUncommittedEvents() {
        return Collections.unmodifiableList(uncommittedEvents);
    }

    public void markEventsAsCommitted() {
        uncommittedEvents.clear();
    }

    public int getVersion() {
        return version;
    }

    protected void applyEvent(DomainEvent event) {
        applyEvent(event, true);
    }

    protected void applyEvent(DomainEvent event, boolean isNew) {
        // Apply the event to change state
        when(event);

        // Track the event if new
        if (isNew) {
            uncommittedEvents.add(event);
        }

        version++;
    }

    // Reconstruct from history
    public void loadFromHistory(List<DomainEvent> history) {
        for (DomainEvent event : history) {
            applyEvent(event, false);
        }
    }

    // Subclass implements event handling
    protected abstract void when(DomainEvent event);
}

// Order aggregate
public class Order extends EventSourcedAggregate {
    private String orderId;
    private String customerId;
    private List<OrderItem> items = new ArrayList<>();
    private BigDecimal totalAmount = BigDecimal.ZERO;
    private OrderStatus status;
    private Address shippingAddress;
    private String trackingNumber;

    // Factory method for creating new order
    public static Order create(String orderId, String customerId,
            List<OrderItem> items, Address shippingAddress) {

        Order order = new Order();

        BigDecimal total = items.stream()
            .map(item -> item.getPrice().multiply(BigDecimal.valueOf(item.getQuantity())))
            .reduce(BigDecimal.ZERO, BigDecimal::add);

        order.applyEvent(new OrderCreated(
            orderId,
            order.getVersion() + 1,
            customerId,
            items,
            total,
            shippingAddress
        ));

        return order;
    }

    // Business methods that generate events
    public void addItem(String productId, int quantity, BigDecimal price) {
        if (status != OrderStatus.DRAFT) {
            throw new IllegalStateException("Cannot add items to confirmed order");
        }

        applyEvent(new OrderItemAdded(
            orderId,
            getVersion() + 1,
            productId,
            quantity,
            price
        ));
    }

    public void confirm(String confirmedBy) {
        if (status != OrderStatus.DRAFT) {
            throw new IllegalStateException("Order already confirmed");
        }

        if (items.isEmpty()) {
            throw new IllegalStateException("Cannot confirm empty order");
        }

        applyEvent(new OrderConfirmed(orderId, getVersion() + 1, confirmedBy));
    }

    public void ship(String trackingNumber, String carrier) {
        if (status != OrderStatus.CONFIRMED) {
            throw new IllegalStateException("Order must be confirmed before shipping");
        }

        applyEvent(new OrderShipped(orderId, getVersion() + 1, trackingNumber, carrier));
    }

    // Event handlers - reconstruct state from events
    @Override
    protected void when(DomainEvent event) {
        switch (event) {
            case OrderCreated e -> {
                this.orderId = e.getAggregateId();
                this.customerId = e.getCustomerId();
                this.items = new ArrayList<>(e.getItems());
                this.totalAmount = e.getTotalAmount();
                this.shippingAddress = e.getShippingAddress();
                this.status = OrderStatus.DRAFT;
            }
            case OrderItemAdded e -> {
                this.items.add(new OrderItem(
                    e.getProductId(),
                    e.getQuantity(),
                    e.getPrice()
                ));
                this.totalAmount = this.totalAmount.add(
                    e.getPrice().multiply(BigDecimal.valueOf(e.getQuantity()))
                );
            }
            case OrderConfirmed e -> {
                this.status = OrderStatus.CONFIRMED;
            }
            case OrderShipped e -> {
                this.status = OrderStatus.SHIPPED;
                this.trackingNumber = e.getTrackingNumber();
            }
            default -> throw new IllegalArgumentException(
                "Unknown event type: " + event.getClass().getSimpleName());
        }
    }

    // Getters...
}
```

### 2.3 Event Store Implementation

```java
// Event store interface
public interface EventStore {
    void append(String streamId, List<DomainEvent> events, int expectedVersion);
    List<DomainEvent> load(String streamId);
    List<DomainEvent> loadFrom(String streamId, int fromVersion);
    void subscribe(String eventType, Consumer<DomainEvent> handler);
}

// JPA-based event store
@Repository
@Slf4j
public class JpaEventStore implements EventStore {

    private final EventStoreRepository repository;
    private final ObjectMapper objectMapper;
    private final Map<String, List<Consumer<DomainEvent>>> subscribers = new ConcurrentHashMap<>();

    public JpaEventStore(EventStoreRepository repository) {
        this.repository = repository;
        this.objectMapper = new ObjectMapper()
            .registerModule(new JavaTimeModule())
            .configure(SerializationFeature.WRITE_DATES_AS_TIMESTAMPS, false);
    }

    @Override
    @Transactional
    public void append(String streamId, List<DomainEvent> events, int expectedVersion) {
        // Optimistic concurrency check
        int currentVersion = repository.findMaxVersionByStreamId(streamId).orElse(0);

        if (currentVersion != expectedVersion) {
            throw new OptimisticLockingException(
                String.format("Expected version %d but found %d for stream %s",
                    expectedVersion, currentVersion, streamId));
        }

        // Append events
        for (DomainEvent event : events) {
            StoredEvent storedEvent = new StoredEvent();
            storedEvent.setEventId(event.getEventId());
            storedEvent.setStreamId(streamId);
            storedEvent.setEventType(event.getEventType());
            storedEvent.setVersion(event.getVersion());
            storedEvent.setOccurredAt(event.getOccurredAt());

            try {
                storedEvent.setPayload(objectMapper.writeValueAsString(event));
            } catch (JsonProcessingException e) {
                throw new RuntimeException("Failed to serialize event", e);
            }

            repository.save(storedEvent);

            // Notify subscribers
            notifySubscribers(event);
        }

        log.info("Appended {} events to stream {}", events.size(), streamId);
    }

    @Override
    public List<DomainEvent> load(String streamId) {
        return loadFrom(streamId, 0);
    }

    @Override
    public List<DomainEvent> loadFrom(String streamId, int fromVersion) {
        List<StoredEvent> storedEvents = repository
            .findByStreamIdAndVersionGreaterThanOrderByVersionAsc(streamId, fromVersion);

        return storedEvents.stream()
            .map(this::deserializeEvent)
            .collect(Collectors.toList());
    }

    @Override
    public void subscribe(String eventType, Consumer<DomainEvent> handler) {
        subscribers.computeIfAbsent(eventType, k -> new CopyOnWriteArrayList<>())
            .add(handler);
    }

    private void notifySubscribers(DomainEvent event) {
        String eventType = event.getEventType();

        List<Consumer<DomainEvent>> handlers = subscribers.get(eventType);
        if (handlers != null) {
            handlers.forEach(handler -> {
                try {
                    handler.accept(event);
                } catch (Exception e) {
                    log.error("Error in event subscriber for {}", eventType, e);
                }
            });
        }

        // Also notify wildcard subscribers
        List<Consumer<DomainEvent>> wildcardHandlers = subscribers.get("*");
        if (wildcardHandlers != null) {
            wildcardHandlers.forEach(handler -> {
                try {
                    handler.accept(event);
                } catch (Exception e) {
                    log.error("Error in wildcard event subscriber", e);
                }
            });
        }
    }

    private DomainEvent deserializeEvent(StoredEvent stored) {
        try {
            Class<?> eventClass = resolveEventClass(stored.getEventType());
            return (DomainEvent) objectMapper.readValue(stored.getPayload(), eventClass);
        } catch (Exception e) {
            throw new RuntimeException("Failed to deserialize event: " + stored.getEventId(), e);
        }
    }

    private Class<?> resolveEventClass(String eventType) {
        return switch (eventType) {
            case "OrderCreated" -> OrderCreated.class;
            case "OrderItemAdded" -> OrderItemAdded.class;
            case "OrderConfirmed" -> OrderConfirmed.class;
            case "OrderShipped" -> OrderShipped.class;
            default -> throw new IllegalArgumentException("Unknown event type: " + eventType);
        };
    }
}

// Stored event entity
@Entity
@Table(name = "event_store", indexes = {
    @Index(name = "idx_stream_version", columnList = "stream_id, version"),
    @Index(name = "idx_event_type", columnList = "event_type"),
    @Index(name = "idx_occurred_at", columnList = "occurred_at")
})
@Data
public class StoredEvent {
    @Id
    private String eventId;

    @Column(name = "stream_id", nullable = false)
    private String streamId;

    @Column(name = "event_type", nullable = false)
    private String eventType;

    @Column(nullable = false)
    private int version;

    @Column(columnDefinition = "TEXT", nullable = false)
    private String payload;

    @Column(name = "occurred_at", nullable = false)
    private Instant occurredAt;
}

// Repository
public interface EventStoreRepository extends JpaRepository<StoredEvent, String> {
    List<StoredEvent> findByStreamIdAndVersionGreaterThanOrderByVersionAsc(
        String streamId, int version);

    Optional<Integer> findMaxVersionByStreamId(String streamId);

    @Query("SELECT MAX(e.version) FROM StoredEvent e WHERE e.streamId = :streamId")
    Optional<Integer> findMaxVersionByStreamId(@Param("streamId") String streamId);
}
```

## 3. Integration Events

### 3.1 Integration Event Publisher

```java
// Integration event (cross-service)
@Data
@Builder
public class IntegrationEvent {
    private String eventId;
    private String eventType;
    private String source;
    private Instant timestamp;
    private Map<String, Object> payload;
    private Map<String, Object> metadata;
}

// Outbox pattern for reliable publishing
@Entity
@Table(name = "outbox_events")
@Data
public class OutboxEvent {
    @Id
    private String id;

    @Column(name = "event_type")
    private String eventType;

    @Column(name = "aggregate_type")
    private String aggregateType;

    @Column(name = "aggregate_id")
    private String aggregateId;

    @Column(columnDefinition = "TEXT")
    private String payload;

    @Column(name = "created_at")
    private Instant createdAt;

    @Column(name = "published_at")
    private Instant publishedAt;

    @Enumerated(EnumType.STRING)
    private OutboxStatus status;
}

public enum OutboxStatus {
    PENDING, PUBLISHED, FAILED
}

// Integration event publisher with outbox
@Service
@Slf4j
@RequiredArgsConstructor
public class IntegrationEventPublisher {

    private final OutboxRepository outboxRepository;
    private final ObjectMapper objectMapper;

    @Transactional
    public void publish(String eventType, String aggregateType,
            String aggregateId, Object payload) {

        OutboxEvent outboxEvent = new OutboxEvent();
        outboxEvent.setId(UUID.randomUUID().toString());
        outboxEvent.setEventType(eventType);
        outboxEvent.setAggregateType(aggregateType);
        outboxEvent.setAggregateId(aggregateId);
        outboxEvent.setCreatedAt(Instant.now());
        outboxEvent.setStatus(OutboxStatus.PENDING);

        try {
            outboxEvent.setPayload(objectMapper.writeValueAsString(payload));
        } catch (JsonProcessingException e) {
            throw new RuntimeException("Failed to serialize event payload", e);
        }

        outboxRepository.save(outboxEvent);

        log.info("Integration event queued: type={}, aggregate={}",
            eventType, aggregateId);
    }

    // Convenience methods for common events
    public void publishOrderCreated(Order order) {
        publish("order.created", "Order", order.getOrderId(),
            Map.of(
                "orderId", order.getOrderId(),
                "customerId", order.getCustomerId(),
                "totalAmount", order.getTotalAmount(),
                "itemCount", order.getItems().size()
            ));
    }

    public void publishOrderShipped(String orderId, String trackingNumber) {
        publish("order.shipped", "Order", orderId,
            Map.of(
                "orderId", orderId,
                "trackingNumber", trackingNumber,
                "shippedAt", Instant.now()
            ));
    }
}

// Outbox processor (runs periodically)
@Service
@Slf4j
@RequiredArgsConstructor
public class OutboxProcessor {

    private final OutboxRepository outboxRepository;
    private final RabbitTemplate rabbitTemplate;
    private final ObjectMapper objectMapper;

    private static final String EXCHANGE = "integration.events";

    @Scheduled(fixedDelay = 1000) // Every second
    @Transactional
    public void processOutbox() {
        List<OutboxEvent> pendingEvents = outboxRepository
            .findByStatusOrderByCreatedAtAsc(OutboxStatus.PENDING);

        for (OutboxEvent event : pendingEvents) {
            try {
                publishToMessageBroker(event);

                event.setStatus(OutboxStatus.PUBLISHED);
                event.setPublishedAt(Instant.now());
                outboxRepository.save(event);

                log.debug("Published outbox event: {}", event.getId());

            } catch (Exception e) {
                log.error("Failed to publish outbox event: {}", event.getId(), e);
                event.setStatus(OutboxStatus.FAILED);
                outboxRepository.save(event);
            }
        }
    }

    private void publishToMessageBroker(OutboxEvent event) throws Exception {
        IntegrationEvent integrationEvent = IntegrationEvent.builder()
            .eventId(event.getId())
            .eventType(event.getEventType())
            .source("order-service")
            .timestamp(event.getCreatedAt())
            .payload(objectMapper.readValue(event.getPayload(), Map.class))
            .metadata(Map.of(
                "aggregateType", event.getAggregateType(),
                "aggregateId", event.getAggregateId()
            ))
            .build();

        String routingKey = event.getEventType().replace(".", "_");

        rabbitTemplate.convertAndSend(EXCHANGE, routingKey, integrationEvent);
    }
}
```

### 3.2 Integration Event Consumer

```java
@Component
@Slf4j
public class IntegrationEventConsumer {

    private final Map<String, IntegrationEventHandler<?>> handlers;

    public IntegrationEventConsumer(List<IntegrationEventHandler<?>> handlerList) {
        this.handlers = handlerList.stream()
            .collect(Collectors.toMap(
                IntegrationEventHandler::getEventType,
                h -> h
            ));
    }

    @RabbitListener(queues = "#{integrationEventQueue.name}")
    public void handleEvent(IntegrationEvent event,
            @Header(AmqpHeaders.MESSAGE_ID) String messageId,
            Channel channel,
            @Header(AmqpHeaders.DELIVERY_TAG) long deliveryTag) throws IOException {

        log.info("Received integration event: type={}, id={}",
            event.getEventType(), event.getEventId());

        try {
            IntegrationEventHandler handler = handlers.get(event.getEventType());

            if (handler == null) {
                log.warn("No handler for event type: {}", event.getEventType());
                channel.basicAck(deliveryTag, false);
                return;
            }

            handler.handle(event);
            channel.basicAck(deliveryTag, false);

        } catch (Exception e) {
            log.error("Error handling integration event: {}", event.getEventId(), e);
            channel.basicNack(deliveryTag, false, true);
        }
    }
}

// Handler interface
public interface IntegrationEventHandler<T> {
    String getEventType();
    void handle(IntegrationEvent event);
}

// Example handlers
@Component
@Slf4j
@RequiredArgsConstructor
public class OrderCreatedHandler implements IntegrationEventHandler<Map<String, Object>> {

    private final InventoryService inventoryService;

    @Override
    public String getEventType() {
        return "order.created";
    }

    @Override
    public void handle(IntegrationEvent event) {
        String orderId = (String) event.getPayload().get("orderId");
        int itemCount = (Integer) event.getPayload().get("itemCount");

        log.info("Processing order created event: orderId={}", orderId);

        // Reserve inventory
        inventoryService.reserveForOrder(orderId);
    }
}

@Component
@Slf4j
@RequiredArgsConstructor
public class OrderShippedHandler implements IntegrationEventHandler<Map<String, Object>> {

    private final NotificationService notificationService;

    @Override
    public String getEventType() {
        return "order.shipped";
    }

    @Override
    public void handle(IntegrationEvent event) {
        String orderId = (String) event.getPayload().get("orderId");
        String trackingNumber = (String) event.getPayload().get("trackingNumber");

        log.info("Processing order shipped event: orderId={}, tracking={}",
            orderId, trackingNumber);

        // Send notification to customer
        notificationService.sendShippingNotification(orderId, trackingNumber);
    }
}
```

## 4. Event Projection

### 4.1 Projection Infrastructure

```java
// Projection interface
public interface Projection {
    String getName();
    void reset();
    void project(DomainEvent event);
    long getLastProcessedPosition();
    void setLastProcessedPosition(long position);
}

// Projection manager
@Service
@Slf4j
@RequiredArgsConstructor
public class ProjectionManager {

    private final List<Projection> projections;
    private final EventStore eventStore;
    private final ProjectionPositionRepository positionRepository;

    @PostConstruct
    public void init() {
        // Subscribe to all events
        eventStore.subscribe("*", this::handleEvent);
    }

    private void handleEvent(DomainEvent event) {
        for (Projection projection : projections) {
            try {
                projection.project(event);
            } catch (Exception e) {
                log.error("Projection {} failed for event {}",
                    projection.getName(), event.getEventId(), e);
            }
        }
    }

    public void rebuildProjection(String projectionName) {
        Projection projection = projections.stream()
            .filter(p -> p.getName().equals(projectionName))
            .findFirst()
            .orElseThrow(() -> new IllegalArgumentException(
                "Projection not found: " + projectionName));

        log.info("Rebuilding projection: {}", projectionName);

        projection.reset();

        // Replay all events (in practice, stream from event store)
        // This is simplified - real implementation would paginate
        List<DomainEvent> allEvents = loadAllEvents();

        for (DomainEvent event : allEvents) {
            projection.project(event);
        }

        log.info("Projection rebuilt: {}", projectionName);
    }

    private List<DomainEvent> loadAllEvents() {
        // Load from event store
        return List.of();
    }
}
```

### 4.2 Read Model Projections

```java
// Order summary read model
@Entity
@Table(name = "order_summaries")
@Data
public class OrderSummary {
    @Id
    private String orderId;

    private String customerId;
    private String customerName;
    private BigDecimal totalAmount;
    private int itemCount;

    @Enumerated(EnumType.STRING)
    private OrderStatus status;

    private String trackingNumber;
    private Instant createdAt;
    private Instant lastUpdatedAt;
}

// Order summary projection
@Component
@Slf4j
@RequiredArgsConstructor
public class OrderSummaryProjection implements Projection {

    private final OrderSummaryRepository repository;
    private final CustomerRepository customerRepository;

    @Override
    public String getName() {
        return "order-summary";
    }

    @Override
    public void reset() {
        repository.deleteAll();
    }

    @Override
    @Transactional
    public void project(DomainEvent event) {
        switch (event) {
            case OrderCreated e -> handleOrderCreated(e);
            case OrderItemAdded e -> handleOrderItemAdded(e);
            case OrderConfirmed e -> handleOrderConfirmed(e);
            case OrderShipped e -> handleOrderShipped(e);
            default -> {} // Ignore other events
        }
    }

    private void handleOrderCreated(OrderCreated event) {
        OrderSummary summary = new OrderSummary();
        summary.setOrderId(event.getAggregateId());
        summary.setCustomerId(event.getCustomerId());
        summary.setTotalAmount(event.getTotalAmount());
        summary.setItemCount(event.getItems().size());
        summary.setStatus(OrderStatus.DRAFT);
        summary.setCreatedAt(event.getOccurredAt());
        summary.setLastUpdatedAt(event.getOccurredAt());

        // Enrich with customer name
        customerRepository.findById(event.getCustomerId())
            .ifPresent(customer -> summary.setCustomerName(customer.getName()));

        repository.save(summary);
    }

    private void handleOrderItemAdded(OrderItemAdded event) {
        repository.findById(event.getAggregateId())
            .ifPresent(summary -> {
                summary.setItemCount(summary.getItemCount() + 1);
                BigDecimal itemTotal = event.getPrice()
                    .multiply(BigDecimal.valueOf(event.getQuantity()));
                summary.setTotalAmount(summary.getTotalAmount().add(itemTotal));
                summary.setLastUpdatedAt(event.getOccurredAt());
                repository.save(summary);
            });
    }

    private void handleOrderConfirmed(OrderConfirmed event) {
        repository.findById(event.getAggregateId())
            .ifPresent(summary -> {
                summary.setStatus(OrderStatus.CONFIRMED);
                summary.setLastUpdatedAt(event.getOccurredAt());
                repository.save(summary);
            });
    }

    private void handleOrderShipped(OrderShipped event) {
        repository.findById(event.getAggregateId())
            .ifPresent(summary -> {
                summary.setStatus(OrderStatus.SHIPPED);
                summary.setTrackingNumber(event.getTrackingNumber());
                summary.setLastUpdatedAt(event.getOccurredAt());
                repository.save(summary);
            });
    }

    @Override
    public long getLastProcessedPosition() {
        return 0; // Simplified
    }

    @Override
    public void setLastProcessedPosition(long position) {
        // Simplified
    }
}

// Query service using read model
@Service
@RequiredArgsConstructor
public class OrderQueryService {

    private final OrderSummaryRepository summaryRepository;

    public Optional<OrderSummary> findById(String orderId) {
        return summaryRepository.findById(orderId);
    }

    public Page<OrderSummary> findByCustomer(String customerId, Pageable pageable) {
        return summaryRepository.findByCustomerIdOrderByCreatedAtDesc(customerId, pageable);
    }

    public Page<OrderSummary> findByStatus(OrderStatus status, Pageable pageable) {
        return summaryRepository.findByStatusOrderByCreatedAtDesc(status, pageable);
    }

    public List<OrderSummary> findPendingShipment() {
        return summaryRepository.findByStatus(OrderStatus.CONFIRMED);
    }

    public OrderStatistics getStatistics(String customerId) {
        return summaryRepository.calculateStatistics(customerId);
    }
}
```

## 5. Eventual Consistency Handling

### 5.1 Saga Pattern

```java
// Saga state
@Entity
@Table(name = "saga_state")
@Data
public class SagaState {
    @Id
    private String sagaId;

    @Column(name = "saga_type")
    private String sagaType;

    @Column(columnDefinition = "TEXT")
    private String state;

    @Enumerated(EnumType.STRING)
    private SagaStatus status;

    private int currentStep;

    @Column(name = "created_at")
    private Instant createdAt;

    @Column(name = "updated_at")
    private Instant updatedAt;

    @Column(name = "compensation_data", columnDefinition = "TEXT")
    private String compensationData;
}

public enum SagaStatus {
    STARTED, PROCESSING, COMPLETED, COMPENSATING, FAILED
}

// Order saga
@Service
@Slf4j
@RequiredArgsConstructor
public class CreateOrderSaga {

    private final SagaStateRepository sagaRepository;
    private final IntegrationEventPublisher eventPublisher;
    private final ObjectMapper objectMapper;

    private static final String SAGA_TYPE = "CreateOrder";

    // Saga steps
    private enum Step {
        RESERVE_INVENTORY,
        PROCESS_PAYMENT,
        CONFIRM_ORDER,
        COMPLETED
    }

    public String start(CreateOrderCommand command) {
        String sagaId = UUID.randomUUID().toString();

        SagaState saga = new SagaState();
        saga.setSagaId(sagaId);
        saga.setSagaType(SAGA_TYPE);
        saga.setStatus(SagaStatus.STARTED);
        saga.setCurrentStep(0);
        saga.setCreatedAt(Instant.now());
        saga.setUpdatedAt(Instant.now());

        try {
            saga.setState(objectMapper.writeValueAsString(command));
            saga.setCompensationData(objectMapper.writeValueAsString(new HashMap<>()));
        } catch (JsonProcessingException e) {
            throw new RuntimeException("Failed to serialize saga state", e);
        }

        sagaRepository.save(saga);

        // Start first step
        executeNextStep(saga);

        return sagaId;
    }

    @Transactional
    public void handleStepCompleted(String sagaId, String stepName, Object result) {
        SagaState saga = sagaRepository.findById(sagaId)
            .orElseThrow(() -> new IllegalArgumentException("Saga not found: " + sagaId));

        log.info("Saga step completed: sagaId={}, step={}", sagaId, stepName);

        // Store compensation data
        storeCompensationData(saga, stepName, result);

        // Move to next step
        saga.setCurrentStep(saga.getCurrentStep() + 1);
        saga.setUpdatedAt(Instant.now());
        sagaRepository.save(saga);

        executeNextStep(saga);
    }

    @Transactional
    public void handleStepFailed(String sagaId, String stepName, String error) {
        SagaState saga = sagaRepository.findById(sagaId)
            .orElseThrow(() -> new IllegalArgumentException("Saga not found: " + sagaId));

        log.error("Saga step failed: sagaId={}, step={}, error={}", sagaId, stepName, error);

        saga.setStatus(SagaStatus.COMPENSATING);
        saga.setUpdatedAt(Instant.now());
        sagaRepository.save(saga);

        // Start compensation
        compensate(saga);
    }

    private void executeNextStep(SagaState saga) {
        Step[] steps = Step.values();

        if (saga.getCurrentStep() >= steps.length - 1) {
            // Saga completed
            saga.setStatus(SagaStatus.COMPLETED);
            saga.setUpdatedAt(Instant.now());
            sagaRepository.save(saga);
            log.info("Saga completed: {}", saga.getSagaId());
            return;
        }

        Step currentStep = steps[saga.getCurrentStep()];
        saga.setStatus(SagaStatus.PROCESSING);
        sagaRepository.save(saga);

        try {
            CreateOrderCommand command = objectMapper.readValue(
                saga.getState(), CreateOrderCommand.class);

            switch (currentStep) {
                case RESERVE_INVENTORY:
                    eventPublisher.publish("inventory.reserve.requested",
                        "Saga", saga.getSagaId(),
                        Map.of(
                            "sagaId", saga.getSagaId(),
                            "orderId", command.getOrderId(),
                            "items", command.getItems()
                        ));
                    break;

                case PROCESS_PAYMENT:
                    eventPublisher.publish("payment.process.requested",
                        "Saga", saga.getSagaId(),
                        Map.of(
                            "sagaId", saga.getSagaId(),
                            "orderId", command.getOrderId(),
                            "amount", command.getTotalAmount(),
                            "paymentMethod", command.getPaymentMethod()
                        ));
                    break;

                case CONFIRM_ORDER:
                    eventPublisher.publish("order.confirm.requested",
                        "Saga", saga.getSagaId(),
                        Map.of(
                            "sagaId", saga.getSagaId(),
                            "orderId", command.getOrderId()
                        ));
                    break;
            }

            log.info("Saga step initiated: sagaId={}, step={}", saga.getSagaId(), currentStep);

        } catch (Exception e) {
            log.error("Failed to execute saga step", e);
            handleStepFailed(saga.getSagaId(), currentStep.name(), e.getMessage());
        }
    }

    private void compensate(SagaState saga) {
        log.info("Starting compensation for saga: {}", saga.getSagaId());

        try {
            Map<String, Object> compensationData = objectMapper.readValue(
                saga.getCompensationData(), Map.class);

            // Compensate in reverse order
            for (int i = saga.getCurrentStep() - 1; i >= 0; i--) {
                Step step = Step.values()[i];
                compensateStep(saga.getSagaId(), step, compensationData);
            }

            saga.setStatus(SagaStatus.FAILED);
            saga.setUpdatedAt(Instant.now());
            sagaRepository.save(saga);

        } catch (Exception e) {
            log.error("Compensation failed for saga: {}", saga.getSagaId(), e);
        }
    }

    private void compensateStep(String sagaId, Step step,
            Map<String, Object> compensationData) {
        switch (step) {
            case RESERVE_INVENTORY:
                String reservationId = (String) compensationData.get("reservationId");
                if (reservationId != null) {
                    eventPublisher.publish("inventory.release.requested",
                        "Saga", sagaId,
                        Map.of("reservationId", reservationId));
                }
                break;

            case PROCESS_PAYMENT:
                String paymentId = (String) compensationData.get("paymentId");
                if (paymentId != null) {
                    eventPublisher.publish("payment.refund.requested",
                        "Saga", sagaId,
                        Map.of("paymentId", paymentId));
                }
                break;
        }
    }

    private void storeCompensationData(SagaState saga, String stepName, Object result) {
        try {
            Map<String, Object> compensationData = objectMapper.readValue(
                saga.getCompensationData(), Map.class);

            if (result instanceof Map) {
                compensationData.putAll((Map<String, Object>) result);
            }

            saga.setCompensationData(objectMapper.writeValueAsString(compensationData));

        } catch (Exception e) {
            log.error("Failed to store compensation data", e);
        }
    }
}
```

### 5.2 Idempotent Event Handler

```java
@Component
@Slf4j
@RequiredArgsConstructor
public class IdempotentEventHandler {

    private final ProcessedEventRepository processedEventRepository;

    @Transactional
    public <T> boolean processOnce(String eventId, Supplier<T> processor) {
        // Check if already processed
        if (processedEventRepository.existsById(eventId)) {
            log.debug("Event already processed: {}", eventId);
            return false;
        }

        try {
            // Process the event
            processor.get();

            // Mark as processed
            ProcessedEvent processed = new ProcessedEvent();
            processed.setEventId(eventId);
            processed.setProcessedAt(Instant.now());
            processedEventRepository.save(processed);

            return true;

        } catch (Exception e) {
            log.error("Failed to process event: {}", eventId, e);
            throw e;
        }
    }
}

// Usage in event handler
@Component
@Slf4j
@RequiredArgsConstructor
public class IdempotentOrderEventHandler {

    private final IdempotentEventHandler idempotentHandler;
    private final OrderService orderService;

    @RabbitListener(queues = "order.events")
    public void handleOrderEvent(IntegrationEvent event,
            @Header(AmqpHeaders.MESSAGE_ID) String messageId,
            Channel channel,
            @Header(AmqpHeaders.DELIVERY_TAG) long deliveryTag) throws IOException {

        boolean processed = idempotentHandler.processOnce(event.getEventId(), () -> {
            // Process the event
            switch (event.getEventType()) {
                case "order.created":
                    return orderService.handleOrderCreated(event.getPayload());
                case "order.shipped":
                    return orderService.handleOrderShipped(event.getPayload());
                default:
                    return null;
            }
        });

        channel.basicAck(deliveryTag, false);

        if (processed) {
            log.info("Event processed: {}", event.getEventId());
        } else {
            log.debug("Event skipped (duplicate): {}", event.getEventId());
        }
    }
}
```

## 6. Complete Example: Order System

### 6.1 Application Service

```java
@Service
@Slf4j
@RequiredArgsConstructor
public class OrderApplicationService {

    private final EventStore eventStore;
    private final IntegrationEventPublisher integrationEventPublisher;
    private final CreateOrderSaga createOrderSaga;

    @Transactional
    public String createOrder(CreateOrderCommand command) {
        // Create the order aggregate
        Order order = Order.create(
            command.getOrderId(),
            command.getCustomerId(),
            command.getItems(),
            command.getShippingAddress()
        );

        // Save events to event store
        eventStore.append(
            order.getOrderId(),
            order.getUncommittedEvents(),
            0 // Expected version for new aggregate
        );

        // Start the saga for cross-service orchestration
        String sagaId = createOrderSaga.start(command);

        log.info("Order created: orderId={}, sagaId={}", order.getOrderId(), sagaId);

        return order.getOrderId();
    }

    @Transactional
    public void confirmOrder(String orderId, String confirmedBy) {
        // Load order from events
        List<DomainEvent> events = eventStore.load(orderId);

        if (events.isEmpty()) {
            throw new IllegalArgumentException("Order not found: " + orderId);
        }

        Order order = new Order();
        order.loadFromHistory(events);

        // Execute business logic
        order.confirm(confirmedBy);

        // Append new events
        eventStore.append(
            orderId,
            order.getUncommittedEvents(),
            order.getVersion() - order.getUncommittedEvents().size()
        );

        // Publish integration event
        integrationEventPublisher.publish("order.confirmed", "Order", orderId,
            Map.of(
                "orderId", orderId,
                "confirmedBy", confirmedBy,
                "confirmedAt", Instant.now()
            ));

        log.info("Order confirmed: {}", orderId);
    }

    @Transactional
    public void shipOrder(String orderId, String trackingNumber, String carrier) {
        List<DomainEvent> events = eventStore.load(orderId);

        Order order = new Order();
        order.loadFromHistory(events);

        order.ship(trackingNumber, carrier);

        eventStore.append(
            orderId,
            order.getUncommittedEvents(),
            order.getVersion() - order.getUncommittedEvents().size()
        );

        integrationEventPublisher.publishOrderShipped(orderId, trackingNumber);

        log.info("Order shipped: orderId={}, tracking={}", orderId, trackingNumber);
    }
}
```

## 7. Hands-on Exercises

### Exercise 1: Event Store Implementation
Build a complete event store with snapshots.

```java
// Requirements:
// 1. Store events with versioning
// 2. Implement snapshot support (every N events)
// 3. Add subscription mechanism
// 4. Implement event replay
```

### Exercise 2: CQRS Read Model
Create a dashboard read model for order analytics.

```java
// Requirements:
// 1. Project events to analytics model
// 2. Track: daily orders, revenue, top customers
// 3. Support time-range queries
// 4. Implement projection rebuild
```

### Exercise 3: Distributed Saga
Implement a complete order fulfillment saga.

```java
// Requirements:
// 1. Steps: Inventory → Payment → Shipping → Notification
// 2. Handle step failures with compensation
// 3. Support timeout handling
// 4. Track saga state persistently
```

### Exercise 4: Event Versioning
Handle event schema evolution.

```java
// Requirements:
// 1. Support multiple event versions
// 2. Implement upcasting (old → new format)
// 3. Handle missing fields gracefully
// 4. Test migration scenarios
```

## 8. Key Takeaways

### Event-Driven Principles
1. **Events are Facts**: Past tense, cannot be rejected
2. **Loose Coupling**: Services communicate via events
3. **Eventual Consistency**: Accept delays in data propagation

### Implementation Patterns
1. **Event Sourcing**: Store events, derive state
2. **CQRS**: Separate read and write models
3. **Outbox Pattern**: Reliable event publishing
4. **Saga Pattern**: Distributed transaction coordination

### Best Practices
1. **Idempotent Handlers**: Handle duplicate events gracefully
2. **Event Versioning**: Plan for schema evolution
3. **Projection Rebuild**: Support replaying from events
4. **Compensation Logic**: Always plan for failures

### Monitoring
1. Track event processing lag
2. Monitor saga completion rates
3. Alert on failed compensations
4. Audit event trails

---

## Navigation

[← Day 19: RabbitMQ Clustering](./day-19-rabbitmq-clustering.md) | [Day 21: Docker Fundamentals →](./day-21-docker-fundamentals.md)

[Back to Overview](./00-overview.md)
