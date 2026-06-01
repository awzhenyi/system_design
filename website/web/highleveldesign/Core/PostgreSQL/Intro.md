# PostgreSQL

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