## Deprovisioning

**Intended Audience**: This document provides instructions for permanently deleting the [Reporting Data Warehouse](../README.md) (RDW) applications deployed in an AWS environment. Access to the production environment is required. Operations will find this useful when deprovisioning a production environment.

> Although these are generic instructions, having an example makes it easier to document and understand. Throughout this document we'll use opus as the sample environment name; this might correspond to staging or production in the real world. These are the same examples provided on the [Deployment Checklist](./deploy/Deployment.AWS.md).

Deprovisioning a tenant includes:

* Deleting the applications
* Deleting data
* Deleting AWS services


### Deleting the applications

**TODO: instructions for kubernetes and kops **

### Deleting the data

#### Amazon S3

To delete all the data stored in S3 for this tenant you will need to delete  the following buckets:

* kops state store bucket
* Archive bucket
* Reconciliation report bucket
 
##### AWS Management Console

1. Sign in to the AWS Management Console and open the Amazon S3 console at [https://console.aws.amazon.com/s3/](https://console.aws.amazon.com/s3/).
2. In the **Bucket name** list, choose the bucket icon next to the name of the bucket that you want to delete and then choose **Delete bucket**.
3. In the **Delete bucket** dialog box, type the name of the bucket that you want to delete for confirmation and then choose **Confirm**.

Amazon S3 buckets can be setup with versioning enabled.  In this case, deleting the individual objects is not enough to permanently remove the data as the previous versions are not removed.  To permanently delete all of the objects within a bucket, you can configure lifecycle on your bucket to expire objects. Amazon S3 then deletes expired objects. 

For more information see the AWS Documentation [How Do I Delete an S3 Bucket?](https://docs.aws.amazon.com/AmazonS3/latest/user-guide/delete-bucket.html)

**Delete the following S3 buckets:**

* [ ] kops state store bucket: kops-rdw-opus-state-store
* [ ] Archive bucket: rdw-opus-archive
* [ ] Reconciliation report bucket: recon-bucket

> **NOTE:** these instructions are targeted toward using the AWS Console since it allows deleting all versions of objects within a bucket. The AWS CLI may be used to delete non-versioned buckets however will not work with versioned buckets such as the kops state store.  For more information, see the AWS Documentation [Deleting or Emptying a Bucket](https://docs.aws.amazon.com/AmazonS3/latest/dev/delete-or-empty-bucket.html)

#### Amazon Aurora Databases

To delete all the data stored in Aurora for this tenant you will need to delete the following databases:

* Warehouse
* Reporting
* TODO: User focused database

When deleting a DB instance, you can choose whether to create a final snapshot of the DB instance.  If you want to be able to restore the DB instance at a later time, you should create a final snapshot.

> **NOTE:** When deleting a DB instance, all automated backups are deleted, while earlier manual snapshots are not deleted.

##### AWS Management Console

1. Sign in to the AWS Management Console and open the Amazon RDS console at [https://console.aws.amazon.com/rds/](https://console.aws.amazon.com/rds/).
2. In the navigation pane, choose **Instances**, and then select the DB instance that you want to delete.
3. Choose **Instance actions**, and then choose **Delete**.
4. For **Create final Snapshot?**, choose **Yes** or **No**.
5. If you chose yes in the previous step, for **Final snapshot name** type the name of your final DB snapshot.
6. Choose **Delete**.

##### AWS CLI

To delete a DB instance by using the AWS CLI, call the [delete-db-instance](http://docs.aws.amazon.com/cli/latest/reference/rds/delete-db-instance.html) command with the following parameters:

* --db-instance-identifier
* --final-db-snapshot-identifier or --skip-final-snapshot

Example With a Final Snapshot

`aws rds delete-db-instance --db-instance-identifier mydbinstance --final-db-snapshot-identifier mydbinstancefinalsnapshot`

Example Without a Final Snapshot

`aws rds delete-db-instance --db-instance-identifier mydbinstance  --skip-final-snapshot`

For more information see the AWS Documentation [Deleting a DB Instance](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/USER_DeleteInstance.html)

**Delete the following DB instances:**

* [ ] Warehouse: rdw-opus-warehouse
* [ ] Reporting: rdw-opus-reporting
* [ ] TODO: User focused database

TODO: talk about deleting manually created snapshots since deleting the DB doesn't clean those up

#### Amazon Redshift

To delete all the data stored in Redshift for this tenant you will need to delete the single Redshift cluster:

* RDW

When deleting a Redshift cluster, you can choose whether to create a final snapshot.  If you plan to provision a new cluster with the same data and configuration as this one, you will need to create the final snapshot. 

> **NOTE:** When deleting a Redshift cluster, all automated backups are deleted, while earlier manual snapshots are not deleted.

##### AWS Management Console

1. Sign in to the AWS Management Console and open the Amazon Redshift console at [https://console.aws.amazon.com/redshift/](https://console.aws.amazon.com/redshift/).
2. In the navigation pane, choose **Clusters**, and then choose the cluster that you want to delete.
3. On the **Configuration** tab of the cluster details page, choose **Cluster**, and then choose **Delete**.
4. In the** Delete Cluster** dialog box, do one of the following:
	* In **Create snapshot**, choose **Yes** to delete the cluster and take a final snapshot. In **Snapshot name**, type a name for the final snapshot, and then choose **Delete**.
	* In **Create snapshot**, choose **No** to delete the cluster without taking a final snapshot, and then choose **Delete**.

> After you initiate the delete of the cluster, it can take several minutes for the cluster to be deleted. You can monitor the status in the cluster list.

For more information see the AWS Documentation [Deleting a Cluster](https://docs.aws.amazon.com/redshift/latest/mgmt/managing-clusters-console.html#delete-cluster)

**Delete the following clusters:**

* [ ] RDW: rdw

TODO: talk about deleting manually created snapshots since deleting the cluster doesn't clean those up

#### ElastiCache Redis Cluster

To delete all the data stored in Redis for this tenant you will need to delete the single Redis cluster.

##### AWS Management Console

1. Sign in to the AWS Management Console and open the Amazon ElastiCache console at [https://console.aws.amazon.com/elasticache/](https://console.aws.amazon.com/elasticache/).
2. In the ElastiCache console dashboard, select the Redis engine.
3. Select the cluster's name from the list of clusters.
4. Select the **Actions** button and then select **Delete** from the list of actions.
5. In the Delete Cluster confirmation screen, specify that a final snapshot should not be taken and choose **Delete** to delete the cluster.

##### AWS CLI

The following code deletes the cache cluster *rdw-opus-redis.aws-randomization*.

`aws elasticache delete-cache-cluster --cache-cluster-id rdw-opus-redis.aws-randomization`

**Delete the following cluster:**

* [ ] RDW: rdw-opus-redis.[aws-randomization]

#### EC2 Instances

TODO: 

* discuss deleting jump server
* include details that AWS wipes the EBS volumes before making it available for use.  
* no need to use a tool like shred since there shouldn't be sensitive data on this server
	* I'm assuming the same goes for the k8s EC2 instances

### Deleting AWS services

#### Route 53

To delete all the DNS data from Route 53 for this tenant you will need to delete the following entries:

* Import route
* Reporting route
* Hosted Zone (optionally)

##### AWS Management Console

1. Sign in to the AWS Management Console and open the Route 53 console at https://console.aws.amazon.com/route53/.
2. On the Hosted Zones page, double-click the row for the hosted zone that contains records that you want to delete.
3. In the list of records, select the record that you want to delete.
4. Click **Delete Record Set**.
5. Click **OK** to confirm.

For more information see the AWS Documentation [Deleting Records](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/resource-record-sets-deleting.html).

**Delete the following CNAMEs:**

* [ ] Import Route: import.sbac.org
* [ ] Reporting Route: reporting.sbac.org

###### Hosted Zone

If the hosted zone was only being used for this particular tenant, you can delete the hosted zone as follows.

1. On the **Hosted Zones** page, choose the row for the hosted zone that you want to delete.
2. Choose **Delete Hosted Zone**.
3. Choose **OK** to confirm.


For more information see the AWS Documentation [Deleting a Public Hosted Zone](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/DeleteHostedZone.html).

> **NOTE:** You won't be able to delete the hosted zone if it contains anything other than an NS and an SOA record.

#### Security Groups

TODO: https://docs.aws.amazon.com/AmazonVPC/latest/UserGuide/VPC_SecurityGroups.html#DeleteSecurityGroup

* Aurora security group
* DB subnet group (where does this go)
* Redis security group (rdw-opus-redis)

#### IAM Groups/Roles/Users

There are groups, roles and users that were created during deployment that must now be removed.

##### AWS CLI

###### Users
The following code deletes a user named `MyTestUser`:

```aws iam delete-user --user-name MyTestUser ```

**Delete the following users:**

* [ ] RDW archive: rdw-opus
	```aws iam delete-user --user-name rdw-opus```

###### Roles
The following code deletes a role named `MyTestRole`:

```aws iam delete-role --role-name MyTestGroup```

**Delete the following roles:**

* [ ] RDW archive: rdw-opus-archive
	```aws iam delete-role --role-name rdsRdwOpusArchiveRole```

* [ ] Redshift archive: redshiftRdwOpusArchiveAccess
	```aws iam delete-role --role-name redshiftRdwOpusArchiveAccess```

###### Groups
The following code deletes a group named `MyTestGroup`:

```aws iam delete-group --group-name MyTestGroup```

**Delete the following groups:**

* [ ] kops group: kops
	```aws iam delete-group --group-name kops```

* [ ] RDW archive: rdw-opus-archive
	```aws iam delete-group --group-name rdw-opus-archive```


#### VPC

##### AWS Management Console

1. Open the Amazon VPC console at https://console.aws.amazon.com/vpc/.
2. In the navigation pane, choose **Your VPCs** and select your VPC.
3. Choose **Actions**, **Delete Endpoint**.
4. In the confirmation screen, choose** Yes, Delete.**

##### AWS CLI

You will need the VPC identifier to delete the VPC via the CLI.  Refer back to your reference doucmentation to retrieve the VPC ID or use the AWS Management Console.

The command to delete a specific VPC:

```aws ec2 delete-vpc --vpc-id vpc-3b779dd0```

For more information see the AWS Documentation [delete-vpc](https://docs.aws.amazon.com/cli/latest/reference/ec2/delete-vpc.html).

**TODO: deployment instructions say to create an ElasticIP for the VPC.  include details on deleting that EIP if needed.**