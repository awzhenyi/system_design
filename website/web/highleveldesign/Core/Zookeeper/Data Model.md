# Zookeeper Data Model

## Overview

Zookeeper's data model is a hierarchical namespace similar to a file system. Understanding this model is crucial for designing effective coordination patterns and avoiding common pitfalls.

## Hierarchical Namespace (Tree Structure)

### File System-like Organization

Zookeeper organizes data as a tree of nodes (called znodes), similar to a file system:

```
/
├── /app
│   ├── /app/config
│   │   └── /app/config/database
│   └── /app/locks
│       └── /app/locks/resource-1
├── /services
│   ├── /services/service-a
│   │   ├── /services/service-a/node-1
│   │   └── /services/service-a/node-2
│   └── /services/service-b
└── /coordination
    ├── /coordination/leader
    └── /coordination/members
```

### Path Semantics and Navigation

**Path Rules**:
- Absolute paths start with `/`
- Paths are case-sensitive
- No relative paths (all paths are absolute)
- Path components separated by `/`
- `.` and `..` are not supported (unlike file systems)

**Valid Paths**:
- `/app/config`
- `/services/service-a/node-1`
- `/coordination/leader`

**Invalid Paths**:
- `app/config` (relative path)
- `/app/../services` (no `..` support)
- `/app//config` (double slashes, though some clients handle this)

### Why Hierarchical vs Flat Structure

**Advantages of Hierarchical**:

1. **Organization**: Natural grouping of related data
   - `/app/config` - application configuration
   - `/app/locks` - application locks
   - `/services/service-a` - service-specific data

2. **Namespace Isolation**: Different applications can use different subtrees
   - `/app1/*` vs `/app2/*`
   - Prevents naming conflicts

3. **ACL Inheritance**: Permissions can be set at parent level
   - `/app` ACL applies to all children
   - Simplifies security management

4. **Efficient Operations**: Can operate on subtrees
   - Delete entire subtree
   - List children of a node

5. **Intuitive**: Familiar file system metaphor
   - Easy to understand and navigate
   - Natural for developers

**Flat Structure Limitations**:
- No natural organization
- Harder to manage permissions
- No subtree operations
- Namespace conflicts

## Znodes Concept

### Znode vs File System Node Differences

**Similarities**:
- Both have paths
- Both can contain data
- Both can have children
- Both support metadata

**Key Differences**:

| Aspect | File System | Zookeeper Znode |
|--------|-------------|-----------------|
| Data Size | Unlimited | ~1MB limit |
| Children | Can be files or directories | All znodes can have children |
| Ephemeral | No | Yes (ephemeral znodes) |
| Sequential | No | Yes (sequential znodes) |
| Watches | No | Yes (built-in) |
| Versioning | No | Yes (optimistic concurrency) |
| Atomic Operations | Limited | Full support |

### In-Memory Representation

**Storage Model**:
- All znodes stored in memory for fast access
- Transaction log for durability
- Periodic snapshots for recovery

**Data Structure**:
```java
class ZNode {
    byte[] data;              // Node data (max 1MB)
    List<String> children;    // Child node names
    Stat stat;               // Metadata (version, timestamps, ACL)
    Set<Watcher> watchers;   // Active watches
}
```

**Memory Efficiency**:
- Only path and data stored (not full path for each child)
- Children stored as list of names (not full paths)
- Efficient for large namespaces

### Data and Metadata Storage

**Data**:
- Stored as byte array
- Maximum size: 1MB (configurable, but not recommended to increase)
- Can be empty (znode exists but no data)

**Metadata (Stat Structure)**:
- **czxid**: Transaction ID that created this znode
- **mzxid**: Transaction ID that last modified this znode
- **ctime**: Creation time (milliseconds since epoch)
- **mtime**: Last modification time
- **version**: Data version (increments on each update)
- **cversion**: Children version (increments when children change)
- **aversion**: ACL version (increments when ACL changes)
- **ephemeralOwner**: Session ID if ephemeral, 0 if persistent
- **dataLength**: Length of data
- **numChildren**: Number of children
- **pzxid**: Transaction ID of last child modification

