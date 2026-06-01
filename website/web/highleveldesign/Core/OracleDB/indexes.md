# Oracle Database Indexes

## Introduction of Indexes and the Need for It

Indexes are database objects that improve the speed of data retrieval operations. They work similarly to a book's index, allowing the database to find data without scanning the entire table.

### Why We Need Indexes

- **Performance**: Without indexes, queries require full table scans (O(n) complexity)
- **Data Integrity**: Unique indexes ensure data uniqueness
- **Query Optimization**: Enable efficient sorting and joining operations
- **Primary Key Enforcement**: Automatically created for primary keys
- **Foreign Key Performance**: Improve referential integrity checks

### Understanding EXPLAIN PLAN

Oracle uses `EXPLAIN PLAN` to show how the database will execute a query. This is crucial for understanding query performance and index usage.

#### Basic EXPLAIN PLAN Output

```sql
EXPLAIN PLAN FOR
SELECT * FROM users WHERE email = 'user@example.com';

SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY);

-- Example output:
PLAN_TABLE_OUTPUT
--------------------------------------------------------------------------------
Plan hash value: 1234567890

--------------------------------------------------------------------------------
| Id | Operation                    | Name          | Rows | Bytes | Cost |
--------------------------------------------------------------------------------
|  0 | SELECT STATEMENT             |               |    1 |    72 |     3 |
|  1 |  TABLE ACCESS BY INDEX ROWID | USERS         |    1 |    72 |     3 |
|  2 |   INDEX UNIQUE SCAN          | IDX_USERS_EMAIL | 1 |       |     2 |
--------------------------------------------------------------------------------
```

This output shows:
- The query uses an index unique scan
- Estimated cost: 3
- Expected to return 1 row
- Row width is 72 bytes

#### EXPLAIN PLAN with Statistics

```sql
EXPLAIN PLAN FOR
SELECT u.name, o.order_date 
FROM users u 
JOIN orders o ON u.id = o.user_id 
WHERE u.email = 'user@example.com';

SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY);

-- Example output:
PLAN_TABLE_OUTPUT
--------------------------------------------------------------------------------
Plan hash value: 2345678901

--------------------------------------------------------------------------------
| Id | Operation                    | Name            | Rows | Bytes | Cost |
--------------------------------------------------------------------------------
|  0 | SELECT STATEMENT             |                 |    1 |    50 |     5 |
|  1 |  NESTED LOOPS                |                 |    1 |    50 |     5 |
|  2 |   TABLE ACCESS BY INDEX ROWID| USERS           |    1 |    30 |     3 |
|  3 |    INDEX UNIQUE SCAN          | IDX_USERS_EMAIL |    1 |       |     2 |
|  4 |   TABLE ACCESS BY INDEX ROWID | ORDERS          |    1 |    20 |     2 |
|  5 |    INDEX RANGE SCAN            | IDX_ORDERS_USER_ID | 1 |       |     1 |
--------------------------------------------------------------------------------
```

This output shows:
- Execution plan with join operations
- Index usage for both tables
- Cost estimates for each operation
- Join method (Nested Loops)

#### Common Plan Types and Their Meanings

1. **Full Table Scan (FTS)**:
```sql
EXPLAIN PLAN FOR
SELECT * FROM users WHERE last_login < DATE '2023-01-01';

SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY);

-- Example output:
--------------------------------------------------------------------------------
| Id | Operation         | Name  | Rows | Bytes | Cost |
--------------------------------------------------------------------------------
|  0 | SELECT STATEMENT  |       | 1000 | 72000 |  450 |
|  1 |  TABLE ACCESS FULL| USERS | 1000 | 72000 |  450 |
--------------------------------------------------------------------------------
```
This indicates:
- Full table scan (inefficient for large tables)
- No index is being used
- Consider adding an index if this query is frequent

