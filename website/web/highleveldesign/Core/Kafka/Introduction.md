# Introduction to Kafka

## What is Kafka?

Kafka is a distributed event streaming platform that is often used as a **message queue** or a **stream processing system**. It excels at high performance, durability and scalability. It is designed to handle vast volume of data in real time, ensuring no message is ever lost.

### Core Characteristics

- **Distributed**: Runs as a cluster of brokers
- **High Performance**: Handles millions of messages per second
- **Durable**: Messages are persisted and replicated
- **Scalable**: Horizontally scalable by adding brokers
- **Real-time**: Low latency message delivery

### Fundamental Concept

The fundamental idea behind Kafka: messages sent and received through Kafka require a user specified distribution strategy. This allows for flexible routing and processing patterns.

## Use Cases

### Real-Time Statistics Updates

**Example**: Football match statistics
- Events are placed into a queue
- Producers put these events into the queue
- Consumers process and update the websites in real-time
- Multiple consumers can process different aspects (scores, player stats, etc.)

### Async Processing

**Example**: YouTube video transcoding
- When users upload a video, standard definition is available immediately
- Video link is put into a Kafka topic for transcoding
- Transcoding happens asynchronously when system has capacity
- Decouples upload from processing

### Ordered Message Processing

**Example**: Virtual waiting queue for Ticketmaster
- Users are allowed into booking page by order
- Messages must be processed in order
- Kafka's partition-level ordering ensures this
- Single partition per queue ensures strict ordering

### Decoupling Producers and Consumers

**Example**: Microservices architecture
- Producers are producing too many messages for consumers to handle
- Kafka acts as buffer between them
- Producers and consumers can be scaled independently
- Prevents backpressure from affecting producers

### Continuous Data Processing

**Example**: Ad clicks aggregator
- Continuous and immediate processing of incoming data
- Real-time flow of click events
- Aggregation happens as messages arrive
- Low latency is critical

### Pub/Sub System

**Example**: Facebook live comments
- Messages need to be processed by multiple consumers simultaneously
- Each consumer group processes independently
- Multiple consumer groups can subscribe to same topic
- Enables fan-out patterns

## Key Concepts

### Topics

- A logical grouping of partitions
- Each event is associated with a topic
- Producer publishes to topics, and consumers subscribe to topics
- Topics are always multi-producer (multiple producers can write to same topic)
- Topics are logical grouping of messages (for organization)
- A topic can have multiple partitions, each on a different broker

### Partitions

- Each partition is an ordered, immutable sequence of messages
- Continually appended to, like a log file
- Partition allows for scaling of Kafka as messages can be consumed in parallel
- Partitions are physical grouping of messages (for scaling)
- Messages with same key go to same partition (ensures ordering)

### Brokers

- A Kafka cluster is made up of multiple brokers
- These are individual servers responsible for storing data and serving clients
- More brokers → more data, clients
- Each broker has multiple partitions
- Brokers coordinate through Zookeeper (or KRaft in newer versions)

### Producers

- Write data/messages to topics
- Can specify partition or let Kafka decide
- Can send messages synchronously or asynchronously
- Support batching for better throughput

### Consumers

- Consumers subscribe to topics they want to listen to
- When scaling consumers, group into a consumer group
- This ensures each event is processed only once by one consumer in the group
- When Kafka is used like a message queue, consumers acknowledge that the message is processed
- When Kafka is used like a stream, consumers will not acknowledge that message has been processed

## When to Use Kafka

**Use Kafka When**:
- Need high throughput message processing
- Need to decouple producers and consumers
- Need ordered message processing (per partition)
- Need to process same messages with multiple consumers
- Need durable message storage
- Need real-time data streaming

**Don't Use Kafka When**:
- Need complex message routing (use message broker)
- Need request-response patterns (use RPC)
- Need very low latency (`<1ms`) (use in-memory queues)
- Need simple task queues (use RabbitMQ, SQS)
- Small scale systems (overhead not worth it)

## Summary

Kafka is a powerful distributed streaming platform that excels at:
- High-throughput message processing
- Decoupling system components
- Real-time data streaming
- Durable message storage
- Scalable architecture

Understanding when and how to use Kafka is crucial for designing scalable distributed systems.
