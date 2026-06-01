# Kafka Producers

## Overview

Producers are responsible for writing data/messages to Kafka topics. Understanding producer behavior, configuration, and delivery semantics is crucial for building reliable systems.

## Producer Basics

### Producer Responsibilities

- Format messages (records) with key, value, timestamp, headers
- Determine target partition (based on key or round-robin)
- Send messages to appropriate broker
- Handle acknowledgments and retries
- Manage batching and compression

### Producer Configuration

**Essential Properties**:
```java
props.put("bootstrap.servers", "localhost:9092");
props.put("key.serializer", "org.apache.kafka.common.serialization.StringSerializer");
props.put("value.serializer", "org.apache.kafka.common.serialization.StringSerializer");
```

**Performance Properties**:
```java
props.put("batch.size", 16384); // Batch size in bytes
props.put("linger.ms", 10); // Wait time for batching
props.put("compression.type", "snappy"); // Compression type
props.put("buffer.memory", 33554432); // Producer buffer size
```

## Message Delivery Semantics

### At-Least-Once Delivery

**How It Works**:
- Producer retries on failure until acknowledgment received
- Messages may be duplicated if producer retries after successful send
- Consumer may process same message multiple times

**Configuration**:
```java
props.put("acks", "all"); // Wait for all in-sync replicas
props.put("retries", Integer.MAX_VALUE); // Retry indefinitely
props.put("max.in.flight.requests.per.connection", 1); // Prevent reordering
```

**Trade-offs**:
- **Pros**: No message loss, simple to implement
- **Cons**: Possible duplicates, requires idempotent consumers

**When to Use**:
- Can tolerate duplicates
- Consumer is idempotent
- Message loss is unacceptable

**Implementation Pattern**:
```java
// Producer with at-least-once guarantees
Producer<String, String> producer = new KafkaProducer<>(props);
ProducerRecord<String, String> record = new ProducerRecord<>("topic", "key", "value");
producer.send(record); // Will retry until acknowledged
```

### At-Most-Once Delivery

**How It Works**:
- Producer sends message once, no retries
- Messages may be lost if send fails
- No duplicates, but possible message loss

**Configuration**:
```java
props.put("acks", "0"); // Fire and forget
props.put("retries", 0); // No retries
```

**Trade-offs**:
- **Pros**: No duplicates, simple
- **Cons**: Possible message loss

**When to Use**:
- Duplicates are worse than message loss
- Can tolerate some message loss
- Metrics/logging use cases

### Exactly-Once Delivery

**How It Works**:
- Combines idempotent producer and transactional consumer
- Ensures each message is delivered exactly once
- Most complex but strongest guarantee

**Components**:

1. **Idempotent Producer**:
   - Producer assigns unique sequence number per partition
   - Broker deduplicates based on producer ID + sequence number
   - Prevents duplicates from producer retries

2. **Transactional Producer**:
   - Producer can send messages atomically across partitions
   - Uses transaction coordinator
   - All-or-nothing semantics

**Configuration**:
```java
// Producer
props.put("enable.idempotence", true); // Enable idempotent producer
props.put("transactional.id", "my-transactional-id"); // Enable transactions
props.put("acks", "all");
props.put("retries", Integer.MAX_VALUE);
props.put("max.in.flight.requests.per.connection", 5); // Can be > 1 with idempotence
```

**Implementation Details**:

**Idempotent Producer**:
- Each producer gets unique Producer ID (PID) from broker
- Producer maintains sequence number per partition
- Broker tracks (PID, partition, sequence) to detect duplicates
- Duplicate sequence numbers are rejected

**Transactional Producer**:
```java
// Initialize transactional producer
Producer<String, String> producer = new KafkaProducer<>(props);
producer.initTransactions();

try {
    producer.beginTransaction();
    producer.send(new ProducerRecord<>("topic1", "key1", "value1"));
    producer.send(new ProducerRecord<>("topic2", "key2", "value2"));
    producer.commitTransaction();
} catch (ProducerFencedException e) {
    producer.close();
} catch (KafkaException e) {
    producer.abortTransaction();
}
```

**Performance Implications**:
- **Overhead**: Additional coordination overhead
- **Latency**: Slightly higher latency (transaction coordination)
- **Throughput**: Slightly lower throughput
- **Complexity**: More complex to implement and debug

**When to Use**:
- Financial transactions
- Critical operations where duplicates are unacceptable
- Exactly-once processing requirements
- When duplicates cause correctness issues

**Limitations**:
- Only works within Kafka ecosystem
- Cross-system exactly-once requires additional coordination
- Performance overhead
- More complex failure handling

## Producer Acknowledgments

### acks Configuration

**acks=0** (No acknowledgment):
- Producer doesn't wait for acknowledgment
- Highest throughput, lowest durability
- No guarantee message is written
- Use for: Metrics, logs where some loss is acceptable

**acks=1** (Leader acknowledgment):
- Producer waits for leader to write
- Good balance of throughput and durability
- May lose messages if leader fails before replication
- Use for: Most use cases, good default

