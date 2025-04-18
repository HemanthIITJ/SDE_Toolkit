## Chapter 6: Data Definition Language (DDL)

Data Definition Language (DDL) encompasses the subset of SQL commands used to define, modify, and remove database schema objects. These commands do not manipulate the data *within* the objects but rather the structure *of* the objects themselves (databases, tables, indexes, etc.). DDL statements typically result in changes to the database's metadata catalog. Operations are generally implicit transactions (varying by RDBMS) and often cannot be easily rolled back once executed, demanding careful planning.

---

### 6.1 Creating Databases (CREATE DATABASE)

The `CREATE DATABASE` statement is used to establish a new, independent database instance or container within the RDBMS. This serves as the primary organizational unit for tables, views, stored procedures, and other related schema objects.

**Syntax:**

```sql
CREATE DATABASE database_name
[CHARACTER SET character_set_name] -- Optional: Specifies the default character set
[COLLATE collation_name];          -- Optional: Specifies the default collation (sorting rules)
```

**Explanation:**

*   `database_name`: The unique identifier for the new database within the server instance. Naming conventions should be followed based on the specific RDBMS and organizational standards.
*   `CHARACTER SET` (Optional): Defines the default character encoding for storing textual data within this database (e.g., `utf8mb4`, `latin1`). If omitted, a server-level default is typically used.
*   `COLLATE` (Optional): Defines the default rules for comparing and sorting character strings within the database (e.g., `utf8mb4_unicode_ci`, `SQL_Latin1_General_CP1_CI_AS`). This affects `ORDER BY`, `GROUP BY`, and string comparisons. If omitted, a default collation associated with the character set or server is used.

**Considerations:**

*   **Permissions:** Executing `CREATE DATABASE` typically requires administrative privileges (e.g., `CREATE DATABASE` privilege in MySQL, membership in `dbcreator` fixed server role in SQL Server).
*   **Physical Storage:** The command triggers the allocation of underlying physical file structures (data files, log files) on the server's storage system, the specifics of which are RDBMS-dependent and configurable.

**Example:**

```sql
-- Creates a database named 'company_db' with UTF-8 encoding and case-insensitive collation
CREATE DATABASE company_db
CHARACTER SET utf8mb4
COLLATE utf8mb4_unicode_ci;
```

---

### 6.2 Creating Tables (CREATE TABLE)

The `CREATE TABLE` statement is fundamental to DDL. It defines the structure of a new table, specifying its columns, their data types, and any constraints that enforce data integrity.

**Syntax:**

```sql
CREATE TABLE table_name (
    column1_name data_type [column_constraints],
    column2_name data_type [column_constraints],
    ...
    columnN_name data_type [column_constraints],
    [table_constraints] -- Optional: Constraints applying to multiple columns
);
```

**Explanation:**

*   `table_name`: The unique identifier for the table *within its database*.
*   The clause within the parentheses `(...)` defines the table's columns and potentially table-level constraints.

#### 6.2.1 Defining Columns and Data Types

Each column represents an attribute or piece of information stored for each record (row) in the table. Every column definition requires:

1.  **`column_name`**: A unique identifier for the column *within the table*.
2.  **`data_type`**: Specifies the kind of data the column can hold, determining storage requirements and allowable values.

#### 6.2.2 Common SQL Data Types

The specific data types available can vary slightly between RDBMS implementations, but common standard types include:

