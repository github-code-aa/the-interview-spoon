# D03 - SQL Server Indexing, Execution Plans & Performance Tuning

**How to use:** Read each topic's *Study notes* first, then try to answer each ladder question out loud before reading the **Answer**. Rungs go Easy -> Follow-up -> Medium -> Hard -> Tricky; stop at the rung where you stumble and revisit that topic.
**Tip:** Run the `Code` blocks against a scratch database (e.g. AdventureWorks / your own) and look at the actual plans (Ctrl+M in SSMS) - seeing is what makes this stick.

## Table of Contents
1. Topic 1: How SQL Server Stores Data (Pages, Extents, Heap vs B-tree, Fill Factor)
2. Topic 2: Clustered vs Non-Clustered Indexes & Choosing the Clustering Key
3. Topic 3: Covering Indexes, INCLUDE Columns & Composite Index Order
4. Topic 4: Seek vs Scan, Key/RID Lookups
5. Topic 5: SARGability (Functions, Implicit Conversions, LIKE, OR) & EF nvarchar/varchar Mismatch
6. Topic 6: Filtered, Unique & Columnstore Indexes
7. Topic 7: Fragmentation, Rebuild vs Reorganize, Large-Table Maintenance
8. Topic 8: Statistics, Histograms & Cardinality Estimation
9. Topic 9: Reading Execution Plans (Operators, Joins, Warnings)
10. Topic 10: Parameter Sniffing & Fixes
11. Topic 11: Plan Cache, Recompilations & Query Store
12. Topic 12: DMVs, Wait Statistics, SET STATISTICS IO/TIME
13. Topic 13: Partitioning, Compression & In-Memory OLTP
14. Topic 14: Index Overhead, Paging, Hints, NOLOCK & tempdb
15. Topic 15: Tuning Methodology & Scenario Ladders (slow today, timing-out report, slow EF query)

---

## Topic 1: How SQL Server Stores Data (Pages, Extents, Heap vs B-tree, Fill Factor)

**Study notes**

- The unit of I/O and storage is the **page**: 8 KB (8192 bytes). 96-byte header, up to ~8060 bytes of row data, row-offset array at the end of the page.
- An **extent** = 8 contiguous pages (64 KB). *Uniform* extents belong to one object; *mixed* extents (shared) were used for small objects (since SQL Server 2016, user databases default to uniform extents; `MIXED_PAGE_ALLOCATION` is OFF).
- Rows normally live in **IN_ROW_DATA** pages. Large values (varchar(max), etc.) can go to **LOB_DATA**; rows exceeding 8060 bytes can spill variable-length columns to **ROW_OVERFLOW_DATA**.
- A table is either a **heap** (no clustered index; unordered pages, found via IAM pages, rows identified by RID = file:page:slot) or a **clustered table** (data *is* the leaf level of a B-tree ordered by clustering key).
- **B-tree (rowstore index)**:

```
                 [ Root page ]                 <- 1 page, key ranges -> child page
                /      |       \
        [Intermediate] [Intermediate] [Intermediate]
         /    |   \
   [Leaf]<->[Leaf]<->[Leaf]<->[Leaf] ...      <- doubly linked list, sorted by key
   (clustered: leaf = full data rows; nonclustered: leaf = key + clustering key/RID + INCLUDEs)
```
- Depth is usually 3-4 levels even for hundreds of millions of rows, so a seek costs ~3-4 logical reads.
- **Fill factor**: % of leaf-page space filled during index create/rebuild (default 0 = 100). Lower values leave free space to delay **page splits** on inserts/updates into the middle of the key range. PAD_INDEX applies it to intermediate levels too.
- **Page split**: when a row must go on a full page, SQL Server allocates a new page and moves ~half the rows -> fragmentation + extra log.

### 1.1 [Easy] What is a page and an extent in SQL Server?
**Answer**
A page is the 8 KB basic unit of storage and I/O; an extent is 8 contiguous pages (64 KB). SQL Server reads and writes whole pages, and buffer pool caching is also page-based.

**Explanation**
Every table/index is a collection of pages of types like data, index, IAM, GAM/SGAM/PFS (allocation maps), LOB. The 8060-byte in-row limit is why very wide rows push data to row-overflow or LOB pages. Allocation is done in extents to keep related pages physically close, which helps read-ahead.

**Code**
```sql
-- Page counts and types for a table
SELECT OBJECT_NAME(ps.object_id) AS tbl, i.name AS idx, ps.index_level,
       ps.page_count, ps.avg_page_space_used_in_percent, ps.alloc_unit_type_desc
FROM sys.dm_db_index_physical_stats(DB_ID(), OBJECT_ID('dbo.Orders'), NULL, NULL, 'DETAILED') ps
JOIN sys.indexes i ON i.object_id = ps.object_id AND i.index_id = ps.index_id;
```

### 1.2 [Follow-up] What is the difference between a heap and a clustered table?
**Answer**
A heap has no clustered index; its pages are unordered and rows are located by RID (file:page:slot). A clustered table stores rows in the leaf level of a B-tree sorted by the clustering key, so the data itself is the clustered index.

**Explanation**
Heaps: fast bulk inserts/staging loads, but range queries need a nonclustered index plus RID lookups; deletes can leave empty pages; updates that grow a row cause **forwarded records** (a pointer left at the old location), hurting scans. Clustered: ordered range scans, key lookups use the clustering key, and the index can be rebuilt/reorganized. Most OLTP tables should be clustered; heaps are for staging/ETL.

**Code**
```sql
-- Find heaps (index_id = 0) and forwarded records
SELECT OBJECT_NAME(ps.object_id) AS tbl, ps.page_count, ps.forwarded_record_count
FROM sys.dm_db_index_physical_stats(DB_ID(), NULL, NULL, NULL, 'DETAILED') ps
WHERE ps.index_id = 0 AND ps.alloc_unit_type_desc = 'IN_ROW_DATA';

-- Fix forwarded records on a heap
ALTER TABLE dbo.StagingOrders REBUILD;
```

### 1.3 [Medium] Describe the structure of a B-tree index and why a seek on a billion rows is only a few reads.
**Answer**
It has a single root page, zero or more intermediate levels, and a leaf level linked in key order. Each non-leaf page holds many key/child-pointer entries (hundreds per page), so fan-out is huge and depth is typically 3-4; a seek is one read per level.

**Explanation**
If a key is 8 bytes (bigint) plus pointer overhead, a page holds ~400+ entries, so 400^3 = 64 million leaf pages addressable with 3 levels. Seek = navigate root->leaf, then range scan along the leaf links. Root and upper levels are almost always in memory, so physical reads are usually only for the leaf. Wider keys reduce fan-out and deepen the tree, which is one reason for narrow clustering keys.

**Code**
```sql
-- Depth and page counts per level
SELECT index_id, index_depth, index_level, page_count, record_count
FROM sys.dm_db_index_physical_stats(DB_ID(), OBJECT_ID('dbo.Orders'), 1, NULL, 'DETAILED');
-- Expected: index_level 0 = leaf (large page_count), top level = 1 root page.
```

### 1.4 [Hard] What does fill factor do, when would you lower it, and what are the downsides?
**Answer**
Fill factor leaves a percentage of each leaf page empty at index build/rebuild time to absorb future inserts/updates and delay page splits. Lower it (e.g. 80-90) for indexes with random-key inserts or growing-row updates; keep it at 100 for ever-increasing keys and read-mostly tables. Downside: more pages -> more I/O, more memory, larger backups.

**Explanation**
Fill factor is only applied at build/rebuild/reorganize-time, not maintained on ongoing inserts. An append-only identity clustered key never splits mid-page, so lowering fill factor there just wastes space. For a random GUID key, every insert lands on a random page, so some slack helps but the index still fragments. A server-wide default exists (`sp_configure 'fill factor (%)'`), but setting it per index is better. Free space of 10% means ~11% more pages scanned.

**Code**
```sql
ALTER INDEX IX_Customers_Guid ON dbo.Customers
REBUILD WITH (FILLFACTOR = 85, PAD_INDEX = ON);

-- Check current setting
SELECT name, fill_factor FROM sys.indexes WHERE object_id = OBJECT_ID('dbo.Customers');
```

**Gotchas / follow-ups**
- `fill_factor = 0` in sys.indexes means 100%.
- Page splits are cheaper for ever-increasing keys (new page allocated at end, no row movement).
- Monitor `sys.dm_db_index_operational_stats` (`leaf_allocation_count`) to see splits; the "Page Splits/sec" counter includes harmless end-of-index allocations.

### 1.5 [Tricky] A table has an nvarchar(max)/varchar(max) column. Why can a "SELECT *" be dramatically slower than selecting other columns, and where does the data live?
**Answer**
Large values may be stored off-row in LOB_DATA pages (depending on size and the `large value types out of row` option), so reading them requires extra page reads per row. Selecting only needed columns avoids touching LOB pages entirely.

**Explanation**
By default, max-type values are in-row if they fit (up to 8000 bytes); otherwise a pointer to LOB pages is stored. Variable-length non-max columns that don't fit push to ROW_OVERFLOW_DATA. Scans that avoid these columns never read the LOB allocation unit. Also, `SELECT *` kills covering indexes. EF projection (`Select(x => new {...})`) avoids this; entity loading pulls all columns.

**Code**
```sql
-- Which allocation units does a table have, and how big are they?
SELECT au.type_desc, au.total_pages, au.used_pages
FROM sys.allocation_units au
JOIN sys.partitions p ON au.container_id = p.partition_id
WHERE p.object_id = OBJECT_ID('dbo.Documents');

-- Force max types off-row
EXEC sp_tableoption 'dbo.Documents', 'large value types out of row', 1;
```

**Gotchas / follow-ups**
- Changing the option affects only newly written values until rows are updated.
- Rows >8060 bytes with fixed-length columns are not allowed (error at create/insert).

**Interview traps**
- Saying "an extent is 8 KB" (page = 8 KB, extent = 64 KB).
- Believing heaps are always bad; they're fine for staging/bulk-load but bad for lookups.
- Thinking fill factor is maintained continuously; it applies only at (re)build.
- Claiming B-trees are "binary trees"; they're balanced, wide, multiway trees.

**Cheat sheet**
- Page 8 KB, extent 8 pages = 64 KB; in-row limit ~8060 bytes.
- Heap = RID (file:page:slot); clustered = data in B-tree leaf.
- B-tree depth 3-4 even for billions of rows.
- Fill factor applies at rebuild; helps random inserts, wastes space on append-only keys.
- Forwarded records = heap-only problem; fix with `ALTER TABLE ... REBUILD`.

---

## Topic 2: Clustered vs Non-Clustered Indexes & Choosing the Clustering Key

**Study notes**

