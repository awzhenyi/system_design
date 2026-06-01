# Zookeeper Features

## Overview

Zookeeper provides a rich set of features that enable distributed coordination. Understanding these features in depth is crucial for designing effective distributed systems and excelling in system design interviews.

## Watches

### Event Notification Mechanism

Watches are Zookeeper's mechanism for notifying clients of changes to the data tree. They provide a way to build reactive systems that respond to changes in real-time.

**How Watches Work**:

1. **Client Sets Watch**: Client registers interest in a znode
2. **Change Occurs**: Znode is modified, created, or deleted
3. **Server Sends Event**: Server sends watch event to client
4. **Client Processes Event**: Client's watcher callback is invoked
5. **Watch is One-Time**: Watch is automatically removed after firing

**Watch Types**:
- **Node Watches**: Triggered on node data changes
- **Child Watches**: Triggered on children list changes
- **Existence Watches**: Triggered on node creation/deletion

### Types of Watches

**1. Data Watches (Node Watches)**:

Triggered when:
- Node data is modified (`setData`)
- Node is deleted
- Node is created (if watching parent)

**Example**:
```java
Stat stat = zk.exists("/app/config", new Watcher() {
    @Override
    public void process(WatchedEvent event) {
        if (event.getType() == Event.EventType.NodeDataChanged) {
            // Config changed, reload it
            reloadConfig();
        }
    }
}, null);
```

**2. Child Watches**:

Triggered when:
- Child is added
- Child is removed
- Child list changes

**Example**:
```java
List<String> children = zk.getChildren("/services", new Watcher() {
    @Override
    public void process(WatchedEvent event) {
        if (event.getType() == Event.EventType.NodeChildrenChanged) {
            // Service list changed, rediscover
            discoverServices();
        }
    }
}, null);
```

**3. Existence Watches**:

Triggered when:
- Node is created
- Node is deleted

**Example**:
```java
Stat stat = zk.exists("/leader", new Watcher() {
    @Override
    public void process(WatchedEvent event) {
        if (event.getType() == Event.EventType.NodeCreated) {
            // Leader elected
            onLeaderElected();
        } else if (event.getType() == Event.EventType.NodeDeleted) {
            // Leader lost
            onLeaderLost();
        }
    }
}, null);
```

### Watch Guarantees

**One-Time Watches**:
- Watch fires once, then is automatically removed
- Client must re-register watch if it wants to continue monitoring
- Prevents infinite watch accumulation

**Ordering Guarantees**:
- Watches are ordered: client sees events in the order they occur
- Watch event is sent before the operation returns to the client that triggered it
- All clients see the same order of events

**Delivery Guarantees**:
- Watch is delivered exactly once (if connection maintained)
- If connection lost, watch may be lost (no delivery guarantee)
- Client must handle missed events (re-register and check state)

**State Change Guarantees**:
- Watch fires only if state actually changed
- Multiple operations that don't change state won't trigger watch
- Example: Setting data to same value won't trigger watch

### Watch Implementation Details

**Server-Side**:
- Watches stored in memory (not persisted)
- Watches are per-session
- Watches cleared on session expiration

**Client-Side**:
- Watches are asynchronous callbacks
- Executed in a separate thread (event thread)
- Should not block (do minimal work, delegate to worker threads)

**Performance Implications**:
- Watches are lightweight (just a callback registration)
- No polling overhead
- Efficient for reactive systems
- But: Lost watches on connection issues

### Common Watch Patterns

**Pattern 1: Continuous Monitoring**:
```java
public void monitorConfig() {
    try {
        byte[] data = zk.getData("/app/config", new Watcher() {
            @Override
            public void process(WatchedEvent event) {
                if (event.getType() == Event.EventType.NodeDataChanged) {
                    monitorConfig(); // Re-register watch
                }
            }
        }, null);
        
        updateConfig(new String(data));
    } catch (Exception e) {
        // Handle error and retry
    }
}
```

**Pattern 2: Leader Election Watch**:
```java
private void watchPredecessor(String path) {
    try {
        Stat stat = zk.exists(path, new Watcher() {
            @Override
            public void process(WatchedEvent event) {
                if (event.getType() == Event.EventType.NodeDeleted) {
                    checkAndBecomeLeader();
                }
            }
        });
        
        if (stat == null) {
            // Predecessor already gone
            checkAndBecomeLeader();
        }
    } catch (Exception e) {
        // Handle error
    }
}
```

