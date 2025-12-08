# Cache

## Write Back (Write-Behind)

- **How it works:**  
  The application writes data to the cache. The cache then asynchronously writes the data to the database after a delay or in batches.
- **Benefits:**  
  - Low write latency for the application  
  - Allows write batching and buffering  
  - Can absorb database outages temporarily
- **Drawbacks:**  
  - Risk of data loss if cache fails before writing to DB  
  - Eventual consistency (data in DB may lag behind cache)
- **Use Cases:**  
  - High write throughput scenarios  
  - Analytics, logging, or non-critical data

**Example Flow:**
1. Application writes to cache.
2. Cache immediately acknowledges write.
3. Cache writes to database in the background.

---

## Write Through

- **How it works:**  
  The application writes data to the cache, and the cache synchronously writes the data to the database before acknowledging the write.
- **Benefits:**  
  - Strong consistency between cache and database  
  - Simple to reason about data state
- **Drawbacks:**  
  - Higher write latency (must wait for DB)  
  - Cache may store rarely used data
- **Use Cases:**  
  - Scenarios requiring strong consistency  
  - User profiles, financial data

**Example Flow:**
1. Application writes to cache.
2. Cache writes to database.
3. Write is acknowledged only after both succeed.

---

## Look Aside (Cache-Aside)

- **How it works:**  
  The application checks the cache before querying the database. On a cache miss, it loads data from the database and updates the cache.
- **Benefits:**  
  - Only frequently accessed data is cached  
  - Simple and flexible pattern
- **Drawbacks:**  
  - Application must handle cache logic  
  - Risk of stale data if not properly invalidated
- **Use Cases:**  
  - Read-heavy workloads  
  - Product catalogs, session data

**Example Flow:**
1. Application reads from cache.
2. If cache miss, read from database and update cache.
3. On data update, application updates both database and invalidates cache.