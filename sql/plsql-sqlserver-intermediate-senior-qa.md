# PL/SQL and SQL Server Interview Q&A (Intermediate to Senior, 30 Questions)

This guide covers both Oracle PL/SQL and Microsoft SQL Server (T-SQL).
Each question includes:
- Short answer
- Real-time example
- Explanation
- Commented SQL code snippet

---

## 1) What is the difference between SQL and PL/SQL (or T-SQL)?

Short answer:
SQL is declarative for querying/manipulating data. PL/SQL and T-SQL are procedural extensions for variables, loops, conditions, and error handling.

Real-time example:
Monthly payroll requires step-by-step business rules, not just a single SELECT.

```sql
-- SQL: declarative query
SELECT employee_id, salary
FROM employees
WHERE department_id = 10;

-- PL/SQL: procedural block example (Oracle)
DECLARE
  v_count NUMBER;
BEGIN
  SELECT COUNT(*) INTO v_count
  FROM employees
  WHERE department_id = 10;

  DBMS_OUTPUT.PUT_LINE('Employees in dept 10: ' || v_count);
END;
/
```

---

## 2) What are stored procedures and why use them?

Short answer:
Stored procedures are precompiled routines in the database used to encapsulate business logic and improve reuse.

Real-time example:
Order placement logic should run consistently regardless of API/client.

```sql
-- SQL Server procedure
CREATE OR ALTER PROCEDURE dbo.usp_CreateOrder
  @CustomerId INT,
  @TotalAmount DECIMAL(18,2)
AS
BEGIN
  SET NOCOUNT ON;

  INSERT INTO dbo.Orders(CustomerId, TotalAmount, CreatedAt)
  VALUES(@CustomerId, @TotalAmount, SYSUTCDATETIME());
END;
GO
```

---

## 3) What is the difference between a function and a procedure?

Short answer:
Functions return a value and are often used in expressions; procedures perform actions and can return multiple outputs.

Real-time example:
Tax calculation is reusable and expression-friendly, so function is suitable.

```sql
-- Oracle function
CREATE OR REPLACE FUNCTION fn_calc_tax(p_amount NUMBER)
RETURN NUMBER
IS
BEGIN
  RETURN p_amount * 0.18;
END;
/

-- Usage in query
SELECT order_id, amount, fn_calc_tax(amount) AS tax
FROM orders;
```

---

## 4) What are transactions and ACID properties?

Short answer:
Transaction is a unit of work. ACID = Atomicity, Consistency, Isolation, Durability.

Real-time example:
Bank transfer must debit and credit together or roll back both.

```sql
-- SQL Server transaction example
BEGIN TRY
  BEGIN TRAN;

  UPDATE dbo.Accounts
  SET Balance = Balance - 100
  WHERE AccountId = 1;

  UPDATE dbo.Accounts
  SET Balance = Balance + 100
  WHERE AccountId = 2;

  COMMIT TRAN;
END TRY
BEGIN CATCH
  IF @@TRANCOUNT > 0 ROLLBACK TRAN;
  THROW;
END CATCH;
GO
```

---

## 5) What is isolation level and why does it matter?

Short answer:
Isolation level controls visibility of uncommitted/changed data between concurrent transactions.

Real-time example:
High-concurrency checkout can suffer dirty reads or deadlocks with wrong isolation choice.

```sql
-- SQL Server: set isolation level for current session
SET TRANSACTION ISOLATION LEVEL READ COMMITTED;
GO

BEGIN TRAN;
  SELECT * FROM dbo.Inventory WHERE ProductId = 1001;
COMMIT TRAN;
GO
```

---

## 6) What are common joins and when to use them?

Short answer:
INNER, LEFT, RIGHT, FULL joins combine rows from tables based on relationships.

Real-time example:
Order report needs all orders, including those without shipment yet (LEFT JOIN).

```sql
SELECT o.OrderId,
       o.CustomerId,
       s.ShipmentId,
       s.ShipDate
FROM dbo.Orders o
LEFT JOIN dbo.Shipments s
  ON o.OrderId = s.OrderId;
```

