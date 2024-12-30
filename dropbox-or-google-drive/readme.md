Design dropbox/google drive
Dropbox/google drive
cloud-based file storage service, that allows users to store and share files across various devices

What I failed to get in the first go
functional requirements

Functional requirements

upload files
download files
auto sync files across devices

Non functional requirements

Availability >> consistency
low latency uploads(depends on the file size and network speed, hence it should be as low as possible)
support for huge files like 50GB
	- resumable uploads
High data integrity (sync accuracy)


Core Entities

File(raw bytes)
FileMetadata (eg. filename, size, mime type, etc)
User(abstraction, it is more of an actor of the system)


Api or Interfaces

Note: below api’s are not total correct and we will refine them later in the deep dives

Upload
POST /file —> 200
body: File,FileMetadata

Download
GET /files/{fileId} —> File(s) and FileMetadata(s)
GET /changes?timestamp={}—> fileId[] (files that needs to be synced )

High Level Design
![high-level](<Pasted Graphic.png>)
Upload File
client will send request for file upload with file and metadata file will be uploaded to s3 bucket and the url of file (stored in s3 successfully ) and metadata will be stored in metadata table in primary db)
Download File
Client will send get file with fileId request to download the file, it will go to the file server which will get the s3 url of the file from the metadata and give it back to the client, client will directly download the file from the s3 server

Sync Files
It will require some changes in the high client design
￼![deep-dive](<Pasted Graphic 3.png>)
We will introduce another service called sync service which will receive the request of getChanges() and send the request to file-server to get the metadata of all the files that have been changed and finally the sync-service will return the metadata to the client and based on the metadata information of the file, the client will make the request to the s3 directly to get download the files that have been changed
We need to add two more attributes in the file table i.e creationTimestamp and updatedTimestamp to keep track of files that are changed.
This will also change the initial assumption of the file sync api and the updated api will look as below

GET /changes/files/{fileId} —> FileMetadata[]

At the client side we will have to keep track of changes made to remote folder and changes made to local folder

- Remote changes:
    - Pull changes
    - Download the new file and replace ( as we discussed above)
- Local Changes:
    - Upload the changed file to remote (will follow the same process of sending request to the file-server which will upload the file in s3 and metadata in metadata table db, we will also update the updatedTimestamp )
    - Operating Systems provides various api’s to keep track of changes in the given local folder
        - Windows provides FileSystemWatcher and MacOs provides FSEvents, these can be used to keep track of changes made in the local folder/directory
- How do we know if the remote changed and we already don’t have those changes?
    - We will also something called local db in on the client application that will keep track of id’s, some other info(metadata) of files in the local file system
    - When a new file is updated we can look into the local db in order to determine if we have already downloaded that fileId or not, from the fileId we will have the reference to the location of the file in our local folder

Deep Dive
![deep-dive](image.png)
Support for huge files like 50GB

If the current network speed of the client is 100megabits/s and we have to upload 50GB files

50 * 1000 *8* megabits/100megabits/s = 4000seconds = 1.12 hr will take to upload the file of size 50GB

How can we upload file of huge size like 50gb ?
We can not rely on the current approach as it is not efficient, in the current approach we are uploading the file to the file-server and the file server is then uploading the file to the s3 storage.

Notice we are uploading the file twice 1 on the file server then on the s3 which is redundant, also in practical sense where the file size is huge this approach will not work, as there is limit to the data we can put in the request body. Hence the request won’t go through the AWS managed api gateway itself and there is a limit to the body size of the request it can allow through which is 10MB (approx.)

Solution:
Upload the file directly to the s3 storage from the client
- The way we download the file from the s3 storage we can we not just upload the file to the s3  ?
- This can be done but how can we upload a 50gb file? This will take huge amount of time as it is dependent on the network speed of the client
- Solution
    - Chunk: (fixed size segments or part of the original data in bytes)
    - We can divide the file (say 50gb file) into fixed size chunks. When we have to upload the file, we can simply send request to the file-server with the metadata of the file including all the details and the chunks details as show in the diagram (assuming we are using dynamoDb which is denormalised and all the chunks[] of a file can be accessed in the File document (table in RDB), the file-server will add these details in the metadata collection and file-server will also send request to the s3 with details like file sizes, mime-type to get pre-signed url
    - Pre-signed url is generated by s3 for fixed duration of time like 30mins or so and this Pre-signed url is returned to the client, and finally the client can use this url to upload the files in chunks serially or paralleled depending on the network bandwidth.
    -  Issues
        - How do we ensure to send or upload only the remaining chunks if failure occurs mid way during upload?
        - For this we can keep track of chunks that has been uploaded on the s3, we can add additional attribute in Chunks  in file document like ’status’ (with values like in-progress and completed once done uploading) and once a given chunk is uploaded in s3 we can update the status to completed (this can be done via change data capture of s3 with s3 notifications or Trust But Verify approach)
        - Once the connection is resumed we will simply fetch the metadata of the File being uploaded and we can compare all the chunks present locally with the chunks from the received response, after identifying the chunks that have status in-progress we can resume upload for only those chunks.
        - How do we identify which chunk is which in the response compared to local chunks correctly?
            - We can use hashing, when we create chunk (of bytes) we can pass the chunk to a hash function to get unique chunk hash, we can make this as the chunkId while updating in the File document, and later when we have to compare chunks to identify which chunk is which( in case of resume upload as we discussed) we will use this chunkId to correctly identify the chunk and upload only those having status ‘in-progress’
    - How Trust But Verify will work here?
        - We can not just rely on the client to update the file-server that a given chunk has been uploaded to s3 successfully and update the status of that chunk to ‘completed’  for the given File
        - We have to verify that update first. So, when the file-server receives an update from the client that a particular chunk has been updated, the file-server then asks the s3 is it true that this given chunks has been uploaded successfully? Based on the response from s3 the file-server will update the status of that chunk
        - This approach is called Trust But Verify 
        - This is a great solution. (if we use CDC change data capture then it will also work but will add additional complexity)
    - How to update the status of chunk using change data capture?
        - So when ever s3 is updated we can put the change in some sort of event stream like kafka  via s3 notifications and we can have  worker groups read from the event stream and update the file-server about the update and file-server can then update the chunk status to ‘completed’
        - As we can see this is a tedious process and we are far better off with the Trust But Verify Approach instead.

