# Kafka Architecture

## Overview

Kafka's architecture is designed for high performance, durability, and scalability. Understanding the internal architecture is crucial for designing effective systems and optimizing performance.

## Cluster Architecture

### Brokers

A Kafka cluster is made up of multiple brokers. These are individual servers that are responsible for:
- Storing data (partitions)
- Serving clients (producers and consumers)
- Replicating data for durability
- Coordinating with other brokers

**Scaling**:
- More brokers → more storage capacity
- More brokers → more client connections
- More brokers → better fault tolerance

### Topics and Partitions

**Topics**:
- Logical grouping of partitions
- Each event is associated with a topic
- Producer publishes to topics, consumers subscribe to topics
- Topics are always multi-producer (multiple producers can write)

**Partitions**:
- Each partition is an ordered, immutable sequence of messages
- Continually appended to, like a log file
- Partition allows for scaling as messages can be consumed in parallel
- Partitions are physical grouping of messages (for scaling)
- A topic can have multiple partitions, each on a different broker

**Partition Benefits**:
- **Parallelism**: Multiple consumers can process different partitions
- **Scalability**: Add partitions to scale throughput
- **Ordering**: Messages within a partition are ordered
- **Distribution**: Partitions distributed across brokers

## Message Structure

### Record Components

A message (referred to as a record) consists of:

1. **Key** (optional):
   - Used to determine which partition the message is sent to
   - Messages with same key go to same partition
   - Enables partition-level ordering

2. **Value** (mandatory):
   - The actual message data
   - Can be any format (JSON, Avro, Protobuf, etc.)
   - **Size Recommendation**: Keep messages under 1MB for optimal performance
   - Configurable via `message.max.bytes` but not recommended to increase

3. **Timestamp** (optional):
   - Used to order messages
   - Can be set by producer or broker

4. **Headers** (optional):
   - Metadata associated with message
   - Key-value pairs for additional information

### Message Size Considerations

**Recommended Size**:
- Keep messages under 1MB for optimal performance
- Larger messages cause:
  - Increased memory pressure
  - Poor network utilization
  - Reduced throughput

**Anti-Pattern: Storing Large Blobs**:
- **Don't**: Store large files/videos directly in Kafka messages
- **Do**: Store large data in distributed file storage (S3, HDFS) and put reference/location in Kafka
- **Example**: YouTube video transcoding
  - ❌ Bad: Put video file in Kafka message
  - ✅ Good: Store video in S3, put S3 location in Kafka message

**Why This Matters**:
- Kafka is not a database or file storage system
- It's designed for small messages that can be processed quickly
- Large messages defeat the purpose of high-throughput streaming

### Partition Determination

**With Key**:
- Kafka hashes the message key to assign message to a specific partition
- Uses murmur2 hash function by default
- Partition calculation: `partition = hash(key) % num_partitions`
- Messages with the same key always go into the same partition
- Keeps partition-level order for messages with same key

**Without Key**:
- Kafka randomly assigns the message a partition (round-robin)
- Or follows another partition configuration set by producer
- No ordering guarantee across messages
- Guarantees even distribution across partitions

**Key Choice Importance**:
- Choose key based on ordering requirements
- Same key → same partition → ordered processing
- Different keys → different partitions → parallel processing
- **Critical**: Bad key choice can lead to hot partitions (uneven distribution)

### Broker Assignment

- Kafka identifies the broker that is holding the particular partition
- Mapping of partitions to brokers is managed by Kafka cluster metadata
- Kafka controller manages partition assignments
- Partitions are distributed across brokers for load balancing

## Append-Only Log Design

Each partition is an append-only log. This design provides:

### Immutability

- Once written, messages cannot be altered or deleted
- Simplifies replication, recovery and avoids consistency issues
- Enables efficient reads (no locking needed)

### Efficiency

- Append only to the end reduces disk seek time
- Disk seeks are usually a bottleneck in many storage systems
- Sequential writes are much faster than random writes

### Scalability

- More partitions can be added and distributed across multiple brokers
- Each partition can be replicated across multiple brokers for fault tolerance
- Horizontal scaling by adding brokers and partitions

## Offset System

### Offset Concept

Each message in a Kafka partition is assigned a unique offset, which is:
- A sequential identifier indicating the message's position in the partition
- Starts at 0 and increments monotonically
- Unique per partition (not globally unique)
- Immutable once assigned

### Offset Usage

- Used by consumers to track their progress in reading messages from the topic
- As consumers read messages, they maintain their current offset
- Periodically commit this offset back to Kafka
- Can resume reading from where they left off in case of failure or restart

### Offset Storage

- Offsets stored in `__consumer_offsets` topic (internal Kafka topic)
- Partitioned by consumer group ID
- Replicated for durability
- Compacted topic (keeps only latest offset per partition)

