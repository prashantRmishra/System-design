## TicketMaster
It is an online platform that allows users to purchase tickets for Concerts, Sport events, Theaters, and other live entertainment.
Has ~100M DAU

Functional Requirements
- Book ticket
- View an event
- Search for events

Out of scope Functional requirements
- Adding events( admin will be adding event, this out of scope for this system design)

Non functional requirements
- **Strong Consistency** should be favoured for booking tickets (no double booking)
- **High Availability** should be favoured for searching and viewing events
-  **Minimum search latency**
-  **Scalability** to handle surges of from popular events

Out of scope Non functional requirements
- GDPR compliance
- fault tolerance
- etc

Core entities (It is nothing but the **data this is persisted** in the system and is exchanged between the **APIs**)
- Event ( Details of the event)
- Venue ( The place where event will be held)
- Performer ( The detail of the artist who will be performing)
- Ticket ( Ticket for the given event) 


<div style="border: 2px solid green; padding: 10px; font-weight: bold; font-style: italic;">
💡 I not gonna list key fields and columns yet because I don't know them yet, I am too early in my design they are going to evolve naturally, when we will go on ahead in deep dives/high level design we will be more clear on exactly the fields that matter
</div>


## Api or interfaces

**Search**
```json
GET /search?token={token}&location={location}&date{date}
///Response: list of events with limited amount of information on the event

```
**Event**

```json
GET /event/:eventId
//Response: event details like venue, date, performer, ticket[] (list of available tickets) etc.
```

**Booking**

It is a two phase process
- You see a seat map and you choose a seat and go to second phase to purchase that ticket(associated to that seat)
    ```json
    header: //Jwt or sessionToken for User Authentication
    POST  /booking/reserve
    body:{
        "ticketId": // associated with the seat clicked

    }
    ```
- The second phase usually have a timer or countdown maybe 10mins to actually purchase that ticket, So for 10mins that ticket is reserved for you, if you don't book it in those 10mins that tickets goes back to being available

```json
    header: //Jwt or sessionToken for User Authentication
    PUT  /booking/confirm // PUT because we are not creating a new entry ( else we will have used POST)
    body:{
        "ticketId": "",// associated with the seat clicked
        "paymentDetails":{}// details of payment (to third part payment service )

    }
```

## High Level Design

***Simple High level design that satisfies our functional requirements, some of the assumption may not scale but that is fine, as that will be refined eventually in deep dives***

We will create simple microservices architecture

**Client**(user application) will interact with **API gateway**( which is responsible for routing the request to the correct micro-service)
Api gateway has responsibilities like Routing,Authentication and authorization, Single point of entry for the client applications, Rate limiting, etc.

***search-service***: Simple search service that will query the database based on the search token given and return the list of events with partial data like event name, venue, date (meaning not all the details will be returned)
This will work but it will be extremely time consuming as the query will go through all the rows in the event table to get the details
`pseudo query`

```sql
select * from 
event where type in [] 
and eventName like '%token%;
```
It will **not satisfy the low latency** search but it is fine at the moment, first we will try to achieve all the functional requirements and then in deep dives we will handle all the issues related to performance and quality of the application.

***event-crud-service***: responsible for creating/updating/deleting/viewing events

We will use postgres(relational db) 
- As we have to insure consistency in ticket booking
- There is mild relationship between tables like Ticket,Event,Venue,Performer etc.
<div style="border: 2px solid green; padding: 10px; font-weight: bold; font-style: italic;">

💡Reality is what ever the SQL database can do NoSql databases can also do nowadays.
For example you can have ACID property on DynamoDB as well.
The better thing to consider is what qualities you need which database can satisfy it, if both will work then choose any does not matter( maybe choose the one you are most familiar with)</div>


***booking-service***: **To handle two phase bookings**

The use will send the ticketId that they want to book (*this ticket will be marked reserved in the db*)and then they will make the payment that the booking-service will handle with the help of 3rd party payment gateway (It will handle the payment request asynchronously and give update back to booking-service which is either failed or success)
You will have some endpoint exposed in you booking service for it(3rd party payment service) to callback to.
If the payment is successful then the booking-service will update the ticket status from reserved to booked.
Ticket will be emailed to the user(out of scope) if payment success.

### Issues

