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

Warehouse tables audited:

| Table                        | Description                                 | Entity Type | Ingest State Changes        |
|------------------------------|---------------------------------------------|-------------|-----------------------------|
| exam                         | One record per test result                  | Parent      | Create, Update, Soft Delete |
| exam_available_accommodation | One record per exam available accommodation | Child       | Create, Delete              |
| exam_claim_score             | One record per exam claim                   | Child       | Create, Update              |
| exam_item                    | One record per exam item                    | Child       | Create, Update, Delete      |
| student                      | One record per student                      | Parent      | Create, Update, Soft Delete |
| student_ethnicity            | One record per student ethnicity            | Child       | Create, Delete              |
| student_group                | One record per student group                | Parent      | Create, Update, Soft Delete |
| student_group_membership     | One record per student membership in group  | Child       | Create, Delete              |
| user_student_group           | One record per user with access to group    | Child       | Create, Delete              |


### Where is audit data stored?
Each audited table has an `audit_...` table that records the state change for each row.  The audit tables contain the state of the row before the change.  In addition to the columns from the table being audited, each audit_ table has the following columns:

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
| audit_student                      | student                      | Update, Soft Delete(as update) |
| audit_student_ethnicity            | student_ethnicity            | Delete                         |
| audit_student_group                | student_group                | Update, Soft Delete(as update) |
| audit_student_group_membership     | student_group_membership     | Delete                         |
| audit_user_student_group           | user_student_group           | Delete                         |

### How can audit data be queried?
Sample queries are provided for analysing audit data combining the warehouse import table, the table being audited, the audit_ table and joining other relations in the warehouse for lookup values.

#### Exam

**Finding modifications to a students exams**
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
The following query outputs one row for each exam for one student with a count of modifications and the date of the most recent update.  If the exam has not been modified `exam_update_count` is zero.

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

The query can be modified to display different or all columns.  The `WHERE` clause can be modified, such as, to include multiple exams or all exams for a student.

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

The query can be modified to display different or all columns.  The `WHERE` clause can be modified, such as, to include multiple exams or all exams for a student.


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

#### Student

**Finding modified students**
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

**Finding modifications to students**
The following query outputs one row for each student.  It includes the count of modifications and the date of the last change.
Students with no modifications have a `student_update_count` of `0`.

A `WHERE` clause can be added to filter students.

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
GROUP BY s.id;
```

```text
+---------+------------+-------------+-----------------+----------------------+----------------------------+
| ssid    | first_name | middle_name | last_or_surname | student_update_count | last_update                |
+---------+------------+-------------+-----------------+----------------------+----------------------------+
| SSID001 | Gladys     | Ruth        | Williams        |                    6 | 2017-11-10 09:26:31.858088 |
| SSID002 | Joe        |             | Smith           |                    1 | 2017-11-10 09:23:01.576030 |
| SSID003 | Mark       |             | Anderson        |                    1 | 2017-11-10 09:25:21.846849 |
| SSID004 | Linda      |             | Smart           |                    0 | NULL                       |
+---------+------------+-------------+-----------------+----------------------+----------------------------+
4 rows in set (0.00 sec)
```

**Finding modifications to students in a date range**
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

**Student audit trail**
The following query lists the details of student modifications in addition to the current state.

The query can be modified to display different or all columns.  The `WHERE` clause can be modified, such as, to include multiple students or all students in a school.

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
**Ethnicity audit trail for students**
Each child table audit trail can be queried in a similar manner.  The following example is for the `student_ethnicity` child table.  Student currently has one child table.

The query can be modified to display different or all columns.  The `WHERE` clause can be modified, such as, to include multiple students or all students in a school.


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

#### Student Groups

**Finding modified student groups**
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

**Finding modifications to student groups**
The following query outputs one row for each student_group.  It includes the count of modifications and the date of the last change.
Students with no modifications have a `update_count` of `0`.

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

**Finding modifications to student groups in a date range**
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

**Student group audit trail**
The following query lists the details of student group modifications in addition to the current state.

The query can be modified to display different or all columns.  The `WHERE` clause can be modified, such as, to include multiple student groups or all student groups in a district.

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
**Student membership audit trail**
Each child table audit trail can be queried in a similar manner.  For student_groups the child tables are `student_group_membership` and `user_student_group`.  The following example is for `student_group_membership`.  

The query can be modified to display different or all columns.  The `WHERE` clauses can be modified, such as, to include multiple student_groups or all student_groups in a district.


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