---

## 7) What is a CTE and where is it useful?

Short answer:
CTE (Common Table Expression) improves readability and supports recursion.

Real-time example:
Organization hierarchy (manager -> employee tree).

```sql
-- SQL Server recursive CTE for hierarchy
WITH OrgCTE AS
(
  SELECT EmployeeId, ManagerId, Name, 0 AS LevelNo
  FROM dbo.Employees
  WHERE ManagerId IS NULL

  UNION ALL

  SELECT e.EmployeeId, e.ManagerId, e.Name, c.LevelNo + 1
  FROM dbo.Employees e
  INNER JOIN OrgCTE c ON e.ManagerId = c.EmployeeId
)
SELECT *
FROM OrgCTE
ORDER BY LevelNo, Name;
GO
```

---

## 8) What are window functions and why are they important?

Short answer:
Window functions perform calculations across sets of rows without collapsing result like GROUP BY.

Real-time example:
Rank top-selling products within each category.

```sql
SELECT CategoryId,
       ProductId,
       SalesAmount,
       DENSE_RANK() OVER (PARTITION BY CategoryId ORDER BY SalesAmount DESC) AS SalesRank
FROM dbo.ProductSales;
```

---

## 9) Difference between ROW_NUMBER, RANK, and DENSE_RANK?

Short answer:
ROW_NUMBER gives unique sequence, RANK skips values on ties, DENSE_RANK does not skip.

```sql
SELECT EmployeeId,
       Salary,
       ROW_NUMBER() OVER (ORDER BY Salary DESC) AS RN,
       RANK() OVER (ORDER BY Salary DESC) AS Rnk,
       DENSE_RANK() OVER (ORDER BY Salary DESC) AS DenseRnk
FROM dbo.Employees;
```

---

## 10) What is indexing and how do you choose indexes?

Short answer:
Indexes speed reads but add write overhead. Choose based on filter/join/sort patterns.

Real-time example:
Frequent lookups by Email should use index on Email.

```sql
-- SQL Server index example
CREATE UNIQUE INDEX IX_Users_Email
ON dbo.Users(Email);
GO

-- Composite index for frequent query patterns
CREATE INDEX IX_Orders_Customer_CreatedAt
ON dbo.Orders(CustomerId, CreatedAt DESC);
GO
```

---

## 11) Clustered vs nonclustered index in SQL Server?

Short answer:
Clustered index defines physical row order (one per table). Nonclustered stores separate index structure with row locators.

Real-time example:
Transaction table often clustered by identity or date for range scans.

```sql
-- Clustered primary key
CREATE TABLE dbo.Transactions
(
  TransactionId BIGINT NOT NULL PRIMARY KEY CLUSTERED,
  AccountId INT NOT NULL,
  Amount DECIMAL(18,2) NOT NULL,
  CreatedAt DATETIME2 NOT NULL
);
GO
```

---

## 12) What is execution plan and how do you use it?

Short answer:
Execution plan shows how query is executed and helps identify scans, expensive operators, and missing indexes.

Real-time example:
Slow dashboard query drops from 12s to 400ms after index and predicate fixes.

```sql
-- SQL Server: inspect estimated/actual plan in SSMS
SET STATISTICS IO ON;
SET STATISTICS TIME ON;

SELECT o.OrderId, o.CustomerId
FROM dbo.Orders o
WHERE o.CustomerId = 12345;

SET STATISTICS IO OFF;
SET STATISTICS TIME OFF;
GO
```

---

## 13) What is parameter sniffing in SQL Server?

Short answer:
SQL Server may cache a plan optimized for first parameter value, which can be bad for different future values.

Real-time example:
Procedure fast for small customer data but slow for large enterprise customer.

