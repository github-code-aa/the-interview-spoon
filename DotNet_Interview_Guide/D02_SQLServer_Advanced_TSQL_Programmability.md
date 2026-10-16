# D02 — SQL Server Advanced T-SQL & Programmability

**How to use:** Read each topic's *Study notes* first, then try to answer each ladder question aloud before reading the answer. Climb the ladder (Easy → Tricky); stop when you can no longer answer and re-read.
Run all examples against the shared sample schema below (SQL Server 2019+ unless noted; features needing 2016/2017/2022 are flagged).

## Sample schema (reused throughout)

```sql
CREATE TABLE dbo.Departments (
    DeptId   INT PRIMARY KEY,
    DeptName NVARCHAR(50) NOT NULL
);

CREATE TABLE dbo.Employees (
    EmployeeId INT PRIMARY KEY,
    Name       NVARCHAR(50)  NOT NULL,
    DeptId     INT           NOT NULL REFERENCES dbo.Departments(DeptId),
    ManagerId  INT           NULL REFERENCES dbo.Employees(EmployeeId),
    Salary     DECIMAL(10,2) NOT NULL,
    HireDate   DATE          NOT NULL
);

CREATE TABLE dbo.Orders (
    OrderId    INT IDENTITY(1,1) PRIMARY KEY,
    CustomerId INT           NOT NULL,
    OrderDate  DATE          NOT NULL,
    Amount     DECIMAL(10,2) NOT NULL,
    Status     VARCHAR(10)   NOT NULL   -- 'New','Paid','Shipped'
);

INSERT dbo.Departments VALUES (1,'Engineering'),(2,'Sales'),(3,'HR');

INSERT dbo.Employees VALUES
 (1,'Asha',  1, NULL, 9000,'2015-01-10'),   -- CEO
 (2,'Bram',  1, 1,    7000,'2016-03-01'),
 (3,'Chen',  1, 2,    6000,'2018-07-15'),
 (4,'Dara',  1, 2,    6000,'2019-02-20'),   -- same salary as Chen (ties)
 (5,'Eli',   2, 1,    5000,'2017-05-05'),
 (6,'Fay',   2, 5,    4000,'2020-08-08'),
 (7,'Gus',   3, 1,    3500,'2021-11-30');

INSERT dbo.Orders (CustomerId, OrderDate, Amount, Status) VALUES
 (10,'2024-01-01',100,'Paid'),
 (10,'2024-01-02',150,'Paid'),
 (10,'2024-01-04',200,'Shipped'),
 (20,'2024-01-01', 80,'New'),
 (20,'2024-01-02', 80,'Paid'),
 (20,'2024-01-03', 90,'Paid'),
 (30,'2024-02-01',500,'Shipped');
```

## Table of contents

1. Window functions (ranking, LAG/LEAD, running totals, frames)
2. Recursive CTEs & MAXRECURSION
3. PIVOT / UNPIVOT & dynamic pivot
4. CROSS / OUTER APPLY & table-valued functions
5. Gaps-and-islands, deduplication, top-N-per-group
6. JSON, XML basics, STRING_AGG, STRING_SPLIT
7. Views (indexed, updatable, SCHEMABINDING)
8. Stored procedures
9. User-defined functions (scalar, inline TVF, multi-statement TVF, UDF inlining)
10. Triggers
11. Error handling & transactions (TRY/CATCH, THROW, XACT_ABORT, nesting myths)
12. Dynamic SQL (sp_executesql, injection)
13. Cursors vs set-based; temp tables vs table variables vs TVPs
14. Temporal tables, Change Tracking & CDC
15. Sequence/identity gaps, computed columns, MERGE alternatives
16. hierarchyid, full-text search, bulk operations, SQL Agent
17. ADO.NET (SqlClient) basics and Dapper

---

## Topic 1: Window functions

**Study notes**

A window function computes a value per row over a *set of related rows* (the window) **without collapsing rows** like GROUP BY does. Syntax:

```
func(...) OVER ( [PARTITION BY cols] [ORDER BY cols] [ROWS|RANGE frame] )
```

| Family | Functions | Needs ORDER BY? |
|---|---|---|
| Ranking | ROW_NUMBER, RANK, DENSE_RANK, NTILE | Yes |
| Offset | LAG, LEAD | Yes |
| Value | FIRST_VALUE, LAST_VALUE | Yes (frame matters) |
| Aggregate-as-window | SUM, AVG, COUNT, MIN, MAX ... OVER | Optional (ORDER BY => running aggregate) |

Ties for salary 6000,6000,5000: ROW_NUMBER 1,2,3 · RANK 1,1,3 · DENSE_RANK 1,1,2.

Frame clause: `ROWS BETWEEN <start> AND <end>` (physical rows) vs `RANGE` (logical: all peers with same ORDER BY value). Bounds: `UNBOUNDED PRECEDING`, `n PRECEDING`, `CURRENT ROW`, `n FOLLOWING`, `UNBOUNDED FOLLOWING`. **Default frame when ORDER BY is present and no frame given: `RANGE BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW`.** Without ORDER BY the frame is the whole partition. Window functions are evaluated after WHERE/GROUP BY/HAVING, so they can only appear in SELECT and ORDER BY — filter on them via a CTE/derived table (SQL Server has no QUALIFY).

### 1.1 [Easy] What is a window function and how does it differ from GROUP BY?

**Answer**
A window function calculates over a set of rows related to the current row (defined by OVER) and returns a value for *every* input row. GROUP BY collapses rows into one per group. Use windows when you need detail rows alongside aggregates.

**Explanation**
`PARTITION BY` is like GROUP BY inside the window but doesn't reduce rows. Omit it and the whole result set is one partition. Window functions can't be used in WHERE or GROUP BY because they are logically evaluated after those phases.

**Code**
```sql
SELECT Name, DeptId, Salary,
       SUM(Salary) OVER (PARTITION BY DeptId) AS DeptTotal,
       Salary * 100.0 / SUM(Salary) OVER (PARTITION BY DeptId) AS PctOfDept
FROM dbo.Employees;
-- 7 rows returned; e.g. Asha, Dept 1, 9000, DeptTotal 28000 (9000+7000+6000+6000)
```

### 1.2 [Follow-up] ROW_NUMBER vs RANK vs DENSE_RANK vs NTILE — differences with ties?

**Answer**
ROW_NUMBER gives unique sequential numbers (ties broken arbitrarily unless ORDER BY is unique). RANK gives equal rank to ties and leaves gaps; DENSE_RANK has no gaps. NTILE(n) splits ordered rows into n roughly equal buckets.

**Explanation**
For NTILE, if rows don't divide evenly the first buckets get one extra row (7 rows, NTILE(3) → sizes 3,2,2). Make ROW_NUMBER deterministic by adding a unique tiebreaker column to ORDER BY.

**Code**
```sql
SELECT Name, Salary,
  ROW_NUMBER() OVER (ORDER BY Salary DESC, EmployeeId) AS rn,
  RANK()       OVER (ORDER BY Salary DESC) AS rnk,
  DENSE_RANK() OVER (ORDER BY Salary DESC) AS drnk,
  NTILE(3)     OVER (ORDER BY Salary DESC, EmployeeId) AS bucket
FROM dbo.Employees;
-- Asha 9000 1 1 1 1 | Bram 7000 2 2 2 1 | Chen 6000 3 3 3 1
-- Dara 6000 4 3 3 2 | Eli 5000 5 5 4 2 | Fay 4000 6 6 5 3 | Gus 3500 7 7 6 3
```

### 1.3 [Medium] How do you compute a running total and month-over-month difference with LAG/LEAD?

**Answer**
Running total: `SUM(x) OVER (ORDER BY d ROWS UNBOUNDED PRECEDING)`. Previous/next values: `LAG(x,1,default) / LEAD(x,1,default) OVER (PARTITION BY ... ORDER BY ...)`. Diff = current − LAG.

**Explanation**
LAG's third argument is the default when there is no previous row (otherwise NULL). Always specify the partition so values don't leak across customers. Specify `ROWS` explicitly for running totals (see 1.4 why).

**Code**
```sql
SELECT CustomerId, OrderDate, Amount,
       SUM(Amount) OVER (PARTITION BY CustomerId ORDER BY OrderDate, OrderId
                         ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW) AS RunningTotal,
       Amount - LAG(Amount, 1, 0) OVER (PARTITION BY CustomerId ORDER BY OrderDate, OrderId) AS DeltaVsPrev,
       LEAD(OrderDate) OVER (PARTITION BY CustomerId ORDER BY OrderDate, OrderId) AS NextOrderDate
FROM dbo.Orders;
-- Customer 10: running 100,250,450 ; delta 100,50,50 ; NextOrderDate 01-02, 01-04, NULL
```

### 1.4 [Hard] Explain the ROWS vs RANGE default-frame trap (and the LAST_VALUE gotcha).