**Pattern 3: Service Discovery Watch**:
```java
public void discoverServices() {
    try {
        List<String> children = zk.getChildren("/services", new Watcher() {
            @Override
            public void process(WatchedEvent event) {
                if (event.getType() == Event.EventType.NodeChildrenChanged) {
                    discoverServices(); // Rediscover and rewatch
                }
            }
        }, null);
        
        updateServiceList(children);
    } catch (Exception e) {
        // Handle error
    }
}
```

### Watch Limitations and Gotchas

**Limitation 1: Lost Events on Disconnection**:
- If client disconnects, watches are lost
- Client may miss events during disconnection
- Solution: Re-register watches and check current state

**Limitation 2: One-Time Only**:
- Watch fires once, then removed
- Must re-register to continue monitoring
- Easy to forget to re-register

**Limitation 3: No Guaranteed Delivery**:
- If connection lost, watch may not be delivered
- Must handle missed events
- Check state after reconnection

**Gotcha 1: Watch on Non-Existent Node**:
```java
// This watch will fire when node is created
Stat stat = zk.exists("/new-node", watcher, null);
// If node doesn't exist, stat is null but watch is set
```

**Gotcha 2: Watch Fires Before Operation Returns**:
- Watch event sent before `setData` returns
- Client may see watch before seeing operation result
- Design handlers to be idempotent

**Gotcha 3: Multiple Watches on Same Node**:
- Each `getData`/`exists`/`getChildren` can set a watch
- Multiple watches on same node all fire on change
- Be careful not to set duplicate watches

### Best Practices for Watch Usage

**1. Always Re-register Watches**:
```java
// Good: Re-register in watch handler
zk.getData(path, new Watcher() {
    public void process(WatchedEvent event) {
        handleChange();
        reRegisterWatch(); // Re-register
    }
}, null);
```

**2. Handle Connection Loss**:
```java
// Good: Re-register on reconnection
zk.register(new Watcher() {
    public void process(WatchedEvent event) {
        if (event.getState() == KeeperState.SyncConnected) {
            reRegisterAllWatches();
        }
    }
});
```

**3. Check State After Reconnection**:
```java
// Good: Check state, not just rely on watches
public void onReconnect() {
    // Re-register watches
    reRegisterWatches();
    
    // Also check current state (might have missed changes)
    checkCurrentState();
}
```

**4. Don't Block in Watch Handlers**:
```java
// Bad: Blocking in watch handler
zk.getData(path, new Watcher() {
    public void process(WatchedEvent event) {
        doLongRunningWork(); // Blocks event thread
    }
}, null);

// Good: Delegate to worker thread
zk.getData(path, new Watcher() {
    public void process(WatchedEvent event) {
        executor.submit(() -> doLongRunningWork());
    }
}, null);
```

**5. Handle Exceptions in Watch Handlers**:
```java
// Good: Handle exceptions
zk.getData(path, new Watcher() {
    public void process(WatchedEvent event) {
        try {
            handleChange();
        } catch (Exception e) {
            logger.error("Error handling watch", e);
            // Re-register watch even on error
            reRegisterWatch();
        }
    }
}, null);
```

### Code Examples for Watch-Based Coordination

**Example 1: Configuration Manager with Watches**:
```java
public class WatchBasedConfigManager {
    private final ZooKeeper zk;
    private final String configPath;
    private volatile Map<String, String> config = new HashMap<>();
    
    public WatchBasedConfigManager(ZooKeeper zk, String configPath) {
        this.zk = zk;
        this.configPath = configPath;
        startWatching();
    }
    
    private void startWatching() {
        watchConfig();
    }
    
    private void watchConfig() {
        try {
            byte[] data = zk.getData(configPath, new Watcher() {
                @Override
                public void process(WatchedEvent event) {
                    if (event.getType() == Event.EventType.NodeDataChanged) {
                        watchConfig(); // Re-register
                    }
                }
            }, null);
            
            updateConfig(parseConfig(new String(data)));
        } catch (Exception e) {
            // Retry after delay
            scheduler.schedule(this::watchConfig, 1, TimeUnit.SECONDS);
        }
    }
    
    private void updateConfig(Map<String, String> newConfig) {
        Map<String, String> oldConfig = this.config;
        this.config = newConfig;
        notifyListeners(oldConfig, newConfig);
    }
    
    public String getConfig(String key) {
        return config.get(key);
    }
}
```