2. **Index Range Scan**:
```sql
EXPLAIN PLAN FOR
SELECT * FROM users WHERE email = 'user@example.com';

SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY);

-- Example output:
--------------------------------------------------------------------------------
| Id | Operation                    | Name          | Rows | Bytes | Cost |
--------------------------------------------------------------------------------
|  0 | SELECT STATEMENT             |               |    1 |    72 |     3 |
|  1 |  TABLE ACCESS BY INDEX ROWID | USERS         |    1 |    72 |     3 |
|  2 |   INDEX RANGE SCAN           | IDX_USERS_EMAIL | 1 |       |     2 |
--------------------------------------------------------------------------------
```
This indicates:
- Efficient index usage
- Direct lookup using the index
- Good performance for equality conditions

3. **Index Unique Scan**:
```sql
EXPLAIN PLAN FOR
SELECT * FROM users WHERE id = 123;

SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY);

-- Example output:
--------------------------------------------------------------------------------
| Id | Operation                    | Name          | Rows | Bytes | Cost |
--------------------------------------------------------------------------------
|  0 | SELECT STATEMENT             |               |    1 |    72 |     2 |
|  1 |  TABLE ACCESS BY INDEX ROWID | USERS         |    1 |    72 |     2 |
|  2 |   INDEX UNIQUE SCAN          | PK_USERS      |    1 |       |     1 |
--------------------------------------------------------------------------------
```
This indicates:
- Unique index scan (fastest)
- Used for primary key or unique constraint lookups
- Very efficient (O(log n))

4. **Hash Join**:
```sql
EXPLAIN PLAN FOR
SELECT u.name, o.order_date 
FROM users u 
JOIN orders o ON u.id = o.user_id;

SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY);

-- Example output:
--------------------------------------------------------------------------------
| Id | Operation          | Name  | Rows | Bytes | Cost |
--------------------------------------------------------------------------------
|  0 | SELECT STATEMENT   |       | 1000 | 50000 |   50 |
|  1 |  HASH JOIN         |       | 1000 | 50000 |   50 |
|  2 |   TABLE ACCESS FULL| USERS |  100 |  3000 |   10 |
|  3 |   TABLE ACCESS FULL| ORDERS| 1000 | 20000 |   40 |
--------------------------------------------------------------------------------
```
This indicates:
- Hash join operation
- Both tables scanned (may indicate missing indexes)
- Good for large result sets

#### Key Metrics to Watch

1. **Cost**:
   - Estimated resource cost
   - Lower is better
   - Relative measure (not absolute time)

2. **Rows**:
   - Estimated number of rows
   - Compare with actual rows (use `AUTOTRACE` or `V$SQL`)
   - Large differences indicate statistics issues

3. **Bytes**:
   - Estimated data size
   - Helps understand memory usage

4. **Cardinality**:
   - Number of distinct values
   - Important for optimizer decisions
   - Affects join method selection

#### Common Performance Issues

1. **Missing Indexes**:
```sql
EXPLAIN PLAN FOR
SELECT * FROM users WHERE last_name LIKE 'Smith%';

SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY);
-- Shows: TABLE ACCESS FULL
```
Solution: Add an index for frequently searched columns

2. **Inefficient Joins**:
```sql
EXPLAIN PLAN FOR
SELECT * FROM users u 
JOIN orders o ON u.id = o.user_id 
WHERE o.total_amount > 1000;

SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY);
-- Shows: HASH JOIN with full table scans
```
Solution: Add appropriate indexes on join columns

3. **Suboptimal Index Usage**:
```sql
EXPLAIN PLAN FOR
SELECT * FROM users 
WHERE UPPER(email) = 'USER@EXAMPLE.COM' 
AND created_at > DATE '2023-01-01';

SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY);
-- May not use index on email due to function
```
Solution: Consider function-based index on `UPPER(email)`

## What Type of Indexes and How They Speed Up Queries

### 1. B-tree Index (Default)

**Most Common Index Type**:
```sql
-- Create a B-tree index
CREATE INDEX idx_users_email ON users(email);

-- Unique B-tree index
CREATE UNIQUE INDEX idx_users_email_unique ON users(email);
```

**Best For**:
- Equality comparisons (=)
- Range queries (`>, <`, BETWEEN)
- Sorting (ORDER BY)
- Primary keys and unique constraints
- High cardinality columns (many distinct values)
- OLTP environments

