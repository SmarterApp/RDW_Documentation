## Upgrade v1.1 <- v1.0

**Intended Audience**: this document provides detailed instructions for upgrading the [Reporting Data Warehouse (RDW)](../README.md) applications in an AWS environment from v1.0.x to v1.1. Operations and system administrators will find it useful.

It is assumed that the [Deployment Checklist](Deployment.AWS.md) was used for the installation of v1.0. Please refer to that documentation for general guidelines.

> Although these are generic instructions, having an example makes it easier to document and understand. Throughout this document we'll use `opus` as the sample environment name; this might correspond to `staging` or `production` in the real world. We'll use `sbac.org` as the domain/organization. Any other ids, usernames or other system-generated values are products of the author's imagination. The reader is strongly encouraged to use their own consistent naming conventions. **Avoid putting real environment-specific details _especially secrets and sensitive information_ in this document.**

#### NOTES from discussion with SmarterBalanced

Schedule
* 2/12-2/13 - Jeff/Mark prep work; do staging with pre-release; finalize this document
* 2/14-2/16 - SB travel and offsite
* 2/14 - UAT
* 2/19 - Presidents Day. Woot, go presidents!
* 2/20-2/23 - refine document; deal with staging issues; remaining production prep
* 2/?? - Brandt/Brad messaging to users that system will be down 3/3/18 - 3/4/18, available morning 3/5
* 2/?? - Louis/Mark schedule with ETS to suspend test result feed: stop 3/3/18 morning, resume and test 3/5/18 (?)
* 2/28/18 - deliver artifacts
* 3/2/18 (late) or 3/3/18 - stop feed (Louis/Mark), put up static status page (Jeff/Alex)
* 3/3/18 - backup Aurora (Jeff)
* 3/3/18 - maintenance window 3:00-7:00pm (pac)


### Overview

This is the first major update to RDW. It adds "Phase 2" functionality:
* Custom Aggregate Reports.
* Key / Distractor Analysis and Writing Trait Scores
* State/District-provided Instructional Links
* District/School groups (from ART).

It involves some significant architectural changes:
* Introduces Redshift as a data mart for aggregate reporting.
* Significant changes to the reporting services.
* Combine webapp applications (i.e. admin-webapp goes away)

Because there are schema changes, the data warehouse and reporting data mart must be migrated. This may take a couple hours depending on the amount of data involved. Combined with the non-trivial changes to the system configuration, this means the upgrade process may take 2-4 hours. It is important to alert the user base and any 3rd party data feeds of the extended downtime required for this upgrade.

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

* [ ] Branch deployment and configuration repositories. All changes to these files will be made in the branch which can be quickly merged during the upgrade.
	```bash
	cd ~/git/../RDW_Deploy_Opus
	git checkout master; git pull
	git checkout -b v11 master
	git push -u origin v11
	
	cd ../RDW_Config_Opus
	git checkout master; git pull
	git checkout -b v11 master
	git push -u origin v11
	```
* [ ] Add a copy of this checklist to deployment and switch to that copy.
* [ ] Changes to deployment files. There are sample deployment files in the `deploy` folder in this repository; use those to copy or help guide edits.
	* Common services. These helper services require no changes.
		* rabbit service
		* configuration service
		* wkhtmltopdf service
	* [ ] Ingest services. For most of these, only the image version needs to be changed. There is one new ingest service.
		* [ ] Change image version to `1.1.0-RELEASE` in the following files:
			* `import-service.yml`
			* `package-processor-service.yml`
			* `group-processor-service.yml`
			* `exam-processor-service.yml`
			* `migrate-reporting-service.yml`
			* `task-service.yml`
		* [ ] Copy `migrate-olap-service.yml` and adjust if necessary (rare).
	* [ ] Reporting services. Because of architecture changes in this release, there are more changes to make for these services.
		* [ ] Rename `reporting-service.yml` to `reporting-webapp.yml` and edit (compare to file in the `deploy` folder to help with this):
			* Change service name and app selector to `reporting-webapp`
			* Change deployment name to `reporting-webapp-deployment`
			* Change deployment spec app to `reporting-webapp`
			* Change image version to `1.1.0-RELEASE`
			* Reduce cpu/memory resources to `300m`/`750M`
			* Remove `volumeMounts` and `volumes` (copy over to new `reporting-service.yml`, next step)
		* [ ] Copy (new) `reporting-service.yml`
			* Add `volumeMounts` and `volumes` from old file (from previous step).
		* [ ] Copy `aggregate-service.yml` and adjust if necessary (rare).
		* [ ] Copy (replace) `admin-service.yml` and adjust if necessary (rare). This completely replaces the old admin service.
		* [ ] Edit `report-processor-service.yml` and set image version to `1.1.0-RELEASE`.
	* Commit changes
		```bash
		cd ~/git/../RDW_Deploy_Opus
		git commit -am "Changes for v1.1"
		git push 
		```
