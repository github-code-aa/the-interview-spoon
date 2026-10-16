# D01 - SQL Server Fundamentals & Joins

**How to use:** Read the *Study notes* of a topic first, then cover the answers and try each ladder question (Easy -> Tricky) out loud before reading the **Answer**.
Run the **Code** blocks against the sample schema below (SQL Server 2019+; `OFFSET/FETCH`, `STRING_AGG` etc. need 2012/2017+).

## Sample schema (reused by all examples)

```sql
CREATE TABLE Departments (
    DeptId   INT           NOT NULL PRIMARY KEY,
    DeptName NVARCHAR(50)  NOT NULL
);
CREATE TABLE Employees (
    EmpId     INT           NOT NULL PRIMARY KEY,
    Name      NVARCHAR(50)  NOT NULL,
    Salary    DECIMAL(10,2) NOT NULL,
    DeptId    INT           NULL REFERENCES Departments(DeptId),
    ManagerId INT           NULL REFERENCES Employees(EmpId),
    HireDate  DATE          NOT NULL
);
CREATE TABLE Orders (
    OrderId   INT           NOT NULL PRIMARY KEY,
    EmpId     INT           NULL REFERENCES Employees(EmpId),
    OrderDate DATE          NOT NULL,
    Amount    DECIMAL(10,2) NOT NULL
);

INSERT INTO Departments VALUES (1,'Engineering'),(2,'Sales'),(3,'HR');   -- HR has no employees

INSERT INTO Employees VALUES
 (1,'Asha', 9000,    1, NULL,'2020-01-15'),
 (2,'Ben',  7000,    1, 1,   '2021-03-01'),
 (3,'Chen', 7000,    2, 1,   '2021-06-10'),
 (4,'Dev',  5000,    2, 3,   '2022-02-20'),
 (5,'Esha', 6000,    NULL, 4,'2023-05-05'),   -- no department; earns more than her manager (Dev)
 (6,'Farid',8000,    1, 1,   '2022-09-09');

INSERT INTO Orders VALUES
 (1,2,   '2024-01-05',100.00),
 (2,2,   '2024-01-20',250.00),
 (3,3,   '2024-02-02', 75.50),
 (4,4,   '2024-02-15',300.00),
 (5,NULL,'2024-03-01', 50.00);               -- order with no employee
```

Facts to remember: employees 1, 5, 6 have **no orders**; department 3 (HR) has **no employees**; Esha has **NULL DeptId**; order 5 has **NULL EmpId**.

## Table of contents

1. SQL categories (DDL / DML / DQL / DCL / TCL)
2. Data types (varchar vs nvarchar, datetime2, decimal, money, uniqueidentifier, bit)
3. Logical query processing order
4. Joins (INNER / LEFT / RIGHT / FULL / CROSS / SELF, semi & anti joins)
5. GROUP BY, HAVING, aggregates, COUNT, DISTINCT (and WHERE vs HAVING)
6. Subqueries (correlated, EXISTS vs IN vs JOIN, NOT IN trap)
7. CTEs (and temp tables vs table variables vs CTEs)
8. UNION / UNION ALL / INTERSECT / EXCEPT
9. NULL handling (three-valued logic, ISNULL vs COALESCE, NULLIF)
10. ORDER BY and OFFSET/FETCH pagination
11. String/date functions, collation, case sensitivity
12. Constraints (PK, FK, UNIQUE, CHECK, DEFAULT) and cascading actions
13. IDENTITY, SEQUENCE, GUID keys
14. Normalization (1NF-BCNF) and denormalization
15. DELETE vs TRUNCATE vs DROP
16. OUTPUT, MERGE and upsert patterns
17. Classic interview queries
18. SQL injection basics
19. SQL Server editions and architecture overview

---

## Topic 1: SQL categories (DDL / DML / DQL / DCL / TCL)

**Study notes**

SQL statements are grouped by purpose. Microsoft docs do not formally define "DQL", but interviewers use it for `SELECT`.

| Category | Meaning | Statements |
|---|---|---|
| DDL | Defines/changes structure | `CREATE`, `ALTER`, `DROP`, `TRUNCATE` (commonly grouped here), `CREATE INDEX` |
| DML | Changes data | `INSERT`, `UPDATE`, `DELETE`, `MERGE` |
| DQL | Reads data | `SELECT` |
| DCL | Permissions | `GRANT`, `DENY`, `REVOKE` |
| TCL | Transactions | `BEGIN TRAN`, `COMMIT`, `ROLLBACK`, `SAVE TRAN` |

In SQL Server, DDL is **transactional** (you can roll back a `CREATE TABLE` inside a transaction), unlike Oracle/MySQL where DDL auto-commits.

### 1.1 [Easy] What are DDL, DML, DQL, DCL and TCL?
**Answer**
DDL defines schema objects (`CREATE/ALTER/DROP`), DML modifies rows (`INSERT/UPDATE/DELETE/MERGE`), DQL queries data (`SELECT`), DCL controls permissions (`GRANT/DENY/REVOKE`) and TCL controls transactions (`BEGIN TRAN/COMMIT/ROLLBACK`).

**Explanation**
The split helps reason about what a statement affects: structure, data, access or transaction boundaries. `SELECT ... INTO` is a hybrid: it queries but also creates a table.

**Code**
```sql
CREATE TABLE Demo (Id INT);                -- DDL
INSERT INTO Demo VALUES (1);               -- DML
SELECT * FROM Demo;                        -- DQL
GRANT SELECT ON Demo TO public;            -- DCL
BEGIN TRAN; DELETE Demo; ROLLBACK;         -- TCL
DROP TABLE Demo;                           -- DDL
```

### 1.2 [Follow-up] Which category does TRUNCATE belong to, and why is that debated?
**Answer**
`TRUNCATE TABLE` is usually classed as DDL because it deallocates pages and resets identity rather than deleting row by row. It still removes data, so some classify it as DML-like.

**Explanation**
Behaviourally it is minimally logged (page deallocations are logged, not each row), requires `ALTER` permission on the table, and cannot have a `WHERE`. See Topic 15.

**Code**
```sql
TRUNCATE TABLE Orders_Archive;   -- needs ALTER permission, no WHERE allowed
```

### 1.3 [Medium] Can DDL be rolled back in SQL Server?
**Answer**
Yes. SQL Server DDL participates in transactions, so a `CREATE TABLE`, `ALTER TABLE` or `TRUNCATE` inside an explicit transaction is undone by `ROLLBACK`.

**Explanation**
This enables transactional schema migrations. Exceptions: a few statements cannot run inside a transaction (e.g. `CREATE DATABASE`, `ALTER DATABASE` for many options, `BACKUP`).

**Code**
```sql
BEGIN TRAN;
    CREATE TABLE T1 (Id INT);
    ALTER TABLE Employees ADD Temp1 INT NULL;
ROLLBACK;                      -- both the table and the column vanish
SELECT OBJECT_ID('T1');   -- NULL: table was never created
```

### 1.4 [Hard] What is the difference between DENY, REVOKE and GRANT, and which wins?
**Answer**
`GRANT` gives a permission, `DENY` explicitly forbids it, `REVOKE` removes a previous GRANT or DENY (returns to "not set"). `DENY` overrides `GRANT`, including permissions inherited via roles.

**Explanation**
Permissions are evaluated as the union of grants across user + roles, minus any DENY. Exception: `sysadmin` members bypass permission checks.

**Code**
```sql
GRANT SELECT ON Employees TO ReportRole;
DENY  SELECT ON Employees TO Intern;      -- Intern also in ReportRole -> still denied
REVOKE SELECT ON Employees FROM Intern;   -- removes the DENY; Intern now gets access via ReportRole
```

**Gotchas / follow-ups**
- Ownership chaining lets a stored procedure read tables the caller has no direct rights on - a common least-privilege pattern.
- `REVOKE` does not equal `DENY`.

**Interview traps**
- Saying DDL cannot be rolled back (true for MySQL/Oracle, not SQL Server).
- Calling `SELECT` "DML" - acceptable in some books, but state your definition.
- Forgetting `TRUNCATE` needs `ALTER` rather than `DELETE` permission.
- Confusing `REVOKE` with `DENY`.

**Cheat sheet**
- DDL = structure, DML = data change, DQL = SELECT, DCL = permissions, TCL = transactions.
- SQL Server DDL is transactional.
- DENY beats GRANT; REVOKE clears either.

---

## Topic 2: Data types

**Study notes**

| Type | Storage | Notes |
|---|---|---|
| `varchar(n)` | 1 byte/char (code-page based), max 8000 or `max` | Non-Unicode; with UTF-8 collations (2019+) can hold Unicode |
| `nvarchar(n)` | 2 bytes/char (UTF-16), n up to 4000 or `max` | Unicode; literals need `N'...'` |
| `datetime` | 8 bytes, 1753+, ~3.33 ms precision | Legacy |
| `datetime2(p)` | 6-8 bytes, 0001-9999, up to 100 ns | Preferred; `p` 0-7 |
| `date`, `time`, `datetimeoffset` | | Offset-aware = `datetimeoffset` |
| `decimal(p,s)` / `numeric` | 5-17 bytes | Exact |
| `float`/`real` | 8/4 bytes | Approximate |
| `money` | 8 bytes, 4 decimals | Exact-ish but rounding quirks |
| `uniqueidentifier` | 16 bytes | GUID |
| `bit` | 1 bit (packed up to 8 per byte) | 0, 1, NULL |

### 2.1 [Easy] What is the difference between varchar and nvarchar?
**Answer**
`varchar` stores non-Unicode text using one byte per character (per code page); `nvarchar` stores UTF-16 Unicode using two bytes per character (surrogate pairs use four). Use `nvarchar` for multilingual data, `varchar` for ASCII-only data like codes.

**Explanation**
`nvarchar` doubles storage and index key size. Mixing them causes implicit conversion: comparing a `varchar` column with an `nvarchar` parameter makes SQL Server convert the *column* (data type precedence) under SQL collations, which can prevent index seeks. EF Core maps `string` to `nvarchar` by default for this reason you may need `IsUnicode(false)`.

**Code**
```sql
SELECT DATALENGTH('abc')  AS v,   -- 3
       DATALENGTH(N'abc') AS nv;  -- 6
SELECT N'日本語' AS ok, '日本語' AS lossy;  -- lossy becomes ??? on non-Japanese code pages
```

### 2.2 [Follow-up] Why prefer datetime2 over datetime?
**Answer**
`datetime2` has a wider range (year 1 to 9999), higher precision (100 ns), configurable storage (6-8 bytes) and is ANSI/ISO compliant. `datetime` rounds to .000/.003/.007 ms and is only 1753+.

**Explanation**
`datetime` rounding causes surprises: `'23:59:59.999'` rounds up to the next day. `datetime2(3)` uses 7 bytes with ms precision versus 8 bytes for `datetime`, so it is also smaller.

**Code**
```sql
SELECT CAST('2024-01-01 23:59:59.999' AS datetime)      AS dt,   -- 2024-01-02 00:00:00.000
       CAST('2024-01-01 23:59:59.999' AS datetime2(3))  AS dt2;  -- 2024-01-01 23:59:59.999
```

### 2.3 [Medium] decimal vs float vs money - which for currency?
**Answer**
Use `decimal(p,s)` for currency. `float` is binary approximate and gives rounding errors; `money` is exact to four decimals but intermediate calculations (division/multiplication) lose precision and it is not portable.

**Explanation**
`money` stores as a scaled integer; `money / money` truncates to four places before multiplying, which can yield wrong results. `decimal` rules: result precision/scale of arithmetic is derived by documented formulas and can be truncated to scale 6 minimum when precision exceeds 38.

**Code**
```sql
SELECT CAST(0.1 AS float) + CAST(0.2 AS float) AS f,            -- 0.30000000000000004 (shown as 0.3 in SSMS)
       CASE WHEN CAST(0.1 AS float)+CAST(0.2 AS float) = 0.3 THEN 'eq' ELSE 'ne' END AS cmp,  -- ne
       CAST(0.1 AS decimal(5,2)) + CAST(0.2 AS decimal(5,2)) AS d;  -- 0.30

DECLARE @m1 money = 1, @m2 money = 3;
SELECT @m1/@m2*@m2 AS money_res;   -- 0.9999 (1/3 truncated to 0.3333 first)
```

### 2.4 [Medium] uniqueidentifier: when to use, and what is the cost as clustered key?
**Answer**
A `uniqueidentifier` is a 16-byte GUID, good for globally unique IDs generated client-side or across systems. As a clustered key, random `NEWID()` values cause page splits and fragmentation; `NEWSEQUENTIALID()` (default constraint only) mitigates this.

**Explanation**
16 bytes is copied into every nonclustered index. `NEWSEQUENTIALID()` is sequential per machine restart window - it can only be used in a `DEFAULT`. See Topic 13.

**Code**
```sql
CREATE TABLE Tokens (
    Id    UNIQUEIDENTIFIER NOT NULL DEFAULT NEWSEQUENTIALID() PRIMARY KEY,
    Label NVARCHAR(50)
);
SELECT NEWID();    -- random each call
```

### 2.5 [Tricky] bit pitfalls and implicit conversion gotchas
**Answer**
`bit` accepts 0, 1 and NULL; any non-zero number converts to 1, and the strings `'TRUE'`/`'FALSE'` convert. There is no Boolean type in T-SQL, so you cannot write `WHERE IsActive` - you must write `WHERE IsActive = 1`.

**Explanation**
Predicates are not values: `SELECT 1 = 1` is invalid; use `CASE`. Another trap: data type precedence - in `'5' + 5` the string converts to int (result 10), not concatenation. `varchar` without length defaults to 1 in a declaration/`CAST` can be 30: `DECLARE @s varchar` is 1 char; `CAST(x AS varchar)` is 30 chars.

**Code**
```sql
DECLARE @b bit = 5;                 SELECT @b;                 -- 1
SELECT CASE WHEN 1 = 1 THEN 1 ELSE 0 END AS IsTrue;
SELECT '5' + 5;                     -- 10 (int addition)
DECLARE @s varchar = 'hello';       SELECT @s;                 -- 'h'
SELECT CAST('hello world, this is long and past thirty' AS varchar);  -- truncated to 30 chars
```

**Gotchas / follow-ups**
- `varchar(max)`/`nvarchar(max)` values > 8000 bytes are stored off-row (LOB) and cannot be an index key.
- Implicit conversion warning in execution plans = scan risk.

**Interview traps**
- `N'...'` prefix forgotten -> data loss to `?`.
- Using `float` for money.
- `datetime` rounding `.999` to next day.
- `varchar` with no length in `DECLARE` is 1 char.
- `BETWEEN` on `datetime` end-of-day boundaries (use `>= start AND < next day`).

**Cheat sheet**
- Money -> `decimal(19,4)` or appropriate.
- Dates -> `datetime2`/`date`/`datetimeoffset`.
- Unicode -> `nvarchar` + `N''`.
- GUID clustered key -> `NEWSEQUENTIALID()` or avoid clustering on it.
- No boolean type; `bit` is 0/1/NULL.

---

## Topic 3: Logical query processing order

**Study notes**

Written order differs from logical (conceptual) evaluation order:

