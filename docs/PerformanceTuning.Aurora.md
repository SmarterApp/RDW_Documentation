## Performance Tuning - Aurora (MySQL)

**Intended Audience**: this document has some instructions for finding performance issues with the Aurora (MySQL) database(s). Operations may find this useful.

### Slow query log

The slow query log is the lowest-overhead way to measure query execution time (https://dev.mysql.com/doc/refman/5.6/en/slow-query-log.html) 
- slow_query_log - enables the log
- long_query_time - configure the threshold to log the data
- log_output - controls the data logged data output location. The data could be configured to be logged into a db table or a file. When 'table' option is
chosen the data is logged into mysql.slow_log table.
- slow_query_log_file - file path, only applicable if the log_output is file

Observation: a server reboot wipes out the logged data.

### SHOW PROFILE command

It is a query profiling tool. It is disabled by default, but can be enabled for the duration of a session (connection) simply by setting a server variable:
(NOTE: This statement is deprecated as of MySQL 5.6.7 and will be removed in a future MySQL release. Use the Performance Schema instead)

```sql
mysql > SET profiling = 1;
```
After this, whenever you issue a statement to the server, it will measure the elapsed time and a few other types of data whenever the query changes from one execution state to another.
It records the profiling information in a temporary table and assigns the statement an integer identifier, starting with 1.

To view the profiled data:
```sql
mysql > SHOW PROFILES;
mysql > SELECT * FROM information_schema.profiling WHERE query_id = 255;
```
The above captures the query’s response time with higher precision, which is nice.
The profile allows you to follow through every step of the query’s execution and see how long it took. 

The SHOW PROFILES displays a list of the most recent statements sent to the server. 
The size of the list is controlled by the profiling_history_size session variable, which has a default value of 15. 
The maximum value is 100. Setting the value to 0 has the practical effect of disabling profiling. 

### Explain EXTENDED
MySQL doesn’t generate byte-code to execute a query, as many other database products do. 
Instead, the query execution plan is actually a tree of instructions that the query execution engine follows to produce the query results. 
The final plan contains enough information to reconstruct the original query. 
If you execute EXPLAIN EXTENDED on a query, followed by SHOW WARNINGS, you’ll see the reconstructed query.

### Performance Schema

The MySQL Performance Schema is a feature for monitoring MySQL Server execution at a low level: https://dev.mysql.com/doc/refman/5.6/en/performance-schema.html.

By default, the Performance Schema is disabled, and you have to turn it on and enable specific instrumentation points (“consumers”) that you wish to collect.
According to some sources, Performance Schema caused around an 8% to 11% overhead even when it was collecting no data, and 19% to 25% with all consumers enabled.

To create schema on MySQL 5.6 follow instructions here https://github.com/mysql/mysql-sys. Note that the last line of the SQL scripts requires super user permissions, that is not available on Aurora. 
But the setting is turned on by default: "SET @@sql_log_bin = @sql_log_bin;"

To enable the performance schema use, you need to configure the server with 
```
performance_schema=ON
```
On Aurora, it will require a server reboot.

Tables in the Performance Schema are in-memory tables that use no persistent on-disk storage. 
The contents are repopulated beginning at server startup and discarded at server shutdown.

There are many tables in the schema. 

For some examples of SQL against the Performance Schema tables, you can read some of the articles below:
- on Oracle engineer Mark Leith’s blog, such as http://www.markleith.co.uk/?p=471.
- http://www.markleith.co.uk/2012/07/04/mysql-performance-schema-statement-digests/

#### Performance Schema Sample Queries
- A high level overview of the statements, sorted by those queries with the highest latency:
```sql 
SELECT DIGEST_TEXT,
       IF(SUM_NO_GOOD_INDEX_USED > 0 OR SUM_NO_INDEX_USED > 0, '*', '') AS full_scan,
       COUNT_STAR AS exec_count,
       SUM_ERRORS AS err_count,
       SUM_WARNINGS AS warn_count,
       SEC_TO_TIME(SUM_TIMER_WAIT/1000000000000) AS exec_time_total,
       SEC_TO_TIME(MAX_TIMER_WAIT/1000000000000) AS exec_time_max,
       (AVG_TIMER_WAIT/1000000000) AS exec_time_avg_ms,
       SUM_ROWS_SENT AS rows_sent,
       ROUND(SUM_ROWS_SENT / COUNT_STAR) AS rows_sent_avg,
       SUM_ROWS_EXAMINED AS rows_scanned
FROM performance_schema.events_statements_summary_by_digest
ORDER BY SUM_TIMER_WAIT DESC LIMIT 5;
```

- List all normalized statements that use temporary tables ordered by number of on disk temporary tables 
descending first, then by the number of memory tables.
```sql
SELECT DIGEST_TEXT,
       COUNT_STAR AS exec_count,
       SUM_CREATED_TMP_TABLES AS memory_tmp_tables,
       SUM_CREATED_TMP_DISK_TABLES AS disk_tmp_tables,
       ROUND(SUM_CREATED_TMP_TABLES / COUNT_STAR) AS avg_tmp_tables_per_query,
       ROUND((SUM_CREATED_TMP_DISK_TABLES / SUM_CREATED_TMP_TABLES) * 100) AS tmp_tables_to_disk_pct
FROM performance_schema.events_statements_summary_by_digest
WHERE SUM_CREATED_TMP_TABLES > 0
ORDER BY SUM_CREATED_TMP_DISK_TABLES DESC, SUM_CREATED_TMP_TABLES DESC LIMIT 5;
```

- List all normalized statements that have done sorts, ordered by sort_merge_passes, sort_scans and sort_rows, 
all descending.
```sql
SELECT DIGEST_TEXT,
       COUNT_STAR AS exec_count,
       SUM_SORT_MERGE_PASSES AS sort_merge_passes,
       ROUND(SUM_SORT_MERGE_PASSES / COUNT_STAR) AS avg_sort_merges,
       SUM_SORT_SCAN AS sorts_using_scans,
       SUM_SORT_RANGE AS sort_using_range,
       SUM_SORT_ROWS AS rows_sorted,
       ROUND(SUM_SORT_ROWS / COUNT_STAR) AS avg_rows_sorted
FROM performance_schema.events_statements_summary_by_digest
WHERE SUM_SORT_ROWS > 0
ORDER BY SUM_SORT_MERGE_PASSES DESC, SUM_SORT_SCAN DESC, SUM_SORT_ROWS DESC LIMIT 5;
```

- List all normalized statements that use have done a full table scan ordered by the percentage of times a full 
scan was done, then by the number of times the statement executed.
```sql
SELECT DIGEST_TEXT,
       COUNT_STAR AS exec_count,
       SUM_NO_INDEX_USED AS no_index_used_count,
       SUM_NO_GOOD_INDEX_USED AS no_good_index_used_count,
       ROUND((SUM_NO_INDEX_USED / COUNT_STAR) * 100) no_index_used_pct
FROM performance_schema.events_statements_summary_by_digest
WHERE SUM_NO_INDEX_USED > 0
      OR SUM_NO_GOOD_INDEX_USED > 0
ORDER BY no_index_used_pct DESC, exec_count DESC LIMIT 5;
```