# Chapter 7 – Constraints and Data Integrity  
Constraints enforce rules at the database level, ensuring data is accurate, consistent, and reliable. They form the backbone of relational schema design and guard against invalid or orphaned data.

---

## 7.1 Importance of Data Integrity  
- **Accuracy**: Prevents invalid data entry.  
- **Consistency**: Ensures related tables stay synchronized.  
- **Reliability**: Guarantees business rules are enforced at the source.  
- **Maintainability**: Self‑documenting schema reduces application logic complexity.

---

## 7.2 PRIMARY KEY Constraint (Entity Integrity)  
- Uniquely identifies each row in a table.  
- Implies `NOT NULL` and `UNIQUE`.  
- Can be **single‑column** or **composite**.

Syntax Example:  
```sql
CREATE TABLE users (
  user_id   INT         PRIMARY KEY,
  username  VARCHAR(50) NOT NULL
);
```

Composite PK:  
```sql
CREATE TABLE order_items (
  order_id   INT,
  product_id INT,
  quantity   INT,
  PRIMARY KEY (order_id, product_id)
);
```

Best Practices:  
- Use surrogate keys (e.g., serial, UUID) for stability.  
- Avoid composite PKs unless natural association demands it.

---

## 7.3 FOREIGN KEY Constraint (Referential Integrity)  
Links child rows to parent rows, preventing orphan records.

Syntax:  
```sql
CREATE TABLE orders (
  order_id   INT         PRIMARY KEY,
  customer_id INT        NOT NULL,
  CONSTRAINT fk_orders_customer
    FOREIGN KEY (customer_id)
    REFERENCES customers (id)
    ON DELETE RESTRICT
    ON UPDATE CASCADE
);
```

### 7.3.1 REFERENCES Clause  
- Specifies parent table and column(s).  
- Parent columns must be indexed (typically a PK or UNIQUE).  

### 7.3.2 ON DELETE / ON UPDATE Actions  
| Action      | ON DELETE Behavior             | ON UPDATE Behavior              |
|-------------|--------------------------------|---------------------------------|
| CASCADE     | Delete child rows automatically | Update child foreign keys       |
| SET NULL    | Set foreign key to NULL         | Set FK to NULL                  |
| SET DEFAULT | Set FK to its `DEFAULT` value   | Set FK to default               |
| RESTRICT    | Prevent delete if children exist | Prevent update if referenced    |
| NO ACTION   | Similar to RESTRICT (check deferred in some DBs) |

Choose cascading rules to match business workflows (e.g., invoice line‑items deleted with invoice).

---

## 7.4 UNIQUE Constraint  
Ensures all values in a column (or column group) are distinct.

```sql
CREATE TABLE products (
  sku      VARCHAR(20) UNIQUE,
  name     VARCHAR(100) NOT NULL
);
-- Or as table constraint:
ALTER TABLE products
  ADD CONSTRAINT uq_products_name
    UNIQUE (name);
```

Notes:  
- Multiple `NULL` values allowed in most RDBMS (SQL Server is exception).  
- Index backing the constraint improves lookup speed.

---

## 7.5 CHECK Constraint  
Enforces custom Boolean expressions on column values.

```sql
CREATE TABLE employees (
  emp_id       INT    PRIMARY KEY,
  salary       DECIMAL(10,2) NOT NULL,
  department   VARCHAR(50)    NOT NULL,
  CONSTRAINT chk_salary_positive
    CHECK (salary > 0),
  CONSTRAINT chk_dept_valid
    CHECK (department IN ('HR','Engineering','Sales'))
);
```

Guidelines:  
- Keep expressions simple to avoid performance impact.  
- For complex rules, consider triggers or application logic.

---

## 7.6 NOT NULL Constraint (Domain Integrity)  
Prevents `NULL` entries, enforcing that a column must always contain a value.

```sql
CREATE TABLE sessions (
  session_id    UUID        PRIMARY KEY,
  user_id       INT         NOT NULL,
  login_time    TIMESTAMP   NOT NULL
);
```

Best Practices:  
- Default to `NOT NULL` unless “unknown” is a legitimate state.  
- Combine with `DEFAULT` for seamless inserts.

---

## 7.7 Naming Constraints  
Explicit naming simplifies migrations and error diagnostics.

Convention:  
- PK: `pk_<table>`  
- FK: `fk_<child>_<parent>`  
- UNIQUE: `uq_<table>_<column(s)>`  
- CHECK: `chk_<table>_<rule>`  

Example:  
```sql
ALTER TABLE orders
  ADD CONSTRAINT fk_orders_customer
    FOREIGN KEY (customer_id) REFERENCES customers(id);
```

---

## 7.8 Enabling and Disabling Constraints  
### PostgreSQL  
```sql
ALTER TABLE table_name DISABLE TRIGGER ALL;  -- disables FK & check
ALTER TABLE table_name ENABLE TRIGGER ALL;
```
### SQL Server  
```sql
ALTER TABLE table_name NOCHECK CONSTRAINT ALL;
ALTER TABLE table_name WITH CHECK CHECK CONSTRAINT ALL;
```
### MySQL  
```sql
SET FOREIGN_KEY_CHECKS = 0;  -- global toggle
SET FOREIGN_KEY_CHECKS = 1;
```

Use cases:  
- Bulk data loads (disable, load, re-enable, then validate).  
- Schema changes that temporarily violate cross‑table rules.

---

# Quick Reference  

Constraint Type | Enforced Rule            | Syntax (Column-Level)  
--------------- | ------------------------ | ----------------------  
PRIMARY KEY     | Uniqueness + NOT NULL    | `col INT PRIMARY KEY`  
FOREIGN KEY     | Referential integrity     | `col INT REFERENCES t(id)`  
UNIQUE          | Unique values            | `col VARCHAR(…) UNIQUE`  
CHECK           | Boolean expression       | `col INT CHECK (col > 0)`  
NOT NULL        | Non‑NULL values only     | `col VARCHAR(…) NOT NULL`  

---

By rigorously applying these constraints, you embed business rules directly in your schema, ensuring consistent, self‑validating data across all applications and services.