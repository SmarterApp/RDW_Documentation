## Performance Tuning - Redshift (PostgreSQL)

**Intended Audience**: this document has some instructions for finding performance issues with the Aurora (MySQL) database(s). Operations may find this useful.

You can find additional resources here:
- [awslabs](https://github.com/awslabs/amazon-redshift-utils)
- [AWS Big Data Blog, Amazon Redshift Engineering’s Advanced Table Design Playbook](https://aws.amazon.com/blogs/big-data/amazon-redshift-engineerings-advanced-table-design-playbook-preamble-prerequisites-and-prioritization/)

### Prerequisites
Admin queries refer to tables by their object ID (OID). You can get this OID in multiple ways, below is one of them:
```sql
SELECT oid, relname FROM pg_class WHERE relname='your_table_name_here';
```

#### Admin schema
Create admin schema in the Amazon Redshift cluster database you’re optimizing.

#####  V_EXTENDED_TABLE_INFO view
Add a new view, named v_extended_table_info, from [here](https://github.com/awslabs/amazon-redshift-utils/blob/master/src/AdminViews/v_extended_table_info.sql).
This is an improved version of the system view, which offers an extended output that makes schema and workload reviews much more efficient.
This view is used to estimate the table significance during the optimization process. 
The scan frequency and table size are the two metrics most relevant in that regards.
```sql
select *  from admin.v_extended_table_info where tablename = 'your_table_name_here';
```

#####  WLM admin views

The following views help with understanding of the behavior of queue processing in Amazon Redshift.
###### WLM_QUEUE_STATE_VW view
```sql
create view admin.WLM_QUEUE_STATE_VW as
select (config.service_class-5) as queue
, trim (class.condition) as description
, config.num_query_tasks as slots
, config.query_working_mem as mem
, config.max_execution_time as max_time
, config.user_group_wild_card as "user_*"
, config.query_group_wild_card as "query_*"
, state.num_queued_queries queued
, state.num_executing_queries executing
, state.num_executed_queries executed
from
STV_WLM_CLASSIFICATION_CONFIG class,
STV_WLM_SERVICE_CLASS_CONFIG config,
STV_WLM_SERVICE_CLASS_STATE state
where
class.action_service_class = config.service_class 
and class.action_service_class = state.service_class 
and config.service_class > 4
order by config.service_class;
```

###### Simple WLM_QUERY_STATE_VW View
```sql
create view admin.WLM_QUERY_STATE_VW as
select query, (service_class-5) as queue, slot_count, trim(wlm_start_time) as start_time, trim(state) as state, trim(queue_time) as queue_time, trim(exec_time) as exec_time
from stv_wlm_query_state;
```

### Troubleshooting by scenarios

##### Scenario: “There are no specific reports of slowness, but I want to ensure I’m getting the most out of my cluster by performing a review on all tables.”
Returns table information for all scanned tables
```sql 
SELECT * FROM admin.v_extended_table_info 
WHERE table_id IN (
  SELECT DISTINCT tbl FROM stl_scan WHERE type=2 
)ORDER BY SPLIT_PART("scans:rr:filt:sel:del",':',1)::int DESC,  size DESC; 
```

##### Scenario: “The query with ID 4941313 is slow.”
```sql 
 Returns table information for all tables scanned by query 4941313
SELECT * FROM admin.v_extended_table_info 
WHERE table_id IN (
  SELECT DISTINCT tbl FROM stl_scan WHERE type=2 AND query = 4941313
) ORDER BY SPLIT_PART("scans:rr:filt:sel:del",':',1)::int DESC,  size DESC; 
```

##### Explain plan alerts

```sql 
SELECT
  trim(s.perm_table_name)                                                                                                                                         AS table,
  (sum(abs(datediff(seconds, coalesce(b.starttime, d.starttime, s.starttime), CASE WHEN coalesce(b.endtime, d.endtime, s.endtime) > coalesce(b.starttime, d.starttime, s.starttime)
    THEN coalesce(b.endtime, d.endtime, s.endtime)
                                                                              ELSE coalesce(b.starttime, d.starttime, s.starttime) END))) / 60) :: NUMERIC(24, 0) AS minutes,
  sum(coalesce(b.rows, d.rows, s.rows))                                                                                                                           AS rows,
  trim(split_part(l.event, ':', 1))                                                                                                                               AS event,
  substring(trim(l.solution), 1, 60)                                                                                                                              AS solution,
  max(l.query)                                                                                                                                                    AS sample_query,
  count(DISTINCT l.query)
FROM stl_alert_event_log AS l
  LEFT JOIN stl_scan AS s ON s.query = l.query AND s.slice = l.slice AND s.segment = l.segment
  LEFT JOIN stl_dist AS d ON d.query = l.query AND d.slice = l.slice AND d.segment = l.segment
  LEFT JOIN stl_bcast AS b ON b.query = l.query AND b.slice = l.slice AND b.segment = l.segment
WHERE l.userid > 1
      AND l.event_time >= dateadd(h, -1, localtimestamp)
-- and s.perm_table_name not like 'volt_tt%'
GROUP BY 1, 4, 5
ORDER BY 2 DESC, 6 DESC;
```

##### Top slowest queries
```sql
SELECT
  trim(database)                 AS DB,
  count(query)                   AS n_qry,
  max(substring(qrytext, 1, 80)) AS qrytext,
  min(run_seconds)               AS "min",
  max(run_seconds)               AS "max",
  avg(run_seconds)               AS "avg",
  sum(run_seconds)               AS total,
  max(query)                     AS max_query_id,
  max(starttime) :: DATE         AS last_run,
  aborted,
  event
FROM (
  SELECT
    userid,
    label,
    stl_query.query,
    trim(database)                                                                                                                                       AS database,
    trim(querytxt)                                                                                                                                       AS qrytext,
    md5(trim(querytxt))                                                                                                                                  AS qry_md5,
    starttime,
    endtime,
    datediff(seconds, starttime, endtime) :: NUMERIC(12, 2)                                                                                              AS run_seconds,
    aborted,
    decode(alrt.event, 'Very selective query filter', 'Filter', 'Scanned a large number of deleted rows', 'Deleted', 'Nested Loop Join in the query plan', 'Nested Loop', 'Distributed a large number of rows across the network',
           'Distributed', 'Broadcasted a large number of rows across the network', 'Broadcast', 'Missing query planner statistics', 'Stats', alrt.event) AS event
  FROM stl_query
    LEFT OUTER JOIN (SELECT
                       query,
                       trim(split_part(event, ':', 1)) AS event
                     FROM STL_ALERT_EVENT_LOG
                     WHERE event_time >= dateadd(day, -7, current_Date)
                     GROUP BY query, trim(split_part(event, ':', 1))) AS alrt ON alrt.query = stl_query.query
  WHERE userid <> 1
        AND (querytxt LIKE 'SELECT count(*)%')
        -- and database = ''
        AND starttime >= dateadd(m, -5, localtimestamp)
)
GROUP BY database, label, qry_md5, aborted, event
ORDER BY total DESC
LIMIT 50;
```

#####  Largest queue time
```sql
SELECT
  w.query,
  substring(q.querytxt, 1, 100)                      AS querytxt,
  w.queue_start_time,
  w.service_class                                    AS class,
  w.slot_count                                       AS slots,
  w.total_queue_time / 1000000                       AS queue_seconds,
  w.total_exec_time / 1000000                           exec_seconds,
  (w.total_queue_time + w.total_Exec_time) / 1000000 AS total_seconds
FROM stl_wlm_query w
  LEFT JOIN stl_query q
    ON q.query = w.query
       AND q.userid = w.userid
WHERE w.queue_start_Time >= dateadd(m, -5, localtimestamp)
      AND w.total_queue_Time > 0
      AND q.starttime >= dateadd(m, -5, localtimestamp)
      AND (querytxt LIKE 'SELECT count(*)%')
ORDER BY w.total_queue_time DESC
  , w.queue_start_time DESC
LIMIT 100;
```