```
 1. FROM  (+ JOINs, APPLY)     -> build the row source
 2. WHERE                       -> filter rows
 3. GROUP BY                    -> form groups
 4. HAVING                      -> filter groups
 5. SELECT (expressions, aliases, window functions)
 6. DISTINCT
 7. ORDER BY                    -> may use SELECT aliases
 8. TOP / OFFSET-FETCH
```

Consequences: `WHERE` cannot see `SELECT` aliases; `ORDER BY` can. Window functions run at the SELECT step, so they cannot appear in `WHERE` (wrap in a CTE/subquery). This is *logical* order - the optimizer may physically reorder as long as results are the same.

### 3.1 [Easy] In what order does SQL Server logically process a SELECT query?
**Answer**
FROM (with joins), WHERE, GROUP BY, HAVING, SELECT, DISTINCT, ORDER BY, then TOP/OFFSET-FETCH.

**Explanation**
The order explains alias visibility and why aggregates are illegal in WHERE.

**Code**
```sql
SELECT DeptId, COUNT(*) AS Cnt          -- 5
FROM Employees                          -- 1
WHERE Salary > 5000                     -- 2
GROUP BY DeptId                         -- 3
HAVING COUNT(*) > 1                     -- 4
ORDER BY Cnt DESC;                      -- 7 (alias allowed)
-- DeptId Cnt
-- 1      3
```

### 3.2 [Follow-up] Why does this query fail: SELECT Salary*12 AS Annual ... WHERE Annual > 100000?
**Answer**
`WHERE` is evaluated before `SELECT`, so the alias `Annual` does not exist yet. Repeat the expression, or use a CTE/`CROSS APPLY` to name it first.

**Explanation**
`ORDER BY` is the only clause evaluated after `SELECT`, so it can use aliases. `GROUP BY` also cannot use aliases in T-SQL.

**Code**
```sql
-- Fails: Invalid column name 'Annual'
-- SELECT Name, Salary*12 AS Annual FROM Employees WHERE Annual > 90000;

SELECT e.Name, x.Annual
FROM Employees e
CROSS APPLY (SELECT e.Salary*12 AS Annual) x
WHERE x.Annual > 90000;
-- Asha 108000, Farid 96000
```

### 3.3 [Medium] Where does TOP fit, and what does TOP without ORDER BY mean?
**Answer**
`TOP` is applied last, after `ORDER BY`. Without `ORDER BY`, which rows are returned is non-deterministic - it depends on the plan.

**Explanation**
`TOP (n) WITH TIES` returns extra rows that tie with the last ordering value and requires `ORDER BY`. `TOP` with `PERCENT` rounds up.

**Code**
```sql
SELECT TOP (2) WITH TIES Name, Salary
FROM Employees
ORDER BY Salary DESC;
-- Asha 9000, Farid 8000   (no tie at position 2)
SELECT TOP (3) WITH TIES Name, Salary FROM Employees ORDER BY Salary DESC;
-- Asha, Farid, Ben 7000, Chen 7000  -> 4 rows because of tie
```

### 3.4 [Hard] Why can't you use a window function in WHERE, and how do you filter on it?
**Answer**
Window functions are computed in the SELECT phase, after WHERE/GROUP BY/HAVING. To filter on one, compute it in a CTE or derived table and filter in the outer query.

**Explanation**
Window functions see the rows *after* WHERE/GROUP/HAVING, which is also why `ROW_NUMBER() OVER (...)` ranks only filtered rows. (SQL Server has no `QUALIFY` clause.)

**Code**
```sql
WITH r AS (
  SELECT Name, DeptId, Salary,
         ROW_NUMBER() OVER (PARTITION BY DeptId ORDER BY Salary DESC) AS rn
  FROM Employees
)
SELECT * FROM r WHERE rn = 1;
-- Asha(1) 9000, Chen(2) 7000, Esha(NULL) 6000
```

### 3.5 [Tricky] ORDER BY in a subquery/view - why does it error and what about TOP?
**Answer**
`ORDER BY` is not allowed in views, inline functions, derived tables, subqueries and CTEs unless `TOP`, `OFFSET` or `FOR XML` is also specified, because tables/sets are unordered. With `TOP`, the order only defines *which* rows, not the output order of the outer query.

**Explanation**
Even `SELECT TOP 100 PERCENT ... ORDER BY` in a view is not guaranteed to order output; the optimizer ignores it. Only the outermost `ORDER BY` guarantees order.

**Code**
```sql
-- Error: ORDER BY invalid in views unless TOP/OFFSET/FOR XML specified
-- CREATE VIEW vBad AS SELECT * FROM Employees ORDER BY Salary;
CREATE VIEW vTop AS SELECT TOP (3) * FROM Employees ORDER BY Salary DESC;
SELECT * FROM vTop ORDER BY Name;   -- outer ORDER BY decides final order
```

**Gotchas / follow-ups**
- `SELECT DISTINCT ... ORDER BY col_not_in_select` errors (ORDER BY items must appear in the select list).
- Optimizer reordering is allowed only if semantics are preserved.

**Interview traps**
- "WHERE can use aliases" - no.
- Assuming a view's ORDER BY orders results.
- Assuming `TOP` without `ORDER BY` gives "first" rows.
- Believing logical order equals physical execution order.

**Cheat sheet**
- FROM -> WHERE -> GROUP BY -> HAVING -> SELECT -> DISTINCT -> ORDER BY -> TOP.
- Aliases visible only in ORDER BY.
- Windows functions: filter via CTE.
- Only the outermost ORDER BY guarantees order.

## Topic 4: Joins

**Study notes**

```
 INNER      : A ∩ B                 matching rows only
 LEFT       : all A + matches in B  (NULLs for unmatched B)
 RIGHT      : all B + matches in A
 FULL       : A ∪ B                 unmatched on either side kept
 CROSS      : A x B                 every combination (no ON)
 SELF       : table joined to itself via aliases
 Semi-join  : rows of A that HAVE a match  (EXISTS / IN)  - no duplication
 Anti-join  : rows of A with NO match     (NOT EXISTS / LEFT JOIN ... IS NULL)
```

Physical join algorithms the optimizer chooses: **Nested Loops** (small outer, indexed inner), **Hash Match** (large unsorted inputs), **Merge Join** (both inputs sorted on the key). Logical join type (LEFT, INNER) is different from physical operator.

### 4.1 [Easy] Explain INNER, LEFT, RIGHT, FULL and CROSS JOIN.
**Answer**
INNER returns only rows matching in both tables; LEFT keeps all left rows with NULLs for missing right matches; RIGHT is the mirror; FULL keeps unmatched rows from both; CROSS returns the Cartesian product.

**Explanation**
With the sample data, `Employees INNER JOIN Departments` returns 5 rows (Esha has NULL DeptId, so she is dropped), `LEFT` returns 6, `Departments LEFT JOIN Employees` returns 6 (5 matches plus HR with NULL employee), `FULL` returns 7 (5 matches + Esha + HR), `CROSS` returns 3 x 6 = 18.

**Code**
```sql
SELECT e.Name, d.DeptName
FROM Employees e
FULL JOIN Departments d ON d.DeptId = e.DeptId;
-- Asha Engineering | Ben Engineering | Farid Engineering
-- Chen Sales | Dev Sales | Esha NULL | NULL HR
```

### 4.2 [Follow-up] How do you do a self join - e.g. employees with their manager names?
**Answer**
Alias the same table twice and join the manager key to the employee key. Use LEFT JOIN so the top-level employee (NULL manager) is kept.

**Explanation**
INNER would silently drop Asha (ManagerId NULL). Hierarchies of arbitrary depth need a recursive CTE (Topic 7).

**Code**
```sql
SELECT e.Name AS Employee, m.Name AS Manager
FROM Employees e
LEFT JOIN Employees m ON m.EmpId = e.ManagerId;
-- Asha NULL | Ben Asha | Chen Asha | Dev Chen | Esha Dev | Farid Asha
```

### 4.3 [Medium] How do you find rows with no match (anti-join) and rows with a match without duplicates (semi-join)?
**Answer**
Anti-join: `NOT EXISTS` (preferred) or `LEFT JOIN ... WHERE right.key IS NULL`. Semi-join: `EXISTS` (or `IN`), which returns each left row at most once even if many right rows match.

**Explanation**
An `INNER JOIN` to find employees with orders would duplicate Ben (2 orders), requiring `DISTINCT`. `EXISTS` stops at the first match. The optimizer usually plans `EXISTS`/`NOT EXISTS` as Left Semi / Left Anti Semi joins.

**Code**
```sql
-- Semi: employees who have orders
SELECT e.Name FROM Employees e
WHERE EXISTS (SELECT 1 FROM Orders o WHERE o.EmpId = e.EmpId);
-- Ben, Chen, Dev

-- Anti: employees with no orders
SELECT e.Name FROM Employees e
WHERE NOT EXISTS (SELECT 1 FROM Orders o WHERE o.EmpId = e.EmpId);
-- Asha, Esha, Farid

-- Anti via LEFT JOIN
SELECT e.Name FROM Employees e
LEFT JOIN Orders o ON o.EmpId = e.EmpId
WHERE o.OrderId IS NULL;       -- test a NOT NULL column of the right table (the key)
```

### 4.4 [Hard] Outer join: filter in ON vs WHERE - what's the difference?
**Answer**
In an outer join, `ON` decides which right-side rows *match* (unmatched left rows are still preserved), whereas `WHERE` runs afterwards and removes rows; a WHERE condition on the right table's columns turns a LEFT JOIN into an effective INNER JOIN (NULL fails the predicate).

**Explanation**
Put predicates about the *optional* side in ON to keep the outer rows; put predicates about the preserved side in WHERE. The WHERE `IS NULL` anti-join pattern is the intentional exception.

**Code**
```sql
-- Filter in ON: all 3 departments kept
SELECT d.DeptName, e.Name
FROM Departments d
LEFT JOIN Employees e ON e.DeptId = d.DeptId AND e.Salary > 7000;
-- Engineering Asha | Engineering Farid | Sales NULL | HR NULL

-- Filter in WHERE: Sales & HR vanish (NULL > 7000 is UNKNOWN)
SELECT d.DeptName, e.Name
FROM Departments d
LEFT JOIN Employees e ON e.DeptId = d.DeptId
WHERE e.Salary > 7000;
-- Engineering Asha | Engineering Farid
```

**Gotchas / follow-ups**
- For an INNER join, ON vs WHERE gives the same result (optimizer treats them equivalently).
- `WHERE e.Salary > 7000 OR e.EmpId IS NULL` would restore unmatched departments, but also changes semantics - prefer ON.
- Multi-table outer join chains: a later INNER JOIN on the optional table also nullifies the outer join.

### 4.5 [Tricky] NULL keys in joins, join explosion and FULL JOIN with WHERE
**Answer**
`NULL = NULL` is UNKNOWN so NULL keys never match in joins (Esha has no department row; order 5 has no employee). Joining on non-unique columns multiplies rows, silently inflating aggregates.

**Explanation**
To match NULLs you must write `a.k = b.k OR (a.k IS NULL AND b.k IS NULL)` (non-SARGable, slow) or use `INTERSECT`-style tricks. Aggregates after a one-to-many join (e.g. SUM of employee salary after joining orders) double count - aggregate before joining, or use `EXISTS`.

**Code**
```sql
-- Join explosion: Ben's salary is counted twice (two orders)
SELECT SUM(e.Salary) FROM Employees e JOIN Orders o ON o.EmpId = e.EmpId;
-- 7000+7000+7000+5000 = 26000 (not 19000)

-- Correct: semi-join
SELECT SUM(Salary) FROM Employees e
WHERE EXISTS (SELECT 1 FROM Orders o WHERE o.EmpId = e.EmpId);   -- 19000

-- NULL-safe equality pattern (non-SARGable; shown for syntax)
-- ON a.k = b.k OR (a.k IS NULL AND b.k IS NULL)
```

**Interview traps**
- WHERE on the right table after LEFT JOIN = accidental INNER JOIN.
- Counting `COUNT(*)` after LEFT JOIN yields 1 for unmatched rows; use `COUNT(o.OrderId)`.
- Forgetting `ON` in old comma syntax creates a Cartesian product.
- RIGHT JOIN is rarely used; swap tables and use LEFT.
- `JOIN` alone means INNER JOIN.

**Cheat sheet**
- INNER = matches; LEFT/RIGHT = preserve one side; FULL = both; CROSS = product.
- Optional-side filter -> ON; preserved-side filter -> WHERE.
- Anti-join: NOT EXISTS; semi-join: EXISTS.
- NULL never equals NULL in a join.
- Physical joins: Nested Loops, Hash, Merge.

---

## Topic 5: GROUP BY, HAVING, aggregates, COUNT, DISTINCT (and WHERE vs HAVING)

**Study notes**

- `GROUP BY` collapses rows into groups; every SELECT item must be a grouped column or an aggregate.
- Aggregates ignore NULLs (except `COUNT(*)`): `SUM`, `AVG`, `MIN`, `MAX`, `COUNT(col)`.
- `WHERE` filters rows **before** grouping (cannot use aggregates); `HAVING` filters groups **after**.
- NULLs form a single group in `GROUP BY` and are treated as equal by `DISTINCT`/`GROUP BY` (unlike `=`).
- Extensions: `GROUPING SETS`, `ROLLUP`, `CUBE`.

| Expression | Counts |
|---|---|
| `COUNT(*)` | all rows |
| `COUNT(col)` | non-NULL values of col |
| `COUNT(DISTINCT col)` | distinct non-NULL values |
| `COUNT(1)` | same as `COUNT(*)` |

### 5.1 [Easy] COUNT(*) vs COUNT(column) vs COUNT(DISTINCT column)?
**Answer**
`COUNT(*)` counts rows, `COUNT(col)` counts non-NULL values, and `COUNT(DISTINCT col)` counts distinct non-NULL values.

**Explanation**
`COUNT(1)` and `COUNT(*)` are identical in cost and result. Aggregates other than `COUNT(*)` print a "Null value is eliminated by an aggregate" warning.

**Code**
```sql
SELECT COUNT(*) AS AllRows,            -- 6
       COUNT(DeptId) AS NonNull,       -- 5
       COUNT(DISTINCT DeptId) AS Dist  -- 2
FROM Employees;
```

### 5.2 [Follow-up] What is the difference between WHERE and HAVING?
**Answer**
WHERE filters individual rows before grouping and can't use aggregates; HAVING filters groups after aggregation and can use aggregates. Filter on non-aggregated columns in WHERE for performance, since it reduces rows before grouping.

**Explanation**
Moving a non-aggregate condition from HAVING to WHERE is semantically the same but cheaper (and the optimizer often does this itself).

**Code**
```sql
SELECT DeptId, COUNT(*) AS Cnt, AVG(Salary) AS AvgSal
FROM Employees
WHERE HireDate >= '2021-01-01'
GROUP BY DeptId
HAVING COUNT(*) >= 2;
-- DeptId Cnt AvgSal
-- 1      2   7500.00   (Ben, Farid)
-- 2      2   6000.00   (Chen, Dev)
```

