# Database Design Best Practices

## Overview

This guide compiles best practices for database design at the staff engineer level. These practices are derived from real-world experience, common pitfalls, and proven patterns for building production-ready database schemas.

## 1. Requirements and Planning

### Gather Comprehensive Requirements

**Functional Requirements:**
- What data needs to be stored?
- What operations are required? (CRUD)
- What are the business rules?
- What are the relationships between entities?

**Non-Functional Requirements:**
- Expected data volume and growth rate
- Read/write patterns and ratios
- Performance requirements (latency, throughput)
- Availability requirements
- Consistency requirements
- Security and compliance needs

**Best Practice:**
- Document requirements clearly
- Involve stakeholders early
- Consider future requirements
- Plan for evolution

### Create Data Model

**Steps:**
1. Identify entities and attributes
2. Define relationships
3. Create Entity-Relationship Diagram (ERD)
4. Validate with stakeholders
5. Iterate based on feedback

**Best Practice:**
- Start with conceptual model
- Progress to logical model
- Finalize physical model
- Use modeling tools (draw.io, Lucidchart, dbdiagram.io)

## 2. Naming Conventions

### Table Names

**Guidelines:**
- Use plural nouns: `users`, `orders`, `products`
- Use snake_case: `order_items` not `orderItems`
- Be descriptive: `user_sessions` not `sessions`
- Avoid abbreviations unless standard
- Keep names concise but clear

**Examples:**
```sql
-- Good
CREATE TABLE users (...);
CREATE TABLE order_items (...);
CREATE TABLE user_sessions (...);

-- Avoid
CREATE TABLE user (...);  -- Singular
CREATE TABLE orditems (...);  -- Abbreviation
CREATE TABLE sess (...);  -- Too short
```

### Column Names

**Guidelines:**
- Use snake_case: `user_id`, `created_at`
- Be consistent: Use `_id` suffix for foreign keys
- Use descriptive names: `order_date` not `date`
- Use `_at` for timestamps, `_on` for dates
- Avoid reserved words

**Examples:**
```sql
-- Good
CREATE TABLE orders (
    order_id INT PRIMARY KEY,
    user_id INT,
    order_date DATE,
    created_at TIMESTAMP,
    total_amount DECIMAL(10,2)
);

-- Avoid
CREATE TABLE orders (
    id INT PRIMARY KEY,  -- Not descriptive
    uid INT,  -- Abbreviation
    date DATE,  -- Ambiguous
    amt DECIMAL(10,2)  -- Abbreviation
);
```

### Index Names

**Guidelines:**
- Prefix with `idx_`: `idx_users_email`
- Include table name: `idx_orders_user_id`
- Include column names for composite: `idx_orders_user_date`
- Be descriptive

**Examples:**
```sql
-- Good
CREATE INDEX idx_users_email ON users(email);
CREATE INDEX idx_orders_user_date ON orders(user_id, order_date);

-- Avoid
CREATE INDEX email_idx ON users(email);  -- Wrong prefix
CREATE INDEX idx1 ON users(email);  -- Not descriptive
```

## 3. Primary Keys

### Choose Appropriate Primary Keys

**Guidelines:**
- Must be unique
- Cannot be NULL
- Should be stable (not change frequently)
- Should be simple (single column preferred)
- Consider surrogate vs natural keys

**Surrogate Keys (Recommended):**
```sql
-- Auto-incrementing integer
CREATE TABLE users (
    user_id INT PRIMARY KEY AUTO_INCREMENT,
    email VARCHAR(255) UNIQUE
);

-- UUID (for distributed systems)
CREATE TABLE users (
    user_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email VARCHAR(255) UNIQUE
);
```

**Natural Keys (Use Carefully):**
```sql
-- Only if truly unique and stable
CREATE TABLE countries (
    country_code CHAR(2) PRIMARY KEY,  -- ISO code, stable
    name VARCHAR(100)
);
```

**Best Practice:**
- Prefer surrogate keys for most tables
- Use natural keys only when appropriate (stable, unique, simple)
- Never use mutable data as primary key

## 4. Foreign Keys and Relationships

### Always Define Foreign Keys

**Benefits:**
- Enforces referential integrity
- Documents relationships
- Enables cascade operations
- Helps query optimizer

