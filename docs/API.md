## API

**Intended Audience**: this document describes the programming interface for the ingest and reporting services of the [Reporting Data Warehouse](../README.md). Operations and system adminstration will find it useful for the configuration data loading end-points and the diagnostic and task service end-points. Third party providers of test results will find it useful for the data loading end-points.

This document describes the service end-points for the ingest and reporting services. *Note: there are many more end-points in the reporting services but most are intended to be consumed by the reporting UI, so are not documented here. Some of those are unofficially documented in [Troubleshooting](Troubleshooting.md#unofficial-api).*
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
1. [Groups Endpoints](#groups-endpoints)
1. [Norms Endpoints](#norms-endpoints)
1. [Task Endpoints](#task-endpoints)
1. [Status Endpoints](#status-endpoints)

### Authentication and Authorization
The import service requires an OAuth2 access token for the data loading end-points. This is a password grant token requested from the OAuth2 server by a trusted client for a user of the system (the permissions are associated with the user).
During the first couple years, the OAuth2 server was an OpenAM server. Recently it was migrated to an Okta server. There are instructions for both. 

#### Fetch Password Grant Access Token - OpenAM
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
* Extracting the access token. A handy util is `jq` which parses json payloads. Using it the access token may be extracted from the payload, something like:
```bash
ACCESS_TOKEN=`curl -s -X POST \
  --data "grant_type=password&username=user@example.com&password=password&client_id=client&client_secret=secret" \
  https://openam/auth/oauth2/access_token?realm=/sbac | jq -r '.access_token'`
```
The resulting environment variable may be used in subsequent calls. Where the samples have `{access_token}` substitute `${ACCESS_TOKEN}`.

#### Fetch Password Grant Access Token - Okta
Accepts x-www-form-urlencoded data including client and user credentials and returns an access token.
* Host: Okta
* URL: `/oauth2/auslw2qcsmsUgzsqr0h7/v1/token` (the exact URL will depend on the installation configuration)
* Method: `POST`
* Data Params (values given are examples and should be replaced with real values):
  * `grant_type=password`
  * `scope=openid profile`
  * `username=user@example.com`
  * `password=UserPassword`
  * `client_id=client`
  * `client_secret=ClientSecret`
* Success Response:
  * Code: 200 (OK)
  * Content: 
    ```json
    {
      "scope": "openid profile",
      "expires_in": 3600,
      "token_type": "Bearer",
      "id_token": "eyJraWQi0...",
      "access_token": "eja8FA..."
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
  --data "grant_type=password&scope=openid profile&username=user@example.com&password=password&client_id=client&client_secret=secret" \
  https://okta/oauth2/auslw2qcsmsUgzsqr0h7/v1/token
```
* Extracting the access token. A handy util is `jq` which parses json payloads. Using it the access token may be extracted from the payload, something like:
```bash
ACCESS_TOKEN=`curl -s -X POST \
  --data "grant_type=password&scope=openid profile&username=user@example.com&password=password&client_id=client&client_secret=secret" \
  https://okta/oauth2/auslw2qcsmsUgzsqr0h7/v1/token | jq -r '.access_token'`
```
The resulting environment variable may be used in subsequent calls. Where the samples have `{access_token}` substitute `${ACCESS_TOKEN}`.

#### Test Access Token - OpenAM
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

#### Test Access Token - Okta
Although not needed during normal operations, this curl call can be used to check an access token:
```bash
  curl -X POST \
  --data 'client_id=<client-id>&client_secret=<client-secret>' \
  'https://smarterbalanced.oktapreview.com/oauth2/auslw2qcsmsUgzsqr0h7/v1/introspect?token=<full-text-of-access-token>&token_type_hint=access_token'
  ```
* Success Response:
  * Code: 200 (OK)
  * Content: 
    ```json
    {
      "active": true,
      "scope": "profile openid",
      "username": "test@example.com",
      "exp": 1566865091,
      "iat": 1566861491,
      "sub": "test@example.com",
      "aud": "api://staging",
      "iss": "https://smarterbalanced.oktapreview.com/oauth2/auslw2qcsmsUgzsqr0h7",
      "token_type": "Bearer",
      "client_id": "<client-id>",
      "uid": "00ukx23x7uJn3O4980h7",
      "sbacTenancyChain": [
        "|CA|ASMTDATALOAD|STATE|1000|ART_DL|||CA|CALIFORNIA|||||||||",
        "|CA|Administrator|STATE|1000|ART_DL|||CA|CALIFORNIA|||||||||",
        "|CA|State Coordinator|STATE|1000|ART_DL|||CA|CALIFORNIA|||||||||"
      ],
      "mail": "test@example.com",
      "sbacUUID": "59946ee4e4b031dfb7d388ca"
    }
    ```
    
* Error Response:
  * Code: 400 (Bad Request)
  * Content (specific error and description will vary):
    ```json
    {
      "error": "invalid_client",
      "error_description": "The client secret supplied for a confidential client is invalid."
    }
    ```

### Import Endpoints
The import service provides the end-points for submitting data to the system. All end-points require a valid token, the examples use `{access_token}` as a placeholder. The token must provide appropriate permissions. For most content, this is the `ASMTDATALOAD` role at the client or state level but refer to individual content end-point documentation for specific permissions. There is one diagnostic end-point provided that helps debug permissions issues:

#### Get User
This end-point may be used to get the credentials of the current user. This is provided for diagnostic purposes.

* Host: import service
* URL: `/imports/user`
* Method: `GET`
* Params: none
* Headers:
  * `Authorization: Bearer {access_token}`
* Success Response:
  * Code: 200 (OK)
  * Content:
    ```json
    {
      "password": "N/A",
      "username": "dwtest@example.com",
      "authorities": [
        {
          "authority": "ROLE_ASMTDATALOAD"
        }
      ],
      "accountNonExpired": true,
      "accountNonLocked": true,
      "credentialsNonExpired": true,
      "enabled": true,
      "tenancyChain": {
        "grants": [
          {
            "level": "CLIENT",
            "name": "",
            "id": "SBAC",
            "role": "ASMTDATALOAD",
            "clientId": "SBAC",
            "stateId": "",
            "entityId": "SBAC"
          }
        ],
      "empty": false
      }
    }
    ```
* Error Response:
  * Code: 401 (Unauthorized) if token is missing or invalid.
  * Code: 403 (Forbidden) if token doesn't provide RDW authentication.
* Sample Call (curl):
```bash
curl --header "Authorization:Bearer {access_token}" https://import-service/imports/user
```

The import requests are processed and migrated to the reporting data mart. Import payloads are hashed and duplicate content is detected, returning any previous import request for the given content. Thus, for most content, submitting a payload a second time will safely no-op and return the current status of the previous import. The one exception to this rule is for Student Groups: because other factors such as permissions and organizations affect the processing of groups, duplicate import requests are reprocessed.

All data submissions result in an import being created. Thus a POST to `/exams/imports` will create an `import` resource which can be accessed at `/imports/{id}` (note that `exams` is not in the path). These are the end-points for querying imports.
 
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

These require the `ASMTDATALOAD` role.

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
curl -X POST --header "Authorization:Bearer {access_token}" -F file=@winterICA.1.xml -F batch=WinterICA https://import-service/exams/imports
```
  
#### Resubmit Exams
The system accepts and stores import payloads regardless of processing status. For imports with certain problems like an unknown school or assessment, the import may be "replayed" once the underlying cause is resolved. For example, if the missing school is added to the system, all affected exams can be resubmitted, without requiring the client to re-post the data. Note that this is meant as an operational utility; if it is being used regularly there are probably fundamental data flow issues. 

* Host: import service
* URL: `/exams/imports/resubmit`
* Method: `POST`
* URL Params: query params can be used to restrict which exam imports to include; typically the status
  * `status=<status>` where status may be the import status name, e.g. `BAD_DATA`, or numeric id, e.g. `-3` (query the `import_status` table for allowed values). 
  * `after=<interval>` where interval is like `-PT1H` 
  * `before=<interval>` where interval is like `-PT1H` 
  * `creator=<creator>`
  * `batch=<batchtag>`
  * `limit=<N>` where N is a count. If there are a lot of imports to resubmit, use a limit to run them in manageable chunks, e.g. `limit=100`.
* Headers:
  * `Authorization: Bearer {access_token}`
* Success Response:
  * Code: 200
  * Content: number of exams found and resubmitted. If the return value is the same as the `limit` param, it is likely there are more exams matching the query and additional resubmit calls should be made. 
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

These require the `ASMTDATALOAD` role.

#### Create Organization Import Request
Accepts JSON payload, compatible with the [format produced by ART](https://github.com/SmarterApp/TDS_AdministrationAndRegistrationTools/blob/master/external_release_docs/SBAC-11%20TestRegistration%20API.pdf),
or CALPADS format. 

The payload should contain all the organization data necessary to resolve all contents; for example, if a school is in a group under a district, all three must be present in the payload. The payload must be valid JSON but the exact structure doesn't matter a lot: the system will parse the payload looking for the required fields: `entityType`, `entityId`, `entityName`, `parentEntityId`. 

There are two ways of posting content: with a raw body of type `application/json` or `text/csv` or form-data (file upload).

* Host: import service
* URL: `/organizations/imports`
* Method: `POST`
* URL Params: any URL param will be preserved as properties for the upload, well-known params:
  * `filename=<originalFileName>`  (not needed for form-data)
* Headers:
  * `Authorization: Bearer {access_token}`
* Headers (raw body):
  * `Content-Type:application/json` (JSON)
  * `Content-Type:text/csv` (CALPADS)
* Body (raw body): JSON payload, contrived example:
```json
{
  "groupofdistricts": [
    {
      "entityId": "8880001",
      "entityType": "GROUPOFDISTRICTS",
      "parentEntityType": "STATE",
      "entityName": "Pern North",
      "parentEntityId": "CA"
    }
  ],
  "districts": [
    {
      "entityId": "88800120000000",
      "entityType": "DISTRICT",
      "parentEntityType": "GROUPOFDISTRICTS",
      "entityName": "Igen District",
      "parentEntityId": "8880001"
    },
    {
      "entityId": "88800130000000",
      "entityType": "DISTRICT",
      "parentEntityType": "STATE",
      "entityName": "Crom District",
      "parentEntityId": "CA"
    }
  ], 
  "institutions": [
    {
      "entityId": "88800120012001",
      "entityType": "INSTITUTION", 
      "parentEntityType": "DISTRICT", 
      "entityName": "Big Bay",
      "parentEntityId": "88800120000000"
    }, 
    {
      "entityId": "88800130013001",
      "entityType": "INSTITUTION", 
      "parentEntityType": "DISTRICT", 
      "entityName": "Camp Natalon",
      "parentEntityId": "88800130000000"
    }, 
    {
      "entityId": "88800120012002",
      "entityType": "INSTITUTION", 
      "parentEntityType": "DISTRICT", 
      "entityName": "Igen Hold",
      "parentEntityId": "88800120000000"
    }, 
    {
      "entityId": "88800130013002",
      "entityType": "INSTITUTION", 
      "parentEntityType": "DISTRICT", 
      "entityName": "Crom Hold",
      "parentEntityId": "88800130000000"
    }, 
    {
      "entityId": "88800120012003",
      "entityType": "INSTITUTION", 
      "parentEntityType": "DISTRICT", 
      "entityName": "Katz Field",
      "parentEntityId": "88800120000000"
    }, 
    {
      "entityId": "88800130013003",
      "entityType": "INSTITUTION", 
      "parentEntityType": "DISTRICT", 
      "entityName": "Greenfields",
      "parentEntityId": "88800130000000"
    }, 
    {
      "entityId": "88800130013004",
      "entityType": "INSTITUTION", 
      "parentEntityType": "DISTRICT", 
      "entityName": "Keogh",
      "parentEntityId": "88800130000000"
    }, 
    {
      "entityId": "88800120012004",
      "entityType": "INSTITUTION", 
      "parentEntityType": "DISTRICT", 
      "entityName": "Tannercraft Hall",
      "parentEntityId": "88800120000000"
    }, 
    {
      "entityId": "88800130013005",
      "entityType": "INSTITUTION", 
      "parentEntityType": "DISTRICT", 
      "entityName": "Three Rivers",
      "parentEntityId": "88800130000000"
    }
  ]
}
```
* Body (raw body): CALPADS payload, same contrived example:
```text
County-District Code^School Code^Auth CDS Code^County Name^District Name^School Name^Charter School^Charter Status^NPS School
8880012^0012001^88800120012001^Pern^Igen District^Big Bay^N^^N
8880012^0012002^88800120012002^Pern^Igen District^Igen Hold^N^^N
8880012^0012003^88800120012003^Pern^Igen District^Katz Field^N^^N
8880012^0012004^88800120012004^Pern^Igen District^Tannercraft Hall^N^^N
8880013^0013001^88800130013001^Pern^Crom District^Camp Natalon^N^^N
8880013^0013002^88800130013002^Pern^Crom District^Crom Hold^N^^N
8880013^0013003^88800130013003^Pern^Crom District^Greenfields^N^^N
8880013^0013004^88800130013004^Pern^Crom District^Keogh^N^^N
8880013^0013005^88800130013005^Pern^Crom District^Three Rivers^N^^N
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

These require the `ASMTDATALOAD` role.

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
<Accessibility schoolYear="2018">
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

These require the `ASMTDATALOAD` role.

#### Create Package Import Request
Accepts payloads in the Smarter Balanced Assessment Tabulator format. This is a CSV format produced by the internal tabulator utility. 

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
AssessmentId,AssessmentName,AssessmentSubject,AssessmentGrade,AssessmentType,AssessmentSubtype,AssessmentLabel,AssessmentVersion,AcademicYear,FullItemKey,BankKey,ItemId,Filename,Version,AnswerKey,NumberOfAnswerOptions,ItemType,Grade,Standard,Claim,Target,ClaimContentTarget,SecondaryClaimContentTarget,CommonCore,SecondaryCommonCore,ASL,Braille,LanguageBraille,DOK,Language,AllowCalculator,MathematicalPractice,MaxPoints,Glossary,ScoringEngine,Spanish,IsFieldTest,IsActive,ResponseRequired,AdminRequired,ItemPosition,MeasurementModel,Weight,ScorePoints,a,b0_b,b1_c,b2,b3,avg_b,bpref1,bpref2,bpref3,bpref4,bpref5,bpref6,bpref7,CutPoint1,ScaledLow1,ScaledHigh1,CutPoint2,ScaledLow2,ScaledHigh2,CutPoint3,ScaledLow3,ScaledHigh3,CutPoint4,ScaledLow4,ScaledHigh4,PtWritingType
(SBAC)SBAC-IAB-FIXED-G11E-BriefWrites-ELA-11-Winter-2016-2017,SBAC-IAB-FIXED-G11E-BriefWrites-ELA-11,ELA,11,interim,IAB,ELA IAB G11 BriefWrites,9832,2017,200-58197,200,58197,item-200-58197.xml,9832,Hand,1,SA,11,SBAC-ELA-v1:2-W|1-11,"2-W	","1-11	","2-W	|1-11	","2-W	|1-11	;2-W |3-11 ",5.RI.10;8.L.5b,5.RI.10;8.L.5b,N,BRF,ENU-Braille,3,ENU,,,2,,HandScored,N,FALSE,TRUE,TRUE,TRUE,1,IRTGPC,1,2,0.55434,0.93104,0.43826,,,0.68465,(SBAC)SBAC-IAB-FIXED-G11E-BriefWrites-ELA-11-Winter-2016-2017,,SBAC-2-W|1-11,,,,,1,2.30E+03,,2,2.49E+03,2.58E+03,3,2.58E+03,,4,2.68E+03,2.80E+03,Argumentative
(SBAC)SBAC-IAB-FIXED-G11E-BriefWrites-ELA-11-Winter-2016-2017,SBAC-IAB-FIXED-G11E-BriefWrites-ELA-11,ELA,11,interim,IAB,ELA IAB G11 BriefWrites,9832,2017,200-59189,200,59189,item-200-59189.xml,9832,Hand,2,SA,11,SBAC-ELA-v1:2-W|1-11,"2-W	","1-11	","2-W	|1-11	","2-W	|1-11	",8.L.5b,8.L.5b;8.L.5b,N,BRF,ENU-Braille,3,ENU,,,2,,HandScored,N,FALSE,TRUE,TRUE,TRUE,2,IRTGPC,1,2,0.51807,0.66337,3.26407,,,1.96372,(SBAC)SBAC-IAB-FIXED-G11E-BriefWrites-ELA-11-Winter-2016-2017,,SBAC-2-W|1-11,,,,,1,2.30E+03,,2,2.49E+03,2.58E+03,3,2.58E+03,,4,2.68E+03,2.80E+03,Explanatory
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
curl -X POST --header "Authorization:Bearer {access_token}" --header "Content-Type:application/csv" \
  --data-binary "AssessmentId,..." https://import-service/packages/imports?filename=2017-2018.csv
```
* Sample Call (form-data):
```bash
curl -X POST --header "Authorization:Bearer {access_token}" -F "file=@2017-2018.csv;type=text/csv" https://import-service/packages/imports
``` 
##### Check Package Import Request result
To check the status of the import use Get Import Request. Imports with "PROCESSED" status will include a messages describing actions taken. An example of the response: 
```
"Assessments processed: 7, created: 5, updated: 1, rejected: 1".
```
              
### Groups Endpoints
End-point for submitting student group data in the Smarter Balanced Student Group CSV format.

These require the `GROUP_ADMIN` permission for the schools involved.

#### Create Groups Import Request
Accepts payloads in the Smarter Balanced Student Group CSV format. This end point can be used to import new groups or update existing ones.

There are two ways of posting content: with a raw body of type `application/csv` or form-data file upload in CSV format.

* Host: import service
* URL: `/groups/imports`
* Method: `POST`
* URL Params: any URL param will be preserved as properties for the upload, well-known params:
  * `filename=<originalFileName>`  (not needed for form-data)
* Headers:
  * `Authorization: Bearer {access_token}`
* Headers (raw body):
  * `Content-Type:application/csv`
* Body (raw body): Smarter Balanced Student Group CSV
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
        "id": 9,
        "content": "GROUPS",
        "contentType": "application/csv",
        "digest": "5EA9607434903493BD274611FA817304",
        "status": "PROCESSED",
        "creator": "dwtest@example.com",
        "created": "2018-02-08T22:29:47.120343Z",
        "updated": "2018-02-08T22:29:47.120343Z",
        "_links": {
            "self": {
                "href": "http://localhost:8080/imports/9"
            },
            "payload": {
                "href": "http://localhost:8080/imports/9/payload"
            },
            "payload-properties": {
                "href": "http://localhost:8080/imports/9/payload/properties"
            }
        }
    }
    ```
* Error Response:
  * Code: 401 (Unauthorized) if token is missing or invalid.
  * Code: 403 (Forbidden) if token doesn't provide the `GROUP_WRITE` permission.
* Sample Call (raw body):
```bash
curl -X POST --header "Authorization:Bearer {access_token}" --header "Content-Type:application/csv" \
  --data-binary "group_name,school_natural_id,..." https://import-service/groups/imports?filename=mygroups.csv
```
* Sample Call (form-data):
```bash
curl -X POST --header "Authorization:Bearer {access_token}" -F "file=@mygroups.csv;type=text/csv" https://import-service/groups/imports
``` 

### Norms Endpoints
End-point for submitting norms percentile tables in the Smarter Balanced [Norms CSV format](Norms.md). This end point can be used to import new norms or update existing ones. When norms are created, the following data elements are required and comprise the unique identifier for the norms percentile table:
* assessment_id : natural id of an assessment that must already be loaded
* start_date : inclusive start date of exam completion
* end_date : inclusive end date of exam completion

When an existing norms percentile table matches the above three data elements on import it is updated.

These require the `ASMTDATALOAD` role.

#### Create Norms Import Request
Accepts payloads in the Smarter Balanced [Norms CSV format](Norms.md).

There are two ways of posting content: with a raw body of type `application/csv` or form-data file upload in CSV format.

* Host: import service
* URL: `/norms/imports`
* Method: `POST`
* URL Params: any URL param will be preserved as properties for the upload, well-known params:
  * `filename=<originalFileName>`  (not needed for form-data)
* Headers:
  * `Authorization: Bearer {access_token}`
* Headers (raw body):
  * `Content-Type:application/csv`
* Body (raw body): Smarter Balanced [Norms CSV format](Norms.md)
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
        "id": 9,
        "content": "NORMS",
        "contentType": "application/csv",
        "digest": "5EA9607434903493BD274611FA817304",
        "status": "PROCESSED",
        "creator": "dwtest@example.com",
        "created": "2018-02-08T22:29:47.120343Z",
        "updated": "2018-02-08T22:29:47.120343Z",
        "message": "Percentiles created: 0 updated: 2",
        "_links": {
            "self": {
                "href": "http://localhost:8080/imports/9"
            },
            "payload": {
                "href": "http://localhost:8080/imports/9/payload"
            },
            "payload-properties": {
                "href": "http://localhost:8080/imports/9/payload/properties"
            }
        }
    }
    ```
* Error Response:
  * Code: 401 (Unauthorized) if token is missing or invalid.
  * Code: 403 (Forbidden) if token doesn't provide the `ASTMDATALOAD` role.
* Sample Call (raw body):
```bash
curl -X POST --header "Authorization:Bearer {access_token}" --header "Content-Type:application/csv" \
  --data-binary "assessment_id,start_date,..." https://import-service/norms/imports?filename=2017-norms.csv
```
* Sample Call (form-data):
```bash
curl -X POST --header "Authorization:Bearer {access_token}" -F "file=@2017-norms.csv;type=text/csv" https://import-service/norms/imports
```
##### Check Norms Import Request result
To check the status of the import use Get Import Request. Imports with "PROCESSED" status will include a messages describing actions taken. An example of the response:
```
"Percentiles created: 0 updated: 2".
```

### Task Endpoints
There are a few tasks that are configured to run periodically (typically once a day). These end-points allow those tasks to be manually triggered on demand. These end-points are part of the actuator framework so they are exposed on a separate port (8008) which is typically kept behind a firewall in a private network. No authentication is required. 

#### Reconciliation Report Task
The reconciliation report task generates a report of all exam imports in the given time range, writing it to the specified location (usually an S3 bucket). This is typically scheduled to happen every day but may be less often in smaller installations (or not at all). To trigger an immediate execution: 

* Host: task service
* URL: `/reconciliationReport`
* Method: `POST`
* URL Params: a tenant id may be specified to restrict the task to just that tenant
  * `tenantId=<tenant-id>`
* Success Response:
  * Code: 200
* Sample Call:
```bash
curl -X POST http://localhost:8008/reconciliationReport
```

#### Update Organizations Task
The update organization task retrieves all schools from ART and posts them to the import service. This is typically scheduled to happen every day in the wee hours. To trigger an immediate execution: 

* Host: task service
* URL: `/updateOrganizations`
* Method: `POST`
* URL Params: a tenant id may be specified to restrict the task to just that tenant
  * `tenantId=<tenant-id>`
* Success Response:
  * Code: 200
* Sample Call:
```bash
curl -X POST http://localhost:8008/updateOrganizations
```

#### Migrate Task
The migrate tasks move data from the main data warehouse to the reporting data stores. If there are problems the migrate tasks will disable themselves by marking a record in the database. This is tenant-specific. To re-enable migrate, DevOps needs to diagnose the problem and then update that record in a specific way. To help with this, there is an actuator end-point to update the record properly. NOTE: if the underlying problem is not addressed, the migrate service will disable itself again.

* Host: migrate-olap service, migrate-reporting service
* URL: `/migrate/enable`
* Method: `POST`
* URL Params: the tenant id must be specified:
  * `tenantId=<tenant-id>`
* Success Response:
  * Code: 200
* Sample Call:
```bash
curl -X POST http://localhost:8008/migrate/enable?tenantId=CA
```

As a reminder, the migrate services also allow themselves to be paused. This is different than when they disable themselves. The services must be both running and enabled to be active. Pausing uses the normal Spring lifecycle end-points.

* Host: migrate-olap service, migrate-reporting service
* URL: `pause`, `resume`
* Method: `POST`
* Success Response:
  * Code: 200
* Sample Call:
```bash
curl -X POST http://localhost:8008/pause
curl -X POST http://localhost:8008/resume
```

The task for moving data to the individual reporting system used by teachers happens very frequently (typically every minute). It is unlikely there will ever be a need to trigger an immediate migrate. However the task for moving data to the aggregate reporting system is less frequent (typically once a day). As such it may be useful to trigger this migration when initially setting up or testing the system. NOTE: the migrate-olap service must be running and be enabled for this to work.

* Host: migrate-olap service
* URL: `/migrate`
* Method: `POST`
* URL Params: there is an optional tenantId parameter; if specified just that tenant is migrated, otherwise migrate will be triggered for all tenants:
  * `tenantId=<tenant-id>`
* Success Response:
  * Code: 200
* Sample Call:
```bash
curl -X POST http://localhost:8008/migrate
curl -X POST http://localhost:8008/migrate?tenantId=CA
```

As a convenience the /migrate end-point will return the current migrate status. This can be useful when troubleshooting. 
NOTE: the `COMPLETED to <timestamp>` is the timestamp of the last *record* migrated, not when the last migrate occurred.

* Host: migrate-olap service, migrate-reporting service
* URL: `/migrate`
* Method: `GET`
* Success Response:
  * Code: 200
  * Content:
  ```
  Migrate: running
  CA: enabled; last migrate: COMPLETED to 2019-05-28T15:26:55.73839
  TS: enabled; last migrate: COMPLETED to 2019-05-28T15:17:17.488472  
  ```
* Sample Call:
```bash
curl http://localhost:8008/migrate
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
