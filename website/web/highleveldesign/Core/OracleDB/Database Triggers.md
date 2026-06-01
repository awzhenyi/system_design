# Oracle Database Triggers

## Overview

Triggers are specialized stored procedures that automatically execute in response to specific events on a table or view. They are primarily used to enforce business rules, maintain data integrity, and automate system tasks. Oracle Database provides comprehensive trigger functionality with various timing and level options.

## What are Triggers?

**Definition**: A trigger is a named PL/SQL block that is stored in the database and automatically executed (fired) when a triggering event occurs.

**Triggering Events**:
- **DML Events**: INSERT, UPDATE, DELETE
- **DDL Events**: CREATE, ALTER, DROP
- **Database Events**: LOGON, LOGOFF, STARTUP, SHUTDOWN

**Common Uses**:
- Enforce business rules
- Maintain audit trails
- Automate data validation
- Implement complex constraints
- Synchronize related tables
- Calculate derived values

## Trigger Types

### Timing: BEFORE vs AFTER vs INSTEAD OF

#### BEFORE Triggers

**When They Fire**: Before the triggering event occurs

**Use Cases**:
- Data validation
- Modify data before it's written
- Prevent invalid operations
- Set default values

**Example**:
```sql
CREATE OR REPLACE TRIGGER validate_salary
BEFORE INSERT OR UPDATE ON employees
FOR EACH ROW
BEGIN
    IF :NEW.salary < 0 THEN
        RAISE_APPLICATION_ERROR(-20001, 'Salary cannot be negative');
    END IF;
    
    IF :NEW.salary > 1000000 THEN
        RAISE_APPLICATION_ERROR(-20002, 'Salary exceeds maximum');
    END IF;
END;
/
```

**Characteristics**:
- Can modify `:NEW` values
- Can prevent operation (raise error)
- Executes before constraint checking
- Useful for validation

#### AFTER Triggers

**When They Fire**: After the triggering event has completed

**Use Cases**:
- Audit logging
- Update related tables
- Send notifications
- Maintain aggregates

**Example**:
```sql
CREATE OR REPLACE TRIGGER log_employee_changes
AFTER INSERT OR UPDATE OR DELETE ON employees
FOR EACH ROW
BEGIN
    IF INSERTING THEN
        INSERT INTO employee_audit (
            employee_id, action, changed_by, changed_at
        ) VALUES (
            :NEW.employee_id, 'INSERT', USER, SYSDATE
        );
    ELSIF UPDATING THEN
        INSERT INTO employee_audit (
            employee_id, action, old_salary, new_salary, changed_by, changed_at
        ) VALUES (
            :NEW.employee_id, 'UPDATE', :OLD.salary, :NEW.salary, USER, SYSDATE
        );
    ELSIF DELETING THEN
        INSERT INTO employee_audit (
            employee_id, action, changed_by, changed_at
        ) VALUES (
            :OLD.employee_id, 'DELETE', USER, SYSDATE
        );
    END IF;
END;
/
```

**Characteristics**:
- Cannot modify `:NEW` values
- Cannot prevent operation
- Executes after constraint checking
- Useful for side effects

#### INSTEAD OF Triggers

**When They Fire**: In place of the triggering event (for views)

**Use Cases**:
- Make views updatable
- Implement complex view update logic
- Handle multi-table views

**Example**:
```sql
-- Create view
CREATE VIEW employee_department_view AS
SELECT 
    e.employee_id,
    e.name,
    d.department_name
FROM employees e
JOIN departments d ON e.department_id = d.department_id;

-- Make view updatable with trigger
CREATE OR REPLACE TRIGGER update_employee_dept_view
INSTEAD OF UPDATE ON employee_department_view
FOR EACH ROW
BEGIN
    UPDATE employees
    SET department_id = (
        SELECT department_id 
        FROM departments 
        WHERE department_name = :NEW.department_name
    )
    WHERE employee_id = :NEW.employee_id;
END;
/
```

**Characteristics**:
- Only for views
- Replaces the DML operation
- Must implement the operation logic
- Useful for complex views

