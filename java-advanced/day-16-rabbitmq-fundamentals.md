# Day 16: RabbitMQ Fundamentals

## Mục tiêu học tập
- Hiểu kiến trúc và core concepts của RabbitMQ
- Phân biệt Exchange types và routing mechanisms
- Implement Producer/Consumer với Java client
- Cấu hình message acknowledgment và durability
- So sánh RabbitMQ với Kafka

## 1. RabbitMQ Overview

### 1.1 AMQP Protocol

```
┌─────────────────────────────────────────────────────────────────┐
│                    AMQP 0-9-1 Model                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌──────────┐    ┌──────────┐    ┌─────────┐    ┌──────────┐  │
│  │ Producer │───▶│ Exchange │───▶│  Queue  │───▶│ Consumer │  │
│  └──────────┘    └──────────┘    └─────────┘    └──────────┘  │
│                       │                                         │
│                       │ Binding (routing key)                   │
│                       ▼                                         │
│              ┌─────────────────┐                               │
│              │ Routing Rules   │                               │
│              │ - Direct        │                               │
│              │ - Topic         │                               │
│              │ - Fanout        │                               │
│              │ - Headers       │                               │
│              └─────────────────┘                               │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘

Key Concepts:
┌─────────────┬─────────────────────────────────────────────────┐
│ Producer    │ Application that publishes messages            │
├─────────────┼─────────────────────────────────────────────────┤
│ Exchange    │ Routes messages to queues based on rules       │
├─────────────┼─────────────────────────────────────────────────┤
│ Queue       │ Buffer that stores messages                     │
├─────────────┼─────────────────────────────────────────────────┤
│ Binding     │ Link between exchange and queue                │
├─────────────┼─────────────────────────────────────────────────┤
│ Consumer    │ Application that receives messages              │
├─────────────┼─────────────────────────────────────────────────┤
│ Virtual Host│ Logical grouping (multi-tenancy)               │
└─────────────┴─────────────────────────────────────────────────┘
```

### 1.2 RabbitMQ Architecture

```
┌────────────────────────────────────────────────────────────────────┐
│                     RabbitMQ Server                                │
├────────────────────────────────────────────────────────────────────┤
│                                                                    │
│  ┌──────────────────────────────────────────────────────────────┐ │
│  │                     Virtual Host: /                          │ │
│  │  ┌────────────────┐                                          │ │
│  │  │   Exchanges    │                                          │ │
│  │  │  ┌──────────┐  │     Bindings      ┌──────────────────┐  │ │
│  │  │  │  direct  │──│─────────────────▶ │ order.queue      │  │ │
│  │  │  └──────────┘  │                   └──────────────────┘  │ │
│  │  │  ┌──────────┐  │     ┌──────────────────────────────┐   │ │
│  │  │  │  topic   │──│────▶│ notification.email.queue     │   │ │
│  │  │  └──────────┘  │     │ notification.sms.queue       │   │ │
│  │  │  ┌──────────┐  │     └──────────────────────────────┘   │ │
│  │  │  │  fanout  │──│────▶ [All bound queues]                │ │
│  │  │  └──────────┘  │                                         │ │
│  │  │  ┌──────────┐  │                                         │ │
│  │  │  │ headers  │  │                                         │ │
│  │  │  └──────────┘  │                                         │ │
│  │  └────────────────┘                                          │ │
│  └──────────────────────────────────────────────────────────────┘ │
│                                                                    │
│  ┌──────────────────────────────────────────────────────────────┐ │
│  │                   Virtual Host: /production                  │ │
│  │  [Isolated exchanges, queues, bindings]                      │ │
│  └──────────────────────────────────────────────────────────────┘ │
│                                                                    │
└────────────────────────────────────────────────────────────────────┘
```

### 1.3 Maven Dependencies

```xml
<dependencies>
    <!-- RabbitMQ Java Client -->
    <dependency>
        <groupId>com.rabbitmq</groupId>
        <artifactId>amqp-client</artifactId>
        <version>5.20.0</version>
    </dependency>

    <!-- Jackson for JSON serialization -->
    <dependency>
        <groupId>com.fasterxml.jackson.core</groupId>
        <artifactId>jackson-databind</artifactId>
        <version>2.17.0</version>
    </dependency>

    <!-- SLF4J Logging -->
    <dependency>
        <groupId>org.slf4j</groupId>
        <artifactId>slf4j-simple</artifactId>
        <version>2.0.12</version>
    </dependency>
</dependencies>
```

## 2. Exchange Types

### 2.1 Direct Exchange

```
Direct Exchange - Exact routing key matching
═══════════════════════════════════════════

Producer                    Exchange                    Queues
┌────────┐                 ┌────────┐                 ┌────────────┐
│        │  routing_key=   │        │  binding_key=  │order.create│
│ Order  │  "order.create" │ direct │  "order.create"│   Queue    │
│Service │ ───────────────▶│exchange│ ───────────────▶│            │
│        │                 │        │                 └────────────┘
│        │  routing_key=   │        │  binding_key=  ┌────────────┐
│        │  "order.cancel" │        │  "order.cancel"│order.cancel│
│        │ ───────────────▶│        │ ───────────────▶│   Queue    │
└────────┘                 └────────┘                 └────────────┘

Only exact matches route to queues
```

