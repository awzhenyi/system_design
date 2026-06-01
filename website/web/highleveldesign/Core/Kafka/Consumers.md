# Kafka Consumers

## Overview

Consumers read messages from Kafka topics. Understanding consumer behavior, offset management, and consumer groups is crucial for building reliable and scalable systems.

## Consumer Basics

### Consumer Responsibilities

- Subscribe to topics
- Poll for messages from assigned partitions
- Process messages
- Commit offsets to track progress
- Handle failures and rebalancing

### Consumer Configuration

**Essential Properties**:
```java
props.put("bootstrap.servers", "localhost:9092");
props.put("group.id", "my-consumer-group");
props.put("key.deserializer", "org.apache.kafka.common.serialization.StringDeserializer");
props.put("value.deserializer", "org.apache.kafka.common.serialization.StringDeserializer");
```

**Offset Management Properties**:
```java
props.put("auto.offset.reset", "earliest"); // or "latest" or "none"
props.put("enable.auto.commit", true); // or false for manual commits
props.put("auto.commit.interval.ms", 5000); // Commit interval
```

## Consumer Groups

### Concept

A consumer group is a set of consumers working together to consume a topic:
- Each partition consumed by only one consumer in group
- Enables parallel processing and load distribution
- Ensures each message processed once per consumer group

### Consumer Group Benefits

- **Parallelism**: Multiple consumers process different partitions
- **Scalability**: Add consumers to scale processing
- **Fault Tolerance**: If consumer fails, partitions reassigned to others
- **Load Distribution**: Partitions distributed across consumers

### Consumer Group Example

```
Topic: orders (3 partitions)
Consumer Group: order-processors (2 consumers)

Consumer 1: Processes partitions 0, 1
Consumer 2: Processes partition 2
```

## Offset Management

### Offset Concept

**What is an Offset**:
- Sequential identifier for each message in a partition
- Starts at 0 and increments monotonically
- Unique per partition (not globally unique)
- Immutable once assigned

**Offset Storage**:
- Offsets stored in `__consumer_offsets` topic (internal Kafka topic)
- Partitioned by consumer group ID
- Replicated for durability
- Compacted topic (keeps only latest offset per partition)

### Consumer Offset Commits

**Commit Strategies**:

1. **Automatic Commit (Default)**:
   ```java
   props.put("enable.auto.commit", true);
   props.put("auto.commit.interval.ms", 5000); // Commit every 5 seconds
   ```
   - Consumer automatically commits offsets periodically
   - Simple but can cause duplicate processing
   - Offsets committed even if processing fails

2. **Manual Commit (Synchronous)**:
   ```java
   props.put("enable.auto.commit", false);
   // After processing
   consumer.commitSync(); // Blocks until commit succeeds
   ```
   - Consumer explicitly commits offsets
   - Blocks until commit confirmed
   - Ensures offset committed before continuing
   - Slower but safer

3. **Manual Commit (Asynchronous)**:
   ```java
   props.put("enable.auto.commit", false);
   // After processing
   consumer.commitAsync((offsets, exception) -> {
       if (exception != null) {
           // Handle commit failure
       }
   });
   ```
   - Non-blocking commit
   - Faster but no guarantee of success
   - Must handle commit failures

**Best Practice - At-Least-Once with Manual Commit**:
```java
try {
    ConsumerRecords<String, String> records = consumer.poll(Duration.ofMillis(100));
    for (ConsumerRecord<String, String> record : records) {
        processRecord(record);
    }
    consumer.commitSync(); // Commit after successful processing
} catch (Exception e) {
    // On error, don't commit - will reprocess on restart
    log.error("Processing failed", e);
}
```

### Offset Reset Policies

**What Happens When No Offset Exists**:

1. **earliest**:
   - Start from beginning of partition
   - Read all messages from offset 0
   - Use for: Reprocessing all data, new consumer groups

2. **latest** (Default):
   - Start from end of partition
   - Only read new messages
   - Use for: Real-time processing, don't need history

3. **none**:
   - Throw exception if no offset found
   - Use for: Fail-fast on misconfiguration

