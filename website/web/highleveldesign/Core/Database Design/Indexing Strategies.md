# Indexing Strategies

## Overview

Indexes are data structures that improve the speed of data retrieval operations on a database table. They work like an index in a book, allowing the database to quickly locate data without scanning the entire table. However, indexes come with trade-offs: they improve read performance but can slow down writes and consume additional storage.

## How Indexes Work

### Without Index (Full Table Scan)

```sql
SELECT * FROM users WHERE email = 'user@example.com';
```

**Process:**
1. Database scans every row in the table
2. Compares email value for each row
3. Returns matching rows
4. **Time Complexity**: O(n) - linear scan

### With Index (Index Seek)

```sql
CREATE INDEX idx_users_email ON users(email);
SELECT * FROM users WHERE email = 'user@example.com';
```

**Process:**
1. Database uses index to find matching rows
2. Directly accesses those rows
3. Returns results
4. **Time Complexity**: O(log n) - tree traversal

## Types of Indexes

### 1. B-Tree Index (Most Common)

**Structure:**
- Balanced tree structure
- Supports range queries
- Efficient for equality and range searches

**Use Cases:**
- Primary keys (automatically indexed)
- Foreign keys
- Columns used in WHERE clauses
- Columns used in ORDER BY
- Columns used in JOIN conditions

**Example:**
```sql
CREATE INDEX idx_orders_user_id ON orders(user_id);
CREATE INDEX idx_products_price ON products(price);
```

### 2. Hash Index

**Structure:**
- Hash table structure
- Very fast for equality lookups
- Does not support range queries

**Use Cases:**
- Exact match queries only
- High-frequency equality searches
- Not suitable for ORDER BY or range queries

**Example:**
```sql
-- PostgreSQL
CREATE INDEX idx_users_email_hash ON users USING HASH(email);
```

### 3. Bitmap Index

**Structure:**
- Bitmap for each distinct value
- Efficient for low-cardinality columns
- Good for data warehousing

**Use Cases:**
- Columns with few distinct values (e.g., status, category)
- Read-heavy analytical workloads
- Data warehousing scenarios

**Example:**
```sql
-- Oracle
CREATE BITMAP INDEX idx_orders_status ON orders(status);
```

### 4. Composite Index (Multi-Column Index)

**Structure:**
- Index on multiple columns
- Order of columns matters
- Supports queries on prefix columns

**Use Cases:**
- Queries filtering on multiple columns
- Queries with WHERE and ORDER BY
- Covering indexes (index-only scans)

**Example:**
```sql
CREATE INDEX idx_orders_user_date ON orders(user_id, order_date);
-- Efficient for:
-- WHERE user_id = ? AND order_date = ?
-- WHERE user_id = ? ORDER BY order_date
-- WHERE user_id = ? (can use prefix)
```

**Column Order Matters:**
```sql
-- Good: user_id has higher selectivity
CREATE INDEX idx_orders_user_date ON orders(user_id, order_date);

-- Less optimal: order_date has lower selectivity
CREATE INDEX idx_orders_date_user ON orders(order_date, user_id);
```

### 5. Partial Index

**Structure:**
- Index on subset of rows
- Smaller index size
- Faster for filtered queries

**Use Cases:**
- Queries that filter on specific conditions
- Reducing index size
- Improving performance for common filters

**Example:**
```sql
-- Index only active users
CREATE INDEX idx_users_active_email ON users(email) 
WHERE is_active = TRUE;

-- Index only recent orders
CREATE INDEX idx_orders_recent ON orders(order_date) 
WHERE order_date > '2024-01-01';
```

### 6. Covering Index (Index-Only Scan)

**Structure:**
- Index contains all columns needed for query
- Avoids table lookup
- Very fast queries

**Use Cases:**
- Queries that only need indexed columns
- Read-heavy workloads
- Reducing I/O operations

**Example:**
```sql
-- Query only needs user_id and email
SELECT user_id, email FROM users WHERE email = 'user@example.com';

-- Covering index
CREATE INDEX idx_users_email_covering ON users(email, user_id);
-- Database can answer query from index alone
```