## Types of Znodes

### Persistent Znodes

**Characteristics**:
- Created and persist until explicitly deleted
- Survive client disconnections
- Survive server restarts
- Default type if not specified

**Use Cases**:
- Configuration data
- Service metadata
- Long-lived coordination data
- Application state that should persist

**Example**:
```java
// Create persistent znode
zk.create("/app/config", 
         "database_url=localhost:5432".getBytes(),
         ZooDefs.Ids.OPEN_ACL_UNSAFE,
         CreateMode.PERSISTENT);
```

**Lifecycle**:
1. Created by client
2. Persists indefinitely
3. Deleted explicitly or when parent is deleted

### Ephemeral Znodes

**Characteristics**:
- Automatically deleted when creating session ends
- Cannot have children (ephemeral nodes are leaf nodes)
- Used for failure detection
- Tied to client session

**Use Cases**:
- Service registration (service goes away when it crashes)
- Leader election (leader node disappears if leader fails)
- Group membership (member removed when it disconnects)
- Locks (lock released when holder crashes)

**Example**:
```java
// Create ephemeral znode for service registration
zk.create("/services/my-service",
         "host:port".getBytes(),
         ZooDefs.Ids.OPEN_ACL_UNSAFE,
         CreateMode.EPHEMERAL);
// Znode automatically deleted when session ends
```

**Lifecycle**:
1. Created by client
2. Exists while client session is active
3. Automatically deleted when:
   - Client disconnects
   - Session expires (timeout)
   - Client explicitly closes session

**Failure Detection**:
- If client crashes, session expires (no heartbeat)
- Ephemeral znode automatically deleted
- Other clients can detect failure via watch on parent

### Sequential Znodes

**Characteristics**:
- Automatically assigned unique, monotonically increasing number
- Number appended to path
- Guarantees global ordering
- Can be combined with persistent or ephemeral

**Use Cases**:
- Leader election (lowest number wins)
- Distributed queues (FIFO ordering)
- Ordered task assignment
- Unique ID generation

**Example**:
```java
// Create sequential znode
String path = zk.create("/tasks/task-",
                       "task_data".getBytes(),
                       ZooDefs.Ids.OPEN_ACL_UNSAFE,
                       CreateMode.PERSISTENT_SEQUENTIAL);
// Path will be: /tasks/task-0000000001
// Next will be: /tasks/task-0000000002
```

**Numbering**:
- Format: 10-digit zero-padded number
- Example: `0000000001`, `0000000002`, etc.
- Guaranteed to be unique and monotonically increasing
- Wraps around after 2^31 (very unlikely in practice)

**Ordering Guarantees**:
- Global ordering across all sequential znodes
- Lower numbers created before higher numbers
- Useful for FIFO queues and leader election

### Combinations: Ephemeral-Sequential Patterns

**Ephemeral-Sequential**:
- Combines ephemeral and sequential
- Automatically deleted on session end
- Globally ordered
- **Most common pattern for coordination**

**Use Cases**:
- Leader election (ephemeral-sequential)
- Distributed locks (ephemeral-sequential)
- Group membership with ordering

**Example - Leader Election**:
```java
// Each candidate creates ephemeral-sequential znode
String myPath = zk.create("/election/candidate-",
                         "my_id".getBytes(),
                         ZooDefs.Ids.OPEN_ACL_UNSAFE,
                         CreateMode.EPHEMERAL_SEQUENTIAL);
// myPath = /election/candidate-0000000003

// Check if I'm the leader (lowest number)
List<String> children = zk.getChildren("/election", false);
Collections.sort(children);
String leader = "/election/" + children.get(0);
if (myPath.equals(leader)) {
    // I'm the leader!
}
```

**Why Ephemeral-Sequential for Leader Election**:
- Sequential: Provides ordering (lowest number = leader)
- Ephemeral: Automatic cleanup if candidate crashes
- No manual cleanup needed
- Natural failure detection

## Znode Metadata

### Version Numbers

**Three Types of Versions**:

