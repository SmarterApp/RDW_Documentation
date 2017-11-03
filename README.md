# Reporting Data Warehouse
The Reporting Data Warehouse (RDW) is a repository for SBAC test results where those results can be viewed and analyzed: from individual student item responses, to aggregate performance results, to longitudinal trends.

**Intended Audience**: this document serves as a convenient entry point to all things RDW. Anybody interested in developing, deploying or maintaining the RDW project should start here.

* TODO - clean up Runbook + Monitoring + Troubleshooting
    * Runbook should be mostly configuration; do we need multiple "chapters"?
    * Monitoring
    * Troubleshooting - perhaps two: flowchart-style "what's the problem", and a scenario-based (problem specific) guide 

## Overview Links
RDW is a suite of applications with lots of moving parts. These documents provide additional information for understanding, deploying and maintaining them.

1. [Main README (this file)](README.md)
1. [Architecture](Architecture.md)
1. [Runbook](Runbook.md)
1. [Monitoring](Monitoring.md)
1. [Troubleshooting](Troubleshooting.md)
1. [API - Ingest](API-Ingest.md)

## Project Links
RDW is separated into multiple project repositories. Each repo has documentation for building and testing.

1. [Ingest](https://github.com/SmarterApp/RDW_Ingest)
1. [Reporting](https://github.com/SmarterApp/RDW_Reporting) 
1. [Common](https://github.com/SmarterApp/RDW_Common)
1. [Schema](https://github.com/SmarterApp/RDW_Schema)


## License
RDW is owned by Smarter Balanced and covered by the Educational Community License:

```text
Copyright 2017 Smarter Balanced Licensed under the
Educational Community License, Version 2.0 (the "License"); you may
not use this file except in compliance with the License. You may
obtain a copy of the License at http://www.osedu.org/licenses/ECL-2.0.

Unless required by applicable law or agreed to in writing,
software distributed under the License is distributed on an "AS IS"
BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express
or implied. See the License for the specific language governing
permissions and limitations under the License.
```
