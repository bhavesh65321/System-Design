# Chapter 6: Transactions & Concurrency

## 1. What is a Transaction?

**Definition:** A transaction is a sequence of database operations that must be executed as a single atomic unit - either all succeed or all fail.

**Real-World Analogy:**
```
Bank Transfer: Alice sends $100 to Bob

Transaction:
1. Debit $100 from Alice's account
2. Credit $100 to Bob's account

MUST happen together:
✅ Both succeed → Transfer complete
❌ Both fail → No transfer
❌ Only 1 succeeds → DISASTER! (Money lost or duplicated)
```

**Visual:**
```
┌─────────────────────────────────────────────────────┐
│              TRANSACTION LIFECYCLE                   │
├─────────────────────────────────────────────────────┤
│                                                      │
│  BEGIN TRANSACTION                                   │
│         ↓                                            │
│  ┌──────────────────────────────────────────┐       │
│  │ Operation 1: Debit Alice $100            │       │
│  │ Operation 2: Credit Bob $100             │       │
│  │ Operation 3: Update transaction log      │       │
│  └──────────────────────────────────────────┘       │
│         ↓                                            │
│  COMMIT (All succeed) or ROLLBACK (All fail)        │
│         ↓                                            │
│  END TRANSACTION                                     │
│                                                      │
└─────────────────────────────────────────────────────┘
```

---

## 2. ACID Properties

### A - Atomicity (All or Nothing)

**Definition:** Transaction is indivisible - either all operations complete or none do.

```
┌─────────────────────────────────────────────────────┐
│ ATOMICITY EXAMPLE                                    │
├─────────────────────────────────────────────────────┤
│                                                      │
│ Transaction: Transfer $100 from Alice to Bob        │
│                                                      │
│ ✅ ATOMIC (Correct):                                 │
│    BEGIN                                             │
│    UPDATE accounts SET balance = balance - 100      │
│    WHERE user_id = 1;  -- Alice                      │
│    UPDATE accounts SET balance = balance + 100      │
│    WHERE user_id = 2;  -- Bob                        │
│    COMMIT;                                           │
│    Result: Both succeed or both fail                 │
│                                                      │
│ ❌ NOT ATOMIC (Dangerous):                           │
│    UPDATE accounts SET balance = balance - 100      │
│    WHERE user_id = 1;  -- Alice                      │
│    -- CRASH HERE! Bob never gets money              │
│    UPDATE accounts SET balance = balance + 100      │
│    WHERE user_id = 2;  -- Bob                        │
│    Result: Alice loses $100, Bob gets nothing!      │
│                                                      │
└─────────────────────────────────────────────────────┘
```

### C - Consistency (Valid State to Valid State)

**Definition:** Database moves from one valid state to another. All constraints maintained.

```
┌─────────────────────────────────────────────────────┐
│ CONSISTENCY EXAMPLE                                  │
├─────────────────────────────────────────────────────┤
│                                                      │
│ Constraint: Total money in system = $1000           │
│                                                      │
│ BEFORE: Alice=$500, Bob=$500 (Total=$1000) ✓        │
│                                                      │
│ Transaction: Transfer $100                          │
│ AFTER: Alice=$400, Bob=$600 (Total=$1000) ✓         │
│                                                      │
│ Consistency maintained!                              │
│                                                      │
│ ❌ INCONSISTENT STATE (if crash mid-transaction):   │
│ Alice=$400, Bob=$500 (Total=$900) ✗                 │
│ Money disappeared! (Prevented by Atomicity)         │
│                                                      │
└─────────────────────────────────────────────────────┘
```

### I - Isolation (Concurrent Independence)

**Definition:** Concurrent transactions don't interfere with each other.