* [ ] Changes to configuration files. There are annotated configuration files in the `config` folder in this repository; use those to help guide edits.
	* [ ] Ingest services. 
		* [ ] Import service, edit `rdw-ingest-import-service.yml`
			* replace `spring.datasource.url` with `spring.datasource.url-server`
		* [ ] Package processor, edit `rdw-ingest-package-processor.yml`
			* replace `spring.datasource.url` with `spring.datasource.url-server`
		* [ ] Group processor, edit `rdw-ingest-group-processor.yml`
			* replace `spring.datasource.url` with `spring.datasource.url-server`
		* [ ] Exam processor, edit `rdw-ingest-exam-processor.yml`
			* replace `spring.datasource.url` with `spring.datasource.url-server`
		* [ ] Migrate reporting, edit `rdw-ingest-migrate-reporting.yml`
			* replace `spring.reporting_datasource.url` with `spring.reporting_datasource.url-server`
			* replace `spring.warehouse_datasource.url` with `spring.warehouse_datasource.url-server`
		* [ ] Task service, edit `rdw-ingest-task-service.yml`
			* replace `spring.datasource.url` with `spring.datasource.url-server`
			* for `task.update-organizations` add `groups-of-districts-url` and `groups-of-schools-url`
		* [ ] Migrate olap, copy `rdw-ingest-migrate-olap.yml` and edit
			* Configure `spring.migrate_datasource` copy from migrate-reporting `warehouse_datasource`
			* Configure `spring.warehouse_datasource`, copy from migrate-reporting `warehouse_datasource`
			* Configure S3 archive, copy from import-service `archive`
			* Configure redshift role - TBD
			* Configure `spring.olap_datasource` - TBD
	* [ ] Reporting services.
		* [ ] Admin service, copy `rdw-reporting-admin-service.yml` and edit
			* Configure `spring.rabbitmq`, copy from `rdw-admin-webapp.yml`
			* Configure `archive`, copy from `rdw-admin-webapp.yml`
			* Configure `spring.warehouse_datasource`, copy from `rdw-admin-webapp.yml`
			* Configure `spring.reporting_datasource`, copy from `rdw-reporting-webapp.yml`
			* Configure `app.state`
		* [ ] Aggregate service, copy `rdw-reporting-aggregate-service.yml` and edit
			* Configure `spring.rabbitmq`, copy from admin service
			* Configure `app.archive`, copy from admin service
			* Configure `app.state`
			* Configure `spring.olap_datasource` - TBD
		* [ ] Reporting service, copy `rdw-reporting-service.yml` and edit
			* Configure `app.iris.vendorId`, copy from `rdw-reporting-webapp.yml`
			* Configure `artifacts`, copy from `rdw-reporting-webapp.yml`
			* Configure `spring.datasource`, copy from `rdw-reporting-webapp.yml`
			* Configure `cloud`, copy from `rdw-reporting-report-processor.yml` `app.archive.cloud` (note different property prefix)
			* Configure `tenant.transfer-access-enabled`
			* Configure `app.state`
		* [ ] Report processor, edit `rdw-reporting-report-processor.yml`
			* Configure `spring.rabbitmq`, copy from admin service
			* Configure `tenant.transfer-access-enabled`
			* Move `app.datamart.datasource` to `spring.datasource` and replace `spring.datasource.url` with `spring.datasource.url-server`
			* Configure `task.remove-stale-reports` task
			* Configure `app.state`
		* [ ] Reporting webapp, edit `rdw-reporting-webapp.yml`
			* Configure `zuul.routes`, copy from annotated sample
			* Configure `tenant.transfer-access-enabled`
			* Configure `app.state`
			* Remove `app.admin-webapp-url`
			* Remove `app.reportservice`
			* Remove `app.wkhtmltopdf`
			* Remove `artifacts`
			* Move `app.datamart.datasource` to `spring.datasource` and replace `spring.datasource.url` with `spring.datasource.url-server`
		* Remove `rdw-admin-webapp.yml`
		* Remove `rdw-reporting-admin-webapp.yml` (rare, will exist only if interim builds were deployed)
	* Commit changes
		```bash
		cd ~/git/../RDW_Config_Opus
		git commit -am "Changes for v1.1"
		git push 
		```		
* [ ] Update ops system. 
    * [ ] Install `psql`.
* [ ] Configure Redshift cluster. Redshift is expensive. Have to decide whether to share a single cluster between your environments.
	* Obviously this work will affect configuration files from previous steps.