**Example 2: Leader Election with Watches**:
```java
public class WatchBasedLeaderElection {
    private final ZooKeeper zk;
    private final String electionPath;
    private String myPath;
    private volatile boolean isLeader = false;
    
    public void participate() throws KeeperException, InterruptedException {
        myPath = zk.create(electionPath + "/candidate-",
                          "my-id".getBytes(),
                          ZooDefs.Ids.OPEN_ACL_UNSAFE,
                          CreateMode.EPHEMERAL_SEQUENTIAL);
        
        checkLeadership();
    }
    
    private void checkLeadership() throws KeeperException, InterruptedException {
        List<String> children = zk.getChildren(electionPath, false);
        Collections.sort(children);
        
        String mySeq = myPath.substring(myPath.lastIndexOf('/') + 1);
        int myIndex = children.indexOf(mySeq);
        
        if (myIndex == 0) {
            becomeLeader();
        } else {
            watchPredecessor(children.get(myIndex - 1));
        }
    }
    
    private void watchPredecessor(String predecessorSeq) {
        String predecessorPath = electionPath + "/" + predecessorSeq;
        try {
            Stat stat = zk.exists(predecessorPath, new Watcher() {
                @Override
                public void process(WatchedEvent event) {
                    if (event.getType() == Event.EventType.NodeDeleted) {
                        try {
                            checkLeadership();
                        } catch (Exception e) {
                            // Handle
                        }
                    }
                }
            });
            
            if (stat == null) {
                // Predecessor already gone
                checkLeadership();
            }
        } catch (Exception e) {
            // Handle error
        }
    }
    
    private void becomeLeader() {
        isLeader = true;
        onElected();
    }
}
```

## Sessions

### Session Lifecycle

**Session Creation**:
- Created when client connects to Zookeeper
- Assigned unique session ID
- Has timeout (session timeout)

**Session States**:
1. **CONNECTING**: Initial connection attempt
2. **CONNECTED**: Connected and authenticated
3. **RECONNECTING**: Connection lost, attempting to reconnect
4. **EXPIRED**: Session timeout exceeded
5. **CLOSED**: Session explicitly closed

**Session Lifecycle Events**:
```java
zk.register(new Watcher() {
    @Override
    public void process(WatchedEvent event) {
        KeeperState state = event.getState();
        switch (state) {
            case SyncConnected:
                // Session established
                break;
            case Disconnected:
                // Temporary disconnection
                break;
            case Expired:
                // Session expired, must recreate
                break;
            case AuthFailed:
                // Authentication failed
                break;
        }
    }
});
```

### Session Timeout Handling

**Session Timeout**:
- Configured when creating Zookeeper client
- Server enforces minimum and maximum
- Typical range: 2-20 seconds (server configurable)

**Timeout Calculation**:
- Client proposes timeout
- Server accepts if within min/max bounds
- Server's decision is final

**What Happens on Timeout**:
- All ephemeral znodes deleted
- All watches cleared
- Session becomes invalid
- Client must create new session

**Handling Timeout**:
```java
public class SessionManager {
    private ZooKeeper zk;
    private final int sessionTimeout;
    
    public SessionManager(int sessionTimeout) {
        this.sessionTimeout = sessionTimeout;
        connect();
    }
    
    private void connect() {
        try {
            zk = new ZooKeeper("localhost:2181", sessionTimeout, new Watcher() {
                @Override
                public void process(WatchedEvent event) {
                    if (event.getState() == KeeperState.Expired) {
                        // Session expired, reconnect
                        reconnect();
                    } else if (event.getState() == KeeperState.Disconnected) {
                        // Temporary disconnection, wait for reconnect
                    } else if (event.getState() == KeeperState.SyncConnected) {
                        // Reconnected, restore state
                        onReconnected();
                    }
                }
            });
        } catch (IOException e) {
            // Handle
        }
    }
    
    private void reconnect() {
        // Close old connection
        try {
            zk.close();
        } catch (InterruptedException e) {
            // Handle
        }
        
        // Create new session
        connect();
        
        // Restore ephemeral nodes, watches, etc.
        restoreState();
    }
}
```

### Reconnection Strategies

