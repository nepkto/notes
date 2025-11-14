# SQL Server Troubleshooting & Performance Toolkit

This README consolidates concepts, scripts, and explanations to help you monitor, diagnose, and tune SQL Server performance. It explains key output columns from sp_WhoIsActive, guides you in tuning heavy queries and indexes, and provides automation scripts for daily use.

---

## sp_WhoIsActive

**Purpose:**  
A robust stored procedure for monitoring SQL Server activity in real time. It provides detailed insight into active sessions, including blocking, waits, query plans, resource usage, and much more.  
Reference: [sp_WhoIsActive GitHub](https://github.com/amachanic/sp_whoisactive)

**How/When to Use:**  
- Routine monitoring, especially when you suspect server slowness or blocking.
- Troubleshooting blocking, deadlocks, or CPU/resource bottlenecks.
- Auditing, investigating SQL Agent jobs, or just keeping tabs on server health.

**Usage Example:**
```sql
EXEC sp_WhoIsActive;
EXEC sp_WhoIsActive @show_system_spids=1, @get_plans=1, @get_locks=1;
```
- Run manually or schedule for regular checks.

---

## Key Output Columns Explained

| Column                  | Meaning                                                                                                        | Typical Use/Interpretation                                                            |
|-------------------------|----------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------|
| `status`                | Session state: `runnable`, `suspended`, `sleeping`, etc.                                                      | `runnable`: Session is waiting for CPU; `suspended`: Waiting on resources; `sleeping`: idle |
| `wait_info`             | What the session is waiting for; resource type or `NULL` if not waiting.                                       | `NULL`: Not waiting; values like `LCK_M_...` for locks, `PAGEIOLATCH_...` for I/O waits   |
| `session_id`            | SQL Server internal session (SPID) number.                                                                    | Identify/query specific session                                                      |
| `tasks`                 | Number of tasks assigned to this session/request.                                                             | Usually 1, more for parallelism                                                      |
| `blocking_session_id`   | Session ID that's blocking this session, or `NULL` if no blocking.                                            | Key to finding blocking chains                                                       |
| `login_name`            | Who initiated the session.                                                                                    | Audit, track problematic logins                                                      |
| `CPU`                   | Total CPU time consumed (ms).                                                                                 | High values signal CPU-heavy queries/jobs                                            |
| `tempdb_allocations`    | tempdb objects allocated by this session/request.                                                             | Monitor tempdb pressure                                                              |
| `tempdb_current`        | Current tempdb usage.                                                                                         | Track tempdb consumption trends                                                      |
| `reads`, `writes`       | Logical reads/writes performed.                                                                               | High reads usually signal poorly indexed queries                                     |
| `context_switches`      | Number of times the thread switched context.                                                                  | Too many can indicate inefficiency                                                   |
| `physical_io`           | Physical IO operations performed.                                                                             | High physical_io means heavy disk access                                             |
| `physical_reads`        | Physical reads from disk.                                                                                     | Monitor IO bottlenecks                                                               |
| `used_memory`           | Memory consumed by the session/request.                                                                       | Useful for spotting memory hogs                                                      |
| `open_tran_count`       | Number of open transactions.                                                                                  | Many open transactions may mean uncommitted work or blocking risks                   |
| `percent_complete`      | Percentage done (for long-running queries like backup/restore).                                               | Spot long jobs and estimate time to finish                                           |
| `host_name`             | Client hostname.                                                                                              | Useful for user tracking, troubleshooting distributed queries/jobs                   |
| `database_name`         | Database of query context.                                                                                    | Spot database hotspots                                                               |
| `program_name`          | Application name (SSMS, SQL Agent, etc.).                                                                    | Track down problematic application sources                                           |
| `start_time`, `login_time`, `collection_time` | When the request/session started and when snapshot was collected.                                 | Track execution time, find long-running queries                                      |
| `additional_info`       | XML with query text, isolation levels, agent job info, etc.                                                   | Drill into command type, agent step, job GUID, plan handle, and more                 |

### Common Status Values

- **runnable:** Waiting for CPU (not blocked, just waiting for core).
- **suspended:** Waiting on a resource (check `wait_info` for what).
- **sleeping:** Idle, not actively running.
- **running:** Actively executing on the CPU.

### Common Wait Info

- **NULL:** Not waiting; usually means session is ready for CPU or running.
- **LCK_M_XXX, PAGEIOLATCH_XXX, CXPACKET, SOS_SCHEDULER_YIELD, WRITELOG** etc.: SQL Server internal wait types.  
  See [Wait Types documentation](https://learn.microsoft.com/en-us/sql/relational-databases/performance-monitoring/wait-types) for deep dive.

_Example: `(1x: 16ms)RUNNABLE` means the session has spent 16ms in the RUNNABLE state (waiting to be picked by CPU scheduler once)._

---

## Tuning Resource-Heavy Queries/Jobs

**How to Identify Them:**

- Use sp_WhoIsActive for real-time insight or DMVs for historical aggregates:
    ```sql
    SELECT TOP 10
        qs.creation_time,
        qs.last_execution_time,
        qs.execution_count,
        qs.total_worker_time / qs.execution_count AS avg_cpu_ms,
        qs.total_logical_reads / qs.execution_count AS avg_logical_reads,
        qs.total_elapsed_time / qs.execution_count AS avg_elapsed_ms,
        SUBSTRING(qt.text, (qs.statement_start_offset/2)+1,
            ((CASE qs.statement_end_offset
                WHEN -1 THEN DATALENGTH(qt.text)
                ELSE qs.statement_end_offset
              END - qs.statement_start_offset)/2)+1) AS query_text
    FROM sys.dm_exec_query_stats qs
    CROSS APPLY sys.dm_exec_sql_text(qs.sql_handle) qt
    ORDER BY avg_cpu_ms DESC, avg_logical_reads DESC, avg_elapsed_ms DESC;
    ```
- Look for high CPU, high reads, or long durations.

**How to Tune:**

- Get the actual execution plan; look for table scans, expensive joins, missing indexes.
- Avoid SELECT *, use WHERE clauses on indexed columns, and reduce row size.
- Use parameterized queries for plan re-use.
- Break up large jobs into smaller batches.

---

## Index Optimization & Fragmentation Detection

**Detect Fragmented Indexes:**
```sql
SELECT
    dbschemas.[name] AS 'Schema',
    dbtables.[name] AS 'Table',
    dbindexes.[name] AS 'Index',
    indexstats.avg_fragmentation_in_percent AS 'Fragmentation',
    indexstats.page_count
FROM sys.dm_db_index_physical_stats(DB_ID(), NULL, NULL, NULL, 'LIMITED') indexstats
INNER JOIN sys.tables dbtables ON dbtables.[object_id] = indexstats.[object_id]
INNER JOIN sys.schemas dbschemas ON dbtables.[schema_id] = dbschemas.[schema_id]
INNER JOIN sys.indexes dbindexes ON dbindexes.[object_id] = indexstats.[object_id]
AND indexstats.index_id = dbindexes.index_id
WHERE indexstats.database_id = DB_ID()
AND indexstats.page_count > 1000
AND indexstats.avg_fragmentation_in_percent > 20
ORDER BY indexstats.avg_fragmentation_in_percent DESC;
```
- Rebuild or reorganize indexes where fragmentation exceeds 20% (and page count > 1000).

**Optimize Index Usage:**
- Create nonclustered indexes on needed columns:
    ```sql
    CREATE NONCLUSTERED INDEX IX_Table_Column ON dbo.TableName(ColumnName);
    ```
- Remove unused indexes (low usage from `sys.dm_db_index_usage_stats`).
- Regularly rebuild/reorganize indexes to keep performance optimal.

---

## Example Workflow

1. Use sp_WhoIsActive to spot slow/heavy sessions.
2. Drill into query text and resource stats.
3. Grab and review actual execution plans; tune queries accordingly.
4. Review and optimize indexes, rebuild fragmented ones, add missing, drop unused.
5. Monitor resource trends, automate reports and maintenance jobs via SQL Agent.

---

## Real-World Troubleshooting Tips

- **Many `runnable` + `wait_info: NULL` sessions:** Check CPU usage, consider query/index tuning, or adding CPU.
- **Repeat offenders:** Analyze and refactor problem query/job logic.
- **Tempdb pressure:** Watch tempdb columns in sp_WhoIsActive during bulk inserts/updates.
- **Blocking chains:** Use `blocking_session_id` to identify and break blocks, then tune root causes.

---

## Helpful References

- [sp_WhoIsActive: GitHub](https://github.com/amachanic/sp_whoisactive)
- [SQL Server DMV Reference](https://learn.microsoft.com/en-us/sql/relational-databases/performance-monitoring-and-tuning)
- [SQL Server Wait Types](https://learn.microsoft.com/en-us/sql/relational-databases/performance-monitoring/wait-types)
- [Index Maintenance](https://learn.microsoft.com/en-us/sql/relational-databases/indexes/indexes-overview-database-engine)

---

**For help on automating reports, maintenance, or troubleshooting a particular job/query, just paste the job details here!**