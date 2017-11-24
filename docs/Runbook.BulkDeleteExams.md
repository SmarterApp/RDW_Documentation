## Bulk Delete Exams

**Intended Audience**: This document provides instructions for bulk deleting exams from the Reporting Data Warehouse. Knowledge of SQL and access to the production data warehouse is required.
Operations will find this useful if and only if a bulk delete operation is required.

Deleting exams includes:
* marking exams as deleted in the warehouse
* migrating (propagating) the changes to the reporting data mart(s)

> **NOTE**: Modifying a large volume of data needs to be done with the consideration of how data is [migrated](Runbook.migrate.md#modify-lots-of-content).
> Since migrating this changes may take time, it is **strongly advisable to perform this task during the maintenance window, and while the system is quiescent 
and the exam processors are paused.**.   

### Other Resources
1. [Import and Migrate](Runbook.migrate.md) - Operations and system administration may find this useful with manually adjusting or cleaning up data.
1. [Migrate Reporting](#migrate-reporting)
1. [Migrate OLAP](#migrate-olap)

### Warehouse exam data store
Test result data (aka exams) depends on the following data being pre-loaded:
* Assessment packages (CSV output from the tabulator)
* Organizations
* Accommodations
* Subject configuration data
Exams data tables enforce referential integrity to the above data.

####  How test results data is stored
The data source (in a table below) refers to the [Test Results Transmission Format](http://www.smarterapp.org/documents/TestResultsTransmissionFormat.pdf) data elements. 

Table   | Data Source       |  Comment  | 
-------------- | ----------- |---------- |
student |Examinee attributes that define a student: StudentIdentifier, FirstName, MiddleName, LastOrSurname, Birthdate, Sex and so on. | If a student is not found, a new entry is created. If a student exists, but some of the student's attributes are different, they are updated.
exam |Examinee and ExamineeRelationship attributes that define a student at a time of testing (GradeLevelWhenAssessed, IDEAIndicator, LEPStatus, Section504Status, and so on), as well as Opportunity's attributes, Opportunity's 'overall' ScaleScore and PerformanceLevel data|  ExamineeRelationship refer to the Organizations data.
exam_item | Items' attributes and data elements | Exam items reference Items definition pre-loaded from the Assessment package.
exam_available_accommodation |Opportunity's Accommodations | Depends on [SBAC Accessbility Accomodataion Configuration](https://github.com/SmarterApp/AccessibilityAccommodationConfigurations/tree/RDW_DataWarehouse) being preloaded. Only accommodations deninded by this configuration are stored. 
exam_claim_score |Opportunity's claims ScaleScore and PerformanceLevel data| Depends on the subject being pre-configured with its claim scores.

### Marking exams as deleted in the warehouse (aka soft-delete)
As defined in [Import and Migrate](Runbook.migrate.md#import-id), to soft-delete exams in the warehouse a corresponding `exam` record must be marked with `deleted = 1`.
   
### Migrating (propagating) the changes to the reporting data mart(s)
As defined in [Import and Migrate](Runbook.migrate.md#create-update), the data updates must adhere to the defined pattern in order for the migrate process to propagate the changes to the data mart(s). 

### Example SQL scripts and instructions

#### Before executing the scripts
1. Stop Exam Processors.
2. Execute validation script to reconcile warehouse and reporting data mart(s). The script could be found in [RDW_Schema](https://github.com/SmarterApp/RDW_Schema) under the `validation` folder.
3. Check if warehouse auditing is enabled and turn it off if you do not want this action to be audited.
```sql 
# to check
mysql> use warehouse;
mysql> SELECT value FROM setting WHERE name = 'AUDIT_TRIGGER_ENABLE';

# to turn audditing off
mysql> UPDATE setting SET value = 'FALSE' WHERE name = 'AUDIT_TRIGGER_ENABLE';
```
4. Find and copy one of the queries that matches your rules for the delete.

4.1 Delete based on the test administration year
```sql 
SELECT id FROM exam WHERE school_year = 2010;  -- replace with your administration year
```
4.2 Delete based on the date range of when the test result was completed
```sql 
SELECT id FROM exam WHERE completed_at >= '2017-03-15 09:12:14.729000' AND completed_at <= '2017-03-15 09:12:14.729000'  -- replace with your completed at dates
```
4.3 Delete based on a specific school
```sql 
SELECT e.id FROM exam e JOIN school s ON s.id = e.school_id WHERE s.natural_id = 'school_natural_id_here'; -- replace with your school id
```
4.4 Delete based on a specific assessment
```sql 
SELECT e.id FROM exam e JOIN asmt a ON a.id = e.asmt_id WHERE a.natural_id = 'asmt_natural_id_here'; -- replace with your assessment id
```
4.5 Delete based on the manner of administration (Standardized or Nonstandardized)
```sql 
SELECT e.id FROM exam e JOIN administration_condition a ON a.id = e.administration_condition_id WHERE a.code = 'SD'; -- use 'SD' for Standardized and 'NS' for Nonstandardized
```
4.6 Delete based on the Valid/Invalid
```sql 
SELECT e.id FROM exam e JOIN administration_condition a ON a.id = e.administration_condition_id WHERE a.code = 'Valid'; -- use 'Valid' for Valid and 'IN' for Invalid
```

5. Open SQL script for bulk delete exams (found in [RDW_Schema](https://github.com/SmarterApp/RDW_Schema) under the `warehouse` folder) and replace the placeholder in STEP 1 with the copied query.
Continue with the steps in this SQL file.
6. Verify that migrate(s) is/are running and wait for them to complete.

### Post-validating exam deletion
8. Execute validation script (from step 2 above) to reconcile warehouse and reporting data mart(s). 
9. Re-enable auditing if it was turned off in the steps above.
10. Re-start Exam Processors.

## Purging exams in the warehouse
> **NOTE: Directly modifying the database cares high risk. Refrain from doing it unless you have a strong reason.** 

**NOTE: this is not a standard operation and should only be pursued in extraordinary circumstances**

Purging exams includes:
* purging exams from the data mart(s).
* purging exam auditing data.
* purging exam from the warehouse.

### Purging from reporting data mart
If exams were deleted using soft-delete and migration (see above) there is no need to do this: the system has already deleted the records in the data mart.

To delete exams bypassing the soft-delete step, follow instructions for [Purging from data warehouse](#purging-from-warehouse). Skip step 3.3 since `reporting` data mart does not have this table.

### Purging from OLAP data mart
(TODO)

### Purging exam auditing data in the warehouse
(TODO)

<a name="purging-from-warehouse"></a>
### Purging from data warehouse

This assumes you know exam rules for delete. **Below example purges soft-deleted exams.**

#### Before purging the data
 
1. Count the number of records be deleted and verify it this matches your expectations.
```sql 
SELECT count(*)  FROM exam WHERE deleted = 1;
```
2. Count the number of records that should be left after you purge the data. Save this number for post-validation.
```sql 
SELECT count(*) FROM exam WHERE deleted != 1;
```
3. Delete exams related data: 
> **NOTE 1**: This deletes all data in one transaction. For a very large volume it is recommended to follow the approach defined in the bulk delete:
>  Each step should be repeated for every table below:
> 1. load exam ids to delete into a staging table
> 1. partition the staging table
> 1. delete by one partition at a time

> **NOTE 2**: This leaves out import records.

3.1 Delete items:
```sql 
DELETE ei FROM exam_item ei
  JOIN exam e ON e.id = ei.exam_id
WHERE e.deleted = 1;
```
3.2 Delete available accommodations:
```sql 
DELETE eaa FROM exam_available_accommodation eaa
  JOIN exam e ON e.id = eaa.exam_id
WHERE e.deleted = 1;
```
3.3 Delete claim scores:
```sql 
DELETE ecs FROM exam_claim_score ecs
  JOIN exam e ON e.id = ecs.exam_id
WHERE e.deleted = 1;
```
4. Delete exams:
```sql
DELETE e FROM exam e WHERE e.deleted = 1;
```

### Post-validating exam deletion
5. Count the number of records and compare it to the count from the step 2 above. The numbers must match.
```sql 
SELECT count(*) FROM exam;
```