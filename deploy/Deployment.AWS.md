## Deployment Checklist

**Intended Audience**: this document provides a detailed checklist for deploying the Reporting Data Warehouse (RDW) applications in an AWS environment. Operations and system administrators will find it useful.

> Although these are generic instructions, having an example makes it easier to document and understand. Throughout this document we'll use `opus` as the sample environment name; this might correspond to `staging` or `production` in the real world. We'll use `sbac.org` as the domain/organization. Any other ids, usernames or other system-generated values are products of the author's imagination. The reader is strongly encouraged to use their own consistent naming conventions. **Avoid putting real environment-specific details _especially secrets and sensitive information_ in this document.**

**TODO - the keystore instructions (for reporting and admin) seem partial, perhaps missing some details?**

**TODO - instructions for creating OpenAM oauth2 agent client id/password?**

### Table Of Contents
* [Reference](#reference)
* [Checklist](#checklist)
* [Updating Applications](#updating-applications)

<a name="reference"></a>
### Summary / Reference
This section records all details that will facilitate configuration and maintenance of the system. It should be filled in as the deployment checklist is worked.

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
    * ingest user: rdw-ingest/password
* Aurora reporting:
    * name: rdw-opus-reporting
    * root user: root/password
    * read-write: rdw-opus-reporting-cluster.cluster-[aws-randomization]
    * read-only: rdw-opus-reporting-cluster.cluster-ro-[aws-randomization]
    * reporting user: rdw-reporting/password
* Redis: rdw-opus-redis.[aws-randomization]
* Archive S3 bucket: 
    * name: rdw-opus-archive
    * user: rdw-opus, accesskey/secretkey
* Reconciliation report S3 bucket:
    * name: recon-bucket
    * user: name, accesskey/secretkey
* Import:
    * end-point: import.sbac.org
    * ELB id: [aws-randomization]
* Reporting:
    * end-point: reporting.sbac.org
    * ELB id: [aws-randomization]
* Admin:
    * end-point: admin.sbac.org
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
    * [ ] Configure ops system (after going through these steps consider creating an image).
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
* [ ] SBAC Application Prerequisites. These are non-RDW applications that are maintained as part of the SBAC ecosystem. They run independently of RDW and are not part of the Kubernetes cluster.
    * [ ] SSO. 
        * Create OpenAM OAuth2 client.
        * *Record the host and client info in the reference.*
    * [ ] ART. 
        * [ ] Create ingest user for the task service user. This is used by the task service to fetch organization information from ART and submit it to the import service. As such it should have `Administrator`, `State Coordinator` and `ASMTDATALOAD` roles at the state level for `CA`.
        * *Record ART host and username/password in the reference.*
    * [ ] Permission Service. 
        * *Record the host in the reference.*
        * Add/verify component 
            * `Reporting`
        * Add/verify roles: 
            * `GROUP_ADMIN` - District, GroupOfInstitutions, Institutions
            * `PII` - all but GroupOfStates
            * `PII_GROUP` - all but GroupOfStates 
        * Add/verify component permissions for `Reporting`
            * `GROUP_PII_READ`, `GROUP_READ`, `GROUP_WRITE`, `INDIVIDUAL_PII_READ`
        * Add/verify mappings of Reporting component permissions (rows) to roles (columns) 
            * `GROUP_PII_READ` - `PII`, `PII_GROUP`
            * `GROUP_READ` - `GROUP_ADMIN`, `PII`, `PII_GROUP`
            * `GROUP_WRITE` - `GROUP_ADMIN`
            * `INDIVIDUAL_PII_READ` - `PII`
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
* [ ] Create Aurora instances. The system requires two aurora instances, `warehouse` and `reporting`.
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
           -Pflyway.user=root -Pflyway.password=password migrateWarehouse
        ./gradlew -Pflyway.url="jdbc:mysql://rdw-opus-reporting-cluster.[aws-randomization]:3306/" \
           -Pflyway.user=root -Pflyway.password=password migrateReporting
        ```   
    1. Create Aurora/MySQL users: rdw-ingest (both), rdw-reporting (reporting)
        ```bash
        mysql -h rdw-opus-warehouse-cluster.[aws-randomization] -u root -p
        mysql> create user 'rdw-ingest'@'%' identified by 'password';
        mysql> grant all privileges on warehouse.* to 'rdw-ingest'@'%';
        mysql> grant LOAD FROM S3 ON *.* to 'rdw-ingest'@'%';
        mysql> grant select on mysql.proc to 'rdw-ingest'@'%';
        mysql -h rdw-opus-reporting-cluster.[aws-randomization] -u root -p
        mysql> create user 'rdw-ingest'@'%' identified by 'password';
        mysql> create user 'rdw-reporting'@'%' identified by 'password';
        mysql> grant all privileges on reporting.* to 'rdw-ingest'@'%';
        mysql> grant all privileges on reporting.* to 'rdw-reporting'@'%';
        mysql> quit       
        ```
        * *Record the username/passwords in the reference.*
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
* [ ] Create and configure archive S3 bucket
    1. Create unversioned S3 bucket for archive and a policy for accessing it using console or CLI.
        ```bash
        aws s3 mb s3://rdw-opus-archive --region us-west-2
        aws iam create-policy --policy-name AllowRdwOpusArchiveBucket --description "Allow any S3 actions in rdw-opus-archive" --policy-document file://AllowRdwOpusArchiveBucket.json
        ```
    1. Create a role to allow RDS access to the S3 bucket using console or CLI.
        ```bash
        aws iam create-role --role-name rdsRdwOpusArchiveRole --description "RDS access to rdw-opus-archive" --assume-role-policy-document file://ArchiveRoleTrustPolicy.json
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
* [ ] Install and configure supporting services (autoscaler, rabbit, wkthmltopdf, configuration service)
    1. Edit cluster-autoscaler.yml to verify --nodes option matches cluster configuration
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
* [ ] Install and configure backend RDW services.
    1. Generate encrypted passwords/secrets and set in configuration files. Need configuration service port-forward or exec into pod and install curl.
        ```bash
        kubectl port-forward configuration-deployment-xxx 8888 
        curl -X POST --data-urlencode "my+secret" http://localhost:8888/encrypt
        ```
    1. Set datasource urls, username, passwords in configuration files.
    1. Set S3 bucket, region, access key, secret key in configuration files.
    1. Set reconciliation S3 bucket in rdw-ingest-task-service.yml
    1. Configure load balancer certificate for import service.
        * Create cert for import.sbac.org (or whever you'll be putting this service)
        * Modify import-service.yml and set ssl-cert to ARN of newly created cert.
    1. Create services.
        ```bash
        kubectl create -f exam-processor-service.yml
        kubectl create -f package-processor-service.yml
        kubectl create -f task-service.yml
        kubectl create -f migrate-reporting-service.yml
        kubectl create -f import-service.yml     
        kubectl create -f report-processor-service.yml
        kubectl create -f group-processor-service.yml
        ```
    1. Add route for import service.
        * Get ELB id by describing service `kubectl describe service import-service`.
        * Verify ELB listeners and security group inbound rules are set to allow HTTP and HTTPS access.  
        * Add Route53 CNAME record for `import.sbac.org` to the ELB for the service.
        * *Record import service ELB info in the reference.*
* [ ] Install RDW webapp services. xxx = {reporting,admin}
    1. Create reporting and admin keystores. 
        * `keytool -genkeypair -alias rdw-xxx-sp -keypass password -keystore rdw-xxx.jks`
        * Keystore password: password
    1. Add SSO IDP's public key to keystore:
        * Capture certificate: `openssl s_client -connect sso.sbac.org:443 > sso.crt`
        * Edit file and delete everything that isn't between (and including) BEGIN/END lines 
        * Import cert into keystore: `keytool -import -trustcacerts -alias sso -file ./sso.crt -keystore ./rdw-xxx.jks`
        (type Y to trust this certificate)
        * Test `keytool -v -list -keystore rdw-xxx.jks | grep -i alias`
        * Add rdw-xxx.jks file to RDW_Config_Opus repo
    1. Configure related properties in xxx-service.yml.
    1. Configure load balancer certificates.
        * Create cert for xxx.sbac.org (or whever you'll be putting these services)
        * Modify xxx-service.yml and set ssl-cert to ARN of newly created cert.
    1. Create services.
        ```bash
        kubectl create -f reporting-service.yml
        kubectl create -f admin-service.yml
        ```
    1. Add route for each of xxx-service. 
        * Get ELB id by describing service `kubectl describe service xxx-service`.
        * Verify ELB listeners and security group inbound rules are set to allow HTTP and HTTPS access.  
        * Add Route53 CNAME record for `xxx.sbac.org` to the ELB for the service.
        * *Record service ELB info in the reference.*
    1. Register reporting and admin services with OpenAM as service providers (SPs).
        * Ensure configuration properties set properly; saml.sp-entity-id and saml.load-balance.*
        * Generate rdw-opus-xxx.sp.xml by using curl to `GET ./saml/metadata`
            * verify SP id and entity id is rdw-opus-xxx 
            * verify all service hosts set to `https://xxx.sbac.org/...`
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
    1. Load assessment packages. These are created by the tabulator; they are not provided here because they are propietary in nature. For each file:
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
        
### Updating Applications
When software updates are available, use the rollout capability of Kubernetes to deploy it.
1. Determine the images needed for the update. Consider that the ingest and reporting applications may be on different release cycles, and certain applications may have patch releases.
1. Update the deployment specs to reflect the changes. Specifically change the container images.
1. Make any corresponding changes to the configuration files served by the configuration server.
1. Either apply the modified spec OR directly update the image. This will be repeated for all affected applications but a single example:
    ```bash
    kubectl apply -f import-service.yml
    OR
    kubectl set image deployment import-deployment api="smarterbalanced/rdw-ingest-import-service:1.2.3-RELEASE"
    ```
