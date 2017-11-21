## Runbook - Import and Migrate

**Intended Audience**: this document contains information on how data is handled for import and migrate in the [Reporting Data Warehouse](../README.md) (RDW). Additional runbook information is available for [Import](Runbook.md#import-service) and [Migrate](Runbook.md#migrate-reporting). Operations and system administration may find this useful to deal with manually adjusting or cleaning up data. 

### Warehouse
The warehouse database contains data from different data sources. Every data element loaded into the warehouse is associated with an **import content type** (defined in `import_content` table) and has an **import id**. 

#### Supported import content to table mapping
Content Type   | Table       |  Comment  | 
-------------- | ----------- |---------- |
n/a | asmt_type, subject, subject_claim_score import_content, import_status, import, language | Considered critical data. Must be pre-loaded as part of the initial schema set. Cannot be modified later.
CODES | administration_condition, common_core_standard, grade, completeness, ethnicity, gender, claim, depth_of_knowledge, math_practice, item_trait_score, target | Pre-loaded from SBAC blueprints and specifications. Allows for manual updates.
CODES | accommodation, accommodation_translation | Ingested using the [Import Service API](https://github.com/SmarterApp/RDW_Ingest/blob/develop/import-service/API.md) and [SBAC Accessbility Accomodataion Configuration](https://github.com/SmarterApp/AccessibilityAccommodationConfigurations/tree/RDW_DataWarehouse).
PACKAGE | **asmt**, asmt_score, item, item_common_core_standard, item_other_target | Ingested using the Import Service API and the output from the tabulator.
ORGANIZATION | **school**, school_group, district, district_group | Uploaded by the Update Organizations task.
EXAM | **exam**, student, exam_item, exam_available_accommodation, exam_claim_score | Ingested from TRTs.
GROUPS | **student_group**, student, student_group_membership, user_student_group | Uploaded via Group Management API. For `student` only student SSID is available from this source.

<a name="import-id"></a>
#### Import table and Import ID

The `import` table is the main control table for importing the data. In order for the system to function properly any data modifications **must** be recorded in the `import` table. Each content type has a main table associated with it, which is listed as the first table on the table list above, and "child" tables. The main tables have 3 common columns:

- **import_id**: id of the import record that created this entity and all its children.
- **update_import_id**: last import id that modified this entity or any of its children. For the newly created entity `update_import_id` and `import_id` are the same. Removing a child entry is considered an update to the main entity.
- **created**: timestamp when content was created.
- **updated**: timestamp when content was last updated.
- **deleted**: a ‘soft’ delete flag.

Tables of the content type CODES are different in a sense that there is no ‘main’ table. Changes to any of these tables are tracked via the same import content type.

### Reporting Database and Migrate Process
The reporting database is the data source for the customer-facing RDW web site. Data must never be manually loaded or modified in this database. Instead they must be migrated from the warehouse. 

There are core tables that are created as part of the initial schema and are not supported for modifications:

- asmt_type
- subject
- subject_claim_score
- exam_claim_score_mapping
- migrate

The `migrate` table is the main control table for the migrate. It stores the migrate status and the timestamp range of content handled by each migrate job. The migrate process is managed by the “migrate-reporting” service.
  
<a name="create-update"></a>
### Create/Update Data
Data shall be ingested into the system using the import mechanism where available. However, there may be rare situations where data must be created or updated manually. In these situations the general workflow is:
1. Create an import record to associate with the changes.
1. Make the content changes, setting the import_id or update_import_id to match.
1. Verify a good distribution of `updated` times for content.
1. Mark the import record as `PROCESSED` to trigger the migrate process.

> Only data changes are supported, **never** make structural table changes.

#### Modify any CODES tables
* Update data in the tables of entity type CODES.
* Insert an entry into import table with the ‘CODES’ content type:
```sql
mysql> USE warehouse;
mysql> INSERT INTO import(status, content, contentType, digest) VALUES (1, 3, 'initial load', 'initial load');
```
* The migrate will pick this up. It will migrate all tables from this category.

#### Modify one main table and any of its children
* Create an import id to associate with your changes. Use an appropriate content type:
```sql
mysql> USE warehouse;
mysql> INSERT INTO import (status, content, contentType, digest, creator) VALUES (0, 5, 'text/plain', left(uuid(), 8), 'dwtest@example.com');
mysql> SELECT LAST_INSERT_ID() into @importid;
```
* Make the content data modifications.
* When you are done, update the main table with the import id value: 
    * If you are creating a new main entity, set both `import_id` and `update_import_id` to the same value, `@importid`.
    * If you are modifying an existing main entity or making any changes to the child tables, set the main table `update_import_id` to the `@importid`.
    * If you are deleting a main entity, set its `deleted ` flag to 1 and `update_import_id` to `@importid`.
* To complete the process, change the status of the import to 1:
```sql
# trigger migration
mysql> UPDATE import SET status = 1 WHERE id = @importid;
```

<a name="modify-lots-of-content"></a>
#### Modify lots of content
Because the migrate process handles many import records in a single iteration, it is important to **not** associate too much content with a single import record. The safest rule is 1 import record per 1 content record; in some situations you may associate a few (3-10) content records with a single import record. 

The migrate process also uses timestamp ranges to partition work. If you are modifying lots of content, make sure all your import and content records have different times associated with them. This can be done by explicitly setting the created/updated fields for the import records, adding microsecond offsets to separate the values, for example:
```sql
-- tweak import record timestamps to have different values
SELECT max(id) into @maxImportId from import;
UPDATE import
SET status = 1,
  created = DATE_ADD(created, INTERVAL (@maxImportId -id)  MICROSECOND),
  updated = DATE_ADD(updated, INTERVAL (@maxImportId -id)  MICROSECOND)
WHERE status = 0
      and content = 1
      and batch = 'mybatch-2017-10-08';
      
-- use the import timestamps to set the the content timestamps
UPDATE exam e JOIN import i ON i.id = e.update_import_id
SET e.updated = i.updated
WHERE i.status = 1
      and content = 1
      and batch = 'mybatch-2017-10-08';
```

#### Modify more than one main table and its children
While the process is the same as modifying one main table, there are a few things to note:
* The import ids should be created in the order of the main tables dependencies. Here is the hierarchy starting for the least dependent:
    * CODES
    * ORGANIZATION, PACKAGE
    * GROUPS, EXAM
* The same import id may be reused for multiple main entities of the same content type. As described above, do not associate more than a few content records with a single import record.

### Other Resources
* [Bulk Delete Exams](Runbook.BulkDeleteExams.md) - more concrete examples on how to delete exams. 