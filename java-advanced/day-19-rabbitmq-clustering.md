# Day 19: RabbitMQ Clustering & High Availability

## Mục tiêu học tập
- Hiểu RabbitMQ clustering architecture
- Cấu hình Quorum Queues cho high availability
- Implement connection failover trong Spring AMQP
- Triển khai RabbitMQ cluster với Docker
- Monitor và troubleshoot cluster issues

## 1. RabbitMQ Cluster Architecture

### 1.1 Cluster Overview

```
RabbitMQ Cluster Architecture
═════════════════════════════

┌────────────────────────────────────────────────────────────────┐
│                     RabbitMQ Cluster                           │
├────────────────────────────────────────────────────────────────┤
│                                                                │
│  ┌────────────┐    ┌────────────┐    ┌────────────┐          │
│  │   Node 1   │    │   Node 2   │    │   Node 3   │          │
│  │  (Master)  │◄──▶│  (Mirror)  │◄──▶│  (Mirror)  │          │
│  │            │    │            │    │            │          │
│  │ ┌────────┐ │    │ ┌────────┐ │    │ ┌────────┐ │          │
│  │ │ Queue  │ │    │ │ Mirror │ │    │ │ Mirror │ │          │
│  │ │ Master │ │    │ │  Copy  │ │    │ │  Copy  │ │          │
│  │ └────────┘ │    │ └────────┘ │    │ └────────┘ │          │
│  └────────────┘    └────────────┘    └────────────┘          │
│                                                                │
│  Shared across all nodes:                                      │
│  - Virtual hosts                                               │
│  - Exchanges                                                   │
│  - Bindings                                                    │
│  - Users/Permissions                                           │
│  - Policies                                                    │
│                                                                │
│  NOT shared (unless mirrored/quorum):                         │
│  - Queue contents (replicated for HA queues)                  │
│                                                                │
└────────────────────────────────────────────────────────────────┘

Node Communication:
  - Erlang distribution protocol (port 25672)
  - Uses Erlang cookie for authentication
  - Full mesh topology (all nodes connected)
```

### 1.2 Queue Types Comparison

```
Queue Types for High Availability
═════════════════════════════════

┌─────────────────┬──────────────────┬──────────────────────────┐
│     Feature     │  Mirrored Queue  │     Quorum Queue         │
│                 │  (Classic HA)    │  (RabbitMQ 3.8+)         │
├─────────────────┼──────────────────┼──────────────────────────┤
│ Consensus       │ None (async)     │ Raft protocol            │
├─────────────────┼──────────────────┼──────────────────────────┤
│ Data Safety     │ Best effort      │ Strong consistency       │
├─────────────────┼──────────────────┼──────────────────────────┤
│ Performance     │ Higher           │ Slightly lower           │
├─────────────────┼──────────────────┼──────────────────────────┤
│ Split-brain     │ Vulnerable       │ Protected                │
├─────────────────┼──────────────────┼──────────────────────────┤
│ Poison Messages │ Manual handling  │ Delivery limit support   │
├─────────────────┼──────────────────┼──────────────────────────┤
│ Use Case        │ Legacy systems   │ New deployments          │
│                 │                  │ Critical data            │
└─────────────────┴──────────────────┴──────────────────────────┘

Recommendation: Use Quorum Queues for new applications
```

## 2. Docker Cluster Setup

### 2.1 Docker Compose Configuration

