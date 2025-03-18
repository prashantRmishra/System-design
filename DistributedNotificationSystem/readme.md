# Design a notification system
like :sending mobile push notification, bank notification(related to transaction),
promotional notification

## Functional
- It should Send notification
- It should be pluggable i.e more notification types(sms,email,whatsapp) should be easily incorporated
- Should be Built as saas (to know who is sending what type of notification, so that we can rate limit)
    rate limiting you need as an offering if you are giving this as a service to other companies 
- Prioritisation of messages: some messages have high priority(Like otp, bank transaction) and some have low priority like promotional messages


## Non functional
Availability<br>
Should be able to add more client and should be able to know which clients made how many requests


## Entities of the system
User<br>
Notification


## Api or interface of the system
Send notification (client wants to send notification to the given email)
```
POST /send/  200 status code
body:{
    content,
    priority, // High | low | moderate
    email
    type: sms | email | in_app | whatsapp
}
```
note: If the notification system is being built for internal use in the company then
```
POST /send/ ---> 200 , where userId is the user in the same company
body{
    content,
    priority, // High | low | moderate
    userId
}
```
## High Level Design

**Notification Service** <br>
Since this Notification system is to be built as SAAS product, it can be used by clients to sending notification (or can also be used within the org. for sending notification).
Client sends the notification details that needs to be sent, the notification service adds then into kafka and respond back saying the notification will be sent in some secods(so, no need to keep the client block untill the notification is sent)
This interface can have two model
1) Client wanting to send notification to some email address (this model will be used by other companies using this product as saas)
2) Notification is to be sent to the given userId and based on this info, the system needs to figure out how to send notification via sms, email, whatapps etc. (this model will be used within the company)
Notification Prioritizer:<br>
Based on the type of message it prioritizes messages  like otp(high),transactional(high) promotional(low) etc and puts them into respective topic queue.

**RateLimitingService**<br>
- There are 2 kinds
    1) the client who is calling me is that client
    allowed to send these many request to me?
        - there could be a subscription with a client that they 
          can send us 10 or may be more number of request in second
          (time window)
    2)The user to whom this notification is to be sent,
    can I send these many request to the user?
        - There could be configuration like we can not send more than 3 notification to 
        this user in a day
- Both of these can be implemented in a very similar way
  We can use redis and we can use clientId as key, value as the count(1 for the first request).
  We can put this in the redis with TTL as the windowsize(sliding/rolling window) (1,10 or more second or minute).
  Clients request will be processed if the clientId is present in the cache and the count is within the threshold limit and once the TTL is reached the entry(of the client) will be evicted. And for the next subsequent request new entry is added with count 1 and TTL as the window size.<br>
- Let say with some clients you have pay for use kind of subscription model, so there is no rate limiting as such(i.e the client can send us any no.of requests) but as the no. of requests increases the amount of money we charge them also increases, so in such case we will not be restricting the requests, we will only keep on incrementing the count

**UserPreferenceAndNotificationHandlerService** <br>
- Some users might have unsubscribed from the notification, some might have preference like they want only email notification and no sms no whatsapp, etc.
- This services get the user details like given the userId it gets the details of email of the user to which the notification is to be sent.
- after getting the email address, and other preferences we will have full notification data
with recipient detail, content and other metadata.
- then we can put the notification data into another queue (kafka3)

**Kafka3** <br>
Why to put the message into another kafka?
let say we have sms handler and in peak hours, we get so many sms notification that needs to be sent, this might put a lot of pressure on the `sms handler service` so it is better to put the messages in the queue and let the sms service sent the sms's on its own pace.

**Handler's and Vendor's** <br>

These handler will read the notification messages from their respective topics.
These handlers might integrate with multiple vendors e.g sms handler integrating with  vendor 1 to send sms in asia another vendor 2 to send sms in emea etc.
### IVRS calls
In e-commerce app like amazon/flipkart, when you place a very high value cash on delivery(cod) order, they send you an IVRS call, asking are you sure want to keep the respective order as cod.

### Pluggability:

Let say now you have to send whatsapp notification as well. You just need to add whatsapp handler at the end (like sms handler) <br>
Add another type at the `NotificationService`<br>
Add another topic for the `UserPreferenceAndNotificationHandlerService`

**Notification Tracker** <br>

We have to keep track of what all notification we have sent for various reason like auditing, so this writes all the notification in cassandra <br>

![highleveldesign](image.png)

## Deep dives

**Bulk Notification** <br>
What if you want to send bulk notification to all the users e.g send notification to all the 
users who have order milk in the last 3 days<br>
For this we can have `BulkNotificationUI` with some filter criteria and the notification and sends it out.
Filter criteria like: all users who have ordered milk in the last 3 days, send them notification with some text

There is something called Transactional data like User added something in the shopping cart, User placed the order, 
User made the payment, user cancelled the order, Returned order placed, etc (these are transactional data) based on such data <br>
We(the application) might want to send some bulk notification, do analysis on these transactional data.

**User transaction Data** <br>
User transactional data can be put into a different topic (trasactional_data) in the kafka1 and this data can be read by `TransactionalDataParser`.

**TransactionalDataParser** <br>
It reads the transaction data(of various formats) and converts them into format that it can understand and stores them into elastic search kind of storage where we(the system) can do some complex search/query to get insightful results.

**Query engine**<br>
It queries the ES(elastic search cluster/db) based on some criteria, it understands the schema of the data present in the ES
Criterial like: users who have ordered milk, cancelled orders more than 10 times recently(for fraud detection),etc. <br>
QueryEngine along with the BulkNotificationUI can also be used by other consumers like 
- rule engine
- fraud engine
- search plateform 

![deepdive](image-1.png)





  





  
 




 


