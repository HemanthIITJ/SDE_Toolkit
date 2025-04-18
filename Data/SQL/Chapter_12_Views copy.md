# Chapter 12 – Views  

Views are virtual—or sometimes materialized—tables that encapsulate SELECT statements behind a table‑like interface.  They deliver abstraction, security, and performance leverage without duplicating business logic across queries.  This chapter dissects creation, maintenance, and advanced usage patterns for both standard and materialized views.

---

## 12.1  What Are Views? (Virtual Tables)  

• Definition: A named, stored SQL query whose result set is exposed as a table.  
• Storage:  
  – Standard views store **only** metadata; result is recomputed at query‑time.  
  – Materialized views persist data to disk (see § 12.8).  
• Schema Object: Lives in the system catalog (`pg_class`, `sys.objects`, `information_schema.views`, etc.).  
• Data Lineage: Columns inherit types, names, nullability from the underlying query.

---

## 12.2  Creating Views – `CREATE VIEW`  

Generic ANSI‑SQL Syntax  
```sql
CREATE [OR REPLACE] VIEW view_name [(col_alias [, …])]
AS
    select_statement
[WITH [CASCADED | LOCAL] CHECK OPTION];        -- row‑level update guard
```

Engine‑Specific Options  
| RDBMS | Extra Clauses | Purpose |
|-------|--------------|---------|
| PostgreSQL | `SECURITY {DEFINER | INVOKER}` | Definer‑rights view (privilege firewall). |
| SQL Server | `WITH SCHEMABINDING` | Locks underlying schema from drop/alter. |
| MySQL | `ALGORITHM = {MERGE | TEMPTABLE}` | Optimizer hint: inline vs. temp table. |
| Oracle | `WITH READ ONLY` | Blocks DML through the view. |

Example – read‑only denormalized customer view  
```sql
CREATE VIEW vw_customer_profile AS
SELECT  c.id       AS customer_id,
        c.name,
        a.city,
        a.state,
        COUNT(o.order_id) AS order_cnt
FROM    customers     c
LEFT JOIN addresses    a ON a.customer_id = c.id
LEFT JOIN orders       o ON o.customer_id = c.id
GROUP  BY c.id, c.name, a.city, a.state
WITH SCHEMABINDING;
```

---

## 12.3  Why Use Views?  (Advantages)  

1. Simplicity & Re‑use  
   • Hide multi‑table joins behind a single object.  
   • Ensure every developer queries the **same** business rules.  

2. Security  
   • Expose only whitelisted columns/rows (RLS complement).  
   • Combine with definer‑rights to grant limited read without direct table access.  

3. Stability / API Layer  
   • Keep legacy reports working even when physical schema evolves (add columns, split tables).  

4. Performance (Indirect)  
   • Centralized, optimized query text → fewer copy‑paste anti‑patterns.  
   • Materialized variant pre‑computes heavy calculations (§ 12.8).  

---

## 12.4  Querying Through Views  

Treat the view as a table:  
```sql
SELECT customer_id, order_cnt
FROM   vw_customer_profile
WHERE  state = 'CA'
ORDER  BY order_cnt DESC
LIMIT  20;
```  

Behind the scenes, planners usually **inline** view SQL into the outer query, enabling:  
• Predicate push‑down (`WHERE state='CA'` dives into underlying tables).  
• Index utilisation on base tables; views themselves have no indexes.  
• Statistics reuse (PostgreSQL 15+ stores per‑view stats if `CREATE STATISTICS` on expression).

---

## 12.5  Updating Data Through Views  

### 12.5.1 Updatable Views — Engine Rules  

| Engine | Simple Projection (1 table) | Join View | Grouped View |
|--------|-----------------------------|-----------|--------------|
| PostgreSQL | Yes | No (except w/ `INSTEAD OF` trigger) | No |
| MySQL | Yes | Partially (if update targets one side of an outer join) | No |
| SQL Server | Yes | Yes *if* unique‑key preserving | No |

Requirements (ANSI):  
• Select list must include **key‑preserving** columns.  
• No `DISTINCT`, `GROUP BY`, set operations, or window functions.  
• `WITH CHECK OPTION` enforces that inserted/updated rows remain visible through the view’s WHERE clause.

```sql
CREATE VIEW active_products AS
SELECT * FROM products
WHERE  discontinued = false
WITH CHECK OPTION;          -- blocks UPDATE that sets discontinued=true
```

