## Upgrade v2.0.0 <- v1.3.x

NOTE: v2.0.0 was labelled v1.4.0 during development, UAT, and initial deployment. Specifically v2.0.0 corresponds to v1.4.0-RC23 for ingest and v1.4.0-RC25 for reporting.

**Intended Audience**: this document provides detailed instructions for upgrading the [Reporting Data Warehouse (RDW)](../README.md) applications in an AWS environment from v1.3.x to v2.0.0. Operations and system administrators will find it useful.

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
    * Ingest services, change image version to `2.0.0-RELEASE` in the following files:
        * `import-service.yml`
        * `package-processor-service.yml`
        * `group-processor-service.yml`
        * `exam-processor-service.yml`
        * `migrate-olap-service.yml`
        * `migrate-reporting-service.yml`
        * `task-service.yml`
    * Increase the memory for `migrate-olap-service.yml` from 500M/600M to 800M/800M
    * Reporting services, change image version to `2.0.0-RELEASE` in the following files:
        * `admin-service.yml`
        * `aggregate-service.yml`
        * `report-processor-service.yml`
        * `reporting-service.yml`
        * `reporting-webapp.yml`
    * Commit changes
        ```bash
        cd ../RDW_Deploy_Opus
        git add *
        git commit -am "Changes for v2.0.0"
        git push 
        ```
* [ ] There are extensive configuration changes required. Generally this will affect database, archive, ... properties. No new information is needed, it will just be moved around.
    * `application.yml` - this now contains a lot more configuration since more properties are common amongst the services.
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
                EconomicDisadvantage: Disabled
                LimitedEnglishProficiency: Disabled
                MigrantStatus: Enabled
                EnglishLanguageAcquisitionStatus: Enabled
                PrimaryLanguage: Enabled
                Ethnicity: Enabled
                Gender: Disabled
                IndividualEducationPlan: Disabled
                MilitaryStudentIdentifier: Disabled
                Section504: Disabled
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
        * Modify any send-reconciliation-report archive senders. For example:
        ```
        - type: archive
          root: s3://rdw-opus-archive
          cloud:
            aws:
              # rdw-opus
               accessKey: AK...
                secretKey: '{cipher}ca36...'
              region:
                static: us-west-2
        ```
        becomes:
        ```
        - type: archive
          uri-root: s3://rdw-opus-archive
          s3-access-key: AK...
          s3-secret-key: '{cipher}ca36...'
          s3-region-static: us-west-2
        ```
    * `rdw-reporting-admin-service.yml`
        * Add the admin datasources, the details can be copied from the existing datasources
        ```
        spring:
          admin_warehouse_datasource:
            url-parts:
              hosts: rdw-aurora-opus-warehouse.cugsexobhx8t.us-west-2.rds.amazonaws.com:3306
            username: 
            password: '{cipher}...'
          admin_reporting_datasource:
            url-parts:
              hosts: rdw-aurora-opus-reporting.cugsexobhx8t.us-west-2.rds.amazonaws.com:3306
            username: 
            password: '{cipher}...'
          admin_olap_datasource:
            url-parts:
              hosts: rdw.cibkulpjrgtr.us-west-2.redshift.amazonaws.com:5439
              database: opus
            username: 
            password: '{cipher}...'
        ```
        * Remove reporting_datasource; copied to application.yml as reporting_ro.
        * Remove warehouse_datasource; copied to application.yml as warehouse_rw.
        * Remove archive properties; copied to application.yml.
        * Remove rabbitmq config if present.
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
        * (Optional) If sandbox datasets are available, add them. The label may be anything, the id should correspond to the folder in S3, e.g. `s3://rdw-opus-archive/sandbox-datasets/sbac-dataset`.
        ```
        sandbox-properties:
          sandboxDatasets:
            - label: SBAC Dataset
              id: sbac-dataset
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
        git commit -am "Changes for v2.0.0"
        git push
        ```
* [ ] Add new roles and permissions for the Reporting component in the permissions application.
    * Role: TENANT_ADMIN. Assignable at CLIENT level only.
        * Permissions: TENANT_READ, TENANT_WRITE
    * Role: PIPELINE_ADMIN. Assignable at STATE level only.
        * Permissions: PIPELINE_READ, PIPELINE_WRITE
    * Role: SandboxTeacher (PII_GROUP)
        * Permissions: GROUP_PII_READ, GROUP_READ
    * Role: SandboxSchoolAdmin (PII, CUSTOM_AGGREGATE_REPORTER)
        * Permissions: INDIVIDUAL_PII_READ, GROUP_PII_READ, GROUP_READ, CUSTOM_AGGREGATE_READ, 
    * Role: SandboxDistrictAdmin (PII, CUSTOM_AGGREGATE_REPORTER, GROUP_ADMIN, EMBARGO_ADMIN, INSTRUCTIONAL_RESOURCE_ADMIN)
        * Permissions: INDIVIDUAL_PII_READ, GROUP_PII_READ, CUSTOM_AGGREGATE_READ, GROUP_READ, GROUP_WRITE, EMBARGO_READ, EMBARGO_WRITE, INSTRUCTIONAL_RESOURCE_WRITE
* [ ] (Optional, as needed) Update IRiS files. This action is included as a reminder: if new assessment items are available, now is a good time to add the IRiS SAAIF files to the shared folder.        
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