## Index Design Principles

### 1. Index Selectivity

**Selectivity** = Number of distinct values / Total number of rows

**High Selectivity** (close to 1.0):
- Many distinct values
- Good candidate for indexing
- Example: email, user_id, order_id

**Low Selectivity** (close to 0):
- Few distinct values
- May not benefit from index
- Example: status (active/inactive), gender

**Rule of Thumb:**
- Index high-selectivity columns
- Consider partial indexes for low-selectivity columns
- Avoid indexing very low-selectivity columns

### 2. Query Patterns

Analyze your query patterns:

**Frequently Filtered Columns:**
```sql
-- If this query is common:
SELECT * FROM orders WHERE user_id = ? AND status = 'pending';
-- Create composite index:
CREATE INDEX idx_orders_user_status ON orders(user_id, status);
```

**Frequently Sorted Columns:**
```sql
-- If this query is common:
SELECT * FROM orders WHERE user_id = ? ORDER BY order_date DESC;
-- Create index:
CREATE INDEX idx_orders_user_date ON orders(user_id, order_date DESC);
```

**Join Columns:**
```sql
-- Always index foreign keys
CREATE INDEX idx_orders_user_id ON orders(user_id);
CREATE INDEX idx_order_items_order_id ON order_items(order_id);
```

### 3. Write vs Read Trade-off

**Indexes Improve:**
- SELECT queries
- JOIN operations
- ORDER BY operations
- WHERE clause filtering

**Indexes Degrade:**
- INSERT operations (must update index)
- UPDATE operations (must update index if indexed column changes)
- DELETE operations (must update index)
- Storage space

**Guidelines:**
- Read-heavy tables: More indexes beneficial
- Write-heavy tables: Fewer indexes better
- Balance based on workload

## Common Indexing Patterns

### Pattern 1: Primary Key Index

Automatically created, always index primary keys:

```sql
CREATE TABLE users (
    user_id INT PRIMARY KEY,  -- Automatically indexed
    email VARCHAR(255)
);
```

### Pattern 2: Foreign Key Index

Always index foreign keys for JOIN performance:

```sql
CREATE TABLE orders (
    order_id INT PRIMARY KEY,
    user_id INT,
    FOREIGN KEY (user_id) REFERENCES users(user_id)
);

-- Always create index on foreign key
CREATE INDEX idx_orders_user_id ON orders(user_id);
```

### Pattern 3: Unique Constraint Index

Automatically created for UNIQUE constraints:

```sql
CREATE TABLE users (
    user_id INT PRIMARY KEY,
    email VARCHAR(255) UNIQUE  -- Automatically indexed
);
```

### Pattern 4: Composite Index for Common Queries

```sql
-- Common query pattern:
SELECT * FROM orders 
WHERE user_id = ? AND status = ? 
ORDER BY order_date DESC 
LIMIT 10;

-- Optimal composite index:
CREATE INDEX idx_orders_user_status_date 
ON orders(user_id, status, order_date DESC);
```

### Pattern 5: Covering Index for Read-Only Queries

```sql
-- Query only needs specific columns:
SELECT user_id, name, email FROM users WHERE email = ?;

-- Covering index:
CREATE INDEX idx_users_email_covering 
ON users(email, user_id, name);
```

## Index Maintenance

### Monitoring Index Usage

**Check Index Usage:**
```sql
-- PostgreSQL
SELECT 
    schemaname,
    tablename,
    indexname,
    idx_scan as index_scans,
    idx_tup_read as tuples_read,
    idx_tup_fetch as tuples_fetched
FROM pg_stat_user_indexes
ORDER BY idx_scan DESC;
```

**Identify Unused Indexes:**
- Indexes with low or zero scans
- Consider dropping if not used
- Monitor before dropping

### Index Fragmentation

**Problem:**
- Indexes become fragmented over time
- Degrades performance
- Requires maintenance

