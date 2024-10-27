# FaceBook News Feed

## [Reference](https://www.hellointerview.com/learn/system-design/answer-keys/fb-news-feed)

[Requirements](#requirements)

[Design Constraint](#design-constraint)

[Core Entities](#core-entities)

[Api or Interfaces](#api-or-interface)

[High Level Design](#high-level-design)

[Questions](#questions)

[Deep Dive](#deep-dive)

---

## Requirements

**Functional**:

- User should be able to post (new post)
- User should be able to follow (follow)
- User should be able to see feed of posts from people they follow
- User should be able to page through their feed

***For the sake of this interview/ or for all the interview we can assume that the user is already authenticated and we have their UserId stored in session or jwt***


**Non Functional**:
- System should be **highly available**,(availability > consistency) eventual consistency with a 
- Posting and viewing the feed should be **fast < 500ms**
- Should be able to **scale**/handle 2B users
- User should be able to **follow unlimited** number of users and User Should be allowed to have **unlimited number of followers**.

## Design Constraint
Hard part of the problem is handling users having huge following/ users that follow huge number of people.

## Core-entities
**User**: User in our system.
**Follow**: A unidirectional relationship between users in the system.
**Post**: A post made by a user in our system, other users following the Owner of the post(another User) should be able to see this post in their feed. 

## Api or interface
Api's are the primary interface using which the user will interact with the system, just define each of your endpoints for each of our functional requirements.

`New post`

```json
POST /post/create
Request:{
    "content":{

    }
}
Response:{
    "postId"://...
}
```
Leave the content blank to account for more reach content or structured data that we might want to add in the post 

`Follow a user`
```json
POST /user/[id]/follow
Request:{

}
```
Follow operation will be binary, we will assume it to be idempotent, so it does not fail even if the user clicks on the follow button twice.

`View feed`

```json
GET /feed
Response:{

}

```

## High level design

---
- **User Should be able to post** and access it via `postId`
  -   We are going to build a basic flow and add more complexities later on
  -   Since we know we will have to scale later we will put horizontally scaled service behind the **API gateway/load-balancer**
  -   Leaving caching for the deep dive part
  -   We can handle more traffic by scaling horizontal i.e by  addition post-service
  -   User hit API gateway, which sends request to one of the instances of post-service which creates an insert event in `Post` table in db (with `postId` being primary key).
  
![post](image.png)

  -   For **database** we can use key-value store like **Amazon's DynamoDb** for the sake of its simplicity and scalability and we can provision nearly limitless storage provided we spread our load evenly on our partitions
  
---
-   **User should be able to follow** other Users
    - Following a friend/page on facebook is like many to many relationship, we can use another table for this called Follow with `userFollowing:userFollowed` as the primary key.
    - We can create partition on user following to quickly see user's they follow economically, we can also create a secondary index on `userFollowed` to get all the users that follow them.
  
![follow](image-1.png)

---


-   **User should be able to view a feed of posts** from people they follow
    -   This has several challenges
        -   Finding all the users the given user follow
        -   Getting all the posts
        -   Showing those posts in chronological order on the feed
    -   We will start with naive approach and we will iteratively improve the design. We will start with the naive solution and solve the scaling problem separately
    -   We have and index on follow table to get the list of users a user follows quickly but we don't have any index on post table to get posts of various users.
    -   We can create a partition key for post table on `userId` and sort key on `timestamp` of the post this will sort the posts of the user within the partition in chronological order. This will allow us to quickly fetch the posts by a user.
    -   Note: partition key on `userId` will store all the posts from that user in the same partition making it quick to get all the `post` and sort key on the `timestamp` with the posts within each users partition by `userId`
  
![feed-service](image-2.png) 


***Here we have the simple feed service which will fetch all the users the given user follow from the follow table and get all the posts made by those users from the post table sorted on the basis of timestamp and finally the feed service will return the result back to user's feed.***

**This simple approach has challenges like**

        1. Our user may be following a lot of people 
        2. Those people may have a lot of posts
        3. Total set of posts will be very large due to 1 and 2
        4. Before we dive into these complexities let finish the functional requirements first

---

- **User should be able to scroll their feed**
  - We want to give almost infinite scroll to the user, for this we need to know what they have already seen in their feed.
  - We can leverage the timestamp of the last post that the user has seen (that will act as a cursor in the user feed) to get the list of next set of posts for the user.
  - Since we can tolerate the eventual consistency of 1 minute we can leverage cache to store the list of postIds in latest to oldest fashion for the give user.
  - When the user request for the first time or when the cache is empty we can get all the posts from the follow and post table and store large number of postID say 500 in the cache in ascending order of their timestamp
  - When the user request for the next set of post based on the timestamp received from the feed service we can lookup in the cache and get set of posts that are older than the timestamp received.
  - To ensure that the cache always has the latest posts for the feed of the given user we can keep the TTL lower that the eventual consistency time i.e. 1 min, by this way eviction will happen and new newer post will be added at at start within the eventual consistency window elapses.
  - This satisfies the basic requirements, but very performant or scalable!

![pagination-feed-service](image-3.png)

---

## Deep dive

- **How to handle users who follow large number of users?**
  - If a user is following a large number of users, the queries to the Follow table will take a while to return and we'll be making a large number of queries to the Post table to build their feed. This problem is known as "**fan-out**" - a single requests fans out to create many more requests. Especially when latency is a concern.
  - We can think of building the feed when a new post is created(at *write*) instead of building it at the time of *read*(when the feed request comes) i.e **creating a precomputed feed table** instead of querying the Post and Follow table 
  - We can create another table called feed that will only have list of posts for the feed of the given user, i.e. when ever a new post is created we will updated the feed table of userId that follow the given user who created given post. i.e **when a new post is created, we'll simply add to the relevant feeds.**
  - The feed table is list of postIds stored in chronological order (limited to say 200 posts)
  - We can create partition on the userId of the feed and its values will be the list of postIDs in order
  
![feed-table](image-4.png)

  All though it improves the read performance significantly, but it introduces another issue, what if a user having huge followers creates a post, then we will have to write to millions of 
  Feeds efficiently. Let us look into that issue next.

---
- **How to handle users who have huge number of following?**
  - When a user a large number of followers we again have the same fan-out problem.
  - When we create a post, we need to write to million of feed records. Since the we allow some inconsistency with eventual consistency within 1min window we have very little time to perform these writes in feed table.
    - **Bad Solution: SHOTGUT**
      - *Approach*:
        - Do it all at once, in worst case we are trying to write to million of feed with new post entry.
      - *Challenges*:
        - This is basically unworkable due to the limitation of no. of connections that can be made from out single Post-Service and the time limit available(<1min)
        - In the best case when it does work the load on our Post-Services will be uneven some of then will be writing to millions of feed records while some will sit idle.
    - **Good Solution: async worker**
      - *Approach*:
        - Make use of async worker behind the queue
        - Since our system can handle some delay of when a post is created and when it can be available in feed, we can queue up the write requests and have a fleet of workers consumes these requests and update feed.
        - Any queue will work here as long as it supports at least once delivery of messages and is highly scalable like Amazon's Simple Queue Service(SQS) will be great.
        - When ever a new post is created an entry will be added in the queue with the postID and the creator userId, the worker will lookup all the followers of that creator and update the feed record by prepending the postID at the front of the feed record.
      - *Challenges*:
        - The Throughput of the workers need to be enormous, for small account with limited no. of followers this is not a problem: we will only be writing to few hundreds of feed.
        - But for mega accounts with millions of followers workers have a lot of work to do.
  
    - **Great Solution: async workers with Hybrid feed**
      - Extending the above approach with slight modification, we can choose which account we would like to precompute and which don't.
      - For justin Bieber (and other accounts having millions of followers) instead of writing to millions of followers feed record we can simply add a flag in `Follow` table which indicate this perticular follow is not precomputed {like `prashant123:justinBeiber232` will have `flag` set to `true`( could a `boolean` value) indicating feed record of `prashant123` is not precomputed}, in the asyc worker queue we will ignore the requests for such users.
      - On the read side when the user requests their feed via feed service, we can grab their (partially) precomputed feed from the `Feed` table and merge it with with those accounts which are not precomputed( basically we will know there is a `flag` set on `prashant123:justinBeiber232` and we will get recent post of `justinBeiber232` and merge it with the partially computed feed of `prashant123`) 
      - This hybrid approach allows us to choose if we want to fan out on read or write, and for most users we will do little bit of both.
      - This way we don't need One-Size-Fit-All Solution.
    - Challenges:
      - Doing the merging of feeds at read time vs at write time means more computation needs to be done in the Feed Service. We can tune the threshold over which an account is ignored in precomputation.
      - Meaning we can put a threshold which once reached the given account will be flagged are will be ignored for precomputation.
   
  
![aync-worker](image-5.png)

- **How to handle uneven read of posts?**
  - Feed service is still going to Post table to get the posts (**Note** `Feed cache` is storing the list of recent `postIds` for the feed of the given user, the feed service after getting these `postIds` going to `Post` table to get the details of the Post and returning the feed to the given user)
  - For vast majority of the posts they will be read for few days and then never again, for some posts(specially from the accounts of user having millions of followers), the no. of reads the post experiences in the first few hours will be massive.
  - DynamoDb offers nearly infinite scaling with conditions like there being even load across the keyspace(partition key) if some posts are getting 500 read requests per seconds and others are getting 0 then it is not even distribution of load.
  - How do we solve this **issue of hot key in post table**?
    - **Good Solution: post cache with large keyspace**
      - *Approach*:
        - Insert distributed post cache between reader of the post and the post table itself.
        - Since the posts are very less likely to be edited than created we can keep a long Time to live (`TTL`) on posts and have our evict the least recently used(LRU) posts.
        - As long as the cache is big enough to house most of the recent posts the vast majority of the post table requests will hit our post cache instead.
        - If we have N hosts (in distributed Redis cache cluster there  are many nodes or server on which the load is distributed read [this](https://github.com/prashantRmishra/System-design/tree/main/redis#issues-and-remediation)) with M space each then we can effectively house N*M posts.
      - When the posts are edited we can simply invalidate the cache for that `postId`
      - For this we can use Redis cache and use postId as key to distribute the load across various server or nodes in the Redis cluster.
    - *Challenge*:
      - It has the same **hot key problem** that the `Post` table had.
      - For the unlucky shard/partition that has multiple viral posts will be getting uneven traffic which will make the cache hard to scale and many of the nodes/server in the Redis cluster will be under utilized.
  - **Great Solution: Redundant Post cache**
    - Like the distributed cache solution above insert a replicated cache between the readers of the post table and the post table itself.
    - Keep same approach for TTL and eviction as above approach.
    - Unlike the distributed cache solution above we can choose to have multiple distinct cache that our reads can hit.
    - If we have N servers/nodes/hosts in each distributed cache( must see:[Redis cluster](https://camo.githubusercontent.com/ef552ec7bc1635f5de57d9a95b9704d94fcd2647ff85b49478dac9609ebf648b/68747470733a2f2f6465762d746f2d75706c6f6164732e73332e616d617a6f6e6177732e636f6d2f75706c6f6164732f61727469636c65732f6c7837306963617a3435707a376c6230396661772e706e67)) with M memory of each server/node/host. In limit we can create N distinct caches(redis cluster) with total effective cachable space of M (instead of N*M) but with N time as much throughput.
      - *Clustered Redis has hash slots distributed across members of the [Redis cluster](https://camo.githubusercontent.com/ef552ec7bc1635f5de57d9a95b9704d94fcd2647ff85b49478dac9609ebf648b/68747470733a2f2f6465762d746f2d75706c6f6164732e73332e616d617a6f6e6177732e636f6d2f75706c6f6164732f61727469636c65732f6c7837306963617a3435707a376c6230396661772e706e67) (nodes). While there is replication, that replication is primary/secondary and the secondary receives no traffic while the primary is active. The problem with this setup is that a hot key means all traffic is routed to a single node.*
      - *Instead of having a cold standby and sharding our keyspace, we can simply have multiple instances which are load balanced by the client. This means even a hot key is distributed across many nodes*
    - This means we can evenly distribute the load of a viral post across our entire cache fleet.


![distributed-cache](image-8.png)

![multi-instance-distributed-cache](image-7.png)
---

## Questions

What is api gateway? is it any different from load balancers ?

What is graph database like Neo4j?

What is partition key of Amazon's DynamoDb?


---