**Automatic Reconnection**:
- Zookeeper client library handles reconnection automatically
- Client maintains connection string
- Automatically tries to reconnect on failure

**Reconnection Behavior**:
- Tries to reconnect to same or different server
- Maintains session if reconnected within timeout
- Session expires if reconnection takes too long

**Handling Reconnection**:
```java
public class ResilientZookeeperClient {
    private ZooKeeper zk;
    private final String connectString;
    private final int sessionTimeout;
    
    public void ensureConnected() {
        if (zk == null || zk.getState() != States.CONNECTED) {
            reconnect();
        }
    }
    
    private void reconnect() {
        try {
            if (zk != null) {
                zk.close();
            }
            
            zk = new ZooKeeper(connectString, sessionTimeout, new Watcher() {
                @Override
                public void process(WatchedEvent event) {
                    handleSessionEvent(event);
                }
            });
            
            // Wait for connection
            CountDownLatch connected = new CountDownLatch(1);
            // ... wait logic
            
        } catch (Exception e) {
            // Retry after delay
            scheduler.schedule(this::reconnect, 1, TimeUnit.SECONDS);
        }
    }
}
```

### Session State and Ephemeral Node Management

**Ephemeral Node Lifecycle**:
- Created when session is active
- Automatically deleted when session ends
- Survives temporary disconnections (if reconnected in time)
- Deleted on session expiration

**Managing Ephemeral Nodes**:
```java
public class EphemeralNodeManager {
    private final ZooKeeper zk;
    private final Set<String> ephemeralPaths = new HashSet<>();
    
    public void createEphemeralNode(String path, byte[] data) 
            throws KeeperException, InterruptedException {
        String createdPath = zk.create(path, data,
                                       ZooDefs.Ids.OPEN_ACL_UNSAFE,
                                       CreateMode.EPHEMERAL);
        ephemeralPaths.add(createdPath);
    }
    
    public void restoreEphemeralNodes() {
        // After reconnection, recreate ephemeral nodes
        for (String path : ephemeralPaths) {
            try {
                // Recreate if doesn't exist
                if (zk.exists(path, false) == null) {
                    createEphemeralNode(path, getDataForPath(path));
                }
            } catch (Exception e) {
                // Handle
            }
        }
    }
}
```

### Handling Session Expiration in Applications

**Session Expiration Impact**:
- All ephemeral nodes deleted
- All watches cleared
- Must recreate session
- Must restore application state

**Application-Level Handling**:
```java
public class ApplicationSessionHandler {
    private final SessionManager sessionManager;
    private final StateRestorer stateRestorer;
    
    public void handleSessionExpiration() {
        // 1. Detect expiration
        sessionManager.registerWatcher(new Watcher() {
            public void process(WatchedEvent event) {
                if (event.getState() == KeeperState.Expired) {
                    onSessionExpired();
                }
            }
        });
    }
    
    private void onSessionExpired() {
        // 2. Recreate session
        sessionManager.reconnect();
        
        // 3. Restore state
        stateRestorer.restore();
        
        // 4. Re-register ephemeral nodes
        stateRestorer.recreateEphemeralNodes();
        
        // 5. Re-register watches
        stateRestorer.reRegisterWatches();
        
        // 6. Notify application
        notifyApplicationOfReconnection();
    }
}
```

### Heartbeat Mechanism and Keep-Alive

**Heartbeat**:
- Client sends periodic heartbeats to server
- Keeps session alive
- Frequency: typically sessionTimeout / 3

**Keep-Alive**:
- Server expects heartbeats within timeout
- If no heartbeat received, session expires
- Client library handles heartbeats automatically

**Tuning**:
- Shorter timeout = faster failure detection
- But more sensitive to network hiccups
- Balance based on use case

### Multi-Threaded Session Handling

**Thread Safety**:
- Zookeeper client is thread-safe
- Multiple threads can use same client
- But: operations are queued and processed sequentially

**Best Practices**:
```java
// Good: Share single client across threads
ZooKeeper zk = new ZooKeeper(...);
// Multiple threads can use 'zk' safely

// Bad: Create client per thread
// Each thread creates its own client (wasteful)
```

**Connection Pooling**:
- Generally not needed (client handles connection)
- One client per application is typical
- Can create multiple clients for isolation if needed

## ACLs (Access Control Lists)

### Security Model

