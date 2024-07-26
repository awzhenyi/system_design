# Kafka

## Introduction
- Kafka is an distributed event streaming platform that is often used as a **message queue** or a **stream processing system**.
- It excels at high performance, durability and scalability.
- It is designed to handle vast volume of data in real time, ensuring no message is ever lost.
- fundamental idea behind Kafka: messages sent and received through Kafka require a user specified distribution strategy.

## Use cases
- real time statistics update (football matches). events can be placed into a queue. producers put these events into the queue. consumers process and updates the websites.
- async processing. YouTube is a good example of this. When users upload a video we can make the standard definition video available immediately and then put the video (via link) a Kafka topic to be transcoded when the system has time.
- when you need messages to be processed in order. eg: virtual waiting queue for ticketmaster. users are allowed into booking page by the order.
- decouple producers and consumers so they can be scaled independently. Usually in microservices, producers are producing too much messages that the consumers can handles.
- continous and immediate processing of incoming data, like a real time flow. eg: Ad clicks aggregator. 
- messages need to be processed by multiple consumers simultaneously. eg Facebook live comments can use kafka as a pub sub system to send comments to multiple consumers.

## Brokers
- A kafka cluster is made up of multiple brokers. These are individual servers that is responsible for storing data and serving clients. more brokers -> more data, clients.
- Each broker has multiple partitions. Each partition is an ordered, immutable sequence of messages that is continually appended to, like a log file.
- Partition allows for scaling of kafka as messages can be consumed in parallel.
- Partitions are phyiscal grouping of messages. (for scaling)

## Topics
- A logical grouping of partitions.
- Each event is associated with a topic.
- Producer publishes to topics, and consumers subscribes to topics.
- Topics are always multi producers.
- Topics are logical grouping of messages (for organization). A topic can have multiple partitions, each on a different broker.

## Producers
- write data / message to topics

## Consumers
- consumers can subscribe to which topic they want to listen to.
- when scaling consumers, group into a consumer group. This ensures that each event is processed only once by one of the consumer in the group.
- when kafka is used like a message queue, consumers acknowledges that the message is processed.
- when kafka is used like a stream, consumers will not acknowledges that message has been processed.


## Kafka internal workings
- On event, producer formats a message (referred to as record) and sends it to a kafka topic. A message consists of 3 optional fields: `key`, `timestamp`, `headers` and a mandatory field `value`.
- `key` is used to determine which partition the message is being sent to. `timestamp` is used to order message. `headers` consists of meta data.
- if no `key` is provided, kafka randomly assigns the message a partition. However, choice of `key` is important.
- **Partition Determination** - kafka hashes the message `key` to assign message to a specific partition. kafka can round robin messages without `key` or follow another partition configuration set by producer. messages with the same `key` always go into the same partition, keeping partition-level order.
- **Broker Assignment** kafka identifies the broker that is holding the particular partition. mapping of partitions to brokers is managed by kafka cluster metadata and kafka controller.
- Each parititon is an append-only log. This designs provides:
    * immutability - once written, messages cannot be altered or deleted. This simplifies replication, recovery and avoids consistency issues.
    * Efficiency - append only to the end reduces disk seek time, which are usually a bottleneck in many storage systems.
    * scalability - more partitions can be added and distributed across multiple brokers to handle increased load. Each partition can be replicated across multiple brokers for fault tolerance.
- Each message in a Kafka partition is assigned a unique offset, which is a sequential identifier indicating the message’s position in the partition. This offset is used by consumers to track their progress in reading messages from the topic. As consumers read messages, they maintain their current offset and periodically commit this offset back to Kafka. This way, they can resume reading from where they left off in case of failure or restart.
- **Replication** - kafka ensures durability of messasges through a leader-follower replication mechanism.
    * **Leader Replica Assignment** - each partition has a designated leader replica which resides on a broker. This leader replica is responsible for all reads/writes for the partition. Leader replica is managed centrally by the cluster controller, which ensures that each partition's leader replica is evenly distributed across the cluster to balance the load.
    * **Follower Replications** - several follower replicas that will not handle any requests, but they will passively replicate data from the leader. They will take over the leader when it fails.
    * **Synchronization and Consistency** - followers continually sync with leader to ensure they have the updated logs. Once a leader fails, a follower which is fully sync can quickly take over, minimising data loss and downtime.
    * **Controller** - controller within the kafka cluster manages this replication process. It monitors the health of all brokers and manages leaders and replication. When a broker fails, the controller reassigns the leader to one of the follower to ensure availability.
- **Consumer Reading pattern**
    * **Push Model** - receives messages as they arrive.
    * **Pull Model** - poll kafka for new messages at regular intervals.


