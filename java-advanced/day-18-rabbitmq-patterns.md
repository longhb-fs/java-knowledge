# Day 18: RabbitMQ Patterns

## Mục tiêu học tập
- Implement các messaging patterns phổ biến với RabbitMQ
- Xây dựng Work Queues cho distributed task processing
- Triển khai Publish/Subscribe pattern
- Implement Request/Reply và Routing patterns
- Handle message ordering và deduplication

## 1. Work Queue Pattern

### 1.1 Pattern Overview

```
Work Queue Pattern (Competing Consumers)
════════════════════════════════════════

                                    ┌──────────┐
                                ┌──▶│ Worker 1 │
                                │   └──────────┘
┌──────────┐    ┌─────────┐    │   ┌──────────┐
│ Producer │───▶│  Queue  │────┼──▶│ Worker 2 │
└──────────┘    └─────────┘    │   └──────────┘
                                │   ┌──────────┐
                                └──▶│ Worker 3 │
                                    └──────────┘

Benefits:
- Load distribution across workers
- Scalability (add more workers)
- Fault tolerance (failed tasks requeued)

Round-Robin Distribution:
  Message 1 → Worker 1
  Message 2 → Worker 2
  Message 3 → Worker 3
  Message 4 → Worker 1  (cycles back)
```

### 1.2 Work Queue Implementation

```java
// Task model
@Data
@Builder
public class Task {
    private String taskId;
    private String type;
    private Map<String, Object> payload;
    private int priority;
    private Instant createdAt;
    private int retryCount;
}

// Task result
@Data
@Builder
public class TaskResult {
    private String taskId;
    private TaskStatus status;
    private Object result;
    private String errorMessage;
    private Duration processingTime;
}

public enum TaskStatus {
    COMPLETED, FAILED, RETRY
}

// Configuration
@Configuration
public class WorkQueueConfiguration {

    public static final String TASK_QUEUE = "task.queue";
    public static final String TASK_DLQ = "task.dlq";

    @Bean
    public Queue taskQueue() {
        return QueueBuilder
            .durable(TASK_QUEUE)
            .withArgument("x-dead-letter-exchange", "")
            .withArgument("x-dead-letter-routing-key", TASK_DLQ)
            .withArgument("x-max-priority", 10) // Priority queue
            .build();
    }

    @Bean
    public Queue taskDlq() {
        return QueueBuilder
            .durable(TASK_DLQ)
            .build();
    }
}

// Task producer
@Service
@Slf4j
@RequiredArgsConstructor
public class TaskProducer {

    private final RabbitTemplate rabbitTemplate;

    public void submitTask(Task task) {
        task.setTaskId(UUID.randomUUID().toString());
        task.setCreatedAt(Instant.now());

        rabbitTemplate.convertAndSend(
            WorkQueueConfiguration.TASK_QUEUE,
            task,
            message -> {
                message.getMessageProperties().setPriority(task.getPriority());
                message.getMessageProperties().setMessageId(task.getTaskId());
                return message;
            }
        );

        log.info("Task submitted: id={}, type={}, priority={}",
            task.getTaskId(), task.getType(), task.getPriority());
    }

    public void submitBatch(List<Task> tasks) {
        tasks.forEach(this::submitTask);
        log.info("Batch of {} tasks submitted", tasks.size());
    }
}

// Task worker
@Component
@Slf4j
public class TaskWorker {

    private final Map<String, TaskHandler> handlers;

    public TaskWorker(List<TaskHandler> taskHandlers) {
        this.handlers = taskHandlers.stream()
            .collect(Collectors.toMap(TaskHandler::getTaskType, h -> h));
    }

    @RabbitListener(
        queues = WorkQueueConfiguration.TASK_QUEUE,
        concurrency = "3-10" // Min 3, max 10 consumers
    )
    public void processTask(
            Task task,
            Channel channel,
            @Header(AmqpHeaders.DELIVERY_TAG) long deliveryTag)
            throws IOException {

        Instant startTime = Instant.now();
        String taskId = task.getTaskId();

        log.info("Processing task: id={}, type={}", taskId, task.getType());

        try {
            TaskHandler handler = handlers.get(task.getType());

            if (handler == null) {
                log.error("No handler for task type: {}", task.getType());
                channel.basicNack(deliveryTag, false, false); // To DLQ
                return;
            }

            TaskResult result = handler.handle(task);

            if (result.getStatus() == TaskStatus.COMPLETED) {
                channel.basicAck(deliveryTag, false);
                log.info("Task completed: id={}, duration={}ms",
                    taskId, Duration.between(startTime, Instant.now()).toMillis());

            } else if (result.getStatus() == TaskStatus.RETRY) {
                handleRetry(task, channel, deliveryTag);

            } else {
                channel.basicNack(deliveryTag, false, false); // To DLQ
                log.error("Task failed: id={}, error={}", taskId, result.getErrorMessage());
            }

        } catch (Exception e) {
            log.error("Exception processing task: id={}", taskId, e);
            handleRetry(task, channel, deliveryTag);
        }
    }

    private void handleRetry(Task task, Channel channel, long deliveryTag)
            throws IOException {
        int maxRetries = 3;

        if (task.getRetryCount() >= maxRetries) {
            channel.basicNack(deliveryTag, false, false); // To DLQ
            log.warn("Max retries exceeded for task: {}", task.getTaskId());
        } else {
            channel.basicNack(deliveryTag, false, true); // Requeue
            log.info("Task requeued for retry: id={}, attempt={}",
                task.getTaskId(), task.getRetryCount() + 1);
        }
    }
}

// Task handler interface
public interface TaskHandler {
    String getTaskType();
    TaskResult handle(Task task);
}

// Example handlers
@Component
public class EmailTaskHandler implements TaskHandler {

    @Override
    public String getTaskType() {
        return "EMAIL";
    }

    @Override
    public TaskResult handle(Task task) {
        // Send email logic
        String recipient = (String) task.getPayload().get("recipient");
        String subject = (String) task.getPayload().get("subject");

        // Simulate email sending
        try {
            Thread.sleep(100);
            return TaskResult.builder()
                .taskId(task.getTaskId())
                .status(TaskStatus.COMPLETED)
                .result(Map.of("sent", true, "recipient", recipient))
                .build();
        } catch (Exception e) {
            return TaskResult.builder()
                .taskId(task.getTaskId())
                .status(TaskStatus.RETRY)
                .errorMessage(e.getMessage())
                .build();
        }
    }
}

@Component
public class ReportTaskHandler implements TaskHandler {

    @Override
    public String getTaskType() {
        return "REPORT";
    }

    @Override
    public TaskResult handle(Task task) {
        String reportType = (String) task.getPayload().get("reportType");

        // Generate report logic
        return TaskResult.builder()
            .taskId(task.getTaskId())
            .status(TaskStatus.COMPLETED)
            .result(Map.of("reportId", UUID.randomUUID().toString()))
            .build();
    }
}
```

