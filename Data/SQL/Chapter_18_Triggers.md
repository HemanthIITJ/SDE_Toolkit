# Chapter 18 – Triggers  

Triggers are database‐side routines that automatically execute in response to table events. They enforce complex rules, audit changes, and integrate systems without modifying application code.

---

## 18.1 Introduction to Triggers: Automated Actions  
- **Definition**: A trigger is a stored procedure bound to a table (or view) that fires when specified events occur.  
- **Purpose**:  
  - Enforce complex business logic (beyond declarative constraints).  
  - Audit or log data changes.  
  - Maintain derived tables or replicate data.  
- **Characteristics**:  
  - Declarative binding—no application changes required.  
  - Executes in the transactional context of the triggering statement.

---

## 18.2 Creating Triggers (CREATE TRIGGER)  
### Generic Syntax (PostgreSQL / MySQL / SQL Server)  
```sql
-- PostgreSQL / MySQL 8+
CREATE TRIGGER trigger_name
  {BEFORE|AFTER|INSTEAD OF}
  {INSERT|UPDATE|DELETE}
  ON schema.table_name
  [FOR EACH ROW|FOR EACH STATEMENT]
  [WHEN (condition)]            -- PostgreSQL only
  EXECUTE FUNCTION function_name();  -- PostgreSQL
  -- OR
  BEGIN
    -- trigger body (MySQL)
  END;

-- SQL Server
CREATE TRIGGER schema.trigger_name
  ON schema.table_name
  AFTER | INSTEAD OF INSERT, UPDATE, DELETE
AS
BEGIN
  -- trigger body (T-SQL)
END;
```

---

## 18.3 Trigger Events (INSERT, UPDATE, DELETE)  
- **INSERT**: Fires when rows are added.  
- **UPDATE**: Fires on column value changes. Can scope to specific columns (e.g., `AFTER UPDATE OF col1, col2`).  
- **DELETE**: Fires when rows are removed.  

You can combine events:  
```sql
CREATE TRIGGER t_audit
  AFTER INSERT OR UPDATE OR DELETE
  ON orders …
```

---

## 18.4 Trigger Timing (BEFORE, AFTER, INSTEAD OF)  
| Timing      | Behavior                                                | Use Cases                                |
|-------------|---------------------------------------------------------|------------------------------------------|
| BEFORE      | Runs before the DML; can modify input values or abort. | Validate/normalize inputs, enforce complex checks. |
| AFTER       | Runs after successful DML; sees committed new state.    | Audit logs, update denormalized tables.  |
| INSTEAD OF  | On views (all dialects) or tables (SQL Server only); replaces DML. | Implement updatable views via triggers.  |

---

## 18.5 Row‑Level vs. Statement‑Level Triggers  
- **FOR EACH ROW** (Row‐Level): Executes once per affected row.  
- **FOR EACH STATEMENT** (Statement‐Level): Executes once per DML statement, regardless of row count.  

Choose based on workload and logic granularity: row‐level for per‐row auditing; statement‐level for coarse validations.

---

## 18.6 Accessing Old and New Data  
| Dialect / Mode   | INSERT          | UPDATE                    | DELETE         |
|------------------|-----------------|---------------------------|----------------|
| PostgreSQL       | `NEW.column`    | `OLD.column`, `NEW.column`| `OLD.column`   |
| MySQL            | `NEW.column`    | `OLD.column`,`NEW.column` | `OLD.column`   |
| SQL Server (T-SQL)| `inserted` table| `deleted` & `inserted` tables | `deleted` table |

**Examples**:  
```sql
-- PostgreSQL row‐level
CREATE TRIGGER trg_update_stock
  BEFORE UPDATE ON inventory
  FOR EACH ROW
BEGIN
  IF NEW.quantity < 0 THEN
    RAISE EXCEPTION 'Negative stock not allowed';
  END IF;
END;

-- SQL Server statement‐level
CREATE TRIGGER trg_log_delete ON orders
  AFTER DELETE
AS
BEGIN
  INSERT INTO audit_orders(order_id, deleted_at)
  SELECT id, GETDATE() FROM deleted;
END;
```

---

## 18.7 Use Cases for Triggers  
- **Auditing**: Log who, when, and how data changed.  
- **Enforcing Complex Constraints**: Cross‐table checks, business rules not expressible by `CHECK` or FK.  
- **Derived Data Maintenance**: Keep summary tables or caches in sync.  
- **Soft Deletes & Archiving**: Move deleted rows to history tables.  
- **Replication & Integration**: Propagate changes to external systems or message queues.

---

## 18.8 Potential Pitfalls and Performance Considerations  
- **Hidden Side Effects**: Triggers can obscure logic flow—document thoroughly.  
- **Recursion & Mutating Table Errors**: Avoid triggers that modify the table they fire on without safeguards.  
- **Performance Overhead**: Row‐level triggers on bulk DML can slow down loads—consider batching or statement‐level.  
- **Transaction Bloat**: Triggers extend transaction duration—monitor lock contention.  
- **Error Handling**: Uncaught exceptions in triggers roll back the entire transaction.

---

## 18.9 Managing Triggers (ALTER TRIGGER, DROP TRIGGER, ENABLE/DISABLE)  
- **ALTER TRIGGER** (limited support):  
  ```sql
  -- PostgreSQL only supports renaming
  ALTER TRIGGER old_name ON table RENAME TO new_name;
  ```
- **DROP TRIGGER**:  
  ```sql
  DROP TRIGGER IF EXISTS trigger_name ON schema.table_name;  -- PostgreSQL/MySQL
  DROP TRIGGER schema.trigger_name;                         -- SQL Server
  ```
- **ENABLE / DISABLE**:  
  ```sql
  -- PostgreSQL
  ALTER TABLE table_name ENABLE/DISABLE TRIGGER trigger_name;
  -- SQL Server
  DISABLE TRIGGER trigger_name ON table_name;
  ENABLE TRIGGER trigger_name ON table_name;
  -- MySQL: no direct support; use session variable
  SET session sql_mode='NO_ENGINE_SUBSTITUTION'; -- disables triggers if engine doesn't support them
  ```

---

By leveraging triggers judiciously—documenting their purpose, limiting side effects, and monitoring performance—you automate critical data‐integrity and auditing tasks directly within the database.