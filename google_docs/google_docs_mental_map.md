Real-time collaborative document editing
│
├── Initial solution:
│   └── Send full document snapshot on every edit
│       ├── Client sends entire document text to server
│       ├── Server stores latest snapshot in blob storage (e.g., S3)
│       └── Server broadcasts full snapshot to all connected editors
│
├── Problem:
│   ├── Massive bandwidth waste (100s of KB per keystroke)
│   ├── Concurrent edits overwrite each other (last-write-wins)
│   └── Example: User A adds ", world" and User B deletes "!" → one edit is lost
│
├── Improved solution:
│   └── Send small edit operations only
│       ├── INSERT(position, text)
│       └── DELETE(position, length)
│
├── Problem:
│   ├── Operations are position-based and context-dependent
│   ├── Same operations in different order produce different documents
│   ├── Example: INSERT at 5 then DELETE at 6 ≠ DELETE at 6 then INSERT at 5
│   └── Users see divergent document states
│
├── Improved solution:
│   └── Operational Transformation (OT) on centralized server
│       ├── All editors connect via WebSocket to Document Service
│       ├── Server imposes canonical order on operations
│       ├── Server transforms concurrent operations against each other
│       ├── Append transformed ops to Document Operations DB (Cassandra)
│       └── Broadcast transformed op to all connected editors
│
├── Problem:
│   ├── Clients apply local edits optimistically before server ack
│   ├── Remote ops may arrive while local ops are still pending
│   └── Blindly applying remote ops corrupts local state (positions drift)
│
├── Improved solution:
│   └── Client-side OT as well
│       ├── Transform remote ops against unacknowledged local ops
│       ├── Apply transformed remote op without losing local work
│       └── Guarantees convergence despite different perceived orders
│
├── Problem:
│   ├── Single Document Service doesn't scale to millions of connections
│   ├── Single point of failure for availability
│   └── All editors of one doc must share same OT context
│
├── Improved solution:
│   └── Horizontally scale Document Service with consistent hashing
│       ├── Hash docId to assign document to a specific server
│       ├── All editors of same doc connect to same server
│       ├── Use coordination (ZooKeeper/K8s) for hash ring membership
│       ├── Redirect clients to correct server before WebSocket upgrade
│       └── Load stored ops from DB when document moves to new server
│
├── Problem:
│   ├── Billions of documents, millions of ops per doc over time
│   ├── Storing every operation forever is expensive
│   ├── New clients must replay too many ops to load document
│   └── Active documents consume too much memory in Document Service
│
├── Improved solution:
│   └── Periodic snapshot/compaction
│       ├── Merge many operations into a compact document state
│       ├── Store snapshot (e.g., in blob store or as single op)
│       ├── Keep only recent "tail" of operations
│       └── Load document as: snapshot + short op tail
│
└── Best practical solution:
    └── OT-based real-time editor with scalable stateful layer
        ├── WebSocket per editor, bi-directional low-latency channel
        ├── Central OT server per document (via consistent hashing)
        ├── Durable op log (Cassandra) + periodic compaction
        ├── Ephemeral cursors/presence in server memory, broadcast over WS
        ├── Client-side OT for optimistic local edits + convergence
        ├── Metadata DB (Postgres) for document info and ACLs
        └── Blob store for compacted snapshots, DB for op tails
		

```		
CREATE TABLE documents (
  document_id UUID PRIMARY KEY,
  active_document_version_id UUID NOT NULL,
  title TEXT,
  updated_at TIMESTAMP
);

CREATE TABLE document_operations (
  document_id UUID,
  document_version_id UUID,
  server_timestamp TIMEUUID,
  operation_id UUID,
  author_id UUID,
  operation_payload BLOB,

  PRIMARY KEY (
    (document_id, document_version_id),
    server_timestamp,
    operation_id
  )
) WITH CLUSTERING ORDER BY (server_timestamp ASC, operation_id ASC);
```