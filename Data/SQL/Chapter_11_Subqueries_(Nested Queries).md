# Chapter 11 – Subqueries (Nested Queries)  
Subqueries are SQL queries embedded within other queries. They enable dynamic filtering, computation, and table derivation without permanent schema changes.

---

## 11.1 Introduction to Subqueries  
- **Definition**: A SELECT statement nested in `WHERE`, `SELECT`, or `FROM`.  
- **Types**:  
  - **Scalar**: Returns a single value.  
  - **Row**: Returns a single row with multiple columns.  
  - **Table**: Returns multiple rows/columns (used in `FROM`).  

---

## 11.2 Subqueries in the WHERE Clause  
Filter rows based on results of an inner query.

### 11.2.1 Single‑Row (Scalar) Subqueries with Comparison Operators  
- Must return exactly one value (or NULL).  
- Compare with `=, <, >, <=, >=, <>`.  
```sql
SELECT *
FROM products
WHERE price > (
  SELECT AVG(price)
  FROM products
);
```

### 11.2.2 Multi‑Row Subqueries with IN, ANY/SOME, ALL  
- **IN**: matches any value in the list.  
- **ANY/SOME**: true if comparison holds for at least one row.  
- **ALL**: true only if comparison holds for every row.  
```sql
-- IN example
SELECT * 
FROM orders
WHERE customer_id IN (
  SELECT id FROM customers WHERE region = 'EMEA'
);

-- ANY example: price > any discount thresholds
SELECT * 
FROM products
WHERE price > ANY (
  SELECT discount_threshold FROM promotions
);

-- ALL example: price higher than all competitors
SELECT * 
FROM products p
WHERE p.price > ALL (
  SELECT c.price FROM competitor_products c WHERE c.sku = p.sku
);
```

---

## 11.3 Subqueries in the SELECT Clause (Scalar Subqueries)  
Compute per-row values dynamically.
```sql
SELECT o.id,
       o.total,
       (SELECT AVG(total) FROM orders WHERE customer_id = o.customer_id)
         AS avg_customer_order
FROM orders o;
```
- Must return one value per outer row.  
- Can impact performance if not optimized.

---

## 11.4 Subqueries in the FROM Clause (Derived Tables)  
Treat subquery results as a temporary table.
```sql
SELECT dt.region, SUM(dt.sales) AS region_sales
FROM (
  SELECT customer_id, region, total AS sales
  FROM orders JOIN customers USING(customer_id)
) AS dt
GROUP BY dt.region;
```
- Alias (`AS dt`) is mandatory.  
- Enables staging, pivoting, or complex joins without CTEs.

---

## 11.5 Correlated Subqueries  
Subqueries referencing outer query columns; evaluated per outer row.

### 11.5.1 How Correlated Subqueries Work  
- Outer-row values drive inner query execution.  
- Potentially expensive due to row-by-row execution.  
```sql
SELECT e.name, e.salary
FROM employees e
WHERE e.salary > (
  SELECT AVG(salary)
  FROM employees
  WHERE department = e.department
);
```

### 11.5.2 The EXISTS and NOT EXISTS Operators  
- **EXISTS**: true if subquery returns at least one row.  
- Short-circuits on first match; efficient for existence checks.  
```sql
-- List customers with orders
SELECT * 
FROM customers c
WHERE EXISTS (
  SELECT 1 
  FROM orders o 
  WHERE o.customer_id = c.id
);

-- Exclude customers with no orders
SELECT * 
FROM customers c
WHERE NOT EXISTS (
  SELECT 1 
  FROM orders o 
  WHERE o.customer_id = c.id
);
```

---

## 11.6 Subqueries vs. Joins: Performance & Readability  
| Aspect         | Subquery                                  | Join                                   |
|----------------|-------------------------------------------|----------------------------------------|
| Use Case       | Filtering by aggregated or existence logic| Combining related rows                 |
| Readability    | Clear for nested logic                    | More straightforward for flat joins    |
| Performance    | May cause nested loops; optimize with indexes and materialization (CTEs) | Optimizer often uses hash/merge joins; large sets perform well if indexed |
| Maintainability| Encapsulates logic; can become deeply nested | Better for multi-table flattening      |

Best Practices:  
- **Prefer JOINs** for straightforward table combinations.  
- **Use EXISTS** for existence tests—leverages short‑circuiting.  
- **Limit correlated subqueries** on large tables; consider rewriting with JOIN or window functions.  
- **Materialize** complex subqueries using CTEs (`WITH`) for clarity and potential performance gains.