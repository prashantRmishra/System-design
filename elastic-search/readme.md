
Used for complex search scenarios like if you need to do GeoSpatial indexes, Vector searches, if you need to do simple full text search, then Elastic search is your best friend.

Potholes
- Not your primary database
  - while the apis of Elastic search looks a lot like typical document database but Elastic search's durability and availability guarantees are not actually that good an over time most people have relied on setting up a primary data source be it Postgres, DynamoDB and then using Change Data Capture (CDC) to push data from database into elastic search, the idea is the elastic search is going to be little bit behind your database but you will have all the updates
- Best with read heavy workloads.
- If you are using Elastic search you will have to be able to tolerate eventual consistency as it is going to have a delayed version of data.
-  De-normalization is the key to making the Elastic search functional.
   -  If you got a particular search page make sure you flatten all of your data and that you are not trying to make nested relational queries.
   -  You basically shouldn't need to do joins in an Elastic search search. If you do you are gonna be in trouble because joins are not supported.
- Do you even need it ?
  - In a lot of cases a simple search capability in you database is enough, if you need text search, if you need to index billions of documents Elastic Search is you best friend.
  - If you got simplistic needs aim for simple solutions.


Elastic search is built on top of lucene (Apache lucene) that provides simple search capability, and Elastic search is an orchestration layer on top of that meaning Elastic search handles the distributed systems aspects.
  - How to use APIs
  - How to manage nodes
  - How to manage backups
  - How to kee`p availabilities and replication
Lucene is going the insure that the data is organized and it is fast, it operates on a single node, Elastic search operates at the cluster level.

Various nodes that make up the elastic clusters (note: these nodes may not necessarily have to be physical servers, meaning elastic search cluster can have a single host or server)
Hence a physical machine can take on the responsibility one or more nodes.