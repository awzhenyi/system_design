# Oracle Database Locking

## Overview

While Oracle Database primarily uses Multi-Version Concurrency Control (MVCC) for concurrency control, it also provides explicit locking mechanisms for scenarios requiring pessimistic locking strategies. Understanding both MVCC and explicit locking is crucial for designing efficient Oracle-based systems.

## Pessimistic Locking

Pessimistic locking is a concurrency control strategy where a transaction locks data resources it needs to access, preventing other transactions from modifying (and sometimes reading) these resources until the lock is released. This approach assumes that conflicts between concurrent transactions are likely, so it's better to lock resources proactively to avoid them.

### How It Works

1. **Acquire Lock**: Before accessing a resource (e.g., a row, a table), a transaction requests a lock on it.
2. **Access Resource**: If the lock is granted, the transaction can safely read or modify the resource. If another transaction holds a conflicting lock, the requesting transaction must wait until the lock is released, or it might time out and fail.
3. **Release Lock**: Locks are typically held until the transaction completes (either commits or rolls back), at which point they are released, allowing other waiting transactions to proceed.

### Use Cases / When to Use Pessimistic Locking

Pessimistic locking is suitable in scenarios where:

- **High Contention is Expected**: When multiple transactions are highly likely to try to modify the same data simultaneously. In such cases, waiting for a lock might be more efficient than repeatedly aborting and retrying transactions.

- **Cost of Conflict is High**: If resolving conflicts after they occur (e.g., through transaction rollbacks, data reconciliation, or user intervention) is more expensive or complex than the performance cost of waiting for a lock.

- **Data Integrity is Paramount**: For operations where even temporary inconsistencies are unacceptable, and strict serialization of access to certain resources is required.

- **Short Transactions**: Transactions that acquire locks, perform their work quickly, and then release locks minimize the duration for which resources are blocked, thereby reducing the negative impact on overall system concurrency.

- **Examples**:
  - Financial systems: Critical operations like transferring funds between accounts where double-spending or inconsistent balances must be strictly prevented.
  - Inventory management: Updating stock levels during high-demand periods (e.g., flash sales) to prevent overselling.
  - Booking systems: Reserving seats, hotel rooms, or other limited resources where concurrent bookings for the same item must be avoided.

### When NOT to Use Pessimistic Locking

It might not be the best choice when:

- **Conflicts are Rare**: In read-heavy systems or where data access patterns lead to infrequent conflicts, pessimistic locking can introduce unnecessary overhead and reduce concurrency without providing significant benefits. MVCC is often better here.

- **High Scalability and Throughput are Primary Goals**: The blocking nature of pessimistic locks can limit system throughput and scalability, as transactions may spend significant time waiting for locks.

- **Long-Running Transactions**: If transactions hold locks for extended periods (e.g., transactions requiring user input while holding a lock), they can block other transactions significantly, leading to poor performance and user experience.

- **Deadlocks are a Major Concern**: While deadlocks can occur in various concurrency control schemes, pessimistic locking inherently increases their likelihood. Systems must have robust deadlock detection and resolution mechanisms.

### Pros

- **Strong Consistency and Data Integrity**: Guarantees that data is not modified by other transactions while locked, effectively preventing conflicts like lost updates.

- **Simpler Conflict Management (for the application)**: Conflicts are prevented upfront by the database system. This can simplify application logic as there's less need for complex client-side conflict detection and resolution mechanisms.

- **Predictable Behavior in High Contention**: In environments where conflicts are frequent, pessimistic locking can lead to more predictable system behavior (transactions wait) rather than numerous transaction aborts and retries that might occur with optimistic approaches.

### Cons

- **Reduced Concurrency and Throughput**: Transactions may have to wait for locks to be released, leading to lower overall system throughput and potentially longer response times for users.

- **Potential for Deadlocks**: A significant risk where two or more transactions are waiting for each other to release locks, creating a standstill. Oracle automatically detects and resolves deadlocks by rolling back one of the transactions.

- **Scalability Bottleneck**: Lock contention can become a major performance bottleneck in highly concurrent systems, limiting the system's ability to scale.

- **Overhead of Lock Management**: Acquiring, checking, and releasing locks introduces processing overhead for the database system.

- **Can Be Overly Restrictive**: If conflicts are actually rare, pessimistic locking might unnecessarily block transactions, leading to underutilization of resources.

## Oracle Locking Mechanisms

While Oracle's primary concurrency control mechanism is Multi-Version Concurrency Control (MVCC), it provides explicit locking mechanisms that allow developers to implement pessimistic strategies when needed:

