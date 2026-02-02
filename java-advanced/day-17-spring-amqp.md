# Day 17: Spring AMQP

## Mục tiêu học tập
- Tích hợp RabbitMQ với Spring Boot sử dụng Spring AMQP
- Cấu hình RabbitTemplate và RabbitListener
- Implement message conversion và error handling
- Xử lý retry và dead letter queues
- Test messaging với Spring AMQP Test

## 1. Spring AMQP Configuration

### 1.1 Dependencies

```xml
<dependencies>
    <!-- Spring Boot Starter AMQP -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-amqp</artifactId>
    </dependency>

    <!-- JSON message conversion -->
    <dependency>
        <groupId>com.fasterxml.jackson.core</groupId>
        <artifactId>jackson-databind</artifactId>
    </dependency>
    <dependency>
        <groupId>com.fasterxml.jackson.datatype</groupId>
        <artifactId>jackson-datatype-jsr310</artifactId>
    </dependency>

    <!-- Testing -->
    <dependency>
        <groupId>org.springframework.amqp</groupId>
        <artifactId>spring-rabbit-test</artifactId>
        <scope>test</scope>
    </dependency>
    <dependency>
        <groupId>org.testcontainers</groupId>
        <artifactId>rabbitmq</artifactId>
        <scope>test</scope>
    </dependency>
</dependencies>
```

### 1.2 Application Configuration

```yaml
# application.yml
spring:
  rabbitmq:
    host: localhost
    port: 5672
    username: guest
    password: guest
    virtual-host: /

    # Connection settings
    connection-timeout: 30000
    channel-rpc-timeout: 30000

    # Publisher confirms
    publisher-confirm-type: correlated
    publisher-returns: true

    # Template settings
    template:
      mandatory: true
      reply-timeout: 30000
      retry:
        enabled: true
        initial-interval: 1000
        max-attempts: 3
        multiplier: 2.0

    # Listener settings
    listener:
      simple:
        acknowledge-mode: manual
        prefetch: 10
        concurrency: 3
        max-concurrency: 10
        retry:
          enabled: true
          initial-interval: 1000
          max-attempts: 3
          max-interval: 10000
          multiplier: 2.0
      direct:
        acknowledge-mode: manual
        prefetch: 5
```

### 1.3 RabbitMQ Configuration Class

