# Intermediate SQL — Advanced Queries and Objects

This section covers more advanced SQL concepts including various types of Joins, Set Operations, Subqueries, Views, Stored Procedures, Functions, Triggers, and Cursors.

---

## Table of Contents

1. [Joins (SQL Server)](#joins-sql-server)
    - [INNER JOIN — matching rows only](#inner-join--matching-rows-only)
    - [LEFT (OUTER) JOIN — keep all left rows](#left-outer-join--keep-all-left-rows)
    - [RIGHT (OUTER) JOIN — keep all right rows](#right-outer-join--keep-all-right-rows)
    - [FULL (OUTER) JOIN — union of left and right](#full-outer-join--union-of-left-and-right)
    - [CROSS JOIN — cartesian product](#cross-join--cartesian-product)
    - [SELF JOIN — relate rows within a table](#self-join--relate-rows-within-a-table)
    - [Join conditions — ON](#join-conditions--on)
    - [Multiple table joins](#multiple-table-joins)
    - [Join performance tips](#join-performance-tips)
2. [Set Operations (SQL Server)](#set-operations-sql-server)
    - [UNION — distinct union](#union--distinct-union)
    - [UNION ALL — append including duplicates](#union-all--append-including-duplicates)
    - [UNION vs UNION ALL — behavior and performance](#union-vs-union-all--behavior-and-performance)
    - [INTERSECT — common rows](#intersect--common-rows)
    - [EXCEPT — rows in A not in B](#except--rows-in-a-not-in-b)
3. [Subqueries (SQL Server)](#subqueries-sql-server)
    - [Scalar subquery — single value per row](#scalar-subquery--single-value-per-row)
    - [Correlated subquery — references outer row](#correlated-subquery--references-outer-row)
    - [IN / NOT IN with subqueries](#in--not-in-with-subqueries)
    - [IN vs EXISTS — common interview question](#in-vs-exists--common-interview-question)
    - [ALL / ANY / SOME](#all--any--some)
    - [Derived tables (subquery in FROM)](#derived-tables-subquery-in-from)
4. [Views (SQL Server)](#views-sql-server)
5. [Stored Procedures & Functions](#stored-procedures-sql-server)
    - [Stored Procedures (usp)](#stored-procedures-sql-server)
    - [User-Defined Functions (udf)](#user-defined-functions-sql-server)
    - [SP vs UDF — when to use which](#sp-vs-udf--when-to-use-which)
6. [Triggers (SQL Server)](#triggers-sql-server)
7. [Cursors (SQL Server)](#cursors-sql-server)
8. [Common Table Expressions (CTE)](#common-table-expressions-cte)
9. [Temporary Tables & Table Variables](#temporary-tables--table-variables-sql-server)
10. [Temporal Tables & CTAS](#temporal-tables--ctas)
11. [PIVOT / UNPIVOT](#pivot--unpivot)
12. [APPLY operators (SQL Server)](#apply-operators-sql-server)
13. [Transaction Isolation Levels](#transaction-isolation-levels)
14. [Index Basics: Clustered vs Non-Clustered](#index-basics-clustered-vs-non-clustered)

---

## Joins (SQL Server)

### INNER JOIN — matching rows only
Return rows where the join predicate matches on both sides.

**Query:**
```sql
SELECT o.OrderID, c.CustomerName, o.OrderDate
FROM dbo.Orders AS o
INNER JOIN dbo.Customers AS c
  ON c.CustomerID = o.CustomerID;
```

**Implementation Tip:** Index both sides on join keys to enable seeks and proper join choices.

### LEFT (OUTER) JOIN — keep all left rows
**Query:**
```sql
SELECT c.CustomerID, c.CustomerName, o.OrderID
FROM dbo.Customers AS c
LEFT JOIN dbo.Orders AS o
  ON o.CustomerID = c.CustomerID;
```

### RIGHT (OUTER) JOIN — keep all right rows
**Query:**
```sql
SELECT c.CustomerID, c.CustomerName, o.OrderID
FROM dbo.Customers AS c
RIGHT JOIN dbo.Orders AS o
  ON o.CustomerID = c.CustomerID;
```

### FULL (OUTER) JOIN — union of left and right
**Query:**
```sql
SELECT a.KeyCol, a.ValA, b.ValB
FROM dbo.A AS a
FULL OUTER JOIN dbo.B AS b
  ON b.KeyCol = a.KeyCol;
```

### CROSS JOIN — cartesian product
**Query:**
```sql
SELECT d.Date, p.ProductName
FROM dbo.DimDate AS d
CROSS JOIN dbo.Products AS p;
```

### SELF JOIN — relate rows within a table
**Query:**
```sql
SELECT e.EmployeeID, e.EmployeeName, m.EmployeeName AS ManagerName
FROM dbo.Employees AS e
LEFT JOIN dbo.Employees AS m
  ON m.EmployeeID = e.ManagerID;
```

### Join conditions — ON
SQL Server requires `ON` with qualified columns. `USING` is not supported in T-SQL.

### Multiple table joins
**Query:**
```sql
SELECT o.OrderID, c.CustomerName, p.ProductName, l.Quantity
FROM dbo.Orders o
JOIN dbo.Customers c ON c.CustomerID = o.CustomerID
JOIN dbo.OrderLines l ON l.OrderID = o.OrderID
JOIN dbo.Products p ON p.ProductID = l.ProductID;
```

### Join performance tips
- Match data types and collations on join keys.
- Add nonclustered indexes on foreign keys; `INCLUDE` selected columns to cover queries.
- Keep predicates sargable. Avoid wrapping indexed columns in functions.

[⬆ Back to Top](#table-of-contents)

---

## Set Operations (SQL Server)

### UNION — distinct union
**Query:**
```sql
SELECT Email FROM dbo.Users_US
UNION
SELECT Email FROM dbo.Users_EU;
```

### UNION ALL — append including duplicates
**Query:**
```sql
SELECT Email FROM dbo.Users_US
UNION ALL
SELECT Email FROM dbo.Users_EU;
```

### UNION vs UNION ALL — behavior and performance
- **UNION:** Merges result sets and performs a **distinct** operation to remove duplicates. This involves a sort/hash operation which can be expensive.
- **UNION ALL:** Simply appends the result sets. No duplicate checking, thus significantly **faster**.

**Interview Tip:** Always prefer `UNION ALL` unless you explicitly need to filter out duplicate rows.

### INTERSECT — common rows
**Query:**
```sql
SELECT Email FROM dbo.Users_US
INTERSECT
SELECT Email FROM dbo.Users_EU;
```

### EXCEPT — rows in A not in B
**Query:**
```sql
SELECT Email FROM dbo.Users_US
EXCEPT
SELECT Email FROM dbo.Users_EU;
```

**Note:** Operand queries must project the same number of columns with compatible types.

[⬆ Back to Top](#table-of-contents)

---

## Subqueries (SQL Server)

### Scalar subquery — single value per row
**Query:**
```sql
SELECT o.OrderID,
       (SELECT SUM(l.LineTotal)
        FROM dbo.OrderLines l
        WHERE l.OrderID = o.OrderID) AS OrderTotal
FROM dbo.Orders o;
```

### Correlated subquery — references outer row
**Query:**
```sql
SELECT c.CustomerID, c.CustomerName
FROM dbo.Customers c
WHERE EXISTS (
  SELECT 1 FROM dbo.Orders o
  WHERE o.CustomerID = c.CustomerID
    AND o.OrderDate >= DATEADD(day, -30, SYSUTCDATETIME())
);
```

### IN / NOT IN with subqueries
**Query:**
```sql
SELECT *
FROM dbo.Orders
WHERE CustomerID IN (SELECT CustomerID FROM dbo.VIPCustomers);
```

**Implementation Tip:** Prefer `NOT EXISTS` over `NOT IN` if the subquery can return `NULL`s.

### IN vs EXISTS — common interview question
- **IN:** Works well for small, static lists or when you need to compare a value against a set.
- **EXISTS:** Usually more efficient for large datasets because it follows "Short-circuit" logic—it stops as soon as a match is found. 
- **NULL Handling:** `NOT IN` returns no results if the subquery contains a `NULL`. `NOT EXISTS` handles `NULL` values correctly.

### ALL / ANY / SOME
**Query:**
```sql
SELECT *
FROM dbo.Products
WHERE Price >= ALL (SELECT Price FROM dbo.Products WHERE CategoryID = 10);
```

### Derived tables (subquery in FROM)
**Query:**
```sql
SELECT d.CustomerID, d.OrderCount
FROM (
  SELECT CustomerID, COUNT(*) AS OrderCount
  FROM dbo.Orders
  GROUP BY CustomerID
) AS d
WHERE d.OrderCount >= 5;
```

[⬆ Back to Top](#table-of-contents)

---

## Views (SQL Server)

### CREATE VIEW — virtual table
**Query:**
```sql
CREATE OR ALTER VIEW dbo.v_RecentOrders
AS
SELECT o.OrderID, o.CustomerID, o.OrderDate, o.TotalAmount
FROM dbo.Orders o
WHERE o.OrderDate >= DATEADD(day, -30, CAST(SYSUTCDATETIME() AS date));
```

### Updatable views — constraints
Single-base-table views without `DISTINCT`, aggregates, `GROUP BY`, `TOP`, `UNION` are typically updatable.

### WITH SCHEMABINDING and indexed views
**Query:**
```sql
CREATE VIEW dbo.v_SalesByDay WITH SCHEMABINDING AS
SELECT CAST(o.OrderDate AS date) AS SalesDate,
       COUNT_BIG(*) AS Orders
FROM dbo.Orders o
GROUP BY CAST(o.OrderDate AS date);
GO
CREATE UNIQUE CLUSTERED INDEX IX_v_SalesByDay ON dbo.v_SalesByDay(SalesDate);
```

**Note:** Requires two-part names, `COUNT_BIG` for aggregates, and specific SET options (`ANSI_NULLS`, `QUOTED_IDENTIFIER ON`, etc.).

### WITH CHECK OPTION — enforce view predicate
**Query:**
```sql
CREATE OR ALTER VIEW dbo.v_ActiveUsers
AS SELECT * FROM dbo.Users WHERE IsActive = 1
WITH CHECK OPTION;
```

### ALTER VIEW / DROP VIEW
**Query:**
```sql
ALTER VIEW dbo.v_RecentOrders AS SELECT TOP 100 PERCENT * FROM dbo.Orders;
DROP VIEW IF EXISTS dbo.v_RecentOrders;
```

[⬆ Back to Top](#table-of-contents)

---

## Stored Procedures & Functions

### Stored Procedures (usp) — reusable T‑SQL
**Query:**
```sql
CREATE OR ALTER PROCEDURE dbo.usp_GetCustomerOrders
  @CustomerID INT,
  @SinceDate  DATE = NULL
AS
BEGIN
  SET NOCOUNT ON;
  SELECT o.OrderID, o.OrderDate, o.TotalAmount
  FROM dbo.Orders o
  WHERE o.CustomerID = @CustomerID
    AND (@SinceDate IS NULL OR o.OrderDate >= @SinceDate)
  ORDER BY o.OrderDate DESC;
END;
```

### Parameters, output, return codes
**Query:**
```sql
CREATE OR ALTER PROCEDURE dbo.usp_CountOrders
  @CustomerID INT,
  @Count INT OUTPUT
AS
BEGIN
  SELECT @Count = COUNT(*) FROM dbo.Orders WHERE CustomerID = @CustomerID;
  RETURN 0;
END;
```

**Implementation Tip:** Capture output with `EXEC dbo.usp_CountOrders @CustomerID=1, @Count=@c OUTPUT`.

### TRY...CATCH error handling
**Query:**
```sql
BEGIN TRY
  BEGIN TRAN;
  -- work
  COMMIT;
END TRY
BEGIN CATCH
  IF XACT_STATE() <> 0 ROLLBACK;
  THROW; -- rethrow
END CATCH;
```

[⬆ Back to Top](#table-of-contents)

---

### User-Defined Functions (udf)

### Scalar UDF — returns a single value
**Query:**
```sql
CREATE OR ALTER FUNCTION dbo.ufn_SafeDivide(@a DECIMAL(18,4), @b DECIMAL(18,4))
RETURNS DECIMAL(18,4)
AS
BEGIN
  RETURN CASE WHEN @b = 0 THEN NULL ELSE @a / @b END;
END;
```

### Inline Table-Valued Function (iTVF)
**Query:**
```sql
CREATE OR ALTER FUNCTION dbo.ufn_OrdersSince(@since DATE)
RETURNS TABLE
AS RETURN (
  SELECT OrderID, CustomerID, OrderDate
  FROM dbo.Orders
  WHERE OrderDate >= @since
);
```

**Implementation Tip:** Prefer inline TVFs over multi-statement TVFs for performance.

### SP vs UDF — when to use which
| Feature | Stored Procedure (SP) | User-Defined Function (UDF) |
| :--- | :--- | :--- |
| **Return Value** | Optional (Return codes / Result sets) | Mandatory (Scalar or Table) |
| **Parameters** | Input and Output | Input only |
| **DML Support** | Can perform `INSERT/UPDATE/DELETE` | Mostly read-only (side-effects restricted) |
| **Calling** | `EXEC procedure_name` | Can be used in `SELECT`, `WHERE`, `JOIN` |
| **Transactions** | Can manage transactions | Cannot manage transactions |

**Interview Tip:** Use Functions for computations and data transformations within queries. Use Stored Procedures for business logic, complex workflows, and data modifications.

[⬆ Back to Top](#table-of-contents)

---

## Triggers (SQL Server)

### DML triggers — AFTER/FOR
**Query:**
```sql
CREATE OR ALTER TRIGGER dbo.trg_Orders_Audit
ON dbo.Orders
AFTER INSERT, UPDATE, DELETE
AS
BEGIN
  SET NOCOUNT ON;
  INSERT INTO dbo.OrderAudit (OrderID, Action, ChangedAt)
  SELECT ISNULL(i.OrderID, d.OrderID),
             CASE WHEN i.OrderID IS NOT NULL AND d.OrderID IS NULL THEN 'INSERT'
                  WHEN i.OrderID IS NOT NULL AND d.OrderID IS NOT NULL THEN 'UPDATE'
                  ELSE 'DELETE' END,
             SYSUTCDATETIME()
  FROM inserted i
  FULL OUTER JOIN deleted d ON d.OrderID = i.OrderID;
END;
```

### INSTEAD OF triggers — override action (views)
**Query:**
```sql
CREATE OR ALTER TRIGGER dbo.trg_vOrders_Insert
ON dbo.v_RecentOrders
INSTEAD OF INSERT
AS
BEGIN
  INSERT INTO dbo.Orders (CustomerID, OrderDate, TotalAmount)
  SELECT CustomerID, OrderDate, TotalAmount FROM inserted;
END;
```

[⬆ Back to Top](#table-of-contents)

---

## Cursors (SQL Server)

### Declaration and fetch
**Query:**
```sql
DECLARE cur CURSOR FAST_FORWARD FOR
  SELECT OrderID FROM dbo.Orders WHERE OrderDate >= DATEADD(day,-7,GETUTCDATE());
OPEN cur;
DECLARE @OrderID INT;
FETCH NEXT FROM cur INTO @OrderID;
WHILE @@FETCH_STATUS = 0
BEGIN
  -- process @OrderID
  FETCH NEXT FROM cur INTO @OrderID;
END
CLOSE cur; DEALLOCATE cur;
```

**Implementation Tip:** Prefer set-based operations; use cursors sparingly.

[⬆ Back to Top](#table-of-contents)

---

## Common Table Expressions (CTE)

### WITH syntax and scope
**Query:**
```sql
WITH Recent AS (
  SELECT * FROM dbo.Orders WHERE OrderDate >= DATEADD(day,-30, SYSUTCDATETIME())
)
SELECT * FROM Recent WHERE TotalAmount > 100;
```

### Recursive CTE — hierarchies
**Query:**
```sql
WITH EmpCTE AS (
  SELECT EmployeeID, ManagerID, 0 AS lvl FROM dbo.Employees WHERE ManagerID IS NULL
  UNION ALL
  SELECT e.EmployeeID, e.ManagerID, c.lvl + 1
  FROM dbo.Employees e
  JOIN EmpCTE c ON c.EmployeeID = e.ManagerID
)
SELECT * FROM EmpCTE OPTION (MAXRECURSION 100);
```

**Implementation Tip:** Use `OPTION (MAXRECURSION n)` to control recursion depth.

[⬆ Back to Top](#table-of-contents)

---

## Temporary Tables & Table Variables

### Overview — when to use temps vs CTE vs tables
Temporary structures persist beyond a single statement and can be indexed and referenced multiple times. 
- **Prefer temp tables** for large or re-used intermediates.
- **Prefer CTEs** for readability when reuse is limited.
- **Table variables** are lightweight but have limited statistics.

**Scope:**
- `#temp` and `@table` live for the current session. 
- `##temp` is global for the server until the last session drops it; avoid unless coordinating across sessions.

**Transaction behavior:** 
Temps participate in transactions; rollbacks undo changes. Creation is metadata-only and fast.

### Local temp tables #temp — features and indexing
Temp tables have statistics and can use parallel plans. Add indexes after load or pre-create and then insert depending on workload.

**Query:**
```sql
CREATE TABLE #TopCustomers (
  CustomerID INT PRIMARY KEY,
  OrderTotal MONEY,
  OrderCount INT
);
CREATE INDEX IX_TopCustomers_OrderTotal ON #TopCustomers(OrderTotal DESC);

INSERT INTO #TopCustomers(CustomerID, OrderTotal, OrderCount)
SELECT CustomerID, SUM(TotalAmount), COUNT(*)
FROM dbo.Orders
GROUP BY CustomerID;

SELECT * FROM #TopCustomers WHERE OrderTotal >= 10000;
```

### Table variables @table — pros and cons
Memory-optimized metadata; fewer recompilations. Prior to SQL Server 2019, cardinality was often assumed as 1 row, hurting plans. In 2019+, in-row deferred compilation improves estimates but limitations remain.

**Query:**
```sql
DECLARE @t TABLE (
  CustomerID INT PRIMARY KEY,
  OrderCount INT
);
INSERT INTO @t SELECT CustomerID, COUNT(*) FROM dbo.Orders GROUP BY CustomerID;
SELECT * FROM @t WHERE OrderCount >= 10;
```

**Implementation Tip:** Use for small, short-lived sets. Avoid complex indexing; cannot create nonclustered indexes after declaration.

### Global temp tables ##temp — coordination use only
Visible to all sessions; suffix a random token to avoid name collisions. Clean up when done.

**Query:**
```sql
CREATE TABLE ##BatchIds_9F2C1 (BatchID INT PRIMARY KEY);
```

### Temp table performance patterns
- Pre-size columns narrowly; avoid `SELECT *` into temps from huge sources without filters.
- Create indexes only when predicates or joins need them; drop when done to save tempdb I/O.
- Use `SELECT INTO #t (...)` as a fast path to create and load, then add indexes.
- Monitor tempdb contention; consider multiple data files if needed.

### Compare: #temp vs @table
- **#temp:** statistics, flexible indexing, better for medium/large sets, can be modified repeatedly.
- **@table:** limited stats pre-2019, good for small sets, simpler lifetime and scope.

[⬆ Back to Top](#table-of-contents)

---

## Temporal Tables & CTAS

### CTAS (CREATE TABLE AS SELECT) — cross-engine note
SQL Server equivalent is `SELECT INTO`. PostgreSQL/MySQL support CTAS directly.

**Query (PostgreSQL reference):**
```sql
-- PostgreSQL
CREATE TABLE staging.big_orders AS
SELECT * FROM orders WHERE total_amount >= 1000;
```

**Query (SQL Server equivalent):**
```sql
SELECT o.CustomerID, o.OrderID, o.TotalAmount
INTO #RecentLarge
FROM dbo.Orders o
WHERE o.OrderDate >= DATEADD(day,-30, SYSUTCDATETIME())
  AND o.TotalAmount >= 1000;

CREATE INDEX IX_RecentLarge_Customer ON #RecentLarge(CustomerID);
```

[⬆ Back to Top](#table-of-contents)

---

### Temporal Tables (system-versioned)
Automatically keeps history of row versions. Two tables: current (system-versioned) and history. Engine manages validity period columns.

**Query (Create Temporal):**
```sql
CREATE TABLE dbo.CustomersTemporal (
  CustomerID INT PRIMARY KEY,
  Email NVARCHAR(320) NOT NULL,
  ValidFrom DATETIME2 GENERATED ALWAYS AS ROW START,
  ValidTo   DATETIME2 GENERATED ALWAYS AS ROW END,
  PERIOD FOR SYSTEM_TIME (ValidFrom, ValidTo)
) WITH (SYSTEM_VERSIONING = ON (HISTORY_TABLE = dbo.CustomersTemporalHistory));
```

**Query (AS OF Query):**
```sql
SELECT * FROM dbo.CustomersTemporal
FOR SYSTEM_TIME AS OF '2025-01-01T00:00:00Z'
WHERE CustomerID = 42;
```

[⬆ Back to Top](#table-of-contents)

---

## PIVOT / UNPIVOT

### PIVOT — rows to columns
**Query:**
```sql
SELECT * FROM (
  SELECT ProductName, DATENAME(month, OrderDate) AS [Month], Quantity
  FROM dbo.Sales
) src
PIVOT (SUM(Quantity) FOR [Month] IN ([Jan],[Feb],[Mar],[Apr],[May],[Jun],[Jul],[Aug],[Sep],[Oct],[Nov],[Dec])) AS p;
```

### UNPIVOT — columns to rows
**Query:**
```sql
SELECT ProductName, [Month], Qty
FROM dbo.SalesByMonth
UNPIVOT (Qty FOR [Month] IN ([Jan],[Feb],[Mar])) AS u;
```

[⬆ Back to Top](#table-of-contents)

---

## APPLY operators (SQL Server)

### CROSS APPLY / OUTER APPLY
Join each row to the result of a table-valued function or derived table expression.

**Query:**
```sql
SELECT o.OrderID, x.TopLineTotal
FROM dbo.Orders o
CROSS APPLY (
  SELECT TOP 1 l.LineTotal AS TopLineTotal
  FROM dbo.OrderLines l WHERE l.OrderID = o.OrderID ORDER BY l.LineTotal DESC
) x;
```

[⬆ Back to Top](#table-of-contents)

---

## Transaction Isolation Levels

Isolation levels define how one transaction is isolated from concurrent changes made by other transactions.

- **READ UNCOMMITTED:** Lowest level; allows "dirty reads" (reading uncommitted data).
- **READ COMMITTED:** (Default in SQL Server) Prevents dirty reads; allows "non-repeatable reads".
- **REPEATABLE READ:** Prevents dirty and non-repeatable reads; allows "phantom reads".
- **SERIALIZABLE:** Highest level; prevents phantoms by locking the entire range.
- **SNAPSHOT:** Uses row versioning instead of locks for concurrency.

**Query Example:**
```sql
SET TRANSACTION ISOLATION LEVEL READ COMMITTED;
BEGIN TRAN;
  -- queries
COMMIT;
```

[⬆ Back to Top](#table-of-contents)

---

## Index Basics: Clustered vs Non-Clustered

### Clustered Index
- Determines the **physical order** of data in the table.
- A table can have only **one** clustered index (usually on the Primary Key).
- The leaf level of the index **is** the actual data.

### Non-Clustered Index
- A separate structure from the data rows.
- Contains pointers (row locators) to the actual data.
- A table can have **multiple** non-clustered indexes.
- Great for covering specific queries without re-sorting the whole table.

**Query example:**
```sql
-- Clustered index is created automatically on PK
-- Creating a Non-Clustered index for fast lookup
CREATE NONCLUSTERED INDEX IX_Customers_Email ON dbo.Customers(Email);
```

[⬆ Back to Top](#table-of-contents)