- **Clustered index**: defines the physical (logical-linked) order of the table; only **one** per table. Leaf = data rows.
- **Nonclustered index (NCI)**: separate B-tree; leaf holds the index key columns + a **row locator** (the clustering key on a clustered table; the RID on a heap) + any INCLUDE columns.
- Hence **the clustering key is silently copied into every NCI** (and into every NCI's non-leaf levels if the NCI is non-unique). A wide clustering key bloats all indexes.

```
 NCI on (LastName)                         Clustered index on (CustomerId)
 leaf: LastName | CustomerId  ---lookup--> leaf: CustomerId | all columns
```
- **Ideal clustering key**: *narrow*, *unique*, *static* (never updated), *ever-increasing* (append-only, so inserts go at the end). Mnemonic: NUSE.
- If the clustered index is not unique, SQL Server adds a hidden 4-byte **uniquifier** to duplicate keys.
- A PRIMARY KEY creates a clustered index by default, but the PK and clustered index are independent choices.
- **GUID keys**: `NEWID()` is random -> inserts scatter across the whole index -> page splits, low page density, fragmentation, more buffer pool pressure. `NEWSEQUENTIALID()` (only as a column DEFAULT) is increasing within a machine boot sequence, reduces this. 16 bytes also widens every NCI.

### 2.1 [Easy] What is the difference between a clustered and a non-clustered index?
**Answer**
A clustered index sorts and stores the table's data rows in key order (one per table). A nonclustered index is a separate structure with key columns and a pointer back to the row (clustering key or RID); a table can have many (up to 999).

**Explanation**
Because the clustered index *is* the table, there's no extra copy of data. NCIs duplicate the columns they include, so they cost storage and write time. Queries that need columns not in the NCI do a lookup into the clustered index.

**Code**
```sql
CREATE TABLE dbo.Orders (
  OrderId     bigint IDENTITY(1,1) NOT NULL,
  CustomerId  int NOT NULL,
  OrderDate   datetime2(0) NOT NULL,
  Total       decimal(12,2) NOT NULL,
  CONSTRAINT PK_Orders PRIMARY KEY CLUSTERED (OrderId)
);
CREATE NONCLUSTERED INDEX IX_Orders_Customer ON dbo.Orders (CustomerId);
```

### 2.2 [Follow-up] What makes a good clustering key?
**Answer**
Narrow, unique, static, and ever-increasing. Narrow because it's copied into every NCI; unique to avoid the uniquifier; static to avoid row moves and NCI updates; ever-increasing so inserts append and avoid page splits.

**Explanation**
An `int`/`bigint IDENTITY` meets all four. Natural keys like email or composite business keys are usually wide and mutable. Exceptions: a time-series table clustered on (EventDate, Id) is deliberately chosen for range queries/partition alignment; a lookup table queried by code might cluster on that code. It's a tradeoff, not a law - but you must justify deviations.

**Code**
```sql
-- Size impact: wide clustering key bloats every NCI
SELECT i.name, SUM(ps.used_page_count)*8/1024 AS MB
FROM sys.dm_db_partition_stats ps
JOIN sys.indexes i ON i.object_id=ps.object_id AND i.index_id=ps.index_id
WHERE ps.object_id = OBJECT_ID('dbo.Orders')
GROUP BY i.name;
```

### 2.3 [Medium] Why are random GUID clustered keys a problem, and what are the alternatives?
**Answer**
`NEWID()` values arrive in random order, so each insert targets a random leaf page; pages fill up and split, leaving ~50-70% density, heavy fragmentation, extra logging and a larger memory footprint. Alternatives: `NEWSEQUENTIALID()` default, a surrogate `bigint IDENTITY` as clustered key with the GUID as a nonclustered unique index, or generating sequential GUIDs in the app (e.g. COMB/`UuidCreateSequential`-style).

**Explanation**
The 16-byte width also bloats every NCI. If GUIDs are needed for external exposure/distributed generation, cluster on an identity and keep the GUID unique NCI. In EF Core, `Guid` keys against SQL Server use a sequential GUID value generator by default on the client (`SequentialGuidValueGenerator`), which mitigates fragmentation compared to `Guid.NewGuid()`. `NEWSEQUENTIALID()` can only be used in a DEFAULT constraint and values are predictable (security consideration); sequence may restart after a Windows restart but stays unique.

**Code**
```sql
CREATE TABLE dbo.Customers (
  CustomerId  bigint IDENTITY(1,1) NOT NULL,
  PublicId    uniqueidentifier NOT NULL CONSTRAINT DF_Cust_Pub DEFAULT NEWSEQUENTIALID(),
  Name        nvarchar(200) NOT NULL,
  CONSTRAINT PK_Customers PRIMARY KEY CLUSTERED (CustomerId),
  CONSTRAINT UQ_Customers_Pub UNIQUE NONCLUSTERED (PublicId)
);
```

### 2.4 [Hard] How does the clustering key affect nonclustered indexes, and what happens to NCIs when you change a clustered index?
**Answer**
Every NCI row carries the clustering key as its row locator. Dropping or changing the clustered index (or heap<->clustered) forces all NCIs to be rebuilt because the locator changes (RID <-> key); a wide/unique-vs-nonunique clustering key changes NCI size and structure.

**Explanation**
In a non-unique NCI, the clustering key is added to both leaf and non-leaf levels; in a unique NCI it is only in the leaf (as a hidden extra column), so non-leaf stays narrower. You can seek on the NCI with the clustering key as an implicit trailing key column, e.g. `WHERE CustomerId = @c AND OrderId > @o` can seek on IX(CustomerId) since OrderId is in the key. Rebuilding a clustered index with `CREATE CLUSTERED INDEX ... WITH (DROP_EXISTING = ON)` avoids rebuilding NCIs twice when keys remain the same; if the key changes, NCIs rebuild once.

**Code**
```sql
-- Change clustering key without dropping/recreating separately
CREATE UNIQUE CLUSTERED INDEX PK_Orders ON dbo.Orders (OrderDate, OrderId)
WITH (DROP_EXISTING = ON, ONLINE = ON);  -- ONLINE requires Enterprise (or Azure SQL)
```

**Gotchas / follow-ups**
- PK constraint backed by clustered index can't be changed with CREATE INDEX DROP_EXISTING; you must drop/recreate the constraint (or use the constraint syntax as appropriate).
- FK references to the PK must be dropped before dropping the PK.

### 2.5 [Tricky] Your clustered index is on an IDENTITY column but the table is hammered by inserts and you see PAGELATCH_EX waits. What's happening and what can you do?
**Answer**
This is **last-page insert contention**: all inserts target the last leaf page of the clustered index, so concurrent sessions contend on its latch. Options: `OPTIMIZE_FOR_SEQUENTIAL_KEY = ON` (SQL Server 2019+), hash-partitioning the key, or using a non-sequential leading key column to spread inserts - accepting tradeoffs.

**Explanation**
Ever-increasing keys are ideal for fragmentation but concentrate inserts on one page. `OPTIMIZE_FOR_SEQUENTIAL_KEY` throttles threads at the latch to reduce convoys (shown as `BTREE_INSERT_FLOW_CONTROL` waits). The alternative of a leading hash/shard column spreads load but loses pure append behavior and makes range scans on the key harder. In-Memory OLTP (hash indexes) is another route.

**Code**
```sql
ALTER INDEX PK_Orders ON dbo.Orders SET (OPTIMIZE_FOR_SEQUENTIAL_KEY = ON);

-- Observe contention
SELECT TOP 10 wait_type, wait_time_ms, waiting_tasks_count
FROM sys.dm_os_wait_stats
WHERE wait_type LIKE 'PAGELATCH%' OR wait_type = 'BTREE_INSERT_FLOW_CONTROL'
ORDER BY wait_time_ms DESC;
```

**Gotchas / follow-ups**
- PAGELATCH_* (in-memory page latch) differs from PAGEIOLATCH_* (disk I/O).
- tempdb allocation contention also shows as PAGELATCH_* - check the resource (`sys.dm_exec_requests.wait_resource`: 2:1:1 = PFS, 2:1:2 = GAM, 2:1:3 = SGAM).

**Interview traps**
- "Primary key = clustered index": only by default.
- Forgetting that the clustering key is in every NCI.
- Recommending `NEWID()` as clustered key without caveats.
- Believing sequential keys have no scaling issue (last-page contention).
- Saying uniquifier is 8 bytes (it is 4 bytes, only for duplicate keys in non-unique clustered index).

**Cheat sheet**
- One clustered per table, up to 999 NCIs.
- NUSE: Narrow, Unique, Static, Ever-increasing.
- NCI locator = clustering key (or RID on heap).
- NEWID = random -> splits; NEWSEQUENTIALID only as DEFAULT.
- PAGELATCH_EX on last page -> OPTIMIZE_FOR_SEQUENTIAL_KEY (2019+).

---

## Topic 3: Covering Indexes, INCLUDE Columns & Composite Index Order

**Study notes**

- A **covering index** contains all columns a query needs (in keys or INCLUDEs) so no lookup into the base table is required.
- **Key columns** are in the B-tree at all levels and sorted; limit 16 columns / 900 bytes clustered (1700 bytes for NCI). **INCLUDE columns** exist only at the leaf, are unsorted, don't count toward the key size limit, and can be types (like `nvarchar(max)`) that can't be keys.
- **Composite index order** matters: the index is sorted by col1, then col2 within col1, etc. A seek needs a predicate on the **leftmost** column(s) (leftmost-prefix rule).

```
IX (A, B, C):     A=1,B=1,C=..  | A=1,B=2 | A=2,B=1 ...
 WHERE A=1                    -> seek
 WHERE A=1 AND B=5            -> seek
 WHERE A=1 AND C=7            -> seek on A, residual predicate on C
 WHERE B=5                    -> scan (or skip-scan not supported)
 WHERE A>1 AND B=5            -> seek on A range; B is a residual filter
```
- Rule of thumb for order: equality columns first, then one range/inequality column, then ORDER BY columns if they can be satisfied; selectivity is secondary to usage pattern (but among equalities, put the more selective/most reused first).
- Index order can also eliminate a **Sort** operator when ORDER BY matches key order (or exact reverse via backward scan).

### 3.1 [Easy] What is a covering index?
**Answer**
An index that includes every column the query references (SELECT, WHERE, JOIN, ORDER BY) so the engine satisfies it from the index alone, without key lookups. It's usually a nonclustered index with INCLUDE columns.

**Explanation**
Eliminating lookups turns many random I/Os into one seek plus a short range scan. The cost is a bigger index and more write overhead, so cover your hottest queries, not everything.

**Code**
```sql
CREATE NONCLUSTERED INDEX IX_Orders_Cust_Date
ON dbo.Orders (CustomerId, OrderDate)
INCLUDE (Total);

-- Covered: plan shows Index Seek only, no Key Lookup
SELECT OrderDate, Total FROM dbo.Orders WHERE CustomerId = 42 AND OrderDate >= '2025-01-01';
```

### 3.2 [Follow-up] What's the difference between key columns and INCLUDE columns?
**Answer**
Key columns define sort order and are present at all B-tree levels, so they support seeks, sorting and uniqueness. INCLUDE columns are stored only at the leaf, are not ordered, and just "ride along" to cover queries.

**Explanation**
Use key columns for those in WHERE/JOIN/ORDER BY; use INCLUDE for columns only in SELECT. This keeps non-leaf pages small (more fan-out) and removes the 900/1700 byte key limit constraints. Include columns can't be used for seeking but can be filtered as residual predicates within the leaf scan. Updating an included column still updates the index.

**Code**
```sql
-- Wide column belongs in INCLUDE, not key
CREATE INDEX IX_Products_Cat ON dbo.Products (CategoryId) INCLUDE (Name, Description);
```

### 3.3 [Medium] Explain the leftmost-prefix rule and how you'd order columns in a composite index for `WHERE Status = @s AND CreatedAt >= @d ORDER BY CreatedAt`.
**Answer**
An index can be seeked only on a contiguous leftmost set of its key columns. Put equality (`Status`) first, then the range/sort column (`CreatedAt`): `(Status, CreatedAt)`. This seeks to the status and reads dates in order, avoiding a sort.

**Explanation**
Reversing to `(CreatedAt, Status)` would seek on the date range and filter Status residually across the whole range, and the order would not match for ORDER BY within a status. Once a range predicate is used on a column, subsequent key columns can't contribute to the seek (they filter residually, shown as "Predicate" vs "Seek Predicates" in the plan). SQL Server has no index skip-scan, so a query on only `CreatedAt` can't seek into `(Status, CreatedAt)` efficiently (it may scan the index).

**Code**
```sql
CREATE INDEX IX_Tickets_Status_Created ON dbo.Tickets (Status, CreatedAt) INCLUDE (Title);

SELECT TOP (50) Title, CreatedAt
FROM dbo.Tickets
WHERE Status = 2 AND CreatedAt >= '2025-06-01'
ORDER BY CreatedAt;   -- no Sort operator
```

### 3.4 [Hard] You have indexes (A), (A,B) and (A,B,C). Which are redundant and how do you find them safely?
**Answer**
(A) is redundant with (A,B) (and (A,B,C)) because any seek on A can use the longer index; (A,B) is redundant with (A,B,C) if key and include needs align. Check usage stats, uniqueness, filters and INCLUDEs before dropping.

**Explanation**
The longer index is bigger, so a narrow (A) can be faster for scans of just A. Duplicates are a clear drop; "left-prefix subsets" are candidates. Exceptions: a unique index/constraint on (A) enforces a rule; FK-supporting indexes; filtered indexes; different INCLUDEs; different sort direction. Use usage DMVs over a full business cycle (stats reset on restart) and keep a script to recreate.

**Code**
```sql
-- Usage vs write cost (reset on restart / index rebuild in some versions)
SELECT OBJECT_NAME(i.object_id) AS tbl, i.name,
       us.user_seeks, us.user_scans, us.user_lookups, us.user_updates
FROM sys.indexes i
LEFT JOIN sys.dm_db_index_usage_stats us
  ON us.object_id=i.object_id AND us.index_id=i.index_id AND us.database_id=DB_ID()
WHERE OBJECTPROPERTY(i.object_id,'IsUserTable')=1 AND i.type_desc='NONCLUSTERED'
ORDER BY (ISNULL(us.user_seeks,0)+ISNULL(us.user_scans,0)+ISNULL(us.user_lookups,0)), us.user_updates DESC;
```

**Gotchas / follow-ups**
- Dropping an index that a hint or `FORCESEEK` references causes errors.
- Quarterly/year-end reports may not show usage in a short window.

### 3.5 [Tricky] Does column order matter for an index used only with equality predicates on all columns? And does the order of predicates in the WHERE clause matter?
**Answer**
For all-equality predicates on every key column, a seek works regardless of order, so order mainly matters for other queries (prefix reuse, sort, selectivity/density in statistics). The textual order of WHERE predicates never matters; the optimizer reorders them.

**Explanation**
Density statistics exist for each left-prefix of the key, so order affects estimate quality: stats on (A,B) give density for A and for (A,B), but not for B alone (B alone has no histogram unless a separate stat/index exists). Put the column that other queries use alone first. Also equality-on-all-columns benefits uniqueness: a unique index may let the optimizer know at most one row exists.

**Code**
```sql
DBCC SHOW_STATISTICS ('dbo.Tickets', 'IX_Tickets_Status_Created') WITH DENSITY_VECTOR;
-- Shows "All density" for (Status) and (Status, CreatedAt); the histogram is only on the first column.
```

**Gotchas / follow-ups**
- Histogram only on the leading column; multi-column correlations are covered only by density (and, in newer CE, by exponential backoff assumptions).

**Interview traps**
- Putting the most selective column first by reflex without regard for query patterns/range columns.
- Thinking INCLUDE columns help seek or sort.
- Thinking `WHERE B=5` can seek on IX(A,B).
- Over-covering: wide indexes with many INCLUDEs duplicate the table.

**Cheat sheet**
- Keys = seek/sort; INCLUDE = leaf-only payload.
- Equality columns first, then range, then ORDER BY.
- No skip scan in SQL Server rowstore.
- Histogram only on leading key column.
- Covering removes key lookups; costs write/storage.

---

## Topic 4: Seek vs Scan, Key/RID Lookups

**Study notes**

- **Index Seek**: navigate the B-tree to a start point using a seek predicate and read a range (can return 1 row or millions). **Index Scan**: read the entire index leaf (or a large part) - not necessarily bad.
- **Table Scan** = heap; **Clustered Index Scan** = whole table.
- **Key Lookup** (clustered table) / **RID Lookup** (heap): for each row found in an NCI, jump into the base table to fetch missing columns. Each lookup is a random read, often 3-4 logical reads.
- The optimizer chooses between "NCI seek + lookups" and "scan" by a **tipping point**: roughly when lookups reach about 25-33% of the table's *pages* (not rows), a scan wins. Narrow NCI + wide rows tips later.

```
 Nested Loops (Inner Join)
   |-- Index Seek (NCI)  IX_Orders_Customer   [outer: N rows]
   `-- Key Lookup (Clustered) PK_Orders       [executed N times]
```
- Fix lookups by covering (INCLUDE), by reducing selected columns, or accepting if N is tiny.

### 4.1 [Easy] Is an Index Scan always bad, and an Index Seek always good?
**Answer**
No. A seek reading millions of rows can be worse than a scan, and a scan is optimal when you need most of the table. What matters is rows read vs rows needed.

**Explanation**
Compare *Estimated/Actual Number of Rows Read* against *Actual Rows* and look at the number of executions. A seek with a residual predicate that discards 99% of rows read is inefficient. A scan of a small table or of a narrow covering index is fine.

**Code**
```sql
SET STATISTICS IO ON;
SELECT COUNT(*) FROM dbo.Orders;             -- scan of narrowest index; check logical reads
SELECT * FROM dbo.Orders WHERE OrderId = 5;  -- seek; ~3 logical reads
```

### 4.2 [Follow-up] What is a Key Lookup and why can it be expensive?
**Answer**
After seeking in an NCI, the engine fetches non-covered columns from the clustered index, one probe per row. With N matching rows you get roughly N x (tree depth) logical reads of random I/O.

**Explanation**
Plan shape: Nested Loops with NCI seek on top and Key Lookup below, executed once per outer row. In the plan, check "Number of Executions" on the lookup operator. At high N, the optimizer switches to a clustered index scan. Fix: add INCLUDE columns. Notice also lookups produce blocking/deadlock potential (reader on NCI, writer on clustered), a classic deadlock pattern fixed by covering.

**Code**
```sql
-- Causes Key Lookup if IX_Orders_Customer only has CustomerId
SELECT OrderId, OrderDate, Total FROM dbo.Orders WHERE CustomerId = 42;

-- Fix
CREATE INDEX IX_Orders_Customer ON dbo.Orders (CustomerId) INCLUDE (OrderDate, Total)
WITH (DROP_EXISTING = ON);
```

### 4.3 [Medium] Explain the "tipping point" and how it can make a query flip between seek+lookup and scan.
**Answer**
The optimizer estimates cost of N random lookups versus a sequential scan; when estimated rows exceed roughly a quarter to a third of the table's page count, a scan is cheaper. A parameter-sensitive query can flip plans between these when estimates differ by parameter value.

**Explanation**
It's about pages, not rows: the comparison is the number of lookups (each a random read) against the number of pages a scan would read. A table with many rows per page has few pages, so a scan is cheap and the tipping point is a small fraction of the rows; wide rows (few per page) push it higher. The commonly quoted range is roughly 25-33% of pages, but the exact value depends on the cost model, so say "a fraction of the table's pages" rather than a fixed number. This is the mechanism behind many parameter-sniffing incidents.

**Code**
```sql
-- Compare plans/IO for a selective vs non-selective value
SET STATISTICS IO ON;
SELECT * FROM dbo.Orders WHERE CustomerId = 42;        -- few rows: seek+lookup
SELECT * FROM dbo.Orders WHERE CustomerId = 1;         -- many rows: clustered scan
```

### 4.4 [Hard] How do you read "Seek Predicates" vs "Predicate" in a plan, and what do "Actual Rows Read" vs "Actual Rows" tell you?
**Answer**
Seek Predicates define the B-tree range navigated; Predicate is a residual filter applied to every row read in that range. If rows read greatly exceeds rows returned, the index doesn't serve the filter well.

**Explanation**
In the operator's properties (F4), `Actual Number of Rows for All Executions` and `Number of Rows Read` (available in SQL Server 2016 SP1+ actual plans). Example: IX(A,B) with `A>10 AND B=5` -> A is the seek range, B residual; large rows-read:rows ratio. Fix with a better key order or additional index. Also check "Ordered: True/False" and "Scan Direction".

**Code**
```sql
SET STATISTICS XML ON;  -- or Include Actual Execution Plan in SSMS
SELECT OrderId FROM dbo.Orders WHERE CustomerId > 100 AND Status = 3;
-- Plan (described): Index Seek IX(CustomerId,Status): Seek Predicate CustomerId > 100;
-- Predicate Status = 3; Rows Read = 900,000; Actual Rows = 1,200  -> poor.
SET STATISTICS XML OFF;
```

**Gotchas / follow-ups**
- Unordered vs Ordered scans affect allocation-order scans (can be skipped/duplicated under READ UNCOMMITTED).

### 4.5 [Tricky] A query returns 1 row via a seek yet shows high logical reads. Why?
**Answer**
Likely a seek with a wide residual predicate range, many lookups, a Nested Loops inner side executing repeatedly, LOB/off-row reads, or reads counted on worktables/spools. Check rows read, executions, and STATISTICS IO per table.

**Explanation**
STATISTICS IO lists reads by object: a `Worktable` entry indicates spools/sorts spills; `lob logical reads` signals max types; a table accessed many times with low per-access reads means a loop join with high outer rows. Also a **Row Goal** (TOP/EXISTS) can mislead the estimate. Forwarded records in heaps add reads too.

**Code**
```sql
SET STATISTICS IO ON;
SELECT TOP (1) o.OrderId FROM dbo.Orders o
JOIN dbo.OrderLines l ON l.OrderId = o.OrderId
WHERE l.ProductId = 7;
-- Output e.g.: Table 'OrderLines'. Scan count 1, logical reads 1,000,000 ... => lots read to find 1 row.
```

**Gotchas / follow-ups**
- Scan count = number of times the access was started (per seek/scan iteration), not number of full scans.

**Interview traps**
- "Seek good, scan bad."
- Not looking at execution count on the inner side of a Nested Loops.
- Quoting a fixed 30% rows tipping point.
- Ignoring that Key Lookup adds deadlock risk.

**Cheat sheet**
- Seek = navigate+range; scan = read all.
- Lookup = per-row jump to base table; fix with INCLUDE.
- Tipping point is in pages, a fraction of the table.
- Compare Rows Read vs Actual Rows.
- Seek Predicate (range) vs Predicate (residual).

---

## Topic 5: SARGability (Functions, Implicit Conversions, LIKE, OR) & EF nvarchar/varchar Mismatch

**Study notes**

- **SARGable** (Search ARGument able): a predicate shape that lets the optimizer seek an index: `column <op> expression` where the column is bare and the expression doesn't depend on the row.
- **Non-SARGable**: wrapping the column in a function/expression: `YEAR(OrderDate)=2025`, `ISNULL(col,0)=0`, `col + 1 = 10`, `LEFT(Name,3)='abc'` (rewrite as `LIKE 'abc%'`), `CONVERT(varchar, col)=...`. (Note: `CAST(datetimeCol AS date) = @d` is a special case the optimizer can still seek, but don't rely on it.)
- **LIKE**: `LIKE 'abc%'` is SARGable (range seek); `LIKE '%abc'` and `'%abc%'` are not (scan). Use full-text search, trigram-like tricks, or computed column + reverse for suffix.
- **Implicit conversion**: when data types differ, SQL Server converts by **data type precedence** (lower precedence converts to higher). If the *column* gets converted (e.g. `varchar` column vs `nvarchar` parameter -> column converted to nvarchar) the index seek is lost / becomes a scan with `CONVERT_IMPLICIT(nvarchar, col)`.
  - nvarchar column vs varchar param: the *parameter* converts to nvarchar -> fine (seek).
  - varchar column vs nvarchar param: *column* converts -> scan (for SQL collations; for Windows collations a seek may still occur but with a costlier plan/dynamic seek).
- **OR** across different columns often prevents a single seek (may use Concatenation/Merge of two seeks, or scan). Rewrite with `UNION ALL` (with exclusion) or separate indexes. `OR` on same column -> `IN` -> seek on multiple ranges.
- Rewrites: `YEAR(d)=2025` -> `d >= '20250101' AND d < '20260101'`.

### 5.1 [Easy] What is a SARGable predicate? Give examples.
**Answer**
One where the indexed column stands alone on one side of the comparison so the engine can seek it. `OrderDate >= '2025-01-01'` is SARGable; `YEAR(OrderDate) = 2025` is not.

**Explanation**
When a function is applied to a column, SQL Server must evaluate it for each row, so it scans. Rewrite to move the computation to the constant side, using half-open ranges for dates.

**Code**
```sql
-- Non-SARGable
SELECT * FROM dbo.Orders WHERE YEAR(OrderDate) = 2025;
-- SARGable
SELECT * FROM dbo.Orders WHERE OrderDate >= '20250101' AND OrderDate < '20260101';
-- Non-SARGable
SELECT * FROM dbo.Orders WHERE ISNULL(Discount,0) = 0;
-- Better
SELECT * FROM dbo.Orders WHERE Discount = 0 OR Discount IS NULL;
```

### 5.2 [Follow-up] Which LIKE patterns can use an index seek?
**Answer**
Prefix patterns like `'abc%'` seek; patterns with a leading wildcard (`'%abc'`, `'%abc%'`) require a scan. For contains-search at scale use Full-Text Search (`CONTAINS`) or restructure data.

**Explanation**
A prefix pattern is converted internally to a range (`>= 'abc' AND < 'abd'`). For suffix search, a persisted computed column with `REVERSE(col)` and `LIKE REVERSE('abc')+'%'` can work. Beware: with a parameter `LIKE @p + '%'` the prefix is unknown at compile time, so the plan includes a dynamic seek and estimates may be poor; sniffing a value like `''` or `'%'` causes bad plans.

**Code**
```sql
SELECT * FROM dbo.Customers WHERE LastName LIKE 'Sm%';   -- seek
SELECT * FROM dbo.Customers WHERE LastName LIKE '%ith';  -- scan

-- Full-text alternative (requires full-text index)
SELECT * FROM dbo.Articles WHERE CONTAINS(Body, '"performance*"');
```

### 5.3 [Medium] Explain how implicit conversion can silently kill performance. Which direction matters?
**Answer**
SQL Server converts the lower-precedence type to the higher-precedence one. If that's the indexed column (e.g. `varchar` column compared to an `nvarchar` parameter), each row's value is converted, so the index can't be sought directly and you get a scan. If the parameter is converted instead, the seek survives.

**Explanation**
Detect with `CONVERT_IMPLICIT` in the plan XML and a warning on the SELECT operator ("Type conversion in expression ... may affect CardinalityEstimate/SeekPlan"). Common sources: ORMs passing `nvarchar` for string params (.NET `string` -> `nvarchar(4000)` by default in some paths), int vs varchar compare (`varchar` column vs int literal converts the column to int - also fails on non-numeric data), `datetime` vs `datetime2` mismatches. Fix by aligning types: change parameter type, change column type, or cast the parameter side explicitly.

**Code**
```sql
-- Column Code is varchar(20), indexed
DECLARE @c nvarchar(20) = N'AB123';
SELECT * FROM dbo.Parts WHERE Code = @c;               -- column converted -> scan/poor seek
DECLARE @c2 varchar(20) = 'AB123';
SELECT * FROM dbo.Parts WHERE Code = @c2;               -- clean seek

-- Find implicit conversions in cached plans
SELECT TOP 20 st.text, qp.query_plan
FROM sys.dm_exec_cached_plans cp
CROSS APPLY sys.dm_exec_query_plan(cp.plan_handle) qp
CROSS APPLY sys.dm_exec_sql_text(cp.plan_handle) st
WHERE CAST(qp.query_plan AS nvarchar(max)) LIKE N'%CONVERT_IMPLICIT%';
```

### 5.4 [Hard] How does this appear in EF/EF Core, and how do you fix it?
**Answer**
By default .NET `string` maps to `nvarchar`; if the DB column is `varchar`, EF sends `nvarchar` parameters, causing `CONVERT_IMPLICIT` on the column. Fix by mapping the property as non-Unicode (`IsUnicode(false)` / `HasColumnType("varchar(50)")` / `[Column(TypeName="varchar(50)")]`) so parameters are sent as `varchar`.

**Explanation**
EF Core sends `@p0 nvarchar(4000)` for unmapped strings, or sized from the model's max length. With a mismatched model, a seek on `varchar` index becomes a scan under SQL collations (and costlier under Windows collations). Also: EF's `Contains` on a list generates `IN`/`OPENJSON` (EF Core 8+ uses OPENJSON for parameterized collections by default on SQL Server 2016+ compat 130+), which may affect cardinality estimates. `EF.Functions.Like(x.Name, "a%")` keeps SARGable form; `x.Name.Contains("a")` produces `LIKE N'%a%'` (non-SARGable).

**Code**
```sql
-- Typical EF Core generated T-SQL (described): 
--   exec sp_executesql N'SELECT ... FROM Parts WHERE Code = @p0', N'@p0 nvarchar(4000)', @p0 = N'AB123'
-- Plan: Index Scan with predicate CONVERT_IMPLICIT(nvarchar(4000), Code) = @p0
-- After mapping .IsUnicode(false): parameter is varchar(20) -> Index Seek.
```
```csharp
modelBuilder.Entity<Part>().Property(p => p.Code).HasMaxLength(20).IsUnicode(false);
```

**Gotchas / follow-ups**
- Also `.HasColumnType("datetime2(0)")` vs `datetime2(7)` is harmless; datetime vs datetime2 mixes can cause conversions.
- Parameter length variability (sized by value) bloats plan cache in raw ADO.NET/Dapper; set explicit sizes (EF handles this).

### 5.5 [Tricky] `WHERE (@Name IS NULL OR Name = @Name)` - "catch-all" search. What's the problem and what are the fixes?
**Answer**
One cached plan must serve every combination of NULL/non-NULL parameters, so it can't be optimal for all; typically it scans or sniffs the first values. Fixes: `OPTION (RECOMPILE)` (the optimizer embeds parameter values so dead branches are eliminated), dynamic SQL via `sp_executesql` with only needed predicates, or separate queries/procs.

**Explanation**
With `OPTION (RECOMPILE)` the parameter embedding optimization simplifies `@Name IS NULL` at compile time, giving a seek when `@Name` is supplied; cost is compile CPU per execution, fine for moderately frequent reports, not for 1000s/sec OLTP. Dynamic SQL gives plan reuse per shape; guard against injection by using parameters, not concatenated values. In EF, build `IQueryable` conditionally (`if (name != null) q = q.Where(...)`) - each shape is a separate query.

**Code**
```sql
SELECT OrderId FROM dbo.Orders
WHERE (@CustomerId IS NULL OR CustomerId = @CustomerId)
  AND (@Status     IS NULL OR Status     = @Status)
OPTION (RECOMPILE);

-- Dynamic alternative
DECLARE @sql nvarchar(max) = N'SELECT OrderId FROM dbo.Orders WHERE 1=1';
IF @CustomerId IS NOT NULL SET @sql += N' AND CustomerId = @CustomerId';
IF @Status     IS NOT NULL SET @sql += N' AND Status = @Status';
EXEC sp_executesql @sql, N'@CustomerId int, @Status tinyint', @CustomerId, @Status;
```

**Gotchas / follow-ups**
- `OPTION (RECOMPILE)` at the statement level enables the embedding optimization; procedure-level `WITH RECOMPILE` does not embed the same way.
- `COALESCE(col, '')=...` / `ISNULL(col,...)` forms are non-SARGable.

**Interview traps**
- Saying a function on the *parameter* side hurts (it's fine); it's the column side.
- Claiming implicit conversion always causes a scan (direction/collation matters).
- Using `BETWEEN` with end-of-day on `datetime` (`'20251231 23:59:59.999'` rounds to next day); use half-open ranges.
- Forgetting EF `string.Contains` -> `LIKE '%x%'`.

**Cheat sheet**
- Keep the column bare; move math to the constant side.
- `LIKE 'x%'` seeks; `LIKE '%x'` scans.
- Precedence: nvarchar > varchar; the lower converts to the higher.
- EF: `IsUnicode(false)` for varchar columns.
- Catch-all queries: `OPTION (RECOMPILE)` or dynamic SQL.

---

## Topic 6: Filtered, Unique & Columnstore Indexes

**Study notes**

- **Filtered index**: an NCI with a `WHERE` clause; indexes only a subset of rows. Smaller, cheaper to maintain, with statistics on the subset. Great for sparse/skewed data (`WHERE IsActive = 1`, `WHERE DeletedAt IS NULL`, `WHERE Col IS NOT NULL`).
- Limits: the optimizer uses it only if it can prove the query's predicate implies the filter; **parameterized predicates often can't** (`WHERE IsActive = @a` won't match a filter `IsActive=1` unless `OPTION (RECOMPILE)` or literal). Filter supports simple comparisons only (no `OR`, no functions, no LIKE). Needs `QUOTED_IDENTIFIER`/`ANSI_NULLS` etc. ON on writes.
- **Unique index**: enforces uniqueness (a UNIQUE constraint is implemented via a unique index). NULLs: only **one** NULL allowed in a unique key column; use a filtered unique index (`WHERE col IS NOT NULL`) for "unique when present". Unique also gives the optimizer information (at most 1 row).
- **Columnstore**: data stored by column in compressed **segments** (~1,048,576 rows per **rowgroup**), plus a **delta store** (B-tree rowgroups for recent inserts) merged by the tuple mover. Excellent for analytics/aggregations (high compression, **batch mode** execution, segment elimination); poor for singleton lookups/point updates (clustered columnstore can be paired with NCIs since SQL Server 2016).
```
Clustered columnstore:  [RowGroup1: col A seg | col B seg | col C seg] ... [Delta store (rowstore)]
Query touching only col B reads only B's segments.
```
- Clustered columnstore (CCI) = the table; nonclustered columnstore (NCCI) = added to rowstore table for real-time analytics (updatable since 2016).

### 6.1 [Easy] What is a filtered index and when would you use it?
**Answer**
A nonclustered index defined with a WHERE clause so it contains only matching rows. Use it for queries targeting a small, well-defined subset (active records, unprocessed queue items, non-NULL values).

**Explanation**
Benefits: smaller storage, less write overhead (only qualifying rows touch it), accurate stats for the subset. For example, a 100M-row `Orders` table where only 50k are `Status='Pending'`: a filtered index on pending orders is tiny and gives fast queue polling.

**Code**
```sql
CREATE NONCLUSTERED INDEX IX_Orders_Pending
ON dbo.Orders (CreatedAt) INCLUDE (CustomerId)
WHERE Status = 0;

SELECT OrderId, CustomerId FROM dbo.Orders WHERE Status = 0 ORDER BY CreatedAt;  -- uses it
```

### 6.2 [Follow-up] How do unique indexes treat NULLs, and how do you make a column unique only when it's not NULL?
**Answer**
A unique index treats NULLs as equal, so only one NULL row is allowed. For "unique where provided", create a filtered unique index with `WHERE col IS NOT NULL`.

**Explanation**
This is a classic requirement (e.g. optional external ID or email). A UNIQUE *constraint* can't be filtered, so use `CREATE UNIQUE INDEX ... WHERE`. EF Core: `HasIndex(x => x.Email).IsUnique()` generates a filtered unique index `WHERE [Email] IS NOT NULL` automatically for nullable columns on SQL Server (single-column); you can override with `.HasFilter(null)`.

**Code**
```sql
CREATE UNIQUE NONCLUSTERED INDEX UX_Users_Email
ON dbo.Users (Email) WHERE Email IS NOT NULL;
```

### 6.3 [Medium] Why might the optimizer ignore my filtered index?
**Answer**
Because it can't guarantee the query's predicate is within the filter - most often because the query uses a parameter or variable (`WHERE Status = @s`) rather than the literal the index was defined with, or because the index doesn't cover the columns needed and a lookup is costed higher than a scan.

**Explanation**
A plan reused for different parameters cannot safely use a filtered index, so the optimizer rejects it at compile time for parameterized/auto-parameterized queries. Workarounds: use literals (or dynamic SQL), `OPTION (RECOMPILE)` (embedding gives a literal), or add the filter column to the INCLUDE/key. Remember also `SET` options requirement (ARITHABORT, QUOTED_IDENTIFIER ON etc.). Forced index hints that can't match fail with error 8622.

**Code**
```sql
DECLARE @s tinyint = 0;
SELECT OrderId FROM dbo.Orders WHERE Status = @s;                    -- typically cannot use IX_Orders_Pending
SELECT OrderId FROM dbo.Orders WHERE Status = @s OPTION (RECOMPILE); -- can use it
```

### 6.4 [Hard] How do columnstore indexes work and when would you choose one over a B-tree?
**Answer**
Columns are stored separately, compressed in rowgroups of up to ~1M rows, and scanned using batch-mode execution (processing ~900 rows at a time). Choose them for large-scale scans, aggregations and data-warehouse/reporting queries; stick with B-trees for selective OLTP seeks and frequent single-row updates.

**Explanation**
Key mechanics: **segment elimination** (min/max metadata per segment skips irrelevant segments, so data order matters when loading); **delta store** absorbs small inserts, the tuple mover compresses closed rowgroups (>=102,400 rows are eligible for direct compressed load in bulk inserts, otherwise delta); deletes mark rows in a **delete bitmap** (so fragmentation = deleted-row ratio; `REORGANIZE` cleans it, `REBUILD` re-creates). Compression of 5-10x typical. Batch mode on rowstore (2019+, compat 150) extends some batch benefits to rowstore tables. NCCI + filtered can serve hybrid (HTAP) workloads.

**Code**
```sql
CREATE CLUSTERED COLUMNSTORE INDEX CCI_FactSales ON dbo.FactSales;

-- Rowgroup health
SELECT OBJECT_NAME(object_id) AS tbl, state_desc, COUNT(*) AS rowgroups,
       SUM(total_rows) AS total_rows, SUM(deleted_rows) AS deleted_rows
FROM sys.dm_db_column_store_row_group_physical_stats
WHERE object_id = OBJECT_ID('dbo.FactSales')
GROUP BY object_id, state_desc;

ALTER INDEX CCI_FactSales ON dbo.FactSales REORGANIZE WITH (COMPRESS_ALL_ROW_GROUPS = ON);
```

**Gotchas / follow-ups**
- Trickle single-row inserts create many small delta rowgroups; batch your loads (>= 102,400 rows).
- Point lookups on CCI scan segments unless an NCI is added.
- Can't have a columnstore on some types (e.g. varchar(max) was unsupported in older versions; check current limits for your version).

### 6.5 [Tricky] You add a unique index to a large live table and it fails/blocks. What do you check?
**Answer**
Check for existing duplicates (including multiple NULLs) before creating, and use `ONLINE = ON` (edition permitting) with a suitable `MAXDOP`/`SORT_IN_TEMPDB` to minimize blocking; schema modification locks are still taken briefly at start and end.

**Explanation**
Duplicate violations abort after long work (rolled back, costly). Pre-validate with a GROUP BY HAVING. `IGNORE_DUP_KEY = ON` silently drops duplicate inserts later - dangerous semantics: it turns an error into a warning and discards the row. `RESUMABLE = ON` (2017+ for rebuild, 2019+ for create) lets you pause/resume long operations.

**Code**
```sql
SELECT Email, COUNT(*) FROM dbo.Users WHERE Email IS NOT NULL GROUP BY Email HAVING COUNT(*) > 1;

CREATE UNIQUE INDEX UX_Users_Email ON dbo.Users (Email) WHERE Email IS NOT NULL
WITH (ONLINE = ON, SORT_IN_TEMPDB = ON, MAXDOP = 4);
```

**Interview traps**
- "Filtered indexes work with any parameterized query" - no.
- Saying a UNIQUE constraint allows many NULLs (that's PostgreSQL/Oracle semantics for some cases, not SQL Server).
- Using columnstore for OLTP point lookups.
- `IGNORE_DUP_KEY` as a "feature" without caveats.

**Cheat sheet**
- Filtered index = small + cheap; beware parameters.
- One NULL only in unique index; use filtered unique for optional uniqueness.
- Columnstore: rowgroup ~1M rows, batch mode, segment elimination, delta store.
- CCI = analytics; B-tree = selective OLTP.
- Create large indexes ONLINE / RESUMABLE where available.

---

## Topic 7: Fragmentation, Rebuild vs Reorganize, Large-Table Maintenance

**Study notes**

- **Internal fragmentation**: low page density (free space within pages). **External (logical) fragmentation**: leaf pages out of physical/logical order (`avg_fragmentation_in_percent`).
- Causes: page splits (random inserts, updates growing rows), deletes. Impact: more pages to read, more read-ahead inefficiency, larger memory use. On modern SSD/SAN storage the *seek-order* penalty is small; **page density** (wasted space) is often the bigger real cost. Fragmentation matters most for large scans/range reads.
- **REORGANIZE**: always online, incremental, single-threaded, compacts/reorders leaf level in place, uses small transactions (can be stopped without losing work), does *not* update statistics.
- **REBUILD**: drops and recreates the index; updates statistics with a full scan (for that index); can be `ONLINE = ON` (Enterprise/Azure; some restrictions), parallel, applies fill factor, needs extra space (~ size of index), and generates large log.
- Guidance (old Microsoft docs): <5% (or <10%) ignore, 5-30% reorganize, >30% rebuild; ignore tiny indexes (<1000 pages). Treat as a starting heuristic, not law.
- Large-table maintenance: use `WAIT_AT_LOW_PRIORITY`, `RESUMABLE`, per-partition rebuilds, batch deletes, maintenance windows, Ola Hallengren's scripts or similar, and keep stats updated.

### 7.1 [Easy] What is index fragmentation and how do you measure it?
**Answer**
Fragmentation is when index pages are out of logical order (external) or have excessive free space (internal) due to page splits and deletes. Measure with `sys.dm_db_index_physical_stats` (`avg_fragmentation_in_percent`, `avg_page_space_used_in_percent`, `page_count`).

**Explanation**
`LIMITED` mode is cheapest (only upper-level pages, no page density); `SAMPLED`/`DETAILED` read leaf pages and are expensive on big tables - run off-hours. Ignore small indexes since fragmentation numbers there are meaningless.

**Code**
```sql
SELECT OBJECT_NAME(ps.object_id) AS tbl, i.name AS idx, ps.avg_fragmentation_in_percent,
       ps.avg_page_space_used_in_percent, ps.page_count
FROM sys.dm_db_index_physical_stats(DB_ID(), NULL, NULL, NULL, 'SAMPLED') ps
JOIN sys.indexes i ON i.object_id = ps.object_id AND i.index_id = ps.index_id
WHERE ps.page_count > 1000 AND ps.index_id > 0
ORDER BY ps.avg_fragmentation_in_percent DESC;
```

### 7.2 [Follow-up] Rebuild vs Reorganize: what are the differences?
**Answer**
Reorganize is an online, lightweight, interruptible in-place leaf-level defrag that doesn't update stats. Rebuild recreates the whole index, can change options (fill factor, compression), updates stats with a full scan, but is heavier in locks (offline unless ONLINE), log and space.

**Explanation**
Reorganize compacts pages to the fill factor and reorders leaf pages but doesn't rearrange intermediate levels. After REORGANIZE, run `UPDATE STATISTICS` manually if needed. REBUILD under full/bulk-logged recovery logs heavily; bulk-logged can minimally log rebuilds but affects point-in-time recovery of that log. With Availability Groups, big rebuilds lag secondaries.

**Code**
```sql
ALTER INDEX IX_Orders_Customer ON dbo.Orders REORGANIZE;
ALTER INDEX IX_Orders_Customer ON dbo.Orders REBUILD WITH (ONLINE = ON, SORT_IN_TEMPDB = ON, MAXDOP = 4);
ALTER INDEX ALL ON dbo.Orders REBUILD;
UPDATE STATISTICS dbo.Orders WITH FULLSCAN;   -- after reorganize if desired
```

### 7.3 [Medium] Is rebuilding indexes nightly a good idea?
**Answer**
Usually not. Blanket nightly rebuilds waste I/O and log, and on SSD-based storage the gain from lower fragmentation is often small; much of the visible benefit is the statistics refresh. Rebuild selectively based on thresholds and workload, and prioritize statistics updates.

**Explanation**
Many "rebuild fixed it" cases are actually the **statistics update** effect (a rebuild updates stats with fullscan, forcing new plans). Fragmentation of a random-key index returns quickly, so repeated rebuilds just churn; consider fill factor or a better key. Where fragmentation matters: large range scans from spinning disks or read-ahead-heavy warehouse scans.

**Code**
```sql
-- Cheaper alternative first: keep stats fresh
UPDATE STATISTICS dbo.Orders WITH FULLSCAN;   -- or SAMPLE / PERSIST_SAMPLE_PERCENT = ON (2016 SP1 CU4+/2017+)
```

### 7.4 [Hard] How do you rebuild a huge index with minimal impact? What are the options?
**Answer**
Use `ONLINE = ON` (Enterprise/Azure SQL), `RESUMABLE = ON` with `MAX_DURATION` (2017+), `WAIT_AT_LOW_PRIORITY` to avoid blocking queues, `SORT_IN_TEMPDB`, controlled `MAXDOP`, and rebuild per partition when partitioned. Watch log space and AG redo lag.

**Explanation**
Online rebuild takes a short Sch-M/S lock at start and end; without low-priority waits, a long-running transaction blocks the final lock and everyone queues behind it. Resumable rebuilds persist progress so you can pause or survive failures but hold extra space. For partitioned tables, `REBUILD PARTITION = n` limits scope. Accelerated Database Recovery (2019+) helps rollback times. Consider log backups frequency during rebuild.

**Code**
```sql
ALTER INDEX PK_Orders ON dbo.Orders
REBUILD WITH (ONLINE = ON (WAIT_AT_LOW_PRIORITY (MAX_DURATION = 5 MINUTES, ABORT_AFTER_WAIT = SELF)),
              RESUMABLE = ON, MAX_DURATION = 120, MAXDOP = 4);

ALTER INDEX PK_Orders ON dbo.Orders PAUSE;
ALTER INDEX PK_Orders ON dbo.Orders RESUME;

SELECT name, percent_complete, state_desc FROM sys.index_resumable_operations;
```

**Gotchas / follow-ups**
- `ABORT_AFTER_WAIT = SELF` abandons your rebuild; `BLOCKERS` kills blockers - dangerous.
- Resumable operations have restrictions (e.g. not for some index types/options).

### 7.5 [Tricky] Deleting 200 million old rows from a 1-billion-row table is filling the log and blocking users. How do you do it?
**Answer**
Delete in batches (e.g. 5,000-10,000 rows) in short transactions with a supporting index, using the cleanup column order, with log backups running; or better, partition by date and **switch out** the old partition (metadata-only). Avoid a single giant DELETE.

**Explanation**
Batches under ~5,000 rows avoid lock escalation to table lock (escalation threshold is ~5,000 locks). A batch loop with `DELETE TOP (n)` needs an index on the filter column to avoid scanning each iteration. Consider `SELECT INTO`/keeping the rows you want and swapping for extreme cases. Under simple recovery the log reuse is by checkpoint; under full recovery needs log backups.

**Code**
```sql
DECLARE @rows int = 1;
WHILE @rows > 0
BEGIN
  DELETE TOP (4000) FROM dbo.AuditLog WHERE CreatedAt < DATEADD(YEAR,-2,SYSUTCDATETIME());
  SET @rows = @@ROWCOUNT;
  WAITFOR DELAY '00:00:00.200';   -- let log backups/other work breathe
END
```

**Gotchas / follow-ups**
- Triggers, FK cascades and many NCIs multiply per-row cost.
- If a table has ghost records, cleanup is async; space is reclaimed later.

**Interview traps**
- "Fragmentation >30% = always rebuild."
- Not recognizing the stats-refresh side effect of rebuild.
- Thinking REORGANIZE updates statistics.
- Ignoring page_count threshold.
- Assuming ONLINE rebuild is available in Standard edition (check version: not on older Standard).

**Cheat sheet**
- Measure via `sys.dm_db_index_physical_stats`; ignore < ~1000 pages.
- <5-10% nothing, 5-30% reorg, >30% rebuild (guideline).
- Reorg: online, interruptible, no stats; Rebuild: stats fullscan, fill factor, may be ONLINE/RESUMABLE.
- Batch big deletes below ~5,000 rows; or partition-switch.
- Keep stats fresh - often the real fix.

---

## Topic 8: Statistics, Histograms & Cardinality Estimation

**Study notes**

- **Statistics** describe the distribution of values in column(s): a **header** (rows, rows sampled, last update), **density vector** (average duplicates for left-prefixes), and a **histogram** (up to 200 steps on the *leading* column: `RANGE_HI_KEY`, `RANGE_ROWS`, `EQ_ROWS`, `DISTINCT_RANGE_ROWS`, `AVG_RANGE_ROWS`).
- Created automatically with each index and (AUTO_CREATE_STATISTICS) on columns used in predicates (`_WA_Sys_...`).
- **Auto-update**: triggers when modifications cross a threshold: legacy = 500 + 20% of rows; since SQL Server 2016 (compat 130+) / trace flag 2371 behavior = dynamic `MIN(500 + 0.20*n, SQRT(1000*n)`) so large tables update sooner. `AUTO_UPDATE_STATISTICS_ASYNC` avoids blocking the query waiting for the update.
- **Cardinality Estimator (CE)**: legacy CE (70) vs new CE (120+, compat 120+). Differences: multi-predicate correlation assumption (legacy = independence; new = exponential backoff), ascending key handling, join estimation. Compat 130/140/150/160 bring additional behaviors (e.g. IQP features).
- Estimates drive join type, join order, memory grants, parallelism, index choice. Big gap between *Estimated* and *Actual* rows is the #1 plan-problem signal.
- Common bad-estimate sources: stale stats, ascending-key problem (new rows beyond histogram), table variables (before 2019 deferred compilation: estimate of 1 row), multi-statement TVFs (fixed guess; 100 since 2014, interleaved execution in 2017), local variables (density-based guess), complex expressions, correlated columns.

### 8.1 [Easy] What are statistics and why does the optimizer need them?
**Answer**
Statistics are metadata about value distribution and row counts for columns/indexes. The optimizer uses them to estimate how many rows each operator will produce, which determines plan choices (seek vs scan, join type and order, memory grants).

**Explanation**
Without good estimates, the optimizer picks plans for the wrong data volume; e.g. thinks 10 rows will return, picks Nested Loops + lookups, but 5 million actually return.

**Code**
```sql
DBCC SHOW_STATISTICS ('dbo.Orders', 'IX_Orders_Customer');   -- header, density vector, histogram

SELECT s.name, sp.last_updated, sp.rows, sp.rows_sampled, sp.modification_counter
FROM sys.stats s
CROSS APPLY sys.dm_db_stats_properties(s.object_id, s.stats_id) sp
WHERE s.object_id = OBJECT_ID('dbo.Orders');
```

### 8.2 [Follow-up] When do statistics auto-update, and what can go wrong?
**Answer**
When the modification counter crosses a threshold (legacy 500 + 20% of rows; dynamic lower thresholds for big tables in 2016+/compat 130). Problems: large tables update too rarely, sampling may miss skew, and an update invalidates plans leading to recompiles at bad moments.

**Explanation**
Auto-update happens at *compile/optimization time* of a query that uses the stat (synchronously by default, so that query waits). On a 100M-row table the legacy 20% would be 20M changes. Ascending-key columns (dates, identities) are a classic problem: newest rows aren't in the histogram -> estimate of ~1 row (older CE) or density-based guess (new CE). Mitigate with scheduled `UPDATE STATISTICS`, async updates, or incremental stats on partitions.

**Code**
```sql
ALTER DATABASE CURRENT SET AUTO_UPDATE_STATISTICS_ASYNC ON;
UPDATE STATISTICS dbo.Orders IX_Orders_Customer WITH FULLSCAN;
-- Persist the sample rate so later auto-updates don't revert to default sampling
UPDATE STATISTICS dbo.Orders WITH SAMPLE 25 PERCENT, PERSIST_SAMPLE_PERCENT = ON;
```

### 8.3 [Medium] How does the histogram produce an estimate for `WHERE CustomerId = 42` versus `WHERE CustomerId = @c` (sniffed) versus a local variable?
**Answer**
For a literal or sniffed parameter, the optimizer finds the value in the histogram (`EQ_ROWS` if it's a step boundary, else `AVG_RANGE_ROWS`). For a local variable the value is unknown at compile time, so it uses the density vector (rows x density = average rows per value), a generic estimate that ignores skew.

**Explanation**
This is why the "local variable trick" yields a "average" plan, stable but mediocre for outliers. For inequality with a variable, a fixed guess is used (e.g. 30% for single inequality, 9-10% for BETWEEN, legacy CE; new CE differs slightly). Using `OPTION (RECOMPILE)` makes variables "known" (embedding).

**Code**
```sql
DECLARE @c int = 42;
SELECT * FROM dbo.Orders WHERE CustomerId = @c;                    -- density-based estimate
SELECT * FROM dbo.Orders WHERE CustomerId = @c OPTION (RECOMPILE);  -- histogram-based
SELECT * FROM dbo.Orders WHERE CustomerId = 42;                    -- histogram-based
```

### 8.4 [Hard] Estimated rows is 1 but actual is 2,000,000. List likely causes and fixes.
**Answer**
Stale/missing statistics, ascending-key rows beyond the histogram, table variable (pre-2019 deferred compilation), multi-statement TVF, predicates on expressions/non-SARGable columns, correlated predicates, or parameter sniffing with skew. Fix by updating stats, rewriting the predicate, using temp tables, or adding multi-column/filtered statistics.

**Explanation**
Approach: compare estimated vs actual per operator (find the *first* operator where they diverge - errors propagate upward). Check `sys.dm_db_stats_properties` for `modification_counter`. For correlated columns, create multi-column statistics (`CREATE STATISTICS ... (ColA, ColB)`) - only helps density, histogram is still leading column only. Temp tables have statistics and trigger recompiles; table variables don't have column statistics (they do have cardinality of the table from 2019 deferred compilation, compat 150). For columns with many distinct values and skew, consider more histogram steps? (Max is 200 - can't change; use filtered stats to improve detail on ranges.)

**Code**
```sql
CREATE STATISTICS ST_Orders_Cust_Status ON dbo.Orders (CustomerId, Status) WITH FULLSCAN;
CREATE STATISTICS ST_Orders_Recent ON dbo.Orders (OrderDate) WHERE OrderDate >= '20250101' WITH FULLSCAN;  -- filtered statistics
-- Compat-level change impacts CE:
ALTER DATABASE CURRENT SET COMPATIBILITY_LEVEL = 150;
-- Per-query override to compare CE models
SELECT ... OPTION (USE HINT('FORCE_LEGACY_CARDINALITY_ESTIMATION'));
```

**Gotchas / follow-ups**
- `AUTO_CREATE_STATISTICS` doesn't create multi-column stats automatically.
- Upgrading compat level can regress some queries; use Query Store to compare/force.

### 8.5 [Tricky] "I rebuilt the index and the query got fast." What actually fixed it?
**Answer**
Most likely the side effects: a full-scan statistics refresh and the plan-cache invalidation that forced a recompile with fresh stats and fresh parameter values - not the defragmentation itself.

**Explanation**
Test the hypothesis: run `UPDATE STATISTICS` alone, or `DBCC FREEPROCCACHE` for that plan handle (surgical: `DBCC FREEPROCCACHE (plan_handle)`), and see if you get the same result. If clearing the plan alone fixes it, it's parameter sniffing / plan problem; if updating stats fixes it, it's estimates. Knowing this prevents cargo-cult nightly rebuild jobs.

**Code**
```sql
-- Evict just one plan
SELECT plan_handle FROM sys.dm_exec_cached_plans cp
CROSS APPLY sys.dm_exec_sql_text(cp.plan_handle) t WHERE t.text LIKE '%FROM dbo.Orders%';
DBCC FREEPROCCACHE (0x06000500...);   -- paste handle
```

**Gotchas / follow-ups**
- On Azure SQL / 2019+ use `ALTER DATABASE SCOPED CONFIGURATION CLEAR PROCEDURE_CACHE` for database-scoped clearing.

**Interview traps**
- Saying "stats update when 20% of rows change" without mentioning the dynamic threshold on 2016+/compat 130+.
- Believing the histogram covers all columns of a multi-column statistic.
- Using table variables for large row counts.
- Giving credit to defragmentation for a stats/plan fix.

**Cheat sheet**
- Stats = header + density vector + <=200-step histogram on the leading column.
- Auto-update at compile time; threshold 500+20% legacy, SQRT(1000*n) dynamic (compat 130+).
- Local variable -> density guess; sniffed/literal -> histogram.
- First divergence of estimated vs actual rows = the culprit.
- Rebuild = hidden stats update + recompile.

---

## Topic 9: Reading Execution Plans (Operators, Joins, Warnings)

**Study notes**

- **Estimated plan**: compiled, not executed (no runtime data). **Actual plan**: includes actual rows, executions, memory grant used, spills, wait stats (2016 SP1+/2017), per-operator times. **Live Query Statistics** shows progress. Plans read **right-to-left, top-to-bottom** for data flow; execution is *pull-based* from the left (root).
- Operator cost % is *estimated* even in actual plans - don't trust it blindly; compare estimates with actuals.
- **Joins**:
  - **Nested Loops**: for each outer row, probe the inner (ideal for small outer + indexed inner). Cost ~ outer rows x inner probe cost.
  - **Hash Match**: build a hash table on the smaller input, probe with the larger; good for large unsorted inputs; needs a memory grant (can spill to tempdb); non-blocking-ish for probe side but build is blocking.
  - **Merge Join**: both inputs sorted on join key; very efficient with pre-sorted (indexed) inputs; may need Sort; many-to-many needs a worktable.
- Other operators: **Sort** (memory grant, may spill; blocking), **Stream Aggregate** (needs sorted input) vs **Hash Aggregate**, **Spools** (Table/Index/Eager/Lazy - temporary storage; Eager Spool often for Halloween protection or repeated inner), **Parallelism** (Gather/Repartition/Distribute Streams), **Compute Scalar** (cheap, hides expression cost), **Filter**, **Top**, **Adaptive Join** (2017+, batch mode: picks NL or Hash at runtime).
```
Typical OLTP plan:                       Typical report plan:
  Nested Loops                             Hash Match (Aggregate)
   |-Index Seek (outer, few rows)            `-Hash Match (Inner Join)
   `-Index Seek (inner, per-row)                 |-Clustered Index Scan (big)
                                                  `-Index Scan (dimension)
```
- **Warnings**: yellow triangle - missing statistics, **implicit conversion**, **spill to tempdb** (Sort/Hash Warning), **no join predicate**, **memory grant** issues (excessive/ insufficient), missing index hints (suggestions, not truth), residual predicate, plan-affecting convert, unmatched indexes.
- Useful properties: *Estimated vs Actual Rows*, *Actual Executions*, *Rows Read*, *Memory Grant* (requested/granted/used), *Parameter list* (compiled vs runtime values), *Optimization Level (TRIVIAL/FULL)*, *Reason for early termination (Timeout)*, *CardinalityEstimationModelVersion*, *Degree of parallelism*.

### 9.1 [Easy] What's the difference between an estimated and an actual execution plan?
**Answer**
The estimated plan is what the optimizer compiles without running the query; the actual plan is the same plan annotated with runtime statistics (actual rows, executions, spills, memory). Use actual plans for diagnosing estimate errors.

**Explanation**
Both show the same operators (except for statement-level recompiles mid-batch). Estimated is safe on production for expensive queries because it doesn't execute (but doesn't reflect temp-table based recompiles). For cached plans you can retrieve the plan (with *last* actual stats in 2019+ via `sys.dm_exec_query_plan_stats` with `LAST_QUERY_PLAN_STATS` enabled).

**Code**
```sql
SET SHOWPLAN_XML ON;     -- estimated, not executed
GO
SELECT * FROM dbo.Orders WHERE CustomerId = 42;
GO
SET SHOWPLAN_XML OFF;
GO
SET STATISTICS XML ON;   -- actual plan (query executes)
SELECT * FROM dbo.Orders WHERE CustomerId = 42;
SET STATISTICS XML OFF;
```

### 9.2 [Follow-up] When does the optimizer choose Nested Loops vs Hash vs Merge join?
**Answer**
Nested Loops for small outer input with an index on the inner; Merge when both inputs are already sorted on the join keys (or cheap to sort) and are large; Hash for large, unsorted, unindexed inputs. The choice is cost-based on estimated rows.

**Explanation**
Wrong estimates are the typical failure: underestimate -> Nested Loops over millions of rows (inner executes millions of times); overestimate -> Hash join with excessive memory grant. Hash needs a memory grant and can spill; Merge requires an equality predicate (and sort order); Nested Loops can handle non-equi joins. Hints (`LOOP`, `HASH`, `MERGE`) force it but are a last resort.

**Code**
```sql
-- Inspect which join was chosen & test alternatives (for diagnosis, not production)
SELECT c.Name, SUM(o.Total) 
FROM dbo.Customers c JOIN dbo.Orders o ON o.CustomerId = c.CustomerId
GROUP BY c.Name
OPTION (HASH JOIN);
```

### 9.3 [Medium] What does a Sort/Hash "spill to tempdb" warning mean and how do you fix it?
**Answer**
The operator's memory grant was too small for the actual data (usually due to underestimated rows), so it wrote intermediate data to tempdb, slowing the query significantly. Fix the estimates (stats, rewrites), reduce rows/columns before the sort, add an index that provides order, or (2022+) rely on memory-grant feedback.

**Explanation**
Memory grants are computed at compile time from estimated rows and row size; they don't grow mid-execution. Memory grant feedback (batch mode 2017, row mode 2019, persisted 2022) adjusts subsequent executions, but is unstable with parameter-sensitive queries. The reverse issue - **excessive grants** - cause `RESOURCE_SEMAPHORE` waits and starve concurrency. Sorting wide rows (many columns/`varchar(max)`) inflates grants: sort only keys, then join back.

**Code**
```sql
-- Memory grant info for running queries
SELECT session_id, requested_memory_kb, granted_memory_kb, used_memory_kb, ideal_memory_kb
FROM sys.dm_exec_query_memory_grants;

-- Hint (use sparingly) to cap/raise a grant
SELECT ... ORDER BY ... OPTION (MAX_GRANT_PERCENT = 10);
```

### 9.4 [Hard] What are spools and "Halloween Protection"? When are spools a red flag?
**Answer**
Spools store intermediate rows in a tempdb worktable. Eager Spools often implement Halloween Protection (preventing an update from re-reading rows it already changed) and fully consume input before output; Lazy spools cache inner-side results for reuse in Nested Loops. They're red flags when large, or when caused by missing indexes/poor plan shapes.

**Explanation**
**Index Spool** ("lazy index spool") on the inner side of NL indicates the optimizer built a temporary index because none exists - the missing index is the fix. Eager spools in UPDATE/INSERT plans (e.g. `INSERT INTO t SELECT ... FROM t`) add I/O but are needed for correctness; sometimes sort/hash alternatives avoid them. A **Table Spool** repeated across a CTE referenced multiple times shows the CTE isn't materialized (it's re-evaluated or spooled depending on the plan).

