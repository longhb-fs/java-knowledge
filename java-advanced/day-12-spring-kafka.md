# Day 12: Spring Kafka

## Mục tiêu
- Spring Kafka integration
- KafkaTemplate for producing
- @KafkaListener for consuming
- Error handling và retry

---

## 1. Spring Kafka Setup

### 1.1. Dependencies

```xml
<dependency>
    <groupId>org.springframework.kafka</groupId>
    <artifactId>spring-kafka</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.kafka</groupId>
    <artifactId>spring-kafka-test</artifactId>
    <scope>test</scope>
</dependency>
```

### 1.2. Basic Configuration

```yaml
# application.yml
spring:
  kafka:
    bootstrap-servers: localhost:9092

    producer:
      key-serializer: org.apache.kafka.common.serialization.StringSerializer
      value-serializer: org.springframework.kafka.support.serializer.JsonSerializer
      acks: all
      retries: 3
      properties:
        enable.idempotence: true
        max.in.flight.requests.per.connection: 5

    consumer:
      group-id: order-service
      key-deserializer: org.apache.kafka.common.serialization.StringDeserializer
      value-deserializer: org.springframework.kafka.support.serializer.JsonDeserializer
      auto-offset-reset: earliest
      enable-auto-commit: false
      properties:
        spring.json.trusted.packages: com.example.events

    listener:
      ack-mode: MANUAL_IMMEDIATE
      concurrency: 3
```

### 1.3. Java Configuration

```java
@Configuration
@EnableKafka
public class KafkaConfig {

    @Value("${spring.kafka.bootstrap-servers}")
    private String bootstrapServers;

    // Producer Configuration
    @Bean
    public ProducerFactory<String, Object> producerFactory() {
        Map<String, Object> config = new HashMap<>();
        config.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, bootstrapServers);
        config.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, StringSerializer.class);
        config.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, JsonSerializer.class);
        config.put(ProducerConfig.ACKS_CONFIG, "all");
        config.put(ProducerConfig.RETRIES_CONFIG, 3);
        config.put(ProducerConfig.ENABLE_IDEMPOTENCE_CONFIG, true);
        return new DefaultKafkaProducerFactory<>(config);
    }

    @Bean
    public KafkaTemplate<String, Object> kafkaTemplate() {
        return new KafkaTemplate<>(producerFactory());
    }

    // Consumer Configuration
    @Bean
    public ConsumerFactory<String, Object> consumerFactory() {
        Map<String, Object> config = new HashMap<>();
        config.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, bootstrapServers);
        config.put(ConsumerConfig.GROUP_ID_CONFIG, "order-service");
        config.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class);
        config.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, JsonDeserializer.class);
        config.put(ConsumerConfig.AUTO_OFFSET_RESET_CONFIG, "earliest");
        config.put(ConsumerConfig.ENABLE_AUTO_COMMIT_CONFIG, false);
        config.put(JsonDeserializer.TRUSTED_PACKAGES, "com.example.events");
        return new DefaultKafkaConsumerFactory<>(config);
    }

    @Bean
    public ConcurrentKafkaListenerContainerFactory<String, Object> kafkaListenerContainerFactory() {
        ConcurrentKafkaListenerContainerFactory<String, Object> factory =
            new ConcurrentKafkaListenerContainerFactory<>();
        factory.setConsumerFactory(consumerFactory());
        factory.setConcurrency(3);
        factory.getContainerProperties().setAckMode(ContainerProperties.AckMode.MANUAL_IMMEDIATE);
        return factory;
    }
}
```

---

## 2. Producing Messages

### 2.1. KafkaTemplate Basic Usage

