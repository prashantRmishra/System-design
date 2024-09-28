
## Twitter

Functional requirements:
- Post tweet(post text,post video/images)
- Mark favorite 
- Share tweet
- Follow
- twitter feed/timeline of top tweets from all the people the user follows. 
  
Non functional requirements
- Availability > consistency (availability is more important than consistency to us)
- Consistency can take a hit, if a user does not see tweet for while it should be fine.
  
Extended requirements:
- Searching for tweet
- retweet
- Tagging other users
- Tweet notification

**Design consideration**: 
Let us assume we have 1B = 100cr total users and 200M = 20cr DAU(daily active users). Also assume we have 100M = 10cr new tweets every day and on an average each user follows 200 people.
How may favorite per day ? If, on an average 5 tweets are made favorite per day we will have: 200M*5 favorite = 1B = 100cr favorites marked per day.

*Traffic estimation*:
How many tweets-views our system will generate ?
-   Let assume active users on an average visit their timeline twice a day and visit profile of 5 other people and on each page the user sees 20 tweets then
    -   200M DAU * (2 + 5)*20 tweets  = 28B/day = 28000cr tweets-view per day

*Storage estimation*:
-   Let say each tweet is of 140 characters and each character takes 2 bytes of storage, also assume 30B of storage for tweet metadata like timestamp,location, etc.
    -   Since in system 100M new tweets are generated every day
        -   100M * (140 * 2 + 30) bytes = {% katex inline %}3 x 10^10{% endkatex %} bytes = 30GB/day
    - Not all tweets will have media like image/video let say every 5 out of all the tweets will have image and every 10 out of all the tweets will have video, assuming an average image size to be 200kb and an average video size to be 2mb
      - (100M/5) * 200KB + (100M/10 )*2MB = 4TB + 20TB = 24TB/day

*Bandwidth estimation*:
-   Since the total ingress(write) is 24TB/day(approx) which is approx 280MB/sec(approx conversion are take into consideration like 1TB = 1000GB, 1GB = 1000MB for simplicity of understanding)
-  for engress(read) we have 28B/day tweets, we must show image of the tweet if it has image, let us assume that the user seems every 3rd video they see in their timeline,total engress will be
   -  28B ( 140 * 2 + 30)/86400secs = 90MB text/sec
   -  (28B/5 *200KB)/86400sec = 13GB/sec
   -  (28B/10/3 * 2MB)/86400sec = 21GB/sec
   -  i.e close to 35GB/sec of bandwidth we need 

  

**API design**:
- post(apiDevKey,tweetData,tweetLocation,userLocation,mediaIds)
  - apiDevKey: api dev key of a registered user account
  - tweetData: 140 characters of tweet data
  - tweetLocation: optional location of tweet like longitude, latitude etc
  - useLocation: optional location of user from where the tweet was posted
  - mediaIds: array of media ids associated with the tweet like image, video etc.
  - return : a successful post will return url to access the same tweet.
   
  



**High level design**:

*Upload/post/tweet*:

We need a system that can store 100M/day tweets and read 28B/day tweets, it is easy to note that tweeter is very read heavy application.

User will post the tweet, the load balancer will receive the tweet and will send to appropriate tweeter-server to handle request.
We need an efficient database that can store huge number of tweets and is also read optimized, we also need a object storage system like s3 for storing all the image and videos
During peak time of application usage the matrix of read/write tweets can go higher, this we have to keep in mind while designing the application.

*Database/schema design*:
We need to store the data about the users, their tweets,their favorite tweets,and people they follow.

If we choose sql database then following tables will be required
user(userId:pk, Name:String,address:varchar,dateofbirth:datetime,lastlogin:datetime)
tweet(tweetId:pk,userId:int,content:varchar, tweetLongitude,tweetLatitude,userLocation)
favorite((tweetId,userId):pk, crationDate:datetime)
userFollow((userId1, userId2):pk) ( note: both can be primary key)

advantage: 
-   structured schema
-   no duplicates values, normalized data

Disadvantage:
- latency will take a toll as to fetch data from more than 1 tables will require join operation to performed

