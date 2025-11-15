# Whatsapp Schema

```
CREATE TABLE messages (
    conversation_id   text,
    message_ts        timeuuid,
    sender_id         text,
    body              text,
    PRIMARY KEY (conversation_id, message_ts)
) WITH CLUSTERING ORDER BY (message_ts DESC);

```

This creates a table called messages with 4 columns:

* **conversation_id (text)** : Which chat/conversation the message belongs to.
* **message_ts (timeuuid)** : A timestamp-encoded unique ID that also gives ordering.
* **sender_id (text)** : Who sent the message.
* **body (text)** : The text message content.

** Meaning of the PRIMARY KEY**

`PRIMARY KEY (conversation_id, message_ts)`


Cassandra’s primary key has two parts:

(A) Partition Key : `conversation_id`

All messages for the same conversation go into the same partition.
A partition is stored on one node (or replicas) based on hashing.
Queries must specify conversation_id to retrieve messages.
So every chat (1:1 or group) is a separate partition.

(B) Clustering Key : `message_ts`

	Within the partition, all rows (messages) are sorted by message_ts.
	
	This allows fast:
		“load latest N messages”
		“scroll up for older messages”
		“read messages in correct order”

🔄 3. Meaning of:

`WITH CLUSTERING ORDER BY (message_ts DESC)`

This tells Cassandra:

	Store messages in descending order of message_ts inside each conversation.
	So the newest messages appear first in the partition.
	This makes the most common messaging query extremely fast:

`SELECT * FROM messages WHERE conversation_id = ? LIMIT 50;`

Cassandra can return the newest 50 messages directly without scanning.