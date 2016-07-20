# Event-Audit-Trail

The Predix Event Audit Trail Service is an data persistence service that represents user-defined data and states as events stored in a chronological manner.  The service provides a simple and scalable way to provide an audit trail for user applications.  The Predix Event Audit Trail is available as a service tile in the Predix Cloud and allows users to create, search, update, and delete events through a REST interface. 

## Table of Contents
  - [Usage](#usage)
    - [Creating service instance](#creating-service-instance)
    - [Auto archive](#auto-archive)
    - [UAA configuration](#uaa-configuration)
        - [Client credentials](#client-credentials)
        - [User login](#user-login)
  - [Event service](#event-service)
    - [Get events with filters](#get-events-with-filters)
    - [Get event by uuid](#get-event-by-uuid)
    - [Add events](#add-events)
    - [Update events](#update-events)
    - [Delete event](#delete-event)
    - [Search events](#search-events)
    - [Get Tenant](#get-tenant)
  - [Archive service](#archive-service)
    - [Get archive configuration](#get-archive-configuration)
    - [Sets or updates the archive configuration](#sets-or-updates-the-archive-configuration)
    - [Get archive list](#get-archive-list)
    - [Get staged events](#get-staged-events)
    - [Get archive by uuid](#get-archive-by-uuid)
  - [Response codes](#response-codes)
  - [Resources](#resources)

## Usage

### Creating service instance


To create an instance of the Predix Event Audit Trail Service, please use the [Cloud Foundry CLI tool](https://github.com/cloudfoundry/cli) to run the following command:

```
cf create-service predix-event-audit-trail <plan> <service_instance_name> -c '{“trustedIssuerId”: “<predix uaa instance url>/oauth/token”}'
```

Once a service instance is created , you will be issued a tenant uuid for the created service instance.  You can obtain this tenant uuid by binding an application to
the service instance.  You can bind an application to the service instance with the following command:

```
cf bind-service <app_name> <service_instance_name>
```

The application's *VCAP SERVICES* will contain the following variables:

```
"predix-event-audit-trail": [
   {
    "credentials": {
     "catalog-uri": "<service url>",
     "tenant-uuid": "<tenant uuid>",
     "version": "1",
     "trusted-issuer-ids": "<trusted issuers>",
     "zone-oauth-scope": "event-audit-trail.zone.<tenant uuid>.user"
    },
    "label": "predix-event-audit-trail",
    "name": "<service_instance_name>",
    "plan": "<plan>",
    "provider": null,
    "syslog_drain_url": null,
    "tags": [
     "audit",
     "event",
     "trail"
    ]
   }
```

You can then use the [service APIs](#event-apis) with using the **<service_url>** as the base url.

**Note** - Please update your [Cloud Foundry CLI tool](https://github.com/cloudfoundry/cli) to the lastest version.

#### UAA configuration

Once a tenant uuid has been issued to your instance you will need to add the **zone-oath-scope** to your UAA instance clients.

##### Client credentials 

To use the event audit trail service with the client credentials flow (the SDK requires this), then you need to add the **zone-oath-scope** to your client authorities.  You can do this using the UAA cli with the following command:

```
uaac client update <client_name> --authorities "<list_of_authorities> event-audit-trail.zone.<tenant uuid>.user"
```

Tokens issue by this client will now be authorized to use the event audit trail service instance.

##### User login

To use the event audit service with a user-authenticated flow (e.g. predix-seed), you will need to add a group with the proper scope using the UAA cli.

```
uaac client update <client_name> --scope "<list_of_scope> event-audit-trail.zone.<tenant uuid>.user"

uaac group add event-audit-trail.zone.<tenant uuid>.user

uaac member add event-audit-trail.zone.<tenant uuid>.user <users...>
```

Now tokens issued to this user will be authorized to use the event audit trail service instance.

### Auto archive

The Event Audit Trail Service has built-in archiving feature.  To use this feature, please configure the event limit using the [Sets archive configuration api](#sets-or-updates-the-archive-configuration).

Events that are marked for archival will be disabled and moved to a "staging" area.  These events will not be fully archived until there are 50 events in the "staging" area.  You can retrieve "staged" events using the [Get staged events](#get-staged-events) API.  Once events have been fully archived they must be retrieved using the [Get archive by uuid](#get-archive-by-uuid) API.  

Archived and staged events will not be returned by the service using the standard get event API.  Please note that archived and staged events still count against the total event usage for billing.
  
## Event service

### Get events by tenant with filters
* GET /tenants/{tenant_uuid}/events?context=<context>&tag=<tag>&classification=<classification>&start_date=<start_date>&end_date=<end_date>
    * parameters
        * tenant_uuid - uuid of tenant
    * filters
        * context - events by context
        * tag - events by tag
        * classification - filters events by classification
        * start_date - filters events by start_date in the format: **yyyy-M-d h:m:s**, inclusive. If a start_date is supplied, an end_date must also be supplied.
        * end_date - filters events by end_date in the format: **yyyy-M-d h:m:s**, exclusive. If a end_date is supplied, an start_date must also be supplied.

* Sample Response Payload
```
  "payload": [
    {
      "id": 12,
      "uuid": "0730853b-6a5b-4894-b6da-e7e88833f1e9",
      "tenantUuid": null,
      "context": "user test",
      "tag": "group2",
      "classification": 0,
      "enabled": true,
      "timestamp": 1459378121815,
      "data": "{\"name\": \"test1.csv\"}"
    },
    {
          "id": 12,
          "uuid": "0730853b-6a5b-4894-b6da-e7e88833f1e9",
          "tenantUuid": null,
          "context": "user test",
          "tag": "group2",
          "classification": 0,
          "enabled": true,
          "timestamp": 1459378121815,
          "lastUpdated": 1459378121815,
          "data": "{\"name\": \"test2.csv\"}"
     }
  ],
  "uuid": "ddc7f50a-6883-45aa-a823-b70e47d2a05c",
  "status": 1000,
  "message": "OK",
  "timestamp": 1459470908881
```

### Get event by uuid
* GET tenant/{tenant_uuid}/events/{uuid}
    * parameters
        * tenant_uuid - uuid of tenant
        * uuid - uuid of the event (REQUIRED)

* Sample Response Payload
```
  "payload": [
    {
      "id": 12,
      "uuid": "0730853b-6a5b-4894-b6da-e7e88833f1e9",
      "tenantUuid": null,
      "context": "user test",
      "tag": "group2",
      "classification": 0,
      "enabled": true,
      "timestamp": 1459378121815,
      "lastUpdated": 1459378121815,
      "data": "{\"name\": \"test1.csv\"}"
    },
  ],
  "uuid": "ddc7f50a-6883-45aa-a823-b70e47d2a05c",
  "status": 1000,
  "message": "OK",
  "timestamp": 1459470908881
```

### Add events
* POST tenants/{tenant_uuid}/events
    * Request Body JSON parameters
        * tenantUuid - uuid of tenant, this is the primary method of tag events (REQUIRED)
        * context - user-defined context (OPTIONAL)
        * tag - user-defined tag (OPTIONAL)
        * classification - user-defined classification (OPTIONAL)
        * timestamp - if no timestamp is given, this will be set to the current time (OPTIONAL)
        * data - user-defined JSON data field (OPTIONAL)

* Sample Request
```
[
    {
      "context": "user test",
      "tag": "tag1",
      "classification": 0,
      "data": "{\"name\": \"event1\",\"type\": \"request\",\"params\": \"test\"}"
    }
  ]
```

### Update event
* PUT tenants/{tenant_uuid}/events/{uuid}
    * URI path parameters
        * tenant_uuid - uuid of tenant
        * uuid - uuid of the event (REQUIRED)
    * Request Body JSON parameters - Event details will be over-ridden.
         * context - user-defined context (OPTIONAL)
         * tenantUuid - uuid of tenant (OPTIONAL)
         * tag - user-defined tag (OPTIONAL)
         * classification - user-defined classification (OPTIONAL)
         * timestamp - if no timestamp is given, this will be set to the current time (OPTIONAL)
         * data - user-defined JSON data field (OPTIONAL)

* Sample Response Payload
```
  "payload": [
    {
      "id": 12,
      "uuid": "0730853b-6a5b-4894-b6da-e7e88833f1e9",
      "tenantUuid": null,
      "context": "user test",
      "tag": "group2",
      "classification": 0,
      "enabled": true,
      "timestamp": 1459378121815,
      "lastUpdated": 1459378121815,
      "data": "{\"name\": \"test1.csv\"}"
    },
  ],
  "uuid": "ddc7f50a-6883-45aa-a823-b70e47d2a05c",
  "status": 1000,
  "message": "OK",
  "timestamp": 1459470908881
```

### Delete event
* DELETE tenants/{tenant_uuid}/events/{uuid}
    * parameters
        * tenant_uuid - uuid of tenant
        * uuid - uuid of the event (REQUIRED)

* Sample Response Payload
```
  "payload": [
    {
      "id": 12,
      "uuid": "0730853b-6a5b-4894-b6da-e7e88833f1e9",
      "tenantUuid": null,
      "context": "user test",
      "tag": "group2",
      "classification": 0,
      "enabled": true,
      "timestamp": 1459378121815,
      "lastUpdated": 1459378121815,
      "data": "{\"name\": \"test1.csv\"}"
    },
  ],
  "uuid": "ddc7f50a-6883-45aa-a823-b70e47d2a05c",
  "status": 1000,
  "message": "OK",
  "timestamp": 1459470908881
```

### Search events
* GET tenants/{tenant_uuid}/event-search?query=<search_query>
    * parameters
        * tenant_uuid - uuid of tenant 
    * filter
        * query - search query    
    
* Search Terms Example
```
* query=NOT removed - Get all the events audit-trails if event doesn't contains "removed".
* query=added - Get all the events audit-trails if event contains "added".
* query=added OR removed - Get all the events audit-trails if event contains "added" or "removed".
* query=added AND removed - Get all the events audit-trails if event contains "added" and "removed".
```

* Sample Response Payload
```
  "payload": [
    {
      "id": 12,
      "uuid": "0730853b-6a5b-4894-b6da-e7e88833f1e9",
      "tenantUuid": null,
      "context": "user test",
      "tag": "group2",
      "classification": 0,
      "enabled": true,
      "timestamp": 1459378121815,
      "lastUpdated": 1459378121815,
      "data": "{\"name\": \"test1.csv\"}"
    },
  ],
  "uuid": "ddc7f50a-6883-45aa-a823-b70e47d2a05c",
  "status": 1000,
  "message": "OK",
  "timestamp": 1459470908881
```

### Get Tenant
* GET tenant/{tenant_uuid}
    * parameters
        * tenant_uuid - uuid of tenant to retrieve

* Sample Response Payload
```
   "payload": {
     "id": 1231,
     "bindingId": "2d5a2ef5-898f-4f3d-80fd-d9266992f99f",
     "uuid": "e314ec8d9-66dc-4782-91d6-634551f492c5",
     "timestamp": 1463508805471,
     "eventCount": 250,
     "trustedIssuers": "https://89cd1af4-be74-4c39-9282-8fef95011afa.predix-uaa.run.aws-usw02-pr.ice.predix.io/oauth/token\n"
   },
   "uuid": "cf6321bc-4be8-48bd-8058-71602fc78f63",
   "status": 1000,
   "message": "OK",
   "timestamp": 1466015044971
```

## Archive service

### Get archive configuration
* GET archive/tenant/{tenant_uuid}/configuration
    * parameters
        * tenant_uuid - tenant uuid (primary event grouping)

* Sample Response Payload
```
  "payload": {
    "uuid": "77173278-2ba0-457b-b660-09f9a9c87855",
    "tenantUuid": "4b2b0ad4-c54b-4e69-90b1-12d73e36662c",
    "maximumNumberOfEvents": 5,
    "maximumNumberOfStoredEventsDays": -1
  }
```

### Sets or updates the archive configuration
* POST archive/tenant/{tenant_uuid}/configuration
    * parameters
        * Request Body JSON parameters
            * maximumNumberOfEvents - Maximum number of events to retain (Default=-1)
            * maximumNumberOfStoredEventsDays - Maximum numbers of days of events to retain (Default=-1)

* Sample Request
```
{
  "maximumNumberOfEvents": 100,
  "maximumNumberOfStoredEventsDays": 30
}
```

### Get archive list
* GET archive/tenant/{tenant_uuid}/archives
    * parameters
            * tenant_uuid - tenant uuid (primary event grouping)

* Sample Response Payload
```
"payload" : [
    {
      "id": 11,
      "uuid": "e006f214-88a6-424b-9a7d-f43180aba455",
      "tenantUuid": "4b2b0ad4-c54b-4e69-90b1-12d73e36662c",
      "fromDate": 1461800406677,
      "toDate": 1461865172861,
      "size": 20
    }]
```

### Get staged events
* PUT archive/tenant/{tenant_uuid}/staged-events
    * URI path parameters
        * tenant_uuid - tenant uuid (primary event grouping)
* Sample Response Payload
```
  "payload": [
    {
      "id": 12,
      "uuid": "0730853b-6a5b-4894-b6da-e7e88833f1e9",
      "tenantUuid": null,
      "context": "user test",
      "tag": "group2",
      "classification": 0,
      "enabled": true,
      "timestamp": 1459378121815,
      "lastUpdated": 1459378121815,
      "data": "{\"name\": \"test1.csv\"}"
    },
  ],
  "uuid": "ddc7f50a-6883-45aa-a823-b70e47d2a05c",
  "status": 1000,
  "message": "OK",
  "timestamp": 1459470908881
```

### Get archive by uuid
* PUT archive/tenant/{tenant_uuid}/archives/{uuid}
    * URI path parameters
        * tenant_uuid - tenant uuid (primary event grouping)
        * uuid - uuid of the archive (REQUIRED)
* Sample Response Payload
```
  "payload": [
    {
      "id": 12,
      "uuid": "0730853b-6a5b-4894-b6da-e7e88833f1e9",
      "tenantUuid": null,
      "context": "user test",
      "tag": "group2",
      "classification": 0,
      "enabled": true,
      "timestamp": 1459378121815,
      "lastUpdated": 1459378121815,
      "data": "{\"name\": \"test1.csv\"}"
    },
  ],
  "uuid": "ddc7f50a-6883-45aa-a823-b70e47d2a05c",
  "status": 1000,
  "message": "OK",
  "timestamp": 1459470908881
```

## Response codes
These are the service-specific responses embedded in service responses

* **1000** - OK
* **1001** - Event does not exist
* **1002** - Event Owner does not exist
* **1003** - Event Tenant does not exist
* **1004** - Event Owner is missing or blank
* **1005** - Encountered exception adding events
* **1006** - Tenant ID mismatch
* **1007** - Tenant is missing or blank
* **1008** - Search query is missing or blank
* **1009** - Invalid search query string
* **1101** - The archive does not exist


## Resources

* [Event Audit Trail SDK](https://github.build.ge.com/emerging-verticals/event-audit-trail-sdk) - The accompanying SDK for the event audit-trail.

* [Event Audit Trail Sample Application](https://github.build.ge.com/emerging-verticals/event-audit-trail-sample) - A sample application which utilizes the Event Audit Trail Service


