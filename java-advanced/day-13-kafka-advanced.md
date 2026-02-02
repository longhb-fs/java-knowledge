# Day 13: Kafka Advanced

## Mục tiêu
- Kafka Streams
- Schema Registry
- Exactly-once semantics
- Consumer rebalancing

---

## 1. Kafka Streams

### 1.1. Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                    KAFKA STREAMS                                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Library for building stream processing applications           │
│                                                                  │
│   Input Topic(s) → Kafka Streams App → Output Topic(s)         │
│                                                                  │
│   Features:                                                      │
│   ├── Stateless: filter, map, flatMap                           │
│   ├── Stateful: aggregate, join, windowing                      │
│   ├── Exactly-once processing                                   │
│   ├── Fault-tolerant state stores                               │
│   └── Horizontal scalability                                    │
│                                                                  │
│   ┌──────────┐     ┌────────────────┐     ┌──────────┐        │
│   │  Source  │────►│ Stream Process │────►│   Sink   │        │
│   │  Topic   │     │  (transform)   │     │  Topic   │        │
│   └──────────┘     └────────────────┘     └──────────┘        │
│                           │                                     │
│                    ┌──────┴──────┐                             │
│                    │ State Store │                             │
│                    └─────────────┘                             │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 1.2. Dependencies

```xml
<dependency>
    <groupId>org.apache.kafka</groupId>
    <artifactId>kafka-streams</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.kafka</groupId>
    <artifactId>spring-kafka</artifactId>
</dependency>
```

### 1.3. Basic Stream Processing

```java
@Configuration
@EnableKafkaStreams
public class KafkaStreamsConfig {

    @Value("${spring.kafka.bootstrap-servers}")
    private String bootstrapServers;

    @Bean(name = KafkaStreamsDefaultConfiguration.DEFAULT_STREAMS_CONFIG_BEAN_NAME)
    public KafkaStreamsConfiguration kafkaStreamsConfig() {
        Map<String, Object> props = new HashMap<>();
        props.put(StreamsConfig.APPLICATION_ID_CONFIG, "order-streams-app");
        props.put(StreamsConfig.BOOTSTRAP_SERVERS_CONFIG, bootstrapServers);
        props.put(StreamsConfig.DEFAULT_KEY_SERDE_CLASS_CONFIG, Serdes.String().getClass());
        props.put(StreamsConfig.DEFAULT_VALUE_SERDE_CLASS_CONFIG, Serdes.String().getClass());
        props.put(StreamsConfig.PROCESSING_GUARANTEE_CONFIG, StreamsConfig.EXACTLY_ONCE_V2);
        props.put(StreamsConfig.STATE_DIR_CONFIG, "/tmp/kafka-streams");
        return new KafkaStreamsConfiguration(props);
    }

    @Bean
    public KStream<String, String> orderStream(StreamsBuilder builder) {
        // Read from input topic
        KStream<String, String> stream = builder.stream("order-events");

        // Filter and transform
        KStream<String, String> processedStream = stream
            .filter((key, value) -> value != null && !value.isEmpty())
            .mapValues(value -> {
                // Transform value
                return processOrder(value);
            });

        // Write to output topic
        processedStream.to("processed-orders");

        return stream;
    }

    private String processOrder(String orderJson) {
        // Business logic
        return orderJson;
    }
}
```

### 1.4. Stateless Operations

