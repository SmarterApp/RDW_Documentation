## Upgrade v1.4.0 <- v1.3.x

**Intended Audience**: this document provides detailed instructions for upgrading the [Reporting Data Warehouse (RDW)](../README.md) applications in an AWS environment from v1.3.x to v1.4.0. Operations and system administrators will find it useful.

It is assume that the official deployment and upgrade instructions were used for the current installation. Please refer to that documentation for general guidelines.

> Although these are generic instructions, having an example makes it easier to document and understand. Throughout this document we'll use `opus` as the sample environment name; this might correspond to `staging` or `production` in the real world. We'll use `sbac.org` as the domain/organization. Any other ids, usernames or other system-generated values are products of the author's imagination. The reader is strongly encouraged to use their own consistent naming conventions. **Avoid putting real environment-specific details _especially secrets and sensitive information_ in this document.**

### Overview

This is an upgrade to RDW that requires extensive configuration changes and minor database modification. It will require a non-trivial amount of preparation time. Once prepared, the upgrade itself will not be too long.

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
    cd ~/git
    git clone https://github.com/SmarterApp/RDW.git
    ```
* [ ] Branch the deployment and configuration repositories. All changes to these files will be made in the branch which can be quickly merged during the upgrade.
    ```bash
    cd ~/git
    git clone https://github.com/SmarterApp/RDW_Deploy_Opus.git
    git clone https://github.com/SmarterApp/RDW_Config_Opus.git

    cd RDW_Deploy_Opus
    git checkout master; git pull
    git checkout -b v1_4_0 master
    git push -u origin v1_4_0

    cd ../RDW_Config_Opus
    git checkout master; git pull
    git checkout -b v1_4_0 master
    git push -u origin v1_4_0
    ```
* [ ] Add a copy of this checklist to deployment and switch to that copy.
* [ ] Changes to deployment files in `RDW_Deploy_Opus`. There are sample deployment files in the `deploy` folder in the `RDW` repository; use those to copy and help guide edits.
    * Common services.
        * `configuration-service.yml` - change the image version to ???
        * `rabbit-service.yml` - no change required
        * `wkhtmltopdf-service.yml` - no change required
    * Ingest services, change image version to `1.4.0-RELEASE` in the following files:
        * `import-service.yml`
        * `package-processor-service.yml`
        * `group-processor-service.yml`
        * `exam-processor-service.yml`
        * `migrate-olap-service.yml`
        * `migrate-reporting-service.yml`
        * `task-service.yml`
    * Reporting services, change image version to `1.4.0-RELEASE` in the following files:
        * `admin-service.yml`
        * `aggregate-service.yml`
        * `report-processor-service.yml`
        * `reporting-service.yml`
        * `reporting-webapp.yml`
    * Commit changes
        ```bash
        cd ~/git/RDW_Deploy_Opus
        git add *
        git commit -am "Changes for v1.4.0"
        git push 
        ```
* [ ] There are extensive configuration changes required. Generally this will affect database, archive, ... properties. No new information is needed, it will just be moved around.
    * `application.yml`
        * TODO
    * `rdw-ingest-exam-processor.yml`
        * The datasource properties moved from `spring.datasource` to `datasources.warehouse_rw`
            * `url-server`, `username`, `password`
        * TODO
    * `rdw-igest-group-processor.yml`
        * The datasource properties moved from `spring.datasource` to `datasources.warehouse_rw`
            * `url-server`, `username`, `password`
        * The archive properties were changed
            * `archive.root` -> `archive.uri-root`
            * `archive.cloud.aws.credentials.accessKey` -> `archive.s3-access-key`
            * `archive.cloud.aws.credentials.secretKey` -> `archive.s3-secret-key`
            * `archive.cloud.aws.region.static` -> `archive.s3-region-static`
        * TODO
    * `rdw-ingest-import-service.yml`
        * The datasource properties moved from `spring.datasource` to `datasources.warehouse_rw`
            * `url-server`, `username`, `password`
        * The archive properties were changed
            * `archive.root` -> `archive.uri-root`
            * `archive.cloud.aws.credentials.accessKey` -> `archive.s3-access-key`
            * `archive.cloud.aws.credentials.secretKey` -> `archive.s3-secret-key`
            * `archive.cloud.aws.region.static` -> `archive.s3-region-static`
        * TODO
    * `rdw-ingest-migrate-olap.yml`
        * The warehouse datasource properties moved from `spring.warehouse_datasource` to `datasources.warehouse_rw`
            * `url-server`, `username`, `password`
        * The migrate datasource properties moved from `spring.migrate_datasource` to `datasources.migrate_rw`
            * `url-server`, `username`, `password`
        * The olap datasource properties moved from `spring.olap_datasource` to `datasources.olap_rw`
            * `url-server`, `url-db`, `username`, `password`
        * The archive properties were changed
            * `archive.root` -> `archive.uri-root`
            * `archive.cloud.aws.credentials.accessKey` -> `archive.s3-access-key`
            * `archive.cloud.aws.credentials.secretKey` -> `archive.s3-secret-key`
            * `archive.cloud.aws.region.static` -> `archive.s3-region-static`
        * TODO
    * `rdw-ingest-migrate-reporting.yml`
        * The warehouse datasource properties moved from `spring.warehouse_datasource` to `datasources.warehouse_rw`
            * `url-server`, `username`, `password`
        * The reporting datasource properties moved from `spring.reporting_datasource` to `datasources.reporting_rw`
            * `url-server`, `username`, `password`
        * TODO
    * `rdw-ingest-package-processor.yml`
        * The datasource properties moved from `spring.datasource` to `datasources.warehouse_rw`
            * `url-server`, `username`, `password`
        * The archive properties were changed
            * `archive.root` -> `archive.uri-root`
            * `archive.cloud.aws.credentials.accessKey` -> `archive.s3-access-key`
            * `archive.cloud.aws.credentials.secretKey` -> `archive.s3-secret-key`
            * `archive.cloud.aws.region.static` -> `archive.s3-region-static`
        * TODO
    * `rdw-ingest-task-service.yml`
        * The datasource properties moved from `spring.datasource` to `datasources.warehouse_rw`
            * `url-server`, `username`, `password`
        * TODO
    * `rdw-reporting-admin-service.yml`
        * The reporting datasource properties moved from `spring.reporting_datasource` to `datasources.reporting_ro`
            * `url-server`, `username`, `password`
        * The warehouse datasource properties moved from `spring.warehouse_datasource` to `datasources.warehouse_rw`
            * `url-server`, `username`, `password`
        * TODO
    * `rdw-reporting-aggregate-service.yml`
        * The datasource properties moved from `spring.olap_datasource` to `datasources.olap_ro`
            * `url-server`, `url-db`, `username`, `password`
        * The archive properties moved from `app.archive` to `archive` and were reorganized
            * `app.archive.root` -> `archive.uri-root`
            * `app.archive.cloud.aws.credentials.accessKey` -> `archive.s3-access-key`
            * `app.archive.cloud.aws.credentials.secretKey` -> `archive.s3-secret-key`
            * `app.archive.cloud.aws.region.static` -> `archive.s3-region-static`
        * The following properties moved from `app.aggregate-reports` to `aggregate-reporting`
            * `assessment-types`, `statewide-user-assessment-types`, `state-aggregate-assessment-types`
        * TODO
    * `rdw-reporting-report-processor.yml`
        * The reporting datasource properties moved from `spring.datasource` to `datasources.reporting_ro`
            * `url-server`, `username`, `password`
        * The archive properties moved from `app.archive` to `archive` and were reorganized
            * `app.archive.root` -> `archive.uri-root`
            * `app.archive.cloud.aws.credentials.accessKey` -> `archive.s3-access-key`
            * `app.archive.cloud.aws.credentials.secretKey` -> `archive.s3-secret-key`
            * `app.archive.cloud.aws.region.static` -> `archive.s3-region-static`
        * TODO
    * `rdw-reporting-service.yml`
        * The datasource properties moved from `spring.datasource` to `datasources.reporting_ro`
            * `url-server`, `username`, `password`
        * The writable datasource properties moved from `spring.writable_datasource` to `datasources.reporting_rw`
            * `url-server`, `username`, `password`
        * TODO
    * `rdw-reporting-webapp.yml`
        * The datasource properties moved from `spring.datasource` to `datasources.reporting_ro`
            * `url-server`, `username`, `password`
        * TODO
    * Commit changes
        ```bash
        cd ~/git/RDW_Config_Opus
        git add *
        git commit -am "Changes for v1.4.0"
        git push
        ```
* [ ] Add new roles and permissions for the Reporting component in the permissions application.
    * Role: TENANT_ADMIN. Assignable at CLIENT level only.
        * Permissions: TENANT_READ, TENANT_WRITE
    * Role: PIPELINE_ADMIN. Assignable at STATE level only.
        * Permissions: PIPELINE_READ, PIPELINE_WRITE
* [ ] (Optional) Run data validation scripts. These scripts compare data between the warehouse and the data marts.
    * You'll need the version of RDW_Schema that was used to install the *current* installation; in this case it is
    probably the tagged 1.3.0 commit:
    ```bash
    # get v1.3.0 version of the schema
    cd ~/git/RDW_Schema
    git fetch --all --tags --prune
    git checkout 1.3.0
    cd validation
    ```
    * If desired, there is a `README.md` that details the use of the script.
    * Create a secrets file for the environment, e.g. `secrets/opus.sh`, filling in the secrets:
    ```bash
    #!/usr/bin/env bash

    warehouse_host=rdw-aurora-opus-warehouse.cugsexobhx8t.us-west-2.rds.amazonaws.com
    warehouse_port=3306
    warehouse_schema=warehouse
    warehouse_user=
    warehouse_password=

    reporting_host=rdw-aurora-opus-reporting.cugsexobhx8t.us-west-2.rds.amazonaws.com
    reporting_port=3306
    reporting_schema=reporting
    reporting_user=
    reporting_password=

    reporting_olap_host=rdw.cibkulpjrgtr.us-west-2.redshift.amazonaws.com
    reporting_olap_port=5439
    reporting_olap_db=opus
    reporting_olap_user=
    reporting_olap_password=
    ```
    * Validate reporting data:
    ```bash
    ./validate-migration.sh secrets/opus.sh reporting
    ```
    This will produce a new `results-<date>` folder, with sub-folders. Each sub-folder has the results of a single
    validation. If there is a `warehouse_reporting.diff` file in a sub-folder, inspect it. There are some legitimate
    issues (e.g. cumulative rounding errors when summing std-err values) but, in general, there should be no diffs
    unless there have been manual tweaks to the data.
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
* [ ] Apply schema changes.
    * Get the latest version of the schema and check the state of the databases.
    ```bash
    # get latest version of the schema
    cd ~/git/RDW_Schema
    git checkout master; git pull

    # test credentials and state of databases
    ./gradlew -Pdatabase_url="jdbc:mysql://rdw-opus-warehouse.cimuvo5urx1e.us-west-2.rds.amazonaws.com:3306/" -Pdatabase_user=root -Pdatabase_password=password infoWarehouse

    ./gradlew -Pdatabase_url="jdbc:mysql://rdw-opus-reporting.cimuvo5urx1e.us-west-2.rds.amazonaws.com:3306/" -Pdatabase_user=root -Pdatabase_password=password infoReporting

    ./gradlew -Pdatabase_url="jdbc:mysql://rdw-opus-warehouse.cimuvo5urx1e.us-west-2.rds.amazonaws.com:3306/" -Pdatabase_user=root -Pdatabase_password=password \
      -Predshift_url=jdbc:redshift://rdw.cs909ohc4ovd.us-west-2.redshift.amazonaws.com:5439/opus -Predshift_user=root -Predshift_password=password infoMigrate_olap infoReporting_olap
    ```
    * Continue, migrating data. If the warehouse and reporting databases are separate it will be more efficient to run the migration tasks in parallel. Use multiple terminal sessions (or `screen`) and run them at the same time.
        * Warehouse
        ```bash
        ./gradlew -Pdatabase_url="jdbc:mysql://rdw-opus-warehouse.cimuvo5urx1e.us-west-2.rds.amazonaws.com:3306/" -Pdatabase_user=root -Pdatabase_password=password migrateWarehouse
        ```
        * Reporting
        ```bash
        ./gradlew -Pdatabase_url="jdbc:mysql://rdw-opus-reporting.cimuvo5urx1e.us-west-2.rds.amazonaws.com:3306/" -Pdatabase_user=root -Pdatabase_password=password migrateReporting
        ```
        * Reporting OLAP. We will be wiping out the olap data and remigrating everything.
        ```bash
        ./gradlew -Pdatabase_url="jdbc:mysql://rdw-opus-warehouse.cimuvo5urx1e.us-west-2.rds.amazonaws.com:3306/" -Pdatabase_user=root -Pdatabase_password=password \
          -Predshift_url=jdbc:redshift://rdw.cs909ohc4ovd.us-west-2.redshift.amazonaws.com:5439/opus -Predshift_user=root -Predshift_password=password \
          cleanMigrate_olap migrateMigrate_olap cleanReporting_olap migrateReporting_olap
        ```
    * After wiping the reporting olap database you'll need to re-grant table permissions.
    ```sql
	\connect opus
	GRANT ALL ON SCHEMA reporting to rdwopusingest;
    GRANT ALL ON ALL TABLES IN SCHEMA reporting TO rdwopusingest;
	GRANT ALL ON SCHEMA reporting to rdwopusreporting;
    GRANT ALL ON ALL TABLES IN SCHEMA reporting TO rdwopusreporting;
	ALTER USER rdwopusingest SET SEARCH_PATH TO reporting;
	ALTER USER rdwopusreporting SET SEARCH_PATH TO reporting;
    ```
* [ ] Merge deployment branch. This can be done via command line or via the repository UI (if you use the repository UI, make sure to checkout and pull the latest `master`). Here are the command line actions:
    ```bash
    cd ~/git/RDW_Deploy_Opus
    git checkout v1_3_0; git pull
    git checkout master
    git merge v1_3_0
    git push origin master
    git push origin --delete v1_3_0; git branch -d v1_3_0
    ```
* (Optional) Although there should be no problem, now is an okay time to verify db connectivity/routing/permissions.
    ```bash
    kubectl run -it --rm --image=mysql:5.6 mysql-client -- mysql -h rdw-opus-warehouse.cimuvo5urx1e.us-west-2.rds.amazonaws.com -P 3306 -uusername -ppassword warehouse
    kubectl run -it --rm --image=mysql:5.6 mysql-client -- mysql -h rdw-opus-reporting.cimuvo5urx1e.us-west-2.rds.amazonaws.com -P 3306 -uusername -ppassword reporting

    kubectl run -it --rm --image=jbergknoff/postgresql-client psql postgresql://username:password@rdw.cs909ohc4ovd.us-west-2.redshift.amazonaws.com:5439/opus
    kubectl run -it --rm --image=jbergknoff/postgresql-client psql postgresql://username:password@rdw.cs909ohc4ovd.us-west-2.redshift.amazonaws.com:5439/opus
    ```
* [ ] Redeploy ingest services.
    ```bash
    cd ~/git/RDW_Deploy_Opus
    # ingest services
    kubectl apply -f package-processor-service.yml
    kubectl apply -f exam-processor-service.yml
    kubectl apply -f group-processor-service.yml
    kubectl apply -f migrate-reporting-service.yml
    kubectl apply -f migrate-olap-service.yml
    kubectl apply -f import-service.yml
    kubectl apply -f task-service.yml
    ```
    The migrate-olap service will take some time (up to an hour) to migrate the data.
* [ ] Redeploy reporting services.
    ```bash
    cd ~/git/RDW_Deploy_Opus
    # reporting services
    kubectl apply -f admin-service.yml
    kubectl apply -f aggregate-service.yml
    kubectl apply -f reporting-service.yml
    kubectl apply -f report-processor-service.yml
    kubectl apply -f reporting-webapp.yml
   ```
Check the logs of the services.
* [ ] (Optional) Run data validation scripts. Once the data migration is complete (you can see this by monitoring the
log for the migrate-reporting and migrate-olap service), you may re-run the validation scripts.
    ```bash
    cd ~/git/RDW_Schema
    git checkout master; git pull
    cd validation
    ./validate-migration.sh secrets/opus.sh reporting
    ```
* [ ] Miscellaneous cleanup
    * [ ] Restore traffic to site (from static web page)
    * [ ] Notify 3rd party to restore data feeds
    * [ ] Reenable any ops monitoring
    * [ ] Deploy new User Guide and Interpretive Guide

### Smoke Test
Smoke 'em if ya got 'em.         


### Hotfixes

#### Upgrade v1.3.1

This upgrade (reporting) contains a hotfix for EBSR answer keys; and an increase to the district export timeout. The upgrade process is simple and requires no downtime:
* [ ] Edit `reporting-service.yml` and `report-processor-service.yml` and change image versions to `1.3.1-RELEASE`
* [ ] Perform a rolling update of those services:
```
kubectl apply -f reporting-service.yml
kubectl apply -f report-processor-service.yml
```

This upgrade (ingest) contains a bug fix for group migration after a large group file upload. The upgrade process is simple and requires no downtime:
* [ ] Edit `migrate-olap-service.yml` and `migrate-reporting-service.yml` and change image versions to `1.3.1-RELEASE`
* [ ] Perform a rolling update of those services. It is slightly more cautious to disable migration first:
```
# (Optional) Disable migrate reporting
kubectl exec -it migrate-reporting-deployment-[randomization] -- curl -X POST http://localhost:8008/migrate/pause
# Rolling update of services
kubectl apply -f migrate-olap-service.yml
kubectl apply -f migrate-reporting-service.yml
```
