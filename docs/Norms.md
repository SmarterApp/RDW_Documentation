## Norms
This document defines the Smarter Balanced Norms CSV format.

### Table of Contents

* [Intended audience](#intended-audience)
* [CSV Format Field Definitions](#csv-format-field-definitions)
* [Import Rules](#import-rules)
* [Samples](#samples)

### Intended audience
- **System Administrators**: This document provides system administrators information on creating and updating norms data in the data warehouse.
- **Producers of Normative Data**: This document provides information to anyone developing percentile tables of normative test results to be loaded into the data warehouse.

### CSV Format Field Definitions

| Field               | Data Type       | Description                                     | Identity Row | Rank Row |
|---------------------|-----------------|-------------------------------------------------|--------------|----------|
| assessment_id       | text to 250     | assessment natural id                           | required     | required |
| start_date          | YYYY-MM-DD      | date equal to or greater than test completed    | required     | required |
| end_date            | YYYY-MM-DD      | date equal to or less than for test completed   | required     | required |
| count               | integer         | number of test results in normative data set    | required     | ignored  |
| mean                | float           | mean score for the normative data set           | required     | ignored  |
| standard_deviation  | float           | standard deviation for the normative data set   | optional     | ignored  |
| min_score           | float           | lowest possible score for this assessment       | optional     | ignored  |
| max_score           | float           | highest possible score for this assessment      | optional     | ignored  |
| percentile_rank     | integer         | percentile rank, e.g. 15 is 15%                 | required     | required |
| score               | float           | minimum inclusive score for percentile rank     | required     | required |

### Import Rules
- **Percentile Table Identifier**: The first three columns, `assessment_id`, `start_date`, and `end_date` uniquely identify a percentile table.
  - **Create**: Loading a percentile table with a unique identifier that does not exist creates it.
  - **Update**: Loading a percentile table with the same unique identifier replaces the existing table.
  - **Overlap**: Loading a percentile table with the same `assessment_id` and overlapping `start_date` and `end_date` produces a validation error.
- **Identity Row**: The first row in a CSV import file with a unique id is the `Identity Row` for a percentile table.  Rows following the identity row with the same percentile table identifier are the `Rank Row`'s.  Following the rank rows, the same percentile table id can not be repeated in a single import file.
  - **Required Fields**: The `Identity Row` column in the above table specifies required fields for an identity row. If the optional `min_score` and `max_score` are not specified their value comes from configuration properties `sbac.scaleScore.min` and `sbac.scaleScore.max`. The first `percentile_rank` and `score` is specified on the identity row.
- **Rank Rows**: A minimum of three percentile ranks are required for each percentile table.
  - **Required Fields**: The `Rank Row` column in the above table specifies required fields for a rank row. The identity fields must match the identity row.  Fields marked `ignored` should be left empty.  The `percentile_rank` and `score` are required.
  - **Order**: The `percentile_rank` must be greater than the previous rows `percentile_rank` if it is not the first row.  The `score` must be greater or equal to the previous rows `score` if it is not the first row.
- **Validation**: Refer to [API - Get Import Request](API.md#get-import-request) on viewing the status of a submitted import request.  The response from getting an import request includes a message.  If the request is not successsful the message indicates what is wrong with the import file. No percentile tables are loaded if there are any errors.  The import process attempts to evaluate the entire import request and report all validation errors if there are any.

                                                                                                                                                                          
### Samples
**Sample 1**: The following sample defines four percentile tables.  Two for assessment `(SBAC)SBAC-IAB-FIXED-G4M-OA-MATH-4-Winter-2017-2018` and two for `(SBAC)SBAC-IAB-FIXED-G3M-Perf-OrderForm-MATH-3-Winter-2017-2018`. Both assessments have one table for date range 2017-08-01 to 2018-01-31 and one for date range 2018-02-01 to 2018-07-31. Each of the four percentile tables have three ranks, 25%, 50% and 75%.

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

**Sample2**: The following sample defines two percentile tables for the assessment `(SBAC)SBAC-IAB-FIXED-G11E-BriefWrites-ELA-11-Winter-2016-2017`. The first table has a date range of 2016-08-01 to 2017-01-31 and the second table has a date range of 2017-02-01 to 2017-07-31. Each of the two percentile tables have 9 ranks from 10% to 90% by ten.
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