# Chapter 14 – Transactions and Concurrency Control  
Transactions group SQL statements into atomic units, ensuring data integrity under concurrent access. Concurrency control balances performance with correctness.

---

## 14.1 Introduction to Transactions  
- A **transaction** is a sequence of operations treated as a single logical unit.  
- Guarantees that either **all** operations succeed or **none** do.  
- Essential in OLTP systems where multiple clients read/write simultaneously.

---

## 14.2 ACID Properties  
1. **Atomicity**  
   - All-or-nothing execution.  
   - On failure, database rolls back to pre-transaction state.

2. **Consistency**  
   - Transactions move the database from one valid state to another.  
   - Integrity constraints, triggers, and business rules always enforced.

3. **Isolation**  
   - Concurrent transactions do not interfere with each other.  
   - Isolation levels tune visibility of uncommitted changes.

4. **Durability**  
   - Once committed, changes persist, surviving crashes or power failures.  
   - Achieved via write-ahead logs and flush-to-disk mechanisms.

---

## 14.3 Transaction Control Language (TCL)  

### 14.3.1 Starting Transactions  
```sql
-- Standard
BEGIN TRANSACTION;
-- MySQL / PostgreSQL
START TRANSACTION;
```  

### 14.3.2 Saving Changes (COMMIT)  
```sql
COMMIT;
```
- Persists all changes made within the transaction.  
- Releases transaction-level locks.

### 14.3.3 Undoing Changes (ROLLBACK)  
```sql
ROLLBACK;
```
- Reverts all uncommitted changes.  
- Can target a specific savepoint (if defined).

### 14.3.4 Savepoints  
```sql
SAVEPOINT sp_name;

-- Rollback to savepoint without aborting entire transaction
ROLLBACK TO SAVEPOINT sp_name;

-- To discard a savepoint
RELEASE SAVEPOINT sp_name;
```
- Enable partial rollbacks within long transactions.  
- Help recover from mid-transaction errors gracefully.

---

## 14.4 Concurrency Issues  
Concurrency anomalies arise when transactions interleave improperly:

| Issue                | Description                                                   |
|----------------------|---------------------------------------------------------------|
| Lost Update          | Two transactions read the same row, both modify and commit – one update is lost. |
| Dirty Read           | Transaction reads uncommitted changes from another – may roll back later.       |
| Non‑Repeatable Read  | A row re-read within the same transaction returns different values due to another commit. |
| Phantom Read         | A query’s result set (e.g., rows matching a `WHERE` clause) changes if another transaction inserts/deletes rows. |

---

## 14.5 Isolation Levels  
Controls visibility of concurrent changes. Higher isolation ⇒ stronger correctness but lower concurrency.

| Level               | Prevents                          | Permits                                |
|---------------------|-----------------------------------|----------------------------------------|
| Read Uncommitted    | None                              | Dirty, non‑repeatable, phantom reads   |
| Read Committed      | Dirty reads                       | Non‑repeatable & phantom reads         |
| Repeatable Read     | Dirty & non‑repeatable reads      | Phantom reads                          |
| Serializable        | Dirty, non‑repeatable, phantom    | Only serializable (equivalent to sequential execution) |

- **Implementation notes**:  
  - PostgreSQL’s **Repeatable Read** also prevents phantoms via Serializable Snapshot Isolation.  
  - SQL Server names levels: `READ UNCOMMITTED`, `READ COMMITTED`, `REPEATABLE READ`, `SERIALIZABLE`.

---

## 14.6 Locking Mechanisms  
Databases use locks to enforce isolation:

### Shared Locks (S‑Lock)  
- Acquired for **read** operations under higher isolation levels.  
- Multiple concurrent S‑Locks allowed on same resource.

### Exclusive Locks (X‑Lock)  
- Acquired for **write** (INSERT/UPDATE/DELETE).  
- Blocks all other S‑ or X‑Locks on the resource.

### Deadlocks  
- Occur when two or more transactions wait indefinitely for locks held by each other.  
- **Detection & Resolution**:  
  - DBMS periodically builds a wait‑for graph and kills one victim (rollback) to break the cycle.  
  - Choose victim based on least work done or lowest priority.

### Lock Granularity  
- **Row‑level**: Maximum concurrency; higher overhead.  
- **Page/table‑level**: Lower overhead; higher contention.  
- Some systems support **intent locks** to coordinate multi‑granularity locking.

---

# Best Practices  
- Keep transactions **short**: minimize lock duration and contention.  
- Use the **lowest** isolation level that meets correctness needs.  
- Apply **optimistic concurrency** (versioning) when contention is low.  
- Catch and **retry** transactions on deadlock errors.  
- Monitor lock metrics (`pg_locks`, `sys.dm_tran_locks`) to identify hotspots.  

By understanding and applying transactions and isolation controls judiciously, you ensure data integrity and optimal concurrency in multi‑user environments.