```java
public class DirectExchangeExample {

    private static final String EXCHANGE_NAME = "order.direct";
    private static final String QUEUE_CREATE = "order.create.queue";
    private static final String QUEUE_CANCEL = "order.cancel.queue";

    public static void main(String[] args) throws Exception {
        ConnectionFactory factory = new ConnectionFactory();
        factory.setHost("localhost");
        factory.setPort(5672);
        factory.setUsername("guest");
        factory.setPassword("guest");
        factory.setVirtualHost("/");

        try (Connection connection = factory.newConnection();
             Channel channel = connection.createChannel()) {

            // Declare direct exchange
            channel.exchangeDeclare(
                EXCHANGE_NAME,
                BuiltinExchangeType.DIRECT,
                true,   // durable
                false,  // auto-delete
                null    // arguments
            );

            // Declare queues
            channel.queueDeclare(QUEUE_CREATE, true, false, false, null);
            channel.queueDeclare(QUEUE_CANCEL, true, false, false, null);

            // Bind queues to exchange with routing keys
            channel.queueBind(QUEUE_CREATE, EXCHANGE_NAME, "order.create");
            channel.queueBind(QUEUE_CANCEL, EXCHANGE_NAME, "order.cancel");

            // Publish messages
            String createMessage = "{\"orderId\": \"123\", \"action\": \"create\"}";
            channel.basicPublish(
                EXCHANGE_NAME,
                "order.create",  // routing key
                MessageProperties.PERSISTENT_TEXT_PLAIN,
                createMessage.getBytes(StandardCharsets.UTF_8)
            );
            System.out.println("Sent to order.create: " + createMessage);

            String cancelMessage = "{\"orderId\": \"123\", \"action\": \"cancel\"}";
            channel.basicPublish(
                EXCHANGE_NAME,
                "order.cancel",
                MessageProperties.PERSISTENT_TEXT_PLAIN,
                cancelMessage.getBytes(StandardCharsets.UTF_8)
            );
            System.out.println("Sent to order.cancel: " + cancelMessage);
        }
    }
}
```

### 2.2 Topic Exchange

```
Topic Exchange - Pattern-based routing
══════════════════════════════════════

Routing Key Pattern:
  - *  matches exactly one word
  - #  matches zero or more words

Producer                    Exchange                    Queues
┌────────┐                 ┌────────┐
│        │  "order.us.     │        │  "order.*.    ┌─────────────┐
│ Order  │   created"      │ topic  │   created"    │ all.creates │
│Service │ ───────────────▶│exchange│ ─────────────▶│   Queue     │
│        │                 │        │               └─────────────┘
│        │  "order.eu.     │        │  "order.eu.#" ┌─────────────┐
│        │   cancelled"    │        │ ─────────────▶│ eu.orders   │
│        │ ───────────────▶│        │               │   Queue     │
└────────┘                 └────────┘               └─────────────┘

Examples:
  "order.us.created"    → matches "order.*.created", NOT "order.eu.#"
  "order.eu.cancelled"  → matches "order.eu.#"
  "order.eu.big.order"  → matches "order.eu.#" (# matches multiple words)
```

```java
public class TopicExchangeExample {

    private static final String EXCHANGE_NAME = "order.topic";

    public static void main(String[] args) throws Exception {
        ConnectionFactory factory = new ConnectionFactory();
        factory.setHost("localhost");

        try (Connection connection = factory.newConnection();
             Channel channel = connection.createChannel()) {

            // Declare topic exchange
            channel.exchangeDeclare(EXCHANGE_NAME, BuiltinExchangeType.TOPIC, true);

            // Queues for different patterns
            String allCreatesQueue = channel.queueDeclare().getQueue();
            String euOrdersQueue = channel.queueDeclare().getQueue();
            String allOrdersQueue = channel.queueDeclare().getQueue();

            // Bind with patterns
            channel.queueBind(allCreatesQueue, EXCHANGE_NAME, "order.*.created");
            channel.queueBind(euOrdersQueue, EXCHANGE_NAME, "order.eu.#");
            channel.queueBind(allOrdersQueue, EXCHANGE_NAME, "order.#");

            // Publish with different routing keys
            publishMessage(channel, "order.us.created", "US order created");
            publishMessage(channel, "order.eu.created", "EU order created");
            publishMessage(channel, "order.eu.cancelled", "EU order cancelled");
            publishMessage(channel, "order.asia.pending.review", "Asia order review");

            // Set up consumers
            setupConsumer(channel, allCreatesQueue, "AllCreates");
            setupConsumer(channel, euOrdersQueue, "EUOrders");
            setupConsumer(channel, allOrdersQueue, "AllOrders");

            Thread.sleep(2000);
        }
    }

    private static void publishMessage(Channel channel, String routingKey,
            String message) throws IOException {
        channel.basicPublish(
            EXCHANGE_NAME,
            routingKey,
            null,
            message.getBytes(StandardCharsets.UTF_8)
        );
        System.out.println("Published [" + routingKey + "]: " + message);
    }

    private static void setupConsumer(Channel channel, String queue,
            String consumerName) throws IOException {
        channel.basicConsume(queue, true, (tag, delivery) -> {
            String message = new String(delivery.getBody(), StandardCharsets.UTF_8);
            String routingKey = delivery.getEnvelope().getRoutingKey();
            System.out.println(consumerName + " received [" + routingKey + "]: " + message);
        }, tag -> {});
    }
}
```

### 2.3 Fanout Exchange

```
Fanout Exchange - Broadcast to all bound queues
═══════════════════════════════════════════════

Producer                    Exchange                    Queues
┌────────┐                 ┌────────┐                 ┌────────────┐
│        │                 │        │ ───────────────▶│  Queue A   │
│ Event  │  (routing key   │ fanout │                 └────────────┘
│ Source │   ignored)      │exchange│                 ┌────────────┐
│        │ ───────────────▶│        │ ───────────────▶│  Queue B   │
│        │                 │        │                 └────────────┘
│        │                 │        │                 ┌────────────┐
└────────┘                 └────────┘ ───────────────▶│  Queue C   │
                                                      └────────────┘

All bound queues receive the message (broadcast)
Routing key is ignored
```