**Example:**
```sql
CREATE TABLE orders (
    order_id INT PRIMARY KEY,
    user_id INT,
    FOREIGN KEY (user_id) REFERENCES users(user_id)
);
```

### Index Foreign Keys

**Always index foreign keys for JOIN performance:**

```sql
CREATE TABLE orders (
    order_id INT PRIMARY KEY,
    user_id INT,
    FOREIGN KEY (user_id) REFERENCES users(user_id)
);

-- Always create index
CREATE INDEX idx_orders_user_id ON orders(user_id);
```

### Cascade Behavior

**Choose appropriate cascade behavior:**

```sql
-- ON DELETE CASCADE: Delete child records when parent deleted
CREATE TABLE order_items (
    order_item_id INT PRIMARY KEY,
    order_id INT,
    FOREIGN KEY (order_id) REFERENCES orders(order_id) 
        ON DELETE CASCADE
);

-- ON DELETE SET NULL: Set foreign key to NULL
CREATE TABLE posts (
    post_id INT PRIMARY KEY,
    user_id INT,
    FOREIGN KEY (user_id) REFERENCES users(user_id) 
        ON DELETE SET NULL
);

-- ON DELETE RESTRICT: Prevent deletion if children exist
CREATE TABLE orders (
    order_id INT PRIMARY KEY,
    user_id INT,
    FOREIGN KEY (user_id) REFERENCES users(user_id) 
        ON DELETE RESTRICT
);
```

## 5. Data Types

### Choose Appropriate Types

**Integers:**
- Use smallest sufficient type
- TINYINT: -128 to 127 (or 0 to 255 unsigned)
- SMALLINT: -32,768 to 32,767
- INT: -2,147,483,648 to 2,147,483,647
- BIGINT: Very large numbers

**Strings:**
- VARCHAR(n): Variable length, max n characters
- CHAR(n): Fixed length, exactly n characters
- TEXT: Large text (use when VARCHAR max insufficient)

**Decimals:**
- DECIMAL(p,s): Fixed precision (use for money)
- FLOAT/DOUBLE: Floating point (avoid for money)

**Dates/Times:**
- DATE: Date only
- TIME: Time only
- TIMESTAMP: Date and time (timezone-aware)
- DATETIME: Date and time (no timezone)

**Best Practice:**
```sql
-- Good
CREATE TABLE products (
    product_id BIGINT PRIMARY KEY,
    name VARCHAR(255),
    price DECIMAL(10,2),  -- Money: use DECIMAL
    stock_quantity INT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Avoid
CREATE TABLE products (
    product_id INT PRIMARY KEY,  -- May overflow
    name TEXT,  -- Overkill for product names
    price FLOAT,  -- Precision issues with money
    stock_quantity BIGINT  -- Overkill for stock
);
```

## 6. Constraints

### Use Constraints Liberally

**NOT NULL:**
```sql
CREATE TABLE users (
    user_id INT PRIMARY KEY,
    email VARCHAR(255) NOT NULL,  -- Required
    name VARCHAR(100) NOT NULL,
    bio TEXT NULL  -- Optional
);
```

**UNIQUE:**
```sql
CREATE TABLE users (
    user_id INT PRIMARY KEY,
    email VARCHAR(255) UNIQUE NOT NULL,  -- Unique email
    username VARCHAR(50) UNIQUE NOT NULL
);
```

**CHECK:**
```sql
CREATE TABLE products (
    product_id INT PRIMARY KEY,
    price DECIMAL(10,2) CHECK (price >= 0),  -- Positive price
    stock_quantity INT CHECK (stock_quantity >= 0),
    discount_percent INT CHECK (discount_percent >= 0 AND discount_percent <= 100)
);
```

**DEFAULT:**
```sql
CREATE TABLE orders (
    order_id INT PRIMARY KEY,
    status VARCHAR(50) DEFAULT 'pending',
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    is_active BOOLEAN DEFAULT TRUE
);
```

## 7. Normalization

### Normalize to 3NF (Generally)

**Best Practice:**
- Start with normalized design (3NF)
- Eliminate redundancy
- Ensure data integrity
- Denormalize only when needed

**When to Stop:**
- 3NF is sufficient for most cases
- BCNF only if needed for specific business rules
- 4NF/5NF rarely needed

