# Design YouTube



**Functional Requirements:**  
- Uploading videos  
- Watching videos  

**Non-Functional Requirements:**  
- Availability > Consistency  

**Extended Requirements:**  
(Not mandatory)

**Design Considerations / Capacity Estimation:**

**Traffic Estimation:**  
Let's assume there are 10 billion (100 crore) total users, out of which 1 billion (10 crore) are active users.  
For every video uploaded, let's assume there are 100 videos watched simultaneously.  
This gives us a ratio of 100:1 (100 reads per 1 write).  

From this analogy, if 5 videos are watched per day, it would mean that 5 * 1 billion = 5 billion (50 crore) videos are watched daily, with 5 million (50 lakh) video uploads.  
Since the read-to-write ratio is 100:1, writes account for 1% of total reads.

**Bandwidth Estimation:**  
Not all videos receive the same traffic. We can assume that only the top 5% of videos will receive 90% of the traffic.

**Storage Estimation:**  
Let's assume the average video size is 10 MB, and 5 million videos are uploaded daily.  
The total storage required daily would be:  
5M * 10 MB = 50,000,000 MB = 50 GB.  

For one month, it will be:  
30 * 50 GB = 1.5 TB.  

For one year:  
1.5 TB * 12 = 18 TB.  

If a YouTube channel remains active for an average of 30 years, the total required storage will be:  
30 * 18 TB = 540 TB.  
(These are just estimates; actual numbers would be much higher.)

**API Design:**  
- `upload(userId, videoUrl, title, description)`  
- `watch(requesterUserId, videoUrl)`

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
With 5 million videos uploaded daily, we get:  
5,000,000 / 24 = 208,333 videos per hour = 3,472 videos per minute.  

Since we are receiving 3,472 uploads per minute, we need at least 3,472 video encoder instances. Anything less will result in backlog on the queue, as the number of writes to the queue will eventually exceed the number of reads by the encoder process, causing delays in processing uploads.

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