**ACL Structure**:
- Each znode has an ACL
- ACL = list of (scheme, id, permissions) tuples
- Permissions: READ, WRITE, CREATE, DELETE, ADMIN

**ACL Evaluation**:
- Checked on every operation
- First matching ACL entry determines access
- Order matters (checked in order)

### Authentication Schemes

**1. World Scheme**:
- `world:anyone` - anyone can access
- No authentication required
- Use for public data only

**2. Auth Scheme**:
- `auth:` - any authenticated user
- User must be authenticated
- Use when all authenticated users should have access

**3. Digest Scheme**:
- `digest:username:password` (SHA1 hash)
- Username/password authentication
- Password stored as hash

**4. IP Scheme**:
- `ip:192.168.1.1` - IP address-based
- Access based on client IP
- Use for network-based access control

**5. SASL Scheme**:
- `sasl:user` - SASL authentication
- Supports Kerberos, LDAP, etc.
- Use for enterprise authentication

### Permission Types

**READ**: Read znode data and list children
**WRITE**: Modify znode data
**CREATE**: Create child znodes
**DELETE**: Delete child znodes
**ADMIN**: Modify ACL

**Permission Combinations**:
- Can combine permissions (READ | WRITE)
- All permissions: READ | WRITE | CREATE | DELETE | ADMIN

### ACL Inheritance and Default Permissions

**Inheritance**:
- Children inherit parent ACL by default
- Can override at child level
- Simplifies security management

**Default ACL**:
- `OPEN_ACL_UNSAFE`: world:anyone with all permissions
- `READ_ACL_UNSAFE`: world:anyone with read only
- `CREATOR_ALL_ACL`: creator gets all permissions

**Example**:
```java
// Create with default ACL
zk.create("/app/config", data, ZooDefs.Ids.OPEN_ACL_UNSAFE, CreateMode.PERSISTENT);

// Create with custom ACL
List<ACL> acl = new ArrayList<>();
acl.add(new ACL(ZooDefs.Perms.READ | ZooDefs.Perms.WRITE,
                new Id("digest", "user:password")));
zk.create("/app/config", data, acl, CreateMode.PERSISTENT);
```

### Security Best Practices

**1. Use Authentication**:
```java
// Good: Authenticate before operations
zk.addAuthInfo("digest", "user:password".getBytes());
```

**2. Principle of Least Privilege**:
```java
// Good: Give minimum required permissions
acl.add(new ACL(ZooDefs.Perms.READ, new Id("digest", "reader:pass")));
```

**3. Secure Sensitive Data**:
```java
// Good: Use ACLs for sensitive config
List<ACL> acl = Arrays.asList(
    new ACL(ZooDefs.Perms.READ | ZooDefs.Perms.WRITE,
            new Id("digest", "admin:pass"))
);
```

**4. Regular ACL Audits**:
- Review ACLs periodically
- Remove unused permissions
- Update on personnel changes

### Integration with Authentication Systems

**Kerberos Integration**:
```java
// Use SASL with Kerberos
System.setProperty("java.security.auth.login.config", "/path/to/jaas.conf");
zk.addAuthInfo("sasl", null);
```

**LDAP Integration**:
- Use SASL with LDAP
- Configure in Zookeeper server
- Clients authenticate via SASL

## Atomic Operations

### Conditional Updates with Version Checking

**Optimistic Concurrency Control**:
```java
public boolean updateIfVersionMatches(String path, byte[] newData, int expectedVersion) {
    try {
        Stat stat = zk.exists(path, false);
        if (stat.getVersion() == expectedVersion) {
            zk.setData(path, newData, expectedVersion);
            return true;
        }
        return false; // Version mismatch
    } catch (KeeperException.BadVersionException e) {
        return false;
    }
}
```

**Compare-and-Swap Pattern**:
```java
public boolean compareAndSwap(String path, byte[] expected, byte[] update) {
    while (true) {
        Stat stat = new Stat();
        byte[] current = zk.getData(path, false, stat);
        
        if (Arrays.equals(current, expected)) {
            try {
                zk.setData(path, update, stat.getVersion());
                return true;
            } catch (KeeperException.BadVersionException e) {
                // Retry
                continue;
            }
        } else {
            return false; // Value changed
        }
    }
}
```

### Multi-Znode Transactions

