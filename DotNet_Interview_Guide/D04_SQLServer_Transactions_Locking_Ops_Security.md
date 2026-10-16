# D04 - SQL Server: Transactions, Locking, Operations & Security

**How to use:** Read the *Study notes* of a topic first, then work the ladder top to bottom (Easy -> Tricky); cover the Answer and try to say it aloud before reading.
Run the **Code** blocks in SSMS (two query windows for locking/deadlock demos) against a scratch database - never against production.

## Table of Contents
1. [ACID & how SQL Server implements durability](#topic-1-acid--how-sql-server-implements-durability)
2. [Transactions: autocommit, implicit, explicit, savepoints, XACT_ABORT](#topic-2-transactions)
3. [Isolation levels & anomalies](#topic-3-isolation-levels--anomalies)
4. [READ_COMMITTED_SNAPSHOT vs SNAPSHOT & row versioning](#topic-4-rcsi-vs-snapshot--row-versioning)
5. [Locking: modes, granularity, escalation, compatibility](#topic-5-locking)
6. [Blocking diagnosis](#topic-6-blocking-diagnosis)
7. [Deadlocks](#topic-7-deadlocks)
8. [Table hints: NOLOCK, READPAST, UPDLOCK, ROWLOCK](#topic-8-table-hints)
9. [Optimistic vs pessimistic concurrency, rowversion, queue pattern](#topic-9-concurrency-control-rowversion--queue-pattern)
10. [Distributed transactions, MSDTC, two-phase commit](#topic-10-distributed-transactions)
11. [Recovery models, log growth, backups, restore, CHECKDB](#topic-11-recovery-models-log-backups--restore)
12. [High availability & DR comparison](#topic-12-high-availability--dr)
13. [Security](#topic-13-security)
14. [Multi-tenancy, soft delete/archive, batched deletes](#topic-14-multi-tenancy-soft-delete--large-deletes)
15. [Connection pooling & Azure SQL specifics](#topic-15-connection-pooling--azure-sql)
16. [Schema migrations & monitoring](#topic-16-migrations--monitoring)

---

## Topic 1: ACID & how SQL Server implements durability

**Study notes**

- **A**tomicity: all or nothing. **C**onsistency: constraints/rules hold before and after. **I**solation: concurrent transactions do not see each other's partial work (to a degree set by isolation level). **D**urability: once committed, data survives a crash.
- SQL Server uses **Write-Ahead Logging (WAL)**: a change is described in the transaction log (`.ldf`) *before* the modified data page may be written to the data file. COMMIT returns only after the log records for the transaction are **hardened** (flushed) to disk.
- Data pages are modified in the **buffer pool** (memory). Dirty pages are written later by **checkpoint** (bounds recovery time) and **lazy writer** (frees memory under pressure). Neither is needed for durability - the log is.
- Crash recovery: **Analysis -> Redo (roll forward committed + uncommitted changes since last checkpoint) -> Undo (roll back uncommitted)**.

```
 UPDATE ──► buffer pool page (dirty) ──┐
    │                                   │ later: checkpoint / lazy writer
    └─► log buffer ──COMMIT──► .ldf (flush, WAL) ─► ack to client
                                        └──────────► .mdf (async)
```
- Every committed change gets an increasing **LSN**. Delayed durability (`DELAYED_DURABILITY`) lets COMMIT return before flush (risk of losing the last moments).

### 1.1 [Easy] What does ACID stand for and what does each letter guarantee?
**Answer**
Atomicity (all-or-nothing), Consistency (valid state to valid state), Isolation (concurrent transactions don't interfere beyond the chosen level), Durability (committed data survives failure). Every SQL Server statement runs inside a transaction, so these apply even to single statements.

**Explanation**
Atomicity is delivered by the log (undo information), consistency by constraints/triggers/your logic, isolation by locks or row versioning, durability by write-ahead logging. Consistency is the only one largely the *application's* responsibility, since the engine only enforces rules you declare.

**Code**
```sql
-- Atomicity demo: both inserts succeed or neither does
CREATE TABLE dbo.Acct(Id INT PRIMARY KEY, Bal MONEY NOT NULL CHECK (Bal >= 0));
INSERT dbo.Acct VALUES (1,100),(2,0);
BEGIN TRAN;
  UPDATE dbo.Acct SET Bal = Bal - 150 WHERE Id = 1;   -- violates CHECK
  UPDATE dbo.Acct SET Bal = Bal + 150 WHERE Id = 2;
COMMIT;  -- never reached; error 547 and (with XACT_ABORT ON) rollback
```

### 1.2 [Follow-up] What is write-ahead logging and why does COMMIT not need data pages written to disk?
**Answer**
WAL means log records are flushed to the transaction log before the corresponding data page can be written, and a commit is acknowledged only after its commit log record is hardened. Data pages can stay dirty in memory because after a crash, redo replays the log to rebuild them.

**Explanation**
Log writes are sequential and small, data page writes are random and large, so deferring page writes is a major performance win. The buffer manager enforces that a page cannot be flushed until the log up to that page's last-modification LSN is on disk.

**Code**
```sql
-- Observe log flush waits (commit latency is bound by log write speed)
SELECT wait_type, waiting_tasks_count, wait_time_ms
FROM sys.dm_os_wait_stats
WHERE wait_type IN ('WRITELOG','LOGBUFFER');
```

### 1.3 [Medium] What do checkpoint and lazy writer each do, and how do they differ?
**Answer**
Checkpoint periodically writes dirty pages to the data files to shorten crash-recovery time (governed by the `RECOVERY INTERVAL`/target recovery time, default 60 s with indirect checkpoint in newer versions). Lazy writer runs when the buffer pool needs free pages and evicts cold pages, writing them if dirty. Checkpoint does not evict pages; lazy writer is about memory pressure, not recovery time.

**Explanation**
Neither guarantees durability; the log does. A checkpoint also occurs at clean shutdown, backup start (for some), and on `CHECKPOINT`. In SIMPLE recovery, a checkpoint is also what allows log truncation. Eager writes can occur for minimally logged/bulk operations.

**Code**
```sql
SELECT name, target_recovery_time_in_seconds FROM sys.databases WHERE name = DB_NAME();
ALTER DATABASE CURRENT SET TARGET_RECOVERY_TIME = 60 SECONDS;
CHECKPOINT;
```

### 1.4 [Hard] Walk through crash recovery. What happens to a transaction committed but whose pages were never flushed, and one uncommitted whose pages *were* flushed?
**Answer**
Recovery has Analysis (find dirty pages/active transactions from the last checkpoint), Redo (reapply logged changes whose LSN is newer than the page LSN), and Undo (roll back transactions with no commit record). The committed-but-unflushed change is redone from the log; the uncommitted-but-flushed change is undone using the log's undo records.

**Explanation**
This is why WAL is required: steal (uncommitted pages may reach disk) + no-force (committed pages need not) is permitted. With **Accelerated Database Recovery (ADR, SQL 2019+)**, a persistent version store plus sLog makes undo/recovery time nearly independent of transaction size. Databases are online for new work during undo in Enterprise-style "fast recovery".

**Gotchas / follow-ups**
- Long-running open transaction => long rollback/recovery (without ADR). Rollback is single-threaded and can take as long as or longer than the original work.
- Don't kill a spid mid-rollback repeatedly; use `KILL n WITH STATUSONLY`.
- Torn pages are detected with `PAGE_VERIFY CHECKSUM`.

**Code**
```sql
SELECT name, page_verify_option_desc, is_accelerated_database_recovery_on
FROM sys.databases WHERE name = DB_NAME();
-- Progress of rollback for session 57:
-- KILL 57 WITH STATUSONLY;
```

### 1.5 [Tricky] Can a committed transaction be lost in SQL Server? When?
**Answer**
Yes: with delayed durability (`DELAYED_DURABILITY = ALLOWED/FORCED` or `COMMIT WITH (DELAYED_DURABILITY = ON)`), or when storage lies about flushes (write cache without battery/FUA), or with async AG replicas after a forced failover. Under default fully durable commit on honest storage, a committed transaction is not lost.

**Explanation**
Delayed durability buffers log blocks and flushes asynchronously (when buffer fills ~60 KB or on `sp_flush_log`). It reduces WRITELOG waits but trades the last few commits on a crash. Synchronous-commit AG replicas harden log on the secondary before acking; async does not.

**Gotchas / follow-ups**
- Delayed durability is fine for ETL/staging, not for payments.
- Memory-optimized tables with `SCHEMA_ONLY` lose all data on restart by design.

**Code**
```sql
ALTER DATABASE CURRENT SET DELAYED_DURABILITY = ALLOWED;
BEGIN TRAN; INSERT dbo.Acct VALUES (9,1);
COMMIT TRAN WITH (DELAYED_DURABILITY = ON);
EXEC sys.sp_flush_log;  -- force hardening
```

**Interview traps**
- "Commit writes data to the .mdf" - wrong; it hardens the log.
- Confusing checkpoint with lazy writer.
- Thinking Consistency is purely engine-enforced.
- Forgetting delayed durability / async replicas weaken D.

**Cheat sheet**
- WAL: log first, data later; commit = log hardened.
- Checkpoint: bounds recovery time; Lazy writer: frees memory.
- Recovery = Analysis, Redo, Undo.
- Delayed durability = faster commits, possible loss.
- ADR makes recovery/rollback fast.

---

## Topic 2: Transactions

**Study notes**

- **Autocommit** (default): each statement is its own transaction.
- **Explicit**: `BEGIN TRAN ... COMMIT/ROLLBACK`. `@@TRANCOUNT` tracks nesting depth; only the **outermost COMMIT** commits, but **any ROLLBACK** (without savepoint) rolls back everything and sets `@@TRANCOUNT = 0`.
- **Implicit**: `SET IMPLICIT_TRANSACTIONS ON` - a transaction starts automatically on certain statements and must be committed manually (ANSI-style; used by some drivers).
- **Savepoints**: `SAVE TRAN name` then `ROLLBACK TRAN name` undoes work back to that point without ending the transaction. Names in `ROLLBACK` only work for savepoints (or the outermost named tran).
- **XACT_ABORT ON**: any run-time error aborts the batch and rolls back the transaction. Strongly recommended in procs that use transactions. Without it, many errors (e.g., constraint violation) only fail the statement and the transaction continues.
- Use `TRY...CATCH` with `XACT_STATE()` (1 = committable, -1 = doomed/uncommittable, 0 = none) and `THROW`.

```
BEGIN TRAN (1)  BEGIN TRAN (2)  COMMIT (->1)  COMMIT (->0 real commit)
                                 ROLLBACK anywhere => 0, everything undone
```

### 2.1 [Easy] What are autocommit and explicit transactions?
**Answer**
In autocommit each statement commits on its own. An explicit transaction groups statements with `BEGIN TRAN` and ends with `COMMIT` or `ROLLBACK`, making them atomic as a unit.

**Explanation**
Even a single `UPDATE` touching many rows is atomic in autocommit. Use explicit transactions when several statements must succeed together. Keep them short to reduce lock time and log retention.

**Code**
```sql
BEGIN TRAN;
  UPDATE dbo.Acct SET Bal -= 10 WHERE Id = 1;
  UPDATE dbo.Acct SET Bal += 10 WHERE Id = 2;
COMMIT;
```

### 2.2 [Follow-up] What does `@@TRANCOUNT` do with nested BEGIN TRAN, and what does ROLLBACK do?
**Answer**
Each `BEGIN TRAN` increments `@@TRANCOUNT`, each `COMMIT` decrements it, and only the commit that brings it to zero makes changes permanent. A `ROLLBACK` rolls back the entire transaction regardless of nesting and sets the count to 0.

**Explanation**
There are no true nested transactions. Inner stored procedures that unconditionally `ROLLBACK` cause error 266 ("Transaction count after EXECUTE indicates a mismatching number of BEGIN and COMMIT statements") for the caller. Pattern: only begin a transaction if `@@TRANCOUNT = 0`, or use a savepoint in the callee.

**Code**
```sql
BEGIN TRAN; BEGIN TRAN;
SELECT @@TRANCOUNT AS afterTwoBegins;  -- 2
COMMIT;  SELECT @@TRANCOUNT;           -- 1 (nothing committed yet)
ROLLBACK; SELECT @@TRANCOUNT;          -- 0 (all undone)
```

### 2.3 [Medium] What does SET XACT_ABORT ON change and why is it recommended?
**Answer**
With `XACT_ABORT ON`, any run-time error terminates the batch and rolls back the open transaction. Without it, many errors only abort the statement and leave the transaction open, so subsequent statements can commit a half-done unit of work. It also ensures a client timeout/attention rolls back instead of leaving an orphan open transaction.

**Explanation**
Compile errors and some errors (e.g., raised by `RAISERROR` with low severity) behave differently. XACT_ABORT is required for transactions involving linked servers/MSDTC distributed work. EF Core and `TransactionScope` rely on connection-level behaviours, but in your own procs set it explicitly.

**Code**
```sql
CREATE OR ALTER PROC dbo.Transfer @From INT, @To INT, @Amt MONEY
AS
BEGIN
  SET NOCOUNT, XACT_ABORT ON;
  BEGIN TRY
    BEGIN TRAN;
      UPDATE dbo.Acct SET Bal -= @Amt WHERE Id = @From;
      UPDATE dbo.Acct SET Bal += @Amt WHERE Id = @To;
    COMMIT;
  END TRY
  BEGIN CATCH
    IF XACT_STATE() <> 0 ROLLBACK;
    THROW;
  END CATCH
END
```

### 2.4 [Medium] How do savepoints work and when would you use them?
**Answer**
`SAVE TRAN sp1` marks a point; `ROLLBACK TRAN sp1` undoes work since it but keeps the transaction (and earlier work) open. Useful for optional/best-effort sub-steps inside a larger unit of work, and for reusable procs that join an outer transaction.

**Explanation**
Locks acquired after the savepoint are *not* necessarily released on partial rollback (they're held until transaction end for most isolation levels - in practice, assume they remain). Savepoints are not allowed in distributed transactions. With `XACT_ABORT ON`, a run-time error still kills the entire transaction, so savepoint recovery needs `XACT_ABORT OFF` or errors that you CATCH while `XACT_STATE() = 1`.

**Code**
```sql
CREATE OR ALTER PROC dbo.InnerWork AS
BEGIN
  DECLARE @own BIT = IIF(@@TRANCOUNT = 0,1,0);
  IF @own = 1 BEGIN TRAN ELSE SAVE TRAN InnerWork;
  BEGIN TRY
    INSERT dbo.Acct VALUES (100,5);
    IF @own = 1 COMMIT;
  END TRY
  BEGIN CATCH
    IF XACT_STATE() = 1 AND @own = 0 ROLLBACK TRAN InnerWork;
    ELSE IF XACT_STATE() <> 0 ROLLBACK;
    THROW;
  END CATCH
END
```

### 2.5 [Tricky] In TRY/CATCH, why check XACT_STATE() before COMMIT/ROLLBACK, and what is an uncommittable transaction?
**Answer**
If a run-time error occurs while `XACT_ABORT` is ON (or certain severe errors), the transaction is "doomed": `XACT_STATE() = -1`. You can only roll it back; `COMMIT` raises error 3930. `XACT_STATE() = 0` means there is nothing to roll back, so an unconditional `ROLLBACK` would itself error (3903).

**Explanation**
Therefore the CATCH template is `IF XACT_STATE() <> 0 ROLLBACK;` then re-raise with `THROW`. `@@TRANCOUNT > 0` is an alternative guard but cannot detect the doomed state. Also, `THROW` (2012+) preserves the original error number; `RAISERROR` does not.

**Gotchas / follow-ups**
- Prior statement before `THROW` must end with `;`.
- Error 266 / 3903 / 3930 are classic proc-nesting bugs.
- Deadlock victim error 1205 rolls back the transaction automatically; the CATCH will see `XACT_STATE() = 0`.
- In C#, a `TransactionScope` not completed rolls back at dispose.

**Code**
```sql
SET XACT_ABORT ON;
BEGIN TRY
  BEGIN TRAN;
  INSERT dbo.Acct VALUES (1, 1);   -- PK violation
  COMMIT;
END TRY
BEGIN CATCH
  SELECT XACT_STATE() AS st, @@TRANCOUNT AS tc, ERROR_NUMBER() AS err;
  IF XACT_STATE() <> 0 ROLLBACK;
END CATCH
```

**Interview traps**
- "ROLLBACK only rolls back the inner transaction" - false.
- Assuming an error inside a transaction auto-rolls back (not without XACT_ABORT).
- Long transactions that include user interaction or HTTP calls.
- `ROLLBACK` with no open transaction errors.

**Cheat sheet**
- Only outermost COMMIT commits; any ROLLBACK undoes everything.
- Always `SET XACT_ABORT ON` in transactional procs.
- CATCH: `IF XACT_STATE() <> 0 ROLLBACK; THROW;`
- Savepoints = partial rollback; not with distributed tran.
- `@@TRANCOUNT` mismatch => error 266.

---

## Topic 3: Isolation levels & anomalies

**Study notes**

Anomalies:
- **Dirty read**: reading uncommitted data.
- **Non-repeatable read**: same row read twice, value/existence changed by another committed tx.
- **Phantom**: same range query returns different *set of rows*.
- **Lost update**: two transactions read then write based on stale value; one overwrites the other.

| Level | Dirty | Non-repeatable | Phantom | Mechanism |
|---|---|---|---|---|
| READ UNCOMMITTED | yes | yes | yes | no shared locks |
| READ COMMITTED (default, locking) | no | yes | yes | S lock released after read |
| READ COMMITTED SNAPSHOT (RCSI) | no | yes | yes | statement-level row versions |
| REPEATABLE READ | no | no | yes | S locks held to end of tx |
| SERIALIZABLE | no | no | no | range (key-range) locks |
| SNAPSHOT | no | no | no* | transaction-level row versions |

*Snapshot avoids phantoms for reads but not write skew. Set with `SET TRANSACTION ISOLATION LEVEL ...`; Snapshot also needs `ALLOW_SNAPSHOT_ISOLATION ON`. Default in Azure SQL Database: RCSI ON; on-prem SQL Server: locking READ COMMITTED.

### 3.1 [Easy] Name the four anomalies and the default isolation level.
**Answer**
Dirty read, non-repeatable read, phantom read, and lost update. SQL Server's default is READ COMMITTED (locking on-prem; RCSI by default on Azure SQL Database).

**Explanation**
READ COMMITTED prevents dirty reads only. A higher level costs concurrency (locking levels) or tempdb/version store space (snapshot levels). Lost update is not on the ANSI list but matters in practice.

**Code**
```sql
SELECT name, is_read_committed_snapshot_on, snapshot_isolation_state_desc
FROM sys.databases WHERE name = DB_NAME();
DBCC USEROPTIONS;  -- shows isolation level of this session
```

### 3.2 [Follow-up] Demonstrate a non-repeatable read and show how REPEATABLE READ prevents it.
**Answer**
Under READ COMMITTED, session A reads a row, session B updates and commits, and A's second read sees the new value. Under REPEATABLE READ, A holds its S lock until commit, so B's update blocks until A finishes.

**Explanation**
REPEATABLE READ protects rows already read, not rows that don't yet exist, so inserts (phantoms) still slip in.

**Code**
```sql
-- Session A
SET TRANSACTION ISOLATION LEVEL READ COMMITTED;   -- try REPEATABLE READ next
BEGIN TRAN;
SELECT Bal FROM dbo.Acct WHERE Id = 1;            -- 100
-- >>> run Session B now
SELECT Bal FROM dbo.Acct WHERE Id = 1;            -- changed (RC) / B blocked (RR)
COMMIT;

-- Session B
UPDATE dbo.Acct SET Bal = 999 WHERE Id = 1;       -- waits under REPEATABLE READ
```

### 3.3 [Medium] How does SERIALIZABLE prevent phantoms, and what is the cost?
**Answer**
It takes key-range locks (RangeS-S etc.) on the index range scanned, so no one can insert, delete or update into that range until commit. The cost is heavy blocking and a higher deadlock rate, and it needs a supporting index - otherwise it locks much more (whole table).

**Explanation**
If no index exists on the predicate column, the engine scans and locks the entire table. For "insert if not exists" correctness, SERIALIZABLE (or `UPDLOCK, HOLDLOCK`, which is the SERIALIZABLE hint) is the standard fix.

**Code**
```sql
-- Session A
SET TRANSACTION ISOLATION LEVEL SERIALIZABLE;
BEGIN TRAN;
SELECT * FROM dbo.Acct WHERE Id BETWEEN 10 AND 20;
-- Session B: INSERT dbo.Acct VALUES (15, 1);  -- blocks
SELECT resource_type, request_mode, request_status
FROM sys.dm_tran_locks WHERE request_session_id = @@SPID;   -- KEY RangeS-S
COMMIT;
```

### 3.4 [Medium] What is a lost update and how do you prevent it?
**Answer**
Two sessions read the same value, compute from it, and write back, so the second write overwrites the first. Prevent it with atomic updates (`SET Bal = Bal - @x`), `UPDLOCK` on the read, optimistic concurrency via `rowversion`, or REPEATABLE READ/SERIALIZABLE (which turns it into a deadlock) or SNAPSHOT (which raises update conflict 3960).

**Explanation**
At plain READ COMMITTED, the S lock from the SELECT is released immediately, so nothing prevents the overwrite. Read-modify-write in app code (EF without concurrency tokens) is the usual source.

**Code**
```sql
-- Safe: single atomic statement
UPDATE dbo.Acct SET Bal = Bal - 10 WHERE Id = 1 AND Bal >= 10;
IF @@ROWCOUNT = 0 PRINT 'insufficient funds';

-- Pessimistic: take update lock while reading
BEGIN TRAN;
DECLARE @b MONEY = (SELECT Bal FROM dbo.Acct WITH (UPDLOCK, ROWLOCK) WHERE Id = 1);
UPDATE dbo.Acct SET Bal = @b - 10 WHERE Id = 1;
COMMIT;
```

### 3.5 [Hard] Is READ COMMITTED guaranteed to return a consistent snapshot of a table scan? What anomalies exist even without "dirty reads"?
**Answer**
No. Under locking READ COMMITTED a scan can read a row twice or miss a row if a concurrent update moves it (e.g., key change on the scanned index, page splits) because locks are released as the scan proceeds. It prevents dirty reads, not inconsistent statement results.

**Explanation**
Allocation-order scans under `NOLOCK` are even worse (duplicates/misses from page splits). RCSI gives each statement a consistent point-in-time view, eliminating these. Aggregates across tables (e.g., sum of two tables in two statements) still see different times under RCSI; only SNAPSHOT/SERIALIZABLE gives transaction-wide consistency.

**Gotchas / follow-ups**
- Moving a row ahead of the scan = read twice; behind = missed.
- Isolation level is per session; `SET TRANSACTION ISOLATION LEVEL` persists until changed (pooled connections reset it via `sp_reset_connection`).
- You cannot change the level from RC to/from SNAPSHOT mid-transaction in certain ways (error 3951/3952 for snapshot transitions).

**Code**
```sql
-- Reader (RC) scanning a big table while another session
-- UPDATEs the clustered key of rows ahead of the scan can produce duplicates.
SET TRANSACTION ISOLATION LEVEL READ COMMITTED;
SELECT COUNT(*), COUNT(DISTINCT Id) FROM dbo.BigTable;
```

### 3.6 [Tricky] What is write skew, and which isolation level allows it?
**Answer**
Write skew: two transactions read overlapping data, each updates a *different* row based on what it read, and together they violate an invariant (e.g., both doctors go off-call). SNAPSHOT allows it because there is no write-write conflict; SERIALIZABLE prevents it.

**Explanation**
SNAPSHOT only detects conflicts when the same row is updated. Fix with SERIALIZABLE, a locking read (`UPDLOCK, HOLDLOCK`) on the checked set, or by materializing a conflict on a shared row.

**Gotchas / follow-ups**
- SNAPSHOT is "serializable-ish" for reads but not truly serializable.
- Mentioning this separates strong candidates.

**Code**
```sql
-- Both sessions run this under SNAPSHOT; both can commit => zero doctors on call
SET TRANSACTION ISOLATION LEVEL SNAPSHOT;
BEGIN TRAN;
IF (SELECT COUNT(*) FROM dbo.Doctor WHERE OnCall = 1) >= 2
  UPDATE dbo.Doctor SET OnCall = 0 WHERE Id = @MyId;
COMMIT;
-- Fix: read with (UPDLOCK, HOLDLOCK) under READ COMMITTED, or use SERIALIZABLE
```

**Interview traps**
- Saying READ COMMITTED prevents all non-repeatable reads.
- Thinking SNAPSHOT == SERIALIZABLE.
- Forgetting a supporting index makes SERIALIZABLE lock the whole table.
- Calling NOLOCK a "faster isolation level".

**Cheat sheet**
- RU: nothing; RC: no dirty; RR: + no non-repeatable; SER: + no phantom.
- Snapshot = tx-level versions; RCSI = statement-level versions.
- Lost update fix: atomic UPDATE / UPDLOCK / rowversion.
- SERIALIZABLE = range locks, needs index.
- Write skew survives SNAPSHOT.

---

## Topic 4: RCSI vs SNAPSHOT & row versioning

**Study notes**

- **Row versioning**: when a row is modified, the old version is copied to the **version store** (in **tempdb**, or in the user database's PVS when ADR is on) and a 14-byte pointer/timestamp is added to the row. Readers read the version valid at their snapshot point instead of taking shared locks.
- **RCSI** (`READ_COMMITTED_SNAPSHOT ON`): changes the behaviour of the default READ COMMITTED - each *statement* sees data committed as of statement start. No code change; readers don't block writers and vice versa. Writers still block writers.
- **SNAPSHOT** (`ALLOW_SNAPSHOT_ISOLATION ON` + `SET TRANSACTION ISOLATION LEVEL SNAPSHOT`): each *transaction* sees data as of its first data access. Update conflicts raise **error 3960** and the transaction must be retried.
- Cost: tempdb version store space/IO, 14 bytes per modified row, a long-running transaction prevents version cleanup.

```
Reader (RCSI)  ──reads old version from version store──► no S lock, no block
Writer         ──X lock on row──► writes new version in place, old copy -> tempdb
```

### 4.1 [Easy] What problem does RCSI solve?
**Answer**
It stops readers from blocking writers and writers from blocking readers under the default READ COMMITTED level, without code changes. Readers get the last committed version of a row rather than waiting for locks.

**Explanation**
Switching needs exclusive access momentarily (other connections must be disconnected, or use `WITH ROLLBACK IMMEDIATE`). Azure SQL Database has it ON by default.

**Code**
```sql
ALTER DATABASE MyDb SET READ_COMMITTED_SNAPSHOT ON WITH ROLLBACK IMMEDIATE;
SELECT is_read_committed_snapshot_on FROM sys.databases WHERE name = 'MyDb';
```

### 4.2 [Follow-up] How do RCSI and SNAPSHOT differ?
**Answer**
RCSI gives statement-level consistency and replaces locking READ COMMITTED with no code changes; SNAPSHOT gives transaction-level consistency, needs explicit opt-in per session, and detects write-write update conflicts (error 3960). SNAPSHOT gives repeatable reads and no phantoms for the whole transaction; RCSI does not.

**Explanation**
Under RCSI a writer that hits a row being changed by another uncommitted writer just waits (blocks), then proceeds against the latest committed row (it re-evaluates the predicate), so no 3960. Under SNAPSHOT if the row changed after your snapshot, you get 3960 and must retry.

**Code**
```sql
ALTER DATABASE MyDb SET ALLOW_SNAPSHOT_ISOLATION ON;
-- Session A
SET TRANSACTION ISOLATION LEVEL SNAPSHOT;
BEGIN TRAN;
SELECT Bal FROM dbo.Acct WHERE Id = 1;        -- takes snapshot
-- Session B: UPDATE dbo.Acct SET Bal = 5 WHERE Id = 1;  (commits)
UPDATE dbo.Acct SET Bal = 7 WHERE Id = 1;     -- Msg 3960 update conflict
```

### 4.3 [Medium] Where is the version store, how do you monitor it, and what makes it grow?
**Answer**
By default in tempdb (with ADR, in the user DB's persistent version store). It grows when there are long-running or forgotten open transactions (or long snapshot readers), because versions can't be cleaned until no active transaction needs them. Monitor `sys.dm_tran_version_store_space_usage`, `sys.dm_tran_active_snapshot_database_transactions`, and the `Version Store Size` counter.

**Explanation**
Cleanup runs about every minute. Fix by finding and shortening the oldest transaction. Pre-size tempdb with multiple equal data files.

**Code**
```sql
SELECT DB_NAME(database_id) db, reserved_page_count*8/1024 AS reserved_mb
FROM sys.dm_tran_version_store_space_usage;

SELECT TOP 5 session_id, elapsed_time_seconds, is_snapshot
FROM sys.dm_tran_active_snapshot_database_transactions
ORDER BY elapsed_time_seconds DESC;
```

### 4.4 [Hard] What behavioural changes can enabling RCSI cause in existing code?
**Answer**
Code that *relied* on readers blocking (e.g., "read then update" patterns assuming a shared lock, queue/claim logic, or triggers/validations that read data expecting another session's uncommitted change to block) may now read stale committed data and break. Also tempdb load rises and each modified row grows by 14 bytes.

**Explanation**
Typical fix: add explicit `UPDLOCK`/`READCOMMITTEDLOCK` hints where blocking semantics are required. `READCOMMITTEDLOCK` forces locking behaviour for that table under RCSI. Foreign-key and constraint checks internally use locking reads, so referential integrity is preserved.

**Gotchas / follow-ups**
- Write skew-style logic ("check count then insert") is unsafe unless the check uses `UPDLOCK, HOLDLOCK`.
- Lost updates become possible for code that used to be saved by blocking.
- Page splits/fragmentation can rise from version pointers.

**Code**
```sql
SELECT Bal FROM dbo.Acct WITH (READCOMMITTEDLOCK) WHERE Id = 1;  -- behaves like locking RC
SELECT Bal FROM dbo.Acct WITH (UPDLOCK) WHERE Id = 1;            -- always reads latest, locks
```

### 4.5 [Tricky] Under SNAPSHOT, can a transaction read a row, see no change, yet fail on a later UPDATE? And can DDL interfere?
**Answer**
Yes: the read comes from the snapshot, but the UPDATE targets the current row; if another transaction committed a change after your snapshot began, you get 3960. DDL (schema modification) against objects you touched after your snapshot started can also cause error 3961 (snapshot failed because object was modified by another DDL).

**Explanation**
Always wrap SNAPSHOT updates in retry logic. Metadata is not versioned, so DDL (`ALTER TABLE`, rebuilds in some forms) conflicts. Also, a snapshot transaction's snapshot starts at the first data access, not at `BEGIN TRAN`.

**Gotchas / follow-ups**
- In EF Core, catch `SqlException.Number == 3960`.
- Cross-database queries in SNAPSHOT need the option enabled on every DB involved.

**Code**
```sql
SET TRANSACTION ISOLATION LEVEL SNAPSHOT;
BEGIN TRAN;                      -- snapshot NOT started yet
SELECT 1;                        -- still not started (no data access)
SELECT TOP 1 * FROM dbo.Acct;    -- snapshot starts here
COMMIT;
```

**Interview traps**
- "RCSI makes everything snapshot" - it's statement-level only.
- Believing RCSI removes writer-writer blocking.
- Ignoring tempdb sizing / long open transactions.
- Forgetting SNAPSHOT needs a separate DB option plus session setting.

**Cheat sheet**
- RCSI: DB option, no code change, per-statement view.
- SNAPSHOT: opt-in per tx, per-tx view, error 3960 on conflict.
- Version store in tempdb (or PVS with ADR).
- Long open tx => version store bloat.
- Use `READCOMMITTEDLOCK`/`UPDLOCK` to restore locking semantics.

---

## Topic 5: Locking

**Study notes**

- **Lock modes**: **S** (shared, reads), **X** (exclusive, writes), **U** (update, read-with-intent-to-update; prevents the S+S->X conversion deadlock), **IS/IX/IU** (intent), **SIX**, **Sch-S/Sch-M** (schema stability/modification), **BU** (bulk update), key-range modes (**RangeS-S**, **RangeX-X** ...).
- **Granularity**: ROW (RID/KEY), PAGE, TABLE (OBJECT), also HoBT, DATABASE. Intent locks at higher levels advertise lower-level locks so the engine need not scan them.
- **Escalation**: when one statement holds ~**5,000 locks** on a single table/index (or memory pressure), the engine tries to convert to a table lock (partition-level if `LOCK_ESCALATION = AUTO` on partitioned tables). Blocked escalation is retried every ~1,250 new locks.
- Compatibility (most-used):

| Held \ Requested | IS | IX | S | U | X |
|---|---|---|---|---|---|
| IS | Y | Y | Y | Y | N |
| IX | Y | Y | N | N | N |
| S | Y | N | Y | Y | N |
| U | Y | N | Y | N | N |
| X | N | N | N | N | N |

### 5.1 [Easy] What are shared and exclusive locks?
**Answer**
Shared (S) locks are taken to read and are compatible with other S locks. Exclusive (X) locks are taken to modify data, are held until the transaction ends, and are incompatible with every other lock.

**Explanation**
Under default READ COMMITTED (locking), S locks are released right after the row/page is read; under REPEATABLE READ and above they persist until commit. X locks are always held to transaction end.

**Code**
```sql
BEGIN TRAN;
UPDATE dbo.Acct SET Bal = Bal WHERE Id = 1;
SELECT resource_type, resource_description, request_mode, request_status
FROM sys.dm_tran_locks WHERE request_session_id = @@SPID;   -- KEY X, PAGE IX, OBJECT IX
ROLLBACK;
```

### 5.2 [Follow-up] Why do intent locks exist?
**Answer**
They signal at table/page level that a transaction holds or intends to hold finer-grained locks below. So a request for a table-level lock can be checked against one intent lock instead of scanning every row lock.

**Explanation**
An `UPDATE` of one row takes IX on the table and page, then X on the key. A table-level S request conflicts with IX immediately. IU/SIX are hybrids (e.g., SIX = S on table + IX for some rows).

**Code**
```sql
-- Session A
BEGIN TRAN; UPDATE dbo.Acct SET Bal = Bal WHERE Id = 1;
-- Session B: needs table S, conflicts with A's IX -> blocks
SELECT * FROM dbo.Acct WITH (TABLOCK);
```

### 5.3 [Medium] What is an update (U) lock and why does it exist?
**Answer**
U is taken while locating rows to modify; it's compatible with S but not with another U. It prevents the classic conversion deadlock where two transactions both hold S on a row and both try to upgrade to X.

**Explanation**
The U lock converts to X when the row is actually changed. Using `UPDLOCK` hint on a SELECT gets this protection for read-then-update logic.

**Code**
```sql
-- Both sessions (run step-by-step); with only S locks (REPEATABLE READ) this deadlocks, with UPDLOCK it serializes
SET TRANSACTION ISOLATION LEVEL REPEATABLE READ;
BEGIN TRAN;
SELECT Bal FROM dbo.Acct WITH (UPDLOCK) WHERE Id = 1;
WAITFOR DELAY '00:00:10';
UPDATE dbo.Acct SET Bal = Bal + 1 WHERE Id = 1;
COMMIT;
```

### 5.4 [Medium] Explain lock escalation: when does it happen and how do you control it?
**Answer**
When a single statement acquires about 5,000 locks on one object (or lock memory is large), SQL Server tries to replace row/page locks with a table lock (or partition lock). If another session holds incompatible locks it fails and retries later. Control by batching, adding indexes so fewer rows are touched, `ALTER TABLE ... SET (LOCK_ESCALATION = DISABLE | TABLE | AUTO)`, or trace flags 1211/1224 (rarely).

**Explanation**
Escalation reduces memory but can cause table-wide blocking (e.g., a big `UPDATE` or `DELETE` blocking everyone). It never escalates row -> page; it goes to table (or partition). Escalated lock is X (or S) table lock held to tx end. Batching in chunks < 5,000 rows (e.g., 4,000) avoids it.

**Code**
```sql
ALTER TABLE dbo.Orders SET (LOCK_ESCALATION = AUTO);   -- partition-level for partitioned tables
SELECT name, lock_escalation_desc FROM sys.tables WHERE name = 'Orders';

-- Observe escalations
SELECT index_id, index_lock_promotion_attempt_count, index_lock_promotion_count
FROM sys.dm_db_index_operational_stats(DB_ID(), OBJECT_ID('dbo.Orders'), NULL, NULL);
```

### 5.5 [Hard] Why can a SELECT acquire many more locks than expected, and why can an UPDATE with no supporting index lock the whole table?
**Answer**
Without a usable index, the engine scans, and under locking isolation it must lock (or at least examine under lock) every row it visits; for UPDATE it takes U locks on visited rows, and a scan can lead to escalation to a table lock. A proper index reduces both locks and blocking.

**Explanation**
Under READ COMMITTED, U locks on non-qualifying rows are released as the scan moves on, but under higher isolation levels (REPEATABLE READ/SERIALIZABLE) they're held. Also, foreign keys without indexes on the child column cause scans of the child table when the parent is deleted/updated, escalating locks.

**Gotchas / follow-ups**
- Unindexed FK columns are a classic blocking/deadlock source.
- Lock hierarchy: check `sys.dm_tran_locks` for `OBJECT` with `X` after escalation.
- Partitioned tables: `LOCK_ESCALATION = AUTO` reduces cross-partition blocking.

**Code**
```sql
-- Find unindexed FK columns (simple check)
SELECT fk.name, OBJECT_NAME(fk.parent_object_id) AS child_tbl
FROM sys.foreign_keys fk
JOIN sys.foreign_key_columns fkc ON fkc.constraint_object_id = fk.object_id
WHERE NOT EXISTS (
  SELECT 1 FROM sys.index_columns ic
  WHERE ic.object_id = fkc.parent_object_id AND ic.column_id = fkc.parent_column_id AND ic.key_ordinal = 1);
```

### 5.6 [Tricky] Schema locks: can a plain SELECT block an ALTER TABLE, and can a blocked ALTER block everyone else?
**Answer**
Yes to both. Queries hold Sch-S (schema stability) while running, which conflicts with Sch-M needed by `ALTER TABLE`/index rebuild. Once the ALTER is queued waiting for Sch-M, new queries requesting Sch-S queue *behind* it, so a blocked DDL can cause a pile-up even though it hasn't done anything yet.

**Explanation**
This is why online schema changes in production use `WAIT_AT_LOW_PRIORITY` (for online index ops / partition switch) or `SET LOCK_TIMEOUT` so DDL gives up quickly. Lock queues are FIFO-ish: compatible new requests can't jump ahead of a waiting incompatible request (except lock conversions).

**Gotchas / follow-ups**
- Long open transaction that merely *selected* from the table is enough to block DDL.
- Use `SET LOCK_TIMEOUT 5000;` before DDL and retry.

**Code**
```sql
-- Session A
BEGIN TRAN; SELECT TOP 1 * FROM dbo.Acct WITH (HOLDLOCK);  -- holds Sch-S/S
-- Session B
SET LOCK_TIMEOUT 3000;
ALTER TABLE dbo.Acct ADD Note NVARCHAR(50) NULL;           -- waits, then error 1222
-- Online index rebuild with low priority
ALTER INDEX ALL ON dbo.Acct REBUILD WITH (ONLINE = ON (WAIT_AT_LOW_PRIORITY (MAX_DURATION = 1 MINUTES, ABORT_AFTER_WAIT = SELF)));
```

**Interview traps**
- "Locks escalate row -> page -> table" - there is no page escalation step.
- Thinking SELECT never blocks anything (schema locks, SERIALIZABLE, etc.).
- Missing indexes on FK columns.
- Believing escalation threshold is a table-size percentage (it's a lock count ~5,000 per statement per object).

**Cheat sheet**
- S compatible with S; X with nothing; U compatible with S not U.
- Intent locks (IS/IX) let table locks be checked cheaply.
- Escalation ~5,000 locks -> table/partition lock.
- `LOCK_ESCALATION` table option; batch under 5k rows.
- DDL needs Sch-M; use LOCK_TIMEOUT / WAIT_AT_LOW_PRIORITY.

---

## Topic 6: Blocking diagnosis

**Study notes**

- **Blocking** = one session waits for a lock held by another (normal, short-lived). A **head blocker** is the root of a chain; often an idle session with an open transaction (`sleeping`, `open_transaction_count > 0`).
- Tools: `sp_who2` / `sp_who` (BlkBy column), `sys.dm_exec_requests.blocking_session_id`, `sys.dm_os_waiting_tasks`, `sys.dm_tran_locks`, `sys.dm_exec_sql_text`, the **blocked process report** (`blocked process threshold` + XE/trace), Extended Events, Query Store wait stats, `sp_whoisactive` (community), Activity Monitor.
- Wait types: `LCK_M_S`, `LCK_M_X`, `LCK_M_U`, `LCK_M_IX`, `LCK_M_SCH_M`, `LCK_M_RS_S` etc. (LCK_M_* = lock waits).

### 6.1 [Easy] How do you find who is blocking whom?
**Answer**
Run `sp_who2` and look at the `BlkBy` column, or query `sys.dm_exec_requests` where `blocking_session_id <> 0`. Follow the chain to the head blocker (a session that blocks others but isn't blocked).

**Explanation**
`sp_who2` is quick but limited; DMVs also give wait type, wait time, and SQL text.

**Code**
```sql
EXEC sp_who2;

SELECT r.session_id, r.blocking_session_id, r.wait_type, r.wait_time, r.status,
       DB_NAME(r.database_id) db, t.text
FROM sys.dm_exec_requests r
CROSS APPLY sys.dm_exec_sql_text(r.sql_handle) t
WHERE r.blocking_session_id <> 0;
```

### 6.2 [Follow-up] The head blocker shows status "sleeping" and no running query. What does that mean?
**Answer**
The session finished its last batch but left a transaction open (e.g., app forgot to commit, or user/UX paused mid-transaction, or implicit transactions). Its locks are held until commit/rollback or connection close.

**Explanation**
Find it via `sys.dm_tran_session_transactions`/`sys.dm_exec_sessions.open_transaction_count`. Use `sys.dm_exec_connections.most_recent_sql_handle` to see the last statement. Fix root cause in the app (scope transactions tightly, `using`/`TransactionScope`, set command timeouts, `XACT_ABORT ON`); `KILL` is a stopgap.

**Code**
```sql
SELECT s.session_id, s.login_name, s.host_name, s.program_name, s.status,
       s.open_transaction_count, s.last_request_end_time, t.text AS last_sql
FROM sys.dm_exec_sessions s
JOIN sys.dm_exec_connections c ON c.session_id = s.session_id
CROSS APPLY sys.dm_exec_sql_text(c.most_recent_sql_handle) t
WHERE s.open_transaction_count > 0 AND s.status = 'sleeping';
```

### 6.3 [Medium] Write a query that shows the full blocking chain with locks being waited for.
**Answer**
Join `sys.dm_os_waiting_tasks` (waiting and blocking session, resource description) with `sys.dm_exec_requests` and `sys.dm_exec_sql_text`. Use a recursive CTE to find the lead blocker.

**Explanation**
`resource_description` in `dm_os_waiting_tasks` tells you the exact lock resource (key/page/object). `sys.dm_tran_locks` with `request_status = 'WAIT'` vs `'GRANT'` provides the holder/waiter picture. Resolve `%%lockres%%` or object via `sys.partitions` for HoBT IDs.

**Code**
```sql
SELECT w.session_id AS waiter, w.blocking_session_id AS blocker,
       w.wait_duration_ms, w.wait_type, w.resource_description
FROM sys.dm_os_waiting_tasks w
WHERE w.blocking_session_id IS NOT NULL
ORDER BY w.wait_duration_ms DESC;

SELECT l.request_session_id, l.resource_type, l.request_mode, l.request_status,
       OBJECT_NAME(p.object_id) AS obj
FROM sys.dm_tran_locks l
LEFT JOIN sys.partitions p ON p.hobt_id = l.resource_associated_entity_id
WHERE l.resource_database_id = DB_ID() AND l.resource_type IN ('KEY','PAGE','RID','OBJECT');
```

### 6.4 [Hard] How do you capture blocking that happened earlier (historic), not just live?
**Answer**
Enable the **blocked process report**: set `sp_configure 'blocked process threshold (s)'` (e.g., 10) and capture the `blocked_process_report` event with Extended Events (or Profiler/trace). Query Store (wait stats), monitoring tools, or an Agent job logging DMV snapshots also help.

**Explanation**
The report is XML listing blocked and blocking processes, input buffers, and lock resources. The threshold generates a report when a wait exceeds N seconds (don't set it very low, e.g. below 5). In Azure SQL Database the threshold is a fixed setting you cannot change via `sp_configure`; use XE/Query Store wait stats there.

**Gotchas / follow-ups**
- Threshold of 0 disables it.
- The event only fires once per interval per blocked task.
- XE `blocked_process_report` goes to a ring_buffer or file target.

**Code**
```sql
EXEC sp_configure 'show advanced options', 1; RECONFIGURE;
EXEC sp_configure 'blocked process threshold (s)', 10; RECONFIGURE;

CREATE EVENT SESSION BlockedReport ON SERVER
ADD EVENT sqlserver.blocked_process_report
ADD TARGET package0.event_file (SET filename = N'BlockedReport.xel', max_file_size = 50)
WITH (STARTUP_STATE = ON);
ALTER EVENT SESSION BlockedReport ON SERVER STATE = START;
```

### 6.5 [Tricky] A query is blocked by `blocking_session_id = -2, -3, -4, -5`. What do these mean, and what is NOT blocking?
**Answer**
-2: an orphaned distributed transaction; -3: deferred recovery transaction; -4: latch owner can't be determined; -5 (2019+): latch held by a session that can't be identified (internal). Also, long waits may be `PAGEIOLATCH`, `CXPACKET`/`CXCONSUMER`, `ASYNC_NETWORK_IO`, which are not blocking in the lock sense.

**Explanation**
-2 requires resolving the MSDTC transaction (kill the UOW GUID from `sys.dm_tran_locks.request_owner_guid`). `ASYNC_NETWORK_IO` means the client is slow consuming results (often a C# app row-by-row). Don't blame blocking before checking wait type.

**Gotchas / follow-ups**
- `KILL '<UOW-guid>'` for orphaned DTC transactions.
- Blocking != deadlock: deadlocks are auto-resolved, blocking is not.

**Code**
```sql
SELECT DISTINCT request_owner_guid FROM sys.dm_tran_locks
WHERE request_owner_type = 'TRANSACTION' AND request_owner_guid <> '00000000-0000-0000-0000-000000000000';
-- KILL 'D5499C66-E398-45CA-BF7E-DC9C194B48CF';   -- UOW from above
```

**Interview traps**
- Reaching for `KILL` without finding why the transaction stayed open.
- Treating `NOLOCK` as the fix for blocking.
- Not distinguishing lock waits (`LCK_M_*`) from other waits.
- Assuming blocked process report is on by default.

**Cheat sheet**
- `sp_who2` -> BlkBy; DMVs: `dm_exec_requests`, `dm_os_waiting_tasks`, `dm_tran_locks`.
- Head blocker often "sleeping" with open tx.
- Historic: blocked process report + XE.
- blocking id -2 = orphaned DTC.
- Fix: shorter tx, indexes, RCSI, app timeouts.

---

## Topic 7: Deadlocks

**Study notes**

- A **deadlock** is a cycle of lock waits: A holds X on row1 and wants row2; B holds row2 and wants row1. Neither can proceed. The **lock monitor** thread checks for cycles (every 5 s by default, faster when deadlocks are frequent) and picks a **victim**: the session with the lowest `DEADLOCK_PRIORITY`; ties broken by least **log used / cheapest to roll back**. The victim gets **error 1205**, its transaction is rolled back.
- Common causes: inconsistent object access order, long transactions, missing indexes (scans lock more), read-then-update conversions (S->X), foreign keys without indexes, escalation, RCSI-off readers vs writers (S vs X cycle through different indexes: key lookup deadlock).
- Prevention: access objects in the same order, keep transactions short, proper indexes/covering indexes, `UPDLOCK` on read-then-write, RCSI/SNAPSHOT for readers, lower isolation, retry logic.
- Capture: the built-in **`system_health` Extended Events session** records `xml_deadlock_report`; trace flag 1222 writes to the error log.

```
T1: X(A) ──────wants──► B          T2: X(B) ──────wants──► A
        ▲_______________________________|     (cycle => victim chosen, 1205)
```

### 7.1 [Easy] What is a deadlock and how does SQL Server resolve it?
**Answer**
A deadlock is a circular wait between two or more sessions, each holding a lock the other needs. SQL Server detects the cycle, chooses a victim, rolls back its transaction and returns error 1205, letting the other proceed.

**Explanation**
It differs from blocking, which resolves when the holder finishes. A deadlock never resolves by itself, so the engine intervenes. The application must catch 1205 and retry.

**Code**
```sql
-- Setup
CREATE TABLE dbo.T1(Id INT PRIMARY KEY, V INT);
CREATE TABLE dbo.T2(Id INT PRIMARY KEY, V INT);
INSERT dbo.T1 VALUES (1,0); INSERT dbo.T2 VALUES (1,0);
```

### 7.2 [Follow-up] Reproduce a deadlock with two sessions.
**Answer**
Each session updates one table then, after a delay, updates the other table in the opposite order. The second step of both requests the lock the other holds, forming a cycle.

**Explanation**
Run both windows within 10 seconds of each other. One gets error 1205 "Transaction (Process ID n) was deadlocked on lock resources with another process and has been chosen as the deadlock victim."

**Code**
```sql
-- Session 1
BEGIN TRAN;
UPDATE dbo.T1 SET V = 1 WHERE Id = 1;
WAITFOR DELAY '00:00:10';
UPDATE dbo.T2 SET V = 1 WHERE Id = 1;   -- waits for session 2
COMMIT;

-- Session 2 (start right after session 1)
BEGIN TRAN;
UPDATE dbo.T2 SET V = 2 WHERE Id = 1;
WAITFOR DELAY '00:00:10';
UPDATE dbo.T1 SET V = 2 WHERE Id = 1;   -- cycle => one gets 1205
COMMIT;
```

### 7.3 [Medium] How is the victim chosen and how can you influence it?
**Answer**
By `SET DEADLOCK_PRIORITY` (LOW/NORMAL/HIGH or -10..10) first; if equal, the transaction that is cheapest to roll back (least log generated) is chosen. Set background/batch jobs to LOW so user-facing work survives.

**Explanation**
Victim choice doesn't guarantee correctness: the victim must be retried. Priority is session-level, so set it at the start of the batch or proc.

**Code**
```sql
SET DEADLOCK_PRIORITY LOW;      -- this session is preferred victim
-- or numeric
SET DEADLOCK_PRIORITY -5;
SELECT @@SPID, deadlock_priority FROM sys.dm_exec_sessions WHERE session_id = @@SPID;
```

### 7.4 [Medium] Where do you find the deadlock graph after the fact, and how do you read it?
**Answer**
In the `system_health` XE session (event `xml_deadlock_report`), in the ring buffer or `system_health*.xel` files. The XML has `<victim-list>`, `<process-list>` (statements, isolation level, input buffer), and `<resource-list>` (keylock/pagelock/objectlock with owner/waiter modes).

**Explanation**
Read it as: each process owns a resource and waits for another; find the two statements and the indexes (`indexname`) involved. The graph in SSMS (save as `.xdl`) shows ovals (processes) and rectangles (resources). Azure SQL DB: use XE or Query Store/Intelligent Insights.

**Code**
```sql
SELECT CAST(f.event_data AS XML) AS deadlock_event
FROM sys.fn_xe_file_target_read_file('system_health*.xel', NULL, NULL, NULL) AS f
WHERE f.object_name = 'xml_deadlock_report';
```

### 7.5 [Hard] Give a deadlock with a single table and a single row involved, with no opposite order. How can it happen?
**Answer**
Two common cases: (1) **conversion deadlock** - both sessions hold S (REPEATABLE READ/HOLDLOCK) on the same row and both try to upgrade to X; (2) **bookmark/key-lookup deadlock** - a reader holds S on a nonclustered index key and wants the clustered key, while a writer holds X on the clustered key and wants to update the nonclustered key. Fix with `UPDLOCK`, a covering index, or RCSI.

**Explanation**
Case 2 happens because a writer updates the clustered index then the NC index, while the reader scans NC then seeks clustered - opposite order on the *same* row. A covering index removes the lookup; RCSI removes the reader's S locks.

**Gotchas / follow-ups**
- Parallelism-related intra-query deadlocks exist (exchange events, `intra-query parallelism deadlock`).
- Deadlock graph shows two `keylock`s on different indexes of the same table - the key-lookup signature.

**Code**
```sql
-- Conversion deadlock (run in both sessions step by step)
SET TRANSACTION ISOLATION LEVEL REPEATABLE READ;
BEGIN TRAN;
SELECT V FROM dbo.T1 WHERE Id = 1;               -- S held
WAITFOR DELAY '00:00:10';
UPDATE dbo.T1 SET V = V + 1 WHERE Id = 1;        -- both want X => 1205
COMMIT;
-- Fix: SELECT ... FROM dbo.T1 WITH (UPDLOCK) WHERE Id = 1;
```

### 7.6 [Tricky] How do you handle deadlocks in C# and is it safe to retry? What if the deadlock victim was inside `TransactionScope` or an EF Core `SaveChanges`?
**Answer**
Catch `SqlException` with `Number == 1205`, ensure the *whole* unit of work (the entire transaction) is re-executed from scratch with a small randomized backoff, capped at a few attempts. It's only safe if the unit of work is idempotent/atomic within the DB transaction (no external side-effects already performed).

**Explanation**
After 1205 the server already rolled back; you must not continue the same transaction. In EF Core, `EnableRetryOnFailure` (SqlServerRetryingExecutionStrategy) treats 1205 as transient and retries, but user-initiated transactions then need `strategy.ExecuteAsync(...)` wrapping the whole transaction. Don't retry inside the transaction block itself.

**Gotchas / follow-ups**
- Recreate the DbContext/tracked state each attempt (stale entities).
- Retrying non-idempotent actions (send email, call payment API) duplicates effects.
- Also treat 3960 (snapshot conflict) similarly.

**Code**
```sql
-- T-SQL retry (server side) for 1205
DECLARE @try INT = 0;
WHILE @try < 3
BEGIN
  BEGIN TRY
    BEGIN TRAN; /* work */ COMMIT; BREAK;
  END TRY
  BEGIN CATCH
    IF XACT_STATE() <> 0 ROLLBACK;
    IF ERROR_NUMBER() = 1205 AND @try < 2
    BEGIN SET @try += 1; WAITFOR DELAY '00:00:00.200'; END
    ELSE THROW;
  END CATCH
END
```
```csharp
// EF Core: wrap whole transaction in the execution strategy
var strategy = db.Database.CreateExecutionStrategy();
await strategy.ExecuteAsync(async () =>
{
    await using var tx = await db.Database.BeginTransactionAsync();
    // ... changes ...
    await db.SaveChangesAsync();
    await tx.CommitAsync();
});
```

**Interview traps**
- "Deadlocks are a SQL Server bug" - they're an application/design problem.
- Retrying only the failed statement instead of the whole transaction.
- Using NOLOCK to "fix" deadlocks (writer-writer cycles remain).
- Forgetting that the victim chosen may be the "innocent" session.

**Cheat sheet**
- Deadlock = cycle; victim gets 1205 and rolls back.
- Victim: DEADLOCK_PRIORITY, then lowest rollback cost.
- Graph: `system_health` XE `xml_deadlock_report`.
- Prevent: same order, short tx, indexes, UPDLOCK, RCSI.
- Retry the whole transaction, with backoff.

---

## Topic 8: Table hints

**Study notes**

- **NOLOCK / READUNCOMMITTED**: no S locks taken, ignores X locks -> dirty reads, duplicates/missed rows, "Could not continue scan with NOLOCK due to data movement" (error 601). Still takes Sch-S. Not a performance silver bullet.
- **READPAST**: skip rows locked by others (row-level locks). Used for queue consumption. Only allowed under READ COMMITTED or REPEATABLE READ (error under SNAPSHOT/SERIALIZABLE).
- **UPDLOCK**: take U locks and hold to end of transaction (read-for-update).
- **HOLDLOCK**: = SERIALIZABLE for that table (range locks held to tx end).
- **ROWLOCK / PAGLOCK / TABLOCK / TABLOCKX / XLOCK**: request granularity/mode. Hints are requests; the engine may still escalate.
- **READCOMMITTEDLOCK**: use locking RC even when RCSI is on.
- Hints with `WITH (...)` syntax (omitting `WITH` is deprecated).

### 8.1 [Easy] What does `WITH (NOLOCK)` do?
**Answer**
It reads without taking shared locks and ignores others' exclusive locks, equivalent to READ UNCOMMITTED for that table. It never blocks on writers but can return uncommitted, duplicate or missing rows.

**Explanation**
Acceptable for rough monitoring/approximate counts where inaccuracy is tolerable. Not acceptable for financial/business logic.

**Code**
```sql
SELECT COUNT(*) FROM dbo.Orders WITH (NOLOCK);
```

### 8.2 [Follow-up] What bad things can happen with NOLOCK beyond dirty reads?
**Answer**
Rows can be read twice or skipped when pages split during an allocation-order scan, you can read data that is later rolled back, and you can get error 601 or even read LOB/off-row data inconsistently. It also doesn't avoid Sch-M blocking.

**Explanation**
The recommended alternative is RCSI/SNAPSHOT: readers don't block, but see only committed data.

**Code**
```sql
-- Prefer this to NOLOCK sprinkled everywhere
ALTER DATABASE MyDb SET READ_COMMITTED_SNAPSHOT ON WITH ROLLBACK IMMEDIATE;
```

### 8.3 [Medium] What do UPDLOCK, HOLDLOCK and ROWLOCK do and when do you combine them?
**Answer**
`UPDLOCK` takes update locks held until end of transaction; `HOLDLOCK` holds range locks (serializable behaviour); `ROWLOCK` asks for row granularity. `UPDLOCK, HOLDLOCK` together are the standard "upsert/insert-if-not-exists" guard because they lock the key range so two sessions can't both see "not found" and both insert.

**Explanation**
Without HOLDLOCK, `IF NOT EXISTS (...) INSERT` has a race: both sessions pass the check. With a unique index as the final safety net, you may still catch duplicate key errors (2627/2601) and retry. `MERGE` has known edge-case bugs; the explicit pattern is generally safer.

**Code**
```sql
BEGIN TRAN;
IF NOT EXISTS (SELECT 1 FROM dbo.Acct WITH (UPDLOCK, HOLDLOCK) WHERE Id = 50)
    INSERT dbo.Acct(Id, Bal) VALUES (50, 0);
ELSE
    UPDATE dbo.Acct SET Bal += 1 WHERE Id = 50;
COMMIT;
```

### 8.4 [Hard] Do hints guarantee a lock granularity? What about TABLOCK on INSERT...SELECT?
**Answer**
No, `ROWLOCK` is a hint: escalation can still happen, and the optimizer/engine may use page or table locks (e.g., if the lock memory limit is reached). `TABLOCK` on `INSERT ... SELECT` into a heap enables **minimal logging** (in simple/bulk-logged recovery) and parallel insert, but takes a table X (BU for bulk) lock, blocking others.

**Explanation**
Also, hints don't change the semantics of the session isolation level in every case; `READPAST` can only be used under READ COMMITTED or REPEATABLE READ (not SNAPSHOT/SERIALIZABLE/HOLDLOCK). Hints bind plan choices (index hints can break when indexes change) - treat them as last resort and document them.

**Gotchas / follow-ups**
- `READPAST` with `UPDLOCK, ROWLOCK` is the queue pattern; without ROWLOCK, page locks could skip entire pages.
- `TABLOCKX` on staging tables in ETL is OK, on OLTP tables is not.

**Code**
```sql
-- Minimal logging load (SIMPLE or BULK_LOGGED, heap or empty clustered)
INSERT INTO dbo.Stage WITH (TABLOCK) (Id, V)
SELECT Id, V FROM dbo.SourceData;
```

### 8.5 [Tricky] Does NOLOCK help when the table is being updated by a long transaction under RCSI? What does `SET TRANSACTION ISOLATION LEVEL READ UNCOMMITTED` differ from the hint?
**Answer**
NOLOCK still reads uncommitted data (it overrides RCSI for that table), so it returns dirty data rather than the committed version - worse than RCSI default. The session-level setting applies to all tables in the session; the hint applies per table reference.

**Explanation**
Under RCSI there is rarely a reason for NOLOCK. Views can embed NOLOCK and surprise callers. EF Core has no direct hint support; devs use interceptors or raw SQL - another reason to prefer RCSI.

**Gotchas / follow-ups**
- Isolation hint inside a view/TVF applies to that reference.
- An uncommitted insert/delete can appear/disappear under NOLOCK.

**Code**
```sql
-- With RCSI on, compare
SELECT Bal FROM dbo.Acct WHERE Id = 1;                  -- committed version
SELECT Bal FROM dbo.Acct WITH (NOLOCK) WHERE Id = 1;    -- possibly uncommitted dirty value
```

**Interview traps**
- "NOLOCK makes queries faster" - it avoids waiting, not work; correctness cost is high.
- Using READPAST without understanding it silently skips rows.
- Assuming ROWLOCK prevents escalation.
- Using `MERGE` for concurrency-safe upsert without hints.

**Cheat sheet**
- NOLOCK = dirty reads + possible dup/missing rows.
- UPDLOCK = read-for-update; HOLDLOCK = serializable range lock.
- READPAST skips locked rows (queues).
- Hints are requests; escalation can still occur.
- Prefer RCSI over NOLOCK.

---

## Topic 9: Concurrency control, rowversion & queue pattern

**Study notes**

- **Pessimistic**: lock up front (`UPDLOCK`, serializable) so others wait; good when conflicts are frequent and tx are short. Don't hold DB locks across user think time.
- **Optimistic**: read without locks, detect conflict at write time and retry/notify. In SQL Server use a **`rowversion`** (aka `timestamp`) column: an 8-byte binary auto-incremented per-DB on every insert/update. Unrelated to date/time.
- EF Core: `[Timestamp] byte[] RowVersion` -> `IsRowVersion()` adds `WHERE Id=@id AND RowVersion=@orig`; 0 rows affected -> `DbUpdateConcurrencyException`.
- **Job-claim/queue pattern**: workers select-and-lock next available row with `UPDLOCK, READPAST, ROWLOCK`, so each worker gets a different row without waiting.

### 9.1 [Easy] What is optimistic concurrency and how does `rowversion` implement it?
**Answer**
Optimistic concurrency assumes conflicts are rare, doesn't lock while the user edits, and checks at update time that the row hasn't changed. A `rowversion` column changes on each write, so `UPDATE ... WHERE Id=@Id AND RowVer=@Orig` affects 0 rows if someone else modified it first.

**Explanation**
`rowversion` values are unique within the database and monotonically increasing; only one per table allowed. Compare as `binary(8)`. It's cheaper and more reliable than comparing all columns or using datetime.

**Code**
```sql
CREATE TABLE dbo.Product(Id INT PRIMARY KEY, Name NVARCHAR(100), Price MONEY, RowVer ROWVERSION);
DECLARE @orig BINARY(8) = (SELECT RowVer FROM dbo.Product WHERE Id = 1);
UPDATE dbo.Product SET Price = 9.99 WHERE Id = 1 AND RowVer = @orig;
IF @@ROWCOUNT = 0 RAISERROR('Concurrency conflict', 16, 1);
```

### 9.2 [Follow-up] How do you do this in EF Core and what happens on conflict?
**Answer**
Map a `byte[]` property with `[Timestamp]` or `.IsRowVersion()`. On `SaveChanges`, EF adds the version to the WHERE clause; if no row matches it throws `DbUpdateConcurrencyException`, where you can reload, merge, or tell the user.

**Explanation**
Alternative is `[ConcurrencyCheck]` on specific columns (compares those original values). SQL shape is the same pattern as above, with `SELECT @@ROWCOUNT` and, for rowversion, an `OUTPUT`/`SELECT` of the new value.

**Code**
```sql
-- SQL EF Core generates (conceptually)
SET NOCOUNT ON;
UPDATE [Product] SET [Price] = @p0
OUTPUT INSERTED.[RowVer]
WHERE [Id] = @p1 AND [RowVer] = @p2;
```

### 9.3 [Medium] Pessimistic vs optimistic - when pick which?
**Answer**
Optimistic for web/user-edit scenarios with low conflict and long think time; pessimistic (UPDLOCK in a short tx) for hot rows (counters, inventory, queues) where retry storms would waste work. Never hold locks while waiting on user input.

**Explanation**
Alternatives for hot counters: atomic update `SET Qty = Qty - 1 WHERE Qty > 0`, sequence tables, or application locks (`sp_getapplock`) for named critical sections.

**Code**
```sql
-- Atomic stock decrement, no read-modify-write
UPDATE dbo.Stock SET Qty = Qty - @n WHERE ProductId = @p AND Qty >= @n;
IF @@ROWCOUNT = 0 THROW 50001, 'Out of stock', 1;

-- Application lock
EXEC sp_getapplock @Resource = 'nightly-job', @LockMode = 'Exclusive', @LockOwner = 'Session', @LockTimeout = 0;
```

### 9.4 [Hard] Implement a work queue where multiple workers each claim one job without blocking each other.
**Answer**
Use a single atomic statement: `WITH cte AS (SELECT TOP (1) ... WITH (UPDLOCK, READPAST, ROWLOCK) ORDER BY ...) UPDATE cte SET Status = 'Processing' OUTPUT inserted.*`. `READPAST` skips rows locked by other workers; `UPDLOCK` ensures the claimed row is locked; `ROWLOCK` keeps skipping row-level.

**Explanation**
A supporting index on `(Status, Id)` is vital so the TOP(1) doesn't scan locked rows. Add a lease/`LockedUntil` timestamp so crashed workers' jobs get reclaimed. Ordering is approximate (skipped rows) - strict FIFO isn't guaranteed.

**Gotchas / follow-ups**
- Don't hold the claim transaction open during job processing; claim by status update, commit, then process.
- Poison messages: add `Attempts` and a dead-letter state.
- Alternative: Service Broker, or an external broker (Azure Service Bus) for heavy needs.
- Under RCSI this pattern still works because of explicit hints.

**Code**
```sql
CREATE TABLE dbo.Job(Id BIGINT IDENTITY PRIMARY KEY, Payload NVARCHAR(MAX), Status TINYINT NOT NULL DEFAULT 0,
                     LockedUntil DATETIME2 NULL, Attempts INT NOT NULL DEFAULT 0);
CREATE INDEX IX_Job_Status ON dbo.Job(Status, Id);

;WITH next AS (
  SELECT TOP (1) *
  FROM dbo.Job WITH (UPDLOCK, READPAST, ROWLOCK)
  WHERE Status = 0 OR (Status = 1 AND LockedUntil < SYSUTCDATETIME())
  ORDER BY Id
)
UPDATE next SET Status = 1, LockedUntil = DATEADD(MINUTE, 5, SYSUTCDATETIME()), Attempts += 1
OUTPUT inserted.Id, inserted.Payload;
```

### 9.5 [Tricky] Why is `IF EXISTS ... UPDATE ELSE INSERT` unsafe, and how does a rowversion interact with `MERGE`/triggers/replication?
**Answer**
Two sessions can both see "not exists" and both insert (race), producing duplicates or PK violations. Fix with a unique constraint plus `UPDLOCK, HOLDLOCK` on the check, or try-insert-catch-2627. Also, `rowversion` changes on *any* update, including no-op updates, and is only meaningful within one database. Triggers that update the same row bump the version again, so EF Core needs to know about the trigger (it then uses `OUTPUT INTO` a temp table rather than a plain `OUTPUT`).

**Explanation**
The race exists because the existence check at READ COMMITTED releases its lock immediately. `UPDLOCK, HOLDLOCK` locks the key range so the second session waits; a unique index is the final safety net regardless.

**Gotchas / follow-ups**
- EF Core + triggers: configure `ToTable(t => t.HasTrigger(...))` so EF uses `OUTPUT INTO`.
- `rowversion` ≠ `datetime`; do not display/sort as time.
- Concurrent upsert alternative: `MERGE ... WITH (HOLDLOCK)` and unique key.

**Code**
```sql
BEGIN TRAN;
UPDATE dbo.Acct WITH (UPDLOCK, HOLDLOCK) SET Bal += 1 WHERE Id = 77;
IF @@ROWCOUNT = 0
  INSERT dbo.Acct(Id, Bal) VALUES (77, 1);
COMMIT;
```

**Interview traps**
- Treating `timestamp` as a date.
- Holding pessimistic locks across user think time.
- `READPAST` without an index -> scans and poor throughput.
- Not adding a lease for claimed jobs (stuck forever when worker dies).

**Cheat sheet**
- Optimistic = rowversion + `WHERE RowVer = @orig`, 0 rows = conflict.
- EF: `[Timestamp]` -> `DbUpdateConcurrencyException`.
- Hot rows: atomic update / UPDLOCK.
- Queue: `UPDLOCK, READPAST, ROWLOCK` + index (Status, Id).
- Upsert: `UPDLOCK, HOLDLOCK` + unique key.

---

## Topic 10: Distributed transactions

**Study notes**

- A **distributed transaction** spans multiple resource managers (two databases on different servers, SQL + MSMQ, linked server writes). Coordinated by **MSDTC** (Microsoft Distributed Transaction Coordinator) using **two-phase commit (2PC)**:
  1. **Prepare**: coordinator asks every participant to durably log "I can commit" and vote.
  2. **Commit**: if all vote yes, coordinator logs the decision and tells everyone to commit; any "no" => rollback everywhere.
- In .NET: `System.Transactions.TransactionScope`. A transaction starts as a lightweight local one and gets **promoted to MSDTC** when a second durable resource (e.g., a second SQL Server, or a second open connection in some scenarios) enlists.
- Costs: latency (extra log forces + network round trips), locks held longer, blocking when the coordinator or a participant is unavailable ("in-doubt" transactions), firewall/DTC configuration, limited cloud support.
- Alternatives: **Saga**/compensating actions, **outbox pattern**, idempotent consumers, eventual consistency, single database boundary.

```
App ─► Coordinator(MSDTC)
          ├─ PREPARE ─► DB1  (log, vote YES)
          ├─ PREPARE ─► DB2  (log, vote YES)
          └─ COMMIT  ─► DB1, DB2    (any NO / timeout => ROLLBACK all)
```

### 10.1 [Easy] What is a distributed transaction and what is two-phase commit?
**Answer**
It's one atomic transaction across multiple independent resource managers. 2PC ensures atomicity: first every participant is asked to prepare (durably promise it can commit), and only if all agree does the coordinator tell them to commit.

**Explanation**
The window between prepare and commit is the risky one: a participant that has voted yes must hold its locks until the coordinator's decision arrives (an "in-doubt" state).

**Code**
```sql
-- T-SQL distributed transaction across a linked server
SET XACT_ABORT ON;   -- required for distributed transactions through linked servers
BEGIN DISTRIBUTED TRANSACTION;
  UPDATE LocalDb.dbo.Acct SET Bal -= 10 WHERE Id = 1;
  UPDATE RemoteSrv.RemoteDb.dbo.Acct SET Bal += 10 WHERE Id = 2;
COMMIT;
```

### 10.2 [Follow-up] When does a `TransactionScope` in C# get promoted to MSDTC?
**Answer**
It starts as a lightweight (local) transaction and is promoted when a second durable resource manager enlists - for example connecting to a different SQL Server/database server, or holding two open connections concurrently that can't share the same transaction. Classic fix: reuse a single connection, or open/close sequentially with the same connection string so the pool reuses one physical connection.

**Explanation**
Promotion to MSDTC is supported in SqlClient on .NET Framework and, on .NET (Core), only on Windows from .NET 7 onward; elsewhere you get `PlatformNotSupportedException`. Promotion shows up as DTC traffic and slower commits. EF Core `SaveChanges` uses its own local transaction; wrapping multiple contexts/connections in one scope is the typical accidental promotion.

**Code**
```csharp
using var scope = new TransactionScope(TransactionScopeAsyncFlowOption.Enabled);
using (var c1 = new SqlConnection(cs1)) { await c1.OpenAsync(); /* work */ }
using (var c2 = new SqlConnection(cs2)) { await c2.OpenAsync(); /* different server => promoted */ }
scope.Complete();
```
```sql
-- Observe distributed transactions on the server
SELECT * FROM sys.dm_tran_active_transactions WHERE transaction_type = 4;  -- 4 = distributed
```

### 10.3 [Medium] Why should you avoid distributed transactions in modern architectures?
**Answer**
They reduce availability (any participant failure stalls others), add latency, hold locks longer, are hard to operate (MSDTC config, firewall, security) and aren't supported across most cloud services/message brokers. Prefer a single transactional boundary plus the outbox pattern and idempotent consumers/sagas.

**Explanation**
CAP/availability: if the coordinator dies between prepare and commit, participants stay blocked in-doubt until recovery. Microservices usually own their data, so 2PC couples services. Outbox: write business data and an "event" row in one local transaction; a relay publishes it with at-least-once delivery; consumers dedupe by message ID.

**Code**
```sql
-- Outbox pattern: single local transaction
BEGIN TRAN;
  INSERT dbo.Orders(Id, Total) VALUES (@id, @total);
  INSERT dbo.Outbox(MessageId, Type, Payload, CreatedAt)
  VALUES (NEWID(), 'OrderCreated', @json, SYSUTCDATETIME());
COMMIT;
```

### 10.4 [Hard] What are in-doubt transactions and how do you resolve one?
**Answer**
A transaction that has prepared but hasn't received the coordinator's commit/abort decision, e.g., after coordinator crash. It keeps locks until resolved. Best fix: restore coordinator connectivity so it resolves automatically; otherwise manually resolve in Component Services (MSDTC "Transactions"/"Transaction List" -> Resolve Commit/Abort) or `KILL <UOW GUID>`.

**Explanation**
Manually forcing commit/abort risks heuristic damage (other participants may have decided the opposite). The orphaned session shows `blocking_session_id = -2`.

**Gotchas / follow-ups**
- Always check the outcome at other participants before forcing.
- Timeouts: `TransactionScope` default 1 minute (max governed by machine.config `maxTimeout`).
- Savepoints not allowed in distributed tx.
- Azure SQL Database supports *elastic transactions* (limited) but not general MSDTC.

**Code**
```sql
SELECT DISTINCT request_owner_guid FROM sys.dm_tran_locks
WHERE request_owner_guid <> '00000000-0000-0000-0000-000000000000';
-- KILL '<uow-guid>';    -- last resort (may cause heuristic mismatch)
```

### 10.5 [Tricky] Does a cross-database transaction on the same SQL Server instance need MSDTC?
**Answer**
No. Databases on the same instance share one transaction manager, so a normal `BEGIN TRAN` spans them, no MSDTC. One caveat: with Always On AGs, cross-database transactions are not guaranteed atomic across a failover unless the AG is created with per-database DTC support (`DTC_SUPPORT = PER_DB`, SQL 2016 SP2+); database mirroring doesn't support them.

**Explanation**
This makes "DB per module" within one instance cheap for transactions, but undermines later separation onto separate servers. Also, in full recovery, restoring one database to a point in time breaks cross-database consistency.

**Gotchas / follow-ups**
- AG + cross-DB writes: consider placing them in the same AG and testing failover; `DTC_SUPPORT = PER_DB` on the AG.
- Backup/restore of related databases should use marked transactions (`BEGIN TRAN ... WITH MARK`) for consistent PITR.

**Code**
```sql
BEGIN TRAN;
  UPDATE DbA.dbo.Acct SET Bal -= 1 WHERE Id = 1;
  UPDATE DbB.dbo.Ledger SET Amt += 1 WHERE Id = 1;   -- no MSDTC
COMMIT;
```

**Interview traps**
- "TransactionScope always means MSDTC" - only when promoted.
- Retrying an in-doubt transaction blindly.
- Believing 2PC makes systems highly available (it doesn't).
- Forgetting that outbox gives at-least-once, so consumers must be idempotent.

**Cheat sheet**
- 2PC = prepare then commit; coordinator = MSDTC.
- Promotion happens on 2nd durable resource/connection.
- Same-instance cross-DB: no MSDTC.
- Avoid: use outbox + idempotency + sagas.
- In-doubt = locks held until resolved; blocking id -2.

---

## Topic 11: Recovery models, log backups & restore

**Study notes**

| Model | Log | Log backups | Point-in-time | Typical use |
|---|---|---|---|---|
| SIMPLE | truncated after checkpoint | not possible | no (only full/diff) | dev/test, data warehouses reloadable |
| FULL | kept until log backup | required | yes | production OLTP |
| BULK_LOGGED | like FULL but minimal logging for bulk ops | yes | not within a log backup that contains bulk-logged ops | temporary during bulk loads/index rebuilds |

- **Backup types**: **Full** (whole DB + enough log to be consistent), **Differential** (changes since last full, base = last full), **Transaction log** (all log since last log backup), plus file/filegroup, partial, `COPY_ONLY`.
- **Restore sequence**: latest full `WITH NORECOVERY` -> latest differential `WITH NORECOVERY` -> all log backups in order `WITH NORECOVERY` ... last one `WITH RECOVERY` (or `STOPAT` for PITR).
- **RPO** (max data loss) is driven by log backup frequency / replication; **RTO** (time to recover) by restore speed, size, automation, and HA.
- Log can't truncate if `log_reuse_wait_desc` is e.g. `LOG_BACKUP`, `ACTIVE_TRANSACTION`, `REPLICATION`, `AVAILABILITY_REPLICA`, `OLDEST_PAGE`.
- **DBCC CHECKDB**: verifies logical and physical integrity; run regularly (weekly) and on restored backups. Backups with `CHECKSUM` detect page corruption at backup time.

### 11.1 [Easy] Compare SIMPLE, FULL and BULK_LOGGED recovery models.
**Answer**
SIMPLE auto-truncates the log at checkpoint and has no log backups or point-in-time restore. FULL keeps all log until you back it up, enabling point-in-time recovery. BULK_LOGGED is FULL with minimal logging for certain bulk operations, trading some PITR granularity for faster loads.

**Explanation**
Production systems that cannot lose more than minutes of data need FULL + frequent log backups. A DB in FULL without log backups is a classic cause of runaway log growth.

**Code**
```sql
SELECT name, recovery_model_desc, log_reuse_wait_desc FROM sys.databases;
ALTER DATABASE MyDb SET RECOVERY FULL;
BACKUP DATABASE MyDb TO DISK = N'D:\bak\MyDb_full.bak' WITH CHECKSUM, COMPRESSION, INIT;  -- start the chain
```

### 11.2 [Follow-up] The transaction log file has grown to 200 GB. What do you check?
**Answer**
Check `log_reuse_wait_desc`. Most often it's `LOG_BACKUP` (FULL model with no/infrequent log backups) or `ACTIVE_TRANSACTION` (a long open transaction). Fix the cause (schedule log backups, resolve the transaction, fix AG/replication lag), then optionally shrink once to a sane size; do not routinely shrink.

**Explanation**
Truncation only marks log space reusable; the file doesn't shrink. Do not switch to SIMPLE and back casually in production - it breaks the log chain (take a full/diff afterwards). Pre-size the log, set fixed autogrowth increments, and watch VLF count.

**Code**
```sql
SELECT name, log_reuse_wait_desc FROM sys.databases WHERE name = 'MyDb';
DBCC SQLPERF(LOGSPACE);
SELECT * FROM sys.dm_db_log_info(DB_ID());   -- VLFs (2016 SP2+)
BACKUP LOG MyDb TO DISK = N'D:\bak\MyDb_log.trn' WITH CHECKSUM;
DBCC OPENTRAN;   -- oldest active transaction
-- after log is truncatable, once:
USE MyDb; DBCC SHRINKFILE (MyDb_log, 4096);   -- target MB
```

### 11.3 [Medium] Describe a restore strategy to recover to 10:42 yesterday.
**Answer**
Restore the most recent full backup before the target time `WITH NORECOVERY`, then the latest differential before the target (optional), then all log backups in sequence, and on the final log use `WITH STOPAT = '<time>', RECOVERY`. Take a tail-log backup first if the damaged DB is still accessible.

**Explanation**
The chain requires unbroken LSN continuity. Restoring to a new name (`WITH MOVE`) lets you extract data without overwriting production. For Azure SQL Database use built-in PITR restore (creates a new database).

**Code**
```sql
RESTORE DATABASE MyDb_Restore FROM DISK = N'D:\bak\MyDb_full.bak'
  WITH MOVE 'MyDb' TO 'D:\data\MyDb_R.mdf', MOVE 'MyDb_log' TO 'D:\data\MyDb_R.ldf', NORECOVERY;
RESTORE DATABASE MyDb_Restore FROM DISK = N'D:\bak\MyDb_diff.bak' WITH NORECOVERY;
RESTORE LOG MyDb_Restore FROM DISK = N'D:\bak\MyDb_log1.trn' WITH NORECOVERY;
RESTORE LOG MyDb_Restore FROM DISK = N'D:\bak\MyDb_log2.trn'
  WITH STOPAT = '2026-10-15T10:42:00', RECOVERY;
```

### 11.4 [Medium] Define RPO and RTO and design a plan for RPO 5 min / RTO 30 min.
**Answer**
RPO is how much data you can afford to lose; RTO is how long the service may be down. RPO 5 min means log backups at least every 5 minutes (or synchronous/near-sync replication); RTO 30 min means you can't rely on a lengthy restore of a huge DB, so use an HA/DR mechanism (AG/failover group) or fast restore automation with tested runbooks.

**Explanation**
Typical: weekly full, daily differential, 5-minute log backups, copied off-site/immutable storage, plus restore tests and CHECKDB. Backups are not HA; they're the last line of defense. An untested backup is not a backup.

**Code**
```sql
-- Verify a backup (does not guarantee restorability fully)
RESTORE VERIFYONLY FROM DISK = N'D:\bak\MyDb_full.bak' WITH CHECKSUM;
-- Which backups exist
SELECT TOP 20 database_name, type, backup_start_date, backup_finish_date, is_copy_only
FROM msdb.dbo.backupset ORDER BY backup_finish_date DESC;
```

### 11.5 [Hard] How does a differential backup work and what breaks the chain? What are COPY_ONLY and tail-log backups?
**Answer**
A differential records extents changed since the last non-copy-only full backup (the "differential base") via the DCM page. A new regular full backup resets the base; `COPY_ONLY` full does not. A tail-log backup captures log not yet backed up, taken before restoring over a damaged DB (`BACKUP LOG ... WITH NORECOVERY` or `NO_TRUNCATE` if the DB is damaged).

**Explanation**
Log backups are independent of differentials: log chain breaks only if a log backup is lost, if you switch to SIMPLE, or if a log backup is skipped. Restoring a full + log chain works without any differential. Differentials grow through the week, so size can approach full backups.

**Gotchas / follow-ups**
- Another tool taking a full backup (non-copy-only) silently becomes the new differential base.
- Backups on an AG secondary: full backups must be `COPY_ONLY`; log backups are allowed (regular) and join the same log chain.
- Backup to URL (Azure Blob) is supported (`TO URL`).

**Code**
```sql
BACKUP DATABASE MyDb TO DISK = N'D:\bak\MyDb_copy.bak' WITH COPY_ONLY, CHECKSUM;     -- doesn't reset diff base
BACKUP DATABASE MyDb TO DISK = N'D:\bak\MyDb_diff.bak' WITH DIFFERENTIAL, CHECKSUM;
BACKUP LOG MyDb TO DISK = N'D:\bak\MyDb_tail.trn' WITH NORECOVERY;                    -- tail-log, leaves DB RESTORING
```

### 11.6 [Tricky] When is DBCC CHECKDB needed and what do you do on corruption? Why isn't a successful backup proof of a healthy database?
**Answer**
Run CHECKDB regularly and against restored backups; it detects physical and logical corruption. A backup (even with CHECKSUM) only verifies page checksums of pages it reads; it won't catch logical corruption (e.g., corrupt allocation structures, bad data between pages) and backs up corrupt pages as they are. On corruption, restore from a clean backup; use `REPAIR_ALLOW_DATA_LOSS` only as a last resort.

**Explanation**
Page-level restore (Enterprise) and AG automatic page repair can fix isolated page corruption. Set `PAGE_VERIFY CHECKSUM`, alert on errors 823/824/825, and keep suspect_pages monitored.

**Gotchas / follow-ups**
- CHECKDB is IO/CPU heavy: offload to a restored copy or secondary, use `PHYSICAL_ONLY` more frequently with a full check periodically.
- Azure SQL Database runs integrity checks itself on the platform but you still need app-level validation.

**Code**
```sql
DBCC CHECKDB (MyDb) WITH NO_INFOMSGS, ALL_ERRORMSGS;
DBCC CHECKDB (MyDb) WITH PHYSICAL_ONLY, NO_INFOMSGS;       -- cheaper
SELECT * FROM msdb.dbo.suspect_pages;
```

**Interview traps**
- "Switch to SIMPLE to shrink the log" - breaks PITR/log chain.
- FULL recovery with no log backups.
- Confusing differential with log backup (a differential is *not* incremental from the last diff).
- Never testing restores; never running CHECKDB.
- Treating `SHRINKFILE` as routine maintenance.

**Cheat sheet**
- SIMPLE: no log backups; FULL: PITR; BULK_LOGGED: minimal logging for bulk.
- Log won't shrink until truncated; check `log_reuse_wait_desc`.
- Restore: full -> diff -> logs, `NORECOVERY` until last, `STOPAT` for PITR.
- RPO = data loss; RTO = downtime.
- CHECKDB regularly; test restores.

---

## Topic 12: High availability & DR

**Study notes**

| Feature | Level | Data copy | Failover | Readable secondary | Notes |
|---|---|---|---|---|---|
| **Always On Availability Groups** | database(s) | yes (log stream, sync/async) | automatic (sync) or manual | yes | Enterprise (Basic AG in Standard: 1 DB, no readable secondary); needs WSFC (or cluster-less read-scale/Linux Pacemaker) |
| **Failover Cluster Instance (FCI)** | whole instance | no (shared storage) | automatic | no (passive node) | protects against node failure, not storage failure |
| **Log shipping** | database | yes (log backup/copy/restore) | manual | standby mode possible (read-only between restores) | simple, delayed, async; RPO = interval |
| **Transactional replication** | table/article level | yes (distribution agent) | none (not HA) | yes (subscriber is writable DB) | for distributing/reporting data; schema limits |
| **Database mirroring** | database | yes | automatic with witness (high safety) | no | **deprecated** since SQL 2012; replaced by AG |

- **Sync commit** = zero data loss (RPO 0) at cost of commit latency; **async** = lower latency but potential loss.
- Azure: **active geo-replication**, **auto-failover groups**, zone-redundant configurations, built-in 99.99% SLA tiers.

```
Primary ──log stream──► Sync replica   (auto failover, RPO 0)
        └─log stream──► Async replica  (DR, manual failover, possible loss)
```

### 12.1 [Easy] What's the difference between HA and DR?
**Answer**
HA keeps the service up through local failures (node/instance/zone) with fast, usually automatic failover. DR recovers from site/region-level disasters or data corruption/human error and typically accepts some data loss and longer recovery time.

**Explanation**
They're complementary and measured by RPO/RTO. HA features do not protect against `DELETE FROM` mistakes (the delete replicates instantly), so backups/PITR remain essential.

**Code**
```sql
-- AG health overview
SELECT ag.name, ar.replica_server_name, ars.role_desc, ars.synchronization_health_desc
FROM sys.availability_groups ag
JOIN sys.availability_replicas ar ON ar.group_id = ag.group_id
JOIN sys.dm_hadr_availability_replica_states ars ON ars.replica_id = ar.replica_id;
```

### 12.2 [Follow-up] Compare Always On AG and Failover Cluster Instance.
**Answer**
FCI protects the *instance* using shared storage: one copy of data, so storage is a single point of failure, and the passive node isn't readable. AG protects *databases* by shipping log to replicas with separate storage, allows readable secondaries and offloaded backups, but system databases/logins/jobs aren't replicated.

**Explanation**
AG needs you to sync logins (SIDs), SQL Agent jobs, linked servers manually (or use contained AGs in SQL 2022). Failover time: FCI restarts the instance and runs recovery; AG failover is usually faster as the secondary is already running.

**Code**
```sql
-- Route read-only workloads to a secondary (client side)
-- Connection string: Server=tcp:AGListener;Database=Sales;ApplicationIntent=ReadOnly;MultiSubnetFailover=True
SELECT DATABASEPROPERTYEX(DB_NAME(), 'Updateability') AS updateability;   -- READ_ONLY on secondary
```

### 12.3 [Medium] Synchronous vs asynchronous commit in an AG - what are the trade-offs?
**Answer**
Synchronous: the primary waits for the secondary to harden the log before committing, so no data loss and automatic failover are possible, but commit latency rises and a slow/failed secondary can hurt the primary. Asynchronous: primary doesn't wait, so lower latency, suited for distant DR sites, but data loss is possible on failover and failover is manual (forced).

**Explanation**
Usually: 2 sync replicas in one datacenter/zone for HA (automatic failover) + 1 async replica in another region for DR. Monitor `HADR_SYNC_COMMIT` waits and `log_send_queue_size` / `redo_queue_size`.

**Code**
```sql
SELECT DB_NAME(database_id) db, synchronization_state_desc, log_send_queue_size, redo_queue_size
FROM sys.dm_hadr_database_replica_states;
ALTER AVAILABILITY GROUP [AG1] MODIFY REPLICA ON N'SQL02' WITH (AVAILABILITY_MODE = ASYNCHRONOUS_COMMIT);
```

### 12.4 [Hard] After an AG failover the application reports errors for 30 seconds then recovers. What do you do about it in the C# client?
**Answer**
Connect through the **AG listener** with `MultiSubnetFailover=True`, and implement transient-fault retry logic (EF `EnableRetryOnFailure`/Polly) for connection errors and read-only/failover errors. Make operations idempotent, because in-flight commits at the moment of failover have an unknown outcome.

**Explanation**
Existing pooled connections die; SqlClient discards broken ones on use. Unknown commit outcome means a retry can double-apply unless you use idempotency keys or unique constraints. Also set sensible `Connect Timeout` and `ConnectRetryCount/Interval` (connection resiliency retries idle connection breaks).

**Gotchas / follow-ups**
- `ApplicationIntent=ReadOnly` routes only if read-only routing is configured.
- Stale reads from async secondaries.
- Logins/jobs not on the new primary after failover.

**Code**
```sql
-- Idempotency guard so a retried command isn't applied twice
CREATE UNIQUE INDEX UX_Payment_RequestId ON dbo.Payment(RequestId);
INSERT dbo.Payment(RequestId, Amount) VALUES (@RequestId, @Amount);   -- dup => error 2601/2627, treat as success
```

### 12.5 [Tricky] Which of these gives you protection against an accidental `DROP TABLE` or `DELETE` without a WHERE: AG, FCI, log shipping, replication?
**Answer**
AG, FCI, and replication propagate the damage almost immediately (replication may apply DELETE to subscribers). Only **log shipping with a restore delay** (e.g., 1-4 hours) or point-in-time restore from backups/PITR can recover it.

**Explanation**
Log shipping delay is the classic "oops buffer". Alternative recovery tools: PITR restore to a side database and copy back, temporal tables (system-versioned) for row-level history, DB snapshots (before risky changes), and separation of duties/permissions (`DENY DROP`, DDL triggers).

**Gotchas / follow-ups**
- Database snapshots (`CREATE DATABASE ... AS SNAPSHOT OF`) are a quick revert mechanism but not a backup.
- AG replicas have no built-in "delayed apply" setting like some other databases; use backups/log shipping delay instead.
- Temporal tables guard against modifications, not against dropping the temporal table itself unless versioning is off.

**Code**
```sql
CREATE DATABASE MyDb_Snap ON (NAME = MyDb, FILENAME = 'D:\snap\MyDb_Snap.ss') AS SNAPSHOT OF MyDb;
-- Revert: RESTORE DATABASE MyDb FROM DATABASE_SNAPSHOT = 'MyDb_Snap';
```

**Interview traps**
- "AG is a backup replacement."
- Forgetting logins/jobs/linked servers on secondaries.
- Calling mirroring current (it's deprecated).
- Assuming FCI protects against storage failure.
- Claiming zero data loss with async commit.

**Cheat sheet**
- AG = database-level, separate storage, readable replicas.
- FCI = instance-level, shared storage.
- Log shipping = simple, delayed, manual failover.
- Replication = data distribution, not HA.
- Sync = RPO 0 + latency; async = DR.

---

## Topic 13: Security

**Study notes**

- **Login** (server-level identity: Windows/Entra, SQL auth) vs **user** (database-level principal mapped to a login). **Contained databases** (`CONTAINMENT = PARTIAL`; Azure SQL by default) let a user authenticate directly against the DB without a server login, making DBs portable. Orphaned users (user SID not matching a login after restore) are fixed with `ALTER USER ... WITH LOGIN = ...`.
- **Roles** (fixed: `db_datareader`, `db_datawriter`, `db_owner`...; custom roles) and **schemas** as a permission boundary (`GRANT EXECUTE ON SCHEMA::app TO role`). **Ownership chaining** lets a proc access tables with only EXECUTE granted. **Least privilege**: app logins never `sysadmin`/`db_owner`; separate migration identity from runtime identity.
- **TDE** (Transparent Data Encryption): encrypts data/log files and backups **at rest**; no app changes; DEK protected by a certificate in `master` (or key in Azure Key Vault). **Back up the certificate** or you can't restore elsewhere. Does not protect data in transit or from privileged users who can query.
- **Always Encrypted**: column-level, encryption/decryption happens in the **client driver**; server (and DBAs) never see plaintext or keys. *Deterministic* allows equality/joins/group by; *randomized* is stronger but no comparison (secure enclaves extend this).
- **Dynamic Data Masking (DDM)**: masks output for non-privileged users (`MASKED WITH (FUNCTION = 'email()')`); `UNMASK` permission shows real data. It's **not** a security boundary (inference via WHERE clauses is possible).
- **Row-Level Security (RLS)**: security policy with an inline table-valued predicate function; **FILTER** (hide rows on read) and **BLOCK** (prevent writes) predicates. Common with `SESSION_CONTEXT`.
- **Auditing**: SQL Server Audit (server audit + audit specifications to file/event log), Azure SQL Auditing (to storage/Log Analytics/Event Hubs).
- **SQL injection** defenses: parameterization (`sp_executesql`, `SqlParameter`, EF Core LINQ/`FromSqlInterpolated`), `QUOTENAME` for identifiers, allow-lists, least-privilege accounts, no string concatenation.

### 13.1 [Easy] What's the difference between a login and a database user?
**Answer**
A login authenticates you to the server instance; a database user is the identity inside one database that holds permissions, and is mapped to a login (or is a contained user with its own credentials). A login without a mapped user in a database cannot access it.

**Explanation**
After restoring to another server, users whose SIDs don't match any login are "orphaned". Prefer Windows/Entra authentication and groups; avoid shared SQL logins.

**Code**
```sql
CREATE LOGIN app_login WITH PASSWORD = 'Str0ng!Passw0rd#2026';
USE MyDb;
CREATE USER app_user FOR LOGIN app_login;
-- fix orphan
ALTER USER app_user WITH LOGIN = app_login;
```

### 13.2 [Follow-up] How do you implement least privilege for an application account?
**Answer**
Create a role with only the needed permissions - typically `EXECUTE` on a schema of stored procedures (and `SELECT` on specific views) - and add the app user to it. Use ownership chaining so it needs no direct table permissions, and keep DDL rights on a separate migration account.

**Explanation**
Avoid `db_owner`, `sysadmin`, and `db_datareader/writer` unless truly required. With EF Core direct table access is needed (SELECT/INSERT/UPDATE/DELETE on specific tables), so grant just those, never `db_owner`.

**Code**
```sql
CREATE SCHEMA app AUTHORIZATION dbo;
CREATE ROLE app_exec;
GRANT EXECUTE ON SCHEMA::app TO app_exec;
GRANT SELECT ON dbo.vw_OrderSummary TO app_exec;
ALTER ROLE app_exec ADD MEMBER app_user;
-- No direct table grants: procs in dbo reach tables through ownership chaining
```

### 13.3 [Medium] Explain TDE vs Always Encrypted vs Dynamic Data Masking. What does each protect against?
**Answer**
TDE protects data files/backups at rest from someone stealing the media; Always Encrypted protects sensitive columns even from DBAs/the server since decryption happens in the client; DDM only hides values in query results from lower-privileged users and is a convenience, not real security.

**Explanation**
TDE is transparent and cheap (CPU overhead small), but anyone who can query the DB sees plaintext. Always Encrypted has query limitations (equality only with deterministic, no LIKE without enclaves), needs driver support and key management (CMK in Key Vault/cert store, CEK in the DB encrypted by CMK).

**Code**
```sql
-- TDE
USE master;
CREATE MASTER KEY ENCRYPTION BY PASSWORD = 'Another#Str0ngPwd1';
CREATE CERTIFICATE TdeCert WITH SUBJECT = 'TDE cert';
BACKUP CERTIFICATE TdeCert TO FILE = 'D:\keys\TdeCert.cer'
  WITH PRIVATE KEY (FILE = 'D:\keys\TdeCert.pvk', ENCRYPTION BY PASSWORD = 'Pvk#Str0ngPwd1');
USE MyDb;
CREATE DATABASE ENCRYPTION KEY WITH ALGORITHM = AES_256 ENCRYPTION BY SERVER CERTIFICATE TdeCert;
ALTER DATABASE MyDb SET ENCRYPTION ON;

-- DDM
ALTER TABLE dbo.Customer ALTER COLUMN Email ADD MASKED WITH (FUNCTION = 'email()');
GRANT UNMASK TO support_lead;
```

### 13.4 [Medium] How does Row-Level Security work and how would you use it for multi-tenancy?
**Answer**
You write an inline TVF that returns a row when the current user/tenant may access a row, and bind it via `CREATE SECURITY POLICY` as a FILTER predicate (limits reads/updates/deletes) and BLOCK predicates (limits inserts/updates that would violate). The app sets `SESSION_CONTEXT('TenantId')` per connection and the policy compares it to the row's `TenantId`.

**Explanation**
The filter is applied by the engine on all queries, so forgotten `WHERE TenantId = ...` can't leak data. With connection pooling, set the session context on every connection checkout because `sp_reset_connection` clears it.

**Code**
```sql
CREATE SCHEMA sec;
GO
CREATE FUNCTION sec.fn_TenantFilter(@TenantId INT)
RETURNS TABLE WITH SCHEMABINDING AS
RETURN SELECT 1 AS ok WHERE @TenantId = CAST(SESSION_CONTEXT(N'TenantId') AS INT);
GO
CREATE SECURITY POLICY sec.TenantPolicy
  ADD FILTER PREDICATE sec.fn_TenantFilter(TenantId) ON dbo.Orders,
  ADD BLOCK  PREDICATE sec.fn_TenantFilter(TenantId) ON dbo.Orders AFTER INSERT
WITH (STATE = ON);
GO
EXEC sp_set_session_context @key = N'TenantId', @value = 42, @read_only = 1;
SELECT * FROM dbo.Orders;   -- only tenant 42 rows
```

### 13.5 [Hard] How do you prevent SQL injection in C# with ADO.NET, EF Core and dynamic SQL inside stored procedures?
**Answer**
Always parameterize: `SqlCommand.Parameters`, LINQ, or EF Core `FromSqlInterpolated` (which turns interpolated values into parameters; `FromSqlRaw` with concatenation is vulnerable). In T-SQL dynamic SQL use `sp_executesql` with parameters, and `QUOTENAME` for object names that must vary (allow-listed). Add least privilege so a successful injection has little blast radius.

**Explanation**
Parameters can't replace identifiers (table/column/`ORDER BY` direction), so validate against a whitelist. Stored procedures are not inherently safe if they concatenate strings. Defense in depth: WAF, error handling that doesn't leak details, no `xp_cmdshell`, no sysadmin.

**Gotchas / follow-ups**
- `FromSqlRaw($"... {x}")` interpolated into a `string` is *vulnerable*; `FromSql($"...")`/`FromSqlInterpolated` is safe.
- `EXEC('...' + @x)` is injectable; `EXEC sp_executesql @sql, N'@p INT', @p = @x` is safe.
- Second-order injection: stored data later concatenated.

**Code**
```sql
-- Vulnerable
DECLARE @sql NVARCHAR(MAX) = N'SELECT * FROM dbo.Orders WHERE Customer = ''' + @name + '''';
EXEC (@sql);

-- Safe: parameters + allow-listed identifier
DECLARE @col SYSNAME = CASE @sortBy WHEN 'Date' THEN N'OrderDate' WHEN 'Total' THEN N'Total' ELSE N'Id' END;
DECLARE @sql2 NVARCHAR(MAX) = N'SELECT * FROM dbo.Orders WHERE Customer = @n ORDER BY ' + QUOTENAME(@col);
EXEC sys.sp_executesql @sql2, N'@n NVARCHAR(100)', @n = @name;
```
```csharp
// EF Core: safe (parameterized)
var orders = await db.Orders.FromSql($"SELECT * FROM Orders WHERE Customer = {name}").ToListAsync();
```

### 13.6 [Tricky] Is Dynamic Data Masking a way to protect PII? What about Always Encrypted combined with EF Core and indexes?
**Answer**
DDM isn't: a user with SELECT can infer masked values with crafted WHERE clauses (e.g., `WHERE Salary > 50000`), and privileged users see everything. Use it as a presentation-layer convenience. Always Encrypted is real protection but breaks server-side logic: randomized columns can't be filtered/joined/sorted, deterministic ones allow only equality, and the app needs `Column Encryption Setting=Enabled` and key access.

**Explanation**
Also remember TDE does not protect data in transit (use TLS with `Encrypt=True`; `TrustServerCertificate=True` skips certificate validation - not for production). Combine: TLS + TDE + RLS + auditing, Always Encrypted only for the truly sensitive columns.

**Gotchas / follow-ups**
- Always Encrypted parameters need exact types matching column types (e.g., `DateTime2` precision, `nvarchar` length) - a common runtime error source.
- Don't forget to back up TDE certs/keys (lose cert = lose data).
- `db_owner` can `UNMASK`/see unencrypted data under DDM/TDE.

**Code**
```sql
-- DDM inference leak: a masked-value user can still binary-search
SELECT COUNT(*) FROM dbo.Employee WHERE Salary > 50000;   -- works even if Salary is masked
-- Audit setup (server audit to file)
CREATE SERVER AUDIT AppAudit TO FILE (FILEPATH = 'D:\audit\') WITH (ON_FAILURE = CONTINUE);
ALTER SERVER AUDIT AppAudit WITH (STATE = ON);
CREATE DATABASE AUDIT SPECIFICATION AppDbAudit FOR SERVER AUDIT AppAudit
  ADD (SELECT ON dbo.Customer BY public) WITH (STATE = ON);
```

**Interview traps**
- "TDE encrypts data in transit / hides data from DBAs" - no.
- Calling DDM encryption.
- Using `sa`/`db_owner` as the application account.
- `FromSqlRaw` with interpolated strings.
- Losing the TDE certificate.

**Cheat sheet**
- Login = server; user = database; contained DBs skip logins.
- Least privilege: role + schema EXECUTE + ownership chaining.
- TDE = at rest; Always Encrypted = client-side, DBA-proof; DDM = cosmetic.
- RLS = security policy + TVF predicate (+ SESSION_CONTEXT).
- Parameterize everything; `QUOTENAME` + allow-list for identifiers.

---

## Topic 14: Multi-tenancy, soft delete & large deletes

**Study notes**

**Multi-tenant models**

| Model | Isolation | Cost/Ops | Scale | Per-tenant restore | Notes |
|---|---|---|---|---|---|
| Shared DB, shared schema (`TenantId` column) | weakest (logical; use RLS/filters) | cheapest, one schema to migrate | thousands of tenants; noisy neighbour | hard | every table/index needs `TenantId` leading column |
| Schema per tenant | medium | schema migrations x N, plan cache bloat | hundreds | moderate | EF model caching complications |
| Database per tenant | strongest | most expensive; use Azure **elastic pools**; migrations x N | per-tenant sizing | easy (restore one DB) | best for compliance, large tenants |

**Soft delete** (`IsDeleted`/`DeletedAt`) keeps rows for recovery/audit at the cost of query filters (EF global filters), bloat, unique-constraint complications (use **filtered unique indexes** `WHERE IsDeleted = 0`) and FK semantics. **Archive** = move old data to another table/DB; **partitioning** (by date) allows `SWITCH` of a partition to a staging table in a metadata-only operation (`TRUNCATE TABLE ... WITH (PARTITIONS (...))` also possible).

**Large deletes**: a single huge `DELETE` generates huge log, holds locks (escalation to a table lock), blocks others, and has a long rollback. Delete in **batches** (say 1,000-5,000 rows; < 5,000 avoids escalation), commit each, optionally pause, take log backups (FULL) to keep log small.

### 14.1 [Easy] Describe the three multi-tenant database designs.
**Answer**
Shared schema: all tenants in the same tables, separated by a `TenantId` column. Schema-per-tenant: one schema per tenant in a shared DB. Database-per-tenant: separate database per tenant. Trade-off is isolation/customization vs cost and operational simplicity.

**Explanation**
Most SaaS starts with shared schema + RLS and moves big/regulated tenants to dedicated DBs (hybrid). Pick based on tenant count, data volume, compliance, per-tenant SLAs, and backup/restore needs.

**Code**
```sql
-- Shared schema: TenantId leads the keys/indexes
CREATE TABLE dbo.Orders(
  TenantId INT NOT NULL, OrderId BIGINT NOT NULL, Total MONEY NOT NULL,
  CONSTRAINT PK_Orders PRIMARY KEY (TenantId, OrderId));
CREATE INDEX IX_Orders_Tenant_Date ON dbo.Orders(TenantId, OrderId) INCLUDE (Total);
```

### 14.2 [Follow-up] What are the main risks of shared-schema multi-tenancy and how do you mitigate them?
**Answer**
Data leakage from a missed tenant filter, noisy neighbours (one tenant's heavy query hurts all), skewed statistics (parameter sniffing with very different tenant sizes), and hard per-tenant restore. Mitigate with RLS/EF global query filters, `TenantId` as the leading key column, resource governance/throttling, `OPTION (RECOMPILE)`/Query Store hints for skewed tenants, and moving big tenants to their own DB.

**Explanation**
Tenant skew makes one cached plan wrong for others - a classic gotcha. Per-tenant restore requires restoring to a side DB and extracting rows.

**Code**
```sql
-- Query Store hint for a skew-prone query (SQL 2022 / Azure SQL)
EXEC sys.sp_query_store_set_hints @query_id = 42, @query_hints = N'OPTION (RECOMPILE)';
```

### 14.3 [Medium] Soft delete vs archive vs hard delete - how do you pick, and what breaks with soft delete?
**Answer**
Soft delete for undo/audit with small volume; archive (move to history table/DB) for large, mostly-cold data; hard delete when legal/privacy requires erasure (GDPR). Soft delete breaks unique constraints, adds `WHERE IsDeleted = 0` everywhere (use EF global filter + filtered indexes), and grows tables forever unless purged.

**Explanation**
Alternative: **system-versioned temporal tables** keep history automatically (deletes preserved in history table) with a retention policy. Cascade relationships must be handled manually for soft deletes.

**Code**
```sql
CREATE TABLE dbo.Customer(Id INT IDENTITY PRIMARY KEY, Email NVARCHAR(200) NOT NULL, IsDeleted BIT NOT NULL DEFAULT 0);
CREATE UNIQUE INDEX UX_Customer_Email_Active ON dbo.Customer(Email) WHERE IsDeleted = 0;  -- filtered unique

-- Temporal alternative
ALTER TABLE dbo.Customer ADD
  ValidFrom DATETIME2 GENERATED ALWAYS AS ROW START HIDDEN NOT NULL DEFAULT SYSUTCDATETIME(),
  ValidTo   DATETIME2 GENERATED ALWAYS AS ROW END   HIDDEN NOT NULL DEFAULT CONVERT(DATETIME2,'9999-12-31 23:59:59.9999999'),
  PERIOD FOR SYSTEM_TIME (ValidFrom, ValidTo);
ALTER TABLE dbo.Customer SET (SYSTEM_VERSIONING = ON (HISTORY_TABLE = dbo.CustomerHistory));
```

### 14.4 [Medium] Delete 200 million old rows from a busy table without blocking or filling the log.
**Answer**
Delete in batches (e.g., `DELETE TOP (4000)`) in a loop, each its own short transaction, ordered by an indexed column so each batch seeks; pause between batches; in FULL recovery run frequent log backups; consider RCSI to keep readers unblocked. If deleting most of the table, copy the keepers to a new table and swap, or use partition switching/truncate.

**Explanation**
Batch size under ~5,000 avoids lock escalation to a table lock. An index on the delete predicate (e.g., `CreatedAt`) is essential; otherwise each batch scans. Disable triggers/cascade costs mindfully.

**Code**
```sql
DECLARE @rows INT = 1;
WHILE @rows > 0
BEGIN
  DELETE TOP (4000) FROM dbo.AuditLog
  WHERE CreatedAt < DATEADD(DAY, -365, SYSUTCDATETIME());
  SET @rows = @@ROWCOUNT;
  WAITFOR DELAY '00:00:00.100';     -- let log backups/readers breathe
END
```

### 14.5 [Hard] How does partition switching make purge/archive nearly instant, and what are the requirements?
**Answer**
`ALTER TABLE ... SWITCH PARTITION n TO staging` is a metadata-only change that moves the partition's pages to another table in the same filegroup, with no row-by-row work. Requirements: identical structure/indexes/constraints, target table empty (or empty partition), same filegroup, and a check constraint on the staging table matching the partition range (for the non-partitioned target). Then `TRUNCATE` or archive the staging table.

**Explanation**
Sliding-window: partition function by month; each month switch the oldest partition out, merge its boundary (`MERGE RANGE`), and add a new future boundary (`SPLIT RANGE`). Switching needs Sch-M on both tables (brief) - use `WAIT_AT_LOW_PRIORITY` (SQL 2014+) to avoid pile-ups. Partitioning is Standard edition-supported since SQL 2016 SP1.

**Gotchas / follow-ups**
- Partitioning is a manageability feature, not a guaranteed performance feature; queries need partition elimination.
- Unique indexes must include the partition key.
- Splitting a non-empty partition is expensive (data movement/log).

**Code**
```sql
CREATE PARTITION FUNCTION pf_Month (DATE) AS RANGE RIGHT FOR VALUES ('2026-01-01','2026-02-01','2026-03-01');
CREATE PARTITION SCHEME ps_Month AS PARTITION pf_Month ALL TO ([PRIMARY]);
CREATE TABLE dbo.Events(Id BIGINT NOT NULL, EventDate DATE NOT NULL, Payload NVARCHAR(200),
  CONSTRAINT PK_Events PRIMARY KEY (EventDate, Id)) ON ps_Month(EventDate);
-- (staging table dbo.Events_Stage must be identical, on the same filegroup)
ALTER TABLE dbo.Events SWITCH PARTITION 2 TO dbo.Events_Stage;
TRUNCATE TABLE dbo.Events_Stage;
-- Or purge directly (SQL 2016+)
TRUNCATE TABLE dbo.Events WITH (PARTITIONS (2));
```

### 14.6 [Tricky] A nightly batch delete keeps running for hours, the log is full, and the app gets blocked. What went wrong?
**Answer**
Likely a single large delete (or batch size too big => lock escalation to a table X lock) with no usable index so it scans, running in FULL recovery without log backups, and an enclosing transaction (or implicit outer transaction) that makes every batch part of one big transaction. Fixes: tight batches, supporting index, outer transaction removed, frequent log backups, and off-peak scheduling.

**Explanation**
A `WHILE` loop inside `BEGIN TRAN ... COMMIT` doesn't commit per batch - the log can't truncate until the outer commit. Also FK cascades and triggers multiply the cost, and indexed views/non-clustered indexes add write amplification per delete (drop/disable non-essential indexes for huge purges).

**Gotchas / follow-ups**
- ORDER BY with TOP in a DELETE isn't supported directly; use a CTE or subquery when specific order matters.
- Batched deletes under the `READ COMMITTED` default still hold X locks for each batch only; check `@@TRANCOUNT = 0` at loop start.
- Use `OUTPUT deleted.*` INTO an archive table to move rows in the same statement.

**Code**
```sql
-- Move-and-delete in one atomic statement per batch (archive pattern)
WHILE 1 = 1
BEGIN
  ;WITH old AS (
    SELECT TOP (2000) * FROM dbo.AuditLog WHERE CreatedAt < '2025-01-01' ORDER BY CreatedAt
  )
  DELETE FROM old OUTPUT deleted.* INTO dbo.AuditLog_Archive;
  IF @@ROWCOUNT = 0 BREAK;
END
```

**Interview traps**
- Believing a `WHILE` loop with batches releases the log when wrapped in an outer transaction.
- Soft delete + unique index without a filter.
- Choosing DB-per-tenant without a plan for N schema migrations.
- Expecting partitioning alone to speed up queries.
- Forgetting FK cascade/trigger costs on mass deletes.

**Cheat sheet**
- Shared schema = cheap, weakest isolation (RLS); DB per tenant = strongest, costly (elastic pools).
- Filtered unique index fixes soft-delete uniqueness.
- Batch deletes < 5,000 rows, per-batch commit, index on predicate.
- Partition `SWITCH`/`TRUNCATE PARTITIONS` for instant purge.
- Temporal tables for automatic history.

---

## Topic 15: Connection pooling & Azure SQL

**Study notes**

- **ADO.NET pooling** (SqlClient): physical connections are reused. A pool exists **per unique connection string** (and per identity with integrated security). Defaults: `Pooling=true`, `Min Pool Size=0`, **`Max Pool Size=100`**, `Connect Timeout=15`. `Close()`/`Dispose()` returns the connection to the pool (and the next use runs `sp_reset_connection`, which resets session state like temp tables, `SET` options, session context).
- **Max pool error**: *"Timeout expired. The timeout period elapsed prior to obtaining a connection from the pool. This may have occurred because all pooled connections were in use and max pool size was reached."* Causes: **connection leaks** (not disposing, e.g., returning a `DataReader` and forgetting the connection), slow queries/blocking holding connections, long transactions, sync-over-async thread starvation, or genuinely too much concurrency for pool size.
- **Azure SQL Database**: purchasing models **DTU** (bundled CPU/IO/memory: Basic/Standard/Premium) vs **vCore** (separate compute/storage; General Purpose, Business Critical, **Hyperscale**; **serverless** compute auto-pauses/scales). **Elastic pools** share a resource budget across many databases with unpredictable peaks. **Active geo-replication** (async readable secondaries, manual failover per DB) and **auto-failover groups** (group of DBs + stable read-write/read-only listener endpoints). Built-in backups with PITR (7-35 days) and long-term retention.
- **Throttling & transient faults**: resource limits (CPU, log rate, workers, sessions) cause errors like **10928/10929** (resource limit reached), **40501** (service busy), **40613** (database unavailable/failover), **40197**; plus network drops (e.g., 10053/10054/10060, 233, 64). They are transient: retry with exponential backoff.

### 15.1 [Easy] What is connection pooling and why use it?
**Answer**
Opening a physical SQL connection (TCP, TLS, login) is expensive, so ADO.NET keeps closed connections in a pool and hands them out again. Always open late and dispose early (`using`/`await using`) to return the connection to the pool.

**Explanation**
Pool identity = exact connection string text; tiny differences (e.g., application name, different users) create separate pools ("pool fragmentation"). Disposing doesn't close the physical connection; the pool does after idle time (~4-8 minutes).

**Code**
```sql
-- Server-side view of connections per app (look for leaks)
SELECT program_name, host_name, login_name, COUNT(*) AS conns,
       SUM(CASE WHEN status = 'sleeping' THEN 1 ELSE 0 END) AS sleeping
FROM sys.dm_exec_sessions
WHERE is_user_process = 1
GROUP BY program_name, host_name, login_name
ORDER BY conns DESC;
```
```csharp
await using var conn = new SqlConnection(cs);   // cs: "...;Max Pool Size=200;Connect Timeout=15;Application Name=OrdersApi"
await conn.OpenAsync();
```

### 15.2 [Follow-up] You get "max pool size was reached" in production. How do you diagnose and fix it?
**Answer**
First find whether connections are leaked or held too long: look at session counts and open transactions on the server, long-running/blocked queries, and code paths missing `using`. Fix leaks and slow/blocked queries; only then consider raising `Max Pool Size` (which just moves the problem) - and keep (pool size x app instances) within server limits.

**Explanation**
Server-side, many `sleeping` sessions with open transactions point to leaks/long transactions; many `running`/blocked point to slow queries. Also avoid `.Result`/`.Wait()` on async calls (thread-pool starvation delays disposal). Never hold a connection while calling external services or waiting on user input.

**Code**
```sql
SELECT s.session_id, s.status, s.open_transaction_count, s.last_request_end_time,
       DATEDIFF(SECOND, s.last_request_end_time, GETDATE()) AS idle_sec, s.program_name
FROM sys.dm_exec_sessions s
WHERE s.is_user_process = 1 AND s.program_name = 'OrdersApi'
ORDER BY idle_sec DESC;
```

### 15.3 [Medium] What does a pooled connection carry over between uses, and what traps does that create?
**Answer**
SqlClient calls `sp_reset_connection` on reuse, which resets session state (temp tables, `SET` options, session context, open transactions rolled back). The trap: relying on session state (temp tables, `SET` options, `SESSION_CONTEXT` tenant id) surviving between checkouts - it doesn't.

**Explanation**
A connection returned with an open transaction (never committed) gets rolled back on reset, but while it's out of the app's control, it still holds locks until reuse/reset - a leak with locks. Also, isolation level changes are reset, but rely on explicit `SET` per use if needed. With RLS via `SESSION_CONTEXT`, set it on every checkout (EF interceptor).

**Code**
```sql
EXEC sp_set_session_context @key = N'TenantId', @value = 42;
SELECT SESSION_CONTEXT(N'TenantId');   -- NULL again after pooled reuse/reset
```

### 15.4 [Medium] DTU vs vCore, and when to use elastic pools?
**Answer**
DTU is a blended measure of CPU/memory/IO, simple but less flexible; vCore lets you size compute and storage independently, supports serverless, Hyperscale, reserved capacity and Azure Hybrid Benefit. Elastic pools make sense when many databases have low average but unpredictable, non-overlapping peaks (e.g., per-tenant DBs) so they share one resource budget.

**Explanation**
Set per-database min/max limits in a pool to avoid noisy neighbours. Hyperscale scales storage up to 100 TB with fast restore. Serverless pauses when idle (first connection after pause has a delay - handle with retry/timeouts).

**Code**
```sql
SELECT * FROM sys.dm_db_resource_stats ORDER BY end_time DESC;   -- last hour, 15 s granularity (Azure SQL DB)
SELECT DATABASEPROPERTYEX(DB_NAME(),'Edition') ed, DATABASEPROPERTYEX(DB_NAME(),'ServiceObjective') so;
```

### 15.5 [Hard] How do you build resilience to transient faults and throttling in a C# app using Azure SQL?
**Answer**
Enable retry with exponential backoff and jitter: EF Core `EnableRetryOnFailure` (SqlServerRetryingExecutionStrategy, which wraps user transactions via `CreateExecutionStrategy`), or Polly for raw ADO.NET, plus SqlClient `ConnectRetryCount`/`ConnectRetryInterval` for idle-connection resiliency. Make commands idempotent, use sensible command timeouts, and address root causes of throttling (missing indexes, chatty queries, undersized tier).

**Explanation**
A retry after a failover during COMMIT has unknown outcome, so idempotency keys/unique constraints matter. Don't retry on permanent errors (e.g., syntax, permissions, constraint violations). Retrying a throttled workload without backoff worsens throttling (retry storm). Use failover group listener endpoints so failover needs no connection-string change.

**Gotchas / follow-ups**
- Execution strategy and user-initiated transactions: the whole block must be inside `strategy.ExecuteAsync`.
- Serverless auto-resume may take tens of seconds; a 15 s `Connect Timeout` might fail the first call - retry covers this.
- Log rate governance (`LOG_RATE_GOVERNOR` wait) throttles bulk inserts/deletes - batch and pause.

**Code**
```sql
SELECT wait_type, wait_time_ms FROM sys.dm_db_wait_stats
WHERE wait_type IN ('LOG_RATE_GOVERNOR','RESOURCE_SEMAPHORE','SOS_SCHEDULER_YIELD')
ORDER BY wait_time_ms DESC;   -- Azure SQL DB specific DMV
```
```csharp
services.AddDbContext<AppDb>(o => o.UseSqlServer(cs, sql =>
    sql.EnableRetryOnFailure(maxRetryCount: 5, maxRetryDelay: TimeSpan.FromSeconds(30), errorNumbersToAdd: null)));
```

### 15.6 [Tricky] How do active geo-replication and failover groups differ, and what happens to in-flight data on a forced failover?
**Answer**
Active geo-replication is per-database, async replication to up to four readable secondaries, with manual failover and no shared endpoint. Auto-failover groups manage a set of databases together with read-write and read-only listener endpoints and optional automatic failover policy. A forced failover can lose the most recent un-replicated commits (async), so RPO is typically in seconds, not zero.

**Explanation**
After failover, the old primary becomes secondary when reachable. Application should use the `<group>.database.windows.net` listener. Logins/users inside contained DBs (or synced logins) must exist on the secondary server for access to work.

**Gotchas / follow-ups**
- Planned (friendly) failover waits for sync - no data loss; forced does not.
- Secondary DB read-only: workloads using `ApplicationIntent=ReadOnly` hit the secondary and may see lag.
- Server-level objects (logins, firewall rules) aren't automatically replicated unless using contained users/DB-level firewall rules.

**Code**
```sql
SELECT partner_server, partner_database, replication_state_desc, replication_lag_sec
FROM sys.dm_geo_replication_link_status;   -- run on the primary (Azure SQL DB)
```

**Interview traps**
- Raising `Max Pool Size` instead of finding leaks.
- Assuming `Close()` kills the physical connection / session state persists.
- Retrying non-transient errors or without backoff.
- Treating geo-replication as zero data loss.
- Thinking `EnableRetryOnFailure` works with manual transactions without `ExecuteAsync`.

**Cheat sheet**
- Pool per connection string; default max 100; dispose to return.
- Max-pool timeout = leak or slow/blocked queries.
- DTU = bundled; vCore = flexible (serverless, Hyperscale); elastic pool = share budget.
- Transient errors: 40613, 40501, 10928/10929 -> retry with backoff/jitter.
- Failover groups = listener endpoints; async => possible data loss on forced failover.

---

## Topic 16: Migrations & monitoring

**Study notes**

**Database change management**
- **State-based** (declarative): **DACPAC** from an SSDT/SDK-style SQL project; `SqlPackage /Action:Publish` diffs the desired state against the target and generates the change script. Good for drift detection; risky for data-preserving transformations and renames (use refactor logs / pre-post deployment scripts).
- **Migration-based** (imperative, versioned): **EF Core migrations** (`dotnet ef migrations add`, `script --idempotent`, `migrations bundle`), **Flyway**, **DbUp**, Liquibase - ordered scripts recorded in a history table (`__EFMigrationsHistory`, `flyway_schema_history`) with checksums. Never edit an applied migration.
- **Zero-downtime / expand-contract (parallel change)**: (1) *Expand*: add new nullable column/table/index (backward compatible); (2) deploy code that writes both and reads old/new; (3) *backfill* in batches; (4) switch reads; (5) *Contract*: drop old column in a later release. Old and new app versions must both work against the intermediate schema.

**Monitoring**
- **Extended Events (XE)**: lightweight event tracing; replaces **SQL Trace/Profiler** (deprecated for the database engine; heavy overhead in production). `system_health` session is always on.
- **Query Store** (2016+; on by default for new DBs in SQL 2022 and Azure SQL): persists query text, plans, runtime stats per interval, wait stats; supports **plan forcing** and regression analysis.
- **Alerts**: SQL Agent alerts on severity 19-25 and errors 823/824/825/1205; Azure Monitor metric/log alerts (CPU, DTU, storage, deadlocks, connections); DMV-based health checks, `sys.dm_os_wait_stats`, Performance Monitor counters.

### 16.1 [Easy] Compare DACPAC and EF Core migrations.
**Answer**
DACPAC is state-based: you define the desired schema and SqlPackage computes the diff against the target. EF Core migrations are change-based: ordered C# migration classes (or generated SQL) applied in sequence and tracked in `__EFMigrationsHistory`. DACPAC suits DBA-owned schemas and drift detection; EF migrations suit code-first teams.

**Explanation**
For production, generate and review SQL scripts instead of applying migrations automatically on app startup (multiple instances can race, and the app account would need DDL rights). Flyway/DbUp are DB-first, script-based alternatives.

**Code**
```sql
-- What EF recorded
SELECT MigrationId, ProductVersion FROM dbo.__EFMigrationsHistory ORDER BY MigrationId;
-- CLI (shell): dotnet ef migrations script --idempotent -o deploy.sql
-- CLI (shell): SqlPackage /Action:Publish /SourceFile:Db.dacpac /TargetConnectionString:"..." /p:BlockOnPossibleDataLoss=True
```

### 16.2 [Follow-up] Rename a column with zero downtime while old and new app versions run side by side.
**Answer**
Don't rename in place. Expand: add the new column, add a trigger or dual-write in the app to keep both in sync, backfill old rows in batches, deploy the app reading the new column, then later drop the old column and trigger. A direct `sp_rename` breaks the running old version immediately.

**Explanation**
Each step is backward compatible. Backfill in small batches to avoid long locks/log growth. Add NOT NULL/constraints only after backfill, using `WITH CHECK` after validating (or `NOCHECK` then `CHECK` carefully).

**Code**
```sql
ALTER TABLE dbo.Customer ADD FullName NVARCHAR(200) NULL;              -- expand (metadata-only)

CREATE OR ALTER TRIGGER dbo.trg_Customer_SyncName ON dbo.Customer AFTER INSERT, UPDATE AS
BEGIN
  SET NOCOUNT ON;
  UPDATE c SET FullName = i.Name
  FROM dbo.Customer c JOIN inserted i ON i.Id = c.Id
  WHERE c.FullName IS NULL OR c.FullName <> i.Name;
END
GO
DECLARE @n INT = 1;
WHILE @n > 0                                                            -- backfill in batches
BEGIN
  UPDATE TOP (4000) dbo.Customer SET FullName = Name WHERE FullName IS NULL;
  SET @n = @@ROWCOUNT;
END
```

### 16.3 [Medium] Which schema changes are safe/online and which take long locks or rewrite the table?
**Answer**
Safe and quick: adding a nullable column, adding a column with a constant default (metadata-only on modern versions), creating indexes online (Enterprise/Azure SQL), adding NULL-able columns, adding a new table. Risky: changing a column's data type/length (often a size-of-data operation), adding a NOT NULL column with a volatile default, adding a validated FK/check on a big table (scans, holds Sch-M/locks), dropping/recreating clustered indexes offline, and any DDL when long transactions are open.

**Explanation**
DDL needs Sch-M and queues behind long readers, then blocks everyone behind it (see Topic 5). Use `SET LOCK_TIMEOUT`, `WAIT_AT_LOW_PRIORITY`, `RESUMABLE = ON` index operations, and run in maintenance windows or use expand/contract. Add constraints `WITH NOCHECK` then validate separately in controlled step (a non-trusted constraint can hurt plans until validated).

**Code**
```sql
ALTER TABLE dbo.Orders ADD Channel TINYINT NOT NULL CONSTRAINT DF_Orders_Channel DEFAULT (0);  -- constant default: metadata-only
CREATE INDEX IX_Orders_Cust ON dbo.Orders(CustomerId)
  WITH (ONLINE = ON, RESUMABLE = ON, MAX_DURATION = 60);     -- Enterprise / Azure SQL
ALTER TABLE dbo.Orders WITH NOCHECK ADD CONSTRAINT FK_Orders_Cust FOREIGN KEY (CustomerId) REFERENCES dbo.Customer(Id);
ALTER TABLE dbo.Orders WITH CHECK CHECK CONSTRAINT FK_Orders_Cust;   -- validate (re-trusts)
```

### 16.4 [Medium] What is Query Store and how do you use it to fix a regression after a deployment?
**Answer**
Query Store records queries, their plans and runtime stats over time inside the database. After a regression you compare plans/duration for the same query before vs after, and if a previous plan was better you can **force** it (or use automatic plan correction) while you fix the root cause (stats, parameter sniffing, missing index).

**Explanation**
Reports: Regressed Queries, Top Resource Consumers, Query Wait Statistics, Tracked Queries. Capture mode `AUTO` skips trivial queries; size and cleanup settings matter. Forced plans can fail if the plan becomes invalid (index dropped) - monitor `force_failure_count`.

**Code**
```sql
ALTER DATABASE MyDb SET QUERY_STORE = ON (OPERATION_MODE = READ_WRITE, QUERY_CAPTURE_MODE = AUTO, MAX_STORAGE_SIZE_MB = 1024);

SELECT TOP 10 q.query_id, qt.query_sql_text, SUM(rs.count_executions) execs,
       AVG(rs.avg_duration) avg_us
FROM sys.query_store_query q
JOIN sys.query_store_query_text qt ON qt.query_text_id = q.query_text_id
JOIN sys.query_store_plan p ON p.query_id = q.query_id
JOIN sys.query_store_runtime_stats rs ON rs.plan_id = p.plan_id
GROUP BY q.query_id, qt.query_sql_text
ORDER BY avg_us DESC;

EXEC sp_query_store_force_plan @query_id = 42, @plan_id = 17;
```

### 16.5 [Hard] Set up an Extended Events session to capture slow queries and explain why XE beats Profiler.
**Answer**
Create an event session on `sql_batch_completed`/`rpc_completed` with a duration predicate (microseconds) and write to an `event_file` target. XE is event-based and filtered inside the engine before collection, so overhead is far lower than Profiler/SQL Trace, and it exposes more events (waits, deadlocks, blocked processes).

**Explanation**
Use predicates (duration > 1 s), a small set of actions (`sql_text`, `database_name`, `client_app_name`), and rollover files. Avoid wide-open sessions and ring buffer for long collections (memory/truncation). Profiler is deprecated for the Database Engine; SQL Trace is also on the deprecated path.

**Gotchas / follow-ups**
- `duration` is in microseconds for XE.
- `event_file` path must be writable by the service account.
- Azure SQL DB supports database-scoped XE sessions (`ON DATABASE`) writing to blob storage.

**Code**
```sql
CREATE EVENT SESSION SlowQueries ON SERVER
ADD EVENT sqlserver.rpc_completed (
  ACTION (sqlserver.sql_text, sqlserver.database_name, sqlserver.client_app_name, sqlserver.session_id)
  WHERE duration > 1000000),                  -- > 1 s
ADD EVENT sqlserver.sql_batch_completed (
  ACTION (sqlserver.sql_text, sqlserver.database_name, sqlserver.client_app_name)
  WHERE duration > 1000000)
ADD TARGET package0.event_file (SET filename = N'SlowQueries.xel', max_file_size = 100, max_rollover_files = 5)
WITH (MAX_MEMORY = 4096 KB, EVENT_RETENTION_MODE = ALLOW_SINGLE_EVENT_LOSS, STARTUP_STATE = OFF);
ALTER EVENT SESSION SlowQueries ON SERVER STATE = START;
```

### 16.6 [Tricky] How do you alert on critical conditions and what are the common blind spots?
**Answer**
Use SQL Agent alerts/operators for severity 19-25 errors, 823/824/825 (IO/corruption), 1205 (deadlocks), failed jobs (backups, CHECKDB), AG health, and low disk/log space; in Azure use Azure Monitor alerts on CPU/DTU %, storage %, deadlocks, failed connections, and replication lag. Blind spots: backups that run but are never restore-tested, CHECKDB not scheduled, alerts pointing to an unmonitored mailbox, and monitoring only averages (missing short spikes).

**Explanation**
Add synthetic checks (a test query via the same path as the app) and track wait-stats deltas, not cumulative totals, since `sys.dm_os_wait_stats` resets only on restart. Baselines make alerts meaningful.

**Gotchas / follow-ups**
- Alerts on error numbers require the error to be logged (`sp_altermessage ... WITH_LOG` for some).
- Profiler on production can itself cause an outage.
- Query Store can go READ_ONLY when full - monitor `actual_state_desc`.

**Code**
```sql
EXEC msdb.dbo.sp_add_operator @name = N'DBA_Team', @email_address = N'dba@contoso.com';
EXEC msdb.dbo.sp_add_alert @name = N'Severity 19 Fatal Error', @message_id = 0, @severity = 19,
     @notification_message = N'Fatal error in resource';
EXEC msdb.dbo.sp_add_notification @alert_name = N'Severity 19 Fatal Error', @operator_name = N'DBA_Team', @notification_method = 1;

SELECT actual_state_desc, desired_state_desc, current_storage_size_mb, max_storage_size_mb
FROM sys.database_query_store_options;
```

**Interview traps**
- Letting the app run EF `Database.Migrate()` at startup across many instances in production.
- Renaming/dropping columns in one deploy.
- Running Profiler against production.
- Not validating NOCHECK constraints afterward (untrusted).
- Considering Query Store a replacement for XE (different purposes).

**Cheat sheet**
- DACPAC = state-based; EF/Flyway/DbUp = versioned scripts.
- Zero downtime = expand / backfill / contract.
- DDL needs Sch-M; use LOCK_TIMEOUT, online/resumable ops.
- Query Store = plan history + forcing; XE = lightweight event tracing.
- Alert on severity 19-25, 823-825, 1205, failed backups, AG lag, low disk.

---

*End of D04.*
