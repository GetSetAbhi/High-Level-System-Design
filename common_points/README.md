# Common Points for System Design
---

* **Push vs Pull** - In Designing Message Queue or Metrics and Monitoring System, there is a common point of Push and Pull
	* In Message Queue, the consumer can either pull messages from broker or Broker can push messages to Consumers
	* Similarly in Metrics and Monitoring System, either the Metrics Collection Workers can pull data from the services or Collection Agent that runs
	on services will push data to the Collection Workers

* **Video Transcoding and Adaptive Bit Rate Streaming** - This is applicable in both the design youtube and design a live streaming service (Twitch)

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
