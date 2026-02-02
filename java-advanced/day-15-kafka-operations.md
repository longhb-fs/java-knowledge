# Day 15: Kafka Operations

## Mục tiêu
- Kafka monitoring và metrics
- Topic management
- Consumer lag monitoring
- Production best practices

---

## 1. Kafka Monitoring

### 1.1. Key Metrics Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                    KAFKA KEY METRICS                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   BROKER METRICS                                                │
│   ├── UnderReplicatedPartitions (should be 0)                   │
│   ├── OfflinePartitionsCount (should be 0)                      │
│   ├── ActiveControllerCount (exactly 1 per cluster)             │
│   ├── RequestHandlerAvgIdlePercent (>0.3)                       │
│   └── NetworkProcessorAvgIdlePercent (>0.3)                     │
│                                                                  │
│   PRODUCER METRICS                                              │
│   ├── record-send-rate (throughput)                             │
│   ├── record-error-rate (should be low)                         │
│   ├── request-latency-avg (response time)                       │
│   └── batch-size-avg (batching efficiency)                      │
│                                                                  │
│   CONSUMER METRICS                                              │
│   ├── records-consumed-rate (throughput)                        │
│   ├── records-lag (backlog)                                     │
│   ├── records-lag-max (worst partition lag)                     │
│   └── fetch-latency-avg (fetch time)                            │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 1.2. JMX Metrics Export

```yaml
# docker-compose.yml with JMX exporter
services:
  kafka:
    image: confluentinc/cp-kafka:7.5.0
    environment:
      KAFKA_JMX_PORT: 9999
      KAFKA_JMX_HOSTNAME: localhost
      KAFKA_OPTS: "-javaagent:/opt/jmx-exporter/jmx_prometheus_javaagent.jar=9404:/opt/jmx-exporter/kafka.yml"
    volumes:
      - ./jmx-exporter:/opt/jmx-exporter
```

```yaml
# jmx-exporter/kafka.yml
lowercaseOutputName: true
lowercaseOutputLabelNames: true
rules:
  # Broker metrics
  - pattern: kafka.server<type=ReplicaManager, name=UnderReplicatedPartitions><>Value
    name: kafka_server_replica_manager_under_replicated_partitions
  - pattern: kafka.controller<type=KafkaController, name=OfflinePartitionsCount><>Value
    name: kafka_controller_offline_partitions_count
  - pattern: kafka.controller<type=KafkaController, name=ActiveControllerCount><>Value
    name: kafka_controller_active_controller_count

  # Topic metrics
  - pattern: kafka.server<type=BrokerTopicMetrics, name=MessagesInPerSec, topic=(.+)><>OneMinuteRate
    name: kafka_server_broker_topic_metrics_messages_in_per_sec
    labels:
      topic: "$1"

  # Consumer group metrics
  - pattern: kafka.server<type=group-coordinator-metrics, name=(.+)><>Value
    name: kafka_server_group_coordinator_$1
```

### 1.3. Prometheus & Grafana Setup

```yaml
# docker-compose.yml
services:
  prometheus:
    image: prom/prometheus:v2.47.0
    ports:
      - "9090:9090"
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml

  grafana:
    image: grafana/grafana:10.0.0
    ports:
      - "3000:3000"
    environment:
      GF_SECURITY_ADMIN_PASSWORD: admin
    volumes:
      - grafana-data:/var/lib/grafana
```

```yaml
# prometheus.yml
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: 'kafka'
    static_configs:
      - targets: ['kafka:9404']

  - job_name: 'spring-boot'
    metrics_path: '/actuator/prometheus'
    static_configs:
      - targets: ['order-service:8081', 'user-service:8082']
```

---

## 2. Spring Boot Kafka Metrics

### 2.1. Micrometer Configuration

```xml
<dependency>
    <groupId>io.micrometer</groupId>
    <artifactId>micrometer-registry-prometheus</artifactId>
</dependency>
```

```yaml
# application.yml
management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics,prometheus
  metrics:
    tags:
      application: ${spring.application.name}
    export:
      prometheus:
        enabled: true

spring:
  kafka:
    producer:
      properties:
        metrics.recording.level: INFO
    consumer:
      properties:
        metrics.recording.level: INFO
```

### 2.2. Custom Kafka Metrics

