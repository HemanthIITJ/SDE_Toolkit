# Chapter 14 – Transactions & Concurrency Control  

Transactions turn a set of individual SQL statements into an all‑or‑nothing unit, shielding data from partial failure and multi‑user anomalies.  Correct ACID compliance is critical to financial ledgers, message queues, and any workload where data accuracy directly drives business correctness.

---

## 14.1  Introduction to Transactions  

Concept | Description
--------|------------
Definition | A sequence of operations executed as a single logical unit.  Either all persist or none do.
Scope | Typically bound to a single database session/connection.
Lifecycle | `BEGIN` → work → `COMMIT` or `ROLLBACK`.
Motivation | Crash safety, business rule consistency, concurrent user protection.

---

## 14.2  ACID Properties  

Property | Guarantee | Real‑World Manifestation
---------|-----------|-------------------------
Atomicity | All modifications inside a transaction succeed or fail together. | Power outage mid‑checkout doesn’t charge card without persisting order rows.
Consistency | Moves DB from one valid state to another, honoring constraints & triggers. | FK, CHECK, and business invariants never violated.
Isolation | Concurrent transactions appear to run sequentially (as per isolation level). | T1 can’t read half‑written values from T2.
Durability | Once committed, data survives crashes, restarts, replication failover. | WAL / redo logs flushed to stable storage before `COMMIT` returns.

---

## 14.3  Transaction Control Language (TCL)

### 14.3.1 Begin / Start  

RDBMS | Syntax
------|-------
PostgreSQL, MySQL | `START TRANSACTION;` or `BEGIN;`
SQL Server | `BEGIN TRANSACTION;`

### 14.3.2 Commit  

```sql
COMMIT;        -- flush log, release locks
```

### 14.3.3 Rollback  

```sql
ROLLBACK;      -- undo uncommitted changes, release locks
```

### 14.3.4 Savepoints  

Fine‑grained partial rollback.

```sql
BEGIN;
-- step 1
SAVEPOINT sp_before_step2;
-- step 2
ROLLBACK TO sp_before_step2;  -- undo step 2 only
COMMIT;
```

Use‑Cases  
• Batch import where only bad row subset needs retry.  
• Complex multi‑step business logic inside a stored proc.

---

## 14.4  Concurrency Anomalies  

Phenomenon | Description | Isolation Level Where Possible
-----------|-------------|---------------------------------
Lost Update | Two txns overwrite each other; later one loses earlier changes. | `Read Committed` (without locking)
Dirty Read | T2 reads uncommitted data from T1. | `Read Uncommitted`
Non‑Repeatable Read | Row read twice returns different values because other txn committed in between. | `Read Committed`
Phantom Read | Re‑executed range query sees new/removed rows. | `Repeatable Read`

Illustration (Lost Update)  
```
T1             | T2
---------------|--------------
BEGIN;         |
SELECT bal...  |
               | BEGIN;
               | SELECT bal...
UPDATE ...     |
COMMIT;        |
               | UPDATE ...  -- overwrites
               | COMMIT;
```

---

## 14.5  Isolation Levels  

Level | Prevents | Allows | Implementation Clues
------|----------|--------|----------------------
Read Uncommitted | Nothing | Dirty/Non‑Repeatable/Phantom | Rarely supported (MySQL w/ `READ UNCOMMITTED`)
Read Committed | Dirty Reads | Non‑Repeatable & Phantom | Default in Postgres, Oracle; uses MVCC snapshot
Repeatable Read | Dirty + Non‑Repeatable | Phantoms | MySQL default (InnoDB gap locks), PostgreSQL MVCC + predicate locks
Serializable | All (strict serial order) | — | 2‑PL (SQL Server), SSI (Postgres), explicit locking

PostgreSQL Example  
```sql
SET TRANSACTION ISOLATION LEVEL SERIALIZABLE;
BEGIN;
... work ...
COMMIT;
```

Trade‑offs  
• Higher isolation ⇒ more locking / validation overhead.  
• Choose per‑txn level (`SET TRANSACTION …`) for critical sections; keep default lower for read‑heavy queries.

---

## 14.6  Locking Mechanisms  

Lock Type | Purpose | Scope
----------|---------|------
Shared (S) / Read | Allow concurrent readers, block writers. | Row/Range/Table
Exclusive (X) / Write | Sole access for DML; blocks S & X. | Row/Range/Table
Intent Locks (SQL Server) | Hierarchical, signal intention at page/table before row locks. |
Update (U) (SQL Server) | Single writer lock to avoid deadlock between two readers upgrading to writers. |
Gap / Next‑Key (InnoDB) | Protect non‑existing index ranges → stop phantom inserts. |
Predicate Locks (PostgreSQL SSI) | Logical range protection in Serializable level. |

### Deadlocks  

Definition: Cyclic dependency of locks → no‐progress.  
Detection: Engine aborts one victim (`ERROR: deadlock detected`, `SQLState 40P01`).  
Prevention Practices  
1. **Consistent lock order** across code paths.  
2. **Keep txns short**—perform reads, then writes, then commit.  
3. **Use appropriate indexes** so range locks are tight.  
4. **Retry logic** in application (idempotent operations).

---

## Cheat‑Sheet – Commands & Patterns  

Action | Snippet
------|---------
Begin + set Isolation | `START TRANSACTION ISOLATION LEVEL REPEATABLE READ;`
Commit | `COMMIT;`
Rollback | `ROLLBACK;`
Savepoint | `SAVEPOINT sp; ... ROLLBACK TO sp;`
Lock row manually (PostgreSQL) | `SELECT ... FOR UPDATE;`
Lock table (write) | `LOCK TABLE accounts IN EXCLUSIVE MODE;`
Detect current locks | `SELECT * FROM pg_locks;` (PostgreSQL)

---

## Best‑Practice Playbook  

1. Default to **Read Committed** + **short transactions**; escalate isolation only where anomalies break business rules.  
2. Always touch rows in **deterministic order** (e.g., ascending PKs) to minimize deadlocks.  
3. Wrap **retry loops** around code that may raise serialization or deadlock errors.  
4. Use **optimistic concurrency** (version/timestamp columns) when lock contention is high and conflict rate is low.  
5. Log **transaction duration** and lock‑wait metrics; alert when 95‑th > SLA.  
6. During batch ETL, consider `SET synchronous_commit = OFF` (Postgres) or bulk‑logged mode (SQL Server) to speed commits—revert afterward.

---

Strong transaction semantics underpin trustworthy data.  By leveraging the right isolation level, understanding engine locking internals, and writing disciplined transactional code, you ensure correctness while keeping throughput high under concurrent load.