### Level: Statement-Level vs Row-Level

#### Statement-Level Triggers

**When They Fire**: Once per triggering statement, regardless of rows affected

**Syntax**:
```sql
CREATE OR REPLACE TRIGGER statement_trigger
BEFORE INSERT ON employees
BEGIN
    -- Executes once, even if 1000 rows inserted
    DBMS_OUTPUT.PUT_LINE('Insert operation started');
END;
/
```

**Use Cases**:
- Initialize variables
- Log statement execution
- Set session parameters
- Operations not dependent on row data

**Characteristics**:
- Fires once per statement
- Cannot access `:NEW` or `:OLD`
- Faster (no per-row overhead)
- Use `FOR EACH ROW` clause to make row-level

#### Row-Level Triggers

**When They Fire**: Once for each row affected by the triggering statement

**Syntax**:
```sql
CREATE OR REPLACE TRIGGER row_trigger
BEFORE INSERT ON employees
FOR EACH ROW
BEGIN
    -- Executes for each row inserted
    IF :NEW.employee_id IS NULL THEN
        :NEW.employee_id := employee_seq.NEXTVAL;
    END IF;
END;
/
```

**Use Cases**:
- Row-specific validation
- Modify row data
- Row-level auditing
- Calculate derived values

**Characteristics**:
- Fires for each row
- Can access `:NEW` and `:OLD`
- More overhead (per-row execution)
- Use when row data matters

### Combining Timing and Level

**Possible Combinations**:

1. **BEFORE Statement-Level**:
```sql
CREATE OR REPLACE TRIGGER before_statement
BEFORE INSERT ON employees
BEGIN
    -- Executes once before statement
END;
/
```

2. **BEFORE Row-Level**:
```sql
CREATE OR REPLACE TRIGGER before_row
BEFORE INSERT ON employees
FOR EACH ROW
BEGIN
    -- Executes before each row
END;
/
```

3. **AFTER Statement-Level**:
```sql
CREATE OR REPLACE TRIGGER after_statement
AFTER INSERT ON employees
BEGIN
    -- Executes once after statement
END;
/
```

4. **AFTER Row-Level**:
```sql
CREATE OR REPLACE TRIGGER after_row
AFTER INSERT ON employees
FOR EACH ROW
BEGIN
    -- Executes after each row
END;
/
```

**Execution Order**:
```
BEFORE Statement Trigger
  → BEFORE Row Trigger (for each row)
  → DML Operation
  → AFTER Row Trigger (for each row)
  → AFTER Statement Trigger
```

## Trigger Examples

### Example 1: Prevent Salary Decrease

```sql
CREATE OR REPLACE TRIGGER prevent_salary_decrease
BEFORE UPDATE OF salary ON employees
FOR EACH ROW
BEGIN
    IF :NEW.salary < :OLD.salary THEN
        RAISE_APPLICATION_ERROR(
            -20001, 
            'Salary decrease is not allowed. Old: ' || :OLD.salary || 
            ', New: ' || :NEW.salary
        );
    END IF;
END;
/
```

**Use Case**: Enforce business rule that salaries cannot decrease

### Example 2: Maintain Audit Trail

```sql
CREATE TABLE employee_audit (
    audit_id NUMBER PRIMARY KEY,
    employee_id NUMBER,
    action VARCHAR2(10),
    old_salary NUMBER,
    new_salary NUMBER,
    changed_by VARCHAR2(100),
    changed_at DATE
);

CREATE OR REPLACE TRIGGER audit_employee_changes
AFTER INSERT OR UPDATE OR DELETE ON employees
FOR EACH ROW
DECLARE
    v_action VARCHAR2(10);
BEGIN
    IF INSERTING THEN
        v_action := 'INSERT';
        INSERT INTO employee_audit (
            audit_id, employee_id, action, new_salary, 
            changed_by, changed_at
        ) VALUES (
            audit_seq.NEXTVAL, :NEW.employee_id, v_action, 
            :NEW.salary, USER, SYSDATE
        );
    ELSIF UPDATING THEN
        v_action := 'UPDATE';
        INSERT INTO employee_audit (
            audit_id, employee_id, action, old_salary, new_salary,
            changed_by, changed_at
        ) VALUES (
            audit_seq.NEXTVAL, :NEW.employee_id, v_action,
            :OLD.salary, :NEW.salary, USER, SYSDATE
        );
    ELSIF DELETING THEN
        v_action := 'DELETE';
        INSERT INTO employee_audit (
            audit_id, employee_id, action, old_salary,
            changed_by, changed_at
        ) VALUES (
            audit_seq.NEXTVAL, :OLD.employee_id, v_action,
            :OLD.salary, USER, SYSDATE
        );
    END IF;
END;
/
```

