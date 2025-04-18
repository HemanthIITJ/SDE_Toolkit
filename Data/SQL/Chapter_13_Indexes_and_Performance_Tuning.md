# Chapter 13 – Indexes and Performance Tuning  

Indexes are critical for high‑performance SQL. They accelerate data retrieval, shape query plans, and directly impact DML throughput. This chapter covers index fundamentals, creation, maintenance, and tuning techniques.

---

## 13.1 Why Indexes? Speeding Up Queries  
- Avoid full‑table scans by letting the database locate rows via indexed structures.  
- Improve performance on `WHERE`, `JOIN`, `ORDER BY`, `GROUP BY` operations.  
- Enable index‑only scans (covering indexes) that satisfy queries without touching the base table.  
- Trade‑off: extra storage and slower writes.

---

## 13.2 How Indexes Work  
### B‑Tree Indexes  
- Balanced tree structure.  
- Log(N) search, insert, delete.  
- Ideal for range scans (`>=`, `BETWEEN`, `ORDER BY`).  

### Hash Indexes  
- Hash‑based buckets.  
- Very fast equality lookups (`=`).  
- Not suited for range queries.  
- Available in MySQL (MEMORY engine), limited in PostgreSQL (`HASH`).

### Other Structures  
- **GiST/GIN** (PostgreSQL): for full‑text, JSONB, geometric data.  
- **R‑Tree**: spatial and 2D indexes (MySQL, SQLite).

---

## 13.3 Creating Indexes (CREATE INDEX)  
### Syntax  
```sql
CREATE [UNIQUE] INDEX index_name
  ON schema.table_name (column1 [ASC|DESC] [, column2 …])
  [USING {BTREE|HASH|GIN|GIST|…}]
  [WHERE predicate]          -- Partial index (PostgreSQL)
  [WITH (fillfactor = n)]    -- Storage parameters
;
```

### Examples  
```sql
-- Simple B‑Tree
CREATE INDEX idx_orders_date
  ON orders (order_date DESC);

-- Composite index
CREATE INDEX idx_orders_customer_date
  ON orders (customer_id, order_date);

-- Partial index (PostgreSQL)
CREATE INDEX idx_active_customers_email
  ON customers (email)
  WHERE status = 'active';
```

---

## 13.4 Types of Indexes  
| Type                | Purpose / Characteristics                           |
|---------------------|------------------------------------------------------|
| Clustered           | Defines physical row order (SQL Server, MySQL InnoDB) |
| Non‑Clustered       | Separate structure pointing to row locations         |
| Unique              | Enforces uniqueness; implicit index on PK/FK         |
| Composite           | Multi‑column; supports multi‑key filters and orders  |
| Full‑Text           | Tokenizes text for linguistic searches               |

---

## 13.5 Choosing Columns to Index  
- **Selectivity**: High cardinality (many distinct values) benefits most.  
- **Filter Usage**: Columns in `WHERE`, `JOIN`, `ORDER BY`, `GROUP BY`.  
- **Leading Column**: In composite indexes, the first column dictates search patterns.  
- **Write Patterns**: Avoid indexing columns heavily updated or inserted unless critical.  
- **Covering Index**: Include all columns needed by query to avoid lookups (`INCLUDE` in SQL Server, PostgreSQL).

---

## 13.6 Index Maintenance (Rebuilding, Reorganizing)  
- **Fragmentation**: B‑Tree pages split or become sparse over time.  
- **Reorganize**: Online defragmentation (SQL Server: `ALTER INDEX … REORGANIZE`).  
- **Rebuild**: Drops and recreates index (offline/online options).  
- **Statistics**: Keep histograms up to date (`ANALYZE`, `UPDATE STATISTICS`) for accurate plans.

---

## 13.7 Dropping Indexes (DROP INDEX)  
```sql
-- PostgreSQL / MySQL
DROP INDEX IF EXISTS idx_name ON schema.table_name;

-- SQL Server
DROP INDEX schema.table_name.idx_name;
```
- Remove unused or redundant indexes to speed up DML and reduce storage.

---

## 13.8 Understanding Query Execution Plans  
- **EXPLAIN** (MySQL, PostgreSQL) / **EXPLAIN PLAN** (Oracle) / **SET SHOWPLAN_ALL** (SQL Server)  
- Key metrics:  
  - **Seq Scan** vs. **Index Scan**  
  - **Cost Estimates** and **Rows**  
  - **Join Algorithms**: nested‑loop, hash, merge  
- **EXPLAIN ANALYZE**: actual runtime statistics and I/O counts.  
- Use GUI tools (PgAdmin, SSMS, MySQL Workbench) for visual plans.

---

## 13.9 Common Performance Bottlenecks and Optimization Strategies  
- **Full Table Scans**: add or refine indexes; rewrite suboptimal predicates.  
- **Parameter Sniffing**: use local variables or plan guides.  
- **Poor Statistics**: schedule regular updates; use sampling.  
- **Lock Contention**: partition large tables; use row‑versioning.  
- **I/O Hotspots**: relocate index files, use SSDs, tune fillfactor.  
- **Excessive Joins**: denormalize with summary tables or materialized views.

---

## 13.10 Impact of Indexes on DML Operations  
- **INSERT/UPDATE/DELETE**  
  - Each index must be maintained → additional I/O and CPU.  
  - Composite indexes amplify overhead when modifying leading columns.  
- **Batch Loads**: disable non‑essential indexes, load data, then rebuild indexes.  
- **Trade‑offs**: balance read‑heavy vs. write‑heavy workload when choosing index strategy.

---

# Best Practices  
1. Regularly audit index usage (`pg_stat_user_indexes`, DMV in SQL Server).  
2. Drop duplicate or low‑selectivity indexes.  
3. Leverage **covering indexes** for critical queries.  
4. Automate statistics collection and index maintenance tasks.  
5. Monitor execution plans and revise indexes alongside schema changes.  

By thoughtfully designing, maintaining, and tuning indexes, you ensure that your database delivers optimal query performance without compromising write throughput or storage efficiency.