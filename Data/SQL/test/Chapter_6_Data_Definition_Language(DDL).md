# Chapter 6 – Data Definition Language (DDL)  
DDL instructions **create, change, and remove** database objects (databases, tables, columns, constraints, etc.). Mastery of these statements is foundational for schema design, version‑control automation, and runtime maintenance.

---

## 6.1  Creating Databases – `CREATE DATABASE`

### Core Syntax  
```sql
CREATE DATABASE [IF NOT EXISTS] db_name
  [ [DEFAULT] CHARACTER SET charset_name ]
  [ [DEFAULT] COLLATE collation_name ]
  [ WITH OWNER = user_name ]          -- PostgreSQL
  [ LOG ON [PRIMARY] ( NAME = logical_name, FILENAME = 'path' ) ] -- SQL Server
;
```

### Key Options  
| Option | Purpose | Example |
|--------|---------|---------|
| `CHARACTER SET` / `ENCODING` | Defines default encoding for all objects inside the DB. | `ENCODING 'UTF8'` (PostgreSQL) |
| `COLLATE` | Specifies sort/compare rules. | `COLLATE 'en_US.utf8'` |
| Ownership / Authorization | Determines default privileges. | `WITH OWNER = reporting_user` |
| File, Log, Size settings (Engine‑specific) | Physical placement, initial size, auto‑growth. | SQL Server filegroups |

### Best Practices  
- Standardize on UTF‑8 or equivalent.  
- Use `IF NOT EXISTS` in idempotent deployment scripts.  
- Keep non‑production databases **off** production storage tiers.

---

## 6.2  Creating Tables – `CREATE TABLE`

### Generic Skeleton  
```sql
CREATE TABLE [IF NOT EXISTS] schema.table_name (
    column_name   data_type [column_constraints],
    -- … more columns …
    table_constraints
) [table_options];
```

### 6.2.1 Defining Columns & Data Types  
- Each column needs **name, data type,** and optional constraints.  
- Declarative constraints keep invalid data out *before* it hits the app layer.

### 6.2.2 Common SQL Data Types  
| Category  | Typical Types | Tips |
|-----------|---------------|------|
| Numeric   | `INT`, `SMALLINT`, `BIGINT`, `DECIMAL(p,s)`, `NUMERIC`, `FLOAT`, `REAL` | Use `DECIMAL`/`NUMERIC` for money to avoid float rounding. |
| Character | `CHAR(n)`, `VARCHAR(n)`, `TEXT`, `NVARCHAR(n)` (SQL Server) | `CHAR` = fixed length, `VARCHAR` = variable; size appropriately. |
| Date/Time | `DATE`, `TIME`, `TIMESTAMP`, `DATETIMEOFFSET` | Store UTC; convert in app/UI layers. |
| Boolean   | `BOOLEAN` / `BIT` | In MySQL, `BOOLEAN` ⇒ `TINYINT(1)`. |
| Binary    | `BLOB`, `VARBINARY` | Keep large binaries in object storage if possible. |
| JSON / XML | `JSON`, `JSONB`, `XML` | Index functional expressions for performance. |

### 6.2.3 NULL vs. NOT NULL  
- `NOT NULL` enforces presence; drives storage engine optimizations.  
- Prefer `NOT NULL` with sensible defaults; allow `NULL` only when “unknown” is a valid state.

### 6.2.4 Default Values – `DEFAULT`  
```sql
created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
status     VARCHAR(20) NOT NULL DEFAULT 'pending'
```
- Defaults are evaluated by the RDBMS, guaranteeing consistency across writers.  
- Use deterministic expressions; avoid calling non‑idempotent functions as defaults.

---

## 6.3  Modifying Table Structures – `ALTER TABLE`

### Generic Pattern  
```sql
ALTER TABLE schema.table_name
    action [, action …] ;
```

Common vendor variants:
- PostgreSQL & MySQL support **multiple** comma‑separated actions in one statement.
- SQL Server generally executes one action per `ALTER TABLE`.

### 6.3.1 Adding Columns – `ADD [COLUMN]`
```sql
ALTER TABLE customers
ADD COLUMN loyalty_points INT NOT NULL DEFAULT 0;
```
Consider back‑filling logic for large tables (chunked updates to avoid lock escalation).

### 6.3.2 Modifying Columns – `ALTER COLUMN` / `MODIFY COLUMN`
| Engine | Keyword | Example |
|--------|---------|---------|
| PostgreSQL | `ALTER COLUMN` | `ALTER TABLE t ALTER COLUMN price TYPE NUMERIC(10,2);` |
| MySQL | `MODIFY COLUMN` | `ALTER TABLE t MODIFY COLUMN email VARCHAR(320) NOT NULL;` |
| SQL Server | `ALTER COLUMN` | `ALTER TABLE t ALTER COLUMN email NVARCHAR(320) NOT NULL;` |

