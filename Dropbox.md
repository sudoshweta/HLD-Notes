PS: Design Dropbox or Google Drive or Amazon S3 Bucket
1. cloud storage service provided by cloud service providers to store files to be accessed through internet
2. simplify storage and sharing of files among multiple devices and different geographical locations
3. Advantages: Availability, 100% reliability and durability of data, Scalability

Requirements gather:
Scale of the system
Size of Files
Sharing Files
offline editing

1. Functional Requirememts
    1. Upload/download file
    2. Able to share files or folders with other users
    3. Able to upload large files
    4. Support offline Editing and as soon as the user comes online all their changes should be synced with remote servers
    5. Service Should support automatic synchronization among devices

    Extended Requirements
    1. The system should support snapshotting data, so that users can go to any version of the files.

2. Non Functional Requirements
    1. Huge read and write 
    2. Internally, the files can be stored in small parts or chunks(like 4MB).if failed only this chunk to be processed again
    3. Reduce data exchange by tranferring updated chuncks only
    4. Keeping a local copy of the metadata(file name, size, etc) with the client can save round trips to server
    5. For small changes, clients can intelligently upload the diffs instead of whole chunk. 


3. Capacity Estimation
    1. 500M total users --> 100M active users
    2. each user has 200 files/photos
    3. total storage = 100KB * 200 * 500M = 500 * 100  * 200 GB = 10000TB = 10 PB
    4. connection per minute  - 1M


6. Algorith and High Level System Design 

    1. Block servers: 
        1. The user will specify a folder as the workspace on their device. Any file/photo/folder placed in this folder will be uploaded to the cloud, and whenever a file is modified or deleted, it will be reflected in the same way in the cloud storage.
        2. Block servers will work with the clients to upload/download files from cloud storage 
    2. Metadata servers: 
        1. At a high level, we need to store files and their metadata information like File Name, File Size, Directory, etc., and who this file is shared with. 
        2. will keep metadata of files updated in a SQL or NoSQL database. 

    3. Synchronization servers 
        1. will handle the workflow of notifying all clients about different changes for synchronization.
        2. mechanism to notify all clients whenever an update happens so they can synchronize their files.


