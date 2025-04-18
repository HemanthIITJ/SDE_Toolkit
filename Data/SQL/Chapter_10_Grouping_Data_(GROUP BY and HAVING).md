# Chapter 10 – Grouping Data (GROUP BY and HAVING)  
Grouping lets you collapse row‑level data into meaningful aggregates. This chapter covers syntax, rules, filtering, and execution order.

---

## 10.1 The GROUP BY Clause: Summarizing Data Subsets  
- **Purpose**: Partition result set into groups based on one or more columns.  
- **Syntax**:  
  ```sql
  SELECT grouping_column, AGG_FUNC(other_column)
  FROM table
  [WHERE predicate]
  GROUP BY grouping_column;
  ```  
- **Example**: Total sales per region:  
  ```sql
  SELECT region, SUM(sales) AS total_sales
  FROM orders
  GROUP BY region;
  ```

---

## 10.2 Interaction between GROUP BY and Aggregate Functions  
- **Aggregates** (`SUM`, `COUNT`, `AVG`, `MIN`, `MAX`) compute over each group.  
- Non‑aggregated columns in `SELECT` must appear in `GROUP BY`.  
- Engines buffer or stream rows per group; order of grouping has performance implications.

---

## 10.3 Grouping by Multiple Columns  
- Create finer‑grained groups by listing multiple columns:  
  ```sql
  SELECT region, product_category, COUNT(*) AS order_count
  FROM orders
  GROUP BY region, product_category;
  ```  
- Result set gives one row per unique `(region, product_category)` pair.

---

## 10.4 Rules for Columns in SELECT with GROUP BY  
1. **Grouped Columns**: Any column in `GROUP BY` may appear as is.  
2. **Aggregated Columns**: Any column wrapped in an aggregate function.  
3. **No Ungrouped Non‑Aggregates**: Columns neither grouped nor aggregated → SQL error.  
4. **Functional Expressions**: You can group by expressions (e.g., `DATE(order_date)`).

Example of invalid query:  
```sql
SELECT region, order_date, SUM(sales)
FROM orders
GROUP BY region;  
-- Error: order_date is neither grouped nor aggregated
```

---

## 10.5 Filtering Groups (HAVING Clause)  
- **HAVING** applies predicates **after** aggregation and grouping.  
- Filters entire groups, not individual rows.  
- **Syntax**:  
  ```sql
  SELECT dept, AVG(salary) AS avg_sal
  FROM employees
  GROUP BY dept
  HAVING AVG(salary) > 70000;
  ```  
- You can reference grouped columns and aggregates in `HAVING`.

---

## 10.6 Difference between WHERE and HAVING  
| Clause | Applies Before/After GROUP BY | Filters        | Can Use Aggregates?    |
|--------|-------------------------------|----------------|------------------------|
| WHERE  | Before grouping               | Individual rows| No                     |
| HAVING | After grouping                | Aggregated groups| Yes (e.g., `SUM()>…`) |

Flow example:
```sql
SELECT c.region, COUNT(*) AS cnt
FROM customers c
WHERE c.status = 'active'          -- row‑level filter
GROUP BY c.region
HAVING COUNT(*) > 100;             -- group‑level filter
```

---

## 10.7 Order of Execution  
Understanding SQL’s logical processing order clarifies when each clause runs:
1. FROM (and JOINs)  
2. WHERE  
3. GROUP BY  
4. HAVING  
5. SELECT  
6. ORDER BY  
7. LIMIT/OFFSET  

Example walkthrough:
```sql
SELECT region, SUM(sales) AS total
FROM orders
WHERE order_date >= '2024-01-01'   -- 2
GROUP BY region                    -- 3
HAVING SUM(sales) > 10000          -- 4
ORDER BY total DESC                -- 6
LIMIT 5;                            -- 7
```

---

# Best Practices  
- Index grouped columns for large tables.  
- Use functional grouping (e.g., `DATE(timestamp)`) to bucket by period.  
- Avoid `SELECT *` with `GROUP BY`; explicitly list needed columns.  
- Push as many filters into `WHERE` as possible to reduce grouping workload.  
- When grouping by multiple columns, order them from high cardinality to low for readability.