### 1.3 Fair Dispatch with Prefetch

```java
@Configuration
public class FairDispatchConfiguration {

    @Bean
    public SimpleRabbitListenerContainerFactory fairDispatchFactory(
            ConnectionFactory connectionFactory,
            MessageConverter messageConverter) {

        SimpleRabbitListenerContainerFactory factory =
            new SimpleRabbitListenerContainerFactory();

        factory.setConnectionFactory(connectionFactory);
        factory.setMessageConverter(messageConverter);
        factory.setAcknowledgeMode(AcknowledgeMode.MANUAL);

        // Prefetch = 1 ensures fair dispatch
        // Worker won't get new message until current one is processed
        factory.setPrefetchCount(1);

        // Dynamic concurrency based on queue depth
        factory.setConcurrentConsumers(3);
        factory.setMaxConcurrentConsumers(10);

        return factory;
    }
}

/*
 * Fair Dispatch Visualization:
 *
 * With prefetch=1:
 * ┌─────────────────────────────────────────────────────┐
 * │ Queue: [M1, M2, M3, M4, M5, M6]                     │
 * │                                                     │
 * │ Worker 1: [M1] processing...                        │
 * │ Worker 2: [M2] processing...                        │
 * │ Worker 3: idle (waiting for M1 or M2 to complete)  │
 * │                                                     │
 * │ M1 completes → Worker 1 gets M3                     │
 * │ M2 completes → Worker 2 gets M4                     │
 * └─────────────────────────────────────────────────────┘
 *
 * With prefetch=10 (unfair):
 * ┌─────────────────────────────────────────────────────┐
 * │ Worker 1: [M1, M2, M3, M4, M5] prefetched          │
 * │ Worker 2: [M6, M7, M8, M9, M10] prefetched         │
 * │ Worker 3: idle (no messages left)                   │
 * │                                                     │
 * │ Even if Worker 3 is faster, it has no work!        │
 * └─────────────────────────────────────────────────────┘
 */
```

## 2. Publish/Subscribe Pattern

### 2.1 Pattern Overview

```
Publish/Subscribe Pattern (Fanout)
══════════════════════════════════

                                    ┌─────────┐    ┌──────────────┐
                              ┌────▶│ Queue 1 │───▶│ Subscriber 1 │
                              │     └─────────┘    └──────────────┘
┌───────────┐    ┌─────────┐  │     ┌─────────┐    ┌──────────────┐
│ Publisher │───▶│ Fanout  │──┼────▶│ Queue 2 │───▶│ Subscriber 2 │
└───────────┘    │Exchange │  │     └─────────┘    └──────────────┘
                 └─────────┘  │     ┌─────────┐    ┌──────────────┐
                              └────▶│ Queue 3 │───▶│ Subscriber 3 │
                                    └─────────┘    └──────────────┘

All subscribers receive a copy of every message
```

### 2.2 Event Broadcasting System

