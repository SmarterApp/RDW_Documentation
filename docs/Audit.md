## Auditing
This document describes auditing in RDW and provides sample queries for analyzing audit data.

### Table of Contents

* [Intended audience](#intended-audience)
* [Terminology](#terminology)
* [What is audited?](#what-is-audited)
* [Where is audit data stored?](#where-is-audit-data-stored)
* [Adding audit table indexes](#adding-audit-table-indexes)
* [Enable and disable auditing](#enable-and-disable-auditing)
* [How can audit data be queried?](#how-can-audit-data-be-queried)
    * [Query Exam](#query-exam)
    * [Query Student](#query-student)
    * [Query Student Groups](#query-student-groups)
* [How can the audit trail be cleared?](#how-can-the-audit-trail-be-cleared)
    * [Clear a specific student record](#clear-a-specific-student-record)
    * [Clear all exams for a specific student](#clear-all-exams-for-a-specific-student)
    * [Clear a specific exam record](#clear-a-specific-exam-record)
    * [Clear all exams for a school](#clear-all-exams-for-a-school)
    * [Clear all exams for a specific assessment](#clear-all-exams-for-a-specific-assessment)
    * [Clear a specific student group](#clear-a-specific-student-group)
    * [Clear all student groups for a specific school](#clear-all-student-groups-for-a-specific-school)

### Intended audience
The intended audience should be familiar with database technology and querying a database with SQL.
- **System and Database Administrators**: This document provides administrators information on what is audited in the warehouse and where it is stored.
- **Developers and Analysts**: Developers and analysts with knowledge of SQL and the appropriate permissions can use this document as a guide to querying exam and student modifications.

### Terminology
- **Test Result**: When a student takes a test the results are transmitted to the data warehouse.
- **TRT**: Is an acronym for an instance of a test result in the Smarter Balanced [Test Results Transmission Format](http://www.smarterapp.org/specs/TestResultsTransmissionFormat.html) where the content adheres to the [Test Results Data Model](http://www.smarterapp.org/news/2015/08/26/DataModelAndSamples.html).
- **Ingest**: Ingest is the process of receiving a submission of data and loading it into the data warehouse.
- **Exam**: Each test result submitted or migrated from legacy data is stored as an exam in the data warehouse.
- **Warehouse Schema**: The warehouse schema is the source of truth for reporting in the data warehouse and is used to populate user reporting and analytical reporting schemas. All schemas are defined in the [SmarterApp/RDW_Schema](https://github.com/SmarterApp/RDW_Schema) repository.  The warehouse schema is in the [SmarterApp/RDW_Schema/warehouse](https://github.com/SmarterApp/RDW_Schema/tree/develop/warehouse) folder.
- **State Changes**: Auditing tracks entity changes.
  - **Create**: A new entity is added to the warehouse.  This is not audited, however, there is an import record that records attributes of the submission.
  - **Update**: A request to change a previously created entity.  This is audited as an update.
  - **Delete**: An entity is removed from the warehouse.  This is audited as a delete.
  - **Soft Delete**: An update that sets a deleted flag to true.  This is audited as an update.
- **Modification**: Update, delete and soft delete state changes.
- **Import**:  All inflows of data to the warehouse create an import record that stores attributes of the inflow including a timestamp and the authorized user.


### What is audited?
1. All actions to release or embargo results are audited. 

Warehouse tables audited:

| Table                        | Description                                                        | Audited actions        |
|------------------------------|--------------------------------------------------------------------|------------------------|
| district_embargo             | Individual and aggregate embargo flags per district and school year| Create, Update, Delete |
| state_embargo                | Statewide individual and aggregate embargo flags per school year   | Create, Update, Delete |


2. Changes for the existing exams and student information.

Warehouse tables audited:

| Table                        | Description                                 | Entity Type | Audited actions               |
|------------------------------|---------------------------------------------|-------------|-------------------------------|
| exam                         | One record per test result                  | Parent      | Update, Soft Delete as Update |
| exam_available_accommodation | One record per exam available accommodation | Child       | Delete                        |
| exam_claim_score             | One record per exam claim                   | Child       | Update                        |
| exam_item                    | One record per exam item                    | Child       | Update, Delete                |
| student                      | One record per student                      | Parent      | Update, Soft Delete as Update |
| student_ethnicity            | One record per student ethnicity            | Child       | Delete                        |
| student_group                | One record per student group                | Parent      | Update, Soft Delete as Update |
| student_group_membership     | One record per student membership in group  | Child       | Delete                        |
| user_student_group           | One record per user with access to group    | Child       | Delete                        |

'Create' event is not audited for the exam and student data.

### Where is audit data stored?
Each audited table has an `audit_...` table that records the state change for each row.  The audit tables contain the state of the row before the change.  In addition to the columns from the table being audited, each audit_ table has the following columns:

- **id**: Unique ID
- **action**: A value of insert, delete or update 
- **audited**: Timestamp when the audit record was created
- **database_user**: The 'username'@'hostname' of the database user connected to the database

MySQL triggers are used to create `audit_` records. Each table being audited has triggers providing a record of changes made by normal application flow and manual modifications if they occur.  

### Adding audit table indexes
There are no indexes on audit tables.  If auditing is queried frequently or the tables become large, indexes could be added to improve query performance.  The trade off is indexes on audit tables could have a negative impact on ingest performance.

### Enable and disable auditing
Embargo related auditing may not be disabled.  
Exams and students audting is controlled by a `setting` table.  The setting record with a `name` of `AUDIT_TRIGGER_ENABLE` controls if audit records will be created.  Only when the `value` is `TRUE` will audit records be created.

To view the current audit setting run the following query.

```mysql
SELECT s.name, s.value FROM setting s WHERE s.name = 'AUDIT_TRIGGER_ENABLE';
```

To enable auditing run the following statement.

```mysql
UPDATE setting s SET s.value = 'TRUE' WHERE s.name = 'AUDIT_TRIGGER_ENABLE';
```

To disable auditing run the following statement.

```mysql
UPDATE setting s SET s.value = 'FALSE' WHERE s.name = 'AUDIT_TRIGGER_ENABLE';
```

### How can audit data be queried?
Sample queries are provided for analyzing audit data combining the warehouse import table, the table being audited, the audit_ table, and joining other relations in the warehouse for lookup values.

#### Query state embargo
```mysql
SELECT * FROM audit_state_embargo
```

#### Query district embargo
```mysql
SELECT * FROM audit_district_embargo
```

#### Query exam

**Finding modifications to a student's exams.**
The following query outputs one row for each modified exam for one student.  It includes the count of modifications and the date of the last change.

The `WHERE` clause can be changed to include multiple students.

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
**Finding modifications to exams in a date range.**
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

**Listing all exams for one student, with update count.** 
The following query outputs one row for each exam for one student, with a count of modifications and the date of the most recent update.  If the exam has not been modified `exam_update_count` is zero.

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

**Exam audit trail.** 
The following query lists the details of exam modifications in addition to the current state.

The query can be modified to display different or all columns.  The `WHERE` clause can be modified, for example, to include multiple exams or all exams for a student.

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

**Accommodation audit trail for exams.** 
Each child table audit trail can be queried in a similar manner.  The following example is for the exam_available_accommodation child table.

The query can be modified to display different or all columns.  The `WHERE` clause can be modified, for example, to include multiple exams or all exams for a student.


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

#### Query student

**Finding modified students.** 
The following query outputs one row for each modified student.  It includes the count of modifications and the date of the last change.

A `WHERE` clause can be added to filter students.

```mysql
SELECT
  s.ssid,
  COALESCE(s.first_name, '') first_name,
  COALESCE(s.middle_name, '') middle_name,
  COALESCE(s.last_or_surname, '') last_or_surname,
  COUNT(1) student_update_count,
  MAX(ast.audited) last_update
FROM student s
JOIN audit_student ast ON s.id = ast.student_id
GROUP BY s.id;
```

```text
+---------+------------+-------------+-----------------+----------------------+----------------------------+
| ssid    | first_name | middle_name | last_or_surname | student_update_count | last_update                |
+---------+------------+-------------+-----------------+----------------------+----------------------------+
| SSID001 | Gladys     | Ruth        | Williams        |                    6 | 2017-11-10 09:26:31.858088 |
| SSID002 | Joe        |             | Smith           |                    1 | 2017-11-10 09:23:01.576030 |
| SSID003 | Mark       |             | Anderson        |                    1 | 2017-11-10 09:25:21.846849 |
+---------+------------+-------------+-----------------+----------------------+----------------------------+
3 rows in set (0.00 sec)
```

**Finding modifications to students by school.** 
The following query outputs one row for each student in a specific current institution.  Current institution is inferred from the most recent exam completed by a student.
The output includes the count of modifications and the date of the last change.  Students with no modifications have a `student_update_count` of `0`.

Running this query without a `WHERE` clause to limit the students will result in a full scan of the `students` table resulting in a long running query on databases with a large number of students.

```mysql
SELECT
  s.ssid,
  COALESCE(s.first_name, '') first_name,
  COALESCE(s.middle_name, '') middle_name,
  COALESCE(s.last_or_surname, '') last_or_surname,
  SUM(CASE WHEN ast.student_id IS NOT NULL THEN 1 ELSE 0 END) student_update_count,
  MAX(ast.audited) last_update
FROM student s
LEFT JOIN audit_student ast ON s.id = ast.student_id
WHERE s.inferred_school_id IN ( SELECT id from school WHERE natural_id = 'TS000001')
GROUP BY s.id;
```

```text
+---------+------------+-------------+-----------------+----------------------+----------------------------+
| ssid    | first_name | middle_name | last_or_surname | student_update_count | last_update                |
+---------+------------+-------------+-----------------+----------------------+----------------------------+
| SSID002 | Joe        |             | Smith           |                    1 | 2017-12-07 01:41:32.825826 |
| SSID004 | Linda      |             | Smart           |                    0 | NULL                       |
+---------+------------+-------------+-----------------+----------------------+----------------------------+
2 rows in set (0.00 sec)
```

**Finding modifications to students in a date range.** 
The following query outputs one row for each modification to a student within a date range.

```mysql
SELECT
  s.ssid,
  COALESCE(s.first_name, '') first_name,
  COALESCE(s.middle_name, '') middle_name,
  COALESCE(s.last_or_surname, '') last_or_surname,
  COUNT(1) student_update_count,
  MAX(ast.audited) last_update
FROM student s
JOIN audit_student ast ON s.id = ast.student_id
WHERE ast.audited BETWEEN '2017-11-10 09:24:00' AND '2017-11-10 09:26:00'
GROUP BY s.id;
```

```text
+---------+------------+-------------+-----------------+----------------------+----------------------------+
| ssid    | first_name | middle_name | last_or_surname | student_update_count | last_update                |
+---------+------------+-------------+-----------------+----------------------+----------------------------+
| SSID001 | Gladys     | Ruth        | Williams        |                    2 | 2017-11-10 09:25:21.681122 |
| SSID003 | Mark       |             | Anderson        |                    1 | 2017-11-10 09:25:21.846849 |
+---------+------------+-------------+-----------------+----------------------+----------------------------+
2 rows in set (0.00 sec)
```

**Student audit trail.** 
The following query lists the details of student modifications in addition to the current state.

The query can be modified to display different or all columns.  The `WHERE` clause can be modified, for example, to include multiple students or all students in a school.

```mysql
SELECT
  s.student_id,
  s.ssid,
  s.first_name,
  s.middle_name,
  s.last_or_surname,
  s.student_import_id import,
  i.creator import_creator,
  s.updated,
  s.action,
  s.birthday,
  sch.name current_school
FROM (
       SELECT
         id student_id,
         ssid,
         COALESCE(s.first_name, '') first_name,
         COALESCE(s.middle_name, '') middle_name,
         COALESCE(s.last_or_surname, '') last_or_surname,
         update_import_id AS student_import_id,
         'current' AS action,
         updated,
         birthday,
         inferred_school_id
       FROM student s
       WHERE import_id != update_import_id
             AND s.id IN ( SELECT id FROM student WHERE ssid IN ('SSID001') )
       UNION ALL
       SELECT
         student_id,
         ssid,
         COALESCE(s.first_name, '') first_name,
         COALESCE(s.middle_name, '') middle_name,
         COALESCE(s.last_or_surname, '') last_or_surname,
         update_import_id AS student_import_id,
         CASE WHEN import_id = update_import_id THEN 'original' ELSE action END action,
         updated,
         birthday,
         inferred_school_id
       FROM audit_student s
       WHERE s.student_id IN ( SELECT id FROM student WHERE s.ssid IN ('SSID001') )
     ) s
JOIN import i ON s.student_import_id = i.id
LEFT JOIN school sch ON s.inferred_school_id = sch.id
ORDER BY s.student_id, s.updated DESC;
```

```text
+------------+---------+------------+-------------+-----------------+--------+--------------------+----------------------------+----------+------------+----------------------------+
| student_id | ssid    | first_name | middle_name | last_or_surname | import | import_creator     | updated                    | action   | birthday   | current_school             |
+------------+---------+------------+-------------+-----------------+--------+--------------------+----------------------------+----------+------------+----------------------------+
|          1 | SSID001 | Gladys     | Ruth        | Williams        |     19 | dwtest@example.com | 2017-11-10 09:26:31.858088 | current  | 2001-06-23 | Kingbird Koala Junior High |
|          1 | SSID001 | Gladys     | Ruth        | Williams        |     16 | dwtest@example.com | 2017-11-10 09:25:21.681122 | update   | 2001-09-23 | Kingbird Koala Junior High |
|          1 | SSID001 | Gladys     | Ruth        | Durrant         |     15 | dwtest@example.com | 2017-11-10 09:24:11.604717 | update   | 2001-06-23 | Llama Sabrewing HS         |
|          1 | SSID001 | Gladys     | Ruth        | Durrant         |     13 | dwtest@example.com | 2017-11-10 09:23:01.498262 | update   | 2001-06-23 | Llama Sabrewing HS         |
|          1 | SSID001 | Gladys     |             | Durrant         |     12 | dwtest@example.com | 2017-11-10 09:21:51.427089 | update   | 2001-06-23 | Llama Sabrewing HS         |
|          1 | SSID001 | Gladys     |             | Durrant         |     10 | dwtest@example.com | 2017-11-10 09:20:41.278709 | update   | 2001-06-23 | Llama Sabrewing HS         |
|          1 | SSID001 | Gladys     |             | Durrant         |      6 | dwtest@example.com | 2017-11-10 09:19:31.054923 | original | 2001-06-23 | Llama Sabrewing HS         |
+------------+---------+------------+-------------+-----------------+--------+--------------------+----------------------------+----------+------------+----------------------------+
7 rows in set (0.00 sec)
```
**Ethnicity audit trail for students.** 
Each child table audit trail can be queried in a similar manner.  The following example is for the `student_ethnicity` child table.  Student currently has one child table.

The query can be modified to display different or all columns.  The `WHERE` clause can be modified, for example, to include multiple students or all students in a school.


```mysql
SELECT
  eth_audit.*,
  s.ssid,
  CONCAT(s.last_or_surname, ', ', first_name) student
FROM (
       SELECT
         students.student_id,
         i.created action_date,
         students.action,
         i.id import_id,
         i.creator,
         ' ' ethnicity
       FROM (
              SELECT
                s.id student_id,
                s.update_import_id,
                'current' AS action
              FROM student s
              WHERE s.id IN ( SELECT id FROM student WHERE ssid IN ('SSID001') )
              UNION ALL
              SELECT
                ast.student_id,
                ast.update_import_id,
                ast.action
              FROM audit_student ast
              WHERE ast.student_id IN ( SELECT id FROM student WHERE ssid IN ('SSID001') )
            ) students
       JOIN import i ON students.update_import_id = i.id
       UNION ALL
       SELECT
         eth_events.student_id,
         eth_events.updated action_date,
         eth_events.action,
         ' ' import_id,
         ' ' creator,
         eth.code ethnicity
       FROM (
              SELECT
                se.student_id,
                se.ethnicity_id,
                'create' action,
                se.created AS updated
              FROM student_ethnicity se
              WHERE se.student_id IN ( SELECT id FROM student WHERE ssid IN ('SSID001') )
              UNION ALL
              SELECT
                ase.student_id,
                ase.ethnicity_id,
                ase.action,
                ase.audited AS updated
              FROM audit_student_ethnicity ase
              WHERE ase.student_id IN ( SELECT id FROM student WHERE ssid IN ('SSID001') )
            ) eth_events
       JOIN ethnicity eth ON eth_events.ethnicity_id = eth.id
     ) eth_audit
JOIN student s ON eth_audit.student_id = s.id
ORDER BY eth_audit.student_id, eth_audit.action_date DESC;
```

```text
+------------+----------------------------+---------+-----------+--------------------+---------------------------+---------+------------------+
| student_id | action_date                | action  | import_id | creator            | ethnicity                 | ssid    | student          |
+------------+----------------------------+---------+-----------+--------------------+---------------------------+---------+------------------+
|          1 | 2017-11-10 09:26:31.820510 | current | 19        | dwtest@example.com |                           | SSID001 | Williams, Gladys |
|          1 | 2017-11-10 09:25:21.629421 | update  | 16        | dwtest@example.com |                           | SSID001 | Williams, Gladys |
|          1 | 2017-11-10 09:24:11.606506 | create  |           |                    | HispanicOrLatinoEthnicity | SSID001 | Williams, Gladys |
|          1 | 2017-11-10 09:24:11.561458 | update  | 15        | dwtest@example.com |                           | SSID001 | Williams, Gladys |
|          1 | 2017-11-10 09:23:01.500648 | delete  |           |                    | BlackOrAfricanAmerican    | SSID001 | Williams, Gladys |
|          1 | 2017-11-10 09:23:01.443541 | update  | 13        | dwtest@example.com |                           | SSID001 | Williams, Gladys |
|          1 | 2017-11-10 09:21:51.448647 | create  |           |                    | White                     | SSID001 | Williams, Gladys |
|          1 | 2017-11-10 09:21:51.448647 | create  |           |                    | Filipino                  | SSID001 | Williams, Gladys |
|          1 | 2017-11-10 09:21:51.371817 | update  | 12        | dwtest@example.com |                           | SSID001 | Williams, Gladys |
|          1 | 2017-11-10 09:20:41.242122 | update  | 10        | dwtest@example.com |                           | SSID001 | Williams, Gladys |
|          1 | 2017-11-10 09:19:31.080708 | create  |           |                    | Asian                     | SSID001 | Williams, Gladys |
|          1 | 2017-11-10 09:19:31.000597 | update  | 6         | dwtest@example.com |                           | SSID001 | Williams, Gladys |
+------------+----------------------------+---------+-----------+--------------------+---------------------------+---------+------------------+
12 rows in set (0.00 sec)
```

#### Query student groups

**Finding modified student groups.** 
The following query outputs one row for each modified student_group.  It includes the count of modifications and the date of the last change.

A `WHERE` clause can be added to filter results.

```mysql
SELECT
  sg.id group_id,
  sg.name group_name,
  sch.name school,
  sg.school_year,
  COALESCE(sub.code, '') subject,
  count(1) update_count,
  MAX(asg.audited) last_update
FROM student_group sg
JOIN audit_student_group asg ON sg.id = asg.student_group_id
LEFT JOIN subject sub ON sg.subject_id = sub.id
JOIN school sch ON sg.school_id = sch.id
GROUP BY sg.id;
```

```text
+----------+-----------------+----------------------+-------------+---------+--------------+----------------------------+
| group_id | group_name      | school               | school_year | subject | update_count | last_update                |
+----------+-----------------+----------------------+-------------+---------+--------------+----------------------------+
|        1 | StudentGroup001 | Test School TS000001 |        2018 | Math    |            3 | 2017-11-29 05:39:10.050177 |
|        2 | StudentGroup002 | Test School TS000001 |        2018 | Math    |            2 | 2017-11-29 05:39:10.050177 |
|        4 | StudentGroup004 | Test School TS000002 |        2018 |         |            1 | 2017-11-29 05:38:00.006242 |
+----------+-----------------+----------------------+-------------+---------+--------------+----------------------------+
3 rows in set (0.00 sec)
```

**Finding modifications to student groups.** 
The following query outputs one row for each student_group.  It includes the count of modifications and the date of the last change.
Students with no modifications have an `update_count` of `0`.

A `WHERE` clause can be added to filter results.

```mysql
SELECT
  sg.id group_id,
  sg.name group_name,
  sch.name school,
  sg.school_year,
  COALESCE(sub.code, '') subject,
  SUM(CASE WHEN asg.student_group_id IS NOT NULL THEN 1 ELSE 0 END) update_count,
  MAX(asg.audited) last_update
FROM student_group sg
LEFT JOIN audit_student_group asg ON sg.id = asg.student_group_id
LEFT JOIN subject sub ON sg.subject_id = sub.id
JOIN school sch ON sg.school_id = sch.id
GROUP BY sg.id;
```

```text
+----------+-----------------+----------------------+-------------+---------+--------------+----------------------------+
| group_id | group_name      | school               | school_year | subject | update_count | last_update                |
+----------+-----------------+----------------------+-------------+---------+--------------+----------------------------+
|        1 | StudentGroup001 | Test School TS000001 |        2018 | Math    |            3 | 2017-11-29 05:39:10.050177 |
|        2 | StudentGroup002 | Test School TS000001 |        2018 | Math    |            2 | 2017-11-29 05:39:10.050177 |
|        3 | StudentGroup003 | Test School TS000002 |        2018 |         |            0 | NULL                       |
|        4 | StudentGroup004 | Test School TS000002 |        2018 |         |            1 | 2017-11-29 05:38:00.006242 |
|        6 | StudentGroup005 | Test School TS000005 |        2018 |         |            0 | NULL                       |
+----------+-----------------+----------------------+-------------+---------+--------------+----------------------------+
5 rows in set (0.00 sec)
```

**Finding modifications to student groups in a date range.** 
The following query outputs one row for each modification to a student_group within a date range.

```mysql
SELECT
  sg.id group_id,
  sg.name group_name,
  sch.name school,
  sg.school_year,
  COALESCE(sub.code, '') subject,
  count(1) update_count,
  MAX(asg.audited) last_update
FROM student_group sg
JOIN audit_student_group asg ON sg.id = asg.student_group_id
LEFT JOIN subject sub ON sg.subject_id = sub.id
JOIN school sch ON sg.school_id = sch.id
WHERE asg.audited BETWEEN '2017-11-29 05:37:00' AND '2017-11-29 05:38:00'
GROUP BY sg.id;
```

```text
+----------+-----------------+----------------------+-------------+---------+--------------+----------------------------+
| group_id | group_name      | school               | school_year | subject | update_count | last_update                |
+----------+-----------------+----------------------+-------------+---------+--------------+----------------------------+
|        1 | StudentGroup001 | Test School TS000001 |        2018 | Math    |            1 | 2017-11-29 05:37:59.986354 |
|        2 | StudentGroup002 | Test School TS000001 |        2018 | Math    |            1 | 2017-11-29 05:37:59.986354 |
+----------+-----------------+----------------------+-------------+---------+--------------+----------------------------+
2 rows in set (0.00 sec)
```

**Student group audit trail.** 
The following query lists the details of student group modifications in addition to the current state.

The query can be modified to display different or all columns.  The `WHERE` clause can be modified, for example, to include multiple student groups or all student groups in a district.

```mysql
SELECT
  sg.student_group_id group_id,
  sg.name group_name,
  sch.name school,
  sg.school_year,
  COALESCE(sub.code, '') subject,
  sg.deleted,
  sg.active,
  sg.group_import_id import,
  i.creator import_creator,
  sg.updated,
  sg.action
FROM (
       SELECT
         sg.id student_group_id,
         sg.name,
         sg.school_id,
         sg.school_year,
         sg.subject_id,
         sg.deleted,
         sg.active,
         update_import_id AS group_import_id,
         'current' AS action,
         sg.updated
       FROM student_group sg
       WHERE import_id != update_import_id
             AND sg.name IN ('StudentGroup001')
       UNION ALL
       SELECT
         asg.student_group_id,
         asg.name,
         asg.school_id,
         asg.school_year,
         asg.subject_id,
         asg.deleted,
         asg.active,
         update_import_id AS group_import_id,
         CASE WHEN import_id = update_import_id THEN 'original' ELSE action END action,
         updated
       FROM audit_student_group asg
       WHERE asg.name IN ('StudentGroup001')
     ) sg
JOIN import i ON sg.group_import_id = i.id
LEFT JOIN school sch ON sg.school_id = sch.id
LEFT JOIN subject sub ON sg.subject_id = sub.id
ORDER BY sg.student_group_id, sg.updated DESC;
```

```text
+----------+-----------------+----------------------+-------------+---------+---------+--------+--------+----------------+----------------------------+----------+
| group_id | group_name      | school               | school_year | subject | deleted | active | import | import_creator | updated                    | action   |
+----------+-----------------+----------------------+-------------+---------+---------+--------+--------+----------------+----------------------------+----------+
|        1 | StudentGroup001 | Test School TS000001 |        2018 | Math    |       0 |      1 |      6 | NULL           | 2017-11-29 05:39:10.050177 | current  |
|        1 | StudentGroup001 | Test School TS000001 |        2018 | Math    |       0 |      1 |      4 | NULL           | 2017-11-29 05:37:59.986354 | update   |
|        1 | StudentGroup001 | Test School TS000001 |        2018 | ELA     |       0 |      1 |      2 | NULL           | 2017-11-29 05:36:49.895244 | update   |
|        1 | StudentGroup001 | Test School TS000001 |        2018 | ELA     |       0 |      1 |      1 | NULL           | 2017-01-01 09:00:00.000000 | original |
+----------+-----------------+----------------------+-------------+---------+---------+--------+--------+----------------+----------------------------+----------+
4 rows in set (0.00 sec)
```
**Student membership audit trail.** 
Each child table audit trail can be queried in a similar manner.  For student_groups the child tables are `student_group_membership` and `user_student_group`.  The following example is for `student_group_membership`.  

The query can be modified to display different or all columns.  The `WHERE` clauses can be modified, for example, to include multiple student_groups or all student_groups in a district.


```mysql
SELECT
  membership_audit.*,
  COALESCE(s.ssid, '') ssid,
  COALESCE(CONCAT(s.last_or_surname, ', ', first_name), '') student,
  sg.name group_name
FROM (
       SELECT
         groups.student_group_id,
         i.created action_date,
         groups.action,
         i.id import_id,
         i.creator,
         ' ' student_id
       FROM (
              SELECT
                sg.id student_group_id,
                sg.update_import_id,
                'current' AS action
              FROM student_group sg
              WHERE sg.name IN ('StudentGroup001')
              UNION ALL
              SELECT
                asg.student_group_id,
                asg.update_import_id,
                asg.action
              FROM audit_student_group asg
              WHERE asg.name IN ('StudentGroup001')
            ) groups
       JOIN import i ON groups.update_import_id = i.id
       UNION ALL
       SELECT
         student_events.student_group_id,
         student_events.updated action_date,
         student_events.action,
         ' ' import_id,
         ' ' creator,
         student_events.student_id
       FROM (
              SELECT
                sgm.student_group_id,
                sgm.student_id,
                'create' action,
                sgm.created AS updated
              FROM student_group_membership sgm
              WHERE sgm.student_group_id IN ( SELECT id FROM student_group WHERE student_group.name IN ('StudentGroup001') )
              UNION ALL
              SELECT
                asgm.student_group_id,
                asgm.student_id,
                asgm.action,
                asgm.audited AS updated
              FROM audit_student_group_membership asgm
              WHERE asgm.student_group_id IN ( SELECT id FROM student_group WHERE student_group.name IN ('StudentGroup001') )
            ) student_events
     ) membership_audit
JOIN student_group sg ON membership_audit.student_group_id = sg.id
Left JOIN student s ON membership_audit.student_id = s.id
ORDER BY membership_audit.student_group_id, membership_audit.action_date DESC;
```

```text
+------------------+----------------------------+---------+-----------+---------+------------+---------+------------------+-----------------+
| student_group_id | action_date                | action  | import_id | creator | student_id | ssid    | student          | group_name      |
+------------------+----------------------------+---------+-----------+---------+------------+---------+------------------+-----------------+
|                1 | 2017-11-29 05:39:10.054242 | create  |           |         | 2          | SSID002 | Smith, Joe       | StudentGroup001 |
|                1 | 2017-11-29 05:39:10.053335 | delete  |           |         | 6          | SSID006 | Gray, Carol      | StudentGroup001 |
|                1 | 2017-11-29 05:39:10.045542 | current | 6         | NULL    |            |         |                  | StudentGroup001 |
|                1 | 2017-11-29 05:37:59.991001 | create  |           |         | 4          | SSID004 | Smart, Linda     | StudentGroup001 |
|                1 | 2017-11-29 05:37:59.991001 | create  |           |         | 5          | SSID005 | Bennett, Donna   | StudentGroup001 |
|                1 | 2017-11-29 05:37:59.990122 | delete  |           |         | 3          | SSID003 | Anderson, Mark   | StudentGroup001 |
|                1 | 2017-11-29 05:37:59.990122 | delete  |           |         | 2          | SSID002 | Smith, Joe       | StudentGroup001 |
|                1 | 2017-11-29 05:37:59.982200 | update  | 4         | NULL    |            |         |                  | StudentGroup001 |
|                1 | 2017-11-29 05:36:49.900632 | create  |           |         | 1          | SSID001 | Williams, Gladys | StudentGroup001 |
|                1 | 2017-11-29 05:36:49.887226 | update  | 2         | NULL    |            |         |                  | StudentGroup001 |
|                1 | 2017-11-29 05:36:19.109393 | update  | 1         | NULL    |            |         |                  | StudentGroup001 |
+------------------+----------------------------+---------+-----------+---------+------------+---------+------------------+-----------------+
11 rows in set (0.00 sec)
```

<a name="clear-audit"></a>
### How can the audit trail be trail?

#### Clear a specific student record

**Step 1: Table `audit_student_ethnicity`.  Query records to delete.** 
The following query outputs the `audit_student_ethnicity` records to be deleted.

```mysql
SELECT ase.id audit_id, ase.student_id, ase.ethnicity_id, ase.action, ase.audited FROM audit_student_ethnicity ase
WHERE student_id IN (SELECT s.id FROM student s WHERE s.ssid = 'SSID001');
```

```text
+----------+------------+--------------+--------+----------------------------+
| audit_id | student_id | ethnicity_id | action | audited                    |
+----------+------------+--------------+--------+----------------------------+
|        1 |          1 |            4 | delete | 2017-12-05 00:12:50.472978 |
+----------+------------+--------------+--------+----------------------------+
1 row in set (0.00 sec)
```

**Step 2: Table `audit_student_ethnicity`.  Delete records.** 
The following statement deletes records with the same `FROM` clause as the previous statement.

```mysql
DELETE ase FROM audit_student_ethnicity ase
WHERE student_id IN (SELECT s.id FROM student s WHERE s.ssid = 'SSID001');
```

```text
Query OK, 1 row affected (0.00 sec)
```

**Step 3: Table `audit_student_ethnicity`.  Validate records deleted.** 
Execute the same query used before the `DELETE` step to validate records no longer exist. 

```mysql
SELECT ase.id audit_id, ase.student_id, ase.ethnicity_id, ase.action, ase.audited FROM audit_student_ethnicity ase
WHERE student_id IN (SELECT s.id FROM student s WHERE s.ssid = 'SSID001');
```

```text
Empty set (0.00 sec)
```

**Step 4: Table `audit_student`.  Query records to delete.** 
The following query outputs the `audit_student` records to be deleted.

```mysql
SELECT ast.id audit_id, ast.student_id, ast.ssid, ast.action, ast.audited FROM audit_student ast
WHERE student_id IN (SELECT id FROM student WHERE ssid = 'SSID001');
```

```text
+----+--------+----------------------------+----------------+------------+---------+-----------------+
| id | action | audited                    | database_user  | student_id | ssid    | last_or_surname |
+----+--------+----------------------------+----------------+------------+---------+-----------------+
|  1 | update | 2017-12-01 23:01:02.854184 | root@localhost |          1 | SSID001 | Durrant         |
|  2 | update | 2017-12-01 23:01:02.928821 | root@localhost |          1 | SSID001 | Durrant         |
|  4 | update | 2017-12-01 23:01:03.080797 | root@localhost |          1 | SSID001 | Durrant         |
|  5 | update | 2017-12-01 23:01:03.151168 | root@localhost |          1 | SSID001 | Durrant         |
|  7 | update | 2017-12-01 23:01:03.364097 | root@localhost |          1 | SSID001 | Williams        |
+----+--------+----------------------------+----------------+------------+---------+-----------------+
5 rows in set (0.00 sec)
```

**Step 5: Table `audit_student`.  Delete records.** 
The following statement deletes records with the same `FROM` clause as the previous statement.

```mysql
DELETE ast FROM audit_student ast
WHERE student_id IN (SELECT id FROM student WHERE ssid = 'SSID001');
```

```text
Query OK, 5 rows affected (0.01 sec)
```

**Step 6: Table `audit_student`.  Validate records deleted.** 
Execute the same query used before the `DELETE` step to validate records no longer exist. 

```mysql
SELECT ast.id audit_id, ast.student_id, ast.ssid, ast.action, ast.audited FROM audit_student ast
WHERE student_id IN (SELECT id FROM student WHERE ssid = 'SSID001');
```

```text
Empty set (0.00 sec)
```

#### Clear all exams for a specific student

**Step 1: Table `audit_exam_claim_score`.  Query records to delete.** 
The following query outputs the `audit_exam_claim_score` records to be deleted.

```mysql
SELECT aecs.id audit_id, aecs.subject_claim_score_id, student_exams.id exam_id, aecs.action, aecs.audited FROM audit_exam_claim_score aecs
JOIN (
       SELECT id FROM exam WHERE student_id IN ( SELECT id FROM student WHERE ssid = 'SSID001')
     ) student_exams
ON aecs.exam_id = student_exams.id;
```

```text
+----------+------------------------+---------+--------+----------------------------+
| audit_id | subject_claim_score_id | exam_id | action | audited                    |
+----------+------------------------+---------+--------+----------------------------+
|        1 |                      7 |       2 | update | 2017-12-04 19:12:41.951761 |
|        2 |                      5 |       2 | update | 2017-12-04 19:12:41.952197 |
|        3 |                      4 |       2 | update | 2017-12-04 19:12:41.952457 |
|        4 |                      6 |       2 | update | 2017-12-04 19:12:41.952693 |
+----------+------------------------+---------+--------+----------------------------+
4 rows in set (0.00 sec)
```

**Step 2: Table `audit_exam_claim_score`.  Delete records.** 
The following statement deletes records with the same `FROM` clause as the previous statement.

```mysql
DELETE aecs FROM audit_exam_claim_score aecs
JOIN (
       SELECT id FROM exam WHERE student_id IN ( SELECT id FROM student WHERE ssid = 'SSID001')
     ) student_exams
ON aecs.exam_id = student_exams.id;
```

```text
Query OK, 4 rows affected (0.01 sec)
```

**Step 3: Table `audit_exam_claim_score`.  Validate records deleted.** 
Execute the same query used before the `DELETE` step to validate records no longer exist. 

```mysql
SELECT aecs.id audit_id, aecs.subject_claim_score_id, student_exams.id exam_id, aecs.action, aecs.audited FROM audit_exam_claim_score aecs
JOIN (
       SELECT id FROM exam WHERE student_id IN ( SELECT id FROM student WHERE ssid = 'SSID001')
     ) student_exams
ON aecs.exam_id = student_exams.id;
```

```text
Empty set (0.00 sec)
```

**Step 4: Table `audit_exam_available_accommodation`.  Query records to delete.** 
The following query outputs the `audit_exam_available_accommodation` records to be deleted.

```mysql
SELECT aeaa.id audit_id, aeaa.accommodation_id, student_exams.id exam_id, aeaa.action, aeaa.audited FROM audit_exam_available_accommodation aeaa
JOIN (
       SELECT id FROM exam WHERE student_id IN ( SELECT id FROM student WHERE ssid = 'SSID001')
     ) student_exams
ON aeaa.exam_id = student_exams.id;
```

```text
+----------+------------------+---------+--------+----------------------------+
| audit_id | accommodation_id | exam_id | action | audited                    |
+----------+------------------+---------+--------+----------------------------+
|        1 |               32 |       1 | delete | 2017-12-04 19:04:27.827775 |
|        2 |                5 |       2 | delete | 2017-12-04 19:04:28.045994 |
+----------+------------------+---------+--------+----------------------------+
2 rows in set (0.00 sec)
```

**Step 5: Table `audit_exam_available_accommodation`.  Delete records.** 
The following statement deletes records with the same `FROM` clause as the previous statement.

```mysql
DELETE aeaa FROM audit_exam_available_accommodation aeaa
JOIN (
       SELECT id FROM exam WHERE student_id IN ( SELECT id FROM student WHERE ssid = 'SSID001')
     ) student_exams
ON aeaa.exam_id = student_exams.id;
```

```text
Query OK, 2 rows affected (0.00 sec)
```

**Step 6: Table `audit_exam_available_accommodation`.  Validate records deleted.** 
Execute the same query used before the `DELETE` step to validate records no longer exist. 

```mysql
SELECT aeaa.id audit_id, aeaa.accommodation_id, student_exams.id exam_id, aeaa.action, aeaa.audited FROM audit_exam_available_accommodation aeaa
JOIN (
       SELECT id FROM exam WHERE student_id IN ( SELECT id FROM student WHERE ssid = 'SSID001')
     ) student_exams
ON aeaa.exam_id = student_exams.id;
```

```text
Empty set (0.00 sec)
```

**Step 7: Table `audit_exam_item`.  Query records to delete.** 
The following query outputs the `audit_exam_item` records to be deleted.

```mysql
SELECT aei.id audit_id, aei.exam_item_id, student_exams.id exam_id, aei.action, aei.audited FROM audit_exam_item aei
JOIN (
       SELECT id FROM exam WHERE student_id IN ( SELECT id FROM student WHERE ssid = 'SSID001')
     ) student_exams
ON aei.exam_id = student_exams.id;
```

```text
+----------+--------------+---------+--------+----------------------------+
| audit_id | exam_item_id | exam_id | action | audited                    |
+----------+--------------+---------+--------+----------------------------+
|        1 |            2 |       1 | update | 2017-12-04 18:59:14.263477 |
|        2 |            3 |       1 | update | 2017-12-04 18:59:14.264113 |
|        3 |            5 |       1 | delete | 2017-12-04 18:59:14.266069 |
|        4 |            8 |       2 | update | 2017-12-04 18:59:14.514066 |
|        5 |            7 |       2 | delete | 2017-12-04 18:59:14.514715 |
+----------+--------------+---------+--------+----------------------------+
5 rows in set (0.00 sec)
```

**Step 8: Table `audit_exam_item`.  Delete records.** 
The following statement deletes records with the same `FROM` clause as the previous statement.

```mysql
DELETE aei FROM audit_exam_item aei
JOIN (
       SELECT id FROM exam WHERE student_id IN ( SELECT id FROM student WHERE ssid = 'SSID001')
     ) student_exams
ON aei.exam_id = student_exams.id;
```

```text
Query OK, 5 rows affected (0.01 sec)
```

**Step 9: Table `audit_exam_item`.  Validate records deleted.** 
Execute the same query used before the `DELETE` step to validate records no longer exist. 

```mysql
SELECT aei.id audit_id, aei.exam_item_id, student_exams.id exam_id, aei.action, aei.audited FROM audit_exam_item aei
JOIN (
       SELECT id FROM exam WHERE student_id IN ( SELECT id FROM student WHERE ssid = 'SSID001')
     ) student_exams
ON aei.exam_id = student_exams.id;
```

```text
Empty set (0.00 sec)
```

**Step 10: Table `audit_exam`.  Query records to delete.** 
The following query outputs the `audit_exam` records to be deleted.

```mysql
SELECT ae.id, ae.exam_id, ae.oppId, ae.student_id, ae.action, ae.audited FROM audit_exam ae
JOIN (
       SELECT id FROM exam WHERE student_id IN ( SELECT id FROM student WHERE ssid = 'SSID001')
     ) student_exams
ON ae.exam_id = student_exams.id;
```

```text
+----+---------+--------------+------------+--------+----------------------------+
| id | exam_id | oppId        | student_id | action | audited                    |
+----+---------+--------------+------------+--------+----------------------------+
|  1 |       1 | 100000000010 |          1 | update | 2017-12-04 17:19:03.837310 |
|  2 |       2 | 100000000030 |          1 | update | 2017-12-04 17:19:03.913670 |
|  3 |       2 | 100000000030 |          1 | update | 2017-12-04 17:19:04.072135 |
|  5 |       2 | 100000000030 |          1 | update | 2017-12-04 17:19:04.204870 |
+----+---------+--------------+------------+--------+----------------------------+
4 rows in set (0.00 sec)
```

**Step 11: Table `audit_exam`.  Delete records.** 
The following statement deletes records with the same `FROM` clause as the previous statement.

```mysql
DELETE ae FROM audit_exam ae
JOIN (
       SELECT id FROM exam WHERE student_id IN ( SELECT id FROM student WHERE ssid = 'SSID001')
     ) student_exams
ON ae.exam_id = student_exams.id;
```

```text
Query OK, 4 rows affected (0.01 sec)
```

**Step 12: Table `audit_exam`.  Validate records deleted.** 
Execute the same query used before the `DELETE` step to validate records no longer exist. 

```mysql
SELECT ae.id, ae.exam_id, ae.oppId, ae.student_id, ae.action, ae.audited FROM audit_exam ae
JOIN (
       SELECT id FROM exam WHERE student_id IN ( SELECT id FROM student WHERE ssid = 'SSID001')
     ) student_exams
ON ae.exam_id = student_exams.id;
```

```text
Empty set (0.00 sec)
```

#### Clear a specific exam record

To clear a specific exams audit records modify the `SELECT` statement that is joined against the audit table and substitute it in the same steps as [Clear all exams for a specific student](#clear-all-exams-for-a-specific-student).

For example the following `JOIN` statement can be substituted for each of the tables,
 `audit_exam_claim_score`, `audit_exam_available_accommodation`, `audit_exam_item` and `audit_exam` to clear one exams audit records by it's `oppId`  

```mysql
JOIN (
       SELECT id FROM exam WHERE oppId = '100000000030'
     ) specific_exam
```

The following query demonstrates this substitution for the `audit_exam` table.  
Use the same `SELECT` to collect exam id's to join the audit table in each statement to query before, delete and query after for each of the above mentioned exam related audit tables.

```mysql
SELECT ae.id, ae.exam_id, ae.oppId, ae.student_id, ae.action, ae.audited FROM audit_exam ae
JOIN (
       SELECT id FROM exam WHERE oppId = '100000000030'
     ) specific_exam
ON ae.exam_id = specific_exam.id;
```

#### Clear all exams for a school

To clear all exam audit records for a specific school modify the `SELECT` statement that is joined against the audit table and substitute it in the same steps as [Clear all exams for a specific student](#clear-all-exams-for-a-specific-student).

For example the following `JOIN` statement can be substituted for each of the tables,
 `audit_exam_claim_score`, `audit_exam_available_accommodation`, `audit_exam_item` and `audit_exam` to clear all exam audit records for the school with a `natural_id` of `TS000001`  

```mysql
JOIN (
       SELECT id FROM exam WHERE school_id IN ( SELECT id FROM school WHERE natural_id = 'TS000001')
     ) school_exams
```

The following query demonstrates this substitution for the `audit_exam` table.  
Use the same `SELECT` to collect exam id's to join the audit table in each statement to query before, delete and query after for each of the above mentioned exam related audit tables.

```mysql
SELECT ae.id, ae.exam_id, ae.oppId, ae.school_id, ae.student_id, ae.action, ae.audited FROM audit_exam ae
JOIN (
       SELECT id FROM exam WHERE school_id IN ( SELECT id FROM school WHERE natural_id = 'TS000001')
     ) school_exams
ON ae.exam_id = school_exams.id;
```

#### Clear all exams for a specific assessment

To clear all exam audit records for a specific assessment modify the `SELECT` statement that is joined against the audit table and substitute it in the same steps as [Clear all exams for a specific student](#clear-all-exams-for-a-specific-student).

For example the following `JOIN` statement can be substituted for each of the tables,
 `audit_exam_claim_score`, `audit_exam_available_accommodation`, `audit_exam_item` and `audit_exam` to clear all exam audit records for the assessment with a `natural_id` of `(naturalId)MOCK-ICA-G11-2017-2018`

```mysql
JOIN (
       SELECT id FROM exam WHERE asmt_id IN ( SELECT id FROM asmt WHERE natural_id = '(naturalId)MOCK-ICA-G11-2017-2018')
     ) asmt_exams
```

The following query demonstrates this substitution for the `audit_exam` table.  
Use the same `SELECT` to collect exam id's to join the audit table in each statement to query before, delete and query after for each of the above mentioned exam related audit tables.

```mysql
SELECT ae.id, ae.exam_id, ae.oppId, ae.student_id, ae.action, ae.audited FROM audit_exam ae
JOIN (
       SELECT id FROM exam WHERE asmt_id IN ( SELECT id FROM asmt WHERE natural_id = '(naturalId)MOCK-ICA-G11-2017-2018')
     ) asmt_exams
ON ae.exam_id = asmt_exams.id;
```

#### Clear a specific student group

**Step 1: Table `audit_user_student_group`.  Query records to delete.** 
The following query outputs the `audit_user_student_group` records to be deleted.

```mysql
SELECT ausg.id, ausg.student_group_id, ausg.user_login, ausg.action, ausg.audited FROM audit_user_student_group ausg
JOIN (
       SELECT id FROM student_group
       WHERE school_id IN ( SELECT id from school WHERE natural_id = 'TS000001')
             AND name = 'StudentGroup001'
             AND school_year = '2018'
     ) specific_group
ON ausg.student_group_id = specific_group.id;
```

```text
+----+------------------+-----------------------+--------+----------------------------+
| id | student_group_id | user_login            | action | audited                    |
+----+------------------+-----------------------+--------+----------------------------+
|  1 |                1 | teacher01@example.com | delete | 2017-12-06 04:37:23.022623 |
|  2 |                1 | teacher03@example.com | delete | 2017-12-06 04:37:23.083122 |
|  3 |                1 | teacher04@example.com | delete | 2017-12-06 04:37:23.083122 |
+----+------------------+-----------------------+--------+----------------------------+
3 rows in set (0.00 sec)
```

**Step 2: Table `audit_user_student_group`.  Delete records.** 
The following statement deletes records with the same `FROM` clause as the previous statement.

```mysql
DELETE ausg FROM audit_user_student_group ausg
JOIN (
       SELECT id FROM student_group
       WHERE school_id IN ( SELECT id from school WHERE natural_id = 'TS000001')
             AND name = 'StudentGroup001'
             AND school_year = '2018'
     ) specific_group
ON ausg.student_group_id = specific_group.id;
```

```text
Query OK, 3 rows affected (0.01 sec)
```

**Step 3: Table `audit_user_student_group`.  Validate records deleted.** 
Execute the same query used before the `DELETE` step to validate records no longer exist. 

```mysql
SELECT ausg.id, ausg.student_group_id, ausg.user_login, ausg.action, ausg.audited FROM audit_user_student_group ausg
JOIN (
       SELECT id FROM student_group
       WHERE school_id IN ( SELECT id from school WHERE natural_id = 'TS000001')
             AND name = 'StudentGroup001'
             AND school_year = '2018'
     ) specific_group
ON ausg.student_group_id = specific_group.id;
```

```text
Empty set (0.00 sec)
```

**Step 4: Table `audit_student_group_membership`.  Query records to delete.** 
The following query outputs the `audit_student_group_membership` records to be deleted.

```mysql
SELECT asgm.id, asgm.student_group_id, asgm.student_id, asgm.action, asgm.audited FROM audit_student_group_membership asgm
JOIN (
       SELECT id FROM student_group
       WHERE school_id IN ( SELECT id from school WHERE natural_id = 'TS000001')
             AND name = 'StudentGroup001'
             AND school_year = '2018'
     ) specific_group
ON asgm.student_group_id = specific_group.id;
```

```text
+----+------------------+------------+--------+----------------------------+
| id | student_group_id | student_id | action | audited                    |
+----+------------------+------------+--------+----------------------------+
|  1 |                1 |          2 | delete | 2017-12-06 04:37:23.024463 |
|  2 |                1 |          3 | delete | 2017-12-06 04:37:23.024463 |
|  4 |                1 |          6 | delete | 2017-12-06 04:37:23.085580 |
+----+------------------+------------+--------+----------------------------+
3 rows in set (0.00 sec) 
```

**Step 5: Table `audit_student_group_membership`.  Delete records.** 
The following statement deletes records with the same `FROM` clause as the previous statement.

```mysql
DELETE asgm FROM audit_student_group_membership asgm
JOIN (
       SELECT id FROM student_group
       WHERE school_id IN ( SELECT id from school WHERE natural_id = 'TS000001')
             AND name = 'StudentGroup001'
             AND school_year = '2018'
     ) specific_group
ON asgm.student_group_id = specific_group.id;
```

```text
Query OK, 3 rows affected (0.01 sec)
```

**Step 6: Table `audit_student_group_membership`.  Validate records deleted.** 
Execute the same query used before the `DELETE` step to validate records no longer exist. 

```mysql
SELECT asgm.id, asgm.student_group_id, asgm.student_id, asgm.action, asgm.audited FROM audit_student_group_membership asgm
JOIN (
       SELECT id FROM student_group
       WHERE school_id IN ( SELECT id from school WHERE natural_id = 'TS000001')
             AND name = 'StudentGroup001'
             AND school_year = '2018'
     ) specific_group
ON asgm.student_group_id = specific_group.id;
```

```text
Empty set (0.00 sec)
```

**Step 7: Table `audit_student_group`.  Query records to delete.** 
The following query outputs the `audit_student_group` records to be deleted.

```mysql
SELECT asg.id, asg.student_group_id, asg.name, asg.action, asg.audited FROM audit_student_group asg
JOIN (
       SELECT id FROM student_group
       WHERE school_id IN ( SELECT id from school WHERE natural_id = 'TS000001')
             AND name = 'StudentGroup001'
             AND school_year = '2018'
     ) specific_group
ON asg.student_group_id = specific_group.id;
```

```text
+----+------------------+-----------------+--------+----------------------------+
| id | student_group_id | name            | action | audited                    |
+----+------------------+-----------------+--------+----------------------------+
|  1 |                1 | StudentGroup001 | update | 2017-12-06 04:37:22.944438 |
|  2 |                1 | StudentGroup001 | update | 2017-12-06 04:37:23.019759 |
|  5 |                1 | StudentGroup001 | update | 2017-12-06 04:37:23.081925 |
+----+------------------+-----------------+--------+----------------------------+
3 rows in set (0.00 sec)
```

**Step 8: Table `audit_student_group`.  Delete records.** 
The following statement deletes records with the same `FROM` clause as the previous statement.

```mysql
DELETE asg FROM audit_student_group asg
JOIN (
       SELECT id FROM student_group
       WHERE school_id IN ( SELECT id from school WHERE natural_id = 'TS000001')
             AND name = 'StudentGroup001'
             AND school_year = '2018'
     ) specific_group
ON asg.student_group_id = specific_group.id;
```

```text
Query OK, 3 rows affected (0.00 sec)
```

**Step 9: Table `audit_student_group`.  Validate records deleted.** 
Execute the same query used before the `DELETE` step to validate records no longer exist. 

```mysql
SELECT asg.id, asg.student_group_id, asg.name, asg.action, asg.audited FROM audit_student_group asg
JOIN (
       SELECT id FROM student_group
       WHERE school_id IN ( SELECT id from school WHERE natural_id = 'TS000001')
             AND name = 'StudentGroup001'
             AND school_year = '2018'
     ) specific_group
ON asg.student_group_id = specific_group.id;
```

```text
Empty set (0.00 sec)
```

#### Clear all student groups for a specific school

To clear all student group audit records for a specific school modify the `SELECT` statement that is joined against the audit table and substitute it in the same steps as [Clear a specific student group](#clear-a-specific-student-group).

For example the following `JOIN` statement can be substituted for each of the tables,
 `audit_user_student_group`, `audit_student_group_membership` and `audit_student_group` to clear all student group audit records for the school with a `natural_id` of `TS000001`

```mysql
JOIN (
       SELECT id FROM student_group
       WHERE school_id IN ( SELECT id from school WHERE natural_id = 'TS000001')
     ) school_groups
```

The following query demonstrates this substitution for the `audit_student_group` table.  
Use the same `SELECT` to collect student_group id's to join the audit table in each statement to query before, delete and query after for each of the above mentioned student group related audit tables.

```mysql
SELECT asg.id, asg.student_group_id, asg.name, asg.action, asg.audited FROM audit_student_group asg
JOIN (
       SELECT id FROM student_group
       WHERE school_id IN ( SELECT id from school WHERE natural_id = 'TS000001')
     ) school_groups
ON asg.student_group_id = school_groups.id;
```