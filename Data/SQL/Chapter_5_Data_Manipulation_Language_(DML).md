## **Chapter 5: Data Manipulation Language (DML)**

### **Introduction**

Data Manipulation Language (DML) comprises a subset of SQL statements used to manage data within existing schema objects. Unlike Data Definition Language (DDL) which defines the structure of the database, DML is concerned with the querying and modification of data stored within tables. The core DML statements are `INSERT`, `UPDATE`, and `DELETE`. This chapter provides a comprehensive examination of these commands, including variations and critical considerations for database developers. We will also cover the `TRUNCATE TABLE` statement, which, while often grouped with DDL due to some of its characteristics, directly pertains to data removal.

---

### **5.1 Inserting Data (INSERT INTO)**

The `INSERT INTO` statement is used to add new rows (records) to a table.

#### **5.1.1 Inserting Single Rows with VALUES**

This is the most fundamental form of the `INSERT` statement, allowing the addition of a single row by explicitly providing the values for its columns.

*   **Syntax:**
    ```sql
    INSERT INTO table_name (column1, column2, column3, ...)
    VALUES (value1, value2, value3, ...);
    ```
    *   `table_name`: The target table for the new row.
    *   `(column1, column2, ...)`: An optional list specifying the columns for which values are being provided. If omitted, values must be supplied for *all* columns in the table, in the order they are defined. **Best practice:** Always specify the column list explicitly to enhance readability and prevent errors if the table structure changes.
    *   `VALUES (value1, value2, ...)`: Specifies the values to be inserted into the corresponding columns. The number and data types of the values must match the number and data types of the specified columns (or all columns if the list is omitted).

*   **Example:**
    Assuming an `employees` table with columns `employee_id` (INT, Primary Key), `first_name` (VARCHAR), `last_name` (VARCHAR), `hire_date` (DATE), and `salary` (DECIMAL):
    ```sql
    INSERT INTO employees (employee_id, first_name, last_name, hire_date, salary)
    VALUES (101, 'Alice', 'Smith', '2023-10-01', 75000.00);
    ```
*   **Considerations:**
    *   Constraints (e.g., `NOT NULL`, `UNIQUE`, `PRIMARY KEY`, `FOREIGN KEY`, `CHECK`) defined on the table columns must be satisfied by the inserted values. Violations will result in an error, and the insertion will fail.
    *   Default values defined for columns will be used if those columns are omitted from the explicit column list.
    *   Auto-incrementing columns (e.g., identity columns, sequences) typically do not need to be specified in the column list or values; the database assigns the next value automatically.

#### **5.1.2 Inserting Multiple Rows**

Most SQL dialects allow inserting multiple rows using a single `INSERT` statement, which is generally more efficient than executing separate `INSERT` statements for each row.

*   **Syntax (Common Standard):**
    ```sql
    INSERT INTO table_name (column1, column2, column3, ...)
    VALUES
        (value1a, value2a, value3a, ...),
        (value1b, value2b, value3b, ...),
        (value1c, value2c, value3c, ...);
    ```
    *   Multiple tuples (rows) of values are listed after the `VALUES` keyword, separated by commas.

*   **Example:**
    ```sql
    INSERT INTO employees (employee_id, first_name, last_name, hire_date, salary)
    VALUES
        (102, 'Bob', 'Jones', '2023-10-05', 68000.00),
        (103, 'Charlie', 'Brown', '2023-11-01', 71000.00);
    ```
*   **Considerations:**
    *   The syntax for multiple row inserts might vary slightly across different Database Management Systems (DBMS).
    *   This method significantly reduces network latency and parsing overhead compared to individual inserts.

#### **5.1.3 Inserting Data from Another Table (INSERT INTO ... SELECT)**

This powerful variant allows inserting rows into a table based on the result set of a `SELECT` query executed against another table (or even the same table).

*   **Syntax:**
    ```sql
    INSERT INTO target_table (column1, column2, ...)
    SELECT source_column1, source_column2, ...
    FROM source_table
    WHERE condition;
    ```
    *   `target_table`: The table receiving the new rows.
    *   `(column1, column2, ...)`: The columns in the `target_table` to be populated.
    *   `SELECT source_column1, ... FROM source_table WHERE condition;`: A standard `SELECT` query that retrieves data. The number, order, and data types of the columns returned by the `SELECT` statement must be compatible with the specified columns in the `target_table`.

*   **Example:**
    Inserting employees hired before 2023 from the `employees` table into an `archived_employees` table with matching columns:
    ```sql
    INSERT INTO archived_employees (employee_id, first_name, last_name, hire_date, salary)
    SELECT employee_id, first_name, last_name, hire_date, salary
    FROM employees
    WHERE hire_date < '2023-01-01';
    ```
*   **Considerations:**
    *   The `SELECT` query can be arbitrarily complex, involving joins, functions, and aggregations, as long as the resulting columns match the target table structure.
    *   This is highly efficient for bulk data copying, migration, or archiving.
    *   Ensure data type compatibility between source `SELECT` columns and target table columns. Implicit conversions might occur, but rely on explicit casting (`CAST` or `CONVERT`) for clarity and control if necessary.

