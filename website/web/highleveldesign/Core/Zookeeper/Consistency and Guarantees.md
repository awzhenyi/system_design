# Zookeeper Consistency and Guarantees

## Overview

Understanding Zookeeper's consistency model and guarantees is crucial for designing distributed systems correctly. This document covers the formal guarantees, their implications, and how to design applications that leverage these guarantees effectively.

## CAP Theorem

### CP System Explanation

Zookeeper is a **CP (Consistency + Partition Tolerance) system**, meaning it prioritizes:

- **Consistency**: All clients see the same data at the same time
- **Partition Tolerance**: System continues operating despite network partitions

**Trade-off**: During network partitions, Zookeeper may become unavailable to maintain consistency.

### Why CP, Not CA or AP?

**CA Systems** (Consistency + Availability):
- Cannot handle network partitions
- Must stop operating during partitions
- Not suitable for distributed systems
- Example: Traditional single-master databases

**AP Systems** (Availability + Partition Tolerance):
- Sacrifice consistency for availability
- Continue operating during partitions
- May serve stale or inconsistent data
- Example: DynamoDB, Cassandra

**CP Systems** (Consistency + Partition Tolerance):
- Sacrifice availability for consistency
- Stop operating if cannot maintain consistency
- Always serve consistent data
- Example: Zookeeper, etcd

