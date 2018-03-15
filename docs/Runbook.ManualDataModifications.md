## Runbook - Manual Data Modifications

**Intended Audience**: This document provides instructions for manual data modifications in the [Reporting Data Warehouse](../README.md) (RDW). Additional information is available in the main [Runbook](Runbook.md) and for the related issues of [Import and Migrate](Runbook.migrate.md) and [Bulk Delete Exams](Runbook.BulkDeleteExams.md). Knowledge of SQL and access to the production databases is required. Operations will find this useful if and only if it is required to manually modify the data bypassing the import mechanism.

### Create/Update Data
Data shall be ingested into the system using the import mechanism where available. However, there may be rare situations where data must be created or updated manually. In these situations the general workflow is:
1. Create an import record to associate with the changes.
1. Make the content changes, setting the import_id or update_import_id to match.
1. Verify a good distribution of `updated` times for content.
1. Mark the import record as `PROCESSED` to trigger the migrate process.

> Only data changes are supported, **never** make structural table changes.

#### Modify any CODES tables
* Update data in the tables of entity type CODES.
* Insert an entry into import table with the `CODES` content type:
```sql
mysql> USE warehouse;
mysql> INSERT INTO import(status, content, contentType, digest) VALUES (1, 3, 'initial load', 'initial load');
```
* The migrate will pick this up. It will migrate all tables from this category.

#### Modify EMBARGO settings
* Update/insert data in the EMBARGO tables.
* Insert an entry in to the import table with the `EMBARGO` content type:
```sql
mysql> USE warehouse;
mysql> INSERT INTO import(status, content, contentType, digest) VALUES (1, 6, 'embargo', 'embargo-settings-2017-12-19');
```
* The migrate will pick this up. It will migrate the embargo settings for all organizations.

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
#### Update assessment item common core standards
> **Assumption**:Common core standards are loaded into warehouse and do not require modifications
##### Prerequisites
1. Assessments natural ids. Ex.:`(SBAC)SBAC-IAB-FIXED-G11E-Perf-Explanatory-Marshmallow-Winter-2016-2017` 
1. Items natural ids (`bankKey` and `key` separated by `-`). Ex.:`200-62023` 
1. Common core standards subject and natural ids. Ex. of the natural id:`1.G.1`. Common core standards are stored in the `common_core_standard` table per subject. Subjects are stored in the `subject` table.
> **Note**: An item could be stored as part of one or multiple assessment(s). Both identities are required.

##### Create an import id to associate with your changes
```sql
mysql> USE warehouse;
mysql> INSERT INTO import (status, content, contentType, digest, creator)
        VALUES (
          0, -- started the update
          (SELECT id FROM import_content WHERE name = 'PACKAGE'),
          'manual item cc update',
          (SELECT CONCAT('item cc upd ', DATE_FORMAT(NOW(), '%Y-%m-%d %T'))), -- make it unique by adding time
          USER()
        );
mysql> SELECT LAST_INSERT_ID() into @importId;
```
##### Repeat for each assessment 
1. Identify an assessment record to be modified
```sql
SELECT id INTO @asmtId FROM asmt WHERE natural_id = 'replace with asmt natural id here';
```
###### Repeat for each item within the assessment
1.1. Identify an item record to be modified
```sql
SELECT id INTO @itemId FROM item WHERE natural_id = 'replace with item natural id here' AND asmt_id = @asmtId;
```
1.2. Identify a common core standard record to be modified, replace X with the subject id.
```sql
SELECT id INTO @ccId FROM common_core_standard WHERE natural_id = 'replace with common core standard natural id here' AND subject_id = X;
```
1.3. Add or deleted the common core standard to/from the item within the assessment

To add:
```sql
INSERT INTO item_common_core_standard (item_id, common_core_standard_id) VALUES
  (@itemId, @ccId);
```
To delete
```sql
DELETE FROM item_common_core_standard WHERE item_id = @itemId AND common_core_standard_id =  @ccId;
```
If you have more items for this assessment, repeat the process for each item.
##### Update assessment
```sql
UPDATE asmt SET update_import_id = @importId WHERE id = @asmtId;
```
If you have more assessments to modify, repeat the process for each assessment.
###### Finalize and trigger the migration
```sql
UPDATE import SET status = 1 WHERE id = @importId;
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