**Code**
```sql
-- Typical: self-referencing insert yields an Eager Spool for Halloween protection
INSERT INTO dbo.Orders_Archive (OrderId, Total)
SELECT OrderId, Total FROM dbo.Orders_Archive WHERE Total > 1000;
-- Plan (described): Table Insert <- Eager Spool <- Clustered Index Scan.
```

**Gotchas / follow-ups**
- Don't assume a CTE is computed once; it's inlined like a view.
- Row-goal queries (TOP, EXISTS, FAST n) can change join choice - see the "Row Goal" operator property.

### 9.5 [Tricky] A plan shows TRIVIAL optimization with a Missing Index warning and no parallelism, but the query is slow. What do you check?
**Answer**
A trivial plan is chosen when there is only one obviously viable plan, so the optimizer skips cost-based exploration (and skips things like parameter-embedding transformations and some stats-creation). Check whether a better query shape, index or a statement rewrite would enable full optimization.

**Explanation**
The plan XML shows `StatementOptmLevel="TRIVIAL"`; also `StatementOptmEarlyAbortReason="TimeOut"` if a FULL optimization ran out of time budget (complex queries with many joins - the plan may be poor; simplify the query or break it into temp tables). Missing index "suggestions" are per-query greedy hints that ignore existing indexes, don't consider consolidation, and are not for production blind use.

