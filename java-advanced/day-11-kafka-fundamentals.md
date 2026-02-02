# Day 11: Kafka Fundamentals

## Mục tiêu
- Apache Kafka architecture
- Core concepts: Topics, Partitions, Producers, Consumers
- Kafka setup và basic operations
- Message delivery guarantees

---

## 1. What is Apache Kafka?

### 1.1. Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                    APACHE KAFKA                                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Distributed event streaming platform                          │
│                                                                  │
│   Use Cases:                                                     │
│   ├── Message Queue (async communication)                        │
│   ├── Event Sourcing (capture all changes)                      │
│   ├── Log Aggregation (collect logs from services)              │
│   ├── Stream Processing (real-time analytics)                   │
│   └── Data Integration (connect systems)                        │
│                                                                  │
│   Key Characteristics:                                           │
│   ├── High Throughput (millions of messages/sec)                │
│   ├── Durability (persisted to disk)                            │
│   ├── Scalability (horizontal scaling)                          │
│   ├── Fault Tolerance (replication)                             │
│   └── Low Latency (milliseconds)                                │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 1.2. Kafka vs Traditional Message Queues

```
┌────────────────────┬────────────────────┬────────────────────┐
│     Feature        │      Kafka         │  Traditional MQ    │
├────────────────────┼────────────────────┼────────────────────┤
│ Message Retention  │ Configurable (days)│ Until consumed     │
│ Consumer Model     │ Pull-based         │ Push-based         │
│ Replay Messages    │ Yes                │ No                 │
│ Ordering           │ Per partition      │ Queue-wide         │
│ Throughput         │ Very High          │ Moderate           │
│ Consumer Groups    │ Native             │ Limited            │
│ Use Case           │ Event Streaming    │ Task Queue         │
└────────────────────┴────────────────────┴────────────────────┘
```

---

## 2. Kafka Architecture

### 2.1. Core Components

```
┌─────────────────────────────────────────────────────────────────┐
│                    KAFKA ARCHITECTURE                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   ┌──────────────────────────────────────────────────────────┐  │
│   │                    KAFKA CLUSTER                          │  │
│   │                                                           │  │
│   │    ┌─────────┐    ┌─────────┐    ┌─────────┐            │  │
│   │    │ Broker 1│    │ Broker 2│    │ Broker 3│            │  │
│   │    │ (Leader)│    │(Replica)│    │(Replica)│            │  │
│   │    └────┬────┘    └────┬────┘    └────┬────┘            │  │
│   │         │              │              │                  │  │
│   │         └──────────────┼──────────────┘                  │  │
│   │                        │                                  │  │
│   │              ┌─────────┴─────────┐                       │  │
│   │              │    ZooKeeper /     │                       │  │
│   │              │    KRaft           │                       │  │
│   │              └───────────────────┘                       │  │
│   └──────────────────────────────────────────────────────────┘  │
│                                                                  │
│   Producers ───────────► Brokers ───────────► Consumers         │
│                                                                  │
│   Components:                                                    │
│   ├── Broker: Kafka server storing messages                     │
│   ├── Topic: Category/stream name                               │
│   ├── Partition: Ordered, immutable log                         │
│   ├── Producer: Publishes messages                              │
│   ├── Consumer: Reads messages                                  │
│   └── Consumer Group: Parallel consumers                        │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 2.2. Topics and Partitions

```
┌─────────────────────────────────────────────────────────────────┐
│                    TOPIC: order-events                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Partition 0:                                                   │
│   ┌───┬───┬───┬───┬───┬───┬───┬───┬───┬───┐                    │
│   │ 0 │ 1 │ 2 │ 3 │ 4 │ 5 │ 6 │ 7 │ 8 │ 9 │ → (append only)   │
│   └───┴───┴───┴───┴───┴───┴───┴───┴───┴───┘                    │
│         ↑ Consumer offset                                        │
│                                                                  │
│   Partition 1:                                                   │
│   ┌───┬───┬───┬───┬───┬───┬───┬───┐                            │
│   │ 0 │ 1 │ 2 │ 3 │ 4 │ 5 │ 6 │ 7 │ →                          │
│   └───┴───┴───┴───┴───┴───┴───┴───┘                            │
│                 ↑ Consumer offset                                │
│                                                                  │
│   Partition 2:                                                   │
│   ┌───┬───┬───┬───┬───┬───┐                                    │
│   │ 0 │ 1 │ 2 │ 3 │ 4 │ 5 │ →                                  │
│   └───┴───┴───┴───┴───┴───┘                                    │
│             ↑ Consumer offset                                    │
│                                                                  │
│   Key Points:                                                    │
│   ├── Messages are ordered within a partition                    │
│   ├── Each partition has its own offset                          │
│   ├── Partitions enable parallelism                              │
│   └── Message key determines partition                           │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 2.3. Consumer Groups

