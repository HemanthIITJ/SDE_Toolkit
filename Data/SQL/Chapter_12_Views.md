# Chapter 12 – Views  

Views are virtual tables defined by queries. They encapsulate complex logic, enforce security, and present consistent interfaces without duplicating data.

---

## 12.1 What Are Views? Virtual Tables  
- A **view** is a named query stored in the database catalog.  
- Acts like a table: you can `SELECT` from it, join it, filter it.  
- **No physical data** stored (except materialized views).  
- Changes in underlying tables are reflected immediately.

---

## 12.2 Creating Views (CREATE VIEW)  
### Generic Syntax  
```sql
CREATE [OR REPLACE] VIEW schema.view_name AS
  SELECT column1, column2, …
  FROM table1
  [JOIN table2 …]
  [WHERE …]
  [WITH [CASCADED | LOCAL] CHECK OPTION];
```

### Key Options  
- `OR REPLACE`: overwrite existing view.  
- `WITH CHECK OPTION`: prevents base‑table changes that violate the view’s `WHERE` clause.  
- Vendor‑specific clauses:  
  - MySQL: `ALGORITHM = MERGE/ TEMPTABLE`  
  - PostgreSQL: `SECURITY INVOKER` / `SECURITY DEFINER`  

### Example  
```sql
CREATE VIEW active_customers AS
SELECT id, name, email
FROM customers
WHERE status = 'active'
WITH LOCAL CHECK OPTION;
```

---

## 12.3 Benefits of Using Views  
- **Simplicity**: Hide complex joins and calculations behind a consistent interface.  
- **Security**: Grant users access to specific columns or rows without exposing base tables.  
- **Consistency**: Centralize business logic (e.g., currency conversion, derived columns).  
- **Maintainability**: Changes to base logic propagate automatically.

---

## 12.4 Querying Data Through Views  
- Use like any table:  
  ```sql
  SELECT name, email FROM active_customers WHERE email LIKE '%@example.com';
  ```  
- Optimizer **inlines** view definition into calling query—no performance penalty if simple.  
- Be cautious with nested or highly complex views; may degrade performance.

---

## 12.5 Updating Data Through Views  
### Updatable Views  
A view is “updatable” if it directly maps to a single base table and meets these criteria:  
- No aggregations (`GROUP BY`, `HAVING`).  
- No `DISTINCT`, `UNION`, `LIMIT`.  
- All non‑nullable columns and PK must be exposed.  
- No subqueries in the select list.

### Restrictions  
- Cannot update columns derived from expressions.  
- `WITH CHECK OPTION` enforces row‑level filters on `INSERT`/`UPDATE`.  

### INSTEAD OF Triggers  
For complex views, use triggers to redirect DML:  
```sql
CREATE VIEW sales_summary AS …
CREATE TRIGGER sales_summary_ins
INSTEAD OF INSERT ON sales_summary
FOR EACH ROW
  INSERT INTO sales(employee_id, amount)
  VALUES (NEW.emp_id, NEW.total);
```

---

## 12.6 Modifying Views (ALTER VIEW)  
- **PostgreSQL/MySQL**: No separate `ALTER VIEW` syntax; use `CREATE OR REPLACE VIEW`.  
- **SQL Server**:  
  ```sql
  ALTER VIEW dbo.active_customers
  AS SELECT id, name, email FROM customers WHERE status='active';
  ```  
- Always include the full new definition; you cannot incrementally add columns.

---

## 12.7 Dropping Views (DROP VIEW)  
```sql
DROP VIEW [IF EXISTS] schema.view_name [, schema.view_name …];
```
- Automatically revokes dependent privileges.  
- In some systems, dropping a view with `CASCADE` also drops dependent views.

---

## 12.8 Materialized Views (Concept and Use Cases)  
- **Definition**: A snapshot of a query persisted on disk.  
- **Benefits**:  
  - Fast reads of precomputed results.  
  - Reduces load on base tables for expensive aggregations.  
- **Refresh Methods**:  
  - **ON COMMIT**: auto‑refresh when base tables change (PostgreSQL).  
  - **ON DEMAND** / **SCHEDULED**: manual or cron‑driven refresh.  
- **Use Cases**:  
  - Data warehousing aggregates (daily summaries).  
  - Denormalized reporting tables.  
  - Caching complex joins for OLAP queries.

### Example (PostgreSQL)  
```sql
CREATE MATERIALIZED VIEW monthly_sales
REFRESH FAST ON COMMIT
AS
SELECT date_trunc('month', order_date) AS month,
       SUM(total) AS total_sales
FROM orders
GROUP BY 1;
```

---

# Best Practices  
- Version‑control all view definitions.  
- Document view purpose and performance characteristics.  
- Limit nesting depth; prefer CTEs for readability.  
- Use `WITH CHECK OPTION` to safeguard updatable views.  
- Evaluate materialized views’ storage and refresh costs against performance gains.