```java
@Configuration
@EnableRabbit
@Slf4j
public class RabbitMQConfiguration {

    // Exchange names
    public static final String ORDER_EXCHANGE = "order.exchange";
    public static final String ORDER_DLX = "order.dlx";
    public static final String NOTIFICATION_EXCHANGE = "notification.exchange";

    // Queue names
    public static final String ORDER_QUEUE = "order.queue";
    public static final String ORDER_DLQ = "order.dlq";
    public static final String ORDER_RETRY_QUEUE = "order.retry.queue";
    public static final String NOTIFICATION_EMAIL_QUEUE = "notification.email.queue";
    public static final String NOTIFICATION_SMS_QUEUE = "notification.sms.queue";

    // Routing keys
    public static final String ORDER_ROUTING_KEY = "order.#";
    public static final String ORDER_CREATED_ROUTING_KEY = "order.created";

    // ============= Exchanges =============

    @Bean
    public TopicExchange orderExchange() {
        return ExchangeBuilder
            .topicExchange(ORDER_EXCHANGE)
            .durable(true)
            .build();
    }

    @Bean
    public DirectExchange orderDlx() {
        return ExchangeBuilder
            .directExchange(ORDER_DLX)
            .durable(true)
            .build();
    }

    @Bean
    public FanoutExchange notificationExchange() {
        return ExchangeBuilder
            .fanoutExchange(NOTIFICATION_EXCHANGE)
            .durable(true)
            .build();
    }

    // ============= Queues =============

    @Bean
    public Queue orderQueue() {
        return QueueBuilder
            .durable(ORDER_QUEUE)
            .withArgument("x-dead-letter-exchange", ORDER_DLX)
            .withArgument("x-dead-letter-routing-key", ORDER_DLQ)
            .build();
    }

    @Bean
    public Queue orderDlq() {
        return QueueBuilder
            .durable(ORDER_DLQ)
            .build();
    }

    @Bean
    public Queue orderRetryQueue() {
        return QueueBuilder
            .durable(ORDER_RETRY_QUEUE)
            .withArgument("x-dead-letter-exchange", ORDER_EXCHANGE)
            .withArgument("x-dead-letter-routing-key", ORDER_CREATED_ROUTING_KEY)
            .withArgument("x-message-ttl", 5000) // 5 seconds delay
            .build();
    }

    @Bean
    public Queue notificationEmailQueue() {
        return QueueBuilder
            .durable(NOTIFICATION_EMAIL_QUEUE)
            .build();
    }

    @Bean
    public Queue notificationSmsQueue() {
        return QueueBuilder
            .durable(NOTIFICATION_SMS_QUEUE)
            .build();
    }

    // ============= Bindings =============

    @Bean
    public Binding orderBinding(Queue orderQueue, TopicExchange orderExchange) {
        return BindingBuilder
            .bind(orderQueue)
            .to(orderExchange)
            .with(ORDER_ROUTING_KEY);
    }

    @Bean
    public Binding orderDlqBinding(Queue orderDlq, DirectExchange orderDlx) {
        return BindingBuilder
            .bind(orderDlq)
            .to(orderDlx)
            .with(ORDER_DLQ);
    }

    @Bean
    public Binding notificationEmailBinding(
            Queue notificationEmailQueue,
            FanoutExchange notificationExchange) {
        return BindingBuilder
            .bind(notificationEmailQueue)
            .to(notificationExchange);
    }

    @Bean
    public Binding notificationSmsBinding(
            Queue notificationSmsQueue,
            FanoutExchange notificationExchange) {
        return BindingBuilder
            .bind(notificationSmsQueue)
            .to(notificationExchange);
    }

    // ============= Message Converter =============

    @Bean
    public MessageConverter jsonMessageConverter() {
        ObjectMapper objectMapper = new ObjectMapper()
            .registerModule(new JavaTimeModule())
            .configure(DeserializationFeature.FAIL_ON_UNKNOWN_PROPERTIES, false)
            .configure(SerializationFeature.WRITE_DATES_AS_TIMESTAMPS, false);

        Jackson2JsonMessageConverter converter =
            new Jackson2JsonMessageConverter(objectMapper);
        converter.setCreateMessageIds(true);

        return converter;
    }

    // ============= RabbitTemplate =============

    @Bean
    public RabbitTemplate rabbitTemplate(
            ConnectionFactory connectionFactory,
            MessageConverter messageConverter) {

        RabbitTemplate template = new RabbitTemplate(connectionFactory);
        template.setMessageConverter(messageConverter);
        template.setMandatory(true);

        // Confirm callback
        template.setConfirmCallback((correlationData, ack, cause) -> {
            if (ack) {
                log.debug("Message confirmed: {}",
                    correlationData != null ? correlationData.getId() : "unknown");
            } else {
                log.error("Message not confirmed: {}, cause: {}",
                    correlationData != null ? correlationData.getId() : "unknown", cause);
            }
        });

        // Return callback for unroutable messages
        template.setReturnsCallback(returned -> {
            log.error("Message returned: exchange={}, routingKey={}, replyCode={}, replyText={}",
                returned.getExchange(),
                returned.getRoutingKey(),
                returned.getReplyCode(),
                returned.getReplyText());
        });

        return template;
    }

    // ============= Listener Container Factory =============

    @Bean
    public SimpleRabbitListenerContainerFactory rabbitListenerContainerFactory(
            ConnectionFactory connectionFactory,
            MessageConverter messageConverter) {

        SimpleRabbitListenerContainerFactory factory =
            new SimpleRabbitListenerContainerFactory();

        factory.setConnectionFactory(connectionFactory);
        factory.setMessageConverter(messageConverter);
        factory.setAcknowledgeMode(AcknowledgeMode.MANUAL);
        factory.setPrefetchCount(10);
        factory.setConcurrentConsumers(3);
        factory.setMaxConcurrentConsumers(10);

        // Error handler
        factory.setErrorHandler(new ConditionalRejectingErrorHandler(
            new CustomFatalExceptionStrategy()));

        // Missing queues fatal
        factory.setMissingQueuesFatal(false);

        return factory;
    }

    // Custom fatal exception strategy
    public static class CustomFatalExceptionStrategy
            extends ConditionalRejectingErrorHandler.DefaultExceptionStrategy {

        @Override
        public boolean isFatal(Throwable t) {
            // Non-retryable exceptions
            if (t instanceof MessageConversionException) {
                return true;
            }
            if (t instanceof MethodArgumentNotValidException) {
                return true;
            }
            return super.isFatal(t);
        }
    }
}
```

## 2. Producer Implementation

### 2.1 Basic Producer

```java
@Service
@Slf4j
@RequiredArgsConstructor
public class OrderProducer {

    private final RabbitTemplate rabbitTemplate;

    /**
     * Simple send without confirmation waiting
     */
    public void sendOrder(OrderEvent event) {
        log.info("Sending order event: {}", event.getEventId());

        rabbitTemplate.convertAndSend(
            RabbitMQConfiguration.ORDER_EXCHANGE,
            "order." + event.getEventType().toLowerCase(),
            event
        );
    }

    /**
     * Send with custom message properties
     */
    public void sendOrderWithProperties(OrderEvent event, int priority) {
        rabbitTemplate.convertAndSend(
            RabbitMQConfiguration.ORDER_EXCHANGE,
            "order." + event.getEventType().toLowerCase(),
            event,
            message -> {
                MessageProperties props = message.getMessageProperties();
                props.setPriority(priority);
                props.setHeader("source", "order-service");
                props.setHeader("version", "1.0");
                props.setExpiration("60000"); // 60 seconds TTL
                return message;
            }
        );
    }

    /**
     * Send with correlation data for confirms
     */
    public void sendOrderWithConfirmation(OrderEvent event) {
        CorrelationData correlationData = new CorrelationData(event.getEventId());

        rabbitTemplate.convertAndSend(
            RabbitMQConfiguration.ORDER_EXCHANGE,
            "order." + event.getEventType().toLowerCase(),
            event,
            correlationData
        );

        // Async check confirmation
        correlationData.getFuture().whenComplete((confirm, throwable) -> {
            if (throwable != null) {
                log.error("Error sending message: {}", event.getEventId(), throwable);
            } else if (confirm != null && confirm.isAck()) {
                log.info("Message confirmed: {}", event.getEventId());
            } else {
                log.warn("Message not confirmed: {}, reason: {}",
                    event.getEventId(),
                    confirm != null ? confirm.getReason() : "unknown");
            }
        });
    }
}
```