```java
@Bean
public KStream<String, Order> orderProcessingStream(StreamsBuilder builder) {
    // JSON Serde
    JsonSerde<Order> orderSerde = new JsonSerde<>(Order.class);

    KStream<String, Order> orders = builder.stream(
        "order-events",
        Consumed.with(Serdes.String(), orderSerde)
    );

    // Filter - keep only completed orders
    KStream<String, Order> completedOrders = orders
        .filter((key, order) -> order.getStatus() == OrderStatus.COMPLETED);

    // Map - transform to different type
    KStream<String, OrderSummary> summaries = orders
        .mapValues(order -> new OrderSummary(
            order.getId(),
            order.getTotalAmount(),
            order.getCreatedAt()
        ));

    // FlatMap - one to many
    KStream<String, OrderItem> items = orders
        .flatMapValues(order -> order.getItems());

    // Branch - split stream
    Map<String, KStream<String, Order>> branches = orders
        .split(Named.as("order-"))
        .branch((key, order) -> order.getTotalAmount().compareTo(new BigDecimal("1000")) > 0,
            Branched.as("high-value"))
        .branch((key, order) -> order.getTotalAmount().compareTo(new BigDecimal("100")) > 0,
            Branched.as("medium-value"))
        .defaultBranch(Branched.as("low-value"));

    KStream<String, Order> highValueOrders = branches.get("order-high-value");
    KStream<String, Order> mediumValueOrders = branches.get("order-medium-value");
    KStream<String, Order> lowValueOrders = branches.get("order-low-value");

    // Peek - side effect (logging)
    orders.peek((key, order) -> log.info("Processing order: {}", order.getId()));

    // Output
    completedOrders.to("completed-orders", Produced.with(Serdes.String(), orderSerde));
    highValueOrders.to("high-value-orders", Produced.with(Serdes.String(), orderSerde));

    return orders;
}
```

### 1.5. Stateful Operations

```java
@Bean
public KStream<String, Order> statefulOrderStream(StreamsBuilder builder) {
    JsonSerde<Order> orderSerde = new JsonSerde<>(Order.class);

    KStream<String, Order> orders = builder.stream(
        "order-events",
        Consumed.with(Serdes.String(), orderSerde)
    );

    // Group by customer
    KGroupedStream<String, Order> ordersByCustomer = orders
        .groupBy(
            (key, order) -> order.getCustomerId(),
            Grouped.with(Serdes.String(), orderSerde)
        );

    // Count orders per customer
    KTable<String, Long> orderCounts = ordersByCustomer.count(
        Materialized.<String, Long, KeyValueStore<Bytes, byte[]>>as("order-counts-store")
            .withKeySerde(Serdes.String())
            .withValueSerde(Serdes.Long())
    );

    // Sum order amounts per customer
    KTable<String, Double> totalAmounts = ordersByCustomer.aggregate(
        () -> 0.0,
        (customerId, order, total) -> total + order.getTotalAmount().doubleValue(),
        Materialized.<String, Double, KeyValueStore<Bytes, byte[]>>as("order-totals-store")
            .withKeySerde(Serdes.String())
            .withValueSerde(Serdes.Double())
    );

    // Reduce - keep latest order
    KTable<String, Order> latestOrders = ordersByCustomer.reduce(
        (oldOrder, newOrder) -> newOrder,
        Materialized.<String, Order, KeyValueStore<Bytes, byte[]>>as("latest-orders-store")
            .withKeySerde(Serdes.String())
            .withValueSerde(orderSerde)
    );

    // Output to topics
    orderCounts.toStream().to("customer-order-counts",
        Produced.with(Serdes.String(), Serdes.Long()));

    return orders;
}
```

### 1.6. Windowing

```java
@Bean
public KStream<String, Order> windowedOrderStream(StreamsBuilder builder) {
    JsonSerde<Order> orderSerde = new JsonSerde<>(Order.class);

    KStream<String, Order> orders = builder.stream(
        "order-events",
        Consumed.with(Serdes.String(), orderSerde)
    );

    // Tumbling window - 1 hour fixed windows
    TimeWindowedKStream<String, Order> tumblingWindowed = orders
        .groupBy((key, order) -> order.getCustomerId(),
            Grouped.with(Serdes.String(), orderSerde))
        .windowedBy(TimeWindows.ofSizeWithNoGrace(Duration.ofHours(1)));

    KTable<Windowed<String>, Long> hourlyOrderCounts = tumblingWindowed.count();

    // Hopping window - 1 hour window, advance every 15 minutes
    TimeWindowedKStream<String, Order> hoppingWindowed = orders
        .groupBy((key, order) -> order.getCustomerId(),
            Grouped.with(Serdes.String(), orderSerde))
        .windowedBy(TimeWindows.ofSizeAndGrace(Duration.ofHours(1), Duration.ofMinutes(5))
            .advanceBy(Duration.ofMinutes(15)));

    // Sliding window - exact time difference
    SlidingWindowedKStream<String, Order> slidingWindowed = orders
        .groupBy((key, order) -> order.getCustomerId(),
            Grouped.with(Serdes.String(), orderSerde))
        .windowedBy(SlidingWindows.ofTimeDifferenceWithNoGrace(Duration.ofMinutes(10)));

    // Session window - gap-based
    SessionWindowedKStream<String, Order> sessionWindowed = orders
        .groupBy((key, order) -> order.getCustomerId(),
            Grouped.with(Serdes.String(), orderSerde))
        .windowedBy(SessionWindows.ofInactivityGapWithNoGrace(Duration.ofMinutes(30)));

    KTable<Windowed<String>, Long> sessionOrderCounts = sessionWindowed.count();

    return orders;
}
```