```yaml
# docker-compose.yml
version: '3.8'

services:
  rabbitmq1:
    image: rabbitmq:3.12-management
    hostname: rabbitmq1
    container_name: rabbitmq1
    environment:
      - RABBITMQ_ERLANG_COOKIE=SWQOKODSQALRPCLNMEQG
      - RABBITMQ_DEFAULT_USER=admin
      - RABBITMQ_DEFAULT_PASS=admin123
      - RABBITMQ_DEFAULT_VHOST=/
    ports:
      - "5672:5672"   # AMQP
      - "15672:15672" # Management UI
      - "25672:25672" # Cluster communication
    volumes:
      - rabbitmq1_data:/var/lib/rabbitmq
      - ./rabbitmq.conf:/etc/rabbitmq/rabbitmq.conf
      - ./enabled_plugins:/etc/rabbitmq/enabled_plugins
    networks:
      - rabbitmq_cluster
    healthcheck:
      test: ["CMD", "rabbitmq-diagnostics", "check_port_connectivity"]
      interval: 30s
      timeout: 10s
      retries: 5

  rabbitmq2:
    image: rabbitmq:3.12-management
    hostname: rabbitmq2
    container_name: rabbitmq2
    environment:
      - RABBITMQ_ERLANG_COOKIE=SWQOKODSQALRPCLNMEQG
      - RABBITMQ_DEFAULT_USER=admin
      - RABBITMQ_DEFAULT_PASS=admin123
      - RABBITMQ_DEFAULT_VHOST=/
    ports:
      - "5673:5672"
      - "15673:15672"
    volumes:
      - rabbitmq2_data:/var/lib/rabbitmq
      - ./rabbitmq.conf:/etc/rabbitmq/rabbitmq.conf
      - ./enabled_plugins:/etc/rabbitmq/enabled_plugins
    networks:
      - rabbitmq_cluster
    depends_on:
      rabbitmq1:
        condition: service_healthy

  rabbitmq3:
    image: rabbitmq:3.12-management
    hostname: rabbitmq3
    container_name: rabbitmq3
    environment:
      - RABBITMQ_ERLANG_COOKIE=SWQOKODSQALRPCLNMEQG
      - RABBITMQ_DEFAULT_USER=admin
      - RABBITMQ_DEFAULT_PASS=admin123
      - RABBITMQ_DEFAULT_VHOST=/
    ports:
      - "5674:5672"
      - "15674:15672"
    volumes:
      - rabbitmq3_data:/var/lib/rabbitmq
      - ./rabbitmq.conf:/etc/rabbitmq/rabbitmq.conf
      - ./enabled_plugins:/etc/rabbitmq/enabled_plugins
    networks:
      - rabbitmq_cluster
    depends_on:
      rabbitmq1:
        condition: service_healthy

  cluster-setup:
    image: rabbitmq:3.12-management
    container_name: rabbitmq-cluster-setup
    depends_on:
      - rabbitmq1
      - rabbitmq2
      - rabbitmq3
    volumes:
      - ./cluster-setup.sh:/cluster-setup.sh
    entrypoint: ["/bin/bash", "/cluster-setup.sh"]
    networks:
      - rabbitmq_cluster

volumes:
  rabbitmq1_data:
  rabbitmq2_data:
  rabbitmq3_data:

networks:
  rabbitmq_cluster:
    driver: bridge
```

### 2.2 Configuration Files

```ini
# rabbitmq.conf
cluster_formation.peer_discovery_backend = rabbit_peer_discovery_classic_config
cluster_formation.classic_config.nodes.1 = rabbit@rabbitmq1
cluster_formation.classic_config.nodes.2 = rabbit@rabbitmq2
cluster_formation.classic_config.nodes.3 = rabbit@rabbitmq3

# Quorum queue defaults
default_queue_type = quorum

# Cluster partition handling
cluster_partition_handling = pause_minority

# Memory and disk alarms
vm_memory_high_watermark.relative = 0.7
disk_free_limit.relative = 1.5

# Management plugin
management.tcp.port = 15672

# Logging
log.console = true
log.console.level = info

# Connection settings
heartbeat = 60
channel_max = 128
```

```
# enabled_plugins
[rabbitmq_management,rabbitmq_prometheus,rabbitmq_peer_discovery_common].
```

```bash
#!/bin/bash
# cluster-setup.sh

# Wait for all nodes to be ready
sleep 30

# Join node 2 to cluster
docker exec rabbitmq2 rabbitmqctl stop_app
docker exec rabbitmq2 rabbitmqctl reset
docker exec rabbitmq2 rabbitmqctl join_cluster rabbit@rabbitmq1
docker exec rabbitmq2 rabbitmqctl start_app

# Join node 3 to cluster
docker exec rabbitmq3 rabbitmqctl stop_app
docker exec rabbitmq3 rabbitmqctl reset
docker exec rabbitmq3 rabbitmqctl join_cluster rabbit@rabbitmq1
docker exec rabbitmq3 rabbitmqctl start_app

# Verify cluster status
docker exec rabbitmq1 rabbitmqctl cluster_status

# Set HA policy for classic queues (if any)
docker exec rabbitmq1 rabbitmqctl set_policy ha-all "^" \
    '{"ha-mode":"all","ha-sync-mode":"automatic"}' \
    --priority 1 \
    --apply-to queues

echo "Cluster setup complete!"
```