### 5.3 [Medium] Why does AVG over a column with NULLs surprise people, and how do GROUP BY and NULL behave?
**Answer**
`AVG` divides by the count of non-NULL values, not rows, so NULLs inflate the average relative to treating them as 0. In `GROUP BY`, all NULLs fall into one group.

**Explanation**
Use `AVG(COALESCE(col,0))` if NULL means zero. Also integer columns: `AVG` of ints returns an int (truncated) - cast to decimal first.

**Code**
```sql
SELECT DeptId, COUNT(*) AS Cnt FROM Employees GROUP BY DeptId;
-- 1 3 | 2 2 | NULL 1

SELECT AVG(DeptId)            AS IntAvg,      -- 1 (int division: (1+1+2+2+1)=7/5=1)
       AVG(CAST(DeptId AS decimal(5,2))) AS DecAvg;  -- 1.400000
```

### 5.4 [Hard] How do you find groups with duplicates and list totals plus a grand total (ROLLUP)?
**Answer**
`GROUP BY key HAVING COUNT(*) > 1` finds duplicates. `ROLLUP` adds subtotal and grand-total rows; `GROUPING()` distinguishes real NULLs from subtotal NULLs.

**Explanation**
`GROUPING SETS` gives explicit control; `CUBE` returns all combinations. `GROUPING(col)` returns 1 when the NULL in that column is a super-aggregate row.

**Code**
```sql
SELECT DeptId, SUM(Salary) AS Total, GROUPING(DeptId) AS IsTotal
FROM Employees
GROUP BY ROLLUP (DeptId);
-- 1    24000 0
-- 2    12000 0
-- NULL  6000 0   (Esha - real NULL)
-- NULL 42000 1   (grand total)

SELECT Salary, COUNT(*) FROM Employees GROUP BY Salary HAVING COUNT(*) > 1;  -- 7000 2
```

### 5.5 [Tricky] DISTINCT vs GROUP BY, and COUNT(DISTINCT) with multiple columns
**Answer**
For plain de-duplication they are equivalent and typically produce the same plan; GROUP BY is needed when you aggregate. `COUNT(DISTINCT a, b)` is not valid T-SQL - concatenate, use a subquery, or `SELECT COUNT(*) FROM (SELECT DISTINCT a, b ...) x`.

**Explanation**
`DISTINCT` applies to the whole select list, not just the first column - `SELECT DISTINCT a, b` is not distinct on a. Also, a selected column not in GROUP BY and not aggregated errors (SQL Server is strict, unlike old MySQL).

**Code**
```sql
SELECT COUNT(*) FROM (SELECT DISTINCT DeptId, Salary FROM Employees) x;   -- 6
-- distinct pairs: (1,9000) (1,7000) (1,8000) (2,7000) (2,5000) (NULL,6000)
```

**Gotchas / follow-ups**
- `SUM` of an all-NULL (or empty) set returns NULL, `COUNT` returns 0.
- `SUM` of ints overflows at 2^31 - use `SUM(CAST(x AS bigint))`; `COUNT_BIG` for large counts.
- An aggregate without GROUP BY over an empty table still returns one row; with GROUP BY returns none.