### Row-Level Locks

**SELECT ... FOR UPDATE**:
```sql
SELECT * FROM employees 
WHERE employee_id = 123 
FOR UPDATE;
```
- Locks the selected rows exclusively
- Prevents other transactions from modifying or acquiring `FOR UPDATE` locks
- Other transactions attempting these operations will block
- Lock released on commit or rollback

**SELECT ... FOR UPDATE NOWAIT**:
```sql
SELECT * FROM employees 
WHERE employee_id = 123 
FOR UPDATE NOWAIT;
```
- Attempts to acquire lock, but doesn't wait if lock unavailable
- Raises error immediately if lock cannot be acquired
- Useful for avoiding indefinite blocking

**SELECT ... FOR UPDATE WAIT `<seconds>`**:
```sql
SELECT * FROM employees 
WHERE employee_id = 123 
FOR UPDATE WAIT 5;
```
- Waits up to specified seconds for lock
- Raises error if lock not acquired within timeout
- Balance between blocking and immediate failure

**SELECT ... FOR UPDATE SKIP LOCKED**:
```sql
SELECT * FROM orders 
WHERE status = 'pending' 
FOR UPDATE SKIP LOCKED;
```
- Skips rows that are already locked
- Returns only unlocked rows
- Useful for work queues (process available work)

**SELECT ... FOR UPDATE OF `<column>`**:
```sql
SELECT * FROM employees e, departments d
WHERE e.dept_id = d.dept_id
FOR UPDATE OF e.salary;
```
- Locks only specific table's rows
- Useful in joins to lock only one table

### Table-Level Locks

**Explicit Table Locks**:
```sql
LOCK TABLE employees IN EXCLUSIVE MODE;
LOCK TABLE employees IN SHARE MODE;
LOCK TABLE employees IN ROW EXCLUSIVE MODE;
LOCK TABLE employees IN SHARE ROW EXCLUSIVE MODE;
```

**Lock Modes**:
- **EXCLUSIVE**: Most restrictive, prevents all other locks
- **SHARE**: Allows other share locks, prevents exclusive locks
- **ROW EXCLUSIVE**: Allows row locks, prevents exclusive table locks
- **SHARE ROW EXCLUSIVE**: More restrictive than share, less than exclusive

**When to Use**:
- Bulk operations (ALTER TABLE, TRUNCATE)
- Schema changes
- Should be used cautiously (highly restrictive)

### DML Locks

**Automatic Locks on DML**:
- **INSERT**: Acquires row-level exclusive lock
- **UPDATE**: Acquires row-level exclusive lock
- **DELETE**: Acquires row-level exclusive lock
- **SELECT**: No locks (MVCC provides read consistency)

**Lock Escalation**:
- Oracle does not escalate row locks to table locks
- Each row lock is independent
- Better concurrency than lock escalation

### Deadlock Detection

**Automatic Detection**:
- Oracle automatically detects deadlocks
- Detects every few seconds
- Chooses victim transaction to roll back

**Deadlock Resolution**:
- One transaction rolled back automatically
- Raises `ORA-00060: deadlock detected`
- Application should retry rolled-back transaction

**Preventing Deadlocks**:
- Acquire locks in consistent order
- Keep transactions short
- Use `NOWAIT` or `WAIT` to avoid indefinite blocking
- Use `SKIP LOCKED` for work queues

## Optimistic Concurrency Control

Optimistic Concurrency Control (OCC) is Oracle's default approach through MVCC. It assumes conflicts between concurrent transactions are rare and allows transactions to proceed without restrictions, checking for conflicts only when necessary.

### How Oracle MVCC Works

1. **Read Phase**:
   - Transaction reads data from database without acquiring locks
   - Uses undo segments to reconstruct consistent view
   - No blocking of other transactions

2. **Validation Phase**:
   - Before committing, Oracle checks for conflicts
   - Uses SCN (System Change Number) for version checking
   - Detects if data was modified by other transactions

3. **Write Phase**:
   - If validation succeeds, makes changes permanent
   - If validation fails (in Serializable), transaction may need to retry
   - Changes written to redo log for durability

### Benefits

- **No Blocking/Waiting**: Readers don't block writers, writers don't block readers (in most cases)
- **Good for Read-Heavy Workloads**: Many concurrent readers without contention
- **Reduces Deadlock Possibility**: Less lock contention means fewer deadlocks
- **High Concurrency**: Multiple transactions can proceed simultaneously

### Drawbacks

