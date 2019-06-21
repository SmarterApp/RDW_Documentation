## MultiTenancy

**Intended Audience**: This document contains information on how multi-tenancy works in the [Reporting Data Warehouse](../README.md) (RDW). Operations and system administration will find it useful.

### Table of Contents

* [Creating Sandbox Data Sets](#creating-sandbox-data-sets)
* [Manual Tenant Creation](#manual-tenant-creation)
* [Manual Sandbox Creation](#manual-sandbox-creation)
* [Manual Configuration Change](#manual-configuration-change)


### Creating Sandbox Data Sets

When creating a Sandbox, the system allows the administrator to select 
from available data sets. A data set is a collection of database table 
contents that are loaded into the warehouse (and migrated to the 
reporting databases). 
 
Unfortunately, at this time, creating the data sets is a manual process.

The goal is to load the warehouse with all the required data, without 
allowing the system to migrate the data. Then dump that data and stage 
it in the data sets folder in S3. Probably the easiest environment to
do this is a local development system.

For this example, we are creating a data set for Smarter Balanced 
assessments for ELA and Math using the demo institutions and a state
code of TS.

* Data generation
    * collect inputs for data generation
        * assessment package definitions
        * institution hierarchy
    * use data generator to create TRTs
    * spot check the TRTs
* Configure RDW services
    * Reset the schema using RDW_Schema. Adjust school_year for generated data.
    ```
    RDW_Schema$ gradle -Pschema_prefix=ts_ cleanAll migrateAll
    RDW_Schema$ echo "DELETE FROM ts_warehouse.school_year WHERE year IN (2015, 2016)" | mysql -u root ts_warehouse
    ```
    * Run ingest services: modify RDW_Ingest docker-compose file, commenting out migrate-reporting and task-service:
    ```
    RDW_Ingest$ gradle clean buildImage
    RDW_Ingest$ docker-compose up -d
    ```
* Load data (the usual data load process using curl or Postman and data generator submit scripts). 
Be sure to use credentials for the `TS` tenant.
    * subject definitions
    * assessment packages
    * institutions
    * TRTs
```
export ACCESS_TOKEN="sbac;dwtest@example.com;|TS|ASMTDATALOAD|STATE|SBAC||||TS||||||||||"
curl -X POST -s --header "Authorization:Bearer ${ACCESS_TOKEN}" -F file=@ELA_subject.xml http://localhost:8080/subjects/imports
curl -X POST -s --header "Authorization:Bearer ${ACCESS_TOKEN}" -F file=@Math_subject.xml http://localhost:8080/subjects/imports
curl -X POST -s --header "Authorization:Bearer ${ACCESS_TOKEN}" -F file=@AccessibilityConfig.2019.xml http://localhost:8080/accommodations/imports
curl -X POST -s --header "Authorization:Bearer ${ACCESS_TOKEN}" -F file=@2017v3.csv http://localhost:8080/packages/imports
... repeat for all assessment packages
curl -X POST -s --header "Authorization:Bearer ${ACCESS_TOKEN}" -F file=@demo.json http://localhost:8080/organizations/imports
# use submit_helper to submit TRTs (tweak ACCESS_TOKEN in script)
find ./out/*/*/* -type d | xargs -I FOLDER -P 3 ./scripts/submit_helper.sh FOLDER
```    
* Create groups. This is a tricky. We want a group per school per grade per subject.
We can use the session id from the data generator to group students, and
we'll base the groups on summative assessments for the current school year.
We'll restrict to medium sized groups.
All of this is dependent on the generated data so adjust the query populating
the temporary table as necessary ...
```
INSERT INTO import (status, content, contentType, digest, creator) VALUES (0, 5, 'text/plain', left(uuid(), 8), 'dwtest@example.com');
SELECT LAST_INSERT_ID() into @importid;

CREATE TEMPORARY TABLE school_grade_session
(tmpid INTEGER NOT NULL AUTO_INCREMENT, PRIMARY KEY (tmpid), INDEX(tmpid), UNIQUE (school_id, grade_id, subject_id))
IGNORE SELECT e.school_id, e.grade_id, a.subject_id, LEFT(e.session_id, 3) AS session, count(*) AS cnt
 FROM exam e JOIN asmt a ON e.asmt_id = a.id
 WHERE e.school_year=2019 AND a.type_id=3
 GROUP BY e.school_id, e.grade_id, a.subject_id, LEFT(e.session_id,3)
 HAVING cnt > 20 and cnt < 50;

INSERT INTO student_group (id, name, school_id, school_year, subject_id, active, creator, import_id, update_import_id)
SELECT tmpid, CONCAT(sc.name, ' (', su.code, ',', g.code, ')') as name, school_id, 2019 as school_year, subject_id, 1 as active, 'dwtest@example.com' as creator, @importid as import_id, @importid as update_import_id
FROM school_grade_session sgs
JOIN school sc ON sgs.school_id = sc.id
JOIN grade g ON sgs.grade_id = g.id
JOIN subject su ON sgs.subject_id = su.id;

INSERT IGNORE INTO student_group_membership (student_group_id, student_id)
SELECT tmpid as student_group_id, e.student_id
FROM school_grade_session sgs
JOIN exam e ON sgs.school_id = e.school_id AND sgs.grade_id = e.grade_id AND e.school_year = 2019 AND LEFT(e.session_id, 3) = sgs.session
JOIN asmt a ON e.asmt_id = a.id AND a.type_id = 3 AND sgs.subject_id = a.subject_id;

UPDATE import SET status=1 WHERE id=@importid;

DROP TABLE school_grade_session;
```
* Dump data, cleanup and create manifest
```
mysqldump -u root --tab=/tmp/sbac-dataset ts_warehouse
cd /tmp/sbac-dataset
rm schema_version.*
rm *.sql
rm audit_*.txt
find . -size 0 -delete
find *.txt > manifest.txt
```
* Upload the files to S3, e.g. `s3://rdw-qa-archive/sandbox-datasets/sbac-dataset/warehouse`

### Manual Tenant Creation

The admin UI allows a user to create a tenant. It handles all the magic under the covers.
However, there may be situations where manually creating a tenant is necessary.

TODO - steps

### Manual Sandbox Creation

The admin UI allows a user to create a sandbox. It handles all the magic under the covers.
However, there may be situations where manually creating a sandbox is necessary.
The steps are very similar to creating a tenant with some important differences:
* The sandbox id should follow the convention of the parent tenant id concatenated with "_Snnn" where nnn goes from 001 to 999. For example, a new sandbox for the OT tenant might have id OT_S012.
* A sandbox configuration must indicate that it is a sandbox and include the dataset name.
* The dataset must be loaded.

TODO - steps


### Manual Configuration Change

As part of the multi-tenant support, all the RDW services now respond dynamically to configuration changes. If a manual change is made to configuration, trigger the system to see the change:
```bash
kubectl exec -it configuration-deployment-<k8s-id> -- curl -d 'path=*' http://localhost:8888/monitor
```
It takes 20-30 seconds for the change to be loaded and propagated to the services.

