# Chapter 20 – Advanced Data Types & Features  

Modern RDBMSs go well beyond classic `INT`, `VARCHAR`, and `DATE`.  They embed semantically‑rich types (JSON, XML, spatial), search engines, and dynamic reshaping operators that allow complex workloads to stay inside SQL.  This chapter catalogues those capabilities, explains cross‑vendor syntax, and offers performance guidance.

---

## 20.1  Date & Time Tool‑box  

Feature | PostgreSQL | SQL Server | MySQL 8
--------|------------|-----------|---------
Current timestamp | `NOW()` / `CURRENT_TIMESTAMP` | `SYSDATETIME()` | `CURRENT_TIMESTAMP`
Extract part | `EXTRACT(isodow FROM ts)` | `DATEPART(weekday, ts)` | `EXTRACT(DAYOFWEEK FROM ts)`
Add / Sub | `ts + INTERVAL '3 days'` | `DATEADD(day, 3, ts)` | `DATE_ADD(ts, INTERVAL 3 DAY)`
Diff | `AGE(t2,t1)` / `ts - ts` | `DATEDIFF(day, t1, t2)` | `TIMESTAMPDIFF(DAY,t1,t2)`
Truncate | `date_trunc('month', ts)` | `DATEFROMPARTS(YEAR(ts),MONTH(ts),1)` | `DATE_FORMAT(ts,'%Y-%m-01')`
TZ convert | `ts AT TIME ZONE 'UTC'` | `SWITCHOFFSET(ts, '+00:00')` | `CONVERT_TZ(ts,'UTC','Europe/Berlin')`

Best‑Practice  
• Store in **UTC TIMESTAMP**; convert in UI.  
• Use `TIMESTAMP WITH TIME ZONE` (PG) / `datetimeoffset` (SQL Server) for cross‑TZ systems.  
• Index `(date_column)` + `WHERE date_column ≥ CURRENT_DATE - INTERVAL '30 days'` to exploit partition pruning.

---

## 20.2  String Manipulation & Search  

Category | Common Calls
---------|--------------
Concatenation | `||` (PG/Oracle), `CONCAT(a,b)` (MySQL), `+` (SQL Server, NVARCHAR)
Length | `LENGTH()` / `CHAR_LENGTH()` / `LEN()`
Substring | `SUBSTRING(str FROM 3 FOR 5)` / `SUBSTR()` / `SUBSTRING(str,3,5)`
Pattern search | `LIKE`, `ILIKE` (case‑insensitive PG), `SIMILAR TO`, `REGEXP_LIKE`
Replace & Trim | `REPLACE()`, `LTRIM/RTRIM`, `TRIM(BOTH 'x' FROM str)`
Regular expressions | PG: `~`, `~*`; Oracle: `REGEXP_SUBSTR`; SQL Server: CLR or `STRING_SPLIT`, `LIKE`, `PATINDEX`
Formatting | PG `TO_CHAR(num,'FM999,999')`; SQL Server `FORMAT()`

Performance Hints  
• Indexes ignore leading `%` in `LIKE '%foo'`; consider functional or trigram indexes.  
• Use `VARCHAR` over `TEXT` when indexing equality in MySQL (prefix length limit).

---

## 20.3  JSON Data  

### 20.3.1  Native Types  

Engine | Type | Highlights
-------|------|-----------
PostgreSQL | `JSON`, `JSONB` | `JSONB` stored in binary → indexable
MySQL 8 | `JSON` column | LOB internally, path indexing via generated columns
SQL Server | `NVARCHAR(max)` + JSON functions | No native type, but strict validation option

### 20.3.2  Core Operators / Functions  

Operation | PostgreSQL (`JSONB`) | MySQL 8 | SQL Server 2019+
----------|----------------------|---------|--------------
Get field | `data -> 'a'` (JSON), `data ->> 'a'` (text) | `JSON_EXTRACT(data,'$.a')` | `JSON_VALUE(data,'$.a')`
Update | `jsonb_set(data,'{a}', '"new"')` | `JSON_SET(data,'$.a','new')` | `JSON_MODIFY(...)`
Search by key/val | `@> '{"status":"new"}'` | `JSON_CONTAINS(data,'{"status":"new"}')` | `OPENJSON()` cross apply
Aggregate | `jsonb_agg(expr)` | `JSON_ARRAYAGG(expr)` | `FOR JSON PATH`
Relational view | `jsonb_to_recordset()` | `JSON_TABLE()` | `OPENJSON()` w/ schema

### 20.3.3  Indexing  

Target | Strategy
-------|----------
Containment (`@>`) PG | GIN on `jsonb` column
Specific path MySQL | Virtual column + B‑tree (`ALTER TABLE ADD COLUMN status VARCHAR(20) AS (data->>'$.status') STORED, ADD INDEX`)
Computed column SQL Server | `CREATE INDEX ON dbo.t (JSON_VALUE(data,'$.status'))`

Guidelines  
• Keep JSON for **flexible** or **sparse** attributes; avoid for hot relational keys.  
• Validate schema via check constraints or application layer.

---

## 20.4  XML Handling  