```
┌─────────────────────────────────────────────────────────────────┐
│                    CONSUMER GROUPS                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Topic: order-events (3 partitions)                            │
│   ┌─────────┐ ┌─────────┐ ┌─────────┐                          │
│   │Partition│ │Partition│ │Partition│                          │
│   │    0    │ │    1    │ │    2    │                          │
│   └────┬────┘ └────┬────┘ └────┬────┘                          │
│        │           │           │                                │
│        ▼           ▼           ▼                                │
│   ┌────────────────────────────────────┐                       │
│   │      Consumer Group: order-processors                       │
│   │  ┌──────────┐ ┌──────────┐ ┌──────────┐                   │
│   │  │Consumer 1│ │Consumer 2│ │Consumer 3│                   │
│   │  │(P0)      │ │(P1)      │ │(P2)      │                   │
│   │  └──────────┘ └──────────┘ └──────────┘                   │
│   └────────────────────────────────────┘                       │
│                                                                  │
│   ┌────────────────────────────────────┐                       │
│   │      Consumer Group: order-analytics                        │
│   │  ┌──────────────────────────────┐                          │
│   │  │       Consumer 1             │ (reads all partitions)   │
│   │  │       (P0, P1, P2)           │                          │
│   │  └──────────────────────────────┘                          │
│   └────────────────────────────────┘                           │
│                                                                  │
│   Rules:                                                         │
│   ├── Each partition → max 1 consumer in a group               │
│   ├── Each consumer → can read multiple partitions             │
│   ├── Different groups → read same messages independently      │
│   └── Consumers > partitions → some consumers idle             │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## 3. Kafka Setup

### 3.1. Docker Compose

```yaml
# docker-compose.yml
version: '3.8'
services:
  zookeeper:
    image: confluentinc/cp-zookeeper:7.5.0
    environment:
      ZOOKEEPER_CLIENT_PORT: 2181
      ZOOKEEPER_TICK_TIME: 2000
    ports:
      - "2181:2181"

  kafka:
    image: confluentinc/cp-kafka:7.5.0
    depends_on:
      - zookeeper
    ports:
      - "9092:9092"
      - "29092:29092"
    environment:
      KAFKA_BROKER_ID: 1
      KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://kafka:29092,PLAINTEXT_HOST://localhost:9092
      KAFKA_LISTENER_SECURITY_PROTOCOL_MAP: PLAINTEXT:PLAINTEXT,PLAINTEXT_HOST:PLAINTEXT
      KAFKA_INTER_BROKER_LISTENER_NAME: PLAINTEXT
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1
      KAFKA_AUTO_CREATE_TOPICS_ENABLE: "true"

  kafka-ui:
    image: provectuslabs/kafka-ui:latest
    ports:
      - "8080:8080"
    environment:
      KAFKA_CLUSTERS_0_NAME: local
      KAFKA_CLUSTERS_0_BOOTSTRAPSERVERS: kafka:29092
      KAFKA_CLUSTERS_0_ZOOKEEPER: zookeeper:2181
    depends_on:
      - kafka
```

### 3.2. Kafka CLI Commands

```bash
# Create topic
kafka-topics --create --topic order-events \
  --bootstrap-server localhost:9092 \
  --partitions 3 \
  --replication-factor 1

# List topics
kafka-topics --list --bootstrap-server localhost:9092

# Describe topic
kafka-topics --describe --topic order-events \
  --bootstrap-server localhost:9092

# Produce messages
kafka-console-producer --topic order-events \
  --bootstrap-server localhost:9092

# Consume messages
kafka-console-consumer --topic order-events \
  --bootstrap-server localhost:9092 \
  --from-beginning

# Consume with consumer group
kafka-console-consumer --topic order-events \
  --bootstrap-server localhost:9092 \
  --group order-processors

# List consumer groups
kafka-consumer-groups --list --bootstrap-server localhost:9092

# Describe consumer group
kafka-consumer-groups --describe --group order-processors \
  --bootstrap-server localhost:9092

# Delete topic
kafka-topics --delete --topic order-events \
  --bootstrap-server localhost:9092
```

---

## 4. Java Producer

### 4.1. Basic Producer

```java
import org.apache.kafka.clients.producer.*;
import org.apache.kafka.common.serialization.StringSerializer;

public class SimpleProducer {