### 1.7. Joins

```java
@Bean
public KStream<String, EnrichedOrder> orderEnrichmentStream(StreamsBuilder builder) {
    JsonSerde<Order> orderSerde = new JsonSerde<>(Order.class);
    JsonSerde<Customer> customerSerde = new JsonSerde<>(Customer.class);
    JsonSerde<EnrichedOrder> enrichedSerde = new JsonSerde<>(EnrichedOrder.class);

    // Order stream
    KStream<String, Order> orders = builder.stream(
        "order-events",
        Consumed.with(Serdes.String(), orderSerde)
    );

    // Customer table (compacted topic)
    KTable<String, Customer> customers = builder.table(
        "customers",
        Consumed.with(Serdes.String(), customerSerde)
    );

    // Stream-Table join
    KStream<String, EnrichedOrder> enrichedOrders = orders
        .selectKey((key, order) -> order.getCustomerId())
        .join(
            customers,
            (order, customer) -> new EnrichedOrder(
                order.getId(),
                order.getTotalAmount(),
                customer.getName(),
                customer.getEmail()
            ),
            Joined.with(Serdes.String(), orderSerde, customerSerde)
        );

    // Left join (customer may not exist)
    KStream<String, EnrichedOrder> leftJoined = orders
        .selectKey((key, order) -> order.getCustomerId())
        .leftJoin(
            customers,
            (order, customer) -> {
                if (customer == null) {
                    return new EnrichedOrder(order.getId(), order.getTotalAmount(),
                        "Unknown", "unknown@example.com");
                }
                return new EnrichedOrder(order.getId(), order.getTotalAmount(),
                    customer.getName(), customer.getEmail());
            }
        );

    // Stream-Stream join (windowed)
    KStream<String, Payment> payments = builder.stream("payment-events",
        Consumed.with(Serdes.String(), new JsonSerde<>(Payment.class)));

    KStream<String, OrderWithPayment> ordersWithPayments = orders
        .selectKey((key, order) -> order.getId())
        .join(
            payments.selectKey((key, payment) -> payment.getOrderId()),
            (order, payment) -> new OrderWithPayment(order, payment),
            JoinWindows.ofTimeDifferenceWithNoGrace(Duration.ofMinutes(5)),
            StreamJoined.with(Serdes.String(), orderSerde, new JsonSerde<>(Payment.class))
        );

    enrichedOrders.to("enriched-orders", Produced.with(Serdes.String(), enrichedSerde));

    return enrichedOrders;
}
```

---

## 2. Schema Registry

### 2.1. Why Schema Registry?

