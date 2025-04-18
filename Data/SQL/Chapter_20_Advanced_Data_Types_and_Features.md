# Chapter 20 – Advanced Data Types and Features  

Modern RDBMSs support rich data types and powerful functions for dates, strings, semi‑structured data, geospatial analytics, full‑text search, and pivoting. This chapter surveys these advanced capabilities.

---

## 20.1 Working with Date and Time Functions  
### Core Types  
- DATE: calendar date  
- TIME [ (p) ] [ WITH TIME ZONE ]  
- TIMESTAMP [ (p) ] [ WITH TIME ZONE ]  
- INTERVAL: span of time  

### Common Functions & Operations  
- CURRENT_DATE, CURRENT_TIME, CURRENT_TIMESTAMP  
- NOW() (alias for current timestamp)  
- EXTRACT(field FROM source)  
- DATE_TRUNC('unit', timestamp)  
- AGE(timestamp1, timestamp2) (PostgreSQL)  
- Arithmetic: `timestamp + INTERVAL '1 day'`, `date2 - date1` → INTERVAL  
- Time‑zone conversion: `AT TIME ZONE 'UTC'`  

Example:  
```sql
SELECT
  order_id,
  order_date,
  DATE_TRUNC('month', order_date) AS month_bucket,
  EXTRACT(DOW FROM order_date)     AS day_of_week,
  order_date + INTERVAL '30 days'  AS due_date
FROM orders;
```

**Best Practices**  
- Store all times in UTC; convert in application/UI.  
- Use DATE_TRUNC for grouping by periods.  
- Beware leap‑seconds and DST shifts; prefer TIMESTAMP WITH TIME ZONE for global apps.

---

## 20.2 Working with String Functions  
### Common Manipulations  
- CONCAT(a, b, …) or `a || b`  
- SUBSTRING(str FROM pos [ FOR len ])  
- LEFT/RIGHT (SQL Server, MySQL)  
- TRIM([LEADING|TRAILING|BOTH] chars FROM str)  
- UPPER(), LOWER()  
- LENGTH(), CHAR_LENGTH()  
- POSITION(sub IN str) or INSTR()  
- REPLACE(str, from, to)  
- LPAD/RPAD (padding)  
- FORMAT() (SQL Server, PostgreSQL)  

Example:  
```sql
SELECT
  username,
  UPPER(SUBSTRING(username FROM 1 FOR 1)) || LOWER(SUBSTRING(username FROM 2))
    AS proper_name,
  LENGTH(bio) AS bio_length,
  REPLACE(bio, '\n', ' ') AS bio_single_line
FROM users;
```

**Advanced**  
- Regular expressions: `REGEXP_REPLACE`, `REGEXP_MATCHES`, `REGEXP_LIKE`.  
- Collation‑aware comparisons for international text.

---

## 20.3 Handling JSON Data in SQL  
### Data Types  
- JSON (textual)  
- JSONB (binary, indexed, faster in PostgreSQL)  

### Insertion & Retrieval  
```sql
-- Store
INSERT INTO events(data) VALUES ('{"user":123,"action":"login"}');

-- Query
SELECT
  data->>'user'      AS user_id,
  (data->'meta')->>'ip' AS ip_address
FROM events
WHERE data->>'action' = 'login';
```

### Functions & Operators (PostgreSQL)  
- `->` (JSON field), `->>` (text), `#>` (path), `#>>`  
- `jsonb_set()`, `jsonb_insert()`  
- `jsonb_array_elements()`, `jsonb_each()`  

### Indexing  
```sql
-- GIN index for containment and key existence
CREATE INDEX idx_events_data ON events USING GIN (data jsonb_path_ops);
```

**Use Cases**  
- Schemaless attributes (user metadata, telemetry).  
- Semi‑structured logs.  

---

## 20.4 Handling XML Data in SQL  
### Storage & Parsing  
- XML type (stores well-formed documents).  
- `EXTRACT()` / `XMLQUERY()` / `XMLTABLE()` integration.  

Example (PostgreSQL):  
```sql
SELECT
  (xpath('/order/customer/text()', xmlcol))[1]::text AS customer,
  xmlcol::xmltype
FROM xml_orders;
```

**Key Functions**  
- `XMLPARSE()`, `XMLSERIALIZE()`  
- `XMLFOREST()`, `XMLELEMENT()`  
- `XMLTABLE()` to shred XML into relational rows.  

---

## 20.5 Geospatial Data Types and Functions (Introduction)  
### Core Types (PostGIS)  
- `geometry` (planar)  
- `geography` (ellipsoidal)  

### Basic Operations  
- `ST_Point(x,y)`, `ST_MakeLine()`, `ST_MakePolygon()`  
- Distance/Containment: `ST_Distance()`, `ST_Within()`, `ST_Contains()`  
- Buffers and transformations: `ST_Buffer()`, `ST_Transform(geom, SRID)`  

### Indexing  
```sql
CREATE INDEX idx_locations_geom
  ON locations USING GIST (geom);
```

**Example**  
```sql
SELECT name
FROM parks
WHERE ST_DWithin(geom::geography,
                 ST_SetSRID(ST_MakePoint(lon, lat), 4326)::geography,
                 5000);  -- within 5 km
```

---

## 20.6 Full-Text Search Capabilities  
### Concepts (PostgreSQL)  
- `tsvector`: document representation  
- `tsquery`: search query  

### Core Functions  
- `to_tsvector('english', text)`  
- `to_tsquery('english', 'foo & bar')` or `plainto_tsquery()`  
- Ranking: `ts_rank()`, `ts_rank_cd()`  

### Indexing  
```sql
ALTER TABLE articles
  ADD COLUMN tsv tsvector GENERATED ALWAYS AS
    (to_tsvector('english', coalesce(title,'') || ' ' || body)) STORED;
CREATE INDEX idx_articles_tsv ON articles USING GIN (tsv);
```

**Search Example**  
```sql
SELECT id, ts_rank(tsv, plainto_tsquery('english','lorem ipsum')) AS rank
FROM articles
WHERE tsv @@ plainto_tsquery('english','lorem ipsum')
ORDER BY rank DESC;
```

---

## 20.7 Pivot and Unpivot Operations  
### Pivot via Aggregate + FILTER/CASE  
```sql
SELECT
  region,
  SUM(amount) FILTER (WHERE month = 'Jan') AS jan_sales,
  SUM(amount) FILTER (WHERE month = 'Feb') AS feb_sales
FROM sales
GROUP BY region;
```

### Unpivot via UNION ALL or LATERAL  
```sql
SELECT region, month, sales
FROM (
  SELECT region, jan_sales, feb_sales FROM pivoted_sales
) ps
UNPIVOT (
  sales FOR month IN (jan_sales AS 'Jan', feb_sales AS 'Feb')
) AS unpvt;
```
Or using `UNION ALL`:
```sql
SELECT region, 'Jan' AS month, jan_sales AS sales FROM pivoted
UNION ALL
SELECT region, 'Feb', feb_sales FROM pivoted;
```

### PostgreSQL crosstab  
```sql
-- Requires tablefunc extension
SELECT *
FROM crosstab(
  'SELECT region, month, amount FROM sales ORDER BY 1,2'
) AS ct (region text, jan numeric, feb numeric, mar numeric);
```

---

By leveraging these advanced data types, functions, and indexing strategies, you unlock powerful capabilities—time‑series analysis, text search, spatial queries, and dynamic cross‑tabulation—directly in SQL, reducing reliance on external processing layers.