    public static void main(String[] args) {
        Properties props = new Properties();
        props.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
        props.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, StringSerializer.class.getName());
        props.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, StringSerializer.class.getName());

        try (KafkaProducer<String, String> producer = new KafkaProducer<>(props)) {
            for (int i = 0; i < 10; i++) {
                String key = "order-" + i;
                String value = "{\"orderId\": \"" + i + "\", \"amount\": 100}";

                ProducerRecord<String, String> record =
                    new ProducerRecord<>("order-events", key, value);

                // Async send
                producer.send(record, (metadata, exception) -> {
                    if (exception == null) {
                        System.out.printf("Sent to partition %d, offset %d%n",
                            metadata.partition(), metadata.offset());
                    } else {
                        exception.printStackTrace();
                    }
                });
            }

            producer.flush();  // Ensure all messages sent
        }
    }
}
```

### 4.2. Producer Configuration

```java
Properties props = new Properties();

// Required
props.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
props.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, StringSerializer.class.getName());
props.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, StringSerializer.class.getName());

// Reliability
props.put(ProducerConfig.ACKS_CONFIG, "all");  // Wait for all replicas
props.put(ProducerConfig.RETRIES_CONFIG, 3);   // Retry on failure
props.put(ProducerConfig.ENABLE_IDEMPOTENCE_CONFIG, true);  // Exactly-once

// Performance
props.put(ProducerConfig.BATCH_SIZE_CONFIG, 16384);  // 16KB batch
props.put(ProducerConfig.LINGER_MS_CONFIG, 5);       // Wait up to 5ms for batching
props.put(ProducerConfig.BUFFER_MEMORY_CONFIG, 33554432);  // 32MB buffer
props.put(ProducerConfig.COMPRESSION_TYPE_CONFIG, "snappy");  // Compression
```

### 4.3. Acknowledgment Modes

```
┌─────────────────────────────────────────────────────────────────┐
│                    ACKS CONFIGURATION                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   acks=0 (Fire and forget)                                      │
│   ┌──────────┐         ┌──────────┐                             │
│   │ Producer │ ──────► │  Broker  │                             │
│   │  (send)  │         │(no wait) │                             │
│   └──────────┘         └──────────┘                             │
│   Fastest, but may lose messages                                │
│                                                                  │
│   acks=1 (Leader acknowledgment)                                │
│   ┌──────────┐         ┌──────────┐         ┌──────────┐       │
│   │ Producer │ ──────► │  Leader  │ ──────► │ Replica  │       │
│   │  (send)  │ ◄────── │  (ack)   │         │(no wait) │       │
│   └──────────┘         └──────────┘         └──────────┘       │
│   Balance of speed and reliability                              │
│                                                                  │
│   acks=all (All replicas)                                       │
│   ┌──────────┐         ┌──────────┐         ┌──────────┐       │
│   │ Producer │ ──────► │  Leader  │ ──────► │ Replica  │       │
│   │  (send)  │ ◄────── │  (wait)  │ ◄────── │  (ack)   │       │
│   └──────────┘         └──────────┘         └──────────┘       │
│   Most reliable, but slower                                     │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## 5. Java Consumer

### 5.1. Basic Consumer

```java
import org.apache.kafka.clients.consumer.*;
import org.apache.kafka.common.serialization.StringDeserializer;

public class SimpleConsumer {

    public static void main(String[] args) {
        Properties props = new Properties();
        props.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
        props.put(ConsumerConfig.GROUP_ID_CONFIG, "order-processors");
        props.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class.getName());
        props.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class.getName());
        props.put(ConsumerConfig.AUTO_OFFSET_RESET_CONFIG, "earliest");

        try (KafkaConsumer<String, String> consumer = new KafkaConsumer<>(props)) {
            consumer.subscribe(Collections.singletonList("order-events"));

            while (true) {
                ConsumerRecords<String, String> records = consumer.poll(Duration.ofMillis(1000));

                for (ConsumerRecord<String, String> record : records) {
                    System.out.printf("Received: key=%s, value=%s, partition=%d, offset=%d%n",
                        record.key(), record.value(), record.partition(), record.offset());

                    // Process message
                    processOrder(record.value());
                }
            }
        }
    }

    private static void processOrder(String orderJson) {
        // Business logic here
    }
}
```

### 5.2. Consumer Configuration

