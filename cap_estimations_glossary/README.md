# Capacity estimates

```
----------------------------------------------
| System              | Base Assumption      |
|---------------------|----------------------|
| URL Shortener       | 10M DAU              |
| Chat App            | 10M DAU              |
| Search              | 10M DAU              |
| Payment             | 10M DAU              |
----------------------------------------------
| Notification System | 100M events/day      |
| Social Network      | 100M DAU             |
| Video Streaming     | 100M DAU             |
| Hotel Reservation   | 100M DAU             |
| Dropbox             | 100M DAU             |
----------------------------------------------


Storage estimation were done for following:
----------------------------------------------
| System              | Base Assumption      |
|---------------------|----------------------|
| URL Shortener       | 10M DAU              |
| Chat App            | 10M DAU              |
| Video Streaming     | 10M DAU              |
| Search Autocomplete | Derived from 10M DAU |
----------------------------------------------
```

Assume 10M DAU, 10 actions/day → 100M requests/day → ~1K QPS average, ~5K peak. At ~100 GB/day append-only data, this fits a sharded relational DB or write-optimized NoSQL with caching. The key challenge is throughput, not storage.

1kb per event is the safe size for payload one can assume. 1kb per event combined with 1000 QPS translates to 1MB per second which then becomes 100GB per day assuming 100000 seconds per day

Peak QPS = 5-10 x QPS calculated

By pareto principle assume that 20% of the QPS are WRITES and 80% of QPS are reads

“If write QPS grows beyond ~10–20K/sec or data becomes multi-TB/day, I’d consider sharding or a NoSQL store.”

We target 99.99% availability, which allows about 50 minutes of downtime per year.

```
--------------------------------------------------------------------------------------------
| DAU Range     | QPS       |    What it implies       | Examples                          |
| ------------- |-----------|--------------------------| ----------------------------------|
| **100K – 1M** |           | Small/medium system      | Internal tools, B2B apps          |
| **~10M**      | 1000 QPS  | Large distributed system | Twitter-like, payments, messaging |
| **100M+**     | 10k QPS   | Internet-scale           | Facebook, WhatsApp, Google        |
--------------------------------------------------------------------------------------------

--------------------------------------------------------
| Peak QPS | Data growth   | Typical DB choice         |
| -------- | ------------- | ------------------------- |
| <1K      | <10 GB/day    | Single Postgres/MySQL     |
| 1–10K    | 10–100 GB/day | RDBMS + replicas          |
| 10–50K   | 100+ GB/day   | Sharded RDBMS / NoSQL     |
| 50K+     | TB/day        | Cassandra / DynamoDB / S3 |
--------------------------------------------------------

----------------------------------------------------
| Media Type | Safe Avg Size | Max Size (Ballpark) |
| ---------- | ------------- | --------------------|
| Image      | ~300 KB       | ~1 MB               |
| Video      | ~20 MB        | ~100 MB             |
| Audio      | ~5 MB         | ~20 MB              |
| Metadata   | ~1 KB         | ~10 KB              |
| Webpage    | ~300 KB       | ~1 MB               |
----------------------------------------------------
In above the webpage size estimate is excluding video and audio

---------------------------------------------------------------------------------------------------
| Availability           | Downtime / year | Downtime / day       | What it means practically     |
| ---------------------- | --------------- | ---------------------|--------------------------------
| **99%**                | ~3.65 days      | ~14.4 minutes/day    | Frequent outages              |
| **99.9% (3 nines)**    | ~8.7 hours      | ~1.44 minutes/day    | Acceptable for internal tools |
| **99.99% (4 nines)**   | ~52 minutes     | ~8.6 seconds/day     | Good production service       |
| **99.999% (5 nines)**  | ~5 minutes      | ~0.86 seconds/day    | Very hard, expensive          |
| **99.9999% (6 nines)** | ~30 seconds     | ~0.08 seconds/day    | Extremely rare, elite systems |
---------------------------------------------------------------------------------------------------

```

# System Design – Universal Capacity & Latency Cheat Sheet

## Universal Assumptions (Safe Defaults)
- **DAU**: 10 million
- **Requests per user per day**: 10
- **Seconds per day**: 100,000 (rounding for easy math)
- **Peak factor**: ×10
- **Read : Write ratio**: 80 : 20
- **Avg request / event size**: 1 KB
- **Latency (end-to-end)**: ~500 ms or 200ms (p99 < 500 ms) anything less than 500ms looks instant for user

---

## QPS Estimation (2-minute math)
- For facebook level DAU can be 100M
- Daily requests = 10M × 10 = **100M**
- Average QPS = 100M / 100K = **1K QPS**
- Peak QPS = 1K × 10 = **10K QPS**

Breakdown:
- Reads ≈ **8K QPS**
- Writes ≈ **2K QPS**

---

