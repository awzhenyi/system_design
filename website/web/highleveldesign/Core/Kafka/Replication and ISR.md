# Replication and In-Sync Replicas (ISR)

## Overview

Kafka's replication mechanism ensures durability and availability of messages. Understanding In-Sync Replicas (ISR), replication lag, and leader election is crucial for designing reliable systems.

## Replication Basics

### Leader-Follower Model

Kafka ensures durability of messages through a leader-follower replication mechanism.

**Leader Replica**:
- Each partition has a designated leader replica which resides on a broker
- This leader replica is responsible for all reads/writes for the partition
- Leader replica is managed centrally by the cluster controller
- Controller ensures that each partition's leader replica is evenly distributed across the cluster to balance the load

**Follower Replicas**:
- Several follower replicas that do not handle any requests
- Passively replicate data from the leader
- They will take over the leader when it fails
- Followers fetch data from leader and apply to their log

**Replication Factor**:
- Number of replicas (leader + followers) for each partition
- Typically 3 (1 leader + 2 followers)
- Higher replication factor = better durability, more storage

### Replication Process

1. Producer sends message to leader
2. Leader appends message to its log
3. Leader replicates to followers
4. Followers acknowledge replication
5. Leader sends acknowledgment to producer (if acks=all)

## In-Sync Replicas (ISR)

### ISR Concept

**What is ISR**:
- Set of replicas (leader + followers) that are "in-sync" with leader
- Replicas that have caught up with leader's log
- Only ISR members can become leader on failure

**ISR Membership Criteria**:
- Replica must be in-sync with leader
- Replica must be actively replicating (not lagging)
- Replica must respond to leader within `replica.lag.time.max.ms` (default 10 seconds)

### ISR Management

**How ISR is Maintained**:
- Leader tracks which followers are in-sync
- Follower sends fetch requests to leader
- If follower doesn't fetch within `replica.lag.time.max.ms`, removed from ISR
- If follower catches up, added back to ISR

**ISR Shrinking**:
- Follower falls behind (network issues, slow disk)
- Leader removes follower from ISR
- Partition continues with smaller ISR
- If ISR size < `min.insync.replicas`, writes may be rejected

**ISR Expansion**:
- Follower catches up with leader
- Leader adds follower back to ISR
- ISR size increases

### Producer Acknowledgment and ISR

**acks Configuration**:

1. **acks=0** (No acknowledgment):
   - Producer doesn't wait for acknowledgment
   - Highest throughput, lowest durability
   - No guarantee message is written

2. **acks=1** (Leader acknowledgment):
   - Producer waits for leader to write
   - Good balance of throughput and durability
   - May lose messages if leader fails before replication

3. **acks=all** (All ISR acknowledgment):
   - Producer waits for all ISR replicas to acknowledge
   - Highest durability, lower throughput
   - Requires `min.insync.replicas` ISR members

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

### min.insync.replicas

**Purpose**:
- Minimum number of ISR replicas required for writes to succeed
- Prevents writes when too few replicas are in-sync
- Ensures durability guarantees

**Configuration**:
```properties
# Broker configuration
min.insync.replicas=2
```

**Behavior**:
- If ISR size < `min.insync.replicas`, writes with `acks=all` are rejected
- Producer receives `NotEnoughReplicasException`
- Prevents data loss but may cause unavailability

**Example**:
- Topic with replication factor 3
- `min.insync.replicas=2`
- If 2 replicas fail, only 1 ISR remains
- Writes with `acks=all` are rejected (ISR size 1 < 2)
- Topic becomes read-only until replicas recover

**Trade-offs**:
- Higher `min.insync.replicas`: Better durability, but more likely to reject writes
- Lower `min.insync.replicas`: More availability, but less durability guarantee

## Replication Lag

### What is Replication Lag

**Definition**:
- Difference between leader's log end offset (LEO) and follower's LEO
- Measured in number of messages or bytes
- Indicates how far behind follower is

