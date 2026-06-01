# Schema Design Patterns

## Overview

Schema design patterns are reusable solutions to common database design problems. Understanding these patterns helps you make informed decisions when designing database schemas for system design interviews and production systems.

## 1. Master-Detail Pattern

### Description

One master record with multiple detail records. The master record represents the main entity, while detail records represent line items or sub-entities.

### Use Cases

- Orders and Order Items
- Invoices and Invoice Lines
- Shopping Carts and Cart Items
- Documents and Document Sections

### Schema Design

```sql
-- Master table
CREATE TABLE orders (
    order_id INT PRIMARY KEY,
    user_id INT,
    order_date TIMESTAMP,
    total_amount DECIMAL(10,2),
    status VARCHAR(50),
    FOREIGN KEY (user_id) REFERENCES users(user_id)
);

-- Detail table
CREATE TABLE order_items (
    order_item_id INT PRIMARY KEY,
    order_id INT,
    product_id INT,
    quantity INT,
    unit_price DECIMAL(10,2),
    subtotal DECIMAL(10,2),
    FOREIGN KEY (order_id) REFERENCES orders(order_id),
    FOREIGN KEY (product_id) REFERENCES products(product_id)
);

-- Indexes
CREATE INDEX idx_order_items_order_id ON order_items(order_id);
CREATE INDEX idx_order_items_product_id ON order_items(product_id);
```

### Key Considerations

- **Referential Integrity**: Detail records must have valid master record
- **Cascade Deletes**: Decide if deleting master should delete details
- **Aggregations**: Master table may store pre-computed totals
- **Queries**: Usually fetch master with all details together

### Query Pattern

```sql
-- Fetch order with items
SELECT 
    o.*,
    oi.*
FROM orders o
LEFT JOIN order_items oi ON o.order_id = oi.order_id
WHERE o.order_id = ?;
```

## 2. Hierarchical Data Patterns

### Pattern 2.1: Adjacency List

Stores parent reference in each node.

**Use Cases:**
- Categories with subcategories
- Organizational charts
- Comments with replies
- File systems

**Schema:**
```sql
CREATE TABLE categories (
    category_id INT PRIMARY KEY,
    name VARCHAR(100),
    parent_id INT,
    FOREIGN KEY (parent_id) REFERENCES categories(category_id)
);

CREATE INDEX idx_categories_parent ON categories(parent_id);
```

**Pros:**
- Simple structure
- Easy inserts/updates
- Easy to find direct children

**Cons:**
- Difficult to query entire subtree
- Requires recursive queries
- Slow for deep hierarchies

**Query - Get Children:**
```sql
-- Direct children
SELECT * FROM categories WHERE parent_id = ?;

-- All descendants (recursive - PostgreSQL)
WITH RECURSIVE category_tree AS (
    SELECT * FROM categories WHERE category_id = ?
    UNION ALL
    SELECT c.* FROM categories c
    INNER JOIN category_tree ct ON c.parent_id = ct.category_id
)
SELECT * FROM category_tree;
```

### Pattern 2.2: Closure Table

Stores all ancestor-descendant relationships explicitly.

**Schema:**
```sql
CREATE TABLE categories (
    category_id INT PRIMARY KEY,
    name VARCHAR(100)
);

CREATE TABLE category_closure (
    ancestor_id INT,
    descendant_id INT,
    depth INT,
    PRIMARY KEY (ancestor_id, descendant_id),
    FOREIGN KEY (ancestor_id) REFERENCES categories(category_id),
    FOREIGN KEY (descendant_id) REFERENCES categories(category_id)
);
```

**Pros:**
- Fast subtree queries
- No recursive queries needed
- Easy to find all ancestors/descendants

**Cons:**
- More storage (O(n²) in worst case)
- More complex inserts/updates
- Must maintain closure table

**Query - Get All Descendants:**
```sql
SELECT c.* 
FROM categories c
JOIN category_closure cc ON c.category_id = cc.descendant_id
WHERE cc.ancestor_id = ?;
```

### Pattern 2.3: Nested Set Model

Uses left and right boundaries to represent hierarchy.

**Schema:**
```sql
CREATE TABLE categories (
    category_id INT PRIMARY KEY,
    name VARCHAR(100),
    lft INT,
    rgt INT
);

CREATE INDEX idx_categories_lft_rgt ON categories(lft, rgt);
```

**Pros:**
- Fast subtree queries
- Single query for entire subtree
- Easy to determine depth

**Cons:**
- Complex inserts/updates
- Requires recalculating boundaries
- Not suitable for frequent updates

**Query - Get Subtree:**
```sql
SELECT * FROM categories
WHERE lft BETWEEN ? AND ?;
```

### Pattern 2.4: Materialized Path

Stores full path from root to node.

**Schema:**
```sql
CREATE TABLE categories (
    category_id INT PRIMARY KEY,
    name VARCHAR(100),
    path VARCHAR(500)  -- e.g., "/1/5/12/"
);

CREATE INDEX idx_categories_path ON categories(path);
```

