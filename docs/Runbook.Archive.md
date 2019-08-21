## Archive RDW Instance Database

**Intended Audience**: This document provides instructions for creating and restoring an archive of the [Reporting Data Warehouse](../README.md) (RDW). Access to the production databases is required. Operations will find this useful if archiving older content (and deleting it from the current database) is required.

Archiving content includes:

* Creating a snapshot of the current database and storing it in an archival location
* [Bulk delete](Runbook.BulkDeleteExams.md) content (if applicable)

Restoring content includes:

* Restoring a snapshot
* Querying the restored snapshot of a database for information

Archiving and restoring content may be done in one of two ways:

* Using the AWS Management Console
* Command Line Interface (CLI) commands

> **NOTE**: It is **required that these tasks are performed during the maintenance window, and while the system is quiescent and the exam processors are paused**.


### Creating a Snapshot

> **NOTE**: If automated backups are available, the most recent snapshot may be copied instead of creating a new snapshot.

#### Creating a snapshot using the AWS Management Console

Archiving content may be done by creating a snapshot of the database using the AWS Management Console as follows:

1.	Sign in to the AWS Management Console and open the Amazon RDS console at [https://console.aws.amazon.com/rds/](https://console.aws.amazon.com/rds/). 
1.	In the navigation pane, click **DB Instances** and select the DB instance to take a snapshot of. 
1.	Click **Instance Actions**, and then click **Take DB Snapshot**. The **Take DB Snapshot** window appears. 
1.	Type the name of the snapshot in the **Snapshot Name** text box and then click **Yes, Take Snapshot**.
1.	A snapshot of the database will be created.

These steps are shown in detail in the **AWS Management Console** section of: [https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/USER_CreateSnapshot.html](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/USER_CreateSnapshot.html)

#### Creating a snapshot using the Command Line Interface (CLI)

Archiving content may be done by creating a snapshot of the database using the [create‑db‑snapshot](http://docs.aws.amazon.com/cli/latest/reference/rds/create-db-snapshot.html) CLI command. Two parameters are needed:

* --db-instance-identifier: The name of the database to back up
* --db-snapshot-identifier: The name of the snapshot

The command is:

`aws rds create-db-snapshot --db-instance-identifier mydbinstance --db-snapshot-identifier mydbsnapshot `

These steps are shown in detail in the **CLI** section of: [https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/USER_CreateSnapshot.html](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/USER_CreateSnapshot.html)


### Restoring a Snapshot

#### AWS Management Console
Restoring content may be done by restoring a snapshot of the database using the AWS Management Console as follows:

1.	Sign in to the AWS Management Console and open the Amazon RDS console at [https://console.aws.amazon.com/rds/](https://console.aws.amazon.com/rds/). 
1.	In the navigation pane, choose **Snapshots** and choose the DB snapshot to be restored.
1.	From the **Actions** drop-down, choose **Restore Snapshot**. 
1.	On the **Restore DB Instance** page, in the **DB Instance Identifier** field, type the name for your restored DB instance. 
1.	Choose **Restore DB Instance**. 
1.	Modify the DB instance to use the desired security group.

These steps are shown in detail in the **AWS Management Console** section of: 
[https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/USER_RestoreFromSnapshot.html](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/USER_RestoreFromSnapshot.html)

#### Restoring a snapshot using the Command Line Interface (CLI)
Restoring content may be done by restoring a snapshot of the database using the [restore‑db‑instance‑from-db‑snapshot](http://docs.aws.amazon.com/cli/latest/reference/rds/create-db-snapshot.html) CLI command. Two parameters are needed:

* --db-instance-identifier: The name of the database to restore to
* --db-snapshot-identifier: The name of the snapshot to restore from

The command is:

`aws rds restore-db-instance-from-db-snapshot --db-instance-identifier mynewdbinstance --db-snapshot-identifier mydbsnapshot`

After the DB instance has been restored, you must add the DB instance to the security group and parameter group used by the DB instance used to create the DB snapshot if you want the same functionality as that of the previous DB instance. 

These steps are shown in detail in the **CLI** section of: 
[https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/USER_RestoreFromSnapshot.html](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/USER_RestoreFromSnapshot.html)
