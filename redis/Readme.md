
What is Redis?
Single-threaded in-memory cache
Single-threaded means it has only one thread and manages all the requests via the same thread. That is, if one request is being served, the rest of the requests will be in a waiting state before they are picked and executed.

Has features that resemble tons of data structures we have already used like Hashes(objets), Lists, Bloom filters, Geospatial indexes, Time series, Sets, Sorted Sets, Streams, etc.

**Disadvantage**:
- It can not guarantee the persistence of the data
 - One of the reasons why you might not want to use Redis is durability, unlike traditional databases where commit guarantees that data is written off in disk, it is intentional in Redis to favor speed, but if you do need durability there are other options available as well like Aws MemoryDb.

---

## How it works

![how-it-works](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/kv3surogo4cjawyn1nvc.png)


The core structure underneath is the Key-Value store.
All the data structures stored in Redis are stored in keys: be it Strings or complex data structures like Sorted Set.

---

## Infrastructure as configurations:


![Infra](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/lx70icaz45pz7lb09faw.png)


Redis can run as a single node with an HA (highly available) replica or as a cluster.
When operating as a cluster, Redis client cache hash slots that map the key to the specific node that way client can connect to a specific node that has the data that the client is requesting, if operating with a limited no. of nodes in a cluster, request to a wrong node can be directed to correct node via a gossip protocol, but since Redis priorities performance, the priority is to hit the correct node first.

As compared to other databases, Redis clusters are surprisingly basic hence they come with severe limitations, so rather than solving the scalability issue for you, it provides you with some basic primitives that you can use to solve them.
￼



## Performance:
it can handle 100k of write requests/seconds and read latencies are often in microseconds, This scale makes some anti-patterns for other database systems actually feasible with Redis, like querying the db 100 times to get a list of results in SQL db is a terrible idea, you're better off writing a SQL query which returns all the data you need in one request. On the other hand, the overhead for doing the same with Redis is rather low - while it'd be great to avoid it if you can, it's doable.
This is completely a function of the in-memory nature of Redis. It's not a good fit for every use case, but it's a great fit for many.

---

## Use cases of Redis:

**As caches**:

![cache](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/1fmckci0nmj5aptppl0t.png)


Since it is in memory it can be used as a lighting-fast cache, 
The root key and values are mapped to the Redis key and values, and Redis can distribute this hashmap across all its nodes in the cluster enabling us to scale without much fuss. If we need more capacity we simply add more nodes.
The server hits the cache first before hitting the DB to get the result faster, technique such as least recently used policy can be used for replacement, or when using Redis as cache a TTL (time to live ) timer is associated with each key, Redis guarantees that you will never read the value of a key after its TTL has expired and TTL is used to decide which item to evict from the server- keeping the cache size manageable even in the case where you're trying to cache more items than memory allows.
Issues ? what if the key becomes hot( not only Redis but also other caches like Memcached, or dynamoDb also face the same issue)

**Redis as Distributed lock**:
Occasionally we have data in out system that we wanna make sure is consistent during updates, in a distributed setting.
Most databases including **Redis will guarantee some consistency if your core database can provide consistency. don’t rely on distributed locks for consistency as it is not guaranteed and it may introduce extra complexity and issues**.

Using atomic increment(INCR) with TTL, when we want to try to acquire the lock we increment INCR if the response is 1 means we can get the lock on the data, but if the response is >1 it means it is already been locked by some other process, we wait and retry again, once we are done we can delete the INCR key ( will be reset to 0), So that others can use it.


**Redis as a rate limiter**:

It provides some primitive data structure likes hash, SortedSet, List, or objects to store data, we can use Redis sorted set to store the timestamp of each request of a given use, and depending on the window size(period window can be sliding/fixed window, assuming we are using sliding window) if the new request of the same user exceeds the allowed no.of request in a given window then that request can be dropped.
Read more about rate limiter how it works and how Redis can be used as a [rate limiter](https://dev.to/prashantrmishra/design-rate-limiter-42oc), rate limiter use case in [web crawler](https://dev.to/prashantrmishra/design-web-crawler-4cg5)


**Redis in ranking/leader board system**:

Redis sorted set keeps ordered data that can be queried in log time which makes then useful in leaderboard applications. The high write throughput and low read latency make them very useful for scaled applications where something like SQL db will start to struggle.

For ranking the search results Redis can be used to build an index that can be used to rank the results and then return the results back to the user.
For example, creating an index for twitter search, we have to find the post that has a given keyword in it and rank the result based on no. of likes.
We can use the Redis sorted set to maintain the list of top-liked posts for the given keyword, periodically we can remove low-ranked posts to save space.



**Redis stream for event sourcing**:

![redis-stream](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/5d78tfv8ibxahp1c5p72.png)


Redis has the feature of steam where append-only messages are added to the log/file and workers( part of the same consumer group) can consume the messages.
A consumer group is nothing but a pointer in the stream of messages/data that is being consumed.
The basic idea behind Redis streams is that we want to durably add items to a log and then have a distributed mechanism for consuming items from these logs
 Redis solves this problem with streams (managed with commands like `XADD`) and consumer groups (commands like `XREADGROUP` and `XCLAIM`).

￼
A simple example is the work queue, we want to add items in the queue and have them processed, at any point in time one of our workers might fail, at these instances we want to reprocess them once the failure is detected.
The Redis steam adds items to the queue using `XADD` and has a single consumer group of workers attached to the steam to process them, and when worker failure occurs Redis stream provides a way for a new worker to `XCLAIM` and restart the processing the messages/items in the queue.

[Redis for proximity search](https://youtu.be/fmT5nlEkl3U?t=1412)
Redis natively supports geospatial index with commands like GEOADD, GEORADIUS
If you have a list of items that have locations and you want to search them by locations then it is good to use a geospatial index for the same.

```
GEOADD key longitude latitude member # Adds "member" to the index at key "key"
GEORADIUS key longitude latitude radius # Searches the index at key "key" at specified position and radius

```
The search command, in this instance, runs in O(N+log(M)) time where N is the number of elements in the radius and M is the number of members in our index.

---
## Issues and remediation:

**Hot key issues**:


![Hot-key-issue](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/zedvz4md2yl2o8k3n0hp.png)

Assume we are an e-commerce application, and we are using Redis to cache the details of items, we have a lot of items hence we have 100 nodes in a cluster, and their details are evenly distributed across the cluster, so far so good.
What if there is a surge in requests related to one item, then the node having the details of the item will be overwhelmed with requests, and unless we are not over-provisioned (i.e. we were only using a small percentage of overall CPU) this server/node will start failing.
￼
**Solutions with trade-off**
- Have read replica and dynamically scale this with load
- In-memory cache on our client so that they are not making so many requests to Redis for the same data.
- Store the same data in multiple keys ( that are used to locate the node where the requesting data is present) and randomize the requests so that they are spread across multiple nodes in the cluster.

---

Questions:
[How redis handles concurrent request with single thread ?](https://codescoddler.medium.com/how-redis-achieve-concurrent-operation-with-single-thread-e0c8d5e33bc3#:~:text=Redis%20operates%20on%20a%20single,processes%20them%20one%20by%20one.)
