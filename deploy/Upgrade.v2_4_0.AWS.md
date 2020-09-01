## Upgrade v2.4.0 <- v2.x.y

**Intended Audience**: this document provides detailed instructions for upgrading the [Reporting Data Warehouse (RDW)](../README.md) applications in an AWS environment from v1.3.x to v2.0.0. Operations and system administrators will find it useful.

It is assume that the official deployment and upgrade instructions were used for the current installation. Please refer to that documentation for general guidelines.

> Although these are generic instructions, having an example makes it easier to document and understand. Throughout this document we'll use `opus` as the sample environment name; this might correspond to `staging` or `production` in the real world. We'll use `sbac.org` as the domain/organization. Any other ids, usernames or other system-generated values are products of the author's imagination. The reader is strongly encouraged to use their own consistent naming conventions. 
> **Avoid putting real environment-specific details _especially secrets and sensitive information_ in this document.**

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


### Prep Work
The goal of this step is to make changes to everything that doesn't directly affect current operations, leaving the absolute minimum work required during the actual maintenance window.

* These instructions expect you to have access to the files in the RDW project, so clone that locally if needed.
    ```bash
    cd ~
    git clone https://github.com/SmarterApp/RDW.git
    ```
* [ ] Branch the deployment and configuration repositories. All changes to these files will be made in the branch which can be quickly merged during the upgrade.
    ```bash
    cd ~
    git clone https://github.com/SmarterApp/RDW_Deploy_Opus.git
    git clone https://github.com/SmarterApp/RDW_Config_Opus.git

    cd RDW_Deploy_Opus
    git checkout master; git pull
    git checkout -b v2_4_0 master
    git push -u origin v2_4_0

    cd ../RDW_Config_Opus
    git checkout master; git pull
    git checkout -b v2_4_0 master
    git push -u origin v2_4_0
    ```
* [ ] Add a copy of this checklist to deployment and switch to that copy.
    ```bash
    cd ../RDW_Deploy_Opus
    cp ../RDW/Upgrade.v2_4_0.AWS.md .
    git add Upgrade.v2_4_0.AWS.md
    ```
* [ ] (Optional) It is a good idea to go through the rest of this document, updating the configuration, credentials, etc. to match the environment. If you don't you'll just have to do it as you go along and there will be an extra commit to do when merging the deploy branch.    
* [ ] Changes to deployment files in `RDW_Deploy_Opus`. There are sample deployment files in the `deploy` folder in the `RDW` repository; use those to copy and help guide edits.
    * The vendor upgraded Kubernetes during this upgrade and so had to update `deployment` definitions for all deployments.
        * `apiVersion: apps/v1`
        * add `spec.selector.matchLabels` to match the label in `spec.template.metadata.labels`
    * Change image versions to 2.4.0-RELEASE
    * Commit changes
        ```bash
        cd ../RDW_Deploy_Opus
        git add *
        git commit -am "Changes for v2.4.0"
        git push 
        ```
* [ ] There are minimal configuration changes for this upgrade.
    * TODO - make changes; not sure if there are any required configuration changes?
    * Commit changes
        ```bash
        cd ../RDW_Config_Opus
        git add *
        git commit -am "Changes for v2.0.0"
        git push
        ```
* [ ] Add new roles and permissions for the Reporting component in the permissions application.
    * There are new permissions to add:
        * TEST_DATA_LOADING_READ
        * TEST_DATA_LOADING_WRITE
        * TEST_DATA_REVIEWING_READ
        * TEST_DATA_REVIEWING_WRITE
        * REPORT_TEMPLATE_WRITE
    * The existing roles EMBARGO_ADMIN, SandboxDistrictAdmin should be granted:
        * TEST_DATA_REVIEWING_READ
        * TEST_DATA_REVIEWING_WRITE
    * A new role, DevOps, should be created, assignable at CLIENT level, and granted:
        * TEST_DATA_LOADING_READ
        * TEST_DATA_LOADING_WRITE
        * TEST_DATA_REVIEWING_READ
        * TEST_DATA_REVIEWING_WRITE
        * REPORT_TEMPLATE_WRITE
        * (Optional) PIPELINE_READ/WRITE, TENANT_READ/WRITE
    * A new role, ISR_TEMPLATE_ADMIN, should be created, assignable at the CLIENT and STATE level, and granted:    
        * REPORT_TEMPLATE_WRITE
* [ ] (Optional) "Before" Smoke Test. You may want to go through some of the steps of the smoke test before doing the
upgrade, just to make sure any problems are new. This may require temporarily providing access for your QA volunteers.


### Quiesce the System
Before upgrading the system it must be made idle.

* [ ] Verify 3rd party data feeds are suspended.
* [ ] Set up static landing page and route all site traffic to it.
    * reporting.sbac.org should redirect to static landing page
    * import.sbac.org can return an error during the upgrade