```java
@Service
@RequiredArgsConstructor
public class OrderEventProducer {

    private final KafkaTemplate<String, Object> kafkaTemplate;

    // Simple send
    public void sendOrderCreated(OrderCreatedEvent event) {
        kafkaTemplate.send("order-events", event.getOrderId(), event);
    }

    // Send with callback
    public void sendOrderCreatedAsync(OrderCreatedEvent event) {
        CompletableFuture<SendResult<String, Object>> future =
            kafkaTemplate.send("order-events", event.getOrderId(), event);

        future.whenComplete((result, ex) -> {
            if (ex == null) {
                log.info("Sent message to partition {} with offset {}",
                    result.getRecordMetadata().partition(),
                    result.getRecordMetadata().offset());
            } else {
                log.error("Failed to send message", ex);
            }
        });
    }

    // Sync send (blocking)
    public void sendOrderCreatedSync(OrderCreatedEvent event) {
        try {
            SendResult<String, Object> result = kafkaTemplate
                .send("order-events", event.getOrderId(), event)
                .get(10, TimeUnit.SECONDS);

            log.info("Sent to partition {} offset {}",
                result.getRecordMetadata().partition(),
                result.getRecordMetadata().offset());
        } catch (Exception e) {
            log.error("Error sending message", e);
            throw new RuntimeException("Failed to send Kafka message", e);
        }
    }
}
```

### 2.2. ProducerRecord with Headers

```java
@Service
public class OrderEventProducer {

    private final KafkaTemplate<String, Object> kafkaTemplate;

    public void sendWithHeaders(OrderCreatedEvent event) {
        ProducerRecord<String, Object> record = new ProducerRecord<>(
            "order-events",
            null,                    // partition (null = auto)
            System.currentTimeMillis(), // timestamp
            event.getOrderId(),      // key
            event                    // value
        );

        // Add headers
        record.headers()
            .add("event-type", "OrderCreated".getBytes())
            .add("correlation-id", UUID.randomUUID().toString().getBytes())
            .add("source", "order-service".getBytes());

        kafkaTemplate.send(record);
    }

    // Using Message abstraction
    public void sendMessage(OrderCreatedEvent event) {
        Message<OrderCreatedEvent> message = MessageBuilder
            .withPayload(event)
            .setHeader(KafkaHeaders.TOPIC, "order-events")
            .setHeader(KafkaHeaders.KEY, event.getOrderId())
            .setHeader("event-type", "OrderCreated")
            .setHeader("correlation-id", UUID.randomUUID().toString())
            .build();

        kafkaTemplate.send(message);
    }
}
```

### 2.3. Transactional Producer

```java
@Configuration
public class KafkaTransactionConfig {

    @Bean
    public ProducerFactory<String, Object> producerFactory() {
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
        return new KafkaTransactionManager<>(producerFactory());
    }
}

@Service
@RequiredArgsConstructor
public class OrderService {

    private final KafkaTemplate<String, Object> kafkaTemplate;

    @Transactional
    public void processOrder(Order order) {
        // Multiple sends in transaction
        kafkaTemplate.executeInTransaction(operations -> {
            operations.send("order-events", new OrderCreatedEvent(order));
            operations.send("inventory-events", new ReserveInventoryCommand(order));
            operations.send("audit-events", new AuditEvent("Order created", order.getId()));
            return true;
        });
    }
}
```

---

## 3. Consuming Messages

### 3.1. Basic @KafkaListener

```java
@Service
@Slf4j
public class OrderEventConsumer {

    // Simple listener
    @KafkaListener(topics = "order-events", groupId = "order-processors")
    public void listen(OrderCreatedEvent event) {
        log.info("Received order event: {}", event);
        processOrder(event);
    }

    // With manual acknowledgment
    @KafkaListener(topics = "order-events", groupId = "order-processors")
    public void listenWithAck(OrderCreatedEvent event, Acknowledgment ack) {
        try {
            processOrder(event);
            ack.acknowledge();
        } catch (Exception e) {
            log.error("Error processing event", e);
            // Don't acknowledge - will be redelivered
        }
    }

    // Full ConsumerRecord access
    @KafkaListener(topics = "order-events", groupId = "order-processors")
    public void listenWithMetadata(
            ConsumerRecord<String, OrderCreatedEvent> record,
            Acknowledgment ack) {

        log.info("Received from partition {} offset {} key {}",
            record.partition(), record.offset(), record.key());

        // Access headers
        Headers headers = record.headers();
        String correlationId = new String(headers.lastHeader("correlation-id").value());

        processOrder(record.value());
        ack.acknowledge();
    }

    // With @Header annotation
    @KafkaListener(topics = "order-events", groupId = "order-processors")
    public void listenWithHeaders(
            @Payload OrderCreatedEvent event,
            @Header(KafkaHeaders.RECEIVED_PARTITION) int partition,
            @Header(KafkaHeaders.OFFSET) long offset,
            @Header(name = "correlation-id", required = false) String correlationId,
            Acknowledgment ack) {

        log.info("Received event from partition {} offset {} correlationId {}",
            partition, offset, correlationId);

        processOrder(event);
        ack.acknowledge();
    }

    private void processOrder(OrderCreatedEvent event) {
        // Business logic
    }
}
```