What if the user clicks on a seat and goes to the payment page where the count down is 10min, what if that 10mins exceeds and what if they just close their laptop or they decide they don't want the ticket, then the status of the ticket will be reserved forever. So whenever we will show the seat map to other users that seat will be shown reserved forever (which should not be the case)
**Solution 1**: we can add a column called **reservedTimestamp** in ticket table,and when we will query to get the available seat map it will be like

```sql
SELECT * 
FROM ticket 
WHERE status = 'available' 
   OR reservedTimestamp <= NOW() - INTERVAL 10 MINUTE; /*get the ticket if the reserved time stamp is more than 10 mins ago (meaning not reserved any more)*/
```
![reservedTimestamp](image-2.png)
Well, this makes the query little bit more complicated but this will surely work.

**Downside/Challenge**:
You make the data model bit more confusing, like you have ticket status='reserved' which are not actually reserved (as we are confirming by the `lastReservedTimestamp`)

**Solution 2**:We can introduce a crone job, which will run for let say every 10 minutes and will check(by querying the db) if there are any tickets whose status='reserved' and `reservedTimestamp` is more than 10mins ago, if yes then it will make the status='available'
**Downside/Challenge**:
This works and this is a totally valid approach but it will still not be most effective, since the crone job is running every 10 minutes let say the crone jobs runs at 12:09 and the ticket was reserved at 12:00 then  the now()-`reservedTimestamp` = 9mis which is still within the window so 
it will not be updated but the next crone job will run at 12:19, meaning the ticket will stay reserved for total of 19mins which should not have been the case.
![croneJob](image-1.png)

**Solution 3(Best Solution)**: User Redis lock with TTL of 10mins.
Instead of using the crone job we can have a distributed redis lock on the ticketId with TTL of 10mins.
We can have some key-value pair like {ticketId:true} ( value part does not matter really) and TTL=10mins, after 10mins this entry will be deleted.
When ever a ticket is selected instead of updating its status in the Ticket table the service(booking-service) will directly update the ticketId in the redis lock.
So for any event when available tickets/SeatMapping is requested (by event-crud-service) it will get all the seats from tickets (status='available') and then for each of the ticketId it will check if it is present in the redis lock or not if present then they will be removed and rest of the tickets/SeatMapping will be returned to the client.
We are using distributed redis lock instead of using in-memory cache in booking-service is because booking-service is not a single instance, it will scale horizontally depending of the need, and we want the lock on the ticketId to be consistent across all the instances of services that are going to use it. Hence using distributed redis lock makes more sense.
**Challenges**:
*What if the redis lock goes down?*
We can immediately bring new one up, but that would mean any user that reserved the ticket in the last 10mins lost their reservation.
So in theory we can have several users going to the payment page and try to book this ticket, Since we are using PostgresDb so we still have ACID property on right to available or booked meaning right needs to be completed before anyone can read it, So whoever ends up completing the purchase first will get the ticket(that will be marked at booked) and rest of the users will get an error.
This will result in bad user experience, this is a conversation we can have with the product team like if this happens then we will loose all the reservation in the last 10mins but the consistency will be guaranteed that one seat will be alloted to only one user.


![hightlevel-design](image-4.png)

## Deep Dives

How to improve the search to achieve low latency search 
Use something like [ElasticSearch](https://github.com/prashantRmishra/System-design/blob/main/elastic-search/readme.md) that uses [Inverted Index](https://github.com/prashantRmishra/System-design/blob/main/elastic-search/readme.md#how-documents-are-stored-inside-the-lucene-segments) to make searching document by terms really quickly
![invertedIndex](image-5.png)

ElasticSearch also support GeoSpatial queries which uses combination of quadTrees and geo-hashing So searching with location filter, date, time will also be supported.
If anything changes in the primary database, the change can be ingested to the ElasticSearch.
- **Update via event-crud-service**:For example anytime a new event is added in the db by *event-crud-service* it can also add the same event in the ElasticSearch.
  - Now this will put some complex logic in *event-crud-service*
    - If primary db write fails then you should not be writing to ElasticSearch
    - If the ElasticSearch write fails (after primary db write success) then you might wanna retry the write in ElasticSearch
    - If you choose to back out the write from the ElasticSearch then you might wanna delete the write from the primary db as well
  - So There are somethings to consider depending on the product requirement.
  ![ingest-via-event-crud-service](image-6.png) 
- **Update via Change data capture** (CDC):
  - So anytimes something changes in the database that change event will be put on a stream and we can consume it and update the ElasticSearch.

