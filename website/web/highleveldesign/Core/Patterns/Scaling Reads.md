# Scaling Reads

## Source

- [Scaling Reads | Hello Interview](https://www.hellointerview.com/learn/system-design/patterns/scaling-reads)

## Active Recall

<details>
<summary><strong>What problem does the scaling reads pattern solve?</strong></summary>

It solves the problem of serving very high read traffic without overwhelming the primary database. In many systems, reads grow much faster than writes: one Instagram post may create one write, but opening feeds can trigger hundreds or thousands of reads across metadata, users, likes, comments, and media.

The bottleneck is not usually a simple bug. It is bounded by physical limits: CPU, memory, disk I/O, network bandwidth, and database connection capacity.

</details>

<details>
<summary><strong>What is the usual progression for scaling reads?</strong></summary>

Use a progressive strategy:

1. Optimize reads inside the database.
2. Scale database reads horizontally.
3. Add external caching layers.

Start with the simplest option that addresses the bottleneck. Jumping directly to sharding, global caches, or complex invalidation systems can add unnecessary operational risk.

</details>

<details>
<summary><strong>How do indexes help with read scaling, and what do they cost?</strong></summary>

Indexes avoid scanning the entire table by maintaining a lookup structure for frequently queried columns. They are useful for `WHERE`, `JOIN`, range, sort, and foreign-key access patterns.

The tradeoff is write overhead. Every insert, update, or delete may also need to update one or more indexes, so too many indexes can slow writes and increase storage cost.

</details>

<details>
<summary><strong>When does denormalization help reads?</strong></summary>

Denormalization helps when read paths are slowed by joins, repeated aggregations, or multi-step lookups. Examples include storing `username` and `avatar_url` directly on a post, keeping a `like_count`, or using a materialized view for expensive reports.

The tradeoff is redundancy. Writes become more complex because multiple copies or derived values must stay consistent.

</details>

<details>
<summary><strong>How do read replicas scale reads?</strong></summary>

A primary database handles writes, while replicas serve read traffic. This offloads read pressure from the primary and allows read capacity to grow by adding replicas.

The key tradeoff is replication lag. Asynchronous replicas may be stale, which is acceptable for feeds, catalogs, reports, and analytics, but risky for financial or correctness-critical flows.

</details>

<details>
<summary><strong>When should sharding enter the design?</strong></summary>

Sharding should be considered when a single database and read replicas are not enough, especially when the dataset is huge or write scaling is also required. Each shard owns a subset of data, so reads and writes can be distributed across machines.

The cost is complexity: cross-shard queries, rebalancing, transactions, hot shards, and routing logic become harder.

</details>

<details>
<summary><strong>What role does caching play in read scaling?</strong></summary>

Caching serves frequently accessed data from memory or edge locations instead of repeatedly hitting the database. It can dramatically reduce latency and database load for hot objects, feeds, metadata, product pages, and API responses.

The main tradeoff is freshness. Cache invalidation, TTL choice, stampede protection, and consistency expectations must be designed explicitly.

</details>

<details>
<summary><strong>How do cache-aside, write-through, write-behind, and refresh-ahead differ?</strong></summary>

- **Cache-aside:** application reads cache first, loads from DB on miss, then stores in cache. Simple and common.
- **Write-through:** writes update cache and database together. Fresher reads, slower writes.
- **Write-behind:** writes update cache first and persist asynchronously. Fast writes, possible data loss.
- **Refresh-ahead:** refresh hot keys before they expire. Fewer misses, more complexity.

</details>

<details>
<summary><strong>What is a cache stampede?</strong></summary>

A cache stampede happens when a hot cache entry expires and many requests miss at the same time. If every request rebuilds the value from the database, the database can be overwhelmed.

Common mitigations include per-key locks, stale-while-revalidate, background refresh, probabilistic early expiration, and limiting concurrent rebuilds with semaphores.

</details>

<details>
<summary><strong>How should cache invalidation be handled when updates must be visible immediately?</strong></summary>

Use stronger invalidation patterns:

- Update or delete cache entries during the write path.
- Publish data-change events so cache services invalidate affected keys.
- Store versions with cached values and refresh when versions mismatch.
- Use write-through caching when the cache must stay current.

TTL-only invalidation is simpler but allows stale reads until expiration.

</details>

<details>
<summary><strong>When is a CDN the right read-scaling layer?</strong></summary>

A CDN is useful when users are geographically distributed or the workload includes cacheable static assets, media, API responses, or public content. Edge caching reduces latency and protects the origin from traffic spikes.

Design details include `Cache-Control`, `ETag`, TTLs, cache busting, purge APIs, and origin shielding.

</details>

<details>
<summary><strong>What should be monitored in a scaled-read architecture?</strong></summary>

Track read latency at P50/P95/P99, cache hit rate, database CPU and I/O, slow queries, replication lag, cache memory pressure, cache stampede signals, and CDN hit ratio.

These metrics tell you whether reads are blocked by query shape, database capacity, replica lag, cache effectiveness, or origin traffic.

</details>

## Interview Notes

- **Start with diagnosis:** estimate read-to-write ratio, identify hot reads, and name the bottleneck.
- **Scale progressively:** indexes and query tuning before replicas, replicas before sharding, caches when data can tolerate staleness or invalidation complexity.
- **Call out tradeoffs:** consistency vs latency, simplicity vs scale, write cost vs read speed, freshness vs cache hit rate.
- **Mention failure modes:** stale reads, replication lag, cache stampede, hot keys, cross-shard queries, and cache invalidation bugs.
- **Use examples:** feeds, product catalogs, video metadata, search results, dashboards, and popular content.

## Quick Revision Checklist

- [ ] Explain why read-heavy systems often have 10:1 to 100:1+ read-to-write ratios.
- [ ] Recall the three-step scaling progression.
- [ ] Explain the read/write tradeoff of indexes.
- [ ] Compare denormalization, materialized views, and precomputed counters.
- [ ] Explain read replicas and replication lag.
- [ ] Know when sharding is necessary and why it is complex.
- [ ] Compare cache-aside, write-through, write-behind, and refresh-ahead.
- [ ] Explain cache stampede prevention.
- [ ] Explain cache invalidation options for immediate visibility.
- [ ] Know when to add CDN or edge caching.
- [ ] List the most important metrics to monitor.

## Detailed Notes

<details>
<summary>Expand detailed notes</summary>

## Overview

The **Scaling Reads Pattern** addresses the challenge of serving high-volume read requests when applications grow from hundreds to millions of users. While writes create data, reads consume it - and read traffic often grows faster than write traffic. This pattern covers architectural strategies to handle massive read loads without overwhelming the primary database.

## The Problem

### Read-to-Write Ratio Imbalance

**The Core Challenge**:
- Read traffic typically far exceeds write traffic
- Standard read-to-write ratio: **10:1** to start
- Often reaches **100:1** or higher for content-heavy applications
- For every write, there are many reads

**Real-World Examples**:

**Instagram Feed**:
- Opening app: 100+ read operations (image metadata, user info, likes, comments)
- Posting photo: 1 write operation
- Ratio: 100:1 or higher

**Twitter/X**:
- For every tweet posted, thousands of users read it
- Ratio: 1000:1 or higher

**Amazon**:
- For every product uploaded, hundreds browse it
- Ratio: 100:1 or higher

**YouTube**:
- Billions of video views daily
- Millions of uploads daily
- Ratio: 1000:1 or higher

### Physical Constraints

**Why This is a Problem**:
- Not a software problem you can debug away
- **Physics**: CPU cores can only execute so many instructions per second
- **Memory**: Can only hold so much data
- **Disk I/O**: Bounded by spinning platters or SSD write cycles
- When you hit these constraints, more code won't help

**Database Bottlenecks**:
- Single database server has finite capacity
- CPU, memory, disk I/O all have limits
- Network bandwidth constraints
- Connection pool limits

## The Solution: Progressive Scaling Strategy

Read scaling follows a natural progression from simple optimization to complex distributed systems:

1. **Optimize read performance within your database**
2. **Scale your database horizontally**
3. **Add external caching layers**

Each step builds on the previous, and you should exhaust simpler solutions before moving to more complex ones.

## Optimize Within Your Database

### Indexing

**What is Indexing**:
- Data structure that improves speed of data retrieval
- Similar to book index - allows finding data without scanning entire table
- Trade-off: Faster reads, slower writes (index maintenance)

**Types of Indexes**:

1. **B-Tree Indexes** (Most Common):
   - Balanced tree structure
   - O(log n) lookup time
   - Good for range queries, equality searches
   - Default in most databases

2. **Hash Indexes**:
   - O(1) lookup time
   - Only for equality searches
   - No range queries

3. **Composite Indexes**:
   - Index on multiple columns
   - Order matters (leftmost prefix rule)
   - Example: `(user_id, created_at)` index

4. **Covering Indexes**:
   - Index contains all columns needed for query
   - Query can be answered from index alone
   - Avoids table lookup (very fast)

**Indexing Best Practices**:
- Index columns used in WHERE clauses
- Index foreign keys
- Index columns used in JOINs
- Don't over-index (slows writes)
- Monitor index usage (remove unused indexes)

**Example**:
```sql
-- Without index: Full table scan (slow)
SELECT * FROM posts WHERE user_id = 123;

-- With index: Index lookup (fast)
CREATE INDEX idx_posts_user_id ON posts(user_id);
SELECT * FROM posts WHERE user_id = 123;
```

### Hardware Upgrades

**Vertical Scaling (Scale Up)**:
- Increase CPU cores
- Increase RAM
- Faster disks (SSD vs HDD)
- More network bandwidth

**When to Use**:
- Quick fix for immediate capacity issues
- Before horizontal scaling
- When single server can handle load

**Limitations**:
- Eventually hits physical limits
- Expensive (non-linear cost increase)
- Single point of failure
- Not infinitely scalable

**Hardware Considerations**:
- **CPU**: More cores for parallel queries
- **RAM**: More memory for caching, larger buffer pools
- **Disk**: SSD for faster I/O, NVMe for even better performance
- **Network**: 10Gbps+ for high-throughput systems

### Denormalization Strategies

**What is Denormalization**:
- Intentionally adding redundancy to database
- Trade normalization for read performance
- Reduces JOINs (which are expensive)

**When to Denormalize**:
- Read-heavy workloads
- JOINs are performance bottleneck
- Can tolerate some data inconsistency
- Writes are infrequent

**Denormalization Patterns**:

1. **Duplicate Data**:
   - Store same data in multiple tables
   - Example: Store username in posts table (not just user_id)
   - Reduces JOINs but increases storage

2. **Precomputed Aggregates**:
   - Store computed values (counts, sums, averages)
   - Example: `like_count` in posts table
   - Update on write, fast reads

3. **Materialized Views**:
   - Pre-computed query results
   - Refreshed periodically or on-demand
   - Example: Daily sales summary table

4. **Embedded Documents** (NoSQL):
   - Store related data together
   - Example: User document with embedded posts
   - Reduces queries but harder to update

**Trade-offs**:
- **Pros**: Faster reads, fewer queries, simpler queries
- **Cons**: More storage, data duplication, consistency challenges, slower writes

**Example**:
```sql
-- Normalized (requires JOIN)
SELECT p.*, u.username, u.avatar_url 
FROM posts p 
JOIN users u ON p.user_id = u.id 
WHERE p.id = 123;

-- Denormalized (no JOIN needed)
SELECT * FROM posts WHERE id = 123;
-- username and avatar_url already in posts table
```

## Scale Your Database Horizontally

### Read Replicas

**What are Read Replicas**:
- Copies of primary database that handle read operations
- Primary handles all writes
- Replicas handle reads
- Data replicated asynchronously (usually)

**Architecture**:
```
Primary (Writes) → Replication → Replica 1 (Reads)
                  → Replication → Replica 2 (Reads)
                  → Replication → Replica 3 (Reads)
```

**Benefits**:
- **Offload Reads**: Primary database only handles writes
- **Horizontal Scaling**: Add more replicas to scale reads
- **Fault Tolerance**: Replicas can become primary if primary fails
- **Geographic Distribution**: Replicas in different regions reduce latency

**Replication Strategies**:

1. **Asynchronous Replication** (Most Common):
   - Primary writes, replicas catch up later
   - Low latency for writes
   - Possible read lag (eventual consistency)
   - Use when: Some staleness acceptable

2. **Synchronous Replication**:
   - Primary waits for replica confirmation
   - Strong consistency
   - Higher write latency
   - Use when: Consistency critical

3. **Semi-Synchronous Replication**:
   - Primary waits for at least one replica
   - Balance of consistency and performance
   - Better than async, faster than sync

**Read Replica Configuration**:
- Typically 2-5 read replicas
- More replicas = more read capacity
- But more replication lag and complexity
- Balance based on read load

**Load Balancing Reads**:
- Round-robin across replicas
- Weighted routing (prefer closer replicas)
- Health checks (route away from unhealthy replicas)
- Consistent hashing (route same user to same replica)

**Replication Lag Considerations**:
- Replicas may be slightly behind primary
- Users may see stale data
- Acceptable for most use cases
- Critical systems may need synchronous replication

**Example Use Cases**:
- Social media feeds (some staleness OK)
- Product catalogs (updates infrequent)
- Analytics queries (don't need latest data)
- Reporting systems

### Database Sharding

**What is Sharding**:
- Horizontal partitioning of data across multiple databases
- Each shard contains subset of data
- Distributes both reads and writes
- Most complex scaling strategy

**Sharding Strategies**:

1. **Range-Based Sharding**:
   - Partition by range (e.g., user_id 1-1000 in shard 1)
   - Simple to implement
   - Can have hot shards (uneven distribution)

2. **Hash-Based Sharding**:
   - Partition by hash of key
   - Even distribution
   - Hard to query across shards

3. **Directory-Based Sharding**:
   - Lookup service maps keys to shards
   - Flexible, can rebalance
   - Single point of failure (lookup service)

4. **Geographic Sharding**:
   - Partition by geographic region
   - Low latency (data close to users)
   - Natural distribution

**Sharding Challenges**:
- **Cross-Shard Queries**: Expensive, may require application-level joins
- **Rebalancing**: Moving data between shards is complex
- **Transactions**: Hard across shards (need distributed transactions)
- **Hot Shards**: Uneven load distribution

**When to Shard**:
- Single database can't handle load
- Read replicas not enough
- Need to scale writes too
- Large dataset (billions of rows)

**Sharding Example**:
```
Shard 1: Users 1-1M
Shard 2: Users 1M-2M
Shard 3: Users 2M-3M

Query for user 1.5M → Route to Shard 2
```

## Add External Caching Layers

### Application-Level Caching

**What is Application-Level Caching**:
- Cache frequently accessed data in memory
- Reduces database queries
- Much faster than database (memory vs disk)

**Caching Layers**:

1. **In-Memory Cache** (Redis, Memcached):
   - Fastest (memory access)
   - Limited by RAM
   - Volatile (data lost on restart)
   - Use for: Hot data, frequently accessed

2. **Application Cache**:
   - Cache in application memory
   - Very fast (no network)
   - Limited by application memory
   - Use for: Small, very hot data

3. **Distributed Cache**:
   - Shared cache across multiple servers
   - Consistent view across instances
   - Network overhead
   - Use for: Shared data across servers

**Caching Strategies**:

1. **Cache-Aside (Lazy Loading)**:
   ```
   Read:
   1. Check cache
   2. If miss, read from DB
   3. Write to cache
   4. Return data
   
   Write:
   1. Write to DB
   2. Invalidate cache (or update cache)
   ```
   - Simple to implement
   - Cache misses cause DB queries
   - Use when: Data access patterns unpredictable

2. **Write-Through**:
   ```
   Write:
   1. Write to cache
   2. Write to DB
   3. Return
   
   Read:
   1. Read from cache (always hit)
   ```
   - Cache always consistent
   - Every write hits DB
   - Use when: Consistency critical

3. **Write-Behind (Write-Back)**:
   ```
   Write:
   1. Write to cache
   2. Return immediately
   3. Async write to DB
   ```
   - Fastest writes
   - Risk of data loss
   - Use when: Can tolerate some data loss

4. **Refresh-Ahead**:
   - Proactively refresh cache before expiration
   - Reduces cache misses
   - More complex
   - Use when: Predictable access patterns

**Cache Invalidation Strategies**:

1. **Time-Based (TTL)**:
   - Cache expires after time
   - Simple
   - May serve stale data
   - Use for: Data that changes infrequently

2. **Event-Based**:
   - Invalidate on data change
   - Always fresh
   - More complex
   - Use for: Data that must be current

3. **Version-Based**:
   - Cache with version number
   - Check version on read
   - Balance of freshness and performance
   - Use for: Frequently changing data

**Cache Patterns**:

**Pattern 1: Single Key Caching**:
```python
def get_user(user_id):
    cache_key = f"user:{user_id}"
    user = cache.get(cache_key)
    if user is None:
        user = db.get_user(user_id)
        cache.set(cache_key, user, ttl=3600)
    return user
```

**Pattern 2: Multi-Key Caching**:
```python
def get_user_posts(user_id):
    cache_key = f"user:{user_id}:posts"
    posts = cache.get(cache_key)
    if posts is None:
        posts = db.get_user_posts(user_id)
        cache.set(cache_key, posts, ttl=300)
    return posts
```

**Pattern 3: Cache Warming**:
- Pre-populate cache with hot data
- Reduces cold start cache misses
- Use for: Predictable hot data

### CDN and Edge Caching

**What is CDN**:
- Content Delivery Network
- Caches content at edge locations (close to users)
- Originally for static assets, now supports dynamic content

**CDN Benefits**:
- **Low Latency**: Content served from nearby edge
- **Reduced Origin Load**: Fewer requests to origin server
- **Geographic Distribution**: Global edge network
- **DDoS Protection**: Absorbs traffic spikes

**CDN Caching Strategies**:

1. **Static Asset Caching**:
   - Images, CSS, JavaScript
   - Long TTL (days/weeks)
   - Cache busting via versioning

2. **Dynamic Content Caching**:
   - API responses
   - Database query results
   - Shorter TTL (minutes/hours)
   - Cache invalidation on updates

3. **Edge Computing**:
   - Run code at edge
   - Customize responses per request
   - Example: Personalize content at edge

**CDN Configuration**:
- Cache headers (Cache-Control, ETag)
- TTL settings
- Cache invalidation (purge API)
- Origin shielding (reduce origin requests)

**When to Use CDN**:
- Global user base
- Static assets
- API responses that can be cached
- High read-to-write ratio

## Common Deep Dives

### "What happens when your queries start taking longer as your dataset grows?"

**The Problem**:
- Query performance degrades as data grows
- Indexes become less effective
- Full table scans become expensive

**Solutions**:

1. **Optimize Queries**:
   - Add appropriate indexes
   - Rewrite inefficient queries
   - Use query hints
   - Analyze query plans

2. **Partition Large Tables**:
   - Partition by date, range, or hash
   - Smaller partitions = faster queries
   - Example: Partition by year for time-series data

3. **Archive Old Data**:
   - Move old data to cold storage
   - Keep only hot data in primary DB
   - Query archive separately if needed

4. **Materialized Views**:
   - Pre-compute expensive queries
   - Refresh periodically
   - Fast reads, slower updates

5. **Read Replicas**:
   - Offload queries to replicas
   - Keep primary for writes
   - Scale reads horizontally

### "How do you handle millions of concurrent reads for the same cached data?"

**The Problem**:
- Popular content (viral post, trending product)
- Millions of requests for same data
- Cache can become bottleneck

**Solutions**:

1. **Multi-Level Caching**:
   ```
   Application Cache → Distributed Cache → Database
   ```
   - Application cache: Fastest, smallest
   - Distributed cache: Shared, larger
   - Database: Source of truth

2. **Cache Replication**:
   - Multiple cache instances
   - Replicate popular data
   - Distribute load

3. **CDN Caching**:
   - Cache at edge locations
   - Serve from nearest edge
   - Reduces origin load

4. **Cache Sharding**:
   - Partition cache by key
   - Distribute across cache servers
   - Better memory utilization

5. **Preemptive Caching**:
   - Cache popular content before it's requested
   - Predict hot data
   - Reduce cache misses

**Example: Viral Post**:
```
1. Post goes viral
2. Preemptively cache in all regions
3. Serve from CDN edge locations
4. Reduce database load by 99%
```

### "What happens when multiple requests try to rebuild an expired cache entry simultaneously?"

**The Problem: Cache Stampede (Thundering Herd)**:
- Cache entry expires
- Multiple requests see cache miss
- All requests query database simultaneously
- Database overwhelmed

**Solutions**:

1. **Lock-Based Approach**:
   ```python
   def get_data(key):
       data = cache.get(key)
       if data is None:
           lock = acquire_lock(key)
           if lock:
               try:
                   # Double-check after acquiring lock
                   data = cache.get(key)
                   if data is None:
                       data = db.get_data(key)
                       cache.set(key, data)
               finally:
                   release_lock(lock)
           else:
               # Another process is loading, wait and retry
               time.sleep(0.1)
               return get_data(key)
       return data
   ```
   - Only one request loads data
   - Others wait for result
   - Prevents stampede

2. **Probabilistic Early Expiration**:
   - Expire cache slightly before TTL
   - Refresh in background
   - Reduces simultaneous expirations

3. **Background Refresh**:
   - Refresh cache before expiration
   - Always serve from cache
   - No cache misses

4. **Semaphore Pattern**:
   - Limit concurrent database queries
   - Queue excess requests
   - Prevents overwhelming database

5. **Stale-While-Revalidate**:
   - Serve stale data while refreshing
   - Update cache in background
   - No user-visible delay

### "How do you handle cache invalidation when data updates need to be immediately visible?"

**The Problem**:
- Data updated in database
- Cache still has old data
- Users see stale data
- Need immediate consistency

**Solutions**:

1. **Write-Through Cache**:
   - Update cache on write
   - Cache always consistent
   - Every write updates cache

2. **Cache Invalidation on Write**:
   ```python
   def update_user(user_id, data):
       db.update_user(user_id, data)
       cache.delete(f"user:{user_id}")
       # Or update cache
       cache.set(f"user:{user_id}", updated_data)
   ```
   - Invalidate or update on write
   - Next read fetches fresh data

3. **Event-Driven Invalidation**:
   - Publish event on data change
   - Cache service listens to events
   - Invalidates affected cache entries
   - Works across distributed systems

4. **Version-Based Caching**:
   ```python
   def get_user(user_id):
       cached = cache.get(f"user:{user_id}")
       db_version = db.get_user_version(user_id)
       if cached and cached['version'] == db_version:
           return cached['data']
       else:
           data = db.get_user(user_id)
           cache.set(f"user:{user_id}", {
               'data': data,
               'version': db_version
           })
           return data
   ```
   - Cache includes version
   - Check version on read
   - Refresh if version mismatch

5. **TTL-Based with Short Expiration**:
   - Short TTL (seconds/minutes)
   - Acceptable staleness
   - Simpler than invalidation

**Trade-offs**:
- **Immediate Invalidation**: More complex, always fresh
- **TTL-Based**: Simpler, some staleness
- **Version-Based**: Balance of complexity and freshness

## When to Use in Interviews

### Common Interview Scenarios

**Use Scaling Reads Pattern When**:
- **Social Media Feeds**: Instagram, Twitter, Facebook
- **Content Platforms**: YouTube, Medium, Reddit
- **E-commerce**: Product catalogs, search results
- **News Aggregators**: Article feeds, trending topics
- **Analytics Dashboards**: Reporting, metrics
- **Search Systems**: Search results, autocomplete
- **Gaming**: Leaderboards, player stats
- **Streaming**: Video metadata, recommendations

### When NOT to Use

**Don't Over-Optimize When**:
- Read load is low (premature optimization)
- Writes are more critical than reads
- Data must be always consistent (may need different approach)
- System is write-heavy (different scaling strategy needed)

## Implementation Considerations

### Consistency vs Performance Trade-offs

**Strong Consistency**:
- Read from primary or synchronous replicas
- Always see latest data
- Slower (higher latency)
- Use for: Financial data, critical operations

**Eventual Consistency**:
- Read from async replicas or cache
- May see slightly stale data
- Faster (lower latency)
- Use for: Social feeds, product catalogs

### Monitoring and Metrics

**Key Metrics to Monitor**:
- **Read Latency**: P50, P95, P99 latencies
- **Cache Hit Rate**: Percentage of requests served from cache
- **Database Load**: CPU, memory, I/O utilization
- **Replication Lag**: How far behind replicas are
- **Query Performance**: Slow query logs, query execution time

**Alerting**:
- High read latency
- Low cache hit rate
- High database load
- Replication lag exceeding threshold
- Cache stampede detection

### Cost Considerations

**Caching Costs**:
- Memory costs (RAM for cache)
- Network costs (cache replication)
- CDN costs (edge caching)

**Replica Costs**:
- Additional database instances
- Storage costs
- Replication bandwidth

**Optimization**:
- Right-size cache (not too large, not too small)
- Choose appropriate TTL (balance freshness and hit rate)
- Monitor cache efficiency

## Best Practices

### Design Principles

1. **Start Simple**: Optimize database first, then scale
2. **Measure First**: Profile before optimizing
3. **Cache Strategically**: Cache hot data, not everything
4. **Monitor Everything**: Track metrics, identify bottlenecks
5. **Plan for Failure**: Cache failures shouldn't break system

### Implementation Patterns

**Pattern 1: Cache-Aside with Lock**:
```python
def get_with_cache(key, loader_func, ttl=3600):
    # Try cache first
    data = cache.get(key)
    if data:
        return data
    
    # Acquire lock to prevent stampede
    lock_key = f"lock:{key}"
    if acquire_lock(lock_key, timeout=1):
        try:
            # Double-check cache
            data = cache.get(key)
            if data:
                return data
            
            # Load from source
            data = loader_func()
            cache.set(key, data, ttl=ttl)
            return data
        finally:
            release_lock(lock_key)
    else:
        # Another process loading, wait and retry
        time.sleep(0.1)
        return get_with_cache(key, loader_func, ttl)
```

**Pattern 2: Multi-Level Cache**:
```python
def get_user_multi_level(user_id):
    # Level 1: Application cache
    user = app_cache.get(f"user:{user_id}")
    if user:
        return user
    
    # Level 2: Distributed cache
    user = redis.get(f"user:{user_id}")
    if user:
        app_cache.set(f"user:{user_id}", user, ttl=60)
        return user
    
    # Level 3: Database
    user = db.get_user(user_id)
    redis.set(f"user:{user_id}", user, ttl=3600)
    app_cache.set(f"user:{user_id}", user, ttl=60)
    return user
```

**Pattern 3: Cache Warming**:
```python
def warm_cache():
    # Pre-load hot data
    hot_user_ids = get_hot_user_ids()
    for user_id in hot_user_ids:
        user = db.get_user(user_id)
        cache.set(f"user:{user_id}", user, ttl=3600)
```

## Summary

Scaling reads is a fundamental challenge in system design:

**Progressive Strategy**:
1. Optimize within database (indexing, hardware, denormalization)
2. Scale horizontally (read replicas, sharding)
3. Add caching layers (application cache, CDN)

**Key Techniques**:
- **Indexing**: Fast lookups, trade-off with write performance
- **Read Replicas**: Horizontal read scaling, eventual consistency
- **Caching**: In-memory speed, invalidation complexity
- **CDN**: Global distribution, edge caching

**Trade-offs**:
- Consistency vs Performance
- Complexity vs Scalability
- Cost vs Performance

**Key Takeaways for Staff-Level Engineers**:
- Understand read-to-write ratios in your system
- Start with simple optimizations before complex solutions
- Design for cache stampede and invalidation
- Monitor metrics to identify bottlenecks
- Balance consistency requirements with performance needs
- Consider cost implications of scaling strategies

Understanding these patterns is crucial for designing systems that can handle millions of users and billions of reads.

</details>