```java
// Domain events
@Data
@Builder
public class DomainEvent {
    private String eventId;
    private String eventType;
    private String aggregateType;
    private String aggregateId;
    private Object payload;
    private Instant occurredAt;
    private Map<String, Object> metadata;
}

// Configuration
@Configuration
public class EventBroadcastConfiguration {

    public static final String EVENTS_EXCHANGE = "domain.events.fanout";

    @Bean
    public FanoutExchange domainEventsExchange() {
        return ExchangeBuilder
            .fanoutExchange(EVENTS_EXCHANGE)
            .durable(true)
            .build();
    }

    // Each service creates its own queue and binding
    @Bean
    public Queue orderServiceEventsQueue() {
        return QueueBuilder
            .durable("order-service.domain-events")
            .build();
    }

    @Bean
    public Queue inventoryServiceEventsQueue() {
        return QueueBuilder
            .durable("inventory-service.domain-events")
            .build();
    }

    @Bean
    public Queue analyticsServiceEventsQueue() {
        return QueueBuilder
            .durable("analytics-service.domain-events")
            .build();
    }

    @Bean
    public Binding orderServiceBinding(
            Queue orderServiceEventsQueue,
            FanoutExchange domainEventsExchange) {
        return BindingBuilder
            .bind(orderServiceEventsQueue)
            .to(domainEventsExchange);
    }

    @Bean
    public Binding inventoryServiceBinding(
            Queue inventoryServiceEventsQueue,
            FanoutExchange domainEventsExchange) {
        return BindingBuilder
            .bind(inventoryServiceEventsQueue)
            .to(domainEventsExchange);
    }

    @Bean
    public Binding analyticsServiceBinding(
            Queue analyticsServiceEventsQueue,
            FanoutExchange domainEventsExchange) {
        return BindingBuilder
            .bind(analyticsServiceEventsQueue)
            .to(domainEventsExchange);
    }
}

// Event publisher
@Service
@Slf4j
@RequiredArgsConstructor
public class DomainEventPublisher {

    private final RabbitTemplate rabbitTemplate;
    private final ApplicationEventPublisher localEventPublisher;

    @Transactional
    public void publish(DomainEvent event) {
        // Set metadata
        event.setEventId(UUID.randomUUID().toString());
        event.setOccurredAt(Instant.now());

        // Publish to RabbitMQ (broadcast to all subscribers)
        rabbitTemplate.convertAndSend(
            EventBroadcastConfiguration.EVENTS_EXCHANGE,
            "", // Routing key ignored for fanout
            event
        );

        // Also publish locally for same-service listeners
        localEventPublisher.publishEvent(event);

        log.info("Domain event published: type={}, aggregateId={}",
            event.getEventType(), event.getAggregateId());
    }

    public void publishOrderCreated(Order order) {
        publish(DomainEvent.builder()
            .eventType("ORDER_CREATED")
            .aggregateType("Order")
            .aggregateId(order.getOrderId())
            .payload(order)
            .metadata(Map.of(
                "customerId", order.getCustomerId(),
                "totalAmount", order.getTotalAmount()
            ))
            .build());
    }
}

// Subscribers in different services
@Component
@Slf4j
public class OrderServiceEventHandler {

    @RabbitListener(queues = "order-service.domain-events")
    public void handleEvent(DomainEvent event) {
        log.info("Order service received event: {}", event.getEventType());

        switch (event.getEventType()) {
            case "PAYMENT_COMPLETED":
                handlePaymentCompleted(event);
                break;
            case "INVENTORY_RESERVED":
                handleInventoryReserved(event);
                break;
            default:
                log.debug("Event type not handled: {}", event.getEventType());
        }
    }

    private void handlePaymentCompleted(DomainEvent event) {
        // Update order status
    }

    private void handleInventoryReserved(DomainEvent event) {
        // Proceed with order
    }
}

@Component
@Slf4j
public class InventoryServiceEventHandler {

    @RabbitListener(queues = "inventory-service.domain-events")
    public void handleEvent(DomainEvent event) {
        if ("ORDER_CREATED".equals(event.getEventType())) {
            Order order = convertPayload(event.getPayload(), Order.class);
            reserveInventory(order);
        }
    }

    private void reserveInventory(Order order) {
        // Reserve inventory logic
    }

    private <T> T convertPayload(Object payload, Class<T> type) {
        ObjectMapper mapper = new ObjectMapper();
        return mapper.convertValue(payload, type);
    }
}

@Component
@Slf4j
public class AnalyticsServiceEventHandler {

    @RabbitListener(queues = "analytics-service.domain-events")
    public void handleEvent(DomainEvent event) {
        // All events are interesting for analytics
        recordEvent(event);
    }

    private void recordEvent(DomainEvent event) {
        log.info("Recording event for analytics: type={}, aggregate={}",
            event.getEventType(), event.getAggregateId());
        // Store in analytics database
    }
}
```

## 3. Routing Pattern

### 3.1 Topic-Based Routing

