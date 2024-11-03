Design yelp like system
Yelp is a web application/mobile application that allows users to find businesses online, view details of the business based on location and rate/review businesses.

- [Requirements](#requirements)
- [Core entities](#core-entities)
- [Api or interface](#api-or-interface)
- [High level design](#high-level-design)

## Requirements

functional:
User should be able to search for business
User should be able to see details of the business
User should be able to rate and review the business


Non-functional:
Low search latency
High availability of the business details 
100M MAU, 10M businesses
user to business rating/review 1:1

out of scope
GDPR compliance
fault tolerance
etc

## Core entities
Business
Review
Location
User

## Api or interface

Search for a Business
```
GET /search/?term={term}?lat={lat}?long={long} > partial details of list of business

```
View Business
```
GET /business/[bId] > Business details with review skeleton and pagination
```
Review Business
```
POST /review > 200 (rating created )
{
    bId://
    rating://1-5
    review:// optional
}
```

## High level design

***search service***: self explanatory in the diagram
For efficient searching we can use **elastic search database** that is optimized for searching, which is very efficient when the data is read heavy and does not change that often, since in yelp like system people will be reading(searching businesses and/or viewing details of the businesses) more than writing reviews (another write use case will be adding details of more businesses in the platform)
Elastic search can utilized Inverted index and doc value index for searching based on terms and it also supports GIS or Geo Spatial Index called **Geohashing** which can also be used to optimize search results based on the `latitude` and `longitude`.

For updating the data in the elastic search (that needs to happen when ever there is some write in the actual data in the primary db) this change can be propagated in the elastic search db using **CDC**(Change Data Capture)

Crude estimations:
10M businesses
Assuming 100M monthly active users
100:1(read to write ration) then 1M users are writing review every months which is nothing but 33k review per day which is not that much
we can assume modern aws managed postgresDb can store upto 100TB of data
If the each business details amounts to 2KB size the for 10M = 10M*2KB = 20GB(which is nothing entire details can fit into a DB) 



