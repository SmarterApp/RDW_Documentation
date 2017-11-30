## API

**Intended Audience**: this document describes the programming interface for the ingest and reporting services of the [Reporting Data Warehouse](../README.md). Operations and system adminstration will find it useful for the configuration data loading end-points and the diagnostic and task service end-points. Third party providers of test results will find it useful for the data loading end-points.

This document describes the service end-points for the ingest and reporting services. *Note: there are many more end-points in the reporting services but most are intended to be consumed by the reporting UI, so are not documented here.*
* The import service has data loading end-points.
* The task and report processor services have task trigger end-points. 
* All services have diagnostic end-points. 

Quick Links:
1. [Authentication and Authorization](#authentication-and-authorization)
1. [Import Endpoints](#import-endpoints)
1. [Exam Endpoints](#exam-endpoints)
1. [Organization Endpoints](#organization-endpoints)
1. [Accommodations Endpoints](#accommodations-endpoints)
1. [Package Endpoints](#package-endpoints)
1. [Task Endpoints](#task-endpoints)
1. [Status Endpoints](#status-endpoints)

### Authentication and Authorization
The import service requires an OAuth2 access token for the data loading end-points. This is a password grant token requested from the OpenAM server by a trusted client for a user of the system (the permissions are associated with the user). 

#### Fetch Password Grant Access Token
Accepts x-www-form-urlencoded data including client and user credentials and returns an access token.
* Host: OpenAM
* URL: `/auth/oauth2/access_token`
* Method: `POST`
* URL Params: 
  * `realm=/sbac`
* Data Params (except for `grant_type` values given are examples and should be replaced with real values):
  * `grant_type=password`
  * `username=user@example.com`
  * `password=UserPassword`
  * `client_id=client`
  * `client_secret=ClientSecret`
* Success Response:
  * Code: 200 (OK)
  * Content: 
    ```json
    {
      "scope": "cn givenName mail sbacTenancyChain sbacUUID sn",
      "expires_in": 36000,
      "token_type": "Bearer",
      "refresh_token": "639ddaa9-f993-4c52-aec5-923d5a21ee23",
      "access_token": "20b55fc2-1b84-4412-8149-88cfa622db01"
    } 
    ```
* Error Response:
  * Code: 400 (Bad Request)
  * Content (specific error and description will vary):
    ```json
    {
      "error": "invalid_client",
      "error_description": "Client authentication failed"
    }
    ```
* Sample Call (curl):
```bash
curl -s -X POST \
  --data "grant_type=password&username=user@example.com&password=password&client_id=client&client_secret=secret" \
  https://openam/auth/oauth2/access_token?realm=/sbac
```

#### Test Access Token
Although not needed during normal operations, this call can be used to check an access token.
* Host: OpenAM
* URL: `/auth/oauth2/tokeninfo`
* Method: `GET`
* URL Params: 
  * `access_token={access_token}`
* Success Response:
  * Code: 200 (OK)
  * Content: 
    ```json
    {
      "sbacUUID": "599758d9e4b0fbecbf5fc586",
      "mail": "test@example.com",
      "sn": "Test",
      "scope": [
         "sbacUUID",
         "mail",
         "sn",
         "cn",
         "sbacTenancyChain",
         "givenName"
      ],
      "grant_type": "password",
      "cn": "Test",
      "realm": "/sbac",
      "sbacTenancyChain": "|CA|ASMTDATALOAD|STATE|1000|ART_DL|||CA|CALIFORNIA|||||||||",
      "token_type": "Bearer",
      "expires_in": 35930,
      "givenName": "Test",
      "access_token": "20b55fc2-1b84-4412-8149-88cfa622db01"
    }
    ```
* Error Response:
  * Code: 400 (Bad Request)
  * Content (specific error and description will vary):
    ```json
    {
      "error": "invalid_request",
      "error_description": "Access Token not valid"
    }
    ```
* Sample Call (curl):
```bash
curl https://openam/auth/oauth2/tokeninfo?access_token=20b55fc2-1b84-4412-8149-88cfa622db01
```

### Import Endpoints
The import service provides the end-points for submitting data to the system. The import requests are processed and migrated to the reporting data mart. Import payloads are hashed and duplicate content is detected, returning any previous import request for the given content. Thus submitting a payload a second time will safely no-op and return the current status of the previous import.

All data submissions result in an import being created. Thus a POST to `/exams/imports` will create an `import` resource which can be accessed at `/imports/{id}` (note that `exams` is not in the path). These are the end-points for query imports.
 
All end-points require a valid token, the examples use `{access_token}` as a placeholder. The token must provide the `ASMTDATALOAD` role at the client or state level.

#### Get Import Request
This end-point may be used to get the current status of an import request.

* Host: import service
* URL: `/imports/{id}`
* Method: `GET`
* Params: none
* Headers:
  * `Authorization: Bearer {access_token}`
* Success Response:
  * Code: 200 (OK)
  * Content:
    ```json
    {
      "id": 19529,
      "content": "EXAM",
      "contentType": "application/xml",
      "digest": "5899C64887FE25BC1F015FD7C26476E4",
      "status": "BAD_DATA",
      "creator": "user@example.com",
      "created": "2017-05-08T19:45:20.476Z",
      "message": "{\"elementName\":\"Sex\",\"value\":\"M\",\"error\":\"unknown gender name [M]\"},{\"elementName\":\"GradeLevelWhenAssessed\",\"value\":\"SIXTHGRADE\",\"error\":\"unknown grade code [SIXTHGRADE]\"}",
      "_links": {
        "self": {
          "href": "http://import-service/imports/19529"
        },
        "payload": {
          "href": "http://import-service/imports/19529/payload"
        },
        "payload-properties": {
          "href": "http://import-service/imports/19529/payload/properties"
        }  
      }
    }
    ```
* Error Response:
  * Code: 401 (Unauthorized) if token is missing or invalid.
  * Code: 403 (Forbidden) if token doesn't provide the `ASTMDATALOAD` role.
  * Code: 404 (Not Found) if no import with the given id exists.
* Sample Call (curl):
```bash
curl --header "Authorization:Bearer {access_token}" https://import-service/imports/19529
```

#### Get Import Payload
This end-point may be used to get the payload for an import request. 

* Host: import service
* URL: `/imports/{id}/payload`
* Method: `GET`
* Params: none
* Headers:
  * `Authorization: Bearer {access_token}`
* Success Response:
  * Code: 200 (OK)
  * Content: File attachment
* Error Response:
  * Code: 401 (Unauthorized) if token is missing or invalid.
  * Code: 403 (Forbidden) if token doesn't provide the `ASTMDATALOAD` role.
  * Code: 404 (Not Found) if no import with the given id exists.
* Sample Call (curl):
```bash
curl --header "Authorization:Bearer {access_token}" https://import-service/imports/19529/payload
```

#### Get Import Payload Properties
This end-point may be used to get the payload properties for an import request. This can be useful because not all the properties for an import are stored in the data warehouse, some are archived only with the payload.

* Host: import service
* URL: `/imports/{id}/payload/properties`
* Method: `GET`
* Params: none
* Headers:
  * `Authorization: Bearer {access_token}`
* Success Response:
  * Code: 200 (OK)
  * Content: 
    ```json
    {
      "Content-Type": "application/xml",
      "filename": "test.xml",
      "batch": "CAF4901EA92C0A77A6DB6C37E8582852",
      "tenancy-chain": "|SBAC|ASMTDATALOAD|CLIENT|SBAC||||||||||||||",
      "username": "user@example.com"
    }
    ```
* Error Response:
  * Code: 401 (Unauthorized) if token is missing or invalid.
  * Code: 403 (Forbidden) if token doesn't provide the `ASTMDATALOAD` role.
  * Code: 404 (Not Found) if no import with the given id exists.
* Sample Call (curl):
```bash
curl --header "Authorization:Bearer {access_token}" https://import-service/imports/19529/payload/properties
```


### Exam Endpoints
End-points for submitting exams aka test results.

#### Create Exam Import Request
Accepts payloads in Test Result Transmission ([TRT][3]) format, creating new exam import requests. Based on the data in the TRT this may result in an exam being created, updated or deleted. Exams are identified using the opportunity id and assessment id. If an incoming exam is not found in the system one will be added. If a matching exam already exists, it will be updated or, if the new status is `reset`, it will be deleted.

There are two ways of posting exam content: with a raw body of type `application/xml` or form-data (file upload).

* Host: import service
* URL: `/exams/imports`
* Method: `POST`
* URL Params: any URL param will be preserved as properties for the upload, well-known params:
  * `batch=<batchtag>`
  * `filename=<originalFileName>`  (not needed for form-data)
* Headers:
  * `Authorization: Bearer {access_token}`
* Headers (raw body):
  * `Content-Type:application/xml`
* Body (raw body): TRT
* Headers (form-data):
  * `Content-Type:multipart/form-data`
* Form Data (form-data):
  * `file` - the upload file
  * `filename=<filename>`
  * any other form data will be preserved as properties for the upload
* Success Response:
  * Code: 202 (Accepted)
  * Content:
    ```json
    {
      "id": 19529,
      "content": "EXAM",
      "contentType": "application/xml",
      "digest": "5899C64887FE25BC1F015FD7C26476E4",
      "status": "ACCEPTED",
      "creator": "user@example.com",
      "created": "2017-05-08T19:45:20.476Z",
      "_links": {
        "self": {
          "href": "http://import-service/imports/19529"
        }
      }
    }
    ```
* Error Response:
  * Code: 401 (Unauthorized) if token is missing or invalid.
  * Code: 403 (Forbidden) if token doesn't provide the `ASTMDATALOAD` role.
* Sample Call (raw body):
```bash
curl -X POST --header "Authorization:Bearer {access_token}" --header "Content-Type:application/xml" \
  --data-binary "<TDSReport><Test..." https://import-service/exams/imports?batch=WinterICA&filename=winterICA.1.xml
```
* Sample Call (form-data):
```bash
curl -X POST --header "Authorization:Bearer {access_token}" -F file=@winterICA.1.xml -F batch=WinterICA \
  https://import-service/exams/imports
```
  
#### Resubmit Exams
The system accepts and stores import payloads regardless of processing status. For imports with certain problems like an unknown school or assessment, the import may be "replayed" once the underlying cause is resolved. For example, if the missing school is added to the system, all affected exams can be resubmitted, without requiring the client to re-post the data. Note that this is meant as an operational utility; if it is being used regularly there are probably fundamental data flow issues. 

* Host: import service
* URL: `/exams/imports/resubmit`
* Method: `POST`
* URL Params: query params can be used to restrict which exams to include; typically the status
  * `status=<status>` where status may be `UNKNOWN_SCHOOL`, `UNKNOWN_ASMT`, `ACCEPTED`
  * `after=<interval>` where interval is like `-PT1H` 
  * `creator=<creator>`
  * `batch=<batchtag>`
* Headers:
  * `Authorization: Bearer {access_token}`
* Success Response:
  * Code: 200
  * Content: number of exams found and resubmitted. Note that a single call is limited to 100. If the return value is
  100, it is likely there are more exams matching the query and additional resubmit calls should be made. 
    ```text
    7
    ```
* Error Response:
  * Code: 401 (Unauthorized) if token is missing or invalid.
  * Code: 403 (Forbidden) if token doesn't provide the `ASTMDATALOAD` role.
* Sample Call:
```bash
curl -X POST --header "Authorization:Bearer {access_token}" https://import-service/exams/imports/resubmit?status=UNKNOWN_SCHOOL
```

  
### Organization Endpoints
End-points for submitting organization data; this includes districts, schools, and groups of institutions. 

#### Create Organization Import Request
Accepts JSON payload, compatible with the format produced by ART. The payload should contain all the organization data necessary to resolve all contents; for example, if a school is in a group under a district, all three must be present in the payload. The payload must be valid JSON but the exact structure doesn't matter a lot: the system will parse the payload looking for the required fields: `entityType`, `entityId`, `entityName`, `parentEntityId`. 

There are two ways of posting content: with a raw body of type `application/json` or form-data (file upload).

* Host: import service
* URL: `/organizations/imports`
* Method: `POST`
* URL Params: any URL param will be preserved as properties for the upload, well-known params:
  * `filename=<originalFileName>`  (not needed for form-data)
* Headers:
  * `Authorization: Bearer {access_token}`
* Headers (raw body):
  * `Content-Type:application/json`
* Body (raw body): JSON payload, contrived example:
```json
{ "districts": [
    {
      "id": "572d7630e4b0ed2c55c37e34",
      "entityId": "DISTRICT9",
      "entityName": "District 9 - Prawn Town",
      "parentEntityId": "CA",
       "entityType": "DISTRICT"
    }, ...
  ], "schools": [
    {
      "id": "57325fc6e4b0ed2c55c37e40",
      "entityId": "DS9001",
      "entityName": "Prawn Town Middle School",
      "parentEntityId": "DISTRICT9",
      "nationwideIdentifier": "NCESID001",
      "entityType": "INSTITUTION"
    }, ...
  ]
}
```
* Headers (form-data):
  * `Content-Type:multipart/form-data`
* Form Data (form-data):
  * `file` - the upload file
  * `filename=<filename>`
  * any other form data will be preserved as properties for the upload
* Success Response:
  * Code: 202 (Accepted)
  * Content:
    ```json
    {
      "id": 19529,
      "content": "ORGANIZATION",
      "contentType": "application/json",
      "digest": "5899C64887FE25BC1F015FD7C26476E4",
      "status": "ACCEPTED",
      "creator": "user@example.com",
      "created": "2017-05-08T19:45:20.476Z",
      "_links": {
        "self": {
          "href": "http://import-service/imports/19529"
        }
      }
    }
    ```
* Error Response:
  * Code: 401 (Unauthorized) if token is missing or invalid.
  * Code: 403 (Forbidden) if token doesn't provide the `ASTMDATALOAD` role.
* Sample Call (raw body):
```bash
curl -X POST --header "Authorization:Bearer {access_token}" --header "Content-Type:application/json" \
  --data-binary "{ \"districts\": [..." https://import-service/organizations/imports?filename=ART.json
```
* Sample Call (form-data):
```bash
curl -X POST --header "Authorization:Bearer {access_token}" -F file=@ART.json https://import-service/organizations/imports
``` 
    
    
### Accommodations Endpoints
End-points for submitting accommodations data. This is data authored by Smarter Balanced that describes the various accessibility features available during testing. 

#### Create Accommodation Import Request
Accepts payloads in the Smarter Balanced [Accessibility Configuration Specification][2] format. There are two ways of posting content: with a raw body of type `application/xml` or form-data (file upload).

* Host: import service
* URL: `/accommodations/imports`
* Method: `POST`
* URL Params: any URL param will be preserved as properties for the upload, well-known params:
  * `filename=<originalFileName>`  (not needed for form-data)
* Headers:
  * `Authorization: Bearer {access_token}`
* Headers (raw body):
  * `Content-Type:application/xml`
* Body (raw body): XML payload, snippet:
```xml
<?xml version="1.0" encoding="utf‐8"?>
<Accessibility>
  <MasterResourceFamily>
    <SingleSelectResource>
      <Code>AmericanSignLanguage</Code>
      <Order>1</Order>
      <DefaultSelection/>
      <ResourceType>Accommodation</ResourceType>
      <Text>
        <Language>eng</Language>
        <Label>American Sign Language</Label>
  ...
</Accessibility>
```
* Headers (form-data):
  * `Content-Type:multipart/form-data`
* Form Data (form-data):
  * `file` - the upload file
  * `filename=<filename>`
  * any other form data will be preserved as properties for the upload
* Success Response:
  * Code: 202 (Accepted)
  * Content:
    ```json
    {
      "id": 179583,
      "content": "CODES",
      "contentType": "application/xml",
      "digest": "2278A9293FE3742C220EEE991E450156",
      "status": "ACCEPTED",
      "creator": "user@example.com",
      "created": "2017-05-08T19:45:20.476Z",
      "_links": {
        "self": {
          "href": "http://import-service/imports/179583"
        }
      }
    }
    ```
* Error Response:
  * Code: 401 (Unauthorized) if token is missing or invalid.
  * Code: 403 (Forbidden) if token doesn't provide the `ASTMDATALOAD` role.
* Sample Call (raw body):
```bash
curl -X POST --header "Authorization:Bearer {access_token}" --header "Content-Type:application/xml" \
  --data-binary "<?xml ..." https://import-service/accommodations/imports?filename=accommodations.xml
```
* Sample Call (form-data):
```bash
curl -X POST --header "Authorization:Bearer {access_token}" -F file=@accommodations.xml https://import-service/accommodations/imports
``` 
        
### Package Endpoints
End-points for submitting assessment package data in CSV format (tabulator output). This is data authored by Smarter Balanced that describes the assessments, items and other test details. This end point can be used to either import new packages or update existing ones. When an assessment is created, the following data elements are considered critical in properly parsing incoming Exam requests and cannot be updated later:
* grade
* subject 
* sub-type (ICA, IAB or Summative)

An attempt to update any of the above data elements will result in the "PROCESSED" status and the message stating that it was rejected.

#### Create Package Import Request
Accepts payloads in the Smarter Balanced Assessment Tabulator format. This is a CSV format produced by the internal
tabulator utility. 

There are two ways of posting content: with a raw body of type `application/csv` or form-data (file upload).

* Host: import service
* URL: `/packages/imports`
* Method: `POST`
* URL Params: any URL param will be preserved as properties for the upload, well-known params:
  * `filename=<originalFileName>`  (not needed for form-data)
* Headers:
  * `Authorization: Bearer {access_token}`
* Headers (raw body):
  * `Content-Type:application/csv`
* Body (raw body): CSV payload, snippet:
```csv
AssessmentId,AssessmentName,AssessmentSubject,AssessmentGrade,AssessmentType,AssessmentSubtype,AssessmentLabel,AssessmentVersion,AcademicYear,FullItemKey,BankKey,ItemId,Filename,Version,ItemType,Grade,Standard,Claim,Target,PassageId,ASL,Braille,LanguageBraille,DOK,Language,AllowCalculator,MathematicalPractice,MaxPoints,Glossary,ScoringEngine,Spanish,IsFieldTest,IsActive,ResponseRequired,AdminRequired,ItemPosition,MeasurementModel,Weight,ScorePoints,a,b0_b,b1_c,b2,b3,avg_b,bpref1,bpref2,bpref3,bpref4,bpref5,bpref6,bpref7,CommonCore,ClaimContentTarget,SecondaryCommonCore,SecondaryClaimContentTarget,CutPoint1,ScaledLow1,ScaledHigh1,CutPoint2,ScaledLow2,ScaledHigh2,CutPoint3,ScaledLow3,ScaledHigh3,CutPoint4,ScaledLow4,ScaledHigh4
(SBAC)SBAC-IAB-FIXED-G11E-BriefWrites-ELA-11-Winter-2014-2015,SBAC-IAB-FIXED-G11E-BriefWrites-ELA-11,ELA,11,interim,IAB,ELA IAB G11 BriefWrites,6579,2015,200-33218,200,33218,item-200-33218.xml,6580,SA,11,SBAC-ELA-v1:2-W|3-11,"2-W	","3-11	",,,,ENU-Braille,3,ENU,,,2,,HandScored,,FALSE,TRUE,TRUE,TRUE,2,IRTGPC,1,2,0.62844,-0.38994,2.66386,,,1.13696,(SBAC)SBAC-IAB-FIXED-G11E-BriefWrites-ELA-11-Winter-2014-2015,,SBAC-2-W|3-11,,,,,"11-12.W.2f	",2-W|3-11,,,1,2.30E+03,2.49E+03,2,2.49E+03,2.58E+03,3,2.58E+03,2.68E+03,4,2.68E+03,2.80E+03
...
```
* Headers (form-data):
  * `Content-Type:multipart/form-data`
* Form Data (form-data):
  * `file` - the upload file
  * `filename=<filename>`
  * any other form data will be preserved as properties for the upload
* Success Response:
  * Code: 202 (Accepted)
  * Content:
    ```json
    {
      "id": 223190,
      "content": "PACKAGE",
      "contentType": "application/csv",
      "digest": "3FE3742C220EEE991E4501562278A929",
      "status": "ACCEPTED",
      "creator": "user@example.com",
      "created": "2017-05-08T19:45:20.476Z",
      "_links": {
        "self": {
          "href": "http://import-service/imports/223190"
        }
      }
    }
    ```
* Error Response:
  * Code: 401 (Unauthorized) if token is missing or invalid.
  * Code: 403 (Forbidden) if token doesn't provide the `ASTMDATALOAD` role.
* Sample Call (raw body):
```bash
curl -X POST --header "Authorization:Bearer {access_token}" --header "Content-Type:application/xml" \
  --data-binary "AssessmentId,..." https://import-service/packages/imports?filename=2017-2018.csv
```
* Sample Call (form-data):
```bash
curl -X POST --header "Authorization:Bearer {access_token}" -F file=@2017-2018.csv https://import-service/packages/imports
``` 
##### Check Package Import Request result
To check the status of the import use Get Import Request. Imports with "PROCESSED" status will include a messages describing actions taken. An example of the response: 
```
"Assessments processed: 7, created: 5, updated: 1, rejected: 1".
```
              
### Task Endpoints
There are a few tasks that are configured to run periodically (typically once a day). These end-points allow those tasks to be manually triggered on demand. These end-points are part of the actuator framework so they are exposed on a separate port (8008) which is typically kept behind a firewall in a private network. No authentication is required. 

#### Update Organizations Task
The update organization task retrieves all schools from ART and posts them to the import service. This is typically scheduled to happen every day in the wee hours. To trigger an immediate execution: 

* Host: task service
* URL: `/updateOrganizations`
* Method: `POST`
* Success Response:
  * Code: 200
* Sample Call:
```bash
curl -X POST http://localhost:8008/updateOrganizations
```

#### Remove Stale Reports Task
This task removes old user reports. This is typically scheduled to happen once a day in the wee hours, removing any reports that are more than 30 days old. To trigger an immediate execution:

* Host: report processor
* URL: `/removeStaleReports`
* Method: `POST`
* Success Response:
  * Code: 200
* Sample Call:
```bash
curl -X POST http://localhost:8008/removeStaleReports
```

### Status Endpoints
End-points for querying the status of the system. These are intended primarily for operations but can be useful when initially connecting to the system. These end-points are exposed on a separate port (8008) which is typically kept behind a firewall in a private network. No authentication is required.

#### Get Diagnostic Status
This end-point may be used to get the status of a service.

* Host: any RDW service
* URL: `/status`
* Method: `GET`
* URL Params:
  * `level=#` where # can be 0-5; optional, default is `level=0`
* Success Response:
  * Code: 200 (OK)
  * Content: varies based on level; for `level=0`:
    ```json
    {
      "statusText": "Ideal",
      "statusRating": 4,
      "level": 0,
      "dateTime": "2017-05-11T23:26:51.523+0000"
    }
    ```
* Sample Call (curl):
```bash
curl http://localhost:8008/status?level=2
```

#### Actuator Endpoints
As Spring Boot applications there are a number of `actuator` end-points that provide information about the status and configuration of the system. See [Actuator Endpoints][1] for a full (technical) description of these end-points. Like the status end-point these are exposed on a separate port (8008) and do not require authentication. 

A list of the most useful:
* Host: any RDW service
* URL:
  * `/health` - a simple health check
  * `/info` - arbitrary application info
  * `/configprops` - configuration properties
  * `/env` - configurable environment properties
  * `/loggers` - shows and modifies configuration of loggers
  * `/metrics` - application metrics
  * `/trace` - trace information, last 100 HTTP requests
* Sample Call (curl):
```bash
curl http://localhost:8008/configprops
```

[1]: https://docs.spring.io/spring-boot/docs/current/reference/html/production-ready-endpoints.html
[2]: http://www.smarterapp.org/documents/AccessibilityConfigurationFileSpecification.pdf
[3]: http://www.smarterapp.org/documents/TestResultsTransmissionFormat.pdf
