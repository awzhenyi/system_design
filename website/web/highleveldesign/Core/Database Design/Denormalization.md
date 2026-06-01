# Denormalization

## Overview

Denormalization is the intentional introduction of redundancy into a database design to improve read performance, simplify queries, or optimize for specific access patterns. While normalization reduces redundancy, denormalization strategically reintroduces it for performance gains.

## When to Denormalize

### 1. Read-Heavy Workloads

When read operations significantly outnumber write operations:
- **Example**: Social media feeds, product catalogs, analytics dashboards
- **Benefit**: Faster reads by avoiding expensive joins
- **Trade-off**: Slower writes, more storage

### 2. Complex Joins Impacting Performance

When normalized queries require multiple joins that become bottlenecks:
- **Example**: User profile with order history, product details, and reviews
- **Benefit**: Single-table queries instead of 5+ table joins
- **Trade-off**: Data redundancy, update complexity

### 3. Reporting and Analytics

When generating reports requires aggregating data from multiple tables:
- **Example**: Sales reports, user activity summaries
- **Benefit**: Pre-computed aggregations, faster reporting
- **Trade-off**: Maintaining consistency, storage overhead

### 4. Real-Time Requirements

When low-latency reads are critical:
- **Example**: Leaderboards, live dashboards, real-time recommendations
- **Benefit**: Sub-millisecond response times
- **Trade-off**: Eventual consistency challenges

### 5. Frequently Accessed Data Together

When certain data is always queried together:
- **Example**: User name and avatar always displayed together
- **Benefit**: Single query instead of joins
- **Trade-off**: Redundant storage

## Denormalization Strategies

### 1. Flattening Tables

Combining related tables into a single table to avoid joins.

**Normalized:**
```sql
CREATE TABLE users (
    user_id INT PRIMARY KEY,
    name VARCHAR(100),
    email VARCHAR(100)
);

CREATE TABLE user_profiles (
    user_id INT PRIMARY KEY,
    bio TEXT,
    avatar_url VARCHAR(255),
    location VARCHAR(100),
    FOREIGN KEY (user_id) REFERENCES users(user_id)
);
```

**Denormalized:**
```sql
CREATE TABLE users (
    user_id INT PRIMARY KEY,
    name VARCHAR(100),
    email VARCHAR(100),
    bio TEXT,
    avatar_url VARCHAR(255),
    location VARCHAR(100)
);
```

**When to Use:**
- User profile always fetched with user data
- One-to-one relationship
- Profile rarely updated independently

### 2. Adding Redundant Columns

Storing frequently accessed related data in the main table.

**Normalized:**
```sql
CREATE TABLE orders (
    order_id INT PRIMARY KEY,
    customer_id INT,
    order_date DATE,
    FOREIGN KEY (customer_id) REFERENCES customers(customer_id)
);

CREATE TABLE customers (
    customer_id INT PRIMARY KEY,
    customer_name VARCHAR(100),
    customer_email VARCHAR(100)
);
```

**Denormalized:**
```sql
CREATE TABLE orders (
    order_id INT PRIMARY KEY,
    customer_id INT,
    customer_name VARCHAR(100),  -- Redundant
    customer_email VARCHAR(100), -- Redundant
    order_date DATE,
    FOREIGN KEY (customer_id) REFERENCES customers(customer_id)
);
```

**When to Use:**
- Order list always displays customer name
- Customer name rarely changes
- Read-heavy order queries

**Maintenance:**
- Update customer name in orders when customer name changes
- Use triggers or application logic to maintain consistency

### 3. Pre-Computed Aggregations

Storing calculated values instead of computing them on-the-fly.

**Normalized:**
```sql
CREATE TABLE users (
    user_id INT PRIMARY KEY,
    name VARCHAR(100)
);

CREATE TABLE orders (
    order_id INT PRIMARY KEY,
    user_id INT,
    total_amount DECIMAL(10,2),
    FOREIGN KEY (user_id) REFERENCES users(user_id)
);
-- Total orders calculated: SELECT COUNT(*) FROM orders WHERE user_id = ?
-- Total spent calculated: SELECT SUM(total_amount) FROM orders WHERE user_id = ?
```

**Denormalized:**
```sql
CREATE TABLE users (
    user_id INT PRIMARY KEY,
    name VARCHAR(100),
    total_orders INT DEFAULT 0,      -- Pre-computed
    total_spent DECIMAL(10,2) DEFAULT 0  -- Pre-computed
);
```

**When to Use:**
- Aggregations frequently queried
- Expensive to compute (large datasets)
- Acceptable slight delay in updates

**Maintenance:**
```sql
-- Update on order creation
UPDATE users 
SET total_orders = total_orders + 1,
    total_spent = total_spent + NEW.total_amount
WHERE user_id = NEW.user_id;
```

### 4. Materialized Views / Summary Tables