```java
Properties props = new Properties();

// Required
props.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
props.put(ConsumerConfig.GROUP_ID_CONFIG, "order-processors");
props.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class.getName());
props.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class.getName());

// Offset management
props.put(ConsumerConfig.AUTO_OFFSET_RESET_CONFIG, "earliest");  // or "latest"
props.put(ConsumerConfig.ENABLE_AUTO_COMMIT_CONFIG, false);  // Manual commit

// Performance
props.put(ConsumerConfig.MAX_POLL_RECORDS_CONFIG, 500);  // Max records per poll
props.put(ConsumerConfig.FETCH_MIN_BYTES_CONFIG, 1);     // Min bytes to fetch
props.put(ConsumerConfig.FETCH_MAX_WAIT_MS_CONFIG, 500); // Max wait time

// Session management
props.put(ConsumerConfig.SESSION_TIMEOUT_MS_CONFIG, 30000);
props.put(ConsumerConfig.HEARTBEAT_INTERVAL_MS_CONFIG, 10000);
props.put(ConsumerConfig.MAX_POLL_INTERVAL_MS_CONFIG, 300000);
```

### 5.3. Manual Offset Commit

```java
Properties props = new Properties();
props.put(ConsumerConfig.ENABLE_AUTO_COMMIT_CONFIG, false);

try (KafkaConsumer<String, String> consumer = new KafkaConsumer<>(props)) {
    consumer.subscribe(Collections.singletonList("order-events"));

    while (true) {
        ConsumerRecords<String, String> records = consumer.poll(Duration.ofMillis(1000));

        for (ConsumerRecord<String, String> record : records) {
            try {
                processOrder(record.value());

                // Commit after successful processing
                consumer.commitSync(Collections.singletonMap(
                    new TopicPartition(record.topic(), record.partition()),
                    new OffsetAndMetadata(record.offset() + 1)
                ));
            } catch (Exception e) {
                // Handle error - message will be reprocessed
                log.error("Error processing message", e);
            }
        }
    }
}

// Or batch commit
while (true) {
    ConsumerRecords<String, String> records = consumer.poll(Duration.ofMillis(1000));

    for (ConsumerRecord<String, String> record : records) {
        processOrder(record.value());
    }

    // Commit all processed in this batch
    consumer.commitSync();
}
```

---

## 6. Message Delivery Guarantees

### 6.1. Delivery Semantics

```
┌─────────────────────────────────────────────────────────────────┐
│                    DELIVERY GUARANTEES                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   AT-MOST-ONCE                                                  │
│   ├── Commit offset before processing                           │
│   ├── May lose messages on failure                              │
│   └── Use: Low-value data, logging                              │
│                                                                  │
│   AT-LEAST-ONCE                                                 │
│   ├── Commit offset after processing                            │
│   ├── May duplicate on failure                                  │
│   └── Use: Most business cases (with idempotency)               │
│                                                                  │
│   EXACTLY-ONCE                                                  │
│   ├── Transactional producer + consumer                         │
│   ├── No duplicates, no losses                                  │
│   └── Use: Financial, critical data                             │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 6.2. Implementing Exactly-Once

```java
// Producer with transactions
Properties producerProps = new Properties();
producerProps.put(ProducerConfig.ENABLE_IDEMPOTENCE_CONFIG, true);
producerProps.put(ProducerConfig.TRANSACTIONAL_ID_CONFIG, "order-processor-1");

KafkaProducer<String, String> producer = new KafkaProducer<>(producerProps);
producer.initTransactions();

try {
    producer.beginTransaction();

    // Send multiple messages atomically
    producer.send(new ProducerRecord<>("topic-a", "key", "value1"));
    producer.send(new ProducerRecord<>("topic-b", "key", "value2"));

    producer.commitTransaction();
} catch (Exception e) {
    producer.abortTransaction();
    throw e;
}

// Consumer with read-committed
Properties consumerProps = new Properties();
consumerProps.put(ConsumerConfig.ISOLATION_LEVEL_CONFIG, "read_committed");
```

---

## 7. Serialization

### 7.1. JSON Serializer

```java
public class JsonSerializer<T> implements Serializer<T> {

    private final ObjectMapper objectMapper = new ObjectMapper();

    @Override
    public byte[] serialize(String topic, T data) {
        if (data == null) return null;
        try {
            return objectMapper.writeValueAsBytes(data);
        } catch (JsonProcessingException e) {
            throw new SerializationException("Error serializing to JSON", e);
        }
    }
}

public class JsonDeserializer<T> implements Deserializer<T> {

    private final ObjectMapper objectMapper = new ObjectMapper();
    private Class<T> targetType;

    public JsonDeserializer(Class<T> targetType) {
        this.targetType = targetType;
    }

