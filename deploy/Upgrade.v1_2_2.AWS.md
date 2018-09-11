## Upgrade v1.2.2 <- v1.2.1

**Intended Audience**: this document provides detailed instructions for upgrading the [Reporting Data Warehouse (RDW)](../README.md) applications in an AWS environment from v1.2.1 to v1.2.2. Operations and system administrators will find it useful.

It is assumed that the official deployment and upgrade instructions were used for the current installation. Please refer to that documentation for general guidelines.

### Overview

This is a very minor upgrade to RDW that normally wouldn't deserve dedicated documentation since it can be deployed
with no downtime using a rolling update. However, there is one new capability that deserves call-out.

* Bug fix for some schools not appearing in the District/School Export screen.
* Add exam transformation capability.

> Note: these instructions assume that the bug fix for the reporting-webapp have not been deployed.
> Because that upgrade was available more than a month ago, it may have already been done and those steps may be skipped.

### Prep Work

As noted in [Configurable Data Transformation](../docs/Runbook.DataSpecifications#configurable-data-transformation) this
feature allows an XSLT to be applied to TRTs as they are processed. Before upgrading the exam-processor service, the
desired XSLT file should be authored and tested. Once it is working, it needs to be added to the repository for the
configuration server.

Because the changes are minimal and will be applied immediately these instructions do not use branches to make changes.
If desired, branching may still be done to adhere to internal processes.

* [ ] Make changes to the configuration repository (in the `master` branch).
    * Add a folder, `xslt`, and the XSLT file, e.g. `exam.xsl`.
    * Modify `rdw-ingest-exam-processor.yml` to specify the exam transformation:
    ```yaml
    transformations:
      exam: "binary-${spring.cloud.config.uri}/*/*/master/xslt/exam.xsl"
    ```
    * Commit the changes.
* [ ] Make changes to the deployment repository (in the `master` branch).
NOTE: only two services *need* to be upgraded to 1.2.2. The other services *may* be upgraded but have no changes since 1.2.1.
    * Modify `reporting-webapp.yml` and set the image to `smarterbalanced/rdw-reporting-webapp:1.2.2-RELEASE`
    * Modify `exam-processor-service.yml` and set the image to `smarterbalanced/rdw-ingest-exam-processor:1.2.2-RELEASE`
    * Commit the changes.
* [ ] Upgrade the services.
    * Using Kubernetes, this will do a rolling upgrade without downtime.
    ```bash
    # from the deployment repository folder
    git checkout master; git pull
    kubectl apply -f reporting-webapp.yml
    kubectl apply -f exam-processor-service.yml
    ```

### Smoke Test
Smoke 'em if ya got 'em.         