```java
public class FanoutExchangeExample {

    private static final String EXCHANGE_NAME = "notifications.fanout";

    public static void main(String[] args) throws Exception {
        ConnectionFactory factory = new ConnectionFactory();
        factory.setHost("localhost");

        try (Connection connection = factory.newConnection();
             Channel channel = connection.createChannel()) {

            // Declare fanout exchange
            channel.exchangeDeclare(EXCHANGE_NAME, BuiltinExchangeType.FANOUT, true);

            // Multiple queues for different notification handlers
            String emailQueue = "notification.email";
            String smsQueue = "notification.sms";
            String pushQueue = "notification.push";
            String auditQueue = "notification.audit";

            // Declare and bind queues
            for (String queue : List.of(emailQueue, smsQueue, pushQueue, auditQueue)) {
                channel.queueDeclare(queue, true, false, false, null);
                channel.queueBind(queue, EXCHANGE_NAME, ""); // routing key ignored
            }

            // Publish notification - will be sent to ALL queues
            NotificationEvent event = new NotificationEvent(
                "user-123",
                "Your order has been shipped!",
                Instant.now()
            );

            ObjectMapper mapper = new ObjectMapper();
            mapper.registerModule(new JavaTimeModule());

            byte[] messageBody = mapper.writeValueAsBytes(event);

            AMQP.BasicProperties props = new AMQP.BasicProperties.Builder()
                .contentType("application/json")
                .deliveryMode(2) // persistent
                .timestamp(new Date())
                .build();

            channel.basicPublish(EXCHANGE_NAME, "", props, messageBody);
            System.out.println("Broadcast notification sent to all handlers");
        }
    }

    @Data
    @AllArgsConstructor
    static class NotificationEvent {
        private String userId;
        private String message;
        private Instant timestamp;
    }
}
```

### 2.4 Headers Exchange

```
Headers Exchange - Route by message headers
═══════════════════════════════════════════

Producer                    Exchange                    Queues
┌────────┐                 ┌────────┐
│        │  headers:       │        │  x-match: all  ┌─────────────┐
│ Report │  type=pdf       │headers │  type=pdf      │ pdf.reports │
│Service │  priority=high  │exchange│  priority=high │   Queue     │
│        │ ───────────────▶│        │ ───────────────▶└─────────────┘
│        │                 │        │
│        │  headers:       │        │  x-match: any  ┌─────────────┐
│        │  type=excel     │        │  type=excel    │ spreadsheet │
│        │  format=xlsx    │        │  OR            │   Queue     │
│        │ ───────────────▶│        │  format=xlsx  │             │
└────────┘                 └────────┘ ───────────────▶└─────────────┘

x-match: all  → ALL headers must match
x-match: any  → ANY header must match
```

```java
public class HeadersExchangeExample {

    private static final String EXCHANGE_NAME = "reports.headers";

    public static void main(String[] args) throws Exception {
        ConnectionFactory factory = new ConnectionFactory();
        factory.setHost("localhost");

        try (Connection connection = factory.newConnection();
             Channel channel = connection.createChannel()) {

            // Declare headers exchange
            channel.exchangeDeclare(EXCHANGE_NAME, BuiltinExchangeType.HEADERS, true);

            // Queue for high priority PDF reports
            String highPriorityPdfQueue = "reports.pdf.high";
            channel.queueDeclare(highPriorityPdfQueue, true, false, false, null);

            Map<String, Object> pdfHighHeaders = new HashMap<>();
            pdfHighHeaders.put("x-match", "all"); // All headers must match
            pdfHighHeaders.put("type", "pdf");
            pdfHighHeaders.put("priority", "high");

            channel.queueBind(highPriorityPdfQueue, EXCHANGE_NAME, "", pdfHighHeaders);

            // Queue for any Excel reports
            String excelQueue = "reports.excel";
            channel.queueDeclare(excelQueue, true, false, false, null);

            Map<String, Object> excelHeaders = new HashMap<>();
            excelHeaders.put("x-match", "any"); // Any header can match
            excelHeaders.put("type", "excel");
            excelHeaders.put("format", "xlsx");

            channel.queueBind(excelQueue, EXCHANGE_NAME, "", excelHeaders);

            // Publish messages with different headers
            publishWithHeaders(channel, Map.of("type", "pdf", "priority", "high"),
                "High priority PDF report");
            publishWithHeaders(channel, Map.of("type", "pdf", "priority", "low"),
                "Low priority PDF report");
            publishWithHeaders(channel, Map.of("type", "excel"),
                "Excel report");
            publishWithHeaders(channel, Map.of("format", "xlsx"),
                "XLSX format report");
        }
    }

    private static void publishWithHeaders(Channel channel,
            Map<String, Object> headers, String message) throws IOException {
        AMQP.BasicProperties props = new AMQP.BasicProperties.Builder()
            .headers(headers)
            .contentType("text/plain")
            .build();

        channel.basicPublish(EXCHANGE_NAME, "", props,
            message.getBytes(StandardCharsets.UTF_8));
        System.out.println("Published with headers " + headers + ": " + message);
    }
}
```

## 3. Producer Implementation

### 3.1 Connection Management

```java
@Slf4j
public class RabbitMQConnectionManager implements AutoCloseable {

    private final Connection connection;
    private final ConcurrentMap<Long, Channel> channelPool = new ConcurrentHashMap<>();

    public RabbitMQConnectionManager(RabbitMQConfig config) throws Exception {
        ConnectionFactory factory = new ConnectionFactory();
        factory.setHost(config.getHost());
        factory.setPort(config.getPort());
        factory.setUsername(config.getUsername());
        factory.setPassword(config.getPassword());
        factory.setVirtualHost(config.getVirtualHost());

        // Connection recovery
        factory.setAutomaticRecoveryEnabled(true);
        factory.setNetworkRecoveryInterval(5000);
        factory.setTopologyRecoveryEnabled(true);

        // Connection timeout
        factory.setConnectionTimeout(30000);
        factory.setHandshakeTimeout(10000);

        // Heartbeat for connection health
        factory.setRequestedHeartbeat(60);

        // Thread pool for consumers
        ExecutorService executor = Executors.newFixedThreadPool(
            config.getConsumerThreads(),
            new ThreadFactoryBuilder()
                .setNameFormat("rabbitmq-consumer-%d")
                .build()
        );

        this.connection = factory.newConnection(executor, config.getConnectionName());

        // Add shutdown listener
        this.connection.addShutdownListener(cause -> {
            if (cause.isInitiatedByApplication()) {
                log.info("Connection closed by application");
            } else {
                log.error("Connection closed unexpectedly: {}", cause.getMessage());
            }
        });

        log.info("Connected to RabbitMQ: {}:{}", config.getHost(), config.getPort());
    }

    public Channel getChannel() throws IOException {
        long threadId = Thread.currentThread().getId();
        return channelPool.computeIfAbsent(threadId, id -> {
            try {
                Channel channel = connection.createChannel();
                channel.addShutdownListener(cause -> {
                    if (!cause.isInitiatedByApplication()) {
                        log.warn("Channel closed: {}", cause.getMessage());
                        channelPool.remove(threadId);
                    }
                });
                return channel;
            } catch (IOException e) {
                throw new UncheckedIOException(e);
            }
        });
    }

    @Override
    public void close() throws Exception {
        channelPool.values().forEach(channel -> {
            try {
                channel.close();
            } catch (Exception e) {
                log.warn("Error closing channel", e);
            }
        });
        connection.close();
    }
}

@Data
@Builder
public class RabbitMQConfig {
    private String host;
    private int port;
    private String username;
    private String password;
    private String virtualHost;
    private String connectionName;
    private int consumerThreads;
}
```

