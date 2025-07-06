# Dropbox / Google Drive

Took references from **System Design Interview by Alex Xu Volume I** and A talk by a [dropbox engineer](https://www.youtube.com/watch?v=PE4gwstWhmc)

![Dropbox](dropbox.svg)

## File Upload and download Workflow

Similar to how [Dropbox](../dropbox/) design was discussed


## File Sync Workflow

1. Change Detection: Both the User Device and the cloud system constantly monitor for changes (new, modified, deleted, renamed files).

2. Client-Side Sync: When a user makes a local change:

	* The User Device chunks and hashes the file.
	* It asks the File Service to initiate an upload with these hashes.
	* The File Service (using the Block Service) de-duplicates blocks, getting `presignedUploadUrls` only for new blocks.
	* The User Device uploads only these new blocks directly to S3.
	* The File Service updates its metadata, potentially detecting and resolving conflicts (e.g., creating a conflict copy if the cloud version is newer).

3. Cloud-to-Client Sync: When a change occurs in the cloud (e.g., from another device):

	* A Notification Service alerts other User Devices about the change.
	* The User Device requests the changes from the File Service.
	* The File Service provides presignedDownloadUrls for any new/modified blocks.
	* The User Device downloads these blocks directly from S3 and applies the changes to its local file system (e.g., overwriting, deleting, recreating files).

## Notification Service

When a device comes online after being offline for a significant period, it needs a way to catch up on all changes that happened while it was disconnected.

* It immediately establishes a long polling connection with the Notification Service, providing its `lastSyncTime`. This connection is designed to quickly trigger a Cloud-to-Local Sync if any changes have occurred since that `lastSyncTime`. The Notification Service acts as a consumer of event coming from the Message Queue for which FileService is a producer. 

* While the long polling connection is active and if the notification service comes across a recent event related to the user id associated with the device, then it will send a signal to the user device to trigger a sync. if a long polling connection times out, the Notification Service will implicitly signal the User Device to proceed with a full sync.

* The device is now online and has potentially caught up (or is in the process of catching up), It then establishes a persistent WebSocket connection with the Notification Service.

When a User Device establishes a connection (either a Long Polling request or a WebSocket):

The Notification Service associates the incoming connection's socket/connection ID with the User ID (and potentially a Device ID) that authenticated the connection.
It stores this mapping in an efficient, concurrent data structure (e.g., a hash map or dictionary).

`Map<UserId, Map<DeviceId, ConnectionId>>`

`Map<ConnectionId, UserSessionDetails> (which includes UserId, DeviceId, lastSyncTime, and the actual socket object/handle)`.


## Why RPC connection between block service and File Service ?

The File Service and Block Service are considered internal, core components of the distributed system. 
They are not exposed directly to external clients (like User Devices). 
In such tightly integrated microservices, RPC simplifies communication

RPC frameworks often use more efficient binary serialization formats (like Protocol Buffers, Apache Thrift, gRPC) 
compared to text-based formats like JSON or XML used in REST. This results in smaller message sizes and faster parsing.

# Databases

FileVersions is a join table. The FileVersions table will have multiple rows for a single file_version_id:

* Each row in the FileVersions table represents one specific block that belongs to a particular file_version_id.

* The block_hash in each of these rows is indeed a foreign key referencing the Blocks table, which holds the actual information about that individual block.

* The block_sequence_index in FileVersions is crucial, as it defines the order of these blocks for that specific file version.

```
+---------------------------------+        +-----------------------------+
|        File MetaData DB         |        |      Block MetaData DB      |
+---------------------------------+        +-----------------------------+
|             Folders             |        |           Blocks            |
+---------------------------------+        +-----------------------------+
| - folder_id (PK)                |<-------|- block_hash (PK)             |
| - parent_folder_id (FK)         |        | - s3_location               |
| - name                          |        | - reference_count (INT)     |
+---------------------------------+        | - size_bytes                |
         |                             ^    +-----------------------------+
         | contains                  | logical_blocks_mapped_to
         V                             |
+---------------------------------+      +-----------------------------+
|             Files               |<-----|         FileVersions      |
+---------------------------------+      | (or FileBlockMapping)       |
| - file_id (PK)                  |      +-----------------------------+
| - user_id (FK)                  |      | - file_version_id (PK)      |
| - parent_folder_id (FK)         |      | - file_id (FK)              |
| - name                          |      | - block_hash (FK)           |
| - current_version_id (FK)       |      | - block_sequence_index (INT)|
| - size_bytes                    |      | - version_timestamp         |
| - status (e.g., 'uploaded', 'pending')|  +-----------------------------+
+---------------------------------+
         |
         | owns
         V
+---------------------------------+
|         UserDevices             |
+---------------------------------+
| - device_id (PK)                |
| - user_id (FK)                  |
| - last_sync_time (TIMESTAMP)    |
+---------------------------------+
```