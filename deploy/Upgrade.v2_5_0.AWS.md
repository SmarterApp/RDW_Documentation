## Upgrade v2.5.0 <- v2.4.0

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
    git checkout -b v2_5_0 master
    git push -u origin v2_5_0

    cd ../RDW_Config_Opus
    git checkout master; git pull
    git checkout -b v2_5_0 master
    git push -u origin v2_5_0
    ```
* [ ] Add a copy of this checklist to deployment and switch to that copy.
    ```bash
    cd ../RDW_Deploy_Opus
    cp ../RDW/Upgrade.v2_5_0.AWS.md .
    git add Upgrade.v2_5_0.AWS.md
    ```
* [ ] (Optional) It is a good idea to go through the rest of this document, updating the configuration, credentials, etc. to match the environment. If you don't you'll just have to do it as you go along and there will be an extra commit to do when merging the deploy branch.    
* [ ] Changes to deployment files in `RDW_Deploy_Opus`. There are sample deployment files in the `deploy` folder in the `RDW` repository; use those to copy and help guide edits.
    * The vendor upgraded Kubernetes during this upgrade and so had to update `deployment` definitions for all deployments.
        * `apiVersion: apps/v1`
        * add `spec.selector.matchLabels` to match the label in `spec.template.metadata.labels`
    * Change image versions to 2.5.0-RELEASE
    * Commit changes
        ```bash
        cd ../RDW_Deploy_Opus
        git add *
        git commit -am "Changes for v2.5.0"
        git push 
        ```
* [ ] Add new roles and permissions for the Reporting component in the permissions application.
    * There are new permissions to add:
        * TEST_DATA_LOADING_READ
        * TEST_DATA_LOADING_WRITE
    * In addition, there are permissions whose names must be changed:
        * EMBARGO_READ changes to TEST_DATA_REVIEWING_READ
        * EMBARGO_WRITE changes to TEST_DATA_REVIEWING_WRITE
    * The four permissions named above must be assigned to the DevOps role. (The renamed ones may
      already be assigned to DevOps, depending on how the renaming was done.)
    * The EMBARGO_ADMIN role should be renamed to EMBARGO_ADMIN_RELEASED, but retain its permissions, i.e., 
      the newly renamed TEST_DATA_REVIEWING_READ and TEST_DATA_REVIEWING_WRITE.
    * A new role EMBARGO_ADMIN, should be created to replace the old EMBARGO_ADMIN that has been renamed.
      It should be assignable at the state and district levels and granted just one permission: 
        * TEST_DATA_REVIEWING_READ
    
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
* [ ] Redeploy ingest services but not the migrate services just yet.
    ```bash
    kubectl apply -f package-processor-service.yml
    kubectl apply -f exam-processor-service.yml
    kubectl apply -f group-processor-service.yml
    kubectl apply -f import-service.yml
    ```
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
    * [ ] Re-enable any ops monitoring
    * [ ] Deploy new User Guide and Interpretive Guide
* [ ] Dataset changes. All datasets must be regenerated or migrated.
This can happen after the upgrade with the services running; any attempt to create a sandbox from an existing (old)
dataset will fail until the dataset is regenerated.
    * simplest is to regenerate datasets from existing (now migrated) deployments
* [ ] Embargo settings. The data migration scripts set all previous years results for all subjects to "Released", 
and all current year results to "Loading". If that is not correct, a DevOps user should use the admin UI to set 
the desired embargo settings.


### Smoke Test
Smoke 'em if ya got 'em.         