Feature | PostgreSQL | SQL Server | Oracle
--------|-----------|-----------|-------
Type | `xml` | `xml` | `XMLType`
XPath | `xpath('/root/value/text()', xmlcol)` | `.value('(/root/value/text())[1]','nvarchar(100)')` | `EXTRACTVALUE`
XQuery | PG add‑on (`pgxquery`) | `.query()`, `.nodes()` | Built‑in
Indexing | Functional GIN on `xpath` text | Primary XML & secondary PATH index | XMLIndex

Use Cases  
• Legacy SOAP payloads, configuration blobs, electronic data interchange (EDI).  
• Prefer JSON for modern apps unless strict XSD validation or XPath is mandatory.

---

## 20.5  Geospatial (GIS) Data  

Component | PostgreSQL + PostGIS | MySQL 8 | SQL Server
----------|---------------------|---------|-----------
Types | `geometry`, `geography` | `POINT`, `POLYGON`, `GEOMETRYCollection` | `geometry`, `geography`
Load WKT / WKB | `ST_GeomFromText()` | `ST_GeomFromText()` | `geometry::STGeomFromText()`
Key Functions | `ST_Distance`, `ST_Intersects`, `ST_Contains`, `ST_Buffer` | Same | Same
Index | GiST / SP‑GiST | R‑tree (InnoDB spatial) | Spatial index
SRID transform | `ST_Transform(geom, 4326)` | `ST_SRID()` + convert | `geom.Reproject()` (via SQLCLR)

Starter Example  

```sql
-- Nearest 10 cafés within 3 km of a point (Postgres)
SELECT name
FROM   cafes
WHERE  ST_DWithin(geog, ST_MakePoint(:lon,:lat)::geography, 3000)
ORDER  BY geog <-> ST_MakePoint(:lon,:lat)::geography
LIMIT 10;
```

---

## 20.6  Full‑Text Search (FTS)  

Aspect | PostgreSQL | MySQL | SQL Server
-------|-----------|-------|------------
Index Object | `tsvector` column + GIN | `FULLTEXT` index | Full‑Text Catalog / Index
Search Query | `to_tsquery('postgres & tutorial')` | `MATCH(col) AGAINST ('+postgres +tutorial' IN BOOLEAN MODE)` | `CONTAINS(col, ' "postgres" AND "tutorial" ')`
Ranking | `ts_rank(tsv, query)` | Relevance score returned | `RANK` in `CONTAINSTABLE`
Stemming & Dict | Built‑in dictionaries, custom | Simple natural language | Word‑breaker dlls, thesaurus

Tips  
1. Store lexemes in a generated `tsvector` column and index it (`CREATE INDEX ... gin(tsv)`).  
2. Refresh vector on INSERT/UPDATE via trigger.  
3. For multi‑lingual data, partition or add language code field.

---

## 20.7  Pivot & Unpivot  

Task | PostgreSQL | SQL Server | MySQL
-----|-----------|-----------|------
Rows→Columns | `crosstab()` (tablefunc ext) or filtered aggregation (`MAX(CASE WHEN key='k1' THEN val END)`) | `PIVOT` operator | Same filtered aggregation
Columns→Rows | `UNION ALL` or `JSON_TABLE()` | `UNPIVOT` operator | `UNION ALL`
Dynamic Pivot | Build SQL via string & `EXECUTE` | Dynamic SQL | Same

Example – monthly sales pivot (SQL Server)

```sql
SELECT *
FROM   (SELECT product_id, MONTH(order_date) AS m, qty FROM sales) AS src
PIVOT  (SUM(qty) FOR m IN ([1],[2],[3],[4],[5],[6],[7],[8],[9],[10],[11],[12])) AS p;
```

Performance  
• Aggregation pivot is fastest; PIVOT/UNPIVOT add optimizer layer but convenient.  
• Beware column explosion (Excel limits, BI tools).  

---

### Quick Reference Cheat‑Sheet  

Need | Function / Clause
----|-------------------
Add 5 business days | `ts + INTERVAL '7 days' - INTERVAL '2 days' * (EXTRACT(isodow FROM ts) > 5)` (PG)
Case‑insensitive search | `ILIKE '%foo%'` (PG) / `COLLATE utf8mb4_general_ci` (MySQL)
Extract JSON scalar | `data ->> 'price'` (PG) / `JSON_VALUE(data,'$.price')` (SQL Server)
Distance < 100 m | `ST_DWithin(geog, :point, 100)` (PostGIS)
Full‑text search rank | `MATCH(title,body) AGAINST (? IN NATURAL LANGUAGE MODE)` (MySQL)
Pivot rows | `MAX(CASE WHEN col='x' THEN val END)` pattern

---

## Best‑Practice Playbook  

1. **Pick native types** (JSONB, GEOGRAPHY) over `TEXT` blobs + string hacks.  
2. **Index early, audit often** – exotic types require explicit index creation and monitoring.  
3. **Validate & constrain** flexible structures (JSON schema, check constraints).  
4. **Off‑load heavy FTS & spatial** to separate replicas if OLTP performance suffers.  
5. **Use window functions or `GROUPING SETS`** before pivoting to minimize intermediate volume.  
6. **Benchmark with representative data**—cost models for JSON path or spatial joins differ by engine/version.  

---

Mastering these advanced data types and engine features allows you to solve non‑relational problems—text search, GIS, semi‑structured docs—directly in SQL while still benefiting from ACID guarantees, indexes, and role‑based security.