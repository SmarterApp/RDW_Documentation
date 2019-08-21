## CI

CI, i.e. Continuous Integration aka Building Artifacts.

Each repository has instructions for building and testing. But if a new team starts working on the project they will want to set up CI. This document serves as a hint for that setup.

The build agent must have mysql 5.6 installed and running locally (this can be changed to use a remote data server but that isn't documented, yet).
The build process produces artifacts (i.e. jars) published to jfrog artifactory, and docker images published to docker hub. You'll need credentials for them.   

The [Schema](https://github.com/SmarterApp/RDW_Schema) project contains scripts and code for creating and migrating databases. 
It is used during installation to create the initial databases and by the tenant administration feature to create databases for new tenants and sandboxes.
It is a gradle project that produces a jar. The gradle script has a number of tasks to run scripts, etc.
The build steps:
1. `gradle cleanAll_test migrateAll_test`
1. `gradle clean distZip -PbuildNumber=<build-number>`
1. publish versioned jar to jfrog

The [Common](https://github.com/SmarterApp/RDW_Common) project is a collection of modules shared by multiple projects.
It is a gradle project that produces a jar.
The build steps:
1. `gradle clean build -PbuildNumber=<build-number>`
    * There is one test that will run only if environment variables are set with AWS S3 credentials:
    ```
    -Dtest-archive.root=s3://rdw-ci 
    -Dtest-archive.cloud.aws.credentials.accessKey=AK... 
    -Dtest-archive.cloud.aws.credentials.secretKey=...
    ```
1. publish versioned jar to jfrog

The [Ingest](https://github.com/SmarterApp/RDW_Ingest) project contains the ingest services.
It is a gradle project that produces a number of docker images.
It has unit tests, integration tests, and Redshift integration tests.
It requires resources (mysql, redshift, s3) to run all the tests.
It depends on RDW_Common and RDW_Schema.
The build steps:
1. Build and test:
    * QA build - `gradle clean build coverage runMTTest -PbuildNumber=<build-number>`
    * Release build - `gradle clean build coverage runMTTest -PbuildNumber=RELEASE -Prelease`
1. (Optional) Run the ITs and RSTs. These require resources (aurora, redshift, s3) and credentials:
```
./gradlew \
-Predshift_url=jdbc:redshift://rdw-ci.cibkulpjrgtr.us-west-2.redshift.amazonaws.com:5439/ci \
-Predshift_user=ci -Predshift_password=<password> \
-Pdatabase_url=jdbc:mysql://rdw-aurora-ci.cugsexobhx8t.us-west-2.rds.amazonaws.com:3306 \
-Pdatabase_user=ci -Pdatabase_password=<password> \
cleanAllTest migrateAllTest

(SERVER=rdw-aurora-ci.cugsexobhx8t.us-west-2.rds.amazonaws.com:3306; USER=sbac; PSWD=sbac4rdw; \
 export ARCHIVE_URI_ROOT=s3://rdw-ci; \
 export ARCHIVE_S3_ACCESS_KEY=AK...; \
 export ARCHIVE_S3_SECRET_KEY=...; \
 export ARCHIVE_S3_REGION_STATIC=us-west-2; \
 export DATASOURCES_REPORTING_RW_URL_SERVER=$SERVER; \
 export DATASOURCES_REPORTING_RW_USERNAME=$USER; export DATASOURCES_REPORTING_RW_PASSWORD=$PSWD; \
 export DATASOURCES_WAREHOUSE_RW_URL_SERVER=$SERVER; \
 export DATASOURCES_WAREHOUSE_RW_USERNAME=$USER; export DATASOURCES_WAREHOUSE_RW_PASSWORD=$PSWD; \
 export DATASOURCES_MIGRATE_RW_PASSWORD=<password>; \
 export DATASOURCES_OLAP_RW_PASSWORD=<password>; \
 ./gradlew manualIT RST)
```
1. Push images:
    * QA build - `gradle dockerPushImage -PbuildNumber=<build-number>`
    * Release build - `gradle dockerPushImage -PbuildNumber=RELEASE -Prelease`

The [Reporting](https://github.com/SmarterApp/RDW_Reporting) project contains the reporting services.
It is a gradle project that produces a number of docker images.
It has unit tests, integration tests, and Redshift integration tests.
It requires resources (mysql, redshift, s3) to run all the tests.
It depends on RDW_Common and RDW_Schema.
The build steps:
1. Build and test:
    * QA build - `gradle clean build coverage testFe runMTTest`
    * Release build - `gradle clean build coverage testFe runMTTest -Pprod -PbuildNumber=RELEASE -Prelease`
1. (Optional) Run the ITs and RSTs. These require resources (aurora, redshift, s3) and credentials:
```
./gradlew \
-Predshift_url=jdbc:redshift://rdw-ci.cibkulpjrgtr.us-west-2.redshift.amazonaws.com:5439/ci \
-Predshift_user=ci -Predshift_password=<password> \
-Pdatabase_url=jdbc:mysql://rdw-aurora-ci.cugsexobhx8t.us-west-2.rds.amazonaws.com:3306 \
-Pdatabase_user=ci -Pdatabase_password=<password> \
cleanAllTest migrateAllTest

(SERVER=rdw-aurora-ci.cugsexobhx8t.us-west-2.rds.amazonaws.com:3306; USER=ci; PSWD=<password>; \
 export DATASOURCES_REPORTING_RO_URL_SERVER=$SERVER; \
 export DATASOURCES_REPORTING_RO_USERNAME=$USER; export DATASOURCES_REPORTING_RO_PASSWORD=$PSWD; \
 export DATASOURCES_WAREHOUSE_RW_URL_SERVER=$SERVER; \
 export DATASOURCES_WAREHOUSE_RW_USERNAME=$USER; export DATASOURCES_WAREHOUSE_RW_PASSWORD=$PSWD; \
 export DATASOURCES_REPORTING_RW_URL_SERVER=$SERVER; \
 export DATASOURCES_REPORTING_RW_USERNAME=$USER; export DATASOURCES_REPORTING_RW_PASSWORD=$PSWD; \
 export TEST_AURORA=true \
 export DATASOURCES_OLAP_RO_PASSWORD=<password>; \
 ./gradlew manualIT RST)
 ```
1. Push images:
    * QA build - `gradle dockerPushImage -PbuildNumber=<build-number>`
    * Release build - `gradle dockerPushImage -PbuildNumber=RELEASE -Prelease`

