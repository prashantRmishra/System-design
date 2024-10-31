# Design Uber like system
- [Design Uber like system](#design-uber-like-system)
  - [Requirements](#requirements)
  - [Core Entities](#core-entities)
  - [Api or interfaces](#api-or-interfaces)
  - [Hight Level Design](#hight-level-design)
  - [Deep Dives](#deep-dives)
    - [Low latency match for drivers](#low-latency-match-for-drivers)
    - [Consistency of matching](#consistency-of-matching)
    - [How to handle surges in request?](#how-to-handle-surges-in-request)
    - [Increased availability](#increased-availability)


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

Last thing that happens is Drives updates (`update()` request from the gateway to ride service is missing in the diagram) that the source/destination has arrived and the `ride service` updates the Ride status to `at pickup | at destination | ride complete` something like that and driver get the response of the next location (lat/long) where they need to be or `null`

***rider matching service***

This will be responsible for matching the Driver to the ride 
Note: a different micro-service is created for this because this task will be asynchronous (matching Driver to the ride asynchronously) and it will have more computations hence it is better to have these in a different micro-service as it will allow us to scale this service independently.
It will query the Location Db to get the location of all the Drivers which are in close range.
After getting the location of the Drivers it will make sure that non of the drivers are in ride at that moment, for this it will send request to `ride service` which will query primary db to get the status of the driver i.e `in-ride` | `offline` | `available` and drop the locations of all the drivers other than `available`
Finally, it will start sending ride request notification via native `notification service` to available drivers.


***location service***

Drivers will periodically (say every 5 second) update their latitude and longitude, `location service` will update the location db

![highlevel](image.png)

## Deep Dives

### Low latency match for drivers
System should be able to get location of drivers given the lat and long fairly quickly.
Locations of drivers are getting updated very often (say every 5 seconds) So system should be able to handle those many write requests.
Let say we have 6M total Drivers and 3M are active at any given point, if every 5 seconds drivers are updating their location then it will result in 3M/5 = 600K Transactions Per Second(TPS)

[Using postgres for location based queries](https://github.com/prashantRmishra/System-design/blob/main/important-concepts/PostGresPerformanceAndSearchLatencyForSpatialQueries.md#using-postgres-for-location-based-queries)

[Optimized approach for location based queries (using in-memory datastore redis cluster)](https://github.com/prashantRmishra/System-design/blob/main/important-concepts/PostGresPerformanceAndSearchLatencyForSpatialQueries.md#using-in-memory-datastore-for-location-based-query)

**Do we really have to overload our system with 600k transactions?**


If we choose to update the location every 20 or 30 or 1min or more this will reduce tps from 600k to something lower say 100k or even less than that.

We don't need to update the every drivers location, we can dynamically update (don't update locations of offline drivers or the drivers that are not accepting ride requests, if they are parked, if they are not moving)

### Consistency of matching
**We don't send more than one request at a time for a single ride** i.e a ride request should be sent to one driver at a time

We can do this in the `ride matching service` itself, we will get list of available Drivers (after querying ride service) and we will send notification to the Driver for the ride but only one Driver will receive the ride request at a time.

This can be achieved using `while` loop, where the ride request will be sent to a driver `i` (from the list of drivers) and we will wait for let say 10s before sending the same ride request to next Driver `i+1` if the current Driver does not accept the ride.

**We don't send more than 1 request at a time to the driver**
Note: Since the ride matching service will scale horizontally depending on the surge of users(due to some event like going/coming from live concert of Arijit Singh) requesting for the rides, different instances of ride matching service will be sending the request to the drivers. It becomes important to not send two requests at the same time to the driver, we need to have shared awareness of driver status between the instances of the ride matching service.

***This is similar to booking ticket in TicketMaster***

**Solution 1**
we can add another status like "notification-sent" in the driver table (for the given driverId)

So while sending another request the `ride matching service` will check db for the driver status and if it is not "notification-sent" then service  will send the notification to that user

***Issues***
As soon as the driver get the request the status of the driver will become "notification-sent" but what if the driver never responds, the status will be stuck at "notification-sent" and for indefinite period of time no instance of `ride matching service` will be able to send request to that driver

**Solution 2**
We could be use crone job that will periodically check if the driver status is "notification-sent" and the `statusUpdateTime` (another column addition in the Driver table), if NOW()-statusUpdateTime >=10s then it will update the status back to "available"

***Issues***
This is a good approach and it will work but still not the efficient one, what if the CroneJob runs every 10s and statusUpdateTime is 12:00:00 PM and CroneJob runs at 12:00:09PM 
since the time difference is still within 10s it will not revert the status of the Driver but next CroneJeb will run at 12:00:19, So for total of 19seconds the status of the driver would be "notification-sent" which should not be the case.

**Solution 3**
Best solution will be to use Distributed redis cache that will store the driverId when a request is sent to the driver the data in the cache would be in the `key:value` pair as `driverId:true` ( value does not matter though) and `TTL` will be 10s, So every instance before sending the request to the driver will check if the driverId is present in cache or not, if yes then the request will be sent to the next driver in the list.

And if the driver does not respond in 10s then the `driverId` will be removed from the cache (eviction after `TTL`) and the same driver will be again available for getting new requests

**Another smart solution**
Use Amazon's DynamoDB as the primaryDb(that can be scaled largely indefinitely) and we can introduce another table called `Driver lock` and we can add driverId in it and set TTL to 10s(we can set TTL in dynamoDb) and it will work same as the redis distributed lock



```
while(noMatch){
    driver = nextDriver;
    lock(driver)
    sendNotification(driver)
    wait(10s)
}

```

### How to handle surges in request?

What if we can't scale the ride matching service quick enough to handle the huge surge in ride booking request?
We can introduce a queue between gateway and the `ride matching service(s)`  which will pull the requests off of the queue and match the request to a ride

This approach is good enough to handle the surges that are not that frequent but can happen every now and then.

Partition the queue based on the location/region that way people will not have to wait for other people in the queue which are based our of different region/location.

![finaldesign](image-1.png)


### Increased availability

We can horizontally scale each service/db depending on the need like surges in the requests in a region.

Additional way of increasing the availability is to create the multiple instances of the whole setup split by region like in india we can have multiple instances of the whole setup for north/south part the country depending on the use case and traffic estimates