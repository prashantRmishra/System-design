# Caching

    Temporary storage that keeps recently used data handy so you get can get it must faster next time.

## Types
### External Caching
Cache is managed externally (a different server), all the servers(instances of the application) have the same global view of the cache.
![external_caching](image.png)

### In Process Caching
- Each App server has it's own cache, that means if one server caches something the others won't see it leading ot **inconsistency**.
- Could also lead to wastage of memory (due to **redundancy**).
![In_Process_caching](image-1.png)

### CDN (Content Delivery Network)
- Geographically distributed network of servers that can cache contents close to the users.
- Here we are not optimizing for the difference between memory and disk speed, instead we are optimizing for network latency.
- Example if the server is in U.S and the user is in India then the requested data will be fetched from the CDN closet to the user, and if the data is not available in CDN then the CDN will fetch the data from the actual server that is from U.S and then serve to the user.
- data includes but not limited to static files i.e images, videos, PDFs etc.

### Client Side Caching

- Data is cached on the client side, like caching in the web-browser or device that avoids unnecessary network cost.
- For web-apps it will be like HttpCache or LocalStorage and for native mobile apps it will be like storing in the memory or in Disk on the device.
- You have less control over it, like browser cache can be refreshed or disk data can be delete.
![ClientSide_Caching](image-2.png)

## Caching Strategies

### Cache Aside
- Caches only what the user has requested, so the data that the user has never requested will never make it to the cache.
- The only downside is added latency when there is a cache miss and we have to fallback to db to fetch the data, cache the same and return it back to the user.
- This is the most imp Cache strategy for interview perspective.

![cacheAside](image-3.png)

### Write Through
- Application server write the data directly to the cache and the cache synchronously writes the data in db, unless the data is written in the db the write is not considered complete.
- So, we need a framework of logic that automatically triggers the action of writing the data in db when the cache is updated. Caches like Redis and memecache they don't natively support this feature, so we have to write our own business logic for that.
- **Disadvantages**
  - Wait time for synchronous write in db.
  - Since we are writing everything in cache, we might bloat the cache with data that the user might never need.
  - If cache write succeeds but db write fails, it will lead to inconsistency, this will again add overhead of retry logic, etc.

![writeThrough](image-4.png)

### Write Behind or Write Back
- The application writes the data to the caches and those are flushed to db in batches in async fashion in the background.
- This makes writes much faster that the Write Through. but if the cache crashes or fails before the data is written back to DB then we will have data loss.
- This should only be used when some data loss is acceptable.
![writeBehind](image-5.png)

### Read Through
- Same as Cache Aside but instead of application server going to db to fetch the data when cache miss happens, cache itself goes to the db get the data updates the cache and return the data back to application server.
- Same approach is used in CDN

![readThrough](image-6.png)

## Cache Eviction Policies

### LRU(Least Recently Used)
- Evicts Items that haven't been used recently. Most common and balanced default.
### LFU(Least Frequency Used)
- Evicts items that are used lest often, even if accessed recently.
- Good for highly skewed access patterns, Like when some data is accessed more than other data (e.g, Celebrity profile viewed on instagram compared to a common person profile viewed).
### FIFO(First In First Out)
- Evicts the oldest item first.
- Simple, but rarely the right choice.

### TTL(Time-To-Live)
- Each item expires after a set time(e.g, 5 minutes).
- Great for data that can go stale, like API responses.