```java
@Component
@RequiredArgsConstructor
public class KafkaMetrics {

    private final MeterRegistry meterRegistry;
    private final KafkaListenerEndpointRegistry registry;

    @Scheduled(fixedRate = 10000)
    public void recordMetrics() {
        registry.getAllListenerContainers().forEach(container -> {
            String listenerId = container.getListenerId();

            // Container running status
            Gauge.builder("kafka.listener.running", container, c -> c.isRunning() ? 1 : 0)
                .tag("listener", listenerId)
                .register(meterRegistry);

            // Paused status
            Gauge.builder("kafka.listener.paused", container, c -> c.isContainerPaused() ? 1 : 0)
                .tag("listener", listenerId)
                .register(meterRegistry);
        });
    }

    // Record message processing
    public void recordProcessingTime(String topic, long durationMs) {
        meterRegistry.timer("kafka.message.processing.time",
            Tags.of("topic", topic))
            .record(Duration.ofMillis(durationMs));
    }

    public void recordMessageProcessed(String topic, boolean success) {
        meterRegistry.counter("kafka.messages.processed",
            Tags.of("topic", topic, "success", String.valueOf(success)))
            .increment();
    }
}

// Usage in consumer
@Service
@RequiredArgsConstructor
public class OrderConsumer {

    private final KafkaMetrics kafkaMetrics;

    @KafkaListener(topics = "order-events")
    public void consume(OrderEvent event, Acknowledgment ack) {
        long start = System.currentTimeMillis();
        boolean success = false;

        try {
            processOrder(event);
            success = true;
            ack.acknowledge();
        } finally {
            kafkaMetrics.recordProcessingTime("order-events",
                System.currentTimeMillis() - start);
            kafkaMetrics.recordMessageProcessed("order-events", success);
        }
    }
}
```

---

## 3. Consumer Lag Monitoring

### 3.1. What is Consumer Lag?

```
┌─────────────────────────────────────────────────────────────────┐
│                    CONSUMER LAG                                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Partition 0:                                                   │
│   ┌───┬───┬───┬───┬───┬───┬───┬───┬───┬───┬───┬───┐           │
│   │ 0 │ 1 │ 2 │ 3 │ 4 │ 5 │ 6 │ 7 │ 8 │ 9 │10 │11 │→ (new)    │
│   └───┴───┴───┴───┴───┴───┴───┴───┴───┴───┴───┴───┘           │
│                       ↑                       ↑                 │
│               Consumer Offset         Log End Offset            │
│                    (5)                    (11)                  │
│                                                                  │
│   LAG = Log End Offset - Consumer Offset = 11 - 5 = 6          │
│                                                                  │
│   Why Lag Matters:                                              │
│   ├── High lag = consumers falling behind                       │
│   ├── Data staleness                                            │
│   ├── Memory pressure (pending messages)                        │
│   └── May indicate scaling needed                               │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 3.2. Monitoring Lag with Admin Client

```java
@Service
@Slf4j
public class ConsumerLagMonitor {

    private final AdminClient adminClient;
    private final MeterRegistry meterRegistry;

    public ConsumerLagMonitor(
            @Value("${spring.kafka.bootstrap-servers}") String bootstrapServers,
            MeterRegistry meterRegistry) {

        Properties props = new Properties();
        props.put(AdminClientConfig.BOOTSTRAP_SERVERS_CONFIG, bootstrapServers);
        this.adminClient = AdminClient.create(props);
        this.meterRegistry = meterRegistry;
    }

    @Scheduled(fixedRate = 30000)
    public void monitorLag() {
        try {
            // List all consumer groups
            ListConsumerGroupsResult groupsResult = adminClient.listConsumerGroups();
            Collection<ConsumerGroupListing> groups = groupsResult.all().get();

            for (ConsumerGroupListing group : groups) {
                String groupId = group.groupId();
                Map<TopicPartition, Long> lag = calculateLag(groupId);

                lag.forEach((tp, lagValue) -> {
                    log.info("Consumer lag: group={}, topic={}, partition={}, lag={}",
                        groupId, tp.topic(), tp.partition(), lagValue);

                    // Record metric
                    Gauge.builder("kafka.consumer.lag", lagValue, Number::doubleValue)
                        .tag("group", groupId)
                        .tag("topic", tp.topic())
                        .tag("partition", String.valueOf(tp.partition()))
                        .register(meterRegistry);

                    // Alert if lag is high
                    if (lagValue > 10000) {
                        alertHighLag(groupId, tp, lagValue);
                    }
                });
            }
        } catch (Exception e) {
            log.error("Error monitoring consumer lag", e);
        }
    }