**Interview traps**
- Using an aggregate in WHERE.
- Thinking `COUNT(col)` counts NULLs.
- `SELECT *` with GROUP BY.
- Assuming GROUP BY sorts output (it doesn't guarantee order).
- `DISTINCT` hiding a join explosion bug.

**Cheat sheet**
- WHERE = rows pre-group; HAVING = groups post-aggregate.
- Aggregates skip NULLs except `COUNT(*)`.
- NULLs group together.
- `ROLLUP` + `GROUPING()` for subtotals.
- `COUNT(DISTINCT a,b)` isn't supported.

---

## Topic 6: Subqueries (correlated, EXISTS vs IN vs JOIN, NOT IN trap)

**Study notes**

- **Scalar** subquery: one value (must return at most one row or error 512).
- **Multi-valued**: used with `IN`, `ANY/ALL`, `EXISTS`.
- **Derived table**: subquery in FROM.
- **Correlated**: references the outer query; logically evaluated per outer row, but the optimizer usually rewrites it into a join.
- `EXISTS` returns TRUE/FALSE, ignores the select list, and short-circuits.

| | IN | EXISTS | JOIN |
|---|---|---|---|
| NULLs in subquery | `NOT IN` trap | safe | n/a |
| Duplicates | none (semi-join) | none | can multiply rows |
| Return columns from inner | no | no | yes |

### 6.1 [Easy] What is a correlated subquery?
**Answer**
A subquery that references columns from the outer query, so its result depends on the current outer row. It is conceptually re-executed for each outer row.

**Explanation**
Example: employees earning more than their department's average. The optimizer often turns this into a join with an aggregate, so it is not necessarily slow.

**Code**
```sql
SELECT e.Name, e.Salary
FROM Employees e
WHERE e.Salary > (SELECT AVG(e2.Salary) FROM Employees e2 WHERE e2.DeptId = e.DeptId);
-- Asha 9000 (Eng avg 8000) | Chen 7000 (Sales avg 6000)
-- Esha is excluded: her DeptId is NULL, e2.DeptId = NULL matches nothing, AVG = NULL
```

### 6.2 [Follow-up] EXISTS vs IN vs JOIN - which is faster?
**Answer**
For uncorrelated, non-null data they usually produce the same plan, so correctness matters more than speed. `EXISTS` is safest with NULLs, `IN` is readable for small lists, `JOIN` is needed when you need inner columns but can multiply rows.

**Explanation**
Check the execution plan rather than relying on folklore. `EXISTS (SELECT 1 ...)` vs `SELECT *` inside EXISTS makes no difference.

**Code**
```sql
SELECT Name FROM Employees WHERE EmpId IN (SELECT EmpId FROM Orders);                    -- Ben, Chen, Dev
SELECT Name FROM Employees e WHERE EXISTS (SELECT 1 FROM Orders o WHERE o.EmpId=e.EmpId); -- same
SELECT DISTINCT e.Name FROM Employees e JOIN Orders o ON o.EmpId=e.EmpId;                -- same; DISTINCT needed
```

### 6.3 [Medium] Explain the NOT IN with NULL trap.
**Answer**
If the subquery returns any NULL, `x NOT IN (...)` evaluates to UNKNOWN for every non-matching x, so the query returns no rows. Use `NOT EXISTS` or filter NULLs out.

**Explanation**
`x NOT IN (1, NULL)` expands to `x <> 1 AND x <> NULL`; the second term is UNKNOWN, so the whole is never TRUE. Order 5 has a NULL EmpId, triggering this.

**Code**
```sql
SELECT Name FROM Employees WHERE EmpId NOT IN (SELECT EmpId FROM Orders);
-- 0 rows!  (should be Asha, Esha, Farid)

SELECT Name FROM Employees e
WHERE NOT EXISTS (SELECT 1 FROM Orders o WHERE o.EmpId = e.EmpId);   -- Asha, Esha, Farid

SELECT Name FROM Employees
WHERE EmpId NOT IN (SELECT EmpId FROM Orders WHERE EmpId IS NOT NULL); -- same
```

### 6.4 [Hard] Scalar subquery returning more than one row; ANY/ALL; subquery in SELECT list
**Answer**
A scalar subquery returning more than one row raises error 512 ("Subquery returned more than 1 value") unless used with `IN/EXISTS/ANY/ALL`. `= ALL`/`> ALL` compare against every value; `> ANY` equals `> MIN`.

**Explanation**
A scalar subquery returning zero rows yields NULL. Subqueries in the SELECT list are evaluated per row logically; often better rewritten as join/`APPLY`. A runtime error happens only when more than one row is actually returned, which can appear later when data changes - guard with `TOP 1`+`ORDER BY` or aggregates.

**Code**
```sql
-- Employees earning more than everyone in Sales
SELECT Name FROM Employees
WHERE Salary > ALL (SELECT Salary FROM Employees WHERE DeptId = 2);   -- Asha, Farid (>7000)

-- Order count per employee via scalar subquery
SELECT e.Name, (SELECT COUNT(*) FROM Orders o WHERE o.EmpId = e.EmpId) AS OrderCnt
FROM Employees e;
-- Asha 0 | Ben 2 | Chen 1 | Dev 1 | Esha 0 | Farid 0
```

**Gotchas / follow-ups**
- `> ALL` over an empty set is TRUE; over a set containing NULL it is UNKNOWN (no rows).
- `ORDER BY` inside subqueries is disallowed without TOP.

### 6.5 [Tricky] Does a correlated subquery really run row by row, and how does APPLY relate?
**Answer**
Logically yes, physically not necessarily: the optimizer "unnests" many correlated subqueries into joins/semi-joins, but some (e.g. with `TOP` per outer row) remain nested-loops. `CROSS/OUTER APPLY` makes the per-row semantics explicit and permits table-valued subqueries.

**Explanation**
`CROSS APPLY` ~ inner join with a correlated table expression; `OUTER APPLY` ~ left join. Good for "top N per group" and table-valued functions.

**Code**
```sql
SELECT d.DeptName, t.Name, t.Salary
FROM Departments d
OUTER APPLY (SELECT TOP (1) e.Name, e.Salary
             FROM Employees e WHERE e.DeptId = d.DeptId
             ORDER BY e.Salary DESC) t;
-- Engineering Asha 9000 | Sales Chen 7000 | HR NULL NULL
```

**Interview traps**
- `NOT IN` + nullable column.
- Believing correlated subqueries are always slower than joins.
- Scalar subquery returning multiple rows after data changes.
- `IN` list with thousands of literals (plan/parse issues) - use a temp table or TVP.

**Cheat sheet**
- Correlated = references outer query.
- `EXISTS` is NULL-safe and short-circuits.
- Prefer `NOT EXISTS` over `NOT IN`.
- `APPLY` = per-row table expression.
- Scalar subquery: 0 rows -> NULL, >1 rows -> error.

## Topic 7: CTEs (and temp tables vs table variables vs CTEs)

**Study notes**

A **CTE** (`WITH name AS (...)`) is a named, statement-scoped result set. It is *not* materialized by SQL Server - it is inlined like a view, so referencing it twice may execute it twice. Recursive CTEs = anchor member `UNION ALL` recursive member.

| | CTE | `#temp` table | `@table variable` |
|---|---|---|---|
| Scope | single statement | session (or proc) | batch/proc |
| Storage | none (inline) | tempdb | tempdb (not memory-only) |
| Statistics | n/a | yes | no column stats (table-level row count since 2019 deferred compilation) |
| Indexes | n/a | any | only via PK/UNIQUE inline (and inline index syntax since 2014) |
| Rolled back by ROLLBACK | n/a | yes | **no** |
| Reuse many times | recomputed | yes, cheap | yes |

### 7.1 [Easy] What is a CTE and why use it?
**Answer**
A Common Table Expression is a temporary named result set defined with `WITH` that exists for one statement. It improves readability, allows recursion and lets you reference a derived result multiple times by name.

**Explanation**
The statement before `WITH` must end with `;`. CTEs can be the target of `UPDATE/DELETE` (affecting the underlying table) - used for deleting duplicates.

**Code**
```sql
WITH DeptAvg AS (
    SELECT DeptId, AVG(Salary) AS AvgSal FROM Employees GROUP BY DeptId
)
SELECT e.Name, e.Salary, d.AvgSal
FROM Employees e JOIN DeptAvg d ON d.DeptId = e.DeptId
WHERE e.Salary > d.AvgSal;
-- Asha 9000 8000 | Chen 7000 6000
```

### 7.2 [Follow-up] How does a recursive CTE work? Give the org-chart example.
**Answer**
It has an anchor query (the root rows) plus a recursive member joined to the CTE itself, combined with `UNION ALL`. Recursion repeats until the recursive member returns no rows; the default limit is 100 levels (`OPTION (MAXRECURSION n)`, 0 = unlimited).

**Explanation**
Each iteration sees only the rows produced by the previous iteration. Cycles in data (A manages B manages A) cause the recursion limit error - guard with a path column.

**Code**
```sql
WITH Org AS (
    SELECT EmpId, Name, ManagerId, 0 AS Lvl
    FROM Employees WHERE ManagerId IS NULL
    UNION ALL
    SELECT e.EmpId, e.Name, e.ManagerId, o.Lvl + 1
    FROM Employees e JOIN Org o ON e.ManagerId = o.EmpId
)
SELECT Name, Lvl FROM Org ORDER BY Lvl, Name;
-- Asha 0 | Ben 1 | Chen 1 | Farid 1 | Dev 2 | Esha 3
```

### 7.3 [Medium] Is a CTE faster than a subquery or temp table? Is it materialized?
**Answer**
No - a CTE is just syntax; the optimizer expands it inline, so performance equals the equivalent derived table. If you reference it several times, it is re-evaluated each time; a `#temp` table materializes the result once and gets statistics and indexes.

**Explanation**
Use a temp table when the intermediate set is large, reused, or needs indexing, or to break up a complex plan with poor cardinality estimates. Do not assume `WITH` materializes (it does in PostgreSQL <12 and some others, not SQL Server).

**Code**
```sql
SELECT EmpId, Salary INTO #HighPaid FROM Employees WHERE Salary >= 7000;
CREATE CLUSTERED INDEX IX_HP ON #HighPaid (EmpId);
SELECT COUNT(*) FROM #HighPaid;             -- 4
DROP TABLE #HighPaid;
```

### 7.4 [Hard] Temp table vs table variable - differences that matter?
**Answer**
Temp tables have column statistics, can have any indexes and participate in rollback; table variables have no column statistics (the optimizer historically assumed 1 row, improved by table-variable deferred compilation in SQL Server 2019 compat level 150), don't roll back, and cause fewer recompiles. Both live in tempdb.

**Explanation**
Table variables aren't "in memory". Because their data survives `ROLLBACK`, they are handy for logging inside a transaction. Use a table variable for small row counts; use `#temp` for larger sets or when stats matter. Table-valued parameters are table variables passed readonly.

**Code**
```sql
DECLARE @t TABLE (Id INT PRIMARY KEY, Note VARCHAR(20));
BEGIN TRAN;
    INSERT @t VALUES (1,'kept');
    CREATE TABLE #t (Id INT); INSERT #t VALUES (1);
ROLLBACK;
SELECT COUNT(*) FROM @t;    -- 1  (survives rollback)
SELECT OBJECT_ID('tempdb..#t'); -- NULL (the temp table creation rolled back)
```

### 7.5 [Tricky] CTE gotchas: multiple CTEs, UPDATE through a CTE, reuse after the statement
**Answer**
A CTE is visible only to the single statement that immediately follows; you can define several separated by commas, and later ones can reference earlier ones. You can `UPDATE`/`DELETE` through a CTE if it maps to a single base table.

**Explanation**
Using the CTE in a second statement gives "Invalid object name". `ORDER BY` inside a CTE needs `TOP`. The recursive member can't use `TOP`, `DISTINCT`, aggregates or `LEFT JOIN` to the CTE itself.

**Code**
```sql
BEGIN TRAN;   -- demo only: roll back so the sample data stays unchanged
WITH Raise AS (SELECT * FROM Employees WHERE DeptId = 2)
UPDATE Raise SET Salary = Salary * 1.1;      -- updates Chen and Dev in Employees
ROLLBACK;

WITH a AS (SELECT 1 AS n), b AS (SELECT n + 1 AS n FROM a)
SELECT * FROM b;                              -- 2
-- SELECT * FROM a;  -- error: Invalid object name 'a' (CTE scope ended)
```

**Interview traps**
- "CTEs are cached/materialized" - false in SQL Server.
- Forgetting `;` before `WITH`.
- Table variable data survives ROLLBACK.
- Infinite recursion on cyclic data.

**Cheat sheet**
- CTE = inline, single-statement scope.
- Recursive CTE = anchor `UNION ALL` recursive; `MAXRECURSION` default 100.
- `#temp`: stats + indexes + rollback; `@var`: none/no rollback.
- Reused big intermediate -> `#temp`.

---

## Topic 8: UNION / UNION ALL / INTERSECT / EXCEPT

**Study notes**

```
 UNION      : A + B, duplicates removed (sort/hash distinct)
 UNION ALL  : A + B, duplicates kept (cheapest)
 INTERSECT  : rows in both,   distinct
 EXCEPT     : rows in A not B, distinct   (SQL Server's MINUS)
```

Rules: same number of columns, compatible types (by position), column names from the first query, `ORDER BY` only at the very end. Set operators treat **NULLs as equal** (unlike `=`). Precedence: `INTERSECT` binds tighter than `UNION`/`EXCEPT`; use parentheses.

### 8.1 [Easy] UNION vs UNION ALL?
**Answer**
`UNION` removes duplicate rows (extra sort/hash cost); `UNION ALL` concatenates everything and is faster. Use `UNION ALL` unless you need de-duplication.

**Explanation**
Also `UNION` de-duplicates within each side, not just across.

**Code**
```sql
SELECT DeptId FROM Employees
UNION        SELECT DeptId FROM Departments;   -- NULL,1,2,3  (4 rows)
SELECT DeptId FROM Employees
UNION ALL    SELECT DeptId FROM Departments;   -- 9 rows
```

### 8.2 [Follow-up] What do INTERSECT and EXCEPT return, and how do they compare to joins?
**Answer**
`INTERSECT` returns distinct rows present in both inputs; `EXCEPT` returns distinct rows from the left that are not in the right. They compare whole rows and treat NULLs as equal, so they are NULL-safe alternatives to join/`NOT IN`.

**Explanation**
Handy for comparing tables column-by-column without writing `a.c = b.c OR (both NULL)` conditions.

**Code**
```sql
SELECT DeptId FROM Departments EXCEPT SELECT DeptId FROM Employees;    -- 3 (HR has no employees)
SELECT DeptId FROM Employees   EXCEPT SELECT DeptId FROM Departments;  -- NULL (Esha)
SELECT DeptId FROM Departments INTERSECT SELECT DeptId FROM Employees; -- 1, 2
```

### 8.3 [Medium] Column rules: naming, types, ORDER BY
**Answer**
Column count must match; types are matched by position and implicitly converted using data type precedence; names come from the first SELECT; a single `ORDER BY` goes at the end and refers to first-query names/ordinals.

**Explanation**
A type mismatch (e.g. int vs varchar 'abc') raises a conversion error at runtime. Use `CAST` explicitly. To sort sections, add a literal sort key column.

**Code**
```sql
SELECT Name AS Label, 1 AS Sort FROM Employees WHERE EmpId = 1
UNION ALL
SELECT DeptName, 2 FROM Departments WHERE DeptId = 3
ORDER BY Sort;
-- Asha 1 | HR 2
```

### 8.4 [Hard] Precedence and duplicates: what does A UNION B INTERSECT C do?
**Answer**
`INTERSECT` is evaluated before `UNION` and `EXCEPT`, so `A UNION B INTERSECT C` = `A UNION (B INTERSECT C)`. `UNION` and `EXCEPT` have equal precedence, evaluated left to right. Use parentheses to be explicit.

**Explanation**
Mixing `UNION ALL` and `UNION` in a chain: evaluated left to right, so a later `UNION` de-duplicates everything before it too.

**Code**
```sql
SELECT 1 AS n UNION SELECT 2 INTERSECT SELECT 2;     -- 1, 2   : 1 UNION (2 INTERSECT 2)
SELECT 1 AS n UNION ALL SELECT 1 UNION SELECT 1;     -- 1      : the final UNION dedups everything
(SELECT 1 AS n UNION SELECT 2) INTERSECT SELECT 2;   -- 2
```

### 8.5 [Tricky] How do you get INTERSECT ALL / EXCEPT ALL behaviour (keeping duplicates)?
**Answer**
SQL Server does not support `INTERSECT ALL`/`EXCEPT ALL`. Emulate with `ROW_NUMBER() OVER (PARTITION BY cols ORDER BY (SELECT NULL))` added to both sides and then using `INTERSECT/EXCEPT`.

**Explanation**
Numbering duplicates makes each occurrence unique so the set operator matches them pairwise.

**Code**
```sql
-- Number the duplicates so each occurrence becomes a distinct row, then INTERSECT
WITH a AS (SELECT Salary, ROW_NUMBER() OVER (PARTITION BY Salary ORDER BY (SELECT NULL)) AS rn FROM Employees),
     b AS (SELECT 7000 AS Salary, 1 AS rn UNION ALL SELECT 7000, 2 UNION ALL SELECT 7000, 3)
SELECT Salary, rn FROM a
INTERSECT
SELECT Salary, rn FROM b;
-- (7000,1), (7000,2): the multiset intersection keeps two 7000 rows (a has two, b has three)
```

**Interview traps**
- Using `UNION` where `UNION ALL` suffices (hidden sort).
- Assuming INTERSECT precedence is left-to-right.
- Expecting column names from the second query.
- Believing `INTERSECT ALL` exists in T-SQL.

**Cheat sheet**
- UNION dedups; UNION ALL doesn't.
- EXCEPT = A minus B (distinct).
- Set ops treat NULL = NULL.
- INTERSECT binds first.
- One ORDER BY, at the end.

---

## Topic 9: NULL handling (three-valued logic, ISNULL vs COALESCE, NULLIF)

**Study notes**

NULL means *unknown/missing*, not zero or empty string. Comparisons yield TRUE, FALSE or UNKNOWN. `WHERE` keeps only TRUE.

```
 AND : TRUE AND UNKNOWN = UNKNOWN   FALSE AND UNKNOWN = FALSE
 OR  : TRUE OR  UNKNOWN = TRUE      FALSE OR  UNKNOWN = UNKNOWN
 NOT UNKNOWN = UNKNOWN
```

- Use `IS NULL` / `IS NOT NULL`, never `= NULL` (with default `ANSI_NULLS ON`).
- Arithmetic/concatenation with NULL gives NULL (default `CONCAT_NULL_YIELDS_NULL ON`). `CONCAT()` treats NULL as empty string.
- `CHECK` constraints pass on UNKNOWN; `UNIQUE` allows one NULL; `GROUP BY`/`DISTINCT`/set operators treat NULLs as equal.

### 9.1 [Easy] How do you test for NULL, and why does `= NULL` not work?
**Answer**
Use `IS NULL`/`IS NOT NULL`. `col = NULL` evaluates to UNKNOWN, never TRUE, so it returns no rows.

**Explanation**
Even `NULL = NULL` is UNKNOWN. `ANSI_NULLS OFF` would change this, but it is deprecated - do not use.

**Code**
```sql
SELECT Name FROM Employees WHERE DeptId = NULL;     -- 0 rows
SELECT Name FROM Employees WHERE DeptId IS NULL;    -- Esha
```

### 9.2 [Follow-up] ISNULL vs COALESCE?
**Answer**
`ISNULL(a,b)` is T-SQL specific with exactly two args, and its result takes the data type (and length) of the first argument. `COALESCE(a,b,c,...)` is ANSI, takes many args and returns the highest-precedence type of all arguments.

**Explanation**
The truncation difference bites with varchar: `ISNULL(@short, 'longer text')` truncates to @short's length. `COALESCE` is translated into a `CASE`, so subqueries may be evaluated twice. Nullability of the result can also differ (ISNULL result is considered non-nullable when second arg is non-null, useful for computed columns/PKs).

**Code**
```sql
DECLARE @s varchar(2) = NULL;
SELECT ISNULL(@s, 'abcdef')   AS isnull_res,     -- 'ab'  (truncated)
       COALESCE(@s, 'abcdef') AS coalesce_res;   -- 'abcdef'
SELECT COALESCE(NULL, NULL, 3);                   -- 3
```

### 9.3 [Medium] Comparison traps: <>, NOT IN, and filtering with NULLs
**Answer**
`<>` and `NOT IN` exclude rows where the column is NULL because the comparison is UNKNOWN. To include NULL rows, add `OR col IS NULL` or use `ISNULL`/`EXISTS` forms.

**Explanation**
Employees where DeptId <> 1 gives Chen and Dev but silently drops Esha. Wrapping the column in `ISNULL` works but kills index seeks (non-SARGable); prefer `OR col IS NULL`, or `IS DISTINCT FROM` (available in SQL Server 2022+).

**Code**
```sql
SELECT Name FROM Employees WHERE DeptId <> 1;                    -- Chen, Dev
SELECT Name FROM Employees WHERE DeptId <> 1 OR DeptId IS NULL;  -- Chen, Dev, Esha
-- SQL Server 2022+:
SELECT Name FROM Employees WHERE DeptId IS DISTINCT FROM 1;      -- Chen, Dev, Esha
```

### 9.4 [Hard] NULLIF, divide-by-zero, and NULL in aggregates/ordering
**Answer**
`NULLIF(a,b)` returns NULL if a = b, else a - ideal for avoiding divide-by-zero (`x / NULLIF(y,0)`). Aggregates skip NULLs; `ORDER BY` sorts NULLs first ascending (lowest value) and last descending.

**Explanation**
`NULLIF(col,'')` turns empty strings into NULL for `COALESCE` chains. Sorting NULLs last requires `ORDER BY CASE WHEN col IS NULL THEN 1 ELSE 0 END, col`. Note `NULLIF(col,'')` is a common way to turn empty strings into NULL for `COALESCE` chains.

**Code**
```sql
SELECT 10 / NULLIF(0, 0);            -- NULL (no error)
SELECT DeptId FROM Employees ORDER BY DeptId;      -- NULL,1,1,1,2,2
SELECT DeptId FROM Employees
ORDER BY CASE WHEN DeptId IS NULL THEN 1 ELSE 0 END, DeptId;  -- 1,1,1,2,2,NULL
```

### 9.5 [Tricky] NULL surprises: CASE, concatenation, CHECK, UNIQUE, SUM
**Answer**
`CASE WHEN col = NULL` never matches (use `IS NULL`); `'a' + NULL` is NULL; a CHECK constraint evaluating to UNKNOWN is accepted; a UNIQUE constraint allows exactly one NULL (a filtered unique index `WHERE col IS NOT NULL` allows many).

**Explanation**
`SUM` over all-NULL returns NULL while `COUNT(col)` returns 0. `NOT (a = b)` with NULLs is still UNKNOWN. `bit` columns also NULL.

**Code**
```sql
SELECT 'a' + NULL;                    -- NULL
SELECT CONCAT('a', NULL, 'b');        -- 'ab'
SELECT CASE WHEN NULL = NULL THEN 'eq' ELSE 'not eq' END;   -- not eq

CREATE TABLE U (Code VARCHAR(10) NULL UNIQUE, Age INT CHECK (Age > 0));
INSERT U VALUES (NULL, NULL);    -- OK: CHECK is UNKNOWN -> passes
INSERT U VALUES (NULL, 5);       -- Fails: UNIQUE allows only one NULL
-- Many NULLs allowed with: CREATE UNIQUE INDEX UX ON U(Code) WHERE Code IS NOT NULL;
```

**Gotchas / follow-ups**
- `SET ANSI_NULLS OFF`/`CONCAT_NULL_YIELDS_NULL OFF` are deprecated.
- EF Core: `a == b` with nullable columns generates extra `IS NULL` logic (null semantics), which can hurt index use.

**Interview traps**
- `WHERE col = NULL`.
- `NOT IN` with NULLs.
- `<>` excluding NULL rows.
- `AVG` ignores NULLs, changing the denominator.
- ISNULL truncation.

**Cheat sheet**
- NULL = unknown; comparisons -> UNKNOWN; WHERE keeps TRUE only.
- `COALESCE` = ANSI, n args; `ISNULL` = 2 args, first-arg type.
- `x / NULLIF(y,0)` guards divide by zero.
- NULLs sort first ascending.
- UNIQUE: one NULL; CHECK: UNKNOWN passes.

---

## Topic 10: ORDER BY and OFFSET/FETCH pagination

**Study notes**

- Result sets have **no guaranteed order** without an outermost `ORDER BY`.
- `OFFSET n ROWS FETCH NEXT m ROWS ONLY` (SQL Server 2012+) requires `ORDER BY` and cannot be combined with `TOP`.
- Make ordering **deterministic** with a unique tie-breaker (e.g. the PK).
- **Offset paging** cost grows with the offset (rows are read and skipped); **keyset (seek) paging** uses `WHERE key > last` and stays fast.

Sample ordering: Salary DESC, EmpId ASC -> Asha 9000, Farid 8000, Ben 7000, Chen 7000, Esha 6000, Dev 5000.

### 10.1 [Easy] How do you page results in SQL Server?
**Answer**
`ORDER BY ... OFFSET @skip ROWS FETCH NEXT @take ROWS ONLY`. `OFFSET (page-1)*size` and `FETCH NEXT size`.

**Explanation**
`ORDER BY` is mandatory with OFFSET. EF Core's `Skip().Take()` translates to exactly this.

**Code**
```sql
DECLARE @page INT = 2, @size INT = 2;
SELECT EmpId, Name, Salary
FROM Employees
ORDER BY Salary DESC, EmpId
OFFSET (@page - 1) * @size ROWS FETCH NEXT @size ROWS ONLY;
-- 2 Ben 7000 | 3 Chen 7000
```

### 10.2 [Follow-up] Why do you need a unique tiebreaker in ORDER BY for paging?
**Answer**
With ties (Ben and Chen both 7000) the order among tied rows is non-deterministic and may differ between executions, so rows can repeat or be skipped across pages. Add a unique column like the PK as the final sort key.

**Explanation**
Even a stable-looking order can change when the plan changes (parallelism, different index).

**Code**
```sql
-- Risky: ORDER BY Salary DESC   (ties unspecified)
-- Safe:
ORDER BY Salary DESC, EmpId ASC
```

### 10.3 [Medium] Offset vs keyset pagination?
**Answer**
Offset paging must read and discard all skipped rows, so deep pages get slow and it can show duplicates when data changes between requests. Keyset paging remembers the last seen sort key and seeks using an index, giving constant-time pages but no random page jumps.

**Explanation**
Keyset needs a deterministic order and an index matching the ORDER BY. For multi-column keys use the expanded OR form (SQL Server has no row-value comparison `(a,b) > (x,y)`).

**Code**
```sql
-- Next page after (Salary=8000, EmpId=6) in Salary DESC, EmpId ASC
DECLARE @lastSal DECIMAL(10,2) = 8000, @lastId INT = 6;
SELECT TOP (2) EmpId, Name, Salary
FROM Employees
WHERE Salary < @lastSal OR (Salary = @lastSal AND EmpId > @lastId)
ORDER BY Salary DESC, EmpId;
-- 2 Ben 7000 | 3 Chen 7000
```

### 10.4 [Hard] ORDER BY details: ordinals, aliases, expressions, DISTINCT, collation, NULLs
**Answer**
`ORDER BY` can use aliases, ordinal positions (discouraged), and expressions not in the SELECT list - except with `DISTINCT` or `UNION`, where it must be in the select list. Strings sort per column collation; NULLs sort lowest.

**Explanation**
Ordinals break silently when the select list changes. With `GROUP BY`, ordering by a non-grouped, non-aggregated column is illegal. `ORDER BY (SELECT NULL)` is the idiom for "no particular order" required by syntax (e.g. in `ROW_NUMBER`).

**Code**
```sql
SELECT Name, Salary * 12 AS Annual FROM Employees ORDER BY Annual DESC, 1;  -- alias + ordinal
-- Error: ORDER BY items must appear in the select list if SELECT DISTINCT is specified
-- SELECT DISTINCT DeptId FROM Employees ORDER BY Salary;
```

### 10.5 [Tricky] OFFSET/FETCH edge cases and total counts
**Answer**
`OFFSET 0 ROWS` without FETCH returns all rows; `FETCH` can't be used without OFFSET; the offset and fetch values can be variables/expressions but must be non-negative (`FETCH` >= 1). To get a total count with the page, use `COUNT(*) OVER()` rather than a second query - but it forces evaluating all rows.

**Explanation**
Paging over a query with joins that multiply rows yields unstable pages - page the parent keys first, then join. `TOP` + `OFFSET` can't coexist.

**Code**
```sql
SELECT EmpId, Name, COUNT(*) OVER () AS TotalRows
FROM Employees
ORDER BY EmpId
OFFSET 4 ROWS FETCH NEXT 10 ROWS ONLY;
-- 5 Esha 6 | 6 Farid 6    (beyond the end -> fewer rows, not an error)
```

**Interview traps**
- Relying on the clustered index for implicit order.
- Paging without a unique tiebreaker.
- Deep OFFSET on large tables.
- Using ordinals in ORDER BY.
- Believing `ORDER BY` in a view/CTE sorts the final output.

**Cheat sheet**
- `OFFSET x ROWS FETCH NEXT y ROWS ONLY` needs ORDER BY.
- Tiebreak with PK.
- Keyset = seek by last key; offset = skip.
- NULLs sort first in ASC.
- `COUNT(*) OVER()` for total.

## Topic 11: String/date functions, collation, case sensitivity

**Study notes**

| Area | Functions |
|---|---|
| Strings | `LEN` (ignores trailing spaces), `DATALENGTH` (bytes), `SUBSTRING` (1-based), `CHARINDEX`, `LEFT/RIGHT`, `REPLACE`, `TRIM` (2017+), `CONCAT`, `STRING_AGG` (2017+), `STRING_SPLIT` (2016+), `FORMAT` (CLR-based, slow) |
| Dates | `GETDATE`/`SYSDATETIME`, `DATEADD`, `DATEDIFF`, `EOMONTH`, `DATEFROMPARTS`, `DATEPART`, `DATETRUNC` (2022+) |

- **Collation** = rules for sorting and comparing text. Default for English installs: `SQL_Latin1_General_CP1_CI_AS` (CI = case-insensitive, AS = accent-sensitive). Collation applies at server, database and column level; expression-level override with `COLLATE`.
- **SARGable**: a predicate that can use an index seek. Wrapping a column in a function (`YEAR(col)`, `LEFT(col,3)`) usually makes it non-SARGable.

### 11.1 [Easy] How do you concatenate, search and slice strings?
**Answer**
Use `+` or `CONCAT` to join, `CHARINDEX(find, text)` for position (0 = not found), `SUBSTRING(text, start, length)` (1-based) and `LEFT/RIGHT` for ends. `CONCAT` treats NULL as empty string, `+` returns NULL.

**Explanation**
`STRING_AGG(col, ',') WITHIN GROUP (ORDER BY col)` aggregates rows into a delimited list.

**Code**
```sql
SELECT CHARINDEX('sh', 'Asha')      AS pos,    -- 3
       SUBSTRING('Asha', 2, 2)      AS sub,    -- 'sh'
       CONCAT('A', NULL, 'B')       AS c,      -- 'AB'
       UPPER(LEFT('ben', 1)) + SUBSTRING('ben', 2, 100) AS proper;  -- 'Ben'

SELECT DeptId, STRING_AGG(Name, ', ') WITHIN GROUP (ORDER BY Name) AS Names
FROM Employees WHERE DeptId IS NOT NULL GROUP BY DeptId;
-- 1 Asha, Ben, Farid | 2 Chen, Dev
```

### 11.2 [Follow-up] LEN vs DATALENGTH, and trailing spaces in comparisons?
**Answer**
`LEN` returns characters excluding trailing spaces; `DATALENGTH` returns bytes including trailing spaces (and 2 bytes per char for nvarchar). In `=` comparisons, trailing spaces are ignored (ANSI padding), but `LIKE` does not ignore them.

**Explanation**
This matters for uniqueness: `'a'` and `'a '` collide in a UNIQUE index. Use `LEN(col + 'x') - 1` to count trailing spaces.

**Code**
```sql
SELECT LEN('ab  ')        AS len_,        -- 2
       DATALENGTH('ab  ') AS bytes,       -- 4
       DATALENGTH(N'ab')  AS nbytes;      -- 4
SELECT CASE WHEN 'a' = 'a ' THEN 'equal' ELSE 'diff' END;          -- equal
SELECT CASE WHEN 'a ' LIKE 'a' THEN 'match' ELSE 'no match' END;   -- no match
```

### 11.3 [Medium] Date arithmetic: DATEDIFF semantics and SARGable date filters
**Answer**
`DATEDIFF(part, start, end)` counts **boundaries crossed**, not elapsed time, so `DATEDIFF(year,'2023-12-31','2024-01-01')` = 1. For date range filters use half-open ranges on the raw column (`>= start AND < next`) instead of wrapping the column in `YEAR()`/`MONTH()`.

**Explanation**
Half-open ranges also avoid the `BETWEEN` end-of-day problem with time components and are index-friendly.

**Code**
```sql
SELECT DATEDIFF(year,  '2023-12-31', '2024-01-01') AS y,   -- 1
       DATEDIFF(day,   '2024-01-31', '2024-02-01') AS d,   -- 1
       EOMONTH('2024-02-10')                        AS eom, -- 2024-02-29
       DATEADD(month, 1, '2024-01-31')              AS m;   -- 2024-02-29

-- Non-SARGable: WHERE YEAR(OrderDate)=2024 AND MONTH(OrderDate)=1
SELECT OrderId FROM Orders
WHERE OrderDate >= '2024-01-01' AND OrderDate < '2024-02-01';   -- 1, 2
```

### 11.4 [Hard] Collation, case sensitivity and accent sensitivity
**Answer**
Case/accent sensitivity depends on the collation: `_CI_`/`_CS_` for case, `_AI_`/`_AS_` for accent. Override per expression with `COLLATE`, e.g. `col COLLATE Latin1_General_CS_AS = 'Asha'` for a case-sensitive match.

**Explanation**
Applying `COLLATE` to a column in a predicate can prevent index seek unless the column's collation already matches. A comparison between columns of different collations raises "Cannot resolve the collation conflict". A classic cause: `tempdb` uses the server default collation while the user database uses another; fix with `COLLATE DATABASE_DEFAULT` in temp table definitions.

**Code**
```sql
SELECT CASE WHEN 'asha' = 'ASHA' THEN 'eq' ELSE 'ne' END;                                    -- eq   (CI default)
SELECT CASE WHEN 'asha' COLLATE Latin1_General_CS_AS = 'ASHA' THEN 'eq' ELSE 'ne' END;       -- ne
CREATE TABLE #T (Name NVARCHAR(50) COLLATE DATABASE_DEFAULT);    -- avoids tempdb conflicts
SELECT name, collation_name FROM sys.databases WHERE name = DB_NAME();
```

### 11.5 [Tricky] Implicit conversion, FORMAT and LIKE pitfalls
**Answer**
Comparing a `varchar` column to an `nvarchar` value can force a column-side conversion (CONVERT_IMPLICIT) and a scan. `LIKE 'abc%'` is SARGable but `LIKE '%abc'` is not. `FORMAT` is convenient but much slower than `CONVERT` for large result sets.

**Explanation**
Wildcards `[ ]`, `_`, `%` in LIKE need escaping with `[%]` or `ESCAPE`. `STRING_SPLIT` returns unordered rows (ordinal column only added in 2022 with `enable_ordinal`). Implicit conversion severity depends on data type precedence and collation (SQL collations vs Windows collations differ).

**Code**
```sql
SELECT Name FROM Employees WHERE Name LIKE 'A%';       -- Asha     (seek possible)
SELECT Name FROM Employees WHERE Name LIKE '%a';       -- Asha, Esha (scan)
SELECT value FROM STRING_SPLIT('a,b,c', ',');          -- a, b, c (order not guaranteed)
SELECT 100 AS pct WHERE 'a_b' LIKE 'a[_]b';            -- literal underscore
```

**Interview traps**
- `BETWEEN` on datetime with end-of-day.
- `YEAR(col) = 2024` in WHERE.
- Thinking `LEN` counts trailing spaces.
- Forgetting `N` prefix.
- Cross-database/temp table collation conflicts.

**Cheat sheet**
- `LEN` trims trailing spaces; `DATALENGTH` = bytes.
- `DATEDIFF` counts boundaries.
- Half-open date ranges.
- Default collation is case-insensitive (CI) in most installs.
- `STRING_AGG ... WITHIN GROUP (ORDER BY)`.

---

## Topic 12: Constraints (PK, FK, UNIQUE, CHECK, DEFAULT) and cascading actions

**Study notes**

| Constraint | Rule | Notes |
|---|---|---|
| PRIMARY KEY | unique + NOT NULL, one per table | creates a **clustered** index by default (can be nonclustered) |
| UNIQUE | unique values; **one NULL** allowed | creates nonclustered unique index |
| FOREIGN KEY | child value must exist in parent key (or be NULL) | parent must be PK or UNIQUE; **no automatic index** on child column |
| CHECK | boolean rule; row rejected only if FALSE | UNKNOWN (NULL) passes |
| DEFAULT | value used when column omitted | not used when NULL is explicitly inserted |

Referential actions on FK: `NO ACTION` (default, error), `CASCADE`, `SET NULL`, `SET DEFAULT`.

### 12.1 [Easy] What are the main constraint types and what does each enforce?
**Answer**
PRIMARY KEY identifies rows uniquely (not null), UNIQUE enforces uniqueness on alternate keys, FOREIGN KEY enforces referential integrity, CHECK enforces a column/row condition, DEFAULT supplies a value when none is given.

**Explanation**
Name your constraints explicitly; auto-generated names (`PK__Employee__A1B2C3`) differ between environments and complicate migrations.

**Code**
```sql
CREATE TABLE Products (
    ProductId INT           NOT NULL CONSTRAINT PK_Products PRIMARY KEY,
    Sku       VARCHAR(20)   NOT NULL CONSTRAINT UQ_Products_Sku UNIQUE,
    Price     DECIMAL(10,2) NOT NULL CONSTRAINT CK_Products_Price CHECK (Price >= 0),
    CreatedAt DATETIME2(0)  NOT NULL CONSTRAINT DF_Products_Created DEFAULT SYSUTCDATETIME(),
    DeptId    INT           NULL CONSTRAINT FK_Products_Dept REFERENCES Departments(DeptId)
);
```

### 12.2 [Follow-up] What are cascading actions, and what are the options?
**Answer**
They define what happens to child rows when the parent key is deleted/updated: `NO ACTION` (reject), `CASCADE` (propagate), `SET NULL`, `SET DEFAULT`. Default is NO ACTION.

**Explanation**
`CASCADE` is convenient but dangerous for large trees (mass deletes, locks, log growth). EF Core defaults required relationships to cascade delete, optional ones to client set-null/restrict depending on version/config - know what your model generates.

**Code**
```sql
CREATE TABLE OrderLines (
    LineId  INT IDENTITY PRIMARY KEY,
    OrderId INT NOT NULL
        CONSTRAINT FK_OrderLines_Orders REFERENCES Orders(OrderId) ON DELETE CASCADE,
    Sku     VARCHAR(20) NOT NULL
);
DELETE FROM Orders WHERE OrderId = 1;   -- also removes its OrderLines
-- Without CASCADE: error 547 "DELETE statement conflicted with the REFERENCE constraint"
```

### 12.3 [Medium] FK does not create an index - why does that matter?
**Answer**
SQL Server doesn't index the child FK column automatically. Without an index, joins from parent to child and parent deletes/updates (which must check children) scan the child table and take more locks, so you should usually index FK columns.

**Explanation**
Deleting one department triggers a lookup in Employees for referencing rows; with no index on `DeptId` that is a table scan under lock - a classic cause of blocking and deadlocks.

**Code**
```sql
CREATE INDEX IX_Employees_DeptId ON Employees (DeptId);
CREATE INDEX IX_Orders_EmpId     ON Orders (EmpId);
```

### 12.4 [Hard] Multiple cascade paths and self-referencing FKs
**Answer**
SQL Server rejects a cascading FK if it creates cycles or multiple cascade paths (error 1785), and a self-referencing FK (e.g. `ManagerId`) cannot use `ON DELETE CASCADE`/`SET NULL` (cycles). Resolve by using NO ACTION and handling deletes in a trigger/stored procedure/application.

**Explanation**
A table reachable through two cascade routes from the same parent is ambiguous. Alternatively, use `ON DELETE SET NULL` on just one of them.

**Code**
```sql
-- Fails: self-reference with cascade
-- ALTER TABLE Employees ADD CONSTRAINT FK_Mgr FOREIGN KEY (ManagerId)
--   REFERENCES Employees(EmpId) ON DELETE CASCADE;
-- Msg 1785: ... may cause cycles or multiple cascade paths.

-- Manual alternative
UPDATE Employees SET ManagerId = NULL WHERE ManagerId = 1;
DELETE Employees WHERE EmpId = 1;
```

### 12.5 [Tricky] Trusted constraints, NOCHECK, CHECK with NULL, DEFAULT vs explicit NULL
**Answer**
`WITH NOCHECK` adds a constraint without validating existing rows, leaving it **not trusted** so the optimizer ignores it. A CHECK passes when it evaluates to UNKNOWN, and DEFAULT does not apply when you explicitly insert NULL (you get NULL or a NOT NULL violation).

**Explanation**
Re-enable with `ALTER TABLE ... WITH CHECK CHECK CONSTRAINT name` to make trusted again. `ALTER TABLE ... NOCHECK CONSTRAINT` disables enforcement entirely (bulk loads). Find untrusted ones in `sys.check_constraints`/`sys.foreign_keys` (`is_not_trusted = 1`).

**Code**
```sql
ALTER TABLE Products NOCHECK CONSTRAINT CK_Products_Price;
ALTER TABLE Products WITH CHECK CHECK CONSTRAINT CK_Products_Price;   -- re-validate, trusted again

SELECT name, is_not_trusted FROM sys.foreign_keys WHERE is_not_trusted = 1;

INSERT Products (ProductId, Sku, Price) VALUES (1, 'A', 5);                    -- CreatedAt = default
-- INSERT Products (ProductId, Sku, Price, CreatedAt) VALUES (2,'B',5,NULL);   -- fails: NOT NULL, default not used
INSERT Products (ProductId, Sku, Price, CreatedAt) VALUES (3,'C',5,DEFAULT);   -- explicit DEFAULT works
```

**Interview traps**
- "A PK must be clustered" - no; it's only the default.
- Assuming FK creates an index.
- Cascade delete surprises (EF Core conventions).
- CHECK allowing NULLs unexpectedly.
- Untrusted constraints silently hurt plans.

**Cheat sheet**
- PK = unique + not null; UNIQUE = one NULL.
- FK default action = NO ACTION; index FK columns yourself.
- CHECK fails only on FALSE.
- Self-referencing FK can't cascade.
- `WITH NOCHECK` => untrusted.

---

## Topic 13: IDENTITY, SEQUENCE, GUID keys

**Study notes**

| Option | Generated by | Scope | Pros | Cons |
|---|---|---|---|---|
| `IDENTITY(seed,inc)` | column property | one table | simple, narrow, ever-increasing | gaps; not reusable; `IDENTITY_INSERT` needed to set manually |
| `SEQUENCE` (2012+) | schema object | any table/many | share across tables, get value before insert, `CACHE` | gaps; app must call `NEXT VALUE FOR` or use DEFAULT |
| `uniqueidentifier` | `NEWID()` / `NEWSEQUENTIALID()` / client | global | generate anywhere, merge-friendly | 16 bytes, random = fragmentation (NEWID) |

### 13.1 [Easy] How does IDENTITY work and how do you get the generated value?
**Answer**
`IDENTITY(1,1)` auto-generates incrementing integers on insert. Retrieve the value with `SCOPE_IDENTITY()` (current scope/session), or the `OUTPUT` clause.

**Explanation**
`@@IDENTITY` returns the last identity in the session regardless of scope (including values inserted by triggers) - risky. `IDENT_CURRENT('table')` is table-wide across sessions - also not safe for your own insert.

**Code**
```sql
CREATE TABLE Tickets (TicketId INT IDENTITY(1,1) PRIMARY KEY, Title NVARCHAR(50));
INSERT Tickets (Title) VALUES (N'First');
SELECT SCOPE_IDENTITY() AS NewId;        -- 1
```

### 13.2 [Follow-up] Do identity values have gaps? Does rollback reuse them?
**Answer**
Yes, gaps happen. A rolled-back insert still consumes the value, and failures, deletes or a server restart can also leave gaps. Never use identity for gap-free numbering (invoice numbers).

**Explanation**
Identity increments are not transactional. Since SQL Server 2012 the engine pre-allocates (caches) identity values; an unexpected restart can jump the value (commonly by up to 1000 for int; configurable with `IDENTITY_CACHE`, 2017+, database-scoped option).

**Code**
```sql
BEGIN TRAN; INSERT Tickets (Title) VALUES (N'Rolled back'); ROLLBACK;
INSERT Tickets (Title) VALUES (N'Second');
SELECT TicketId, Title FROM Tickets;
-- 1 First | 3 Second   (2 was consumed by the rolled-back insert)
```

### 13.3 [Medium] DELETE vs TRUNCATE effect on identity, and inserting explicit values
**Answer**
`DELETE` leaves the identity counter where it is; `TRUNCATE` resets it to the seed. To insert explicit values into an identity column use `SET IDENTITY_INSERT table ON` (one table per session at a time, column list required). `DBCC CHECKIDENT` can reseed.

**Explanation**
Reseeding below existing values can cause PK violations later.

**Code**
```sql
SET IDENTITY_INSERT Tickets ON;
INSERT Tickets (TicketId, Title) VALUES (100, N'Imported');   -- column list is mandatory
SET IDENTITY_INSERT Tickets OFF;
INSERT Tickets (Title) VALUES (N'Next');                       -- 101 (max+1 after explicit insert)
DBCC CHECKIDENT ('Tickets', RESEED, 500);
```

### 13.4 [Hard] SEQUENCE vs IDENTITY - when to use SEQUENCE?
**Answer**
Use a SEQUENCE when several tables share a number series, when you need the key *before* inserting (e.g. EF Core HiLo to batch inserts and set parent/child keys client-side), or when you need to restart/cycle the series. IDENTITY is simpler for a single table.

**Explanation**
Sequence options: `START WITH`, `INCREMENT BY`, `MINVALUE/MAXVALUE`, `CYCLE`, `CACHE n`. Gaps still occur. A default constraint `DEFAULT (NEXT VALUE FOR dbo.SeqName)` makes it behave like identity. `sp_sequence_get_range` reserves blocks.

**Code**
```sql
CREATE SEQUENCE dbo.SeqOrderNo AS INT START WITH 1000 INCREMENT BY 1 CACHE 50;
SELECT NEXT VALUE FOR dbo.SeqOrderNo;     -- 1000

CREATE TABLE Invoices (
    InvoiceNo INT NOT NULL DEFAULT (NEXT VALUE FOR dbo.SeqOrderNo) PRIMARY KEY,
    Note NVARCHAR(50)
);
INSERT Invoices (Note) VALUES (N'x');      -- InvoiceNo = 1001
```

### 13.5 [Tricky] GUID keys: NEWID vs NEWSEQUENTIALID, clustering and security
**Answer**
`NEWID()` is random, so as a clustered key it causes page splits and fragmentation. `NEWSEQUENTIALID()` is increasing (until restart) and only allowed in a DEFAULT constraint, but it is predictable and its sequence can restart lower after a reboot. A common design: a narrow `INT/BIGINT` identity clustered PK plus a GUID as a nonclustered unique public identifier.

**Explanation**
Sorting GUIDs in SQL Server compares byte groups in a peculiar order (last 6 bytes most significant), so .NET `Guid.NewGuid()` values don't sort chronologically; EF Core's SQL Server sequential GUID generator (`SequentialGuidValueGenerator`) accounts for this. GUIDs are larger in every nonclustered index (16 bytes in each leaf entry).

**Code**
```sql
CREATE TABLE Docs (
    DocId  INT IDENTITY(1,1) NOT NULL,
    PublicId UNIQUEIDENTIFIER NOT NULL DEFAULT NEWSEQUENTIALID(),
    CONSTRAINT PK_Docs PRIMARY KEY CLUSTERED (DocId),
    CONSTRAINT UQ_Docs_Public UNIQUE (PublicId)
);
-- SELECT NEWSEQUENTIALID();  -- error: only valid in DEFAULT constraints
```

**Interview traps**
- Using `@@IDENTITY` instead of `SCOPE_IDENTITY()`.
- Expecting gap-free identity.
- Clustering on random GUIDs.
- TRUNCATE resets identity; DELETE doesn't.
- Exposing sequential ids publicly (enumeration).

**Cheat sheet**
- `SCOPE_IDENTITY()` > `@@IDENTITY`.
- Identity/sequence = gaps possible.
- `IDENTITY_INSERT` is per table/session, list columns.
- SEQUENCE shares across tables; get value before insert.
- Random GUID clustered key = fragmentation.

---

## Topic 14: Normalization (1NF-BCNF) and denormalization

**Study notes**

Goal: remove redundancy and update anomalies by organizing data according to dependencies.

```
 1NF  : atomic values, no repeating groups, a primary key
 2NF  : 1NF + no partial dependency (non-key column depends on PART of a composite key)
 3NF  : 2NF + no transitive dependency (non-key -> non-key)
 BCNF : every determinant is a candidate key
```
Mnemonic: "every non-key attribute depends on the key (1NF), the whole key (2NF), and nothing but the key (3NF), so help me Codd (BCNF)."

### 14.1 [Easy] What is normalization and why do it?
**Answer**
Normalization organizes tables so each fact is stored once, removing redundancy and insert/update/delete anomalies. It produces more, narrower tables linked by keys.

**Explanation**
Anomalies: *update* (change one copy but not others), *insert* (can't add a fact without another), *delete* (losing a fact when deleting another).

**Code**
```sql
-- Unnormalized: department name repeated in every employee row (update anomaly)
-- EmployeeFlat(EmpId, Name, DeptId, DeptName)
-- Normalized: Employees(DeptId) -> Departments(DeptId, DeptName)  (the sample schema)
```

### 14.2 [Follow-up] Explain 1NF, 2NF and 3NF with examples.
**Answer**
1NF: one value per cell and no repeating columns (no `Phone1, Phone2` or comma lists). 2NF: in a composite-key table every non-key column depends on the whole key. 3NF: non-key columns must not depend on other non-key columns.

**Explanation**
Example 2NF violation: `OrderLine(OrderId, ProductId, Qty, ProductName)` - ProductName depends only on ProductId. 3NF violation: `Employee(EmpId, DeptId, DeptName)` - DeptName depends on DeptId (non-key).

**Code**
```sql
-- 1NF fix: move repeating phones to a child table
CREATE TABLE EmployeePhones (
    EmpId INT NOT NULL REFERENCES Employees(EmpId),
    Phone VARCHAR(20) NOT NULL,
    PRIMARY KEY (EmpId, Phone)
);
-- 2NF fix: ProductName moves to Products(ProductId, ProductName)
-- 3NF fix: DeptName lives only in Departments
```

### 14.3 [Medium] What is BCNF and how does it differ from 3NF?
**Answer**
BCNF requires that for every functional dependency X -> Y, X is a candidate key. 3NF tolerates a dependency where Y is a prime (key) attribute; BCNF does not. Violations occur with overlapping composite candidate keys.

**Explanation**
Example: `Enrollment(Student, Course, Teacher)` where each teacher teaches one course (Teacher -> Course) and key is (Student, Course) or (Student, Teacher). Teacher is a determinant but not a candidate key -> not BCNF. Decompose into `Teaches(Teacher, Course)` and `StudentTeacher(Student, Teacher)`. BCNF decomposition can sometimes lose the ability to enforce a dependency with a constraint.

**Code**
```sql
CREATE TABLE Teaches (Teacher VARCHAR(30) PRIMARY KEY, Course VARCHAR(30) NOT NULL);
CREATE TABLE StudentTeacher (
    Student VARCHAR(30) NOT NULL,
    Teacher VARCHAR(30) NOT NULL REFERENCES Teaches(Teacher),
    PRIMARY KEY (Student, Teacher)
);
```

### 14.4 [Hard] When and how should you denormalize? What are the trade-offs?
**Answer**
Denormalize deliberately for read performance: store computed totals, copy frequently joined columns, or build reporting tables/indexed views. The cost is redundancy, extra write work and the risk of inconsistency that you must manage with constraints, triggers, computed columns or jobs.

**Explanation**
Prefer first: proper indexes, covering indexes, indexed views, read replicas, caching. OLTP schemas are normally 3NF; OLAP/reporting uses star schemas (denormalized dimensions). Snapshotting values (e.g. unit price on an order line) is *not* denormalization - it is a legitimate historical fact.

**Code**
```sql
-- Persisted computed column keeps derived data consistent automatically
ALTER TABLE Employees ADD AnnualSalary AS (Salary * 12) PERSISTED;

-- Indexed view: materialized aggregate maintained by the engine
CREATE VIEW dbo.vOrderTotals WITH SCHEMABINDING AS
SELECT EmpId, SUM(Amount) AS Total, COUNT_BIG(*) AS Cnt
FROM dbo.Orders WHERE EmpId IS NOT NULL GROUP BY EmpId;
-- Amount is NOT NULL, so SUM is allowed in an indexed view
GO
CREATE UNIQUE CLUSTERED INDEX IX_vOrderTotals ON dbo.vOrderTotals (EmpId);
```

### 14.5 [Tricky] Is a many-to-many relationship normalized without a junction table? What about EAV and JSON columns?
**Answer**
Many-to-many needs a junction (bridge) table with a composite PK of both foreign keys. EAV (entity-attribute-value) and JSON columns trade integrity and query performance for flexibility; they are acceptable for truly sparse/variable attributes, not as a replacement for modelled data.

**Explanation**
SQL Server supports JSON via functions (`JSON_VALUE`, `OPENJSON`) on `nvarchar` (a native `json` type exists only in newer versions/Azure), no FK or type enforcement. Computed columns over JSON values can be indexed.

**Code**
```sql
CREATE TABLE Projects (ProjectId INT PRIMARY KEY, Title NVARCHAR(50));
CREATE TABLE EmployeeProjects (
    EmpId     INT NOT NULL REFERENCES Employees(EmpId),
    ProjectId INT NOT NULL REFERENCES Projects(ProjectId),
    PRIMARY KEY (EmpId, ProjectId)
);
```

**Interview traps**
- Confusing 2NF and 3NF (partial vs transitive).
- "Always normalize to BCNF" or "always denormalize for speed".
- Treating historic snapshots (price at time of order) as redundancy.
- Comma-separated lists in a column (violates 1NF).

**Cheat sheet**
- 1NF atomic; 2NF whole key; 3NF nothing but the key; BCNF every determinant is a key.
- Partial dependency = 2NF; transitive = 3NF.
- Denormalize last, after indexing/caching options.
- Many-to-many = junction table.

## Topic 15: DELETE vs TRUNCATE vs DROP

**Study notes**

| | DELETE | TRUNCATE TABLE | DROP TABLE |
|---|---|---|---|
| Category | DML | DDL | DDL |
| Removes | rows (with `WHERE`) | all rows | table + data + indexes + permissions |
| Logging | each row fully logged | page deallocations logged (minimal) | metadata |
| Triggers | `DELETE` triggers fire | do not fire | n/a |
| Identity | not reset | reset to seed | n/a |
| FK referenced | allowed if no children | **blocked** if referenced by an FK (even if child empty) | blocked if referenced |
| Permission | `DELETE` | `ALTER` on table | `ALTER` on schema / control |
| Rollback | yes | yes (in a transaction) | yes (in a transaction) |
| Locking | row/page/table as needed | schema-modify + table lock | schema lock |

### 15.1 [Easy] Difference between DELETE, TRUNCATE and DROP?
**Answer**
`DELETE` removes selected rows and is fully logged and trigger-aware; `TRUNCATE` quickly removes all rows and resets identity; `DROP` removes the entire table definition and data.

**Explanation**
`TRUNCATE` is fast because it deallocates data pages instead of logging each row, but has no `WHERE` clause.

**Code**
```sql
DELETE FROM Orders WHERE OrderId = 5;     -- one row
TRUNCATE TABLE Tickets;                   -- all rows, identity back to seed
DROP TABLE IF EXISTS Tickets;             -- table gone (IF EXISTS: 2016+)
```

### 15.2 [Follow-up] Can TRUNCATE be rolled back? Does it fire triggers?
**Answer**
Yes, inside an explicit transaction TRUNCATE can be rolled back, because the page deallocations are logged. It does not fire `DELETE` triggers.

**Explanation**
The common myth "TRUNCATE can't be rolled back" comes from other databases. Because it doesn't touch rows individually, it is not suitable when you need row-level auditing via triggers.

**Code**
```sql
CREATE TABLE T (Id INT IDENTITY, V INT);
INSERT T (V) VALUES (1),(2);
BEGIN TRAN; TRUNCATE TABLE T; ROLLBACK;
SELECT COUNT(*) FROM T;          -- 2
DROP TABLE T;
```

### 15.3 [Medium] Why does TRUNCATE fail on a table referenced by a foreign key?
**Answer**
SQL Server refuses `TRUNCATE` on any table that is referenced by a FK constraint, even when the child table is empty, because it can't verify referential integrity without scanning. Drop/disable the FK, or use `DELETE`.

**Explanation**
Also `TRUNCATE` is not allowed on tables used in indexed views, published by transactional replication, or with system versioning (temporal) enabled unless handled. SQL Server 2016+ allows truncating specific partitions.

**Code**
```sql
TRUNCATE TABLE Departments;
-- Msg 4712: Cannot truncate table 'Departments' because it is being referenced by a FOREIGN KEY constraint.

TRUNCATE TABLE Sales_Part WITH (PARTITIONS (1 TO 3));   -- partition-level truncate (2016+)
```

### 15.4 [Hard] Deleting millions of rows without blowing up the log or blocking everyone
**Answer**
Delete in batches (`DELETE TOP (n)` in a loop) so each transaction is small, letting log backups/checkpoints truncate the log and avoiding lock escalation. Alternatively, switch out a partition or `TRUNCATE` if removing everything.

**Explanation**
One big `DELETE` writes a huge log, holds locks (may escalate to table lock at ~5000 locks) and a rollback takes as long as the work. If deleting most of the table, copying the keepers to a new table and swapping can be faster. Put an index on the filter column.

**Code**
```sql
DECLARE @batch INT = 5000;
WHILE 1 = 1
BEGIN
    DELETE TOP (@batch) FROM Orders WHERE OrderDate < '2020-01-01';
    IF @@ROWCOUNT < @batch BREAK;
    -- optional: WAITFOR DELAY '00:00:01' to reduce pressure
END
```

### 15.5 [Tricky] DELETE with joins, DELETE without WHERE accidents, and heap space
**Answer**
T-SQL supports `DELETE ... FROM ... JOIN` to delete based on another table; always run the equivalent `SELECT` first or wrap in a transaction. Deleting from a **heap** may not release pages unless a table lock is taken (`WITH (TABLOCK)`) or the heap is rebuilt.

**Explanation**
In a safe workflow: `BEGIN TRAN`, run the delete, check `@@ROWCOUNT`, then commit or rollback. `OUTPUT deleted.*` (Topic 16) gives an audit trail.

**Code**
```sql
BEGIN TRAN;
DELETE o
FROM Orders o
LEFT JOIN Employees e ON e.EmpId = o.EmpId
WHERE e.EmpId IS NULL;                -- orphan orders (order 5)
SELECT @@ROWCOUNT AS Deleted;          -- 1
ROLLBACK;                              -- demo only
```

**Interview traps**
- "TRUNCATE can't be rolled back."
- TRUNCATE doesn't fire triggers.
- TRUNCATE blocked by FKs even for empty child tables.
- Giant single-transaction deletes.
- Forgetting the `WHERE`.

**Cheat sheet**
- DELETE = rows, logged, triggers, no identity reset.
- TRUNCATE = all rows, minimal log, identity reset, needs ALTER.
- DROP = object removed.
- Batch big deletes.
- All three can be rolled back inside a transaction.

---

## Topic 16: INSERT/UPDATE/DELETE with OUTPUT, MERGE and upsert patterns

**Study notes**

- `OUTPUT` returns rows affected by `INSERT/UPDATE/DELETE/MERGE`. Pseudo-tables: `inserted` (new values), `deleted` (old values). In `MERGE`, `$action` reports `INSERT/UPDATE/DELETE`.
- Send OUTPUT to the client or `OUTPUT ... INTO @table`. If the target table has enabled triggers, `OUTPUT INTO` is required (plain OUTPUT is not allowed).
- **Upsert** = update if exists, else insert. Must be concurrency-safe.

### 16.1 [Easy] What does the OUTPUT clause do?
**Answer**
It returns data from the rows modified by a DML statement, including identity values and old/new values, without a separate SELECT. It's ideal for getting generated keys and auditing.

**Explanation**
`UPDATE` can reference both `deleted.Col` (before) and `inserted.Col` (after). It reports what actually happened in that statement, so it is race-free, unlike selecting afterward.

**Code**
```sql
CREATE TABLE #Log (EmpId INT, OldSal DECIMAL(10,2), NewSal DECIMAL(10,2));

BEGIN TRAN;
UPDATE Employees
SET Salary = Salary * 1.10
OUTPUT inserted.EmpId, deleted.Salary, inserted.Salary INTO #Log
WHERE DeptId = 2;
SELECT * FROM #Log;       -- 3 7000.00 7700.00 | 4 5000.00 5500.00
ROLLBACK;                 -- demo only

INSERT Tickets (Title) OUTPUT inserted.TicketId VALUES (N'New');   -- returns generated identity
```

### 16.2 [Follow-up] How do you write a safe upsert without MERGE?
**Answer**
Run `UPDATE`, and if `@@ROWCOUNT = 0` then `INSERT`, inside a transaction with `UPDLOCK, SERIALIZABLE` (or `HOLDLOCK`) hints so two sessions cannot both decide the row is missing. Back it with a unique constraint as the final safety net.

**Explanation**
The naive `IF NOT EXISTS (...) INSERT` has a race: two concurrent sessions both pass the check, then one fails with a duplicate-key error (or you get duplicates without a unique constraint). Catch error 2627/2601 and retry as an alternative.

**Code**
```sql
CREATE TABLE Stock (Sku VARCHAR(20) PRIMARY KEY, Qty INT NOT NULL);

DECLARE @sku VARCHAR(20) = 'A1', @qty INT = 5;
BEGIN TRAN;
    UPDATE Stock WITH (UPDLOCK, SERIALIZABLE) SET Qty = Qty + @qty WHERE Sku = @sku;
    IF @@ROWCOUNT = 0
        INSERT Stock (Sku, Qty) VALUES (@sku, @qty);
COMMIT;
```

### 16.3 [Medium] How does MERGE work?
**Answer**
`MERGE target USING source ON condition` then `WHEN MATCHED THEN UPDATE/DELETE`, `WHEN NOT MATCHED [BY TARGET] THEN INSERT`, and `WHEN NOT MATCHED BY SOURCE THEN UPDATE/DELETE`. It is one statement performing multiple actions, and must end with a semicolon.

**Explanation**
`OUTPUT $action` shows what was done per row. Each target row may be matched by at most one source row.

**Code**
```sql
MERGE Stock AS t
USING (VALUES ('A1', 3), ('B2', 7)) AS s (Sku, Qty)
   ON t.Sku = s.Sku
WHEN MATCHED THEN UPDATE SET t.Qty = t.Qty + s.Qty
WHEN NOT MATCHED BY TARGET THEN INSERT (Sku, Qty) VALUES (s.Sku, s.Qty)
OUTPUT $action, inserted.Sku, inserted.Qty;
-- UPDATE A1 8 | INSERT B2 7   (given A1 had 5)
```

### 16.4 [Hard] What are MERGE's pitfalls, and should you use it?
**Answer**
MERGE has a history of bugs (wrong results/access violations in some versions with certain triggers, indexed views, filtered indexes), is **not concurrency-safe by default** (needs `HOLDLOCK`), errors if a target row matches multiple source rows, and fires insert/update/delete triggers at once. Many teams avoid it in OLTP and use explicit UPDATE/INSERT (or use MERGE only for batch/ETL with testing).

**Explanation**
Without `WITH (HOLDLOCK)` on the target, concurrent MERGEs can hit duplicate key errors. `WHEN NOT MATCHED BY SOURCE` can delete more than intended if the source is partial. Triggers see all the actions with `inserted/deleted` combined; `@@ROWCOUNT` is the total. Always put the filtering of the target in the `WHEN` clause (`AND ...`), not the `ON` - an `ON` filter on the target does not limit which rows can be deleted/updated as people expect.

**Code**
```sql
MERGE Stock WITH (HOLDLOCK) AS t
USING (VALUES ('A1', 1)) AS s (Sku, Qty) ON t.Sku = s.Sku
WHEN MATCHED THEN UPDATE SET Qty = t.Qty + s.Qty
WHEN NOT MATCHED THEN INSERT (Sku, Qty) VALUES (s.Sku, s.Qty);

-- Error 8672 if the source has duplicate keys:
-- USING (VALUES ('A1',1),('A1',2)) AS s(Sku,Qty)
-- "The MERGE statement attempted to UPDATE or DELETE the same row more than once."
```

**Gotchas / follow-ups**
- Aggregate/de-duplicate the source before merging.
- `WHEN NOT MATCHED BY SOURCE THEN DELETE` + partial source = data loss.
- EF Core's `ExecuteUpdate` / third-party bulk libraries sometimes generate MERGE; know what they emit.

### 16.5 [Tricky] OUTPUT with triggers, computed values, and capturing source columns
**Answer**
If the target table has enabled triggers, `OUTPUT` must use `INTO`. OUTPUT from a plain `INSERT` can return only `inserted` columns, but from `MERGE` you can also reference source columns, which is useful to map generated identity values back to input rows.

**Explanation**
The `OUTPUT` table can't have triggers or be on either side of an FK. Values in OUTPUT reflect the state *before* AFTER-triggers run.

**Code**
```sql
CREATE TABLE #Map (SrcKey INT, NewId INT);
MERGE Tickets AS t
USING (VALUES (10, N'A'), (11, N'B')) AS s (SrcKey, Title) ON 1 = 0   -- never matches: always insert
WHEN NOT MATCHED THEN INSERT (Title) VALUES (s.Title)
OUTPUT s.SrcKey, inserted.TicketId INTO #Map (SrcKey, NewId);
SELECT * FROM #Map;     -- (10, <id>) (11, <id>)
```

**Interview traps**
- `IF NOT EXISTS ... INSERT` race condition.
- MERGE without HOLDLOCK in concurrent code.
- Duplicate source rows -> error 8672.
- OUTPUT on tables with triggers requires `INTO`.
- Reading `SCOPE_IDENTITY()` for multi-row inserts (returns only the last).

**Cheat sheet**
- `OUTPUT inserted.*, deleted.*` = new/old values.
- Upsert: UPDATE; if `@@ROWCOUNT=0` INSERT with UPDLOCK/SERIALIZABLE.
- MERGE needs `HOLDLOCK` + de-duplicated source + `;`.
- `$action` in MERGE OUTPUT.
- Unique constraint = ultimate safety net.

---

## Topic 17: Classic interview queries

**Study notes**

Pattern toolbox: ranking window functions, aggregate + self join, `EXISTS`, `LAG/LEAD`, CTEs.

| Function | Ties behaviour |
|---|---|
| `ROW_NUMBER()` | unique 1,2,3 even for ties |
| `RANK()` | ties share rank, gap after (1,1,3) |
| `DENSE_RANK()` | ties share rank, no gap (1,1,2) |

Sample salaries: 9000, 8000, 7000, 7000, 6000, 5000.

### 17.1 [Easy] Find the second highest salary.
**Answer**
Use `DENSE_RANK()` (handles ties) or `OFFSET/FETCH` over distinct salaries, or a nested `MAX` with `< (SELECT MAX)`. Mention that it returns nothing (or NULL) if there is no second value.

**Explanation**
`ROW_NUMBER` would be wrong with ties at the top; `TOP 1 ... NOT IN` style works but is clumsy. For the Nth highest, change the rank.

**Code**
```sql
-- 1) nested MAX (returns NULL if no 2nd value)
SELECT MAX(Salary) AS SecondHighest
FROM Employees WHERE Salary < (SELECT MAX(Salary) FROM Employees);     -- 8000

-- 2) DENSE_RANK (generalizes to Nth)
SELECT DISTINCT Salary FROM (
    SELECT Salary, DENSE_RANK() OVER (ORDER BY Salary DESC) AS rnk FROM Employees
) x WHERE rnk = 2;                                                      -- 8000

-- 3) OFFSET/FETCH
SELECT DISTINCT Salary FROM Employees
ORDER BY Salary DESC OFFSET 1 ROW FETCH NEXT 1 ROW ONLY;                -- 8000
```

### 17.2 [Follow-up] Find and delete duplicate rows.
**Answer**
Find with `GROUP BY ... HAVING COUNT(*) > 1`. Delete by numbering duplicates with `ROW_NUMBER() OVER (PARTITION BY dup-columns ORDER BY id)` in a CTE and deleting rows with `rn > 1`.

**Explanation**
Deleting through a CTE removes rows from the underlying table. Decide which copy survives via the `ORDER BY`. Add a unique constraint afterwards to prevent recurrence.

**Code**
```sql
CREATE TABLE #Emails (Id INT IDENTITY PRIMARY KEY, Email VARCHAR(50));
INSERT #Emails VALUES ('a@x.com'),('b@x.com'),('a@x.com'),('a@x.com');

SELECT Email, COUNT(*) AS Cnt FROM #Emails GROUP BY Email HAVING COUNT(*) > 1;   -- a@x.com 3

WITH d AS (
    SELECT ROW_NUMBER() OVER (PARTITION BY Email ORDER BY Id) AS rn FROM #Emails
)
DELETE FROM d WHERE rn > 1;           -- keeps Ids 1 and 2
SELECT * FROM #Emails;                -- 1 a@x.com | 2 b@x.com
```

### 17.3 [Medium] Employees earning more than their manager; running totals.
**Answer**
Self-join employee to manager and compare salary. For a running total use `SUM() OVER (ORDER BY ... ROWS UNBOUNDED PRECEDING)`.

**Explanation**
Specify `ROWS` explicitly: the default frame with `ORDER BY` is `RANGE UNBOUNDED PRECEDING`, which treats ties as peers (rows with equal ORDER BY value get the same total) and is slower (spools to disk). Add a tiebreaker column to the ORDER BY.

**Code**
```sql
SELECT e.Name, e.Salary, m.Name AS Manager, m.Salary AS MgrSalary
FROM Employees e
JOIN Employees m ON m.EmpId = e.ManagerId
WHERE e.Salary > m.Salary;
-- Esha 6000 | Dev 5000

SELECT OrderId, OrderDate, Amount,
       SUM(Amount) OVER (ORDER BY OrderDate, OrderId ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW) AS RunningTotal
FROM Orders;
-- 1 2024-01-05 100.00 100.00
-- 2 2024-01-20 250.00 350.00
-- 3 2024-02-02  75.50 425.50
-- 4 2024-02-15 300.00 725.50
-- 5 2024-03-01  50.00 775.50
```

### 17.4 [Hard] Top N per group (top 2 earners per department).
**Answer**
Rank within each partition using `ROW_NUMBER/DENSE_RANK() OVER (PARTITION BY group ORDER BY metric DESC)` in a CTE, then filter `rn <= N`. Choose `DENSE_RANK` or `RANK` if ties should be included, or `CROSS APPLY (SELECT TOP (N) ...)`.

**Explanation**
Window functions can't be filtered directly (Topic 3). An index on `(DeptId, Salary DESC)` makes this efficient; `CROSS APPLY` + TOP is great when groups are few and the index exists. A `DeptId` of NULL forms its own partition.

**Code**
```sql
WITH r AS (
    SELECT Name, DeptId, Salary,
           DENSE_RANK() OVER (PARTITION BY DeptId ORDER BY Salary DESC) AS rnk
    FROM Employees
)
SELECT DeptId, Name, Salary, rnk FROM r WHERE rnk <= 2 ORDER BY DeptId, rnk;
-- NULL Esha 6000 1
-- 1 Asha 9000 1 | 1 Farid 8000 2
-- 2 Chen 7000 1 | 2 Dev 5000 2
```

### 17.5 [Tricky] Gaps and islands (introduction).
**Answer**
Islands are runs of consecutive values; gaps are the missing ranges. For islands, subtract `ROW_NUMBER()` from the value: consecutive values share the same difference, which you group on.

**Explanation**
`value - ROW_NUMBER()` is constant inside a consecutive run, so `GROUP BY` that grouping key and take `MIN/MAX`. Gaps can be found with `LEAD()`: where the next value minus current is greater than 1. Dates work the same way using `DATEADD(day, -ROW_NUMBER(), date)`.

**Code**
```sql
CREATE TABLE #Seq (Id INT PRIMARY KEY);
INSERT #Seq VALUES (1),(2),(3),(7),(8),(10);

-- Islands
SELECT MIN(Id) AS StartId, MAX(Id) AS EndId
FROM (SELECT Id, Id - ROW_NUMBER() OVER (ORDER BY Id) AS grp FROM #Seq) x
GROUP BY grp ORDER BY StartId;
-- 1-3 | 7-8 | 10-10

-- Gaps
SELECT Id + 1 AS GapStart, NextId - 1 AS GapEnd
FROM (SELECT Id, LEAD(Id) OVER (ORDER BY Id) AS NextId FROM #Seq) x
WHERE NextId - Id > 1;
-- 4-6 | 9-9
```

**Gotchas / follow-ups**
- Duplicates in the sequence break the `ROW_NUMBER` trick - use `DENSE_RANK` or de-duplicate first.
- Show the "Nth highest per dept" and "employees with no orders" variants too.
- Median: `PERCENTILE_CONT(0.5) WITHIN GROUP (ORDER BY ...) OVER (...)`.

**Interview traps**
- `ROW_NUMBER` for "second highest" with ties.
- Default RANGE frame for running totals.
- Deleting duplicates without defining which row survives.
- `NOT IN` in "no orders" queries (Topic 6).
- Filtering a window function in WHERE.

**Cheat sheet**
- 2nd highest: `DENSE_RANK() = 2` or `MAX(... < MAX)`.
- Duplicates: `ROW_NUMBER() ... DELETE rn > 1` in a CTE.
- Running total: `SUM() OVER (ORDER BY k ROWS UNBOUNDED PRECEDING)`.
- Top N per group: `PARTITION BY` + rank filter / `CROSS APPLY TOP`.
- Islands: `value - ROW_NUMBER()` grouping key.

---

## Topic 18: SQL injection basics

**Study notes**

SQL injection happens when untrusted input is concatenated into SQL text so it is parsed as code.

```
 input:  ' OR 1=1 --
 SQL  :  SELECT * FROM Users WHERE Name = '' OR 1=1 --' AND Pwd = '...'
```

Defences: **parameterized queries**, least-privilege DB accounts, input validation/whitelisting, safe handling of identifiers (`QUOTENAME`), avoiding dynamic SQL, stored procedures (only safe if they themselves don't build unsafe dynamic SQL).

### 18.1 [Easy] What is SQL injection and how do you prevent it?
**Answer**
It is an attack where malicious input alters the structure of a SQL statement. Prevent it by always using parameterized queries (`SqlCommand.Parameters`, `sp_executesql` with parameters, EF Core LINQ) rather than string concatenation.

**Explanation**
Parameters are sent as typed values separate from the SQL text, so they can never become code. Escaping quotes by hand is error-prone and not recommended.

**Code**
```sql
-- Vulnerable
DECLARE @name NVARCHAR(50) = N''' OR 1=1 --';
EXEC (N'SELECT * FROM Employees WHERE Name = ''' + @name + N'''');   -- returns ALL rows

-- Safe
EXEC sp_executesql N'SELECT * FROM Employees WHERE Name = @n', N'@n NVARCHAR(50)', @n = @name;  -- 0 rows
```

### 18.2 [Follow-up] Are stored procedures automatically safe?
**Answer**
No. A stored procedure that concatenates parameters into a string and runs `EXEC(@sql)` is just as injectable. Procedures are safe only when they use static SQL or parameterized dynamic SQL via `sp_executesql`.

**Explanation**
Procedures help with permissions (ownership chaining lets you deny direct table access) and plan reuse, not injection by themselves.

**Code**
```sql
CREATE PROCEDURE dbo.FindBad @name NVARCHAR(50) AS
    EXEC (N'SELECT * FROM Employees WHERE Name = ''' + @name + N'''');   -- injectable

CREATE PROCEDURE dbo.FindGood @name NVARCHAR(50) AS
    SELECT * FROM Employees WHERE Name = @name;                          -- safe
```

### 18.3 [Medium] How do you handle dynamic identifiers (table/column names, ORDER BY)?
**Answer**
Parameters can't be used for object names or sort direction. Whitelist allowed values, and wrap identifiers with `QUOTENAME()` when building dynamic SQL. Better: map user choices to fixed constants in code.

**Explanation**
`QUOTENAME` doubles embedded `]` characters, producing a safely delimited identifier. It still doesn't validate that the object is allowed - whitelist as well. Direction should be mapped to `ASC`/`DESC` literals, not user text.

**Code**
```sql
DECLARE @col SYSNAME = N'Salary';
IF @col NOT IN (N'Name', N'Salary', N'HireDate') THROW 50001, 'Invalid sort column', 1;
DECLARE @sql NVARCHAR(MAX) = N'SELECT Name, Salary FROM Employees ORDER BY ' + QUOTENAME(@col) + N' DESC';
EXEC sp_executesql @sql;
```

### 18.4 [Hard] Types of SQL injection and EF Core specifics
**Answer**
In-band (error/union based), blind (boolean/time based) and second-order (payload stored first, executed later). EF Core LINQ queries are parameterized by default; `FromSqlRaw` with concatenated strings is vulnerable, while `FromSql`/`FromSqlInterpolated` with interpolated values turn them into parameters.

**Explanation**
Second-order: a username like `x'; DROP TABLE...` is stored safely but later concatenated into a different query by a trusted job. `ExecuteSqlRaw` has the same caution as `FromSqlRaw`. Also `LIKE` input needs wildcard escaping to avoid unintended pattern matches (not injection but related).

**Code**
```sql
-- Time-based blind probe an attacker may send:
-- ' ; IF (SELECT COUNT(*) FROM Employees) > 5 WAITFOR DELAY '0:0:5' --

-- Parameterized LIKE with escaped wildcards
DECLARE @term NVARCHAR(50) = N'50%';
SELECT Name FROM Employees WHERE Name LIKE REPLACE(REPLACE(@term,'[','[[]'),'%','[%]') + N'%';
```
```csharp
// EF Core: safe (parameterized)
var emps = db.Employees.FromSql($"SELECT * FROM Employees WHERE Name = {name}");
// Unsafe: FromSqlRaw("... '" + name + "'")
```

### 18.5 [Tricky] Defence in depth: least privilege and what else reduces blast radius?
**Answer**
Application accounts should have only `EXECUTE`/DML rights they need (no `db_owner`, never `sa`), no `xp_cmdshell`, separate read/write roles, and errors should not leak schema details. Parameterization is primary; the rest limits damage.

**Explanation**
Use contained users or Windows/managed identity auth, encrypt connections, audit failed logins, and keep secrets out of source code. WAFs are a supplementary control, not a substitute. Consider Row-Level Security/Always Encrypted for sensitive data.

**Code**
```sql
CREATE ROLE AppRole;
GRANT SELECT, INSERT, UPDATE ON SCHEMA::dbo TO AppRole;
DENY  DELETE, ALTER ON SCHEMA::dbo TO AppRole;
-- Prefer: GRANT EXECUTE ON SCHEMA::app TO AppRole;  with logic in procedures
```

**Interview traps**
- "Escape the quotes" as the main defence.
- Believing stored procedures are inherently safe.
- Parameterizing values but not identifiers.
- Running the app as `sa`/`db_owner`.
- `FromSqlRaw` with string interpolation (use `FromSql`/`FromSqlInterpolated`).

**Cheat sheet**
- Always parameterize.
- `sp_executesql` with params for dynamic SQL.
- `QUOTENAME` + whitelist for identifiers.
- Least privilege accounts.
- Second-order injection exists.

---

## Topic 19: SQL Server editions and architecture overview

**Study notes**

```
 Database
  ├─ Primary data file   .mdf  (one)
  ├─ Secondary data files .ndf (optional)
  └─ Transaction log     .ldf  (sequence of Virtual Log Files)

 File -> Extents (8 pages = 64 KB) -> Pages (8 KB)
 Page = 96-byte header + data rows (max ~8060 bytes in-row) + row-offset array
```

- **Page** (8 KB) is the unit of I/O. **Extent** = 8 contiguous pages (64 KB). *Uniform* extents belong to one object; *mixed* extents are shared by small objects (since 2016, user databases default to uniform extents).
- Allocation maps: GAM, SGAM, PFS, IAM.
- **System databases**: `master` (instance metadata), `model` (template), `msdb` (Agent jobs, backup history), `tempdb` (temp objects, row versions, sorts - recreated on restart), `Resource` (hidden system objects).
- **Editions** (verify limits per version): Enterprise (all features), Standard (core features, lower scale limits), Developer (full Enterprise features, non-production, free), Express (free, small: DB size cap ~10 GB, limited CPU/memory), Web (hosting). Cloud: Azure SQL Database, Managed Instance, SQL Server on VMs.
- **Recovery models**: Simple, Full, Bulk-logged.

### 19.1 [Easy] What files make up a SQL Server database?
**Answer**
A primary data file (`.mdf`), optional secondary data files (`.ndf`) and at least one transaction log file (`.ldf`). Data files can be grouped into filegroups.

**Explanation**
The log is separate from data and is written sequentially. Putting tempdb, log and data on separate fast storage is common practice. Memory-optimized (In-Memory OLTP) and FILESTREAM data add their own containers.

**Code**
```sql
SELECT name, type_desc, size * 8 / 1024 AS SizeMB, physical_name
FROM sys.database_files;
-- ROWS (.mdf/.ndf) and LOG (.ldf) rows
```

### 19.2 [Follow-up] What are pages and extents?
**Answer**
A page is the 8 KB unit of storage and I/O; an extent is eight contiguous pages (64 KB), the unit of space allocation. Rows are stored on pages; a row (non-LOB) must fit in about 8060 bytes in-row, with overflow going to row-overflow/LOB pages.

**Explanation**
Page types: data, index, IAM, GAM/SGAM, PFS, LOB. Index key structures are B-trees whose leaf pages are the data (clustered) or pointers (nonclustered). Page splits happen when a row needs to go into a full page - fragmentation. Fill factor leaves free space per page.

**Code**
```sql
SELECT OBJECT_NAME(object_id) AS TableName, index_type_desc,
       page_count, avg_fragmentation_in_percent
FROM sys.dm_db_index_physical_stats(DB_ID(), OBJECT_ID('dbo.Employees'), NULL, NULL, 'LIMITED');
```

### 19.3 [Medium] How does the transaction log provide durability (write-ahead logging)?
**Answer**
Changes are applied to data pages in the buffer pool (memory) and the log record is flushed to the log file *before* the transaction commits (write-ahead logging). Dirty data pages are written to disk later by checkpoint/lazy writer. On crash, recovery redoes committed work and undoes uncommitted work from the log.

**Explanation**
Commit = log record hardened to disk, not data pages. Recovery has analysis, redo and undo phases. Log space is reused after it is *truncated*: in Simple recovery after checkpoint; in Full after a log backup. A long-open transaction (or missing log backups, replication, AG lag) prevents truncation and makes the log grow.

**Code**
```sql
SELECT name, recovery_model_desc, log_reuse_wait_desc FROM sys.databases;
-- log_reuse_wait_desc: NOTHING / LOG_BACKUP / ACTIVE_TRANSACTION / ...
CHECKPOINT;
```

### 19.4 [Hard] Recovery models and what they mean for backups
**Answer**
Simple: log auto-truncates at checkpoint, no point-in-time restore. Full: all operations fully logged, requires log backups, enables point-in-time restore. Bulk-logged: like Full but bulk operations are minimally logged, trading point-in-time granularity during those operations for speed.

**Explanation**
Typical production OLTP uses Full. A classic problem: Full recovery with no log backups -> log grows until disk is full. Backup types: full, differential, log, copy-only. A backup strategy should also include restore tests.

**Code**
```sql
ALTER DATABASE MyDb SET RECOVERY FULL;
BACKUP DATABASE MyDb TO DISK = N'D:\bak\MyDb.bak' WITH CHECKSUM;
BACKUP LOG      MyDb TO DISK = N'D:\bak\MyDb.trn';
```

### 19.5 [Tricky] tempdb, system databases and common architecture gotchas
**Answer**
`tempdb` is recreated at every restart and holds temp tables, table variables, sort/hash spills, version store (for RCSI/snapshot) and internal objects, so it can become a contention/growth hotspot. Never store permanent data there. Multiple equally sized tempdb data files reduce allocation contention (default setup does this in 2016+).

**Explanation**
Also: `master` loss prevents instance start (back it up); `model` changes affect every new database (including tempdb's initial settings); `msdb` stores Agent jobs. Autogrowth in tiny increments causes many VLFs and slow recovery; percentage-based growth on huge files can be excessive. Shrinking files causes fragmentation - avoid routine shrink. The logical engine layers are: protocol layer, relational engine (parser, optimizer, query executor) and storage engine (access methods, buffer manager, transaction manager).

**Code**
```sql
SELECT name, size * 8 / 1024 AS SizeMB FROM tempdb.sys.database_files;
SELECT name FROM sys.databases WHERE database_id <= 4;   -- master, tempdb, model, msdb
```

**Interview traps**
- Believing the log is optional or that commit writes data pages to disk.
- Full recovery without log backups.
- Calling an extent 8 KB (it's 64 KB; page = 8 KB).
- Regularly shrinking databases.
- Quoting Express limits from memory without checking the version.

**Cheat sheet**
- .mdf = primary data; .ndf = secondary; .ldf = log.
- Page 8 KB; extent 8 pages = 64 KB.
- WAL: log hardened before commit; data pages lazily written.
- Simple/Full/Bulk-logged recovery models.
- tempdb recreated at restart.

---

*End of D01. Next: indexing, execution plans and performance (see subsequent files).*




