# Chapter 7 – Constraints & Data Integrity  

Relational databases guarantee **ACID** behavior only when the schema enforces rigorous rules on the data it stores. Constraints are those rules. Correctly applied, they:  
• prevent invalid states from arising,  
• accelerate query planning via statistics/cardinality,  
• shift consistency checks out of the application layer and into the engine (single source of truth).

---

## 7.1  Why Data Integrity Matters  
1. Consistency — Every row that reaches permanent storage is immediately valid.  
2. Performance — Constraints drive better execution plans by telling the optimiser what *cannot* exist.  
3. Security — Invalid or orphaned records can leak sensitive info through side channels.  
4. Maintainability — A sound schema reduces defensive coding and special‑case handling.

---

## 7.2  PRIMARY KEY – Entity Integrity  

A primary key uniquely identifies each tuple (row) within its table.  
Syntax (ANSI‑92 core):

```sql
CREATE TABLE devices (
    device_id  SERIAL PRIMARY KEY,
    serial_no  VARCHAR(40) NOT NULL UNIQUE,
    registered TIMESTAMP   NOT NULL
);
```

Properties  
- **Uniqueness** & **NOT NULL** automatically enforced.  
- Implicitly backed by an index (clustered in SQL Server, MySQL InnoDB).  
- Only **one** primary key per table, but it may span multiple columns (composite PK).

Best Practices  
- Prefer short, immutable keys (`INT`, `BIGINT`, `UUID`) to minimise index bloat.  
- Avoid natural keys that can change (`email`, `SSN`)—breaks FK relationships on update.  
- In sharded systems, choose keys amenable to hashing or range partitioning.

---

## 7.3  FOREIGN KEY – Referential Integrity  

Foreign keys tie child rows to parent rows, ensuring graph consistency.

### 7.3.1 `REFERENCES` Clause  
```sql
CREATE TABLE invoices (
    invoice_id   SERIAL PRIMARY KEY,
    customer_id  INT  NOT NULL REFERENCES customers(customer_id),
    issued_on    DATE NOT NULL
);
```
• Column list in `REFERENCES` must match parent PK or UNIQUE key (data type & ordering).  
• Engine auto‑indexes the child column(s) in some RDBMS (e.g., Oracle), but **not** all—create the index manually in PostgreSQL/MySQL for delete/update performance.

### 7.3.2 Cascade Actions  

Action | When Triggered | Effect
-------|----------------|-------
`CASCADE` | Parent row deleted/updated | Same operation ripples to child rows.
`SET NULL` | Parent row deleted/updated | Child FK is set to `NULL` (column must allow nulls).
`SET DEFAULT` | (Vendor‑specific) | FK set to column default.
`RESTRICT` | Attempt before change | Reject operation if dependent children exist (checked *before* execution).
`NO ACTION` | Checked *after* statement | Semantically similar to `RESTRICT` but at different phase (most engines treat them the same).

Example—soft‑delete cascade:

```sql
ALTER TABLE invoices
  ADD CONSTRAINT fk_customer
  FOREIGN KEY (customer_id)
  REFERENCES customers(customer_id)
  ON DELETE SET NULL
  ON UPDATE RESTRICT;
```

Implementation Notes  
- Cascades can fan out; design to avoid surprise mass deletes.  
- Circular cascades prohibited in most engines or require `DEFERRABLE INITIALLY DEFERRED`.  
- Online migration tip: add `NOT VALID` FK in PostgreSQL, back‑fill, then `VALIDATE CONSTRAINT` to avoid full‑table locks.

---

## 7.4  UNIQUE Constraint  

Guarantees that the combination of target columns appears at most once.

```sql
ALTER TABLE users
  ADD CONSTRAINT uq_username UNIQUE (lower(username));
-- Postgres supports functional uniqueness
```

Guidelines  
- Name constraints (`uq_<table>_<cols>`) to ease diff‑based migrations.  
- Multi‑column `UNIQUE` rules supersede `DISTINCT` queries for performance.  
- For partially unique data, use partial indexes (PostgreSQL) or filtered indexes (SQL Server).

---

## 7.5  CHECK Constraint  

Arbitrary boolean predicates enforced per row.

