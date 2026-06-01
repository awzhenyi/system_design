# Indexes

## Introduction of indexes and the need for it

Indexes are database objects that improve the speed of data retrieval operations. They work similarly to a book's index, allowing the database to find data without scanning the entire table.

### Why We Need Indexes
- **Performance**: Without indexes, queries require full table scans (O(n) complexity)
- **Data Integrity**: Unique indexes ensure data uniqueness
- **Query Optimization**: Enable efficient sorting and joining operations

### Understanding EXPLAIN and EXPLAIN ANALYZE

#### Basic EXPLAIN Output
```sql
EXPLAIN SELECT * FROM users WHERE email = 'user@example.com';

-- Example output:
QUERY PLAN
----------------------------------------------------------
Index Scan using idx_users_email on users  (cost=0.42..8.44 rows=1 width=72)
  Index Cond: (email = 'user@example.com'::text)
```
This output shows:
- The query uses an index scan
- Estimated cost: 0.42 to 8.44
- Expected to return 1 row
- Row width is 72 bytes

#### EXPLAIN ANALYZE Output
```sql
EXPLAIN ANALYZE 
SELECT u.name, o.order_date 
FROM users u 
JOIN orders o ON u.id = o.user_id 
WHERE u.email = 'user@example.com';

-- Example output:
QUERY PLAN
------------------------------------------------------------------------------------------------
Nested Loop  (cost=0.42..16.47 rows=1 width=72) (actual time=0.023..0.024 rows=1 loops=1)
  ->  Index Scan using idx_users_email on users u  (cost=0.42..8.44 rows=1 width=36) (actual time=0.015..0.016 rows=1 loops=1)
        Index Cond: (email = 'user@example.com'::text)
  ->  Index Scan using idx_orders_user_id on orders o  (cost=0.00..8.02 rows=1 width=36) (actual time=0.006..0.006 rows=1 loops=1)
        Index Cond: (user_id = u.id)
Planning Time: 0.123 ms
Execution Time: 0.045 ms
```
This output shows:
- Actual execution time for each step
- Number of rows processed
- Planning and execution times
- Index usage details

#### Common Plan Types and Their Meanings

1. **Sequential Scan**
```sql
EXPLAIN SELECT * FROM users WHERE last_login < '2023-01-01';

-- Example output:
QUERY PLAN
----------------------------------------------------------
Seq Scan on users  (cost=0.00..1450.00 rows=1000 width=72)
  Filter: (last_login < '2023-01-01'::date)
```
This indicates:
- Full table scan (inefficient for large tables)
- No index is being used
- Consider adding an index if this query is frequent

2. **Index Scan**
```sql
EXPLAIN SELECT * FROM users WHERE email = 'user@example.com';

-- Example output:
QUERY PLAN
----------------------------------------------------------
Index Scan using idx_users_email on users  (cost=0.42..8.44 rows=1 width=72)
  Index Cond: (email = 'user@example.com'::text)
```
This indicates:
- Efficient index usage
- Direct lookup using the index
- Good performance for equality conditions

3. **Bitmap Heap Scan**
```sql
EXPLAIN SELECT * FROM users WHERE age > 30 AND status = 'active';

-- Example output:
QUERY PLAN
----------------------------------------------------------
Bitmap Heap Scan on users  (cost=4.36..25.17 rows=10 width=72)
  Recheck Cond: ((age > 30) AND (status = 'active'::text))
  ->  Bitmap Index Scan on idx_users_age_status  (cost=0.00..4.36 rows=10 width=0)
        Index Cond: ((age > 30) AND (status = 'active'::text))
```
This indicates:
- Multiple conditions being checked
- Index is used to find matching rows
- Additional heap scan to fetch actual data

4. **Nested Loop**
```sql
EXPLAIN SELECT u.name, o.order_date 
FROM users u 
JOIN orders o ON u.id = o.user_id;

-- Example output:
QUERY PLAN
----------------------------------------------------------
Nested Loop  (cost=0.42..16.47 rows=1 width=72)
  ->  Index Scan using idx_users_email on users u  (cost=0.42..8.44 rows=1 width=36)
  ->  Index Scan using idx_orders_user_id on orders o  (cost=0.00..8.02 rows=1 width=36)
        Index Cond: (user_id = u.id)
```
This indicates:
- Join operation using indexes
- Good for small result sets
- Can be inefficient for large datasets

#### Key Metrics to Watch