### 2.2 Async Producer with CompletableFuture

```java
@Service
@Slf4j
public class AsyncOrderProducer {

    private final RabbitTemplate rabbitTemplate;
    private final AsyncRabbitTemplate asyncRabbitTemplate;

    public AsyncOrderProducer(RabbitTemplate rabbitTemplate,
            ConnectionFactory connectionFactory,
            MessageConverter messageConverter) {
        this.rabbitTemplate = rabbitTemplate;
        this.asyncRabbitTemplate = new AsyncRabbitTemplate(
            rabbitTemplate,
            new SimpleMessageListenerContainer(connectionFactory),
            "amq.rabbitmq.reply-to" // Direct reply-to
        );
        asyncRabbitTemplate.setReceiveTimeout(30000);
    }

    /**
     * Send and wait for reply (RPC pattern)
     */
    public CompletableFuture<OrderResponse> sendOrderAndWaitForReply(OrderEvent event) {
        RabbitConverterFuture<OrderResponse> future = asyncRabbitTemplate.convertSendAndReceive(
            RabbitMQConfiguration.ORDER_EXCHANGE,
            "order.request",
            event
        );

        return future.thenApply(response -> {
            if (response == null) {
                throw new RuntimeException("No response received for order: " +
                    event.getEventId());
            }
            return response;
        });
    }

    /**
     * Batch send with parallel confirmation
     */
    public CompletableFuture<BatchSendResult> sendBatch(List<OrderEvent> events) {
        List<CompletableFuture<Boolean>> futures = events.stream()
            .map(event -> {
                CorrelationData correlationData = new CorrelationData(event.getEventId());

                rabbitTemplate.convertAndSend(
                    RabbitMQConfiguration.ORDER_EXCHANGE,
                    "order." + event.getEventType().toLowerCase(),
                    event,
                    correlationData
                );

                return correlationData.getFuture()
                    .thenApply(confirm -> confirm != null && confirm.isAck())
                    .exceptionally(ex -> false);
            })
            .collect(Collectors.toList());

        return CompletableFuture.allOf(futures.toArray(new CompletableFuture[0]))
            .thenApply(v -> {
                long successCount = futures.stream()
                    .map(CompletableFuture::join)
                    .filter(Boolean::booleanValue)
                    .count();

                return new BatchSendResult(
                    events.size(),
                    (int) successCount,
                    events.size() - (int) successCount
                );
            });
    }

    @Data
    @AllArgsConstructor
    public static class BatchSendResult {
        private int total;
        private int success;
        private int failed;
    }
}
```

### 2.3 Notification Producer

```java
@Service
@Slf4j
@RequiredArgsConstructor
public class NotificationProducer {

    private final RabbitTemplate rabbitTemplate;

    /**
     * Broadcast notification to all handlers (email, sms, push)
     */
    public void broadcastNotification(NotificationEvent notification) {
        log.info("Broadcasting notification: {}", notification.getNotificationId());

        rabbitTemplate.convertAndSend(
            RabbitMQConfiguration.NOTIFICATION_EXCHANGE,
            "", // Routing key ignored for fanout
            notification
        );
    }

    /**
     * Send with delivery guarantee
     */
    public CompletableFuture<Boolean> sendWithGuarantee(NotificationEvent notification) {
        CorrelationData correlationData =
            new CorrelationData(notification.getNotificationId());

        rabbitTemplate.convertAndSend(
            RabbitMQConfiguration.NOTIFICATION_EXCHANGE,
            "",
            notification,
            message -> {
                message.getMessageProperties().setDeliveryMode(MessageDeliveryMode.PERSISTENT);
                return message;
            },
            correlationData
        );

        return correlationData.getFuture()
            .thenApply(confirm -> confirm != null && confirm.isAck())
            .exceptionally(ex -> {
                log.error("Failed to send notification: {}",
                    notification.getNotificationId(), ex);
                return false;
            });
    }
}
```

## 3. Consumer Implementation

### 3.1 Basic Consumer with @RabbitListener