```sql
CREATE OR ALTER PROCEDURE dbo.usp_GetOrdersByCustomer
  @CustomerId INT
AS
BEGIN
  SET NOCOUNT ON;

  SELECT OrderId, CustomerId, CreatedAt
  FROM dbo.Orders
  WHERE CustomerId = @CustomerId
  OPTION (RECOMPILE); -- one mitigation when needed
END;
GO
```

---

## 14) What is dynamic SQL and how to avoid SQL injection?

Short answer:
Dynamic SQL builds query text at runtime. Always parameterize input.

Real-time example:
Advanced report supports optional filters.

```sql
-- SQL Server safe dynamic SQL
DECLARE @sql NVARCHAR(MAX) = N'
  SELECT OrderId, CustomerId, TotalAmount
  FROM dbo.Orders
  WHERE CreatedAt >= @FromDate';

DECLARE @FromDate DATETIME2 = '2025-01-01';

EXEC sp_executesql
  @sql,
  N'@FromDate DATETIME2',
  @FromDate = @FromDate;
GO
```

---

## 15) How do you handle errors in PL/SQL and T-SQL?

Short answer:
Use EXCEPTION block in PL/SQL and TRY/CATCH in SQL Server.

Real-time example:
Billing procedure must log failure and rollback safely.

```sql
-- Oracle PL/SQL error handling
DECLARE
  v_total NUMBER;
BEGIN
  SELECT SUM(amount) INTO v_total FROM invoices;
  DBMS_OUTPUT.PUT_LINE('Total: ' || v_total);
EXCEPTION
  WHEN OTHERS THEN
    DBMS_OUTPUT.PUT_LINE('Error: ' || SQLERRM);
END;
/
```

---

## 16) What are triggers and when should you use them carefully?

Short answer:
Triggers automatically execute on DML events. Use sparingly for auditing or integrity when app-level control is insufficient.

Real-time example:
Audit trail required for every salary update.

```sql
-- SQL Server audit trigger example
CREATE OR ALTER TRIGGER dbo.trg_Employees_Audit
ON dbo.Employees
AFTER UPDATE
AS
BEGIN
  SET NOCOUNT ON;

  INSERT INTO dbo.EmployeeAudit(EmployeeId, OldSalary, NewSalary, ChangedAt)
  SELECT d.EmployeeId, d.Salary, i.Salary, SYSUTCDATETIME()
  FROM deleted d
  INNER JOIN inserted i ON d.EmployeeId = i.EmployeeId
  WHERE d.Salary <> i.Salary;
END;
GO
```

---

## 17) What are temporary tables and table variables?

Short answer:
Temporary tables (`#Temp`) are better for larger/intermediate result sets with indexes/stats. Table variables are lighter for small sets.

Real-time example:
Complex ETL step joins multiple staged datasets.

```sql
-- SQL Server temp table
CREATE TABLE #TopCustomers
(
  CustomerId INT PRIMARY KEY,
  TotalSpent DECIMAL(18,2)
);

INSERT INTO #TopCustomers(CustomerId, TotalSpent)
SELECT CustomerId, SUM(TotalAmount)
FROM dbo.Orders
GROUP BY CustomerId;

SELECT * FROM #TopCustomers;
DROP TABLE #TopCustomers;
GO
```

---

## 18) What is normalization and when denormalization is acceptable?

Short answer:
Normalization reduces redundancy and anomalies. Denormalization may improve read performance for analytics/reporting.

Real-time example:
Operational DB normalized; reporting table denormalized for BI dashboard speed.

```sql
-- Normalized example (conceptual)
-- Customers(CustomerId, Name)
-- Orders(OrderId, CustomerId, CreatedAt)

-- Denormalized reporting table
-- DailySalesSummary(SalesDate, Region, TotalOrders, TotalRevenue)
```

---

## 19) What is partitioning and when is it useful?

Short answer:
Partitioning splits large table/index into manageable pieces, often by date or key range.

Real-time example:
5-year transaction table queried mostly by month.

```sql
-- SQL Server partitioning requires partition function/scheme setup.
-- Conceptual query benefits from partition elimination:
SELECT *
FROM dbo.Transactions
WHERE CreatedAt >= '2026-01-01' AND CreatedAt < '2026-02-01';
GO
```