    @Override
    public T deserialize(String topic, byte[] data) {
        if (data == null) return null;
        try {
            return objectMapper.readValue(data, targetType);
        } catch (IOException e) {
            throw new SerializationException("Error deserializing from JSON", e);
        }
    }
}

// Usage
Properties props = new Properties();
props.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, JsonSerializer.class.getName());

// Producer
KafkaProducer<String, Order> producer = new KafkaProducer<>(props);
producer.send(new ProducerRecord<>("orders", "order-123", new Order()));
```

### 7.2. Avro with Schema Registry

```xml
<dependency>
    <groupId>io.confluent</groupId>
    <artifactId>kafka-avro-serializer</artifactId>
    <version>7.5.0</version>
</dependency>
```

```java
Properties props = new Properties();
props.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG,
    "io.confluent.kafka.serializers.KafkaAvroSerializer");
props.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG,
    "io.confluent.kafka.serializers.KafkaAvroSerializer");
props.put("schema.registry.url", "http://localhost:8081");
```

---

## 8. Partitioning Strategies

### 8.1. Default Partitioner

```java
// Key-based partitioning (default)
// Same key → Same partition → Ordered
producer.send(new ProducerRecord<>("orders", "customer-123", order));

// Round-robin (when key is null)
producer.send(new ProducerRecord<>("orders", order));
```

### 8.2. Custom Partitioner

```java
public class CustomerPartitioner implements Partitioner {

    @Override
    public int partition(String topic, Object key, byte[] keyBytes,
            Object value, byte[] valueBytes, Cluster cluster) {

        List<PartitionInfo> partitions = cluster.partitionsForTopic(topic);
        int numPartitions = partitions.size();

        if (key == null) {
            // Round-robin for null keys
            return ThreadLocalRandom.current().nextInt(numPartitions);
        }

        String keyStr = key.toString();

        // VIP customers → Dedicated partition
        if (keyStr.startsWith("VIP-")) {
            return 0;
        }

        // Hash-based for others
        return Math.abs(keyStr.hashCode()) % numPartitions;
    }

    @Override
    public void close() {}

    @Override
    public void configure(Map<String, ?> configs) {}
}

// Register
props.put(ProducerConfig.PARTITIONER_CLASS_CONFIG, CustomerPartitioner.class.getName());
```

---

## 9. Error Handling

### 9.1. Producer Errors

```java
producer.send(record, (metadata, exception) -> {
    if (exception != null) {
        if (exception instanceof RetriableException) {
            // Kafka will retry automatically
            log.warn("Retriable error, Kafka will retry", exception);
        } else if (exception instanceof RecordTooLargeException) {
            // Message too large - won't retry
            log.error("Message too large", exception);
            // Store in dead letter queue
            deadLetterQueue.send(record);
        } else {
            log.error("Non-retriable error", exception);
        }
    }
});
```

### 9.2. Consumer Errors

```java
while (true) {
    try {
        ConsumerRecords<String, String> records = consumer.poll(Duration.ofMillis(1000));

        for (ConsumerRecord<String, String> record : records) {
            try {
                processOrder(record.value());
            } catch (TransientException e) {
                // Retry later
                log.warn("Transient error, will retry", e);
                retryTopic.send(record);
            } catch (PermanentException e) {
                // Dead letter queue
                log.error("Permanent error, sending to DLQ", e);
                deadLetterTopic.send(record);
            }
        }

        consumer.commitSync();

    } catch (WakeupException e) {
        // Shutdown signal
        break;
    } catch (Exception e) {
        log.error("Error in consumer loop", e);
    }
}
```

---

## 10. Bài tập thực hành

### Bài 1: Setup Kafka
Setup Kafka cluster với Docker, tạo topics, test với CLI.

### Bài 2: Producer/Consumer
Viết Java producer và consumer, test message flow.

### Bài 3: Consumer Groups
Test parallel processing với consumer groups.

### Bài 4: Error Handling
Implement error handling với retry và dead letter queue.

---

## 11. Key Takeaways

| Concept | Description |
|---------|-------------|
| Topic | Named stream of records |
| Partition | Ordered, append-only log |
| Offset | Position in partition |
| Consumer Group | Parallel processing |
| Replication | Fault tolerance |
| Acks | Delivery guarantee level |

```
Kafka Best Practices:
1. Use appropriate number of partitions (throughput)
2. Configure acks=all for reliability
3. Enable idempotence for exactly-once
4. Use manual commits for at-least-once
5. Implement dead letter queues
6. Monitor consumer lag
```

---

## Navigation

- [← Day 10: Microservices Testing](./day-10-microservices-testing.md)
- [Day 12: Spring Kafka →](./day-12-spring-kafka.md)