```java
@Component
@Slf4j
public class OrderConsumer {

    /**
     * Basic consumer with auto acknowledgment
     */
    @RabbitListener(queues = RabbitMQConfiguration.ORDER_QUEUE)
    public void handleOrder(OrderEvent event) {
        log.info("Received order: {}", event.getEventId());

        // Process order
        processOrder(event);

        log.info("Order processed: {}", event.getEventId());
    }

    /**
     * Consumer with manual acknowledgment
     */
    @RabbitListener(
        queues = RabbitMQConfiguration.ORDER_QUEUE,
        ackMode = "MANUAL"
    )
    public void handleOrderManual(
            OrderEvent event,
            Channel channel,
            @Header(AmqpHeaders.DELIVERY_TAG) long deliveryTag,
            @Header(AmqpHeaders.MESSAGE_ID) String messageId) {

        log.info("Received order (manual ack): messageId={}", messageId);

        try {
            processOrder(event);
            channel.basicAck(deliveryTag, false);
            log.info("Order acknowledged: {}", messageId);

        } catch (RetryableException e) {
            log.warn("Retryable error, requeuing: {}", e.getMessage());
            try {
                channel.basicNack(deliveryTag, false, true);
            } catch (IOException ex) {
                log.error("Failed to nack message", ex);
            }

        } catch (Exception e) {
            log.error("Non-retryable error, rejecting: {}", e.getMessage());
            try {
                channel.basicNack(deliveryTag, false, false); // Goes to DLQ
            } catch (IOException ex) {
                log.error("Failed to nack message", ex);
            }
        }
    }

    /**
     * Consumer with access to Message object
     */
    @RabbitListener(queues = RabbitMQConfiguration.ORDER_QUEUE)
    public void handleOrderWithMessage(
            OrderEvent event,
            Message message,
            @Header(AmqpHeaders.RECEIVED_ROUTING_KEY) String routingKey) {

        MessageProperties props = message.getMessageProperties();

        log.info("Received order: routingKey={}, redelivered={}, priority={}",
            routingKey,
            props.isRedelivered(),
            props.getPriority());

        processOrder(event);
    }

    private void processOrder(OrderEvent event) {
        // Business logic
        log.info("Processing order: {} for customer: {}",
            event.getOrder().getOrderId(),
            event.getOrder().getCustomerId());

        // Simulate processing
        try {
            Thread.sleep(100);
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }
    }

    public static class RetryableException extends RuntimeException {
        public RetryableException(String message) {
            super(message);
        }
    }
}
```

### 3.2 Declarative Queue Configuration

```java
@Component
@Slf4j
public class DeclarativeConsumer {

    /**
     * Consumer with inline queue/binding declaration
     */
    @RabbitListener(bindings = @QueueBinding(
        value = @org.springframework.amqp.rabbit.annotation.Queue(
            name = "payment.queue",
            durable = "true",
            arguments = {
                @Argument(name = "x-dead-letter-exchange", value = "payment.dlx"),
                @Argument(name = "x-max-priority", value = "10", type = "java.lang.Integer")
            }
        ),
        exchange = @org.springframework.amqp.rabbit.annotation.Exchange(
            name = "payment.exchange",
            type = ExchangeTypes.TOPIC
        ),
        key = {"payment.#"}
    ))
    public void handlePayment(PaymentEvent event) {
        log.info("Processing payment: {}", event.getPaymentId());
        // Process payment
    }

    /**
     * Multiple queues in one listener
     */
    @RabbitListener(queues = {
        RabbitMQConfiguration.NOTIFICATION_EMAIL_QUEUE,
        RabbitMQConfiguration.NOTIFICATION_SMS_QUEUE
    })
    public void handleNotification(
            NotificationEvent event,
            @Header(AmqpHeaders.CONSUMER_QUEUE) String queue) {

        log.info("Notification received from queue {}: {}", queue, event.getNotificationId());

        switch (queue) {
            case RabbitMQConfiguration.NOTIFICATION_EMAIL_QUEUE:
                sendEmail(event);
                break;
            case RabbitMQConfiguration.NOTIFICATION_SMS_QUEUE:
                sendSms(event);
                break;
        }
    }

    private void sendEmail(NotificationEvent event) {
        log.info("Sending email to: {}", event.getRecipient());
    }

    private void sendSms(NotificationEvent event) {
        log.info("Sending SMS to: {}", event.getRecipient());
    }
}
```

### 3.3 Batch Consumer

```java
@Component
@Slf4j
public class BatchConsumer {

    /**
     * Process messages in batches
     */
    @RabbitListener(
        queues = "analytics.queue",
        containerFactory = "batchContainerFactory"
    )
    public void handleBatch(List<AnalyticsEvent> events) {
        log.info("Received batch of {} events", events.size());

        // Process batch
        processBatch(events);

        log.info("Batch processed successfully");
    }

    /**
     * Batch with manual ack
     */
    @RabbitListener(
        queues = "analytics.queue",
        containerFactory = "batchContainerFactory",
        ackMode = "MANUAL"
    )
    public void handleBatchManual(
            List<AnalyticsEvent> events,
            Channel channel,
            @Header(AmqpHeaders.DELIVERY_TAG) long lastDeliveryTag) {

        try {
            processBatch(events);
            // Acknowledge all messages in batch
            channel.basicAck(lastDeliveryTag, true);

        } catch (Exception e) {
            log.error("Batch processing failed", e);
            try {
                channel.basicNack(lastDeliveryTag, true, true); // Requeue all
            } catch (IOException ex) {
                log.error("Failed to nack batch", ex);
            }
        }
    }

    private void processBatch(List<AnalyticsEvent> events) {
        // Batch insert to database, etc.
    }

    @Bean
    public SimpleRabbitListenerContainerFactory batchContainerFactory(
            ConnectionFactory connectionFactory,
            MessageConverter messageConverter) {

        SimpleRabbitListenerContainerFactory factory =
            new SimpleRabbitListenerContainerFactory();

        factory.setConnectionFactory(connectionFactory);
        factory.setMessageConverter(messageConverter);
        factory.setBatchListener(true);
        factory.setBatchSize(100);
        factory.setReceiveTimeout(5000L); // Wait max 5 seconds for batch
        factory.setConsumerBatchEnabled(true);

        return factory;
    }
}
```

