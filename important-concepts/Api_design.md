## API Design
### Types
- Rest
- GraphQL
- RPC

### Rest
- Buch of standards for creating/sharing/updating resources using HTTP method
- URL always has nouns(plural) no verb the verb is represented by the HTTP Method
    - Example Event details in the BookMyShow app
        - `GET /events/` -> gets all the events
        - `GET /events/{id}` -> get the details of the event having id ={id}
        - `GET /venues` -> get the details of all the venues
        - `GET /events/{id}/tickets` -> get all the tickets for the given event

#### HTTP methods:
- GET -> Retrieves data 
- POST -> Creates data
- PUT -> Idempotent update/replace
- PATCH -> partial update
- DELETE -> delete data

#### Inputs
- Path Param
    - `GET /events/{id}`
        - Required to identify the resource we are working with
- Query Param 
    - `GET /events/?city=mumbai?date=2026-09-29`
    - Used when we want to specify optional filters
- Request body
    - `POST /events`
     ```json
        {
            title,
            location,
            description,
            date
        }
     ```
    - Used when we want to create/update/patch the resource 
**In a nutshell: If it is required to identify resource-> use Path param, if it is optional filter -> use Query param, and if you are sending data use the Request body**

#### Responses: Example GET /events/ -> Event[]
- Status code
    - `200, 201` -> success, created
    - `400, 401,404` -> bad request, unauthorised, not found
    - `500` -> server error
    - usually `2xx,4xx,5xx` are enough
- Response body (Usually JSON)


### GraphQL
- Single HTTP request that whose body includes exactly what is needed and that is what is returned.
    - For example for below multiple REST requests:
```Json
        GET /events/123
        GET /events/123/tickets
        GET /venues/5342

```

    - Request body of graphQL will have something like

```json
        query{
            event("123"){
                name
                date
                venue{
                    name
                    address
                }
                tickets{
                    section
                    price
                    available
                }
            }
        }
```

- It was developed for mobile apps when rest calls were talking too long and had to hit multiple endpoint to get the data to get to the targeted data.
- A single call would have all the the data that is needed in response which is sent to one endpoint and the response returns just that.
- It authorisation at filed level via schema resolver 
- Limitations:
    - It creates fan out event like get top 100 events then get all the venues of the event those 100 events resulting in 100 more subqueries -  how this is solved is via batching or a tool called data loader where we group all the venues needed into a single query which is then executed on the db.



### RPC
- Intra mico-service communication i.e service to service comm, it is like local function call but over network
- Very efficient, lighter and faster than rest when using binary protocols like gRPC
- When dealing with rest call we have to deal with http headers, status code, url parsing and we are conveting to and from JSON. But rpc cuts through all that by making direct function call using binary format (for example Protocol buffers or ProtBuff it stores data in raw bytes insteaed of raw user readable format meaning it takes far less space hence much faster)
- Why not use RPC between client and server then -  because the clients are diverse from browsers to mobile apps and they all need to understand your api's without much setup and because rest uses standard HTTP methods that all clients already understand, it passes through firewall easily and since it is in JSON it is easy to debug by developers, rpc on the other hand requires both sides to agree on specific protocols and the binary formats.
#### So in a nutshell
- REST : Global language which is inefficient(compared to RPC) but everyone understand
- RPC : Internal language that is hyper efficient that internal services understand that you control/manage

Example:
```Java
//instead of POST /events/123
getEvent(event: "123") 

//instead of POST /events/123/bookings
createBooking(event:"123", userId:"234", tickets:[])
```

**These method are defined by datatype called protobuff which is compact strongly typed language**


