**Characteristics**:
- Balanced tree structure
- O(log n) lookup time
- Supports range scans
- Default index type in Oracle

**Performance**:
- Very fast for equality searches
- Efficient for range queries
- Good for sorting operations

### 2. Bitmap Index

**For Low Cardinality Columns**:
```sql
-- Create a bitmap index
CREATE BITMAP INDEX idx_users_status ON users(status);

-- Composite bitmap index
CREATE BITMAP INDEX idx_orders_status_region 
ON orders(status, region);
```

**Best For**:
- Low cardinality columns (few distinct values)
- Data warehousing environments
- Columns with values like: status, gender, region
- Queries with multiple AND/OR conditions
- Star schema fact tables

**Characteristics**:
- Uses bitmap (bit array) for each distinct value
- Efficient combination using bitwise operations
- Very compact for low cardinality
- Not suitable for high DML activity

**Performance**:
- Excellent for data warehousing queries
- Fast combination of multiple conditions
- Very space-efficient for low cardinality

**Limitations**:
- High lock contention on updates
- Not suitable for OLTP with frequent updates
- Best for read-heavy, update-light workloads

### 3. Function-Based Index

**Index on Expression**:
```sql
-- Create a function-based index
CREATE INDEX idx_users_upper_email ON users(UPPER(email));

-- Index on expression
CREATE INDEX idx_products_price_tax 
ON products(price * 1.2);

-- Index on date function
CREATE INDEX idx_orders_year 
ON orders(EXTRACT(YEAR FROM order_date));
```

**Best For**:
- Case-insensitive searches
- Computed columns
- Date/time functions
- Mathematical expressions
- String functions

**Characteristics**:
- Stores result of function/expression
- Query must match expression exactly
- Requires query rewrite to use index

**Example Usage**:
```sql
-- Index created on UPPER(email)
CREATE INDEX idx_users_upper_email ON users(UPPER(email));

-- This query can use the index:
SELECT * FROM users WHERE UPPER(email) = 'USER@EXAMPLE.COM';

-- This query CANNOT use the index:
SELECT * FROM users WHERE email = 'user@example.com';
```

### 4. Composite Index (Multi-Column)

**Index on Multiple Columns**:
```sql
-- Create a composite index
CREATE INDEX idx_users_name_email 
ON users(last_name, first_name, email);
```

**Best For**:
- Queries filtering on multiple columns
- Queries with ORDER BY on multiple columns
- Covering indexes (all columns in query)

**Column Order Importance**:
```sql
-- Good: Most selective column first
CREATE INDEX idx_users_status_created 
ON users(status, created_at);

-- Bad: Less selective column first
CREATE INDEX idx_users_created_status 
ON users(created_at, status);

-- Example queries that can use the good index:
SELECT * FROM users 
WHERE status = 'active' 
AND created_at > DATE '2023-01-01';

SELECT * FROM users WHERE status = 'active';
-- But this query cannot use the index effectively:
SELECT * FROM users WHERE created_at > DATE '2023-01-01';
```

**Leftmost Prefix Rule**:
- Index can be used for queries on:
  1. First column only
  2. First and second columns
  3. All columns
- Cannot skip columns (left-to-right)

### 5. Reverse Key Index

**For Sequential Key Distribution**:
```sql
-- Create a reverse key index
CREATE INDEX idx_orders_id_reverse 
ON orders(order_id) REVERSE;
```

**Best For**:
- Sequential primary keys (sequences)
- Reduces hot block contention in RAC
- Improves insert performance in high-concurrency

**Characteristics**:
- Reverses bytes of key before indexing
- Distributes inserts across index blocks
- Reduces right-hand side index growth
- Trade-off: Cannot use for range scans

### 6. Compressed Index

**Reduce Index Size**:
```sql
-- Create a compressed index
CREATE INDEX idx_users_name_compressed 
ON users(last_name, first_name) COMPRESS 2;
```

