# Chapter 16 – Common Table Expressions (CTEs)  

CTEs provide a way to define named result sets for use within a single SQL statement. They improve readability, modularity, and support recursion.

---

## 16.1 Introduction to CTEs (WITH Clause)  
- **Definition**: A CTE is a temporary, named result set defined at the start of a query using `WITH`.  
- **Basic Syntax**:  
  ```sql
  WITH cte_name (col1, col2, …) AS (
    -- any valid SELECT
    SELECT …
    FROM …
    WHERE …
  )
  SELECT …
  FROM cte_name
  JOIN …
  ```
- CTEs exist only for the duration of the enclosing statement.

---

## 16.2 Benefits of CTEs (Readability, Modularity)  
- **Readability**: Break complex queries into logical building blocks.  
- **Modularity**: Reuse the same subquery multiple times without repeating SQL.  
- **Maintainability**: Easier to debug and refactor than deeply nested subqueries.  
- **Recursion**: Native support for hierarchical/graph queries.

---

## 16.3 Non‑Recursive CTEs  
- Behave like inline views but with a clear name and scope.  
- **Example**: Top 5 customers by revenue.  
  ```sql
  WITH top_customers AS (
    SELECT customer_id, SUM(total_amount) AS revenue
    FROM orders
    GROUP BY customer_id
    ORDER BY revenue DESC
    LIMIT 5
  )
  SELECT c.name, tc.revenue
  FROM top_customers tc
  JOIN customers c ON c.id = tc.customer_id;
  ```

---

## 16.4 Multiple CTEs in a Single Query  
- You can chain multiple CTEs separated by commas.  
- Later CTEs can reference earlier ones.  
- **Example**:  
  ```sql
  WITH
    regional_sales AS (
      SELECT region, SUM(total) AS total_sales
      FROM orders
      GROUP BY region
    ),
    top_regions AS (
      SELECT region
      FROM regional_sales
      WHERE total_sales > 100000
    )
  SELECT o.*
  FROM orders o
  JOIN top_regions tr ON o.region = tr.region;
  ```

---

## 16.5 Recursive CTEs  
Allow a CTE to reference itself, enabling hierarchical or graph traversals.

### 16.5.1 Anchor Member and Recursive Member  
```sql
WITH RECURSIVE hierarchy AS (
  -- Anchor: root nodes
  SELECT id, parent_id, name, 1 AS level
  FROM categories
  WHERE parent_id IS NULL

  UNION ALL

  -- Recursive: join CTE to base table
  SELECT c.id, c.parent_id, c.name, h.level + 1
  FROM categories c
  JOIN hierarchy h ON c.parent_id = h.id
)
SELECT * FROM hierarchy;
```

### 16.5.2 Use Cases  
- **Hierarchical Data**: organizational charts, product BOMs.  
- **Graph Traversal**: finding connected components, shortest paths (subject to performance considerations).

### 16.5.3 Preventing Infinite Recursion  
- Always include a termination condition (`WHERE ...`).  
- Limit depth via a `level` column and filter (e.g., `WHERE level < 10`).  
- Some engines support `MAXRECURSION` (SQL Server) to cap iterations.

---

## 16.6 CTEs vs. Subqueries and Views  

| Aspect              | CTE                          | Subquery                     | View                           |
|---------------------|------------------------------|------------------------------|--------------------------------|
| Scope               | Single statement             | Single clause                | Persisted, schema‑wide         |
| Readability         | High (named blocks)          | Lower (nested)               | High, but separate definition  |
| Recursion           | Supported (`RECURSIVE`)      | Not supported                | Not supported natively         |
| Materialization     | Optimizer may inline or materialize | Usually inlined             | Inlined or materialized depends on engine |
| Maintenance         | Version‑controlled with code | Harder to trace/backref      | Centralized but may lag code   |

**Recommendation**: Use CTEs for complex, modular, or recursive logic within queries; use views for sharing reusable logic across multiple statements.