# Database Normalization

## Overview

Normalization is a systematic approach to organizing data in a relational database to minimize redundancy and improve data integrity. The process involves decomposing larger tables into smaller, related tables and defining relationships between them.

## Why Normalize?

### Problems with Unnormalized Data

**1. Data Redundancy**
- Same data stored multiple times
- Wastes storage space
- Increases maintenance overhead
- Example: Storing customer name in every order record

**2. Update Anomalies**
- Need to update data in multiple places
- Risk of inconsistent data
- More complex update operations
- Example: Updating customer address requires updating all order records

**3. Insert Anomalies**
- Cannot insert data without related data
- Forced to insert incomplete records
- Violates data integrity
- Example: Cannot add a product without an order

**4. Delete Anomalies**
- Deleting one record may lose related data
- Unintended data loss
- Referential integrity issues
- Example: Deleting last order for a customer loses customer information

### Benefits of Normalization

- **Reduced Redundancy**: Data stored once
- **Data Integrity**: Easier to maintain consistency
- **Simpler Updates**: Update data in one place
- **Better Design**: Clear relationships between entities
- **Easier Maintenance**: Less data to manage
- **Storage Efficiency**: Reduced storage requirements

## Normal Forms

Normal forms are progressive levels of normalization, each building on the previous one.

### First Normal Form (1NF)

**Requirements:**
- All columns contain atomic (indivisible) values
- Each row is unique (primary key)
- No repeating groups
- No arrays or lists in single column
- Each column has a single data type

**Example - Violating 1NF:**

```sql
CREATE TABLE students (
    student_id INT PRIMARY KEY,
    name VARCHAR(100),
    subjects VARCHAR(500)  -- Contains: "Math, Science, English"
);
```

**Problems:**
- `subjects` column contains multiple values
- Cannot query individual subjects easily
- Cannot enforce constraints on subjects
- Difficult to update individual subjects

**Solution - 1NF:**

```sql
CREATE TABLE students (
    student_id INT PRIMARY KEY,
    name VARCHAR(100)
);

CREATE TABLE student_subjects (
    student_id INT,
    subject VARCHAR(100),
    PRIMARY KEY (student_id, subject),
    FOREIGN KEY (student_id) REFERENCES students(student_id)
);
```

**Key Points:**
- Each column contains single value
- No repeating groups
- Primary key ensures uniqueness
- Relationships properly defined

### Second Normal Form (2NF)

**Requirements:**
- Already in 1NF
- All non-key attributes are fully functionally dependent on the entire primary key
- No partial dependencies (for composite keys)
- If primary key is single column, table is automatically in 2NF

**Functional Dependency:**
- Attribute B is functionally dependent on attribute A if each value of A determines exactly one value of B
- Written as: A → B

**Example - Violating 2NF:**

```sql
CREATE TABLE enrollments (
    student_id INT,
    course_id INT,
    instructor VARCHAR(100),
    course_name VARCHAR(100),
    enrollment_date DATE,
    PRIMARY KEY (student_id, course_id)
);
```

**Problems:**
- `instructor` and `course_name` depend only on `course_id`, not on the combination
- Partial dependency: (student_id, course_id) → course_name, but course_id → course_name
- Redundancy: Course information repeated for each enrollment
- Update anomaly: Changing instructor requires updating multiple rows

**Solution - 2NF:**

```sql
CREATE TABLE courses (
    course_id INT PRIMARY KEY,
    course_name VARCHAR(100),
    instructor VARCHAR(100)
);

CREATE TABLE enrollments (
    student_id INT,
    course_id INT,
    enrollment_date DATE,
    PRIMARY KEY (student_id, course_id),
    FOREIGN KEY (course_id) REFERENCES courses(course_id)
);
```

**Key Points:**
- All non-key attributes depend on entire primary key
- No partial dependencies
- Related data separated into appropriate tables

### Third Normal Form (3NF)

**Requirements:**
- Already in 2NF
- All attributes are only dependent on the primary key
- No transitive dependencies
- Non-key attributes should not depend on other non-key attributes

**Transitive Dependency:**
- If A → B and B → C, then A → C is a transitive dependency
- In 3NF, we eliminate C depending on B when B is not the primary key

**Example - Violating 3NF:**

```sql
CREATE TABLE employees (
    employee_id INT PRIMARY KEY,
    name VARCHAR(100),
    department_id INT,
    department_name VARCHAR(100),
    department_location VARCHAR(100)
);
```

**Problems:**
- `department_name` and `department_location` depend on `department_id`, not directly on `employee_id`
- Transitive dependency: employee_id → department_id → department_name
- Redundancy: Department information repeated for each employee
- Update anomaly: Changing department location requires updating multiple rows

**Solution - 3NF:**

```sql
CREATE TABLE departments (
    department_id INT PRIMARY KEY,
    department_name VARCHAR(100),
    department_location VARCHAR(100)
);

CREATE TABLE employees (
    employee_id INT PRIMARY KEY,
    name VARCHAR(100),
    department_id INT,
    FOREIGN KEY (department_id) REFERENCES departments(department_id)
);
```

**Key Points:**
- No transitive dependencies
- All attributes depend directly on primary key
- Related data properly separated

### Boyce-Codd Normal Form (BCNF)

**Requirements:**
- Already in 3NF
- Every determinant is a candidate key
- Stricter than 3NF
- Eliminates overlapping candidate keys

**Determinant:**
- An attribute or set of attributes that determines another attribute

**Example - Violating BCNF:**

```sql
CREATE TABLE enrollments (
    student_id INT,
    course_id INT,
    instructor VARCHAR(100),
    PRIMARY KEY (student_id, course_id)
);
-- Assumption: Each instructor teaches only one course
-- Assumption: Each course can have multiple instructors
```

