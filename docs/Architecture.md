## Architecture 

**Intended Audience**: This document describes the overall architecture of the Reporting Data Warehouse (RDW). It includes a discussion of the various applications, data flow, deployment philosophy, etc.

### Overview

The RDW is divided into sub-systems based on data, performance and scalability requirements. These include:

* **Data Warehouse**. This is where data enters the system. It is the collection of all assessment definitions, institutions, students, student groups, and test results. It includes supplemental reference data such as subject claims and traits, common core standards, accessibility/accommodation codes and descriptions, ethnicities, and grades.
* **Reporting**. This is the main reporting data mart and web UI. The test results have been optimized for presentation to teachers and administrators to allow them to quickly view results.
* **Aggregate Reporting**. This is the system for viewing test results through the lens of aggregation, trends, etc. The test results have been optimized for slicing and dicing.

The RDW uses other systems from the SBAC Open Test System environment, including:

* **OpenAM** for OAuth2 and Single Sign On (SSO).
* **ART** for management of users (teachers and administrators) and institutions (schools and districts).
* **Perm** service for management of roles and permissions for the reporting component.
* **IRIS** for displaying test items as the students see them, as well as for looking up rubric and example data.

![RDW Overview](rdw-overview.png)


### Applications

To make deployment easier, all the RDW applications have been containerized. The orchestration of the applications is handled with Kubernetes. The current deployment is in **AWS** so kops is used to configure the many resources needed by the cluster.

* **Redshift**. Amazon OLAP database.

![RDW Processes](rdw-processes.png)

*The* ***bold*** *boxes represent services that are deployed as kubernetes deployments; multiple boxes represent horizontally scalable processes with replicas > 1. Most of the data flow lines have not been included to avoid making the diagram unreadable.*

**RabbitMQ**. The message queue is an infrastructure service (like the database or config repository); however, it is deployed in a container as part of the cluster so is called out here. Note that the queue is not used as the sole storage for any critical data so an HA configuration is good for system reliability but not necessary for data integrity.
* Resubmitting unprocessed data (daily).