### 3.2. Batch Listener

```java
@Configuration
public class KafkaBatchConfig {

    @Bean
    public ConcurrentKafkaListenerContainerFactory<String, Object> batchFactory(
            ConsumerFactory<String, Object> consumerFactory) {

        ConcurrentKafkaListenerContainerFactory<String, Object> factory =
            new ConcurrentKafkaListenerContainerFactory<>();
        factory.setConsumerFactory(consumerFactory);
        factory.setBatchListener(true);  // Enable batch
        factory.setConcurrency(3);
        factory.getContainerProperties().setAckMode(ContainerProperties.AckMode.BATCH);
        return factory;
    }
}

@Service
public class BatchOrderConsumer {

    @KafkaListener(
        topics = "order-events",
        groupId = "batch-processors",
        containerFactory = "batchFactory"
    )
    public void listenBatch(List<OrderCreatedEvent> events, Acknowledgment ack) {
        log.info("Received batch of {} events", events.size());

        for (OrderCreatedEvent event : events) {
            processOrder(event);
        }

        ack.acknowledge();  // Acknowledge entire batch
    }

    // With ConsumerRecords
    @KafkaListener(topics = "order-events", containerFactory = "batchFactory")
    public void listenBatchRecords(
            ConsumerRecords<String, OrderCreatedEvent> records,
            Acknowledgment ack) {

        records.forEach(record -> {
            log.info("Processing: partition={}, offset={}, key={}",
                record.partition(), record.offset(), record.key());
            processOrder(record.value());
        });

        ack.acknowledge();
    }
}
```

### 3.3. Multiple Topics and Partitions

```java
@Service
public class MultiTopicConsumer {

    // Multiple topics
    @KafkaListener(topics = {"order-events", "payment-events"}, groupId = "event-processors")
    public void listenMultipleTopics(
            @Payload Object event,
            @Header(KafkaHeaders.RECEIVED_TOPIC) String topic) {

        switch (topic) {
            case "order-events" -> processOrder((OrderCreatedEvent) event);
            case "payment-events" -> processPayment((PaymentEvent) event);
        }
    }

    // Topic pattern
    @KafkaListener(topicPattern = "order-.*", groupId = "order-processors")
    public void listenTopicPattern(Object event) {
        // Handles order-events, order-updates, order-cancellations, etc.
    }

    // Specific partition assignment
    @KafkaListener(
        groupId = "partition-processors",
        topicPartitions = @TopicPartition(
            topic = "order-events",
            partitions = {"0", "1"}
        )
    )
    public void listenSpecificPartitions(OrderCreatedEvent event) {
        // Only processes partitions 0 and 1
    }

    // Partition with offset
    @KafkaListener(
        groupId = "offset-processors",
        topicPartitions = @TopicPartition(
            topic = "order-events",
            partitionOffsets = {
                @PartitionOffset(partition = "0", initialOffset = "100"),
                @PartitionOffset(partition = "1", initialOffset = "200")
            }
        )
    )
    public void listenFromOffset(OrderCreatedEvent event) {
        // Starts from specific offsets
    }
}
```

---

## 4. Error Handling

### 4.1. Error Handler Configuration