```
Topic Routing Pattern
═════════════════════

Routing Key Format: <entity>.<region>.<action>

                                    ┌─────────────────────┐
                              ┌────▶│ order.*.created     │ All new orders
                              │     └─────────────────────┘
┌───────────┐    ┌─────────┐  │     ┌─────────────────────┐
│ Publisher │───▶│  Topic  │──┼────▶│ order.us.#          │ All US orders
└───────────┘    │Exchange │  │     └─────────────────────┘
                 └─────────┘  │     ┌─────────────────────┐
                              └────▶│ *.*.shipped         │ All shipped items
                                    └─────────────────────┘

Pattern Matching:
  * = exactly one word
  # = zero or more words
```

```java
@Configuration
public class TopicRoutingConfiguration {

    public static final String ORDER_TOPIC_EXCHANGE = "order.topic";

    @Bean
    public TopicExchange orderTopicExchange() {
        return ExchangeBuilder.topicExchange(ORDER_TOPIC_EXCHANGE).durable(true).build();
    }

    // Queue for all new orders
    @Bean
    public Queue allNewOrdersQueue() {
        return QueueBuilder.durable("order.all-new").build();
    }

    @Bean
    public Binding allNewOrdersBinding(Queue allNewOrdersQueue,
            TopicExchange orderTopicExchange) {
        return BindingBuilder
            .bind(allNewOrdersQueue)
            .to(orderTopicExchange)
            .with("order.*.created");
    }

    // Queue for US region orders
    @Bean
    public Queue usOrdersQueue() {
        return QueueBuilder.durable("order.us-region").build();
    }

    @Bean
    public Binding usOrdersBinding(Queue usOrdersQueue,
            TopicExchange orderTopicExchange) {
        return BindingBuilder
            .bind(usOrdersQueue)
            .to(orderTopicExchange)
            .with("order.us.#");
    }

    // Queue for high-value orders (any region)
    @Bean
    public Queue highValueOrdersQueue() {
        return QueueBuilder.durable("order.high-value").build();
    }

    @Bean
    public Binding highValueOrdersBinding(Queue highValueOrdersQueue,
            TopicExchange orderTopicExchange) {
        return BindingBuilder
            .bind(highValueOrdersQueue)
            .to(orderTopicExchange)
            .with("order.*.high-value.#");
    }
}

// Publisher with routing key builder
@Service
@Slf4j
@RequiredArgsConstructor
public class OrderEventPublisher {

    private final RabbitTemplate rabbitTemplate;

    public void publishOrderEvent(Order order, String action) {
        String routingKey = buildRoutingKey(order, action);

        OrderEvent event = OrderEvent.builder()
            .eventId(UUID.randomUUID().toString())
            .eventType(action.toUpperCase())
            .order(order)
            .timestamp(Instant.now())
            .build();

        rabbitTemplate.convertAndSend(
            TopicRoutingConfiguration.ORDER_TOPIC_EXCHANGE,
            routingKey,
            event
        );

        log.info("Order event published: routingKey={}", routingKey);
    }

    private String buildRoutingKey(Order order, String action) {
        StringBuilder key = new StringBuilder("order.");

        // Region
        key.append(order.getRegion().toLowerCase()).append(".");

        // Value tier
        if (order.getTotalAmount().compareTo(new BigDecimal("1000")) > 0) {
            key.append("high-value.");
        }

        // Action
        key.append(action.toLowerCase());

        return key.toString();
    }
}

// Consumers for different routing patterns
@Component
@Slf4j
public class OrderRoutingConsumers {

    @RabbitListener(queues = "order.all-new")
    public void handleAllNewOrders(OrderEvent event) {
        log.info("New order notification: {}", event.getOrder().getOrderId());
        // Send new order notification
    }

    @RabbitListener(queues = "order.us-region")
    public void handleUsOrders(OrderEvent event) {
        log.info("US region order: {}", event.getOrder().getOrderId());
        // US-specific processing
    }

    @RabbitListener(queues = "order.high-value")
    public void handleHighValueOrders(OrderEvent event) {
        log.info("High-value order alert: {} - ${}",
            event.getOrder().getOrderId(),
            event.getOrder().getTotalAmount());
        // Priority handling, VIP notifications
    }
}
```

### 3.2 Header-Based Routing

