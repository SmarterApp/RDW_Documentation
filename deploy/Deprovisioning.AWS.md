## Deprovisioning

**Intended Audience**: This document provides instructions for permanently deleting the [Reporting Data Warehouse](../README.md) (RDW) applications deployed in an AWS environment. Access to the production environment is required. Operations will find this useful when deprovisioning a production environment.

> Although these are generic instructions, having an example makes it easier to document and understand. Throughout this document we'll use `opus` as the sample environment name; this might correspond to staging or production in the real world. We'll use `sbac.org` as the domain/organization. These are the same examples provided on the [Deployment Checklist](Deployment.AWS.md).

> **NOTE:** decommissioning an entire deployment is a significant IT task. These instructions should be used as a guide, and each and every step should be fully understood and adapted to a specific environment. DO NOT just type in all the commands and hope for the best.

This document assumes the AWS deployment instructions were used. Deprovisioning a tenant generally reverses the steps performed during deployment.

* [Quiesce the system](#quiesce)
* [Delete data and data stores](#delete-data)
* [Delete or reconfigure shared SBAC services](#shared-services)
* [Delete the cluster](#delete-cluster)
* [Clean up AWS resources](#cleanup-aws)

One advantage of using AWS is that they take care of wiping released resources. There is no need to use tools like shred to securely wipe data.


<a name="quiesce"></a>
### Quiesce the system

1. [ ] It is beyond the scope of this document to describe providing a static notification page or otherwise notifying the users. However, the public-facing services should be un-published. And preparations should be made for all the applications being taken down:
    * Notify users
    * Notify 3rd party data feeds
    * Remove (external) DNS routing
    * Remove application monitoring
1. [ ] Modify all the deployments to have zero replicas. This is the easiest way of shutting down all the applications. From the ops server:
```bash
for d in `kubectl get deployment -o name`; do kubectl scale $d --replicas=0; done
```

<a name="delete-data"></a>
### Delete data and data stores

#### Amazon S3

To delete all the data stored in S3 for this tenant you will need to delete the following buckets:
* kops state store bucket
* Archive bucket

These instructions use both the AWS console and AWS S3 command-line. This is because the command-line cannot be used to delete all versions of objects within a versioned bucket, and the console cannot be used to delete very large buckets. For more information see the AWS Documentation [How Do I Delete an S3 Bucket?](https://docs.aws.amazon.com/AmazonS3/latest/user-guide/delete-bucket.html), [Deleting or Emptying a Bucket](https://docs.aws.amazon.com/AmazonS3/latest/dev/delete-or-empty-bucket.html).

1. [ ] Delete the archive bucket using the command line (because this bucket is very large)
    ```bash
    aws s3 rb s3://rdw-opus-archive --force
    ```

#### Redis

To delete all the cached web session data for this tenant you will need to delete the Redis cluster.

To use the console, sign in to AWS and navigate to the Elasticache Management Console. For command-line the [aws redis reference](https://docs.aws.amazon.com/cli/latest/reference/rds/index.html) documentation may be useful.

There are many AWS resources associated with a Redis cluster including parameter groups, security groups, subnet groups, etc. It is a good idea to capture information about these resources before deleting the cluster.

> **Terminology Note:** The AWS Redis console and command-line use different terminology. In the console a "Cluster" contains multiple "Nodes". In the CLI a "Replication Group" contains multiple "Clusters". Bit confusing.

1. [ ] Capture information about the cluster. Note there will typically be a single replication group with multiple clusters, e.g.
    ```bash
    aws elasticache describe-replication-groups --replication-group-id rdw-opus-redis
    aws elasticache describe-cache-clusters --cache-cluster-id rdw-opus-redis
    ```
    And/or you can see this information in the console by selecting the cluster.
1. [ ] Delete Cluster (Replication Group)
    * Use the console to delete the cluster, or use the command-line:
    ```bash
    aws elasticache delete-replication-group --replication-group-id rdw-opus-redis
    ```
1. [ ] Delete Parameter Groups
    * Use the console to delete the parameter groups, or use the command-line:
    ```bash
    aws elasticache delete-cache-parameter-group --cache-parameter-group-name rdw-opus-redis32
    ```
1. [ ] Delete Subnet Groups
    * Use the console to delete the subnet group, or use the command-line:
    ```bash
    aws elasticache delete-cache-subnet-group --cache-subnet-group-name rdw-opus-redis
    ```

#### Amazon Aurora

To delete all the data stored in Aurora for this tenant you will need to delete the following databases:
* Warehouse
* Reporting

When deleting a DB instance, you can choose whether to create a final snapshot of the DB instance.  If you want to be able to restore the DB instance at a later time, you should create a final snapshot (or copy an existing snapshot). Alternatively, if you are responsible for completing purging all tenant data, you will need to delete any and all manual snapshots.

> **NOTE:** When deleting a DB instance, all automated backups are deleted, while manual snapshots are not deleted.

For more information see the AWS Documentation [Deleting a DB Instance](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/USER_DeleteInstance.html).

To use the console, sign in to AWS and navigate to the RDS Management Console. For command-line the [aws rds reference](https://docs.aws.amazon.com/cli/latest/reference/rds/index.html) documentation may be useful.

There are many AWS resources associated with an Aurora cluster including parameter groups, security groups, subnet groups, etc. It is a good idea to capture information about these resources before deleting the cluster.

The procedure for deleting a database cluster varies depending on how it was created. In general you will use the RDS Management Console or command-line to delete the cluster. For each of the clusters, `rdw-opus-warehouse` and `rdw-opus-reporting`:
1. [ ] Capture information about the cluster, e.g.
    ```bash
    aws rds describe-db-clusters --db-cluster-identifier rdw-opus-reporting-cluster
    aws rds describe-db-cluster-snapshots --db-cluster-identifier rdw-opus-reporting-cluster
    ```
    And/or you can see some of this information in the AWS RDS Management Console by selecting a cluster. The snapshots can be seen on the appropriate screen. Take notes or screenshots.
1. [ ] Delete Cluster
    * As necessary, use console to delete all instances, then delete the cluster. **OR** use the command-line (you may be able to delete cluster or you may have to delete instances separately):
    ```bash
    # delete cluster with final snapshot
    aws rds delete-db-cluster --db-cluster-identifier rdw-opus-reporting-cluster --final-db-snapshot-identifier rdw-opus-reporting
    # delete cluster without final snapshot
    aws rds delete-db-cluster --db-cluster-identifier rdw-opus-reporting-cluster --skip-final-snapshot

    # delete instance with final snapshot
    aws rds delete-db-instance --db-instance-identifier rdw-opus-reporting --final-db-snapshot-identifier rdw-opus-reporting
    # delete instance without final snapshot
    aws rds delete-db-instance --db-instance-identifier rdw-opus-reporting --skip-final-snapshot
    ```
1. [ ] Delete Parameter Groups
    * There will typically be two parameter groups to remove, a parameter group and a cluster parameter group, and they typically have the same name. These can be deleted from the console or the command-line:
    ```bash
    aws rds delete-db-parameter-group --db-parameter-group-name rdw-opus-reporting
    aws rds delete-db-cluster-parameter-group --db-cluster-parameter-group-name rdw-opus-reporting
    ```
1. [ ] Delete Subnet Groups
    * Delete any subnet group associated with the cluster. These can be deleted from the console or the command-line:
    ```bash
    aws rds delete-db-subnet-group --db-subnet-group-name rdw-opus
    ```
1. [ ] Delete Security Groups
    * Delete any security groups associated with the cluster. These can be deleted from the console or the command-line.
    ```bash
    aws rds delete-db-security-group --db-security-group-name rdw-launch-wizard-4
    ```
1. [ ] (Optional) Delete Snapshots
    * Delete any remaining (manual) snapshots as necessary.

#### Amazon Redshift

To delete all the data stored in Redshift for this tenant you will need to delete the Redshift cluster:
* rdw

> **NOTE:** If the redshift cluster is shared across environments or tenants (perhaps for cost-savings reasons) then, obviously, the cluster should not be deleted.

When deleting a Redshift cluster, you can choose whether to create a final snapshot.  If you plan to provision a new cluster with the same data and configuration as this one, you will need to create the final snapshot. Alternatively, if you are responsible for completing purging all tenant data, you will need to delete any and all manual snapshots.

> **NOTE:** When deleting a Redshift cluster, all automated backups are deleted, while earlier manual snapshots are not deleted.

To use the console, sign in to AWS and navigate to the Redshift Management Console. For command-line the [aws redshift reference](https://docs.aws.amazon.com/cli/latest/reference/redshift/index.html) documentation may be useful.

There are many AWS resources associated with a Redshift cluster including parameter groups, security groups, subnet groups, etc. It is a good idea to capture information about these resources before deleting the cluster.

1. [ ] Capture information about the cluster, e.g.
    ```bash
    aws redshift describe-clusters --cluster-identifier rdw
    ```
    And/or you can see some of this information in the AWS Redshift Management Console by selecting a cluster. The snapshots can be seen on the appropriate screen. Take notes or screenshots.
1. [ ] Delete Cluster
    * As necessary, use console to delete all instances, then delete the cluster. **OR** use the command-line (you may be able to delete cluster or you may have to delete instances separately):
    ```bash
    # delete cluster with final snapshot
    aws redshift delete-cluster --cluster-identifier rdw --final-cluster-snapshot-identifier rdw
    # delete cluster without final snapshot
    aws redshift delete-cluster --cluster-identifier rdw --skip-final-cluster-snapshot true
    ```
1. [ ] Delete Parameter Group
    * Delete the parameter group from the console or the command-line:
    ```bash
    aws redshift delete-cluster-parameter-group --parameter-group-name rdw-opus
    ```
1. [ ] Delete Subnet Groups
    * Delete any subnet group associated with the cluster. These can be deleted from the console or the command-line:
    ```bash
    aws redshift delete-cluster-subnet-group --cluster-subnet-group-name rdw-opus
    ```
1. [ ] Delete Security Groups
    * Delete any security groups associated with the cluster. These can be deleted from the console or the command-line.
1. [ ] (Optional) Delete Snapshots
    * Delete any remaining (manual) snapshots as necessary.

##### Delete Database in Shared Redshift Cluster

If the redshift cluster is shared then just remove the database and users for this RDW tenant. The details depend on how the sharing was set up; and example using `psql --host=rdw.[aws-randomization] --port=5439 --username=root --password --dbname=dev`:
```sql
ALTER DATABASE opus OWNER TO root;
DROP USER rdwopusreporting;
DROP USER rdwopusingest;
DROP DATABASE opus;
```


<a name="shared-services"></a>
### Delete or reconfigure shared SBAC services

RDW runs as part of the SBAC ecosystem which consists of other services that may have been modified to integrate RDW. If the shared services will be left running (e.g. to support other applications), then RDW-specific settings should be reverted. If other RDW tenants are using the shared services then only remove routes for the tenant being deprovisioned.

1. [ ] IRiS.
    * Remove any routes for the RDW tenant being deprovisioned.
1. [ ] Permissions.
    * If no other RDW tenants are using the system, remove RDW-specific components, roles and permissions.
        * RDW permissions may be removed:
            * GROUP_PII_READ
            * GROUP_READ
            * GROUP_WRITE
            * INDIVIDUAL_PII_READ
            * CUSTOM_AGGREGATE_READ
            * EMBARGO_READ
            * EMBARGO_WRITE
            * INSTRUCTIONAL_RESOURCE_WRITE
        * RDW roles may be removed:
            * ASMTDATALOAD
            * GROUP_ADMIN
            * PII
            * PII_GROUP
            * Instructional Resource Admin
            * Embargo Admin
            * Custom Aggregate Reporter
        * RDW component may be removed:
            * Reporting
    * Remove any routes for the RDW tenant being deprovisioned.
1. [ ] ART / SSO.
    * These systems are used to grant users access to applications other than RDW. If a user will continue to use other services, remove just the RDW roles for users. If a user doesn't have access to other services and was created just for RDW, that user should be removed from the system.
        * Remove end-users (teachers, administrators, etc.) or revoke RDW roles as appropriate.
        * Remove vendor credentials (e.g. ASMTDATALOAD user)
        * Remove support / test credentials or revoke RDW roles as appropriate.
    * Remove any routes for the RDW tenant being deprovisioned.
1. [ ] OpenAM
    * Remove the rdw-reporting-opus SAML Entity Provider
    * Remove the OAuth2 agent client id: rdw


<a name="delete-cluster"></a>
### Delete the cluster

Use kops to delete the cluster. From the ops system (aka jump-server aka bastion):
```bash
kops delete cluster \
  --state s3://kops-rdw-opus-state-store \
  --name opus.rdw.sbac.org \
  --yes
```

Although this will remove most resources created for the cluster there may be some things that may need additional cleanup:
* Route 53 entries
    * import.sbac.org
    * reporting.sbac.org
* If the hosted zone for sbac.org was created for just RDW and will no longer be used it should be deleted.


<a name="cleanup-aws"></a>
### Clean up AWS resources

The preceding steps clean up a lot of AWS resource but there are a few others.

1. [ ] Delete the kops state store bucket using the console (because this bucket should be versioned).
    * Sign into AWS and navigate to the S3 Management Console
    * Select the `kops-rdw-opus-state-store` bucket, click **Delete Bucket** and follow directions to confirm.

#### IAM Groups/Roles/Users

There are groups, roles and users that were created during deployment that must now be removed.

To use the console, sign in to AWS and navigate to the IAM Management Console. For command-line the [aws iam reference](https://docs.aws.amazon.com/cli/latest/reference/iam/index.html) documentation may be useful.

1. [ ] Delete users
    * Use the console to delete any RDW users, or use the command-line:
    ```bash
    aws iam delete-user --user-name rdw-opus
    ```
1. [ ] Delete groups
    * Use the console to delete any RDW groups, or use the command-line:
    ```bash
    aws iam delete-group --group-name rdw-opus-archive
    aws iam delete-group --group-name kops
    ```
1. [ ] Delete roles
    * Use the console to delete any RDW roles, or use the command-line:
    ```bash
    aws iam delete-role --role-name rdsRdwOpusArchiveRole
    aws iam delete-role --role-name redshiftRdwOpusArchiveAccess
    ```
    This is a good time to check that kops properly deleted the masters/nodes ec2 roles.
1. [ ] Delete policies
    * Use the console to delete any RDW policies, or use the command-line:
    ```bash
    aws iam delete-policy --policy-arn arn:aws:iam::479572410002:policy/AllowRdwOpusArchiveBucket
    ```

#### VPC

If a dedicated VPC was created for RDW then it should be deleted.

To use the console, sign in to AWS and navigate to the EC2 Management Console. For command-line the [aws ec2 reference](https://docs.aws.amazon.com/cli/latest/reference/ec2/index.html) documentation may be useful.

For more information see the AWS Documentation [delete-vpc](https://docs.aws.amazon.com/cli/latest/reference/ec2/delete-vpc.html).

1. [ ] Delete RDW VPC.
    * Use the console to delete the VPC, or use the command-line:
    ```bash
    # list vpcs to find id
    aws ec2 describe-vpcs

    # delete vpc
    aws ec2 delete-vpc --vpc-id vpc-3b779dd0
    ```
1. [ ] Delete EIP

#### Ops Server

If the system is being completely decommissioned the ops server (aka jump server aka bastion) should be deleted.

1. [ ] Delete ops server
    * Use the console to delete the ops server.

#### Repositories

Technically these aren't AWS resources but the configuration and deployment repositories may be deleted.