## Replication

### Leader-Follower Model

Kafka ensures durability of messages through a leader-follower replication mechanism.

**Leader Replica Assignment**:
- Each partition has a designated leader replica which resides on a broker
- This leader replica is responsible for all reads/writes for the partition
- Leader replica is managed centrally by the cluster controller
- Controller ensures that each partition's leader replica is evenly distributed across the cluster to balance the load

**Follower Replicas**:
- Several follower replicas that do not handle any requests
- Passively replicate data from the leader
- They will take over the leader when it fails
- Followers fetch data from leader and apply to their log

**Synchronization and Consistency**:
- Followers continually sync with leader to ensure they have the updated logs
- Once a leader fails, a follower which is fully sync can quickly take over
- Minimizes data loss and downtime

### Controller

The controller within the Kafka cluster manages the replication process:
- Monitors the health of all brokers
- Manages leaders and replication
- When a broker fails, the controller reassigns the leader to one of the followers
- Ensures availability and load balancing

**Controller Responsibilities**:
- Leader election
- Partition assignment
- Replica management
- Broker failure detection

## Consumer Reading Patterns

### Pull Model (Default)

- Consumer polls Kafka for new messages at regular intervals
- Consumer controls when to fetch messages
- More efficient (consumer pulls when ready)
- Can batch multiple messages per fetch

**Advantages**:
- Consumer controls rate
- Can batch efficiently
- Prevents overwhelming slow consumers

### Push Model (Conceptual)

- Receives messages as they arrive
- Server pushes messages to consumer
- Not natively supported by Kafka (uses pull with short intervals)
- Can be implemented with very short poll intervals

**Trade-offs**:
- May overwhelm consumer
- Less efficient batching
- Consumer has less control

## Internal Message Flow

### Producer Flow

1. Producer formats a message (record) with key, value, timestamp, headers
2. Producer determines partition (based on key or round-robin)
3. Producer sends message to broker holding that partition
4. Broker (leader) appends message to partition log
5. Broker replicates to followers (if replication configured)
6. Broker sends acknowledgment to producer

### Consumer Flow

1. Consumer polls broker for messages
2. Consumer specifies topic and partition to read from
3. Consumer specifies starting offset (or uses committed offset)
4. Broker returns messages from that offset
5. Consumer processes messages
6. Consumer commits offset (manually or automatically)

## What Makes Kafka Fast

Understanding the fundamental design decisions that make Kafka high-performance is crucial for system design interviews. Kafka achieves millions of messages per second through several key architectural choices.

### 1. Append-Only Log (Sequential Writes)

**The Foundation of Kafka's Speed**:
- All writes are sequential appends to the end of the log
- No random disk seeks (which are slow)
- Sequential writes are **10-100x faster** than random writes on spinning disks
- Even on SSDs, sequential writes are significantly faster

**Why Sequential Writes are Fast**:
- Disk head doesn't need to move (no seek time)
- OS can optimize sequential I/O
- Better utilization of disk bandwidth
- Predictable I/O patterns

**Performance Impact**:
- Traditional databases: Random writes for updates → slow
- Kafka: Sequential appends only → fast
- This single design decision is one of the biggest performance wins

### 2. Zero-Copy Transfers

**What is Zero-Copy**:
- Technique to transfer data without copying between kernel and user space
- Data stays in kernel space, reducing CPU overhead
- Uses `sendfile()` system call (Linux) or similar

**How It Works**:
```
Traditional Approach:
1. Read from disk to kernel buffer
2. Copy from kernel buffer to user space
3. Copy from user space to kernel buffer (socket)
4. Send to network

Zero-Copy Approach:
1. Read from disk to kernel buffer
2. Send directly from kernel buffer to network (no user space copy)
```

**Performance Impact**:
- Eliminates unnecessary memory copies
- Reduces CPU usage
- Reduces memory bandwidth
- Faster data transfer to consumers

**When Zero-Copy is Used**:
- Consumer reads from disk
- Replication between brokers
- Any time data is read from disk and sent over network

### 3. Page Cache Utilization

**Leveraging OS Page Cache**:
- Kafka relies heavily on OS page cache (not JVM heap)
- Frequently accessed data stays in RAM
- OS manages cache eviction (LRU)
- No manual cache management needed

**How It Works**:
- When Kafka writes, data goes to page cache first
- OS flushes to disk asynchronously
- When Kafka reads, data may already be in page cache
- If in cache, read is from memory (very fast)

**Benefits**:
- Automatic caching by OS
- No application-level cache management
- Leverages OS optimizations
- Memory-efficient (shared across processes)