*   **`INT` or `INTEGER`**: Stores whole numbers (positive, negative, or zero). Variations like `TINYINT`, `SMALLINT`, `BIGINT` offer different ranges and storage sizes.
*   **`VARCHAR(n)`**: Stores variable-length character strings up to a maximum length `n`. Storage is optimized based on the actual string length.
*   **`CHAR(n)`**: Stores fixed-length character strings of length `n`. Shorter strings are typically padded with spaces; longer strings might be truncated or cause errors depending on the RDBMS.
*   **`TEXT`**: Stores large variable-length character strings. Specific limits depend on the RDBMS (e.g., `TINYTEXT`, `TEXT`, `MEDIUMTEXT`, `LONGTEXT` in MySQL).
*   **`DATE`**: Stores calendar dates (year, month, day). Format and range depend on the RDBMS.
*   **`TIME`**: Stores time of day (hours, minutes, seconds). May include fractional seconds.
*   **`TIMESTAMP` or `DATETIME`**: Stores both date and time information. `TIMESTAMP` behaviour (e.g., automatic updates, time zone handling) can vary significantly between RDBMS.
*   **`DECIMAL(p, s)` or `NUMERIC(p, s)`**: Stores exact fixed-point numbers. `p` (precision) is the total number of digits allowed, and `s` (scale) is the number of digits to the right of the decimal point. Crucial for financial calculations where precision is paramount.
*   **`FLOAT(p)` or `REAL`**: Stores approximate-value floating-point numbers. Suitable for scientific calculations where exact precision is less critical than range. `p` specifies minimum required precision in binary digits. `DOUBLE PRECISION` offers higher precision.
*   **`BOOLEAN`**: Stores logical TRUE or FALSE values. Some RDBMS implement this using other types (e.g., `TINYINT(1)` in MySQL where 0 is FALSE and non-zero is TRUE).
*   **`BLOB` (Binary Large Object)**: Stores large binary data, such as images, audio, or serialized objects. Variations like `TINYBLOB`, `BLOB`, `MEDIUMBLOB`, `LONGBLOB` exist.

#### 6.2.3 Specifying NULL/NOT NULL Constraints

Constraints enforce rules on the data within columns or tables.

*   **`NULL`**: The default behaviour (if not specified). Allows the column to contain no value (represented as `NULL`). `NULL` is distinct from zero or an empty string.
*   **`NOT NULL`**: A column constraint ensuring that every row in the table *must* have a non-`NULL` value for this column. Essential for mandatory fields (e.g., primary keys, required identifiers).

**Syntax (Column Constraint):**

```sql
column_name data_type NOT NULL
```

#### 6.2.4 Default Values (DEFAULT)

The `DEFAULT` constraint specifies a value to be automatically inserted into a column if no explicit value is provided during an `INSERT` operation.

**Syntax (Column Constraint):**

```sql
column_name data_type DEFAULT default_value
```

*   `default_value`: Must be a literal value compatible with the column's data type (e.g., `0` for `INT`, `'N/A'` for `VARCHAR`, `CURRENT_TIMESTAMP` for a `TIMESTAMP` column).

**Example `CREATE TABLE`:**

```sql
CREATE TABLE employees (
    employee_id INT PRIMARY KEY, -- Primary key constraint implicitly enforces NOT NULL and UNIQUE
    first_name VARCHAR(50) NOT NULL,
    last_name VARCHAR(50) NOT NULL,
    email VARCHAR(100) UNIQUE, -- Ensures email addresses are unique across the table
    hire_date DATE NOT NULL,
    salary DECIMAL(10, 2) CHECK (salary >= 0), -- Ensures salary is non-negative
    department_id INT,
    is_active BOOLEAN DEFAULT TRUE, -- Defaults to TRUE if not specified
    last_updated TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP, -- Example: Auto-update timestamp (Syntax varies)
    -- Table-level constraint example (Foreign Key)
    FOREIGN KEY (department_id) REFERENCES departments(department_id)
);
```
*Note: `PRIMARY KEY`, `UNIQUE`, `CHECK`, `FOREIGN KEY` are also common constraints often defined during table creation.*

---

### 6.3 Modifying Table Structures (ALTER TABLE)

The `ALTER TABLE` statement allows modification of an existing table's structure without deleting and recreating it. This includes adding, dropping, or modifying columns and constraints.

**Base Syntax:**

```sql
ALTER TABLE table_name
[action1]
[, action2]...;
```