```
┌─────────────────────────────────────────────────────────────────┐
│                    SCHEMA REGISTRY                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Problem: Schema evolution                                      │
│   ├── Producer adds new field → Consumer breaks                 │
│   ├── Field type changes → Deserialization fails                │
│   └── No contract between services                              │
│                                                                  │
│   Solution: Centralized schema management                        │
│                                                                  │
│   ┌──────────┐     ┌─────────────────┐     ┌──────────┐        │
│   │ Producer │────►│ Schema Registry │◄────│ Consumer │        │
│   └────┬─────┘     │  (Avro/JSON)    │     └────┬─────┘        │
│        │           └─────────────────┘          │               │
│        │                                        │               │
│        ▼                                        ▼               │
│   ┌──────────────────────────────────────────────────┐         │
│   │              Kafka Cluster                        │         │
│   └──────────────────────────────────────────────────┘         │
│                                                                  │
│   Features:                                                      │
│   ├── Schema versioning                                         │
│   ├── Compatibility checks (backward, forward, full)            │
│   ├── Schema validation                                         │
│   └── Automatic schema evolution                                │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 2.2. Setup

```yaml
# docker-compose.yml
services:
  schema-registry:
    image: confluentinc/cp-schema-registry:7.5.0
    ports:
      - "8081:8081"
    environment:
      SCHEMA_REGISTRY_HOST_NAME: schema-registry
      SCHEMA_REGISTRY_KAFKASTORE_BOOTSTRAP_SERVERS: kafka:29092
      SCHEMA_REGISTRY_LISTENERS: http://0.0.0.0:8081
    depends_on:
      - kafka
```

```xml
<!-- Maven dependency -->
<dependency>
    <groupId>io.confluent</groupId>
    <artifactId>kafka-avro-serializer</artifactId>
    <version>7.5.0</version>
</dependency>
```

### 2.3. Avro Schema

```avro
// src/main/avro/order.avsc
{
  "namespace": "com.example.events",
  "type": "record",
  "name": "OrderEvent",
  "fields": [
    {"name": "orderId", "type": "string"},
    {"name": "customerId", "type": "string"},
    {"name": "totalAmount", "type": "double"},
    {"name": "status", "type": {
      "type": "enum",
      "name": "OrderStatus",
      "symbols": ["PENDING", "CONFIRMED", "SHIPPED", "DELIVERED", "CANCELLED"]
    }},
    {"name": "createdAt", "type": "long", "logicalType": "timestamp-millis"},
    {"name": "items", "type": {
      "type": "array",
      "items": {
        "type": "record",
        "name": "OrderItem",
        "fields": [
          {"name": "productId", "type": "string"},
          {"name": "quantity", "type": "int"},
          {"name": "price", "type": "double"}
        ]
      }
    }},
    {"name": "shippingAddress", "type": ["null", "string"], "default": null}
  ]
}
```

### 2.4. Configuration

```java
@Configuration
public class AvroKafkaConfig {

    @Value("${spring.kafka.bootstrap-servers}")
    private String bootstrapServers;

    @Value("${spring.kafka.schema-registry-url}")
    private String schemaRegistryUrl;

    @Bean
    public ProducerFactory<String, OrderEvent> avroProducerFactory() {
        Map<String, Object> config = new HashMap<>();
        config.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, bootstrapServers);
        config.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, StringSerializer.class);
        config.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, KafkaAvroSerializer.class);
        config.put("schema.registry.url", schemaRegistryUrl);
        config.put("auto.register.schemas", true);
        return new DefaultKafkaProducerFactory<>(config);
    }

    @Bean
    public KafkaTemplate<String, OrderEvent> avroKafkaTemplate() {
        return new KafkaTemplate<>(avroProducerFactory());
    }

    @Bean
    public ConsumerFactory<String, OrderEvent> avroConsumerFactory() {
        Map<String, Object> config = new HashMap<>();
        config.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, bootstrapServers);
        config.put(ConsumerConfig.GROUP_ID_CONFIG, "avro-consumers");
        config.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class);
        config.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, KafkaAvroDeserializer.class);
        config.put("schema.registry.url", schemaRegistryUrl);
        config.put("specific.avro.reader", true);
        return new DefaultKafkaConsumerFactory<>(config);
    }

    @Bean
    public ConcurrentKafkaListenerContainerFactory<String, OrderEvent> avroKafkaListenerContainerFactory() {
        ConcurrentKafkaListenerContainerFactory<String, OrderEvent> factory =
            new ConcurrentKafkaListenerContainerFactory<>();
        factory.setConsumerFactory(avroConsumerFactory());
        return factory;
    }
}
```

### 2.5. Schema Compatibility

```bash
# Set compatibility level
curl -X PUT -H "Content-Type: application/json" \
  --data '{"compatibility": "BACKWARD"}' \
  http://localhost:8081/config/order-events-value