**Best For**:
- Composite indexes with repeated values
- Large indexes with storage constraints
- Columns with low cardinality in composite index

**Benefits**:
- Reduced storage space
- Faster index scans (less I/O)
- Better cache utilization

**Trade-offs**:
- Slightly higher CPU for compression/decompression
- Compression overhead on inserts

### 7. Invisible Index

**Test Index Without Dropping**:
```sql
-- Create an invisible index
CREATE INDEX idx_users_test ON users(email) INVISIBLE;

-- Make index visible
ALTER INDEX idx_users_test VISIBLE;

-- Make index invisible
ALTER INDEX idx_users_test INVISIBLE;
```

**Use Cases**:
- Test index impact without dropping
- Gradual index migration
- A/B testing index strategies

## EXPLAIN PLAN Analysis

### Using DBMS_XPLAN

**Basic Display**:
```sql
EXPLAIN PLAN FOR
SELECT * FROM users WHERE email = 'user@example.com';

SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY);
```

**Display with Statistics**:
```sql
EXPLAIN PLAN FOR
SELECT * FROM users WHERE email = 'user@example.com';

SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY('PLAN_TABLE', NULL, 'ALL'));
```

**Display with I/O Statistics**:
```sql
-- Enable statistics collection
ALTER SESSION SET STATISTICS_LEVEL = ALL;

-- Execute query
SELECT * FROM users WHERE email = 'user@example.com';

-- Display plan with statistics
SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY_CURSOR(NULL, NULL, 'ALLSTATS LAST'));
```

### Understanding Execution Plans

**Access Methods**:

1. **TABLE ACCESS FULL**:
   - Full table scan
   - Reads all blocks
   - High cost for large tables
   - Consider adding index

2. **TABLE ACCESS BY INDEX ROWID**:
   - Uses index to find rowids
   - Then fetches rows from table
   - Efficient for selective queries

3. **INDEX UNIQUE SCAN**:
   - Unique index lookup
   - Fastest access method
   - O(log n) complexity

4. **INDEX RANGE SCAN**:
   - Range scan on index
   - Good for range queries
   - Efficient for sorted data

5. **INDEX FULL SCAN**:
   - Scans entire index
   - Used when index has all needed columns
   - Faster than table scan if index smaller

**Join Methods**:

1. **NESTED LOOPS**:
   - Good for small result sets
   - One table drives, other probed
   - Efficient with indexes

2. **HASH JOIN**:
   - Good for large result sets
   - Builds hash table on smaller table
   - Probes with larger table

3. **SORT MERGE JOIN**:
   - Sorts both tables
   - Merges sorted results
   - Good when data pre-sorted

## Index Maintenance

### Monitoring Index Usage

**View Index Statistics**:
```sql
SELECT 
    index_name,
    table_name,
    num_rows,
    distinct_keys,
    clustering_factor,
    last_analyzed
FROM user_indexes
WHERE table_name = 'USERS';
```

**Check Index Usage**:
```sql
SELECT 
    object_name,
    object_type,
    value
FROM v$segment_statistics
WHERE object_name LIKE 'IDX_%'
AND statistic_name = 'logical reads';
```

**Identify Unused Indexes**:
```sql
SELECT 
    i.index_name,
    i.table_name,
    s.num_rows,
    s.distinct_keys
FROM user_indexes i
LEFT JOIN user_ind_statistics s 
    ON i.index_name = s.index_name
WHERE i.index_name NOT IN (
    SELECT index_name 
    FROM v$object_usage 
    WHERE used = 'YES'
);
```

### Index Rebuilding

**Rebuild Index**:
```sql
-- Rebuild index to reduce fragmentation
ALTER INDEX idx_users_email REBUILD;

-- Rebuild online (no table lock)
ALTER INDEX idx_users_email REBUILD ONLINE;

-- Rebuild with compression
ALTER INDEX idx_users_name REBUILD COMPRESS 2;
```

**When to Rebuild**:
- High fragmentation
- After bulk deletes
- To change storage parameters
- To enable/disable compression