**Pros:**
- Simple structure
- Easy to query by path
- Easy to determine depth

**Cons:**
- Path updates required when moving nodes
- String operations for queries
- Path length limitations

**Query - Get Subtree:**
```sql
SELECT * FROM categories
WHERE path LIKE '/1/%';
```

## 3. Audit Trail Pattern

Tracking changes to data over time.

### Pattern 3.1: Timestamp Columns

Simple tracking with created/updated timestamps.

```sql
CREATE TABLE users (
    user_id INT PRIMARY KEY,
    email VARCHAR(255),
    name VARCHAR(100),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
);
```

### Pattern 3.2: Version History Table

Separate table for historical versions.

```sql
CREATE TABLE products (
    product_id INT PRIMARY KEY,
    name VARCHAR(100),
    price DECIMAL(10,2),
    current_version INT
);

CREATE TABLE product_history (
    product_id INT,
    version INT,
    name VARCHAR(100),
    price DECIMAL(10,2),
    changed_at TIMESTAMP,
    changed_by INT,
    PRIMARY KEY (product_id, version),
    FOREIGN KEY (product_id) REFERENCES products(product_id)
);
```

### Pattern 3.3: Event Sourcing

Store all changes as events.

```sql
CREATE TABLE product_events (
    event_id BIGINT PRIMARY KEY,
    product_id INT,
    event_type VARCHAR(50),  -- CREATED, UPDATED, DELETED
    event_data JSON,
    occurred_at TIMESTAMP,
    user_id INT
);

CREATE INDEX idx_product_events_product ON product_events(product_id, occurred_at);
```

## 4. Soft Delete Pattern

Marking records as deleted instead of removing them.

```sql
CREATE TABLE users (
    user_id INT PRIMARY KEY,
    email VARCHAR(255),
    name VARCHAR(100),
    deleted_at TIMESTAMP NULL,
    is_deleted BOOLEAN DEFAULT FALSE
);

CREATE INDEX idx_users_deleted ON users(is_deleted) WHERE is_deleted = FALSE;
```

**Query Pattern:**
```sql
-- Always filter deleted records
SELECT * FROM users 
WHERE is_deleted = FALSE 
AND email = ?;
```

**Considerations:**
- Unique constraints must account for soft deletes
- Indexes should filter deleted records
- Cascade deletes become updates

## 5. Polymorphic Associations

Handling relationships where a foreign key can reference multiple tables.

### Pattern 5.1: Single Table Inheritance

All types in one table with type discriminator.

```sql
CREATE TABLE content_items (
    item_id INT PRIMARY KEY,
    item_type VARCHAR(50),  -- 'post', 'comment', 'video'
    title VARCHAR(255),
    content TEXT,
    author_id INT,
    created_at TIMESTAMP
);

CREATE INDEX idx_content_type ON content_items(item_type);
```

### Pattern 5.2: Class Table Inheritance

Separate table for each type with shared base table.

```sql
CREATE TABLE content_items (
    item_id INT PRIMARY KEY,
    item_type VARCHAR(50),
    author_id INT,
    created_at TIMESTAMP
);

CREATE TABLE posts (
    item_id INT PRIMARY KEY,
    title VARCHAR(255),
    content TEXT,
    FOREIGN KEY (item_id) REFERENCES content_items(item_id)
);

CREATE TABLE videos (
    item_id INT PRIMARY KEY,
    title VARCHAR(255),
    video_url VARCHAR(500),
    duration INT,
    FOREIGN KEY (item_id) REFERENCES content_items(item_id)
);
```

### Pattern 5.3: Concrete Table Inheritance

Separate table for each type, no shared table.

```sql
CREATE TABLE posts (
    post_id INT PRIMARY KEY,
    title VARCHAR(255),
    content TEXT,
    author_id INT,
    created_at TIMESTAMP
);

CREATE TABLE videos (
    video_id INT PRIMARY KEY,
    title VARCHAR(255),
    video_url VARCHAR(500),
    author_id INT,
    created_at TIMESTAMP
);
```

## 6. Tagging Pattern

Many-to-many relationship for tags.

```sql
CREATE TABLE articles (
    article_id INT PRIMARY KEY,
    title VARCHAR(255),
    content TEXT
);

CREATE TABLE tags (
    tag_id INT PRIMARY KEY,
    name VARCHAR(100) UNIQUE
);

CREATE TABLE article_tags (
    article_id INT,
    tag_id INT,
    PRIMARY KEY (article_id, tag_id),
    FOREIGN KEY (article_id) REFERENCES articles(article_id),
    FOREIGN KEY (tag_id) REFERENCES tags(tag_id)
);

CREATE INDEX idx_article_tags_article ON article_tags(article_id);
CREATE INDEX idx_article_tags_tag ON article_tags(tag_id);
```

**Query Pattern:**
```sql
-- Articles with specific tags
SELECT a.* 
FROM articles a
JOIN article_tags at ON a.article_id = at.article_id
WHERE at.tag_id IN (?, ?, ?)
GROUP BY a.article_id
HAVING COUNT(DISTINCT at.tag_id) = 3;
```

