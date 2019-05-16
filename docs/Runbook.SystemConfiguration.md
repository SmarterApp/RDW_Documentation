## Runbook - System Configuration

**Intended Audience**: This document contains information for the general configuration of the [Reporting Data Warehouse](../README.md) (RDW). Additional information is available in the main [Runbook](Runbook.md). Operations and system administration may find it useful.

### Table of Contents

* [System Configuration](#system-configuration)
    * [School Year](#school-year)
    * [Subjects](#subjects)
    * [Assessment Packages](#assessment-packages)
    * [Accommodations](#accommodations)
    * [Instructional Resources](#instructional-resources)
    * [Normative Data](#normative-data)
    * [Student Groups](#student-groups)
    * [Target Exclusions](#target-exclusions)
    * [Transfer Enabled](#transfer-enabled)
    * [Student Fields](#student-fields)
    * [Language Codes](#language-codes)
    * [English Learners](#english-learners)
    * [Ethnicity](#ethnicity)
    * [Military Connected](#military-connected)
* [System Configuration Changes for a New School Year](#new-school-year)

<a name="system-configuration"></a>
### System Configuration

Once the system is deployed it is necessary to configure the system by loading subjects, assessments, accommodations, instructional resources, student groups, normative data. There are also end-user features that may be enabled or disabled.

#### School Year
The system restricts reporting to the "known" school years. Verify that the `school_year` table has all the desired years (usually this means adding the upcoming school year to the table) and trigger a `CODES` migration as described in [Manual Data Modifictions](./Runbook.ManualDataModifications).
```sql
mysql> USE warehouse;
mysql> SELECT * FROM school_year;
+------+
| year |
+------+
| 2015 |
| 2016 |
| 2017 |
| 2018 |
+------+
mysql> INSERT INTO school_year (year) VALUES (2019);
mysql> INSERT INTO import(status, content, contentType, digest) VALUES (1, 3, 'add school year 2019', 'add school year 2019');
```

The embargo feature requires the current school year be set. In the common configuration file (usually `application.yml`) set `reporting.school_year` to the appropriate value and restart the migration and reporting applications.

#### Subjects

The subject XML defines a subject's attributes for the RDW system. It is the tenant's responsibility to define a subject XML based on the schema, [RDW_Subject.xsd](https://github.com/SmarterApp/RDW_Common/blob/master/model/src/main/resources/RDW_Subject.xsd). SmarterBalanced's [Math](../deploy/Math_subject.xml) and [ELA](../deploy/ELA_subject.xml) subjects XML may be found in the `deploy` folder of this project. Subjects must be loaded into the system before assessment packages. The system allows for subject updates but only for the data attributes that have not been used by the system at the time of the update.
We have also proved two additional sample subject definition XMLs as samples/templates for additional subjects: [Sample Subject](../deploy/new_subject_config.xml) and [Mini Subject.](../deploy/mini_subject_config.xml)
Loading the packages is an IT/DevOps function and requires data load permissions:
```bash
export ACCESS_TOKEN=`curl -s -X POST --data 'grant_type=password&username=rdw-ingest-opus@sbac.org&password=password&client_id=rdw&client_secret=password' 'https://sso.sbac.org/auth/oauth2/access_token?realm=/sbac' | jq -r '.access_token'`
curl -X POST --header "Authorization: Bearer ${ACCESS_TOKEN}" -F file=@Math_subject.csv https://import.sbac.org/subjects/imports
```
>NOTE: To optimize system performance, subject data is cached in the Exam Processor and Reporting services (Admin Service, Aggregate Service, Reporting Service, Report Processor, and Webapp). When subjects are loaded please do the following:
> 1. Stop all Exam Processors before ingesting a subject XML
> 2. Ingest the subject XML
> 3. Re-start the Exam Processors after the successful subject changes
> 4. Wait for migration from warehouse to reporting to complete
> 5. Re-start the Reporting services

#### Assessment Packages

The assessment packages define the tests that are administered to the students. They include performance parameters which enable the student results to be appropriately interpreted.
SmarterBalanced provides these packages; specifically, there is a `Tabulator` tool which produces a CSV file that is loaded into the warehouse. In general this will be done during the break between school years, but sometimes updates are necessary to correct data. *Note that `subject` data must be loaded before loading assessments*. Loading the packages is an IT/DevOps function and requires data load permissions:
```bash
export ACCESS_TOKEN=`curl -s -X POST --data 'grant_type=password&username=rdw-ingest-opus@sbac.org&password=password&client_id=rdw&client_secret=password' 'https://sso.sbac.org/auth/oauth2/access_token?realm=/sbac' | jq -r '.access_token'`
curl -X POST --header "Authorization: Bearer ${ACCESS_TOKEN}" -F file=@2017-2018.csv https://import.sbac.org/packages/imports
```

#### Accommodations

Accommodations describe inline and external resources that may be made available to students during testing. Loading them into the warehouse allows them to be displayed properly in student reports.
SmarterBalanced provides this [file][1]. In general this will be loaded at the same time as the assessment packages. Loading the file is an IT/DevOps function and requires data load permissions:
```bash
export ACCESS_TOKEN=`curl -s -X POST --data 'grant_type=password&username=rdw-ingest-opus@sbac.org&password=password&client_id=rdw&client_secret=password' 'https://sso.sbac.org/auth/oauth2/access_token?realm=/sbac' | jq -r '.access_token'`
curl -X POST --header "Authorization: Bearer ${ACCESS_TOKEN}" -F file=@accommodations.xml https://import.sbac.org/accommodations/imports
```

#### Instructional Resources

Instructional resources are links to content that help teachers address common core topics.
Although organization-specific resources may be configured in the system by administrative users, the SmarterBalanced "system" resources must be loaded by IT/DevOps. There is a SQL script to facilitate this; because these resources are proprietary they are not included in this public repository.
```bash
mysql -h rdw-prod-reporting-cluster.cluster-cimuvo5urx1e.us-west-2.rds.amazonaws.com -u root -p < load_instructional_resources.sql
```

#### Normative Data

Normative or percentile data allows teachers and administrators to compare their students' performance against the entire student population.
SmarterBalanced provides this data. The system must be configured to enable the viewing of percentiles, and the data must be loaded. This will be done by IT/DevOps.
To enable the feature, modify the `application.yml` configuration file (the reporting application must be restarted after this change):
```yml
reporting:
  percentile-display-enabled: true
```
Then load the [normative data](Norms.md). *Note that assessments must be loaded before normative data*.
```bash
export ACCESS_TOKEN=`curl -s -X POST --data 'grant_type=password&username=rdw-ingest-opus@sbac.org&password=password&client_id=rdw&client_secret=password' 'https://sso.sbac.org/auth/oauth2/access_token?realm=/sbac' | jq -r '.access_token'`
curl -X POST --header "Authorization: Bearer ${ACCESS_TOKEN}" -F file=@norms.csv https://import.sbac.org/norms/imports
```

#### Student Groups

Student groups provide a focused view of test results for teachers and school administrators. The system supports "assigned" groups configured by administrators and "teacher-created" groups. The teacher created groups are managed by teachers in the reporting UI but the assigned groups must be loaded into the system either in the admin section or by directly posting files to the RESTful API.
The reporting UI, including the admin section, is documented in the user guide. Posting files is described in detail in [Student Groups](StudentGroups.md).

#### Target Exclusions

Assessment items are categorized into broad "claims" and more specific "targets". The reporting UI provides target-level reporting. But not all targets have enough coverage to provide statistically significant conclusions so they are excluded from these reports (note that these exclusions are applied on top of other restrictions in the system, for example, target reports are only available for summative assessments, only claim 1 math targets are included, etc.)
Because the exclusions vary for assessments, the system allows these exclusions to be configured. This is done by IT/DevOps adding entries to a table (after modifying the table, a migration must be triggered as well).
How the assessments and targets are determined will depend on the source of the knowledge. Typically the natural ids are known, and the target's claim must be included for uniqueness, so the SQL for a single exclusion might look like:
```sql
INSERT INTO asmt_target_exclusion
  SELECT a.id AS asmt_id, t.id AS target_id FROM
    (SELECT id FROM asmt WHERE natural_id = '(SBAC)SBAC-OP-G5E-2017-COMBINED-Spring-2017-2018') a
    JOIN
    (SELECT t.id FROM target t JOIN claim c ON t.claim_id = c.id WHERE c.code = '1-LT' AND t.natural_id = '5-4') t

-- trigger migration
INSERT INTO import(status, content, contentType, digest)
  SELECT 0, ic.id, 'target exclusions', left(uuid(), 8) from import_content ic where name = 'PACKAGE';

SELECT LAST_INSERT_ID() INTO @import_id;
UPDATE asmt SET update_import_id = @import_id WHERE natural_id = '(SBAC)SBAC-OP-G5E-2017-COMBINED-Spring-2017-2018';
UPDATE import SET status = 1 WHERE id = @import_id;
```

#### Transfer Enabled

The system controls visibility of students and their test results based on permissions granted at the institution level. For example, a teacher may have PII group permissions to see test results for their class given at their school. There is an optional feature that allows users to see test results for their students that were administered at another institution. By default this feature is disabled. To enable it, IT/DevOps may add the following property to the `application.yml` configuration file:
```yml
reporting:
  transfer-access-enabled: true
```
Any time a configuration option is changed, the affected services must be restarted. For the transfer enabled flag, restart the `report-processor` instances.


#### Student Fields

The system controls visibility into student demographic data for teachers. By default all student fields are available to all users. To change that behavior, modify properties in `rdw-reporting-service.yml`. Fields can be completely disabled, enabled for administrators but not teachers or enabled for all users including teachers.

This example configuration is similar to California's policy: they use ELAS instead of LEP, nobody is allowed to see the students' economic disadvantage, and their teachers are restricted to just four fields (ELAS, migrant status, language, race/ethnicity):
```yml
reporting:
  student-fields:
    EconomicDisadvantage: disabled
    LimitedEnglishProficiency: disabled
    EnglishLanguageAcquisitionStatus: enabled
    MigrantStatus: enabled
    PrimaryLanguage: enabled
    Ethnicity: enabled
    Gender: admin
    IndividualEducationPlan: admin
    MilitaryStudentIdentifier: admin
    Section504: admin
```
Any time a configuration option is modified, the affected services must be restarted. For this setting, restart the `reporting-webapp` instances.


#### Language Codes

The system is configured with the known language codes. These are used to validate the student primary language if specified in the TRT test results. And to allow filtering and aggregation in reports.
To change the list of valid language codes, modify the `warehouse.language` table and trigger a `CODES` migration, for example:
```sql
USE warehouse;
-- change valid languages for tenant
INSERT INTO language (id, code, altcode, display_order, name) VALUES (319, 'non', NULL, 99, 'Norse');
DELETE FROM language WHERE code = 'mkh';
-- trigger CODES migration:
INSERT INTO import (status, content, contentType, digest) VALUES (1, 3, 'update language', REPLACE(UUID(), '-', ''));
```

#### English Learners

There are different systems for categorizing english learners. The system supports two of them, Limited English Proficiency (LEP) and English Language Acquisition Status (ELAS). Only one should be used, the default is ELAS; this should correspond to the other systems in the testing ecosystem, especially the test delivery system. To switch to use LEP, IT/DevOps may change the following property in the `application.yml` configuration file:
```yml
reporting:
  english-learners:
    - lep
```
This setting is used by all the reporting services so they should be restarted: `aggregate-service`, `report-processor`, `reporting-service`, `reporting-webapp`.

#### Ethnicity

Although there are federal standards for student ethnicity, some states have different requirements. To change the list of ethnicities in the system there are two things that must be changed: the allowed ethnicities (in the database) and the ethnicity list in the language file.

Modifying the database is a [manual data modification](./Runbook.ManualDataModifications.md). Add or delete entries in the ethnicity table and then trigger a migration:
```sql
USE warehouse;
-- change ethnicity table
DELETE FROM ethnicity WHERE code = 'Filipino';
-- trigger CODES migration:
INSERT INTO import (status, content, contentType, digest) VALUES (1, 3, 'update ethnicity', REPLACE(UUID(), '-', ''));
```

Updating the language file is detailed in [language support](./Runbook.LanguageSupport.md). The section that needs to be updated (for this example, remove the `"Filipino"` line):
```json
{
  ...
  "common": {
    ...
    "ethnicity": {
      "AmericanIndianOrAlaskaNative": "American Indian or Alaska Native",
      "Asian": "Asian",
      "BlackOrAfricanAmerican": "Black or African American",
      "DemographicRaceTwoOrMoreRaces": "Demographic Race of Two or More",
      "Filipino": "Filipino",
      "HispanicOrLatinoEthnicity": "Hispanic/Latino",
      "NativeHawaiianOrOtherPacificIslander": "Native Hawaiian or Pacific Islander",
      "White": "White"
    },
```

<a name="military-conntected"></a>
#### Military Student Connected Identifier

There are ESSA guidelines for the military student connected identifier. However some states, California in particular, have different requirements. To change the list of these codes in the system, the allowed values in the database must be set. It is expected the allowed values will be either the ESSA set, a simplified yes/no set, or the superset of both (if a tenant has changed adherence policy over the years). NOTE: the id/code values must not be changed if they are already in use. In that situation, new values may be added but existing used values should remain unchanged.

Modifying the database is a [manual data modification](./Runbook.ManualDataModifications.md). Add or delete entries in the military_connected table and then trigger a migration:
```sql
USE warehouse;
-- query for use of the field
SELECT military_connected_id, count(*) FROM exam GROUP BY military_connected_id;

-- ESSA values
INSERT INTO military_connected (id, code) VALUES (1, 'NotMilitaryConnected'), (2, 'ActiveDuty'), (3, 'NationalGuardOrReserve');
-- CA simplified values
INSERT INTO military_connected (id, code) VALUES (4, 'No'), (5, 'Yes');

-- trigger CODES migration:
INSERT INTO import (status, content, contentType, digest) VALUES (1, 3, 'update military connected codes', REPLACE(UUID(), '-', ''));
```


<a name="new-school-year"></a>
### System Configuration Changes for a New School Year

As the new school year approaches, there are configuration and data changes that will be necessary.

1. Update/Add subjects.
Every year, there may be changes to subjects, for example adjusting verbiage, adding standards, claims, etc. If the changes are aesthetic the subject may be updated: Load as [described above](#subjects).

> NOTE: Some changes to a subject may be incompatible with existing assessments and test results -- the system will reject the update. In this situation, some in-depth knowledge will be needed: Should the subject changes be reverted, is a new subject needed, etc.?

2. Add new year.
Add the new year and set the current school year as [described above](#school-year).

3. Load new assessment packages.
Assessment packages are updated every year so the new ones must be loaded as [described above](#assessment-packages).

4. Load new accessibility file.
Accommodation codes and translations are extracted from the accessibility file. A new one must be loaded every year as [described above](#accommodations).

5. User reports.
Although users can manage their own reports, it may be desirable to do a bulk purge of user reports from the previous school year.
NOTE: This will remove *all* user reports; selective deletion is tricky and beyond the scope of this document.

First, delete the database records for the reports.
```sql
DELETE FROM reporting.user_report;
```
Next, remove the artifacts from the (S3) archive.
```bash
aws s3 rm s3://rdw-archive/REPORTS/USER --recursive
```

6. Student groups.
This depends on policy, but it may be desirable to disable/remove student groups from the previous school year.

Student groups are associated with a school and a school year. They have an `active` flag and the system supports soft deletes. These features can be used to manipulate the groups (please refer to [Manual Data Modifications](Runbook.ManualDataModifications.md) for general data modification instructions). For example, to delete student groups from a particular district for the last school year, do something like:
```
INSERT INTO import (status, content, contentType, digest, creator) VALUES (0, 5, 'text/plain', left(uuid(), 8), 'dwtest@example.com');
SELECT LAST_INSERT_ID() into @importid;

UPDATE student_group sg
  JOIN school s ON sg.school_id = s.id
SET sg.deleted=1, sg.updated=@importid
WHERE sg.school_year=2018 AND s.district_id IN (1,3);

UPDATE import SET status=1 WHERE id=@importid;
```

---
[1]: https://github.com/SmarterApp/AccessibilityAccommodationConfigurations/blob/master/AccessibilityConfig.xml
