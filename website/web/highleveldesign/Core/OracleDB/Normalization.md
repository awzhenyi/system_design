# Database Normalization

## Overview

Database normalization is a systematic approach to organizing data in a relational database to minimize redundancy and improve data integrity. This process involves decomposing larger tables into smaller, related tables and defining relationships between them. The primary goal is to eliminate data anomalies and ensure efficient data management.

## Why Normalize?

### Problems with Unnormalized Data

**Data Redundancy**:
- Same data stored multiple times
- Wastes storage space
- Increases maintenance overhead

**Update Anomalies**:
- Need to update data in multiple places
- Risk of inconsistent data
- More complex update operations

**Insert Anomalies**:
- Cannot insert data without related data
- Forced to insert incomplete records
- Violates data integrity

**Delete Anomalies**:
- Deleting one record may lose related data
- Unintended data loss
- Referential integrity issues

### Benefits of Normalization

- **Reduced Redundancy**: Data stored once
- **Data Integrity**: Easier to maintain consistency
- **Simpler Updates**: Update data in one place
- **Better Design**: Clear relationships between entities
- **Easier Maintenance**: Less data to manage

## Normal Forms

### First Normal Form (1NF)

**Requirements**:
- All columns contain atomic (indivisible) values
- Each row is unique (primary key)
- No repeating groups
- No arrays or lists in single column

**Example - Violating 1NF**:
```sql
CREATE TABLE students (
    student_id NUMBER PRIMARY KEY,
    name VARCHAR2(100),
    subjects VARCHAR2(500)  -- Contains: "Math, Science, English"
);
```

**Problem**: `subjects` column contains multiple values (comma-separated)

**Solution - 1NF**:
```sql
CREATE TABLE students (
    student_id NUMBER PRIMARY KEY,
    name VARCHAR2(100)
);

CREATE TABLE student_subjects (
    student_id NUMBER,
    subject VARCHAR2(100),
    PRIMARY KEY (student_id, subject),
    FOREIGN KEY (student_id) REFERENCES students(student_id)
);
```

**Key Points**:
- Each column contains single value
- No repeating groups
- Primary key ensures uniqueness

### Second Normal Form (2NF)

**Requirements**:
- Already in 1NF
- All non-key attributes are fully functionally dependent on the entire primary key
- No partial dependencies (for composite keys)

**Example - Violating 2NF**:
```sql
CREATE TABLE enrollments (
    student_id NUMBER,
    course_id NUMBER,
    instructor VARCHAR2(100),
    course_name VARCHAR2(100),
    PRIMARY KEY (student_id, course_id)
);
```

**Problem**: `instructor` and `course_name` depend only on `course_id`, not on the combination of `student_id` and `course_id`

**Solution - 2NF**:
```sql
CREATE TABLE courses (
    course_id NUMBER PRIMARY KEY,
    course_name VARCHAR2(100),
    instructor VARCHAR2(100)
);

CREATE TABLE enrollments (
    student_id NUMBER,
    course_id NUMBER,
    PRIMARY KEY (student_id, course_id),
    FOREIGN KEY (course_id) REFERENCES courses(course_id)
);
```

**Key Points**:
- Eliminate partial dependencies
- Create separate tables for independent entities
- Use foreign keys to maintain relationships

### Third Normal Form (3NF)

**Requirements**:
- Already in 2NF
- No transitive dependencies
- Non-key attributes do not depend on other non-key attributes

**Example - Violating 3NF**:
```sql
CREATE TABLE employees (
    employee_id NUMBER PRIMARY KEY,
    name VARCHAR2(100),
    department_id NUMBER,
    department_name VARCHAR2(100),
    department_location VARCHAR2(100)
);
```

**Problem**: `department_name` and `department_location` depend on `department_id`, which is not a primary key (transitive dependency)

**Solution - 3NF**:
```sql
CREATE TABLE departments (
    department_id NUMBER PRIMARY KEY,
    department_name VARCHAR2(100),
    department_location VARCHAR2(100)
);

CREATE TABLE employees (
    employee_id NUMBER PRIMARY KEY,
    name VARCHAR2(100),
    department_id NUMBER,
    FOREIGN KEY (department_id) REFERENCES departments(department_id)
);
```

**Key Points**:
- Eliminate transitive dependencies
- Non-key attributes depend only on primary key
- Create separate tables for dependent entities

### Boyce-Codd Normal Form (BCNF)

**Requirements**:
- Already in 3NF
- For every functional dependency X → Y, X should be a super key
- Stricter than 3NF