Creating separate tables with pre-computed data.

**Example: Daily Sales Summary**
```sql
CREATE TABLE daily_sales_summary (
    date DATE PRIMARY KEY,
    total_orders INT,
    total_revenue DECIMAL(10,2),
    average_order_value DECIMAL(10,2),
    unique_customers INT
);
```

**When to Use:**
- Complex aggregations
- Historical data analysis
- Reporting dashboards

**Maintenance:**
- Batch jobs to refresh summaries
- Incremental updates
- Scheduled during low-traffic periods

### 5. Duplicating Data Across Tables

Storing the same data in multiple tables for different access patterns.

**Example: Product Information**
```sql
-- Products table (source of truth)
CREATE TABLE products (
    product_id INT PRIMARY KEY,
    name VARCHAR(100),
    price DECIMAL(10,2),
    description TEXT,
    category_id INT
);

-- Order items (denormalized for historical accuracy)
CREATE TABLE order_items (
    order_id INT,
    product_id INT,
    product_name VARCHAR(100),  -- Duplicated
    product_price DECIMAL(10,2), -- Duplicated
    quantity INT,
    PRIMARY KEY (order_id, product_id)
);
```

**When to Use:**
- Historical accuracy (price at time of purchase)
- Different access patterns (current vs historical)
- Read performance critical

### 6. JSON/Array Columns

Storing related data as JSON or arrays (NoSQL-like approach).

**Example: Product Variants**
```sql
CREATE TABLE products (
    product_id INT PRIMARY KEY,
    name VARCHAR(100),
    variants JSON  -- {sizes: ["S", "M", "L"], colors: ["red", "blue"]}
);
```

**When to Use:**
- Flexible schema requirements
- Rarely queried individually
- Read together as a unit

**Trade-offs:**
- Harder to query individual elements
- No referential integrity
- Database-specific features

## Maintaining Consistency

### Challenge: Data Synchronization

Denormalization introduces the risk of inconsistent data. Strategies to maintain consistency:

### 1. Application-Level Updates

Update all copies in the same transaction:

```sql
BEGIN TRANSACTION;
    UPDATE customers SET name = 'New Name' WHERE customer_id = 1;
    UPDATE orders SET customer_name = 'New Name' WHERE customer_id = 1;
COMMIT;
```

**Pros:**
- Full control
- Can handle complex logic

**Cons:**
- Easy to miss updates
- Error-prone
- Requires careful coding

### 2. Database Triggers

Automatically update denormalized data:

```sql
CREATE TRIGGER update_order_customer_name
AFTER UPDATE ON customers
FOR EACH ROW
BEGIN
    UPDATE orders 
    SET customer_name = NEW.name 
    WHERE customer_id = NEW.customer_id;
END;
```

**Pros:**
- Automatic
- Centralized logic
- Less error-prone

**Cons:**
- Hidden logic (harder to debug)
- Performance overhead
- Database-specific

### 3. Event-Driven Updates

Use message queues or event streams:

```
Customer Updated Event → Update Orders Service → Update orders table
```

**Pros:**
- Decoupled
- Scalable
- Can handle eventual consistency

**Cons:**
- More complex architecture
- Eventual consistency
- Requires infrastructure

### 4. Periodic Reconciliation

Batch jobs to fix inconsistencies:

```sql
-- Run periodically (e.g., nightly)
UPDATE orders o
JOIN customers c ON o.customer_id = c.customer_id
SET o.customer_name = c.name
WHERE o.customer_name != c.name;
```

**Pros:**
- Simple
- Catches all inconsistencies

**Cons:**
- Not real-time
- May have temporary inconsistencies

## Denormalization Patterns

### Pattern 1: Read-Optimized Tables

Separate tables optimized for different read patterns.

**Example: Social Media Feed**
```sql
-- Write-optimized (normalized)
CREATE TABLE posts (
    post_id INT PRIMARY KEY,
    user_id INT,
    content TEXT,
    created_at TIMESTAMP
);

-- Read-optimized (denormalized)
CREATE TABLE user_feeds (
    user_id INT,
    post_id INT,
    author_name VARCHAR(100),  -- Denormalized
    author_avatar VARCHAR(255), -- Denormalized
    content TEXT,              -- Denormalized
    created_at TIMESTAMP,
    PRIMARY KEY (user_id, post_id)
);
```

### Pattern 2: Write-Through Cache

Denormalized table acts as a cache.

**Example: Product Catalog**
```sql
-- Source of truth (normalized)
CREATE TABLE products (
    product_id INT PRIMARY KEY,
    name VARCHAR(100),
    price DECIMAL(10,2),
    category_id INT
);

-- Denormalized cache
CREATE TABLE product_catalog_cache (
    product_id INT PRIMARY KEY,
    name VARCHAR(100),
    price DECIMAL(10,2),
    category_name VARCHAR(100),  -- Denormalized
    category_path VARCHAR(255)    -- Denormalized (e.g., "Electronics > Phones")
);
```