## 3. Quorum Queues

### 3.1 Quorum Queue Configuration

```java
@Configuration
public class QuorumQueueConfiguration {

    public static final String ORDER_QUEUE = "order.quorum.queue";
    public static final String PAYMENT_QUEUE = "payment.quorum.queue";

    /**
     * Quorum queue with all options
     */
    @Bean
    public Queue orderQuorumQueue() {
        return QueueBuilder
            .durable(ORDER_QUEUE)
            .quorum()
            // Initial group size (number of replicas)
            .withArgument("x-quorum-initial-group-size", 3)
            // Delivery limit before dead-lettering
            .withArgument("x-delivery-limit", 5)
            // Dead letter exchange
            .withArgument("x-dead-letter-exchange", "order.dlx")
            .withArgument("x-dead-letter-routing-key", "order.dlq")
            // Queue length limit
            .withArgument("x-max-length", 100000)
            .withArgument("x-overflow", "reject-publish")
            // Message TTL
            .withArgument("x-message-ttl", 86400000) // 24 hours
            .build();
    }

    @Bean
    public Queue paymentQuorumQueue() {
        return QueueBuilder
            .durable(PAYMENT_QUEUE)
            .quorum()
            .withArgument("x-quorum-initial-group-size", 3)
            .withArgument("x-delivery-limit", 3)
            .build();
    }

    /**
     * Using @Argument annotation in listener
     */
    @RabbitListener(bindings = @QueueBinding(
        value = @org.springframework.amqp.rabbit.annotation.Queue(
            name = "notification.quorum",
            durable = "true",
            arguments = {
                @Argument(name = "x-queue-type", value = "quorum"),
                @Argument(name = "x-quorum-initial-group-size",
                    value = "3", type = "java.lang.Integer"),
                @Argument(name = "x-delivery-limit",
                    value = "5", type = "java.lang.Integer")
            }
        ),
        exchange = @org.springframework.amqp.rabbit.annotation.Exchange(
            name = "notification.exchange",
            type = ExchangeTypes.DIRECT
        ),
        key = "notification"
    ))
    public void handleNotification(NotificationEvent event) {
        // Process notification
    }
}
```

### 3.2 Delivery Limit Handling

```java
@Component
@Slf4j
public class QuorumQueueConsumer {

    /**
     * Handle message with delivery count tracking
     */
    @RabbitListener(queues = QuorumQueueConfiguration.ORDER_QUEUE)
    public void handleOrder(
            OrderEvent event,
            Channel channel,
            @Header(AmqpHeaders.DELIVERY_TAG) long deliveryTag,
            @Header(name = "x-delivery-count", required = false) Integer deliveryCount)
            throws IOException {

        int count = deliveryCount != null ? deliveryCount : 1;

        log.info("Processing order: id={}, deliveryCount={}",
            event.getOrder().getOrderId(), count);

        try {
            processOrder(event);
            channel.basicAck(deliveryTag, false);

        } catch (RetryableException e) {
            log.warn("Retryable error (delivery {}): {}", count, e.getMessage());

            // For quorum queues, basicNack with requeue=true increments delivery count
            // Message will be dead-lettered when x-delivery-limit is exceeded
            channel.basicNack(deliveryTag, false, true);

        } catch (Exception e) {
            log.error("Non-retryable error: {}", e.getMessage());
            channel.basicNack(deliveryTag, false, false); // To DLQ immediately
        }
    }

    private void processOrder(OrderEvent event) {
        // Business logic
    }
}
```

### 3.3 Quorum Queue Monitoring

