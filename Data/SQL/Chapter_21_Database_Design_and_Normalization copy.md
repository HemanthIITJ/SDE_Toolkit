# Chapter 21 – Database Design and Normalization  

Robust database design ensures data integrity, scalability, and maintainability. This chapter covers foundational principles, modeling stages, ERDs, normalization theory, and pragmatic denormalization and key‐type choices.

---

## 21.1 Principles of Good Database Design  
- **Atomicity & Simplicity**: Break data into the smallest meaningful pieces.  
- **Consistency**: Enforce integrity via constraints (PK, FK, unique, check).  
- **Minimal Redundancy**: Avoid duplicate storage to reduce anomalies.  
- **Flexibility & Extensibility**: Anticipate future requirements; design loosely coupled schemas.  
- **Performance & Scalability**: Balance normalization with query patterns and indexing.  
- **Security & Compliance**: Apply least privilege, separate sensitive data, and enable auditing.

---

## 21.2 Data Modeling Stages  
1. **Conceptual Model**  
   - High‐level view of business entities and relationships.  
   - Independent of technology; often captured in ER diagrams.  
2. **Logical Model**  
   - Translates entities into tables, attributes into columns.  
   - Defines primary keys, foreign keys, and normalized structures.  
   - Still vendor‐agnostic.  
3. **Physical Model**  
   - Implements logical design in a specific RDBMS.  
   - Chooses data types, indexes, partitioning, storage settings.

---

## 21.3 Entity‑Relationship Diagrams (ERDs)  
- **Entities**: Objects (tables) with attributes (columns).  
- **Relationships**: Lines connecting entities with cardinalities (1:1, 1:N, N:M).  
- **Attributes**:  
  - **Key**: underlined (PK), foreign keys (FK) annotated.  
  - **Composite**: multi‐attribute keys.  
  - **Multivalued / Derived**: special notation (double ovals).  
- **Notation**: Chen (rectangles/diamonds), Crow’s Foot (boxes and forked lines).  

Use ERDs to validate requirements, communicate design, and drive schema creation.

---

## 21.4 Normalization: Reducing Data Redundancy  
Progressive decomposition of tables to eliminate anomalies.

| Normal Form | Requirement                                                 |
|-------------|-------------------------------------------------------------|
| 1NF         | Atomic values; no repeating groups or arrays.               |
| 2NF         | In 1NF; every non‑key attribute fully depends on the PK.    |
| 3NF         | In 2NF; no transitive dependencies (non‑key → non‑key).     |
| BCNF        | Stronger than 3NF; every determinant is a candidate key.   |

### 21.4.1 First Normal Form (1NF)  
- Eliminate multi‐valued attributes; ensure each column holds a single value.  
- Example: 
  - ❌ orders(order_id, product_ids)  
  - ✅ orders(order_id) + order_items(order_id, product_id)

### 21.4.2 Second Normal Form (2NF)  
- Apply only to composite PKs.  
- Move attributes that depend on part of the key into separate tables.  
- Example: 
  - ❌ order_items(order_id, product_id, product_name)  
  - ✅ product(product_id, product_name); order_items(order_id, product_id)

### 21.4.3 Third Normal Form (3NF)  
- Remove attributes that depend on other non‑key attributes.  
- Example: 
  - ❌ employees(emp_id, dept_id, dept_name)  
  - ✅ department(dept_id, dept_name); employees(emp_id, dept_id)

### 21.4.4 Boyce‑Codd Normal Form (BCNF)  
- Every functional dependency X → Y implies X is a candidate key.  
- Handles edge cases where 3NF still allows anomalies.

---

## 21.5 Denormalization: When and Why  
Purposeful introduction of redundancy to optimize read performance.

**Use Cases**  
- High‑volume reporting tables where joins are costly.  
- Precomputed summary or aggregate columns.  
- Caching lookup values to reduce join overhead.

**Techniques**  
- Add summary tables or materialized views.  
- Embed frequently accessed attributes (e.g., customer_name in orders).  
- Use appropriate index strategies (covering indexes).

**Risks**  
- Update anomalies; additional triggers or application logic needed.  
- Increased storage and potential consistency overhead.

---

## 21.6 Choosing Appropriate Data Types and Keys  

### Data Types  
- **Size & Precision**: Match business requirements (e.g., `DECIMAL(10,2)` for currency).  
- **Specialized Types**: `UUID`, `ENUM`, `JSONB`, date/time with time zone.  
- **Avoid** over‑sizing (e.g., `VARCHAR(65535)`) which increases I/O.

### Keys  
- **Natural Keys**: Meaningful business identifiers (e.g., ISO country code).  
  - Pros: Readability; avoids joins to lookup.  
  - Cons: Subject to change; length may impact index size.  
- **Surrogate Keys**: Synthetic (integer, sequence, UUID).  
  - Pros: Stable, compact, simple FK relationships.  
  - Cons: Lack inherent meaning.  
- **Composite Keys**: When uniqueness spans multiple attributes.  
  - Use sparingly; may complicate FKs and indexing.  

**Guidelines**  
- Use surrogate PKs as default; enforce natural uniqueness via UNIQUE constraints.  
- Choose data types that balance storage efficiency with functional requirements.  
- Document key choices and data‐type rationale for future maintenance.

---

By following rigorous modeling and normalization rules—and applying denormalization judiciously—you create schemas that are both logically sound and performant, while choosing data types and keys aligned with your application’s needs.