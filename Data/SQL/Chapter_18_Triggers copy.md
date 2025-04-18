# Chapter 18 – Triggers  

A trigger is a database object that *fires* automatically in response to DML events (`INSERT`, `UPDATE`, `DELETE`).  
Executed inside the same transaction as the triggering statement, it can read/modify data, call procedures, or raise errors—making it a powerful, but double‑edged, mechanism for enforcing rules and side‑effects.

---

## 18.1  What Do Triggers Do?  

• Inject logic **without** changing application code.  
• Guarantee actions at the data layer (auditing, derived‑column updates).  
• Operate under the same ACID scope → rollback if the base statement fails.  

Caveat: Hidden execution flow can obscure behavior and hurt performance—use judiciously.

---

## 18.2  Creating Triggers — Core Syntax  

| Engine         | Skeleton |
|----------------|----------|
| PostgreSQL     | ```CREATE TRIGGER trg_name BEFORE INSERT ON tbl FOR EACH ROW EXECUTE FUNCTION fn_name();``` |
| SQL Server     | ```CREATE TRIGGER trg_name ON tbl AFTER INSERT, UPDATE AS BEGIN … END;``` |
| MySQL 8        | ```CREATE TRIGGER trg_name BEFORE UPDATE ON tbl FOR EACH ROW …``` |

Parameters  
• Timing – `BEFORE`, `AFTER`, `INSTEAD OF`.  
• Event – `INSERT`, `UPDATE [OF col_list]`, `DELETE`.  
• Granularity – `FOR EACH ROW` (row‑level) vs. `FOR EACH STATEMENT` (statement‑level; PG/Oracle only).  
• Body language – PL/pgSQL, T‑SQL, SQL/PSM, PL/SQL, etc.

---

## 18.3  Trigger Events  

Event | Fires When
------|-----------
`INSERT` | New row(s) added (`INSERT …` / bulk load).
`UPDATE` | Any column changes; optionally restricted to specific columns.
`DELETE` | Row(s) removed (`DELETE`, cascade, truncation exceptions).

Multiple events can be combined (SQL Server, Oracle).

---

## 18.4  Trigger Timing  

Timing | Purpose | Notes
-------|---------|------
`BEFORE` | Validate or transform data **before** it hits table. | Can modify `NEW.*` values (except MySQL for PK/AI fields).  
`AFTER` | Side‑effects that rely on committed row state (audit, logging). | Cannot influence DML outcome.  
`INSTEAD OF` | Applied to views to make them pseudo‑updatable. | Body responsible for performing equivalent DML on base tables.

Execution order (per engine) matters when multiple triggers of same timing exist; name triggers with sequence numbers, or use `FOLLOWS` / `PRECEDES` (MySQL 8.0.19+).

---

## 18.5  Row‑Level vs. Statement‑Level  

| Aspect | Row‑Level | Statement‑Level |
|--------|-----------|-----------------|
| Invocation | Once per affected **row** | Once per DML statement |
| Access | `OLD`/`NEW` (row vars) | No per‑row vars—use set operations |
| Use‑Cases | Complex per‑row validation, cascading updates | Audit log header, summary counters |

PostgreSQL default is row‑level; specify `FOR EACH STATEMENT` for statement triggers.

---

## 18.6  Accessing Transition Data  

Engine | NEW / OLD Row | Set‑Based Pseudo‑Tables
-------|---------------|-------------------------
PostgreSQL | `NEW.col`, `OLD.col` | `REFERENCING NEW TABLE AS nt` (PG 10+) |
SQL Server | — | `inserted`, `deleted` virtual tables |
MySQL | `NEW.col`, `OLD.col` (row‑level only) | None |

Example – audit in PostgreSQL:

```sql
CREATE TABLE account_audit(
    id          BIGSERIAL,
    account_id  INT,
    old_balance NUMERIC,
    new_balance NUMERIC,
    changed_at  TIMESTAMP DEFAULT now()
);

CREATE OR REPLACE FUNCTION audit_balance() RETURNS TRIGGER AS $$
BEGIN
    IF NEW.balance <> OLD.balance THEN
        INSERT INTO account_audit(account_id, old_balance, new_balance)
        VALUES (OLD.id, OLD.balance, NEW.balance);
    END IF;
    RETURN NEW;
END $$ LANGUAGE plpgsql;

CREATE TRIGGER trg_balance_audit
BEFORE UPDATE OF balance ON accounts
FOR EACH ROW EXECUTE FUNCTION audit_balance();
```

