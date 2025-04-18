# Chapter 11 – Subqueries (Nested Queries)

Subqueries—queries nested inside other SQL statements—enable composable, declarative problem‑solving.  They return a result that the outer query consumes as a scalar, a set, or a virtual table.  Understanding their flavors, execution strategies, and performance implications empowers you to write concise yet efficient SQL.

---

## 11.1  Introduction to Subqueries

Concept | Description
--------|------------
Definition | A complete `SELECT` embedded inside another SQL clause (`WHERE`, `SELECT`, `FROM`, etc.).
Result Types | • Scalar (single value)  • Row (single row)  • Table (multi‑row, multi‑col)
Correlation | • Non‑correlated – evaluated once.  • Correlated – re‑evaluated per row of the outer query.
Typical Use‑Cases | Filtering (`WHERE id IN (…)`), on‑the‑fly metrics (`SELECT …, (SELECT …)`), derived tables, existence checks.

Execution Flow  
1. Optimiser flattens / rewrites when possible (e.g., converts subquery → join).  
2. For correlated forms, the engine may use **nested‑loops** with index lookups or transform into semi‑joins / anti‑joins.

---

## 11.2  Subqueries in the `WHERE` Clause

### 11.2.1 Single‑Row (Scalar) Subqueries

```sql
SELECT *
FROM   employees
WHERE  salary > (SELECT AVG(salary) FROM employees);
```

• Must return **exactly one row & one column**—else error.  
• Used with comparison operators `=  <>  >  >=  <  <=`.  
• Planner often rewrites as a window function for better performance.

Common Pitfalls  
- Forgetting to constrain the inner query → “more than one row” error.  
- Scalar subquery evaluated **after** `WHERE` filters but before `GROUP BY`.

### 11.2.2 Multi‑Row Subqueries (`IN`, `ANY`/`SOME`, `ALL`)

```sql
-- IN
SELECT order_id
FROM   orders
WHERE  customer_id IN (SELECT customer_id
                       FROM   vip_customers);

-- ANY / SOME  (same keyword)
WHERE price > ANY (SELECT price FROM competitors WHERE …);

-- ALL
WHERE rating >= ALL (SELECT rating FROM reviews WHERE product_id = p.id);
```

Operator Semantics  
| Operator | Condition Holds When… |
|----------|-----------------------|
| `IN` | Value matches **at least one** row. |
| `ANY / SOME` | Comparison is true for **at least one** row. |
| `ALL` | Comparison is true for **every** row; empty set ⇒ `TRUE`. |

Performance Tips  
- Index the subquery’s output column(s).  
- `IN` with large result sets can be expensive; consider `JOIN` or temporary table.

---

## 11.3  Subqueries in the `SELECT` List (Scalar)

```sql
SELECT c.customer_id,
       c.name,
       (SELECT COUNT(*)         -- scalar subquery
        FROM   orders o
        WHERE  o.customer_id = c.customer_id) AS order_cnt
FROM   customers c;
```

Characteristics  
• Executed **per outer row** (implicitly correlated).  
• Convenient for KPIs but can degenerate into N+1 lookups.  
• Optimiser may transform into `LEFT JOIN` + `GROUP BY` (PostgreSQL: “pull‑up aggregates”).

When to Use  
- Small outer result sets (management dashboards).  
- Situations where a window function can’t express the calculation.

---

## 11.4  Subqueries in the `FROM` Clause (Derived Tables)

```sql
SELECT  dt.customer_id,
        dt.monthly_spend
FROM   (SELECT customer_id,
               SUM(total_amount) AS monthly_spend
        FROM   orders
        WHERE  order_date >= date_trunc('month', CURRENT_DATE)
        GROUP  BY customer_id) AS dt
WHERE  dt.monthly_spend > 1000;
```

Key Points  
• The inner query materialises a **virtual table**; must be aliased.  
• Acts like a one‑off view; scope limited to the statement.  
• Supports indexing via temp files in some engines (`MATERIALIZED` hint in Postgres 14+).  