1. **Data Version (version)**:
   - Increments on each data update
   - Used for optimistic concurrency control
   - Prevents lost updates

2. **Children Version (cversion)**:
   - Increments when children are added/removed
   - Tracks changes to child list

3. **ACL Version (aversion)**:
   - Increments when ACL is modified
   - Tracks security changes

**Version Usage**:
```java
// Optimistic concurrency control
Stat stat = zk.exists("/app/config", false);
byte[] data = zk.getData("/app/config", false, stat);
// Modify data...
// Update only if version hasn't changed
zk.setData("/app/config", newData, stat.getVersion());
// If version changed, throws BadVersionException
```

**Why Versions Matter**:
- Prevent lost updates (optimistic locking)
- Detect concurrent modifications
- Enable compare-and-swap operations

### Timestamps

**Creation Time (ctime)**:
- Milliseconds since epoch
- Set when znode is created
- Never changes

**Modification Time (mtime)**:
- Milliseconds since epoch
- Updated on each modification
- Useful for cache invalidation

**Usage**:
```java
Stat stat = zk.exists("/app/config", false);
long lastModified = stat.getMtime();
// Check if config changed since last read
if (lastModified > cachedTimestamp) {
    // Reload config
}
```

### ACL (Access Control Lists)

**ACL Structure**:
- Each znode has an ACL
- ACL is a list of (scheme, id, permissions) tuples
- Permissions: READ, WRITE, CREATE, DELETE, ADMIN

**Authentication Schemes**:
- **world**: Anyone (no authentication)
- **auth**: Authenticated users (any authenticated user)
- **digest**: Username:password (SHA1 hash)
- **ip**: IP address-based
- **sasl**: Kerberos, LDAP, etc.

**ACL Inheritance**:
- Children inherit parent ACL by default
- Can override at child level
- Simplifies security management

**Example**:
```java
// Create znode with ACL
List<ACL> acl = new ArrayList<>();
acl.add(new ACL(ZooDefs.Perms.READ | ZooDefs.Perms.WRITE,
                new Id("digest", "user:password")));
zk.create("/app/config", data, acl, CreateMode.PERSISTENT);
```

### Data Length and Storage Limits

**Data Size Limit**:
- Default: 1MB per znode (jute.maxbuffer)
- Configurable but not recommended to increase
- Why limit exists:
  - All data in memory (performance)
  - Large data slows down operations
  - Zookeeper is for coordination, not data storage

**Practical Limits**:
- Recommended: `<1KB` per znode
- Acceptable: `<10KB` per znode
- Avoid: `>100KB` per znode
- Never: `>1MB` per znode

**Total Data Size**:
- All data stored in memory
- Typical deployments: `<100GB` total
- Monitor memory usage

## Path Structure and Naming Conventions

### Best Practices for Path Design

**1. Use Hierarchical Organization**:
```
✅ Good:
/app/config
/app/locks
/services/service-a/nodes

❌ Bad:
/app_config
/app_locks
/service_a_nodes
```

**2. Use Descriptive Names**:
```
✅ Good:
/app/database/connection-pool/max-connections
/services/payment-service/instances

❌ Bad:
/app/db/cp/mc
/services/ps/i
```

**3. Avoid Deep Nesting**:
```
✅ Good (3-4 levels):
/app/config
/services/service-a/node-1

❌ Bad (too deep):
/app/config/database/connection/pool/settings/max/connections
```

**4. Use Consistent Conventions**:
```
✅ Good (consistent):
/services/service-a
/services/service-b
/services/service-c

❌ Bad (inconsistent):
/services/service-a
/services/ServiceB
/services/service_c
```

### Namespace Organization Patterns

**Pattern 1: Application-Based**:
```
/app1/config
/app1/locks
/app1/coordination
/app2/config
/app2/locks
```

**Pattern 2: Function-Based**:
```
/config/app1
/config/app2
/locks/app1
/locks/app2
/coordination/app1
```

**Pattern 3: Service-Based**:
```
/services/service-a/config
/services/service-a/instances
/services/service-a/locks
```

