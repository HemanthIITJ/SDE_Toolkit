# Chapter 15 – Window Functions

Window (a.k.a. analytic) functions extend SQL with the ability to run calculations “across rows … yet keep each row.”  
Unlike `GROUP BY`, they do **not** collapse result sets; instead, they add new computed columns derived from peer rows in a specified window.

---

## 15.1  Why Window Functions?

Problem | Old‑School Solution | Window Solution
--------|--------------------|----------------
Running total | Self‑join or correlated subquery | `SUM(amount) OVER (ORDER BY date)`
Rank within category | Sub‑select + `COUNT(*) < value` | `RANK() OVER (PARTITION BY category ORDER BY score DESC)`
Previous / next row | Self‑join on sequence | `LAG(col) OVER (ORDER BY ts)`

Benefits  
• Declarative & terse.  
• Engine executes in a single pass (often SIMD‑vectorised).  
• Plays well with indexes and parallel plans.

---

## 15.2  The `OVER()` Clause

Syntax  
```sql
func_name (expr [, …]) 
OVER (
      [PARTITION BY col_list]
      [ORDER BY     col_list [ASC|DESC] [NULLS LAST|FIRST]]
      [window_frame]
)
```

Component | Purpose
-----------|--------
PARTITION BY | Defines independent “mini‑tables.” Similar to `GROUP BY` but only for the window function.
ORDER BY     | Defines deterministic row order **inside** each partition; required for ranking, frame‑based aggregates, and LAG/LEAD.
Alias reuse  | `WINDOW w AS (PARTITION BY … ORDER BY …)` then `… OVER w`.

Default Behavior  
• Omitting `PARTITION BY` ⇒ single global partition.  
• Omitting `ORDER BY` ⇒ undefined order (deterministic only if physical plan remains stable).

---

## 15.3  Ranking Functions  

Function | Same Rank for Ties? | Gap After Ties? | Common Use
---------|--------------------|-----------------|------------
`ROW_NUMBER()` | No (unique seq) | n/a | Pagination
`RANK()`       | Yes            | Yes | Competition ranking (1, 2, 2, 4)
`DENSE_RANK()` | Yes            | No  | Dense ranking (1, 2, 2, 3)
`NTILE(n)`     | Buckets n equal‑sized groups | — | Quartiles, deciles

Example  
```sql
SELECT player,
       score,
       RANK() OVER (PARTITION BY league ORDER BY score DESC) AS pos
FROM   matches;
```

---

## 15.4  Aggregate Window Functions  

Any normal aggregate may become a window aggregate.

```sql
SELECT  order_id,
        amount,
        SUM(amount) OVER (PARTITION BY customer_id
                          ORDER BY order_date
                          ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW) AS running_total
FROM    orders;
```

Key Points  
• Still ignores `NULL` like standard aggregate.  
• Frame clause (see § 15.6) changes which rows feed the calculation.  
• No `GROUP BY` needed; both detail and aggregate appear simultaneously.

---

## 15.5  Positional / Navigation Functions  

Function | Purpose | Signature
---------|---------|----------
`LAG(expr [, offset [, default]])`  | Return value `offset` rows **before** current. | Default `offset=1`
`LEAD(expr [, offset [, default]])` | Return value `offset` rows **after** current. |
`FIRST_VALUE(expr)` | First row in frame. |
`LAST_VALUE(expr)`  | Last row in frame (beware frame definition!). |

Example – detect price change:  
```sql
SELECT date,
       price,
       price - LAG(price) OVER (ORDER BY date) AS delta
FROM   daily_prices;
```

---

## 15.6  Window Frames

Frame syntax (after `ORDER BY`):

```
ROWS | RANGE | GROUPS
BETWEEN <start> AND <end>
```

Frame Boundaries  

Keyword | Meaning
--------|--------
`UNBOUNDED PRECEDING` | Start of partition
`n PRECEDING` / `FOLLOWING` | Relative offset
`CURRENT ROW` | Self
`UNBOUNDED FOLLOWING` | End of partition

Common Patterns  

Frame | Use Case
------|---------
`ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW` | Running totals
`ROWS BETWEEN n PRECEDING AND CURRENT ROW` | Moving window (rolling average)
`ROWS BETWEEN 1 PRECEDING AND 1 FOLLOWING` | Centered window smoothing
`RANGE BETWEEN INTERVAL '7 days' PRECEDING AND CURRENT ROW` | Time‑based windows (requires interval‑capable column)

Implementation Nuances  
• `ROWS` counts physical rows (fast, uses row offsets).  
• `RANGE` groups rows with equal ordering values ⇒ may require sort & bound computations.  
• `GROUPS` adds tie‑aware grouping (SQL 2012+).

---

## 15.7  Typical Use‑Cases

1. Running totals / cumulative sums.  
2. Moving averages, rolling standard deviation.  
3. Year‑to‑date (YTD), month‑to‑date (MTD) metrics.  
4. Percentile rankings, quartile bucketing.  
5. De‑duplicating to keep first/last row per group (`ROW_NUMBER()` filter).  
6. Gap & island analysis (`LAG` + conditions).  
7. Top‑N per partition (`RANK()` + `WHERE rank <= n`).  
8. SCD type‑2 change detection (`LEAD(valid_from)`).

---

## Performance Tips

• Index on `(PARTITION BY cols, ORDER BY cols)` to enable **index‑only ordered scans**.  
• Avoid large `RANGE` frames on high‑cardinality floats—engine may materialize partitions.  
• Use `ROWS` frame for deterministic and faster performance when order column is unique/int.  
• In PostgreSQL, watch `work_mem`; window sorts spill if exceeded.  
• Materialize heavy sub‑selects with CTE `MATERIALIZED` (PG 14+) before window step.

---

### Quick Reference – Cheat‑Sheet

Need | Function / Frame
----|-----------------
Row number per partition | `ROW_NUMBER() OVER (PARTITION BY … ORDER BY …)`
Top‑3 per group | `…WHERE ROW_NUMBER() OVER (…) <= 3`
Running balance | `SUM(amount) OVER (ORDER BY id ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW)`
7‑day moving avg | `AVG(val) OVER (ORDER BY ts RANGE BETWEEN INTERVAL '6 days' PRECEDING AND CURRENT ROW)`
Prev / next row | `LAG(col) OVER (ORDER BY ts)` / `LEAD(col) …`

---

Harnessing window functions transforms multi‑step procedural logic into concise, optimizable SQL, yielding cleaner code and faster analytics without sacrificing row‑level detail.