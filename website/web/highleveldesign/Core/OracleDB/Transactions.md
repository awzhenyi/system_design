# Oracle Database Transactions

## What are Transactions?

A transaction is a sequence of one or more SQL operations that are treated as a single, atomic unit of work. In Oracle Database, transactions are fundamental for maintaining data integrity and consistency, especially in concurrent environments.

**Key characteristics and uses (ACID Properties):**

1. **Atomicity**: All operations within a transaction either complete successfully (commit) or none of them do (rollback). If any part of the transaction fails, the entire transaction is rolled back, leaving the database in its state prior to the transaction.

2. **Consistency**: A transaction brings the database from one valid state to another. It ensures that all data integrity constraints (e.g., primary keys, foreign keys, check constraints) are satisfied.

3. **Isolation**: Transactions are isolated from each other. This means that concurrent transactions do not interfere with each other's operations. Oracle achieves this through different isolation levels and MVCC.

4. **Durability**: Once a transaction is committed, its changes are permanent and will survive system failures. Oracle ensures durability through mechanisms like Redo Logs and Write-Ahead Logging.

**Common uses in Oracle Database:**

- **Ensuring Data Integrity**: When multiple related operations must succeed or fail together. For example, transferring money from one bank account to another involves debiting one account and crediting another; both operations must complete, or neither should.

- **Managing Concurrent Access**: Allowing multiple users/applications to access and modify data simultaneously without corrupting it.

- **Error Handling**: Providing a way to undo a series of changes if an error occurs during the process.

In Oracle Database, transactions are typically started implicitly with the first SQL command in a session, or explicitly using `BEGIN` or `SET TRANSACTION`. They are concluded with `COMMIT` (to make changes permanent) or `ROLLBACK` (to undo changes).

## Isolation Levels

Oracle provides the following isolation levels:

### 1. Read Committed (Default)

- **Default isolation level** in Oracle Database
- Each query sees only data committed before the query began
- Prevents dirty reads but allows non-repeatable reads and phantom reads
- Good balance between consistency and performance
- Uses statement-level read consistency

**Characteristics**:
- Each statement sees a consistent snapshot
- Different statements in same transaction may see different data
- No locks acquired for reads (MVCC)
- Writers don't block readers

### 2. Serializable

- Highest isolation level in Oracle
- Transaction sees database snapshot as of transaction start
- Prevents dirty reads, non-repeatable reads, and phantom reads
- Provides full serializability
- May cause serialization errors in high-concurrency environments

**Characteristics**:
- Transaction-level read consistency
- All statements in transaction see same snapshot
- May need to retry on serialization errors
- Higher overhead than Read Committed

**Serialization Errors**:
- Oracle may raise `ORA-08177: can't serialize access for this transaction`
- Application must handle and retry
- Indicates concurrent transaction conflict

### Oracle's Isolation Level Implementation

**Read Committed**:
- Uses **statement-level read consistency**
- Each statement sees data committed before statement execution
- Undo segments used to reconstruct consistent view
- No locks for reads (MVCC)

**Serializable**:
- Uses **transaction-level read consistency**
- All statements see data as of transaction start
- More undo segment usage
- May require serialization checks

**Note**: Oracle does not support Read Uncommitted or Repeatable Read isolation levels as defined in SQL standard. Read Committed provides statement-level consistency, and Serializable provides transaction-level consistency.

## Transaction Anomalies

### Dirty Read

**Definition**: A dirty read occurs when a transaction reads data that has been written by another transaction that has not yet been committed.

**Oracle Behavior**:
- **Prevented** in both Read Committed and Serializable
- MVCC ensures readers never see uncommitted data
- Undo segments used to provide consistent view