```
┌─────────────────────────────────────────────────────┐
│ ISOLATION EXAMPLE                                    │
├─────────────────────────────────────────────────────┤
│                                                      │
│ Transaction 1: Transfer $100 Alice → Bob            │
│ Transaction 2: Transfer $50 Alice → Carol           │
│                                                      │
│ ✅ ISOLATED (Correct):                               │
│    T1 and T2 execute as if alone                     │
│    Result: Alice=$350, Bob=$600, Carol=$550         │
│                                                      │
│ ❌ NOT ISOLATED (Dirty Read):                        │
│    T1 reads Alice=$500                               │
│    T2 reads Alice=$500 (before T1 deducts)           │
│    Both deduct from same $500                        │
│    Result: Alice=$400 (should be $350!)             │
│                                                      │
└─────────────────────────────────────────────────────┘
```

### D - Durability (Permanent After Commit)

**Definition:** Once committed, data survives any failure (crash, power loss).

```
┌─────────────────────────────────────────────────────┐
│ DURABILITY EXAMPLE                                   │
├─────────────────────────────────────────────────────┤
│                                                      │
│ Transaction: Transfer $100 Alice → Bob              │
│                                                      │
│ Step 1: Write to transaction log (disk)             │
│ Step 2: Execute operations (memory)                 │
│ Step 3: COMMIT (write to disk)                      │
│ Step 4: Acknowledge to user                         │
│                                                      │
│ ✅ DURABLE:                                          │
│    Even if crash after COMMIT, data persists        │
│    Recovery process reads log and restores          │
│                                                      │
│ ❌ NOT DURABLE:                                      │
│    If crash before COMMIT, transaction rolled back  │
│    Data lost (as intended)                          │
│                                                      │
└─────────────────────────────────────────────────────┘
```

---

## 3. Concurrency Problems

### Problem 1: Dirty Read

**Definition:** Transaction reads uncommitted data from another transaction.

```
┌─────────────────────────────────────────────────────┐
│ DIRTY READ SCENARIO                                  │
├─────────────────────────────────────────────────────┤
│                                                      │
│ Initial: Alice balance = $500                       │
│                                                      │
│ TIME │ Transaction 1 (T1)    │ Transaction 2 (T2)   │
│──────┼──────────────────────┼──────────────────────│
│ 1    │ BEGIN                │                      │
│ 2    │ UPDATE Alice=$400    │                      │
│ 3    │                      │ BEGIN                │
│ 4    │                      │ READ Alice=$400 ❌   │
│ 5    │ ROLLBACK             │ (Dirty read!)        │
│ 6    │ Alice=$500 (restored)│                      │
│ 7    │                      │ COMMIT               │
│ 8    │                      │ (Used wrong value!)  │
│                                                      │
│ PROBLEM: T2 read uncommitted data from T1           │
│ T1 rolled back, but T2 already used the value       │
│                                                      │
└─────────────────────────────────────────────────────┘
```

### Problem 2: Lost Update

**Definition:** Two transactions update same data, one update is lost.

```
┌─────────────────────────────────────────────────────┐
│ LOST UPDATE SCENARIO                                 │
├─────────────────────────────────────────────────────┤
│                                                      │
│ Initial: Product stock = 100                        │
│                                                      │
│ TIME │ Transaction 1 (T1)    │ Transaction 2 (T2)   │
│──────┼──────────────────────┼──────────────────────│
│ 1    │ BEGIN                │ BEGIN                │
│ 2    │ READ stock=100       │                      │
│ 3    │                      │ READ stock=100       │
│ 4    │ stock = 100 - 10     │                      │
│ 5    │ WRITE stock=90       │                      │
│ 6    │ COMMIT               │                      │
│ 7    │                      │ stock = 100 - 5      │
│ 8    │                      │ WRITE stock=95 ❌    │
│ 9    │                      │ COMMIT               │
│                                                      │
│ RESULT: stock=95 (should be 85!)                    │
│ T1's update (10 units) was lost!                    │
│                                                      │
└─────────────────────────────────────────────────────┘
```

### Problem 3: Non-Repeatable Read

**Definition:** Same query returns different results within same transaction.