#### Upgrade v2.0.0 (Sandbox Edition)
The Tenant/Sandbox admin functionality caused a delay of the initial 2.0.0 release.
If you are upgrading from 1.4.0-RC19 or earlier the following changes may have to be made:
* [ ] Verify the TENANT_ADMIN role and related permissions are configured
(these weren't required before the Sandbox functionality was added).
* [ ] Fix student-fields values. If the configuration files include `reporting.student-fields`
those values must be properly cased. Specifically 
    * `admin` -> `Admin`
    * `disabled` -> `Disabled`
    * `enabled` -> `Enabled`
* [ ] Remove `tenant-configuration-lookup` values if in any configuration files.
* [ ] Remove `state.code` and `state.name` from root `application.yml` (make sure they are set for all tenants and sandboxes)
* [ ] Remove `reporting.school-year` (make sure it is set for all tenants and sandboxes)
* [ ] If any Kubernetes deployments have the actuator port (8008) exposed, remove that. Check `admin-service.yml`, `aggregate-service.yml`, `report-processor-service.yml` and `reporting-service.yml`.
* [ ] Add `import-status-service` to `import-service.yml`
```
apiVersion: v1
kind: Service
metadata:
  name: import-status-service
spec:
  type: ClusterIP
  ports:
    - port: 80
      targetPort: 8008
      name: status
  selector:
    app: import-server
---
```
* [ ] Reorganize task-service update organizations configuration properties (might help to refer to the canonical rdw-ingest-task-service.yml)
    * In the tenant-specific application.yml
        * add taskUpdateOrganizationsImportServiceClient with oauth2 username/password copied from the root application.yml:
        ```
        taskUpdateOrganizationsImportServiceClient:
          tenants:
            CA:
              oauth2:
                username: tenant-user@example.com
                password: '{cipher}...'
        ``` 
    * In the root `rdw-ingest-task-service.yml`
        * move `task.update-organizations.art-client` -> `task-update-organizations-art-client`
        * move `task.update-organizations.import-service-client` -> `task-update-organizations-import-service-client`
        * remove username/password from `task-update-organizations-import-service-client.oauth2` (used in previous step)
* [ ] Reorganize task-service reconciliation report configuration properties (might help to refer to the canonical rdw-ingest-task-service.yml)
    * In the tenant-specific application.yml
        * copy archive settings from root application.yml and put it under taskSendReconciliationReport.
        It no longer uses `senders`: the log sender becomes `log: true`, the archive sender properties go under `archives`, and the ftp sender is no longer supported: 
        ```
        taskSendReconciliationReport:
          tenants:
            TS:
              log: true
              archives:
                - uri-root: s3://rdw-opus-archive
                  s3-access-key: AccessKey
                  s3-secret-key: '{cipher}...'
                  s3-region-static: us-west-2
                  s3-sse: AES256
        ``` 
    * In the root `rdw-ingest-task-service.yml`
        * remove `task.send-reconciliation-report.senders`
        * leave `task.send-reconciliation-report.cron`
        * leave `task.send-reconciliation-report.query`     
* [ ] Add ART client config to admin service, `rdw-reporting-admin-service.yml`. Use client-level credentials that work for all states.
```
art-client:
  url: https://art-deployment.opus.org/rest/
  oauth2:
    access-token-uri: https://sso-deployment.opus.org:443/auth/oauth2/access_token?realm=/sbac
    client-id: sbacdw
    client-secret: '{cipher}...'
    username: super-user@example.com
    password: '{cipher}...'
```
* [ ] Reminder to add data set declarations in `rdw-reporting-admin-service.yml` if there are any available. For example,
```
sandbox-properties:
  sandboxDatasets:
    - label: CA Dataset
      id: ca-dataset
    - label: SB Dataset (ELA, Math, Latin)
      id: sb-dataset
```
* [ ] Add admin datasources to `rdw-reporting-admin-service.yml`
You'll need to create users that have full schema creation permissions.
``` 
spring:
  admin_warehouse_datasource:
    url-parts:
      hosts: rdw-opus-warehouse.cimuvo5urx1e.us-west-2.rds.amazonaws.com:3306
    username: root
    password: '{cipher}...'
  admin_reporting_datasource:
    url-parts:
      hosts: rdw-opus-reporting.cimuvo5urx1e.us-west-2.rds.amazonaws.com:3306
    username: root
    password: '{cipher}...'
  admin_olap_datasource:
    url-parts:
      hosts: rdw.cs909ohc4ovd.us-west-2.redshift.amazonaws.com:5439
      database: opus
    username: root
    password: '{cipher}...'
```

#### Upgrade v2.0.0 (Okta Integration Edition)
Both Okta and OpenAM will be supported during a transitional phase, but require different
configuration in rdw-ingest-import-service.yml. 

OpenAM is still the default, so minimal configuration is needed. For example:

```
security:
  oauth2:
    token-info-url: https://sso.smarterbalanced.org:443/auth/oauth2/tokeninfo?access_token={access_token}
```

A bit more configuration is needed for Okta configuration. For example:

```
security:
  oauth2:
    provider: okta
    token-info-url: https://smarterbalanced.oktapreview.com/oauth2/auslw2qcsmsUgzsqr0h7
    audience: api://staging
    connection-timeout: 1000
```

The token-info-url will depend on where the Okta server is displayed and the ID of the
application. The audience will also vary by environment. The connection timeout is optional
and defaults to 1000 (ms). It should not be necessary to set it in most cases.