**Code**
```sql
-- Find plans where optimization timed out
SELECT TOP 20 qp.query_plan, st.text
FROM sys.dm_exec_cached_plans cp
CROSS APPLY sys.dm_exec_query_plan(cp.plan_handle) qp
CROSS APPLY sys.dm_exec_sql_text(cp.plan_handle) st
WHERE CAST(qp.query_plan AS nvarchar(max)) LIKE N'%StatementOptmEarlyAbortReason="TimeOut"%';
```

**Interview traps**
- Trusting operator "cost %" as actual time/cost.
- Treating the missing index green text as a mandate.
- Reading plans left-to-right for data flow (data flows right-to-left).
- Ignoring "Actual Executions" on the inner side of NL.
- Assuming a spill is fixed by more server RAM (it's an estimate issue).

**Cheat sheet**
- Estimated = no run; Actual = with runtime rows/spills/grants.
- NL: small outer + indexed inner; Hash: big unsorted; Merge: pre-sorted.
- First big estimated-vs-actual gap = root cause.
- Spill = grant too small (bad estimate); big grant = RESOURCE_SEMAPHORE risk.
- Index Spool = missing index.
- Check optimization level and early-abort reason.

---

## Topic 10: Parameter Sniffing & Fixes

**Study notes**

- **Parameter sniffing**: at first compile (or recompile) the optimizer *sniffs* the actual parameter values and builds a plan optimized for them; that plan is cached and reused for later calls with different values. It is a **feature** (good plan reuse), but becomes a **problem** with skewed data distributions or tipping-point differences (one customer has 10 rows, another 5M).
- Symptoms: same proc fast in SSMS but slow from the app (different `SET ARITHABORT` option -> different cache entry); fast after `sp_recompile`/restart then slow later; intermittent timeouts; wildly different runtimes by parameter value.
- Diagnosing: compare `ParameterCompiledValue` vs `ParameterRuntimeValue` in the plan's parameter list; compare plans in Query Store for the same `query_id`; check estimate vs actual rows on the first operator.
- **Fixes (with tradeoffs)**:
  | Fix | Effect | Cost/Risk |
  |---|---|---|
  | `OPTION (RECOMPILE)` | new plan per execution, uses runtime values | CPU compile cost; no plan reuse/no Query Store forcing |
  | `OPTIMIZE FOR (@p = value)` | plan for a representative value | bad for other values; hardcoded |
  | `OPTIMIZE FOR UNKNOWN` / local variable copy | density-based average plan, no sniff | mediocre for outliers |
  | Query Store plan forcing | pin a known-good plan | plan may rot; forced plan failures |
  | Dynamic SQL / separate procs by value class | different plans per class | code complexity |
  | `DISABLE_PARAMETER_SNIFFING` hint / db-scoped config `PARAMETER_SNIFFING = OFF` | equivalent to OPTIMIZE FOR UNKNOWN for all | blunt |
  | Better indexes (covering), stats | reduce plan sensitivity | design work |
  | PSP optimization (SQL Server 2022, compat 160) | multiple plans per query for parameter-sensitive predicates | limited scenarios |

### 10.1 [Easy] What is parameter sniffing?
**Answer**
SQL Server uses the parameter values from the first execution to compile and cache a plan, then reuses that plan for subsequent executions. It's good for reuse but can produce a poor plan when later values have very different row counts.

**Explanation**
Example: `GetOrdersByCustomer @CustomerId`: first call with a customer with 5 orders gets Nested Loops + Key Lookups; later a customer with 2M orders reuses that plan and runs for minutes with millions of lookups (or the reverse: scan plan used for a tiny customer).

**Code**
```sql
CREATE OR ALTER PROC dbo.GetOrdersByCustomer @CustomerId int AS
BEGIN
  SELECT OrderId, OrderDate, Total FROM dbo.Orders WHERE CustomerId = @CustomerId;
END
GO
EXEC dbo.GetOrdersByCustomer 42;       -- compiled for 42 (small)
EXEC dbo.GetOrdersByCustomer 1;        -- reuses the plan (customer 1 is huge)
```

### 10.2 [Follow-up] "It's fast in SSMS but slow in the application." Why?
**Answer**
SSMS and the app often use different `SET` options (notably `ARITHABORT`: SSMS ON, many drivers OFF), which gives them *separate cache entries*; each compiled with different sniffed values. The app's cached plan is bad for some values while SSMS got a fresh good plan.

**Explanation**
Plan cache keys include `set_options`. This is a quick diagnostic: run `SET ARITHABORT OFF` in SSMS and rerun - if it is slow now, you've reproduced the app's plan (parameter sniffing). Compare `set_options` in `sys.dm_exec_plan_attributes`. Don't "fix" by turning ARITHABORT on in the app; that just hides the issue until the next recompile.

**Code**
```sql
SELECT cp.plan_handle, pa.value AS set_options, st.text
FROM sys.dm_exec_cached_plans cp
CROSS APPLY sys.dm_exec_plan_attributes(cp.plan_handle) pa
CROSS APPLY sys.dm_exec_sql_text(cp.plan_handle) st
WHERE pa.attribute = 'set_options' AND st.text LIKE '%GetOrdersByCustomer%';
```

### 10.3 [Medium] Compare OPTION (RECOMPILE), OPTIMIZE FOR, and the local-variable trick. When use each?
**Answer**
RECOMPILE gives an optimal plan per execution at the cost of compile CPU - best for infrequent, variable-cost reports. OPTIMIZE FOR (value) forces a plan built for a representative value - good when one shape dominates. Local variable / OPTIMIZE FOR UNKNOWN uses average density - stable but mediocre; good when no value is typical.

**Explanation**
RECOMPILE (statement-level) also enables the parameter embedding optimization (useful for catch-all queries) but plans aren't cached/reused (and don't appear usefully in some DMVs; Query Store still captures). The local-variable trick is equivalent to OPTIMIZE FOR UNKNOWN: estimate = density x rows. OPTIMIZE FOR hardcodes a value, so data drift makes it stale. Higher-end alternatives: branch on a count (`IF @cnt > 100000 EXEC procBig ELSE EXEC procSmall`) so each branch gets its own plan, or SQL Server 2022 PSP.