```
┌─────────────────────────────────────────────────────┐
│ NON-REPEATABLE READ SCENARIO                         │
├─────────────────────────────────────────────────────┤
│                                                      │
│ TIME │ Transaction 1 (T1)    │ Transaction 2 (T2)   │
│──────┼──────────────────────┼──────────────────────│
│ 1    │ BEGIN                │                      │
│ 2    │ SELECT Alice=$500    │                      │
│ 3    │                      │ BEGIN                │
│ 4    │                      │ UPDATE Alice=$400    │
│ 5    │                      │ COMMIT               │
│ 6    │ SELECT Alice=$400 ❌ │                      │
│ 7    │ (Different result!)  │                      │
│ 8    │ COMMIT               │                      │
│                                                      │
│ PROBLEM: Same query in T1 returned different values │
│ T2 modified data between T1's two reads             │
│                                                      │
└─────────────────────────────────────────────────────┘
```

### Problem 4: Phantom Read

**Definition:** Query returns different rows within same transaction.

```
┌─────────────────────────────────────────────────────┐
│ PHANTOM READ SCENARIO                                │
├─────────────────────────────────────────────────────┤
│                                                      │
│ TIME │ Transaction 1 (T1)    │ Transaction 2 (T2)   │
│──────┼──────────────────────┼──────────────────────│
│ 1    │ BEGIN                │                      │
│ 2    │ SELECT COUNT(*)      │                      │
│ 3    │ FROM orders          │                      │
│ 4    │ WHERE status='pending'│                      │
│ 5    │ Result: 5 orders     │                      │
│ 6    │                      │ BEGIN                │
│ 7    │                      │ INSERT new order     │
│ 8    │                      │ COMMIT               │
│ 9    │ SELECT COUNT(*)      │                      │
│ 10   │ FROM orders          │                      │
│ 11   │ WHERE status='pending'│                      │
│ 12   │ Result: 6 orders ❌  │                      │
│ 13   │ (Phantom row!)       │                      │
│ 14   │ COMMIT               │                      │
│                                                      │
│ PROBLEM: New row appeared (phantom) between queries │
│                                                      │
└─────────────────────────────────────────────────────┘
```

---

## 4. Isolation Levels

### Level 1: Read Uncommitted (Lowest)

```
┌─────────────────────────────────────────────────────┐
│ READ UNCOMMITTED                                     │
├─────────────────────────────────────────────────────┤
│ • Allows dirty reads                                │
│ • Fastest (no locks)                                │
│ • Least safe                                        │
│ • Rarely used in production                         │
│                                                      │
│ Problems Prevented: NONE                            │
│ Problems Allowed: Dirty Read, Lost Update,          │
│                  Non-Repeatable Read, Phantom       │
│                                                      │
│ SQL: SET TRANSACTION ISOLATION LEVEL                │
│      READ UNCOMMITTED;                              │
└─────────────────────────────────────────────────────┘
```

### Level 2: Read Committed (Common)

```
┌─────────────────────────────────────────────────────┐
│ READ COMMITTED                                       │
├─────────────────────────────────────────────────────┤
│ • Only reads committed data                         │
│ • Prevents dirty reads                              │
│ • Default in most databases                         │
│ • Good balance of safety and performance            │
│                                                      │
│ Problems Prevented: Dirty Read                      │
│ Problems Allowed: Lost Update, Non-Repeatable Read, │
│                  Phantom Read                       │
│                                                      │
│ SQL: SET TRANSACTION ISOLATION LEVEL                │
│      READ COMMITTED;                                │
│                                                      │
│ EXAMPLE:                                             │
│ T1: BEGIN                                            │
│ T1: SELECT Alice=$500                               │
│ T2: BEGIN                                            │
│ T2: UPDATE Alice=$400                               │
│ T2: COMMIT                                           │
│ T1: SELECT Alice=$400 (reads committed value) ✓     │
│ T1: COMMIT                                           │
└─────────────────────────────────────────────────────┘
```

### Level 3: Repeatable Read (Strict)

