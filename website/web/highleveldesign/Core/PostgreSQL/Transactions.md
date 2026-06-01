# Transactions
## What are transactions?
A transaction is a sequence of one or more SQL operations that are treated as a single, atomic unit of work. In PostgreSQL, transactions are fundamental for maintaining data integrity and consistency, especially in concurrent environments.

**Key characteristics and uses (ACID Properties):**

1.  **Atomicity**: All operations within a transaction either complete successfully (commit) or none of them do (rollback). If any part of the transaction fails, the entire transaction is rolled back, leaving the database in its state prior to the transaction.
2.  **Consistency**: A transaction brings the database from one valid state to another. It ensures that all data integrity constraints (e.g., primary keys, foreign keys, check constraints) are satisfied.
3.  **Isolation**: Transactions are isolated from each other. This means that concurrent transactions do not interfere with each other's operations. PostgreSQL achieves this through different isolation levels (discussed later).
4.  **Durability**: Once a transaction is committed, its changes are permanent and will survive system failures (e.g., power outages, crashes). PostgreSQL ensures durability through mechanisms like Write-Ahead Logging (WAL).

**Common uses in PostgreSQL:**

*   **Ensuring Data Integrity**: When multiple related operations must succeed or fail together. For example, transferring money from one bank account to another involves debiting one account and crediting another; both operations must complete, or neither should.
*   **Managing Concurrent Access**: Allowing multiple users/applications to access and modify data simultaneously without corrupting it.
*   **Error Handling**: Providing a way to undo a series of changes if an error occurs during the process.

In PostgreSQL, transactions are typically started implicitly with the first SQL command in a session, or explicitly using `BEGIN` or `START TRANSACTION`. They are concluded with `COMMIT` (to make changes permanent) or `ROLLBACK` (to undo changes).

## Isolation levels
Postgre provides the following isolation levels:

1. Read-Committed
   - Default isolation level in PostgreSQL
   - Each query sees only data committed before the query began
   - Prevents dirty reads but allows non-repeatable reads and phantom reads
   - Good balance between consistency and performance

2. Repeatable Read
   - Each transaction sees a snapshot of the database as of the start of the transaction
   - Prevents dirty reads and non-repeatable reads
   - May still allow phantom reads
   - Provides stronger consistency than Read-Committed

3. Serializable
   - Highest isolation level
   - Provides full serializability
   - Prevents dirty reads, non-repeatable reads, and phantom reads
   - Ensures transactions appear to execute one after another
   - May impact performance due to increased locking

## Transaction Anomalies

### Dirty Read
A dirty read occurs when a transaction reads data that has been written by another transaction that has not yet been committed. This is problematic because the uncommitted transaction might be rolled back, making the read data invalid. For example:
- Transaction A updates a row but hasn't committed yet
- Transaction B reads the updated row
- Transaction A rolls back
- Transaction B now has incorrect data

### Phantom Read
A phantom read occurs when a transaction re-executes a query returning a set of rows that satisfy a search condition and finds that the set of rows satisfying the condition has changed due to another recently-committed transaction. For example:
- Transaction A executes a query: `SELECT * FROM users WHERE age > 30`
- Transaction B inserts a new user with age 35 and commits
- Transaction A executes the same query again
- The new row appears in the second result set, creating a "phantom" row

### Non-repeatable Read
A non-repeatable read occurs when a transaction reads the same row twice and gets different data each time because another transaction modified the row between the two reads. For example:
- Transaction A reads a row
- Transaction B updates the same row and commits
- Transaction A reads the same row again
- The data has changed between the two reads