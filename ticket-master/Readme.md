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

***Event-crud-service***: responsible for creating/updating/deleting/viewing events


We will use postgres(relational db) 
- As we have to insure consistency in ticket booking
- There is mild relationship between tables like Ticket,Event,Venue,Performer etc.
<div style="border: 2px solid green; padding: 10px; font-weight: bold; font-style: italic;">

💡Reality is what ever the SQL database can do NoSql databases can also do nowadays.
For example you can have ACID property on DynamoDB as well.
The better thing to consider is what qualities you need which database can satisfy it, if both will work then choose any does not matter( maybe choose the one you are most familiar with)</div>


***booking-service***: To handle two phase bookings
The use will send the ticketId that they want to book and 
then they will make the payment that the booking-service will handle with the help of 3rd party payment gateway (It will handle the payment request asynchronously and give update back to booking-service)
You will have some endpoint exposed in you booking service for it(3rd party payment service) to callback to.

https://youtu.be/fhdPyoO6aXI?t=1664