```
┌─────────────────────────────────────────────────────┐
│ REPEATABLE READ                                      │
├─────────────────────────────────────────────────────┤
│ • Same query always returns same rows               │
│ • Prevents dirty reads & non-repeatable reads       │
│ • Uses row-level locks                              │
│ • Slower than Read Committed                        │
│                                                      │
│ Problems Prevented: Dirty Read, Non-Repeatable Read │
│ Problems Allowed: Phantom Read, Lost Update         │
│                                                      │
│ SQL: SET TRANSACTION ISOLATION LEVEL                │
│      REPEATABLE READ;                               │
│                                                      │
│ EXAMPLE:                                             │
│ T1: BEGIN                                            │
│ T1: SELECT Alice=$500 (locks row)                   │
│ T2: BEGIN                                            │
│ T2: UPDATE Alice=$400 (waits for lock)              │
│ T1: SELECT Alice=$500 (same value) ✓                │
│ T1: COMMIT (releases lock)                          │
│ T2: UPDATE Alice=$400 (now succeeds)                │
│ T2: COMMIT                                           │
└─────────────────────────────────────────────────────┘
```

### Level 4: Serializable (Highest)

```
┌─────────────────────────────────────────────────────┐
│ SERIALIZABLE                                         │
├─────────────────────────────────────────────────────┤
│ • Transactions execute sequentially                 │
│ • Prevents all concurrency problems                 │
│ • Slowest (maximum locking)                         │
│ • Used for critical operations                      │
│                                                      │
│ Problems Prevented: ALL (Dirty Read, Lost Update,   │
│                    Non-Repeatable Read, Phantom)    │
│                                                      │
│ SQL: SET TRANSACTION ISOLATION LEVEL                │
│      SERIALIZABLE;                                  │
│                                                      │
│ EXAMPLE:                                             │
│ T1: BEGIN                                            │
│ T1: SELECT * FROM accounts (locks entire table)     │
│ T2: BEGIN                                            │
│ T2: SELECT * FROM accounts (waits)                  │
│ T1: COMMIT (releases all locks)                     │
│ T2: SELECT * FROM accounts (now executes)           │
│ T2: COMMIT                                           │
└─────────────────────────────────────────────────────┘
```

### Isolation Levels Comparison

```
┌──────────────────────────────────────────────────────────────────┐
│                    ISOLATION LEVELS MATRIX                        │
├──────────────────────┬──────┬──────┬──────┬──────┬──────────────┤
│ Level                │ DR   │ LU   │ NRR  │ PR   │ Performance  │
├──────────────────────┼──────┼──────┼──────┼──────┼──────────────┤
│ Read Uncommitted     │ ✓    │ ✗    │ ✗    │ ✗    │ Fastest      │
│ Read Committed       │ ✗    │ ✗    │ ✗    │ ✗    │ Fast         │
│ Repeatable Read      │ ✗    │ ✗    │ ✓    │ ✗    │ Slow         │
│ Serializable         │ ✗    │ ✗    │ ✓    │ ✓    │ Slowest      │
├──────────────────────┼──────┼──────┼──────┼──────┼──────────────┤
│ Legend:              │ DR=Dirty Read, LU=Lost Update,             │
│                      │ NRR=Non-Repeatable Read, PR=Phantom Read   │
│                      │ ✓=Prevented, ✗=Allowed                    │
└──────────────────────┴──────┴──────┴──────┴──────┴──────────────┘
```

---

## 5. Locking Mechanisms

### Shared Lock (Read Lock)

```
┌─────────────────────────────────────────────────────┐
│ SHARED LOCK (S-Lock)                                 │
├─────────────────────────────────────────────────────┤
│ • Multiple transactions can hold simultaneously      │
│ • Used for READ operations                          │
│ • Prevents writes while reading                     │
│                                                      │
│ EXAMPLE:                                             │
│ T1: SELECT * FROM accounts WHERE id=1 (S-Lock)     │
│ T2: SELECT * FROM accounts WHERE id=1 (S-Lock) ✓   │
│ T3: UPDATE accounts SET balance=100 WHERE id=1 ✗   │
│     (Waits for S-Locks to release)                  │
│                                                      │
│ Multiple readers allowed, writers blocked           │
│                                                      │
└─────────────────────────────────────────────────────┘
```

