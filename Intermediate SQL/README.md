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
    - [INTERSECT — common rows](#intersect--common-rows)
    - [EXCEPT — rows in A not in B](#except--rows-in-a-not-in-b)
3. [Subqueries (SQL Server)](#subqueries-sql-server)
    - [Scalar subquery — single value per row](#scalar-subquery--single-value-per-row)
    - [Correlated subquery — references outer row](#correlated-subquery--references-outer-row)
    - [IN / NOT IN with subqueries](#in--not-in-with-subqueries)
    - [ALL / ANY / SOME](#all--any--some)
    - [Derived tables (subquery in FROM)](#derived-tables-subquery-in-from)
4. [Views (SQL Server)](#views-sql-server)
    - [CREATE VIEW — virtual table](#create-view--virtual-table)
    - [Updatable views — constraints](#updatable-views--constraints)
    - [WITH SCHEMABINDING and indexed views](#with-schemabinding-and-indexed-views)
    - [WITH CHECK OPTION — enforce view predicate](#with-check-option--enforce-view-predicate)
    - [ALTER VIEW / DROP VIEW](#alter-view--drop-view)
5. [Stored Procedures (SQL Server)](#stored-procedures-sql-server)
    - [CREATE PROCEDURE — reusable T‑SQL](#create-procedure--reusable-t-sql)
    - [Parameters, output, return codes](#parameters-output-return-codes)
    - [TRY...CATCH error handling](#trycatch-error-handling)
6. [User-Defined Functions (SQL Server)](#user-defined-functions-sql-server)
    - [Scalar UDF — returns a single value](#scalar-udf--returns-a-single-value)
    - [Inline Table-Valued Function (iTVF)](#inline-table-valued-function-itvf)
7. [Triggers (SQL Server)](#triggers-sql-server)
    - [DML triggers — AFTER/FOR](#dml-triggers--afterfor)
    - [INSTEAD OF triggers — override action (views)](#instead-of-triggers--override-action-views)
8. [Cursors (SQL Server)](#cursors-sql-server)
    - [Declaration and fetch](#declaration-and-fetch)

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

## Stored Procedures (SQL Server)

### CREATE PROCEDURE — reusable T‑SQL
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

## User-Defined Functions (SQL Server)

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