## 4. Error Handling & Retry

### 4.1 Retry with @Retryable

```java
@Component
@Slf4j
public class RetryableConsumer {

    @RabbitListener(queues = RabbitMQConfiguration.ORDER_QUEUE)
    @Retryable(
        retryFor = RetryableException.class,
        maxAttempts = 3,
        backoff = @Backoff(delay = 1000, multiplier = 2)
    )
    public void handleOrder(OrderEvent event) {
        log.info("Processing order: {}, attempt: {}",
            event.getEventId(), getRetryCount());

        // Might throw RetryableException
        processOrder(event);
    }

    @Recover
    public void recover(RetryableException e, OrderEvent event) {
        log.error("All retries failed for order: {}", event.getEventId());
        // Send to DLQ or alert
        sendToDlq(event, e);
    }

    private int getRetryCount() {
        // Access retry context if needed
        return 1; // Simplified
    }

    private void processOrder(OrderEvent event) {
        // Business logic that might fail
    }

    private void sendToDlq(OrderEvent event, Exception e) {
        // Manual DLQ handling
    }
}
```

### 4.2 Custom Error Handler

```java
@Configuration
public class ErrorHandlerConfiguration {

    @Bean
    public SimpleRabbitListenerContainerFactory retryContainerFactory(
            ConnectionFactory connectionFactory,
            MessageConverter messageConverter,
            MessageRecoverer messageRecoverer) {

        SimpleRabbitListenerContainerFactory factory =
            new SimpleRabbitListenerContainerFactory();

        factory.setConnectionFactory(connectionFactory);
        factory.setMessageConverter(messageConverter);
        factory.setAcknowledgeMode(AcknowledgeMode.AUTO);

        // Retry advice
        RetryInterceptorBuilder.StatelessRetryInterceptorBuilder builder =
            RetryInterceptorBuilder.stateless()
                .maxAttempts(3)
                .backOffOptions(1000, 2.0, 10000)
                .recoverer(messageRecoverer);

        factory.setAdviceChain(builder.build());

        return factory;
    }

    @Bean
    public MessageRecoverer messageRecoverer(RabbitTemplate rabbitTemplate) {
        // Send to DLQ after all retries exhausted
        return new RepublishMessageRecoverer(
            rabbitTemplate,
            RabbitMQConfiguration.ORDER_DLX,
            RabbitMQConfiguration.ORDER_DLQ
        );
    }
}

// Custom recoverer with additional logic
@Component
@Slf4j
@RequiredArgsConstructor
public class CustomMessageRecoverer implements MessageRecoverer {

    private final RabbitTemplate rabbitTemplate;
    private final AlertService alertService;

    @Override
    public void recover(Message message, Throwable cause) {
        String messageId = message.getMessageProperties().getMessageId();
        String routingKey = message.getMessageProperties().getReceivedRoutingKey();

        log.error("Message recovery triggered: messageId={}, routingKey={}, cause={}",
            messageId, routingKey, cause.getMessage());

        // Add failure metadata
        message.getMessageProperties().setHeader("x-exception-message", cause.getMessage());
        message.getMessageProperties().setHeader("x-exception-type",
            cause.getClass().getName());
        message.getMessageProperties().setHeader("x-original-routing-key", routingKey);
        message.getMessageProperties().setHeader("x-failure-time",
            System.currentTimeMillis());

        // Send to DLQ
        rabbitTemplate.send(
            RabbitMQConfiguration.ORDER_DLX,
            RabbitMQConfiguration.ORDER_DLQ,
            message
        );

        // Alert team
        alertService.sendAlert("Message processing failed after retries",
            Map.of("messageId", messageId, "error", cause.getMessage()));
    }
}
```

### 4.3 DLQ Processor

