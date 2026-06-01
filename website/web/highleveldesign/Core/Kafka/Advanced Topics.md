# Kafka Advanced Topics

## Overview

This document covers advanced Kafka topics including transactional messaging, performance optimization, failure handling, and operational considerations for staff-level engineers.

## Transactional Messaging

### Use Case

- Exactly-once processing across multiple topics
- Atomic writes to multiple partitions
- Exactly-once semantics for stream processing
- Ensuring consistency across distributed operations

### How It Works

1. Producer begins transaction
2. Producer sends messages (marked as part of transaction)
3. Producer commits or aborts transaction
4. Transaction coordinator manages transaction state
5. Consumers with `isolation.level=read_committed` only see committed messages

### Transaction Coordinator

**Responsibilities**:
- Manages transaction state
- One coordinator per producer
- Coordinates with partition leaders
- Handles transaction timeouts

**Transaction State**:
- Tracks which messages are part of transaction
- Manages commit/abort decisions
- Ensures atomicity across partitions

### Transaction Lifecycle

```
BEGIN TRANSACTION
  → Send messages to partitions
  → All messages marked with transaction ID
COMMIT TRANSACTION
  → Transaction coordinator commits
  → All messages become visible
  → Or ABORT → All messages discarded
```

### Implementation

**Producer Configuration**:
```java
props.put("transactional.id", "my-transactional-id");
props.put("enable.idempotence", true);
props.put("acks", "all");
```