# Compatibility levels:
# BACKWARD - New schema can read old data
# FORWARD - Old schema can read new data
# FULL - Both backward and forward compatible
# NONE - No compatibility check
```

---

## 3. Exactly-Once Semantics (EOS)

### 3.1. Producer Idempotence

```java
@Configuration
public class IdempotentProducerConfig {

    @Bean
    public ProducerFactory<String, Object> idempotentProducerFactory() {
        Map<String, Object> config = new HashMap<>();
        config.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
        config.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, StringSerializer.class);
        config.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, JsonSerializer.class);

        // Enable idempotence
        config.put(ProducerConfig.ENABLE_IDEMPOTENCE_CONFIG, true);
        config.put(ProducerConfig.ACKS_CONFIG, "all");
        config.put(ProducerConfig.RETRIES_CONFIG, Integer.MAX_VALUE);
        config.put(ProducerConfig.MAX_IN_FLIGHT_REQUESTS_PER_CONNECTION, 5);

        return new DefaultKafkaProducerFactory<>(config);
    }
}
```

### 3.2. Transactional Processing

```java
@Configuration
public class TransactionalKafkaConfig {

    @Bean
    public ProducerFactory<String, Object> transactionalProducerFactory() {
        Map<String, Object> config = new HashMap<>();
        config.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
        config.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, StringSerializer.class);
        config.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, JsonSerializer.class);
        config.put(ProducerConfig.ENABLE_IDEMPOTENCE_CONFIG, true);
        config.put(ProducerConfig.TRANSACTIONAL_ID_CONFIG, "order-tx-");

        DefaultKafkaProducerFactory<String, Object> factory =
            new DefaultKafkaProducerFactory<>(config);
        factory.setTransactionIdPrefix("order-tx-");
        return factory;
    }

    @Bean
    public KafkaTransactionManager<String, Object> kafkaTransactionManager() {
        return new KafkaTransactionManager<>(transactionalProducerFactory());
    }

    @Bean
    public ConsumerFactory<String, Object> transactionalConsumerFactory() {
        Map<String, Object> config = new HashMap<>();
        config.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
        config.put(ConsumerConfig.GROUP_ID_CONFIG, "transactional-consumers");
        config.put(ConsumerConfig.ISOLATION_LEVEL_CONFIG, "read_committed");
        config.put(ConsumerConfig.ENABLE_AUTO_COMMIT_CONFIG, false);
        return new DefaultKafkaConsumerFactory<>(config);
    }
}

@Service
@RequiredArgsConstructor
public class TransactionalOrderService {

    private final KafkaTemplate<String, Object> kafkaTemplate;
    private final OrderRepository orderRepository;

    @Transactional("kafkaTransactionManager")
    public void processOrder(Order order) {
        // Save to database
        orderRepository.save(order);

        // Send multiple messages atomically
        kafkaTemplate.send("order-events", new OrderCreatedEvent(order));
        kafkaTemplate.send("inventory-events", new ReserveInventoryEvent(order));
        kafkaTemplate.send("audit-events", new AuditEvent("Order created"));

        // All succeed or all fail
    }
}
```

### 3.3. Consume-Transform-Produce Pattern

```java
@Service
public class ExactlyOnceProcessor {

    @KafkaListener(
        topics = "input-events",
        groupId = "eos-processors",
        containerFactory = "eosKafkaListenerContainerFactory"
    )
    @Transactional("kafkaTransactionManager")
    public void process(
            ConsumerRecord<String, InputEvent> record,
            @Header(KafkaHeaders.OFFSET) long offset,
            Acknowledgment ack) {

        // Transform
        OutputEvent output = transform(record.value());

        // Produce (within same transaction)
        kafkaTemplate.send("output-events", record.key(), output);

        // Commit consumer offset as part of transaction
        ack.acknowledge();
    }

