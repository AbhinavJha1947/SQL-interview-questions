# Advanced SQL — Optimization, Analytics, and Administration

This section covers deep-dive SQL topics including Window Functions, Query Optimization, Execution Plans, Dynamic SQL, Advanced Data Types, and Database Administration.

---

## Table of Contents

1. [JSON & XML Support](#json--xml-support)
2. [Window Functions (Analytic)](#window-functions-analytic)
    - [What are window functions](#what-are-window-functions)
    - [Syntax building blocks](#syntax-building-blocks)
    - [Ranking functions](#ranking-functions)
    - [Offset functions](#offset-functions)
    - [Windowed aggregates](#windowed-aggregates)
    - [Percentiles and distributions](#percentiles-and-distributions)
    - [De-duplication pattern](#de-duplication-pattern)
    - [Gaps and islands](#gaps-and-islands)
    - [Performance and indexes](#performance-and-indexes-for-windows)
3. [Common Table Expressions (CTE)](#common-table-expressions-cte)
    - [Advanced CTE Usage](#advanced-cte-usage)
    - [Recursive CTEs & Cycle Detection](#recursive-cte--hierarchies-and-graphs)
4. [Dynamic SQL](#dynamic-sql)
    - [EXEC vs sp_executesql](#exec-vs-sp_executesql)
    - [Optional predicates and safety](#building-optional-predicates-safely)
    - [Dynamic PIVOT and ORDER BY](#dynamic-pivot-and-dynamic-order-by)
    - [Permissions and injection safety](#permissions-and-injection-safety)
5. [Performance & Optimization](#performance--optimization)
    - [Execution plans](#execution-plans--actual-vs-estimated)
    - [Query optimization guidelines](#query-optimization--practical-guidelines)
    - [Query hints](#query-hints--use-sparingly)
    - [Statistics](#statistics--column-and-index-stats)
    - [Partitioning](#partitioning--range-list-hash)
    - [Query Store](#query-store-sql-server)
    - [Query rewrites](#query-rewrites--common-transformations)
6. [Pivot & Unpivot](#pivot--unpivot)
7. [Advanced Joins & Set Patterns](#advanced-joins--set-patterns)
    - [LATERAL / APPLY](#lateral-postgresql)
    - [Anti-join / Semi-join patterns](#anti-join-patterns)
8. [Advanced Data Types](#advanced-data-types)
    - [Arrays, HSTORE, JSONB](#arrays-postgresql)
    - [Spatial types](#spatial-types--geometrygeography)
    - [User-defined types & ENUM](#user-defined-types-udt)
9. [Sequences & Identity](#sequences--identity)
10. [Data Import/Export](#data-importexport)
11. [Security & Permissions](#security--permissions)
12. [Backup & Recovery](#backup--recovery)
13. [Monitoring & Maintenance](#database-monitoring--maintenance)

---

## JSON & XML Support

### JSON_VALUE / JSON_QUERY / OPENJSON
**Query:**
```sql
SELECT JSON_VALUE(Meta, '$.source') AS Source
FROM dbo.Events;
```

### XML nodes()
**Query:**
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
- `PARTITION BY` groups rows into independent windows.
- `ORDER BY` defines sequence within each partition.
- Frame (`ROWS` or `RANGE`) narrows which ordered rows contribute to the function.

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

## Common Table Expressions (CTE)

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

[⬆ Back to Top](#table-of-contents)

---

## Dynamic SQL

### What and why
Constructing SQL strings at runtime to handle variable filters or dynamic object names. **Always parameterize user input** to prevent SQL injection.

### sp_executesql — parameterized dynamic SQL (SQL Server)
Executes a parameterized statement for plan reuse and safety.

**Query:**
```sql
DECLARE @sql NVARCHAR(MAX) = N'
  SELECT OrderID, CustomerID, TotalAmount
  FROM dbo.Orders
  WHERE (@custId IS NULL OR CustomerID = @custId)
    AND OrderDate >= @since
  ORDER BY OrderDate DESC';
DECLARE @params NVARCHAR(200) = N'@custId INT, @since DATETIME2';
EXEC sp_executesql @sql, @params, @custId = @CustomerID, @since = @SinceDate;
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

### Query optimization — practical guidelines
- **Select only needed columns.** Avoid `SELECT *`.
- **Filter early and sargably.** Avoid functions on indexed columns in predicates.
- **Prefer set-based operations.** Replace cursors with joins and window functions.
- **Watch for OR-heavy predicates.** Consider `UNION ALL`.

### Query hints — use sparingly
Overrides the optimizer. Examples include `FORCESEEK`, `OPTION (RECOMPILE)`, and `MAXDOP`.

### Statistics — column and index stats
Crucial for cardinality estimates.
**Query:**
```sql
UPDATE STATISTICS dbo.Orders WITH FULLSCAN; -- SQL Server
ANALYZE VERBOSE public.orders; -- PostgreSQL
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

## Pivot & Unpivot

### PIVOT (SQL Server)
**Query:**
```sql
SELECT ProductName, [Jan],[Feb],[Mar]
FROM (
  SELECT ProductName, DATENAME(month, OrderDate) AS [Month], Quantity
  FROM dbo.Sales
) s
PIVOT (SUM(Quantity) FOR [Month] IN ([Jan],[Feb],[Mar])) p;
```

### UNPIVOT (SQL Server)
**Query:**
```sql
SELECT ProductName, [Month], Qty
FROM dbo.SalesByMonth
UNPIVOT (Qty FOR [Month] IN ([Jan],[Feb],[Mar])) AS u;
```

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

## Advanced Data Types

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