**Configuration**:
```java
props.put("auto.offset.reset", "earliest"); // or "latest" or "none"
```

### Offset Management Patterns

**Pattern 1: Process and Commit**:
```java
while (true) {
    ConsumerRecords<String, String> records = consumer.poll(Duration.ofMillis(100));
    for (ConsumerRecord<String, String> record : records) {
        process(record);
    }
    consumer.commitSync(); // Commit after batch processing
}
```

**Pattern 2: Per-Message Commit**:
```java
while (true) {
    ConsumerRecords<String, String> records = consumer.poll(Duration.ofMillis(100));
    for (ConsumerRecord<String, String> record : records) {
        process(record);
        consumer.commitSync(Collections.singletonMap(
            new TopicPartition(record.topic(), record.partition()),
            new OffsetAndMetadata(record.offset() + 1)
        )); // Commit each message
    }
}
```

**Pattern 3: Transactional Commit**:
```java
props.put("isolation.level", "read_committed");
// Process and commit atomically
consumer.beginTransaction();
// ... process messages ...
consumer.sendOffsetsToTransaction(offsets, consumerGroupId);
consumer.commitTransaction();
```

### Offset Tracking and Monitoring

**Key Metrics**:
- **Consumer Lag**: Difference between latest offset and committed offset
- **Offset Commit Rate**: How often offsets are committed
- **Offset Commit Failures**: Number of failed commits

**Monitoring Tools**:
- `kafka-consumer-groups.sh`: View consumer group offsets and lag
- JMX metrics: `records-lag-max`, `records-consumed-rate`
- Burrow (LinkedIn): Consumer lag monitoring

**Common Issues**:
- **Stale Offsets**: Consumer lagging behind (processing too slow)
- **Offset Reset**: Consumer reset to beginning (reprocessing)
- **Commit Failures**: Network issues, broker failures

## Consumer Groups and Rebalancing

### Rebalancing Triggers

- Consumer joins group
- Consumer leaves group (failure, shutdown)
- New partition added to topic
- Consumer group metadata changes

### Rebalancing Process

1. Coordinator detects need for rebalance
2. All consumers stop processing
3. Coordinator assigns partitions to consumers
4. Consumers resume processing with new assignments

### Rebalancing Strategies

1. **Range** (Default):
   - Assigns consecutive partitions to consumers
   - Simple but can be uneven
   - Example: Partitions 0-2 to consumer 1, 3-5 to consumer 2

2. **Round Robin**:
   - Distributes partitions evenly
   - Better load distribution
   - Example: Partition 0 to consumer 1, 1 to consumer 2, 2 to consumer 1, etc.

3. **Sticky**:
   - Minimizes partition movement
   - Better for stateful consumers
   - Reduces rebalancing overhead

4. **Cooperative Sticky** (Incremental):
   - Incremental rebalancing
   - Consumers don't all stop
   - Better performance

**Configuration**:
```java
props.put("partition.assignment.strategy", 
    "org.apache.kafka.clients.consumer.CooperativeStickyAssignor");
```

**Rebalancing Impact**:
- **Stop-the-world**: All consumers stop during rebalance (old behavior)
- **Incremental**: Only affected consumers stop (new behavior with cooperative)
- Can cause processing delays
- Should minimize rebalancing frequency

### Consumer Group Coordinator

- One broker acts as coordinator for each consumer group
- Manages consumer group membership
- Handles rebalancing
- Tracks offsets

## Consumer Reading Patterns

### Pull Model (Default)

- Consumer polls Kafka for new messages at regular intervals
- Consumer controls when to fetch messages
- More efficient (consumer pulls when ready)
- Can batch multiple messages per fetch

**Polling Pattern**:
```java
while (true) {
    ConsumerRecords<String, String> records = consumer.poll(Duration.ofMillis(100));
    for (ConsumerRecord<String, String> record : records) {
        process(record);
    }
}
```

**Advantages**:
- Consumer controls rate
- Can batch efficiently
- Prevents overwhelming slow consumers

### Fetch Configuration

**Fetch Size**:
```java
props.put("fetch.min.bytes", 1); // Minimum bytes to fetch
props.put("fetch.max.wait.ms", 500); // Maximum wait time
```

