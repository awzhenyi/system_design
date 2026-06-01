# Zookeeper Use Cases

## Overview

Zookeeper excels at providing coordination primitives for distributed systems. This document covers the most common use cases with detailed implementation patterns, code examples, and real-world applications.

## Leader Election

### Detailed Implementation Algorithm

Leader election is one of the most common Zookeeper use cases. The algorithm uses ephemeral-sequential znodes to ensure only one leader exists at a time.

**Algorithm Steps**:

1. **Create Ephemeral-Sequential Znode**: Each candidate creates a znode with sequential flag
2. **Get All Candidates**: List all candidate znodes
3. **Find Leader**: Leader is the candidate with the lowest sequence number
4. **Watch Predecessor**: If not leader, watch the predecessor znode
5. **Become Leader**: When predecessor is deleted (leader failed), check if you're now the leader

**Why This Works**:
- Sequential numbers provide ordering
- Ephemeral nodes auto-delete on failure
- Watching predecessor ensures quick re-election
- Only one leader at a time (lowest number)

### Code Example

```java
public class LeaderElection {
    private final ZooKeeper zk;
    private final String electionPath;
    private String myZnodePath;
    private String currentLeaderPath;
    private LeaderCallback leaderCallback;
    
    public LeaderElection(ZooKeeper zk, String electionPath, LeaderCallback callback) {
        this.zk = zk;
        this.electionPath = electionPath;
        this.leaderCallback = callback;
    }
    
    public void startElection() throws KeeperException, InterruptedException {
        // Ensure election path exists
        ensurePath(electionPath);
        
        // Create ephemeral-sequential znode
        myZnodePath = zk.create(electionPath + "/candidate-",
                                "my-id".getBytes(),
                                ZooDefs.Ids.OPEN_ACL_UNSAFE,
                                CreateMode.EPHEMERAL_SEQUENTIAL);
        
        // Check if I'm the leader
        checkAndBecomeLeader();
    }
    
    private void checkAndBecomeLeader() throws KeeperException, InterruptedException {
        List<String> children = zk.getChildren(electionPath, false);
        Collections.sort(children);
        
        String mySequence = myZnodePath.substring(myZnodePath.lastIndexOf('/') + 1);
        int myIndex = children.indexOf(mySequence);
        
        if (myIndex == 0) {
            // I'm the leader!
            currentLeaderPath = myZnodePath;
            leaderCallback.onElected();
        } else {
            // Watch my predecessor
            String predecessor = electionPath + "/" + children.get(myIndex - 1);
            watchPredecessor(predecessor);
        }
    }
    
    private void watchPredecessor(String predecessorPath) throws KeeperException, InterruptedException {
        Stat stat = zk.exists(predecessorPath, new Watcher() {
            @Override
            public void process(WatchedEvent event) {
                if (event.getType() == Event.EventType.NodeDeleted) {
                    try {
                        checkAndBecomeLeader();
                    } catch (Exception e) {
                        // Handle error
                    }
                }
            }
        });
        
        if (stat == null) {
            // Predecessor already deleted, check again
            checkAndBecomeLeader();
        }
    }
    
    public boolean isLeader() {
        return currentLeaderPath != null && currentLeaderPath.equals(myZnodePath);
    }
    
    public void stop() throws InterruptedException, KeeperException {
        if (myZnodePath != null) {
            zk.delete(myZnodePath, -1);
        }
    }
    
    private void ensurePath(String path) throws KeeperException, InterruptedException {
        // Implementation from Data Model examples
    }
}

interface LeaderCallback {
    void onElected();
    void onRevoked();
}
```

### Ephemeral-Sequential Pattern Explanation

**Why Ephemeral**:
- Automatically deleted when candidate crashes
- No manual cleanup needed
- Natural failure detection

**Why Sequential**:
- Provides global ordering
- Lowest number = leader (deterministic)
- No race conditions

**Combined Benefits**:
- Automatic cleanup on failure
- Deterministic leader selection
- Fast re-election (watch predecessor)

### Handling Leader Failures and Re-election

**Failure Scenarios**:

1. **Leader Crashes**:
   - Leader's ephemeral znode is deleted
   - Predecessor watcher fires
   - Next candidate becomes leader
   - Re-election time: ~1-2 seconds

2. **Network Partition**:
   - Partition with quorum continues
   - Partition without quorum stops
   - Leader in quorum partition continues
   - New leader elected in quorum partition if needed

3. **Leader Voluntarily Steps Down**:
   - Leader deletes its znode
   - Predecessor watcher fires
   - Next candidate becomes leader

**Re-election Process**:
```
1. Current leader fails (znode deleted)
2. Predecessor watcher fires on next candidate
3. Candidate checks if it's now leader
4. If yes, becomes leader and notifies callback
5. If no, watches new predecessor
```

### Real-World Examples

**Kafka Controller Election**:
- Kafka uses Zookeeper to elect a controller broker
- Controller manages partition leadership
- Only one controller at a time
- Fast failover critical for Kafka availability

**Distributed Service Coordination**:
- Multiple service instances coordinate work
- Leader handles coordination tasks
- Followers ready to take over
- Common in microservices architectures

### Common Pitfalls and Edge Cases

**Pitfall 1: Not Watching Predecessor**:
```java
❌ Bad: Only check once
checkAndBecomeLeader(); // Only checks once

✅ Good: Watch predecessor for changes
watchPredecessor(predecessor);
```

**Pitfall 2: Not Handling Session Expiration**:
```java
❌ Bad: No session handling
// If session expires, znode deleted but no reconnection

✅ Good: Handle session expiration
zk.register(new Watcher() {
    public void process(WatchedEvent event) {
        if (event.getState() == KeeperState.Expired) {
            reconnectAndRejoin();
        }
    }
});
```

**Pitfall 3: Race Condition in Leader Check**:
```java
❌ Bad: Check and act separately
if (isLeader()) {
    doLeaderWork(); // Might not be leader anymore
}

✅ Good: Atomic check and act
synchronized (this) {
    if (isLeader()) {
        doLeaderWork();
    }
}
```

### Performance Considerations

- **Election Time**: ~1-2 seconds typically
- **Re-election Time**: ~1-2 seconds after leader failure
- **Overhead**: Minimal (one znode per candidate)
- **Scalability**: Works with hundreds of candidates

## Distributed Locks

### Exclusive Locks

Exclusive locks ensure only one process can hold the lock at a time.

**Implementation Pattern**:
1. Create ephemeral-sequential znode under lock path
2. Get all lock znodes, sorted
3. If my znode is first, I have the lock
4. Otherwise, watch predecessor
5. When predecessor deleted, try again

**Code Example**:
```java
public class DistributedLock {
    private final ZooKeeper zk;
    private final String lockPath;
    private String myLockPath;
    private final long timeout;
    
    public DistributedLock(ZooKeeper zk, String lockPath, long timeoutMs) {
        this.zk = zk;
        this.lockPath = lockPath;
        this.timeout = timeoutMs;
    }
    
    public boolean tryLock() throws KeeperException, InterruptedException {
        ensurePath(lockPath);
        
        // Create ephemeral-sequential znode
        myLockPath = zk.create(lockPath + "/lock-",
                               Thread.currentThread().getName().getBytes(),
                               ZooDefs.Ids.OPEN_ACL_UNSAFE,
                               CreateMode.EPHEMERAL_SEQUENTIAL);
        
        return attemptLock();
    }
    
    private boolean attemptLock() throws KeeperException, InterruptedException {
        List<String> children = zk.getChildren(lockPath, false);
        Collections.sort(children);
        
        String mySequence = myLockPath.substring(myLockPath.lastIndexOf('/') + 1);
        int myIndex = children.indexOf(mySequence);
        
        if (myIndex == 0) {
            // I have the lock!
            return true;
        } else {
            // Watch predecessor
            String predecessor = lockPath + "/" + children.get(myIndex - 1);
            return waitForLock(predecessor);
        }
    }
    
    private boolean waitForLock(String predecessorPath) throws KeeperException, InterruptedException {
        CountDownLatch latch = new CountDownLatch(1);
        AtomicBoolean acquired = new AtomicBoolean(false);
        
        Stat stat = zk.exists(predecessorPath, new Watcher() {
            @Override
            public void process(WatchedEvent event) {
                if (event.getType() == Event.EventType.NodeDeleted) {
                    try {
                        if (attemptLock()) {
                            acquired.set(true);
                            latch.countDown();
                        }
                    } catch (Exception e) {
                        latch.countDown();
                    }
                }
            }
        });
        
        if (stat == null) {
            // Predecessor already gone, try again
            return attemptLock();
        }
        
        // Wait for timeout or lock acquisition
        boolean timedOut = !latch.await(timeout, TimeUnit.MILLISECONDS);
        return !timedOut && acquired.get();
    }
    
    public void unlock() throws InterruptedException, KeeperException {
        if (myLockPath != null) {
            zk.delete(myLockPath, -1);
            myLockPath = null;
        }
    }
}
```

