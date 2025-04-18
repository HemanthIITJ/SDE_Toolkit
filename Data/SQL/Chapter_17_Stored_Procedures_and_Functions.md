# Chapter 17 – Stored Procedures & User‑Defined Functions  

Stored routines (procedures & functions) encapsulate business logic **inside** the RDBMS, enabling atomic, version‑controlled, and reusable operations that execute close to the data.

---

## 17.1  What Are Stored Procedures & Functions?  

Object Type | Returns | Invocation Context | Typical Use
------------|---------|--------------------|------------
Procedure   | 0‑N result sets + OUT params | Stand‑alone call (`CALL`, `EXEC`) | ETL batches, complex multi‑step updates
Function    | Scalar or table value | Inline in queries (`SELECT my_fn(..)`) or `CALL` | Calculated columns, reusable predicates

Both are persisted in the system catalog, compiled (or cached) by the engine, and executed with definable ownership semantics.

---

## 17.2  Why Use Them? — Key Benefits  

• **Performance**   Logic executes server‑side → fewer round‑trips, reduced network latency.  
• **Security**       Definer‑rights routines hide underlying tables; grant EXECUTE only.  
• **Reusability**    One tested routine vs. duplicated SQL across apps.  
• **Maintainability** Centralized versioning; facilitate blue/green data‑layer deploys.  
• **Transaction Scope** Group multiple DML steps in a single ACID unit.

---

## 17.3  Creating Stored Procedures — `CREATE PROCEDURE`

### 17.3.1  Parameters  

Attribute | Variants
----------|----------
Direction | `IN` (default), `OUT`, `INOUT`
Data Type | Any SQL type (`INT`, `TEXT`, `UUID`, composite/ROW types)
Default Values | Supported in PostgreSQL/MySQL 8; not in SQL Server

### 17.3.2  Procedure Body  

• **Imperative blocks**: variables, loops, conditionals.  
• **Embedded SQL**: DML, DDL, dynamic SQL via `EXECUTE`.  
• **Error handling**: `EXCEPTION` (PL/pgSQL), `TRY…CATCH` (T‑SQL), `DECLARE … HANDLER` (MySQL).

#### Cross‑Engine Skeletons  

PostgreSQL (PL/pgSQL)
```sql
CREATE PROCEDURE transfer_funds(IN p_from INT, IN p_to INT, IN p_amt NUMERIC)
LANGUAGE plpgsql AS $$
BEGIN
    PERFORM raise_exception_if(p_amt <= 0, 'amount must be positive');

    UPDATE accounts SET balance = balance - p_amt WHERE id = p_from;
    UPDATE accounts SET balance = balance + p_amt WHERE id = p_to;

    COMMIT;   -- optional; else caller controls txn
END;
$$;
```

SQL Server (T‑SQL)
```sql
CREATE PROCEDURE dbo.transfer_funds
    @from INT,
    @to   INT,
    @amt  DECIMAL(12,2)
AS
BEGIN
    SET NOCOUNT ON;
    BEGIN TRY
        BEGIN TRAN;
        UPDATE accounts SET balance -= @amt WHERE id = @from;
        UPDATE accounts SET balance += @amt WHERE id = @to;
        COMMIT;
    END TRY
    BEGIN CATCH
        ROLLBACK;
        THROW;
    END CATCH
END;
```

MySQL 8 (SQL/PSM)
```sql
DELIMITER //
CREATE PROCEDURE transfer_funds(IN p_from INT, IN p_to INT, IN p_amt DECIMAL(12,2))
BEGIN
    IF p_amt <= 0 THEN SIGNAL SQLSTATE '45000' SET MESSAGE_TEXT='amount must be positive';
    END IF;

    START TRANSACTION;
        UPDATE accounts SET balance = balance - p_amt WHERE id = p_from;
        UPDATE accounts SET balance = balance + p_amt WHERE id = p_to;
    COMMIT;
END //
DELIMITER ;
```

---

## 17.4  Executing Procedures  

Engine | Command
-------|---------
PostgreSQL | `CALL transfer_funds(1, 2, 100);`
SQL Server | `EXEC dbo.transfer_funds @from=1, @to=2, @amt=100;`
MySQL | `CALL transfer_funds(1, 2, 100);`

Returned `OUT` parameters are fetched client‑side; result sets stream like normal queries.

---