```java
@Service
@Slf4j
@RequiredArgsConstructor
public class QuorumQueueMonitor {

    private final RabbitAdmin rabbitAdmin;
    private final RabbitTemplate rabbitTemplate;

    /**
     * Get quorum queue status
     */
    public QuorumQueueStatus getQueueStatus(String queueName) {
        Properties queueProperties = rabbitAdmin.getQueueProperties(queueName);

        if (queueProperties == null) {
            throw new IllegalArgumentException("Queue not found: " + queueName);
        }

        return QuorumQueueStatus.builder()
            .queueName(queueName)
            .messageCount((Integer) queueProperties.get(
                RabbitAdmin.QUEUE_MESSAGE_COUNT))
            .consumerCount((Integer) queueProperties.get(
                RabbitAdmin.QUEUE_CONSUMER_COUNT))
            .leader((String) queueProperties.get("x-leader"))
            .members((List<String>) queueProperties.get("x-members"))
            .online((List<String>) queueProperties.get("x-online"))
            .build();
    }

    /**
     * Check cluster health
     */
    public ClusterHealth checkClusterHealth() {
        try {
            // Try to declare a temporary queue
            String testQueue = rabbitAdmin.declareQueue();
            rabbitAdmin.deleteQueue(testQueue);

            return ClusterHealth.builder()
                .healthy(true)
                .timestamp(Instant.now())
                .build();

        } catch (Exception e) {
            log.error("Cluster health check failed", e);

            return ClusterHealth.builder()
                .healthy(false)
                .error(e.getMessage())
                .timestamp(Instant.now())
                .build();
        }
    }

    @Data
    @Builder
    public static class QuorumQueueStatus {
        private String queueName;
        private int messageCount;
        private int consumerCount;
        private String leader;
        private List<String> members;
        private List<String> online;

        public boolean isHealthy() {
            return online != null && online.size() > members.size() / 2;
        }
    }

    @Data
    @Builder
    public static class ClusterHealth {
        private boolean healthy;
        private String error;
        private Instant timestamp;
    }
}
```

## 4. Connection Failover

### 4.1 Spring AMQP Cluster Configuration

```yaml
# application.yml
spring:
  rabbitmq:
    addresses: rabbitmq1:5672,rabbitmq2:5672,rabbitmq3:5672
    username: admin
    password: admin123
    virtual-host: /

    # Connection recovery
    connection-timeout: 30000
    channel-rpc-timeout: 30000

    # Automatic recovery
    # Spring AMQP handles this automatically

    # Publisher confirms for reliability
    publisher-confirm-type: correlated
    publisher-returns: true

    # Cache settings
    cache:
      channel:
        size: 25
        checkout-timeout: 1000
      connection:
        mode: channel # or connection for separate connections

    # Listener settings
    listener:
      simple:
        acknowledge-mode: manual
        prefetch: 10
        retry:
          enabled: true
          initial-interval: 1000
          max-attempts: 3
          max-interval: 10000
          multiplier: 2
        missing-queues-fatal: false
```

### 4.2 Custom Connection Factory with Failover

