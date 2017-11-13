# Runbook

**Intended Audience**: the runbook describes behavior and configuration options of the various applications in the Reporting Data Warehouse (RDW). Operations, system administration, and tier 3 support will find it useful.

1. [Common Conventions](#common)
1. [Import Service](#import-service)
1. [Package Processor](#package-processor)
1. [Exam Processor](#exam-processor)
1. [Group Processor](#group-processor)
1. [Migrate Reporting](#migrate-reporting)
1. [Migrate OLAP](#migrate-olap)
1. [Task Service](#task-service)

<a name="common"></a>
## Common Service Conventions
All the applications in the RDW share common conventions and behaviors. This consistency across services makes maintaining the system less prone to error.

* **Dockerized**. They have been built to run in containers managed by an orchestration framework like Kubernetes.  
* **Java 8**. (non-issue since JDK is part of the docker image?)
* **Spring Boot**. 
    * [Configuration][1] and [Common Properties][2]. 
    * [Actuator end-points][3]. 
    * Logging. 
    * what else? 
* 

#### Configuration
As [Spring Boot][1] applications, the configuration settings for all applications come from:

* **Built-in defaults**. All applications have built-in defaults for everything except secrets (e.g. credentials).
* **Environment variables**. These may be used to override any default setting. However, they are used primarily to configure the environment profile and configuration server settings.
* **Command Line Options**. These may be used to override any default setting. In a container orchestration framework, these are seldom used.
* **Configuration server**. There is a central configuration server that all applications use to get environment-specific settings. Properties served up by the configuration server may be encrypted, protecting environment secrets. 

There are settings for the configuration server that all the applications use. These are generally set using environment variables in the orchestration framwork, for example: 

```bash
CONFIG_SERVICE_ENABLED=true
CONFIG_SERVICE_LABEL=master
CONFIG_SERVICE_URL=http://configuration-service
```

TODO - document how to encrypt secrets and use them in configuration
TODO - ?document how `spring.profiles` can be used with configuration server?


<a name="import-service"></a>
## Import Service
The import service is the REST end-point for submitting data to the system. It is responsible for archiving all imported data and then passing the work, via message queue, to payload processors. It uses OAuth2 for client validation. It is horizontally scalable for HA and overall throughput. A single process can handle a few dozen clients with an average latency of 200-300ms per request. 

#### Configuration
The [Annotated Configuration](../config/rdw-ingest-import-service.yml) describes the properties and their effects.


<a name="package-processor"></a>
## Package Processor
The package processor processes assessment packages, organizations and accommodations submitted to the system. It is responsible for parsing and validating the data before writing it to the data warehouse. Due to infrequent demand this processor has not been designed for high concurrency and only a single instance should be run.

##### Properties
* `spring.datasource.url-server`
* `spring.datasource.username`
* `spring.datasource.password`
* `spring.rabbitmq.host`
* `spring.rabbitmq.username`
* `spring.rabbitmq.password`


<a name="exam-processor"></a>
## Exam Processor
This processor handles parsing, validating and writing test results to the data warehouse. It is horizontally scalable with each process handling 20-30 exams/sec.

##### Properties
* `spring.datasource.url-server`
* `spring.datasource.username`
* `spring.datasource.password`
* `spring.rabbitmq.host`
* `spring.rabbitmq.username`
* `spring.rabbitmq.password`


<a name="group-processor"></a>
## Group Processor
This processor handles parsing, validating and writing student group information to the data warehouse. It is horizontally scalable but a single process can handle a large volume of data, and group changes are relatively infrequent. Unlike other processors it must access the archive store directly (instead of getting its data from the message queue).

##### Properties
* `archive.root`
* `archive.cloud.aws.credentials.accessKey`
* `archive.cloud.aws.credentials.secretKey`
* `archive.cloud.aws.region.static`
* `spring.datasource.url-server`
* `spring.datasource.username`
* `spring.datasource.password`
* `spring.rabbitmq.host`
* `spring.rabbitmq.username`
* `spring.rabbitmq.password`


<a name="migrate-reporting"></a>
## Migrate Reporting
The migrate reporting service moves data from the warehouse to the reporting database. The service is not built to be horizontally scalable. Having more than one migrate reporting process will result in unpredictable behavior. 

The process is controlled by the `warehouse.import` table and the `reporting.migrate` table. The service pulls data in two steps, first from `warehouse` to `staging`, and then from `staging` to `reporting`. Data is migrated based on import status (PROCESSED) and created/updated timestamps. The migration process is scheduled to run periodically. In each period, data is processed in batches until there is no full batch remaining.  

##### Properties
* `migrate.batch.delay=60000`: time in milliseconds between migrate periods.
* `migrate.batch.size=2000`: a number of import ids migrated in one chunk/iteration. The batch size can be adjusted for performance; it is a trade off between size and overall throughput. Because the database is the limiting resource, the effect may not be large. As an example, in a particular staging environment adjusting the batch size from 1000 to 2000 increased overall migrate rate by 33% (from 33/s to 44/s).
* `spring.reporting_datasource.url-server`
* `spring.reporting_datasource.username`
* `spring.reporting_datasource.password`
* `spring.warehouse_datasource.url-server`
* `spring.warehouse_datasource.username`
* `spring.warehouse_datasource.password`

#### Maintenance Guidelines

The migrate service is controlled via the status end-point which is natively served on port 
This end-point is not publicly available and is accessed via deployment orchestration. For 
in kubernetes port-forwarding can be used.

##### Pausing the service
Pausing will temporarily suspend the migrate task which may be desired during the maintenance downtime.

```bash
$ curl -X POST http://migrate-reporting/pause
```
#####  Restarting the service
Restarting will resume a previously paused task.

```bash
$ curl -X POST http://migrate-reporting/resume
```

##### Check the current service status
```bash
$ curl http://migrate-reporting/status
{
  "statusText": "Warning",
  "statusRating": 2,
  "level": 0,
  "dateTime": "2017-06-13T15:10:46.925+0000",
  "migrate-reporting-job": {
    "statusText": "Warning",
    "statusRating": 2,
    "lifecycle": "paused,enabled"
  }
}
```

#### Troubleshooting Guide

#####  Disabled service 

The migrate reporting service will disable itself if there is an error during processing. You can see this by
getting the service status or analyzing the logs.

#####  Finding the cause of the failure
To research the cause of the service failure:

* check the status and message in the `reporting.migrate` table: 
```sql
mysql> select * from reporting.migrate order by id desc limit 1;
```
* review the service logs for error messages.

###### Failed Migrate
A migrate job may fail. In this case the status of the most recent migrate will be `-20` and there should be a message in the record. If the message does not provide enough information about the failure, check the batch table:

```sql
mysql> SELECT be.* FROM reporting.migrate m JOIN spring_batch_reporting.BATCH_JOB_EXECUTION be ON be.JOB_INSTANCE_ID = m.job_id WHERE m.id = 124
```
If you see any failed SQLs in the logs, you can manually analyze the data in the `staging` db. 
Data in the `staging` schema will be left at the state of the failure. You do not need to worry about cleaning this data, the system will do it automatically on the restart. 

It is important to resolve the underlying problem before you [Restart Migrations](#restart-migrations).

###### Zombie Migrate
A migrate job may not complete. Although this should not happen during normal operation, if the status of the most recent migrate remains at `10` for a long time, it may be a zombie. You may be able to glean some information by looking at the batch table as with [Failed Migrate](#failed-migrate). It is safe to simply abandon the migrate record and [Restart Migrations](#restart-migrations).

###### Blocked Migrate
A migrate job could be blocked by an import in the `warehouse`. The records are migrated in chunks of contiguous import ids. If the next chunk has at least one import that stays in the status 0, the migrate will be blocked at that record and will disable itself. The log file will emit a warning like:

```text
WARN  MigrateReportingJobRunner    : migrate job blocked by import record 9512156
```
Although this seldom happens during regular operation, it can occur if there is a problem with the message queue, or if a process fails unexpectedly. To fix it, resubmit the import record. 
The migrate process will automatically continue once the import record is adjusted.

##### Restart Migrations
Once the cause of the problem has been resolved the service can be enabled by adjusting the status of the most recent migrate task in the reporting database. Connect with your favorite SQL client using a write-enabled account and change the status to -10 (Abandoned) setting a message to explain.

```sql
mysql> select * from reporting.migrate order by id desc limit 1;
+------+--------+--------+-----------------+----------------+----------------------------+----------------------------+---------+
| id   | job_id | status | first_import_id | last_import_id | created                    | updated                    | message |
+------+--------+--------+-----------------+----------------+----------------------------+----------------------------+---------+
| 1443 |   1443 |    -20 |         1434005 |        1435004 | 2017-06-13 14:31:57.582819 | 2017-06-13 14:34:00.395498 | NULL    |
+------+--------+--------+-----------------+----------------+----------------------------+----------------------------+---------+
1 row in set (0.04 sec)

mysql> update reporting.migrate set status=-10, message='manually abandoned, fixed problem' where id=1443;
```

<a name="migrate-olap"></a>
## Migrate OLAP
The migrate OLAP service is responsible for migrating data from the data warehouse to the aggregate reporting OLAP database.


<a name="taks-service"></a>
## Task Service
This service is responsible for executing scheduled tasks. Currently this includes:* Synchronizing organization data from ART (daily).* Generating an import reconciliation report (daily).
* Resubmitting unprocessed data (daily).
Only a single instance should be run since the task execution uses a simple, uncoordinated, time-based strategy.

##### Properties
Properties for synchronizing organization data from ART:
* `task.update-organizations.cron=0 0 10 * * *`
* `task.update-organizations.state=CA`
* `task.update-organizations.art-client.districts-url`
* `task.update-organizations.art-client.groups-of-schools-url`
* `task.update-organizations.art-client.schools-url`
* `task.update-organizations.art-client.status-url`
* `task.update-organizations.art-client.oauth2.access-token-uri`
* `task.update-organizations.art-client.oauth2.client-id`
* `task.update-organizations.art-client.oauth2.client-secret`
* `task.update-organizations.art-client.oauth2.username`
* `task.update-organizations.art-client.oauth2.password`
* `task.update-organizations.import-service-client.organizations-imports-url`
* `task.update-organizations.import-service-client.status-url`
* `task.update-organizations.import-service-client.oauth2.access-token-uri`
* `task.update-organizations.import-service-client.oauth2.client-id`
* `task.update-organizations.import-service-client.oauth2.client-secret`
* `task.update-organizations.import-service-client.oauth2.username`
* `task.update-organizations.import-service-client.oauth2.password`

Properties for reconciliation report:
* `task.send-reconciliation-report.cron=0 0 12 * * *`
* `task.send-reconciliation-report.query`
* `task.send-reconciliation-report.senders` - TODO: depends on type of sender


---
[1]: https://docs.spring.io/spring-boot/docs/1.5.2.RELEASE/reference/htmlsingle/#boot-features-external-config
[2]: https://docs.spring.io/spring-boot/docs/1.5.2.RELEASE/reference/htmlsingle/#common-application-properties
[3]: https://docs.spring.io/spring-boot/docs/1.5.2.RELEASE/reference/htmlsingle/#production-ready-endpoints
 

			 