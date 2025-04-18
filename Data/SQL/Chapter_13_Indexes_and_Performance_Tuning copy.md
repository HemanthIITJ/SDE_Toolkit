# Chapter 13 – Indexes & Performance Tuning

Indexes are the cornerstone of high‑performance relational databases.  They transform **O(n)** scans into **O(log n)** or **O(1)** look‑ups, but each index adds storage, write overhead, and maintenance complexity.  This chapter explains how indexes work, how to choose and maintain them, and how to read execution plans to make data‑driven tuning decisions.

---

## 13.1  Why Indexes? — Speeding‑Up Queries

Goal  | Without Index | With Index
------|--------------|-----------
Lookup single row (PK) | Table scan → proportional to table size | B‑Tree seek → < 10 page reads for millions of rows
Join two large tables | Nested‑loop over full scans | Index nested‑loop / hash join → prunes non‑matching rows early
ORDER BY / MIN / MAX | Full sort / scan | Use index order to stream rows

Implications  
• Faster **SELECT** & **MERGE JOIN** operations.  
• Reduced memory + I/O → lower CPU, better concurrency.  
• Trade‑off: slower `INSERT/UPDATE/DELETE`, extra disk space.

---

## 13.2  How Indexes Work

### 13.2.1 B‑Tree (default, general‑purpose)
• Balanced tree; root → leaf path depth ≈ `log_fanout(N)`.  
• Supports equality, range (`<, >, BETWEEN`) and prefix queries.  
• Maintains sorted order ⇒ index‑ordered scans & quick `MIN/MAX`.  

### 13.2.2 Hash
• Key hashed to bucket ⇒ **O(1)** equality lookup.  
• No order ⇒ can’t service range queries or `ORDER BY`.  
• PostgreSQL: not WAL‑logged pre‑10, now safe; MySQL: Memory engine.  

### 13.2.3 Specialised Structures
Type | Engine(s) | Use‑Case
-----|-----------|---------
GiST / GIN | PostgreSQL | Full‑text, arrays, ranges, geometries
R‑Tree | Oracle, SQLite | Spatial rectangles
Bitmap | Oracle, SQL Server | Low‑cardinality columns in DSS
Columnstore | SQL Server, PostgreSQL, MySQL 8 | Analytics, vectorised scans
Inverted / Full‑Text | MySQL FTS, PostgreSQL GIN, Elastic‑compatible | Text search

---

## 13.3  Creating Indexes – `CREATE INDEX`

Generic ANSI
```sql
CREATE [UNIQUE] INDEX idx_name
ON table_name (col1 [ASC|DESC] [, col2 …])
[INCLUDE (non_key_col1, …)]          -- SQL Server, PostgreSQL 14+
[WITH (fillfactor = 80)];            -- engine‑specific options
```

Vendor Highlights  
| Engine | Extras |
|--------|--------|
| PostgreSQL | `CREATE INDEX CONCURRENTLY`, expression indexes (`LOWER(email)`), partial indexes (`WHERE deleted=false`) |
| MySQL | `ALTER TABLE … ADD INDEX` (online if `ALGORITHM=INPLACE`); support for `INVISIBLE` indexes |
| SQL Server | `CREATE INDEX … WITH (ONLINE = ON, DATA_COMPRESSION = PAGE)` |

---

## 13.4  Index Types & Characteristics

| Type | Ordered? | Unique? | Notes |
|------|----------|---------|-------|
| Clustered PK (InnoDB, SQL Server) | Yes | Usually | Table data stored in key order; only **one** per table |
| Non‑Clustered | Yes | Optional | Separate structure; may store PK lookup pointer |
| Composite (Multi‑column) | Yes | Optional | Order matters: `(a, b)` supports `WHERE a = 1 AND b = 2`; not `WHERE b = 2` alone |
| Unique | Yes | Enforces entity integrity, aids optimiser selectivity |
| Partial / Filtered | Yes | Optional | Stores subset of rows; smaller, faster (`WHERE active`) |
| Covering (Included cols) | Yes | Optional | Non‑key columns stored only at leaf → index‑only scans |
| Full‑Text / Inverted | No | N‑gram or token based search |
| Hash | No | Equality only; engine‑specific restrictions |

---

## 13.5  Choosing Columns to Index

Heuristics  
1. **Predicate Frequency**   Columns appearing in `WHERE`, `JOIN ON`, `ORDER BY`, `GROUP BY`.  
2. **Selectivity**   High **cardinality** (> 95 % distinct) for equality indexes; low cardinality → bitmap or skip index.  
3. **Data Distribution**   Skewed columns (Zipf) may need composite with second key to improve fan‑out.  
4. **Composite Ordering**   Place most selective predicate first; keep sort columns next for covering `ORDER BY`.  
5. **Write Penalty**   Limit OLTP tables to ≤ 5 well‑justified indexes; batch/report tables can afford more.  
6. **Storage / Memory**   Index size fits in memory buffer?  SSD vs HDD?  

Decision Matrix  

| Query Pattern | Recommended Index |
|---------------|-------------------|
| `WHERE email = ?` | Unique, single‑col on `email` |
| `WHERE status = ? AND created_at >= ?` | Composite `(status, created_at DESC)` |
| `ORDER BY updated_at DESC LIMIT 50` | `(updated_at DESC)` covering index |
| Full‑text search | Inverted / GIN / FTS |