**Recommendation**: Choose based on your use case, but be consistent.

### Common Anti-patterns to Avoid

**1. Flat Structure**:
```
❌ Bad:
/node1
/node2
/node3
/config1
/config2

✅ Good:
/app/nodes/node1
/app/config/config1
```

**2. Inconsistent Naming**:
```
❌ Bad:
/App/Config
/app_config
/APP/CONFIG

✅ Good:
/app/config
```

**3. Too Deep Nesting**:
```
❌ Bad:
/app/config/database/connection/pool/settings/max/connections

✅ Good:
/app/config/db-max-connections
```

**4. Special Characters**:
```
❌ Bad:
/app/config@prod
/app/config#v1

✅ Good:
/app/config/prod
/app/config/v1
```

**5. Spaces in Paths**:
```
❌ Bad:
/app/my config
/app/config with spaces

✅ Good:
/app/my-config
/app/config-with-spaces
```

## Data Size Limitations

### Why Zookeeper is Not a Data Store

**Design Philosophy**:
- Zookeeper is for **coordination**, not data storage
- Small metadata and configuration only
- Large data belongs in databases or object storage

**Limitations**:
- 1MB per znode limit
- All data in memory (limits total size)
- Not optimized for large data operations
- No query capabilities (no SQL, no indexes)

**When to Use Zookeeper**:
- Configuration (small, frequently accessed)
- Coordination metadata (locks, elections)
- Service discovery (endpoints, not full service state)
- Small state (counters, flags)

**When NOT to Use Zookeeper**:
- Large configuration files (>100KB)
- User data
- Application logs
- Binary files
- Any data >1MB

### Performance Implications of Large Znodes

**Impact of Large Data**:

1. **Memory Usage**:
   - All znodes in memory
   - Large znodes consume more memory
   - Limits total number of znodes

2. **Network Transfer**:
   - Large data = more network bandwidth
   - Slower getData() operations
   - Impacts all servers (replication)

3. **Serialization**:
   - Large data = more CPU for serialization
   - Slower operations overall

4. **Snapshot/Log**:
   - Large data = larger snapshots
   - Slower recovery time

**Performance Guidelines**:
- `<1KB`: Excellent performance
- 1KB-10KB: Good performance
- 10KB-100KB: Acceptable, monitor
- 100KB-1MB: Poor performance, avoid
- `>1MB`: Not supported

### Design Patterns for Storing Larger Data

**Pattern 1: Reference Pattern**:
```
# Store reference in Zookeeper
/app/config/database-url = "s3://bucket/config.json"

# Actual data in S3/database
```

**Pattern 2: Chunking Pattern**:
```
# Split large data into chunks
/app/config/chunk-1 = "first 1KB"
/app/config/chunk-2 = "next 1KB"
/app/config/chunk-3 = "last 1KB"
```

**Pattern 3: External Storage**:
```
# Store metadata in Zookeeper
/app/config/version = "v1.2.3"
/app/config/location = "database://configs/123"

# Store actual data externally
```

**Pattern 4: Compression**:
```
# Compress data before storing
byte[] compressed = compress(largeData);
zk.create("/app/config", compressed, ...);
```

## Practical Examples of Znode Usage

### Code Examples for Creating/Managing Znodes

**Creating Persistent Znode**:
```java
ZooKeeper zk = new ZooKeeper("localhost:2181", 3000, null);

// Create persistent znode
String path = zk.create("/app/config",
                       "key=value".getBytes(),
                       ZooDefs.Ids.OPEN_ACL_UNSAFE,
                       CreateMode.PERSISTENT);
```

**Creating Ephemeral Znode**:
```java
// Create ephemeral znode (auto-deleted on disconnect)
String path = zk.create("/services/my-service",
                       "host:port".getBytes(),
                       ZooDefs.Ids.OPEN_ACL_UNSAFE,
                       CreateMode.EPHEMERAL);
```

**Creating Sequential Znode**:
```java
// Create sequential znode
String path = zk.create("/tasks/task-",
                       "task_data".getBytes(),
                       ZooDefs.Ids.OPEN_ACL_UNSAFE,
                       CreateMode.PERSISTENT_SEQUENTIAL);
// Returns: /tasks/task-0000000001
```

