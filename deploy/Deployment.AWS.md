## Deployment Checklist

**Intended Audience**: this document provides a detailed checklist for deploying the Reporting Data Warehouse (RDW) applications in an AWS environment. Operations and system administrators will find it useful.

> Although these are generic instructions, having an example makes it easier to document and understand. Throughout this document we'll use `opus` as the sample environment name; this might correspond to `staging` or `production` in the real world. We'll use `sbac.org` as the domain/organization. Any other ids, usernames or other system-generated values are products of the author's imagination. The reader is strongly encouraged to use their own consistent naming conventions. **Avoid putting real environment-specific details _especially secrets and sensitive information_ in this document.**

> **NOTE:** these instructions were originally written for a v1.0 installation. The v1.0 -> v1.1 upgrade instructions were then merged, but the consolidated instructions haven't been thoroughly tested for a clean v1.1 installation. 

### Table Of Contents
* [Reference](#reference)
* [Checklist](#checklist)
* [Updating Applications](#updating-applications)
* [Scalability](#performance-and-scalability)
* [Sizing Kubernetes Cluster](#sizing-kubernetes-cluster)

### Other Resources
* [Deprovisioning Instructions](Deprovisioning.AWS.md)

<a name="reference"></a>
### Summary / Reference
This section records all details that will facilitate configuration and maintenance of the system. It should be filled in as the deployment checklist is worked. The instructions include reminders to *Record values in the reference.*

* Deployment Repository: https://github.com/RDW_Deploy_Opus
* Configuration Repository: 
    * URL: https://github.com/RDW_Config_Opus
    * config user: rdw-opus/password
* AWS
    * AWS account: aws-sbac-rdw
    * Signin link: https://aws-sbac-rdw.signin.aws.amazon.com/console
    * Region: us-west-2
    * Hosted Zone: rdw.sbac.org
    * VPC: opus, vpc-3b779dd0, 10.1.0.0/16
        * subnet-3523237c, 10.1.32.0/19, us-west-2a, private
        * subnet-1aec42d9, 10.1.64.0/19, us-west-2b, private
        * subnet-82dac27d, 10.1.96.0/19, us-west-2c, private
* Ops system / jump server: rdw-opus-jump.sbac.org
* SSO 
    * host: sso.sbac.org
    * OpenAM oauth2 agent client id: rdw/password
* ART 
    * host: art.sbac.org
    * ingest user: rdw-ingest-opus@sbac.org/password
* Perms host: perm.sbac.org
* IRIS:
    * hosts: rdw-opus-iris-1, rdw-opus-iris-2
    * ELB: rdw-opus-iris (internal-rdw-ops-iris-[aws-randomization])
    * EFS: fs-[aws-randomization]
* kops state store bucket: s3://kops-rdw-opus-state-store
* kubernetes cluster name: opus.rdw.sbac.org
* Aurora warehouse:
    * name: rdw-opus-warehouse
    * root user: root/password
    * read-write: rdw-opus-warehouse-cluster.cluster-[aws-randomization]
    * read-only: rdw-opus-warehouse-cluster.cluster-ro-[aws-randomization]
    * ingest user: rdwingest/password
* Aurora reporting:
    * name: rdw-opus-reporting
    * root user: root/password
    * read-write: rdw-opus-reporting-cluster.cluster-[aws-randomization]
    * read-only: rdw-opus-reporting-cluster.cluster-ro-[aws-randomization]
    * reporting user: rdwreporting/password
* Redshift:
    * name: rdw
    * root user: root/password
    * end-point: rdw.[aws-randomization]
    * ingest user: rdwopusingest/password
    * reporting user: rdwopusreporting/password
    * role (to access S3): arn:aws:iam::[aws-acct]:role/rdw-redshift-access
* Redis: rdw-opus-redis.[aws-randomization]
* Archive S3 bucket: 
    * name: rdw-opus-archive
    * user: rdw-opus, accesskey/secretkey
* Reconciliation report S3 bucket:
    * name: recon-bucket
    * user: name, accesskey/secretkey
* Log collection:
    * GELF sink: 1.2.3.4:12201
    * Source name: opus-prod
* Import:
    * end-point: import.sbac.org
    * ELB id: [aws-randomization]
* Reporting:
    * end-point: reporting.sbac.org
    * ELB id: [aws-randomization]

<a name="checklist"></a>
### Checklist

* [ ] Deployment Repository. This is the source repository that will store your deployment documentation and the kubernetes specifications for the applications. This must be a private repository since it contains credentials and other sensitive information. 
    * Create repository with name like RDW_Deploy_Opus
    * Copy the files from this deploy folder to the repo and commit them.
    * Switch to working with the files in the repository, making changes specific to your environment.
    * *Record the repository URL in the reference.*
* [ ] Configuration Repository. This is the source repository that will store the application configuration files. This must be a private repository as it will contain some sensitive credentials (note that most credentials will be one-way hashed even in the private repository). 
    * Create repository with name like RDW_Config_Opus
    * Copy the files from the config folder to the repo and commit them.
    * Create a user that has read access to the repository.
    * *Record the repository URL and username/password in the reference.*
* [ ] Prepare AWS
    * [ ] Create an AWS account. All the RDW services will run in this account. 
        * *Record the account and signin link in the reference.*
    * [ ] Create a hosted zone in Route53 using console or CLI.
        ```bash
        aws route53 create-hosted-zone --name rdw.sbac.org --caller-reference 2014-04-01 --hosted-zone-config Comment="RDW"
        ```
        * *Record the zone in the reference.*
    * [ ] Create VPC for environment.
        * Allocate a new EIP address.
        * Wizard: VPC with Public and Private subnets.
        *  *Record the VPC info in the reference.*
        * Do **not** create any extra subnets; `kops` will do that
    * [ ] Create group for kops permissions using console or CLI.
        ```bash
        # from https://github.com/kubernetes/kops/blob/master/docs/aws.md
        aws iam create-group --group-name kops
        aws iam attach-group-policy --policy-arn arn:aws:iam::aws:policy/AmazonEC2FullAccess --group-name kops
        aws iam attach-group-policy --policy-arn arn:aws:iam::aws:policy/AmazonRoute53FullAccess --group-name kops
        aws iam attach-group-policy --policy-arn arn:aws:iam::aws:policy/AmazonS3FullAccess --group-name kops
        aws iam attach-group-policy --policy-arn arn:aws:iam::aws:policy/IAMFullAccess --group-name kops
        aws iam attach-group-policy --policy-arn arn:aws:iam::aws:policy/AmazonVPCFullAccess --group-name kops
        ```
    * [ ] Grant ops permissions to existing ops users.
        ```bash
        aws iam add-user-to-group --user-name <user> --group-name kops
        aws iam attach-user-policy --user-name <user> --policy-arn arn:aws:iam::aws:policy/AmazonElastiCacheFullAccess
        aws iam attach-user-policy --user-name <user> --policy-arn arn:aws:iam::aws:policy/AmazonElasticFileSystemFullAccess
        aws iam attach-user-policy --user-name <user> --policy-arn arn:aws:iam::aws:policy/AmazonRDSFullAccess
        aws iam attach-user-policy --user-name <user> --policy-arn arn:aws:iam::aws:policy/AWSMarketplaceManageSubscriptions
        ```
* [ ] Configure ops system. Working with k8s on a machine that has a GUI/browser is nicer than relying completely on command-line capabilities. However, that introduces the complexity of setting up multiple systems for different ops users, as well as exposing the kubernetes subnet. So the recommendation is to set up an EC2 instance with all the command-line tools.
    * [ ] Generate ssh key for accessing the ops system and for cluster (later on). You can generate locally and upload to AWS Key Pairs, or generate via AWS and keep private key local, or ...
        ```bash
        ssh-keygen -t rsa -C "rdw-ops" -f id_ops
        ```
    * [ ] Create EC2 instance.
        * CentOS Linux 7, t2.micro
        * Network: select environment VPC, Subnet: public, Auto-assign Public IP: Enable 
        * Tag: Name=rdw-opus-jump, Security: Jump SSH
        * Key Pair: rdw-key
        * Verify connectivity (username = centos)
        * Install ssh credentials for ops users
    * [ ] Install kops and kubectl (requires sudo)
        ```bash
        # from https://kubernetes.io/docs/getting-started-guides/kops/
        sudo yum install -y wget
        wget https://github.com/kubernetes/kops/releases/download/1.7.0/kops-linux-amd64
        chmod +x kops-linux-amd64
        sudo mv kops-linux-amd64 /usr/local/bin/kops
        kops version
        ```
        ```bash
        # from https://kubernetes.io/docs/tasks/tools/install-kubectl/
        curl -LO https://storage.googleapis.com/kubernetes-release/release/v1.7.3/bin/linux/amd64/kubectl
        chmod +x ./kubectl
        sudo mv ./kubectl /usr/local/bin/kubectl
        kubectl version
        # each user will want to do this for auto-completion support - TODO: this doesn't work!  _get_comp_words_by_ref
        echo "source <(kubectl completion bash)" >> ~/.bashrc
        ```
    * [ ] Install MySQL client
        ```bash
        sudo yum install http://dev.mysql.com/get/mysql-community-release-el7-5.noarch.rpm
        # this may be an upgrade
        sudo yum -y install mysql
        # verify this is version 5.6
        mysql --version
        ```
    * [ ] Install `psql`, e.g. `sudo yum install postgresql`. Install just the client, not the server!
    * [ ] Install git (requires sudo)
        ```bash
         sudo yum -y install git
         git --version
         ```
    * [ ] Each user of the system will want to configure AWS CLI, kops, kubectl
        * `aws configure` - enter access key, secret key, region, default output format (json)
        * `aws ec2 describe-availability-zones --output table` to test configuration
        * `export KOPS_STATE_STORE=s3://kops-rdw-opus-state-store` - add to bash profile
        * `kops export kubecfg rdw-opus.sbac.org` - perhaps add to bash profile
    * [ ] (Optional) Install kubetail. This utility is pretty darn handy for tailing the logs of multiple pods at once.
    You can get it from the source (https://github.com/johanhaleby/kubetail) or just use the copy included here. Put it
    somewhere in your path and mark it executable, for example:
        ```bash
        cd ~/git/
        git clone https://github.com/johanhaleby/kubetail
        cp kubetail/kubetail /usr/local/bin/
        chmod +x /usr/local/bin/kubetail
        ```
* [ ] SBAC Application Prerequisites. These are non-RDW applications that are maintained as part of the SBAC ecosystem. They run independently of RDW and are not part of the Kubernetes cluster.
    * [ ] SSO. 
        * Create OpenAM OAuth2 client. The steps for this vary but here are the highlights:
            * Navigate to the OpenAM site and login with an administrative account.
            * Go to `Access Control`, selecting the appropriate realm, e.g. `sbac`
            * Go to `Agents`, `Oauth 2.0 Client`
            * Create a new client, noting name and password.
            * Typically, a group is used to manage settings:
                * Edit the newly created client, assign the `Group` to, for example, sbac_client_group, and then Save.
                * Continue editing, set the `Inheritance Settings` to include at least:
                    * `Default Scope(s)`
                    * `ID Token Signed Response Algorithm`
                    * `Scope(s)`
        * *Record the host and client info in the reference.*
    * [ ] ART. 
        * [ ] Create ingest user for the task service user. This is used by the task service to fetch organization information from ART and submit it to the import service. As such it should have `Administrator`, `State Coordinator` and `ASMTDATALOAD` roles at the state level for `CA`.
        * *Record ART host and username/password in the reference.*
    * [ ] Permission Service. Note that roles are tenant-specific. Examples here are what were created in production
    but roles (and their associated permissions) should be based more on the real-world user roles. 
        * *Record perm service host in the reference.*
        * Add/verify component 
            * `Reporting`
        * Add/verify roles: 
            * `GROUP_ADMIN` - District, GroupOfInstitutions, Institutions
            * `PII` - all but GroupOfStates
            * `PII_GROUP` - all but GroupOfStates 
            * `Instructional Resource Admin` - State, District, GroupOfInstitutions, Institution
            * `Embargo Admin` - State, District
            * `Custom Aggregate Reporter` - State, District, GroupOfInstitutions, Institution
        * Add/verify component permissions for `Reporting`
            * `GROUP_PII_READ`, `GROUP_READ`, `GROUP_WRITE`, `INDIVIDUAL_PII_READ`
            * `CUSTOM_AGGREGATE_READ`, `EMBARGO_READ`, `EMBARGO_WRITE`, `INSTRUCTIONAL_RESOURCE_WRITE`
        * Add/verify mappings of Reporting component permissions (rows) to roles (columns) 
            * `GROUP_PII_READ` - `PII`, `PII_GROUP`
            * `GROUP_READ` - `GROUP_ADMIN`, `PII`, `PII_GROUP`
            * `GROUP_WRITE` - `GROUP_ADMIN`
            * `INDIVIDUAL_PII_READ` - `PII`
            * `CUSTOM_AGGREGATE_READ` - `PII`, `Custom Aggregate Reporter`
            * `EMBARGO_READ` - `Embargo Admin`
            * `EMBARGO_WRITE` - `Embargo Admin`
            * `INSTRUCTIONAL_RESOURCE_WRITE` - `Instructional Resource Admin`
    * [ ] IRiS. 
        * Follow directions for setting up [IRiS](IRIS.AWS.md).
        * *Record IRiS hosts, IRiS ELB and IRiS EFS in the reference.*
* [ ] Create Kubernetes cluster.
    1. Create versioned S3 bucket for storing configuration using console or CLI. 
        ```bash
        aws s3 mb s3://kops-rdw-opus-state-store --region us-west-2
        aws s3api put-bucket-versioning --bucket kops-rdw-opus-state-store --versioning-configuration Status=Enabled
        ```
        * *Record `kops` bucket in the reference.*
    1. Create cluster configuration. Need to know and substitute region, dns-zone, VPC id and CIDR, ssh key, EC2 size (master: m4.large, nodes: m4.xlarge), kops state store bucket. And you'll need to come up with a cluster name.
        ```bash
        kops create cluster \
          --cloud=aws \
          --node-count=4 \
          --zones=us-west-2a,us-west-2b,us-west-2c \
          --master-zones=us-west-2a,us-west-2b,us-west-2c \
          --dns-zone=rdw.sbac.org \
          --vpc="vpc-3b779dd0" \
          --network-cidr="10.1.0.0/16" \
          --node-size=m4.xlarge \
          --master-size=m4.large \
          --ssh-public-key="~/.ssh/id_ops.pub" \
          --topology private --networking weave --bastion \
          --state s3://kops-rdw-opus-state-store \
          --name opus.rdw.sbac.org
        ```
        * *Record cluster name in the reference.*
    1. Edit cluster configuration to set the CIDR for the subnet to avoid conflict with existing subnets. You can see existing subnets in the console or with CLI `aws ec2 describe-subnets --filters Name=vpc-id,Values=vpc-3b779dd0`.
        ```bash
        kops edit cluster opus.rdw.sbac.org --state s3://kops-rdw-opus-state-store
        ```
        ```bash
        ...
          networkCIDR: 10.1.0.0/16
          networkID: vpc-3b779dd0
        ...
          subnets:
        ...
        ```
    1. Edit nodes instance group if desired (e.g. to set min/max nodes).
        ```bash
        kops edit ig nodes --name opus.rdw.sbac.org --state s3://kops-rdw-opus-state-store
        ```
    1. Update cluster to create
        ```bash
        kops update cluster opus.rdw.sbac.org --state s3://kops-rdw-opus-state-store --yes
        ```
    1. After a sufficient waiting period (it can take 10-20 minutes), validate cluster is set up
        ```bash
        kops validate cluster
        kubectl get nodes --show-labels
        ```
* [ ] Create Aurora instances. The system requires two aurora instances, `warehouse` and `reporting`. If the installation is small and cost is a concern, a single instance may be used.
    1. Create Parameter Groups
        * Family: aurora5.6, Type: DB Parameter Group, Name: rdw-opus-warehouse, Description: DB Params for RDW Opus Warehouse
        * Family: aurora5.6, Type: DB Cluster Parameter Group, Name: rdw-opus-warehouse, Description: DB Cluster Params for RDW Opus Warehouse
        * Family: aurora5.6, Type: DB Parameter Group, Name: rdw-opus-reporting, Description: DB Params for RDW Opus Reporting Data Mart
        * Family: aurora5.6, Type: DB Cluster Parameter Group, Name: rdw-opus-reporting, Description: DB Cluster Params for RDW Opus Reporting Data Mart
        * Set max_connections = 500 for both DB Parameter Groups.
        * Enable logging of slow queries in rdw-opus-reporting: `long_query_time = 1`, `slow_query_log = 1`
    1. Create MySQL/Aurora Security Group
        * Name: rdw-opus-aurora
        * Type: MYSQL/Aurora, Protocol: TCP, Port Range: 3306, Source: 10.1.0.0/16    
    1. Create DB Subnet Group
        * Name: opus private
        * VPC: vpc-3b779dd0
        * Subnets: three private ones from kops
    1. Create two Aurora instances
        * DB Instance Class: db.r3.large
        * Multi-AZ Deployment: Yes
        * DB Instance Identifier: rdw-opus-warehouse, rdw-opus-reporting
        * Note master username and password
        * VPC: select one for environment
        * Subnet Group: opus private
        * Publicly Accessible: No
        * Security Group: rdw-opus-aurora
        * DB Parameter Group: rdw-opus-warehouse, rdw-opus-reporting
        * DB Cluster Parameter Group: rdw-opus-warehouse, rdw-opus-reporting
        * Enable Encryption: Yes
        * *Record the root username/password in the reference.*
    1. Test connectivity. From Configuration Details, copy Endpoint. From ops instance use endpoint as host to connect:
        ```bash
        echo "show status" | mysql -h rdw-opus-warehouse.[aws-randomization] -u root -p
        echo "show status" | mysql -h rdw-opus-reporting.[aws-randomization] -u root -p
        ```
    1. Run initialization scripts
        ```bash
        git clone https://github.com/SmarterApp/RDW_Schema.git
        cd RDW_Schema
        ./gradlew -Pflyway.url="jdbc:mysql://rdw-opus-warehouse-cluster.[aws-randomization]:3306/" \
           -Pflyway.user=root -Pflyway.password=password migrateWarehouse migrateMigrate_olap
        ./gradlew -Pflyway.url="jdbc:mysql://rdw-opus-reporting-cluster.[aws-randomization]:3306/" \
           -Pflyway.user=root -Pflyway.password=password migrateReporting
        ```   
    1. Create Aurora/MySQL users: rdwingest for warehouse and migrate_olap, rdwingest and rdwreporting for reporting.
        1. Create warehouse/migrate_olap user. Using mysql, `mysql -h rdw-opus-warehouse-cluster.[aws-randomization] -u root -p`:
        ```sql
        create user 'rdwingest'@'%' identified by 'password';
        grant all privileges on warehouse.* to 'rdwingest'@'%';
        grant all privileges on migrate_olap.* to 'rdwingest'@'%';
        grant select on mysql.proc to 'rdwingest'@'%';
        grant LOAD FROM S3 ON *.* to 'rdwingest'@'%';
        grant SELECT INTO S3 ON *.* to 'rdwingest'@'%';
        ```
        1. Create reporting users. Using mysql, `mysql -h rdw-opus-reporting-cluster.[aws-randomization] -u root -p`:
        ```sql
        create user 'rdwingest'@'%' identified by 'password';
        create user 'rdwreporting'@'%' identified by 'password';
        grant all privileges on reporting.* to 'rdwingest'@'%';
        grant SELECT INTO S3 ON *.* to 'rdwingest'@'%';
        grant all privileges on reporting.* to 'rdwreporting'@'%';
        grant SELECT INTO S3 ON *.* to 'rdwreporting'@'%';
        ```
        * *Record the username/passwords in the reference.*
* [ ] Create Redshift cluster. Redshift is expensive. Decide on cluster size and whether to share a single cluster between all environments.
NOTE: the security and routing for Redshift can be tricky, especially if the cluster is shared between environments. These instructions are intentionally vague; the ops group should understand the requirements and set things up to match their system policies.
    1. Create a Cluster Parameter Group, e.g. rdw-opus-redshift10
        * Enable Short Query Acceleration, 5 seconds
        * Set WLM Conurrency = 12
    1. Create Cluster Subnet Group in opus VPC, rdw-opus is a good name, add multiple zones.
    1. Create Security Group
        * rdw-redshift
        * opus VPC
        * open for redshift traffic from nodes (use security group or CIDRs) and jump server
    1. Create cluster
        * name like rdw-opus
        * 2 nodes, dc2.large 
        * opus VPC, use cluster subnet group, security group, etc. that were just created
    1. *Record the end-point URLs, root username/password in the reference.*
    1. Check the connection to Redshift. From ops server:
        ```bash
        psql --host=rdw-opus.[aws-randomization] --port=5439 --username=root --password --dbname=dev
        ```
        * (Optional) Configure psql to automatically connect. One way to do this ...
            * Create `~/.pg_service.conf` with something like this:
                 ```ini
                [rdw]
                host=rdw-opus.[aws-randomization]
                port=5439
                dbname=dev
                user=root
                password=password
                ```
             NOTE: there is a password in there so do something like `chmod 0600 ~/.pg_service.conf` to protect it.
             * Add to `.bashrc` (and source it)
                ```bash
                export PGSERVICE=rdw
    1. Create database and users. The users shown here have both role (ingest/reporting) and environment in their name because this is an example for when the Redshift instance is used by all environments. Obviously, if this is for a single environment these could be rdwingest/rdwreporting. Using psql, `psql --host=rdw.[aws-randomization] --port=5439 --username=root --password --dbname=dev`:
        ```sql
        CREATE USER rdwopusingest PASSWORD 'password';
        CREATE USER rdwopusreporting PASSWORD 'password';
        CREATE DATABASE opus;
        -- TODO - making rdwopusingest the OWNER should be unnecessary once vacuum/analyze handling is cleaned up
        ALTER DATABASE opus OWNER TO rdwopusingest;
        \connect opus
        CREATE SCHEMA reporting;
        GRANT ALL ON SCHEMA reporting to rdwopusingest;
        GRANT ALL ON ALL TABLES IN SCHEMA reporting to rdwopusingest;
        GRANT ALL ON SCHEMA reporting to rdwopusreporting;
        GRANT ALL ON ALL TABLES IN SCHEMA reporting to rdwopusreporting;
        ALTER USER rdwopusingest SET SEARCH_PATH TO reporting;
        ALTER USER rdwopusreporting SET SEARCH_PATH TO reporting;
        ```
        * *Record the user/passwords in the reference.*
    1. Run initialization scripts
        ```bash
        git clone https://github.com/SmarterApp/RDW_Schema.git
        cd RDW_Schema
        ./gradlew -Predshift_url=jdbc:redshift://rdw-opus.[aws-randomization]:5439/opus \
           -Predshift_user=root -Predshift_password=password migrateReporting_olap
        ``` 
    1. Verify Redshift user search path by using their credentials to connect and list tables (or something similar).
    For rdwopusingest also verify they can vacuum the fact table. For example,
        ```bash
        psql --host=rdw-opus.[aws-randomization] --port=5439 --username=rdwopusingest --password --dbname=opus
        opus=> \dt
                              List of relations
          schema   |               name               | type  | owner 
        -----------+----------------------------------+-------+-------
         reporting | administration_condition         | table | root
         reporting | asmt                             | table | root
        ...
        opus=> VACUUM REINDEX exam;
        VACUUM
        opus=> VACUUM;
        VACUUM
        opus=> ANALYZE;
        ANALYZE
        opus=> \q
        ```
* [ ] Create and configure archive S3 bucket
    1. Create unversioned S3 bucket for archive and a policy for accessing it using console or CLI.
        ```bash
        aws s3 mb s3://rdw-opus-archive --region us-west-2
        aws iam create-policy --policy-name AllowRdwOpusArchiveBucket --description "Allow any S3 actions in rdw-opus-archive" --policy-document file://AllowRdwOpusArchiveBucket.json
        ```
    1. Create a role to allow RDS access to the S3 bucket using console or CLI.
        ```bash
        aws iam create-role --role-name rdsRdwOpusArchiveRole --description "RDS access to rdw-opus-archive" --assume-role-policy-document file://RdsRoleTrustPolicy.json
        aws iam attach-role-policy --role-name rdsRdwOpusArchiveRole --policy-arn arn:aws:iam::478575410002:policy/AllowRdwOpusArchiveBucket
        ```
    1. Create IAM group and user: rdw-opus. 
        ```bash
        aws iam create-group --group-name rdw-opus-archive
        aws iam attach-group-policy --group-name rdw-opus-archive --policy-arn arn:aws:iam::479572410002:policy/AllowRdwOpusArchiveBucket 
        aws iam create-user --user-name rdw-opus
        aws iam add-user-to-group --user-name rdw-opus --group-name rdw-opus-archive
        aws iam create-access-key --user-name rdw-opus
        ```
        * *Record username and access/secret key in the reference.*
    1. Associate the RDW/S3 role with the warehouse DB Cluster to allow access to the S3 bucket using console or CLI. (http://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/Aurora.Authorizing.AWSServices.AddRoleToDBCluster.html) 
        ```bash
        aws rds add-role-to-db-cluster --db-cluster-identifier rdw-opus-warehouse-cluster --role-arn arn:aws:iam::479572410002:role/rdsRdwOpusArchiveRole
        aws rds modify-db-cluster-parameter-group --db-cluster-parameter-group-name rdw-opus-warehouse --parameters ParameterName=aws_default_s3_role,ParameterValue=arn:aws:iam::479572410002:role/rdsRdwOpusArchiveRole,ApplyMethod=immediate
        ```
    1. Create a role to allow Redshift access to the S3 bucket using console or CLI. **Note:** if the cluster is being used in multiple environments, there will be multiple policies to add to the role.   
        ```bash
        aws iam create-role --role-name redshiftRdwOpusArchiveAccess --description "Redshift access to rdw-opus-archive" --assume-role-policy-document file://RedshiftRoleTrustPolicy.json
        aws iam attach-role-policy --role-name redshiftRdwOpusArchiveAccess --policy-arn arn:aws:iam::478575410002:policy/AllowRdwOpusArchiveBucket
        ```
    1. Associate that role with the Redshift cluster. Use the console (Manage IAM roles) or CLI (TBD)  
* [ ] Test connectivity between S3, Aurora and Redshift. The data migration uses S3 to stage data between the warehouse
and the data marts. This is a good time to verify that the required connectivity and permissions are in place for that.
    * Export test file from Aurora. Connect to the warehouse using the ingest user and export a table to a test file.
        ```bash
        mysql -h rdw-opus-warehouse-cluster.[aws-randomization] --user=rdwingest --password=password warehouse
        mysql> SELECT id, code FROM completeness INTO OUTFILE S3 's3://rdw-opus-archive/completeness' FIELDS TERMINATED BY ',' OPTIONALLY ENCLOSED BY '"' LINES TERMINATED BY '\n';
        Query OK, 2 rows affected (0.13 sec)
        ```
    * Import test file to Redshift. Connect to redshift using the ingest user and import the test file into the table.
        ```bash
        psql --host=rdw-opus.[aws-randomization] --port=5439 --username=rdwopusingest --password --dbname=opus
        opus=> COPY reporting.staging_completeness (id, code) FROM 's3://rdw-opus-archive/completeness.part_00000' CREDENTIALS 'aws_iam_role=arn:aws:iam::[aws-randomization]' FORMAT AS CSV DELIMITER ',' COMPUPDATE OFF;
        INFO:  Load into table 'staging_completeness' completed, 2 record(s) loaded successfully.
        opus=> select * from staging_completeness;
         id |   code   
        ----+----------
          1 | Partial
          2 | Complete
        (2 rows)
        opus=> delete from staging_completeness;
        DELETE 2
        opus=> \q
        ```
* [ ] Create redis cluster. This is used by the webapps to cache/share session information.
    1. Create a security group that grants redis access to the master/nodes security groups.
        * Name: rdw-opus-redis
        * Description: RDW Opus redis to kubernetes cluster
        * VPC: vpc-3b779dd0
        * Inbound: Custom TCP, 6379, masters.rdw-opus and nodes.rdw-opus
    1. Create Redis Parameter Group rdw-opus-redis32
        * Based on default-redis3.2
        * Set `notify-keyspace-events = "eA"`
    1. Create Redis cache
        * Redis, version 3.2.4
        * Name: rdw-opus-redis
        * Parameter group: rdw-opus-redis32
        * Node type: cache.r3.2xlarge
        * Number of replicas: 2
        * Subnet group: create new in same VPC/subnets as cluster, call it rdw-opus-redis
        * Security Group: rdw-opus-redis
* [ ] Test access to the reconciliation report bucket. This is the landing place for reconciliation reports and is owned by the vendor submitting test results. They should have provided the bucket name and access/secret keys. This step is optional but is worthwhile since it will eliminate certain possibilities when troubleshooting. *Obviously, if another mechanism is being used to deliver reconciliation reports or if the report is not being used modify this step appropriately.*
    1. Create an AWS CLI credential profile on the jump server by adding the following to `~/.aws/credentials`:
        ```properties
        [reconciliation]
        aws_access_key_id = access-key
        aws_secret_access_key = secret-key
        ```  
    1. Test basic CLI access to the bucket:
        ```bash
        aws --profile reconciliation s3 ls s3://recon-bucket/REPORT/
        aws --profile reconciliation s3 cp opustest.txt s3://recon-bucket/REPORT/opustest.txt --sse AES256
        aws --profile reconciliation s3 rm s3://recon-bucket/REPORT/opustest.txt
        ```
    1. *Record reconciliation details in the reference.*
* [ ] Install Kubernetes utilities.
    1. Install Heapset, InfluxDB, Grafana.
        ```bash
        git clone https://github.com/kubernetes/heapster.git
        cd heapster
        vi deploy/kube-config/influxdb/grafana.yaml
        ...
            - name: GF_SERVER_ROOT_URL
              # If you're only using the API Server proxy, set this value instead:
              value: /api/v1/proxy/namespaces/kube-system/services/monitoring-grafana/
              # value: /
        kubectl create -f deploy/kube-config/influxdb
        ```
* [ ] Install and configure supporting services (autoscaler, logging daemonset, rabbit, wkthmltopdf, configuration service)
    1. Edit cluster-autoscaler.yml to verify --nodes option matches cluster configuration
    1. Edit fluentd-gelf.yml to set the GELF_HOST and the daemonset name.
        * Set GELF_HOST to the accessible IP address.
        * Set `metadata.name` to a name that is meaningful when filtering in Graylog. This value will appear as the "source" in Graylog messages.
    1. Edit configuration-service.yml to set git repo and credentials, and encrypt key
    1. Create the services:
    ```bash
    git clone https://github.com/RDW_Deploy_Opus.git
    cd RDW_Deploy_Opus
    kubectl create -f cluster-autoscaler.yml
    kubectl create -f rabbit-service.yml
    kubectl create -f wkhtmltopdf-service.yml
    kubectl create -f configuration-service.yml
    kubectl create -f iris-efs-volume.yml
    ```
* [ ] Install and configure backend RDW services. This step glosses over a **lot** of configuration details, highlighting just a few high-level steps. Read the architecture and runbook documents, and the annotated configuration files to fully understand all the configuration options.
    1. Generate encrypted passwords/secrets and set in configuration files. Need configuration service port-forward or exec into pod and install curl.
        ```bash
        kubectl exec -it configuration-deployment-xxx -- curl -X POST --data-urlencode "my+secret" http://localhost:8888/encrypt
        ```
    1. Set datasource urls, username, passwords in configuration files.
    1. Set S3 bucket, region, access key, secret key in configuration files.
    1. Set reconciliation S3 bucket in rdw-ingest-task-service.yml.
    1. Set schedules for migrate (reporting/olap), task service tasks, stale-reports cleanup, cache flushing.
    1. Configure load balancer certificate for import service.
        * Create cert for import.sbac.org (or wherever you'll be putting this service)
        * Modify import-service.yml and set ssl-cert to ARN of newly created cert.
    1. Create services.
        ```bash
        # ingest services
        kubectl apply -f exam-processor-service.yml
        kubectl apply -f group-processor-service.yml
        kubectl apply -f import-service.yml
        kubectl apply -f package-processor-service.yml
        kubectl apply -f migrate-olap-service.yml
        kubectl apply -f migrate-reporting-service.yml
        kubectl apply -f task-service.yml
        # reporting services
        kubectl apply -f admin-service.yml
        kubectl apply -f aggregate-service.yml
        kubectl apply -f reporting-service.yml
        kubectl apply -f report-processor-service.yml
        ```
    1. Add route for import service.
        * Get ELB id by describing service `kubectl describe service import-service`.
        * Verify ELB listeners and security group inbound rules are set to allow HTTP and HTTPS access.  
        * Add Route53 CNAME record for `import.sbac.org` to the ELB for the service.
        * *Record import service ELB info in the reference.*
* [ ] Install RDW webapp service.
    1. Create reporting keystore. 
        * `keytool -genkeypair -alias rdw-reporting-sp -keypass password -keystore rdw-reporting.jks`
        * Keystore password: password
    1. Add SSO IDP's public key to keystore. This may vary depending on intermediate key chains, etc. 
        * Capture certificate: `openssl s_client -connect sso.sbac.org:443 > sso.crt`
        * Edit file and delete everything that isn't between (and including) BEGIN/END lines 
        * If necessary, concatenate with any required intermediate certificates.
        * Import cert into keystore: `keytool -import -trustcacerts -alias sso -file ./sso.crt -keystore ./rdw-reporting.jks`
        (type Y to trust this certificate)
        * Test `keytool -v -list -keystore rdw-reporting.jks | grep -i alias`
        * Add rdw-reporting.jks file to RDW_Config_Opus repo
    1. Configure related properties in reporting-webapp.yml.
    1. Configure load balancer certificates.
        * Create cert for reporting.sbac.org (or whever you'll be putting these services)
        * Modify reporting-webapp.yml and set ssl-cert to ARN of newly created cert.
    1. Create reporting webapp service.
        ```bash
        kubectl create -f reporting-webapp.yml
        ```
    1. Add route for reporting webapp service. 
        * Get ELB id by describing service `kubectl describe service reporting-webapp`.
        * Verify ELB listeners and security group inbound rules are set to allow HTTP and HTTPS access.  
        * Add Route53 CNAME record for `reporting.sbac.org` to the ELB for the service.
        * *Record service ELB info in the reference.*
    1. Register reporting service with OpenAM as service providers (SPs).
        * Ensure configuration properties set properly; saml.sp-entity-id and saml.load-balance.*
        * Generate rdw-opus-reporting.sp.xml by using curl to `GET ./saml/metadata`
            * verify SP id and entity id is rdw-opus-reporting 
            * verify all service hosts set to `https://reporting.sbac.org/...`
        * Register SPs in ForgeRock OpenAM 
            * Navigate to Federation, Entity Providers
            * Import Entity, upload the sp.xml file
            * Add newly created entity to circle-of-trust
* [ ] Load data.
    1. Get access token. You need both the OAuth2 client id/secret and credentials for an ART user with ASMTDATALOAD.
    `jq` is a handy utility for parsing json payloads.
        ```bash
        sudo yum install -y jq
        export ACCESS_TOKEN=`curl -s -X POST --data 'grant_type=password&username=rdw-ingest-opus@sbac.org&password=password&client_id=rdw&client_secret=password' 'https://sso.sbac.org/auth/oauth2/access_token?realm=/sbac' | jq -r '.access_token'`
        ```
    1. Load accommodations. Need an appropriate accommodations.xml file. There is a copy included here but the source of truth is: https://github.com/SmarterApp/AccessibilityAccommodationConfigurations/blob/master/AccessibilityConfig.xml.
        ```bash
        curl -X POST --header "Authorization: Bearer ${ACCESS_TOKEN}" -F file=@accommodations.xml https://import.sbac.org/accommodations/imports
        ```
    1. Load subjects.  The subject definitions register the assessment subjects available in the system.  Each subject definition contains the associated performance levels, difficulty cutpoints, targets, claims, etc that define an assessment subject.  By default the system is pre-registered with Math and ELA as available subjects.  However, the associated definition files must still be imported to define display text.
        ```bash
        curl -X POST --header "Authorization: Bearer ${ACCESS_TOKEN}" -F file=@Math_subject.xml https://import.sbac.org/subjects/imports
        curl -X POST --header "Authorization: Bearer ${ACCESS_TOKEN}" -F file=@ELA_subject.xml https://import.sbac.org/subjects/imports
        ```
    1. Load assessment packages. These are created by the tabulator; they are not provided here because they are proprietary in nature. For each file:
        ```bash
        curl -X POST --header "Authorization: Bearer ${ACCESS_TOKEN}" -F file=@2017-2018.csv https://import.sbac.org/packages/imports
        ```
    1. Load organizations. Wait for the daily sync to occur or, to trigger the update-organization task to run immediately, use port-forwarding or shell into the task-service pod to POST to the task end-point:
        ```bash
        kubectl exec task-service-[k8s-randomization] -it /bin/sh
        apk update
        apk add curl
        curl -X POST http://localhost:8008/updateOrganizations
        ```
    1. There are other data that may need to be applied directly to the database. These require site-specific scripts so are not included here. Examples include:
        * Adjust assessment labels. 
        * Load instructional resources.
* [ ] Miscellaneous tasks.
    * [ ] Set up scheduled task to do Redshift `ANALYZE`. Please refer to [Analyze & Vacuum](../docs/Performance.md#redshift-analyze-and-vacuum).
    * [ ] Set up centralized log collection. Please refer to [Log Collection](../docs/Monitoring.md#log-collection).
    * [ ] Set up application monitoring.
        
### Updating Applications
When software updates are available there may be a number of steps involved in deploying them.
1. Determine which images are being updated. This may range from a single application getting a hot fix, to all the applications being updated to a new major release.
1. There may be configuration changes required. The CHANGELOG should give an indication of this, and the source history for the project can be reviewed. Configuration changes may be made just before new software is deployed -- they won't affect the running software and will be loaded as the new software comes online.
1. There may be schema changes required. The CHANGELOG should give an indication of this, and the source history for the project can be reviewed. Schema changes can take a while and the coordination can be tricky. All but the most trivial schema changes require downtime of the system.
1. Use the rollout capability of Kubernetes to deploy changes.

##### Configuration Change
Sometimes just a simple change to configuration is required. After making the change this trick can be used to force Kubernetes to "bounce" the applications:
1. Make the configuration changes and commit them.
2. Trick Kubernetes into thinking the deployment spec has changed. Using the webapp as an example,
    ```bash
    export DATE_PATCH="{\"spec\":{\"template\":{\"metadata\":{\"labels\":{\"date\":\"`date +'%s'`\"}}}}}"
    kubectl patch deployment reporting-webapp-deployment -p "$DATE_PATCH"
    ```

##### Software Patch
The simplest update is a backward compatible patch, usually to deploy an urgent bug fix. The major and minor versions don't change, there is no significant schema change, and only minor configuration changes expected. Typically only one or a few applications are being updated. 
1. Update the deployment specs with the new image value.
1. Make any corresponding changes to the configuration files.
1. Either apply the modified specs OR directly update the images. This will be repeated for all affected applications but a single example:
    ```bash
    kubectl apply -f import-service.yml
    OR
    kubectl set image deployment import-deployment api="smarterbalanced/rdw-ingest-import-service:1.2.3-RELEASE"
    ```
    
##### Major Version Upgrades
Major version updates often require additional steps. Please refer to specific documentation:
* [Upgrade v1.1](Upgrade.v1_1.AWS.md)
* [Upgrade v1.2](Upgrade.v1_2.AWS.md)


### Performance and Scalability
The main reporting webapp with 750m CPU and 1G memory will support about 2000 concurrent users. The reporting service supports the individual test result queries and should be able to support about the same, 2000 concurrent users per pod with 500m CPU and 750M memory. 

NOTE: When allocating pod memory please consider the ratio of memory/processor for the nodes. It is silly to restrict memory if it is just going to go to waste because of CPU allocation. So, if the nodes have a 4/1 memory to processor ratio then the allocation should be similarly scaled: using the webapp as an example, it requires 750m processor so giving it 4*750m = 3G memory would be okay. That said, none of these services will do much with >2G, so cap it.
 
Redshift is the backing OLAP data mart for the aggregate reports. Redshift deals with lots of data and is not 
particularly good at high concurrency. The RDW architecture attempts to mitigate this but know that Redshift itself
is the bottleneck when lots of requests are being made. Adding more aggregate-report pods will actually degrade 
performance. And a larger Redshift instance will provide only incremental improvements to overall performance of the
system. Given the cost of the larger instance, it is not advised to increase the size. The aggregate-report pod pool 
size should be coordinated with the Redshift queue size (WLM concurrency). Know that a report request typically needs 
less than 10 queries so, if the queue size is greater than that, the consumer concurrency should be increased so the 
Redshift queue is saturated. A couple examples:
* Typical configuration
  * Redshift queue size = 12  
  * Aggregate-report service pool size = 12
  * Aggregate-report consumer concurrency = 2
* Small configuration
  * Redshift queue size = 6
  * Aggregate-report service pool size = 6
  * Aggregate-report consumer concurrency = 1
* Large configuration
  * Redshift queue size = 30
  * Aggregate-report service pool size = 30
  * Aggregate-report consumer concurrency = 3
* Large configuration, alternate (not extensively tested)
  * Redshift queue size = 30
  * Aggregate-report service pods = 2
  * Aggregate-report service pool size = 15
  * Aggregate-report consumer concurrency = 2

If Redshift performance seems poor please refer to [Redshift Performance Tuning](../docs/PerformanceTuning.Redshift.md).

### Sizing Kubernetes Cluster
Deciding how many nodes and what size they should be requires some consideration (and a little math).

The RDW apps are generally CPU constrained. That is, for a typical node where the ratio of memory to CPU is 4/1 (GB/CPU), it is the CPU that will constrain things. So this discussion will focus on CPU, with the measure in milli-cpu, `m`. A Kubernetes cluster has some overhead for each node (~300m) and for the cluster overall (~600m). The RDW applications have requirements ranging from 200m to 750m. Also consider that each node should have some "headroom" to allow for the surge during updates. 

Given N nodes with C cpu (full CPU not milli-cpu), application requirements of A milli-cpu, ignoring headroom, `1000 x N x C >= A + 600 + 300 x N`. Solving for N, `N >= (A + 600) / (1000C - 300)`. For a given configuration the headroom will be `H = 1000 x N x C - A - 600 - 300 x N`

For example consider a moderate installation with four webapp instances with a total of A = 10000m for all the apps, something like:
```
1 x configuration (100m, 250M)
1 x rabbit (250m, 1G)
1 x admin (500m, 750M)
1 x aggregate (500m, 1G)
2 x exam (300m, 500M)
1 x group (300m, 500M)
2 x import (200m, 400M)
1 x migrate-olap (200m, 400M)
1 x migrate-reporting (200m, 1G)
1 x package (300m, 1G)
4 x wkhtmltopdf (500m, 300M)
2 x report processor (200m, 850M)
3 x report service (500m, 1G)
4 x webapp (750m, 1G)
```

If this were deployed using m4.large nodes (C = 2), N >= 10600 / 1700 = 6.2, it would require 7 nodes. With headroom of 1300m, about 200m per node. To have sufficient headroom, adding a node would provide 3000m, about 375m per node.

If this were deployed using m4.xlarge nodes (C = 4), N >= 10600 / 3700 = 2.9, it would require 3 nodes. With headroom of 500m, barely 150m per node. To have sufficient headroom, adding a node would provide 4200, more than 1000m per node. 

The alert reader will note that larger nodes provide higher overall capacity with larger headroom per node. However, having a single gigantic node is also a bad idea since it fails HA considerations, OS resource constraints, etc. And a node that large is likely to be more expensive per CPU unit. It is a trade-off. Our general advice is to use m4.xlarge nodes.

> NOTE: this discussion is about the standard nodes in a cluster. Master nodes may be sized much smaller: 1000m, 2Gi is sufficient. Given the choices on AWS this implies m4.large master nodes.


### Useful Tricks

#### Run a Database Client Pod

Although the jump server may be used to access the database, a client pod installed in the cluster is the best test of
connectivity and routing.

```bash
# mysql-client (MySQL/Aurora)
kubectl run -it --rm --image=mysql:5.6 mysql-client -- mysql -h hostname -P port -uusername -ppassword opus
# psql (PostgreSQL/Redshift)
kubectl run -it --rm --image=jbergknoff/postgresql-client psql postgresql://username:password@host:5439/opus
```

