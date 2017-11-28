# Reporting Data Warehouse
The Reporting Data Warehouse (RDW) is a repository for SBAC test results where those results can be viewed and analyzed: from individual student item responses, to aggregate performance results, to longitudinal trends.

**Intended Audience**: this document serves as a convenient entry point to all things RDW. Anybody interested in developing, deploying or maintaining the RDW project should start here. That includes developers, operations, system administrators, tier 3 support, etc.

Before updating resources in this project, please reference [Contributing](CONTRIBUTING.md).

## Document Links
RDW is a suite of applications with lots of moving parts. These documents provide additional information for understanding, deploying and maintaining them.

1. [Main README (this file)](README.md)
1. [Architecture](docs/Architecture.md)
1. [Runbook](docs/Runbook.md)
1. [Monitoring](docs/Monitoring.md)
1. [Troubleshooting](docs/Troubleshooting.md)
1. [Deployment Checklist](deploy/Deployment.AWS.md)
1. [Performance Tuning for Aurora](docs/PerformanceTuning.Aurora.md)
1. [Performance Tuning for Redshift](docs/PerformanceTuning.Redshift.md)
1. [API - Ingest](docs/API-Ingest.md)
1. [Auditing](docs/Audit.md)

## Project Repositories
RDW is separated into multiple project repositories. Each repo has documentation for building and testing.

1. [Ingest](https://github.com/SmarterApp/RDW_Ingest) is the primary data store for the system and is responsible for accepting and processing most data coming into the system.
1. [Reporting](https://github.com/SmarterApp/RDW_Reporting) is the UI (and supporting stack) for end-users to view test results.
1. [Common](https://github.com/SmarterApp/RDW_Common) is a collection of modules shared by multiple projects.
1. [Schema](https://github.com/SmarterApp/RDW_Schema) has scripts for creating the various database schemas.


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
