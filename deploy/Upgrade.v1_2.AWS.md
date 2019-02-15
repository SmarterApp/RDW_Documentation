## Upgrade v1.2 <- v1.1

**Intended Audience**: this document provides detailed instructions for upgrading the [Reporting Data Warehouse (RDW)](../README.md) applications in an AWS environment from v1.1.x to v1.2. Operations and system administrators will find it useful.

It is assumed that either the [v1.1 Deployment Checklist](https://github.com/SmarterApp/RDW/blob/1.1/deploy/Deployment.AWS.md) or the combination of the [v1.0 Deployment Checklist](https://github.com/SmarterApp/RDW/blob/1.0/deploy/Deployment.AWS.md) & [v1.1 Upgrade](https://github.com/SmarterApp/RDW/blob/1.1/deploy/Upgrade.AWS.md) was used for the current installation. Please refer to that documentation for general guidelines.

> Although these are generic instructions, having an example makes it easier to document and understand. Throughout this document we'll use `opus` as the sample environment name; this might correspond to `staging` or `production` in the real world. We'll use `sbac.org` as the domain/organization. Any other ids, usernames or other system-generated values are products of the author's imagination. The reader is strongly encouraged to use their own consistent naming conventions. **Avoid putting real environment-specific details _especially secrets and sensitive information_ in this document.**

### Overview

This is the second significant upgrade to RDW. It adds "Phase 3" functionality:
* Extend Custom Aggregate Reports to include IABs
* Student Response Reports
* Configurable Subjects
* Configure Required/Optional Fields for Ingest Validation
* Teacher-Created Groups
* Unknown Gender
* English Language Acquisition Status

It involves some minor technical changes:
* Fix/change the memory settings for all apps
* Reorganize application and tenant properties


The high level steps for the upgrade include:
* [Notification](#notification)
* [Prep Work](#prep-work)
* [Quiesce the System](#quiesce-the-system)
* [Backup](#backup)
* [Upgrade](#upgrade)
* [Smoke Test](#smoke-test)

### Notification
Well in advance of this work:

* [ ] Message user base about the upgrade and extended maintenance window.
* [ ] Alert 3rd party data feeds about the extended maintenance window.


### Prep Work
The goal of this step is to make changes to everything that doesn't directly affect current operations, leaving the absolute minimum work required during the actual maintenance window.

* These instructions expect you to have access to the files in the RDW project, so clone that locally if needed.
    ```bash
    cd ~/git
    git clone https://github.com/SmarterApp/RDW.git
    ```
* [ ] Branch deployment and configuration repositories. All changes to these files will be made in the branch which can be quickly merged during the upgrade.
    ```bash
    cd ~/git
    git clone https://github.com/SmarterApp/RDW_Deploy_Opus.git
    git clone https://github.com/SmarterApp/RDW_Config_Opus.git

    cd RDW_Deploy_Opus
    git checkout master; git pull
    git checkout -b v1_2 master
    git push -u origin v1_2
    
    cd ../RDW_Config_Opus
    git checkout master; git pull
    git checkout -b v1_2 master
    git push -u origin v1_2
    ```
* [ ] Add a copy of this checklist to deployment and switch to that copy.
* [ ] Changes to deployment files in `RDW_Deploy_Opus`. There are sample deployment files in the `deploy` folder in the `RDW` repository; use those to copy and help guide edits.
    * Common services. These helper services require no changes.
        * rabbit service
        * configuration service
        * wkhtmltopdf service
    * Ingest services.
        * Change image version to `1.2.0-RELEASE` in the following files:
            * `import-service.yml`
            * `package-processor-service.yml`
            * `group-processor-service.yml`
            * `exam-processor-service.yml`
            * `migrate-olap-service.yml`
            * `migrate-reporting-service.yml`
            * `task-service.yml`
        * Note: the ingest services should not require changes to max heap size or container memory limits but please refer to the runbook (https://github.com/SmarterApp/RDW/blob/master/docs/Runbook.md#common) for guidance.
    * Reporting services. Only the image version needs to be changed.
        * Change image version to `1.2.0-RELEASE` in the following files:
            * `admin-service.yml`
            * `aggregate-service.yml`
            * `report-processor-service.yml`
            * `reporting-service.yml`
            * `reporting-webapp.yml`
        * Fix/change memory settings:
            * `admin-service.yml`
                * Reduce requests/limits memory to 500M (it was probably 750M or higher)
            * `aggregate-service.yml`
                * Set requests/limits memory to 1G
                * Use default heap size; remove any `MAX_HEAP_SIZE` environment variable.
            * `report-processor-service.yml`
                * Set requests/limits memory to at least 750M
                * Set max heap size by setting environment variable `MAX_HEAP_SIZE = "-Xmx500m"`
            * `reporting-service.yml`
                * Set requests/limits memory to at least 750M
                * Set max heap size by setting environment variable `MAX_HEAP_SIZE = "-Xmx500m"`
            * `reporting-webapp.yml`
                * Set requests/limits memory to at least 1G
    * Commit changes
        ```bash
        cd ~/git/RDW_Deploy_Opus
        git add *
        git commit -am "Changes for v1.2"
        git push 
        ```
* [ ] Changes to configuration files in `RDW_Config_Opus`. There are annotated configuration files in the `config` folder in the `RDW` repository; use those to help guide edits.
    * Common service properties.
        * Edit `application.yml`
            * Change `app:` to `reporting:`. Many of these have good defaults and are optional but it could have
                * school-year: 2018
                * client: SBAC
                * state.code: CA
                * state.name: California
            * Delete `tenant:` line (and move things around if necessary) so those properties are now under `reporting:`
                * Probably want to put `transfer-access-enabled: true` in here (and remove from other config files)
    * Ingest services.
        * Import service, edit `rdw-ingest-import-service.yml`
            * Set `security.permission-service.endpoint` (copy from reporting webapp config)
            * Remove `security.state` if present
        * Package processor, edit `rdw-ingest-package-processor.yml`
            * Add Amazon S3 credentials to configuration (copy from import-service)
    * Reporting services.
        * Admin service, edit `rdw-reporting-admin-service.yml`
            * Change `app:` to `reporting:`
        * Aggregate service, edit `rdw-reporting-aggregate-service.yml`
            * If `app.state` exists, change it to be under `reporting.state` (but this is probably specified in `application.yml`)
        * Report processor, edit `rdw-reporting-report-processor.yml`
            * If `app.state` exists, change it to be under `reporting.state` (but this is probably specified in `application.yml`)
            * Move any properties under `tenant:` to be under `reporting:`
        * Reporting service, edit `rdw-reporting-service.yml`
            * If `app.state` exists, change it to be under `reporting.state` (but this is probably specified in `application.yml`)
            * Move any properties under `tenant:` to be under `reporting:`
            * Move all properties under `app:` to `rdw-reporting-webapp.yml` and change to be under `reporting:`
            (when you're done there will be no properties under `app:` in the reporting service config file)
            * Add `spring.writable_datasource` and set url and credentials. Perhaps a copy from `spring.datasource` but make sure it is not read-only.
        * Reporting webapp, edit `rdw-reporting-webapp.yml`
            * Copy `app:` properties from `rdw-reporting-service.yml` and put them under `reporting:`
                * Simplify `analytics.trackingId` to `analytics-tracking-id`
                * Simplify `iris.vendorId` to `iris-vendor-id`
                * Retain `interpretive-guide-url`, `user-guide-url`
                * Retain `min-item-data-year`
                * Retain `report-languages`, `ui-languages`
            * If `app.state` exists, change it to be under `reporting.state` (but this is probably specified in `application.yml`)
            * Move any properties under `tenant:` to be under `reporting:`
            * Copy `app.iris.url` value to `zuul.routes.iris.url` then remove `app.iris.url`
            (when you're done there will be no properties under `app:` in this reporting webapp config file)
    * Commit changes
        ```bash
        cd ~/git/RDW_Config_Opus
        git add *
        git commit -am "Changes for v1.2"
        git push 
        ```
* [ ] Permissions
    * Because of a quirk in the permissions service there must be a permission associated with every role so, for the
    Reporting component add a `DATA_WRITE` permission and associate it with the role `ASMTDATALOAD`.
* [ ] Translations
    * Verify there are no custom entries in the reporting.translation table. If there are, create an `en.json` with the
    custom values. For example,
    ```json
    ...
    "html": {
      "system-news": "<h2 class=\"blue-dark h3 mb-md\">Note</h2><div class=\"summary-reports-container mb-md\"><p>Item level data and session IDs are not available for tests administered prior to the 2017-18 school year.</p></div>"
    },
    ```
* [ ] (Optional) Run data validation scripts. These scripts compare data between the warehouse and the data marts.
    * You'll need the version of RDW_Schema that was used to install the *current* installation; in this case it is
    probably the tagged 1.1.0 commit:
    ```bash
    # get v1.1 version of the schema
    cd ~/git/RDW_Schema
    git fetch --all --tags --prune
    git checkout 1.1.0
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
the system. It is suggested that scale them down, wait 5 minutes to all migration to complete, then scale down the rest. 
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
* [ ] Upgrade cluster. If the version of the cluster is old (< 1.8 at the time of this writing), consider upgrading it.
See [Deployment Checklist](./Deployment.AWS.md#upgrading_the_cluster) for details. Upgrading can take a long time
(perhaps an hour or more). This is a good opportunity to jump down a couple steps and start the schema migration.
* [ ] Upgrade cluster system services
    * Get latest heapster, tweak and apply
    ```bash
    cd ~/git/heapster
    # may need to discard changes to grafana spec file
    # git checkout -- deploy/kube-config/influxdb/grafana.yaml
    git pull
    # edit and tweak the spec file per deployment instructions
    vi deploy/kube-config/influxdb/grafana.yaml
    kubectl apply -f ~/git/heapster/deploy/kube-config/influxdb
    ```
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
    * After wiping the reporting olap database you'll need to re-grant permissions.
    I suspect only the `... ON ALL TABLES ...` commands are needed (because database and schema are not recreated).
    ```sql
	\connect opus
	GRANT ALL ON SCHEMA reporting to rdwopusingest;
    GRANT ALL ON ALL TABLES IN SCHEMA reporting TO rdwopusingest;
	GRANT ALL ON SCHEMA reporting to rdwopusreporting;
    GRANT ALL ON ALL TABLES IN SCHEMA reporting TO rdwopusreporting;
	ALTER USER rdwopusingest SET SEARCH_PATH TO reporting;
	ALTER USER rdwopusreporting SET SEARCH_PATH TO reporting;
    ```
* [ ] Merge deployment and configuration branches. This can be done via command line or via the repository UI (if you use the repository UI, make sure to checkout and pull the latest `master`). Here are the command line actions:
    ```bash
    cd ~/git/RDW_Deploy_Opus
    git checkout v1_2; git pull
    git checkout master
    git merge v1_2
    git push origin master
    git push origin --delete v1_2; git branch -d v1_2
    
    cd ~/git/RDW_Config_Opus
    git checkout v1_2; git pull
    git checkout master
    git merge v1_2
    git push origin master
    git push origin --delete v1_2; git branch -d v1_2
    ```
* (Optional) Although there should be no problem, now is an okay time to verify db connectivity/routing/permissions.
    ```bash
    kubectl run -it --rm --image=mysql:5.6 mysql-client -- mysql -h rdw-opus-warehouse.cimuvo5urx1e.us-west-2.rds.amazonaws.com -P 3306 -uusername -ppassword warehouse
    kubectl run -it --rm --image=mysql:5.6 mysql-client -- mysql -h rdw-opus-reporting.cimuvo5urx1e.us-west-2.rds.amazonaws.com -P 3306 -uusername -ppassword reporting

    kubectl run -it --rm --image=jbergknoff/postgresql-client psql postgresql://username:password@rdw.cs909ohc4ovd.us-west-2.redshift.amazonaws.com:5439/opus
    kubectl run -it --rm --image=jbergknoff/postgresql-client psql postgresql://username:password@rdw.cs909ohc4ovd.us-west-2.redshift.amazonaws.com:5439/opus
    ```
* [ ] Redeploy ingest services. Since we are loading subject definitions, start only the import service and package processor.
    ```bash
    cd ~/git/RDW_Deploy_Opus
    # ingest services
    kubectl apply -f package-processor-service.yml
    kubectl apply -f import-service.yml
    ```
* [ ] Import Math and ELA subject definitions.  The standard ELA and Math definition XML files can be found in this project in the `deploy` directory.
    * Import the Math and ELA definition XML files
        ```bash
        export ACCESS_TOKEN=`curl -s -X POST --data 'grant_type=password&username=rdw-ingest-opus@sbac.org&password=password&client_id=rdw&client_secret=password' 'https://sso.sbac.org/auth/oauth2/access_token?realm=/sbac' | jq -r '.access_token'`
        curl -X POST --header "Authorization: Bearer ${ACCESS_TOKEN}" -F file=@Math_subject.xml https://import.sbac.org/subjects/imports
        curl -X POST --header "Authorization: Bearer ${ACCESS_TOKEN}" -F file=@ELA_subject.xml https://import.sbac.org/subjects/imports
        ```
* [ ] Redeploy ingest services.
    ```bash
    cd ~/git/RDW_Deploy_Opus
    # ingest services
    kubectl apply -f exam-processor-service.yml
    kubectl apply -f group-processor-service.yml
    kubectl apply -f migrate-reporting-service.yml
    kubectl apply -f migrate-olap-service.yml
    kubectl apply -f task-service.yml
    ```
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
log for the migrate-reporting service), you may re-run the validation scripts.
    ```bash
    cd ~/git/RDW_Schema
    git checkout master; git pull
    cd validation
    ./validate-migration.sh secrets/opus.sh reporting
    ```
* [ ] Miscellaneous cleanup
    * [ ] Restore traffic to site (from static web page)
    * [ ] Notify 3rd party to restore data feeds
    * [ ] Reenable any ops monitoring (remember to clean up any admin-webapp monitoring)
    * [ ] Deploy new User Guide and Interpretive Guide

### Smoke Test
Smoke 'em if ya got 'em.         
