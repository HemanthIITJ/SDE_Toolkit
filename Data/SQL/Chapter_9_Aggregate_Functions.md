# Chapter 9 – Aggregate Functions  
Aggregate functions compute summary values over multiple rows. They’re essential for reporting, analytics, and deriving key metrics from relational data.

---

## 9.1 Introduction to Aggregation  
- Operate on a set of rows and return a single scalar.  
- Commonly used with `GROUP BY` to produce grouped summaries:  
  ```sql
  SELECT department, AVG(salary)
  FROM employees
  GROUP BY department;
  ```  
- Ignore non‑grouped columns unless wrapped in aggregates.

---

## 9.2 COUNT(): Counting Rows or Non‑Null Values  
### Syntax Variants  
- `COUNT(*)` → counts all rows, including NULLs.  
- `COUNT(column)` → counts non‑NULL values only.  
- `COUNT(DISTINCT column)` → counts unique non‑NULL values.

### Examples  
```sql
-- Total rows in orders
SELECT COUNT(*) FROM orders;

-- Number of orders with non-null shipped_date
SELECT COUNT(shipped_date) FROM orders;

-- Distinct customers who placed orders
SELECT COUNT(DISTINCT customer_id) FROM orders;
```

### Notes  
- `COUNT(*)` is optimizable by most engines (metadata only).  
- Prefer `COUNT(*)` for row counts; use `COUNT(col)` when filtering NULLs.

---

## 9.3 SUM(): Summing Numeric Values  
### Syntax  
```sql
SUM([DISTINCT] numeric_expression)
```

### Examples  
```sql
-- Total revenue
SELECT SUM(total_amount) FROM orders;

-- Sum of unique discounts applied
SELECT SUM(DISTINCT discount_pct) FROM invoices;
```

### Notes  
- Returns `NULL` if all inputs are `NULL`; wrap in `COALESCE(SUM(...), 0)` to default zero.  
- Precision: watch `DECIMAL(p,s)` vs. floating types to avoid rounding errors.

---

## 9.4 AVG(): Calculating the Average  
### Syntax  
```sql
AVG([DISTINCT] numeric_expression)
```

### Examples  
```sql
-- Average order value
SELECT AVG(total_amount) FROM orders;

-- Average unique discount percentage
SELECT AVG(DISTINCT discount_pct) FROM invoices;
```

### Notes  
- Excludes `NULL` values automatically.  
- Returns same data type as `SUM()/COUNT()`; cast as needed for scale.

---

## 9.5 MIN(): Finding the Minimum Value  
### Syntax  
```sql
MIN(expression)
```

### Examples  
```sql
-- Earliest order date
SELECT MIN(order_date) FROM orders;

-- Lowest product price
SELECT MIN(price) FROM products;
```

### Notes  
- Works on numeric, date/time, and string types (lexicographical for strings).  
- Ignores `NULL` values.

---

## 9.6 MAX(): Finding the Maximum Value  
### Syntax  
```sql
MAX(expression)
```

### Examples  
```sql
-- Latest login timestamp
SELECT MAX(login_time) FROM sessions;

-- Highest employee salary
SELECT MAX(salary) FROM employees;
```

### Notes  
- Parallel to `MIN()`, also ignores `NULL`.  
- For strings, returns lexicographically greatest value.

---

## 9.7 Using Aggregate Functions with DISTINCT  
- `DISTINCT` inside aggregates removes duplicates before computation.  
- Supported by `COUNT()`, `SUM()`, `AVG()`.  
- Example:  
  ```sql
  SELECT AVG(DISTINCT rating) AS avg_unique_rating
  FROM reviews;
  ```

---

## 9.8 Handling NULL Values in Aggregations  
- Aggregates ignore `NULL` inputs except `COUNT(*)`.  
- To include `NULL` as zero in sums/averages:  
  ```sql
  SELECT SUM(COALESCE(quantity, 0)) FROM order_items;
  ```  
- When all rows are `NULL`, `SUM()`/`AVG()`/`MIN()`/`MAX()` return `NULL`.

---

# Best Practices  
- Always pair aggregates with appropriate `GROUP BY`.  
- Use `COALESCE` to provide defaults for empty sets.  
- Choose precise data types (e.g., `DECIMAL`) for financial aggregates.  
- Leverage engine optimizations (`COUNT(*)` vs. `COUNT(1)`).  
- Avoid `DISTINCT` unless necessary—its sorting overhead can impact performance.