* [ ] Disable any ops monitoring on RDW services.
* We're on the clock. Play the Jeopardy theme in a loop ...    
* [ ] Scale down all service deployments to have no pods. The first three deployments listed allow/cause data changes in
the system. It is suggested that scale them down, wait a few minutes to all migration to complete, then scale down the rest.
    ```bash
    # these allow/cause data changes
    kubectl scale deployment task-server-deployment --replicas=0
    kubectl scale deployment import-deployment --replicas=0
    kubectl scale deployment reporting-webapp-deployment --replicas=0
    # pause here for 2-3 minutes to allow processing and migration to complete 
    kubectl scale deployment package-processor-deployment --replicas=0
    kubectl scale deployment group-processor-deployment --replicas=0
    kubectl scale deployment exam-processor-deployment --replicas=0
    kubectl scale deployment migrate-olap-deployment --replicas=0
    kubectl scale deployment migrate-reporting-deployment --replicas=0
    kubectl scale deployment admin-service-deployment --replicas=0
    kubectl scale deployment aggregate-service-deployment --replicas=0
    kubectl scale deployment reporting-service-deployment --replicas=0
    kubectl scale deployment report-processor-deployment --replicas=0
    ```

### Backup
All cluster deployment and configuration is stored in version control, so nothing is necessary for that.

* [ ] Backup Aurora databases.
* [ ] Backup Redshift database.


### Upgrade

* [ ] Gentle reminder to start `screen` on the ops machine so steps may be run in parallel.
* [ ] Apply schema changes. In a multi-tenant installation, the schema changes must be applied for every
tenant and sandbox. The following instructions use two fake tenants, OT and TS, and one sandbox, OT_S001 
to demonstrate.
    * Get the latest version of the schema and check the state of the databases.
    ```bash
    # get latest version of the schema
    cd ../RDW_Schema
    git checkout master; git pull

    # test credentials and state of databases
    for s in _OT _TS _OT_S001; do ./gradlew -Pschema_suffix=$s \
      -Pdatabase_url="jdbc:mysql://rdw-opus-warehouse.cimuvo5urx1e.us-west-2.rds.amazonaws.com:3306/" -Pdatabase_user=root -Pdatabase_password=password \
      -Predshift_url=jdbc:redshift://rdw.cs909ohc4ovd.us-west-2.redshift.amazonaws.com:5439/opus -Predshift_user=root -Predshift_password=password \
      infoReporting infoWarehouse infoMigrate_olap infoReporting_olap; done
    ```
    * Continue, migrate reporting and warehouse, and clear the OLAP data.
        * Reporting OLAP.
        ```bash
        for s in _OT _TS _OT_S001; do ./gradlew -Pschema_suffix=$s \
          -Pdatabase_url="jdbc:mysql://rdw-opus-warehouse.cimuvo5urx1e.us-west-2.rds.amazonaws.com:3306/" -Pdatabase_user=root -Pdatabase_password=password \
          -Predshift_url=jdbc:redshift://rdw.cs909ohc4ovd.us-west-2.redshift.amazonaws.com:5439/opus -Predshift_user=root -Predshift_password=password \
          migrateReporting migrateWarehouse migrateMigrate_olap migrateReporting_olap
        ```
    * After migrating the reporting olap database you'll need to re-grant permissions because there are some new tables.
    You'll need to verify the user names by inspecting the configuration repo for each tenant and sandbox.
    ```sql
    \connect opus
    GRANT ALL ON ALL TABLES IN SCHEMA reporting_ot TO rdw_ot;
    GRANT ALL ON ALL TABLES IN SCHEMA reporting_ts TO rdw_ts;
    GRANT ALL ON ALL TABLES IN SCHEMA reporting_ot_s001 TO ots001;
    ```
* [ ] Wipe embargo tables?
    * TODO - this may have been handled by the flyway migration scripts, not sure yet.
* [ ] Redeploy ingest services but not the migrate services just yet.
    ```bash
    kubectl apply -f package-processor-service.yml
    kubectl apply -f exam-processor-service.yml
    kubectl apply -f group-processor-service.yml
    kubectl apply -f import-service.yml
    ```
* [ ] Reload the ELA subject file. This is necessary because it has additional metadata that needs to be ingested and migrated. The updated subject file can be found in this deploy folder.
    ```bash
    export ACCESS_TOKEN=`curl -s -X POST --data 'grant_type=password&username=rdw-ingest-opus@sbac.org&password=password&client_id=rdw&client_secret=password' 'https://sso.sbac.org/auth/oauth2/access_token?realm=/sbac' | jq -r '.access_token'`
    curl -X POST --header "Authorization: Bearer ${ACCESS_TOKEN}" -F file=@ELA_subject.xml https://import.sbac.org/subjects/imports
    ```    
* [ ] Redeploy the remaining ingest services.
    ```bash
    kubectl apply -f migrate-reporting-service.yml
    kubectl apply -f migrate-olap-service.yml
    kubectl apply -f task-service.yml
    ```
    The migrate services will take a little time to migrate the subject updates.
