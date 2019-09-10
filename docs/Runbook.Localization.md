## Localization

**Intended Audience**: This document contains information on how localization works in the [Reporting Data Warehouse](../README.md) (RDW). Operations and system administration will find it useful.

### Table of Contents

* [Overview](#overview)
* [Configuration](#configuration)
* [Categories](#categories)

### Overview

Localization is the mechanism used to configure text elements within the Reporting Data Warehouse application.  The [default localization](https://github.com/SmarterApp/RDW_Reporting/blob/master/webapp/src/main/webapp/src/assets/i18n/en.json) localizations can be overridden and are [multi-tenant aware](Runbook.MultiTenancy.md).

### Configuration

Localization file locations are described in the [multi-tenancy runbook configuration](Runbook.MultiTenancy.md#configuration) section.

Instance-level localization is specified in files stored (by convention) in the `i18n` folder.  

Tenant-specific localization is set in `xx.json` files in the tenant folder. Note there is no `i18n` folder for tenants.

Tenant specific overrides can be set with the tenant and sandbox configuration tools or edited directly.

The typical work-flow to override a localization consists of the following steps:

1. Identify the text in the user interface that should be changes.
2. Search the existing configuration for that text in the tenant administration tool or the default localization file directly.
3. Add an override to localization file.

If there are multiple uses of that text, the following breakdown of the configuration categories may be helpful identifying the correct configuration to change.

### Categories

The localization file is a hierarchy organized JSON document, the categories here represent the first level of the hierarchy.

| Category                            | Description                                                                        |
|-------------------------------------|------------------------------------------------------------------------------------|
| access-denied                       | Security common access denied UI component for permissions errors.                 |
| activate-pipeline-modal             | Pipeline administration activate UI popup component.                               |
| admin-dropdown                      | Common application administration dropdown labels.                                 |
| admin-groups                        | Groups administration UI component labels and messages.                            |
| adv-filters                         | Assessments advanced student filter UI component.                                  |
| advanced-filters                    | Student advanced filters label.                                                    |
| aggregate-report                    | Aggregate report shared common labels used by many aggregate report UI components. |
| aggregate-report-form               | Aggregate report entry form UI component elements.                                 |
| aggregate-report-table              | Aggregate report table UI component labels for columns and other table features.   |
| aggregate-reports                   | Aggregate reports section headings.                                                |
| aggregate-reports-summary           | Aggregate report summary inclusion messaging.                                      |
| assessment-card                     | Student assessment card labels.                                                    |
| assessment-percentile-history       | Assessment percentiles history UI elements.                                        |
| assessment-percentile-table         | Assessment percentiles table column headers and labels.                            |
| assessment-results                  | Assessments results UI component elements.                                         |
| Assessments                         | Assessments core UI component titles, headers, and other elements.                 |
| average-scale-score                 | Exam score table, claim score summary, scale score, and aggregate target labels.   |
| code-editor                         | Pipeline code editor keyboard trap prompt.                                         |
| Common                              | Cross site common labels such as “OK”, “Save”, “Hide” and many others.             |
| common-ngx                          | Application wide common such as the title bar, user drop down and others.          |
| create-instructional-resource-modal | Instructional resource creation modal labels and titles.                           |
| csv-builder                         | CSV export related primarily CSV column headers.                                   |
| deactivate-pipeline-modal           | Pipeline administration, deactivate UI popup component.                            |
| delete-group-modal                  | Groups administration delete popup UI component elements.                          |
| delete-instructional-resource-modal | Instructional resource administration, delete popup UI component elements.         |
| delete-modal                        | Common “Are you sure?” style prompt for delete operations.                         |
| embargo                             | Embargo administration common labels used in embargo components.                   |
| embargo-alert                       | Embargo administration alert component, alert message label.                       |
| embargo-confirmation-modal          | Embargo administration conformation popup, label titles and messages.              |
| embargo-table                       | Embargo administration table column headers and other elements.                    |
| exam-filter                         | Assessment exam filter elements.                                                   |
| file-format                         | Groups administration file format UI labels.                                       |
| group-dashboard                     | Group dashboard heading and name.                                                  |
| group-import                        | Groups administration import UI elements.                                          |
| group-results                       | Group result UI component labels.                                                  |
| groups                              | Groups administration common UI elements, column headers and other labels.         |
| home                                | Home page UI component labels.                                                     |
| html                                | Home page system news message.                                                     |
| import-history                      | Groups administration import history UI component elements.                        |
| import-table                        | Groups administration import table UI component headers and other labels.          |
| info-button                         | Common “more info” button label.                                                   |
| ingest-pipeline                     | Ingest pipeline administration common name and descriptions.                       |
| instructional-resource              | Instructional resource administration common title and label elements.             |
| item-exemplar                       | Assessments item exemplar component UI labels.                                     |
| item-info                           | Assessments item info component UI labels and descriptions.                        |
| item-scores                         | Assessments item info scores related labels.                                       |
| item-tab                            | Assessments item tab UI component.                                                 |
| item-viewer                         | Assessments item viewer UI component.                                              |
| item-writing-trait-score            | Assessments item writing trait UI component.                                       |
| longitudinal-cohort-chart           | Aggregate report longitudinal cohort chart UI component.                           |
| order-selector                      | Common shared order selector UI component.                                         |
| organization-export                 | Organization export UI component common labels and descriptions.                   |
| organization-tree                   | Organization export tree UI component.                                             |
| organization-typeahead              | Organization export typeahead UI component.                                        |
| pipeline                            | Ingest pipeline administration common labels and messages.                         |
| pipeline-compilation-state          | Ingest pipeline administration compilation state messages.                         |
| pipeline-editor                     | Ingest pipeline administration editor labels.                                      |
| pipeline-explorer                   | Ingest pipeline administration explorer labels.                                    |
| pipeline-item                       | Ingest pipeline administration item labels.                                        |
| pipeline-publish-state              | Ingest pipeline administration publish state labels.                               |
| pipeline-published-scripts          | Ingest pipeline administration published script labels.                            |
| pipeline-publishing-history         | Ingest pipeline administration publishing history labels.                          |
| pipeline-save-state                 | Ingest pipeline administration save state labels.                                  |
| pipeline-test-form                  | Ingest pipeline administration test form labels.                                   |
| pipeline-test-result                | Ingest pipeline administration test result labels.                                 |
| pipeline-test-results               | Ingest pipeline administration test results heading.                               |
| pipeline-test-state                 | Ingest pipeline administration test state labels.                                  |
| pipelines                           | Ingest pipeline administration common headings and menu labels.                    |
| popup-menu                          | Common shared menu UI component labels.                                            |
| report                              | Report UI component labels and headers.                                            |
| report-action                       | Aggregate report UI component action labels.                                       |
| report-download                     | Printable report download labels and messages.                                     |
| reports                             | Reports common labels and messages.                                                |
| results-by-student                  | Assessments results by student UI component labels.                                |
| sample-aggregate-table-data-service | Aggregate report sample table labels.                                              |
| sandbox-login                       | Sandbox login UI component elements.                                               |
| school-and-group-typeahead          | Student search form typeahead labels.                                              |
| school-grade                        | School grade UI component elements.                                                |
| school-results                      | School grade results UI component labels.                                          |
| school-typeahead                    | School grade results typeahead component.                                          |
| score-table                         | Exam score table UI component labels.                                              |
| select-assessments                  | Assessments selector component.                                                    |
| student                             | Common student related component messages and labels.                              |
| student-assessment-card             | Student history table assessment card elements.                                    |
| student-exam-history-table          | Student exam table component labels.                                               |
| student-responses                   | Student response breadcrumb and title header.                                      |
| student-results                     | Student results labels and messages.                                               |
| student-search-form                 | Student search form labels.                                                        |
| target-report                       | Assessment target report messages, headers, and labels.                            |
| tenant                              | Tenant administration labels common elements.                                      |
| tenant-configuration-form           | Tenant administration form labels.                                                 |
| tenant-form                         | Tenant administration form labels.                                                 |
| tenant-link                         | Tenant administration tenant link UI component.                                    |
| tenant-localization-form            | Tenant administration localization form UI component.                              |
| tenant-metric-table                 | Tenant administration metrics table UI component.                                  |
| tenant-metric-type                  | Tenant administration metrics type specific elements.                              |
| tenant-metrics                      | Tenant administration metrics common labels.                                       |
| tenant-type                         | Tenant administration form element.                                                |
| tenants                             | Tenant administration common labels.                                               |
| update-instructional-resource-modal | Instructional resource administration update form elements.                        |
| user-group                          | User group common title, headings and labels.                                      |
| user-group-form                     | User group form labels.                                                            |
| user-group-table                    | User group table column headers and other elements.                                |
| user-groups                         | User group listing labels.                                                         |
| user-query                          | Aggregate report user query labels.                                                |
| user-query-menu-option              | Aggregate report user query options.                                               |
| user-query-table                    | User query table UI component column headers.                                      |
| user-report-table                   | User report table UI component column headers.                                     |
| writing-trait-scores                | Assessments writing trait scores UI component labels.                              |


