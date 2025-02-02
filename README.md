# System-design
This repo has System design concepts for various application, which is very useful for system design interviews
I have created these as a reading material for me as well as for anyone who is interested in learning System Design concepts.


## Some common pointers to keep in handy

### AWS instance 
Top aws network optimized instances can handle 400Gbits/s of data.
And modern server and operating systems can handle 100k to 1M concurrent connections listening on a given port. Realistically, system resources like CPU, memory, and file descriptors become the bottleneck before you hit any theoretical limit

An average ec2 instance can handle on an average of 1000 requests/second ( this is reasonable enough assumption) 

### Aws managed api gateaway
It has 10mb size limit on the POST body size, this restriction is in place to insure that the requests body size does not exceed a specific limit.

### [Postgres](important-concepts/PostGresPerformanceAndSearchLatencyForSpatialQueries.md)
**1k  to 4k request/s** is the capacity of highly optimized **Postgres** instance.

We can assume modern aws managed postgresDb can store upto **100TB** of data

### [Redis](redis/Readme.md)
**400k to 1M request/second** is the capacity of highly optimized **Redis** in-memory cache

### [Elastic search](elastic-search/readme.md)
A shard in **elastic search** has a hard limit of **2Billion** documents

### [Kafka](kafka/Readme.md)
Each broker can **store up to 1tb of data** and can handle **10k request/sec**
![kafka cluster](<kafka/image copy 6.png>)