```java
@Configuration
@Slf4j
public class RabbitMQClusterConfiguration {

    @Value("${spring.rabbitmq.addresses}")
    private String addresses;

    @Value("${spring.rabbitmq.username}")
    private String username;

    @Value("${spring.rabbitmq.password}")
    private String password;

    @Bean
    public ConnectionFactory connectionFactory() {
        CachingConnectionFactory factory = new CachingConnectionFactory();

        // Set addresses for cluster
        factory.setAddresses(addresses);
        factory.setUsername(username);
        factory.setPassword(password);

        // Connection settings
        factory.setConnectionTimeout(30000);
        factory.setRequestedHeartBeat(60);
        factory.setChannelCacheSize(25);

        // Recovery settings
        factory.getRabbitConnectionFactory().setAutomaticRecoveryEnabled(true);
        factory.getRabbitConnectionFactory().setNetworkRecoveryInterval(5000);
        factory.getRabbitConnectionFactory().setTopologyRecoveryEnabled(true);

        // Connection naming for easier debugging
        factory.setConnectionNameStrategy(
            connectionFactory -> "order-service-" + System.currentTimeMillis());

        // Publisher confirms
        factory.setPublisherConfirmType(
            CachingConnectionFactory.ConfirmType.CORRELATED);
        factory.setPublisherReturns(true);

        // Add connection listener
        factory.addConnectionListener(new ConnectionListener() {
            @Override
            public void onCreate(Connection connection) {
                log.info("RabbitMQ connection created: {}",
                    connection.getLocalAddress());
            }

            @Override
            public void onClose(Connection connection) {
                log.warn("RabbitMQ connection closed");
            }

            @Override
            public void onShutDown(ShutdownSignalException signal) {
                if (signal.isInitiatedByApplication()) {
                    log.info("Connection shutdown initiated by application");
                } else {
                    log.error("Connection shutdown: {}", signal.getMessage());
                }
            }
        });

        return factory;
    }

    @Bean
    public RabbitTemplate rabbitTemplate(ConnectionFactory connectionFactory,
            MessageConverter messageConverter) {

        RabbitTemplate template = new RabbitTemplate(connectionFactory);
        template.setMessageConverter(messageConverter);
        template.setMandatory(true);

        // Retry template for failed operations
        RetryTemplate retryTemplate = new RetryTemplate();

        ExponentialBackOffPolicy backOffPolicy = new ExponentialBackOffPolicy();
        backOffPolicy.setInitialInterval(1000);
        backOffPolicy.setMultiplier(2.0);
        backOffPolicy.setMaxInterval(10000);

        SimpleRetryPolicy retryPolicy = new SimpleRetryPolicy();
        retryPolicy.setMaxAttempts(3);

        retryTemplate.setBackOffPolicy(backOffPolicy);
        retryTemplate.setRetryPolicy(retryPolicy);

        template.setRetryTemplate(retryTemplate);

        // Confirm callback
        template.setConfirmCallback((correlationData, ack, cause) -> {
            if (!ack) {
                log.error("Message not confirmed: {}, cause: {}",
                    correlationData != null ? correlationData.getId() : "unknown",
                    cause);
            }
        });

        // Return callback
        template.setReturnsCallback(returned -> {
            log.error("Message returned: exchange={}, routingKey={}, reason={}",
                returned.getExchange(),
                returned.getRoutingKey(),
                returned.getReplyText());
        });

        return template;
    }
}
```

### 4.3 Connection Recovery Handler

```java
@Component
@Slf4j
@RequiredArgsConstructor
public class ConnectionRecoveryHandler implements ApplicationListener<ListenerContainerConsumerFailedEvent> {

    private final SimpleRabbitListenerContainerFactory containerFactory;

    @Override
    public void onApplicationEvent(ListenerContainerConsumerFailedEvent event) {
        log.warn("Consumer failed: reason={}, fatal={}",
            event.getReason(), event.isFatal());

        if (event.isFatal()) {
            log.error("Fatal consumer error, container will not restart automatically");
            // Alert operations team
            alertOperations(event);
        }
    }

    @EventListener
    public void handleRecoveryEvent(ListenerContainerConsumerTerminatedEvent event) {
        log.info("Consumer terminated, recovery may be in progress");
    }

    private void alertOperations(ListenerContainerConsumerFailedEvent event) {
        // Send alert to monitoring system
    }
}

@Configuration
public class RecoveryConfiguration {

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

        // Recovery settings
        factory.setMissingQueuesFatal(false);
        factory.setFailedDeclarationRetryInterval(5000);
        factory.setRecoveryInterval(5000);

        // Error handler with recovery
        factory.setErrorHandler(new ConditionalRejectingErrorHandler(
            new RecoveryAwareExceptionStrategy()));

        return factory;
    }

    public static class RecoveryAwareExceptionStrategy
            extends ConditionalRejectingErrorHandler.DefaultExceptionStrategy {

        private static final Logger log =
            LoggerFactory.getLogger(RecoveryAwareExceptionStrategy.class);

        @Override
        public boolean isFatal(Throwable t) {
            // Connection-related exceptions are not fatal (will recover)
            if (t instanceof AmqpConnectException) {
                log.warn("Connection exception, will retry: {}", t.getMessage());
                return false;
            }

            // Channel exceptions may need retry
            if (t instanceof AmqpIOException) {
                log.warn("IO exception, will retry: {}", t.getMessage());
                return false;
            }

            return super.isFatal(t);
        }
    }
}
```

## 5. Partition Handling

### 5.1 Split-Brain Scenarios