## Storage Estimation (Rule of Thumb)
- Writes per second = 2K
- Data per write = 1 KB
- Daily data = 2K × 100K × 1 KB = **200 GB/day**
- Yearly data ≈ 200 × 400 = **~80 TB/year**

---

## When to Use a Queue
Use a queue when:
- QPS ≥ **1K** and traffic is bursty
- Work takes > **30–50 ms**
- Retries or durability required
- Async processing acceptable

Rule:
> If it doesn’t need to be synchronous → put it behind a queue

---

## Eviction Policy
- **LRU** → Fast-changing, temporal access (feeds, sessions)
- **LFU** → Stable hot keys (trending, popular items)
- Default when unsure: **LRU**

---

## Availability vs Durability
- **Availability** = uptime (time-based)
- **Durability** = data safety (probability-based)

Targets:
- User-facing services: **99.9–99.99%**
- Payments / storage: **99.99%+ durability**


## LRU VS LFU 

```
--------------------------------------------------------------
| System              | Why LRU works                        |
|-------------------- | -------------------------------------|
| Timeline/feed pages | Users scroll recently viewed content |
| Session cache       | Recent sessions are likely reused    |
| Search results      | Recent queries repeat                |
| API response cache  | Temporal locality                    |
--------------------------------------------------------------

-------------------------------------------------
| System                  | Why LFU works       |
|------------------------ | --------------------|
| Trending items          | Always popular      |
| Hot products            | Frequently accessed |
| Popular videos          | Consistent traffic  |
| Config / reference data | Reused constantly   |
-------------------------------------------------
```

# URL Shortener

let us assume my url shortening service has a traffic of 10M users assume 1 user is shortening 10 URLs and only 20% of users are shortening URLS then 20M URLs are being shortened in a day which is 200 writes per second and 2000 writes per second as peakwith 10x load factor. Similarly with 80% users on read path and assuming 1 users is accesing 10 URLs per day then read traffic is 80M and 800 redirects/sec with 8K redirects per second as peak. If 1 urls has on an average 100 chars and 1char = 1B then database is growing at 2GB per day. if we include metadata too then storage becomes ~5GB per day and 2TB per year (5*400 days)

Inferences:

- Read Heavy system
- A single primary database with read replicas is sufficient.
	
	Why:
	
	* Write QPS (~2K peak) is easily handled by a modern RDBMS
	* Storage growth (~2GB/day, ~2TB/year) is modest
	* No immediate need for sharding
- Availability can be achieved without complex distributed systems.

	Why:

	* Single primary DB + replicas
	* Cache failure falls back to DB
	*Stateless application servers

# Hotel reservation of event reservation system

For a hotel reservation system with 100M daily active users, assuming 20% of users make a reservation and each makes one booking per day, the system handles approximately 20M bookings per day, or about 200 bookings per second on average.

Accounting for a 10× peak load factor during traffic spikes, the system must be designed to handle ~2K booking requests per second.

Key inferences an interviewer expects you to draw

- Bookings are low-QPS but high-value operations
- Strong consistency is required (inventory, pricing, payments)
- Read-heavy system overall (browse vs book)
- Write path must be carefully serialized and protected
- Queue + transactional DB needed for bookings
- Caching applies only to browsing, not booking
- The write path / booking path should be durable and bookings should not be lost

# Dropbox

if I am designing a dropbox kind of system then let me assume that there are 100M DAU.
Let's just say every user is uploading 5 files per day and only 20% of daily active users are uploading files.
So now we have 100M files being uploaded per day which translates to 10K file uploads per second considering a load factor of 10x.
Modern day relational databases can handle upto 10K writes these days and we are almost touching that limit, so it is safe to assume that we
can go with a sharded relational database to store file metadata and store the file object separately in a file storage.

Coming to 80% users, let's just say on an average these 80% users are downloading 5 files per day so by calculation my read traffic becomes 40k downloads per second considering a load factor of 10x.

Clearly we have to make sure that the files are being downloaded with minimum latency, so download path needs to be optimised

# Google Docs Multiuser document edit

10M Daily Active Users (DAU)

20% users actively editing per day

Each active user edits 5 documents/day

Peak load factor: 10×

Average edit operation size: 100 bytes

10M edits/day × 100 bytes ≈ 1 GB/day
≈ 365 GB/year (excluding snapshots & metadata)


Reads dominate numerically, but are cacheable.

# Distributed Job Scheduler

Assuming 10 million unique jobs per day, the system handles about 100 job submissions per second on average and roughly 1,000 jobs per second at peak. This write volume can be comfortably handled by a relational database.

However, in a distributed job scheduler, each job generates multiple state transitions during its lifecycle, As a result, each job causes several durable writes, leading to write amplification. When these lifecycle writes are accounted for, the system performs several thousand writes per second at peak, making it a write-heavy system. Reads are comparatively fewer and often predictable.