**Causes of Lag**:
- Slow network between leader and follower
- Slow disk I/O on follower
- High write rate (follower can't keep up)
- Follower broker overloaded

### Monitoring Lag

**Kafka Metrics**:
- `replication-lag` per partition
- JMX: `kafka.server:type=ReplicaFetcherManager,name=MaxLag`
- Should alert if lag exceeds threshold

**Monitoring Tools**:
- Kafka built-in metrics
- External monitoring (Prometheus, Grafana)
- Custom lag monitoring tools

**Impact of High Lag**:
- Follower removed from ISR if lag too high
- Reduced fault tolerance (smaller ISR)
- Slower recovery on leader failure
- Possible data loss if leader fails before follower catches up

### Reducing Replication Lag

**Strategies**:
- Increase `replica.fetch.size` (fetch more data per request)
- Increase `replica.fetch.wait.max.ms` (wait longer for data)
- Improve network between brokers
- Use faster disks on followers
- Reduce write rate (if possible)

## Leader Election

### Leader Election Process

1. Leader fails (detected by controller)
2. Controller selects new leader from ISR
3. Only ISR members eligible (ensures no data loss)
4. If no ISR members available, election may wait or use out-of-sync replica (data loss possible)

### Preferred Leader

**Concept**:
- Each partition has preferred leader (original leader)
- Kafka tries to keep preferred leader as actual leader
- Better load distribution
- `auto.leader.rebalance.enable=true` enables automatic rebalancing

**Benefits**:
- Even load distribution across brokers
- Predictable leader assignments
- Better performance (leaders on preferred brokers)

### Unclean Leader Election

**What is Unclean Leader Election**:
- If `unclean.leader.election.enable=true` (default: false)
- Allows election of out-of-sync replica
- May cause data loss (replica missing messages)
- Only used if no ISR members available

**Configuration**:
```properties
# Broker configuration
unclean.leader.election.enable=false # Prevent data loss
auto.leader.rebalance.enable=true # Prefer preferred leaders
```

**When It Happens**:
- All ISR replicas fail
- No in-sync replicas available
- System must choose: unavailability or data loss

**Recommendation**:
- Keep `unclean.leader.election.enable=false` (default)
- Prefer unavailability over data loss
- Ensure sufficient replication factor and ISR size

## High Watermark and LEO

### Log End Offset (LEO)

**Definition**:
- Offset of next message to be written
- Different for leader and followers
- Leader's LEO advances as messages written
- Follower's LEO advances as it replicates

**LEO Tracking**:
- Leader tracks its own LEO
- Leader tracks each follower's LEO
- Used to determine replication lag

### High Watermark (HW)

**Definition**:
- Highest offset that is replicated to all ISR replicas
- Consumers can only read up to high watermark
- Prevents reading uncommitted messages
- Advances when all ISR replicas acknowledge

**Relationship**:
```
LEO (Leader) >= HW >= LEO (Follower)
```

**Why High Watermark Matters**:
- Ensures consumers don't read messages that may be lost
- If leader fails, new leader's LEO may be lower
- High watermark ensures only committed messages are readable
- Prevents reading uncommitted data

### High Watermark Advancement

**Process**:
1. Leader writes message (LEO advances)
2. Leader replicates to followers
3. Followers acknowledge (their LEO advances)
4. When all ISR replicas acknowledge, HW advances
5. Consumers can now read up to new HW

**Example**:
```
Leader LEO: 100
Follower 1 LEO: 98
Follower 2 LEO: 99
High Watermark: 98 (lowest LEO in ISR)
```

## Replication Configuration

### Key Configuration Parameters

**replica.lag.time.max.ms**:
- Maximum time follower can lag behind leader
- Default: 10 seconds
- If exceeded, follower removed from ISR

**replica.fetch.size**:
- Maximum bytes to fetch per request
- Default: 1MB
- Larger = fewer requests, more memory

**replica.fetch.wait.max.ms**:
- Maximum wait time for fetch request
- Default: 500ms
- Longer = better batching, higher latency

**min.insync.replicas**:
- Minimum ISR size for writes with acks=all
- Default: 1
- Recommended: replication_factor - 1

### Recommended Configuration

```properties
# Replication settings
replica.lag.time.max.ms=10000
replica.fetch.size=1048576
replica.fetch.wait.max.ms=500

# Durability settings
min.insync.replicas=2
unclean.leader.election.enable=false
auto.leader.rebalance.enable=true
```

## Failure Scenarios

### Leader Failure

**Scenario**:
- Leader broker fails
- Controller detects failure
- New leader elected from ISR
- Producers/consumers reconnect to new leader

**Recovery**:
- Fast (typically `<1` second)
- No data loss if ISR members available
- Temporary unavailability during election

### Follower Failure

**Scenario**:
- Follower broker fails
- Removed from ISR
- Partition continues with remaining ISR
- Replica recovers and rejoins ISR

**Impact**:
- Reduced fault tolerance
- If ISR size < min.insync.replicas, writes may be rejected
- No immediate data loss

### Network Partition

**Scenario**:
- Network split between brokers
- Some followers can't reach leader
- Followers removed from ISR
- Partition continues with remaining ISR

**Impact**:
- Reduced ISR size
- Possible write rejection if ISR too small
- Recovers when network heals

## Kafka Availability Philosophy

### "Kafka is Always Available, Sometimes Consistent"

**What This Means**:
- Kafka is designed for high availability
- Question "what happens if Kafka goes down?" is not very realistic
- Kafka cluster failures are rare in well-designed systems
- More relevant question: "what happens if a consumer goes down?"

**Implications**:
- Kafka cluster is highly available through replication
- Individual broker failures don't take down cluster
- Consumer failures are more common and relevant
- Design for consumer failures, not Kafka failures

**When Consumer Goes Down**:
- Kafka's fault tolerance mechanisms ensure continuity
- Offset management: consumer resumes from last committed offset
- Rebalancing: partitions redistributed among remaining consumers
- No messages missed or duplicated (with proper offset management)

## Summary

Kafka's replication and ISR mechanism provides:

- **Durability**: Messages replicated to multiple brokers
- **Availability**: Automatic failover on leader failure
- **Consistency**: High watermark ensures only committed messages readable
- **Fault Tolerance**: Survives broker failures

**Key Takeaways**:
- ISR ensures only up-to-date replicas can become leaders
- `min.insync.replicas` prevents data loss
- Monitor replication lag to ensure ISR health
- High watermark prevents reading uncommitted data
- Configure appropriately for durability vs availability trade-offs
- Kafka is highly available - design for consumer failures, not Kafka failures
