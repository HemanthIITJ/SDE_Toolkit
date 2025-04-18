# Chapter 17 – Stored Procedures and Functions  

Stored procedures and functions encapsulate SQL and procedural logic on the database server, improving performance, security, and maintainability.

---

## 17.1 Introduction to Stored Procedures and Functions  
- **Stored Procedure**: A named set of SQL statements (with control flow) stored in the database.  
- **Function (UDF)**: A routine that returns a value (scalar) or table, usable in SQL expressions.  

---

## 17.2 Benefits  
- **Performance**: Pre‑compiled execution plans, reduced parse/plan overhead.  
- **Security**: Grant execute rights without exposing underlying tables.  
- **Reusability**: Centralize complex logic for consistency.  
- **Reduced Network Traffic**: Batch multiple statements in one call.

---

## 17.3 Creating Stored Procedures (CREATE PROCEDURE)  

### 17.3.1 Input and Output Parameters  
```sql
CREATE PROCEDURE schema.calculate_bonus (
  IN  emp_id      INT,
  IN  year        INT,
  OUT bonus_amount DECIMAL(10,2)
)
LANGUAGE SQL
AS $$
  SELECT salary * 0.10
    INTO bonus_amount
  FROM employees
  WHERE id = emp_id;
$$;
```
- `IN` parameters: inputs.  
- `OUT` parameters: returned to caller.  
- Some dialects use `INOUT`.

### 17.3.2 Procedure Body (SQL Statements, Control Flow)  
Most procedural languages support:  
- **Variable declaration** (`DECLARE`).  
- **Control flow**: `IF…ELSE`, `LOOP`, `WHILE`, `FOR`.  
- **Exception handling**: `TRY…CATCH` (T‑SQL) or `EXCEPTION` blocks (PL/pgSQL).  

Example (PL/pgSQL):  
```sql
CREATE PROCEDURE adjust_stock(
  IN prod_id INT,
  IN delta   INT
)
LANGUAGE plpgsql
AS $$
BEGIN
  UPDATE inventory
  SET quantity = quantity + delta
  WHERE product_id = prod_id;

  IF NOT FOUND THEN
    RAISE EXCEPTION 'Product % not found', prod_id;
  END IF;
END;
$$;
```

---

## 17.4 Executing Stored Procedures (EXEC, CALL)  
- **MySQL / PostgreSQL**:  
  ```sql
  CALL schema.calculate_bonus(1001, 2023, @bonus);
  SELECT @bonus;
  ```
- **SQL Server (T‑SQL)**:  
  ```sql
  DECLARE @bonus DECIMAL(10,2);
  EXEC dbo.calculate_bonus @emp_id = 1001,
                          @year = 2023,
                          @bonus_amount = @bonus OUTPUT;
  SELECT @bonus;
  ```

---

## 17.5 Creating User‑Defined Functions (CREATE FUNCTION)  

### 17.5.1 Scalar Functions vs. Table‑Valued Functions  
- **Scalar UDF**: Returns single value; use in SELECT, WHERE, etc.  
  ```sql
  CREATE FUNCTION get_full_name(emp_id INT)
  RETURNS TEXT
  LANGUAGE plpgsql
  AS $$
  DECLARE
    fname TEXT;
    lname TEXT;
  BEGIN
    SELECT first_name, last_name
      INTO fname, lname
    FROM employees
    WHERE id = emp_id;
    RETURN fname || ' ' || lname;
  END;
  $$;
  ```
- **Table‑Valued UDF**: Returns a result set; behaves like a view with parameters.  
  ```sql
  CREATE FUNCTION sales_by_region(region_name TEXT)
  RETURNS TABLE(order_id INT, total DECIMAL)
  LANGUAGE sql
  AS $$
    SELECT id, total_amount
    FROM orders
    WHERE region = region_name;
  $$;
  ```

---

## 17.6 Using Functions in SQL Queries  
- **Scalar**:  
  ```sql
  SELECT id, get_full_name(id) AS full_name
  FROM employees;
  ```
- **Table‑Valued**:  
  ```sql
  SELECT *
  FROM sales_by_region('EMEA')
  WHERE total > 1000;
  ```

---

## 17.7 Modifying and Dropping Procedures and Functions  
- **ALTER**: Limited support—often requires `CREATE OR REPLACE`.  
  ```sql
  CREATE OR REPLACE FUNCTION get_full_name(emp_id INT) …;
  ```
- **DROP**:  
  ```sql
  DROP PROCEDURE IF EXISTS schema.adjust_stock;
  DROP FUNCTION  IF EXISTS schema.get_full_name(INT);
  ```

---

## 17.8 Procedural Language Extensions  
- **T‑SQL (SQL Server)**: `BEGIN…END`, `TRY…CATCH`, system stored procedures (`sp_`).  
- **PL/pgSQL (PostgreSQL)**: Variables, loops, `EXCEPTION` blocks, `PERFORM` for void queries.  
- **PL/SQL (Oracle)**: Packages, cursors, `%TYPE` declarations, robust exception handling.  

Each dialect offers unique features; choose based on your RDBMS, performance needs, and team expertise.