1. **Cost**
- First number: Startup cost
- Second number: Total cost
- Lower is better

2. **Rows**
- Estimated number of rows
- Compare with actual rows in EXPLAIN ANALYZE
- Large differences indicate statistics issues

3. **Width**
- Estimated row size in bytes
- Helps understand memory usage

4. **Actual Time**
- Real execution time in milliseconds
- Only available in EXPLAIN ANALYZE
- More accurate than cost estimates

#### Common Performance Issues

1. **Missing Indexes**
```sql
EXPLAIN SELECT * FROM users WHERE last_name LIKE 'Smith%';

-- Example output:
QUERY PLAN
----------------------------------------------------------
Seq Scan on users  (cost=0.00..1450.00 rows=1000 width=72)
  Filter: (last_name ~~ 'Smith%'::text)
```
Solution: Add an index for frequently searched columns

2. **Inefficient Joins**
```sql
EXPLAIN SELECT * FROM users u 
JOIN orders o ON u.id = o.user_id 
WHERE o.total_amount > 1000;

-- Example output:
QUERY PLAN
----------------------------------------------------------
Hash Join  (cost=1000.00..2000.00 rows=1000 width=144)
  Hash Cond: (o.user_id = u.id)
  ->  Seq Scan on orders o  (cost=0.00..1000.00 rows=1000 width=72)
        Filter: (total_amount > 1000)
  ->  Hash  (cost=500.00..500.00 rows=1000 width=72)
        ->  Seq Scan on users u  (cost=0.00..500.00 rows=1000 width=72)
```
Solution: Add appropriate indexes on join columns

3. **Suboptimal Index Usage**
```sql
EXPLAIN SELECT * FROM users 
WHERE email LIKE '%@gmail.com' 
AND created_at > '2023-01-01';

-- Example output:
QUERY PLAN
----------------------------------------------------------
Bitmap Heap Scan on users  (cost=100.00..200.00 rows=1000 width=72)
  Recheck Cond: ((email ~~ '%@gmail.com'::text) AND (created_at > '2023-01-01'::date))
  ->  Bitmap Index Scan on idx_users_created_at  (cost=0.00..100.00 rows=1000 width=0)
        Index Cond: (created_at > '2023-01-01'::date)
```
Solution: Consider partial indexes or expression indexes

## What type of indexes and how they speed up query

### 1. B-tree Index (Default)
```sql
-- Create a B-tree index
CREATE INDEX idx_users_email ON users(email);

-- Best for:
-- - Equality comparisons (=)
-- - Range queries (>, <, BETWEEN)
-- - Sorting (ORDER BY)
-- - Unique constraints
```

### 2. Hash Index
```sql
-- Create a hash index
CREATE INDEX idx_users_email_hash ON users USING hash(email);

-- Best for:
-- - Simple equality comparisons
-- - No support for range queries
-- - Smaller than B-tree for large values
```

### 3. GiST (Generalized Search Tree)
```sql
-- Create a GiST index
CREATE INDEX idx_geometries ON geometries USING gist(geom);

-- Best for:
-- - Geometric data
-- - Full-text search
-- - Network address types
```

### 4. GIN (Generalized Inverted Index)
```sql
-- Create a GIN index
CREATE INDEX idx_documents_content ON documents USING gin(to_tsvector('english', content));

-- Best for:
-- - Full-text search
-- - Array operations
-- - JSON/JSONB data
```

### 5. BRIN (Block Range INdexes)
```sql
-- Create a BRIN index
CREATE INDEX idx_orders_date ON orders USING brin(order_date);

-- Best for:
-- - Large tables
-- - Naturally ordered data
-- - Range queries on timestamp/date columns
```
### 6. Composite Index
Composite indexes (also known as multi-column indexes) are indexes that include multiple columns. They are useful when queries frequently filter or sort by multiple columns.

#### Basic Usage
```sql
-- Create a composite index on multiple columns
CREATE INDEX idx_users_name_email ON users(last_name, first_name, email);

-- The index can be used for:
-- 1. Queries on last_name only
-- 2. Queries on last_name AND first_name
-- 3. Queries on last_name AND first_name AND email
```

#### Column Order Importance
```sql
-- Good: Most selective column first
CREATE INDEX idx_users_status_created ON users(status, created_at);

-- Bad: Less selective column first
CREATE INDEX idx_users_created_status ON users(created_at, status);

-- Example queries that can use the good index:
SELECT * FROM users WHERE status = 'active' AND created_at > '2023-01-01';
SELECT * FROM users WHERE status = 'active';
-- But this query cannot use the index effectively:
SELECT * FROM users WHERE created_at > '2023-01-01';
```