**Coalesce Index**:
```sql
-- Coalesce index (less disruptive than rebuild)
ALTER INDEX idx_users_email COALESCE;
```

### Index Statistics

**Gather Statistics**:
```sql
-- Gather statistics for index
EXEC DBMS_STATS.GATHER_INDEX_STATS('SCHEMA', 'IDX_USERS_EMAIL');

-- Gather statistics for all indexes on table
EXEC DBMS_STATS.GATHER_TABLE_STATS('SCHEMA', 'USERS', CASCADE => TRUE);
```

**Why Statistics Matter**:
- Optimizer uses statistics for cost estimation
- Outdated statistics lead to poor plans
- Regular statistics collection is critical

## Best Practices

### Index Design

1. **Index Primary Keys**:
   - Automatically created
   - Essential for referential integrity

2. **Index Foreign Keys**:
   - Improve join performance
   - Speed up referential integrity checks
   - Often overlooked but important

3. **Index Frequently Queried Columns**:
   - Columns in WHERE clauses
   - Columns in JOIN conditions
   - Columns in ORDER BY

4. **Consider Composite Indexes**:
   - For multi-column queries
   - Match query patterns
   - Consider column order

5. **Avoid Over-Indexing**:
   - Each index slows writes
   - Monitor index usage
   - Remove unused indexes

### Performance Optimization

1. **Use Appropriate Index Type**:
   - B-tree for high cardinality
   - Bitmap for low cardinality (warehouse)
   - Function-based for expressions

2. **Monitor Index Usage**:
   - Regular statistics collection
   - Check execution plans
   - Identify unused indexes

3. **Maintain Indexes**:
   - Rebuild when fragmented
   - Update statistics regularly
   - Monitor index growth

4. **Consider Partitioning**:
   - Partition large tables
   - Local indexes on partitions
   - Global indexes when needed

## Common Use Cases

### 1. Primary Keys

```sql
-- Automatically creates a B-tree index
CREATE TABLE users (
    id NUMBER PRIMARY KEY,
    email VARCHAR2(255)
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

### 5. Case-Insensitive Search

```sql
-- Function-based index for case-insensitive search
CREATE INDEX idx_users_upper_email ON users(UPPER(email));

-- Query must match:
SELECT * FROM users WHERE UPPER(email) = 'USER@EXAMPLE.COM';
```

## Pros and Cons

### Pros

1. **Query Performance**:
   - Faster data retrieval
   - Reduced I/O operations
   - Optimized join operations

2. **Data Integrity**:
   - Enforce uniqueness constraints
   - Maintain referential integrity
   - Prevent duplicate entries

3. **Sorting Optimization**:
   - Faster ORDER BY operations
   - Efficient GROUP BY processing
   - Improved DISTINCT operations

4. **Join Performance**:
   - Faster table joins
   - Better query planning
   - Reduced memory usage

### Cons

1. **Storage Overhead**:
   - Additional disk space required
   - Increased database size
   - More I/O operations for writes

2. **Write Performance**:
   - Slower INSERT operations (index maintenance)
   - Slower UPDATE operations (if indexed columns updated)
   - Slower DELETE operations (index cleanup)

3. **Maintenance Overhead**:
   - Regular index maintenance required
   - Statistics collection needed
   - Index fragmentation

4. **Query Planner Complexity**:
   - More complex query planning
   - Potential for suboptimal plans
   - Additional memory usage

## Summary

Oracle Database provides multiple index types optimized for different scenarios:

- **B-tree Indexes**: Default, high cardinality, OLTP
- **Bitmap Indexes**: Low cardinality, data warehousing
- **Function-Based Indexes**: Expressions, case-insensitive search
- **Composite Indexes**: Multi-column queries
- **Reverse Key Indexes**: Sequential keys, RAC environments
- **Compressed Indexes**: Storage optimization

**Key Takeaways**:
- Choose index type based on data characteristics
- Monitor index usage and remove unused indexes
- Maintain statistics for optimal query plans
- Balance read performance with write overhead
- Use EXPLAIN PLAN to analyze query performance

Understanding Oracle indexes is crucial for designing high-performance database systems.