**Performance Impact**:
- Hot data served from memory (nanoseconds)
- Cold data from disk (milliseconds)
- Significant performance boost for frequently accessed partitions

### 4. Batching

**Batching Messages**:
- Multiple messages sent in single network request
- Reduces network round-trips
- Amortizes overhead across many messages

**Batching Benefits**:
- **Network Efficiency**: Fewer TCP packets, less overhead
- **Compression**: Better compression ratios with more data
- **Throughput**: Can send thousands of messages per request
- **Latency Trade-off**: Slight increase in latency for much better throughput

**Configuration**:
```java
props.put("batch.size", 16384); // Batch size in bytes
props.put("linger.ms", 10); // Wait up to 10ms to fill batch
```

**Performance Impact**:
- Without batching: 1 message = 1 network request
- With batching: 1000 messages = 1 network request
- **100-1000x reduction in network overhead**

### 5. Compression

**Message Compression**:
- Compress messages before sending over network
- Reduces network bandwidth usage
- Faster transmission (less data to send)

**Compression Types**:
- **snappy**: Good balance (fast compression, moderate ratio)
- **gzip**: Better compression (slower)
- **lz4**: Fastest compression
- **zstd**: Best overall (good compression, good speed)

**Compression Benefits**:
- **Network Bandwidth**: 2-10x reduction in data size
- **Storage**: Less disk space needed
- **Throughput**: Can send more messages per second
- **Cost**: Lower network and storage costs

**Trade-off**:
- CPU overhead for compression/decompression
- Usually worth it (network is often the bottleneck)

### 6. Pull Model (Consumer-Controlled)

**Why Pull is Faster Than Push**:
- Consumer controls consumption rate
- Consumer can batch requests efficiently
- No backpressure issues (consumer pulls when ready)
- Server doesn't need to track consumer state

**Pull Model Benefits**:
- **Batching**: Consumer can request many messages at once
- **Rate Control**: Consumer controls how fast to consume
- **Efficiency**: No wasted network bandwidth
- **Simplicity**: Server doesn't need complex push logic

**Performance Impact**:
- Consumer can request 1MB of messages in single request
- Much more efficient than server pushing individual messages
- Reduces network overhead significantly

### 7. Partition Parallelism

**Horizontal Scaling Through Partitions**:
- Multiple partitions allow parallel processing
- Each partition can be on different broker
- Consumers can read from multiple partitions simultaneously
- Linear scaling with number of partitions

**Parallelism Benefits**:
- **Read Parallelism**: Multiple consumers read different partitions
- **Write Parallelism**: Multiple producers write to different partitions
- **No Contention**: Partitions are independent
- **Linear Scaling**: 10 partitions = ~10x throughput

**Performance Impact**:
- Single partition: Limited by single broker
- Multiple partitions: Throughput scales linearly
- Can achieve millions of messages/sec with enough partitions

### 8. Efficient Serialization

**Binary Protocol**:
- Kafka uses efficient binary protocol (not JSON/XML)
- Compact message format
- Fast serialization/deserialization

**Serialization Formats**:
- **Avro**: Compact, schema evolution
- **Protobuf**: Very efficient, widely used
- **JSON**: Human-readable but slower
- **Custom**: Can use any format

**Performance Impact**:
- Binary formats: 2-10x faster than text formats
- Less CPU overhead
- Smaller message size (less network bandwidth)

### 9. Network Efficiency

**Efficient Network Usage**:
- Batch multiple messages per request
- Compress messages
- Reuse connections (connection pooling)
- Efficient protocol design

**Network Optimizations**:
- **Connection Reuse**: Don't create new connection per request
- **Pipelining**: Multiple requests in flight
- **Compression**: Reduce data sent over network
- **Batching**: Amortize network overhead

**Performance Impact**:
- Network is often the bottleneck
- These optimizations significantly improve throughput
- Can saturate network bandwidth efficiently

### 10. No Random Disk Seeks

**The Disk Seek Problem**:
- Random disk seeks are extremely slow (milliseconds)
- Sequential reads/writes are fast (microseconds)
- Traditional databases: Many random seeks for updates
- Kafka: Only sequential operations

**Why This Matters**:
- **Spinning Disks**: Random seek = `10-20ms, Sequential = <1ms`
- **SSDs**: Random seek = `100-200μs, Sequential = <10μs`
- **Difference**: 10-100x faster for sequential operations

**Kafka's Approach**:
- All writes: Sequential append
- All reads: Sequential read from offset
- No updates: Immutable log
- No deletes: Log compaction or retention

**Performance Impact**:
- This is one of the biggest performance wins
- Enables high throughput even on spinning disks
- Makes Kafka suitable for high-volume scenarios

### 11. Immutability Simplifies Everything