`action` refers to specific modifications like `ADD COLUMN`, `DROP COLUMN`, `ALTER COLUMN`, etc.

#### 6.3.1 Adding Columns (ADD COLUMN)

Adds one or more new columns to an existing table.

**Syntax:**

```sql
ALTER TABLE table_name
ADD COLUMN column_name data_type [column_constraints] [AFTER existing_column | FIRST]; -- Placement clause is RDBMS-specific
```

**Example:**

```sql
-- Adds a 'phone_number' column to the 'employees' table
ALTER TABLE employees
ADD COLUMN phone_number VARCHAR(20) NULL; -- Allows NULL values by default
```

#### 6.3.2 Modifying Columns (ALTER COLUMN, MODIFY COLUMN)

Changes the data type, nullability, default value, or other properties of an existing column. The exact keyword varies:

*   **SQL Standard / PostgreSQL / SQL Server:** `ALTER COLUMN`
*   **MySQL / Oracle:** `MODIFY COLUMN` (MySQL also uses `CHANGE COLUMN` for renaming + modifying)

**Syntax (Conceptual - Adapt for specific RDBMS):**

```sql
-- SQL Server / PostgreSQL Example: Change data type
ALTER TABLE table_name
ALTER COLUMN column_name TYPE new_data_type; -- PostgreSQL
ALTER TABLE table_name
ALTER COLUMN column_name new_data_type; -- SQL Server

-- MySQL Example: Change data type and add NOT NULL
ALTER TABLE table_name
MODIFY COLUMN column_name new_data_type NOT NULL;

-- Oracle Example: Change data type
ALTER TABLE table_name
MODIFY (column_name new_data_type);
```

**Example (MySQL):**

```sql
-- Changes 'phone_number' in 'employees' to require a value and potentially a different type/length
ALTER TABLE employees
MODIFY COLUMN phone_number VARCHAR(25) NOT NULL;


#### 6.3.3 Renaming Columns

Changes the name of an existing column. Syntax varies significantly across database systems.

**Syntax (Conceptual - Adapt for specific RDBMS):**

```sql
-- MySQL Example
ALTER TABLE table_name
CHANGE COLUMN old_column_name new_column_name data_type [column_constraints];
-- Note: MySQL's CHANGE requires specifying the data type even if it's not changing.

-- PostgreSQL Example
ALTER TABLE table_name
RENAME COLUMN old_column_name TO new_column_name;

-- SQL Server Example
EXEC sp_rename 'table_name.old_column_name', 'new_column_name', 'COLUMN';

-- Oracle Example
ALTER TABLE table_name
RENAME COLUMN old_column_name TO new_column_name;
```

**Example (PostgreSQL):**

```sql
-- Renames the 'phone_number' column to 'contact_number' in the 'employees' table
ALTER TABLE employees
RENAME COLUMN phone_number TO contact_number;
```

**Considerations:**

*   **Dependencies:** Renaming a column can break stored procedures, views, triggers, or application code that references the old column name. Thorough impact analysis is crucial.
*   **Data Type Specification:** As noted, some systems (like MySQL with `CHANGE COLUMN`) require restating the column's data type and constraints during a rename operation.

#### 6.3.4 Dropping Columns (DROP COLUMN)

Removes a column and all its associated data from a table. This operation is typically irreversible.

**Syntax:**

```sql
ALTER TABLE table_name
DROP COLUMN column_name;
```

**Example:**

```sql
-- Assuming 'contact_number' is no longer needed in the 'employees' table
ALTER TABLE employees
DROP COLUMN contact_number;
```

**Considerations:**

*   **Data Loss:** All data stored within the dropped column is permanently lost.
*   **Dependencies:** Similar to renaming, dropping a column can break dependent objects and applications. Exercise extreme caution.
*   **Constraints:** If the column is part of any constraints (e.g., Primary Key, Foreign Key, Unique, Check), those constraints might need to be dropped first, or the `DROP COLUMN` command might fail, depending on the RDBMS and constraint type. Some systems support `CASCADE` options to automatically drop related constraints, but this must be used judiciously.

#### 6.3.5 Adding and Dropping Constraints

Constraints enforce data integrity rules. `ALTER TABLE` can add new constraints or remove existing ones after a table has been created.

**Adding Constraints:**

**Syntax:**

```sql
ALTER TABLE table_name
ADD CONSTRAINT constraint_name constraint_definition;
```

*   `constraint_name`: An optional (but highly recommended) name for the constraint. If omitted, the RDBMS usually assigns a system-generated name. Naming constraints explicitly aids in managing and dropping them later.
*   `constraint_definition`: The type and specification of the constraint (e.g., `PRIMARY KEY (col1, col2)`, `FOREIGN KEY (col_fk) REFERENCES parent_table(col_pk)`, `UNIQUE (col_unique)`, `CHECK (condition)`).

**Examples:**

```sql
-- Add a UNIQUE constraint to the email column in employees
ALTER TABLE employees
ADD CONSTRAINT uq_employee_email UNIQUE (email);

