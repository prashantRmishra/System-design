# System-design
This repo has System design concepts for various application, which is very useful for system design interviews
I have created these as a reading material for me as well as for anyone who is interested in learning System Design concepts.


## Some common pointers to keep in handy
### Postgres
**1k  to 4k request/s** is the capacity of highly optimized **Postgres** instance

### Redis
**400k to 1M request/second** is the capacity of highly optimized **Redis** in-memory cache

### Elastic search
A shard in **elastic search** has a hard limit of **2Billion** documents

### Kafka
Each broker can **store up to 1tb of data** and can handle **10k request/sec**
![kafka cluster](<kafka/image copy 6.png>)