```java
@Configuration
public class HeaderRoutingConfiguration {

    public static final String HEADERS_EXCHANGE = "notification.headers";

    @Bean
    public HeadersExchange notificationHeadersExchange() {
        return ExchangeBuilder.headersExchange(HEADERS_EXCHANGE).durable(true).build();
    }

    // Urgent email notifications
    @Bean
    public Queue urgentEmailQueue() {
        return QueueBuilder.durable("notification.urgent-email").build();
    }

    @Bean
    public Binding urgentEmailBinding(Queue urgentEmailQueue,
            HeadersExchange notificationHeadersExchange) {
        return BindingBuilder
            .bind(urgentEmailQueue)
            .to(notificationHeadersExchange)
            .whereAll(Map.of(
                "channel", "email",
                "priority", "urgent"
            )).match();
    }

    // Any SMS notification
    @Bean
    public Queue smsQueue() {
        return QueueBuilder.durable("notification.sms").build();
    }

    @Bean
    public Binding smsBinding(Queue smsQueue,
            HeadersExchange notificationHeadersExchange) {
        return BindingBuilder
            .bind(smsQueue)
            .to(notificationHeadersExchange)
            .where("channel").matches("sms");
    }
}

// Publisher
@Service
@Slf4j
@RequiredArgsConstructor
public class NotificationPublisher {

    private final RabbitTemplate rabbitTemplate;

    public void sendNotification(Notification notification) {
        rabbitTemplate.convertAndSend(
            HeaderRoutingConfiguration.HEADERS_EXCHANGE,
            "", // Routing key not used
            notification,
            message -> {
                MessageProperties props = message.getMessageProperties();
                props.setHeader("channel", notification.getChannel().name().toLowerCase());
                props.setHeader("priority", notification.getPriority().name().toLowerCase());
                props.setHeader("userId", notification.getUserId());
                return message;
            }
        );
    }
}
```

## 4. Request/Reply Pattern

### 4.1 Synchronous RPC

```
Request/Reply Pattern
═════════════════════

┌────────────────────────────────────────────────────────────────┐
│                                                                │
│  Client                          Server                        │
│  ┌──────┐                       ┌──────┐                       │
│  │      │ ──── Request ────────▶│      │                       │
│  │      │      (correlationId)  │      │                       │
│  │      │                       │      │ Process               │
│  │      │ ◀─── Response ────────│      │                       │
│  │      │      (correlationId)  │      │                       │
│  └──────┘                       └──────┘                       │
│                                                                │
│  Request Queue: rpc.requests                                   │
│  Reply Queue: amq.rabbitmq.reply-to (direct reply)            │
│                                                                │
└────────────────────────────────────────────────────────────────┘
```

```java
// RPC Request/Response models
@Data
@Builder
public class CalculationRequest {
    private String requestId;
    private String operation;
    private List<BigDecimal> operands;
}

@Data
@Builder
public class CalculationResponse {
    private String requestId;
    private BigDecimal result;
    private String status;
    private String errorMessage;
}

// RPC Server
@Component
@Slf4j
public class CalculatorRpcServer {

    @RabbitListener(queues = "calculator.rpc")
    @SendTo
    public CalculationResponse calculate(CalculationRequest request) {
        log.info("RPC request received: {} {}", request.getOperation(), request.getOperands());

        try {
            BigDecimal result = performCalculation(request);

            return CalculationResponse.builder()
                .requestId(request.getRequestId())
                .result(result)
                .status("SUCCESS")
                .build();

        } catch (Exception e) {
            log.error("Calculation failed", e);

            return CalculationResponse.builder()
                .requestId(request.getRequestId())
                .status("ERROR")
                .errorMessage(e.getMessage())
                .build();
        }
    }

    private BigDecimal performCalculation(CalculationRequest request) {
        List<BigDecimal> operands = request.getOperands();

        return switch (request.getOperation().toUpperCase()) {
            case "ADD" -> operands.stream()
                .reduce(BigDecimal.ZERO, BigDecimal::add);
            case "MULTIPLY" -> operands.stream()
                .reduce(BigDecimal.ONE, BigDecimal::multiply);
            case "SUBTRACT" -> operands.get(0)
                .subtract(operands.subList(1, operands.size()).stream()
                    .reduce(BigDecimal.ZERO, BigDecimal::add));
            case "DIVIDE" -> operands.get(0)
                .divide(operands.get(1), 10, RoundingMode.HALF_UP);
            default -> throw new IllegalArgumentException(
                "Unknown operation: " + request.getOperation());
        };
    }
}

// RPC Client
@Service
@Slf4j
@RequiredArgsConstructor
public class CalculatorRpcClient {

    private final RabbitTemplate rabbitTemplate;

    public CalculationResponse calculate(String operation, BigDecimal... operands) {
        CalculationRequest request = CalculationRequest.builder()
            .requestId(UUID.randomUUID().toString())
            .operation(operation)
            .operands(List.of(operands))
            .build();

        log.info("Sending RPC request: {}", request.getRequestId());

        CalculationResponse response = (CalculationResponse) rabbitTemplate.convertSendAndReceive(
            "calculator.rpc",
            request
        );

        if (response == null) {
            throw new RuntimeException("RPC timeout");
        }

        log.info("RPC response: {} = {}", request.getRequestId(), response.getResult());
        return response;
    }

    public CompletableFuture<CalculationResponse> calculateAsync(
            String operation, BigDecimal... operands) {
        return CompletableFuture.supplyAsync(() -> calculate(operation, operands));
    }
}
```

### 4.2 Async Request/Reply with Correlation

