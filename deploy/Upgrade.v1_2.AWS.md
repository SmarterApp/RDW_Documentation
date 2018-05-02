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

TODO - Because there are schema changes, the data warehouse and reporting data mart must be migrated. This may take a couple hours depending on the amount of data involved. Combined with the non-trivial changes to the system configuration, this means the upgrade process may take 2-4 hours. It is important to alert the user base and any 3rd party data feeds of the extended downtime required for this upgrade.

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
    * [ ] Ingest services.
        * [ ] Change image version to `1.2.0-RELEASE` in the following files:
            * `import-service.yml`
            * `package-processor-service.yml`
            * `group-processor-service.yml`
            * `exam-processor-service.yml`
            * `migrate-olap-service.yml`
            * `migrate-reporting-service.yml`
            * `task-service.yml`
        * Note: the ingest services should not require changes to max heap size or container memory limits
    * [ ] Reporting services. Only the image version needs to be changed.
        * [ ] Change image version to `1.2.0-RELEASE` in the following files:
            * `admin-service.yml`
            * `aggregate-service.yml`
            * `report-processor-service.yml`
            * `reporting-service.yml`
            * `reporting-webapp.yml`
        * [ ] Fix/change memory settings:
            * `admin-service.yml`
                * Reduce requests/limits memory to 500M (it was probably 750M or higher)
            * `aggregate-service.yml`
                * Set requests/limits memory to at least 800M
                * Set max heap size by setting environment variable `MAX_HEAP_SIZE = "-Xmx600m"`
            * `report-processor-service.yml`
                * Set requests/limits memory to at least 750M
                * Set max heap size by setting environment variable `MAX_HEAP_SIZE = "-Xmx500m"`
            * `reporting-service.yml`
                * Set requests/limits memory to at least 750M
                * Set max heap size by setting environment variable `MAX_HEAP_SIZE = "-Xmx500m"`
            * `reporting-webapp.yml`
                * Set requests/limits memory to at least 1G
    * [ ] Commit changes
        ```bash
        cd ~/git/RDW_Deploy_Opus
        git add *
        git commit -am "Changes for v1.2"
        git push 
        ```