**See [Normalization](./Normalization.md) for details.**

## 8. Denormalization

### Denormalize Strategically

**Best Practice:**
- Start normalized
- Measure performance
- Denormalize only hot paths
- Document denormalization decisions
- Maintain consistency

**When to Denormalize:**
- Read/write ratio > 10:1
- Queries require many joins
- Performance is critical
- Data rarely changes

**See [Denormalization](./Denormalization.md) for details.**

## 9. Indexing

### Index Strategically

**Always Index:**
- Primary keys (automatic)
- Foreign keys
- Unique constraints (automatic)
- Frequently filtered columns
- Frequently sorted columns
- JOIN columns

**Consider Indexing:**
- Columns in WHERE clauses
- Columns in ORDER BY
- Columns in GROUP BY
- Covering indexes for read-only queries

**Avoid Over-Indexing:**
- Too many indexes slow writes
- Monitor index usage
- Remove unused indexes

**See [Indexing Strategies](./Indexing%20Strategies.md) for details.**

## 10. Timestamps and Audit Fields

### Include Audit Fields

**Standard Fields:**
```sql
CREATE TABLE users (
    user_id INT PRIMARY KEY,
    email VARCHAR(255),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    created_by INT,
    updated_by INT
);
```

**Soft Delete:**
```sql
CREATE TABLE users (
    user_id INT PRIMARY KEY,
    email VARCHAR(255),
    deleted_at TIMESTAMP NULL,
    is_deleted BOOLEAN DEFAULT FALSE
);

-- Always filter deleted records
SELECT * FROM users 
WHERE is_deleted = FALSE;
```

## 11. Documentation

### Document Your Schema

**What to Document:**
- Table purposes and use cases
- Column meanings and constraints
- Relationships and foreign keys
- Indexes and their purposes
- Denormalization decisions
- Business rules
- Query patterns

**Example:**
```sql
-- Users table
-- Stores user account information
-- Used for authentication and user profiles
CREATE TABLE users (
    user_id INT PRIMARY KEY AUTO_INCREMENT,  -- Surrogate key
    email VARCHAR(255) UNIQUE NOT NULL,       -- Login identifier
    password_hash VARCHAR(255) NOT NULL,      -- Hashed password (bcrypt)
    full_name VARCHAR(100) NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    is_active BOOLEAN DEFAULT TRUE            -- Soft delete flag
);

-- Index for email lookups (login)
CREATE INDEX idx_users_email ON users(email);
```

## 12. Security

### Protect Sensitive Data

**Encryption:**
- Encrypt sensitive data at rest
- Use TLS for data in transit
- Hash passwords (never store plaintext)
- Consider field-level encryption for PII

**Access Control:**
- Use database users with minimal privileges
- Implement row-level security when needed
- Audit sensitive operations
- Use parameterized queries (prevent SQL injection)

**Example:**
```sql
-- Never store plaintext passwords
CREATE TABLE users (
    user_id INT PRIMARY KEY,
    email VARCHAR(255),
    password_hash VARCHAR(255) NOT NULL  -- Hashed with bcrypt/argon2
);

-- Use parameterized queries (application level)
-- Good: SELECT * FROM users WHERE email = ?
-- Bad: SELECT * FROM users WHERE email = '$email'
```

## 13. Performance Considerations

### Design for Performance

**Query Patterns:**
- Design schema based on query patterns
- Optimize for common queries
- Consider access patterns

**Partitioning:**
- Partition large tables by date or range
- Improves query performance
- Easier maintenance

**Caching:**
- Cache frequently accessed data
- Use Redis/Memcached for hot data
- Invalidate cache on updates

**Read Replicas:**
- Use read replicas for read-heavy workloads
- Distribute read load
- Accept eventual consistency

## 14. Migration and Versioning

### Plan for Schema Evolution

**Migration Strategy:**
- Version control schema changes
- Use migration tools (Flyway, Liquibase)
- Test migrations on staging
- Plan rollback strategies

**Backward Compatibility:**
- Add columns as nullable initially
- Deprecate before removing
- Support multiple versions during transition