```java
@Component
@Slf4j
@RequiredArgsConstructor
public class DlqProcessor {

    private final RabbitTemplate rabbitTemplate;
    private final ObjectMapper objectMapper;

    /**
     * Process DLQ messages for analysis
     */
    @RabbitListener(queues = RabbitMQConfiguration.ORDER_DLQ)
    public void processDlq(
            Message message,
            @Header("x-exception-message") String exceptionMessage,
            @Header("x-original-routing-key") String originalRoutingKey,
            @Header("x-failure-time") Long failureTime) {

        log.warn("DLQ message received: originalRoutingKey={}, exception={}, failedAt={}",
            originalRoutingKey, exceptionMessage, Instant.ofEpochMilli(failureTime));

        // Parse message for logging/analysis
        try {
            String body = new String(message.getBody(), StandardCharsets.UTF_8);
            log.info("DLQ message body: {}", body);

            // Store in dead letter table for later analysis
            storeDlqMessage(message, exceptionMessage);

        } catch (Exception e) {
            log.error("Failed to process DLQ message", e);
        }
    }

    /**
     * Replay message from DLQ
     */
    public boolean replayMessage(String messageId) {
        // Find message in DLQ storage
        DlqMessage dlqMessage = findDlqMessage(messageId);

        if (dlqMessage == null) {
            log.warn("Message not found in DLQ: {}", messageId);
            return false;
        }

        try {
            // Republish to original queue
            rabbitTemplate.send(
                RabbitMQConfiguration.ORDER_EXCHANGE,
                dlqMessage.getOriginalRoutingKey(),
                MessageBuilder
                    .withBody(dlqMessage.getBody())
                    .setContentType(MessageProperties.CONTENT_TYPE_JSON)
                    .setHeader("x-replayed-from-dlq", true)
                    .setHeader("x-replay-time", System.currentTimeMillis())
                    .build()
            );

            log.info("Message replayed from DLQ: {}", messageId);
            return true;

        } catch (Exception e) {
            log.error("Failed to replay message: {}", messageId, e);
            return false;
        }
    }

    private void storeDlqMessage(Message message, String exception) {
        // Store to database for later analysis/replay
    }

    private DlqMessage findDlqMessage(String messageId) {
        // Retrieve from database
        return null;
    }

    @Data
    static class DlqMessage {
        private String messageId;
        private byte[] body;
        private String originalRoutingKey;
        private Instant failureTime;
    }
}
```

## 5. Request/Reply Pattern (RPC)

### 5.1 RPC Server

```java
@Component
@Slf4j
public class OrderRpcServer {

    @RabbitListener(queues = "order.rpc.queue")
    @SendTo // Uses replyTo from message properties
    public OrderResponse processOrderRequest(OrderRequest request) {
        log.info("RPC request received: {}", request.getOrderId());

        try {
            // Process request
            Order order = processOrder(request);

            return OrderResponse.builder()
                .orderId(order.getOrderId())
                .status("SUCCESS")
                .message("Order processed successfully")
                .processedAt(Instant.now())
                .build();

        } catch (Exception e) {
            log.error("RPC request failed: {}", request.getOrderId(), e);

            return OrderResponse.builder()
                .orderId(request.getOrderId())
                .status("ERROR")
                .message(e.getMessage())
                .processedAt(Instant.now())
                .build();
        }
    }

    /**
     * RPC with explicit reply handling
     */
    @RabbitListener(queues = "order.rpc.queue")
    public void processOrderRequestExplicit(
            OrderRequest request,
            Channel channel,
            @Header(AmqpHeaders.REPLY_TO) String replyTo,
            @Header(AmqpHeaders.CORRELATION_ID) String correlationId,
            @Header(AmqpHeaders.DELIVERY_TAG) long deliveryTag)
            throws IOException {

        log.info("RPC request received: orderId={}, correlationId={}",
            request.getOrderId(), correlationId);

        try {
            Order order = processOrder(request);

            OrderResponse response = OrderResponse.builder()
                .orderId(order.getOrderId())
                .status("SUCCESS")
                .build();

            // Send reply
            AMQP.BasicProperties props = new AMQP.BasicProperties.Builder()
                .correlationId(correlationId)
                .contentType("application/json")
                .build();

            ObjectMapper mapper = new ObjectMapper().registerModule(new JavaTimeModule());

            channel.basicPublish(
                "",
                replyTo,
                props,
                mapper.writeValueAsBytes(response)
            );

            channel.basicAck(deliveryTag, false);

        } catch (Exception e) {
            log.error("RPC failed", e);
            channel.basicNack(deliveryTag, false, false);
        }
    }

    private Order processOrder(OrderRequest request) {
        // Business logic
        return Order.builder()
            .orderId(request.getOrderId())
            .status(OrderStatus.CREATED)
            .build();
    }
}
```

### 5.2 RPC Client