**Code**
```sql
-- 1. RECOMPILE
SELECT ... FROM dbo.Orders WHERE CustomerId = @CustomerId OPTION (RECOMPILE);
-- 2. OPTIMIZE FOR value / UNKNOWN
SELECT ... FROM dbo.Orders WHERE CustomerId = @CustomerId OPTION (OPTIMIZE FOR (@CustomerId = 42));
SELECT ... FROM dbo.Orders WHERE CustomerId = @CustomerId OPTION (OPTIMIZE FOR UNKNOWN);
-- 3. Local variable
DECLARE @c int = @CustomerId;
SELECT ... FROM dbo.Orders WHERE CustomerId = @c;
```

### 10.4 [Hard] How do you use Query Store to detect and fix a parameter-sniffing regression?
**Answer**
Find queries with multiple plans and high duration variance, compare plans and runtime stats, and force the better plan with `sp_query_store_force_plan` (or via the SSMS Regressed Queries report). Monitor forcing failures and revisit if data changes.

**Explanation**
Query Store records plans per `query_id` with runtime stats per interval. Force a plan only if it's reasonably good for all common parameter values - otherwise you just swap one problem for another. Automatic plan correction (`AUTOMATIC_TUNING (FORCE_LAST_GOOD_PLAN = ON)`, 2017 Enterprise/Azure) forces last known good plan after regression. A forced plan can fail (index dropped) -> check `last_force_failure_reason_desc`; query falls back to normal compilation.

**Code**
```sql
-- Queries with multiple plans
SELECT q.query_id, qt.query_sql_text, COUNT(DISTINCT p.plan_id) AS plan_count
FROM sys.query_store_query q
JOIN sys.query_store_query_text qt ON qt.query_text_id = q.query_text_id
JOIN sys.query_store_plan p ON p.query_id = q.query_id
GROUP BY q.query_id, qt.query_sql_text
HAVING COUNT(DISTINCT p.plan_id) > 1;

-- Compare plans' avg duration
SELECT p.plan_id, AVG(rs.avg_duration)/1000.0 AS avg_ms, SUM(rs.count_executions) AS execs
FROM sys.query_store_plan p
JOIN sys.query_store_runtime_stats rs ON rs.plan_id = p.plan_id
WHERE p.query_id = 123
GROUP BY p.plan_id;

EXEC sp_query_store_force_plan @query_id = 123, @plan_id = 456;
-- EXEC sp_query_store_unforce_plan @query_id = 123, @plan_id = 456;
```

**Gotchas / follow-ups**
- Forcing doesn't work with some constructs and requires the same query hash text; plan with `RECOMPILE` hint can't be forced to be reused as there's no reuse (the forced shape can still be applied).
- Query Store hints (2022, `sys.sp_query_store_set_hints`) allow adding hints without code changes.

### 10.5 [Tricky] You apply OPTION (RECOMPILE) and the problem goes away, but CPU spikes at 500 executions/sec. What now?
**Answer**
Recompiling per call is too expensive at that rate. Use a plan-per-class approach: branch or split by parameter class, use OPTIMIZE FOR UNKNOWN, fix the root cause with a covering index, or enable SQL Server 2022 Parameter Sensitive Plan optimization / force a good plan via Query Store.

**Explanation**
Measure compile cost via `sys.dm_exec_query_stats` / compile CPU and the `SQL Compilations/sec` counter. For high-frequency queries, the best answer is usually to make the plan insensitive: a covering index removes the seek+lookup vs scan tipping behavior so both parameter classes use the same seek. Dynamic SQL splitting by a "big vs small" flag gives two cached plans.