#### Common Use Cases

1. **Name Searches**
```sql
-- Index for name searches
CREATE INDEX idx_users_name ON users(last_name, first_name);

-- These queries can use the index:
SELECT * FROM users WHERE last_name = 'Smith';
SELECT * FROM users WHERE last_name = 'Smith' AND first_name = 'John';
```

2. **Date Range with Status**
```sql
-- Index for date range queries with status filter
CREATE INDEX idx_orders_status_date ON orders(status, order_date);

-- These queries can use the index:
SELECT * FROM orders WHERE status = 'pending' AND order_date > '2023-01-01';
SELECT * FROM orders WHERE status = 'pending';
```

3. **Composite Primary Keys**
```sql
-- Index for composite primary key
CREATE TABLE order_items (
    order_id INTEGER,
    item_id INTEGER,
    quantity INTEGER,
    PRIMARY KEY (order_id, item_id)
);
```

#### Best Practices

1. **Column Order**
   - Put the most selective column first
   - Consider query patterns
   - Think about equality vs. range conditions

2. **Index Size**
   - More columns = larger index
   - Consider memory usage
   - Balance between coverage and size

3. **Query Patterns**
   - Match index to common query patterns
   - Consider left-to-right column usage
   - Think about sorting requirements

#### Examples with Different Scenarios

1. **Equality and Range**
```sql
-- Index for equality and range conditions
CREATE INDEX idx_products_category_price ON products(category, price);

-- These queries can use the index:
SELECT * FROM products WHERE category = 'Electronics' AND price < 1000;
SELECT * FROM products WHERE category = 'Electronics';
```

2. **Sorting with Filtering**
```sql
-- Index for filtered sorting
CREATE INDEX idx_orders_customer_date ON orders(customer_id, order_date DESC);

-- These queries can use the index:
SELECT * FROM orders 
WHERE customer_id = 123 
ORDER BY order_date DESC;
```

3. **Partial Composite Index**
```sql
-- Index for specific conditions
CREATE INDEX idx_active_users_name ON users(last_name, first_name)
WHERE status = 'active';

-- This query can use the index:
SELECT * FROM users 
WHERE status = 'active' 
AND last_name = 'Smith' 
AND first_name = 'John';
```

#### Performance Considerations

1. **Index Size**
```sql
-- Check index size
SELECT pg_size_pretty(pg_relation_size('idx_users_name_email')) as index_size;

-- Compare with table size
SELECT pg_size_pretty(pg_relation_size('users')) as table_size;
```

2. **Index Usage**
```sql
-- Check if index is being used
EXPLAIN ANALYZE 
SELECT * FROM users 
WHERE last_name = 'Smith' 
AND first_name = 'John';
```

3. **Maintenance**
```sql
-- Rebuild index if needed
REINDEX INDEX idx_users_name_email;

-- Check index fragmentation
SELECT schemaname, tablename, indexname, 
       pg_size_pretty(pg_relation_size(schemaname||'.'||indexname::text)) as index_size,
       idx_scan as number_of_scans
FROM pg_stat_user_indexes;
```

#### Common Mistakes to Avoid

1. **Wrong Column Order**
```sql
-- Bad: Less selective column first
CREATE INDEX idx_users_created_status ON users(created_at, status);

-- Good: More selective column first
CREATE INDEX idx_users_status_created ON users(status, created_at);
```

2. **Too Many Columns**
```sql
-- Bad: Too many columns
CREATE INDEX idx_users_all ON users(last_name, first_name, email, phone, address, city, state, zip);

-- Good: Focus on most used columns
CREATE INDEX idx_users_name_email ON users(last_name, first_name, email);
```

3. **Ignoring Query Patterns**
```sql
-- Bad: Index doesn't match query pattern
CREATE INDEX idx_users_email_name ON users(email, last_name);

-- Good: Index matches common query pattern
CREATE INDEX idx_users_name_email ON users(last_name, email);
```

## Pros

1. **Query Performance**
   - Faster data retrieval
   - Reduced I/O operations
   - Optimized join operations

2. **Data Integrity**
   - Enforce uniqueness constraints
   - Maintain referential integrity
   - Prevent duplicate entries

3. **Sorting Optimization**
   - Faster ORDER BY operations
   - Efficient GROUP BY processing
   - Improved DISTINCT operations