### 3.2 Reliable Producer

```java
@Slf4j
public class ReliableProducer {

    private final Channel channel;
    private final ObjectMapper objectMapper;
    private final String exchangeName;

    // For publisher confirms
    private final ConcurrentNavigableMap<Long, CompletableFuture<Boolean>>
        outstandingConfirms = new ConcurrentSkipListMap<>();

    public ReliableProducer(Channel channel, String exchangeName) throws IOException {
        this.channel = channel;
        this.exchangeName = exchangeName;
        this.objectMapper = new ObjectMapper()
            .registerModule(new JavaTimeModule())
            .configure(DeserializationFeature.FAIL_ON_UNKNOWN_PROPERTIES, false);

        setupPublisherConfirms();
    }

    private void setupPublisherConfirms() throws IOException {
        // Enable publisher confirms
        channel.confirmSelect();

        // Async confirm handler
        channel.addConfirmListener(new ConfirmListener() {
            @Override
            public void handleAck(long deliveryTag, boolean multiple) {
                if (multiple) {
                    ConcurrentNavigableMap<Long, CompletableFuture<Boolean>> confirmed =
                        outstandingConfirms.headMap(deliveryTag, true);
                    confirmed.values().forEach(f -> f.complete(true));
                    confirmed.clear();
                } else {
                    CompletableFuture<Boolean> future =
                        outstandingConfirms.remove(deliveryTag);
                    if (future != null) {
                        future.complete(true);
                    }
                }
                log.debug("Message confirmed: tag={}, multiple={}", deliveryTag, multiple);
            }

            @Override
            public void handleNack(long deliveryTag, boolean multiple) {
                if (multiple) {
                    ConcurrentNavigableMap<Long, CompletableFuture<Boolean>> rejected =
                        outstandingConfirms.headMap(deliveryTag, true);
                    rejected.values().forEach(f -> f.complete(false));
                    rejected.clear();
                } else {
                    CompletableFuture<Boolean> future =
                        outstandingConfirms.remove(deliveryTag);
                    if (future != null) {
                        future.complete(false);
                    }
                }
                log.warn("Message rejected: tag={}, multiple={}", deliveryTag, multiple);
            }
        });

        // Return listener for unroutable messages
        channel.addReturnListener((replyCode, replyText, exchange, routingKey,
                properties, body) -> {
            log.error("Message returned: exchange={}, routingKey={}, reason={}",
                exchange, routingKey, replyText);
        });
    }

    public <T> CompletableFuture<Boolean> publish(String routingKey, T message) {
        return publish(routingKey, message, MessageOptions.defaults());
    }

    public <T> CompletableFuture<Boolean> publish(String routingKey, T message,
            MessageOptions options) {
        CompletableFuture<Boolean> future = new CompletableFuture<>();

        try {
            byte[] body = objectMapper.writeValueAsBytes(message);

            AMQP.BasicProperties props = new AMQP.BasicProperties.Builder()
                .contentType("application/json")
                .deliveryMode(options.isPersistent() ? 2 : 1)
                .priority(options.getPriority())
                .messageId(options.getMessageId() != null ?
                    options.getMessageId() : UUID.randomUUID().toString())
                .timestamp(new Date())
                .expiration(options.getTtlMillis() != null ?
                    String.valueOf(options.getTtlMillis()) : null)
                .headers(options.getHeaders())
                .correlationId(options.getCorrelationId())
                .replyTo(options.getReplyTo())
                .build();

            long sequenceNumber = channel.getNextPublishSeqNo();
            outstandingConfirms.put(sequenceNumber, future);

            // mandatory=true returns message if not routable
            channel.basicPublish(exchangeName, routingKey, options.isMandatory(),
                props, body);

            log.debug("Published message: exchange={}, routingKey={}, seqNo={}",
                exchangeName, routingKey, sequenceNumber);

        } catch (Exception e) {
            future.completeExceptionally(e);
        }

        return future;
    }

    public <T> boolean publishSync(String routingKey, T message, Duration timeout)
            throws Exception {
        CompletableFuture<Boolean> future = publish(routingKey, message);
        return future.get(timeout.toMillis(), TimeUnit.MILLISECONDS);
    }

    @Data
    @Builder
    public static class MessageOptions {
        @Builder.Default
        private boolean persistent = true;
        @Builder.Default
        private boolean mandatory = false;
        @Builder.Default
        private int priority = 0;
        private String messageId;
        private Long ttlMillis;
        private Map<String, Object> headers;
        private String correlationId;
        private String replyTo;

        public static MessageOptions defaults() {
            return MessageOptions.builder().build();
        }
    }
}
```

### 3.3 Batch Publishing