    private Map<TopicPartition, Long> calculateLag(String groupId) throws Exception {
        // Get consumer group offsets
        ListConsumerGroupOffsetsResult offsetsResult =
            adminClient.listConsumerGroupOffsets(groupId);
        Map<TopicPartition, OffsetAndMetadata> consumerOffsets =
            offsetsResult.partitionsToOffsetAndMetadata().get();

        if (consumerOffsets.isEmpty()) {
            return Collections.emptyMap();
        }

        // Get end offsets
        Set<TopicPartition> partitions = consumerOffsets.keySet();
        Map<TopicPartition, ListOffsetsResult.ListOffsetsResultInfo> endOffsets =
            adminClient.listOffsets(
                partitions.stream().collect(Collectors.toMap(
                    tp -> tp,
                    tp -> OffsetSpec.latest()
                ))
            ).all().get();

        // Calculate lag
        Map<TopicPartition, Long> lag = new HashMap<>();
        for (TopicPartition tp : partitions) {
            long consumerOffset = consumerOffsets.get(tp).offset();
            long endOffset = endOffsets.get(tp).offset();
            lag.put(tp, endOffset - consumerOffset);
        }

        return lag;
    }

    private void alertHighLag(String groupId, TopicPartition tp, long lag) {
        // Send alert via Slack, PagerDuty, etc.
    }
}
```

### 3.3. Burrow for Lag Monitoring

```yaml
# docker-compose.yml
services:
  burrow:
    image: linkedin/burrow:latest
    ports:
      - "8000:8000"
    volumes:
      - ./burrow.toml:/etc/burrow/burrow.toml
```

```toml
# burrow.toml
[general]
pidfile="burrow.pid"
stdout-logfile="burrow.log"

[logging]
level="info"

[zookeeper]
servers=["zookeeper:2181"]

[client-profile.local]
kafka-version="2.0.0"

[cluster.local]
class-name="kafka"
servers=["kafka:9092"]
client-profile="local"

[consumer.local]
class-name="kafka"
cluster="local"
servers=["kafka:9092"]
client-profile="local"

[httpserver.default]
address=":8000"
```

---

## 4. Topic Management

### 4.1. Topic Operations

```java
@Service
@Slf4j
public class TopicManagementService {

    private final AdminClient adminClient;

    public TopicManagementService(
            @Value("${spring.kafka.bootstrap-servers}") String bootstrapServers) {
        Properties props = new Properties();
        props.put(AdminClientConfig.BOOTSTRAP_SERVERS_CONFIG, bootstrapServers);
        this.adminClient = AdminClient.create(props);
    }

    // Create topic
    public void createTopic(String topicName, int partitions, short replicationFactor) {
        NewTopic topic = new NewTopic(topicName, partitions, replicationFactor);

        // With configs
        Map<String, String> configs = new HashMap<>();
        configs.put("retention.ms", String.valueOf(7 * 24 * 60 * 60 * 1000L)); // 7 days
        configs.put("cleanup.policy", "delete");
        configs.put("min.insync.replicas", "2");
        topic.configs(configs);

        try {
            adminClient.createTopics(Collections.singletonList(topic)).all().get();
            log.info("Created topic: {}", topicName);
        } catch (Exception e) {
            log.error("Failed to create topic: {}", topicName, e);
        }
    }

    // List topics
    public Set<String> listTopics() throws Exception {
        return adminClient.listTopics().names().get();
    }

    // Describe topic
    public TopicDescription describeTopic(String topicName) throws Exception {
        Map<String, TopicDescription> descriptions =
            adminClient.describeTopics(Collections.singletonList(topicName)).allTopicNames().get();
        return descriptions.get(topicName);
    }

    // Delete topic
    public void deleteTopic(String topicName) {
        try {
            adminClient.deleteTopics(Collections.singletonList(topicName)).all().get();
            log.info("Deleted topic: {}", topicName);
        } catch (Exception e) {
            log.error("Failed to delete topic: {}", topicName, e);
        }
    }

    // Increase partitions
    public void increasePartitions(String topicName, int newPartitionCount) {
        try {
            Map<String, NewPartitions> newPartitions = new HashMap<>();
            newPartitions.put(topicName, NewPartitions.increaseTo(newPartitionCount));
            adminClient.createPartitions(newPartitions).all().get();
            log.info("Increased partitions for {}: {}", topicName, newPartitionCount);
        } catch (Exception e) {
            log.error("Failed to increase partitions", e);
        }
    }

