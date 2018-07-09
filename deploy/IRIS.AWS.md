## IRiS

**Intended Audience**: this document describes how to standup an AWS EC2 instance running IRiS. Operations and system administrators will find it useful.

> Although these are generic instructions, having an example makes it easier to document and understand. Throughout this document we'll use `opus` as the sample environment name; this might correspond to `staging` or `production` in the real world. We'll use `sbac.org` as the domain/organization. Any other ids, usernames or other system-generated values are products of the author's imagination. The reader is strongly encouraged to use their own consistent naming conventions. **Avoid putting real environment-specific details _especially secrets and sensitive information_ in this document.**

### General Steps
1. Create new EC2 instance
1. Install IRIS
1. Configure IRIS
1. Create and mount EFS
1. Upload content to EFS
1. Create ELB

### Detailed Steps
1. Create new EC2 instance
    * Centos, i3.large
    * Network: select environment VPC, Subnet: any private
    * Tag: Name=rdw-opus-iris-1 
    * Security: inbound allow SSH (22) and Custom TCP (8080) from VPC
1. Install IRIS. It has stated dependencies on Java 7 and Tomcat 7.
    ```bash
    sudo yum install -y java-1.7.0-openjdk
    # Install Tomcat Server 7 (if not installed already), stop it and wipe out default ROOT folder
    sudo yum install -y tomcat  
    sudo service tomcat stop
    sudo rm -rf /var/lib/tomcat7/webapps/ROOT
    # download latest (2.1.2 as of this writing) IRIS WAR file
    sudo yum install -y wget
    sudo wget -O /var/lib/tomcat/webapps/iris.war https://airdev.artifactoryonline.com/airdev/libs-releases-local/org/opentestsystem/delivery/iris/2.1.2/iris-2.1.2.war
    # configure tomcat to start on boot
    sudo chkconfig tomcat on
    ```
1. Configure IRIS. 
   More details on configuring IRIS can be found on the IRIS README if needed here: https://github.com/SmarterApp/TDS_IRIS
    * The IRIS Encryption key is set in Tomcat's context file: `context.xml`. With Ubuntu and Tomcat 7 this file is located `/etc/tomcat/context.xml` but may differ depending on the operating system. Add the following line within the `<Context>` element in the context file:
        * `<Parameter name="tds.iris.EncryptionKey" override="false" value="24 character or more alphanumeric Encryption key" />`
    * Restart Tomcat
        * `sudo service tomcat start`
1. Copy item content to `/home/tomcat7/content/`.
Copy item content to EFS by using `scp` or similar utility to `/home/tomcat7/content/`. The structure of the content should be like (note that folder names are case-sensitive and must match; also permissions may have to be adjusted so tomcat user can see files):
    * items/
        * Item-200-xxxx/
        * Item-200-yyyy/
    * stimuli/
        * stim-200-xxx/
        * stim-200-yyy/
    ```bash
    chown -R tomcat:tomcat /home/tomcat7
    find . -type d -exec chmod 755 {} \;
    find . -type f -exec chmod 644 {} \;
    ```
1. Create ELB for IRiS.
    * Classic ELB
    * Name: rdw-opus-iris
    * Inside: opus VPC
    * Create an internal load balancer 
    * Listeners: HTTP (80) -> HTTP (8080)
    * Subnets: add the three opus private subnets
    * Load Balancer rules (via new security group, rdw-opus-iris-elb):
        * inbound: HTTP/TCP port 80, VPC CIDR  (10.3.0.0/16)
    * Health check: ping path `/iris/`
1. Test. From a box inside stage VPC:
    * curl -I http://internal-rdw-opus-iris-[aws-randomization]/iris/

       
### Upgrading IRiS        
If a compatible IRiS release is available, follow these steps to upgrade.
```bash
sudo su
# get new release of IRiS
wget https://airdev.artifactoryonline.com/airdev/libs-releases-local/org/opentestsystem/delivery/iris/3.2.3.RELEASE/iris-3.2.3.RELEASE.war
# if that doesn't work you can try
# wget https://github.com/SmarterApp/TDS_IRIS/releases/download/3.2.3.RELEASE/iris.war -O iris-3.2.3.RELEASE.war
# stop tomcat, remove old app, clean up old (>7 days) logs, install new war, restart tomcat
service tomcat stop
rm -rf /var/lib/tomcat/webapps/*
find /var/log/tomcat/ -mtime +7 -exec rm {} \;
cp iris-3.2.3.RELEASE.war /var/lib/tomcat/webapps/iris.war
chown root:tomcat /var/lib/tomcat/webapps/iris.war
service tomcat start
```

### Reload IRiS
There is an end-point that causes IRiS to flush its cache and reload the content. This may be necessary if the content is modified. It also fixed things when IRiS decided to stop serving up certain item content.
```bash
curl http://localhost:8080/iris/Pages/API/content/reload
```
    
### Performance and Scalability
As part of another project Fairway performed comprehensive IRIS load testing that indicated that a single i3.large instance can support 500 concurrent IRIS users. The i3.large instance type has high-performance I/O (disk and network).

In the current configuration the IRIS instances are using a shared EFS for the resources. New load testing has shown that this leads to unacceptable performance with average latency approaching 1s (300-1200ms) and overall capacity of <800 concurrent reporting web app users. Switching to local SSD storage reduces average latency to <40ms and overall capacity of >6000 concurrent reporting web app users. To support the target of 15000 concurrent users we are recommending 2-3 i3.large instances with the data loaded locally.