**Transaction API**:
```java
List<Op> ops = Arrays.asList(
    Op.create("/app/config", "data".getBytes(), ZooDefs.Ids.OPEN_ACL_UNSAFE, CreateMode.PERSISTENT),
    Op.setData("/app/version", "v2".getBytes(), -1),
    Op.delete("/app/old", -1)
);

List<OpResult> results = zk.multi(ops);
// All operations succeed or all fail (atomic)
```

**Use Cases**:
- Update multiple znodes atomically
- Ensure consistency across operations
- Avoid partial updates

### Optimistic Concurrency Control Patterns

**Pattern 1: Retry on Conflict**:
```java
public void updateWithRetry(String path, byte[] newData, int maxRetries) {
    for (int i = 0; i < maxRetries; i++) {
        try {
            Stat stat = zk.exists(path, false);
            zk.setData(path, newData, stat.getVersion());
            return; // Success
        } catch (KeeperException.BadVersionException e) {
            if (i == maxRetries - 1) {
                throw e; // Give up
            }
            // Retry after delay
            Thread.sleep(100);
        }
    }
}
```

**Pattern 2: Read-Modify-Write**:
```java
public void incrementCounter(String path) throws KeeperException, InterruptedException {
    while (true) {
        Stat stat = new Stat();
        byte[] data = zk.getData(path, false, stat);
        long current = Long.parseLong(new String(data));
        long next = current + 1;
        
        try {
            zk.setData(path, String.valueOf(next).getBytes(), stat.getVersion());
            return; // Success
        } catch (KeeperException.BadVersionException e) {
            // Retry (someone else modified it)
            continue;
        }
    }
}
```

### Compare-and-Swap Operations

**CAS Implementation**:
```java
public boolean compareAndSet(String path, byte[] expected, byte[] update) {
    while (true) {
        Stat stat = new Stat();
        byte[] current = zk.getData(path, false, stat);
        
        if (!Arrays.equals(current, expected)) {
            return false; // Value doesn't match
        }
        
        try {
            zk.setData(path, update, stat.getVersion());
            return true; // Success
        } catch (KeeperException.BadVersionException e) {
            // Retry (value changed between read and write)
            continue;
        }
    }
}
```

### Handling Concurrent Updates

**Strategy 1: Last Writer Wins**:
```java
// Just overwrite, no version check
zk.setData(path, data, -1);
```

**Strategy 2: First Writer Wins**:
```java
// Only update if not modified
Stat stat = zk.exists(path, false);
if (stat.getVersion() == expectedVersion) {
    zk.setData(path, data, expectedVersion);
}
```

**Strategy 3: Merge Conflicts**:
```java
// Read both versions, merge, write
Stat stat1 = zk.exists(path, false);
byte[] data1 = zk.getData(path, false, stat1);
// ... get other version ...
byte[] merged = merge(data1, data2);
zk.setData(path, merged, stat1.getVersion());
```

## Sequential Consistency

### Ordering Guarantees

**Write Ordering**:
- All writes are totally ordered
- Order determined by zxid (transaction ID)
- All servers see same order

**Read Ordering**:
- Reads see writes in order
- Sequential consistency: all operations appear to execute in some sequential order
- Consistent with program order

**Causal Ordering**:
- If operation A happens before B, all clients see A before B
- Maintains causality relationships

### How Sequential Consistency is Achieved

**ZAB Protocol**:
- All writes go through leader
- Leader assigns zxid (ordered)
- All servers apply in zxid order
- Ensures global ordering

### Implications for Application Design

**Design for Ordering**:
- Don't assume operations complete in submission order
- Use zxid or version numbers for ordering
- Handle out-of-order operations gracefully

**When Ordering Matters**:
- Leader election (order matters)
- Distributed locks (order matters)
- Configuration updates (order may matter)

**When Ordering Doesn't Matter**:
- Independent operations
- Idempotent operations
- Operations on different znodes

## High Availability

### Failover Mechanisms

**Automatic Leader Election**:
- On leader failure, new leader elected automatically
- Election time: ~200ms typically
- Cluster continues operating

**Client Failover**:
- Client automatically reconnects to another server
- Transparent to application (if session maintained)
- Must handle session expiration

### Recovery Time Objectives (RTO) and Recovery Point Objectives (RPO)

**RTO (Recovery Time Objective)**:
- Time to recover from failure
- Leader election: ~200ms
- Full recovery: ~1-2 seconds
- Depends on cluster size and network