    @Bean
    public ConcurrentKafkaListenerContainerFactory<String, Object> eosKafkaListenerContainerFactory() {
        ConcurrentKafkaListenerContainerFactory<String, Object> factory =
            new ConcurrentKafkaListenerContainerFactory<>();
        factory.setConsumerFactory(transactionalConsumerFactory());
        factory.getContainerProperties().setTransactionManager(kafkaTransactionManager());
        factory.getContainerProperties().setAckMode(ContainerProperties.AckMode.RECORD);
        return factory;
    }
}
```

---

## 4. Consumer Rebalancing

### 4.1. Rebalance Listener

```java
@Component
public class OrderRebalanceListener implements ConsumerRebalanceListener {

    private final Map<TopicPartition, OffsetAndMetadata> currentOffsets = new HashMap<>();

    @Override
    public void onPartitionsRevoked(Collection<TopicPartition> partitions) {
        log.info("Partitions revoked: {}", partitions);

        // Commit offsets before rebalance
        for (TopicPartition partition : partitions) {
            OffsetAndMetadata offset = currentOffsets.get(partition);
            if (offset != null) {
                log.info("Committing offset {} for partition {}", offset.offset(), partition);
            }
        }

        // Clean up resources
        cleanupPartitionState(partitions);
    }

    @Override
    public void onPartitionsAssigned(Collection<TopicPartition> partitions) {
        log.info("Partitions assigned: {}", partitions);

        // Initialize state for new partitions
        for (TopicPartition partition : partitions) {
            initializePartitionState(partition);
        }
    }

    public void trackOffset(TopicPartition partition, long offset) {
        currentOffsets.put(partition, new OffsetAndMetadata(offset + 1));
    }

    private void cleanupPartitionState(Collection<TopicPartition> partitions) {
        partitions.forEach(currentOffsets::remove);
    }

    private void initializePartitionState(TopicPartition partition) {
        // Load state from database or external store
    }
}
```

### 4.2. Spring Kafka Rebalance Listener

```java
@Configuration
public class RebalanceConfig {

    @Bean
    public ConcurrentKafkaListenerContainerFactory<String, Object> kafkaListenerContainerFactory(
            ConsumerFactory<String, Object> consumerFactory) {

        ConcurrentKafkaListenerContainerFactory<String, Object> factory =
            new ConcurrentKafkaListenerContainerFactory<>();
        factory.setConsumerFactory(consumerFactory);

        factory.getContainerProperties().setConsumerRebalanceListener(
            new ConsumerAwareRebalanceListener() {
                @Override
                public void onPartitionsRevokedBeforeCommit(
                        Consumer<?, ?> consumer, Collection<TopicPartition> partitions) {
                    log.info("Partitions revoked before commit: {}", partitions);
                    // Save state before losing partition
                }

                @Override
                public void onPartitionsRevokedAfterCommit(
                        Consumer<?, ?> consumer, Collection<TopicPartition> partitions) {
                    log.info("Partitions revoked after commit: {}", partitions);
                }

                @Override
                public void onPartitionsAssigned(
                        Consumer<?, ?> consumer, Collection<TopicPartition> partitions) {
                    log.info("Partitions assigned: {}", partitions);
                    // Restore state for new partitions
                }

                @Override
                public void onPartitionsLost(
                        Consumer<?, ?> consumer, Collection<TopicPartition> partitions) {
                    log.warn("Partitions lost: {}", partitions);
                    // Handle unexpected loss (crash scenario)
                }
            }
        );

        return factory;
    }
}
```

### 4.3. Cooperative Rebalancing

```yaml
spring:
  kafka:
    consumer:
      properties:
        partition.assignment.strategy: org.apache.kafka.clients.consumer.CooperativeStickyAssignor
```

```java
// Benefits of cooperative rebalancing:
// 1. Incremental rebalance - only affected partitions stop
// 2. Reduced stop-the-world events
// 3. Better consumer availability

@Configuration
public class CooperativeRebalanceConfig {

    @Bean
    public ConsumerFactory<String, Object> consumerFactory() {
        Map<String, Object> config = new HashMap<>();
        config.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
        config.put(ConsumerConfig.GROUP_ID_CONFIG, "cooperative-group");

        // Use cooperative sticky assignor
        config.put(ConsumerConfig.PARTITION_ASSIGNMENT_STRATEGY_CONFIG,
            CooperativeStickyAssignor.class.getName());

        return new DefaultKafkaConsumerFactory<>(config);
    }
}
```

---

## 5. Interactive Queries

### 5.1. Querying State Stores

```java
@Service
@RequiredArgsConstructor
public class OrderQueryService {