```java
@Slf4j
public class BatchProducer {

    private final Channel channel;
    private final ObjectMapper objectMapper;
    private final String exchangeName;
    private final int batchSize;

    public BatchProducer(Channel channel, String exchangeName, int batchSize) {
        this.channel = channel;
        this.exchangeName = exchangeName;
        this.batchSize = batchSize;
        this.objectMapper = new ObjectMapper().registerModule(new JavaTimeModule());
    }

    public <T> BatchResult publishBatch(String routingKey, List<T> messages)
            throws Exception {
        channel.confirmSelect();

        int successCount = 0;
        int failCount = 0;
        List<String> errors = new ArrayList<>();

        for (int i = 0; i < messages.size(); i += batchSize) {
            List<T> batch = messages.subList(i,
                Math.min(i + batchSize, messages.size()));

            try {
                for (T message : batch) {
                    byte[] body = objectMapper.writeValueAsBytes(message);
                    AMQP.BasicProperties props = new AMQP.BasicProperties.Builder()
                        .contentType("application/json")
                        .deliveryMode(2)
                        .build();

                    channel.basicPublish(exchangeName, routingKey, props, body);
                }

                // Wait for confirms for this batch
                boolean confirmed = channel.waitForConfirms(5000);

                if (confirmed) {
                    successCount += batch.size();
                } else {
                    failCount += batch.size();
                    errors.add("Batch " + (i / batchSize) + " not confirmed");
                }

            } catch (Exception e) {
                failCount += batch.size();
                errors.add("Batch " + (i / batchSize) + " failed: " + e.getMessage());
            }
        }

        return new BatchResult(successCount, failCount, errors);
    }

    @Data
    @AllArgsConstructor
    public static class BatchResult {
        private int successCount;
        private int failCount;
        private List<String> errors;

        public boolean isAllSuccess() {
            return failCount == 0;
        }
    }
}
```

## 4. Consumer Implementation

### 4.1 Basic Consumer

```java
@Slf4j
public class BasicConsumer {

    private final Channel channel;
    private final ObjectMapper objectMapper;

    public BasicConsumer(Channel channel) {
        this.channel = channel;
        this.objectMapper = new ObjectMapper().registerModule(new JavaTimeModule());
    }

    public <T> String consume(String queueName, Class<T> messageType,
            MessageHandler<T> handler) throws IOException {

        // QoS - prefetch count
        channel.basicQos(10); // Process 10 messages at a time

        DeliverCallback deliverCallback = (consumerTag, delivery) -> {
            String messageId = delivery.getProperties().getMessageId();
            long deliveryTag = delivery.getEnvelope().getDeliveryTag();

            try {
                T message = objectMapper.readValue(delivery.getBody(), messageType);

                log.debug("Received message: id={}, tag={}", messageId, deliveryTag);

                // Process message
                handler.handle(message, new MessageContext(
                    delivery.getEnvelope(),
                    delivery.getProperties()
                ));

                // Manual acknowledgment
                channel.basicAck(deliveryTag, false);
                log.debug("Message acknowledged: tag={}", deliveryTag);

            } catch (Exception e) {
                log.error("Error processing message: id={}", messageId, e);

                // Reject and requeue or send to DLQ
                boolean requeue = !delivery.getEnvelope().isRedeliver();
                channel.basicNack(deliveryTag, false, requeue);

                if (!requeue) {
                    log.warn("Message sent to DLQ: id={}", messageId);
                }
            }
        };

        CancelCallback cancelCallback = consumerTag -> {
            log.info("Consumer cancelled: {}", consumerTag);
        };

        // Start consuming with manual ack
        return channel.basicConsume(
            queueName,
            false,  // autoAck = false
            deliverCallback,
            cancelCallback
        );
    }

    @FunctionalInterface
    public interface MessageHandler<T> {
        void handle(T message, MessageContext context) throws Exception;
    }

    @Data
    @AllArgsConstructor
    public static class MessageContext {
        private Envelope envelope;
        private AMQP.BasicProperties properties;

        public String getRoutingKey() {
            return envelope.getRoutingKey();
        }

        public String getExchange() {
            return envelope.getExchange();
        }

        public boolean isRedelivered() {
            return envelope.isRedeliver();
        }
    }
}
```

### 4.2 Consumer with Retry