**Trade-offs**:
- Larger fetch size: Better throughput, higher latency
- Smaller fetch size: Lower latency, more requests

## Consumer Optimizations

### Parallel Processing

**Multiple Consumer Threads**:
```java
ExecutorService executor = Executors.newFixedThreadPool(10);
while (true) {
    ConsumerRecords<String, String> records = consumer.poll(Duration.ofMillis(100));
    for (ConsumerRecord<String, String> record : records) {
        executor.submit(() -> process(record));
    }
}
```

**Multiple Consumers**:
- Run multiple consumer instances
- Each in same consumer group
- Partitions distributed across consumers

### Fetch Optimization

**Batching**:
```java
props.put("fetch.min.bytes", 1024); // Wait for at least 1KB
props.put("fetch.max.wait.ms", 500); // Or wait up to 500ms
```

**Benefits**:
- Fewer network requests
- Better throughput
- Lower broker load

## Transactional Consumers

### Exactly-Once Processing

**Configuration**:
```java
props.put("isolation.level", "read_committed"); // Only read committed messages
props.put("enable.auto.commit", false); // Manual offset commits
```

**How It Works**:
- Consumer reads only committed messages
- Commits offsets atomically with processing
- If processing fails, offset not committed (reprocess on restart)
- Works with transactional producers

**Implementation**:
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

## Failure Handling

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

### Consumer Retries and Dead Letter Queues

**Kafka Doesn't Support Built-in Retries**:
- Unlike AWS SQS, Kafka doesn't have built-in retry mechanism
- Must implement custom retry logic

**Retry Pattern**:
1. Set up custom retry topic for failed messages
2. Move failed messages to retry topic
3. Separate consumer processes retry topic
4. Retry with exponential backoff
5. After max retries, move to Dead Letter Queue (DLQ)

**Dead Letter Queue (DLQ)**:
- Topic for messages that failed after all retries
- Allows investigation of persistent failures
- Prevents retry loop from blocking main consumer
- Common pattern in production systems

**Implementation Pattern**:
```java
try {
    process(record);
} catch (Exception e) {
    int retryCount = getRetryCount(record);
    if (retryCount < MAX_RETRIES) {
        // Send to retry topic with incremented retry count
        sendToRetryTopic(record, retryCount + 1);
    } else {
        // Send to DLQ after max retries
        sendToDLQ(record, e);
    }
}
```

**Alternative: Use SQS for Built-in Retries**:
- Some systems use SQS instead of Kafka for built-in retry/DLQ
- Example: Web Crawler design may prefer SQS for this reason
- Trade-off: Lose Kafka's high throughput for simpler retry handling

**Best Practice: Minimize Consumer Work**:
- More work consumer does → more likely to redo work if consumer fails
- **Best Practice**: Keep consumer work minimal, break into phases if needed
- **Example: Web Crawler Design**:
  - Phase 1: Download HTML (quick, commit offset)
  - Phase 2: Parse HTML (slower, separate topic)
  - This minimizes work lost if consumer fails during parsing

### Error Handling Pattern

```java
try {
    ConsumerRecords<String, String> records = consumer.poll(Duration.ofMillis(100));
    for (ConsumerRecord<String, String> record : records) {
        try {
            process(record);
        } catch (Exception e) {
            // Handle per-message error
            // Options: log and continue, send to DLQ, retry
            handleError(record, e);
        }
    }
    consumer.commitSync(); // Commit if all processed successfully
} catch (Exception e) {
    // Handle batch error
    log.error("Batch processing failed", e);
    // Don't commit - will reprocess
}
```

## Summary

Kafka consumers provide:

- **Consumer Groups**: Parallel processing and load distribution
- **Offset Management**: Track progress, resume after failures
- **Rebalancing**: Automatic partition reassignment
- **Multiple Patterns**: Automatic/manual commits, transactional

**Key Takeaways**:
- Use consumer groups for parallel processing
- Choose appropriate commit strategy based on requirements
- Monitor consumer lag
- Handle failures gracefully
- Optimize fetch configuration for your use case
- Use transactional consumers for exactly-once processing
