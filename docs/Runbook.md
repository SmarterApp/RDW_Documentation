# Runbook

**Intended Audience**: The runbook describes behavior and configuration options of the various applications in the [Reporting Data Warehouse](../README.md) (RDW). Operations, system administration, developers, and tier 3 support may find it useful.

### Table of Contents

* Application Overview
    * [Common Conventions](#common)
    * [Import Service](#import-service)
    * [Package Processor](#package-processor)
    * [Exam Processor](#exam-processor)
    * [Group Processor](#group-processor)
    * [Migrate Reporting](#migrate-reporting)
    * [Migrate OLAP](#migrate-olap)
    * [Task Service](#task-service)
    * [Reporting Web App](#reporting-webapp)
    * [Reporting Service](#reporting-service)
    * [Report Processor](#report-processor)
    * [Aggregate Service](#aggregate-service)
    * [Admin Service](#admin-service)
    * [PDF Generator](#pdf-generator)
* [Embargo](#embargo)
* Advanced Resources
    * [System Configuration](Runbook.SystemConfiguration.md)
    * [Import and Migrate](Runbook.ImportMigrate.md)
    * [Manual Data Modification](Runbook.ManualDataModifications.md)
    * [Bulk Delete Exams](Runbook.BulkDeleteExams.md)
    * [Language Support](Runbook.LanguageSupport.md)
    * [Data Specifications](Runbook.DataSpecifications.md)
    * [Archive Instance Database](Runbook.Archive.md)
    * [PII Audit Data](Runbook.Audit.md)
    * [MultiTenancy](Runbook.MultiTenancy.md)

<a name="common"></a>
## Common Service Conventions
All the applications in the RDW share common conventions and behaviors. This consistency across services makes maintaining the system less prone to error.

* **Dockerized**. They have been built to run in containers managed by an orchestration framework like Kubernetes.  
* **Java 8**.
* **Spring Boot**. 
    * [Configuration][1] and [Common Properties][2]. 
    * [Actuator end-points][3]. 
    * Logging. 

#### Configuration
As [Spring Boot][1] applications, the configuration settings for all applications come from:

* **Built-in defaults**. All applications have built-in defaults for everything except secrets (e.g. credentials).
* **Environment variables**. These may be used to override any default setting. However, they are used primarily to configure the environment profile and configuration server settings.
* **Command Line Options**. These may be used to override any default setting. In a container orchestration framework, these are seldom used.
* **Configuration server**. There is a central configuration server that all applications use to get environment-specific settings. Properties served up by the configuration server may be encrypted, protecting environment secrets. 

There are settings that all the applications use to bootstrap to the configuration server. These are generally set using environment variables in the orchestration framework, for example: 

```bash
CONFIG_SERVICE_ENABLED=true
CONFIG_SERVICE_LABEL=master
CONFIG_SERVICE_URL=http://configuration-service
```

Secrets in the configuration files may be encrypted to further protect them. The configuration service automatically decrypts these values when providing them to a service. To encrypt a value, pass it to the configuration service `encrypt` end-point. Using curl this looks like: `curl -X POST --data-urlencode "my+secret" http://localhost:8888/encrypt`. The encrypted value is then prefixed with `{cipher}` and wrapped in quotes, for example:
```yaml
spring:
  datasource:
    password: '{cipher}5e0e421375abd307f87f07a8ed4dab5ee9f105e8d4845ecf037f7ebdaeaf5709' 
``` 

The configuration server can combine multiple configuration files for a service. One use of this feature is to have common settings in one file, as shown in this [Annotated Configuration](../config/application.yml).

#### Resource Allocation
Dockerized applications need to respect resource allocations when deployed in a container orchestration environment.
This includes CPU allocation but is especially important for memory utilization. Most of the applications in RDW are
java applications. Their memory utilization can be broadly divided into two parts: heap and off-heap. The heap grows
and shrinks as the application does work, while the off-heap is relatively fixed overhead. For most RDW applications
the off-heap is 150MB-240MB. The heap varies and is described for each service.

The memory allocation is controlled by a few settings:
* Heap size. This tells java how much memory to give the application. Every application has a default value but that may not be optimal for a specific environment. This is controlled by setting an environment variable recognized by the docker command, `MAX_HEAP_SIZE`, which should look like the java setting, e.g. `-Xmx500m`.
* Container memory request/limit. This tells the orchestration framework how much memory to allow a pod. The framework will typically stop a pod that blows through that limit so it is important to coordinate this value with the heap size. A rule of thumb is to make the container limit a bit larger than the sum of the off-heap and max heap size. Obviously, if the framework is killing containers due to memory limits, this value should be increased.
* Java options. It is not recommended to use this except in extraordinary circumstances, but the docker image command recognizes the environment variable `JAVA_OPTS` and will add it to the java command line when starting the application.

Together, these can be used to fine-tune memory utilization. As an example the following (contrived) snippet gives an application extra startup memory (initial heap size), larger max heap size, and more container memory:
```yml
    spec:
      containers:
      - name: my-service
        resources:
          requests:
            memory: 1G
          limits:
            memory: 1G
        env:
        - name: MAX_HEAP_SIZE
          value: "-Xmx700m"
        - name: JAVA_OPTS
          value: "-Xms400m"
```
Without the environment variables it would use the default values (typically -Xms256m -Xmx384m).

In most orchestration environments, the ratio of memory to CPU is fixed. For example, most general purpose nodes in AWS have 4GB per CPU. The applications tend to be CPU constrained so it is okay to throw a little extra memory at them.


<a name="import-service"></a>
## Import Service
The import service is the REST end-point for submitting data to the system. It is responsible for archiving all imported data and then passing the work, via message queue, to payload processors. It uses OAuth2 for client validation. It is horizontally scalable for HA and overall throughput. A single process can handle a few dozen clients with an average latency of 200-300ms per request. 

![Import Service](import-service.png)

#### Configuration
The [Annotated Configuration](../config/rdw-ingest-import-service.yml) describes the properties and their effects.

#### Deployment Spec
The default max heap size is -Xmx384m which is more than enough and should be fine for any environment. The off-heap is about 200MB so the container should have a memory limit of about 600M.
The [Sample Kubernetes Spec](../deploy/import-service.yml) runs two replicas.


<a name="package-processor"></a>
## Package Processor
The package processor processes assessment packages, organizations and accommodations submitted to the system. It is responsible for parsing and validating the data before writing it to the data warehouse. Due to infrequent demand this processor has not been designed for high concurrency and only a single instance should be run.

![Package Processor](package-processor.png)

#### Configuration
The [Annotated Configuration](../config/rdw-ingest-package-processor.yml) describes the properties and their effects.

#### Deployment Spec
The default max heap size is -Xmx384m which should be fine for any environment. The off-heap is about 160MB so the container should have a memory limit of about 600M.
The [Sample Kubernetes Spec](../deploy/package-processor-service.yml) runs a single replica.


<a name="exam-processor"></a>
## Exam Processor
This processor handles parsing, validating and writing test results to the data warehouse. It also extracts student information from the test results, creating and updating them as necessary. It is horizontally scalable with each process handling 20-30 exams/sec.

![Exam Processor](exam-processor.png)

#### Configuration
The [Annotated Configuration](../config/rdw-ingest-exam-processor.yml) describes the properties and their effects.

#### Deployment Spec
The default max heap size is -Xmx384m which is more than enough and should be fine for any environment. The off-heap is about 180MB so the container should have a memory limit of about 600M.
The [Sample Kubernetes Spec](../deploy/exam-processor-service.yml) runs two replicas.


<a name="group-processor"></a>
## Group Processor
This processor handles parsing, validating and writing student group information to the data warehouse. It is horizontally scalable but a single process can handle a large volume of data (because most of the work is being done by the database itself), and group changes are relatively infrequent. Unlike other processors it must access the archive store directly (instead of getting its data from the message queue). It also uses Aurora's native ability to load directly from S3 to avoid copying data excessively.

![Group Processor](group-processor.png)

#### Configuration
The [Annotated Configuration](../config/rdw-ingest-group-processor.yml) describes the properties and their effects.

#### Deployment Spec
The default max heap size is -Xmx384m which should be fine for any environment. The off-heap is about 170MB so the container should have a memory limit of about 600M.
The [Sample Kubernetes Spec](../deploy/group-processor-service.yml) runs a single replica.


<a name="migrate-reporting"></a>
## Migrate Reporting
The migrate reporting service moves data from the warehouse to the reporting database. The service is not built to be horizontally scalable. Having more than one migrate reporting process will result in unpredictable behavior. 

Data is migrated based on import status (PROCESSED) and created/updated timestamps. The migration process is scheduled to run periodically. In each period, data is processed in small batches until there is no full batch remaining. The migration occurs in two steps, first from warehouse to staging, then from staging to reporting (the staging tables exist in the reporting database).

![Migrate Reporting](migrate-reporting.png)

The migrate service is controlled by two conditions: the user-controlled run state and the system-generated enabled state. The status end-points can be used to see the current status and pause/resume the service. Please refer to [Troubleshooting Migrate](Troubleshooting.md#migrate) for more details.

#### Configuration
The [Annotated Configuration](../config/rdw-ingest-migrate-reporting.yml) describes the properties and their effects.
 
#### Deployment Spec
The default max heap size is -Xmx512m because this service requires more memory. If the system is configured for a larger batch size (e.g. 4000 instead of 2000), the max heap size may have to be increased. The off-heap is about 160MB so the container should have a memory limit of at least 800M.
The [Sample Kubernetes Spec](../deploy/migrate-reporting-service.yml) runs a single replica with a larger memory limit.


<a name="migrate-olap"></a>
## Migrate OLAP
The migrate OLAP service is responsible for migrating data from the data warehouse to the aggregate reporting OLAP data store. The service is not built to be horizontally scalable. Having more than one migrate olap process will result in unpredictable behavior.

Data is migrated based on import status (PROCESSED) and created/updated timestamps. The migration process is scheduled to run daily. Each time data is processed in large batches until there is no full batch remaining. The migration occurs in two steps, first from warehouse to staging, then from staging to olap (the staging tables exist in the olap database). The migration tables exist in a separate, non-OLAP database.

![Migrate OLAP](migrate-olap.png)

The migrate service is controlled by two conditions: the user-controlled run state and the system-generated enabled state. The status end-points can be used to see the current status and pause/resume the service. Please refer to [Troubleshooting Migrate](Troubleshooting.md#migrate) for more details.

#### Configuration
The [Annotated Configuration](../config/rdw-ingest-migrate-olap.yml) describes the properties and their effects.
 
#### Deployment Spec
The default max heap size is -Xmx384m which should be fine for any environment (unlike migrate-reporting, the migrate-olap service offloads more of the work to the database). The off-heap is about 170MB so the container should have a memory limit of about 600M.
The [Sample Kubernetes Spec](../deploy/migrate-olap-service.yml) runs a single replica.


<a name="taks-service"></a>
## Task Service
This service is responsible for executing scheduled tasks. Currently this includes:
* Synchronizing organization data from ART (daily).
* Generating an import reconciliation report (daily).
* Resubmitting unprocessed test results (daily).

Only a single instance should be run since the task execution uses a simple, uncoordinated, time-based strategy.

![Task Service](task-service.png)

#### Configuration
The [Annotated Configuration](../config/rdw-ingest-task-service.yml) describes the properties and their effects.

#### Deployment Spec
The default max heap size is -Xmx384m which should be fine for any environment. The off-heap is about 160MB so the container should have a memory limit of about 600M.
The [Sample Kubernetes Spec](../deploy/task-service.yml) runs a single replica.


<a name="reporting-webapp"></a>
## Reporting Web App
This is the main reporting web application used by customers. It provides the UI experience for all users, including admin functionality. It is horizontally scalable with each process handling about 2000 concurrent users (this deployment is expected to have up to 15000 concurrent users).

The reporting web app is a UI-only application that handles some security (SSO redirects) and the presentation of data. All the "heavy lifting" of querying data is handled by the API services.

![Reporting Web App](reporting-webapp.png)

#### Configuration
The [Annotated Configuration](../config/rdw-reporting-webapp.yml) describes the properties and their effects.

#### Deployment Spec
The default max heap size is -Xmx768m which should be fine for any environment. The off-heap is about 240MB so the container should have a memory limit of about 1200M.
The [Sample Kubernetes Spec](../deploy/reporting-webapp.yml) runs four replicas with a higher memory limit.


<a name="reporting-service"></a>
## Reporting Service
This service provides the back-end API for reports against the reporting data mart, i.e. individual test results. It is horizontally scalable and many instances should be run to deal with report requests from the webapp.

![Reporting Service](reporting-service.png)

#### Configuration
The [Annotated Configuration](../config/rdw-reporting-service.yml) describes the properties and their effects.

#### Deployment Spec
The default max heap size is -Xmx384m and should be increased in all but the smallest environments. The off-heap is about 200MB so the container should have a memory limit of at least 600M.
The [Sample Kubernetes Spec](../deploy/reporting-service.yml) runs a single replica with increased heap size and memory limit.


<a name="report-processor"></a>
## Report Processor
This processor generates reports. It is horizontally scalable and many instances should be run to deal with reporting load.

![Report Processor](report-processor.png)

#### Configuration
The [Annotated Configuration](../config/rdw-reporting-report-processor.yml) describes the properties and their effects.

#### Deployment Spec
The default max heap size is -Xmx384m and should be increased in all but the smallest environments (very large, multi-student PDF reports may cause memory pressure). The off-heap overhead is about 240MB so the container should have a memory limit of about 650M.
The [Sample Kubernetes Spec](../deploy/report-processor-service.yml) runs two replicas with increased heap size and memory limit.


<a name="aggregate-service"></a>
## Aggregate Service
This service provides the back-end API for reports against the OLAP data store, i.e. aggregate reports. It is horizontally scalable; however, the OLAP data store is typically the bottleneck and running many instances of this service may not result in better overall throughput.

![Aggregate Service](aggregate-service.png)

#### Configuration
The [Annotated Configuration](../config/rdw-reporting-aggregate-service.yml) describes the properties and their effects.

#### Deployment Spec
The default max heap size is -Xmx768m which should be fine for most environments, perhaps it could be lowered (no less than 600m) in small environments. The off-heap is about 240MB so the container should have a memory limit of at least 1G.
The [Sample Kubernetes Spec](../deploy/aggregate-service.yml) runs a single replica with default settings.


<a name="admin-service"></a>
## Admin Service
This service provides the back-end API for administrative functionality including management of student groups, instructional resource links, embargo settings. It is horizontally scalable, however administrative management is limited in scope and a single instance is usually sufficient.

![Admin Service](admin-service.png)

#### Configuration
The [Annotated Configuration](../config/rdw-reporting-admin-service.yml) describes the properties and their effects.

#### Deployment Spec
The default max heap size is -Xmx384m which should be fine for most environments. The off-heap overhead is about 200MB so the container should have a memory limit of about 600M.
The [Sample Kubernetes Spec](../deploy/admin-service.yml) runs a single replica.


<a name="pdf-generator"></a>
## PDF Generator
This application converts HTML to PDF. It is used by the report processor. It is horizontally scalable and many instances should be run to deal with reporting load. Note the PDF generator is not a Spring Boot application: it doesn't use the central configuration server, it doesn't have the same actuator end-points, and logging is different.

![PDF Generator](pdf-generator.png)

#### Configuration
There are no configuration options for the PDF generator.

#### Deployment Spec
This service is not java-based and does not support changing the heap size.
The [Sample Kubernetes Spec](../deploy/wkhtmltopdf-service.yml) runs four replicas.


## System Configuration

> This section has been moved to a dedicated document: [System Configuration](Runbook.SystemConfiguration.md). Please update any saved links.

## Embargo

Embargo is a feature where summative test results are held back until all the data is available and validated. An embargo affects the viewing of individual test results separately from aggregate report data. Lifting the embargo releases test results.

When individual test results are embargoed, those results will not be visible in the reporting UI to normal users. Similarly, when aggregate test results are embargoed, those reports will not be visible in the aggregate reporting UI to normal users. Embargo administrators will be able to see the results (and the UI will have a notice to that effect) regardless of embargo settings.

Embargo is set at the state and district levels. Districts may release test results while the state is still embargoed. However, once a state releases test results (i.e. lifts the embargo), all results for all districts are released, regardless of the district embargo settings.

Embargo settings are managed in the Admin UI. The functionality will be available if a user has embargo write permissions for the state or districts.

#### Import and Migrate
As discussed in the [Import and Migrate](Runbook.migrate.md) the embargo setting is migrated as part of the general ingest process. Although not recommended, it is possible to manually modify embargo settings. For example, to lift the statewide embargo on individual test results for the school year 2018, then trigger the migration:
```sql
UPDATE state_embargo SET individual=0 WHERE school_year=2018;
INSERT INTO import (status, content, contentType, digest) VALUES (1, 6, 'lift embargo', 'lift embargo 2017-12-12');
```


---
[1]: https://docs.spring.io/spring-boot/docs/1.5.2.RELEASE/reference/htmlsingle/#boot-features-external-config
[2]: https://docs.spring.io/spring-boot/docs/1.5.2.RELEASE/reference/htmlsingle/#common-application-properties
[3]: https://docs.spring.io/spring-boot/docs/1.5.2.RELEASE/reference/htmlsingle/#production-ready-endpoints


			 
