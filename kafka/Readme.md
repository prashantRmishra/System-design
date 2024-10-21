
![](./image.png)
![](./image%20copy%202.png)
![](./image%20copy%203.png)
![](./image%20copy%203.png)
![](./image%20copy%204.png)

### Terminologies
**Brokers**: Servers(physical or virtual) that hold the "queue"
**Partitions**: The "queue" an ordered, immutable sequence of messages that we append to, like a log file. Each broker can have multiple partitions.
**Topics**: Logical groupings of partitions, you publish to and consume from topics in kafka
**Producer**: Write messages/records to topics
**Consumer**: Read messages/records off of topics

### Message/record structure:
- Header
- Key
- Value
- Timestamp

### Create Producer and send message to kafka partition
- create producer
- specify topic param
- specify message object having (key, value) pair
- send
- Kafka assigns the message to correct topic, broker, and partition.
- There is a controller in kafka cluster that keeps mapping of brokers for each partition 
- Messages are appended to partition via append only log file.
- Each message has an offset, consumer consume message by specifying the offset of the message that they last read.


![](./image%20copy%205.png)

## Consumer reads next message based on offset
- Kafka maintains the latest offset of message that was read.
- Consumer reads message by offset, kafka gives the latest message to read.
- Consumer commits offset after successfully consuming the message back to kakfa, in case of failure resume reading from the last committed offset.








Usecases:
    messaging queue
    for async communication
    write heavy applications
    
Places where it is used, steaming realtime data
like in crickbuzz
For queueing applications like ticketmaster, web-crawler,





partitions: are the files where append immutable messages are written and are read by consumers.


Motivating example:
Streaming real time data of app like crickbuzz/football_related platform

if we are getting a lot of messages/records to write and the brokers capacity is reached then it might go down
, we will create more servers/queue/files to append data/messages/
now we are getting a lot of messages that are not ordered and consumers are getting those messages as and when then are reading it 
which are also not in order, so an event happening at at t1 could be read after a event happen at time t2 < t1 which is not 
correct as it is leading to confusion.

We want to maintain some order
Hence for this logical partitioning is required called topics:
hence all the data related to a match between two countries are written to one topic queue and consumer will
read from there in order that way there won't be any inconsistency.


How are we avoiding duplicate read
Each message is associated with an offset, and once a message is consumed the consumer after reading it successfully commits the offset back to kafka,
that way once an offset is committed other consumer in the group will know that this offset has been consumed and will not be consumed again.
lets take an example of web-crawler consumer of web-crawler reads the url from the message queue and fetches the html page
and only after the html page is stored in persistence storage like s3, it commits back to kakfa the offset.


non functional requirements:
Scaling----> 
How scaling of kafka works?
Adding new kafka queue in the broker
first we have to think of key using which we want to distribute the write/update/ messages in the kafka queue/server on the broker in kafka cluster
we don't want a single queue in the broker to be overwhelmed by most of the writes there has to be some uniformity

We know each topic specifies a domain, let say a topic call follball will have queues of message of various football matches
Let say we have one match between France and Germany ( this is one key) So based on the key we will pass it through a hash function to get
has value % N to get the queue where the message should be appended.

if we are getting more messages that usual the queue, we will either have to increase/spin off more consumers in the consumer group.
or we can notify the produces to snow down the production of messages as the consumption is not as fast as it is producing the messages.


Fault tolerance:
How durability is ensured in kafka ?
it create leader and follower queue for each queue in a given topic,
and we can set policies to sync messages between these leaders and followers, as soon a leader goes done the follower will take its place.
the leader can wait for the sync ack from all the followers of it can wait ack from some of the followers before it continues to allow the consumer  to consume the message,
now lesser the no. of acks it needs lesser will the latency of kafka. So it is use case dependent. and more the no. of followers synced with the leader queue, more will be durability percentage.


What if there is some issue at consumer side ?
like in case of web-crawler if the the URL that the crawler is hitting is down/not responding, in this case we can have in memory timer for back off before we try again, but what if the crawler fails it self,
then the timer will also be lost, we need some efficient way for exponential back off, we can use kafka for that( but kafka does not support exponential back off out of the box)
We will have to implement some sort of logic for the backOff, like if the URL is down set the backoff time to 30seconds and put the msg back in retry queue and any other consumer/crawler reading from the 
retry queue will retry after the backoff time, and if it still fails then the crawler will updated the backOff time exponentially like 1min, 2min, etc and put in back in the retry queue,
but we don't want to try infinitely once we have have retried for let say some threshold no. of times we will remove the url/message from the retry queue and put it in a deal letter queue( DLQ) and no consumer will
read from that queue.
This approach is alright but seems like a hassle, instead we can use Amazonsqs that supports exponential backoff out of the box,
So it has visibility timer associated with each message, and if it is failed to be consumed by a consumer then 
the visibility timer will increase exponentially, as long as the visibility timer is not elapsed the message/url will not be visible to any other consumer/crawler.


What if the consumer goes down ?
in this case another consumer will take up that tasks of consuming the message, since each consumer commits the offset of message/url it consumes, failure of consumer will allow another consumer to pick the last message that is not committed by the dead consumer.