**Code**
```sql
-- Compilations per second (compare two samples)
SELECT cntr_value FROM sys.dm_os_performance_counters
WHERE counter_name IN ('SQL Compilations/sec','SQL Re-Compilations/sec');

-- Plan-per-class with separate procs
IF @CustomerId IN (SELECT CustomerId FROM dbo.BigCustomers)
    EXEC dbo.GetOrders_Big @CustomerId;
ELSE
    EXEC dbo.GetOrders_Small @CustomerId;
```

**Gotchas / follow-ups**
- Those perf counters are cumulative; sample twice and diff.
- PSP in 2022 only applies to equality predicates on columns with skewed statistics and creates up to 3 variants.

**Interview traps**
- Calling sniffing "a bug"; it's by design.
- Using `DBCC FREEPROCCACHE` on production as a "fix".
- Recommending `OPTION (RECOMPILE)` universally.
- Forgetting that ARITHABORT/set options create separate cache entries.
- Believing the local variable trick uses the histogram (it uses density).

**Cheat sheet**
- Sniff = plan built from first params, reused.
- Fast in SSMS/slow in app -> SET options (ARITHABORT) -> sniffing.
- RECOMPILE (cost: CPU), OPTIMIZE FOR (hardcode), UNKNOWN/local var (average), Query Store force (pin).
- Covering index often removes the sensitivity.
- SQL 2022 PSP: multiple plans per parameter-sensitive query.

---

## Topic 11: Plan Cache, Recompilations & Query Store

**Study notes**

- **Plan cache**: compiled plans stored in memory keyed by query text (hash) + set options + other attributes (e.g. `user_id`/schema, database, language, DATEFORMAT). Stored procs, prepared statements, and (parameterized) ad hoc queries are cached; ad hoc statements with literals each get their own plan -> **plan cache bloat**.
- **Simple parameterization** (default): trivial queries auto-parameterized. **Forced parameterization** (`ALTER DATABASE ... SET PARAMETERIZATION FORCED`) parameterizes more. `optimize for ad hoc workloads` stores only a small stub on first compile.
- **Recompilation** triggers: schema change, statistics change (threshold crossing), `SET` option changes, `sp_recompile`, `RECOMPILE` hints/options, temp-table changes (rowcount thresholds), plan eviction by memory pressure or `DBCC FREEPROCCACHE`. **Statement-level recompile** (since 2005).
- **Query Store** (2016+; on by default for new databases in SQL Server 2022 and Azure SQL): persists queries, plans and runtime stats per interval in the user database; survives restarts. Views: `sys.query_store_query`, `_query_text`, `_plan`, `_runtime_stats`, `_runtime_stats_interval`, `_wait_stats` (2017+). Modes: `OFF`, `READ_ONLY`, `READ_WRITE`. Capture modes: `ALL`, `AUTO`, `NONE`, `CUSTOM`.
- SSMS reports: *Regressed Queries*, *Top Resource Consuming Queries*, *Tracked Queries*, *Queries With Forced Plans*, *Overall Resource Consumption*.

### 11.1 [Easy] What is the plan cache and why does it exist?
**Answer**
It's an in-memory store of compiled execution plans so repeated queries skip the expensive compile step. Plans are matched by query text/hash and environment settings.

**Explanation**
Compilation costs CPU; reuse gives consistency and speed. Ad hoc queries with embedded literals defeat reuse; use parameterized queries (EF Core does by default) or stored procedures. Memory pressure or schema/stat changes evict/invalidate plans.

**Code**
```sql
SELECT TOP 20 cp.usecounts, cp.objtype, cp.size_in_bytes, st.text
FROM sys.dm_exec_cached_plans cp
CROSS APPLY sys.dm_exec_sql_text(cp.plan_handle) st
ORDER BY cp.usecounts DESC;
```

### 11.2 [Follow-up] What is plan cache bloat and how do you fix it?
**Answer**
Thousands of single-use plans for ad hoc queries with different literals consume memory and evict useful plans. Fix by parameterizing in the application, enabling `optimize for ad hoc workloads`, or using forced parameterization as a stopgap.

**Explanation**
Check the proportion of `usecounts = 1` plans. The ad hoc stub setting stores only a compiled-plan stub until second execution, reducing memory. Forced parameterization has side effects (can cause sniffing issues for varying data, blocks some features like indexed views usage) - test it. For .NET: avoid string concatenation of values; use `SqlParameter`.

**Code**
```sql
SELECT objtype, COUNT(*) plans, SUM(CASE WHEN usecounts=1 THEN 1 ELSE 0 END) single_use,
       SUM(size_in_bytes)/1024/1024 AS MB
FROM sys.dm_exec_cached_plans GROUP BY objtype;

EXEC sp_configure 'show advanced options', 1; RECONFIGURE;
EXEC sp_configure 'optimize for ad hoc workloads', 1; RECONFIGURE;
```

### 11.3 [Medium] What causes recompilations and how do you find them?
**Answer**
Schema changes, statistics updates, SET option changes, temp table row-count changes, explicit recompile hints and plan eviction. Find them with the `SQL Re-Compilations/sec` counter, Extended Events (`sql_statement_recompile` with `recompile_cause`), and `plan_generation_num` in `sys.dm_exec_query_stats`.

**Explanation**
Recompiles are not inherently bad - sometimes the whole point (fresh stats). Problem when excessive: e.g. a proc creating/populating temp tables repeatedly triggers recompiles by row-count thresholds; DDL inside procs; `OPTION (RECOMPILE)` on hot queries. Remedies: create temp tables up-front, avoid DDL after DML on temp tables, `KEEP PLAN`/`KEEPFIXED PLAN` hints (rarely), reduce stats churn.

**Code**
```sql
SELECT TOP 20 qs.plan_generation_num, qs.execution_count, st.text
FROM sys.dm_exec_query_stats qs
CROSS APPLY sys.dm_exec_sql_text(qs.sql_handle) st
ORDER BY qs.plan_generation_num DESC;

-- Extended Events: capture recompiles with causes
CREATE EVENT SESSION xe_recompile ON SERVER
ADD EVENT sqlserver.sql_statement_recompile (ACTION (sqlserver.sql_text))
ADD TARGET package0.ring_buffer;
```

### 11.4 [Hard] How does Query Store differ from the plan cache DMVs, and how would you configure/use it in production?
**Answer**
Plan cache DMVs are volatile (lost on restart/eviction) and show only currently cached plans with cumulative stats; Query Store is persistent, per-interval, and keeps multiple plans per query, enabling regression analysis and plan forcing. In production: set `READ_WRITE`, `QUERY_CAPTURE_MODE = AUTO`, sensible `MAX_STORAGE_SIZE_MB`, `INTERVAL_LENGTH_MINUTES`, and size-based cleanup.