**RPO (Recovery Point Objective)**:
- Maximum acceptable data loss
- Zookeeper: Zero data loss (all committed writes durable)
- Uncommitted writes may be lost

### Multi-Datacenter Deployment Considerations

**Challenges**:
- Network latency between datacenters
- Network partitions
- Consistency vs availability

**Solutions**:
1. **Per-DC Zookeeper**: Each datacenter has its own cluster
2. **Observer Mode**: Use observers for cross-DC replication
3. **Application-Level**: Application coordinates across DCs

## Performance

### Read vs Write Patterns

**Read Performance**:
- Very fast: `<1ms` (local memory access)
- No consensus needed
- Can scale with observers
- Throughput: 100K+ reads/sec per server

**Write Performance**:
- Slower: 5-10ms (consensus required)
- Must go through leader
- Limited by consensus
- Throughput: ~10K-20K writes/sec per cluster

### Throughput and Latency Characteristics

**Typical Performance**:
- Read latency: `<1ms`
- Write latency: 5-10ms (P50), 20-50ms (P99)
- Read throughput: 100K+ ops/sec per server
- Write throughput: 10K-20K ops/sec per cluster

### Performance Bottlenecks

**Write Bottlenecks**:
1. Consensus overhead
2. Network latency
3. Disk I/O (transaction log)
4. Single leader (serialization)

**Read Bottlenecks**:
1. Memory access (minimal)
2. Network (if remote)
3. CPU (deserialization)

### Optimization Strategies

**Write Optimization**:
- Batch operations
- Reduce write frequency
- Use fast SSDs for transaction log
- Tune snapshot frequency

**Read Optimization**:
- Add observers for scaling
- Use local reads (default)
- Client-side caching
- Connection pooling

### Observer Nodes for Read Scaling

**Observers**:
- Don't participate in consensus
- Can serve reads
- Don't affect write quorum
- Scale reads without affecting writes

**When to Use**:
- Read-heavy workloads
- Cross-datacenter replication
- Scaling reads without write impact

### Performance Tuning Guidelines

**Server Tuning**:
- JVM heap size
- Transaction log disk (fast SSD)
- Snapshot disk (can be slower)
- Network tuning

**Client Tuning**:
- Session timeout
- Connection timeout
- Request timeout
- Batch size

## Operational Features

### Monitoring and Metrics

**JMX Metrics**:
- Connection count
- Request latency
- Request throughput
- Server state

**Four-Letter Commands**:
- `ruok`: Is server running?
- `stat`: Server statistics
- `mntr`: Detailed metrics
- `conf`: Configuration

### Logging and Debugging

**Log Files**:
- Transaction log
- Snapshot files
- Server logs
- Client logs

**Debugging**:
- Enable debug logging
- Monitor watch counts
- Check connection state
- Analyze transaction log

### Backup and Restore Procedures

**Backup**:
- Copy snapshot files
- Copy transaction logs
- Regular backups recommended

**Restore**:
- Stop server
- Restore snapshot and logs
- Start server
- Verify data

### Upgrade Strategies

**Rolling Upgrade**:
1. Upgrade one server at a time
2. Ensure quorum maintained
3. Wait for synchronization
4. Continue with next server

**Zero-Downtime Upgrade**:
- Possible with proper planning
- Maintain quorum throughout
- Test in staging first

### Common Operational Issues and Solutions

**Issue 1: Out of Memory**:
- Symptom: Server crashes
- Solution: Increase heap, reduce data size

**Issue 2: Disk Full**:
- Symptom: Writes fail
- Solution: Clean old snapshots, increase disk

**Issue 3: Network Partitions**:
- Symptom: Cluster splits
- Solution: Fix network, ensure low latency

**Issue 4: Leader Election Loops**:
- Symptom: Frequent elections
- Solution: Increase timeouts, check network

## Summary

Zookeeper's features provide:

- **Watches**: Real-time event notifications
- **Sessions**: Connection management and failure handling
- **ACLs**: Security and access control
- **Atomic Operations**: Consistency guarantees
- **Sequential Consistency**: Ordering guarantees
- **High Availability**: Automatic failover
- **Performance**: Optimized for coordination workloads

Understanding these features is essential for:
- Designing effective distributed systems
- Handling failures gracefully
- Optimizing performance
- Ensuring security
- Operating Zookeeper in production

These features work together to provide a robust coordination service for distributed systems.