-- Add a FOREIGN KEY constraint
ALTER TABLE employees
ADD CONSTRAINT fk_employee_dept
FOREIGN KEY (department_id) REFERENCES departments(department_id);

-- Add a CHECK constraint
ALTER TABLE products
ADD CONSTRAINT chk_product_price CHECK (unit_price > 0);
```

**Dropping Constraints:**

**Syntax:**

```sql
ALTER TABLE table_name
DROP CONSTRAINT constraint_name; -- Requires knowing the constraint's name
```
*Note: Syntax for dropping PRIMARY KEY varies (e.g., `DROP PRIMARY KEY` in MySQL).*

**Example:**

```sql
-- Drop the unique constraint on email
ALTER TABLE employees
DROP CONSTRAINT uq_employee_email;

-- Drop the foreign key constraint (using its defined name)
ALTER TABLE employees
DROP CONSTRAINT fk_employee_dept;
```

**Considerations:**

*   **Naming:** Explicitly naming constraints during creation significantly simplifies dropping them later. Finding system-generated names can be cumbersome.
*   **Impact:** Adding constraints might fail if existing data violates the new rule. Dropping constraints removes integrity enforcement, potentially allowing invalid data insertions or updates.

---

### 6.4 Deleting Tables (DROP TABLE)

The `DROP TABLE` statement permanently removes a table, including its structure, data, indexes, constraints, and triggers associated with it.

**Syntax:**

```sql
DROP TABLE [IF EXISTS] table_name [, table_name2 ...];
```

**Explanation:**

*   `table_name`: The name of the table to be deleted. Multiple tables can often be dropped in a single statement (RDBMS-dependent).
*   `IF EXISTS` (Optional): A useful clause that prevents an error from occurring if the specified table does not actually exist. The statement completes without action in that case.

**Example:**

```sql
-- Deletes the 'inactive_employees' table, only if it exists
DROP TABLE IF EXISTS inactive_employees;