**Use Case**: Track all changes to employee table for compliance/audit

### Example 3: Auto-Generate Primary Key

```sql
CREATE OR REPLACE TRIGGER generate_employee_id
BEFORE INSERT ON employees
FOR EACH ROW
BEGIN
    IF :NEW.employee_id IS NULL THEN
        :NEW.employee_id := employee_seq.NEXTVAL;
    END IF;
END;
/
```

**Use Case**: Automatically assign primary key if not provided

### Example 4: Maintain Denormalized Data

```sql
CREATE OR REPLACE TRIGGER update_order_count
AFTER INSERT OR DELETE ON orders
FOR EACH ROW
BEGIN
    IF INSERTING THEN
        UPDATE users
        SET order_count = order_count + 1
        WHERE user_id = :NEW.user_id;
    ELSIF DELETING THEN
        UPDATE users
        SET order_count = order_count - 1
        WHERE user_id = :OLD.user_id;
    END IF;
END;
/
```

**Use Case**: Maintain denormalized aggregate (order_count) in users table

### Example 5: Complex Validation

```sql
CREATE OR REPLACE TRIGGER validate_order
BEFORE INSERT OR UPDATE ON orders
FOR EACH ROW
DECLARE
    v_user_status VARCHAR2(20);
BEGIN
    -- Check user status
    SELECT status INTO v_user_status
    FROM users
    WHERE user_id = :NEW.user_id;
    
    IF v_user_status != 'ACTIVE' THEN
        RAISE_APPLICATION_ERROR(
            -20002,
            'Cannot create order for inactive user'
        );
    END IF;
    
    -- Validate order amount
    IF :NEW.total_amount < 0 THEN
        RAISE_APPLICATION_ERROR(
            -20003,
            'Order amount cannot be negative'
        );
    END IF;
END;
/
```

**Use Case**: Complex validation involving multiple tables

## Trigger Best Practices

### Design Principles

1. **Keep Triggers Simple**:
   - Avoid complex business logic
   - Delegate to stored procedures if needed
   - Maintain readability

2. **Avoid Recursive Triggers**:
   - Trigger that modifies same table
   - Can cause infinite loops
   - Use `ALTER TRIGGER ... DISABLE` if needed

3. **Handle Errors Gracefully**:
   - Use `RAISE_APPLICATION_ERROR` for validation
   - Log errors appropriately
   - Don't silently fail

4. **Consider Performance**:
   - Row-level triggers execute for each row
   - Minimize work in triggers
   - Avoid queries in triggers when possible

### Common Patterns

**Pattern 1: Validation**:
```sql
CREATE OR REPLACE TRIGGER validate_data
BEFORE INSERT OR UPDATE ON table_name
FOR EACH ROW
BEGIN
    -- Validation logic
    IF condition THEN
        RAISE_APPLICATION_ERROR(-20001, 'Error message');
    END IF;
END;
/
```

**Pattern 2: Audit Trail**:
```sql
CREATE OR REPLACE TRIGGER audit_changes
AFTER INSERT OR UPDATE OR DELETE ON table_name
FOR EACH ROW
BEGIN
    INSERT INTO audit_table VALUES (...);
END;
/
```

**Pattern 3: Maintain Aggregates**:
```sql
CREATE OR REPLACE TRIGGER maintain_aggregate
AFTER INSERT OR UPDATE OR DELETE ON detail_table
FOR EACH ROW
BEGIN
    UPDATE summary_table
    SET aggregate_column = (
        SELECT aggregate_function(...)
        FROM detail_table
        WHERE ...
    );
END;
/
```