---

### **5.2 Updating Existing Data (UPDATE)**

The `UPDATE` statement is used to modify existing data within one or more rows of a table.

#### **5.2.1 The SET Clause**

The `SET` clause is mandatory within an `UPDATE` statement. It specifies which columns are to be updated and defines the new values they should hold.

*   **Syntax Fragment:**
    ```sql
    UPDATE table_name
    SET column1 = value1,
        column2 = value2,
        ...
    ```
    *   `columnN = valueN`: Assigns a new `valueN` to `columnN`. Values can be literals, expressions, scalar subqueries, or values from other columns in the same row.

#### **5.2.2 Using WHERE to Specify Rows for Update**

The `WHERE` clause is crucial for controlling *which* rows are affected by the `UPDATE` statement. It defines a condition that rows must meet to be updated.

*   **Syntax:**
    ```sql
    UPDATE table_name
    SET column1 = value1, column2 = value2, ...
    WHERE condition;
    ```
    *   `condition`: A logical expression (e.g., `employee_id = 101`, `salary < 50000`, `department_id IS NULL`). Only rows evaluating this condition to `TRUE` will be updated.

*   **Example:**
    Giving a 5% raise to employee with ID 101:
    ```sql
    UPDATE employees
    SET salary = salary * 1.05
    WHERE employee_id = 101;
    ```

#### **5.2.3 Updating Multiple Columns**

A single `UPDATE` statement can modify multiple columns simultaneously by listing multiple assignments in the `SET` clause, separated by commas.

*   **Example:**
    Updating the last name and salary for employee ID 102:
    ```sql
    UPDATE employees
    SET last_name = 'Williams',
        salary = 70000.00
    WHERE employee_id = 102;
    ```

#### **5.2.4 Potential Pitfalls (Updating without WHERE)**

**Critical Warning:** Omitting the `WHERE` clause in an `UPDATE` statement is extremely dangerous.

*   **Behavior:** If the `WHERE` clause is absent, the `UPDATE` statement will apply the changes specified in the `SET` clause to **every single row** in the table.
*   **Example (Dangerous - Do Not Run Without Caution):**
    ```sql
    -- WARNING: This updates the salary of ALL employees to 50000.00
    UPDATE employees
    SET salary = 50000.00;
    ```
*   **Consequences:** This often leads to catastrophic data loss or corruption that can be difficult to recover from without proper backups.
*   **Mitigation:**
    *   **Always** double-check `UPDATE` statements for the presence and correctness of the `WHERE` clause before execution, especially in production environments.
    *   Execute `UPDATE` statements within transactions (`BEGIN TRANSACTION`, `COMMIT`, `ROLLBACK`) where possible, allowing for reversal if an error is detected immediately.
    *   Test `UPDATE` statements on non-production environments first.
    *   Consider using a `SELECT` statement with the same `WHERE` clause first to verify which rows *would* be affected.

---

### **5.3 Deleting Data (DELETE)**

The `DELETE` statement is used to remove one or more rows from a table.

#### **5.3.1 Using WHERE to Specify Rows for Deletion**

Similar to `UPDATE`, the `WHERE` clause is essential for specifying *which* rows should be deleted.

*   **Syntax:**
    ```sql
    DELETE FROM table_name
    WHERE condition;
    ```
    *   `table_name`: The table from which rows will be deleted.
    *   `condition`: A logical expression identifying the rows
```sql
    DELETE FROM table_name
    WHERE condition;
```
    *   `table_name`: The table from which rows will be deleted.
    *   `condition`: A logical expression identifying the rows to be removed. Only rows for which the condition evaluates to `TRUE` will be deleted.

*   **Example:**
    Deleting the employee with ID 103:
    ```sql
    DELETE FROM employees
    WHERE employee_id = 103;
    ```
    Deleting all employees hired before January 1st, 2022:
    ```sql
    DELETE FROM employees
    WHERE hire_date < '2022-01-01';
    ```

*   **Considerations:**
    *   **Referential Integrity:** If foreign key constraints are defined referencing the table from which rows are being deleted, the deletion might fail unless `ON DELETE CASCADE` or `ON DELETE SET NULL` actions are specified in the foreign key definition.
    *   **Triggers:** `DELETE` triggers associated with the table, if any, will be activated for each row that is deleted.

#### **5.3.2 Deleting All Rows (DELETE FROM table vs. TRUNCATE TABLE)**

Similar to `UPDATE`, omitting the `WHERE` clause in a `DELETE` statement has significant consequences.

*   **Syntax (Dangerous - Use with Extreme Caution):**
    ```sql
    DELETE FROM table_name;
    ```
*   **Behavior:** When executed without a `WHERE` clause, the `DELETE` statement removes **all rows** from the specified table.
*   **Mechanism:**
    *   `DELETE` typically operates on a row-by-row basis.
    *   It logs each individual row deletion in the transaction log (if the database is configured for logging and recovery). This allows the operation to be rolled back within a transaction.
    *   It fires any `DELETE` triggers associated with the table for each deleted row.
    *   It generally does not reset auto-incrementing counter values (like Identity columns or sequences) associated with the table. The next `INSERT` will typically use the next value in the sequence, not restart from the beginning.

