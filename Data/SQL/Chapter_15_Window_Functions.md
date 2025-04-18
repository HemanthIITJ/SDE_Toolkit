# Chapter 15 – Window Functions  

Window functions perform calculations across sets of rows related to the current row without collapsing result rows. They enable advanced analytics—running totals, moving averages, rankings—directly in SQL.

---

## 15.1 Introduction to Window Functions: Calculations Across Row Sets  
- Unlike aggregate functions (which group and reduce rows), window functions **retain** one output row per input row.  
- They “slide” a window (frame) over ordered partitions of the result set.  
- Common use cases: cumulative sums, per‑group rankings, lead/lag analysis.

---

## 15.2 The OVER() Clause (PARTITION BY, ORDER BY)  
Defines the window’s scope and ordering:  
```sql
<window_function>()  
  OVER (
    [PARTITION BY col1 [, col2 …]]   -- splits data into groups
    [ORDER BY col3 [ASC|DESC] …]     -- defines row order within each partition
    [window_frame]                   -- optional frame definition
  )
```

- **PARTITION BY**: resets the window at each distinct key.  
- **ORDER BY**: necessary for functions that depend on sequence (e.g., ranking, cumulative).  
- Without PARTITION BY, the window is the entire result set.

---

## 15.3 Ranking Functions (ROW_NUMBER, RANK, DENSE_RANK, NTILE)  

| Function      | Description                                                  |
|---------------|--------------------------------------------------------------|
| ROW_NUMBER()  | Sequential index per partition, no ties.                     |
| RANK()        | Assigns same rank to ties, gaps after ties.                 |
| DENSE_RANK()  | Same as RANK, but no gaps after ties.                       |
| NTILE(n)      | Divides rows into n approximately equal buckets, assigns bucket number. |

Example – rank sales by region:  
```sql
SELECT
  region,
  sales_rep,
  total_sales,
  ROW_NUMBER() OVER (PARTITION BY region ORDER BY total_sales DESC)    AS rn,
  RANK()      OVER (PARTITION BY region ORDER BY total_sales DESC)     AS rnk,
  DENSE_RANK()OVER (PARTITION BY region ORDER BY total_sales DESC)     AS drnk,
  NTILE(4)    OVER (PARTITION BY region ORDER BY total_sales)          AS quartile
FROM sales;
```

---

## 15.4 Aggregate Window Functions (SUM() OVER, AVG() OVER, etc.)  
Apply traditional aggregates across a window, preserving row details:

```sql
SELECT
  order_date,
  sales_amount,
  SUM(sales_amount)   OVER (ORDER BY order_date)           AS cumulative_sales,
  AVG(sales_amount)   OVER (PARTITION BY region)           AS avg_sales_by_region,
  COUNT(*)            OVER (PARTITION BY region ORDER BY order_date
                            ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW)
                                                      AS running_count
FROM orders;
```

- By default, `SUM() OVER (ORDER BY…)` uses a frame of `RANGE BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW`.

---

## 15.5 Positional Functions (LAG, LEAD, FIRST_VALUE, LAST_VALUE)  

| Function         | Purpose                                           |
|------------------|---------------------------------------------------|
| LAG(expr, offset, default)   | Value of `expr` at *offset* rows before current. |
| LEAD(expr, offset, default)  | Value of `expr` at *offset* rows after current.  |
| FIRST_VALUE(expr)            | First `expr` value in the window frame.         |
| LAST_VALUE(expr)             | Last `expr` value in the window frame.          |

Example – compute prior and next day sales:  
```sql
SELECT
  order_date,
  sales_amount,
  LAG(sales_amount, 1, 0) OVER (ORDER BY order_date) AS prev_day_sales,
  LEAD(sales_amount,1, 0) OVER (ORDER BY order_date) AS next_day_sales
FROM daily_sales;
```

---

## 15.6 Window Frames (ROWS /RANGE BETWEEN)  
Frames refine the set of rows used by the function:

- **ROWS**: counts physical row positions.  
- **RANGE**: groups rows with equivalent ORDER BY values.

Syntax:  
```sql
<func>() OVER (
  [PARTITION BY …]  
  ORDER BY col  
  ROWS|RANGE BETWEEN start_bound AND end_bound
)
```

Bounds:  
- `UNBOUNDED PRECEDING` / `UNBOUNDED FOLLOWING`  
- `CURRENT ROW`  
- `<n> PRECEDING` / `<n> FOLLOWING`

Example – 7‑day moving average:  
```sql
SELECT
  order_date,
  AVG(sales_amount) OVER (
    ORDER BY order_date
    ROWS BETWEEN 6 PRECEDING AND CURRENT ROW
  ) AS moving_avg_7d
FROM daily_sales;
```

---

## 15.7 Use Cases for Window Functions  

- **Running Totals**  
  ```sql
  SUM(amount) OVER (ORDER BY txn_time) AS run_total
  ```  
- **Moving Averages / Rolling Metrics**  
  ```sql
  AVG(score) OVER (
    ORDER BY event_date
    ROWS BETWEEN 2 PRECEDING AND 2 FOLLOWING
  ) AS moving_avg
  ```  
- **Top‑N per Group**  
  ```sql
  ROW_NUMBER() OVER (PARTITION BY dept ORDER BY salary DESC) AS rn
  ```  
- **Year‑over‑Year or Month‑over‑Month Comparisons**  
  ```sql
  LAG(metric, 12) OVER (ORDER BY month) AS last_year_metric
  ```  
- **Percentile and Distribution**  
  ```sql
  CUME_DIST() OVER (PARTITION BY category ORDER BY value) AS cum_dist
  ```  

---

By leveraging window functions, you perform sophisticated analytics—ranking, trend analysis, cohort comparisons—within single SQL statements, eliminating complex self‑joins and subqueries while preserving full row detail.