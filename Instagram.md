Designing Instagram

1. Requirement Gather
   1. Scale of the system
   2. Considering videos with images?
   3. adding tags

2. Functional Requirements
    1. User able to upload/view photos
    2. Search photos based on title
    3. User can follow othe users
    4. News Fees consisting of latest photos by people user follows

3. Non Functional Requirements
    1. Highly available - system should work even in case of a component failure
    2. Efficient Storage management. Data should never be lost
    3. Accepted latency is 200ms for newsfeed generation
    4. consistency can take a hit

4. Capacity Estimation

    1. 500M total users, with 1M daily active users
        1 user uploads  - 2 photos per day - 2M photos/day
        Avarage photo size = 200Kb
        2M * 200KB => 400 GB
    2. total space for 5 years = 400 * 5 * 365 = 730 TB

5.  API Designs
    1. POST/ uploadPost
        parama: api_dev_key, user_id, image_name, description, file, 
        Response : 'succes' or error code 500

    2. GET/ viewPost | post_id, user_id
    3. GET/ getFeed | user_id, count, from, key
    4. GET/ searchPost | query, sort_order
        

6. DB Schema

    user: user_id(PK), name, email, lastLogin
    Post: image_id, user_id, description, photo_path, created_at, permision
    Follow: user1, user2
    

7. Algorithm and HLD
        client1  ------ upload photo -----------> Image Hosting 
        client2  ------ view/search image ------> Image Hosting ------>S3
                                                                -------> Image Metadata

8. Component Design 
    Issue: 
    Uploading users can consume all the available connections, as uploading is a slow process. This means that ‘reads’ cannot be served if the system gets busy with all the write requests. We should keep in mind that web servers have a connection limit before designing our system. If we assume that a web server can have a maximum of 500 connections at any time, then it can’t have more than 500 concurrent uploads or reads. 
    
    Solution 1 : 
    1. To handle this bottleneck we can split reads and writes into separate services. We will have dedicated servers for reads and different servers for writes to ensure that uploads don’t hog the system.
    2. Separating photos’ read and write requests will also allow us to scale and optimize each of these operations independently.

9. Reliability and Redundancy
    Issue : Losing files is not an option for our service
    Solution: 
    1. Therefore, we will store multiple copies of each file so that if one storage server dies we can retrieve the photo from the other copy present on a different storage server.
    2. Creating redundancy in a system can remove single points of failure and provide a backup or spare functionality if needed in a crisis

10. DB Partition

    1. Partitioning based on UserID: All photo of same user in 1 shard.  one DB shard is 1TB. shard number by UserID % 10
        Issues:
        1. Uneven Distribution : How would we handle hot users? Several people follow such hot users and a lot of other people see any photo they upload.
        Some users will have a lot of photos compared to others, thus making a non-uniform distribution of storage.
        2. Latency if data in diffrenet shards: What if we cannot store all pictures of a user on one shard? If we distribute photos of a user onto multiple shards will it cause higher latencies?
        3. unavailability: Storing all photos of a user on one shard can cause issues like unavailability of all of the user’s data if that shard is down or higher latency if it is serving high load etc.

    2. Partitioning based on PhotoID : “PhotoID % 10”,


11. Caching

    1. How much Storage -- 80-20 Principle - 70 GB --- normally single server can hold - 256 GB -- only 1 server required
    2. Cache Eviction policy --- LRU 
    3. To further increase the efficiency, we can replicate our caching servers to distribute the load between them.
    4. Update Cache Replicas ----> if cache miss --> hit db --- > before returning set cache
                             ----> if a replica has it can ignore,  other replica will add a entry


12. LB 

13. Replication 