```java
@Service
@Slf4j
public class AsyncRpcClient {

    private final RabbitTemplate rabbitTemplate;
    private final Map<String, CompletableFuture<Object>> pendingRequests;
    private final String replyQueue;

    public AsyncRpcClient(RabbitTemplate rabbitTemplate,
            ConnectionFactory connectionFactory) {
        this.rabbitTemplate = rabbitTemplate;
        this.pendingRequests = new ConcurrentHashMap<>();

        // Create dedicated reply queue
        this.replyQueue = "rpc.replies." + UUID.randomUUID().toString();

        // Set up reply listener
        SimpleMessageListenerContainer container =
            new SimpleMessageListenerContainer(connectionFactory);
        container.setQueueNames(replyQueue);
        container.setMessageListener((MessageListener) this::handleReply);
        container.start();
    }

    public <T> CompletableFuture<T> sendRequest(String queue, Object request,
            Class<T> responseType) {
        String correlationId = UUID.randomUUID().toString();
        CompletableFuture<Object> future = new CompletableFuture<>();
        pendingRequests.put(correlationId, future);

        rabbitTemplate.convertAndSend(queue, request, message -> {
            message.getMessageProperties().setReplyTo(replyQueue);
            message.getMessageProperties().setCorrelationId(correlationId);
            return message;
        });

        // Timeout handling
        return future.orTimeout(30, TimeUnit.SECONDS)
            .thenApply(responseType::cast)
            .whenComplete((result, error) -> {
                pendingRequests.remove(correlationId);
                if (error != null) {
                    log.warn("RPC request failed: {}", correlationId);
                }
            });
    }

    private void handleReply(Message message) {
        String correlationId = message.getMessageProperties().getCorrelationId();
        CompletableFuture<Object> future = pendingRequests.get(correlationId);

        if (future != null) {
            try {
                Object response = rabbitTemplate.getMessageConverter()
                    .fromMessage(message);
                future.complete(response);
            } catch (Exception e) {
                future.completeExceptionally(e);
            }
        }
    }
}
```

## 5. Message Ordering & Deduplication

### 5.1 Ordered Message Processing

```java
/*
 * RabbitMQ guarantees order within a single queue with single consumer.
 * For ordered processing with multiple consumers, use consistent hashing.
 */

@Configuration
public class OrderedProcessingConfiguration {

    @Bean
    public DirectExchange orderedExchange() {
        return new DirectExchange("ordered.exchange");
    }

    // Multiple queues for parallel processing while maintaining order per key
    @Bean
    public Queue orderedQueue1() {
        return new Queue("ordered.queue.1");
    }

    @Bean
    public Queue orderedQueue2() {
        return new Queue("ordered.queue.2");
    }

    @Bean
    public Queue orderedQueue3() {
        return new Queue("ordered.queue.3");
    }
}

@Service
@Slf4j
@RequiredArgsConstructor
public class OrderedPublisher {

    private final RabbitTemplate rabbitTemplate;
    private static final int NUM_QUEUES = 3;

    public void publish(String orderKey, Object message) {
        // Consistent hash to determine queue
        int queueIndex = Math.abs(orderKey.hashCode()) % NUM_QUEUES;
        String routingKey = "ordered.queue." + (queueIndex + 1);

        rabbitTemplate.convertAndSend("ordered.exchange", routingKey, message);

        log.debug("Message sent to {}: key={}", routingKey, orderKey);
    }
}

// Consumer with single concurrency per queue for ordering
@Component
@Slf4j
public class OrderedConsumer {

    @RabbitListener(queues = "ordered.queue.1", concurrency = "1")
    public void consume1(OrderEvent event) {
        processInOrder(event);
    }

    @RabbitListener(queues = "ordered.queue.2", concurrency = "1")
    public void consume2(OrderEvent event) {
        processInOrder(event);
    }

    @RabbitListener(queues = "ordered.queue.3", concurrency = "1")
    public void consume3(OrderEvent event) {
        processInOrder(event);
    }

    private void processInOrder(OrderEvent event) {
        log.info("Processing ordered event: {}", event.getEventId());
    }
}
```

### 5.2 Idempotent Consumer Pattern