If we choose to go ahead with NOSQl data base like mongo db, we can store duplicated data in multiple documents and there won't be any need of performing joins hence read time will be faster.
-   for example favorite tweet in sql will reqired join between favorite, user table, since it is read heavy application this operation will eventually increase the response time. But if we use mongoDB we can store duplicate data like userDetails in favorite document and we can store tweet details in user document this way we will only require to access either user or favorite document and we will get all the details.

**Data sharding**: 
Since our application is read heavy and huge numbers of tweets are getting written on dailiy basis, we have to do some replication and sharding of the database as well.

*Sharding based on userId*:

- Starting based on user ID, if we shared our database, we support user ID whenever we supply user ID to the hash function, it will generate the server number to which the details of the user like post or tweets like follows. Every thing will be stored on that server. So we just have to look into that server and we will get all the detail

- Disadvantage:
  -  What if a user becomes hot, which means if a user is very popular, he or she has a lot of followers then most of the people will try to see posts from this user that might create a lot of load on a single server that has details of hot user, that scenario user ID based sharding may not be ideal.

*sharding based on tweetId*:

- Sharding based on tweet ID given the tweet ID. The hash function will randomly distribute the details of the tweet onto different server. This will distribute the data uniformly, but it might introduce increased latency.
- Consider an example of timeline of a user, our application server will find all the users that current user follows and application server will query. All the database starts. All these database starts or instances will return a set of queries. Then an aggregate server will aggregate these queries and give it back to our requesting server, and then requesting server will sort it based on timeline and then displayed on users timeline, since we are searching in all the database shard, this will increase the latency.
We can further improve the performance by introducing cache in front of the database that will store hot tweets

*Sharding based on tweetCreationTime*:

- Storing tweets based on tweet creation time, it will help us fetch recent tweets quickly because we will have to look into smaller set of database servers, but the distribution of tweets will not be uniform as the load on one of the servers will increase while other servers will sit idle.
- Similarly, we can think of other shading techniques and depending upon the use case, we can decide which technique is best for database sharding.

Cache:
- Application servers before hitting the database. They will check if the desired tweet is present in the cache. If yes, the tweet will be fetched from the cash cache and will be displayed to use this timeline. We can use cache that can store the hot tweets, resulting in increased performance.
- If we go by the logic, 80-20 rule meaning, 20% of the users generate 80% of the traffic then we can store 20% of all the tweets in the cashier.

- We can use least recently used replacement policy for removing older tweets and adding new tweets in the cache
- Depending upon the clients usage pattern, we can create multiple instances of our cache
- Can we leverage from latest tweets?
  - If we assume that 80% of the user, see the latest tweets in the last three days by that logic, we can cash tweets from last three days since we know hundred million new tweets are written every day which is equal to 30 GB tweets are created per day, then the cache size would be less than hundred GB and we can cash latest tweets from the last three days
  - Cache will have details in key value pair by key will be user ID and value will be tweets of the user in linked list format. Since we're trying to store the latest tweets, we can always add the latest tweets at the head of the link list and older tweets will be at the end that can be removed to make a room for newer tweets that can be added at the head again.
  - On a similar design, we can think of cashing images videos as well for the given user.
These data should be replicated on multiple servers to handle multiple read requests from users
  
Replication and fall tolerance
Since our application is read heavy, we can think of creating a secondary application of all the charts. So in the event of a failure, we can quickly fill over to secondary shard. This way we can ensure availability

**Load balancing**:
- We can have a load balancer between client and application server to distribute clients request between different application servers depending upon the traffic on each server. For this, we can make use of round-robin algorithm that will ensure uniform distribution of request to application server, but it will not have any way of knowing if a server fails, so we need an intelligent logic for load balancer so that it can check health status of application serve on timely basis and based on that it can relay the requests
- We can have another load balancer between application server and distributed caches 
- Another load balancer between application server and replicated or sharded database servers


**Feed/view posts**:

The dashboard of the client/tweeter user will have list of posts from the tweeter user this user follows.
A request will be sent to twitter-server

Tweets/video/image/static content
- The static content of the tweet will be made available using CDN (system of edge server distributed geographically to server static data quickly).
- CDN will pull static data from the distributed instance of encoded s3. and will cache on nearest edge server.
- So the tweeter server will make request to nearest edge server to get details of tweets of tweeter user that the current user follows and the same will be given back by the edge server.








