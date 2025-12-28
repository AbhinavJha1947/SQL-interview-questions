# Basic SQL — CRUD and Core Queries

This section covers foundational SQL concepts including Data Definition Language (DDL), Data Manipulation Language (DML), basic filtering, sorting, and aggregate functions.

---

## Table of Contents

1. [Data Definition Language (DDL)](#data-definition-language-ddl)
    - [CREATE — database, table, schema](#create--database-table-schema)
    - [ALTER — add, drop, modify columns and constraints](#alter--add-drop-modify-columns-and-constraints)
    - [DROP — remove database objects](#drop--remove-database-objects)
    - [TRUNCATE — fast delete all rows](#truncate--fast-delete-all-rows)
    - [RENAME — rename objects](#rename--rename-objects)
2. [Data Manipulation Language (DML)](#data-manipulation-language-dml)
    - [INSERT — single, multi-row, INSERT...SELECT](#insert--single-multi-row-insertselect)
    - [SELECT — basic query structure](#select--basic-query-structure)
    - [UPDATE — conditional updates](#update--conditional-updates)
    - [DELETE — remove rows](#delete--remove-rows)
    - [MERGE/UPSERT — insert or update](#mergeupsert--insert-or-update)
3. [Basic SELECT](#basic-select)
    - [SELECT list — expressions and aliases](#select-list--expressions-and-aliases)
    - [FROM — table and aliases](#from--table-and-aliases)
    - [WHERE — filtering rows](#where--filtering-rows)
    - [Comparison operators (=, <>, <, >, <=, >=)](#comparison-operators--v---)
    - [Logical operators (AND, OR, NOT)](#logical-operators-and-or-not)
    - [IN and BETWEEN](#in-and-between)
    - [LIKE — pattern matching](#like--pattern-matching)
    - [IS NULL / IS NOT NULL](#is-null--is-not-null)
    - [DISTINCT — remove duplicates](#distinct--remove-duplicates)
    - [ORDER BY — sorting](#order-by--sorting)
    - [Pagination — OFFSET/FETCH and keyset](#pagination--offsetfetch-and-keyset)
4. [Aggregate Functions](#aggregate-functions)
    - [COUNT, SUM, AVG, MIN, MAX](#count-sum-avg-min-max)
    - [GROUP BY — grouping rows](#group-by--grouping-rows)
    - [HAVING — filter groups](#having--filter-groups)
5. [String Functions](#string-functions)
    - [CONCAT, SUBSTRING, LEN, UPPER/LOWER, TRIM, REPLACE, CHARINDEX](#concat-substring-len-upperlower-trim-replace-charindex)
6. [Date/Time Functions](#datetime-functions)
    - [GETUTCDATE, SYSUTCDATETIME, DATEADD, DATEDIFF, DATEPART, CONVERT/CAST](#getutcdate-sysutcdatetime-dateadd-datediff-datepart-convertcast)
7. [Numeric Functions](#numeric-functions)
    - [ROUND, CEILING, FLOOR, ABS, POWER, SQRT, %](#round-ceiling-floor-abs-power-sqrt-)
8. [Conditional Logic](#conditional-logic)
    - [CASE — simple and searched](#case--simple-and-searched)
    - [COALESCE, NULLIF, ISNULL, IIF](#coalesce-nullif-isnull-iif)

---

## Data Definition Language (DDL)

### CREATE — database, table, schema
Define new objects with columns, data types, and constraints.

**Query (SQL Server):**
```sql
CREATE SCHEMA reporting;
CREATE TABLE reporting.Users (
  UserID BIGINT IDENTITY(1,1) PRIMARY KEY,
  Email NVARCHAR(320) NOT NULL UNIQUE,
  CreatedAt DATETIME2 NOT NULL CONSTRAINT DF_Users_Created DEFAULT SYSUTCDATETIME()
);
```

### ALTER — add, drop, modify columns and constraints
**Query (SQL Server):**
```sql
ALTER TABLE reporting.Users ADD IsActive BIT NOT NULL CONSTRAINT DF_Users_IsActive DEFAULT 1;
```

### DROP — remove database objects
**Query (SQL Server):**
```sql
DROP TABLE IF EXISTS reporting.Users;
```

### TRUNCATE — fast delete all rows
**Query (SQL Server):**
```sql
TRUNCATE TABLE staging.Events;
```

### RENAME — rename objects
**Query (SQL Server):**
```sql
EXEC sp_rename 'reporting.Users.Email', 'EmailAddress', 'COLUMN';
```

[⬆ Back to Top](#table-of-contents)

---

## Data Manipulation Language (DML)

### INSERT — single, multi-row, INSERT...SELECT
**Query (SQL Server):**
```sql
INSERT INTO reporting.Users (Email) VALUES ('a@ex.com');
INSERT INTO reporting.Users (Email) VALUES ('b@ex.com'), ('c@ex.com');
INSERT INTO reporting.ActiveUsers (UserID, Email)
SELECT UserID, Email FROM reporting.Users WHERE IsActive = 1;
```

### SELECT — basic query structure
**Query (SQL Server):**
```sql
SELECT UserID, Email AS Address
FROM reporting.Users
WHERE IsActive = 1
ORDER BY CreatedAt DESC
OFFSET 0 ROWS FETCH NEXT 10 ROWS ONLY;
```

### UPDATE — conditional updates
**Query (SQL Server):**
```sql
UPDATE reporting.Users
SET IsActive = 0
WHERE DATEDIFF(day, CreatedAt, SYSUTCDATETIME()) > 365;
```

### DELETE — remove rows
**Query (SQL Server):**
```sql
DELETE FROM reporting.Users WHERE IsActive = 0;
```

### MERGE/UPSERT — insert or update
**Query (SQL Server):**
```sql
MERGE reporting.Users AS t
USING (SELECT 'a@ex.com' AS Email, 1 AS IsActive) s
ON t.Email = s.Email
WHEN MATCHED THEN UPDATE SET IsActive = s.IsActive
WHEN NOT MATCHED THEN INSERT (Email, IsActive) VALUES (s.Email, s.IsActive);
```

[⬆ Back to Top](#table-of-contents)

---

## Basic SELECT

### SELECT list — expressions and aliases
**Query:**
```sql
SELECT UserID, UPPER(Email) AS EmailUpper, CAST(CreatedAt AS date) AS CreatedDate
FROM reporting.Users;
```

### FROM — table and aliases
**Query:**
```sql
SELECT u.UserID, u.Email FROM reporting.Users AS u;
```

### WHERE — filtering rows
**Query:**
```sql
SELECT * FROM reporting.Users WHERE Email LIKE 'a%';
```

### Comparison operators (=, <>, <, >, <=, >=)
**Query:**
```sql
SELECT * FROM Sales WHERE TotalAmount >= 100 AND TotalAmount < 500;
```

### Logical operators (AND, OR, NOT)
**Query:**
```sql
SELECT * FROM Users WHERE IsActive = 1 AND NOT IsBanned = 1;
```

### IN and BETWEEN
**Query:**
```sql
SELECT * FROM Products WHERE Category IN ('Books','Games');
SELECT * FROM Invoices WHERE InvoiceDate BETWEEN '2025-01-01' AND '2025-12-31';
```

### LIKE — pattern matching
**Query:**
```sql
SELECT * FROM Users WHERE Email LIKE '%@example.com';
```

### IS NULL / IS NOT NULL
**Query:**
```sql
SELECT * FROM Shipments WHERE DeliveredAt IS NULL;
```

### DISTINCT — remove duplicates
**Query:**
```sql
SELECT DISTINCT Country FROM Customers;
```

### ORDER BY — sorting
**Query:**
```sql
SELECT * FROM Users ORDER BY CreatedAt DESC, UserID ASC;
```

### Pagination — OFFSET/FETCH and keyset
**Query:**
```sql
SELECT * FROM Users ORDER BY UserID OFFSET 40 ROWS FETCH NEXT 20 ROWS ONLY;
-- Keyset
SELECT TOP (20) * FROM Users WHERE UserID > @LastId ORDER BY UserID;
```

[⬆ Back to Top](#table-of-contents)

---

## Aggregate Functions

### COUNT, SUM, AVG, MIN, MAX
**Query:**
```sql
SELECT COUNT(*) AS UsersTotal,
       SUM(CASE WHEN IsActive = 1 THEN 1 ELSE 0 END) AS ActiveUsers
FROM reporting.Users;
```

### GROUP BY — grouping rows
**Query:**
```sql
SELECT CAST(CreatedAt AS date) AS CreatedDate, COUNT(*) AS NewUsers
FROM reporting.Users
GROUP BY CAST(CreatedAt AS date)
ORDER BY CreatedDate;
```

### HAVING — filter groups
**Query:**
```sql
SELECT Country, COUNT(*) AS Cnt
FROM Customers
GROUP BY Country
HAVING COUNT(*) >= 100;
```

[⬆ Back to Top](#table-of-contents)

---

## String Functions

### CONCAT, SUBSTRING, LEN, UPPER/LOWER, TRIM, REPLACE, CHARINDEX
**Query:**
```sql
SELECT CONCAT(FirstName,' ',LastName) AS FullName,
       SUBSTRING(Email,1,5) AS Prefix,
       LEN(Email) AS L,
       TRIM(Name) AS Trimmed
FROM People;
```

[⬆ Back to Top](#table-of-contents)

---

## Date/Time Functions

### GETUTCDATE, SYSUTCDATETIME, DATEADD, DATEDIFF, DATEPART, CONVERT/CAST
**Query:**
```sql
SELECT DATEADD(day,7,SYSUTCDATETIME()) AS InAWeek,
       DATEDIFF(day, CreatedAt, SYSUTCDATETIME()) AS AgeDays,
       DATEPART(year, SYSUTCDATETIME()) AS Yr;
```

[⬆ Back to Top](#table-of-contents)

---

## Numeric Functions

### ROUND, CEILING, FLOOR, ABS, POWER, SQRT, %
**Query:**
```sql
SELECT ROUND(12.345,2) AS R,
       CEILING(4.1) AS C,
       FLOOR(4.9) AS F,
       ABS(-5) AS A,
       POWER(2,8) AS P,
       17 % 5 AS M;
```

[⬆ Back to Top](#table-of-contents)

---

## Conditional Logic

### CASE — simple and searched
**Query:**
```sql
SELECT OrderID,
       CASE WHEN TotalAmount >= 1000 THEN 'VIP'
            WHEN TotalAmount >= 100 THEN 'Regular'
            ELSE 'Small' END AS OrderBucket
FROM Orders;
```

### COALESCE, NULLIF, ISNULL, IIF
**Query:**
```sql
SELECT COALESCE(Phone,'N/A') AS PhoneSafe,
       NULLIF(Status,'Unknown') AS StatusClean,
       IIF(IsActive=1,'Active','Inactive') AS StatusTxt
FROM Customers;
```

[⬆ Back to Top](#table-of-contents)