### Pattern 3: Time-Series Denormalization

Pre-aggregating time-series data.

**Example: Metrics**
```sql
-- Raw data (normalized)
CREATE TABLE metrics (
    metric_id INT PRIMARY KEY,
    timestamp TIMESTAMP,
    value DECIMAL(10,2),
    device_id INT
);

-- Denormalized aggregates
CREATE TABLE hourly_metrics (
    device_id INT,
    hour TIMESTAMP,
    avg_value DECIMAL(10,2),
    min_value DECIMAL(10,2),
    max_value DECIMAL(10,2),
    count INT,
    PRIMARY KEY (device_id, hour)
);
```

## Decision Framework

### Questions to Ask

1. **What is the read/write ratio?**
   - High reads → Consider denormalization
   - High writes → Prefer normalization

2. **What are the performance requirements?**
   - Sub-100ms reads → May need denormalization
   - Acceptable 500ms reads → Normalization may suffice

3. **How often does the data change?**
   - Rarely changes → Safe to denormalize
   - Frequently changes → Consider consistency costs

4. **What is the query pattern?**
   - Always joins same tables → Denormalize
   - Varies → Keep normalized

5. **What is the data size?**
   - Small → Normalization fine
   - Large → Denormalization may help

### Decision Matrix

| Factor | Normalize | Denormalize |
|--------|-----------|-------------|
| Write-heavy | ✅ | ❌ |
| Read-heavy | ❌ | ✅ |
| Data changes frequently | ✅ | ❌ |
| Data rarely changes | ❌ | ✅ |
| Complex queries | ❌ | ✅ |
| Simple queries | ✅ | ❌ |
| Storage cost sensitive | ✅ | ❌ |
| Performance critical | ❌ | ✅ |

## Best Practices

### 1. Start Normalized
- Begin with normalized design
- Denormalize only when needed
- Measure before optimizing

### 2. Document Denormalization
- Clearly document what is denormalized
- Explain why it was denormalized
- Document maintenance strategy

### 3. Monitor Consistency
- Implement consistency checks
- Alert on inconsistencies
- Regular reconciliation jobs

### 4. Measure Impact
- Benchmark before/after
- Monitor query performance
- Track storage overhead

### 5. Limit Scope
- Denormalize only what's needed
- Don't denormalize everything
- Focus on hot paths

## Common Mistakes

### 1. Premature Denormalization
- Denormalizing before measuring
- Optimizing without data
- Solution: Measure first, optimize second

### 2. Inconsistent Updates
- Forgetting to update denormalized data
- Partial updates
- Solution: Use triggers or transactions

### 3. Over-Denormalization
- Denormalizing everything
- No clear benefit
- Solution: Target specific use cases

### 4. Ignoring Consistency
- No consistency checks
- No reconciliation
- Solution: Implement monitoring

## Example: E-commerce Product Catalog

### Requirements
- Display products with category information
- Filter by category
- Search products
- High read volume (1000:1 read/write ratio)

### Normalized Design
```sql
CREATE TABLE categories (
    category_id INT PRIMARY KEY,
    name VARCHAR(100),
    parent_category_id INT
);

CREATE TABLE products (
    product_id INT PRIMARY KEY,
    name VARCHAR(100),
    price DECIMAL(10,2),
    category_id INT,
    FOREIGN KEY (category_id) REFERENCES categories(category_id)
);
```

**Query:**
```sql
SELECT p.*, c.name as category_name
FROM products p
JOIN categories c ON p.category_id = c.category_id
WHERE c.name = 'Electronics';
```

### Denormalized Design
```sql
CREATE TABLE products (
    product_id INT PRIMARY KEY,
    name VARCHAR(100),
    price DECIMAL(10,2),
    category_id INT,
    category_name VARCHAR(100),      -- Denormalized
    category_path VARCHAR(255)       -- Denormalized: "Electronics > Phones"
);
```

**Query:**
```sql
SELECT * FROM products
WHERE category_name = 'Electronics';
```

**Benefits:**
- No join required
- Faster queries
- Simpler queries

**Maintenance:**
```sql
CREATE TRIGGER update_product_category
AFTER UPDATE ON categories
FOR EACH ROW
BEGIN
    UPDATE products 
    SET category_name = NEW.name,
        category_path = build_category_path(NEW.category_id)
    WHERE category_id = NEW.category_id;
END;
```

## Next Steps

- Review [Indexing Strategies](./Indexing%20Strategies.md) to optimize denormalized tables
- Study [Schema Design Patterns](./Schema%20Design%20Patterns.md) for common denormalization patterns
- Understand [Tradeoffs](./Tradeoffs.md) in database design decisions