```java
@Configuration
public class KafkaErrorConfig {

    @Bean
    public ConcurrentKafkaListenerContainerFactory<String, Object> kafkaListenerContainerFactory(
            ConsumerFactory<String, Object> consumerFactory,
            KafkaTemplate<String, Object> kafkaTemplate) {

        ConcurrentKafkaListenerContainerFactory<String, Object> factory =
            new ConcurrentKafkaListenerContainerFactory<>();
        factory.setConsumerFactory(consumerFactory);

        // Default error handler
        factory.setCommonErrorHandler(new DefaultErrorHandler(
            new DeadLetterPublishingRecoverer(kafkaTemplate,
                (record, ex) -> new TopicPartition(record.topic() + ".DLT", record.partition())),
            new FixedBackOff(1000L, 3)  // 3 retries, 1 second apart
        ));

        return factory;
    }

    // Custom error handler
    @Bean
    public DefaultErrorHandler customErrorHandler(KafkaTemplate<String, Object> kafkaTemplate) {
        // Dead letter recoverer
        DeadLetterPublishingRecoverer recoverer = new DeadLetterPublishingRecoverer(
            kafkaTemplate,
            (record, exception) -> {
                log.error("Sending to DLT: {} due to {}", record.key(), exception.getMessage());
                return new TopicPartition(record.topic() + ".DLT", record.partition());
            }
        );

        // Backoff strategy
        ExponentialBackOff backOff = new ExponentialBackOff(1000L, 2.0);
        backOff.setMaxElapsedTime(60000L);  // Max 1 minute

        DefaultErrorHandler errorHandler = new DefaultErrorHandler(recoverer, backOff);

        // Don't retry for specific exceptions
        errorHandler.addNotRetryableExceptions(
            ValidationException.class,
            DeserializationException.class
        );

        // Retry for specific exceptions
        errorHandler.addRetryableExceptions(
            TransientDataAccessException.class,
            ServiceUnavailableException.class
        );

        return errorHandler;
    }
}
```

### 4.2. @RetryableTopic (Spring Kafka 2.7+)

```java
@Service
public class RetryableOrderConsumer {

    @RetryableTopic(
        attempts = "4",
        backoff = @Backoff(delay = 1000, multiplier = 2, maxDelay = 10000),
        autoCreateTopics = "true",
        topicSuffixingStrategy = TopicSuffixingStrategy.SUFFIX_WITH_INDEX_VALUE,
        dltStrategy = DltStrategy.FAIL_ON_ERROR,
        include = {ServiceUnavailableException.class, TransientException.class},
        exclude = {ValidationException.class}
    )
    @KafkaListener(topics = "order-events", groupId = "retry-processors")
    public void listen(OrderCreatedEvent event) {
        log.info("Processing order: {}", event.getOrderId());

        if (someCondition) {
            throw new ServiceUnavailableException("Temporary failure");
        }

        processOrder(event);
    }

    @DltHandler
    public void handleDlt(OrderCreatedEvent event, @Header(KafkaHeaders.RECEIVED_TOPIC) String topic) {
        log.error("Received message in DLT: {} from topic: {}", event, topic);
        // Alert, store for manual processing, etc.
        alertService.sendAlert("Order processing failed", event);
    }
}

// Creates topics:
// - order-events (main)
// - order-events-retry-0 (1st retry)
// - order-events-retry-1 (2nd retry)
// - order-events-retry-2 (3rd retry)
// - order-events-dlt (dead letter)
```

### 4.3. Custom Exception Handler

```java
@Component
public class KafkaListenerExceptionHandler implements KafkaListenerErrorHandler {

    @Override
    public Object handleError(Message<?> message, ListenerExecutionFailedException exception) {
        log.error("Error handling message: {}", message.getPayload(), exception);

        // Return null to suppress the exception
        // Or throw to propagate
        if (exception.getCause() instanceof ValidationException) {
            // Log and skip
            return null;
        }

        throw exception;
    }
}

// Usage
@KafkaListener(topics = "order-events", errorHandler = "kafkaListenerExceptionHandler")
public void listen(OrderCreatedEvent event) {
    // ...
}
```

---

## 5. Message Filtering

### 5.1. RecordFilterStrategy

