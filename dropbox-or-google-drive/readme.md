# Design dropbox/google drive

*cloud-based file storage service, that allows users to store and share files across various devices*

- [Design dropbox/google drive](#design-dropboxgoogle-drive)
  - [Functional requirements](#functional-requirements)
  - [Non functional requirements](#non-functional-requirements)
  - [Core Entities](#core-entities)
  - [Api or Interfaces](#api-or-interfaces)
  - [High Level Design](#high-level-design)
  - [Deep Dive](#deep-dive)
    - [Support for huge files like 50GB](#support-for-huge-files-like-50gb)
    - [High data integrity (sync accuracy)](#high-data-integrity-sync-accuracy)
    - [Finally the updated API's will look like below](#finally-the-updated-apis-will-look-like-below)

## Functional requirements

upload files

download files

auto sync files across devices

## Non functional requirements

Availability >> consistency

Low latency uploads(depends on the file size and network speed, hence it should be as low as possible)

Support for huge files like 50GB
	- resumable uploads

High data integrity (sync accuracy)


## Core Entities

File(raw bytes)

FileMetadata (eg. filename, size, mime type, etc)

User(abstraction, it is more of an actor of the system)


## Api or Interfaces

*Note: below api’s are not totally correct and we will refine them later in the deep dives*

Upload
```
POST /file —> 200
body: File,FileMetadata
```

Download
```
GET /files/{fileId} —> File(s) and FileMetadata(s)
GET /changes?timestamp={}—> fileId[] (files that needs to be synced)
```

## High Level Design

![high-level](<Pasted Graphic.png>)

**Upload File**

Client will send request for file upload with file and metadata file will be uploaded to s3 bucket and the url of file (stored in s3 successfully ) and metadata will be stored in metadata table in primary db)

**Download File**

Client will send get file with fileId request to download the file, it will go to the file server which will get the s3 url of the file from the metadata and give it back to the client, client will directly download the file from the s3 server

**Sync Files**

It will require some changes in the high client design

￼![deep-dive](<Pasted Graphic 3.png>)

We will introduce another service called sync service which will receive the request of getChanges() and send the request to file-server to get the metadata of all the files that have been changed and finally the sync-service will return the metadata to the client and based on the metadata information of the file, the client will make the request to the s3 directly to get download the files that have been changed

We need to add two more attributes in the file table i.e creationTimestamp and updatedTimestamp to keep track of files that are changed.

This will also change the initial assumption of the file sync api and the updated api will look as below
```
GET /changes/files/{fileId} —> FileMetadata[]
```
At the client side we will have to keep track of changes made to remote folder and changes made to local folder

- **Remote changes**:
    - Pull changes
    - Download the new file and replace ( as we discussed above)
- **Local Changes**:
    - Upload the changed file to remote (will follow the same process of sending request to the file-server which will upload the file in s3 and metadata in metadata table db, we will also update the updatedTimestamp )
    - Operating Systems provides various api’s to keep track of changes in the given local folder
        - Windows provides FileSystemWatcher and MacOs provides FSEvents, these can be used to keep track of changes made in the local folder/directory
- **How do we know if the remote changed and we already don’t have those changes?**
    - We will also something called local db in on the client application that will keep track of id’s, some other info(metadata) of files in the local file system
    - When a new file is updated we can look into the local db in order to determine if we have already downloaded that fileId or not, from the fileId we will have the reference to the location of the file in our local folder

## Deep Dive

![deep-dive](image.png)

### Support for huge files like 50GB

If the current network speed of the client is 100megabits/s and we have to upload 50GB files

50 * 1000 *8* megabits/100megabits/s = 4000seconds = 1.12 hr will take to upload the file of size 50GB

How can we upload file of huge size like 50gb ?
We can not rely on the current approach as it is not efficient, in the current approach we are uploading the file to the file-server and the file server is then uploading the file to the s3 storage.

Notice we are uploading the file twice 1 on the file server then on the s3 which is redundant, also in practical sense where the file size is huge this approach will not work, as there is limit to the data we can put in the request body. Hence the request won’t go through the AWS managed api gateway itself and there is a limit to the body size of the request it can allow through which is 10MB (approx.)

**Solution**:
Upload the file directly to the s3 storage from the client
- The way we download the file from the s3 storage we can we not just upload the file to the s3  ?
- This can be done but how can we upload a 50gb file? This will take huge amount of time as it is dependent on the network speed of the client
- Solution
    - ***Chunk***: (fixed size segments or part of the original data in bytes)
    - We can divide the file (say 50gb file) into fixed size chunks. When we have to upload the file, we can simply send request to the file-server with the metadata of the file including all the details and the chunks details as show in the diagram (assuming we are using dynamoDb which is denormalised and all the chunks[] of a file can be accessed in the File document (table in RDB), the file-server will add these details in the metadata collection and file-server will also send request to the s3 with details like file sizes, mime-type to get pre-signed url
    - Pre-signed url is generated by s3 for fixed duration of time like 30mins or so and this Pre-signed url is returned to the client, and finally the client can use this url to upload the files in chunks serially or paralleled depending on the network bandwidth.
    -  ***Issues***
        - How do we ensure to send or upload only the remaining chunks if failure occurs mid way during upload?
        - For this we can keep track of chunks that has been uploaded on the s3, we can add additional attribute in Chunks  in file document like ’status’ (with values like in-progress and completed once done uploading) and once a given chunk is uploaded in s3 we can update the status to completed (this can be done via change data capture of s3 with s3 notifications or Trust But Verify approach)
        - Once the connection is resumed we will simply fetch the metadata of the File being uploaded and we can compare all the chunks present locally with the chunks from the received response, after identifying the chunks that have status in-progress we can resume upload for only those chunks.
        - How do we identify which chunk is which in the response compared to local chunks correctly?
            - We can use hashing, when we create chunk (of bytes) we can pass the chunk to a hash function to get unique chunk hash, we can make this as the chunkId while updating in the File document, and later when we have to compare chunks to identify which chunk is which( in case of resume upload as we discussed) we will use this chunkId to correctly identify the chunk and upload only those having status ‘in-progress’
    - **How Trust But Verify will work here?**
        - We can not just rely on the client to update the file-server that a given chunk has been uploaded to s3 successfully and update the status of that chunk to ‘completed’  for the given File
        - We have to verify that update first. So, when the file-server receives an update from the client that a particular chunk has been updated, the file-server then asks the s3 is it true that this given chunks has been uploaded successfully? Based on the response from s3 the file-server will update the status of that chunk
        - This approach is called Trust But Verify 
        - This is a great solution. (if we use CDC change data capture then it will also work but will add additional complexity)
    - **How to update the status of chunk using change data capture?**
        - So when ever s3 is updated we can put the change in some sort of event stream like kafka  via s3 notifications and we can have  worker groups read from the event stream and update the file-server about the update and file-server can then update the chunk status to ‘completed’
        - As we can see this is a tedious process and we are far better off with the Trust But Verify Approach instead.

### High data integrity (sync accuracy)

We have to achieve two things

- **Sync to be fast** ( within reason obviously)
    - The client can periodically poll from the server to get new updates
    - We can play with the frequency and the intervals of polling
    - Other approaches are like web-sockets or SSE but that will overkill and that will introduce additional complexity
    - When we are updating a file in local we shouldn’t just dumbly download the whole file and replace the old in the local directly, we can leverage the chunk mechanism, we will have to add another attribute like updatedAt in chunks[] in the File document, and we can only download the chunks that have been updated in the remote directly (this we will know by downloading the metadata details of the file in local directory through polling )
    - And if a new file is added that is not present in local directly then we will have no option but to download the entire file
- **Sync to be consistent** i.e whatever is in local folder should be in remote and vice versa
    - As we discussed we will be polling the changes periodically (and this is file and it will work)
    - Another approach for consistent sync would be use some sort of event bus(this is overkill for this design but worth knowing)
      - We will use some streaming services like kafka and we will keep track of cursor (pointer to the stream that tells what was the last data or event we read)   
      - Note: kafka assigns unique offset for each event or data in the stream for unique identification that is what we can can set our cursor or pointer to.
      - We can store the cursor or pointer in a separate document like Folder as show in the image below
      - Whenever a new change occurs in a file after updating the db we will simply add the event in event bus/kafka, so very first time when we setup the local folder  (whose remote state already exists)(the cursor will be empty or null) then we will simply replay all the events from the event bus that has been added in the lifetime of that folder, by doing this we can construct the local folder with all the files
      - This can be done via sync-service which will communicate with the event bus instead of going to the db for all the file metadata and updates it will just get the cursor from the Folder document and use it to directly get the events that have come after the cursor pointer(if the cursor is empty replay will happen as we discussed)
      - This will come with some added complexity and we will have to partition based on userId or folderId at some point in time
- **Final Reconciliation**:
  - Even after our best efforts there is still a chance that our local folder is outdated or in-consistent with the remote folder
  - Hence periodically like daily or every couple of days or weekly the client app can fetch all the details for the given folder from its remote folder (from db) and client app can compare the response from with the help of its local db, compare the finger print of the file chunk present in the local folder and fetch or update the missing or outdated onces

![finaldeepdive](image-2.png)

### Finally the updated API's will look like below

upload file

```
//getting the presigned url
POST /files----> presigned URL from s3
body: FileMetadata

//uploading chunks to presigned URL
PUT {presigned URL from s3}
body: FileChunk

//updating the file-server that given chunk has been uploaded to s3 so file-server can update the status
//of the chunk to completed (after trust but verify)
PATCH /files ----> 200
body:Partial<FileMetadata> (chunk status updates)
```

download file(s)
```
GET /file/{fileId}---> File and FileMetadata
GET /changes?since={timestamp}---->FileMetadata[]
```