**Immutability Benefits**:
- No locking needed (no concurrent modifications)
- No transaction overhead
- Simple concurrency model
- Enables efficient replication

**Performance Impact**:
- **No Locks**: No lock contention, no waiting
- **No Transactions**: No transaction overhead
- **Simple Reads**: Just read sequentially, no consistency checks
- **Fast Replication**: Just copy log segments

### 12. Efficient Replication

**Replication Design**:
- Replication is just copying log segments
- No complex consistency protocols during normal operation
- Followers fetch from leader sequentially
- Efficient network usage

**Replication Performance**:
- Sequential reads from leader
- Sequential writes to followers
- Batch replication (multiple messages at once)
- No random I/O

### Combined Performance Impact

**These optimizations work together**:
- Sequential writes → Fast disk I/O
- Zero-copy → Low CPU overhead
- Page cache → Hot data in memory
- Batching → Efficient network usage
- Compression → Less data to transfer
- Partition parallelism → Linear scaling
- Pull model → Consumer efficiency

**Result**: Kafka can achieve:
- **Write Throughput**: Millions of messages per second
- **Read Throughput**: Very high (limited by network/CPU)
- **Latency**: Low (1-5ms typical)
- **Scalability**: Linear with partitions

## Performance Characteristics

### Write Performance

- **Throughput**: Millions of messages per second (depends on hardware)
- **Latency**: Typically 1-5ms (depends on replication and acks)
- **Batching**: Significantly improves throughput
- **Compression**: Reduces network usage

### Read Performance

- **Throughput**: Very high (can read from multiple partitions in parallel)
- **Latency**: Typically `<1ms` for local reads (if in page cache)
- **Sequential Reads**: Very efficient (append-only log)
- **Parallelism**: Multiple consumers can read different partitions

### Scalability

**Single Broker Constraints**:
- On good hardware, a single broker can store around **1TB of data**
- Can handle approximately **1M messages per second** (depends on message size and hardware)
- If your design doesn't require more than this, scaling may not be relevant

**Scaling Strategies**:

1. **Horizontal Scaling with More Brokers**:
   - Simplest way to scale Kafka
   - Distributes load and offers greater fault tolerance
   - Each broker handles portion of traffic
   - **Important**: Ensure topics have sufficient partitions to take advantage of additional brokers
   - If under-partitioned, won't benefit from new brokers

2. **Partitioning Strategy**:
   - Main scaling decision in interviews
   - Choose appropriate key for messages
   - Partition = `hash(key) % num_partitions`
   - Bad key choice → hot partitions (uneven distribution)
   - Good keys are evenly distributed across partition space

**Scaling Topics vs Cluster**:
- Usually think about scaling topics rather than entire cluster
- Different topics have different requirements
- High-throughput topic: many partitions across many brokers
- Low-throughput topic: few partitions, single broker sufficient
- To scale a topic: increase number of partitions

## Summary

Kafka's architecture provides:

- **High Performance**: Append-only log, batching, compression
- **Durability**: Replication, persistent storage
- **Scalability**: Horizontal scaling, partition-based parallelism
- **Flexibility**: Multiple producers, multiple consumers, multiple consumer groups
- **Ordering**: Partition-level ordering guarantees

Understanding the architecture helps in:
- Designing efficient systems
- Optimizing performance
- Handling failures
- Scaling effectively

## Hot Partitions

### The Problem

**What is a Hot Partition**:
- A partition that receives significantly more traffic than others
- Causes uneven load distribution
- Can become a bottleneck
- Example: Ad click aggregator partitioned by ad ID - when Nike launches popular ad, that partition gets overwhelmed

### Strategies to Handle Hot Partitions

**1. Random Partitioning (No Key)**:
- Don't provide a key, Kafka randomly assigns partition
- Guarantees even distribution
- **Downside**: Lose ability to guarantee message order
- **Use When**: Ordering not important, even distribution critical

**2. Random Salting**:
- Add random number or timestamp to key when generating partition key
- Example: `partition_key = ad_id + random_number`
- Helps distribute load more evenly
- **Downside**: Complicates aggregation logic on consumer side
- **Use When**: Need some ordering but want better distribution

**3. Compound Key**:
- Use combination of attributes instead of single key
- Example: `partition_key = ad_id + geographic_region`
- Distributes traffic more evenly
- Particularly useful if attributes vary independently
- **Use When**: Can identify attributes that vary independently

**4. Back Pressure**:
- Slow down producer if partition lag too high
- Producer checks lag on partition and throttles if needed
- Managed services may have built-in mechanisms
- Self-hosted: implement custom back pressure logic
- **Use When**: Can tolerate slower production rate

**Key Insight**: Partition strategy is the main scaling decision. Choose keys that distribute evenly while maintaining necessary ordering guarantees.
