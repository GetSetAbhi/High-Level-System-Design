# Snapchat capacity estimates

suppose there are 10M users of the app out of which only 10% are concurrently active.
If every user has on an average 100 friends and location update is sent every 30 seconds, then the system is handling 33K location writes per second.
If friends have to be notified about latest location of a user, then we will be having 3.3M location updates per second

## Inferences

- High write throughput

  ~33K location updates/sec require horizontally scalable ingestion.

- Fan-out amplification dominates

  Each update fans out to ~100 friends, creating ~3.3M fan-out events/sec.

- Asynchronous processing is mandatory
 
  Synchronous fan-out would block requests and collapse throughput.

- Queue-based architecture required
  
  Queues absorb bursts and decouple ingestion from fan-out.

- Database cannot handle fan-out writes
  
  Persist only the latest location per user; avoid per-friend DB writes.

- Push vs pull trade-off favors pull
  
  Friends should fetch latest location on demand instead of being pushed updates.

- Eventual consistency is acceptable
  
  Slightly stale location data is tolerable for user experience.

- Partitioning by user_id is necessary
  
  User-keyed sharding minimizes contention and improves write scalability.

- In-memory systems are preferred for fan-out
  
  Fan-out delivery should use caches or streams, not disk-backed storage.
  
## Why redis pub-sub for fan out and not kafka ?

- Location updates expire in seconds,Older updates are useless once a new one arrives. Location data is ephemeral in this case
- 3.3M events/sec is massive and disk based system would increase latency while we are aiming for near - real time latest update.
- We don't need a Kafka like queue as we don't need durability, data replay capability and data ordering guarantees.