```
Network Partition Scenarios
═══════════════════════════

Normal State:
┌──────────────────────────────────────────────────┐
│  Node 1 ◄──────▶ Node 2 ◄──────▶ Node 3         │
│    ▲                                 ▲           │
│    └─────────────────────────────────┘           │
│           Full mesh connectivity                  │
└──────────────────────────────────────────────────┘

Network Partition:
┌────────────────────┐     ┌────────────────────┐
│  Partition A       │ ✗✗✗ │  Partition B       │
│  ┌──────┐          │     │          ┌──────┐  │
│  │Node 1│◄──▶│Node 2│     │          │Node 3│  │
│  └──────┘          │     │          └──────┘  │
│  (Majority: 2/3)   │     │  (Minority: 1/3)   │
└────────────────────┘     └────────────────────┘

Partition Handling Strategies:
┌───────────────────┬─────────────────────────────────────┐
│ Strategy          │ Behavior                            │
├───────────────────┼─────────────────────────────────────┤
│ pause_minority    │ Minority partition stops            │
│                   │ (Data safety, no split-brain)       │
├───────────────────┼─────────────────────────────────────┤
│ pause_if_all_down │ Pause if all visible nodes down     │
├───────────────────┼─────────────────────────────────────┤
│ autoheal          │ Auto-restart minority on heal       │
│                   │ (May lose messages in minority)     │
├───────────────────┼─────────────────────────────────────┤
│ ignore            │ No action (split-brain risk!)       │
└───────────────────┴─────────────────────────────────────┘
```

### 5.2 Partition Handling Configuration

```java
@Component
@Slf4j
@RequiredArgsConstructor
public class PartitionAwarePublisher {

    private final RabbitTemplate rabbitTemplate;
    private final QuorumQueueMonitor monitor;

    /**
     * Publish with partition awareness
     */
    public PublishResult publish(String exchange, String routingKey, Object message) {
        // Check cluster health first
        ClusterHealth health = monitor.checkClusterHealth();

        if (!health.isHealthy()) {
            log.warn("Cluster unhealthy, storing message for later: {}",
                health.getError());
            return storeForLater(exchange, routingKey, message);
        }

        try {
            CorrelationData correlationData =
                new CorrelationData(UUID.randomUUID().toString());

            rabbitTemplate.convertAndSend(exchange, routingKey, message, correlationData);

            // Wait for confirm with timeout
            Boolean confirmed = correlationData.getFuture()
                .get(5, TimeUnit.SECONDS)
                .isAck();

            if (confirmed) {
                return PublishResult.success(correlationData.getId());
            } else {
                return storeForLater(exchange, routingKey, message);
            }

        } catch (TimeoutException e) {
            log.error("Publish timeout, cluster may be partitioned");
            return storeForLater(exchange, routingKey, message);

        } catch (Exception e) {
            log.error("Publish failed", e);
            return storeForLater(exchange, routingKey, message);
        }
    }

    private PublishResult storeForLater(String exchange, String routingKey,
            Object message) {
        // Store in database/file for retry later
        String storedId = storeMessage(exchange, routingKey, message);
        return PublishResult.queued(storedId);
    }

    private String storeMessage(String exchange, String routingKey, Object message) {
        // Implementation to store failed messages
        return UUID.randomUUID().toString();
    }

    @Data
    @Builder
    public static class PublishResult {
        private String messageId;
        private PublishStatus status;

        public static PublishResult success(String id) {
            return PublishResult.builder()
                .messageId(id)
                .status(PublishStatus.CONFIRMED)
                .build();
        }

        public static PublishResult queued(String id) {
            return PublishResult.builder()
                .messageId(id)
                .status(PublishStatus.QUEUED_FOR_RETRY)
                .build();
        }
    }

    public enum PublishStatus {
        CONFIRMED, QUEUED_FOR_RETRY, FAILED
    }
}
```

### 5.3 Stored Message Retry Service

