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
    git checkout -b v1_4_0 master
    git push -u origin v1_4_0

    cd ../RDW_Config_Opus
    git checkout master; git pull
    git checkout -b v1_4_0 master
    git push -u origin v1_4_0
    ```
* [ ] Add a copy of this checklist to deployment and switch to that copy.
    ```bash
    cd ../RDW_Deploy_Opus
    cp ../RDW/Upgrade.v1_4_0.AWS.md .
    git add Upgrade.v1_4_0.AWS.md
    ```
* [ ] (Optional) It is a good idea to go through the rest of this document, updating the configuration, credentials, etc. to match the environment. If you don't you'll just have to do it as you go along and there will be an extra commit to do when merging the deploy branch.    
* [ ] Changes to deployment files in `RDW_Deploy_Opus`. There are sample deployment files in the `deploy` folder in the `RDW` repository; use those to copy and help guide edits.
    * Common services.
        * `configuration-service.yml`
            * Change the image version to `3.1.2-RELEASE`
            * Set the search path in the environment variables:
            ```
            env:
            - name: CONFIG_SERVICE_SEARCH_PATH
              value: "tenant*"
            ```
        * `rabbit-service.yml` - no change required
        * `wkhtmltopdf-service.yml` - no change required
    * Ingest services, change image version to `1.4.0-RC7` in the following files:
        * `import-service.yml`
        * `package-processor-service.yml`
        * `group-processor-service.yml`
        * `exam-processor-service.yml`
        * `migrate-olap-service.yml`
        * `migrate-reporting-service.yml`
        * `task-service.yml`
    * Increase the memory for `migrate-olap-service.yml` from 500M/600M to 800M/800M
    * Reporting services, change image version to `1.4.0-RC7` in the following files:
        * `admin-service.yml`
        * `aggregate-service.yml`
        * `report-processor-service.yml`
        * `reporting-service.yml`
        * `reporting-webapp.yml`
    * The services need to have their actuator port exposed. In `admin-service.yml`, `aggregate-service.yml`, `report-processor-service.yml` and `reporting-service.yml` change something like this:
    ```
    spec:
      ports:
      - port: 80
        targetPort: 8080
    ``` 
    to:
    ```
    spec:
      ports:
      - port: 80
        targetPort: 8080
        name: api
      - port: 8008
        targetPort: 8008
        name: config
    ```
    * Commit changes
        ```bash
        cd ../RDW_Deploy_Opus
        git add *
        git commit -am "Changes for v1.4.0"
        git push 
        ```
* [ ] There are extensive configuration changes required. Generally this will affect database, archive, ... properties. No new information is needed, it will just be moved around.
    * `application.yml` - this now contains a lot more configuration since we've made more properties common amongst the services.
        * The datasource server/db settings are common and should be put here. Copy from existing files.
            * `datasources.migrate_rw` from `spring.migrate_datasource` (look in `rdw-ingest-migrate-olap.yml`)
            * `datasources.olap_ro` from `spring.olap_datasource` (look in `rdw-reporting-aggregate-service.yml`)
            * `datasources.olap_rw` from `spring.olap_datasource` (look in `rdw-ingest-migrate-olap.yml`)
            * `datasources.reporting_ro` from `spring.reporting_datasource` (look in `rdw-reporting-admin-service.yml`)
            * `datasources.reporting_rw` from `spring.reporting_datasource` (look in `rdw-ingest-migrate-reporting.yml`)
            * `datasources.warehouse_rw` from `spring.warehouse_datasource` (look in `rdw-ingest-migrate-olap.yml`)
        ```
        datasources:
          migrate_rw:
            url-server: rdw-aurora-opus-reporting.cugsexobhx8t.us-west-2.rds.amazonaws.com:3306
          olap_ro:
            url-server: rdw.cibkulpjrgtr.us-west-2.redshift.amazonaws.com:5439
            url-db: opus
          olap_rw:
            url-server: rdw.cibkulpjrgtr.us-west-2.redshift.amazonaws.com:5439
            url-db: opus
          reporting_ro:
            url-server: rdw-aurora-opus-reporting.cugsexobhx8t.us-west-2.rds.amazonaws.com:3306
          reporting_rw:
            url-server: rdw-aurora-opus-reporting.cugsexobhx8t.us-west-2.rds.amazonaws.com:3306
          warehouse_rw:
            url-server: rdw-aurora-opus-warehouse.cugsexobhx8t.us-west-2.rds.amazonaws.com:3306
        ```
        * The archive settings are common and should be put here (look in `rdw-reporting-aggregate-service.yml`)
        ```
        archive:
          uri-root: s3://rdw-opus-archive
          s3-access-key: A...
          s3-secret-key: '{cipher}82...'
          s3-region-static: us-west-2
        ```
        * Add/move RabbitMQ configuration into the base `application.yml` file (look in `rdw-reporting-aggregate-service.yml`)
        ```
        spring:
          rabbitmq:
            host: rabbit-service
            username: guest
            password: '{cipher}8195fa959bd85f65b64a3f1cb910186ae626edd7a5bb3e0f6f3259265d89f599'
        ```
    * Tenant-specific `application.yml`. Create a tenant folder using the tenant state code, e.g. `tenant-OT`. Create a new `application.yml` file in that folder. This will be the location of the common configuration overrides for this tenant.
    ```bash
    cd ../RDW_Config_Opus
    mkdir tenant-OT
    touch tenant-OT/application.yml
    git add tenant-OT/application.yml
    ```
        * Add the tenant declaration properties:
        ```
        tenantProperties:
          tenants:
            OT:
              id: OT
              key: OT
              name: Other State
        ```
        * Add the common reporting overrides:
        ```
        reporting:
          tenants:
            OT:
              state:
                code: OT
                name: Other State
        ```
        * Add the datasource overrides. The database, username and password for every datasource should go here; copy from existing config files. For a single tenant system the values can be copied from the existing configuration, there is no need to rename databases with tenant discriminating prefixes/suffixes.
            * `datasources.migrate_rw` from `spring.migrate_datasource` (look in `rdw-ingest-migrate-olap.yml`)
            * `datasources.olap_ro` from `spring.olap_datasource` (look in `rdw-reporting-aggregate-service.yml`)
            * `datasources.olap_rw` from `spring.olap_datasource` (look in `rdw-ingest-migrate-olap.yml`)
            * `datasources.reporting_ro` from `spring.reporting_datasource` (look in `rdw-reporting-admin-service.yml`)
            * `datasources.reporting_rw` from `spring.reporting_datasource` (look in `rdw-ingest-migrate-reporting.yml`)
            * `datasources.warehouse_rw` from `spring.warehouse_datasource` (look in `rdw-ingest-migrate-olap.yml`)
        ```
        datasources:
          migrate_rw:
            tenants:
              OT:
                urlParts:
                  database: migrate_olap
                username: ...
                password: '{cipher}53...'
          olap_ro:
            tenants:
              OT:
                username: ...
                password: '{cipher}53...'
          olap_rw:
            tenants:
              OT:
                username: ...
                password: '{cipher}53...'
          reporting_ro:
            tenants:
              OT:
                urlParts:
                  database: reporting
                username: ...
                password: '{cipher}53...'
          reporting_rw:
            tenants:
              OT:
                urlParts:
                  database: reporting
                username: ...
                password: '{cipher}53...'
          warehouse_rw:
            tenants:
              OT:
                urlParts:
                  database: warehouse
                username: ...
                password: '{cipher}53...'
        ```
        * For a single tenant system, there is no need to provide an archive path prefix.
        * This is a good time to configure the student fields if necessary. For example:
        ```
        reporting:
          tenants:
            OT:
              student-fields:
                EconomicDisadvantage: disabled
                LimitedEnglishProficiency: disabled
                MigrantStatus: enabled
                EnglishLanguageAcquisitionStatus: enabled
                PrimaryLanguage: enabled
                Ethnicity: enabled
                Gender: disabled
                IndividualEducationPlan: disabled
                MilitaryStudentIdentifier: disabled
                Section504: disabled
        ```
    * `rdw-ingest-exam-processor.yml`
        * Remove datasource; copied to application.yml as warehouse_rw.
        * Remove rabbitmq config if present.
        * The exam XSLT has been replaced with the ingest pipeline script.
            * Remove the `transformations.exam` property
        * File may be empty.
    * `rdw-ingest-group-processor.yml`
        * Remove datasource; copied to application.yml as warehouse_rw.
        * Remove archive properties; copied to application.yml.
        * Remove rabbitmq config if present.
        * File may be empty.
    * `rdw-ingest-import-service.yml`
        * Remove datasource; copied to application.yml as warehouse_rw.
        * Remove archive properties; copied to application.yml.
        * Remove rabbitmq config if present.
    * `rdw-ingest-migrate-olap.yml`
        * Remove migrate_datasource; copied to application.yml as migrate_rw.
        * Remove olap_datasource; copied to application.yml as olap_rw.
        * Remove warehouse_datasource; copied to application.yml as warehouse_rw.
        * Remove archive properties; copied to application.yml.
    * `rdw-ingest-migrate-reporting.yml`
        * Remove reporting_datasource; copied to application.yml as reporting_rw.
        * Remove warehouse_datasource; copied to application.yml as warehouse_rw.
        * File may be empty.
    * `rdw-ingest-package-processor.yml`
        * Remove datasource; copied to application.yml as warehouse_rw.
        * Remove archive properties; copied to application.yml.
        * Remove rabbitmq config if present.
        * File may be empty.
    * `rdw-ingest-task-service.yml`
        * Remove datasource; copied to application.yml as warehouse_rw.
        * The task service is not multi-tenant aware as of 1.4.0-RC7 so comment out all task configurations.
        * TODO
    * `rdw-reporting-admin-service.yml`
        * Remove reporting_datasource; copied to application.yml as reporting_ro.
        * Remove warehouse_datasource; copied to application.yml as warehouse_rw.
        * Remove archive properties; copied to application.yml.
        * Remove rabbitmq config if present.
        * Add the tenant configuration service urls.
        ```
        tenant-configuration-lookup:
          services:
            # talking to itself
            admin-service:
              url: http://localhost:8008
            aggregate-service:
              url: http://aggregate-service:8008
            report-processor:
              url: http://report-processor-service:8008
            reporting-service:
              url: http://reporting-service:8008
        ```
        * Add the tenant configuration persistence properties. The credentials must allow updates (i.e. not read-only)
        The config repo name can probably be found in the `configuration-service.yml` (in RDW_Deploy_Opus).
        ```
        tenant-configuration-persistence:
          local-repository-path: /tmp/rdw_config
          remote-repository-uri: https://gitlab.com/rdw_config_opus.git
          git-username: ...
          git-password: '{cipher}96...'
          author: "Zaphod Beeblebrox"
          author-email: "zaphodb@example.com"
        ```
    * `rdw-reporting-aggregate-service.yml`
        * Remove olap_datasource; copied to application.yml as olap_ro.
        * Remove archive properties; copied to application.yml.
        * Remove rabbitmq config if present.
        * Remove the following properties from `app.aggregate-reports` (summative assessments are now enabled)
            * `assessment-types`, `statewide-user-assessment-types`, `state-aggregate-assessment-types`
    * `rdw-reporting-report-processor.yml`
        * Remove datasource; copied to application.yml as reporting_ro.
        * Remove archive properties; copied to application.yml.
        * Remove rabbitmq config if present.
        * Remove `task.remove-stale-reports`.
    * `rdw-reporting-service.yml`
        * Remove datasource; copied to application.yml as reporting_ro.
        * Remove writable_datasource; copied to application.yml as reporting_rw.
        * Remove archive properties; copied to application.yml.
    * `rdw-reporting-webapp.yml`
        * Remove datasource; copied to application.yml as reporting_ro.
        * Remove `cloud.aws` properties if present.
    * Remove the xslt folder (and exam.xsl) if present
    * There may be new or modified localization overrides, this is a good time to create or modify the `i18n/en.json` file.
    * Commit changes
        ```bash
        cd ../RDW_Config_Opus
        git add *
        git commit -am "Changes for v1.4.0"
        git push
        ```
* [ ] TODO - Update IRiS files        
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
    cd ../RDW_Schema
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
    cd ../RDW_Schema
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
        * Reporting OLAP.
        ```bash
        ./gradlew -Pdatabase_url="jdbc:mysql://rdw-opus-warehouse.cimuvo5urx1e.us-west-2.rds.amazonaws.com:3306/" -Pdatabase_user=root -Pdatabase_password=password \
          -Predshift_url=jdbc:redshift://rdw.cs909ohc4ovd.us-west-2.redshift.amazonaws.com:5439/opus -Predshift_user=root -Predshift_password=password \
          migrateMigrate_olap migrateReporting_olap
        ```
    * After migrating the reporting olap database you'll need to re-grant permissions because there are some new tables.
    ```sql
    \connect opus
    GRANT ALL ON ALL TABLES IN SCHEMA reporting TO rdwopusingest;
    GRANT ALL ON ALL TABLES IN SCHEMA reporting TO rdwopusreporting;
    ```
* [ ] Merge deployment branch. This can be done via command line or via the repository UI (if you use the repository UI, make sure to checkout and pull the latest `master`). Here are the command line actions:
    ```bash
    cd ../RDW_Deploy_Opus
    # (if this document has been updated during the process you'll want to commit it (on the branch) before these steps)
    git checkout v1_4_0; git pull
    git checkout master
    git merge v1_4_0
    git push origin master
    git push origin --delete v1_4_0; git branch -d v1_4_0
    
    cd ../RDW_Config_Opus
    git checkout v1_4_0; git pull
    git checkout master
    git merge v1_4_0
    git push origin master
    git push origin --delete v1_4_0; git branch -d v1_4_0
    ```
* (Optional) Although there should be no problem, now is an okay time to verify db connectivity/routing/permissions.
    ```bash
    kubectl run -it --rm --image=mysql:5.6 mysql-client -- mysql -h rdw-opus-warehouse.cimuvo5urx1e.us-west-2.rds.amazonaws.com -P 3306 -uusername -ppassword warehouse
    kubectl run -it --rm --image=mysql:5.6 mysql-client -- mysql -h rdw-opus-reporting.cimuvo5urx1e.us-west-2.rds.amazonaws.com -P 3306 -uusername -ppassword reporting

    kubectl run -it --rm --image=jbergknoff/postgresql-client psql postgresql://username:password@rdw.cs909ohc4ovd.us-west-2.redshift.amazonaws.com:5439/opus
    kubectl run -it --rm --image=jbergknoff/postgresql-client psql postgresql://username:password@rdw.cs909ohc4ovd.us-west-2.redshift.amazonaws.com:5439/opus
    ```
* [ ] Redeploy the configuration service, and wait for it to fully come up
    ```bash
    cd ~/RDW_Deploy_Staging
    kubectl apply -f configuration-service.yml
    # wait for configuration-deployment to come up (`watch -n3 kubectl get po`)
    ```    
* [ ] Redeploy ingest services but not the migrate services just yet.
    ```bash
    kubectl apply -f package-processor-service.yml
    kubectl apply -f exam-processor-service.yml
    kubectl apply -f group-processor-service.yml
    kubectl apply -f import-service.yml
    ```
* [ ] Reload the subject files. This is necessary because they have additional metadata that needs to be ingested and migrated. The updated subject files can be found in this deploy folder.
    ```bash
    export ACCESS_TOKEN=`curl -s -X POST --data 'grant_type=password&username=rdw-ingest-opus@sbac.org&password=password&client_id=rdw&client_secret=password' 'https://sso.sbac.org/auth/oauth2/access_token?realm=/sbac' | jq -r '.access_token'`
    curl -X POST --header "Authorization: Bearer ${ACCESS_TOKEN}" -F file=@Math_subject.xml https://import.sbac.org/subjects/imports
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

### Smoke Test
Smoke 'em if ya got 'em.         


### Hotfixes

#### Upgrade v1.4.x
