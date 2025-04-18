# Chapter 16 – Common Table Expressions (CTEs)

A Common Table Expression is a named, temporary result set defined with the `WITH` clause and visible only to the statement that declares it.  
CTEs act like *inline, single‑use views*: they make complex queries readable, support recursion, and can replace self‑joins and procedural loops with pure SQL.

---

## 16.1  Introduction — The `WITH` Clause

Generic syntax (ANSI SQL‑99+)

```sql
WITH cte_name [(col_alias [, …])] AS (
    <sub‑query>
)
SELECT …
FROM   cte_name;
```

Lifecycle  
1. Parser registers each CTE name.  
2. Optimizer decides whether to **inline** (treat as macro) or **materialize** (execute once, store temp) depending on hints, reuse count, and engine version.  
3. Scope ends when the outer statement finishes.

---

## 16.2  Why Use CTEs? — Benefits

Benefit | Details
--------|---------
Readability | Breaks monolithic SQL into logical blocks; eliminates deeply nested parentheses.
Modularity | Re‑use intermediate results within the same statement (join, filter, aggregate).
Maintainability | Named sub‑queries describe intent (`active_users`, `sales_last_year`).
Debuggability | Each CTE can be run standalone during troubleshooting.
Recursion Support | The only portable way to express recursive queries (graphs, hierarchies).

---

## 16.3  Non‑Recursive CTEs

Example – top‑selling products last month

```sql
WITH sales_last_month AS (
    SELECT product_id, SUM(quantity) AS qty
    FROM   sales
    WHERE  sale_date >= date_trunc('month', CURRENT_DATE - INTERVAL '1 month')
    GROUP  BY product_id
)
SELECT  p.product_name, s.qty
FROM    sales_last_month s
JOIN    products p USING (product_id)
ORDER BY s.qty DESC
LIMIT 10;
```

Execution notes  
• If `sales_last_month` is referenced once, most engines inline it.  
• PostgreSQL ≥ 14 supports `MATERIALIZED` / `NOT MATERIALIZED` hints:

```sql
WITH sales_last_month AS MATERIALIZED ( … )
```

Use materialization when:  
– CTE referenced > 1 time;  
– underlying query is expensive and small in result size.

---

## 16.4  Multiple CTEs in One Query

Separate comma‑delimited blocks; later CTEs can reference former ones.

```sql
WITH raw AS (SELECT …),
     filtered AS (SELECT … FROM raw WHERE …),
     ranked AS (
         SELECT *,
                ROW_NUMBER() OVER (PARTITION BY category ORDER BY score DESC) AS rn
         FROM   filtered
)
SELECT * FROM ranked WHERE rn = 1;
```

• Order matters—each CTE can only depend on those declared above.  
• Keep naming consistent for readability (`raw_`, `stg_`, `rnk_`).

---

## 16.5  Recursive CTEs

### 16.5.1  Anchor & Recursive Members

```sql
WITH RECURSIVE cte_name AS (
    -- Anchor (base case)
    SELECT start_val …
    UNION ALL
    -- Recursive member (refers to cte_name)
    SELECT next_val …
    FROM   cte_name
    WHERE  <termination_predicate>
)
SELECT … FROM cte_name;
```

Rules  
1. `UNION ALL` (or `UNION`) separates the two members.  
2. Columns in both parts must be type‑compatible & count‑compatible.  
3. The recursive query **must** reference the CTE exactly once.

### 16.5.2  Typical Use‑Cases

Use‑Case | Sketch
---------|-------
Hierarchy path (org chart, category tree) | Find all sub‑ordinates of a manager.
Graph traversal | Breadth‑/depth‑first search on adjacency lists (`edges` table).
Iterative computations | Factorial, Fibonacci, calendar date series generation.
Topological sorts | Ordering tasks by dependency.

Example – organization subtree

```sql
WITH RECURSIVE subordinates AS (
    SELECT id, manager_id, name, 1 AS lvl          -- anchor
    FROM   employees
    WHERE  id = :manager_id

    UNION ALL

    SELECT e.id, e.manager_id, e.name, s.lvl + 1   -- recursive step
    FROM   employees e
    JOIN   subordinates s ON s.id = e.manager_id
)
SELECT * FROM subordinates ORDER BY lvl, name;
```

### 16.5.3  Preventing Infinite Recursion

Strategy | Implementation
---------|---------------
Depth limiter | `WHERE lvl < 100`
Cycle check   | Maintain path set: `AND NOT employee_id = ANY(path)` (PostgreSQL arrays)  
Unique rows   | Use `UNION` (deduplicates) instead of `UNION ALL`; costlier but safe.  
Engine guard  | Most RDBMS default `max_recursion_depth`: SQL Server (`OPTION (MAXRECURSION 32767)`), PostgreSQL (`max_stack_depth`).

---

## 16.6  CTEs vs. Subqueries & Views

Aspect | CTE | Derived Subquery | View/Materialized View
-------|-----|-----------------|-------------------------
Scope | Single statement | Single statement (inline only) | Database object (persisted)
Reusability in statement | Yes (multiple refs) | No (must repeat subquery) | Yes (across sessions)
Recursion | Supported | Not supported | Not supported
Optimization | Inline or materialize (engine decision) | Always inline | Depends on view type (standard vs. materialized)
Permission abstraction | No (inherits outer privileges) | n/a | Yes (definer/invoker)
DDL overhead | None | None | Requires CREATE/DROP

Guidelines  
• Use CTEs for **one‑off, complex queries** and recursive logic.  
• Use views for **shared business logic** across many queries.  
• Inline subqueries suffice for **simple, single‑use** components.

---

### Cheat‑Sheet — CTE Quick Commands

Task | Template
-----|---------
Simple CTE | `WITH cte AS (SELECT …) SELECT … FROM cte;`
Materialize | `WITH cte AS MATERIALIZED (SELECT …) …`
Recursive tree | `WITH RECURSIVE cte AS (SELECT … UNION ALL SELECT … FROM cte …) …`
Multiple CTEs | `WITH first AS (…), second AS (SELECT … FROM first), …`
Depth guard | `WHERE lvl < :max_depth` in recursive member
Limit recursion (SQL Server) | `OPTION (MAXRECURSION 1000);`

---

## Best‑Practice Checklist

1. **Name descriptively** (`daily_sales`, `parents`)—future you will thank you.  
2. **Run CTEs standalone** while debugging.  
3. **Beware performance:**  
   • Avoid unnecessary materialization; add `NOT MATERIALIZED` (PostgreSQL 13+).  
   • Keep anchor query selective to shrink recursion work.  
4. **Enforce termination**—always add depth/cycle checks.  
5. **Monitor plans** (`EXPLAIN`): ensure CTEs are inlined when expected; materialization shows as `CTE Scan`.  
6. **Document recursion limits** and complexity in code comments.  

---

Mastering CTEs equips you with expressive, modular SQL constructs that shrink procedural code and elegantly solve hierarchical or iterative problems directly in the database engine.