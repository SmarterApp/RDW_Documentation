## Norms
This document defines the Smarter Balanced Norms CSV format.

### Table of Contents

* [Intended audience](#intended-audience)
* [CSV format](#csv-format)
    * [Rules](#rules)
    * [Samples](#samples)

### Intended audience
The intended audience should be familiar with database technology and querying a database with SQL.
- **System Administrators**: This document provides administrators information on creating and updating norms data in the data warehouse.
- **Producers of Normative Data**: This document provides information to anyone developing percentile tables of normative test results to be loaded in the data warehouse.

### CSV format

| Column              | Data Type       | Description                                       | first row | rank rows |
|---------------------|-----------------|---------------------------------------------------|-----------|-----------|
| assessment_id       | text to 250     | assessment natural id                             | required  | required  |
| start_date          | YYYY-MM-DD      | date equal to or greater than for a test result   | required  | required  |
| end_date            | YYYY-MM-DD      | date equal to or less than for a test result      | required  | required  |
| count               | Child           | number of test results in normative data set      | required  | ignored   |
| mean                | Parent          | mean score of normative data                      | required  | ignored   |
| standard_deviation  | Child           | standard deviation for normative data set         | optional  | ignored   |
| min_score           | Parent          | lowest possible score for this assessment         | optional  | ignored   |
| max_score           | Child           | highest possible score for this assessment        | optional  | ignored   |
| percentile_rank     | Child           | percentile rank, 15 is 15%                        | required  | required  |
| score               | Child           | minimum inclusive score for this percentile rank  | required  | required  |


#### Rules
- **Percentile Table Identifier**: The first three columns, `assessment_id`, `start_date`, and `end_date` uniquely identify a percentile table.
  - **Update**: Loading a percentile table with the same unique identifier replaces the existing table.
  - **Overlap**: Loading a percentile table with the same `assessment_id` and overlapping `start_date` and `end_date` produces a validation error.
- **Identifying Row**: 
                                                                                                                                                                          
#### Samples
Sample 1
```csv
assessment_id,start_date,end_date,count,mean,standard_deviation,min_score,max_score,percentile_rank,score
(SBAC)SBAC-IAB-FIXED-G4M-OA-MATH-4-Winter-2017-2018,2017-08-01,2018-01-31,812345,2425.5,88.9,,,25,2365
(SBAC)SBAC-IAB-FIXED-G4M-OA-MATH-4-Winter-2017-2018,2017-08-01,2018-01-31,,,,,,50,2425
(SBAC)SBAC-IAB-FIXED-G4M-OA-MATH-4-Winter-2017-2018,2017-08-01,2018-01-31,,,,,,75,2495
(SBAC)SBAC-IAB-FIXED-G4M-OA-MATH-4-Winter-2017-2018,2018-02-01,2018-07-31,812345,2430.5,88.9,,,25,2370
(SBAC)SBAC-IAB-FIXED-G4M-OA-MATH-4-Winter-2017-2018,2018-02-01,2018-07-31,,,,,,50,2430
(SBAC)SBAC-IAB-FIXED-G4M-OA-MATH-4-Winter-2017-2018,2018-02-01,2018-07-31,,,,,,75,2500
(SBAC)SBAC-IAB-FIXED-G3M-Perf-OrderForm-MATH-3-Winter-2017-2018,2017-08-01,2018-01-31,812345,2425.5,88.9,,,25,2365
(SBAC)SBAC-IAB-FIXED-G3M-Perf-OrderForm-MATH-3-Winter-2017-2018,2017-08-01,2018-01-31,,,,,,50,2425
(SBAC)SBAC-IAB-FIXED-G3M-Perf-OrderForm-MATH-3-Winter-2017-2018,2017-08-01,2018-01-31,,,,,,75,2495
(SBAC)SBAC-IAB-FIXED-G3M-Perf-OrderForm-MATH-3-Winter-2017-2018,2018-02-01,2018-07-31,812345,2430.5,88.9,,,25,2370
(SBAC)SBAC-IAB-FIXED-G3M-Perf-OrderForm-MATH-3-Winter-2017-2018,2018-02-01,2018-07-31,,,,,,50,2430
(SBAC)SBAC-IAB-FIXED-G3M-Perf-OrderForm-MATH-3-Winter-2017-2018,2018-02-01,2018-07-31,,,,,,75,2500
```

Sample 2
```csv
assessment_id,start_date,end_date,count,mean,standard_deviation,min_score,max_score,percentile_rank,score
(SBAC)SBAC-IAB-FIXED-G11E-BriefWrites-ELA-11-Winter-2016-2017,2016-08-01,2017-01-31,812345,2425.5,88.9,,,10,2295
(SBAC)SBAC-IAB-FIXED-G11E-BriefWrites-ELA-11-Winter-2016-2017,2016-08-01,2017-01-31,,,,,,20,2335
(SBAC)SBAC-IAB-FIXED-G11E-BriefWrites-ELA-11-Winter-2016-2017,2016-08-01,2017-01-31,,,,,,30,2365
(SBAC)SBAC-IAB-FIXED-G11E-BriefWrites-ELA-11-Winter-2016-2017,2016-08-01,2017-01-31,,,,,,40,2395
(SBAC)SBAC-IAB-FIXED-G11E-BriefWrites-ELA-11-Winter-2016-2017,2016-08-01,2017-01-31,,,,,,50,2425
(SBAC)SBAC-IAB-FIXED-G11E-BriefWrites-ELA-11-Winter-2016-2017,2016-08-01,2017-01-31,,,,,,60,2445
(SBAC)SBAC-IAB-FIXED-G11E-BriefWrites-ELA-11-Winter-2016-2017,2016-08-01,2017-01-31,,,,,,70,2475
(SBAC)SBAC-IAB-FIXED-G11E-BriefWrites-ELA-11-Winter-2016-2017,2016-08-01,2017-01-31,,,,,,80,2505
(SBAC)SBAC-IAB-FIXED-G11E-BriefWrites-ELA-11-Winter-2016-2017,2016-08-01,2017-01-31,,,,,,90,2565
(SBAC)SBAC-IAB-FIXED-G11E-BriefWrites-ELA-11-Winter-2016-2017,2017-02-01,2017-07-31,812345,2430.5,88.9,,,10,2300
(SBAC)SBAC-IAB-FIXED-G11E-BriefWrites-ELA-11-Winter-2016-2017,2017-02-01,2017-07-31,,,,,,20,2340
(SBAC)SBAC-IAB-FIXED-G11E-BriefWrites-ELA-11-Winter-2016-2017,2017-02-01,2017-07-31,,,,,,30,2370
(SBAC)SBAC-IAB-FIXED-G11E-BriefWrites-ELA-11-Winter-2016-2017,2017-02-01,2017-07-31,,,,,,40,2400
(SBAC)SBAC-IAB-FIXED-G11E-BriefWrites-ELA-11-Winter-2016-2017,2017-02-01,2017-07-31,,,,,,50,2430
(SBAC)SBAC-IAB-FIXED-G11E-BriefWrites-ELA-11-Winter-2016-2017,2017-02-01,2017-07-31,,,,,,60,2450
(SBAC)SBAC-IAB-FIXED-G11E-BriefWrites-ELA-11-Winter-2016-2017,2017-02-01,2017-07-31,,,,,,70,2480
(SBAC)SBAC-IAB-FIXED-G11E-BriefWrites-ELA-11-Winter-2016-2017,2017-02-01,2017-07-31,,,,,,80,2510
(SBAC)SBAC-IAB-FIXED-G11E-BriefWrites-ELA-11-Winter-2016-2017,2017-02-01,2017-07-31,,,,,,90,2570
```