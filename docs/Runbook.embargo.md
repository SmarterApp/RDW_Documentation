## Runbook - Embargo

**Intended Audience**: this document contains information on how embargo is handled in the [Reporting Data Warehouse](../README.md) (RDW). Operations, system administrators, support and developers may find this useful to understand this mechanism. 

Embargo is a feature where summative test results are held back until all the data is available and validated. An embargo affects the viewing of individual test results separately from aggregate report data. Lifting the embargo releases test results.

When individual test results are embargoed, those results will not be visible in the reporting UI to normal users. Similarly, when aggregate test results are embargoed, those reports will not be visible in the aggregate reporting UI to normal users. Embargo administrators will be able to see the results (and the UI will have a notice to that effect) regardless of embargo settings. 

Embargo is set at the state and district levels. Districts may release test results while the state is still embargoed. However, once a state releases test results (i.e. lifts the embargo), all results for all districts are released, regardless of the district embargo settings.

Embargo settings are managed in the Admin UI. The functionality will be available if a user has embargo write permissions for the state or districts.

#### Import and Migrate
As discussed in the [Import and Migrate](Runbook.migrate.md) the embargo setting is migrated as part of the general ingest process. Although not recommended, it is possible to manually modify embargo settings. For example, to lift the statewide embargo on individual test results for the school year 2018, then trigger the migration:
```sql
UPDATE state_embargo SET individual=0 WHERE school_year=2018;
INSERT INTO import (status, content, contentType, digest) VALUES (1, 6, 'lift embargo', 'lift embargo 2017-12-12');
```
