# Chapter 21 – Database Design & Normalization

Well‑structured schemas prevent data anomalies, scale gracefully, and keep query logic intuitive. This chapter synthesizes modelling workflow, formal normal forms, and pragmatic trade‑offs you’ll face in real systems.

---

## 21.1  Principles of Good Database Design

Principle | Why It Matters | Manifestation
----------|---------------|---------------
Single Source of Truth | Eliminates conflicting facts | Each fact stored in exactly one place
Integrity First | Protects data quality | PK/FK, CHECK, UNIQUE, NOT NULL
Clear Semantics | Low cognitive load | Descriptive names, explicit domains
Scalability | Handles growing volume & traffic | Proper indexing, partitioning options
Performance Balance | Reads vs. writes trade‑offs | Selective denormalization, materialized views
Maintainability | Smooth evolution | Migration scripts, version‑controlled DDL

---

## 21.2  Data‑Modeling Stages

Stage | Deliverable | Tools / Techniques
------|-------------|-------------------
Conceptual | High‑level domain map (entities & relationships) | Whiteboard, UML class diagram
Logical | Attribute lists, PK/FK, 3NF compliance | ERD software (Draw.io, Vertabelo)
Physical | Engine‑specific DDL, indexes, partitioning | SQL DDL scripts, pgModeler, SQL Server SSDT

Transition Flow  
Conceptual → Logical (apply normalization, decide keys) → Physical (choose data types, add constraints, performance tweaks).

---

## 21.3  Entity‑Relationship Diagrams (ERDs)

Element | Symbol | Notes
--------|--------|------
Entity | Rectangle | Table‑to‑be
Attribute | Oval (crow’s foot) | Mark PK with underline; FK with “#”
Relationship | Diamond / line | Cardinality (1:1, 1:N, N:M)

Many‑to‑many turns into associative entity (junction table) during logical design.

---

## 21.4  Normalization — Minimizing Redundancy

Normalization decomposes tables so that *each fact lives once*; resolves update, insert, and delete anomalies.

### 21.4.1 First Normal Form (1NF)

Rule | “No repeating groups, each intersection scalar.”
Fix | Split multivalued columns into child table or separate rows.

Example (anti‑pattern)  

| order_id | product_ids |
|----------|-------------|
| 42       | 3,5,9       |

✓ 1NF Version: `order_items(order_id, product_id)`.

---

### 21.4.2 Second Normal Form (2NF)

Rule | 1NF **AND** every non‑key attribute depends on **whole** composite PK (no partial dependency).
Applies | Tables with composite PKs.
Fix | Move partially‑dependent columns to new table.

Example  

```
order_items(order_id, product_id, product_name)  --  product_name depends only on product_id
```

→ Split `products(product_id, product_name)`; keep FK.

---

### 21.4.3 Third Normal Form (3NF)

Rule | 2NF **AND** no transitive dependency (non‑key → non‑key → key).
Fix | Extract attributes into new table keyed by the determinant.

Example  

| employee_id | dept_id | dept_location |
|-------------|---------|---------------|

`dept_location` transitively depends on `employee_id` via `dept_id`.

→ Create `departments(dept_id, dept_location)`.

---

### 21.4.4 Boyce‑Codd Normal Form (BCNF)

Stronger 3NF: **every determinant is a candidate key**.  
Handles edge cases where overlapping composite keys still allow anomalies.

Example  

`course_schedule(course_id, instructor_id, room)` with business rule “room uniquely determined by instructor”.  
`instructor_id → room` violates BCNF if `instructor_id` is **not** key.  
Resolve: split to `instructor_room(instructor_id, room)`.

---

## 21.5  Denormalization — Informed Compromise

When to Consider | Techniques | Risk Mitigation
----------------|------------|-----------------
Heavy read join hotspots | Add redundant foreign column or summary table | Use triggers/materialized view to sync
Static reference data | Inline code lists in fact table | Periodic consistency checks
Time‑series roll‑ups | Store daily aggregates | Keep raw data for back‑fill
Distributed systems (sharding) | Embed child data to avoid cross‑shard joins | Versioned writes, background re‑hydration

Always **measure** before denormalizing and document why + how consistency is preserved.

---

## 21.6  Choosing Data Types & Keys

Topic | Guidelines
------|-----------
Primary Keys | • Surrogate integer (`BIGINT` identity/sequence) → performant joins.  • Natural key only if small, immutable (`ISO country code`).  • Never expose auto‑increment IDs externally—use UUID if guessability is a concern.
Foreign Keys | Mirror parent PK exactly in type/length; index child column for cascade performance.
Data Types | • Numeric money → `DECIMAL(19,4)` to avoid float rounding.  • Choose smallest integer that fits 5‑year growth (storage & index efficiency).  • Fixed‑length codes → `CHAR(n)`; variable text → `VARCHAR(n)` or `TEXT`.  • Timestamps in UTC, `TIMESTAMP WITH TIME ZONE` when offset matters.
Constraints | Declare `NOT NULL` by default; annotate nullable columns with comment on business meaning.

---

### Quick Reference – Normal Form Decision Tree

1. Repeating columns? → break: **1NF**.  
2. Composite key? non‑key depends on subset? → break: **2NF**.  
3. Non‑key depends on another non‑key? → break: **3NF**.  
4. Any determinant not a key? → break: **BCNF**.  

---

## Best‑Practice Checklist

1. **Model → Normalize → Benchmark → Denormalize (if justified)**.  
2. **Start with ERD**, validate with domain experts before generating DDL.  
3. **Name consistently** (`snake_case`, singular tables, meaningful FK names).  
4. **Enforce via constraints**—don’t rely on application validation alone.  
5. **Use surrogate PK + alternate UNIQUE** when natural candidate exists; eases future schema changes.  
6. **Document assumptions** (cardinality, invariants) in schema comments and migrations.  
7. Review design whenever cardinality or query patterns shift (feature flags, new reports).  

---

By grounding design in normalization theory yet pragmatically bending rules when performance dictates, you craft schemas that are both robust and responsive to real‑world workloads.