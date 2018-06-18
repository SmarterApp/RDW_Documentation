## Troubleshooting RDW in the AWS + Kubernetes Environment

**NOTE: please avoid putting environment-specific details _especially secrets and sensitive information_ in this document.**

### Table of Contents

* [Kubernetes](#kubernetes)
    * [Jump Server](#jump-server)
    * [Working With Nodes](#working-with-nodes)
    * [Working With Pods](#working-with-pods)
* [Specific Issues](#specific-issues)
    * [Improperly configured schools](#state-schools)
    * [Migrate](#migrate)
    * [Reconciliation Report](#reconciliation-report)
    * [Unknown School](#unknown-school)
    * [Duplicate Schools](#duplicate-schools)
    * [Empty Student Report](#empty-report)
    * [Invalid Student Group](#invalid-group)
    * [Missing Student Group](#missing-student-group)
    * [Can't See Student Group](#missing-student-group-one-user)
    * [Unable To Admin District](#unable-to-admin-district)
    * [ART Organization Data](#art-organization-data)
    * [Item Viewer](#item-viewer)
* [Unofficial API](#unofficial-api)
    * [User Context](#user-context)
    * [User Organizations](#user-organizations)

### Kubernetes

<a name="jump-server"></a>
#### Jump Server

All work is done from a properly configured workstation, bastion or jump server. It is assumed that `kubectl` is configured appropriately. Typically this involves using kops to export the kubecfg, e.g.

```bash
# replace cfgname and S3 location with environment-specific values
kops export kubecfg cfgname --state s3://kops-state-store 
```
It is also assumed that a mysql client with connectivity to the databases is installed.

<a name="working-with-nodes"></a>
#### Working With Nodes

Although it is seldom needed, you can ssh into the nodes, which are just EC2 instances.
```bash
ssh -i ~/.ssh/ssh-dev.pem admin@34.212.27.241 
```

<a name="working-with-pods"></a>
#### Working With Pods

Most diagnostic work will be done by interacting with pods. You can get a list of pods (example elided for brevity):

```bash
kubectl get po
NAME                                            READY     STATUS    RESTARTS   AGE
configuration-deployment-678327073-wbvz0        1/1       Running   0          16h
migrate-reporting-deployment-3157337915-34zqn   1/1       Running   0          15h
package-processor-deployment-1539920652-lm7ck   1/1       Running   0          15h
rabbit-deployment-2895814383-gjqcd              1/1       Running   0          16h
...
```

The simplest diagnostic is to look at a pod's logs, either downloading them or tailing them:

```bash
kubectl logs migrate-reporting-deployment-3157337915-34zqn > migrate.log
kubectl logs -f migrate-reporting-deployment-3157337915-34zqn
```

To get a pod's diagnostic status, you'll need two sessions: one to port-forward and the other to curl. One way to do this is to use `screen`:

```bash
screen
kubectl port-forward migrate-reporting-deployment-3157337915-34zqn 8008
Ctrl-A, C
# the SBAC diagnostic status
curl http://localhost:8008/status?level=5
# all the Spring actuator end-points are available, e.g.:
curl http://localhost:8008/health
curl http://localhost:8008/env
curl http://localhost:8008/configprops
```

You can start a shell in a pod but know that these are thin Alpine images. You'll likely need to install any utility you need. Messing with an image isn't a good long-term idea but it can get you through troubleshooting. For example, suppose you want to try hitting an image's own status end-point from within the image:

```bash
$ kubectl exec migrate-reporting[k8s-randomization] -it /bin/sh
# apk -v --update add curl jq
# curl http://localhost:8008/health | jq .
```

Whenever a `curl` command is presented in this document it is assumed one of these two techniques is being used.

In one debugging session we wanted to run the AWS CLI from within a pod. No stable aws-cli apk so:

```bash
apk -v --update add python py-pip groff less mailcap 
pip install --upgrade awscli s3cmd python-magic
apk -v --purge del py-pip
rm /var/cache/apk/*
# configure AWS
aws configure
```

### Specific Issues

<a name="state-schools"></a>
#### Organization Sync - Improperly configured schools 

> NOTE: this particular example has been resolved by making the school processing ignore such schools.
> However, the diagnostic steps have been retained to demonstrate the problem-solving techniques.

The system periodically gets organization data from ART and imports it. This is done by the task service and is configured to run at night once a day. To see how it is doing, inspect the log for the task service:

```bash
# find task-service pod then tail the log
kubectl get po | grep task
kubectl logs -f task-server-deployment-1565566610-xfsml
```

You should see something like:

```text
2017-08-19 04:00:00.001  INFO 1 --- [pool-1-thread-1] onsConfiguration$UpdateOrganizationsTask : Scheduled task triggered: Update Organizations
2017-08-19 04:01:05.437  INFO 1 --- [pool-1-thread-1] o.o.r.i.t.s.i.RestImportServiceClient    : posted organizations import, id: 11, status: ACCEPTED
```

But you may also see indications of a problem, something like:

```text
2017-08-20 04:00:00.000  INFO 1 --- [pool-1-thread-1] onsConfiguration$UpdateOrganizationsTask : Scheduled task triggered: Update Organizations
2017-08-20 04:00:11.275  WARN 1 --- [pool-1-thread-1] o.o.r.i.t.s.i.RestImportServiceClient    : organization import not accepted, id: 11, status: BAD_DATA
```

At this point you can use either the database to inspect import record 11, or look at the package processor log.

```bash
# use mysql to query import table in warehouse
mysql> select id, digest, message from import where id=11;
# find package processor pod then tail the log
kubectl get po | grep package
kubectl logs -f package-processor-deployment-1539920652-2pdrj
```

Here is a snippet of the error message:

```json
{"messages":[
  {"elementName":"54721570000000 district","value":"CA","error":"unknown district [CA]"},
  {"elementName":"54722640000000 district","value":"CA","error":"unknown district [CA]"},
  ...
```

It looks like a number of schools are parented by a state. You can see the payload by looking in the S3 archive bucket. Using the digest value from the mysql command above: the archive layout uses the first characters of the digest to partition the files. In this case the digest is `0B01C47442601FFC20E090D476A20983` so we are looking for the file 
`s3://archive/0B/01/0B01C47442601FFC20E090D476A20983`. Indeed, in the file we see something like:

```json
{
    "id" : "5489fdd4e4b08780969fc317",
    "entityId" : "52105200000000",
    "entityName" : "Tehama eLearning Academy",
    "parentEntityId" : "CA",
    "stateAbbreviation" : "CA",
    "nationwideIdentifier" : "10699",
    "parentId" : "5425732de4b02ab2d550cbc0",
    "entityType" : "INSTITUTION",
    "formatType" : "INSTITUTION",
    "tenantType" : "INSTITUTION",
    "alternateKey" : "entityId: 52105200000000, stateAbbreviation: CA",
    "action" : "UPD",
    "parentEntityType" : "STATE"
}
```

<a name="migrate"></a>
#### Migrate

The migrate service is responsible for moving data from the data warehouse to the reporting data mart. As a Spring Boot application, the migrate reporting service can be paused/resumed. This will temporarily suspend the migrate task which may be desired to perform upgrades, etc. This is independent of the migrate service disabling itself when an error occurs. Simply POST an empty body to `pause` or `resume` end-point.

To see the current status of the migrate service:

```bash
$ kubectl exec migrate-reporting[k8s-randomization] -it /bin/sh
# curl http://localhost:8008/status
{
  "statusText": "Ideal",
  "statusRating": 2,
  "level": 0,
  "dateTime": "2017-06-13T15:10:46.925+0000",
  "migrate-reporting-job": {
    "statusText": "Warning",
    "statusRating": 2,
    "lifecycle": "running,enabled"
  }
}
```

To pause/resume the migrate service:

```bash
# curl -X POST http://localhost:8008/pause
true

# curl http://localhost:8008/status
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

# curl -X POST http://localhost:8008/resume
true
```

The migrate reporting service will disable itself if there is an error during processing. You can see this by getting the status (see above) which will contain `"lifecycle": "running,disabled"`. Once the cause of the problem has been resolved the service can be enabled by adjusting the status of the most recent migrate task in the reporting database. Connect with your favorite SQL client using a write-enabled account and change the status to `-10` setting a message to explain. Note: if the status of the most recent migrate entry is not `10` or `-20` there is something else wrong, please contact a system administrator to help.

```bash
mysql> select * from reporting.migrate order by id desc limit 1;
+------+--------+--------+-----------------+----------------+----------------------------+----------------------------+---------+
| id   | job_id | status | first_import_id | last_import_id | created                    | updated                    | message |
+------+--------+--------+-----------------+----------------+----------------------------+----------------------------+---------+
| 1443 |   1443 |    -20 |         1434005 |        1435004 | 2017-06-13 14:31:57.582819 | 2017-06-13 14:34:00.395498 | NULL    |
+------+--------+--------+-----------------+----------------+----------------------------+----------------------------+---------+
1 row in set (0.04 sec)

mysql> update reporting.migrate set status=-10, message='manually abandoned, fixed problem' where id=1443;
Query OK, 1 row affected (0.04 sec)
Rows matched: 1  Changed: 1  Warnings: 0
```

#### TODO - clean this up, it was just copied from another document

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





<a name="reconciliation-report"></a>
#### Reconciliation Report

The most common issue with the reconciliation report is a mismatch in counts. The monitoring document describes, in detail, versions of queries for looking at import status. For a system that is generally running fine it is simplest to do this to get to the heart of the problem:

```sql
mysql> SELECT count(*) cnt, message FROM import WHERE content=1 AND status<0 GROUP BY message;
+-----+------------------------------------------------------------------------------------------------------------+
| cnt | message                                                                                                    |
+-----+------------------------------------------------------------------------------------------------------------+
|  10 | unable to find a school with natural id [37103710136085]                                                   |
+-----+------------------------------------------------------------------------------------------------------------+
```

The reconciliation report and/or the complaint about any discrepancy will usually have information to follow-up on the issue. Resolve the issue (usually by adding the school to ART and forcing the updateOrganization task to run see [Unknown School](#unknown-school)).

This shouldn't be necessary or even desired once things are working but when we were testing things it was handy to **trigger the reconciliation report** without waiting for the scheduled time; post to the actuator endpoint:

```text
$ kubectl exec task-server[k8s-randomization] -it /bin/sh
# apk -v --update add curl
# curl -X POST http://localhost:8008/reconciliationReport
```

<a name="unknown-school"></a>
#### Unprocessed Test Result - UNKNOWN_SCHOOL

There is a monitoring query that returns the count of imports by status. Running that query recently produced this:

```bash
mysql> select name, count(*) count from import join import_status on import.status=import_status.id group by status;
+----------------+---------+
| name           | count   |
+----------------+---------+
| PROCESSED      | 2399812 |
| UNKNOWN_SCHOOL |      82 |
+----------------+---------+
```
Now what? First, look at the import records that triggered the unknown school to get additional details:

```text
# UNKNOWN_SCHOOL = -6 which can be seen in the import_status table
mysql> select distinct message from import where status = -6;
+----------------------------------------------------------+
| message                                                  |
+----------------------------------------------------------+
| unable to find a school with natural id [30665226130553] |
+----------------------------------------------------------+
mysql> select id, digest, message from import where status = -6 limit 1 \G
         id: 2443682
     digest: 18B972210FEB91F3DDC545CB4CA5B33B
    message: unable to find a school with natural id [30665226130553]
```
You can retrieve the actual TRT payload by either hitting the import API end-point or by navigating to S3. The data is partitioned in S3 using the content type and first two pairs of characters in the `digest`. In this case it will be s3://archive/EXAM/18/B9/18B972210FEB91F3DDC545CB4CA5B33B. Viewing the TRT, the school info: `District: (30665220000000, Garden Grove Unified), School: (30665226130553, Spectrum Center - Rossier Elementary)`.

Resolving the missing school is beyond the scope of this document. Once the missing school is in ART ...

Force the organization sync task to run by poking the task service:

```bash
$ kubectl exec task-server[k8s-randomization] -it /bin/sh
# curl -X POST http://localhost:8008/updateOrganizations
```

After that completes, trigger a resubmit of data by hitting the import API. This requires ASMTDATALOAD credentials so set the ACCESS_TOKEN and then (jump server works fine but these commands can be executed from any internet-enabled machine):

```bash
$ ACCESS_TOKEN=`curl -s -X POST --data 'grant_type=password&username=dwtest@example.com&password=password&client_id=rdw&client_secret=rdw-secret' 'https://sso.smarterbalanced.org:443/auth/oauth2/access_token?realm=/sbac' | jq -r '.access_token' `
$ echo $ACCESS_TOKEN
a0e7a6fc-a5df-4bd6-94c6-f8a9355374db

$ curl -X POST --header "Authorization: Bearer ${ACCESS_TOKEN}" https://import.smarterbalanced.org/exams/imports/resubmit?status=UNKNOWN_SCHOOL
```

<a name="duplicate-schools"></a>
#### Duplicate Schools
The user complained that there were a bunch of duplicate schools and LEA's showing in the system. They gave the example
of "Pioneer High" and "Freedom Middle".

Querying the system for schools with those prefixes showed something interesting: schools with trailing '0000000' in
their state id. That doesn't match the spec which indicates the first 7 characters represent the district while the
last 7 characters represent the school. No valid school should end with '0000000'. Also, it appears like the first 7
characters in these schools are actually the school id. And these schools have no exams associated with them.
```bash
mysql> select s.natural_id sid, s.name sname, d.natural_id did, d.name dname, count(e.id) from school s join district d on d.id = s.district_id left join exam e on e.school_id = s.id where s.name like 'Pioneer High%' or s.name like 'Freedom Middle%' group by s.id;
+----------------+-----------------------------+----------------+---------------------------+-------------+
| sid            | sname                       | did            | dname                     | count(e.id) |
+----------------+-----------------------------+----------------+---------------------------+-------------+
| 19651281936889 | Pioneer High                | 19651280000000 | Whittier Union High       |        2746 |
| 24736192430031 | Pioneer High (Continuation) | 24736190000000 | Gustine Unified           |           0 |
| 45302000000000 | Pioneer High School         | 45701360000000 | Shasta Union High         |           0 |
| 43359490000000 | Pioneer High School         | 43696660000000 | San Jose Unified          |           0 |
| 57727100101162 | Pioneer High School         | 57727100000000 | Woodland Joint Unified    |          90 |
| 43696664335949 | Pioneer High                | 43696660000000 | San Jose Unified          |           0 |
| 61177170000000 | Freedom Middle School       | 15637500000000 | Rosedale Union Elementary |           0 |
| 15637506117717 | Freedom Middle              | 15637500000000 | Rosedale Union Elementary |           0 |
+----------------+-----------------------------+----------------+---------------------------+-------------+
```
This analysis lead to the following query which finds similar "bad" schools.
```bash
mysql> select bad.natural_id, bad.name, s.natural_id, s.name from school bad join school s on substr(bad.natural_id, 1, 7) = substr(s.natural_id, 8) where substr(bad.natural_id, 8) = '0000000';
+----------------+-----------------------------------------+----------------+--------------------------------------------+
| natural_id     | name                                    | natural_id     | name                                       |
+----------------+-----------------------------------------+----------------+--------------------------------------------+
| 60126940000000 | Farragut Elementary School              | 19644446012694 | Farragut Elementary                        |
| 01231410000000 | Alliance Ted K. Tajima High             | 19647330123141 | Alliance Ted K. Tajima High                |
| 47327070000000 | Dunsmuir                                | 47702504732707 | Dunsmuir High                              |
| 60126780000000 | El Marino Elementary School             | 19644446012678 | El Marino Elementary                       |
| 60126860000000 | El Rincon Elementary School             | 19644446012686 | El Rincon Elementary                       |
| 01214260000000 | IvyTech Charter                         | 56739400121426 | IvyTech Charter                            |
...
| 01294940000000 | Kairos Public School Vacaville Academy  | 48705730129494 | Kairos Public School Vacaville Academy     |
| 07365040000000 | San Ramon Valley High                   | 07618040736504 | SAN RAMON VALLEY HIGH                      |
| 60450820000000 | Los Cerritos Elementary                 | 41690706045082 | Los Cerritos Elementary                    |
| 15359050000000 | Taft Union High School                  | 15638181535905 | Taft Union High                            |
| 37312620000000 | Steele Canyon High                      | 37681303731262 | Steele Canyon High                         |
| 01235050000000 | Mission Preparatory                     | 38684780123505 | Mission Preparatory                        |
+----------------+-----------------------------------------+----------------+--------------------------------------------+
617 rows in set (2.24 sec)
```
These schools should be removed from ART. Then they must be deleted from RDW by hand because the ART sync process will
never remove schools from RDW. 
```sql
use warehouse;
delete s from school s where s.natural_id like '%0000000' and not exists (select sg.id from student_group sg where sg.school_id = s.id) and not exists (select es.id from exam_student es where es.school_id = s.id);

use reporting;
delete s from school s where s.natural_id like '%0000000' and not exists (select sg.id from student_group sg where sg.school_id = s.id) and not exists (select e.id from exam e where e.school_id = s.id);
```

<a name="empty-report"></a>
#### Student Report - Empty PDF

Support verified that the report is really empty. With the low bandwidth with support we really only know the school
name, "Awesome Academy" and that the incident happened yesterday. All the initial work will be in MySQL, reporting:

* Get school info, specifically the id, in this case id=6470
    ```sql
    SELECT * FROM school WHERE name LIKE 'Awesome Academy%';
    ```
* Find a user report record(s) for that school:
    ```sql
    SELECT * FROM user_report WHERE report_request LIKE '%schoolID":6470%';
    ```
    * {"@class":"org.opentestsystem.rdw.reporting.common.report.SchoolGradeExamReportRequest","schoolYear":2018,"language":"eng","name":"Awesome Academy 2018","order":"STUDENT_NAME","schoolId":6470,"gradeId":11}
    * schoolId: 6470
    * gradeId: 11
    * schoolYear: 2018
* See what exam data we have matching those criteria
    ```sql
    select count(*) from exam where school_id=6470 and school_year=2018 and grade_id=11;
    +----------+
    | count(*) |
    +----------+
    |        0 |
    +----------+
    ```
* Oops. What's up with that?
    ```sql
    select grade_id, count(*) cnt from exam where school_id=6470 and school_year=2018 group by grade_id;
    +----------+-----+
    | grade_id | cnt |
    +----------+-----+
    |       12 |   9 |
    +----------+-----+
    ```

There is no grade 11 exam data, it is grade 12. The user will have to specify grade 12 when requesting the report.
But that's not possible. So this is clearly a bug: the system should use the assessment grade when filtering results
for the printed report.

<a name="failed-group"></a>
#### Student Group - Failed Import

If a student group import fails, the system retains the intermediate data to help when diagnosing the problem. This is an advanced effort usually requiring tier 3 support which won't be detailed here. However, it does require that that intermediate data be deleted.

If you know the import that failed, cleaning up the data is easy, something like:
```sql
DELETE FROM upload_student_group WHERE import_id = 33962;
```

If individual failures were not addressed, deleting all "old" entries may be more appropriate. For example, to delete failures more than 3 days old:
```sql
DELETE usg FROM upload_student_group usg JOIN import i ON i.id = usg.import_id WHERE i.updated < TIMESTAMPADD(DAY, -3, CURRENT_TIMESTAMP);
```

<a name="invalid-group"></a>
#### Student Groups - Invalid Upload Status

A number of users were having problems uploading student group CSV files. One of them provided a screen shot showing
that the file was accepted but then switch to Upload Status `Invalid` (Upload ID = 234). The information provided by the user varies; typically they know their email and might know the upload id since it is displayed on the screen.

A quick check in the database shows that the status is invalid but the error message isn't that useful:
```sql
mysql> select * from upload_student_group_batch where id = 234 \G
*************************** 1. row ***************************
      id: 234
  digest: E5157285E83BEC46273F441696A9FA7E
  status: -1
 creator: user@example.com
 created: 2017-09-28 21:41:40.137750
 updated: 2017-09-28 21:41:42.786444
 message: Problem processing Batch: 234
filename: Sample Unified.csv
```

If they provide their email and not the id, something like this may be more appropriate:
```sql
mysql> select * from upload_student_group_batch where creator = 'user@school.org' and message is not null \G
*************************** 78. row ***************************
      id: 14669
  digest: D91B31A2650C0D75206155E1A4AB39B7
  status: -1
 creator: user@school.org
 created: 2018-01-13 02:02:00.053236
 updated: 2018-01-13 02:02:00.201750
 message: Problem processing Batch: 14669
filename: Harding test.csv
```

A dump of the log was more useful. The process is the group-processor
```bash
kubectl logs group-processor-deployment-1684562224-r8km9 > log.txt
view log.txt
...
2017-09-28 21:41:40.148  INFO 1 --- [ROUPS.default-1] o.o.r.i.g.GroupProcessorConfiguration    : Received payload: {"digest":"E5157285E83BEC46273F441696A9FA7E","uploadId":234}
2017-09-28 21:41:42.784 ERROR 1 --- [ROUPS.default-1] o.o.r.i.g.s.impl.DefaultGroupProcessor   : Problem processing batch: 234

org.springframework.jdbc.UncategorizedSQLException: PreparedStatementCallback; uncategorized SQLException for SQL [LOAD DATA FROM S3 ? INTO TABLE upload_student_group FIELDS TERMINATED BY ',' IGNORE 1 LINES (@vgroup_name, @vschool_natural_id, school_year, @vsubject_code, @vstudent_ssid, @vgroup_user_login) SET
  batch_id = ?,
  group_name = TRIM(REPLACE(@vgroup_name, '\r', '')),
  school_natural_id = TRIM(REPLACE(@vschool_natural_id, '\r', '')),
  subject_code = nullif(TRIM(REPLACE(@vsubject_code, '\r', '')), ''),
  student_ssid = nullif(TRIM(REPLACE(@vstudent_ssid, '\r', '')), ''),
  group_user_login = nullif(TRIM(REPLACE(@vgroup_user_login, '\r', '')), '')]; SQL state [HY000]; error code [1366]; Incorrect integer value: ' Period 1"' for column 'school_year' at row 1; nested exception is java.sql.SQLException: Incorrect integer value: ' Period 1"' for column 'school_year' at row 1
...
```

Looking at the uploaded file (either from user or from S3):
```csv
group_name,school_natural_id,school_year,subject_code,student_ssid,group_user_login
"Smith, John, Period 1",37682966039010,2018,All,1178197650,jsmith@example.com
"Smith, John, Period 1",37682966039010,2018,All,1307844641,jsmith@example.com
...
```

The problem is the S3 loader doesn't have the necessary `ENCLOSED BY '"'` clause so it is failing to parse the first
column properly. 

<a name="missing-student-group"></a>
#### Missing Student Group
A user complained that they were uploading student group CSV files successfully but that the groups are not appearing.
* User is melissa@fubar.org

Let's see if the uploads are in there:
```sql
select * from upload_student_group_batch sgb where sgb.creator='mespi5@lausd.net' and sgb.status=1;
+------+----------------------------------+--------+------------------+----------------------------+----------------------------+---------+---------------------+
| id   | digest                           | status | creator          | created                    | updated                    | message | filename            |
+------+----------------------------------+--------+------------------+----------------------------+----------------------------+---------+---------------------+
|  924 | 91038F3D58C079FC446EEE6C7EF49035 |      1 | mespi5@lausd.net | 2017-10-19 04:09:35.985610 | 2017-10-19 04:09:36.509949 |         | kenastondone.csv    |
|  984 | 49E2C7AB31A78B7D310771F191589E1D |      1 | mespi5@lausd.net | 2017-10-19 20:22:40.659808 | 2017-10-19 20:22:40.887697 |         | done2.csv           |
...
| 8709 | 487EE591DCDE5C245CC90F8F13F5F327 |      1 | mespi5@lausd.net | 2017-11-30 06:10:07.019702 | 2017-11-30 06:10:07.218732 |         | maxwellnew4.csv     |
| 8710 | 2BC77BF6CD1F8BEAF61C4AEB9346EA79 |      1 | mespi5@lausd.net | 2017-11-30 06:12:36.422368 | 2017-11-30 06:12:39.371562 |         | Mancilla7.csv       |
+------+----------------------------------+--------+------------------+----------------------------+----------------------------+---------+---------------------+
44 rows in set (0.01 sec)
```

Yup, looks good. So, are there any student groups associated with any of those records?
```sql
select sgb.id, sgb.digest, sgb.creator, sgb.filename, count(sg.id) as groups from upload_student_group_batch sgb
  join import i on i.content=5 and sgb.id = i.batch
  join student_group sg on sg.import_id=i.id
where sgb.creator='mespi5@lausd.net' and sgb.status=1
group by sgb.id;
Empty set (0.05 sec)
```

Yikes, that's not good. Let's try that another way:
```sql
select sgb.* from upload_student_group_batch sgb
  left outer join import i on i.content=5 and i.batch = sgb.id
where sgb.status=1 and sgb.creator='mespi5@lausd.net'
  and i.id is null;
+------+----------------------------------+--------+------------------+----------------------------+----------------------------+---------+---------------------+
| id   | digest                           | status | creator          | created                    | updated                    | message | filename            |
+------+----------------------------------+--------+------------------+----------------------------+----------------------------+---------+---------------------+
|  924 | 91038F3D58C079FC446EEE6C7EF49035 |      1 | mespi5@lausd.net | 2017-10-19 04:09:35.985610 | 2017-10-19 04:09:36.509949 |         | kenastondone.csv    |
|  984 | 49E2C7AB31A78B7D310771F191589E1D |      1 | mespi5@lausd.net | 2017-10-19 20:22:40.659808 | 2017-10-19 20:22:40.887697 |         | done2.csv           |
...
| 8708 | FD2DFBAC33E9A2DBB3ECE4F12EE99DBD |      1 | mespi5@lausd.net | 2017-11-30 06:08:52.794912 | 2017-11-30 06:08:53.042526 |         | Keuper6.csv         |
| 8709 | 487EE591DCDE5C245CC90F8F13F5F327 |      1 | mespi5@lausd.net | 2017-11-30 06:10:07.019702 | 2017-11-30 06:10:07.218732 |         | maxwellnew4.csv     |
| 8710 | 2BC77BF6CD1F8BEAF61C4AEB9346EA79 |      1 | mespi5@lausd.net | 2017-11-30 06:12:36.422368 | 2017-11-30 06:12:39.371562 |         | Mancilla7.csv       |
+------+----------------------------------+--------+------------------+----------------------------+----------------------------+---------+---------------------+
44 rows in set, 65535 warnings (1.43 sec)
```

That confirms that the files were processed "successfully" but no student groups were created. Using the `digest` values, we can inspect the files. Looking closely at the files, it is apparent that the newline character is `0x0D` aka `CR` aka `\r`. Our parser doesn't know how to deal with that, it is expecting `\n` (aka `0x0A` aka `LF`) or `\r\n` to be the newline. 

Ask the user to switch to that by saving in Windows CSV format.

Might also file a JIRA to properly handle the various newline characters.

<a name="missing-student-group-one-user"></a>
#### One User Can't See Student Group
Screen shots show that a student group was successfully uploaded but the teacher in the group does not see the student group when they login. We know a few things:
* School is Riverdale High, 10754081035575
* User is krathburn@fubar.org (yeah, okay i changed that)

* First, get the school id

    ```sql
    select * from school where natural_id='10754081035575';
    +-------+-------------+----------------+----------------+---
    | id    | district_id | natural_id     | name           | im
    +-------+-------------+----------------+----------------+---
    | 13070 |        1016 | 10754081035575 | Riverdale High | ...  
    +-------+-------------+----------------+----------------+---    
    ```
* Does student group exist? 
 
    ```sql
    select * from student_group where school_id=13070;
    +-------+------------------------------+-----------+-------------+------------+--------+---------+----------------------------+-----------+------------------+---------+----------------------------+
    | id    | name                         | school_id | school_year | subject_id | active | creator | created                    | import_id | update_import_id | deleted | updated                    |
    +-------+------------------------------+-----------+-------------+------------+--------+---------+----------------------------+-----------+------------------+---------+----------------------------+
    | 42791 | ChavezP3                     |     13070 |        2018 |          2 |      1 | NULL    | 2017-11-01 16:03:53.058082 |   2774667 |          2774667 |       0 | 2017-11-01 16:03:53.058082 |
    | 42794 | CurwickP11thgrade            |     13070 |        2018 |          2 |      1 | NULL    | 2017-11-01 16:10:58.099894 |   2775278 |          2775278 |       0 | 2017-11-01 16:10:58.099894 |
    | 43404 | DistrictELApractice11thgrade |     13070 |        2018 |          2 |      1 | NULL    | 2017-11-01 23:02:46.545053 |   2797758 |          2797758 |       0 | 2017-11-01 23:02:46.545053 |
    | 60917 | REStestP3                    |     13070 |        2018 |          2 |      1 | NULL    | 2017-11-09 23:23:48.340152 |   3060753 |          3060753 |       0 | 2017-11-09 23:23:48.340152 |
    +-------+------------------------------+-----------+-------------+------------+--------+---------+----------------------------+-----------+------------------+---------+----------------------------+

    ```
* That matches the screen shot provided by the administrator. The group of interest is the DistrictELApractice11thgrade, id=43404. We know this because of hints in the JIRA ticket and the CSV provided. Let's check the membership of that group:

    ```sql
    select * from user_student_group where student_group_id=43404;
    +------------------+-----------------------+
    | student_group_id | user_login            |
    +------------------+-----------------------+
    |            43404 | cstilson@fubar.org    |
    |            43404 | dsimpson@fubar.org    |
    |            43404 | ginadaniels@fubar.org |
    |            43404 | jmoore@fubar.org      |
    |            43404 | jpercell@fubar.org    |
    |            43404 | mallein@fubar.org     |
    |            43404 | mlocke@fubar.org      |
    |            43404 | skirk@fubar.org       |
    +------------------+-----------------------+    
    ``` 
* This matches the CSV provided in the JIRA, except one user is missing. Given what we know about the processing of the CSV this doesn't make sense. Let's see what was actually uploaded to the system. We need the digest for the file so we can find it in S3. For student group upload that requires an extra join:

    ```sql
    select * from import i join upload_student_group_batch b on b.id=i.batch where i.id=2797758 \G
    *************************** 1. row ***************************
             id: 2797758
         status: 1
        content: 5
    contentType: group batch
         digest: 13070
          batch: 2839
        creator: NULL
        created: 2017-11-01 23:02:44.947794
        updated: 2017-11-01 23:02:46.682500
        message: NULL
             id: 2839
         digest: 9A462BCD48E9AB16999F3D52EC6C5DA1
         status: 1
        creator: briancurwick@rjusd.org
        created: 2017-11-01 23:02:42.088673
        updated: 2017-11-01 23:02:46.708171
        message: 
       filename: DistrictELApractice11thgrade.csv    
    ``` 
* Retrieving the file from S3 shows a CSV file that is not the same as the sample, specifically the rows for that 9th user are not in it. User error. 


<a name="unable-to-admin-district"></a>
#### Unable To Admin District
A user has been granted admin privileges for a district (in ART). Typically this is GROUP_ADMIN but there are other admin roles. But in the application, they do not see the button that takes them to the admin screen. We usually know a few things:
* User login (and perhaps credentials for mirroring).
* District name and id, e.g. KIPP Adelante Preparatory Academy (01013450000000)

* If we have the user credentials the [User Context](#user-context) will show us permissions for that user. In this situation, it is likely that the permissions list is empty, which is why the button isn't displayed.

* Check to see if the district is in the system. There are two likely scenarios:
	* The district doesn't exist in the system.
		```sql
		select * from district where natural_id='01611190119222';
		Empty set (0.00 sec)
		```
	  In this situation, the user will not have permissions because, although ART grants them a role, that role doesn't apply to any districts in RDW. But why isn't the district in the system? The most likely reason is that there are no schools in ART for that district. You can query ART (or its underlying mongo db) or use the [ART Organization Data](#art-organization-data) to verify this. Districts are only brought into RDW if they are associated with a school.
	* The district does exist.
		```sql
		select * from district where natural_id='37683380101345' OR name='KIPP Adelante Preparatory Academy';
		+-----+----------------+-----------------------------------+
	   | id  | natural_id     | name                              |
	   +-----+----------------+-----------------------------------+
	   | 809 | 37683380101345 | KIPP Adelante Preparatory Academy |
	   +-----+----------------+-----------------------------------+
		```
	In this situation, more digging is required. Suspicion is raised because the id doesn't match and it looks more like a school id than a district id.  Using the [ART Organization Data](#art-organization-data) we observe that there are two districts in the system with that name, one with id 01013450000000 (with no schools) and one with 37683380101345 (with one school). Looking at permissions, it turns out they granted privileges for the one that had no schools. It is likely that the school needs to be assigned to the correct district in ART. Then the daily sync will catch the change.

<a name="art-organization-data"></a>
#### ART Organization Data
The system pulls data from ART periodically (once a day by default). The payload returned by ART is preserved in S3 and can be used to do forensics on organization problems. 
* First, get the digest for the most recent payload:
	```sql
	select * from import where content=4 order by id desc limit 1 \G
	*************************** 1. row ***************************
	         id: 4708387
	     status: 1
	    content: 4
	contentType: application/json
	     digest: 7A108B8E5BFF30C6B279D4A3890A2FA6
	      batch: NULL
	    creator: rdw-ingest-prod@smarterbalanced.org
	    created: 2018-02-09 10:00:53.921911
	    updated: 2018-02-09 10:01:30.468025
	    message: 13529 schools processed
	```
* Use the digest and navigate to the appropriate S3 folder. The files are partitioned by the first two pairs of digits in the digest: in this case it will be something like S3://archive-bucket/ORGANIZATION/7A/10/7A108B8E5BFF30C6B279D4A3890A2FA6. Download the file and view with a good JSON viewer. Organization data includes fields that let you search for the parent, children, etc.:
	```json
	{
     "id" : "542f120ce4b0c757f5e1807d",
     "entityId" : "23655990000000",
     "entityName" : "Point Arena Joint Union High",
     "parentEntityId" : "CA",
     "stateAbbreviation" : "CA",
     "parentId" : "5425732de4b02ab2d550cbc0",
     "entityType" : "DISTRICT",
     "parentEntityType" : "STATE",
   }
	```

<a name="item-viewer"></a>
#### Item Viewer
When the user views the detail of a test item they are being presented with proxied content from IRiS. Although a full troubleshooting guide for IRiS doesn't belong here, there is at least one issue that RDW users run into: items not rendering because IRiS has gotten grumpy. This can often be fixed by forcing IRiS to reload its content. First, confirm that the IRiS logs have telltale exceptions.
```bash
$ ssh iris
$ sudo su
# view /var/log/tomcat/student/student.log
...
01:53:55.373 [http-bio-8080-exec-120176] ERROR tds.blackbox.web.handlers.TDSHandler - No content found by id I-200-83838
tds.blackbox.ContentRequestException: No content found by id I-200-83838
...
```
If there are a lot of such messages, force IRiS to reload:
```bash
$ curl http://localhost:8080/iris/Pages/API/content/reload
```

For additional IRiS documentation please refer to [IRiS in RDW](../deploy/IRIS.AWS.md) and [TDS IRIS](https://github.com/SmarterApp/TDS_IRIS).


### Unofficial API
There are a number of API end-points that support the reporting webapp. These end-points are not official and may change without notice. However, some of them can be useful for diagnosing issues. In general to use these, you must log into the reporting system and then, in the same browser session, navigate to the appropriate end-point.

<a name="user-context"></a>
#### User Context
To get metadata about the user's context: navigate to https://reporting.smarterbalanced.org/api/reporting-service/user. This should return a payload that includes some useful context information about the user including permissions and authorities. For example:
```json
{
  "username": "mlaffoon@fairwaytech.com",
  "authorities": [
    {
      "authority": "PERM_DATA_WRITE"
    },
    {
      "authority": "PERM_GROUP_PII_READ"
    },
    {
      "authority": "PERM_GROUP_READ"
    },
    {
      "authority": "PERM_GROUP_WRITE"
    },
    {
      "authority": "ROLE_ASMTDATALOAD"
    },
    {
      "authority": "ROLE_GROUP_ADMIN"
    },
    {
      "authority": "ROLE_PII_GROUP"
    }
  ],
  "accountNonExpired": true,
  "accountNonLocked": true,
  "credentialsNonExpired": true,
  "enabled": true,
  "id": "5aff0274e4b03dd700aff595",
  "email": "user@example.com",
  "firstName": "User",
  "lastName": "Test",
  "permissionsById": {
    "GROUP_PII_READ": {
      "id": "GROUP_PII_READ",
      "scope": {
        "districtGroupIds": [],
        "districtIds": [],
        "institutionGroupIds": [],
        "institutionIds": [],
        "statewide": true
      }
    },
    "GROUP_WRITE": {
      "id": "GROUP_WRITE",
      "scope": {
        "districtGroupIds": [],
        "districtIds": [
          1,
          2
        ],
        "institutionGroupIds": [],
        "institutionIds": [],
        "statewide": false
      }
    },
    "GROUP_READ": {
      "id": "GROUP_READ",
      "scope": {
        "districtGroupIds": [],
        "districtIds": [
          1,
          2
        ],
        "institutionGroupIds": [],
        "institutionIds": [],
        "statewide": true
      }
    },
    "DATA_WRITE": {
      "id": "DATA_WRITE",
      "scope": {
        "districtGroupIds": [],
        "districtIds": [],
        "institutionGroupIds": [],
        "institutionIds": [],
        "statewide": true
      }
    }
  }
}
```

A user's groups may be seen by navigating to https://reporting.smarterbalanced.org/api/reporting-service/groups:
```json
[
  {
    "id": 244,
    "name": "Katz G1",
    "schoolName": "Katz Field",
    "schoolId": 5,
    "userCreated": false
  }
]
```

<a name="user-organizations"></a>
#### User Organizations
To see organizations that a user has permission to access:
* https://reporting.smarterbalanced.org/api/reporting-service/organizations/schools
* https://reporting.smarterbalanced.org/api/reporting-service/organizations/schoolGroups
* https://reporting.smarterbalanced.org/api/reporting-service/organizations/districts
* https://reporting.smarterbalanced.org/api/reporting-service/organizations/districtGroups
