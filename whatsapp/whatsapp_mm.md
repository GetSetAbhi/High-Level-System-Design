```
Core problem statement
│
└── Build a low-latency, reliable chat app with offline delivery
    ├── Real-time 1:1 and group messaging
    ├── Media attachments
    └── Delivery despite failures and offline users

├── Initial solution:
│   └── Single Chat Server
│       ├── WebSocket keeps users connected
│       ├── In-memory map tracks connections
│       └── Server directly sends messages to recipients
│
├── Problem:
│   ├── Cannot scale to billions of connections
│   ├── Fails when users connect to different servers
│   └── Messages are lost when users are offline or server fails
│
├── Improved solution:
│   └── Durable message delivery
│       ├── Store messages in a database
│       ├── Store undelivered messages in an Inbox
│       └── Delete Inbox entries after client ACK
│
├── Problem:
│   ├── Multiple servers need message routing
│   ├── Pub/Sub delivery can be lost
│   └── Multiple devices need separate delivery tracking
│
└── Best practical solution:
    └── Scalable real-time messaging architecture
        ├── Load balancer + horizontally scaled WebSocket servers
        ├── Pub/Sub routes messages between servers
        ├── Durable Inbox guarantees offline delivery
        ├── ACKs, retries, heartbeats, and sequence numbers handle failures
        ├── Per-device Inbox supports multi-device sync
        ├── Separate media service handles attachments
        └── Fast delivery uses Pub/Sub; durability comes from the database
```