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

---
[1]: http://www.smarterapp.org/documents/TestResultsTransmissionFormat.pdf
[2]: http://www.smarterapp.org/documents/TestResults-DataModel.pdf