**Solution:**
```sql
-- PostgreSQL: REINDEX
REINDEX INDEX idx_users_email;

-- MySQL: OPTIMIZE TABLE
OPTIMIZE TABLE users;

-- Oracle: REBUILD INDEX
ALTER INDEX idx_users_email REBUILD;
```

### Index Statistics

Keep statistics updated for query optimizer:

```sql
-- PostgreSQL
ANALYZE users;

-- MySQL
ANALYZE TABLE users;

-- Oracle (automatic, but can manually update)
EXEC DBMS_STATS.GATHER_TABLE_STATS('SCHEMA', 'USERS');
```

## Indexing Best Practices

### 1. Index Foreign Keys

Always index foreign keys for JOIN performance:

```sql
CREATE INDEX idx_orders_user_id ON orders(user_id);
CREATE INDEX idx_order_items_order_id ON order_items(order_id);
CREATE INDEX idx_order_items_product_id ON order_items(product_id);
```

### 2. Index Frequently Queried Columns

Columns used in WHERE, JOIN, ORDER BY:

```sql
-- If frequently queried:
SELECT * FROM users WHERE email = ?;
CREATE INDEX idx_users_email ON users(email);

-- If frequently sorted:
SELECT * FROM orders ORDER BY order_date DESC;
CREATE INDEX idx_orders_date ON orders(order_date DESC);
```

### 3. Use Composite Indexes Wisely

Order columns by selectivity and query patterns:

```sql
-- Good: High selectivity first
CREATE INDEX idx_orders_user_date ON orders(user_id, order_date);

-- Consider query patterns:
-- WHERE user_id = ? AND order_date = ?  -- Both columns
-- WHERE user_id = ?                      -- Prefix usable
-- WHERE order_date = ?                   -- Prefix NOT usable
```

### 4. Avoid Over-Indexing

Too many indexes:
- Slow down writes
- Consume storage
- Increase maintenance overhead

**Guidelines:**
- Index based on actual query patterns
- Monitor index usage
- Remove unused indexes

### 5. Consider Partial Indexes

For filtered queries:

```sql
-- Only index active users
CREATE INDEX idx_users_active_email ON users(email) 
WHERE is_active = TRUE;
```

### 6. Use Covering Indexes When Possible

For queries that only need indexed columns:

```sql
-- Covering index
CREATE INDEX idx_orders_user_date_covering 
ON orders(user_id, order_date, total_amount);

-- Query can use index-only scan
SELECT user_id, order_date, total_amount 
FROM orders 
WHERE user_id = ?;
```

## Common Mistakes

### 1. Indexing Every Column

**Problem:**
```sql
-- Bad: Indexing everything
CREATE INDEX idx_users_id ON users(user_id);
CREATE INDEX idx_users_name ON users(name);
CREATE INDEX idx_users_email ON users(email);
CREATE INDEX idx_users_created_at ON users(created_at);
CREATE INDEX idx_users_updated_at ON users(updated_at);
-- ... and so on
```

**Solution:**
- Index based on query patterns
- Monitor index usage
- Remove unused indexes

### 2. Wrong Column Order in Composite Index

**Problem:**
```sql
-- Bad: Low selectivity first
CREATE INDEX idx_orders_status_user ON orders(status, user_id);
-- status has only few values (pending, completed, cancelled)
```

**Solution:**
```sql
-- Good: High selectivity first
CREATE INDEX idx_orders_user_status ON orders(user_id, status);
```

### 3. Not Indexing Foreign Keys

**Problem:**
```sql
CREATE TABLE orders (
    order_id INT PRIMARY KEY,
    user_id INT,
    FOREIGN KEY (user_id) REFERENCES users(user_id)
);
-- Missing index on user_id
```

**Solution:**
```sql
CREATE INDEX idx_orders_user_id ON orders(user_id);
```

### 4. Ignoring Query Patterns

**Problem:**
- Creating indexes without analyzing queries
- Not considering ORDER BY, GROUP BY
- Missing composite indexes for multi-column queries

**Solution:**
- Analyze query logs
- Use EXPLAIN to understand query plans
- Create indexes based on actual usage