### Exclusive Lock (Write Lock)

```
┌─────────────────────────────────────────────────────┐
│ EXCLUSIVE LOCK (X-Lock)                              │
├─────────────────────────────────────────────────────┤
│ • Only one transaction can hold                      │
│ • Used for WRITE operations                         │
│ • Prevents all other access (read or write)         │
│                                                      │
│ EXAMPLE:                                             │
│ T1: UPDATE accounts SET balance=100 WHERE id=1      │
│     (X-Lock acquired)                               │
│ T2: SELECT * FROM accounts WHERE id=1 ✗             │
│     (Waits for X-Lock to release)                   │
│ T3: UPDATE accounts SET balance=200 WHERE id=1 ✗    │
│     (Waits for X-Lock to release)                   │
│                                                      │
│ Exclusive access - no other transactions allowed    │
│                                                      │
└─────────────────────────────────────────────────────┘
```

### Deadlock

**Definition:** Two transactions wait for each other's locks indefinitely.

```
┌─────────────────────────────────────────────────────┐
│ DEADLOCK SCENARIO                                    │
├─────────────────────────────────────────────────────┤
│                                                      │
│ Initial: Account A=$100, Account B=$200             │
│                                                      │
│ TIME │ Transaction 1 (T1)    │ Transaction 2 (T2)   │
│──────┼──────────────────────┼──────────────────────│
│ 1    │ BEGIN                │ BEGIN                │
│ 2    │ LOCK Account A       │                      │
│ 3    │ (X-Lock acquired)    │                      │
│ 4    │                      │ LOCK Account B       │
│ 5    │                      │ (X-Lock acquired)    │
│ 6    │ LOCK Account B ✗     │                      │
│ 7    │ (Waits for T2)       │                      │
│ 8    │                      │ LOCK Account A ✗     │
│ 9    │                      │ (Waits for T1)       │
│ 10   │ DEADLOCK! 💀         │ DEADLOCK! 💀         │
│                                                      │
│ T1 waits for T2, T2 waits for T1 → Infinite wait    │
│                                                      │
│ SOLUTION: Database detects deadlock, rolls back     │
│ one transaction, allows other to proceed            │
│                                                      │
└─────────────────────────────────────────────────────┘
```

---

## 6. MVCC (Multi-Version Concurrency Control)

**Definition:** Each transaction sees a consistent snapshot of data. Multiple versions of rows exist simultaneously.

```
┌─────────────────────────────────────────────────────┐
│ MVCC CONCEPT                                         │
├─────────────────────────────────────────────────────┤
│                                                      │
│ Instead of locking, maintain multiple versions      │
│                                                      │
│ Row: Account A                                       │
│ ┌──────────────────────────────────────────────┐    │
│ │ Version 1: balance=$100 (created by T1)      │    │
│ │ Version 2: balance=$90  (created by T2)      │    │
│ │ Version 3: balance=$85  (created by T3)      │    │
│ └──────────────────────────────────────────────┘    │
│                                                      │
│ T1 sees Version 1: $100                             │
│ T2 sees Version 2: $90                              │
│ T3 sees Version 3: $85                              │
│                                                      │
│ No locks needed! Each transaction sees its own      │
│ consistent snapshot                                 │
│                                                      │
└─────────────────────────────────────────────────────┘
```

### MVCC vs Locking

```
┌──────────────────────────────────────────────────────────────────┐
│                    MVCC vs LOCKING                                │
├──────────────────────┬──────────────────┬──────────────────────┤
│ Aspect               │ Locking          │ MVCC                 │
├──────────────────────┼──────────────────┼──────────────────────┤
│ Concurrency          │ Low              │ High                 │
│ Blocking             │ Yes              │ No                   │
│ Deadlocks            │ Possible         │ Rare                 │
│ Storage              │ Low              │ High (versions)      │
│ Read Performance     │ Slow (waits)     │ Fast (snapshot)      │
│ Write Performance    │ Fast             │ Slower (cleanup)     │
│ Databases            │ MySQL, SQL Server│ PostgreSQL, Oracle   │
└──────────────────────┴──────────────────┴──────────────────────┘
```

