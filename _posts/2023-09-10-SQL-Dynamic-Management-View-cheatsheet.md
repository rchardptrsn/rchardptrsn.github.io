---
layout: post
author: Richard Peterson
tags: [sql, performance]
---
This is a cheatsheet that was created while studying for the [DP-300 Administering Microsoft Azure SQL Solutions certification.](https://learn.microsoft.com/en-us/certifications/exams/dp-300/)

> Dynamic Management Views (DMVs) Returns internal data about the state of the database or the instance. 
> - [What are Dynamic Management Views?](https://learn.microsoft.com/en-us/training/modules/explore-query-performance-optimization/4-describe-dynamic-management-views-functions?ns-enrollment-type=learningpath&ns-enrollment-id=learn.wwl.optimize-query-performance-sql-server)
> - [Monitor performance with DMVs](https://learn.microsoft.com/en-us/azure/azure-sql/database/monitoring-with-dmvs?view=azuresql) 


## Grant Permissions to view Performance DMVs
```sql
-- Server scoped objects
GRANT VIEW SERVER STATE TO MyUser
-- Database scoped objects
GRANT VIEW DATABASE STATE TO MyUser
-- database configurations that override server configurations
sys.database_scoped_configurations
```

## Resource contention and blocking
DMVs to collect info about page allocation specific to tempdb
- sys.dm_db_task_space_usage
- sys.dm_db_session_space_usage

Data that can only be captured from DMVs is data and transaction log file read/write latency as exposed in `sys.dm_os_volume_stats`

Identify IO wait problems or if you suspect blocking caused by tempdb database allocation contention
- **sys.dm_exec_requests** - each request that is currently being executed in the Azure SQL database
- **sys.dm_os_waiting_tasks** - wait queue of tasks waiting on some resource

#### Azure SQL only queries
Monitor cpu util associated with Azure SQL process for elastic pool by examining value of **avg_cpu_percent** column in **sys.elastic_pool_resource_stats** system catalog
- make sure average is continuously below **70%**
- select avg_cpu_percent, * from sys.elastic_pool_resource_stats

Collect CPU usage and storage data usage statistics for a five-day period using **sys.resource_stats** *ONLY AZURE SQL DATABASE*
```sql
SELECT *
FROM sys.resource_stats
WHERE database_name = 'MyDb' 
	AND START_TIME > DATEADD(day, -5, GETDATE())
ORDER BY START_TIME DESC;
```

## Query Performance
```sql
ALTER DATABASE SCOPED CONFIGURATION SET LAST_QUERY_PLAN_STATS = ON;
```
- option enables the collection of last query plan statistics in `sys.dm_exec_query_plan_stats`
	- equivalent to capturing actual query plan
For older versions of SQL Server without query store, you can use these views to return information about query plans
- sys.dm_exec_cached_plans
- sys.dm_exec_sql_text
- sys.dm_exec_query_plan
 New dynamic management function called `sys.dm_exec_query_plan_stats` which can show the last known actual query execution plan for a given handle.
	- To see the last actual query plan using this function, enable trace flag 2451 server-wide.
		- `DBCC TRACEON (2451, -1)`
	- OR scoped configuration `LAST_QUERY_PLAN_STATS`
- Query to get last execution plan for all cached queries
```sql
SELECT *
FROM sys.dm_exec_cached_plans AS cp
    CROSS APPLY sys.dm_exec_sql_text(plan_handle) AS st
    CROSS APPLY sys.dm_exec_query_plan_stats(plan_handle) AS qps; 
GO
```

Identify additional indexes most likely to improve query performance: *Azure SQL Only*  sys.dm_db_mising_index_group_stats**

Query to associate session ID with a windows thread id to monitor performance of the thread in Windows Performance Monitor
```sql
SELECT s.session_id, st.os_thread_id
FROM sys.dom_os_tasks AS S
INNER JOIN sys.dm_os_threads AS st
	ON s.worker_address = st.worker_address
WHERE st.session_id IS NOT NULL
ORDER BY st.session_id;
GO
```

## Monitor Propagation latency
Monitor propagation **latency** on a read only replica (think transfer speed): sys.dm_database_replica_states and columns redo_rate and redo_queue_size
- time in seconds that replica is behind: secondary_lag_seconds


DMV to monitor geo-replication lag and last replication time of the transactions committed on the primary and hardened to the transaction log on the secondary
```sql
SELECT
	link_guid
	, partner_server
	, last_replication
	, replication_lag_sec
FROM sys.dm_geo_replication_link_status;
```

Monitor open transactions awaiting commit or rollback:
```sql
SELECT tst.session_id, [database_name] = db_name(s.database_id)
    , tat.transaction_begin_time
    , transaction_duration_s = datediff(s, tat.transaction_begin_time, sysdatetime()) 
    , transaction_type = CASE tat.transaction_type  WHEN 1 THEN 'Read/write transaction'
        WHEN 2 THEN 'Read-only transaction'
        WHEN 3 THEN 'System transaction'
        WHEN 4 THEN 'Distributed transaction' END
    , input_buffer = ib.event_info, tat.transaction_uow     
    , transaction_state  = CASE tat.transaction_state    
        WHEN 0 THEN 'The transaction has not been completely initialized yet.'
        WHEN 1 THEN 'The transaction has been initialized but has not started.'
        WHEN 2 THEN 'The transaction is active - has not been committed or rolled back.'
        WHEN 3 THEN 'The transaction has ended. This is used for read-only transactions.'
        WHEN 4 THEN 'The commit process has been initiated on the distributed transaction.'
        WHEN 5 THEN 'The transaction is in a prepared state and waiting resolution.'
        WHEN 6 THEN 'The transaction has been committed.'
        WHEN 7 THEN 'The transaction is being rolled back.'
        WHEN 8 THEN 'The transaction has been rolled back.' END 
    , transaction_name = tat.name, request_status = r.status
    , tst.is_user_transaction, tst.is_local
    , session_open_transaction_count = tst.open_transaction_count  
    , s.host_name, s.program_name, s.client_interface_name, s.login_name, s.is_user_process
FROM sys.dm_tran_active_transactions tat 
INNER JOIN sys.dm_tran_session_transactions tst  on tat.transaction_id = tst.transaction_id
INNER JOIN sys.dm_exec_sessions s on s.session_id = tst.session_id 
LEFT OUTER JOIN sys.dm_exec_requests r on r.session_id = s.session_id
CROSS APPLY sys.dm_exec_input_buffer(s.session_id, null) AS ib
ORDER BY tat.transaction_begin_time DESC;
```

Monitoring solutions to identify blocking sessions
- Extended events
- Dynamic management views
    - `sys.dm_tran_active_transactions`
    - `sys.dm_exec_sessions`
    - `sys.dm_exec_requests`
    - `sys.dm_os_wait_stats`
- SQL Insights