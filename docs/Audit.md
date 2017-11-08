## Auditing
This document describes auditing in RDW and provides sample queries for analysing audit data.

### Intended Audience
The intended audience should be familiar with database technology and querying a database with SQL.
- **System and Database Administrators**: This document provides administrators information on what is audited in the warehouse and where it is stored.
- **Developers and Analysts**: Developers and analysts with knowledge of SQL and the appropriate permissions can use this document as a guide to querying exam and student modifications.

### Terminology
- **Test Result**: When a student takes a test the results are transmitted to the data warehouse.  This submission is a test result.  It is for one instance of a student taking a given test.
- **TRT**: Is an acronym for an instance of a test result in the Smarter Balanced [Test Results Transmission Format](http://www.smarterapp.org/specs/TestResultsTransmissionFormat.html) where the content adheres to the [Test Results Data Model](http://www.smarterapp.org/news/2015/08/26/DataModelAndSamples.html)
- **Ingest**: Ingest is the process of receiving a submission of data and loading it into the data warehouse.
- **Exam**: Each test result submitted or migrated from legacy data is stored as an exam in the data warehouse.
- **warehouse schema**: The warehouse schema is the source of truth for reporting in the data warehouse and is used to populate user reporting and analytical reporting schemas. All schemas are defined in the [SmarterApp/RDW_Schema](https://github.com/SmarterApp/RDW_Schema) repository.  The warehouse schema is in the [SmarterApp/RDW_Schema/warehouse](https://github.com/SmarterApp/RDW_Schema/tree/develop/warehouse) folder.
- **State Changes**: Auditing tracks entity changes.
  - **Create**: A new entity such as an exam is added to the warehouse.  This is not audited, however, there is an import record that records attributes of the submission.
  - **Update**: A request to change a previously created entity.  This is audited as an update.
  - **Delete**: An entity is removed from the warehouse.  Does not occur for entities being audited such as exam.  Does occur for entity attributes stored in supporting child tables and is audited as a delete.
  - **Soft Delete**: A request to delete an entity from the warehouse that is being audited is updated with it's deleted flag set to true.  This is audited as an update.
- **Modification**: update, delete and soft delete state changes
- **Import**:  All inflows of data to the warehouse create an import record that stores attributes of the inflow including a timestamp and the authorized user.
  

### What is audited?
The warehouse audits entity state changes for exams and student information.

Warehouse Exam Tables:

| Table                        | Description                                 | Entity Type | Ingest State Changes        |
|------------------------------|---------------------------------------------|-------------|-----------------------------|
| exam                         | One record per test result                  | Parent      | Create, Update, Soft Delete |
| exam_available_accommodation | One record per exam available accommodation | Child       | Create, Delete              |
| exam_claim_score             | One record per exam claim                   | Child       | Create, Update              |
| exam_item                    | One record per exam item                    | Child       | Create, Update, Delete      |


### Where is audit data stored?
Each exam table has an 'audit_...' table that records the state change for each row.  The audit tables contain the state of the row before the change.
In addition to the columns from the table being audited, each audit_ table has the following columns:

- **id**: Unique ID
- **action**: delete or update
- **audited**: timestamp when the audit record was created
- **database_user**: The 'username'@'hostname' of the database user connected to the database.

The Ingest Event column is the normal application flow for loading data into the warehouse where events are recorded.  Create is not recorded, instead, queries combining the the audit_ table and the table being audited can represent a complete audit trail.

MySQL triggers are used to capture audit_ records.  Each table being audited has update and delete triggers providing a record of changes made by normal application flow and manual modifications if they occur.  

| Audit Table                        | Audits                       | Ingest Event                   |
|------------------------------------|------------------------------|--------------------------------|
| audit_exam                         | exam                         | Update, Soft Delete(as update) |
| audit_exam_available_accommodation | exam_available_accommodation | Delete                         |
| audit_exam_claim_score             | exam_claim_score             | Update                         |
| audit_exam_item                    | exam_item                    | Update, Delete                 |

### How can audit data be queried?
Sample queries are provided for analysing audit data combining the warehouse import table, the table being audited, the audit_ table and joining other relations in the warehouse for lookup values.

##### Exam

**Finding modifications to a students exams**
The following query outputs one row for each modified exam for one student.  It includes the count of modifications and the date of the last change.

The where clause can be changed to include multiple students.

```mysql
SELECT
  s.ssid,
  CONCAT(s.last_or_surname, ', ', first_name) student,
  e.id exam_id,
  e.oppId,
  e.opportunity,
  asmt.name assessment_name,
  COUNT(1) exam_update_count,
  MAX(ae.audited) last_update
FROM exam e
LEFT JOIN audit_exam ae ON e.id = ae.exam_id
JOIN student s ON e.student_id = s.id
JOIN asmt ON e.asmt_id = asmt.id
WHERE ae.exam_id IS NOT NULL
AND e.student_id IN ( SELECT id FROM student WHERE ssid = 'SSID001')
GROUP BY e.id
```

```text
+---------+-----------------+---------+--------------+-------------+-----------------------------------+-------------------+----------------------------+
| ssid    | student         | exam_id | oppId        | opportunity | assessment_name                   | exam_update_count | last_update                |
+---------+-----------------+---------+--------------+-------------+-----------------------------------+-------------------+----------------------------+
| SSID001 | Durrant, Gladys |       1 | 100000000010 |           5 | SBAC-IAB-FIXED-G11M-AlgQuad       |                 1 | 2017-10-11 09:42:39.986370 |
| SSID001 | Durrant, Gladys |       3 | 100000000030 |           1 | SBAC-ICA-FIXED-G11E-COMBINED-2017 |                 3 | 2017-10-11 09:45:10.235463 |
+---------+-----------------+---------+--------------+-------------+-----------------------------------+-------------------+----------------------------+
2 rows in set (0.00 sec)
```
**Finding modifications to exams in a date range**
The following query outputs one row for each modification to an exam within a date range.

```mysql
SELECT
  s.ssid,
  CONCAT(s.last_or_surname, ', ', first_name) student,
  e.id exam_id,
  e.oppId,
  e.opportunity,
  asmt.name assessment_name,
  COUNT(1) exam_update_count,
  MAX(ae.audited) last_update
FROM exam e
LEFT JOIN audit_exam ae ON e.id = ae.exam_id
JOIN student s ON e.student_id = s.id
JOIN asmt ON e.asmt_id = asmt.id
WHERE ae.exam_id IS NOT NULL
AND ae.audited BETWEEN '2017-10-11 09:45' AND '2017-10-12 09:45'
GROUP BY e.id
```

```text
+---------+-----------------+---------+--------------+-------------+-----------------------------------+-------------------+----------------------------+
| ssid    | student         | exam_id | oppId        | opportunity | assessment_name                   | exam_update_count | last_update                |
+---------+-----------------+---------+--------------+-------------+-----------------------------------+-------------------+----------------------------+
| SSID001 | Durrant, Gladys |       3 | 100000000030 |           1 | SBAC-ICA-FIXED-G11E-COMBINED-2017 |                 1 | 2017-10-11 09:45:10.235463 |
| SSID003 | Anderson, Mark  |       5 | 100000000050 |           2 | SBAC-ICA-FIXED-G11E-COMBINED-2017 |                 1 | 2017-10-11 22:45:26.773878 |
+---------+-----------------+---------+--------------+-------------+-----------------------------------+-------------------+----------------------------+
2 rows in set (0.00 sec)
```

**Listing all exams for one student with update count**
The following query outputs one row for each exam for one student with a count of modifications and the date of the most recent update.

```mysql
SELECT
  s.ssid,
  CONCAT(s.last_or_surname, ', ', first_name) student,
  e.id exam_id,
  e.oppId,
  e.opportunity,
  asmt.name assessment_name,
  SUM(CASE WHEN ae.exam_id IS NOT NULL THEN 1 ELSE 0 END) exam_update_count,
  MAX(ae.audited) last_update
FROM exam e
LEFT JOIN audit_exam ae ON e.id = ae.exam_id
JOIN student s ON e.student_id = s.id
JOIN asmt ON e.asmt_id = asmt.id
WHERE e.student_id IN ( SELECT id FROM student WHERE ssid = 'SSID001')
GROUP BY e.id
```

```text
+---------+-----------------+---------+--------------+-------------+-------------------------------------+-------------------+----------------------------+
| ssid    | student         | exam_id | oppId        | opportunity | assessment_name                     | exam_update_count | last_update                |
+---------+-----------------+---------+--------------+-------------+-------------------------------------+-------------------+----------------------------+
| SSID001 | Durrant, Gladys |       1 | 100000000010 |           5 | SBAC-IAB-FIXED-G11M-AlgQuad         |                 1 | 2017-10-11 09:42:39.986370 |
| SSID001 | Durrant, Gladys |       2 | 100000000020 |           5 | SBAC-IAB-FIXED-G11E-ReadInfo-ELA-11 |                 0 | NULL                       |
| SSID001 | Durrant, Gladys |       3 | 100000000030 |           1 | SBAC-ICA-FIXED-G11E-COMBINED-2017   |                 3 | 2017-10-11 09:45:10.235463 |
+---------+-----------------+---------+--------------+-------------+-------------------------------------+-------------------+----------------------------+
3 rows in set (0.00 sec)
```

**Exam audit trail**
The following query lists the details of exam modifications in addition to the current state.

The query can be modified to display different or all columns.  The where clause can be modified, such as, to include multiple exams or all exams for a student.

```mysql
SELECT
  s.ssid ssid,
  concat(s.last_or_surname, ', ', first_name) student,
  e.exam_id,
  e.oppId,
  e.opportunity,
  asmt.name assessment_name,
  e.exam_import_id import,
  i.creator import_creator,
  e.updated,
  e.action,
  c.code completeness,
  ac.code admin,
  sc.name school,
  g.code grade,
  e.scale_score
FROM (
       SELECT
         id exam_id,
         oppId,
         opportunity,
         update_import_id AS exam_import_id,
         'current' AS action,
         type_id,
         school_year,
         asmt_id,
         asmt_version,
         completeness_id,
         administration_condition_id,
         session_id,
         scale_score,
         scale_score_std_err,
         performance_level,
         completed_at,
         deleted,
         updated,
         grade_id,
         student_id,
         school_id,
         iep,
         lep,
         section504,
         economic_disadvantage,
         migrant_status,
         eng_prof_lvl,
         t3_program_type,
         language_code,
         prim_disability_type
       FROM exam e
       WHERE import_id != update_import_id
             AND id IN (3)
       UNION ALL
       SELECT
         exam_id,
         oppId,
         opportunity,
         update_import_id AS exam_import_id,
         CASE WHEN import_id = update_import_id THEN 'original' ELSE action END action,
         type_id,
         school_year,
         asmt_id,
         asmt_version,
         completeness_id,
         administration_condition_id,
         session_id,
         scale_score,
         scale_score_std_err,
         performance_level,
         completed_at,
         deleted,
         updated,
         grade_id,
         student_id,
         school_id,
         iep,
         lep,
         section504,
         economic_disadvantage,
         migrant_status,
         eng_prof_lvl,
         t3_program_type,
         language_code,
         prim_disability_type
       FROM audit_exam e
       WHERE e.exam_id IN (3)
     ) e
LEFT JOIN administration_condition ac ON e.administration_condition_id = ac.id
LEFT JOIN completeness c ON e.completeness_id = c.id
LEFT JOIN student s ON e.student_id = s.id
LEFT JOIN asmt asmt ON e.asmt_id = asmt.id
LEFT JOIN school sc ON e.school_id = sc.id
LEFT JOIN grade g ON e.grade_id = g.id
JOIN import i ON e.exam_import_id = i.id
ORDER BY e.exam_id, e.updated DESC;
```

```text
+---------+-----------------+---------+--------------+-------------+-----------------------------------+--------+--------------------+----------------------------+----------+--------------+-------+--------------------+-------+-------------+
| ssid    | student         | exam_id | oppId        | opportunity | assessment_name                   | import | import_creator     | updated                    | action   | completeness | admin | school             | grade | scale_score |
+---------+-----------------+---------+--------------+-------------+-----------------------------------+--------+--------------------+----------------------------+----------+--------------+-------+--------------------+-------+-------------+
| SSID001 | Durrant, Gladys |       3 | 100000000030 |           1 | SBAC-ICA-FIXED-G11E-COMBINED-2017 |     18 | dwtest@example.com | 2017-10-11 09:45:10.235463 | current  | Complete     | NS    | Llama Sabrewing HS | 11    |        2621 |
| SSID001 | Durrant, Gladys |       3 | 100000000030 |           1 | SBAC-ICA-FIXED-G11E-COMBINED-2017 |     17 | dwtest@example.com | 2017-10-11 09:43:55.197663 | update   | Complete     | NS    | Llama Sabrewing HS | 11    |        2621 |
| SSID001 | Durrant, Gladys |       3 | 100000000030 |           1 | SBAC-ICA-FIXED-G11E-COMBINED-2017 |     16 | dwtest@example.com | 2017-10-11 09:42:40.190306 | update   | Complete     | SD    | Llama Sabrewing HS | 11    |        2601 |
| SSID001 | Durrant, Gladys |       3 | 100000000030 |           1 | SBAC-ICA-FIXED-G11E-COMBINED-2017 |     12 | dwtest@example.com | 2017-10-11 09:41:24.976929 | original | Complete     | SD    | Llama Sabrewing HS | 11    |        2601 |
+---------+-----------------+---------+--------------+-------------+-----------------------------------+--------+--------------------+----------------------------+----------+--------------+-------+--------------------+-------+-------------+
4 rows in set (0.00 sec)
```

**Accommodation audit trail for exams**
Each child table audit trail can be queried in a similar manner.  The following example is for the exam_available_accommodation child table.

The query can be modified to display different or all columns.  The where clause can be modified, such as, to include multiple exams or all exams for a student.


```mysql
SELECT
  acc_audit.*,
  e.oppId,
  s.ssid,
  concat(s.last_or_surname, ', ', s.first_name) student
FROM (
       SELECT
         exams.exam_id,
         i.created action_date,
         exams.action,
         i.id import_id,
         i.creator,
         ' ' accommodation
       FROM (
              SELECT
                e.id exam_id,
                e.update_import_id,
                'current' AS action
              FROM exam e
              WHERE e.id IN (3)
              UNION ALL
              SELECT
                e.exam_id,
                e.update_import_id,
                e.action
              FROM audit_exam e
              WHERE e.exam_id IN (3)
            ) exams
       JOIN import i ON exams.update_import_id = i.id
       UNION ALL
       SELECT
         acc_events.exam_id,
         acc_events.updated action_date,
         acc_events.action,
         ' ' import_id,
         ' ' creator,
         acc.code accommodation
       FROM (
              SELECT
                eaa.exam_id,
                eaa.accommodation_id,
                'create' action,
                eaa.created AS updated
              FROM exam_available_accommodation eaa
              WHERE exam_id IN (3)
              UNION ALL
              SELECT
                aeaa.exam_id,
                aeaa.accommodation_id,
                aeaa.action,
                aeaa.audited AS updated
              FROM audit_exam_available_accommodation aeaa
              WHERE aeaa.exam_id IN (3)
            ) acc_events
       JOIN accommodation acc ON acc_events.accommodation_id = acc.id
     ) acc_audit
JOIN exam e ON acc_audit.exam_id = e.id
JOIN student s ON e.student_id = s.id
ORDER BY acc_audit.exam_id, acc_audit.action_date DESC;
```

```text
+---------+----------------------------+---------+-----------+--------------------+----------------+--------------+---------+-----------------+
| exam_id | action_date                | action  | import_id | creator            | accommodation  | oppId        | ssid    | student         |
+---------+----------------------------+---------+-----------+--------------------+----------------+--------------+---------+-----------------+
|       3 | 2017-10-11 09:45:10.234547 | create  |           |                    | TDS_PM0        | 100000000030 | SSID001 | Durrant, Gladys |
|       3 | 2017-10-11 09:45:10.088999 | current | 18        | dwtest@example.com |                | 100000000030 | SSID001 | Durrant, Gladys |
|       3 | 2017-10-11 09:43:55.196671 | delete  |           |                    | TDS_BT0        | 100000000030 | SSID001 | Durrant, Gladys |
|       3 | 2017-10-11 09:43:54.980010 | update  | 17        | dwtest@example.com |                | 100000000030 | SSID001 | Durrant, Gladys |
|       3 | 2017-10-11 09:42:40.189155 | create  |           |                    | NEA_Calc       | 100000000030 | SSID001 | Durrant, Gladys |
|       3 | 2017-10-11 09:42:40.189155 | create  |           |                    | TDS_ClosedCap0 | 100000000030 | SSID001 | Durrant, Gladys |
|       3 | 2017-10-11 09:42:39.890355 | update  | 16        | dwtest@example.com |                | 100000000030 | SSID001 | Durrant, Gladys |
|       3 | 2017-10-11 09:41:24.993423 | create  |           |                    | TDS_ASL0       | 100000000030 | SSID001 | Durrant, Gladys |
|       3 | 2017-10-11 09:41:24.621145 | update  | 12        | dwtest@example.com |                | 100000000030 | SSID001 | Durrant, Gladys |
+---------+----------------------------+---------+-----------+--------------------+----------------+--------------+---------+-----------------+
9 rows in set (0.00 sec)
```

