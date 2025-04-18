# Chapter 6: Data Definition Language (DDL)

DDL commands define, modify and remove database structures (databases, tables, constraints). This chapter covers all facets of DDL.

---

## 6.1 CREATE DATABASE  
Creates a new database instance.

**Syntax**  
```sql
CREATE DATABASE [IF NOT EXISTS] db_name
  [CHARACTER SET charset_name]
  [COLLATE collation_name];
```

**Key Points**  
- `IF NOT EXISTS` prevents error if database exists.  
- `CHARACTER SET` & `COLLATE` define default encoding and sort order.

**Example**  
```sql
CREATE DATABASE IF NOT EXISTS sales_db
  CHARACTER SET utf8mb4
  COLLATE utf8mb4_unicode_ci;
```

---

## 6.2 CREATE TABLE  
Defines a new table and its columns, types, constraints.

```sql
CREATE TABLE [IF NOT EXISTS] table_name (
  column1 datatype [NULL | NOT NULL] [DEFAULT default_value] [column_constraints],
  column2 …,
  table_constraints
) [ENGINE=engine_name] [AUTO_INCREMENT = n];
```

### 6.2.1 Defining Columns & Data Types  
- **Column definition** = `name` + `datatype` + optional constraints.  
- Order matters: primary key columns usually first.

### 6.2.2 Common SQL Data Types  
- `INT[(M)] [UNSIGNED]`  
- `VARCHAR(n)`—variable-length string  
- `CHAR(n)`—fixed-length string  
- `DATE`—YYYY‑MM‑DD  
- `DATETIME`, `TIMESTAMP`  
- `DECIMAL(p, s)`—exact numeric  
- `FLOAT`, `DOUBLE`—approximate numeric  
- `BOOLEAN` (alias for `TINYINT(1)`)  

### 6.2.3 NULL / NOT NULL  
- `NOT NULL`: value required.  
- `NULL`: default, allows missing.  

### 6.2.4 DEFAULT Values  
- Sets implicit value when `INSERT` omits column.  
- Can be constant or expression (e.g., `CURRENT_TIMESTAMP`).  

**Example**  
```sql
CREATE TABLE employees (
  emp_id      INT AUTO_INCREMENT PRIMARY KEY,
  first_name  VARCHAR(50) NOT NULL,
  last_name   VARCHAR(50) NOT NULL,
  hire_date   DATE NOT NULL DEFAULT CURRENT_DATE,
  salary      DECIMAL(10,2) DEFAULT 0.00,
  is_active   BOOLEAN NOT NULL DEFAULT TRUE
) ENGINE=InnoDB;
```

---

## 6.3 ALTER TABLE  
Modifies existing table structure without data loss (in most cases).

```sql
ALTER TABLE table_name
  <operation_1>,
  <operation_2>,
  …;
```

### 6.3.1 ADD COLUMN  
```sql
ALTER TABLE employees
  ADD COLUMN middle_name VARCHAR(50) NULL AFTER first_name;
```

### 6.3.2 MODIFY / ALTER COLUMN  
- **MySQL**: `MODIFY COLUMN col_name new_datatype [constraints]`  
- **PostgreSQL**: `ALTER COLUMN col_name TYPE new_datatype;`  
```sql
ALTER TABLE employees
  MODIFY COLUMN salary DECIMAL(12,2) NOT NULL;
```

### 6.3.3 RENAME COLUMN  
- **MySQL**:  
  ```sql
  ALTER TABLE employees
    CHANGE COLUMN old_name new_name datatype;
  ```  
- **PostgreSQL**:  
  ```sql
  ALTER TABLE employees
    RENAME COLUMN old_name TO new_name;
  ```

### 6.3.4 DROP COLUMN  
```sql
ALTER TABLE employees
  DROP COLUMN middle_name;
```

### 6.3.5 Adding & Dropping Constraints  
- **Primary Key**  
  ```sql
  ALTER TABLE employees
    ADD PRIMARY KEY (emp_id);
  ```
- **Foreign Key**  
  ```sql
  ALTER TABLE orders
    ADD CONSTRAINT fk_customer
      FOREIGN KEY (customer_id) REFERENCES customers(id);
  ```
- **Unique**  
  ```sql
  ALTER TABLE employees
    ADD UNIQUE (email);
  ```
- **Check** (if supported)  
  ```sql
  ALTER TABLE employees
    ADD CHECK (salary >= 0);
  ```
- **Dropping**  
  ```sql
  ALTER TABLE table_name DROP CONSTRAINT constraint_name;
  ```

---

## 6.4 DROP TABLE  
Removes table definition and all data permanently.

```sql
DROP TABLE [IF EXISTS] table_name [, table_name2, …] [CASCADE | RESTRICT];
```

- `CASCADE`: drops dependent objects (e.g., views).  
- `RESTRICT`: prevents drop if dependencies exist (default in many RDBMS).

---

## 6.5 DROP DATABASE  
Deletes entire database and its contents.

```sql
DROP DATABASE [IF EXISTS] db_name;
```

---

## 6.6 Temporary Tables  
Session-scoped tables for intermediate computations.

- **Local temp table** (only current connection):  
  ```sql
  CREATE TEMPORARY TABLE temp_sales (
    id INT, amount DECIMAL(10,2)
  );
  ```
- Dropped automatically at session end or via `DROP TABLE temp_sales;`  
- Useful for staging, complex joins, reporting.

---

## Quick Reference

| Command          | Purpose                                | Key Options                      |
|------------------|----------------------------------------|----------------------------------|
| CREATE DATABASE  | Create new database                    | IF NOT EXISTS, CHARSET, COLLATE |
| CREATE TABLE     | Define new table & columns             | ENGINE, AUTO_INCREMENT           |
| ALTER TABLE      | Modify structure (columns/constraints) | ADD, DROP, MODIFY, RENAME        |
| DROP TABLE       | Remove table                           | IF EXISTS, CASCADE/RESTRICT      |
| DROP DATABASE    | Remove database                        | IF EXISTS                        |
| CREATE TEMPORARY TABLE | Session-only table              | N/A                              |

This comprehensive overview covers all DDL operations in Chapter 6, ensuring clarity on syntax, options and best practices.