* [ ] Redeploy reporting services.
    ```bash
    cd ~/RDW_Deploy_Opus
    # reporting services
    kubectl apply -f aggregate-service.yml
    kubectl apply -f reporting-service.yml
    kubectl apply -f report-processor-service.yml
    kubectl apply -f admin-service.yml
    kubectl apply -f reporting-webapp.yml
   ```
Check the logs of the services.
* [ ] (Optional) Run data validation scripts. Once the data migration is complete (you can see this by monitoring the
log for the migrate-reporting and migrate-olap service), you may re-run the validation scripts.
    ```bash
    cd ../RDW_Schema
    git checkout master; git pull
    cd validation
    ./validate-migration.sh secrets/opus.sh reporting
    ```
* [ ] Miscellaneous cleanup
    * [ ] Restore traffic to site (from static web page)
    * [ ] Notify 3rd party to restore data feeds
    * [ ] Reenable any ops monitoring
    * [ ] Deploy new User Guide and Interpretive Guide
* [ ] Dataset changes. All datasets must be regenerated or migrated.
This can happen after the upgrade with the services running; any attempt to create a sandbox from an existing (old)
dataset will fail until the dataset is regenerated.
    * TODO - simplest is to regenerate datasets from existing (now migrated) deployments
* [ ] Embargo settings. 
    * TODO - depending on previous steps it may be necessary for DevOps to set embargo settings, not sure yet


### Smoke Test
Smoke 'em if ya got 'em.         





TODO - after making the additional permissions changes, update this section for:
    * REPORT_TEMPLATE_WRITE
Our permissions server UI was broken, so i made the changes directly to the permissions db:
```mysql
-- The goal is to create the new roles and permissions for RDW Phase 6.
--
-- All changes apply to only the Reporting component.
--
-- There are four new permissions for allowing changes to test data visibility.
-- One each for read/write of loading/reviewing state. This will replace the
-- existing EMBARGO_READ/WRITE permissions.
--
-- The existing EMBARGO_ADMIN role can be granted at State or District level.
-- It should be granted the (two) read/write reviewing permissions.
--
-- The existing SandboxDistrictAdmin role can be granted at District level.
-- It should be granted the (two) read/write reviewing permissions.
--
-- There is a new role, DevOps. It should be at Client level only.
-- It should be granted all four new permissions.

SELECT r.name as Role, p.name as Permission FROM permission_role pr
    LEFT JOIN role r on pr._fk_rid = r._id
    JOIN permission p on pr._fk_pid = p._id
    JOIN component c on pr._fk_cid = c._id
WHERE c.name = 'Reporting'
ORDER BY r._id, p._id;

SELECT _id FROM component WHERE name = 'Reporting';
-- 15
SELECT _id FROM role WHERE name = 'EMBARGO_ADMIN';
-- 39
SELECT _id FROM role WHERE name = 'SandboxDistrictAdmin';
-- 2000010
SELECT _id FROM entitytype WHERE name = 'Client';
-- 1


INSERT INTO permission (_id, name) VALUES
 (100, 'TEST_DATA_LOADING_WRITE'),
 (101, 'TEST_DATA_LOADING_READ'),
 (102, 'TEST_DATA_REVIEWING_WRITE'),
 (103, 'TEST_DATA_REVIEWING_READ');

INSERT INTO role (_id, name) VALUES
  (100, 'DevOps');

INSERT INTO role_entity (_fk_rid, _fk_etuk) VALUES
  (100, 'CLIENT');

INSERT INTO permission_role (_fk_cid, _fk_rid, _fk_pid) VALUES
  (15, 100, 100),
  (15, 100, 101),
  (15, 100, 102),
  (15, 100, 103),
  (15, 100, 2000006),
  (15, 100, 2000007),
  (15, 100, 2000008),
  (15, 100, 2000009),
  (15, 39, 102),
  (15, 39, 103),
  (15, 2000010, 102),
  (15, 2000010, 103);
```


### Migrate Scripts Notes
* TODO - clean these up based on experience upgrading staging/production environments.
My best guess is that this section will *not* apply to staging/production and can be deleted.

* Dev issue with warehouse/sql/V2_4_0_3__wer_purpose.sql
Not sure why, since i don't see any modification history, but i did see this:
```
Execution failed for task ':migrateWarehouse'.
> Error occurred while executing migrateWarehouse
  Validate failed: Migration checksum mismatch for migration version 2.4.0.3
  -> Applied to database : 707681325
  -> Resolved locally    : -1187190027
```

* warehouse/sql/V2_4_0_4__test_results_availability.sql
This script had to be modified well after the initial checkin. Check the schema_version
table: if the script was successful, you'll need to update the checksum. If the script
was not successful, then delete that row in the schema_version. 
The original checksum was `-385633165`.
The updated checksum is `-1901338595`.