**Example - Violating BCNF**:
```sql
CREATE TABLE course_schedule (
    professor_id NUMBER,
    course_id NUMBER,
    room_number VARCHAR2(10),
    PRIMARY KEY (professor_id, course_id)
);
```

**Assumptions**:
- Each professor teaches only one course
- A course can be taught in multiple rooms
- Room depends on course, not on (professor, course)

**Problem**: `room_number` depends on `course_id`, but `course_id` is not a super key

**Solution - BCNF**:
```sql
CREATE TABLE professor_courses (
    professor_id NUMBER,
    course_id NUMBER,
    PRIMARY KEY (professor_id, course_id)
);

CREATE TABLE course_rooms (
    course_id NUMBER,
    room_number VARCHAR2(10),
    PRIMARY KEY (course_id, room_number)
);
```

**Key Points**:
- Every determinant must be a super key
- Eliminates remaining anomalies
- More restrictive than 3NF

## Denormalization

### What is Denormalization

Denormalization is the intentional process of combining tables or adding redundant data to improve read performance. It trades normalization benefits (reduced redundancy, better integrity) for performance gains.

### When to Denormalize

**Read-Heavy Workloads**:
- Many more reads than writes
- Queries frequently join multiple tables
- Performance is critical

**Performance Requirements**:
- Joins are performance bottleneck
- Response time requirements are strict
- Can tolerate some data inconsistency

**Specific Use Cases**:
- Data warehousing (analytical queries)
- Reporting systems
- Read-only replicas
- Cached/precomputed data

### Denormalization Strategies

**1. Duplicate Data**:
```sql
-- Normalized
CREATE TABLE orders (
    order_id NUMBER PRIMARY KEY,
    user_id NUMBER,
    FOREIGN KEY (user_id) REFERENCES users(user_id)
);

-- Denormalized (add username to orders)
CREATE TABLE orders (
    order_id NUMBER PRIMARY KEY,
    user_id NUMBER,
    username VARCHAR2(100),  -- Denormalized
    FOREIGN KEY (user_id) REFERENCES users(user_id)
);
```

**Benefits**: Eliminates JOIN for username
**Costs**: Redundancy, update complexity

**2. Precomputed Aggregates**:
```sql
-- Normalized: Calculate on-the-fly
SELECT user_id, COUNT(*) as order_count
FROM orders
GROUP BY user_id;

-- Denormalized: Store aggregate
CREATE TABLE users (
    user_id NUMBER PRIMARY KEY,
    name VARCHAR2(100),
    order_count NUMBER  -- Denormalized aggregate
);
```

**Benefits**: Fast reads, no aggregation needed
**Costs**: Must maintain on every insert/delete

**3. Flattened Hierarchies**:
```sql
-- Normalized: Multiple levels
CREATE TABLE regions (
    region_id NUMBER PRIMARY KEY,
    country_id NUMBER,
    FOREIGN KEY (country_id) REFERENCES countries(country_id)
);

-- Denormalized: Flatten
CREATE TABLE locations (
    location_id NUMBER PRIMARY KEY,
    city VARCHAR2(100),
    region VARCHAR2(100),  -- Denormalized
    country VARCHAR2(100)  -- Denormalized
);
```

**Benefits**: Single table, no joins
**Costs**: Redundancy, harder to maintain

**4. Materialized Views** (Oracle Feature):
```sql
-- Create materialized view (denormalized)
CREATE MATERIALIZED VIEW mv_user_orders
AS
SELECT 
    u.user_id,
    u.name,
    u.email,
    o.order_id,
    o.order_date,
    o.total_amount
FROM users u
JOIN orders o ON u.user_id = o.user_id;

-- Refresh periodically
EXEC DBMS_MVIEW.REFRESH('mv_user_orders');
```

**Benefits**: Pre-computed joins, fast queries
**Costs**: Storage, refresh overhead

### Denormalization Trade-offs

**Pros**:
- Faster reads (fewer joins)
- Simpler queries
- Better performance for read-heavy workloads
- Reduced database load

**Cons**:
- Data redundancy
- Update complexity (must update multiple places)
- Storage overhead
- Consistency challenges
- Slower writes

### Best Practices for Denormalization

1. **Measure First**: Profile queries to identify bottlenecks
2. **Denormalize Selectively**: Only where performance gains justify costs
3. **Maintain Consistency**: Use triggers or application logic to keep data in sync
4. **Document Decisions**: Clearly document denormalization choices
5. **Monitor Performance**: Track both read and write performance