- **Higher Abort Rate Under High Contention**: In Serializable mode, may see more serialization errors
- **Overhead of Undo Segments**: Maintaining undo information has storage and performance cost
- **May Need to Retry**: Serializable transactions may need retry logic

### Oracle MVCC Implementation

- **Uses SCN (System Change Number)**: For versioning and read consistency
- **Undo Segments**: Store previous versions of rows
- **Read Consistency**: Statement-level (Read Committed) or transaction-level (Serializable)
- **No Read Locks**: Readers don't acquire locks (MVCC provides consistency)
- **Write Locks**: Writers acquire row-level locks for modifications

## Lock Types and Compatibility

### Lock Compatibility Matrix

| Lock Mode | None | Row Share | Row Exclusive | Share | Share Row Exclusive | Exclusive |
|-----------|-----|-----------|-------------|------|---------------------|-----------|
| **None** | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ |
| **Row Share** | ✓ | ✓ | ✓ | ✓ | ✓ | ✗ |
| **Row Exclusive** | ✓ | ✓ | ✓ | ✗ | ✗ | ✗ |
| **Share** | ✓ | ✓ | ✗ | ✓ | ✗ | ✗ |
| **Share Row Exclusive** | ✓ | ✓ | ✗ | ✗ | ✗ | ✗ |
| **Exclusive** | ✓ | ✗ | ✗ | ✗ | ✗ | ✗ |

### Lock Modes Explained

**Row Share (RS)**:
- Acquired with `SELECT ... FOR UPDATE`
- Allows other row share locks
- Prevents exclusive locks

**Row Exclusive (RX)**:
- Acquired automatically on INSERT, UPDATE, DELETE
- Allows other row exclusive locks
- Prevents share and exclusive table locks

**Share (S)**:
- Acquired with `LOCK TABLE ... IN SHARE MODE`
- Allows other share locks
- Prevents exclusive locks

**Exclusive (X)**:
- Most restrictive
- Prevents all other locks
- Used for DDL operations

## Best Practices

### Locking Strategies

1. **Use MVCC When Possible**:
   - Default behavior (no explicit locks)
   - High concurrency
   - Good for read-heavy workloads

2. **Use FOR UPDATE Sparingly**:
   - Only when pessimistic locking needed
   - Keep transactions short
   - Consider `NOWAIT` or `WAIT` to avoid blocking

3. **Avoid Table Locks**:
   - Highly restrictive
   - Use only for DDL operations
   - Consider row-level locks instead

4. **Handle Deadlocks**:
   - Implement retry logic
   - Acquire locks in consistent order
   - Use timeouts to avoid indefinite blocking

### Performance Considerations

1. **Lock Duration**:
   - Minimize time locks are held
   - Commit or rollback promptly
   - Don't hold locks during user input

2. **Lock Granularity**:
   - Prefer row-level locks over table locks
   - Lock only what you need
   - Consider lock escalation implications

3. **Deadlock Prevention**:
   - Consistent lock ordering
   - Short transactions
   - Use `SKIP LOCKED` for work queues

## Monitoring Locks

### Viewing Current Locks

**DBA_LOCKS View**:
```sql
SELECT * FROM dba_locks 
WHERE blocking = 'YES';
```

**V$LOCK View**:
```sql
SELECT * FROM v$lock 
WHERE type != 'MR';
```

**V$SESSION View**:
```sql
SELECT s.sid, s.serial#, s.username, s.status, s.blocking_session
FROM v$session s
WHERE s.blocking_session IS NOT NULL;
```

### Identifying Blocking Sessions

```sql
SELECT 
    blocking.session_id AS blocking_sid,
    blocking.serial# AS blocking_serial,
    waiting.session_id AS waiting_sid,
    waiting.serial# AS waiting_serial,
    waiting.event AS wait_event
FROM v$session blocking, v$session waiting
WHERE blocking.sid = waiting.blocking_session;
```

## Summary

Oracle Database provides both optimistic (MVCC) and pessimistic locking:

**MVCC (Default)**:
- High concurrency
- No read locks
- Read consistency through undo segments
- Good for most use cases

**Pessimistic Locking**:
- Explicit locks with `FOR UPDATE`
- Table locks for DDL
- Use when conflicts are likely
- Trade-off: Lower concurrency for stronger guarantees

**Key Takeaways**:
- Oracle primarily uses MVCC (optimistic)
- Explicit locks available when needed
- Row-level locks preferred over table locks
- Deadlocks automatically detected and resolved
- Monitor locks to identify contention
- Keep transactions short to minimize lock duration

Understanding Oracle's locking mechanisms is crucial for designing systems that balance consistency, concurrency, and performance.