**Why CP for Coordination**:
- Coordination requires consistency (can't have two leaders)
- Temporary unavailability acceptable for coordination
- Inconsistency is worse than unavailability for coordination

### Detailed Analysis of Consistency vs Availability Trade-offs

**Consistency Benefits**:
- No conflicting decisions
- Predictable behavior
- Easier to reason about
- Correct coordination

**Availability Costs**:
- May be unavailable during partitions
- Minority partition stops serving
- Must wait for quorum

**When Consistency Matters**:
- Leader election (only one leader)
- Distributed locks (only one holder)
- Configuration (all nodes see same config)
- Critical coordination decisions

**When Availability Matters More**:
- User-facing data (can show stale data)
- Non-critical operations
- Eventually consistent is acceptable
- High availability required

### What Happens During Network Partitions

**Scenario: 5-Node Cluster, 3+2 Partition**:

1. **Partition Occurs**:
   - 3 nodes in partition A
   - 2 nodes in partition B

2. **Partition A (Majority)**:
   - Has quorum (3 out of 5)
   - Continues serving requests
   - Can elect new leader if needed
   - Accepts writes

3. **Partition B (Minority)**:
   - No quorum (2 out of 5)
   - Stops serving requests
   - Cannot accept writes
   - Clients connected to this partition cannot perform operations

4. **Partition Heals**:
   - Network reconnects
   - Minority synchronizes with majority
   - All nodes have consistent state
   - Cluster resumes normal operation

**Key Insight**: Only the majority partition continues operating. This prevents split-brain scenarios where both partitions make conflicting decisions.

### Why CP is the Right Choice for Coordination Services

**Coordination Requirements**:
- Must prevent conflicting decisions
- Must maintain global state consistently
- Temporary unavailability is acceptable
- Inconsistency is catastrophic

**Examples**:
- **Leader Election**: Can't have two leaders
- **Distributed Locks**: Can't have two holders
- **Configuration**: All nodes must see same config
- **Group Membership**: Must have consistent view

**If AP Instead**:
- Two leaders could be elected
- Two processes could hold same lock
- Nodes could see different configs
- Coordination would fail

### Comparison with AP Systems

**DynamoDB (AP System)**:
- Prioritizes availability
- May serve stale reads
- Eventually consistent
- Good for user-facing data

**Zookeeper (CP System)**:
- Prioritizes consistency
- Always serves consistent data
- May be unavailable during partitions
- Good for coordination

**When to Use Which**:
- **AP Systems**: User data, non-critical operations, high availability needed
- **CP Systems**: Coordination, critical decisions, consistency required

## Consistency Guarantees

### Sequential Consistency

**Definition**: All operations appear to execute in some sequential order that is consistent with the order seen by each individual client.

**What This Means**:
- All clients see operations in the same order
- Each client's operations appear in program order
- Global ordering of all operations

**Implications**:
- No race conditions
- Predictable behavior
- Easier to reason about
- Enables coordination primitives

**Example**:
```
Client A: write("/app/config", "v1")
Client B: write("/app/config", "v2")
Client C: read("/app/config")

All clients see the same order:
1. A writes v1
2. B writes v2
3. C reads v2 (not v1)
```

### Atomicity

**Definition**: All operations are atomic - they either complete entirely or not at all.

**What This Means**:
- No partial updates
- All-or-nothing semantics
- Transaction-like behavior

**Implications**:
- Safe concurrent updates
- No corrupted state
- Can use for coordination

**Example**:
```java
// This is atomic
zk.setData("/app/config", "new_value".getBytes(), version);

// Either:
// - Update succeeds completely, OR
// - Update fails completely (no partial update)
```

**Multi-Operation Atomicity**:
```java
// Transaction: all operations succeed or all fail
List<Op> ops = Arrays.asList(
    Op.create("/app/config", data1, ...),
    Op.setData("/app/version", data2, ...),
    Op.delete("/app/old", ...)
);
zk.multi(ops); // Atomic: all or nothing
```

### Single-System Image

**Definition**: A client will see the same view of the service regardless of which server it connects to.

**What This Means**:
- All servers have same data
- No server-specific views
- Consistent across servers

**Implications**:
- Can connect to any server
- Load balancing works
- No server affinity needed

**How It's Achieved**:
- All writes replicated to all servers
- Consensus ensures all servers agree
- All servers apply same transactions

### Reliability

**Definition**: If a change is applied to one server, it will be applied to all servers that are functioning properly.

**What This Means**:
- Changes are durable
- Survive server failures
- Replicated to all servers

**Implications**:
- No data loss (for committed writes)
- High durability
- Can recover from failures

**Durability Guarantees**:
- Committed writes are on majority of servers
- Survive individual server failures
- Recoverable from transaction log

### Timeliness

**Definition**: A client's view of the system is guaranteed to be up-to-date within a bounded time.

**What This Means**:
- Changes propagate quickly
- Bounded staleness
- Not eventually consistent (stronger)

**Implications**:
- Fast updates
- Low latency
- Real-time coordination possible

**Bounded Staleness**:
- Changes visible within seconds
- Not minutes or hours
- Suitable for coordination

### Formal Definitions and Practical Implications

**Formal Model**:
- Linearizability: All operations appear to execute atomically at some point between invocation and response
- Sequential consistency: Operations appear to execute in some sequential order
- Causal consistency: Maintains causality relationships

**Practical Implications**:
- Can build coordination primitives
- Can reason about system behavior
- Can implement distributed algorithms
- Can ensure correctness

## Ordering Guarantees

### Write Ordering

**Global Ordering**:
- All writes are totally ordered
- Order determined by zxid (transaction ID)
- All servers see same order

**How It's Maintained**:
- Leader assigns zxid to each write
- Zxid is monotonically increasing
- All servers apply writes in zxid order

**Example**:
```
Write 1: zxid = 100, write("/app/config", "v1")
Write 2: zxid = 101, write("/app/config", "v2")
Write 3: zxid = 102, write("/app/config", "v3")

All servers see: 100 → 101 → 102
```

**Implications**:
- No race conditions
- Deterministic behavior
- Enables coordination

### Read Ordering

**Read-Your-Writes Consistency**:
- Client sees its own writes immediately
- Subsequent reads see previous writes

**Monotonic Reads**:
- Client never sees older data after seeing newer data
- Reads are monotonic (never go backwards)

**Example**:
```
Client A:
1. write("/app/config", "v1")
2. read("/app/config") → "v1" (sees own write)

Client B:
1. read("/app/config") → "v1" (after A's write)
2. read("/app/config") → "v1" or "v2" (never goes back to older)
```

**How It's Achieved**:
- Reads from local state (fast)
- Or consistent reads (sync with leader)
- Sequential consistency ensures ordering

### Causal Ordering

**Definition**: If operation A causally precedes operation B, all clients see A before B.

**Causality Relationships**:
- If client writes X then reads X, write causally precedes read
- If client reads X then writes Y based on X, read causally precedes write
- Transitive: if A → B and B → C, then A → C

**Example**:
```
Client A:
1. write("/app/config", "v1")
2. write("/app/version", "1") // Based on config

All clients see:
1. config = v1
2. version = 1 (after config)
```

**Implications**:
- Maintains logical dependencies
- Prevents causality violations
- Enables correct distributed algorithms

### Implementation Details (Transaction IDs, zxid)

**Zxid Structure**:
- 64-bit number: (epoch, counter)
- Epoch: Increments on leader election
- Counter: Increments for each transaction

**Example**:
```
Epoch 1: zxid = 0x0000000100000001 (epoch 1, counter 1)
Epoch 1: zxid = 0x0000000100000002 (epoch 1, counter 2)
Leader election
Epoch 2: zxid = 0x0000000200000001 (epoch 2, counter 1)
```

**Ordering Guarantees**:
- Zxid provides global ordering
- All servers see same zxid order
- Enables sequential consistency

### When Ordering is Critical vs When It's Not

**Critical Ordering**:
- Leader election (order determines leader)
- Distributed locks (order determines lock holder)
- Configuration updates (order matters for correctness)
- State machine replication (order is critical)

**Non-Critical Ordering**:
- Independent operations
- Idempotent operations
- Operations on different znodes (if independent)
- Read operations (usually)

**Design Implications**:
- Use ordering when it matters
- Don't rely on ordering when it doesn't
- Understand when operations are independent

### Examples of Ordering Guarantees in Practice

**Example 1: Leader Election**:
```
Candidate 1: creates /election/candidate-0000000001
Candidate 2: creates /election/candidate-0000000002
Candidate 3: creates /election/candidate-0000000003

All see same order: 1, 2, 3
Leader is candidate 1 (lowest number)
```

**Example 2: Distributed Lock**:
```
Process A: creates /locks/resource-0000000001
Process B: creates /locks/resource-0000000002

All see: A gets lock first, then B
Order is deterministic
```

**Example 3: Configuration Update**:
```
Update 1: set /app/config = "v1" (zxid 100)
Update 2: set /app/config = "v2" (zxid 101)

All servers see: v1 → v2 (never v2 → v1)
```

## Durability

### Persistence Guarantees

**Transaction Log**:
- All writes appended to transaction log
- Log is persistent (on disk)
- Survives server crashes

**Snapshots**:
- Periodic snapshots of in-memory state
- Used for fast recovery
- Combined with log for full recovery

**Durability Levels**:
- **Committed Writes**: Durable on majority of servers
- **Uncommitted Writes**: May be lost on failure
- **Snapshot + Log**: Full state recoverable

### What Happens on Server Crashes

**Scenario 1: Follower Crashes**:
- Leader continues with remaining quorum
- Follower's state becomes stale
- On recovery, follower synchronizes with leader
- No data loss (majority has all data)

**Scenario 2: Leader Crashes**:
- New leader elected
- New leader has all committed transactions
- Uncommitted transactions may be lost
- Committed transactions are durable

**Scenario 3: Multiple Server Crashes**:
- If quorum maintained, cluster continues
- If quorum lost, cluster stops
- On recovery, synchronize with majority
- No data loss if quorum maintained

### Recovery Procedures and Data Integrity

**Recovery Process**:
1. Load latest snapshot
2. Replay transactions from log after snapshot
3. Synchronize with other servers if needed
4. Verify data integrity

**Data Integrity**:
- Snapshots are atomic
- Transaction log is append-only
- Recovery ensures consistency
- No corruption possible

**Verification**:
- Checksums on data
- Transaction log verification
- Snapshot verification
- Cross-server consistency checks

### Durability vs Performance Trade-offs

**High Durability**:
- Sync transaction log to disk (fsync)
- Slower writes (disk I/O)
- Better durability

**Lower Durability**:
- Async transaction log writes
- Faster writes
- Risk of data loss on crash

**Zookeeper Default**:
- Balances durability and performance
- Configurable fsync policy
- Typically: sync every N transactions or time interval

### fsync Policies and Their Implications

**Policy 1: Sync Every Write**:
- Maximum durability
- Slowest performance
- Use for critical data

**Policy 2: Sync Every N Writes**:
- Balance durability and performance
- Some risk of data loss
- Typical configuration

**Policy 3: Sync on Time Interval**:
- Good performance
- Bounded data loss window
- Common configuration

**Recommendation**: Use policy 2 or 3 for most use cases, policy 1 only for critical data.

## Trade-offs

### Consistency vs Availability

**The Fundamental Trade-off**:
- Cannot have both perfect consistency and perfect availability during partitions
- Must choose which to prioritize

**Zookeeper's Choice**:
- Prioritizes consistency
- Sacrifices availability during partitions
- Right choice for coordination

**When to Accept Weaker Consistency**:
- User-facing data (can show stale)
- Non-critical operations
- High availability required
- Use AP systems instead

**When Consistency is Critical**:
- Coordination primitives
- Critical decisions
- Configuration management
- Use CP systems (Zookeeper)

### Consistency vs Performance

**Strong Consistency Costs**:
- Consensus overhead
- Network round-trips
- Slower operations
- Limited throughput

**Weaker Consistency Benefits**:
- No consensus needed
- Local reads
- Faster operations
- Higher throughput

**Zookeeper's Approach**:
- Strong consistency for writes
- Fast local reads (weaker consistency)
- Balance based on operation type

**Optimization Strategies**:
- Use local reads when possible
- Batch writes for better throughput
- Accept eventual consistency for non-critical data
- Use observers for read scaling

### When to Accept Weaker Consistency

**Acceptable Scenarios**:
- Read operations (can be slightly stale)
- Non-critical configuration
- Monitoring data
- Cached data

**Not Acceptable**:
- Leader election
- Distributed locks
- Critical configuration
- Coordination decisions

**Design Pattern**:
- Use strong consistency where needed
- Use weaker consistency where acceptable
- Balance based on requirements

### Design Patterns for Working with Zookeeper's Consistency Model

**Pattern 1: Optimistic Concurrency**:
```java
// Read current state
Stat stat = zk.exists(path, false);
byte[] data = zk.getData(path, false, stat);

// Modify locally
String newData = modify(data);

// Update only if unchanged
try {
    zk.setData(path, newData.getBytes(), stat.getVersion());
} catch (BadVersionException e) {
    // Retry or handle conflict
}
```

**Pattern 2: Eventual Consistency for Reads**:
```java
// Use local reads (fast, may be slightly stale)
byte[] data = zk.getData(path, false, null);

// Use consistent reads only when needed
byte[] consistentData = zk.getData(path, true, null); // sync with leader
```

**Pattern 3: Batch Operations**:
```java
// Batch multiple operations for better performance
List<Op> ops = Arrays.asList(
    Op.create(path1, data1, ...),
    Op.setData(path2, data2, ...),
    Op.delete(path3, ...)
);
zk.multi(ops); // Atomic batch
```

**Pattern 4: Read-Heavy Optimization**:
```java
// Cache reads locally
private Map<String, CachedData> cache = new HashMap<>();

public String getConfig(String key) {
    CachedData cached = cache.get(key);
    if (cached != null && !cached.isStale()) {
        return cached.data;
    }
    
    // Refresh from Zookeeper
    byte[] data = zk.getData("/app/config/" + key, false, null);
    cache.put(key, new CachedData(data, System.currentTimeMillis()));
    return new String(data);
}
```

## Limitations

### What Zookeeper Doesn't Guarantee

**1. Eventual Consistency Aspects**:
- Reads may be slightly stale (local reads)
- Not all reads see latest writes immediately
- Consistent reads available but slower

**2. Network Partition Behavior**:
- Minority partition becomes unavailable
- Cannot serve requests during partition
- Must wait for partition to heal

**3. Performance Guarantees**:
- No guaranteed throughput
- Latency can vary
- Write throughput limited (~20K/sec)

**4. Data Size Limits**:
- 1MB per znode limit
- Not suitable for large data
- Not a general-purpose data store

**5. Query Capabilities**:
- No SQL-like queries
- No indexes
- Must know exact paths
- No search capabilities

**6. Multi-Datacenter**:
- Limited multi-DC support
- Better for single datacenter
- Cross-DC latency issues

### Eventual Consistency Aspects

**Local Reads**:
- May be slightly stale
- Don't see latest committed writes immediately
- Fast but not strongly consistent

**Consistent Reads**:
- See all committed writes
- Slower (sync with leader)
- Use when needed

**When Staleness Matters**:
- Critical coordination (use consistent reads)
- Configuration (local reads usually OK)
- Monitoring (local reads usually OK)

### Stale Reads and How to Handle Them

**Stale Read Scenarios**:
- Follower serves read from local state
- Leader has newer data not yet replicated
- Read may return stale data

**Handling Stale Reads**:
```java
// Option 1: Accept staleness (default, fast)
byte[] data = zk.getData(path, false, null);

// Option 2: Use consistent read (slower, always fresh)
byte[] data = zk.getData(path, true, null); // sync with leader

// Option 3: Check version and retry if changed
Stat stat = new Stat();
byte[] data = zk.getData(path, false, stat);
int version = stat.getVersion();
// ... use data ...
// Later, check if version changed
Stat newStat = zk.exists(path, false);
if (newStat.getVersion() != version) {
    // Data changed, reload
    data = zk.getData(path, false, null);
}
```

### Network Partition Scenarios and Behavior

**Scenario 1: Clean Partition (3+2)**:
- Majority continues
- Minority stops
- No split-brain

**Scenario 2: Uneven Partition (4+1)**:
- 4-node partition has quorum
- 1-node partition stops
- No issues

**Scenario 3: Equal Partition (2+2 in 4-node)**:
- Neither has quorum
- Both stop
- Cluster unavailable

**Scenario 4: Multiple Partitions**:
- Only majority partition continues
- Others stop
- Heal when reconnected

**Key Behavior**:
- Only majority can operate
- Prevents split-brain
- Ensures consistency

### Known Limitations and Workarounds

**Limitation 1: Write Throughput**:
- Limited to ~20K writes/sec
- **Workaround**: Batch operations, reduce write frequency

**Limitation 2: Data Size**:
- 1MB per znode limit
- **Workaround**: Store references, use external storage

**Limitation 3: Multi-Datacenter**:
- Limited cross-DC support
- **Workaround**: Per-DC clusters, application-level coordination

**Limitation 4: Query Capabilities**:
- No SQL, no indexes
- **Workaround**: Know paths, use hierarchical organization

**Limitation 5: Stale Reads**:
- Local reads may be stale
- **Workaround**: Use consistent reads when needed

## Failure Scenarios

### Network Partitions and Split-Brain Prevention

**Split-Brain Problem**:
- Two partitions both think they're the quorum
- Both make conflicting decisions
- System becomes inconsistent

**Zookeeper's Solution**:
- Only majority can form quorum
- Minority cannot operate
- Prevents split-brain

**Example**:
```
5-node cluster, 3+2 partition:
- 3-node partition: Has quorum (3 > 5/2), continues
- 2-node partition: No quorum (2 < 5/2), stops
- No split-brain possible
```

### Server Failures and Recovery

**Single Server Failure**:
- If quorum maintained, cluster continues
- Failed server's state becomes stale
- On recovery, synchronizes with cluster
- No data loss

**Multiple Server Failures**:
- If quorum maintained, cluster continues
- If quorum lost, cluster stops
- On recovery, synchronize with majority
- No data loss if quorum was maintained

**Leader Failure**:
- New leader elected automatically
- Election time: ~200ms
- Cluster continues with new leader
- No data loss (committed writes durable)

### Client Failures and Session Expiration

**Client Crash**:
- Session expires (no heartbeat)
- Ephemeral nodes deleted
- Watches cleared
- Other clients notified

**Network Disconnection**:
- Temporary disconnection: session maintained if reconnected in time
- Long disconnection: session expires
- Must recreate session and restore state

**Handling**:
```java
zk.register(new Watcher() {
    public void process(WatchedEvent event) {
        if (event.getState() == KeeperState.Expired) {
            // Session expired, must reconnect
            reconnect();
            restoreState();
        }
    }
});
```

### Data Loss Scenarios (When It Can Happen)

**Scenario 1: Uncommitted Writes**:
- Write proposed but not committed
- Leader fails before commit
- Write is lost
- **Prevention**: Wait for commit confirmation

**Scenario 2: Minority Partition**:
- Write committed in minority partition
- Partition heals, but majority has different state
- Minority's writes may be lost
- **Prevention**: Only majority can commit

**Scenario 3: Disk Failure**:
- Server's disk fails
- Transaction log lost
- Can recover from other servers
- **Prevention**: Replication (data on multiple servers)

**Key Point**: Committed writes on majority are never lost.

### Byzantine Failures (Not Handled by Zookeeper)

**Byzantine Failures**:
- Servers behave maliciously
- Send incorrect messages
- Try to corrupt data

**Zookeeper's Assumption**:
- Assumes fail-stop failures (servers crash, don't misbehave)
- Does not handle Byzantine failures
- Requires trusted environment

**If Byzantine Needed**:
- Use Byzantine Fault Tolerant (BFT) protocols
- More complex and slower
- Zookeeper is not BFT

## Consistency in Practice

### How to Design Applications Around Zookeeper's Guarantees

**Design Principle 1: Leverage Ordering**:
- Use sequential znodes for ordering
- Rely on zxid for global ordering
- Design algorithms that use ordering

**Design Principle 2: Handle Staleness**:
- Accept staleness for non-critical reads
- Use consistent reads when needed
- Check versions for critical operations

**Design Principle 3: Expect Failures**:
- Design for network partitions
- Handle session expiration
- Implement retry logic

**Design Principle 4: Use Appropriate Patterns**:
- Ephemeral nodes for failure detection
- Sequential nodes for ordering
- Watches for reactive systems

### Common Mistakes and How to Avoid Them

**Mistake 1: Assuming Perfect Availability**:
```java
❌ Bad: Assume always available
zk.setData(path, data, -1); // May fail during partition

✅ Good: Handle unavailability
try {
    zk.setData(path, data, -1);
} catch (KeeperException e) {
    // Handle: retry, queue, or fail gracefully
}
```

**Mistake 2: Ignoring Session Expiration**:
```java
❌ Bad: No session handling
// Ephemeral nodes deleted, but app doesn't know

✅ Good: Handle session expiration
zk.register(watcher); // Reconnect and restore on expiration
```

**Mistake 3: Not Handling Stale Reads**:
```java
❌ Bad: Assume reads are always fresh
byte[] data = zk.getData(path, false, null);
useData(data); // May be stale

✅ Good: Use consistent reads when needed
byte[] data = zk.getData(path, true, null); // Sync with leader
```

**Mistake 4: Not Using Version Checking**:
```java
❌ Bad: Blind updates
zk.setData(path, newData, -1); // May overwrite concurrent update

✅ Good: Optimistic concurrency control
Stat stat = zk.exists(path, false);
zk.setData(path, newData, stat.getVersion()); // Fails if changed
```

**Mistake 5: Blocking in Watch Handlers**:
```java
❌ Bad: Block in watch handler
zk.getData(path, new Watcher() {
    public void process(WatchedEvent event) {
        doLongRunningWork(); // Blocks event thread
    }
}, null);

✅ Good: Delegate to worker thread
zk.getData(path, new Watcher() {
    public void process(WatchedEvent event) {
        executor.submit(() -> doLongRunningWork());
    }
}, null);
```

### Testing Consistency Guarantees

**Test 1: Ordering**:
```java
@Test
public void testWriteOrdering() {
    // Write multiple values
    zk.setData("/test", "v1".getBytes(), -1);
    zk.setData("/test", "v2".getBytes(), -1);
    zk.setData("/test", "v3".getBytes(), -1);
    
    // Read should see v3 (latest)
    byte[] data = zk.getData("/test", false, null);
    assertEquals("v3", new String(data));
}
```

**Test 2: Atomicity**:
```java
@Test
public void testAtomicity() {
    List<Op> ops = Arrays.asList(
        Op.create("/test1", "data1".getBytes(), ...),
        Op.create("/test2", "data2".getBytes(), ...)
    );
    
    try {
        zk.multi(ops);
    } catch (KeeperException e) {
        // Both should fail or both succeed
        assertFalse(zk.exists("/test1", false) != null ^ 
                   zk.exists("/test2", false) != null);
    }
}
```

**Test 3: Consistency Across Servers**:
```java
@Test
public void testConsistency() {
    // Write to server 1
    ZooKeeper zk1 = connectToServer1();
    zk1.setData("/test", "value".getBytes(), -1);
    
    // Read from server 2
    ZooKeeper zk2 = connectToServer2();
    byte[] data = zk2.getData("/test", false, null);
    assertEquals("value", new String(data));
}
```

### Monitoring Consistency in Production

**Metrics to Monitor**:

1. **Latency**:
   - Read latency (should be `<1ms`)
   - Write latency (should be 5-10ms)
   - Consistent read latency (should be 5-10ms)

2. **Throughput**:
   - Reads per second
   - Writes per second
   - Watch events per second

3. **Consistency Indicators**:
   - Zxid progression (should be monotonic)
   - Server state synchronization
   - Transaction log lag

4. **Failure Indicators**:
   - Leader election frequency
   - Session expiration rate
   - Network partition events

**Monitoring Tools**:
- JMX metrics
- Four-letter commands
- External monitoring (Prometheus, Grafana)
- Custom health checks

**Alerting**:
- High latency alerts
- Frequent leader elections
- Session expiration spikes
- Network partition detection

## Summary

Zookeeper provides strong consistency guarantees:

- **CP System**: Consistency + Partition Tolerance
- **Sequential Consistency**: Global ordering of operations
- **Atomicity**: All-or-nothing operations
- **Single-System Image**: Consistent view across servers
- **Reliability**: Durable, replicated data
- **Timeliness**: Bounded staleness

**Key Trade-offs**:
- Consistency vs Availability (chooses consistency)
- Consistency vs Performance (optimizes for coordination)
- Durability vs Performance (configurable)

**Design Implications**:
- Leverage ordering for coordination
- Handle staleness appropriately
- Design for failures
- Use appropriate patterns

**Limitations**:
- Not perfect availability (CP system)
- Write throughput limited
- Data size limited
- Multi-DC challenges

Understanding these guarantees is essential for:
- Designing correct distributed systems
- Making informed architectural decisions
- Handling edge cases and failures
- Optimizing for your use case

This knowledge is critical for staff-level system design interviews where deep understanding of consistency models and trade-offs is expected.