## 7. Star Schema (Data Warehousing)

Denormalized schema for analytical queries.

```sql
-- Fact table (central)
CREATE TABLE sales_fact (
    sale_id INT PRIMARY KEY,
    date_id INT,
    product_id INT,
    customer_id INT,
    store_id INT,
    quantity INT,
    amount DECIMAL(10,2)
);

-- Dimension tables
CREATE TABLE date_dim (
    date_id INT PRIMARY KEY,
    date DATE,
    year INT,
    quarter INT,
    month INT,
    day_of_week VARCHAR(20)
);

CREATE TABLE product_dim (
    product_id INT PRIMARY KEY,
    product_name VARCHAR(255),
    category_name VARCHAR(100),
    brand_name VARCHAR(100)
);

CREATE TABLE customer_dim (
    customer_id INT PRIMARY KEY,
    customer_name VARCHAR(255),
    city VARCHAR(100),
    state VARCHAR(100),
    country VARCHAR(100)
);
```

**Use Cases:**
- Data warehousing
- Business intelligence
- Reporting systems
- Analytics dashboards

## 8. Event Sourcing Pattern

Store all changes as a sequence of events.

```sql
CREATE TABLE events (
    event_id BIGINT PRIMARY KEY,
    aggregate_id VARCHAR(255),
    event_type VARCHAR(100),
    event_data JSON,
    occurred_at TIMESTAMP,
    version INT
);

CREATE INDEX idx_events_aggregate ON events(aggregate_id, version);
```

**Benefits:**
- Complete audit trail
- Time travel (reconstruct state at any point)
- Event replay
- Decoupled systems

**Challenges:**
- Eventual consistency
- Snapshot management
- Query complexity

## 9. CQRS Pattern (Command Query Responsibility Segregation)

Separate read and write models.

**Write Model (Normalized):**
```sql
CREATE TABLE orders (
    order_id INT PRIMARY KEY,
    user_id INT,
    status VARCHAR(50),
    created_at TIMESTAMP
);

CREATE TABLE order_items (
    order_item_id INT PRIMARY KEY,
    order_id INT,
    product_id INT,
    quantity INT
);
```

**Read Model (Denormalized):**
```sql
CREATE TABLE order_summaries (
    order_id INT PRIMARY KEY,
    user_id INT,
    user_name VARCHAR(255),
    total_items INT,
    total_amount DECIMAL(10,2),
    status VARCHAR(50),
    created_at TIMESTAMP
);
```

**Benefits:**
- Optimized for reads and writes separately
- Scalability
- Performance

**Challenges:**
- Data synchronization
- Eventual consistency
- Complexity

## 10. Sharding Pattern

Distributing data across multiple databases.

### Pattern 10.1: Range Sharding

```sql
-- Shard 1: user_id 1-1000000
CREATE TABLE users_1 (
    user_id INT PRIMARY KEY,
    email VARCHAR(255),
    name VARCHAR(100)
);

-- Shard 2: user_id 1000001-2000000
CREATE TABLE users_2 (
    user_id INT PRIMARY KEY,
    email VARCHAR(255),
    name VARCHAR(100)
);
```

### Pattern 10.2: Hash Sharding

```sql
-- Shard determined by hash(user_id) % num_shards
-- Shard 1: hash % 4 = 0
-- Shard 2: hash % 4 = 1
-- Shard 3: hash % 4 = 2
-- Shard 4: hash % 4 = 3
```

### Pattern 10.3: Directory-Based Sharding

```sql
-- Shard mapping table
CREATE TABLE shard_mapping (
    user_id INT PRIMARY KEY,
    shard_id INT
);
```

## Choosing the Right Pattern

### Decision Factors

1. **Query Patterns**: How will data be accessed?
2. **Update Frequency**: How often does data change?
3. **Scale Requirements**: Expected data volume?
4. **Consistency Requirements**: Strong vs eventual consistency?
5. **Performance Requirements**: Latency and throughput needs?

### Pattern Selection Guide

| Pattern | Use When | Avoid When |
|---------|----------|------------|
| Master-Detail | One-to-many relationships | Many-to-many relationships |
| Adjacency List | Shallow hierarchies, frequent updates | Deep hierarchies, complex queries |
| Closure Table | Complex hierarchy queries | Frequent hierarchy changes |
| Soft Delete | Need audit trail, referential integrity | High delete volume, storage sensitive |
| Star Schema | Analytical queries, reporting | OLTP, transactional systems |
| Event Sourcing | Need complete history, audit | Simple CRUD, strong consistency needed |
| CQRS | Different read/write patterns | Simple systems, consistency critical |

## Next Steps

- Review [Tradeoffs](./Tradeoffs.md) when choosing patterns
- Study [Interview Examples](./Interview%20Examples.md) for pattern applications
- Understand [Best Practices](./Best%20Practices.md) for production use

