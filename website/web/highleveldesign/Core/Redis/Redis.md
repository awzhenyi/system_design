# Redis

- Single Threaded
- key value store
- accept commands like SET, GET, INCR, XADD

Scaling Reads

Scaling Writes
- Each node takes up different range of slots. eg main1(0 - 100), main2(0 - 200), ... Each main knows the existence of each other and which slots each of them have.

- Sharding based on the keys is the only way to scale redis.
- Hot key problem, one of the node experiences uneven distribution of traffic. How to solve? append random number to key to distribute

-keyspace is key to scaling redis.

Common Usage
1. Cache. 
- check cache before query db. when key evicts / expires then persist. Expiration policy -> ttl 60s, LRU
- different strategies (read through, write through, etc)
   a. Read-through pattern
      - Application requests data from cache
      - Cache automatically loads from database on miss
      - Cache updates itself and returns value
      Benefits:
      - Simpler application logic
      - Consistent caching strategy
      - Single source of cache updates
      Drawbacks:
      - Cache must handle database connectivity
      - All reads pay cache overhead
      - May cache infrequently used data

   b. Write-through pattern
      - Application writes to cache
      - Cache synchronously writes to database
      - Write confirmed when both complete
      Benefits:
      - Data consistency guaranteed
      - Good for write-heavy workloads
      Drawbacks:
      - Added write latency
      - Cache stores unnecessary data
      - More complex implementation

   c. Write-behind pattern
      - Application writes to cache
      - Cache asynchronously writes to database
      - Write confirmed when cache updated
      Benefits:
      - Lower write latency
      - Write batching possible
      - Buffer against DB failures
      Drawbacks:
      - Risk of data loss
      - Complex failure handling
      - Eventual consistency only
      
- cache invalidation flow
   a. Basic Flow:
      - Application updates database
      - Application invalidates cache entry
      - Subsequent reads fetch fresh data

   b. Preventing Dirty Reads:
      - Use write-through pattern
         * Write to DB first
         * Then invalidate cache
         * New reads get fresh data
      
      - Double-check pattern
         * Read from cache
         * If found, verify version/timestamp
         * If stale, fetch from DB
         * Update cache with fresh data

      - Lock-based approach
         * Acquire lock before update
         * Update DB and invalidate cache
         * Release lock
         * Prevents concurrent access

      - Versioning strategy
         * Each cache entry has version
         * DB update increments version
         * Cache checks version before serving
         * Ensures consistency

   c. Best Practices:
      - Use atomic operations when possible
      - Implement retry logic for failures
      - Log invalidation events for debugging
      - Monitor cache hit/miss rates
      - Consider bulk invalidation patterns

- cache invalidation strategies
   a. Time-based invalidation (TTL)
      - Cache entries expire after set time
      - Simple to implement and understand
      - Good for relatively static data
      - May keep stale data until expiry
      - No guarantee of consistency

   b. Event-based invalidation
      - Cache invalidated when data changes
      - Application explicitly removes/updates cache
      - Better consistency than time-based
      - More complex to implement
      - Risk of race conditions

   c. Version-based invalidation
      - Each cache entry has version number
      - Version incremented on data changes
      - Cache checked against source version
      - Good consistency guarantees
      - Higher overhead from version checks

   d. Pattern-based invalidation
      - Invalidate groups of related cache entries
      - Uses key patterns/prefixes
      - Good for hierarchical data
      - Can lead to over-invalidation
      - More complex pattern management

   e. Database trigger-based invalidation
      - Database triggers detect data changes
      - Triggers automatically invalidate cache
      - Benefits:
         * Immediate cache invalidation
         * No application code changes needed
         * Consistent across all data changes
      - Drawbacks:
         * Additional database overhead
         * Complex trigger management
         * Potential performance impact
         * Tight coupling between DB and cache
      - Implementation considerations:
         * Trigger granularity (row vs table)
         * Handle trigger failures gracefully
         * Monitor trigger performance
         * Consider async invalidation patterns




2. rate limiter. use atomic command INCR to check how many request have been made to expensive service. Once request is made, expires in 60s / the calculated time range.

- redis primitive STREAM. ordered list of items, id = timestamp of insertion. Each item can have key-value

- async job queue. Job/Items that needs to be processed goes into the stream. There is a consumer group that points to the next item to be processed in stream, and allocates it to workers. The worker has claim to this item. If the worker fails, the consumer group can reclaim the item and allocate it to another worker. Worker give heartbeat to consumer group to let it know it is still alive so consumer group do not reclaim back the job. Consumer group offers at least once gaurantee. Why? Imagine worker lose connection but still processing, consumer group reclaims and allocate to worker 2.

- sorted set, heap
- geospatial data


3. Distributed lock

4. Geospatial