4. **Join Performance**
   - Faster table joins
   - Better query planning
   - Reduced memory usage

## Cons

1. **Storage Overhead**
   - Additional disk space required
   - Increased database size
   - More I/O operations for writes

2. **Write Performance**
   - Slower INSERT operations
   - Slower UPDATE operations
   - Slower DELETE operations

3. **Maintenance Overhead**
   - Regular index maintenance required
   - Vacuum operations needed
   - Index fragmentation

4. **Query Planner Complexity**
   - More complex query planning
   - Potential for suboptimal plans
   - Additional memory usage

## Common use cases

### 1. Primary Keys
```sql
-- Automatically creates a B-tree index
CREATE TABLE users (
    id SERIAL PRIMARY KEY,
    email VARCHAR(255)
);
```

### 2. Foreign Keys
```sql
-- Index on foreign key for better join performance
CREATE INDEX idx_orders_user_id ON orders(user_id);
```

### 3. Frequently Queried Columns
```sql
-- Index on commonly searched columns
CREATE INDEX idx_users_email ON users(email);
CREATE INDEX idx_products_category ON products(category);
```

### 4. Sorting Operations
```sql
-- Index for efficient sorting
CREATE INDEX idx_orders_date ON orders(order_date);
```

### 5. Full-Text Search
```sql
-- GIN index for full-text search
CREATE INDEX idx_documents_content ON documents 
USING gin(to_tsvector('english', content));
```

## Niche use cases

### 1. Partial Indexes
```sql
-- Index only active users
CREATE INDEX idx_active_users ON users(email) 
WHERE status = 'active';

-- Index only recent orders
CREATE INDEX idx_recent_orders ON orders(order_date) 
WHERE order_date > CURRENT_DATE - INTERVAL '1 year';
```

### 2. Expression Indexes
```sql
-- Index on function result
CREATE INDEX idx_users_lower_email ON users(lower(email));

-- Index on computed column
CREATE INDEX idx_products_price_tax ON products(price * 1.2);
```

### 3. Multi-column Indexes
```sql
-- Composite index for specific query patterns
CREATE INDEX idx_orders_user_date ON orders(user_id, order_date);

-- Index for specific sort order
CREATE INDEX idx_products_category_price ON products(category, price DESC);
```

### 4. Covering Indexes
```sql
-- Include additional columns in index
CREATE INDEX idx_orders_covering ON orders(order_date) 
INCLUDE (user_id, total_amount);
```

### 5. Specialized Indexes
```sql
-- GiST index for geometric data
CREATE INDEX idx_geometries ON geometries USING gist(geom);

-- BRIN index for time-series data
CREATE INDEX idx_metrics_time ON metrics USING brin(timestamp);
```

### 6. Concurrent Index Creation
```sql
-- Create index without blocking writes
CREATE INDEX CONCURRENTLY idx_users_email ON users(email);
```

### 7. Indexes on JSON/JSONB
```sql
-- GIN index for JSON data
CREATE INDEX idx_users_metadata ON users USING gin(metadata);

-- B-tree index on JSON field
CREATE INDEX idx_users_metadata_name ON users((metadata->>'name'));
```

### 8. Indexes for Array Operations
```sql
-- GIN index for array operations
CREATE INDEX idx_products_tags ON products USING gin(tags);

-- Index for array containment
CREATE INDEX idx_products_tags_btree ON products USING btree(tags);
```

## Index Fragmentation

Index fragmentation occurs when the physical order of index entries doesn't match their logical order, leading to inefficient disk usage and potentially degraded query performance.

### What Causes Fragmentation

1. **Frequent Updates**
   - When index entries are frequently updated
   - When rows are deleted and new ones are inserted
   - When tables are heavily modified

2. **Vacuum Issues**
   - Insufficient VACUUM operations
   - Long-running transactions preventing cleanup
   - High update/delete activity

3. **Page Splits**
   - When new entries are inserted into full index pages
   - When updates cause index entries to grow
   - When B-tree nodes need to be rebalanced

### Detecting Fragmentation

1. **Using pg_stat_user_indexes**
```sql
-- Check index usage and size
SELECT 
    schemaname,
    tablename,
    indexname,
    pg_size_pretty(pg_relation_size(schemaname||'.'||indexname::text)) as index_size,
    idx_scan as number_of_scans,
    idx_tup_read as tuples_read,
    idx_tup_fetch as tuples_fetched
FROM pg_stat_user_indexes
WHERE schemaname = 'public'
ORDER BY pg_relation_size(schemaname||'.'||indexname::text) DESC;
```

