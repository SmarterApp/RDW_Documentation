## Student Groups

**Intended Audience**: this document provides file format information for anyone creating student group files to be loaded into the data warehouse.

This document defines the Smarter Balanced Student Group CSV format. Files in this format may be uploaded in the 
Manage Student Groups are of the reporting web application. A template for the file may be downloaded there.

### CSV Format Field Definitions

The following fields are supported in the CSV file. The first three fields, `group_name`, `school_natural_id` and 
`school_year` uniquely identify a group and are required in every row. The other fields can be optionally specified 
to add a student and/or user to a group. In other words, a row may contain either a `student_ssid`, a `group_user_login` or both.

| Field             | Data Type | Description                          |
|-------------------|-----------|--------------------------------------|
| group_name        | text      | group name                           |
| school_natural_id | text      | school id                            |
| school_year       | integer   | school year, e.g. 2018 for 2017-18   |
| subject_code      | text      | All, Math, ELA                       |
| student_ssid      | text      | student ssid                         |
| group_user_login  | text      | user login                           |

### Samples

This sample defines two groups for 2017-18 for the school with id 88800120012001. Each group has three students and
two teachers associated with them. 
```csv
group_name,school_natural_id,school_year,subject_code,student_ssid,group_user_login
JT4thGrade,88800120012001,2018,All,,
JT4thGrade,88800120012001,2018,,,jteacher@example.com
JT4thGrade,88800120012001,2018,,,btutor@example.com
JT4thGrade,88800120012001,2018,,SSID001,
JT4thGrade,88800120012001,2018,,SSID002,
JT4thGrade,88800120012001,2018,,SSID003,
ME6thGradeMath,88800120012001,2018,Math,,
ME6thGradeMath,88800120012001,2018,,,meducator@example.com
ME6thGradeMath,88800120012001,2018,,,sinstructor@example.com
ME6thGradeMath,88800120012001,2018,,SSID010,
ME6thGradeMath,88800120012001,2018,,SSID011,
ME6thGradeMath,88800120012001,2018,,SSID012,
```