Comparison with CTE (`WITH`)  
| Aspect | Derived Table | CTE |
|--------|---------------|-----|
| Scope | Only in `FROM` | Any clause after definition |
| Materialisation | Engine‑decided | Inline by default; can force with `MATERIALIZED` |
| Reuse | Single | Multiple references |

---

## 11.5  Correlated Subqueries

### 11.5.1 How They Work

A correlated subquery references columns from its parent query, thus forming a dependency chain.

```sql
SELECT e.*
FROM   employees e
WHERE  salary > (
    SELECT AVG(salary)
    FROM   employees
    WHERE  department_id = e.department_id
);
```

Execution Strategy  
1. For each outer row, plug values into the inner query.  
2. Evaluate inner query—often via index seek.  
3. Compare result to outer predicate.

### 11.5.2 `EXISTS` / `NOT EXISTS`

```sql
-- Semi‑join
SELECT *
FROM   customers c
WHERE  EXISTS (SELECT 1
               FROM   orders o
               WHERE  o.customer_id = c.customer_id);

-- Anti‑join
SELECT *
FROM   products p
WHERE  NOT EXISTS (SELECT 1
                   FROM   order_items oi
                   WHERE  oi.product_id = p.product_id);
```

Behavior  
| Operator | Returns TRUE when… |
|----------|-------------------|
| `EXISTS` | Inner query returns **any** row. |
| `NOT EXISTS` | Inner query returns **no** rows. |

Advantages  
- Short‑circuit evaluation—stops on first match.  
- Handles `NULL`s safely (unlike `NOT IN`).  
- Often optimised to JOIN with semi‑/anti‑join execution nodes.

---

## 11.6  Subqueries vs. Joins — Choosing Wisely

Criterion | Subquery (`IN`, `EXISTS`) | Join
----------|---------------------------|-----
Readability | Closer to English; encapsulates logic | Expands flat row set; clearer when fetching columns from both sides
Performance (Equality) | Planner may auto‑rewrite → join; similar speed | Naturally index‑friendly
Performance (Inequality) | `ALL` / `ANY` may prevent optimisations | Range join often faster (`JOIN … ON a.val > b.val`) with proper indexes
Returning Extra Columns | Requires moving to `SELECT` list with separate subquery | Directly select columns
Null Semantics | `NOT IN` + `NULL` traps | `LEFT JOIN … WHERE b.id IS NULL` is safe

Guidelines  
1. Use `EXISTS / NOT EXISTS` for existence checks—robust and optimisable.  
2. Prefer joins when you need columns from both tables or large set membership checks.  
3. Benchmark: in PostgreSQL `EXPLAIN (ANALYZE, BUFFERS)` reveals if rewrite happenned (`Semi Join`, `Hash Join`).  

---

## Quick Reference – Subquery Patterns

Task | Idiom | Notes
-----|-------|------
Top‑1 per group | `WHERE score = (SELECT MAX(score) …)` | May rewrite to window function.
Filter by set | `col IN (SELECT …)` | If inner set > 100k rows, consider join.
Correlated aggregate | `salary > (SELECT AVG(salary) FROM … WHERE dept_id = outer.dept_id)` | Ensure index on filter columns.
Existence | `WHERE EXISTS (SELECT 1 …)` | Planner uses semi‑join.
Anti‑existence | `WHERE NOT EXISTS (SELECT 1 …)` | NULL‑safe alternative to `NOT IN`.

---

## Performance Checklist

1. **Index inner query columns** (join predicates, filters).  
2. Beware **N+1**: Scalar subqueries in `SELECT` against large outer sets ⇒ rewrite as join.  
3. Enable query planner rewrites (`enable_hashjoin`, `optimizer_switch`) unless you need strict subquery semantics.  
4. In MySQL, avoid `IN (SELECT …)` on older versions (< 8.0.16) due to poor optimisation—rewrite to join.  
5. Use `LATERAL` (PostgreSQL, SQL Server CROSS APPLY) for correlated derived tables that must access outer columns efficiently.

---

By mastering subqueries—scalar, correlated, derived, and existence‑oriented—you gain a versatile toolset to express complex relationships succinctly while still respecting the set‑based ethos of SQL. Continual benchmarking and thoughtful indexing ensure that elegance does not come at the cost of performance.