### Shared/Read-Write Locks

Read-write locks allow multiple readers or one writer.

**Implementation**:
- Read locks: Create znode with "read-" prefix
- Write locks: Create znode with "write-" prefix
- Readers can proceed if no writer ahead
- Writer waits for all readers/writers ahead

**Code Example**:
```java
public class ReadWriteLock {
    private final ZooKeeper zk;
    private final String lockPath;
    private String myLockPath;
    
    public ReadWriteLock(ZooKeeper zk, String lockPath) {
        this.zk = zk;
        this.lockPath = lockPath;
    }
    
    public boolean tryReadLock() throws KeeperException, InterruptedException {
        ensurePath(lockPath);
        
        myLockPath = zk.create(lockPath + "/read-",
                              Thread.currentThread().getName().getBytes(),
                              ZooDefs.Ids.OPEN_ACL_UNSAFE,
                              CreateMode.EPHEMERAL_SEQUENTIAL);
        
        return checkReadLock();
    }
    
    private boolean checkReadLock() throws KeeperException, InterruptedException {
        List<String> children = zk.getChildren(lockPath, false);
        Collections.sort(children);
        
        String mySequence = myLockPath.substring(myLockPath.lastIndexOf('/') + 1);
        int myIndex = children.indexOf(mySequence);
        
        // Check if any write lock is ahead of me
        for (int i = 0; i < myIndex; i++) {
            if (children.get(i).startsWith("write-")) {
                // Wait for write lock to be released
                return waitForLock(lockPath + "/" + children.get(i));
            }
        }
        
        // No write lock ahead, I have read lock
        return true;
    }
    
    public boolean tryWriteLock() throws KeeperException, InterruptedException {
        ensurePath(lockPath);
        
        myLockPath = zk.create(lockPath + "/write-",
                              Thread.currentThread().getName().getBytes(),
                              ZooDefs.Ids.OPEN_ACL_UNSAFE,
                              CreateMode.EPHEMERAL_SEQUENTIAL);
        
        return checkWriteLock();
    }
    
    private boolean checkWriteLock() throws KeeperException, InterruptedException {
        List<String> children = zk.getChildren(lockPath, false);
        Collections.sort(children);
        
        String mySequence = myLockPath.substring(myLockPath.lastIndexOf('/') + 1);
        int myIndex = children.indexOf(mySequence);
        
        if (myIndex == 0) {
            // I'm first, I have write lock
            return true;
        } else {
            // Wait for all locks ahead to be released
            String predecessor = lockPath + "/" + children.get(myIndex - 1);
            return waitForLock(predecessor);
        }
    }
    
    private boolean waitForLock(String lockPath) {
        // Similar to exclusive lock implementation
        return false; // Simplified
    }
    
    public void unlock() throws InterruptedException, KeeperException {
        if (myLockPath != null) {
            zk.delete(myLockPath, -1);
            myLockPath = null;
        }
    }
}
```

### Lock Timeouts and Deadlock Prevention

**Timeout Handling**:
- Set maximum lock hold time
- Automatically release if exceeded
- Use session timeout as backup