## Trigger Limitations and Considerations

### Performance Impact

**Row-Level Triggers**:
- Execute for each row affected
- Can significantly slow down bulk operations
- Consider disabling for bulk loads

**Disabling Triggers**:
```sql
-- Disable trigger
ALTER TRIGGER trigger_name DISABLE;

-- Disable all triggers on table
ALTER TABLE table_name DISABLE ALL TRIGGERS;

-- Re-enable
ALTER TRIGGER trigger_name ENABLE;
```

### Mutating Table Error

**Problem**: Trigger tries to read/modify table that's being modified

**Example**:
```sql
CREATE OR REPLACE TRIGGER check_total
BEFORE INSERT ON order_items
FOR EACH ROW
BEGIN
    -- This causes mutating table error
    SELECT SUM(amount) INTO v_total
    FROM order_items
    WHERE order_id = :NEW.order_id;
END;
/
```

**Solution**: Use compound triggers or package variables

**Compound Trigger Solution**:
```sql
CREATE OR REPLACE TRIGGER check_total_compound
FOR INSERT ON order_items
COMPOUND TRIGGER

    TYPE order_totals_t IS TABLE OF NUMBER INDEX BY NUMBER;
    g_order_totals order_totals_t;

    BEFORE STATEMENT IS
    BEGIN
        g_order_totals.DELETE;
    END BEFORE STATEMENT;

    AFTER EACH ROW IS
    BEGIN
        IF NOT g_order_totals.EXISTS(:NEW.order_id) THEN
            SELECT NVL(SUM(amount), 0) 
            INTO g_order_totals(:NEW.order_id)
            FROM order_items
            WHERE order_id = :NEW.order_id;
        END IF;
        g_order_totals(:NEW.order_id) := 
            g_order_totals(:NEW.order_id) + :NEW.amount;
    END AFTER EACH ROW;

    AFTER STATEMENT IS
    BEGIN
        -- Validate totals
        FOR i IN g_order_totals.FIRST .. g_order_totals.LAST LOOP
            IF g_order_totals(i) > 10000 THEN
                RAISE_APPLICATION_ERROR(-20004, 'Order total exceeds limit');
            END IF;
        END LOOP;
    END AFTER STATEMENT;

END;
/
```

### Trigger Dependencies

**Cascading Triggers**:
- Trigger can fire other triggers
- Can create complex execution chains
- Document trigger dependencies
- Test thoroughly

**Viewing Trigger Dependencies**:
```sql
SELECT 
    name,
    type,
    referenced_name,
    referenced_type
FROM user_dependencies
WHERE type = 'TRIGGER'
AND name = 'TRIGGER_NAME';
```

## Advanced Trigger Features

### Conditional Triggers

**WHEN Clause**:
```sql
CREATE OR REPLACE TRIGGER conditional_trigger
BEFORE UPDATE ON employees
FOR EACH ROW
WHEN (NEW.salary > 100000)
BEGIN
    -- Only fires if salary > 100000
    INSERT INTO high_salary_audit VALUES (...);
END;
/
```

**Benefits**:
- Reduces unnecessary trigger execution
- Better performance
- Cleaner logic

### Compound Triggers

**Purpose**: Single trigger with multiple timing points

**Structure**:
```sql
CREATE OR REPLACE TRIGGER compound_trigger
FOR INSERT ON table_name
COMPOUND TRIGGER

    -- Shared variables
    g_counter NUMBER := 0;

    BEFORE STATEMENT IS
    BEGIN
        -- Initialize
    END BEFORE STATEMENT;

    BEFORE EACH ROW IS
    BEGIN
        -- Per-row before
    END BEFORE EACH ROW;

    AFTER EACH ROW IS
    BEGIN
        -- Per-row after
    END AFTER EACH ROW;

    AFTER STATEMENT IS
    BEGIN
        -- Finalize
    END AFTER STATEMENT;

END;
/
```