## Indexing for System Design Interviews

### Example 1: Social Media Feed

**Requirements:**
- Display user's feed (posts from followed users)
- Order by timestamp (newest first)
- Pagination support

**Schema:**
```sql
CREATE TABLE posts (
    post_id INT PRIMARY KEY,
    user_id INT,
    content TEXT,
    created_at TIMESTAMP
);

CREATE TABLE follows (
    follower_id INT,
    followee_id INT,
    PRIMARY KEY (follower_id, followee_id)
);
```

**Query:**
```sql
SELECT p.* 
FROM posts p
JOIN follows f ON p.user_id = f.followee_id
WHERE f.follower_id = ?
ORDER BY p.created_at DESC
LIMIT 20 OFFSET ?;
```

**Indexes:**
```sql
-- Index foreign keys
CREATE INDEX idx_posts_user_id ON posts(user_id);
CREATE INDEX idx_follows_follower ON follows(follower_id);
CREATE INDEX idx_follows_followee ON follows(followee_id);

-- Composite index for feed query
CREATE INDEX idx_posts_user_created ON posts(user_id, created_at DESC);
```

### Example 2: E-commerce Product Search

**Requirements:**
- Search products by name
- Filter by category
- Sort by price or popularity
- Pagination

**Schema:**
```sql
CREATE TABLE products (
    product_id INT PRIMARY KEY,
    name VARCHAR(255),
    category_id INT,
    price DECIMAL(10,2),
    popularity_score INT
);
```

**Queries:**
```sql
-- Search by name
SELECT * FROM products WHERE name LIKE '%keyword%';

-- Filter by category, sort by price
SELECT * FROM products 
WHERE category_id = ? 
ORDER BY price ASC 
LIMIT 20;

-- Filter by category, sort by popularity
SELECT * FROM products 
WHERE category_id = ? 
ORDER BY popularity_score DESC 
LIMIT 20;
```

**Indexes:**
```sql
-- Full-text search index (for name search)
CREATE FULLTEXT INDEX idx_products_name_ft ON products(name);

-- Composite indexes for filtered + sorted queries
CREATE INDEX idx_products_category_price ON products(category_id, price);
CREATE INDEX idx_products_category_popularity ON products(category_id, popularity_score DESC);
```

### Example 3: Leaderboard System

**Requirements:**
- Get top N users by score
- Get user's rank
- Get users around a specific user

**Schema:**
```sql
CREATE TABLE user_scores (
    user_id INT PRIMARY KEY,
    score INT,
    updated_at TIMESTAMP
);
```

**Queries:**
```sql
-- Top N users
SELECT * FROM user_scores 
ORDER BY score DESC 
LIMIT 10;

-- User's rank
SELECT COUNT(*) + 1 as rank
FROM user_scores
WHERE score > (SELECT score FROM user_scores WHERE user_id = ?);
```

**Indexes:**
```sql
-- Composite index for leaderboard queries
CREATE INDEX idx_scores_score_user ON user_scores(score DESC, user_id);

-- Alternative: Use Redis Sorted Set for real-time leaderboards
```

## Performance Considerations

### Index Scan vs Table Scan

**Index Scan:**
- Uses index to find rows
- Faster for selective queries
- Requires index lookup + table lookup

**Table Scan:**
- Scans entire table
- Faster for non-selective queries
- No index overhead

**When Table Scan is Better:**
- Query returns large percentage of rows (>10-20%)
- Table is small
- Index selectivity is very low

### Index Selectivity Threshold

**General Guidelines:**
- High selectivity (`>10%` unique): Index beneficial
- Medium selectivity (`1-10%`): May benefit from index
- Low selectivity (`<1%`): Index may not help, consider partial index

## Next Steps

- Review [Schema Design Patterns](./Schema%20Design%20Patterns.md) for indexing in common patterns
- Understand [Tradeoffs](./Tradeoffs.md) in indexing decisions
- Study [Best Practices](./Best%20Practices.md) for production systems

