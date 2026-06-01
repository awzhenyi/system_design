# Zookeeper Architecture

## Overview

Zookeeper's architecture is designed to provide high availability, strong consistency, and excellent read performance. Understanding the internal architecture is crucial for system design interviews, as it explains the trade-offs and design decisions.

## Cluster Architecture (Ensemble)

### Optimal Cluster Sizes

Zookeeper clusters (ensembles) should have an **odd number of nodes**:

- **3 nodes**: Can tolerate 1 failure (quorum = 2)
- **5 nodes**: Can tolerate 2 failures (quorum = 3)
- **7 nodes**: Can tolerate 3 failures (quorum = 4)

**Why Odd Numbers?**

- Prevents split-brain scenarios
- Ensures a clear majority can always be formed
- Even numbers (e.g., 4 nodes) can split into two equal groups of 2, causing deadlock

**Mathematical Analysis**:
- With n nodes, quorum = (n/2) + 1
- Maximum failures tolerated = (n-1)/2
- 4 nodes: quorum = 3, can tolerate 1 failure (same as 3 nodes, but more expensive)

### Network Partition Scenarios

**3-Node Cluster Example**:
- Normal: All 3 nodes connected, quorum = 2
- Partition 1: 2 nodes together, 1 isolated
  - 2-node partition: Has quorum, continues serving
  - 1-node partition: No quorum, stops serving
- Partition 2: 1-1-1 split
  - No partition has quorum, all stop serving

**5-Node Cluster Example**:
- Normal: All 5 nodes connected, quorum = 3
- Partition 1: 3 nodes together, 2 isolated
  - 3-node partition: Has quorum, continues serving
  - 2-node partition: No quorum, stops serving
- Partition 2: 2-2-1 split
  - No partition has quorum, all stop serving

### Client Connection Handling

- Clients connect to any Zookeeper server
- Connection is maintained through heartbeats
- On connection loss, client automatically reconnects to another server
- Load is distributed across all servers (clients connect randomly or via load balancer)

## Server Roles

### Leader

The leader is responsible for:

1. **Processing Write Requests**: All writes must go through the leader
2. **Proposing Transactions**: Leader proposes transactions to followers
3. **Ordering Transactions**: Ensures global ordering of all writes
4. **Coordinating Consensus**: Manages the consensus process

**Leader Election Process**:

1. Each server has a unique ID (myid file)
2. Servers vote for the server with the highest ID that is up-to-date
3. Server becomes leader when it receives votes from majority
4. Election completes in ~200ms typically

**Detailed Algorithm**:
```
1. Server enters LOOKING state
2. Broadcasts vote (myid, zxid) to all servers
3. Receives votes from other servers
4. Updates vote if received server has:
   - Higher zxid (more up-to-date), OR
   - Same zxid but higher myid
5. If receives votes from majority for itself → becomes LEADING
6. If receives votes from majority for another → becomes FOLLOWING
```

### Followers

Followers are responsible for:

1. **Processing Read Requests**: Can serve reads locally (no consensus needed)
2. **Participating in Consensus**: Vote on leader's proposals
3. **Replicating Data**: Receive and apply transactions from leader
4. **Electing New Leader**: Participate in leader election if leader fails

**Read Optimization**:
- Followers can serve reads directly from local state
- No need to contact leader or achieve consensus
- This is why Zookeeper has excellent read performance

### Observers

Observers are a special type of follower that:

1. **Do NOT Vote**: Cannot participate in consensus voting
2. **Serve Reads**: Can serve read requests like followers
3. **Replicate Data**: Receive transactions from leader
4. **Scale Reads**: Allow read scaling without affecting write quorum

**Why Observers?**

- Adding followers increases write quorum size
- More followers = more servers needed for consensus = slower writes
- Observers allow read scaling without write performance degradation
- Ideal for read-heavy workloads or cross-datacenter replication

**Example**:
- 5-node cluster: 3 followers + 2 observers
- Write quorum still = 3 (only followers vote)
- Read capacity = 5 (all serve reads)

## ZAB Protocol (Zookeeper Atomic Broadcast)

ZAB is Zookeeper's consensus protocol, designed specifically for coordination workloads.

### Protocol Phases

ZAB operates in three main phases:

#### 1. Discovery Phase

When a new leader is elected or a follower connects:

1. **Leader Discovery**: Follower discovers current leader
2. **Epoch Detection**: Determines current epoch (leader's term)
3. **State Comparison**: Compares its state with leader's state

**Purpose**: Ensure all servers agree on the current leader and epoch.

#### 2. Synchronization Phase

Leader synchronizes followers:

1. **Transaction History**: Leader sends missing transactions to followers
2. **State Alignment**: Followers apply transactions to match leader
3. **Quorum Confirmation**: Leader waits for majority to synchronize

**Purpose**: Ensure all servers have consistent state before accepting new writes.

#### 3. Broadcast Phase

Normal operation - processing write requests:

1. **Proposal**: Leader proposes transaction to followers
2. **Acknowledgment**: Followers acknowledge proposal
3. **Commit**: Leader commits when majority acknowledges
4. **Application**: All servers apply committed transaction

**Purpose**: Ensure all writes are consistently applied across the cluster.

### Transaction Ordering and Epoch Management

**Zxid (Zookeeper Transaction ID)**:
- 64-bit number: (epoch, counter)
- Epoch: Increments on each leader election
- Counter: Increments for each transaction in an epoch
- Provides global ordering of all transactions

**Example**:
- Epoch 1, transactions 1-1000
- Leader fails, new leader elected
- Epoch 2, transactions 1-500
- All transactions are globally ordered by zxid

### Why ZAB Over Paxos?

**Paxos Characteristics**:
- General-purpose consensus algorithm
- Can handle arbitrary state machine replication
- More complex to implement correctly

**ZAB Advantages for Zookeeper**:
- **Optimized for Coordination**: Designed specifically for coordination workloads
- **Transaction Ordering**: Built-in total ordering of transactions
- **Simpler Implementation**: Less complex than full Paxos
- **Better Performance**: Optimized for Zookeeper's use case
- **Primary-Backup Model**: Natural leader-follower model

**Key Difference**:
- Paxos: Can handle arbitrary state machines
- ZAB: Optimized for ordered transaction log (Zookeeper's data model)

### Comparison with Raft

**Raft Characteristics**:
- Simpler than Paxos, easier to understand
- Strong leader model
- Log replication with strong consistency

**ZAB vs Raft**:
- **Similarities**: Both use leader-based consensus, both provide strong consistency
- **Differences**:
  - ZAB: Optimized for transaction ordering, designed for coordination
  - Raft: More general-purpose, better for key-value storage
  - ZAB: Transaction-based, Raft: Log-entry based

## Consensus Mechanism

### Quorum-Based Voting

**Quorum Calculation**:
- Quorum = (n/2) + 1, where n = number of voting servers
- 3 servers: quorum = 2
- 5 servers: quorum = 3
- 7 servers: quorum = 4

**Voting Process**:
1. Leader proposes transaction with zxid
2. Followers vote (acknowledge) the proposal
3. Leader commits when majority (quorum) acknowledges
4. All servers apply committed transaction

### Transaction Proposal and Commit Process

**Detailed Flow**:

```
Client → Leader: Write request
Leader:
  1. Assigns zxid to transaction
  2. Proposes transaction to all followers
  3. Waits for acknowledgments from quorum
  4. Commits transaction (sends COMMIT message)
  5. Applies transaction locally
  6. Responds to client

Followers:
  1. Receive proposal from leader
  2. Acknowledge proposal
  3. Receive COMMIT message
  4. Apply transaction locally
```

### Handling Concurrent Proposals

**ZAB Guarantees**:
- Only leader can propose transactions
- All proposals are ordered by zxid
- Followers process proposals in order
- No concurrent proposals from same leader (single-threaded proposal processing)

**During Leader Election**:
- Old leader's proposals are discarded if new leader elected
- New leader's epoch ensures no confusion
- Transactions from old epoch are not applied

## Request Processing Flow

### Write Request Path

**Steps**:

1. **Client Sends Write**: Client sends write request to any server
2. **Forward to Leader**: If received by follower, forwarded to leader
3. **Leader Proposes**: Leader assigns zxid and proposes to followers
4. **Quorum Acknowledgment**: Leader waits for majority to acknowledge
5. **Commit**: Leader commits transaction
6. **Apply**: All servers apply transaction
7. **Response**: Leader responds to client

**Latency Components**:
- Network latency to leader
- Consensus round-trip (proposal → ack → commit)
- Disk write (transaction log)
- Network latency back to client

**Typical Write Latency**: 5-10ms in same datacenter

### Read Request Path

**Local Reads (Default)**:
- Follower/Observer serves read directly from local state
- No consensus needed
- No network round-trip to leader
- **Very Fast**: `<1ms` typically

**Consistent Reads (Optional)**:
- Client can request consistent read
- Server syncs with leader before serving read
- Ensures read sees all committed writes
- **Slower**: Similar to write latency

**When to Use Consistent Reads**:
- Need to see latest committed writes
- Critical for coordination decisions
- Default local reads are usually sufficient (sequential consistency)

### Request Ordering and Pipelining

**Ordering Guarantees**:
- All writes are totally ordered (by zxid)
- Reads see writes in order
- Client's requests are processed in order per session

**Pipelining**:
- Leader can pipeline multiple proposals
- Followers process proposals in order
- Improves throughput for multiple concurrent writes

### Transaction Batching

**Batching for Performance**:
- Multiple client requests can be batched into single transaction
- Reduces consensus overhead
- Improves throughput

**Trade-off**:
- Slightly higher latency (waiting for batch)
- Much better throughput

## Replication Model

### In-Memory Database with Transaction Log

**Data Storage**:
- **In-Memory**: All data stored in memory for fast access
- **Transaction Log**: All writes appended to persistent log
- **Snapshots**: Periodic snapshots of in-memory state

**Why This Design**:
- Fast reads (memory access)
- Durability (transaction log)
- Fast recovery (snapshots + log replay)

### Snapshot Mechanism

**Snapshot Creation**:
- Taken periodically (configurable, default: every 100K transactions)
- Asynchronous process (doesn't block writes)
- Contains full in-memory state at snapshot time

**Snapshot + Log Recovery**:
1. Load latest snapshot into memory
2. Replay transactions from log after snapshot
3. Restore full state

**Log Compaction**:
- Old snapshots can be deleted
- Keep snapshot + transactions after snapshot
- Prevents unbounded log growth

### Replication Lag and Consistency

**Replication Model**:
- Synchronous replication (quorum must acknowledge)
- No replication lag for committed transactions
- All committed transactions are on majority of servers

**Consistency Guarantees**:
- Sequential consistency: All servers see same order
- All committed writes are durable on majority
- No stale reads for committed data

## Quorum Requirements

### Mathematical Analysis of Fault Tolerance

**Fault Tolerance Formula**:
- With n servers, can tolerate f failures
- f = (n-1)/2 (for odd n)
- Quorum = n - f = (n+1)/2

**Examples**:
- 3 servers: tolerate 1 failure, quorum = 2
- 5 servers: tolerate 2 failures, quorum = 3
- 7 servers: tolerate 3 failures, quorum = 4

**Why Majority?**
- Ensures at least one server in quorum has all committed transactions
- Prevents split-brain (two groups claiming to be quorum)

### Split-Brain Prevention

**Problem**: Network partition creates two groups, both think they're quorum

**Solution**: Only majority can form quorum
- 5 servers, partition into 3+2
- 3-server group: Has quorum, continues
- 2-server group: No quorum, stops serving
- Prevents conflicting decisions

### Minimum Quorum Calculations

**For Different Cluster Sizes**:

| Cluster Size | Quorum | Tolerate Failures | Availability |
|--------------|--------|-------------------|--------------|
| 1            | 1      | 0                 | No HA        |
| 3            | 2      | 1                 | 33% failure  |
| 5            | 3      | 2                 | 40% failure  |
| 7            | 4      | 3                 | 43% failure  |

**Diminishing Returns**:
- Larger clusters provide more fault tolerance
- But also more overhead (more servers to coordinate)
- 3-5 nodes is typical for most deployments

## Failure Handling and Recovery

### Leader Failure Scenarios

**Detection**:
- Followers detect leader failure via timeout
- Enter leader election immediately

**Recovery Time**:
- Leader election: ~200ms typically
- State synchronization: Depends on lag (usually `<1s`)
- **Total Recovery Time**: ~1-2 seconds typically

**During Recovery**:
- Cluster cannot accept writes
- Reads can still be served (from followers/observers)
- Client requests may timeout, need retry logic

### Follower Failure and Reconnection

**Follower Failure**:
- Leader continues with remaining quorum
- Failed follower's state becomes stale
- No impact on cluster operation (if quorum maintained)

**Reconnection**:
- Failed follower reconnects
- Discovers current leader
- Synchronizes state (receives missing transactions)
- Rejoins cluster

**State Synchronization**:
- Leader sends missing transactions
- Follower applies transactions to catch up
- Once synchronized, follower can serve requests

### Network Partition Handling

**Partition Scenarios**:

1. **Minority Partition**:
   - Loses quorum
   - Stops serving requests
   - Clients connected to this partition cannot perform writes

2. **Majority Partition**:
   - Maintains quorum
   - Continues serving requests
   - Elects new leader if needed

3. **Healing**:
   - When partition heals, minority synchronizes with majority
   - No data loss (majority has all committed transactions)

### Data Recovery from Snapshots and Logs

**Recovery Process**:

1. **Load Snapshot**: Load latest snapshot into memory
2. **Replay Log**: Apply transactions from log after snapshot
3. **Synchronize**: If needed, synchronize with other servers
4. **Ready**: Server ready to serve requests

**Data Integrity**:
- Snapshots are taken atomically
- Transaction log is append-only
- Recovery ensures consistency

## Performance Characteristics

### Read vs Write Performance

**Read Performance**:
- **Very Fast**: `<1ms` typically (local memory access)
- **Scalable**: Can add observers to scale reads
- **No Consensus**: No network round-trip needed
- **Throughput**: 100K+ reads/sec per server

**Write Performance**:
- **Slower**: 5-10ms typically (consensus required)
- **Limited**: ~10K-20K writes/sec per cluster (not per server)
- **Consensus Overhead**: Requires quorum acknowledgment
- **Bottleneck**: Consensus is the limiting factor

**Why Reads are Fast**:
- Served from local memory
- No consensus needed
- No network round-trip to leader (for local reads)

**Why Writes are Slower**:
- Must go through leader
- Requires consensus (quorum acknowledgment)
- Must write to transaction log
- Network round-trips add latency

### Throughput Limits and Bottlenecks

**Write Throughput Bottlenecks**:

1. **Consensus Overhead**: Each write requires quorum acknowledgment
2. **Network Latency**: Round-trips between leader and followers
3. **Disk I/O**: Transaction log writes (can be optimized with batching)
4. **Leader CPU**: Single leader processes all writes

**Typical Throughput**:
- Small cluster (3 nodes): ~5K-10K writes/sec
- Medium cluster (5 nodes): ~10K-15K writes/sec
- Large cluster (7 nodes): ~15K-20K writes/sec

**Read Throughput**:
- Per server: 100K+ reads/sec
- Can scale horizontally with observers
- Limited by network and CPU, not consensus

### Latency Characteristics

**Read Latency**:
- Local read: `<1ms` (memory access)
- Consistent read: 5-10ms (sync with leader)

**Write Latency**:
- Typical: 5-10ms (consensus + disk write)
- P99: 20-50ms (network hiccups, disk spikes)
- P999: 100-200ms (rare, during leader election)

**Factors Affecting Latency**:
- Network latency between nodes
- Disk I/O performance (transaction log)
- Cluster size (more nodes = more network hops)
- Load (high load = queuing delays)

### Scalability Limits

**Why Zookeeper Doesn't Scale Horizontally for Writes**:

1. **Single Leader**: All writes go through one leader
2. **Consensus Overhead**: More nodes = more network round-trips
3. **Quorum Requirements**: Larger quorum = slower consensus
4. **Not Sharded**: All data on all servers (no horizontal partitioning)

**Scaling Strategies**:

1. **Scale Reads**: Add observers (unlimited)
2. **Optimize Writes**: Batch operations, reduce write frequency
3. **Separate Concerns**: Use Zookeeper for coordination, other systems for data
4. **Accept Limits**: Zookeeper is for coordination, not high-throughput data storage

**Practical Limits**:
- Cluster size: 3-7 nodes typical (larger is possible but diminishing returns)
- Write throughput: ~20K writes/sec maximum
- Read throughput: Scales with observers
- Data size: Keep znodes small (`<1MB`), total data `<100GB` typically

### Performance Tuning and Optimization Strategies

**Write Optimization**:

1. **Batching**: Batch multiple operations into single transaction
2. **Reduce Write Frequency**: Cache configuration, write only on changes
3. **Optimize Transaction Log**: Use fast SSDs, separate log disk
4. **Tune Snapshot Frequency**: Balance recovery time vs disk usage

**Read Optimization**:

1. **Add Observers**: Scale reads without affecting writes
2. **Local Reads**: Use default local reads (not consistent reads)
3. **Connection Pooling**: Reuse connections, reduce overhead
4. **Client-Side Caching**: Cache frequently accessed data

**Network Optimization**:

1. **Low Latency Network**: Deploy in same datacenter, low latency links
2. **Dedicated Network**: Separate Zookeeper network from application traffic
3. **Tune Timeouts**: Balance responsiveness vs false failures

## Operational Concerns

### Monitoring Metrics

**Key Metrics to Monitor**:

1. **Latency**:
   - Average read latency
   - Average write latency
   - P99, P999 latencies

2. **Throughput**:
   - Reads per second
   - Writes per second
   - Connections per second

3. **Cluster Health**:
   - Leader/follower status
   - Connection count
   - Outstanding requests

4. **Resource Usage**:
   - Memory usage
   - Disk I/O (transaction log)
   - Network I/O

**Monitoring Tools**:
- JMX metrics (built-in)
- Four-letter commands (ruok, stat, mntr)
- External monitoring (Prometheus, Grafana)

### Common Failure Modes and Debugging

**Common Issues**:

1. **Leader Election Loops**:
   - Symptom: Frequent leader elections
   - Cause: Network issues, timeouts too short
   - Fix: Increase timeouts, check network

2. **Out of Memory**:
   - Symptom: Server crashes, OOM errors
   - Cause: Too much data, memory leak
   - Fix: Increase heap, reduce data size, check for leaks

3. **Disk Full**:
   - Symptom: Writes fail, server stops
   - Cause: Transaction log or snapshots filling disk
   - Fix: Clean old snapshots, increase disk space

4. **Network Partitions**:
   - Symptom: Cluster splits, some clients can't write
   - Cause: Network issues between datacenters
   - Fix: Fix network, ensure low latency between nodes

**Debugging Commands**:
- `ruok`: Check if server is running
- `stat`: Server statistics
- `mntr`: Detailed metrics
- `conf`: Configuration
- `cons`: Client connections

### Capacity Planning Guidelines

**Data Size Planning**:
- Keep znodes small (`<1MB` each)
- Total data: `<100GB` typically
- Monitor memory usage (all data in memory)

**Throughput Planning**:
- Estimate write rate (writes/sec)
- Ensure cluster can handle peak load
- Plan for 2-3x headroom

**Cluster Sizing**:
- Start with 3 nodes (can tolerate 1 failure)
- Scale to 5 nodes for more fault tolerance
- Use observers for read scaling

**Network Planning**:
- Low latency between nodes (`<1ms` ideally)
- Dedicated network if possible
- Plan for network partitions

**Storage Planning**:
- Transaction log: Fast SSD, separate disk
- Snapshots: Can be on slower disk
- Plan for log growth (keep old snapshots for recovery)

## Summary

Zookeeper's architecture is optimized for coordination workloads:

- **Strong Consistency**: CP system with quorum-based consensus
- **Fast Reads**: Local reads from memory, no consensus
- **Ordered Writes**: Global ordering via ZAB protocol
- **High Availability**: Automatic failover, fault tolerance
- **Scalable Reads**: Observers allow read scaling

Understanding the architecture helps explain:
- Why reads are fast (local, no consensus)
- Why writes are slower (consensus required)
- Why it doesn't scale horizontally for writes (single leader, consensus)
- How to design systems that use Zookeeper effectively

This knowledge is crucial for staff-level system design interviews where deep understanding of trade-offs and architectural decisions is expected.