```java
@Slf4j
public class RetryableConsumer {

    private final Channel channel;
    private final ObjectMapper objectMapper;
    private final int maxRetries;
    private final String dlqExchange;

    public RetryableConsumer(Channel channel, int maxRetries, String dlqExchange) {
        this.channel = channel;
        this.objectMapper = new ObjectMapper().registerModule(new JavaTimeModule());
        this.maxRetries = maxRetries;
        this.dlqExchange = dlqExchange;
    }

    public <T> void consumeWithRetry(String queueName, Class<T> messageType,
            MessageHandler<T> handler) throws IOException {

        channel.basicQos(5);

        channel.basicConsume(queueName, false, (consumerTag, delivery) -> {
            long deliveryTag = delivery.getEnvelope().getDeliveryTag();
            AMQP.BasicProperties props = delivery.getProperties();

            // Get retry count from headers
            Map<String, Object> headers = props.getHeaders();
            int retryCount = 0;
            if (headers != null && headers.containsKey("x-retry-count")) {
                retryCount = ((Number) headers.get("x-retry-count")).intValue();
            }

            try {
                T message = objectMapper.readValue(delivery.getBody(), messageType);
                handler.handle(message);
                channel.basicAck(deliveryTag, false);

            } catch (RetryableException e) {
                log.warn("Retryable error (attempt {}): {}", retryCount + 1, e.getMessage());
                handleRetry(delivery, retryCount, queueName);
                channel.basicAck(deliveryTag, false);

            } catch (NonRetryableException e) {
                log.error("Non-retryable error, sending to DLQ: {}", e.getMessage());
                sendToDlq(delivery, e.getMessage());
                channel.basicAck(deliveryTag, false);

            } catch (Exception e) {
                log.error("Unexpected error: {}", e.getMessage(), e);
                handleRetry(delivery, retryCount, queueName);
                channel.basicAck(deliveryTag, false);
            }
        }, consumerTag -> {});
    }

    private void handleRetry(Delivery delivery, int currentRetry,
            String originalQueue) throws IOException {
        if (currentRetry >= maxRetries) {
            log.warn("Max retries exceeded, sending to DLQ");
            sendToDlq(delivery, "Max retries exceeded: " + maxRetries);
            return;
        }

        // Exponential backoff delay
        long delay = (long) Math.pow(2, currentRetry) * 1000;

        Map<String, Object> newHeaders = new HashMap<>();
        if (delivery.getProperties().getHeaders() != null) {
            newHeaders.putAll(delivery.getProperties().getHeaders());
        }
        newHeaders.put("x-retry-count", currentRetry + 1);
        newHeaders.put("x-original-queue", originalQueue);
        newHeaders.put("x-first-failure-time",
            newHeaders.getOrDefault("x-first-failure-time", System.currentTimeMillis()));

        AMQP.BasicProperties props = new AMQP.BasicProperties.Builder()
            .contentType(delivery.getProperties().getContentType())
            .deliveryMode(2)
            .headers(newHeaders)
            .expiration(String.valueOf(delay)) // Message TTL for delay
            .build();

        // Publish to delay queue (with TTL and DLX back to original queue)
        String delayQueue = originalQueue + ".delay";
        channel.basicPublish("", delayQueue, props, delivery.getBody());

        log.info("Message scheduled for retry {} in {}ms", currentRetry + 1, delay);
    }

    private void sendToDlq(Delivery delivery, String reason) throws IOException {
        Map<String, Object> headers = new HashMap<>();
        if (delivery.getProperties().getHeaders() != null) {
            headers.putAll(delivery.getProperties().getHeaders());
        }
        headers.put("x-dlq-reason", reason);
        headers.put("x-dlq-time", System.currentTimeMillis());
        headers.put("x-original-routing-key", delivery.getEnvelope().getRoutingKey());

        AMQP.BasicProperties props = new AMQP.BasicProperties.Builder()
            .headers(headers)
            .contentType(delivery.getProperties().getContentType())
            .deliveryMode(2)
            .build();

        channel.basicPublish(dlqExchange, "dlq", props, delivery.getBody());
    }

    public static class RetryableException extends RuntimeException {
        public RetryableException(String message) { super(message); }
        public RetryableException(String message, Throwable cause) { super(message, cause); }
    }

    public static class NonRetryableException extends RuntimeException {
        public NonRetryableException(String message) { super(message); }
    }

    @FunctionalInterface
    public interface MessageHandler<T> {
        void handle(T message) throws Exception;
    }
}
```

### 4.3 Queue Configuration with DLQ

```java
public class QueueSetup {

    public static void setupQueueWithDlq(Channel channel, String queueName)
            throws IOException {

        // Dead Letter Queue
        String dlqName = queueName + ".dlq";
        String dlxName = queueName + ".dlx";

        // Declare DLX (Dead Letter Exchange)
        channel.exchangeDeclare(dlxName, BuiltinExchangeType.DIRECT, true);

        // Declare DLQ
        channel.queueDeclare(dlqName, true, false, false, null);
        channel.queueBind(dlqName, dlxName, queueName);

        // Main queue with DLX configuration
        Map<String, Object> mainQueueArgs = new HashMap<>();
        mainQueueArgs.put("x-dead-letter-exchange", dlxName);
        mainQueueArgs.put("x-dead-letter-routing-key", queueName);
        // Optional: message TTL
        // mainQueueArgs.put("x-message-ttl", 60000);
        // Optional: max length
        // mainQueueArgs.put("x-max-length", 10000);
        // Optional: max length behavior
        // mainQueueArgs.put("x-overflow", "reject-publish"); // or "drop-head"

        channel.queueDeclare(queueName, true, false, false, mainQueueArgs);

        // Delay queue for retries
        String delayQueue = queueName + ".delay";
        Map<String, Object> delayQueueArgs = new HashMap<>();
        delayQueueArgs.put("x-dead-letter-exchange", ""); // Default exchange
        delayQueueArgs.put("x-dead-letter-routing-key", queueName);

        channel.queueDeclare(delayQueue, true, false, false, delayQueueArgs);
    }

    public static void setupPriorityQueue(Channel channel, String queueName,
            int maxPriority) throws IOException {
        Map<String, Object> args = new HashMap<>();
        args.put("x-max-priority", maxPriority);

        channel.queueDeclare(queueName, true, false, false, args);
    }

    public static void setupQuorumQueue(Channel channel, String queueName)
            throws IOException {
        // Quorum queues for high availability (RabbitMQ 3.8+)
        Map<String, Object> args = new HashMap<>();
        args.put("x-queue-type", "quorum");
        args.put("x-quorum-initial-group-size", 3);

        channel.queueDeclare(queueName, true, false, false, args);
    }
}
```

## 5. RabbitMQ vs Kafka

### 5.1 Comparison Matrix

```
┌─────────────────────┬─────────────────────────┬─────────────────────────┐
│      Feature        │       RabbitMQ          │        Kafka            │
├─────────────────────┼─────────────────────────┼─────────────────────────┤
│ Protocol            │ AMQP 0-9-1              │ Custom binary protocol  │
├─────────────────────┼─────────────────────────┼─────────────────────────┤
│ Message Routing     │ Complex (exchanges,     │ Simple (topic-based)    │
│                     │ bindings, routing keys) │                         │
├─────────────────────┼─────────────────────────┼─────────────────────────┤
│ Message Ordering    │ Per-queue               │ Per-partition           │
├─────────────────────┼─────────────────────────┼─────────────────────────┤
│ Delivery Semantics  │ At-most-once,           │ At-least-once,          │
│                     │ At-least-once           │ Exactly-once (with TX)  │
├─────────────────────┼─────────────────────────┼─────────────────────────┤
│ Message Retention   │ Until consumed          │ Configurable time/size  │
│                     │ (acknowledgment-based)  │ (log-based)             │
├─────────────────────┼─────────────────────────┼─────────────────────────┤
│ Consumer Model      │ Push (broker pushes)    │ Pull (consumer polls)   │
├─────────────────────┼─────────────────────────┼─────────────────────────┤
│ Replay Capability   │ No (message deleted     │ Yes (offset-based)      │
│                     │ after ack)              │                         │
├─────────────────────┼─────────────────────────┼─────────────────────────┤
│ Throughput          │ Moderate (~50k msg/s)   │ High (~1M+ msg/s)       │
├─────────────────────┼─────────────────────────┼─────────────────────────┤
│ Latency             │ Very low (μs)           │ Low (ms)                │
├─────────────────────┼─────────────────────────┼─────────────────────────┤
│ Use Cases           │ Task queues, RPC,       │ Event streaming, logs,  │
│                     │ complex routing         │ data pipelines          │
└─────────────────────┴─────────────────────────┴─────────────────────────┘
```