2. **Using pg_stat_all_tables**
```sql
-- Check table statistics
SELECT 
    schemaname,
    relname,
    n_live_tup,
    n_dead_tup,
    last_vacuum,
    last_autovacuum
FROM pg_stat_all_tables
WHERE schemaname = 'public';
```

3. **Using pgstattuple Extension**
```sql
-- Install extension if not exists
CREATE EXTENSION IF NOT EXISTS pgstattuple;

-- Analyze index fragmentation
SELECT * FROM pgstattuple('idx_users_name_email');
```

### Impact of Fragmentation

1. **Performance Issues**
   - Increased I/O operations
   - Slower index scans
   - Higher memory usage

2. **Space Usage**
   - Wasted disk space
   - Inefficient page utilization
   - Increased storage requirements

3. **Query Performance**
   - Slower index lookups
   - Increased buffer cache usage
   - Higher CPU utilization

### Managing Fragmentation

1. **Regular VACUUM**
```sql
-- Regular VACUUM
VACUUM ANALYZE users;

-- Full VACUUM (locks table)
VACUUM FULL users;
```

2. **Index Rebuilding**
```sql
-- Rebuild specific index
REINDEX INDEX idx_users_name_email;

-- Rebuild all indexes in a table
REINDEX TABLE users;

-- Rebuild all indexes in a database
REINDEX DATABASE mydatabase;
```

3. **Concurrent Reindexing**
```sql
-- Rebuild index without blocking writes
REINDEX INDEX CONCURRENTLY idx_users_name_email;
```

### Best Practices

1. **Regular Maintenance**
```sql
-- Schedule regular VACUUM
VACUUM ANALYZE users;

-- Monitor fragmentation
SELECT * FROM pg_stat_user_indexes 
WHERE indexrelname = 'idx_users_name_email';
```

2. **Automated Maintenance**
```sql
-- Set autovacuum parameters
ALTER TABLE users SET (
    autovacuum_vacuum_scale_factor = 0.1,
    autovacuum_analyze_scale_factor = 0.05
);
```

3. **Monitoring**
```sql
-- Create a function to check fragmentation
CREATE OR REPLACE FUNCTION check_index_fragmentation()
RETURNS TABLE (
    index_name text,
    fragmentation_ratio numeric
) AS $$
BEGIN
    RETURN QUERY
    SELECT 
        i.indexrelname::text,
        (pg_relation_size(i.indexrelid)::numeric / 
         (pg_relation_size(i.indexrelid) + 
          pg_relation_size(i.relid))::numeric * 100)::numeric
    FROM pg_stat_user_indexes i
    JOIN pg_indexes idx ON i.indexrelname = idx.indexname
    WHERE idx.schemaname = 'public';
END;
$$ LANGUAGE plpgsql;
```

### Prevention Strategies

1. **Proper Index Design**
   - Choose appropriate index types
   - Consider update patterns
   - Use partial indexes when appropriate

2. **Regular Maintenance Schedule**
   - Set up automated VACUUM
   - Monitor index growth
   - Schedule regular REINDEX

3. **Monitoring and Alerts**
   - Track fragmentation levels
   - Set up alerts for high fragmentation
   - Monitor index usage patterns

### Recovery from High Fragmentation

1. **Immediate Actions**
```sql
-- For critical indexes
REINDEX INDEX CONCURRENTLY idx_users_name_email;

-- For non-critical indexes
REINDEX INDEX idx_users_name_email;
```

2. **Long-term Solutions**
```sql
-- Adjust autovacuum settings
ALTER TABLE users SET (
    autovacuum_vacuum_scale_factor = 0.1,
    autovacuum_analyze_scale_factor = 0.05,
    autovacuum_vacuum_threshold = 50,
    autovacuum_analyze_threshold = 50
);
```

3. **Preventive Measures**
```sql
-- Create maintenance function
CREATE OR REPLACE FUNCTION maintain_indexes()
RETURNS void AS $$
DECLARE
    idx record;
BEGIN
    FOR idx IN 
        SELECT indexrelname 
        FROM pg_stat_user_indexes 
        WHERE schemaname = 'public'
    LOOP
        EXECUTE 'REINDEX INDEX CONCURRENTLY ' || idx.indexrelname;
    END LOOP;
END;
$$ LANGUAGE plpgsql;
```