* [ ] Roles / Permissions
	* Create new permissions:
	* Create new role: 
	* Add roles and permissions
	* Chain admin roles to PII users


### Quiesce the System
Before upgrading the system it must be made idle.

* [ ] Verify 3rd party data feeds are suspended.
* [ ] Set up static landing page and route all site traffic to it.
	* reporting.sbac.org should redirect to static landing page
	* admin.sbac.org should go away
	* import.sbac.org can return an error during the upgrade
* [ ] Scale down all service deployments to have no pods. 
	```bash
	kubectl scale deployment import-deployment --replicas=0
	kubectl scale deployment package-processor-deployment --replicas=0
	kubectl scale deployment group-processor-deployment --replicas=0
	kubectl scale deployment exam-processor-deployment --replicas=0
	kubectl scale deployment migrate-reporting-deployment --replicas=0
	kubectl scale deployment task-server-deployment --replicas=0
	kubectl scale deployment admin-deployment --replicas=0
	kubectl scale deployment reporting-deployment --replicas=0
	kubectl scale deployment report-processor-deployment --replicas=0
	```
* We're on the clock. Play the Jeopardy theme in a loop ...	

### Backup
All cluster deployment and configuration is stored in version control, so nothing is necessary for that.

* [ ] Backup Aurora databases.


### Upgrade

* [ ] Delete current deployments as necessary. For many of the services, only the configuration has changed. However, some of the services have significant changes that require completely recreating them. NOTE: this is being done with the spec files **before** merging the branch.
	```bash
	kubectl delete -f admin-service.yml
	kubectl delete -f reporting-service.yml
	```
TODO - this will wipe the reporting service load balancer; that okay, did we customize it at all?
* [ ] Upgrade cluster (?)
* [ ] Increase cluster size
* [ ] Apply schema changes.
	```bash
	# get latest version of the schema
	cd ~/git/../RDW_Schema
	git checkout master; git pull
	
	# test credentials and state of warehouse, then migrate it (this may take a while)
	gradle --Pdatabase_url="jdbc:mysql://rdw-aurora-warehouse-[aws-randomization]:3306/" \
   		-Pdatabase_user=user -Pdatabase_password=password infoWarehouse
	gradle --Pdatabase_url="jdbc:mysql://rdw-aurora-warehouse-[aws-randomization]:3306/" \
   		-Pdatabase_user=user -Pdatabase_password=password migrateWarehouse	
   # test credentials and state of reporting, then migrate it (this may take a while)
   gradle -Pdatabase_url="jdbc:mysql://rdw-aurora-reporting-[aws-randomization]:3306/" \
   		-Pdatabase_user=user -Pdatabase_password=password infoReporting
   gradle -Pdatabase_url="jdbc:mysql://rdw-aurora-reporting-[aws-randomization]:3306/" \
   		-Pdatabase_user=user -Pdatabase_password=password migrateReporting

	# create olap data mart
    gradle -Predshift_url=jdbc:redshift://rdw-[aws-randomization]:5439/opus \
    	-Predshift_user=user -Predshift_password=password \
    	-Pdatabase_url=jdbc:mysql://rdw-aurora-warehouse-[aws-randomization]:3306 \
    	-Pdatabase_user=user -Pdatabase_password=password \
    	migrateReporting_olap migrateMigrate_olap
	```
* [ ] Merge deployment and configuration branches. This can be done via command line or via the repository UI; if you use the repository UI, make sure to checkout and pull the latest `master`. Here are the command line actions:
	```bash
	cd ~/git/../RDW_Deploy_Opus
	git checkout v11; git pull
	git checkout master
	git merge v11
	git push origin master
	git push origin -d v11; git branch -d v11
	
	cd ~/git/../RDW_Config_Opus
	git checkout v11; git pull
	git checkout master
	git merge v11
	git push origin master
	git push origin -d v11; git branch -d v11
	```
* [ ] Redeploy services
	```bash
	cd ~/git/../RDW_Deploy_Opus
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
* [ ] Route 53
	* redo for reporting webapp (pretty sure it will get deleted)
	* remove for admin webapp
* [ ] Load new instructional resources	
	```bash
	TODO - sql command to run SQL; probably also need the actual SQL
	```
* [ ] Reload assessment packages. The tabulator output has been updated to include additional assessment data (especially for items). This upgrade supports updating assessment packages. So, submit the new tabulator output for all assessments. Note: the assessment labels are still wrong in the tabulator output so the `label_assessments` script must be rerun as well.
	```bash
	TODO - curl commands to POST CSVs
	```
	```sql
	TODO - mysql command for running script
	```
* [ ] Miscellaneous cleanup
	* [ ] Remove admin-webapp stuff from OpenAM
	* 

### Smoke Test
Smoke 'em if ya got 'em.		 

		 