### 5.2 When to Use RabbitMQ

```java
/*
 * Use RabbitMQ when you need:
 *
 * 1. Complex Routing
 *    - Multiple exchange types
 *    - Routing key patterns
 *    - Header-based routing
 *
 * 2. Traditional Message Queue Patterns
 *    - Work queues (competing consumers)
 *    - Request/Reply (RPC)
 *    - Publish/Subscribe
 *
 * 3. Message Priority
 *    - Priority queues supported natively
 *    - Consumers can set prefetch
 *
 * 4. Low Latency Requirements
 *    - Push-based delivery is faster for small volumes
 *    - Better for real-time applications
 *
 * 5. Message-Level Control
 *    - Per-message TTL
 *    - Individual message acknowledgment
 *    - Message rejection and requeue
 */

// Example: Request/Reply Pattern (RPC)
public class RpcClient {

    private final Channel channel;
    private final String requestQueue;
    private final String replyQueue;
    private final Map<String, CompletableFuture<String>> pendingRequests;

    public RpcClient(Channel channel, String requestQueue) throws IOException {
        this.channel = channel;
        this.requestQueue = requestQueue;
        this.pendingRequests = new ConcurrentHashMap<>();

        // Create exclusive reply queue
        this.replyQueue = channel.queueDeclare().getQueue();

        // Consume replies
        channel.basicConsume(replyQueue, true, (tag, delivery) -> {
            String correlationId = delivery.getProperties().getCorrelationId();
            CompletableFuture<String> future = pendingRequests.remove(correlationId);
            if (future != null) {
                future.complete(new String(delivery.getBody(), StandardCharsets.UTF_8));
            }
        }, tag -> {});
    }

    public CompletableFuture<String> call(String message) throws IOException {
        String correlationId = UUID.randomUUID().toString();
        CompletableFuture<String> future = new CompletableFuture<>();
        pendingRequests.put(correlationId, future);

        AMQP.BasicProperties props = new AMQP.BasicProperties.Builder()
            .correlationId(correlationId)
            .replyTo(replyQueue)
            .build();

        channel.basicPublish("", requestQueue, props,
            message.getBytes(StandardCharsets.UTF_8));

        return future;
    }
}
```

### 5.3 When to Use Kafka

```java
/*
 * Use Kafka when you need:
 *
 * 1. High Throughput
 *    - Millions of messages per second
 *    - Horizontal scaling with partitions
 *
 * 2. Event Sourcing / Event Log
 *    - Messages retained after consumption
 *    - Replay from any offset
 *
 * 3. Stream Processing
 *    - Kafka Streams for real-time analytics
 *    - Windowing, aggregations, joins
 *
 * 4. Multiple Consumer Groups
 *    - Each group processes independently
 *    - Different read positions per group
 *
 * 5. Data Pipeline / ETL
 *    - Connect sources to sinks
 *    - Schema evolution with Schema Registry
 */

// Example: Multi-Consumer Event Log
// Same topic can be consumed by multiple independent consumers
// without message duplication concerns
```

## 6. Complete Working Example

### 6.1 Order Processing System

