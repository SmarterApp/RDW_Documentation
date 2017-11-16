## Deployment Checklist

**NOTE: please avoid putting environment-specific details _especially secrets and sensitive information_ in this document.**

**Intended Audience**: this document provides a detailed checklist for deploying the Reporting Data Warehouse (RDW) applications in an AWS environment. Operations and system administrators will find it useful.

The checklist includes all steps needed to deploy RDW. The reference section notes all details that should be recorded to facilitate maintenance of the system.

### Summary / Reference

* Deployment Repository: https://github.com/
* Configuration Repository: https://github.com/
* AWS
    * AWS account:
    * Signin link:
    * Region: us-west-2
* Hosted Zone: rdw.smarterbalanced.org
* AWS VPC: (name, CIDR block, region)
    * (subnet name, CIDR block, region)
* SSO host: sso.smarterbalanced.org
* ART host: art.smarterbalanced.org
* Perms host: perm.smarterbalanced.org
* Ops system / jump server:
* IRIS

### Checklist

* [ ] Deployment Repository. This is the source repository that will store your deployment documentation and the kubernetes specifications for the applications. You are probably reading this document from there. This must be a private repository since it contains credentials and other sensitive information.
* [ ] Configuration Repository. This is the source repository that will store the application configuration files. This must be a private repository as it will contain some sensitive credentials (note that most credentials will be one-way hashed even in the private repository).
* [ ] Prepare AWS
    * [ ] Create an AWS account. All the RDW services will run in this account.