---

## 20) How do you optimize pagination queries?

Short answer:
Use keyset pagination for large pages and OFFSET/FETCH for simple scenarios.

Real-time example:
Order history page with millions of rows.

```sql
-- SQL Server OFFSET pagination
SELECT OrderId, CustomerId, CreatedAt
FROM dbo.Orders
ORDER BY CreatedAt DESC
OFFSET 0 ROWS FETCH NEXT 20 ROWS ONLY;
GO

-- Keyset pagination idea:
-- WHERE CreatedAt < @LastSeenCreatedAt ORDER BY CreatedAt DESC
```

---

## 21) How do you write MERGE or UPSERT safely?

Short answer:
Use UPSERT pattern carefully with proper locking/constraints; MERGE has edge cases depending on DB/version.

Real-time example:
Sync product price feed into master table.

```sql
-- SQL Server UPSERT pattern (transaction + lock)
BEGIN TRAN;

UPDATE dbo.Products WITH (UPDLOCK, SERIALIZABLE)
SET Price = @Price
WHERE ProductCode = @ProductCode;

IF @@ROWCOUNT = 0
BEGIN
  INSERT INTO dbo.Products(ProductCode, Price)
  VALUES(@ProductCode, @Price);
END;

COMMIT TRAN;
GO
```

---

## 22) What are views and materialized views?

Short answer:
Views are virtual query definitions. Materialized views (Oracle) store results physically for faster reads.

Real-time example:
Finance dashboard frequently queries monthly revenue summary.

```sql
-- SQL Server regular view
CREATE OR ALTER VIEW dbo.vw_OrderSummary
AS
SELECT CustomerId, COUNT(*) AS OrderCount, SUM(TotalAmount) AS TotalAmount
FROM dbo.Orders
GROUP BY CustomerId;
GO

-- Oracle materialized view concept:
-- CREATE MATERIALIZED VIEW mv_sales_summary AS SELECT ...;
```

---

## 23) How do you enforce data integrity at DB level?

Short answer:
Use primary keys, foreign keys, unique constraints, check constraints, and not null.

Real-time example:
Prevent negative quantity and duplicate user emails.

```sql
ALTER TABLE dbo.OrderItems
ADD CONSTRAINT CK_OrderItems_Quantity_Positive CHECK (Quantity > 0);
GO

ALTER TABLE dbo.Users
ADD CONSTRAINT UQ_Users_Email UNIQUE (Email);
GO
```

---

## 24) How do you design audit logging in SQL Server/Oracle?

Short answer:
Capture who changed what and when, using audit tables/triggers or built-in features.

Real-time example:
Compliance requires salary change history for 7 years.

```sql
CREATE TABLE dbo.AuditLog
(
  AuditId BIGINT IDENTITY PRIMARY KEY,
  TableName NVARCHAR(128) NOT NULL,
  RecordId NVARCHAR(128) NOT NULL,
  ActionType NVARCHAR(20) NOT NULL,
  ChangedBy NVARCHAR(128) NOT NULL,
  ChangedAt DATETIME2 NOT NULL DEFAULT SYSUTCDATETIME()
);
GO
```

---

## 25) What are scalar, inline table-valued, and multi-statement functions in SQL Server?

Short answer:
Inline TVFs are generally more optimizer-friendly than scalar/multi-statement functions for set-based work.

Real-time example:
Reusable filtered order set for reporting.

```sql
CREATE OR ALTER FUNCTION dbo.fn_OrdersByDate(@FromDate DATE)
RETURNS TABLE
AS
RETURN
(
  SELECT OrderId, CustomerId, TotalAmount, CreatedAt
  FROM dbo.Orders
  WHERE CreatedAt >= @FromDate
);
GO

SELECT * FROM dbo.fn_OrdersByDate('2026-01-01');
```

---

## 26) How do you monitor and troubleshoot blocking/deadlocks?