```java
@Service
@Slf4j
@RequiredArgsConstructor
public class OrderRpcClient {

    private final RabbitTemplate rabbitTemplate;

    /**
     * Synchronous RPC call
     */
    public OrderResponse callOrderService(OrderRequest request) {
        log.info("Sending RPC request: {}", request.getOrderId());

        OrderResponse response = (OrderResponse) rabbitTemplate.convertSendAndReceive(
            "", // Default exchange
            "order.rpc.queue",
            request,
            message -> {
                message.getMessageProperties()
                    .setReplyTo("amq.rabbitmq.reply-to"); // Direct reply-to
                return message;
            }
        );

        if (response == null) {
            throw new RuntimeException("RPC timeout for order: " + request.getOrderId());
        }

        log.info("RPC response received: {}", response.getStatus());
        return response;
    }

    /**
     * Async RPC with CompletableFuture
     */
    public CompletableFuture<OrderResponse> callOrderServiceAsync(OrderRequest request) {
        return CompletableFuture.supplyAsync(() -> callOrderService(request));
    }

    /**
     * RPC with timeout
     */
    public OrderResponse callOrderServiceWithTimeout(OrderRequest request,
            Duration timeout) {
        RabbitTemplate templateWithTimeout = new RabbitTemplate(
            rabbitTemplate.getConnectionFactory());
        templateWithTimeout.setMessageConverter(rabbitTemplate.getMessageConverter());
        templateWithTimeout.setReplyTimeout(timeout.toMillis());

        return (OrderResponse) templateWithTimeout.convertSendAndReceive(
            "",
            "order.rpc.queue",
            request
        );
    }
}

@Data
@Builder
class OrderRequest {
    private String orderId;
    private String customerId;
    private List<OrderItem> items;
}

@Data
@Builder
class OrderResponse {
    private String orderId;
    private String status;
    private String message;
    private Instant processedAt;
}
```

## 6. Testing

### 6.1 Integration Test with Embedded RabbitMQ

```java
@SpringBootTest
@EmbeddedRabbit
@Slf4j
class OrderMessagingIntegrationTest {

    @Autowired
    private RabbitTemplate rabbitTemplate;

    @Autowired
    private OrderProducer orderProducer;

    @Autowired
    private RabbitAdmin rabbitAdmin;

    @Test
    void shouldSendAndReceiveOrder() throws Exception {
        // Given
        Order order = Order.builder()
            .orderId("test-123")
            .customerId("customer-456")
            .totalAmount(new BigDecimal("99.99"))
            .build();

        OrderEvent event = OrderEvent.builder()
            .eventId(UUID.randomUUID().toString())
            .eventType("CREATED")
            .order(order)
            .timestamp(Instant.now())
            .build();

        // When
        orderProducer.sendOrder(event);

        // Then - receive with blocking
        OrderEvent received = (OrderEvent) rabbitTemplate.receiveAndConvert(
            RabbitMQConfiguration.ORDER_QUEUE,
            5000
        );

        assertThat(received).isNotNull();
        assertThat(received.getOrder().getOrderId()).isEqualTo("test-123");
    }

    @Test
    void shouldRouteToCorrectQueue() {
        // Given
        OrderEvent highValueOrder = createOrderEvent("high-value", new BigDecimal("5000"));
        OrderEvent normalOrder = createOrderEvent("normal", new BigDecimal("50"));

        // When
        rabbitTemplate.convertAndSend(
            RabbitMQConfiguration.ORDER_EXCHANGE,
            "order.high-value.created",
            highValueOrder
        );
        rabbitTemplate.convertAndSend(
            RabbitMQConfiguration.ORDER_EXCHANGE,
            "order.normal.created",
            normalOrder
        );

        // Then - both should be in order queue (topic matches order.#)
        assertThat(rabbitAdmin.getQueueInfo(RabbitMQConfiguration.ORDER_QUEUE)
            .getMessageCount()).isEqualTo(2);
    }

    private OrderEvent createOrderEvent(String orderId, BigDecimal amount) {
        return OrderEvent.builder()
            .eventId(UUID.randomUUID().toString())
            .eventType("CREATED")
            .order(Order.builder()
                .orderId(orderId)
                .totalAmount(amount)
                .build())
            .timestamp(Instant.now())
            .build();
    }
}
```

### 6.2 Testing with Testcontainers

```java
@SpringBootTest
@Testcontainers
@Slf4j
class OrderMessagingTestcontainersTest {

    @Container
    static RabbitMQContainer rabbitMQ = new RabbitMQContainer("rabbitmq:3.12-management")
        .withExposedPorts(5672, 15672);

    @DynamicPropertySource
    static void registerRabbitMQProperties(DynamicPropertyRegistry registry) {
        registry.add("spring.rabbitmq.host", rabbitMQ::getHost);
        registry.add("spring.rabbitmq.port", rabbitMQ::getAmqpPort);
        registry.add("spring.rabbitmq.username", () -> "guest");
        registry.add("spring.rabbitmq.password", () -> "guest");
    }

    @Autowired
    private RabbitTemplate rabbitTemplate;

    @Autowired
    private OrderProducer orderProducer;

    @Test
    void shouldProcessOrderWithRealRabbitMQ() {
        // Given
        OrderEvent event = createOrderEvent();

        // When
        orderProducer.sendOrderWithConfirmation(event);

        // Then
        await().atMost(Duration.ofSeconds(5)).untilAsserted(() -> {
            OrderEvent received = (OrderEvent) rabbitTemplate.receiveAndConvert(
                RabbitMQConfiguration.ORDER_QUEUE,
                1000
            );
            assertThat(received).isNotNull();
        });
    }

    @Test
    void shouldSendToDlqAfterMaxRetries() {
        // Given - message that will always fail
        String badMessage = "invalid-json";

        // When
        rabbitTemplate.send(
            RabbitMQConfiguration.ORDER_EXCHANGE,
            "order.created",
            MessageBuilder.withBody(badMessage.getBytes())
                .setContentType("application/json")
                .build()
        );

        // Then - should end up in DLQ
        await().atMost(Duration.ofSeconds(30)).untilAsserted(() -> {
            Message dlqMessage = rabbitTemplate.receive(
                RabbitMQConfiguration.ORDER_DLQ,
                1000
            );
            assertThat(dlqMessage).isNotNull();
        });
    }

    private OrderEvent createOrderEvent() {
        return OrderEvent.builder()
            .eventId(UUID.randomUUID().toString())
            .eventType("CREATED")
            .order(Order.builder()
                .orderId("test-" + System.currentTimeMillis())
                .customerId("cust-123")
                .totalAmount(new BigDecimal("199.99"))
                .build())
            .timestamp(Instant.now())
            .build();
    }
}
```