    // Update topic config
    public void updateTopicConfig(String topicName, Map<String, String> configs) {
        try {
            ConfigResource resource = new ConfigResource(ConfigResource.Type.TOPIC, topicName);
            List<AlterConfigOp> ops = configs.entrySet().stream()
                .map(e -> new AlterConfigOp(
                    new ConfigEntry(e.getKey(), e.getValue()),
                    AlterConfigOp.OpType.SET
                ))
                .collect(Collectors.toList());

            adminClient.incrementalAlterConfigs(Map.of(resource, ops)).all().get();
            log.info("Updated config for {}: {}", topicName, configs);
        } catch (Exception e) {
            log.error("Failed to update config", e);
        }
    }
}
```

### 4.2. Topic Naming Conventions

```
Topic Naming Best Practices:

Format: <domain>.<entity>.<event-type>.<version>

Examples:
├── order.orders.created.v1
├── order.orders.updated.v1
├── payment.transactions.completed.v1
├── inventory.stock.reserved.v1
└── notification.emails.sent.v1

Special Topics:
├── <topic>.retry-0, .retry-1, .retry-2  (retry topics)
├── <topic>.DLQ                           (dead letter)
└── __consumer_offsets                    (internal)

Compacted Topics (for state):
├── user.profiles.state
├── product.catalog.state
└── inventory.levels.state
```

---

## 5. Production Best Practices

### 5.1. Broker Configuration

```properties
# server.properties
# Replication
default.replication.factor=3
min.insync.replicas=2
unclean.leader.election.enable=false

# Performance
num.network.threads=8
num.io.threads=16
socket.send.buffer.bytes=102400
socket.receive.buffer.bytes=102400
socket.request.max.bytes=104857600

# Log retention
log.retention.hours=168
log.retention.bytes=1073741824
log.segment.bytes=1073741824
log.retention.check.interval.ms=300000

# Compression
compression.type=producer
```

### 5.2. Producer Best Practices

```java
@Configuration
public class ProductionProducerConfig {

    @Bean
    public ProducerFactory<String, Object> producerFactory() {
        Map<String, Object> config = new HashMap<>();

        // Reliability
        config.put(ProducerConfig.ACKS_CONFIG, "all");
        config.put(ProducerConfig.ENABLE_IDEMPOTENCE_CONFIG, true);
        config.put(ProducerConfig.RETRIES_CONFIG, Integer.MAX_VALUE);
        config.put(ProducerConfig.MAX_IN_FLIGHT_REQUESTS_PER_CONNECTION, 5);

        // Performance
        config.put(ProducerConfig.BATCH_SIZE_CONFIG, 16384);
        config.put(ProducerConfig.LINGER_MS_CONFIG, 5);
        config.put(ProducerConfig.BUFFER_MEMORY_CONFIG, 33554432);
        config.put(ProducerConfig.COMPRESSION_TYPE_CONFIG, "snappy");

        // Timeouts
        config.put(ProducerConfig.REQUEST_TIMEOUT_MS_CONFIG, 30000);
        config.put(ProducerConfig.DELIVERY_TIMEOUT_MS_CONFIG, 120000);

        return new DefaultKafkaProducerFactory<>(config);
    }
}
```

### 5.3. Consumer Best Practices

```java
@Configuration
public class ProductionConsumerConfig {

    @Bean
    public ConsumerFactory<String, Object> consumerFactory() {
        Map<String, Object> config = new HashMap<>();

        // Reliability
        config.put(ConsumerConfig.ENABLE_AUTO_COMMIT_CONFIG, false);
        config.put(ConsumerConfig.ISOLATION_LEVEL_CONFIG, "read_committed");

        // Performance
        config.put(ConsumerConfig.MAX_POLL_RECORDS_CONFIG, 500);
        config.put(ConsumerConfig.FETCH_MIN_BYTES_CONFIG, 1);
        config.put(ConsumerConfig.FETCH_MAX_WAIT_MS_CONFIG, 500);

        // Session management
        config.put(ConsumerConfig.SESSION_TIMEOUT_MS_CONFIG, 45000);
        config.put(ConsumerConfig.HEARTBEAT_INTERVAL_MS_CONFIG, 15000);
        config.put(ConsumerConfig.MAX_POLL_INTERVAL_MS_CONFIG, 300000);

        // Rebalancing
        config.put(ConsumerConfig.PARTITION_ASSIGNMENT_STRATEGY_CONFIG,
            CooperativeStickyAssignor.class.getName());

        return new DefaultKafkaConsumerFactory<>(config);
    }