Because job execution is asynchronous and jobs must not be lost, the system cannot rely on synchronous database polling. A durable queue is required to decouple job submission from job execution, handle bursty traffic, and ensure reliability.

# Gaming leaderboard

for a leaderboard system, let's say there are 10M DAU and each user plays 10 games per day.
every time a game is finished a user score is submitted Leaderboard is opened to display top users and users rank.

100M games being played daily so score api is being called 100M times which means 10k QPS, with 10x as peak load factor, for score API

and similarly 10k QPS for leaderboards.

In this case read and writes seem to be balanced, but its actually the read path (get leaderboard and rank) that needs to be optimised so that we can get the leaderboard with minimum latency

# Notification service

if the system is handling 100M notifcations per day then it means 1K notifciation per second and on peak 10K notifications per second assuming load factor of 10x

- **The system is write-heavy**

	Optimize for high write throughput and asynchronous processing.

- **Asynchronous processing is mandatory**
	
	At 10K notifications/sec:
	Synchronous delivery would block API threads
	
# Typeahead System

I am designing a typeahead (autocomplete) system with 10 million daily active users (DAU).
If each user performs 10 searches per day, the system handles approximately 100 million searches per day.

Typeahead suggestions are fetched on every keystroke, which means there is one API call per character typed.
Assuming that each search contains 2 words on average, each word has 5 characters, and each character is 1 byte, a single search consists of 10 characters.

This results in 10 API calls per search, leading to a total of 1 billion suggestion fetch requests per day.
That translates to roughly 10K requests per second on average, and 100K requests per second at peak, assuming a 10× peak load factor.

Now, assuming that only 20% of searches result in unique words, about 20 million searches per day contribute new vocabulary.
Since each search has 2 words on average, the system ingests 40 million unique words per day.
At 5 characters per word, this corresponds to 200 million characters per day, or approximately 200 MB of storage growth per day, which is about 80 GB per year.

This amount of data can be comfortably stored in a partitioned and replicated persistent database.
However, given the high read throughput of up to 100K suggestion requests per second and the need for very low latency, serving suggestions directly from disk-backed storage is not sufficient.

To meet latency requirements, the system must rely on in-memory indexes.
This naturally leads to using Trie-based in-memory data structures, which are well suited for efficient prefix lookups and can serve autocomplete suggestions with minimal latency.

# Web Crawler

If a web crawler downloads 10 million pages per day, that corresponds to roughly 116 pages per second on average and about 1,100 pages per second at peak, assuming a 10× load factor.

Assuming an average page size of 300 KB, the crawler ingests approximately 3 TB of data per day. At this scale, raw page content must be stored in distributed object storage, while crawl metadata and indexing information can be stored separately in a database.”

# Whatsapp Chat

With 10M DAU and 10 messages per user per day, we expect ~100M messages per day, or ~1–1.2K messages per second on average and ~10–12K per second at peak. Assuming ~100 bytes per message, storage grows by ~10GB per day or ~4TB per year, excluding metadata. At this scale, we need horizontal scaling and partitioned storage.

Assuming each user is opening the app 10 times a day and on each app open, recent converstaions are loaded then there are 100M load conversations calls which is 1K calls per second or 10k calls per second at peak. And this is only for opening the app, but when the user opens individual chats then there are more requests sent to fetch the conversation history itself, which adds up to the 10k calls we calculated. So the system is a net- read system mostly.

By calculation, the system appears read heavy, but it is write heavy in terms of correctness.

# Facebook twitter Timeline/feed

for a social network with 100 Million daily active users. 

Following pareto principle, i'll assume only 20% of daily active users are posting, and 80% of the daily active users are browsing their timeline feed

20M DAU for writing posts and 80M DAU for scrolling the timeline.

If we say that only 20% of users post per day and 1 user posts about 10 post in a day then there are 200Million posts per day which becomes 2k posts per second amd accounting for 10x as peak load my peak load now becomes 20k posts written per second.

At this scale, the system becomes write-heavy, and we need sharding and asynchronous processing via queues to handle bursty traffic and downstream fan-out safely.”

Out of 100M DAU, 80M users are readers. Assuming each opens the app 10 times a day, we get around 800M read requests per day, which is roughly 8K QPS on average and about 80K QPS at peak. Since the system is read-heavy, we rely heavily on caching, feed precomputation, and read replicas to serve reads efficiently.

At peak, we expect ~80K read QPS. Assuming an 80% cache hit rate, only ~16K QPS reaches the database, which is manageable with read replicas.

# Youtube Video Streaming

For a video streaming service like youtube, if I assume 100M DAU and assume only 20% of users are uploading 1 video of average size 100 MB then daily storage is 2PB and yearly storage is 800 PB

Even if only 20% upload, all 100M users will stream videos, so:

Video read traffic (bandwidth) will far exceed write traffic

This is critical for scaling the CDN, network, and edge caches, more so than the database size