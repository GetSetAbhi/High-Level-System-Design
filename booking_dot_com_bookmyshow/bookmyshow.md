Prevent double booking
│
├── Initial solution:
│   └── Transactional database
│       ├── Check ticket availability
│       ├── Update ticket status
│       └── Create booking
│
├── Problem:
│   └── User may lose the ticket while entering payment details
│
├── Improved solution:
│   └── Long-running database lock
│       └── Keep the ticket locked during checkout
│
├── Problem:
│   ├── Database transaction stays open for minutes
│   ├── Locks consume database resources
│   ├── Abandoned users may block tickets
│   └── High contention can damage performance
│
├── Improved solution:
│   └── Ticket status + expiration time + cleanup job
│       ├── Mark ticket RESERVED
│       ├── Store reserved_until
│       └── Cron job releases expired reservations
│
├── Problem:
│   ├── Cron cleanup may be delayed
│   ├── Expired tickets may appear unavailable temporarily
│   └── Cleanup creates additional database load
│
└── Best practical solution:
    └── Distributed lock with TTL
        ├── Redis atomically acquires the ticket lock
        ├── TTL automatically releases abandoned locks
        ├── Database remains the final source of truth
        ├── Payment success marks ticket SOLD
        └── Database transaction prevents double booking
		
###########################################################################

Search for events
│
├── Initial solution:
│   └── Search directly in the Events database
│
├── Problem:
│   ├── Wildcard queries can scan many rows
│   ├── Search latency increases with data size
│   └── Primary database is overloaded
│
├── Improved solution:
│   └── Add SQL indexes and optimize queries
│
├── Problem:
│   └── Full-text and ranking requirements may exceed normal SQL indexes
│
└── Best practical solution:
    ├── Build a search index from event data
    ├── Use Elasticsearch/OpenSearch
    ├── Cache common queries
    └── Use CDN or edge caching where appropriate
	
############################################################################

Support many users viewing an event
│
├── Initial solution:
│   └── One Event Service connected to one database
│
├── Problem:
│   ├── One service instance becomes a bottleneck
│   └── Repeated reads overload the database
│
├── Improved solution:
│   ├── Multiple Event Service instances
│   └── Load balancer
│
├── Problem:
│   └── Millions of users may still request the same event data
│
├── Best practical solution:
│   ├── CDN for publicly cacheable content
│   ├── Redis/Memcached for event metadata
│   ├── Horizontal service scaling
│   ├── Database read replicas where useful
│   └── Virtual waiting room for extremely popular events

################################################################################

Show users available seats
│
├── Initial solution:
│   └── Client fetches seat status from the database
│
├── Problem:
│   └── The seat map becomes stale during a popular sale
│
├── Improved solution:
│   └── Push updates using SSE or WebSockets
│
├── Problem:
│   ├── Many persistent connections are expensive
│   ├── Updates may be difficult to broadcast
│   └── Users can still race for the same seat
│
└── Best practical solution:
    ├── Virtual waiting room limits active users
    ├── Seat map remains manageable
    ├── Reservation endpoint performs the real check
    └── Database transaction guarantees final consistency