    @Bean
    public ConcurrentKafkaListenerContainerFactory<String, Object> kafkaListenerContainerFactory() {
        ConcurrentKafkaListenerContainerFactory<String, Object> factory =
            new ConcurrentKafkaListenerContainerFactory<>();
        factory.setConsumerFactory(consumerFactory());
        factory.setConcurrency(3);
        factory.getContainerProperties().setAckMode(ContainerProperties.AckMode.MANUAL_IMMEDIATE);

        // Graceful shutdown
        factory.getContainerProperties().setShutdownTimeout(30000L);

        return factory;
    }
}
```

### 5.4. Health Checks

```java
@Component
public class KafkaHealthIndicator implements HealthIndicator {

    private final AdminClient adminClient;

    @Override
    public Health health() {
        try {
            // Check cluster connectivity
            DescribeClusterResult cluster = adminClient.describeCluster();
            Collection<Node> nodes = cluster.nodes().get(5, TimeUnit.SECONDS);

            if (nodes.isEmpty()) {
                return Health.down()
                    .withDetail("error", "No brokers available")
                    .build();
            }

            // Check controller
            Node controller = cluster.controller().get(5, TimeUnit.SECONDS);
            if (controller == null) {
                return Health.down()
                    .withDetail("error", "No controller available")
                    .build();
            }

            return Health.up()
                .withDetail("clusterId", cluster.clusterId().get())
                .withDetail("brokers", nodes.size())
                .withDetail("controller", controller.id())
                .build();

        } catch (Exception e) {
            return Health.down()
                .withDetail("error", e.getMessage())
                .build();
        }
    }
}
```

---

## 6. Troubleshooting

### 6.1. Common Issues

```
┌─────────────────────────────────────────────────────────────────┐
│                    COMMON KAFKA ISSUES                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   High Consumer Lag                                             │
│   ├── Cause: Slow processing, insufficient consumers            │
│   ├── Fix: Scale consumers, optimize processing                 │
│   └── Monitor: records-lag metric                               │
│                                                                  │
│   UnderReplicatedPartitions                                     │
│   ├── Cause: Broker down, network issues                        │
│   ├── Fix: Check broker health, network                         │
│   └── Monitor: UnderReplicatedPartitions metric                 │
│                                                                  │
│   Producer Timeout                                              │
│   ├── Cause: Slow broker, network latency                       │
│   ├── Fix: Increase timeout, check broker load                  │
│   └── Monitor: request-latency metric                           │
│                                                                  │
│   Consumer Rebalancing                                          │
│   ├── Cause: Consumer crash, slow processing                    │
│   ├── Fix: Increase session.timeout.ms                          │
│   └── Monitor: Rebalance events                                 │
│                                                                  │
│   Disk Full                                                     │
│   ├── Cause: Retention too long, high volume                    │
│   ├── Fix: Reduce retention, add storage                        │
│   └── Monitor: Disk usage metrics                               │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 6.2. Diagnostic Commands

```bash
# Check cluster health
kafka-metadata --snapshot /var/kafka-logs/__cluster_metadata-0/00000000000000000000.log --command "describe"

# Check consumer group status
kafka-consumer-groups --bootstrap-server localhost:9092 \
  --group order-processors --describe

# Check topic configuration
kafka-configs --bootstrap-server localhost:9092 \
  --entity-type topics --entity-name order-events --describe

# Check broker configuration
kafka-configs --bootstrap-server localhost:9092 \
  --entity-type brokers --entity-name 0 --describe

# Reset consumer offset
kafka-consumer-groups --bootstrap-server localhost:9092 \
  --group order-processors \
  --topic order-events \
  --reset-offsets --to-earliest --execute

# Delete consumer group
kafka-consumer-groups --bootstrap-server localhost:9092 \
  --delete --group order-processors
```

---

## 7. Bài tập thực hành

### Bài 1: Monitoring Setup
Setup Prometheus + Grafana dashboard cho Kafka metrics.

### Bài 2: Consumer Lag Alert
Implement consumer lag monitoring với alerting.

### Bài 3: Topic Management
Tạo admin API cho topic operations.

### Bài 4: Health Check
Implement comprehensive Kafka health indicator.

---

## 8. Key Takeaways

| Metric | Target | Action if Exceeded |
|--------|--------|-------------------|
| Consumer Lag | < 1000 | Scale consumers |
| UnderReplicated | 0 | Check broker health |
| Request Latency | < 100ms | Optimize broker |
| Disk Usage | < 80% | Reduce retention |

```
Operations Checklist:
☑ Monitor consumer lag
☑ Alert on high lag
☑ Track broker health
☑ Backup topic configs
☑ Test disaster recovery
☑ Document runbooks
```

---

## Navigation

- [← Day 14: Kafka Patterns](./day-14-kafka-patterns.md)
- [Day 16: RabbitMQ Fundamentals →](./day-16-rabbitmq-fundamentals.md)