---

## 18.7  Common Use Cases  

Use Case | Sketch
---------|-------
Change auditing | Insert into history table in `AFTER INSERT/UPDATE/DELETE`.
Complex constraint | `BEFORE` trigger to assert cross‑table rule, raise exception if violated.
Denormalized column maintenance | Update `order_totals.total` whenever items change.
Soft delete | On `DELETE`, flip `deleted_at` column & `RETURN NULL` (PostgreSQL).
Async replication cue | Push row IDs into a message table (`AFTER` trigger) for downstream process.

---

## 18.8  Pitfalls & Performance Considerations  

Issue | Impact | Mitigation
------|--------|-----------
Hidden logic | Surprises devs, complicates debugging | Document & centralize trigger definitions.
Row‑by‑row loops | Slow on bulk DML | Prefer set operations or statement‑level triggers w/ `inserted` sets.
Mutating‑table errors (Oracle) | Reading the same table within row trigger causes exception | Switch to statement trigger with `:NEW TABLE`.
Cascade chains | Trigger A changes table B → trigger C … | Keep chain depth shallow, monitor recursion.
Deadlocks & Lock Escalation | Additional DML inside trigger increases lock footprint | Acquire locks in deterministic order; shorten body.
Replication / logical decoding | Some engines replicate triggers’ side‑effects separately | Flag triggers as `REPLICA IDENTITY` aware or disable during ETL.

Benchmark heavy triggers with realistic batch sizes (`EXPLAIN ANALYZE` in PG w/ `auto_explain.log_nested_statements`).

---

## 18.9  Managing Triggers  

Action | PostgreSQL | SQL Server | MySQL
------|------------|-----------|-------
Disable | `ALTER TABLE tbl DISABLE TRIGGER trg;` | `DISABLE TRIGGER trg ON tbl;` | `ALTER TABLE tbl DISABLE TRIGGER trg;`
Enable  | `ALTER TABLE tbl ENABLE TRIGGER trg;` | `ENABLE TRIGGER trg ON tbl;` | `ALTER TABLE tbl ENABLE TRIGGER trg;`
Alter   | `CREATE OR REPLACE FUNCTION …` (if body only) else `DROP + CREATE TRIGGER` | `ALTER TRIGGER trg ON tbl …` | Same as PG
Drop    | `DROP TRIGGER trg ON tbl;` | `DROP TRIGGER trg ON tbl;` | `DROP TRIGGER trg;`
Introspection | `\dS+ tbl` (psql), `pg_trigger` | `sys.triggers`, `sys.trigger_events` | `information_schema.triggers`

Always version‑control trigger DDL; wrap destructive ops in `BEGIN … COMMIT` migrations.

---

### Quick Reference Cheat‑Sheet  

Need | Command / Pattern
----|------------------
Create BEFORE row trigger | `CREATE TRIGGER … BEFORE INSERT ON t FOR EACH ROW …`
Audit table changes | `AFTER INSERT OR UPDATE OR DELETE` + write to audit table
Disable all triggers (PG) | `ALTER TABLE tbl DISABLE TRIGGER ALL;`
Avoid recursion (PG) | `CREATE CONSTRAINT TRIGGER … DEFERRABLE INITIALLY DEFERRED;`
Inspect firing order (MySQL) | `SHOW TRIGGERS WHERE \G`

---

## Best‑Practice Checklist  

1. **Use triggers sparingly**—prefer declarative constraints or application logic when possible.  
2. **Keep bodies short & set‑based**; offload heavy work to background jobs.  
3. **Return `NEW` (or `NULL` for DELETE) explicitly** in row triggers (PG, MySQL).  
4. **Handle exceptions** and propagate meaningful errors to callers.  
5. **Document side‑effects** in schema README and code review gatekeepers.  
6. **Load‑test** trigger impact on bulk operations; adjust batching or disable temporarily during ETL.  
7. **Monitor** trigger execution time via engine statistics (`pg_stat_statements`, SQL Server Extended Events).

---

Properly designed triggers deliver powerful automated enforcement and auditing while remaining transparent to consuming applications. Balance their strength with disciplined governance to avoid turning the database into an opaque black box.