### 12.5.2 `INSTEAD OF` Triggers  

When a view is inherently non‑updatable, SQL Server and PostgreSQL allow row‑level triggers:  
```sql
CREATE TRIGGER trg_vw_order_summary
INSTEAD OF INSERT ON vw_order_summary
FOR EACH ROW
EXECUTE FUNCTION sync_order_tables();
```  
Trigger takes full control—manually splits data into normalized tables, logs, etc.

---

## 12.6  Modifying Views – `ALTER VIEW` / `CREATE OR REPLACE`  

• PostgreSQL / Oracle: `CREATE OR REPLACE VIEW vw AS SELECT …;`  
• SQL Server: `ALTER VIEW vw AS SELECT … WITH SCHEMABINDING;`  
• MySQL: `ALTER ALGORITHM=MERGE DEFINER=… VIEW vw AS SELECT …;`  

Notes  
• Column reordering requires explicit list or recreation.  
• After altering base tables, refresh metadata (`sp_refreshview` in SQL Server, `ALTER VIEW …` in Oracle).  

---

## 12.7  Dropping Views – `DROP VIEW`  

```sql
DROP VIEW [IF EXISTS] vw_customer_profile
       [CASCADE];       -- PostgreSQL: also drops dependent objects
```  
• Implicit commit in MySQL & Oracle — schedule during low‑traffic windows.  
• SQL Server: cannot drop schemabound view while referencing tables are altered; drop view first or remove `SCHEMABINDING`.

---

## 12.8  Materialized Views  

Materialized views (MV) cache the result set on disk, trading storage for faster read performance.

### 12.8.1 Creation Syntax (PostgreSQL example)  
```sql
CREATE MATERIALIZED VIEW mv_monthly_sales
AS
SELECT  date_trunc('month', order_date) AS month,
        product_id,
        SUM(quantity)  AS qty,
        SUM(total)     AS revenue
FROM    orders
GROUP  BY 1, 2
WITH NO DATA;              -- postpone initial build
```

### 12.8.2 Refresh Strategies  

| Engine | Manual | Automatic |
|--------|--------|-----------|
| PostgreSQL | `REFRESH MATERIALIZED VIEW [CONCURRENTLY] mv;` | Use cron / pg_cron / triggers |
| Oracle | `DBMS_MVIEW.REFRESH` | `ON COMMIT FAST` or `EVERY '1 HOUR'` |
| SQL Server | `Indexed View` auto‑maintained under restrictions | Triggers manage if DIY |

`CONCURRENTLY` (PostgreSQL) allows reads during refresh; requires unique index.

### 12.8.3 Use‑Cases  

• Heavy aggregations (dashboards, KPIs).  
• Pre‑joined star schemas for BI tools.  
• Snapshotting slowly changing dimensions.  
• Accelerating geographic / GIS spatial joins.

Caveats  
• Refresh lag — data staleness.  
• Extra write I/O during refresh or incremental maintenance.  
• Complex dependency trees → manage with migration tool (Liquibase, Flyway).

---

## Cheat‑Sheet – View Commands  

Operation | Statement
----------|-----------
Create view | `CREATE [OR REPLACE] VIEW v AS SELECT …`
Create updatable view | `… WITH CHECK OPTION`
Alter definition | `CREATE OR REPLACE VIEW …` (or `ALTER VIEW …`)
Drop view | `DROP VIEW [IF EXISTS] v`
Create materialized view | `CREATE MATERIALIZED VIEW mv AS SELECT …`
Refresh MV | `REFRESH MATERIALIZED VIEW [CONCURRENTLY] mv`
Secure view | `CREATE VIEW v SECURITY DEFINER …`

---

## Proven Practices  

1. **Name Predictably** – prefix logical layer (`vw_`, `mv_`) to separate from base tables.  
2. **SCHEMABINDING / SECURITY DEFINER** – lock schema, tighten privileges.  
3. **Don’t Over‑Nest** – deeply nested views confuse optimizers; periodically inline or materialize.  
4. **Monitor Staleness** – expose MV refresh timestamp (`last_refresh`) for downstream users.  
5. **Index Supporting Columns** – underlying tables still drive performance of standard views.  
6. **Document Updatability** – make it clear which views accept DML to avoid runtime surprises.

---

Harnessing both traditional and materialized views lets you craft a clean, secure, and performant data access layer, insulating application code from schema churn while providing targeted acceleration where it matters.