**Benefits**:
- Share state across timing points
- Avoid mutating table errors
- More efficient than multiple triggers

### DDL Triggers

**System-Level Events**:
```sql
CREATE OR REPLACE TRIGGER ddl_trigger
AFTER CREATE OR ALTER OR DROP ON DATABASE
BEGIN
    INSERT INTO ddl_audit (
        event_type, object_type, object_name, 
        user_name, timestamp
    ) VALUES (
        ora_sysevent, ora_dict_obj_type, ora_dict_obj_name,
        ora_login_user, SYSDATE
    );
END;
/
```

**Use Cases**:
- Audit schema changes
- Enforce naming conventions
- Prevent certain DDL operations

### Database Event Triggers

**System Events**:
```sql
CREATE OR REPLACE TRIGGER logon_trigger
AFTER LOGON ON DATABASE
BEGIN
    INSERT INTO login_audit (
        username, login_time
    ) VALUES (
        USER, SYSDATE
    );
END;
/
```

**Available Events**:
- LOGON, LOGOFF
- STARTUP, SHUTDOWN
- SERVERERROR

## Monitoring and Debugging

### Viewing Triggers

**User Triggers**:
```sql
SELECT 
    trigger_name,
    table_name,
    triggering_event,
    status,
    trigger_type
FROM user_triggers
ORDER BY table_name, trigger_name;
```

**Trigger Code**:
```sql
SELECT text
FROM user_source
WHERE name = 'TRIGGER_NAME'
AND type = 'TRIGGER'
ORDER BY line;
```

### Enabling/Disabling Triggers

```sql
-- Enable trigger
ALTER TRIGGER trigger_name ENABLE;

-- Disable trigger
ALTER TRIGGER trigger_name DISABLE;

-- Enable all triggers on table
ALTER TABLE table_name ENABLE ALL TRIGGERS;

-- Disable all triggers on table
ALTER TABLE table_name DISABLE ALL TRIGGERS;
```

### Debugging Triggers

**Enable Debugging**:
```sql
ALTER SESSION SET PLSQL_DEBUG = TRUE;
```

**Use DBMS_OUTPUT**:
```sql
CREATE OR REPLACE TRIGGER debug_trigger
BEFORE INSERT ON employees
FOR EACH ROW
BEGIN
    DBMS_OUTPUT.PUT_LINE('Trigger fired for employee: ' || :NEW.employee_id);
END;
/
```

## Best Practices

### When to Use Triggers

**Good Use Cases**:
- Data validation
- Audit trails
- Maintaining denormalized data
- Enforcing complex business rules
- Automating routine tasks

**Avoid Triggers For**:
- Complex business logic (use stored procedures)
- Performance-critical operations (consider alternatives)
- Operations that can be done in application
- Logic that changes frequently

### Design Guidelines

1. **Keep It Simple**: Triggers should be focused and simple
2. **Document Well**: Clearly document trigger purpose and behavior
3. **Test Thoroughly**: Test all trigger paths
4. **Consider Performance**: Monitor trigger performance impact
5. **Handle Errors**: Proper error handling and logging

### Performance Considerations

1. **Minimize Work**: Keep trigger logic minimal
2. **Avoid Queries**: Avoid queries in triggers when possible
3. **Use WHEN Clause**: Conditionally fire triggers
4. **Bulk Operations**: Consider disabling for bulk loads
5. **Monitor Impact**: Track trigger execution time

## Summary

Oracle Database triggers provide powerful automation capabilities:

**Trigger Types**:
- **BEFORE/AFTER/INSTEAD OF**: Timing of execution
- **Statement/Row Level**: Granularity of execution
- **DML/DDL/Database Events**: Types of events

**Common Uses**:
- Data validation
- Audit trails
- Maintaining aggregates
- Enforcing business rules

**Key Takeaways**:
- Use triggers for automation, not complex logic
- Consider performance impact (especially row-level)
- Handle errors appropriately
- Test thoroughly
- Document trigger behavior
- Monitor and maintain triggers

Understanding triggers is crucial for implementing data integrity, audit requirements, and automated business logic in Oracle Database systems.
