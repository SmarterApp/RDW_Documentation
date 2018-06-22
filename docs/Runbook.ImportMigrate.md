## Runbook - Import and Migrate

**Intended Audience**: this document contains information on how data is handled for import and migrate in the [Reporting Data Warehouse](../README.md) (RDW). Additional runbook information is available for [Import](Runbook.md#import-service) and [Migrate](Runbook.md#migrate-reporting). Operations and system administration may find this useful to deal with manually adjusting or cleaning up data. 

### Warehouse
The warehouse database contains data from different data sources. Every data element loaded into the warehouse is associated with an **import content type** (defined in `import_content` table) and has an **import id**. 

#### Supported import content to table mapping
|Content Type   | Table       |  Comment  | 
|-------------- | ----------- |---------- |
| n/a | asmt_type, subject, subject_claim_score import_content, import_status, import, language | Considered critical data. Must be pre-loaded as part of the initial schema set. Cannot be modified later. |
| CODES | administration_condition, common_core_standard, grade, completeness, ethnicity, gender, claim, depth_of_knowledge, math_practice, item_trait_score, target | Pre-loaded from SBAC blueprints and specifications. Allows for manual updates. |
| CODES | accommodation, accommodation_translation | Ingested using the [Import Service API](https://github.com/SmarterApp/RDW_Ingest/blob/master/import-service/API.md) and [SBAC Accessbility Accomodataion Configuration](https://github.com/SmarterApp/AccessibilityAccommodationConfigurations/tree/RDW_DataWarehouse). |
| NORMS | percentile, percentile_score | [Norms Data](Norms.md) |
| EMBARGO | state_embargo, district_embargo | Embargo settings are edited using the Admin UI. |
| PACKAGE | **asmt**, asmt_score, item, item_common_core_standard, item_other_target | Ingested using the Import Service API and the output from the tabulator. |
| ORGANIZATION | **school**, school_group, district, district_group | Uploaded by the Update Organizations task. |
| EXAM | **exam**, student, exam_item, exam_available_accommodation, exam_claim_score | Ingested from TRTs. |
| GROUPS | **student_group**, student, student_group_membership, user_student_group | [Student Groups](StudentGroups.md) |

<a name="import-id"></a>
#### Import table and Import ID

The `import` table is the main control table for importing the data. In order for the system to function properly any data modifications **must** be recorded in the `import` table. Each content type has a main table associated with it, which is listed as the first table on the table list above, and "child" tables. The main tables have 3 common columns:

- **import_id**: id of the import record that created this entity and all its children.
- **update_import_id**: last import id that modified this entity or any of its children. For the newly created entity `update_import_id` and `import_id` are the same. Removing a child entry is considered an update to the main entity.
- **created**: timestamp when content was created.
- **updated**: timestamp when content was last updated.
- **deleted**: a ‘soft’ delete flag.

Tables of the content type CODES are different in a sense that there is no ‘main’ table. Changes to any of these tables are tracked via the same import content type.

### Reporting Databases and Migrate
The reporting databases are the data source for the customer-facing RDW web site. Data must never be manually loaded or modified in these databases. Instead they must be migrated from the warehouse. 

#### Reporting Databases
There are two reporting databases: 
1. [Individual Students Results (AKA Reporting)](https://github.com/SmarterApp/RDW_Schema/tree/master/reporting/sql)
2. [Aggregate Reporting (AKA OLAP Reporting)](https://github.com/SmarterApp/RDW_Schema/tree/master/reporting_olap/sql)

Reporting databases have core tables that are created as part of the initial schema and are not supported for modifications via migrate processes. 
They are defined in the corresponding `*.dml.sql` files. 

##### Reporting Database
The `migrate` table is the main control table for the reporting migrate. It stores the migrate status and the timestamp range of content handled by each migrate job. The migrate process is managed by the “migrate-reporting” service. The service migrates data in chunks using **created** and **updated** timestamps in the master tables. The starting and ending timestamps for each chunk are derived from the **import** table.

##### OLAP Reporting Database
OLAP migrate process is controlled by the `migrate` table in the [Migrate Olap DB] (https://github.com/SmarterApp/RDW_Schema/tree/master/migrate_olap/sql). It stores the migrate status and the timestamp range of content handled by each OLAP migrate job. The migrate process is managed by the “migrate-olap“ service. The service migrates data in chunks using **created** and **updated** timestamps in the master tables. The starting and ending timestamps for each chunk are derived from the **import** table.

### Other Resources
* [Manual Data Modifications](Runbook.ManualDataModifications.md) - more concrete examples on how manually update the data.