---

## 7. Real-World Examples

### Example 1: Bank Transfer (Critical)

```sql
-- Requires: SERIALIZABLE isolation level
BEGIN TRANSACTION;
SET TRANSACTION ISOLATION LEVEL SERIALIZABLE;

-- Debit from Alice
UPDATE accounts 
SET balance = balance - 100 
WHERE user_id = 1;

-- Credit to Bob
UPDATE accounts 
SET balance = balance + 100 
WHERE user_id = 2;

-- Record transaction
INSERT INTO transactions (from_user, to_user, amount, status)
VALUES (1, 2, 100, 'completed');

COMMIT;
```

### Example 2: E-commerce Inventory (High Concurrency)

```sql
-- Requires: READ COMMITTED with optimistic locking
BEGIN TRANSACTION;
SET TRANSACTION ISOLATION LEVEL READ COMMITTED;

-- Check stock
SELECT quantity, version 
FROM products 
WHERE product_id = 5;
-- Result: quantity=100, version=5

-- Deduct stock
UPDATE products 
SET quantity = quantity - 1, version = version + 1
WHERE product_id = 5 
AND version = 5;  -- Optimistic lock check

-- If no rows updated, version changed (conflict)
-- Retry transaction

COMMIT;
```

---

## 8. Interview Questions

### Beginner Level

**Q1: What is a transaction?**
> Sequence of operations that must execute as atomic unit - all succeed or all fail.

**Q2: What are ACID properties?**
> Atomicity (all/nothing), Consistency (valid state), Isolation (concurrent independence), Durability (permanent after commit).

**Q3: What is a dirty read?**
> Reading uncommitted data from another transaction that might rollback.

### Intermediate Level

**Q4: Explain isolation levels and their trade-offs.**
> Read Uncommitted (fastest, unsafe) → Read Committed (default) → Repeatable Read (strict) → Serializable (safest, slowest).

**Q5: What is a deadlock and how to prevent it?**
> Two transactions wait for each other's locks. Prevent by: lock ordering, timeouts, or MVCC.

### Senior Level

**Q6: Design transaction strategy for high-concurrency e-commerce system.**
> Use READ COMMITTED with optimistic locking. Maintain version numbers on rows. Retry on conflict. Use MVCC for reads. Batch updates to reduce lock contention.

**Q7: When would you use SERIALIZABLE vs MVCC?**
> SERIALIZABLE for critical operations (payments). MVCC for high-concurrency reads (feeds, analytics). Hybrid approach: SERIALIZABLE for writes, MVCC for reads.

---

## Summary

```
┌──────────────────────────────────────────────────────────────────┐
│                    KEY TAKEAWAYS                                  │
├──────────────────────────────────────────────────────────────────┤
│ 1. Transaction = Atomic unit (all or nothing)                    │
│ 2. ACID = Atomicity, Consistency, Isolation, Durability          │
│ 3. Concurrency Problems: Dirty Read, Lost Update, Non-Repeatable │
│    Read, Phantom Read                                            │
│ 4. Isolation Levels: Read Uncommitted → Read Committed →         │
│    Repeatable Read → Serializable                                │
│ 5. Locking: Shared (read) vs Exclusive (write)                   │
│ 6. Deadlock: Two transactions wait for each other                │
│ 7. MVCC: Multiple versions, no locks, high concurrency           │
│ 8. Trade-off: Safety vs Performance                              │
└──────────────────────────────────────────────────────────────────┘
```

---

**Estimated Reading Time**: 45-50 minutes  
**Next Chapter**: [Chapter 7: Database Security](../Chapter-7-Database-Security/)

*[← Back to Chapter 5: Indexing & Performance](../Chapter-5-Indexing-Performance/)*
