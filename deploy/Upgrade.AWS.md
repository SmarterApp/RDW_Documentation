## Upgrade v1.1 <- v1.0

**Intended Audience**: this document provides detailed instructions for upgrading the [Reporting Data Warehouse (RDW)](../README.md) applications in an AWS environment from v1.0.x to v1.1. Operations and system administrators will find it useful.

It is assumed that the [Deployment Checklist](Deployment.AWS.md) was used for the installation of v1.0. Please refer to that documentation for general guidelines.

This is the first major update to RDW. It adds "Phase 2" functionality. 
* Aggregate reporting.
* District/School groups (from ART).
* Combine webapp applications (i.e. admin-webapp goes away)

Quick list of changes; place to keep notes until we finalize this
1. Redshift.
1. Changes to configuration files.
    1. Remove rdw-admin-webapp.yml
    1. Remove rdw-reporting-admin-webapp.yml (if present)
    1. Add rdw-ingest-migrate-olap.yml
    1. Add rdw-reporting-admin-service.yml
    1. Add rdw-reporting-aggregate-service.yml
    1. Add rdw-reporting-service.yml
    1. Update rdw-ingest-task-service.yml: add groups-of-districts/schools-url setting
    1. Update ... TODO (all the config files need to be updated, need to document)
        1. Update datasource URL configuration
1. Changes to Kubernetes configuration files.
    1. Add aggregate-service.yml
    1. Add migrate-olap-service.yml
    1. Rename reporting-service.yml to reporting-webapp.yml
    1. Add reporting-service.yml
    1. Update admin-service.yml
    1. Update image version in all configuration files.


