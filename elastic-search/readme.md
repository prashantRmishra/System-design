
# Elastic Search

[Potential Potholes in Elastic search](#potholes)

[Nodes in Elastic search](#nodes-in-elastic-search)

[Shards and replicas](#shards-and-replicas)

[What is inside a shard?](#what-is-inside-of-the-shards)

[How updates/deletes work in Elastic search?](#how-we-support-updates-in-elastic-search-and-how-that-works-from-lucene-perspective-)

[How documents are stored in Lucene segments?](#how-documents-are-stored-inside-the-lucene-segments)

[How the search queries are matched](#how-to-match-the-search-query)
[Activity diagram of Elastic search working](#a-walk-through-the-entire-layer-of-the-onion)



![](./image%20copy%204.png)

Used for complex search scenarios like if you need to do GeoSpatial indexes, Vector searches, if you need to do simple full text search, then Elastic search is your best friend.

---

## Potholes

- **Not your primary database**
  - while the apis of Elastic search looks a lot like typical document database but Elastic search's durability and availability guarantees are not actually that good an over time most people have relied on setting up a primary data source be it Postgres, DynamoDB and then using Change Data Capture (CDC) to push data from database into elastic search, the idea is the elastic search is going to be little bit behind your database but you will have all the updates
- **Best with read heavy workloads**
- **Eventual consistency**
  - If you are using Elastic search you will have to be able to tolerate eventual consistency as it is going to have a delayed version of data.
-  **De-normalization is the key to making the Elastic search functional**
   -  If you got a particular search page make sure you flatten all of your data and that you are not trying to make nested relational queries.
   -  You basically shouldn't need to do joins in an Elastic search search. If you do you are gonna be in trouble because joins are not supported.
- **Do you even need it?**
  - In a lot of cases a simple search capability in you database is enough, if you need text search, if you need to index billions of documents Elastic Search is you best friend.
  - If you got simplistic needs aim for simple solutions.

---

**Elastic search provides APIs to search/add/update/delete records/documents,these APIs are pretty straight forward to understand**

---

Elastic search is built on top of **Lucene** (Apache lucene) that provides simple search capability, and Elastic search is an orchestration layer on top of that meaning Elastic search handles the distributed systems aspects.

  - How to use APIs
  - How to manage nodes
  - How to manage backups
  - How to keep availabilities and replication
  
Lucene is going the insure that the data is organized and it is fast, it operates on a single node, Elastic search operates at the cluster level

---

## Nodes in Elastic search
Various nodes that make up the elastic clusters (note: these nodes may not necessarily have to be physical servers, meaning elastic search cluster can have a single host or server)
Hence a physical machine can take on the responsibility one or more nodes.

- **Coordinating node**
  - This is kind of API layer of the cluster.
  - The wast majority of the request that are coming in the cluster are going to be search request, the coordinating nodes take those request, parse then and send them to the node that has the relevant data that we need in order to return the search result.
  - Will have lof of network traffic with outside world
- **Master node**
  - We want to create only one master node that can make administrative decisions like when we create indexes or which nodes are part of the cluster.
  - It has to be robust and reliable, it should not go down.
- **Machine learning node**
  - They are used for machine learning intensive tasks, requiring access to GPU and will have very different access patterns.
- **Data node**
  - This is where he indexes are stored, this is functionally where we keep out documents.
  - The need to have really heavy IO (lots of memory, plenty of fast disks)
  - The coordinating nodes are going to send search request and they expect the data nodes to be able to respond.
- **Ingest node**
  - It is like the backend of the our server, this is how the documents make their way into the cluster.
  - They are more CPU bound and they will do a lot of analysis on the incoming documents.


## Shards and replicas

- The fundamental grouping of Elastic search document is index, this might be the book index or review index.
- Inside of the index we have shards and replicas, these shards are going to responsible for containing all the documents but also building up index data structure for making search fast.
- The shards are mutually exclusive i.e. if  I got a document it will be assigned to one document.
- The replica are going to be carbon copy of the shards, we might have replica of a shard on multiple machines and while searching the Coordinating node can choose to either search in shard or in their replicas to distribute the load.
- Hence by replicating the shards we get increased throughput on the read side.
- By sharding we can spread out very large index across many different machines.
- A shard has a hard limit of 2Billion documents, but even if we had smaller number than that we still want to divide those documents either because
the aggregate size is too large for a single machine or I want to spread the load across the cluster to make my query faster.

![](./image%20copy%202.png)

## What is inside of the Shards?

- Shards are responsible for breaking down the data and inside the Shards we have Lucene Index, The lucene index are one to one with Shards, basically a Shard is encapsulating a lucene index.
- Inside the Lucene Index we have Segments, basically the way Segments works is by accumulating the immutable segments.
- When we create a new document, lucene tries to batch them in as much write as possible before flushing them out to create a segment with multiple documents.
- When new documents are added later a new segment might be created and in some point in future Lucene may decide it will merge the segments to create a third segment and then delete the old ones.
- The important thing is these segments are immutable and are not changing.


## How we support updates in Elastic search and how that works from Lucene perspective ?

- With updates what we are doing is,we are doing soft delete, lucene inside the segment maintains a map of deleted documents
- Example if the segment 2 contains documents 1,2 and 3 and we deleted the document 3 then segment 2 will store that identifier too internally and when we query that segment it will remove any result that contains 2(2nd document)
- When we are doing update we are going to mark soft delete the document in the old segment and add the updated document in the new segment. This maintains the immutability aside from the map of deleted documents these segments are not changing.
- By these we can exploit bunch of caching and concurrency benefits since segments are not necessarily changing when we load them from memory we can do that in compressed format and we can load them all at once as we know they are not changing, we can also have multiple readers touching the data in the segments in a concurrent way without having to worry about the race condition.


## How documents are stored inside the lucene segments?

- We want to store documents in the segments such that the searching is very fast, we don't want to just dumbly store the documents in the segment.
- There are two ways with which we could store the data
  - Like if we store the documents in random fashion in an array in the memory then we will have to search in the entire array `O(n)` times, similarly if we sort the document/data in the array then we can search in `O(logn)` time, and if we hash out data then we might be able to access it O(1) time.
  - If we can make a copy of out data and store it in slightly different way then we can make potentially make access to data really fast (of corse there is a trade of storage cost) like using **Inverted Index**

![](./image.png)

There are a lot of complexity involved while creating the index e.g like the words like 'are' might be present in almost all the documents so we might not want to include that word in our index. This is fundamentally a question of tokenization, stemming or limitization.


## How to match the search query?
Sometimes the search query is such that it wants the results to be sorted on the basis of price(taking example of books index), but since our index is giving us the list of ID's of the document that has the keyword present in the search query, how do we transform it to get the results sorted by the price ?

We can think of something like columnar data store where fixed length column value are stored that can be concatenated together to get the desired result.
What columnar type storage does is it allows us to get single of small number of field values across all the documents very quickly. So if we have large number of docIds we can grab all of their prices without having to worry about other fields values ( like iterating or crawling over all the rows of the table grabbing each row (in sql database)).


![](./image%20copy.png)

In lucene this is implemented as an Index called **Doc values** and doc values contains these columnar fields that allow it to do very quick sorting and filtering based on those docIds that are identified by those **Inverted Index**.
Lucene has very low level optimizations for how data is organized in order to make the queries as fast as possible



## A walk through the entire layer of the onion

![](./image%20copy%203.png)