**Explanation**
Overhead is small but non-zero; AUTO capture avoids trivial/insignificant queries. It lives in the database, so it travels with backups/restores (and AG secondaries aren't captured on readable secondaries before 2022). If storage fills it goes READ_ONLY - monitor `actual_state_desc` vs `desired_state_desc`. Wait stats per query (2017+) help correlate. Queries with `RECOMPILE` are tracked but may lack stable plan reuse.

**Code**
```sql
ALTER DATABASE MyDb SET QUERY_STORE = ON
  (OPERATION_MODE = READ_WRITE, QUERY_CAPTURE_MODE = AUTO,
   MAX_STORAGE_SIZE_MB = 2048, INTERVAL_LENGTH_MINUTES = 60,
   SIZE_BASED_CLEANUP_MODE = AUTO, STALE_QUERY_THRESHOLD_DAYS = 30);

SELECT actual_state_desc, desired_state_desc, current_storage_size_mb, max_storage_size_mb
FROM sys.database_query_store_options;

-- Top CPU consumers in the last 24h
SELECT TOP 10 qt.query_sql_text, SUM(rs.count_executions) execs,
       SUM(rs.avg_cpu_time*rs.count_executions)/1000.0 AS total_cpu_ms
FROM sys.query_store_runtime_stats rs
JOIN sys.query_store_runtime_stats_interval i ON i.runtime_stats_interval_id = rs.runtime_stats_interval_id
JOIN sys.query_store_plan p ON p.plan_id = rs.plan_id
JOIN sys.query_store_query q ON q.query_id = p.query_id
JOIN sys.query_store_query_text qt ON qt.query_text_id = q.query_text_id
WHERE i.start_time > DATEADD(HOUR,-24,SYSUTCDATETIME())
GROUP BY qt.query_sql_text ORDER BY total_cpu_ms DESC;
```

**Gotchas / follow-ups**
- Runtime durations are in microseconds.
- Query Store can't capture queries when it's READ_ONLY (storage full).

### 11.5 [Tricky] After upgrading the compatibility level (or SQL version), some queries regress. How do you de-risk it?
**Answer**
Enable Query Store on the old compat level first to capture a baseline, upgrade the compat level, compare regressed queries, and force the old plans (or use per-query `USE HINT` / Query Store hints) while fixing root causes. This is the recommended Query Store-based upgrade workflow.

**Explanation**
New CE or Intelligent Query Processing features change plans; most improve, some regress. Pinning known-good plans gives a safety net. Alternatively use database-scoped config `LEGACY_CARDINALITY_ESTIMATION = ON` temporarily. Don't revert compat level wholesale if you can force plans per query.

**Code**
```sql
ALTER DATABASE CURRENT SET COMPATIBILITY_LEVEL = 160;
ALTER DATABASE SCOPED CONFIGURATION SET LEGACY_CARDINALITY_ESTIMATION = ON;  -- temporary
SELECT ... OPTION (USE HINT('FORCE_LEGACY_CARDINALITY_ESTIMATION'));
```

**Interview traps**
- Believing Query Store only works in Azure SQL.
- Treating plan-cache DMV stats as history (they reset on eviction/restart).
- Forgetting single-use plans waste memory.
- Assuming every recompile is bad.

**Cheat sheet**
- Plan cache = volatile; Query Store = persistent, per-interval, multi-plan.
- Cache key includes SET options.
- Bloat fix: parameterize, ad hoc stub setting, forced parameterization.
- Recompile causes: schema, stats, SET, temp tables, hints.
- Query Store: READ_WRITE + AUTO capture + force plan.

---

## Topic 12: DMVs, Wait Statistics, SET STATISTICS IO/TIME

**Study notes**

- **Wait statistics**: every request is either RUNNING, RUNNABLE (waiting for CPU: `SOS_SCHEDULER_YIELD`) or SUSPENDED (waiting on a resource: the wait type). Aggregate waits in `sys.dm_os_wait_stats` (cumulative since restart/clear) - diff over time and filter benign waits.
- Common waits:
  | Wait | Meaning |
  |---|---|
  | `PAGEIOLATCH_SH/EX` | reading data pages from disk (I/O or memory pressure) |
  | `PAGELATCH_*` | in-memory page latch contention (last-page insert, tempdb allocation) |
  | `LCK_M_*` | blocking by locks (`LCK_M_S`, `_X`, `_U`, `_IX`) |
  | `CXPACKET` / `CXCONSUMER` | parallelism coordination (`CXPACKET` alone isn't necessarily a problem) |
  | `SOS_SCHEDULER_YIELD` | CPU pressure |
  | `WRITELOG` | transaction log flush latency |
  | `RESOURCE_SEMAPHORE` | waiting for a memory grant |
  | `ASYNC_NETWORK_IO` | client not consuming results fast enough (often app row-by-row processing / huge result sets) |
  | `THREADPOOL` | out of worker threads |
- **Index DMVs**: `sys.dm_db_index_usage_stats` (seeks/scans/lookups/updates; reset on restart), `sys.dm_db_index_operational_stats` (low-level latch/lock/split), `sys.dm_db_missing_index_details/groups/group_stats` (optimizer suggestions; cleared on restart; ignore if existing indexes nearly match).
- **Query DMVs**: `sys.dm_exec_query_stats` (cumulative per cached statement), `sys.dm_exec_requests` (running now), `sys.dm_exec_sessions`, `sys.dm_exec_sql_text`, `sys.dm_exec_query_plan`, `sys.dm_tran_locks`, `sys.dm_os_waiting_tasks`.
- **SET STATISTICS IO ON**: per-table Scan count, logical reads (pages from buffer pool; most reliable metric), physical reads, read-ahead reads, LOB reads. **SET STATISTICS TIME ON**: parse/compile time and execution CPU/elapsed ms. Logical reads are deterministic; time varies with cache/concurrency.

### 12.1 [Easy] How do you measure the cost of a query in SSMS?
**Answer**
`SET STATISTICS IO, TIME ON` and look at logical reads, CPU time and elapsed time, plus the actual execution plan. Logical reads are the best repeatable metric for comparing versions of a query.

**Explanation**
Elapsed time includes waits (blocking, I/O, client network); CPU time is work done. Compare before/after changes with warm cache (run twice) or use `DBCC DROPCLEANBUFFERS` **only on test servers** for cold-cache tests.

**Code**
```sql
SET STATISTICS IO ON; SET STATISTICS TIME ON;
SELECT o.OrderId, c.Name FROM dbo.Orders o JOIN dbo.Customers c ON c.CustomerId = o.CustomerId WHERE o.OrderDate >= '20250101';
-- Output (example): Table 'Orders'. Scan count 1, logical reads 12043, physical reads 0, read-ahead reads 0.
--                   SQL Server Execution Times: CPU time = 250 ms, elapsed time = 310 ms.
```

### 12.2 [Follow-up] How do you find the most expensive queries on a server?
**Answer**
Query `sys.dm_exec_query_stats` ordered by total worker time (CPU), total logical reads or elapsed time, joined to `sys.dm_exec_sql_text`/`query_plan`; or use Query Store for persistent history.

**Explanation**
Sort by *total* (cumulative) cost to find the biggest overall consumers, and by *average* to find slow individual calls. Use `statement_start_offset` to extract the exact statement within a batch/proc. Stats are lost on eviction/restart.

**Code**
```sql
SELECT TOP 20
  qs.execution_count,
  qs.total_worker_time/1000 AS total_cpu_ms,
  qs.total_worker_time/qs.execution_count/1000 AS avg_cpu_ms,
  qs.total_logical_reads/qs.execution_count AS avg_reads,
  SUBSTRING(st.text, qs.statement_start_offset/2 + 1,
    (CASE qs.statement_end_offset WHEN -1 THEN DATALENGTH(st.text) ELSE qs.statement_end_offset END
      - qs.statement_start_offset)/2 + 1) AS stmt,
  qp.query_plan
FROM sys.dm_exec_query_stats qs
CROSS APPLY sys.dm_exec_sql_text(qs.sql_handle) st
CROSS APPLY sys.dm_exec_query_plan(qs.plan_handle) qp
ORDER BY qs.total_worker_time DESC;
```

### 12.3 [Medium] How do you use wait stats to decide what's wrong with the server?
**Answer**
Snapshot `sys.dm_os_wait_stats` twice (or clear it on a test box), exclude benign system waits, and rank by wait time. The top waits point to the bottleneck class: I/O (PAGEIOLATCH), CPU (SOS_SCHEDULER_YIELD), locking (LCK_M_*), log (WRITELOG), memory grants (RESOURCE_SEMAPHORE), client (ASYNC_NETWORK_IO).

**Explanation**
Always interpret deltas over a problem window, not lifetime totals. Then drill down: `sys.dm_exec_requests` + `sys.dm_os_waiting_tasks` for live waits, Query Store wait stats per query, and Extended Events for specifics. High `CXPACKET` + `CXCONSUMER` often is just parallel queries; check MAXDOP and cost threshold for parallelism (default 5 is too low for modern workloads - commonly raised to 25-50 after testing). `ASYNC_NETWORK_IO` usually indicates application-side slowness, not SQL Server.

**Code**
```sql
SELECT TOP 10 wait_type, wait_time_ms/1000.0 AS wait_s,
       signal_wait_time_ms/1000.0 AS signal_s, waiting_tasks_count
FROM sys.dm_os_wait_stats
WHERE wait_type NOT IN ('SLEEP_TASK','BROKER_TASK_STOP','XE_TIMER_EVENT','LAZYWRITER_SLEEP',
  'SQLTRACE_BUFFER_FLUSH','REQUEST_FOR_DEADLOCK_SEARCH','CHECKPOINT_QUEUE','DIRTY_PAGE_POLL',
  'HADR_FILESTREAM_IOMGR_IOCOMPLETION','WAITFOR','LOGMGR_QUEUE','BROKER_TO_FLUSH','XE_DISPATCHER_WAIT')
ORDER BY wait_time_ms DESC;

-- Live: what is blocked on what right now
SELECT r.session_id, r.status, r.wait_type, r.wait_time, r.blocking_session_id, r.wait_resource,
       t.text
FROM sys.dm_exec_requests r CROSS APPLY sys.dm_exec_sql_text(r.sql_handle) t
WHERE r.session_id > 50 AND r.session_id <> @@SPID;
```

### 12.4 [Hard] How do you use the missing-index DMVs responsibly?
**Answer**
Treat them as hints to investigate: rank by `avg_total_user_cost * avg_user_impact * (user_seeks + user_scans)`, check for overlap with existing indexes, merge suggestions into fewer indexes, validate with a test and actual plans, and consider write overhead. Never blindly create all suggestions.

**Explanation**
Limitations: suggestions come from single-query optimization, don't consider existing index consolidation, order key columns equality-first without regard to actual best order, can suggest huge INCLUDE lists, are limited to 500 groups, ignore filtered/columnstore, and reset on restart/index changes. Also check `sys.dm_db_index_usage_stats` for existing near-duplicates.

**Code**
```sql
SELECT TOP 20
  migs.avg_total_user_cost * migs.avg_user_impact * (migs.user_seeks + migs.user_scans) AS improvement_measure,
  mid.statement, mid.equality_columns, mid.inequality_columns, mid.included_columns,
  migs.user_seeks, migs.user_scans
FROM sys.dm_db_missing_index_group_stats migs
JOIN sys.dm_db_missing_index_groups mig ON mig.index_group_handle = migs.group_handle
JOIN sys.dm_db_missing_index_details mid ON mid.index_handle = mig.index_handle
WHERE mid.database_id = DB_ID()
ORDER BY improvement_measure DESC;
```

**Gotchas / follow-ups**
- Same table may get 5 near-duplicate suggestions; combine.
- Suggestions can't see queries that never ran since restart.

### 12.5 [Tricky] STATISTICS IO shows tiny reads yet the query takes 30 seconds. What could explain it?
**Answer**
Time spent isn't in page reads: blocking (LCK_M_*), waiting for a memory grant (RESOURCE_SEMAPHORE), slow client consumption (ASYNC_NETWORK_IO), scalar UDFs/CPU-bound expressions, compile time, or reads in other sessions (triggers) / not counted (e.g. scalar functions' internal reads aren't shown).

**Explanation**
Check `STATISTICS TIME` (compile vs execution), the request's waits in `sys.dm_exec_requests`, and the actual plan's wait stats. Scalar UDFs run row-by-row (inlined in 2019+ compat 150 where eligible) and their reads/CPU aren't attributed in STATISTICS IO of the outer statement. Large result set rendering in SSMS itself can dominate - use "Discard results after execution".

**Code**
```sql
SET STATISTICS TIME ON;
SELECT dbo.fn_ComputeTax(Total) FROM dbo.Orders;    -- scalar UDF: CPU time high, IO low
-- Check waits for the session
SELECT wait_type, wait_time_ms FROM sys.dm_exec_session_wait_stats WHERE session_id = @@SPID;  -- SQL Server 2016+
```

**Interview traps**
- Treating cumulative `dm_os_wait_stats` as current.
- Saying CXPACKET is always bad.
- Creating every missing-index suggestion.
- Relying on elapsed time alone to compare tuning attempts.
- Forgetting DMV data resets on restart.

**Cheat sheet**
- IO/TIME ON: logical reads = best comparison metric.
- `dm_exec_query_stats` for top queries; Query Store for history.
- Wait stats: delta, filter benign, map to resource.
- PAGEIOLATCH=disk, PAGELATCH=memory latch, LCK=blocking, WRITELOG=log, ASYNC_NETWORK_IO=client.
- Missing-index DMVs = hints only.

---

## Topic 13: Partitioning, Compression & In-Memory OLTP

**Study notes**

- **Partitioning** splits one logical table/index into physical partitions by a partitioning column. Components: **partition function** (boundary values, `RANGE LEFT/RIGHT`), **partition scheme** (maps partitions to filegroups), and the table/index created `ON scheme(column)`.
```
 Function: boundaries 2024-01-01, 2025-01-01, 2026-01-01 (RANGE RIGHT)
 Partition 1: < 2024-01-01 | P2: [2024,2025) | P3: [2025,2026) | P4: >= 2026
```
- Benefits: manageability (per-partition rebuild/compression/backup, **fast SWITCH** of data in/out as metadata-only), **partition elimination** for queries filtering on the partition key, sliding-window archival. It is **not primarily a performance feature** - queries that don't filter on the partition column can get slower (more B-trees to touch).
- **Aligned indexes**: index partitioned the same way as the table (required for SWITCH). Partition column must be part of the clustered key/unique indexes (unique indexes must include it).
- **SWITCH** requirements: identical structure/indexes, same filegroup, a CHECK constraint on the staging table proving the range, target partition empty (for switch-in), not-null partition column.
- **Sliding window**: add a new future partition with `SPLIT RANGE` (keep it empty!), switch oldest partition out to a staging table, truncate/archive, `MERGE RANGE` the empty boundary.
- **Compression**: ROW (variable-length storage for fixed types, little CPU), PAGE (ROW + prefix + dictionary compression per page, higher ratio, more CPU), columnstore/archival compression. Reduces I/O and memory footprint; costs CPU on write/read. Available in Standard since SQL Server 2016 SP1. Estimate with `sp_estimate_data_compression_savings`. Rebuild required to apply.
- **In-Memory OLTP** (Hekaton, 2014+): memory-optimized tables (durable `SCHEMA_AND_DATA` or non-durable `SCHEMA_ONLY`), **lock-free/latch-free** MVCC optimistic concurrency, hash indexes (point lookups, fixed bucket count) and range (Bw-tree) nonclustered indexes, natively compiled procedures (T-SQL compiled to machine code). Good for high-concurrency inserts, session state, staging, tempdb metadata contention (memory-optimized tempdb metadata, 2019). Limitations: memory-resident, feature restrictions, no cross-database transactions, no row-versioning surprises (snapshot isolation conflicts error 41302), migration effort.

### 13.1 [Easy] What is table partitioning and what is it for?
**Answer**
It divides a large table into partitions by a key (usually date) via a partition function and scheme. Its main value is manageability (maintaining/archiving partitions independently, switching partitions in/out instantly); it may also enable partition elimination.

**Explanation**
Common misconception: "partition it to make queries faster". Performance gains appear only when predicates include the partition key (elimination) or for parallel per-partition operations; otherwise there can be overhead. A well-indexed unpartitioned table often outperforms a partitioned one for OLTP seeks.

**Code**
```sql
CREATE PARTITION FUNCTION pf_OrderDate (datetime2(0))
AS RANGE RIGHT FOR VALUES ('2024-01-01','2025-01-01','2026-01-01');

CREATE PARTITION SCHEME ps_OrderDate AS PARTITION pf_OrderDate ALL TO ([PRIMARY]);

CREATE TABLE dbo.OrdersP (
  OrderId bigint NOT NULL, OrderDate datetime2(0) NOT NULL, Total decimal(12,2) NOT NULL,
  CONSTRAINT PK_OrdersP PRIMARY KEY CLUSTERED (OrderDate, OrderId)
) ON ps_OrderDate (OrderDate);

SELECT $PARTITION.pf_OrderDate('2025-06-01') AS partition_no;
```

### 13.2 [Follow-up] RANGE LEFT vs RANGE RIGHT - which would you pick for dates?
**Answer**
RANGE RIGHT makes each boundary value the *first* value of its partition, so `'2025-01-01'` starts the 2025 partition. That's usually the natural choice for dates/datetime.

**Explanation**
With RANGE LEFT the boundary value belongs to the lower partition (boundary = last value), which is awkward for `datetime` where "last value" is e.g. `23:59:59.997`. RIGHT with a half-open interval avoids precision issues and matches how you write predicates `>= start AND < next`.

**Code**
```sql
-- Which partition has rows, and how many?
SELECT p.partition_number, p.rows, prv.value AS lower_boundary
FROM sys.partitions p
JOIN sys.indexes i ON i.object_id = p.object_id AND i.index_id = p.index_id
LEFT JOIN sys.partition_schemes ps ON ps.data_space_id = i.data_space_id
LEFT JOIN sys.partition_range_values prv ON prv.function_id = ps.function_id AND prv.boundary_id = p.partition_number - 1
WHERE p.object_id = OBJECT_ID('dbo.OrdersP') AND i.index_id <= 1;
```

### 13.3 [Medium] Explain the sliding-window archive pattern and the SWITCH requirements.
**Answer**
Each period you split a new empty partition at the head and switch the oldest partition out to a staging table (metadata operation), archive/truncate it, then merge away the empty boundary. SWITCH needs identical schema/indexes, same filegroup, a matching CHECK constraint and an empty target.

**Explanation**
Always keep the last partition empty so `SPLIT RANGE` doesn't move data (a split of a populated partition is a heavy, logged operation). Likewise merge only empty partitions. Switch is nearly instant, taking only a brief Sch-M lock - massively cheaper than DELETE. Switching in bulk-loaded staging data works the same in reverse. Use `NEXT USED` filegroup before splitting.

**Code**
```sql
-- 1. add future partition
ALTER PARTITION SCHEME ps_OrderDate NEXT USED [PRIMARY];
ALTER PARTITION FUNCTION pf_OrderDate() SPLIT RANGE ('2027-01-01');

-- 2. switch oldest out (staging table has same structure, PK, on same filegroup)
ALTER TABLE dbo.OrdersP SWITCH PARTITION 1 TO dbo.OrdersP_Staging;
TRUNCATE TABLE dbo.OrdersP_Staging;   -- or archive first

-- 3. merge the emptied boundary
ALTER PARTITION FUNCTION pf_OrderDate() MERGE RANGE ('2024-01-01');
```

**Gotchas / follow-ups**
- Statistics on partitioned tables are table-wide (`INCREMENTAL = ON` stats per partition available since 2014).
- Foreign keys referencing a partitioned table complicate switching (SQL Server 2016+ allows some cases; check restrictions).

### 13.4 [Hard] Compare ROW and PAGE compression. When would you use each, and what are the risks?
**Answer**
ROW compression stores fixed-length types compactly (cheap CPU, modest savings); PAGE compression adds prefix/dictionary compression for a larger ratio at higher CPU. Use PAGE for read-heavy, scan-heavy, cold data and ROW for write-heavy; always estimate and test.

**Explanation**
Compression saves I/O and buffer-pool memory (pages stay compressed in memory) but costs CPU on compress at write and decompress at access. Rebuilds are needed to apply or change it; online possible with Enterprise. Heavily updated hot partitions may suffer from PAGE. Partition-level choice: PAGE for old partitions, ROW/none for current. Won't help already-compressed data (images, random GUIDs). LOB/off-row data isn't page-compressed except via COMPRESS()/GZIP functions or columnstore archival.

**Code**
```sql
EXEC sp_estimate_data_compression_savings 'dbo','Orders',NULL,NULL,'PAGE';

ALTER TABLE dbo.Orders REBUILD WITH (DATA_COMPRESSION = PAGE);
ALTER INDEX IX_Orders_Customer ON dbo.Orders REBUILD WITH (DATA_COMPRESSION = ROW);
ALTER TABLE dbo.OrdersP REBUILD PARTITION = 1 WITH (DATA_COMPRESSION = PAGE);
```

### 13.5 [Tricky] When is In-Memory OLTP the right answer, and when not?
**Answer**
Right for extreme-concurrency, short transactions with latch/lock contention (high-rate inserts, session/state tables, ETL staging with SCHEMA_ONLY, tempdb-like workloads). Wrong as a general speed-up for scan/reporting queries or for tables bigger than available memory.

**Explanation**
It removes locks/latches via optimistic MVCC, so write conflicts surface as errors (41302 update conflict, 41325 serializable validation) requiring retry logic in the app. Hash index bucket count must be sized to the expected distinct key count (too low -> long chains; too high -> wasted memory); hash indexes support equality seeks only, so use range indexes for ordered/inequality queries. Durable tables still log and recover on startup (long recovery for large data). Native compilation requires a limited T-SQL surface. Quick win alternatives: memory-optimized table variables to replace tempdb table variables; `OPTIMIZE_FOR_SEQUENTIAL_KEY` for the last-page problem.

**Code**
```sql
-- Requires a MEMORY_OPTIMIZED_DATA filegroup
CREATE TABLE dbo.SessionState (
  SessionId uniqueidentifier NOT NULL PRIMARY KEY NONCLUSTERED HASH WITH (BUCKET_COUNT = 1000000),
  Payload   varbinary(8000) NULL,
  Expires   datetime2 NOT NULL,
  INDEX ix_expires NONCLUSTERED (Expires)
) WITH (MEMORY_OPTIMIZED = ON, DURABILITY = SCHEMA_AND_DATA);
```

**Gotchas / follow-ups**
- Needs retry logic for conflict errors.
- Memory-optimized table size limits in non-Enterprise editions (memory quotas apply in Standard).
- ALTER of bucket count requires rebuild (`ALTER TABLE ... ALTER INDEX ... REBUILD`).

**Interview traps**
- "Partitioning speeds up all queries."
- Splitting/merging populated partitions casually.
- Using PAGE compression on write-hot tables without testing.
- Thinking In-Memory OLTP = just "put the DB in RAM" (the buffer pool already caches).

**Cheat sheet**
- Partition = manageability + switch; elimination needs key in predicate.
- RANGE RIGHT for dates; keep edge partitions empty before SPLIT/MERGE.
- SWITCH: same schema/indexes/filegroup + CHECK + empty target.
- ROW (light) vs PAGE (heavier); estimate first.
- In-Memory OLTP: contention-bound OLTP; plan for conflict retries.

---

## Topic 14: Index Overhead, Paging, Hints, NOLOCK & tempdb

**Study notes**

- **Index write cost**: every INSERT/DELETE touches all NCIs; an UPDATE touches every NCI containing the changed column (key or INCLUDE) - and updating a clustering key column touches all of them. More indexes -> slower writes, more log, more locking/deadlocks, more storage/memory/backup/maintenance, slower stats maintenance. Rule: index for the workload, drop unused/duplicate ones.
- **Paging**: `OFFSET n ROWS FETCH NEXT m ROWS ONLY` (needs ORDER BY) is simple but cost grows with offset (it reads and discards n rows). **Keyset (seek) pagination** `WHERE (SortKey, Id) > (@lastKey, @lastId) ORDER BY SortKey, Id` with a supporting index is O(page size). EF Core `Skip/Take` -> OFFSET/FETCH.
- **Hints**: table (`WITH (INDEX(...))`, `FORCESEEK`, `NOLOCK`), join (`LOOP/HASH/MERGE`), query (`OPTION (RECOMPILE, MAXDOP n, OPTIMIZE FOR, USE HINT(...), FAST n)`). Avoid because they freeze today's assumptions: data/schema change, optimizer improves, and a dropped index can cause errors (8622). Prefer fixing statistics/indexes/query shape; use Query Store plan forcing or Query Store hints (2022) as less invasive alternatives.
- **NOLOCK / READ UNCOMMITTED**: no shared locks, ignores others' exclusive locks -> **dirty reads**, can read rows twice or miss rows (allocation-order scans during page splits), can fail with error 601 ("could not continue scan with NOLOCK due to data movement"), read in-flight rollbacks. Not a go-faster switch. Alternative: **READ_COMMITTED_SNAPSHOT (RCSI)** or snapshot isolation (row versioning in tempdb/PVS) - readers don't block writers.
- **tempdb**: shared by temp tables/variables, sort/hash spills, row versions, worktables, internal objects. Contention: **allocation page contention** (PAGELATCH_UP/EX on PFS 2:1:1, GAM 2:1:2, SGAM 2:1:3) and metadata contention. Mitigations: multiple equally sized data files (start at 1 per core up to 8, then add in groups of 4 if contention persists), trace flags 1117/1118 behavior is default from 2016, memory-optimized tempdb metadata (2019), avoid unnecessary temp objects, fast storage, don't autogrow tiny.

### 14.1 [Easy] Why can adding more indexes hurt performance?
**Answer**
Every data modification must maintain every affected index, so writes get slower and generate more log, locks and I/O. Indexes also consume storage/memory and extend maintenance.

**Explanation**
A table with 12 NCIs means an INSERT performs ~13 B-tree insertions. Updates to a column present in many indexes touch all of them. Typical OLTP guidance: a handful of well-designed indexes per table; reporting tables can carry more. Prove value with usage stats (reads vs updates).

**Code**
```sql
SELECT OBJECT_NAME(i.object_id) AS tbl, i.name,
       ISNULL(us.user_seeks+us.user_scans+us.user_lookups,0) AS reads,
       ISNULL(us.user_updates,0) AS writes
FROM sys.indexes i
LEFT JOIN sys.dm_db_index_usage_stats us ON us.object_id=i.object_id AND us.index_id=i.index_id AND us.database_id=DB_ID()
WHERE i.object_id = OBJECT_ID('dbo.Orders') AND i.index_id > 0
ORDER BY reads, writes DESC;   -- reads=0, writes high = drop candidates
```

### 14.2 [Follow-up] Why is OFFSET/FETCH slow for deep pages and what's the alternative?
**Answer**
SQL Server must still produce and skip all rows before the offset, so cost grows linearly with page number. Keyset pagination remembers the last key seen and seeks past it, giving constant cost per page.

**Explanation**
Keyset requires a deterministic, unique ordering (add the PK as tiebreaker) and an index matching the sort. Downsides: can't jump to arbitrary page N, more complex with multi-column sorts. Always include `ORDER BY` (without it paging is nondeterministic). Also get total counts separately/cached since `COUNT(*)` over a large filtered set is costly. EF Core: `.Where(o => o.CreatedAt < last || (o.CreatedAt == last && o.Id < lastId)).OrderByDescending(...).Take(n)`.

**Code**
```sql
-- OFFSET paging (cost grows with @page)
SELECT OrderId, CreatedAt FROM dbo.Orders ORDER BY CreatedAt DESC, OrderId DESC
OFFSET (@page-1)*@size ROWS FETCH NEXT @size ROWS ONLY;

-- Keyset paging (constant cost) with index (CreatedAt DESC, OrderId DESC)
SELECT TOP (@size) OrderId, CreatedAt FROM dbo.Orders
WHERE CreatedAt < @lastCreatedAt OR (CreatedAt = @lastCreatedAt AND OrderId < @lastOrderId)
ORDER BY CreatedAt DESC, OrderId DESC;
```

### 14.3 [Medium] What are the dangers of NOLOCK and what should you use instead?
**Answer**
NOLOCK allows dirty reads, duplicated/missed rows during page splits or allocation-order scans, and occasional errors; it also doesn't avoid Sch-S locks. Use READ_COMMITTED_SNAPSHOT or SNAPSHOT isolation so readers see a consistent committed version without blocking writers.

**Explanation**
Real consequences: reports with wrong totals, double-counted rows, phantom ghost data from rolled-back transactions. Acceptable only for rough monitoring/diagnostic queries where inaccuracy is fine. RCSI costs tempdb/PVS space and version-store overhead, and changes semantics for code relying on blocking (e.g. "check then insert" patterns need `UPDLOCK`/`HOLDLOCK`). In EF, `TransactionScope` with `IsolationLevel.ReadUncommitted` is the equivalent mistake. Note that Azure SQL Database enables RCSI by default.

**Code**
```sql
ALTER DATABASE MyDb SET READ_COMMITTED_SNAPSHOT ON WITH ROLLBACK IMMEDIATE;
ALTER DATABASE MyDb SET ALLOW_SNAPSHOT_ISOLATION ON;

-- Version store usage
SELECT DB_NAME(database_id) AS db, reserved_page_count*8/1024 AS version_store_mb
FROM sys.dm_tran_version_store_space_usage;
```

### 14.4 [Hard] How do you diagnose and mitigate tempdb contention?
**Answer**
Look for PAGELATCH_UP/EX waits whose `wait_resource` is in database 2 on PFS/GAM/SGAM pages (page 1, 2, 3 or every 8088/ 511232 pages). Mitigate with multiple equally sized tempdb data files (up to 8 per start), reduce temp object churn, use memory-optimized tempdb metadata (2019), and fix queries spilling to tempdb.

**Explanation**
Allocation contention occurs when many sessions create/drop temp objects concurrently. Equal file sizing + proportional fill spreads allocations. Metadata contention (system tables) is addressed by memory-optimized tempdb metadata (`ALTER SERVER CONFIGURATION SET MEMORY_OPTIMIZED TEMPDB_METADATA = ON`, restart). Also large version stores, spills, and sorts consume tempdb I/O; check with `sys.dm_db_task_space_usage` / `sys.dm_db_session_space_usage`. Temp table caching in procs reduces create/drop overhead; DDL on a temp table (e.g. adding named constraints/indexes after creation) can defeat caching.

**Code**
```sql
SELECT session_id, wait_type, wait_resource, wait_duration_ms
FROM sys.dm_os_waiting_tasks
WHERE wait_type LIKE 'PAGELATCH%' AND wait_resource LIKE '2:%';

SELECT name, size*8/1024 AS size_mb, growth FROM tempdb.sys.database_files;

-- Who is using tempdb space
SELECT TOP 10 session_id, user_objects_alloc_page_count*8/1024 AS user_mb,
       internal_objects_alloc_page_count*8/1024 AS internal_mb
FROM sys.dm_db_session_space_usage ORDER BY internal_objects_alloc_page_count DESC;
```

**Gotchas / follow-ups**
- More files than needed can slow things (proportional fill/ checkpoint overhead); stop at 8 and test.
- Uneven file sizes defeat proportional fill.

### 14.5 [Tricky] Should you ever use query hints like FORCESEEK, INDEX(...) or MAXDOP? What's your policy?
**Answer**
Hints are last-resort tactical tools: fine for a documented emergency or a proven optimizer limitation, but they hard-code decisions that rot as data/schema changes. First fix statistics, indexes, SARGability and parameter sniffing; if pinning is required, prefer Query Store plan forcing or hints that are narrow and reviewed.

**Explanation**
Examples that are generally acceptable: `OPTION (RECOMPILE)` for catch-all queries, `MAXDOP 1` on a specific parallelism-prone query, `OPTIMIZE FOR UNKNOWN` for known skew, `FAST n` for UI-first-rows. Risky: `FORCE ORDER`, join hints, `INDEX(n)` by number (use name), `NOLOCK`. Hints in EF require interceptors/`FromSqlRaw` or `TagWith` plus Query Store hints rather than code changes. Forced index errors if the index is dropped/disabled.

**Code**
```sql
SELECT OrderId FROM dbo.Orders WITH (INDEX (IX_Orders_Customer)) WHERE CustomerId = @c;   -- brittle
SELECT ... FROM dbo.Orders WHERE ... OPTION (MAXDOP 1);                                    -- targeted

-- SQL Server 2022 / Azure SQL: Query Store hints (no code change)
EXEC sys.sp_query_store_set_hints @query_id = 123, @query_hints = N'OPTION (MAXDOP 1)';
```

**Gotchas / follow-ups**
- Statement hints stay forever unless someone removes them; document with a comment/ticket.

**Interview traps**
- "NOLOCK makes it faster, so it's fine on reports."
- Deep OFFSET paging in the UI "because EF Skip/Take is simple".
- Adding an index per slow query without considering write cost.
- Believing more tempdb files always help.
- Forcing an index by hint instead of fixing the predicate.

**Cheat sheet**
- Each NCI adds write cost; drop reads=0 indexes (after full cycle).
- Keyset paging > OFFSET for deep pages.
- NOLOCK = dirty/duplicate/missing rows; use RCSI/snapshot.
- tempdb: PAGELATCH on 2:1:1/2/3 -> equal-sized files, fewer temp objects.
- Hints last; Query Store force/hints preferred.

---

## Topic 15: Tuning Methodology & Scenario Ladders (slow today, timing-out report, slow EF query)

**Study notes**

**Step-by-step tuning methodology**
1. **Define the problem**: which query/endpoint, how slow, since when, what changed (deploy, data growth, config, stats, patching, load). Get a baseline (Query Store/APM).
2. **Scope**: is it one query or the whole server? Check waits, CPU, I/O latency, blocking, memory.
3. **Capture the actual plan and parameters** for the slow case (and a fast case if exists).
4. **Measure** with `SET STATISTICS IO, TIME ON`.
5. **Find the biggest cost**: highest logical reads / most executions / first big estimate error.
6. **Hypothesize + fix the cause**: stats, SARGability, index (key order, covering), query rewrite, set-based instead of row-by-row, parameter sniffing fix, data volume reduction (paging/filters).
7. **Test** on realistic data; compare reads/CPU/time and plan; check effect on writes and other queries.
8. **Deploy safely** (online, off-peak), **monitor** (Query Store), keep rollback.
Principle: change one thing at a time; measure; prefer fixing the cause over symptom hints.

```
Symptom -> Is it SQL Server? (waits, blocking, client)
        -> Which query? (Query Store / DMVs)
        -> Why? (plan: estimates, scans, lookups, spills, params)
        -> Fix (stats / index / rewrite / sniffing) -> Verify -> Monitor
```

### 15.1 [Easy] Walk me through how you'd tune a slow query.
**Answer**
Reproduce with real parameters, get the actual plan plus STATISTICS IO/TIME, identify the costliest operators and estimate errors, then fix the cause (statistics, SARGability, index, query shape, parameter sniffing) and re-measure. Verify the change doesn't hurt writes or other queries.

**Explanation**
Resist guessing: first prove where time goes (CPU, reads, waits/blocking, client). Check for easy wins - non-SARGable predicates, missing/covering index, SELECT *, implicit conversions, row-by-row loops/cursors, scalar UDFs, over-fetching rows. Keep before/after numbers for the team and the interview story.

**Code**
```sql
SET STATISTICS IO, TIME ON;
-- run query with representative parameters; capture actual plan (Ctrl+M)
-- look for: Table Scan/Clustered Index Scan, Key Lookup (high executions), Sort/Hash spill warnings,
-- big gaps between Estimated and Actual Rows, Parallelism skew, implicit conversion warnings
```

### 15.2 [Follow-up] Scenario: "The query was fast yesterday, slow today." What do you do?
**Answer**
Establish what changed: plan change (parameter sniffing, recompile, stats update, plan eviction), data volume/skew shift, blocking, resource contention, deployment/config/compat-level change. Query Store lets you compare yesterday's plan and metrics against today's and force the good plan as immediate mitigation.

**Explanation**
Checklist: (1) Is it blocking/waits right now? (`sys.dm_exec_requests`, `sp_whoisactive` if installed). (2) Query Store: multiple plans for the query? duration/reads regressions at a timestamp? (3) Stats: `last_updated`/`modification_counter` - a stats refresh or ascending-key issue? (4) Plan cache: was the plan evicted/recompiled (restart, memory pressure, schema change)? (5) Parameters: new skewed value? (6) Environment: disk latency, memory pressure, new job/backup/index rebuild running concurrently, AG redo lag, CPU from another query. (7) Deployments: new index/dropped index/compat level/CU. Mitigate (force plan, update stats), then root-cause.

**Code**
```sql
-- Regression check: same query, per-plan stats by day
SELECT p.plan_id, CAST(i.start_time AS date) AS day, 
       SUM(rs.count_executions) AS execs, AVG(rs.avg_duration)/1000.0 AS avg_ms, AVG(rs.avg_logical_io_reads) AS avg_reads
FROM sys.query_store_runtime_stats rs
JOIN sys.query_store_runtime_stats_interval i ON i.runtime_stats_interval_id = rs.runtime_stats_interval_id
JOIN sys.query_store_plan p ON p.plan_id = rs.plan_id
WHERE p.query_id = @query_id
GROUP BY p.plan_id, CAST(i.start_time AS date)
ORDER BY day, p.plan_id;

-- Is anything blocked right now?
SELECT session_id, blocking_session_id, wait_type, wait_time, DB_NAME(database_id) AS db
FROM sys.dm_exec_requests WHERE blocking_session_id <> 0;
```

### 15.3 [Medium] Scenario: "A report times out after 30 seconds." How do you approach it?
**Answer**
Capture the actual plan and waits, then decide between: fix the plan (indexes/stats/SARGability/rewrite), reduce work (pre-aggregation, indexed view, columnstore, summary table, narrower date range), or change architecture (read replica/reporting DB, async generation). Also check isolation and blocking, and the app's `CommandTimeout`.

**Explanation**
Typical causes: scanning large fact tables without date filters; non-SARGable filters; large sorts/spills; multiple correlated subqueries; scalar UDFs; `DISTINCT` hiding duplicate joins; many-to-many joins exploding rows; parameter sniffing; blocked by OLTP writers (use RCSI or a readable secondary); returning huge result sets (ASYNC_NETWORK_IO). Techniques: break into steps with temp tables (stats + accurate estimates), window functions instead of self-joins, covering/columnstore indexes, partition elimination, `OPTION (RECOMPILE)` for catch-all filters, `MAXDOP`/memory grant tuning. Don't raise the timeout as the "fix".

**Code**
```sql
-- Staging heavy intermediate result gives the optimizer real stats
SELECT CustomerId, SUM(Total) AS Total
INTO #Agg
FROM dbo.Orders
WHERE OrderDate >= @from AND OrderDate < @to
GROUP BY CustomerId;
CREATE CLUSTERED INDEX cx ON #Agg (CustomerId);

SELECT c.Name, a.Total FROM #Agg a JOIN dbo.Customers c ON c.CustomerId = a.CustomerId;

-- Columnstore on a reporting copy
CREATE NONCLUSTERED COLUMNSTORE INDEX NCCI_Orders_Report
ON dbo.Orders (OrderDate, CustomerId, Total);
```

### 15.4 [Hard] Scenario: "An EF Core query is slow." How do you investigate and fix it?
**Answer**
Capture the actual SQL (logging `LogTo`/`ToQueryString()`/SQL Profiler/XEvents/Query Store), run it in SSMS with the same parameter types, and inspect the plan. Common EF causes: loading entire entities instead of projecting, N+1 queries, cartesian explosion from multiple `Include`s (use `AsSplitQuery`), client evaluation, `nvarchar` vs `varchar` mismatch, `Contains` -> `LIKE '%x%'`, deep `Skip/Take`, tracking overhead, and missing indexes.

**Explanation**
Investigation steps: (1) enable logging: `optionsBuilder.LogTo(Console.WriteLine, LogLevel.Information)` or `query.ToQueryString()`. (2) Count queries per request (N+1). (3) Compare rows returned vs rows needed. (4) Check the plan for scans, implicit conversions, lookups. Fixes: `.Select()` projections; `AsNoTracking()` for read-only; `AsSplitQuery()` for multi-collection `Include`; compiled queries for hot paths (CPU, not DB); filter early; keyset pagination; map `IsUnicode(false)`; add indexes; avoid `ToList()` before `Where`; use `ExecuteUpdate/ExecuteDelete` (EF 7+) to avoid loading entities; use raw SQL or views for complex reports. Verify which SQL is generated rather than guess.

**Code**
```csharp
var q = db.Orders
    .AsNoTracking()
    .Where(o => o.CustomerId == id && o.OrderDate >= from)
    .Select(o => new { o.OrderId, o.OrderDate, o.Total })
    .OrderByDescending(o => o.OrderDate).Take(50);
Console.WriteLine(q.ToQueryString());   // paste into SSMS with correct parameter types
```
```sql
-- Find the EF statement in the plan cache / Query Store by text
SELECT TOP 10 qs.total_logical_reads, qs.execution_count, st.text
FROM sys.dm_exec_query_stats qs CROSS APPLY sys.dm_exec_sql_text(qs.sql_handle) st
WHERE st.text LIKE N'%FROM [[]Orders]%'   -- escape '[' in LIKE
ORDER BY qs.total_logical_reads DESC;
```

**Gotchas / follow-ups**
- `ToQueryString()` shows parameter declarations as `DECLARE @p0 ...`; running as-is makes them local variables (changes sniffing behaviour!). Use `sp_executesql` to reproduce the app's behavior.
- `TagWith("name")` adds a comment to find the query in Query Store/DMVs.
- Split queries trade fewer duplicated rows for multiple round trips and potential consistency issues without a snapshot.

### 15.5 [Tricky] Scenario: Everything is slow, CPU 100%. What's your first 10 minutes?
**Answer**
Confirm the bottleneck (CPU vs I/O vs blocking vs memory) using current requests and wait stats, find the top CPU queries (Query Store/`dm_exec_query_stats`), and look for a sudden change - plan regression, a new query, runaway parallelism, missing index, compile storm. Mitigate (force plan, kill the runaway, add an index) before deeper root-cause.

**Explanation**
Order of checks: (1) `sys.dm_exec_requests` for current heavy sessions and blocking; (2) wait deltas (SOS_SCHEDULER_YIELD, CXPACKET, THREADPOOL); (3) perf counters: Batch Requests/sec, Compilations/sec (compile storm), Page Life Expectancy trend, buffer cache; (4) top queries by CPU in the last hour (Query Store); (5) check for stats updates, index rebuilds, backups, checkdb jobs; (6) also non-SQL causes: other processes on the host, VM CPU steal. Communicate status while diagnosing.

**Code**
```sql
SELECT TOP 10 r.session_id, r.cpu_time, r.total_elapsed_time, r.logical_reads, r.dop, r.wait_type,
       SUBSTRING(t.text, r.statement_start_offset/2+1, 200) AS stmt
FROM sys.dm_exec_requests r CROSS APPLY sys.dm_exec_sql_text(r.sql_handle) t
WHERE r.session_id > 50 AND r.session_id <> @@SPID
ORDER BY r.cpu_time DESC;
```

**Gotchas / follow-ups**
- Killing a long query that's mid-rollback can take as long again to roll back; check `percent_complete`.
- High CPU with high `SQL Compilations/sec` points to unparameterized SQL, not heavy queries.

**Interview traps**
- Jumping to "add an index" before looking at the plan and waits.
- Using only elapsed time as the metric.
- Reproducing with local variables instead of the real parameter types/values.
- Fixing symptoms (bigger timeout, NOLOCK, restart) rather than causes.
- Not mentioning verification and monitoring after the fix.

**Cheat sheet**
- Baseline -> scope -> actual plan -> measure -> fix cause -> test -> deploy -> monitor.
- Slow today: blocking? plan change? stats? params? environment? deploy?
- Report timeout: reduce work first (filters, pre-aggregate, columnstore), then tune.
- EF: log SQL, project, AsNoTracking, split queries, IsUnicode(false), keyset paging.
- One change at a time; keep before/after metrics.