7. Detailed Component Design 

    1. Client :  Client Application monitors the workspace folder on the user’s machine and syncs all files/folders in it with the remote Cloud Storage. The client application will work with the storage servers to upload, download, and modify actual files to backend Cloud Storage. The client also interacts with the remote Synchronization Service to handle any file metadata updates, e.g., change in the file name, size, modification date, etc.
        1. Upload and download files.
        2. Detect file changes in the workspace folder.
        3. Handle conflict due to offline or concurrent updates.

    Issues: 
    1. How do we handle file transfer efficiently? 
       1.  we can break each file into smaller chunks so that we transfer only those chunks that are modified and not the whole file. Let’s say we divide each file into fixed sizes of 4MB chunks
       2. In our metadata, we should also keep a record of each file and the chunks that constitute it.
    
    2. Should we keep a copy of metadata with Client?
        1. Keeping a local copy of metadata not only enable us to do offline updates 
        2. but also saves a lot of round trips to update remote metadata. 

    3. How can clients efficiently listen to changes happening with other clients?
        1. Solution 1: Pull Method : the clients periodically check with the server if there are any changes
            Problems: 
            1. a delay in reflecting changes locally as clients will be checking for changes periodically compared to a server notifying whenever there is some change
            2. client frequently checks the server for changes, it will not only be wasting bandwidth
            3. the server has to return an empty response most of the time, but will also be keeping the server busy
            4. therefore not scalable

        2. HTTP Long Polling:
            1. With long polling the client requests information from the server with the expectation that the server may not respond immediately. If the server has no new data for the client when the poll is received, instead of sending an empty response, the server holds the request open and waits for response information to become available. Once it does have new information, the server immediately sends an HTTP/S response to the client, completing the open HTTP/S Request. Upon receipt of the server response, the client can immediately issue another server request for future updates.

        Based on the above considerations, we can divide our client into following four parts:

        I. Internal Metadata Database will keep track of all the files, chunks, their versions, and their location in the file system.

        II. Chunker will split the files into smaller pieces called chunks. It will also be responsible for reconstructing a file from its chunks. Our chunking algorithm will detect the parts of the files that have been modified by the user and only transfer those parts to the Cloud Storage; this will save us bandwidth and synchronization time.

        III. Watcher will monitor the local workspace folders and notify the Indexer (discussed below) of any action performed by the users, e.g. when users create, delete, or update files or folders. Watcher also listens to any changes happening on other clients that are broadcasted by Synchronization service.

        IV. Indexer will process the events received from the Watcher and update the internal metadata database with information about the chunks of the modified files. Once the chunks are successfully submitted/downloaded to the Cloud Storage, the Indexer will communicate with the remote Synchronization Service to broadcast changes to other clients and update remote metadata database.

        Should mobile clients sync remote changes immediately? 
        Unlike desktop or web clients, mobile clients usually sync on demand to save user’s bandwidth and space.



        b. Design DB Schema

        The metadata Database should be storing information about following objects:

        Chunks
        Files
        User
        Devices
        Workspace (sync folders)

        The Metadata Database can be a relational database such as MySQL, or a NoSQL database service such as DynamoDB. Regardless of the type of the database, the Synchronization Service should be able to provide a consistent view of the files using a database, especially if more than one user is working with the same file simultaneously. Since NoSQL data stores do not support ACID properties in favor of scalability and performance, we need to incorporate the support for ACID properties programmatically in the logic of our Synchronization Service in case we opt for this kind of database. However, using a relational database can simplify the implementation of the Synchronization Service as they natively support ACID properties.

        c. Synchronization Service
        
        1. The Synchronization Service is the component that processes file updates made by a client and applies these changes to other subscribed clients. It also synchronizes clients’ local databases with the information stored in the remote Metadata DB. 
        2. Synchronization Service is the most important part of the system architecture due to its critical role in managing the metadata and synchronizing users’ files. Desktop clients communicate with the Synchronization Service to either obtain updates from the Cloud Storage or send files and updates to the Cloud Storage and, potentially, other users. 
        3. If a client was offline for a period, it polls the system for new updates as soon as they come online. 
        4. When the Synchronization Service receives an update request, it checks with the Metadata Database for consistency and then proceeds with the update.  
        
        
        5. Subsequently, a notification is sent to all subscribed users or devices to report the file update.


        2. The Synchronization Service  DEsign 
            1. The Synchronization Service should be designed in such a way that it transmits less data between clients and the Cloud Storage to achieve a better response time. 
            2. To meet this design goal, the Synchronization Service can employ a differencing algorithm to reduce the amount of the data that needs to be synchronized. Instead of transmitting entire files from clients to the server or vice versa, we can just transmit the difference between two versions of a file. Therefore, only the part of the file that has been changed is transmitted. This also decreases bandwidth consumption and cloud data storage for the end user. 
            3. As described above, we will be dividing our files into 4MB chunks and will be transferring modified chunks only. Server and clients can calculate a hash (e.g., SHA-256) to see whether to update the local copy of a chunk or not. 
            4. On the server, if we already have a chunk with a similar hash (even from another user), we don’t need to create another copy, we can use the same chunk. This is discussed in detail later under Data Deduplication.

            5. To be able to provide an efficient and scalable synchronization protocol 
            
            we can consider using a communication middleware between clients and the Synchronization Service. 
            
            6. The messaging middleware should provide scalable message queuing and change notifications to support a high number of clients using pull or push strategies. This way, multiple Synchronization Service instances can receive requests from a global request Queue, and the communication middleware will be able to balance its load.

        d. Message Queuing Service #
        1. An important part of our architecture is a messaging middleware that should be able to handle a substantial number of requests. A scalable Message Queuing Service that supports asynchronous message-based communication between clients and the Synchronization Service best fits the requirements of our application.
        2.  The Message Queuing Service supports asynchronous and loosely coupled message-based communication between distributed components of the system. The Message Queuing Service should be able to efficiently store any number of messages in a highly available, reliable and scalable queue.

        The Message Queuing Service will implement two types of queues in our system. 
        
        1. The Request Queue is a global queue and all clients will share it. Clients’ requests to update the Metadata Database will be sent to the Request Queue first, from there the Synchronization Service will take it to update metadata. 
        
        2. The Response Queues that correspond to individual subscribed clients are responsible for delivering the update messages to each client. Since a message will be deleted from the queue once received by a client, we need to create separate Response Queues for each subscribed client to share update messages.


        e. Cloud/Block Storage #
        Cloud/Block Storage stores chunks of files uploaded by the users. Clients directly interact with the storage to send and receive objects from it. Separation of the metadata from storage enables us to use any storage either in the cloud or in-house.


8. Data Partitioning


9. Caching

We can have two kinds of caches in our system. To deal with hot files/chunks we can introduce a cache for Block storage. We can use an off-the-shelf solution like Memcached that can store whole chunks with its respective IDs/Hashes and Block servers before hitting Block storage can quickly check if the cache has desired chunk.

LRU


10. LB


11. Reliability and Redundancy 



4. Design Apis

POST/  addFile
POST/ addChange


7. File Processing Workflow

Client A uploads chunks to cloud storage.
Client A updates metadata and commits changes.
Client A gets confirmation and notifications are sent to Clients B and C about the changes.
Client B and C receive metadata changes and download updated chunks.


12. Security, Permissions and File Sharing
 To handle this, we will be storing the permissions of each file in our metadata DB to reflect what files are visible or modifiable by any user.