```java
@Service
@Slf4j
@RequiredArgsConstructor
public class MessageRetryService {

    private final RabbitTemplate rabbitTemplate;
    private final FailedMessageRepository failedMessageRepository;
    private final QuorumQueueMonitor monitor;

    /**
     * Retry stored messages when cluster recovers
     */
    @Scheduled(fixedDelay = 30000) // Every 30 seconds
    public void retryStoredMessages() {
        if (!monitor.checkClusterHealth().isHealthy()) {
            log.debug("Cluster unhealthy, skipping retry");
            return;
        }

        List<FailedMessage> messages = failedMessageRepository
            .findByStatusOrderByCreatedAtAsc(MessageStatus.PENDING);

        log.info("Found {} messages to retry", messages.size());

        for (FailedMessage msg : messages) {
            try {
                CorrelationData correlationData = new CorrelationData(msg.getId());

                rabbitTemplate.convertAndSend(
                    msg.getExchange(),
                    msg.getRoutingKey(),
                    msg.deserializePayload(),
                    correlationData
                );

                Boolean confirmed = correlationData.getFuture()
                    .get(5, TimeUnit.SECONDS)
                    .isAck();

                if (confirmed) {
                    msg.setStatus(MessageStatus.SENT);
                    msg.setSentAt(Instant.now());
                    failedMessageRepository.save(msg);
                    log.info("Message retry successful: {}", msg.getId());
                }

            } catch (Exception e) {
                log.warn("Retry failed for message {}: {}",
                    msg.getId(), e.getMessage());

                msg.setRetryCount(msg.getRetryCount() + 1);
                msg.setLastError(e.getMessage());

                if (msg.getRetryCount() >= 10) {
                    msg.setStatus(MessageStatus.FAILED);
                }

                failedMessageRepository.save(msg);
            }
        }
    }
}

@Entity
@Table(name = "failed_messages")
@Data
public class FailedMessage {
    @Id
    private String id;
    private String exchange;
    private String routingKey;

    @Lob
    private byte[] payload;

    @Enumerated(EnumType.STRING)
    private MessageStatus status;

    private int retryCount;
    private String lastError;
    private Instant createdAt;
    private Instant sentAt;

    public Object deserializePayload() {
        // Deserialize using ObjectMapper
        return null;
    }
}

public enum MessageStatus {
    PENDING, SENT, FAILED
}
```

## 6. Monitoring & Alerting

### 6.1 Prometheus Metrics

```yaml
# prometheus.yml
scrape_configs:
  - job_name: 'rabbitmq'
    static_configs:
      - targets: ['rabbitmq1:15692', 'rabbitmq2:15692', 'rabbitmq3:15692']
    metrics_path: '/metrics'
```

```java
@Component
@Slf4j
@RequiredArgsConstructor
public class RabbitMQMetricsCollector {

    private final RabbitAdmin rabbitAdmin;
    private final MeterRegistry meterRegistry;

    @PostConstruct
    public void registerMetrics() {
        // Gauge for queue message count
        Gauge.builder("rabbitmq.queue.messages", this, collector -> {
                Properties props = rabbitAdmin.getQueueProperties("order.quorum.queue");
                return props != null ?
                    (Integer) props.get(RabbitAdmin.QUEUE_MESSAGE_COUNT) : 0;
            })
            .tag("queue", "order.quorum.queue")
            .register(meterRegistry);

        // Gauge for consumer count
        Gauge.builder("rabbitmq.queue.consumers", this, collector -> {
                Properties props = rabbitAdmin.getQueueProperties("order.quorum.queue");
                return props != null ?
                    (Integer) props.get(RabbitAdmin.QUEUE_CONSUMER_COUNT) : 0;
            })
            .tag("queue", "order.quorum.queue")
            .register(meterRegistry);
    }

    @Scheduled(fixedRate = 60000)
    public void checkThresholds() {
        Properties props = rabbitAdmin.getQueueProperties("order.quorum.queue");

        if (props != null) {
            int messageCount = (Integer) props.get(RabbitAdmin.QUEUE_MESSAGE_COUNT);
            int consumerCount = (Integer) props.get(RabbitAdmin.QUEUE_CONSUMER_COUNT);

            if (messageCount > 10000 && consumerCount < 3) {
                log.warn("Queue backlog detected: messages={}, consumers={}",
                    messageCount, consumerCount);
                // Send alert
            }
        }
    }
}
```

### 6.2 Health Indicator

