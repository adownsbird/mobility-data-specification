# Mobility Data Specification: **Provider**

This specification contains a data standard for *mobility as a service* providers to define a RESTful API for municipalities to access on-demand.

## Table of Contents

* [General Information](#general-information)
* [Trips](#trips)
* [Status Changes](#status-changes)

## General Information

The following information applies to all `provider` API endpoints.

### Response Format

Responses must be `UTF-8` encoded `application/json` and must minimally include the MDS `version` and a `data` payload:

```js
{
    "version": "0.1.0",
    "data": {
        "trips": [{
            "provider_id": "...",
            "trip_id": "...",
            //etc.
        }]
    }
}
```

All response fields must use `lower_case_with_underscores`.

### Pagination

`provider` APIs may decide to paginate the data payload. If so, pagination must comply with the [JSON API](http://jsonapi.org/format/#fetching-pagination) specification.

The following keys must be used for pagination links:

 - `first`: url to the first page of data
 - `last`: url to the last page of data
 - `prev`: url to the previous page of data
 - `next`: url to the next page of data

```js
{
    "version": "0.1.0",
    "data": {
        "trips": [{
            "provider_id": "...",
            "trip_id": "...",
            //etc.
        }]
    },
    "links": {
        "first": "https://...",
        "last": "https://...",
        "prev": "https://...",
        "next": "https://..."
    }
}
```

### UUIDs for Devices

**MDS** defines the *device* as the unit that transmits GPS signals for a particular vehicle. A given device must have a UUID (`device_id` below) that is unique within the Provider's fleet.

Additionally, `device_id` must remain constant for the device's lifetime of service, regardless of the vehicle components that house the device.

### Geographic Data

References to geographic datatypes (Point, MultiPolygon, etc.) imply coordinates encoded in the [WGS 84 (EPSG:4326)](https://en.wikipedia.org/wiki/World_Geodetic_System) standard GPS projection expressed as [Decimal Degrees](https://en.wikipedia.org/wiki/Decimal_degrees).

[Top][toc]

## Trips

A trip represents a journey taken by a *mobility as a service* customer with a geo-tagged start and stop point.

The trips API allows a user to query historical trip data. The API should allow querying trips at least by ID, geofence for start or end, and time.

Endpoint: `/trips`  
Method: `GET`  
Data: `{ "trips": [] }`, an array of objects with the following structure

| Field | Type    | Required/Optional | Comments |
| ----- | -------- | ----------------- | ----- |
| `provider_id` | UUID | Required | A UUID for the Provider, unique within MDS |
| `provider_name` | String | Required | The public-facing name of the Provider |
| `device_id` | UUID | Required | A unique device ID in UUID format |
| `vehicle_id` | String | Required | The Vehicle Identification Number visible on the vehicle itself |
| `vehicle_type` | Enum | Required | See [vehicle types](#vehicle-types) table |
| `propulsion_type` | Enum[] | Required | Array of [propulsion types](#propulsion-types); allows multiple values |
| `trip_id` | UUID | Required | A unique ID for each trip |
| `trip_duration` | Integer | Required | Time, in Seconds |
| `trip_distance` | Integer | Required | Trip Distance, in Meters |
| `route` | Route | Required | See detail below |
| `accuracy` | Integer | Required | The approximate level of accuracy, in meters, of `Points` within `route` |
| `start_time` | Unix Timestamp | Required | |
| `end_time` | Unix Timestamp | Required | |
| `parking_verification_url` | String | Optional | A URL to a photo (or other evidence) of proper vehicle parking |
| `standard_cost` | Integer | Optional | The cost, in cents, that it would cost to perform that trip in the standard operation of the System |
| `actual_cost` | Integer | Optional | The actual cost, in cents, paid by the customer of the *mobility as a service* provider |

### Vehicle Types

| `vehicle_type` |
|--------------|
| bicycle      |
| scooter      |

### Propulsion Types

| `propulsion_type` |
|-----------------|
| human           |
| electric        |
| combustion      |

### Routes

To represent a route, MDS `provider` APIs should create a GeoJSON Feature Collection, which includes every observed point in the route, and a timestamp. 

Routes must include at least 2 points: the start point and end point. Additionally, routes must include all possible GPS samples collected by a provider.

```js
"route": {
    "type": "FeatureCollection",
    "features": [{
        "type": "Feature",
        "properties": {
            "timestamp": 1529968782.421409
        },
        "geometry": {
            "type": "Point",
            "coordinates": [
                -118.46710503101347,
                33.9909333514159
            ]
        }
    },
    {
        "type": "Feature",
        "properties": {
            "timestamp": 1531007628.3774529
        },
        "geometry": {
            "type": "Point",
            "coordinates": [
                -118.464851975441,
                33.990366257735
            ]
        }
    }]
}
```

[Top][toc]

## Status Changes

The status of the inventory of vehicles available for customer use.

This API allows a user to query the historical availability for a system within a time range. The API should allow queries at least by time period and geographical area.

Endpoint: `/status_changes`  
Method: `GET`  
Data: `{ "status_changes": [] }`, an array of objects with the following structure

| Field | Type | Required/Optional | Comments |
| ----- | ---- | ----------------- | ----- |
| `provider_id` | UUID | Required | A UUID for the Provider, unique within MDS |
| `provider_name` | String | Required | The public-facing name of the Provider |
| `device_id` | UUID | Required | A unique device ID in UUID format |
| `vehicle_id` | String | Required | The Vehicle Identification Number visible on the vehicle itself |
| `vehicle_type` | Enum | Required | see [vehicle types](#vehicle-types) table |
| `propulsion_type` | Enum[] | Required | Array of [propulsion types](#propulsion-types); allows multiple values |
| `event_type` | Enum | Required | See [event types](#event-types) table |
| `event_type_reason` | Enum | Required | Reason for status change, allowable values determined by [`event type`](#event-types) |
| `event_time` | Unix Timestamp | Required | Date/time that event occurred, based on device clock |
| `event_location` | Point | Required | |
| `battery_pct` | Float | Required if Applicable | Percent battery charge of device, expressed between 0 and 1 |
| `associated_trips` | UUID[] | Optional based on device | Array of UUID's. For "Reserved" event types, associated trips (foreign key to Trips API) |

### Event Types 

| `event_type` | Description | `event_type_reason` | Description |
| ---------- | ---------------------- | ------- | ------------------ |
| `available` | A device becomes available for customer use | `service_start` | Device introduced into service at the beginning of the day (if program does not operate 24/7) |
| | | `user_drop_off` | User ends reservation |
| | | `rebalance_drop_off` | Device moved for rebalancing |
| | | `maintenance_drop_off` | Device introduced into service after being removed for maintenance |
| `reserved` | A customer reserves a device (even if trip has not started yet) | `user_pick_up` | Customer reserves device |
| `unavailable` | A device is on the street but becomes unavailable for customer use | `maintenance` | A device is no longer available due to equipment issues |
| | | `low_battery` | A device is no longer available due to insufficient battery |
| `removed` | A device is removed from the street and unavailable for customer use | `service_end` | Device removed from street because service has ended for the day (if program does not operate 24/7) |
| | | `rebalance_pick_up` | Device removed from street and will be placed at another location to rebalance service |
| | | `maintenance_pick_up` | Device removed from street so it can be worked on |

### Realtime Data

All MDS compatible `provider` APIs must expose a [GBFS](https://github.com/NABSA/gbfs) feed as well. For historical data, a `time` parameter should be provided to access what the GBFS feed showed at a given time.

[Top][toc]

[toc]: #table-of-contents
