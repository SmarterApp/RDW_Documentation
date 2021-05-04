## Special upgrade to v2.5.x -- back-fills Alt Score aggregate data to predate the 2.5 release.

**Intended Audience**: this document provides detailed instructions for upgrading the [Reporting Data Warehouse (RDW)](../README.md) applications in an AWS environment 
from any v2.5.x release to back-fill Alt Score aggregate. Unlike other upgrades, no new software is added. It can be thought of like a large, one time
migration from the warehouse exam table to the Redshift exam_alt_score table. 

### Overview

This is an upgrade to RDW that requires some database modification.

The high level steps for the upgrade include:
* [Notification](#notification)
* [Prep Work](#prep-work)
* [Quiesce the System](#quiesce-the-system)
* [Backup](#backup)
* [Upgrade](#upgrade)
* [Smoke Test](#smoke-test)

### Notification
Well in advance of this work:

* [ ] Message user base about the upgrade and maintenance window.
* [ ] Alert 3rd party data feeds about the maintenance window.


### Prep
Before upgrading the system it must be made idle.

* [ ] Verify 3rd party data feeds are suspended.
* [ ] Set up static landing page and route all site traffic to it.
    * reporting.sbac.org should redirect to static landing page
    * import.sbac.org can return an error during the upgrade
* [ ] Disable any ops monitoring on RDW services.
* We're on the clock. Play the Jeopardy theme in a loop ...    
* [ ] Shut down the OLAP migration. The other services can remain running.
    ```bash
    kubectl scale deployment migrate-olap-deployment --replicas=0
    ```

### Backup

The only table being altered is Redshift exam_alt_score. It's a good idea to back up that one table. The whole Redshift
database could be backed up, but it isn't necessary. 

### Upgrade

Each tenant and instance must be upgraded separately. Data will be first be extracted from the warehouse schema into
s3, then copied from s3 and loaded into Redshift. Most of the work is done
by the extract.

##### Extract from warehouse

* [ ] Lookup the s3 bucket in the configuration's main application.yml file under the
  property archive.uri-root:
```yaml
archive:
  uri-root: s3://rdw-qa-archive
```
* [ ] Lookup the prefix for the tenant you're working on in the configuration, where
  it will be in the tenant-specific application.yml. 
  (Note: this is optional. You can ignore prefixes and just uses the same file path 
  for all tenants. However, you will have to delete the old file before starting a new tenant.)
```yaml
archive:
  tenants:
    PA:
      pathPrefix: "pa"
```
* [ ] The s3 output path will be ```s3://rdw-qa-archive/pa/MigrateAdHoc/exam_alt_score```
* [ ] Choose a start date for the back-fill. This has now been specified as '2019-11-01 00:00:00.0'
* [ ] Connect to tenant-specific warehouse schema with root level credentials.
  Ensure that your client is configured to not timeout on long-running queries. The
  extract may take an hour or more to complete. 
*  [ ] Query for a list of subject score ids to for alt scores. (This will make the
   extract query run faster.)
```mysql
SELECT id FROM subject_score 
WHERE asmt_type_id in (1,3) and score_type_id = 2;
```
```
Results:
19
20
```
* [ ] Run the extract query. Remember to substitute in the s3 output path, start date,
and query results from the previous steps. 
```mysql
SELECT wes.id, wes.exam_id, wes.subject_score_id,  we.student_id, we.asmt_id, we.school_year, wes.scale_score,
       wes.performance_level, we.completed_at, -1, we.updated, we.update_import_id
    FROM exam we
        JOIN exam_score wes ON we.id = wes.exam_id
WHERE we.deleted = 0
  AND we.updated >= '2019-11-01 00:00:00.0'
  AND we.scale_score IS NOT NULL
  AND we.performance_level IS NOT NULL
  AND wes.performance_level IS NOT NULL
  AND wes.subject_score_id in (19, 20)
ORDER BY we.completed_at DESC, we.updated DESC
INTO OUTFILE S3 's3://rdw-qa-archive/pa/MigrateAdHoc/exam_alt_score'
        FIELDS TERMINATED BY ','
        OPTIONALLY ENCLOSED BY '"'
        LINES TERMINATED BY '\n'
```
* [ ] Wait until this query has completed. You can monitor the s3 output to verify
  that the extract is still in progress.

#### Load into Redshift
* [ ] Lookup the IAM credentials in the configuration migrate.aws.redshift.role property:
```yaml
migrate:
  aws:
    redshift:
      role: arn:aws:iam::26...32:role/rdw-redshift
```
* [ ] Connect to the tenant-specific schema in Redshift with root credentials.
* [ ] Empty out the current alt_exam_score table. This table should have been backed up in a previous step. 
```redshift
DELETE FROM exam_alt_score WHERE true;
```
* [ ] Run the load query. Remember to substitute the IAM role from  the previous step 
  and also the s3 output path that was used for the extract.
```redshift
COPY exam_alt_score
    FROM 's3://rdw-qa-archive/pa/MigrateAdHoc/exam_alt_score'
    CREDENTIALS 'aws_iam_role=arn:aws:iam::26...32:role/rdw-redshift'
    FORMAT AS CSV
    MAXERROR 500
    DELIMITER ','
    TIMEFORMAT 'auto'
    COMPUPDATE OFF;
   ```

### Restart OLAP Migration
```bash
kubectl scale deployment migrate-olap-deployment --replicas=1
```

### Smoke Test

Test the Alt Score aggregate reports ensuring that the data has been back-filled.
