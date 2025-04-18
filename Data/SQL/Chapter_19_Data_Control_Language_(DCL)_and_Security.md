# Chapter 19 – Data Control Language (DCL) and Security  
Effective security hinges on robust user management, precise privilege control, and defense against injection attacks. This chapter covers DCL statements, role management, and best practices.

---

## 19.1 Database Security Principles  
- **Confidentiality**: Prevent unauthorized data access.  
- **Integrity**: Ensure only valid, auditable changes occur.  
- **Availability**: Guard against denial‑of‑service and ensure uptime.  
- **Accountability**: Maintain audit trails tying actions to identities.  
- **Defense in Depth**: Layer security at network, host, database, and object levels.

---

## 19.2 User Management  
### CREATE USER  
```sql
CREATE USER alice
  IDENTIFIED BY 'S3cur3P@ssw0rd';
```
- Defines a login and credentials.  
- Dialect variants:  
  - Oracle/PostgreSQL: `CREATE USER alice WITH PASSWORD …`  
  - SQL Server: `CREATE LOGIN alice WITH PASSWORD …; CREATE USER alice FOR LOGIN alice;`

### ALTER USER  
```sql
ALTER USER alice
  WITH PASSWORD 'N3wP@ss!';
```
- Change password, lock/unlock account, set default schema.

### DROP USER  
```sql
DROP USER IF EXISTS alice;
```
- Removes user; in some systems must drop owned objects or reassign them first.

---

## 19.3 Granting Privileges (GRANT)  
Privileges control what actions principals can perform.

### Object vs. System Privileges  
| Scope           | Examples                            | Syntax                                     |
|-----------------|-------------------------------------|--------------------------------------------|
| System          | CREATE DATABASE, CREATE TABLE, BACKUP DATABASE | `GRANT CREATE DATABASE TO bob;`            |
| Object (Schema) | SELECT, INSERT, UPDATE, DELETE, REFERENCES | `GRANT SELECT, INSERT ON schema.tbl TO alice;` |

### Examples  
```sql
-- Grant read‐only access to a table
GRANT SELECT ON sales.orders TO reporting_user;

-- Grant INSERT and UPDATE on multiple tables
GRANT INSERT, UPDATE ON hr.employees, hr.salaries TO hr_clerk;
```
- Use `WITH GRANT OPTION` sparingly to delegate privilege management.

---

## 19.4 Revoking Privileges (REVOKE)  
```sql
REVOKE UPDATE ON hr.employees FROM hr_clerk;
```
- Strips specified privileges.  
- If cascaded (e.g., roles), revocation may remove downstream grants.

---

## 19.5 Roles (CREATE ROLE, GRANT ROLE, REVOKE ROLE)  
Roles bundle privileges for easier management.

### CREATE ROLE  
```sql
CREATE ROLE analytics;
```

### Granting a Role to Users  
```sql
GRANT analytics TO alice, bob;
```

### Assigning Privileges to a Role  
```sql
GRANT SELECT ON sales.* TO analytics;
GRANT EXECUTE ON PROCEDURE calculate_bonus TO analytics;
```

### Revoking Roles  
```sql
REVOKE analytics FROM bob;
```

Roles enable centralized updates: modify role’s privileges and all members inherit changes.

---

## 19.6 SQL Injection: Understanding and Prevention Techniques  
### What Is SQL Injection?  
Injection occurs when untrusted input alters the intended SQL command, enabling unauthorized data access or manipulation.

### Common Attack Vectors  
- Concatenating user input into SQL strings.  
- Dynamic SQL without parameterization.

### Prevention Strategies  
1. **Parameterized Queries / Prepared Statements**  
   ```sql
   PREPARE stmt (INT) AS
     SELECT * FROM users WHERE id = $1;
   EXECUTE stmt(123);
   ```
2. **Stored Procedures or ORM APIs**: Abstract SQL generation.  
3. **Input Validation & Escaping**: Enforce strict types and sanitize special characters.  
4. **Least Privilege**: Database accounts should only have the minimal rights required.  
5. **Web Application Firewalls (WAFs)**: Detect and block anomalous patterns.

---

## 19.7 Principle of Least Privilege  
- **Definition**: Grant only the permissions necessary for a role or user to perform their tasks.  
- **Implementation Steps**:  
  - Audit existing privileges (`INFORMATION_SCHEMA`, system views).  
  - Create roles aligned with job functions.  
  - Assign roles, not individual privileges, where possible.  
  - Regularly review and revoke stale or over‑privileged grants.  
  - Combine with multi‑factor authentication (MFA) and strong password policies.

---

By rigorously managing users, roles, and privileges—and safeguarding against injection—you establish a secure foundation for your database environment, balancing functionality with risk mitigation.