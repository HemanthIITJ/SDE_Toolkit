# Chapter 10 – Grouping Data (`GROUP BY`, `HAVING`)

`GROUP BY` partitions rows into buckets, then aggregate functions condense each bucket into a single result. The optional `HAVING` clause filters these aggregate rows. Correct use unlocks dashboards, cohort analyses, and window‑level summarization without resorting to procedural code.

---

## 10.1  `GROUP BY` — Summarising Data Subsets  

Syntax Skeleton  
```sql
SELECT  group_expr [, …],
        aggregate_fn(expr) AS alias [, …]
FROM    source_tables
[WHERE  pre_filter]
GROUP BY group_expr [, …]
```

Conceptual Steps  
1. FROM / WHERE retrieve base rows.  
2. Planner hashes or sorts rows by `group_expr`.  
3. Each bucket feeds aggregate functions → one output row per group.  

Key Traits  
• If `GROUP BY` is absent, the whole result set is a single group.  
• Rows with `NULL` in grouping columns form their own group.  
• Non‑deterministic expressions (e.g., `random()`) make unsafe group keys—avoid.  

---

## 10.2  Aggregates & `GROUP BY` Interaction  

Aggregate functions (`COUNT`, `SUM`, `AVG`, `MIN`, `MAX`, user‑defined) are evaluated **after** grouping. They have access only to rows within their bucket.

Example  
```sql
SELECT  customer_id,
        COUNT(*)                   AS orders,
        SUM(total_amount)          AS revenue
FROM    orders
GROUP BY customer_id;
```

Execution Plan  
• Hash aggregate (`HashAggregate`) for unsorted inputs;  
• Merge aggregate (`GroupAggregate`) if input arrives ordered on `customer_id`.  

---

## 10.3  Grouping by Multiple Columns  

You may define compound buckets by listing several expressions.

```sql
SELECT  extract(year  FROM order_date) AS yr,
        extract(month FROM order_date) AS mo,
        COUNT(*) AS order_cnt
FROM    orders
GROUP BY yr, mo
ORDER BY yr, mo;
```

Rules  
• Order in `GROUP BY` list does not affect semantics, but affects physical sort plan.  
• For dimensional models, index `(yr, mo)` to enable merge aggregation.  

---

## 10.4  Columns Allowed in `SELECT` with `GROUP BY`  

Engine enforces one of two policies:

| Policy | Requirement | Engines |
|--------|-------------|---------|
| **Strict** (`ONLY_FULL_GROUP_BY`) | Every non‑aggregated column in `SELECT` must appear in `GROUP BY`. | MySQL ≥ 5.7 (default ON), Oracle |
| **Functional‑Dependency Aware** | Non‑group columns allowed if functionally dependent on group keys (superset of PK or unique keys). | PostgreSQL, SQL Server |

Violation yields `GROUP BY` error.

Work‑arounds  
• Wrap in aggregates (`MIN(col)`), or  
• Use `ANY_VALUE(col)` (MySQL 8+) to assert irrelevance, or  
• Decompose query with window functions or sub‑selects.

---

## 10.5  `HAVING` — Filtering Aggregate Groups  

`HAVING` is evaluated *after* aggregates are calculated, enabling conditions on aggregate results.

```sql
SELECT  customer_id,
        COUNT(*) AS orders
FROM    orders
GROUP BY customer_id
HAVING  COUNT(*) >= 5;
```

• Predicate may reference both group keys and aggregate aliases (`orders >= 5`).  
• Adds an additional pass over grouped rows; predicate push‑down is impossible—plan accordingly.

---

## 10.6  `WHERE` vs. `HAVING`  

| Aspect | `WHERE` | `HAVING` |
|--------|---------|----------|
| Evaluation Phase | Before grouping | After grouping |
| Can reference aggregates? | No | Yes |
| Row vs. Group filter | Individual rows | Entire groups |
| Performance Impact | Reduces input size early | Post‑aggregation filter; potentially more work |

Pattern  
1. Put **row‑level criteria** in `WHERE` for maximal pruning.  
2. Reserve `HAVING` strictly for constraints that depend on aggregate output.

---

## 10.7  Logical Execution Order  

1. `FROM` (incl. `JOIN`s, `ON`)  
2. `WHERE`  
3. `GROUP BY`  
4. Aggregate functions (`SELECT` list & internal)  
5. `HAVING`  
6. `SELECT` (projection, scalar expressions)  
7. `ORDER BY`  
8. `LIMIT / OFFSET`  

Understanding this order eliminates “column not found” errors and clarifies why aliases defined in `SELECT` are unusable in `WHERE` but valid in `ORDER BY`.

---

## Performance & Design Guidelines  

• **Indexing**: Align leading index columns with `GROUP BY` keys to favour merge aggregation.  
• **Cardinality Control**: Use rolling‑up hierarchies (`GROUP BY ROLLUP`, `CUBE`) for multi‑level summaries instead of multiple queries.  
• **Derived Tables**: Pre‑aggregate large fact tables hourly/daily into summary tables to avoid re‑scanning billions of rows.  
• **Window vs. Group**: When you need both detail and aggregates, prefer window functions (`COUNT(*) OVER (PARTITION BY …)`) to avoid two scans.  
• **Explain Always**: Validate planned strategy (`HashAggregate` vs. `GroupAggregate`) and memory spilling thresholds (`work_mem`, `hash_area_size`).  

---

### Quick Reference Cheat‑Sheet  

Need | Clause / Technique | Example
-----|--------------------|---------
Group and sum | `GROUP BY` + `SUM()` | `… GROUP BY dept_id`
Filter groups | `HAVING` | `HAVING SUM(sales) > 10000`
Multi‑level roll‑up | `GROUP BY ROLLUP` | `GROUP BY ROLLUP (yr, mo)`
Top‑N per group | `ORDER BY` + `LIMIT` in lateral join / window | see “greatest‑N‑per‑group” pattern
Dense distinct counts | Window instead of re‑join | `COUNT(*) OVER (PARTITION BY …)`

---

Mastering the interplay of `GROUP BY`, aggregates, and `HAVING` transforms raw event streams into actionable KPIs, all while harnessing the set‑processing strengths of the SQL engine.