```java
@Component
@Slf4j
@RequiredArgsConstructor
public class IdempotentConsumer {

    private final MessageIdRepository messageIdRepository;
    private final OrderService orderService;

    @RabbitListener(queues = "order.process")
    @Transactional
    public void processOrder(
            OrderEvent event,
            Channel channel,
            @Header(AmqpHeaders.DELIVERY_TAG) long deliveryTag,
            @Header(AmqpHeaders.MESSAGE_ID) String messageId) throws IOException {

        // Check for duplicate
        if (messageIdRepository.existsById(messageId)) {
            log.info("Duplicate message detected, skipping: {}", messageId);
            channel.basicAck(deliveryTag, false);
            return;
        }

        try {
            // Process message
            orderService.processOrder(event.getOrder());

            // Record message ID (within same transaction)
            messageIdRepository.save(new ProcessedMessage(messageId, Instant.now()));

            channel.basicAck(deliveryTag, false);
            log.info("Message processed successfully: {}", messageId);

        } catch (Exception e) {
            log.error("Failed to process message: {}", messageId, e);
            channel.basicNack(deliveryTag, false, true);
        }
    }
}

@Entity
@Table(name = "processed_messages")
@Data
@NoArgsConstructor
@AllArgsConstructor
public class ProcessedMessage {
    @Id
    private String messageId;
    private Instant processedAt;
}

@Repository
public interface MessageIdRepository extends JpaRepository<ProcessedMessage, String> {
}

// Cleanup old message IDs
@Service
@Slf4j
@RequiredArgsConstructor
public class MessageIdCleanupService {

    private final MessageIdRepository messageIdRepository;

    @Scheduled(cron = "0 0 * * * *") // Every hour
    @Transactional
    public void cleanupOldMessageIds() {
        Instant cutoff = Instant.now().minus(Duration.ofDays(7));

        int deleted = messageIdRepository.deleteByProcessedAtBefore(cutoff);

        log.info("Cleaned up {} old message IDs", deleted);
    }
}
```

### 5.3 Deduplication with Redis

```java
@Component
@Slf4j
@RequiredArgsConstructor
public class RedisDeduplicationConsumer {

    private final StringRedisTemplate redisTemplate;
    private static final Duration DEDUP_TTL = Duration.ofHours(24);

    @RabbitListener(queues = "dedup.queue")
    public void consume(
            @Payload OrderEvent event,
            @Header(AmqpHeaders.MESSAGE_ID) String messageId,
            Channel channel,
            @Header(AmqpHeaders.DELIVERY_TAG) long deliveryTag) throws IOException {

        String dedupKey = "msg:processed:" + messageId;

        // Try to set with NX (only if not exists)
        Boolean isNew = redisTemplate.opsForValue()
            .setIfAbsent(dedupKey, "1", DEDUP_TTL);

        if (Boolean.TRUE.equals(isNew)) {
            try {
                processEvent(event);
                channel.basicAck(deliveryTag, false);
            } catch (Exception e) {
                // Remove dedup key on failure to allow retry
                redisTemplate.delete(dedupKey);
                channel.basicNack(deliveryTag, false, true);
            }
        } else {
            log.info("Duplicate message skipped: {}", messageId);
            channel.basicAck(deliveryTag, false);
        }
    }

    private void processEvent(OrderEvent event) {
        log.info("Processing event: {}", event.getEventId());
    }
}
```

## 6. Delayed Message Pattern

### 6.1 Delay Queue with TTL and DLX

```java
@Configuration
public class DelayedMessageConfiguration {

    @Bean
    public DirectExchange mainExchange() {
        return new DirectExchange("main.exchange");
    }

    @Bean
    public DirectExchange delayExchange() {
        return new DirectExchange("delay.exchange");
    }

    @Bean
    public Queue mainQueue() {
        return QueueBuilder.durable("main.queue").build();
    }

    // Delay queues with different TTLs
    @Bean
    public Queue delay1sQueue() {
        return QueueBuilder.durable("delay.1s.queue")
            .withArgument("x-dead-letter-exchange", "main.exchange")
            .withArgument("x-dead-letter-routing-key", "main")
            .withArgument("x-message-ttl", 1000)
            .build();
    }

    @Bean
    public Queue delay5sQueue() {
        return QueueBuilder.durable("delay.5s.queue")
            .withArgument("x-dead-letter-exchange", "main.exchange")
            .withArgument("x-dead-letter-routing-key", "main")
            .withArgument("x-message-ttl", 5000)
            .build();
    }

    @Bean
    public Queue delay30sQueue() {
        return QueueBuilder.durable("delay.30s.queue")
            .withArgument("x-dead-letter-exchange", "main.exchange")
            .withArgument("x-dead-letter-routing-key", "main")
            .withArgument("x-message-ttl", 30000)
            .build();
    }

    @Bean
    public Queue delay5mQueue() {
        return QueueBuilder.durable("delay.5m.queue")
            .withArgument("x-dead-letter-exchange", "main.exchange")
            .withArgument("x-dead-letter-routing-key", "main")
            .withArgument("x-message-ttl", 300000)
            .build();
    }

    // Bindings
    @Bean
    public Binding mainBinding(Queue mainQueue, DirectExchange mainExchange) {
        return BindingBuilder.bind(mainQueue).to(mainExchange).with("main");
    }

    @Bean
    public Binding delay1sBinding(Queue delay1sQueue, DirectExchange delayExchange) {
        return BindingBuilder.bind(delay1sQueue).to(delayExchange).with("delay.1s");
    }

    @Bean
    public Binding delay5sBinding(Queue delay5sQueue, DirectExchange delayExchange) {
        return BindingBuilder.bind(delay5sQueue).to(delayExchange).with("delay.5s");
    }

    @Bean
    public Binding delay30sBinding(Queue delay30sQueue, DirectExchange delayExchange) {
        return BindingBuilder.bind(delay30sQueue).to(delayExchange).with("delay.30s");
    }

    @Bean
    public Binding delay5mBinding(Queue delay5mQueue, DirectExchange delayExchange) {
        return BindingBuilder.bind(delay5mQueue).to(delayExchange).with("delay.5m");
    }
}

@Service
@Slf4j
@RequiredArgsConstructor
public class DelayedMessagePublisher {

    private final RabbitTemplate rabbitTemplate;

    public void publishWithDelay(Object message, Duration delay) {
        String routingKey = selectDelayQueue(delay);

        rabbitTemplate.convertAndSend("delay.exchange", routingKey, message,
            msg -> {
                msg.getMessageProperties().setHeader("x-scheduled-time",
                    Instant.now().plus(delay).toString());
                return msg;
            });

        log.info("Message scheduled for delivery in {}", delay);
    }

    private String selectDelayQueue(Duration delay) {
        long millis = delay.toMillis();

        if (millis <= 1000) return "delay.1s";
        if (millis <= 5000) return "delay.5s";
        if (millis <= 30000) return "delay.30s";
        return "delay.5m";
    }

    // Schedule retry with exponential backoff
    public void scheduleRetry(Object message, int attempt) {
        Duration delay = Duration.ofSeconds((long) Math.pow(2, attempt));
        publishWithDelay(message, delay);
    }
}
```

