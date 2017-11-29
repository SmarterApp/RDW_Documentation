## API - Reporting

**Intended Audience**: this document describes the programming interface for the reporting services of the [Reporting Data Warehouse](../README.md). Operations and system adminstration will find it useful for the diagnostic and task end-points.

This document describes the service end-points for the reporting services.
* The report processor has a task trigger end-point. 
* All services have diagnostic end-points. 

> There are many more end-points in the reporting services. Most are intended to be consumed by the reporting UI, so are not documented here. 
        
### Task Endpoints
The report processor has a task configured to run periodically (typically once a day). This end-point is part of the actuator framework so it is exposed on a separate port (8008) which is typically kept behind a firewall in a private network. No authentication is required.

#### Remove Stale Reports Task
This task removes old user reports. This is typically scheduled to happen once a day in the wee hours, removing any reports that are more than 30 days old.

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
As Spring Boot applications there are a number of `actuator` end-points that provide information about the status and configuration of the system. See [Actuator Endpoints][1] for a full (technical) description of these end-points. Like the status end-point these are exposed on a separate port (8008) and do not require authentication. A list of the most useful:

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
