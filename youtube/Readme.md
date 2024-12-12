# Design YouTube



## Functional Requirements:**  
- Uploading videos  
- Watching videos  

## Non-Functional Requirements:**  
- Availability > Consistency  

**Extended Requirements:**  
(Not mandatory)

## Scale of the system
**Design Considerations / Capacity Estimation:**
**Traffic Estimation:**  
1B (100cr) total user and 100M DAU(10cr)  
1000:1 (read:write) ratio 
Which means 10^8/1001 ~= 10^5 write/day ~=66 write/minute (66 videos are created per minute)

**Bandwidth Estimation:**  
Not all videos receive the same traffic. We can assume that only the top 5% of videos will receive 90% of the traffic.

**Storage Estimation:**  
Let's assume the average video size is 10 MB, and 10^5 are uploaded daily ~= 10^6MB = 1TB ~=30TB(monthly) ~=360TB or 0.36PB (yearly)

## Core entities 
Video

User

## API or interface of the system

Upload video

```
POST /upload --> 200 success HTTP code
body{
    videoFile,
    Title,
    desc,
    Metadata
}
```

View video

```
GET /watch/[videoId]?start={start}&end={end} ---> next chunk of video from start to end
```

**Database Design:**  
For storing videos, we can consider Amazon S3 or similar cloud services that provide high availability and reliability.

User details and video metadata, including video URLs, can be stored in a NoSQL database like MongoDB. Since MongoDB is non-relational and denormalized, it will allow faster searches. Additionally, we will store video URLs that can be used to access videos stored in Amazon S3 or in a cache (discussed later).

We can have two collections in MongoDB:  
1. **User:** Stores user details in JSON format.  
2. **Video:** Stores video details such as the owner of the video, size of the video, user profile photo URL, and video URL, all in JSON format.

**Handling Joins in MongoDB:**  
What if a user wants to see a specific video of a content creator (another user)? Wouldn't this require a join between the `User` and `Video` collections?  
Not necessarily. Since MongoDB is denormalized, we can store duplicate data across collections. For each video, we can store the user metadata (like `userId` and `username`) within the video document itself, eliminating the need for joins.

**CDN (Content Delivery Network):**  
CDNs distribute static files geographically. When a user accesses a video, it is loaded via the CDN, which pulls the data from distributed encoded object storage.

**Cache:**  
For video metadata, users can access it directly from MongoDB, but it may be slow. To speed this up, we can use a cache like Memcached.

We will cache only the most popular or most viewed videos, since only 5% of videos generate 90% of the traffic.

For cache replacement, we can use policies like **Least Recently Used (LRU)**.

**High-Level Design:**

_Uploading_: 

![upload](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/h5mfdwotcbm0mxpaafw0.png)

**How many instances of video encoders are needed?**  
Let’s assume the video encoder takes 1 minute to encode a video.  
And we are getting 66 videos per minute to write.  

We need at least 66 instances of encoder, anything less will result in backlog on the queue, as the number of writes to the queue will eventually exceed the number of reads by the encoder process, causing delays in processing uploads.

_Watching_: 

![watch](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/y0gu82kb4m499qnmubea.png)
  
For watching videos, they will be cached. When a user requests a video, instead of delivering the entire video, only a chunk or part of it will be sent. As soon as the user is about to finish watching the buffered portion, another request will be sent to fetch the next chunk of the video, and so on.

Transmitting 1 MB of data is easier and faster than 10 MB, leading to reduced latency and less buffering.

**What protocol might we use?**  
We would use UDP (User Datagram Protocol) for live streaming, where losing a few seconds of video is acceptable since it is fed live. However, for YouTube videos, which are pre-recorded and stored, we want to ensure no data is lost. Therefore, it's better to use TCP (Transmission Control Protocol) for YouTube videos.

**TCP** is preferred for reliability, while **UDP** is favored for real-time fast data streaming.  
YouTube uses HTTP, which is built on top of TCP.

**Related Question:**  
- **Why use a NoSQL database? How is it faster than SQL databases?**
- How will you implement youtube feed? 
  - [Use the same approach as the facebook feed](https://github.com/prashantRmishra/System-design/blob/main/facebook-news-feed/Readme.md)
- How will you implement search feature in youtube?
  - You can use [elastic search to optimize the search query](https://github.com/prashantRmishra/System-design/tree/main/elastic-search)
