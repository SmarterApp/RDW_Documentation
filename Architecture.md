# Architecture Overview

The RDW is divided into sub-systems based on data, performance and scalability requirements. For Phase 1 these include:
* The Data Warehouse. This is where data enters the system. It is the collection of all assessment definitions, institutions, students, student groups, and test results. It includes supplemental reference data such as subject claims and traits, common core standards, accessibility/accommodation codes and descriptions, ethnicities, and grades. 
* The Reporting Data Mart. This is the data store for the reporting web UI. The test results have been optimized for presentation to teachers and administrators to allow them to quickly view results.

The RDW uses other systems from the SBAC Open Test System environment, including:
* OpenAM for OAuth2 and Single Sign On (SSO).
* ART for management of users (teachers and administrators) and institutions (schools and districts).
* Perm service for management of roles and permissions for the reporting component.
* IRIS for displaying test items as the students see them, as well as for looking up rubric and example data.

