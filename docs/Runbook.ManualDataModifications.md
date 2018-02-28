## Runbook - Manual Data Modifications
**Intended Audience**: This document provides instructions for manual data modifications in the Reporting Data Warehouse. Knowledge of SQL and access to the production databases is required.
Operations will find this useful if and only if it is required to manually modify the data bypassing the import mechanism.

### Other Resources
1. [Import and Migrate](Runbook.migrate.md) 
2. [Bulk Delete Exams](Runbook.BulkDeleteExams.md)  

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