**Problems:**
- If instructor → course_id (each instructor teaches one course)
- But course_id is not a candidate key
- Violates BCNF: instructor is a determinant but not a candidate key

**Solution - BCNF:**

```sql
CREATE TABLE course_instructors (
    course_id INT,
    instructor VARCHAR(100),
    PRIMARY KEY (course_id, instructor)
);

CREATE TABLE enrollments (
    student_id INT,
    course_id INT,
    instructor VARCHAR(100),
    PRIMARY KEY (student_id, course_id),
    FOREIGN KEY (course_id, instructor) 
        REFERENCES course_instructors(course_id, instructor)
);
```

### Fourth Normal Form (4NF)

**Requirements:**
- Already in BCNF
- No multi-valued dependencies
- Each multi-valued dependency is represented by a separate table

**Multi-Valued Dependency (MVD):**
- A →→ B means for each value of A, there is a set of values for B independent of other attributes

**Example - Violating 4NF:**

```sql
CREATE TABLE employees (
    employee_id INT PRIMARY KEY,
    skill VARCHAR(100),
    language VARCHAR(100)
);
-- An employee can have multiple skills and multiple languages
```

**Problems:**
- Skills and languages are independent
- Multi-valued dependency: employee_id →→ skill, employee_id →→ language
- Redundancy: All combinations of skills and languages stored

**Solution - 4NF:**

```sql
CREATE TABLE employee_skills (
    employee_id INT,
    skill VARCHAR(100),
    PRIMARY KEY (employee_id, skill)
);

CREATE TABLE employee_languages (
    employee_id INT,
    language VARCHAR(100),
    PRIMARY KEY (employee_id, language)
);
```

### Fifth Normal Form (5NF) / Project-Join Normal Form (PJNF)

**Requirements:**
- Already in 4NF
- No join dependencies
- Every join dependency is implied by candidate keys

**When to Use:**
- Rarely needed in practice
- Only for complex scenarios with multiple overlapping relationships

## Normalization Example: E-commerce System

### Unnormalized Design

```sql
CREATE TABLE orders (
    order_id INT,
    customer_name VARCHAR(100),
    customer_email VARCHAR(100),
    customer_address VARCHAR(255),
    order_date DATE,
    product_id INT,
    product_name VARCHAR(100),
    product_price DECIMAL(10,2),
    quantity INT,
    category_name VARCHAR(100)
);
```

**Problems:**
- Customer information repeated for each order item
- Product information repeated for each order
- Cannot store customer without order
- Cannot store product without order

### 1NF: Atomic Values

```sql
CREATE TABLE orders (
    order_id INT PRIMARY KEY,
    customer_id INT,
    order_date DATE
);

CREATE TABLE order_items (
    order_id INT,
    product_id INT,
    quantity INT,
    PRIMARY KEY (order_id, product_id)
);
```

### 2NF: Remove Partial Dependencies

Already in 2NF (no composite keys with partial dependencies in this example).

### 3NF: Remove Transitive Dependencies

```sql
CREATE TABLE customers (
    customer_id INT PRIMARY KEY,
    customer_name VARCHAR(100),
    customer_email VARCHAR(100),
    customer_address VARCHAR(255)
);

CREATE TABLE products (
    product_id INT PRIMARY KEY,
    product_name VARCHAR(100),
    product_price DECIMAL(10,2),
    category_id INT
);

CREATE TABLE categories (
    category_id INT PRIMARY KEY,
    category_name VARCHAR(100)
);

CREATE TABLE orders (
    order_id INT PRIMARY KEY,
    customer_id INT,
    order_date DATE,
    FOREIGN KEY (customer_id) REFERENCES customers(customer_id)
);

CREATE TABLE order_items (
    order_id INT,
    product_id INT,
    quantity INT,
    price_at_purchase DECIMAL(10,2),  -- Store price at time of purchase
    PRIMARY KEY (order_id, product_id),
    FOREIGN KEY (order_id) REFERENCES orders(order_id),
    FOREIGN KEY (product_id) REFERENCES products(product_id)
);
```

**Key Design Decision:**
- Store `price_at_purchase` in `order_items` because product price may change over time
- Historical accuracy is important for orders

## When to Stop Normalizing

### Over-Normalization Problems

**1. Performance Issues**
- Too many joins required for queries
- Slower query execution
- Increased complexity

**2. Maintenance Complexity**
- More tables to manage
- More complex queries
- Harder to understand

**3. Query Complexity**
- Simple queries become complex
- Requires deep understanding of relationships
- Difficult for developers

### General Guidelines

**Stop at 3NF for Most Cases:**
- 3NF is sufficient for most applications
- Good balance between normalization and performance
- Easy to understand and maintain

**Consider BCNF When:**
- Complex business rules
- Overlapping candidate keys
- Data integrity critical

**Rarely Need 4NF/5NF:**
- Only for very specific scenarios
- Usually overkill for most applications

## Normalization Checklist

- [ ] All columns contain atomic values (1NF)
- [ ] No repeating groups (1NF)
- [ ] All non-key attributes fully depend on primary key (2NF)
- [ ] No transitive dependencies (3NF)
- [ ] Every determinant is a candidate key (BCNF, if needed)
- [ ] Multi-valued dependencies handled (4NF, if needed)

## Trade-offs

### Normalization Benefits
- Data integrity
- Reduced redundancy
- Easier updates
- Storage efficiency

### Normalization Costs
- More complex queries
- More joins required
- Potential performance impact
- More tables to manage

## Next Steps

After normalization, consider:
- [Denormalization](./Denormalization.md) for performance optimization
- [Indexing Strategies](./Indexing%20Strategies.md) for query optimization
- [Schema Design Patterns](./Schema%20Design%20Patterns.md) for common scenarios

