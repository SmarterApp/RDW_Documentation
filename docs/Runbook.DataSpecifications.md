## Runbook - Data Specifications

**Intended Audience**: This document contains information on data specifications in the [Reporting Data Warehouse](../README.md) (RDW). Additional information is available in the main [Runbook](Runbook.md). *Operations and system administration* may find this useful. It is a good reference for *developers*.

Things to put in here:

* links to SB TRT spec, logical data model, etc.
* discussion of required fields
* schemas for warehouse, reporting data mart, aggregate data mart
* 

### Test Results

Test results (aka exams) are pushed into the RDW using the [Test Result Transmission][1] (TRT) format using the [Exam Endpoints](API.md#exam-endpoints). The TRT is a flexible format so additional clarification may be found in the [Logical Data Model][2]. If there is a conflict between the two, the logical data model should be used as the source of truth.

This section further clarifies the data in the TRT, including requirements specific to the RDW.

| Mandatory Field | Comment |
| -------------- | ------- |
| Test@name | This is the unique assessment id |
| Test@subject | Used for validation |
| Test@grade | Used for validation |
| Test@assessmentType | Used for validation |
| Test@academicYear | Used for validation |
| Examinee - StudentIdentifier | This may be de-identified but must be the same  year over year |
| ExamineeRelationship - SchoolId | This may be de-identified but must be the same year over year, and the value must be in ART |
| ExamineeAttribute@GradeLevelWhenAssessed |  |
| Opportunity@oppId | This is the unique id for a test |
| Opportunity@dateCompleted |  |

All other TRT data elements could be configured to be required or optional. For example please refer to the [annotated configuration](../config/rdw-ingest-exam-processor.yml).

#### Missing Data

Many data fields in the test results are technically optional while being functionally required. There are is a SQL script
that can be found in [RDW_Schema](https://github.com/SmarterApp/RDW_Schema) as `warehouse/sql/missing_data_report.sql`
that queries for and calculates the percent of populated data for these fields. Here is a snippet showing the results
for student ethnicity:
```
+--------------------------+------------------+---------------------------------------------------------------------------+---------------+--------------------------------------+
| test_administration_year | assessment_db_id | asessment_natural_id                                                      | total_results | percent_of_results_with_student_race |
+--------------------------+------------------+---------------------------------------------------------------------------+---------------+--------------------------------------+
|                     2018 |                1 | (SBAC)SBAC-IAB-FIXED-G11E-BriefWrites-ELA-11-Winter-2017-2018             |          4712 |                              99.6392 |
|                     2018 |                2 | (SBAC)SBAC-IAB-FIXED-G11E-Editing-ELA-11-Winter-2017-2018                 |         43090 |                              99.6612 |
|                     2018 |                3 | (SBAC)SBAC-IAB-FIXED-G11E-LangVocab-ELA-11-Winter-2017-2018               |         67952 |                              99.2142 |
|                     2018 |                4 | (SBAC)SBAC-IAB-FIXED-G11E-ListenInterpet-ELA-11-Winter-2017-2018          |         62151 |                              99.2534 |
|                     2018 |                5 | (SBAC)SBAC-IAB-FIXED-G11E-Perf-Explanatory-Marshmallow-Winter-2017-2018   |         15793 |                              99.6454 |
|                     2018 |                6 | (SBAC)SBAC-IAB-FIXED-G11E-ReadInfo-ELA-11-Winter-2017-2018                |         47358 |                              99.6030 |
...
```
And another that tests the other optional fields (output pivoted for readability):
```
                        test_administration_year: 2016
                                assessment_db_id: 338
                            asessment_natural_id: (SBAC)SBAC-ICA-FIXED-G7M-COMBINED-Winter-2014-2015
                                   total_results: 15579
 percent_of_results_with_student_last_or_surname: 100.0000
        percent_of_results_with_student_birthday: 99.7753
          percent_of_results_with_student_gender: 100.0000
          percent_of_results_with_enrolled_grade: 100.0000
                     percent_of_results_with_iep: 100.0000
                     percent_of_results_with_lep: 100.0000
              percent_of_results_with_section504: 100.0000
   percent_of_results_with_economic_disadvantage: 100.0000
          percent_of_results_with_migrant_status: 26.3817
percent_of_results_with_administration_condition: 100.0000
            percent_of_results_with_completeness: 100.0000
              percent_of_results_with_session_id: 100.0000
```
> NOTE: these queries can be computationally expensive and should be executed with care.


---
[1]: http://www.smarterapp.org/documents/TestResultsTransmissionFormat.pdf
[2]: http://www.smarterapp.org/documents/TestResults-DataModel.pdf