Short answer:
Use DMVs/Extended Events (SQL Server) and AWR/ASH (Oracle) to identify blockers and optimize transaction scope/indexes.

Real-time example:
Flash sale causes deadlocks between inventory update and order insert.

```sql
-- SQL Server quick blocking insight
SELECT
  r.session_id,
  r.blocking_session_id,
  r.wait_type,
  r.wait_time,
  t.text AS running_sql
FROM sys.dm_exec_requests r
CROSS APPLY sys.dm_exec_sql_text(r.sql_handle) t
WHERE r.blocking_session_id <> 0;
GO
```

---

## 27) What is ETL vs ELT in data pipelines?

Short answer:
ETL transforms before loading; ELT loads raw then transforms inside target system.

Real-time example:
Daily sales data from multiple systems landed in warehouse and transformed there (ELT).

```sql
-- ELT style example: load raw first, then transform
INSERT INTO dbo.SalesStaging(RawOrderId, RawAmount, RawDate)
VALUES ('A-1001', '199.99', '2026-04-01');

INSERT INTO dbo.SalesFact(OrderId, Amount, SalesDate)
SELECT TRY_CONVERT(BIGINT, REPLACE(RawOrderId, 'A-', '')),
       TRY_CONVERT(DECIMAL(18,2), RawAmount),
       TRY_CONVERT(DATE, RawDate)
FROM dbo.SalesStaging;
GO
```

---

## 28) How do you secure database access in enterprise apps?

Short answer:
Use least privilege, role-based access, parameterized SQL, encrypted connections, and secret rotation.

Real-time example:
Reporting app has read-only DB role; no DDL/DML permissions.

```sql
-- SQL Server role-based access example
CREATE ROLE ReportingReader;
GO

GRANT SELECT ON dbo.Orders TO ReportingReader;
GO

-- Then assign user/login to role.
```

---

## 29) How do you handle schema changes with minimal downtime?

Short answer:
Use backward-compatible migrations, deploy in phases, and avoid long blocking changes during peak.

Real-time example:
Add nullable column first, deploy app reading both old/new, then backfill and enforce constraints later.

```sql
-- Phase 1: additive non-breaking change
ALTER TABLE dbo.Orders
ADD ExternalRef NVARCHAR(100) NULL;
GO

-- Later phase: backfill + optional constraint
-- UPDATE dbo.Orders SET ExternalRef = ... WHERE ExternalRef IS NULL;
```

---

## 30) What are top SQL performance best practices for senior engineers?

Short answer:
Write set-based queries, index for workload, keep transactions short, avoid SELECT *, monitor plans/IO/time, and tune iteratively.

Real-time example:
API latency reduced 80 percent by replacing RBAR cursor logic with set-based update.

```sql
-- Set-based update example (preferred over row-by-row loops)
UPDATE oi
SET oi.DiscountAmount = oi.Quantity * oi.UnitPrice * 0.10
FROM dbo.OrderItems oi
INNER JOIN dbo.Orders o ON oi.OrderId = o.OrderId
WHERE o.CreatedAt >= '2026-01-01';
GO
```

---

## Rapid Fire Senior Follow-ups

1. Difference between READ COMMITTED SNAPSHOT and SNAPSHOT isolation?
2. How do you choose fill factor for heavy-update indexes?
3. What is SARGability and why does it matter?
4. How do computed columns help indexing strategy?
5. How to detect implicit conversions hurting index usage?
6. How do you archive historical data safely?
7. How to build idempotent ETL jobs?
8. When would you use columnstore indexes?
9. How do you validate migration rollback strategy?
10. How to separate OLTP and OLAP workloads?

---

## Practice Plan

1. Implement 5 procedures with robust error/transaction handling.
2. Build a report query with CTE plus window functions and tune it.
3. Add indexes for 3 slow queries and compare IO/time stats.
4. Simulate deadlock/blocking and document mitigation.
5. Create a zero-downtime migration plan for one schema change.