## 17.5  Creating User‑Defined Functions — `CREATE FUNCTION`

### 17.5.1  Function Categories  

Type | Returns | Usage
-----|---------|------
Scalar (SRF) | Single value per call | `SELECT price * tax_rate(product_id) …`
Inline Table‑Valued | `TABLE (…)` inlined by optimiser | Replace simple views; parameterized result sets.
Multi‑Statement Table‑Valued | Table variable built in body | ETL or dynamic pivoting.

#### Examples  

Scalar – PostgreSQL
```sql
CREATE FUNCTION tax_rate(product_id INT) RETURNS NUMERIC
LANGUAGE sql IMMUTABLE AS $$
    SELECT COALESCE(r.rate, 0)
    FROM   tax_rules r
    WHERE  r.product_id = tax_rate.product_id
$$;
```

Inline TVF – SQL Server
```sql
CREATE FUNCTION dbo.top_orders(@cust INT)
RETURNS TABLE
RETURN (
    SELECT TOP 10 * FROM orders
    WHERE customer_id = @cust
    ORDER BY total DESC
);
```

---

## 17.6  Using Functions inside SQL  

• In `SELECT`, `WHERE`, `JOIN`, `ORDER BY`.  
• In constraints/triggers (`CHECK (total = calc_total(id))`).  
• As default expressions (`created_at TIMESTAMP DEFAULT utc_now()`).

Optimization Flags  
Attribute | Meaning
----------|---------
`IMMUTABLE` / `STABLE` / `VOLATILE` (PG) | Planner caching & parallelism eligibility.
`SCHEMABINDING` (SQL Server) | Disallows underlying table changes; needed for indexed views.
`DETERMINISTIC` (MySQL) | Required for binary logging in replication.

---

## 17.7  Alter & Drop Routines  

Action | PostgreSQL | SQL Server | MySQL
------|------------|-----------|-------
Modify code | `CREATE OR REPLACE PROCEDURE …` | `ALTER PROCEDURE …` | `ALTER PROCEDURE …`
Rename | `ALTER ROUTINE … RENAME TO …` | `sp_rename` | Drop & recreate
Drop | `DROP PROCEDURE proc_name;` | `DROP PROCEDURE …` | same

Versioning Tip – include `IF EXISTS`/`OR REPLACE` in migration scripts to keep deployments idempotent.

---

## 17.8  Overview of Procedural Languages  

Language | Engine | Key Traits
---------|--------|-----------
T‑SQL | SQL Server / Azure SQL | Rich system catalog API, powerful error handling, CLR integration.
PL/pgSQL | PostgreSQL | Elegant syntax, tight MVCC integration, supports `PERFORM`, `RETURN QUERY`.
PL/SQL | Oracle, Yugabyte | Advanced bulk processing (`FORALL`), associative arrays, exception hierarchy.
SQL/PSM | MySQL, MariaDB | ANSI procedural extension, `DECLARE … HANDLER`.
PL/python, PL/v8, etc. | PostgreSQL extensions | Write routines in Python, JavaScript, etc.—but watch SECURITY.

---

### Cheat‑Sheet — Routine Metadata Queries  

Engine | List Routines | Source Code Column
-------|--------------|-------------------
PostgreSQL | `pg_proc`, `\df+` psql | `prosrc`
SQL Server | `sys.procedures`, `sys.sql_modules` | `definition`
MySQL | `information_schema.routines` | `ROUTINE_DEFINITION`

---

## Best‑Practice Checklist  

1. **Keep logic set‑based**; avoid row‑by‑row cursors unless unavoidable.  
2. **Pin privilege boundary** – GRANT `EXECUTE` on routine, revoke base table access.  
3. **Document volatility & side‑effects** for functions (`IMMUTABLE` when possible).  
4. **Unit‑test** routines with fixtures (pgTAP, tSQLt, MySQL Unit).  
5. **Version‑control** source alongside application code; generate migration diff scripts.  
6. **Guard against silent failures**: implement robust error handling and raise informative messages.  
7. **Benchmark**: Measure execution plans inside routines (`SET SHOWPLAN` / `EXPLAIN`). Refactor hot paths into plain SQL if faster.

---

Stored procedures and functions, when crafted with clear contracts, performance awareness, and proper privilege hygiene, become a powerful layer that unifies business logic, boosts throughput, and simplifies application codebases.