**Reading Znode Data**:
```java
// Read data
byte[] data = zk.getData("/app/config", false, null);
String value = new String(data);

// Read with stat
Stat stat = new Stat();
byte[] data = zk.getData("/app/config", false, stat);
System.out.println("Version: " + stat.getVersion());
System.out.println("Modified: " + new Date(stat.getMtime()));
```

**Updating Znode**:
```java
// Update data
zk.setData("/app/config", "new_value".getBytes(), -1);

// Conditional update (optimistic locking)
Stat stat = zk.exists("/app/config", false);
zk.setData("/app/config", "new_value".getBytes(), stat.getVersion());
```

**Deleting Znode**:
```java
// Delete znode
zk.delete("/app/config", -1);

// Conditional delete
Stat stat = zk.exists("/app/config", false);
zk.delete("/app/config", stat.getVersion());
```

**Listing Children**:
```java
// List children
List<String> children = zk.getChildren("/app", false);
for (String child : children) {
    System.out.println("/app/" + child);
}
```

### Common Patterns and Idioms

**Pattern 1: Ensure Path Exists**:
```java
public void ensurePath(ZooKeeper zk, String path) throws KeeperException, InterruptedException {
    String[] parts = path.split("/");
    StringBuilder currentPath = new StringBuilder();
    for (String part : parts) {
        if (part.isEmpty()) continue;
        currentPath.append("/").append(part);
        try {
            zk.create(currentPath.toString(), null, 
                     ZooDefs.Ids.OPEN_ACL_UNSAFE, CreateMode.PERSISTENT);
        } catch (KeeperException.NodeExistsException e) {
            // Already exists, continue
        }
    }
}
```

**Pattern 2: Atomic Update**:
```java
public boolean updateIfExists(ZooKeeper zk, String path, byte[] newData) {
    try {
        Stat stat = zk.exists(path, false);
        if (stat != null) {
            zk.setData(path, newData, stat.getVersion());
            return true;
        }
        return false;
    } catch (KeeperException.BadVersionException e) {
        // Retry or handle conflict
        return false;
    }
}
```

**Pattern 3: Safe Delete**:
```java
public void deleteRecursive(ZooKeeper zk, String path) throws KeeperException, InterruptedException {
    List<String> children = zk.getChildren(path, false);
    for (String child : children) {
        deleteRecursive(zk, path + "/" + child);
    }
    zk.delete(path, -1);
}
```

### Real-World Znode Structures

**Kafka Znode Structure**:
```
/brokers
  /ids
    /0 (broker metadata)
    /1
  /topics
    /my-topic
      /partitions
        /0
          /state (partition leader, ISR)
/controller (controller election)
/controller_epoch
/config
  /topics
    /my-topic (topic configuration)
```

**HBase Znode Structure**:
```
/hbase
  /master (master election)
  /backup-masters
  /region-in-transition
  /table
    /my-table (table metadata)
  /rs
    /regionserver-1 (region server registration)
```

**Service Discovery Structure**:
```
/services
  /payment-service
    /instances
      /node-1 (ephemeral, host:port)
      /node-2 (ephemeral, host:port)
  /user-service
    /instances
      /node-1 (ephemeral, host:port)
```

## Summary

Zookeeper's data model provides:

- **Hierarchical Organization**: Tree structure for natural grouping
- **Multiple Znode Types**: Persistent, ephemeral, sequential, and combinations
- **Rich Metadata**: Versions, timestamps, ACLs for advanced use cases
- **Size Limitations**: 1MB per znode (by design, not a data store)
- **Practical Patterns**: Common idioms for coordination primitives

Understanding the data model is essential for:
- Designing effective coordination patterns
- Avoiding performance issues (large znodes)
- Implementing leader election, locks, and other primitives
- Making informed decisions in system design interviews

The key insight: Zookeeper is optimized for small, frequently accessed coordination data, not large data storage. Design your system accordingly.