**Deadlock Prevention**:
- Zookeeper prevents deadlocks through:
  - Ephemeral nodes (auto-cleanup on crash)
  - Global ordering (no circular waits)
  - Timeout mechanisms

**Best Practices**:
```java
public class TimeoutLock {
    private final long maxHoldTime;
    private long lockAcquiredTime;
    private ScheduledExecutorService scheduler;
    
    public boolean tryLockWithTimeout(long timeoutMs, long maxHoldMs) {
        maxHoldTime = maxHoldMs;
        if (tryLock()) {
            lockAcquiredTime = System.currentTimeMillis();
            scheduleTimeout();
            return true;
        }
        return false;
    }
    
    private void scheduleTimeout() {
        scheduler.schedule(() -> {
            if (System.currentTimeMillis() - lockAcquiredTime > maxHoldTime) {
                try {
                    unlock();
                } catch (Exception e) {
                    // Handle
                }
            }
        }, maxHoldTime, TimeUnit.MILLISECONDS);
    }
}
```

### Lock Granularity and Performance Trade-offs

**Fine-Grained Locks**:
- More locks = better concurrency
- More overhead (more znodes)
- Better for high-contention scenarios

**Coarse-Grained Locks**:
- Fewer locks = less overhead
- Less concurrency
- Better for low-contention scenarios

**Recommendation**: Start with coarse-grained, move to fine-grained if needed.

### Comparison with Redis Distributed Locks

| Aspect | Zookeeper | Redis |
|--------|-----------|-------|
| **Consistency** | Strong (CP) | Eventual (AP with Redis Cluster) |
| **Automatic Cleanup** | Yes (ephemeral) | Yes (TTL) |
| **Performance** | Slower (consensus) | Faster (single node) |
| **Deadlock Prevention** | Built-in | Manual (timeout) |
| **Complexity** | Higher | Lower |
| **Use Case** | Critical coordination | High-throughput scenarios |

