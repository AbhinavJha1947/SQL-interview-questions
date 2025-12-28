# Advanced SQL — Optimization, Analytics, and Administration

This section covers deep-dive SQL topics including Window Functions, Query Optimization, Execution Plans, Dynamic SQL, Advanced Data Types, and Database Administration.

---

## Table of Contents

1. [JSON & XML Support](#json--xml-support)
2. [Window Functions (Analytic)](#window-functions-analytic)
    - [Syntax building blocks](#syntax-building-blocks)
    - [ROWS vs RANGE — common interview question](#rows-vs-range--common-interview-question)
    - [Ranking, Offset, and Aggregate functions](#ranking-functions)
    - [Gaps and islands](#gaps-and-islands)
3. [Common Table Expressions (CTE) & HierarchyID](#common-table-expressions-cte--hierarchyid)
    - [Recursive CTEs & Cycle Detection](#recursive-cte--hierarchies-and-graphs)
    - [HierarchyID data type](#hierarchyid-data-type)
4. [Dynamic SQL](#dynamic-sql)
    - [EXEC vs sp_executesql & Parameter Sniffing](#exec-vs-sp_executesql--parameter-sniffing)
    - [Optional predicates and safety](#building-optional-predicates-safely)
5. [Performance & Optimization](#performance--optimization)
    - [Execution plans](#execution-plans--actual-vs-estimated)
    - [SARGability — Good vs Bad queries](#sargability--good-vs-bad-queries)
    - [Deadlocks — Handling & Detection](#deadlocks--handling--detection)
    - [Statistics & Partitioning](#statistics--column-and-index-stats)
    - [Index Maintenance — Rebuild vs Reorganize](#index-maintenance--rebuild-vs-reorganize)
    - [Query Store](#query-store-sql-server)
6. [Advanced Joins & Set Patterns](#advanced-joins--set-patterns)
7. [Advanced Data Types & Security](#advanced-data-types--security)
    - [Spatial, Arrays, JSONB](#spatial-types--geometrygeography)
    - [Advanced Security (DDM, Always Encrypted)](#advanced-security-ddm-always-encrypted)
8. [Administration: Import/Export, Backup, Monitoring](#administration-importexport-backup-monitoring)

---

## JSON & XML Support

### Parsing with JSON_VALUE / JSON_QUERY / OPENJSON
**Query (JSON):**
```sql
DECLARE @json NVARCHAR(MAX) = N'{"id": 1, "metadata": {"source": "web"}}';
SELECT JSON_VALUE(@json, '$.metadata.source') AS Source;

-- OPENJSON to table
SELECT * FROM OPENJSON(@json) WITH (id INT '$.id', source NVARCHAR(50) '$.metadata.source');
```

### Generating with FOR JSON & FOR XML
Convert query results into structured documents.

**Query (JSON):**
```sql
SELECT TOP 5 CustomerID, CustomerName
FROM dbo.Customers
FOR JSON PATH, ROOT('Customers');
```

**Query (XML nodes()):**
```sql
SELECT x.n.value('@id','int') AS Id, x.n.value('name[1]','nvarchar(100)') AS Name
FROM dbo.XmlData CROSS APPLY XmlCol.nodes('/items/item') AS x(n);
```

[⬆ Back to Top](#table-of-contents)

---

## Window Functions (Analytic)

### What are window functions
Compute values across a set of rows related to the current row without collapsing rows (unlike `GROUP BY`). 

**Use cases:** Rankings, running totals, moving averages, gaps-and-islands, percentiles, de-duplication.

### Syntax building blocks — OVER(), PARTITION BY, ORDER BY, FRAME
**Query (SQL Server):**
```sql
<func>() OVER (
  PARTITION BY <partition_cols>
  ORDER BY <order_cols>
  ROWS BETWEEN <frame_start> AND <frame_end>
) AS alias
```

**Notes:**
- Frame (`ROWS` or `RANGE`) narrows which ordered rows contribute to the function.

### ROWS vs RANGE — common interview question
- **ROWS:** Operates on physical row offsets from the current row. Predictable performance.
- **RANGE:** Operates on logically identical values based on the `ORDER BY` column. 
- **Performance:** `ROWS` is generally faster. `RANGE` with `UNBOUNDED PRECEDING` is the default but can be slower if there are many duplicate values in the ordering column.

**Query Example:**
```sql
-- Running total using ROWS (Faster)
SUM(Amount) OVER (ORDER BY OrderDate ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW)

-- Running total using RANGE (Default, potential speed trap)
SUM(Amount) OVER (ORDER BY OrderDate RANGE BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW)
```

### Ranking functions — ROW_NUMBER, RANK, DENSE_RANK, NTILE
**Query:**
```sql
SELECT o.CustomerID, o.OrderID, o.TotalAmount,
       ROW_NUMBER()  OVER (PARTITION BY o.CustomerID ORDER BY o.TotalAmount DESC) AS rn,
       RANK()        OVER (PARTITION BY o.CustomerID ORDER BY o.TotalAmount DESC) AS rnk,
       DENSE_RANK()  OVER (PARTITION BY o.CustomerID ORDER BY o.TotalAmount DESC) AS drnk,
       NTILE(4)      OVER (PARTITION BY o.CustomerID ORDER BY o.TotalAmount DESC) AS quartile
FROM dbo.Orders o;
```

**Note:** Use `ROW_NUMBER` for de-duplication (`rn=1`). `RANK` has gaps on ties; `DENSE_RANK` does not.

### Offset functions — LAG, LEAD, FIRST_VALUE, LAST_VALUE, NTH_VALUE
**Query:**
```sql
SELECT o.CustomerID, o.OrderDate, o.TotalAmount,
       LAG(o.TotalAmount, 1, 0)  OVER (PARTITION BY o.CustomerID ORDER BY o.OrderDate) AS prev_amount,
       LEAD(o.TotalAmount, 1, 0) OVER (PARTITION BY o.CustomerID ORDER BY o.OrderDate) AS next_amount,
       FIRST_VALUE(o.TotalAmount) OVER (PARTITION BY o.CustomerID ORDER BY o.OrderDate
           ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW) AS first_amt_to_date,
       LAST_VALUE(o.TotalAmount)  OVER (PARTITION BY o.CustomerID ORDER BY o.OrderDate
           ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING) AS last_amt_in_partition
FROM dbo.Orders o;
```

**Note:** `LAST_VALUE` requires a frame that reaches the end (`UNBOUNDED FOLLOWING`) or it returns the current row’s value.

### Windowed aggregates — SUM, AVG, MIN, MAX, COUNT with OVER()
**Query (running totals, moving average):**
```sql
-- Running total per customer by order date
SELECT o.CustomerID, o.OrderDate, o.TotalAmount,
       SUM(o.TotalAmount) OVER (
         PARTITION BY o.CustomerID
         ORDER BY o.OrderDate
         ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
       ) AS running_total,
       AVG(o.TotalAmount) OVER (
         PARTITION BY o.CustomerID
         ORDER BY o.OrderDate
         ROWS BETWEEN 2 PRECEDING AND CURRENT ROW
       ) AS moving_avg_3
FROM dbo.Orders o;
```

### Percentiles and distributions — PERCENT_RANK, CUME_DIST
**Query:**
```sql
SELECT ProductID, Price,
       PERCENT_RANK() OVER (ORDER BY Price) AS pct_rank,
       CUME_DIST()    OVER (ORDER BY Price) AS cume_dist
FROM dbo.Products;
```

### Conditional windowed counts — de-duplication pattern
**Query:**
```sql
WITH Ranked AS (
  SELECT *, ROW_NUMBER() OVER (PARTITION BY Email ORDER BY CreatedAt DESC) AS rn
  FROM dbo.Users
)
DELETE FROM Ranked WHERE rn > 1;
```

### Gaps and islands — identifying streaks
**Query (consecutive days with activity per user):**
```sql
WITH X AS (
  SELECT u.UserID, a.ActivityDate,
         DATEADD(day,
           -ROW_NUMBER() OVER (PARTITION BY u.UserID ORDER BY a.ActivityDate),
           a.ActivityDate) AS grp
  FROM dbo.UserActivity a
  JOIN dbo.Users u ON u.UserID = a.UserID
)
SELECT UserID,
       MIN(ActivityDate) AS start_date,
       MAX(ActivityDate) AS end_date,
       COUNT(*) AS days_in_streak
FROM X
GROUP BY UserID, grp
ORDER BY UserID, start_date;
```

### Performance and indexes for windows
- `ORDER BY` columns should be indexed to support window sorts.
- Add `PARTITION BY` columns to leading index positions when possible.
- Beware large frames with expensive sorts.

[⬆ Back to Top](#table-of-contents)

---

## Common Table Expressions (CTE) & HierarchyID

### What is a CTE and when to use it
A named result set defined just before a statement, scoped to that statement. Improves readability and enables recursion.

### Basic CTE syntax — WITH clause
**Query:**
```sql
WITH filtered_orders AS (
  SELECT * FROM dbo.Orders WHERE OrderDate >= DATEADD(day, -30, SYSUTCDATETIME())
)
SELECT CustomerID, COUNT(*) AS cnt
FROM filtered_orders
GROUP BY CustomerID
ORDER BY cnt DESC;
```

### Multiple CTEs — sequential definitions
**Query:**
```sql
WITH recent AS (
  SELECT * FROM dbo.Orders WHERE OrderDate >= DATEADD(day,-90,SYSUTCDATETIME())
), big AS (
  SELECT * FROM recent WHERE TotalAmount >= 1000
), by_customer AS (
  SELECT CustomerID, COUNT(*) AS cnt, SUM(TotalAmount) AS revenue
  FROM big GROUP BY CustomerID
)
SELECT * FROM by_customer ORDER BY revenue DESC;
```

### Recursive CTE — hierarchies and graphs
**Query (employee hierarchy, SQL Server):**
```sql
WITH EmpTree AS (
  -- Anchor: top-level managers
  SELECT e.EmployeeID, e.ManagerID, e.EmployeeName, 0 AS lvl, CAST(e.EmployeeName AS NVARCHAR(4000)) AS path
  FROM dbo.Employees e
  WHERE e.ManagerID IS NULL
  UNION ALL
  -- Recursive: attach direct reports
  SELECT c.EmployeeID, c.ManagerID, c.EmployeeName, p.lvl + 1,
             CAST(p.path + N' > ' + c.EmployeeName AS NVARCHAR(4000))
  FROM dbo.Employees c
  JOIN EmpTree p ON p.EmployeeID = c.ManagerID
)
SELECT * FROM EmpTree ORDER BY path OPTION (MAXRECURSION 100);
```

### Cycle detection — preventing infinite loops
**Query (PostgreSQL idiom):**
```sql
WITH RECURSIVE GraphCTE AS (
  SELECT from_id, to_id, ARRAY[from_id] AS visited
  FROM edges
  UNION ALL
  SELECT g.from_id, e.to_id, visited || e.to_id
  FROM GraphCTE g
  JOIN edges e ON e.from_id = g.to_id
  WHERE NOT (e.to_id = ANY(visited))
)
SELECT * FROM GraphCTE;
```

### HierarchyID data type
Used to store and query hierarchical data efficiently within a single column.

**Query Example:**
```sql
CREATE TABLE OrgChart (
    Node HIERARCHYID PRIMARY KEY,
    EmployeeName NVARCHAR(100)
);

-- Insert top level
INSERT INTO OrgChart (Node, EmployeeName) VALUES ('/', 'CEO');
-- Insert report to CEO
DECLARE @CEO HIERARCHYID = (SELECT Node FROM OrgChart WHERE EmployeeName = 'CEO');
INSERT INTO OrgChart (Node, EmployeeName) VALUES (@CEO.GetDescendant(NULL, NULL), 'VP');
```

[⬆ Back to Top](#table-of-contents)

---

## Dynamic SQL

### What and why
Constructing SQL strings at runtime to handle variable filters or dynamic object names. **Always parameterize user input** to prevent SQL injection.

### EXEC vs sp_executesql — Parameter Sniffing
`sp_executesql` is superior as it supports parameterization, improving security and plan reuse. 

**Parameter Sniffing:** Occurs when the SQL optimizer creates an execution plan based on the specific parameter values passed during the first compilation. This plan might be inefficient for other values.

**Fixing it:**
- Use `OPTION (RECOMPILE)` for highly skewed data.
- Use `OPTIMIZE FOR (@Param = Value)` or `OPTIMIZE FOR UNKNOWN`.
- Local variables (can sometimes hide the value from the optimizer).

**Query:**
```sql
DECLARE @sql NVARCHAR(MAX) = N'SELECT * FROM Orders WHERE ClientID = @id';
EXEC sp_executesql @sql, N'@id INT', @id = 101 OPTION (RECOMPILE);
```

### Building optional predicates safely
**Query:**
```sql
DECLARE @sql NVARCHAR(MAX) = N'SELECT * FROM dbo.Users WHERE 1=1';
IF @IsActive IS NOT NULL SET @sql += N' AND IsActive = @IsActive';
IF @EmailDomain IS NOT NULL SET @sql += N' AND Email LIKE ''%'' + @EmailDomain';
EXEC sp_executesql @sql, N'@IsActive BIT, @EmailDomain NVARCHAR(100)', @IsActive, @EmailDomain;
```

### Dynamic PIVOT and dynamic ORDER BY
**Query:**
```sql
DECLARE @cols NVARCHAR(MAX) = (
  SELECT STRING_AGG(QUOTENAME([Month]), ',')
  FROM (SELECT DISTINCT DATENAME(month, OrderDate) AS [Month] FROM dbo.Sales) d
);
DECLARE @sql NVARCHAR(MAX) = N'
SELECT ProductName, ' + @cols + N'
FROM (
  SELECT ProductName, DATENAME(month, OrderDate) AS [Month], Quantity
  FROM dbo.Sales
) s
PIVOT (SUM(Quantity) FOR [Month] IN (' + @cols + N')) p';
EXEC sp_executesql @sql;
```

[⬆ Back to Top](#table-of-contents)

---

## Performance & Optimization

### Execution plans — actual vs estimated
Actual plans include runtime metrics; estimated plans use statistics to predict cardinalities.

**Commands (SQL Server):**
```sql
-- Estimated plan
SET SHOWPLAN_XML ON;
-- Actual plan
SET STATISTICS XML ON;
```

### SARGability — Good vs Bad queries
SARGable (Search ARGumentable) queries allow the optimizer to use index seeks instead of full scans.

- **Bad (Non-SARGable):** `WHERE YEAR(OrderDate) = 2024` (Wrapped in function)
- **Good (SARGable):** `WHERE OrderDate >= '2024-01-01' AND OrderDate < '2025-01-01'`

- **Bad:** `WHERE Name LIKE '%Smith'` (Leading wildcard)
- **Good:** `WHERE Name LIKE 'Smith%'`

### Deadlocks — Handling & Detection
A deadlock occurs when two processes hold locks that the other needs, creating a cycle.

- **Detection:** SQL Server automatically detects deadlocks and kills one "victim" based on deadlock priority.
- **Monitoring:** Use **Extended Events (`xml_deadlock_report`)** or Profiler.
- **Resolution:**
    - Access objects in the same order across all transactions.
    - Keep transactions short.
    - Use lower isolation levels (e.g., Read Committed Snapshot) if appropriate.
    - Implement retry logic in application code.

[⬆ Back to Top](#table-of-contents)

### Query hints — use sparingly
Overrides the optimizer. Examples include `FORCESEEK`, `OPTION (RECOMPILE)`, and `MAXDOP`.

### Statistics — column and index stats
Crucial for cardinality estimates.
**Query:**
```sql
UPDATE STATISTICS dbo.Orders WITH FULLSCAN; -- SQL Server
ANALYZE VERBOSE public.orders; -- PostgreSQL
```

### Index Maintenance — Rebuild vs Reorganize
- **Reorganize:** (< 30% fragmentation) Online operation, defragments leaf level of indexes.
- **Rebuild:** (> 30% fragmentation) Can be offline or online (Enterprise), drops and recreates the index entirely.

**Query:**
```sql
ALTER INDEX IX_Orders_Date ON dbo.Orders REORGANIZE;
ALTER INDEX IX_Orders_Date ON dbo.Orders REBUILD WITH (ONLINE = ON);
```

### Partitioning — range, list, hash
Splits large tables horizontally to improve maintenance and pruning.
**Query (SQL Server):**
```sql
CREATE PARTITION FUNCTION pf_OrderDate (DATE) AS RANGE RIGHT FOR VALUES ('2024-01-01','2025-01-01');
CREATE PARTITION SCHEME ps_Orders AS PARTITION pf_OrderDate ALL TO ([PRIMARY]);
```

### Query Store (SQL Server)
Captures query texts, plans, and runtime stats over time for regression analysis.

[⬆ Back to Top](#table-of-contents)

---

## Advanced Joins & Set Patterns

### LATERAL (PostgreSQL) / APPLY (SQL Server)
Allows subqueries to reference columns from preceding tables.

**Query (SQL Server):**
```sql
SELECT o.OrderID, x.TopLineTotal
FROM dbo.Orders o
CROSS APPLY (
  SELECT TOP 1 l.LineTotal AS TopLineTotal
  FROM dbo.OrderLines l WHERE l.OrderID = o.OrderID ORDER BY l.LineTotal DESC
) x;
```

### Anti-join / Semi-join patterns
**Anti-join (NOT EXISTS):**
```sql
SELECT c.CustomerID FROM dbo.Customers c
WHERE NOT EXISTS (SELECT 1 FROM dbo.Orders o WHERE o.CustomerID = c.CustomerID);
```

**Semi-join (EXISTS):**
```sql
SELECT c.CustomerID FROM dbo.Customers c
WHERE EXISTS (SELECT 1 FROM dbo.Orders o WHERE o.CustomerID = c.CustomerID);
```

[⬆ Back to Top](#table-of-contents)

---

## Advanced Data Types & Security

### Arrays, HSTORE, JSONB (PostgreSQL)
**JSONB Query:**
```sql
SELECT id FROM events WHERE meta @> '{"type":"click","source":"web"}';
```

### Spatial types — GEOMETRY/GEOGRAPHY
**Query (SQL Server):**
```sql
SELECT Id, Name FROM Places
WHERE Loc.STDistance(geography::Point(37.819, -122.478, 4326)) <= 5000;
```

### User-defined types (UDT) & ENUM
**ENUM Query (PostgreSQL):**
```sql
CREATE TYPE order_status AS ENUM ('new','processing','shipped','cancelled');
```

### Advanced Security (DDM, Always Encrypted)
- **Dynamic Data Masking (DDM):** Limits sensitive data exposure by masking it in the result set for non-privileged users.
- **Always Encrypted:** Data is encrypted at the client-side; SQL Server never see the plaintext values, protecting against high-privileged but unauthorized access (like DBAs).

**Query (DDM Example):**
```sql
ALTER TABLE Users
ALTER COLUMN Email ADD MASKED WITH (FUNCTION = 'email()');
```

[⬆ Back to Top](#table-of-contents)

---

## Sequences & Identity
- **SQL Server:** `IDENTITY(1,1)` or `CREATE SEQUENCE`.
- **PostgreSQL:** `SERIAL` or `GENERATED ALWAYS AS IDENTITY`.
- **MySQL:** `AUTO_INCREMENT`.

---

## Data Import/Export
- **SQL Server:** `BULK INSERT`, `bcp` utility.
- **PostgreSQL:** `COPY` command.
- **MySQL:** `LOAD DATA INFILE`.

---

## Security & Permissions
- **RBAC:** `GRANT`, `REVOKE`, `DENY`.
- **Row-Level Security (RLS):** Filter rows based on user context.
- **Audit:** Track sensitive changes using triggers or built-in audit features.

---

## Backup & Recovery
- **SQL Server:** `BACKUP DATABASE`, recovery models (Simple, Full, Bulk-Logged).
- **PostgreSQL:** `pg_dump`, `pg_restore`.
- **MySQL:** `mysqldump`.

---

## Monitoring & Maintenance
- **System Views:** `information_schema`, `sys.*`, `pg_catalog`.
- **Index Maintenance:** Rebuild (offline/online) or Reorganize based on fragmentation.
- **Vacuum (PostgreSQL):** Reclamation of storage from deleted/updated rows.

[⬆ Back to Top](#table-of-contents)