### 6.2 Delayed Message Plugin (RabbitMQ 3.8+)

```java
// Using rabbitmq-delayed-message-exchange plugin

@Configuration
public class PluginDelayConfiguration {

    @Bean
    public CustomExchange delayedExchange() {
        Map<String, Object> args = new HashMap<>();
        args.put("x-delayed-type", "direct");

        return new CustomExchange(
            "delayed.exchange",
            "x-delayed-message", // Plugin exchange type
            true,
            false,
            args
        );
    }

    @Bean
    public Queue delayedQueue() {
        return new Queue("delayed.queue");
    }

    @Bean
    public Binding delayedBinding(Queue delayedQueue, CustomExchange delayedExchange) {
        return BindingBuilder
            .bind(delayedQueue)
            .to(delayedExchange)
            .with("delayed")
            .noargs();
    }
}

@Service
@Slf4j
@RequiredArgsConstructor
public class PluginDelayPublisher {

    private final RabbitTemplate rabbitTemplate;

    public void publishWithExactDelay(Object message, Duration delay) {
        rabbitTemplate.convertAndSend("delayed.exchange", "delayed", message,
            msg -> {
                msg.getMessageProperties().setHeader("x-delay", delay.toMillis());
                return msg;
            });

        log.info("Message will be delivered in exactly {}", delay);
    }
}
```

## 7. Hands-on Exercises

### Exercise 1: Distributed Task Processor
Build a task processing system with priority queues.

```java
// Requirements:
// 1. Tasks have priorities: HIGH, NORMAL, LOW
// 2. Higher priority tasks processed first
// 3. Failed tasks retry with exponential backoff
// 4. Max 5 retries then move to DLQ
// 5. Implement task result callback
```

### Exercise 2: Event Sourcing with RabbitMQ
Implement event sourcing pattern.

```java
// Requirements:
// 1. Publish domain events to fanout exchange
// 2. Event store service saves all events
// 3. Multiple projection services build read models
// 4. Support event replay from beginning
```

### Exercise 3: Priority Notification System
Build notification routing based on urgency.

```java
// Requirements:
// 1. Notifications have: channel (email/sms/push), priority (urgent/normal)
// 2. Urgent notifications go to dedicated fast queue
// 3. Implement rate limiting per channel
// 4. Track delivery status
```

### Exercise 4: Saga Orchestrator
Implement saga pattern for distributed transactions.

```java
// Requirements:
// 1. Order saga: CreateOrder → ReserveInventory → ProcessPayment → Ship
// 2. Each step publishes completion event
// 3. On failure, publish compensation events
// 4. Orchestrator tracks saga state
```

## 8. Key Takeaways

### Pattern Selection
1. **Work Queue**: Distribute tasks among multiple workers
2. **Pub/Sub**: Broadcast to all interested subscribers
3. **Routing**: Selective message delivery based on criteria
4. **RPC**: Synchronous request/reply communication

### Message Reliability
1. **Acknowledgments**: Use manual ack for reliability
2. **Persistence**: Durable queues and persistent messages
3. **Dead Letter Queues**: Handle failed messages gracefully

### Ordering & Deduplication
1. **Ordering**: Single consumer per queue or consistent hashing
2. **Idempotency**: Track processed message IDs
3. **Deduplication**: Redis or database for fast lookup

### Best Practices
1. Use appropriate prefetch count for workload
2. Implement proper error handling and retry logic
3. Monitor queue depths and consumer lag
4. Design for eventual consistency

---

## Navigation

[← Day 17: Spring AMQP](./day-17-spring-amqp.md) | [Day 19: RabbitMQ Clustering →](./day-19-rabbitmq-clustering.md)

[Back to Overview](./00-overview.md)