Hints  
- Increasing length is usually instant; decreasing or changing type can trigger full‑table rewrite.  
- Always check for dependent indexes / constraints.

### 6.3.3 Renaming Columns
PostgreSQL: `ALTER TABLE t RENAME COLUMN old TO new;`  
MySQL: `ALTER TABLE t CHANGE old new VARCHAR(…) …;`  
SQL Server: `sp_rename 't.old', 'new', 'COLUMN';`

### 6.3.4 Dropping Columns – `DROP [COLUMN]`
```sql
ALTER TABLE orders DROP COLUMN obsolete_flag;
```
- In PostgreSQL ≥ 12 and MySQL, drop is **fast** (metadata only); in earlier versions it rewrites data files.

### 6.3.5 Adding / Dropping Constraints
```sql
-- Add
ALTER TABLE orders
  ADD CONSTRAINT fk_customer
  FOREIGN KEY (customer_id) REFERENCES customers(id);

-- Drop
ALTER TABLE orders
  DROP CONSTRAINT fk_customer;
```
- Name constraints explicitly to simplify future migrations.  
- Index foreign key columns to avoid locking cascades on parent updates/deletes.

---

## 6.4  Deleting Tables – `DROP TABLE`

```sql
DROP TABLE [IF EXISTS] schema.table_name
  [CASCADE | RESTRICT];   -- Syntax varies
```

Implications  
- Removes table definition, data, indexes, constraints.  
- `CASCADE` (PostgreSQL) also drops dependent objects (views, sequences).  
- In MySQL, `DROP TABLE` implicitly commits; wrap destructive DDL in maintenance windows.

---

## 6.5  Deleting Databases – `DROP DATABASE`

```sql
DROP DATABASE [IF EXISTS] db_name;
```

Effects & Safeguards  
- **Irreversible** in most engines. Always require manual confirmation or tier‑3 approval in CI/CD pipelines.  
- Ensure no active connections; PostgreSQL: `REVOKE CONNECT` + terminate sessions, or use `DROP DATABASE ... WITH (FORCE)` (v13+).

---

## 6.6  Temporary Tables

### Scope & Lifetime  
| Type | Scope | Lifetime |
|------|-------|----------|
| Local Temp (`#tmp` – SQL Server, `CREATE TEMP TABLE` – standard) | Current session | Auto‑dropped on disconnect |
| Global Temp (`##tmp` – SQL Server) | All sessions | Dropped when last session ends |
| MySQL `TEMPORARY TABLE` | Current connection | Dropped on connection close |
| PostgreSQL `ON COMMIT DELETE ROWS / PRESERVE ROWS` | Transaction / session | Controlled by `ON COMMIT` clause |

### Usage Patterns  
- Stage intermediate results for complex ETL.  
- Persist per‑session scratch data without polluting the permanent schema.  
- Avoid overuse in OLTP paths; creation incurs metadata locks.

### Example – PostgreSQL  
```sql
CREATE TEMP TABLE stage_orders
ON COMMIT DROP AS
SELECT * FROM orders WHERE order_date = CURRENT_DATE;
```

---

## Quick Reference “Cheat Sheet”

Action | Statement | Minimal Example
-------|-----------|----------------
Create DB | `CREATE DATABASE` | `CREATE DATABASE analytics;`
Drop DB   | `DROP DATABASE`   | `DROP DATABASE analytics;`
Create Table | `CREATE TABLE` | `CREATE TABLE users(id INT PRIMARY KEY, name VARCHAR(50));`
Alter – Add Col | `ALTER TABLE … ADD COLUMN` | `ALTER TABLE users ADD email VARCHAR(320);`
Alter – Change Col | `ALTER TABLE … ALTER/MODIFY COLUMN` | `ALTER TABLE users ALTER COLUMN name TYPE TEXT;`
Drop Table | `DROP TABLE` | `DROP TABLE users;`
Temp Table | `CREATE TEMP TABLE` | `CREATE TEMP TABLE tmp AS SELECT * FROM users;`

---

## Practical Recommendations

1. Version‑control all DDL in migration scripts (Flyway, Liquibase, Alembic, etc.).  
2. For high‑traffic tables, test schema changes on replicas, measure lock times, and use **online** DDL tools (`gh-ost`, `pt-osc`, SQL Server Online Index Rebuild).  
3. Always add explicit constraint names and comments—future you (or SREs) will thank you.  
4. Document *why* a column is nullable or has a default; schema is living documentation.  
5. Use database‑native **roles** and **schemas** to segregate environments (e.g., `auth`, `domain`, `log`).

---

🎯  By mastering the above DDL operations—`CREATE`, `ALTER`, and `DROP`—you control the structural backbone of every relational database. This knowledge enables clean initial schema design, pain‑free migrations, and confident refactoring at any scale.