```sql
CREATE TABLE products (
    product_id SERIAL PRIMARY KEY,
    price      NUMERIC(10,2) NOT NULL,
    CHECK (price > 0)
);
```

Advanced Examples  
- Postgres: `CHECK (status IN ('draft','published','archived'))`  
- SQL Server: `CHECK (DATEDIFF(day, created_at, updated_at) >= 0)`  

Performance & Caveats  
- Evaluated on INSERT/UPDATE; not enforced on SELECTs.  
- Complex expressions may disable index‑only scans; keep them deterministic and sargable.  
- Cross‑row or cross‑table checks need triggers or materialized constraints.

---

## 7.6  NOT NULL – Domain Integrity  

Simplest yet most powerful rule: field must contain a value.

```sql
ALTER TABLE payments 
  ALTER COLUMN gateway_txn_id SET NOT NULL;
```

Tips  
- Combine with sensible defaults to avoid double‑write patterns (`created_at TIMESTAMP NOT NULL DEFAULT now()`).  
- For text fields requiring at least one visible char, a mere `NOT NULL` isn’t enough (empty string ≠ NULL)—add `CHECK (length(trim(col)) > 0)`.

---

## 7.7  Naming Conventions  

| Constraint Type | Prefix Example |
|-----------------|----------------|
| Primary Key     | `pk_<table>`              |
| Foreign Key     | `fk_<table>_<ref_table>` |
| Unique          | `uq_<table>_<cols>`      |
| Check           | `ck_<table>_<rule>`      |
| Not‑Null change | Don’t rename; governed by column |

Why Name Explicitly?  
- Predictable diffs in CI migrations.  
- Easier to locate in error messages (`violates unique constraint "uq_users_email"`).  
- Speeds up root‑cause analysis versus engine‑generated gibberish.

---

## 7.8  Enabling / Disabling Constraints  

Operation              | PostgreSQL                                    | SQL Server                         | MySQL (≥8)
-----------------------|-----------------------------------------------|------------------------------------|-----------
Disable FK             | `ALTER TABLE … ALTER CONSTRAINT … NOT VALID`  | `ALTER TABLE … NOCHECK CONSTRAINT` | `SET foreign_key_checks = 0;`
Re‑Enable              | `VALIDATE CONSTRAINT`                         | `CHECK CONSTRAINT`                 | `SET foreign_key_checks = 1;`
Disable All            | n/a (loop plpgsql)                            | `EXEC sp_msforeachtable …`         | session/global var

Use‑Cases  
• Bulk loads / ETL where source is trusted and performance is critical.  
• One‑time data repairs.  
Never leave constraints disabled in production; integrate enable/validate in the same migration.

---

## Quick Reference Cheat‑Sheet  

Purpose | Statement Pattern
--------|------------------
Create PK | `CONSTRAINT pk_<t> PRIMARY KEY (col1 [, col2])`
Create FK | `CONSTRAINT fk_<t>_<p> FOREIGN KEY (…) REFERENCES … (…) [ON DELETE … ON UPDATE …]`
Unique    | `CONSTRAINT uq_<t>_<cols> UNIQUE (col1 [, col2])`
Check     | `CONSTRAINT ck_<t>_<rule> CHECK (expression)`
Set NOT NULL | `ALTER TABLE t ALTER COLUMN c SET NOT NULL`
Disable FK (SQL Server) | `ALTER TABLE t NOCHECK CONSTRAINT fk_name`

---

## Field‑Tested Recommendations  

1. Model every relationship first in an ER diagram, then map to PK/FK.  
2. Apply `NOT NULL` by default; opt‑in to nullable only with documented reason.  
3. Use `CHECK` to encode business rules (price > 0, enum whitelists).  
4. Keep cascade paths shallow; explicit application logic is often safer than multi‑hop cascades.  
5. Protect large deletes with `CHECK (deleted_at IS NULL)` constraints + partial indexes instead of full physical cascade deletes.  
6. Automate constraint/state verification in CI using `pg_assert` / `tSQLt` / db‑unit tests.  

---

By mastering **constraints**, you weaponise the database to police itself, drastically reducing application complexity and runtime defects. Treat the RDBMS as a co‑author of your business logic, and let it veto any attempt to persist broken data.