-- Deletes the 'temp_data' and 'staging_area' tables
DROP TABLE temp_data, staging_area;
```

**Considerations:**

*   **Irreversibility:** This operation is permanent and database backups are the only way to recover a dropped table and its data.
*   **Permissions:** Requires specific privileges (e.g., `DROP` privilege on the table or schema/database).
*   **Dependencies:** Attempting to drop a table referenced by a `FOREIGN KEY` constraint in another table will typically fail unless cascading options are specified (`CASCADE CONSTRAINTS` in some RDBMS, which also drops the referencing foreign key) or the referencing constraint is dropped first. Views or stored procedures referencing the table will become invalid.

---

### 6.5 Deleting Databases (DROP DATABASE)

The `DROP DATABASE` statement permanently removes an entire database, including all its tables, views, indexes, stored procedures, functions, data, and associated log files.

**Syntax:**

```sql
DROP DATABASE [IF EXISTS] database_name;
```

**Explanation:**

*   `database_name`: The name of the database to be deleted.
*   `IF EXISTS` (Optional): Prevents an error if the database does not exist.

**Example:**

```sql
-- Permanently deletes the 'test_archive_db' database, if it exists
DROP DATABASE IF EXISTS test_archive_db;
```

**Critical Considerations:**

*   **Extreme Caution:** This is the most destructive DDL command. It erases the entire database and all its contents. There is typically NO undo capability within the RDBMS itself. Recovery is only possible via prior backups.
*   **Permissions:** Requires high-level administrative privileges (e.g., `DROP` privilege on the database, membership in `sysadmin` or `dbcreator` roles).
*   **Exclusive Access:** Most RDBMS require that no active connections exist to the database being dropped. You may need to disconnect all users and applications, potentially putting the database in a single-user mode or taking it offline before dropping it.
*   **System Databases:** Core system databases (e.g., `master`, `model`, `msdb`, `tempdb` in SQL Server; `mysql`, `information_schema`, `performance_schema` in MySQL) cannot be dropped.

---

### 6.6 Temporary Tables

Temporary tables are tables that exist only for the duration of a specific database session or transaction. They are useful for storing intermediate results during complex queries or procedures without permanently altering the database schema. Their scope and behavior vary between RDBMS.

**Purpose:**

*   Hold intermediate data sets within a session or procedure.
*   Break down complex queries into simpler steps.
*   Provide workspace without interfering with persistent tables or concurrency.

**Characteristics:**

*   **Scope:** Usually limited to the session that created them. They are automatically dropped when the session ends. Some RDBMS offer global temporary tables visible across sessions but still temporary in nature.
*   **Naming:** Often require special syntax or prefixes (e.g., `#` or `##` in SQL Server, `CREATE TEMPORARY TABLE` in MySQL/PostgreSQL). Name conflicts are usually resolved per-session (private temporary tables).
*   **Storage:** May be stored in memory or within a dedicated temporary tablespace (e.g., `tempdb` in SQL Server).
*   **DDL/DML:** Typically support standard `CREATE`, `INSERT`, `UPDATE`, `DELETE`, `SELECT` operations within their scope. Indexes can often be created on them.

**Syntax Examples:**

```sql
-- MySQL / PostgreSQL Example
CREATE TEMPORARY TABLE temp_customer_subset (
    customer_id INT PRIMARY KEY,
    customer_name VARCHAR(100),
    last_order_date DATE
);

-- Populate the temporary table
INSERT INTO temp_customer_subset (customer_id, customer_name, last_order_date)
SELECT c.id, c.name, MAX(o.order_date)
FROM customers c
JOIN orders o ON c.id = o.customer_id
WHERE c.region = 'North'
GROUP BY c.id, c.name;

-- Use the temporary table
SELECT * FROM temp_customer_subset WHERE last_order_date < '2023-01-01';

-- Table is automatically dropped at session end. Can also be explicitly dropped:
-- DROP TABLE temp_customer_subset;


-- SQL Server Example (Local Temporary Table - prefix #)
CREATE TABLE #temp_product_list (
    product_id INT,
    product_name VARCHAR(150),
    inventory_count INT
);

-- SQL Server Example (Global Temporary Table - prefix ##)
CREATE TABLE ##global_temp_config (
    config_key VARCHAR(50),
    config_value VARCHAR(200)
);
-- Global temp tables (##) are visible to all sessions until the creating session ends
-- AND all other sessions stop referencing it.
```

**Considerations:**

*   **Performance:** While useful, overuse or inefficient use (e.g., huge temporary tables, missing indexes) can impact performance, especially contention on shared temporary resources like `tempdb`.
*   **Syntax Variance:** The exact syntax for creation, naming conventions, and scope rules differ significantly between RDBMS platforms. Consult the specific documentation for your database system.<ctrl63>