**acks=all** (All ISR acknowledgment):
- Producer waits for all ISR replicas to acknowledge
- Highest durability, lower throughput
- Requires `min.insync.replicas` ISR members
- Use for: Critical data, cannot lose messages

**Configuration**:
```java
props.put("acks", "all"); // Wait for all ISR replicas
// Also configure on broker:
// min.insync.replicas=2 // Minimum ISR size for writes
```

**Trade-offs**:
- **acks=all**: Strongest durability, but slower and may reject writes if ISR too small
- **acks=1**: Good balance, but possible data loss on leader failure
- **acks=0**: Fastest, but no durability guarantee

## Producer Optimizations

### Batching

**Purpose**: Improve throughput by sending multiple messages in one request

**Configuration**:
```java
props.put("batch.size", 16384); // Batch size in bytes
props.put("linger.ms", 10); // Wait up to 10ms to fill batch
```

**How It Works**:
- Producer collects messages into batches
- Sends batch when size threshold or time threshold reached
- Reduces network overhead
- Improves throughput significantly

**Trade-offs**:
- **Pros**: Much higher throughput, lower network overhead
- **Cons**: Slightly higher latency (waiting for batch)

### Compression

**Purpose**: Reduce network bandwidth and storage

**Configuration**:
```java
props.put("compression.type", "snappy"); // or "gzip", "lz4", "zstd"
```

**Compression Types**:
- **snappy**: Good balance of speed and compression
- **gzip**: Better compression, slower
- **lz4**: Fast, moderate compression
- **zstd**: Best compression, good speed

**Trade-offs**:
- **Pros**: Less network bandwidth, less storage
- **Cons**: CPU overhead for compression/decompression

### Idempotence

**Purpose**: Enable exactly-once semantics and allow batching with multiple in-flight requests

**Configuration**:
```java
props.put("enable.idempotence", true);
// Allows max.in.flight.requests.per.connection > 1
props.put("max.in.flight.requests.per.connection", 5);
```

**Benefits**:
- Prevents duplicates from retries
- Allows better batching (multiple in-flight requests)
- Enables exactly-once semantics

## Producer Patterns

### Synchronous Send

```java
ProducerRecord<String, String> record = new ProducerRecord<>("topic", "key", "value");
RecordMetadata metadata = producer.send(record).get(); // Blocks until complete
System.out.println("Sent to partition " + metadata.partition() + " at offset " + metadata.offset());
```

**Use When**: Need to know if send succeeded before continuing

### Asynchronous Send

```java
ProducerRecord<String, String> record = new ProducerRecord<>("topic", "key", "value");
producer.send(record, new Callback() {
    @Override
    public void onCompletion(RecordMetadata metadata, Exception exception) {
        if (exception != null) {
            // Handle error
        } else {
            // Handle success
        }
    }
});
```

**Use When**: Don't need to wait for acknowledgment, better throughput

### Fire and Forget

```java
ProducerRecord<String, String> record = new ProducerRecord<>("topic", "key", "value");
producer.send(record); // Don't wait or handle callback
```

**Use When**: Loss is acceptable, maximum throughput needed

## Error Handling

### Retry Configuration

**Java Configuration**:
```java
props.put("retries", 3); // Number of retries
props.put("retry.backoff.ms", 100); // Wait time between retries
```

**Node.js/kafkajs Configuration**:
```javascript
const producer = kafka.producer({
  retry: {
    retries: 5, // Retry up to 5 times
    initialRetryTime: 100, // Wait 100ms between retries
  },
  idempotent: true, // Enable idempotent producer to avoid duplicates
});
```

### Retriable Errors

- Network errors
- Broker temporarily unavailable
- Leader election in progress
- Transient failures

### Non-Retriable Errors

- Invalid topic
- Invalid message format
- Authorization failures
- Message too large
- Invalid configuration

### Idempotent Producer for Retries

**Why Enable Idempotence**:
- When retries are enabled, duplicates are possible
- Producer may retry after successful send (network issue, delayed ack)
- Idempotent producer ensures messages sent only once
- Prevents duplicates from retry mechanism

**Configuration**:
```java
props.put("enable.idempotence", true); // Enable idempotent producer
props.put("retries", Integer.MAX_VALUE); // Can retry indefinitely with idempotence
```

### Error Handling Pattern

```java
try {
    producer.send(record, new Callback() {
        @Override
        public void onCompletion(RecordMetadata metadata, Exception exception) {
            if (exception != null) {
                if (exception instanceof RetriableException) {
                    // Will be retried automatically
                } else {
                    // Non-retriable, handle manually
                    handleNonRetriableError(exception);
                }
            }
        }
    });
} catch (Exception e) {
    // Handle synchronous errors
}
```

## Summary

Kafka producers provide:

- **Multiple Delivery Semantics**: At-least-once, at-most-once, exactly-once
- **Flexible Configuration**: Tune for throughput, latency, or durability
- **Optimization Options**: Batching, compression, idempotence
- **Error Handling**: Automatic retries, error callbacks

**Key Takeaways**:
- Choose delivery semantics based on requirements
- Configure acks based on durability needs
- Use batching and compression for better performance
- Enable idempotence for exactly-once semantics
- Handle errors appropriately based on use case