    private final StreamsBuilderFactoryBean factoryBean;

    public Long getOrderCount(String customerId) {
        KafkaStreams streams = factoryBean.getKafkaStreams();

        ReadOnlyKeyValueStore<String, Long> store = streams.store(
            StoreQueryParameters.fromNameAndType(
                "order-counts-store",
                QueryableStoreTypes.keyValueStore()
            )
        );

        return store.get(customerId);
    }

    public Map<String, Long> getAllOrderCounts() {
        KafkaStreams streams = factoryBean.getKafkaStreams();

        ReadOnlyKeyValueStore<String, Long> store = streams.store(
            StoreQueryParameters.fromNameAndType(
                "order-counts-store",
                QueryableStoreTypes.keyValueStore()
            )
        );

        Map<String, Long> results = new HashMap<>();
        try (KeyValueIterator<String, Long> iterator = store.all()) {
            while (iterator.hasNext()) {
                KeyValue<String, Long> entry = iterator.next();
                results.put(entry.key, entry.value);
            }
        }
        return results;
    }

    public List<Order> getOrdersInWindow(String customerId, Instant from, Instant to) {
        KafkaStreams streams = factoryBean.getKafkaStreams();

        ReadOnlyWindowStore<String, Order> store = streams.store(
            StoreQueryParameters.fromNameAndType(
                "windowed-orders-store",
                QueryableStoreTypes.windowStore()
            )
        );

        List<Order> results = new ArrayList<>();
        try (WindowStoreIterator<Order> iterator =
                store.fetch(customerId, from, to)) {
            while (iterator.hasNext()) {
                results.add(iterator.next().value);
            }
        }
        return results;
    }
}
```

### 5.2. REST API for Queries

```java
@RestController
@RequestMapping("/api/orders")
@RequiredArgsConstructor
public class OrderQueryController {

    private final OrderQueryService queryService;

    @GetMapping("/count/{customerId}")
    public ResponseEntity<Long> getOrderCount(@PathVariable String customerId) {
        Long count = queryService.getOrderCount(customerId);
        if (count == null) {
            return ResponseEntity.notFound().build();
        }
        return ResponseEntity.ok(count);
    }

    @GetMapping("/counts")
    public ResponseEntity<Map<String, Long>> getAllCounts() {
        return ResponseEntity.ok(queryService.getAllOrderCounts());
    }

    @GetMapping("/window/{customerId}")
    public ResponseEntity<List<Order>> getOrdersInWindow(
            @PathVariable String customerId,
            @RequestParam Instant from,
            @RequestParam Instant to) {
        return ResponseEntity.ok(queryService.getOrdersInWindow(customerId, from, to));
    }
}
```

---

## 6. Bài tập thực hành

### Bài 1: Kafka Streams App
Tạo Kafka Streams app để aggregate orders theo customer.

### Bài 2: Schema Registry
Setup Avro serialization với Schema Registry.

### Bài 3: EOS Processing
Implement exactly-once consume-transform-produce pattern.

### Bài 4: Interactive Queries
Expose state store queries qua REST API.

---

## 7. Key Takeaways

| Feature | Description |
|---------|-------------|
| Kafka Streams | Stream processing library |
| Schema Registry | Centralized schema management |
| Exactly-Once | Transactional producer + consumer |
| Rebalancing | Partition reassignment |
| State Stores | Local queryable storage |

```
Advanced Kafka Best Practices:
1. Use Schema Registry for schema evolution
2. Enable idempotence for reliable delivery
3. Use transactions for atomic operations
4. Implement proper rebalance handling
5. Monitor state store size and performance
```

---

## Navigation

- [← Day 12: Spring Kafka](./day-12-spring-kafka.md)
- [Day 14: Kafka Patterns →](./day-14-kafka-patterns.md)