---

## 13.6  Index Maintenance

Task | Purpose | Command (PostgreSQL example)
-----|---------|------------------------------
Rebuild | Remove bloat, restore physical order | `REINDEX INDEX idx_name;`
Reorganise (defrag) | Light, online compaction (SQL Server) | `ALTER INDEX idx REORGANIZE;`
Statistics Update | Accurate planner estimates | `ANALYZE;` or `UPDATE STATISTICS`
Fillfactor Tuning | Reserve space for HOT updates | `WITH (fillfactor = 70)`
Monitoring | Detect fragmentation, usage | `pg_stat_user_indexes`, `sys.dm_db_index_usage_stats`

Scheduling Guidelines  
• Heavy rebuilds during low‑traffic windows.  
• Automate with cron / SQL Agent / pg_cron.  
• For VLDB, use online rebuild options (`CONCURRENTLY`, `ONLINE=ON`).

---

## 13.7  Dropping Indexes – `DROP INDEX`

```sql
DROP INDEX [CONCURRENTLY] idx_name;          -- PostgreSQL
DROP INDEX table.idx_name;                   -- MySQL
DROP INDEX idx_name ON table WITH (ONLINE = ON); -- SQL Server
```

When to Drop  
• Duplicate / overlapping indexes (`(a)` + `(a, b)` => drop first).  
• Unused for > 30 days (check usage stats).  
• High write penalty; not critical for workload.

---

## 13.8  Understanding Query Execution Plans

Obtain Plan | Command
------------|---------
PostgreSQL | `EXPLAIN [ANALYZE, BUFFERS] <query>;`
MySQL | `EXPLAIN FORMAT=JSON <query>;`
SQL Server | `SET SHOWPLAN_XML ON;`

Key Nodes & Metrics  
Node | Meaning | What to Check
-----|---------|--------------
Seq Scan / Table Scan | Full read | Missing index? small table?  
Index Scan / Seek | Using B‑Tree | Rows / Loops, Index name  
Bitmap Heap Scan | Combined index paths | Recheck conditions count  
Hash Join / Merge Join | Join strategy | Hash size, Sort spills  
Sort | Expensive without LIMIT | Work‑mem spills (`disk:`)  
Cost & Actual Time | Planner estimate vs reality | Large gap ⇒ stats stale  

---

## 13.9  Common Bottlenecks & Optimisation Strategies

Bottleneck | Symptom | Fix
-----------|---------|----
Missing index | High disk I/O, seq scans | Create appropriate index; consider partial/covering
Over‑indexing | Slow writes, bloated storage | Drop redundant indexes
Parameter sniffing (SQL Server) | Fast for some params, slow for others | Use `OPTION (RECOMPILE)` or optimize for typical value
Inefficient predicate (`LIKE '%foo'`) | Can’t seek | Reverse/ trigram index, full‑text, or architectural change
Large sort | Temp file spill | Composite index to supply ordering, increase work memory
Row‑by‑row loops (`N+1`) | High CPU | Rewrite with set logic; joins or window functions

---

## 13.10  Impact of Indexes on DML Operations

Operation | Effect of Indexes
----------|------------------
`INSERT` | Extra page splits, logging; consider disabling non‑essential indexes during bulk load
`UPDATE` | If key changes, old index entry delete + new insert; HOT updates (PostgreSQL) avoid when non‑indexed cols change
`DELETE` | Removes index entries; cascading deletes amplify cost
Bulk Loads | Disable / drop, load, recreate (`COPY`, `BULK INSERT`)
High‑Volume OLTP | Tune `fillfactor`, employ partial indexes to keep hot data small
Locking | Index rebuild can block writers unless online/concurrent options used

Best Practice  
1. Batch DML in transactions ≤ 10 k rows to reduce lock escalation.  
2. During ETL, `DROP INDEX`, load, `CREATE INDEX CONCURRENTLY`.  
3. Monitor write latency after adding new indexes—rollback if unacceptable.

---

## Quick Reference Cheat‑Sheet

Task | Command Snippet
-----|----------------
Create unique composite | `CREATE UNIQUE INDEX ux_user_email ON users (tenant_id, LOWER(email));`
Partial (“active only”) | `CREATE INDEX idx_active ON sessions (user_id) WHERE expires_at > now();`
Covering incl. cols | `CREATE INDEX ix_orders_cover ON orders (customer_id) INCLUDE (status, total);`
Online rebuild SQL Server | `ALTER INDEX ALL ON t REBUILD WITH (ONLINE = ON);`
Explain & run | `EXPLAIN ANALYZE SELECT …;`

---

## Field‑Tested Guidelines

1. **Start with the Query** — profile realistic workload before designing indexes.  
2. **One Problem ‑ One Index** — avoid “kitchen‑sink” composites; match specific patterns.  
3. **Revisit Regularly** — data distribution shifts; monthly index audit pays dividends.  
4. **Measure, Don’t Guess** — rely on `EXPLAIN`, latency histograms, and I/O metrics.  
5. **Automate Maintenance** — integrate rebuild/analyze tasks into DevOps pipelines.  
6. **Educate the Team** — an index added by one developer can hurt another path; code review all DDL.

---

Armed with a solid grasp of index internals and performance diagnostics, you can craft schemas that meet demanding SLAs, scale gracefully with data growth, and keep both query and write latency predictable.