**Example**:
- Transaction A updates a row but hasn't committed yet
- Transaction B reads the row
- Transaction B sees the old version (not Transaction A's uncommitted change)
- Transaction A rolls back
- Transaction B still has correct (old) data

### Non-Repeatable Read

**Definition**: A non-repeatable read occurs when a transaction reads the same row twice and gets different data each time because another transaction modified the row between the two reads.

**Oracle Behavior**:
- **Allowed** in Read Committed (statement-level consistency)
- **Prevented** in Serializable (transaction-level consistency)

**Example in Read Committed**:
- Transaction A reads a row (sees value = 100)
- Transaction B updates the row (value = 200) and commits
- Transaction A reads the same row again (sees value = 200)
- Data has changed between reads

**Example in Serializable**:
- Transaction A reads a row (sees value = 100)
- Transaction B updates the row (value = 200) and commits
- Transaction A reads the same row again (still sees value = 100)
- Consistent view maintained throughout transaction

### Phantom Read

**Definition**: A phantom read occurs when a transaction re-executes a query returning a set of rows that satisfy a search condition and finds that the set of rows satisfying the condition has changed due to another recently-committed transaction.

**Oracle Behavior**:
- **Allowed** in Read Committed
- **Prevented** in Serializable

**Example in Read Committed**:
- Transaction A executes: `SELECT * FROM users WHERE age > 30` (returns 10 rows)
- Transaction B inserts a new user with age 35 and commits
- Transaction A executes the same query again (returns 11 rows)
- New row appears, creating a "phantom" row

**Example in Serializable**:
- Transaction A executes: `SELECT * FROM users WHERE age > 30` (returns 10 rows)
- Transaction B inserts a new user with age 35 and commits
- Transaction A executes the same query again (still returns 10 rows)
- Consistent result set maintained

## System Change Number (SCN)

### What is SCN

**SCN (System Change Number)** is Oracle's internal timestamp mechanism that uniquely identifies a committed version of the database at a point in time.

**Characteristics**:
- Monotonically increasing number
- Assigned to each committed transaction
- Used for versioning and read consistency
- Enables point-in-time recovery

### SCN Usage

**Read Consistency**:
- Each transaction assigned SCN at start
- Queries see data committed at or before transaction's SCN
- Undo segments used to reconstruct older versions

**Flashback Technology**:
- Query historical data using SCN
- `SELECT ... AS OF SCN <scn_number>`
- Useful for point-in-time queries

**Example**:
```sql
-- Query data as of specific SCN
SELECT * FROM users AS OF SCN 12345678;

-- Query data as of timestamp
SELECT * FROM users AS OF TIMESTAMP (SYSTIMESTAMP - INTERVAL '1' HOUR);
```

## Undo Segments and Read Consistency

### Undo Segments

**Purpose**:
- Store undo information for active transactions
- Enable rollback of uncommitted transactions
- Provide read consistency through MVCC
- Support flashback operations

**How They Work**:
- When row is updated, old version stored in undo segment
- Readers use undo to reconstruct consistent view
- Undo retained until all transactions that need it complete
- Automatic undo management (AUM) handles retention

**Undo Tablespace**:
- Dedicated tablespace for undo data
- Configurable retention period
- `UNDO_RETENTION` parameter controls retention
- Flashback operations depend on undo retention

### Read Consistency Mechanism

**Statement-Level (Read Committed)**:
1. Statement begins, gets current SCN
2. For each row accessed:
   - If row's SCN `>` statement SCN, use undo to reconstruct older version
   - If row's SCN `<=` statement SCN, use current version
3. Statement sees consistent snapshot

**Transaction-Level (Serializable)**:
1. Transaction begins, gets SCN
2. All statements use same SCN
3. Consistent view throughout transaction
4. May require serialization checks

## Transaction Control

### Starting Transactions

**Implicit Start**:
- First SQL statement in session starts transaction
- No explicit BEGIN needed

**Explicit Start**:
```sql
BEGIN;
-- or
SET TRANSACTION READ WRITE;
SET TRANSACTION ISOLATION LEVEL SERIALIZABLE;
```

### Committing Transactions

```sql
COMMIT;
-- or
COMMIT WORK;
```

**What Happens**:
- All changes made permanent
- Locks released
- SCN assigned
- Redo log flushed (for durability)

### Rolling Back Transactions

```sql
ROLLBACK;
-- or
ROLLBACK TO SAVEPOINT savepoint_name;
```

**What Happens**:
- All changes undone
- Locks released
- Undo segments used to restore previous state

### Savepoints

**Purpose**: Create intermediate points within transaction

```sql
SAVEPOINT sp1;
-- ... some operations ...
ROLLBACK TO SAVEPOINT sp1; -- Undo operations after sp1
-- ... continue transaction ...
COMMIT; -- Commit remaining changes
```

**Use Cases**:
- Partial rollback within transaction
- Error recovery
- Complex transaction logic

## Durability Mechanisms

### Redo Logs

**Purpose**: Ensure durability of committed transactions

**How It Works**:
1. Changes written to redo log before commit
2. Commit flushes redo log to disk
3. Database can recover from redo logs after crash
4. Write-Ahead Logging (WAL) principle

**Redo Log Files**:
- Circular set of redo log files
- Multiple groups for redundancy
- Archived redo logs for point-in-time recovery
- Critical for database recovery

### Write-Ahead Logging (WAL)

**Principle**: Write log before writing data

**Oracle Implementation**:
- Changes logged to redo log buffer
- Redo log buffer flushed to disk on commit
- Data blocks written asynchronously
- Ensures durability even if data blocks not yet written

## Best Practices

### Transaction Design

1. **Keep Transactions Short**:
   - Minimize lock duration
   - Reduce undo segment usage
   - Better concurrency

2. **Commit Frequently**:
   - Don't hold locks unnecessarily
   - Release resources promptly
   - Better for high-concurrency systems

3. **Handle Errors**:
   - Use savepoints for partial rollback
   - Implement retry logic for serialization errors
   - Proper error handling and logging

4. **Choose Appropriate Isolation Level**:
   - Use Read Committed for most cases
   - Use Serializable only when needed
   - Understand trade-offs

### Performance Considerations

1. **Undo Segment Usage**:
   - Long transactions use more undo
   - Monitor undo tablespace usage
   - Configure appropriate `UNDO_RETENTION`

2. **Serialization Errors**:
   - Serializable isolation may cause errors
   - Implement retry logic
   - Consider if Serializable is really needed

3. **Lock Contention**:
   - Even with MVCC, locks exist for writes
   - Minimize transaction duration
   - Use appropriate locking strategies

## Summary

Oracle Database transactions provide:

- **ACID Properties**: Atomicity, Consistency, Isolation, Durability
- **Isolation Levels**: Read Committed (default), Serializable
- **MVCC**: High concurrency through multi-versioning
- **Read Consistency**: Statement-level or transaction-level
- **Durability**: Redo logs ensure committed changes survive failures

**Key Takeaways**:
- Oracle uses MVCC for high concurrency
- Read Committed is default (statement-level consistency)
- Serializable provides transaction-level consistency
- SCN enables versioning and flashback
- Undo segments provide read consistency
- Redo logs ensure durability

Understanding Oracle's transaction model is crucial for designing systems that require strong consistency and high concurrency.