```java
@Component
@Slf4j
@RequiredArgsConstructor
public class RabbitMQClusterHealthIndicator implements HealthIndicator {

    private final ConnectionFactory connectionFactory;
    private final RabbitAdmin rabbitAdmin;

    @Override
    public Health health() {
        try {
            // Check connection
            CachingConnectionFactory cachingFactory =
                (CachingConnectionFactory) connectionFactory;

            if (cachingFactory.getCacheMode() !=
                    CachingConnectionFactory.CacheMode.CHANNEL) {
                return Health.unknown()
                    .withDetail("reason", "Unexpected cache mode")
                    .build();
            }

            // Try to declare a queue
            String testQueue = rabbitAdmin.declareQueue();
            rabbitAdmin.deleteQueue(testQueue);

            // Get cluster status (if available)
            return Health.up()
                .withDetail("mode", "cluster")
                .withDetail("addresses", cachingFactory.getHost())
                .build();

        } catch (AmqpConnectException e) {
            return Health.down()
                .withDetail("error", "Connection failed")
                .withException(e)
                .build();

        } catch (Exception e) {
            return Health.down()
                .withException(e)
                .build();
        }
    }
}
```

### 6.3 Alerting Rules

```yaml
# alertmanager/rules/rabbitmq.yml
groups:
  - name: rabbitmq
    rules:
      - alert: RabbitMQClusterNodeDown
        expr: up{job="rabbitmq"} == 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "RabbitMQ node is down"
          description: "RabbitMQ node {{ $labels.instance }} has been down for more than 1 minute"

      - alert: RabbitMQHighQueueMessages
        expr: rabbitmq_queue_messages > 10000
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "High message count in queue"
          description: "Queue {{ $labels.queue }} has {{ $value }} messages"

      - alert: RabbitMQNoConsumers
        expr: rabbitmq_queue_consumers == 0 and rabbitmq_queue_messages > 0
        for: 2m
        labels:
          severity: warning
        annotations:
          summary: "No consumers for queue with messages"
          description: "Queue {{ $labels.queue }} has no consumers but {{ $value }} messages"

      - alert: RabbitMQQuorumQueueUnhealthy
        expr: rabbitmq_quorum_queue_members_online < (rabbitmq_quorum_queue_members / 2) + 1
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "Quorum queue has lost quorum"
          description: "Queue {{ $labels.queue }} has insufficient online members"
```

## 7. Hands-on Exercises

### Exercise 1: Cluster Setup
Set up a 3-node RabbitMQ cluster with Docker.

```bash
# Requirements:
# 1. Create docker-compose.yml with 3 nodes
# 2. Configure cluster joining
# 3. Create quorum queue
# 4. Verify failover by stopping one node
```

### Exercise 2: Partition Recovery
Implement partition-aware publishing.

```java
// Requirements:
// 1. Detect cluster health issues
// 2. Store messages locally during partition
// 3. Retry when cluster recovers
// 4. Alert operations team
```

### Exercise 3: Load Balancing
Configure client-side load balancing.

```java
// Requirements:
// 1. Round-robin connection to cluster nodes
// 2. Health check before publishing
// 3. Automatic failover on node failure
// 4. Connection pooling
```

### Exercise 4: Monitoring Dashboard
Build a cluster monitoring dashboard.

```java
// Requirements:
// 1. Real-time queue depths
// 2. Consumer counts per queue
// 3. Node health status
// 4. Message rate charts
```

## 8. Key Takeaways

### Cluster Architecture
1. **Erlang Cookie**: All nodes must share the same cookie
2. **Full Mesh**: All nodes communicate with each other
3. **Metadata Replication**: Exchanges, bindings, users replicated automatically

### Queue Types
1. **Quorum Queues**: Use for new deployments, strong consistency
2. **Classic HA**: Legacy, may be deprecated in future
3. **Delivery Limit**: Built-in retry limit for quorum queues

### High Availability
1. **Multiple Addresses**: Configure all cluster nodes in connection
2. **Automatic Recovery**: Spring AMQP handles connection recovery
3. **Publisher Confirms**: Verify message delivery to cluster

### Partition Handling
1. **pause_minority**: Recommended for data safety
2. **Quorum Queues**: Protected against split-brain
3. **Local Buffering**: Store messages during partitions

### Monitoring
1. **Prometheus**: Native metrics support
2. **Management UI**: Built-in cluster visualization
3. **Health Checks**: Critical for load balancers

---

## Navigation

[← Day 18: RabbitMQ Patterns](./day-18-rabbitmq-patterns.md) | [Day 20: Event-Driven Architecture →](./day-20-event-driven-architecture.md)

[Back to Overview](./00-overview.md)