**Producer Usage**:
```java
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

**Consumer Configuration**:
```java
props.put("isolation.level", "read_committed"); // Only read committed messages
props.put("enable.auto.commit", false); // Manual offset commits
```

**Consumer Usage**:
```java
consumer.beginTransaction();
try {
    ConsumerRecords<String, String> records = consumer.poll(Duration.ofMillis(100));
    for (ConsumerRecord<String, String> record : records) {
        process(record);
    }
    consumer.sendOffsetsToTransaction(offsets, consumerGroupId);
    consumer.commitTransaction();
} catch (Exception e) {
    consumer.abortTransaction();
}
```

### Limitations

- Only works within Kafka ecosystem
- Cross-system transactions require additional coordination
- Performance overhead
- More complex failure handling
- Transaction coordinator is single point of coordination

## Performance Optimization

### Producer Optimizations

**Batching**:
```java
props.put("batch.size", 16384); // Batch size in bytes
props.put("linger.ms", 10); // Wait time for batching
```
- Collects multiple messages into batches
- Reduces network overhead
- Significantly improves throughput
- Trade-off: Slightly higher latency

**Batching Configuration Example** (Node.js/kafkajs):
```javascript
const producer = kafka.producer({
  batch: {
    maxSize: 16384, // Maximum batch size in bytes
    maxTime: 100,   // Maximum time to wait before sending a batch (ms)
  }
});
```

**Compression**:
```java
props.put("compression.type", "snappy"); // or "gzip", "lz4", "zstd"
```
- Reduces network bandwidth
- Reduces storage requirements
- CPU overhead for compression
- Types: snappy (balanced), gzip (best compression), lz4 (fastest), zstd (best overall)

**Compression Configuration Example** (Node.js/kafkajs):
```javascript
const producer = kafka.producer({
  compression: CompressionTypes.GZIP,
});
```

**Idempotence**:
```java
props.put("enable.idempotence", true);
props.put("max.in.flight.requests.per.connection", 5); // Can be > 1 with idempotence
```
- Enables batching with multiple in-flight requests
- Prevents duplicates
- Better throughput

**Buffer Configuration**:
```java
props.put("buffer.memory", 33554432); // 32MB buffer
```
- Larger buffer = better batching
- More memory usage
- Better for high-throughput scenarios

### Consumer Optimizations

**Fetch Size**:
```java
props.put("fetch.min.bytes", 1024); // Minimum bytes to fetch
props.put("fetch.max.wait.ms", 500); // Maximum wait time
```
- Larger fetch = fewer requests
- Better throughput
- Higher latency

**Parallel Processing**:
- Multiple consumer threads per consumer
- Multiple consumers in consumer group
- Process different partitions in parallel

**Offset Management**:
- Balance between commit frequency and duplicate risk
- Manual commits for better control
- Batch commits for better performance

### Broker Optimizations

**Replication**:
```properties
replica.fetch.size=1048576 # 1MB
replica.fetch.wait.max.ms=500
```
- Tune fetch size and wait time
- Balance replication lag and throughput

**Log Retention**:
```properties
retention.ms=604800000 # 7 days
retention.bytes=1073741824 # 1GB
```
- Configure based on requirements
- Balance storage and retention needs

**Compaction**:
```properties
log.cleanup.policy=compact
```
- Enable log compaction for keyed topics
- Keeps only latest value per key
- Reduces storage for update-heavy topics

**Segment Size**:
```properties
log.segment.bytes=1073741824 # 1GB
```
- Larger segments = fewer files
- Better for high-throughput
- Slower recovery

## Failure Scenarios and Handling

### Producer Failures

**Network Failures**:
- Producer retries automatically
- Configure retry count and backoff
- Handle retriable vs non-retriable errors

**Broker Failures**:
- Producer fails over to another broker
- Automatic broker discovery
- Temporary unavailability possible

**Transaction Failures**:
- Producer aborts and retries
- Handle ProducerFencedException
- Reinitialize producer if needed

**Error Handling Pattern**:
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

### Consumer Failures

**Consumer Crash**:
- Offset not committed, reprocesses on restart
- Partitions reassigned to other consumers in group
- No message loss (messages still in Kafka)

**Network Failure**:
- Consumer reconnects, continues from last committed offset
- May reprocess some messages (if offset not committed)

**Processing Failure**:
- Don't commit offset, reprocess on restart
- Or handle error and continue (commit offset)
- Consider dead letter queue for persistent failures

**Error Handling Pattern**:
```java
try {
    ConsumerRecords<String, String> records = consumer.poll(Duration.ofMillis(100));
    for (ConsumerRecord<String, String> record : records) {
        try {
            process(record);
        } catch (Exception e) {
            // Handle per-message error
            handleError(record, e);
            // Options: log and continue, send to DLQ, retry
        }
    }
    consumer.commitSync(); // Commit if all processed successfully
} catch (Exception e) {
    // Handle batch error
    log.error("Batch processing failed", e);
    // Don't commit - will reprocess
}
```

### Broker Failures

**Leader Failure**:
- New leader elected from ISR
- Producers/consumers reconnect automatically
- Temporary unavailability during election
- No data loss if ISR members available

**Follower Failure**:
- Removed from ISR
- Partition continues with remaining ISR
- Replica recovers and rejoins ISR
- Reduced fault tolerance during failure

**Controller Failure**:
- New controller elected
- Temporary unavailability during election
- No data loss

**Partition Unavailability**:
- If no ISR members available, partition becomes unavailable
- Reads and writes fail
- Recovers when replicas come back online
- Consider unclean leader election (not recommended)

## Operational Considerations

### Monitoring

**Key Metrics**:
- **Producer**: Throughput, latency, error rate
- **Consumer**: Lag, throughput, commit rate
- **Broker**: Disk usage, network I/O, replication lag
- **Topic**: Message rate, partition count, replication status

**Monitoring Tools**:
- Kafka built-in JMX metrics
- External monitoring (Prometheus, Grafana)
- Kafka Manager, Confluent Control Center
- Custom monitoring solutions

### Capacity Planning

**Partition Count**:
- More partitions = more parallelism
- But more overhead (files, connections)
- Typical: 10-100 partitions per topic
- Consider consumer parallelism needs

**Replication Factor**:
- Higher = better durability
- But more storage and network
- Typical: 3 (1 leader + 2 followers)
- Critical data: 5 or more

**Retention**:
- Balance storage cost and requirements
- Consider compaction for update-heavy topics
- Monitor disk usage

### Retention Policies

**Default Retention**:
- Default retention policy: **7 days**
- Configurable via `retention.ms` (time-based) and `retention.bytes` (size-based)

**Configuration**:
```properties
retention.ms=604800000 # 7 days in milliseconds
retention.bytes=1073741824 # 1GB in bytes
```

**When to Extend Retention**:
- Need to store messages for longer period
- Replay/reprocessing requirements
- Compliance/audit requirements
- Historical data analysis needs

**Trade-offs**:
- Longer retention = more storage costs
- Longer retention = more disk usage
- Longer retention = slower recovery (more data to scan)
- Balance based on requirements and costs

**Retention Strategies**:
- **Time-based**: Delete messages older than X days
- **Size-based**: Delete oldest messages when topic exceeds size
- **Compaction**: Keep only latest value per key (for keyed topics)
- **Hybrid**: Combine time and size limits

### Troubleshooting

**Common Issues**:
- **High Consumer Lag**: Slow processing, need more consumers
- **Replication Lag**: Network or disk issues
- **Partition Unavailability**: Not enough ISR members
- **High Latency**: Network, disk, or configuration issues

**Debugging Tools**:
- `kafka-console-producer.sh` / `kafka-console-consumer.sh`
- `kafka-consumer-groups.sh`
- `kafka-topics.sh`
- JMX metrics
- Logs

## Best Practices

### Producer Best Practices

1. **Use Appropriate acks**: Balance durability and performance
2. **Enable Idempotence**: For exactly-once semantics
3. **Use Batching**: For better throughput
4. **Handle Errors**: Retriable vs non-retriable
5. **Monitor Metrics**: Throughput, latency, errors

### Consumer Best Practices

1. **Use Consumer Groups**: For parallel processing
2. **Manual Commits**: For better control
3. **Handle Failures**: Don't commit on error
4. **Monitor Lag**: Critical for real-time systems
5. **Idempotent Processing**: Handle duplicates

### Topic Design Best Practices

1. **Partition Count**: Based on consumer parallelism
2. **Replication Factor**: 3 for most cases
3. **Retention**: Based on requirements
4. **Compaction**: For keyed topics with updates
5. **Naming**: Clear, consistent naming conventions

## Summary

Advanced Kafka topics include:

- **Transactional Messaging**: Exactly-once across topics
- **Performance Optimization**: Batching, compression, tuning
- **Failure Handling**: Producer, consumer, broker failures
- **Operational Considerations**: Monitoring, capacity planning, troubleshooting

**Key Takeaways for Staff-Level Engineers**:
- Understand transactional messaging for exactly-once semantics
- Optimize for your use case (throughput vs latency)
- Design for failures (idempotency, error handling)
- Monitor critical metrics (lag, throughput, errors)
- Plan capacity appropriately (partitions, replication, retention)
