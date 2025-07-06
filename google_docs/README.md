# Google Docs

![Dropbox](dropbox.svg)

## File Upload and download Workflow

Similar to how [Dropbox](../dropbox/) design was discussed

## How to Represent Document Content?

**Why we can't use flat sequence of characters to represent document content ?**

1. No Structure: Fails to represent rich document hierarchy (paragraphs, headings, lists).
2. Fragile Formatting: Difficult to manage and apply rich text attributes (bold, color) as offsets change with edits.
3. Inefficient Complex Operations: Operations like moving entire sections or tables are cumbersome, requiring extensive re-calculation of offsets.
4. Challenging for Collaboration: Leads to complex conflict resolution and operation transformation issues due to constantly shifting character positions, making consistency difficult.

**Why a tree structure is used to represent document content ?**

1. Representing document content as a tree structure in collaborative editing systems like Google Docs is preferred because it:
2. Models Hierarchy: Naturally reflects the nested structure of documents (paragraphs, headings, lists).
3. Manages Rich Formatting: Allows attributes (like bold, color, alignment) to be easily attached to specific content nodes.
4. Enables Efficient Complex Operations: Simplifies operations like moving or deleting entire sections, tables, or paragraphs by manipulating whole nodes.
5. Enhances Collaboration Robustness: Provides stable reference points (nodes) for changes, which greatly aids conflict resolution in algorithms like Operational Transformation (OT) or CRDTs.

Following is an example of how a document content tree is represented.

```
[Document] (Root Node)
  |
  +-- [Paragraph] (Attributes: align="left", id="p123")
  |     |
  |     +-- [Text Span] (Content: "This is some ")
  |     +-- [Text Span] (Content: "bold text.", Attributes: bold=true)
  |     +-- [Text Span] (Content: " And ")
  |     +-- [Text Span] (Content: "italicized!", Attributes: italic=true)
  |     +-- [Text Span] (Content: ".")
  |
  +-- [Heading 1] (Content: "Introduction", Attributes: level=1, id="h456")
  |
  +-- [Paragraph] (Content: "Here's a list of items:", id="p789")
  |
  +-- [List] (Attributes: type="bullet", id="l101")
        |
        +-- [List Item] (Attributes: id="li202")
        |     |
        |     +-- [Paragraph] (Content: "First item.")
        |
        +-- [List Item] (Attributes: id="li303")
              |
              +-- [Paragraph] (Content: "Second item, with a ")
              +-- [Paragraph] (Content: "new line inside.")
```

## How does collaborative editing happen ?

let's walk through a real-time example of document editing with two users, focusing on how differential synchronization works with a tree representation.

1. Let's say our document initially looks like this (simplified tree):
	```
	[Document]
	  |
	  +-- [Paragraph_A] (id: pA)
	  |     +-- [Text_Span_1] (content: "Hello, ")
	  |     +-- [Text_Span_2] (content: "world!")
	  |
	  +-- [Paragraph_B] (id: pB)
			+-- [Text_Span_3] (content: "This is a ")
			+-- [Text_Span_4] (content: "sample.")
	```
	Both Client 1 (User A) and Client 2 (User B) start with this exact version of the document.

2. Client 1 (User A): User A types "big " before "world!" in Paragraph_A.

	* Local Application (Optimistic UI): Client 1 immediately updates its local display: "Hello, big world!"
	* Diff/Operation Generation: Client 1's synchronization logic compares its new local state to its last acknowledged state. It detects a change within Text_Span_2 of Paragraph_A.
	* It generates an Operation (Op): [Op_A_1] = { type: "insert", target_node_id: "Text_Span_2", position: 0, content: "big " }
	* Sending Op: Client 1 sends Op_A_1 to the Collaboration Server.

3. Simultaneously, Client 2(User B) decides to bold "sample" in Paragraph_B.

	* Local Application (Optimistic UI): Client 2 immediately updates its local display: "This is a sample."

	* Diff/Operation Generation: Client 2 generates an Op:
	```
	[Op_B_1] = { type: "apply_attribute", target_node_id: "Text_Span_4", attribute: { bold: true } }
	```
	* Sending Op: Client 2 sends Op_B_1 to the Collaboration Server.Simultaneously, User B decides to bold "sample" in Paragraph_B.

4. The Collaboration Server Op_A_1 and other Broadcasts

	* The server receives Op_A_1 from Client 1.

	* It applies Op_A_1 to its authoritative document tree. Text_Span_2 in Paragraph_A is updated to "big world!".

	* The server then broadcasts Op_A_1 to all other connected clients (in this case, Client 2).
	
5.  Client 2 Receives Op_A_1 (and Potentially Transforms its Own Pending Op)

	* Client 2 receives Op_A_1 from the server.

	* Crucial Step (Transformation for OT): Client 2 has a pending local change (Op_B_1) that it hasn't sent/received acknowledgement for yet. Since Op_A_1 affects Paragraph_A and Op_B_1 affects Paragraph_B, these operations are non-conflicting in terms of their target nodes. 
	
	Therefore, no complex transformation of Op_B_1 is strictly needed against Op_A_1 in this specific case, as they operate on different branches of the tree.

	* Application: Client 2 applies Op_A_1 to its local document tree. Its view updates to: "Hello, big world! This is a sample."

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