*   **Consequences of `DELETE` without `WHERE`:**
    *   Can lead to unintentional, complete removal of data from a table.
    *   Can be very slow and resource-intensive on large tables due to row-by-row processing and extensive logging.

*   **Comparison with `TRUNCATE TABLE`:**
    When the goal is to remove *all* data from a table quickly, `TRUNCATE TABLE` is often a more efficient alternative. The `TRUNCATE TABLE` statement is detailed in the next section (5.4), but the key differences are introduced here for context:
    *   **`DELETE FROM table_name;`**: Logs each row deletion, fires delete triggers, can be rolled back easily, doesn't reset identity counters, requires `DELETE` permissions.
    *   **`TRUNCATE TABLE table_name;`**: Usually minimally logged (often just page deallocations), much faster for large tables, does *not* fire delete triggers, may not be easily rolled back (depending on DBMS and transaction context), often resets identity counters, typically requires higher privileges (like `ALTER TABLE` or specific `TRUNCATE` permissions).

*   **Mitigation:**
    *   **Always** verify the presence and correctness of the `WHERE` clause before executing `DELETE`.
    *   Use transactions (`BEGIN TRANSACTION`, `COMMIT`, `ROLLBACK`).
    *   Consider using a `SELECT COUNT(*)` with the same `WHERE` clause first to estimate the number of affected rows.
    *   If the intent is to remove *all* rows, evaluate whether `TRUNCATE TABLE` is a more suitable and efficient option (see Section 5.4).

---

### **5.4 The TRUNCATE TABLE Statement**

`TRUNCATE TABLE` is a command used to rapidly remove all rows from a table. While it achieves a similar outcome to `DELETE FROM table_name;`, its underlying mechanism and characteristics are significantly different. It is sometimes classified as DDL rather than DML because it often involves storage deallocation more akin to dropping and recreating the table structure, although it only affects the data, not the table definition itself.

*   **Syntax:**
    ```sql
    TRUNCATE TABLE table_name;
    ```
    *   `table_name`: The table from which all rows will be removed.

*   **Behavior and Characteristics:**
    *   **Speed:** `TRUNCATE` is generally much faster than `DELETE` without a `WHERE` clause, especially for large tables. This is because it typically works by deallocating the data pages used by the table rather than deleting rows individually.
    *   **Logging:** The operation is usually minimally logged. Instead of logging each deleted row, the database might only log the deallocation of data pages. This dramatically reduces transaction log usage but can make point-in-time recovery or fine-grained rollbacks more difficult or impossible, depending on the DBMS.
    *   **`WHERE` Clause:** `TRUNCATE TABLE` **cannot** have a `WHERE` clause. It always removes *all* rows.
    *   **Triggers:** `TRUNCATE TABLE` does **not** fire `ON DELETE` triggers associated with the table. If trigger logic must execute upon row removal, `DELETE` must be used.
    *   **Identity Resets:** In many database systems (e.g., SQL Server, MySQL), `TRUNCATE TABLE` resets any associated identity columns (auto-increment counters) back to their seed value. The next `INSERT` will start from the initial value. PostgreSQL's behavior might differ depending on sequence definitions.
    *   **Foreign Key Constraints:** `TRUNCATE TABLE` often cannot be used on a table that is referenced by a `FOREIGN KEY` constraint from another table, unless the constraint is disabled, deferred, or the referencing rows are removed first. `DELETE` allows managing such dependencies on a row-by-row basis or via cascade options.
    *   **Transactional Behavior:** While `TRUNCATE TABLE` can often be included in a transaction block (`BEGIN`/`COMMIT`/`ROLLBACK`), its ability to be truly rolled back can vary. In some systems or configurations, the deallocation might be immediate and harder to undo than a logged `DELETE`.
    *   **Permissions:** Executing `TRUNCATE TABLE` typically requires higher Cprivileges than `DELETE` (e.g., `ALTER` permission on the table or a specific `TRUNCATE` privilege).

*   **When to Use `TRUNCATE` vs. `DELETE`:**
    *   Use `TRUNCATE TABLE` when:
        *   You need to delete *all* rows from a table.
        *   Performance is critical, especially on large tables.
        *   You want to reset identity counters.
        *   Firing `DELETE` triggers is not required or desired.
        *   Reduced transaction logging is acceptable or beneficial.
    *   Use `DELETE FROM table_name;` (without `WHERE`) when:
        *   You need to fire `DELETE` triggers for each row.
        *   The operation must be fully logged for replication or rollback purposes.
        *   You do *not* want to reset identity counters.
        *   A row-by-row operation is required for other reasons (e.g., specific locking behavior).
        *   You only have `DELETE` privileges, not `ALTER` or `TRUNCATE`.
    *   Use `DELETE FROM table_name WHERE condition;` when:
        *   You need to delete a *subset* of rows based on specific criteria.

---