### 6.3 Mocking RabbitTemplate

```java
@ExtendWith(MockitoExtension.class)
class OrderProducerTest {

    @Mock
    private RabbitTemplate rabbitTemplate;

    @InjectMocks
    private OrderProducer orderProducer;

    @Captor
    private ArgumentCaptor<OrderEvent> eventCaptor;

    @Captor
    private ArgumentCaptor<String> routingKeyCaptor;

    @Test
    void shouldSendOrderWithCorrectRoutingKey() {
        // Given
        OrderEvent event = OrderEvent.builder()
            .eventId("test-event")
            .eventType("CREATED")
            .order(Order.builder()
                .orderId("order-123")
                .build())
            .build();

        // When
        orderProducer.sendOrder(event);

        // Then
        verify(rabbitTemplate).convertAndSend(
            eq(RabbitMQConfiguration.ORDER_EXCHANGE),
            routingKeyCaptor.capture(),
            eventCaptor.capture()
        );

        assertThat(routingKeyCaptor.getValue()).isEqualTo("order.created");
        assertThat(eventCaptor.getValue().getEventId()).isEqualTo("test-event");
    }

    @Test
    void shouldSetCorrectPriority() {
        // Given
        OrderEvent event = createOrderEvent();
        int expectedPriority = 5;

        // When
        orderProducer.sendOrderWithProperties(event, expectedPriority);

        // Then
        verify(rabbitTemplate).convertAndSend(
            anyString(),
            anyString(),
            any(OrderEvent.class),
            any(MessagePostProcessor.class)
        );
    }

    private OrderEvent createOrderEvent() {
        return OrderEvent.builder()
            .eventId(UUID.randomUUID().toString())
            .eventType("CREATED")
            .order(Order.builder().build())
            .timestamp(Instant.now())
            .build();
    }
}
```

## 7. Hands-on Exercises

### Exercise 1: Multi-Service Order System
Build a complete order processing system with RabbitMQ.

```java
// Requirements:
// 1. Order Service: Publishes order events
// 2. Inventory Service: Reserves stock, replies with confirmation
// 3. Payment Service: Processes payment
// 4. Notification Service: Sends email/SMS
// 5. Use topic exchange for routing
// 6. Implement saga pattern with compensation
```

### Exercise 2: Priority Queue Implementation
Create a priority-based task processing system.

```java
// Requirements:
// 1. Tasks have priority levels 1-10
// 2. Higher priority tasks processed first
// 3. Implement fair distribution among workers
// 4. Add monitoring for queue depth by priority
```

### Exercise 3: Delayed Message Processing
Implement scheduled message delivery.

```java
// Requirements:
// 1. Accept messages with delivery delay
// 2. Use delay queues with TTL
// 3. Support variable delays (1s, 5s, 30s, 5m)
// 4. Handle message expiration
```

### Exercise 4: DLQ Dashboard
Build a DLQ management system.

```java
// Requirements:
// 1. REST API to list DLQ messages
// 2. Replay single or batch messages
// 3. Purge old messages
// 4. Alert when DLQ exceeds threshold
```

## 8. Key Takeaways

### Configuration
1. **Publisher Confirms**: Enable `publisher-confirm-type: correlated` for delivery guarantees
2. **Manual Ack**: Use `acknowledge-mode: manual` for reliable processing
3. **Prefetch**: Balance throughput and fairness with appropriate prefetch count

### Producer Patterns
1. **CorrelationData**: Track message confirmations
2. **MessagePostProcessor**: Customize message properties
3. **Mandatory**: Detect unroutable messages

### Consumer Patterns
1. **@RabbitListener**: Declarative message handling
2. **Manual Acknowledgment**: Control when messages are removed
3. **Batch Processing**: Efficient bulk operations

### Error Handling
1. **Retry Configuration**: Exponential backoff for transient failures
2. **MessageRecoverer**: Custom DLQ handling
3. **Error Handlers**: Distinguish fatal vs retryable errors

---

## Navigation

[← Day 16: RabbitMQ Fundamentals](./day-16-rabbitmq-fundamentals.md) | [Day 18: RabbitMQ Patterns →](./day-18-rabbitmq-patterns.md)

[Back to Overview](./00-overview.md)
