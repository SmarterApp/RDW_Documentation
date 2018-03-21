## Monitoring

**NOTE: please avoid putting environment-specific details _especially secrets and sensitive information_ in this document.**

**Intended Audience**: this document provides information for monitoring the Reporting Data Warehouse. Operations and system administrators will find it useful.

Monitoring RDW applications includes monitoring:

* [Database](#database)
    * [Import Status](#import-status)
    * [Monitor Ingest Speed](#monitor-ingest-speed)
    * [Monitor Time-To-Warehouse](#monitor-time-to-warehouse) 
    * [Validate Migrate Data Integrity](#validate-migrate-data-integrity)
    * [Monitor Migrate Failures](#monitor-migrate-failures)
    * [Monitor Migrate Rate](#monitor-migrate-rate)
    * [System Use By Date](#system-use-by-date)
    * [Organization Queries](#organization-queries)
* [Logging](#logging)
    * [Log Level](#log-level)
    * [Log Collection](#log-collection)
    * [Log Messages](#log-messages)
* [Application Status](#application-status)

### Database 
There are a number of tables that provide useful information about the state of the system.

When working with the database, know that there are multiple databases used by the system. The `warehouse` database is the primary data store: all imported data resides there. The `reporting` database is the backing store for the individual test result system. The `reporting_olap` database is the backing store for the aggregate reports. In each of the monitoring sections, the database will be called out -- be sure to connect to the right one!

Some of these queries may be used to dump data for automated monitoring systems, reports, etc. With the `mysql` client a good way to output the data is to use the `--batch` flag which prints results using tab as the column separator, then pipe the output to a file. If you prefer CSV you can pipe it through sed to replace tabs. For example:
```bash
$ mysql -h host -u username -p --batch warehouse < myquery.sql > results.tsv
$ mysql -h host -u username -p --batch warehouse < myquery.sql | sed 's/\t/,/g' > results.csv
```

#### Import Status
As data is accepted into the system an import record is created. Once the data is processed the status of the import record is updated to reflect success or a number of different error conditions. Monitoring the import table will catch any such issues. A query against the `warehouse` that counts all failures:

```sql
SELECT s.name status,  i.count
FROM
  (SELECT status, count(*) count FROM import WHERE content = 1 AND status < 0   GROUP BY status) i
  JOIN import_status s ON s.id = i.status
UNION
SELECT '--------------', '-----'
UNION
SELECT 'TOTAL', count(*)
FROM import
WHERE content = 1 AND status < 0
```
A more refined query that shows similar information for just the last day:

```sql
SELECT
  count(*) count,
  i.status,
  s.name AS status_name,
  cast(i.updated AS DATE) date
FROM import i
   JOIN import_status s ON i.status = s.id
WHERE 
  i.updated >= (CURRENT_DATE - INTERVAL 1 DAY) and i.updated < CURRENT_DATE -- or a specific date '2017-07-04'
  AND status < 0
GROUP by i.status, status_name, cast(i.updated AS DATE);
```

Another query to show more information about the failures:

```sql
SELECT
  i.id,
  creator,
  created,
  updated,
  s.name AS status,
  message
FROM import i
  JOIN import_status s ON s.id = i.status
WHERE i.content = 1 AND i.status < 0 AND s.name IN
  ('BAD_DATA',
   'BAD_FORMAT',
   'INVALID',
   'UNAUTHORIZED',
   'UNKNOWN_ASMT',
   'UNKNOWN_SCHOOL')
ORDER BY created DESC
```
The above query can be further refined to keep only interesting statuses.

For any of the queries, a non-empty result set indicates that there is unprocessed data in the system. Refer to 
[Troubleshooting][1] to resolve issues.

#### Monitor Ingest Speed
A new ingest request is captured by the ACCEPTED status of the import. Once the data is loaded into the `warehouse` the status is updated accordingly. Each ingest is different and hence the processing time will vary, but in general it is expected to take less than a minute.

To monitor for slow imports:

```sql
SELECT count(*) FROM import WHERE status = 0 AND updated > (CURRENT_TIMESTAMP + INTERVAL 60 SECOND);
```
If there are slow imports please refer to [Troubleshooting][1] to resolve. Although not urgent, this will affect the timeliness of the reporting data.

#### Monitor Time-To-Warehouse
Test results include the completed-at timestamp. Using the import create time we can calculate the time it takes for the test delivery and scoring system to get the results to the `warehouse`.

```sql
SELECT
  CASE WHEN last_24_hours.id IS NOT NULL THEN timestampdiff(HOUR, completed_at, created) ELSE timestampdiff(DAY, completed_at, created) END AS delay,
  CASE WHEN last_24_hours.id IS NOT NULL THEN 'hour' ELSE 'day' END AS bucket,
  count(*) cnt
FROM
  (SELECT id FROM exam WHERE timestampdiff(HOUR, completed_at, created) < 24 AND deleted = 0 AND school_year=2018) AS last_24_hours
  RIGHT JOIN exam e ON e.id = last_24_hours.id
WHERE e.deleted = 0 and e.school_year=2018
GROUP BY delay, bucket
ORDER BY bucket DESC, delay;
```

#### Validate Migrate Data Integrity
There is a script that reports on data discrepancies between the data warehouse and the reporting data mart(s). Detailed instructions for running that scripts and interpreting the results can be found in [RDW_Schema](https://github.com/SmarterApp/RDW_Schema) under the `validation` folder.

#### Monitor Migrate Failures
The migrate process is managed by the “migrate-reporting” service. It records progress and status in the migrate table in the `reporting` database. For all possible migrate statuses please refer to `migrate_status` table:

```sql
select * from migrate_status;
+-----+-----------+
| id  | name      |
+-----+-----------+
| -10 | ABANDONED |
|  20 | COMPLETED |
| -20 | FAILED    |
|  10 | STARTED   |
+-----+-----------+
```

The service will suspend itself if there is a failure. To check for the failure, run:

```sql
SELECT * FROM migrate WHERE status = -20;
```
If there are failures refer to [Troubleshooting][1] to resolve. 

**This condition requires immediate attention since new test results will not be visible in the reporting application until it is resolved**.
 
#### Monitor Migrate Rate
The  “migrate-reporting” service continuously migrates newly imported data from `warehouse` to `reporting`. The data is moved in batches defined by the `migrate`'s `first_at` and `last_at` timestamps. Each batch is different and hence the processing time will vary, but in general it is expected to take less than a minute.

To establish an average speed of the migrate for a particular installation, check the processing speed of the successful migrates on any given day by querying against the `reporting` database:

```sql
SELECT timestampdiff(SECOND, created, updated) runtime_in_sec
FROM migrate
WHERE status = 20 AND
      created >= (CURRENT_DATE - INTERVAL 1 DAY) AND created < CURRENT_DATE; -- or a specific date '2017-07-04'
```

Or check the average processing time over time:

```sql
SELECT avg(timestampdiff(SECOND, created, updated)) avg_runtime_in_sec
FROM migrate
WHERE status = 20 AND created >= '2017-07-04' AND created < '2017-12-04'; -- substitute dates with your values
```

To monitor the top 5 slowest successful migrates on a day before any given day run the following:

```sql
SELECT timestampdiff(SECOND, created, updated) runtime_in_sec
 FROM migrate
 WHERE status = 20 AND
       created >= (CURRENT_DATE - INTERVAL 1 DAY) AND created < CURRENT_DATE
 ORDER BY runtime_in_sec DESC
 LIMIT 5;
```  

If migrates are taking longer than expected, or if the monitoring shows a consistent increase in run time over time the system is degrading. Refer to [Troubleshooting][1] for instructions on diagnosing the situation. Although not urgent, this will affect the timeliness of the reporting data.

#### System Use By Date

Management may find it useful to report on system use by date, showing the increasing adoption over time. This is done in the `warehouse` database where all data enters the system. First, create some date sequence views using the beginning of the overall testing window, `2017-09-07`, as the start:

```sql
CREATE VIEW digits AS SELECT 0 AS digit UNION ALL SELECT 1 UNION ALL SELECT 2 UNION ALL SELECT 3 UNION ALL SELECT 4 UNION ALL SELECT 5 UNION ALL SELECT 6 UNION ALL SELECT 7 UNION ALL SELECT 8 UNION ALL SELECT 9;
CREATE VIEW numbers AS SELECT ones.digit + tens.digit * 10 + hundreds.digit * 100 AS number FROM digits as ones, digits as tens, digits as hundreds;
CREATE VIEW dates AS SELECT SUBDATE(CURRENT_DATE(), number) AS date FROM numbers;
-- prod is now a collection of all dates from start of production to today 
CREATE VIEW prod as SELECT date FROM dates WHERE date BETWEEN '2017-09-07' AND CURRENT_DATE() ORDER BY date;
```

Now it is easy to do by-date queries:

```sql
-- exams updated by date
SELECT p.date, IFNULL(s.count, 0) count FROM prod p 
  LEFT JOIN (SELECT DATE(updated) date, COUNT(*) count FROM exam WHERE updated > '2017-09-07' and deleted = 0 GROUP BY DATE(updated)) s ON s.date=p.date
  ORDER BY p.date; 
  
-- cumulative unique schools (may be a bit slow)
SELECT p.date, count(DISTINCT sub.sid) FROM prod p 
  LEFT JOIN (SELECT DATE(e.updated) date, es.school_id sid FROM exam e JOIN exam_student es ON es.id = e.exam_student_id WHERE e.updated > '2017-09-07' and e.deleted = 0) sub
    ON sub.date <= p.date
  GROUP BY p.date
  ORDER BY p.date;
  
-- cumulative unique students (may be a bit slow)
SELECT p.date, count(DISTINCT sub.sid) FROM prod p 
  LEFT JOIN (SELECT DATE(e.updated) date, es.student_id sid FROM exam e JOIN exam_student es ON es.id = e.exam_student_id WHERE e.updated > '2017-09-07' and e.deleted = 0) sub
    ON sub.date <= p.date
  GROUP BY p.date
  ORDER BY p.date; 
```

#### Organization Queries
As part of the suite of SmarterBalanced applications, the RDW uses ART to get organization information. Part of maintenance of ART is knowing which organizations are being used and which aren't. Here are a couple queries against the `warehouse` that may help with that.

```sql
-- schools with counts of exams (deleted or not) 
SELECT s.natural_id AS school_id, concat('"', s.name, '"') AS school_name, count(es.id) AS exam_count FROM school s 
  LEFT JOIN exam_student es ON es.school_id = s.id
GROUP BY s.id
ORDER BY s.natural_id;

-- districts with counts of exams (deleted or not)
SELECT d.natural_id AS district_id, concat('"', d.name, '"') AS district_name, count(es.id) AS exam_count FROM district d
  JOIN school s ON s.district_id = d.id
  LEFT JOIN exam_student es ON es.school_id = s.id
GROUP BY d.id
ORDER BY d.natural_id;
```

### Logging

#### Log Level
The applications log messages at different log levels depending on the severity of the message (the default log level is `INFO`):

* `ERROR` - an error the application can't properly deal with; requires immediate attention
* `WARN` - a problem that the application handled but deserves attention
* `INFO` - messages about the normal operation of the application
* `DEBUG` - messages that provide additional information for diagnosing issues

Log messages are grouped in a package hierarchy by area of functionality. For example, the package for log messages coming from ingest processes is `org.opentestsystem.rdw.ingest.processor`, while the one for Spring Boot functionality is `org.springframework.boot`. The log level may be set at any level of granularity using that package hierarchy. This provides very precise control of the messages that are emitted. The log level is set in the configuration file.

> Note: in the log file, the logger package is often abbreviated with one letter per level for brevity. For example `org.opentestsystem.rdw.ingest.processor.ExamProcessorConfiguration` may be `o.o.r.i.p.ExamProcessorConfiguration`.

As an example, there is an `INFO` message emitted every 5 minutes by Spring configuration. Although useful during initial configuration, this message just fills the log. To disable it, we can set the log level for that component higher than `INFO`:

```yml
logging:
  level:
    # this outputs a couple INFO messages every 5 minutes, which just fills the log
    org.springframework.cloud.config.client.ConfigServicePropertySourceLocator: warn
```

For another example, suppose we suspect a problem with message delivery in the ingest system. To make the logging of all ingest processing more verbose, set the log level to `DEBUG`:

```yml
logging:
  level:
    # make ingest logging output more verbose
    org.opentestsystem.rdw.ingest.processor: debug
```

##### Logger Endpoints
Spring Boot applications expose logger controls via the actuator end-points. To get all the known loggers in the system:

```bash
$ curl http://localhost:8008/loggers
{
  "levels": [
    "OFF",
    "ERROR",
    "WARN",
    "INFO",
    "DEBUG",
    "TRACE"
  ],
  "loggers": {
    "ROOT": {
      "configuredLevel": "INFO",
      "effectiveLevel": "INFO"
    },
    "com": {
      "effectiveLevel": "INFO"
    },
    ...
    "org.opentestsystem": {
      "effectiveLevel": "INFO"
    },
    "org.opentestsystem.rdw": {
      "effectiveLevel": "INFO"
    },
    "org.opentestsystem.rdw.common": {
      "effectiveLevel": "INFO"
    },
    ...    
```

To get/set the setting for a specific logger:

```bash
$ curl http://localhost:8008/loggers/org.opentestsystem.rdw.ingest.processor
{"effectiveLevel":"INFO"}

$ curl -X POST -H 'Content-Type: application/json' -d '{"configuredLevel": "DEBUG"}' http://localhost:8008/loggers/org.opentestsystem.rdw.ingest.processor

$ curl http://localhost:8008/loggers/org.opentestsystem.rdw.ingest.processor
{"configuredLevel":"DEBUG","effectiveLevel":"DEBUG"}
```

Note that setting the log level via the actuator end-point will be in effect only for the currently running process. Other instances of the process will not be affected, and restarting the process will revert back to the configured log levels.

#### Log Collection
The approach is to use fluentd on the nodes to tail logs and forward entries to a central service like Elasticsearch or Graylog. If you ask six people how to install and configure fluentd in a kubernetes cluster you'll get a dozen different answers. If you want to do a little reading:

* [Overview of Kubernetes logging](https://kubernetes.io/docs/concepts/cluster-administration/logging/)
* [Kubernetes logging with Elasticsearch/Kibana](https://kubernetes.io/docs/tasks/debug-application-cluster/logging-elasticsearch-kibana/)
* [Kubernetes logging with Fluentd](https://docs.fluentd.org/v0.12/articles/kubernetes-fluentd)
* [Logging to AWS Elasticsearch from Kubernetes](https://medium.com/@while1eq1/logging-to-aws-elasticsearch-service-from-kubernetes-855ad0959251)
* [Alternative to below using Fluentd to send logs to GELF/Graylog](https://github.com/xbernpa/fluentd-kubernetes-gelf/)

It is left as an exercise for the reader to set up the service that will catch the logs. When configuring the system
there must be a route from the nodes in the cluster to the service for whatever protocol is appropriate.

We have provided a Kubernetes spec for a fluentd DaemonSet that sends logs to a GELF endpoint, [fluentd-gelf.yml](../deploy/fluentd-gelf.yml).
Modify the spec to set the host and port then deploy it, `kubectl apply -f fluentd-gelf.yml`.

Once the logs are rolling in, create filters using the Kubernetes metadata.


#### Log Messages
These are log messages that can be used to trigger alerts about rare but expected issues. In general, all ERROR 
messages should trigger alerts, and messages at INFO, DEBUG, TRACE levels may be safely ignored. This section is
about WARN messages, and exceptions to the general handling. 

##### Exam Processing
Any failure while processing an exam (aka test result, aka TRT) will produce a log message. The text will vary depending on the error, but it will look like: 
```text
2017-09-24 03:58:04.124  WARN 1 --- [nio-8080-exec-9] o.o.r.i.p.ExamProcessorConfiguration  : failed with an ...
```
It is important to understand and resolve these failures since the test result will not appear in reports.

##### Migrate Disable
The migrate process will disable itself if it encounters problem. The log entry looks like this:
```text
2017-09-24 03:58:04.124  WARN 1 --- [nio-8080-exec-9] o.o.r.m.c.MigrateJobRunner  : disabling scheduled migrate jobs; manually fix last migrate status to enable
```
See [Troubleshooting](Troubleshooting.md#migrate) for details on the migrate service.

##### Update Organizations
The update-organizations task can encounter a number of problems. Because these failures are not immediately obvious monitoring these warnings is recommended:
```text
2017-09-24 03:58:04.124  WARN 1 --- [nio-8080-exec-9] o.o.r.i.t.r.ArtClient                  : Unexpected response status from ART:404
...
2017-09-24 03:58:04.124  WARN 1 --- [nio-8080-exec-9] o.o.r.i.t.s.i.RestImportServiceClient  : organization import not accepted, id: 123, status: BAD_DATA
```
If desired successful organization posts can be monitored by watching for:
```text
2017-09-24 03:58:04.124  INFO 1 --- [nio-8080-exec-9] o.o.r.i.t.s.i.RestImportServiceClient  : posted organizations import, id: 123, status: ACCEPTED
```

##### Reconciliation Report
The reconciliation report task can fail either completely or with partial delivery. Messages will look like:
```text
2017-09-24 03:58:04.124  WARN 1 --- [nio-8080-exec-9] o.o.r.i.t.s.i.DefaultReconciliationService  : Failed to send reconciliation report to ArchiveReportSender: ...
...
2017-09-24 03:58:04.124  WARN 1 --- [nio-8080-exec-9] o.o.r.i.t.ReconciliationReportConfiguration : Error sending reconciliation report ...
``` 

##### Duplicate Payload
The import service calculates a hash for any payload submitted and will ignore duplicates while logging a message at INFO level. Although this is harmless from a system point of view, excessive occurrences could indicate bad data flow, especially for EXAMs. Note that there is an update-organization task that runs daily and will usually produce this log message since the organization data in ART doesn't change regularly.  
```text
2017-09-24 03:58:04.124  INFO 1 --- [nio-8080-exec-9] o.o.r.i.service.DefaultImportService     : Ignoring EXAM payload with existing digest 498B7E8EB17D624C2C6C5829B9664576
...
2017-09-24 04:00:06.797  INFO 1 --- [nio-8080-exec-9] o.o.r.i.service.DefaultImportService     : Ignoring ORGANIZATION payload with existing digest EDD59FFE5432D9234177B2770B0A2FAB
```

#### Red Herrings
There are some log messages that look scary but are not. Because they are emitted by 3rd party libraries we won't clean them up but we will note some of them here. 

##### AmazonS3 Failure with Retry
The following info message may appear in the log followed by a stack trace. Although stack traces are normally an indicator of Bad Things, in this case the system will retry and get past the problem. The exact stack trace depends on the circumstances but it will be one of the following:
```text
2017-10-12 21:25:54.836  INFO 41018 --- [pool-2-thread-1] com.amazonaws.http.AmazonHttpClient      : Unable to execute HTTP request: The target server failed to respond

org.apache.http.NoHttpResponseException: The target server failed to respond ...
org.apache.http.conn.ConnectTimeoutException: Connect to ...
```

##### Load Balancer HTTP/HTTPS
The following info message may appear in the log followed by a stack trace. Although stack traces are normally an indicator of Bad Things, in this case it is not a problem. 
```text
2017-10-27 14:39:52.458  INFO 1 --- [nio-8080-exec-4] o.apache.coyote.http11.Http11Processor   : Error parsing HTTP request header
 Note: further occurrences of HTTP header parsing errors will be logged at DEBUG level.

java.lang.IllegalArgumentException: Invalid character found in the HTTP protocol
...
``` 

### Application Status
The applications present the Spring Boot Actuator endpoints as well as the SmarterBalanced diagnostic status endpoint. These are served on a non-standard port (8008) and are not exposed outside the VPC. 


[1]: ./Troubleshooting.md