```java
@Configuration
public class KafkaFilterConfig {

    @Bean
    public ConcurrentKafkaListenerContainerFactory<String, Object> filteringFactory(
            ConsumerFactory<String, Object> consumerFactory) {

        ConcurrentKafkaListenerContainerFactory<String, Object> factory =
            new ConcurrentKafkaListenerContainerFactory<>();
        factory.setConsumerFactory(consumerFactory);

        // Filter strategy - return true to discard
        factory.setRecordFilterStrategy(record -> {
            OrderCreatedEvent event = (OrderCreatedEvent) record.value();

            // Skip test orders
            if (event.getOrderId().startsWith("TEST-")) {
                log.debug("Filtering out test order: {}", event.getOrderId());
                return true;  // Discard
            }

            return false;  // Process
        });

        // Acknowledge filtered records
        factory.setAckDiscarded(true);

        return factory;
    }
}

@Service
public class FilteredOrderConsumer {

    @KafkaListener(topics = "order-events", containerFactory = "filteringFactory")
    public void listen(OrderCreatedEvent event) {
        // Only receives non-test orders
        log.info("Processing order: {}", event.getOrderId());
    }
}
```

### 5.2. @KafkaCondition (Custom)

```java
@Service
public class ConditionalConsumer {

    @KafkaListener(topics = "order-events", groupId = "conditional-processors")
    public void listen(
            @Payload OrderCreatedEvent event,
            @Header(name = "order-type", required = false) String orderType) {

        // Skip based on header
        if ("INTERNAL".equals(orderType)) {
            log.debug("Skipping internal order");
            return;
        }

        // Skip based on payload
        if (event.getTotalAmount().compareTo(BigDecimal.ZERO) <= 0) {
            log.warn("Skipping zero-amount order: {}", event.getOrderId());
            return;
        }

        processOrder(event);
    }
}
```

---

## 6. Concurrency and Scaling

### 6.1. Concurrency Configuration

```yaml
spring:
  kafka:
    listener:
      concurrency: 3  # 3 consumer threads per listener
```

```java
@KafkaListener(
    topics = "order-events",
    groupId = "order-processors",
    concurrency = "5"  // Override global setting
)
public void listen(OrderCreatedEvent event) {
    // 5 concurrent consumers for this listener
}
```

### 6.2. Container Lifecycle

```java
@Service
public class KafkaContainerManager {

    @Autowired
    private KafkaListenerEndpointRegistry registry;

    // Stop specific container
    public void pauseOrderProcessing() {
        MessageListenerContainer container = registry.getListenerContainer("order-listener");
        if (container != null) {
            container.pause();
        }
    }

    // Resume
    public void resumeOrderProcessing() {
        MessageListenerContainer container = registry.getListenerContainer("order-listener");
        if (container != null) {
            container.resume();
        }
    }

    // Stop all containers
    public void stopAll() {
        registry.getAllListenerContainers().forEach(MessageListenerContainer::stop);
    }
}

// Listener with ID
@KafkaListener(
    id = "order-listener",
    topics = "order-events",
    groupId = "order-processors"
)
public void listen(OrderCreatedEvent event) {
    // ...
}
```

---

## 7. Testing

### 7.1. EmbeddedKafka Test

```java
@SpringBootTest
@EmbeddedKafka(
    partitions = 1,
    topics = {"order-events"},
    brokerProperties = {
        "listeners=PLAINTEXT://localhost:9092",
        "auto.create.topics.enable=true"
    }
)
class OrderEventTest {

    @Autowired
    private KafkaTemplate<String, Object> kafkaTemplate;

    @Autowired
    private OrderEventConsumer consumer;

    @SpyBean
    private OrderService orderService;

    @Test
    void shouldProcessOrderEvent() throws Exception {
        // Given
        OrderCreatedEvent event = new OrderCreatedEvent("order-123", "user-456");

        // When
        kafkaTemplate.send("order-events", "order-123", event).get();

        // Then
        await().atMost(5, TimeUnit.SECONDS)
            .untilAsserted(() -> {
                verify(orderService).processOrder(any(OrderCreatedEvent.class));
            });
    }
}
```

### 7.2. Consumer Test

