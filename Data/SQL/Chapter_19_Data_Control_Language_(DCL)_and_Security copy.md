# Chapter 19 – Data Control Language (DCL) & Security  

DCL commands (`GRANT`, `REVOKE`, user/role DDL) and secure‑coding disciplines protect the database from unauthorized access, data tampering, and service disruption.  This chapter distills core security ideas, vendor‑neutral syntax patterns, and field‑tested hardening checklists.

---

## 19.1  Database Security Principles  

Principle | Brief
----------|-------
Confidentiality | Only authorized identities can read data.
Integrity | Only authorized, validated changes are allowed.
Availability | Legitimate users always have timely access.
Defense‑in‑Depth | Combine network, OS, DB, and app controls; assume each layer *can* fail.
Auditability | Every critical action is traceable (who, what, when, where).

---

## 19.2  User Management  

Action | PostgreSQL | MySQL 8 | SQL Server
-------|------------|---------|-----------
Create | `CREATE USER alice WITH PASSWORD 'S3c!'` | `CREATE USER 'alice' IDENTIFIED BY 'S3c!';` | `CREATE LOGIN alice WITH PASSWORD = 'S3c!';  CREATE USER alice FOR LOGIN alice;`
Alter pwd / attrs | `ALTER USER alice WITH PASSWORD 'N3w!'` | `ALTER USER 'alice' IDENTIFIED BY 'N3w!';` | `ALTER LOGIN alice WITH PASSWORD='N3w!';`
Drop | `DROP USER alice;` | `DROP USER 'alice';` | `DROP USER alice;  DROP LOGIN alice;`

Best‑Practice Notes  
• Disable or rename default superusers (`sa`, `root`) where possible.  
• Enforce strong password policies or use external auth (LDAP, Kerberos, Azure AD).  
• Lock idle or dormant accounts; automate review every 90 days.

---

## 19.3  Granting Privileges — `GRANT`  

### 19.3.1 Object vs. System Privileges  

Scope | Example Privileges
------|-------------------
System‑wide | `CREATE DATABASE`, `CREATE ROLE`
Database | `CONNECT`, `TEMP`
Schema | `USAGE`, `CREATE`
Table / View | `SELECT`, `INSERT`, `UPDATE(col)`, `DELETE`, `REFERENCES`
Function | `EXECUTE`
Sequence | `USAGE`, `SELECT`, `UPDATE`

### 19.3.2 Syntax Patterns  

```sql
-- Grant object privilege
GRANT SELECT, UPDATE (price) ON products TO sales_app;

-- Grant admin option
GRANT sales_read TO bob WITH ADMIN OPTION;

-- Grant system privilege (Oracle/PG superuser)
GRANT CREATE ROLE TO dba_ops;
```

Implementation Tips  
• Grant **least** privilege required; start with `SELECT` only, add DML as needed.  
• Use column‑level updates for sensitive attributes (`UPDATE(salary)`).

---

## 19.4  Revoking Privileges — `REVOKE`  

```sql
REVOKE UPDATE ON products FROM sales_app;
REVOKE ROLE sales_read FROM bob;       -- cascades unless `GRANTED BY` others
```

Engines resolve final privilege set as:

```
effective_privs = explicit_grants − explicit_revokes
```

Always test after revocation; hidden chain grants via roles may still exist.

---

## 19.5  Roles  

Roles bundle privileges; grant the role to users, not individual rights.

Action | PostgreSQL | MySQL | SQL Server
-------|------------|-------|-----------
Create | `CREATE ROLE sales_read;` | `CREATE ROLE sales_read;` | `CREATE ROLE sales_read;`
Grant privs to role | `GRANT SELECT ON ALL TABLES IN SCHEMA public TO sales_read;` | `GRANT SELECT ON db.* TO 'sales_read';` | `GRANT SELECT ON dbo.* TO sales_read;`
Assign role to user | `GRANT sales_read TO alice;` | `GRANT sales_read TO 'alice';` | `EXEC sp_addrolemember 'sales_read', 'alice';`

Guidelines  
1. Model application personas as roles (`app_ro`, `app_rw`, `etl_admin`).  
2. Nest roles for inheritance (PG: role can GRANT other roles).  
3. For SaaS multi‑tenant, leverage row‑level security + per‑tenant role.

---

## 19.6  SQL Injection — Awareness & Mitigation  

Threat Vector | Example
--------------|---------
String concatenation | `"SELECT * FROM users WHERE name = '" + input + "';"`  
Batch injection | `1; DROP TABLE users; --`  
Blind / timing | Conditional time delays to extract bits.  

Defensive Measures  

1. **Parameterized Queries / Prepared Statements** — never concatenate untrusted input.  
2. **Least‑Privilege Accounts** — application role has `SELECT`, `INSERT`, no `DROP`.  
3. **Input Validation / Escaping** — regex, allow‑lists.  
4. **Stored Routines with Strict Interfaces** — surface‑area reduction.  
5. **WAF / Runtime Guards** — detect anomalous query patterns.

---

## 19.7  Principle of Least Privilege (PoLP)  

Rule | Implementation Checklist
-----|-------------------------
Separate Duties | Distinct DB roles for app, reporting, ETL, DBA.
Minimal Grants | Start with zero; add just enough for functionality.
Temporal Scope | Use *just‑in‑time* or *temporary* privilege escalation (`SET ROLE`, Azure AD PIM).
Monitoring | Audit tables / Extended Events capture privilege escalation & failed logins.
Review Cycle | Quarterly scripts diff current privileges vs. baseline; remediate drift.

---

### Quick Reference Cheat‑Sheet  

Task | Command
-----|--------
Create read‑only role | `CREATE ROLE app_ro; GRANT CONNECT ON DATABASE db TO app_ro; GRANT SELECT ON ALL TABLES IN SCHEMA public TO app_ro;`
Grant column‑specific update | `GRANT UPDATE(sku, price) ON products TO pricing_bot;`
Find effective grants (PG) | `\du+ alice` or query `information_schema.role_table_grants`
Block injection (code) | `cursor.execute("SELECT * FROM t WHERE id = %s", (user_id,))`

---

## Best‑Practice Playbook  

1. **Role‑based architecture first**, then bind users → roles.  
2. **Automate DCL scripts** in migration tool; avoid ad‑hoc grants via GUI.  
3. **Log & alert** on superuser activity; rotate superuser passwords; prefer certificate or key auth.  
4. **Encrypt in transit** (`TLS`) and at rest (`TDE`, disk encryption).  
5. **Disable command‑level dangerous features** (e.g., `xp_cmdshell` in SQL Server).  
6. **Red‑team regularly**—run automated scanners (`sqlmap`, custom fuzzers) against staging to catch injection vectors.

---

By rigorously applying DCL controls, role hierarchies, and injection‑proof coding practices, you convert the database from a potential attack surface into a hardened, auditable foundation for application data.