**Example Migration:**
```sql
-- Migration 001: Add new column
ALTER TABLE users 
ADD COLUMN phone VARCHAR(20) NULL;  -- Nullable initially

-- Migration 002: Populate data
UPDATE users SET phone = ... WHERE phone IS NULL;

-- Migration 003: Make required (after data populated)
ALTER TABLE users 
MODIFY COLUMN phone VARCHAR(20) NOT NULL;
```

## 15. Testing

### Test Your Schema

**What to Test:**
- Data integrity constraints
- Foreign key relationships
- Index performance
- Query performance
- Migration scripts
- Edge cases

**Test Data:**
- Use realistic test data
- Test with large datasets
- Test edge cases (NULL, empty, max values)

## 16. Common Mistakes to Avoid

### 1. Over-Normalization
- Creating too many tables
- Complex queries with many joins
- **Solution**: Denormalize when needed

### 2. Under-Normalization
- Storing redundant data
- Update anomalies
- **Solution**: Normalize to 3NF

### 3. Poor Naming
- Unclear or inconsistent names
- **Solution**: Follow naming conventions

### 4. Missing Constraints
- No data validation
- Invalid data in database
- **Solution**: Use constraints liberally

### 5. Not Indexing Foreign Keys
- Slow JOIN operations
- **Solution**: Always index foreign keys

### 6. Wrong Data Types
- Using FLOAT for money
- Using VARCHAR for everything
- **Solution**: Choose appropriate types

### 7. Ignoring Query Patterns
- Schema doesn't match queries
- **Solution**: Design based on access patterns

### 8. Premature Optimization
- Optimizing without data
- **Solution**: Measure first, optimize second

### 9. No Documentation
- Unclear schema purpose
- **Solution**: Document thoroughly

### 10. No Migration Strategy
- Manual schema changes
- No version control
- **Solution**: Use migration tools

## 17. Interview-Specific Best Practices

### 1. Start with Requirements
- Clarify functional requirements
- Understand scale requirements
- Ask about performance needs

### 2. Design Incrementally
- Start with core entities
- Add relationships
- Consider edge cases
- Discuss scaling

### 3. Explain Tradeoffs
- Normalization vs denormalization
- Consistency vs availability
- Read vs write performance
- Storage vs performance

### 4. Consider Scaling
- Mention partitioning
- Discuss sharding
- Consider caching
- Plan for read replicas

### 5. Think About Queries
- Design indexes based on queries
- Consider query patterns
- Optimize for common operations

### 6. Discuss Alternatives
- Consider different approaches
- Explain why you chose your approach
- Mention when alternatives might be better

## 18. Production Checklist

### Before Going to Production

- [ ] All tables have primary keys
- [ ] Foreign keys defined and indexed
- [ ] Appropriate data types chosen
- [ ] Constraints in place (NOT NULL, UNIQUE, CHECK)
- [ ] Indexes created for query patterns
- [ ] Normalization appropriate (3NF typically)
- [ ] Denormalization documented and justified
- [ ] Audit fields included (created_at, updated_at)
- [ ] Soft delete implemented if needed
- [ ] Security considerations addressed
- [ ] Documentation complete
- [ ] Migration strategy in place
- [ ] Performance tested
- [ ] Backup strategy defined
- [ ] Monitoring and alerting set up

## Summary

Following these best practices will help you design robust, scalable, and maintainable database schemas. Remember:

1. **Start with requirements** - Understand what you're building
2. **Follow conventions** - Consistency is key
3. **Use constraints** - Let the database enforce rules
4. **Index strategically** - Balance reads and writes
5. **Normalize first** - Denormalize when needed
6. **Document everything** - Future you will thank you
7. **Plan for evolution** - Schemas change over time
8. **Test thoroughly** - Catch issues early
9. **Consider tradeoffs** - There are no perfect solutions
10. **Think about scale** - Design for growth

## Next Steps

- Review [Introduction](./Introduction.md) for fundamentals
- Study [Normalization](./Normalization.md) and [Denormalization](./Denormalization.md)
- Learn [Indexing Strategies](./Indexing%20Strategies.md)
- Explore [Schema Design Patterns](./Schema%20Design%20Patterns.md)
- Understand [Tradeoffs](./Tradeoffs.md) in design decisions
- Practice with [Interview Examples](./Interview%20Examples.md)

