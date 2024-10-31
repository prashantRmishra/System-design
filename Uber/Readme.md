# Design Uber like system
- [Design Uber like system](#design-uber-like-system)
  - [Requirements](#requirements)
  - [Core Entities](#core-entities)
  - [Api or interfaces](#api-or-interfaces)
  - [Hight Level Design](#hight-level-design)


## Requirements

**Functional**:
User Should be able to Search for source and destination and get ride fare estimate
User should be able to request a ride based on estimates
Driver should be able to accept/deny request and navigate to pickup/drop-off

*Out of scope* 
Multiple car types (assuming only one type of vehicle will be shown i.e **Uber Go**)
Rating for Drivers and Riders
Schedule a ride in advance

**Non-functional**:
**Low latency matching** < 1min to match to a Driver (or failure if no match)
**Strong consistency** for the ride booking (Ride to Driver is 1:1) 
**High availability** outside the matching
**Handle high throughput**, surges in requests in pick hour or in case of special events (100 or 1000 of requests with a given region)

***Let the interviewer know you are not gonna do back of the envelope estimation just yet***

*Out of scope*
Fault tolerance
GDPR(global data protection regulation)  compliance 
Monitoring,logging alerting
CI/CD pipeline


## Core Entities

Ride
Driver
Rider
Location


## Api or interfaces

**Create Ride estimate based on source and destination**
*Note: Ride details will be created and returned back to user(If the user does not choose go ahead or kills the app or does not do anything, in this case we will have unused data laying around in db(Ride details) that can be removed as cleanup activity or can be used for analytics like what rides are people looking for and not booking)*

```
POST /ride/fair-estimate > partial ride detail like rideId, ETA, price
"body":{
    "source"://
    "destination"://
}
```

**User Request Ride**
```
PATCH /ride/request > 200 (HTTP code) this is asynchronous process( as the driver will be assigned once the ride is accepted by the Driver, hence only confirmation is fine like 200: meaning ride request has been successfully updated)
"body":{
    "rideId"://
}
```

**Driver updates location periodically**

```
POST /location/update
"body":{
    "lat"://
    "long"://
}
```
**Driver accepts request**
```
PATCH /ride/driver/accept 
"body":{
   "rideId"://  
   "accepted":true or false
}
```
**Driver navigating to pickup or drop-off**

```
PATCH /ride/driver/update > next pickup(lat/long) where the driver have to go | null
"body":{
    "rideId"://
    "status:"pickup" : "drop-off"
}
```

## Hight Level Design

***ride service***
Based on the Rider's input of source and destination the ride service will (with the help of third party mapping service provider like GoogleMap ETA will be obtained based on the source and destination given) compute the fare based on the ETA.
The details of the ride will be created in the primary db with attributes like `rideId, ETA,Source,destination,fare or price, status`,etc and the partial details of the ride like `ETA, fare` and `rideId` will be given as response to the `Rider`

Note: below happens once the Driver has received the Ride notification from the `ride matching service`

Once the Driver accepts the ride request, request will be sent to `ride service` which will update the `Ride` (by updating the `driverId`) `status` like `matched` or something similar.

Last thing that happens is Drives updates that the source/destination has arrived and the `ride service` updates the Ride status to `at pickup | at destination | ride complete` something like that and driver get the response of the next location (lat/long) where they need to be or null

***rider matching service***

This will be responsible for matching the Driver to the ride 
Note: a different micro-service is created for this because this task will be asynchronous (matching Driver to the ride asynchronously) and it will have more computations hence it is better to have these in a different micro-service as it will allow us to scale this service independently.
It will query the Location Db to get the location of all the Drivers which are in close range.
After getting the location of the Drivers it will make sure that non of the drivers are in ride at that moment, for this it will send request to `ride service` which will query primary db to get the status of the driver i.e `in-ride` | `offline` | `available` and drop the locations of all the drivers other than `available`
Finally, it will start sending ride request notification via native `notification service` to available drivers.


***location service***

Drivers will periodically (say every 5 second) update their latitude and longitude, `location service` will update the location db

![HighLevel](image.png)


