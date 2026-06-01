# Locking

## Pessimistic Locking
Pessimistic locking is a concurrency control strategy where a transaction locks data resources it needs to access, preventing other transactions from modifying (and sometimes reading) these resources until the lock is released. This approach assumes that conflicts between concurrent transactions are likely, so it's better to lock resources proactively to avoid them.

### How it Works
1.  **Acquire Lock**: Before accessing a resource (e.g., a row, a table), a transaction requests a lock on it.
2.  **Access Resource**: If the lock is granted, the transaction can safely read or modify the resource. If another transaction holds a conflicting lock, the requesting transaction must wait until the lock is released, or it might time out and fail.
3.  **Release Lock**: Locks are typically held until the transaction completes (either commits or rolls back), at which point they are released, allowing other waiting transactions to proceed.

### Use Cases / When to Use Pessimistic Locking
Pessimistic locking is suitable in scenarios where:
-   **High Contention is Expected**: When multiple transactions are highly likely to try to modify the same data simultaneously. In such cases, waiting for a lock might be more efficient than repeatedly aborting and retrying transactions (as might happen with optimistic locking).
-   **Cost of Conflict is High**: If resolving conflicts after they occur (e.g., through transaction rollbacks, data reconciliation, or user intervention) is more expensive or complex than the performance cost of waiting for a lock.
-   **Data Integrity is Paramount**: For operations where even temporary inconsistencies are unacceptable, and strict serialization of access to certain resources is required.
-   **Short Transactions**: Transactions that acquire locks, perform their work quickly, and then release locks minimize the duration for which resources are blocked, thereby reducing the negative impact on overall system concurrency.
-   **Examples**:
    -   Financial systems: Critical operations like transferring funds between accounts where double-spending or inconsistent balances must be strictly prevented.
    -   Inventory management: Updating stock levels during high-demand periods (e.g., flash sales) to prevent overselling.
    -   Booking systems: Reserving seats, hotel rooms, or other limited resources where concurrent bookings for the same item must be avoided.

### When Not to Use Pessimistic Locking
It might not be the best choice when:
-   **Conflicts are Rare**: In read-heavy systems or where data access patterns lead to infrequent conflicts, pessimistic locking can introduce unnecessary overhead and reduce concurrency without providing significant benefits. Optimistic locking is often better here.
-   **High Scalability and Throughput are Primary Goals**: The blocking nature of pessimistic locks can limit system throughput and scalability, as transactions may spend significant time waiting for locks.
-   **Long-Running Transactions**: If transactions hold locks for extended periods (e.g., transactions requiring user input while holding a lock), they can block other transactions significantly, leading to poor performance and user experience.
-   **Deadlocks are a Major Concern**: While deadlocks can occur in various concurrency control schemes, pessimistic locking inherently increases their likelihood. Systems must have robust deadlock detection and resolution mechanisms.

### Pros
-   **Strong Consistency and Data Integrity**: Guarantees that data is not modified by other transactions while locked, effectively preventing conflicts like lost updates or dirty reads (depending on lock type and scope).
-   **Simpler Conflict Management (for the application)**: Conflicts are prevented upfront by the database system. This can simplify application logic as there's less need for complex client-side conflict detection and resolution mechanisms.
-   **Predictable Behavior in High Contention**: In environments where conflicts are frequent, pessimistic locking can lead to more predictable system behavior (transactions wait) rather than numerous transaction aborts and retries that might occur with optimistic approaches.

### Cons
-   **Reduced Concurrency and Throughput**: Transactions may have to wait for locks to be released, leading to lower overall system throughput and potentially longer response times for users.
-   **Potential for Deadlocks**: A significant risk where two or more transactions are waiting for each other to release locks, creating a standstill. Systems must have mechanisms to detect and resolve deadlocks (e.g., by aborting one of the transactions), which can add complexity.
-   **Scalability Bottleneck**: Lock contention can become a major performance bottleneck in highly concurrent systems, limiting the system's ability to scale.
-   **Overhead of Lock Management**: Acquiring, checking, and releasing locks introduces processing overhead for the database system.
-   **Can Be Overly Restrictive**: If conflicts are actually rare, pessimistic locking might unnecessarily block transactions, leading to underutilization of resources.

### Pessimistic Locking Mechanisms in PostgreSQL
While PostgreSQL's primary concurrency control mechanism is Multi-Version Concurrency Control (MVCC), which is inherently optimistic, it provides explicit locking mechanisms that allow developers to implement pessimistic strategies when needed:

-   **Row-Level Locks**: Acquired using `SELECT ... FOR UPDATE` or `SELECT ... FOR SHARE` clauses.
    -   `SELECT ... FOR UPDATE`: Locks the selected rows, preventing other transactions from modifying them or acquiring `FOR UPDATE` / `FOR SHARE` locks on them until the current transaction ends. Other transactions attempting these operations will block.
    -   `SELECT ... FOR NO KEY UPDATE`: A weaker version of `FOR UPDATE`. It locks the selected rows but does not prevent other transactions from acquiring `SELECT ... FOR KEY SHARE` locks.
    -   `SELECT ... FOR SHARE`: Locks selected rows, preventing other transactions from modifying them or acquiring `FOR UPDATE` locks, but allows other transactions to read them or acquire `FOR SHARE` or `FOR KEY SHARE` locks.
    -   `SELECT ... FOR KEY SHARE`: A weaker version of `FOR SHARE`. It only blocks transactions attempting to acquire `FOR UPDATE` or `FOR NO KEY UPDATE` locks that would modify the key values of the locked rows.
-   **Table-Level Locks**: Acquired explicitly using the `LOCK TABLE table_name [IN lock_mode]` command (e.g., `LOCK TABLE my_table IN EXCLUSIVE MODE`). These are generally more restrictive and can severely impact concurrency, so they should be used cautiously.
-   **Advisory Locks**: These are application-defined locks. PostgreSQL provides functions (e.g., `pg_advisory_lock()`, `pg_try_advisory_lock()`) to acquire and release these locks. They don't lock data directly in tables but are used by applications to coordinate access to shared resources based on application-specific logic.

These mechanisms allow developers to selectively apply pessimistic locking patterns for specific operations or resources within a PostgreSQL database, complementing the default MVCC behavior.

## Optimistic Concurrency Control
Optimistic Concurrency Control (OCC) is a concurrency control method that assumes conflicts between transactions are rare. Instead of locking resources, it allows transactions to proceed without restrictions but checks for conflicts before committing. Here's how it works:

1. Read Phase
   - Transaction reads data from database without acquiring locks
   - Keeps track of read set (data items read)
   - Maintains a local copy of data items to be updated

2. Validation Phase
   - Before committing, checks if other transactions have modified data
   - Verifies no conflicts with concurrent transactions
   - Common validation approaches:
     - Timestamp-based validation
     - Version number checking

3. Write Phase
   - If validation succeeds, makes changes permanent
   - If validation fails, transaction aborts and restarts

Benefits:
- No blocking/waiting during transaction execution
- Good for read-heavy workloads
- Reduces deadlock possibility

Drawbacks:
- Higher abort rate under high contention
- Overhead of validation phase
- May need to restart transactions frequently

PostgreSQL Implementation:
- Uses Multi-Version Concurrency Control (MVCC)
- Maintains multiple versions of rows
- Combines with isolation levels for consistency
- Detects conflicts during transaction commit