```java
// Domain Models
@Data
@Builder
public class Order {
    private String orderId;
    private String customerId;
    private List<OrderItem> items;
    private BigDecimal totalAmount;
    private OrderStatus status;
    private Instant createdAt;
}

@Data
@Builder
public class OrderItem {
    private String productId;
    private int quantity;
    private BigDecimal price;
}

public enum OrderStatus {
    CREATED, VALIDATED, PAID, SHIPPED, DELIVERED, CANCELLED
}

// Order Event
@Data
@Builder
public class OrderEvent {
    private String eventId;
    private String eventType;
    private Order order;
    private Instant timestamp;

    public static OrderEvent created(Order order) {
        return OrderEvent.builder()
            .eventId(UUID.randomUUID().toString())
            .eventType("ORDER_CREATED")
            .order(order)
            .timestamp(Instant.now())
            .build();
    }
}

// Order Service - Producer
@Slf4j
public class OrderService {

    private final ReliableProducer producer;

    public OrderService(Channel channel) throws IOException {
        this.producer = new ReliableProducer(channel, "orders.topic");
    }

    public CompletableFuture<Boolean> createOrder(Order order) {
        order.setOrderId(UUID.randomUUID().toString());
        order.setStatus(OrderStatus.CREATED);
        order.setCreatedAt(Instant.now());

        OrderEvent event = OrderEvent.created(order);

        // Route based on order amount (priority routing)
        String routingKey = order.getTotalAmount().compareTo(new BigDecimal("1000")) > 0
            ? "order.high-value.created"
            : "order.standard.created";

        return producer.publish(routingKey, event,
            ReliableProducer.MessageOptions.builder()
                .persistent(true)
                .priority(order.getTotalAmount().compareTo(new BigDecimal("1000")) > 0 ? 5 : 0)
                .messageId(event.getEventId())
                .build()
        );
    }
}

// Validation Service - Consumer
@Slf4j
public class OrderValidationService {

    private final Channel channel;
    private final BasicConsumer consumer;

    public OrderValidationService(Channel channel) {
        this.channel = channel;
        this.consumer = new BasicConsumer(channel);
    }

    public void start() throws IOException {
        consumer.consume("order.validation.queue", OrderEvent.class,
            (event, context) -> {
                log.info("Validating order: {}", event.getOrder().getOrderId());

                // Validate order
                validateOrder(event.getOrder());

                // Publish validation result
                publishValidationResult(event.getOrder());
            });

        log.info("Order validation service started");
    }

    private void validateOrder(Order order) {
        // Business validation logic
        if (order.getItems() == null || order.getItems().isEmpty()) {
            throw new RetryableConsumer.NonRetryableException("Order has no items");
        }

        if (order.getTotalAmount().compareTo(BigDecimal.ZERO) <= 0) {
            throw new RetryableConsumer.NonRetryableException("Invalid total amount");
        }
    }

    private void publishValidationResult(Order order) throws IOException {
        order.setStatus(OrderStatus.VALIDATED);

        AMQP.BasicProperties props = new AMQP.BasicProperties.Builder()
            .contentType("application/json")
            .deliveryMode(2)
            .build();

        ObjectMapper mapper = new ObjectMapper().registerModule(new JavaTimeModule());

        channel.basicPublish("orders.topic", "order.validated",
            props, mapper.writeValueAsBytes(OrderEvent.builder()
                .eventId(UUID.randomUUID().toString())
                .eventType("ORDER_VALIDATED")
                .order(order)
                .timestamp(Instant.now())
                .build()));
    }
}

// Application Bootstrap
public class OrderApplication {

    public static void main(String[] args) throws Exception {
        RabbitMQConfig config = RabbitMQConfig.builder()
            .host("localhost")
            .port(5672)
            .username("guest")
            .password("guest")
            .virtualHost("/")
            .connectionName("order-service")
            .consumerThreads(4)
            .build();

        try (RabbitMQConnectionManager connectionManager =
                new RabbitMQConnectionManager(config)) {

            Channel channel = connectionManager.getChannel();

            // Setup infrastructure
            setupInfrastructure(channel);

            // Start services
            OrderService orderService = new OrderService(channel);
            OrderValidationService validationService =
                new OrderValidationService(channel);

            validationService.start();

            // Create test order
            Order order = Order.builder()
                .customerId("cust-123")
                .items(List.of(
                    OrderItem.builder()
                        .productId("prod-1")
                        .quantity(2)
                        .price(new BigDecimal("99.99"))
                        .build()
                ))
                .totalAmount(new BigDecimal("199.98"))
                .build();

            orderService.createOrder(order)
                .thenAccept(confirmed -> {
                    if (confirmed) {
                        System.out.println("Order published successfully");
                    }
                });

            // Keep running
            Thread.sleep(10000);
        }
    }

    private static void setupInfrastructure(Channel channel) throws IOException {
        // Topic exchange for orders
        channel.exchangeDeclare("orders.topic", BuiltinExchangeType.TOPIC, true);

        // Validation queue
        QueueSetup.setupQueueWithDlq(channel, "order.validation.queue");
        channel.queueBind("order.validation.queue", "orders.topic", "order.*.created");

        // Payment queue (for high-value orders)
        QueueSetup.setupQueueWithDlq(channel, "order.payment.queue");
        channel.queueBind("order.payment.queue", "orders.topic", "order.validated");

        // Notification queue (all events)
        channel.queueDeclare("order.notification.queue", true, false, false, null);
        channel.queueBind("order.notification.queue", "orders.topic", "order.#");
    }
}
```

## 7. Hands-on Exercises

### Exercise 1: Work Queue with Fair Dispatch
Implement a work queue pattern where multiple workers process tasks fairly (round-robin).

```java
// Requirements:
// 1. Create a durable queue "tasks"
// 2. Producer sends 100 tasks with varying processing times
// 3. Start 3 workers with prefetch=1
// 4. Each worker should process tasks one at a time
// 5. Verify fair distribution
```

### Exercise 2: Pub/Sub with Filtering
Build a notification system where subscribers can filter messages.

```java
// Requirements:
// 1. Use topic exchange "notifications"
// 2. Publishers send: notification.{type}.{priority}
//    Types: email, sms, push
//    Priority: high, normal, low
// 3. Subscribers:
//    - All high priority: "notification.*.high"
//    - All email: "notification.email.*"
//    - Everything: "notification.#"
```

### Exercise 3: Request/Reply Calculator
Implement an RPC calculator service.

```java
// Requirements:
// 1. Calculator server processes: add, subtract, multiply, divide
// 2. Client sends request with correlationId
// 3. Server responds to replyTo queue
// 4. Handle division by zero gracefully
// 5. Add timeout handling on client side
```

### Exercise 4: Dead Letter Processing
Create a system with retry and DLQ handling.

```java
// Requirements:
// 1. Main queue with 3 retries
// 2. Exponential backoff (1s, 2s, 4s)
// 3. After max retries, send to DLQ
// 4. Create a DLQ processor that logs failed messages
// 5. Add ability to replay from DLQ
```

## 8. Key Takeaways

### Core Concepts
1. **AMQP Model**: Producer → Exchange → Queue → Consumer
2. **Exchange Types**: Direct (exact), Topic (pattern), Fanout (broadcast), Headers
3. **Message Flow**: Bindings connect exchanges to queues with routing rules
4. **Acknowledgments**: Manual ack ensures message processed before removal

### Best Practices
1. **Always use publisher confirms** for reliable delivery
2. **Set appropriate prefetch** to balance throughput and fairness
3. **Configure DLQ** for failed message handling
4. **Use durable queues and persistent messages** for important data
5. **Implement idempotency** in consumers

### Common Patterns
- **Work Queue**: Competing consumers for load distribution
- **Pub/Sub**: Fanout exchange for broadcasting
- **Routing**: Direct/Topic exchange for selective delivery
- **RPC**: Request/Reply with correlation IDs

---

## Navigation

[← Day 15: Kafka Operations](./day-15-kafka-operations.md) | [Day 17: Spring AMQP →](./day-17-spring-amqp.md)

[Back to Overview](./00-overview.md)