**Answer**
With ORDER BY and no frame, SQL Server uses `RANGE UNBOUNDED PRECEDING AND CURRENT ROW`. RANGE treats rows with equal ORDER BY values as peers and includes all of them, so running totals show duplicates' combined sum on every tied row. RANGE with the default is also slower because it uses an on-disk worktable, while ROWS uses an in-memory spool. LAST_VALUE with the default frame returns the *current row's* value (not the partition's last), a classic bug.

**Explanation**
RANGE only supports UNBOUNDED/CURRENT ROW bounds in SQL Server (no `RANGE 5 PRECEDING`). Fix: always write `ROWS BETWEEN ...`, and for LAST_VALUE use `ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING` (or use FIRST_VALUE with reversed ORDER BY).

**Code**
```sql
-- Customer 20 has 3 rows; make a tie by looking at a date duplicated across customers
SELECT OrderId, OrderDate, Amount,
  SUM(Amount) OVER (ORDER BY OrderDate)                         AS RangeDefault,
  SUM(Amount) OVER (ORDER BY OrderDate ROWS UNBOUNDED PRECEDING) AS RowsRunning
FROM dbo.Orders WHERE OrderDate <= '2024-01-02';
-- 2024-01-01 has two rows (100, 80): RangeDefault = 180 for BOTH rows; RowsRunning = 100 and 180 (order among ties arbitrary)
-- 2024-01-02 has two rows (150, 80): RangeDefault = 410 for both; RowsRunning = 330 / 410

SELECT Name, Salary,
  LAST_VALUE(Name) OVER (ORDER BY Salary)  AS WrongLast,   -- = Name itself (or tie peer)
  LAST_VALUE(Name) OVER (ORDER BY Salary
        ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING) AS RealLast -- Asha
FROM dbo.Employees;
```

**Gotchas / follow-ups**
- Ties + ROWS running total → nondeterministic split unless ORDER BY has a unique tiebreaker.
- Window functions can't be nested inside each other; wrap in CTE.
- `IGNORE NULLS`/`RESPECT NULLS` for FIRST_VALUE/LAG is supported only from SQL Server 2022.

### 1.5 [Tricky] How do you filter on a window function result, and what happens to NULLs in ORDER BY / ranking?

**Answer**
You can't use it in WHERE; compute it in a CTE/subquery and filter outside. NULLs sort first in ASC (lowest) and last in DESC in SQL Server, which affects ROW_NUMBER/RANK and LAG ordering. Also an ORDER BY inside OVER does not guarantee the final result ordering.

**Explanation**
The final presentation order still requires an outer ORDER BY. `ROW_NUMBER() OVER (ORDER BY (SELECT NULL))` assigns arbitrary numbers without sorting cost (but no guaranteed order).

**Code**
```sql
WITH r AS (
  SELECT Name, DeptId, Salary,
         DENSE_RANK() OVER (PARTITION BY DeptId ORDER BY Salary DESC) AS rk
  FROM dbo.Employees)
SELECT Name, DeptId, Salary FROM r WHERE rk = 1 ORDER BY DeptId;
-- Asha(1), Eli(2), Gus(3)

-- Invalid: SELECT ... WHERE ROW_NUMBER() OVER (...) = 1;  -- Msg 4108
```

**Gotchas / follow-ups**
- A supporting index on (PartitionCols, OrderCols) INCLUDE (needed cols) removes the Sort operator.

**Interview traps**
- "RANK vs DENSE_RANK" — remember gaps vs no gaps with a concrete 1,1,3 example.
- Default frame is RANGE, not ROWS; always spell out ROWS for running totals.
- LAST_VALUE bug with default frame.
- Window functions can't appear in WHERE/GROUP BY/HAVING.
- Non-unique ORDER BY in ROW_NUMBER → non-deterministic results.

**Cheat sheet**
- OVER(PARTITION BY .. ORDER BY .. frame) — rows are kept.
- Ranking: ROW_NUMBER unique, RANK gaps, DENSE_RANK none, NTILE buckets.
- Default frame with ORDER BY = RANGE UNBOUNDED PRECEDING..CURRENT ROW.
- Use ROWS for running totals; unbounded-following frame for LAST_VALUE.
- Filter via CTE; index on partition+order columns avoids sorts.

---

## Topic 2: Recursive CTEs & MAXRECURSION

**Study notes**

A recursive CTE has an **anchor member** (non-recursive), `UNION ALL`, and a **recursive member** that references the CTE name. Execution: run anchor → result set 0; repeatedly feed the previous iteration's rows to the recursive member until it returns no rows.

```
WITH cte AS (
   anchor SELECT ...            -- level 0
   UNION ALL
   recursive SELECT ... FROM t JOIN cte ON ...   -- level n+1
)
SELECT ... FROM cte OPTION (MAXRECURSION n);
```

Default recursion limit is **100** (error 530 when exceeded); `MAXRECURSION 0` = unlimited (dangerous with cycles). Restrictions in the recursive member: no `TOP`, `DISTINCT`, `GROUP BY`, aggregates, `LEFT/RIGHT/FULL OUTER JOIN` with the CTE on the outer side, or subqueries referencing the CTE; the column types of anchor and recursive parts must match exactly (cast in the anchor). Typical uses: org charts, bill-of-materials, paths, date/number series.

### 2.1 [Easy] Write a recursive CTE to list all employees under Asha with their level.

**Answer**
Anchor selects the root (ManagerId IS NULL); the recursive member joins Employees to the CTE on `ManagerId = cte.EmployeeId` and increments level.

**Explanation**
Each iteration finds the direct reports of the previous iteration's rows. Termination happens naturally when leaf employees have no reports.

**Code**
```sql
WITH Org AS (
    SELECT EmployeeId, Name, ManagerId, 0 AS Lvl
    FROM dbo.Employees WHERE ManagerId IS NULL
    UNION ALL
    SELECT e.EmployeeId, e.Name, e.ManagerId, o.Lvl + 1
    FROM dbo.Employees e JOIN Org o ON e.ManagerId = o.EmployeeId
)
SELECT Lvl, Name FROM Org ORDER BY Lvl, Name;
-- 0 Asha | 1 Bram,Eli,Gus | 2 Chen,Dara,Fay
```

### 2.2 [Follow-up] How do you produce an indented tree/path (e.g. "Asha > Bram > Chen") in correct order?

**Answer**
Carry a path string (and a sortable path) through recursion and ORDER BY the sort path. Cast the anchor path to a wide enough type, since anchor and recursive types must match.

**Explanation**
Without ORDER BY on a path, output order is not guaranteed (it often appears breadth-first). Use a zero-padded id path or the name path for depth-first display.

**Code**
```sql
WITH Org AS (
    SELECT EmployeeId, Name, CAST(Name AS NVARCHAR(400)) AS Path, 0 AS Lvl
    FROM dbo.Employees WHERE ManagerId IS NULL
    UNION ALL
    SELECT e.EmployeeId, e.Name, CAST(o.Path + N' > ' + e.Name AS NVARCHAR(400)), o.Lvl + 1
    FROM dbo.Employees e JOIN Org o ON e.ManagerId = o.EmployeeId
)
SELECT REPLICATE('  ', Lvl) + Name AS Tree, Path FROM Org ORDER BY Path;
-- Asha / Asha > Bram / Asha > Bram > Chen / Asha > Bram > Dara / Asha > Eli / Asha > Eli > Fay / Asha > Gus
```

### 2.3 [Medium] Generate a date series with a recursive CTE; what about MAXRECURSION?

**Answer**
Anchor = start date; recursive member = `DATEADD(DAY,1,d)` WHILE `d < end`. Beyond 100 rows add `OPTION (MAXRECURSION 0)` (or a specific number, max 32767 for a nonzero value).

**Explanation**
The OPTION hint goes on the *outer* statement using the CTE, not inside the CTE. For large series a numbers/tally table (or `GENERATE_SERIES` in SQL Server 2022) is faster, since recursion is row-by-row.

**Code**
```sql
WITH d AS (
    SELECT CAST('2024-01-01' AS DATE) AS dt
    UNION ALL
    SELECT DATEADD(DAY, 1, dt) FROM d WHERE dt < '2024-12-31'
)
SELECT COUNT(*) AS Days FROM d OPTION (MAXRECURSION 400);   -- 366 (2024 is a leap year)
-- Without the OPTION: Msg 530 "maximum recursion 100 has been exhausted"

-- SQL Server 2022+:
SELECT DATEADD(DAY, value, '2024-01-01') FROM GENERATE_SERIES(0, 365);
```

### 2.4 [Hard] How do you handle cycles and what does MAXRECURSION really protect you from?

**Answer**
SQL Server has no `CYCLE` clause. Detect cycles by keeping the visited path and excluding rows already in it (e.g. `CHARINDEX` on a delimited id path). MAXRECURSION only caps *iterations (depth)*, so it throws an error rather than correctly handling the cycle, and `MAXRECURSION 0` on cyclic data runs forever.

**Explanation**
The limit counts recursion levels, not rows: a wide tree of 1M rows at depth 5 is fine. An error aborts the whole statement. To find hierarchy loops, a path-check predicate keeps the graph traversal finite.

**Code**
```sql
-- Suppose bad data: Asha.ManagerId = Chen (cycle). Safe traversal:
WITH Walk AS (
    SELECT EmployeeId, ManagerId,
           CAST('|' + CAST(EmployeeId AS VARCHAR(10)) + '|' AS VARCHAR(4000)) AS Seen
    FROM dbo.Employees WHERE EmployeeId = 1
    UNION ALL
    SELECT e.EmployeeId, e.ManagerId,
           CAST(w.Seen + CAST(e.EmployeeId AS VARCHAR(10)) + '|' AS VARCHAR(4000))
    FROM dbo.Employees e JOIN Walk w ON e.ManagerId = w.EmployeeId
    WHERE w.Seen NOT LIKE '%|' + CAST(e.EmployeeId AS VARCHAR(10)) + '|%'
)
SELECT * FROM Walk OPTION (MAXRECURSION 1000);
```

**Gotchas / follow-ups**
- Anchor/recursive column type or nullability mismatch → Msg 240 ("types don't match between the anchor and the recursive part").
- Query plan shows Index Spool (Lazy) — each iteration is a nested-loop lookup, so index `ManagerId`.
- Alternative for read-heavy trees: `hierarchyid` (Topic 16) or materialized path.

### 2.5 [Tricky] Can the recursive member contain an aggregate, TOP, or an outer join? How to compute subtree totals?

**Answer**
No — aggregates, TOP, DISTINCT, GROUP BY and outer join on the CTE are disallowed in the recursive member. Compute subtree totals by recursing to produce (root, descendant) pairs, then aggregate outside the CTE.

**Explanation**
Emit each descendant tagged with the ancestor it rolls up to, then GROUP BY ancestor in the outer query.

**Code**
```sql
WITH Sub AS (
    SELECT EmployeeId AS RootId, EmployeeId AS MemberId FROM dbo.Employees
    UNION ALL
    SELECT s.RootId, e.EmployeeId
    FROM Sub s JOIN dbo.Employees e ON e.ManagerId = s.MemberId
)
SELECT r.Name, SUM(m.Salary) AS SubtreeSalary
FROM Sub s
JOIN dbo.Employees r ON r.EmployeeId = s.RootId
JOIN dbo.Employees m ON m.EmployeeId = s.MemberId
GROUP BY r.Name ORDER BY SubtreeSalary DESC;
-- Asha 40500 (all) | Bram 19000 (7000+6000+6000) | Eli 9000 | Chen 6000 | Dara 6000 | Fay 4000 | Gus 3500
```

**Interview traps**
- Forgetting UNION ALL (plain UNION is not allowed in the recursive part).
- MAXRECURSION is a statement-level OPTION, default 100, 0 = unlimited.
- Type mismatch between anchor and recursive column (string path lengths!).
- Assuming output order without ORDER BY.
- Using recursion for big number series when a tally table is faster.

**Cheat sheet**
- Anchor UNION ALL recursive; terminates when recursive part returns zero rows.
- Limit 100 default; `OPTION (MAXRECURSION n)`; 0 = no limit.
- No CYCLE keyword — track a path.
- No aggregates/TOP/DISTINCT/outer join in recursive member.
- Index the parent key column.

---

## Topic 3: PIVOT / UNPIVOT & dynamic pivot

**Study notes**

PIVOT rotates distinct row values into columns with an aggregate; UNPIVOT does the reverse.

```
SELECT <non-pivot cols>, [v1],[v2]
FROM (source) AS src
PIVOT ( AGG(valueCol) FOR pivotCol IN ([v1],[v2]) ) AS p;
```

Rules: the IN list must be **static** literals; all columns not named in the aggregate/FOR become implicit grouping keys (so select only needed columns in the source subquery); missing combinations yield NULL. Dynamic pivots build the column list with `STRING_AGG`/`STUFF+FOR XML` and run via `sp_executesql`, quoting names with `QUOTENAME`. UNPIVOT drops NULLs and needs same-typed columns. The portable alternative is conditional aggregation (`SUM(CASE WHEN ... END)`), often just as fast and more flexible. `CROSS APPLY (VALUES ...)` is the modern unpivot.

### 3.1 [Easy] Pivot order amounts by status per customer.

**Answer**
Select customer, status, amount in a derived table and PIVOT SUM(Amount) FOR Status IN ([New],[Paid],[Shipped]).

**Explanation**
The derived table limits columns so OrderId etc. don't become grouping keys.

**Code**
```sql
SELECT CustomerId, [New], [Paid], [Shipped]
FROM (SELECT CustomerId, Status, Amount FROM dbo.Orders) s
PIVOT (SUM(Amount) FOR Status IN ([New],[Paid],[Shipped])) p;
-- 10: NULL,250,200 | 20: 80,170,NULL | 30: NULL,NULL,500
```

### 3.2 [Follow-up] Do the same with conditional aggregation — why might you prefer it?

**Answer**
`SUM(CASE WHEN Status='Paid' THEN Amount END)`. It allows multiple aggregates, different conditions per column, and COALESCE defaults, and the intent is clearer than PIVOT syntax.

**Explanation**
PIVOT permits only one aggregate function and one pivot column. Conditional aggregation yields the same plan (hash/stream aggregate).

**Code**
```sql
SELECT CustomerId,
  SUM(CASE WHEN Status='New'     THEN Amount ELSE 0 END) AS [New],
  SUM(CASE WHEN Status='Paid'    THEN Amount ELSE 0 END) AS [Paid],
  COUNT(CASE WHEN Status='Paid'  THEN 1 END)             AS PaidCount
FROM dbo.Orders GROUP BY CustomerId;
-- 10: 0,250,2 | 20: 80,170,2 | 30: 0,0,0
```

### 3.3 [Medium] How do you UNPIVOT, and what is the CROSS APPLY VALUES alternative?

**Answer**
`UNPIVOT (value FOR name IN (c1,c2))` turns columns into rows but silently drops NULLs. `CROSS APPLY (VALUES ('c1',c1),('c2',c2)) v(name,value)` is more flexible (keeps NULLs, mixed types after conversion).

**Explanation**
UNPIVOT requires identical data types across the listed columns.

**Code**
```sql
SELECT p.CustomerId, v.Status, v.Amt
FROM (SELECT CustomerId,
             SUM(CASE WHEN Status='New'  THEN Amount END) AS [New],
             SUM(CASE WHEN Status='Paid' THEN Amount END) AS [Paid]
      FROM dbo.Orders GROUP BY CustomerId) p
CROSS APPLY (VALUES ('New', p.[New]), ('Paid', p.[Paid])) v(Status, Amt);
-- 6 rows, NULL Amts retained (UNPIVOT would return only non-NULL rows)
```

### 3.4 [Hard] Build a dynamic pivot when statuses are not known in advance — safely.

**Answer**
Build the column list from the data using `STRING_AGG(QUOTENAME(Status), ',')`, concatenate into a PIVOT statement, and execute with `sp_executesql`. `QUOTENAME` protects against injection via data values.

**Explanation**
The query text is built at runtime so plan reuse is per distinct column list. Static parts that are user input (like filter values) should still be passed as parameters. `STRING_AGG` requires SQL Server 2017+; older versions use `STUFF((SELECT ',' + QUOTENAME(x) ... FOR XML PATH(''), TYPE).value('.','nvarchar(max)'),1,1,'')`.

**Code**
```sql
DECLARE @cols NVARCHAR(MAX) =
  (SELECT STRING_AGG(QUOTENAME(Status), ',') WITHIN GROUP (ORDER BY Status)
   FROM (SELECT DISTINCT Status FROM dbo.Orders) d);

DECLARE @sql NVARCHAR(MAX) = N'
SELECT CustomerId, ' + @cols + N'
FROM (SELECT CustomerId, Status, Amount FROM dbo.Orders WHERE OrderDate >= @from) s
PIVOT (SUM(Amount) FOR Status IN (' + @cols + N')) p;';

EXEC sp_executesql @sql, N'@from DATE', @from = '2024-01-01';
-- Same result as 3.1; adding a new status automatically adds a column
```

**Gotchas / follow-ups**
- Forgetting QUOTENAME → injection and breaks on names with spaces.
- `NVARCHAR(MAX)` for the variable; plain VARCHAR(8000) silently truncates.
- Result column set varies, so you cannot consume it into a typed C# DTO; read into DataTable/dynamic.

### 3.5 [Tricky] Why does my PIVOT return multiple rows per customer / unexpected NULLs?

**Answer**
Extra columns in the source subquery (e.g. OrderId, OrderDate) become implicit grouping columns, so you get one row per unique combination. Select only the grouping key, pivot column and value column. NULLs appear for missing combinations; wrap in `ISNULL` in the outer SELECT.

**Explanation**
Also PIVOT is case/collation sensitive about IN list values matching data, and non-matching values are silently ignored (rows with Status='Cancelled' vanish if not in the list).

**Code**
```sql
SELECT CustomerId, [Paid]
FROM dbo.Orders
PIVOT (SUM(Amount) FOR Status IN ([Paid])) p;
-- Wrong: OrderId, OrderDate still in the source → one row per order (7 groups), not per customer
```

**Interview traps**
- Static IN list only; unknown pivot values are dropped silently.
- Extra source columns create unintended groups.
- UNPIVOT eliminates NULL rows.
- Dynamic pivot without QUOTENAME / with VARCHAR truncation.

**Cheat sheet**
- PIVOT = aggregate + FOR col IN (static list).
- Conditional aggregation = same result, more flexible.
- UNPIVOT drops NULLs; CROSS APPLY VALUES keeps them.
- Dynamic: STRING_AGG(QUOTENAME) + sp_executesql.

---

## Topic 4: CROSS / OUTER APPLY & table-valued functions

**Study notes**

`APPLY` invokes a table expression (inline TVF, derived table, or `VALUES`) **once per outer row**, allowing correlation to outer columns — something a JOIN's right side can't do.

| Operator | Like | Outer row with empty inner result |
|---|---|---|
| CROSS APPLY | INNER JOIN | dropped |
| OUTER APPLY | LEFT JOIN | kept, inner columns NULL |

Typical uses: top-N per group, calling a TVF per row, splitting/unpivoting values, reusing a computed expression (`CROSS APPLY (SELECT x*2 AS y) a`), `STRING_SPLIT`/`OPENJSON` per row. The optimizer often turns APPLY into a normal join/semi-join, but with TOP inside it often stays a Nested Loops with an index seek per outer row — fast if the index supports it.

### 4.1 [Easy] What is the difference between CROSS APPLY and OUTER APPLY?

**Answer**
CROSS APPLY returns only outer rows for which the right-side expression returns at least one row (like INNER JOIN); OUTER APPLY keeps all outer rows with NULLs when empty (like LEFT JOIN).

**Explanation**
Use OUTER APPLY when absence is meaningful (e.g. employees with no reports).

**Code**
```sql
SELECT m.Name, x.ReportCount
FROM dbo.Employees m
OUTER APPLY (SELECT COUNT(*) AS ReportCount FROM dbo.Employees r WHERE r.ManagerId = m.EmployeeId) x;
-- Asha 3 | Bram 2 | Chen 0 | Dara 0 | Eli 1 | Fay 0 | Gus 0   (aggregate always returns a row)

SELECT d.DeptName, t.Name
FROM dbo.Departments d
OUTER APPLY (SELECT TOP (1) Name FROM dbo.Employees e WHERE e.DeptId = d.DeptId ORDER BY Salary DESC) t;
```

### 4.2 [Follow-up] Why use APPLY instead of JOIN? Give the top-N-per-group example.

**Answer**
APPLY lets the right side reference the left row, so you can use TOP ... ORDER BY per outer row or call a parameterized function. A JOIN can't correlate that way.

**Explanation**
Top 2 highest-paid employees per department: for each dept, a TOP(2) subquery. With an index on `(DeptId, Salary DESC)` this is an efficient seek per department; usually beats ROW_NUMBER when there are few groups and many rows per group.

**Code**
```sql
SELECT d.DeptName, t.Name, t.Salary
FROM dbo.Departments d
CROSS APPLY (SELECT TOP (2) e.Name, e.Salary
             FROM dbo.Employees e
             WHERE e.DeptId = d.DeptId
             ORDER BY e.Salary DESC, e.EmployeeId) t;
-- Engineering: Asha 9000, Bram 7000 | Sales: Eli 5000, Fay 4000 | HR: Gus 3500
```

### 4.3 [Medium] Inline TVF + APPLY: write one and explain how it differs from a view with parameters.

**Answer**
An inline TVF (`RETURNS TABLE AS RETURN (SELECT ...)`) is essentially a parameterized view: the optimizer expands it into the calling query. APPLY calls it per outer row but the plan is merged, so it is efficient and supports predicate pushdown.

**Explanation**
Views can't take parameters; inline TVFs can. Multi-statement TVFs by contrast are black boxes (Topic 9).

**Code**
```sql
CREATE OR ALTER FUNCTION dbo.fn_TopOrders (@CustomerId INT, @N INT)
RETURNS TABLE AS RETURN
(
    SELECT TOP (@N) OrderId, OrderDate, Amount
    FROM dbo.Orders WHERE CustomerId = @CustomerId
    ORDER BY Amount DESC
);
GO
SELECT DISTINCT o.CustomerId, f.OrderId, f.Amount
FROM dbo.Orders o CROSS APPLY dbo.fn_TopOrders(o.CustomerId, 1) f;
-- One row per customer: 10 → 200 ; 20 → 90 ; 30 → 500
```

### 4.4 [Hard] When can APPLY be slow, and how do you diagnose it?

**Answer**
APPLY with a non-sargable or unindexed correlated predicate does a full scan per outer row (Nested Loops with a table scan inside), giving O(n·m). Check the actual plan for the inner-side scan and "Number of Executions". Fix by indexing the correlation + ORDER BY columns, or rewrite with ROW_NUMBER / a set-based join.

**Explanation**
Conversely, with many groups and no index, `ROW_NUMBER()` does one scan + sort, which wins; with few groups and a perfect index, APPLY+TOP wins. Measure with `SET STATISTICS IO ON`.

**Code**
```sql
SET STATISTICS IO ON;
-- Supporting index for 4.2:
CREATE INDEX IX_Emp_Dept_Salary ON dbo.Employees (DeptId, Salary DESC) INCLUDE (Name);
-- Re-run 4.2: plan shows Index Seek on IX_Emp_Dept_Salary with Top operator, 1 seek per department.
```

**Gotchas / follow-ups**
- OUTER APPLY of a multi-statement TVF → fixed 1-row (2014+: 100-row) estimate and no inlining.
- `CROSS APPLY` to a scalar subquery that returns zero rows silently removes the outer row — a frequent cause of "missing data".

### 4.5 [Tricky] CROSS APPLY returns fewer rows than expected — why? And can you use APPLY to reuse expressions?

**Answer**
Zero-row inner results drop the outer row under CROSS APPLY; use OUTER APPLY. Aggregate-only subqueries (no GROUP BY) always return a row so are unaffected. You can also use `CROSS APPLY (SELECT <expr> AS alias) a` to define a computed value once and reference it in WHERE/ORDER BY/other expressions.

**Explanation**
This works around T-SQL's restriction that a SELECT alias can't be referenced within the same SELECT list or WHERE.

**Code**
```sql
SELECT e.Name, c.AnnualSalary
FROM dbo.Employees e
CROSS APPLY (SELECT e.Salary * 12 AS AnnualSalary) c
WHERE c.AnnualSalary > 60000;
-- Asha 108000, Bram 84000, Chen 72000, Dara 72000 (Eli = 60000 excluded: not >)
```

**Interview traps**
- CROSS APPLY ≈ INNER JOIN semantics; many forget the dropped rows.
- Thinking APPLY is always a loop — optimizer may flatten it.
- Multi-statement TVF in APPLY → terrible cardinality estimates.
- TOP without ORDER BY in the inner query = arbitrary rows.

**Cheat sheet**
- CROSS APPLY = inner, OUTER APPLY = left.
- Right side can reference left columns.
- Best for top-N-per-group, per-row TVF, STRING_SPLIT/OPENJSON expansion, alias reuse.
- Index (group key, order key) for APPLY+TOP.
- Prefer inline TVFs.

---

## Topic 5: Gaps-and-islands, deduplication, top-N-per-group

**Study notes**

Three classic set-based patterns, all built on window functions:

| Pattern | Core trick |
|---|---|
| Islands (consecutive runs) | `value - ROW_NUMBER() OVER (ORDER BY value)` is constant within a run → group by it |
| Gaps | `LEAD(value) OVER (ORDER BY value) - value > 1` marks where a gap starts |
| Dedup | `ROW_NUMBER() OVER (PARTITION BY dupeKey ORDER BY keeper)`, delete rn > 1 (can DELETE through a CTE) |
| Top-N per group | `ROW_NUMBER/DENSE_RANK` in CTE then `WHERE rn <= N`, or `CROSS APPLY (SELECT TOP N ...)` |

For dates convert to an integer first (`DATEDIFF(DAY,'19000101',d)`), and de-duplicate the values first if they can repeat (ROW_NUMBER trick needs distinct values).

### 5.1 [Easy] Delete duplicate rows keeping one.

**Answer**
Number rows within each duplicate group with ROW_NUMBER and delete those with rn > 1 via a CTE.

**Explanation**
Deleting from a CTE deletes from the underlying single base table. Choose ORDER BY to decide which row survives (e.g. lowest id).

**Code**
```sql
-- Treat (CustomerId, OrderDate, Amount) as the duplicate key
WITH d AS (
  SELECT ROW_NUMBER() OVER (PARTITION BY CustomerId, OrderDate, Amount ORDER BY OrderId) AS rn
  FROM dbo.Orders)
DELETE FROM d WHERE rn > 1;
-- No duplicates in sample data → 0 rows affected
```

### 5.2 [Follow-up] Top-N-per-group: the highest-paid employee per department (including ties?).

**Answer**
Use `ROW_NUMBER` for exactly one row per group (arbitrary among ties unless tiebreaker) or `RANK`/`DENSE_RANK` to include ties. Filter in an outer query.

**Explanation**
Alternative: `CROSS APPLY (SELECT TOP (1) WITH TIES ...)`. WITH TIES requires ORDER BY and returns all tied rows.

**Code**
```sql
WITH r AS (
  SELECT Name, DeptId, Salary,
         RANK() OVER (PARTITION BY DeptId ORDER BY Salary DESC) AS rk
  FROM dbo.Employees)
SELECT DeptId, Name, Salary FROM r WHERE rk <= 2;
-- Dept1: Asha 9000, Bram 7000 | Dept2: Eli, Fay | Dept3: Gus
-- Note: with RANK <= 3 in Dept 1 you'd get Chen AND Dara (tie)
```

### 5.3 [Medium] Find islands: consecutive-day order streaks per customer.

**Answer**
Compute `DATEADD(DAY, -ROW_NUMBER() OVER (PARTITION BY CustomerId ORDER BY OrderDate), OrderDate)` — constant for consecutive days — then GROUP BY customer and that key.

**Explanation**
If a customer can have multiple orders per day, SELECT DISTINCT customer/date first, otherwise the row number offsets break the constant.

**Code**
```sql
WITH days AS (SELECT DISTINCT CustomerId, OrderDate FROM dbo.Orders),
g AS (
  SELECT CustomerId, OrderDate,
         DATEADD(DAY, -ROW_NUMBER() OVER (PARTITION BY CustomerId ORDER BY OrderDate), OrderDate) AS grp
  FROM days)
SELECT CustomerId, MIN(OrderDate) AS StreakStart, MAX(OrderDate) AS StreakEnd,
       COUNT(*) AS Days
FROM g GROUP BY CustomerId, grp;
-- 10: 01-01..01-02 (2 days) and 01-04..01-04 (1) | 20: 01-01..01-03 (3) | 30: 02-01 (1)
```

### 5.4 [Hard] Find the gaps (missing numbers/dates) and compare with the islands approach.

**Answer**
Use LEAD to compare each value with the next; where the next is more than 1 higher, a gap spans (value+1) to (next−1). Gaps and islands are complements: islands are the present runs, gaps the missing ranges between them.

**Explanation**
This performs one ordered scan (index on the column) — far better than a self-join `NOT EXISTS (value+1)` which can be quadratic without an index.

**Code**
```sql
WITH n AS (
  SELECT OrderDate AS cur,
         LEAD(OrderDate) OVER (ORDER BY OrderDate) AS nxt
  FROM (SELECT DISTINCT OrderDate FROM dbo.Orders WHERE CustomerId = 10) x)
SELECT DATEADD(DAY,1,cur) AS GapStart, DATEADD(DAY,-1,nxt) AS GapEnd
FROM n WHERE DATEDIFF(DAY, cur, nxt) > 1;
-- 2024-01-03 .. 2024-01-03
```

**Gotchas / follow-ups**
- Gap before the first value / after the last aren't detected unless you add boundaries.
- Status-change islands (consecutive identical statuses): use `ROW_NUMBER() OVER (ORDER BY d) - ROW_NUMBER() OVER (PARTITION BY status ORDER BY d)`.

### 5.5 [Tricky] Dedupe edge cases: deleting dupes in a table with no key, NULLs, and huge tables.

**Answer**
ROW_NUMBER over `PARTITION BY` works on heaps without a key (the CTE delete still targets physical rows). NULLs in PARTITION BY group together (unlike `=` comparison), which usually is what you want. For huge tables, batch the delete (`DELETE TOP (n)` loop) or copy distinct rows to a new table and swap to limit log growth and blocking.

**Explanation**
Unlike `GROUP BY`/`PARTITION BY`, a JOIN-based dedupe `a.x = b.x` doesn't match NULLs. To prevent recurrence add a UNIQUE constraint or unique index (`IGNORE_DUP_KEY = ON` silently skips duplicates).

**Code**
```sql
-- Capture the keys to delete once, then delete in batches
SELECT OrderId INTO #ToDelete
FROM (SELECT OrderId,
             ROW_NUMBER() OVER (PARTITION BY CustomerId, OrderDate, Amount ORDER BY OrderId) AS rn
      FROM dbo.Orders) x
WHERE rn > 1;
CREATE UNIQUE CLUSTERED INDEX IX ON #ToDelete(OrderId);

WHILE 1 = 1
BEGIN
  DELETE TOP (5000) o FROM dbo.Orders o JOIN #ToDelete t ON t.OrderId = o.OrderId;
  IF @@ROWCOUNT = 0 BREAK;
END
-- Putting TOP inside the ROW_NUMBER CTE would pick arbitrary rows (maybe all rn = 1) - avoid that.
```

**Interview traps**
- ROW_NUMBER islands technique needs distinct values.
- RANK vs ROW_NUMBER for "top N" when ties matter.
- Forgetting a tiebreaker → non-deterministic survivors.
- DELETE on huge tables in one transaction → log blow-up and lock escalation.

**Cheat sheet**
- Islands: value − ROW_NUMBER() = group key.
- Gaps: LEAD(value) − value > 1.
- Dedup: ROW_NUMBER over dupe key, DELETE rn>1 via CTE.
- Top N per group: window filter in CTE, or CROSS APPLY TOP.
- `TOP (n) WITH TIES` includes ties.

---

## Topic 6: JSON, XML basics, STRING_AGG, STRING_SPLIT

**Study notes**

JSON support (2016+) is **text-based** — there is no JSON data type (a native `json` type arrived only in Azure SQL / SQL Server 2025). Store in `NVARCHAR(MAX)` and validate with `ISJSON()`.

| Function | Purpose |
|---|---|
| `JSON_VALUE(json, path)` | extract a scalar (returns NVARCHAR(4000)); NULL if missing (lax mode) |
| `JSON_QUERY(json, path)` | extract an object/array |
| `JSON_MODIFY(json, path, v)` | update/insert/delete a property |
| `OPENJSON(json [, path]) [WITH (...)]` | shred JSON into rows/columns |
| `FOR JSON PATH / AUTO` | format a result set as JSON |
| `ISJSON` | validity check |

Path syntax: `'$.address.city'`, `'$.items[0].sku'`; prefix `strict` to error on missing paths. To index a JSON property: add a computed column `AS JSON_VALUE(doc,'$.city')` and index it.

XML: `xml` data type with methods `.value()`, `.query()`, `.nodes()`, `.exist()`, `.modify()`; `FOR XML PATH/RAW/AUTO`; `OPENXML` (legacy, needs `sp_xml_preparedocument`/remove). XQuery paths are case-sensitive.

String aggregation: `STRING_AGG(expr, sep) [WITHIN GROUP (ORDER BY ...)]` (2017+) ignores NULLs; `STRING_SPLIT(str, sep [, enable_ordinal])` (2016+) returns a `value` column; the ordinal (3rd) arg is **2022+**; before that ordering isn't guaranteed.

### 6.1 [Easy] Extract a scalar from JSON and produce JSON from a query.

**Answer**
`JSON_VALUE(doc,'$.prop')` returns a scalar string; `SELECT ... FOR JSON PATH` returns query rows as a JSON array of objects.

**Explanation**
JSON_VALUE returns NVARCHAR(4000), so wrap with CAST/TRY_CAST for numbers. Use `JSON_QUERY` (not JSON_VALUE) for nested objects/arrays, or you'll get NULL.

**Code**
```sql
DECLARE @j NVARCHAR(MAX) = N'{"name":"Asha","address":{"city":"Pune"},"skills":["C#","SQL"]}';
SELECT JSON_VALUE(@j,'$.name')          AS Name,       -- Asha
       JSON_VALUE(@j,'$.address.city')  AS City,       -- Pune
       JSON_QUERY(@j,'$.skills')        AS Skills,     -- ["C#","SQL"]
       JSON_VALUE(@j,'$.skills')        AS WrongNull;  -- NULL (array, not scalar)

SELECT TOP (2) EmployeeId, Name FROM dbo.Employees ORDER BY EmployeeId FOR JSON PATH;
-- [{"EmployeeId":1,"Name":"Asha"},{"EmployeeId":2,"Name":"Bram"}]
```

### 6.2 [Follow-up] Shred a JSON array into rows with OPENJSON.

**Answer**
`OPENJSON(@json) WITH (col type '$.path', ...)` returns typed columns; use `CROSS APPLY OPENJSON(t.jsonCol)` to expand per-row JSON. Use `AS JSON` for nested fragments.

**Explanation**
Without WITH, OPENJSON returns key/value/type columns (type 0=null,1=string,2=number,3=bool,4=array,5=object).

**Code**
```sql
DECLARE @orders NVARCHAR(MAX) = N'[{"id":1,"amt":100.5,"tags":["a","b"]},{"id":2,"amt":20}]';
SELECT * FROM OPENJSON(@orders)
WITH (Id INT '$.id', Amt DECIMAL(10,2) '$.amt', Tags NVARCHAR(MAX) '$.tags' AS JSON);
-- 1 | 100.50 | ["a","b"]
-- 2 |  20.00 | NULL
```

### 6.3 [Medium] STRING_AGG vs old FOR XML PATH concatenation; STRING_SPLIT usage and limits.

**Answer**
`STRING_AGG` is the 2017+ built-in; it supports `WITHIN GROUP (ORDER BY)` and ignores NULLs. Old approach: `STUFF((SELECT ',' + x FROM t FOR XML PATH(''),TYPE).value('.','nvarchar(max)'),1,1,'')`. `STRING_SPLIT` returns rows but, before 2022, no ordinal and no guaranteed order.

**Explanation**
STRING_AGG result type: if input is non-MAX varchar/nvarchar, the result is limited to 8000/4000 chars → overflow error 9829; cast to `NVARCHAR(MAX)` first. STRING_SPLIT values may include empty strings for adjacent separators and aren't trimmed. Compatibility level 130+ required for STRING_SPLIT.

**Code**
```sql
SELECT d.DeptName,
       STRING_AGG(e.Name, ', ') WITHIN GROUP (ORDER BY e.Name) AS Members
FROM dbo.Departments d JOIN dbo.Employees e ON e.DeptId = d.DeptId
GROUP BY d.DeptName;
-- Engineering: Asha, Bram, Chen, Dara | HR: Gus | Sales: Eli, Fay

SELECT value FROM STRING_SPLIT('1,2,,3', ',');   -- 1 | 2 | (empty) | 3
SELECT * FROM dbo.Employees WHERE EmployeeId IN (SELECT CAST(value AS INT) FROM STRING_SPLIT('1,3,5', ','));
-- Asha, Chen, Eli
-- 2022+: SELECT value, ordinal FROM STRING_SPLIT('a,b,c', ',', 1);
```

### 6.4 [Hard] How would you index and query a JSON column efficiently?

**Answer**
There's no native JSON index. Expose the hot property via a computed column (`JSON_VALUE`) and index it, and query with the *same expression* or the column. For array lookups, normalize into a child table or use OPENJSON in a join (scan-heavy). Consider relational columns for frequently queried data.

**Explanation**
JSON_VALUE is deterministic so the computed column can be persisted/indexed. A `CHECK (ISJSON(doc)=1)` constraint guarantees valid JSON. JSON_VALUE returns max 4000 chars; for longer use OPENJSON WITH.

**Code**
```sql
CREATE TABLE dbo.Docs (Id INT IDENTITY PRIMARY KEY, Doc NVARCHAR(MAX) NOT NULL CHECK (ISJSON(Doc)=1));
ALTER TABLE dbo.Docs ADD City AS JSON_VALUE(Doc, '$.address.city');
CREATE INDEX IX_Docs_City ON dbo.Docs(City);

INSERT dbo.Docs(Doc) VALUES (N'{"address":{"city":"Pune"}}'),(N'{"address":{"city":"Delhi"}}');
SELECT Id FROM dbo.Docs WHERE City = N'Pune';   -- Index Seek on IX_Docs_City → Id 1
```

**Gotchas / follow-ups**
- `JSON_VALUE` in lax mode returns NULL silently on a bad path; use `strict` to surface typos.
- Large `FOR JSON` output is streamed to clients as multiple ~2033-char rows; `ExecuteScalar` returns only the first chunk, so read with `SqlDataReader` and append all rows (or assign the JSON to a variable/`NVARCHAR(MAX)` column server-side).
- Escaping: JSON_VALUE returns unescaped text; JSON_QUERY returns raw JSON.

### 6.5 [Tricky] Gotchas: FOR JSON with NULLs, nested results, XML .nodes()/.value(), and WITHOUT_ARRAY_WRAPPER.

**Answer**
FOR JSON omits NULL properties unless `INCLUDE_NULL_VALUES`. A correlated subquery that itself ends in `FOR JSON PATH` nests as real JSON; but JSON held as plain text (a variable, column, or function result) is escaped as a string unless wrapped in `JSON_QUERY()`. `WITHOUT_ARRAY_WRAPPER` returns a single object without `[]`. For XML, `.nodes()` shreds to rows and `.value('(path)[1]','type')` extracts scalars — the `[1]` singleton is mandatory.

**Explanation**
`JSON_QUERY(textColumn)` marks text as already-JSON so FOR JSON embeds it rather than quoting it.

**Code**
```sql
DECLARE @txt NVARCHAR(MAX) = N'[{"Name":"Gus"}]';
SELECT DeptName,
       (SELECT e.Name FROM dbo.Employees e WHERE e.DeptId = d.DeptId FOR JSON PATH) AS Emps,  -- nested array
       @txt AS Escaped,                    -- becomes a string: "[{\"Name\":\"Gus\"}]"
       JSON_QUERY(@txt) AS NotEscaped      -- nested array
FROM dbo.Departments d WHERE d.DeptId = 3
FOR JSON PATH;
-- [{"DeptName":"HR","Emps":[{"Name":"Gus"}],"Escaped":"[{\"Name\":\"Gus\"}]","NotEscaped":[{"Name":"Gus"}]}]

DECLARE @x XML = N'<o><i id="1">A</i><i id="2">B</i></o>';
SELECT n.value('@id','int') AS Id, n.value('(text())[1]','nvarchar(10)') AS Val
FROM @x.nodes('/o/i') t(n);       -- 1 A | 2 B
```

**Interview traps**
- No JSON type (pre-2025): it's NVARCHAR; JSON_VALUE ≠ JSON_QUERY for arrays/objects.
- STRING_AGG overflow at 8000 bytes unless cast to MAX.
- STRING_SPLIT order not guaranteed before 2022; returns `value` column only.
- Lax-mode JSON paths hide typos.

**Cheat sheet**
- JSON_VALUE scalar, JSON_QUERY object/array, OPENJSON rows, FOR JSON PATH output.
- Index JSON via computed column.
- STRING_AGG(x, sep) WITHIN GROUP (ORDER BY ..); cast to NVARCHAR(MAX).
- STRING_SPLIT → `value` (+`ordinal` 2022).
- XML: `.nodes()` + `.value('(x)[1]', type)`.

---

## Topic 7: Views (indexed, updatable, SCHEMABINDING)

**Study notes**

A view is a stored SELECT (no data stored, except **indexed views**). Uses: abstraction, security (expose columns/rows), compatibility layer. Views are expanded inline into the outer query by the optimizer.

- **Updatable view**: simple view over one base table (no DISTINCT/GROUP BY/TOP/UNION/aggregates) supports INSERT/UPDATE/DELETE; multi-table views only if the change affects one base table; `WITH CHECK OPTION` rejects modifications that would make rows vanish from the view. Otherwise use `INSTEAD OF` triggers.
- **WITH SCHEMABINDING**: view locks referenced tables/columns (can't drop/alter them); requires two-part names, no `SELECT *`.
- **Indexed (materialized) view**: unique clustered index on a schema-bound view persists results. Requirements: SCHEMABINDING, deterministic expressions, `COUNT_BIG(*)` when GROUP BY, no outer joins, no DISTINCT/TOP/UNION/subqueries, specific SET options (`ANSI_NULLS`, `QUOTED_IDENTIFIER` ON, etc.). Enterprise edition auto-matches queries to the indexed view; other editions need `WITH (NOEXPAND)`. Cost: DML on base tables must maintain the view.
- `ORDER BY` in a view is not allowed without TOP/OFFSET and never guarantees output order.
- `SELECT *` in a view is frozen at creation; schema changes require `sp_refreshview`.

### 7.1 [Easy] What is a view and what are its benefits?

**Answer**
A view is a named, stored query exposed like a table. It simplifies complex joins, restricts access to columns/rows, and provides a stable interface over changing schemas. It does not store data (unless indexed).

**Explanation**
Permissions: granting SELECT on the view doesn't require access to the base table (ownership chaining, same owner).

**Code**
```sql
CREATE OR ALTER VIEW dbo.vEmployeePublic AS
SELECT EmployeeId, Name, DeptId FROM dbo.Employees;   -- hides Salary
GO
SELECT * FROM dbo.vEmployeePublic WHERE DeptId = 2;   -- Eli, Fay
```

### 7.2 [Follow-up] When is a view updatable, and what does WITH CHECK OPTION do?

**Answer**
A view over a single table without aggregates/DISTINCT/TOP/GROUP BY is updatable. `WITH CHECK OPTION` ensures rows inserted/updated through the view still satisfy the view's WHERE.

**Explanation**
Without it you can insert a row via the view that you then can't see.

**Code**
```sql
CREATE OR ALTER VIEW dbo.vSales AS
SELECT EmployeeId, Name, DeptId, Salary, HireDate FROM dbo.Employees WHERE DeptId = 2
WITH CHECK OPTION;
GO
UPDATE dbo.vSales SET Salary = 5500 WHERE EmployeeId = 5;      -- OK
UPDATE dbo.vSales SET DeptId = 1 WHERE EmployeeId = 5;         -- Msg 550: attempted insert/update failed (CHECK OPTION)
```

### 7.3 [Medium] What does SCHEMABINDING do and why does it matter?

**Answer**
It binds the view (or function) to the schema of the referenced objects: referenced tables/columns can't be dropped or altered in a way that breaks the view. It's required for indexed views and for deterministic/schema-bound UDFs (needed e.g. for indexing a computed column using a UDF).

**Explanation**
Requires two-part names and explicit columns (no `*`). Trade-off: migrations must drop/recreate the view first.

**Code**
```sql
CREATE OR ALTER VIEW dbo.vOrderCount WITH SCHEMABINDING AS
SELECT CustomerId, COUNT_BIG(*) AS Cnt, SUM(Amount) AS Total   -- Amount is NOT NULL, required for SUM in indexed view
FROM dbo.Orders GROUP BY CustomerId;
GO
ALTER TABLE dbo.Orders DROP COLUMN Amount;   -- Fails: object 'vOrderCount' is dependent on column 'Amount'
```

### 7.4 [Hard] Create an indexed view; what are the restrictions and the cost?

**Answer**
Create a SCHEMABINDING view with `COUNT_BIG(*)` (for GROUP BY) and then a unique clustered index on it. The optimizer (Enterprise, or `NOEXPAND` elsewhere) can answer aggregate queries from the stored result. Cost: extra storage and every INSERT/UPDATE/DELETE on base tables also updates the view, increasing write latency and potential blocking.

**Explanation**
Restrictions: no OUTER JOIN, subquery, DISTINCT, TOP, UNION, non-deterministic functions, `SUM` over a nullable expression (wrap in `ISNULL` or use a NOT NULL column), no `MIN/MAX`, no `AVG` (compute from SUM/COUNT_BIG). Good fit: read-heavy aggregate dashboards over rarely-changing detail.

**Code**
```sql
CREATE UNIQUE CLUSTERED INDEX UCI_vOrderCount ON dbo.vOrderCount(CustomerId);
-- Query base table; optimizer may use the view (Enterprise) or force with NOEXPAND:
SELECT CustomerId, Total FROM dbo.vOrderCount WITH (NOEXPAND) WHERE CustomerId = 10;   -- 450.00
```

**Gotchas / follow-ups**
- SET options must match when the session modifies base tables (ARITHABORT/ANSI settings); a connection with wrong options gets error 1934 on DML.
- Updates on one hot customer row serialize through the view's row → contention.

### 7.5 [Tricky] Gotchas: SELECT * in views, ORDER BY in views, and view stacking.

**Answer**
`SELECT *` in a view is expanded at creation; adding a base column doesn't appear (and dropped/renamed columns cause wrong data/errors) until `sp_refreshview`. ORDER BY in a view is illegal without TOP/OFFSET and doesn't guarantee order even then. Nested views hide complexity and can produce giant plans.

**Explanation**
Always order in the outermost query. Prefer explicit column lists and SCHEMABINDING.

**Code**
```sql
CREATE VIEW dbo.vBad AS SELECT * FROM dbo.Departments;
GO
ALTER TABLE dbo.Departments ADD Location NVARCHAR(50) NULL;
SELECT * FROM dbo.vBad;               -- Location NOT shown (stale metadata)
EXEC sp_refreshview 'dbo.vBad';       -- now shown
-- CREATE VIEW v AS SELECT Name FROM dbo.Employees ORDER BY Name;  -- Msg 1033
```

**Interview traps**
- "A view stores data" — only if indexed.
- Assuming ORDER BY inside a view orders results.
- Indexed view requires COUNT_BIG(*) with GROUP BY and no nullable SUM.
- WITH CHECK OPTION vs SCHEMABINDING confusion.

**Cheat sheet**
- View = inlined stored SELECT.
- Updatable if single-table, simple; add WITH CHECK OPTION.
- SCHEMABINDING: two-part names, no `*`, blocks breaking DDL.
- Indexed view: schemabound + unique clustered index; NOEXPAND on non-Enterprise.
- Never rely on view ORDER BY.

---

## Topic 8: Stored procedures

**Study notes**

A stored procedure is a named, compiled batch stored in the database. Features: input/`OUTPUT` parameters, default values, multiple result sets, `RETURN <int>` status code (0 = success by convention; **not** for data), `SET NOCOUNT ON`, transactions, error handling, permissions via `EXECUTE` grant (ownership chaining lets callers use the proc without table rights).

```
App --EXEC dbo.usp_X @p=1--> plan cache lookup --hit--> reuse plan
                                    |miss
                                    v
                            compile (parameter SNIFFING: plan built for first @p value)
```

- **Plan caching**: the plan is compiled on first execution and cached, sniffing the parameter values. Good for repeated calls; bad when data skew makes the sniffed plan unsuitable (*parameter sniffing problem*). Mitigations: `OPTION (RECOMPILE)`, `OPTIMIZE FOR`, `OPTIMIZE FOR UNKNOWN`, local variable copy, `WITH RECOMPILE`, split procs, Query Store plan forcing, 2022 PSP optimization.
- `SET NOCOUNT ON` suppresses "n rows affected" messages (DONE_IN_PROC), reducing network chatter and avoiding issues with clients that treat them as results.
- Procs vs ad hoc SQL: parameterized ad hoc SQL (sp_executesql / ORM) also gets cached plans; a proc's advantages are encapsulation, permissions, fewer round-trips, centralized tuning without redeploying the app. Disadvantages: logic split across tiers, harder versioning/testing, no portability.
- Procs vs ORM (EF Core): EF is productive for CRUD/maintainable LINQ; procs suit complex set-based logic, bulk operations, reports. They coexist (`FromSqlRaw`, `ExecuteSqlRaw`).
- Name procs without the `sp_` prefix (SQL Server looks in master first).

### 8.1 [Easy] Create a stored procedure with parameters, defaults, an OUTPUT parameter and a return code.

**Answer**
Declare parameters after the name; mark outputs with `OUTPUT`; `RETURN n` gives an integer status. The caller must also specify OUTPUT when executing.

**Explanation**
Return values can only be integers; use OUTPUT parameters or result sets to return data.

**Code**
```sql
CREATE OR ALTER PROCEDURE dbo.usp_GetDeptStats
    @DeptId INT,
    @MinSalary DECIMAL(10,2) = 0,
    @Headcount INT OUTPUT
AS
BEGIN
    SET NOCOUNT ON;
    IF NOT EXISTS (SELECT 1 FROM dbo.Departments WHERE DeptId = @DeptId)
        RETURN -1;                                   -- status code

    SELECT @Headcount = COUNT(*) FROM dbo.Employees
    WHERE DeptId = @DeptId AND Salary >= @MinSalary;

    SELECT Name, Salary FROM dbo.Employees WHERE DeptId = @DeptId AND Salary >= @MinSalary;
    RETURN 0;
END
GO
DECLARE @cnt INT, @rc INT;
EXEC @rc = dbo.usp_GetDeptStats @DeptId = 1, @MinSalary = 6500, @Headcount = @cnt OUTPUT;
SELECT @rc AS ReturnCode, @cnt AS Headcount;   -- 0, 2  (result set earlier: Asha 9000, Bram 7000)
```

### 8.2 [Follow-up] What does SET NOCOUNT ON do, and why do people always add it?

**Answer**
It stops SQL Server from sending the "(n rows affected)" informational message after each statement. That reduces network traffic and avoids confusing clients (e.g. older ADO code stopping at the first count "result"). It doesn't affect `@@ROWCOUNT`.

**Explanation**
In ADO.NET `ExecuteNonQuery` returns the rows-affected total of DML statements that report counts; with `SET NOCOUNT ON` inside a proc it returns -1. So don't rely on `ExecuteNonQuery`'s return for a procedure that sets NOCOUNT; return `@@ROWCOUNT` via OUTPUT or a result set instead.

**Code**
```sql
SET NOCOUNT ON;
UPDATE dbo.Employees SET Salary = Salary WHERE DeptId = 1;
SELECT @@ROWCOUNT AS Affected;   -- 4, even though the message was suppressed
```

### 8.3 [Medium] Explain plan caching and parameter sniffing in a procedure, and how you'd fix a bad sniffed plan.

**Answer**
The first execution compiles a plan using the parameter values passed (sniffing) and caches it; subsequent calls reuse it. If the first value was atypical (e.g. a rare customer with 1 row vs. a customer with 1M rows) later calls can get a very poor plan. Fixes: `OPTION (RECOMPILE)` on the statement, `OPTIMIZE FOR (@p UNKNOWN)`, restructure/split logic, update stats, or force a good plan via Query Store.

**Explanation**
Trade-offs: RECOMPILE costs CPU on each execution but gives best plan; OPTIMIZE FOR UNKNOWN uses average density (mediocre but stable). Diagnose by comparing "Parameter Compiled Value" vs "Runtime Value" in the actual plan XML.

**Code**
```sql
CREATE OR ALTER PROCEDURE dbo.usp_OrdersByCustomer @CustomerId INT AS
BEGIN
    SET NOCOUNT ON;
    SELECT OrderId, OrderDate, Amount
    FROM dbo.Orders
    WHERE CustomerId = @CustomerId
    OPTION (RECOMPILE);          -- or OPTION (OPTIMIZE FOR UNKNOWN)
END
```

### 8.4 [Hard] Stored procedures vs inline parameterized SQL vs an ORM — how do you decide, and do procs really perform better?

**Answer**
Performance is largely equal today: both procs and parameterized queries get compiled and cached plans. Choose procs for complex set-based/batch logic, security boundaries, and centralized tuning; choose ORM (EF Core) for CRUD and maintainability; mix them via `FromSql`/`ExecuteSql`. The real danger is non-parameterized concatenated SQL.

**Explanation**
Ad hoc, non-parameterized text creates a plan per literal (plan cache bloat; "optimize for ad hoc workloads" and forced parameterization can mitigate). EF Core generates parameterized `sp_executesql` calls. Procs make deployment of DB-only fixes easy but spread logic across layers and are harder to unit test/version.

**Code**
```csharp
// EF Core calling a proc (parameterized; safe)
var orders = await db.Orders
    .FromSqlInterpolated($"EXEC dbo.usp_OrdersByCustomer @CustomerId = {customerId}")
    .ToListAsync();   // FromSql with EXEC can't be composed with further LINQ operators (Where/OrderBy) on the server
```

**Gotchas / follow-ups**
- EXEC-based FromSql results can't be composed; SELECT-based (view/inline TVF) can.
- Check `sys.dm_exec_cached_plans` / Query Store for plan variation.
- Two copies of a plan can exist with different SET options (e.g. `ARITHABORT` ON in SSMS vs OFF in app) — the classic "fast in SSMS, slow in app" symptom.

### 8.5 [Tricky] Gotchas: deferred name resolution, RETURN in TRY/CATCH, multiple result sets, and CREATE vs ALTER.

**Answer**
Procs are created even when referenced tables don't exist yet (deferred name resolution) — but a missing *column* of an existing table fails at compile. `RETURN` can't return non-integers. `CREATE OR ALTER` (2016 SP1+) preserves permissions, unlike DROP/CREATE. Procs can return multiple result sets (`SqlDataReader.NextResult()`).

**Explanation**
Temp tables created inside a proc are dropped when it ends; a temp table created by the caller is visible to callees. `EXEC` with a string/`sp_executesql` runs in a child scope (variables from caller aren't visible).

**Code**
```sql
CREATE OR ALTER PROCEDURE dbo.usp_Deferred AS SELECT * FROM dbo.NoSuchTable;  -- succeeds (warning-free)
GO
EXEC dbo.usp_Deferred;   -- Msg 208 Invalid object name 'dbo.NoSuchTable' at runtime
```

**Interview traps**
- RETURN is for status codes, not data.
- "Procs are precompiled so always faster" — outdated.
- Forgetting OUTPUT keyword at the call site.
- Parameter sniffing blamed on "bad indexes".
- `sp_` prefix.

**Cheat sheet**
- Params, defaults, OUTPUT, RETURN int, multiple result sets.
- SET NOCOUNT ON for chatter; `@@ROWCOUNT` unaffected.
- Plan cached per proc; sniffing → RECOMPILE / OPTIMIZE FOR / Query Store.
- Parameterized SQL ≈ proc performance.
- `CREATE OR ALTER` keeps permissions.

---

## Topic 9: User-defined functions

**Study notes**

| Type | Syntax | Optimizer treatment |
|---|---|---|
| Scalar UDF | `RETURNS int AS BEGIN ... RETURN x END` | Historically a black box, invoked **row by row**, blocks parallelism, bad estimates |
| Inline TVF (iTVF) | `RETURNS TABLE AS RETURN (SELECT ...)` | Expanded into the outer query like a parameterized view — best performance |
| Multi-statement TVF (MSTVF) | `RETURNS @t TABLE(...) AS BEGIN ... RETURN END` | Opaque; fixed row estimate (1 before 2014, 100 in 2014/2016+; improved by *interleaved execution* in 2017+ and deferred table-variable compilation in 2019+) |

**Scalar UDF Inlining (SQL Server 2019, compat level 150)** automatically transforms eligible scalar UDFs into relational expressions inside the calling query, removing per-row call overhead. Not all UDFs are inlineable: the body must essentially be DECLARE/SET/SELECT/IF-ELSE/RETURN (no `WHILE` loops), must not use table variables or time-dependent/side-effecting intrinsics (e.g. `GETDATE()`, `NEWSEQUENTIALID()`), and must not use `EXECUTE AS` other than CALLER; check `sys.sql_modules.is_inlineable`. Functions can't modify database state (no DML on permanent tables, no TRY/CATCH, no dynamic SQL, no temp tables). Use `WITH SCHEMABINDING` for deterministic functions to allow use in indexed computed columns.

### 9.1 [Easy] Scalar UDF vs inline TVF — what's the difference?

**Answer**
A scalar UDF returns a single value and is called per row; an inline TVF returns a table and is just a parameterized single SELECT. Both are reusable, but iTVFs are generally much faster because the optimizer inlines them.

**Explanation**
Use scalar UDFs for small, genuinely scalar logic; prefer iTVF + `CROSS APPLY` (or the expression inline) for set-based logic.

**Code**
```sql
CREATE OR ALTER FUNCTION dbo.fn_Tax (@amount DECIMAL(10,2)) RETURNS DECIMAL(10,2)
AS BEGIN RETURN @amount * 0.18; END;
GO
CREATE OR ALTER FUNCTION dbo.itvf_Tax (@amount DECIMAL(10,2)) RETURNS TABLE
AS RETURN (SELECT @amount * 0.18 AS Tax);
GO
SELECT o.OrderId, dbo.fn_Tax(o.Amount) AS T1, t.Tax AS T2
FROM dbo.Orders o CROSS APPLY dbo.itvf_Tax(o.Amount) t;   -- both 18.00 for Amount=100
```

### 9.2 [Follow-up] Why are scalar UDFs "bad" for performance (pre-2019)?

**Answer**
They execute once per row like an interpreted call, force serial plans, hide their cost from the optimizer (so estimates are poor), and `SET STATISTICS` doesn't show their per-call IO. On large tables this can turn seconds into minutes.

**Explanation**
Fixes: rewrite as an inline TVF / inline expression, or upgrade to SQL Server 2019 compat 150 where inlining applies, or compute and persist.

**Code**
```sql
-- Check whether SQL Server 2019 can inline it:
SELECT OBJECT_NAME(object_id) AS fn, is_inlineable
FROM sys.sql_modules WHERE object_id = OBJECT_ID('dbo.fn_Tax');
-- Disable inlining for one function if it misbehaves:
-- ALTER FUNCTION ... WITH INLINE = OFF
```

### 9.3 [Medium] MSTVF vs iTVF — explain the estimates and when MSTVF is acceptable.

**Answer**
An iTVF is merged into the plan and gets real statistics. An MSTVF fills a table variable and the optimizer sees a fixed estimate (100 rows in 2014+, 1 before; 2017 interleaved execution fixes it for MSTVF specifically), so joins pick bad plans and it can't use indexes on the result. MSTVF is fine for small, complex logic that cannot be a single SELECT.

**Explanation**
You can often turn an MSTVF into an iTVF using CTEs/CASE/UNION ALL. For genuinely procedural logic use a stored proc with temp tables instead.

**Code**
```sql
CREATE OR ALTER FUNCTION dbo.mstvf_DeptEmps (@DeptId INT)
RETURNS @r TABLE (EmployeeId INT, Name NVARCHAR(50))
AS BEGIN
    INSERT @r SELECT EmployeeId, Name FROM dbo.Employees WHERE DeptId = @DeptId;
    RETURN;
END;
GO
CREATE OR ALTER FUNCTION dbo.itvf_DeptEmps (@DeptId INT) RETURNS TABLE
AS RETURN SELECT EmployeeId, Name FROM dbo.Employees WHERE DeptId = @DeptId;
```

### 9.4 [Hard] How does Scalar UDF Inlining work and what blocks it?

**Answer**
In SQL Server 2019 with database compat level ≥ 150 the optimizer rewrites a qualifying scalar UDF body into relational operators (typically a `OUTER APPLY` of a derived table) so it participates in costing and parallelism. Blockers include `WHILE` loops, table variables, time-dependent or side-effecting intrinsics (`GETDATE()`, `NEWSEQUENTIALID()`), `EXECUTE AS` other than CALLER, and use of the UDF inside computed columns or CHECK constraints (not inlined there).

**Explanation**
Verify via `sys.sql_modules.is_inlineable` and the plan (the function body appears as relational operators rather than a per-row call). Inlining can increase compile time for very large UDFs. Microsoft documents the full requirements list — don't memorize it, know to check `is_inlineable`.

**Code**
```sql
ALTER DATABASE SCOPED CONFIGURATION SET TSQL_SCALAR_UDF_INLINING = ON;   -- default ON at compat 150
SELECT o.OrderId, dbo.fn_Tax(o.Amount) FROM dbo.Orders o;   -- plan: Compute Scalar expression, no per-row function calls
```

**Gotchas / follow-ups**
- Inlining doesn't apply when UDF is used in a computed column that is indexed/persisted, nor CHECK constraints.
- A UDF in a WHERE clause on a column makes the predicate non-sargable (`WHERE dbo.f(col) = x`).

### 9.5 [Tricky] Why does a UDF in a computed column or WHERE disable parallelism, and how do you tell determinism?

**Answer**
Before inlining (and when not inlineable) a scalar UDF forces a serial plan, and a UDF that accesses data (even a SELECT) is non-deterministic/imprecise, so it can't be persisted or indexed. Determinism is exposed by `OBJECTPROPERTY(id,'IsDeterministic')` and requires `SCHEMABINDING` without data access.

**Explanation**
Another gotcha: a scalar UDF that does a SELECT runs a query per row, but its I/O is hidden from `SET STATISTICS IO`. Use extended events / `sys.dm_exec_function_stats` (2016+).

**Code**
```sql
CREATE FUNCTION dbo.fn_Det(@x INT) RETURNS INT WITH SCHEMABINDING AS BEGIN RETURN @x * 2; END;
GO
SELECT OBJECTPROPERTY(OBJECT_ID('dbo.fn_Det'), 'IsDeterministic') AS IsDet;   -- 1
```

**Interview traps**
- "Functions are always reusable and cheap" — scalar UDFs are a classic perf trap.
- Functions can't have side effects (no INSERT into permanent tables).
- `WHERE dbo.fn(col)=...` is not sargable.
- MSTVF estimates = fixed 100 (2014+).

**Cheat sheet**
- iTVF ≫ MSTVF; scalar UDF row-by-row until 2019 inlining.
- Inlining: compat 150, check `is_inlineable`.
- SCHEMABINDING + no data access → deterministic.
- Prefer iTVF + APPLY.

---

## Topic 10: Triggers

**Study notes**

A trigger runs automatically on DML (or DDL/logon). **DML triggers** fire *once per statement* (not per row) and expose pseudo-tables `inserted` (new row images) and `deleted` (old row images); UPDATE = both. Types:

| Type | Fires | Notes |
|---|---|---|
| AFTER (FOR) | after the DML & constraints succeed, in the same transaction | multiple allowed per event; `ROLLBACK` in the trigger undoes the statement |
| INSTEAD OF | replaces the DML | one per event per table/view; used to make complex views updatable |
| DDL / logon | schema changes / logins | auditing |

Key facts: the trigger participates in the caller's transaction; an error raised inside a trigger rolls back the transaction and aborts the batch; handle multi-row operations via set-based logic; `UPDATE(col)` / `COLUMNS_UPDATED()` test which columns were targeted; `TRIGGER_NESTLEVEL()`, recursive triggers (off by default) and nested triggers (on) options; `@@ROWCOUNT` at top of trigger. Use `OUTPUT` clause or temporal tables/CDC/Change Tracking as alternatives for auditing.

### 10.1 [Easy] What are the `inserted` and `deleted` tables?

**Answer**
They're special in-memory pseudo-tables available inside DML triggers: `inserted` holds the new rows for INSERT/UPDATE, `deleted` the old rows for DELETE/UPDATE. They are scoped to the trigger's firing statement.

**Explanation**
For UPDATE, join `inserted` and `deleted` on the primary key to see before/after. They can hold many rows, so triggers must be written set-based.

**Code**
```sql
CREATE TABLE dbo.SalaryAudit (AuditId INT IDENTITY PRIMARY KEY, EmployeeId INT, OldSalary DECIMAL(10,2),
                              NewSalary DECIMAL(10,2), ChangedAt DATETIME2 DEFAULT SYSUTCDATETIME());
GO
CREATE OR ALTER TRIGGER dbo.trg_Emp_SalaryAudit ON dbo.Employees AFTER UPDATE AS
BEGIN
    SET NOCOUNT ON;
    IF NOT UPDATE(Salary) RETURN;
    INSERT dbo.SalaryAudit (EmployeeId, OldSalary, NewSalary)
    SELECT i.EmployeeId, d.Salary, i.Salary
    FROM inserted i JOIN deleted d ON d.EmployeeId = i.EmployeeId
    WHERE i.Salary <> d.Salary;
END
GO
UPDATE dbo.Employees SET Salary = Salary + 500 WHERE DeptId = 3;   -- Gus
SELECT EmployeeId, OldSalary, NewSalary FROM dbo.SalaryAudit;      -- 7 | 3500 | 4000
```

### 10.2 [Follow-up] AFTER vs INSTEAD OF triggers — when to use each?

**Answer**
AFTER triggers run after the data change and constraints pass; use for auditing or cascading actions and `ROLLBACK` to veto. INSTEAD OF triggers replace the operation; use them to make a multi-table view updatable, to implement soft-delete, or to custom-route writes.

**Explanation**
Only INSTEAD OF triggers can be defined on views. A table can have only one INSTEAD OF trigger per operation. INSTEAD OF fires before constraint checks, so you must perform the actual change yourself.

**Code**
```sql
CREATE OR ALTER TRIGGER dbo.trg_Orders_SoftDelete ON dbo.Orders INSTEAD OF DELETE AS
BEGIN
    SET NOCOUNT ON;
    UPDATE o SET Status = 'Deleted' FROM dbo.Orders o JOIN deleted d ON d.OrderId = o.OrderId;
END
-- Note: Status is VARCHAR(10); 'Deleted' fits. DELETE FROM dbo.Orders WHERE OrderId = 1 → row stays, Status='Deleted'
```

### 10.3 [Medium] What are the main pitfalls of triggers?

**Answer**
(1) Assuming one row per firing — they fire once per statement; (2) hidden logic and performance cost inside every DML; (3) lock duration extends the caller's transaction; (4) recursion/nesting chains; (5) errors roll back the entire statement/transaction; (6) `@@IDENTITY` returning an identity generated by a trigger's insert into another table (use `SCOPE_IDENTITY()`); (7) enabled triggers block a plain `OUTPUT` clause (Msg 334), which affects EF Core; (8) `TRUNCATE` and bulk loads without `FIRE_TRIGGERS` don't fire them.

**Explanation**
Scalar-variable pattern `SELECT @x = col FROM inserted` handles only one row and is a classic bug. Prefer constraints, computed columns, temporal tables for audit, and keep triggers short.

**Code**
```sql
-- BAD (single-row assumption): 
-- DECLARE @id INT = (SELECT EmployeeId FROM inserted);   -- error/undefined on multi-row insert
-- GOOD: INSERT ... SELECT ... FROM inserted;
SELECT @@IDENTITY AS TrigPolluted, SCOPE_IDENTITY() AS Mine;  -- @@IDENTITY may return an identity from a trigger's insert into another table
```

### 10.4 [Hard] Triggers and EF Core / OUTPUT clause, and transaction behaviour.

**Answer**
EF Core uses `OUTPUT INSERTED.<col>` to read generated keys; SQL Server doesn't allow a plain OUTPUT (without INTO) on a table that has enabled triggers (Msg 334), so EF must be told the table has triggers (`ToTable(t => t.HasTrigger("..."))`) so it uses `OUTPUT INTO` a temp table. Triggers run inside the statement's transaction, so a failure rolls back the DML.

**Explanation**
Modern EF Core (7+) configures this automatically when `HasTrigger` is specified. A trigger that issues `ROLLBACK TRANSACTION` ends the transaction; the calling batch then gets error 3609 ("transaction ended in the trigger") — avoid ROLLBACK, use `THROW`.

**Code**
```sql
CREATE OR ALTER TRIGGER dbo.trg_Orders_Validate ON dbo.Orders AFTER INSERT AS
BEGIN
    SET NOCOUNT ON;
    IF EXISTS (SELECT 1 FROM inserted WHERE Amount <= 0)
        THROW 50001, 'Amount must be positive', 1;     -- aborts statement and the transaction
END
```
```csharp
modelBuilder.Entity<Order>().ToTable(tb => tb.HasTrigger("trg_Orders_Validate"));
```

**Gotchas / follow-ups**
- A CHECK constraint is better than a validation trigger.
- `OUTPUT` without INTO fails when triggers exist → Msg 334.

### 10.5 [Tricky] Which operations do NOT fire triggers? What is TRIGGER_NESTLEVEL and recursion?

**Answer**
`TRUNCATE TABLE`, `WRITETEXT`, and bulk loads without `FIRE_TRIGGERS` don't fire DML triggers. Nested triggers (a trigger's DML fires another) are on by default up to 32 levels; direct recursion (a trigger firing itself) is off by default (`ALTER DATABASE ... SET RECURSIVE_TRIGGERS`).

**Explanation**
Indirect recursion (A→B→A) happens when nested triggers is ON regardless of RECURSIVE_TRIGGERS. Protect with `IF TRIGGER_NESTLEVEL() > 1 RETURN;` or a guard column.

**Code**
```sql
BULK INSERT dbo.Orders FROM 'C:\data\o.csv' WITH (FIRE_TRIGGERS);  -- triggers fire only with this option
```

**Interview traps**
- Per-statement, not per-row.
- ROLLBACK inside a trigger → error 3609 follow-ups.
- `@@IDENTITY` vs `SCOPE_IDENTITY()`.
- Any enabled trigger blocks a plain OUTPUT clause (Msg 334) — EF needs `HasTrigger`.
- TRUNCATE doesn't fire.

**Cheat sheet**
- `inserted`/`deleted`, set-based always.
- AFTER = post-constraint; INSTEAD OF = replaces, works on views.
- Use `UPDATE(col)` guard.
- Prefer constraints/temporal tables over triggers.
- EF: `HasTrigger`.

---

## Topic 11: Error handling & transactions

**Study notes**

```
BEGIN TRY
    BEGIN TRAN; ... COMMIT;
END TRY
BEGIN CATCH
    IF @@TRANCOUNT > 0 ROLLBACK;
    THROW;            -- rethrow original error (needs preceding statement ended with ';')
END CATCH
```

- **TRY/CATCH** catches errors with severity 11–19 raised at run time. It does *not* catch: severity ≥ 20 that drops the connection, compile errors/deferred name resolution errors in the *same* batch scope, and messages with severity ≤ 10 (warnings). Inside CATCH use `ERROR_NUMBER()`, `ERROR_MESSAGE()`, `ERROR_SEVERITY()`, `ERROR_STATE()`, `ERROR_LINE()`, `ERROR_PROCEDURE()`, `XACT_STATE()`.
- **THROW** (2012+): rethrows original error when used without arguments in CATCH; with args `THROW 50001,'msg',1` (error number ≥ 50000, severity always 16). Statement before must end with `;`. **RAISERROR**: older, supports printf-style formatting, custom severity, `WITH NOWAIT`, `WITH LOG`; doesn't abort the batch by itself and isn't a rethrow of the original.
- **XACT_ABORT ON**: any run-time error rolls back the whole transaction and aborts the batch. Recommended in procs with transactions (also required for distributed transactions and makes timeouts/attention clean up). With XACT_ABORT ON, errors enter CATCH with the transaction **uncommittable** (`XACT_STATE() = -1`) — you must ROLLBACK.
- `@@ERROR` holds the error number of the *last statement* only (reset by every statement) — legacy; capture immediately.
- **Nested transactions are a myth**: `BEGIN TRAN` increments `@@TRANCOUNT`; inner `COMMIT` only decrements; only the outermost COMMIT commits; any `ROLLBACK` rolls back everything and sets `@@TRANCOUNT` to 0. Use **savepoints** (`SAVE TRAN name`; `ROLLBACK TRAN name`) for partial rollback.

### 11.1 [Easy] How do you handle errors in T-SQL?

**Answer**
Wrap code in `BEGIN TRY ... END TRY BEGIN CATCH ... END CATCH`, inspect `ERROR_*()` functions, roll back any open transaction, and rethrow with `THROW` so the caller knows it failed.

**Explanation**
Swallowing an error silently (no THROW) makes the caller think it succeeded — one of the most common bugs.

**Code**
```sql
BEGIN TRY
    SELECT 1 / 0;
END TRY
BEGIN CATCH
    SELECT ERROR_NUMBER() AS Num, ERROR_MESSAGE() AS Msg, ERROR_LINE() AS Ln;
    -- 8134 | Divide by zero error encountered.
END CATCH
```

### 11.2 [Follow-up] THROW vs RAISERROR — differences?

**Answer**
THROW is the modern approach: it can rethrow the original error (no args), always uses severity 16 when given arguments, requires error number ≥ 50000 and terminates the batch. RAISERROR supports message formatting/`sys.messages` ids, arbitrary severity, and `WITH NOWAIT`, but doesn't rethrow the original error number and (without XACT_ABORT) continues executing.

**Explanation**
Use RAISERROR mainly for progress messages (`RAISERROR('step %d',0,1,@i) WITH NOWAIT`). Use `THROW` for errors. Statement before THROW needs a terminating `;` — otherwise THROW is parsed as part of the previous statement (e.g. as a transaction name/alias).

**Code**
```sql
BEGIN TRY
    THROW 50001, 'Custom failure', 1;
END TRY
BEGIN CATCH
    PRINT CONCAT(ERROR_NUMBER(), ': ', ERROR_MESSAGE());   -- 50001: Custom failure
    THROW;   -- rethrow original: Msg 50001, Level 16, State 1
END CATCH
```

### 11.3 [Medium] Write the standard transactional stored procedure template with XACT_ABORT.

**Answer**
`SET XACT_ABORT, NOCOUNT ON;` BEGIN TRY → BEGIN TRAN → work → COMMIT; CATCH → `IF @@TRANCOUNT > 0 ROLLBACK; THROW;`.

**Explanation**
XACT_ABORT ensures an open transaction is rolled back on errors that CATCH wouldn't see (e.g. client timeout/attention). For nested calls where the proc might already be in a transaction, use `XACT_STATE()` or a savepoint pattern.

**Code**
```sql
CREATE OR ALTER PROCEDURE dbo.usp_TransferBonus @FromId INT, @ToId INT, @Amount DECIMAL(10,2)
AS
BEGIN
    SET NOCOUNT, XACT_ABORT ON;
    BEGIN TRY
        BEGIN TRAN;
            UPDATE dbo.Employees SET Salary = Salary - @Amount WHERE EmployeeId = @FromId;
            UPDATE dbo.Employees SET Salary = Salary + @Amount WHERE EmployeeId = @ToId;
            IF EXISTS (SELECT 1 FROM dbo.Employees WHERE Salary < 0) THROW 50002, 'Negative salary', 1;
        COMMIT;
    END TRY
    BEGIN CATCH
        IF @@TRANCOUNT > 0 ROLLBACK;
        THROW;
    END CATCH
END
```

### 11.4 [Hard] Explain @@TRANCOUNT, nested transactions and savepoints — what really happens on inner COMMIT/ROLLBACK?

**Answer**
BEGIN TRAN increments @@TRANCOUNT, COMMIT decrements it, and only the outermost COMMIT (count 1→0) makes changes durable. ROLLBACK (without savepoint) rolls back the whole transaction to the outermost BEGIN and sets the count to 0, regardless of nesting. For partial rollback use `SAVE TRAN sp` and `ROLLBACK TRAN sp`.

**Explanation**
Consequence: a proc that does BEGIN/ROLLBACK when called inside an outer transaction destroys the caller's transaction (error 266, "transaction count mismatch" on exit). Pattern: only start a transaction if none exists (`@own = IIF(@@TRANCOUNT=0,1,0)`), else use a savepoint. EF Core's `TransactionScope`/`SaveChanges` have analogous nuances.

**Code**
```sql
BEGIN TRAN;                         -- @@TRANCOUNT = 1
  BEGIN TRAN;                       -- 2
  UPDATE dbo.Employees SET Salary = 1 WHERE EmployeeId = 1;
  COMMIT;                           -- 1  (NOT committed yet!)
SELECT @@TRANCOUNT AS Cnt;          -- 1
ROLLBACK;                           -- everything undone, @@TRANCOUNT = 0
SELECT Salary FROM dbo.Employees WHERE EmployeeId = 1;   -- 9000 (unchanged)

BEGIN TRAN;
  UPDATE dbo.Employees SET Salary = 1 WHERE EmployeeId = 2;
  SAVE TRAN s1;
  UPDATE dbo.Employees SET Salary = 1 WHERE EmployeeId = 3;
  ROLLBACK TRAN s1;                 -- undoes only Chen's update
SELECT EmployeeId, Salary FROM dbo.Employees WHERE EmployeeId IN (2,3);   -- Bram 1, Chen 6000
ROLLBACK;                           -- discard the demo (use COMMIT to keep Bram's change)
```

**Gotchas / follow-ups**
- `@@TRANCOUNT` counts nesting; named transactions' names are ignored for nested rollback (only the outermost name can be rolled back).
- Locks are held until the outermost commit.

### 11.5 [Tricky] CATCH with XACT_STATE() = -1, errors TRY/CATCH can't catch, and @@ERROR traps.

**Answer**
When a doomed (uncommittable) transaction occurs (XACT_ABORT ON, or certain errors), `XACT_STATE() = -1`: you can only ROLLBACK, any COMMIT or write fails. TRY/CATCH can't catch compile errors in the same scope, severity ≥ 20, killed sessions, or attention (client cancel/timeout). `@@ERROR` is reset by the next statement, so `IF @@ERROR <> 0` after an intervening statement tests the wrong thing.

**Explanation**
Catching "object doesn't exist" errors: deferred name resolution means a missing table in a nested proc/`EXEC` *is* caught, but a missing table referenced in the same batch fails at compile before TRY starts. Client-side: .NET gets a `SqlException`; `Errors` collection and `Number`.

**Code**
```sql
SET XACT_ABORT ON;
BEGIN TRY
    BEGIN TRAN;
    INSERT dbo.Orders (CustomerId, OrderDate, Amount, Status) VALUES (1,'2024-01-01',NULL,'New');  -- NOT NULL violation
    COMMIT;
END TRY
BEGIN CATCH
    SELECT XACT_STATE() AS St;                 -- -1 : doomed
    IF XACT_STATE() <> 0 ROLLBACK;
    THROW;
END CATCH
GO
UPDATE dbo.Employees SET Salary = Salary WHERE EmployeeId = -1;
SELECT @@ERROR;     -- 0: zero rows affected is not an error; and any later statement resets @@ERROR anyway
```

**Interview traps**
- Inner COMMIT doesn't commit; any ROLLBACK is global.
- THROW needs the preceding `;`.
- Swallowing errors without rethrow.
- RAISERROR doesn't terminate batch / doesn't rethrow original.
- XACT_ABORT is OFF by default (except inside triggers where an error rolls back the transaction).

**Cheat sheet**
- TRY/CATCH + `IF @@TRANCOUNT>0 ROLLBACK; THROW;`.
- SET XACT_ABORT ON in transactional procs.
- Only outermost COMMIT commits; ROLLBACK is all-or-nothing; SAVE TRAN for partial.
- XACT_STATE: 1 committable, 0 none, -1 doomed.
- `@@ERROR` volatile.

---

## Topic 12: Dynamic SQL

**Study notes**

Dynamic SQL builds T-SQL text at run time and executes it with `EXEC(@sql)` (string only, no parameters, new plan per distinct text) or `sp_executesql @stmt, @params_definition, @p1 = ...` (parameterized, plan reusable, supports OUTPUT params).

Why/when: variable table/column names, dynamic pivots, optional-filter search ("catch-all") procedures, admin scripts. Dangers: **SQL injection** (concatenating user input), plan cache bloat, permission context (the caller needs rights on underlying objects — ownership chaining does not apply inside dynamic SQL), scope (dynamic batch can't see the caller's local variables or temp variables, but sees caller's `#temp` tables; `#temp` created inside is gone afterwards).

Defenses: **always parameterize values** with sp_executesql; for **identifiers** (table/column names) which can't be parameters, validate against a whitelist (`sys.objects`/`sys.columns`) and wrap with `QUOTENAME()`; use `NVARCHAR` for the string (sp_executesql requires Unicode); least-privilege accounts.

### 12.1 [Easy] EXEC vs sp_executesql — differences?

**Answer**
`EXEC(@sql)` runs a string with no parameter support; each distinct string compiles separately. `sp_executesql` takes a parameter definition and values, so the statement text stays constant and its plan is reused, and it can return OUTPUT values. It also protects against injection for values.

**Explanation**
The statement and parameter definition must be `NVARCHAR`.

**Code**
```sql
DECLARE @sql NVARCHAR(MAX) = N'SELECT COUNT(*) FROM dbo.Employees WHERE DeptId = @d',
        @cnt INT;
EXEC sp_executesql @sql, N'@d INT', @d = 1;     -- 4

SET @sql = N'SELECT @c = COUNT(*) FROM dbo.Employees WHERE DeptId = @d';
EXEC sp_executesql @sql, N'@d INT, @c INT OUTPUT', @d = 2, @c = @cnt OUTPUT;
SELECT @cnt;   -- 2
```

### 12.2 [Follow-up] Show a SQL injection example and the fix.

**Answer**
Concatenating input like `'; DROP TABLE ...--` into the query lets the attacker alter statement structure. Fix: pass the value as a parameter; for identifiers whitelist + `QUOTENAME`.

**Explanation**
Even `QUOTENAME` on values isn't a complete defense; parameters are. In .NET always use `SqlParameter`/Dapper/EF parameters — string interpolation into `FromSqlRaw` is dangerous (use `FromSqlInterpolated`).

**Code**
```sql
DECLARE @name NVARCHAR(50) = N'x''; DELETE dbo.Orders; --';
-- VULNERABLE:
DECLARE @bad NVARCHAR(MAX) = N'SELECT * FROM dbo.Employees WHERE Name = ''' + @name + N'''';
PRINT @bad;   -- SELECT * FROM dbo.Employees WHERE Name = 'x'; DELETE dbo.Orders; --'
-- SAFE:
EXEC sp_executesql N'SELECT * FROM dbo.Employees WHERE Name = @n', N'@n NVARCHAR(50)', @n = @name;  -- 0 rows, no delete
```

### 12.3 [Medium] Build a safe dynamic ORDER BY / table name search procedure.

**Answer**
Optional filters: use parameters in the dynamic text and append only the clauses needed. Identifiers: validate against a whitelist (here via `sys.columns`) and apply `QUOTENAME`.

**Explanation**
Alternative for catch-all search without dynamic SQL: `WHERE (@d IS NULL OR DeptId=@d) OPTION (RECOMPILE)`. Dynamic SQL gives each shape its own optimal plan and keeps `sp_executesql` caching.

**Code**
```sql
CREATE OR ALTER PROCEDURE dbo.usp_SearchEmployees
    @DeptId INT = NULL, @MinSalary DECIMAL(10,2) = NULL, @SortBy SYSNAME = N'Name'
AS
BEGIN
    SET NOCOUNT ON;
    IF NOT EXISTS (SELECT 1 FROM sys.columns WHERE object_id = OBJECT_ID('dbo.Employees') AND name = @SortBy)
        THROW 50010, 'Invalid sort column', 1;

    DECLARE @sql NVARCHAR(MAX) = N'SELECT EmployeeId, Name, Salary FROM dbo.Employees WHERE 1 = 1';
    IF @DeptId    IS NOT NULL SET @sql += N' AND DeptId = @DeptId';
    IF @MinSalary IS NOT NULL SET @sql += N' AND Salary >= @MinSalary';
    SET @sql += N' ORDER BY ' + QUOTENAME(@SortBy);

    EXEC sp_executesql @sql, N'@DeptId INT, @MinSalary DECIMAL(10,2)', @DeptId, @MinSalary;
END
```

### 12.4 [Hard] Scope, permissions and plan-cache issues of dynamic SQL.

**Answer**
Dynamic SQL runs in a child scope: it can't see the caller's local variables (pass them as parameters), but it sees caller `#temp` tables. Ownership chaining doesn't flow in, so the executing user needs direct permissions (or use `EXECUTE AS`/module signing). Using `EXEC(@sql)` with embedded literals creates one plan per literal → plan cache bloat; use sp_executesql or enable `optimize for ad hoc workloads`/forced parameterization.

**Explanation**
Also: `sp_executesql` with different parameter *types* creates separate plans; define the parameter types explicitly (EF/ADO.NET `SqlParameter.Size` for strings avoids cache fragmentation).

**Code**
```sql
CREATE TABLE #t (x INT); INSERT #t VALUES (1);
EXEC (N'SELECT * FROM #t');        -- works: sees caller's temp table
DECLARE @v INT = 5;
EXEC (N'SELECT @v');               -- Msg 137: Must declare the scalar variable "@v"
EXEC (N'CREATE TABLE #inner (y INT)');
SELECT * FROM #inner;              -- Msg 208: dropped when the dynamic batch ended
```

**Gotchas / follow-ups**
- `USE db` inside dynamic SQL doesn't persist after it ends.
- Concatenating NULL into the string yields NULL → `EXEC(NULL)` does nothing silently; use `CONCAT` or ISNULL.
- `NVARCHAR(MAX)` vs `NVARCHAR(4000)` truncation; prefix `N` on literals.

### 12.5 [Tricky] Is QUOTENAME enough? What about LIKE patterns and second-order injection?

**Answer**
`QUOTENAME` safely quotes *identifiers* (doubling the closing bracket); values must still be parameters. Second-order injection happens when stored data (e.g. a table name saved earlier) is later concatenated into dynamic SQL — it must be quoted/whitelisted at use time. LIKE wildcards in parameters (`%`, `_`) aren't an injection but can cause unexpected matches/perf — escape them.

**Explanation**
Never trust "it came from our own database". `QUOTENAME` has input-length limits (identifier names are max 128 chars; longer input can return NULL), another reason it is not a substitute for parameterizing values.

**Code**
```sql
SELECT QUOTENAME(N'Order]s');           -- [Order]]s]   (closing bracket doubled)
SELECT QUOTENAME(N'a''b', '''');        -- 'a''b'  (works, but prefer parameters for values)
SELECT QUOTENAME(NULL);                 -- NULL
```

**Interview traps**
- "Stored procedures prevent injection" — not if they build dynamic SQL by concatenation.
- Identifiers can't be parameters.
- `EXEC(@sql)` for values → plan bloat + injection.
- Dynamic SQL loses ownership chaining.
- NULL concatenation silently kills the statement.

**Cheat sheet**
- `sp_executesql @stmt NVARCHAR, @params, values` → parameterized & cached.
- Identifiers: whitelist + QUOTENAME.
- Child scope: sees #temp, not local variables.
- Caller needs direct object permissions.
- EF: `FromSqlInterpolated`, never concatenate into `FromSqlRaw`.

---

## Topic 13: Cursors vs set-based; temp tables vs table variables vs TVPs

**Study notes**

**Cursors** process rows one at a time (`DECLARE c CURSOR FOR ...; OPEN; FETCH NEXT ...; CLOSE; DEALLOCATE`). They're procedural, slow, resource-heavy (locks/tempdb), but occasionally justified: per-row side effects (call a proc per row), complex running logic impossible with windows (rare after 2012), admin scripts across databases. Prefer set-based SQL, window functions, `APPLY`, or `WHILE` over a key range. If needed: `LOCAL FAST_FORWARD` (forward-only, read-only) is the cheapest.

| | `#temp` table | `@table` variable | TVP (table-valued parameter) |
|---|---|---|---|
| Stored in | tempdb | tempdb (not memory only!) | tempdb-backed table variable |
| Statistics | Yes (auto + manual) | No column stats; fixed 1-row estimate pre-2019; **deferred compilation** in 2019+ gives real row count | Same as table variable (read-only in proc) |
| Indexes | Any, created after | Only inline PK/UNIQUE/INDEX in declaration | Primary key/unique inline in the table type |
| Transactions | Participates in rollback | **Not** affected by ROLLBACK (changes persist) | n/a (read-only) |
| Scope | session/proc (`##` global) | batch/proc | parameter passed to proc; requires `CREATE TYPE` |
| Recompiles | May trigger recompiles | Fewer | Fewer |

Use `#temp` for larger result sets/where stats matter; table variables for small (<~100s rows) sets; TVPs to pass a set of rows from .NET (`SqlDbType.Structured` / `DataTable`/`IEnumerable<SqlDataRecord>`) instead of CSV strings or XML.

### 13.1 [Easy] Why avoid cursors?

**Answer**
They process one row at a time with heavy overhead — locks, tempdb, context switching — whereas SQL Server's engine is optimized for set-based operations. Use set-based UPDATE/INSERT or window functions instead.

**Explanation**
A single `UPDATE ... FROM ... JOIN` replaces loops that run thousands of statements, each with its own log record overhead and plan lookup.

**Code**
```sql
-- Cursor version (slow)
DECLARE @id INT, @sal DECIMAL(10,2);
DECLARE c CURSOR LOCAL FAST_FORWARD FOR SELECT EmployeeId, Salary FROM dbo.Employees;
OPEN c; FETCH NEXT FROM c INTO @id, @sal;
WHILE @@FETCH_STATUS = 0
BEGIN
    UPDATE dbo.Employees SET Salary = @sal * 1.1 WHERE EmployeeId = @id;
    FETCH NEXT FROM c INTO @id, @sal;
END
CLOSE c; DEALLOCATE c;

-- Set-based (one statement, usually orders of magnitude faster)
UPDATE dbo.Employees SET Salary = Salary * 1.1;
```

### 13.2 [Follow-up] When is a cursor acceptable, and what cursor options matter?

**Answer**
Acceptable for per-row operations that can't be set-based (executing a stored proc/DDL per row, maintenance across databases, small row counts). Use `LOCAL FAST_FORWARD` (or `FORWARD_ONLY READ_ONLY`) to minimize overhead; avoid `STATIC/KEYSET/DYNAMIC` unless needed; always CLOSE and DEALLOCATE.

**Explanation**
Alternatives: `WHILE` loop over a #temp of keys with `TOP (1)` / batched `TOP (n)` processing; for DDL across tables, generate statements with STRING_AGG and run once.

**Code**
```sql
DECLARE @sql NVARCHAR(MAX) = (
  SELECT STRING_AGG(CAST(N'UPDATE STATISTICS ' + QUOTENAME(s.name) + N'.' + QUOTENAME(t.name) + N';' AS NVARCHAR(MAX)), N' ')
  FROM sys.tables t JOIN sys.schemas s ON s.schema_id = t.schema_id);
EXEC sp_executesql @sql;   -- one batch instead of a cursor loop
```

### 13.3 [Medium] Temp table vs table variable: key differences.

**Answer**
Both live in tempdb. Temp tables have statistics, support any indexes and participate in transactions (rollback undoes them); table variables lack column statistics (fixed low estimate before 2019 deferred compilation), allow only inline index definitions, and are *not* rolled back with the transaction. Choose temp tables for bigger or join-heavy intermediate results.

**Explanation**
A table variable's estimate of 1 row can lead to nested loop plans that explode with many rows. Adding `OPTION (RECOMPILE)` gives the true row count. Table variables don't cause recompiles when changed; `#temp` tables can.

**Code**
```sql
DECLARE @tv TABLE (Id INT PRIMARY KEY, Name NVARCHAR(50));
CREATE TABLE #t (Id INT PRIMARY KEY, Name NVARCHAR(50));

BEGIN TRAN;
  INSERT @tv VALUES (1,'a');  INSERT #t VALUES (1,'a');
ROLLBACK;
SELECT COUNT(*) FROM @tv;   -- 1  (table variable ignores ROLLBACK)
SELECT COUNT(*) FROM #t;    -- 0
```

### 13.4 [Hard] How do you pass a list of rows from C# to a stored procedure with a TVP, and why is it better than CSV/XML?

**Answer**
Create a user-defined table type, make the proc parameter `READONLY`, and in ADO.NET set `SqlParameter.SqlDbType = SqlDbType.Structured` with `TypeName`, passing a `DataTable` (or `IEnumerable<SqlDataRecord>`). It's strongly typed, avoids parsing overhead/limits of CSV or XML, and it's set-based.

**Explanation**
TVP limitations: read-only, no statistics (use `OPTION (RECOMPILE)` or copy into #temp for large sets), type name must match, and the type can't be altered while referenced. For tens of thousands+ rows consider `SqlBulkCopy` into staging instead. For the "IN list" case in EF Core 8+, `Contains` translates to OPENJSON/parameter collections.

**Code**
```sql
CREATE TYPE dbo.IdList AS TABLE (Id INT PRIMARY KEY);
GO
CREATE OR ALTER PROCEDURE dbo.usp_GetEmployeesByIds @Ids dbo.IdList READONLY AS
    SELECT e.EmployeeId, e.Name FROM dbo.Employees e JOIN @Ids i ON i.Id = e.EmployeeId;
```
```csharp
var dt = new DataTable();
dt.Columns.Add("Id", typeof(int));
foreach (var id in new[] { 1, 3, 5 }) dt.Rows.Add(id);

using var cmd = new SqlCommand("dbo.usp_GetEmployeesByIds", conn) { CommandType = CommandType.StoredProcedure };
var p = cmd.Parameters.AddWithValue("@Ids", dt);
p.SqlDbType = SqlDbType.Structured;
p.TypeName = "dbo.IdList";
using var r = await cmd.ExecuteReaderAsync();   // Asha, Chen, Eli
```

**Gotchas / follow-ups**
- An `IEnumerable<SqlDataRecord>` source must not be empty (throws); use a `DataTable` (0 rows is fine) or skip the parameter.
- Column order/types of DataTable must match the table type.

### 13.5 [Tricky] Temp table caching, scope and naming pitfalls; global temp tables.

**Answer**
`#temp` inside a procedure is dropped on exit (and cached/reused by SQL Server when certain rules are met — no named constraints, no DDL after creation like ALTER). Named constraints on temp tables (`CONSTRAINT PK_x`) collide across concurrent sessions executing the same proc; use unnamed constraints. `##global` temp tables are visible to all sessions until the creating session ends and no one references it.

**Explanation**
Also a pre-existing check `IF OBJECT_ID('tempdb..#t') IS NOT NULL DROP TABLE #t;` (or `DROP TABLE IF EXISTS #t` in 2016+) is idiomatic in scripts. Temp tables created in dynamic SQL aren't visible outside.

**Code**
```sql
CREATE TABLE #a (Id INT CONSTRAINT PK_a PRIMARY KEY);    -- second concurrent session creating it: Msg 2714 "There is already an object named PK_a"
CREATE TABLE #b (Id INT PRIMARY KEY);                    -- safe: auto-generated unique name
DROP TABLE IF EXISTS #a, #b;
```

**Interview traps**
- "Table variables live in memory" — they're in tempdb.
- Table variables ignore ROLLBACK.
- Named constraints in `#temp` break concurrency.
- Cursor "because it's easier" in a perf-sensitive path.
- TVP is READONLY inside the proc.

**Cheat sheet**
- Cursor = last resort; `LOCAL FAST_FORWARD`.
- #temp: stats + indexes + transactional; table var: no stats (2019 deferred compile), not rolled back.
- TVP: `CREATE TYPE ... AS TABLE`, `READONLY`, `SqlDbType.Structured`.
- Unnamed constraints on temp tables.
- For huge row sets use SqlBulkCopy.

---

## Topic 14: Temporal tables, Change Tracking & CDC

**Study notes**

| Feature | Captures | Storage | Typical use |
|---|---|---|---|
| **System-versioned temporal table** (2016+) | Full row history with validity period (`ValidFrom`/`ValidTo`, `datetime2`, UTC) | Automatic history table | Auditing, point-in-time queries, undo |
| **Change Tracking (CT)** | *Which rows* changed (PK + version, optional changed-column mask), not old values | Internal side tables | Sync scenarios (client pulls changes since version N) |
| **Change Data Capture (CDC)** | Row-level before/after values via log reader | `cdc.*` change tables, SQL Agent capture/cleanup jobs | ETL/incremental loads, auditing |

Temporal specifics: requires a PK; period columns `GENERATED ALWAYS AS ROW START/END` with `PERIOD FOR SYSTEM_TIME`; query with `FOR SYSTEM_TIME AS OF | FROM..TO | BETWEEN..AND | CONTAINED IN | ALL`. UPDATE/DELETE writes the *old* row version to history (INSERT writes none). Period columns can be `HIDDEN`. History retention via `HISTORY_RETENTION_PERIOD`. Schema changes on the current table propagate to history. History table can't be modified directly while versioning is ON; to truncate/alter, set `SYSTEM_VERSIONING = OFF` first. Not supported: `TRUNCATE` of current table, `INSTEAD OF` triggers on temporal tables, etc.

### 14.1 [Easy] What is a system-versioned temporal table?

**Answer**
A table that automatically keeps the full history of data changes in a paired history table, with the period of validity of each row version tracked by two system-maintained `datetime2` columns. You query the past with `FOR SYSTEM_TIME`.

**Explanation**
The engine sets ValidFrom/ValidTo (UTC transaction start time). You can't write those columns directly.

**Code**
```sql
CREATE TABLE dbo.Product (
    ProductId INT PRIMARY KEY,
    Name  NVARCHAR(50) NOT NULL,
    Price DECIMAL(10,2) NOT NULL,
    ValidFrom DATETIME2 GENERATED ALWAYS AS ROW START HIDDEN NOT NULL,
    ValidTo   DATETIME2 GENERATED ALWAYS AS ROW END   HIDDEN NOT NULL,
    PERIOD FOR SYSTEM_TIME (ValidFrom, ValidTo)
) WITH (SYSTEM_VERSIONING = ON (HISTORY_TABLE = dbo.ProductHistory, HISTORY_RETENTION_PERIOD = 6 MONTHS));

INSERT dbo.Product (ProductId, Name, Price) VALUES (1, N'Widget', 10.00);
UPDATE dbo.Product SET Price = 12.00 WHERE ProductId = 1;   -- old row (10.00) goes to dbo.ProductHistory
```

### 14.2 [Follow-up] How do you query data as of a past time and view all versions?

**Answer**
Add `FOR SYSTEM_TIME AS OF '<utc time>'` for a point-in-time snapshot, or `ALL` for every version (current + history). Times are in UTC.

**Explanation**
`BETWEEN..AND` includes rows that became active exactly at the upper bound; `FROM..TO` excludes it; `CONTAINED IN` returns only versions whose whole lifetime is inside the range. The optimizer unions current + history tables behind the scenes (use an index on history `(ValidTo, ValidFrom)` — the default clustered index).

**Code**
```sql
SELECT ProductId, Price, ValidFrom, ValidTo
FROM dbo.Product FOR SYSTEM_TIME ALL ORDER BY ValidFrom;    -- hidden cols must be selected explicitly
-- 1 | 10.00 | t0 | t1       (history)
-- 1 | 12.00 | t1 | 9999-12-31 23:59:59.9999999   (current)

SELECT * FROM dbo.Product FOR SYSTEM_TIME AS OF '2024-06-01T00:00:00';   -- state at that time (UTC)
```

### 14.3 [Medium] Temporal tables vs Change Tracking vs CDC — how do you choose?

**Answer**
Temporal = automatic row history with time-travel queries inside the database. Change Tracking = lightweight "what changed since version X" for sync, no old values. CDC = asynchronous capture of every change with before/after values from the transaction log, for ETL/downstream consumers. Choose by need: audit/time-travel → temporal; sync clients → CT; data pipeline → CDC.

**Explanation**
CT is synchronous (inside the transaction, small overhead), CDC is asynchronous (log reader via SQL Agent jobs; needs Agent; Standard edition support from SQL 2016 SP1), temporal is synchronous (writes history in same transaction). Temporal doesn't record *who* changed — add an audit column (`SUSER_SNAME()` default) yourself.

**Code**
```sql
-- Change Tracking
ALTER DATABASE CURRENT SET CHANGE_TRACKING = ON (CHANGE_RETENTION = 2 DAYS, AUTO_CLEANUP = ON);
ALTER TABLE dbo.Employees ENABLE CHANGE_TRACKING WITH (TRACK_COLUMNS_UPDATED = ON);
DECLARE @last BIGINT = CHANGE_TRACKING_CURRENT_VERSION();
-- later:
SELECT ct.EmployeeId, ct.SYS_CHANGE_OPERATION, e.Name
FROM CHANGETABLE(CHANGES dbo.Employees, @last) ct
LEFT JOIN dbo.Employees e ON e.EmployeeId = ct.EmployeeId;     -- I/U/D

-- CDC
EXEC sys.sp_cdc_enable_db;
EXEC sys.sp_cdc_enable_table @source_schema = N'dbo', @source_name = N'Employees', @role_name = NULL;
-- Changes via cdc.fn_cdc_get_all_changes_dbo_Employees(@from_lsn, @to_lsn, 'all')
```

### 14.4 [Hard] How do temporal tables behave with schema changes, deletes of history, and in-memory/EF Core?

**Answer**
`ALTER TABLE` add/alter column on the current table is applied to the history table automatically. To purge history use retention or switch versioning off: `SET (SYSTEM_VERSIONING = OFF)`, delete, then back ON with `DATA_CONSISTENCY_CHECK`. EF Core 6+ supports temporal tables: `ToTable("t", b => b.IsTemporal())` and query operators `TemporalAsOf`, `TemporalAll`, etc. Migrations create the tables.

**Explanation**
History grows without bound unless retention is set (retention cleanup runs in background on pages/rowgroups; best with clustered columnstore history). The hidden period columns aren't returned by `SELECT *`. All changes within one transaction share the same period timestamps (the transaction start time), so several updates in one transaction don't produce distinguishable versions.

**Code**
```sql
ALTER TABLE dbo.Product SET (SYSTEM_VERSIONING = OFF);
-- ... maintenance on dbo.ProductHistory (e.g. DELETE old rows) ...
ALTER TABLE dbo.Product SET (SYSTEM_VERSIONING = ON (HISTORY_TABLE = dbo.ProductHistory, DATA_CONSISTENCY_CHECK = ON));
```
```csharp
modelBuilder.Entity<Product>().ToTable("Product", b => b.IsTemporal());
var past = await db.Products.TemporalAsOf(DateTime.UtcNow.AddDays(-7)).ToListAsync();
```

**Gotchas / follow-ups**
- Times are UTC; passing local time yields wrong snapshots.
- Dropping the current table requires turning versioning off first.
- `TRUNCATE TABLE` on the current table isn't allowed.

### 14.5 [Tricky] CDC/CT gotchas — what are the operational traps?

**Answer**
CDC depends on the SQL Agent capture job; if the log reader lags, log truncation is delayed (log growth). CT retention shorter than a client's sync interval means the client must reinitialize (`CHANGE_TRACKING_MIN_VALID_VERSION`). Neither captures *who* made the change. Bulk loads into CDC-enabled tables are fully logged (no minimal logging), since the capture process reads the log.

**Explanation**
CT reports the net change since a version (an insert followed by an update looks like one insert); CDC reports each individual change. For temporal, time-travel queries over large histories need appropriate indexes/columnstore.

**Code**
```sql
SELECT CHANGE_TRACKING_MIN_VALID_VERSION(OBJECT_ID('dbo.Employees')) AS MinValid;
-- If client's last-synced version < MinValid → full resync required
```

**Interview traps**
- Temporal times are UTC.
- CT doesn't store old values; CDC does.
- CDC needs SQL Agent (jobs) and affects log truncation.
- History table is not directly writable while versioning is ON.

**Cheat sheet**
- Temporal: `PERIOD FOR SYSTEM_TIME` + `SYSTEM_VERSIONING = ON`; `FOR SYSTEM_TIME AS OF/ALL`.
- CT: net change keys since version; CDC: before/after values, async.
- Retention: `HISTORY_RETENTION_PERIOD`, CT `CHANGE_RETENTION`.
- EF Core: `IsTemporal()`, `TemporalAsOf`.

---

## Topic 15: Sequence/identity gaps, computed columns, MERGE alternatives

**Study notes**

**IDENTITY**: per-table auto-increment (`IDENTITY(seed, incr)`), assigned at insert time and **not rolled back** → gaps after rolled-back/failed inserts, deletes, restarts (since 2012 server restart may skip a block of values—with identity cache of 1000 (int)—or 10000 for bigint; trace flag 272 / `IDENTITY_CACHE = OFF` database scoped option (2017+) avoids this). `SET IDENTITY_INSERT t ON` allows explicit values (one table per session at a time). Retrieve new id with `SCOPE_IDENTITY()` or `OUTPUT`; avoid `@@IDENTITY` and `IDENT_CURRENT`. `TRUNCATE` resets identity, `DELETE` doesn't; `DBCC CHECKIDENT` reseeds.

**SEQUENCE** (2012+): schema-level object, `NEXT VALUE FOR`, usable across tables, as a default constraint, with `CACHE n`, `CYCLE`, and `sp_sequence_get_range`. Also gaps with rollback/crash. Gap-free numbering requires a locked counter table (serializes writers) — rarely worth it.

**Computed columns**: `col AS expr [PERSISTED]`. Non-persisted: calculated on read (no storage; can be indexed if deterministic & precise). `PERSISTED`: stored on disk, updated with row, indexable, can be used when expression is deterministic. Good for JSON_VALUE indexing, normalization (`LOWER(email)`), search-friendly forms. Not allowed: non-deterministic indexed columns (`GETDATE()`), subqueries.

**MERGE**: single statement UPSERT/sync (`WHEN MATCHED / NOT MATCHED [BY TARGET] / NOT MATCHED BY SOURCE`). Has a history of bugs and race conditions (needs `HOLDLOCK`), so many teams prefer separate `UPDATE` + `INSERT` in a transaction.

### 15.1 [Easy] Why do identity values have gaps and can you rely on them being gap-free?

**Answer**
IDENTITY values are generated outside the transaction, so a rollback, failed insert, deleted row or server restart leaves a gap. They guarantee uniqueness (with a unique index) and increasing order, not contiguity. Never use identity as a business "invoice number" that must be gapless.

**Explanation**
The identity increment is not undone by rollback because it must be concurrently available to other sessions.

**Code**
```sql
BEGIN TRAN;
INSERT dbo.Orders (CustomerId, OrderDate, Amount, Status) VALUES (99,'2024-03-01',1,'New');
ROLLBACK;
INSERT dbo.Orders (CustomerId, OrderDate, Amount, Status) OUTPUT inserted.OrderId VALUES (99,'2024-03-01',1,'New');
-- OrderId = 9, not 8 (7 sample rows; the rolled-back row consumed 8)
```

### 15.2 [Follow-up] SCOPE_IDENTITY vs @@IDENTITY vs IDENT_CURRENT vs OUTPUT.

**Answer**
`SCOPE_IDENTITY()` = last identity in the current session *and* scope (safe default). `@@IDENTITY` = last in the session across scopes (picks up trigger inserts — dangerous). `IDENT_CURRENT('t')` = last value for a table across all sessions (racy). `OUTPUT inserted.Id` returns ids for multi-row inserts and is the best option.

**Explanation**
Only OUTPUT (or `OUTPUT INTO @t`) gives all ids of a multi-row insert, which is how EF Core fetches keys.

**Code**
```sql
DECLARE @ids TABLE (OrderId INT);
INSERT dbo.Orders (CustomerId, OrderDate, Amount, Status)
OUTPUT inserted.OrderId INTO @ids
VALUES (40,'2024-03-02',10,'New'),(40,'2024-03-03',20,'New');
SELECT OrderId FROM @ids;       -- two ids
SELECT SCOPE_IDENTITY();        -- last one only
```

### 15.3 [Medium] SEQUENCE vs IDENTITY: when to prefer a sequence?

**Answer**
Use a sequence when you need to obtain a number *before* insert (e.g. client-side key assignment/EF Core HiLo), share numbering across tables, control caching/cycling, or reset/restart easily. Identity is simpler for single-table surrogate keys. Both can have gaps.

**Explanation**
Sequences are not tied to a column's lifetime; a default `NEXT VALUE FOR` on a column mimics identity. Sequence `CACHE` improves performance but a crash loses cached values (gaps). EF Core `UseHiLo` uses a sequence to allocate blocks.

**Code**
```sql
CREATE SEQUENCE dbo.seq_Ticket AS INT START WITH 1000 INCREMENT BY 1 CACHE 50;
SELECT NEXT VALUE FOR dbo.seq_Ticket;    -- 1000
SELECT NEXT VALUE FOR dbo.seq_Ticket;    -- 1001
CREATE TABLE dbo.Ticket (TicketId INT PRIMARY KEY DEFAULT (NEXT VALUE FOR dbo.seq_Ticket), Title NVARCHAR(50));
```

### 15.4 [Hard] Persisted vs non-persisted computed columns, and indexing them.

**Answer**
A non-persisted computed column is evaluated on each read; a PERSISTED one is stored and recomputed only when its inputs change. Either can be indexed if the expression is deterministic and precise; an index on a computed column materializes the values in the index anyway. Persisted columns cost storage and write time but avoid recomputation on reads, and the optimizer can use an index on the computed column even when the query uses the expression text rather than the column name.

**Explanation**
Use cases: searching on normalized form, `JSON_VALUE` extracts, `DATEFROMPARTS` buckets. Requirements for indexing: deterministic, precise, correct `SET` options (`ANSI_NULLS`, `QUOTED_IDENTIFIER`, etc. ON) at create and DML time. Scalar UDFs in computed columns block parallelism (even inlined UDFs aren't inlined there).

**Code**
```sql
ALTER TABLE dbo.Orders ADD OrderYear AS YEAR(OrderDate) PERSISTED;
CREATE INDEX IX_Orders_Year ON dbo.Orders(OrderYear) INCLUDE (Amount);
SELECT SUM(Amount) FROM dbo.Orders WHERE YEAR(OrderDate) = 2024;   -- may seek IX_Orders_Year (expression matching)
SELECT COLUMNPROPERTY(OBJECT_ID('dbo.Orders'),'OrderYear','IsComputed');   -- 1
```

**Gotchas / follow-ups**
- Computed column using `GETDATE()` can't be PERSISTED/indexed (non-deterministic).
- Altering the expression requires dropping and re-adding the column.
- Wrong SET options → error when modifying table with an indexed computed column.

### 15.5 [Tricky] MERGE — why do many avoid it, and what's the safe upsert pattern?

**Answer**
MERGE has had numerous bugs and, without `HOLDLOCK` (SERIALIZABLE), concurrent sessions can both see "not matched" and insert duplicates / fail on PK. Safer: `UPDATE ... ; IF @@ROWCOUNT = 0 INSERT ...` in a transaction with `UPDLOCK, HOLDLOCK` hints, or `INSERT ... WHERE NOT EXISTS` plus `UPDATE`. If using MERGE, use `HOLDLOCK`, a unique index on the join key and end with `;`.

**Explanation**
MERGE must terminate with semicolon, a source row must match at most one target row (else error 8672), and `OUTPUT $action` shows the action per row. Triggers fire per action type.

**Code**
```sql
-- Safe upsert
BEGIN TRAN;
  UPDATE dbo.Departments WITH (UPDLOCK, HOLDLOCK) SET DeptName = N'People Ops' WHERE DeptId = 3;
  IF @@ROWCOUNT = 0 INSERT dbo.Departments (DeptId, DeptName) VALUES (3, N'People Ops');
COMMIT;

-- MERGE version (with HOLDLOCK)
MERGE dbo.Departments WITH (HOLDLOCK) AS t
USING (VALUES (4, N'Finance')) AS s(DeptId, DeptName) ON t.DeptId = s.DeptId
WHEN MATCHED THEN UPDATE SET DeptName = s.DeptName
WHEN NOT MATCHED THEN INSERT (DeptId, DeptName) VALUES (s.DeptId, s.DeptName)
OUTPUT $action, inserted.DeptId;      -- INSERT | 4
```

**Interview traps**
- Identity is not gapless; `@@IDENTITY` vs `SCOPE_IDENTITY()`.
- TRUNCATE resets identity, DELETE doesn't.
- Persisted ≠ indexed (but indexes need determinism).
- MERGE without HOLDLOCK race / semicolon.
- Sequence values can also have gaps.

**Cheat sheet**
- Identity: gaps on rollback/restart; use OUTPUT for ids.
- Sequence: `NEXT VALUE FOR`, cache, usable before insert, HiLo.
- Computed: PERSISTED stores; index requires determinism.
- Upsert: UPDATE-then-INSERT with UPDLOCK,HOLDLOCK or MERGE+HOLDLOCK.

---

## Topic 16: hierarchyid, full-text search, bulk operations, SQL Agent

**Study notes**

**hierarchyid**: CLR-based system data type for tree paths (e.g. `/1/2/3/`). Methods: `GetRoot()`, `GetLevel()`, `GetAncestor(n)`, `GetDescendant(c1,c2)`, `IsDescendantOf(x)`, `ToString()`, `Parse()`, `GetReparentedValue`. Index depth-first (`INDEX ... (hid)`) for subtree queries and breadth-first (`(level, hid)`) for children. Pros: compact, fast subtree queries. Cons: manual management of paths, not enforced uniqueness/tree validity, moves are awkward; adjacency list + recursive CTE is simpler and more common.

**Full-text search**: word/phrase/inflection/proximity search on character/`varbinary`/`xml` columns via a **full-text index** (needs a unique, non-null, single-column key index; catalog; population via change tracking/crawl). Predicates: `CONTAINS`, `FREETEXT`; TVFs `CONTAINSTABLE`, `FREETEXTTABLE` return `RANK`. `LIKE '%x%'` can't use indexes (scan); `LIKE 'x%'` can seek.

**Bulk operations**: `BULK INSERT` (T-SQL, server-side file), `bcp` (command-line utility), `OPENROWSET(BULK ...)`, `SqlBulkCopy` (.NET, via TDS bulk-load protocol), SSIS. **Minimal logging** requires the bulk-logged or simple recovery model, a table lock (`TABLOCK`), and a suitable target (heap or empty table; extra rules for tables with indexes — see 16.4); it logs page/extent allocations instead of every row, making loads much faster. Always batch (`BATCHSIZE`) for huge loads and consider dropping/disabling nonclustered indexes.

**SQL Server Agent**: scheduler service (not in Express) for **jobs** → **steps** (T-SQL, PowerShell, SSIS, OS cmd) → **schedules**, with alerts, operators and Database Mail notifications; metadata in `msdb` (`sp_add_job`, `sp_add_jobstep`, `sp_add_jobschedule`, `sp_add_jobserver`); history in `msdb.dbo.sysjobhistory`. Azure SQL DB doesn't have Agent (use Elastic Jobs, Azure Automation, Functions/Logic Apps); Managed Instance does.

### 16.1 [Easy] How do you do a full-text search, and how does it differ from LIKE?

**Answer**
Create a full-text catalog/index on the column and query with `CONTAINS`/`FREETEXT`. It does linguistic matching (word forms, thesaurus, proximity, ranking) via an inverted index, while `LIKE '%word%'` does a scan with pattern matching and no linguistics.

**Explanation**
The index needs a unique key index (`KEY INDEX`). Population is asynchronous by default (change tracking AUTO), so new rows can lag briefly. Noise words (stoplist) are ignored.

**Code**
```sql
CREATE FULLTEXT CATALOG ftCat AS DEFAULT;
CREATE UNIQUE NONCLUSTERED INDEX UX_Employees_EmployeeId ON dbo.Employees (EmployeeId);   -- full-text KEY INDEX must be a unique, single-column, non-nullable index
CREATE FULLTEXT INDEX ON dbo.Employees (Name) KEY INDEX UX_Employees_EmployeeId
  WITH CHANGE_TRACKING AUTO;

SELECT Name FROM dbo.Employees WHERE CONTAINS(Name, N'"Asha" OR "Bram"');   -- Asha, Bram
SELECT Name FROM dbo.Employees WHERE CONTAINS(Name, N'"Ch*"');              -- prefix: Chen
SELECT k.[RANK], e.Name FROM CONTAINSTABLE(dbo.Employees, Name, N'Asha') k JOIN dbo.Employees e ON e.EmployeeId = k.[KEY];
```

### 16.2 [Follow-up] CONTAINS vs FREETEXT vs LIKE; which can use indexes?

**Answer**
CONTAINS: precise boolean/prefix/proximity/inflectional queries. FREETEXT: fuzzy natural-language match of meaning (splits into words, thesaurus/inflection). LIKE with leading wildcard can't seek an index; `LIKE 'abc%'` can. Full-text is the right tool for word search on large text columns.

**Explanation**
Use `FORMSOF(INFLECTIONAL, run)` for word forms, `NEAR` for proximity. Wrap user input to avoid syntax errors in CONTAINS (quote phrases; escape/validate).

**Code**
```sql
SELECT Name FROM dbo.Employees WHERE Name LIKE N'A%';          -- sargable prefix → index seek (if indexed)
SELECT Name FROM dbo.Employees WHERE Name LIKE N'%ra%';        -- scan: Dara
SELECT Name FROM dbo.Employees WHERE CONTAINS(Name, N'FORMSOF(INFLECTIONAL, "Asha")');
```

### 16.3 [Medium] hierarchyid: build and query a tree; compare with adjacency list.

**Answer**
Store each node's path as hierarchyid; `GetAncestor(1)` is the parent, `IsDescendantOf` selects a subtree, `GetLevel()` gives depth. Compared with adjacency list + recursive CTE, hierarchyid gives compact keys and fast subtree/ancestor queries (with a depth-first index), but inserts/moves need helper code and nothing prevents orphaned/invalid paths.

**Explanation**
Use `GetDescendant(NULL, NULL)` for the first child; to insert a child after an existing sibling pass it as the first arg. Lock a parent row while generating siblings to avoid duplicate paths.

**Code**
```sql
CREATE TABLE dbo.OrgNode (Node HIERARCHYID PRIMARY KEY, Name NVARCHAR(50), Lvl AS Node.GetLevel());
DECLARE @root HIERARCHYID = HIERARCHYID::GetRoot();
INSERT dbo.OrgNode VALUES (@root, N'Asha');
DECLARE @bram HIERARCHYID = @root.GetDescendant(NULL, NULL);
INSERT dbo.OrgNode VALUES (@bram, N'Bram');
INSERT dbo.OrgNode VALUES (@bram.GetDescendant(NULL, NULL), N'Chen');

SELECT Node.ToString() AS Path, Name FROM dbo.OrgNode
WHERE Node.IsDescendantOf(@bram) = 1;      -- /1/ Bram (a node is its own descendant) ; /1/1/ Chen
```

### 16.4 [Hard] Bulk loading: BULK INSERT vs bcp vs SqlBulkCopy, and when is it minimally logged?

**Answer**
All use the bulk-load path. `BULK INSERT` reads a file visible to the *server*; `bcp` is a client CLI utility; `SqlBulkCopy` streams from a .NET `DataTable`/`IDataReader` over TDS. Minimal logging requires bulk-logged or simple recovery, `TABLOCK` (or the bulk-load API's table-lock option), and a suitable target (heap, or empty clustered index table, with specific rules for populated indexed tables); triggers and replication prevent it in some cases. In full recovery model loads are fully logged.

**Explanation**
Minimally logged loads log extent/page allocations rather than individual rows, so they are far faster and use less log, but point-in-time restore inside the bulk-logged window is restricted. Tips: batch size (`BATCHSIZE`), order hint (`ORDER`) matching clustered key, load into a staging heap then `INSERT...SELECT`, constraints are not checked by default for BULK INSERT (`CHECK_CONSTRAINTS` option), triggers don't fire unless `FIRE_TRIGGERS`.

**Code**
```sql
BULK INSERT dbo.Orders_Staging
FROM 'C:\data\orders.csv'
WITH (FORMAT = 'CSV', FIRSTROW = 2, FIELDTERMINATOR = ',', ROWTERMINATOR = '\n',
      TABLOCK, BATCHSIZE = 100000, MAXERRORS = 10, ERRORFILE = 'C:\data\orders.err');
```
```text
bcp MyDb.dbo.Orders_Staging in C:\data\orders.dat -S . -T -c -b 100000 -h "TABLOCK"
```

**Gotchas / follow-ups**
- `BULK INSERT` path is relative to the SQL Server machine, and the service account needs read access.
- `FORMAT='CSV'` requires 2017+.
- INSERT...SELECT can also be minimally logged with `TABLOCK` (2008+) into heap/empty table under bulk-logged/simple.

### 16.5 [Tricky] SQL Agent: create a job from T-SQL and know its limitations.

**Answer**
Agent jobs are defined in `msdb`: add the job, one or more steps, a schedule and attach to the server. Limitations: not available in Express or Azure SQL Database (use Elastic Jobs/external schedulers), job runs under the Agent service account/proxy (permissions pitfalls), overlapping runs are not prevented automatically (a job won't start if it's still running, but separate jobs can collide), and failure notifications require Database Mail + operator.

**Explanation**
Design jobs idempotently, log to a table, set retry attempts, and use proxies for non-T-SQL steps. Check `msdb.dbo.sysjobhistory` / `sp_help_jobhistory` for failures.

**Code**
```sql
USE msdb;
EXEC dbo.sp_add_job @job_name = N'Nightly Purge';
EXEC dbo.sp_add_jobstep @job_name = N'Nightly Purge', @step_name = N'Delete old orders',
     @subsystem = N'TSQL', @database_name = N'MyDb',
     @command = N'DELETE TOP (5000) FROM dbo.Orders WHERE OrderDate < DATEADD(YEAR,-2,GETDATE());',
     @retry_attempts = 2, @retry_interval = 5;
EXEC dbo.sp_add_jobschedule @job_name = N'Nightly Purge', @name = N'Daily 2AM',
     @freq_type = 4, @freq_interval = 1, @active_start_time = 020000;
EXEC dbo.sp_add_jobserver @job_name = N'Nightly Purge';   -- attach to local server (required)
```

**Interview traps**
- hierarchyid doesn't enforce valid parent existence.
- Full-text index population is asynchronous (stale results).
- BULK INSERT file path is server-side; minimally logged requires TABLOCK + recovery model.
- SQL Agent isn't in Express/Azure SQL DB.

**Cheat sheet**
- hierarchyid: `GetRoot/GetDescendant/GetAncestor/IsDescendantOf/GetLevel`.
- FTS: `CONTAINS`, `FREETEXT`, `CONTAINSTABLE`; needs unique key index.
- Bulk: BULK INSERT (server path), bcp (CLI), SqlBulkCopy (.NET); TABLOCK + bulk-logged/simple = minimal logging.
- Agent: job → steps → schedule; metadata in msdb.

---

## Topic 17: ADO.NET (SqlClient) basics and Dapper

**Study notes**

Package: `Microsoft.Data.SqlClient` (current; `System.Data.SqlClient` is legacy). Core types:

```
SqlConnection --(opens from pool)--> SqlCommand (text/proc + SqlParameters)
      |                                  |-- ExecuteReader  -> SqlDataReader (forward-only, streaming)
      |                                  |-- ExecuteScalar  -> first column of first row
      |                                  |-- ExecuteNonQuery-> rows affected (DML/DDL)
      +-- BeginTransaction -> SqlTransaction ;  SqlBulkCopy for bulk loads
```

- **Connection pooling** is on by default, keyed by exact connection string (+ identity/credentials). `Close()`/`Dispose()` returns the connection to the pool (it doesn't close the physical connection). Default `Max Pool Size=100`, `Min Pool Size=0`, `Connection Timeout=15`. Open late, close early (`using`/`await using`). Pool exhaustion ("timeout expired… max pool size reached") = leaked connections or long transactions. First command after pool reuse does `sp_reset_connection`.
- **Parameters**: always use `SqlParameter` (prevents injection, enables plan reuse). Specify type/size (`Add(name, SqlDbType, size)`) rather than `AddWithValue` to avoid implicit conversions/plan-cache fragmentation (e.g. string lengths, `nvarchar` vs `varchar` index-scan problems).
- **Async**: `OpenAsync`, `ExecuteReaderAsync`, `ReadAsync`, `ExecuteScalarAsync`, pass `CancellationToken`. Use `CommandBehavior.SequentialAccess` for large blobs. A `SqlDataReader` holds the connection busy until disposed; MARS (`MultipleActiveResultSets=True`) permits multiple active readers but has caveats.
- `SqlBulkCopy`: fast bulk insert (`DestinationTableName`, `ColumnMappings`, `BatchSize`, `BulkCopyTimeout`, options like `TableLock`).
- Transactions: `SqlTransaction` (assign to every command) or `TransactionScope` (with `TransactionScopeAsyncFlowOption.Enabled` for async).
- **Dapper**: micro-ORM extension methods on `IDbConnection` (`Query<T>`, `QueryFirstOrDefault<T>`, `QuerySingle`, `Execute`, `ExecuteScalar`, `QueryMultiple`, multi-mapping) that parameterize via anonymous objects. It maps columns → properties by name; no change tracking, no LINQ, you write SQL.

### 17.1 [Easy] Read rows with SqlConnection/SqlCommand/SqlDataReader safely (parameterized, async).

**Answer**
Create the connection in a `using`, pass inputs as `SqlParameter`s, `ExecuteReaderAsync`, loop `ReadAsync`. Dispose everything so the connection returns to the pool.

**Explanation**
The reader streams rows; get values by ordinal (fastest) or name, and check `IsDBNull` for nullable columns.

**Code**
```csharp
using Microsoft.Data.SqlClient;

await using var conn = new SqlConnection(connectionString);
await conn.OpenAsync(ct);

await using var cmd = new SqlCommand(
    "SELECT EmployeeId, Name, ManagerId FROM dbo.Employees WHERE DeptId = @DeptId", conn);
cmd.Parameters.Add("@DeptId", SqlDbType.Int).Value = 1;

await using var reader = await cmd.ExecuteReaderAsync(ct);
while (await reader.ReadAsync(ct))
{
    int id = reader.GetInt32(0);
    string name = reader.GetString(1);
    int? manager = reader.IsDBNull(2) ? null : reader.GetInt32(2);
    Console.WriteLine($"{id} {name} {manager}");    // 1 Asha | 2 Bram 1 | 3 Chen 2 | 4 Dara 2
}
```

### 17.2 [Follow-up] ExecuteScalar vs ExecuteNonQuery vs ExecuteReader; calling a stored procedure with OUTPUT and return value.

**Answer**
ExecuteReader streams result sets; ExecuteScalar returns the first column of the first row (or null/DBNull); ExecuteNonQuery returns affected rows (DML) or -1 for non-DML/NOCOUNT ON. For procs set `CommandType.StoredProcedure` and add parameters with `Direction = Output` / `ReturnValue`; read them *after* the reader is closed.

**Explanation**
Output and return parameters are only populated once all result sets are consumed/closed.

**Code**
```csharp
await using var cmd = new SqlCommand("dbo.usp_GetDeptStats", conn) { CommandType = CommandType.StoredProcedure };
cmd.Parameters.Add("@DeptId", SqlDbType.Int).Value = 1;
cmd.Parameters.Add("@MinSalary", SqlDbType.Decimal).Value = 6500m;
var cnt = cmd.Parameters.Add("@Headcount", SqlDbType.Int); cnt.Direction = ParameterDirection.Output;
var rc  = cmd.Parameters.Add("@rc", SqlDbType.Int);        rc.Direction  = ParameterDirection.ReturnValue;

await using (var r = await cmd.ExecuteReaderAsync()) { while (await r.ReadAsync()) { /* consume rows */ } }
Console.WriteLine($"{(int)cnt.Value} {(int)rc.Value}");   // 2 0

var total = (int)(await new SqlCommand("SELECT COUNT(*) FROM dbo.Orders", conn).ExecuteScalarAsync())!;
```

### 17.3 [Medium] Explain connection pooling: what it is, what breaks it, and how to diagnose pool exhaustion.

**Answer**
The pool keeps physical connections open and hands them out on `Open()`; `Close/Dispose` returns them. Pools are separate per distinct connection string (and per Windows identity for integrated security). Exhaustion (max 100 by default) happens when connections aren't disposed, transactions/readers are long-lived, or too much parallelism. Fixes: always dispose (`using`), shorten work held under a connection, don't hold connections across slow I/O, tune Max Pool Size only after fixing leaks.

**Explanation**
Things that fragment pools: varying connection strings (e.g. different `Application Name`, per-user credentials). `SqlConnection.ClearAllPools()` clears them. Don't cache a `SqlConnection` object statically — cache the connection *string*; `SqlConnection` instances aren't thread-safe. EF Core's `DbContext` pooling is a different thing (reuses context objects) but still uses ADO.NET pool underneath.

**Code**
```csharp
var csb = new SqlConnectionStringBuilder {
    DataSource = "localhost", InitialCatalog = "MyDb", IntegratedSecurity = true,
    MaxPoolSize = 200, MinPoolSize = 5, ApplicationName = "MyApp", Encrypt = true, TrustServerCertificate = false
};
// Diagnose: SELECT COUNT(*) FROM sys.dm_exec_sessions WHERE program_name = 'MyApp';
```

**Gotchas / follow-ups**
- A connection leaked (never disposed) stays "in use" until GC finalizes it → pool timeouts under load.
- Pooled connection state (e.g. `SET` options, `USE db`) is reset by `sp_reset_connection` on reuse; still, don't rely on session state (temp tables, SET options) persisting across separate `Open()` calls.
- Transient faults: use retry logic (`SqlRetryLogicOption` in Microsoft.Data.SqlClient 3.0+, or Polly / EF's `EnableRetryOnFailure`).

### 17.4 [Hard] SqlBulkCopy: load a DataTable/IDataReader, mappings, batching, and TVP vs bulk copy.

**Answer**
`SqlBulkCopy` streams rows via the bulk-load protocol into a destination table; configure `DestinationTableName`, `ColumnMappings` (by name/ordinal — unmapped columns default to ordinal order which causes bugs), `BatchSize`, `BulkCopyTimeout`, and `SqlBulkCopyOptions.TableLock` for minimal logging potential. Use an `IDataReader` source (e.g. FastMember/`DbDataReader`) to stream rather than load everything into a DataTable. For many rows (>~1000s) use SqlBulkCopy; for small sets processed by logic in a proc use a TVP.

**Explanation**
Default options don't fire triggers or check constraints (add `FireTriggers`/`CheckConstraints`). Wrap in a `SqlTransaction` (pass to the constructor) for atomicity; with `UseInternalTransaction` each batch commits separately. Upsert pattern: bulk copy to a staging/temp table then `MERGE`/`UPDATE+INSERT` set-based.

**Code**
```csharp
var table = new DataTable();
table.Columns.Add("CustomerId", typeof(int));
table.Columns.Add("OrderDate", typeof(DateTime));
table.Columns.Add("Amount", typeof(decimal));
table.Columns.Add("Status", typeof(string));
table.Rows.Add(50, new DateTime(2024, 3, 1), 99.5m, "New");
table.Rows.Add(50, new DateTime(2024, 3, 2), 10m, "Paid");

await using var tx = (SqlTransaction)await conn.BeginTransactionAsync();
using var bulk = new SqlBulkCopy(conn, SqlBulkCopyOptions.TableLock, tx)
{
    DestinationTableName = "dbo.Orders",
    BatchSize = 5000,
    BulkCopyTimeout = 120
};
foreach (DataColumn c in table.Columns) bulk.ColumnMappings.Add(c.ColumnName, c.ColumnName);  // OrderId identity is skipped
await bulk.WriteToServerAsync(table);
await tx.CommitAsync();
```

**Gotchas / follow-ups**
- Identity column: omit from mappings (or use `KeepIdentity`).
- Type mismatch errors reported as "Received an invalid column length from the bcp client" — check string lengths/mappings.
- SqlBulkCopy can't return generated ids; use staging + `OUTPUT`.

### 17.5 [Tricky] Transactions, cancellation and common traps in ADO.NET.

**Answer**
Assign the `SqlTransaction` to every command (or use `TransactionScope` with async flow); commit explicitly or the transaction rolls back on dispose. `CommandTimeout` (default 30 s) is per command and is *not* the connection timeout; on timeout a `SqlException` (Number -2) is raised and the server may keep running briefly unless attention is processed. Use `CancellationToken` to cancel. Avoid sync-over-async (`.Result`) and don't share a `SqlConnection` across threads.

**Explanation**
Catch `SqlException` and inspect `Number` / `Errors` (deadlock 1205, unique violation 2627/2601, timeout -2). Retry only transient errors and idempotent operations. If two readers are active on one connection without MARS you get "There is already an open DataReader associated with this Command".

**Code**
```csharp
await using var tx = (SqlTransaction)await conn.BeginTransactionAsync();
try
{
    await using var c1 = new SqlCommand("UPDATE dbo.Employees SET Salary = Salary + 100 WHERE EmployeeId = @id", conn, tx);
    c1.Parameters.Add("@id", SqlDbType.Int).Value = 2;
    await c1.ExecuteNonQueryAsync();
    await tx.CommitAsync();
}
catch (SqlException ex) when (ex.Number == 1205)   // deadlock victim → retry whole unit of work
{
    await tx.RollbackAsync();
    throw;
}
```

### 17.6 [Medium] Dapper: common usage — queries, parameters, multi-mapping, multiple result sets, TVP, stored procs.

**Answer**
Dapper adds extension methods on `IDbConnection`. Pass parameters as an anonymous object (parameterized automatically), map to POCOs by column name (alias columns as needed), use `splitOn` for joins, `QueryMultiple` for several result sets, `DynamicParameters` for OUTPUT params, and `AsTableValuedParameter` for TVPs. Use `buffered: false` for streaming large results.

**Explanation**
Dapper is thin: you control the SQL and performance, but there's no change tracking, migrations or LINQ. Common trap: `WHERE Id IN @ids` is expanded by Dapper into individual parameters (limit 2100 SQL Server parameters) — use a TVP/`STRING_SPLIT`/`OPENJSON` for large lists. Enum/DateTime/`DateOnly` mappings may need type handlers.

**Code**
```csharp
using Dapper;

public record EmpDto(int EmployeeId, string Name, decimal Salary);
public class Dept { public int DeptId { get; set; } public string DeptName { get; set; } = ""; public List<EmpDto> Emps { get; set; } = new(); }

await using var conn = new SqlConnection(cs);

// 1) Query with parameters
var emps = (await conn.QueryAsync<EmpDto>(
    "SELECT EmployeeId, Name, Salary FROM dbo.Employees WHERE DeptId = @DeptId AND Salary >= @Min",
    new { DeptId = 1, Min = 6500m })).ToList();                 // Asha, Bram

// 2) Single row / scalar / execute
var one   = await conn.QuerySingleOrDefaultAsync<EmpDto>("SELECT EmployeeId, Name, Salary FROM dbo.Employees WHERE EmployeeId = @id", new { id = 3 });
var count = await conn.ExecuteScalarAsync<int>("SELECT COUNT(*) FROM dbo.Employees");
var rows  = await conn.ExecuteAsync("UPDATE dbo.Employees SET Salary = Salary * 1.05 WHERE DeptId = @d", new { d = 2 });

// 3) Multi-mapping (JOIN) with splitOn
var lookup = new Dictionary<int, Dept>();
await conn.QueryAsync<Dept, EmpDto, Dept>(
    @"SELECT d.DeptId, d.DeptName, e.EmployeeId, e.Name, e.Salary
      FROM dbo.Departments d JOIN dbo.Employees e ON e.DeptId = d.DeptId",
    (d, e) => { if (!lookup.TryGetValue(d.DeptId, out var dept)) lookup[d.DeptId] = dept = d; dept.Emps.Add(e); return dept; },
    splitOn: "EmployeeId");

// 4) Multiple result sets
using var multi = await conn.QueryMultipleAsync(
    "SELECT * FROM dbo.Departments; SELECT TOP (3) * FROM dbo.Orders ORDER BY OrderId;");
var depts  = (await multi.ReadAsync<Dept>()).ToList();
var orders = (await multi.ReadAsync()).ToList();

// 5) Stored proc with OUTPUT + return value
var p = new DynamicParameters();
p.Add("@DeptId", 1); p.Add("@MinSalary", 0m);
p.Add("@Headcount", dbType: DbType.Int32, direction: ParameterDirection.Output);
p.Add("@rc", dbType: DbType.Int32, direction: ParameterDirection.ReturnValue);
await conn.ExecuteAsync("dbo.usp_GetDeptStats", p, commandType: CommandType.StoredProcedure);
int headcount = p.Get<int>("@Headcount");                       // 4

// 6) TVP
var dt = new DataTable(); dt.Columns.Add("Id", typeof(int)); dt.Rows.Add(1); dt.Rows.Add(3);
var picked = await conn.QueryAsync<EmpDto>(
    "dbo.usp_GetEmployeesByIds", new { Ids = dt.AsTableValuedParameter("dbo.IdList") },
    commandType: CommandType.StoredProcedure);                  // Asha, Chen
```

**Gotchas / follow-ups**
- `Query<T>` buffers by default (full list); `buffered: false` streams but ties up the connection.
- Dapper doesn't open/close for you if you pass an already open connection; if closed, it opens and closes it for the call.
- Property names must match column names (case-insensitive); use `AS` aliases or `DefaultTypeMap.MatchNamesWithUnderscores = true`.
- Anonymous string parameters default to `nvarchar(4000)`; use `DbString` (`IsAnsi = true, Length = n`) for `varchar` columns to keep index seeks.

**Interview traps**
- `AddWithValue` with strings → `nvarchar` vs `varchar` column → implicit conversion, index scan.
- Not disposing connections → pool exhaustion.
- Reading OUTPUT params before the reader is closed.
- Dapper `IN @ids` limit of ~2100 parameters.
- `SqlBulkCopy` bypasses triggers/constraints by default and maps by ordinal unless mappings given.

**Cheat sheet**
- `using`/`await using` for connection, command, reader; pool keyed by connection string.
- Parameters: typed `SqlParameter` / Dapper anonymous object; never concatenate.
- Reader: forward-only, streaming; Scalar = first cell; NonQuery = affected rows.
- Bulk: SqlBulkCopy (+ColumnMappings, TableLock) ; small sets: TVP (`SqlDbType.Structured`).
- Dapper: `Query/Execute/QueryMultiple/splitOn/DynamicParameters/AsTableValuedParameter`.
- Retry transient errors (1205, -2, 40613…) with idempotent operations.
