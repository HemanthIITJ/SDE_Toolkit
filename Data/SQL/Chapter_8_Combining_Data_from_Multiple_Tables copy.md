# Chapter 8 – Combining Data from Multiple Tables (Joins)

Relational design intentionally scatters facts across normalized tables. SQL joins re‑assemble those facts at query time, allowing you to pose questions that span entities (customers ↔ orders, employees ↔ managers, etc.). Mastering join syntax, semantics, and performance trade‑offs is essential for high‑quality analytical and OLTP workloads alike.

---

## 8.1  Why Join?  
• Eliminate duplication: keep *orders* separate from *customers*, yet still report “orders per customer.”  
• Enforce single‑source‑of‑truth: update a customer’s name once; all reports reflect it.  
• Enable rich analytics: trend lines, cohort analysis, graph traversals.  
Rule‑of‑thumb: *If you need columns that live in different tables, you need a join.*

---

## 8.2  Cartesian Product – `CROSS JOIN`

```sql
SELECT *
FROM colors         -- 3 rows (red, green, blue)
CROSS JOIN sizes;   -- 2 rows (S, M)  ⇒ 6‑row result
```

• Produces every possible row combination → `|A| × |B|`.  
• Rarely used by itself; most engines materialize it only when necessary (e.g., generating date or numbers tables).  
• 🔥 *Danger*: Can explode to billions of rows if inputs are unfiltered.

---

## 8.3  Inner Joins – `INNER JOIN` (default `JOIN`)

### 8.3.1 The `ON` Clause (Join Predicate)

```sql
SELECT o.order_id, c.customer_name
FROM   orders  AS o
JOIN   customers AS c
       ON c.customer_id = o.customer_id;
```

• Returns only rows that satisfy the predicate.  
• Predicate types  
  – Equi‑joins (most common, leverage indexes)  
  – Non‑equi joins (`BETWEEN`, `<`, `>`, etc.)  
  – Composite joins (`ON a.id = b.a_id AND a.seq = b.seq`)

### 8.3.2 Joining Multiple Tables

```sql
SELECT o.order_id, c.name, s.status_name
FROM   orders      o
JOIN   customers   c ON c.customer_id   = o.customer_id
JOIN   statuses    s ON s.status_code   = o.status_code
WHERE  o.placed_at >= CURRENT_DATE - INTERVAL '30 days';
```

• Join order in the query does **not** force execution order—planner optimizes.  
• Always alias tables to avoid ambiguity (`o`, `c`, `s`).  
• Keep predicates in the `ON` clause for outer joins; otherwise, they may switch to an inner‑join filter (see §8.4 caveats).

---

## 8.4  Outer Joins – Rows + “Gaps Filled with NULLs”

### 8.4.1 `LEFT [OUTER] JOIN`

```sql
SELECT c.customer_id, c.name, o.order_id
FROM   customers c
LEFT JOIN orders o
       ON o.customer_id = c.customer_id;
```

• Preserves every row from the **left** table (`customers`).  
• If no match in `orders`, right‑hand columns are `NULL`.

### 8.4.2 `RIGHT [OUTER] JOIN`

Same, but preserves the **right** table. Used less frequently; many teams rewrite as `LEFT JOIN` for clarity.

### 8.4.3 `FULL [OUTER] JOIN`

```sql
SELECT a.key, a.val AS val_a, b.val AS val_b
FROM   a
FULL JOIN b USING (key);
```

• Returns union of left + right rows; unmatched columns filled `NULL`.  
• Not available in MySQL ≤ 8.0 (work‑around: `UNION` of two `LEFT JOIN`s).

#### Null‑Extending vs. Filtering

```sql
-- WRONG: converts outer → inner
LEFT JOIN orders o ON o.customer_id = c.customer_id
WHERE o.order_id IS NOT NULL;   -- cancels null‑extension
```

Filter in the `ON` clause when you intend to keep outer semantics.

---

## 8.5  Self Joins (Table to Itself)

```sql
SELECT e.employee_id,
       e.name            AS employee,
       m.name            AS manager
FROM   employees e
LEFT JOIN employees m ON m.employee_id = e.manager_id;
```

Use‑cases  
• Hierarchies (employees, categories)  
• Temporal gaps (find consecutive events)  
• Deduping via anti‑join (`LEFT JOIN … WHERE other.id IS NULL`)

---

## 8.6  Natural Joins – ⚠️ Use with Caution

```sql
SELECT … FROM orders NATURAL JOIN order_items;
```

• Auto‑joins by *all columns with identical names*.  
• Schema‑change fragile—adding a new like‑named column silently alters results.  
Recommendation: **Avoid** in production code; be explicit with `JOIN … ON`.

---

## 8.7  `USING` Clause

Shorthand when join columns share the same name:

```sql
SELECT *
FROM   orders
JOIN   customers USING (customer_id);
```

• Produces one `customer_id` column instead of two.  
• Not compatible with mixed column names (`orders.cust_id = customers.id`).  
• Keep code‑review discipline similar to NATURAL joins—ensure future safety.

---

## 8.8  Choosing the Right Join Type

| Need | Join Type | Notes |
|------|-----------|-------|
| Return only matching rows | `INNER` | Fastest when indexed. |
| Keep all from driver table, mark missing matches | `LEFT` | Analytics, “show 0 orders.” |
| Symmetric gap fill | `FULL` | Slowly changing dimension merges. |
| Generate permutations | `CROSS` | Data scaffolding (#dates × #products). |
| Compare different rows in same table | Self join | Ad‑hoc analytics, hierarchies. |

### Performance Heuristics

1. Index join keys on both sides (foreign‑key & primary‑key columns).  
2. Avoid functions in the join predicate (`ON date_trunc('day', t1.ts) = …`)—pre‑compute or index *keys* instead.  
3. For large equi‑joins → HASH JOIN (PostgreSQL) or GRACE HASH (SQL Server).  
4. For range joins (`<`, `BETWEEN`) → leverage `(col)` + `ORDER BY` to trigger MERGE JOIN.  
5. Materialize derived tables with `WITH` (CTE) only when reuse > 1 or readability wins; many engines inline CTEs.

---

## Cheat‑Sheet – Syntax Quick Hits

Action | Template
-------|----------
Cross product | `SELECT … FROM a CROSS JOIN b;`
Inner join | `… FROM a JOIN b ON a.id = b.a_id;`
Left join | `… FROM a LEFT JOIN b ON …;`
Multiple tables | `a JOIN b ON … JOIN c ON …`
Self join | `a AS x JOIN a AS y ON x.pk = y.fk;`
Join via `USING` | `… JOIN b USING (common_col);`
Avoid accidental filter | Move predicates from `WHERE` to `ON` when outer joining.

---

## Proven Practices

1. **Design first, code later**: draw ER diagrams; confirm cardinalities before writing joins.  
2. **Alias everything** – even two‑table joins – keeps code maintainable.  
3. **Prefer `LEFT JOIN` to `RIGHT JOIN`** for consistency and readability.  
4. **Validate NULL‑handling**: outer‑join outputs often feed into aggregations (`COUNT`, `SUM`)—wrap with `COALESCE`.  
5. **Benchmark** with `EXPLAIN`/`EXPLAIN ANALYZE`; identify missing indexes or bad join order.  
6. **Unit‑test** complex joins: seed minimal fixtures, assert row counts & column values.

---

By mastering these join strategies you unlock the full expressive power of SQL, transforming isolated tables into coherent, high‑performance datasets ready for transactional workloads and analytical storytelling alike.