**When to Use Zookeeper Locks**:
- Need strong consistency
- Critical coordination (can't tolerate inconsistency)
- Already using Zookeeper

**When to Use Redis Locks**:
- High throughput needed
- Eventual consistency acceptable
- Already using Redis

### Common Anti-patterns

**Anti-pattern 1: Not Using Ephemeral Nodes**:
```java
❌ Bad: Persistent nodes
zk.create(lockPath, data, acl, CreateMode.PERSISTENT);
// Lock never released if process crashes

✅ Good: Ephemeral nodes
zk.create(lockPath, data, acl, CreateMode.EPHEMERAL_SEQUENTIAL);
// Auto-released on crash
```

**Anti-pattern 2: Not Handling Session Expiration**:
```java
❌ Bad: No session handling
// Lock lost on session expiration, but process doesn't know

✅ Good: Handle session expiration
zk.register(watcher); // Reconnect and reacquire if needed
```

**Anti-pattern 3: Holding Lock Too Long**:
```java
❌ Bad: No timeout
lock.tryLock();
doLongRunningWork(); // Might hold lock for hours

✅ Good: Timeout
lock.tryLockWithTimeout(30, TimeUnit.SECONDS);
```

## Configuration Management

### Centralized Configuration Patterns

**Pattern 1: Single Config Znode**:
```
/app/config = {
  "database_url": "localhost:5432",
  "cache_size": 1000
}
```

**Pattern 2: Hierarchical Config**:
```
/app/config/database/url = "localhost:5432"
/app/config/database/pool_size = "10"
/app/config/cache/size = "1000"
```

**Pattern 3: Versioned Config**:
```
/app/config/v1 = {...}
/app/config/v2 = {...}
/app/config/current -> v2
```

### Dynamic Configuration Updates with Watches

**Implementation**:
```java
public class ConfigManager {
    private final ZooKeeper zk;
    private final String configPath;
    private Map<String, String> config = new HashMap<>();
    private ConfigListener listener;
    
    public ConfigManager(ZooKeeper zk, String configPath, ConfigListener listener) {
        this.zk = zk;
        this.configPath = configPath;
        this.listener = listener;
        loadAndWatch();
    }
    
    private void loadAndWatch() throws KeeperException, InterruptedException {
        byte[] data = zk.getData(configPath, new Watcher() {
            @Override
            public void process(WatchedEvent event) {
                if (event.getType() == Event.EventType.NodeDataChanged) {
                    try {
                        loadAndWatch(); // Reload and rewatch
                    } catch (Exception e) {
                        // Handle error
                    }
                }
            }
        }, null);
        
        // Parse and update config
        updateConfig(new String(data));
    }
    
    private void updateConfig(String configData) {
        // Parse JSON/YAML/Properties
        Map<String, String> newConfig = parseConfig(configData);
        Map<String, String> oldConfig = new HashMap<>(config);
        config = newConfig;
        
        // Notify listener of changes
        if (listener != null) {
            listener.onConfigChanged(oldConfig, newConfig);
        }
    }
    
    public String getConfig(String key) {
        return config.get(key);
    }
}

interface ConfigListener {
    void onConfigChanged(Map<String, String> oldConfig, Map<String, String> newConfig);
}
```

### Versioning and Rollback Strategies

**Versioned Configuration**:
```java
public void updateConfig(String newConfig, int version) throws KeeperException {
    Stat stat = zk.exists(configPath, false);
    if (stat.getVersion() != version) {
        throw new KeeperException.BadVersionException();
    }
    
    zk.setData(configPath, newConfig.getBytes(), version);
    
    // Also store version history
    String versionPath = configPath + "/versions/" + System.currentTimeMillis();
    zk.create(versionPath, newConfig.getBytes(), 
              ZooDefs.Ids.OPEN_ACL_UNSAFE, CreateMode.PERSISTENT);
}

public void rollback(long timestamp) throws KeeperException, InterruptedException {
    List<String> versions = zk.getChildren(configPath + "/versions", false);
    // Find version closest to timestamp
    String versionPath = findVersion(versions, timestamp);
    byte[] data = zk.getData(versionPath, false, null);
    zk.setData(configPath, data, -1);
}
```

### Configuration Distribution Patterns

**Pattern 1: Push Model** (Zookeeper watches)
- Config changes pushed via watches
- Clients notified immediately
- Low latency updates

**Pattern 2: Pull Model** (Periodic polling)
- Clients poll for changes
- Simpler but higher latency
- Less efficient

**Recommendation**: Use push model (watches) for real-time updates.

### Handling Configuration Conflicts

**Optimistic Concurrency Control**:
```java
public boolean updateConfig(String newConfig) throws KeeperException {
    Stat stat = zk.exists(configPath, false);
    try {
        zk.setData(configPath, newConfig.getBytes(), stat.getVersion());
        return true;
    } catch (KeeperException.BadVersionException e) {
        // Conflict detected, retry or notify
        return false;
    }
}
```

## Service Discovery

### Service Registration Patterns

**Ephemeral Nodes for Registration**:
```java
public class ServiceRegistry {
    private final ZooKeeper zk;
    private final String servicePath;
    private String myServicePath;
    
    public ServiceRegistry(ZooKeeper zk, String servicePath) {
        this.zk = zk;
        this.servicePath = servicePath;
    }
    
    public void register(String serviceName, String host, int port) 
            throws KeeperException, InterruptedException {
        ensurePath(servicePath);
        
        String serviceData = host + ":" + port;
        myServicePath = zk.create(servicePath + "/" + serviceName + "-",
                                 serviceData.getBytes(),
                                 ZooDefs.Ids.OPEN_ACL_UNSAFE,
                                 CreateMode.EPHEMERAL_SEQUENTIAL);
    }
    
    public void unregister() throws InterruptedException, KeeperException {
        if (myServicePath != null) {
            zk.delete(myServicePath, -1);
        }
    }
}
```

### Service Discovery Mechanisms

**Discovery Implementation**:
```java
public class ServiceDiscovery {
    private final ZooKeeper zk;
    private final String servicePath;
    private List<String> services = new ArrayList<>();
    private ServiceListener listener;
    
    public ServiceDiscovery(ZooKeeper zk, String servicePath, ServiceListener listener) {
        this.zk = zk;
        this.servicePath = servicePath;
        this.listener = listener;
        discoverAndWatch();
    }
    
    private void discoverAndWatch() throws KeeperException, InterruptedException {
        List<String> children = zk.getChildren(servicePath, new Watcher() {
            @Override
            public void process(WatchedEvent event) {
                if (event.getType() == Event.EventType.NodeChildrenChanged) {
                    try {
                        discoverAndWatch(); // Rediscover
                    } catch (Exception e) {
                        // Handle
                    }
                }
            }
        }, null);
        
        List<String> newServices = new ArrayList<>();
        for (String child : children) {
            byte[] data = zk.getData(servicePath + "/" + child, false, null);
            newServices.add(new String(data));
        }
        
        List<String> oldServices = new ArrayList<>(services);
        services = newServices;
        
        if (listener != null) {
            listener.onServicesChanged(oldServices, newServices);
        }
    }
    
    public List<String> getServices() {
        return new ArrayList<>(services);
    }
    
    public String getRandomService() {
        if (services.isEmpty()) {
            return null;
        }
        return services.get(new Random().nextInt(services.size()));
    }
}

interface ServiceListener {
    void onServicesChanged(List<String> oldServices, List<String> newServices);
}
```

### Health Checking Integration

**Health Check Pattern**:
```java
public class HealthCheckedService {
    private final ServiceRegistry registry;
    private final ScheduledExecutorService healthChecker;
    private volatile boolean healthy = true;
    
    public void start() {
        // Register service
        registry.register(serviceName, host, port);
        
        // Start health checks
        healthChecker.scheduleAtFixedRate(() -> {
            boolean isHealthy = performHealthCheck();
            if (isHealthy != healthy) {
                healthy = isHealthy;
                if (healthy) {
                    registry.register(serviceName, host, port);
                } else {
                    registry.unregister();
                }
            }
        }, 0, 5, TimeUnit.SECONDS);
    }
    
    private boolean performHealthCheck() {
        // Check database, dependencies, etc.
        return true; // Simplified
    }
}
```

### Load Balancing with Service Discovery

**Load Balancing Strategies**:

1. **Random**: Pick random service
2. **Round Robin**: Cycle through services
3. **Least Connections**: Track connections per service
4. **Weighted**: Use service metadata for weights

**Implementation**:
```java
public class LoadBalancer {
    private final ServiceDiscovery discovery;
    private int currentIndex = 0;
    
    public String getNextService() {
        List<String> services = discovery.getServices();
        if (services.isEmpty()) {
            return null;
        }
        
        // Round robin
        String service = services.get(currentIndex);
        currentIndex = (currentIndex + 1) % services.size();
        return service;
    }
}
```

### Multi-Datacenter Considerations

**Challenges**:
- Zookeeper typically per-datacenter
- Cross-datacenter latency
- Network partitions

**Solutions**:
1. **Per-DC Zookeeper**: Each datacenter has its own cluster
2. **Observer Mode**: Use observers for cross-DC replication
3. **Application-Level**: Application coordinates across datacenters

## Group Membership

### Tracking Active Nodes

**Membership Tracking**:
```java
public class GroupMembership {
    private final ZooKeeper zk;
    private final String groupPath;
    private String myMemberPath;
    private Set<String> members = new HashSet<>();
    
    public void join(String memberId) throws KeeperException, InterruptedException {
        ensurePath(groupPath);
        
        myMemberPath = zk.create(groupPath + "/member-",
                                memberId.getBytes(),
                                ZooDefs.Ids.OPEN_ACL_UNSAFE,
                                CreateMode.EPHEMERAL_SEQUENTIAL);
        
        updateMembership();
    }
    
    private void updateMembership() throws KeeperException, InterruptedException {
        List<String> children = zk.getChildren(groupPath, new Watcher() {
            @Override
            public void process(WatchedEvent event) {
                if (event.getType() == Event.EventType.NodeChildrenChanged) {
                    try {
                        updateMembership();
                    } catch (Exception e) {
                        // Handle
                    }
                }
            }
        }, null);
        
        Set<String> newMembers = new HashSet<>();
        for (String child : children) {
            byte[] data = zk.getData(groupPath + "/" + child, false, null);
            newMembers.add(new String(data));
        }
        
        members = newMembers;
    }
    
    public Set<String> getMembers() {
        return new HashSet<>(members);
    }
}
```

### Membership Change Notifications

**Notification Pattern**:
```java
public class MembershipManager {
    private MembershipListener listener;
    
    public void setListener(MembershipListener listener) {
        this.listener = listener;
    }
    
    private void notifyMembershipChange(Set<String> oldMembers, Set<String> newMembers) {
        Set<String> added = new HashSet<>(newMembers);
        added.removeAll(oldMembers);
        
        Set<String> removed = new HashSet<>(oldMembers);
        removed.removeAll(newMembers);
        
        if (listener != null) {
            if (!added.isEmpty()) {
                listener.onMemberAdded(added);
            }
            if (!removed.isEmpty()) {
                listener.onMemberRemoved(removed);
            }
        }
    }
}

interface MembershipListener {
    void onMemberAdded(Set<String> members);
    void onMemberRemoved(Set<String> members);
}
```

### Handling Network Partitions

**Partition Handling**:
- Members in minority partition cannot update membership
- Members in majority partition continue
- When partition heals, minority synchronizes with majority
- No data loss (majority has all updates)

## Barriers and Queues

### Distributed Barriers

**Barrier Implementation**:
```java
public class DistributedBarrier {
    private final ZooKeeper zk;
    private final String barrierPath;
    private final int threshold;
    private String myBarrierPath;
    
    public DistributedBarrier(ZooKeeper zk, String barrierPath, int threshold) {
        this.zk = zk;
        this.barrierPath = barrierPath;
        this.threshold = threshold;
    }
    
    public void await() throws KeeperException, InterruptedException {
        ensurePath(barrierPath);
        
        myBarrierPath = zk.create(barrierPath + "/barrier-",
                                 Thread.currentThread().getName().getBytes(),
                                 ZooDefs.Ids.OPEN_ACL_UNSAFE,
                                 CreateMode.EPHEMERAL_SEQUENTIAL);
        
        while (true) {
            List<String> children = zk.getChildren(barrierPath, false);
            if (children.size() >= threshold) {
                return; // Barrier reached
            }
            
            // Wait for more participants
            String predecessor = findPredecessor(children);
            if (predecessor != null) {
                waitForNode(barrierPath + "/" + predecessor);
            }
        }
    }
    
    public void leave() throws InterruptedException, KeeperException {
        if (myBarrierPath != null) {
            zk.delete(myBarrierPath, -1);
        }
    }
}
```

### Distributed Queues

**FIFO Queue**:
```java
public class DistributedQueue {
    private final ZooKeeper zk;
    private final String queuePath;
    
    public void enqueue(byte[] data) throws KeeperException, InterruptedException {
        ensurePath(queuePath);
        zk.create(queuePath + "/item-",
                 data,
                 ZooDefs.Ids.OPEN_ACL_UNSAFE,
                 CreateMode.PERSISTENT_SEQUENTIAL);
    }
    
    public byte[] dequeue() throws KeeperException, InterruptedException {
        while (true) {
            List<String> children = zk.getChildren(queuePath, false);
            if (children.isEmpty()) {
                // Wait for items
                waitForChildren();
                continue;
            }
            
            Collections.sort(children);
            String firstItem = children.get(0);
            
            try {
                byte[] data = zk.getData(queuePath + "/" + firstItem, false, null);
                zk.delete(queuePath + "/" + firstItem, -1);
                return data;
            } catch (KeeperException.NoNodeException e) {
                // Another consumer got it, try next
                continue;
            }
        }
    }
}
```

## Real-World Examples

### Kafka's Use of Zookeeper

**Controller Election**:
- `/controller` - ephemeral znode for controller
- Only one controller at a time
- Controller manages partition leadership

**Broker Registration**:
- `/brokers/ids/{id}` - broker metadata
- Ephemeral nodes for active brokers
- Automatic cleanup on broker failure

**Topic Configuration**:
- `/config/topics/{topic}` - topic configuration
- Watches for configuration changes
- Dynamic topic reconfiguration

### Hadoop's Use of Zookeeper

**NameNode Coordination**:
- High availability for NameNode
- Active/standby NameNode coordination
- Automatic failover

**Resource Management**:
- YARN resource manager coordination
- Application master coordination

### HBase's Use of Zookeeper

**Master Election**:
- `/hbase/master` - master election
- Only one active master
- Automatic failover

**Region Server Coordination**:
- `/hbase/rs/{server}` - region server registration
- Ephemeral nodes for active servers
- Region assignment coordination

## When NOT to Use Zookeeper

### Alternatives for Different Needs

**Message Queues**:
- Use: Kafka, RabbitMQ, Amazon SQS
- Zookeeper: Not designed for message queuing

**Data Storage**:
- Use: Databases, object storage
- Zookeeper: 1MB limit, not a database

**High-Throughput Writes**:
- Use: Redis, etcd (better write performance)
- Zookeeper: Limited to ~20K writes/sec

**Multi-Datacenter Coordination**:
- Use: Consul, etcd with proper setup
- Zookeeper: Better for single datacenter

### Performance Limitations

- Write throughput: ~20K writes/sec maximum
- Not horizontally scalable for writes
- Single leader bottleneck
- Consensus overhead

### Scalability Constraints

- Cluster size: 3-7 nodes typical
- Data size: `<100GB total`
- Znode size: `<1MB each`
- Not sharded (all data on all servers)

### Decision Framework

**Use Zookeeper when**:
- Need coordination primitives
- Strong consistency required
- Read-heavy workload
- Single datacenter

**Don't use Zookeeper when**:
- Need high write throughput
- Need data storage
- Need message queuing
- Multi-datacenter coordination critical

## Advanced Patterns

### Two-Phase Commit Coordination

**2PC Coordinator**:
```java
public class TwoPhaseCommitCoordinator {
    private final ZooKeeper zk;
    private final String commitPath;
    
    public boolean coordinateCommit(List<String> participants, byte[] transaction) {
        // Phase 1: Prepare
        for (String participant : participants) {
            if (!prepare(participant, transaction)) {
                abort(participants);
                return false;
            }
        }
        
        // Phase 2: Commit
        for (String participant : participants) {
            commit(participant);
        }
        return true;
    }
}
```

### Workflow Orchestration

**Workflow State Management**:
- Store workflow state in Zookeeper
- Use watches for state transitions
- Coordinate workflow steps

### Distributed Counters

**Counter Implementation**:
```java
public class DistributedCounter {
    private final ZooKeeper zk;
    private final String counterPath;
    
    public long increment() throws KeeperException, InterruptedException {
        while (true) {
            Stat stat = new Stat();
            byte[] data = zk.getData(counterPath, false, stat);
            long current = Long.parseLong(new String(data));
            long next = current + 1;
            
            try {
                zk.setData(counterPath, String.valueOf(next).getBytes(), stat.getVersion());
                return next;
            } catch (KeeperException.BadVersionException e) {
                // Retry
                continue;
            }
        }
    }
}
```

### Rate Limiting Coordination

**Rate Limiter**:
- Use Zookeeper to coordinate rate limits
- Track requests across instances
- Enforce global rate limits

## Summary

Zookeeper provides powerful coordination primitives:

- **Leader Election**: Ephemeral-sequential pattern
- **Distributed Locks**: Exclusive and read-write locks
- **Configuration Management**: Centralized, dynamic config
- **Service Discovery**: Automatic service registration/discovery
- **Group Membership**: Track active nodes
- **Barriers and Queues**: Synchronization primitives

Key insights for system design interviews:
- Understand when to use Zookeeper vs alternatives
- Know the implementation patterns (ephemeral-sequential is key)
- Understand performance characteristics and limitations
- Design for failure (session expiration, network partitions)
- Use appropriate patterns for your use case