* [ ] Changes to configuration files in `RDW_Config_Opus`. There are annotated configuration files in the `config` folder in the `RDW` repository; use those to help guide edits.
    * [ ] Common service properties.
        * Edit `application.yml`
            * Change `app:` to `reporting:`.  <-- seriously?! look into this
            * Set `app.client` to `SBAC` (this is the client-id). Set `app.state.code` to `CA`.
            * Delete `tenant:` line (and move things around if necessary) so those properties are now under `reporting:`
    * [ ] Ingest services.
        * [ ] Import service, edit `rdw-ingest-import-service.yml`
            * Set `security.permission-service.endpoint` (copy from reporting webapp config)
        * [ ] Migrate Olap service, edit `rdw-ingest-migrate-olap.yml`
            * TODO - change batch size?
    * [ ] Reporting services.
        * [ ] Admin service, edit `rdw-reporting-admin-service.yml`
            * Change `app:` to `reporting:`
        * [ ] Aggregate service, edit `rdw-reporting-aggregate-service.yml`
            * If `app.state` exists, change it to be under `reporting.state` (but this is probably specified in `application.yml`)
        * [ ] Report processor, edit `rdw-reporting-report-processor.yml`
            * If `app.state` exists, change it to be under `reporting.state` (but this is probably specified in `application.yml`)
            * Move any properties under `tenant:` to be under `reporting:`
        * [ ] Reporting service, edit `rdw-reporting-service.yml`
            * If `app.state` exists, change it to be under `reporting.state` (but this is probably specified in `application.yml`)
            * Move any properties under `tenant:` to be under `reporting:`
            * Move all properties under `app:` to `rdw-reporting-webapp.yml` and change to be under `reporting:`
            (when you're done there will be no properties under `app:` in the reporting service config file)
        * [ ] Reporting webapp, edit `rdw-reporting-webapp.yml`
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
    TODO - perhaps fix the code
* [ ] Translations
    * Verify there are no custom entries in the reporting.translation table. If there are, create an `en.json` with the
    custom values. For example,
    ```json
    ...
    "html": {
      "system-news": "<h2 class=\"blue-dark h3 mb-md\">Note</h2><div class=\"summary-reports-container mb-md\"><p>Item level data and session IDs are not available for tests administered prior to the 2017-18 school year.</p></div>"
    },
    ```
* [ ] TODO - any other prep

* [ ] Run data validation scripts. These scripts compare data between the warehouse and the data marts.
    * TODO - describe this
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

* [ ] Get the latest deploy repo
    ```bash
    cd ~/git/RDW_Deploy_Opus
    git checkout master; git pull
    ```
* [ ] Upgrade cluster. If the version of the cluster is old (< 1.7 at the time of this writing), consider upgrading it.
    ```bash
    kops upgrade cluster --name awsopus.sbac.org --state s3://kops-awsopus-sbac-org-state-store --yes
    ```

* [ ] TODO - figure out how to handle migration of IABs and Longitudinal fact tables to the olap data mart
    * Recommend wiping the olap data mart during the upgrade and allowing the migrate to go; it will take an hour or two to migrate everything.
    If we take that approach we should wipe the Redshift data before applying schema changes.

* [ ] Apply schema changes. If the warehouse and reporting databases are separate it will be more efficient to run the migration tasks in parallel. Use multiple terminal sessions (or `screen`) and run them at the same time.
   * Estimated schema changes run time:
    * [ ] TODO: Aurora/reporting (from AWS DEV reporting.schema_version) : 
    ```mysql
    select version, script, execution_time/1000/60 from reporting.schema_version where script like 'V1_2%'`;
    ```
   |version  | script       |  execution_time (in min) | 
   |-------------- | ----------- |---------- |
   |1.2.0.0|V1_2_0_0__elas_gender.sql|25.22918333|
   |1.2.0.1|V1_2_0_1__add_user_report_type.sql|0.00156667 |
   |1.2.0.2|V1_2_0_2__elas_date.sql|25.28641667|
   |1.2.0.3|V1_2_0_3__iab_dashboard_exam_index.sql|50.55576667|
   |1.2.0.4|V1_2_0_4__add_grade_order.sql|0.00538333|
   
   * [ ] TODO: Aurora/warehouse :
   
    |version  | script       |  execution_time (in min) | 
    |-------------- | ----------- |---------- |
    |1.2.0.0|V1_2_0_0__elas_gender.sql|44.80201667|
    |1.2.0.1|V1_2_0_1__elas_audit.sql|0.01543333|
    |1.2.0.2|V1_2_0_2__ccs_for_summatives.sql|0.00030000|
    |1.2.0.3|V1_2_0_3__add_grade_order.sql|0.00291667|

   * [ ] Redshift : TODO

    ```bash
    # get latest version of the schema
    cd ~/git/RDW_Schema
    git checkout master; git pull
    
    # test credentials and state of databases
    ./gradlew -Pdatabase_url="jdbc:mysql://rdw-aurora-warehouse-[aws-randomization]:3306/" -Pdatabase_user=user -Pdatabase_password=password infoWarehouse
    ./gradlew -Pdatabase_url="jdbc:mysql://rdw-aurora-reporting-[aws-randomization]:3306/" -Pdatabase_user=user -Pdatabase_password=password infoReporting
    TODO - redshift
    
    # migrate warehouse (this may take a while)
    ./gradlew -Pdatabase_url="jdbc:mysql://rdw-aurora-warehouse-[aws-randomization]:3306/" -Pdatabase_user=user -Pdatabase_password=password migrateWarehouse    
    # migrate reporting (this may take a while)
    ./gradlew -Pdatabase_url="jdbc:mysql://rdw-aurora-reporting-[aws-randomization]:3306/" -Pdatabase_user=user -Pdatabase_password=password migrateReporting
    # migrate olap (this may take a while)
    TODO - redshift
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
* [ ] Redeploy services. 
    ```bash
    cd ~/git/RDW_Deploy_Opus
    # ingest services
    kubectl apply -f exam-processor-service.yml
    kubectl apply -f group-processor-service.yml
    kubectl apply -f package-processor-service.yml
    kubectl apply -f import-service.yml
    kubectl apply -f migrate-reporting-service.yml
    kubectl apply -f migrate-olap-service.yml
    kubectl apply -f task-service.yml
    # reporting services
    kubectl apply -f admin-service.yml
    kubectl apply -f aggregate-service.yml
    kubectl apply -f reporting-service.yml
    kubectl apply -f report-processor-service.yml
    kubectl apply -f reporting-webapp.yml
   ```
Check the logs of the services.
* [ ] Load data - TODO
* [ ] Run data validation scripts.
    * TODO - instructions
* [ ] Miscellaneous cleanup
    * [ ] Restore traffic to site (from static web page)
    * [ ] Notify 3rd party to restore data feeds
    * [ ] Reenable any ops monitoring (remember to clean up any admin-webapp monitoring)
    * [ ] Deploy new User Guide and Interpretive Guide

### Smoke Test
Smoke 'em if ya got 'em.         