## Normalization in Practice

### Design Process

1. **Start with Requirements**:
   - Understand data entities
   - Identify relationships
   - Determine access patterns

2. **Normalize to 3NF/BCNF**:
   - Start with normalized design
   - Ensures data integrity
   - Provides solid foundation

3. **Evaluate Performance**:
   - Profile queries
   - Identify bottlenecks
   - Measure join costs

4. **Selective Denormalization**:
   - Denormalize only where needed
   - Document decisions
   - Monitor impact

### Common Patterns

**Pattern 1: One-to-Many**:
```sql
-- Normalized
CREATE TABLE users (
    user_id NUMBER PRIMARY KEY,
    name VARCHAR2(100)
);

CREATE TABLE orders (
    order_id NUMBER PRIMARY KEY,
    user_id NUMBER,
    FOREIGN KEY (user_id) REFERENCES users(user_id)
);
```

**Pattern 2: Many-to-Many**:
```sql
-- Normalized
CREATE TABLE students (
    student_id NUMBER PRIMARY KEY
);

CREATE TABLE courses (
    course_id NUMBER PRIMARY KEY
);

CREATE TABLE enrollments (
    student_id NUMBER,
    course_id NUMBER,
    PRIMARY KEY (student_id, course_id),
    FOREIGN KEY (student_id) REFERENCES students(student_id),
    FOREIGN KEY (course_id) REFERENCES courses(course_id)
);
```

**Pattern 3: Hierarchical**:
```sql
-- Normalized (self-referencing)
CREATE TABLE employees (
    employee_id NUMBER PRIMARY KEY,
    name VARCHAR2(100),
    manager_id NUMBER,
    FOREIGN KEY (manager_id) REFERENCES employees(employee_id)
);
```

## Oracle-Specific Considerations

### Materialized Views

**Purpose**: Pre-computed query results (denormalized views)

```sql
CREATE MATERIALIZED VIEW mv_sales_summary
BUILD IMMEDIATE
REFRESH FAST ON COMMIT
AS
SELECT 
    product_id,
    SUM(quantity) as total_quantity,
    SUM(amount) as total_amount,
    COUNT(*) as order_count
FROM order_items
GROUP BY product_id;
```

**Refresh Options**:
- **IMMEDIATE**: Refresh immediately after creation
- **DEFERRED**: Refresh on demand
- **ON COMMIT**: Refresh when base table commits
- **ON DEMAND**: Manual refresh

**Benefits**:
- Fast queries (pre-computed)
- Reduced load on base tables
- Can be indexed separately

### Partitioning

**Purpose**: Divide large tables for better performance

```sql
-- Range partitioning
CREATE TABLE orders (
    order_id NUMBER,
    order_date DATE,
    customer_id NUMBER
)
PARTITION BY RANGE (order_date) (
    PARTITION p2023_q1 VALUES LESS THAN (DATE '2023-04-01'),
    PARTITION p2023_q2 VALUES LESS THAN (DATE '2023-07-01'),
    PARTITION p2023_q3 VALUES LESS THAN (DATE '2023-10-01'),
    PARTITION p2023_q4 VALUES LESS THAN (DATE '2024-01-01')
);
```

**Benefits**:
- Partition pruning (query only relevant partitions)
- Parallel operations
- Easier maintenance

### Index-Organized Tables (IOT)

**Purpose**: Store table data in index structure

```sql
CREATE TABLE users (
    user_id NUMBER PRIMARY KEY,
    email VARCHAR2(255),
    name VARCHAR2(100)
)
ORGANIZATION INDEX;
```

**Benefits**:
- Faster primary key lookups
- Reduced storage (no separate table)
- Good for lookup tables

## Summary

Database normalization is a fundamental design principle:

**Normal Forms**:
- **1NF**: Atomic values, no repeating groups
- **2NF**: No partial dependencies
- **3NF**: No transitive dependencies
- **BCNF**: Every determinant is super key

**Normalization Benefits**:
- Reduced redundancy
- Better data integrity
- Easier maintenance
- Clear relationships

**Denormalization**:
- Trade normalization for performance
- Use when reads heavily outweigh writes
- Implement selectively
- Maintain consistency

**Key Takeaways**:
- Start with normalized design (3NF/BCNF)
- Measure performance before denormalizing
- Denormalize selectively where needed
- Document design decisions
- Balance integrity and performance

Understanding normalization is crucial for designing efficient, maintainable database schemas that balance data integrity with performance requirements.