```java
@SpringBootTest
@EmbeddedKafka(topics = {"order-events"})
@TestPropertySource(properties = {
    "spring.kafka.bootstrap-servers=${spring.embedded.kafka.brokers}"
})
class OrderConsumerTest {

    @Autowired
    private EmbeddedKafkaBroker embeddedKafka;

    @Autowired
    private KafkaTemplate<String, Object> kafkaTemplate;

    @Autowired
    private TestOrderListener testOrderListener;

    @Test
    void shouldConsumeOrderEvent() throws Exception {
        // Send message
        OrderCreatedEvent event = new OrderCreatedEvent("order-123");
        kafkaTemplate.send("order-events", "order-123", event).get();

        // Wait for consumption
        boolean messageReceived = testOrderListener.getLatch().await(10, TimeUnit.SECONDS);

        assertThat(messageReceived).isTrue();
        assertThat(testOrderListener.getReceivedEvent().getOrderId()).isEqualTo("order-123");
    }

    @TestConfiguration
    static class TestConfig {
        @Bean
        TestOrderListener testOrderListener() {
            return new TestOrderListener();
        }
    }

    static class TestOrderListener {
        private final CountDownLatch latch = new CountDownLatch(1);
        private OrderCreatedEvent receivedEvent;

        @KafkaListener(topics = "order-events", groupId = "test-group")
        public void listen(OrderCreatedEvent event) {
            this.receivedEvent = event;
            latch.countDown();
        }

        public CountDownLatch getLatch() { return latch; }
        public OrderCreatedEvent getReceivedEvent() { return receivedEvent; }
    }
}
```

---

## 8. Best Practices

```java
// 1. Use specific event classes
public record OrderCreatedEvent(
    String eventId,
    String orderId,
    String userId,
    Instant createdAt
) {}

// 2. Include metadata
@KafkaListener(topics = "order-events")
public void listen(
        @Payload OrderCreatedEvent event,
        @Header(KafkaHeaders.RECEIVED_KEY) String key,
        @Header(KafkaHeaders.RECEIVED_TIMESTAMP) long timestamp) {
    // ...
}

// 3. Idempotent processing
@KafkaListener(topics = "order-events")
public void listen(OrderCreatedEvent event, Acknowledgment ack) {
    if (eventRepository.existsByEventId(event.getEventId())) {
        log.info("Event already processed: {}", event.getEventId());
        ack.acknowledge();
        return;
    }

    processOrder(event);
    eventRepository.save(new ProcessedEvent(event.getEventId()));
    ack.acknowledge();
}

// 4. Proper exception handling
@KafkaListener(topics = "order-events")
public void listen(OrderCreatedEvent event, Acknowledgment ack) {
    try {
        processOrder(event);
        ack.acknowledge();
    } catch (TransientException e) {
        throw e;  // Will be retried
    } catch (PermanentException e) {
        log.error("Permanent failure, sending to DLT", e);
        ack.acknowledge();  // Don't retry, let error handler send to DLT
    }
}
```

---

## 9. Bài tập thực hành

### Bài 1: Producer Service
Tạo OrderEventProducer với KafkaTemplate, send events khi order created.

### Bài 2: Consumer Service
Tạo OrderEventConsumer với @KafkaListener, process events và store to DB.

### Bài 3: Retry and DLT
Implement @RetryableTopic với exponential backoff và DLT handler.

### Bài 4: Integration Test
Viết test với EmbeddedKafka verify message flow.

---

## 10. Key Takeaways

| Feature | Description |
|---------|-------------|
| KafkaTemplate | Send messages with callbacks |
| @KafkaListener | Declarative message consumption |
| Acknowledgment | Manual offset commit |
| @RetryableTopic | Built-in retry with DLT |
| RecordFilterStrategy | Filter messages before processing |

```
Spring Kafka Best Practices:
1. Use JSON serialization with trusted packages
2. Enable manual acknowledgment for reliability
3. Configure appropriate concurrency
4. Implement idempotent consumers
5. Use @RetryableTopic for automatic retry
6. Always have DLT handler for failed messages
```

---

## Navigation

- [← Day 11: Kafka Fundamentals](./day-11-kafka-fundamentals.md)
- [Day 13: Kafka Advanced →](./day-13-kafka-advanced.md)
